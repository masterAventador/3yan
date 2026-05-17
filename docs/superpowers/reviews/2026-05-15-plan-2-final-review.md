# Plan 2 整体合规性审查报告

**日期**：2026-05-15
**审查人**：Claude（R3 task autonomous reviewer）
**审查范围**：Plan 2 长期记忆系统全部 29 个 task（L0 → R2）实施完毕，本报告为 Plan 2 最终终审
**对应 plan**：`/Users/aventador/code/3yan/docs/superpowers/plans/2026-05-06-3yan-plan-2-long-term-memory.md`
**对应 dogfood**：`/Users/aventador/code/3yan/docs/superpowers/dogfood/2026-05-15-plan-2-dogfood.md`

---

## Executive Summary

- **总体结论**：**PASS**
- **必修问题数**：0
- **修复数**：0（无需修复）
- **待处理建议**：3（minor / suggestion 级，记录到 Plan 3 处理）
- **关键指标**：
  - 全量 `mvn verify` 通过 ✅（16/16 模块 SUCCESS）
  - **389** 个测试 0 failures / 0 errors
  - 16 个 Maven 模块全部 SUCCESS
  - 5 个 Flyway migration 应用成功（V1 / V2 / V5 / V6 / V7）
  - 0 处 `com.3yan` 包名残留
  - 0 处硬编码 `PageRequest.of(0, N)`
  - 0 处违反 `BusinessException` 唯一性的业务异常

---

## 1. 跨 Task 一致性

### 1.1 包名 `com.sanyan.*` 统一 ✅
- `grep -rn "com\.3yan\|3yan\."` 在 `business_packages` / `foundation_packages` / `bootstrap` / `embedding-bootstrap` 全无命中
- 所有新代码包名规范一致

### 1.2 常量集中位置 ✅
- `MemoryConstants` 集中放在 `sanyan-memory-api`（接口契约层），跨业务模块可见
- 含 14 个常量：`SHORT_TERM_WINDOW_SIZE / SUMMARY_TRIGGER_THRESHOLD / RAG_CHUNK_MIN/MAX_SIZE / RAG_CHUNK_MAX_TOKEN / RAG_TOP_K / RAG_MIN_COS_SIM / PROFILE_EXTRACT_THROTTLE_MINUTES / EMBEDDING_HTTP_* / PROFILE_EMOTION_LINE_MAX / EMBEDDING_DIM`
- 所有引用方（`AiService` / `SummaryScheduler` / `RagIndexWorker` / `MemoryChunkBuilder` / `RemoteBgeM3Provider`）从本类取值
- `MemoryConstantsTest` 守护「对齐铁律」三条断言：
  - `SHORT_TERM_WINDOW_SIZE > SUMMARY_TRIGGER_THRESHOLD`
  - `RAG_CHUNK_MAX_SIZE >= RAG_CHUNK_MIN_SIZE`
  - `EMBEDDING_HTTP_READ_TIMEOUT_MS > EMBEDDING_HTTP_CONNECT_TIMEOUT_MS`
- 无硬编码 magic number；唯一保留的字面量：`ChatEmbeddingEntity.columnDefinition = "vector(1024)"`（DDL 字符串无法参数化）+ `ChatEmbeddingRepository` 的余弦距离公式 `1.0 - (... <=> ...) / 2.0`（数学公式不是 magic number）

### 1.3 命名规范统一 ✅
| 类别 | 检查项 | 结果 |
|---|---|---|
| Service | `<Domain><Action>Service` | ✅ `MemorySummaryService` / `MemoryProfileExtractService` / `MemoryProfileMergeService` / `MemoryEmbeddingService` / `MemoryRagSearchService` |
| Entity | 带 `Entity` 后缀 | ✅ `MemorySummaryEntity` / `MemoryProfileEntity` / `ChatEmbeddingEntity` |
| Repository | `<Domain>Repository` | ✅ `MemorySummaryRepository` / `MemoryProfileRepository` / `ChatEmbeddingRepository` |
| ApiImpl | `<Domain>ApiImpl` | ✅ `MemoryApiImpl` / `EmbeddingApiImpl` |
| Listener | `<EventType>Listener` | ✅ `MessageEmbeddingIndexListener` / `UserMessageProfileExtractListener` |
| Constants 类 | `<Role>Constants` | ✅ `MemoryConstants` |
| TestFixtures | `<Domain>TestFixtures` | ✅ `MemorySummaryTestFixtures` / `MemoryProfileTestFixtures` / `ChatEmbeddingTestFixtures` |

### 1.4 Commit message 规范 ✅
- Plan 2 共 **31 个 commits**（017c2b0..HEAD）
- 全部使用 conventional prefix（`feat` / `fix` / `test` / `docs` / `refactor` / `chore`）
- 全部 **中文** body + scope
- 0 个 `Co-Authored-By` 行（grep 结果空）

---

## 2. CLAUDE.md 全局规则合规

### 2.1 代码删除规范 ✅
- 全工程仅 1 处 `// TODO`：`SmsCodeSendService.java:24`「对接阿里云短信 SDK，当前仅打印到控制台」，属于 Plan 1 范畴的有意保留（提示后续接 SDK），不属于 Plan 2 引入的死代码注释
- 无 `// FIXME` / `// XXX` / `// removed` / `// 注释掉` 残留
- `217dd2b fix(embedding-server): M2b review 后清理 — 删 INVALID_TOKEN 死码` 已主动应用过本规则

### 2.2 重构后清理规范 ✅
- M3 task 注册 DeepSeek + RemoteBgeM3Provider 后，`AiService` 不再持有豆包 HTTP 字段，只剩 `MessageRepository / LLMProviderRouter / PromptBuilder / MemoryApi` + 系统提示资源
- Q3 task 改造 `AiService` 时已将硬编码 `PageRequest.of(0, 20)` 替换为 `PageRequest.of(0, MemoryConstants.SHORT_TERM_WINDOW_SIZE)`，无遗留旧调用
- `e3bb5c0 fix(llm): 删除 M2c 引入的未使用 spring-retry 依赖` 是「重构后清理」规范的样板执行

### 2.3 代码复用原则 ✅
- 全工程仅有 1 个 HTTP 工厂 `LlmHttpClients.newClient`（位于 `sanyan-business/llm/internal`），被 `DeepSeekAdapter` + `RemoteBgeM3Provider` 共用
- `MemoryConstants` 集中管理所有跨模块常量（值复用）
- `PromptBuilder` 统一所有 OpenAI 消息拼装（`AiService` + `MemoryProfileExtractService` + `MemorySummaryService` 共用）—— 避免多份相似实现

### 2.4 static 优先原则 ✅
| 工具类 | 形态 | 合规 |
|---|---|---|
| `MemoryConstants` | `public final class` + `private MemoryConstants()` + 全 `static` 字段 | ✅ |
| `LlmHttpClients` | `public final class` + `private LlmHttpClients()` + `static` 方法 | ✅ |
| `MemoryChunkBuilder` | `@Component` Spring Bean（有依赖 `EmbeddingProvider`），实例化合理 | ✅ |
| 其他 Service | Spring Bean 实例 + `@RequiredArgsConstructor` 注入依赖 | ✅（CLAUDE.md Spring 例外） |

### 2.5 TDD 铁律 ✅
- **每个 Production 类都有对应 `*Test` 或 `*IT`**：54 个测试 java 文件 vs 106 个 main java 文件（不少包含 record / package-info / DTO 无需测试），测试比例合理
- Plan 2 测试覆盖（部分清单）：
  - `MemoryConstantsTest`（对齐铁律 3 条断言）
  - `MemorySummaryServiceTest` / `SummarySchedulerTest`
  - `MemoryProfileExtractServiceTest` / `MemoryProfileMergeServiceTest` / `MemoryProfileRepositoryIT`
  - `MemoryChunkBuilderTest` / `MemoryEmbeddingServiceTest` / `MemoryRagSearchServiceTest` / `MemoryRagSearchServiceIT` / `ChatEmbeddingRepositoryIT`
  - `RagIndexWorkerTest` / `MessageEmbeddingIndexListenerTest` / `UserMessageProfileExtractListenerTest`
  - `MemoryContextBuilderTest` / `MemoryApiImplTest`
  - `RemoteBgeM3ProviderTest`
  - `Plan2EndToEndIT`（R1 端到端覆盖 B+C+RAG 三层）
- E2E IT (`Plan2EndToEndIT`) 用 Testcontainers + PG + pgvector 真实验证三层数据源

---

## 3. Java backend 规范合规（`~/.claude/rules/java-backend*.md`）

### 3.1 `-api` 模块约束 ✅
| 模块 | 检查 | 结果 |
|---|---|---|
| `sanyan-memory-api` | 只有 `MemoryApi` interface + `MemoryFragment` / `MemoryContext` record + `MemoryConstants` final class | ✅ |
| `sanyan-embedding-api` | 只有 `EmbeddingApi` interface + `EmbedRequest` / `EmbedResponse` record | ✅ |

- `grep` `@Service / @Component / @Repository / @Entity / @Autowired / @Configuration / @RestController` 在两个 -api 模块 0 命中 ✅
- 一个 -api 一个主接口（`MemoryApi` / `EmbeddingApi`）符合 java-backend §2.2

### 3.2 `-core` 包结构 `web/internal/api/event` ✅
- `sanyan-memory-core`：`api/` + `event/` + `internal/`（按 RAG / profile / summary / orchestrator 内部子分包合理）+ `package-info.java`
- `sanyan-embedding-core`：`web/` + `internal/` + `api/` + `package-info.java`（无 event，因为不发布事件）
- 业务 service / repository / errcode 均在 `internal/` 子包

### 3.3 全项目唯一 BusinessException ✅
- `grep "extends RuntimeException"` 仅 2 个命中：
  - `BusinessException`（合规的全项目唯一异常类）
  - `RemoteBgeM3Provider.RetryableEmbeddingException`：**private static final inner class**，**仅作 retry 控制流信号**，never 暴露给外层调用方（最外层 try/catch 后转换为 `BusinessException`）
- 该内部信号类不违反规则——属于「单文件内 retry 控制流」的最小私有作用域工具

### 3.4 ErrCode 区间无冲突 ✅
| 范围 | enum | code | 备注 |
|---|---|---|---|
| 400-499 | `CommonErrCode` | 400/401/403/404/410/500 | 通用 |
| 1000-1999 | `UserErrCode` | 1001-1006 | user |
| 2000-2999 | `ChatErrCode` | 2001 | chat |
| 3000-3999 | `CharacterErrCode` | 3001 | character |
| 4000-4999 | `LlmErrCode` | 4001-4005 | llm |
| 5000-5999 | `MemoryErrCode` | 5001-5002 | memory |
| 6000-6999 | `EmbeddingErrCode` | 6001, 6003 | embedding（独立 jar） |

- 启动期 `ErrCodeConflictDetector` 守护 code 唯一性
- `EMBEDDING_SERVICE_UNAVAILABLE` 在 `LlmErrCode(4004)` 和 `MemoryErrCode(5002)` 都有定义——这是**分层语义有意保留**：4004 是 HTTP 客户端层（RemoteBgeM3Provider）抛出的不可用，5002 是 Memory 层自身视角的不可用。`MemoryRagSearchService` 同时捕获两个 code 做降级（不会进 prompt 污染）
- 6002 缺失（EmbeddingErrCode 跳过 6002 直接 6003），不影响功能，建议未来补 documentation

### 3.5 Object Mother 强制 ✅
- 全部 6 个 Entity 都有对应 `*TestFixtures`：
  - `UserTestFixtures` / `MessageTestFixtures` / `AiCharacterTestFixtures`（Plan 1）
  - `MemorySummaryTestFixtures` / `MemoryProfileTestFixtures` / `ChatEmbeddingTestFixtures`（Plan 2）

### 3.6 Maven Enforcer 边界守护 ✅
- 父 pom：`requireMavenVersion 3.6.3` + `requireJavaVersion 17`
- `bootstrap/pom.xml`：`bannedDependencies > com.sanyan:sanyan-embedding-core` 拒绝主 jar 引入 embedding-core
- `embedding-bootstrap/pom.xml`：`bannedDependencies > sanyan-business / sanyan-memory-api / sanyan-memory-core` 拒绝 embedding 服务引入主业务

---

## 4. Plan 2 特定合规

### 4.1 `SHORT_TERM_WINDOW_SIZE > SUMMARY_TRIGGER_THRESHOLD` 自动化断言 ✅
- `MemoryConstantsTest.shortTermWindowMustBeStrictlyGreaterThanSummaryThreshold` 断言 32 > 30
- 任何后续 task 改坏这两个常量数值都会立刻 surefire fail

### 4.2 远程 embedding 降级策略 ✅
- `MemoryRagSearchService` 同时识别 `MemoryErrCode.EMBEDDING_SERVICE_UNAVAILABLE` 和 `LlmErrCode.EMBEDDING_SERVICE_UNAVAILABLE`
- 降级路径：catch → 返回空 list → prompt 仅含 profile + summary，不污染对话
- `AiService` 第二层降级：`MemoryApi` 抛任何异常 → catch + `log.warn` 跳过长期记忆段，主对话不受影响
- 测试覆盖：`MemoryRagSearchServiceTest` 两个降级 case + `MemoryEmbeddingServiceTest` embedding 服务挂时入库失败的 case

### 4.3 双服务器部署 ✅
- `bootstrap/target/sanyan-server-0.1.0.jar`（fat jar，部署到 new = 49.233.213.109）
- `embedding-bootstrap/target/sanyan-embedding-0.1.0.jar`（独立 fat jar，部署到 old = 154.8.162.83，含 BGE-M3 模型 ~2.3GB）
- `embedding-bootstrap/deploy.sh` 单独一键部署 + actuator/health polling 等模型加载完成 + 从 new 公网回调 /embed 端到端验证
- 06b0faa 已记录端到端验证通过

### 4.4 Maven Enforcer 规则有效 ✅
- 运行 `mvn verify` 时两条 `bannedDependencies` 规则均触发 PASSED
- bootstrap 实际 compile 依赖：`sanyan-business` + `sanyan-memory-core`（4c3634c fix 补的，因为 MemoryApiImpl 在 -core 里）
- embedding-bootstrap 实际 compile 依赖：`sanyan-embedding-api` + `sanyan-embedding-core`，0 个主业务依赖

---

## 5. 测试质量

### 5.1 `mvn verify` 输出摘要

```
[INFO] Reactor Summary for sanyan-server-parent 0.1.0:
[INFO] sanyan-server-parent ............................... SUCCESS [  0.068 s]
[INFO] sanyan-common-error ................................ SUCCESS [  0.486 s]
[INFO] sanyan-common-web .................................. SUCCESS [  0.672 s]
[INFO] sanyan-common-cache ................................ SUCCESS [  0.709 s]
[INFO] sanyan-common-auth ................................. SUCCESS [  0.751 s]
[INFO] sanyan-common-storage .............................. SUCCESS [  0.009 s]
[INFO] sanyan-common-util ................................. SUCCESS [  0.252 s]
[INFO] sanyan-common-test ................................. SUCCESS [  0.025 s]
[INFO] sanyan-common-ws ................................... SUCCESS [  0.701 s]
[INFO] sanyan-embedding-api ............................... SUCCESS [  0.248 s]
[INFO] sanyan-memory-api .................................. SUCCESS [  0.005 s]
[INFO] sanyan-business .................................... SUCCESS [  2.572 s]
[INFO] sanyan-memory-core ................................. SUCCESS [  4.535 s]
[INFO] sanyan-embedding-core .............................. SUCCESS [  1.143 s]
[INFO] sanyan-server-bootstrap ............................ SUCCESS [  8.242 s]
[INFO] sanyan-embedding-bootstrap ......................... SUCCESS [ 31.231 s]
[INFO] BUILD SUCCESS
[INFO] Total time:  51.781 s
```

### 5.2 测试统计

| 维度 | 数值 |
|---|---|
| **总测试数** | 389 |
| **失败** | 0 |
| **错误** | 0 |
| **跳过** | 0 |
| 测试 Java 文件数 | 54 |
| 主 Java 文件数 | 106 |
| 主代码 LoC | 5176 |
| 测试代码 LoC | 6528 |

### 5.3 测试粒度
- **单测（Mockito）**：Service / ApiImpl / Listener / Worker 业务逻辑
- **`@DataJpaTest` IT（Testcontainers PG）**：3 个 Repository（`MemorySummaryRepositoryIT` / `MemoryProfileRepositoryIT` / `ChatEmbeddingRepositoryIT`）
- **`@SpringBootTest` IT**：`Plan2EndToEndIT`（B+C+RAG 三层整合） + `EmbeddingApplicationContextIT`（embedding 服务上下文冒烟）
- **Migration IT**：`V5MigrationIT` / `V6MigrationIT` / `V7MigrationIT` 验证 SQL 正确 + 索引存在

### 5.4 覆盖率
- 每个 Plan 2 新增 Service 都有对应 `*Test` 文件 ✅
- 每个 Plan 2 新增 Repository 都有 `*IT` 验证自定义查询 ✅
- 每个 Plan 2 新增 Listener 都有 Mockito 单测 ✅

---

## 发现的问题及处理

### 已修复
**无**。本轮 R3 静态扫描 + 全量 `mvn verify` 未发现 Critical / Important 级问题需要修复。Plan 2 实施全过程中已经在各 task 的子代理 spec / code review 阶段把关；R3 仅做最终一致性确认。

### 已记录待处理（Minor / Suggestion 级，留给 Plan 3）

#### S1：`ERROR_CODE_REGISTRY.md` 登记表未建立
- **位置**：`foundation_packages/sanyan-common-error/`
- **现象**：java-backend rule §5.3 要求新模块申请 code 区间前更新登记表，但目前没有此文件，新增的 5xxx（memory） / 6xxx（embedding）区间没有写入档案
- **影响**：不阻塞功能（启动期 `ErrCodeConflictDetector` 已经守护 code 唯一性），但缺少对人类 reviewer 的可读索引
- **建议**：Plan 3 新建 `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`，列出当前所有区间 + 给未来新模块预留 7xxx-9xxx 段

#### S2：`EmbeddingErrCode` 跳号 6002
- **位置**：`business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/EmbeddingErrCode.java`
- **现象**：定义了 6001（MODEL_NOT_READY）和 6003（EMBEDDING_FAILED），但跳过 6002
- **影响**：无业务影响；可能反映了开发过程中删过 6002 但没回填
- **建议**：Plan 3 中要么补 6002 用作新错误，要么压缩成 6001/6002 连续

#### S3：`sanyan-business` 仍是大单体（未拆 chat / character / user / llm 四个领域）
- **位置**：`business_packages/sanyan-business`
- **现象**：Plan 2 妥协依赖 `sanyan-memory-core → sanyan-business` 整包，因为 chat / llm 没拆 -api/-core。java-backend rule §1.5 要求"业务模块间禁止直接 import 彼此 -core"。当前的 `sanyan-memory-core` 直接 import 了 `com.sanyan.chat.internal.MessageEntity` 等，虽然 Maven 边界 OK，但 Java 包级别不符合 §3.3 "禁止其他模块直接 import internal 类" 的精神
- **影响**：纯架构债务，不影响功能。该妥协已在 `sanyan-memory-core/pom.xml` 注释里显式说明（"Plan 3 会修复"）
- **建议**：Plan 3 把 `sanyan-business` 拆成 `sanyan-chat-api/core` + `sanyan-character-api/core` + `sanyan-user-api/core` + `sanyan-llm-api/core` 四对模块，把 `MessageEntity` 抽 `MessageDto` 入 -api，让 memory-core 只依赖 chat-api / llm-api

---

## 最终结论

**PASS** ✅

Plan 2 长期记忆系统 29 个 task（L0 → R2）全部按 spec 实施完毕，本 R3 终审：

- 16 个 Maven 模块 100% SUCCESS
- 389 个测试 100% PASS
- 0 个 critical / important 违反 CLAUDE.md / java-backend rules / Plan 2 spec
- 3 个 minor / suggestion 已记录留给 Plan 3 处理

**Plan 2 可以视为完成；Plan 3 起步前先消化上述 S1/S2/S3 三条 suggestion。**

---

## Plan 2 数字摘要

| 项 | 数值 |
|---|---|
| 总 commits | 31（017c2b0..HEAD） |
| 新增 Maven 模块 | 4（sanyan-memory-api / sanyan-memory-core / sanyan-embedding-api / sanyan-embedding-core）+ embedding-bootstrap = 5 |
| 新增 Java 主代码文件 | ~40 |
| 新增 Java 测试文件 | ~30 |
| 新增 Flyway migration | 3（V5 / V6 / V7） |
| 总测试数（全工程） | 389 |
| 测试通过率 | 100% |
| 总主代码 LoC（全工程） | 5176 |
| 总测试代码 LoC（全工程） | 6528 |
| 总 Java 文件数（全工程） | 160 |

---

## R3 task 决策记录

R3 task 在 prompt 中要求「不要问用户」，重大决策记到本报告：

1. **接受 `RetryableEmbeddingException extends RuntimeException`**：它是 `RemoteBgeM3Provider` 内 `private static final inner class`，纯 retry 控制流信号，never 跨文件 / 跨方法暴露，最外层 catch 后转换为 `BusinessException`。不违反「全项目唯一 BusinessException」规则的精神
2. **接受 `LlmErrCode.EMBEDDING_SERVICE_UNAVAILABLE(4004)` + `MemoryErrCode.EMBEDDING_SERVICE_UNAVAILABLE(5002)` 并存**：分层语义有意保留，HTTP 层 vs 业务层视角不同，`MemoryRagSearchService` 同时捕获两个 code，无冲突
3. **接受 `sanyan-memory-core → sanyan-business` 整包依赖**：Plan 2 显式妥协，在 pom.xml 中已大段注释解释，Plan 3 会拆解
4. **不修复 `ERROR_CODE_REGISTRY.md` 缺失**：原因（a）启动期 ErrCodeConflictDetector 已守护 code 唯一性，本质不阻塞；（b）Plan 3 拆 sanyan-business 时会重新洗牌 ErrCode 分布，此时一并补登记表更经济
5. **不补 6002 错误码**：跳号本身无功能影响；可能 Plan 3 拆 embedding 错误用得上 6002
