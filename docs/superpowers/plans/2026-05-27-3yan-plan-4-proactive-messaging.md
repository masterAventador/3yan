# 三言 · Plan 4 实现计划：主动消息系统（4 场景 + 结构化记忆 + 4 层投递）

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development`（推荐）逐 task 实现。Steps 用 `- [ ]` checkbox。
> **TDD 铁律：** 每个有业务逻辑的 task 严格 Red → Green → Refactor。Migration / pom / 配置类属 TDD 例外（配置文件），但 migration 仍先写 schema IT 验证。
> **本计划取代** `docs/superpowers/plans/2026-05-06-3yan-plan-4-proactive-messaging.md`（脱节草稿）。
> **设计依据：** `docs/superpowers/specs/2026-05-27-plan-4-proactive-messaging-design.md`

**Goal:** 小婉会主动找你——早晚安、失联召回、事件追问、情绪关怀，按亲密度阶段控频，在线 WS 投递 + 离线 sync 兜底。

**Architecture:** memory 域扩展（结构化记忆 `memory_item` + 实时抽取）；新建 proactive 域（4 触发器 + 4 生成器 + 调度循环 + 频率门控）；新建 push 域（推送结构骨架，实推待证书）；chat 域扩展（4 层投递 DeliveryService + WS ACK）；common-ws 扩展（在线状态心跳 TTL）。跨域全走 -api 契约 + 事件，不碰别人 -core（Maven Enforcer 守护）。

**Tech Stack:** Java 21 + Spring Boot 3.x + Maven 多模块 + PostgreSQL（Flyway migration）+ Redis（KvCache）+ Testcontainers PG（IT）+ DeepSeek（LLM）+ Flutter（前端 ACK 回报）。

> 注：项目 JDK 2026-05-27 从 17 升到 21（与线上一致）。本 plan 代码未使用 17+ 新特性，向下兼容；执行时以 server pom 实际 `java.version` 为准。

---

## 0 · 现状缺口 & 解决决策（写代码前必读）

子代理核对现状后确认的 7 处"需新建/改造"（避免重蹈草稿脱节）：

| # | 缺口 | 解决决策 | 落在 |
|---|---|---|---|
| 1 | `ChatApi` 无投递方法 | 新增 `deliverProactiveMessage(userId, characterId, List<String> segments)` + chat-core 新建 `DeliveryService` | Phase H |
| 2 | `MemoryApi` 无条目查询、memory-api 无 `event/` 包 | 新增 `getMemoryItem` / `markMemoryItemDone` + 新建 `MemoryItemDto` + 新建 `MemoryItemScheduledEvent` | Phase F |
| 3 | L2 ACK 兜底完全不存在（客户端不回 ACK、服务端不处理入站 ACK、`WsMessage` 无 ack 字段） | 全新建：`WsMessage` 加 `ackMsgId` 字段 + handler 处理入站 `ack` 帧 → complete `CompletableFuture`；`DeliveryService` 维护 `Map<msgId, CompletableFuture>` + 5s 超时 | Phase H + M（前端回 ACK） |
| 4 | `KvCache` 无原子 INCR（每日上限计数需要） | 给 `KvCache` 加 `long increment(String key, Duration ttlIfNew)`（基础层改动，跑全量） | Phase C |
| 5 | Controller 不走全局自动包装 | 照现状手动 `BaseResp.success(data)` 返回 | Phase G（Controller） |
| 6 | `SessionManager` 的 `ws:online` 无 TTL、`isOnline` 与 `getSession` 口径不一致 | `register` 带 TTL + handler PING 分支续期 + 统一 `isOnline` 口径（基础层改动，跑全量） | Phase E |
| 7 | 新增 migration 需同步所有副本 | V9/V10/V11 主拷贝（bootstrap）+ 同步到 memory-core / character-core 已有副本 + 新建 proactive-core / push-core 副本，各配 `FlywayMigrationSyncTest` | Phase A/B |

**两个 LLM 调用的 taskType（spec 已定）：** 记忆抽取走 `BACKGROUND`（DeepSeek）；主动消息文案生成走 `USER_FACING`（主链路）。

---

## 文件结构映射

### 新建模块（4 个 Maven 子模块）

```
business_packages/
├── sanyan-proactive-api/          # 契约：ProactiveApi（薄，供测试/未来手动触发）+ dto
│   └── src/main/java/com/sanyan/proactive/
│       ├── ProactiveApi.java
│       └── dto/ProactiveTriggerResult.java
├── sanyan-proactive-core/         # 实现：触发器 + 生成器 + 调度 + 分发 + 门控
│   └── src/main/java/com/sanyan/proactive/
│       ├── internal/
│       │   ├── EventPendingEntity.java / EventPendingRepository.java
│       │   ├── EventType.java / EventStatus.java
│       │   ├── ProactiveErrCode.java
│       │   ├── ProactiveProperties.java        # @ConfigurationProperties("sanyan.proactive")
│       │   ├── ProactiveScheduler.java         # @Scheduled 30s 主循环
│       │   ├── ProactiveDispatcher.java        # 取 event → 门控 → generate → deliver
│       │   ├── FrequencyGate.java              # 每日上限 + stage 开关 + 免打扰
│       │   └── generator/
│       │       ├── ProactiveGenerator.java (接口) / GenerateContext.java / ProactiveContent.java
│       │       ├── GreetingGenerator.java / RecallGenerator.java
│       │       ├── EventFollowupGenerator.java / EmotionCareGenerator.java
│       │       └── ProactivePromptBuilder.java
│       ├── trigger/
│       │   ├── GreetingDailyTrigger.java / RecallTrigger.java
│       │   └── MemoryItemScheduledListener.java   # 订阅 MemoryItemScheduledEvent → 排 c/d
│       └── api/ProactiveApiImpl.java
├── sanyan-push-api/               # 契约：PushChannel + PushRouter 接口 + dto
│   └── src/main/java/com/sanyan/push/
│       ├── PushApi.java
│       ├── PushChannel.java (接口)
│       └── dto/{PushPayload,DeviceTokenDto,PushResult}.java
└── sanyan-push-core/              # 实现：device_tokens + Controller + Router + ApnsChannel 骨架
    └── src/main/java/com/sanyan/push/
        ├── internal/
        │   ├── DeviceTokenEntity.java / DeviceTokenRepository.java
        │   ├── PushErrCode.java / PushRouter.java
        │   └── apns/ApnsPushChannel.java       # pushy 骨架，实推待证书
        ├── web/DeviceTokenController.java       # POST /api/devices/register
        └── api/PushApiImpl.java
```

### 修改现有模块

```
foundation_packages/
├── sanyan-common-cache/.../KvCache.java          # 加 increment()
└── sanyan-common-ws/.../SessionManager.java      # ws:online 加 TTL + 统一 isOnline

business_packages/
├── sanyan-memory-api/.../MemoryApi.java           # 加 getMemoryItem / markMemoryItemDone
│   ├── dto/MemoryItemDto.java                      # 新建
│   └── event/MemoryItemScheduledEvent.java        # 新建（新建 event/ 包）
├── sanyan-memory-core/
│   ├── internal/item/MemoryItemEntity.java / Repository.java / MemoryItemKind.java / MemoryItemStatus.java
│   ├── internal/item/MemoryItemExtractService.java   # LLM 抽取 + 去重
│   ├── internal/item/MemoryItemExtractResult.java     # LLM JSON 解析 DTO
│   ├── event/MemoryItemExtractListener.java          # 挂 MessagePersistedEvent
│   └── api/MemoryApiImpl.java                          # 实现新方法
├── sanyan-chat-api/.../ChatApi.java               # 加 deliverProactiveMessage
├── sanyan-chat-core/
│   ├── internal/DeliveryService.java              # 新建：4 层投递
│   ├── ws/ChatWebSocketHandler.java               # 加入站 ack 帧处理 + last_active 刷新
│   ├── ws/WsProactiveMessage.java                 # 新建：主动消息推送帧
│   └── api/ChatApiImpl.java                        # 实现 deliverProactiveMessage
├── sanyan-character-core/.../web/RelationshipController.java  # 刷 last_active
└── (common-ws) ChatWebSocketHandler afterConnectionEstablished # 刷 last_active

bootstrap/
├── src/main/resources/db/migration/V9,V10,V11    # 主拷贝
├── src/main/resources/application.yml             # sanyan.proactive.* 配置
└── pom.xml                                         # 加 4 个新模块依赖

各模块 src/test/resources/db/migration/            # V9/V10/V11 副本同步

app/  (Flutter)
└── business_packages/sanyan_chat/lib/src/chat/chat_controller.dart  # 收到主动消息回 ACK
```

### last_active Redis（基础设施）

- key `user:last_active:{userId}` → ISO 时间戳；更新点：WS 连接、收到用户消息、GET /me
- 新建 `LastActiveTracker`（放 common-ws 或 chat-core？→ 放 **common-ws**，与 SessionManager 同层，跨域可复用）

---

## Phase 概览与依赖

| Phase | 内容 | 依赖 | 测试粒度 |
|---|---|---|---|
| A | 4 新模块脚手架（pom）| — | mvn compile |
| B | migration V9/V10/V11 + 副本 + schema IT | A | 各模块 IT |
| C | KvCache.increment（基础层）| — | **全量** |
| D | last_active（LastActiveTracker + 接入）| C | common-ws + chat + character |
| E | common-ws 心跳 TTL（基础层）| — | **全量** |
| F | memory 域结构化记忆 + 实时抽取 | B,C | memory-core |
| G | push 域（结构骨架）| A,B | push-core |
| H | chat 域 4 层投递 + WS ACK | E,G | chat-core |
| I | proactive 基础（EventPending/Properties/ErrCode）| A,B | proactive-core |
| J | proactive 调度 + 门控 + 分发 | C,F,H,I | proactive-core |
| K | proactive 生成器（4 个）| I,J | proactive-core |
| L | proactive 触发器（4 个）| F,I,J | proactive-core |
| M | 前端 Flutter ACK 回报 + device token 注册 | H | sanyan_chat |
| N | 端到端 + ERROR_CODE_REGISTRY + final gate | 全部 | **全量 mvn verify + flutter test** |

> **Phase checkpoint**：A-B 后、F 后、H 后、L 后、N 各跑一次相关全量。涉及 foundation（C/E）的 Phase 必跑 server 全量。

---

## Phase A · 模块脚手架

> 4 个新模块的 pom + 注册父 pom + bootstrap 依赖。属配置（TDD 例外），验证 = `mvn compile` 通过、Enforcer 不报错。

### Task A1: 新建 sanyan-proactive-api 模块

**Files:**
- Create: `server/business_packages/sanyan-proactive-api/pom.xml`
- Create: `server/business_packages/sanyan-proactive-api/src/main/java/com/sanyan/proactive/ProactiveApi.java`
- Create: `server/business_packages/sanyan-proactive-api/src/main/java/com/sanyan/proactive/dto/ProactiveTriggerResult.java`
- Modify: `server/pom.xml`（`<modules>` + `<dependencyManagement>`）

- [ ] **Step 1: pom.xml（照 sanyan-llm-api 模板，-api 零依赖）**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.sanyan</groupId>
        <artifactId>sanyan-server-parent</artifactId>
        <version>0.1.0</version>
        <relativePath>../../pom.xml</relativePath>
    </parent>
    <artifactId>sanyan-proactive-api</artifactId>
    <packaging>jar</packaging>
</project>
```

> 注：`<parent>` 的 groupId/artifactId/version 必须与 `server/pom.xml` 实际值一致——实现时先读 `server/pom.xml` 顶部确认（这里按惯例写，A1 实现者务必核对替换）。

- [ ] **Step 2: ProactiveApi.java（薄契约）**

```java
package com.sanyan.proactive;

import com.sanyan.proactive.dto.ProactiveTriggerResult;

/**
 * 主动消息域对内契约。proactive 主要靠 @Scheduled + 事件订阅驱动，
 * 本接口供测试 / 未来 admin 手动触发用。
 */
public interface ProactiveApi {
    /** 手动触发一次某用户的指定场景调度（测试 / 运维用）。 */
    ProactiveTriggerResult triggerNow(Long userId, Long characterId, String eventType);
}
```

```java
package com.sanyan.proactive.dto;

public record ProactiveTriggerResult(boolean scheduled, String reason) {}
```

- [ ] **Step 3: 注册父 pom**

在 `server/pom.xml` 的 `<modules>` 加：
```xml
<module>business_packages/sanyan-proactive-api</module>
```
在 `<dependencyManagement>` 加：
```xml
<dependency>
    <groupId>com.sanyan</groupId>
    <artifactId>sanyan-proactive-api</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 4: 验证编译**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-api -am compile`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-api server/pom.xml
git commit -m "build(proactive): 新建 sanyan-proactive-api 模块脚手架"
```

---

### Task A2: 新建 sanyan-proactive-core 模块

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/pom.xml`
- Create: 占位 `src/main/java/com/sanyan/proactive/internal/package-info.java`
- Modify: `server/pom.xml` + `server/bootstrap/pom.xml`

- [ ] **Step 1: pom.xml（照 sanyan-character-core 模板）**

依赖：本域 `sanyan-proactive-api` + 跨域 `sanyan-memory-api` / `sanyan-character-api` / `sanyan-chat-api` / `sanyan-push-api` + foundation（`sanyan-common-error` / `sanyan-common-web` / `sanyan-common-cache`）+ `spring-boot-starter-data-jpa` + `postgresql`(runtime) + `lombok` + `spring-boot-starter-data-redis`；test 加 `sanyan-common-test`(test) + `spring-boot-starter-test`(test) + `testcontainers` + `junit-jupiter`(test)。`<build>` 显式声明 `maven-failsafe-plugin`（空配置激活 IT）。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.sanyan</groupId>
        <artifactId>sanyan-server-parent</artifactId>
        <version>0.1.0</version>
        <relativePath>../../pom.xml</relativePath>
    </parent>
    <artifactId>sanyan-proactive-core</artifactId>
    <packaging>jar</packaging>
    <dependencies>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-proactive-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-memory-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-character-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-chat-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-push-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-error</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-web</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-cache</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-redis</artifactId></dependency>
        <dependency><groupId>org.postgresql</groupId><artifactId>postgresql</artifactId><scope>runtime</scope></dependency>
        <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId><scope>provided</scope></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.testcontainers</groupId><artifactId>postgresql</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.testcontainers</groupId><artifactId>junit-jupiter</artifactId><scope>test</scope></dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin><groupId>org.apache.maven.plugins</groupId><artifactId>maven-failsafe-plugin</artifactId></plugin>
        </plugins>
    </build>
</project>
```

> 实现者：以 `sanyan-character-core/pom.xml` 为准核对 foundation 依赖清单（common-error/web/cache 等 artifactId 拼写）和 testcontainers 版本管理方式，对齐后再写。

- [ ] **Step 2: package-info 占位**

```java
/** 主动消息域实现层。 */
package com.sanyan.proactive.internal;
```

- [ ] **Step 3: 注册父 pom + bootstrap**

`server/pom.xml`：`<modules>` 加 `<module>business_packages/sanyan-proactive-core</module>`；`<dependencyManagement>` 加 proactive-core。
`server/bootstrap/pom.xml`：`<dependencies>` 加：
```xml
<dependency><groupId>com.sanyan</groupId><artifactId>sanyan-proactive-core</artifactId></dependency>
```

- [ ] **Step 4: 验证编译 + Enforcer**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core -am compile && mvn -q validate`
Expected: BUILD SUCCESS（Enforcer 不报 banned-dependencies，因为只依赖 -api 不依赖别人 -core）

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core server/pom.xml server/bootstrap/pom.xml
git commit -m "build(proactive): 新建 sanyan-proactive-core 模块脚手架"
```

---

### Task A3: 新建 sanyan-push-api 模块

**Files:**
- Create: `server/business_packages/sanyan-push-api/pom.xml`
- Create: `.../com/sanyan/push/PushApi.java` / `PushChannel.java` / `dto/{PushPayload,DeviceTokenDto,PushResult}.java`
- Modify: `server/pom.xml`

- [ ] **Step 1: pom.xml**（同 A1 -api 模板，artifactId `sanyan-push-api`，零依赖）

- [ ] **Step 2: 契约类**

```java
package com.sanyan.push;

import com.sanyan.push.dto.DeviceTokenDto;
import com.sanyan.push.dto.PushPayload;
import com.sanyan.push.dto.PushResult;
import java.util.List;

/** 推送域对内契约：注册设备 token + 给用户推送。 */
public interface PushApi {
    void registerDevice(Long userId, String platform, String vendor, String token);
    /** 给用户所有 active 设备推送；本期 L3 实推为占位。 */
    PushResult pushToUser(Long userId, PushPayload payload);
    List<DeviceTokenDto> listActiveTokens(Long userId);
}
```

```java
package com.sanyan.push;

import com.sanyan.push.dto.DeviceTokenDto;
import com.sanyan.push.dto.PushPayload;
import com.sanyan.push.dto.PushResult;

/** 单一推送通道（APNs / 个推 / ...）。本期只 APNs 骨架。 */
public interface PushChannel {
    String vendor();                                  // "apns" / "getui"
    boolean supports(String platform);                // "ios" / "android"
    PushResult sendToDevice(DeviceTokenDto token, PushPayload payload);
}
```

```java
package com.sanyan.push.dto;

public record PushPayload(String title, String body, Long messageId) {}
```

```java
package com.sanyan.push.dto;

public record DeviceTokenDto(Long userId, String platform, String vendor, String token) {}
```

```java
package com.sanyan.push.dto;

/** 推送结果。本期 status 可为 PENDING（占位未实推）。 */
public record PushResult(String status, String detail) {
    public static PushResult pending(String detail) { return new PushResult("PENDING", detail); }
    public static PushResult sent()                 { return new PushResult("SENT", null); }
    public static PushResult failed(String detail)  { return new PushResult("FAILED", detail); }
}
```

- [ ] **Step 3: 注册父 pom**（modules + dependencyManagement，加 sanyan-push-api）

- [ ] **Step 4: 验证编译** `cd server && mvn -q -pl business_packages/sanyan-push-api -am compile`

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-push-api server/pom.xml
git commit -m "build(push): 新建 sanyan-push-api 模块脚手架 + 推送契约"
```

---

### Task A4: 新建 sanyan-push-core 模块

**Files:**
- Create: `server/business_packages/sanyan-push-core/pom.xml`
- Create: 占位 `.../com/sanyan/push/internal/package-info.java`
- Modify: `server/pom.xml` + `server/bootstrap/pom.xml`

- [ ] **Step 1: pom.xml**（照 A2 -core 模板；依赖本域 `sanyan-push-api` + foundation（common-error/web/auth/cache）+ JPA + postgresql + lombok + redis；test 同 A2；额外加 APNs 库 `com.eatthepath:pushy`——版本在父 pom dependencyManagement 里加，或这里写明版本，实现者确认最新稳定版）

> APNs 依赖：
> ```xml
> <dependency><groupId>com.eatthepath</groupId><artifactId>pushy</artifactId><version>0.15.4</version></dependency>
> ```
> 实现者核对 pushy 最新稳定版。

- [ ] **Step 2: package-info 占位**

```java
/** 推送域实现层。 */
package com.sanyan.push.internal;
```

- [ ] **Step 3: 注册父 pom + bootstrap**（modules + dependencyManagement + bootstrap 依赖 push-core）

- [ ] **Step 4: 验证** `cd server && mvn -q -pl business_packages/sanyan-push-core -am compile && mvn -q validate`

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-push-core server/pom.xml server/bootstrap/pom.xml
git commit -m "build(push): 新建 sanyan-push-core 模块脚手架"
```

---

## 附录 · 共享契约定义（所有 Phase 引用，类型以本附录为准）

> **约定：所有枚举列在 DB 存 Java enum 的 `name()`（大写 SNAKE）**，Entity 用 `@Enumerated(EnumType.STRING)`，CHECK 约束用大写值——比 spec §3.1 的小写示例统一为大写，减少映射摩擦。

### A. Entity / enum

```java
// memory-core/internal/item/MemoryItemKind.java
public enum MemoryItemKind { PLAN_EVENT, EMOTION, PROMISE }
// memory-core/internal/item/MemoryItemStatus.java
public enum MemoryItemStatus { PENDING, DONE, EXPIRED }

// memory-core/internal/item/MemoryItemEntity.java  (table: memory_item)
@Data @Entity @Table(name = "memory_item")
class MemoryItemEntity {
    @Id @GeneratedValue(strategy = IDENTITY) Long id;
    @Column(name="user_id", nullable=false) Long userId;
    @Column(name="character_id", nullable=false) Long characterId;
    @Enumerated(STRING) @Column(nullable=false, length=20) MemoryItemKind kind;
    @Column(nullable=false, columnDefinition="text") String content;
    @Column(name="salient_at", nullable=false) Instant salientAt;
    @Enumerated(STRING) @Column(nullable=false, length=20) MemoryItemStatus status = MemoryItemStatus.PENDING;
    @Column(name="source_message_id") Long sourceMessageId;
    @CreationTimestamp @Column(name="created_at") Instant createdAt;
}
// Repository
List<MemoryItemEntity> findByUserIdAndCharacterIdAndStatus(Long u, Long c, MemoryItemStatus s);

// proactive-core/internal/EventType.java
public enum EventType { A_GREETING, B_RECALL, C_EVENT_FOLLOWUP, D_EMOTION_CARE }
// proactive-core/internal/EventStatus.java
public enum EventStatus { SCHEDULED, PROCESSING, SENT, FAILED, CANCELLED }

// proactive-core/internal/EventPendingEntity.java  (table: events_pending)
@Data @Entity @Table(name = "events_pending")
class EventPendingEntity {
    @Id @GeneratedValue(strategy = IDENTITY) Long id;
    @Column(name="user_id", nullable=false) Long userId;
    @Column(name="character_id", nullable=false) Long characterId;
    @Enumerated(STRING) @Column(name="event_type", nullable=false, length=40) EventType eventType;
    @Enumerated(STRING) @Column(nullable=false, length=20) EventStatus status = EventStatus.SCHEDULED;
    @JdbcTypeCode(SqlTypes.JSON) @Column(columnDefinition="jsonb", nullable=false) String payload = "{}";
    @Column(name="scheduled_at", nullable=false) Instant scheduledAt;
    @Column(name="sent_at") Instant sentAt;
    @Column(name="fail_count", nullable=false) Integer failCount = 0;
    @Column(name="last_error", columnDefinition="text") String lastError;
    @CreationTimestamp @Column(name="created_at") Instant createdAt;
}

// push-core/internal/DeviceTokenEntity.java  (table: device_tokens)
@Data @Entity @Table(name="device_tokens",
  uniqueConstraints=@UniqueConstraint(columnNames={"user_id","platform","vendor","token"}))
class DeviceTokenEntity {
    @Id @GeneratedValue(strategy=IDENTITY) Long id;
    @Column(name="user_id", nullable=false) Long userId;
    @Column(nullable=false, length=20) String platform;     // ios / android
    @Column(nullable=false, length=20) String vendor;       // apns / getui / ...
    @Column(nullable=false, length=500) String token;
    @Column(nullable=false) Boolean active = true;
    @CreationTimestamp @Column(name="registered_at") Instant registeredAt;
    @Column(name="last_seen") Instant lastSeen;
}
```

### B. 跨域契约（-api 新增）

```java
// memory-api/dto/MemoryItemDto.java
public record MemoryItemDto(Long id, Long userId, Long characterId,
                            String kind, String content, Instant salientAt, String status) {}
// memory-api/event/MemoryItemScheduledEvent.java  (新建 event/ 包)
public record MemoryItemScheduledEvent(Long memoryItemId, Long userId, Long characterId,
                                       String kind, Instant salientAt) {}
// memory-api/MemoryApi.java 新增方法
MemoryItemDto getMemoryItem(Long itemId);     // 不存在抛 BusinessException
void markMemoryItemDone(Long itemId);

// chat-api/ChatApi.java 新增方法
/** 落库每条 segment 为 ai message + 4 层投递；返回落库 messageId 列表。 */
List<Long> deliverProactiveMessage(Long userId, Long characterId, List<String> segments);
```

### C. chat-core 投递 + WS ACK

```java
// WsMessage.java 加字段（入站帧）
private Long ackMsgId;   // 客户端 ACK：确认已收到的 serverMsgId

// ChatWebSocketHandler.handleTextMessage switch 加：
case WsEventType.ACK -> deliveryService.confirmAck(wsMsg.getAckMsgId());

// 主动消息复用现有 WsNewMessage 帧推送（客户端当新消息显示并回 ACK）

// DeliveryService.java (chat-core/internal)
@Service class DeliveryService {
    List<Long> deliver(Long userId, Long characterId, List<String> segments); // 落库 + 逐条 L1→L2→L3
    void confirmAck(Long messageId);                                          // 收到客户端 ACK，complete future
    // 内部 Map<Long, CompletableFuture<Void>> pendingAcks；L2 超时 5s（future.orTimeout）
}
```

### D. proactive-core 内部契约

```java
// generator/ProactiveGenerator.java
public interface ProactiveGenerator {
    EventType supportsType();
    List<String> generate(GenerateContext ctx);   // 返回多条 segment
}
// generator/GenerateContext.java
public record GenerateContext(Long userId, Long characterId,
        RelationshipDto relationship,   // 来自 character-api（含 currentStage / currentStageName）
        String stagePromptSegment,      // character-api.getStagePromptSegment
        MemoryContext memoryContext,    // memory-api.getRelevantContext
        java.util.Map<String,Object> payload) {}

// FrequencyGate.java
@Service class FrequencyGate {
    /** 发送前门控：每日上限 + stage 场景开关 + 免打扰。放行 true。 */
    boolean allow(Long userId, Long characterId, EventType type, int currentStage);
    void recordSent(Long userId);   // 发送后每日计数 +1
}
```

### E. 基础层改动签名

```java
// common-cache/KvCache.java 新增
/** 原子自增；key 不存在则置 1 并设 TTL。返回自增后的值。 */
long increment(String key, Duration ttlIfNew);

// common-ws/LastActiveTracker.java （新建，与 SessionManager 同层）
@Component class LastActiveTracker {
    void touch(Long userId);                  // set user:last_active:{id}=now
    Optional<Instant> lastActiveAt(Long userId);
}

// common-ws/SessionManager.java 改动
private static final Duration ONLINE_TTL = Duration.ofSeconds(90);
// register(): redis.opsForValue().set(ONLINE_PREFIX+userId, "1", ONLINE_TTL);
void refreshOnline(Long userId);   // 刷新 TTL，handler 收到 PING 时调
// isOnline() 保持查 redis（TTL 过期自动 false，与心跳一致）
```

### F. 错误码

```java
// proactive-core/internal/ProactiveErrCode.java (7000-7999)
PROACTIVE_GENERATE_FAILED(7001, "主动消息生成失败"),
PROACTIVE_EVENT_NOT_FOUND (7002, "主动事件不存在"),
// push-core/internal/PushErrCode.java (8000-8999)
DEVICE_TOKEN_INVALID(8001, "设备 token 无效"),
PUSH_SEND_FAILED   (8002, "推送发送失败"),
```

### G. Redis key 规范

| key | 用途 | TTL |
|---|---|---|
| `ws:online:{userId}` | 在线标记（心跳续期）| 90s |
| `user:last_active:{userId}` | 最后活跃时间戳 | 30d |
| `proactive:sent:{userId}:{yyyy-MM-dd}` | 当日已发主动消息数（INCR）| 36h |
| `proactive:recall:{userId}:{level}` | 召回阶梯去重（24h/3d/7d 各一次）| 8d |

### H. ProactiveProperties 结构（`sanyan.proactive.*`）

```yaml
sanyan:
  proactive:
    greeting: { morningCron: "0 0 8 * * *", nightCron: "0 30 22 * * *" }
    recall:   { thresholdsHours: [24, 72, 168] }          # 24h / 3天 / 7天
    quietHours: { start: 23, end: 8 }
    scatterWindowMinutes: 30
    dailyCapByStage: [2, 3, 4, 5, 6]                       # stage 0..4（spec §6.1 +1 后）
    scenesByStage:                                         # 每 stage 允许的场景
      0: { morning: false, night: false, recall: true,  eventFollowup: true,  emotionCare: false }
      1: { morning: false, night: true,  recall: true,  eventFollowup: true,  emotionCare: true  }
      2: { morning: true,  night: true,  recall: true,  eventFollowup: true,  emotionCare: true  }
      3: { morning: true,  night: true,  recall: true,  eventFollowup: true,  emotionCare: true  }
      4: { morning: true,  night: true,  recall: true,  eventFollowup: true,  emotionCare: true  }
```

---

### Phase B · 数据库 migration（V9/V10/V11 + 副本同步 + schema IT）

> 依赖 Phase A 已建 proactive-core / push-core 模块（含各自 `pom.xml` + 模块注册）。
> Migration SQL 属 TDD 例外（配置文件），但每张表仍**先写 schema IT 验证字段/类型/索引**（照 `RelationshipsSchemaIT` / `IntimacyLogsSchemaIT`：`extends PostgresTestcontainerSupport` + `runMigrationsUpTo("N")` + `describeColumns`）。
> **类型对照表（`describeColumns` 返回的 `udt_name` / `typeName()`）：** `BIGSERIAL`→`bigserial`、`BIGINT`→`bigint`、`INT`→`int4`、`SMALLINT`→`int2`、`BOOLEAN`→`bool`、`VARCHAR(n)`→`varchar`、`TEXT`→`text`、`JSONB`→`jsonb`、`TIMESTAMPTZ`→`timestamptz`、`TIMESTAMP`→`timestamp`。
> **副本同步规则（缺口 #7）：** 主拷贝在 `bootstrap/src/main/resources/db/migration/`；现状 memory-core / character-core 各有 V1-V8 全量副本。新增 V9/V10/V11 时：① 写主拷贝 ② **逐字节**复制到 memory-core + character-core 副本 ③ **新建** proactive-core / push-core 的 `src/test/resources/db/migration/` 目录，放 **V1-V11 全量副本**（不是只放新增的——schema IT 用 `runMigrationsUpTo` 会从 V1 开始全跑）。
> 副本一致性由各模块的 `FlywayMigrationSyncTest`（plain JUnit，逐字节比对 bootstrap 与本模块副本）守护。

#### Task B1: V9 migration — memory_item 表 + 副本同步 + schema IT

**Files:**
- Create: `server/bootstrap/src/main/resources/db/migration/V9__create_memory_item.sql`（主拷贝）
- Create: `server/business_packages/sanyan-memory-core/src/test/resources/db/migration/V9__create_memory_item.sql`（逐字节副本）
- Create: `server/business_packages/sanyan-character-core/src/test/resources/db/migration/V9__create_memory_item.sql`（逐字节副本）
- Create: `server/business_packages/sanyan-proactive-core/src/test/resources/db/migration/V1__initial_schema.sql` … `V9__create_memory_item.sql`（**V1-V9 全量副本**，逐字节复制自 bootstrap）
- Create: `server/business_packages/sanyan-push-core/src/test/resources/db/migration/V1__initial_schema.sql` … `V9__create_memory_item.sql`（**V1-V9 全量副本**，逐字节复制自 bootstrap）
- Test: `server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/internal/item/MemoryItemSchemaIT.java`

- [ ] **Step 1: 先写失败的 schema IT（验全字段 + 类型 + 索引）**

照 `IntimacyLogsSchemaIT` 风格——`extends PostgresTestcontainerSupport`，无 Spring 注解，`runMigrationsUpTo("9")` + `describeColumns`。

```java
package com.sanyan.memory.internal.item;

import com.sanyan.common.test.PostgresTestcontainerSupport;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * V9 migration 验证：memory_item 结构化记忆表（kind / status 大写 enum CHECK + 双索引）。
 */
class MemoryItemSchemaIT extends PostgresTestcontainerSupport {

    @Test
    void v9_should_create_memory_item_with_kind_status_and_indexes() throws Exception {
        runMigrationsUpTo("9");

        Map<String, ColumnSpec> cols = describeColumns("memory_item");

        // 9 列存在
        assertThat(cols).containsKeys("id", "user_id", "character_id", "kind", "content",
                "salient_at", "status", "source_message_id", "created_at");

        // 类型
        assertThat(cols.get("id").typeName()).isEqualTo("bigserial");
        assertThat(cols.get("user_id").typeName()).isEqualTo("bigint");
        assertThat(cols.get("character_id").typeName()).isEqualTo("bigint");
        assertThat(cols.get("kind").typeName()).isEqualTo("varchar");
        assertThat(cols.get("content").typeName()).isEqualTo("text");
        assertThat(cols.get("salient_at").typeName()).isEqualTo("timestamptz");
        assertThat(cols.get("status").typeName()).isEqualTo("varchar");
        assertThat(cols.get("source_message_id").typeName()).isEqualTo("bigint");
        assertThat(cols.get("created_at").typeName()).isEqualTo("timestamptz");

        // nullability（source_message_id 唯一可空）
        assertThat(cols.get("id").nullable()).isFalse();
        assertThat(cols.get("user_id").nullable()).isFalse();
        assertThat(cols.get("character_id").nullable()).isFalse();
        assertThat(cols.get("kind").nullable()).isFalse();
        assertThat(cols.get("content").nullable()).isFalse();
        assertThat(cols.get("salient_at").nullable()).isFalse();
        assertThat(cols.get("status").nullable()).isFalse();
        assertThat(cols.get("source_message_id").nullable()).isTrue();
        assertThat(cols.get("created_at").nullable()).isFalse();

        // 两个索引存在
        assertThat(indexExists("memory_item", "idx_memory_item_salient")).isTrue();
        assertThat(indexExists("memory_item", "idx_memory_item_user")).isTrue();

        // CHECK 约束用大写 enum 值（PLAN_EVENT / PENDING）
        String checkClause = checkConstraintsOf("memory_item");
        assertThat(checkClause).contains("PLAN_EVENT", "EMOTION", "PROMISE");
        assertThat(checkClause).contains("PENDING", "DONE", "EXPIRED");
    }

    private boolean indexExists(String table, String index) throws Exception {
        try (Connection conn = newConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "SELECT COUNT(*) FROM pg_indexes WHERE tablename=? AND indexname=?")) {
            ps.setString(1, table);
            ps.setString(2, index);
            try (ResultSet rs = ps.executeQuery()) {
                rs.next();
                return rs.getInt(1) == 1;
            }
        }
    }

    private String checkConstraintsOf(String table) throws Exception {
        StringBuilder sb = new StringBuilder();
        try (Connection conn = newConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "SELECT pg_get_constraintdef(c.oid) AS def "
                             + "FROM pg_constraint c JOIN pg_class t ON c.conrelid = t.oid "
                             + "WHERE t.relname = ? AND c.contype = 'c'")) {
            ps.setString(1, table);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    sb.append(rs.getString("def")).append('\n');
                }
            }
        }
        return sb.toString();
    }
}
```

- [ ] **Step 2: 运行确认失败**

```bash
cd server && mvn -pl business_packages/sanyan-memory-core -Dit.test=MemoryItemSchemaIT -Dtest='void' verify
```
Expected: FAIL —— Flyway 跑到 V9 时找不到 `V9__create_memory_item.sql`（`FlywayException: target version 9 ... not available`）或 `describeColumns("memory_item")` 返回空 Map（表不存在）。

> 注：本仓库 schema IT 命名 `*IT`，由 failsafe 在 `verify` 阶段执行；用 `-Dit.test=` 指定单个 IT。若实现者改用 surefire（`*Test`）习惯则用 `-Dtest=`，按现状 character-core IT 已是 `verify` 路径。

- [ ] **Step 3: 写 migration（最小实现）+ 同步副本**

主拷贝 `V9__create_memory_item.sql`（DDL 见 spec §3.1，**CHECK 用大写 enum 值**，时间列用 `TIMESTAMPTZ`）：

```sql
-- V9__create_memory_item.sql
-- memory 域结构化记忆：一条条带时间的、日后值得主动提起的点
CREATE TABLE memory_item (
    id                BIGSERIAL   PRIMARY KEY,
    user_id           BIGINT      NOT NULL,
    character_id      BIGINT      NOT NULL,
    kind              VARCHAR(20) NOT NULL CHECK (kind IN ('PLAN_EVENT','EMOTION','PROMISE')),
    content           TEXT        NOT NULL,
    salient_at        TIMESTAMPTZ NOT NULL,
    status            VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                      CHECK (status IN ('PENDING','DONE','EXPIRED')),
    source_message_id BIGINT,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_memory_item_salient
    ON memory_item (status, salient_at) WHERE status = 'PENDING';
CREATE INDEX idx_memory_item_user
    ON memory_item (user_id, character_id, status);
```

然后**逐字节复制**主拷贝到 4 个副本目录：
- `sanyan-memory-core/src/test/resources/db/migration/V9__create_memory_item.sql`
- `sanyan-character-core/src/test/resources/db/migration/V9__create_memory_item.sql`
- 新建 `sanyan-proactive-core/src/test/resources/db/migration/`，把 bootstrap 现有 **V1-V8 + 新 V9** 全部复制进来
- 新建 `sanyan-push-core/src/test/resources/db/migration/`，同样复制 **V1-V9** 全量

复制命令（保证逐字节一致）：

```bash
cd server
SRC=bootstrap/src/main/resources/db/migration
cp "$SRC/V9__create_memory_item.sql" business_packages/sanyan-memory-core/src/test/resources/db/migration/
cp "$SRC/V9__create_memory_item.sql" business_packages/sanyan-character-core/src/test/resources/db/migration/
mkdir -p business_packages/sanyan-proactive-core/src/test/resources/db/migration
mkdir -p business_packages/sanyan-push-core/src/test/resources/db/migration
cp "$SRC"/V*.sql business_packages/sanyan-proactive-core/src/test/resources/db/migration/
cp "$SRC"/V*.sql business_packages/sanyan-push-core/src/test/resources/db/migration/
```

- [ ] **Step 4: 运行确认通过**

```bash
cd server && mvn -pl business_packages/sanyan-memory-core -Dit.test=MemoryItemSchemaIT verify
```
Expected: BUILD SUCCESS（V9 跑通，9 列 + 双索引 + 两组 CHECK 大写值都命中）。

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/main/resources/db/migration/V9__create_memory_item.sql \
        server/business_packages/sanyan-memory-core/src/test/resources/db/migration/V9__create_memory_item.sql \
        server/business_packages/sanyan-character-core/src/test/resources/db/migration/V9__create_memory_item.sql \
        server/business_packages/sanyan-proactive-core/src/test/resources/db/migration/ \
        server/business_packages/sanyan-push-core/src/test/resources/db/migration/ \
        server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/internal/item/MemoryItemSchemaIT.java
git commit -m "feat(db): V9 创建 memory_item 结构化记忆表（kind/status 大写 enum CHECK + 双索引）+ 4 模块副本同步"
```

---

#### Task B2: V10 migration — events_pending 表 + proactive-core schema IT + SyncTest

**Files:**
- Create: `server/bootstrap/src/main/resources/db/migration/V10__create_events_pending.sql`（主拷贝）
- Create: 逐字节副本到 memory-core / character-core / proactive-core / push-core 各 `src/test/resources/db/migration/V10__create_events_pending.sql`
- Create: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/EventsPendingSchemaIT.java`
- Create: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/FlywayMigrationSyncTest.java`（**新建**，照 character-core 模板，包 `com.sanyan.proactive`）

- [ ] **Step 1: 先写失败的 schema IT + proactive-core SyncTest**

`EventsPendingSchemaIT`（`runMigrationsUpTo("10")`）：

```java
package com.sanyan.proactive.internal;

import com.sanyan.common.test.PostgresTestcontainerSupport;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * V10 migration 验证：events_pending 调度表（event_type / status 大写 enum CHECK + payload jsonb + due/dedup 索引）。
 */
class EventsPendingSchemaIT extends PostgresTestcontainerSupport {

    @Test
    void v10_should_create_events_pending_with_enums_jsonb_and_indexes() throws Exception {
        runMigrationsUpTo("10");

        Map<String, ColumnSpec> cols = describeColumns("events_pending");

        // 11 列存在
        assertThat(cols).containsKeys("id", "user_id", "character_id", "event_type", "status",
                "payload", "scheduled_at", "sent_at", "fail_count", "last_error", "created_at");

        // 类型
        assertThat(cols.get("id").typeName()).isEqualTo("bigserial");
        assertThat(cols.get("user_id").typeName()).isEqualTo("bigint");
        assertThat(cols.get("character_id").typeName()).isEqualTo("bigint");
        assertThat(cols.get("event_type").typeName()).isEqualTo("varchar");
        assertThat(cols.get("status").typeName()).isEqualTo("varchar");
        assertThat(cols.get("payload").typeName()).isEqualTo("jsonb");
        assertThat(cols.get("scheduled_at").typeName()).isEqualTo("timestamptz");
        assertThat(cols.get("sent_at").typeName()).isEqualTo("timestamptz");
        assertThat(cols.get("fail_count").typeName()).isEqualTo("int4");
        assertThat(cols.get("last_error").typeName()).isEqualTo("text");
        assertThat(cols.get("created_at").typeName()).isEqualTo("timestamptz");

        // nullability（sent_at / last_error 可空）
        assertThat(cols.get("event_type").nullable()).isFalse();
        assertThat(cols.get("status").nullable()).isFalse();
        assertThat(cols.get("payload").nullable()).isFalse();
        assertThat(cols.get("scheduled_at").nullable()).isFalse();
        assertThat(cols.get("fail_count").nullable()).isFalse();
        assertThat(cols.get("sent_at").nullable()).isTrue();
        assertThat(cols.get("last_error").nullable()).isTrue();

        // 索引
        assertThat(indexExists("events_pending", "idx_events_pending_due")).isTrue();
        assertThat(indexExists("events_pending", "idx_events_pending_dedup")).isTrue();

        // CHECK 用大写 enum（A_GREETING / SCHEDULED 等）
        String checks = checkConstraintsOf("events_pending");
        assertThat(checks).contains("A_GREETING", "B_RECALL", "C_EVENT_FOLLOWUP", "D_EMOTION_CARE");
        assertThat(checks).contains("SCHEDULED", "PROCESSING", "SENT", "FAILED", "CANCELLED");
    }

    private boolean indexExists(String table, String index) throws Exception {
        try (Connection conn = newConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "SELECT COUNT(*) FROM pg_indexes WHERE tablename=? AND indexname=?")) {
            ps.setString(1, table);
            ps.setString(2, index);
            try (ResultSet rs = ps.executeQuery()) {
                rs.next();
                return rs.getInt(1) == 1;
            }
        }
    }

    private String checkConstraintsOf(String table) throws Exception {
        StringBuilder sb = new StringBuilder();
        try (Connection conn = newConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "SELECT pg_get_constraintdef(c.oid) AS def "
                             + "FROM pg_constraint c JOIN pg_class t ON c.conrelid = t.oid "
                             + "WHERE t.relname = ? AND c.contype = 'c'")) {
            ps.setString(1, table);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    sb.append(rs.getString("def")).append('\n');
                }
            }
        }
        return sb.toString();
    }
}
```

`FlywayMigrationSyncTest`（proactive-core 版，照 character-core 模板，仅改包名/常量名/文案）：

```java
package com.sanyan.proactive;

import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Plan 4 B2 引入的 Flyway 副本同步守护测试（proactive-core 版）。
 *
 * <p>proactive-core 的 schema IT（如 {@code EventsPendingSchemaIT}）依赖
 * {@code PostgresTestcontainerSupport.runMigrationsUpTo(...)}，后者从 classpath
 * 加载 {@code db/migration}，所以本模块在 {@code src/test/resources/db/migration/}
 * 维护一份 V1-Vn 全量 SQL 副本。
 *
 * <p>本测试防止 bootstrap 主拷贝与本模块副本漂移——每次改 bootstrap 的 V*.sql
 * 必须同步本副本，否则本测试 fail 提醒。
 */
class FlywayMigrationSyncTest {

    private static final Path BOOTSTRAP_DIR = Path.of(
            "../../bootstrap/src/main/resources/db/migration");
    private static final Path PROACTIVE_TEST_DIR = Path.of(
            "src/test/resources/db/migration");

    @Test
    void flywayMigrationsMustBeInSyncBetweenBootstrapAndProactiveCoreTest() throws IOException {
        if (!Files.isDirectory(BOOTSTRAP_DIR)) {
            return;
        }
        List<Path> bootstrapSqls;
        List<Path> proactiveTestSqls;
        try (Stream<Path> bs = Files.list(BOOTSTRAP_DIR);
             Stream<Path> pt = Files.list(PROACTIVE_TEST_DIR)) {
            bootstrapSqls = bs.filter(p -> p.getFileName().toString().endsWith(".sql"))
                    .sorted().toList();
            proactiveTestSqls = pt.filter(p -> p.getFileName().toString().endsWith(".sql"))
                    .sorted().toList();
        }

        List<String> bootstrapNames = bootstrapSqls.stream()
                .map(p -> p.getFileName().toString()).sorted().toList();
        List<String> proactiveNames = proactiveTestSqls.stream()
                .map(p -> p.getFileName().toString()).sorted().toList();
        assertThat(proactiveNames)
                .as("proactive-core/src/test/resources/db/migration 必须与 bootstrap 主拷贝同名同数")
                .containsExactlyElementsOf(bootstrapNames);

        for (int i = 0; i < bootstrapSqls.size(); i++) {
            Path bs = bootstrapSqls.get(i);
            Path pt = proactiveTestSqls.get(i);
            assertThat(Files.readAllBytes(pt))
                    .as("文件 %s 在 bootstrap 与 proactive-core test 拷贝不一致——需同步",
                            bs.getFileName())
                    .containsExactly(Files.readAllBytes(bs));
        }
    }
}
```

- [ ] **Step 2: 运行确认失败**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dit.test=EventsPendingSchemaIT -Dtest=FlywayMigrationSyncTest verify
```
Expected: 两个都 FAIL —— `EventsPendingSchemaIT` 因 V10 不存在 / 表不存在失败；`FlywayMigrationSyncTest` 因 proactive-core 副本里缺 `V10__create_events_pending.sql`（名称列表与 bootstrap 不一致）失败。

- [ ] **Step 3: 写 migration（最小实现）+ 同步 4 个副本**

主拷贝 `V10__create_events_pending.sql`（**CHECK 用大写 enum**，时间列 `TIMESTAMPTZ`）：

```sql
-- V10__create_events_pending.sql
-- proactive 域调度表：4 触发器写 scheduled，ProactiveScheduler 扫描 due 派发
CREATE TABLE events_pending (
    id            BIGSERIAL   PRIMARY KEY,
    user_id       BIGINT      NOT NULL,
    character_id  BIGINT      NOT NULL,
    event_type    VARCHAR(40) NOT NULL
                  CHECK (event_type IN ('A_GREETING','B_RECALL','C_EVENT_FOLLOWUP','D_EMOTION_CARE')),
    status        VARCHAR(20) NOT NULL DEFAULT 'SCHEDULED'
                  CHECK (status IN ('SCHEDULED','PROCESSING','SENT','FAILED','CANCELLED')),
    payload       JSONB       NOT NULL DEFAULT '{}',
    scheduled_at  TIMESTAMPTZ NOT NULL,
    sent_at       TIMESTAMPTZ,
    fail_count    INT         NOT NULL DEFAULT 0,
    last_error    TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_events_pending_due
    ON events_pending (scheduled_at) WHERE status = 'SCHEDULED';
CREATE INDEX idx_events_pending_dedup
    ON events_pending (user_id, event_type, status);
```

> 注：spec §3.1 示例 CHECK 用小写（`a_greeting` / `scheduled`），但附录约定「枚举存 Java enum `name()`（大写 SNAKE）」，本计划统一**大写**，与 `EventType` / `EventStatus` enum 的 `@Enumerated(STRING)` 落库值一致。`idx_events_pending_due` 的 partial 条件也对应大写 `'SCHEDULED'`。

同步 4 个副本：

```bash
cd server
SRC=bootstrap/src/main/resources/db/migration
for d in sanyan-memory-core sanyan-character-core sanyan-proactive-core sanyan-push-core; do
  cp "$SRC/V10__create_events_pending.sql" business_packages/$d/src/test/resources/db/migration/
done
```

- [ ] **Step 4: 运行确认通过**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dit.test=EventsPendingSchemaIT -Dtest=FlywayMigrationSyncTest verify
```
Expected: BUILD SUCCESS（schema IT 11 列 + 双索引 + 两组大写 CHECK 命中；SyncTest 副本一致）。

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/main/resources/db/migration/V10__create_events_pending.sql \
        server/business_packages/sanyan-memory-core/src/test/resources/db/migration/V10__create_events_pending.sql \
        server/business_packages/sanyan-character-core/src/test/resources/db/migration/V10__create_events_pending.sql \
        server/business_packages/sanyan-proactive-core/src/test/resources/db/migration/V10__create_events_pending.sql \
        server/business_packages/sanyan-push-core/src/test/resources/db/migration/V10__create_events_pending.sql \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/EventsPendingSchemaIT.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/FlywayMigrationSyncTest.java
git commit -m "feat(db): V10 创建 events_pending 调度表（大写 enum CHECK + jsonb payload + due/dedup 索引）+ proactive-core 副本同步守护"
```

---

#### Task B3: V11 migration — device_tokens 表 + push-core schema IT + SyncTest

**Files:**
- Create: `server/bootstrap/src/main/resources/db/migration/V11__create_device_tokens.sql`（主拷贝）
- Create: 逐字节副本到 memory-core / character-core / proactive-core / push-core 各 `src/test/resources/db/migration/V11__create_device_tokens.sql`
- Create: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/DeviceTokensSchemaIT.java`
- Create: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/FlywayMigrationSyncTest.java`（**新建**，包 `com.sanyan.push`）

- [ ] **Step 1: 先写失败的 schema IT + push-core SyncTest**

`DeviceTokensSchemaIT`（`runMigrationsUpTo("11")`）：

```java
package com.sanyan.push.internal;

import com.sanyan.common.test.PostgresTestcontainerSupport;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * V11 migration 验证：device_tokens 表（platform 大写? 否——platform 存 'ios'/'android' 小写，见约定 +
 * UNIQUE(user_id, platform, vendor, token)）。
 */
class DeviceTokensSchemaIT extends PostgresTestcontainerSupport {

    @Test
    void v11_should_create_device_tokens_with_unique_and_platform_check() throws Exception {
        runMigrationsUpTo("11");

        Map<String, ColumnSpec> cols = describeColumns("device_tokens");

        // 8 列存在
        assertThat(cols).containsKeys("id", "user_id", "platform", "vendor",
                "token", "active", "registered_at", "last_seen");

        // 类型
        assertThat(cols.get("id").typeName()).isEqualTo("bigserial");
        assertThat(cols.get("user_id").typeName()).isEqualTo("bigint");
        assertThat(cols.get("platform").typeName()).isEqualTo("varchar");
        assertThat(cols.get("vendor").typeName()).isEqualTo("varchar");
        assertThat(cols.get("token").typeName()).isEqualTo("varchar");
        assertThat(cols.get("active").typeName()).isEqualTo("bool");
        assertThat(cols.get("registered_at").typeName()).isEqualTo("timestamptz");
        assertThat(cols.get("last_seen").typeName()).isEqualTo("timestamptz");

        // nullability —— 全 NOT NULL
        assertThat(cols.get("user_id").nullable()).isFalse();
        assertThat(cols.get("platform").nullable()).isFalse();
        assertThat(cols.get("vendor").nullable()).isFalse();
        assertThat(cols.get("token").nullable()).isFalse();
        assertThat(cols.get("active").nullable()).isFalse();
        assertThat(cols.get("registered_at").nullable()).isFalse();
        assertThat(cols.get("last_seen").nullable()).isFalse();

        // platform CHECK（小写值，与 DeviceTokenEntity.platform 存的 'ios'/'android' 一致）
        String checks = checkConstraintsOf("device_tokens");
        assertThat(checks).contains("ios", "android");

        // UNIQUE(user_id, platform, vendor, token)
        assertThat(uniqueConstraintExists("device_tokens")).isTrue();
    }

    private String checkConstraintsOf(String table) throws Exception {
        StringBuilder sb = new StringBuilder();
        try (Connection conn = newConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "SELECT pg_get_constraintdef(c.oid) AS def "
                             + "FROM pg_constraint c JOIN pg_class t ON c.conrelid = t.oid "
                             + "WHERE t.relname = ? AND c.contype = 'c'")) {
            ps.setString(1, table);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    sb.append(rs.getString("def")).append('\n');
                }
            }
        }
        return sb.toString();
    }

    private boolean uniqueConstraintExists(String table) throws Exception {
        try (Connection conn = newConnection();
             PreparedStatement ps = conn.prepareStatement(
                     "SELECT COUNT(*) FROM pg_constraint c JOIN pg_class t ON c.conrelid = t.oid "
                             + "WHERE t.relname = ? AND c.contype = 'u'")) {
            ps.setString(1, table);
            try (ResultSet rs = ps.executeQuery()) {
                rs.next();
                return rs.getInt(1) >= 1;
            }
        }
    }
}
```

`FlywayMigrationSyncTest`（push-core 版，包 `com.sanyan.push`，常量名 `PUSH_TEST_DIR`，与 B2 模板对等，仅替换标识）：

```java
package com.sanyan.push;

import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Plan 4 B3 引入的 Flyway 副本同步守护测试（push-core 版）。防止 bootstrap 主拷贝
 * 与本模块 {@code src/test/resources/db/migration/} 副本漂移。
 */
class FlywayMigrationSyncTest {

    private static final Path BOOTSTRAP_DIR = Path.of(
            "../../bootstrap/src/main/resources/db/migration");
    private static final Path PUSH_TEST_DIR = Path.of(
            "src/test/resources/db/migration");

    @Test
    void flywayMigrationsMustBeInSyncBetweenBootstrapAndPushCoreTest() throws IOException {
        if (!Files.isDirectory(BOOTSTRAP_DIR)) {
            return;
        }
        List<Path> bootstrapSqls;
        List<Path> pushTestSqls;
        try (Stream<Path> bs = Files.list(BOOTSTRAP_DIR);
             Stream<Path> pt = Files.list(PUSH_TEST_DIR)) {
            bootstrapSqls = bs.filter(p -> p.getFileName().toString().endsWith(".sql"))
                    .sorted().toList();
            pushTestSqls = pt.filter(p -> p.getFileName().toString().endsWith(".sql"))
                    .sorted().toList();
        }

        List<String> bootstrapNames = bootstrapSqls.stream()
                .map(p -> p.getFileName().toString()).sorted().toList();
        List<String> pushNames = pushTestSqls.stream()
                .map(p -> p.getFileName().toString()).sorted().toList();
        assertThat(pushNames)
                .as("push-core/src/test/resources/db/migration 必须与 bootstrap 主拷贝同名同数")
                .containsExactlyElementsOf(bootstrapNames);

        for (int i = 0; i < bootstrapSqls.size(); i++) {
            Path bs = bootstrapSqls.get(i);
            Path pt = pushTestSqls.get(i);
            assertThat(Files.readAllBytes(pt))
                    .as("文件 %s 在 bootstrap 与 push-core test 拷贝不一致——需同步",
                            bs.getFileName())
                    .containsExactly(Files.readAllBytes(bs));
        }
    }
}
```

- [ ] **Step 2: 运行确认失败**

```bash
cd server && mvn -pl business_packages/sanyan-push-core -Dit.test=DeviceTokensSchemaIT -Dtest=FlywayMigrationSyncTest verify
```
Expected: 两个都 FAIL —— `DeviceTokensSchemaIT` 因 V11 不存在 / 表不存在失败；`FlywayMigrationSyncTest` 因 push-core 副本缺 `V11__create_device_tokens.sql` 失败。

- [ ] **Step 3: 写 migration（最小实现）+ 同步 4 个副本**

主拷贝 `V11__create_device_tokens.sql`（platform CHECK 用小写 `'ios'/'android'`——这是设备平台标识不是 Java enum，附录 D `DeviceTokenEntity.platform` 字段注释即 `ios / android`；时间列 `TIMESTAMPTZ`）：

```sql
-- V11__create_device_tokens.sql
-- push 域：客户端上报的设备推送 token（本期只搭结构，实推待证书 / SDK）
CREATE TABLE device_tokens (
    id            BIGSERIAL    PRIMARY KEY,
    user_id       BIGINT       NOT NULL,
    platform      VARCHAR(20)  NOT NULL CHECK (platform IN ('ios','android')),
    vendor        VARCHAR(20)  NOT NULL,
    token         VARCHAR(500) NOT NULL,
    active        BOOLEAN      NOT NULL DEFAULT TRUE,
    registered_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    last_seen     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, platform, vendor, token)
);
```

同步 4 个副本：

```bash
cd server
SRC=bootstrap/src/main/resources/db/migration
for d in sanyan-memory-core sanyan-character-core sanyan-proactive-core sanyan-push-core; do
  cp "$SRC/V11__create_device_tokens.sql" business_packages/$d/src/test/resources/db/migration/
done
```

- [ ] **Step 4: 运行确认通过**

```bash
cd server && mvn -pl business_packages/sanyan-push-core -Dit.test=DeviceTokensSchemaIT -Dtest=FlywayMigrationSyncTest verify
```
Expected: BUILD SUCCESS（8 列 + platform CHECK + UNIQUE 约束命中；SyncTest 副本一致）。

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/main/resources/db/migration/V11__create_device_tokens.sql \
        server/business_packages/sanyan-memory-core/src/test/resources/db/migration/V11__create_device_tokens.sql \
        server/business_packages/sanyan-character-core/src/test/resources/db/migration/V11__create_device_tokens.sql \
        server/business_packages/sanyan-proactive-core/src/test/resources/db/migration/V11__create_device_tokens.sql \
        server/business_packages/sanyan-push-core/src/test/resources/db/migration/V11__create_device_tokens.sql \
        server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/DeviceTokensSchemaIT.java \
        server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/FlywayMigrationSyncTest.java
git commit -m "feat(db): V11 创建 device_tokens 表（platform CHECK + 四列 UNIQUE）+ push-core 副本同步守护"
```

---

#### Task B4: 验证 memory-core / character-core 既有 FlywayMigrationSyncTest 在补齐 V9-V11 后仍通过

> 既有两个 SyncTest（`com.sanyan.memory.FlywayMigrationSyncTest` / `com.sanyan.character.FlywayMigrationSyncTest`）逐字节比对 bootstrap 与各自副本。B1-B3 每个 task 都同步过它们的副本，本 task 是**回归确认**——没有新代码，纯跑测试守住前 3 个 task 的副本同步没漏。

**Files:** 无新增（仅运行既有测试）。

- [ ] **Step 1: （回归确认，无新测试——既有 SyncTest 即验证目标）**

- [ ] **Step 2: 运行既有 SyncTest 确认通过**

```bash
cd server && mvn -pl business_packages/sanyan-memory-core,business_packages/sanyan-character-core -Dtest=FlywayMigrationSyncTest test
```
Expected: BUILD SUCCESS —— 两模块副本目录都含 V1-V11 且与 bootstrap 逐字节一致。
若 FAIL：说明 B1-B3 某次 `cp` 漏了某个模块副本，回到对应 task 补 `cp` + 重新 commit（amend 到对应 task commit 或追加修订 commit）。

- [ ] **Step 3: （无实现）**
- [ ] **Step 4: （Step 2 已确认通过）**

- [ ] **Step 5:（Phase B checkpoint）跑一次 4 个新副本模块 + 既有副本模块的 migration 相关 IT**

```bash
cd server && mvn -pl business_packages/sanyan-memory-core,business_packages/sanyan-character-core,business_packages/sanyan-proactive-core,business_packages/sanyan-push-core verify
```
Expected: BUILD SUCCESS。无代码改动则无需 commit；本 task 是 Phase B 的收尾门禁。

---

### Phase C · KvCache.increment（基础层，测试粒度 = 全量）

> 缺口 #4：`KvCache` 无原子 INCR，proactive 每日上限计数（`proactive:sent:{userId}:{yyyy-MM-dd}`，附录 G）需要。基础层改动 → **跑 server 全量**。
> 单测照现状 `KvCacheTest`（Mockito mock `StringRedisTemplate` + `ValueOperations`）。

#### Task C1: KvCache 加 increment(key, ttlIfNew)

**Files:**
- Modify: `server/foundation_packages/sanyan-common-cache/src/main/java/com/sanyan/common/cache/KvCache.java`
- Modify: `server/foundation_packages/sanyan-common-cache/src/test/java/com/sanyan/common/cache/KvCacheTest.java`

- [ ] **Step 1: 先写失败测试**

在 `KvCacheTest` 末尾追加（mock `ops.increment(key)` + `redis.expire(...)`）：

```java
    @Test
    void increment_setsExpiryOnFirstHit() {
        when(redis.opsForValue()).thenReturn(ops);
        when(ops.increment("cnt")).thenReturn(1L);   // 返回 1 ⇒ 新建
        KvCache cache = new KvCache(redis);

        long v = cache.increment("cnt", Duration.ofHours(36));

        assertThat(v).isEqualTo(1L);
        verify(ops).increment("cnt");
        verify(redis).expire("cnt", Duration.ofHours(36));   // 仅新建时设 TTL
    }

    @Test
    void increment_doesNotResetExpiryWhenKeyAlreadyExists() {
        when(redis.opsForValue()).thenReturn(ops);
        when(ops.increment("cnt")).thenReturn(5L);   // 返回 >1 ⇒ 已存在
        KvCache cache = new KvCache(redis);

        long v = cache.increment("cnt", Duration.ofHours(36));

        assertThat(v).isEqualTo(5L);
        verify(ops).increment("cnt");
        verify(redis, never()).expire(anyString(), any(Duration.class));   // 不重设 TTL
    }

    @Test
    void increment_treatsNullReturnAsZero() {
        when(redis.opsForValue()).thenReturn(ops);
        when(ops.increment("cnt")).thenReturn(null);   // 连接异常等
        KvCache cache = new KvCache(redis);

        long v = cache.increment("cnt", Duration.ofHours(36));

        assertThat(v).isZero();
        verify(redis, never()).expire(anyString(), any(Duration.class));
    }

    @Test
    void increment_throwsWhenTtlIsZero() {
        KvCache cache = new KvCache(redis);
        assertThatThrownBy(() -> cache.increment("k", Duration.ZERO))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("ttl");
    }

    @Test
    void increment_throwsWhenTtlIsNull() {
        KvCache cache = new KvCache(redis);
        assertThatThrownBy(() -> cache.increment("k", null))
            .isInstanceOf(NullPointerException.class);
    }
```

补 import（`KvCacheTest` 现有 `import static org.mockito.Mockito.*;` 已覆盖 `never()` / `anyString()` / `any()`；`Duration` 已 import）。

- [ ] **Step 2: 运行确认失败**

```bash
cd server && mvn -pl foundation_packages/sanyan-common-cache -Dtest=KvCacheTest test
```
Expected: FAIL —— 编译错误 `cannot find symbol: method increment(String,Duration)`。

- [ ] **Step 3: 写最小实现**

在 `KvCache` 加方法（key 不存在 → `INCR` 得 1 → `expire` 设 TTL；已存在只 `INCR`；null 当 0；TTL 非正抛异常）：

```java
    /**
     * 原子自增；key 不存在时（自增结果为 1）顺带设置 TTL，已存在则只自增不重设 TTL。
     *
     * <p>用于「滑动窗口外的固定窗口计数」场景，典型用例：
     * Plan 4 proactive 每日已发主动消息数 {@code proactive:sent:{userId}:{yyyy-MM-dd}}，
     * 当天首次发送时计 1 并设 36h TTL，之后只累加，跨天后 key 自然过期重置。
     *
     * <p>Redis 返回 {@code null}（连接异常等）视为 0，调用方据此判断本次计数失败。
     *
     * @param key      计数 key
     * @param ttlIfNew key 首次创建时设置的过期时间（必须为正）
     * @return 自增后的值；Redis 异常返回 0
     */
    public long increment(String key, Duration ttlIfNew) {
        Objects.requireNonNull(ttlIfNew, "ttl must not be null");
        if (ttlIfNew.isZero() || ttlIfNew.isNegative()) {
            throw new IllegalArgumentException("ttl must be positive");
        }
        Long value = redis.opsForValue().increment(key);
        if (value == null) {
            return 0L;
        }
        if (value == 1L) {
            redis.expire(key, ttlIfNew);
        }
        return value;
    }
```

- [ ] **Step 4: 运行确认通过**

```bash
cd server && mvn -pl foundation_packages/sanyan-common-cache -Dtest=KvCacheTest test
```
Expected: BUILD SUCCESS（5 个新用例 + 原有用例全绿）。

- [ ] **Step 5: Commit**

```bash
git add server/foundation_packages/sanyan-common-cache/src/main/java/com/sanyan/common/cache/KvCache.java \
        server/foundation_packages/sanyan-common-cache/src/test/java/com/sanyan/common/cache/KvCacheTest.java
git commit -m "feat(common-cache): KvCache 加 increment 原子自增（首次创建设 TTL）"
```

> **Phase C checkpoint（基础层改动，跑 server 全量）：** `cd server && mvn -q test`（surefire 单测全量；基础层被广泛依赖，确认无下游编译/行为回归）。

---

### Phase D · last_active（LastActiveTracker + 3 处接入，依赖 C）

> spec §3.2：失联召回依赖「用户最后活跃时间」，新增 Redis key `user:last_active:{userId}`（附录 G，TTL 30d）。
> 新建 `LastActiveTracker`（放 **common-ws**，与 SessionManager 同层，跨域可复用——common-ws 已依赖 common-cache，可直接注入 `KvCache`）。
> 接入点 3 处（任一活跃信号即刷新）：WS `afterConnectionEstablished`、收到用户消息、`GET /api/relationships/me`。

#### Task D1: 新建 LastActiveTracker（common-ws，注入 KvCache）

**Files:**
- Create: `server/foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/LastActiveTracker.java`
- Test: `server/foundation_packages/sanyan-common-ws/src/test/java/com/sanyan/common/ws/LastActiveTrackerTest.java`

- [ ] **Step 1: 先写失败测试（Mockito mock KvCache）**

```java
package com.sanyan.common.ws;

import com.sanyan.common.cache.KvCache;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Duration;
import java.time.Instant;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class LastActiveTrackerTest {

    @Mock KvCache kvCache;

    @Test
    void touch_setsLastActiveWithUserKeyAnd30dTtl() {
        LastActiveTracker tracker = new LastActiveTracker(kvCache);

        tracker.touch(42L);

        ArgumentCaptor<String> valueCap = ArgumentCaptor.forClass(String.class);
        verify(kvCache).set(org.mockito.ArgumentMatchers.eq("user:last_active:42"),
                valueCap.capture(),
                org.mockito.ArgumentMatchers.eq(Duration.ofDays(30)));
        // 写入值是可解析的 ISO-8601 Instant
        assertThat(Instant.parse(valueCap.getValue())).isNotNull();
    }

    @Test
    void lastActiveAt_parsesStoredTimestamp() {
        Instant now = Instant.parse("2026-05-27T09:00:00Z");
        when(kvCache.get("user:last_active:42")).thenReturn(now.toString());
        LastActiveTracker tracker = new LastActiveTracker(kvCache);

        Optional<Instant> result = tracker.lastActiveAt(42L);

        assertThat(result).contains(now);
    }

    @Test
    void lastActiveAt_returnsEmptyWhenKeyMissing() {
        when(kvCache.get("user:last_active:42")).thenReturn(null);
        LastActiveTracker tracker = new LastActiveTracker(kvCache);

        assertThat(tracker.lastActiveAt(42L)).isEmpty();
    }
}
```

- [ ] **Step 2: 运行确认失败**

```bash
cd server && mvn -pl foundation_packages/sanyan-common-ws -Dtest=LastActiveTrackerTest test
```
Expected: FAIL —— 编译错误 `cannot find symbol: class LastActiveTracker`。

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.common.ws;

import com.sanyan.common.cache.KvCache;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.time.Instant;
import java.util.Optional;

/**
 * 用户最后活跃时间追踪。失联召回（Plan 4 b 场景）据此判断 24h/3d/7d 阶梯。
 *
 * <p>Redis key {@code user:last_active:{userId}} → ISO-8601 时间戳字符串，TTL 30 天。
 * 走 {@link KvCache} 封装（禁止业务直接碰 RedisTemplate）。粗粒度判断，Redis 丢失
 * 最多漏一次召回，可接受。
 */
@Component
@RequiredArgsConstructor
public class LastActiveTracker {

    private static final String KEY_PREFIX = "user:last_active:";
    private static final Duration TTL = Duration.ofDays(30);

    private final KvCache kvCache;

    /** 记录某用户当前为活跃（覆盖写当前时间）。 */
    public void touch(Long userId) {
        kvCache.set(KEY_PREFIX + userId, Instant.now().toString(), TTL);
    }

    /** 读取最后活跃时间；无记录返回 empty。 */
    public Optional<Instant> lastActiveAt(Long userId) {
        String raw = kvCache.get(KEY_PREFIX + userId);
        if (raw == null || raw.isBlank()) {
            return Optional.empty();
        }
        return Optional.of(Instant.parse(raw));
    }
}
```

- [ ] **Step 4: 运行确认通过**

```bash
cd server && mvn -pl foundation_packages/sanyan-common-ws -Dtest=LastActiveTrackerTest test
```
Expected: BUILD SUCCESS。

- [ ] **Step 5: Commit**

```bash
git add server/foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/LastActiveTracker.java \
        server/foundation_packages/sanyan-common-ws/src/test/java/com/sanyan/common/ws/LastActiveTrackerTest.java
git commit -m "feat(common-ws): 新建 LastActiveTracker（user:last_active key，TTL 30d）"
```

---

#### Task D2: 3 处接入 LastActiveTracker.touch

> 三个更新点：① `ChatWebSocketHandler.afterConnectionEstablished`（WS 连接）② `ChatWebSocketHandler.handleSendMessage`（收到用户消息）③ `RelationshipController` GET /me（进聊天页）。
> ①② 在 chat-core（已依赖 common-ws，可注入 `LastActiveTracker`）；③ 在 character-core。**character-core 需确认是否已依赖 common-ws**——若未依赖则本 task 先在 `sanyan-character-core/pom.xml` 加 `<dependency>sanyan-common-ws</dependency>`（实现者核对，见末尾假设）。

**Files:**
- Modify: `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java`
- Test: `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/ws/ChatWebSocketHandlerLastActiveTest.java`（新建）
- Modify: `server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/web/RelationshipController.java`
- Modify（按需）: `server/business_packages/sanyan-character-core/pom.xml`（若缺 common-ws 依赖）
- Test: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/web/RelationshipControllerIT.java`（已存在则在内补断言；不存在则新建 `@WebMvcTest`）

- [ ] **Step 1: 先写失败测试**

chat-core 侧——新建 `ChatWebSocketHandlerLastActiveTest`，验证连接建立和收到消息都 `touch`：

```java
package com.sanyan.chat.ws;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.chat.internal.MessageEntity;
import com.sanyan.chat.internal.MessageService;
import com.sanyan.common.ws.LastActiveTracker;
import com.sanyan.common.ws.SessionManager;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;

import java.util.HashMap;
import java.util.Map;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.lenient;
import static org.mockito.Mockito.timeout;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ChatWebSocketHandlerLastActiveTest {

    @Mock SessionManager sessionManager;
    @Mock MessageService messageService;
    @Mock LastActiveTracker lastActiveTracker;
    @Mock WebSocketSession session;

    ObjectMapper objectMapper = new ObjectMapper();
    ChatWebSocketHandler handler;

    @BeforeEach
    void setUp() {
        Map<String, Object> attrs = new HashMap<>();
        attrs.put("userId", 7L);
        lenient().when(session.getAttributes()).thenReturn(attrs);
        lenient().when(session.isOpen()).thenReturn(true);
        handler = new ChatWebSocketHandler(sessionManager, objectMapper, messageService, lastActiveTracker);
    }

    @Test
    void afterConnectionEstablished_touchesLastActive() {
        handler.afterConnectionEstablished(session);
        verify(lastActiveTracker).touch(7L);
    }

    @Test
    void handleSendMessage_touchesLastActive() throws Exception {
        MessageEntity userMsg = new MessageEntity();
        userMsg.setId(100L);
        when(messageService.saveUserMessage(eqL(7L), any())).thenReturn(userMsg);
        lenient().when(messageService.handleAiReply(7L)).thenReturn(java.util.List.of());

        handler.handleTextMessage(session,
                new TextMessage("{\"type\":\"send_message\",\"content\":\"hi\",\"clientMsgId\":\"c1\"}"));

        // touch 在收到用户消息时同步发生（不依赖异步 AI 回复）
        verify(lastActiveTracker, timeout(1000)).touch(7L);
    }

    private static Long eqL(Long v) { return org.mockito.ArgumentMatchers.eq(v); }
}
```

> 实现者注：`MessageEntity` / `MessageService.saveUserMessage` 的真实签名以现状为准（`saveUserMessage(Long userId, String content)` 返回 `MessageEntity`，已确认）。若构造 `MessageEntity` 需要更多字段，用 chat-core 既有的 `MessageTestFixtures`（若存在）替代 `new MessageEntity()`。

character-core 侧——在既有 `RelationshipControllerIT`（`@WebMvcTest`，若不存在则新建）补一条断言 GET /me 触发 `lastActiveTracker.touch(userId)`：

```java
    @Test
    void getMe_touchesLastActive() throws Exception {
        // given：mock characterApi.fetchMyRelationship 返回任意 RelationshipDto
        // when：GET /api/relationships/me（带登录用户 99）
        // then：verify(lastActiveTracker).touch(99L)
        // 注：@WebMvcTest 里 LastActiveTracker 用 @MockBean 注入；登录用户注入沿用现有 RelationshipControllerIT 的 @LoginUser mock 方式
    }
```

- [ ] **Step 2: 运行确认失败**

```bash
cd server && mvn -pl business_packages/sanyan-chat-core -Dtest=ChatWebSocketHandlerLastActiveTest test
cd server && mvn -pl business_packages/sanyan-character-core -Dit.test=RelationshipControllerIT verify
```
Expected: 均 FAIL —— chat 侧因构造器参数数量不符 / `touch` 未被调用；character 侧因 `LastActiveTracker` 未注入 / `touch` 未被调用。

- [ ] **Step 3: 写最小实现**

chat-core `ChatWebSocketHandler` 加注入 + 两处 `touch`：

```java
// 字段（构造器注入，@RequiredArgsConstructor 自动加入参数）
private final LastActiveTracker lastActiveTracker;
```

`afterConnectionEstablished` 加一行：

```java
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        Long userId = (Long) session.getAttributes().get("userId");
        sessionManager.register(userId, session);
        lastActiveTracker.touch(userId);   // 刷新最后活跃时间（失联召回依赖）
        log.info("用户 {} WebSocket 已连接", userId);
    }
```

`handleSendMessage` 开头（落库前，确保收到即刷新）加：

```java
    private void handleSendMessage(Long userId, WsMessage wsMsg, WebSocketSession session) {
        lastActiveTracker.touch(userId);   // 收到用户消息即刷新最后活跃时间
        String preview = ...   // 原有逻辑不变
```

character-core `RelationshipController` 加注入 + GET /me 里 `touch`：

```java
@RestController
@RequestMapping("/api/relationships")
@RequiredArgsConstructor
public class RelationshipController {

    private static final Long DEFAULT_CHARACTER_ID = 1L;

    private final CharacterApi characterApi;
    private final LastActiveTracker lastActiveTracker;

    @GetMapping("/me")
    public BaseResp<RelationshipDto> fetchMyRelationship(@LoginUser Long userId) {
        lastActiveTracker.touch(userId);   // 进聊天页即刷新最后活跃时间
        return BaseResp.success(characterApi.fetchMyRelationship(userId, DEFAULT_CHARACTER_ID));
    }
}
```

（import `com.sanyan.common.ws.LastActiveTracker`；若 character-core 此前未依赖 common-ws，在其 pom 加 `<dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-ws</artifactId></dependency>`。）

- [ ] **Step 4: 运行确认通过**

```bash
cd server && mvn -pl business_packages/sanyan-chat-core -Dtest=ChatWebSocketHandlerLastActiveTest test
cd server && mvn -pl business_packages/sanyan-character-core -Dit.test=RelationshipControllerIT verify
```
Expected: 均 BUILD SUCCESS。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java \
        server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/ws/ChatWebSocketHandlerLastActiveTest.java \
        server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/web/RelationshipController.java \
        server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/web/RelationshipControllerIT.java \
        server/business_packages/sanyan-character-core/pom.xml
git commit -m "feat(last-active): WS 连接/收消息/GET me 三处接入 LastActiveTracker.touch"
```

> **Phase D checkpoint：** 跑 common-ws + chat-core + character-core 三模块测试 `cd server && mvn -pl foundation_packages/sanyan-common-ws,business_packages/sanyan-chat-core,business_packages/sanyan-character-core verify`。

---

### Phase E · common-ws 心跳 TTL（基础层，测试粒度 = 全量）

> 缺口 #6 + spec §7.3：消除现存「假在线」半成品——`register` 写 `ws:online` 不带 TTL、`isOnline` 与 `getSession` 口径不一致、心跳无续期。
> 改造：`register` 写带 `ONLINE_TTL=Duration.ofSeconds(90)`（附录 E + G）；新增 `refreshOnline(Long)` 刷新 TTL；`ChatWebSocketHandler` 收到 PING 时调 `refreshOnline`（保留回 PONG）。基础层改动 → **跑 server 全量**。

#### Task E1: SessionManager.register 带 TTL + 新增 refreshOnline

**Files:**
- Modify: `server/foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/SessionManager.java`
- Modify: `server/foundation_packages/sanyan-common-ws/src/test/java/com/sanyan/common/ws/SessionManagerTest.java`

- [ ] **Step 1: 先写失败测试**

改 `SessionManagerTest`：`register` 断言带 TTL；新增 `refreshOnline` 用例。把现有 `shouldRegisterAndRetrieveSession` 的 `verify(valueOps).set("ws:online:1", "1")` 改为带 TTL 版本，并追加 refresh 用例：

```java
    @Test
    void shouldRegisterAndRetrieveSession() {
        when(wsSession.isOpen()).thenReturn(true);
        sessionManager.register(1L, wsSession);

        assertThat(sessionManager.getSession(1L)).isPresent();
        verify(valueOps).set("ws:online:1", "1", java.time.Duration.ofSeconds(90));
    }

    @Test
    void refreshOnline_resetsTtlOnExistingKey() {
        sessionManager.refreshOnline(7L);
        // 续期：直接 set 同 key + value + 90s TTL（覆盖写即续期）
        verify(valueOps).set("ws:online:7", "1", java.time.Duration.ofSeconds(90));
    }
```

> `shouldCheckOnlineStatus` 不变（`isOnline` 仍查 key 是否为 "1"，TTL 过期后 Redis 自动返 null ⇒ false，与心跳口径一致，无需改）。

- [ ] **Step 2: 运行确认失败**

```bash
cd server && mvn -pl foundation_packages/sanyan-common-ws -Dtest=SessionManagerTest test
```
Expected: FAIL —— `shouldRegisterAndRetrieveSession` 因实际 `set` 调用是 2 参（无 TTL）与期望 3 参不符；`refreshOnline` 编译错误（方法不存在）。

- [ ] **Step 3: 写最小实现**

`SessionManager` 加 `ONLINE_TTL` 常量、改 `register`、加 `refreshOnline`：

```java
    private static final String ONLINE_PREFIX = "ws:online:";
    private static final String ONLINE_VALUE = "1";
    private static final Duration ONLINE_TTL = Duration.ofSeconds(90);

    public void register(Long userId, WebSocketSession session) {
        sessions.put(userId, session);
        redisTemplate.opsForValue().set(ONLINE_PREFIX + userId, ONLINE_VALUE, ONLINE_TTL);
    }

    /** 心跳续期：客户端 PING 时刷新在线 TTL（覆盖写同 key 即续期）。 */
    public void refreshOnline(Long userId) {
        redisTemplate.opsForValue().set(ONLINE_PREFIX + userId, ONLINE_VALUE, ONLINE_TTL);
    }
```

（顶部加 `import java.time.Duration;`；`isOnline` 保持不变——仍 `"1".equals(get(...))`，TTL 过期自动 false。）

- [ ] **Step 4: 运行确认通过**

```bash
cd server && mvn -pl foundation_packages/sanyan-common-ws -Dtest=SessionManagerTest test
```
Expected: BUILD SUCCESS。

- [ ] **Step 5: Commit**

```bash
git add server/foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/SessionManager.java \
        server/foundation_packages/sanyan-common-ws/src/test/java/com/sanyan/common/ws/SessionManagerTest.java
git commit -m "feat(common-ws): ws:online 写入带 90s TTL + 新增 refreshOnline 心跳续期"
```

---

#### Task E2: ChatWebSocketHandler PING 分支调 refreshOnline

> spec §7.3：客户端已有定时 PING；handler 收到 PING 时刷新 TTL（保留回 PONG），无需服务端主动 ping。

**Files:**
- Modify: `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java`
- Modify: `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/ws/ChatWebSocketHandlerLastActiveTest.java`（同文件加 PING 用例，复用 D2 的 setUp）

- [ ] **Step 1: 先写失败测试**

在 D2 的 `ChatWebSocketHandlerLastActiveTest` 追加（验证收到 PING 既续期又回 PONG）：

```java
    @Test
    void handlePing_refreshesOnlineAndRepliesPong() throws Exception {
        handler.handleTextMessage(session, new TextMessage("{\"type\":\"ping\"}"));

        verify(sessionManager).refreshOnline(7L);
        // 仍回 PONG：session.sendMessage 至少被调用一次且 payload 含 "pong"
        org.mockito.ArgumentCaptor<org.springframework.web.socket.TextMessage> cap =
                org.mockito.ArgumentCaptor.forClass(org.springframework.web.socket.TextMessage.class);
        verify(session).sendMessage(cap.capture());
        org.assertj.core.api.Assertions.assertThat(cap.getValue().getPayload()).contains("pong");
    }
```

- [ ] **Step 2: 运行确认失败**

```bash
cd server && mvn -pl business_packages/sanyan-chat-core -Dtest=ChatWebSocketHandlerLastActiveTest test
```
Expected: FAIL —— `verify(sessionManager).refreshOnline(7L)` 未被调用（当前 PING 分支只回 PONG）。

- [ ] **Step 3: 写最小实现**

`ChatWebSocketHandler.handleTextMessage` 的 PING case 加 `refreshOnline`（保留回 PONG）：

```java
        switch (wsMsg.getType()) {
            case WsEventType.PING -> {
                sessionManager.refreshOnline(userId);   // 心跳续期在线 TTL
                sendToSession(session, "{\"type\":\"" + WsEventType.PONG + "\"}");
            }
            case WsEventType.SEND_MESSAGE -> handleSendMessage(userId, wsMsg, session);
            case WsEventType.SYNC -> handleSync(userId, wsMsg, session);
            default -> log.warn("未知消息类型: {}", wsMsg.getType());
        }
```

- [ ] **Step 4: 运行确认通过**

```bash
cd server && mvn -pl business_packages/sanyan-chat-core -Dtest=ChatWebSocketHandlerLastActiveTest test
```
Expected: BUILD SUCCESS（PING 用例 + D2 既有用例全绿）。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java \
        server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/ws/ChatWebSocketHandlerLastActiveTest.java
git commit -m "feat(chat-core): WS PING 分支接入 refreshOnline 心跳续期（保留回 PONG）"
```

> **Phase E checkpoint（基础层改动，跑 server 全量）：** `cd server && mvn -q verify`（surefire 单测 + failsafe IT 全量，确认 common-ws TTL 改动无下游回归——尤其 chat-core 在线投递相关测试）。
### Phase F · memory 域结构化记忆 + 实时抽取（依赖 B,C）

> **域归属：** 全部落在已有的 `sanyan-memory-core` / `sanyan-memory-api`，无新建模块。
> **pom 现状（无需改）：** memory-core 已依赖 `sanyan-chat-api`（MessagePersistedEvent / SenderType / MessageDto）、`sanyan-llm-api`（LlmApi / ChatMessage / LlmTaskType）、`sanyan-common-cache`、`sanyan-common-error`、`jackson-databind`、`spring-boot-starter-data-jpa`、`sanyan-common-test`(test)。本 Phase 不新增任何依赖。
> **前置：** Phase B 已建 V9 `memory_item` migration（bootstrap 主拷贝 + memory-core `src/test/resources/db/migration/V9__*.sql` 副本 + schema IT）。F1 的仓储 IT 复用该副本。
> **错误码现状：** `MemoryErrCode` 已占 5001（PROFILE_REFRESH_CONFLICT）、5002（EMBEDDING_SERVICE_UNAVAILABLE）→ F3 新增码取 **5003**。
> **JSON 工具现状：** 项目**无 JsonUtil**；既有代码（`RagIndexWorker` / `MessageEmbeddingIndexListener`）一律用 Jackson `ObjectMapper`（`private static final ObjectMapper MAPPER = new ObjectMapper();`）。F4 沿用此模式。
> **枚举存库约定（见附录）：** `@Enumerated(EnumType.STRING)` 存大写 enum `name()`；CHECK 约束用大写值。
> **Listener 模板：** `UserMessageProfileRefreshListener`（@Async + @TransactionalEventListener AFTER_COMMIT + SenderType.USER 过滤 + try/catch 吞异常 log.error）。
> **LLM 调用模板：** `MemorySummaryService`（拼 `List<ChatMessage>` → `llmApi.chat(BACKGROUND, ...)`，对话历史塞进单条 user message content，不展开 turns）。

---

#### Task F1: MemoryItemKind / MemoryItemStatus enum + MemoryItemEntity + MemoryItemRepository + MemoryItemTestFixtures

**Files:**
- Create: `server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/item/MemoryItemKind.java`
- Create: `server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/item/MemoryItemStatus.java`
- Create: `server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/item/MemoryItemEntity.java`
- Create: `server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/item/MemoryItemRepository.java`
- Create: `server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/internal/item/fixtures/MemoryItemTestFixtures.java`
- Test: `server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/internal/item/MemoryItemRepositoryIT.java`

> 注：F1 依赖 Phase B 已落地的 V9 `memory_item` migration 测试副本（`sanyan-memory-core/src/test/resources/db/migration/V9__create_memory_item.sql`）。`memory_item` 无外键（spec §3.1），IT 无需 `insertTestUser`，直接存取。

- [ ] **Step 1: 写失败测试（仓储 IT，照 RelationshipRepositoryIT 模板）**

```java
package com.sanyan.memory.internal.item;

import com.sanyan.memory.internal.item.fixtures.MemoryItemTestFixtures;
import com.sanyan.common.test.PostgresTestcontainerSupport;
import com.sanyan.common.test.TestApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase.Replace.NONE;

/**
 * F1：MemoryItemEntity + Repository roundtrip + findByUserIdAndCharacterIdAndStatus 验证。
 *
 * <p>Testcontainers PG（真实 PG enum/jsonb 行为），schema 由 Flyway V1-V9 完整 migration 生成，
 * ddl-auto=none，与生产链路对齐。memory_item 无外键，直接存取。
 */
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@ContextConfiguration(classes = TestApplication.class)
@EntityScan(basePackages = "com.sanyan.memory")
@EnableJpaRepositories(basePackages = "com.sanyan.memory")
class MemoryItemRepositoryIT extends PostgresTestcontainerSupport {

    @DynamicPropertySource
    static void pgProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", PostgresTestcontainerSupport::jdbcUrl);
        registry.add("spring.datasource.username", PostgresTestcontainerSupport::username);
        registry.add("spring.datasource.password", PostgresTestcontainerSupport::password);
        registry.add("spring.datasource.driver-class-name", () -> "org.postgresql.Driver");
        registry.add("spring.flyway.locations", () -> "classpath:db/migration");
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "none");
    }

    @Autowired
    MemoryItemRepository repo;

    @Autowired
    TestEntityManager em;

    @Test
    void save_and_reload_should_roundtrip_enum_and_defaults() {
        MemoryItemEntity item = MemoryItemTestFixtures.planEvent(
                7L, 1L, "周三下午有面试", Instant.parse("2026-06-03T09:00:00Z"));
        Long id = repo.save(item).getId();
        em.flush();
        em.clear();

        MemoryItemEntity loaded = repo.findById(id).orElseThrow();
        assertThat(loaded.getUserId()).isEqualTo(7L);
        assertThat(loaded.getCharacterId()).isEqualTo(1L);
        assertThat(loaded.getKind()).isEqualTo(MemoryItemKind.PLAN_EVENT);
        assertThat(loaded.getContent()).isEqualTo("周三下午有面试");
        assertThat(loaded.getStatus()).isEqualTo(MemoryItemStatus.PENDING);
        assertThat(loaded.getSalientAt()).isEqualTo(Instant.parse("2026-06-03T09:00:00Z"));
        assertThat(loaded.getCreatedAt()).isNotNull();
    }

    @Test
    void findByUserIdAndCharacterIdAndStatus_should_filter_by_status() {
        repo.save(MemoryItemTestFixtures.planEvent(8L, 1L, "周五体检", Instant.now()));
        repo.save(MemoryItemTestFixtures.emotion(8L, 1L, "最近压力大", Instant.now()));
        MemoryItemEntity done = MemoryItemTestFixtures.planEvent(8L, 1L, "已追问过的事", Instant.now());
        done.setStatus(MemoryItemStatus.DONE);
        repo.save(done);
        em.flush();
        em.clear();

        List<MemoryItemEntity> pending =
                repo.findByUserIdAndCharacterIdAndStatus(8L, 1L, MemoryItemStatus.PENDING);

        assertThat(pending).hasSize(2);
        assertThat(pending).extracting(MemoryItemEntity::getStatus)
                .containsOnly(MemoryItemStatus.PENDING);
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryItemRepositoryIT test`
Expected: 编译失败 / 测试失败（`MemoryItemEntity` / `MemoryItemRepository` / `MemoryItemKind` / `MemoryItemStatus` / `MemoryItemTestFixtures` 尚不存在）。

- [ ] **Step 3: 写最小实现**

```java
// MemoryItemKind.java
package com.sanyan.memory.internal.item;

/**
 * 结构化记忆条目分类（spec §4.1）。存库为大写 enum name（@Enumerated STRING）。
 *
 * <ul>
 *   <li>{@link #PLAN_EVENT}：有明确时间的事（"周三有面试"）—— 排事件追问（c）</li>
 *   <li>{@link #EMOTION}：情绪状态（"最近压力大"）—— 排情绪关怀（d）</li>
 *   <li>{@link #PROMISE}：承诺（"答应周末看电影"）—— 本期仅留存，不排期</li>
 * </ul>
 */
public enum MemoryItemKind {
    PLAN_EVENT,
    EMOTION,
    PROMISE
}
```

```java
// MemoryItemStatus.java
package com.sanyan.memory.internal.item;

/**
 * 结构化记忆条目状态（spec §3.1）。存库为大写 enum name。
 *
 * <ul>
 *   <li>{@link #PENDING}：待处理（抽取后初始态）</li>
 *   <li>{@link #DONE}：已被主动消息消费（发送后 markMemoryItemDone）</li>
 *   <li>{@link #EXPIRED}：过期作废（预留，本期不自动流转）</li>
 * </ul>
 */
public enum MemoryItemStatus {
    PENDING,
    DONE,
    EXPIRED
}
```

```java
// MemoryItemEntity.java
package com.sanyan.memory.internal.item;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;

import java.time.Instant;

/**
 * 结构化记忆条目（V9 memory_item 表，Plan 4）。
 *
 * <p>与 {@code memory_profiles.summary_text}（宏观画像，缓慢演化）互补：本表存一条条
 * 具体的、带时间的、日后值得主动提起的点（spec §4.1）。由
 * {@link com.sanyan.memory.internal.item.MemoryItemExtractService} 实时抽取写入。
 *
 * <p>无外键（spec §3.1，与 intimacy_logs 等审计/状态表一致，降低耦合）。
 *
 * <p>{@code kind} / {@code status} 用 {@code @Enumerated(STRING)} 存大写 enum name，
 * 与 V9 migration 的 CHECK 约束（大写值）对齐（附录约定）。
 */
@Data
@Entity
@Table(name = "memory_item")
public class MemoryItemEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "character_id", nullable = false)
    private Long characterId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private MemoryItemKind kind;

    @Column(nullable = false, columnDefinition = "text")
    private String content;

    /** 该条记忆该"冒出来"的时间（PLAN_EVENT=事件当天 9:00；EMOTION=次日 9:00）。 */
    @Column(name = "salient_at", nullable = false)
    private Instant salientAt;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private MemoryItemStatus status = MemoryItemStatus.PENDING;

    @Column(name = "source_message_id")
    private Long sourceMessageId;

    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;
}
```

```java
// MemoryItemRepository.java
package com.sanyan.memory.internal.item;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

/**
 * memory_item 仓储。{@code findByUserIdAndCharacterIdAndStatus} 供抽取去重（拉该用户当前
 * PENDING 条目喂给 LLM 判定新建/更新/跳过，spec §4.3）。
 */
public interface MemoryItemRepository extends JpaRepository<MemoryItemEntity, Long> {

    List<MemoryItemEntity> findByUserIdAndCharacterIdAndStatus(
            Long userId, Long characterId, MemoryItemStatus status);
}
```

```java
// MemoryItemTestFixtures.java
package com.sanyan.memory.internal.item.fixtures;

import com.sanyan.memory.internal.item.MemoryItemEntity;
import com.sanyan.memory.internal.item.MemoryItemKind;
import com.sanyan.memory.internal.item.MemoryItemStatus;

import java.time.Instant;

/**
 * MemoryItemEntity Object Mother（java-backend-business-layer.md §5.2）。
 * 测试中需要 MemoryItemEntity 时一律通过这里构造，不允许裸 new。
 */
public final class MemoryItemTestFixtures {

    private MemoryItemTestFixtures() {}

    public static MemoryItemEntity planEvent(Long userId, Long characterId, String content, Instant salientAt) {
        return build(userId, characterId, MemoryItemKind.PLAN_EVENT, content, salientAt);
    }

    public static MemoryItemEntity emotion(Long userId, Long characterId, String content, Instant salientAt) {
        return build(userId, characterId, MemoryItemKind.EMOTION, content, salientAt);
    }

    public static MemoryItemEntity promise(Long userId, Long characterId, String content, Instant salientAt) {
        return build(userId, characterId, MemoryItemKind.PROMISE, content, salientAt);
    }

    private static MemoryItemEntity build(Long userId, Long characterId,
                                          MemoryItemKind kind, String content, Instant salientAt) {
        MemoryItemEntity e = new MemoryItemEntity();
        e.setUserId(userId);
        e.setCharacterId(characterId);
        e.setKind(kind);
        e.setContent(content);
        e.setSalientAt(salientAt);
        e.setStatus(MemoryItemStatus.PENDING);
        return e;
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryItemRepositoryIT test`
Expected: BUILD SUCCESS，2 个测试通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/item/ \
        server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/internal/item/
git commit -m "feat(memory-core): 新增 MemoryItem 实体/枚举/仓储/Fixture（结构化记忆条目）"
```

---

#### Task F2: memory-api 加 MemoryItemDto + MemoryItemScheduledEvent + MemoryApi 接口加 getMemoryItem / markMemoryItemDone

**Files:**
- Create: `server/business_packages/sanyan-memory-api/src/main/java/com/sanyan/memory/dto/MemoryItemDto.java`
- Create: `server/business_packages/sanyan-memory-api/src/main/java/com/sanyan/memory/event/MemoryItemScheduledEvent.java`
- Modify: `server/business_packages/sanyan-memory-api/src/main/java/com/sanyan/memory/MemoryApi.java`

> 本 task 只动 -api 契约（接口签名 + DTO + 事件，无业务逻辑），属"契约定义"。验证方式 = `mvn compile` 通过 + memory-core 因 MemoryApiImpl 尚未实现新方法而**编译失败**（证明接口确实新增了抽象方法）。F3 补实现让其恢复编译。

- [ ] **Step 1: 写契约（DTO + 事件 record，照附录 B 严格签名）**

```java
// MemoryItemDto.java
package com.sanyan.memory.dto;

import java.time.Instant;

/**
 * 结构化记忆条目跨模块 DTO（Plan 4）。proactive 域生成器通过 {@code MemoryApi.getMemoryItem}
 * 拿 {@code content} 拼主动消息文案（"周三面试怎么样啦"）。
 *
 * <p>{@code kind} / {@code status} 为字符串形态（大写 enum name），避免把 memory-core 的
 * 内部枚举 {@code MemoryItemKind} / {@code MemoryItemStatus} 泄漏到 -api 契约。
 */
public record MemoryItemDto(
        Long id,
        Long userId,
        Long characterId,
        String kind,
        String content,
        Instant salientAt,
        String status) {
}
```

```java
// MemoryItemScheduledEvent.java
package com.sanyan.memory.event;

import java.time.Instant;

/**
 * 结构化记忆条目"已排期"领域事件（Plan 4，spec §4.4）。
 *
 * <p>{@link com.sanyan.memory.internal.item.MemoryItemExtractService} 抽出需要排期主动消息的
 * 条目（PLAN_EVENT / EMOTION）并落库后发布；proactive-core 订阅后按 {@code kind} 排
 * {@code events_pending}（c 事件追问 / d 情绪关怀），{@code scheduledAt = salientAt}。
 *
 * <p>事件类放发布方（memory）的 -api/event/（java-backend §6），订阅方只需依赖 memory-api。
 *
 * @param memoryItemId 条目 id（proactive 排期后存 payload，发送时回查 + markDone）
 * @param userId       用户 id
 * @param characterId  角色 id
 * @param kind         条目分类大写 name（PLAN_EVENT / EMOTION）
 * @param salientAt    该冒出来的时间（proactive 用作 scheduled_at）
 */
public record MemoryItemScheduledEvent(
        Long memoryItemId,
        Long userId,
        Long characterId,
        String kind,
        Instant salientAt) {
}
```

修改 `MemoryApi.java`，在已有 `getRelevantContext` 之后新增两个方法签名（保留原 import + 加 MemoryItemDto import）：

```java
// MemoryApi.java （新增片段）
import com.sanyan.memory.dto.MemoryItemDto;

// ... interface MemoryApi 内，getRelevantContext 之后：

    /**
     * 按 id 查结构化记忆条目（Plan 4）。供 proactive 生成器拿 content 拼文案。
     *
     * @param itemId 条目 id
     * @return 条目 DTO
     * @throws com.sanyan.common.error.BusinessException 条目不存在时抛 MEMORY_ITEM_NOT_FOUND
     */
    MemoryItemDto getMemoryItem(Long itemId);

    /**
     * 标记条目已被主动消息消费（status=DONE）。主动消息发送成功后由 proactive 调用。
     *
     * @param itemId 条目 id
     */
    void markMemoryItemDone(Long itemId);
```

- [ ] **Step 2: 运行编译确认 memory-core 失败（证明接口新增了抽象方法）**

Run: `cd server && mvn -q -pl business_packages/sanyan-memory-api -am compile && mvn -q -pl business_packages/sanyan-memory-core compile`
Expected: memory-api 编译 SUCCESS；memory-core 编译 **FAILURE**（`MemoryApiImpl is not abstract and does not override abstract method getMemoryItem / markMemoryItemDone`）。

- [ ] **Step 3: 无需实现（本 task 仅契约）**

本 task 不写实现。memory-core 的编译恢复由 F3 完成（MemoryApiImpl 实现两个新方法）。

- [ ] **Step 4: 验证 -api 模块自身编译通过**

Run: `cd server && mvn -q -pl business_packages/sanyan-memory-api -am compile`
Expected: BUILD SUCCESS（-api 不含实现，自身能编过；core 失败是预期的，留给 F3）。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-memory-api/src/main/java/com/sanyan/memory/
git commit -m "feat(memory-api): MemoryApi 加 getMemoryItem/markMemoryItemDone + MemoryItemDto + MemoryItemScheduledEvent"
```

---

#### Task F3: MemoryErrCode 加 5003 + MemoryApiImpl 实现 getMemoryItem / markMemoryItemDone

**Files:**
- Modify: `server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/MemoryErrCode.java`
- Modify: `server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/api/MemoryApiImpl.java`
- Test: `server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/api/MemoryApiImplTest.java`

> 现有 MemoryApiImpl 是薄壳，只注入 `MemoryContextBuilder`。F3 给它再注入 `MemoryItemRepository`，实现两个新方法。ApiImpl 直接操作仓储（条目查/标记是简单 CRUD，无复杂编排，无需另抽 internal Service —— java-backend §3.3「简单规则直接在方法里」）。

- [ ] **Step 1: 写失败测试（Mockito 单测，mock MemoryItemRepository + MemoryContextBuilder）**

```java
package com.sanyan.memory.api;

import com.sanyan.common.error.BusinessException;
import com.sanyan.memory.dto.MemoryItemDto;
import com.sanyan.memory.internal.MemoryErrCode;
import com.sanyan.memory.internal.item.MemoryItemEntity;
import com.sanyan.memory.internal.item.MemoryItemKind;
import com.sanyan.memory.internal.item.MemoryItemRepository;
import com.sanyan.memory.internal.item.MemoryItemStatus;
import com.sanyan.memory.internal.item.fixtures.MemoryItemTestFixtures;
import com.sanyan.memory.internal.orchestrator.MemoryContextBuilder;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class MemoryApiImplTest {

    @Mock MemoryContextBuilder builder;
    @Mock MemoryItemRepository itemRepository;
    @InjectMocks MemoryApiImpl api;

    @Test
    void getMemoryItem_should_map_entity_to_dto() {
        MemoryItemEntity entity = MemoryItemTestFixtures.planEvent(
                7L, 1L, "周三面试", Instant.parse("2026-06-03T09:00:00Z"));
        entity.setId(99L);
        when(itemRepository.findById(99L)).thenReturn(Optional.of(entity));

        MemoryItemDto dto = api.getMemoryItem(99L);

        assertThat(dto.id()).isEqualTo(99L);
        assertThat(dto.userId()).isEqualTo(7L);
        assertThat(dto.characterId()).isEqualTo(1L);
        assertThat(dto.kind()).isEqualTo(MemoryItemKind.PLAN_EVENT.name());
        assertThat(dto.content()).isEqualTo("周三面试");
        assertThat(dto.salientAt()).isEqualTo(Instant.parse("2026-06-03T09:00:00Z"));
        assertThat(dto.status()).isEqualTo(MemoryItemStatus.PENDING.name());
    }

    @Test
    void getMemoryItem_should_throw_when_not_found() {
        when(itemRepository.findById(404L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> api.getMemoryItem(404L))
                .isInstanceOf(BusinessException.class)
                .satisfies(ex -> assertThat(((BusinessException) ex).getErrCode())
                        .isEqualTo(MemoryErrCode.MEMORY_ITEM_NOT_FOUND));

        verify(itemRepository, never()).save(org.mockito.ArgumentMatchers.any());
    }

    @Test
    void markMemoryItemDone_should_set_status_done_and_save() {
        MemoryItemEntity entity = MemoryItemTestFixtures.emotion(
                7L, 1L, "最近压力大", Instant.now());
        entity.setId(55L);
        when(itemRepository.findById(55L)).thenReturn(Optional.of(entity));

        api.markMemoryItemDone(55L);

        ArgumentCaptor<MemoryItemEntity> captor = ArgumentCaptor.forClass(MemoryItemEntity.class);
        verify(itemRepository).save(captor.capture());
        assertThat(captor.getValue().getStatus()).isEqualTo(MemoryItemStatus.DONE);
    }

    @Test
    void markMemoryItemDone_should_throw_when_not_found() {
        when(itemRepository.findById(404L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> api.markMemoryItemDone(404L))
                .isInstanceOf(BusinessException.class)
                .satisfies(ex -> assertThat(((BusinessException) ex).getErrCode())
                        .isEqualTo(MemoryErrCode.MEMORY_ITEM_NOT_FOUND));
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryApiImplTest test`
Expected: 编译失败（`MemoryErrCode.MEMORY_ITEM_NOT_FOUND` 不存在；`MemoryApiImpl` 无 `MemoryItemRepository` 字段、无两个新方法）。

- [ ] **Step 3: 写最小实现**

`MemoryErrCode.java` 新增 5003（在 EMBEDDING_SERVICE_UNAVAILABLE 之后）：

```java
    PROFILE_REFRESH_CONFLICT(5001, "Profile 刷新失败（乐观锁冲突）"),
    EMBEDDING_SERVICE_UNAVAILABLE(5002, "Embedding 服务不可用"),
    MEMORY_ITEM_NOT_FOUND(5003, "结构化记忆条目不存在"),
    ;
```

`MemoryApiImpl.java` 注入仓储 + 实现两个方法：

```java
package com.sanyan.memory.api;

import com.sanyan.common.error.BusinessException;
import com.sanyan.memory.MemoryApi;
import com.sanyan.memory.dto.MemoryContext;
import com.sanyan.memory.dto.MemoryItemDto;
import com.sanyan.memory.internal.MemoryErrCode;
import com.sanyan.memory.internal.item.MemoryItemEntity;
import com.sanyan.memory.internal.item.MemoryItemRepository;
import com.sanyan.memory.internal.item.MemoryItemStatus;
import com.sanyan.memory.internal.orchestrator.MemoryContextBuilder;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * {@link MemoryApi} 在 -core 的实现。
 *
 * <p>getRelevantContext 委托 {@link MemoryContextBuilder}（Plan 2）；
 * Plan 4 新增 getMemoryItem / markMemoryItemDone 直接操作 {@link MemoryItemRepository}
 * （简单 CRUD，无复杂编排，按 java-backend §3.3 直接在方法里处理）。
 */
@Service
@RequiredArgsConstructor
public class MemoryApiImpl implements MemoryApi {

    private final MemoryContextBuilder builder;
    private final MemoryItemRepository itemRepository;

    @Override
    public MemoryContext getRelevantContext(Long userId, Long characterId, String currentUserMessage) {
        return builder.build(userId, characterId, currentUserMessage);
    }

    @Override
    public MemoryItemDto getMemoryItem(Long itemId) {
        MemoryItemEntity e = itemRepository.findById(itemId)
                .orElseThrow(() -> new BusinessException(MemoryErrCode.MEMORY_ITEM_NOT_FOUND));
        return toDto(e);
    }

    @Override
    @Transactional
    public void markMemoryItemDone(Long itemId) {
        MemoryItemEntity e = itemRepository.findById(itemId)
                .orElseThrow(() -> new BusinessException(MemoryErrCode.MEMORY_ITEM_NOT_FOUND));
        e.setStatus(MemoryItemStatus.DONE);
        itemRepository.save(e);
    }

    private static MemoryItemDto toDto(MemoryItemEntity e) {
        return new MemoryItemDto(
                e.getId(),
                e.getUserId(),
                e.getCharacterId(),
                e.getKind().name(),
                e.getContent(),
                e.getSalientAt(),
                e.getStatus().name());
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryApiImplTest test`
Expected: BUILD SUCCESS，4 个测试通过。

> 同步更新 `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`：MemoryErrCode（5000-5999）表加一行 `| 5003 | MEMORY_ITEM_NOT_FOUND | 结构化记忆条目不存在 |`，历史变更附 `| 2026-05-27 | Plan 4：memory 域新增 5003 |`。（registry 是 docs，非业务逻辑，与本 task 一起改一起 commit。）

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/MemoryErrCode.java \
        server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/api/MemoryApiImpl.java \
        server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/api/MemoryApiImplTest.java \
        server/foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md
git commit -m "feat(memory-core): MemoryApiImpl 实现条目查询/标记完成 + MemoryErrCode 5003"
```

---

#### Task F4: MemoryItemExtractResult（LLM JSON 解析 DTO）+ 解析方法

**Files:**
- Create: `server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/item/MemoryItemExtractResult.java`
- Test: `server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/internal/item/MemoryItemExtractResultParserTest.java`

> 解析 LLM 抽取返回的 JSON。无 JsonUtil，沿用项目既有 Jackson `ObjectMapper` 模式（见 RagIndexWorker）。解析方法 `parse(String json)` 做成 record 的 static 工厂（无状态 → static，符合 static 优先原则），容错：LLM 可能用 ```json 代码块包裹，先剥壳。

LLM 抽取约定的 JSON 形态（F5 prompt 会要求 LLM 严格输出）：

```json
{
  "items": [
    { "action": "NEW",    "kind": "PLAN_EVENT", "content": "周三下午有面试", "dateHint": "2026-06-03", "targetId": null },
    { "action": "UPDATE", "kind": "EMOTION",    "content": "面试很焦虑、压力大", "dateHint": null, "targetId": 55 },
    { "action": "SKIP",   "kind": "PLAN_EVENT", "content": "", "dateHint": null, "targetId": 99 }
  ]
}
```

- [ ] **Step 1: 写失败测试**

```java
package com.sanyan.memory.internal.item;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class MemoryItemExtractResultParserTest {

    @Test
    void parse_should_read_items_with_all_fields() {
        String json = """
                {
                  "items": [
                    { "action": "NEW",    "kind": "PLAN_EVENT", "content": "周三下午有面试", "dateHint": "2026-06-03", "targetId": null },
                    { "action": "UPDATE", "kind": "EMOTION",    "content": "面试很焦虑",     "dateHint": null,         "targetId": 55 },
                    { "action": "SKIP",   "kind": "PLAN_EVENT", "content": "",              "dateHint": null,         "targetId": 99 }
                  ]
                }
                """;

        MemoryItemExtractResult result = MemoryItemExtractResult.parse(json);

        assertThat(result.items()).hasSize(3);

        MemoryItemExtractResult.Item first = result.items().get(0);
        assertThat(first.action()).isEqualTo("NEW");
        assertThat(first.kind()).isEqualTo("PLAN_EVENT");
        assertThat(first.content()).isEqualTo("周三下午有面试");
        assertThat(first.dateHint()).isEqualTo("2026-06-03");
        assertThat(first.targetId()).isNull();

        MemoryItemExtractResult.Item second = result.items().get(1);
        assertThat(second.action()).isEqualTo("UPDATE");
        assertThat(second.targetId()).isEqualTo(55L);

        MemoryItemExtractResult.Item third = result.items().get(2);
        assertThat(third.action()).isEqualTo("SKIP");
        assertThat(third.targetId()).isEqualTo(99L);
    }

    @Test
    void parse_should_strip_markdown_code_fence() {
        String json = """
                ```json
                { "items": [ { "action": "NEW", "kind": "EMOTION", "content": "心情不错", "dateHint": null, "targetId": null } ] }
                ```
                """;

        MemoryItemExtractResult result = MemoryItemExtractResult.parse(json);

        assertThat(result.items()).hasSize(1);
        assertThat(result.items().get(0).kind()).isEqualTo("EMOTION");
    }

    @Test
    void parse_should_return_empty_items_on_blank_or_garbage() {
        assertThat(MemoryItemExtractResult.parse(null).items()).isEmpty();
        assertThat(MemoryItemExtractResult.parse("").items()).isEmpty();
        assertThat(MemoryItemExtractResult.parse("这不是 JSON").items()).isEmpty();
    }

    @Test
    void parse_should_default_items_to_empty_list_when_field_missing() {
        MemoryItemExtractResult result = MemoryItemExtractResult.parse("{}");
        assertThat(result.items()).isEmpty();
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryItemExtractResultParserTest test`
Expected: 编译失败（`MemoryItemExtractResult` 不存在）。

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.memory.internal.item;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.util.List;

/**
 * LLM 记忆抽取返回的 JSON 解析结果（spec §4.2 / §4.3）。
 *
 * <p>LLM 对最近消息 + 现有 PENDING 条目判定后，逐条给出 action（NEW / UPDATE / SKIP）。
 * {@link MemoryItemExtractService} 拿到后按 action 落库新建 / 更新 / 跳过。
 *
 * <p>容错：LLM 可能用 ```json 代码块包裹、可能输出非 JSON 噪音；{@link #parse} 一律降级为
 * 空 items（不抛异常），保证抽取链路即使 LLM 抽风也不影响主对话（抽取是后台异步任务）。
 *
 * <p>沿用项目既有 Jackson 模式（无 JsonUtil；见 {@code RagIndexWorker}）。
 */
@JsonIgnoreProperties(ignoreUnknown = true)
public record MemoryItemExtractResult(List<Item> items) {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    /**
     * 单条抽取项。
     *
     * @param action   NEW / UPDATE / SKIP
     * @param kind     PLAN_EVENT / EMOTION / PROMISE（大写 enum name）
     * @param content  条目内容（SKIP 时可空）
     * @param dateHint 事件日期提示（ISO 日期串，如 "2026-06-03"；无则 null）
     * @param targetId UPDATE / SKIP 时指向的已有条目 id；NEW 时 null
     */
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Item(String action, String kind, String content, String dateHint, Long targetId) {}

    /** items 为 null 时归一化为空列表，避免下游空指针。 */
    public MemoryItemExtractResult {
        items = items == null ? List.of() : items;
    }

    /**
     * 解析 LLM 返回文本为抽取结果。容错：blank / 非 JSON / 缺字段一律降级为空 items。
     *
     * @param raw LLM 原始返回（可能含 markdown 代码块围栏）
     * @return 解析结果；解析失败时 items 为空列表
     */
    public static MemoryItemExtractResult parse(String raw) {
        if (raw == null || raw.isBlank()) {
            return new MemoryItemExtractResult(List.of());
        }
        String cleaned = stripCodeFence(raw);
        try {
            return MAPPER.readValue(cleaned, MemoryItemExtractResult.class);
        } catch (Exception e) {
            return new MemoryItemExtractResult(List.of());
        }
    }

    /** 剥掉 LLM 常见的 ```json ... ``` 围栏，只保留中间内容。 */
    private static String stripCodeFence(String raw) {
        String s = raw.trim();
        if (s.startsWith("```")) {
            int firstNl = s.indexOf('\n');
            if (firstNl >= 0) {
                s = s.substring(firstNl + 1);
            }
            if (s.endsWith("```")) {
                s = s.substring(0, s.length() - 3);
            }
        }
        return s.trim();
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryItemExtractResultParserTest test`
Expected: BUILD SUCCESS，4 个测试通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/item/MemoryItemExtractResult.java \
        server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/internal/item/MemoryItemExtractResultParserTest.java
git commit -m "feat(memory-core): MemoryItemExtractResult LLM JSON 解析 DTO（容错降级）"
```

---

#### Task F5: MemoryItemExtractService（核心：查现有 PENDING → 拼 prompt → LLM → 按 action 落库 → 发事件）

**Files:**
- Create: `server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/item/MemoryItemExtractService.java`
- Test: `server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/internal/item/MemoryItemExtractServiceTest.java`

> **核心服务。** 流程（spec §4.2/§4.3）：
> 1. 查该 (user, character) 现有 PENDING 条目（去重上下文）
> 2. 拼抽取 prompt（system 规则 + user：最近消息 + 现有条目清单），LLM 调用照 MemorySummaryService（对话/条目塞进单条 user message content，`llmApi.chat(BACKGROUND, ...)`）
> 3. 解析 `MemoryItemExtractResult`（F4）
> 4. 按 action：NEW → 新建条目；UPDATE → 更新 targetId 指向条目的 content；SKIP → 不动
> 5. 对 NEW 的 PLAN_EVENT / EMOTION 条目 `publishEvent(MemoryItemScheduledEvent)`（PROMISE 本期仅留存不发事件，spec §12）
>
> **salient_at 计算（spec §4.2）：** PLAN_EVENT = dateHint 解析出的事件日期当天 9:00（无 dateHint 则默认次日 9:00）；EMOTION = 观察日次日 9:00。时区按系统默认 `ZoneId.systemDefault()`（dogfood 单实例够用）。注入 `Clock` 便于测试。
>
> **mock：** LlmApi / MemoryItemRepository / ApplicationEventPublisher 全 mock。

- [ ] **Step 1: 写失败测试**

```java
package com.sanyan.memory.internal.item;

import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.llm.dto.ChatMessage;
import com.sanyan.memory.event.MemoryItemScheduledEvent;
import com.sanyan.memory.internal.item.fixtures.MemoryItemTestFixtures;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.context.ApplicationEventPublisher;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneId;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyList;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class MemoryItemExtractServiceTest {

    @Mock LlmApi llmApi;
    @Mock MemoryItemRepository repository;
    @Mock ApplicationEventPublisher events;

    // 固定时钟：2026-05-27T12:00:00Z（便于断言次日 9:00 等派生时间）
    private final Clock clock = Clock.fixed(Instant.parse("2026-05-27T12:00:00Z"), ZoneId.of("UTC"));

    private MemoryItemExtractService service() {
        return new MemoryItemExtractService(llmApi, repository, events, clock);
    }

    @Test
    void extract_should_use_background_task_type_and_feed_existing_pending_items() {
        when(repository.findByUserIdAndCharacterIdAndStatus(7L, 1L, MemoryItemStatus.PENDING))
                .thenReturn(List.of(withId(MemoryItemTestFixtures.emotion(7L, 1L, "最近压力大", Instant.now()), 55L)));
        when(llmApi.chat(eq(LlmTaskType.BACKGROUND), anyList()))
                .thenReturn("{\"items\":[]}");

        service().extract(7L, 1L, "今天面试好紧张", 1001L);

        ArgumentCaptor<List<ChatMessage>> captor = ArgumentCaptor.forClass(List.class);
        verify(llmApi).chat(eq(LlmTaskType.BACKGROUND), captor.capture());
        // prompt 里必须带上现有 PENDING 条目（去重上下文）+ 最新用户消息
        String joined = captor.getValue().stream().map(ChatMessage::content).reduce("", String::concat);
        assertThat(joined).contains("最近压力大");
        assertThat(joined).contains("今天面试好紧张");
    }

    @Test
    void extract_NEW_plan_event_should_save_pending_and_publish_event() {
        when(repository.findByUserIdAndCharacterIdAndStatus(any(), any(), any()))
                .thenReturn(List.of());
        when(llmApi.chat(eq(LlmTaskType.BACKGROUND), anyList())).thenReturn("""
                {"items":[{"action":"NEW","kind":"PLAN_EVENT","content":"周三下午有面试","dateHint":"2026-06-03","targetId":null}]}
                """);
        when(repository.save(any())).thenAnswer(inv -> withId(inv.getArgument(0), 200L));

        service().extract(7L, 1L, "周三我有个面试", 1001L);

        ArgumentCaptor<MemoryItemEntity> saved = ArgumentCaptor.forClass(MemoryItemEntity.class);
        verify(repository).save(saved.capture());
        MemoryItemEntity e = saved.getValue();
        assertThat(e.getKind()).isEqualTo(MemoryItemKind.PLAN_EVENT);
        assertThat(e.getContent()).isEqualTo("周三下午有面试");
        assertThat(e.getStatus()).isEqualTo(MemoryItemStatus.PENDING);
        assertThat(e.getSourceMessageId()).isEqualTo(1001L);
        // PLAN_EVENT salient_at = 事件日 2026-06-03 当天 9:00（UTC）
        assertThat(e.getSalientAt()).isEqualTo(Instant.parse("2026-06-03T09:00:00Z"));

        ArgumentCaptor<MemoryItemScheduledEvent> evt = ArgumentCaptor.forClass(MemoryItemScheduledEvent.class);
        verify(events).publishEvent(evt.capture());
        assertThat(evt.getValue().memoryItemId()).isEqualTo(200L);
        assertThat(evt.getValue().kind()).isEqualTo("PLAN_EVENT");
        assertThat(evt.getValue().salientAt()).isEqualTo(Instant.parse("2026-06-03T09:00:00Z"));
    }

    @Test
    void extract_NEW_emotion_should_set_salient_next_day_9am_and_publish() {
        when(repository.findByUserIdAndCharacterIdAndStatus(any(), any(), any()))
                .thenReturn(List.of());
        when(llmApi.chat(eq(LlmTaskType.BACKGROUND), anyList())).thenReturn("""
                {"items":[{"action":"NEW","kind":"EMOTION","content":"最近压力很大","dateHint":null,"targetId":null}]}
                """);
        when(repository.save(any())).thenAnswer(inv -> withId(inv.getArgument(0), 201L));

        service().extract(7L, 1L, "唉 最近压力好大", 1002L);

        ArgumentCaptor<MemoryItemEntity> saved = ArgumentCaptor.forClass(MemoryItemEntity.class);
        verify(repository).save(saved.capture());
        // EMOTION salient_at = 观察日(2026-05-27)次日 9:00 UTC = 2026-05-28T09:00:00Z
        assertThat(saved.getValue().getSalientAt()).isEqualTo(Instant.parse("2026-05-28T09:00:00Z"));
        verify(events, times(1)).publishEvent(any(MemoryItemScheduledEvent.class));
    }

    @Test
    void extract_UPDATE_should_modify_existing_content_and_not_publish_event() {
        MemoryItemEntity existing = withId(MemoryItemTestFixtures.emotion(7L, 1L, "压力大", Instant.now()), 55L);
        when(repository.findByUserIdAndCharacterIdAndStatus(7L, 1L, MemoryItemStatus.PENDING))
                .thenReturn(List.of(existing));
        when(repository.findById(55L)).thenReturn(java.util.Optional.of(existing));
        when(llmApi.chat(eq(LlmTaskType.BACKGROUND), anyList())).thenReturn("""
                {"items":[{"action":"UPDATE","kind":"EMOTION","content":"面试相关的焦虑、压力大","dateHint":null,"targetId":55}]}
                """);

        service().extract(7L, 1L, "面试好紧张", 1003L);

        ArgumentCaptor<MemoryItemEntity> saved = ArgumentCaptor.forClass(MemoryItemEntity.class);
        verify(repository).save(saved.capture());
        assertThat(saved.getValue().getId()).isEqualTo(55L);
        assertThat(saved.getValue().getContent()).isEqualTo("面试相关的焦虑、压力大");
        // UPDATE 不重新排期（已有条目已排过）
        verify(events, never()).publishEvent(any());
    }

    @Test
    void extract_SKIP_should_not_save_nor_publish() {
        when(repository.findByUserIdAndCharacterIdAndStatus(any(), any(), any()))
                .thenReturn(List.of(withId(MemoryItemTestFixtures.planEvent(7L, 1L, "周三面试", Instant.now()), 99L)));
        when(llmApi.chat(eq(LlmTaskType.BACKGROUND), anyList())).thenReturn("""
                {"items":[{"action":"SKIP","kind":"PLAN_EVENT","content":"","dateHint":null,"targetId":99}]}
                """);

        service().extract(7L, 1L, "对 就是周三那个面试", 1004L);

        verify(repository, never()).save(any());
        verify(events, never()).publishEvent(any());
    }

    @Test
    void extract_NEW_promise_should_save_but_not_publish_event() {
        when(repository.findByUserIdAndCharacterIdAndStatus(any(), any(), any()))
                .thenReturn(List.of());
        when(llmApi.chat(eq(LlmTaskType.BACKGROUND), anyList())).thenReturn("""
                {"items":[{"action":"NEW","kind":"PROMISE","content":"答应周末陪你看电影","dateHint":null,"targetId":null}]}
                """);
        when(repository.save(any())).thenAnswer(inv -> withId(inv.getArgument(0), 202L));

        service().extract(7L, 1L, "周末陪你看电影好不好", 1005L);

        verify(repository, times(1)).save(any());
        // PROMISE 本期仅留存，不发排期事件（spec §12）
        verify(events, never()).publishEvent(any());
    }

    private static MemoryItemEntity withId(MemoryItemEntity e, long id) {
        e.setId(id);
        return e;
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryItemExtractServiceTest test`
Expected: 编译失败（`MemoryItemExtractService` 不存在）。

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.memory.internal.item;

import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.llm.dto.ChatMessage;
import com.sanyan.memory.event.MemoryItemScheduledEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.time.Instant;
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.ZoneId;
import java.util.ArrayList;
import java.util.List;

/**
 * Plan 4：结构化记忆「实时抽取」核心服务（spec §4.2 / §4.3）。
 *
 * <p>由 {@link com.sanyan.memory.event.MemoryItemExtractListener} 在用户消息持久化后
 * 异步调用。流程：
 * <ol>
 *   <li>查该 (user, character) 现有 PENDING 条目作为去重上下文</li>
 *   <li>拼抽取 prompt（system 规则 + user：最新消息 + 现有条目清单），调
 *       {@link LlmApi#chat}（{@link LlmTaskType#BACKGROUND}，走 DeepSeek）</li>
 *   <li>{@link MemoryItemExtractResult#parse} 解析（容错降级空 items）</li>
 *   <li>按 action 落库：NEW 新建 / UPDATE 改已有 content / SKIP 不动</li>
 *   <li>对 NEW 的 PLAN_EVENT / EMOTION 发 {@link MemoryItemScheduledEvent} 让 proactive 排期；
 *       PROMISE 本期仅留存不发事件（spec §12）</li>
 * </ol>
 *
 * <p>LLM 调用模式照 {@code MemorySummaryService}：对话历史/条目清单塞进单条 user message
 * content，不展开成多个 turns（避免 LLM 接话而非执行指令）。
 *
 * <p>salient_at 计算（spec §4.2）：
 * <ul>
 *   <li>PLAN_EVENT：dateHint 解析出的事件日期当天 09:00；dateHint 缺失 / 非法时降级为次日 09:00</li>
 *   <li>EMOTION：观察日（now）次日 09:00</li>
 * </ul>
 * 时区用系统默认（单实例 dogfood 够用）；{@link Clock} 注入便于测试固定时间。
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class MemoryItemExtractService {

    /** 抽取规则 system prompt。硬约束：JSON-only 输出 + action/kind 枚举值大写。 */
    static final String SYSTEM_PROMPT = """
            你是用户长期记忆的抽取器。下面会给你「用户最新的一句话」和「该用户当前已记录但还没处理的记忆条目清单」。
            请你从最新这句话里抽取值得日后主动提起的「具体的、带时间或情绪的点」，并对照已有条目判定。

            只抽三类（kind 取大写）：
            - PLAN_EVENT：有明确时间的事（如"周三有面试"）
            - EMOTION：情绪状态（如"最近压力大、很焦虑"）
            - PROMISE：承诺（如"答应周末陪你看电影"）
            喜好 / 长期画像不要抽（那属于另一条画像通道）。

            对每条信息判定 action（大写）：
            - NEW：是一件已有条目里没有的新事 → 新建
            - UPDATE：是对某条已有条目的补充 / 更新 → 给出该条目的 targetId
            - SKIP：已有条目已经覆盖、无需变化 → 给出对应 targetId

            PLAN_EVENT 若能判断出事件日期，填 dateHint（ISO 日期，如 "2026-06-03"）；判断不出填 null。

            严格只输出 JSON（不要解释、不要 markdown 围栏）：
            {"items":[{"action":"NEW|UPDATE|SKIP","kind":"PLAN_EVENT|EMOTION|PROMISE","content":"...","dateHint":"YYYY-MM-DD 或 null","targetId":数字或 null}]}
            没有可抽取的内容时输出 {"items":[]}。
            """;

    private final LlmApi llmApi;
    private final MemoryItemRepository repository;
    private final ApplicationEventPublisher events;
    private final Clock clock;

    @Transactional
    public void extract(Long userId, Long characterId, String latestUserMessage, Long sourceMessageId) {
        List<MemoryItemEntity> existing =
                repository.findByUserIdAndCharacterIdAndStatus(userId, characterId, MemoryItemStatus.PENDING);

        String userPrompt = buildUserPrompt(latestUserMessage, existing);
        List<ChatMessage> messages = new ArrayList<>();
        messages.add(ChatMessage.system(SYSTEM_PROMPT));
        messages.add(ChatMessage.user(userPrompt));

        String raw = llmApi.chat(LlmTaskType.BACKGROUND, messages);
        MemoryItemExtractResult result = MemoryItemExtractResult.parse(raw);

        for (MemoryItemExtractResult.Item item : result.items()) {
            apply(userId, characterId, sourceMessageId, item);
        }
    }

    private void apply(Long userId, Long characterId, Long sourceMessageId,
                       MemoryItemExtractResult.Item item) {
        String action = item.action() == null ? "" : item.action().toUpperCase();
        switch (action) {
            case "NEW" -> handleNew(userId, characterId, sourceMessageId, item);
            case "UPDATE" -> handleUpdate(item);
            default -> { /* SKIP 或未知 action：不动 */ }
        }
    }

    private void handleNew(Long userId, Long characterId, Long sourceMessageId,
                           MemoryItemExtractResult.Item item) {
        MemoryItemKind kind = parseKind(item.kind());
        if (kind == null) {
            return;
        }
        Instant salientAt = computeSalientAt(kind, item.dateHint());

        MemoryItemEntity entity = new MemoryItemEntity();
        entity.setUserId(userId);
        entity.setCharacterId(characterId);
        entity.setKind(kind);
        entity.setContent(item.content());
        entity.setSalientAt(salientAt);
        entity.setStatus(MemoryItemStatus.PENDING);
        entity.setSourceMessageId(sourceMessageId);
        MemoryItemEntity saved = repository.save(entity);

        // 仅 PLAN_EVENT / EMOTION 排期主动消息；PROMISE 本期仅留存（spec §12）
        if (kind == MemoryItemKind.PLAN_EVENT || kind == MemoryItemKind.EMOTION) {
            events.publishEvent(new MemoryItemScheduledEvent(
                    saved.getId(), userId, characterId, kind.name(), salientAt));
        }
    }

    private void handleUpdate(MemoryItemExtractResult.Item item) {
        if (item.targetId() == null) {
            return;
        }
        repository.findById(item.targetId()).ifPresent(existing -> {
            existing.setContent(item.content());
            repository.save(existing);
        });
    }

    private Instant computeSalientAt(MemoryItemKind kind, String dateHint) {
        ZoneId zone = clock.getZone();
        LocalTime nineAm = LocalTime.of(9, 0);
        if (kind == MemoryItemKind.PLAN_EVENT) {
            LocalDate eventDate = parseDateOrNull(dateHint);
            if (eventDate != null) {
                return eventDate.atTime(nineAm).atZone(zone).toInstant();
            }
            // 无法判断事件日 → 降级为次日 9:00（spec §4.2）
            return LocalDate.now(clock).plusDays(1).atTime(nineAm).atZone(zone).toInstant();
        }
        // EMOTION：观察日次日 9:00
        return LocalDate.now(clock).plusDays(1).atTime(nineAm).atZone(zone).toInstant();
    }

    private static LocalDate parseDateOrNull(String dateHint) {
        if (dateHint == null || dateHint.isBlank() || "null".equalsIgnoreCase(dateHint.trim())) {
            return null;
        }
        try {
            return LocalDate.parse(dateHint.trim());
        } catch (Exception e) {
            return null;
        }
    }

    private static MemoryItemKind parseKind(String kind) {
        if (kind == null) {
            return null;
        }
        try {
            return MemoryItemKind.valueOf(kind.trim().toUpperCase());
        } catch (Exception e) {
            return null;
        }
    }

    private static String buildUserPrompt(String latestUserMessage, List<MemoryItemEntity> existing) {
        StringBuilder sb = new StringBuilder();
        sb.append("【用户最新的一句话】\n").append(latestUserMessage == null ? "" : latestUserMessage).append("\n\n");
        sb.append("【当前已记录的待处理条目】\n");
        if (existing.isEmpty()) {
            sb.append("（暂无）\n");
        } else {
            for (MemoryItemEntity e : existing) {
                sb.append("- id=").append(e.getId())
                        .append(" [").append(e.getKind().name()).append("] ")
                        .append(e.getContent()).append("\n");
            }
        }
        return sb.toString();
    }
}
```

> **Clock Bean：** `MemoryItemExtractService` 注入 `java.time.Clock`。若 bootstrap 尚无 `Clock` Bean，在本 task 一并加（配置类，TDD 例外）：在 memory-core 现有配置或新建 `internal/item/...` 同级不放——遵循"等第二个调用方再下沉"，先在 bootstrap 注册全局 `@Bean Clock clock() { return Clock.systemDefaultZone(); }`。实现者核对 bootstrap 是否已有 `Clock` Bean（Plan 3 时间感知若已引入则直接复用，不重复定义）。

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryItemExtractServiceTest test`
Expected: BUILD SUCCESS，6 个测试通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/item/MemoryItemExtractService.java \
        server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/internal/item/MemoryItemExtractServiceTest.java
git commit -m "feat(memory-core): MemoryItemExtractService 实时抽取（LLM 去重 + 落库 + 排期事件）"
```

---

#### Task F6: MemoryItemExtractListener（挂 MessagePersistedEvent，仅 USER，AFTER_COMMIT + @Async，吞异常）

**Files:**
- Create: `server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/event/MemoryItemExtractListener.java`
- Test: `server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/event/MemoryItemExtractListenerTest.java`

> 照 `UserMessageProfileRefreshListener` 模板：`@Async @TransactionalEventListener(AFTER_COMMIT)`、`SenderType.USER.equalsIgnoreCase` 过滤、try/catch 吞异常 log.error。
> 取最新用户消息文本：listener 收到的 `MessagePersistedEvent` 只有 messageId，需调 `chatApi` 拿内容。`UserMessageProfileRefreshListener` 用 `chatApi.listRecentByUser(userId, N)`；本 listener 只需最新一条 → 取 `listRecentByUser(userId, 1)` 的第 0 条 content（ChatApi 按 id 降序返回，第 0 条即最新）。
> **去重/节流：** 抽取本身靠 LLM 判定去重（spec §4.3），listener 不再加 KvCache 节流（与 profile 刷新不同——profile 是重写整段画像所以节流，抽取是逐条增量所以每条用户消息都抽）。

- [ ] **Step 1: 写失败测试（Mockito 单测，mock ChatApi + MemoryItemExtractService）**

```java
package com.sanyan.memory.event;

import com.sanyan.chat.ChatApi;
import com.sanyan.chat.SenderType;
import com.sanyan.chat.dto.MessageDto;
import com.sanyan.chat.event.MessagePersistedEvent;
import com.sanyan.memory.internal.item.MemoryItemExtractService;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.LocalDateTime;
import java.util.List;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyInt;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class MemoryItemExtractListenerTest {

    @Mock ChatApi chatApi;
    @Mock MemoryItemExtractService extractService;
    @InjectMocks MemoryItemExtractListener listener;

    @Test
    void should_skip_when_role_is_ai() {
        listener.onMessagePersisted(new MessagePersistedEvent(10L, 7L, 1L, SenderType.AI));

        verifyNoInteractions(extractService);
        verify(chatApi, never()).listRecentByUser(anyLong(), anyInt());
    }

    @Test
    void should_extract_latest_user_message_for_user_role() {
        // MessageDto 实际签名 (id, userId, senderType, content, createdAt)，无 characterId 字段、createdAt 为 LocalDateTime
        when(chatApi.listRecentByUser(7L, 1))
                .thenReturn(List.of(new MessageDto(10L, 7L, SenderType.USER, "周三我有个面试", LocalDateTime.now())));

        listener.onMessagePersisted(new MessagePersistedEvent(10L, 7L, 1L, SenderType.USER));

        verify(extractService).extract(7L, 1L, "周三我有个面试", 10L);
    }

    @Test
    void should_skip_when_no_recent_message_found() {
        when(chatApi.listRecentByUser(7L, 1)).thenReturn(List.of());

        listener.onMessagePersisted(new MessagePersistedEvent(10L, 7L, 1L, SenderType.USER));

        verify(extractService, never()).extract(anyLong(), anyLong(), any(), anyLong());
    }

    @Test
    void should_swallow_exception_from_extract_service() {
        when(chatApi.listRecentByUser(7L, 1))
                .thenReturn(List.of(new MessageDto(10L, 7L, SenderType.USER, "面试好紧张", LocalDateTime.now())));
        org.mockito.Mockito.doThrow(new RuntimeException("LLM 挂了"))
                .when(extractService).extract(eq(7L), eq(1L), any(), eq(10L));

        // 不抛异常（吞掉 + log.error），后台任务失败不影响主对话
        listener.onMessagePersisted(new MessagePersistedEvent(10L, 7L, 1L, SenderType.USER));

        verify(extractService).extract(7L, 1L, "面试好紧张", 10L);
    }
}
```

> 实现者核对 `MessageDto` 与 `ChatApi.listRecentByUser` 的真实签名/字段顺序（Read `sanyan-chat-api/.../dto/MessageDto.java` 与 `ChatApi.java`），按真实记录类型构造测试数据；上面按 `(messageId, userId, characterId, senderType, content, createdAt)` 假定，若不一致以真实为准调整。

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryItemExtractListenerTest test`
Expected: 编译失败（`MemoryItemExtractListener` 不存在）。

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.memory.event;

import com.sanyan.chat.ChatApi;
import com.sanyan.chat.SenderType;
import com.sanyan.chat.dto.MessageDto;
import com.sanyan.chat.event.MessagePersistedEvent;
import com.sanyan.memory.internal.item.MemoryItemExtractService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

import java.util.List;

/**
 * Plan 4：用户消息触发结构化记忆「实时抽取」（spec §4.2）。
 *
 * <p>监听 chat 域 {@link MessagePersistedEvent}，与 {@code UserMessageProfileRefreshListener}
 * 并行（两条 listener 各管一条记忆通道：画像 / 结构化条目）。
 *
 * <p>关键约束（照 profile 刷新 listener）：
 * <ul>
 *   <li>仅 {@code role=user}（AI 回复不触发）</li>
 *   <li>{@code @TransactionalEventListener(AFTER_COMMIT)}：等消息事务提交后再抽，避免回滚不一致</li>
 *   <li>{@code @Async}：LLM 调用数秒，异步不阻塞主对话</li>
 *   <li>异常吞掉 + log.error：后台任务失败决不能影响主对话</li>
 * </ul>
 *
 * <p>与 profile 刷新不同——本 listener<b>不加 KvCache 节流</b>：抽取靠 LLM 逐条增量判定去重
 * （spec §4.3），关键信息常在"临别最后一句"，每条用户消息都要抽，不能节流跳过。
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class MemoryItemExtractListener {

    private final ChatApi chatApi;
    private final MemoryItemExtractService extractService;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onMessagePersisted(MessagePersistedEvent event) {
        if (!SenderType.USER.equalsIgnoreCase(event.role())) {
            return;
        }
        try {
            // ChatApi 按 id 降序返回，第 0 条即本次持久化的最新用户消息
            List<MessageDto> recent = chatApi.listRecentByUser(event.userId(), 1);
            if (recent == null || recent.isEmpty()) {
                return;
            }
            String latest = recent.get(0).content();
            extractService.extract(event.userId(), event.characterId(), latest, event.messageId());
            log.info("记忆条目抽取完成 user={} char={} msg={}",
                    event.userId(), event.characterId(), event.messageId());
        } catch (Exception e) {
            // 后台任务失败不影响主对话——吞掉 + ERROR 日志便于排查
            log.error("记忆条目抽取失败 user={} char={} msg={}",
                    event.userId(), event.characterId(), event.messageId(), e);
        }
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-memory-core -Dtest=MemoryItemExtractListenerTest test`
Expected: BUILD SUCCESS，4 个测试通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/event/MemoryItemExtractListener.java \
        server/business_packages/sanyan-memory-core/src/test/java/com/sanyan/memory/event/MemoryItemExtractListenerTest.java
git commit -m "feat(memory-core): MemoryItemExtractListener 挂消息事件实时抽取记忆条目"
```

---

> **Phase F checkpoint：** 跑 memory-core 全包测试 + analyze 等价（mvn 编译）：
> `cd server && mvn -pl business_packages/sanyan-memory-core test`
> 期望全绿。memory 域改动未触碰 foundation 层，按测试粒度规范跑本包即可（B 阶段的 V9 schema IT 已在 Phase B 跑过）。

---

### Phase G · push 域结构骨架（依赖 A,B；实推待证书）

> **域归属：** 新建 `sanyan-push-api`（Phase A3 已建：PushApi / PushChannel / dto）+ `sanyan-push-core`（Phase A4 已建脚手架）。本 Phase 填充 push-core 实现。
> **前置：** Phase A 已建两模块 pom（含 pushy 0.15.4 依赖）；Phase B 已建 V11 `device_tokens` migration（bootstrap 主拷贝 + push-core `src/test/resources/db/migration/` 全量副本 V1-V11 + schema IT）。
> **本期边界（spec §8）：** 只搭结构——device_tokens 实体/仓储/注册接口 + PushChannel/PushRouter + ApnsPushChannel 骨架。**APNs 实推等 .p8 证书**，本期 sendToDevice 返回 `PushResult.pending(...)` 不真发。
> **错误码（附录 F）：** PushErrCode 8000-8999：`DEVICE_TOKEN_INVALID(8001)` / `PUSH_SEND_FAILED(8002)`。
> **契约（附录 A/B，严格照用）：** `DeviceTokenEntity`（device_tokens 表，唯一约束 user_id+platform+vendor+token）、`PushApi`（registerDevice / pushToUser / listActiveTokens）、`PushChannel`（vendor / supports / sendToDevice）、`DeviceTokenDto`、`PushPayload`、`PushResult`（pending/sent/failed 工厂）。
> **parent pom 坐标：** `com.sanyan:sanyan-server-parent:0.1.0`（实现者以此为准，勿用骨架早期写的 1.0.0-SNAPSHOT）。
> **postgresql scope：** runtime（照 character-core pom）。

---

#### Task G1: DeviceTokenEntity + DeviceTokenRepository + DeviceTokenTestFixtures

**Files:**
- Create: `server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/DeviceTokenEntity.java`
- Create: `server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/DeviceTokenRepository.java`
- Create: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/fixtures/DeviceTokenTestFixtures.java`
- Test: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/DeviceTokenRepositoryIT.java`

> 依赖 Phase B 的 V11 `device_tokens` migration 测试副本（`sanyan-push-core/src/test/resources/db/migration/V11__create_device_tokens.sql` + V1-V10 全量副本）。device_tokens 无外键，IT 直接存取。

- [ ] **Step 1: 写失败测试（仓储 IT，照 RelationshipRepositoryIT 模板，basePackages 换 com.sanyan.push）**

```java
package com.sanyan.push.internal;

import com.sanyan.push.internal.fixtures.DeviceTokenTestFixtures;
import com.sanyan.common.test.PostgresTestcontainerSupport;
import com.sanyan.common.test.TestApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase.Replace.NONE;

/**
 * G1：DeviceTokenEntity + Repository roundtrip + 自定义查询验证。
 * Testcontainers PG，schema 由 Flyway V1-V11 生成，ddl-auto=none。device_tokens 无外键。
 */
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@ContextConfiguration(classes = TestApplication.class)
@EntityScan(basePackages = "com.sanyan.push")
@EnableJpaRepositories(basePackages = "com.sanyan.push")
class DeviceTokenRepositoryIT extends PostgresTestcontainerSupport {

    @DynamicPropertySource
    static void pgProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", PostgresTestcontainerSupport::jdbcUrl);
        registry.add("spring.datasource.username", PostgresTestcontainerSupport::username);
        registry.add("spring.datasource.password", PostgresTestcontainerSupport::password);
        registry.add("spring.datasource.driver-class-name", () -> "org.postgresql.Driver");
        registry.add("spring.flyway.locations", () -> "classpath:db/migration");
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "none");
    }

    @Autowired
    DeviceTokenRepository repo;

    @Autowired
    TestEntityManager em;

    @Test
    void save_and_reload_should_roundtrip_with_defaults() {
        DeviceTokenEntity token = DeviceTokenTestFixtures.iosApns(7L, "tok-abc");
        Long id = repo.save(token).getId();
        em.flush();
        em.clear();

        DeviceTokenEntity loaded = repo.findById(id).orElseThrow();
        assertThat(loaded.getUserId()).isEqualTo(7L);
        assertThat(loaded.getPlatform()).isEqualTo("ios");
        assertThat(loaded.getVendor()).isEqualTo("apns");
        assertThat(loaded.getToken()).isEqualTo("tok-abc");
        assertThat(loaded.getActive()).isTrue();
        assertThat(loaded.getRegisteredAt()).isNotNull();
    }

    @Test
    void findByUserIdAndActiveTrue_should_return_only_active() {
        repo.save(DeviceTokenTestFixtures.iosApns(8L, "active-1"));
        DeviceTokenEntity inactive = DeviceTokenTestFixtures.iosApns(8L, "inactive-1");
        inactive.setActive(false);
        repo.save(inactive);
        em.flush();
        em.clear();

        List<DeviceTokenEntity> active = repo.findByUserIdAndActiveTrue(8L);

        assertThat(active).hasSize(1);
        assertThat(active.get(0).getToken()).isEqualTo("active-1");
    }

    @Test
    void findByUserIdAndPlatformAndVendorAndToken_should_locate_existing_row() {
        repo.save(DeviceTokenTestFixtures.iosApns(9L, "tok-xyz"));
        em.flush();
        em.clear();

        Optional<DeviceTokenEntity> found =
                repo.findByUserIdAndPlatformAndVendorAndToken(9L, "ios", "apns", "tok-xyz");

        assertThat(found).isPresent();
        assertThat(found.get().getUserId()).isEqualTo(9L);
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=DeviceTokenRepositoryIT test`
Expected: 编译失败（实体/仓储/Fixture 不存在）。

- [ ] **Step 3: 写最小实现**

```java
// DeviceTokenEntity.java
package com.sanyan.push.internal;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;

import java.time.Instant;

/**
 * 设备推送 token（V11 device_tokens 表，Plan 4）。
 *
 * <p>客户端通过 {@code POST /api/devices/register} 上报；{@link com.sanyan.push.internal.PushRouter}
 * 投递离线推送时按 platform/vendor 路由到对应 {@link com.sanyan.push.PushChannel}。
 *
 * <p>无外键（spec §3.1）。唯一约束 (user_id, platform, vendor, token)：同一设备 token 重复注册
 * 走"已存在则置 active+刷新 last_seen"（{@link com.sanyan.push.api.PushApiImpl#registerDevice}）。
 */
@Data
@Entity
@Table(name = "device_tokens",
        uniqueConstraints = @UniqueConstraint(
                columnNames = {"user_id", "platform", "vendor", "token"}))
public class DeviceTokenEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(nullable = false, length = 20)
    private String platform;   // ios / android

    @Column(nullable = false, length = 20)
    private String vendor;     // apns / getui / jpush / ...

    @Column(nullable = false, length = 500)
    private String token;

    @Column(nullable = false)
    private Boolean active = true;

    @CreationTimestamp
    @Column(name = "registered_at", nullable = false, updatable = false)
    private Instant registeredAt;

    @Column(name = "last_seen")
    private Instant lastSeen;
}
```

```java
// DeviceTokenRepository.java
package com.sanyan.push.internal;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;

/**
 * device_tokens 仓储。
 * <ul>
 *   <li>{@code findByUserIdAndActiveTrue}：PushRouter 取用户所有 active 设备</li>
 *   <li>{@code findByUserIdAndPlatformAndVendorAndToken}：注册时按唯一键找已有行（幂等注册）</li>
 * </ul>
 */
public interface DeviceTokenRepository extends JpaRepository<DeviceTokenEntity, Long> {

    List<DeviceTokenEntity> findByUserIdAndActiveTrue(Long userId);

    Optional<DeviceTokenEntity> findByUserIdAndPlatformAndVendorAndToken(
            Long userId, String platform, String vendor, String token);
}
```

```java
// DeviceTokenTestFixtures.java
package com.sanyan.push.internal.fixtures;

import com.sanyan.push.internal.DeviceTokenEntity;

/**
 * DeviceTokenEntity Object Mother（java-backend-business-layer.md §5.2）。
 */
public final class DeviceTokenTestFixtures {

    private DeviceTokenTestFixtures() {}

    public static DeviceTokenEntity iosApns(Long userId, String token) {
        return build(userId, "ios", "apns", token);
    }

    public static DeviceTokenEntity androidGetui(Long userId, String token) {
        return build(userId, "android", "getui", token);
    }

    private static DeviceTokenEntity build(Long userId, String platform, String vendor, String token) {
        DeviceTokenEntity e = new DeviceTokenEntity();
        e.setUserId(userId);
        e.setPlatform(platform);
        e.setVendor(vendor);
        e.setToken(token);
        e.setActive(true);
        return e;
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=DeviceTokenRepositoryIT test`
Expected: BUILD SUCCESS，3 个测试通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/DeviceTokenEntity.java \
        server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/DeviceTokenRepository.java \
        server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/
git commit -m "feat(push-core): DeviceTokenEntity + Repository + Fixture（设备 token 持久化）"
```

---

#### Task G2: PushErrCode（8000-8999）

**Files:**
- Create: `server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/PushErrCode.java`
- Modify: `server/foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`
- Test: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/PushErrCodeTest.java`

> ErrCode enum，照 `CharacterErrCode`（@Getter @AllArgsConstructor implements ErrCode）。错误码区间 8000-8999（spec §9）。本 task 含轻量单测守护 code 值与区间（防误改），并同步 ERROR_CODE_REGISTRY.md。

- [ ] **Step 1: 写失败测试**

```java
package com.sanyan.push.internal;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class PushErrCodeTest {

    @Test
    void codes_should_match_appendix_values() {
        assertThat(PushErrCode.DEVICE_TOKEN_INVALID.getCode()).isEqualTo(8001);
        assertThat(PushErrCode.PUSH_SEND_FAILED.getCode()).isEqualTo(8002);
    }

    @Test
    void all_codes_should_be_within_push_range() {
        for (PushErrCode c : PushErrCode.values()) {
            assertThat(c.getCode()).isBetween(8000, 8999);
        }
    }

    @Test
    void messages_should_not_be_blank() {
        for (PushErrCode c : PushErrCode.values()) {
            assertThat(c.getDefaultMessage()).isNotBlank();
        }
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=PushErrCodeTest test`
Expected: 编译失败（`PushErrCode` 不存在）。

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.push.internal;

import com.sanyan.common.error.ErrCode;
import lombok.AllArgsConstructor;
import lombok.Getter;

/**
 * Push 模块错误码 enum（Plan 4 引入）。
 *
 * <p>code 区间 <b>8000-8999</b>，登记在 {@code sanyan-common-error/ERROR_CODE_REGISTRY.md}。
 * 启动时由 {@code ErrCodeConflictDetector} 扫描；同 code 重复定义会让上下文启动失败。
 */
@Getter
@AllArgsConstructor
public enum PushErrCode implements ErrCode {

    DEVICE_TOKEN_INVALID(8001, "设备 token 无效"),
    PUSH_SEND_FAILED(8002, "推送发送失败"),
    ;

    private final int code;
    private final String defaultMessage;
}
```

修改 `ERROR_CODE_REGISTRY.md`：区间总览加 `8000-8999 push（PushErrCode）`；明细新增表
```
| 8001 | DEVICE_TOKEN_INVALID | 设备 token 无效 |
| 8002 | PUSH_SEND_FAILED | 推送发送失败 |
```
历史变更附 `| 2026-05-27 | Plan 4：push 域新增 8001 / 8002 |`。

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=PushErrCodeTest test`
Expected: BUILD SUCCESS，3 个测试通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/PushErrCode.java \
        server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/PushErrCodeTest.java \
        server/foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md
git commit -m "feat(push-core): PushErrCode 8001/8002 + ERROR_CODE_REGISTRY 同步"
```

---

#### Task G3: PushApiImpl 实现 registerDevice + listActiveTokens

**Files:**
- Create: `server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/api/PushApiImpl.java`
- Test: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/api/PushApiImplTest.java`

> 实现 `PushApi` 的 registerDevice + listActiveTokens（`pushToUser` 留待 G7 委托 PushRouter，本 task 先抛 UnsupportedOperationException 占位，G7 替换）。registerDevice 幂等：按唯一键找已有行 → 存在则置 active=true + 刷新 last_seen；不存在则新建。listActiveTokens → 查 active 行映射成 `DeviceTokenDto`。Mockito mock DeviceTokenRepository。

- [ ] **Step 1: 写失败测试**

```java
package com.sanyan.push.api;

import com.sanyan.push.dto.DeviceTokenDto;
import com.sanyan.push.internal.DeviceTokenEntity;
import com.sanyan.push.internal.DeviceTokenRepository;
import com.sanyan.push.internal.fixtures.DeviceTokenTestFixtures;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class PushApiImplTest {

    @Mock DeviceTokenRepository repository;
    @InjectMocks PushApiImpl api;

    @Test
    void registerDevice_should_create_new_when_not_exists() {
        when(repository.findByUserIdAndPlatformAndVendorAndToken(7L, "ios", "apns", "tok-1"))
                .thenReturn(Optional.empty());

        api.registerDevice(7L, "ios", "apns", "tok-1");

        ArgumentCaptor<DeviceTokenEntity> captor = ArgumentCaptor.forClass(DeviceTokenEntity.class);
        verify(repository).save(captor.capture());
        DeviceTokenEntity saved = captor.getValue();
        assertThat(saved.getUserId()).isEqualTo(7L);
        assertThat(saved.getPlatform()).isEqualTo("ios");
        assertThat(saved.getVendor()).isEqualTo("apns");
        assertThat(saved.getToken()).isEqualTo("tok-1");
        assertThat(saved.getActive()).isTrue();
        assertThat(saved.getLastSeen()).isNotNull();
    }

    @Test
    void registerDevice_should_reactivate_existing_and_refresh_last_seen() {
        DeviceTokenEntity existing = DeviceTokenTestFixtures.iosApns(7L, "tok-1");
        existing.setId(42L);
        existing.setActive(false);
        when(repository.findByUserIdAndPlatformAndVendorAndToken(7L, "ios", "apns", "tok-1"))
                .thenReturn(Optional.of(existing));

        api.registerDevice(7L, "ios", "apns", "tok-1");

        ArgumentCaptor<DeviceTokenEntity> captor = ArgumentCaptor.forClass(DeviceTokenEntity.class);
        verify(repository).save(captor.capture());
        DeviceTokenEntity saved = captor.getValue();
        assertThat(saved.getId()).isEqualTo(42L);          // 复用同一行，不新建
        assertThat(saved.getActive()).isTrue();
        assertThat(saved.getLastSeen()).isNotNull();
    }

    @Test
    void listActiveTokens_should_map_entities_to_dto() {
        DeviceTokenEntity a = DeviceTokenTestFixtures.iosApns(7L, "tok-a");
        DeviceTokenEntity b = DeviceTokenTestFixtures.androidGetui(7L, "tok-b");
        when(repository.findByUserIdAndActiveTrue(7L)).thenReturn(List.of(a, b));

        List<DeviceTokenDto> dtos = api.listActiveTokens(7L);

        assertThat(dtos).hasSize(2);
        assertThat(dtos).extracting(DeviceTokenDto::userId).containsOnly(7L);
        assertThat(dtos).extracting(DeviceTokenDto::vendor).containsExactlyInAnyOrder("apns", "getui");
        assertThat(dtos).extracting(DeviceTokenDto::token).containsExactlyInAnyOrder("tok-a", "tok-b");
    }

    @Test
    void listActiveTokens_should_return_empty_when_none() {
        when(repository.findByUserIdAndActiveTrue(7L)).thenReturn(List.of());
        assertThat(api.listActiveTokens(7L)).isEmpty();
        verify(repository, never()).save(any());
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=PushApiImplTest test`
Expected: 编译失败（`PushApiImpl` 不存在）。

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.push.api;

import com.sanyan.push.PushApi;
import com.sanyan.push.dto.DeviceTokenDto;
import com.sanyan.push.dto.PushPayload;
import com.sanyan.push.dto.PushResult;
import com.sanyan.push.internal.DeviceTokenEntity;
import com.sanyan.push.internal.DeviceTokenRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;

/**
 * {@link PushApi} 在 push-core 的实现。
 *
 * <p>本期（spec §8）只实现设备注册 + 列举 active token；{@code pushToUser} 在 G7 委托
 * {@link com.sanyan.push.internal.PushRouter}（本 task 先占位）。
 *
 * <p>registerDevice 幂等：按唯一键 (user, platform, vendor, token) 找已有行——存在则置
 * active=true + 刷新 last_seen（设备重装/重登复用同一行），不存在则新建。
 */
@Service
@RequiredArgsConstructor
public class PushApiImpl implements PushApi {

    private final DeviceTokenRepository repository;

    @Override
    @Transactional
    public void registerDevice(Long userId, String platform, String vendor, String token) {
        DeviceTokenEntity entity = repository
                .findByUserIdAndPlatformAndVendorAndToken(userId, platform, vendor, token)
                .orElseGet(() -> {
                    DeviceTokenEntity fresh = new DeviceTokenEntity();
                    fresh.setUserId(userId);
                    fresh.setPlatform(platform);
                    fresh.setVendor(vendor);
                    fresh.setToken(token);
                    return fresh;
                });
        entity.setActive(true);
        entity.setLastSeen(Instant.now());
        repository.save(entity);
    }

    @Override
    public PushResult pushToUser(Long userId, PushPayload payload) {
        // G7 替换为委托 PushRouter
        throw new UnsupportedOperationException("pushToUser 由 G7 接入 PushRouter");
    }

    @Override
    public List<DeviceTokenDto> listActiveTokens(Long userId) {
        return repository.findByUserIdAndActiveTrue(userId).stream()
                .map(e -> new DeviceTokenDto(e.getUserId(), e.getPlatform(), e.getVendor(), e.getToken()))
                .toList();
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=PushApiImplTest test`
Expected: BUILD SUCCESS，4 个测试通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/api/PushApiImpl.java \
        server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/api/PushApiImplTest.java
git commit -m "feat(push-core): PushApiImpl 设备注册（幂等）+ 列举 active token"
```

---

#### Task G4: DeviceTokenController（POST /api/devices/register）

**Files:**
- Create: `server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/web/DeviceTokenController.java`（含 `RegisterDeviceReq` 同文件）
- Test: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/web/DeviceTokenControllerIT.java`

> Controller 薄壳，照 `AuthController`（POST @RequestBody + @Valid）+ `RelationshipController`（@LoginUser Long userId + 手动 BaseResp.success）。`RegisterDeviceReq` 与 Controller 同文件（java-backend §3.2：单接口请求 DTO 同文件）。@WebMvcTest 照 RelationshipControllerIT（mock JwtUtil 让 test-token → userId=42）。
> push-core pom 需含 `sanyan-common-web`（BaseResp）+ `sanyan-common-auth`（@LoginUser）+ `spring-boot-starter-web` + `spring-boot-starter-validation`（@Valid）—— Phase A4 建 pom 时核对补齐（character-core pom 有 web/auth；validation 由 starter-web 间接带或显式加，实现者确认）。

- [ ] **Step 1: 写失败测试（@WebMvcTest）**

```java
package com.sanyan.push.web;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.common.auth.JwtUtil;
import com.sanyan.common.auth.LoginUserArgumentResolver;
import com.sanyan.common.auth.WebMvcConfig;
import com.sanyan.common.test.TestApplication;
import com.sanyan.common.web.GlobalExceptionHandler;
import com.sanyan.push.PushApi;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.web.servlet.MockMvc;

import java.util.Map;

import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(controllers = DeviceTokenController.class)
@ContextConfiguration(classes = TestApplication.class)
@Import({DeviceTokenController.class, WebMvcConfig.class, LoginUserArgumentResolver.class, GlobalExceptionHandler.class})
class DeviceTokenControllerIT {

    @Autowired
    private MockMvc mockMvc;

    private final ObjectMapper objectMapper = new ObjectMapper();

    @MockBean
    private PushApi pushApi;

    @MockBean
    private JwtUtil jwtUtil;

    @Test
    void register_should_call_pushApi_and_return_success() throws Exception {
        when(jwtUtil.parseUserId("test-token")).thenReturn(42L);
        String body = objectMapper.writeValueAsString(
                Map.of("platform", "ios", "vendor", "apns", "token", "device-tok-123"));

        mockMvc.perform(post("/api/devices/register")
                        .header("Authorization", "Bearer test-token")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(body))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data").doesNotExist());

        verify(pushApi).registerDevice(42L, "ios", "apns", "device-tok-123");
    }

    @Test
    void register_should_reject_blank_token() throws Exception {
        when(jwtUtil.parseUserId("test-token")).thenReturn(42L);
        String body = objectMapper.writeValueAsString(
                Map.of("platform", "ios", "vendor", "apns", "token", ""));

        mockMvc.perform(post("/api/devices/register")
                        .header("Authorization", "Bearer test-token")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(body))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(false));
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=DeviceTokenControllerIT test`
Expected: 编译失败（`DeviceTokenController` / `RegisterDeviceReq` 不存在）。

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.push.web;

import com.sanyan.common.auth.LoginUser;
import com.sanyan.common.web.BaseResp;
import com.sanyan.push.PushApi;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * 设备 token 注册接口（Plan 4，spec §8）。客户端拿到 APNs / 厂商 token 后上报，
 * 供离线推送（L3）路由。本期接口可用，实推待证书（{@code ApnsPushChannel} 占位）。
 */
@RestController
@RequestMapping("/api/devices")
@RequiredArgsConstructor
public class DeviceTokenController {

    private final PushApi pushApi;

    @PostMapping("/register")
    public BaseResp<Void> register(@LoginUser Long userId, @Valid @RequestBody RegisterDeviceReq req) {
        pushApi.registerDevice(userId, req.getPlatform(), req.getVendor(), req.getToken());
        return BaseResp.success(null);
    }

    /** 注册设备请求体（单接口专用，与 Controller 同文件，java-backend §3.2）。 */
    @Data
    public static class RegisterDeviceReq {
        @NotBlank
        private String platform;   // ios / android
        @NotBlank
        private String vendor;     // apns / getui / ...
        @NotBlank
        private String token;
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=DeviceTokenControllerIT test`
Expected: BUILD SUCCESS，2 个测试通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/web/DeviceTokenController.java \
        server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/web/DeviceTokenControllerIT.java
git commit -m "feat(push-core): DeviceTokenController POST /api/devices/register"
```

---

#### Task G5: ApnsPushChannel（骨架，实推待证书）+ ApnsProperties

**Files:**
- Create: `server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/apns/ApnsProperties.java`
- Create: `server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/apns/ApnsPushChannel.java`
- Test: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/apns/ApnsPushChannelTest.java`

> implements `PushChannel`：`vendor()="apns"`、`supports(p)=="ios".equals(p)`、`sendToDevice` 本期返回 `PushResult.pending("APNs 实推待证书接入")`（不真发，不引入真实 pushy 客户端连接——避免无证书时启动/调用失败）。`@ConfigurationProperties("sanyan.push.apns")` 读 p8/keyId/teamId/topic 占位字段（配置类，TDD 例外，但配置绑定有可测的字段读取，单测验证 vendor/supports/pending + 配置类 getter/setter）。
> pushy 0.15.4 已在 Phase A4 pom 引入；本 task 骨架**先不实例化 `ApnsClient`**（待证书），仅保留依赖在 classpath，注释记待办。

- [ ] **Step 1: 写失败测试**

```java
package com.sanyan.push.internal.apns;

import com.sanyan.push.dto.DeviceTokenDto;
import com.sanyan.push.dto.PushPayload;
import com.sanyan.push.dto.PushResult;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ApnsPushChannelTest {

    private final ApnsPushChannel channel = new ApnsPushChannel(new ApnsProperties());

    @Test
    void vendor_should_be_apns() {
        assertThat(channel.vendor()).isEqualTo("apns");
    }

    @Test
    void supports_should_be_true_only_for_ios() {
        assertThat(channel.supports("ios")).isTrue();
        assertThat(channel.supports("android")).isFalse();
        assertThat(channel.supports(null)).isFalse();
    }

    @Test
    void sendToDevice_should_return_pending_placeholder_this_phase() {
        DeviceTokenDto token = new DeviceTokenDto(7L, "ios", "apns", "tok-1");
        PushPayload payload = new PushPayload("小婉", "在想你哦", 100L);

        PushResult result = channel.sendToDevice(token, payload);

        assertThat(result.status()).isEqualTo("PENDING");
        assertThat(result.detail()).contains("证书");
    }

    @Test
    void properties_should_bind_fields() {
        ApnsProperties props = new ApnsProperties();
        props.setP8("AuthKey_ABC.p8");
        props.setKeyId("KEY123");
        props.setTeamId("TEAM456");
        props.setTopic("com.sanyan.app");

        assertThat(props.getP8()).isEqualTo("AuthKey_ABC.p8");
        assertThat(props.getKeyId()).isEqualTo("KEY123");
        assertThat(props.getTeamId()).isEqualTo("TEAM456");
        assertThat(props.getTopic()).isEqualTo("com.sanyan.app");
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=ApnsPushChannelTest test`
Expected: 编译失败（`ApnsPushChannel` / `ApnsProperties` 不存在）。

- [ ] **Step 3: 写最小实现**

```java
// ApnsProperties.java
package com.sanyan.push.internal.apns;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

/**
 * APNs 接入配置（spec §8）：基于 token-based auth（.p8 AuthKey）。
 * 本期占位——字段读取就绪，实推待证书下发后接入 pushy {@code ApnsClient}。
 *
 * <pre>
 * sanyan:
 *   push:
 *     apns:
 *       p8: classpath:AuthKey_XXXX.p8   # AuthKey 文件路径
 *       keyId: XXXXXXXXXX               # Key ID
 *       teamId: YYYYYYYYYY              # Apple Team ID
 *       topic: com.sanyan.app           # bundle id
 * </pre>
 */
@Data
@Component
@ConfigurationProperties(prefix = "sanyan.push.apns")
public class ApnsProperties {
    private String p8;
    private String keyId;
    private String teamId;
    private String topic;
}
```

```java
// ApnsPushChannel.java
package com.sanyan.push.internal.apns;

import com.sanyan.push.PushChannel;
import com.sanyan.push.dto.DeviceTokenDto;
import com.sanyan.push.dto.PushPayload;
import com.sanyan.push.dto.PushResult;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

/**
 * APNs（iOS）推送通道骨架（spec §8）。
 *
 * <p><b>本期边界：</b>只搭结构——{@link #sendToDevice} 返回 {@link PushResult#pending}
 * 占位，<b>不真发</b>。理由：APNs token-based auth 需要 Apple 下发的 {@code .p8} AuthKey，
 * 证书未到位前实例化 pushy {@code ApnsClient} 会失败。离线推送本期靠 L4 启动拉取兜底
 * （spec §7.1），不阻塞主流程。
 *
 * <p><b>实推待办（证书到位后）：</b>注入 {@link ApnsProperties} 构建 pushy
 * {@code ApnsClient}（{@code com.eatthepath:pushy} 0.15.4，已在 pom），
 * {@code sendToDevice} 改为真发 {@code SimpleApnsPushNotification} 并把投递结果映射为
 * {@link PushResult#sent()} / {@link PushResult#failed(String)}。
 *
 * <p>Spring 自动收集所有 {@link PushChannel} Bean 注入 {@link com.sanyan.push.internal.PushRouter}。
 */
@Component
@RequiredArgsConstructor
public class ApnsPushChannel implements PushChannel {

    /** 本期占位文案；实推接入后删除此分支。 */
    static final String PENDING_DETAIL = "APNs 实推待证书接入";

    static final String VENDOR_APNS = "apns";
    static final String PLATFORM_IOS = "ios";

    private final ApnsProperties properties;

    @Override
    public String vendor() {
        return VENDOR_APNS;
    }

    @Override
    public boolean supports(String platform) {
        return PLATFORM_IOS.equals(platform);
    }

    @Override
    public PushResult sendToDevice(DeviceTokenDto token, PushPayload payload) {
        // 本期不真发——证书到位后用 properties 构建 ApnsClient 并真正投递
        return PushResult.pending(PENDING_DETAIL);
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=ApnsPushChannelTest test`
Expected: BUILD SUCCESS，4 个测试通过。

> push-core pom 需含 `spring-boot-starter`（@ConfigurationProperties / @Component）—— 由 starter-web/data-jpa 间接带；若 `@ConfigurationProperties` 绑定需要 `spring-boot-configuration-processor`（可选，仅生成元数据），不强制。实现者确认 ApnsProperties 能被上下文加载（G7 / 端到端 IT 会验证）。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/apns/ \
        server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/apns/
git commit -m "feat(push-core): ApnsPushChannel 骨架 + ApnsProperties（实推待证书占位）"
```

---

#### Task G6: PushRouter（注入 List<PushChannel>，pushToUser 汇总投递）

**Files:**
- Create: `server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/PushRouter.java`
- Test: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/PushRouterTest.java`

> 注入 `List<PushChannel>`（Spring 自动收集所有通道 Bean）+ `DeviceTokenRepository`。`pushToUser(userId, payload)`：查 active token → 每 token 找第一个 `supports(platform)` 的 channel → `sendToDevice` → 汇总成单个 `PushResult`。汇总规则：无 active token → `PushResult.pending("无可推送设备")`；任一 channel 返回 SENT → 整体 SENT；全 PENDING → PENDING；出现 FAILED 且无 SENT → FAILED。Mockito mock PushChannel + DeviceTokenRepository。

- [ ] **Step 1: 写失败测试**

```java
package com.sanyan.push.internal;

import com.sanyan.push.PushChannel;
import com.sanyan.push.dto.DeviceTokenDto;
import com.sanyan.push.dto.PushPayload;
import com.sanyan.push.dto.PushResult;
import com.sanyan.push.internal.fixtures.DeviceTokenTestFixtures;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.lenient;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class PushRouterTest {

    @Mock DeviceTokenRepository repository;
    @Mock PushChannel apnsChannel;

    private final PushPayload payload = new PushPayload("小婉", "在想你", 1L);

    private PushRouter router() {
        return new PushRouter(List.of(apnsChannel), repository);
    }

    @Test
    void pushToUser_should_route_ios_token_to_supporting_channel() {
        when(repository.findByUserIdAndActiveTrue(7L))
                .thenReturn(List.of(DeviceTokenTestFixtures.iosApns(7L, "tok-1")));
        when(apnsChannel.supports("ios")).thenReturn(true);
        when(apnsChannel.sendToDevice(any(DeviceTokenDto.class), any())).thenReturn(PushResult.sent());

        PushResult result = router().pushToUser(7L, payload);

        assertThat(result.status()).isEqualTo("SENT");
        verify(apnsChannel).sendToDevice(any(DeviceTokenDto.class), any());
    }

    @Test
    void pushToUser_should_return_pending_when_no_active_token() {
        when(repository.findByUserIdAndActiveTrue(7L)).thenReturn(List.of());

        PushResult result = router().pushToUser(7L, payload);

        assertThat(result.status()).isEqualTo("PENDING");
        verify(apnsChannel, never()).sendToDevice(any(), any());
    }

    @Test
    void pushToUser_should_skip_token_with_no_supporting_channel() {
        when(repository.findByUserIdAndActiveTrue(7L))
                .thenReturn(List.of(DeviceTokenTestFixtures.androidGetui(7L, "tok-android")));
        when(apnsChannel.supports("android")).thenReturn(false);

        PushResult result = router().pushToUser(7L, payload);

        // 无通道能处理 android token → 整体 pending（本期 android 通道未实现）
        assertThat(result.status()).isEqualTo("PENDING");
        verify(apnsChannel, never()).sendToDevice(any(), any());
    }

    @Test
    void pushToUser_should_aggregate_sent_over_pending() {
        when(repository.findByUserIdAndActiveTrue(7L)).thenReturn(List.of(
                DeviceTokenTestFixtures.iosApns(7L, "tok-1"),
                DeviceTokenTestFixtures.iosApns(7L, "tok-2")));
        when(apnsChannel.supports("ios")).thenReturn(true);
        when(apnsChannel.sendToDevice(any(DeviceTokenDto.class), any()))
                .thenReturn(PushResult.pending("待证书"))
                .thenReturn(PushResult.sent());

        PushResult result = router().pushToUser(7L, payload);

        // 一条 SENT 即整体 SENT
        assertThat(result.status()).isEqualTo("SENT");
    }

    @Test
    void pushToUser_should_return_failed_when_only_failures() {
        when(repository.findByUserIdAndActiveTrue(7L))
                .thenReturn(List.of(DeviceTokenTestFixtures.iosApns(7L, "tok-1")));
        when(apnsChannel.supports("ios")).thenReturn(true);
        when(apnsChannel.sendToDevice(any(DeviceTokenDto.class), any()))
                .thenReturn(PushResult.failed("BadDeviceToken"));

        PushResult result = router().pushToUser(7L, payload);

        assertThat(result.status()).isEqualTo("FAILED");
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=PushRouterTest test`
Expected: 编译失败（`PushRouter` 不存在）。

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.push.internal;

import com.sanyan.push.PushChannel;
import com.sanyan.push.dto.DeviceTokenDto;
import com.sanyan.push.dto.PushPayload;
import com.sanyan.push.dto.PushResult;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * 推送路由（spec §8）：按设备 platform 把推送分发到对应 {@link PushChannel}，汇总投递结果。
 *
 * <p>Spring 自动注入所有 {@link PushChannel} Bean（本期仅 {@code ApnsPushChannel}）。
 * 用户多设备时所有 active token 逐一尝试，找第一个 {@code supports(platform)} 的 channel 投递。
 *
 * <p>汇总规则（多设备多结果归一为单个 {@link PushResult}）：
 * <ul>
 *   <li>无 active token → PENDING（"无可推送设备"）</li>
 *   <li>任一设备 SENT → 整体 SENT</li>
 *   <li>无 SENT 但有 FAILED → 整体 FAILED（detail 取首个失败原因）</li>
 *   <li>其余（全 PENDING / 无可处理通道）→ PENDING</li>
 * </ul>
 *
 * <p>本期 sendToDevice 多为 PENDING（ApnsPushChannel 占位），离线推送靠 L4 启动拉取兜底。
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class PushRouter {

    static final String NO_DEVICE_DETAIL = "无可推送设备";

    private final List<PushChannel> channels;
    private final DeviceTokenRepository repository;

    public PushResult pushToUser(Long userId, PushPayload payload) {
        List<DeviceTokenEntity> activeTokens = repository.findByUserIdAndActiveTrue(userId);
        if (activeTokens.isEmpty()) {
            return PushResult.pending(NO_DEVICE_DETAIL);
        }

        boolean anySent = false;
        String firstFailure = null;
        for (DeviceTokenEntity tokenEntity : activeTokens) {
            PushChannel channel = findChannel(tokenEntity.getPlatform());
            if (channel == null) {
                log.warn("无可处理 platform={} 的推送通道，跳过 token id={}",
                        tokenEntity.getPlatform(), tokenEntity.getId());
                continue;
            }
            DeviceTokenDto dto = new DeviceTokenDto(
                    tokenEntity.getUserId(), tokenEntity.getPlatform(),
                    tokenEntity.getVendor(), tokenEntity.getToken());
            PushResult result = channel.sendToDevice(dto, payload);
            if ("SENT".equals(result.status())) {
                anySent = true;
            } else if ("FAILED".equals(result.status()) && firstFailure == null) {
                firstFailure = result.detail();
            }
        }

        if (anySent) {
            return PushResult.sent();
        }
        if (firstFailure != null) {
            return PushResult.failed(firstFailure);
        }
        return PushResult.pending(NO_DEVICE_DETAIL);
    }

    private PushChannel findChannel(String platform) {
        return channels.stream()
                .filter(c -> c.supports(platform))
                .findFirst()
                .orElse(null);
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=PushRouterTest test`
Expected: BUILD SUCCESS，5 个测试通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/PushRouter.java \
        server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/internal/PushRouterTest.java
git commit -m "feat(push-core): PushRouter 多设备分发 + 结果汇总"
```

---

#### Task G7: PushApiImpl.pushToUser 委托 PushRouter

**Files:**
- Modify: `server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/api/PushApiImpl.java`
- Modify: `server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/api/PushApiImplTest.java`

> G3 占位的 `pushToUser` 替换为委托 `PushRouter`。给 PushApiImpl 注入 PushRouter。补单测验证委托。

- [ ] **Step 1: 写失败测试（在 PushApiImplTest 加 PushRouter mock + 委托用例）**

在 `PushApiImplTest` 中：字段加 `@Mock PushRouter pushRouter;`（`@InjectMocks PushApiImpl api;` 不变，Mockito 会注入两个依赖），新增用例：

```java
    @Test
    void pushToUser_should_delegate_to_router() {
        com.sanyan.push.dto.PushPayload payload =
                new com.sanyan.push.dto.PushPayload("小婉", "在想你", 100L);
        com.sanyan.push.dto.PushResult expected = com.sanyan.push.dto.PushResult.sent();
        when(pushRouter.pushToUser(7L, payload)).thenReturn(expected);

        com.sanyan.push.dto.PushResult result = api.pushToUser(7L, payload);

        assertThat(result).isEqualTo(expected);
        verify(pushRouter).pushToUser(7L, payload);
    }
```

> 加 import：`import com.sanyan.push.internal.PushRouter;`。注意：原 G3 的 `registerDevice` / `listActiveTokens` 用例只用到 `repository` mock；新增 PushRouter mock 后这些用例不受影响（Mockito 注入多依赖）。

- [ ] **Step 2: 运行测试确认失败**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=PushApiImplTest test`
Expected: `pushToUser_should_delegate_to_router` 失败（当前实现抛 UnsupportedOperationException），编译需先加 PushRouter 字段——按 TDD 先让测试编译失败再改实现。

- [ ] **Step 3: 写最小实现（替换 pushToUser，注入 PushRouter）**

```java
@Service
@RequiredArgsConstructor
public class PushApiImpl implements PushApi {

    private final DeviceTokenRepository repository;
    private final PushRouter pushRouter;   // G7 新增

    // ... registerDevice / listActiveTokens 不变 ...

    @Override
    public PushResult pushToUser(Long userId, PushPayload payload) {
        return pushRouter.pushToUser(userId, payload);
    }
}
```

> 加 import：`import com.sanyan.push.internal.PushRouter;`。

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server && mvn -pl business_packages/sanyan-push-core -Dtest=PushApiImplTest test`
Expected: BUILD SUCCESS，5 个测试全通过（含新增委托用例）。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-push-core/src/main/java/com/sanyan/push/api/PushApiImpl.java \
        server/business_packages/sanyan-push-core/src/test/java/com/sanyan/push/api/PushApiImplTest.java
git commit -m "feat(push-core): PushApiImpl.pushToUser 委托 PushRouter"
```

---

> **Phase G checkpoint：** 跑 push-core 全包测试：
> `cd server && mvn -pl business_packages/sanyan-push-core test`
> 期望全绿（含 DeviceTokenRepositoryIT 需 Testcontainers PG）。push 域为新建独立模块，未触碰 foundation 层与其他业务域，按测试粒度规范跑本包即可。Phase A/B 的脚手架编译 + V11 schema IT 已在对应 Phase 跑过。
### Phase H · chat 域 4 层投递 + WS ACK（依赖 E, G）

> **依赖：** Phase E（`SessionManager` 心跳 TTL + 统一 `isOnline`，但本 Phase 投递 L1 用 `getSession()`）、Phase G（`sanyan-push-api` 已建，`PushApi.pushToUser(Long, PushPayload)` 返回 `PushResult`、`PushPayload(String title, String body, Long messageId)` 已就绪）。
> **现状关键点（实现者务必照抄，勿臆测）：**
> - `MessageService`（chat-core/internal）**只有** `saveUserMessage(Long, String)` 与 `handleAiReply(Long)`（内部调 LLM）+ `toData(MessageEntity)`，**没有**「落单条 ai message」方法 → 本 Phase H0 先补 `saveAiMessage(Long, String)`。
> - `WsNewMessage` 构造：`new WsNewMessage(MessageData message)`；`MessageData` 字段 `id / senderType / content / createdAt`；主动消息复用 `WsNewMessage` 帧推送（客户端当新消息显示并回 ACK），**不新建** `WsProactiveMessage`。
> - `ChatWebSocketHandler.pushToUser(Long, Object)` 已存在（`sessionManager.getSession(userId).ifPresent(...)`）；`sendObject(WebSocketSession, Object)` 私有方法负责序列化发送。
> - `SenderType.AI = "ai"`（chat-api 顶层 String 常量，非 enum）；`MessageEntity.senderType` 是 String。
> - `chat-core/pom.xml` 当前**没有** `sanyan-push-api` 依赖 → H2 加（DeliveryService 调 PushApi）。
> - `WsAck` 已有 `serverMsgId`；`WsEventType.ACK = "ack"` 已存在。`WsMessage`（common-ws）当前字段 `type / content / clientMsgId / lastMsgId`，**无** `ackMsgId` → H1 加。
> - L1 在线判断用 `getSession()`（带 `isOpen()`），**不用** `isOnline()`（spec §7.1）。
> - L2 ACK 超时 5s：用 `CompletableFuture.orTimeout(...)`，超时窗口做成**可注入字段**（默认 5s，测试注入短超时），见 H2。

---

#### Task H0: MessageService 补 `saveAiMessage(Long, String)`（落单条 ai message）

**Files:**
- Modify: `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/MessageService.java`
- Test: `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/MessageServiceSaveAiMessageTest.java`

> **为什么需要：** 主动消息要逐条 segment 落库为一条 `MessageEntity(role=ai)` 拿 `messageId`，但现状 `handleAiReply` 是「调 LLM → 拆气泡 → 多条落库」一条龙，无法复用来落「外部已生成好的单条文案」。`saveUserMessage` 是最接近的模板（落单条 + 发 `MessagePersistedEvent`），照它写 ai 版本。
> **与 `handleAiReply` 的关系：** `handleAiReply` 内部循环落 ai 消息那段逻辑（new entity → set USER/AI → save → publish event）与本方法重复。本 task **只新增** `saveAiMessage`，**不重构** `handleAiReply`（避免牵动 Plan 1 直发链路，spec §1 决策 4「不重构普通对话直发链路」）。

- [ ] **Step 1: 先写失败的 Mockito 单测（RED）**

照 `MessageServiceSenderTypeTest` 风格（`@ExtendWith(MockitoExtension.class)` + `@Mock MessageRepository` + `@Mock ApplicationEventPublisher` + `@InjectMocks MessageService`）。

```java
package com.sanyan.chat.internal;

import com.sanyan.chat.SenderType;
import com.sanyan.chat.event.MessagePersistedEvent;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.context.ApplicationEventPublisher;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class MessageServiceSaveAiMessageTest {

    @Mock MessageRepository messageRepository;
    @Mock ApplicationEventPublisher eventPublisher;
    // 其余构造器依赖（characterApi / aiService）本测试不触达，给 @Mock 占位避免 NPE
    @Mock com.sanyan.character.CharacterApi characterApi;
    @Mock AiService aiService;
    @InjectMocks MessageService messageService;

    @Test
    void saveAiMessage_should_persist_single_ai_message_and_publish_event() {
        when(messageRepository.save(any(MessageEntity.class))).thenAnswer(inv -> {
            MessageEntity e = inv.getArgument(0);
            e.setId(777L);   // 模拟 DB 生成 id
            return e;
        });

        MessageEntity saved = messageService.saveAiMessage(42L, "早上好呀，今天也要加油哦");

        assertThat(saved.getId()).isEqualTo(777L);
        assertThat(saved.getUserId()).isEqualTo(42L);
        assertThat(saved.getSenderType()).isEqualTo(SenderType.AI);
        assertThat(saved.getContent()).isEqualTo("早上好呀，今天也要加油哦");

        ArgumentCaptor<MessagePersistedEvent> evt = ArgumentCaptor.forClass(MessagePersistedEvent.class);
        verify(eventPublisher).publishEvent(evt.capture());
        assertThat(evt.getValue().messageId()).isEqualTo(777L);
        assertThat(evt.getValue().userId()).isEqualTo(42L);
        assertThat(evt.getValue().role()).isEqualTo(SenderType.AI);
    }

    @Test
    void saveAiMessage_should_default_blank_content_to_empty_string() {
        when(messageRepository.save(any(MessageEntity.class))).thenAnswer(inv -> inv.getArgument(0));

        MessageEntity saved = messageService.saveAiMessage(42L, null);

        assertThat(saved.getContent()).isEmpty();
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-chat-core test -Dtest=MessageServiceSaveAiMessageTest`
Expected: 编译失败 / `saveAiMessage` 方法不存在（cannot find symbol）。

- [ ] **Step 3: 写最小实现（GREEN）**

在 `MessageService` 加方法（照 `saveUserMessage` 模板，仅 senderType 改 AI；`MessagePersistedEvent` 用 `SenderType.AI`）：

```java
    /**
     * 落单条 ai message（外部已生成好文案，如主动消息）→ 返回 server 端 id。
     * 与 {@link #saveUserMessage} 同构，仅 senderType 为 AI；独立事务便于调用方逐条拿 id。
     * <p>注意：不调 LLM、不拆气泡（那是 {@link #handleAiReply} 的职责）；
     * 调用方（DeliveryService）已把文案拆好成一条条 segment。
     */
    @Transactional
    public MessageEntity saveAiMessage(Long userId, String content) {
        MessageEntity aiMsg = new MessageEntity();
        aiMsg.setUserId(userId);
        aiMsg.setSenderType(SenderType.AI);
        aiMsg.setContent(content != null ? content : "");
        messageRepository.save(aiMsg);
        log.info("AI 消息已保存（单条）: userId={}, msgId={}", userId, aiMsg.getId());
        eventPublisher.publishEvent(new MessagePersistedEvent(
                aiMsg.getId(), userId, DEFAULT_CHARACTER_ID, SenderType.AI));
        return aiMsg;
    }
```

- [ ] **Step 4: 运行测试，确认通过（GREEN）**

Run: `cd server && mvn -q -pl business_packages/sanyan-chat-core test -Dtest=MessageServiceSaveAiMessageTest`
Expected: BUILD SUCCESS，2 个用例通过。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/MessageService.java \
        server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/MessageServiceSaveAiMessageTest.java
git commit -m "feat(chat): MessageService 新增 saveAiMessage 落单条 ai 消息（供主动消息投递）"
```

---

#### Task H1: `WsMessage` 加 `ackMsgId` 字段（入站 ACK 帧）

**Files:**
- Modify: `server/foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/WsMessage.java`
- Test: `server/foundation_packages/sanyan-common-ws/src/test/java/com/sanyan/common/ws/WsMessageDeserializeTest.java`

> **缺口 #3：** 客户端回 ACK 时帧体形如 `{"type":"ack","ackMsgId":777}`，handler 反序列化成 `WsMessage` 后取 `getAckMsgId()`。当前 `WsMessage` 无该字段。`@JsonInclude(NON_NULL)` 已在类上，新字段为 `Long`（可空），序列化时缺省不输出，不影响出站帧。
> **基础层改动（common-ws）→ 按 CLAUDE.md「Superpowers Task 测试粒度规范」本 task 改基础层公共 DTO，H 阶段 checkpoint 跑全量；本 task 自身只跑 common-ws 单测。**

- [ ] **Step 1: 先写失败的反序列化单测（RED）**

```java
package com.sanyan.common.ws;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class WsMessageDeserializeTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void should_deserialize_ack_frame_with_ackMsgId() throws Exception {
        WsMessage msg = mapper.readValue("{\"type\":\"ack\",\"ackMsgId\":777}", WsMessage.class);

        assertThat(msg.getType()).isEqualTo(WsEventType.ACK);
        assertThat(msg.getAckMsgId()).isEqualTo(777L);
    }

    @Test
    void ackMsgId_should_be_null_when_absent() throws Exception {
        WsMessage msg = mapper.readValue("{\"type\":\"send_message\",\"content\":\"hi\"}", WsMessage.class);

        assertThat(msg.getAckMsgId()).isNull();
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl foundation_packages/sanyan-common-ws test -Dtest=WsMessageDeserializeTest`
Expected: 编译失败（`getAckMsgId` 不存在）。

- [ ] **Step 3: 加字段（GREEN）**

在 `WsMessage` 加：

```java
    // ack（客户端确认已收到的 serverMsgId）
    private Long ackMsgId;
```

（`@Data` 自动生成 `getAckMsgId()` / `setAckMsgId()`，无需手写。）

- [ ] **Step 4: 运行测试，确认通过（GREEN）**

Run: `cd server && mvn -q -pl foundation_packages/sanyan-common-ws test -Dtest=WsMessageDeserializeTest`
Expected: BUILD SUCCESS。

- [ ] **Step 5: Commit**

```bash
git add server/foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/WsMessage.java \
        server/foundation_packages/sanyan-common-ws/src/test/java/com/sanyan/common/ws/WsMessageDeserializeTest.java
git commit -m "feat(ws): WsMessage 加 ackMsgId 字段支持入站 ACK 帧"
```

---

#### Task H2: `DeliveryService`（chat-core/internal · 4 层投递核心）

**Files:**
- Create: `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/DeliveryService.java`
- Modify: `server/business_packages/sanyan-chat-core/pom.xml`（加 `sanyan-push-api` 依赖）
- Test: `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/DeliveryServiceTest.java`

> **附录 C 签名（严格照用）：**
> - `List<Long> deliver(Long userId, Long characterId, List<String> segments)` — 落库 + 逐条 L1→L2→L3，返回落库 messageId 列表
> - `void confirmAck(Long messageId)` — 收到客户端 ACK，complete 对应 future
> - 内部 `Map<Long, CompletableFuture<Void>> pendingAcks`；L2 超时 5s（`future.orTimeout(...)`）
> **设计要点（spec §7.1）：**
> 1. 逐条 segment：`messageService.saveAiMessage(userId, seg)` 拿 `messageId`。
> 2. **L1 在线直推**：`sessionManager.getSession(userId)` 拿到 open session → WS 推 `WsNewMessage(messageService.toData(entity))`，并注册 `CompletableFuture<Void>` 到 `pendingAcks[messageId]`，挂 `orTimeout(ackTimeout)`；ACK 到达 → complete → `sent`；超时（`TimeoutException`）→ 走 L3 离线推送。
> 3. **离线（`getSession` empty）** → 不注册 future，直接走 L3。
> 4. **L3 离线推送**：`pushApi.pushToUser(userId, new PushPayload("小婉", seg, messageId))`；推送失败（抛异常 / `PushResult.status=FAILED`）记 ERROR 日志（不抛断流程，spec §5.1「投递层失败不回滚已生成内容，靠 L4 sync 兜底」）。
> 5. **WS 序列化/推送**：DeliveryService 不直接持有 `ChatWebSocketHandler`（避免 handler↔service 循环依赖），而是**直接用 `sessionManager.getSession()` + `objectMapper` 写 session**——封装一个私有 `pushFrame(WebSocketSession, Object)`。注：与 handler 的 `sendObject` 同构，但 handler 在 ws/ 包、本类在 internal/，各持一份私有发送方法（一个 synchronized session 写）可接受；若后续要消除重复，下沉到 common-ws 的 `SessionManager.send(userId, json)`——本 task **不**下沉（避免基础层改动扩大；记 TODO）。
> **超时可注入：** 字段 `private Duration ackTimeout = Duration.ofSeconds(5);`（`@Value("${sanyan.delivery.ack-timeout-ms:5000}")` 注入 ms，或测试用 `ReflectionTestUtils.setField`）。本 task 用 `ReflectionTestUtils.setField(service, "ackTimeout", Duration.ofMillis(150))` 在超时用例里缩短。

- [ ] **Step 1: 先加 pom 依赖（编译前置，非业务逻辑）**

在 `sanyan-chat-core/pom.xml` 的 `<dependencies>` 加（紧挨现有 `sanyan-memory-api` 后）：

```xml
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-push-api</artifactId></dependency>
```

> 验证：`cd server && mvn -q -pl business_packages/sanyan-chat-core -am compile` 应通过（Phase G 已建 push-api）。

- [ ] **Step 2: 先写失败的 Mockito 单测（RED）**

覆盖 4 条路径：① 在线 + ACK 及时到达 → 不走 push；② 在线 + 超时 → 转 push；③ 离线 → 直接 push；④ push 抛异常 → 不抛断、记录（用 `verify` push 被调用 + 不抛）。

```java
package com.sanyan.chat.internal;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.chat.web.MessageData;
import com.sanyan.common.ws.SessionManager;
import com.sanyan.push.PushApi;
import com.sanyan.push.dto.PushPayload;
import com.sanyan.push.dto.PushResult;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.util.ReflectionTestUtils;
import org.springframework.web.socket.WebSocketSession;

import java.time.Duration;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class DeliveryServiceTest {

    @Mock MessageService messageService;
    @Mock SessionManager sessionManager;
    @Mock PushApi pushApi;
    @Mock WebSocketSession session;

    DeliveryService deliveryService;

    @BeforeEach
    void setUp() {
        deliveryService = new DeliveryService(messageService, sessionManager, pushApi, new ObjectMapper());
        // 缩短 ACK 超时，便于「超时转 push」用例快速触发
        ReflectionTestUtils.setField(deliveryService, "ackTimeout", Duration.ofMillis(150));
    }

    private MessageData dataOf(long id, String content) {
        MessageData d = new MessageData();
        d.setId(id);
        d.setContent(content);
        return d;
    }

    private MessageEntity entityWithId(long id, String content) {
        MessageEntity e = new MessageEntity();
        e.setId(id);
        e.setContent(content);
        return e;
    }

    @Test
    void deliver_should_save_each_segment_and_return_message_ids() {
        when(messageService.saveAiMessage(eq(1L), any())).thenReturn(entityWithId(10L, "a"), entityWithId(11L, "b"));
        when(messageService.toData(any())).thenReturn(dataOf(10L, "a"), dataOf(11L, "b"));
        when(sessionManager.getSession(1L)).thenReturn(Optional.empty()); // 离线，专注验证落库 + 返回 id

        List<Long> ids = deliveryService.deliver(1L, 99L, List.of("a", "b"));

        assertThat(ids).containsExactly(10L, 11L);
        verify(messageService, times(2)).saveAiMessage(eq(1L), any());
    }

    @Test
    void deliver_online_and_acked_should_not_push_offline() throws Exception {
        when(messageService.saveAiMessage(eq(1L), any())).thenReturn(entityWithId(10L, "hi"));
        when(messageService.toData(any())).thenReturn(dataOf(10L, "hi"));
        when(sessionManager.getSession(1L)).thenReturn(Optional.of(session));
        when(session.isOpen()).thenReturn(true);

        // 在投递线程里：先发出 WsNewMessage → 注册 future → 等 ACK。用单独线程跑 deliver，主线程 confirmAck。
        Thread t = new Thread(() -> deliveryService.deliver(1L, 99L, List.of("hi")));
        t.start();
        // 等到 future 注册后再 ACK（用 awaitility 轮询 pendingAcks 非空）
        await().atMost(Duration.ofSeconds(2)).until(() ->
                deliveryService.hasPendingAck(10L));
        deliveryService.confirmAck(10L);
        t.join(2000);

        verify(pushApi, never()).pushToUser(anyLong(), any());
    }

    @Test
    void deliver_online_but_ack_timeout_should_push_offline() {
        when(messageService.saveAiMessage(eq(1L), any())).thenReturn(entityWithId(10L, "hi"));
        when(messageService.toData(any())).thenReturn(dataOf(10L, "hi"));
        when(sessionManager.getSession(1L)).thenReturn(Optional.of(session));
        when(session.isOpen()).thenReturn(true);
        when(pushApi.pushToUser(anyLong(), any())).thenReturn(PushResult.pending("APNs 占位"));

        // 不 confirmAck → 150ms 后超时 → 转 push
        deliveryService.deliver(1L, 99L, List.of("hi"));

        verify(pushApi).pushToUser(eq(1L), any(PushPayload.class));
    }

    @Test
    void deliver_offline_should_push_directly() {
        when(messageService.saveAiMessage(eq(1L), any())).thenReturn(entityWithId(10L, "hi"));
        when(sessionManager.getSession(1L)).thenReturn(Optional.empty());
        when(pushApi.pushToUser(anyLong(), any())).thenReturn(PushResult.pending("APNs 占位"));

        deliveryService.deliver(1L, 99L, List.of("hi"));

        verify(pushApi).pushToUser(eq(1L), any(PushPayload.class));
        verify(messageService, never()).toData(any()); // 离线不推 WS，不需要 toData
    }

    @Test
    void deliver_push_failure_should_not_throw() {
        when(messageService.saveAiMessage(eq(1L), any())).thenReturn(entityWithId(10L, "hi"));
        when(sessionManager.getSession(1L)).thenReturn(Optional.empty());
        when(pushApi.pushToUser(anyLong(), any())).thenThrow(new RuntimeException("APNs down"));

        // 不应抛出（失败靠 L4 sync 兜底）
        List<Long> ids = deliveryService.deliver(1L, 99L, List.of("hi"));

        assertThat(ids).containsExactly(10L);
    }
}
```

> **测试依赖：** `awaitility` 需在 chat-core test scope 可用——多数项目 `spring-boot-starter-test` 已传递引入 `org.awaitility:awaitility`。实现时先 `mvn -pl business_packages/sanyan-chat-core dependency:tree | rg awaitility` 确认；若无则在 chat-core pom `<dependencies>` test scope 加 `<dependency><groupId>org.awaitility</groupId><artifactId>awaitility</artifactId><scope>test</scope></dependency>`（版本走 spring-boot BOM 管理，不写版本号）。
> `hasPendingAck(Long)` 为测试可见辅助方法（package-private），见 Step 3。

- [ ] **Step 3: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-chat-core test -Dtest=DeliveryServiceTest`
Expected: 编译失败（`DeliveryService` 类不存在）。

- [ ] **Step 4: 写最小实现（GREEN）**

```java
package com.sanyan.chat.internal;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.chat.ws.WsNewMessage;
import com.sanyan.common.ws.SessionManager;
import com.sanyan.push.PushApi;
import com.sanyan.push.dto.PushPayload;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeoutException;

/**
 * 主动消息 4 层可靠投递（spec §7.1）。**仅服务主动消息**，不重构 Plan 1 普通对话直发链路。
 *
 * <p>单条 segment 投递流程：
 * <ol>
 *   <li>落库 ai message（{@link MessageService#saveAiMessage}）拿 messageId</li>
 *   <li><b>L1</b> 在线直推：{@link SessionManager#getSession} 拿到 open session → WS 推 {@link WsNewMessage}</li>
 *   <li><b>L2</b> ACK 超时兜底：注册 {@link CompletableFuture} + {@code orTimeout(ackTimeout)}；
 *       收到 ACK → 完成（视为 sent）；超时 / 离线 → 转 L3</li>
 *   <li><b>L3</b> 离线推送：{@link PushApi#pushToUser}（本期 APNs 占位）。失败不抛断流程，靠 L4 sync 兜底</li>
 *   <li><b>L4</b> 启动拉取兜底：客户端下次打开 app sync /api/messages（Plan 1 已有，不在本类）</li>
 * </ol>
 *
 * <p><b>L1 用 getSession() 不用 isOnline()</b>（spec §7.1）：单实例下能拿到 open session 更可靠，
 * TCP 半开极端情况由 L2 ACK 兜底——送达正确性靠 ACK，不靠在线标记精度。
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class DeliveryService {

    private final MessageService messageService;
    private final SessionManager sessionManager;
    private final PushApi pushApi;
    private final ObjectMapper objectMapper;

    /** L2 ACK 等待超时；可经 yml 配置 / 测试反射覆盖。 */
    @Value("${sanyan.delivery.ack-timeout-ms:5000}")
    private long ackTimeoutMs = 5000;
    private Duration ackTimeout = Duration.ofSeconds(5);

    /** messageId → 等待 ACK 的 future。在线推送后注册，confirmAck 完成，超时/离线移除。 */
    private final Map<Long, CompletableFuture<Void>> pendingAcks = new ConcurrentHashMap<>();

    /** 主动消息标题（推送通知用），单角色 MVP 硬编码。 */
    private static final String PUSH_TITLE = "小婉";

    public List<Long> deliver(Long userId, Long characterId, List<String> segments) {
        List<Long> messageIds = new ArrayList<>(segments.size());
        for (String segment : segments) {
            MessageEntity saved = messageService.saveAiMessage(userId, segment);
            Long messageId = saved.getId();
            messageIds.add(messageId);
            deliverOne(userId, messageId, saved, segment);
        }
        return messageIds;
    }

    private void deliverOne(Long userId, Long messageId, MessageEntity entity, String segment) {
        Optional<WebSocketSession> session = sessionManager.getSession(userId);
        if (session.isEmpty()) {
            // 离线 → 直接 L3
            pushOffline(userId, messageId, segment);
            return;
        }
        // L1 在线直推 + L2 注册 ACK future
        pushFrame(session.get(), new WsNewMessage(messageService.toData(entity)));
        CompletableFuture<Void> future = new CompletableFuture<>();
        pendingAcks.put(messageId, future);
        try {
            future.orTimeout(effectiveTimeout().toMillis(), java.util.concurrent.TimeUnit.MILLISECONDS).join();
            log.info("主动消息 ACK 确认送达: userId={}, msgId={}", userId, messageId);
        } catch (Exception e) {
            if (e.getCause() instanceof TimeoutException) {
                log.info("主动消息 ACK 超时，转离线推送: userId={}, msgId={}", userId, messageId);
            } else {
                log.warn("主动消息 ACK 等待异常，转离线推送: userId={}, msgId={}", userId, messageId, e);
            }
            pushOffline(userId, messageId, segment);
        } finally {
            pendingAcks.remove(messageId);
        }
    }

    public void confirmAck(Long messageId) {
        if (messageId == null) return;
        CompletableFuture<Void> future = pendingAcks.get(messageId);
        if (future != null) {
            future.complete(null);
        }
    }

    private void pushOffline(Long userId, Long messageId, String segment) {
        try {
            pushApi.pushToUser(userId, new PushPayload(PUSH_TITLE, segment, messageId));
        } catch (Exception e) {
            // 投递层失败不回滚已生成内容（已落 message），靠 L4 sync 兜底（spec §5.1）
            log.error("离线推送失败（不影响流程，靠 sync 兜底）: userId={}, msgId={}", userId, messageId, e);
        }
    }

    private void pushFrame(WebSocketSession session, Object payload) {
        try {
            synchronized (session) {
                if (session.isOpen()) {
                    session.sendMessage(new TextMessage(objectMapper.writeValueAsString(payload)));
                }
            }
        } catch (Exception e) {
            log.error("主动消息 WS 推送失败", e);
        }
    }

    private Duration effectiveTimeout() {
        // 测试可反射覆盖 ackTimeout；运行时优先用 yml 注入的 ackTimeoutMs
        return ackTimeout != null && ackTimeout.toMillis() != 5000
                ? ackTimeout
                : Duration.ofMillis(ackTimeoutMs);
    }

    /** 测试可见：判断某 messageId 是否已注册等待 ACK 的 future。 */
    boolean hasPendingAck(Long messageId) {
        return pendingAcks.containsKey(messageId);
    }
}
```

> **实现者注意（超时字段二选一收敛）：** 上面同时留了 `ackTimeoutMs`（`@Value`）和 `ackTimeout`（`Duration`，测试反射用）两个字段 + `effectiveTimeout()` 桥接，是为兼顾「yml 可配」+「测试可注入短超时」。实现时**简化为单字段**：直接用 `@Value("${sanyan.delivery.ack-timeout-ms:5000}") private long ackTimeoutMs;`，测试改 `ReflectionTestUtils.setField(service, "ackTimeoutMs", 150L)`，删掉 `ackTimeout` / `effectiveTimeout()`。测试的 `setField` 同步改成 `"ackTimeoutMs", 150L`。**写实现时按单字段版本落地，附录 C 只约束方法签名（deliver/confirmAck），超时实现细节自定。**

- [ ] **Step 5: 运行测试 + commit**

Run: `cd server && mvn -q -pl business_packages/sanyan-chat-core test -Dtest=DeliveryServiceTest,MessageServiceSaveAiMessageTest`
Expected: BUILD SUCCESS（5+2 用例通过）。

```bash
git add server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/DeliveryService.java \
        server/business_packages/sanyan-chat-core/pom.xml \
        server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/DeliveryServiceTest.java
git commit -m "feat(chat): 新增 DeliveryService 主动消息 4 层投递（L1 在线直推 + L2 ACK 超时兜底 + L3 离线推送）"
```

---

#### Task H3: `ChatWebSocketHandler` 注入 DeliveryService + 处理入站 `ack` 帧

**Files:**
- Modify: `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java`
- Test: `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/ws/ChatWebSocketHandlerAckTest.java`

> **附录 C：** `handleTextMessage` switch 加 `case WsEventType.ACK -> deliveryService.confirmAck(wsMsg.getAckMsgId());`。`DeliveryService` 通过构造器注入（`@RequiredArgsConstructor` 已在类上，加 final 字段即可）。
> **注意区分两类 ACK：** 现有出站 `WsAck`（服务端落 user 消息后回客户端的 `serverMsgId`）与本 task 入站 `ack`（客户端确认收到主动消息的 `ackMsgId`）方向相反，复用同一个 `type="ack"` 字符串：**出站帧是 `WsAck`，入站帧是 `WsMessage` 带 `ackMsgId`**。handler 收到 `type=ack` 入站帧才进 `confirmAck`，出站 `WsAck` 不会回流到 handler（客户端不会把它再发回），无冲突。

- [ ] **Step 1: 先写失败的单测（RED）**

handler 的 `handleTextMessage` 是 `protected`，照现有 `ChatWebSocketHandlerEventListenerTest` 用反射或直接构造 handler 调用。这里用直接构造 + 反射调 protected 方法（或把 ACK 分支抽 package-private `handleAck(WsMessage)` 便于测，**优先后者**，更干净）。

```java
package com.sanyan.chat.ws;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.chat.internal.DeliveryService;
import com.sanyan.chat.internal.MessageService;
import com.sanyan.common.ws.SessionManager;
import com.sanyan.common.ws.WsEventType;
import com.sanyan.common.ws.WsMessage;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.verify;

@ExtendWith(MockitoExtension.class)
class ChatWebSocketHandlerAckTest {

    @Mock SessionManager sessionManager;
    @Mock MessageService messageService;
    @Mock DeliveryService deliveryService;

    @Test
    void handleAck_should_delegate_ackMsgId_to_deliveryService() {
        ChatWebSocketHandler handler = new ChatWebSocketHandler(
                sessionManager, new ObjectMapper(), messageService, deliveryService);

        WsMessage ack = new WsMessage();
        ack.setType(WsEventType.ACK);
        ack.setAckMsgId(777L);

        handler.handleAck(ack);

        verify(deliveryService).confirmAck(777L);
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-chat-core test -Dtest=ChatWebSocketHandlerAckTest`
Expected: 编译失败（构造器无 `DeliveryService` 参数 / `handleAck` 不存在）。

- [ ] **Step 3: 改 handler（GREEN）**

加 `import com.sanyan.chat.internal.DeliveryService;`，加 final 字段：

```java
    private final DeliveryService deliveryService;
```

`handleTextMessage` 的 switch 加分支：

```java
            case WsEventType.ACK -> handleAck(wsMsg);
```

新增 package-private 方法（便于单测，不暴露给外部）：

```java
    /** 客户端确认已收到主动消息（入站 ack 帧）→ complete 对应投递 future。 */
    void handleAck(WsMessage wsMsg) {
        deliveryService.confirmAck(wsMsg.getAckMsgId());
    }
```

- [ ] **Step 4: 运行测试，确认通过（GREEN）**

Run: `cd server && mvn -q -pl business_packages/sanyan-chat-core test -Dtest=ChatWebSocketHandlerAckTest`
Expected: BUILD SUCCESS。

> **回归检查：** handler 构造器签名变化会影响 `ChatWebSocketHandlerEventListenerTest` / `WebSocketHandlerErrorTest`（若它们 `new ChatWebSocketHandler(...)`）→ 运行 `mvn -q -pl business_packages/sanyan-chat-core test -Dtest='ChatWebSocketHandler*'` 确认未破。若破，给这些测试的构造调用补 `@Mock DeliveryService deliveryService` 参数。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java \
        server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/ws/ChatWebSocketHandlerAckTest.java
git commit -m "feat(chat): WS handler 接入站 ack 帧 → DeliveryService.confirmAck"
```

---

#### Task H4: `ChatApi.deliverProactiveMessage` + `ChatApiImpl` 实现委托

**Files:**
- Modify: `server/business_packages/sanyan-chat-api/src/main/java/com/sanyan/chat/ChatApi.java`
- Modify: `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/api/ChatApiImpl.java`
- Test: `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/api/ChatApiImplDeliverProactiveTest.java`

> **附录 B 签名（严格照用）：** `List<Long> deliverProactiveMessage(Long userId, Long characterId, List<String> segments);`
> **跨域边界（spec §7.2）：** proactive-core 调 chat-api 的本方法委托投递（不能依赖 chat-core）。`ChatApiImpl` 薄委托 `DeliveryService.deliver`。
> **注意：** `ChatApiImpl` 当前只注入 `MessageRepository`。本 task 加注入 `DeliveryService`。

- [ ] **Step 1: 先写失败的单测（RED）**

```java
package com.sanyan.chat.api;

import com.sanyan.chat.internal.DeliveryService;
import com.sanyan.chat.internal.MessageRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ChatApiImplDeliverProactiveTest {

    @Mock MessageRepository repository;
    @Mock DeliveryService deliveryService;
    @InjectMocks ChatApiImpl chatApi;

    @Test
    void deliverProactiveMessage_should_delegate_to_deliveryService_and_return_ids() {
        when(deliveryService.deliver(eq(1L), eq(99L), eq(List.of("早安", "今天也要开心")))).
                thenReturn(List.of(10L, 11L));

        List<Long> ids = chatApi.deliverProactiveMessage(1L, 99L, List.of("早安", "今天也要开心"));

        assertThat(ids).containsExactly(10L, 11L);
        verify(deliveryService).deliver(1L, 99L, List.of("早安", "今天也要开心"));
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-chat-core test -Dtest=ChatApiImplDeliverProactiveTest`
Expected: 编译失败（`deliverProactiveMessage` 不存在 / `ChatApiImpl` 无 `DeliveryService` 字段）。

- [ ] **Step 3: 加接口方法 + 实现（GREEN）**

`ChatApi.java` 加（带 Javadoc，与现有方法风格一致）：

```java
    /**
     * 投递一条主动消息（proactive 域委托入口）：把每条 segment 落库为 ai message，
     * 经 DeliveryService 走 4 层投递（在线 WS / ACK 兜底 / 离线推送）。
     *
     * <p>跨域边界（spec §7.2）：proactive-core 不能依赖 chat-core，故投递能力经本契约暴露。
     *
     * @return 落库的 ai message id 列表（与 segments 一一对应、顺序一致）
     */
    List<Long> deliverProactiveMessage(Long userId, Long characterId, List<String> segments);
```

`ChatApiImpl.java` 加 final 字段 + 实现：

```java
    private final DeliveryService deliveryService;

    @Override
    public List<Long> deliverProactiveMessage(Long userId, Long characterId, List<String> segments) {
        return deliveryService.deliver(userId, characterId, segments);
    }
```

> 注：`@RequiredArgsConstructor` 已在 `ChatApiImpl` 上，加 `private final DeliveryService deliveryService;` 即纳入构造器。`import com.sanyan.chat.internal.DeliveryService;`。

- [ ] **Step 4: 运行测试，确认通过（GREEN）**

Run: `cd server && mvn -q -pl business_packages/sanyan-chat-core test -Dtest=ChatApiImplDeliverProactiveTest`
Expected: BUILD SUCCESS。

> **回归检查：** `ChatApplicationContextIT` 装配 `ChatApiImpl`，现在多了 `DeliveryService` 依赖（其又依赖 `SessionManager` / `PushApi` / `ObjectMapper`）。`ChatTestConfig` 的 `@ComponentScan("com.sanyan.chat" + "com.sanyan.common")` 会扫到 `DeliveryService` + `SessionManager`，但 `PushApi` 实现在 push-core（不在 chat-core）→ 需 `@MockBean PushApi pushApi`。运行 `mvn -q -pl business_packages/sanyan-chat-core test -Dtest=ChatApplicationContextIT` 确认；若装配失败，给该 IT 补 `@MockBean private com.sanyan.push.PushApi pushApi;`。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-chat-api/src/main/java/com/sanyan/chat/ChatApi.java \
        server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/api/ChatApiImpl.java \
        server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/api/ChatApiImplDeliverProactiveTest.java
git commit -m "feat(chat): ChatApi 暴露 deliverProactiveMessage 委托 DeliveryService 投递"
```

---

#### Phase H checkpoint（基础层 common-ws 改动 → 跑 chat-core + common-ws 全量）

> H1 改了基础层 `WsMessage`、H2/H3 改了 chat-core handler 与 ApiImpl 构造器签名。按 CLAUDE.md「Superpowers Task 测试粒度规范」涉及 foundation + handler 装配变化，Phase 末跑这两个模块全量。

- [ ] Run: `cd server && mvn -q -pl foundation_packages/sanyan-common-ws -am test`
- [ ] Run: `cd server && mvn -q -pl business_packages/sanyan-chat-core -am verify`
- [ ] Expected: 两者 BUILD SUCCESS（含 `ChatApplicationContextIT` 装配通过）。

---

### Phase I · proactive 基础（依赖 A, B）

> **依赖：** Phase A（`sanyan-proactive-core` 模块 + pom 已建，含 JPA / redis / common-* 依赖）、Phase B（V10 `events_pending` migration + proactive-core 全量 `src/test/resources/db/migration/V1-V11` 副本已就绪，schema IT 已验过表结构）。
> **附录 A/F/H 类型严格照用**（`EventType` / `EventStatus` 大写、`EventPendingEntity` 字段、`ProactiveErrCode` 7000-7999、`ProactiveProperties` 结构）。
> **包路径：** `com.sanyan.proactive.internal`（Entity / enum / Repository / ErrCode / Properties）、`com.sanyan.proactive.api`（ApiImpl）。

---

#### Task I1: `EventType` / `EventStatus` enum + `EventPendingEntity` + `EventPendingRepository` + Fixtures + 仓储 IT

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/EventType.java`
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/EventStatus.java`
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/EventPendingEntity.java`
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/EventPendingRepository.java`
- Create: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/fixtures/EventPendingTestFixtures.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/EventPendingRepositoryIT.java`

> **Repository 查询（附录 + spec §5.1 SKIP LOCKED）：**
> - `findByStatusAndScheduledAtBefore(EventStatus status, Instant time)` —— 普通到期扫描
> - 原生 SKIP LOCKED：`@Query(nativeQuery = true)` `SELECT * FROM events_pending WHERE status = 'SCHEDULED' AND scheduled_at <= :now ORDER BY scheduled_at LIMIT :limit FOR UPDATE SKIP LOCKED`（防 scheduler 并发重复领取），方法名 `findDueForUpdate(Instant now, int limit)`
> **存大写：** `@Enumerated(STRING)` + DB CHECK 大写值（Phase B migration 已按附录约定写大写）。
> **payload JSONB：** `@JdbcTypeCode(SqlTypes.JSON)`（附录 A）。

- [ ] **Step 1: 先写失败的仓储 IT（RED）**

照 `RelationshipRepositoryIT`（`@DataJpaTest` + `@AutoConfigureTestDatabase(NONE)` + `PostgresTestcontainerSupport` + Flyway `classpath:db/migration` + ddl-auto=none）。`events_pending` 无 FK，不需 insertTestUser。

```java
package com.sanyan.proactive.internal;

import com.sanyan.common.test.PostgresTestcontainerSupport;
import com.sanyan.common.test.TestApplication;
import com.sanyan.proactive.internal.fixtures.EventPendingTestFixtures;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase.Replace.NONE;

/**
 * I1：EventPendingEntity + Repository roundtrip + findByStatusAndScheduledAtBefore + findDueForUpdate(SKIP LOCKED)。
 * Testcontainers PG（SKIP LOCKED 是 PG 真实语法，H2 不支持，必须 PG）。Schema 由 Flyway V1-V11 生成。
 */
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@ContextConfiguration(classes = TestApplication.class)
@EntityScan(basePackages = "com.sanyan.proactive")
@EnableJpaRepositories(basePackages = "com.sanyan.proactive")
class EventPendingRepositoryIT extends PostgresTestcontainerSupport {

    @DynamicPropertySource
    static void pgProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", PostgresTestcontainerSupport::jdbcUrl);
        registry.add("spring.datasource.username", PostgresTestcontainerSupport::username);
        registry.add("spring.datasource.password", PostgresTestcontainerSupport::password);
        registry.add("spring.datasource.driver-class-name", () -> "org.postgresql.Driver");
        registry.add("spring.flyway.locations", () -> "classpath:db/migration");
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "none");
    }

    @Autowired EventPendingRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void save_and_findById_should_roundtrip_with_enums_and_jsonb_payload() {
        EventPendingEntity e = EventPendingTestFixtures.scheduled(
                1L, 99L, EventType.A_GREETING, Instant.parse("2026-05-27T00:00:00Z"));
        repo.save(e);
        em.flush();
        em.clear();

        EventPendingEntity loaded = repo.findById(e.getId()).orElseThrow();
        assertThat(loaded.getEventType()).isEqualTo(EventType.A_GREETING);
        assertThat(loaded.getStatus()).isEqualTo(EventStatus.SCHEDULED);
        assertThat(loaded.getPayload()).isEqualTo("{}");
        assertThat(loaded.getFailCount()).isZero();
        assertThat(loaded.getCreatedAt()).isNotNull();
    }

    @Test
    void findByStatusAndScheduledAtBefore_should_return_only_due_scheduled() {
        Instant now = Instant.parse("2026-05-27T08:00:00Z");
        repo.save(EventPendingTestFixtures.scheduled(1L, 99L, EventType.A_GREETING, now.minusSeconds(60))); // 到期
        repo.save(EventPendingTestFixtures.scheduled(2L, 99L, EventType.B_RECALL, now.plusSeconds(3600)));  // 未到
        em.flush();

        List<EventPendingEntity> due = repo.findByStatusAndScheduledAtBefore(EventStatus.SCHEDULED, now);

        assertThat(due).extracting(EventPendingEntity::getUserId).containsExactly(1L);
    }

    @Test
    void findDueForUpdate_should_return_due_scheduled_with_limit() {
        Instant now = Instant.parse("2026-05-27T08:00:00Z");
        repo.save(EventPendingTestFixtures.scheduled(1L, 99L, EventType.A_GREETING, now.minusSeconds(60)));
        repo.save(EventPendingTestFixtures.scheduled(2L, 99L, EventType.A_GREETING, now.minusSeconds(30)));
        em.flush();

        List<EventPendingEntity> due = repo.findDueForUpdate(now, 10);

        assertThat(due).hasSize(2);
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=EventPendingRepositoryIT`
（IT 后缀本应 `mvn verify`，但 RED 阶段先用 `test -Dtest=` 触发编译失败即可。）
Expected: 编译失败（`EventPendingEntity` / `EventType` / `EventStatus` / `EventPendingRepository` / Fixture 不存在）。

- [ ] **Step 3: 写 enum + Entity + Repository + Fixtures（GREEN）**

`EventType.java`：
```java
package com.sanyan.proactive.internal;

/** 主动消息场景类型（DB events_pending.event_type 存 name() 大写）。 */
public enum EventType { A_GREETING, B_RECALL, C_EVENT_FOLLOWUP, D_EMOTION_CARE }
```

`EventStatus.java`：
```java
package com.sanyan.proactive.internal;

/** 主动事件调度状态（DB events_pending.status 存 name() 大写）。 */
public enum EventStatus { SCHEDULED, PROCESSING, SENT, FAILED, CANCELLED }
```

`EventPendingEntity.java`（严格照附录 A）：
```java
package com.sanyan.proactive.internal;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

import java.time.Instant;

/** events_pending：主动消息调度表（spec §3.1）。投递状态记这里，不动 messages 表。 */
@Data
@Entity
@Table(name = "events_pending")
public class EventPendingEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "character_id", nullable = false)
    private Long characterId;

    @Enumerated(EnumType.STRING)
    @Column(name = "event_type", nullable = false, length = 40)
    private EventType eventType;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private EventStatus status = EventStatus.SCHEDULED;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb", nullable = false)
    private String payload = "{}";

    @Column(name = "scheduled_at", nullable = false)
    private Instant scheduledAt;

    @Column(name = "sent_at")
    private Instant sentAt;

    @Column(name = "fail_count", nullable = false)
    private Integer failCount = 0;

    @Column(name = "last_error", columnDefinition = "text")
    private String lastError;

    @CreationTimestamp
    @Column(name = "created_at")
    private Instant createdAt;
}
```

`EventPendingRepository.java`：
```java
package com.sanyan.proactive.internal;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.Instant;
import java.util.List;

public interface EventPendingRepository extends JpaRepository<EventPendingEntity, Long> {

    List<EventPendingEntity> findByStatusAndScheduledAtBefore(EventStatus status, Instant time);

    /**
     * 领取到期的 SCHEDULED 事件，FOR UPDATE SKIP LOCKED 防多 scheduler 实例并发重复领取（spec §5.1）。
     * 调用方应在同一事务内随即把领到的行标 PROCESSING。
     */
    @Query(value = "SELECT * FROM events_pending WHERE status = 'SCHEDULED' AND scheduled_at <= :now "
            + "ORDER BY scheduled_at LIMIT :limit FOR UPDATE SKIP LOCKED", nativeQuery = true)
    List<EventPendingEntity> findDueForUpdate(@Param("now") Instant now, @Param("limit") int limit);
}
```

`EventPendingTestFixtures.java`（Object Mother，强制）：
```java
package com.sanyan.proactive.internal.fixtures;

import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventStatus;
import com.sanyan.proactive.internal.EventType;

import java.time.Instant;
import java.util.Map;

/** EventPendingEntity Object Mother（java-backend-business-layer.md §5.2）。测试一律经此构造。 */
public final class EventPendingTestFixtures {

    private EventPendingTestFixtures() {}

    public static EventPendingEntity scheduled(Long userId, Long characterId, EventType type, Instant scheduledAt) {
        EventPendingEntity e = new EventPendingEntity();
        e.setUserId(userId);
        e.setCharacterId(characterId);
        e.setEventType(type);
        e.setStatus(EventStatus.SCHEDULED);
        e.setScheduledAt(scheduledAt);
        return e;
    }

    public static EventPendingEntity withPayload(Long userId, Long characterId, EventType type,
                                                 Instant scheduledAt, String jsonPayload) {
        EventPendingEntity e = scheduled(userId, characterId, type, scheduledAt);
        e.setPayload(jsonPayload);
        return e;
    }

    public static EventPendingEntity withStatus(Long userId, Long characterId, EventType type,
                                                Instant scheduledAt, EventStatus status) {
        EventPendingEntity e = scheduled(userId, characterId, type, scheduledAt);
        e.setStatus(status);
        return e;
    }
}
```

- [ ] **Step 4: 运行 IT，确认通过（GREEN）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core verify -Dit.test=EventPendingRepositoryIT -DfailIfNoTests=false`
Expected: BUILD SUCCESS（3 个 IT 用例通过，SKIP LOCKED 在真实 PG 跑通）。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/EventType.java \
        server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/EventStatus.java \
        server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/EventPendingEntity.java \
        server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/EventPendingRepository.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/fixtures/EventPendingTestFixtures.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/EventPendingRepositoryIT.java
git commit -m "feat(proactive): EventPending 实体 + 仓储（findDueForUpdate SKIP LOCKED）+ Fixtures + 仓储 IT"
```

---

#### Task I2: `ProactiveErrCode`（7000-7999）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/ProactiveErrCode.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactiveErrCodeTest.java`

> **照 `CharacterErrCode` 模板**（`implements ErrCode` + `@Getter @AllArgsConstructor` + `code/defaultMessage` 字段）。附录 F 值：`PROACTIVE_GENERATE_FAILED(7001, ...)`、`PROACTIVE_EVENT_NOT_FOUND(7002, ...)`。
> **错误码 enum 属业务常量**——但它有「区间约束」这一隐含规则，写个轻量单测守护（区间 7000-7999 + code 唯一），符合 TDD（先写守护测试）。

- [ ] **Step 1: 先写失败的单测（RED）**

```java
package com.sanyan.proactive.internal;

import org.junit.jupiter.api.Test;

import java.util.Arrays;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;

class ProactiveErrCodeTest {

    @Test
    void all_codes_should_be_within_7000_7999_range() {
        for (ProactiveErrCode c : ProactiveErrCode.values()) {
            assertThat(c.getCode()).isBetween(7000, 7999);
            assertThat(c.getDefaultMessage()).isNotBlank();
        }
    }

    @Test
    void codes_should_be_unique() {
        long distinct = Arrays.stream(ProactiveErrCode.values()).map(ProactiveErrCode::getCode).distinct().count();
        assertThat(distinct).isEqualTo(ProactiveErrCode.values().length);
    }

    @Test
    void should_expose_expected_codes() {
        assertThat(ProactiveErrCode.PROACTIVE_GENERATE_FAILED.getCode()).isEqualTo(7001);
        assertThat(ProactiveErrCode.PROACTIVE_EVENT_NOT_FOUND.getCode()).isEqualTo(7002);
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactiveErrCodeTest`
Expected: 编译失败（`ProactiveErrCode` 不存在）。

- [ ] **Step 3: 写实现（GREEN）**

```java
package com.sanyan.proactive.internal;

import com.sanyan.common.error.ErrCode;
import lombok.AllArgsConstructor;
import lombok.Getter;

/** 主动消息域错误码（7000-7999，见 ERROR_CODE_REGISTRY）。 */
@Getter
@AllArgsConstructor
public enum ProactiveErrCode implements ErrCode {
    PROACTIVE_GENERATE_FAILED(7001, "主动消息生成失败"),
    PROACTIVE_EVENT_NOT_FOUND(7002, "主动事件不存在");

    private final int code;
    private final String defaultMessage;
}
```

- [ ] **Step 4: 运行测试，确认通过（GREEN）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactiveErrCodeTest`
Expected: BUILD SUCCESS。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/ProactiveErrCode.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactiveErrCodeTest.java
git commit -m "feat(proactive): ProactiveErrCode 错误码（7000-7999）"
```

---

#### Task I3: `ProactiveProperties`（`@ConfigurationProperties("sanyan.proactive")`）+ 绑定测试

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/ProactiveProperties.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactivePropertiesBindingTest.java`

> **照 `IntimacyProperties` 模板**（`@ConfigurationProperties` + `@Data` + 嵌套静态类）。结构严格照附录 H：
> - `greeting.morningCron` / `greeting.nightCron`（String）
> - `recall.thresholdsHours`（`List<Integer>`，默认 `[24, 72, 168]`）
> - `quietHours.start` / `quietHours.end`（int，默认 23 / 8）
> - `scatterWindowMinutes`（int，默认 30）
> - `dailyCapByStage`（`List<Integer>`，默认 `[2, 3, 4, 5, 6]`）
> - `scenesByStage`（`Map<Integer, SceneFlags>`，`SceneFlags { boolean morning/night/recall/eventFollowup/emotionCare; }`）
> **绑定测试用 `ApplicationContextRunner` + `@EnableConfigurationProperties`**，喂 `sanyan.proactive.*` 属性，断言绑定结果（嵌套 Map + List 都验）。

- [ ] **Step 1: 先写失败的绑定测试（RED）**

```java
package com.sanyan.proactive.internal;

import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import static org.assertj.core.api.Assertions.assertThat;

class ProactivePropertiesBindingTest {

    private final ApplicationContextRunner runner = new ApplicationContextRunner()
            .withConfiguration(org.springframework.boot.autoconfigure.AutoConfigurations.of(
                    ConfigurationPropertiesAutoConfiguration.class))
            .withUserConfiguration(EnableProps.class);

    @EnableConfigurationProperties(ProactiveProperties.class)
    static class EnableProps {}

    @Test
    void should_bind_defaults_when_no_properties_given() {
        runner.run(ctx -> {
            ProactiveProperties props = ctx.getBean(ProactiveProperties.class);
            assertThat(props.getQuietHours().getStart()).isEqualTo(23);
            assertThat(props.getQuietHours().getEnd()).isEqualTo(8);
            assertThat(props.getScatterWindowMinutes()).isEqualTo(30);
            assertThat(props.getDailyCapByStage()).containsExactly(2, 3, 4, 5, 6);
            assertThat(props.getRecall().getThresholdsHours()).containsExactly(24, 72, 168);
        });
    }

    @Test
    void should_bind_custom_greeting_cron_and_scenes_by_stage() {
        runner.withPropertyValues(
                "sanyan.proactive.greeting.morning-cron=0 0 8 * * *",
                "sanyan.proactive.greeting.night-cron=0 30 22 * * *",
                "sanyan.proactive.scenes-by-stage.0.morning=false",
                "sanyan.proactive.scenes-by-stage.0.recall=true",
                "sanyan.proactive.scenes-by-stage.2.morning=true",
                "sanyan.proactive.scenes-by-stage.2.emotion-care=true"
        ).run(ctx -> {
            ProactiveProperties props = ctx.getBean(ProactiveProperties.class);
            assertThat(props.getGreeting().getMorningCron()).isEqualTo("0 0 8 * * *");
            assertThat(props.getGreeting().getNightCron()).isEqualTo("0 30 22 * * *");
            assertThat(props.getScenesByStage().get(0).isMorning()).isFalse();
            assertThat(props.getScenesByStage().get(0).isRecall()).isTrue();
            assertThat(props.getScenesByStage().get(2).isMorning()).isTrue();
            assertThat(props.getScenesByStage().get(2).isEmotionCare()).isTrue();
        });
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactivePropertiesBindingTest`
Expected: 编译失败（`ProactiveProperties` 不存在）。

- [ ] **Step 3: 写实现（GREEN）**

```java
package com.sanyan.proactive.internal;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * 主动消息频率 / 时机参数（spec §6.3 全参数化，dogfood 改 yml 不改代码）。
 * 结构见 plan 附录 H。stage 下标 0..4 对应陌生人/朋友/暧昧/恋人/老夫老妻。
 */
@ConfigurationProperties("sanyan.proactive")
@Data
public class ProactiveProperties {

    private Greeting greeting = new Greeting();
    private Recall recall = new Recall();
    private QuietHours quietHours = new QuietHours();
    private int scatterWindowMinutes = 30;
    /** 每 stage 每日主动消息上限（硬顶）；下标 = stage。 */
    private List<Integer> dailyCapByStage = new ArrayList<>(List.of(2, 3, 4, 5, 6));
    /** 每 stage 允许的场景开关；key = stage。 */
    private Map<Integer, SceneFlags> scenesByStage = new HashMap<>();

    @Data
    public static class Greeting {
        private String morningCron = "0 0 8 * * *";
        private String nightCron = "0 30 22 * * *";
    }

    @Data
    public static class Recall {
        /** 失联召回阶梯阈值（小时）：24h / 3天 / 7天。 */
        private List<Integer> thresholdsHours = new ArrayList<>(List.of(24, 72, 168));
    }

    @Data
    public static class QuietHours {
        /** 免打扰起始小时（含）。默认 23。 */
        private int start = 23;
        /** 免打扰结束小时（不含）。默认 8（早安窗口起点）。 */
        private int end = 8;
    }

    @Data
    public static class SceneFlags {
        private boolean morning;
        private boolean night;
        private boolean recall;
        private boolean eventFollowup;
        private boolean emotionCare;
    }
}
```

- [ ] **Step 4: 运行测试，确认通过（GREEN）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactivePropertiesBindingTest`
Expected: BUILD SUCCESS（默认值 + 自定义绑定两个用例通过）。

> 注：`ProactiveProperties` 需被 `@EnableConfigurationProperties` 激活——在 J3 的 `ProactiveScheduler` 或一个 proactive 配置类上加 `@EnableConfigurationProperties(ProactiveProperties.class)`；本 task 仅定义 + 绑定测试，激活点留给后续注入它的 task（J1 FrequencyGate / L 触发器）。bootstrap `application.yml` 的 `sanyan.proactive.*` 默认值由 Phase N（配置）补；本 task 默认值已能 fallback。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/ProactiveProperties.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactivePropertiesBindingTest.java
git commit -m "feat(proactive): ProactiveProperties 频率/时机参数化配置 + 绑定测试"
```

---

#### Task I4: `ProactiveApiImpl.triggerNow`（插一条 scheduled 事件）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/api/ProactiveApiImpl.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/api/ProactiveApiImplTest.java`

> **契约（Phase A 已建 `ProactiveApi.triggerNow(Long userId, Long characterId, String eventType)` → `ProactiveTriggerResult(boolean scheduled, String reason)`）：** 插一条 `EventPendingEntity(SCHEDULED, scheduledAt=now)`，返回 `scheduled=true`。
> **eventType 入参是 String**（如 `"a_greeting"` 或 `"A_GREETING"`），需映射到 `EventType` enum；映射失败 → 抛 `BusinessException(PROACTIVE_EVENT_NOT_FOUND)`（或返回 `scheduled=false`，本 task 选**抛异常**更明确——运维误传一眼可见）。映射用大小写不敏感：先 `toUpperCase` 再 `EventType.valueOf`。
> **注意 spec §3.1 event_type DB 是小写示例，但附录约定统一大写 enum name()**——triggerNow 入参容忍两种写法（大小写不敏感），存库永远是 enum name() 大写。

- [ ] **Step 1: 先写失败的 Mockito 单测（RED）**

```java
package com.sanyan.proactive.api;

import com.sanyan.common.error.BusinessException;
import com.sanyan.proactive.dto.ProactiveTriggerResult;
import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventPendingRepository;
import com.sanyan.proactive.internal.EventStatus;
import com.sanyan.proactive.internal.EventType;
import com.sanyan.proactive.internal.ProactiveErrCode;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ProactiveApiImplTest {

    @Mock EventPendingRepository repo;
    @InjectMocks ProactiveApiImpl api;

    @Test
    void triggerNow_should_insert_scheduled_event_and_return_scheduled_true() {
        when(repo.save(any(EventPendingEntity.class))).thenAnswer(inv -> inv.getArgument(0));

        ProactiveTriggerResult result = api.triggerNow(1L, 99L, "a_greeting");

        assertThat(result.scheduled()).isTrue();
        ArgumentCaptor<EventPendingEntity> cap = ArgumentCaptor.forClass(EventPendingEntity.class);
        verify(repo).save(cap.capture());
        assertThat(cap.getValue().getUserId()).isEqualTo(1L);
        assertThat(cap.getValue().getCharacterId()).isEqualTo(99L);
        assertThat(cap.getValue().getEventType()).isEqualTo(EventType.A_GREETING);
        assertThat(cap.getValue().getStatus()).isEqualTo(EventStatus.SCHEDULED);
        assertThat(cap.getValue().getScheduledAt()).isNotNull();
    }

    @Test
    void triggerNow_should_accept_uppercase_event_type() {
        when(repo.save(any(EventPendingEntity.class))).thenAnswer(inv -> inv.getArgument(0));

        ProactiveTriggerResult result = api.triggerNow(1L, 99L, "D_EMOTION_CARE");

        assertThat(result.scheduled()).isTrue();
    }

    @Test
    void triggerNow_should_throw_on_unknown_event_type() {
        assertThatThrownBy(() -> api.triggerNow(1L, 99L, "x_unknown"))
                .isInstanceOf(BusinessException.class)
                .satisfies(ex -> assertThat(((BusinessException) ex).getErrCode())
                        .isEqualTo(ProactiveErrCode.PROACTIVE_EVENT_NOT_FOUND));
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactiveApiImplTest`
Expected: 编译失败（`ProactiveApiImpl` 不存在）。

- [ ] **Step 3: 写实现（GREEN）**

```java
package com.sanyan.proactive.api;

import com.sanyan.common.error.BusinessException;
import com.sanyan.proactive.ProactiveApi;
import com.sanyan.proactive.dto.ProactiveTriggerResult;
import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventPendingRepository;
import com.sanyan.proactive.internal.EventStatus;
import com.sanyan.proactive.internal.EventType;
import com.sanyan.proactive.internal.ProactiveErrCode;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

/**
 * {@link ProactiveApi} 实现：手动触发一次某用户某场景的调度（测试 / 运维用）。
 * 仅负责「插一条 scheduled 事件」，后续门控 / 生成 / 投递由 ProactiveScheduler → Dispatcher 处理。
 */
@Service
@RequiredArgsConstructor
public class ProactiveApiImpl implements ProactiveApi {

    private final EventPendingRepository repo;

    @Override
    @Transactional
    public ProactiveTriggerResult triggerNow(Long userId, Long characterId, String eventType) {
        EventType type = parseEventType(eventType);
        EventPendingEntity event = new EventPendingEntity();
        event.setUserId(userId);
        event.setCharacterId(characterId);
        event.setEventType(type);
        event.setStatus(EventStatus.SCHEDULED);
        event.setScheduledAt(Instant.now());
        repo.save(event);
        return new ProactiveTriggerResult(true, "已排入调度队列");
    }

    private static EventType parseEventType(String raw) {
        try {
            return EventType.valueOf(raw.trim().toUpperCase());
        } catch (IllegalArgumentException | NullPointerException e) {
            throw new BusinessException(ProactiveErrCode.PROACTIVE_EVENT_NOT_FOUND,
                    "未知的主动事件类型: " + raw);
        }
    }
}
```

- [ ] **Step 4: 运行测试，确认通过（GREEN）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactiveApiImplTest`
Expected: BUILD SUCCESS（3 个用例通过）。

> 注：`parseEventType` 是无状态私有 static（仅用入参），符合 CLAUDE.md「static 优先」；`triggerNow` 用实例方法（通过 `this.repo` 注入字段）符合 Spring Bean 例外。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/api/ProactiveApiImpl.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/api/ProactiveApiImplTest.java
git commit -m "feat(proactive): ProactiveApiImpl.triggerNow 插入 scheduled 事件（运维/测试触发入口）"
```

---

### Phase J · proactive 调度 + 门控 + 分发（依赖 C, F, H, I）

> **依赖：** Phase C（`KvCache.increment(String, Duration)` 已就绪）、Phase F（memory-api 的 `getMemoryItem(Long)→MemoryItemDto` / `markMemoryItemDone(Long)` 已就绪，`MemoryItemDto` 已建）、Phase H（`ChatApi.deliverProactiveMessage` 已就绪）、Phase I（`EventPendingEntity` / `EventType` / `EventStatus` / `EventPendingRepository.findDueForUpdate` / `ProactiveProperties` / `ProactiveErrCode` 已就绪）。
> **跨域依赖全 mock**（单测）：`CharacterApi`（findOrCreateRelationship 不涨分 / getStagePromptSegment）、`MemoryApi`（getRelevantContext / getMemoryItem / markMemoryItemDone）、`ChatApi`（deliverProactiveMessage）、`KvCache`（get / increment）、`LlmApi`（生成器内部，J 阶段生成器是占位/真实由 Phase K 实现——J2 注入 `List<ProactiveGenerator>` 全 mock）。
> **附录 D（GenerateContext / FrequencyGate / ProactiveGenerator）严格照用。**
> **生成器接口 `ProactiveGenerator`（附录 D）由本 Phase J2 引入定义**（K 阶段实现 4 个具体生成器）——J2 单测里 mock `ProactiveGenerator`，按 `supportsType()` 匹配。

---

#### Task J1: `FrequencyGate`（每日上限 + stage 场景开关 + 免打扰）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/FrequencyGate.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/FrequencyGateTest.java`

> **附录 D 签名：**
> - `boolean allow(Long userId, Long characterId, EventType type, int currentStage)`
> - `void recordSent(Long userId)`
> **门控三关（spec §6，全放行才 true）：**
> 1. **免打扰**：当前本地时间在 `[quietHours.start, quietHours.end)`（跨午夜）内 → 拒。`start=23,end=8` 表示 23:00–次日 8:00。判断：`hour >= start || hour < end`（start>end 跨午夜）；若配置 `start < end`（不跨午夜）则 `hour >= start && hour < end`。
> 2. **stage 场景开关**：`scenesByStage[currentStage]` 取 `SceneFlags`，按 `type` 映射对应字段——`A_GREETING` 看 morning/night（**早晚安合并到一个 event_type a_greeting**；morning 或 night 任一开即允许，spec §6.1 表把早安/晚安分列但 event 合并，门控按「该时段对应开关」放行：本期简化为 morning||night 任一开则 A_GREETING 放行，cron 时段决定发早还是晚）、`B_RECALL` 看 recall、`C_EVENT_FOLLOWUP` 看 eventFollowup、`D_EMOTION_CARE` 看 emotionCare。stage 越界（无配置）→ 拒。
> 3. **每日上限**：当日已发计数 `kvCache.get("proactive:sent:{userId}:{yyyy-MM-dd}")` 解析为 int，`>= dailyCapByStage[currentStage]` → 拒。
> `recordSent`：`kvCache.increment("proactive:sent:{userId}:{yyyy-MM-dd}", Duration.ofHours(36))`（附录 G TTL 36h）。
> **时间可测：** 注入 `Clock`（`private final Clock clock`）+ `LocalTime.now(clock)` / `LocalDate.now(clock)`，测试用固定 Clock 控时段；Spring 无 Clock Bean 时默认 `Clock.systemDefaultZone()`——构造器给默认或加 `@Autowired(required=false)`。**本 task 用 field + 默认值**：`private Clock clock = Clock.systemDefaultZone();`，测试 `ReflectionTestUtils.setField` 注入固定 Clock。

- [ ] **Step 1: 先写失败的 Mockito 单测（RED）**

覆盖：超上限拒 / 场景关拒 / 免打扰拒 / 全放行 true / recordSent 调 increment。

```java
package com.sanyan.proactive.internal;

import com.sanyan.common.cache.KvCache;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.util.ReflectionTestUtils;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class FrequencyGateTest {

    @Mock KvCache kvCache;

    ProactiveProperties props;
    FrequencyGate gate;

    /** 固定一个「白天 10:00」的 Clock，避开免打扰；UTC 简化。 */
    private static Clock dayClock() {
        return Clock.fixed(Instant.parse("2026-05-27T10:00:00Z"), ZoneId.of("UTC"));
    }

    /** 固定一个「凌晨 2:00」的 Clock，落在免打扰 [23,8) 内。 */
    private static Clock nightClock() {
        return Clock.fixed(Instant.parse("2026-05-27T02:00:00Z"), ZoneId.of("UTC"));
    }

    @BeforeEach
    void setUp() {
        props = new ProactiveProperties();
        props.setDailyCapByStage(List.of(2, 3, 4, 5, 6));
        Map<Integer, ProactiveProperties.SceneFlags> scenes = new HashMap<>();
        ProactiveProperties.SceneFlags s2 = new ProactiveProperties.SceneFlags();
        s2.setMorning(true); s2.setNight(true); s2.setRecall(true);
        s2.setEventFollowup(true); s2.setEmotionCare(true);
        scenes.put(2, s2);
        ProactiveProperties.SceneFlags s0 = new ProactiveProperties.SceneFlags();
        s0.setRecall(true); s0.setEventFollowup(true); // 陌生人不开早晚安/情绪
        scenes.put(0, s0);
        props.setScenesByStage(scenes);

        gate = new FrequencyGate(kvCache, props);
        ReflectionTestUtils.setField(gate, "clock", dayClock());
    }

    @Test
    void allow_should_pass_when_under_cap_scene_on_and_not_quiet() {
        when(kvCache.get(anyString())).thenReturn("1"); // stage2 cap=4，已发 1 条

        boolean allowed = gate.allow(1L, 99L, EventType.A_GREETING, 2);

        assertThat(allowed).isTrue();
    }

    @Test
    void allow_should_reject_when_daily_cap_reached() {
        when(kvCache.get(anyString())).thenReturn("4"); // stage2 cap=4，已满

        assertThat(gate.allow(1L, 99L, EventType.A_GREETING, 2)).isFalse();
    }

    @Test
    void allow_should_reject_when_scene_flag_off_for_stage() {
        when(kvCache.get(anyString())).thenReturn("0");
        // stage0 emotionCare=false → D_EMOTION_CARE 拒
        assertThat(gate.allow(1L, 99L, EventType.D_EMOTION_CARE, 0)).isFalse();
    }

    @Test
    void allow_should_reject_during_quiet_hours() {
        ReflectionTestUtils.setField(gate, "clock", nightClock());
        // 凌晨 2:00 落在 [23,8) 免打扰，直接拒（不论计数/场景）
        assertThat(gate.allow(1L, 99L, EventType.A_GREETING, 2)).isFalse();
    }

    @Test
    void allow_should_reject_when_stage_has_no_scene_config() {
        when(kvCache.get(anyString())).thenReturn("0");
        // stage 9 无配置 → 拒
        assertThat(gate.allow(1L, 99L, EventType.A_GREETING, 9)).isFalse();
    }

    @Test
    void recordSent_should_increment_daily_counter_with_36h_ttl() {
        gate.recordSent(1L);
        verify(kvCache).increment(anyString(), eq(Duration.ofHours(36)));
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=FrequencyGateTest`
Expected: 编译失败（`FrequencyGate` 不存在）。

- [ ] **Step 3: 写实现（GREEN）**

```java
package com.sanyan.proactive.internal;

import com.sanyan.common.cache.KvCache;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.Clock;
import java.time.Duration;
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
import java.util.List;

/**
 * 主动消息发送前频率门控（spec §6）：免打扰 + stage 场景开关 + 每日上限，三关全过才放行。
 * stage 取自 character-api findOrCreateRelationship（不涨分），由调用方 Dispatcher 传入。
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class FrequencyGate {

    /** 当日已发计数 key（附录 G）：proactive:sent:{userId}:{yyyy-MM-dd}。 */
    private static final String SENT_KEY_PREFIX = "proactive:sent:";
    private static final Duration SENT_TTL = Duration.ofHours(36);
    private static final DateTimeFormatter DATE_FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd");

    private final KvCache kvCache;
    private final ProactiveProperties props;

    /** 可测时钟；默认系统时区。测试经 ReflectionTestUtils 注入固定 Clock。 */
    private Clock clock = Clock.systemDefaultZone();

    public boolean allow(Long userId, Long characterId, EventType type, int currentStage) {
        if (isQuietHours()) {
            log.debug("门控拒绝（免打扰时段）: userId={}, type={}", userId, type);
            return false;
        }
        if (!sceneEnabled(type, currentStage)) {
            log.debug("门控拒绝（stage {} 场景 {} 未开启）: userId={}", currentStage, type, userId);
            return false;
        }
        if (dailyCountReached(userId, currentStage)) {
            log.debug("门控拒绝（当日已达上限）: userId={}, stage={}", userId, currentStage);
            return false;
        }
        return true;
    }

    public void recordSent(Long userId) {
        kvCache.increment(sentKey(userId), SENT_TTL);
    }

    private boolean isQuietHours() {
        int hour = LocalTime.now(clock).getHour();
        int start = props.getQuietHours().getStart();
        int end = props.getQuietHours().getEnd();
        if (start <= end) {
            // 不跨午夜
            return hour >= start && hour < end;
        }
        // 跨午夜（如 23..8）
        return hour >= start || hour < end;
    }

    private boolean sceneEnabled(EventType type, int stage) {
        ProactiveProperties.SceneFlags flags = props.getScenesByStage().get(stage);
        if (flags == null) {
            return false;
        }
        return switch (type) {
            // 早晚安合并为 A_GREETING：morning 或 night 任一开即放行，具体发早/晚由触发 cron 决定
            case A_GREETING -> flags.isMorning() || flags.isNight();
            case B_RECALL -> flags.isRecall();
            case C_EVENT_FOLLOWUP -> flags.isEventFollowup();
            case D_EMOTION_CARE -> flags.isEmotionCare();
        };
    }

    private boolean dailyCountReached(Long userId, int stage) {
        List<Integer> caps = props.getDailyCapByStage();
        if (stage < 0 || stage >= caps.size()) {
            return true; // stage 越界视为已满（保守拒绝）
        }
        int cap = caps.get(stage);
        int sent = parseCount(kvCache.get(sentKey(userId)));
        return sent >= cap;
    }

    private static int parseCount(String raw) {
        if (raw == null || raw.isBlank()) {
            return 0;
        }
        try {
            return Integer.parseInt(raw.trim());
        } catch (NumberFormatException e) {
            return 0;
        }
    }

    private String sentKey(Long userId) {
        return SENT_KEY_PREFIX + userId + ":" + LocalDate.now(clock).format(DATE_FMT);
    }
}
```

- [ ] **Step 4: 运行测试，确认通过（GREEN）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=FrequencyGateTest`
Expected: BUILD SUCCESS（6 个用例通过）。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/FrequencyGate.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/FrequencyGateTest.java
git commit -m "feat(proactive): FrequencyGate 频率门控（免打扰 + stage 场景开关 + 每日上限）"
```

---

#### Task J2: `ProactiveGenerator` 接口 + `GenerateContext` + `ProactiveDispatcher`

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/ProactiveGenerator.java`
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/GenerateContext.java`
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/ProactiveDispatcher.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactiveDispatcherTest.java`

> **附录 D 签名（严格照用）：**
> ```java
> public interface ProactiveGenerator { EventType supportsType(); List<String> generate(GenerateContext ctx); }
> public record GenerateContext(Long userId, Long characterId,
>         RelationshipDto relationship, String stagePromptSegment,
>         MemoryContext memoryContext, java.util.Map<String,Object> payload) {}
> ```
> **`GenerateContext` 包路径放 `internal/generator/`**（附录文件结构 `generator/GenerateContext.java`）。
> **Dispatcher.dispatch(EventPendingEntity) 流程（spec §5.1 ③④⑤ + §6.2 门控位置）：**
> 1. `characterApi.findOrCreateRelationship(userId, characterId)` 拿 `RelationshipDto`（不涨分），取 `currentStage`。
> 2. `frequencyGate.allow(userId, characterId, eventType, currentStage)` 不放行 → 标 `CANCELLED`（本期简化：a/b 类丢弃即 CANCELLED；c/d 类顺延逻辑留待 L/触发器，dispatcher 统一标 CANCELLED，spec §6.2「a/b 类丢弃」——c/d 顺延由调用方/后续 task 处理，本 task dispatch 不放行一律 CANCELLED）。
> 3. 放行：按 `eventType` 从注入的 `List<ProactiveGenerator>` 选 `supportsType()==eventType` 的生成器；找不到 → 抛 `BusinessException(PROACTIVE_GENERATE_FAILED)`。
> 4. 组 `GenerateContext`：`getStagePromptSegment` + `memoryApi.getRelevantContext` + 解析 payload（JSON→Map，本 task 用 ObjectMapper readValue 成 `Map<String,Object>`，payload 为 "{}" 时空 Map）。
> 5. `generator.generate(ctx)` 拿 `List<String>` segments → `chatApi.deliverProactiveMessage(userId, characterId, segments)`。
> 6. 标 `SENT`（`setStatus(SENT)` + `setSentAt(now)`）+ `frequencyGate.recordSent(userId)`。
> 7. `C_EVENT_FOLLOWUP` / `D_EMOTION_CARE` 时：取 payload 的 `memoryItemId` → `memoryApi.markMemoryItemDone(itemId)`。
> **Dispatcher 不直接持久化（无 repo.save）？** —— dispatch 改了 entity 的 status/sentAt，由调用方 Scheduler 在事务内 save（J3）。本 task **dispatch 只设置 entity 字段 + 调外部 api，不调 repo**——保持单一职责，repo.save 归 Scheduler。单测验证 entity 状态被正确设置 + 各 api 被调。
> **跨域全 mock**：CharacterApi / MemoryApi / ChatApi / FrequencyGate / ProactiveGenerator。

- [ ] **Step 1: 先写接口 + record（编译前置，非业务逻辑——接口/record 无逻辑，可先建）**

`ProactiveGenerator.java`：
```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.proactive.internal.EventType;

import java.util.List;

/** 主动消息文案生成器：每种 EventType 一个实现（Phase K）。返回多条 segment（按 typing 节奏逐条投递）。 */
public interface ProactiveGenerator {

    /** 本生成器负责的场景类型。 */
    EventType supportsType();

    /** 生成多条消息 segment。 */
    List<String> generate(GenerateContext ctx);
}
```

`GenerateContext.java`（严格照附录 D）：
```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.memory.dto.MemoryContext;

import java.util.Map;

/**
 * 生成器输入上下文（spec §5.4）。
 *
 * @param relationship       character-api 返回的关系（含 currentStage / currentStageName）
 * @param stagePromptSegment character-api.getStagePromptSegment 拼好的 stage 称呼/语调片段
 * @param memoryContext      memory-api.getRelevantContext 整合的长期记忆（可能为 null）
 * @param payload            事件 payload 解析出的键值（如 escalationLevel / memoryItemId）
 */
public record GenerateContext(
        Long userId,
        Long characterId,
        RelationshipDto relationship,
        String stagePromptSegment,
        MemoryContext memoryContext,
        Map<String, Object> payload) {}
```

- [ ] **Step 2: 先写失败的 Mockito 单测（RED）**

```java
package com.sanyan.proactive.internal;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.character.CharacterApi;
import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.chat.ChatApi;
import com.sanyan.common.error.BusinessException;
import com.sanyan.memory.MemoryApi;
import com.sanyan.memory.dto.MemoryContext;
import com.sanyan.proactive.internal.fixtures.EventPendingTestFixtures;
import com.sanyan.proactive.internal.generator.GenerateContext;
import com.sanyan.proactive.internal.generator.ProactiveGenerator;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ProactiveDispatcherTest {

    @Mock CharacterApi characterApi;
    @Mock MemoryApi memoryApi;
    @Mock ChatApi chatApi;
    @Mock FrequencyGate frequencyGate;
    @Mock ProactiveGenerator greetingGenerator;

    private RelationshipDto relAtStage(int stage) {
        return new RelationshipDto(1L, 99L, 350, stage, "暧昧", 600, 0.16);
    }

    private ProactiveDispatcher newDispatcher() {
        return new ProactiveDispatcher(
                characterApi, memoryApi, chatApi, frequencyGate,
                List.of(greetingGenerator), new ObjectMapper());
    }

    @Test
    void dispatch_blocked_by_gate_should_mark_cancelled_and_not_generate() {
        when(characterApi.findOrCreateRelationship(1L, 99L)).thenReturn(relAtStage(2));
        when(frequencyGate.allow(1L, 99L, EventType.A_GREETING, 2)).thenReturn(false);

        EventPendingEntity event = EventPendingTestFixtures.scheduled(
                1L, 99L, EventType.A_GREETING, Instant.now());

        newDispatcher().dispatch(event);

        assertThat(event.getStatus()).isEqualTo(EventStatus.CANCELLED);
        verify(greetingGenerator, never()).generate(any());
        verify(chatApi, never()).deliverProactiveMessage(anyLong(), anyLong(), any());
    }

    @Test
    void dispatch_allowed_should_generate_deliver_and_mark_sent_and_record() {
        when(characterApi.findOrCreateRelationship(1L, 99L)).thenReturn(relAtStage(2));
        when(frequencyGate.allow(1L, 99L, EventType.A_GREETING, 2)).thenReturn(true);
        when(characterApi.getStagePromptSegment(1L, 99L)).thenReturn("当前关系阶段：暧昧。");
        when(memoryApi.getRelevantContext(eq(1L), eq(99L), anyString())).thenReturn(new MemoryContext("记忆片段"));
        when(greetingGenerator.supportsType()).thenReturn(EventType.A_GREETING);
        when(greetingGenerator.generate(any(GenerateContext.class))).thenReturn(List.of("早安宝", "今天也要开心"));

        EventPendingEntity event = EventPendingTestFixtures.scheduled(
                1L, 99L, EventType.A_GREETING, Instant.now());

        newDispatcher().dispatch(event);

        assertThat(event.getStatus()).isEqualTo(EventStatus.SENT);
        assertThat(event.getSentAt()).isNotNull();
        verify(chatApi).deliverProactiveMessage(1L, 99L, List.of("早安宝", "今天也要开心"));
        verify(frequencyGate).recordSent(1L);
        // a 类不标记 memory item done
        verify(memoryApi, never()).markMemoryItemDone(anyLong());
    }

    @Test
    void dispatch_event_followup_should_mark_memory_item_done() {
        ProactiveGenerator followup = org.mockito.Mockito.mock(ProactiveGenerator.class);
        when(followup.supportsType()).thenReturn(EventType.C_EVENT_FOLLOWUP);
        when(followup.generate(any())).thenReturn(List.of("面试怎么样啦"));
        when(characterApi.findOrCreateRelationship(1L, 99L)).thenReturn(relAtStage(2));
        when(frequencyGate.allow(1L, 99L, EventType.C_EVENT_FOLLOWUP, 2)).thenReturn(true);
        when(characterApi.getStagePromptSegment(1L, 99L)).thenReturn("");
        when(memoryApi.getRelevantContext(anyLong(), anyLong(), anyString())).thenReturn(null);

        ProactiveDispatcher dispatcher = new ProactiveDispatcher(
                characterApi, memoryApi, chatApi, frequencyGate, List.of(followup), new ObjectMapper());

        EventPendingEntity event = EventPendingTestFixtures.withPayload(
                1L, 99L, EventType.C_EVENT_FOLLOWUP, Instant.now(), "{\"memoryItemId\": 555}");

        dispatcher.dispatch(event);

        assertThat(event.getStatus()).isEqualTo(EventStatus.SENT);
        verify(memoryApi).markMemoryItemDone(555L);
    }

    @Test
    void dispatch_should_throw_when_no_generator_supports_type() {
        when(characterApi.findOrCreateRelationship(1L, 99L)).thenReturn(relAtStage(2));
        when(frequencyGate.allow(1L, 99L, EventType.B_RECALL, 2)).thenReturn(true);
        when(greetingGenerator.supportsType()).thenReturn(EventType.A_GREETING); // 只有 greeting，无 recall 生成器

        EventPendingEntity event = EventPendingTestFixtures.scheduled(
                1L, 99L, EventType.B_RECALL, Instant.now());

        assertThatThrownBy(() -> newDispatcher().dispatch(event))
                .isInstanceOf(BusinessException.class)
                .satisfies(ex -> assertThat(((BusinessException) ex).getErrCode())
                        .isEqualTo(ProactiveErrCode.PROACTIVE_GENERATE_FAILED));
    }
}
```

- [ ] **Step 3: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactiveDispatcherTest`
Expected: 编译失败（`ProactiveDispatcher` 不存在）。

- [ ] **Step 4: 写实现（GREEN）**

```java
package com.sanyan.proactive.internal;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.character.CharacterApi;
import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.chat.ChatApi;
import com.sanyan.common.error.BusinessException;
import com.sanyan.memory.MemoryApi;
import com.sanyan.memory.dto.MemoryContext;
import com.sanyan.proactive.internal.generator.GenerateContext;
import com.sanyan.proactive.internal.generator.ProactiveGenerator;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.Collections;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;

/**
 * 分发层（spec §5.1 ③④⑤）：取 event → 频率门控 → 选生成器 → 组上下文 → 生成 → 委托投递 → 标状态。
 *
 * <p>不放行 → 标 {@link EventStatus#CANCELLED}（a/b 类丢弃；c/d 顺延逻辑由触发器/后续 task 处理）。
 * <p>本类只设置传入 entity 的状态字段 + 调外部 -api，**不调 repo 持久化**——持久化由
 * {@link ProactiveScheduler} 在其事务内 save（单一职责）。
 */
@Slf4j
@Service
public class ProactiveDispatcher {

    private static final String PAYLOAD_MEMORY_ITEM_ID = "memoryItemId";

    private final CharacterApi characterApi;
    private final MemoryApi memoryApi;
    private final ChatApi chatApi;
    private final FrequencyGate frequencyGate;
    private final ObjectMapper objectMapper;
    /** 按 EventType 索引的生成器（Spring 注入所有 ProactiveGenerator Bean，启动时建索引）。 */
    private final Map<EventType, ProactiveGenerator> generators;

    public ProactiveDispatcher(CharacterApi characterApi, MemoryApi memoryApi, ChatApi chatApi,
                               FrequencyGate frequencyGate, List<ProactiveGenerator> generatorList,
                               ObjectMapper objectMapper) {
        this.characterApi = characterApi;
        this.memoryApi = memoryApi;
        this.chatApi = chatApi;
        this.frequencyGate = frequencyGate;
        this.objectMapper = objectMapper;
        Map<EventType, ProactiveGenerator> map = new EnumMap<>(EventType.class);
        for (ProactiveGenerator g : generatorList) {
            map.put(g.supportsType(), g);
        }
        this.generators = map;
    }

    public void dispatch(EventPendingEntity event) {
        Long userId = event.getUserId();
        Long characterId = event.getCharacterId();
        EventType type = event.getEventType();

        RelationshipDto relationship = characterApi.findOrCreateRelationship(userId, characterId);
        int stage = relationship.currentStage();

        if (!frequencyGate.allow(userId, characterId, type, stage)) {
            log.info("主动消息被门控拦截，标 CANCELLED: userId={}, type={}, stage={}", userId, type, stage);
            event.setStatus(EventStatus.CANCELLED);
            return;
        }

        ProactiveGenerator generator = generators.get(type);
        if (generator == null) {
            throw new BusinessException(ProactiveErrCode.PROACTIVE_GENERATE_FAILED,
                    "无可用生成器: " + type);
        }

        Map<String, Object> payload = parsePayload(event.getPayload());
        String stagePromptSegment = characterApi.getStagePromptSegment(userId, characterId);
        MemoryContext memoryContext = memoryApi.getRelevantContext(userId, characterId, "");

        GenerateContext ctx = new GenerateContext(
                userId, characterId, relationship, stagePromptSegment, memoryContext, payload);
        List<String> segments = generator.generate(ctx);

        chatApi.deliverProactiveMessage(userId, characterId, segments);

        event.setStatus(EventStatus.SENT);
        event.setSentAt(Instant.now());
        frequencyGate.recordSent(userId);

        if (type == EventType.C_EVENT_FOLLOWUP || type == EventType.D_EMOTION_CARE) {
            Object itemId = payload.get(PAYLOAD_MEMORY_ITEM_ID);
            if (itemId != null) {
                memoryApi.markMemoryItemDone(((Number) itemId).longValue());
            }
        }
    }

    private Map<String, Object> parsePayload(String payloadJson) {
        if (payloadJson == null || payloadJson.isBlank()) {
            return Collections.emptyMap();
        }
        try {
            return objectMapper.readValue(payloadJson, new TypeReference<Map<String, Object>>() {});
        } catch (Exception e) {
            log.warn("payload 解析失败，按空处理: {}", payloadJson, e);
            return Collections.emptyMap();
        }
    }
}
```

> **proactive-core 需依赖 character-api / memory-api / chat-api**——Phase A 的 `sanyan-proactive-core/pom.xml` 已声明这些 -api 依赖（见骨架 Task A2），无需本 task 改 pom。`ProactiveProperties` 经 `@EnableConfigurationProperties` 激活：在 J3 Scheduler 类上加（见 J3）。

- [ ] **Step 5: 运行测试 + commit**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactiveDispatcherTest`
Expected: BUILD SUCCESS（4 个用例通过）。

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/ProactiveGenerator.java \
        server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/GenerateContext.java \
        server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/ProactiveDispatcher.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactiveDispatcherTest.java
git commit -m "feat(proactive): ProactiveDispatcher 门控+选生成器+组上下文+委托投递+标状态（c/d 标记记忆 done）"
```

---

#### Task J3: `ProactiveScheduler`（@Scheduled 30s 主循环 + 失败退避）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/ProactiveScheduler.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactiveSchedulerTest.java`

> **照 `RagIndexWorker` @Scheduled 模板**（`@Component` + `@Scheduled(fixedDelay=...)` + try-catch 兜底 + 失败处理私有方法）。
> **流程（spec §5.1 ② + §5.1 失败段）：**
> 1. `@Scheduled(fixedDelay = 30000)` `poll()`：`repo.findDueForUpdate(Instant.now(), BATCH_LIMIT)` 领取到期 SCHEDULED。
> 2. 逐条：先标 `PROCESSING`（save）→ `dispatcher.dispatch(event)` → save（dispatch 已设 SENT/CANCELLED）。
> 3. dispatch 抛异常 → `failCount++`、`setLastError(e.getMessage())`；`failCount > MAX_FAIL(=3)` → 标 `FAILED`；否则回 `SCHEDULED` + 退避（`scheduledAt = now + 退避`，指数：`failCount` 分钟 ×… 简化为 `now.plusSeconds(60L * failCount)`），最后 save。
> 4. try-catch 兜底照 RagIndexWorker（整个 poll 包 try-catch，Redis/DB 异常 log.debug 跳过，下次 fixedDelay 重试）。
> **@Transactional 边界：** `findDueForUpdate` 是 `FOR UPDATE SKIP LOCKED`，需在事务内才有意义——`poll()` 整体或「领取+标 PROCESSING」加 `@Transactional`。简化：`poll()` 不加 `@Transactional`（@Scheduled 方法加事务有自调用代理坑），改为**领取后逐条用独立方法处理**，每条一个事务。**本 task 简化**：`findDueForUpdate` + 标 PROCESSING 放一个 `@Transactional` 方法 `claimDue()`（返回领到的 list），再逐条 `processOne(event)`（各自 `@Transactional`）。为避免 Spring 自调用代理失效，`claimDue` / `processOne` 经注入 self 或拆到独立 Bean——**本 task 用最简方案：poll() 直接调 repo + dispatcher（不强制事务隔离 claim），SKIP LOCKED 在 findDueForUpdate 的 repo 默认事务内生效**。单测聚焦行为（领取→dispatch→save、失败退避、超限 FAILED），不验事务传播（事务行为留 J3 可选 IT 或 Phase N e2e）。
> **`@EnableConfigurationProperties(ProactiveProperties.class)`** 加在本类上（激活 Properties，供 FrequencyGate 注入）。
> **单测全 mock**：`EventPendingRepository` / `ProactiveDispatcher`。

- [ ] **Step 1: 先写失败的 Mockito 单测（RED）**

```java
package com.sanyan.proactive.internal;

import com.sanyan.proactive.internal.fixtures.EventPendingTestFixtures;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyInt;
import static org.mockito.Mockito.doNothing;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ProactiveSchedulerTest {

    @Mock EventPendingRepository repo;
    @Mock ProactiveDispatcher dispatcher;

    private ProactiveScheduler scheduler() {
        return new ProactiveScheduler(repo, dispatcher);
    }

    @Test
    void poll_should_mark_processing_dispatch_and_save() {
        EventPendingEntity event = EventPendingTestFixtures.scheduled(
                1L, 99L, EventType.A_GREETING, Instant.now());
        when(repo.findDueForUpdate(any(), anyInt())).thenReturn(List.of(event));
        doNothing().when(dispatcher).dispatch(event);

        scheduler().poll();

        // 标 PROCESSING（领取时）+ 最终 dispatch 后 save
        verify(dispatcher).dispatch(event);
        verify(repo, org.mockito.Mockito.atLeastOnce()).save(event);
    }

    @Test
    void poll_dispatch_failure_under_max_should_backoff_to_scheduled() {
        EventPendingEntity event = EventPendingTestFixtures.scheduled(
                1L, 99L, EventType.A_GREETING, Instant.now());
        event.setFailCount(0);
        when(repo.findDueForUpdate(any(), anyInt())).thenReturn(List.of(event));
        doThrow(new RuntimeException("generate boom")).when(dispatcher).dispatch(event);

        scheduler().poll();

        assertThat(event.getStatus()).isEqualTo(EventStatus.SCHEDULED);
        assertThat(event.getFailCount()).isEqualTo(1);
        assertThat(event.getLastError()).contains("generate boom");
        verify(repo, org.mockito.Mockito.atLeastOnce()).save(event);
    }

    @Test
    void poll_dispatch_failure_over_max_should_mark_failed() {
        EventPendingEntity event = EventPendingTestFixtures.scheduled(
                1L, 99L, EventType.A_GREETING, Instant.now());
        event.setFailCount(3); // 已失败 3 次，再失败 → 超限
        when(repo.findDueForUpdate(any(), anyInt())).thenReturn(List.of(event));
        doThrow(new RuntimeException("boom")).when(dispatcher).dispatch(event);

        scheduler().poll();

        assertThat(event.getStatus()).isEqualTo(EventStatus.FAILED);
        assertThat(event.getFailCount()).isEqualTo(4);
    }

    @Test
    void poll_should_swallow_repo_exception() {
        when(repo.findDueForUpdate(any(), anyInt())).thenThrow(new RuntimeException("db down"));

        // 不应抛出（整体 try-catch 兜底，照 RagIndexWorker）
        scheduler().poll();
    }
}
```

- [ ] **Step 2: 运行测试，确认失败（RED）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactiveSchedulerTest`
Expected: 编译失败（`ProactiveScheduler` 不存在）。

- [ ] **Step 3: 写实现（GREEN）**

```java
package com.sanyan.proactive.internal;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.Scheduled;

import java.time.Instant;
import java.util.List;

/**
 * 调度主循环（spec §5.1 ②）：每 30s 扫到期 SCHEDULED 事件，逐条标 PROCESSING → 委托
 * {@link ProactiveDispatcher} 分发 → 保存最终状态。失败 failCount++ 退避重排，超
 * {@link #MAX_FAIL} 次标 FAILED（spec §5.1 失败段）。
 *
 * <p>整个 {@link #poll} 包 try-catch 兜底（照 {@code RagIndexWorker}）：Redis/DB 异常
 * 安静跳过，下次 fixedDelay 重试，避免每 30s 一条 ERROR stack trace 噪音。
 *
 * <p>{@code @EnableConfigurationProperties} 在此激活 {@link ProactiveProperties}（供 FrequencyGate 注入）。
 */
@Slf4j
@Configuration
@EnableConfigurationProperties(ProactiveProperties.class)
@RequiredArgsConstructor
public class ProactiveScheduler {

    /** 单轮最多领取条数（SKIP LOCKED 防并发重复）。 */
    static final int BATCH_LIMIT = 50;
    /** 失败重试上限：超过则标 FAILED。 */
    static final int MAX_FAIL = 3;

    private final EventPendingRepository repo;
    private final ProactiveDispatcher dispatcher;

    @Scheduled(fixedDelay = 30000)
    public void poll() {
        try {
            List<EventPendingEntity> due = repo.findDueForUpdate(Instant.now(), BATCH_LIMIT);
            if (due == null || due.isEmpty()) {
                return;
            }
            for (EventPendingEntity event : due) {
                processOne(event);
            }
        } catch (Exception e) {
            log.debug("ProactiveScheduler.poll skipped (Redis/DB 不可用?): {}", e.getMessage());
        }
    }

    private void processOne(EventPendingEntity event) {
        // 领取即标 PROCESSING，防止本轮 / 其他实例重复处理
        event.setStatus(EventStatus.PROCESSING);
        repo.save(event);
        try {
            dispatcher.dispatch(event);          // 内部设置 SENT / CANCELLED
            repo.save(event);
        } catch (Exception err) {
            handleFailure(event, err);
        }
    }

    private void handleFailure(EventPendingEntity event, Exception err) {
        int newFailCount = event.getFailCount() + 1;
        event.setFailCount(newFailCount);
        event.setLastError(err.getMessage());
        if (newFailCount > MAX_FAIL) {
            event.setStatus(EventStatus.FAILED);
            log.error("主动消息分发超 {} 次，标 FAILED: eventId={}, userId={}, err={}",
                    MAX_FAIL, event.getId(), event.getUserId(), err.getMessage());
        } else {
            // 退避重排：失败次数越多排得越靠后（线性退避，简化）
            event.setStatus(EventStatus.SCHEDULED);
            event.setScheduledAt(Instant.now().plusSeconds(60L * newFailCount));
            log.warn("主动消息分发失败第 {} 次，退避重排: eventId={}, userId={}, err={}",
                    newFailCount, event.getId(), event.getUserId(), err.getMessage());
        }
        repo.save(event);
    }
}
```

> **bootstrap 需开启 @Scheduled**：项目应已有 `@EnableScheduling`（RagIndexWorker 已在用 @Scheduled，说明 bootstrap 主类或某配置已 `@EnableScheduling`）。实现者 `rg "@EnableScheduling" server/bootstrap` 确认；若无则在 bootstrap 主类加（属配置，不在本 task 范围，记 Phase N 核对）。

- [ ] **Step 4: 运行测试，确认通过（GREEN）**

Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core test -Dtest=ProactiveSchedulerTest`
Expected: BUILD SUCCESS（4 个用例通过）。

> **可选 IT（建议但非阻塞）：** SKIP LOCKED 的真实并发行为（两个 scheduler 并发 poll 不重复领取同一行）由 `EventPendingRepositoryIT.findDueForUpdate` 已覆盖查询正确性；完整并发竞争 IT 成本高，留 Phase N e2e。本 task 不强制写 Scheduler IT。

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/ProactiveScheduler.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactiveSchedulerTest.java
git commit -m "feat(proactive): ProactiveScheduler @Scheduled 30s 主循环（领取→分发→失败退避/超限标 FAILED）"
```

---

#### Phase J checkpoint（proactive-core 模块内 → 跑 proactive-core 全量）

> J 阶段都在 proactive-core 内，无跨域/基础层改动；Phase 末跑本模块全量（含 I 阶段仓储 IT）确认无回归。

- [ ] Run: `cd server && mvn -q -pl business_packages/sanyan-proactive-core -am verify`
- [ ] Expected: BUILD SUCCESS（所有 `*Test` + `EventPendingRepositoryIT` 通过）。

---
### Phase K · 生成器（4 个）

> 依赖 I（EventPending/Properties/ErrCode）、J（调度 + 门控 + 分发）。
> 所有生成器走主链路 LLM（`LlmApi.chat(LlmTaskType.USER_FACING, List<ChatMessage>)`），照 `AiService` 模板用 `ChatMessage.system(...)` / `ChatMessage.user(...)` 拼装。
> 接口 / `GenerateContext` 见骨架附录 §D；`RelationshipDto` 来自 character-api（含 `currentStage` / `currentStageName`）；`MemoryContext.text()` 取整合记忆文本。
> 单测一律 Mockito，mock `LlmApi` / `MemoryApi`（K4/K5 用）；不连真 LLM。

#### Task K1: ProactiveGenerator 接口 + GenerateContext + ProactivePromptBuilder

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/ProactiveGenerator.java`
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/GenerateContext.java`
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/ProactivePromptBuilder.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/ProactivePromptBuilderTest.java`

- [ ] **Step 1: Write the failing test**

`ProactivePromptBuilder` 负责把"基底人设占位 + stagePromptSegment + memoryContext.text() + 场景指令"拼成一条 system `ChatMessage` + 一条 user `ChatMessage`（user 段是给 LLM 的"现在请你说话"指令），拼装顺序参照 chat-core `PromptBuilder`（人设 → stage → 记忆）。

```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.llm.dto.ChatMessage;
import com.sanyan.memory.dto.MemoryContext;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ProactivePromptBuilderTest {

    private final ProactivePromptBuilder builder = new ProactivePromptBuilder();

    private GenerateContext ctx(String stageSegment, MemoryContext mem, Map<String, Object> payload) {
        RelationshipDto rel = new RelationshipDto(1L, 1L, 250, 1, "朋友", 300, 0.5);
        return new GenerateContext(1L, 1L, rel, stageSegment, mem, payload);
    }

    @Test
    void build_should_put_persona_then_stage_then_memory_in_system_then_scene_instruction_in_user() {
        List<ChatMessage> messages = builder.build(
                ctx("当前关系阶段：朋友。称呼用户用：你。语调：自然。",
                        new MemoryContext("他喜欢喝美式咖啡。"),
                        Map.of()),
                "现在请你主动跟他说一句早安。");

        // 至少一条 system + 一条 user
        assertThat(messages).hasSizeGreaterThanOrEqualTo(2);
        assertThat(messages.get(0).role()).isEqualTo("system");
        // system 段同时含人设基底、stage、记忆三块
        String system = messages.get(0).content();
        assertThat(system).contains("小婉");                  // 基底人设占位
        assertThat(system).contains("当前关系阶段：朋友");      // stage segment
        assertThat(system).contains("他喜欢喝美式咖啡");        // memoryContext.text()
        // 最后一条 user 段是场景指令
        ChatMessage last = messages.get(messages.size() - 1);
        assertThat(last.role()).isEqualTo("user");
        assertThat(last.content()).isEqualTo("现在请你主动跟他说一句早安。");
    }

    @Test
    void build_should_skip_blank_stage_and_null_memory() {
        List<ChatMessage> messages = builder.build(
                ctx("", null, Map.of()),
                "现在说句话。");

        String system = messages.get(0).content();
        assertThat(system).contains("小婉");
        assertThat(system).doesNotContain("当前关系阶段");
        // 不应出现记忆前缀
        assertThat(system).doesNotContain("她记得");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=ProactivePromptBuilderTest test
```
Expected: 编译失败 / FAIL（`ProactiveGenerator` / `GenerateContext` / `ProactivePromptBuilder` 尚不存在）。

- [ ] **Step 3: Write minimal implementation**

```java
// ProactiveGenerator.java
package com.sanyan.proactive.internal.generator;

import com.sanyan.proactive.internal.EventType;

import java.util.List;

/**
 * 主动消息文案生成器。每种 {@link EventType} 对应一个实现，Spring 自动收集所有 Bean，
 * {@code ProactiveDispatcher} 按 {@link #supportsType()} 选对应生成器。
 */
public interface ProactiveGenerator {

    /** 本生成器支持的事件类型。 */
    EventType supportsType();

    /** 生成主动消息文案，返回多条 segment（逐条按 typing 节奏投递）。 */
    List<String> generate(GenerateContext ctx);
}
```

```java
// GenerateContext.java
package com.sanyan.proactive.internal.generator;

import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.memory.dto.MemoryContext;

import java.util.Map;

/**
 * 生成器入参上下文。由 {@code ProactiveDispatcher} 在门控放行后组装：
 * <ul>
 *   <li>{@code relationship}：character-api {@code findOrCreateRelationship}（含 currentStage / currentStageName）</li>
 *   <li>{@code stagePromptSegment}：character-api {@code getStagePromptSegment}（可能为空串）</li>
 *   <li>{@code memoryContext}：memory-api {@code getRelevantContext}（可能为 null）</li>
 *   <li>{@code payload}：events_pending.payload 反序列化出的 Map（如 timeOfDay / escalationLevel / memoryItemId）</li>
 * </ul>
 */
public record GenerateContext(
        Long userId,
        Long characterId,
        RelationshipDto relationship,
        String stagePromptSegment,
        MemoryContext memoryContext,
        Map<String, Object> payload) {}
```

```java
// ProactivePromptBuilder.java
package com.sanyan.proactive.internal.generator;

import com.sanyan.llm.dto.ChatMessage;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;

/**
 * 把生成器上下文拼成 LLM 的 {@code List<ChatMessage>}。
 *
 * <p>拼装顺序对齐 chat-core {@code PromptBuilder}：
 * <ol>
 *   <li>system: 基底人设（{@link #PERSONA_BASE}）+ stagePromptSegment（非 blank）
 *       + 记忆段（memoryContext.text() 非 blank）—— 合并为一条 system message</li>
 *   <li>user: 场景指令（caller 拼好的"现在请你……"，由各 Generator 传入）</li>
 * </ol>
 *
 * <p>无实例字段，注册成 Bean 仅为让各 Generator 走 DI 注入便于测试。
 */
@Component
public class ProactivePromptBuilder {

    /**
     * 基底人设占位。
     * <p>说明：理想情况应从 character-api 拿基底 base_prompt（小婉人设），
     * 但本期 character-api 未暴露 base_prompt 查询方法，先用占位常量保证语气不跑偏；
     * stage 语调 / 记忆由后续段补齐。base_prompt 接入列为待办（不阻塞主流程）。
     */
    static final String PERSONA_BASE =
            "你是小婉，用户的 AI 伴侣。下面是你主动找用户聊天的场景——按你的人设和当前关系语气，自然地开口，"
                    + "像真人发消息一样口语、简短，不要像客服或机器人。";

    static final String MEMORY_PREFIX = "她记得关于你的事：\n";

    /**
     * @param ctx             生成上下文
     * @param sceneInstruction 场景指令（user 段），由各 Generator 按场景拼好传入
     * @return system + user 两条 ChatMessage
     */
    public List<ChatMessage> build(GenerateContext ctx, String sceneInstruction) {
        StringBuilder system = new StringBuilder(PERSONA_BASE);

        if (ctx.stagePromptSegment() != null && !ctx.stagePromptSegment().isBlank()) {
            system.append("\n\n").append(ctx.stagePromptSegment());
        }
        if (ctx.memoryContext() != null && !ctx.memoryContext().isEmpty()) {
            system.append("\n\n").append(MEMORY_PREFIX).append(ctx.memoryContext().text());
        }

        List<ChatMessage> messages = new ArrayList<>();
        messages.add(ChatMessage.system(system.toString()));
        messages.add(ChatMessage.user(sceneInstruction));
        return messages;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=ProactivePromptBuilderTest test
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/ \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/ProactivePromptBuilderTest.java
git commit -m "feat(proactive): ProactiveGenerator 接口 + GenerateContext + ProactivePromptBuilder（拼 system+user prompt）"
```

---

#### Task K2: GreetingGenerator（A_GREETING · 早晚安）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/GreetingGenerator.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/GreetingGeneratorTest.java`

- [ ] **Step 1: Write the failing test**

`payload.timeOfDay` 取 `"morning"` / `"night"`；按 stage 语调发问候，≤30 字，避免"早上好"生硬开头。单测验证：①supportsType=A_GREETING；②不同 timeOfDay 传入 LLM 的场景指令不同；③morning 与 night 用不同措辞；④返回 LLM 文案。

```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.llm.dto.ChatMessage;
import com.sanyan.memory.dto.MemoryContext;
import com.sanyan.proactive.internal.EventType;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class GreetingGeneratorTest {

    @Mock LlmApi llmApi;
    ProactivePromptBuilder promptBuilder = new ProactivePromptBuilder();

    private GreetingGenerator generator() {
        return new GreetingGenerator(llmApi, promptBuilder);
    }

    private GenerateContext ctx(String timeOfDay) {
        RelationshipDto rel = new RelationshipDto(1L, 1L, 250, 1, "朋友", 300, 0.5);
        return new GenerateContext(1L, 1L, rel,
                "当前关系阶段：朋友。称呼用户用：你。语调：自然。",
                MemoryContext.EMPTY, Map.of("timeOfDay", timeOfDay));
    }

    @Test
    void supportsType_should_be_a_greeting() {
        assertThat(generator().supportsType()).isEqualTo(EventType.A_GREETING);
    }

    @Test
    void generate_morning_should_call_user_facing_llm_and_return_segments() {
        when(llmApi.chat(eq(LlmTaskType.USER_FACING), any())).thenReturn("醒啦？今天也要加油哦");

        List<String> out = generator().generate(ctx("morning"));

        assertThat(out).containsExactly("醒啦？今天也要加油哦");
        verify(llmApi).chat(eq(LlmTaskType.USER_FACING), any());
    }

    @Test
    void generate_should_pass_different_scene_instruction_for_morning_vs_night() {
        when(llmApi.chat(eq(LlmTaskType.USER_FACING), any())).thenReturn("x");

        ArgumentCaptor<List<ChatMessage>> cap = ArgumentCaptor.forClass(List.class);

        generator().generate(ctx("morning"));
        verify(llmApi).chat(eq(LlmTaskType.USER_FACING), cap.capture());
        String morningUser = cap.getValue().get(cap.getValue().size() - 1).content();

        generator().generate(ctx("night"));
        verify(llmApi, org.mockito.Mockito.times(2)).chat(eq(LlmTaskType.USER_FACING), cap.capture());
        String nightUser = cap.getValue().get(cap.getValue().size() - 1).content();

        assertThat(morningUser).contains("早");
        assertThat(nightUser).contains("晚");
        assertThat(morningUser).isNotEqualTo(nightUser);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=GreetingGeneratorTest test
```
Expected: 编译失败（`GreetingGenerator` 不存在）。

- [ ] **Step 3: Write minimal implementation**

```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.proactive.internal.EventType;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Objects;

/**
 * a 早晚安生成器（{@link EventType#A_GREETING}）。
 *
 * <p>payload.timeOfDay = {@code "morning"} / {@code "night"}，按 stage 语调发一句问候：
 * ≤30 字、口语、避免"早上好"这类生硬开头。
 */
@Component
@RequiredArgsConstructor
public class GreetingGenerator implements ProactiveGenerator {

    static final String TIME_MORNING = "morning";

    private final LlmApi llmApi;
    private final ProactivePromptBuilder promptBuilder;

    @Override
    public EventType supportsType() {
        return EventType.A_GREETING;
    }

    @Override
    public List<String> generate(GenerateContext ctx) {
        String timeOfDay = Objects.toString(ctx.payload().get("timeOfDay"), TIME_MORNING);
        boolean morning = TIME_MORNING.equals(timeOfDay);
        String scene = morning
                ? "现在是早上，主动给他发一句早安问候。要自然口语、≤30 字，"
                + "不要用\"早上好\"\"早安\"这种生硬开头，像真的想起他随口发的一条。"
                : "现在是晚上，主动给他发一句晚安问候。要自然口语、≤30 字，"
                + "不要用\"晚安好梦\"这种套话开头，像睡前随口关心一句。";

        String text = llmApi.chat(LlmTaskType.USER_FACING, promptBuilder.build(ctx, scene));
        return List.of(text);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=GreetingGeneratorTest test
```

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/GreetingGenerator.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/GreetingGeneratorTest.java
git commit -m "feat(proactive): GreetingGenerator 早晚安文案（按 stage 语调 + timeOfDay 区分）"
```

---

#### Task K3: RecallGenerator（B_RECALL · 失联召回三档）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/RecallGenerator.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/RecallGeneratorTest.java`

- [ ] **Step 1: Write the failing test**

`payload.escalationLevel` 取 `0` / `1` / `2`，对应 关心 / 撒娇 / 占有欲 三档语调。单测：supportsType=B_RECALL；三档传入 LLM 的场景指令各不相同（含对应情绪关键词）。

```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.llm.dto.ChatMessage;
import com.sanyan.memory.dto.MemoryContext;
import com.sanyan.proactive.internal.EventType;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class RecallGeneratorTest {

    @Mock LlmApi llmApi;
    ProactivePromptBuilder promptBuilder = new ProactivePromptBuilder();

    private RecallGenerator generator() {
        return new RecallGenerator(llmApi, promptBuilder);
    }

    private GenerateContext ctx(int level) {
        RelationshipDto rel = new RelationshipDto(1L, 1L, 250, 1, "朋友", 300, 0.5);
        return new GenerateContext(1L, 1L, rel, "", MemoryContext.EMPTY,
                Map.of("escalationLevel", level));
    }

    @Test
    void supportsType_should_be_b_recall() {
        assertThat(generator().supportsType()).isEqualTo(EventType.B_RECALL);
    }

    @Test
    void generate_should_use_distinct_tone_per_escalation_level() {
        when(llmApi.chat(eq(LlmTaskType.USER_FACING), any())).thenReturn("x");
        ArgumentCaptor<List<ChatMessage>> cap = ArgumentCaptor.forClass(List.class);

        generator().generate(ctx(0));
        generator().generate(ctx(1));
        generator().generate(ctx(2));

        verify(llmApi, org.mockito.Mockito.times(3)).chat(eq(LlmTaskType.USER_FACING), cap.capture());
        List<List<ChatMessage>> all = cap.getAllValues();
        String care = all.get(0).get(all.get(0).size() - 1).content();
        String coquetry = all.get(1).get(all.get(1).size() - 1).content();
        String possessive = all.get(2).get(all.get(2).size() - 1).content();

        assertThat(care).isNotEqualTo(coquetry);
        assertThat(coquetry).isNotEqualTo(possessive);
        assertThat(care).contains("关心");
        assertThat(coquetry).contains("撒娇");
        assertThat(possessive).contains("占有");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=RecallGeneratorTest test
```

- [ ] **Step 3: Write minimal implementation**

```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.proactive.internal.EventType;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * b 失联召回生成器（{@link EventType#B_RECALL}）。
 *
 * <p>payload.escalationLevel = 0/1/2，对应三档语调（离线越久越浓）：
 * <ul>
 *   <li>0（≈24h）：关心——"好几天没见你了，最近还好吗"</li>
 *   <li>1（≈3d）：撒娇——带点小情绪、想被搭理</li>
 *   <li>2（≈7d）：占有欲——更黏、半埋怨"是不是把我忘了"</li>
 * </ul>
 */
@Component
@RequiredArgsConstructor
public class RecallGenerator implements ProactiveGenerator {

    private final LlmApi llmApi;
    private final ProactivePromptBuilder promptBuilder;

    @Override
    public EventType supportsType() {
        return EventType.B_RECALL;
    }

    @Override
    public List<String> generate(GenerateContext ctx) {
        int level = toInt(ctx.payload().get("escalationLevel"));
        String scene = switch (level) {
            case 1 -> "他有几天没来找你了。用带点撒娇的语气主动找他，"
                    + "想被搭理但别太黏，≤30 字，口语自然。";
            case 2 -> "他很久没来找你了。用带点占有欲、半埋怨的语气主动找他"
                    + "（\"是不是把我忘了\"那种），≤40 字，口语自然别说教。";
            default -> "他有一阵没来找你了。用关心的语气主动问候他一句，"
                    + "≤30 字，口语自然，别像群发。";
        };
        String text = llmApi.chat(LlmTaskType.USER_FACING, promptBuilder.build(ctx, scene));
        return List.of(text);
    }

    private static int toInt(Object o) {
        if (o instanceof Number n) {
            return n.intValue();
        }
        try {
            return o == null ? 0 : Integer.parseInt(o.toString());
        } catch (NumberFormatException e) {
            return 0;
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=RecallGeneratorTest test
```

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/RecallGenerator.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/RecallGeneratorTest.java
git commit -m "feat(proactive): RecallGenerator 失联召回三档语调（关心/撒娇/占有欲）"
```

---

#### Task K4: EventFollowupGenerator（C_EVENT_FOLLOWUP · 事件追问）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/EventFollowupGenerator.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/EventFollowupGeneratorTest.java`

- [ ] **Step 1: Write the failing test**

`payload.memoryItemId` → `memoryApi.getMemoryItem(id)` 拿 `content` 注入场景指令，追问该事（"周三面试怎么样啦"）。单测：supportsType=C_EVENT_FOLLOWUP；调 getMemoryItem 并把 content 拼进 user 段；item 不存在（getMemoryItem 抛 BusinessException）时安静降级返回空 List。

```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.common.error.BusinessException;
import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.llm.dto.ChatMessage;
import com.sanyan.memory.MemoryApi;
import com.sanyan.memory.dto.MemoryContext;
import com.sanyan.memory.dto.MemoryItemDto;
import com.sanyan.proactive.internal.EventType;
import com.sanyan.proactive.internal.ProactiveErrCode;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class EventFollowupGeneratorTest {

    @Mock LlmApi llmApi;
    @Mock MemoryApi memoryApi;
    ProactivePromptBuilder promptBuilder = new ProactivePromptBuilder();

    private EventFollowupGenerator generator() {
        return new EventFollowupGenerator(llmApi, memoryApi, promptBuilder);
    }

    private GenerateContext ctx(long itemId) {
        RelationshipDto rel = new RelationshipDto(1L, 1L, 250, 1, "朋友", 300, 0.5);
        return new GenerateContext(1L, 1L, rel, "", MemoryContext.EMPTY,
                Map.of("memoryItemId", itemId));
    }

    @Test
    void supportsType_should_be_c_event_followup() {
        assertThat(generator().supportsType()).isEqualTo(EventType.C_EVENT_FOLLOWUP);
    }

    @Test
    void generate_should_inject_memory_item_content_into_scene() {
        when(memoryApi.getMemoryItem(7L)).thenReturn(new MemoryItemDto(
                7L, 1L, 1L, "PLAN_EVENT", "周三有一场面试", Instant.now(), "PENDING"));
        when(llmApi.chat(eq(LlmTaskType.USER_FACING), any())).thenReturn("面试顺利吗？");

        ArgumentCaptor<List<ChatMessage>> cap = ArgumentCaptor.forClass(List.class);
        List<String> out = generator().generate(ctx(7L));

        assertThat(out).containsExactly("面试顺利吗？");
        verify(llmApi).chat(eq(LlmTaskType.USER_FACING), cap.capture());
        String user = cap.getValue().get(cap.getValue().size() - 1).content();
        assertThat(user).contains("周三有一场面试");
    }

    @Test
    void generate_should_return_empty_when_memory_item_not_found() {
        when(memoryApi.getMemoryItem(404L))
                .thenThrow(new BusinessException(ProactiveErrCode.PROACTIVE_EVENT_NOT_FOUND));

        List<String> out = generator().generate(ctx(404L));

        assertThat(out).isEmpty();
        verify(llmApi, never()).chat(any(), any());
    }
}
```

> 注：测试用的 `ProactiveErrCode.PROACTIVE_EVENT_NOT_FOUND` 来自 Phase I 已建的错误码枚举（见骨架附录 §F）。这里仅作为"任意 BusinessException"占位触发降级分支；实际抛错的是 memory-core 的 MemoryApiImpl（Phase F）。

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=EventFollowupGeneratorTest test
```

- [ ] **Step 3: Write minimal implementation**

```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.memory.MemoryApi;
import com.sanyan.memory.dto.MemoryItemDto;
import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.proactive.internal.EventType;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * c 事件追问生成器（{@link EventType#C_EVENT_FOLLOWUP}）。
 *
 * <p>payload.memoryItemId → {@code memoryApi.getMemoryItem} 取 content，
 * 注入场景指令让小婉追问那件事（"周三面试怎么样啦"）。
 * 条目查不到（已被删 / EXPIRED）时安静降级返回空 List，dispatcher 据此跳过。
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class EventFollowupGenerator implements ProactiveGenerator {

    private final LlmApi llmApi;
    private final MemoryApi memoryApi;
    private final ProactivePromptBuilder promptBuilder;

    @Override
    public EventType supportsType() {
        return EventType.C_EVENT_FOLLOWUP;
    }

    @Override
    public List<String> generate(GenerateContext ctx) {
        long itemId = toLong(ctx.payload().get("memoryItemId"));
        MemoryItemDto item;
        try {
            item = memoryApi.getMemoryItem(itemId);
        } catch (Exception e) {
            log.warn("事件追问：memory_item {} 取不到，跳过本次主动消息: {}", itemId, e.getMessage());
            return List.of();
        }

        String scene = "他之前跟你提过这件事：\"" + item.content() + "\"。"
                + "现在主动追问一下这件事的进展（自然地关心，别复述原话、别像查岗），≤40 字，口语。";
        String text = llmApi.chat(LlmTaskType.USER_FACING, promptBuilder.build(ctx, scene));
        return List.of(text);
    }

    private static long toLong(Object o) {
        if (o instanceof Number n) {
            return n.longValue();
        }
        return o == null ? 0L : Long.parseLong(o.toString());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=EventFollowupGeneratorTest test
```

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/EventFollowupGenerator.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/EventFollowupGeneratorTest.java
git commit -m "feat(proactive): EventFollowupGenerator 事件追问（注入 memory_item.content 追问进展）"
```

---

#### Task K5: EmotionCareGenerator（D_EMOTION_CARE · 情绪关怀）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/EmotionCareGenerator.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/EmotionCareGeneratorTest.java`

- [ ] **Step 1: Write the failing test**

`payload.memoryItemId` → `getMemoryItem` 拿 EMOTION 条目 content，间接关心（**不直说**"你昨天难过"）。单测：supportsType=D_EMOTION_CARE；场景指令包含"间接""不要直接提"等约束 + item.content；查不到时降级空 List。

```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.llm.dto.ChatMessage;
import com.sanyan.memory.MemoryApi;
import com.sanyan.memory.dto.MemoryContext;
import com.sanyan.memory.dto.MemoryItemDto;
import com.sanyan.proactive.internal.EventType;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class EmotionCareGeneratorTest {

    @Mock LlmApi llmApi;
    @Mock MemoryApi memoryApi;
    ProactivePromptBuilder promptBuilder = new ProactivePromptBuilder();

    private EmotionCareGenerator generator() {
        return new EmotionCareGenerator(llmApi, memoryApi, promptBuilder);
    }

    private GenerateContext ctx(long itemId) {
        RelationshipDto rel = new RelationshipDto(1L, 1L, 250, 1, "朋友", 300, 0.5);
        return new GenerateContext(1L, 1L, rel, "", MemoryContext.EMPTY,
                Map.of("memoryItemId", itemId));
    }

    @Test
    void supportsType_should_be_d_emotion_care() {
        assertThat(generator().supportsType()).isEqualTo(EventType.D_EMOTION_CARE);
    }

    @Test
    void generate_should_instruct_indirect_care_with_emotion_context() {
        when(memoryApi.getMemoryItem(9L)).thenReturn(new MemoryItemDto(
                9L, 1L, 1L, "EMOTION", "最近工作压力大、很焦虑", Instant.now(), "PENDING"));
        when(llmApi.chat(eq(LlmTaskType.USER_FACING), any())).thenReturn("今天有没有好好吃饭呀");

        ArgumentCaptor<List<ChatMessage>> cap = ArgumentCaptor.forClass(List.class);
        List<String> out = generator().generate(ctx(9L));

        assertThat(out).containsExactly("今天有没有好好吃饭呀");
        verify(llmApi).chat(eq(LlmTaskType.USER_FACING), cap.capture());
        String user = cap.getValue().get(cap.getValue().size() - 1).content();
        assertThat(user).contains("最近工作压力大、很焦虑");
        assertThat(user).contains("间接");        // 关键约束：间接关心
        assertThat(user).contains("不要直接");     // 不直说"你昨天难过"
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=EmotionCareGeneratorTest test
```

- [ ] **Step 3: Write minimal implementation**

```java
package com.sanyan.proactive.internal.generator;

import com.sanyan.memory.MemoryApi;
import com.sanyan.memory.dto.MemoryItemDto;
import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.proactive.internal.EventType;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * d 情绪关怀生成器（{@link EventType#D_EMOTION_CARE}）。
 *
 * <p>payload.memoryItemId → {@code memoryApi.getMemoryItem} 取 EMOTION 条目 content，
 * <b>间接</b>关心：不直接提"你昨天说难过"，而是用一句温柔日常的关心去贴近那个情绪。
 * 条目查不到时安静降级返回空 List。
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class EmotionCareGenerator implements ProactiveGenerator {

    private final LlmApi llmApi;
    private final MemoryApi memoryApi;
    private final ProactivePromptBuilder promptBuilder;

    @Override
    public EventType supportsType() {
        return EventType.D_EMOTION_CARE;
    }

    @Override
    public List<String> generate(GenerateContext ctx) {
        long itemId = toLong(ctx.payload().get("memoryItemId"));
        MemoryItemDto item;
        try {
            item = memoryApi.getMemoryItem(itemId);
        } catch (Exception e) {
            log.warn("情绪关怀：memory_item {} 取不到，跳过本次主动消息: {}", itemId, e.getMessage());
            return List.of();
        }

        String scene = "你最近观察到他的情绪状态是：\"" + item.content() + "\"。"
                + "现在用一句温柔的日常关心去贴近他这个情绪——"
                + "要间接，不要直接复述或点破他的情绪（比如不要说\"你昨天说很焦虑\"），"
                + "用关心生活的小事（吃饭/睡觉/天气）侧面传达\"我在意你\"，≤30 字，口语。";
        String text = llmApi.chat(LlmTaskType.USER_FACING, promptBuilder.build(ctx, scene));
        return List.of(text);
    }

    private static long toLong(Object o) {
        if (o instanceof Number n) {
            return n.longValue();
        }
        return o == null ? 0L : Long.parseLong(o.toString());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=EmotionCareGeneratorTest test
```

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/generator/EmotionCareGenerator.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/generator/EmotionCareGeneratorTest.java
git commit -m "feat(proactive): EmotionCareGenerator 情绪关怀（间接关心 EMOTION 条目，不直说）"
```

> **Phase K checkpoint**：4 生成器 + PromptBuilder 完成。跑 proactive-core 包测试确认全绿：
> `cd server && mvn -pl business_packages/sanyan-proactive-core test`

---

### Phase L · 触发器（4 个 — 含 L0 缺口补齐）

> 依赖 F（memory_item + MemoryItemScheduledEvent）、I（EventPendingEntity/Repository/EventType/ProactiveProperties）、J（dispatcher）。
> `@Scheduled` 写法照 memory-core `RagIndexWorker`（`@Component` + `@Scheduled` + try/catch 安静降级）。
> 排期 = 存一行 `EventPendingEntity(status=SCHEDULED)`。
> 单测一律 Mockito，mock `CharacterApi` / `RelationshipRepositoryReadApi`（L0 新增）/ `EventPendingRepository` / `LastActiveTracker` / `KvCache` / `ProactiveProperties`。

#### Task L0: character-api 补齐「列出有 active 关系的用户」（缺口）

**Files:**
- Modify: `server/business_packages/sanyan-character-api/src/main/java/com/sanyan/character/CharacterApi.java`
- Create: `server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/api/CharacterApiImplListActiveUsersTest.java`
- Modify: `server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/api/CharacterApiImpl.java`
- Modify: `server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/internal/RelationshipRepository.java`

> **缺口说明**：L1/L2 触发器需要"遍历所有有关系的用户"才能逐个判断该不该排早晚安 / 召回。当前 `CharacterApi` 无此能力，`RelationshipRepository` 也只有 `findByUserIdAndCharacterId`。本 task 补齐：
> - `RelationshipRepository` 加一个派生查询（用 `@Query` 取 distinct userId）
> - `CharacterApi` 加 `List<Long> listActiveRelationshipUserIds(Long characterId)`
> - `CharacterApiImpl` 薄委托
>
> **方案合理性**：本期单角色（characterId 固定 1L），"active 关系"= relationships 表里存在该 (userId, characterId) 行（懒创建即代表已建立关系）。spec §5.3 写"扫有 active 关系的 user"，relationships 行的存在即 active 语义（无单独 active 标志位，与现有模型一致）。**返回 userId 列表**（不是整行 RelationshipDto）——触发器只需要 userId 去遍历，stage 由触发器内部各自 `findOrCreateRelationship` 拿（spec §6.2 要求 stage 取自 findOrCreateRelationship 不涨分），避免一次性 load 全部 DTO。

- [ ] **Step 1: Write the failing test**

```java
package com.sanyan.character.api;

import com.sanyan.character.internal.RelationshipEntity;
import com.sanyan.character.internal.RelationshipRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class CharacterApiImplListActiveUsersTest {

    @Mock RelationshipRepository relationshipRepository;
    @InjectMocks CharacterApiImpl api;

    @Test
    void listActiveRelationshipUserIds_should_delegate_to_repository() {
        when(relationshipRepository.findUserIdsByCharacterId(1L))
                .thenReturn(List.of(10L, 20L, 30L));

        List<Long> ids = api.listActiveRelationshipUserIds(1L);

        assertThat(ids).containsExactly(10L, 20L, 30L);
    }
}
```

> 注：`CharacterApiImpl` 已有的其他依赖（RelationshipFindOrCreateService / RelationshipFetchService / StageOverrideQueryService / AiCharacterRepository 等）`@InjectMocks` 会注 null——本测试只触发 `listActiveRelationshipUserIds` 一条路径，不碰其它字段，安全。若 `@InjectMocks` 因构造器注入报错，改为构造时手动 new 并对其余依赖传 `mock(...)`（参照 character-core 现有 ApiImpl 单测写法）。

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-character-core -Dtest=CharacterApiImplListActiveUsersTest test
```
Expected: 编译失败（`findUserIdsByCharacterId` / `listActiveRelationshipUserIds` 不存在）。

- [ ] **Step 3: Write minimal implementation**

`RelationshipRepository` 加：

```java
import org.springframework.data.jpa.repository.Query;
import java.util.List;

// ... 接口内 ...
/** 列出与指定角色已建立关系的所有 userId（本期单角色，proactive 触发器遍历用户用）。 */
@Query("SELECT r.userId FROM RelationshipEntity r WHERE r.characterId = :characterId")
List<Long> findUserIdsByCharacterId(Long characterId);
```

`CharacterApi` 加方法：

```java
import java.util.List;

// ... 接口内，紧接现有 relationship 方法之后 ...
/**
 * 列出与指定角色已建立关系（relationships 表存在该行）的所有 userId。
 * <p>proactive 域早晚安 / 失联召回触发器遍历用户用；本期单角色 characterId 固定 1L。
 */
List<Long> listActiveRelationshipUserIds(Long characterId);
```

`CharacterApiImpl` 加实现（薄委托）：

```java
@Override
public List<Long> listActiveRelationshipUserIds(Long characterId) {
    return relationshipRepository.findUserIdsByCharacterId(characterId);
}
```

> 实现者：`CharacterApiImpl` 当前若未注入 `RelationshipRepository`，新增 final 字段（`@RequiredArgsConstructor` 会自动纳入构造器）。读现状文件确认字段名后对齐。

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-character-core -Dtest=CharacterApiImplListActiveUsersTest test
```

> **跨域影响**：本 task 改了 character-api 契约（被多模块依赖）。按「Superpowers Task 测试粒度规范」跑 character-core 全量回归：
> `cd server && mvn -pl business_packages/sanyan-character-core test`

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-character-api/src/main/java/com/sanyan/character/CharacterApi.java \
        server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/api/CharacterApiImpl.java \
        server/business_packages/sanyan-character-core/src/main/java/com/sanyan/character/internal/RelationshipRepository.java \
        server/business_packages/sanyan-character-core/src/test/java/com/sanyan/character/api/CharacterApiImplListActiveUsersTest.java
git commit -m "feat(character): CharacterApi 加 listActiveRelationshipUserIds（proactive 触发器遍历用户用）"
```

---

#### Task L1: GreetingDailyTrigger（A_GREETING · cron 扫早晚安）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/GreetingDailyTrigger.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/trigger/GreetingDailyTriggerTest.java`

- [ ] **Step 1: Write the failing test**

两个 `@Scheduled` 方法（`scheduleMorning` cron=morningCron / `scheduleNight` cron=nightCron），各自 `enqueueAll(timeOfDay)`：遍历 `listActiveRelationshipUserIds`；对每个 user `findOrCreateRelationship` 拿 stage；查 `ProactiveProperties.scenesByStage[stage]` 的 morning/night 开关，开 → 排 `EventPendingEntity(A_GREETING, scheduledAt=now+random(0..scatterWindowMinutes), payload={"timeOfDay":...})`。单测 mock 全部依赖，验证：①开关开的 user 排 1 条且 payload.timeOfDay 正确；②开关关的 stage（如 stage 0 morning=false）不排。

```java
package com.sanyan.proactive.trigger;

import com.sanyan.character.CharacterApi;
import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventPendingRepository;
import com.sanyan.proactive.internal.EventType;
import com.sanyan.proactive.internal.ProactiveProperties;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class GreetingDailyTriggerTest {

    @Mock CharacterApi characterApi;
    @Mock EventPendingRepository eventRepo;
    ProactiveProperties props;
    GreetingDailyTrigger trigger;

    private static final Long CHARACTER_ID = 1L;

    @BeforeEach
    void setup() {
        props = ProactivePropertiesFixtures.defaults();   // 见下方 Fixture 说明
        trigger = new GreetingDailyTrigger(characterApi, eventRepo, props);
        when(characterApi.listActiveRelationshipUserIds(CHARACTER_ID)).thenReturn(List.of(100L));
    }

    private void stubStage(long userId, int stage) {
        when(characterApi.findOrCreateRelationship(userId, CHARACTER_ID))
                .thenReturn(new RelationshipDto(userId, CHARACTER_ID, 0, stage, "x", 100, 0.0));
    }

    @Test
    void morning_should_enqueue_a_greeting_for_stage_with_morning_enabled() {
        stubStage(100L, 2);   // stage 2: morning=true

        trigger.scheduleMorning();

        ArgumentCaptor<EventPendingEntity> cap = ArgumentCaptor.forClass(EventPendingEntity.class);
        verify(eventRepo).save(cap.capture());
        EventPendingEntity e = cap.getValue();
        assertThat(e.getEventType()).isEqualTo(EventType.A_GREETING);
        assertThat(e.getUserId()).isEqualTo(100L);
        assertThat(e.getPayload()).contains("morning");
    }

    @Test
    void morning_should_not_enqueue_for_stage0_where_morning_disabled() {
        stubStage(100L, 0);   // stage 0: morning=false

        trigger.scheduleMorning();

        verify(eventRepo, never()).save(any());
    }

    @Test
    void night_should_enqueue_for_stage1_where_night_enabled() {
        stubStage(100L, 1);   // stage 1: night=true

        trigger.scheduleNight();

        ArgumentCaptor<EventPendingEntity> cap = ArgumentCaptor.forClass(EventPendingEntity.class);
        verify(eventRepo).save(cap.capture());
        assertThat(cap.getValue().getPayload()).contains("night");
    }
}
```

> **Fixture 说明**：`ProactivePropertiesFixtures.defaults()` 是 proactive-core 测试 fixtures（建议 Phase I 已建；若无则本 task 顺带建 `src/test/.../fixtures/ProactivePropertiesFixtures.java`），按骨架附录 §H 默认值填好 scenesByStage（stage0 morning=false/night=false，stage1 night=true，stage2+ morning=true/night=true）、greeting cron、scatterWindowMinutes=30。Object Mother 强制（spec §11）。

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=GreetingDailyTriggerTest test
```

- [ ] **Step 3: Write minimal implementation**

```java
package com.sanyan.proactive.trigger;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.character.CharacterApi;
import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventPendingRepository;
import com.sanyan.proactive.internal.EventType;
import com.sanyan.proactive.internal.ProactiveProperties;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ThreadLocalRandom;

/**
 * a 早晚安触发器（{@link EventType#A_GREETING}）。
 *
 * <p>两个 cron 方法分别在 morningCron / nightCron 命中时遍历有关系的 user，
 * 按当前 stage 的 scenesByStage[stage].morning/night 开关决定是否排期。
 * 每个 user 的 scheduledAt = now + random(0..scatterWindowMinutes)，分散并发负载（spec §5.3）。
 *
 * <p>@Scheduled 安静降级风格参照 memory-core RagIndexWorker：单 user 处理异常不阻断其余。
 * 本期单角色 characterId 固定 {@link #CHARACTER_ID}。
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class GreetingDailyTrigger {

    static final Long CHARACTER_ID = 1L;
    static final String TIME_MORNING = "morning";
    static final String TIME_NIGHT = "night";

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final CharacterApi characterApi;
    private final EventPendingRepository eventRepo;
    private final ProactiveProperties props;

    @Scheduled(cron = "${sanyan.proactive.greeting.morning-cron}")
    public void scheduleMorning() {
        enqueueAll(TIME_MORNING);
    }

    @Scheduled(cron = "${sanyan.proactive.greeting.night-cron}")
    public void scheduleNight() {
        enqueueAll(TIME_NIGHT);
    }

    void enqueueAll(String timeOfDay) {
        List<Long> userIds;
        try {
            userIds = characterApi.listActiveRelationshipUserIds(CHARACTER_ID);
        } catch (Exception e) {
            log.warn("早晚安触发：拉取活跃用户失败，跳过本轮: {}", e.getMessage());
            return;
        }
        for (Long userId : userIds) {
            try {
                enqueueOne(userId, timeOfDay);
            } catch (Exception e) {
                log.warn("早晚安触发：user {} 排期失败，跳过: {}", userId, e.getMessage());
            }
        }
    }

    private void enqueueOne(Long userId, String timeOfDay) {
        RelationshipDto rel = characterApi.findOrCreateRelationship(userId, CHARACTER_ID);
        ProactiveProperties.SceneSwitches scenes = props.sceneSwitchesFor(rel.currentStage());
        boolean enabled = TIME_MORNING.equals(timeOfDay) ? scenes.morning() : scenes.night();
        if (!enabled) {
            return;
        }

        int scatterMin = ThreadLocalRandom.current()
                .nextInt(Math.max(1, props.getScatterWindowMinutes()));
        Instant scheduledAt = Instant.now().plus(scatterMin, ChronoUnit.MINUTES);

        EventPendingEntity e = new EventPendingEntity();
        e.setUserId(userId);
        e.setCharacterId(CHARACTER_ID);
        e.setEventType(EventType.A_GREETING);
        e.setScheduledAt(scheduledAt);
        e.setPayload(writePayload(Map.of("timeOfDay", timeOfDay)));
        eventRepo.save(e);
    }

    private static String writePayload(Map<String, Object> payload) {
        try {
            return MAPPER.writeValueAsString(payload);
        } catch (Exception e) {
            return "{}";
        }
    }
}
```

> 实现者注意 `ProactiveProperties` 的访问方法（`sceneSwitchesFor(int stage)` 返回内嵌 `SceneSwitches` record + `getScatterWindowMinutes()`）需在 Phase I 的 `ProactiveProperties` 里提供；若 Phase I 当时未加，本 task 顺带补这两个 accessor + 内嵌 `SceneSwitches(boolean morning, boolean night, boolean recall, boolean eventFollowup, boolean emotionCare)`，并补对应 properties 单测。`scenesByStage` 是 `Map<Integer, SceneSwitches>`（yml key 为 stage 数字）。

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=GreetingDailyTriggerTest test
```

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/GreetingDailyTrigger.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/trigger/GreetingDailyTriggerTest.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/fixtures/
git commit -m "feat(proactive): GreetingDailyTrigger 早晚安 cron 排期（按 stage 场景开关 + 随机分散）"
```

---

#### Task L2: RecallTrigger（B_RECALL · 每 15min 扫失联）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/RecallTrigger.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/trigger/RecallTriggerTest.java`

- [ ] **Step 1: Write the failing test**

`@Scheduled(fixedDelay=...)` 每 15min：遍历用户；`lastActiveTracker.lastActiveAt(userId)` 算离线时长 → 命中 `thresholdsHours` 的哪档（24/72/168 → level 0/1/2，取命中的最高档）；`kvCache.setIfAbsent("proactive:recall:{userId}:{level}", "1", 8d)` 去重，成功才排 `EventPendingEntity(B_RECALL, payload={"escalationLevel":level})`。stage scenes.recall=false 不排。单测：①离线 25h → level0 排；②离线 8d → level2 排；③离线 10h（未到任何档）→ 不排；④setIfAbsent 返 false（已排过）→ 不排；⑤lastActiveAt 为空 → 不排（无活跃记录视为从未来过，不召回）。

```java
package com.sanyan.proactive.trigger;

import com.sanyan.character.CharacterApi;
import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.common.cache.KvCache;
import com.sanyan.common.ws.LastActiveTracker;
import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventPendingRepository;
import com.sanyan.proactive.internal.EventType;
import com.sanyan.proactive.internal.ProactiveProperties;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.lenient;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class RecallTriggerTest {

    @Mock CharacterApi characterApi;
    @Mock EventPendingRepository eventRepo;
    @Mock LastActiveTracker lastActiveTracker;
    @Mock KvCache kvCache;
    ProactiveProperties props;
    RecallTrigger trigger;

    private static final Long CHARACTER_ID = 1L;
    private static final Long USER = 100L;

    @BeforeEach
    void setup() {
        props = ProactivePropertiesFixtures.defaults();   // thresholdsHours = [24,72,168]
        trigger = new RecallTrigger(characterApi, eventRepo, lastActiveTracker, kvCache, props);
        when(characterApi.listActiveRelationshipUserIds(CHARACTER_ID)).thenReturn(List.of(USER));
        // stage 2 recall=true（默认所有 stage recall=true）
        lenient().when(characterApi.findOrCreateRelationship(USER, CHARACTER_ID))
                .thenReturn(new RelationshipDto(USER, CHARACTER_ID, 0, 2, "x", 100, 0.0));
        lenient().when(kvCache.setIfAbsent(anyString(), eq("1"), any(Duration.class))).thenReturn(true);
    }

    private void offlineHours(long hours) {
        when(lastActiveTracker.lastActiveAt(USER))
                .thenReturn(Optional.of(Instant.now().minus(Duration.ofHours(hours))));
    }

    @Test
    void offline_25h_should_enqueue_recall_level0() {
        offlineHours(25);

        trigger.scanAndEnqueue();

        ArgumentCaptor<EventPendingEntity> cap = ArgumentCaptor.forClass(EventPendingEntity.class);
        verify(eventRepo).save(cap.capture());
        assertThat(cap.getValue().getEventType()).isEqualTo(EventType.B_RECALL);
        assertThat(cap.getValue().getPayload()).contains("\"escalationLevel\":0");
        verify(kvCache).setIfAbsent(eq("proactive:recall:100:0"), eq("1"), any(Duration.class));
    }

    @Test
    void offline_8d_should_enqueue_recall_level2() {
        offlineHours(24 * 8);

        trigger.scanAndEnqueue();

        ArgumentCaptor<EventPendingEntity> cap = ArgumentCaptor.forClass(EventPendingEntity.class);
        verify(eventRepo).save(cap.capture());
        assertThat(cap.getValue().getPayload()).contains("\"escalationLevel\":2");
    }

    @Test
    void offline_under_first_threshold_should_not_enqueue() {
        offlineHours(10);

        trigger.scanAndEnqueue();

        verify(eventRepo, never()).save(any());
    }

    @Test
    void should_not_enqueue_when_dedup_key_already_present() {
        offlineHours(25);
        when(kvCache.setIfAbsent(anyString(), eq("1"), any(Duration.class))).thenReturn(false);

        trigger.scanAndEnqueue();

        verify(eventRepo, never()).save(any());
    }

    @Test
    void should_not_enqueue_when_never_active() {
        when(lastActiveTracker.lastActiveAt(USER)).thenReturn(Optional.empty());

        trigger.scanAndEnqueue();

        verify(eventRepo, never()).save(any());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=RecallTriggerTest test
```

- [ ] **Step 3: Write minimal implementation**

```java
package com.sanyan.proactive.trigger;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.character.CharacterApi;
import com.sanyan.character.dto.RelationshipDto;
import com.sanyan.common.cache.KvCache;
import com.sanyan.common.ws.LastActiveTracker;
import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventPendingRepository;
import com.sanyan.proactive.internal.EventType;
import com.sanyan.proactive.internal.ProactiveProperties;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;

/**
 * b 失联召回触发器（{@link EventType#B_RECALL}）。
 *
 * <p>每 15min 扫一遍有关系的 user：按 last_active 离线时长命中 thresholdsHours 哪一档
 * （24/72/168h → escalationLevel 0/1/2，取命中的最高档），用
 * {@code proactive:recall:{userId}:{level}} 去重（每档每用户只召回一次，TTL 8d），
 * 成功才排 B_RECALL。用户回访刷新 last_active 后阶梯自然重置（去重 key 过期 / 离线时长归零）。
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class RecallTrigger {

    static final Long CHARACTER_ID = 1L;
    static final Duration DEDUP_TTL = Duration.ofDays(8);
    static final String DEDUP_KEY_PREFIX = "proactive:recall:";

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final CharacterApi characterApi;
    private final EventPendingRepository eventRepo;
    private final LastActiveTracker lastActiveTracker;
    private final KvCache kvCache;
    private final ProactiveProperties props;

    @Scheduled(fixedDelay = 15 * 60 * 1000)   // 15min
    public void scanAndEnqueue() {
        List<Long> userIds;
        try {
            userIds = characterApi.listActiveRelationshipUserIds(CHARACTER_ID);
        } catch (Exception e) {
            log.warn("失联召回：拉取活跃用户失败，跳过本轮: {}", e.getMessage());
            return;
        }
        for (Long userId : userIds) {
            try {
                enqueueOne(userId);
            } catch (Exception e) {
                log.warn("失联召回：user {} 处理失败，跳过: {}", userId, e.getMessage());
            }
        }
    }

    private void enqueueOne(Long userId) {
        Optional<Instant> lastActive = lastActiveTracker.lastActiveAt(userId);
        if (lastActive.isEmpty()) {
            return;   // 从无活跃记录，不召回
        }
        long offlineHours = Duration.between(lastActive.get(), Instant.now()).toHours();
        int level = resolveLevel(offlineHours);
        if (level < 0) {
            return;   // 未到第一档
        }

        // stage 场景开关
        RelationshipDto rel = characterApi.findOrCreateRelationship(userId, CHARACTER_ID);
        if (!props.sceneSwitchesFor(rel.currentStage()).recall()) {
            return;
        }

        // 每档每用户去重
        String dedupKey = DEDUP_KEY_PREFIX + userId + ":" + level;
        if (!kvCache.setIfAbsent(dedupKey, "1", DEDUP_TTL)) {
            return;
        }

        EventPendingEntity e = new EventPendingEntity();
        e.setUserId(userId);
        e.setCharacterId(CHARACTER_ID);
        e.setEventType(EventType.B_RECALL);
        e.setScheduledAt(Instant.now());
        e.setPayload(writePayload(Map.of("escalationLevel", level)));
        eventRepo.save(e);
    }

    /** 返回命中的最高档 level（0/1/2）；未到第一档返回 -1。 */
    private int resolveLevel(long offlineHours) {
        List<Integer> thresholds = props.getRecall().getThresholdsHours();
        int level = -1;
        for (int i = 0; i < thresholds.size(); i++) {
            if (offlineHours >= thresholds.get(i)) {
                level = i;
            }
        }
        return level;
    }

    private static String writePayload(Map<String, Object> payload) {
        try {
            return MAPPER.writeValueAsString(payload);
        } catch (Exception e) {
            return "{}";
        }
    }
}
```

> 实现者：`props.getRecall().getThresholdsHours()` 取自骨架附录 §H 的 `recall.thresholdsHours: [24,72,168]`。`LastActiveTracker` 在 common-ws（Phase D 已建，签名 `Optional<Instant> lastActiveAt(Long userId)`）。

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=RecallTriggerTest test
```

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/RecallTrigger.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/trigger/RecallTriggerTest.java
git commit -m "feat(proactive): RecallTrigger 失联召回（last_active 阶梯 + KvCache 去重）"
```

---

#### Task L3: MemoryItemScheduledListener（C/D · 订阅事件排期）

**Files:**
- Create: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/MemoryItemScheduledListener.java`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/trigger/MemoryItemScheduledListenerTest.java`

- [ ] **Step 1: Write the failing test**

订阅 memory-api 的 `MemoryItemScheduledEvent`（`@Async @TransactionalEventListener(AFTER_COMMIT)`）；按 `kind`：`PLAN_EVENT` → C_EVENT_FOLLOWUP / `EMOTION` → D_EMOTION_CARE；排 `EventPendingEntity(scheduledAt=salientAt, payload={"memoryItemId":...})`。`PROMISE` 不排（spec §12 本期不做）。单测：①PLAN_EVENT → 排 C 且 scheduledAt=salientAt、payload 含 memoryItemId；②EMOTION → 排 D；③PROMISE → 不排。

```java
package com.sanyan.proactive.trigger;

import com.sanyan.memory.event.MemoryItemScheduledEvent;
import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventPendingRepository;
import com.sanyan.proactive.internal.EventType;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;

@ExtendWith(MockitoExtension.class)
class MemoryItemScheduledListenerTest {

    @Mock EventPendingRepository eventRepo;
    @InjectMocks MemoryItemScheduledListener listener;

    @Test
    void plan_event_should_enqueue_c_event_followup_at_salient_at() {
        Instant salient = Instant.now().plusSeconds(3600);
        listener.onMemoryItemScheduled(
                new MemoryItemScheduledEvent(7L, 1L, 1L, "PLAN_EVENT", salient));

        ArgumentCaptor<EventPendingEntity> cap = ArgumentCaptor.forClass(EventPendingEntity.class);
        verify(eventRepo).save(cap.capture());
        EventPendingEntity e = cap.getValue();
        assertThat(e.getEventType()).isEqualTo(EventType.C_EVENT_FOLLOWUP);
        assertThat(e.getScheduledAt()).isEqualTo(salient);
        assertThat(e.getPayload()).contains("\"memoryItemId\":7");
    }

    @Test
    void emotion_should_enqueue_d_emotion_care() {
        listener.onMemoryItemScheduled(
                new MemoryItemScheduledEvent(9L, 1L, 1L, "EMOTION", Instant.now()));

        ArgumentCaptor<EventPendingEntity> cap = ArgumentCaptor.forClass(EventPendingEntity.class);
        verify(eventRepo).save(cap.capture());
        assertThat(cap.getValue().getEventType()).isEqualTo(EventType.D_EMOTION_CARE);
    }

    @Test
    void promise_should_not_enqueue() {
        listener.onMemoryItemScheduled(
                new MemoryItemScheduledEvent(11L, 1L, 1L, "PROMISE", Instant.now()));

        verify(eventRepo, never()).save(any());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=MemoryItemScheduledListenerTest test
```

- [ ] **Step 3: Write minimal implementation**

```java
package com.sanyan.proactive.trigger;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.memory.event.MemoryItemScheduledEvent;
import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventPendingRepository;
import com.sanyan.proactive.internal.EventType;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

import java.util.Map;

/**
 * c 事件追问 / d 情绪关怀触发器（订阅 memory 域事件）。
 *
 * <p>订阅 {@link MemoryItemScheduledEvent}（@Async + AFTER_COMMIT），按 kind 排对应 event：
 * <ul>
 *   <li>PLAN_EVENT → {@link EventType#C_EVENT_FOLLOWUP}</li>
 *   <li>EMOTION    → {@link EventType#D_EMOTION_CARE}</li>
 *   <li>PROMISE    → 不排（spec §12 本期仅留存不主动跟进）</li>
 * </ul>
 * scheduledAt = event.salientAt，payload 记 memoryItemId（供生成器取 content）。
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class MemoryItemScheduledListener {

    static final String KIND_PLAN_EVENT = "PLAN_EVENT";
    static final String KIND_EMOTION = "EMOTION";

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final EventPendingRepository eventRepo;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onMemoryItemScheduled(MemoryItemScheduledEvent event) {
        EventType type = switch (event.kind()) {
            case KIND_PLAN_EVENT -> EventType.C_EVENT_FOLLOWUP;
            case KIND_EMOTION -> EventType.D_EMOTION_CARE;
            default -> null;   // PROMISE / 未知 kind 不排
        };
        if (type == null) {
            return;
        }

        EventPendingEntity e = new EventPendingEntity();
        e.setUserId(event.userId());
        e.setCharacterId(event.characterId());
        e.setEventType(type);
        e.setScheduledAt(event.salientAt());
        e.setPayload(writePayload(Map.of("memoryItemId", event.memoryItemId())));
        eventRepo.save(e);
    }

    private static String writePayload(Map<String, Object> payload) {
        try {
            return MAPPER.writeValueAsString(payload);
        } catch (Exception e) {
            return "{}";
        }
    }
}
```

> 注：`switch` case 用常量需 Java 21 的 pattern/常量限制——若 `case KIND_PLAN_EVENT` 因常量表达式限制编译失败，改写为 `if/else` 比较 `event.kind()` 与两个常量（保持值复用，不硬编码字面量）。实现者按编译结果二选一。

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=MemoryItemScheduledListenerTest test
```

- [ ] **Step 5: Commit**

```bash
git add server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/MemoryItemScheduledListener.java \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/trigger/MemoryItemScheduledListenerTest.java
git commit -m "feat(proactive): MemoryItemScheduledListener 订阅 memory 事件排 c/d（PLAN_EVENT/EMOTION）"
```

> **Phase L checkpoint**：L0 改 character-api（跨域）+ 4 触发器完成。跑两包测试：
> `cd server && mvn -pl business_packages/sanyan-proactive-core test && mvn -pl business_packages/sanyan-character-core test`

---

### Phase M · 前端 Flutter（ACK 回报 + device token 注册骨架）

> 依赖 H（chat-core 收到入站 ack 帧 → complete CompletableFuture）。
> 测试照 plan3 前端 task 风格：`flutter test`，`_FakeWsClient` 注入帧 + mocktail。
> 完成后更新 `sanyan_chat_suite.dart`（flutter-business-layer.md §10 强制）。

#### Task M1: chat_controller 收到 new_message 后回 ACK 帧

**Files:**
- Modify: `app/foundation_packages/sanyan_network/lib/src/ws_client.dart`（加 `sendAck` 方法）
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`（newMessage 分支回 ACK）
- Modify/Test: `app/business_packages/sanyan_chat/test/chat/chat_controller_test.dart`

> **现状（已确认）**：
> - `WsClient` 已有 `sendMessage` / `syncMessages` 走私有 `_send`，但**无**对外的 `sendAck`。
> - `chat_controller._handleWsFrame` 已处理 `WsEventType.newMessage`（`messages.add(Message.fromJson(...))`），但**收到后不回 ACK**。
> - 入站 `WsEventType.ack` 当前语义是「server 确认 client 自己发出的消息」（用 `clientMsgId` 找回 `_pending`）。M1 要新增的是**反方向**：client 收到 server 主动消息（new_message，带 `serverMsgId`）后，回一条 `{type:"ack", ackMsgId: serverMsgId}` 让 server 的 L2 ACK 兜底 complete future（骨架附录 §C）。
> - 因此 ACK 出站帧字段用 `ackMsgId`（与服务端 `WsMessage.ackMsgId` 对齐），区别于入站 ack 的 `clientMsgId`。

- [ ] **Step 1: Write the failing test**

在 `chat_controller_test.dart` 追加（沿用现有 `_FakeWsClient`，给它加捕获 sentAcks 的能力）：

```dart
// _FakeWsClient 内追加：
//   final sentAcks = <int>[];
//   @override
//   bool sendAck(int serverMsgId) { sentAcks.add(serverMsgId); return true; }

group('new_message ACK 回报', () {
  test('收到 new_message（带 serverMsgId）后向 WS 回一条 ack 帧', () async {
    controller.listenWsForTest();

    fakeWs.inject(WsEvent.fromJson({
      'type': WsEventType.newMessage,
      'serverMsgId': 8888,
      'message': {
        'id': 8888,
        'senderType': 'ai',
        'content': '在吗？想你了',
        'createdAt': '2026-05-27T09:00:00.000Z',
      },
    }));

    await Future.microtask(() {});

    expect(fakeWs.sentAcks, contains(8888));
    // 消息仍正常入列
    expect(controller.messages.any((m) => m.id == 8888), isTrue);
  });

  test('new_message 无 serverMsgId 时不回 ack（不抛异常）', () async {
    controller.listenWsForTest();

    fakeWs.inject(WsEvent.fromJson({
      'type': WsEventType.newMessage,
      'message': {
        'id': 123,
        'senderType': 'ai',
        'content': 'x',
        'createdAt': '2026-05-27T09:00:00.000Z',
      },
    }));

    await Future.microtask(() {});

    expect(fakeWs.sentAcks, isEmpty);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd app && fvm flutter test business_packages/sanyan_chat/test/chat/chat_controller_test.dart
```
Expected: FAIL（`sendAck` 未定义 / newMessage 分支未回 ack）。

- [ ] **Step 3: Write minimal implementation**

`ws_client.dart` 加出站 ack 方法（紧接 `sendMessage` 之后）：

```dart
/// 回报「已收到 server 主动消息」。serverMsgId = server 落库的消息 id。
/// 与入站 ack（server 确认 client 消息，走 clientMsgId）反向：这里 client 确认 server 消息，走 ackMsgId。
bool sendAck(int serverMsgId) {
  return _send({
    'type': WsEventType.ack,
    'ackMsgId': serverMsgId,
  });
}
```

`chat_controller.dart` 的 `newMessage` 分支加回 ACK（在 `messages.add(...)` 之后）：

```dart
case WsEventType.newMessage:
  isAiTyping.value = false;
  if (event.message != null) {
    messages.add(Message.fromJson(event.message!));
    _scrollToBottom();
    // 主动消息可靠投递：收到 server 推送（带 serverMsgId）后回 ACK，
    // 让服务端 L2 ACK 兜底 complete future（确认送达，不走离线推送）。
    if (event.serverMsgId != null) {
      Get.find<WsClient>().sendAck(event.serverMsgId!);
    }
  }
  break;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd app && fvm flutter test business_packages/sanyan_chat/test/chat/chat_controller_test.dart
```

- [ ] **Step 5: Commit**

```bash
git add app/foundation_packages/sanyan_network/lib/src/ws_client.dart \
        app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart \
        app/business_packages/sanyan_chat/test/chat/chat_controller_test.dart
git commit -m "feat(chat-ui): 收到 server 主动消息后回 ACK 帧（ackMsgId）"
```

---

#### Task M2: device token 注册 api 封装 + 登录后调用占位

**Files:**
- Create: `app/foundation_packages/sanyan_user/lib/src/api/req/register_device_token_req.dart`
- Modify: `app/foundation_packages/sanyan_user/lib/src/api/sanyan_user_api.dart`（或现有 user api 聚合入口；实现者读现状确认文件名）
- Modify: `app/foundation_packages/sanyan_user/lib/src/user_center.dart`（登录成功后调用占位，拿 token 部分标 TODO）
- Test: `app/foundation_packages/sanyan_user/test/api/register_device_token_req_test.dart`

> **范围（spec §8 / §12 待办）**：本期只搭**前端结构**——api 封装 + 登录后调用入口。**真正拿 device token** 依赖 iOS APNs 证书（`.p8`）/ 安卓推送 SDK（个推/极光）选型，本期不接，拿 token 处标 `TODO`。所以 M2 只做：①`registerDeviceToken` api 封装（POST `/api/devices/register`）；②登录后调用占位（token 为空时跳过实发，留 TODO）。轻测试只覆盖 api 封装的请求构造。
>
> **放 sanyan_user 而非 sanyan_chat 的理由**：device token 是「当前用户 × 设备」维度，属用户中心职责（foundation_packages/sanyan_user，flutter-foundation-layer.md §6）；多个业务模块（不只聊天）都可能触发推送，沉到基础层用户中心更合理。实现者先 `ls foundation_packages/sanyan_user/lib/src/api/` 确认聚合入口文件名与现有 Req 写法（参照 login_req.dart 风格）。

- [ ] **Step 1: Write the failing test**

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_user/src/api/req/register_device_token_req.dart';

void main() {
  group('RegisterDeviceTokenReq', () {
    test('path 与 method 正确', () {
      final req = RegisterDeviceTokenReq(
        platform: 'ios', vendor: 'apns', token: 'abc123',
      );
      expect(req.path, '/api/devices/register');
      expect(req.method, 'POST');
    });

    test('toJson 带 platform / vendor / token', () {
      final req = RegisterDeviceTokenReq(
        platform: 'ios', vendor: 'apns', token: 'abc123',
      );
      expect(req.toJson(), {
        'platform': 'ios',
        'vendor': 'apns',
        'token': 'abc123',
      });
    });
  });
}
```

> 注：实现者按 sanyan_user 现有 `BaseReq` 子类风格对齐（path/method/toJson 的具体覆盖方式，参照同目录 login_req.dart）。若 sanyan_user 的网络层 Req 基类签名与 sanyan_chat 不同，以 sanyan_user 现状为准。

- [ ] **Step 2: Run test to verify it fails**

```bash
cd app && fvm flutter test foundation_packages/sanyan_user/test/api/register_device_token_req_test.dart
```

- [ ] **Step 3: Write minimal implementation**

`register_device_token_req.dart`：

```dart
import 'package:sanyan_network/sanyan_network.dart';

/// POST /api/devices/register 请求：客户端上报推送 device token。
/// 本期 L3 实推未通，token 真正获取依赖 iOS .p8 证书 / 安卓推送 SDK（spec §8 待办）。
class RegisterDeviceTokenReq extends BaseReq {
  final String platform;   // 'ios' / 'android'
  final String vendor;     // 'apns' / 'getui' / 'jpush'
  final String token;

  RegisterDeviceTokenReq({
    required this.platform,
    required this.vendor,
    required this.token,
  });

  @override
  String get path => '/api/devices/register';

  @override
  String get method => 'POST';

  @override
  Map<String, dynamic> toJson() => {
        'platform': platform,
        'vendor': vendor,
        'token': token,
      };
}
```

user api 聚合入口加方法（薄封装，照现有 api 风格）：

```dart
/// 上报 device token（推送注册）。本期 L3 实推未通，仅打通通道。
static Future<ApiResponse<void>> registerDeviceToken({
  required String platform,
  required String vendor,
  required String token,
}) =>
    _client.send(
      RegisterDeviceTokenReq(platform: platform, vendor: vendor, token: token),
      fromData: (_) {},
    );
```

`user_center.dart` 登录成功后调用占位（拿 token 标 TODO）：

```dart
/// 登录成功后注册推送 token（占位）。
/// TODO(plan5-push): 接入 iOS APNs（.p8 证书）/ 安卓推送 SDK（个推/极光）后，
/// 在此处通过原生 channel 拿真实 device token 再调 registerDeviceToken。
/// 当前 token 获取未接入，直接跳过实发，仅保留调用入口结构。
void registerPushTokenAfterLogin() {
  const String? deviceToken = null;   // TODO: 接入推送 SDK 后填真实 token
  // ignore: dead_code
  if (deviceToken == null || deviceToken.isEmpty) {
    return;
  }
  // SanyanUserApi.registerDeviceToken(platform: ..., vendor: ..., token: deviceToken);
}
```

并在登录成功的现有流程里调一次 `registerPushTokenAfterLogin()`（实现者找到 user_center 登录成功回调处插入）。

- [ ] **Step 4: Run test to verify it passes**

```bash
cd app && fvm flutter test foundation_packages/sanyan_user/test/api/register_device_token_req_test.dart
```

- [ ] **Step 5: Commit**

```bash
git add app/foundation_packages/sanyan_user/
git commit -m "feat(user): device token 注册 api 封装 + 登录后调用占位（实拿 token 待证书/SDK）"
```

---

#### Task M3: sanyan_chat_suite 聚合 M1 新增用例

**Files:**
- Modify: `app/business_packages/sanyan_chat/test/sanyan_chat_suite.dart`

> M1 的新用例追加在已聚合的 `chat_controller_test.dart` 内（suite 已 import 它），无需改 suite。但若 M2 的 sanyan_user 测试需要聚合，sanyan_user 包另有自己的 suite——本 task 仅确认 sanyan_chat suite 跑通含 M1 新 group。

- [ ] **Step 1: 跑 suite 确认 M1 新 group 被覆盖**

```bash
cd app && fvm flutter test business_packages/sanyan_chat/test/sanyan_chat_suite.dart
```
Expected: PASS（含 `new_message ACK 回报` group）。

- [ ] **Step 2: 若 sanyan_user 有 suite，加入 register_device_token_req_test**

```bash
cd app && find foundation_packages/sanyan_user/test -name "sanyan_user_suite.dart"
```
有则在其 `main()` 追加 `register_device_token_req_test.main();` 并 import；无则跳过（flutter-business-layer.md §10 的 suite 要求针对 business 包，foundation 包按现状）。

- [ ] **Step 3: Commit（仅当有 suite 改动）**

```bash
git add app/foundation_packages/sanyan_user/test/sanyan_user_suite.dart
git commit -m "test(user): sanyan_user_suite 聚合 device token 注册测试"
```

> **Phase M checkpoint**：跑 sanyan_chat + sanyan_user 两包测试：
> `cd app && fvm flutter test business_packages/sanyan_chat && fvm flutter test foundation_packages/sanyan_user`

---

### Phase N · 集成 + 收尾

> 依赖全部（A-M）。final gate 跑 server 全量 + flutter 全量。

#### Task N1: ERROR_CODE_REGISTRY 补 7000/8000 区间

**Files:**
- Modify: `server/foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`

> **现状（已确认）**：区间总览第 21 行为 `| **7000-9999** | _（保留）_ | — | 留给未来新模块 |`，需拆出 7000-7999 proactive + 8000-8999 push，剩 9000-9999 保留。proactive/push 的 ErrCode 枚举本身在 Phase I（`ProactiveErrCode` 7001/7002）+ Phase G（`PushErrCode` 8001/8002）已建（骨架附录 §F）；本 task 只补登记表。

- [ ] **Step 1（无 TDD，文档）：改区间总览**

把保留行 `| **7000-9999** | _（保留）_ |` 替换为：

```
| **7000-7999** | proactive | `ProactiveErrCode` | `business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/internal/` |
| **8000-8999** | push | `PushErrCode` | `business_packages/sanyan-push-core/src/main/java/com/sanyan/push/internal/` |
| **9000-9999** | _（保留）_ | — | 留给未来新模块 |
```

- [ ] **Step 2: 加明细清单**（在 EmbeddingErrCode 6xxx 段之后）

```
### ProactiveErrCode（7000-7999）

| Code | 常量 | 文案 |
|---|---|---|
| 7001 | `PROACTIVE_GENERATE_FAILED` | 主动消息生成失败 |
| 7002 | `PROACTIVE_EVENT_NOT_FOUND` | 主动事件不存在 |

### PushErrCode（8000-8999）

| Code | 常量 | 文案 |
|---|---|---|
| 8001 | `DEVICE_TOKEN_INVALID` | 设备 token 无效 |
| 8002 | `PUSH_SEND_FAILED` | 推送发送失败 |
```

- [ ] **Step 3: 历史变更加一行**

```
| 2026-05-27 | Plan 4：proactive 域启用 7000-7999（7001/7002）；push 域启用 8000-8999（8001/8002）；9000-9999 保留 |
```

- [ ] **Step 4: 验证 ErrCode 冲突扫描通过**

```bash
cd server && mvn -pl foundation_packages/sanyan-common-error -Dtest=ErrCodeConflictDetectorTest test
```
Expected: PASS（启动期冲突扫描无冲突；枚举值已在 Phase G/I 落地）。

- [ ] **Step 5: Commit**

```bash
git add server/foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md
git commit -m "docs(error): ERROR_CODE_REGISTRY 登记 proactive 7xxx + push 8xxx 区间"
```

---

#### Task N2: bootstrap application.yml 加 sanyan.proactive.* + sanyan.push.apns.* 占位

**Files:**
- Modify: `server/bootstrap/src/main/resources/application.yml`
- Test: `server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactivePropertiesBindingIT.java`

> **现状（已确认）**：`application.yml` 第 35 行起 `sanyan:` 段已有 `jwt` / `intimacy` / `doubao` / `llm` / `embedding`。本 task 加 `sanyan.proactive.*`（骨架附录 §H 全部值）+ `sanyan.push.apns.*` 占位。属配置（TDD 例外），但 proactive properties 写一个 `@SpringBootTest` 绑定 IT 验证 yml → `ProactiveProperties` 映射正确（防 relaxed-binding 拼错 key）。

- [ ] **Step 1: Write the failing test（绑定 IT）**

```java
package com.sanyan.proactive.internal;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * 验证 sanyan.proactive.* yml → ProactiveProperties 绑定正确。
 * 用 @TestPropertySource 内联最小配置，不依赖完整 application.yml（避免 IT 拉全上下文）。
 */
@SpringBootTest(classes = ProactiveProperties.class)
@org.springframework.boot.context.properties.EnableConfigurationProperties(ProactiveProperties.class)
@TestPropertySource(properties = {
        "sanyan.proactive.greeting.morning-cron=0 0 8 * * *",
        "sanyan.proactive.greeting.night-cron=0 30 22 * * *",
        "sanyan.proactive.recall.thresholds-hours=24,72,168",
        "sanyan.proactive.quiet-hours.start=23",
        "sanyan.proactive.quiet-hours.end=8",
        "sanyan.proactive.scatter-window-minutes=30",
        "sanyan.proactive.daily-cap-by-stage=2,3,4,5,6",
        "sanyan.proactive.scenes-by-stage.0.morning=false",
        "sanyan.proactive.scenes-by-stage.0.recall=true",
        "sanyan.proactive.scenes-by-stage.2.morning=true",
})
class ProactivePropertiesBindingIT {

    @Autowired ProactiveProperties props;

    @Test
    void should_bind_thresholds_and_scenes() {
        assertThat(props.getRecall().getThresholdsHours()).containsExactly(24, 72, 168);
        assertThat(props.getScatterWindowMinutes()).isEqualTo(30);
        assertThat(props.sceneSwitchesFor(0).morning()).isFalse();
        assertThat(props.sceneSwitchesFor(0).recall()).isTrue();
        assertThat(props.sceneSwitchesFor(2).morning()).isTrue();
    }
}
```

> 实现者：`ProactiveProperties` 的 accessor（`getRecall().getThresholdsHours()` / `getScatterWindowMinutes()` / `sceneSwitchesFor(int)`）应与 L1/L2 实现里调用的一致（Phase I 定义 + L1 task 补的内嵌 `SceneSwitches`）。如绑定形态有差异以 Phase I 实际 `ProactiveProperties` 为准，同步调整本 IT 断言。

- [ ] **Step 2: Run test to verify it fails**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=ProactivePropertiesBindingIT test
```
Expected: FAIL（yml/properties 还没把这些 key 配齐 / accessor 缺失）。

- [ ] **Step 3: Write minimal implementation（yml）**

在 `application.yml` 的 `sanyan:` 段（intimacy 之后）加：

```yaml
  proactive:
    greeting:
      morning-cron: "0 0 8 * * *"
      night-cron: "0 30 22 * * *"
    recall:
      thresholds-hours: [24, 72, 168]        # 24h / 3天 / 7天
    quiet-hours:
      start: 23
      end: 8
    scatter-window-minutes: 30
    daily-cap-by-stage: [2, 3, 4, 5, 6]       # stage 0..4
    scenes-by-stage:
      0: { morning: false, night: false, recall: true,  event-followup: true,  emotion-care: false }
      1: { morning: false, night: true,  recall: true,  event-followup: true,  emotion-care: true  }
      2: { morning: true,  night: true,  recall: true,  event-followup: true,  emotion-care: true  }
      3: { morning: true,  night: true,  recall: true,  event-followup: true,  emotion-care: true  }
      4: { morning: true,  night: true,  recall: true,  event-followup: true,  emotion-care: true  }

  push:
    apns:
      # 本期 L3 实推未通，占位配置；实接入等 .p8 证书（spec §8 待办）
      enabled: false
      auth-key-path: ${APNS_AUTH_KEY_PATH:}
      key-id: ${APNS_KEY_ID:}
      team-id: ${APNS_TEAM_ID:}
      topic: ${APNS_TOPIC:}                   # bundleId
```

> 实现者：`scenes-by-stage` 的内层 key（morning/night/recall/...）relaxed-binding 下 `event-followup` ↔ `eventFollowup` 字段名映射要对齐 Phase I 的 `SceneSwitches` 字段；若 Phase I 用 `eventFollowup`/`emotionCare` 驼峰字段，yml 用 `event-followup`/`emotion-care` 横线即可自动绑定。

- [ ] **Step 4: Run test to verify it passes**

```bash
cd server && mvn -pl business_packages/sanyan-proactive-core -Dtest=ProactivePropertiesBindingIT test
```

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/main/resources/application.yml \
        server/business_packages/sanyan-proactive-core/src/test/java/com/sanyan/proactive/internal/ProactivePropertiesBindingIT.java
git commit -m "feat(proactive): application.yml 加 sanyan.proactive.* 全量 + sanyan.push.apns.* 占位"
```

---

#### Task N3: 端到端 IT（消息 → 抽取 → 排期 → 调度 → 门控 → 生成 → 投递）

**Files:**
- Create: `server/bootstrap/src/test/java/com/sanyan/bootstrap/ProactiveFlowE2EIT.java`

> **参照**：现有 `server/bootstrap/src/test/java/com/sanyan/bootstrap/MessageFlowE2EIT.java`（Plan 3）——`@SpringBootTest(webEnvironment=NONE)` + `PostgresTestcontainerSupport` + Flyway 真 schema + `@MockBean LlmApi/StringRedisTemplate/RagIndexWorker` + `AopTestUtils.getTargetObject` 绕过 @Async/AFTER_COMMIT 直接调 listener。本 IT 沿用同一套基建。
>
> **链路**：mock `LlmApi`——抽取调用（BACKGROUND）返结构化 JSON（一条 PLAN_EVENT），文案生成调用（USER_FACING）返成文案 → 发 `MessagePersistedEvent`(role=user) → memory 抽取 listener 落 `memory_item(PENDING)` + 发 `MemoryItemScheduledEvent` → proactive `MemoryItemScheduledListener` 排 `events_pending(C_EVENT_FOLLOWUP, scheduledAt 已到)` → 调 `ProactiveScheduler.tick()`（直接调，不等定时）→ 门控放行 → generator → `chatApi.deliverProactiveMessage` → 断言 ①message 落库（role=ai）②events_pending=SENT ③memory_item=DONE。

> **降级实现策略（重要）**：完整链路涉及 memory 抽取的 LLM JSON 解析（Phase F）+ scheduler SKIP LOCKED（Phase J）+ 多 listener 异步，端到端一次性串通脆弱。**推荐分两段直调**（参照 MessageFlowE2EIT「为什么直接调 Listener 而不是发事件」）：
> - 第一段：直接 new `MemoryItemEntity(PENDING)` 存库 + 调 `proactiveScheduler`/`MemoryItemScheduledListener` 把 events_pending 排好（绕过 memory LLM 抽取的不确定性，抽取链路本身在 Phase F 单测/IT 覆盖）
> - 第二段：调 `ProactiveScheduler` 的扫描方法 → dispatcher → generator → deliver，断言三处状态
> - LLM 文案生成用 `@MockBean LlmApi` 返固定串；`chatApi.deliverProactiveMessage` 可用真实 chat-core（需 Postgres）或 `@MockBean` 后单独断言被调用 + 手动落 message。实现者按 Phase H 的 DeliveryService 是否依赖 WS session 决定：E2E 无 WS session（webEnvironment=NONE），离线路径会走 L2 超时→L3，**建议 mock ChatApi.deliverProactiveMessage 返回落库 messageId 列表**，把"真实投递"留给 chat-core 自己的 IT，本 E2E 聚焦 proactive 编排链路。

- [ ] **Step 1: Write the failing test**

```java
package com.sanyan.bootstrap;

import com.sanyan.chat.ChatApi;
import com.sanyan.common.test.PostgresTestcontainerSupport;
import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.memory.event.MemoryItemScheduledEvent;
import com.sanyan.memory.internal.rag.RagIndexWorker;
import com.sanyan.proactive.internal.EventPendingEntity;
import com.sanyan.proactive.internal.EventPendingRepository;
import com.sanyan.proactive.internal.EventStatus;
import com.sanyan.proactive.internal.EventType;
import com.sanyan.proactive.internal.ProactiveScheduler;
import com.sanyan.proactive.trigger.MemoryItemScheduledListener;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.util.AopTestUtils;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.lenient;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;
import static org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase.Replace.NONE;

/**
 * Plan 4 端到端 final gate：memory_item 排期 → events_pending → 调度 → 门控 → 生成 → 投递。
 *
 * <p>参照 MessageFlowE2EIT：webEnvironment=NONE + Testcontainers PG + Flyway 真 schema，
 * mock LlmApi / StringRedisTemplate / RagIndexWorker / ChatApi.deliverProactiveMessage。
 * 直接调 listener / scheduler 方法绕过 @Async / @TransactionalEventListener / @Scheduled 时机。
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureTestDatabase(replace = NONE)
class ProactiveFlowE2EIT extends PostgresTestcontainerSupport {

    @DynamicPropertySource
    static void overrideProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", PostgresTestcontainerSupport::jdbcUrl);
        registry.add("spring.datasource.username", PostgresTestcontainerSupport::username);
        registry.add("spring.datasource.password", PostgresTestcontainerSupport::password);
        registry.add("spring.datasource.driver-class-name", () -> "org.postgresql.Driver");
        registry.add("spring.flyway.enabled", () -> "true");
        registry.add("spring.flyway.locations", () -> "classpath:db/migration");
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
        registry.add("spring.autoconfigure.exclude", () ->
                "org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,"
                        + "org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration");
        // 复用 MessageFlowE2EIT 的必填占位配置（jwt/doubao/cos/tts...）——实现者照抄那份
        registry.add("sanyan.jwt.secret", () -> "test-secret-key-at-least-256-bits-long-for-hmac-sha-testing");
        registry.add("sanyan.jwt.expiration-days", () -> "1");
        registry.add("sanyan.doubao.api-key", () -> "test");
        registry.add("sanyan.doubao.model", () -> "test");
        registry.add("sanyan.doubao.base-url", () -> "http://localhost:9999");
        registry.add("sanyan.cos.secret-id", () -> "test");
        registry.add("sanyan.cos.secret-key", () -> "test");
        registry.add("sanyan.cos.bucket", () -> "test-bucket");
        registry.add("sanyan.cos.region", () -> "ap-guangzhou");
        registry.add("sanyan.tts.enabled", () -> "false");
        registry.add("sanyan.tts.app-id", () -> "test");
        registry.add("sanyan.tts.access-token", () -> "test");
    }

    private static final Long USER_ID = 9051L;
    private static final Long CHARACTER_ID = 1L;

    @MockBean LlmApi llmApi;
    @MockBean StringRedisTemplate stringRedisTemplate;
    @MockBean RagIndexWorker ragIndexWorker;
    @MockBean ChatApi chatApi;

    @Autowired MemoryItemScheduledListener scheduledListener;
    @Autowired ProactiveScheduler proactiveScheduler;
    @Autowired EventPendingRepository eventRepo;
    // @Autowired MemoryItemRepository memoryItemRepo;  // Phase F 的仓储，断言 memory_item=DONE 用

    @Test
    void plan_event_should_flow_to_sent_event_and_done_memory_item() throws Exception {
        // ── 准备：Redis mock（每日上限计数 increment 返回 1，门控放行）
        @SuppressWarnings("unchecked")
        ValueOperations<String, String> valueOps = mock(ValueOperations.class);
        lenient().when(stringRedisTemplate.opsForValue()).thenReturn(valueOps);
        lenient().when(valueOps.get(anyString())).thenReturn(null);
        lenient().when(valueOps.increment(anyString(), anyLong())).thenReturn(1L);

        // ── LLM 文案生成（USER_FACING）返固定串
        when(llmApi.chat(any(LlmTaskType.class), any())).thenReturn("面试顺利吗？");

        // ── 投递 mock：返回落库 messageId（chat-core DeliveryService 的真实投递另有 IT）
        when(chatApi.deliverProactiveMessage(any(), any(), any())).thenReturn(List.of(7777L));

        // 1) 先在 memory_item 落一条 PLAN_EVENT(PENDING)，记其 id（用 Phase F 仓储或 JDBC 直插）
        //    long itemId = memoryItemRepo.save(MemoryItemTestFixtures.planEvent(USER_ID, CHARACTER_ID)).getId();
        long itemId = /* 见 Phase F Fixture */ 0L;

        // 2) 调 listener 排 events_pending（scheduledAt = 过去时刻，确保 scheduler 立即取到）
        var rawListener = AopTestUtils.getTargetObject(scheduledListener);
        ((MemoryItemScheduledListener) rawListener).onMemoryItemScheduled(
                new MemoryItemScheduledEvent(itemId, USER_ID, CHARACTER_ID, "PLAN_EVENT",
                        Instant.now().minusSeconds(60)));

        var pendingBefore = eventRepo.findAll();
        assertThat(pendingBefore)
                .anyMatch(e -> e.getEventType() == EventType.C_EVENT_FOLLOWUP
                        && e.getStatus() == EventStatus.SCHEDULED);

        // 3) 调度一轮（直接调 scheduler 扫描方法，绕过 @Scheduled）
        proactiveScheduler.tick();   // 方法名以 Phase J 实现为准

        // 4) 断言：events_pending=SENT + deliverProactiveMessage 被调 + memory_item=DONE
        assertThat(eventRepo.findAll())
                .anyMatch(e -> e.getEventType() == EventType.C_EVENT_FOLLOWUP
                        && e.getStatus() == EventStatus.SENT);
        org.mockito.Mockito.verify(chatApi).deliverProactiveMessage(any(), any(), any());
        // assertThat(memoryItemRepo.findById(itemId).orElseThrow().getStatus()).isEqualTo(MemoryItemStatus.DONE);
    }
}
```

> 实现者必须补齐被注释的两处（`MemoryItemRepository` + Fixture 落 PLAN_EVENT 拿真实 id；`memory_item=DONE` 断言）——它们依赖 Phase F 实际类名 / Fixture。`proactiveScheduler.tick()` 的方法名以 Phase J `ProactiveScheduler` 实际 @Scheduled 方法名为准。FK：若 V1 schema 的 memory_item 无 FK 到 users 则不用 seed users；若有，照 MessageFlowE2EIT 的 `@BeforeEach seedFkFixtures` 先插 users 行。

- [ ] **Step 2: Run to verify FAIL → Step 3: 补齐桩 → Step 4: PASS**

```bash
cd server && mvn -pl bootstrap -Dtest=ProactiveFlowE2EIT verify
```

- [ ] **Step 5: Commit**

```bash
git add server/bootstrap/src/test/java/com/sanyan/bootstrap/ProactiveFlowE2EIT.java
git commit -m "test(proactive): ProactiveFlowE2EIT 端到端（排期→调度→门控→生成→投递→状态）"
```

---

#### Task N4: final gate

**Files:** 无（验证 + checklist）

- [ ] **Step 1: server 全量**

```bash
cd server && mvn verify
```
Expected: BUILD SUCCESS（所有 `*Test` + `*IT` 绿，含 Enforcer banned-dependencies 不报、ErrCodeConflictDetector 无冲突、各模块 FlywayMigrationSyncTest 通过）。

- [ ] **Step 2: flutter 全量**

```bash
cd app/business_packages/sanyan_chat && fvm flutter test
cd app/foundation_packages/sanyan_user && fvm flutter test
```
Expected: All tests passed。

- [ ] **Step 3: 完成度 checklist（逐项核对）**

- [ ] 4 生成器（K2-K5）单测全绿，supportsType 覆盖 4 个 EventType
- [ ] 4 触发器（L1-L3）+ L0 character-api 缺口补齐，单测覆盖各档 + 去重 + 开关
- [ ] 前端 M1 收到 server 主动消息回 ACK（ackMsgId）；M2 device token 注册结构（实拿 token 标 TODO）
- [ ] ERROR_CODE_REGISTRY 7xxx/8xxx 区间 + 明细 + 历史变更已登记
- [ ] application.yml sanyan.proactive.* 全量 + sanyan.push.apns.* 占位，绑定 IT 通过
- [ ] ProactiveFlowE2EIT 端到端通过
- [ ] `git log` 所有 commit 中文 conventional 前缀、无 AI 署名
- [ ] spec §12 待办项已在代码 TODO 标注（APNs 实接入 / 安卓 SDK / 客户端拿 token / 用户级设置 UI）

- [ ] **Step 4: 无 commit（纯验证 task）**；若 final gate 暴露问题，按 TDD 回到对应 Phase 修复。

---
