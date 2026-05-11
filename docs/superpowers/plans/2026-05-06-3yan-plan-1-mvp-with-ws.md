# 三言 · Plan 1 实施计划：端到端 MVP（含 WebSocket + 真人节奏）

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` to execute this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **TDD 铁律（来自 `~/.claude/CLAUDE.md`）：每个 task 严格按 Red → Green → Refactor 5 步执行：① 写失败测试 → ② 跑测试看到失败 → ③ 写最小实现 → ④ 跑测试通过 → ⑤ commit。没有失败测试，不允许写任何生产代码。**
>
> **本 plan 配套规则文件（路径触发自动加载）：**
> - 后端：`~/.claude/rules/java-backend.md` + `java-backend-foundation-layer.md` + `java-backend-business-layer.md`
> - 前端：`~/.claude/rules/flutter.md` + `flutter-foundation-layer.md` + `flutter-business-layer.md`

**Goal：** 跑通"端到端能聊天 + 不出戏"的 MVP，含 WebSocket 双向通信完整基础设施 + 真人节奏模拟。完成后用户能注册→登录→和林小满聊天，体验上消除 AI 流式感。

**Architecture：**
- 后端 Java 17 + Spring Boot 3.x + Maven 多模块（`bootstrap` + `business_packages/` 4 个 `-api`/`-core` 双模块 + `foundation_packages/` 7 个 `common-*` 包）
- 前端 Flutter + GetX + GetStorage（`lib/` + `business_packages/` 2 个业务包 + `foundation_packages/` 6 个固定基础包）
- 数据库 PostgreSQL 15 + Flyway 迁移；缓存 Redis 7
- LLM 接入阿里云百炼 `qwen-plus-character`（主链路），暂不接 DeepSeek（Plan 2 时再加）
- WS 完整基础设施（心跳 30s + 断线指数退避重连 + ACK + 消息落库 + Redis 单实例连接映射）
- 单实例部署，多实例 Pub/Sub 路由不在 Plan 1（架构层预留接口）

**Tech Stack：**
- 后端：Java 17, Spring Boot 3.3.x, Maven, Spring Data JPA, Flyway, jjwt, spring-security-crypto (BCrypt), spring-boot-starter-websocket, OkHttp 4.x（调 LLM）, Mockito, Testcontainers, ArchUnit
- 前端：Flutter 3.x, get, get_storage, dio, web_socket_channel, mocktail, flutter_test, integration_test

---

## 文件结构

### 后端（3yan-server/）

```
3yan-server/
├── pom.xml                                       # 父 POM（聚合 + 依赖管理 + Enforcer）
├── scripts/
│   ├── dev-up.sh                                 # 本地开发环境启动（brew services 幂等）
│   └── dev-down.sh                               # 本地开发环境停止（保留数据）
├── DEVELOPMENT.md                                # 开发指南（brew install / 凭据 / 故障排查）
├── README.md
│
├── bootstrap/                                    # 启动工程
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/3yan/Application.java
│       └── resources/
│           ├── application.yml                   # 公共配置
│           ├── application-local.yml             # 本地 profile
│           ├── application-prod.yml              # 生产 profile（git 忽略真实 key）
│           └── db/migration/                     # Flyway 迁移脚本
│               ├── V1__init_users.sql
│               ├── V2__init_characters_relationships.sql
│               ├── V3__init_messages.sql
│               └── V4__seed_sanyan_xiaoman.sql
│
├── foundation_packages/
│   ├── sanyan-common-error/
│   │   └── src/main/java/com/3yan/common/error/
│   │       ├── BusinessException.java
│   │       ├── ErrCode.java
│   │       ├── CommonErrCode.java
│   │       ├── ErrCodeConflictDetector.java
│   │       └── ERROR_CODE_REGISTRY.md
│   ├── sanyan-common-web/
│   │   └── src/main/java/com/3yan/common/web/
│   │       ├── BaseResp.java
│   │       └── GlobalExceptionHandler.java
│   ├── sanyan-common-auth/
│   │   └── src/main/java/com/3yan/common/auth/
│   │       ├── JwtProvider.java
│   │       ├── JwtAuthFilter.java
│   │       ├── CurrentUser.java
│   │       ├── LoginRequired.java                # @LoginRequired 注解
│   │       └── PasswordEncoderConfig.java        # BCryptPasswordEncoder Bean
│   ├── sanyan-common-cache/
│   │   └── src/main/java/com/3yan/common/cache/
│   │       ├── KvCache.java
│   │       └── RedisConfig.java
│   ├── sanyan-common-storage/                     # 一期空骨架，未来接入 COS
│   │   └── src/main/java/com/3yan/common/storage/
│   │       └── ObjectStorage.java                # 接口（一期只定义不实现）
│   ├── sanyan-common-util/
│   │   └── src/main/java/com/3yan/common/util/
│   │       ├── PhoneUtil.java
│   │       └── JsonUtil.java
│   ├── sanyan-common-test/
│   │   └── src/main/java/com/3yan/common/test/
│   │       ├── BaseIntegrationTest.java
│   │       └── ArchitectureTestBase.java         # ArchUnit 基类
│   └── sanyan-common-ws/
│       └── src/main/java/com/3yan/common/ws/
│           ├── WsConnectionRegistry.java         # session ↔ userId 映射
│           ├── WsPushService.java                # pushTo(userId, msg)
│           └── WsConfig.java
│
└── business_packages/
    ├── sanyan-user-api/                           # 用户领域契约
    │   └── src/main/java/com/3yan/user/
    │       ├── UserApi.java
    │       ├── dto/UserDto.java
    │       └── event/UserRegisteredEvent.java
    ├── sanyan-user-core/
    │   └── src/main/java/com/3yan/user/
    │       ├── web/
    │       │   ├── AuthController.java
    │       │   ├── SendSmsCodeReq.java          # +SendSmsCodeData
    │       │   ├── RegisterReq.java             # +RegisterData
    │       │   └── LoginReq.java                # +LoginData
    │       ├── internal/
    │       │   ├── UserEntity.java
    │       │   ├── UserRepository.java
    │       │   ├── UserRegisterService.java
    │       │   ├── UserLoginService.java
    │       │   ├── SmsCodeService.java          # 一期 mock 实现
    │       │   └── UserErrCode.java             # 1000-1999
    │       └── api/UserApiImpl.java
    ├── sanyan-character-api/
    │   └── src/main/java/com/3yan/character/
    │       ├── CharacterApi.java
    │       └── dto/CharacterDto.java
    ├── sanyan-character-core/
    │   └── src/main/java/com/3yan/character/
    │       ├── internal/
    │       │   ├── CharacterEntity.java
    │       │   ├── CharacterRepository.java
    │       │   ├── RelationshipEntity.java
    │       │   ├── RelationshipRepository.java
    │       │   ├── CharacterErrCode.java        # 2000-2999
    │       │   └── RelationshipBootstrapService.java
    │       ├── api/CharacterApiImpl.java
    │       └── event/UserRegisteredListener.java # 监听 UserRegisteredEvent，初始化关系
    ├── sanyan-llm-api/
    │   └── src/main/java/com/3yan/llm/
    │       ├── LLMProvider.java
    │       └── dto/{ChatMessage,ChatRequest,ChatResponse,LLMTaskType}.java
    ├── sanyan-llm-core/
    │   └── src/main/java/com/3yan/llm/
    │       ├── internal/
    │       │   ├── QwenAdapter.java
    │       │   ├── LLMProviderFactory.java
    │       │   ├── LLMConfig.java
    │       │   └── LLMErrCode.java              # 3000-3999
    │       └── api/LLMProviderRouter.java       # 对外暴露按 LLMTaskType 选择适配器
    ├── sanyan-chat-api/
    │   └── src/main/java/com/3yan/chat/
    │       ├── ChatApi.java
    │       └── dto/MessageDto.java
    └── sanyan-chat-core/
        └── src/main/java/com/3yan/chat/
            ├── web/
            │   ├── MessageController.java       # /api/messages/sync
            │   └── SyncMessagesReq.java
            ├── internal/
            │   ├── MessageEntity.java
            │   ├── MessageRepository.java
            │   ├── MessageStatus.java           # pending/delivered/push_sent/read/failed
            │   ├── MessageRole.java             # USER/ASSISTANT/SYSTEM
            │   ├── ChatService.java             # 构造 prompt + 调 LLM + 拆分多条 + 模拟延迟
            │   ├── ContentSafetyFilter.java     # 关键词黑名单
            │   ├── PromptBuilder.java           # 林小满 base_prompt + 短期上下文
            │   ├── TypingDelayCalculator.java   # 模拟打字时长公式
            │   └── ChatErrCode.java             # 4000-4999
            ├── ws/
            │   ├── ChatWebSocketHandler.java
            │   └── ChatWsMessage.java           # WS 帧 JSON 协议
            ├── api/ChatApiImpl.java
            └── event/                           # （Plan 1 暂无 listener）
```

### 前端（3yan-app/）

```
3yan-app/
├── pubspec.yaml                                  # 顶层壳工程
├── android/...
├── ios/...
├── lib/
│   ├── main.dart                                 # 启动：await GetStorage.init() → runApp
│   └── home_page.dart                            # 集成各业务模块（一期只 chat）
│
├── foundation_packages/
│   ├── sanyan_uikit/
│   │   ├── pubspec.yaml
│   │   └── lib/
│   │       ├── sanyan_uikit.dart
│   │       └── src/
│   │           ├── theme.dart
│   │           ├── colors.dart
│   │           ├── primary_button.dart
│   │           ├── input_field.dart
│   │           └── toast.dart
│   ├── sanyan_network/
│   │   └── lib/src/{http_client,base_req,base_resp}.dart
│   ├── sanyan_routes/
│   │   └── lib/sanyan_routes.dart
│   ├── sanyan_user/
│   │   └── lib/src/
│   │       ├── api/{sanyan_user_api,req/login_req,req/refresh_token_req}.dart
│   │       ├── models/user.dart
│   │       ├── local/user_cache.dart
│   │       └── user_center.dart
│   ├── sanyan_bizkit/                             # 一期空壳，后续填充
│   │   └── lib/{sanyan_bizkit,src}/
│   └── sanyan_util/
│       └── lib/src/{phone_util,date_util}.dart
│
└── business_packages/
    ├── sanyan_auth/
    │   └── lib/
    │       ├── sanyan_auth.dart                   # barrel
    │       ├── sanyan_auth_routes.dart
    │       └── src/
    │           ├── api/
    │           │   ├── sanyan_auth_api.dart
    │           │   └── req/{send_sms_code_req,register_req,login_req}.dart
    │           ├── register/
    │           │   ├── register_page.dart
    │           │   └── register_controller.dart
    │           ├── login/
    │           │   ├── login_page.dart
    │           │   └── login_controller.dart
    │           └── widgets/sms_code_button.dart  # 倒计时按钮
    ├── sanyan_chat/
    │   └── lib/
    │       ├── sanyan_chat.dart
    │       ├── sanyan_chat_routes.dart
    │       └── src/
    │           ├── api/
    │           │   ├── sanyan_chat_api.dart
    │           │   ├── req/sync_messages_req.dart
    │           │   └── models/{message,chat_ws_event}.dart
    │           ├── chat/
    │           │   ├── chat_page.dart
    │           │   ├── chat_controller.dart
    │           │   └── widgets/{message_bubble,typing_indicator,input_bar}.dart
    │           └── ws/
    │               ├── chat_ws_service.dart
    │               └── typing_delay_simulator.dart
```

---

## Phase A · Maven 父工程与开发环境（后端）

### Task A1：父 POM + Enforcer + dependencyManagement

**Files:**
- Create: `3yan-server/pom.xml`

**Steps:**
- [ ] **A1.1**：创建父 POM，packaging=pom，groupId=`com.sanyan`，artifactId=`3yan-server`，version=`0.1.0-SNAPSHOT`
- [ ] **A1.2**：在 `<modules>` 列出所有子模块（bootstrap + 8 个 foundation: error/web/auth/cache/storage/util/test/ws + 4 个业务领域各 `-api`+`-core`：user/character/llm/chat）
- [ ] **A1.3**：`<dependencyManagement>` 集中管理版本：spring-boot-starter-parent 3.3.x、jjwt 0.12.x、testcontainers、archunit、mocktail（前端无关）等
- [ ] **A1.4**：配置 `maven-enforcer-plugin` `banned-dependencies` 规则禁止 `business_packages/*-core` 互依（按 `java-backend.md` §4 第一层）
- [ ] **A1.5**：配置 `maven-surefire-plugin`（`*Test`）+ `maven-failsafe-plugin`（`*IT`）按 `java-backend.md` §7.2
- [ ] **A1.6**：`mvn validate` 通过，commit `chore: maven 父 POM + 模块聚合 + enforcer`

### Task A2：本地开发环境（brew 直装，**修订版**）

> **注：** 原 spec 是 docker-compose，因用户偏好不装 Docker Desktop（4GB+ 常驻内存）改为本机 brew 直装。

**Files:**
- Create: `3yan-server/scripts/dev-up.sh`、`3yan-server/scripts/dev-down.sh`
- Create: `3yan-server/DEVELOPMENT.md`

**Steps:**
- [ ] **A2.1**：实跑 `brew install postgresql@15 pgvector redis` 装基础服务；如 `brew install pgvector` bottle 不兼容 PG15（截至 2026-05 实测 0.8.2 bottle 仅支持 PG17/18），从源码编译 pgvector v0.8.0：`git clone -b v0.8.0 https://github.com/pgvector/pgvector.git /tmp/pgvector_src && cd /tmp/pgvector_src && make PG_CONFIG=$(brew --prefix postgresql@15)/bin/pg_config && make install PG_CONFIG=$(brew --prefix postgresql@15)/bin/pg_config`
- [ ] **A2.2**：写 `scripts/dev-up.sh` 幂等启动脚本：检查 brew formulas 已装 → 检查端口 5432/6379 不被非项目进程占用 → `brew services start postgresql@15 redis` → 创建 `3yan` 用户/库 → `CREATE EXTENSION IF NOT EXISTS vector` → 健康检查（psql + redis-cli ping）。所有路径用 `$(brew --prefix)` 命令替换避免硬编码（兼容 Apple Silicon + Intel Mac）。`chmod +x`
- [ ] **A2.3**：写 `scripts/dev-down.sh` 简洁停止脚本：`brew services stop postgresql@15 2>/dev/null || true` 和 `brew services stop redis 2>/dev/null || true`（防御 brew 未来行为变更）。`chmod +x`
- [ ] **A2.4**：写 `DEVELOPMENT.md`：一次性环境准备 / 日常启停命令 / 凭据连接串 / 故障排查（重点：**任何 `brew upgrade postgresql@15` 后必须重新执行源码编译步骤**，否则 vector.so 丢失）
- [ ] **A2.5**：实跑 `./scripts/dev-up.sh` 两次验证幂等；实跑 `./scripts/dev-down.sh` 验证停止
- [ ] **A2.6**：commit `feat(dev): 本地开发环境（brew postgresql@15+pgvector+redis）+ 启停脚本 + DEVELOPMENT.md`

### Task A3：bootstrap 启动工程

**Files:**
- Create: `3yan-server/bootstrap/pom.xml`
- Create: `3yan-server/bootstrap/src/main/java/com/3yan/Application.java`
- Create: `3yan-server/bootstrap/src/main/resources/application.yml`
- Create: `3yan-server/bootstrap/src/main/resources/application-local.yml`
- Create: `3yan-server/bootstrap/src/main/resources/application-prod.yml`

**Steps:**
- [ ] **A3.1**：bootstrap pom 依赖所有 foundation 包 + 所有 business `-core` 包 + spring-boot-starter-web/data-jpa/data-redis/websocket/validation
- [ ] **A3.2**：`Application.java` 用 `@SpringBootApplication`，main 启动；指定扫描包 `com.sanyan`
- [ ] **A3.3**：`application.yml` 默认 `spring.profiles.active=local`、`server.port=8080`、Flyway 配置 `baseline-on-migrate=true`、`locations=classpath:db/migration`
- [ ] **A3.4**：`application-local.yml` 配 jdbc 到本机 postgres（`postgresql://3yan:sanyan_dev@localhost:5432/3yan`）+ redis（`redis://localhost:6379`）；JPA `ddl-auto=validate`、`show-sql=true`；JWT secret 走环境变量 `${JWT_SECRET:dev-only-not-for-prod}`；LLM API key 走 `${QWEN_API_KEY:}`（空时跳过 LLM 真实调用，测试用 mock）
- [ ] **A3.5**：`application-prod.yml` 全部走环境变量；`ddl-auto=validate`
- [ ] **A3.6**：`mvn -pl bootstrap spring-boot:run` 启动成功（暂时还没 Flyway 脚本，启动会失败 —— 等 Phase B Flyway 后再验证）
- [ ] **A3.7**：commit `feat(bootstrap): 启动工程 + 多 profile 配置`

---

## Phase B · 基础包 foundation_packages（按 java-backend-foundation-layer.md）

### Task B1：sanyan-common-error

**Files:**
- Create: `foundation_packages/sanyan-common-error/pom.xml`
- Create: `.../com/3yan/common/error/{BusinessException,ErrCode,CommonErrCode,ErrCodeConflictDetector}.java`
- Create: `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`
- Test: `.../test/com/3yan/common/error/ErrCodeConflictDetectorTest.java`

**Steps:**
- [ ] **B1.1**：写 `BusinessException` `ErrCode` `CommonErrCode`（按 `java-backend.md` §5.1-5.2 提供的代码模板）
- [ ] **B1.2**：写 `ErrCodeConflictDetectorTest`（Mockito 单测）—— 构造两个 `ErrCode` enum 包含冲突 code，断言 `run()` 抛 `IllegalStateException`，再用合规 enum 断言不抛
- [ ] **B1.3**：跑测试看到失败（class 不存在）
- [ ] **B1.4**：写 `ErrCodeConflictDetector` 实现（按 `java-backend.md` §5.4），用 `Reflections` 库或 `ApplicationContext.getBeansOfType(ErrCode.class.getEnclosingClass()...)` 扫描 enum
- [ ] **B1.5**：测试通过
- [ ] **B1.6**：维护 `ERROR_CODE_REGISTRY.md`，登记 400-499 通用、1000-1999 user、2000-2999 character、3000-3999 llm、4000-4999 chat
- [ ] **B1.7**：commit `feat(common-error): BusinessException + ErrCode + 启动期冲突检测`

### Task B2：sanyan-common-web

**Files:**
- Create: `foundation_packages/sanyan-common-web/pom.xml`（依赖 `sanyan-common-error`）
- Create: `.../com/3yan/common/web/{BaseResp,GlobalExceptionHandler}.java`
- Test: `.../test/com/3yan/common/web/{BaseRespTest,GlobalExceptionHandlerTest}.java`

**Steps:**
- [ ] **B2.1**：写 `BaseRespTest`：`success()` 返回 success=true、code=0；`failed(int,String)` 返回 success=false
- [ ] **B2.2**：跑测试 fail
- [ ] **B2.3**：实现 `BaseResp<T>`（按 `java-backend.md` §5.5 模板）
- [ ] **B2.4**：写 `GlobalExceptionHandlerTest`：mock 一个 `BusinessException`，断言返回 BaseResp.failed(code, message)；mock `MethodArgumentNotValidException`、未知 `Exception`，断言返回对应 BaseResp
- [ ] **B2.5**：跑测试 fail
- [ ] **B2.6**：实现 `GlobalExceptionHandler`（按 §5.6 模板）
- [ ] **B2.7**：commit `feat(common-web): BaseResp + GlobalExceptionHandler`

### Task B3：sanyan-common-auth

**Files:**
- Create: `foundation_packages/sanyan-common-auth/pom.xml`（依赖 jjwt-api/impl/jackson、spring-security-crypto、sanyan-common-error、sanyan-common-cache）
- Create: `.../com/3yan/common/auth/{JwtProvider,JwtAuthFilter,CurrentUser,LoginRequired,PasswordEncoderConfig}.java`
- Test: `.../test/com/3yan/common/auth/{JwtProviderTest,JwtAuthFilterTest}.java`

**Steps:**
- [ ] **B3.1**：先写 `JwtProviderTest`：`generateAccessToken(userId)` 返回非空字符串；`parseUserId(token)` 返回原 userId；过期 token 抛 `BusinessException(CommonErrCode.TOKEN_EXPIRED)`；篡改 token 抛 `BusinessException(CommonErrCode.TOKEN_EXPIRED)`
- [ ] **B3.2**：跑测试 fail
- [ ] **B3.3**：实现 `JwtProvider`：access token 7 天、refresh token 30 天；从 `application.yml` 读 secret；HS256 签名
- [ ] **B3.4**：写 `JwtAuthFilterTest`：mock HttpRequest 带合法 token → filter 后 `CurrentUser.id()` 返回 userId；带过期 token → ResponseEntity 返回 BaseResp.failed(400)；无 token → 跳过（不强制登录，由 `@LoginRequired` 注解决定）
- [ ] **B3.5**：跑测试 fail
- [ ] **B3.6**：实现 `CurrentUser`（ThreadLocal）+ `JwtAuthFilter`（Spring `OncePerRequestFilter`）+ `@LoginRequired` 注解 + 一个 AOP 切面强制登录检查
- [ ] **B3.7**：实现 `PasswordEncoderConfig` Bean（BCryptPasswordEncoder strength=10）
- [ ] **B3.8**：commit `feat(common-auth): JWT + 拦截器 + CurrentUser + @LoginRequired + BCrypt`

### Task B4：sanyan-common-cache

**Files:**
- Create: `foundation_packages/sanyan-common-cache/pom.xml`（依赖 spring-boot-starter-data-redis）
- Create: `.../com/3yan/common/cache/{KvCache,RedisConfig}.java`
- Test: `.../test/com/3yan/common/cache/KvCacheIT.java`（Testcontainers Redis）

**Steps:**
- [ ] **B4.1**：写 `KvCacheIT` 用 Testcontainers redis：`set(key, value, ttl)` 后 `get(key)` 返回原 value；过期后 get 返回 null；`delete(key)` 后 get 返回 null
- [ ] **B4.2**：跑测试 fail
- [ ] **B4.3**：实现 `RedisConfig`（`StringRedisTemplate` Bean、key 序列化用 String、value 序列化用 Jackson JSON）+ `KvCache` 封装（不暴露 RedisTemplate）
- [ ] **B4.4**：commit `feat(common-cache): KvCache + RedisConfig`

### Task B4b：sanyan-common-storage（一期空骨架）

**Files:**
- Create: `foundation_packages/sanyan-common-storage/pom.xml`
- Create: `.../com/3yan/common/storage/ObjectStorage.java`

**Steps:**
- [ ] **B4b.1**：定义 `ObjectStorage` 接口：`String upload(String key, InputStream data, String contentType, long size)`、`String presignedUrl(String key, Duration ttl)`、`void delete(String key)`、`InputStream download(String key)`
- [ ] **B4b.2**：本期**不实现具体子类**（按 `java-backend-foundation-layer.md` §3.5"前 7 个默认全要"建立骨架，COS 接入留到上线前/Plan 4）；pom 不引入腾讯云 SDK 依赖避免污染依赖图
- [ ] **B4b.3**：commit `feat(common-storage): ObjectStorage 接口骨架`

### Task B5：sanyan-common-util

**Files:**
- Create: `foundation_packages/sanyan-common-util/pom.xml`
- Create: `.../com/3yan/common/util/{PhoneUtil,JsonUtil}.java`
- Test: `.../test/com/3yan/common/util/{PhoneUtilTest,JsonUtilTest}.java`

**Steps:**
- [ ] **B5.1**：`PhoneUtilTest`：`isValid("13800138000")` true；`isValid("12345")` false（不是 11 位 1 开头）；非数字 false
- [ ] **B5.2**：`JsonUtilTest`：`toJson(map)` 返回字符串可 `fromJson` 还原
- [ ] **B5.3**：跑测试 fail → 实现 → 测试通过
- [ ] **B5.4**：commit `feat(common-util): PhoneUtil + JsonUtil`

### Task B6：sanyan-common-test

**Files:**
- Create: `foundation_packages/sanyan-common-test/pom.xml`（test scope 依赖 spring-boot-starter-test、archunit、testcontainers）
- Create: `.../com/3yan/common/test/{BaseIntegrationTest,ArchitectureTestBase}.java`

**Steps:**
- [ ] **B6.1**：`BaseIntegrationTest`：抽象类，标 `@SpringBootTest(webEnvironment=RANDOM_PORT)` + `@ActiveProfiles("test")` + `@AutoConfigureMockMvc`；提供 Testcontainers 的 PostgreSQL/Redis 静态容器（`@Container static`，所有子测试共享）
- [ ] **B6.2**：`ArchitectureTestBase`：抽象类，提供 ArchUnit 规则方法 `business_core_modules_must_not_depend_on_each_other()` —— 按 `java-backend.md` §4 第二层
- [ ] **B6.3**：在 `bootstrap` 模块下放一个 `ArchitectureIT` 继承 `ArchitectureTestBase` 实例化测试运行
- [ ] **B6.4**：commit `feat(common-test): BaseIntegrationTest + ArchUnit 守护`

### Task B7：sanyan-common-ws

**Files:**
- Create: `foundation_packages/sanyan-common-ws/pom.xml`（依赖 spring-boot-starter-websocket、sanyan-common-cache、sanyan-common-error）
- Create: `.../com/3yan/common/ws/{WsConnectionRegistry,WsPushService,WsConfig}.java`
- Test: `.../test/com/3yan/common/ws/WsConnectionRegistryTest.java`

**Steps:**
- [ ] **B7.1**：`WsConnectionRegistryTest`：`register(userId, sessionId, instanceId)` 后 `findInstanceByUserId(userId)` 返回 instanceId；`unregister(userId, sessionId)` 后返回 null；同一 user 在新 session register 时旧 session 标记为 stale
- [ ] **B7.2**：跑 fail → 实现 `WsConnectionRegistry`（用 `KvCache` 存 `ws:user:<userId>` → `instanceId:sessionId`，TTL 60s 心跳续期）
- [ ] **B7.3**：`WsPushService` 提供 `pushTo(userId, message)`：查 registry → 如本实例就 `WebSocketSession.sendMessage`，否则发 Redis Pub/Sub 让对应实例处理（Plan 1 单实例可暂时只走本实例分支，但接口预留多实例）
- [ ] **B7.4**：`WsConfig` 提供 `WebSocketContainer.setMaxSessionIdleTimeout(60000L)` 配置
- [ ] **B7.5**：commit `feat(common-ws): 连接注册 + 推送 + 心跳超时`

---

## Phase C · 用户领域（business_packages/sanyan-user-{api,core}）

### Task C1：sanyan-user-api

**Files:**
- Create: `business_packages/sanyan-user-api/pom.xml`（**不依赖任何其他模块**）
- Create: `.../com/3yan/user/UserApi.java` `dto/UserDto.java` `event/UserRegisteredEvent.java`

**Steps:**
- [ ] **C1.1**：`UserApi` 接口：`UserDto findById(Long userId)`、`boolean existsByPhone(String phone)`
- [ ] **C1.2**：`UserDto` record(Long id, String phone, String nickname, String avatarUrl)
- [ ] **C1.3**：`UserRegisteredEvent` record(Long userId, String phone, Instant registeredAt)
- [ ] **C1.4**：commit `feat(user-api): UserApi + UserDto + UserRegisteredEvent`

### Task C2：数据库迁移 V1（users 表）

**Files:**
- Create: `bootstrap/src/main/resources/db/migration/V1__init_users.sql`

**Steps:**
- [ ] **C2.1**：写 SQL：`users` 表（id BIGINT PK auto / phone VARCHAR(20) UNIQUE NOT NULL / password_hash VARCHAR(120) NULL / email VARCHAR(120) NULL UNIQUE / nickname VARCHAR(60) / avatar_url VARCHAR(500) / subscription_status SMALLINT DEFAULT 0 / subscription_expires_at TIMESTAMPTZ NULL / daily_quota_used INT DEFAULT 0 / created_at TIMESTAMPTZ NOT NULL / updated_at TIMESTAMPTZ）+ phone unique index
- [ ] **C2.2**：`mvn -pl bootstrap spring-boot:run` 启动 Flyway 自动迁移，psql 连上验证表存在
- [ ] **C2.3**：commit `feat(db): V1 users 表`

### Task C3：UserEntity + UserRepository

**Files:**
- Create: `business_packages/sanyan-user-core/pom.xml`（依赖 sanyan-user-api、sanyan-character-api、sanyan-common-error/web/auth/cache/util、spring-boot-starter-data-jpa）
- Create: `.../user/internal/{UserEntity,UserRepository}.java`
- Test: `.../test/user/internal/UserRepositoryIT.java`（@DataJpaTest）
- Create: `.../test/user/fixtures/UserTestFixtures.java`

**Steps:**
- [ ] **C3.1**：`UserTestFixtures.validUser()` 返回带 phone="13800138000" 等默认字段的 entity（按 `java-backend-business-layer.md` §5.2 模板）
- [ ] **C3.2**：写 `UserRepositoryIT`（@DataJpaTest）：保存 → existsByPhone 返回 true；不同 phone existsByPhone 返回 false；保存重复 phone 抛异常
- [ ] **C3.3**：跑 fail → 实现 `UserEntity`（@Entity, @Table("users")，按 V1 schema 字段一一对应）+ `UserRepository extends JpaRepository<UserEntity, Long>` 声明 `boolean existsByPhone(String phone)` + `Optional<UserEntity> findByPhone(String phone)`
- [ ] **C3.4**：commit `feat(user-core): UserEntity + UserRepository + Fixtures`

### Task C4：UserErrCode

**Files:**
- Create: `.../user/internal/UserErrCode.java`

**Steps:**
- [ ] **C4.1**：定义 enum：`PHONE_ALREADY_REGISTERED(1001, "手机号已注册")`、`PHONE_NOT_REGISTERED(1002, "手机号未注册")`、`SMS_CODE_INVALID(1003, "验证码错误")`、`SMS_CODE_EXPIRED(1004, "验证码过期")`、`PASSWORD_INCORRECT(1005, "密码错误")`、`SMS_TOO_FREQUENT(1006, "发送验证码过于频繁")`
- [ ] **C4.2**：更新 `ERROR_CODE_REGISTRY.md` 加 1000-1999 区间
- [ ] **C4.3**：commit `feat(user-core): UserErrCode`

### Task C5：SmsCodeService（mock 实现）

**Files:**
- Create: `.../user/internal/SmsCodeService.java`
- Test: `.../test/user/internal/SmsCodeServiceTest.java`

**Steps:**
- [ ] **C5.1**：测试：`sendCode(phone)` 后 60s 内再发抛 `BusinessException(SMS_TOO_FREQUENT)`；`verifyCode(phone, code)` 在 5 分钟有效期内 code 正确返回 true，错误返回 false，过期抛 `SMS_CODE_EXPIRED`；mock 实现 `sendCode` 把 6 位数字写入 `KvCache` 的 `sms:code:<phone>` key，TTL 5 分钟，并打印 log（一期不真发短信，开发自测从日志拿验证码）
- [ ] **C5.2**：跑 fail → 实现（生成 6 位随机数字、用 KvCache 存 / 验证 / 删除、TTL 配置）
- [ ] **C5.3**：commit `feat(user-core): SmsCodeService（mock 实现，本地从日志拿验证码）`

### Task C6：UserRegisterService

**Files:**
- Create: `.../user/internal/UserRegisterService.java`
- Test: `.../test/user/internal/UserRegisterServiceTest.java`

**Steps:**
- [ ] **C6.1**：测试（Mockito 单测，按 `java-backend.md` §9.4 模板）：
  - phone 已存在 → 抛 `BusinessException(PHONE_ALREADY_REGISTERED)`，未保存，未发事件
  - 验证码错 → 抛 `SMS_CODE_INVALID`
  - 成功路径 → 保存 entity，密码哈希 != 原密码（如有），发布 `UserRegisteredEvent`
- [ ] **C6.2**：跑 fail → 实现：方法签名 `register(String phone, String smsCode, String password, String nickname)`：先 `smsCodeService.verifyCode` → 错则抛；再 `repo.existsByPhone` → 重复抛；再 hash 密码（password 可为空，空时存 null，用户后续设密码）；保存 entity；`events.publishEvent(new UserRegisteredEvent(...))`；返回 userId
- [ ] **C6.3**：commit `feat(user-core): UserRegisterService（验证码 + 哈希 + 事件）`

### Task C7：UserLoginService

**Files:**
- Create: `.../user/internal/UserLoginService.java`
- Test: `.../test/user/internal/UserLoginServiceTest.java`

**Steps:**
- [ ] **C7.1**：测试：
  - `loginByVerifyCode(phone, code)`：phone 不存在抛 `PHONE_NOT_REGISTERED`；code 错抛 `SMS_CODE_INVALID`；成功返回 `LoginResult(userId, accessToken, refreshToken)`
  - `loginByPassword(phone, password)`：phone 不存在抛同上；password 与哈希不匹配抛 `PASSWORD_INCORRECT`；用户从未设密码（password_hash=null）抛 `PASSWORD_INCORRECT`；成功返回 LoginResult
- [ ] **C7.2**：跑 fail → 实现：注入 `UserRepository`、`SmsCodeService`、`BCryptPasswordEncoder`、`JwtProvider`；返回 `record LoginResult(Long userId, String accessToken, String refreshToken)`
- [ ] **C7.3**：commit `feat(user-core): UserLoginService（验证码 + 密码双登录路径）`

### Task C8：AuthController + 集成测试

**Files:**
- Create: `.../user/web/{AuthController,SendSmsCodeReq,RegisterReq,LoginReq}.java`
- Test: `.../test/user/web/AuthControllerIT.java`（@WebMvcTest）

**Steps:**
- [ ] **C8.1**：写 `AuthControllerIT`（@WebMvcTest + MockMvc）：
  - POST `/api/auth/sms-code` body `{"phone":"13800138000"}` → 200 BaseResp.success
  - POST `/api/auth/register` body `{"phone":"...","smsCode":"123456","nickname":"小明"}` 服务端 mock → 200 success，data 含 userId
  - POST `/api/auth/login` 用验证码或密码两种 mode → 200 success，data 含 token
  - 各种失败路径 → BaseResp.failed
- [ ] **C8.2**：跑 fail → 实现 Controller：3 个端点，每个 `@RequestBody` 用 `@Valid`，DTO 上加 `@NotBlank/@Pattern` 校验；body 有 `mode: "code" | "password"` 决定 login 走哪条；调对应 Service；返回 `BaseResp.success(data)`
- [ ] **C8.3**：DTO（`SendSmsCodeReq` `RegisterReq` `LoginReq` 含对应 `LoginData` 单接口响应）按 `java-backend-business-layer.md` §3.2 与 Req 同文件
- [ ] **C8.4**：commit `feat(user-core): AuthController + 注册/登录/验证码三个 HTTP 端点`

### Task C9：UserApiImpl

**Files:**
- Create: `.../user/api/UserApiImpl.java`
- Test: `.../test/user/api/UserApiImplTest.java`

**Steps:**
- [ ] **C9.1**：测试：`findById(1L)` 存在返回 UserDto、不存在返回 null；`existsByPhone(...)` 委托给 repo
- [ ] **C9.2**：跑 fail → 实现（按 `java-backend.md` §9.2 模板）
- [ ] **C9.3**：commit `feat(user-core): UserApiImpl 对内契约实现`

---

## Phase D · 角色 + 关系领域（sanyan-character-{api,core}）

### Task D1：sanyan-character-api

**Files:**
- Create: `business_packages/sanyan-character-api/pom.xml`
- Create: `.../character/CharacterApi.java` `dto/CharacterDto.java`

**Steps:**
- [ ] **D1.1**：`CharacterApi` 接口：`CharacterDto findById(Long id)`、`CharacterDto findDefault()`（一期就一个角色，前端拿默认即可）
- [ ] **D1.2**：`CharacterDto` record(Long id, String name, String avatarUrl, String basePromptVersion)
- [ ] **D1.3**：commit `feat(character-api): 契约`

### Task D2：数据库迁移 V2（characters + relationships）+ 林小满种子（V4）

**Files:**
- Create: `bootstrap/src/main/resources/db/migration/V2__init_characters_relationships.sql`
- Create: `bootstrap/src/main/resources/db/migration/V4__seed_sanyan_xiaoman.sql`

**Steps:**
- [ ] **D2.1**：V2：`characters` 表（id / name / avatar_url / base_prompt TEXT / persona_config JSONB / base_prompt_version VARCHAR(20) / created_at）+ `relationships` 表（id / user_id BIGINT FK / character_id BIGINT FK / intimacy_score INT DEFAULT 0 / current_stage SMALLINT DEFAULT 0 / last_active_at / created_at，UNIQUE(user_id, character_id)）+ 索引
- [ ] **D2.2**：V4：插入林小满种子数据 —— name='林小满'，avatar_url 暂用 placeholder，base_prompt 写入完整人设（职场新人 + 软萌可爱+俏皮 + 网络口语 + 网友身份 + 当前关系阶段=陌生人 + 严守内容尺度 L2），base_prompt_version='v1'
- [ ] **D2.3**：启动 Flyway，psql 验证 `SELECT * FROM characters;` 看到林小满
- [ ] **D2.4**：commit `feat(db): V2 characters/relationships + V4 林小满种子`

### Task D3：CharacterEntity + RelationshipEntity + Repos

**Files:**
- Create: `sanyan-character-core/pom.xml`（依赖 sanyan-character-api、sanyan-user-api、common-*）
- Create: `.../character/internal/{CharacterEntity,CharacterRepository,RelationshipEntity,RelationshipRepository,CharacterErrCode}.java`
- Create: `.../test/character/fixtures/CharacterTestFixtures.java`
- Test: `.../test/character/internal/{CharacterRepositoryIT,RelationshipRepositoryIT}.java`

**Steps:**
- [ ] **D3.1**：Fixtures + Repository IT 测试（保存查找正常）
- [ ] **D3.2**：跑 fail → 实现 entities + repos
- [ ] **D3.3**：`CharacterErrCode`：`CHARACTER_NOT_FOUND(2001, "角色不存在")`、`RELATIONSHIP_NOT_FOUND(2002, "关系记录不存在")`；更新 REGISTRY
- [ ] **D3.4**：commit `feat(character-core): entities + repositories`

### Task D4：UserRegisteredListener + RelationshipBootstrapService

**Files:**
- Create: `.../character/internal/RelationshipBootstrapService.java`
- Create: `.../character/event/UserRegisteredListener.java`
- Test: `.../test/character/event/UserRegisteredListenerTest.java`

**Steps:**
- [ ] **D4.1**：测试：发布 `UserRegisteredEvent` → listener 调用 service.bootstrap → 创建 user 与默认角色（findDefault）的初始 relationship 记录（intimacy=0, stage=0）
- [ ] **D4.2**：跑 fail → 实现 Service `@Transactional bootstrap(Long userId)`：找默认 character → 检查是否已存在 relationship → 不存在则创建；Listener `@Async @TransactionalEventListener(phase=AFTER_COMMIT)` 调 service
- [ ] **D4.3**：commit `feat(character-core): 注册即建立用户与林小满的初始关系`

### Task D5：CharacterApiImpl

**Files:**
- Create: `.../character/api/CharacterApiImpl.java`
- Test: `.../test/character/api/CharacterApiImplTest.java`

**Steps:**
- [ ] **D5.1**：测试 + 实现 `findById` `findDefault`（findDefault 返回 id=1 即林小满）
- [ ] **D5.2**：commit `feat(character-core): CharacterApiImpl`

---

## Phase E · LLM 适配层（sanyan-llm-{api,core}）

### Task E1：sanyan-llm-api

**Files:**
- Create: `sanyan-llm-api/pom.xml`
- Create: `.../llm/LLMProvider.java` `dto/{ChatMessage,ChatRequest,ChatResponse,LLMTaskType}.java`

**Steps:**
- [ ] **E1.1**：定义：
  - `ChatMessage(String role, String content)` —— role: "system" | "user" | "assistant"
  - `ChatRequest(List<ChatMessage> messages, double temperature, int maxTokens, Map<String,Object> extra)`
  - `ChatResponse(String content, int promptTokens, int completionTokens)`
  - `LLMTaskType` enum：`MAIN_CHAT, BACKGROUND`（Plan 1 只用 MAIN_CHAT）
  - `LLMProvider` 接口：`ChatResponse chat(ChatRequest request)`、`LLMTaskType supports()`（标识自己服务哪个任务类型）
- [ ] **E1.2**：commit `feat(llm-api): LLMProvider 抽象 + DTO`

### Task E2：QwenAdapter

**Files:**
- Create: `sanyan-llm-core/pom.xml`（依赖 sanyan-llm-api、common-error/cache/util、okhttp 4.x）
- Create: `.../llm/internal/{QwenAdapter,LLMConfig,LLMErrCode}.java`
- Test: `.../test/llm/internal/QwenAdapterTest.java`（用 OkHttp MockWebServer）

**Steps:**
- [ ] **E2.1**：`LLMErrCode`：`LLM_HTTP_FAIL(3001)`、`LLM_RESPONSE_INVALID(3002)`、`LLM_API_KEY_MISSING(3003)`、`LLM_RATE_LIMIT(3004)`；更新 REGISTRY
- [ ] **E2.2**：测试：用 MockWebServer 模拟阿里云百炼接口（`/api/v1/services/aigc/text-generation/generation`）
  - 200 + 成功 body → 返回 ChatResponse 含 content
  - 200 + 错误 body（业务错误）→ 抛 `BusinessException(LLM_RESPONSE_INVALID)`
  - 401 → 抛 `LLM_API_KEY_MISSING`
  - 429 → 抛 `LLM_RATE_LIMIT`
  - 500 → 抛 `LLM_HTTP_FAIL`
- [ ] **E2.3**：跑 fail → 实现 `QwenAdapter`：
  - `@Component`，注入 `LLMConfig`（`@ConfigurationProperties("3yan.llm.qwen")` 读 endpoint/apiKey/model）
  - 构造 OkHttp Request，body 按阿里云百炼协议（`{"model":"qwen-plus-character","input":{"messages":[...]},"parameters":{"temperature":..,"max_tokens":..}}`）
  - Header `Authorization: Bearer ${apiKey}`
  - 解析 response.output.text 字段
  - `supports() == MAIN_CHAT`
  - 30s 超时
- [ ] **E2.4**：`application-local.yml` 加 `3yan.llm.qwen.endpoint=https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation`、`apiKey=${QWEN_API_KEY:}`、`model=qwen-plus-character`
- [ ] **E2.5**：commit `feat(llm-core): QwenAdapter 接入阿里云百炼 qwen-plus-character`

### Task E3：LLMProviderFactory + LLMProviderRouter

**Files:**
- Create: `.../llm/internal/LLMProviderFactory.java`
- Create: `.../llm/api/LLMProviderRouter.java`
- Test: `.../test/llm/internal/LLMProviderFactoryTest.java`

**Steps:**
- [ ] **E3.1**：测试：注入多个 `LLMProvider`（mock），`getFor(LLMTaskType.MAIN_CHAT)` 返回 supports() == MAIN_CHAT 的那个；找不到抛 `IllegalStateException`（启动期就该发现配置问题）
- [ ] **E3.2**：跑 fail → 实现 Factory：构造时收集所有 `LLMProvider` Bean，按 `supports()` 建 map；`LLMProviderRouter` 暴露给业务层 `chat(LLMTaskType, ChatRequest)` 方法，内部委托 Factory 拿 Provider 调 `chat`
- [ ] **E3.3**：commit `feat(llm-core): Factory + Router 按 TaskType 路由`

---

## Phase F · 聊天 + WS（sanyan-chat-{api,core}）

### Task F1：数据库迁移 V3（messages 表）

**Files:**
- Create: `bootstrap/src/main/resources/db/migration/V3__init_messages.sql`

**Steps:**
- [ ] **F1.1**：SQL：`messages` 表（id BIGSERIAL PK / user_id BIGINT FK / character_id BIGINT FK / role VARCHAR(16) NOT NULL CHECK(role IN ('user','assistant','system')) / content TEXT NOT NULL / status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK(status IN ('pending','delivered','push_sent','read','failed')) / client_msg_id VARCHAR(60) NULL UNIQUE / created_at / delivered_at NULL / read_at NULL）+ 索引 (user_id, created_at DESC)
- [ ] **F1.2**：迁移并验证
- [ ] **F1.3**：commit `feat(db): V3 messages`

### Task F2：sanyan-chat-api

**Files:**
- Create: `sanyan-chat-api/pom.xml`
- Create: `.../chat/ChatApi.java` `dto/MessageDto.java`

**Steps:**
- [ ] **F2.1**：`ChatApi`：`List<MessageDto> listRecent(Long userId, Long characterId, int limit)`、`List<MessageDto> syncSince(Long userId, Long sinceMessageId)`
- [ ] **F2.2**：`MessageDto` record(Long id, Long userId, Long characterId, String role, String content, String status, Instant createdAt)
- [ ] **F2.3**：commit `feat(chat-api): 契约`

### Task F3：MessageEntity + Repository + 枚举

**Files:**
- Create: `sanyan-chat-core/pom.xml`（依赖 chat-api/llm-api/character-api/user-api/common-*/common-ws）
- Create: `.../chat/internal/{MessageEntity,MessageRepository,MessageStatus,MessageRole,ChatErrCode}.java`
- Test: `.../test/chat/internal/MessageRepositoryIT.java`
- Create: `.../test/chat/fixtures/MessageTestFixtures.java`

**Steps:**
- [ ] **F3.1**：枚举 `MessageStatus { PENDING, DELIVERED, PUSH_SENT, READ, FAILED }` + `MessageRole { USER, ASSISTANT, SYSTEM }`
- [ ] **F3.2**：`ChatErrCode`：`CONTENT_BLOCKED(4001)`、`LLM_FAILED(4002)`、`MESSAGE_NOT_FOUND(4003)`；更新 REGISTRY
- [ ] **F3.3**：Fixtures + Repository IT 测试 `findRecentByUserAndCharacter(userId, characterId, Pageable)` 返回按 created_at DESC
- [ ] **F3.4**：跑 fail → 实现 Entity（@Enumerated(EnumType.STRING)）+ Repository
- [ ] **F3.5**：commit `feat(chat-core): MessageEntity + Repository + 枚举`

### Task F4：PromptBuilder

**Files:**
- Create: `.../chat/internal/PromptBuilder.java`
- Test: `.../test/chat/internal/PromptBuilderTest.java`

**Steps:**
- [ ] **F4.1**：测试：
  - `buildPrompt(character, recentMessages)` 返回 `List<ChatMessage>`：第 1 条 `system` 是 character.basePrompt；后面是 recent 转 ChatMessage（保持时间顺序）
  - 空 recent → 仅 system
  - recent 超过 30 条只取最后 30 条
- [ ] **F4.2**：跑 fail → 实现
- [ ] **F4.3**：commit `feat(chat-core): PromptBuilder（system + 短期上下文 30 条）`

### Task F5：ContentSafetyFilter

**Files:**
- Create: `.../chat/internal/ContentSafetyFilter.java`
- Test: `.../test/chat/internal/ContentSafetyFilterTest.java`

**Steps:**
- [ ] **F5.1**：测试：`shouldBlock(text)` 命中黑名单（一份初始 ~30 个露骨词清单，写在 `application-local.yml` 的 `3yan.content.safety.blacklist` 列表）返回 true，不命中返回 false；输入侧（用户消息）和输出侧（LLM 回复）都用同一组规则
- [ ] **F5.2**：跑 fail → 实现：从 config 读取黑名单 + 简单 contains 匹配（一期不上 AC 自动机，单 list 30 词性能够用）；命中输入抛 `BusinessException(CONTENT_BLOCKED)`；命中输出 fallback 到固定话术 "嗯…我们换个话题聊好吗 (｡•́︿•̀｡)"
- [ ] **F5.3**：commit `feat(chat-core): ContentSafetyFilter（输入输出双侧黑名单）`

### Task F6：TypingDelayCalculator

**Files:**
- Create: `.../chat/internal/TypingDelayCalculator.java`
- Test: `.../test/chat/internal/TypingDelayCalculatorTest.java`

**Steps:**
- [ ] **F6.1**：测试：
  - 短消息（10 字以下）→ 1500-2500ms
  - 中消息（10-50 字）→ 2500-5000ms
  - 长消息（50+ 字）→ 按字数线性递增，最大 8000ms 封顶
  - 消息间间隔（拆多条时）→ 800-1500ms
- [ ] **F6.2**：跑 fail → 实现：用基础 `1500 + chars * 80` 公式（每字 80ms 模拟打字节奏）+ 随机 ±20% 抖动避免太规整；返回 `record DelayPlan(long preTypingMs, long messageGapMs)`
- [ ] **F6.3**：commit `feat(chat-core): TypingDelayCalculator（按字数模拟打字时长）`

### Task F7：ChatService（核心编排）

**Files:**
- Create: `.../chat/internal/ChatService.java`
- Test: `.../test/chat/internal/ChatServiceTest.java`

**Steps:**
- [ ] **F7.1**：测试（Mockito 单测，重点路径）：
  - 用户发消息 → 落库（USER, PENDING）→ 内容检查通过 → 拉取最近 30 条 → buildPrompt → 调 LLMProviderRouter → 输出内容检查 → 拆分多条（按句号/换行）→ 每条落库（ASSISTANT, PENDING）→ 计算 typing delays → 返回 List<ScheduledMessage>（含 message + preDelay）
  - 输入命中黑名单 → 抛 CONTENT_BLOCKED，未调 LLM
  - LLM 抛异常 → 用户消息状态保持 PENDING（之后由 worker 重试），不入 ASSISTANT 消息
  - 输出命中黑名单 → ASSISTANT 内容替换为 fallback 话术
- [ ] **F7.2**：跑 fail → 实现：方法 `handleUserMessage(Long userId, Long characterId, String content, String clientMsgId)` 返回 `record AssistantPlan(MessageEntity userMessage, List<ScheduledAssistantMessage> assistantMessages)`；`ScheduledAssistantMessage(MessageEntity entity, long preDelayMs)`
- [ ] **F7.3**：消息拆分逻辑：按 `[。！？\n]+` 分句，每 1-3 句聚成一条（阈值由 LLM 输出 prompt 控制）；空消息合并；最少 1 条最多 4 条
- [ ] **F7.4**：commit `feat(chat-core): ChatService 端到端编排`

### Task F8：ChatWebSocketHandler + JSON 协议

**Files:**
- Create: `.../chat/ws/{ChatWebSocketHandler,ChatWsMessage}.java`
- Test: `.../test/chat/ws/ChatWebSocketHandlerTest.java`

**Steps:**
- [ ] **F8.1**：定义 `ChatWsMessage` JSON schema：
  - 入站：`{"type":"send","clientMsgId":"...","content":"..."}` / `{"type":"ack","messageId":123}` / `{"type":"ping"}`
  - 出站：`{"type":"typing_start","characterId":1}` / `{"type":"message","id":...,"role":"assistant","content":"...","createdAt":...}` / `{"type":"typing_end"}` / `{"type":"pong"}` / `{"type":"error","code":...,"message":"..."}`
- [ ] **F8.2**：测试 `ChatWebSocketHandlerTest`（用 Spring Test 的 mock WebSocketSession）：
  - 收到 `send` 调 ChatService.handleUserMessage → 立即推送 `typing_start` → 按 plan 延迟 → 推 `message` → 推 `typing_end`
  - 收到 `ack` → 更新对应 message 的 status=delivered + delivered_at=now
  - 收到 `ping` → 立即回 `pong`
  - 连接建立 → 从 token 拿 userId（用 query param `?token=xxx` 或第一帧鉴权），register 到 WsConnectionRegistry；连接断开 → unregister
- [ ] **F8.3**：跑 fail → 实现：继承 `TextWebSocketHandler`，重写 `afterConnectionEstablished` / `handleTextMessage` / `afterConnectionClosed` / `handleTransportError`；用 ScheduledExecutorService 处理延迟推送（一期单实例够用）
- [ ] **F8.4**：在 `sanyan-chat-core` 的 `WebSocketConfigurer` 注册 endpoint `/ws/chat`
- [ ] **F8.5**：commit `feat(chat-core): WebSocket handler + JSON 协议 + ACK + 心跳`

### Task F9：MessageController（HTTP /sync）

**Files:**
- Create: `.../chat/web/{MessageController,SyncMessagesReq}.java`
- Test: `.../test/chat/web/MessageControllerIT.java`

**Steps:**
- [ ] **F9.1**：测试 `@WebMvcTest`：
  - GET `/api/messages/sync?sinceId=0` 带 token → 返回 BaseResp.success(List<MessageDto>) 该 user 所有 status != READ 的消息
  - 不带 token → 401（@LoginRequired）
- [ ] **F9.2**：跑 fail → 实现 controller `@LoginRequired` `@GetMapping("/api/messages/sync")` 接收 `@RequestParam Long sinceId`，从 `CurrentUser.id()` 拿 userId，委托 ChatService.syncSince
- [ ] **F9.3**：commit `feat(chat-core): GET /api/messages/sync 客户端启动拉取兜底`

### Task F10：ChatApiImpl

**Files:**
- Create: `.../chat/api/ChatApiImpl.java`
- Test: `.../test/chat/api/ChatApiImplTest.java`

**Steps:**
- [ ] **F10.1**：测试 + 实现 `listRecent` `syncSince`，委托 Repository
- [ ] **F10.2**：commit `feat(chat-core): ChatApiImpl`

---

## Phase G · 后端集成回归

### Task G1：端到端 SpringBootTest

**Files:**
- Test: `bootstrap/src/test/java/com/3yan/EndToEndIT.java`

**Steps:**
- [ ] **G1.1**：用 Testcontainers 拉真实 postgres+redis；mock LLM（在测试 profile 下用 `@Primary` 替换 QwenAdapter 为返回固定文本的 fake）；流程：
  1. POST /api/auth/sms-code → 200
  2. 从 Redis KvCache 拿验证码（测试模式直接读 `KvCache.get("sms:code:13800138000")`）
  3. POST /api/auth/register → 拿 userId
  4. POST /api/auth/login → 拿 token
  5. 验证 relationship 已 bootstrap（user 和林小满的初始关系记录存在）
  6. 用 Spring `WebSocketStompClient` 或裸 `StandardWebSocketClient` 连接 `/ws/chat?token=...`
  7. 发送 send 帧 `{"type":"send","content":"你好"}`
  8. 客户端依次收到 typing_start → message(s) → typing_end，断言内容包含 fake LLM 返回的文本
  9. 发 ack 帧，验证消息 status=delivered
  10. GET /api/messages/sync?sinceId=0 返回所有消息
- [ ] **G1.2**：跑通整个流程
- [ ] **G1.3**：commit `test(bootstrap): 端到端 SpringBootTest 覆盖注册→登录→WS聊天→sync`

### Task G2：ArchUnit 守护测试

**Files:**
- Test: `bootstrap/src/test/java/com/3yan/ArchitectureIT.java`

**Steps:**
- [ ] **G2.1**：继承 `ArchitectureTestBase`，跑全套 ArchUnit 规则（business -core 互不依赖、foundation 不依赖 business 等）
- [ ] **G2.2**：commit `test: ArchUnit 模块边界守护`

---

## Phase H · Flutter 项目骨架

### Task H1：创建 Flutter 主工程

**Files:**
- Modify: `3yan-app/pubspec.yaml`（已有空骨架）
- Create: `3yan-app/lib/main.dart`
- Create: `3yan-app/lib/home_page.dart`

**Steps:**
- [ ] **H1.1**：在 `3yan-app` 目录跑 `flutter create --project-name sanyan_app .`（覆盖式），删除自动生成的 `lib/main.dart` 内容
- [ ] **H1.2**：`pubspec.yaml` 添加依赖：`get`、`get_storage`、`dio`，dev 加 `mocktail`、`flutter_test`
- [ ] **H1.3**：`pubspec.yaml` 用 `path` 依赖 6 个 foundation_packages + 2 个 business_packages（Plan 1 内会逐个 create，先占 path 待补）
- [ ] **H1.4**：`main.dart`：
  ```dart
  void main() async {
    WidgetsFlutterBinding.ensureInitialized();
    await GetStorage.init();
    runApp(const 三言App());
  }
  class 三言App extends StatelessWidget {
    const 三言App({super.key});
    @override
    Widget build(BuildContext context) => GetMaterialApp(
      title: '三言',
      initialRoute: 三言Routes.splash,
      getPages: [
        ...三言AuthRoutes.routes(),
        ...三言ChatRoutes.routes(),
      ],
    );
  }
  ```
- [ ] **H1.5**：`home_page.dart`：StatelessWidget + 空 controller，先放占位 Text "三言"（一期 chat 是默认页，之后 H2 完成后切换 initialRoute 即可）
- [ ] **H1.6**：commit `feat(app): Flutter 主工程 + GetStorage 初始化 + GetMaterialApp 路由聚合`

### Task H2：foundation_packages 6 个空骨架

**Files:**
- Create 6 packages: `foundation_packages/{sanyan_uikit,sanyan_network,sanyan_routes,sanyan_user,sanyan_bizkit,sanyan_util}` 各自 pubspec + lib/<name>.dart 空 barrel + lib/src/

**Steps:**
- [ ] **H2.1**：每个用 `flutter create --template=package <name>`
- [ ] **H2.2**：commit `feat(foundation): 6 个 foundation_packages 骨架（uikit/network/routes/user/bizkit/util）`

### Task H3：sanyan_routes（路由名表）

**Files:**
- Create: `foundation_packages/sanyan_routes/lib/sanyan_routes.dart`

**Steps:**
- [ ] **H3.1**：定义所有跨模块路由名常量：`splash` / `register` / `login` / `chat`（Plan 1 仅这些）
- [ ] **H3.2**：commit `feat(routes): 路由名集中表`

### Task H4：sanyan_network（HttpClient + Base）

**Files:**
- Create: `foundation_packages/sanyan_network/lib/src/{http_client,base_req,base_resp}.dart`
- Test: `foundation_packages/sanyan_network/test/{base_resp_test,http_client_test}.dart`

**Steps:**
- [ ] **H4.1**：测试 `BaseResp.fromJson` 字段映射（success/code/message/data 泛型）；`HttpClient.post` 200 返回 BaseResp
- [ ] **H4.2**：跑 fail → 实现 `HttpClient`（基于 dio，封装 get/post/put/delete，统一 baseUrl 和 token interceptor）+ `BaseReq` 抽象类（要求子类实现 `Map<String, dynamic> parameters()` + `String path()` + 请求方法）+ `BaseResp<T>` 泛型类
- [ ] **H4.3**：用 mocktail mock dio 写测试
- [ ] **H4.4**：commit `feat(network): HttpClient + BaseReq + BaseResp`

### Task H5：sanyan_user（UserCenter）

**Files:**
- Create: `foundation_packages/sanyan_user/lib/src/{models/user,local/user_cache,user_center}.dart`
- Create: `foundation_packages/sanyan_user/lib/src/api/{sanyan_user_api,req/refresh_token_req}.dart`
- Test: `foundation_packages/sanyan_user/test/{user_cache_test,user_center_test}.dart`

**Steps:**
- [ ] **H5.1**：`User` 模型 record 风格（id, phone, nickname, avatarUrl）+ fromJson/toJson
- [ ] **H5.2**：`UserCache`：封装 `GetStorage('user')`，方法 `saveToken/saveUser/loadToken/loadUser/clear`，禁止暴露 GetStorage 实例
- [ ] **H5.3**：`UserCenter`（按 `flutter-foundation-layer.md` §6 强制 + 内存态 + 冷启动同步读 + 异步拉新）：
  - 单例 `UserCenter.instance`
  - `bootstrapSync()` —— 在 main 中 `await GetStorage.init()` 之后立即调用，**同步**从 UserCache 读 token+user 到内存
  - `currentToken`/`currentUser` getter（同步访问内存态）
  - `loginCompleted(LoginData)` —— 登录成功后写入：内存 + UserCache
  - `logout()` —— 清空内存 + UserCache
  - `isLoggedIn` getter
- [ ] **H5.4**：测试覆盖 cache 与 center
- [ ] **H5.5**：commit `feat(user): UserCenter + 冷启动同步读 token`

### Task H6：sanyan_uikit（基础组件）

**Files:**
- Create: `foundation_packages/sanyan_uikit/lib/src/{theme,colors,primary_button,input_field,toast}.dart`

**Steps:**
- [ ] **H6.1**：定义全局色板（主色调用三言品牌色 —— 粉紫渐变 `#E8C8E0` → `#A78BCB`），Theme 用 Material 3
- [ ] **H6.2**：`PrimaryButton` `InputField` `Toast`（基于 `Get.snackbar` 封装）
- [ ] **H6.3**：每个组件至少一个 widget test
- [ ] **H6.4**：commit `feat(uikit): 主题色 + 基础组件`

### Task H7：sanyan_util

**Files:**
- Create: `foundation_packages/sanyan_util/lib/src/{phone_util,date_util}.dart`

**Steps:**
- [ ] **H7.1**：`PhoneUtil.isValid(phone)` 同后端逻辑；`DateUtil.formatRelative(Instant)` "刚刚"/"x 分钟前"/"x 小时前"/"yyyy-MM-dd"
- [ ] **H7.2**：测试覆盖
- [ ] **H7.3**：commit `feat(util): PhoneUtil + DateUtil`

---

## Phase I · 业务包 sanyan_auth

### Task I1：包骨架 + barrel + routes

**Files:**
- Create: `business_packages/sanyan_auth/lib/{sanyan_auth.dart,sanyan_auth_routes.dart}`
- Create: `business_packages/sanyan_auth/test/sanyan_auth_suite.dart`

**Steps:**
- [ ] **I1.1**：`flutter create --template=package sanyan_auth` + 调整结构
- [ ] **I1.2**：barrel：export login_page、register_page、sanyan_auth_routes
- [ ] **I1.3**：routes 类：`三言AuthRoutes.routes() => [GetPage(name: 三言Routes.register, page: () => RegisterPage()), GetPage(name: 三言Routes.login, page: () => LoginPage())]`
- [ ] **I1.4**：suite 文件 main 函数聚合所有 _test.dart
- [ ] **I1.5**：commit `feat(auth): 包骨架 + 路由`

### Task I2：API 层（SmsCodeReq / RegisterReq / LoginReq）

**Files:**
- Create: `business_packages/sanyan_auth/lib/src/api/{sanyan_auth_api,req/{send_sms_code_req,register_req,login_req}}.dart`
- Test: `business_packages/sanyan_auth/test/api/sanyan_auth_api_test.dart`

**Steps:**
- [ ] **I2.1**：每个 `XxxReq` 类继承 `BaseReq`，实现 `path()` `method()` `parameters()`；同文件定义 `XxxData`（如 `LoginData(token, refreshToken, userId)`）
- [ ] **I2.2**：`三言AuthApi` 聚合入口提供 `sendSmsCode(phone)` `register(...)` `login(...)`
- [ ] **I2.3**：用 mocktail mock HttpClient 测试
- [ ] **I2.4**：commit `feat(auth-api): 三个接口 Req+Data + 聚合 API`

### Task I3：register_page + register_controller + 倒计时按钮

**Files:**
- Create: `business_packages/sanyan_auth/lib/src/register/{register_page,register_controller}.dart`
- Create: `business_packages/sanyan_auth/lib/src/widgets/sms_code_button.dart`
- Test: `business_packages/sanyan_auth/test/register_controller_test.dart`

**Steps:**
- [ ] **I3.1**：测试 controller：
  - `sendCode(phone)` 校验手机号格式 → 调 api → 启动 60s 倒计时 `RxInt smsCountdown`
  - `register(phone, code, nickname)` 校验非空 → 调 api → 成功后 UserCenter.loginCompleted（注册即登录）→ Get.offAllNamed(三言Routes.chat)
  - 失败 → Toast.show
- [ ] **I3.2**：跑 fail → 实现 controller（Get.put 在 Page 属性声明处，命名 c）
- [ ] **I3.3**：实现 register_page：手机号 InputField + 验证码 InputField + SmsCodeButton（带倒计时 Obx）+ 昵称 InputField + 注册 PrimaryButton；**Page 用 StatelessWidget，禁止 setState，所有响应式靠 Obx**
- [ ] **I3.4**：commit `feat(auth-register): 注册页 + 倒计时验证码`

### Task I4：login_page + login_controller（验证码 / 密码双 tab）

**Files:**
- Create: `business_packages/sanyan_auth/lib/src/login/{login_page,login_controller}.dart`
- Test: `business_packages/sanyan_auth/test/login_controller_test.dart`

**Steps:**
- [ ] **I4.1**：controller 测试：
  - `loginByCode(phone, code)` → 调 api → 成功 UserCenter.loginCompleted → 跳 chat
  - `loginByPassword(phone, password)` → 同上
  - `RxInt currentTab` 0/1 切换
  - 倒计时同 register
- [ ] **I4.2**：实现 login_page：顶部 TabBar（验证码登录 / 密码登录）+ 对应 form；用 `Obx(() => IndexedStack(index: c.currentTab.value, ...))` 切换；StatefulWidget 仅用于持有 TabController（按 `flutter.md` §3 例外）
- [ ] **I4.3**：commit `feat(auth-login): 双 tab 登录页`

---

## Phase J · 业务包 sanyan_chat

### Task J1：包骨架 + barrel + routes

**Files:**
- Create: `business_packages/sanyan_chat/lib/{sanyan_chat.dart,sanyan_chat_routes.dart}`

**Steps:**
- [ ] **J1.1**：barrel export ChatPage + routes 类
- [ ] **J1.2**：commit `feat(chat): 包骨架 + 路由`

### Task J2：API + models（Message + ChatWsEvent）

**Files:**
- Create: `business_packages/sanyan_chat/lib/src/api/{sanyan_chat_api,req/sync_messages_req,models/{message,chat_ws_event}}.dart`
- Test: `business_packages/sanyan_chat/test/api/sanyan_chat_api_test.dart`

**Steps:**
- [ ] **J2.1**：`Message` 跨接口领域模型（id/userId/characterId/role/content/status/createdAt + fromJson）
- [ ] **J2.2**：`ChatWsEvent` sealed class 风格（`TypingStart`/`TypingEnd`/`MessageEvent`/`Pong`/`ErrorEvent`）+ `parse(Map<String,dynamic>)` 工厂
- [ ] **J2.3**：`SyncMessagesReq` + `SyncMessagesData` 同文件
- [ ] **J2.4**：`三言ChatApi.syncSince(sinceId)`
- [ ] **J2.5**：测试覆盖 fromJson + ws event parse
- [ ] **J2.6**：commit `feat(chat-api): models + sync 接口`

### Task J3：ChatWsService（连接 + 心跳 + 重连）

**Files:**
- Create: `business_packages/sanyan_chat/lib/src/ws/chat_ws_service.dart`
- Test: `business_packages/sanyan_chat/test/ws/chat_ws_service_test.dart`

**Steps:**
- [ ] **J3.1**：测试：
  - `connect(token)` 建立 WebSocketChannel → 立刻启动 30s ping 定时器
  - 收到 pong → 重置心跳超时（90s 没收到任何帧则断开）
  - 收到 typing_start/message/typing_end → 通过 `Stream<ChatWsEvent>` 暴露
  - 用户调 `send(content)` → channel.sink.add(json)
  - 用户调 `ack(messageId)` → 同上
  - 连接断开（onError/onDone）→ 指数退避重连（1s, 2s, 5s, 10s, 30s, 30s...）
  - 主动 `disconnect()` 不重连
- [ ] **J3.2**：跑 fail → 实现（mock WebSocketChannel 测试）；Service 是 GetX `Service` 风格但实例化由 ChatController 持有
- [ ] **J3.3**：commit `feat(chat-ws): 连接 + 心跳 + 重连`

### Task J4：TypingDelaySimulator（客户端对应后端 plan）

**Files:**
- Create: `business_packages/sanyan_chat/lib/src/ws/typing_delay_simulator.dart`

**Steps:**
- [ ] **J4.1**：当客户端收到 typing_start → 显示输入指示器；收到 message → 隐藏指示器、append 气泡（一期客户端只信服务端推送的节奏，不主动加延迟）
- [ ] **J4.2**：commit `feat(chat-ws): 客户端按服务端节奏渲染`

### Task J5：chat_page + chat_controller

**Files:**
- Create: `business_packages/sanyan_chat/lib/src/chat/{chat_page,chat_controller}.dart`
- Create: `business_packages/sanyan_chat/lib/src/chat/widgets/{message_bubble,typing_indicator,input_bar}.dart`
- Test: `business_packages/sanyan_chat/test/chat/chat_controller_test.dart`

**Steps:**
- [ ] **J5.1**：controller 测试：
  - `onInit` → 拉取 `/api/messages/sync?sinceId=lastLocalId` 填充 `RxList<Message> messages` → 启动 `ChatWsService.connect(token)` → 监听 stream
  - `RxBool isTyping` —— 收到 typing_start 设 true，收到 typing_end / message 后过几百毫秒设 false
  - 收到 message event → 调 `ack(messageId)` → 加入 messages
  - 用户在 input_bar 点发送 → controller.send(content) → 立即 optimistic 加入 messages（role=user, status=pending）+ 调 `ChatWsService.send`
  - 错误 → Toast
- [ ] **J5.2**：跑 fail → 实现 controller
- [ ] **J5.3**：实现 widgets：
  - `MessageBubble`：根据 role 左右对齐，用户气泡渐变粉色，AI 气泡白色；时间戳分组显示（5 分钟内同 role 不重复显示时间）
  - `TypingIndicator`：3 个圆点循环动画（用 AnimatedBuilder + StatefulWidget 例外）
  - `InputBar`：底部输入框 + 发送按钮，发送时动画反馈
- [ ] **J5.4**：实现 chat_page（StatelessWidget；列表用 `Obx(() => ListView.builder(...))` 反向（reverse:true）；底部 InputBar 不在 Obx 内
- [ ] **J5.5**：commit `feat(chat): 聊天页 + 气泡 + 输入指示器 + 输入条`

### Task J6：suite 聚合

**Files:**
- Create: `business_packages/sanyan_chat/test/sanyan_chat_suite.dart`

**Steps:**
- [ ] **J6.1**：聚合本包所有 _test.dart 的 main 函数
- [ ] **J6.2**：commit `test(chat): 测试聚合入口`

---

## Phase K · 端到端联调 + dogfood

### Task K1：iOS Keychain / Android EncryptedSharedPreferences token 持久化

**Files:**
- Modify: `sanyan_user` 包内 `user_cache.dart` 评估 GetStorage 默认存储是否够用；如不够安全，引入 `flutter_secure_storage` 仅用于 token

**Steps:**
- [ ] **K1.1**：本期决定：**用 GetStorage 默认存储**（按 flutter rule 强制规范）。GetStorage 在 iOS 走 ApplicationDocuments 沙盒、在 Android 走 EncryptedSharedPreferences 等价存储，对内测够用。如果未来需要更严格的 token 加密，再引入额外加密层
- [ ] **K1.2**：commit `chore(user): 确认 token 持久化策略走 GetStorage 默认存储`

### Task K2：联调 + 体感记录

**Steps:**
- [ ] **K2.1**：本地 `./scripts/dev-up.sh` 启动 postgres+redis；后端 `mvn -pl bootstrap spring-boot:run`；手机模拟器 `flutter run`（设置 baseUrl 指向 `http://本机 IP:8080` 和 ws `ws://本机 IP:8080/ws/chat`）
- [ ] **K2.2**：完整跑：注册 → 收到验证码（看后端日志）→ 输入验证码完成注册 → 自动跳到聊天页 → 发送 "你好" → 看到"正在输入..." → 收到林小满回复（拆成 1-3 条气泡）
- [ ] **K2.3**：dogfood checklist：
  - [ ] 聊半小时是否出戏？
  - [ ] 林小满人设稳定吗？（不会突然变成 AI 助手）
  - [ ] 内容尺度（暧昧）合适吗？
  - [ ] 打字延迟感觉自然吗？还是太规整？
  - [ ] 网络抖动 WS 重连是否丝滑？
  - [ ] kill app 后重开历史能拉回吗？
- [ ] **K2.4**：把 dogfood 发现的 issues 列在 `docs/dogfood/plan-1-issues.md`
- [ ] **K2.5**：commit `docs: Plan 1 dogfood 反馈记录`

---

## Self-Review Checklist

完成上述 task 后，对照 spec 和规则检查：

**Spec 覆盖**：
- [x] 真北星 / 反 AI 感原则 → 落在 F6+F8（typing delay + 完整消息推送非流式）+ J4（客户端按服务端节奏）
- [x] 单角色（林小满）→ D2 种子 + D5 默认查询
- [x] 关系养成（基础设施）→ D2 表 + D4 注册即建立关系（亲密度引擎留 Plan 3）
- [x] 长期记忆 B/C/RAG → 不在 Plan 1（属 Plan 2）
- [x] LLM Adapter + qwen-character → E1-E3
- [x] 内容尺度 L2 → F5 黑名单 + 输入输出双侧
- [x] WebSocket 全栈 → B7 + F8 + J3
- [x] 4 层可靠投递（局部）→ Plan 1 实现 L1（WS push）+ L4（/sync），L2/L3（ACK 超时 fallback 到推送、APNs/个推）属 Plan 4
- [x] 主动消息 → 不在 Plan 1（Plan 4）
- [x] 用户体系（手机号 + JWT，邮箱字段预留）→ B3 + C2-C8
- [x] 变现字段预留 → C2 SQL 已含 subscription_status 等

**Placeholder scan**：
- 所有 task 已展开关键文件路径 + 测试要点 + 关键代码片段，无 "TBD" / "implement later"
- TDD 5 步文档头声明强制约束，子代理执行时严格 Red-Green-Refactor

**Type consistency**：
- `MessageStatus` 枚举值在 SQL（V3）+ Java enum（F3）+ Flutter Message model（J2）+ WS JSON 中一致：`pending/delivered/push_sent/read/failed`
- `MessageRole`：`user/assistant/system`（小写，匹配 LLM 协议）
- `LLMTaskType`：`MAIN_CHAT/BACKGROUND`（一期只用 MAIN_CHAT，BACKGROUND 留 Plan 2）
- `LoginData` 字段在后端（C8）+ 前端（I2）一致：`token / refreshToken / userId`

---

## Execution Handoff

**用户已选 subagent-driven 模式。** 进入执行流程：

1. 调用 `superpowers:subagent-driven-development` skill
2. 每个 Task（A1, A2, ..., K2）派一个独立 general-purpose 子代理执行
3. 每个 Task 完成后**依次**派两个独立子代理审查：spec 审查（general-purpose）+ 代码质量审查（superpowers:code-reviewer）
4. 双轮审查通过才能进入下一 Task
5. 任何 task 内的 TDD 5 步必须严格执行，子代理 prompt 中显式注入 TDD 铁律
