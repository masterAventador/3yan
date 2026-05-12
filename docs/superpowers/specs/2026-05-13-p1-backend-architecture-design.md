# P1：后端架构对齐设计 spec

> Plan-1 收尾 sub-plan 系列的第 1 个。配套 brainstorm 可视化产物保留在 `.superpowers/brainstorm/`（设计期间使用，会清理）。

## 1. 背景与目标

### 1.1 现状
- 后端 `sanyan-server` 是单 Maven module
- 包结构按"技术分层"组织（`controller / service / entity / repository / dto / ...`）
- 异常用裸 `IllegalArgumentException` + 一个自定义 `InvalidTokenException`
- DB schema 由 Hibernate `ddl-auto: update`（dev）/ `validate`（prod）+ `data.sql` 管
- 测试 194 个，无 ArchUnit 架构守护

### 1.2 目标
对齐 plan-1（`docs/superpowers/plans/2026-05-06-3yan-plan-1-mvp-with-ws.md`）设计，但根据当前项目规模（单人 + 单业务领域）做务实简化：foundation 8 个包按规范拆分，业务层暂保留单 module（不拆 `-api/-core`），等 plan-2 引第二个业务领域时再拆。

### 1.3 非目标
- ❌ 不拆 `-api/-core`（推迟到 plan-2）
- ❌ 不动前端
- ❌ 不引 CI/CD（暂无）
- ❌ 不做内容安全（P1 范围外）
- ❌ 不做 SMS 验证码登录（P0，跳过）

## 2. 架构

### 2.1 Maven 模块拓扑（共 10 个）

```
sanyan-server/                          ← 父 POM（dependencyManagement + Enforcer）
├── bootstrap/                          ← 启动工程（main + application.yml）
├── business_packages/
│   └── sanyan-business/                ← 唯一业务模块
└── foundation_packages/
    ├── sanyan-common-error/            ← BusinessException + ErrCode 接口
    ├── sanyan-common-web/              ← BaseResp + GlobalExceptionHandler
    ├── sanyan-common-auth/             ← JwtUtil + WebSocketInterceptor + LoginUser ArgumentResolver
    ├── sanyan-common-cache/            ← KvCache (Redis 封装)
    ├── sanyan-common-storage/          ← 空骨架，预留对象存储
    ├── sanyan-common-util/             ← TextProcessor / TypingDelayCalculator
    ├── sanyan-common-test/             ← ArchUnit 基类（后续可加 BaseIntegrationTest）
    └── sanyan-common-ws/               ← WsSessionManager + WS 协议 DTO
```

### 2.2 依赖方向（Enforcer + ArchUnit 双层守护）

| 关系 | 允许 |
|------|------|
| `bootstrap` → 所有 | ✅ |
| `sanyan-business` → `foundation_packages/*` | ✅ |
| `foundation_packages` 之间互依 | ✅（避免循环） |
| `foundation_packages` → `business_packages` | ❌ 严禁 |

**第一层守护：父 POM Enforcer**

每个 foundation 子模块的 pom.xml 都加 `maven-enforcer-plugin` 的 `banned-dependencies` 规则：

```xml
<plugin>
  <artifactId>maven-enforcer-plugin</artifactId>
  <executions><execution>
    <id>enforce-foundation-no-business-dependency</id>
    <goals><goal>enforce</goal></goals>
    <configuration><rules>
      <bannedDependencies>
        <excludes>
          <exclude>com.sanyan:sanyan-business</exclude>
        </excludes>
      </bannedDependencies>
    </rules></configuration>
  </execution></executions>
</plugin>
```

违规时 `mvn validate` 阶段直接失败，比 ArchUnit 测试阶段更早拦住。
**第二层守护**（ArchUnit）见 §5。

### 2.3 业务模块内部 4 领域包

```
sanyan-business/src/main/java/com/sanyan/
├── user/
│   ├── web/        AuthController
│   └── internal/   AuthService, SmsService, UserEntity, UserRepository, UserErrCode
├── character/
│   └── internal/   AiCharacterEntity, AiCharacterRepository, CharacterErrCode
├── chat/
│   ├── web/        MessageController
│   ├── ws/         ChatWebSocketHandler + WsXxx DTO（业务相关 DTO）
│   └── internal/   MessageService, MessageEntity, MessageRepository, ChatErrCode
└── llm/
    └── internal/   AiService, LlmErrCode
```

**跨领域调用规则**：
- ✅ 任一领域的 `.internal` 可被同业务模块内其他领域 `.internal` 调用（如 `chat.internal` 调 `llm.internal.AiService`）
- ❌ 不允许跨领域引用 `.web` / `.ws` 包（web/ws 是领域对外的入口，不应被同业务的其他领域绕过）

**与规则 `java-backend-business-layer.md` §3.1 的已知偏差**：
| 规则要求的子包 | 本 spec 状态 | 偏差来源 |
|---|---|---|
| `web/` | ✅ 有 | — |
| `internal/` | ✅ 有 | — |
| `ws/` | ✅ 有（WebSocket 业务特有，规则未列但 plan-1 已设计） | — |
| `api/` | ❌ 无 | 衍生自"业务模块不拆 -api/-core"决策（§1.2） |
| `event/` | ❌ 无 | P1 范围内无跨领域事件需求；未来某个 sub-plan 真要发事件时新增 `<领域>/event/` 子包，定义类和 Listener 都放那里 |

## 3. 核心组件

### 3.1 `BusinessException` + `ErrCode`（在 `common-error`）

```java
public interface ErrCode {
    int getCode();
    String getDefaultMessage();
}

public class BusinessException extends RuntimeException {
    private final ErrCode errCode;
    public BusinessException(ErrCode errCode) { ... }
    public BusinessException(ErrCode errCode, String overrideMessage) { ... }
    public ErrCode getErrCode() { return errCode; }
}
```

**全项目唯一异常类，不拆子异常**。所有业务失败统一 `throw new BusinessException(SomeErrCode)`。

### 3.2 ErrCode 区间分配

| enum | 位置 | 区间 | 本 P1 涉及码 |
|---|---|---|---|
| `CommonErrCode` | `common-error` | 400-499 | TOKEN_EXPIRED(400) / TOKEN_INVALID(401) / FORBIDDEN(403) / NOT_FOUND(404) / PARAM_INVALID(410) / INTERNAL_ERROR(500) |
| `UserErrCode` | `user.internal` | 1000-1999 | PHONE_ALREADY_REGISTERED(1001) / USER_NOT_FOUND(1002) / WRONG_PASSWORD(1003) / SMS_CODE_INVALID(1004) / SMS_CODE_EXPIRED(1005) / SMS_SEND_TOO_FREQUENT(1006) |
| `ChatErrCode` | `chat.internal` | 2000-2999 | MESSAGE_PROCESSING_FAILED(2001) |
| `CharacterErrCode` | `character.internal` | 3000-3999 | CHARACTER_NOT_FOUND(3001) |
| `LlmErrCode` | `llm.internal` | 4000-4999 | LLM_CALL_FAILED(4001) |

### 3.3 `ErrCodeConflictDetector`（启动时冲突扫描）

`common-error` 提供 `@Component implements ApplicationRunner`，启动时反射扫描所有 implements `ErrCode` 的 enum，发现 code 重复直接 `throw IllegalStateException` 让启动失败。防止两个模块申请同一个 code。

### 3.4 现有 `IllegalArgumentException` 替换映射

| 现位置 | 抛的内容 | 替换为 |
|---|---|---|
| `AuthService.login` | "用户不存在" | `BusinessException(UserErrCode.USER_NOT_FOUND)` |
| `AuthService.login` | "密码错误" | `BusinessException(UserErrCode.WRONG_PASSWORD)` |
| `AuthService.register` | "手机号已注册" | `BusinessException(UserErrCode.PHONE_ALREADY_REGISTERED)` |
| `AuthService` SMS 相关 | 验证码各种错 | `BusinessException(UserErrCode.SMS_CODE_*)` |
| `JwtUtil` | token 解析失败 | `BusinessException(CommonErrCode.TOKEN_INVALID)` |
| `MessageService` | "默认角色不存在" | `BusinessException(CharacterErrCode.CHARACTER_NOT_FOUND)` |

**`InvalidTokenException` 类删除**，归并到 `BusinessException(CommonErrCode.TOKEN_INVALID)`。

### 3.5 `BaseResp<T>`（在 `common-web`，替换现有 `ApiResponse`）

```json
{ "success": true,  "code": 0,    "message": "ok",        "data": { ... } }
{ "success": false, "code": 1003, "message": "密码错误",   "data": null }
```

字段语义按 plan-1 第 5.5 节。HTTP 状态码永远 200（除非通信层错），业务成败看 `success`。

### 3.6 `GlobalExceptionHandler`（在 `common-web`）

`@RestControllerAdvice` 统一拦截：
- `BusinessException` → `BaseResp.failed(e.getErrCode().getCode(), e.getMessage())`
- `MethodArgumentNotValidException` → `BaseResp.failed(PARAM_INVALID.getCode(), 字段错误信息)`
- `Exception` 兜底 → `BaseResp.failed(500, "服务器错误，请稍后重试")` + log.error

## 4. Flyway 接管策略

### 4.1 迁移步骤

1. **dump 现状**：`ssh new "pg_dump --schema-only --no-owner --no-privileges sanyan"` → 整理为 `V1__initial_schema.sql` 放到 `sanyan-business/src/main/resources/db/migration/`
2. **种子数据迁移**：`data.sql` 中的"插入小婉"挪到 `V2__seed_xiaowan.sql`（用 `INSERT ... WHERE NOT EXISTS` 保持幂等）
3. **关闭 data.sql**：`spring.sql.init.mode: never`
4. **配置 Flyway**：`spring.flyway.enabled: true` + `baseline-on-migrate: true` + `baseline-version: 1`
5. **统一 ddl-auto**：dev 和 prod 都改 `validate`（schema 完全由 Flyway 控制，entity 必须匹配）

### 4.2 命名 + 不可变性
- 文件名：`V{N}__{snake_case_description}.sql`，N 单调递增
- 已部署过的 `V{N}` **永不再改**；错了写 `V{N+1}` 来覆盖
- 所有 SQL 写成幂等（`IF NOT EXISTS` / `WHERE NOT EXISTS`），重跑不爆炸

### 4.3 H2 兼容性
现有测试用 H2 内存库。V1 SQL 需要兼容 H2：
- 优先用 `MODE=PostgreSQL` 让 H2 接受常见 PG 语法
- V1 dump 后**人工审查**，剔除 H2 不支持的 PG 专有语法（如 `JSONB` / `SERIAL`），改写成等价 H2 写法（项目现状表都是基础 BIGSERIAL/VARCHAR/TEXT/TIMESTAMP，预期能直接兼容）

**P1 不切 Testcontainers**——切 Testcontainers 是 P4 的范围，本 P1 范围内若 H2 兼容性挡路，处理方式是简化 V1 SQL 写法而非引入新测试基础设施。

## 5. ArchUnit 守护规则

### 5.1 位置
- 依赖：`sanyan-common-test` 模块加 ArchUnit 依赖
- 测试：`bootstrap/src/test/java/.../ArchitectureTest.java`（bootstrap 能看到所有 class）

### 5.2 4 条最小规则

```java
@AnalyzeClasses(packages = "com.sanyan")
class ArchitectureTest {

    // R1: foundation 不能反向 import business
    @ArchTest
    static final ArchRule foundation_cannot_depend_on_business =
        noClasses().that().resideInAPackage("com.sanyan.common..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "com.sanyan.user..",
                "com.sanyan.chat..",
                "com.sanyan.character..",
                "com.sanyan.llm..");

    // R2: 业务领域间不能引彼此的 web/ws 包（用 SlicesRuleDefinition 校验切片间不允许跨 web/ws）
    @ArchTest
    static final ArchRule no_cross_domain_web_dependency =
        slices().matching("com.sanyan.(user|chat|character|llm)..")
            .namingSlices("$1")
            .should().notDependOnEachOther()
            .ignoreDependency(
                JavaClass.Predicates.resideOutsideOfPackage("..web.."),
                JavaClass.Predicates.resideOutsideOfPackage("..ws.."));
    // 注：领域间 .internal 互相调用允许；上面只拦 web/ws 跨领域引用

    // R3: web/ws 层不能直接依赖 Repository（强制走 service）
    @ArchTest
    static final ArchRule web_layer_must_not_access_repositories =
        noClasses().that().resideInAnyPackage("..web..", "..ws..")
            .should().dependOnClassesThat().haveSimpleNameEndingWith("Repository");

    // R4: ErrCode enum 必须放 internal 或 common.error 包
    @ArchTest
    static final ArchRule errcode_enums_in_internal =
        classes().that().implement(ErrCode.class).and().areEnums()
            .should().resideInAnyPackage("..internal..", "com.sanyan.common.error..");
}
```

> R2 的 slices 写法是个**初稿**，实际编码 M5 时可能需要根据 ArchUnit API 细调；核心语义是"跨领域 import .web/.ws 类不允许"。

## 6. 实施路径（5 个里程碑）

| M | 名称 | 工作量 | 验收 |
|---|---|---|---|
| M1 | Maven 骨架 + Enforcer | ~半天 | `mvn validate` 全过（Enforcer 包含 banned-dependencies）；`mvn package` 能产出 bootstrap.jar |
| M2 | foundation_packages 充实 | ~1 天 | foundation 自身单测全过 |
| M3 | 业务代码搬家 + 命名/Service 规范化 + 测试搬家 | **~1-2 天**（最大改动） | 现有 194 个测试全过；部署 dev 能起、能登录、能聊天 |
| M4 | Flyway 接管 | ~半天 | 服务器启动日志显示 baseline V1 + migrate to V2；schema 不变 |
| M5 | ArchUnit 守护 | ~半天 | `mvn test` 包含 4 条 ArchUnit 测试且全过 |

**每个 M 完成后 commit + 跑全量测试**。

### 6.M3 详细子任务（按规则 `java-backend-business-layer.md` 对齐）

| 子任务 | 内容 |
|---|---|
| **M3.1** 按领域搬家 | 现有 `com.sanyan.{controller,service,entity,repository,websocket,...}` → 按 §2.3 重组到 `com.sanyan.{user,character,chat,llm}.{web,ws,internal}`。测试代码**镜像 src/main 路径**同步搬家（如 `AuthServiceTest` 跟 `AuthService` 一起到 `com.sanyan.user.internal`） |
| **M3.2** Entity 加 `Entity` 后缀 | `User → UserEntity`、`Message → MessageEntity`、`AiCharacter → AiCharacterEntity`。同时更新 Repository 泛型参数、所有引用点。**注意**：JPA `@Table(name=...)` 显式指定旧表名，避免 schema 变更 |
| **M3.3** Service 按动作拆 | `AuthService` 现有的 register + login + SMS 多个动作拆成：`UserRegisterService` / `UserLoginService` / `SmsCodeSendService`（命名 `<Domain><Action>Service`）。每个 Service 方法 = 一个 `@Transactional` 边界 |
| **M3.4** 替换异常 | 现有 `IllegalArgumentException` / `InvalidTokenException` 全部替换为 `BusinessException(具体 ErrCode)`（按 §3.4 映射表）。删除 `com.sanyan.exception.InvalidTokenException` 类 |
| **M3.5** Controller 集成测试用 `*IT` 后缀 | 用 `@WebMvcTest` / `@SpringBootTest` 启动 Spring context 的测试改名：`AuthControllerTest → AuthControllerIT`、`RepositoryTest`（含 `@DataJpaTest` 部分）→ `*IT`。父 POM 配 surefire 跑 `*Test`、failsafe 跑 `*IT` |
| **M3.6** 引入 `<Domain>TestFixtures` | 给每个 Entity 写一个 fixture 类（位置：`<domain>/internal/`，包路径在测试目录下镜像）：`UserTestFixtures` / `MessageTestFixtures` / `AiCharacterTestFixtures`。最低提供"默认有效对象"方法 `validXxx()`。现有测试里所有裸 `new UserEntity()` 等 entity 构造改用 fixture |
| **M3.7** `ApiResponse → BaseResp` 改名 | 现有 `com.sanyan.dto.ApiResponse` 删，由 `common-web` 的 `BaseResp` 替换。所有 Controller 返回类型同步更新 |

**M3 整体验收**：现有 194 测试全过、`mvn compile` + `mvn test` 双过、能跑通端到端 dogfood 路径。

## 7. 测试策略

- 现有测试文件**跟着代码搬家**到对应包路径
- M3 完成后测试总数仍是 194（搬家不增不减）
- M5 完成后测试总数 = 194 + 4 ArchUnit = 198
- H2 优先；H2 兼容性挡路时人工简化 V1 SQL（不引入 Testcontainers，那是 P4）

## 8. 风险 + 缓解

| ID | 风险 | 缓解 |
|---|---|---|
| R1 | Flyway baseline 失败（V1 与生产现状字段顺序/约束不一致） | pg_dump 用 `--schema-only --no-owner --no-privileges`；本机临时建 sanyan PG 库重放验证；部署前 `diff <(pg_dump) V1.sql` |
| R2 | 搬家漏改 import 编译失败 | 每搬一个领域跑 `mvn compile`；IDE 用 "Move package" 重构 |
| R3 | H2 跑 V1 失败（PG 专有语法） | V1 dump 时审查；H2 用 `MODE=PostgreSQL`；最坏切 Testcontainers |
| R4 | ArchUnit 暴露存量违规 | M5 先诊断模式打印违规、修干净再 enforce |
| R5 | 长开发期影响 dogfood | 在 `feat/p1-architecture` 分支独立做，master/dev 不动；M5 + dogfood 通过后整组 merge |

## 9. 分支策略

- 在 server 子模块开 `feat/p1-architecture` 分支
- M1~M5 逐步 commit 到该分支
- M5 全过 + dogfood 一遍 → merge dev（`--no-ff`） → 部署测试 → 没问题 → merge master（`--no-ff`）
- app 分支不动（前端在 P3 / P5 才动）
- 主仓库 3yan：在 server 子模块合并完后再更新子模块引用 commit

## 10. 验收标准

**架构层**
- [ ] 10 个 Maven 子模块拓扑搭起来；`mvn validate` 通过 Enforcer 检查（含 banned-dependencies）
- [ ] foundation_packages 8 个模块全部按规则 `java-backend-foundation-layer.md` 命名（`sanyan-common-<role>`）

**命名规范**
- [ ] 所有 Entity 类带 `Entity` 后缀（`UserEntity` / `MessageEntity` / `AiCharacterEntity`）
- [ ] Service 按动作拆（`UserRegisterService` / `UserLoginService` / `SmsCodeSendService`）；不再有 `AuthService` 这种多动作聚合类
- [ ] Controller 集成测试用 `*IT` 后缀；纯 Mockito 单测用 `*Test` 后缀

**异常 + 响应**
- [ ] 所有现有 `IllegalArgumentException` / `InvalidTokenException` 替换为 `BusinessException` + 对应 `ErrCode`
- [ ] `BaseResp` 替换 `ApiResponse`；`GlobalExceptionHandler` 工作正常
- [ ] `ErrCodeConflictDetector` 在启动时跑过、无冲突

**测试基础设施**
- [ ] 至少 3 个 `<Domain>TestFixtures` 类（User / Message / AiCharacter）建立；现有测试里裸 `new XxxEntity()` 已改用 fixture
- [ ] 测试代码路径镜像 src/main（测试和被测代码在同一包路径下）

**Flyway**
- [ ] Flyway 在本机 + 生产都成功 baseline V1 + 跑 V2
- [ ] schema 不变（前后 `pg_dump --schema-only` diff 为空）

**ArchUnit**
- [ ] 4 条 ArchUnit 规则全过
- [ ] 现有 194 个测试全部通过；新增 ArchUnit 测试通过（约 198 总数）

**端到端**
- [ ] dogfood 一遍：注册 → 登录 → 聊天 → 收到 AI 回复，端到端正常
- [ ] 部署到 new 服务器，能正常服务

## 11. 范围外（后续 sub-plan）
- P2：消息状态枚举完整化（pending/delivered/read/failed）
- P3：前端 `sanyan_util` / `sanyan_bizkit` 包补全
- P4：端到端集成测试（Testcontainers）
- P5：前端 `sanyan_chat` 包测试
