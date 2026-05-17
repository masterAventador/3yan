# S3 设计：sanyan-business 单体拆解为 6 对 -api/-core 模块

**日期**：2026-05-17
**作者**：brainstorming 阶段产出
**前置**：Plan 2 R3 final review 的 S3 建议
**后续**：本 design doc 通过后由 writing-plans skill 接手产出实施 plan

## 1. 背景

Plan 2 R3 final review 留了 3 个 minor / suggestion，其中 S3 是：

> `sanyan-business` 仍是大单体（未拆 chat / character / user / llm 四个领域）。Plan 2 妥协依赖 `sanyan-memory-core → sanyan-business` 整包，因为 chat / llm 没拆 -api/-core。当前 `sanyan-memory-core` 直接 import 了 `com.sanyan.chat.internal.MessageEntity` 等，虽然 Maven 边界 OK，但 Java 包级别不符合 java-backend rule §3.3 "禁止其他模块直接 import internal 类" 的精神。

本 design 解决 S3。

## 2. 目标

按 `~/.claude/rules/java-backend.md` + `java-backend-business-layer.md` 把 `sanyan-business` 拆成符合规范的多个业务领域模块，让：

- 每个业务领域成为独立的 `-api` + `-core` 双模块
- 跨模块通信全部走 `-api` 暴露的接口 + DTO，**禁止直接 import 其他 -core 的 internal/**
- Maven Enforcer + ArchUnit 物理隔离 + 代码层守护
- 为 Plan 3（关系养成系统）准备干净的模块边界

## 3. 目标架构

### 3.1 模块清单（12 个新模块，6 对 -api/-core）

```
business_packages/
├── sanyan-user-api/                       # UserApi + UserDto record
├── sanyan-user-core/                      # web/AuthController + LoginReq/Data + RegisterReq + SmsSendReq
│                                          # api/UserApiImpl
│                                          # internal/UserEntity / Repository / UserRegisterService /
│                                          #          UserLoginService / SmsCodeSendService / UserErrCode
│
├── sanyan-character-api/                  # CharacterApi + AiCharacterDto record
├── sanyan-character-core/                 # internal/AiCharacterEntity / Repository / CharacterErrCode
│                                          # api/CharacterApiImpl
│
├── sanyan-chat-api/                       # ChatApi + MessageDto record + SenderType enum (顶层)
│                                          # event/MessagePersistedEvent record
├── sanyan-chat-core/                      # web/MessageController + ws/ChatWebSocketHandler + ws/WsXxx DTO
│                                          # api/ChatApiImpl
│                                          # internal/MessageEntity / Repository / MessageService /
│                                          #          ChatErrCode / PromptBuilder (chat 自己拼 user message)
│
├── sanyan-llm-api/                        # LlmApi + LlmTaskType enum + dto/ChatMessage record
├── sanyan-llm-core/                       # api/LlmApiImpl
│                                          # internal/Router + DoubaoAdapter + DeepSeekAdapter +
│                                          #          LLMProvider + LlmErrCode
│
├── sanyan-embedding-api/                  # EmbeddingApi（单接口业务域，区别于历史的自建 BGE-M3 服务）
├── sanyan-embedding-core/                 # api/EmbeddingApiImpl
│                                          # internal/SiliconFlowEmbeddingProvider + EmbeddingErrCode
│
├── sanyan-memory-api/                     # (已有，不动) MemoryApi + dto/* + event/* + MemoryConstants
└── sanyan-memory-core/                    # (已有，要改 import：走 ChatApi/LlmApi/EmbeddingApi)

foundation_packages/                       # 不增加新模块，只补 config / client 子包
├── sanyan-common-error                    # (已有，不动)
├── sanyan-common-web/                     # 补 client/HttpClientFactory + config/WebMvcConfig
├── sanyan-common-auth                     # (已有，不动)
├── sanyan-common-cache                    # (已有，不动)
├── sanyan-common-storage                  # (已有，不动)
├── sanyan-common-util                     # (已有，不动)
├── sanyan-common-test                     # (已有，不动)
└── sanyan-common-ws/                      # 补 config/WebSocketConfig

bootstrap/                                 # 只剩 SanyanApplication，依赖 6 个 -core
└── src/test/java/ArchitectureTest         # 规则简化为"任何 -core 不依赖其他 -core/internal/" 一条
```

### 3.2 跨模块依赖白名单

```
chat-core      → user-api + character-api + llm-api + memory-api
character-core → (无业务依赖；只用 common-*)
user-core      → (无业务依赖；只用 common-*)
llm-core       → (无业务依赖；只用 common-*)
embedding-core → (无业务依赖；只用 common-*)
memory-core    → chat-api + llm-api + embedding-api
```

任何 `-core` **禁止**依赖其他 `-core` —— 由父 pom Maven Enforcer 物理拒绝 + ArchUnit 双重守护。

### 3.3 sanyan-llm-api 设计原则

- 严格遵守 java-backend §2.2：**一个 -api 一个主接口**
- 这是 embedding 必须独立成 `sanyan-embedding-api/core` 的根本原因——chat 推理与 embedding 是不同领域，强行塞一个接口违反单一职责
- 历史上 Plan 2 时代的"sanyan-embedding-* 自建服务"删除是因为该服务本身（HTTP server / DJL / 模型加载）已下线；本次新建的 `sanyan-embedding-*` 是**业务域抽象**（包装硅基流动 API），概念不同

### 3.4 PromptBuilder 不进 -api 的原因

PromptBuilder 拼 OpenAI 消息的规则在 **chat（短期窗口 + system prompt）** 与 **memory 的 summary / profile 提取** 之间是**完全不同的拼装逻辑**。强行共用会让两边的演化互相牵制。

正确做法：
- chat 域的 PromptBuilder 留在 `sanyan-chat-core/internal/`
- memory 域自己写 `SummaryPromptBuilder` + `ProfilePromptBuilder` 在 `sanyan-memory-core/internal/`
- 共用的"构造 ChatMessage record"的能力下沉到 `sanyan-llm-api/dto/ChatMessage` 静态工厂方法（如果有的话）

### 3.5 ErrCode 全部留在 -core/internal/

修正自 v1 设计：v1 一度提出 "ErrCode 也是契约层，进 -api"。但 java-backend §2.2 明确写 "-api 只含接口 + DTO + 事件定义"，ErrCode enum 不属于这三类。

跨模块识别错误码靠 `BusinessException.getErrCode().getCode()` 拿数字比较——数字本身就是契约，集中管理在 `ERROR_CODE_REGISTRY.md`。

## 4. 拆分流程（7 个 Phase）

### Phase 0 · HttpClientFactory 上提到 foundation

把 `sanyan-business/llm/internal/LlmHttpClients` 改名为 `HttpClientFactory` 迁到 `sanyan-common-web/client/`。

**原因**：llm-core 和 embedding-core 两边都要用 RestClient 工厂；按"代码复用"原则下沉到 foundation。提前做避免后续 Phase 卡在循环依赖。

**改动**：
- 新建 `sanyan-common-web/src/main/java/com/sanyan/common/web/client/HttpClientFactory.java`
- 删 `sanyan-business/llm/internal/LlmHttpClients.java`
- 全工程 `import LlmHttpClients` 改成 `import HttpClientFactory`

**验证**：`mvn verify` 全绿

**Commit**：`refactor(common-web): 抽 HttpClientFactory 下沉到 foundation （Phase 0 / 7）`

### Phase 1 · sanyan-user-api/core

**被依赖**：0（user 是叶子）

**改动**：
- 新建 `sanyan-user-api`：`UserApi` 接口 + `UserDto` record（顶层 `com.sanyan.user`）
- 新建 `sanyan-user-core`：
  - `web/`：AuthController + LoginReq/Data + RegisterReq + SmsSendReq
  - `api/UserApiImpl`
  - `internal/`：UserEntity / Repository / UserRegisterService / UserLoginService / SmsCodeSendService / **UserErrCode（留 internal）**
- 父 pom 加 2 个 module 声明 + dependencyManagement
- bootstrap pom 加 `sanyan-user-core` 依赖

**验证**：`mvn verify` 全绿 + 新增 `UserApplicationContextIT` 冒烟测试

**Commit**：`refactor(modules): 拆 sanyan-user-api/core （Phase 1 / 7）`

### Phase 2 · sanyan-character-api/core

**被依赖**：chat 和 llm 引 `character.internal.*`

**改动**：
- 新建 `sanyan-character-api`：`CharacterApi` 接口 + `AiCharacterDto` record
- 新建 `sanyan-character-core`：`internal/AiCharacterEntity / Repository / CharacterErrCode` + `api/CharacterApiImpl`
- `sanyan-business` 仍存在，但 chat + llm 两包改 import：
  - `character.internal.AiCharacterEntity` → `AiCharacterDto`
  - `character.internal.AiCharacterRepository.findById` → `characterApi.findById(...)`
  - `character.internal.CharacterErrCode` 不再跨包用：character 自己抛 `BusinessException(CharacterErrCode.x)`，调用方 catch `BusinessException` 看 `errCode.getCode() == 3001`

**验证**：`mvn verify` 全绿

**Commit**：`refactor(modules): 拆 sanyan-character-api/core，chat/llm 改走 CharacterApi （Phase 2 / 7）`

### Phase 3 · sanyan-llm-api/core （不含 embedding）

**被依赖**：chat（AiService）+ memory-core（4 处）

**改动**：
- 新建 `sanyan-llm-api`：`LlmApi` 接口 + `LlmTaskType` enum + `dto/ChatMessage` record
- 新建 `sanyan-llm-core`：
  - `internal/`：Router + DoubaoAdapter + DeepSeekAdapter + LLMProvider + **LlmErrCode（留 internal）**
  - `api/LlmApiImpl`（委托 Router）
- SiliconFlowEmbeddingProvider 暂时仍在 `sanyan-business`（Phase 4 才拆 embedding 域）
- chat + memory-core 改 import：
  - `LLMProviderRouter` → `LlmApi`
  - `LLMTaskType` → 顶层 `com.sanyan.llm.LlmTaskType`
  - PromptBuilder 复杂：
    - chat 域 PromptBuilder 留 `sanyan-business/chat/internal/`
    - memory-core 自己写 `SummaryPromptBuilder` / `ProfilePromptBuilder`，放 `sanyan-memory-core/internal/`

**验证**：`mvn verify` 全绿

**Commit**：`refactor(modules): 拆 sanyan-llm-api/core + PromptBuilder 各业务方独立 （Phase 3 / 7）`

### Phase 4 · sanyan-embedding-api/core

**被依赖**：memory-core（`EmbeddingProvider` 一处）

**改动**：
- 新建 `sanyan-embedding-api`：`EmbeddingApi` 接口（`embed(List<String>) → List<float[]>`）
- 新建 `sanyan-embedding-core`：
  - `internal/SiliconFlowEmbeddingProvider`（从 sanyan-business 迁过来）
  - `internal/EmbeddingErrCode`（从 LlmErrCode 拆出 4004 → 6001 重新编号）
  - `api/EmbeddingApiImpl`（委托 SiliconFlowEmbeddingProvider）
- `ERROR_CODE_REGISTRY.md` 更新：6xxx 区间从"保留"改成 embedding，4004 退役
- memory-core 改 import：`EmbeddingProvider` → `EmbeddingApi`

**验证**：`mvn verify` 全绿（含 `MemoryRagSearchServiceTest` 两条 fallback case 验证新 ErrCode）

**Commit**：`refactor(modules): 拆 sanyan-embedding-api/core （Phase 4 / 7）`

### Phase 5 · sanyan-chat-api/core （最复杂）

**被依赖**：memory-core（5 处 import `chat.internal.*`）

**改动**：
- 新建 `sanyan-chat-api`：
  - `ChatApi` 接口（findById / listRecent 等）
  - `MessageDto` record
  - `SenderType` enum（上提到 chat-api 顶层包，让 ws DTO 和 service 共用）
  - `event/MessagePersistedEvent` record
- 新建 `sanyan-chat-core`：
  - `web/MessageController` + `ws/ChatWebSocketHandler` + `ws/WsXxx` DTO
  - `api/ChatApiImpl`
  - `internal/`：MessageEntity / Repository / MessageService / **ChatErrCode（留 internal）**
  - `internal/PromptBuilder`（chat 自己的，从 sanyan-business 搬过来）
- memory-core 改 import（5 个 service / listener / scheduler）：
  - `chat.internal.MessageEntity` → `MessageDto`
  - `chat.internal.MessageRepository.find*` → `ChatApi.listRecent(...)`
  - `chat.internal.SenderType` → `com.sanyan.chat.SenderType`（顶层）
  - `chat.event.MessagePersistedEvent` → `com.sanyan.chat.event.MessagePersistedEvent`
  - 影响范围：MemorySummaryService / SummaryScheduler / MemoryProfileRefreshService / MessageEmbeddingIndexListener / UserMessageProfileRefreshListener / MemoryChunkBuilder

**验证**：`mvn verify` 全绿（含 memory-core 所有 IT + Plan2EndToEndIT）

**Commit**：`refactor(modules): 拆 sanyan-chat-api/core + memory-core 全部改走 ChatApi （Phase 5 / 7）`

### Phase 6 · 删 sanyan-business + 配置类下沉 foundation

**改动**：
- 经过 Phase 1-5，`sanyan-business` 只剩 `config/WebMvcConfig` + `config/WebSocketConfig`
- WebMvcConfig 迁到 `sanyan-common-web/src/main/java/com/sanyan/common/web/config/`
- WebSocketConfig 迁到 `sanyan-common-ws/src/main/java/com/sanyan/common/ws/config/`
- 删 `business_packages/sanyan-business` 整个目录 + 父 pom module 声明 + dependencyManagement
- bootstrap pom 删 `sanyan-business` 依赖

**验证**：`mvn verify` 全绿

**Commit**：`refactor(modules): 删 sanyan-business 空壳 + 配置类下沉 foundation （Phase 6 / 7）`

### Phase 7 · ArchUnit 升级 + Maven Enforcer 加守护 + 文档

**改动**：
- ArchUnit 老的"业务域不依赖 web/ws"规则全删（Maven 边界物理强制）
- 新增 ArchUnit 规则："任何 -core 不依赖其他 -core 包的 internal/"
- 父 pom 加 Enforcer `bannedDependencies`：任何 `*-core` 不允许引另一个 `*-core`
- `ERROR_CODE_REGISTRY.md` 更新：
  - 每个 ErrCode 的位置改成新模块路径
  - 6xxx 区间从"保留"改回 embedding（Phase 4 启用了）
  - 删 4004 EMBEDDING_SERVICE_UNAVAILABLE，迁到 6001
  - memory-core 自己的 5002 保留

**验证**：`mvn verify` 全绿

**Commit**：`refactor(modules): ArchUnit + Enforcer 守护 + 文档同步 （Phase 7 / 7）`

### Phase 依赖图

```
Phase 0 (HttpClientFactory 上提)
   ├──→ Phase 1 (user-api/core)        [叶子，独立]
   ├──→ Phase 2 (character-api/core)
   │           └──→ chat / llm（仍在 business）改 import 到 character-api
   ├──→ Phase 3 (llm-api/core)
   │           └──→ chat（仍在 business）+ memory-core 改 import 到 llm-api
   ├──→ Phase 4 (embedding-api/core)
   │           └──→ memory-core 改 import 到 embedding-api
   ├──→ Phase 5 (chat-api/core) ★ 最复杂节点
   │           └──→ memory-core 改 5 处 import 到 chat-api
   ├──→ Phase 6 (删 business 空壳 + config 下沉)
   └──→ Phase 7 (ArchUnit/Enforcer 守护 + 文档同步)
```

## 5. 每个 Phase 的执行节奏（superpowers 子代理模式）

按 `superpowers:subagent-driven-development` 规范，每个 Phase 派 3 个子代理：

### Step 1 · 实现子代理（`subagent_type=general-purpose`）

Prompt 必含：
- 完整 TDD 铁律说明
- 本 Phase 在 design doc 中对应章节
- 当前 Phase 应做改动清单 + 验证命令（`mvn verify`）
- 关键约束：仅本 Phase 改动；不顺手改其他东西

子代理执行 Red → Green → Refactor，报告"改动 N 文件 / 新建 M 个测试 / verify 通过"。

### Step 2 · spec 审查子代理（`subagent_type=general-purpose`）

Prompt 必含：
- design doc 对应章节
- 实现子代理改了哪些文件（`git diff --stat`）
- 检查项：是否漏了 / 是否做了 design 外的改动 / DTO 字段是否完整 / ErrCode 位置是否正确

有 issue → 回 Step 1 修 → 再来 Step 2。

### Step 3 · 代码质量审查子代理（`subagent_type=superpowers:code-reviewer`）

Prompt 必含：
- `git diff` 文件清单
- 检查项：-api 模块是否只含接口/DTO/事件（ErrCode 不应进 -api）/ 是否有 internal/ 泄露 / 命名规范

有 issue → 回 Step 1 修 → 再来 Step 3。

### Step 4 · 全部 PASS 后

`git commit` + push，更新 task 状态，进入下一 Phase。

## 6. 重构任务的 TDD 节奏

本任务 = 重构（业务逻辑零变化，只挪动 + 改 import + 加抽象层）。

TDD 铁律里"先写失败测试再写实现"的标准节奏在这里适配：
- **业务逻辑测试**：全部已存在（Plan 1+2 留下的 389+ 个测试就是 safety net）
- **新增测试**：仅"边界规则测试"（新模块 `ApplicationContextIT` 冒烟 + ArchUnit 断言）

实际节奏（符合 TDD 铁律的解读）：
- **Red**：先在新模块加一条 ArchUnit 断言 / Bean 注入冒烟测试，跑 → fail
  - 例：Phase 2 加 "sanyan-character-api 模块只能含接口/DTO/事件"
  - 例：Phase 4 加 "sanyan-embedding-api 模块只含 EmbeddingApi 一个接口"
- **Green**：实施迁移 + 调整代码让该断言通过
- **全部 mvn verify**：旧业务测试全部仍然 PASS = 业务逻辑零回归保护

Phase 0 比较特殊（单类下沉）：
- **Red**：common-web 内加 `HttpClientFactoryTest`（验证 `newClient(...)` 返回非 null + 注入 timeout 生效）
- **Green**：把 LlmHttpClients 改名搬过去
- **mvn verify**：业务测试不动

## 7. 异常处理 / 风险预案

| 风险 | 预案 |
|---|---|
| 中途某 Phase 的 `mvn verify` 卡住（依赖循环 / Bean 找不到） | 回滚该 Phase 的 commit；重新拆边界；不影响已 merge 的前序 Phase |
| memory-core 改 import 后业务行为变了（Entity → DTO 字段映射漏了） | memory-core 现有 IT（MemoryRagSearchServiceIT / Plan2EndToEndIT）立即捕获；`mvn verify` 失败 = stop the line |
| JPA Entity 跨包后 Hibernate scan 找不到 `@Entity` | bootstrap 的 `@EntityScan` / `@EnableJpaRepositories` 用 `basePackages = "com.sanyan"`（已经是）；改包名映射自动覆盖 |
| ws/ 包跨模块拆分后 Spring WebSocket 注册路径变化 | WebSocketConfig（在 common-ws）+ ChatWebSocketHandler（在 chat-core）不在同一 jar，要靠 Spring auto-config 拼起来；端点 `/ws` 不变；前端 ws url 不变 |
| 数据库 schema 在拆模块中遭意外迁移 | Flyway migrations 文件位置不变（bootstrap/src/main/resources/db/migration）；本次 0 个新 V*.sql；schema 完全无改动 |
| 线上服务部署后行为异常 | 合 master 前本机 dogfood 跑一遍（rag / profile / summary / throttle 4 个场景）；本次零业务逻辑改动，理论上 dogfood 行为完全等同 Plan 2 终态 |
| Phase 0 HttpClientFactory 重命名后所有 import 漏改 | `grep "LlmHttpClients"` 全工程命中应为 0；compile fail 立即定位 |
| Phase 4 EmbeddingErrCode 从 4004 改 6xxx 后调用方比较 code 值错乱 | MemoryRagSearchService 现有 fallback 测试（两条 case）验证两个 ErrCode 都能识别 |
| Phase 3 PromptBuilder 在 memory-core 自己写两份后跟 chat 的不一致 | 本来就不应该共享（拼装规则不同）；各自的 PromptBuilderTest 单独维护，无跨模块约束 |
| Phase 5 chat-core 拆完后 ws 帧序列化（jackson）找不到 SenderType 枚举 | SenderType 上提到 chat-api 顶层包；ws DTO 和 service 共用同一 enum class，无问题 |

## 8. 完成判定（Definition of Done）

- **Maven 模块清单**：`business_packages/` 下从 1 个 sanyan-business → 6 对 -api/-core（共 12 个模块）+ sanyan-business 目录消失
- **Maven Enforcer**：父 pom 增加 `bannedDependencies` "任何 *-core 不依赖另一个 *-core"，`mvn validate` 阶段守护
- **ArchUnit**：bootstrap/ArchitectureTest 旧的 "业务域不依赖 web/ws" 规则全删（已被 Maven 边界物理隔离），新增 "任何 -core 不依赖其他 -core 包的 internal/" 规则
- **跨模块通信**：`grep "import com\.sanyan\.<other-domain>\.internal\."` 命中数 = 0
- **JPA Entity / Repository**：全部留在各自 -core/internal/，0 个出现在 -api 模块
- **ErrCode**：全部留在 -core/internal/；`ERROR_CODE_REGISTRY.md` 同步
- **事件类**：发布方的 -api 模块（如 chat-api/event/MessagePersistedEvent），订阅方 -core/event/ 监听
- **foundation 下沉**：HttpClientFactory 在 sanyan-common-web/client/，WebMvcConfig 在 sanyan-common-web/config/，WebSocketConfig 在 sanyan-common-ws/config/
- **测试**：`mvn verify` 全绿——所有 389+ 个测试 PASS；每个 -core 新建至少 1 个 ApplicationContextIT 冒烟
- **dogfood**：本机跑 4 个场景 PASS（profile / throttle / summary / rag）
- **线上**：merge master + deploy.sh + 健康检查通过

## 9. 文档产出

1. **本 design doc**（`docs/superpowers/specs/2026-05-17-s3-business-modular-split-design.md`）
2. **实施 plan**（writing-plans skill 产出，将放 `docs/superpowers/plans/2026-05-17-s3-business-modular-split-plan.md`），7 个 Phase × 实现/spec审/code审 子任务
3. **ERROR_CODE_REGISTRY.md 更新**（Phase 4 启用 6xxx + Phase 7 同步位置字段）
4. **拆完后终审报告**（路径：`docs/superpowers/reviews/<完成日期>-s3-final-review.md`，完成日期以 R-task 实际执行那天的日期为准），S3 完成后由 R-task 子代理写

## 10. 时间估算

| Phase | 复杂度 | 实现+审查总耗时 |
|---|---|---|
| Phase 0 抽 HttpClientFactory 到 common-web | 低 | ~15 min |
| Phase 1 user-api/core | 低 | ~20 min |
| Phase 2 character-api/core | 中 | ~30 min |
| Phase 3 llm-api/core（PromptBuilder 各方独立） | 高 | ~50 min |
| Phase 4 embedding-api/core | 低-中 | ~25 min |
| Phase 5 chat-api/core（memory-core 全部改 DTO） | 很高 | ~70 min |
| Phase 6 删 business + config 下沉 | 低 | ~20 min |
| Phase 7 ArchUnit + Enforcer + 文档同步 | 中 | ~25 min |
| **总计** | | **~4.4 小时**（含子代理实现 + spec 审 + code 审 + 反馈循环） |

建议拆 2-3 个 session 分批做完：
- **Session 1 = Phase 0-3**（~2h）
- **Session 2 = Phase 4-5**（~1.5h）
- **Session 3 = Phase 6-7**（~45 min）

## 11. 不在范围内

- **Plan 3 任何功能**：本次只是结构重整，零业务行为变化
- **schema 改动**：0 个新 Flyway migration
- **外部依赖升级**：Spring Boot / pgvector / 其他第三方包版本全部不动
- **API 协议变更**：app/web 端调用的 HTTP/WS endpoint 完全不变
- **性能优化**：不在本次范围
- **新业务模块（如 relationship / intimacy）**：是 Plan 3 的范围，本次不动

## 12. 后续

本 design doc 通过用户审查后，由 writing-plans skill 接手产出 `docs/superpowers/plans/2026-05-17-s3-business-modular-split-plan.md`，按 7 个 Phase 拆成可执行的 task 清单。
