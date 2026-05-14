# 三言 · Plan 2 实施计划：长期记忆系统（B 摘要 + C 档案 + RAG）

> **For agentic workers:** REQUIRED SUB-SKILL: `superpowers:subagent-driven-development`. Steps use `- [ ]` checkboxes.
>
> **TDD 铁律：每个 task 严格 Red → Green → Refactor 5 步。没有失败测试不允许写生产代码。**
>
> **前置依赖：Plan 1 必须已合并并跑通。本 plan 在 Plan 1 产出的代码基础上增量。**
>
> **配套规则：`~/.claude/rules/java-backend*.md`（路径触发自动加载，子代理执行时也会触发）。**

**Goal：** 让小婉"记得"用户。聊到第 3 天用户提"上次说的那只猫"她能接住；提"我家那个项目"她能想起背景。dogfood 第二关：聊到第 3 天会不会失忆？

**Architecture：**
- **B 滚动摘要**：每 N 条消息（默认 30）后台异步生成一段"对话纪要"，存 `memory_summaries` 表
- **C 结构化档案**：每次用户发消息后异步抽取结构化字段（基本信息 / 偏好 / 关键事件 / 情感线），merge 到 `memory_profiles` 表的 `profile_jsonb`
- **RAG 向量检索**：每次对话轮组完成后，把该轮组（5-10 条消息）embedding 存入 `chat_embeddings`（pgvector）；用户发消息时按语义相似度检索 Top 5 历史片段塞进 prompt
- **后台任务**：用 Redis Streams（不是消息队列重量级方案）作为简单异步任务调度
- **B/C 提取走 DeepSeek V4-Flash**（便宜）；RAG embedding 走 BGE-M3（开源、可本地部署、不依赖外部 API key）
- **部署拓扑：双服务器架构**
  - **主服务器 `new`**（`49.233.213.109`，腾讯云 CVM，4 核 / 3.6 GiB）：跑 `sanyan-server` 主应用 + PostgreSQL + Redis；调用 embedding 服务时通过 HTTP 走公网
  - **Embedding 服务器 `old`**（`154.8.162.83`，腾讯云 Lighthouse，4 核 / 3.6 GiB）：跑独立 Spring Boot 工程 `sanyan-embedding-server`，加载 DJL (Deep Java Library) + BGE-M3 ONNX 模型常驻内存（~1 GiB），暴露 `POST /embed` 给主服务器调用
  - **为什么拆开**：主服务器内存吃紧（PG + JVM 已占 1 GiB，再加 BGE-M3 的 1 GiB 余量只剩 ~1.5 GiB，OOM 风险高）；闲置 old 物尽其用；故障隔离（embedding server 挂不影响主聊天，仅 RAG 失效降级）
  - **网络链路**：北京同地域跨账号公网，实测 RTT ~12-15ms（用 nc + curl 验证过），对 embedding 调用可忽略
  - **多重 IP 白名单**：Lighthouse 防火墙 + OS ufw 双层只放行 `49.233.213.109/32 → TCP:8080`；应用层再加 `X-Internal-Token` header 校验

**仓库与模块组织（server 子模块内多 Maven 模块）：**
- **顶层 3yan repo 是 meta repo**，下面挂三个 git submodule：`app`（`3yan-app`）、`web`（`3yan-web`）、`server`（`3yan-server`）。每个 submodule 是独立 git 仓库
- **embedding 相关代码全部在 `3yan-server` 这个 submodule 内部**——不动 app / web。理由：协议契约（`sanyan-embedding-api` DTO）放在同一仓库才能保证编译期一致、改协议一次 PR 同时改两边、基础层（`foundation_packages/common-*`）可直接复用、避免发内部 maven 包的运维负担
- **两个独立 Maven 启动工程（在 `3yan-server` 内部并列存在）**：
  - `server/bootstrap/` → 打包 `sanyan-server.jar`，部署到 `new`
  - `server/embedding-bootstrap/` ← **本 Plan 新增**，打包 `sanyan-embedding.jar`，部署到 `old`
- **业务包共享 server repo，部署单元分离**——`mvn package` 产出两个独立 jar，各自 scp 到各自服务器
- **Maven Enforcer 守护**：主 `bootstrap` 只能依赖 `sanyan-embedding-api`（契约），**禁止**依赖 `sanyan-embedding-core`（实现）；反向同理，`embedding-bootstrap` 禁止引主业务包。违规 `mvn validate` 失败
- **提交流程**：在 `server/` 子模块里开发 + 提交到 `3yan-server` 仓库 → 回顶层 3yan repo `git add server` 更新子模块指针 → 顶层 commit `chore: 更新 server 子模块引用（xxx）` → 顶层 push。跟现有的子模块提交流程一致

---

## 关键常量与对齐约束（强制）

为避免"夹缝消息"（既不在短期窗口内、又未被摘要覆盖、RAG 又因语义不相似没捞回），三个关键常量必须按以下关系对齐：

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `SHORT_TERM_WINDOW_SIZE` | **32** | 调 AI 时拼到 prompt 里的短期上下文条数 |
| `SUMMARY_TRIGGER_THRESHOLD` | **30** | 每累积 N 条新消息触发一次后台摘要 |
| `RAG_CHUNK_SIZE` | **5-10** | 每段 RAG chunk 包含的消息数 |

**对齐铁律：`SHORT_TERM_WINDOW_SIZE > SUMMARY_TRIGGER_THRESHOLD`**（严格大于，不是等于）

**为什么严格大于：**
- 最坏情况"距上次摘要的增量"= `SUMMARY_TRIGGER_THRESHOLD - 1`（即 29 条）。短期窗口 ≥ 29 才能在摘要刚好还没触发的那一瞬间兜住所有"未摘要"消息
- 摘要是后台异步任务，从"触发"到"落库"有几秒延迟。这段时间里按理已该被摘要、但还没写入的消息如果掉出短期窗口，就出现 AI 看不到的夹缝
- 所以短期窗口要**严格 >** 摘要阈值，给异步落库留 buffer。本 spec 取 `32 / 30`，buffer = 2 条

**RAG 不能当夹缝兜底：** RAG 是按语义相似度 Top 5（cos sim > 0.6）检索，跟当前 query 不相似的夹缝消息根本不会被捞回。**对齐约束不允许靠 RAG 救场**——夹缝消息必须由短期窗口直接兜住。

**常量统一位置：** 集中放在 `sanyan-memory-core/internal/MemoryConstants.java`，`AiService` / `SummaryScheduler` / 任何引用方都从这里取，**禁止任何地方硬编码字面量**（遵守全局 CLAUDE.md「代码复用原则」中的值复用规则）。

**Plan 1 已有代码的兼容：** Plan 1 `AiService.chat` 当前硬编码 `PageRequest.of(0, 20)`。Plan 2 落地时 **Task Q3 必须把这个 `20` 改为 `MemoryConstants.SHORT_TERM_WINDOW_SIZE`（= 32）**，并补一条短期上下文 size 的回归测试，确保后续不会有人改回字面量。

### Embedding 服务远程调用配置（强制集中位置）

主服务调用远程 embedding server 的所有参数也走常量集中管理，放在 `sanyan-memory-core/internal/MemoryConstants.java` 同一个类里：

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `EMBEDDING_SERVER_BASE_URL` | 走 `application.yml` 配置注入，**禁止硬编码** | 主配置项 `sanyan.embedding.base-url`，本地开发为 `http://localhost:8080`，生产为 `http://154.8.162.83:8080` |
| `EMBEDDING_HTTP_CONNECT_TIMEOUT_MS` | **3000** | TCP 连接超时 |
| `EMBEDDING_HTTP_READ_TIMEOUT_MS` | **10000** | 读响应超时（首次调用涉及模型加载可能更慢，但稳态下 < 100ms） |
| `EMBEDDING_HTTP_MAX_RETRIES` | **2** | 网络异常重试次数（4xx 不重试，5xx + Timeout 重试） |
| `EMBEDDING_HTTP_RETRY_BACKOFF_MS` | **500** | 每次重试间隔（指数退避基数） |
| `EMBEDDING_INTERNAL_TOKEN` | 走 `application.yml` + env，**禁止写代码里** | 配置项 `sanyan.embedding.internal-token`，主服务和 embedding server 共享同一个 secret |

**降级策略（强制）：**
- 调用失败（重试后仍失败）时，**RAG 检索必须返回空列表**，由 `MemoryContextBuilder` 跳过 RAG 段，但 chat 主流程**不能挂**——小婉仍能用 short window + summary + profile 正常回答
- 异步索引（Task P5）失败时，丢回 Redis Streams 重试队列；最多重试 5 次后落到 dead-letter 日志，不影响主消息持久化
- 监控：每次远程调用失败必须打 ERROR 日志（含 url / 状态码 / 耗时），便于线上发现 embedding server 异常

---

**Tech Stack：**
- 在 Plan 1 基础上新增：
  - **主服务侧**（部署到 `new`）：DeepSeekAdapter（接 DeepSeek V4-Flash API）、`RemoteBgeM3Provider`（HTTP 客户端调用 embedding server）、pgvector JPA 集成（`com.pgvector:pgvector` Java client + 自定义 Hibernate `UserType`）
  - **Embedding 服务侧**（部署到 `old`）：独立 Spring Boot 工程 `sanyan-embedding-server`，依赖 **DJL (Deep Java Library, ai.djl:api + ai.djl.huggingface:tokenizers + ai.djl.pytorch:pytorch-engine)** （M2b 实施时发现 plan 原计划的 `io.qdrant:fastembed-java` 在 Maven Central **不存在**——fastembed 生态没有 Java 移植，DJL 是 Java ML 事实标准。契约不变），本地 ONNX 推理 BGE-M3。模型 URL `djl://ai.djl.huggingface.pytorch/BAAI/bge-m3`
    - **jar 大小 vs 模型大小要区分**：fat jar 含 DJL (Deep Java Library) + ONNX Runtime native lib，约 **80 MB**；**BGE-M3 模型权重不打进 jar**，首次启动时由 fastembed 自动从 Hugging Face CDN 下载到本地缓存目录（约 **1.2 GB** 磁盘 + 加载后 **1 GiB 内存常驻**）
    - **缓存目录**：`/var/cache/fastembed/`（在 `embedding-bootstrap` 的 `application-production.yml` 里配置 `fastembed.cache-dir`），让多次重启 / 部署都能复用同一份模型，避免重复下载
    - **预热**：首次启动下载 + 加载约 30~120 秒，期间 `/actuator/health` 报 `OUT_OF_SERVICE`；ready 后才接流量。后续重启秒级（从本地缓存加载）
  - **不引入新组件**：用现有 PostgreSQL + Redis Streams

---

## 文件结构（Plan 1 之后的增量）

### 新增模块

```
business_packages/
├── sanyan-memory-api/                            # 新模块
│   └── src/main/java/com/sanyan/memory/
│       ├── MemoryApi.java                       # 对外契约：getRelevantContext(userId, characterId, query)
│       └── dto/{MemoryContext,MemorySummary,MemoryProfile,MemoryFragment}.java
└── sanyan-memory-core/                           # 新模块
    └── src/main/java/com/sanyan/memory/
        ├── internal/
        │   ├── summary/
        │   │   ├── MemorySummaryEntity.java
        │   │   ├── MemorySummaryRepository.java
        │   │   ├── SummaryScheduler.java        # 每 30 条消息后触发
        │   │   └── SummaryService.java          # 调 LLM 生成摘要
        │   ├── profile/
        │   │   ├── MemoryProfileEntity.java
        │   │   ├── MemoryProfileRepository.java
        │   │   ├── ProfileExtractService.java   # 从用户消息抽取结构化字段
        │   │   └── ProfileMergeService.java     # 增量 merge JSONB
        │   ├── rag/
        │   │   ├── ChatEmbeddingEntity.java
        │   │   ├── ChatEmbeddingRepository.java
        │   │   ├── PgVectorType.java            # Hibernate UserType for vector
        │   │   ├── EmbeddingService.java        # 调用 BGE-M3
        │   │   ├── ChunkBuilder.java            # 5-10 条消息打包
        │   │   └── RagSearchService.java        # 检索 Top 5
        │   ├── orchestrator/
        │   │   └── MemoryContextBuilder.java    # 组合 B+C+RAG 输出最终 context
        │   └── MemoryErrCode.java               # 5000-5999
        ├── api/MemoryApiImpl.java
        └── event/
            └── MessagePersistedListener.java     # 监听消息落库事件触发后台流程

business_packages/sanyan-llm-core/  （扩展）
└── src/main/java/com/sanyan/llm/internal/
    ├── DeepSeekAdapter.java                     # 新增 V4-Flash 适配器
    └── RemoteBgeM3Provider.java                 # 新增：HTTP 客户端，调远程 embedding server

business_packages/sanyan-business/  （扩展，注意：Plan 1 把 chat 放在 sanyan-business 里，没拆 chat-api/chat-core）
└── src/main/java/com/sanyan/chat/internal/
    └── PromptBuilder.java  ← 修改：注入 MemoryContext 替代纯短期上下文
└── src/main/java/com/sanyan/llm/internal/
    └── AiService.java  ← 修改：把硬编码 PageRequest.of(0, 20) 改为 MemoryConstants.SHORT_TERM_WINDOW_SIZE

# ── 远程 embedding 服务（部署在 old 服务器，独立 Spring Boot 工程）──
business_packages/
├── sanyan-embedding-api/                        # 新模块：协议 DTO（主服务和 embedding-server 双方依赖）
│   └── src/main/java/com/sanyan/embedding/
│       ├── EmbeddingApi.java                   # 接口契约：embed(List<String> texts) → List<float[]>
│       └── dto/{EmbedRequest,EmbedResponse}.java   # HTTP 请求/响应 DTO
└── sanyan-embedding-core/                       # 新模块：服务端实现
    └── src/main/java/com/sanyan/embedding/
        ├── web/
        │   └── EmbeddingController.java         # POST /embed（带 X-Internal-Token 校验）
        ├── internal/
        │   ├── BgeM3DjlAdapter.java       # DJL (Deep Java Library) 包装（启动时加载 BGE-M3 ONNX）
        │   ├── EmbeddingErrCode.java            # 6000-6999
        │   └── InternalTokenInterceptor.java    # Spring 拦截器，统一校验 X-Internal-Token
        └── api/EmbeddingApiImpl.java

embedding-bootstrap/                              # 新顶级模块：独立启动工程（部署到 old）
├── pom.xml                                       # 依赖 sanyan-embedding-core + common-error + common-web
├── deploy.sh                                     # scp jar + systemd 重启
├── README.md                                     # 部署 / 运维 / 健康检查说明
└── src/main/
    ├── java/com/sanyan/embedding/EmbeddingApplication.java
    └── resources/
        ├── application.yml                       # server.port=8080 / fastembed 配置
        └── application-production.yml            # 生产环境覆盖
```

### 数据库迁移

```
bootstrap/src/main/resources/db/migration/
├── V5__init_memory_summaries.sql
├── V6__init_memory_profiles.sql
└── V7__init_chat_embeddings.sql                # CREATE EXTENSION IF NOT EXISTS vector + 表 + ivfflat 索引
```

---

## Subagent Prompt 必注入项（每次派遣子代理时强制注入）

主对话不能默认 subagent 看到了 CLAUDE.md 或 plan 的某些上下文——subagent 启动是无状态的。所以每个实现 subagent 的 prompt **必须显式包含以下全部内容**，缺一不可：

### A. TDD 铁律 完整流程（**不能简写，不能省略**）

```
你必须严格执行 TDD Red → Green → Refactor 5 步循环：

1. RED：先写一条失败的测试。测试覆盖本 task 描述的某个具体场景。
2. 运行测试，亲眼看到失败输出（不要"脑补它会失败"）。把失败截图/错误堆栈贴在 commit message 或 PR 描述里。
3. GREEN：写**刚好让这条测试通过**的最小实现代码。不多写一行。
4. 运行测试，亲眼看到通过。
5. REFACTOR：测试通过后才能优化代码结构（命名、提取方法、消除重复等）。每次 refactor 后重新跑测试。

绝对禁止的行为：
- ❌ 先写实现代码再补测试（这不是 TDD）
- ❌ 同时写测试和实现（必须分开，中间必须有一次失败测试）
- ❌ 跳过 RED 步骤（"代码太简单了不需要先写测试"——没这个例外）
- ❌ 改 bug 时直接修复（必须先写一个能暴露 bug 的失败测试）

唯一例外（不允许自行扩展）：纯 SQL Flyway migration 文件（DB schema）允许"先写 migration → 启动验证 → 写 RepositoryIT 测试增删改"的流程，但 RepositoryIT 测试本身仍必须先 fail（即测试方法先 ASSERT 错误的预期 → 跑 → 失败 → 改对 → 跑 → 通过）。

如果发现自己已经写了生产代码但没有先写失败测试，必须立即停下来，删掉刚写的生产代码，回到 RED 步骤重新开始。不允许"下次注意"。
```

### B. 项目级上下文（必注入）

```
- 项目根：/Users/aventador/code/3yan/server/（server 是 git submodule，独立 repo 3yan-server）
- 主项目 Java 包前缀：com.sanyan（不是 com.3yan，3yan 数字开头不是合法 Java 包名）
- 配置 key 前缀：sanyan.xxx（如 sanyan.embedding.base-url、sanyan.llm.deepseek.api-key）
- AI 角色名：小婉（不是林小满，旧名已废弃）
- 当前 Java：21+（new 上是 21，old 上是 25，编译目标按 server/pom.xml 决定）
- 现有业务包结构：sanyan-business（包含 chat / user / llm / character 等子目录，未拆 -api/-core）
- Plan 2 新增双模块结构：sanyan-memory-api/-core + sanyan-embedding-api/-core
- 用户 CLAUDE.md 全局规则路径：/Users/aventador/.claude/CLAUDE.md
- Java 后端 rule 路径：/Users/aventador/.claude/rules/java-backend.md 和 java-backend-business-layer.md（自动按 path 触发加载）
- 本 Plan spec 路径：/Users/aventador/code/3yan/docs/superpowers/plans/2026-05-06-3yan-plan-2-long-term-memory.md
- 本次 task 的具体步骤、文件路径、TDD 测试用例：见 Plan 中本 task 的全部内容
```

### C. 验证步骤（必注入）

```
完成实现后必须：
1. 跑测试粒度行指定的 `mvn -pl <module> test`（不要无脑跑全量）
2. 跑 `mvn -pl <module> spotless:apply` 或类似格式化（如项目有配）
3. 提交 commit 用 plan 里指定的 commit message 模板
4. 在汇报里贴出：
   - 跑测试的实际命令 + 真实输出（看到几个绿色 PASSED）
   - 创建/修改的文件列表
   - 任何偏离 plan 的决策 + 理由
```

### D. 子代理类型固定（按 superpowers:subagent-driven-development 规则）

- 实现 subagent：`general-purpose`
- spec 审查 subagent（第一轮独立审查）：`general-purpose`
- 代码质量审查 subagent（第二轮独立审查）：`superpowers:code-reviewer`
- **绝对不能**用 `feature-dev:code-reviewer`（非 superpowers 体系，跟当前流程不兼容）

### E. 每个 Task 完成的判定（必注入）

```
本 task 视为完成的条件：
1. 所有指定文件已创建/修改
2. 所有 TDD 测试都先 RED 后 GREEN，commit message 体现这一步
3. 测试粒度行指定的命令 mvn test 全部通过
4. spec 审查 subagent 报告"PASS"或"仅有锦上添花级别 issue"
5. 代码质量审查 subagent 报告"PASS"或"仅有锦上添花级别 issue"

任一未达成 → 本 task 重新派遣实现 subagent 修复 → 重新审查 → 直到 PASS
```

---

## Task 列表

### Phase L · 基础设施（父 pom + Enforcer + 数据库 schema）

#### Task L0：父 pom 模块注册 + Maven Enforcer 规则 + pgvector 扩展安装
- **Files：**
  - `server/pom.xml`：`<modules>` 段加入 `business_packages/sanyan-memory-api`、`sanyan-memory-core`、`sanyan-embedding-api`、`sanyan-embedding-core`，以及顶级 `embedding-bootstrap`
  - `server/pom.xml` 的 `maven-enforcer-plugin` 配置加 `banned-dependencies` 规则：
    - 主 `bootstrap` 模块禁止依赖 `com.sanyan:sanyan-embedding-core`（只能依赖 `sanyan-embedding-api`）
    - `embedding-bootstrap` 模块禁止依赖 `com.sanyan:sanyan-business`、`com.sanyan:sanyan-memory-core`、`com.sanyan:sanyan-memory-api`
    - 所有业务 `-core` 模块禁止互相依赖（如果 Plan 1 已经有此规则可复用）
  - 各新模块的 `pom.xml`（空骨架，依赖父 pom，packaging=jar）
- **运维步骤（在 new 服务器上执行，需要 ssh）：**
  - 验证 PostgreSQL 版本：`ssh new 'psql --version'`（确认是 PG 15+）
  - 安装 pgvector 扩展系统包：`ssh new 'sudo apt update && sudo apt install -y postgresql-17-pgvector'`（包名按 PG 版本调整）
  - 不在这里跑 `CREATE EXTENSION`，留给 L3 的 Flyway migration 做
- **测试：** `mvn validate` 通过；`mvn clean package` 能产出两个 fat jar；新模块空壳能编译
- **测试粒度：** 全量（改父 pom 影响所有模块）
- **Commit：** `chore(infra): Plan 2 父 pom 注册新模块 + Enforcer 规则 + 服务器安装 pgvector 扩展`

#### Task L1：V5 memory_summaries
- **Files：** `business_packages/sanyan-business/src/main/resources/db/migration/V5__init_memory_summaries.sql`
- **Schema：** id BIGSERIAL / user_id BIGINT NOT NULL / character_id BIGINT NOT NULL / period_start_message_id BIGINT / period_end_message_id BIGINT / summary_text TEXT NOT NULL / message_count INT NOT NULL / created_at TIMESTAMP NOT NULL DEFAULT NOW()；索引 `CREATE INDEX idx_memory_summaries_user_char_created ON memory_summaries(user_id, character_id, created_at DESC)`
- **TDD（迁移文件的 RED-GREEN）：** 先写 `V5MigrationIT`（`@SpringBootTest` + Testcontainers PG）断言 schema 应有的字段和索引 → 跑测试失败（migration 未写） → 写 V5 SQL → 跑测试通过
- **测试粒度：** 跑 `bootstrap` 包测试
- **Commit：** `feat(db): V5 memory_summaries 滚动摘要表`

#### Task L2：V6 memory_profiles
- **Files：** `business_packages/sanyan-business/src/main/resources/db/migration/V6__init_memory_profiles.sql`
- **Schema：** user_id BIGINT / character_id BIGINT / 复合 PK (user_id, character_id) / profile_jsonb JSONB NOT NULL DEFAULT '{}' / version INT NOT NULL DEFAULT 1（乐观锁）/ updated_at TIMESTAMP NOT NULL DEFAULT NOW()；初始 jsonb 默认 schema 由应用层在插入时确保：
  ```json
  {
    "basic_info": { "name": null, "age": null, "occupation": null, "hometown": null },
    "preferences": { "foods": [], "movies": [], "colors": [], "music": [] },
    "events": [],
    "emotion_line": []
  }
  ```
- **TDD：** `V6MigrationIT` 断言 schema + JSONB 字段可插入 + 复合 PK 约束生效
- **测试粒度：** 跑 `bootstrap` 包测试
- **Commit：** `feat(db): V6 memory_profiles 结构化档案表`

#### Task L3：V7 chat_embeddings + pgvector
- **Files：** `business_packages/sanyan-business/src/main/resources/db/migration/V7__init_chat_embeddings.sql`
- **SQL：** `CREATE EXTENSION IF NOT EXISTS vector;` + `chat_embeddings` 表（id BIGSERIAL / user_id BIGINT NOT NULL / character_id BIGINT NOT NULL / message_ids BIGINT[] NOT NULL / chunk_text TEXT NOT NULL / embedding vector(1024) NOT NULL / token_count INT / created_at TIMESTAMP NOT NULL DEFAULT NOW()）+ `CREATE INDEX ON chat_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);`
- **TDD：** `V7MigrationIT` 用 Testcontainers `pgvector/pgvector:pg17` 镜像，断言：
  - 扩展加载成功：`SELECT * FROM pg_extension WHERE extname='vector'`
  - 表存在 + embedding 列类型 vector(1024)
  - ivfflat 索引存在
- **依赖前置：** L0 已在 new 上 `apt install postgresql-17-pgvector`，本 task 的 IT 用 Testcontainers 不依赖宿主 PG
- **测试粒度：** 跑 `bootstrap` 包测试（含 Testcontainers）
- **Commit：** `feat(db): V7 chat_embeddings + pgvector ivfflat 索引`

### Phase M · LLM 扩展（DeepSeek + Embedding）

#### Task M1：DeepSeekAdapter
- **位置说明：** Plan 1 把 LLM 放在 `sanyan-business/src/main/java/com/sanyan/llm/internal/`。本 task 把新 adapter 加到这里（不另起 sanyan-llm-core 模块，保持 Plan 1 现状）
- **Files：** `sanyan-business/src/main/java/com/sanyan/llm/internal/DeepSeekAdapter.java` + 测试
- **关键：** OpenAI 兼容协议（DeepSeek API 走 `https://api.deepseek.com/chat/completions` 兼容格式）；如 Plan 1 已有 `LLMTaskType` enum + `LLMProvider` 接口 + `supports()` 方法则复用，没有则本 task 先在 `sanyan-business/llm/internal/` 加；模型名 `deepseek-v4-flash`；`application-local.yml` 加 `sanyan.llm.deepseek.api-key=${DEEPSEEK_API_KEY:}`
- **依赖：** 用 Spring `RestClient`（Spring Boot 3.4+ 内置）；不引入 OkHttp / Apache HttpClient
- **测试：** MockWebServer (`com.squareup.okhttp3:mockwebserver`) 模拟 OpenAI Chat Completion 协议，覆盖成功 / 401 / 429 / 500
- **TDD 5 步严格执行**
- **测试粒度：** 跑 `sanyan-business` 包测试 + `mvn -pl bootstrap test` 验证 Spring 启动
- **Commit：** `feat(llm): DeepSeekAdapter（V4-Flash，BACKGROUND task）`

#### Task M2a：sanyan-embedding-api（协议 DTO + 接口契约）
- **Files：** 新建 `business_packages/sanyan-embedding-api/`，按全局 java-backend rule 的 `-api` 模块约束写
  - `EmbeddingApi.java`（接口）：`List<float[]> embed(List<String> texts)`
  - `dto/EmbedRequest.java`（record）：`List<String> texts`，**JSR-303 校验**：`@NotEmpty`、每条 text `@Size(max = 2000)`
  - `dto/EmbedResponse.java`（record）：`List<float[]> vectors`, `int dim`, `long latencyMs`
- **关键：** 只放接口 + DTO，不含任何实现。主服务和 embedding-server 都依赖这个模块，保证协议契约一致
- **依赖：** 只依赖 `jakarta.validation:jakarta.validation-api`（DTO 用 `@NotEmpty` 等注解，但不强制 Hibernate Validator）
- **测试：** 不需要（纯契约模块）
- **测试粒度：** 不跑
- **Commit：** `feat(embedding-api): 远程 embedding 服务协议契约`

#### Task M2b：sanyan-embedding-core + embedding-bootstrap（独立部署服务）
- **Files：**
  - `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/web/EmbeddingController.java`：`POST /embed` 入口
  - `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/BgeM3DjlAdapter.java`：包装 `DJL (ai.djl:api + ai.djl.huggingface:tokenizers + ai.djl.pytorch:pytorch-engine)`，启动时 `@PostConstruct` 加载 BGE-M3 ONNX 模型到内存（1024 维输出）
  - `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/InternalTokenInterceptor.java`：Spring `HandlerInterceptor`，校验 `X-Internal-Token` header，不通过返回 401
  - `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/EmbeddingErrCode.java`：错误码 enum（区间 6000-6999）
  - `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/api/EmbeddingApiImpl.java`：薄壳，委托 adapter
  - `embedding-bootstrap/pom.xml`：依赖 `sanyan-embedding-core` + `sanyan-common-error` + `sanyan-common-web`，配置 `spring-boot-maven-plugin` 的 `repackage` goal
  - `embedding-bootstrap/src/main/java/com/sanyan/embedding/EmbeddingApplication.java`：`@SpringBootApplication`，`main` 方法
  - `embedding-bootstrap/src/main/resources/application.yml`：`server.port=8080` / `sanyan.embedding.internal-token=${EMBEDDING_INTERNAL_TOKEN}` / `sanyan.embedding.model.cache-dir=/var/cache/fastembed` / `management.endpoints.web.exposure.include=health,info`
  - `embedding-bootstrap/src/main/resources/application-production.yml`：生产覆盖（如更长的 JVM warmup 时间）
- **依赖（pom.xml）：**
  - `DJL (ai.djl:api + ai.djl.huggingface:tokenizers + ai.djl.pytorch:pytorch-engine):<latest>`（核对：https://central.sonatype.com/search?q=DJL (Deep Java Library)）
  - `org.springframework.boot:spring-boot-starter-web`
  - `org.springframework.boot:spring-boot-starter-actuator`
  - 项目内 `sanyan-embedding-api` + `sanyan-common-error` + `sanyan-common-web`
- **关键：**
  - 启动时模型加载约 30~120 秒 + 占 1 GiB 内存，**必须等 `ApplicationReadyEvent` 触发 + adapter 加载完模型后再标 ready**
  - `/actuator/health` 实现自定义 `HealthIndicator`，真的调一次 `embed("ping")` 验证模型 ready；未 ready 时返回 `OUT_OF_SERVICE`
  - 日志格式 `request_id={uuid} latency_ms={n} text_count={n}`，方便排查
  - JVM 参数：`-Xmx1500m -XX:+UseG1GC`（在 systemd unit 里设）
- **测试：**
  - `BgeM3DjlAdapterIT`（`@SpringBootTest`）：真实加载模型，验证 `embed("你好")` 返回维度 1024 + 浮点数 finite
  - `EmbeddingControllerIT`（`@WebMvcTest`）：mock adapter，覆盖 token 校验通过 / 缺失 / 错误 三种场景；`@NotEmpty` 校验空 list 返回 400
- **TDD 5 步严格执行**
- **测试粒度：** 跑 `sanyan-embedding-core` + `embedding-bootstrap` 两个包测试。`BgeM3DjlAdapterIT` 因为要真实加载模型耗时长，**只在 `@Tag("slow")` 标记下跑**，本地默认跳过，CI 跑
- **Commit：** `feat(embedding-server): 独立 Spring Boot 服务 + BGE-M3 DJL (Deep Java Library) + token 校验`

#### Task M2c：主服务侧 RemoteBgeM3Provider（HTTP 客户端）
- **位置：** 跟 M1 一致，放在 `sanyan-business/src/main/java/com/sanyan/llm/internal/RemoteBgeM3Provider.java`（保持 Plan 1 现状不另拆模块）
- **接口：** `EmbeddingProvider` 接口本 task 在 `sanyan-business/llm/internal/EmbeddingProvider.java` 定义（`List<float[]> embed(List<String> texts)`），`RemoteBgeM3Provider` implements 它。这样 Plan 3 如果想换 embedding 提供方，只要换 Bean 不动调用方
- **依赖（bootstrap/pom.xml）：**
  - `org.springframework.boot:spring-boot-starter-web`（已有）
  - `org.springframework.retry:spring-retry`（新加）+ `org.springframework:spring-aspects`（Retry 切面需要）
  - 项目内 `sanyan-embedding-api`（用 EmbedRequest / EmbedResponse DTO）
- **启用 Retry：** 在 `SanyanApplication.java` 加 `@EnableRetry`
- **关键：**
  - 用 Spring `RestClient` 调用远程 server，URL 从 `sanyan.embedding.base-url` 配置注入
  - 自动加 `X-Internal-Token` header（从 `sanyan.embedding.internal-token` 配置注入）
  - 设置 `RestClient.Builder` 的 timeout：connect `EMBEDDING_HTTP_CONNECT_TIMEOUT_MS`、read `EMBEDDING_HTTP_READ_TIMEOUT_MS`
  - 用 `@Retryable(value = {ResourceAccessException.class, HttpServerErrorException.class}, maxAttempts = EMBEDDING_HTTP_MAX_RETRIES, backoff = @Backoff(delay = EMBEDDING_HTTP_RETRY_BACKOFF_MS, multiplier = 2))` 重试 5xx + IOException；**4xx 直接抛，不重试**
  - 重试耗尽后抛 `BusinessException(MemoryErrCode.EMBEDDING_SERVICE_UNAVAILABLE)`，由调用方决定降级
  - **不在这里加 fallback 返回空向量**——降级在调用方决策（P4 RagSearchService）
- **测试：**
  - `RemoteBgeM3ProviderTest`：用 MockWebServer 模拟 server，覆盖：
    - 200 成功 + 验证 `X-Internal-Token` header 被正确发出
    - 401 token 错（4xx 不重试，直接抛 `BusinessException`）
    - 503 重试后成功（计数验证重试次数 = 1）
    - 503 重试耗尽（重试 `EMBEDDING_HTTP_MAX_RETRIES` 次后抛 `BusinessException`）
    - 连接 timeout（`ResourceAccessException` 重试 + 最终抛）
- **TDD 5 步严格执行**
- **测试粒度：** 跑 `sanyan-business` 包测试
- **Commit：** `feat(llm): RemoteBgeM3Provider HTTP 客户端 + Spring Retry + token`

#### Task M2d：embedding 服务部署到 old + 端到端连通性验证
- **Token 生成与分发（本 task 第一步）：**
  - 在本地生成 32 字节随机 token：`openssl rand -hex 32` → 得到形如 `a3b8...` 的 64 字符字符串
  - 把 token 写到 **new** 的 `/etc/sanyan/embedding-token.env`：`EMBEDDING_INTERNAL_TOKEN=<生成的 token>`，文件权限 `chmod 600` 仅 root 可读
  - 把同一个 token 写到 **old** 的 `/etc/sanyan/embedding-token.env`，同样权限
  - **绝对禁止**把这个 token 提交进 git——`.gitignore` 加 `**/embedding-token*`
  - new 上的 systemd unit（主服务）`EnvironmentFile=/etc/sanyan/embedding-token.env`
  - old 上的 systemd unit（embedding 服务）`EnvironmentFile=/etc/sanyan/embedding-token.env`
- **Files：**
  - `embedding-bootstrap/deploy.sh`：本地 `mvn package -pl embedding-bootstrap -am` → `scp` jar 到 `old:/opt/sanyan-embedding/sanyan-embedding.jar` → ssh 写 / 更新 systemd unit → `systemctl daemon-reload && systemctl restart sanyan-embedding`
  - `embedding-bootstrap/systemd/sanyan-embedding.service`（模板，本地存）：
    ```
    [Unit]
    Description=Sanyan Embedding Service (BGE-M3)
    After=network.target

    [Service]
    Type=simple
    User=root
    EnvironmentFile=/etc/sanyan/embedding-token.env
    Environment=SPRING_PROFILES_ACTIVE=production
    ExecStart=/usr/bin/java -Xmx1500m -XX:+UseG1GC -jar /opt/sanyan-embedding/sanyan-embedding.jar
    Restart=on-failure
    RestartSec=10s
    StandardOutput=append:/var/log/sanyan-embedding.log
    StandardError=append:/var/log/sanyan-embedding.log

    [Install]
    WantedBy=multi-user.target
    ```
  - old 上预创建：`mkdir -p /var/cache/fastembed && mkdir -p /opt/sanyan-embedding && mkdir -p /etc/sanyan`
- **关键：**
  - 部署前**必须**用 `ssh new 'timeout 3 bash -c "</dev/tcp/154.8.162.83/8080"'` 验证 Lighthouse 防火墙 + ufw 已经放行（运维侧已完成，见 plan 顶部「部署拓扑」段）
  - 部署后从主服务发一次真实 embedding 请求，验证端到端链路：
    ```bash
    TOKEN=$(ssh new 'cat /etc/sanyan/embedding-token.env | cut -d= -f2')
    ssh new "curl -H 'X-Internal-Token: $TOKEN' -H 'Content-Type: application/json' \
         -d '{\"texts\":[\"你好世界\"]}' \
         http://154.8.162.83:8080/embed"
    ```
    预期返回 `{"vectors":[[...1024 个 float...]], "dim":1024, "latencyMs":xx}`
  - 首次启动 BGE-M3 模型下载约 30-120 秒，期间 `/actuator/health` 返回 `OUT_OF_SERVICE`。要 tail `/var/log/sanyan-embedding.log` 看到 "model loaded" 才能跑验证
- **测试：** 真实部署 + curl 验证 + Spring Boot Actuator `/actuator/health` 返回 UP + 三次连续 curl 看 latencyMs 是否稳定（应该 < 100ms）
- **测试粒度：** 这是个运维/部署 task，无单元测试；但 curl + health check 必须 PASS
- **Commit：** `chore(embedding): 部署 embedding service 到 old + token 分发 + 端到端验证通过`

#### Task M3：注册多 Adapter 到 Factory（如 Plan 1 已有 Factory）/ 新建 Factory（如没有）
- **前置检查：** grep Plan 1 是否有 `LLMProviderFactory` / `LLMProviderRouter`：
  - 有 → 直接修改注册新 adapter
  - 没有 → 本 task 先建（在 `sanyan-business/llm/internal/`），按"按 LLMTaskType 路由 Provider"模式实现
- **Files：** `sanyan-business/src/main/java/com/sanyan/llm/internal/{LLMProviderFactory,LLMProviderRouter}.java`（按现状增量）
- **关键：**
  - `LLMProviderRouter.chat(LLMTaskType.USER_FACING, ...)` 走豆包（Plan 1 现状）
  - `LLMProviderRouter.chat(LLMTaskType.BACKGROUND, ...)` 走 DeepSeek
  - `EmbeddingProvider` Bean 默认装配 `RemoteBgeM3Provider`（用 `@Primary` 或单一实现避免歧义）
- **测试：** Mockito 单测验证路由逻辑；Spring `@Test` 验证 Bean 装配
- **测试粒度：** 跑 `sanyan-business` 包测试
- **Commit：** `feat(llm): Factory 注册 DeepSeek + RemoteBgeM3Provider`

### Phase N · 摘要（B 模块）

#### Task N0：MemoryConstants（关键常量集中位置）
- **Files：** `sanyan-memory-core/src/main/java/com/sanyan/memory/internal/MemoryConstants.java`
- **内容：** `SHORT_TERM_WINDOW_SIZE = 32` / `SUMMARY_TRIGGER_THRESHOLD = 30` / `RAG_CHUNK_MIN_SIZE = 5` / `RAG_CHUNK_MAX_SIZE = 10` / `RAG_CHUNK_MAX_TOKEN = 400` / `RAG_TOP_K = 5` / `RAG_MIN_COS_SIM = 0.6` / `PROFILE_EXTRACT_THROTTLE_MINUTES = 5` / `EMBEDDING_HTTP_CONNECT_TIMEOUT_MS = 3000` / `EMBEDDING_HTTP_READ_TIMEOUT_MS = 10000` / `EMBEDDING_HTTP_MAX_RETRIES = 2` / `EMBEDDING_HTTP_RETRY_BACKOFF_MS = 500` / `PROFILE_EMOTION_LINE_MAX = 50` / `EMBEDDING_DIM = 1024`
- **关键：** 见上文「关键常量与对齐约束」。class 必须 `final` + `private` 构造，所有字段 `public static final`
- **测试：** 编译期生效，无运行时测试；但需要一个 `MemoryConstantsTest` 断言：
  1. `SHORT_TERM_WINDOW_SIZE > SUMMARY_TRIGGER_THRESHOLD`（对齐铁律）
  2. `RAG_CHUNK_MAX_SIZE >= RAG_CHUNK_MIN_SIZE`
  3. `EMBEDDING_HTTP_READ_TIMEOUT_MS > EMBEDDING_HTTP_CONNECT_TIMEOUT_MS`
- **TDD：** 先写断言测试（fail） → 写常量（pass）
- **测试粒度：** 跑 `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-core): MemoryConstants 关键常量集中位置 + 对齐铁律断言`

#### Task N1：MemorySummaryEntity + Repository + Fixtures
- **Files：**
  - `sanyan-memory-core/internal/summary/MemorySummaryEntity.java`（Entity 后缀强制）
  - `sanyan-memory-core/internal/summary/MemorySummaryRepository.java` extends `JpaRepository<MemorySummaryEntity, Long>`
  - `sanyan-memory-core/src/test/java/.../fixtures/MemorySummaryTestFixtures.java`（Object Mother 强制）
- **关键查询方法：** `Optional<MemorySummaryEntity> findFirstByUserIdAndCharacterIdOrderByCreatedAtDesc(Long, Long)`、`long countByUserIdAndCharacterIdAndIdGreaterThan(...)` 用于"自上次摘要以来的新增消息数"
- **TDD：** `MemorySummaryRepositoryIT`（`@DataJpaTest` + H2），验证查询。Fixtures 至少有 `validSummary()` 和 `summaryWithMessageCount(int n)`
- **测试粒度：** 跑 `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-core): MemorySummaryEntity + Repository + Fixtures`

#### Task N2：MemorySummaryService（生成摘要）
- **类命名：** `MemorySummaryService`（按 java-backend rule `<Domain><Action>Service` 命名规范，加 `Memory` 前缀）
- **Files：** `sanyan-memory-core/internal/summary/MemorySummaryService.java`
- **关键：** 输入 `List<MessageEntity>`（30 条），输出摘要文本；调 `LLMProviderRouter.chat(LLMTaskType.BACKGROUND, ...)` 用 DeepSeek，prompt 模板：
  ```
  下面是用户和「小婉」的一段对话（{{count}} 条），请你总结成一段简洁的"对话纪要"，
  保留关键事实、情感线索、用户提到的人事物。100-200 字。
  对话：{{messages}}
  纪要：
  ```
- **依赖反向：** 本 service 需要调 `sanyan-business/llm` 里的 LLM Router——因此 sanyan-memory-core 的 pom 加 sanyan-business 依赖（理由同 Task N3 注释，Plan 1 未拆 -api/-core 的妥协）
- **TDD：** `MemorySummaryServiceTest` Mockito 测试，传入 fixture messages（用 `MessageTestFixtures` 如 Plan 1 有）→ 验证 prompt 字符串构造正确 + 委托给 LLM router；mock LLM 返回 fake 摘要，断言返回值
- **测试粒度：** 跑 `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-core): MemorySummaryService（DeepSeek V4-Flash 生成纪要）`

#### Task N3：SummaryScheduler（消息落库后异步触发）
- **前置检查：** 先在 `sanyan-business/chat` 里 grep 找 `MessagePersistedEvent`。如果 Plan 1 没发这个事件，本 task **先在 `sanyan-business/chat/internal/MessageService` 里加事件发布**（用 `ApplicationEventPublisher`），事件类放在 `sanyan-business/chat/event/MessagePersistedEvent.java`（record）
- **关键：** 监听 `MessagePersistedEvent`；统计自上次摘要以来新增消息数 ≥ `MemoryConstants.SUMMARY_TRIGGER_THRESHOLD`（= 30）→ 触发 SummaryService。**禁止硬编码字面量 `30`**
- **依赖方向：** sanyan-memory-core 监听 sanyan-business 发的事件——需要在 `sanyan-memory-core/pom.xml` 加 `sanyan-business` 依赖。Maven Enforcer 规则需要把这条依赖加入允许列表（默认禁止 -core 跨业务依赖）
- **TDD：** Mockito 单测覆盖触发 / 不触发 / 边界
- **测试粒度：** 改了 sanyan-business 的事件发布，跑 `sanyan-business` + `sanyan-memory-core` 两个包的测试
- **Commit：** `feat(memory-core): SummaryScheduler 30 条阈值触发 + chat MessagePersistedEvent 发布`

### Phase O · 结构化档案（C 模块）

#### Task O1：MemoryProfileEntity + Repository + Fixtures
- **Files：**
  - `sanyan-memory-core/internal/profile/MemoryProfileEntity.java`
  - `sanyan-memory-core/internal/profile/MemoryProfileRepository.java`
  - `MemoryProfileTestFixtures`
- **JSONB 映射：** 检查项目 Hibernate 版本：
  - Hibernate 6.4+ → 用内置 `@JdbcTypeCode(SqlTypes.JSON)` 映射 `Map<String, Object>` 或自定义 record 类型（推荐）
  - Hibernate < 6.4 → 加 `io.hypersistence:hypersistence-utils-hibernate-63:<latest>` 依赖，用 `@Type(JsonType.class)`
- **乐观锁：** Entity 加 `@Version Integer version`
- **TDD：** `MemoryProfileRepositoryIT`（`@DataJpaTest`）验证 JSONB 读写往返；Fixtures 至少 `validProfile()` / `profileWithFoods(List<String>)`
- **测试粒度：** 跑 `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-core): MemoryProfileEntity + Repository + Fixtures + JSONB 映射`

#### Task O2：MemoryProfileExtractService（结构化抽取）
- **类命名：** `MemoryProfileExtractService`（加 Memory 前缀）
- **关键：** 输入用户的单条消息（或最近 N 条），调 DeepSeek V4-Flash，prompt 要求输出 JSON：
  ```json
  {
    "basic_info_updates": {"occupation": "前端工程师"},
    "preferences_appended": {"foods": ["寿司"]},
    "events_appended": [{"type": "interview", "content": "周三技术面试", "date": null}],
    "emotion_signal": "焦虑"
  }
  ```
- **解析容错：** LLM 返回非合法 JSON 时，**不抛异常**，打 WARN 日志后返回 `null`（让 O4 listener 跳过）。**用 Jackson 严格解析，不要尝试"修复"非法 JSON**
- **TDD：** `MemoryProfileExtractServiceTest` Mockito 测试 LLM 返回各种 fake JSON（合法 / 字段缺失 / 完全乱码），验证服务方法正确解析或优雅返回 null
- **测试粒度：** 跑 `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-core): MemoryProfileExtractService 结构化抽取 + 容错`

#### Task O3：MemoryProfileMergeService（JSONB 增量 merge）
- **类命名：** `MemoryProfileMergeService`
- **关键：** Java 端读出 → 合并 → 写回（带乐观锁 version），不用 PG `jsonb_set`（Java 端更易测）；用户名字优先保留首次抽取的值，避免 LLM 反复改写
- **合并规则细则：**
  - `basic_info.name`：仅当当前为 null 时写入；后续抽取出不同 name 时**忽略**
  - `basic_info.age/occupation/hometown`：每次抽取允许覆盖（用户可能搬家、换工作）
  - `preferences.foods/movies/colors/music`：append + dedupe（小写规范化后去重）
  - `events`：追加，每条带 `extracted_at` 时间戳
  - `emotion_line`：追加最新一条，**保留最近 `MemoryConstants.PROFILE_EMOTION_LINE_MAX`（= 50）条**，超过则丢最早
- **乐观锁冲突处理：** `OptimisticLockingFailureException` 重试一次，再失败抛 `BusinessException(MemoryErrCode.PROFILE_MERGE_CONFLICT)`，由调用方决定（O4 listener 重新入队或丢弃）
- **TDD：** 单测覆盖：
  - basic_info.name 已存在不覆盖（再传新 name 也不变）
  - basic_info.occupation 已存在被覆盖
  - preferences.foods append + dedupe
  - events 追加 + 带时间戳
  - emotion_line 超过 50 条时丢最早
  - 乐观锁冲突时重试一次后成功 / 重试后再冲突抛异常
- **测试粒度：** 跑 `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-core): MemoryProfileMergeService（增量 merge + 乐观锁）`

#### Task O4：触发链路（每条用户消息后异步抽取 + Redis 节流）
- **Files：** `sanyan-memory-core/event/UserMessageProfileExtractListener.java`
- **关键：** Listener 监听 `MessagePersistedEvent`，filter `role == USER`（如果事件类有 role 字段，没有的话先在事件类加 + 让 chat 发布方传入）；调 `MemoryProfileExtractService` → `MemoryProfileMergeService`
- **节流实现（强制 Redis 实现）：**
  - 调 Service 前先执行 `SET sanyan:memory:profile:throttle:{userId}:{characterId} 1 NX EX 300`
  - `NX` = 仅当 key 不存在时才设置，`EX 300` = TTL 5 分钟
  - 命令返回 `OK` → 拿到 token，继续执行抽取
  - 返回 `nil` → 5 分钟内已经触发过，本次跳过 + 打 INFO 日志
- **依赖：** 复用 Plan 1 已有的 `sanyan-common-cache`（应该已经包了 Spring Data Redis）
- **TDD：**
  - Mockito + `@DataRedisTest` 或 Testcontainers Redis：
    - 首次触发：SETNX 成功 → Service 被调用
    - 5 分钟内再次触发：SETNX 失败 → Service 未被调用
    - 5 分钟后再触发：key 过期 → 重新成功
- **测试粒度：** 跑 `sanyan-memory-core` 包测试 + 启动 Redis Testcontainer
- **Commit：** `feat(memory-core): 用户消息抽取触发链路 + Redis NX TTL 5 分钟节流`

### Phase P · RAG 向量检索

#### Task P1：PgVectorType（Hibernate UserType）+ pgvector-java 依赖
- **依赖：** `sanyan-memory-core/pom.xml` 加 `com.pgvector:pgvector:<latest>`（核对 Maven Central）
- **Files：**
  - `sanyan-memory-core/internal/rag/PgVectorType.java`：自定义 Hibernate `UserType` 或用 `@JdbcTypeCode` 把 `float[]` 与 PG `vector(1024)` 映射；包装 `com.pgvector.PGvector`
  - `sanyan-memory-core/internal/rag/ChatEmbeddingEntity.java`：`embedding` 字段用 `float[]` 类型，加 `@Type(PgVectorType.class)` 或等价注解
- **TDD：** `ChatEmbeddingRepositoryIT` 用 Testcontainers `pgvector/pgvector:pg17` 镜像（与生产 PG 版本对齐，本 task 之前 L0 已经在 new 上装了 pgvector 包），保存查询往返
- **测试粒度：** 跑 `sanyan-memory-core` 包测试 + 启动 PG Testcontainer
- **Commit：** `feat(memory-core): PgVectorType Hibernate 映射 + pgvector-java 依赖`

#### Task P2：MemoryChunkBuilder
- **类命名：** `MemoryChunkBuilder`
- **Files：** `sanyan-memory-core/internal/rag/MemoryChunkBuilder.java`
- **关键：** 输入 `List<MessageEntity>`（按时间序），输出 `List<Chunk>`（内部 record，含 `List<Long> messageIds`, `String chunkText`）
- **切片策略：**
  - 最少 `MemoryConstants.RAG_CHUNK_MIN_SIZE`（=5）条
  - 最多 `MemoryConstants.RAG_CHUNK_MAX_SIZE`（=10）条
  - 单 chunk 文本 token 数估算 ≤ `MemoryConstants.RAG_CHUNK_MAX_TOKEN`（=400），按 1 token ≈ 2 个中文字符粗估
  - 不跨越自然边界（如同一句话不切开）
- **TDD：** 单测覆盖：
  - 5 条短消息 → 1 个 chunk
  - 20 条短消息 → 2 个 chunk（10 + 10）
  - 单条消息 token 超 400 → 单独 chunk
- **测试粒度：** 跑 `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-core): MemoryChunkBuilder 5-10 条切片`

#### Task P3：MemoryEmbeddingService + ChatEmbeddingEntity 入库
- **类命名：** `MemoryEmbeddingService`（不跟 sanyan-business 里的 `EmbeddingProvider` 接口同名）
- **Files：**
  - `sanyan-memory-core/internal/rag/MemoryEmbeddingService.java`
  - `sanyan-memory-core/internal/rag/ChatEmbeddingRepository.java`
  - `ChatEmbeddingTestFixtures`
- **关键：** Service 接受 `List<Chunk>` → 调 `EmbeddingProvider.embed`（HTTP 到远程 embedding server，单次最多 10 条 chunk 一起送）→ 拿到向量 list → 批量保存 ChatEmbeddingEntity
- **TDD：** Mockito（mock EmbeddingProvider）+ `@DataJpaTest` Repository IT
- **测试粒度：** 跑 `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-core): MemoryEmbeddingService 批量入库`

#### Task P4：MemoryRagSearchService（带降级）
- **类命名：** `MemoryRagSearchService`
- **Files：** `sanyan-memory-core/internal/rag/MemoryRagSearchService.java`
- **关键：** 输入 `userId, characterId, queryText`，先 embed query → repo 查 cosine similarity Top `MemoryConstants.RAG_TOP_K`（=5）且 cos sim > `MemoryConstants.RAG_MIN_COS_SIM`（=0.6） → 返回 `List<MemoryFragment>`（含 chunk_text + 时间）
- **SQL：** Repository 用 `@Query(value = "SELECT * FROM chat_embeddings WHERE user_id = :userId AND character_id = :characterId ORDER BY embedding <=> :queryVec LIMIT :topK", nativeQuery = true)`，pgvector cosine distance 操作符 `<=>` 返回 0 ~ 2，cos sim = 1 - distance/2，过滤 `cos sim > 0.6` 等价 `distance < 0.8`
- **降级（强制）：** 远程 embedding server 不可用时（捕获 `BusinessException(MemoryErrCode.EMBEDDING_SERVICE_UNAVAILABLE)`），**返回空列表 + 打 ERROR 日志**，不抛出。`MemoryContextBuilder` 见到空列表跳过 RAG 段，主聊天不受影响
- **TDD：**
  - `MemoryRagSearchServiceIT`：用 Testcontainers pgvector 插入 fixture chunks（手工塞已知向量）→ 用相似 query embed → 验证返回 Top K 且过滤掉低相似度的
  - **降级测试：** mock `EmbeddingProvider` 抛 `EMBEDDING_SERVICE_UNAVAILABLE` → 断言 `MemoryRagSearchService.search` 返回 `Collections.emptyList()` + 用 `@Slf4jLogCaptor` 或类似工具验证 ERROR 日志被写
- **测试粒度：** 跑 `sanyan-memory-core` 包测试 + PG Testcontainer
- **Commit：** `feat(memory-core): MemoryRagSearchService 语义检索 Top 5 + embedding 不可用时优雅降级`

#### Task P5：异步索引 trigger（消息落库后 + Redis Streams 重试 + dead-letter）
- **Files：**
  - `sanyan-memory-core/event/MessageEmbeddingIndexListener.java`：监听 `MessagePersistedEvent` 把 chunk 推入 Redis Stream `sanyan:memory:rag:index-queue`
  - `sanyan-memory-core/internal/rag/RagIndexWorker.java`：Spring `@Scheduled`（或 Spring Data Redis 的 `StreamListener`）消费队列，调 `MemoryEmbeddingService.index`
  - `sanyan-memory-core/internal/rag/RagIndexDeadLetterLog.java`：dead-letter 写日志（最简方案，不上独立监控系统）
- **关键流程：**
  - Listener 累积消息到 chunk（用 `MemoryChunkBuilder`），当满 5-10 条时把 chunk 推 Redis Stream `XADD sanyan:memory:rag:index-queue * userId X characterId Y chunk {json}`
  - Worker `StreamMessageListenerContainer` 消费，调 `MemoryEmbeddingService` 入库
  - **重试策略：** 失败时把消息 retry-count++ 后重新 XADD 到 stream；最多 **5 次**（间隔指数退避 1s/2s/5s/15s/60s，用 `XADD` + 延迟 key 实现简易延迟，或直接重新入队靠 worker 自身节奏退避）
  - 超过 5 次：写 ERROR 日志 `[RAG_INDEX_DLQ] userId=X chunk=...`，**不阻塞主消息流，不回滚消息持久化**
- **依赖：** Spring Data Redis（Plan 1 应该已有）；不引入额外消息队列
- **TDD：**
  - `MessageEmbeddingIndexListenerTest`：mock Redis template，断言 chunk 满时被 XADD
  - `RagIndexWorkerIT`：用 Redis Testcontainer + mock `MemoryEmbeddingService`，覆盖：
    - 成功一次过：消息 XADD → worker 消费 → embedding service 被调用
    - 失败重试：mock service 抛异常 → 消息回到 stream + retry-count 增加
    - 重试耗尽：retry-count 达到 5 → ERROR 日志写出 + 消息从 stream 移除
- **测试粒度：** 跑 `sanyan-memory-core` 包测试 + Redis Testcontainer
- **Commit：** `feat(memory-core): 消息落库后异步索引到 pgvector + Redis Streams 重试 + dead-letter 日志`

### Phase Q · 整合：MemoryContextBuilder + PromptBuilder 改造

#### Task Q1：MemoryContextBuilder
- **类命名：** `MemoryContextBuilder`（不冲突，本身就是 Builder 模式）
- **Files：** `sanyan-memory-core/internal/orchestrator/MemoryContextBuilder.java`
- **关键：** 输入 `userId, characterId, queryText, recentMessages`，返回组合后的 `MemoryContext`（DTO 定义在 `sanyan-memory-api/dto/MemoryContext.java`）：
  - 最新一段 summary（如有）
  - profile_jsonb 转纯文本"她记得的关于你的事"
  - RAG 检索 Top 5 fragments（如有）
- **输出格式：** 一段 system message 文本，插入到 system prompt 之后、user/assistant 历史之前的"她对你的记忆"段。空内容时返回 null，让调用方跳过
- **TDD：** Mockito 单测覆盖：
  - 无 summary + 无 profile + 无 RAG → 返回 null
  - 仅有 summary → context 只含 summary 段
  - 三者都有 → 完整组合
- **测试粒度：** 跑 `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-core): MemoryContextBuilder 整合 B+C+RAG`

#### Task Q2：MemoryApi + MemoryApiImpl
- **Files：**
  - `sanyan-memory-api/src/main/java/com/sanyan/memory/MemoryApi.java`（接口）
  - `sanyan-memory-api/src/main/java/com/sanyan/memory/dto/{MemoryContext,MemorySummary,MemoryProfile,MemoryFragment}.java`（record）
  - `sanyan-memory-core/api/MemoryApiImpl.java`：implements MemoryApi，委托 MemoryContextBuilder
- **接口：** `MemoryContext getRelevantContext(Long userId, Long characterId, String currentUserMessage)`
- **TDD：** `MemoryApiImplTest` Mockito 覆盖
- **测试粒度：** 跑 `sanyan-memory-api` + `sanyan-memory-core` 包测试
- **Commit：** `feat(memory-api): MemoryApi 契约 + Impl 委托给 MemoryContextBuilder`

#### Task Q3：PromptBuilder 改造（注入 MemoryContext + 短期窗口对齐）
- **位置说明：** PromptBuilder 在 Plan 1 的 `sanyan-business/src/main/java/com/sanyan/chat/internal/PromptBuilder.java`（如果存在）；如果 Plan 1 没有独立 PromptBuilder（直接在 AiService 里拼），本 task **先抽出 PromptBuilder**（重构 Plan 1 代码，遵守"重构后清理规范"——抽出后旧的内联拼接代码必须删除，不允许并存）
- **修改 1（注入记忆）：** `PromptBuilder.build` 增加 `memoryContext` 参数；构造顺序：
  1. system: character.basePrompt
  2. system: 「她对你的记忆：」+ memoryContext 文本（仅当非空）
  3. 短期上下文 `MemoryConstants.SHORT_TERM_WINDOW_SIZE`（= 32）条
- **修改 2（对齐铁律落地）：** Plan 1 `AiService.chat` 当前硬编码 `PageRequest.of(0, 20)`（在 `sanyan-business/src/main/java/com/sanyan/llm/internal/AiService.java:66-67`），本 task **必须**改为 `PageRequest.of(0, MemoryConstants.SHORT_TERM_WINDOW_SIZE)`。**禁止保留字面量 `20` 或 `32`**
- **测试更新：**
  - PromptBuilderTest 加断言：传入 32 条消息时全部进 prompt，传入 33 条时只取最近 32 条
  - AiServiceTest 加回归断言：调用 repository 时传入的 `PageRequest` 大小 == `MemoryConstants.SHORT_TERM_WINDOW_SIZE`（防止后续有人改回字面量）
- **测试粒度：** 跑 `sanyan-business` + `sanyan-memory-core` 包测试
- **Commit：** `feat(chat): PromptBuilder 注入长期记忆 + AiService 短期窗口对齐 MemoryConstants`

#### Task Q4：AiService / ChatService 注入 MemoryApi
- **位置说明：** Plan 1 实际是 `AiService.chat` 编排了 prompt 拼装 + LLM 调用。本 task 在 `AiService` 注入 `MemoryApi`
- **修改：** `AiService` 注入 `MemoryApi`；`chat` 方法开头先调 `memoryApi.getRelevantContext(userId, characterId, currentUserMessage)` 拿 context，再传给 PromptBuilder
- **测试更新：** AiServiceTest 用 mock MemoryApi（返回 null / 空 context / 满 context 三种场景）
- **测试粒度：** 跑 `sanyan-business` + `sanyan-memory-core` 包测试
- **Commit：** `feat(chat): AiService 整合 MemoryApi 注入长期记忆`

### Phase R · 端到端 + 整体审查 + dogfood

#### Task R1：端到端 IT
- **覆盖：** 用户聊够 30 条 → 验证 summary 生成；用户提"我家有只猫"→ 验证 profile_jsonb 含 events；20 天后用户提"上次那只猫"→ 验证 RAG 检索返回相关历史片段
- **测试粒度：** 端到端 IT 本身就是全栈测试，跑全量 + `mvn verify`（含 failsafe）+ Spring `@SpringBootTest` 启动验证
- **Commit：** `test: Plan 2 端到端覆盖 B+C+RAG`

#### Task R2：dogfood 第二关（**USER_ONLY**，subagent 跳过实际试用）
- **subagent 部分（必须做）：** 在 `docs/superpowers/dogfood/` 下创建 `2026-xx-xx-plan-2-dogfood.md` 模板文件，包含下面的 checklist 框架等用户事后填写
- **USER 部分（subagent 不要执行）：** 自己用 3-7 天实测，填写下面的 checklist：
  - [ ] 第 3 天后小婉还记得我说过的事吗？
  - [ ] profile 抽取的 occupation/preferences 准确率？
  - [ ] RAG 检索的相关性？有"答非所问"的情况吗？
  - [ ] LLM 成本日均（DeepSeek V4-Flash 用量）？
- **测试粒度：** 不跑测试（纯文档模板创建）
- **Commit：** `docs: Plan 2 dogfood 反馈模板（subagent 创建框架，用户实测后补充）`

#### Task R3：整体 review + 代码合规性检查（**用户明确要求的 final gate**）
- **目标：** Plan 2 所有 Task 全部完成后，由一个**独立的 superpowers:code-reviewer 子代理**做一次**全局**的代码审查 + 合规性 sweep
- **审查范围（subagent prompt 必注入）：**
  1. **跨 Task 一致性**：常量都从 `MemoryConstants` 取 / `com.sanyan` 包名统一 / 命名规范统一 / commit message 中英文格式统一
  2. **CLAUDE.md 全局规则合规性 sweep**：
     - 代码删除规范：有没有遗留废弃属性、注释掉的死代码？
     - 重构后清理规范：Q3 改完 AiService 有没有清理废弃 import / 旧测试？
     - 代码复用原则：值复用（无硬编码常量）、逻辑复用（重复代码抽函数）
     - static 优先原则：utility 类是不是 `final + private constructor + static`；Spring Bean 用实例方法（不要乱改 static）
     - TDD 铁律：每个 commit 是否对应 RED→GREEN→REFACTOR 三步？所有业务代码必须有先行失败的测试 commit
     - Bug 修复最小改动：Q3 改 AiService 有没有夹带无关改动？
  3. **Java backend 规范 sweep**：
     - 业务领域 `-api` + `-core` 双模块拆分 ✓
     - `-api` 只含接口 + DTO + 事件，不含实现 ✓
     - `-core` 子包结构 `web/internal/api/event/` ✓
     - Service 按动作拆，命名 `<Domain><Action>Service` ✓
     - Entity 带 `Entity` 后缀 ✓
     - 全项目唯一 `BusinessException` + `<Domain>ErrCode` enum ✓
     - 领域事件放发布方 `-api/event/`，record 类型 ✓
     - Object Mother（`<Domain>TestFixtures`）测试构造 Entity 强制 ✓
     - HTTP 响应统一 `BaseResp<T>` ✓
  4. **Plan 2 特定合规检查**：
     - `SHORT_TERM_WINDOW_SIZE > SUMMARY_TRIGGER_THRESHOLD` 对齐铁律有自动化断言 ✓
     - 远程 embedding service 降级策略生效（mock 失败时 RAG 返回空列表）✓
     - `MemoryConstants` 集中，禁止硬编码字面量 ✓
     - 双服务器部署：embedding-bootstrap 实际可在 old 上跑 + 主服务能调通 ✓
     - Maven Enforcer 规则有效（试着伪造一条违规 dep，跑 `mvn validate` 应该 fail）
  5. **测试质量 sweep**：
     - 所有 Service 有 Mockito 单测
     - 所有 Controller 有 `@WebMvcTest` IT
     - Repository 自定义查询有 `@DataJpaTest` IT
     - 关键路径有 `@SpringBootTest` 端到端 IT（R1 覆盖）
     - 测试覆盖率不强求百分比，但每个分支必须有用例
- **执行方式：** 派遣 `superpowers:code-reviewer` agent，**完整 prompt 要包含**：
  - 本 task 的全部审查范围
  - Plan 2 spec 路径
  - 用户 CLAUDE.md 全文路径
  - java-backend* rules 路径
  - 要求：输出"问题清单 + 严重度 + 建议修复方案 + 修复优先级"
- **如果发现高优先级问题（严重/重要级）：** 派遣实现 subagent 修复 → 修复后再派一次 code-reviewer 验证 → 直到全部清掉。**不停下来问用户**
- **如果只是建议性问题（次要/锦上添花）：** 记录到 review 报告里，不强制修复，留待 Plan 3 一并处理
- **输出：** 在 `docs/superpowers/reviews/` 下生成 `2026-xx-xx-plan-2-final-review.md`，含问题清单、修复记录、最终结论（PASS / NEEDS-FIX / BLOCKED）
- **测试粒度：** 全量 `mvn verify` 必须绿（最终 gate）
- **Commit：** `docs(review): Plan 2 整体合规性审查报告 + 修复记录`

---

## Self-Review Checklist

- [x] Spec §4.1 长期记忆 B+C+RAG 三层全覆盖
- [x] Spec §4.1 RAG 锁死参数：BGE-M3 + Top 5 + cos sim > 0.6 + chunk 5-10 条 + ivfflat 索引（参数全部固定，**一期不调优**）
- [x] LLM 后台任务用 V4-Flash（spec §4.2）
- [x] 配套 rule：`java-backend-business-layer.md`（DDD `-api`+`-core` 双模块、Service 按动作拆、Object Mother 强制）严格遵守
- [x] 关键常量对齐铁律 `SHORT_TERM_WINDOW_SIZE > SUMMARY_TRIGGER_THRESHOLD` 落地：常量集中到 `MemoryConstants`、Task N0 自动化断言、Task Q3 把 Plan 1 的硬编码 `20` 迁移到常量
- [x] 双服务器部署拓扑确定：embedding service 拆到 old，HTTP 公网调用，多重 IP 白名单（Lighthouse 防火墙 + ufw + 应用层 token）已经在运维侧验证过 12-15ms RTT
- [x] 远程依赖降级策略：embedding server 挂时 RAG 返回空列表、聊天不挂；异步索引失败走 Redis Streams 重试 + dead-letter
- [x] Subagent prompt 必注入项明确（TDD 铁律 + 项目级上下文 + 验证步骤 + 子代理类型 + 完成判定）
- [x] R3 整体 review task 作为 final gate，全局合规性 sweep
- [x] Service 命名遵守 `<Domain><Action>Service` 规范（如 MemorySummaryService、MemoryProfileExtractService、MemoryRagSearchService 等）

---

## Execution Handoff

**Plan 1 已经合并并跑通。本 Plan 2 由主对话以 subagent-driven 方式自动执行，无 per-task checkpoint（用户已授权）。每个 Task 完成后自动进入下一个 Task；所有 Task 完成后自动跑 R3 整体 review，再用 superpowers:finishing-a-development-branch 收尾。**
