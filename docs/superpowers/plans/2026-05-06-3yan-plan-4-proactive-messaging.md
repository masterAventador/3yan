# 三言 · Plan 4 实施计划：主动消息系统（4 场景 + APNs/个推 + 4 层可靠投递）

> **For agentic workers:** REQUIRED SUB-SKILL: `superpowers:subagent-driven-development`. Steps use `- [ ]` checkboxes.
>
> **TDD 铁律：每个 task 严格 Red → Green → Refactor 5 步。**
>
> **前置依赖：Plan 1 + Plan 2 + Plan 3 完成。**

**Goal：** 林小满会主动找你。早安/晚安、你好几天没来、你昨天提的面试怎么样了、深夜聊得不开心今天关心一下……dogfood 第四关：能不能让她主动来勾起聊天兴趣？

**Architecture：**
- **4 种主动场景**（spec §5.1）：
  - **a 早晚安问候**（`a_greeting`）：固定时间点（默认 8:00 / 22:30）触发
  - **b 失联召回**（`b_recall`）：用户连续未打开 24h / 3天 / 7天 阶梯
  - **c 重要事件追问**（`c_event_followup`）：基于 memory_profile.events 中 `date` 字段定时
  - **d 情境关怀**（`d_emotion_care`）：基于 memory_profile.emotion_line 检测低落 → 次日关心
- **生成内容**：调 LLM 主链路（qwen-plus-character）按 prompt 模板生成（每个场景独立 prompt + 当前 stage override + memory context）
- **4 层可靠投递**（spec §5.2）：
  - L1：WS push（用户在线）
  - L2：5s ACK 超时 → fallback 离线通道
  - L3：APNs（iOS）+ 个推（Android 国内）
  - L4：客户端启动拉 `/api/messages/sync`（**Plan 1 已实现**，本 plan 不重做）
- **调度**：Spring `@Scheduled` 简单调度（每分钟扫一次 events_pending 表）；Redis Streams 作为执行 worker（横向扩展时可多实例消费）

---

## 文件结构（增量）

### 后端

```
business_packages/
├── sanyan-proactive-api/
│   └── src/main/java/com/3yan/proactive/
│       ├── ProactiveApi.java
│       └── dto/{ProactiveScheduleRequest,ScheduleResult}.java
├── sanyan-proactive-core/
│   └── src/main/java/com/3yan/proactive/
│       ├── internal/
│       │   ├── EventPendingEntity.java
│       │   ├── EventPendingRepository.java
│       │   ├── EventType.java                   # a_greeting / b_recall / c_event_followup / d_emotion_care
│       │   ├── EventStatus.java                 # scheduled / processing / sent / failed / cancelled
│       │   ├── ProactiveScheduler.java          # @Scheduled 主循环
│       │   ├── ProactiveDispatcher.java         # 拉一条 → 生成内容 → 落库 message → 投递
│       │   ├── ProactiveErrCode.java            # 6000-6999
│       │   └── generator/
│       │       ├── GreetingGenerator.java       # a 模板
│       │       ├── RecallGenerator.java         # b 模板
│       │       ├── EventFollowupGenerator.java  # c 模板
│       │       └── EmotionCareGenerator.java    # d 模板
│       ├── trigger/
│       │   ├── GreetingDailyTrigger.java        # 每日扫所有用户调度 a
│       │   ├── RecallTrigger.java               # 监听 user 离线超过阈值
│       │   ├── EventFollowupTrigger.java        # 监听 memory_profile 新事件 → 调度 c
│       │   └── EmotionCareTrigger.java          # 监听 emotion_line 低落 → 调度 d
│       ├── api/ProactiveApiImpl.java
│       └── event/MessagePersistedListener.java  # 复用 Plan 2 事件
└── sanyan-push-api/
│   └── src/main/java/com/3yan/push/
│       ├── PushChannel.java                     # 接口：sendToDevice(deviceToken, payload)
│       └── dto/{PushPayload,DeviceToken}.java
├── sanyan-push-core/
│   └── src/main/java/com/3yan/push/
│       ├── internal/
│       │   ├── DeviceTokenEntity.java
│       │   ├── DeviceTokenRepository.java
│       │   ├── apns/ApnsPushChannel.java        # iOS（基于 com.eatthepath:pushy）
│       │   ├── getui/GetuiPushChannel.java      # Android 国内（个推 SDK）
│       │   ├── PushRouter.java                  # 按 platform 选 channel
│       │   └── PushErrCode.java                 # 7000-7999
│       └── web/
│           └── DeviceTokenController.java       # POST /api/devices/register

business_packages/sanyan-chat-core/  （扩展）
└── src/main/java/com/3yan/chat/internal/
    └── DeliveryService.java                     # 4 层投递核心：L1+L2+L3 路由（L4 在 Plan 1 已有）
```

### 数据库迁移

```
bootstrap/src/main/resources/db/migration/
├── V10__init_events_pending.sql
└── V11__init_device_tokens.sql
```

### 前端

```
business_packages/sanyan_chat/lib/src/
├── chat_controller.dart  ← 修改：处理收到主动消息时若 app 已 background 触发推送提示

foundation_packages/sanyan_user/lib/src/
└── push/
    ├── apns_setup.dart                          # iOS 推送权限申请 + token 上报
    └── getui_setup.dart                         # 个推初始化 + token 上报

3yan-app/ios/Runner/  （iOS 推送配置）
3yan-app/android/app/  （Android 个推配置）
```

---

## Task 列表

### Phase AA · 数据库

#### Task AA1：V10 events_pending
- **Schema：**
  ```sql
  CREATE TABLE events_pending (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    character_id BIGINT NOT NULL,
    event_type VARCHAR(40) NOT NULL CHECK (event_type IN ('a_greeting','b_recall','c_event_followup','d_emotion_care')),
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled' CHECK (status IN ('scheduled','processing','sent','failed','cancelled')),
    payload JSONB DEFAULT '{}',
    scheduled_at TIMESTAMPTZ NOT NULL,
    sent_at TIMESTAMPTZ NULL,
    fail_count INT DEFAULT 0,
    last_error TEXT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  );
  CREATE INDEX ON events_pending (status, scheduled_at) WHERE status='scheduled';
  CREATE INDEX ON events_pending (user_id, event_type, status);
  ```
- **Commit：** `feat(db): V10 events_pending`

#### Task AA2：V11 device_tokens
- **Schema：** id / user_id / platform VARCHAR(20)（'ios'|'android'）/ token VARCHAR(500) / vendor VARCHAR(20)（'apns'|'getui'|'huawei'|...）/ active BOOLEAN DEFAULT TRUE / registered_at / last_seen；UNIQUE(user_id, platform, vendor, token)
- **Commit：** `feat(db): V11 device_tokens`

### Phase BB · 推送通道

#### Task BB1：DeviceTokenEntity + Controller
- **Files：** entity + repository + `DeviceTokenController.POST /api/devices/register`
- **TDD：** Repository IT + Controller IT
- **Commit：** `feat(push-core): 设备 token 注册`

#### Task BB2：ApnsPushChannel
- **关键：** 用 `com.eatthepath:pushy` 库；从 `application.yml` 读 APNs P8 token / topic（com.sanyan.app）；`sendToDevice(token, payload)` 返回 `PushResult`；payload 格式 `{"aps":{"alert":{"title":"三言","body":"<msg>"},"sound":"default"},"messageId":<id>}`
- **TDD：** Mockito mock pushy ApnsClient，覆盖成功/无效 token/网络错误
- **Commit：** `feat(push-core): APNs 推送通道`

#### Task BB3：GetuiPushChannel
- **关键：** 个推 Java SDK；按个推官方文档配置 appId/appKey/masterSecret；payload 走个推标准透传
- **TDD：** Mockito mock SDK
- **Commit：** `feat(push-core): 个推推送通道（Android 国内）`

#### Task BB4：PushRouter
- **关键：** 按 device_tokens.platform/vendor 字段分发；用户多设备时所有 active token 都推
- **Commit：** `feat(push-core): PushRouter`

### Phase CC · 4 层投递核心（DeliveryService）

#### Task CC1：DeliveryService.deliver(messageId, userId)
- **流程：**
  1. 查 message status：必须是 PENDING
  2. 查 WsConnectionRegistry：用户在线吗？
     - 在线：调 `WsPushService.pushTo(userId, payload)`；订阅 ACK 5 秒超时（用 ScheduledFuture）
       - 收到 ACK → status=DELIVERED + delivered_at=now
       - 5s 超时 → 走 L3 推送
     - 离线：直接走 L3 推送
  3. L3：查 device_tokens active → PushRouter.send → status=PUSH_SENT
  4. 推送也失败：status=FAILED，记录 last_error，写日志
- **关键：** ACK 等待用 `CompletableFuture<MessageId>`，存在 ConcurrentHashMap，超时自动 complete with TIMEOUT
- **TDD：** Mockito 单测覆盖：在线+ACK / 在线+超时fallback / 离线直推 / 全失败
- **Commit：** `feat(chat-core): DeliveryService 4 层投递（L1+L2+L3，L4 已存在）`

#### Task CC2：ChatWebSocketHandler 升级（接入 DeliveryService）
- **修改：** Plan 1 中 ChatWebSocketHandler 直接 send 的逻辑改为通过 DeliveryService.deliver；ACK 帧不再直接更新 status，由 DeliveryService 的 future complete
- **Commit：** `feat(chat-core): WS Handler 接入 DeliveryService`

### Phase DD · 主动消息生成器

#### Task DD1：generator 接口 + 共享 prompt 工具
- **关键：** `interface ProactiveGenerator { String supportsType(); ProactiveContent generate(GenerateContext ctx); }`；`GenerateContext` 包含 user/character/relationship/memoryContext/payload
- **`ProactiveContent(List<String> messageSegments)`**：拆好的多条消息（同 ChatService 输出格式，给 DeliveryService 推送）
- **Commit：** `feat(proactive-core): 生成器接口`

#### Task DD2：GreetingGenerator (a)
- **Prompt 模板：**
  ```
  你是「林小满」，你和用户当前关系阶段是「{stage_name}」。
  现在是 {time_of_day}，请用你的语调发一句问候（不超过 30 字）。
  避免用 "早上好" 这种生硬开头，要自然、有你的专属感。
  关于他你记得：{profile_brief}
  ```
- **TDD：** Mockito + LLM mock，覆盖不同 stage 输出风格不同
- **Commit：** `feat(proactive-core): GreetingGenerator`

#### Task DD3：RecallGenerator (b) — 阶梯式 3 条不同语调
- **Prompt 不同档：** 24h "关心"、3天 "撒娇"、7天 "占有欲"；payload 含 `escalation_level`
- **Commit：** `feat(proactive-core): RecallGenerator`

#### Task DD4：EventFollowupGenerator (c)
- **关键：** payload 含 event 描述（来自 memory_profile.events），prompt 把 event 注入；如 "用户提到周三面试，今天周四，问问结果"
- **Commit：** `feat(proactive-core): EventFollowupGenerator`

#### Task DD5：EmotionCareGenerator (d)
- **关键：** payload 含上次低落事件描述；prompt 引导关心但不直接提到"昨天你说难过"（太机械），而是间接关心
- **Commit：** `feat(proactive-core): EmotionCareGenerator`

### Phase EE · 触发器

#### Task EE1：GreetingDailyTrigger
- **关键：** `@Scheduled(cron = "0 0 8,22 * * *")` 每天 8:00 和 22:30（22:30 改成 `30 22 * * *`）；扫所有有 active 关系的 user，调度 a_greeting 事件 30 分钟后执行（避免所有用户同一秒发，分散负载）
- **Commit：** `feat(proactive-core): GreetingDailyTrigger`

#### Task EE2：RecallTrigger
- **关键：** `@Scheduled(cron = "0 */15 * * * *")` 每 15 分钟扫一次：找 last_active_at 跨 24h / 3天 / 7天阈值的 user → 调度 b_recall（每个阶梯只触发一次，用 events_pending + 去重 key）
- **Commit：** `feat(proactive-core): RecallTrigger`

#### Task EE3：EventFollowupTrigger
- **关键：** Listener 监听 Plan 2 的 `MemoryProfileUpdatedEvent`（如果没有就在 memory-core 加上）；如果新增 event 含 date，调度 c_event_followup 在 date+1 天 9:00 执行
- **Commit：** `feat(proactive-core): EventFollowupTrigger`

#### Task EE4：EmotionCareTrigger
- **关键：** Listener 监听 emotion_line append 事件，如果是 "焦虑" "难过" "低落" 等关键词 → 调度 d_emotion_care 在 +12h 后（次日早上）执行
- **Commit：** `feat(proactive-core): EmotionCareTrigger`

### Phase FF · 调度器主循环

#### Task FF1：ProactiveScheduler
- **关键：** `@Scheduled(cron = "*/30 * * * * *")` 每 30 秒扫 events_pending 表 status=scheduled AND scheduled_at <= NOW() LIMIT 50；用 SELECT ... FOR UPDATE SKIP LOCKED 防多实例重复（PostgreSQL 支持）；逐条标记 processing → 委托 ProactiveDispatcher.dispatch；失败 fail_count++ 重新 scheduled（指数退避，最多 3 次）
- **Commit：** `feat(proactive-core): ProactiveScheduler`

#### Task FF2：ProactiveDispatcher
- **关键：** 选生成器 → generate → 拆成多条 messageSegments → 每条落库 messages 表（role=ASSISTANT, status=PENDING）→ 调 DeliveryService.deliver（一次推一条 + 中间 typing_start/typing_end + 模拟延迟）
- **Commit：** `feat(proactive-core): ProactiveDispatcher`

### Phase GG · 前端

#### Task GG1：iOS 推送权限申请
- **Files：** `3yan-app/ios/Runner/` 加推送权限配置；`foundation_packages/sanyan_user/lib/src/push/apns_setup.dart` 申请权限 + 拿 deviceToken + 上报后端
- **Commit：** `feat(app-push): iOS 推送权限 + token 上报`

#### Task GG2：Android 个推集成
- **Files：** `3yan-app/android/app/build.gradle` 加个推依赖；`getui_setup.dart` 初始化 + token 上报
- **Commit：** `feat(app-push): Android 个推 + token 上报`

#### Task GG3：用户登录后注册 device token
- **修改：** UserCenter.loginCompleted 后立即调用 push setup → 拿 token → 调 `/api/devices/register`
- **Commit：** `feat(user): 登录后注册 push token`

### Phase HH · dogfood + 数值调优

#### Task HH1：端到端测试
- **覆盖：**
  1. 用户登录 + 注册 device token
  2. 模拟 8:00 触发 greeting → 用户在线收到 WS push
  3. 用户离线（关 WS）→ 触发 greeting → 收到 APNs 推送
  4. 用户点推送回 app → /api/messages/sync 拉到主动消息
  5. 24h 不打开 → 收到 b_recall（关心档）
  6. 3 天不打开 → 收到 b_recall（撒娇档）
  7. 用户对话提"明天面试" → 24h 后收到 c_event_followup
- **Commit：** `test: Plan 4 端到端覆盖 4 场景 + 4 层投递`

#### Task HH2：dogfood 第四关
- **目标：** 自己用 14 天，记录：
  - [ ] 主动消息频率合适吗？会不会觉得"骚扰"？
  - [ ] 每个场景的内容感觉自然吗？还是机械感？
  - [ ] APNs / 个推 投递率（看后端日志统计）
  - [ ] 半小时不出戏总目标完成度
- **Commit：** `docs: Plan 4 dogfood 反馈 + 一期收尾`

---

## Self-Review Checklist

- [x] Spec §5.1 4 个主动场景全覆盖
- [x] Spec §5.2 4 层可靠投递（L1 WS + L2 ACK 超时 + L3 推送 + L4 已在 Plan 1）
- [x] DDD 边界：proactive 单独 -api/-core；push 单独 -api/-core；不破坏其他领域
- [x] 配套 rule：`java-backend-business-layer.md` `flutter-business-layer.md` 严格遵守

---

## 一期收尾

Plan 4 完成 = 一期 MVP 全部跑通。下一步：

1. **算法备案完成 + 商标查重**（行政事项，与开发并行做的，到此应已落地）
2. **iOS 国区上架准备**（隐私政策 / 用户协议 / 应用商店素材）
3. **Android 官网下载页 + APK 签名 / 升级机制**
4. **数据备份验证**（实际跑一次 pg_dump 恢复演练）
5. **性能压测**（100 / 500 / 1000 并发）
6. **正式上线**（这部分交给独立"上线 plan"，不在本 plan 范围）

---

## Execution Handoff

**Plan 1+2+3 都跑完后再启动 Plan 4。**
