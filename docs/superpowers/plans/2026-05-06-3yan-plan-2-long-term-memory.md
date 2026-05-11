# 三言 · Plan 2 实施计划：长期记忆系统（B 摘要 + C 档案 + RAG）

> **For agentic workers:** REQUIRED SUB-SKILL: `superpowers:subagent-driven-development`. Steps use `- [ ]` checkboxes.
>
> **TDD 铁律：每个 task 严格 Red → Green → Refactor 5 步。没有失败测试不允许写生产代码。**
>
> **前置依赖：Plan 1 必须已合并并跑通。本 plan 在 Plan 1 产出的代码基础上增量。**
>
> **配套规则：`~/.claude/rules/java-backend*.md`（路径触发自动加载，子代理执行时也会触发）。**

**Goal：** 让林小满"记得"用户。聊到第 3 天用户提"上次说的那只猫"她能接住；提"我家那个项目"她能想起背景。dogfood 第二关：聊到第 3 天会不会失忆？

**Architecture：**
- **B 滚动摘要**：每 N 条消息（默认 30）后台异步生成一段"对话纪要"，存 `memory_summaries` 表
- **C 结构化档案**：每次用户发消息后异步抽取结构化字段（基本信息 / 偏好 / 关键事件 / 情感线），merge 到 `memory_profiles` 表的 `profile_jsonb`
- **RAG 向量检索**：每次对话轮组完成后，把该轮组（5-10 条消息）embedding 存入 `chat_embeddings`（pgvector）；用户发消息时按语义相似度检索 Top 5 历史片段塞进 prompt
- **后台任务**：用 Redis Streams（不是消息队列重量级方案）作为简单异步任务调度
- **B/C 提取走 DeepSeek V4-Flash**（便宜）；RAG embedding 走 BGE-M3（**先选 BGE-M3，因为开源可本地部署，不依赖外部 API key**，避免 Plan 1 已经依赖一家 LLM 之外又新加 API 依赖）

**Tech Stack：**
- 在 Plan 1 基础上新增：
  - 后端：DeepSeekAdapter（接 DeepSeek V4-Flash API）、BGE-M3 通过 Hugging Face Inference / 本地 fastembed-java（一期决策见下文 Task M2）、pgvector JPA 集成（`com.pgvector:pgvector` Java client + 自定义 Hibernate `UserType`）
  - **不引入新组件**：用现有 PostgreSQL + Redis Streams

---

## 文件结构（Plan 1 之后的增量）

### 新增模块

```
business_packages/
├── sanyan-memory-api/                            # 新模块
│   └── src/main/java/com/3yan/memory/
│       ├── MemoryApi.java                       # 对外契约：getRelevantContext(userId, characterId, query)
│       └── dto/{MemoryContext,MemorySummary,MemoryProfile,MemoryFragment}.java
└── sanyan-memory-core/                           # 新模块
    └── src/main/java/com/3yan/memory/
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
└── src/main/java/com/3yan/llm/internal/
    ├── DeepSeekAdapter.java                     # 新增 V4-Flash 适配器
    └── EmbeddingProvider.java + BgeM3Provider.java  # 新增 embedding 抽象

business_packages/sanyan-chat-core/  （扩展）
└── src/main/java/com/3yan/chat/internal/
    └── PromptBuilder.java  ← 修改：注入 MemoryContext 替代纯短期上下文
```

### 数据库迁移

```
bootstrap/src/main/resources/db/migration/
├── V5__init_memory_summaries.sql
├── V6__init_memory_profiles.sql
└── V7__init_chat_embeddings.sql                # CREATE EXTENSION IF NOT EXISTS vector + 表 + ivfflat 索引
```

---

## Task 列表

### Phase L · 数据库 schema

#### Task L1：V5 memory_summaries
- **Files：** `bootstrap/src/main/resources/db/migration/V5__init_memory_summaries.sql`
- **Schema：** id BIGSERIAL / user_id / character_id / period_start_message_id / period_end_message_id / summary_text TEXT / message_count INT / created_at；索引 (user_id, character_id, created_at DESC)
- **测试：** 启动后跑 Flyway 迁移成功，psql 验证表结构
- **Commit：** `feat(db): V5 memory_summaries 滚动摘要表`

#### Task L2：V6 memory_profiles
- **Files：** `bootstrap/src/main/resources/db/migration/V6__init_memory_profiles.sql`
- **Schema：** user_id (PK) / character_id (PK) / profile_jsonb JSONB NOT NULL DEFAULT '{}' / version INT DEFAULT 1（乐观锁）/ updated_at；初始 jsonb schema：
  ```json
  {
    "basic_info": { "name": null, "age": null, "occupation": null, "hometown": null },
    "preferences": { "foods": [], "movies": [], "colors": [], "music": [] },
    "events": [],
    "emotion_line": []
  }
  ```
- **Commit：** `feat(db): V6 memory_profiles 结构化档案表`

#### Task L3：V7 chat_embeddings + pgvector
- **Files：** `bootstrap/src/main/resources/db/migration/V7__init_chat_embeddings.sql`
- **SQL：** `CREATE EXTENSION IF NOT EXISTS vector;` + `chat_embeddings` 表（id / user_id / character_id / message_ids BIGINT[] / chunk_text TEXT / embedding vector(1024) / token_count INT / created_at）+ `CREATE INDEX ON chat_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);`
- **测试：** 启动后用 psql 验证扩展加载成功 `SELECT * FROM pg_extension WHERE extname='vector';`
- **Commit：** `feat(db): V7 chat_embeddings + pgvector ivfflat 索引`

### Phase M · LLM 扩展（DeepSeek + Embedding）

#### Task M1：DeepSeekAdapter
- **Files：** `sanyan-llm-core/src/main/java/com/3yan/llm/internal/DeepSeekAdapter.java` + 测试
- **关键：** OpenAI 兼容协议（DeepSeek API 走 `https://api.deepseek.com/chat/completions` 兼容格式）；`supports() == LLMTaskType.BACKGROUND`；模型名 `deepseek-v4-flash`；`application-local.yml` 加 `3yan.llm.deepseek.apiKey=${DEEPSEEK_API_KEY:}`
- **测试：** MockWebServer 模拟 OpenAI Chat Completion 协议，覆盖成功 / 401 / 429 / 500
- **TDD 5 步严格执行**
- **Commit：** `feat(llm-core): DeepSeekAdapter（V4-Flash，BACKGROUND task）`

#### Task M2：EmbeddingProvider 抽象 + BGE-M3 实现
- **Files：** 新增 `sanyan-llm-api/dto/EmbeddingRequest,EmbeddingResponse.java` + `EmbeddingProvider` 接口；`sanyan-llm-core/internal/BgeM3Provider.java`
- **决策：** 一期 BGE-M3 走 **`fastembed-java`**（`io.qdrant:fastembed-java`），本地 ONNX 推理，启动时下载模型到本地（约 1.2GB），`embed(text)` 返回 1024 维向量。**不依赖外部 API key**。如果 ECS 4G 内存吃紧（fastembed 加载模型约占 1G），改走 Hugging Face Inference API（需 HF token）作为 fallback；本期 Plan 决策走 `fastembed-java`，spec 后续可调整
- **测试：** 单测 mock + IT 实际跑 embedding 看维度是 1024
- **TDD：** 测试 → fail → 实现 → 通过
- **Commit：** `feat(llm-core): EmbeddingProvider + BgeM3 本地 ONNX 推理`

#### Task M3：注册多 Adapter 到 Factory
- **Files：** 修改 `LLMProviderFactory`、确认 `LLMProviderRouter.chat(LLMTaskType.BACKGROUND, ...)` 走 DeepSeek
- **Commit：** `feat(llm-core): Factory 注册 DeepSeek + Embedding`

### Phase N · 摘要（B 模块）

#### Task N1：MemorySummaryEntity + Repository + Fixtures
- **TDD：** Repository IT 测试，验证 `findLatestByUserAndCharacter` 等查询
- **Commit：** `feat(memory-core): summary entity + repository`

#### Task N2：SummaryService（生成摘要）
- **关键：** 输入 `List<MessageEntity>`（30 条），输出摘要文本；调 `LLMProviderRouter.chat(LLMTaskType.BACKGROUND, ...)` 用 DeepSeek，prompt 模板：
  ```
  下面是用户和「林小满」的一段对话（30 条），请你总结成一段简洁的"对话纪要"，
  保留关键事实、情感线索、用户提到的人事物。100-200 字。
  对话：{{messages}}
  纪要：
  ```
- **TDD：** Mockito 测试，传入 fixture messages → 验证 prompt 构造正确 + 委托给 LLM router
- **Commit：** `feat(memory-core): SummaryService（DeepSeek V4-Flash 生成纪要）`

#### Task N3：SummaryScheduler（消息落库后异步触发）
- **关键：** 监听 `MessagePersistedEvent`（需要在 Plan 1 chat-core 已发的事件，如果没发，本任务先在 chat-core 加上事件发布）；统计自上次摘要以来新增消息数 ≥ 30 → 触发 SummaryService
- **TDD：** Mockito 单测覆盖触发 / 不触发 / 边界
- **Commit：** `feat(memory-core): SummaryScheduler 30 条阈值触发`

### Phase O · 结构化档案（C 模块）

#### Task O1：MemoryProfileEntity + Repository
- **关键：** JSONB 字段映射用 `JsonType`（hibernate-types 库或 Hibernate 6.4+ 内置 `@JdbcTypeCode(SqlTypes.JSON)`）
- **Commit：** `feat(memory-core): profile entity + repository`

#### Task O2：ProfileExtractService
- **关键：** 输入用户的单条消息（或最近 N 条），调 DeepSeek V4-Flash，prompt 要求输出 JSON：
  ```json
  {
    "basic_info_updates": {"occupation": "前端工程师"},
    "preferences_appended": {"foods": ["寿司"]},
    "events_appended": [{"type": "interview", "content": "周三技术面试", "date": null}],
    "emotion_signal": "焦虑"
  }
  ```
- **TDD：** Mockito 测试 LLM 返回 fake JSON，验证服务方法正确解析
- **Commit：** `feat(memory-core): ProfileExtractService 结构化抽取`

#### Task O3：ProfileMergeService（JSONB 增量 merge）
- **关键：** 用 `jsonb_set` 或 Java 端读出 → 合并 → 写回（带乐观锁 version）；用户名字优先保留首次抽取的值，避免 LLM 反复改写
- **TDD：** 单测覆盖：basic_info 已存在不覆盖 / preferences 数组去重 append / events append / emotion_line append（保留最近 50 条）
- **Commit：** `feat(memory-core): ProfileMergeService（增量 merge + 乐观锁）`

#### Task O4：触发链路（每条用户消息后异步抽取）
- **关键：** Listener 监听 `UserMessagePersistedEvent`（或 `MessagePersistedEvent` filter role=user）→ 调 ProfileExtractService → ProfileMergeService
- **降速：** 同一用户每 5 分钟最多触发一次，避免连续短消息 LLM 浪费
- **Commit：** `feat(memory-core): 用户消息抽取触发链路 + 5 分钟节流`

### Phase P · RAG 向量检索

#### Task P1：PgVectorType（Hibernate UserType）
- **关键：** 把 `float[]` 与 PG `vector(1024)` 映射；用 `com.pgvector:pgvector-java` SDK 的 `PGvector` 类
- **TDD：** Repository IT 用 Testcontainers postgres + pgvector 镜像（`pgvector/pgvector:pg15`），保存查询往返
- **Commit：** `feat(memory-core): PgVectorType Hibernate 映射`

#### Task P2：ChunkBuilder
- **关键：** 输入 `List<MessageEntity>`（按时间序），输出 `List<Chunk>`，每 chunk 5-10 条消息（按 token 数 < 400 切）；返回 `List<Long>` message_ids + 拼接后的 chunk_text
- **TDD：** 单测覆盖切片
- **Commit：** `feat(memory-core): ChunkBuilder 5-10 条切片`

#### Task P3：EmbeddingService + ChatEmbeddingEntity
- **关键：** Service 接受 chunk_text → 调 `BgeM3Provider.embed` → 拿到向量 → 保存 ChatEmbeddingEntity；批量入库
- **TDD：** Mockito + Repository IT
- **Commit：** `feat(memory-core): EmbeddingService 批量入库`

#### Task P4：RagSearchService
- **关键：** 输入 `userId, characterId, queryText`，先 embed query → repo 查 cosine similarity Top 5 且 > 0.6 → 返回 List<MemoryFragment>（含 chunk_text + 时间）
- **SQL：** `SELECT * FROM chat_embeddings WHERE user_id=? AND character_id=? ORDER BY embedding <=> ? LIMIT 5`（pgvector cosine distance 操作符 `<=>`）
- **TDD：** IT 测试：插入 fixture chunks → 用一段相似 query embed → 验证返回
- **Commit：** `feat(memory-core): RagSearchService 语义检索 Top 5`

#### Task P5：异步索引 trigger（消息落库后）
- **关键：** Listener 监听 `MessagePersistedEvent`，累积到 5-10 条后触发 ChunkBuilder + EmbeddingService 落库；用 Redis Streams 实现简单 worker（避免阻塞主消息流）
- **Commit：** `feat(memory-core): 消息落库后异步索引到 pgvector`

### Phase Q · 整合：MemoryContextBuilder + PromptBuilder 改造

#### Task Q1：MemoryContextBuilder
- **关键：** 输入 `userId, characterId, queryText, recentMessages`，返回组合后的 `MemoryContext`：
  - 最新一段 summary（如有）
  - profile_jsonb 转纯文本"她记得的关于你的事"
  - RAG 检索 Top 5 fragments（如有）
- **输出格式：** 一段插入到 system prompt 之后、user/assistant 历史之前的"她对你的记忆"段
- **TDD：** Mockito 单测覆盖各组件返回不同情况下 context 构造正确
- **Commit：** `feat(memory-core): MemoryContextBuilder 整合 B+C+RAG`

#### Task Q2：MemoryApi + ApiImpl
- **接口：** `MemoryContext getRelevantContext(Long userId, Long characterId, String currentUserMessage)`
- **Commit：** `feat(memory-api): 暴露给 chat-core 的契约`

#### Task Q3：PromptBuilder 改造（注入 MemoryContext）
- **修改：** Plan 1 的 `PromptBuilder.build` 增加 `memoryContext` 参数；构造顺序：
  1. system: character.basePrompt
  2. system: 「她对你的记忆：」+ memoryContext 文本（仅当非空）
  3. 短期上下文 30 条
- **测试更新：** PromptBuilderTest 加新断言
- **Commit：** `feat(chat-core): PromptBuilder 注入长期记忆`

#### Task Q4：ChatService 注入 MemoryApi
- **修改：** ChatService 注入 MemoryApi；`handleUserMessage` 中先调 `memoryApi.getRelevantContext` 拿 context，再传给 PromptBuilder
- **测试更新：** ChatServiceTest 用 mock MemoryApi
- **Commit：** `feat(chat-core): ChatService 整合 MemoryApi`

### Phase R · 端到端 + dogfood

#### Task R1：端到端 IT
- **覆盖：** 用户聊够 30 条 → 验证 summary 生成；用户提"我家有只猫"→ 验证 profile_jsonb 含 events；20 天后用户提"上次那只猫"→ 验证 RAG 检索返回相关历史片段
- **Commit：** `test: Plan 2 端到端覆盖 B+C+RAG`

#### Task R2：dogfood 第二关
- **目标：** 自己用 3-7 天，记录 dogfood checklist：
  - [ ] 第 3 天后林小满还记得我说过的事吗？
  - [ ] profile 抽取的 occupation/preferences 准确率？
  - [ ] RAG 检索的相关性？有"答非所问"的情况吗？
  - [ ] LLM 成本日均（DeepSeek V4-Flash 用量）？
- **Commit：** `docs: Plan 2 dogfood 反馈`

---

## Self-Review Checklist

- [x] Spec §4.1 长期记忆 B+C+RAG 三层全覆盖
- [x] Spec §4.1 RAG 锁死参数：BGE-M3 + Top 5 + cos sim > 0.6 + chunk 5-10 条 + ivfflat 索引（参数全部固定，**一期不调优**）
- [x] LLM 后台任务用 V4-Flash（spec §4.2）
- [x] 配套 rule：`java-backend-business-layer.md`（DDD `-api`+`-core` 双模块、Service 按动作拆、Object Mother 强制）严格遵守

---

## Execution Handoff

**Plan 1 跑完后再启动 Plan 2 的 subagent-driven 流程。**
