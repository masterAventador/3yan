# 三言 · Plan 3 实施计划：关系养成系统（5 阶段 + 亲密度引擎）

> **For agentic workers:** REQUIRED SUB-SKILL: `superpowers:subagent-driven-development`. Steps use `- [ ]` checkboxes.
>
> **TDD 铁律：每个 task 严格 Red → Green → Refactor 5 步。**
>
> **前置依赖：Plan 1 + Plan 2 完成。**

**Goal：** 关系会推进。用户从陌生人聊到朋友、暧昧、恋人……每天回来看到亲密度涨一点，看到林小满称呼变化、说话方式变化。dogfood 第三关：每天打开有进度感吗？

**Architecture：**
- **5 阶段**：陌生人(0-100) → 朋友(100-300) → 暧昧(300-600) → 恋人(600-1000) → 老夫老妻(1000+)
- **亲密度引擎 D 方案**（spec §3.3）：
  - 基础行为分（每条消息 +1~+10，每日封顶 +50；连续登录加成 N 天 +N×5）
  - 关键剧情节点（条件命中 +30~+100 + 触发剧情演出）
  - AI 评估加成（DeepSeek V4-Flash 评估对话深度，+0~+20）
- **称呼/语调动态**：每个 stage 有自己的 `prompt_overrides`（在 character.persona_config JSONB 里）—— PromptBuilder 根据 relationship.current_stage 选对应 overrides 注入 system prompt
- **进度 UI**：聊天页顶部条显示当前阶段 + 距下一阶段进度

---

## 文件结构（Plan 1+2 之后的增量）

### 新增 / 修改后端

```
business_packages/
├── sanyan-character-core/  （扩展）
│   └── src/main/java/com/3yan/character/internal/
│       ├── intimacy/
│       │   ├── IntimacyEvent.java                # record 描述一次涨幅来源
│       │   ├── IntimacyCalculator.java           # 把 IntimacyEvent 转分数
│       │   ├── IntimacyService.java              # 累加 + 阶段切换
│       │   ├── DailyBehaviorCounter.java         # Redis 计数器（每日封顶）
│       │   ├── ConsecutiveLoginService.java      # 连续登录天数（Redis ZSET）
│       │   ├── PlotMilestoneRule.java            # 接口
│       │   ├── plotrule/
│       │   │   ├── DeepNightChatRule.java        # 连续 3 晚聊到深夜 → +50 + 触发"她半夜想你"
│       │   │   ├── FirstHonestShareRule.java     # 第一次深度分享 → +30
│       │   │   └── ...
│       │   └── ai/
│       │       └── ConversationQualityEvaluator.java  # 调 V4-Flash 评估对话深度
│       ├── stage/
│       │   ├── StageDefinition.java              # 5 阶段范围常量
│       │   ├── StageTransitionService.java       # 检测阶段切换 + 发事件
│       │   └── StageOverrideRepository.java      # 从 character.persona_config 读 stage 覆盖
│       └── ...
└── sanyan-character-api/  （扩展）
    └── event/{IntimacyChangedEvent,StageTransitionEvent}.java

business_packages/sanyan-chat-core/  （修改 PromptBuilder 注入 stage overrides）

business_packages/sanyan-character-api/  （扩展）
└── dto/RelationshipDto.java                      # 新增：暴露 intimacy_score / current_stage / next_stage_threshold

business_packages/sanyan-character-core/  （扩展）
└── web/RelationshipController.java               # GET /api/relationships/me （查当前关系）
```

### 数据库迁移

```
bootstrap/src/main/resources/db/migration/
├── V8__init_intimacy_logs.sql                    # 亲密度变化日志（审计 + 调试）
└── V9__seed_sanyan_xiaoman_stage_overrides.sql    # 林小满 5 个 stage 的 prompt 覆盖（更新 characters.persona_config）
```

### 新增 / 修改前端

```
business_packages/sanyan_chat/lib/src/
├── chat/
│   ├── widgets/
│   │   ├── intimacy_progress_bar.dart           # 顶部条
│   │   └── stage_transition_dialog.dart         # 阶段切换庆祝弹窗
│   └── chat_controller.dart  ← 修改：监听 intimacy 推送
└── api/models/relationship.dart                  # 新增

foundation_packages/sanyan_user/  （扩展，但更适合放 chat 包，因为是关系数据；保留在 chat 包内）
```

---

## Task 列表

### Phase S · 数据库 + 种子

#### Task S1：V8 intimacy_logs
- **Schema：** id / user_id / character_id / delta INT / reason VARCHAR(60) / metadata JSONB / new_score INT / new_stage SMALLINT / created_at；索引 (user_id, character_id, created_at DESC)
- **Commit：** `feat(db): V8 intimacy_logs（亲密度变化审计日志）`

#### Task S2：V9 林小满 stage overrides 种子
- **修改：** `characters.persona_config` JSONB 增加 `stage_overrides`：
  ```json
  {
    "stage_overrides": {
      "0": { "address": "你 / {nickname}", "tone_hint": "礼貌、有距离感、偶尔害羞", "topics_unlock": [] },
      "1": { "address": "{nickname} / 真名", "tone_hint": "自然、放松、会开玩笑", "topics_unlock": ["吐槽","生活琐事"] },
      "2": { "address": "笨蛋/傻瓜/{nickname}+宝", "tone_hint": "撒娇、试探、害羞", "topics_unlock": ["想见面","未来设想","暗示性话题"] },
      "3": { "address": "老公/宝贝", "tone_hint": "撒娇、依赖、占有欲", "topics_unlock": ["纪念日","思念","情感深度"] },
      "4": { "address": "老公/笨蛋（带亲昵感）", "tone_hint": "生活感、小吵小闹、深度依赖", "topics_unlock": ["小日常","未来生活"] }
    }
  }
  ```
- **Commit：** `feat(db): V9 林小满 5 阶段 stage overrides`

### Phase T · 亲密度核心组件

#### Task T1：StageDefinition 常量
- **关键：** `enum Stage { STRANGER(0,100), FRIEND(100,300), AMBIGUOUS(300,600), LOVER(600,1000), OLD_COUPLE(1000,Integer.MAX_VALUE) }`
- **Commit：** `feat(character-core): StageDefinition`

#### Task T2：DailyBehaviorCounter
- **关键：** Redis key `behavior:user:<userId>:date:<yyyy-MM-dd>` → 计数器 INCR；`isOverDailyCap(userId, currentDelta, cap=50)` 判断今日累计是否已到封顶；TTL 36h
- **TDD：** IT（Testcontainers Redis）
- **Commit：** `feat(character-core): DailyBehaviorCounter（Redis 每日封顶）`

#### Task T3：ConsecutiveLoginService
- **关键：** 用户每日首次登录调 `recordLogin(userId)`：检查 `last_login` 与 today；连续则 streak++，否则 streak=1；Redis key `streak:user:<userId>` 存 `{streak: N, last_date: yyyy-MM-dd}`；返回当前 streak（用于加成 streak×5，封顶 50）
- **TDD：** IT 覆盖连续 / 中断 / 重复登录
- **Commit：** `feat(character-core): ConsecutiveLoginService 连续登录加成`

#### Task T4：IntimacyCalculator
- **关键：** 输入 `IntimacyEvent`（type, payload），输出 `int delta`；type 包括：
  - `MESSAGE_SENT` → +1（限频每日 cap）
  - `DAILY_FIRST_LOGIN` → +10 + streak×5
  - `PLOT_MILESTONE` → 由 rule 携带值
  - `AI_QUALITY_BONUS` → 0-20（由 evaluator 携带）
- **TDD：** Mockito 单测，覆盖每种 type、每日封顶
- **Commit：** `feat(character-core): IntimacyCalculator`

#### Task T5：IntimacyService（累加 + 阶段切换 + 事件）
- **关键：** `recordEvent(userId, characterId, IntimacyEvent)`：
  1. calculator.compute → delta
  2. 更新 RelationshipEntity.intimacyScore（乐观锁 version）
  3. 写 IntimacyLogEntity 审计
  4. 检测 stage 是否跨边界 → 调 StageTransitionService.maybeTransition
  5. 发 `IntimacyChangedEvent(userId, characterId, oldScore, newScore, reason, delta)`
- **TDD：** Mockito 单测覆盖：成功累加 / 跨阶段触发 transition / 同一日多次 message 累加但不超 cap
- **Commit：** `feat(character-core): IntimacyService（累加 + 阶段切换 + 事件）`

#### Task T6：StageTransitionService
- **关键：** `maybeTransition(relationship, newScore)`：检测 newScore 是否进入新 stage；是则更新 current_stage + 发 `StageTransitionEvent(userId, characterId, fromStage, toStage)`
- **Commit：** `feat(character-core): StageTransitionService`

### Phase U · 剧情节点规则（PlotMilestoneRule）

#### Task U1：规则引擎接口 + 注册
- **关键：** `PlotMilestoneRule` 接口：`Optional<IntimacyEvent> evaluate(MessageContext context)`；context 含 user/character/最近 N 条消息/已触发节点列表；规则放 `plotrule/` 子包，Spring 自动收集所有 Bean 注入 `PlotMilestoneEngine`
- **触发时机：** Listener 监听消息落库后调 engine.evaluate → 如果有规则返回 IntimacyEvent，调 IntimacyService.recordEvent
- **去重：** `relationship_milestones` 简单表（user_id / character_id / rule_id / triggered_at）防止同一规则重复触发
- **Commit：** `feat(character-core): PlotMilestoneEngine + 规则注册`

#### Task U2：DeepNightChatRule（连续 3 晚聊到深夜）
- **关键：** 检测最近 3 个晚上（22:00-02:00）都有过聊天 → +50 + 携带 metadata 让前端展示剧情卡片"她半夜悄悄想你"
- **TDD：** Mockito 单测覆盖触发 / 不触发
- **Commit：** `feat(character-core): DeepNightChatRule`

#### Task U3：FirstHonestShareRule（第一次深度分享）
- **关键：** 检测用户消息中含"我有点难过"/"我跟家里"/"我心里"等情感关键词 + memory_profile 中 emotion_line 为空 → +30
- **Commit：** `feat(character-core): FirstHonestShareRule`

#### Task U4：StageEntryCelebrationRule（每次首次进入新 stage 触发）
- **关键：** 监听 StageTransitionEvent → 触发庆祝剧情（"林小满第一次叫你宝贝"等）；不直接加分（StageTransition 本身就是结果），但产生剧情演出消息
- **Commit：** `feat(character-core): 阶段切换剧情演出`

### Phase V · AI 评估加成

#### Task V1：ConversationQualityEvaluator
- **关键：** 输入最近 10-20 条对话，调 DeepSeek V4-Flash 给出 0-20 分评估 + 简短说明；prompt：
  ```
  评估以下用户与林小满的对话片段质量（0-20 分）。
  评分维度：
  - 用户分享深度（透露真实信息、情感、想法）
  - 对话连贯性（不是单字"嗯"/"哦"敷衍）
  - 互动深度（双向，不是单方面输出）
  返回 JSON：{"score": <0-20>, "reason": "<10字内>"}
  对话：{{messages}}
  ```
- **触发时机：** 每 10 条用户消息触发一次（避免每条都调，浪费）
- **TDD：** Mockito 测试 LLM 返回 fake JSON 解析正确
- **Commit：** `feat(character-core): ConversationQualityEvaluator AI 评估`

#### Task V2：QualityBonusListener
- **关键：** 监听 `MessagePersistedEvent` → 累计用户消息数到 10 → 触发 evaluator → 如果 score > 0 调 IntimacyService.recordEvent(AI_QUALITY_BONUS, score)
- **Commit：** `feat(character-core): QualityBonusListener 触发链路`

### Phase W · PromptBuilder stage 注入

#### Task W1：StageOverrideRepository
- **关键：** 从 `character.persona_config.stage_overrides[currentStage]` 读取 address/tone_hint/topics_unlock；返回 `StageOverride` record
- **Commit：** `feat(character-core): StageOverrideRepository`

#### Task W2：PromptBuilder 改造（注入 stage override）
- **修改：** `build` 方法签名加 `Relationship relationship` 参数；构造顺序：
  1. system: character.basePrompt
  2. system: stage override 段（"当前关系阶段：{stage_name}。称呼用户用：{address}。语调：{tone_hint}。"）
  3. system: memory context（来自 Plan 2）
  4. 短期上下文 30 条
- **Commit：** `feat(chat-core): PromptBuilder 注入 stage override`

#### Task W3：ChatService 传递 relationship
- **修改：** ChatService 在 handleUserMessage 中先 `relationshipRepo.findByUserAndCharacter` → 传给 PromptBuilder
- **Commit：** `feat(chat-core): ChatService 取关系数据`

### Phase X · API + WS 推送进度更新

#### Task X1：RelationshipController GET /api/relationships/me
- **关键：** 返回 `RelationshipDto(intimacyScore, currentStage, currentStageName, nextStageThreshold, percentToNextStage)`
- **Commit：** `feat(character-core): GET /api/relationships/me`

#### Task X2：WS 推送 IntimacyChangedEvent + StageTransitionEvent
- **关键：** ChatWebSocketHandler 注册监听 `IntimacyChangedEvent` / `StageTransitionEvent`，按 userId 推送 `{"type":"intimacy_update", ...}` 和 `{"type":"stage_transition", from, to}` 帧
- **Commit：** `feat(chat-core): WS 推送亲密度 / 阶段变化`

### Phase Y · 前端

#### Task Y1：Relationship 模型 + sanyan_chat_api 加 fetchMyRelationship
- **Commit：** `feat(chat-api): RelationshipReq + 模型`

#### Task Y2：IntimacyProgressBar widget
- **关键：** 顶部条显示阶段名 + 进度（粉紫渐变填充）；点击展开详情弹窗（上次涨分原因、距下一阶段还差 X 分等）
- **TDD：** widget test 覆盖不同 stage / 进度
- **Commit：** `feat(chat-ui): 亲密度进度条`

#### Task Y3：StageTransitionDialog
- **关键：** 收到 stage_transition 帧 → 弹窗动画 + 文案（"她第一次叫你宝贝……"）
- **Commit：** `feat(chat-ui): 阶段切换庆祝动画`

#### Task Y4：chat_controller 监听 intimacy_update
- **修改：** 收到 ws intimacy_update → 更新 `Rx<Relationship> relationship` → 顶部条 Obx 自动刷新
- **Commit：** `feat(chat-ui): controller 监听亲密度推送`

### Phase Z · dogfood 第三关

#### Task Z1：dogfood 7 天验证
- **目标：** 自己用 7 天，记录：
  - [ ] 7 天能从陌生人推到朋友吗？过快还是过慢？
  - [ ] 阶段切换的剧情演出是否打动？
  - [ ] 亲密度涨分是否透明、可预期？
  - [ ] 林小满称呼/语调切换感受？
- **调整数值：** 根据 dogfood 调整 IntimacyCalculator 的 delta 数值（写在 application.yml 而不是硬编码）
- **Commit：** `docs: Plan 3 dogfood 反馈 + 数值调优`

---

## Self-Review Checklist

- [x] Spec §3.2 5 阶段 + §3.3 亲密度引擎 D 方案全覆盖
- [x] 行为 + 剧情节点 + AI 评估三种来源都有
- [x] 称呼/语调切换通过 prompt overrides 实现（不写死代码）
- [x] DDD `-api`+`-core` 双模块严格遵守，新组件全部放在 character-core 内（不破坏边界）

---

## Execution Handoff

**Plan 1 + Plan 2 都跑完后再启动 Plan 3。**
