# Java 后端通用架构设计规范

**类型**：通用架构规范（不涉及任何具体业务）
**技术栈**：Java 17 + Spring Boot 3.x + Maven + MySQL + Redis
**定位**：适用于所有新建的 Java 后端项目；现有项目按本规范重写或渐进迁移
**对应关系**：与 app 侧 `~/.claude/rules/flutter.md` + `flutter-business-layer.md` + `flutter-foundation-layer.md` 同构，属于另一端的"通用技术栈规范"

---

## 1. 架构总览

### 1.1 两层结构（强制）

```
<project_root>/
├── pom.xml                                    # 父 POM（聚合 + 依赖管理 + Enforcer 规则）
├── bootstrap/                                 # 启动工程（聚合所有业务 + 基础模块）
│   └── src/main/java/.../Application.java
│
├── business_packages/                         # ── 业务层（按业务领域划分，不对齐 app tab）
│   ├── <proj>-<domain-a>-api/                 # 领域 A 的对内契约
│   ├── <proj>-<domain-a>-core/                # 领域 A 的实现
│   ├── <proj>-<domain-b>-api/
│   ├── <proj>-<domain-b>-core/
│   └── ...
│
└── foundation_packages/                       # ── 基础层（通用能力）
    ├── <proj>-common-error/                   # 异常 + 错误码（强制）
    ├── <proj>-common-web/                     # HTTP 响应封装（强制）
    ├── <proj>-common-auth/                    # 鉴权（强制）
    ├── <proj>-common-cache/                   # Redis 封装（强制）
    ├── <proj>-common-storage/                 # 对象存储封装（强制）
    ├── <proj>-common-util/                    # 通用工具（强制）
    ├── <proj>-common-test/                    # 测试基础（强制）
    └── <proj>-common-ws/                      # WebSocket（按需）
```

### 1.2 依赖方向（强制）

```
business_packages ──► foundation_packages              ✓ 业务层可依赖任意基础层模块
foundation_packages ─► foundation_packages             ✓ 基础层之间可互相依赖（避免循环）
business_packages ─x─ business_packages                ✗ 业务模块间禁止直接 import 彼此的 -core
business_packages ──► 其他 business_packages 的 -api   ✓ 通过 -api 契约调用
```

业务模块之间**不允许**直接 import 彼此的 `-core`（通过 Maven Enforcer + ArchUnit 强制）。跨业务模块通信走两种机制：

- **同步查询**：依赖对方 `-api` 模块，`@Autowired XxxApi`
- **通知扩散**：Spring 领域事件（`ApplicationEventPublisher` + `@TransactionalEventListener`）

### 1.3 结构不变性声明

- 所有 Java 后端项目**必须**按本文档的目录与模块划分组织代码
- 遇到规范未覆盖的新场景，**先更新规范再落地项目**，禁止在具体项目里临时发挥
- 新增业务领域模块时，**一律**按 `-api` + `-core` 双模块创建
- 新增基础能力模块时，按本文档第 8 节的清单命名与边界

---

## 2. 包命名规范

### 2.1 项目前缀（强制）

所有 Maven 子模块**强制统一使用项目短名前缀** `<proj>-`。前缀值由项目定，但**同一项目内所有模块必须一致**。

### 2.2 命名模式

| 类别 | 命名模式 | 示例 |
|------|---------|------|
| 业务领域契约模块 | `<proj>-<domain>-api` | `<proj>-user-api` |
| 业务领域实现模块 | `<proj>-<domain>-core` | `<proj>-user-core` |
| 基础能力模块 | `<proj>-common-<role>` | `<proj>-common-error` |
| Java 包根 | `com.<proj>.<domain>` / `com.<proj>.common.<role>` | `com.myapp.user` |
| 启动工程 | `<proj>-bootstrap` 或 `<proj>-server` | `<proj>-bootstrap` |

---

## 3. 业务领域模块（`-api` + `-core`）

每个业务领域由两个 Maven 子模块构成。

### 3.1 `-api` 模块 —— 对内契约

**作用**：模块对**后端内部**其他模块暴露的契约。只含接口与 DTO，**不含任何实现**。

#### 3.1.1 目录结构

```
<proj>-<domain>-api/
└── src/main/java/com/<proj>/<domain>/
    ├── <Domain>Api.java                     # 唯一的主接口，放顶层
    ├── dto/
    │   ├── <Domain>Dto.java                 # 对外 DTO
    │   └── <Domain>SummaryDto.java          # 简化版 DTO（按需）
    └── event/
        ├── <Domain><Action1>Event.java      # 领域事件（对外契约的一部分）
        └── <Domain><Action2>Event.java
```

#### 3.1.2 核心约束

- **一个 `-api` 模块只有一个主 `<Domain>Api` 接口**。接口职责太多说明领域划分过粗，应拆领域
- 接口放在**顶层包**（不进 `dto/` 或其他子包）
- DTO 放 `dto/` 子包
- 领域事件放 `event/` 子包（类型必须是 Java 17 `record`）
- **禁止**在 `-api` 模块里写任何实现代码或业务逻辑

#### 3.1.3 示例

```java
// <proj>-user-api/src/main/java/com/<proj>/user/UserApi.java
package com.<proj>.user;

import com.<proj>.user.dto.UserDto;

public interface UserApi {
    UserDto findById(Long userId);
    boolean existsByPhone(String phone);
}

// <proj>-user-api/src/main/java/com/<proj>/user/dto/UserDto.java
package com.<proj>.user.dto;

public record UserDto(Long id, String phone, String nickname, String avatarUrl) {}

// <proj>-user-api/src/main/java/com/<proj>/user/event/UserRegisteredEvent.java
package com.<proj>.user.event;

import java.time.Instant;

public record UserRegisteredEvent(Long userId, String phone, Instant registeredAt) {}
```

### 3.2 `-core` 模块 —— 实现

**作用**：模块的业务逻辑实现。对外提供**两个入口**：

- **HTTP 入口**（给 app/web/第三方）：`web/` 子包下的 Controller
- **Java 接口入口**（给后端内部模块）：`api/` 子包下的 `XxxApiImpl`

#### 3.2.1 目录结构（强制）

```
<proj>-<domain>-core/
└── src/main/java/com/<proj>/<domain>/
    ├── web/                                 # HTTP 对外层
    │   ├── <Domain>Controller.java
    │   ├── <Action1>Req.java                # HTTP 请求 DTO（与响应同文件）
    │   └── <Action2>Req.java
    ├── internal/                            # 对内实现层（业务逻辑核心）
    │   ├── <Domain><Action1>Service.java    # 按动作拆 Service
    │   ├── <Domain><Action2>Service.java
    │   ├── <Domain>Repository.java          # 数据访问（共享查询下沉到这里）
    │   ├── <Domain>Entity.java              # JPA 实体（带 Entity 后缀）
    │   └── <Domain>ErrCode.java             # 本模块错误码 enum
    ├── api/                                 # 对内 Java 接口实现
    │   └── <Domain>ApiImpl.java             # 实现 -api 模块的 <Domain>Api
    └── event/                               # 本模块订阅别的模块事件的监听器
        └── <Other><Event>EventListener.java
```

#### 3.2.2 `web/` 子包

放**外部流量入口**：Controller + HTTP 请求/响应 DTO。

- **Controller 命名**：`<Domain>Controller`
- **Controller 只做三件事**：接请求 → 调 Service → 返回响应。**禁止**在 Controller 写业务逻辑
- **HTTP 请求 DTO 命名**：`<Action>Req`
- **HTTP 响应 DTO 命名**（分两类，与 app 侧 `接口请求与应答命名规范` 对齐）：
  - **单接口专用响应**：`<Action>Data`，**必须与 `<Action>Req` 放在同一个文件**
  - **跨接口复用的领域模型**：保留业务命名（如 `UserProfile`），不加 `Data` 后缀

#### 3.2.3 `internal/` 子包

放**对内实现细节**：Service + Repository + Entity + 领域逻辑 + 错误码。

- **禁止**其他模块直接 import `internal/` 下的任何类
- **Service 按动作拆**：命名 `<Domain><Action>Service`（名词开头，方便按字母聚合）
  - 示例：`UserRegisterService`、`UserLoginService`、`UserProfileUpdateService`
  - 一个 Service 方法 = 一个 `@Transactional` 边界
  - 跨 Service 的共享查询逻辑**下沉到 Repository**（写在 Repository 里作为 `findByXxx` / `existsByXxx` 方法），**不要**再抽 `UserFinderService` 这种查询专用 Service
- **Repository 命名**：`<Domain>Repository`，继承 `JpaRepository<Entity, Id>`
- **Entity 命名**：`<Domain>Entity`（**带 `Entity` 后缀**，区别于同领域的 `<Domain>Dto` 和 HTTP 响应模型）
- **领域逻辑**：简单业务规则直接在 Service 方法里用 `if` + 抛 `BusinessException`；复杂规则抽独立类（如 `XxxPolicy`），仍放 `internal/`

#### 3.2.4 `api/` 子包

放**对内 Java 接口实现**：`<Domain>ApiImpl` implements `<Domain>Api`。

- 通常直接委托给 `internal/` 下的 Service
- 同一个 Service 既被 Controller 调用（HTTP 入口）也被 ApiImpl 调用（后端内部入口）
- 避免在 ApiImpl 里写业务逻辑，让 ApiImpl 保持**薄**，实际逻辑在 Service 里

#### 3.2.5 `event/` 子包

放**事件监听器（Listener）**。Listener 可以订阅：

- **本模块自己发的事件**（事件定义在本模块 `-api/event/`，本模块订阅做后续响应，如发完消息更新会话元数据）
- **其他模块发的事件**（事件定义在对方 `-api/event/`，本模块订阅做扩散响应，如用户注册后发欢迎邮件）

事件**定义**一律放在**发布方**的 `-api/event/` 里；Listener 放在**订阅方**的 `-core/event/` 下。

- 命名：`<EventType>Listener`（例：`UserRegisteredListener`）
- 若同一个模块有多个 Listener 订阅同一事件（按关注点拆分），命名可加后缀：`UserRegisteredWelcomeMailListener` / `UserRegisteredStatsListener`
- 订阅默认：`@Async @TransactionalEventListener(phase = AFTER_COMMIT)`
- 若需同步订阅（罕见），用普通 `@EventListener` 并**在代码注释里明确说明理由**

#### 3.2.6 示例

```java
// <proj>-user-core/src/main/java/com/<proj>/user/internal/UserEntity.java
package com.<proj>.user.internal;

import jakarta.persistence.*;
import java.time.Instant;
import lombok.Data;

@Entity
@Table(name = "user")
@Data
public class UserEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String phone;
    private String nickname;
    private String avatarUrl;
    private String passwordHash;
    private Instant createdAt;
}

// <proj>-user-core/src/main/java/com/<proj>/user/internal/UserRepository.java
package com.<proj>.user.internal;

import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<UserEntity, Long> {
    boolean existsByPhone(String phone);
}

// <proj>-user-core/src/main/java/com/<proj>/user/internal/UserRegisterService.java
package com.<proj>.user.internal;

import com.<proj>.common.error.BusinessException;
import com.<proj>.user.event.UserRegisteredEvent;
import lombok.RequiredArgsConstructor;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.Instant;

@Service
@RequiredArgsConstructor
public class UserRegisterService {
    private final UserRepository repo;
    private final BCryptPasswordEncoder passwordEncoder;
    private final ApplicationEventPublisher events;

    @Transactional
    public Long register(String phone, String password, String nickname) {
        if (repo.existsByPhone(phone)) {
            throw new BusinessException(UserErrCode.PHONE_ALREADY_REGISTERED);
        }
        UserEntity user = new UserEntity();
        user.setPhone(phone);
        user.setNickname(nickname);
        user.setPasswordHash(passwordEncoder.encode(password));
        user.setCreatedAt(Instant.now());
        repo.save(user);

        events.publishEvent(new UserRegisteredEvent(user.getId(), phone, user.getCreatedAt()));
        return user.getId();
    }
}

// <proj>-user-core/src/main/java/com/<proj>/user/api/UserApiImpl.java
package com.<proj>.user.api;

import com.<proj>.user.UserApi;
import com.<proj>.user.dto.UserDto;
import com.<proj>.user.internal.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserApiImpl implements UserApi {
    private final UserRepository repo;

    @Override
    public UserDto findById(Long userId) {
        return repo.findById(userId)
                .map(e -> new UserDto(e.getId(), e.getPhone(), e.getNickname(), e.getAvatarUrl()))
                .orElse(null);
    }

    @Override
    public boolean existsByPhone(String phone) {
        return repo.existsByPhone(phone);
    }
}
```

---

## 4. 模块间依赖与边界守护

### 4.1 依赖方向规则

| 依赖关系 | 允许 | 说明 |
|---------|------|------|
| business `-core` → 自己的 `-api` | ✅ | 实现自己的契约 |
| business `-core` → 其他 business 的 `-api` | ✅ | 同步调用其他领域 |
| business `-core` → foundation 任意模块 | ✅ | 用基础能力 |
| business `-core` → 其他 business 的 `-core` | ❌ | **严禁** |
| business `-api` → 其他任何模块 | ❌ | `-api` 是纯契约，不依赖任何东西（只依赖 JDK 标准库） |
| foundation → foundation | ✅ | 避免循环 |
| foundation → business 任意模块 | ❌ | **严禁**（基础层不知道业务层） |

### 4.2 同步调用：`@Autowired XxxApi`

跨业务领域的同步查询走 `-api` 接口：

```java
// message-core 里的 Service
@Service
@RequiredArgsConstructor
public class MessageSendService {
    private final UserApi userApi;          // 来自 <proj>-user-api
    private final RelationApi relationApi;  // 来自 <proj>-relation-api

    public void send(Long fromId, Long toId, String content) {
        if (!relationApi.isFriend(fromId, toId)) {
            throw new BusinessException(MessageErrCode.NOT_FRIEND);
        }
        // ...
    }
}
```

### 4.3 通知扩散：领域事件

详见第 6 节。

### 4.4 边界守护（两层）

#### 第一层：Maven Enforcer（POM 层）

在父 POM 里配置 `maven-enforcer-plugin` 的 `banned-dependencies` 规则：

```xml
<!-- 示例片段 -->
<rule implementation="org.apache.maven.enforcer.rules.dependency.BannedDependencies">
  <excludes>
    <!-- 任何 -core 模块不允许依赖其他 -core -->
    <exclude>com.<proj>:*-core</exclude>
  </excludes>
  <includes>
    <!-- 自己的 -core 除外 -->
    <include>com.<proj>:${project.artifactId}</include>
  </includes>
</rule>
```

违规时 `mvn validate` 阶段即失败。

#### 第二层：ArchUnit（代码层 CI）

`common-test` 提供一个 ArchUnit 测试基类，CI 跑的测试会扫所有模块的依赖图：

```java
@AnalyzeClasses(packages = "com.<proj>")
class ArchitectureTest {

    @ArchTest
    static final ArchRule business_core_modules_must_not_depend_on_each_other =
        noClasses().that().resideInAPackage("..<domain>.core..")
            .should().dependOnClassesThat().resideInAPackage("..<other-domain>.core..");
}
```

#### 第三层（业务异常专用）：ErrCodeConflictDetector

启动时扫描所有 `ErrCode` enum，发现 code 值冲突直接启动失败（详见第 5 节）。

---

## 5. 业务异常设计

### 5.1 全项目唯一 `BusinessException`

**不拆子异常类**。所有业务失败都抛 `BusinessException`，携带 `ErrCode` 和可选的覆盖文案。

```java
// common-error/BusinessException.java
public class BusinessException extends RuntimeException {
    @Getter private final ErrCode errCode;

    public BusinessException(ErrCode errCode) {
        super(errCode.getDefaultMessage());
        this.errCode = errCode;
    }

    public BusinessException(ErrCode errCode, String overrideMessage) {
        super(overrideMessage);
        this.errCode = errCode;
    }
}
```

### 5.2 `ErrCode` 接口 + 按模块拆 enum

```java
// common-error/ErrCode.java
public interface ErrCode {
    int getCode();
    String getDefaultMessage();
}

// common-error/CommonErrCode.java（通用错误，400-499 区间）
@Getter
@AllArgsConstructor
public enum CommonErrCode implements ErrCode {
    TOKEN_EXPIRED(400, "登录态过期"),
    KICKED_OFFLINE(405, "账号被踢下线"),
    PARAM_INVALID(410, "参数错误"),
    FORBIDDEN(403, "无权限"),
    NOT_FOUND(404, "资源不存在"),
    ;
    private final int code;
    private final String defaultMessage;
}

// <proj>-user-core/internal/UserErrCode.java（1000-1999 区间）
@Getter
@AllArgsConstructor
public enum UserErrCode implements ErrCode {
    PHONE_ALREADY_REGISTERED(1001, "手机号已注册"),
    AVATAR_INVALID_FORMAT(1002, "修改头像失败：文件格式不支持"),
    AVATAR_SIZE_TOO_LARGE(1003, "修改头像失败：文件过大"),
    NICKNAME_DUPLICATE(1004, "昵称已被占用"),
    ;
    private final int code;
    private final String defaultMessage;
}
```

### 5.3 code 区间约定

在基础层 `common-error` 模块的 `README.md` 或专门的 `ERROR_CODE_REGISTRY.md` 里维护一张登记表：

```
400-499    通用错误（CommonErrCode）
1000-1999  <domain-a>
2000-2999  <domain-b>
3000-3999  <domain-c>
...
9000-9999  预留
```

**新模块申请 code 区间时必须先更新登记表**，避免分配冲突。

### 5.4 `ErrCodeConflictDetector`（启动时硬约束）

`common-error` 提供的组件，启动时自动扫描所有实现 `ErrCode` 的 enum，发现 code 冲突**立即抛异常**让应用启动失败：

```java
// common-error/ErrCodeConflictDetector.java
@Component
@RequiredArgsConstructor
public class ErrCodeConflictDetector implements ApplicationRunner {
    private final ApplicationContext context;

    @Override
    public void run(ApplicationArguments args) {
        Map<Integer, String> seen = new HashMap<>();
        for (Class<? extends ErrCode> clazz : findAllErrCodeEnums()) {
            if (!clazz.isEnum()) continue;
            for (ErrCode code : clazz.getEnumConstants()) {
                String location = clazz.getSimpleName() + "." + ((Enum<?>) code).name();
                String existing = seen.put(code.getCode(), location);
                if (existing != null) {
                    throw new IllegalStateException(String.format(
                        "ErrCode 冲突：code=%d 被 %s 和 %s 同时定义",
                        code.getCode(), existing, location));
                }
            }
        }
    }

    private Set<Class<? extends ErrCode>> findAllErrCodeEnums() {
        // 用 Reflections 或 Spring 的 ClassPathScanningCandidateComponentProvider
        // 扫描 classpath 下所有实现 ErrCode 的 enum 类
        // 实现细节省略
    }
}
```

### 5.5 统一响应体 `BaseResp<T>`

由 `common-web` 提供，所有 HTTP 响应统一用这个结构：

```java
// common-web/BaseResp.java
@Data
public class BaseResp<T> {
    public static final int SUCCESS_CODE = 0;
    public static final String SUCCESS_MESSAGE = "ok";

    private boolean success;           // 业务是否成功：app/web 端直接判断这个字段
    private int code;                  // 失败时的错误类型区分（成功时固定为 0）
    private String message;            // 展示文案（成功时为 "ok"，失败时为错误描述）
    private T data;                    // 业务数据（仅成功时有值）

    public static <T> BaseResp<T> success(T data) {
        BaseResp<T> r = new BaseResp<>();
        r.setSuccess(true);
        r.setCode(SUCCESS_CODE);
        r.setMessage(SUCCESS_MESSAGE);
        r.setData(data);
        return r;
    }

    public static <T> BaseResp<T> failed(int code, String message) {
        BaseResp<T> r = new BaseResp<>();
        r.setSuccess(false);
        r.setCode(code);
        r.setMessage(message);
        return r;
    }
}
```

Controller 方法可以直接返回业务数据（如 `UserDto`），由 `common-web` 的全局响应包装器自动包成 `BaseResp<UserDto>` 返回给客户端。

**字段语义约定**：
- `success`：**业务是否成功**。app / web 端解析后**直接用这个字段判断成功与否**，不必再判断 code。与 app 侧 `BaseResp.success` 同构。
- `code`：仅用于**失败时区分错误类型**（通用错误 vs 各模块业务错误）。成功时固定为 `0`，任何 `ErrCode` 实现**不得**使用 `0` 作为 code 值。
- `message`：展示文案。成功时 `"ok"`；失败时是具体错误描述，app 端可直接 `Toast` 展示。
- `data`：业务数据。成功时有值，失败时为 `null`。

### 5.6 全局异常处理器

`common-web` 提供 `GlobalExceptionHandler`：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public BaseResp<Void> onBusinessException(BusinessException e) {
        return BaseResp.failed(e.getErrCode().getCode(), e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public BaseResp<Void> onValidation(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        return BaseResp.failed(CommonErrCode.PARAM_INVALID.getCode(), msg);
    }

    @ExceptionHandler(Exception.class)
    public BaseResp<Void> onUnknown(Exception e) {
        log.error("unhandled exception", e);
        return BaseResp.failed(500, "服务器错误，请稍后重试");
    }
}
```

### 5.7 HTTP 响应契约

HTTP 层永远返回 200（除非通信层出错），业务成败看 body 里的 `success`：

```json
// 成功
{ "success": true,  "code": 0,    "message": "ok",           "data": { ... } }

// 通用错误（400-499）
{ "success": false, "code": 400,  "message": "登录态过期",    "data": null }

// 业务错误（各模块 1000+）
{ "success": false, "code": 1001, "message": "手机号已注册",  "data": null }
```

前端处理逻辑：
- `success === true`：业务成功，取 `data`
- `success === false`：业务失败，按 `code` 区分处理
  - `code >= 400 && code < 500`：通用错误（400 跳登录页，405 提示被踢下线，等）
  - `code >= 1000`：业务错误（按 code 精确处理；一般场景直接 Toast 展示 `message`）

---

## 6. 领域事件规范

### 6.1 事件类定义位置

**`-api/event/` 下**（作为对外契约的一部分）。订阅方只需依赖事件发布方的 `-api` 模块即可订阅，不依赖 `-core`。

### 6.2 命名规则

`<Domain><动作过去式>Event`：

- ✅ `UserRegisteredEvent` / `MessageSentEvent` / `AvatarUpdatedEvent`
- ❌ `UserRegisterEvent`（动词原形暗示"动作进行中"）/ `RegisterUser`（缺后缀）

事件描述"**已发生的事实**"，订阅者看到事件名就知道该事已经发生，应该做后续响应。

### 6.3 事件类型：Java 17 `record`

```java
public record UserRegisteredEvent(Long userId, String phone, Instant registeredAt) {}
```

**不可变** + **字段即全部状态** + **简洁**。事件天生适合 record。

### 6.4 发布：`ApplicationEventPublisher`

Service 里注入 Spring 内置发布器，在业务方法里发布：

```java
@Service
@RequiredArgsConstructor
public class UserRegisterService {
    private final ApplicationEventPublisher events;

    @Transactional
    public Long register(...) {
        // ... 存库
        events.publishEvent(new UserRegisteredEvent(user.getId(), phone, Instant.now()));
        return user.getId();
    }
}
```

### 6.5 订阅：默认 `@TransactionalEventListener(AFTER_COMMIT) + @Async`

```java
@Component
@RequiredArgsConstructor
public class UserRegisteredEventListener {
    private final WelcomeMailService mailService;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onUserRegistered(UserRegisteredEvent event) {
        mailService.sendWelcome(event.userId());
    }
}
```

**为什么默认这样**：
- `AFTER_COMMIT`：保证事务**成功提交后**才触发订阅者，避免"发布时事务还没提交，订阅者却看到了事件"的数据不一致
- `@Async`：异步执行，发布者不等订阅者处理完

### 6.6 同步订阅的例外

若必须在主事务内同步处理（例如审计日志必须在事务内落库），可以用普通 `@EventListener`，但**必须在代码注释里写明理由**：

```java
@EventListener
public void onXxx(XxxEvent event) {
    // 同步订阅：审计日志必须在事务内落库，保证原子性
    ...
}
```

---

## 7. 测试规范

### 7.1 目录结构

```
<proj>-<domain>-core/
└── src/test/java/com/<proj>/<domain>/          # 镜像 src/main
    ├── web/
    │   └── <Domain>ControllerIT.java           # @WebMvcTest 集成测试
    ├── internal/
    │   ├── <Domain><Action>ServiceTest.java    # Mockito 单测
    │   └── <Domain>RepositoryIT.java           # @DataJpaTest（如果有自定义查询）
    ├── api/
    │   └── <Domain>ApiImplTest.java            # Mockito 单测
    ├── event/
    │   └── <Domain><Event>EventListenerTest.java
    └── fixtures/
        └── <Domain>TestFixtures.java           # Object Mother（强制）
```

### 7.2 分层测试工具

| 测试对象 | 推荐工具 | 后缀 | 速度 |
|---------|---------|------|------|
| Service / ApiImpl / EventListener 业务逻辑 | 纯 **Mockito 单测**（不启 Spring） | `*Test` | ⚡⚡⚡ |
| Controller | `@WebMvcTest`（只启 Web 切片，mock Service） | `*IT` | ⚡⚡ |
| Repository 的自定义查询 | `@DataJpaTest`（只启 JPA 切片，用 H2） | `*IT` | ⚡⚡ |
| 端到端关键路径 | `@SpringBootTest`（完整上下文 + H2） | `*IT` | ⚡ |

### 7.3 Maven 插件分工

- `maven-surefire-plugin`：默认只跑 `*Test`（每次 `mvn test` 都跑，快）
- `maven-failsafe-plugin`：默认只跑 `*IT`（`mvn verify` / CI 时跑，慢）

不需要聚合入口，Maven 自动发现。

### 7.4 端到端测试的数据库

- **默认 H2 内存库**（pom.xml 已引入）
- **关键路径**（涉及 H2 不支持的语法或特定索引）补 **Testcontainers + 真实 MySQL**

### 7.5 Object Mother 强制

**Entity 和领域对象在测试中使用时，必须通过对应的 `<Domain>TestFixtures` 类构造**。简单 DTO（字段少、一次性使用）不强制。

**位置与命名**：

```
src/test/java/com/<proj>/<domain>/fixtures/
└── <Domain>TestFixtures.java
```

**最低要求**：每个 Fixture 类至少提供一个"默认有效对象"方法：

```java
public class UserTestFixtures {
    public static UserEntity validUser() {
        UserEntity u = new UserEntity();
        u.setPhone("13800138000");
        u.setNickname("测试用户");
        u.setCreatedAt(Instant.now());
        return u;
    }

    public static UserEntity userWithPhone(String phone) {
        UserEntity u = validUser();
        u.setPhone(phone);
        return u;
    }
}
```

测试代码调用：

```java
@Test
void register_shouldFailIfPhoneAlreadyRegistered() {
    UserEntity existing = UserTestFixtures.userWithPhone("13800138000");
    when(repo.existsByPhone("13800138000")).thenReturn(true);

    assertThrows(BusinessException.class,
        () -> service.register("13800138000", "pwd", "昵称"));
}
```

### 7.6 Mock 工具

**Mockito**（Spring Boot Test Starter 自带），不引入其他 mock 库。

### 7.7 TDD 要求

引用全局 CLAUDE.md 的 **TDD 铁律**：有业务逻辑的代码必须先写失败测试，再写最小实现。

**强制覆盖范围**：
- Service 业务逻辑 → 单测必须有
- Controller → `@WebMvcTest` 集成测试必须有
- Repository 自定义查询 → `@DataJpaTest` 必须有
- 领域事件 Listener → 单测必须有

---

## 8. 基础能力层

### 8.1 构成（7 强制 + 1 可选）

| 编号 | 包名 | 级别 | 职责 |
|------|------|------|------|
| 1 | `<proj>-common-error` | 强制 | `BusinessException` + `ErrCode` + 冲突扫描 + 全局异常处理 |
| 2 | `<proj>-common-web` | 强制 | `BaseResp<T>` + 全局响应包装 + 参数校验 + 通用拦截器 |
| 3 | `<proj>-common-auth` | 强制 | JWT + 密码加密（BCrypt）+ 鉴权拦截器 + `CurrentUser` 上下文 + `@LoginRequired` 注解 |
| 4 | `<proj>-common-cache` | 强制 | Redis 封装（KV/Hash/List/分布式锁） |
| 5 | `<proj>-common-storage` | 强制 | 对象存储抽象（`ObjectStorage` 接口 + COS/OSS/S3 实现） |
| 6 | `<proj>-common-util` | 强制 | 通用工具（`DateUtil` / `StringUtil` / `JsonUtil` / `CryptoUtil` 等） |
| 7 | `<proj>-common-test` | 强制 | 测试基础（`BaseIntegrationTest` / `ArchUnit` 基类 / Fixture 父类） |
| 8 | `<proj>-common-ws` | 可选 | WebSocket 连接管理 + 在线状态 + 推送封装（有 WebSocket 需求的项目才建） |

"强制"的含义：**当项目需要该能力时，必须单拉一个基础模块，不允许把这类代码塞在业务模块或其他基础模块里**。对绝大多数业务后端，前 7 个是基本需求。

### 8.2 内部结构

初始阶段扁平组织：

```
<proj>-common-<role>/
└── src/main/java/com/<proj>/common/<role>/
    ├── XxxClass1.java
    ├── XxxClass2.java
    └── ...
```

同一基础能力下文件**超过 10 个**时再按子领域拆子包。不提前建空目录。

### 8.3 边界约束

- 基础能力**只提供通用抽象与工具**，**禁止**包含任何业务领域知识
- 基础能力之间可以互相依赖（典型依赖：`common-web` 依赖 `common-error`；`common-auth` 依赖 `common-error` + `common-cache`）
- **避免循环依赖**（`common-X` 依赖 `common-Y`，`common-Y` 不得反过来依赖 `common-X`）
- **禁止**基础能力依赖任何业务模块

### 8.4 不建议单独建的基础能力

| 能力 | 理由 |
|------|------|
| `common-event` | Spring 自带 `ApplicationEventPublisher` + `@TransactionalEventListener` 已够用 |
| `common-scheduler` | `@Scheduled` 内置够用，除非有复杂调度需求 |
| `common-mq` | 需要 Kafka/RabbitMQ 时再建；无 MQ 需求时不提前引入 |

---

## 9. 命名规范汇总表

### 9.1 模块与包

| 类别 | 模式 | 示例 |
|------|------|------|
| 业务领域契约模块 | `<proj>-<domain>-api` | `<proj>-user-api` |
| 业务领域实现模块 | `<proj>-<domain>-core` | `<proj>-user-core` |
| 基础能力模块 | `<proj>-common-<role>` | `<proj>-common-error` |
| Java 包根 | `com.<proj>.<domain>` / `com.<proj>.common.<role>` | `com.myapp.user` |
| core 子包 | `web/` `internal/` `api/` `event/` | — |
| api 子包 | `dto/` `event/` | — |

### 9.2 类命名

| 类型 | 位置 | 模式 | 示例 |
|------|------|------|------|
| 对内 API 接口 | `-api/` 顶层 | `<Domain>Api` | `UserApi` |
| 对内 API 实现 | `-core/api/` | `<Domain>ApiImpl` | `UserApiImpl` |
| HTTP Controller | `-core/web/` | `<Domain>Controller` | `UserController` |
| 业务 Service | `-core/internal/` | `<Domain><Action>Service` | `UserRegisterService` |
| Repository | `-core/internal/` | `<Domain>Repository` | `UserRepository` |
| JPA Entity | `-core/internal/` | `<Domain>Entity`（带后缀） | `UserEntity` |
| HTTP 请求 DTO | `-core/web/` | `<Action>Req` | `CreateUserReq` |
| HTTP 单接口响应 | `-core/web/`（与 Req 同文件） | `<Action>Data` | `LoginData` |
| HTTP 跨接口复用模型 | `-core/web/` | 业务描述名 | `UserProfile` |
| Java API DTO | `-api/dto/` | `<Domain>Dto` / `<Domain><描述>Dto` | `UserDto` / `UserSummaryDto` |
| 领域事件 | `-api/event/` | `<Domain><动作过去式>Event`（record） | `UserRegisteredEvent` |
| 本模块订阅其他模块事件的监听器 | `-core/event/` | `<Other><Event>EventListener` | `MessageSentEventListener` |
| 错误码 enum | `-core/internal/` 或 `common-error/` | `<Domain>ErrCode` | `UserErrCode` |
| 错误文案常量 | 同上 | enum 值（UPPER_SNAKE_CASE） | `PHONE_ALREADY_REGISTERED` |

### 9.3 测试与工具

| 类型 | 模式 | 示例 |
|------|------|------|
| 单元测试 | `<ClassUnderTest>Test` | `UserRegisterServiceTest` |
| 集成测试 | `<ClassUnderTest>IT` | `UserControllerIT` |
| Object Mother | `<Domain>TestFixtures` | `UserTestFixtures` |
| 工具类 | `<Role>Util`（单数） | `DateUtil` / `JsonUtil` |
| Spring 配置类 | `<Role>Config` | `RedisConfig` / `WebSocketConfig` |
| 常量类 | `<Role>Constants` | `MessageConstants` |

---

## 10. 代码模板速查

### 10.1 典型业务方法链路（注册用户）

```
HTTP 请求
    ↓
UserController.register(CreateUserReq req)
    ↓
UserRegisterService.register(phone, password, nickname)    [@Transactional]
    ↓
UserRepository.existsByPhone(phone)       →  走 JPA
UserEntity.<new>                          →  组装 Entity
UserRepository.save(entity)               →  落库
ApplicationEventPublisher.publishEvent(   →  发事件
    new UserRegisteredEvent(...))
    ↓
事务提交后 @Async 触发：
    WelcomeMailListener.onUserRegistered(event)
    StatsIncrementListener.onUserRegistered(event)
    ...
    ↓
返回 BaseResp.success(data)
```

### 10.2 典型跨模块调用（发消息前查好友关系）

```java
@Service
@RequiredArgsConstructor
public class MessageSendService {
    private final RelationApi relationApi;    // 跨领域依赖，来自 <proj>-relation-api
    private final UserApi userApi;            // 跨领域依赖，来自 <proj>-user-api
    private final MessageRepository repo;
    private final ApplicationEventPublisher events;

    @Transactional
    public Long send(Long fromId, Long toId, String content) {
        if (!relationApi.isFriend(fromId, toId)) {
            throw new BusinessException(MessageErrCode.NOT_FRIEND);
        }
        MessageEntity msg = new MessageEntity();
        msg.setFromId(fromId);
        msg.setToId(toId);
        msg.setContent(content);
        msg.setSentAt(Instant.now());
        repo.save(msg);

        events.publishEvent(new MessageSentEvent(msg.getId(), fromId, toId, Instant.now()));
        return msg.getId();
    }
}
```

### 10.3 典型单元测试（Mockito）

```java
@ExtendWith(MockitoExtension.class)
class UserRegisterServiceTest {
    @Mock UserRepository repo;
    @Mock BCryptPasswordEncoder passwordEncoder;
    @Mock ApplicationEventPublisher events;
    @InjectMocks UserRegisterService service;

    @Test
    void register_shouldFailIfPhoneAlreadyRegistered() {
        when(repo.existsByPhone("13800138000")).thenReturn(true);

        BusinessException ex = assertThrows(BusinessException.class,
            () -> service.register("13800138000", "pwd", "昵称"));

        assertEquals(UserErrCode.PHONE_ALREADY_REGISTERED, ex.getErrCode());
        verify(repo, never()).save(any());
        verify(events, never()).publishEvent(any());
    }

    @Test
    void register_shouldSaveAndPublishEvent() {
        when(repo.existsByPhone("13900139000")).thenReturn(false);
        when(passwordEncoder.encode("pwd")).thenReturn("hashed");
        when(repo.save(any())).thenAnswer(inv -> {
            UserEntity u = inv.getArgument(0);
            u.setId(42L);
            return u;
        });

        Long id = service.register("13900139000", "pwd", "测试");

        assertEquals(42L, id);
        verify(events).publishEvent(any(UserRegisteredEvent.class));
    }
}
```

### 10.4 典型集成测试（Controller）

```java
@WebMvcTest(UserController.class)
class UserControllerIT {
    @Autowired MockMvc mockMvc;
    @MockBean UserRegisterService registerService;

    @Test
    void register_shouldReturn200WithUserId() throws Exception {
        when(registerService.register(any(), any(), any())).thenReturn(42L);

        mockMvc.perform(post("/api/users/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"phone":"13800138000","password":"pwd","nickname":"测试"}
                    """))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.code").value(0))
            .andExpect(jsonPath("$.data.userId").value(42));
    }
}
```

---

## 11. 落地检查清单（从零新建项目时按序做）

- [ ] 创建父 POM：声明父 `spring-boot-starter-parent`，配 `maven-enforcer-plugin` 规则
- [ ] 创建 `bootstrap/` 启动工程模块
- [ ] 创建 `foundation_packages/` 下 7 个强制基础模块（error / web / auth / cache / storage / util / test）
- [ ] 在 `common-error` 里实现 `BusinessException` / `ErrCode` / `CommonErrCode` / `ErrCodeConflictDetector`
- [ ] 在 `common-web` 里实现 `BaseResp<T>` / `GlobalExceptionHandler`
- [ ] 在 `common-auth` 里实现 JWT 工具 / 鉴权拦截器 / `CurrentUser`
- [ ] 在 `common-test` 里实现 `BaseIntegrationTest` / ArchUnit 基类测试
- [ ] 在 `bootstrap/` 里配置 ArchUnit 跨模块依赖约束测试
- [ ] 创建业务模块时：**一律** `-api` + `-core` 双模块
- [ ] 新业务模块：在 `ERROR_CODE_REGISTRY.md` 登记本模块 code 区间
- [ ] 所有新代码走 TDD：先写失败测试 → 写实现 → 测试通过
- [ ] CI 配 `mvn verify`（跑 `*Test` + `*IT` + ArchUnit）

---

## 附录 A：与 app 侧架构规范的对应关系

| app 侧规则（flutter.md 体系） | 后端对应规则（本文档） |
|-----------------------------|----------------------|
| `business_packages/` + `foundation_packages/` 两层 | 同构（§1） |
| 业务模块的 `lib/<module>.dart` 门面 | `-api` 模块作为对外契约入口（§3.1） |
| 业务模块 `src/` 对外隐藏 | `-core/internal/` 对外隐藏（§3.2.3） |
| Page + Controller 必须配对 | Controller + Service 必须配对（§3.2） |
| `controller` 属性名固定 `c` | 每个 Service 独立类，不走属性命名约束（后端习惯） |
| `Obx` / `GetBuilder` / 禁 `setState` | 后端无 UI 状态，不适用 |
| 本地存储 GetStorage（禁 SharedPreferences） | 后端缓存 `common-cache`（Redis）（§8） |
| 网络请求 `.then()` + `parameters` | 后端是服务端，不发起 HTTP 调用（调用第三方 API 时另定规则） |
| `XxxReq` + `XxxData` 同文件 | 同构（§3.2.2） |
| 业务模块间禁止相互引用 | 同构（§4.1） |
| 跨模块跳转 `Get.toNamed` | 跨模块调用 `@Autowired XxxApi`（§4.2） |
| 测试聚合 `xxx_suite.dart` | 后端不需要（Maven 自动发现）（§7.3） |
| TDD 铁律 | 同构引用全局 CLAUDE.md（§7.7） |

---

## 附录 B：未来演进方向（非本轮实施）

- **拆微服务**：当单体性能/团队规模成瓶颈时，将某个领域的 `-core` 打包成独立 Spring Boot 应用，`-api` 升级为 gRPC / OpenAPI 契约。当前 `-api` + `-core` 双模块结构为此留好了迁移路径。
- **引入 MQ**（Kafka / RabbitMQ）：跨服务事件或需要持久化重试时引入。当前 Spring 本地事件满足需求。
- **升级 Java 21**：用 virtual threads 替代 `@Async` 线程池。届时领域事件订阅机制需重新评估。
- **DDD 战术分层升级**：如果某个业务领域的 `internal/` 变得复杂（聚合根行为丰富、领域规则多），再细分 `domain/` `application/` `infrastructure/`。当前扁平结构对大多数项目够用。

---

## 附录 C：配套的全局规则分片（计划）

本 spec 验收通过后，将提炼成 3 个分片规则文件，放在 `~/.claude/rules/`：

| 文件 | paths 触发条件 | 内容 |
|------|--------------|------|
| `java-backend.md` | `**/*.java` / `**/pom.xml` | 整体目录树 + 技术规范（类比 `flutter.md`） |
| `java-backend-business-layer.md` | `**/business_packages/**/*.java` | 业务领域模块（`-api` + `-core`）细则 |
| `java-backend-foundation-layer.md` | `**/foundation_packages/**/*.java` | 基础能力层 8 个包细则 |

本 spec 文档（本文件）作为**完整设计依据**保留在项目 `docs/superpowers/specs/` 下，将来修改规范时以本文档为准。

---

## 文档信息

- **作者**：brainstorming 协作产出（Aventador + Claude）
- **日期**：2026-04-25
- **状态**：待评审（spec 自审 → 用户审 → 进入 writing-plans）
- **后续步骤**：通过评审后，用 `writing-plans` skill 把本 spec 拆成可执行的实施计划（从零搭父 POM 开始，按 foundation 先行 → 业务领域后建 的顺序）
