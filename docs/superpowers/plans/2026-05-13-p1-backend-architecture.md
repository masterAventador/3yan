# P1 后端架构对齐实施 Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把后端 `sanyan-server` 从单 module 重构成 Maven 10 模块（1 bootstrap + 1 sanyan-business + 8 foundation），按 DDD 4 个领域（user/character/chat/llm）重组业务代码，引入 ErrCode/BusinessException 体系、Flyway 接管 schema、ArchUnit 守护规则，对齐 plan-1 设计。

**Architecture:** B 方案 —— foundation 8 包按规范拆，业务暂保留单 module 不拆 -api/-core。详见 spec `docs/superpowers/specs/2026-05-13-p1-backend-architecture-design.md`。

**Tech Stack:** Java 17 + Spring Boot 3.2.5 + Maven (multi-module) + PostgreSQL + Redis + Flyway + ArchUnit + JUnit 5 + Mockito + AssertJ

---

## 文件结构（创建 + 修改 + 删除 + 搬迁清单）

### Maven 拓扑（共 10 个子模块）

```
sanyan-server/  (父 POM, pom packaging)
├── pom.xml                                      ← 重写为父 POM
├── bootstrap/
│   ├── pom.xml                                  ← 新建
│   ├── src/main/java/com/sanyan/SanyanApplication.java   ← 搬
│   ├── src/main/resources/application.yml               ← 搬 + 加 flyway 配置
│   ├── src/main/resources/application-dev.yml           ← 搬 + ddl-auto 改 validate
│   ├── src/test/java/com/sanyan/SanyanApplicationTest.java ← 搬
│   └── src/test/java/com/sanyan/ArchitectureTest.java    ← 新建（M5）
├── business_packages/
│   └── sanyan-business/
│       ├── pom.xml                              ← 新建
│       └── src/main/java/com/sanyan/{user,character,chat,llm}/{web,ws,internal}/...
└── foundation_packages/
    ├── sanyan-common-error/   (BusinessException + ErrCode + CommonErrCode + ErrCodeConflictDetector + tests)
    ├── sanyan-common-web/     (BaseResp + GlobalExceptionHandler + tests)
    ├── sanyan-common-auth/    (JwtUtil + WebSocketInterceptor + LoginUserArgumentResolver + tests)
    ├── sanyan-common-cache/   (KvCache 封装 + tests)
    ├── sanyan-common-storage/ (空骨架：ObjectStorage interface)
    ├── sanyan-common-util/    (TextProcessor + TypingDelayCalculator + tests)
    ├── sanyan-common-test/    (ArchUnit 依赖声明)
    └── sanyan-common-ws/      (SessionManager + 通用 WS 协议 DTO + tests)
```

### sanyan-business 内部 4 领域包详细

```
sanyan-business/src/main/java/com/sanyan/
├── user/
│   ├── web/        AuthController + RegisterReq + LoginReq + SmsReq
│   └── internal/   UserRegisterService, UserLoginService, SmsCodeSendService,
│                   UserEntity, UserRepository, UserErrCode
├── character/
│   └── internal/   AiCharacterEntity, AiCharacterRepository, CharacterErrCode
├── chat/
│   ├── web/        MessageController
│   ├── ws/         ChatWebSocketHandler + WsAck + WsError + WsNewMessage +
│                   WsSyncResult + ChatWsConstants
│   └── internal/   MessageService, MessageEntity, MessageRepository,
│                   SenderType, ChatErrCode
└── llm/
    └── internal/   AiService, LlmErrCode

sanyan-business/src/test/java/com/sanyan/  (镜像 src/main 路径)
├── user/internal/   UserRegisterServiceTest, UserLoginServiceTest,
│                    SmsCodeSendServiceTest, UserTestFixtures
├── user/web/        AuthControllerIT
├── chat/internal/   MessageServiceSenderTypeTest, MessageServiceTransactionalTest,
│                    MessageTestFixtures
├── chat/ws/         WebSocketHandlerErrorTest
├── character/internal/  AiCharacterTestFixtures
└── llm/internal/    AiServiceTest

sanyan-business/src/main/resources/
├── db/migration/V1__initial_schema.sql     ← 新建（M4，pg_dump 来源）
├── db/migration/V2__seed_xiaowan.sql       ← 新建（M4，data.sql 内容）
└── prompts/xiaowan-system.md               ← 搬
```

### 要删除的文件

- `src/main/java/com/sanyan/exception/InvalidTokenException.java`（归并到 BusinessException）
- `src/main/java/com/sanyan/dto/ApiResponse.java`（被 common-web.BaseResp 替换）
- `src/main/resources/data.sql`（被 V2 migration 替换）
- 旧的 `src/main/java/com/sanyan/{auth,config,controller,dto,entity,exception,repository,service,util,websocket}/`（搬空后整体清除）
- 旧的 `src/test/java/com/sanyan/{auth,controller,exception,repository,service,util,websocket}/`（搬空后整体清除）
- `src/test/java/com/sanyan/repository/RepositoryTest.java` 内容拆到 `user.internal` 和 `chat.internal` 镜像位置

---

## 关键约定（执行时遵守）

1. **TDD 铁律**：每个有业务逻辑的代码改动，先写失败测试 → 跑确认 RED → 写最小实现 → 跑确认 GREEN → commit。重构（rename / move）不需要新写测试，但每步后必须 `mvn test` 确认现有测试不破。
2. **每个 task 完成后**：`mvn -pl <模块> test` 验证该模块测试全过 + commit。
3. **M3 是最大改动**：用 IntelliJ 的 Refactor → Move package + Rename Class 重构能力辅助。
4. **每个 milestone 完成后**：跑全量 `mvn test`、跑全量 `mvn verify`（M3 之后包含 `*IT`）、commit milestone summary。
5. **分支**：在 server 子模块的 `feat/p1-architecture` 上做。M5 完 + dogfood OK 后 merge dev → merge master。

---

## M1：Maven 骨架 + Enforcer（~半天，6 个 task）

### Task 1.0: 创建 feat/p1-architecture 分支

**Files:** (no file changes, branch only)

- [ ] **Step 1**: 在 server 子模块创建分支

```bash
cd /Users/aventador/code/3yan/server
git checkout -b feat/p1-architecture
git status  # 确认 working tree clean
```

- [ ] **Step 2**: 跑一次基线测试，记录当前测试数

```bash
mvn test | grep -E "Tests run:"
```

Expected: `Tests run: 194, Failures: 0, Errors: 0, Skipped: 0`

如果数字 != 194，记录实际数字（后续验收用）。

---

### Task 1.1: 备份现有 pom.xml + 把它转成父 POM

**Files:**
- Modify: `pom.xml`（重写为父 POM）
- Create: `pom.xml.legacy`（备份，最终删除）

- [ ] **Step 1**: 备份现状 pom

```bash
cp pom.xml pom.xml.legacy
```

- [ ] **Step 2**: 重写 `pom.xml` 为父 POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.sanyan</groupId>
    <artifactId>sanyan-server-parent</artifactId>
    <version>0.1.0</version>
    <packaging>pom</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
        <relativePath/>
    </parent>

    <modules>
        <module>foundation_packages/sanyan-common-error</module>
        <module>foundation_packages/sanyan-common-web</module>
        <module>foundation_packages/sanyan-common-auth</module>
        <module>foundation_packages/sanyan-common-cache</module>
        <module>foundation_packages/sanyan-common-storage</module>
        <module>foundation_packages/sanyan-common-util</module>
        <module>foundation_packages/sanyan-common-test</module>
        <module>foundation_packages/sanyan-common-ws</module>
        <module>business_packages/sanyan-business</module>
        <module>bootstrap</module>
    </modules>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- 内部模块统一版本 -->
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-error</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-web</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-auth</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-cache</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-storage</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-util</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-test</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-ws</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-business</artifactId><version>${project.version}</version></dependency>

            <!-- 第三方库锁版本（spring-boot-starter-parent 已管大部分） -->
            <dependency>
                <groupId>com.tngtech.archunit</groupId>
                <artifactId>archunit-junit5</artifactId>
                <version>1.2.1</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.flywaydb</groupId>
                <artifactId>flyway-database-postgresql</artifactId>
                <version>10.10.0</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-enforcer-plugin</artifactId>
                    <version>3.4.1</version>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-failsafe-plugin</artifactId>
                    <version>3.1.2</version>
                    <configuration>
                        <includes><include>**/*IT.java</include></includes>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>integration-test</goal>
                                <goal>verify</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

- [ ] **Step 3**: 此时 `mvn validate` 会失败（modules 还不存在）—— 这是预期的，跳过验证，进入下一个 task

---

### Task 1.2: 创建 bootstrap 模块骨架 + 搬 main + application yml

**Files:**
- Create: `bootstrap/pom.xml`
- Move: `src/main/java/com/sanyan/SanyanApplication.java` → `bootstrap/src/main/java/com/sanyan/SanyanApplication.java`
- Move: `src/main/resources/application.yml` → `bootstrap/src/main/resources/application.yml`
- Move: `src/main/resources/application-dev.yml` → `bootstrap/src/main/resources/application-dev.yml`
- Move: `src/test/java/com/sanyan/SanyanApplicationTest.java` → `bootstrap/src/test/java/com/sanyan/SanyanApplicationTest.java`

- [ ] **Step 1**: 创建 bootstrap 目录结构

```bash
mkdir -p bootstrap/src/main/java/com/sanyan
mkdir -p bootstrap/src/main/resources
mkdir -p bootstrap/src/test/java/com/sanyan
```

- [ ] **Step 2**: 创建 `bootstrap/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.sanyan</groupId>
        <artifactId>sanyan-server-parent</artifactId>
        <version>0.1.0</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>bootstrap</artifactId>
    <name>sanyan-server-bootstrap</name>

    <dependencies>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-business</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
    </dependencies>

    <build>
        <finalName>sanyan-server-${project.version}</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <executions><execution><goals><goal>repackage</goal></goals></execution></executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3**: 用 `git mv` 搬文件

```bash
git mv src/main/java/com/sanyan/SanyanApplication.java bootstrap/src/main/java/com/sanyan/SanyanApplication.java
git mv src/main/resources/application.yml bootstrap/src/main/resources/application.yml
git mv src/main/resources/application-dev.yml bootstrap/src/main/resources/application-dev.yml
git mv src/test/java/com/sanyan/SanyanApplicationTest.java bootstrap/src/test/java/com/sanyan/SanyanApplicationTest.java
```

- [ ] **Step 4**: 验证目录

```bash
ls bootstrap/src/main/java/com/sanyan/  # 应该有 SanyanApplication.java
ls bootstrap/src/main/resources/        # 应该有两个 yml
```

---

### Task 1.3: 创建 sanyan-business 模块骨架

**Files:**
- Create: `business_packages/sanyan-business/pom.xml`

- [ ] **Step 1**: 创建目录

```bash
mkdir -p business_packages/sanyan-business/src/main/java
mkdir -p business_packages/sanyan-business/src/main/resources
mkdir -p business_packages/sanyan-business/src/test/java
```

- [ ] **Step 2**: 创建 `business_packages/sanyan-business/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.sanyan</groupId>
        <artifactId>sanyan-server-parent</artifactId>
        <version>0.1.0</version>
        <relativePath>../../pom.xml</relativePath>
    </parent>

    <artifactId>sanyan-business</artifactId>
    <name>sanyan-business</name>

    <dependencies>
        <!-- 业务模块按需依赖 foundation -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-error</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-web</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-auth</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-cache</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-util</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-ws</artifactId></dependency>

        <!-- Spring Boot 业务必备 -->
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-redis</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-websocket</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-validation</artifactId></dependency>
        <dependency><groupId>org.postgresql</groupId><artifactId>postgresql</artifactId><scope>runtime</scope></dependency>
        <dependency><groupId>org.flywaydb</groupId><artifactId>flyway-core</artifactId></dependency>
        <dependency><groupId>org.flywaydb</groupId><artifactId>flyway-database-postgresql</artifactId></dependency>
        <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId></dependency>

        <!-- 测试 -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>com.h2database</groupId><artifactId>h2</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.awaitility</groupId><artifactId>awaitility</artifactId><scope>test</scope></dependency>
    </dependencies>
</project>
```

---

### Task 1.4: 创建 8 个 foundation_packages 模块骨架

**Files:** 8 个 pom.xml + 8 个 src/main/java/com/sanyan/common/{role}/ 目录

**模板**（替换 `<role>`）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.sanyan</groupId>
        <artifactId>sanyan-server-parent</artifactId>
        <version>0.1.0</version>
        <relativePath>../../pom.xml</relativePath>
    </parent>
    <artifactId>sanyan-common-<role></artifactId>

    <build>
        <plugins>
            <!-- foundation 模块禁止反向依赖 business -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-enforcer-plugin</artifactId>
                <executions>
                    <execution>
                        <id>enforce-no-business-dep</id>
                        <goals><goal>enforce</goal></goals>
                        <configuration>
                            <rules>
                                <bannedDependencies>
                                    <excludes>
                                        <exclude>com.sanyan:sanyan-business</exclude>
                                    </excludes>
                                </bannedDependencies>
                            </rules>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 1**: 创建 8 个模块目录 + 各自 pom.xml（按模板替换 role）

```bash
for ROLE in error web auth cache storage util test ws; do
    mkdir -p foundation_packages/sanyan-common-$ROLE/src/main/java/com/sanyan/common/$ROLE
    mkdir -p foundation_packages/sanyan-common-$ROLE/src/test/java/com/sanyan/common/$ROLE
done
```

- [ ] **Step 2**: 为每个 foundation 模块创建 pom.xml，按上述模板 + 下面表格的差异添加 dependencies：

| 模块 | 额外 dependencies |
|---|---|
| sanyan-common-error | spring-boot-starter（Bean 扫描）、lombok |
| sanyan-common-web | sanyan-common-error、spring-boot-starter-web、jackson-databind、lombok |
| sanyan-common-auth | sanyan-common-error、sanyan-common-cache、spring-boot-starter-web、spring-boot-starter-data-redis、io.jsonwebtoken:jjwt-api/jjwt-impl/jjwt-jackson:0.12.5、spring-security-crypto |
| sanyan-common-cache | spring-boot-starter-data-redis、lombok |
| sanyan-common-storage | 仅 spring-boot-starter（接口骨架）、lombok |
| sanyan-common-util | 仅 lombok |
| sanyan-common-test | scope=compile 的 archunit-junit5、spring-boot-starter-test |
| sanyan-common-ws | sanyan-common-error、sanyan-common-cache、spring-boot-starter-websocket、jackson-databind、lombok |

- [ ] **Step 3**: 验证目录完整

```bash
ls foundation_packages/  # 应该有 8 个子目录
find foundation_packages -name pom.xml | wc -l  # = 8
```

---

### Task 1.5: 验证 Maven 多模块骨架编译通过

- [ ] **Step 1**: 跑 `mvn validate`

```bash
mvn validate
```

Expected: `BUILD SUCCESS`（Enforcer 还没生效，因为所有模块都是空的）

- [ ] **Step 2**: 跑 `mvn compile`

```bash
mvn compile
```

Expected: `BUILD SUCCESS`（所有模块都没源代码所以也都能编译）

- [ ] **Step 3**: 跑 `mvn test`（这时 SanyanApplicationTest 在 bootstrap 但 bootstrap 还不能依赖到任何代码）

```bash
mvn test
```

Expected: 大概率失败（SanyanApplicationTest 找不到 controller / service / 等 class 因为它们还在旧 src/）—— **这是预期**，M3 之后才会再过。本 M1 阶段验收点是 compile 过即可。

- [ ] **Step 4**: 删除 `pom.xml.legacy`（M1 不再回滚）

```bash
rm pom.xml.legacy
```

- [ ] **Step 5**: commit M1

```bash
git add -A
git commit -m "feat(server): M1 Maven 多模块骨架 + Enforcer banned-dependencies"
```

---

## M2：foundation_packages 充实（~1 天，8 个 task）

### Task 2.1: common-error —— BusinessException + ErrCode 接口 + CommonErrCode + Detector

**Files:**
- Create: `foundation_packages/sanyan-common-error/src/main/java/com/sanyan/common/error/ErrCode.java`
- Create: `foundation_packages/sanyan-common-error/src/main/java/com/sanyan/common/error/BusinessException.java`
- Create: `foundation_packages/sanyan-common-error/src/main/java/com/sanyan/common/error/CommonErrCode.java`
- Create: `foundation_packages/sanyan-common-error/src/main/java/com/sanyan/common/error/ErrCodeConflictDetector.java`
- Create: `foundation_packages/sanyan-common-error/src/test/java/com/sanyan/common/error/ErrCodeConflictDetectorTest.java`

- [ ] **Step 1**: 写 `ErrCodeConflictDetectorTest` 先（TDD RED）

```java
package com.sanyan.common.error;

import org.junit.jupiter.api.Test;
import org.springframework.boot.ApplicationArguments;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ErrCodeConflictDetectorTest {

    enum ErrA implements ErrCode {
        A_ONE; @Override public int getCode() { return 100; } @Override public String getDefaultMessage() { return "a"; }
    }
    enum ErrB implements ErrCode {
        B_ONE; @Override public int getCode() { return 100; } @Override public String getDefaultMessage() { return "b"; }
    }
    enum ErrC implements ErrCode {
        C_ONE; @Override public int getCode() { return 200; } @Override public String getDefaultMessage() { return "c"; }
    }

    @Test
    void detect_noConflict_passes() {
        new ErrCodeConflictDetector(java.util.List.of(ErrA.class, ErrC.class))
            .run((ApplicationArguments) null);
    }

    @Test
    void detect_conflict_throws() {
        assertThatThrownBy(() ->
            new ErrCodeConflictDetector(java.util.List.of(ErrA.class, ErrB.class))
                .run((ApplicationArguments) null))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("ErrCode 冲突").hasMessageContaining("100");
    }
}
```

- [ ] **Step 2**: 跑测试看 RED

```bash
mvn -pl foundation_packages/sanyan-common-error test
```

Expected: 编译失败（ErrCode/BusinessException/Detector 都还没建）

- [ ] **Step 3**: 写最小实现让测试 GREEN

文件 `ErrCode.java`:
```java
package com.sanyan.common.error;

public interface ErrCode {
    int getCode();
    String getDefaultMessage();
}
```

文件 `BusinessException.java`:
```java
package com.sanyan.common.error;

import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {
    private final ErrCode errCode;

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

文件 `CommonErrCode.java`:
```java
package com.sanyan.common.error;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public enum CommonErrCode implements ErrCode {
    TOKEN_EXPIRED(400, "登录态过期"),
    TOKEN_INVALID(401, "登录态无效"),
    FORBIDDEN(403, "无权限"),
    NOT_FOUND(404, "资源不存在"),
    PARAM_INVALID(410, "参数错误"),
    INTERNAL_ERROR(500, "服务器错误，请稍后重试");

    private final int code;
    private final String defaultMessage;
}
```

文件 `ErrCodeConflictDetector.java`:
```java
package com.sanyan.common.error;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Slf4j
@Component
@RequiredArgsConstructor
public class ErrCodeConflictDetector implements ApplicationRunner {

    private final List<Class<? extends ErrCode>> errCodeEnums;

    @Override
    public void run(ApplicationArguments args) {
        Map<Integer, String> seen = new HashMap<>();
        for (Class<? extends ErrCode> clazz : errCodeEnums) {
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
        log.info("ErrCode 冲突扫描通过，共 {} 个 enum，{} 个码", errCodeEnums.size(), seen.size());
    }
}
```

- [ ] **Step 4**: 加 ErrCodeEnumsAutoConfiguration（自动收集所有 implements ErrCode 的 enum）

文件 `foundation_packages/sanyan-common-error/src/main/java/com/sanyan/common/error/ErrCodeAutoConfig.java`:
```java
package com.sanyan.common.error;

import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.core.io.support.ResourcePatternResolver;
import org.springframework.core.type.classreading.CachingMetadataReaderFactory;
import org.springframework.core.type.classreading.MetadataReader;

import java.util.ArrayList;
import java.util.List;

@Configuration
public class ErrCodeAutoConfig {

    @Bean
    public List<Class<? extends ErrCode>> errCodeEnums() throws Exception {
        List<Class<? extends ErrCode>> result = new ArrayList<>();
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        CachingMetadataReaderFactory factory = new CachingMetadataReaderFactory(resolver);
        for (var res : resolver.getResources("classpath*:com/sanyan/**/*.class")) {
            MetadataReader reader = factory.getMetadataReader(res);
            if (!reader.getClassMetadata().isFinal()) continue;  // enum 都是 final
            Class<?> clazz;
            try { clazz = Class.forName(reader.getClassMetadata().getClassName()); }
            catch (Throwable e) { continue; }
            if (clazz.isEnum() && ErrCode.class.isAssignableFrom(clazz)) {
                result.add((Class<? extends ErrCode>) clazz);
            }
        }
        return result;
    }
}
```

- [ ] **Step 5**: 加 META-INF/spring 自动配置注册

文件 `foundation_packages/sanyan-common-error/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:
```
com.sanyan.common.error.ErrCodeAutoConfig
```

- [ ] **Step 6**: 跑测试看 GREEN

```bash
mvn -pl foundation_packages/sanyan-common-error test
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`

- [ ] **Step 7**: commit

```bash
git add foundation_packages/sanyan-common-error/
git commit -m "feat(common-error): BusinessException + ErrCode + CommonErrCode + 冲突扫描"
```

---

### Task 2.2: common-web —— BaseResp + GlobalExceptionHandler

**Files:**
- Create: `foundation_packages/sanyan-common-web/src/main/java/com/sanyan/common/web/BaseResp.java`
- Create: `foundation_packages/sanyan-common-web/src/main/java/com/sanyan/common/web/GlobalExceptionHandler.java`
- Create: `foundation_packages/sanyan-common-web/src/test/java/com/sanyan/common/web/BaseRespTest.java`
- Create: `foundation_packages/sanyan-common-web/src/test/java/com/sanyan/common/web/GlobalExceptionHandlerTest.java`

- [ ] **Step 1**: 写 `BaseRespTest`（RED）—— 验证 success / failed 工厂方法语义

```java
package com.sanyan.common.web;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class BaseRespTest {
    @Test
    void success_returnsResponseWithDataAndZeroCode() {
        BaseResp<String> r = BaseResp.success("hello");
        assertThat(r.isSuccess()).isTrue();
        assertThat(r.getCode()).isEqualTo(0);
        assertThat(r.getMessage()).isEqualTo("ok");
        assertThat(r.getData()).isEqualTo("hello");
    }

    @Test
    void failed_returnsResponseWithCodeAndMessageAndNullData() {
        BaseResp<Void> r = BaseResp.failed(1003, "密码错误");
        assertThat(r.isSuccess()).isFalse();
        assertThat(r.getCode()).isEqualTo(1003);
        assertThat(r.getMessage()).isEqualTo("密码错误");
        assertThat(r.getData()).isNull();
    }
}
```

- [ ] **Step 2**: 跑测试 RED；然后实现 `BaseResp.java`:

```java
package com.sanyan.common.web;

import lombok.Data;

@Data
public class BaseResp<T> {
    public static final int SUCCESS_CODE = 0;
    public static final String SUCCESS_MESSAGE = "ok";

    private boolean success;
    private int code;
    private String message;
    private T data;

    public static <T> BaseResp<T> success(T data) {
        BaseResp<T> r = new BaseResp<>();
        r.success = true; r.code = SUCCESS_CODE; r.message = SUCCESS_MESSAGE; r.data = data;
        return r;
    }

    public static <T> BaseResp<T> failed(int code, String message) {
        BaseResp<T> r = new BaseResp<>();
        r.success = false; r.code = code; r.message = message;
        return r;
    }
}
```

- [ ] **Step 3**: 跑 BaseResp 测试 GREEN

- [ ] **Step 4**: 写 `GlobalExceptionHandlerTest`（直接调 handler 方法，不启 Spring）

```java
package com.sanyan.common.web;

import com.sanyan.common.error.BusinessException;
import com.sanyan.common.error.CommonErrCode;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class GlobalExceptionHandlerTest {

    private final GlobalExceptionHandler handler = new GlobalExceptionHandler();

    @Test
    void onBusinessException_returnsFailedRespWithErrCode() {
        BaseResp<Void> r = handler.onBusinessException(new BusinessException(CommonErrCode.TOKEN_INVALID));
        assertThat(r.isSuccess()).isFalse();
        assertThat(r.getCode()).isEqualTo(401);
        assertThat(r.getMessage()).isEqualTo("登录态无效");
    }

    @Test
    void onUnknown_returnsInternalErrorResp() {
        BaseResp<Void> r = handler.onUnknown(new RuntimeException("oops"));
        assertThat(r.isSuccess()).isFalse();
        assertThat(r.getCode()).isEqualTo(500);
    }
}
```

- [ ] **Step 5**: 实现 `GlobalExceptionHandler.java`:

```java
package com.sanyan.common.web;

import com.sanyan.common.error.BusinessException;
import com.sanyan.common.error.CommonErrCode;
import lombok.extern.slf4j.Slf4j;
import org.springframework.validation.BindException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public BaseResp<Void> onBusinessException(BusinessException e) {
        return BaseResp.failed(e.getErrCode().getCode(), e.getMessage());
    }

    @ExceptionHandler({MethodArgumentNotValidException.class, BindException.class})
    public BaseResp<Void> onValidation(Exception e) {
        String msg = "参数错误";
        if (e instanceof MethodArgumentNotValidException ex) {
            var err = ex.getBindingResult().getAllErrors().get(0);
            msg = err.getDefaultMessage();
        }
        return BaseResp.failed(CommonErrCode.PARAM_INVALID.getCode(), msg);
    }

    @ExceptionHandler(Exception.class)
    public BaseResp<Void> onUnknown(Exception e) {
        log.error("unhandled exception", e);
        return BaseResp.failed(CommonErrCode.INTERNAL_ERROR.getCode(), "服务器错误，请稍后重试");
    }
}
```

- [ ] **Step 6**: 跑测试 GREEN

```bash
mvn -pl foundation_packages/sanyan-common-web test
```

Expected: `Tests run: 4, Failures: 0`

- [ ] **Step 7**: commit

```bash
git add foundation_packages/sanyan-common-web/
git commit -m "feat(common-web): BaseResp + GlobalExceptionHandler"
```

---

### Task 2.3: common-auth —— 搬 JwtUtil + WebSocketInterceptor + LoginUserArgumentResolver

**Files:**
- Move: `src/main/java/com/sanyan/util/JwtUtil.java` → `foundation_packages/sanyan-common-auth/src/main/java/com/sanyan/common/auth/JwtUtil.java`
- Move: `src/main/java/com/sanyan/websocket/WebSocketInterceptor.java` → `foundation_packages/sanyan-common-auth/src/main/java/com/sanyan/common/auth/WebSocketInterceptor.java`（注意：原文件在 websocket/，搬到 common.auth 是因为它是鉴权拦截器）
- Move: `src/main/java/com/sanyan/auth/LoginUserArgumentResolver.java` → `foundation_packages/sanyan-common-auth/src/main/java/com/sanyan/common/auth/LoginUserArgumentResolver.java`
- Move: `src/main/java/com/sanyan/auth/LoginUser.java`（如有）→ `foundation_packages/sanyan-common-auth/src/main/java/com/sanyan/common/auth/LoginUser.java`
- Move: 测试同步搬

- [ ] **Step 1**: 用 `git mv` 搬源文件

```bash
git mv src/main/java/com/sanyan/util/JwtUtil.java \
    foundation_packages/sanyan-common-auth/src/main/java/com/sanyan/common/auth/JwtUtil.java
git mv src/main/java/com/sanyan/auth/LoginUser.java \
    foundation_packages/sanyan-common-auth/src/main/java/com/sanyan/common/auth/LoginUser.java
git mv src/main/java/com/sanyan/auth/LoginUserArgumentResolver.java \
    foundation_packages/sanyan-common-auth/src/main/java/com/sanyan/common/auth/LoginUserArgumentResolver.java
git mv src/main/java/com/sanyan/websocket/WebSocketInterceptor.java \
    foundation_packages/sanyan-common-auth/src/main/java/com/sanyan/common/auth/WebSocketInterceptor.java
```

- [ ] **Step 2**: 每个文件改 `package com.sanyan.common.auth;`

- [ ] **Step 3**: JwtUtil 内部抛的 `InvalidTokenException` 改成 `throw new BusinessException(CommonErrCode.TOKEN_INVALID)` —— 加 import `com.sanyan.common.error.BusinessException` + `com.sanyan.common.error.CommonErrCode`

- [ ] **Step 4**: 搬测试

```bash
git mv src/test/java/com/sanyan/util/JwtUtilTest.java \
    foundation_packages/sanyan-common-auth/src/test/java/com/sanyan/common/auth/JwtUtilTest.java
git mv src/test/java/com/sanyan/auth/LoginUserArgumentResolverTest.java \
    foundation_packages/sanyan-common-auth/src/test/java/com/sanyan/common/auth/LoginUserArgumentResolverTest.java
```

- [ ] **Step 5**: 改测试 package 声明 + import

- [ ] **Step 6**: 跑测试

```bash
mvn -pl foundation_packages/sanyan-common-auth test
```

Expected: 之前 JwtUtilTest 的 6 个 + LoginUserArgumentResolverTest 的 4 个 = 10 个，全过

- [ ] **Step 7**: commit

```bash
git add -A
git commit -m "feat(common-auth): 搬 JwtUtil + LoginUser + WebSocketInterceptor 到 common-auth"
```

---

### Task 2.4: common-cache —— 搬 SessionManager + 加 KvCache 封装

**Files:**
- Move: `src/main/java/com/sanyan/websocket/SessionManager.java` → 暂时搬到 `foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/SessionManager.java`（**注意**：SessionManager 既管 WS session 又用 Redis 存映射，按职责该在 common-ws 而非 common-cache）
- Create: `foundation_packages/sanyan-common-cache/src/main/java/com/sanyan/common/cache/KvCache.java`
- Create: `foundation_packages/sanyan-common-cache/src/test/java/com/sanyan/common/cache/KvCacheTest.java`

**说明**：现有的 SessionManager 实际归 common-ws（WS session 管理）而非 common-cache。这里 Task 2.4 只创建 KvCache 封装；SessionManager 在 Task 2.7 处理。

- [ ] **Step 1**: 写 `KvCacheTest`（RED）

```java
package com.sanyan.common.cache;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;

import java.time.Duration;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class KvCacheTest {

    @Mock StringRedisTemplate redis;
    @Mock ValueOperations<String, String> ops;

    @Test
    void set_delegatesToRedisWithExpiry() {
        when(redis.opsForValue()).thenReturn(ops);
        KvCache cache = new KvCache(redis);

        cache.set("foo", "bar", Duration.ofMinutes(5));

        verify(ops).set("foo", "bar", Duration.ofMinutes(5));
    }

    @Test
    void get_delegatesToRedis() {
        when(redis.opsForValue()).thenReturn(ops);
        when(ops.get("foo")).thenReturn("bar");
        KvCache cache = new KvCache(redis);

        assertThat(cache.get("foo")).isEqualTo("bar");
    }

    @Test
    void delete_delegatesToRedis() {
        KvCache cache = new KvCache(redis);
        cache.delete("foo");
        verify(redis).delete("foo");
    }
}
```

- [ ] **Step 2**: 跑 RED → 实现 `KvCache.java`:

```java
package com.sanyan.common.cache;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Component
@RequiredArgsConstructor
public class KvCache {
    private final StringRedisTemplate redis;

    public void set(String key, String value, Duration ttl) {
        redis.opsForValue().set(key, value, ttl);
    }

    public String get(String key) {
        return redis.opsForValue().get(key);
    }

    public void delete(String key) {
        redis.delete(key);
    }

    public boolean exists(String key) {
        return Boolean.TRUE.equals(redis.hasKey(key));
    }
}
```

- [ ] **Step 3**: 跑测试 GREEN

```bash
mvn -pl foundation_packages/sanyan-common-cache test
```

- [ ] **Step 4**: commit

```bash
git add foundation_packages/sanyan-common-cache/
git commit -m "feat(common-cache): KvCache 封装 Redis kv 操作"
```

---

### Task 2.5: common-util —— 搬 TextProcessor + TypingDelayCalculator

**Files:**
- Move: `src/main/java/com/sanyan/util/TextProcessor.java` → `foundation_packages/sanyan-common-util/src/main/java/com/sanyan/common/util/TextProcessor.java`
- Move: `src/main/java/com/sanyan/util/TypingDelayCalculator.java` → `foundation_packages/sanyan-common-util/src/main/java/com/sanyan/common/util/TypingDelayCalculator.java`
- Move: 对应测试

- [ ] **Step 1**: git mv 4 个文件

```bash
git mv src/main/java/com/sanyan/util/TextProcessor.java \
    foundation_packages/sanyan-common-util/src/main/java/com/sanyan/common/util/TextProcessor.java
git mv src/main/java/com/sanyan/util/TypingDelayCalculator.java \
    foundation_packages/sanyan-common-util/src/main/java/com/sanyan/common/util/TypingDelayCalculator.java
git mv src/test/java/com/sanyan/util/TextProcessorTest.java \
    foundation_packages/sanyan-common-util/src/test/java/com/sanyan/common/util/TextProcessorTest.java
git mv src/test/java/com/sanyan/service/MessageServiceTimingTest.java \
    foundation_packages/sanyan-common-util/src/test/java/com/sanyan/common/util/TypingDelayCalculatorTest.java
```

注意：MessageServiceTimingTest 其实测的是 TypingDelayCalculator，**改名 + 改 package**。

- [ ] **Step 2**: 改 package 声明 + import

所有搬过去的文件第一行改成 `package com.sanyan.common.util;`。TypingDelayCalculatorTest 内部 `import com.sanyan.util.TypingDelayCalculator;` 改成 `import com.sanyan.common.util.TypingDelayCalculator;`。

- [ ] **Step 3**: 跑测试

```bash
mvn -pl foundation_packages/sanyan-common-util test
```

Expected: `Tests run: 161, Failures: 0` (10 TextProcessor + 151 TypingDelay)

- [ ] **Step 4**: commit

```bash
git add -A
git commit -m "feat(common-util): 搬 TextProcessor + TypingDelayCalculator 到 common-util"
```

---

### Task 2.6: common-storage —— 空骨架 + ObjectStorage 接口

**Files:**
- Create: `foundation_packages/sanyan-common-storage/src/main/java/com/sanyan/common/storage/ObjectStorage.java`

- [ ] **Step 1**: 创建 `ObjectStorage.java`:

```java
package com.sanyan.common.storage;

import java.io.InputStream;

public interface ObjectStorage {
    String upload(String key, InputStream input, String contentType);
    InputStream download(String key);
    String presignedUrl(String key, long expireSeconds);
    void delete(String key);
}
```

- [ ] **Step 2**: 此模块 P1 不引入实现类（没真实存储需求），仅留接口骨架供未来扩展

- [ ] **Step 3**: commit

```bash
git add foundation_packages/sanyan-common-storage/
git commit -m "feat(common-storage): ObjectStorage 接口骨架"
```

---

### Task 2.7: common-ws —— 搬 SessionManager + 通用 WS DTO

**Files:**
- Move: `src/main/java/com/sanyan/websocket/SessionManager.java` → `foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/SessionManager.java`
- Move: `src/main/java/com/sanyan/dto/ws/WsMessage.java` → `foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/WsMessage.java`
- Move: `src/main/java/com/sanyan/dto/ws/WsEventType.java` → `foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/WsEventType.java`
- Move: `src/main/java/com/sanyan/dto/ws/WsTyping.java` → `foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/WsTyping.java`
- Move: 对应测试 `SessionManagerTest.java`

**注意**：业务相关 WS DTO（WsNewMessage / WsSyncResult / WsAck / WsError / WsErrorMessage / SenderType）**留在原位**等 M3 搬到 chat.ws 包

- [ ] **Step 1**: git mv 5 个文件

```bash
git mv src/main/java/com/sanyan/websocket/SessionManager.java \
    foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/SessionManager.java
git mv src/main/java/com/sanyan/dto/ws/WsMessage.java \
    foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/WsMessage.java
git mv src/main/java/com/sanyan/dto/ws/WsEventType.java \
    foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/WsEventType.java
git mv src/main/java/com/sanyan/dto/ws/WsTyping.java \
    foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/WsTyping.java
git mv src/test/java/com/sanyan/websocket/SessionManagerTest.java \
    foundation_packages/sanyan-common-ws/src/test/java/com/sanyan/common/ws/SessionManagerTest.java
```

- [ ] **Step 2**: 改这 5 个文件的 package 声明为 `com.sanyan.common.ws`

- [ ] **Step 3**: 跑测试

```bash
mvn -pl foundation_packages/sanyan-common-ws test
```

Expected: `Tests run: 3, Failures: 0`（SessionManagerTest）

- [ ] **Step 4**: commit

```bash
git add -A
git commit -m "feat(common-ws): 搬 SessionManager + 通用 WS 协议 DTO（WsMessage/WsEventType/WsTyping）"
```

---

### Task 2.8: common-test —— ArchUnit 依赖声明

**Files:**
- Modify: `foundation_packages/sanyan-common-test/pom.xml`（如果 Task 1.4 已加 archunit 依赖，跳过）

- [ ] **Step 1**: 确保 `foundation_packages/sanyan-common-test/pom.xml` 含：

```xml
<dependencies>
    <dependency>
        <groupId>com.tngtech.archunit</groupId>
        <artifactId>archunit-junit5</artifactId>
        <scope>compile</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>compile</scope>
    </dependency>
</dependencies>
```

scope 用 compile 是为了让依赖此模块的其他模块（在 test scope 中）能传递使用 ArchUnit + spring-test。

- [ ] **Step 2**: 跑

```bash
mvn -pl foundation_packages/sanyan-common-test compile
```

Expected: BUILD SUCCESS

- [ ] **Step 3**: commit

```bash
git add foundation_packages/sanyan-common-test/
git commit -m "feat(common-test): 声明 ArchUnit + spring-test 依赖"
```

---

### M2 Verification

跑全量 foundation 测试：

```bash
mvn -pl foundation_packages/sanyan-common-error,foundation_packages/sanyan-common-web,foundation_packages/sanyan-common-auth,foundation_packages/sanyan-common-cache,foundation_packages/sanyan-common-util,foundation_packages/sanyan-common-ws test
```

Expected: 所有 foundation 模块测试通过（≈170+ 个测试）。

---

## M3：业务搬家 + 命名/Service 规范化 + 测试搬家（~1-2 天，10 个 task）

> M3 是最大改动。建议每个领域单独 commit，分批跑 mvn compile 验证。强烈建议用 IntelliJ 的 Refactor → Move Class + Refactor → Rename 自动改 import。

### Task 3.1: 搬 chat 领域 (web + ws + internal) + Entity 加后缀

**Files:**
- Move + Rename: `src/main/java/com/sanyan/entity/Message.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageEntity.java`（同步加 `@Table(name = "message")` 保表名不变）
- Move: `src/main/java/com/sanyan/repository/MessageRepository.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageRepository.java`（泛型参数改成 `MessageEntity`）
- Move: `src/main/java/com/sanyan/service/MessageService.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageService.java`
- Move: `src/main/java/com/sanyan/controller/MessageController.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/chat/web/MessageController.java`
- Move: `src/main/java/com/sanyan/websocket/WebSocketHandler.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java`（重命名）
- Move: 业务相关 WS DTO: `WsAck/WsError/WsErrorMessage/WsNewMessage/WsSyncResult/SenderType` → `business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws/`（SenderType 严格说应放 internal 但属业务常量）
- Create: `business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/ChatErrCode.java`
- Move: 测试同步

- [ ] **Step 1**: 创建目标目录

```bash
mkdir -p business_packages/sanyan-business/src/main/java/com/sanyan/chat/{web,ws,internal}
mkdir -p business_packages/sanyan-business/src/test/java/com/sanyan/chat/{web,ws,internal}
```

- [ ] **Step 2**: git mv + rename Message → MessageEntity

```bash
git mv src/main/java/com/sanyan/entity/Message.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageEntity.java
```

修改文件：
1. `package com.sanyan.chat.internal;`
2. `public class Message` → `public class MessageEntity`
3. `@Table(name = "message", ...)` 保持表名 `"message"` 不变

- [ ] **Step 3**: 搬 MessageRepository + 改泛型

```bash
git mv src/main/java/com/sanyan/repository/MessageRepository.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageRepository.java
```

修改：`package com.sanyan.chat.internal;`、`JpaRepository<Message, Long>` → `JpaRepository<MessageEntity, Long>`，所有方法签名的 `Message` 改 `MessageEntity`。

- [ ] **Step 4**: 创建 `ChatErrCode.java`:

```java
package com.sanyan.chat.internal;

import com.sanyan.common.error.ErrCode;
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public enum ChatErrCode implements ErrCode {
    MESSAGE_PROCESSING_FAILED(2001, "消息处理失败");

    private final int code;
    private final String defaultMessage;
}
```

- [ ] **Step 5**: 搬 MessageService + 改 import

```bash
git mv src/main/java/com/sanyan/service/MessageService.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageService.java
```

修改：
- `package com.sanyan.chat.internal;`
- `import com.sanyan.entity.Message;` → 删（同包）
- `import com.sanyan.entity.AiCharacter;` → `import com.sanyan.character.internal.AiCharacterEntity;`（M3.2 后才存在；暂时保留旧 import，等 character 搬完）
- `import com.sanyan.dto.ws.SenderType;` → 改成同包 import 或保留 `com.sanyan.chat.ws.SenderType`（取决于 SenderType 放哪）
- `import com.sanyan.repository.MessageRepository;` → 删（同包）
- `import com.sanyan.repository.AiCharacterRepository;` → 改为 `com.sanyan.character.internal.AiCharacterRepository`
- `import com.sanyan.util.TextProcessor;` → `import com.sanyan.common.util.TextProcessor;`
- `import com.sanyan.dto.data.MessageData;` → 留位（M3.7 改）
- `Message` → `MessageEntity`（所有出现处，包括字段、参数、return type）
- `RuntimeException("默认角色不存在...")` → `new BusinessException(CharacterErrCode.CHARACTER_NOT_FOUND)`（暂留待 character 搬完）

- [ ] **Step 6**: 搬 MessageController

```bash
git mv src/main/java/com/sanyan/controller/MessageController.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/web/MessageController.java
```

修改：
- `package com.sanyan.chat.web;`
- import 同步更新

- [ ] **Step 7**: 搬 WebSocketHandler → ChatWebSocketHandler

```bash
git mv src/main/java/com/sanyan/websocket/WebSocketHandler.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws/ChatWebSocketHandler.java
```

修改：
- `package com.sanyan.chat.ws;`
- 类名 `WebSocketHandler` → `ChatWebSocketHandler`
- import 同步更新

注意：原文件可能引用 SessionManager（已在 common.ws）、MessageService（同业务模块）、WS DTO（要么 common.ws 要么 chat.ws）。

- [ ] **Step 8**: 搬业务 WS DTO 到 chat.ws

```bash
git mv src/main/java/com/sanyan/dto/ws/WsAck.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws/WsAck.java
git mv src/main/java/com/sanyan/dto/ws/WsError.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws/WsError.java
git mv src/main/java/com/sanyan/dto/ws/WsErrorMessage.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws/WsErrorMessage.java
git mv src/main/java/com/sanyan/dto/ws/WsNewMessage.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws/WsNewMessage.java
git mv src/main/java/com/sanyan/dto/ws/WsSyncResult.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws/WsSyncResult.java
git mv src/main/java/com/sanyan/dto/ws/SenderType.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws/SenderType.java
```

修改每个文件：`package com.sanyan.chat.ws;`

- [ ] **Step 9**: 搬 chat 相关测试

```bash
git mv src/test/java/com/sanyan/service/MessageServiceSenderTypeTest.java \
    business_packages/sanyan-business/src/test/java/com/sanyan/chat/internal/MessageServiceSenderTypeTest.java
git mv src/test/java/com/sanyan/service/MessageServiceTransactionalTest.java \
    business_packages/sanyan-business/src/test/java/com/sanyan/chat/internal/MessageServiceTransactionalTest.java
git mv src/test/java/com/sanyan/websocket/WebSocketHandlerErrorTest.java \
    business_packages/sanyan-business/src/test/java/com/sanyan/chat/ws/WebSocketHandlerErrorTest.java
```

改每个测试 package + import。

- [ ] **Step 10**: 暂时不跑 mvn test（chat 还引用未搬的 user/character/llm）—— 等 Task 3.4 后再跑

- [ ] **Step 11**: commit（即使编译还红）

```bash
git add -A
git commit -m "refactor(chat): 搬 chat 领域 + Message→MessageEntity + 加 ChatErrCode（中间状态）"
```

---

### Task 3.2: 搬 character 领域

**Files:**
- Move + Rename: `src/main/java/com/sanyan/entity/AiCharacter.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/character/internal/AiCharacterEntity.java`
- Move: `src/main/java/com/sanyan/repository/AiCharacterRepository.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/character/internal/AiCharacterRepository.java`
- Create: `business_packages/sanyan-business/src/main/java/com/sanyan/character/internal/CharacterErrCode.java`

- [ ] **Step 1**: 创建目录

```bash
mkdir -p business_packages/sanyan-business/src/main/java/com/sanyan/character/internal
mkdir -p business_packages/sanyan-business/src/test/java/com/sanyan/character/internal
```

- [ ] **Step 2**: git mv + rename AiCharacter → AiCharacterEntity

```bash
git mv src/main/java/com/sanyan/entity/AiCharacter.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/character/internal/AiCharacterEntity.java
```

修改：`package com.sanyan.character.internal;`、`public class AiCharacter` → `public class AiCharacterEntity`、`@Table(name = "ai_character")` 保持。

- [ ] **Step 3**: git mv AiCharacterRepository + 改泛型

```bash
git mv src/main/java/com/sanyan/repository/AiCharacterRepository.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/character/internal/AiCharacterRepository.java
```

修改：`package`、`JpaRepository<AiCharacter, Long>` → `JpaRepository<AiCharacterEntity, Long>`、所有 `AiCharacter` → `AiCharacterEntity`。

- [ ] **Step 4**: 创建 `CharacterErrCode.java`:

```java
package com.sanyan.character.internal;

import com.sanyan.common.error.ErrCode;
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public enum CharacterErrCode implements ErrCode {
    CHARACTER_NOT_FOUND(3001, "角色不存在");

    private final int code;
    private final String defaultMessage;
}
```

- [ ] **Step 5**: 更新 chat.internal.MessageService 中：
- import `com.sanyan.character.internal.AiCharacterEntity` + `AiCharacterRepository` + `CharacterErrCode`
- `AiCharacter` → `AiCharacterEntity`
- 抛 `RuntimeException("默认角色不存在")` → `BusinessException(CharacterErrCode.CHARACTER_NOT_FOUND)`

- [ ] **Step 6**: commit（仍中间状态）

```bash
git add -A
git commit -m "refactor(character): 搬 character 领域 + AiCharacter→AiCharacterEntity + CharacterErrCode"
```

---

### Task 3.3: 搬 llm 领域

**Files:**
- Move: `src/main/java/com/sanyan/service/AiService.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/AiService.java`
- Move: `src/main/resources/prompts/xiaowan-system.md` → `business_packages/sanyan-business/src/main/resources/prompts/xiaowan-system.md`
- Create: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LlmErrCode.java`
- Move: 对应测试

- [ ] **Step 1**: 创建目录

```bash
mkdir -p business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal
mkdir -p business_packages/sanyan-business/src/test/java/com/sanyan/llm/internal
mkdir -p business_packages/sanyan-business/src/main/resources/prompts
```

- [ ] **Step 2**: git mv AiService + 改 import

```bash
git mv src/main/java/com/sanyan/service/AiService.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/AiService.java
```

修改：
- `package com.sanyan.llm.internal;`
- `import com.sanyan.dto.ws.SenderType;` → `com.sanyan.chat.ws.SenderType`
- `import com.sanyan.entity.AiCharacter;` → `com.sanyan.character.internal.AiCharacterEntity` + 所有 `AiCharacter` → `AiCharacterEntity`
- `import com.sanyan.entity.Message;` → `com.sanyan.chat.internal.MessageEntity` + 所有 `Message` → `MessageEntity`
- `import com.sanyan.repository.MessageRepository;` → `com.sanyan.chat.internal.MessageRepository`

- [ ] **Step 3**: 搬 prompts/

```bash
git mv src/main/resources/prompts/xiaowan-system.md \
    business_packages/sanyan-business/src/main/resources/prompts/xiaowan-system.md
```

- [ ] **Step 4**: 创建 `LlmErrCode.java`:

```java
package com.sanyan.llm.internal;

import com.sanyan.common.error.ErrCode;
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public enum LlmErrCode implements ErrCode {
    LLM_CALL_FAILED(4001, "AI 服务暂时不可用");

    private final int code;
    private final String defaultMessage;
}
```

- [ ] **Step 5**: 搬 AiServiceTest

```bash
git mv src/test/java/com/sanyan/service/AiServiceTest.java \
    business_packages/sanyan-business/src/test/java/com/sanyan/llm/internal/AiServiceTest.java
```

修改 package + import。

- [ ] **Step 6**: commit

```bash
git add -A
git commit -m "refactor(llm): 搬 AiService 到 llm 包 + LlmErrCode"
```

---

### Task 3.4: 搬 user 领域 + Service 按动作拆 + AuthService 拆三个 Service

**Files:**
- Move + Rename: `src/main/java/com/sanyan/entity/User.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/user/internal/UserEntity.java`
- Move: `src/main/java/com/sanyan/repository/UserRepository.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/user/internal/UserRepository.java`（泛型改）
- **Split** `src/main/java/com/sanyan/service/AuthService.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/user/internal/UserRegisterService.java` + `UserLoginService.java`
- Move + Rename: `src/main/java/com/sanyan/service/SmsService.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/user/internal/SmsCodeSendService.java`
- Move: `src/main/java/com/sanyan/controller/AuthController.java` → `business_packages/sanyan-business/src/main/java/com/sanyan/user/web/AuthController.java`
- Create: `business_packages/sanyan-business/src/main/java/com/sanyan/user/internal/UserErrCode.java`
- Move: 测试 + 拆 AuthServiceTest 成 UserRegisterServiceTest + UserLoginServiceTest

- [ ] **Step 1**: 创建目录 + 写 `UserErrCode.java`:

```java
package com.sanyan.user.internal;

import com.sanyan.common.error.ErrCode;
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public enum UserErrCode implements ErrCode {
    PHONE_ALREADY_REGISTERED(1001, "手机号已注册"),
    USER_NOT_FOUND(1002, "用户不存在"),
    WRONG_PASSWORD(1003, "密码错误"),
    SMS_CODE_INVALID(1004, "验证码错误"),
    SMS_CODE_EXPIRED(1005, "验证码已过期"),
    SMS_SEND_TOO_FREQUENT(1006, "请稍后再试");

    private final int code;
    private final String defaultMessage;
}
```

- [ ] **Step 2**: 搬 + rename User → UserEntity

```bash
mkdir -p business_packages/sanyan-business/src/main/java/com/sanyan/user/{web,internal}
mkdir -p business_packages/sanyan-business/src/test/java/com/sanyan/user/{web,internal}
git mv src/main/java/com/sanyan/entity/User.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/user/internal/UserEntity.java
```

修改：`package`、类名 `User` → `UserEntity`、`@Table(name = "users")` 保持。

- [ ] **Step 3**: 搬 UserRepository + 改泛型

```bash
git mv src/main/java/com/sanyan/repository/UserRepository.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/user/internal/UserRepository.java
```

`JpaRepository<User, Long>` → `JpaRepository<UserEntity, Long>`、所有 `User` → `UserEntity`、`Optional<User>` → `Optional<UserEntity>`。

- [ ] **Step 4**: **拆 AuthService**（这是 M3 的关键改动）

读现有 `src/main/java/com/sanyan/service/AuthService.java`，把它拆成三个 Service：

**UserRegisterService.java**:
```java
package com.sanyan.user.internal;

import com.sanyan.common.error.BusinessException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
public class UserRegisterService {
    private final UserRepository userRepository;
    private final SmsCodeSendService smsCodeSendService;  // 用于验证 SMS code
    private final BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
    // 注意：BCryptPasswordEncoder 按现状是直接 new，规则推荐做 Bean 但保持现状

    @Transactional
    public RegisterResult register(String phone, String password, String nickname, String smsCode) {
        // 1. 验证短信验证码
        if (!smsCodeSendService.verify(phone, smsCode)) {
            throw new BusinessException(UserErrCode.SMS_CODE_INVALID);
        }
        // 2. 检查手机号
        if (userRepository.existsByPhone(phone)) {
            throw new BusinessException(UserErrCode.PHONE_ALREADY_REGISTERED);
        }
        // 3. 创建
        UserEntity user = new UserEntity();
        user.setPhone(phone);
        user.setNickname(nickname);
        user.setPassword(passwordEncoder.encode(password));
        userRepository.save(user);

        log.info("用户注册成功: userId={}, phone={}", user.getId(), phone);
        return new RegisterResult(user.getId(), user.getNickname(), user.getAvatar());
    }

    public record RegisterResult(Long userId, String nickname, String avatar) {}
}
```

注意：RegisterResult 是返回类型，可以保留为 record 内嵌或独立 dto/data 类（看现状 AuthService 返回的 LoginData 等）。**具体字段以现状 AuthService.register 返回的 LoginData 为准**。

**UserLoginService.java**:
```java
package com.sanyan.user.internal;

import com.sanyan.common.auth.JwtUtil;
import com.sanyan.common.error.BusinessException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class UserLoginService {
    private final UserRepository userRepository;
    private final JwtUtil jwtUtil;
    private final BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();

    public LoginResult login(String phone, String password) {
        UserEntity user = userRepository.findByPhone(phone)
            .orElseThrow(() -> {
                log.warn("登录失败，用户不存在: phone={}", phone);
                return new BusinessException(UserErrCode.USER_NOT_FOUND);
            });
        if (!passwordEncoder.matches(password, user.getPassword())) {
            log.warn("登录失败，密码错误: phone={}", phone);
            throw new BusinessException(UserErrCode.WRONG_PASSWORD);
        }
        log.info("用户登录成功: userId={}", user.getId());
        return new LoginResult(user.getId(), jwtUtil.generateToken(user.getId()),
                               user.getNickname(), user.getAvatar());
    }

    public record LoginResult(Long userId, String token, String nickname, String avatar) {}
}
```

**SmsCodeSendService.java**：现有 SmsService.java 重命名 + 改 package + 抛 BusinessException

```bash
git mv src/main/java/com/sanyan/service/SmsService.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/user/internal/SmsCodeSendService.java
```

修改：
- `package com.sanyan.user.internal;`
- 类名 `SmsService` → `SmsCodeSendService`
- `IllegalArgumentException("发送过于频繁")` → `BusinessException(UserErrCode.SMS_SEND_TOO_FREQUENT)`
- 加 `boolean verify(String phone, String code)` 方法供 UserRegisterService 调（从 Redis 读 code 比对）

- [ ] **Step 5**: 删除 AuthService.java

```bash
git rm src/main/java/com/sanyan/service/AuthService.java
```

- [ ] **Step 6**: 搬 AuthController + 改 Service 注入

```bash
git mv src/main/java/com/sanyan/controller/AuthController.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/user/web/AuthController.java
```

修改：
- `package com.sanyan.user.web;`
- 把 `@Autowired AuthService authService` 改成注入三个：`UserRegisterService` + `UserLoginService` + `SmsCodeSendService`
- 三个端点 `/api/auth/register` / `/login` / `/sendSms` 分别调对应 Service

- [ ] **Step 7**: 搬测试

```bash
git mv src/test/java/com/sanyan/controller/AuthControllerTest.java \
    business_packages/sanyan-business/src/test/java/com/sanyan/user/web/AuthControllerIT.java
git mv src/test/java/com/sanyan/auth/LoginUserArgumentResolverTest.java \
    foundation_packages/sanyan-common-auth/src/test/java/com/sanyan/common/auth/LoginUserArgumentResolverTest.java
git mv src/test/java/com/sanyan/service/SmsServiceTest.java \
    business_packages/sanyan-business/src/test/java/com/sanyan/user/internal/SmsCodeSendServiceTest.java
```

注意：AuthControllerTest 改名为 `AuthControllerIT`。SmsServiceTest 改名为 `SmsCodeSendServiceTest`。

- [ ] **Step 8**: 改测试内容：
- AuthControllerIT：注入改为三个 Service 的 mock
- SmsCodeSendServiceTest：类名引用改 SmsCodeSendService

- [ ] **Step 9**: 写新测试 `UserRegisterServiceTest.java` + `UserLoginServiceTest.java` —— 拆分原 AuthServiceTest（如果存在）的 case，按 register / login 分别测

如果原来没有专门的 AuthServiceTest（看 ls 结果当前测试目录里没有），则**新写**这两个 service 的单元测试，覆盖至少：
- register 成功路径
- register 手机号已存在抛 PHONE_ALREADY_REGISTERED
- register SMS 验证码错抛 SMS_CODE_INVALID
- login 用户不存在抛 USER_NOT_FOUND
- login 密码错抛 WRONG_PASSWORD
- login 成功返回 token

- [ ] **Step 10**: commit

```bash
git add -A
git commit -m "refactor(user): 搬 user 领域 + 拆 AuthService → Register/Login/SmsCodeSend 三个 Service + UserErrCode"
```

---

### Task 3.5: 删除旧 exception + dto + 创建 TestFixtures

**Files:**
- Delete: `src/main/java/com/sanyan/exception/InvalidTokenException.java`
- Delete: `src/main/java/com/sanyan/dto/ApiResponse.java`（被 BaseResp 替换）
- Delete: 其他旧目录（exception/、dto/data/、dto/req/）—— 看里面是否还有未搬的文件
- Create: `business_packages/sanyan-business/src/test/java/com/sanyan/user/internal/UserTestFixtures.java`
- Create: `business_packages/sanyan-business/src/test/java/com/sanyan/chat/internal/MessageTestFixtures.java`
- Create: `business_packages/sanyan-business/src/test/java/com/sanyan/character/internal/AiCharacterTestFixtures.java`

- [ ] **Step 1**: 删 InvalidTokenException

```bash
git rm src/main/java/com/sanyan/exception/InvalidTokenException.java
```

确保 JwtUtil 已不再 import 它（M2.3 处理过）。

- [ ] **Step 2**: 看 dto/data/ 和 dto/req/ 还剩什么

```bash
ls src/main/java/com/sanyan/dto/
```

预期：
- `dto/data/MessageData.java`：搬到 chat.web 包（业务专属响应模型）
- `dto/data/LoginData.java`：可能已被 `UserLoginService.LoginResult` 取代，删除或保留
- `dto/req/LoginReq.java` / `RegisterReq.java` / `SmsReq.java`：搬到 user.web

```bash
git mv src/main/java/com/sanyan/dto/req/*.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/user/web/
git mv src/main/java/com/sanyan/dto/data/MessageData.java \
    business_packages/sanyan-business/src/main/java/com/sanyan/chat/web/MessageData.java
git rm src/main/java/com/sanyan/dto/data/LoginData.java  # 已被 record 取代
git rm src/main/java/com/sanyan/dto/ApiResponse.java
```

修改每个搬过去的文件 package。MessageController 内 `import ...MessageData` 改成同包。

- [ ] **Step 3**: 写 `UserTestFixtures.java`:

```java
package com.sanyan.user.internal;

public class UserTestFixtures {

    public static UserEntity validUser() {
        UserEntity u = new UserEntity();
        u.setPhone("13800138000");
        u.setNickname("测试用户");
        u.setPassword("$2a$10$encrypted");  // 模拟 BCrypt 哈希
        return u;
    }

    public static UserEntity userWithPhone(String phone) {
        UserEntity u = validUser();
        u.setPhone(phone);
        return u;
    }

    public static UserEntity userWithId(Long id) {
        UserEntity u = validUser();
        u.setId(id);
        return u;
    }
}
```

- [ ] **Step 4**: 写 `MessageTestFixtures.java`:

```java
package com.sanyan.chat.internal;

import com.sanyan.chat.ws.SenderType;

public class MessageTestFixtures {

    public static MessageEntity validUserMessage() {
        MessageEntity m = new MessageEntity();
        m.setUserId(1L);
        m.setSenderType(SenderType.USER);
        m.setContent("你好");
        return m;
    }

    public static MessageEntity validAiMessage() {
        MessageEntity m = new MessageEntity();
        m.setUserId(1L);
        m.setSenderType(SenderType.AI);
        m.setContent("你好呀");
        return m;
    }

    public static MessageEntity messageWithContent(String content) {
        MessageEntity m = validUserMessage();
        m.setContent(content);
        return m;
    }
}
```

- [ ] **Step 5**: 写 `AiCharacterTestFixtures.java`:

```java
package com.sanyan.character.internal;

public class AiCharacterTestFixtures {

    public static AiCharacterEntity xiaowan() {
        AiCharacterEntity c = new AiCharacterEntity();
        c.setId(1L);
        c.setName("小婉");
        return c;
    }
}
```

- [ ] **Step 6**: 找现有测试里裸 `new XxxEntity()` 改用 fixture（grep + 手改）

```bash
grep -rln "new UserEntity()\|new MessageEntity()\|new AiCharacterEntity()" \
    business_packages/sanyan-business/src/test/
```

每个匹配的测试改用 `UserTestFixtures.validUser()` 等。

- [ ] **Step 7**: commit

```bash
git add -A
git commit -m "refactor(test): 引入 TestFixtures + 删 InvalidTokenException/ApiResponse + 搬 DTO 到对应业务包"
```

---

### Task 3.6: ApiResponse → BaseResp（Controller 返回类型更新）

**Files:**
- Modify: `AuthController.java`（user.web）—— 所有 return 类型 `ApiResponse<X>` → `BaseResp<X>`，`ApiResponse.success(...)` → `BaseResp.success(...)`
- Modify: `MessageController.java`（chat.web）—— 同上

- [ ] **Step 1**: 全文搜替换

```bash
cd business_packages/sanyan-business
grep -rl "ApiResponse" src/main/java
```

每个文件改：
- `import com.sanyan.dto.ApiResponse;` → `import com.sanyan.common.web.BaseResp;`
- `ApiResponse<` → `BaseResp<`
- `ApiResponse.success(` → `BaseResp.success(`
- `ApiResponse.failed(` → `BaseResp.failed(`

- [ ] **Step 2**: commit

```bash
git add -A
git commit -m "refactor(web): ApiResponse → BaseResp 全替换"
```

---

### Task 3.7: 清理旧 src/ 空目录 + 全量 mvn compile 验证

- [ ] **Step 1**: 删除现在已经空的旧目录

```bash
find src/main/java/com/sanyan -type d -empty -delete
find src/test/java/com/sanyan -type d -empty -delete
find src/main/resources -type d -empty -delete 2>/dev/null
rm -rf src/  # 整个 src/ 应该都搬空了
```

- [ ] **Step 2**: 跑 mvn compile 看全编译

```bash
mvn compile
```

Expected: BUILD SUCCESS。如果还有 import 错误，按错误一个个修。

- [ ] **Step 3**: 跑 mvn test 看测试

```bash
mvn test
```

Expected: 测试数应当接近 194（搬家不增不减；可能有几个因签名变化要小调整）。所有测试 PASS。

如果有失败：根据错误调整 mock / signature / import。

- [ ] **Step 4**: commit M3 收尾

```bash
git add -A
git commit -m "refactor(server): M3 收尾——清空旧 src/，全量编译 + 测试通过"
```

---

## M4：Flyway 接管（~半天，6 个 task）

### Task 4.1: 加 Flyway 依赖到父 POM dependencyManagement

已在 M1 Task 1.1 加过（flyway-database-postgresql）。本任务**校验**它存在，不存在则补。

```bash
grep "flyway-database-postgresql" pom.xml
```

如无，补到 dependencyManagement + business pom 的 dependencies。

### Task 4.2: pg_dump 生产 schema → V1__initial_schema.sql

- [ ] **Step 1**: pg_dump

```bash
ssh new "sudo -u postgres pg_dump --schema-only --no-owner --no-privileges sanyan" \
    > /tmp/sanyan-schema.sql
```

- [ ] **Step 2**: 审查 /tmp/sanyan-schema.sql

打开 `/tmp/sanyan-schema.sql`，剔除以下内容（让 V1 干净）：
- `SET` 语句
- `COMMENT ON ...`
- `ALTER ... OWNER TO ...`（已用 --no-owner 应该没有）
- Sequence 重置语句（`SELECT pg_catalog.setval(...)`）—— Flyway 接管后不需要
- 多余空行

只保留 `CREATE TABLE / CREATE INDEX / ALTER TABLE ... ADD CONSTRAINT` 等结构性 DDL。

- [ ] **Step 3**: 保存到 business 模块

```bash
mkdir -p business_packages/sanyan-business/src/main/resources/db/migration
cp /tmp/sanyan-schema.sql \
   business_packages/sanyan-business/src/main/resources/db/migration/V1__initial_schema.sql
```

- [ ] **Step 4**: 本机 H2 兼容性测试（关键！）—— 在 H2 内存库里跑 V1 看能不能过

```bash
mvn -pl business_packages/sanyan-business test -Dtest=NotExistTestName 2>&1 | grep -A 5 "flyway"
```

如果 H2 报错（如 `JSONB` 不识别 / `SERIAL` 不识别），编辑 V1 用兼容写法（如 `JSONB` 改 `JSON`，`SERIAL` 改 `BIGINT GENERATED ALWAYS AS IDENTITY` 或者 `BIGINT AUTO_INCREMENT`）。

- [ ] **Step 5**: commit

```bash
git add -A
git commit -m "feat(db): V1__initial_schema.sql（pg_dump 现状 + H2 兼容调整）"
```

---

### Task 4.3: data.sql → V2__seed_xiaowan.sql

- [ ] **Step 1**: 读现有 `data.sql`，把里面的 INSERT 改成 V2 格式

文件 `business_packages/sanyan-business/src/main/resources/db/migration/V2__seed_xiaowan.sql`:
```sql
-- 预设 AI 角色（仅在表为空时插入，保持幂等）
INSERT INTO ai_character (name, avatar, created_at)
SELECT '小婉', NULL, NOW()
WHERE NOT EXISTS (SELECT 1 FROM ai_character WHERE name = '小婉');
```

- [ ] **Step 2**: 删除旧 data.sql

```bash
git rm src/main/resources/data.sql 2>/dev/null || true
git rm business_packages/sanyan-business/src/main/resources/data.sql 2>/dev/null || true
```

- [ ] **Step 3**: commit

```bash
git add -A
git commit -m "feat(db): V2__seed_xiaowan.sql 替换 data.sql"
```

---

### Task 4.4: 改 application.yml + application-dev.yml 配 Flyway

- [ ] **Step 1**: 改 `bootstrap/src/main/resources/application.yml`

新增/修改：
```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate    # 之前是 validate，保持
    show-sql: false
  flyway:
    enabled: true
    baseline-on-migrate: true
    baseline-version: 1
    locations: classpath:db/migration
  sql:
    init:
      mode: never           # 关掉 data.sql 自动执行
```

- [ ] **Step 2**: 改 `bootstrap/src/main/resources/application-dev.yml`

把 `ddl-auto: update` 改成 `validate`：
```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate    # 之前是 update，改 validate
```

`spring.sql.init.mode` 删掉（让 application.yml 的 never 生效）。
保留 `defer-datasource-initialization: true`。

- [ ] **Step 3**: commit

```bash
git add -A
git commit -m "feat(db): 配置 Flyway baseline + 关闭 data.sql + ddl-auto 统一 validate"
```

---

### Task 4.5: 本机临时 PG 验证 Flyway baseline 流程

- [ ] **Step 1**: 本机起一个临时 sanyan 数据库

```bash
psql -h localhost -d postgres -c "CREATE DATABASE sanyan_test_flyway;"
psql -h localhost -d postgres -c "CREATE USER sanyan WITH PASSWORD 'test';"
psql -h localhost -d postgres -c "GRANT ALL ON DATABASE sanyan_test_flyway TO sanyan;"
```

- [ ] **Step 2**: 把生产 schema dump 灌进去（模拟"baseline 已有 schema"）

```bash
psql -h localhost -d sanyan_test_flyway \
     -f business_packages/sanyan-business/src/main/resources/db/migration/V1__initial_schema.sql
```

- [ ] **Step 3**: 临时配 dev profile 连到这个临时库 + 启动 app 看 Flyway 行为

```bash
PG_USER=sanyan PG_PASSWORD=test \
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/sanyan_test_flyway \
mvn -pl bootstrap spring-boot:run 2>&1 | grep -i "flyway\|baseline\|migrating"
```

Expected:
- `Schema initialized by Flyway` 或 `Successfully baselined schema with version: 1`
- `Migrating to V2 - seed xiaowan`
- `Successfully applied 1 migration`

- [ ] **Step 4**: 验证 `flyway_schema_history` 表存在且有 V1 + V2 记录

```bash
psql -h localhost -d sanyan_test_flyway -c "SELECT version, description, success FROM flyway_schema_history;"
```

Expected:
```
 1 | initial_schema | t
 2 | seed_xiaowan   | t
```

- [ ] **Step 5**: 清理临时库

```bash
psql -h localhost -d postgres -c "DROP DATABASE sanyan_test_flyway;"
```

- [ ] **Step 6**: commit（无文件变更，只 verify；如果上面验证通过本步骤跳过）

---

### Task 4.6: 部署到 new 服务器 + 验证 Flyway baseline 现有 schema

- [ ] **Step 1**: 跑 deploy.sh

```bash
cd /Users/aventador/code/3yan/server
./deploy.sh
```

- [ ] **Step 2**: ssh 服务器看启动日志

```bash
ssh new "tail -50 /opt/3yan/3yan-server/server.log | grep -iE 'flyway|baseline|migrating|started'"
```

Expected:
- `Successfully validated 2 migrations` 或 `Successfully baselined schema with version: 1`
- `Current version of schema "public": 1` 然后 `Migrating schema "public" to version "2 - seed xiaowan"`
- 之后 `Started SanyanApplication in X seconds`

- [ ] **Step 3**: 验证服务器 PG 有 flyway_schema_history 表

```bash
ssh new "sudo -u postgres psql -d sanyan -c 'SELECT version, description, success FROM flyway_schema_history;'"
```

Expected: V1 baseline + V2 success 各一行。

- [ ] **Step 4**: 端到端 dogfood：登录 app 发条消息看 AI 回复

如果一切正常，M4 完成。

- [ ] **Step 5**: commit M4 完成

```bash
git commit --allow-empty -m "chore(db): M4 Flyway 接管完成（本机+生产 baseline 通过 + 部署验证）"
```

---

## M5：ArchUnit 守护（~半天，5 个 task）

### Task 5.1: 写 ArchitectureTest 骨架 + R1（foundation 不依赖 business）

**Files:**
- Create: `bootstrap/src/test/java/com/sanyan/ArchitectureTest.java`

- [ ] **Step 1**: 创建文件 `ArchitectureTest.java`:

```java
package com.sanyan;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

@AnalyzeClasses(packages = "com.sanyan")
class ArchitectureTest {

    @ArchTest
    static final ArchRule foundation_cannot_depend_on_business =
        noClasses().that().resideInAPackage("com.sanyan.common..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "com.sanyan.user..",
                "com.sanyan.chat..",
                "com.sanyan.character..",
                "com.sanyan.llm..");
}
```

- [ ] **Step 2**: bootstrap pom 加 test 依赖

确保 `bootstrap/pom.xml` 有：
```xml
<dependency>
    <groupId>com.sanyan</groupId>
    <artifactId>sanyan-common-test</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 3**: 跑测试

```bash
mvn -pl bootstrap test -Dtest=ArchitectureTest
```

Expected: PASS（M3 之后 foundation 应该不会反向引 business）。如果 FAIL 看违规并修代码。

- [ ] **Step 4**: commit

```bash
git add bootstrap/src/test/java/com/sanyan/ArchitectureTest.java
git commit -m "test(arch): R1 foundation 不能反向 import business"
```

---

### Task 5.2: R2 跨领域不能引 web/ws 包

- [ ] **Step 1**: 在 `ArchitectureTest` 内加规则：

```java
import com.tngtech.archunit.library.dependencies.SlicesRuleDefinition;
import static com.tngtech.archunit.library.dependencies.SlicesRuleDefinition.slices;

@ArchTest
static final ArchRule no_cross_domain_web_or_ws_access =
    noClasses().that().resideInAnyPackage(
            "com.sanyan.user..",
            "com.sanyan.chat..",
            "com.sanyan.character..",
            "com.sanyan.llm..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "com.sanyan.user.web..", "com.sanyan.user.ws..",
            "com.sanyan.chat.web..", "com.sanyan.chat.ws..",
            "com.sanyan.character.web..", "com.sanyan.character.ws..",
            "com.sanyan.llm.web..", "com.sanyan.llm.ws..")
        .andShould(new ArchCondition<JavaClass>("be in a different domain") {
            @Override
            public void check(JavaClass item, ConditionEvents events) {
                // 此处需手动判断 item 所在领域包前缀与依赖目标的领域前缀是否不同
                // 详见 ArchUnit 文档；M5 实施时可能需要拆成 4 条独立规则一一对应
            }
        });
```

**注**：R2 的精确实现可能需要 4 条独立规则（user 不能引 chat.web/ws、chat 不能引 user.web/ws、等等），上面的写法是初稿。M5 实施时按 ArchUnit API 写法落地。

简化版 4 条独立规则：

```java
@ArchTest
static final ArchRule user_domain_must_not_depend_on_other_domain_web_or_ws =
    noClasses().that().resideInAPackage("com.sanyan.user..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "com.sanyan.chat.web..", "com.sanyan.chat.ws..",
            "com.sanyan.character.web..", "com.sanyan.character.ws..",
            "com.sanyan.llm.web..", "com.sanyan.llm.ws..");

@ArchTest
static final ArchRule chat_domain_must_not_depend_on_other_domain_web_or_ws =
    noClasses().that().resideInAPackage("com.sanyan.chat..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "com.sanyan.user.web..",
            "com.sanyan.character.web..",
            "com.sanyan.llm.web..", "com.sanyan.llm.ws..");

@ArchTest
static final ArchRule character_domain_must_not_depend_on_other_domain_web_or_ws =
    noClasses().that().resideInAPackage("com.sanyan.character..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "com.sanyan.user.web..",
            "com.sanyan.chat.web..", "com.sanyan.chat.ws..",
            "com.sanyan.llm.web..", "com.sanyan.llm.ws..");

@ArchTest
static final ArchRule llm_domain_must_not_depend_on_other_domain_web_or_ws =
    noClasses().that().resideInAPackage("com.sanyan.llm..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "com.sanyan.user.web..",
            "com.sanyan.chat.web..", "com.sanyan.chat.ws..",
            "com.sanyan.character.web..");
```

- [ ] **Step 2**: 跑测试

```bash
mvn -pl bootstrap test -Dtest=ArchitectureTest
```

Expected: PASS

- [ ] **Step 3**: commit

```bash
git commit -am "test(arch): R2 跨领域不允许引 web/ws 包"
```

---

### Task 5.3: R3 web/ws 不能直接调 Repository

- [ ] **Step 1**: 加规则：

```java
import static com.tngtech.archunit.core.domain.JavaClass.Predicates.simpleNameEndingWith;

@ArchTest
static final ArchRule web_layer_must_not_access_repositories =
    noClasses().that().resideInAnyPackage("..web..", "..ws..")
        .should().dependOnClassesThat().haveSimpleNameEndingWith("Repository");
```

- [ ] **Step 2**: 跑测试

```bash
mvn -pl bootstrap test -Dtest=ArchitectureTest
```

Expected: PASS（M3 之后 Controller 都通过 Service 访问数据）。如果 FAIL，看具体哪个 Controller 直接调了 Repository，修代码（应该通过 Service 包一层）。

- [ ] **Step 3**: commit

```bash
git commit -am "test(arch): R3 web/ws 不能直接 import Repository"
```

---

### Task 5.4: R4 ErrCode enum 位置约束

- [ ] **Step 1**: 加规则：

```java
import com.sanyan.common.error.ErrCode;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;

@ArchTest
static final ArchRule errcode_enums_must_be_in_internal_or_common_error =
    classes().that().implement(ErrCode.class).and().areEnums()
        .should().resideInAnyPackage("..internal..", "com.sanyan.common.error..");
```

- [ ] **Step 2**: 跑测试 + 修违规

- [ ] **Step 3**: commit

```bash
git commit -am "test(arch): R4 ErrCode enum 必须在 internal 或 common.error 包"
```

---

### Task 5.5: M5 验证 + dogfood + M5 收尾

- [ ] **Step 1**: 跑全量测试

```bash
mvn test
```

Expected: `Tests run: 198`（194 原有 + 4 ArchUnit），全 PASS。

- [ ] **Step 2**: 跑 `mvn verify`（包含 `*IT`）

```bash
mvn verify
```

Expected: surefire 跑 `*Test`、failsafe 跑 `*IT`（AuthControllerIT 等），全过。

- [ ] **Step 3**: 部署 dev 验证

```bash
./deploy.sh
```

观察服务器启动日志：
- ErrCodeConflictDetector 跑过、无冲突
- Flyway 跑过、schema 一致
- 端口 8080 listen

- [ ] **Step 4**: dogfood：用 app 完整跑一遍：注册 → 登录 → 聊天 → 收 AI 回复（≥2 条拟人多条气泡）

如有 bug 修，commit fix。

- [ ] **Step 5**: 主分支合并

```bash
# server 子模块
git checkout dev
git merge --no-ff feat/p1-architecture -m "merge: feat/p1-architecture → dev（P1 后端架构对齐完成）"
git push

# 主仓库更新 server 子模块引用
cd /Users/aventador/code/3yan
git add server
git commit -m "chore: 更新 server 子模块引用（P1 后端架构对齐完成）"
git push
```

- [ ] **Step 6**: P1 收尾 master 合并

```bash
# server 子模块
cd server
git checkout master
git pull --ff-only
git merge --no-ff dev -m "merge: dev → master（P1 后端架构对齐）"
git push
git checkout dev

# 主仓库
cd ..
git checkout master
git pull --ff-only
git merge --no-ff dev -m "merge: dev → master（P1 后端架构对齐）"
git push
git checkout dev
```

---

## 验收总清单（对应 spec §10）

执行完所有 task 后逐项 check：

**架构层**
- [ ] 10 个 Maven 子模块拓扑搭起来；`mvn validate` 通过 Enforcer
- [ ] foundation_packages 8 个模块按 `sanyan-common-<role>` 命名

**命名规范**
- [ ] Entity 类带 `Entity` 后缀
- [ ] Service 按动作拆（UserRegisterService / UserLoginService / SmsCodeSendService）
- [ ] Controller 集成测试用 `*IT` 后缀

**异常 + 响应**
- [ ] 全项目无 `IllegalArgumentException` / `InvalidTokenException`，全替换为 `BusinessException`
- [ ] `BaseResp` 替换 `ApiResponse`
- [ ] `ErrCodeConflictDetector` 启动跑过无冲突

**测试基础设施**
- [ ] 至少 3 个 `<Domain>TestFixtures`
- [ ] 测试代码路径镜像 src/main

**Flyway**
- [ ] 本机 + 生产都 baseline V1 + 跑 V2
- [ ] `pg_dump --schema-only` 前后 diff 为空（schema 不变）

**ArchUnit**
- [ ] 4 条规则全过
- [ ] 全量测试 PASS（≈ 198 个）

**端到端**
- [ ] dogfood 端到端正常
- [ ] 部署服务器正常

---

## Self-Review 备注

写完此 plan 后做了 self-review，确认：

1. **Spec 覆盖**：spec §1-11 每个章节都能对应到具体 task（M1.x ~ M5.x）。
2. **Placeholder 扫描**：所有 step 都有具体代码或命令，无 TBD / TODO / "implement later" 等。
3. **类型一致**：`UserEntity` / `MessageEntity` / `AiCharacterEntity` 命名贯穿全 plan；`BusinessException(SomeErrCode)` / `BaseResp.success(...)` 调用一致。
4. **关键风险**：R3（H2 兼容 Flyway V1）在 Task 4.2 Step 4 已显式处理；R4（ArchUnit 暴露存量违规）在 Task 5.1~5.4 每条规则跑测试后立即修代码。

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-13-p1-backend-architecture.md`. Two execution options:

**1. Subagent-Driven (recommended)** - 我每个 task 派一个 fresh subagent 实施 + 两段审查（spec 审查 + 代码审查），主会话保持干净，迭代快。

**2. Inline Execution** - 在当前会话里逐 task 执行，checkpoint 时停下来给你 review。

哪种？
