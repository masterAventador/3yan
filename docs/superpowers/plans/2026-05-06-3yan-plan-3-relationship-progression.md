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
- **称呼/语调动态**：每个 stage 在 `character.persona_config.stage_overrides` 里有 `address` / `tone_hint` / `topics_unlock`；StageOverrideService 取出 → 拼成 stage prompt segment 文本 → 由 chat-core AiService 传给 PromptBuilder
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
├── CharacterApi.java                       # 已有
├── RelationshipApi.java                    # 新增 — findOrCreate / getStagePromptSegment
├── dto/
│   ├── AiCharacterDto.java                 # 已有
│   └── RelationshipDto.java                # 新增
└── event/
    ├── IntimacyChangedEvent.java           # 新增 (record)
    └── StageTransitionEvent.java           # 新增 (record)
```

`RelationshipApi` 草签：
```java
public interface RelationshipApi {
    RelationshipDto findOrCreate(Long userId, Long characterId);

    /** 返回拼好的 stage prompt 片段，供 chat-core 注入到 system 消息。
     *  示例返回："当前关系阶段：暧昧。称呼用户用：宝。语调：撒娇、试探、害羞。" */
    String getStagePromptSegment(Long userId, Long characterId);
}
```

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
│   ├── CharacterErrCode.java               # 已有 — 增 RELATIONSHIP_NOT_FOUND 等
│   ├── RelationshipEntity.java             # 新增（@IdClass 复合 PK + @Version）
│   ├── RelationshipRepository.java         # 新增
│   ├── intimacy/
│   │   ├── IntimacyEvent.java              # record (type, payload)
│   │   ├── IntimacyCalculator.java
│   │   ├── IntimacyService.java            # 累加 + 阶段切换 + 事件
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
│       ├── StageTransitionService.java     # 检测阶段切换 + 发事件
│       └── StageOverrideService.java       # 读 persona_config.stage_overrides → 拼 prompt 段
├── api/
│   ├── CharacterApiImpl.java               # 已有
│   └── RelationshipApiImpl.java            # 新增
├── web/
│   └── RelationshipController.java         # GET /api/relationships/me
└── event/
    ├── MessagePersistedListener.java       # 订阅 chat-api 的 MessagePersistedEvent
    └── StageTransitionStoryListener.java   # 订阅自家 StageTransitionEvent → 拼 story_message（不加分；通过 IntimacyChangedEvent 元数据或单独事件下发给 chat-core WS）
```

### 3.3 sanyan-chat-core（仅 3 处接入）

```
internal/PromptBuilder.java         # build 多 stagePromptSegment 参数
internal/AiService.java             # 调 RelationshipApi.getStagePromptSegment 拼好传入
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

### 3.4 跨模块边界守则

- character-core **订阅** chat-api 的 `MessagePersistedEvent`（已存在，Plan 2 N3 加的）
- chat-core **调用** character-api 的 `RelationshipApi.getStagePromptSegment`（拿字符串，不引入 Relationship 类型依赖）
- chat-core **订阅** character-api 的 `IntimacyChangedEvent / StageTransitionEvent` → WS 推送
- 双向依赖通过 event + API 解耦，符合 java-backend.md 跨模块通信规则

---

## 4 · 数据流

### 4.1 链路 A：用户发消息 → 涨分 → 推送 → UI 刷新

1. **[App]** 用户发消息 → WS frame
2. **[chat-core · AiService]**
   1. `relationshipApi.getStagePromptSegment(uid, cid)` → "当前关系：暧昧。称呼：宝。语调：撒娇..."（内部走 findOrCreate）
   2. `PromptBuilder.build(charPrompt, stagePromptSegment, memoryContext, recent)`
   3. LLM 调用 → 流式回写 WS
   4. `MessageService.persist(...)` → DB COMMIT → `publishEvent(MessagePersistedEvent)`
3. **[character-core · MessagePersistedListener]**（`@TransactionalEventListener(phase = AFTER_COMMIT)` + `@Async`）
   - `intimacyService.recordEvent(MESSAGE_SENT)` —— +1 ~ +10（受每日封顶限）
   - `plotEngine.evaluate(context)` → 0..N rule 命中 → `recordEvent(PLOT_MILESTONE, +30~+100)`
   - 累计每 10 条用户消息 → `evaluator.score()` → `recordEvent(AI_QUALITY_BONUS, 0~20)`
4. **[character-core · IntimacyService.recordEvent]**
   1. `calculator.compute(event)` → delta
   2. `UPDATE relationships SET intimacy_score+=delta WHERE ... AND version=?`（乐观锁，retry 3 次）
   3. `INSERT INTO intimacy_logs ...`
   4. `stageTransitionService.maybeTransition(rel, newScore)`
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

1. `relationshipApi.findOrCreate(userId, characterId)`（唯一懒创建入口；当日首次访问时建 relationship 行）
2. `consecutiveLoginService.recordLogin(userId)` 比对 Redis last_date
   - 今天已记 → return streak（不涨分）
   - 今天首次 → streak++（或重置为 1） + 写 Redis → `intimacyService.recordEvent(DAILY_LOGIN, streak)` —— +10 + streak×5（封顶 +60）
3. 接口返回 `RelationshipDto`

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
│   ├── models/relationship.dart            # 新增
│   └── reqs/relationship_req.dart          # 新增 — fetchMyRelationship + 解析 ws frame
└── chat/
    ├── widgets/
    │   ├── intimacy_progress_bar.dart      # 顶部进度条
    │   └── stage_transition_dialog.dart    # 阶段切换庆祝弹窗
    └── chat_controller.dart                # 修改 — 监听 ws intimacy_update / stage_transition
```

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

- **intimacy/**：`IntimacyCalculatorTest`（6 种 type × 边界） / `IntimacyServiceTest`（累加 / 乐观锁 retry / 跨阶段 / cap 命中 delta=0 但仍写 log）
- **stage/**：`StageDefinitionTest`（0/99/100/300/600/1000 边界） / `StageTransitionServiceTest` / `StageOverrideServiceTest`（persona_config 解析）
- **plotrule/**：每个 rule 一个 Test（覆盖触发 / 不触发 / 去重） + `PlotMilestoneEngineTest`
- **intimacy/ai/**：`ConversationQualityEvaluatorTest`（LLM fake JSON 解析）
- **event/**：`MessagePersistedListenerTest`（链路 mock）

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

> Task 拆分将由 `superpowers:writing-plans` skill 在本 spec 通过后生成，追加到本文件末尾。

---

## Execution Handoff

- **前置依赖：** Plan 1（MVP+WS） + Plan 2（长期记忆） + S3（business 模块拆分）全部完成 ✅
- **当前分支：** `plan3`（git worktree at `../3yan-plan3`）
- **下一步：** spec 通过后 → `superpowers:writing-plans` → `subagent-driven-development`
