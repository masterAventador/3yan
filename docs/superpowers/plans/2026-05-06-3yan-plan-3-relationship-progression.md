# 三言 · Plan 3 实施计划：关系养成系统（5 阶段 + 亲密度引擎）

> **For agentic workers:** REQUIRED SUB-SKILL: `superpowers:subagent-driven-development`.
> **TDD 铁律：** 每个 task 严格 Red → Green → Refactor。
> **前置依赖：** Plan 1（MVP+WS）+ Plan 2（长期记忆） + S3（business 模块拆分）已全部完成 ✅
> **本版本：** 2026-05-18 重新 brainstorming（修订角色名 / 模块结构 / migration 版本号 / PromptBuilder 注入方式）。

**Goal：** 关系会推进。用户从陌生人聊到朋友、暧昧、恋人、老夫老妻。每天回来打开聊天页看到亲密度涨一点，看到**小婉**称呼变化、说话方式变化。dogfood 第三关：每天打开有进度感吗？

---

## 1 · 设计概览

**架构：**
- **5 阶段**：陌生人(0-100) → 朋友(100-300) → 暧昧(300-600) → 恋人(600-1000) → 老夫老妻(1000+)
- **亲密度引擎 D 方案**（行为分 + 剧情节点 + AI 评估三种来源）
  - 行为分：消息(+1，每日封顶 50) + 每日首次登录(+10 + streak×5，封顶 +60)
  - 剧情节点：DeepNightChatRule(+50) / FirstHonestShareRule(+30) / StageEntryCelebrationRule（仅剧情演出，不加分）
  - AI 评估：ConversationQualityEvaluator 调 DeepSeek V4-Flash，每 10 条用户消息 +0~+20
- **称呼/语调动态**：每个 stage 在 `character.persona_config.stage_overrides` 里有 `address` / `tone_hint` / `topics_unlock`；`StageOverrideQueryService` 取出 → 拼成 stage prompt segment 文本 → 由 chat-core AiService 传给 PromptBuilder
- **进度 UI**：聊天页顶部 IntimacyProgressBar 显示当前阶段 + 距下一阶段进度；阶段切换 StageTransitionDialog 弹窗

**与旧版 spec 的关键差异（brainstorm 2026-05-18 校准）：**
- 角色名 "林小满" → **"小婉"**（V1 baseline 已 seed）
- S3 已完成，character-core / chat-core / character-api 模块都已存在
- embedding 切硅基流动 BGE-M3 API（无独立 embedding-server）
- **PromptBuilder 接收 `stagePromptSegment` 字符串**，不耦合 Relationship 类型；caller (AiService) 调 character-api 拼好字符串再传
- migration 版本号从 **V2** 起（V1 已是 baseline）

---

## 2 · 数据模型

### 2.1 修改 ai_character 表

```sql
ALTER TABLE ai_character
    ADD COLUMN base_prompt TEXT NOT NULL DEFAULT '',
    ADD COLUMN persona_config JSONB NOT NULL DEFAULT '{}';
```

`persona_config` 结构示例（小婉）：
```json
{
  "stage_overrides": {
    "0": { "address": "你 / {nickname}",            "tone_hint": "礼貌、有距离感、偶尔害羞",  "topics_unlock": [] },
    "1": { "address": "{nickname} / 真名",          "tone_hint": "自然、放松、会开玩笑",      "topics_unlock": ["吐槽","生活琐事"] },
    "2": { "address": "笨蛋/傻瓜/{nickname}+宝",    "tone_hint": "撒娇、试探、害羞",          "topics_unlock": ["想见面","未来设想","暗示性话题"] },
    "3": { "address": "老公/宝贝",                  "tone_hint": "撒娇、依赖、占有欲",        "topics_unlock": ["纪念日","思念","情感深度"] },
    "4": { "address": "老公/笨蛋（带亲昵感）",       "tone_hint": "生活感、小吵小闹、深度依赖", "topics_unlock": ["小日常","未来生活"] }
  }
}
```

### 2.2 新建 relationships 表（核心）

```sql
CREATE TABLE relationships (
    user_id         BIGINT   NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    character_id    BIGINT   NOT NULL REFERENCES ai_character(id) ON DELETE RESTRICT,
    intimacy_score  INT      NOT NULL DEFAULT 0,
    current_stage   SMALLINT NOT NULL DEFAULT 0,   -- 0=陌生 1=朋友 2=暧昧 3=恋人 4=老夫老妻
    version         INT      NOT NULL DEFAULT 0,   -- 乐观锁
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, character_id)
);
```

### 2.3 新建 intimacy_logs 表（审计 + 前端展示）

```sql
CREATE TABLE intimacy_logs (
    id            BIGSERIAL PRIMARY KEY,
    user_id       BIGINT      NOT NULL,
    character_id  BIGINT      NOT NULL,
    delta         INT         NOT NULL,
    reason        VARCHAR(60) NOT NULL,    -- 'MESSAGE_SENT' | 'DAILY_LOGIN' | 'PLOT:deep_night_chat' | 'AI_QUALITY' | 'CAPPED' ...
    metadata      JSONB,
    new_score     INT         NOT NULL,
    new_stage     SMALLINT    NOT NULL,
    created_at    TIMESTAMP   NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_intimacy_logs_user_char_time
    ON intimacy_logs (user_id, character_id, created_at DESC);
```

不加外键：审计表，降低耦合，便于按需归档。

### 2.4 新建 relationship_milestones 表（节点去重）

```sql
CREATE TABLE relationship_milestones (
    user_id       BIGINT      NOT NULL,
    character_id  BIGINT      NOT NULL,
    rule_id       VARCHAR(60) NOT NULL,    -- 'deep_night_chat' | 'first_honest_share' | 'stage_entry_1' ...
    triggered_at  TIMESTAMP   NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, character_id, rule_id)
);
```

### 2.5 Redis（不入 DB）

- **streak**：key `streak:user:<userId>` → `{streak: N, last_date: 'yyyy-MM-dd'}`，无 TTL
- **每日行为封顶**：key `behavior:user:<userId>:date:<yyyy-MM-dd>` → 累计 delta；TTL 36h

### 2.6 migration 拆分

dev 阶段拆 5 个迁移文件，review 简便；后续合并 baseline 时再 squash 进 V1：

```
V2__ai_character_add_persona.sql            -- ALTER 加 base_prompt + persona_config
V3__create_relationships.sql                -- relationships
V4__create_intimacy_logs.sql                -- intimacy_logs
V5__create_relationship_milestones.sql      -- relationship_milestones
V6__seed_xiaowan_persona.sql                -- UPDATE ai_character SET base_prompt+persona_config WHERE name='小婉'
```

---

## 3 · 模块组件结构

按 `~/.claude/rules/java-backend-business-layer.md`，新组件全部归属 **character-api / character-core**；chat-core 只做 3 处接入。

### 3.1 sanyan-character-api（契约层 · 新增）

```
com.sanyan.character/
├── CharacterApi.java                       # 已有 — 扩充：新增 4 个 relationship 相关方法
├── dto/
│   ├── AiCharacterDto.java                 # 已有
│   └── RelationshipDto.java                # 新增
└── event/
    ├── IntimacyChangedEvent.java           # 新增 (record)
    └── StageTransitionEvent.java           # 新增 (record)
```

> **架构决策**：按 `java-backend-business-layer.md §2.2`「一个 -api 只有一个主 `<Domain>Api` 接口」，
> Relationship 是 character 域的内嵌从属概念（user × character 之间的状态），关系操作合并进 `CharacterApi`，
> **不**新建 `RelationshipApi`。如果未来 relationship 业务膨胀（如多角色 / 关系网 / 跨域查询）再考虑拆 `sanyan-relationship` 独立模块。

`CharacterApi` 新签名（已有 + 新增）：
```java
public interface CharacterApi {
    /* —— 已有 —— */
    AiCharacterDto findById(Long characterId);                          // 不存在返回 null
    AiCharacterDto getById(Long characterId);                           // 不存在抛 BusinessException

    /* —— 新增（Plan 3） —— */
    RelationshipDto findOrCreateRelationship(Long userId, Long characterId);   // 唯一懒创建入口
    RelationshipDto fetchMyRelationship(Long userId, Long characterId);        // GET /me 编排：findOrCreate + recordLogin + DTO
    /** 拼好的 stage prompt 片段，供 chat-core 注入 system 消息。
     *  示例："当前关系阶段：暧昧。称呼用户用：宝。语调：撒娇、试探、害羞。" */
    String getStagePromptSegment(Long userId, Long characterId);
}
```

> `IntimacyEvent` 是 character-core 内部 record，**不暴露**到 -api；character-core 内部 listener / engine 直接调 `internal/intimacy/IntimacyRecordService` 即可。

`RelationshipDto`：
```java
public record RelationshipDto(
    Long userId, Long characterId,
    int intimacyScore, int currentStage, String currentStageName,
    int nextStageThreshold, double percentToNextStage
) {}
```

事件：
```java
public record IntimacyChangedEvent(Long userId, Long characterId,
                                   int oldScore, int newScore, int delta, String reason) {}
public record StageTransitionEvent(Long userId, Long characterId,
                                   int fromStage, int toStage) {}
```

### 3.2 sanyan-character-core（实现层 · 大幅扩充）

```
com.sanyan.character/
├── internal/
│   ├── AiCharacterEntity.java              # 已有 — 增 basePrompt + personaConfig 字段
│   ├── AiCharacterRepository.java          # 已有
│   ├── CharacterErrCode.java               # 已有 — 增 3002 RELATIONSHIP_NOT_FOUND / 3003 INTIMACY_CONCURRENT_UPDATE
│   ├── RelationshipEntity.java             # 新增（@IdClass 复合 PK + @Version）
│   ├── RelationshipRepository.java         # 新增
│   ├── RelationshipFindOrCreateService.java # 新增 — 唯一懒创建入口；事务 + 并发安全
│   ├── RelationshipFetchService.java       # 新增 — 编排 GET /me：findOrCreate + recordLogin + 拼 DTO
│   ├── intimacy/
│   │   ├── IntimacyEvent.java              # record (type, payload)；internal 不出 -api
│   │   ├── IntimacyCalculator.java
│   │   ├── IntimacyRecordService.java      # 累加 + 阶段切换 + 事件（按动作命名）
│   │   ├── IntimacyLogEntity.java
│   │   ├── IntimacyLogRepository.java
│   │   ├── DailyBehaviorCounter.java       # Redis
│   │   ├── ConsecutiveLoginService.java    # Redis
│   │   └── ai/
│   │       └── ConversationQualityEvaluator.java
│   ├── plotrule/
│   │   ├── PlotMilestoneRule.java          # 接口 — evaluate(MessageContext) → Optional<IntimacyEvent>
│   │   ├── PlotMilestoneEngine.java        # 注册器（Spring 自动收集所有 Rule Bean）
│   │   ├── RelationshipMilestoneEntity.java
│   │   ├── RelationshipMilestoneRepository.java
│   │   ├── DeepNightChatRule.java          # 连续 3 晚 22-02 +50（PlotMilestoneRule 实现）
│   │   └── FirstHonestShareRule.java       # 情感关键词首次命中 +30（PlotMilestoneRule 实现）
│   └── stage/
│       ├── StageDefinition.java            # enum 5 档（名/序号）；阈值来自 ConfigurationProperties
│       ├── StageTransitionDetectService.java   # 检测阶段切换 + 发事件
│       └── StageOverrideQueryService.java  # 读 persona_config.stage_overrides → 拼 prompt 段
├── api/
│   └── CharacterApiImpl.java               # 已有 — 扩充 4 个 relationship 方法（薄委托 internal Service）
├── web/
│   └── RelationshipController.java         # GET /api/relationships/me — 薄壳，调 CharacterApi.fetchMyRelationship
└── event/
    ├── MessagePersistedListener.java       # 订阅 chat-api 的 MessagePersistedEvent
    └── StageTransitionStoryListener.java   # 订阅自家 StageTransitionEvent → 拼 story_message（不加分；通过 IntimacyChangedEvent 元数据或单独事件下发给 chat-core WS）
```

### 3.3 sanyan-chat-core（仅 3 处接入）

```
internal/PromptBuilder.java         # build 多 stagePromptSegment 参数
internal/AiService.java             # 调 CharacterApi.getStagePromptSegment 拼好传入
ws/ChatWebSocketHandler.java        # 监听 IntimacyChangedEvent / StageTransitionEvent → WS 推送
```

PromptBuilder 新签名：
```java
public List<Map<String, String>> build(
    String characterPrompt,
    String stagePromptSegment,        // 新增 — caller 拼好（可为 null/blank）
    MemoryContext memoryContext,
    List<MessageEntity> recentMessages);
```

拼装顺序（强约束）：
1. system: `characterPrompt`（人设 + 当前时间等基底）
2. system: `stagePromptSegment` —— 非 blank 时
3. system: 「她对你的记忆：\n」+ `memoryContext.text()` —— 非 blank 时
4. 短期上下文消息（尾部 32 条）

### 3.4 跨模块边界守则 + pom 依赖

- character-core **订阅** chat-api 的 `MessagePersistedEvent`（已存在，Plan 2 N3 加的）
  → **新增依赖**：`sanyan-character-core/pom.xml` 加 `<dependency>sanyan-chat-api</dependency>`
- chat-core **调用** character-api 的 `CharacterApi.getStagePromptSegment`（拿字符串，不引入 Relationship 类型依赖）
  → chat-core 已经依赖 character-api（Plan 2 注入 characterPrompt 时已加），**无需新增**
- chat-core **订阅** character-api 的 `IntimacyChangedEvent / StageTransitionEvent` → WS 推送
  → 依赖关系同上
- 双向依赖通过 event + API 解耦，符合 `java-backend.md §4` 跨模块通信规则
- Maven Enforcer `banned-dependencies` 守护：禁止 `-core` 互依

### 3.5 错误码与配套 docs 同步

- 新增错误码沿用 character 域既有区间 **3000-3999**（当前已占 3001 `CHARACTER_NOT_FOUND`）
  - **3002** `RELATIONSHIP_NOT_FOUND` —— 调用 `getById` 类强约束方法但 relationship 不存在
  - **3003** `INTIMACY_CONCURRENT_UPDATE` —— 乐观锁 retry 3 次仍失败
- **必须同步更新** `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`：
  - "CharacterErrCode（3000-3999）" 表内增加 3002 / 3003 两行
  - 历史变更附一行：「2026-05-18 Plan 3：character 域新增 3002 / 3003」
- 启动时 `ErrCodeConflictDetector` 自动扫描；冲突即启动失败

---

## 4 · 数据流

### 4.1 链路 A：用户发消息 → 涨分 → 推送 → UI 刷新

1. **[App]** 用户发消息 → WS frame
2. **[chat-core · AiService]**
   1. `characterApi.getStagePromptSegment(uid, cid)` → "当前关系：暧昧。称呼：宝。语调：撒娇..."（内部走 RelationshipFindOrCreateService）
   2. `PromptBuilder.build(charPrompt, stagePromptSegment, memoryContext, recent)`
   3. LLM 调用 → 流式回写 WS
   4. `MessageService.persist(...)` → DB COMMIT → `publishEvent(MessagePersistedEvent)`
3. **[character-core · MessagePersistedListener]**（`@TransactionalEventListener(phase = AFTER_COMMIT)` + `@Async`）
   - `intimacyRecordService.recordEvent(MESSAGE_SENT)` —— +1 ~ +10（受每日封顶限）
   - `plotEngine.evaluate(context)` → 0..N rule 命中 → `intimacyRecordService.recordEvent(PLOT_MILESTONE, +30~+100)`
   - 累计每 10 条用户消息 → `evaluator.score()` → `intimacyRecordService.recordEvent(AI_QUALITY_BONUS, 0~20)`
4. **[character-core · IntimacyRecordService.recordEvent]**
   1. `calculator.compute(event)` → delta
   2. `UPDATE relationships SET intimacy_score+=delta WHERE ... AND version=?`（乐观锁，retry 3 次）
   3. `INSERT INTO intimacy_logs ...`
   4. `stageTransitionDetectService.maybeTransition(rel, newScore)`
      - 跨阶段？→ UPDATE current_stage + `publishEvent(StageTransitionEvent)`
   5. `publishEvent(IntimacyChangedEvent)`
5. **[chat-core · ChatWebSocketHandler 事件监听]**
   - `IntimacyChangedEvent` → `sendToUser(uid, {type:"intimacy_update", score, stage, delta, reason})`
   - `StageTransitionEvent` → `sendToUser(uid, {type:"stage_transition", from, to, story_message})`
6. **[App · chat_controller]**
   - `intimacy_update` → `Rx<Relationship>` 更新 → 进度条 Obx 自动刷新
   - `stage_transition` → 触发 `StageTransitionDialog` 动画 + 文案

### 4.2 链路 B：首次登录涨分（轻量）

触发点：`GET /api/relationships/me`（前端进聊天页时调；接口内**日级幂等**——同日多次调用只首次涨分）

**Controller → ApiImpl → RelationshipFetchService.fetchMyRelationship(uid, cid)** 内部编排：

1. `relationshipFindOrCreateService.findOrCreate(uid, cid)`（唯一懒创建入口；当日首次访问时建 relationship 行）
2. `consecutiveLoginService.recordLogin(userId)` 比对 Redis last_date
   - 今天已记 → return streak（不涨分）
   - 今天首次 → streak++（或重置为 1） + 写 Redis → `intimacyRecordService.recordEvent(DAILY_LOGIN, streak)` —— +10 + streak×5（封顶 +60）
3. 拼 `RelationshipDto` 返回（intimacy_score / current_stage / next_threshold / percent）

### 4.3 关键守则

- **事务边界**：listener 用 `AFTER_COMMIT + @Async`，亲密度更新不阻塞消息回写
- **乐观锁**：`RelationshipEntity` 加 `@Version Integer version`，并发涨分 retry 3 次
- **每日封顶**：`IntimacyCalculator` 在 compute 阶段 query `DailyBehaviorCounter`；超过 50 时本次 delta=0（仍写 log，reason=`CAPPED` 便于审计）
- **失败降级**：character-core listener 失败不影响消息持久化（已 COMMIT）；记 ERROR 日志即可，不补偿

---

## 5 · 5 阶段定义 + 涨分来源 + 参数化

### 5.1 阶段定义

| 序号 | 名称 | 阈值区间 |
|------|------|---------|
| 0 | 陌生人 STRANGER | 0 — 100 |
| 1 | 朋友 FRIEND | 100 — 300 |
| 2 | 暧昧 AMBIGUOUS | 300 — 600 |
| 3 | 恋人 LOVER | 600 — 1000 |
| 4 | 老夫老妻 OLD_COUPLE | 1000+ |

跨度递增，避免"7 天到顶"。dogfood 后调比例。

### 5.2 涨分来源（D 方案）

| 来源 | 触发 | delta | 封顶 |
|------|------|-------|------|
| MESSAGE_SENT | 每条用户消息 | +1 | 每日 50 |
| DAILY_LOGIN | 每日首次进聊天页 | +10 + streak × 5 | streak ≤ 10 → 最高 +60 |
| PLOT:deep_night_chat | 连续 3 晚 22-02 都有聊 | +50 | 一次性（milestones 去重） |
| PLOT:first_honest_share | 首次深度分享（情感关键词） | +30 | 一次性 |
| STORY:stage_entry_N | 每次首次进入新阶段（由 StageTransitionStoryListener 处理，不走 plotEngine） | 0（仅剧情演出 / WS story_message 推送） | 每阶段一次性（relationship_milestones 去重） |
| AI_QUALITY_BONUS | 每 10 条用户消息 → V4-Flash 评估 | +0 ~ +20 | 单次 ≤ 20 |

### 5.3 参数化（application.yml）

`StageDefinition` enum 只放阶段名/序号（编译期常量），所有数值从 `@ConfigurationProperties("sanyan.intimacy")` 加载：

```yaml
sanyan:
  intimacy:
    stages:
      strangerEnd: 100
      friendEnd: 300
      ambiguousEnd: 600
      loverEnd: 1000
    delta:
      messageSent: 1
      messageDailyCap: 50
      dailyLogin: 10           # 基础分（每日首次登录无条件给）
      streakBonusPerDay: 5     # 每多 1 天连续登录额外加的分
      streakBonusCap: 50       # streak 部分的封顶（不含基础分）；整体最高 = dailyLogin + streakBonusCap = +60
    rules:
      deepNightChat: 50
      firstHonestShare: 30
    ai:
      qualityMaxScore: 20
      triggerEveryNMessages: 10
```

### 5.4 AI 评估器 prompt（草案）

```
评估以下用户与小婉的对话片段质量（0-20 分）。
评分维度：
- 用户分享深度（透露真实信息、情感、想法）
- 对话连贯性（不是单字"嗯"/"哦"敷衍）
- 互动深度（双向，不是单方面输出）
返回 JSON：{"score": <0-20>, "reason": "<10字内>"}
对话：{{messages}}
```

---

## 6 · 前端组件

### 6.1 sanyan_chat 包内新增

```
business_packages/sanyan_chat/lib/src/
├── api/
│   ├── sanyan_chat_api.dart                # 修改 — 加 fetchMyRelationship 方法（聚合入口）
│   ├── models/relationship.dart            # 新增 — 跨接口复用的领域模型
│   └── req/relationship_req.dart           # 新增 — RelationshipReq + RelationshipData
└── chat/
    ├── widgets/
    │   ├── intimacy_progress_bar.dart      # 顶部进度条
    │   └── stage_transition_dialog.dart    # 阶段切换庆祝弹窗
    └── chat_controller.dart                # 修改 — 监听 ws intimacy_update / stage_transition
```

> 命名约束：按 `flutter-business-layer.md §6`，请求/响应类同文件放（`RelationshipReq` + `RelationshipData`），目录用单数 `req/`；
> `Relationship` 模型本身跨接口（HTTP `/me` 响应 + ws `intimacy_update` 推送都用），所以放 `models/` 而非 `req/`。

### 6.2 IntimacyProgressBar 行为

- 顶部固定条：左 = 当前阶段名（"暧昧"），右 = `score / next_threshold`，中部进度填充（粉紫渐变，按阶段调色）
- 点击展开详情弹窗：上次涨分原因、最近 10 条 intimacy_logs、距下一阶段还差 X 分
- ws `intimacy_update` 触发动画（数字滚动 + 进度填充动画 ~1.5s）

### 6.3 StageTransitionDialog 行为

- 收到 ws `stage_transition` 帧 → 全屏遮罩 + 卡片动画
- 文案从 ws 帧 `story_message` 字段读（后端拼好，如"小婉第一次叫你宝贝……"）
- 用户点击 / 3 秒自动关闭

---

## 7 · 测试策略

严格 TDD：每个 Service / Listener / Rule 必须先有失败的 *Test 才能写实现。

### 7.1 单测（*Test, Mockito）

- **intimacy/**：`IntimacyCalculatorTest`（6 种 type × 边界） / `IntimacyRecordServiceTest`（累加 / 乐观锁 retry / 跨阶段 / cap 命中 delta=0 但仍写 log）
- **stage/**：`StageDefinitionTest`（0/99/100/300/600/1000 边界） / `StageTransitionDetectServiceTest` / `StageOverrideQueryServiceTest`（persona_config 解析）
- **plotrule/**：每个 rule 一个 Test（覆盖触发 / 不触发 / 去重） + `PlotMilestoneEngineTest`
- **intimacy/ai/**：`ConversationQualityEvaluatorTest`（LLM fake JSON 解析）
- **event/**：`MessagePersistedListenerTest`（链路 mock） / `StageTransitionStoryListenerTest`
- **internal/**：`RelationshipFindOrCreateServiceTest`（懒创建并发） / `RelationshipFetchServiceTest`（编排 mock）

### 7.2 集成测试（*IT）

- **Testcontainers Redis**：`DailyBehaviorCounterIT`（INCR / TTL / 封顶） / `ConsecutiveLoginServiceIT`（连续 / 中断 / 重复）
- **@DataJpaTest (H2)**：`RelationshipRepositoryIT`（复合 PK + @Version） / `IntimacyLogRepositoryIT`（时间倒序） / `RelationshipMilestoneRepositoryIT`
- **@WebMvcTest**：`RelationshipControllerIT`（GET /me 返回 + 触发 DAILY_LOGIN）
- **@SpringBootTest 端到端 final gate**：`MessageFlowE2EIT` —— 发消息 → AFTER_COMMIT → 涨分 → IntimacyChangedEvent → WS frame 推送 → DB relationships 状态正确

### 7.3 前端测试

- `intimacy_progress_bar_test.dart`（5 stage × 进度百分比 × 展开详情）
- `stage_transition_dialog_test.dart`（动画 + 文案）
- `chat_controller_intimacy_test.dart`（收到 ws frame 后 Rx 更新）
- `relationship_req_test.dart`（GET /me 解析）
- **更新 `test/sanyan_chat_suite.dart`**：把以上 4 个测试 import 进聚合入口（按 `flutter-business-layer.md §10` 强制要求）

### 7.x Object Mother（强制 · 按 `java-backend-business-layer.md §5.2`）

新建 Entity 必须有对应 Fixture，放 `sanyan-character-core/src/test/java/com/sanyan/character/internal/fixtures/`：

- `RelationshipTestFixtures.java`：`validRelationship(uid, cid)` / `relationshipWithScore(score)` / `relationshipAtStage(stage)`
- `IntimacyLogTestFixtures.java`：`validLog(uid, cid, reason, delta)`
- `RelationshipMilestoneTestFixtures.java`：`validMilestone(uid, cid, ruleId)`

已有 `AiCharacterTestFixtures` 增 `withPersonaConfig(...)` 工厂方法支持 Plan 3 字段。

### 7.4 测试粒度（按 CLAUDE.md「Superpowers Task 测试粒度规范」）

- **每个 Task**：跑该模块测试 + `mvn -pl <module> test` / `flutter test <package>`；TDD RED → GREEN → REFACTOR
- **每个 Phase checkpoint**（约 3-5 个 task 一组）：跑一次 server 全量 + flutter 全量
- **final gate**（merge plan3 → master 前）：`mvn verify` + `yes | fvm flutter test` 全量

### 7.5 覆盖率目标

- character-core/intimacy 与 stage 包 ≥ 85%
- plotrule ≥ 80%
- 其余按 java-backend.md 规则 70%+

---

## 8 · dogfood 验证 + 数值调参

### 8.1 7 天 dogfood checklist

每天写一篇 `docs/dogfood/plan-3-day-N.md`：

- **速度感** — 7 天内能否推到朋友？跨阶段是惊喜还是平淡？一天聊 20-30 句涨多少分？
- **剧情演出** — DeepNightChat 命中那次卡片是否打动？StageTransitionDialog 动画文案够不够"戳"？
- **透明性** — 涨分理由清晰吗？进度条详情里"上次涨分原因"够不够人话？AI 评估 reason 是否令人信服？
- **称呼语调** — 小婉从"你"切到"宝"是自然还是突兀？暧昧阶段"撒娇 / 试探"语调真撒娇吗？

### 8.2 数值调参（只改 application.yml，不动代码）

常见方向：
- 太慢 → 提高 messageSent / dailyLogin / streakBonus；降低 stage 阈值
- 太快 → 降低 cap；抬高 stage 阈值（尤其 friend → ambiguous 这段）
- 剧情命中率低 → 检查 plotrule 触发条件（如 deepNightChat 时间窗）
- AI 评估漂 → 调 evaluator prompt 或降低 qualityMaxScore

调参流程：
1. 看 `intimacy_logs` 近 24h：`SELECT reason, SUM(delta), COUNT(*) FROM intimacy_logs WHERE user_id=<me> GROUP BY reason`
2. 改 application.yml → 重启或 `@RefreshScope` reload
3. 记到 day-N.md：「day3 改 messageSent 1→2，理由：每天 30 句只涨 30 太少」

### 8.3 配套观测（SQL 看板）

每天看一眼：
```sql
-- 当前关系
SELECT * FROM relationships WHERE user_id=<me>;
-- 涨分明细
SELECT reason, delta, new_score, created_at FROM intimacy_logs
  WHERE user_id=<me> ORDER BY created_at DESC LIMIT 50;
-- 已触发 milestone
SELECT * FROM relationship_milestones WHERE user_id=<me>;
```
```bash
# streak
redis-cli GET streak:user:<me>
```

App 侧（dev 模式）：进度条点开 → 显示完整 intimacy_logs 列表（prod 简化）。

### 8.4 交付定义（DoD）

- 所有自动化测试通过（`mvn verify` + `yes | fvm flutter test` 全绿）
- 7 天 dogfood 完成 + 每天一篇 day-N.md 记录
- 数值调参至少一轮（基于 dogfood 反馈）
- 切换阶段时小婉称呼/语调感受"自然"——主观验收
- plan-3.md 末尾追加一段「dogfood 反馈 + 最终数值」

---

## Task 列表

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` to implement task-by-task. Each task has 5 bite-sized steps. TDD 铁律：Step 1 写失败测试 → Step 2 确认红 → Step 3 写最小实现 → Step 4 确认绿 → Step 5 commit。

### 文件结构总览

**Server / 新建（character-core）：**

| 路径 | 责任 |
|---|---|
| `bootstrap/src/main/resources/db/migration/V2__ai_character_add_persona.sql` | ALTER 加 base_prompt + persona_config |
| `bootstrap/src/main/resources/db/migration/V3__create_relationships.sql` | relationships 表 |
| `bootstrap/src/main/resources/db/migration/V4__create_intimacy_logs.sql` | intimacy_logs 表 |
| `bootstrap/src/main/resources/db/migration/V5__create_relationship_milestones.sql` | milestones 表 |
| `bootstrap/src/main/resources/db/migration/V6__seed_xiaowan_persona.sql` | UPDATE 小婉 persona |
| `business_packages/sanyan-character-core/pom.xml` | **修改**：加 chat-api 依赖 |
| `business_packages/sanyan-character-core/src/main/resources/application.yml`（或全局） | sanyan.intimacy.* 配置 |
| `internal/AiCharacterEntity.java` | **修改**：加 basePrompt + personaConfig 字段 |
| `internal/CharacterErrCode.java` | **修改**：加 3002 / 3003 |
| `internal/RelationshipEntity.java` + `RelationshipId.java` | 复合 PK + @Version |
| `internal/RelationshipRepository.java` | Spring Data JPA |
| `internal/RelationshipFindOrCreateService.java` | 唯一懒创建入口 |
| `internal/RelationshipFetchService.java` | GET /me 编排 |
| `internal/intimacy/IntimacyProperties.java` | @ConfigurationProperties("sanyan.intimacy") |
| `internal/intimacy/IntimacyEvent.java` | record (type, payload, characterPayload) |
| `internal/intimacy/IntimacyCalculator.java` | type → delta + 每日封顶查询 |
| `internal/intimacy/IntimacyRecordService.java` | recordEvent 累加 + 阶段切换 + 事件 |
| `internal/intimacy/IntimacyLogEntity.java` + Repo | 审计 |
| `internal/intimacy/DailyBehaviorCounter.java` | Redis INCR + TTL |
| `internal/intimacy/ConsecutiveLoginService.java` | Redis streak |
| `internal/intimacy/ai/ConversationQualityEvaluator.java` | 调 V4-Flash 评分 |
| `internal/plotrule/PlotMilestoneRule.java`（接口） | evaluate(MessageContext) |
| `internal/plotrule/PlotMilestoneEngine.java` | Spring 自动收集 Rule Bean |
| `internal/plotrule/RelationshipMilestoneEntity.java` + Repo | 去重 |
| `internal/plotrule/DeepNightChatRule.java` | 连续 3 晚 22-02 +50 |
| `internal/plotrule/FirstHonestShareRule.java` | 情感关键词首次 +30 |
| `internal/stage/StageDefinition.java` | enum 5 档（名/序号） |
| `internal/stage/StageTransitionDetectService.java` | maybeTransition + 发事件 |
| `internal/stage/StageOverrideQueryService.java` | 拼 stage prompt 文本 |
| `api/CharacterApiImpl.java` | **修改**：扩 4 个方法 |
| `web/RelationshipController.java` | GET /api/relationships/me |
| `event/MessagePersistedListener.java` | AFTER_COMMIT + @Async |
| `event/StageTransitionStoryListener.java` | 阶段切换剧情演出 |

**Server / 新建（character-api）：**

| 路径 | 责任 |
|---|---|
| `business_packages/sanyan-character-api/src/main/java/com/sanyan/character/CharacterApi.java` | **修改**：扩 4 个方法 |
| `.../dto/RelationshipDto.java` | API 跨模块 DTO |
| `.../event/IntimacyChangedEvent.java` | record |
| `.../event/StageTransitionEvent.java` | record |

**Server / 修改（chat-core）：**

| 路径 | 责任 |
|---|---|
| `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/PromptBuilder.java` | **修改**：build 加 stagePromptSegment 参数 |
| `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/AiService.java` | **修改**：调 characterApi.getStagePromptSegment |
| `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java` | **修改**：监听事件 + WS 推送 |

**Server / 测试：** 每个新组件对应 *Test (Mockito) 或 *IT；Fixtures 见各 task。

**Server / docs：** `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md` 加 3002 / 3003 + 历史变更。

**App / sanyan_chat：**

| 路径 | 责任 |
|---|---|
| `business_packages/sanyan_chat/lib/src/api/models/relationship.dart` | 跨接口模型 |
| `business_packages/sanyan_chat/lib/src/api/req/relationship_req.dart` | RelationshipReq + RelationshipData |
| `business_packages/sanyan_chat/lib/src/api/sanyan_chat_api.dart` | **修改**：加 fetchMyRelationship |
| `business_packages/sanyan_chat/lib/src/chat/widgets/intimacy_progress_bar.dart` | 顶部条 widget |
| `business_packages/sanyan_chat/lib/src/chat/widgets/stage_transition_dialog.dart` | 弹窗 widget |
| `business_packages/sanyan_chat/lib/src/chat/chat_controller.dart` | **修改**：监听 ws frame |
| `business_packages/sanyan_chat/test/sanyan_chat_suite.dart` | **修改**：聚合新 widget test |

---

### Phase A · 数据库 migration + 错误码登记

#### Task A1: V2 migration — ai_character 加 base_prompt + persona_config

**Files:**
- Create: `server/bootstrap/src/main/resources/db/migration/V2__ai_character_add_persona.sql`
- Test: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/AiCharacterPersonaSchemaIT.java`

- [ ] **Step 1: Write the failing test**

```java
package com.sanyan.character.internal;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.jdbc.core.JdbcTemplate;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
class AiCharacterPersonaSchemaIT {
    @Autowired JdbcTemplate jdbc;

    @Test
    void v2_should_add_base_prompt_and_persona_config_columns() {
        List<Map<String, Object>> cols = jdbc.queryForList(
            "SELECT column_name, data_type FROM information_schema.columns " +
            "WHERE table_name='ai_character' AND column_name IN ('base_prompt','persona_config')");
        assertThat(cols).hasSize(2);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn -pl business_packages/sanyan-character-core -Dtest=AiCharacterPersonaSchemaIT test
```
Expected: FAIL with 0 columns matched (V2 migration not created yet).

- [ ] **Step 3: Write minimal implementation**

```sql
-- V2__ai_character_add_persona.sql
ALTER TABLE ai_character
    ADD COLUMN base_prompt    TEXT  NOT NULL DEFAULT '',
    ADD COLUMN persona_config JSONB NOT NULL DEFAULT '{}'::jsonb;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn -pl business_packages/sanyan-character-core -Dtest=AiCharacterPersonaSchemaIT test
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/main/resources/db/migration/V2__ai_character_add_persona.sql \
        server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/AiCharacterPersonaSchemaIT.java
git commit -m "feat(db): V2 ai_character 加 base_prompt + persona_config 列"
```

---

#### Task A2: V3 migration — relationships 表

**Files:**
- Create: `server/bootstrap/src/main/resources/db/migration/V3__create_relationships.sql`
- Test: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/RelationshipsSchemaIT.java`

- [ ] **Step 1: Write the failing test**

```java
@DataJpaTest
class RelationshipsSchemaIT {
    @Autowired JdbcTemplate jdbc;

    @Test
    void v3_should_create_relationships_with_composite_pk_and_version() {
        Map<String, Object> tbl = jdbc.queryForMap(
            "SELECT table_name FROM information_schema.tables WHERE table_name='relationships'");
        assertThat(tbl).isNotEmpty();

        List<String> pk = jdbc.queryForList(
            "SELECT column_name FROM information_schema.key_column_usage " +
            "WHERE table_name='relationships' ORDER BY ordinal_position", String.class);
        assertThat(pk).containsExactly("user_id", "character_id");

        Integer version = jdbc.queryForObject(
            "SELECT COUNT(*) FROM information_schema.columns " +
            "WHERE table_name='relationships' AND column_name='version'", Integer.class);
        assertThat(version).isEqualTo(1);
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
mvn -pl business_packages/sanyan-character-core -Dtest=RelationshipsSchemaIT test
```
Expected: FAIL（表不存在）.

- [ ] **Step 3: Write minimal implementation**

```sql
-- V3__create_relationships.sql
CREATE TABLE relationships (
    user_id         BIGINT   NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    character_id    BIGINT   NOT NULL REFERENCES ai_character(id) ON DELETE RESTRICT,
    intimacy_score  INT      NOT NULL DEFAULT 0,
    current_stage   SMALLINT NOT NULL DEFAULT 0,
    version         INT      NOT NULL DEFAULT 0,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, character_id)
);
```

- [ ] **Step 4: Run to verify PASS**

```bash
mvn -pl business_packages/sanyan-character-core -Dtest=RelationshipsSchemaIT test
```

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/main/resources/db/migration/V3__create_relationships.sql \
        server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/RelationshipsSchemaIT.java
git commit -m "feat(db): V3 创建 relationships 表（复合 PK + 乐观锁 version）"
```

---

#### Task A3: V4 migration — intimacy_logs 表

**Files:**
- Create: `server/bootstrap/src/main/resources/db/migration/V4__create_intimacy_logs.sql`
- Test: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/intimacy/IntimacyLogsSchemaIT.java`

- [ ] **Step 1: Write the failing test**

```java
@DataJpaTest
class IntimacyLogsSchemaIT {
    @Autowired JdbcTemplate jdbc;

    @Test
    void v4_should_create_intimacy_logs_with_descending_index() {
        Map<String, Object> tbl = jdbc.queryForMap(
            "SELECT table_name FROM information_schema.tables WHERE table_name='intimacy_logs'");
        assertThat(tbl).isNotEmpty();

        Integer idx = jdbc.queryForObject(
            "SELECT COUNT(*) FROM pg_indexes WHERE tablename='intimacy_logs' " +
            "AND indexname='idx_intimacy_logs_user_char_time'", Integer.class);
        assertThat(idx).isEqualTo(1);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

```bash
mvn -pl business_packages/sanyan-character-core -Dtest=IntimacyLogsSchemaIT test
```

- [ ] **Step 3: Write minimal implementation**

```sql
-- V4__create_intimacy_logs.sql
CREATE TABLE intimacy_logs (
    id            BIGSERIAL PRIMARY KEY,
    user_id       BIGINT      NOT NULL,
    character_id  BIGINT      NOT NULL,
    delta         INT         NOT NULL,
    reason        VARCHAR(60) NOT NULL,
    metadata      JSONB,
    new_score     INT         NOT NULL,
    new_stage     SMALLINT    NOT NULL,
    created_at    TIMESTAMP   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_intimacy_logs_user_char_time
    ON intimacy_logs (user_id, character_id, created_at DESC);
```

- [ ] **Step 4: Run to verify PASS** (same `mvn` command)

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/main/resources/db/migration/V4__create_intimacy_logs.sql \
        server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/intimacy/IntimacyLogsSchemaIT.java
git commit -m "feat(db): V4 创建 intimacy_logs 审计表 + 时间倒序索引"
```

---

#### Task A4: V5 migration — relationship_milestones 表

**Files:**
- Create: `server/bootstrap/src/main/resources/db/migration/V5__create_relationship_milestones.sql`
- Test: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/plotrule/RelationshipMilestonesSchemaIT.java`

- [ ] **Step 1: Write the failing test**

```java
@DataJpaTest
class RelationshipMilestonesSchemaIT {
    @Autowired JdbcTemplate jdbc;

    @Test
    void v5_should_create_milestones_with_3col_pk() {
        List<String> pk = jdbc.queryForList(
            "SELECT column_name FROM information_schema.key_column_usage " +
            "WHERE table_name='relationship_milestones' ORDER BY ordinal_position", String.class);
        assertThat(pk).containsExactly("user_id", "character_id", "rule_id");
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```sql
-- V5__create_relationship_milestones.sql
CREATE TABLE relationship_milestones (
    user_id       BIGINT      NOT NULL,
    character_id  BIGINT      NOT NULL,
    rule_id       VARCHAR(60) NOT NULL,
    triggered_at  TIMESTAMP   NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, character_id, rule_id)
);
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/main/resources/db/migration/V5__create_relationship_milestones.sql \
        server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/plotrule/RelationshipMilestonesSchemaIT.java
git commit -m "feat(db): V5 创建 relationship_milestones 表（3 列复合 PK 防重）"
```

---

#### Task A5: V6 migration — seed 小婉 persona + ERROR_CODE_REGISTRY 同步

**Files:**
- Create: `server/bootstrap/src/main/resources/db/migration/V6__seed_xiaowan_persona.sql`
- Modify: `server/foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`
- Test: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/XiaowanPersonaSeedIT.java`

- [ ] **Step 1: Write the failing test**

```java
@DataJpaTest
class XiaowanPersonaSeedIT {
    @Autowired JdbcTemplate jdbc;

    @Test
    void v6_should_seed_xiaowan_persona_config_with_5_stage_overrides() {
        String json = jdbc.queryForObject(
            "SELECT persona_config::text FROM ai_character WHERE name='小婉'", String.class);
        assertThat(json).contains("stage_overrides");
        for (int i = 0; i <= 4; i++) {
            assertThat(json).contains("\"" + i + "\"");
        }
    }
}
```

- [ ] **Step 2: Run to verify FAIL** (persona_config 是空 `{}`)

- [ ] **Step 3: Write minimal implementation**

```sql
-- V6__seed_xiaowan_persona.sql
UPDATE ai_character
SET base_prompt = '你是小婉。22 岁，性格温暖、有点害羞、爱开玩笑。和用户的关系会随聊天加深而变化。回复尽量自然像朋友/恋人聊天，不要冗长说教。',
    persona_config = '{
      "stage_overrides": {
        "0": { "address": "你",                       "tone_hint": "礼貌、有距离感、偶尔害羞",  "topics_unlock": [] },
        "1": { "address": "{nickname}",               "tone_hint": "自然、放松、会开玩笑",      "topics_unlock": ["吐槽","生活琐事"] },
        "2": { "address": "笨蛋",                     "tone_hint": "撒娇、试探、害羞",          "topics_unlock": ["想见面","未来设想","暗示性话题"] },
        "3": { "address": "宝贝",                     "tone_hint": "撒娇、依赖、占有欲",        "topics_unlock": ["纪念日","思念","情感深度"] },
        "4": { "address": "老公",                     "tone_hint": "生活感、小吵小闹、深度依赖", "topics_unlock": ["小日常","未来生活"] }
      }
    }'::jsonb
WHERE name = '小婉';
```

修改 `ERROR_CODE_REGISTRY.md` —— 在 CharacterErrCode（3000-3999）表下加：

```
| 3002 | `RELATIONSHIP_NOT_FOUND` | 关系不存在 |
| 3003 | `INTIMACY_CONCURRENT_UPDATE` | 亲密度并发更新失败 |
```

并在"历史变更"末尾加：

```
| 2026-05-18 | Plan 3：character 域新增 3002 / 3003 |
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/main/resources/db/migration/V6__seed_xiaowan_persona.sql \
        server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/XiaowanPersonaSeedIT.java \
        server/foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md
git commit -m "feat(db): V6 seed 小婉 5 阶段 persona_config + ERROR_CODE_REGISTRY 同步 3002/3003"
```

---

### Phase B · Entity + Repository + Fixture

#### Task B1: AiCharacterEntity 加 basePrompt + personaConfig 字段

**Files:**
- Modify: `server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/internal/AiCharacterEntity.java`
- Modify: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/AiCharacterTestFixtures.java`
- Test: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/AiCharacterEntityFieldIT.java`

- [ ] **Step 1: Write the failing test**

```java
@DataJpaTest
class AiCharacterEntityFieldIT {
    @Autowired TestEntityManager em;

    @Test
    void should_persist_base_prompt_and_persona_config_jsonb() {
        AiCharacterEntity ch = AiCharacterTestFixtures.withPersonaConfig(
            "你是测试角色", "{\"stage_overrides\":{\"0\":{}}}");
        Long id = em.persistAndFlush(ch).getId();
        em.clear();

        AiCharacterEntity reloaded = em.find(AiCharacterEntity.class, id);
        assertThat(reloaded.getBasePrompt()).isEqualTo("你是测试角色");
        assertThat(reloaded.getPersonaConfig()).contains("stage_overrides");
    }
}
```

- [ ] **Step 2: Run to verify FAIL** (字段还没有)

- [ ] **Step 3: Write minimal implementation**

```java
// AiCharacterEntity.java
@Data
@Entity
@Table(name = "ai_character")
public class AiCharacterEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, length = 50)
    private String name;
    private String avatar;
    @CreationTimestamp
    private LocalDateTime createdAt;

    @Column(name = "base_prompt", nullable = false, columnDefinition = "text")
    private String basePrompt = "";

    @Column(name = "persona_config", nullable = false, columnDefinition = "jsonb")
    @org.hibernate.annotations.JdbcTypeCode(org.hibernate.type.SqlTypes.JSON)
    private String personaConfig = "{}";
}
```

```java
// AiCharacterTestFixtures.java —— 加工厂方法
public static AiCharacterEntity withPersonaConfig(String basePrompt, String personaConfigJson) {
    AiCharacterEntity e = validCharacter();
    e.setBasePrompt(basePrompt);
    e.setPersonaConfig(personaConfigJson);
    return e;
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): AiCharacterEntity 加 basePrompt + personaConfig 字段"
```

---

#### Task B2: RelationshipEntity + RelationshipId + Repository + Fixture

**Files:**
- Create: `server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/internal/RelationshipEntity.java`
- Create: `server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/internal/RelationshipId.java`
- Create: `server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/internal/RelationshipRepository.java`
- Create: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/fixtures/RelationshipTestFixtures.java`
- Test: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/RelationshipRepositoryIT.java`

- [ ] **Step 1: Write the failing test**

```java
@DataJpaTest
class RelationshipRepositoryIT {
    @Autowired RelationshipRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void save_and_findById_should_roundtrip_with_composite_pk() {
        RelationshipEntity rel = RelationshipTestFixtures.validRelationship(1L, 1L);
        repo.save(rel);
        em.flush(); em.clear();

        RelationshipEntity loaded = repo.findById(new RelationshipId(1L, 1L)).orElseThrow();
        assertThat(loaded.getIntimacyScore()).isZero();
        assertThat(loaded.getCurrentStage()).isZero();
        assertThat(loaded.getVersion()).isZero();
    }

    @Test
    void version_should_increment_on_update() {
        RelationshipEntity rel = repo.save(RelationshipTestFixtures.validRelationship(1L, 1L));
        em.flush();
        rel.setIntimacyScore(10);
        repo.save(rel);
        em.flush();
        assertThat(rel.getVersion()).isEqualTo(1);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
// RelationshipId.java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class RelationshipId implements Serializable {
    private Long userId;
    private Long characterId;
}
```

```java
// RelationshipEntity.java
@Data
@Entity
@Table(name = "relationships")
@IdClass(RelationshipId.class)
public class RelationshipEntity {
    @Id @Column(name = "user_id")      private Long userId;
    @Id @Column(name = "character_id") private Long characterId;

    @Column(name = "intimacy_score", nullable = false) private Integer intimacyScore = 0;
    @Column(name = "current_stage",  nullable = false) private Short  currentStage   = 0;
    @Version @Column(nullable = false)                  private Integer version       = 0;

    @CreationTimestamp @Column(name = "created_at") private LocalDateTime createdAt;
    @UpdateTimestamp   @Column(name = "updated_at") private LocalDateTime updatedAt;
}
```

```java
// RelationshipRepository.java
public interface RelationshipRepository extends JpaRepository<RelationshipEntity, RelationshipId> {
    Optional<RelationshipEntity> findByUserIdAndCharacterId(Long userId, Long characterId);
}
```

```java
// RelationshipTestFixtures.java
public final class RelationshipTestFixtures {
    private RelationshipTestFixtures() {}

    public static RelationshipEntity validRelationship(Long userId, Long characterId) {
        RelationshipEntity r = new RelationshipEntity();
        r.setUserId(userId);
        r.setCharacterId(characterId);
        return r;
    }

    public static RelationshipEntity relationshipWithScore(Long userId, Long characterId, int score) {
        RelationshipEntity r = validRelationship(userId, characterId);
        r.setIntimacyScore(score);
        return r;
    }

    public static RelationshipEntity relationshipAtStage(Long userId, Long characterId, int stage, int score) {
        RelationshipEntity r = relationshipWithScore(userId, characterId, score);
        r.setCurrentStage((short) stage);
        return r;
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): RelationshipEntity + Repository + Fixture（复合 PK + 乐观锁）"
```

---

#### Task B3: IntimacyLogEntity + Repository + Fixture

**Files:**
- Create: `internal/intimacy/IntimacyLogEntity.java` + `Repository.java`
- Create: `test/.../fixtures/IntimacyLogTestFixtures.java`
- Test: `test/.../intimacy/IntimacyLogRepositoryIT.java`

- [ ] **Step 1: Write the failing test**

```java
@DataJpaTest
class IntimacyLogRepositoryIT {
    @Autowired IntimacyLogRepository repo;

    @Test
    void findTop10_should_return_descending_by_created_at() {
        for (int i = 0; i < 15; i++) {
            repo.save(IntimacyLogTestFixtures.validLog(1L, 1L, "MESSAGE_SENT", 1, 1 + i, 0));
        }
        List<IntimacyLogEntity> logs = repo.findTop10ByUserIdAndCharacterIdOrderByCreatedAtDesc(1L, 1L);
        assertThat(logs).hasSize(10);
        assertThat(logs.get(0).getNewScore()).isGreaterThan(logs.get(9).getNewScore());
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
// IntimacyLogEntity.java
@Data
@Entity
@Table(name = "intimacy_logs")
public class IntimacyLogEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "user_id",      nullable = false) private Long   userId;
    @Column(name = "character_id", nullable = false) private Long   characterId;
    @Column(nullable = false)                         private Integer delta;
    @Column(nullable = false, length = 60)            private String  reason;
    @Column(columnDefinition = "jsonb")
    @org.hibernate.annotations.JdbcTypeCode(org.hibernate.type.SqlTypes.JSON)
    private String metadata;
    @Column(name = "new_score", nullable = false) private Integer newScore;
    @Column(name = "new_stage", nullable = false) private Short   newStage;
    @CreationTimestamp @Column(name = "created_at") private LocalDateTime createdAt;
}
```

```java
public interface IntimacyLogRepository extends JpaRepository<IntimacyLogEntity, Long> {
    List<IntimacyLogEntity> findTop10ByUserIdAndCharacterIdOrderByCreatedAtDesc(Long userId, Long characterId);
}
```

```java
public final class IntimacyLogTestFixtures {
    private IntimacyLogTestFixtures() {}

    public static IntimacyLogEntity validLog(Long userId, Long characterId, String reason, int delta, int newScore, int newStage) {
        IntimacyLogEntity log = new IntimacyLogEntity();
        log.setUserId(userId);
        log.setCharacterId(characterId);
        log.setReason(reason);
        log.setDelta(delta);
        log.setNewScore(newScore);
        log.setNewStage((short) newStage);
        return log;
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): IntimacyLogEntity + Repository + Fixture（审计 / 倒序查询）"
```

---

#### Task B4: RelationshipMilestoneEntity + Repository + Fixture

**Files:**
- Create: `internal/plotrule/RelationshipMilestoneEntity.java` + `MilestoneId.java` + `Repository.java`
- Create: `test/.../fixtures/RelationshipMilestoneTestFixtures.java`
- Test: `test/.../plotrule/RelationshipMilestoneRepositoryIT.java`

- [ ] **Step 1: Write the failing test**

```java
@DataJpaTest
class RelationshipMilestoneRepositoryIT {
    @Autowired RelationshipMilestoneRepository repo;

    @Test
    void existsBy_should_be_true_only_after_save() {
        var id = new RelationshipMilestoneId(1L, 1L, "deep_night_chat");
        assertThat(repo.existsById(id)).isFalse();
        repo.save(RelationshipMilestoneTestFixtures.validMilestone(1L, 1L, "deep_night_chat"));
        assertThat(repo.existsById(id)).isTrue();
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
// RelationshipMilestoneId.java
@Data @NoArgsConstructor @AllArgsConstructor
public class RelationshipMilestoneId implements Serializable {
    private Long userId; private Long characterId; private String ruleId;
}
```

```java
// RelationshipMilestoneEntity.java
@Data @Entity @Table(name = "relationship_milestones") @IdClass(RelationshipMilestoneId.class)
public class RelationshipMilestoneEntity {
    @Id @Column(name = "user_id")      private Long   userId;
    @Id @Column(name = "character_id") private Long   characterId;
    @Id @Column(name = "rule_id", length = 60) private String ruleId;
    @CreationTimestamp @Column(name = "triggered_at") private LocalDateTime triggeredAt;
}
```

```java
public interface RelationshipMilestoneRepository
        extends JpaRepository<RelationshipMilestoneEntity, RelationshipMilestoneId> {}
```

```java
public final class RelationshipMilestoneTestFixtures {
    private RelationshipMilestoneTestFixtures() {}

    public static RelationshipMilestoneEntity validMilestone(Long userId, Long characterId, String ruleId) {
        RelationshipMilestoneEntity m = new RelationshipMilestoneEntity();
        m.setUserId(userId); m.setCharacterId(characterId); m.setRuleId(ruleId);
        return m;
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): RelationshipMilestoneEntity + Repository + Fixture（3 列复合 PK 去重）"
```

---

### Phase C · Stage 基础

#### Task C1: IntimacyProperties (@ConfigurationProperties) + StageDefinition enum

**Files:**
- Create: `internal/intimacy/IntimacyProperties.java`
- Create: `internal/stage/StageDefinition.java`
- Modify: `server/business_packages/sanyan-character-core/src/main/resources/application-character.yml`（**或全局** `bootstrap/src/main/resources/application.yml`，按现有约定选）
- Test: `test/.../stage/StageDefinitionTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class StageDefinitionTest {
    IntimacyProperties props;

    @BeforeEach
    void setup() {
        props = new IntimacyProperties();
        props.getStages().setStrangerEnd(100);
        props.getStages().setFriendEnd(300);
        props.getStages().setAmbiguousEnd(600);
        props.getStages().setLoverEnd(1000);
    }

    @Test
    void stage_for_score_boundary() {
        StageDefinition def = new StageDefinition(props);
        assertThat(def.stageFor(0)).isEqualTo(0);
        assertThat(def.stageFor(99)).isEqualTo(0);
        assertThat(def.stageFor(100)).isEqualTo(1);
        assertThat(def.stageFor(299)).isEqualTo(1);
        assertThat(def.stageFor(300)).isEqualTo(2);
        assertThat(def.stageFor(600)).isEqualTo(3);
        assertThat(def.stageFor(1000)).isEqualTo(4);
        assertThat(def.stageFor(99999)).isEqualTo(4);
    }

    @Test
    void stage_name_should_be_chinese() {
        StageDefinition def = new StageDefinition(props);
        assertThat(def.nameOf(0)).isEqualTo("陌生人");
        assertThat(def.nameOf(2)).isEqualTo("暧昧");
        assertThat(def.nameOf(4)).isEqualTo("老夫老妻");
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
// IntimacyProperties.java
@Component
@ConfigurationProperties("sanyan.intimacy")
@Data
public class IntimacyProperties {
    private Stages stages = new Stages();
    private Delta delta = new Delta();
    private Rules rules = new Rules();
    private Ai ai = new Ai();

    @Data public static class Stages { int strangerEnd, friendEnd, ambiguousEnd, loverEnd; }
    @Data public static class Delta {
        int messageSent = 1, messageDailyCap = 50, dailyLogin = 10, streakBonusPerDay = 5, streakBonusCap = 50;
    }
    @Data public static class Rules { int deepNightChat = 50, firstHonestShare = 30; }
    @Data public static class Ai { int qualityMaxScore = 20, triggerEveryNMessages = 10; }
}
```

```java
// StageDefinition.java
@Component
@RequiredArgsConstructor
public class StageDefinition {
    public static final String[] NAMES = {"陌生人", "朋友", "暧昧", "恋人", "老夫老妻"};

    private final IntimacyProperties props;

    public int stageFor(int score) {
        var s = props.getStages();
        if (score < s.getStrangerEnd()) return 0;
        if (score < s.getFriendEnd())   return 1;
        if (score < s.getAmbiguousEnd())return 2;
        if (score < s.getLoverEnd())    return 3;
        return 4;
    }

    public String nameOf(int stage) { return NAMES[stage]; }

    public int nextThresholdOf(int stage) {
        var s = props.getStages();
        return switch (stage) {
            case 0 -> s.getStrangerEnd();
            case 1 -> s.getFriendEnd();
            case 2 -> s.getAmbiguousEnd();
            case 3 -> s.getLoverEnd();
            default -> Integer.MAX_VALUE;
        };
    }
}
```

加 yml 配置：
```yaml
sanyan:
  intimacy:
    stages:
      strangerEnd: 100
      friendEnd: 300
      ambiguousEnd: 600
      loverEnd: 1000
    delta: { messageSent: 1, messageDailyCap: 50, dailyLogin: 10, streakBonusPerDay: 5, streakBonusCap: 50 }
    rules: { deepNightChat: 50, firstHonestShare: 30 }
    ai:    { qualityMaxScore: 20, triggerEveryNMessages: 10 }
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/
git commit -m "feat(character-core): IntimacyProperties + StageDefinition（参数化 + 5 阶段定义）"
```

---

#### Task C2: StageTransitionDetectService

**Files:**
- Create: `internal/stage/StageTransitionDetectService.java`
- Test: `test/.../stage/StageTransitionDetectServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class StageTransitionDetectServiceTest {
    @Mock ApplicationEventPublisher publisher;
    @Mock RelationshipRepository repo;
    StageDefinition stageDef;
    StageTransitionDetectService service;

    @BeforeEach
    void setup() {
        IntimacyProperties props = new IntimacyProperties();
        props.getStages().setStrangerEnd(100); props.getStages().setFriendEnd(300);
        props.getStages().setAmbiguousEnd(600); props.getStages().setLoverEnd(1000);
        stageDef = new StageDefinition(props);
        service = new StageTransitionDetectService(stageDef, repo, publisher);
    }

    @Test
    void should_publish_event_when_crossing_boundary() {
        RelationshipEntity rel = RelationshipTestFixtures.relationshipAtStage(1L, 1L, 0, 95);
        service.maybeTransition(rel, 105);

        assertThat(rel.getCurrentStage()).isEqualTo((short) 1);
        verify(publisher).publishEvent(any(StageTransitionEvent.class));
    }

    @Test
    void should_not_publish_event_when_same_stage() {
        RelationshipEntity rel = RelationshipTestFixtures.relationshipAtStage(1L, 1L, 1, 200);
        service.maybeTransition(rel, 250);

        assertThat(rel.getCurrentStage()).isEqualTo((short) 1);
        verifyNoInteractions(publisher);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Service
@RequiredArgsConstructor
public class StageTransitionDetectService {
    private final StageDefinition stageDef;
    private final RelationshipRepository repo;
    private final ApplicationEventPublisher publisher;

    public void maybeTransition(RelationshipEntity rel, int newScore) {
        int newStage = stageDef.stageFor(newScore);
        int oldStage = rel.getCurrentStage();
        if (newStage == oldStage) return;

        rel.setCurrentStage((short) newStage);
        repo.save(rel);
        publisher.publishEvent(new StageTransitionEvent(
            rel.getUserId(), rel.getCharacterId(), oldStage, newStage));
    }
}
```

> 注：本 task 引用 `StageTransitionEvent`，该 record 由 Phase G Task G1 创建。如果 Phase 顺序执行，G1 在 C2 之前。如要 C2 先行，G1 中 StageTransitionEvent 部分必须先提取出来。**实际推荐顺序：先做 Phase G1（创建事件契约），再做 C2。**

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): StageTransitionDetectService（检测阶段切换 + 发事件）"
```

---

#### Task C3: StageOverrideQueryService

**Files:**
- Create: `internal/stage/StageOverrideQueryService.java`
- Test: `test/.../stage/StageOverrideQueryServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class StageOverrideQueryServiceTest {
    @Mock AiCharacterRepository charRepo;
    StageDefinition stageDef;
    StageOverrideQueryService service;

    @BeforeEach
    void setup() {
        IntimacyProperties props = new IntimacyProperties();
        props.getStages().setStrangerEnd(100); props.getStages().setFriendEnd(300);
        props.getStages().setAmbiguousEnd(600); props.getStages().setLoverEnd(1000);
        stageDef = new StageDefinition(props);
        service = new StageOverrideQueryService(charRepo, stageDef, new ObjectMapper());
    }

    @Test
    void should_assemble_prompt_segment_from_persona_config() {
        AiCharacterEntity ch = AiCharacterTestFixtures.withPersonaConfig("基底", """
            { "stage_overrides": {
                "2": {"address":"宝","tone_hint":"撒娇","topics_unlock":["想见面"]}
            }}
            """);
        when(charRepo.findById(1L)).thenReturn(Optional.of(ch));

        String segment = service.buildSegment(1L, /*currentStage=*/ 2);

        assertThat(segment).contains("当前关系阶段：暧昧");
        assertThat(segment).contains("称呼用户用：宝");
        assertThat(segment).contains("语调：撒娇");
    }

    @Test
    void should_return_blank_when_no_override_for_stage() {
        AiCharacterEntity ch = AiCharacterTestFixtures.withPersonaConfig("基底", "{\"stage_overrides\":{}}");
        when(charRepo.findById(1L)).thenReturn(Optional.of(ch));

        assertThat(service.buildSegment(1L, 0)).isEmpty();
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Service
@RequiredArgsConstructor
public class StageOverrideQueryService {
    private final AiCharacterRepository charRepo;
    private final StageDefinition stageDef;
    private final ObjectMapper mapper;

    public String buildSegment(Long characterId, int stage) {
        AiCharacterEntity ch = charRepo.findById(characterId).orElse(null);
        if (ch == null || ch.getPersonaConfig() == null) return "";

        try {
            JsonNode root = mapper.readTree(ch.getPersonaConfig());
            JsonNode override = root.path("stage_overrides").path(String.valueOf(stage));
            if (override.isMissingNode() || override.isEmpty()) return "";

            String address  = override.path("address").asText("");
            String toneHint = override.path("tone_hint").asText("");

            return String.format("当前关系阶段：%s。称呼用户用：%s。语调：%s。",
                    stageDef.nameOf(stage), address, toneHint);
        } catch (JsonProcessingException e) {
            return "";
        }
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): StageOverrideQueryService（从 persona_config 拼 stage prompt 段）"
```

---

### Phase D · 亲密度核心

#### Task D1: CharacterErrCode 加 3002 / 3003

**Files:**
- Modify: `internal/CharacterErrCode.java`
- Test: `test/.../internal/CharacterErrCodeTest.java`

- [ ] **Step 1: Write the failing test**

```java
class CharacterErrCodeTest {
    @Test
    void should_have_relationship_not_found_3002() {
        assertThat(CharacterErrCode.RELATIONSHIP_NOT_FOUND.getCode()).isEqualTo(3002);
        assertThat(CharacterErrCode.RELATIONSHIP_NOT_FOUND.getDefaultMessage()).isEqualTo("关系不存在");
    }

    @Test
    void should_have_intimacy_concurrent_update_3003() {
        assertThat(CharacterErrCode.INTIMACY_CONCURRENT_UPDATE.getCode()).isEqualTo(3003);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

```bash
mvn -pl business_packages/sanyan-character-core -Dtest=CharacterErrCodeTest test
```

- [ ] **Step 3: Write minimal implementation**

```java
@Getter
@AllArgsConstructor
public enum CharacterErrCode implements ErrCode {
    CHARACTER_NOT_FOUND(3001, "角色不存在"),
    RELATIONSHIP_NOT_FOUND(3002, "关系不存在"),
    INTIMACY_CONCURRENT_UPDATE(3003, "亲密度并发更新失败");

    private final int code;
    private final String defaultMessage;
}
```

- [ ] **Step 4: Run to verify PASS** + `mvn -pl foundation_packages/sanyan-common-error test`（守护 ErrCodeConflictDetectorTest 仍绿）

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/internal/CharacterErrCode.java \
        server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/internal/CharacterErrCodeTest.java
git commit -m "feat(character-core): CharacterErrCode 增 3002/3003（关系不存在 / 亲密度并发更新失败）"
```

---

#### Task D2: IntimacyEvent + IntimacyCalculator

**Files:**
- Create: `internal/intimacy/IntimacyEvent.java`
- Create: `internal/intimacy/IntimacyCalculator.java`
- Test: `test/.../intimacy/IntimacyCalculatorTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class IntimacyCalculatorTest {
    @Mock DailyBehaviorCounter dailyCounter;
    IntimacyProperties props = new IntimacyProperties();
    IntimacyCalculator calculator;

    @BeforeEach
    void setup() { calculator = new IntimacyCalculator(props, dailyCounter); }

    @Test
    void message_sent_should_yield_1() {
        when(dailyCounter.consumedToday(1L)).thenReturn(0);
        var event = IntimacyEvent.messageSent(1L, 1L);
        assertThat(calculator.compute(event)).isEqualTo(1);
    }

    @Test
    void message_sent_should_be_zero_after_daily_cap() {
        when(dailyCounter.consumedToday(1L)).thenReturn(50);
        var event = IntimacyEvent.messageSent(1L, 1L);
        assertThat(calculator.compute(event)).isZero();
    }

    @Test
    void daily_login_with_streak_3() {
        var event = IntimacyEvent.dailyLogin(1L, 1L, /*streak=*/ 3);
        // dailyLogin(10) + streak(3) * 5 = 25
        assertThat(calculator.compute(event)).isEqualTo(25);
    }

    @Test
    void daily_login_should_cap_streak_bonus() {
        var event = IntimacyEvent.dailyLogin(1L, 1L, /*streak=*/ 100);
        // dailyLogin(10) + min(100*5, 50) = 60
        assertThat(calculator.compute(event)).isEqualTo(60);
    }

    @Test
    void plot_milestone_carries_explicit_delta() {
        var event = IntimacyEvent.plot(1L, 1L, "deep_night_chat", 50);
        assertThat(calculator.compute(event)).isEqualTo(50);
    }

    @Test
    void ai_quality_carries_score() {
        var event = IntimacyEvent.aiQuality(1L, 1L, 15);
        assertThat(calculator.compute(event)).isEqualTo(15);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
// IntimacyEvent.java
public record IntimacyEvent(Type type, Long userId, Long characterId, int payloadInt, String payloadStr) {
    public enum Type { MESSAGE_SENT, DAILY_LOGIN, PLOT_MILESTONE, AI_QUALITY_BONUS }

    public static IntimacyEvent messageSent(Long uid, Long cid) {
        return new IntimacyEvent(Type.MESSAGE_SENT, uid, cid, 0, null);
    }
    public static IntimacyEvent dailyLogin(Long uid, Long cid, int streak) {
        return new IntimacyEvent(Type.DAILY_LOGIN, uid, cid, streak, null);
    }
    public static IntimacyEvent plot(Long uid, Long cid, String ruleId, int delta) {
        return new IntimacyEvent(Type.PLOT_MILESTONE, uid, cid, delta, ruleId);
    }
    public static IntimacyEvent aiQuality(Long uid, Long cid, int score) {
        return new IntimacyEvent(Type.AI_QUALITY_BONUS, uid, cid, score, null);
    }
}
```

```java
// IntimacyCalculator.java
@Component
@RequiredArgsConstructor
public class IntimacyCalculator {
    private final IntimacyProperties props;
    private final DailyBehaviorCounter dailyCounter;

    public int compute(IntimacyEvent event) {
        return switch (event.type()) {
            case MESSAGE_SENT -> {
                int consumed = dailyCounter.consumedToday(event.userId());
                int remaining = props.getDelta().getMessageDailyCap() - consumed;
                yield Math.max(0, Math.min(props.getDelta().getMessageSent(), remaining));
            }
            case DAILY_LOGIN -> {
                int streak = event.payloadInt();
                int bonus = Math.min(streak * props.getDelta().getStreakBonusPerDay(),
                                     props.getDelta().getStreakBonusCap());
                yield props.getDelta().getDailyLogin() + bonus;
            }
            case PLOT_MILESTONE  -> event.payloadInt();
            case AI_QUALITY_BONUS -> Math.min(event.payloadInt(), props.getAi().getQualityMaxScore());
        };
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): IntimacyEvent record + IntimacyCalculator（6 种 type × 边界）"
```

---

#### Task D3: DailyBehaviorCounter（Redis 每日封顶）

**Files:**
- Create: `internal/intimacy/DailyBehaviorCounter.java`
- Test: `test/.../intimacy/DailyBehaviorCounterIT.java`（Testcontainers Redis）

- [ ] **Step 1: Write the failing test**

```java
@SpringBootTest
@Testcontainers
class DailyBehaviorCounterIT {
    @Container static GenericContainer<?> redis =
        new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProps(DynamicPropertyRegistry r) {
        r.add("spring.data.redis.host", redis::getHost);
        r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired DailyBehaviorCounter counter;

    @Test
    void incr_and_consumedToday_should_match() {
        counter.incr(1L, 5);
        counter.incr(1L, 3);
        assertThat(counter.consumedToday(1L)).isEqualTo(8);
    }

    @Test
    void different_users_should_not_interfere() {
        counter.incr(1L, 10);
        counter.incr(2L, 5);
        assertThat(counter.consumedToday(1L)).isEqualTo(10);
        assertThat(counter.consumedToday(2L)).isEqualTo(5);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

```bash
mvn -pl business_packages/sanyan-character-core -Dtest=DailyBehaviorCounterIT verify
```

- [ ] **Step 3: Write minimal implementation**

```java
@Component
@RequiredArgsConstructor
public class DailyBehaviorCounter {
    private static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd");

    private final StringRedisTemplate redis;

    public void incr(Long userId, int delta) {
        String key = keyFor(userId);
        redis.opsForValue().increment(key, delta);
        redis.expire(key, Duration.ofHours(36));
    }

    public int consumedToday(Long userId) {
        String v = redis.opsForValue().get(keyFor(userId));
        return v == null ? 0 : Integer.parseInt(v);
    }

    private String keyFor(Long userId) {
        return "behavior:user:" + userId + ":date:" + LocalDate.now().format(FMT);
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): DailyBehaviorCounter（Redis INCR + 36h TTL）"
```

---

#### Task D4: ConsecutiveLoginService（Redis streak）

**Files:**
- Create: `internal/intimacy/ConsecutiveLoginService.java`
- Test: `test/.../intimacy/ConsecutiveLoginServiceIT.java`

- [ ] **Step 1: Write the failing test**

```java
@SpringBootTest @Testcontainers
class ConsecutiveLoginServiceIT {
    @Container static GenericContainer<?> redis =
        new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);
    @DynamicPropertySource
    static void p(DynamicPropertyRegistry r) {
        r.add("spring.data.redis.host", redis::getHost);
        r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired ConsecutiveLoginService svc;

    @Test
    void first_login_starts_streak_1() {
        var result = svc.recordLogin(1L, LocalDate.now());
        assertThat(result.isFirstToday()).isTrue();
        assertThat(result.streak()).isEqualTo(1);
    }

    @Test
    void second_call_same_day_is_idempotent() {
        svc.recordLogin(1L, LocalDate.now());
        var result = svc.recordLogin(1L, LocalDate.now());
        assertThat(result.isFirstToday()).isFalse();
        assertThat(result.streak()).isEqualTo(1);
    }

    @Test
    void consecutive_days_should_increment_streak() {
        svc.recordLogin(1L, LocalDate.of(2026, 5, 18));
        var d2 = svc.recordLogin(1L, LocalDate.of(2026, 5, 19));
        assertThat(d2.streak()).isEqualTo(2);
        var d3 = svc.recordLogin(1L, LocalDate.of(2026, 5, 20));
        assertThat(d3.streak()).isEqualTo(3);
    }

    @Test
    void gap_should_reset_streak() {
        svc.recordLogin(1L, LocalDate.of(2026, 5, 18));
        var afterGap = svc.recordLogin(1L, LocalDate.of(2026, 5, 22));
        assertThat(afterGap.streak()).isEqualTo(1);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Component
@RequiredArgsConstructor
public class ConsecutiveLoginService {

    public record LoginResult(boolean isFirstToday, int streak) {}

    private final StringRedisTemplate redis;

    public LoginResult recordLogin(Long userId, LocalDate today) {
        String key = "streak:user:" + userId;
        Map<Object, Object> entries = redis.opsForHash().entries(key);

        String lastDateStr = (String) entries.get("last_date");
        int prevStreak = entries.containsKey("streak") ? Integer.parseInt((String) entries.get("streak")) : 0;

        if (today.toString().equals(lastDateStr)) {
            return new LoginResult(false, prevStreak);
        }

        int newStreak;
        if (lastDateStr != null && LocalDate.parse(lastDateStr).plusDays(1).equals(today)) {
            newStreak = prevStreak + 1;
        } else {
            newStreak = 1;
        }

        redis.opsForHash().put(key, "streak",    String.valueOf(newStreak));
        redis.opsForHash().put(key, "last_date", today.toString());
        return new LoginResult(true, newStreak);
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): ConsecutiveLoginService（Redis streak / 中断重置 / 当日幂等）"
```

---

#### Task D5: IntimacyRecordService（累加 + 阶段切换 + 事件 + 乐观锁 retry）

**Files:**
- Create: `internal/intimacy/IntimacyRecordService.java`
- Test: `test/.../intimacy/IntimacyRecordServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class IntimacyRecordServiceTest {
    @Mock RelationshipRepository relRepo;
    @Mock IntimacyLogRepository logRepo;
    @Mock IntimacyCalculator calculator;
    @Mock DailyBehaviorCounter dailyCounter;
    @Mock StageTransitionDetectService stageTransition;
    @Mock ApplicationEventPublisher publisher;
    @Mock StageDefinition stageDef;
    @InjectMocks IntimacyRecordService service;

    @Test
    void should_accumulate_and_publish_intimacy_changed_event() {
        RelationshipEntity rel = RelationshipTestFixtures.relationshipWithScore(1L, 1L, 50);
        when(relRepo.findByUserIdAndCharacterId(1L, 1L)).thenReturn(Optional.of(rel));
        when(calculator.compute(any())).thenReturn(5);
        when(stageDef.stageFor(55)).thenReturn(0);

        service.recordEvent(IntimacyEvent.messageSent(1L, 1L));

        assertThat(rel.getIntimacyScore()).isEqualTo(55);
        verify(logRepo).save(any(IntimacyLogEntity.class));
        verify(publisher).publishEvent(any(IntimacyChangedEvent.class));
    }

    @Test
    void should_call_stage_transition_when_score_changes() {
        RelationshipEntity rel = RelationshipTestFixtures.relationshipWithScore(1L, 1L, 95);
        when(relRepo.findByUserIdAndCharacterId(1L, 1L)).thenReturn(Optional.of(rel));
        when(calculator.compute(any())).thenReturn(10);

        service.recordEvent(IntimacyEvent.messageSent(1L, 1L));

        verify(stageTransition).maybeTransition(rel, 105);
    }

    @Test
    void should_still_write_log_when_delta_is_zero_capped() {
        RelationshipEntity rel = RelationshipTestFixtures.relationshipWithScore(1L, 1L, 50);
        when(relRepo.findByUserIdAndCharacterId(1L, 1L)).thenReturn(Optional.of(rel));
        when(calculator.compute(any())).thenReturn(0);

        service.recordEvent(IntimacyEvent.messageSent(1L, 1L));

        ArgumentCaptor<IntimacyLogEntity> cap = ArgumentCaptor.forClass(IntimacyLogEntity.class);
        verify(logRepo).save(cap.capture());
        assertThat(cap.getValue().getReason()).isEqualTo("CAPPED");
    }

    @Test
    void should_throw_intimacy_concurrent_update_after_retry_exhausted() {
        RelationshipEntity rel = RelationshipTestFixtures.relationshipWithScore(1L, 1L, 50);
        when(relRepo.findByUserIdAndCharacterId(1L, 1L)).thenReturn(Optional.of(rel));
        when(calculator.compute(any())).thenReturn(5);
        when(relRepo.save(any())).thenThrow(new OptimisticLockingFailureException("lock"));

        assertThatThrownBy(() -> service.recordEvent(IntimacyEvent.messageSent(1L, 1L)))
            .isInstanceOf(BusinessException.class)
            .hasFieldOrPropertyWithValue("errCode", CharacterErrCode.INTIMACY_CONCURRENT_UPDATE);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Service
@RequiredArgsConstructor
public class IntimacyRecordService {
    private static final int MAX_RETRY = 3;

    private final RelationshipRepository relRepo;
    private final IntimacyLogRepository logRepo;
    private final IntimacyCalculator calculator;
    private final DailyBehaviorCounter dailyCounter;
    private final StageTransitionDetectService stageTransition;
    private final ApplicationEventPublisher publisher;
    private final StageDefinition stageDef;

    @Transactional
    public void recordEvent(IntimacyEvent event) {
        OptimisticLockingFailureException last = null;
        for (int attempt = 0; attempt < MAX_RETRY; attempt++) {
            try {
                doRecord(event);
                return;
            } catch (OptimisticLockingFailureException e) {
                last = e;
            }
        }
        throw new BusinessException(CharacterErrCode.INTIMACY_CONCURRENT_UPDATE);
    }

    private void doRecord(IntimacyEvent event) {
        RelationshipEntity rel = relRepo.findByUserIdAndCharacterId(event.userId(), event.characterId())
            .orElseThrow(() -> new BusinessException(CharacterErrCode.RELATIONSHIP_NOT_FOUND));

        int delta = calculator.compute(event);
        int oldScore = rel.getIntimacyScore();
        int newScore = oldScore + delta;
        rel.setIntimacyScore(newScore);
        relRepo.save(rel);

        if (event.type() == IntimacyEvent.Type.MESSAGE_SENT && delta > 0) {
            dailyCounter.incr(event.userId(), delta);
        }

        IntimacyLogEntity log = new IntimacyLogEntity();
        log.setUserId(event.userId());
        log.setCharacterId(event.characterId());
        log.setDelta(delta);
        log.setReason(delta == 0 ? "CAPPED" : reasonOf(event));
        log.setNewScore(newScore);
        log.setNewStage((short) stageDef.stageFor(newScore));
        logRepo.save(log);

        stageTransition.maybeTransition(rel, newScore);
        publisher.publishEvent(new IntimacyChangedEvent(
            event.userId(), event.characterId(), oldScore, newScore, delta, log.getReason()));
    }

    private String reasonOf(IntimacyEvent event) {
        return switch (event.type()) {
            case MESSAGE_SENT      -> "MESSAGE_SENT";
            case DAILY_LOGIN       -> "DAILY_LOGIN";
            case PLOT_MILESTONE    -> "PLOT:" + event.payloadStr();
            case AI_QUALITY_BONUS  -> "AI_QUALITY";
        };
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): IntimacyRecordService（累加 + 阶段切换 + 事件 + 乐观锁 retry）"
```

---

### Phase E · 剧情节点规则

#### Task E1: PlotMilestoneRule 接口 + PlotMilestoneEngine + MessageContext

**Files:**
- Create: `internal/plotrule/PlotMilestoneRule.java`（接口）
- Create: `internal/plotrule/MessageContext.java`（record）
- Create: `internal/plotrule/PlotMilestoneEngine.java`
- Test: `test/.../plotrule/PlotMilestoneEngineTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class PlotMilestoneEngineTest {
    @Mock RelationshipMilestoneRepository milestoneRepo;
    PlotMilestoneEngine engine;

    @Test
    void should_aggregate_events_from_all_rules() {
        PlotMilestoneRule alwaysTrigger = ctx -> Optional.of(IntimacyEvent.plot(1L, 1L, "always", 10));
        PlotMilestoneRule neverTrigger  = ctx -> Optional.empty();
        engine = new PlotMilestoneEngine(List.of(alwaysTrigger, neverTrigger), milestoneRepo);

        var events = engine.evaluate(new MessageContext(1L, 1L, List.of(), Set.of()));
        assertThat(events).hasSize(1);
        assertThat(events.get(0).payloadStr()).isEqualTo("always");
    }

    @Test
    void should_skip_rule_already_in_milestones() {
        PlotMilestoneRule trigger = new PlotMilestoneRule() {
            @Override public String ruleId() { return "deep_night_chat"; }
            @Override public Optional<IntimacyEvent> evaluate(MessageContext ctx) {
                return Optional.of(IntimacyEvent.plot(1L, 1L, ruleId(), 50));
            }
        };
        engine = new PlotMilestoneEngine(List.of(trigger), milestoneRepo);
        var ctx = new MessageContext(1L, 1L, List.of(), Set.of("deep_night_chat"));

        assertThat(engine.evaluate(ctx)).isEmpty();
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
// MessageContext.java
public record MessageContext(
    Long userId, Long characterId,
    List<MessageEntity> recentMessages,
    Set<String> triggeredRuleIds
) {}
```

```java
// PlotMilestoneRule.java
public interface PlotMilestoneRule {
    /** 规则唯一 id（用于 milestones 去重）；默认按类名小写下划线 */
    default String ruleId() {
        return getClass().getSimpleName().replace("Rule", "")
                .replaceAll("([a-z])([A-Z])", "$1_$2").toLowerCase();
    }

    Optional<IntimacyEvent> evaluate(MessageContext ctx);
}
```

```java
// PlotMilestoneEngine.java
@Component
@RequiredArgsConstructor
public class PlotMilestoneEngine {
    private final List<PlotMilestoneRule> rules;
    private final RelationshipMilestoneRepository milestoneRepo;

    public List<IntimacyEvent> evaluate(MessageContext ctx) {
        return rules.stream()
            .filter(r -> !ctx.triggeredRuleIds().contains(r.ruleId()))
            .map(r -> r.evaluate(ctx))
            .flatMap(Optional::stream)
            .toList();
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): PlotMilestoneRule 接口 + Engine + MessageContext"
```

---

#### Task E2: DeepNightChatRule（连续 3 晚 22-02 +50）

**Files:**
- Create: `internal/plotrule/DeepNightChatRule.java`
- Test: `test/.../plotrule/DeepNightChatRuleTest.java`

- [ ] **Step 1: Write the failing test**

```java
class DeepNightChatRuleTest {
    DeepNightChatRule rule;
    IntimacyProperties props = new IntimacyProperties();

    @BeforeEach void setup() { rule = new DeepNightChatRule(props); }

    @Test
    void should_trigger_when_user_chatted_3_consecutive_nights() {
        LocalDateTime base = LocalDateTime.of(2026, 5, 18, 23, 0);
        var msgs = List.of(
            messageAt(base),
            messageAt(base.plusDays(1)),
            messageAt(base.plusDays(2))
        );
        var ctx = new MessageContext(1L, 1L, msgs, Set.of());

        var evt = rule.evaluate(ctx);
        assertThat(evt).isPresent();
        assertThat(evt.get().payloadInt()).isEqualTo(50);
    }

    @Test
    void should_not_trigger_with_only_2_nights() {
        LocalDateTime base = LocalDateTime.of(2026, 5, 18, 23, 0);
        var ctx = new MessageContext(1L, 1L,
            List.of(messageAt(base), messageAt(base.plusDays(1))),
            Set.of());

        assertThat(rule.evaluate(ctx)).isEmpty();
    }

    @Test
    void should_not_trigger_daytime_messages() {
        LocalDateTime base = LocalDateTime.of(2026, 5, 18, 14, 0);
        var ctx = new MessageContext(1L, 1L,
            List.of(messageAt(base), messageAt(base.plusDays(1)), messageAt(base.plusDays(2))),
            Set.of());

        assertThat(rule.evaluate(ctx)).isEmpty();
    }

    private MessageEntity messageAt(LocalDateTime t) {
        MessageEntity m = new MessageEntity();
        m.setCreatedAt(t); m.setSenderType("user");
        return m;
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Component
@RequiredArgsConstructor
public class DeepNightChatRule implements PlotMilestoneRule {
    private final IntimacyProperties props;

    @Override public String ruleId() { return "deep_night_chat"; }

    @Override
    public Optional<IntimacyEvent> evaluate(MessageContext ctx) {
        Set<LocalDate> nightDates = ctx.recentMessages().stream()
            .filter(m -> "user".equals(m.getSenderType()))
            .filter(m -> {
                int hour = m.getCreatedAt().getHour();
                return hour >= 22 || hour < 2;
            })
            .map(m -> {
                // 22:00 之后归该日，00:00-01:59 归前日（"那一晚"）
                LocalDateTime t = m.getCreatedAt();
                return t.getHour() < 2 ? t.toLocalDate().minusDays(1) : t.toLocalDate();
            })
            .collect(Collectors.toSet());

        if (nightDates.size() < 3) return Optional.empty();

        List<LocalDate> sorted = nightDates.stream().sorted().toList();
        for (int i = 0; i <= sorted.size() - 3; i++) {
            if (sorted.get(i).plusDays(1).equals(sorted.get(i + 1))
                && sorted.get(i + 1).plusDays(1).equals(sorted.get(i + 2))) {
                return Optional.of(IntimacyEvent.plot(
                    ctx.userId(), ctx.characterId(), ruleId(), props.getRules().getDeepNightChat()));
            }
        }
        return Optional.empty();
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): DeepNightChatRule（连续 3 晚 22-02 +50）"
```

---

#### Task E3: FirstHonestShareRule（情感关键词首次命中 +30）

**Files:**
- Create: `internal/plotrule/FirstHonestShareRule.java`
- Test: `test/.../plotrule/FirstHonestShareRuleTest.java`

- [ ] **Step 1: Write the failing test**

```java
class FirstHonestShareRuleTest {
    IntimacyProperties props = new IntimacyProperties();
    FirstHonestShareRule rule = new FirstHonestShareRule(props);

    @Test
    void should_trigger_when_user_says_emotional_keyword() {
        var msgs = List.of(userMsg("我有点难过"));
        var evt = rule.evaluate(new MessageContext(1L, 1L, msgs, Set.of()));
        assertThat(evt).isPresent();
        assertThat(evt.get().payloadInt()).isEqualTo(30);
    }

    @Test
    void should_not_trigger_on_neutral_message() {
        var msgs = List.of(userMsg("今天天气不错"));
        assertThat(rule.evaluate(new MessageContext(1L, 1L, msgs, Set.of()))).isEmpty();
    }

    @Test
    void should_skip_if_already_triggered() {
        var msgs = List.of(userMsg("我心里好乱"));
        var ctx = new MessageContext(1L, 1L, msgs, Set.of("first_honest_share"));
        // engine 层去重，本 rule 自身也判一下
        assertThat(rule.evaluate(ctx)).isEmpty();
    }

    private MessageEntity userMsg(String content) {
        MessageEntity m = new MessageEntity();
        m.setSenderType("user"); m.setContent(content);
        m.setCreatedAt(LocalDateTime.now());
        return m;
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Component
@RequiredArgsConstructor
public class FirstHonestShareRule implements PlotMilestoneRule {
    private static final List<String> KEYWORDS = List.of(
        "我有点难过", "我跟家里", "我心里", "我最近", "我其实", "其实我"
    );

    private final IntimacyProperties props;

    @Override public String ruleId() { return "first_honest_share"; }

    @Override
    public Optional<IntimacyEvent> evaluate(MessageContext ctx) {
        if (ctx.triggeredRuleIds().contains(ruleId())) return Optional.empty();

        boolean hit = ctx.recentMessages().stream()
            .filter(m -> "user".equals(m.getSenderType()))
            .map(MessageEntity::getContent)
            .filter(Objects::nonNull)
            .anyMatch(content -> KEYWORDS.stream().anyMatch(content::contains));

        if (!hit) return Optional.empty();
        return Optional.of(IntimacyEvent.plot(
            ctx.userId(), ctx.characterId(), ruleId(), props.getRules().getFirstHonestShare()));
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): FirstHonestShareRule（情感关键词首次命中 +30）"
```

---

#### Task E4: StageTransitionStoryListener（阶段切换剧情演出）

**Files:**
- Create: `event/StageTransitionStoryListener.java`
- Test: `test/.../event/StageTransitionStoryListenerTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class StageTransitionStoryListenerTest {
    @Mock RelationshipMilestoneRepository milestoneRepo;
    @Mock ApplicationEventPublisher publisher;
    @InjectMocks StageTransitionStoryListener listener;

    @Test
    void should_save_milestone_and_publish_story_event_once() {
        when(milestoneRepo.existsById(any())).thenReturn(false);
        listener.onStageTransition(new StageTransitionEvent(1L, 1L, 1, 2));

        verify(milestoneRepo).save(any(RelationshipMilestoneEntity.class));
        verify(publisher).publishEvent(any(StageEntryStoryEvent.class));
    }

    @Test
    void should_skip_when_milestone_already_recorded() {
        when(milestoneRepo.existsById(any())).thenReturn(true);
        listener.onStageTransition(new StageTransitionEvent(1L, 1L, 1, 2));

        verify(milestoneRepo, never()).save(any());
        verifyNoInteractions(publisher);
    }
}
```

> 新增内部事件 `StageEntryStoryEvent`（character-api/event/）携带 `storyMessage` 文案——它由 character-core 自家 listener 拼好，再由 chat-core 监听并下发 WS。

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

新增 `business_packages/sanyan-character-api/src/main/java/com/sanyan/character/event/StageEntryStoryEvent.java`：
```java
public record StageEntryStoryEvent(Long userId, Long characterId, int toStage, String storyMessage) {}
```

```java
// event/StageTransitionStoryListener.java
@Component
@RequiredArgsConstructor
public class StageTransitionStoryListener {
    private static final String[] STORIES = {
        "",                                    // 0
        "她第一次自然地叫了你的名字……",          // 1 朋友
        "她半夜悄悄打字又删掉，最后还是发了出来……", // 2 暧昧
        "她第一次叫你「宝贝」……",                // 3 恋人
        "她说："你做的饭比我妈做的还好吃。""        // 4 老夫老妻
    };

    private final RelationshipMilestoneRepository milestoneRepo;
    private final ApplicationEventPublisher publisher;

    @EventListener
    public void onStageTransition(StageTransitionEvent event) {
        String ruleId = "stage_entry_" + event.toStage();
        var id = new RelationshipMilestoneId(event.userId(), event.characterId(), ruleId);
        if (milestoneRepo.existsById(id)) return;

        RelationshipMilestoneEntity m = new RelationshipMilestoneEntity();
        m.setUserId(event.userId()); m.setCharacterId(event.characterId()); m.setRuleId(ruleId);
        milestoneRepo.save(m);

        String story = STORIES[event.toStage()];
        publisher.publishEvent(new StageEntryStoryEvent(
            event.userId(), event.characterId(), event.toStage(), story));
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-api/ server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): StageTransitionStoryListener + StageEntryStoryEvent"
```

---

### Phase F · AI 评估

#### Task F1: ConversationQualityEvaluator

**Files:**
- Create: `internal/intimacy/ai/ConversationQualityEvaluator.java`
- Create: `internal/intimacy/ai/QualityScoreResponse.java`（record）
- Test: `test/.../intimacy/ai/ConversationQualityEvaluatorTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class ConversationQualityEvaluatorTest {
    @Mock LLMApi llmApi;
    IntimacyProperties props = new IntimacyProperties();
    ObjectMapper mapper = new ObjectMapper();
    ConversationQualityEvaluator evaluator;

    @BeforeEach void setup() {
        evaluator = new ConversationQualityEvaluator(llmApi, mapper, props);
    }

    @Test
    void should_parse_score_from_llm_json_response() {
        when(llmApi.callForText(anyString(), eq(LlmTaskType.QUALITY_EVAL)))
            .thenReturn("{\"score\":15,\"reason\":\"深度分享情绪\"}");

        var result = evaluator.score(List.of(userMsg("我今天有点难过……")));
        assertThat(result.score()).isEqualTo(15);
        assertThat(result.reason()).isEqualTo("深度分享情绪");
    }

    @Test
    void should_cap_score_at_max() {
        when(llmApi.callForText(anyString(), any())).thenReturn("{\"score\":999,\"reason\":\"x\"}");
        assertThat(evaluator.score(List.of()).score()).isEqualTo(20);
    }

    @Test
    void should_return_zero_on_parse_failure() {
        when(llmApi.callForText(anyString(), any())).thenReturn("not json");
        assertThat(evaluator.score(List.of()).score()).isZero();
    }

    private MessageEntity userMsg(String content) {
        MessageEntity m = new MessageEntity();
        m.setSenderType("user"); m.setContent(content);
        return m;
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
// QualityScoreResponse.java
public record QualityScoreResponse(int score, String reason) {}
```

```java
// ConversationQualityEvaluator.java
@Component
@RequiredArgsConstructor
@Slf4j
public class ConversationQualityEvaluator {
    private static final String PROMPT_TEMPLATE = """
        评估以下用户与小婉的对话片段质量（0-20 分）。
        评分维度：
        - 用户分享深度（透露真实信息、情感、想法）
        - 对话连贯性（不是单字"嗯"/"哦"敷衍）
        - 互动深度（双向，不是单方面输出）
        返回 JSON：{"score": <0-20>, "reason": "<10字内>"}
        对话：%s
        """;

    private final LLMApi llmApi;
    private final ObjectMapper mapper;
    private final IntimacyProperties props;

    public QualityScoreResponse score(List<MessageEntity> recentMessages) {
        String convo = recentMessages.stream()
            .map(m -> m.getSenderType() + "：" + m.getContent())
            .collect(Collectors.joining("\n"));
        String prompt = String.format(PROMPT_TEMPLATE, convo);

        try {
            String raw = llmApi.callForText(prompt, LlmTaskType.QUALITY_EVAL);
            QualityScoreResponse resp = mapper.readValue(raw, QualityScoreResponse.class);
            int capped = Math.min(resp.score(), props.getAi().getQualityMaxScore());
            return new QualityScoreResponse(capped, resp.reason());
        } catch (Exception e) {
            log.warn("评估失败", e);
            return new QualityScoreResponse(0, "评估失败");
        }
    }
}
```

> 注：本 task 引用 `LLMApi#callForText` 和 `LlmTaskType.QUALITY_EVAL`。若不存在则在 llm-api 模块中新增——属于 `llm-api` 模块的小扩。

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/
git commit -m "feat(character-core): ConversationQualityEvaluator（V4-Flash 评分 + 兜底）"
```

---

### Phase G · 事件契约 + Listener 织入

> ⚠️ **执行顺序提示**：Task G1 应该在 Phase C2（StageTransitionDetectService）之前做，因为 C2 引用 StageTransitionEvent。
> 实际推荐执行顺序：A → B → **G1** → C → D → E → F → G2 → H → I → J → K。

#### Task G1: IntimacyChangedEvent + StageTransitionEvent + character-core pom 加 chat-api 依赖

**Files:**
- Create: `sanyan-character-api/.../event/IntimacyChangedEvent.java`
- Create: `sanyan-character-api/.../event/StageTransitionEvent.java`
- Modify: `sanyan-character-core/pom.xml` 加 `<dependency>sanyan-chat-api</dependency>`
- Test: `test/.../event/CharacterEventContractTest.java`

- [ ] **Step 1: Write the failing test**

```java
class CharacterEventContractTest {
    @Test
    void intimacy_changed_event_should_be_record_with_6_fields() {
        var e = new IntimacyChangedEvent(1L, 2L, 10, 15, 5, "MESSAGE_SENT");
        assertThat(e.userId()).isEqualTo(1L);
        assertThat(e.delta()).isEqualTo(5);
        assertThat(e.reason()).isEqualTo("MESSAGE_SENT");
    }

    @Test
    void stage_transition_event_should_be_record_with_4_fields() {
        var e = new StageTransitionEvent(1L, 2L, 0, 1);
        assertThat(e.fromStage()).isZero();
        assertThat(e.toStage()).isEqualTo(1);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
public record IntimacyChangedEvent(
    Long userId, Long characterId,
    int oldScore, int newScore, int delta, String reason) {}
```

```java
public record StageTransitionEvent(
    Long userId, Long characterId,
    int fromStage, int toStage) {}
```

修改 `business_packages/sanyan-character-core/pom.xml`：
```xml
<dependency>
    <groupId>com.sanyan</groupId>
    <artifactId>sanyan-chat-api</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 4: Run to verify PASS** + `mvn -pl business_packages/sanyan-character-core validate`（Maven Enforcer 允许 character-core → chat-api，但禁止 character-core → chat-core）

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-api/ server/business_packages/sanyan-character-core/pom.xml
git commit -m "feat(character-api): IntimacyChangedEvent + StageTransitionEvent + character-core 依赖 chat-api"
```

---

#### Task G2: MessagePersistedListener（订阅消息事件，触发涨分链路）

**Files:**
- Create: `event/MessagePersistedListener.java`
- Test: `test/.../event/MessagePersistedListenerTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class MessagePersistedListenerTest {
    @Mock IntimacyRecordService recordService;
    @Mock PlotMilestoneEngine plotEngine;
    @Mock ConversationQualityEvaluator evaluator;
    @Mock MessageRepository msgRepo;
    @Mock RelationshipMilestoneRepository milestoneRepo;
    IntimacyProperties props = new IntimacyProperties();
    MessagePersistedListener listener;

    @BeforeEach void setup() {
        listener = new MessagePersistedListener(recordService, plotEngine, evaluator, msgRepo, milestoneRepo, props);
    }

    @Test
    void should_always_record_message_sent_on_user_message() {
        when(msgRepo.findTop32ByUserIdOrderByCreatedAtDesc(1L)).thenReturn(List.of());
        listener.onMessagePersisted(new MessagePersistedEvent(99L, 1L, 1L, "user", "你好", Instant.now()));

        verify(recordService).recordEvent(argThat(e -> e.type() == IntimacyEvent.Type.MESSAGE_SENT));
    }

    @Test
    void should_skip_on_ai_sender() {
        listener.onMessagePersisted(new MessagePersistedEvent(99L, 1L, 1L, "ai", "你好", Instant.now()));
        verifyNoInteractions(recordService);
    }

    @Test
    void should_dispatch_plot_events() {
        when(msgRepo.findTop32ByUserIdOrderByCreatedAtDesc(1L)).thenReturn(List.of());
        when(plotEngine.evaluate(any()))
            .thenReturn(List.of(IntimacyEvent.plot(1L, 1L, "deep_night_chat", 50)));

        listener.onMessagePersisted(new MessagePersistedEvent(99L, 1L, 1L, "user", "晚安", Instant.now()));

        verify(recordService).recordEvent(argThat(e -> e.type() == IntimacyEvent.Type.PLOT_MILESTONE));
    }

    @Test
    void should_trigger_ai_eval_every_n_messages() {
        var msgs = IntStream.range(0, 10).mapToObj(i -> userMsg("hi")).toList();
        when(msgRepo.findTop32ByUserIdOrderByCreatedAtDesc(1L)).thenReturn(msgs);
        when(evaluator.score(any())).thenReturn(new QualityScoreResponse(15, "深度"));

        listener.onMessagePersisted(new MessagePersistedEvent(99L, 1L, 1L, "user", "x", Instant.now()));

        verify(recordService).recordEvent(argThat(e -> e.type() == IntimacyEvent.Type.AI_QUALITY_BONUS));
    }

    private MessageEntity userMsg(String content) {
        MessageEntity m = new MessageEntity(); m.setSenderType("user"); m.setContent(content); return m;
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class MessagePersistedListener {
    private static final Long DEFAULT_CHARACTER_ID = 1L;  // 当前单角色（小婉），多角色时从 event 取

    private final IntimacyRecordService recordService;
    private final PlotMilestoneEngine plotEngine;
    private final ConversationQualityEvaluator evaluator;
    private final MessageRepository msgRepo;
    private final RelationshipMilestoneRepository milestoneRepo;
    private final IntimacyProperties props;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onMessagePersisted(MessagePersistedEvent event) {
        if (!"user".equals(event.senderType())) return;

        Long uid = event.userId();
        Long cid = DEFAULT_CHARACTER_ID;

        try {
            recordService.recordEvent(IntimacyEvent.messageSent(uid, cid));

            List<MessageEntity> recent = msgRepo.findTop32ByUserIdOrderByCreatedAtDesc(uid);
            Set<String> triggered = milestoneRepo.findAllByUserIdAndCharacterId(uid, cid).stream()
                .map(RelationshipMilestoneEntity::getRuleId).collect(Collectors.toSet());
            var ctx = new MessageContext(uid, cid, recent, triggered);

            for (var plotEvent : plotEngine.evaluate(ctx)) {
                recordService.recordEvent(plotEvent);
            }

            long userMsgCount = recent.stream().filter(m -> "user".equals(m.getSenderType())).count();
            if (userMsgCount > 0 && userMsgCount % props.getAi().getTriggerEveryNMessages() == 0) {
                QualityScoreResponse score = evaluator.score(recent);
                if (score.score() > 0) {
                    recordService.recordEvent(IntimacyEvent.aiQuality(uid, cid, score.score()));
                }
            }
        } catch (Exception e) {
            log.error("MessagePersistedListener 失败 userId={} msgId={}", uid, event.messageId(), e);
        }
    }
}
```

> 同时给 `RelationshipMilestoneRepository` 加方法 `findAllByUserIdAndCharacterId(Long, Long)`；给 `MessageRepository` 加 `findTop32ByUserIdOrderByCreatedAtDesc(Long)`（如已有 N-限制查询，直接复用）。

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/ server/business_packages/sanyan-chat-core/
git commit -m "feat(character-core): MessagePersistedListener（AFTER_COMMIT 触发涨分 / 剧情 / AI 评估）"
```

---

### Phase H · Relationship 服务 + CharacterApi 扩

#### Task H1: RelationshipDto + CharacterApi 扩 4 个方法

**Files:**
- Create: `sanyan-character-api/.../dto/RelationshipDto.java`
- Modify: `sanyan-character-api/.../CharacterApi.java`
- Test: `test/.../dto/RelationshipDtoTest.java`

- [ ] **Step 1: Write the failing test**

```java
class RelationshipDtoTest {
    @Test
    void should_be_record_with_7_fields() {
        var dto = new RelationshipDto(1L, 1L, 250, 1, "朋友", 300, 0.75);
        assertThat(dto.percentToNextStage()).isEqualTo(0.75);
    }
}

class CharacterApiContractTest {
    @Test
    void should_have_4_new_methods() throws NoSuchMethodException {
        Class<?> api = CharacterApi.class;
        assertThat(api.getMethod("findOrCreateRelationship", Long.class, Long.class)).isNotNull();
        assertThat(api.getMethod("fetchMyRelationship", Long.class, Long.class)).isNotNull();
        assertThat(api.getMethod("getStagePromptSegment", Long.class, Long.class)).isNotNull();
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
// RelationshipDto.java
public record RelationshipDto(
    Long userId, Long characterId,
    int intimacyScore, int currentStage, String currentStageName,
    int nextStageThreshold, double percentToNextStage) {}
```

```java
// CharacterApi.java（扩）
public interface CharacterApi {
    AiCharacterDto findById(Long characterId);
    AiCharacterDto getById(Long characterId);
    RelationshipDto findOrCreateRelationship(Long userId, Long characterId);
    RelationshipDto fetchMyRelationship(Long userId, Long characterId);
    String getStagePromptSegment(Long userId, Long characterId);
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-api/
git commit -m "feat(character-api): RelationshipDto + CharacterApi 扩 4 方法"
```

---

#### Task H2: RelationshipFindOrCreateService（唯一懒创建入口）

**Files:**
- Create: `internal/RelationshipFindOrCreateService.java`
- Test: `test/.../internal/RelationshipFindOrCreateServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class RelationshipFindOrCreateServiceTest {
    @Mock RelationshipRepository repo;
    @InjectMocks RelationshipFindOrCreateService service;

    @Test
    void should_return_existing_relationship() {
        var rel = RelationshipTestFixtures.relationshipWithScore(1L, 1L, 100);
        when(repo.findByUserIdAndCharacterId(1L, 1L)).thenReturn(Optional.of(rel));
        assertThat(service.findOrCreate(1L, 1L)).isSameAs(rel);
        verify(repo, never()).save(any());
    }

    @Test
    void should_create_new_relationship_when_absent() {
        when(repo.findByUserIdAndCharacterId(1L, 1L)).thenReturn(Optional.empty());
        when(repo.save(any())).thenAnswer(inv -> inv.getArgument(0));

        var rel = service.findOrCreate(1L, 1L);

        assertThat(rel.getUserId()).isEqualTo(1L);
        assertThat(rel.getIntimacyScore()).isZero();
        assertThat(rel.getCurrentStage()).isZero();
        verify(repo).save(any(RelationshipEntity.class));
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Service
@RequiredArgsConstructor
public class RelationshipFindOrCreateService {
    private final RelationshipRepository repo;

    @Transactional
    public RelationshipEntity findOrCreate(Long userId, Long characterId) {
        return repo.findByUserIdAndCharacterId(userId, characterId).orElseGet(() -> {
            RelationshipEntity r = new RelationshipEntity();
            r.setUserId(userId);
            r.setCharacterId(characterId);
            return repo.save(r);
        });
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): RelationshipFindOrCreateService（唯一懒创建入口）"
```

---

#### Task H3: RelationshipFetchService（GET /me 编排）

**Files:**
- Create: `internal/RelationshipFetchService.java`
- Test: `test/.../internal/RelationshipFetchServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class RelationshipFetchServiceTest {
    @Mock RelationshipFindOrCreateService findOrCreateService;
    @Mock ConsecutiveLoginService loginService;
    @Mock IntimacyRecordService intimacyRecordService;
    @Mock StageDefinition stageDef;
    @InjectMocks RelationshipFetchService service;

    @Test
    void should_record_daily_login_only_on_first_today() {
        var rel = RelationshipTestFixtures.relationshipWithScore(1L, 1L, 50);
        when(findOrCreateService.findOrCreate(1L, 1L)).thenReturn(rel);
        when(loginService.recordLogin(eq(1L), any()))
            .thenReturn(new ConsecutiveLoginService.LoginResult(true, 3));
        when(stageDef.nameOf(0)).thenReturn("陌生人");
        when(stageDef.nextThresholdOf(0)).thenReturn(100);

        var dto = service.fetchMyRelationship(1L, 1L);

        verify(intimacyRecordService).recordEvent(argThat(e ->
            e.type() == IntimacyEvent.Type.DAILY_LOGIN && e.payloadInt() == 3));
        assertThat(dto.currentStageName()).isEqualTo("陌生人");
    }

    @Test
    void should_skip_recording_when_not_first_today() {
        var rel = RelationshipTestFixtures.relationshipWithScore(1L, 1L, 50);
        when(findOrCreateService.findOrCreate(1L, 1L)).thenReturn(rel);
        when(loginService.recordLogin(eq(1L), any()))
            .thenReturn(new ConsecutiveLoginService.LoginResult(false, 3));
        when(stageDef.nameOf(0)).thenReturn("陌生人");
        when(stageDef.nextThresholdOf(0)).thenReturn(100);

        service.fetchMyRelationship(1L, 1L);
        verifyNoInteractions(intimacyRecordService);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Service
@RequiredArgsConstructor
public class RelationshipFetchService {
    private final RelationshipFindOrCreateService findOrCreateService;
    private final ConsecutiveLoginService loginService;
    private final IntimacyRecordService intimacyRecordService;
    private final StageDefinition stageDef;

    @Transactional
    public RelationshipDto fetchMyRelationship(Long userId, Long characterId) {
        RelationshipEntity rel = findOrCreateService.findOrCreate(userId, characterId);

        var login = loginService.recordLogin(userId, LocalDate.now());
        if (login.isFirstToday()) {
            intimacyRecordService.recordEvent(IntimacyEvent.dailyLogin(userId, characterId, login.streak()));
            // 重新读取保证 score 是 recordEvent 后的最新值
            rel = findOrCreateService.findOrCreate(userId, characterId);
        }

        int stage = rel.getCurrentStage();
        int nextThreshold = stageDef.nextThresholdOf(stage);
        int prevThreshold = stage == 0 ? 0 : stageDef.nextThresholdOf(stage - 1);
        double percent = nextThreshold == Integer.MAX_VALUE ? 1.0
            : (double)(rel.getIntimacyScore() - prevThreshold) / (nextThreshold - prevThreshold);

        return new RelationshipDto(
            userId, characterId, rel.getIntimacyScore(), stage,
            stageDef.nameOf(stage), nextThreshold, percent);
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): RelationshipFetchService（GET /me 编排：findOrCreate + recordLogin + 拼 DTO）"
```

---

#### Task H4: CharacterApiImpl 扩 4 个方法

**Files:**
- Modify: `api/CharacterApiImpl.java`
- Test: `test/.../api/CharacterApiImplTest.java`

- [ ] **Step 1: Write the failing test**

```java
@ExtendWith(MockitoExtension.class)
class CharacterApiImplTest {
    @Mock AiCharacterRepository charRepo;
    @Mock RelationshipFindOrCreateService findOrCreate;
    @Mock RelationshipFetchService fetchService;
    @Mock StageOverrideQueryService stageOverride;
    @Mock RelationshipRepository relRepo;
    @Mock StageDefinition stageDef;
    @InjectMocks CharacterApiImpl impl;

    @Test
    void findOrCreateRelationship_should_delegate_and_return_dto() {
        var rel = RelationshipTestFixtures.relationshipWithScore(1L, 1L, 0);
        when(findOrCreate.findOrCreate(1L, 1L)).thenReturn(rel);
        when(stageDef.nameOf(0)).thenReturn("陌生人");
        when(stageDef.nextThresholdOf(0)).thenReturn(100);

        var dto = impl.findOrCreateRelationship(1L, 1L);

        assertThat(dto.intimacyScore()).isZero();
        assertThat(dto.currentStageName()).isEqualTo("陌生人");
    }

    @Test
    void fetchMyRelationship_should_delegate() {
        var expected = new RelationshipDto(1L, 1L, 50, 0, "陌生人", 100, 0.5);
        when(fetchService.fetchMyRelationship(1L, 1L)).thenReturn(expected);
        assertThat(impl.fetchMyRelationship(1L, 1L)).isSameAs(expected);
    }

    @Test
    void getStagePromptSegment_should_findOrCreate_then_call_override_service() {
        var rel = RelationshipTestFixtures.relationshipAtStage(1L, 1L, 2, 400);
        when(findOrCreate.findOrCreate(1L, 1L)).thenReturn(rel);
        when(stageOverride.buildSegment(1L, 2)).thenReturn("当前关系阶段：暧昧。称呼：宝。");

        assertThat(impl.getStagePromptSegment(1L, 1L)).contains("暧昧");
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@Service
@RequiredArgsConstructor
public class CharacterApiImpl implements CharacterApi {
    private final AiCharacterRepository charRepo;
    private final RelationshipFindOrCreateService findOrCreate;
    private final RelationshipFetchService fetchService;
    private final StageOverrideQueryService stageOverride;
    private final StageDefinition stageDef;

    @Override
    public AiCharacterDto findById(Long id) {
        return charRepo.findById(id).map(this::toDto).orElse(null);
    }
    @Override
    public AiCharacterDto getById(Long id) {
        return charRepo.findById(id).map(this::toDto)
            .orElseThrow(() -> new BusinessException(CharacterErrCode.CHARACTER_NOT_FOUND));
    }

    @Override
    public RelationshipDto findOrCreateRelationship(Long userId, Long characterId) {
        RelationshipEntity rel = findOrCreate.findOrCreate(userId, characterId);
        return toRelationshipDto(userId, characterId, rel);
    }

    @Override
    public RelationshipDto fetchMyRelationship(Long userId, Long characterId) {
        return fetchService.fetchMyRelationship(userId, characterId);
    }

    @Override
    public String getStagePromptSegment(Long userId, Long characterId) {
        RelationshipEntity rel = findOrCreate.findOrCreate(userId, characterId);
        return stageOverride.buildSegment(characterId, rel.getCurrentStage());
    }

    private AiCharacterDto toDto(AiCharacterEntity e) {
        return new AiCharacterDto(e.getId(), e.getName(), e.getAvatar(), e.getCreatedAt());
    }
    private RelationshipDto toRelationshipDto(Long uid, Long cid, RelationshipEntity rel) {
        int stage = rel.getCurrentStage();
        int nextThreshold = stageDef.nextThresholdOf(stage);
        int prevThreshold = stage == 0 ? 0 : stageDef.nextThresholdOf(stage - 1);
        double percent = nextThreshold == Integer.MAX_VALUE ? 1.0
            : (double)(rel.getIntimacyScore() - prevThreshold) / (nextThreshold - prevThreshold);
        return new RelationshipDto(uid, cid, rel.getIntimacyScore(), stage,
            stageDef.nameOf(stage), nextThreshold, percent);
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): CharacterApiImpl 扩 4 个 relationship 方法（薄委托）"
```

---

### Phase I · Controller + chat-core 接入

#### Task I1: RelationshipController GET /api/relationships/me

**Files:**
- Create: `web/RelationshipController.java`
- Test: `test/.../web/RelationshipControllerIT.java`

- [ ] **Step 1: Write the failing test**

```java
@WebMvcTest(RelationshipController.class)
@Import(WebTestSecurityConfig.class)
class RelationshipControllerIT {
    @Autowired MockMvc mvc;
    @MockBean CharacterApi characterApi;

    @Test
    @WithMockUser(username = "42")
    void get_me_should_return_dto() throws Exception {
        when(characterApi.fetchMyRelationship(42L, 1L))
            .thenReturn(new RelationshipDto(42L, 1L, 150, 1, "朋友", 300, 0.25));

        mvc.perform(get("/api/relationships/me"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data.intimacyScore").value(150))
            .andExpect(jsonPath("$.data.currentStageName").value("朋友"));
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
@RestController
@RequestMapping("/api/relationships")
@RequiredArgsConstructor
public class RelationshipController {
    private static final Long DEFAULT_CHARACTER_ID = 1L;

    private final CharacterApi characterApi;

    @LoginRequired
    @GetMapping("/me")
    public RelationshipDto fetchMyRelationship() {
        Long userId = CurrentUser.id();
        return characterApi.fetchMyRelationship(userId, DEFAULT_CHARACTER_ID);
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-character-core/
git commit -m "feat(character-core): GET /api/relationships/me"
```

---

#### Task I2: PromptBuilder 加 stagePromptSegment 参数

**Files:**
- Modify: `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/PromptBuilder.java`
- Modify/Test: `test/.../internal/PromptBuilderTest.java`

- [ ] **Step 1: Write the failing test**

```java
class PromptBuilderTest {
    PromptBuilder builder = new PromptBuilder();

    @Test
    void should_insert_stage_segment_between_character_and_memory() {
        var result = builder.build(
            "你是小婉",
            "当前关系阶段：暧昧。称呼：宝。",
            MemoryContext.empty(),
            List.of()
        );

        assertThat(result).hasSize(2);
        assertThat(result.get(0).get("content")).isEqualTo("你是小婉");
        assertThat(result.get(1).get("content")).startsWith("当前关系阶段：暧昧");
    }

    @Test
    void should_skip_stage_segment_when_blank() {
        var result = builder.build("你是小婉", null, MemoryContext.empty(), List.of());
        assertThat(result).hasSize(1);
    }

    @Test
    void should_skip_stage_segment_when_blank_string() {
        var result = builder.build("你是小婉", "  ", MemoryContext.empty(), List.of());
        assertThat(result).hasSize(1);
    }
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```java
public List<Map<String, String>> build(
        String characterPrompt,
        String stagePromptSegment,
        MemoryContext memoryContext,
        List<MessageEntity> recentMessages) {

    List<Map<String, String>> messages = new ArrayList<>();

    if (characterPrompt != null && !characterPrompt.isBlank()) {
        messages.add(Map.of("role", "system", "content", characterPrompt));
    }
    if (stagePromptSegment != null && !stagePromptSegment.isBlank()) {
        messages.add(Map.of("role", "system", "content", stagePromptSegment));
    }
    if (memoryContext != null && !memoryContext.isEmpty()) {
        messages.add(Map.of("role", "system", "content", MEMORY_PREFIX + memoryContext.text()));
    }
    List<MessageEntity> limited = limitToWindow(recentMessages);
    for (MessageEntity msg : limited) {
        String role = SenderType.USER.equals(msg.getSenderType()) ? "user" : "assistant";
        String content = msg.getContent() == null ? "" : msg.getContent();
        messages.add(Map.of("role", role, "content", content));
    }
    return messages;
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-chat-core/
git commit -m "feat(chat-core): PromptBuilder 加 stagePromptSegment 参数（非空插入到 character 后 memory 前）"
```

---

#### Task I3: AiService 调用 getStagePromptSegment 拼好传 PromptBuilder

**Files:**
- Modify: `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/AiService.java`
- Modify/Test: `test/.../internal/AiServiceTest.java`（已有 test，扩展）

- [ ] **Step 1: Write the failing test**

```java
@Test
void handleUserMessage_should_inject_stage_prompt_from_character_api() {
    when(characterApi.getStagePromptSegment(42L, 1L))
        .thenReturn("当前关系阶段：朋友。称呼：你。");

    aiService.handleUserMessage(42L, 1L, "你好");

    verify(promptBuilder).build(
        anyString(),
        eq("当前关系阶段：朋友。称呼：你。"),
        any(),
        anyList()
    );
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation（AiService 中调用处）**

```java
public void handleUserMessage(Long userId, Long characterId, String content) {
    AiCharacterDto character = characterApi.getById(characterId);
    String stagePromptSegment = characterApi.getStagePromptSegment(userId, characterId);
    MemoryContext memoryContext = memoryApi.buildContext(userId, characterId, content);
    List<MessageEntity> recent = messageRepo.findTop32ByUserIdOrderByCreatedAtDesc(userId);

    List<Map<String, String>> messages = promptBuilder.build(
        character.basePrompt(), stagePromptSegment, memoryContext, recent);

    // ... 后续走 LLM + persist 不变
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-chat-core/
git commit -m "feat(chat-core): AiService 调 characterApi.getStagePromptSegment 注入 prompt"
```

---

#### Task I4: ChatWebSocketHandler 监听事件 + WS 推送

**Files:**
- Modify: `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java`
- Modify/Test: `test/.../ws/ChatWebSocketHandlerTest.java`

- [ ] **Step 1: Write the failing test**

```java
@Test
void should_push_intimacy_update_on_event() {
    var session = mock(WebSocketSession.class);
    handler.registerSession(42L, session);

    handler.onIntimacyChanged(new IntimacyChangedEvent(42L, 1L, 100, 110, 10, "MESSAGE_SENT"));

    verify(session).sendMessage(argThat(msg -> {
        String json = ((TextMessage) msg).getPayload();
        return json.contains("intimacy_update") && json.contains("\"delta\":10");
    }));
}

@Test
void should_push_stage_transition() throws Exception {
    var session = mock(WebSocketSession.class);
    handler.registerSession(42L, session);

    handler.onStageTransition(new StageTransitionEvent(42L, 1L, 0, 1));

    verify(session).sendMessage(argThat(msg ->
        ((TextMessage) msg).getPayload().contains("stage_transition")));
}

@Test
void should_push_story_message_on_stage_entry_story() throws Exception {
    var session = mock(WebSocketSession.class);
    handler.registerSession(42L, session);

    handler.onStageEntryStory(new StageEntryStoryEvent(42L, 1L, 2, "她半夜……"));

    verify(session).sendMessage(argThat(msg -> {
        String json = ((TextMessage) msg).getPayload();
        return json.contains("stage_story") && json.contains("她半夜");
    }));
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation（在 handler 内加 3 个监听 + 推送方法）**

```java
@EventListener
public void onIntimacyChanged(IntimacyChangedEvent event) {
    pushToUser(event.userId(), Map.of(
        "type", "intimacy_update",
        "score", event.newScore(),
        "delta", event.delta(),
        "reason", event.reason()
    ));
}

@EventListener
public void onStageTransition(StageTransitionEvent event) {
    pushToUser(event.userId(), Map.of(
        "type", "stage_transition",
        "from", event.fromStage(),
        "to", event.toStage()
    ));
}

@EventListener
public void onStageEntryStory(StageEntryStoryEvent event) {
    pushToUser(event.userId(), Map.of(
        "type", "stage_story",
        "stage", event.toStage(),
        "story_message", event.storyMessage()
    ));
}

private void pushToUser(Long userId, Map<String, Object> payload) {
    WebSocketSession session = sessions.get(userId);
    if (session == null || !session.isOpen()) return;
    try {
        session.sendMessage(new TextMessage(mapper.writeValueAsString(payload)));
    } catch (IOException e) {
        log.warn("WS 推送失败 userId={}", userId, e);
    }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A server/business_packages/sanyan-chat-core/
git commit -m "feat(chat-core): WS 推送 intimacy_update / stage_transition / stage_story 三种帧"
```

---

### Phase J · 前端

#### Task J1: Relationship model + relationship_req

**Files:**
- Create: `app/business_packages/sanyan_chat/lib/src/api/models/relationship.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/api/req/relationship_req.dart`
- Test: `app/business_packages/sanyan_chat/test/api/relationship_req_test.dart`

- [ ] **Step 1: Write the failing test**

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_chat/src/api/models/relationship.dart';
import 'package:sanyan_chat/src/api/req/relationship_req.dart';

void main() {
  test('Relationship.fromJson parses all fields', () {
    final r = Relationship.fromJson({
      'userId': 1, 'characterId': 1,
      'intimacyScore': 250, 'currentStage': 1,
      'currentStageName': '朋友', 'nextStageThreshold': 300,
      'percentToNextStage': 0.75,
    });
    expect(r.intimacyScore, 250);
    expect(r.currentStageName, '朋友');
    expect(r.percentToNextStage, 0.75);
  });

  test('FetchMyRelationshipData wraps Relationship', () {
    final data = FetchMyRelationshipData.fromJson({
      'userId': 1, 'characterId': 1, 'intimacyScore': 100,
      'currentStage': 1, 'currentStageName': '朋友',
      'nextStageThreshold': 300, 'percentToNextStage': 0.0,
    });
    expect(data.relationship.intimacyScore, 100);
  });
}
```

- [ ] **Step 2: Run to verify FAIL**

```bash
cd app && fvm flutter test business_packages/sanyan_chat/test/api/relationship_req_test.dart
```

- [ ] **Step 3: Write minimal implementation**

```dart
// relationship.dart
class Relationship {
  final int userId;
  final int characterId;
  final int intimacyScore;
  final int currentStage;
  final String currentStageName;
  final int nextStageThreshold;
  final double percentToNextStage;

  const Relationship({
    required this.userId,
    required this.characterId,
    required this.intimacyScore,
    required this.currentStage,
    required this.currentStageName,
    required this.nextStageThreshold,
    required this.percentToNextStage,
  });

  factory Relationship.fromJson(Map<String, dynamic> json) => Relationship(
        userId: json['userId'] as int,
        characterId: json['characterId'] as int,
        intimacyScore: json['intimacyScore'] as int,
        currentStage: json['currentStage'] as int,
        currentStageName: json['currentStageName'] as String,
        nextStageThreshold: json['nextStageThreshold'] as int,
        percentToNextStage: (json['percentToNextStage'] as num).toDouble(),
      );

  Relationship copyWith({int? intimacyScore, int? currentStage, String? currentStageName,
                          int? nextStageThreshold, double? percentToNextStage}) =>
      Relationship(
        userId: userId, characterId: characterId,
        intimacyScore: intimacyScore ?? this.intimacyScore,
        currentStage: currentStage ?? this.currentStage,
        currentStageName: currentStageName ?? this.currentStageName,
        nextStageThreshold: nextStageThreshold ?? this.nextStageThreshold,
        percentToNextStage: percentToNextStage ?? this.percentToNextStage,
      );
}
```

```dart
// relationship_req.dart
class FetchMyRelationshipReq {
  Map<String, dynamic> parameters() => {};
}

class FetchMyRelationshipData {
  final Relationship relationship;
  FetchMyRelationshipData(this.relationship);

  factory FetchMyRelationshipData.fromJson(Map<String, dynamic> json) =>
      FetchMyRelationshipData(Relationship.fromJson(json));
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A app/business_packages/sanyan_chat/
git commit -m "feat(chat-api): Relationship model + FetchMyRelationshipReq/Data"
```

---

#### Task J2: sanyan_chat_api 加 fetchMyRelationship 方法

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/api/sanyan_chat_api.dart`
- Modify/Test: `app/business_packages/sanyan_chat/test/api/sanyan_chat_api_test.dart`

- [ ] **Step 1: Write the failing test**

```dart
test('fetchMyRelationship returns Relationship from API', () async {
  final api = SanyanChatApi(network: mockNetwork);
  when(() => mockNetwork.get(any(), parameters: any(named: 'parameters')))
      .thenAnswer((_) async => BaseResp.success({
            'userId': 1, 'characterId': 1, 'intimacyScore': 100,
            'currentStage': 1, 'currentStageName': '朋友',
            'nextStageThreshold': 300, 'percentToNextStage': 0.0,
          }));

  final rel = await api.fetchMyRelationship();
  expect(rel.currentStageName, '朋友');
});
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation（加方法到 sanyan_chat_api.dart）**

```dart
Future<Relationship> fetchMyRelationship() async {
  final resp = await network.get('/api/relationships/me',
      parameters: FetchMyRelationshipReq().parameters());
  if (!resp.success) {
    Toast.show(resp.message);
    throw Exception(resp.message);
  }
  return FetchMyRelationshipData.fromJson(resp.data as Map<String, dynamic>).relationship;
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A app/business_packages/sanyan_chat/
git commit -m "feat(chat-api): sanyan_chat_api 加 fetchMyRelationship 聚合方法"
```

---

#### Task J3: IntimacyProgressBar widget

**Files:**
- Create: `app/business_packages/sanyan_chat/lib/src/chat/widgets/intimacy_progress_bar.dart`
- Test: `app/business_packages/sanyan_chat/test/chat/widgets/intimacy_progress_bar_test.dart`

- [ ] **Step 1: Write the failing test**

```dart
void main() {
  testWidgets('IntimacyProgressBar renders stage name and percent', (tester) async {
    final rel = Relationship(
      userId: 1, characterId: 1, intimacyScore: 250, currentStage: 1,
      currentStageName: '朋友', nextStageThreshold: 300, percentToNextStage: 0.75,
    );
    await tester.pumpWidget(MaterialApp(home: Scaffold(
      body: IntimacyProgressBar(relationship: rel, onTap: () {}),
    )));

    expect(find.text('朋友'), findsOneWidget);
    expect(find.text('250 / 300'), findsOneWidget);
    final fraction = tester.widget<FractionallySizedBox>(find.byType(FractionallySizedBox));
    expect(fraction.widthFactor, closeTo(0.75, 0.001));
  });

  testWidgets('IntimacyProgressBar shows 1.0 fill on old_couple stage', (tester) async {
    final rel = Relationship(
      userId: 1, characterId: 1, intimacyScore: 9999, currentStage: 4,
      currentStageName: '老夫老妻', nextStageThreshold: 2147483647, percentToNextStage: 1.0,
    );
    await tester.pumpWidget(MaterialApp(home: Scaffold(
      body: IntimacyProgressBar(relationship: rel, onTap: () {}),
    )));
    expect(find.text('老夫老妻'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```dart
import 'package:flutter/material.dart';
import '../../api/models/relationship.dart';

class IntimacyProgressBar extends StatelessWidget {
  final Relationship relationship;
  final VoidCallback onTap;

  const IntimacyProgressBar({
    super.key,
    required this.relationship,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    final percent = relationship.percentToNextStage.clamp(0.0, 1.0);
    return GestureDetector(
      onTap: onTap,
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(relationship.currentStageName,
                    style: const TextStyle(fontSize: 13, fontWeight: FontWeight.w600)),
                Text('${relationship.intimacyScore} / ${relationship.nextStageThreshold}',
                    style: const TextStyle(fontSize: 12, color: Colors.grey)),
              ],
            ),
            const SizedBox(height: 6),
            ClipRRect(
              borderRadius: BorderRadius.circular(4),
              child: Container(
                height: 6,
                color: Colors.black12,
                child: FractionallySizedBox(
                  widthFactor: percent,
                  alignment: Alignment.centerLeft,
                  child: Container(
                    decoration: const BoxDecoration(
                      gradient: LinearGradient(colors: [Color(0xFFFF8FB7), Color(0xFFA374FF)]),
                    ),
                  ),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A app/business_packages/sanyan_chat/
git commit -m "feat(chat-ui): IntimacyProgressBar widget（粉紫渐变 + 阶段名 + 进度）"
```

---

#### Task J4: StageTransitionDialog widget

**Files:**
- Create: `app/business_packages/sanyan_chat/lib/src/chat/widgets/stage_transition_dialog.dart`
- Test: `app/business_packages/sanyan_chat/test/chat/widgets/stage_transition_dialog_test.dart`

- [ ] **Step 1: Write the failing test**

```dart
testWidgets('StageTransitionDialog shows story_message and dismisses on tap', (tester) async {
  await tester.pumpWidget(MaterialApp(home: Builder(builder: (ctx) {
    return TextButton(
      onPressed: () => showStageTransitionDialog(ctx,
          fromStage: 1, toStage: 2, storyMessage: '她半夜悄悄想你……'),
      child: const Text('open'));
  })));

  await tester.tap(find.text('open'));
  await tester.pumpAndSettle();

  expect(find.text('她半夜悄悄想你……'), findsOneWidget);

  await tester.tap(find.byType(GestureDetector).last);
  await tester.pumpAndSettle();
  expect(find.text('她半夜悄悄想你……'), findsNothing);
});
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation**

```dart
import 'package:flutter/material.dart';

Future<void> showStageTransitionDialog(BuildContext context, {
  required int fromStage,
  required int toStage,
  required String storyMessage,
}) {
  return showDialog(
    context: context,
    barrierDismissible: true,
    builder: (_) => GestureDetector(
      onTap: () => Navigator.of(context).pop(),
      child: Material(
        color: Colors.black.withOpacity(0.6),
        child: Center(
          child: AnimatedScale(
            scale: 1.0,
            duration: const Duration(milliseconds: 300),
            child: Container(
              padding: const EdgeInsets.all(24),
              margin: const EdgeInsets.symmetric(horizontal: 32),
              decoration: BoxDecoration(
                color: Colors.white,
                borderRadius: BorderRadius.circular(16),
              ),
              child: Column(mainAxisSize: MainAxisSize.min, children: [
                const Icon(Icons.favorite, color: Color(0xFFFF8FB7), size: 48),
                const SizedBox(height: 16),
                Text(storyMessage,
                    textAlign: TextAlign.center,
                    style: const TextStyle(fontSize: 16, height: 1.5)),
              ]),
            ),
          ),
        ),
      ),
    ),
  );
}
```

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A app/business_packages/sanyan_chat/
git commit -m "feat(chat-ui): StageTransitionDialog（全屏遮罩 + 卡片动画 + story_message）"
```

---

#### Task J5: chat_controller 监听 ws frame + Rx<Relationship>

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`
- Modify/Test: `app/business_packages/sanyan_chat/test/chat/chat_controller_intimacy_test.dart`

- [ ] **Step 1: Write the failing test**

```dart
test('chat_controller handles intimacy_update frame', () {
  final c = ChatController(...);
  c.handleWsFrame({
    'type': 'intimacy_update', 'score': 120, 'delta': 5, 'reason': 'MESSAGE_SENT',
  });
  expect(c.relationship.value!.intimacyScore, 120);
});

test('chat_controller handles stage_story frame by showing dialog', () {
  final c = ChatController(...);
  c.handleWsFrame({
    'type': 'stage_story', 'stage': 2, 'story_message': '她半夜……',
  });
  expect(c.pendingStoryMessage.value, '她半夜……');
});
```

- [ ] **Step 2: Run to verify FAIL**

- [ ] **Step 3: Write minimal implementation（修改 chat_controller.dart）**

```dart
final Rx<Relationship?> relationship = Rx<Relationship?>(null);
final RxString pendingStoryMessage = ''.obs;

@override
void onInit() {
  super.onInit();
  fetchInitialRelationship();
  // ws 已有 onMessage 接入；这里在 onMessage 内分发 handleWsFrame
}

Future<void> fetchInitialRelationship() async {
  try {
    relationship.value = await chatApi.fetchMyRelationship();
  } catch (_) {/* 已 toast */}
}

void handleWsFrame(Map<String, dynamic> frame) {
  switch (frame['type']) {
    case 'intimacy_update':
      final old = relationship.value;
      if (old == null) return;
      relationship.value = old.copyWith(
        intimacyScore: frame['score'] as int,
      );
      // 注：percent/threshold 由下次 fetchMyRelationship 或服务端推送时一并刷新
      break;
    case 'stage_transition':
      // 后端会紧接着发 stage_story 帧，本帧仅做埋点 / 日志
      break;
    case 'stage_story':
      pendingStoryMessage.value = frame['story_message'] as String;
      break;
  }
}
```

页面 `chat_page.dart` 中通过 `Obx` + `ever(pendingStoryMessage, ...)` 监听并调 `showStageTransitionDialog` 弹窗。

- [ ] **Step 4: Run to verify PASS**

- [ ] **Step 5: Commit**

```bash
git add -A app/business_packages/sanyan_chat/
git commit -m "feat(chat-ui): chat_controller 监听 intimacy_update / stage_story / stage_transition frame"
```

---

#### Task J6: sanyan_chat_suite.dart 聚合新测试

**Files:**
- Modify: `app/business_packages/sanyan_chat/test/sanyan_chat_suite.dart`

- [ ] **Step 1: Write the failing test** —— 本 task 是测试聚合，验证 suite 入口能 import 全部新测试。

```dart
import 'package:flutter_test/flutter_test.dart';
import 'api/relationship_req_test.dart' as relationship_req_test;
import 'api/sanyan_chat_api_test.dart' as chat_api_test;
import 'chat/widgets/intimacy_progress_bar_test.dart' as progress_bar_test;
import 'chat/widgets/stage_transition_dialog_test.dart' as transition_dialog_test;
import 'chat/chat_controller_intimacy_test.dart' as controller_intimacy_test;

void main() {
  group('relationship_req', relationship_req_test.main);
  group('sanyan_chat_api', chat_api_test.main);
  group('intimacy_progress_bar', progress_bar_test.main);
  group('stage_transition_dialog', transition_dialog_test.main);
  group('chat_controller_intimacy', controller_intimacy_test.main);

  // 已有测试 imports 保留
}
```

- [ ] **Step 2: Run** —— 直接跑 suite 确认所有 group 都跑

```bash
cd app && fvm flutter test business_packages/sanyan_chat/test/sanyan_chat_suite.dart
```

- [ ] **Step 3-4: 调整 import 路径直到 PASS**

- [ ] **Step 5: Commit**

```bash
git add app/business_packages/sanyan_chat/test/sanyan_chat_suite.dart
git commit -m "test(chat): sanyan_chat_suite 聚合 Plan 3 新增 5 个测试入口"
```

---

### Phase K · 端到端 IT + dogfood + 整体 Review

#### Task K1: MessageFlowE2EIT（端到端 final gate）

**Files:**
- Create: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/MessageFlowE2EIT.java`

- [ ] **Step 1: Write the failing test**

```java
@SpringBootTest @AutoConfigureMockMvc @Testcontainers
class MessageFlowE2EIT {
    @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17-alpine");
    @Container static GenericContainer<?>    redis = new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url",      pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
        r.add("spring.data.redis.host",     redis::getHost);
        r.add("spring.data.redis.port",     () -> redis.getMappedPort(6379));
    }

    @Autowired ApplicationEventPublisher publisher;
    @Autowired RelationshipRepository relRepo;
    @Autowired IntimacyLogRepository logRepo;

    @Test
    void user_message_should_accumulate_intimacy_and_trigger_stage_event() throws Exception {
        // 模拟 chat-core 发 MessagePersistedEvent
        publisher.publishEvent(new MessagePersistedEvent(1L, 42L, 1L, "user", "你好小婉", Instant.now()));

        // AFTER_COMMIT @Async 异步，等一下
        Awaitility.await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            var rel = relRepo.findByUserIdAndCharacterId(42L, 1L);
            assertThat(rel).isPresent();
            assertThat(rel.get().getIntimacyScore()).isGreaterThan(0);

            var logs = logRepo.findTop10ByUserIdAndCharacterIdOrderByCreatedAtDesc(42L, 1L);
            assertThat(logs).isNotEmpty();
            assertThat(logs.get(0).getReason()).isEqualTo("MESSAGE_SENT");
        });
    }
}
```

- [ ] **Step 2-4: 跑 → 修测试桩 → 通过**

- [ ] **Step 5: Commit**

```bash
git add -A server/
git commit -m "test(character): MessageFlowE2EIT 端到端链路（发消息 → 涨分 → DB 状态）"
```

---

#### Task K2: 7 天 dogfood + 数值调参

**Files:**
- Create: `docs/dogfood/plan-3-day-1.md` 到 `plan-3-day-7.md`
- Modify: `application.yml`（数值微调）

- [ ] **Step 1: 每天 dogfood 后写一篇 day-N.md**

按 spec §8.1 的 4 维度（速度感 / 剧情演出 / 透明性 / 称呼语调）写感受。

- [ ] **Step 2: 第 3 天和第 7 天分别调一次数值**

`application.yml` 改 sanyan.intimacy.* 任一系数，记到当天 day-N.md：「day3 改 X：1→2，理由：……」

- [ ] **Step 3: 在 plan-3.md 末尾追加「dogfood 反馈 + 最终数值」段**

- [ ] **Step 4: 跑全量回归**

```bash
mvn verify
yes | fvm flutter test
```

- [ ] **Step 5: Commit**

```bash
git add docs/dogfood/ server/bootstrap/src/main/resources/application.yml docs/superpowers/plans/2026-05-06-3yan-plan-3-relationship-progression.md
git commit -m "docs(plan-3): 7 天 dogfood 反馈 + 数值调参"
```

---

#### Task K3: 整体对照 plan review（你要求加的最终 task）

**Goal：** 实现完成、所有测试通过、dogfood 跑完之后，由 Claude 自己亲自把整个 plan 从头过一遍，对照已实现的代码和测试，检查：

- [ ] **Step 1: 对照每一个 phase 的每一个 task**

逐条对照 `docs/superpowers/plans/2026-05-06-3yan-plan-3-relationship-progression.md` 的 §1-§8 spec 和 Phase A-K 所有 task，逐条勾选：

- 该 task 的 Files 列表中，每个文件都实际存在 / 实际修改了？
- 该 task 的测试代码实际写了吗？跑了吗？过了吗？
- 该 task 的 commit 实际发生了吗？

- [ ] **Step 2: 对照 spec 里 §3 模块结构 + §6 前端组件**

逐个文件名查现状（`ls` + `rg`），确认所有应该存在的文件都存在；命名一致（IntimacyRecordService 不是 IntimacyService 等）。

- [ ] **Step 3: 对照规范规则**

重新跑一遍 `~/.claude/rules/java-backend*.md` 和 `flutter*.md` 的关键约束：
- 一个 -api 一个主 Api？✓
- Service 按动作命名？✓
- Object Mother Fixture 三个新 entity 都有？✓
- ERROR_CODE_REGISTRY.md 同步了 3002 / 3003？✓
- flutter req/ 不是 reqs/？✓
- sanyan_chat_suite 聚合了新测试？✓

- [ ] **Step 4: 跑全量测试 final gate**

```bash
mvn verify
yes | fvm flutter test
```

确认全绿。

- [ ] **Step 5: 出一份 final-review.md**

写到 `docs/superpowers/plans/2026-05-06-3yan-plan-3-relationship-progression-final-review.md`，列出：

1. 已完成的所有 task ID（Phase A1-K2）
2. 任何**实际实现与 plan 不一致**的地方（命名变化 / 临时增删的文件 / 数值变化）+ 原因说明
3. 任何**未达成的 DoD 条款**（如有）+ 后续 follow-up plan

最后 commit：

```bash
git add docs/superpowers/plans/2026-05-06-3yan-plan-3-relationship-progression-final-review.md
git commit -m "docs(plan-3): 整体对照 plan 的 final review（K3）"
```

---

## Self-Review（writing-plans 阶段自检）

按 writing-plans skill 要求，本 plan 写完后自检：

1. **Spec coverage：** §1 设计概览 → 散落 phase 中；§2 数据模型 → Phase A + B；§3 模块结构 → Phase B-I；§4 数据流 → Phase D5 + G2 + I3 + I4；§5 5 阶段 + 涨分 → Phase C1 + D2-D5；§6 前端 → Phase J；§7 测试 → 各 task 内置 + Phase K1；§8 dogfood → Phase K2；K3 整体 review 用户特别要求。
2. **Placeholder scan：** 无 TBD / TODO / "见 task N"占位。
3. **Type consistency：** 全文 `IntimacyRecordService`（非 IntimacyService）、`StageTransitionDetectService`、`StageOverrideQueryService`、`CharacterApi` 含 4 新方法、`req/`（非 reqs/）均一致。
4. **Ambiguity：** Task G1 与 Task C2 有循环引用，已在 G1 头部注明推荐执行顺序「A → B → G1 → C → D → E → F → G2 → H → I → J → K」。

---

## Execution Handoff

- **前置依赖：** Plan 1（MVP+WS） + Plan 2（长期记忆） + S3（business 模块拆分）全部完成 ✅
- **当前分支：** `plan3`（git worktree at `../3yan-plan3`，主仓 + app/server/web 三个 submodule 都在 plan3 分支）
- **下一步：** 选择执行方式 → 通过 `superpowers:subagent-driven-development`（推荐：每 task 一个新子代理 + 两轮审查）或 `superpowers:executing-plans`（当前会话 inline 跑）逐 task 实现
