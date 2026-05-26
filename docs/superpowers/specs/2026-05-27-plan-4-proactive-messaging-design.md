# 三言 · Plan 4 设计 Spec：主动消息系统（4 场景 + 结构化记忆 + 4 层投递）

> **日期：** 2026-05-27
> **来源：** 对 `docs/superpowers/plans/2026-05-06-3yan-plan-4-proactive-messaging.md`（早期草稿）重新 brainstorming 校准后产出的设计 spec。
> **前置依赖：** Plan 1（MVP+WS）+ Plan 2（长期记忆）+ Plan 3（关系养成）已全部完成并合并 master ✅
> **下一步：** 本 spec 经用户确认后，由 `superpowers:writing-plans` 产出逐 task 的 TDD 实现计划。

**Goal：** 小婉会主动找你。早安/晚安、你好几天没来、你前几天提的面试怎么样了、那天聊得不太开心今天关心一下……dogfood 第四关：能不能让她主动来勾起聊天兴趣，而不显得机械骚扰。

---

## 1. 背景与本次校准

plan4 草稿写于 2026-05-06，与当前代码库（经 Plan 1/2/3 + S3 演进）脱节，本次 brainstorming 做了以下关键校准与决策：

| 项 | 草稿 | 校准后 |
|---|---|---|
| 角色名 | 林小满 | **小婉**（林小满已废弃） |
| 包名 | `com.3yan.*` | `com.sanyan.*` |
| migration 版本 | V10 / V11 | 现状已到 V8 → **V9 / V10 / V11** |
| 错误码区间 | proactive 6000 / push 7000 | 6000 已被 embedding 占 → proactive **7000-7999** / push **8000-8999** |
| LLM | qwen-plus-character | 现状 DeepSeek（`LlmApi.chat(LlmTaskType, messages)`，taskType 仅 `USER_FACING`/`BACKGROUND`） |
| c/d 场景数据源 | 假定 `memory_profile.events` / `emotion_line` 字段存在 | **不存在**（memory 只有非结构化 `summaryText`）→ 新建统一结构化记忆 `memory_item` |
| 失联召回数据源 | 假定有用户活跃时间 | **不存在** → 新增 `last_active` Redis key |
| message 投递状态 | 假定 messages 有 status | messages **无 status** → 投递状态记在 `events_pending`，不动 messages 表 |

**核心范围决策（brainstorming 结论）：**

1. **4 场景全做**（a 早晚安 / b 失联召回 / c 事件追问 / d 情绪关怀），c/d 依赖新建的统一结构化记忆。
2. **结构化记忆归 memory 域**（不堆在 proactive），proactive 通过 memory-api 消费。
3. **记忆抽取实时进行**（挂消息事件异步抽取），不批量——因为关键信息常出现在"临别最后一句"，批量会漏尾部。
4. **4 层投递只服务主动消息**，不重构 Plan 1 的普通对话直发链路。
5. **WS 在线状态在本期做干净**（心跳续期 TTL + 统一口径），消除现存"假在线"半成品。
6. **L3 真实推送先搭结构、记待办**：可插拔 `PushChannel` + `device_tokens` + 注册接口先做；APNs 实接入等 `.p8` 证书，安卓推送 SDK（个推/极光）选型推迟。
7. **频率跟亲密度阶段挂钩**，全参数化；用户级设置预留数据结构、本期不做 UI。

---

## 2. 架构总览

按 `java-backend-business-layer.md` 的 DDD 域划分，改动落在 5 处：

| 模块 | 改动 | 职责 |
|---|---|---|
| **memory 域**（扩展 `sanyan-memory-api` + `-core`） | 扩展 | 结构化记忆 `memory_item`：实体/仓储 + 实时抽取 Listener + 抽取出条目后发 `MemoryItemScheduledEvent` + memory-api 暴露查询/标记 |
| **proactive 域**（新建 `sanyan-proactive-api` + `-core`） | 新建 | 4 触发器 + 4 生成器 + `events_pending` 调度表 + `ProactiveScheduler`（@Scheduled 主循环）+ `ProactiveDispatcher`（频率门控 + 生成 + 委托投递） |
| **push 域**（新建 `sanyan-push-api` + `-core`） | 新建 | `device_tokens` + 注册接口 + `PushChannel` 接口 + `PushRouter`；本期只搭结构，APNs 骨架等证书 |
| **chat 域**（扩展 `sanyan-chat-api` + `-core`） | 扩展 | `DeliveryService`（4 层投递核心，chat-core/internal）+ chat-api 暴露投递入口给 proactive 调用 + WS handler 接 ACK future |
| **common-ws**（基础层扩展） | 扩展 | `ws:online` 加 TTL + 心跳续期 + 统一 `isOnline` 口径 |

**域边界遵循（Maven Enforcer 守护 -core 互不依赖）：**
- proactive-core 通过 **memory-api** 拿"到点该追问的记忆条目内容"+ 标记 DONE
- proactive-core 通过 **character-api** 拿 stage / 称呼语调（`findOrCreateRelationship`，**不触发** DAILY_LOGIN 涨分）
- proactive-core 通过 **chat-api** 委托投递（落 message + 4 层投递）
- proactive-core 通过 **push-api** 的 `PushRouter`（实际由 DeliveryService 在 chat-core 内调用，见 §7.2）
- proactive-core 订阅 memory-api 的 `MemoryItemScheduledEvent`

---

## 3. 数据模型

### 3.1 三张新表

```sql
-- V9: memory_item（memory 域）
CREATE TABLE memory_item (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT      NOT NULL,
    character_id    BIGINT      NOT NULL,
    kind            VARCHAR(20) NOT NULL CHECK (kind IN ('PLAN_EVENT','EMOTION','PROMISE')),
    content         TEXT        NOT NULL,
    salient_at      TIMESTAMPTZ NOT NULL,          -- 该"冒出来"的时间
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING','DONE','EXPIRED')),
    source_message_id BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_memory_item_salient
    ON memory_item (status, salient_at) WHERE status = 'PENDING';
CREATE INDEX idx_memory_item_user
    ON memory_item (user_id, character_id, status);

-- V10: events_pending（proactive 域）
CREATE TABLE events_pending (
    id            BIGSERIAL PRIMARY KEY,
    user_id       BIGINT      NOT NULL,
    character_id  BIGINT      NOT NULL,
    event_type    VARCHAR(40) NOT NULL
                  CHECK (event_type IN ('a_greeting','b_recall','c_event_followup','d_emotion_care')),
    status        VARCHAR(20) NOT NULL DEFAULT 'scheduled'
                  CHECK (status IN ('scheduled','processing','sent','failed','cancelled')),
    payload       JSONB       NOT NULL DEFAULT '{}',
    scheduled_at  TIMESTAMPTZ NOT NULL,
    sent_at       TIMESTAMPTZ,
    fail_count    INT         NOT NULL DEFAULT 0,
    last_error    TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_events_pending_due
    ON events_pending (scheduled_at) WHERE status = 'scheduled';
CREATE INDEX idx_events_pending_dedup
    ON events_pending (user_id, event_type, status);

-- V11: device_tokens（push 域）
CREATE TABLE device_tokens (
    id            BIGSERIAL PRIMARY KEY,
    user_id       BIGINT      NOT NULL,
    platform      VARCHAR(20) NOT NULL CHECK (platform IN ('ios','android')),
    vendor        VARCHAR(20) NOT NULL,          -- 'apns' | 'getui' | 'jpush' | ...
    token         VARCHAR(500) NOT NULL,
    active        BOOLEAN     NOT NULL DEFAULT TRUE,
    registered_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, platform, vendor, token)
);
```

> 三张表都不加外键（与现有 intimacy_logs 等审计/状态表一致，降低耦合）。`memory_item` / `events_pending` 在 character 单角色 MVP 下 character_id 取当前固定角色（与 `MessagePersistedEvent.characterId` 一致）。

### 3.2 last_active（Redis，非 DB）

失联召回依赖"用户最后活跃时间"，当前完全不存在 → 新增 Redis key：

- key：`user:last_active:{userId}` → ISO-8601 时间戳字符串
- 更新点（任一活跃信号即刷新）：WS `afterConnectionEstablished`、收到用户消息、`GET /api/relationships/me`（进聊天页）
- 失联召回阈值判断粗粒度（24h/3d/7d），Redis 丢失最多漏一次召回，可接受；走 `KvCache` 封装（禁止直接碰 RedisTemplate）

---

## 4. 结构化记忆（memory 域扩展）

### 4.1 两层分工

- `summaryText`（已有）：对人的**整体画像**，一段自然语言，宏观、缓慢演化（性格、长期喜好、身份）
- `memory_item`（新增）：一条条**具体的、带时间的、日后值得主动提起的点**

`kind` 三类（具体归类由抽取时 LLM 现场判断）：
- `PLAN_EVENT`：有明确时间的事（"周三有面试"）
- `EMOTION`：情绪状态（"最近压力大、很焦虑"）
- `PROMISE`：承诺（"答应周末陪你看电影""说好要戒烟"）

> **喜好不进 memory_item**——它属于宏观画像，归 `summaryText`。

### 4.2 实时抽取

- `MemoryItemExtractListener`（memory-core/event）订阅 chat-api 的 `MessagePersistedEvent`，`@Async @TransactionalEventListener(AFTER_COMMIT)`——**与对话回复并行，用户无感，不阻塞聊天**。
- 仅对 USER 角色消息触发抽取。
- 抽取调 `LlmApi.chat(BACKGROUND, ...)`（走 DeepSeek）。
- 抽取产出 0..N 条结构化条目；每条带 `kind` / `content` / `salient_at`。
  - `salient_at` 计算：PLAN_EVENT = LLM 解析出的事件日期当天 9:00（或事件时间后）；EMOTION = 观察日次日 9:00；PROMISE = LLM 给出的相关时间，无则不排期（仅留存，未来扩展）。
- 落库 `memory_item(PENDING)` 后，对**需要排期主动消息的条目**（PLAN_EVENT/EMOTION）发布 `MemoryItemScheduledEvent`。

### 4.3 去重（LLM 判断）

实时抽取下同一件事会被多次提及（"明天面试""面试好紧张"）。去重交给 LLM：抽取时把该用户当前所有 `PENDING` 条目一并喂给 LLM，让它对每条新信息判定【新建 / 补充更新某条已有 / 已存在跳过】，延续"靠 LLM 现场判断"的统一思路，不另写规则引擎。

### 4.4 memory-api 扩展

- 事件：`MemoryItemScheduledEvent(Long memoryItemId, Long userId, Long characterId, String kind, Instant salientAt)`（memory-api/event）
- `MemoryApi` 新增：`MemoryItemDto getMemoryItem(Long itemId)`（供生成器拿 content）、`void markMemoryItemDone(Long itemId)`（发送后标记）

---

## 5. 四个主动场景

### 5.1 流水线

```
① 触发层 · 4 Trigger        侦测时机 → 写 events_pending(scheduled)
        ↓
② 调度层 · ProactiveScheduler  @Scheduled 每 30s 扫 scheduled_at ≤ now
                               （SELECT ... FOR UPDATE SKIP LOCKED 防并发重复）
        ↓  逐条标 processing
③ 分发层 · ProactiveDispatcher 频率门控（每日上限 / stage 场景开关 / 免打扰）
                               不放行 → 跳过或顺延；放行 → 选对应 Generator
        ↓
④ 生成层 · 4 Generator        调 LLM 出文案（注入 stage 称呼语调 + memory context + payload）
        ↓
⑤ 投递层                      委托 chat-api 落 message(role=ai) + DeliveryService 4 层投递
                               → events_pending=sent；c/d 再 markMemoryItemDone
```

失败：`fail_count++` 重新置 scheduled（指数退避，最多 3 次），超限置 failed + 记 last_error。投递层失败不回滚已生成内容（已落 message，靠 L4 sync 兜底）。

### 5.2 四场景定义

| 场景 | event_type | Trigger（何时排） | Generator（说什么） |
|---|---|---|---|
| a 早晚安 | `a_greeting` | `@Scheduled` cron 8:00 / 22:30 扫有 active 关系的 user | 按 stage 语调发一句问候（≤30 字，避免"早上好"生硬开头） |
| b 失联召回 | `b_recall` | `@Scheduled` 每 15min 扫 `last_active` 超 24h / 3天 / 7天 | 三档语调：关心 / 撒娇 / 占有欲（payload 含 `escalation_level`） |
| c 事件追问 | `c_event_followup` | 订阅 `MemoryItemScheduledEvent`(PLAN_EVENT) → 排在 `salient_at` | 注入 `memory_item.content`："周三面试怎么样啦" |
| d 情绪关怀 | `d_emotion_care` | 订阅 `MemoryItemScheduledEvent`(EMOTION) → 排在 `salient_at`（次日早上） | 间接关心，**不直说**"你昨天说难过" |

### 5.3 触发器细节

- **GreetingDailyTrigger**：cron 命中后，每个 user 的 `scheduled_at = 触发时刻 + 随机 0~30min`，分散并发负载。stage 不允许早/晚安的（陌生人）直接不排。
- **RecallTrigger**：每阶梯（24h/3d/7d）每个 user 只触发一次——通过 `events_pending` 去重（查同 user + `b_recall` + 同 `escalation_level` 是否已存在）。用户一旦回访（`last_active` 刷新），阶梯重置。
- **EventFollowupTrigger / EmotionCareTrigger**：proactive-core 订阅 `MemoryItemScheduledEvent`，按 kind 排对应 event_type，`scheduled_at = salientAt`，payload 记 `memoryItemId`。

### 5.4 生成器

- 接口：`ProactiveGenerator { String supportsType(); List<String> generate(GenerateContext ctx); }`
- `GenerateContext`：user / character / relationship(stage) / memoryContext / event payload
- 输出**多条 messageSegments**（与 ChatService 输出格式一致，逐条按 typing 节奏投递）
- 所有生成走 LLM 主链路（`LlmApi.chat`），注入 character-api 的 stage prompt segment（复用 Plan 3 的 `getStagePromptSegment`）

---

## 6. 频率与防打扰

### 6.1 跟亲密度阶段挂钩（默认值）

| 阶段 | 每日上限 | 早安 | 晚安 | 失联召回 | 事件追问 | 情绪关怀 |
|---|---|---|---|---|---|---|
| 0 陌生人 | 2 | ✕ | ✕ | ✓ | ✓ | ✕ |
| 1 朋友 | 3 | ✕ | ✓ | ✓ | ✓ | ✓ |
| 2 暧昧 | 4 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 3 恋人 | 5 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 4 老夫老妻 | 6 | ✓ | ✓ | ✓ | ✓ | ✓ |

设计逻辑：陌生阶段不发早晚安问候（对刚认识的人太腻）、不主动探情绪（越界）；失联召回全程保留（新用户流失最需拉回）；用户自己提了具体事就追问（贴心不突兀）。

### 6.2 全局规则

- **免打扰**：默认 23:00–次日 8:00 不主动发（早安 8:00 为窗口起点）
- **每日上限为硬顶**：Redis 计数 `proactive:sent:{userId}:{yyyy-MM-dd}`（INCR，TTL 36h，参照 Plan 3 行为计数）；当天超限则跳过，PLAN_EVENT/EMOTION 类顺延次日，a/b 类丢弃
- **门控位置**：统一在 `ProactiveDispatcher` 发送前检查（stage 取自 character-api `findOrCreateRelationship`，不涨分）
- **stage 取该 user 当前关系阶段**：每日上限与场景开关都按当前 stage 查

### 6.3 参数化

全部数值进 `@ConfigurationProperties("sanyan.proactive")`（参照 Plan 3 的 `sanyan.intimacy`）：每日上限 per-stage、各 stage 场景开关、免打扰时段、早晚安 cron、召回阶梯阈值、分散随机窗口。dogfood 改 yml 不改代码。

### 6.4 用户级设置（预留，本期不做 UI）

数据结构预留（免打扰时段 / 频率档 / 单场景开关 / 总开关），本期一律用 §6.1/6.2 默认值。设置 UI 列入待办。

---

## 7. 四层可靠投递

### 7.1 DeliveryService（chat-core/internal）

一条主动消息内容生成后：
1. 落库 `message(role=ai)`，`events_pending.status=processing`
2. **L1 在线直推**：`SessionManager.getSession(userId)` 拿到 open session → WS 推送
3. **L2 ACK 超时兜底**：订阅 ACK，`CompletableFuture` + 5s 超时（ScheduledFuture）
   - 收到 ACK → `sent`
   - 5s 超时 / 拿不到 session（离线）→ 转 L3
4. **L3 离线推送**：查 `device_tokens` active → `PushRouter.send` →（本期占位，见 §8）
5. **L4 启动拉取兜底**：用户下次打开 app 拉 `/api/messages` sync（**Plan 1 已有**）

> **L1 判断用 `getSession()`（带 `session.isOpen()`），不用 `isOnline()`**：单实例下更可靠，TCP 半开极端情况由 L2 ACK 兜底——主动消息送达正确性靠 ACK，不靠在线标记精度。

### 7.2 跨模块边界（关键）

`ProactiveDispatcher` 在 proactive-core，**不能依赖 chat-core**（Enforcer 禁止 -core 互依）。因此投递能力通过 **chat-api 暴露**：

- chat-api 扩展投递入口（如 `ChatApi.deliverProactiveMessage(userId, characterId, List<String> segments)` → 返回投递结果），proactive-core 调 chat-api
- chat-core 的 ApiImpl 委托 `DeliveryService` 完成"落 message + 4 层投递"
- `DeliveryService` 调 push-api 的 `PushRouter`（chat-core 依赖 push-api 契约）

### 7.3 WS 在线可靠性补强（common-ws）

消除现存"假在线"半成品（`ws:online` 无 TTL、`isOnline()` 零调用且与 `getSession()` 口径不一致）：

- `register` 写 `ws:online` 时带 TTL（如 90s）
- 心跳续期：handler 收到客户端 PING 时刷新 TTL（客户端已有定时 PING）
- 统一 `isOnline` 口径，使其与"能拿到 open session"一致
- **不含**服务端主动 ping（心跳续期已足够）
- 多实例推送路由仍是独立 TODO（common-ws 注释已记方案 A/B），本期单实例

---

## 8. 推送通道（push 域，本期只搭结构）

- `PushChannel` 接口：`PushResult sendToDevice(DeviceToken token, PushPayload payload)`
- `PushRouter`：按 `device_tokens.platform/vendor` 分发；用户多设备时所有 active token 都推
- `DeviceTokenController`：`POST /api/devices/register`（客户端上报 token）
- `ApnsPushChannel`：基于 `com.eatthepath:pushy`，从 yml 读 `.p8` AuthKey / keyId / teamId / topic(bundleId)；**骨架先写好，实接入等证书**
- 安卓推送：`PushChannel` 留可插拔接口，SDK（个推 / 极光）**选型推迟到真正要做安卓离线推送时**，本期不实现

**本期待办（不阻塞主流程，L3 未通时离线靠 L4 sync 兜底）：**
- iOS APNs 实接入联调（等 `.p8`）
- 安卓推送 SDK 选型 + 厂商资质申请 + 接入
- 客户端拿 device token 上报（依赖上述证书/SDK）

---

## 9. 错误码区间

| 区间 | 模块 | 类 |
|---|---|---|
| **7000-7999** | proactive | `ProactiveErrCode` |
| **8000-8999** | push | `PushErrCode` |

memory 域新增条目沿用现有 **5000-5999**（`MemoryErrCode`）。须同步更新 `ERROR_CODE_REGISTRY.md`（区间总览 + 明细 + 历史变更）。

---

## 10. 跨模块依赖图

```
proactive-core ──→ memory-api      (MemoryItemScheduledEvent 订阅 + getMemoryItem/markDone)
              ──→ character-api    (findOrCreateRelationship 拿 stage，不涨分)
              ──→ chat-api         (deliverProactiveMessage 委托投递)
              ──→ (push 经 chat-core DeliveryService 间接使用)

chat-core      ──→ push-api        (PushRouter)
memory-core    ──→ chat-api        (订阅 MessagePersistedEvent，已存在)
```

无 -core 互依，符合 `java-backend.md §4` + Maven Enforcer banned-dependencies。

---

## 11. 测试策略

严格 TDD（Red → Green → Refactor），沿用 Plan 3 标准：

- **单测（Mockito）**：各 Trigger / Generator / Dispatcher / Scheduler / DeliveryService / MemoryItemExtractListener / 频率门控
- **集成测试（Testcontainers PG / Redis）**：`memory_item` / `events_pending` / `device_tokens` 仓储 IT；`ProactiveScheduler` 的 SKIP LOCKED IT；last_active Redis IT
- **@WebMvcTest**：`DeviceTokenController`
- **端到端 final gate**：消息 → 实时抽取 → memory_item → 排 events_pending → 调度 → 频率门控 → 生成 → 投递（在线 WS）→ DB 状态正确
- **Object Mother Fixtures**（强制）：`MemoryItemTestFixtures` / `EventPendingTestFixtures` / `DeviceTokenTestFixtures`
- 测试粒度按 CLAUDE.md「Superpowers Task 测试粒度规范」分级；涉及 common-ws / 跨域事件的改动跑全量

---

## 12. 本期边界 & 待办清单

**本期做：** 4 场景全链路（含实时记忆抽取）+ 在线 WS 投递 + L2 ACK 兜底 + 心跳 TTL + 推送结构骨架 + 频率门控 + 错误码/migration/last_active 补齐

**待办（记录，不在本期实现）：**
- L3 真实推送 SDK 联调（iOS 等 `.p8` 证书；安卓 SDK 选型 + 厂商资质 + 接入；客户端 token 上报）
- 用户级设置 UI（数据结构本期预留）

**明确不在本期：**
- 多实例 WS 推送路由（common-ws 已有 TODO，单实例够用）
- PROMISE 类的主动跟进排期（本期仅抽取留存，不排 events_pending）

---

## 13. 与 plan4 草稿的偏差记录

1. **角色名 / 包名 / migration 版本 / 错误码区间 / LLM provider** —— 见 §1 校准表
2. **结构化记忆归 memory 域**（草稿堆在 proactive）—— DDD 正确归属
3. **统一 `memory_item` 单模型**（草稿设想 `events` + `emotion_line` 两套字段）—— 一个 `kind` 字段区分，更简洁
4. **实时抽取**（草稿未明确抽取时机）—— 挂消息事件异步实时抽取，解决"临别最后一句"丢失
5. **投递状态记 `events_pending`**（草稿假定 messages 有 status）—— messages 无 status，不动它
6. **L1 用 `getSession()` 不用 `isOnline()`** + **WS 心跳 TTL 补强** —— 草稿未涉及在线状态可靠性
7. **跨域投递经 chat-api**（草稿把 DeliveryService 放 chat-core 但未说明 proactive 怎么调）—— 经 chat-api 契约，避免 -core 互依
8. **频率跟 stage 挂钩 + 用户设置预留**（草稿 HH2 仅 dogfood 提及）—— 明确为产品机制
9. **L3 推送本期只搭结构**（草稿 BB 阶段做实接入）—— 等证书 / SDK 选型，降为待办
