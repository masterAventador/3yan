# Plan 3 整体对照 plan Final Review

> 日期：2026-05-18
> 范围：Plan 3「关系养成系统」实施完成后对照 spec + plan 逐 task 核对
> 由 Claude 自己亲自执行（用户在 K3 task 中特别要求）

## 1. 完成度总览

| Phase | Tasks | 状态 | 备注 |
|---|---|---|---|
| A · DB migration | A1-A5 (5) | ✅ 全部完成 | V2-V6 migration + ERROR_CODE_REGISTRY 同步 |
| B · Entity + Repo + Fixture | B1-B4 (4) | ✅ 全部完成 | 4 个 Entity / Repository / Fixture |
| C · Stage 基础 | C1-C3 (3) | ✅ 全部完成 | Properties / Definition / Transition / Override |
| D · 亲密度核心 | D1-D5 (5) | ✅ 全部完成 | ErrCode / Event / Calculator / Counter / Login / RecordService |
| E · 剧情节点规则 | E1-E4 (4) | ✅ 全部完成 | Rule 接口 + Engine + 2 个具体 rule + StoryListener |
| F · AI 评估 | F1 (1) | ✅ 完成 | ConversationQualityEvaluator |
| G · 事件 + Listener | G1, G2 (2) | ✅ 完成 | 事件契约 + MessagePersistedListener |
| H · CharacterApi 扩 | H1-H4 (4) | ✅ 全部完成 | DTO / FindOrCreate / Fetch / ApiImpl |
| I · Controller + chat-core 接入 | I1-I4 (4) | ✅ 全部完成 | Controller / PromptBuilder / AiService / WS |
| J · 前端 Flutter | J1-J6 (6) | ✅ 全部完成 | model / api / 2 个 widget / controller / suite |
| K · 端到端 + dogfood + review | K1 ✅ / K2 ⏳ / K3 ✅ | K2 留给用户 | dogfood 需人工 7 天 |

**实施完成：40/41 task（K2 dogfood 留待用户人工跑 7 天）**

## 2. 提交记录

- **主仓 worktree**（4 commit，全部 docs）：spec 重写 + 自检 + 规范符合性修复 + plan 完整 task 列表
- **server submodule plan3 分支**：**45 个 commit**
- **app submodule plan3 分支**：**6 个 commit**

## 3. 测试 final gate（K3 验证）

```bash
cd /Users/aventador/code/3yan-plan3/server && mvn verify
```

**结果：BUILD SUCCESS** ✅
- 21 个 Maven 模块全部 SUCCESS
- 耗时 34.7s
- 含 Unit Test + IT（含 Testcontainers PG + Redis 的 IT，含 MessageFlowE2EIT 端到端）

```bash
cd /Users/aventador/code/3yan-plan3/app/business_packages/sanyan_chat && fvm flutter test
```

**结果：All 28 tests passed!** ✅
- 含 Relationship model / chat_api / IntimacyProgressBar / StageTransitionDialog / chat_controller 测试

## 4. Spec 关键文件核对（plan §3 模块结构 + §6 前端组件）

### character-api 新增（plan §3.1 要求）
- ✅ `dto/RelationshipDto.java`
- ✅ `event/IntimacyChangedEvent.java`
- ✅ `event/StageTransitionEvent.java`
- ✅ `event/StageEntryStoryEvent.java`（spec 在 §3.2 E4 task 中追加）
- ✅ `CharacterApi.java`（扩 3 个新方法 `findOrCreateRelationship` / `fetchMyRelationship` / `getStagePromptSegment`）

### character-core 新增（plan §3.2 要求）

| 路径 | 状态 |
|---|---|
| `internal/AiCharacterEntity.java`（加 basePrompt + personaConfig） | ✅ |
| `internal/CharacterErrCode.java`（加 3002 / 3003） | ✅ |
| `internal/RelationshipEntity.java` + `RelationshipId.java` | ✅ |
| `internal/RelationshipRepository.java` | ✅ |
| `internal/RelationshipFindOrCreateService.java` | ✅ |
| `internal/RelationshipFetchService.java` | ✅ |
| `internal/intimacy/IntimacyEvent.java` | ✅ |
| `internal/intimacy/IntimacyCalculator.java` | ✅ |
| `internal/intimacy/IntimacyRecordService.java` | ✅ |
| `internal/intimacy/IntimacyLogEntity.java` + Repository | ✅ |
| `internal/intimacy/IntimacyProperties.java`（@ConfigurationProperties） | ✅ |
| `internal/intimacy/DailyBehaviorCounter.java`（Redis） | ✅ |
| `internal/intimacy/ConsecutiveLoginService.java`（Redis） | ✅ |
| `internal/intimacy/ai/ConversationQualityEvaluator.java` | ✅ |
| `internal/intimacy/ai/QualityScoreResponse.java` | ✅ |
| `internal/plotrule/PlotMilestoneRule.java`（接口） | ✅ |
| `internal/plotrule/PlotMilestoneEngine.java` | ✅ |
| `internal/plotrule/MessageContext.java` | ✅ |
| `internal/plotrule/DeepNightChatRule.java` | ✅ |
| `internal/plotrule/FirstHonestShareRule.java` | ✅ |
| `internal/plotrule/RelationshipMilestoneEntity.java` + Id + Repo | ✅ |
| `internal/stage/StageDefinition.java` | ✅ |
| `internal/stage/StageTransitionDetectService.java` | ✅ |
| `internal/stage/StageOverrideQueryService.java` | ✅ |
| `api/CharacterApiImpl.java`（扩 3 个 relationship 方法实现） | ✅ |
| `web/RelationshipController.java` | ✅ |
| `event/MessagePersistedListener.java` | ✅ |
| `event/StageTransitionStoryListener.java` | ✅ |

### chat-core 修改（plan §3.3 要求）

| 路径 | 状态 | 说明 |
|---|---|---|
| `internal/PromptBuilder.java` | ✅ | `build` 加 stagePromptSegment 参数 |
| `internal/AiService.java` | ✅ | 调 `characterApi.getStagePromptSegment` |
| `ws/ChatWebSocketHandler.java` | ✅ | 加 3 个 @EventListener |
| `ws/WsIntimacyUpdate.java` / `WsStageTransition.java` / `WsStageStory.java` | ✅ | 3 个新 frame DTO |

### App / sanyan_chat 新增（plan §6 要求）

| 路径 | 状态 |
|---|---|
| `lib/src/api/models/relationship.dart` | ✅ |
| `lib/src/api/req/relationship_req.dart` | ✅ |
| `lib/src/api/chat_api.dart` 加 `fetchMyRelationship` | ✅ |
| `lib/src/chat/widgets/intimacy_progress_bar.dart` | ✅ |
| `lib/src/chat/widgets/stage_transition_dialog.dart` | ✅ |
| `lib/src/chat/chat_controller.dart` 监听 ws frame | ✅ |
| `test/sanyan_chat_suite.dart` | ✅ |

### 数据库 migration（plan §2.6 要求）

| 文件 | bootstrap 主拷贝 | memory-core 测试副本 | character-core 测试副本 |
|---|---|---|---|
| `V2__ai_character_add_persona.sql` | ✅ | ✅ | ✅ |
| `V3__create_relationships.sql` | ✅ | ✅ | ✅ |
| `V4__create_intimacy_logs.sql` | ✅ | ✅ | ✅ |
| `V5__create_relationship_milestones.sql` | ✅ | ✅ | ✅ |
| `V6__seed_xiaowan_persona.sql` | ✅ | ✅ | ✅ |

`FlywayMigrationSyncTest`（memory-core + character-core 各一份）守护 byte-identical 同步。

### 配套 docs

- ✅ `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md` 加 3002 / 3003 + 历史变更（A5 task）
- ✅ `bootstrap/src/main/resources/application.yml` 加 `sanyan.intimacy.*` 配置（C1 task）

## 5. 规范规则对照（`~/.claude/rules/`）

### Java 后端规范

| 规则 | 状态 |
|---|---|
| 一个 -api 一个主 `<Domain>Api` 接口 | ✅ 只扩 `CharacterApi`，不新建 RelationshipApi |
| Service 按动作命名（`<Domain><Action>Service`） | ✅ `IntimacyRecordService` / `StageTransitionDetectService` / `StageOverrideQueryService` / `RelationshipFindOrCreateService` / `RelationshipFetchService` |
| Object Mother Fixture 强制 | ✅ `RelationshipTestFixtures` / `IntimacyLogTestFixtures` / `RelationshipMilestoneTestFixtures`；`AiCharacterTestFixtures` 加 `withPersonaConfig` 工厂 |
| ERROR_CODE_REGISTRY 同步 | ✅ 3002 / 3003 同步 |
| 事件 record + 默认 AFTER_COMMIT @Async | ✅ 4 个事件全是 record；`MessagePersistedListener` 用 AFTER_COMMIT @Async |
| Controller 不写业务逻辑 | ✅ `RelationshipController` 仅薄壳调 `CharacterApi` |
| `-core` 之间互不依赖（Maven Enforcer 守护） | ✅ `mvn verify` 全绿，未触发 banned-dependencies |
| Object Mother + TDD 铁律 | ✅ 41 task 全部 TDD（RED → GREEN → COMMIT），所有 Entity 都通过 Fixture 构造 |

### Flutter 规范

| 规则 | 状态 |
|---|---|
| 业务包内 `api/req/`（单数，非 reqs/） | ✅ |
| Widget 与 page 分离 | ✅ 2 个新 widget 都是 StatelessWidget |
| 跨接口复用 model 放 `api/models/` | ✅ `Relationship` 放 models/ |
| 单接口 Req + Data 同文件 | ✅ `FetchMyRelationshipReq` + `FetchMyRelationshipData` 同 `relationship_req.dart` |
| 业务包 `test/<pkg>_suite.dart` 聚合入口 | ✅ J6 task 创建（项目此前漏写） |

## 6. 实际实施与 plan 的偏差（值得记录）

### 重要架构调整（在实施过程中发现并修复）

1. **Testcontainers + Colima 兼容**（A1 期间发现）
   - plan 模板写 `@DataJpaTest`（默认 H2），但 H2 不支持 jsonb / pgvector，IT 必须走 Testcontainers PG
   - **修复路径**：A1 第一版 commit 把 surefire/failsafe 配置硬编码到根 pom，后被发现是 scope 越界。最终重构：把 `api.version=1.44` + `RYUK_DISABLED=true` 上提到父 pom pluginManagement，统一给所有子模块用；DOCKER_HOST 由 `~/.testcontainers.properties` 提供
   - 这次重构同时清理了 bootstrap / memory-core 的重复 surefire 配置

2. **`FlywayMigrationSyncTest` 守护范围扩展**（A1 code review I-1）
   - 原项目只在 memory-core 守护 bootstrap ↔ memory-core 双拷贝
   - A1 在 character-core 加了第三份测试副本，需要对等守护测试 → 在 character-core 新建 `FlywayMigrationSyncTest`

3. **PG 类型字符串映射**（A2 期间发现）
   - plan IT 模板写 `typeName() == "int8"`，实际 `PostgresTestcontainerSupport.describeColumns` 对无 serial sequence 的 BIGINT 返回 `"bigint"`（带 serial 的才是 `"bigserial"`）
   - 后续 A3-A4 IT 都用了正确的 PG 字符串

4. **`CharacterApi` 暴露 3 个方法而非 4 个**（spec 自检阶段）
   - plan brainstorm 阶段写 `recordIntimacyEvent` 也暴露到 -api
   - spec self-review 时识别为不必要（character-core 内部调 internal Service 即可）→ 删

5. **`LlmApi.chat(BACKGROUND, messages)` 而非 `callForText / QUALITY_EVAL`**（F1 task）
   - plan 写了不存在的 API 方法和枚举值
   - F1 实施时改用现有 `chat` 接口 + `BACKGROUND` task type

6. **`@LoginUser Long userId` 注入而非 `@LoginRequired + CurrentUser.id()`**（I1 task）
   - plan 用的是 spec 通用模板写法
   - 项目惯例是 `@LoginUser` argument resolver（参考 MessageController）

7. **MessagePersistedEvent 实际字段**（G2 期间发现）
   - plan 模板写 `senderType / content / createdAt Instant`，实际是 `messageId / userId / characterId / role`
   - G2 implementer 用实际字段；`ChatApi.listRecentByUser` 拉最近消息而非直接查 MessageRepository

8. **`ChatApplicationContextIT` 加 @MockBean**（F1/G2 后 regression）
   - F1 / G2 给 character-core 引入了 ChatApi / LlmApi 跨模块依赖
   - character-core 测试上下文没有这些 -core 实现 → 加 `@MockBean ChatApi chatApi; @MockBean LlmApi llmApi;`

9. **`sanyan_network/WsEvent` 加 `rawJson` 字段 + ws_event_type 加 3 个常量**（J5 顺手）
   - 让 chat_controller 能读 ws frame 的扩展字段（score / story_message 等）
   - 这是基础设施层小幅扩展，合理改动

10. **`sanyan_chat_suite.dart` 项目此前漏写**（J6 task）
    - flutter-business-layer.md §10 强制要求，但项目实际只有 sanyan_util_suite.dart
    - J6 task 顺手补齐 + 聚合所有现有测试

### 子代理小问题（已识别并处理）

- **A1 第一版 push 到 origin**：子代理擅自 push（CLAUDE.md 要求"只 commit 不 push"未明确传达）。后续 prompt 都加了"不准 push"强约束，未再发生
- **E4 task StageEntryStoryEvent 文件未真创建**（F1 子代理顺手补）：implementer 报告完成但实际 git 未 add；F1 implementer 编译时发现并修

## 7. 未达成的 DoD 条款

按 plan §8.4 DoD：
- ✅ 所有自动化测试通过（`mvn verify` + `flutter test` 全绿）
- ⏳ 7 天 dogfood 完成 + 每天一篇 `docs/dogfood/plan-3-day-N.md` —— **K2 task，留给用户人工跑**
- ⏳ 数值调参至少一轮 —— **依赖 K2 dogfood 反馈**
- ⏳ 切换阶段时小婉称呼/语调感受"自然"主观验收 —— **依赖 K2 dogfood**
- ⏳ plan-3.md 末尾追加「dogfood 反馈 + 最终数值」段 —— **dogfood 完成后追加**

## 8. Follow-up 建议（非阻塞）

1. **重构 `MessagePersistedListener.ensureRelationship` inline 改走 `RelationshipFindOrCreateService`**
   - G2 task 当时 H2（FindOrCreateService）还没建，临时 inline。现在 H2 已就位，可重构走 service 让代码更整洁。但 inline 的实现也正确 work，非阻塞。

2. **`memory_profiles.version DEFAULT 1` vs `relationships.version DEFAULT 0` 不一致**
   - A2 code review 发现的 Important issue，被评估为"memory_profiles 历史包袱，不在 A2 scope"defer
   - 若未来想统一，可在后续 plan 把 memory_profiles 改为 DEFAULT 0（与 JPA `@Version` 默认行为一致）

3. **API 版本 `api.version=1.44` 是否升到更高**
   - 当前父 pom pluginManagement 锁 1.44。Docker Engine 已支持更高（实测 1.53 downgraded from 1.54）
   - 若未来 Testcontainers 升级触发兼容问题，可考虑升

## 9. 总结

**Plan 3「关系养成系统」的 40 个可代码化 task 全部完成。1 个 dogfood 任务留给你人工跑 7 天。**

- 后端：21 个 Maven 模块 + 45 commit
- 前端：sanyan_chat 包 + 6 commit
- 数据库：5 个 migration（V2-V6）
- 测试：单元测试 + IT + Testcontainers PG/Redis IT + E2E final gate 全绿
- 完整链路验证（MessageFlowE2EIT）：消息事件 → AFTER_COMMIT → 涨分 → DB 状态正确

下一步：
- 用户跑 K2 dogfood 7 天
- 根据 dogfood 反馈调 `application.yml` 数值
- 完成后追加「dogfood 反馈 + 最终数值」段
- 准备 merge plan3 → master
