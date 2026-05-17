# S3 sanyan-business 单体拆解实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal：** 把 `sanyan-business` 单体按 java-backend rule 拆成 6 对 `-api`/`-core` 业务模块 + 配置类下沉 foundation，让跨模块通信走 `-api` 契约，由 Maven Enforcer + ArchUnit 守护边界。零业务逻辑变化。

**Architecture：** 7 个 Phase 拓扑递增：Phase 0（HttpClientFactory 上提）→ Phase 1 (user) → Phase 2 (character) → Phase 3 (llm) → Phase 4 (embedding) → Phase 5 (chat) → Phase 6 (删 business 空壳 + config 下沉) → Phase 7 (ArchUnit/Enforcer 加固 + 文档同步)。每 Phase 独立 commit，跑 `mvn verify` 通过才进入下一 Phase。

**Tech Stack：** Java 17 + Spring Boot 3.2 + Maven multi-module + JUnit 5 + ArchUnit 1.2 + Testcontainers PG 17 + pgvector。

**前置依赖：** spec doc `docs/superpowers/specs/2026-05-17-s3-business-modular-split-design.md` 已通过。

**全局约束（所有 Task 都遵守）：**
- TDD 铁律：先写失败测试 → 跑出 fail → 写最小实现 → 跑出 pass → commit
- 本任务是**重构**：业务逻辑零变化，旧 389+ 测试是 safety net
- 每 Task 改完跑 `mvn verify` 全绿才能 commit
- Commit message 用中文 + conventional prefix（无 Co-Authored-By）
- 工作目录：`/Users/aventador/code/3yan/server`
- git 分支：`s3-modular-split`（已切好）
- 跨业务模块**禁止** import 对方 `internal/*`；必须走 `-api`

**子代理执行模式：**
每个 Task 派 3 个子代理依次执行：
1. **实现子代理** (`general-purpose`)：按 Task 描述完成代码改动
2. **spec 审查子代理** (`general-purpose`)：对照 design doc + 本 plan 检查改动是否完整、是否越界
3. **代码质量审查子代理** (`superpowers:code-reviewer`)：检查代码规范、命名、TDD 节奏

3 个子代理都 PASS 后 commit 并进入下一 Task。

---

## File Structure Overview

完成后 `business_packages/` 和 `foundation_packages/` 的最终目录见 spec §3.1。每个 Phase 改动哪些文件，在每个 Task 的 **Files** 段标明。

---

## Phase 0 · HttpClientFactory 上提到 foundation

### Task 0.1: 把 LlmHttpClients 抽到 sanyan-common-web/client/HttpClientFactory

**Files:**
- Create: `foundation_packages/sanyan-common-web/src/main/java/com/sanyan/common/web/client/HttpClientFactory.java`
- Create: `foundation_packages/sanyan-common-web/src/test/java/com/sanyan/common/web/client/HttpClientFactoryTest.java`
- Delete: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LlmHttpClients.java`
- Modify: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/DoubaoAdapter.java`（import 替换）
- Modify: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/DeepSeekAdapter.java`（import 替换）
- Modify: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/SiliconFlowEmbeddingProvider.java`（import 替换）

- [ ] **Step 1: 写失败测试 HttpClientFactoryTest**

```java
package com.sanyan.common.web.client;

import org.junit.jupiter.api.Test;
import org.springframework.web.client.RestClient;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class HttpClientFactoryTest {

    @Test
    void newClient_shouldReturnNonNullRestClient() {
        RestClient client = HttpClientFactory.newClient(
                "https://example.com",
                Duration.ofSeconds(3),
                Duration.ofSeconds(10));
        assertThat(client).isNotNull();
    }

    @Test
    void newClient_shouldRejectNullBaseUrl() {
        assertThatThrownBy(() -> HttpClientFactory.newClient(null,
                Duration.ofSeconds(3), Duration.ofSeconds(10)))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: 跑测试确认 fail（compile error: HttpClientFactory 不存在）**

```bash
mvn -pl foundation_packages/sanyan-common-web test -Dtest=HttpClientFactoryTest
```

预期：`Compilation failure ... HttpClientFactory not found`

- [ ] **Step 3: 实现 HttpClientFactory（从 LlmHttpClients 改名搬过来）**

```java
package com.sanyan.common.web.client;

import org.springframework.http.MediaType;
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.web.client.RestClient;

import java.time.Duration;

/**
 * 通用 RestClient 工厂。配 connect/read timeout + JSON Content-Type 默认头 + baseUrl。
 *
 * <p>从原 sanyan-business/llm/internal/LlmHttpClients 改名上提到 foundation，
 * 让 llm-core / embedding-core 等业务方都能复用，避免每个模块自己重复一份。
 *
 * <p>调用方仍需为每次请求显式设置 {@code Authorization} / {@code X-Internal-Token}
 * 等私密 header，本工厂不知道凭证语义。
 */
public final class HttpClientFactory {

    private HttpClientFactory() {
        // utility class，禁止实例化
    }

    public static RestClient newClient(String baseUrl, Duration connectTimeout, Duration readTimeout) {
        if (baseUrl == null) {
            throw new IllegalArgumentException("baseUrl must not be null");
        }
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout((int) connectTimeout.toMillis());
        factory.setReadTimeout((int) readTimeout.toMillis());
        return RestClient.builder()
                .baseUrl(baseUrl)
                .requestFactory(factory)
                .defaultHeader("Content-Type", MediaType.APPLICATION_JSON_VALUE)
                .build();
    }
}
```

- [ ] **Step 4: 跑测试确认 pass**

```bash
mvn -pl foundation_packages/sanyan-common-web test -Dtest=HttpClientFactoryTest
```

预期：`Tests run: 2, Failures: 0, Errors: 0`

- [ ] **Step 5: 删 LlmHttpClients + 改三个 adapter 的 import**

删 `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LlmHttpClients.java`。

把以下三个文件的 `import com.sanyan.llm.internal.LlmHttpClients;` 全部改成 `import com.sanyan.common.web.client.HttpClientFactory;`，并把方法调用 `LlmHttpClients.newClient(...)` 改成 `HttpClientFactory.newClient(...)`：

```bash
rg "LlmHttpClients" business_packages/sanyan-business/src/ -l
```

逐文件用 Edit 工具替换（每个文件 2 处：import 行 + 调用行）。

- [ ] **Step 6: 跑全量 verify 确认零回归**

```bash
mvn verify
```

预期：`BUILD SUCCESS` + 全部测试通过

- [ ] **Step 7: 全工程 grep 确认无残留**

```bash
rg "LlmHttpClients" .
```

预期：空（无任何命中）

- [ ] **Step 8: Commit**

```bash
git add foundation_packages/sanyan-common-web/ \
        business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/

git commit -m "refactor(common-web): 抽 HttpClientFactory 下沉到 foundation（Phase 0 / 7）

把 sanyan-business/llm/internal/LlmHttpClients 改名为 HttpClientFactory
搬到 sanyan-common-web/client/，让后续 llm-core 和 embedding-core 都能
复用同一份 RestClient 工厂。本 commit 是 S3 拆模块的准备步骤。"
```

---

## Phase 1 · sanyan-user-api/core

### Task 1.1: 新建 sanyan-user-api 模块（pom + UserApi 接口 + UserDto）

**Files:**
- Create: `business_packages/sanyan-user-api/pom.xml`
- Create: `business_packages/sanyan-user-api/src/main/java/com/sanyan/user/UserApi.java`
- Create: `business_packages/sanyan-user-api/src/main/java/com/sanyan/user/dto/UserDto.java`
- Create: `business_packages/sanyan-user-api/src/main/java/com/sanyan/user/package-info.java`
- Modify: `pom.xml`（父 pom 加 module 声明 + dependencyManagement）

- [ ] **Step 1: 创建 pom.xml**

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

    <artifactId>sanyan-user-api</artifactId>
    <name>sanyan-user-api</name>
    <packaging>jar</packaging>

    <!--
        S3 Phase 1：user 域对内契约。
        只含 UserApi 接口 + UserDto record。
        无任何业务/基础模块依赖（纯 JDK，便于其他业务 -core 引用而不引入传递依赖）。
    -->
    <dependencies>
        <!-- 故意为空：-api 只含接口 + DTO + 事件，纯 Java，无依赖 -->
    </dependencies>
</project>
```

- [ ] **Step 2: 创建 UserApi 接口**

```java
package com.sanyan.user;

import com.sanyan.user.dto.UserDto;

/**
 * User 域对内 API 契约。其他业务 -core 只能依赖本接口，不能引 user-core/internal/。
 *
 * <p>本接口故意保持窄（findById / existsByPhone）—— 当前业务里其他域只需要这两个查询。
 * 后续如果有跨模块更新需求（如 chat 模块要改 nickname），先加到本接口再实现，
 * 不允许其他模块绕过 UserApi 直接动 UserEntity。
 */
public interface UserApi {
    /** 按 userId 查用户摘要；不存在返回 null。 */
    UserDto findById(Long userId);

    /** 检查手机号是否已注册。用于注册前的校验。 */
    boolean existsByPhone(String phone);
}
```

- [ ] **Step 3: 创建 UserDto record**

```java
package com.sanyan.user.dto;

/**
 * User 跨模块 DTO。字段是其他业务实际需要的最小集合。
 *
 * <p>不要直接暴露 UserEntity 的 passwordHash / smsCode 等内部字段。
 * 如果某个跨模块场景需要新字段，先在本 record 加，再让 UserApiImpl 填充。
 */
public record UserDto(
        Long id,
        String phone,
        String nickname,
        String avatarUrl) {}
```

- [ ] **Step 4: 创建 package-info.java（标注模块契约）**

```java
/**
 * sanyan-user-api：User 域对内契约。
 *
 * <p>规则约束（java-backend §2.2）：
 * <ul>
 *   <li>本模块只能含 接口 + DTO + 事件定义，不能有任何 @Service / @Component / @Entity</li>
 *   <li>本模块零依赖（不引其他业务模块也不引基础模块）</li>
 *   <li>实现在 sanyan-user-core 内，通过 UserApiImpl 暴露</li>
 * </ul>
 */
package com.sanyan.user;
```

- [ ] **Step 5: 父 pom 注册 module + dependencyManagement**

修改 `pom.xml`：

在 `<modules>` 段 `<module>business_packages/sanyan-memory-core</module>` 行前面加：

```xml
        <module>business_packages/sanyan-user-api</module>
        <module>business_packages/sanyan-user-core</module>
```

注意：user-core 此 Task 还没建，但提前注册没事，Step 6 跑 mvn 会忽略不存在的模块——所以应该 Task 1.2 一起注册。**修正：此 Step 只注册 sanyan-user-api，user-core 在 Task 1.2 加。**

```xml
        <module>business_packages/sanyan-user-api</module>
```

在 `<dependencyManagement><dependencies>` 段 `sanyan-memory-core` 行下方加：

```xml
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-user-api</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-user-core</artifactId><version>${project.version}</version></dependency>
```

- [ ] **Step 6: 跑 mvn compile 验证 user-api 模块独立可编译**

```bash
mvn -pl business_packages/sanyan-user-api compile
```

预期：`BUILD SUCCESS`

- [ ] **Step 7: Commit（不 commit，等 Task 1.2 user-core 建好一起 commit）**

本 Task 不单独 commit。改动留在 working dir，跟下一个 Task 一起 commit。

---

### Task 1.2: 新建 sanyan-user-core 模块（pom + 迁 user 包内全部类 + 实现 UserApiImpl）

**Files:**
- Create: `business_packages/sanyan-user-core/pom.xml`
- Move 11 files from `business_packages/sanyan-business/src/main/java/com/sanyan/user/**` to `business_packages/sanyan-user-core/src/main/java/com/sanyan/user/**`
- Move existing tests from `business_packages/sanyan-business/src/test/java/com/sanyan/user/**` to `business_packages/sanyan-user-core/src/test/java/com/sanyan/user/**`
- Create: `business_packages/sanyan-user-core/src/main/java/com/sanyan/user/api/UserApiImpl.java`
- Create: `business_packages/sanyan-user-core/src/test/java/com/sanyan/user/UserApplicationContextIT.java`
- Modify: `bootstrap/pom.xml`（加 sanyan-user-core 依赖）
- Modify: `business_packages/sanyan-business/pom.xml`（加 sanyan-user-api 依赖，给后续可能残留的引用过渡用——实际上 user 没被任何 internal/* 引用，所以这个依赖加不加无所谓，跳过此步）
- Modify: `pom.xml`（父 pom 加 user-core module 声明，已在 Task 1.1 提前加 user-api，本 step 加 user-core）

- [ ] **Step 1: 在父 pom 加 user-core module 声明**

把 Task 1.1 Step 5 加的 `sanyan-user-api` 行旁边再补一行：

```xml
        <module>business_packages/sanyan-user-api</module>
        <module>business_packages/sanyan-user-core</module>
```

- [ ] **Step 2: 创建 sanyan-user-core/pom.xml**

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

    <artifactId>sanyan-user-core</artifactId>
    <name>sanyan-user-core</name>
    <packaging>jar</packaging>

    <dependencies>
        <!-- 本域 -api 契约 -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-user-api</artifactId></dependency>

        <!-- foundation -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-error</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-web</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-auth</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-cache</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-util</artifactId></dependency>

        <!-- Spring Boot 必备 -->
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-validation</artifactId></dependency>
        <dependency><groupId>org.postgresql</groupId><artifactId>postgresql</artifactId><scope>runtime</scope></dependency>
        <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId></dependency>

        <!-- 测试 -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>com.h2database</groupId><artifactId>h2</artifactId><scope>test</scope></dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: 物理迁移 user 包 main 代码**

```bash
mkdir -p business_packages/sanyan-user-core/src/main/java/com/sanyan/user
git mv business_packages/sanyan-business/src/main/java/com/sanyan/user/internal \
       business_packages/sanyan-user-core/src/main/java/com/sanyan/user/
git mv business_packages/sanyan-business/src/main/java/com/sanyan/user/web \
       business_packages/sanyan-user-core/src/main/java/com/sanyan/user/
```

预期：`git status` 显示 11 个文件 renamed

- [ ] **Step 4: 物理迁移 user 包 test 代码**

先看 user 包有哪些测试：

```bash
find business_packages/sanyan-business/src/test/java/com/sanyan/user/ -type f
```

如果有，整目录搬：

```bash
mkdir -p business_packages/sanyan-user-core/src/test/java/com/sanyan/user
git mv business_packages/sanyan-business/src/test/java/com/sanyan/user/* \
       business_packages/sanyan-user-core/src/test/java/com/sanyan/user/
```

如果没有该 test 目录，跳过这步。

- [ ] **Step 5: 写 UserApiImpl（在新模块 api/ 包下）**

`business_packages/sanyan-user-core/src/main/java/com/sanyan/user/api/UserApiImpl.java`:

```java
package com.sanyan.user.api;

import com.sanyan.user.UserApi;
import com.sanyan.user.dto.UserDto;
import com.sanyan.user.internal.UserEntity;
import com.sanyan.user.internal.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserApiImpl implements UserApi {

    private final UserRepository repository;

    @Override
    public UserDto findById(Long userId) {
        return repository.findById(userId)
                .map(this::toDto)
                .orElse(null);
    }

    @Override
    public boolean existsByPhone(String phone) {
        return repository.existsByPhone(phone);
    }

    private UserDto toDto(UserEntity e) {
        return new UserDto(e.getId(), e.getPhone(), e.getNickname(), e.getAvatarUrl());
    }
}
```

注意：UserEntity 的 getter 名是 `getAvatarUrl` 还是别的，请先 Read UserEntity.java 确认；如果 UserEntity 没有 avatarUrl 字段，UserDto 的对应位置传 null。

- [ ] **Step 6: 写失败测试 UserApplicationContextIT**

`business_packages/sanyan-user-core/src/test/java/com/sanyan/user/UserApplicationContextIT.java`:

```java
package com.sanyan.user;

import com.sanyan.user.api.UserApiImpl;
import com.sanyan.user.internal.UserRepository;
import com.sanyan.common.test.TestApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.AutoConfigureTestDatabase;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * sanyan-user-core 上下文冒烟测试：UserApi Bean 能装配。
 * 确认拆模块后 Spring 仍然能 @ComponentScan 到 com.sanyan.user.api/internal。
 */
@SpringBootTest(classes = TestApplication.class)
@AutoConfigureTestDatabase
class UserApplicationContextIT {

    @Autowired
    private UserApi userApi;

    @Autowired
    private UserRepository userRepository;

    @Test
    void contextLoads_userApiBeanInjected() {
        assertThat(userApi).isNotNull();
        assertThat(userApi).isInstanceOf(UserApiImpl.class);
        assertThat(userRepository).isNotNull();
    }
}
```

- [ ] **Step 7: 跑测试确认 pass（仅本模块）**

```bash
mvn -pl business_packages/sanyan-user-core verify -am
```

预期：`BUILD SUCCESS` + UserApplicationContextIT PASS + 原 user 域单测（如 UserRegisterServiceTest）也 PASS

- [ ] **Step 8: 改 bootstrap pom 加 sanyan-user-core 依赖**

修改 `bootstrap/pom.xml`，在 `<dependencies>` 段补：

```xml
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-user-core</artifactId></dependency>
```

- [ ] **Step 9: 跑全量 verify**

```bash
mvn verify
```

预期：`BUILD SUCCESS` + 全部测试通过（这一步会跑包括 bootstrap 的所有 IT）

- [ ] **Step 10: Commit Phase 1**

```bash
git add -A
git commit -m "refactor(modules): 拆 sanyan-user-api/core（Phase 1 / 7）

- 新建 sanyan-user-api（UserApi 接口 + UserDto record，零依赖）
- 新建 sanyan-user-core（迁 user 包全部 + 新增 UserApiImpl + 冒烟 IT）
- 父 pom + bootstrap pom 注册新模块
- 旧 sanyan-business/com/sanyan/user/ 目录消失（git mv 留 history）
- 业务逻辑零变化，389+ 测试全绿"
```

---

## Phase 2 · sanyan-character-api/core

### Task 2.1: 新建 sanyan-character-api 模块

**Files:**
- Create: `business_packages/sanyan-character-api/pom.xml`
- Create: `business_packages/sanyan-character-api/src/main/java/com/sanyan/character/CharacterApi.java`
- Create: `business_packages/sanyan-character-api/src/main/java/com/sanyan/character/dto/AiCharacterDto.java`
- Create: `business_packages/sanyan-character-api/src/main/java/com/sanyan/character/package-info.java`
- Modify: `pom.xml`（父 pom 加 module 声明 + dependencyManagement）

- [ ] **Step 1: 先 Read 现有 AiCharacterEntity 了解字段，确定 DTO 暴露哪些**

```bash
cat business_packages/sanyan-business/src/main/java/com/sanyan/character/internal/AiCharacterEntity.java
```

记录字段名 + 类型，下面 DTO 用。

- [ ] **Step 2: 创建 pom.xml**

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
    <artifactId>sanyan-character-api</artifactId>
    <name>sanyan-character-api</name>
    <packaging>jar</packaging>
    <dependencies>
        <!-- 零依赖，纯 Java -->
    </dependencies>
</project>
```

- [ ] **Step 3: 创建 CharacterApi 接口**

```java
package com.sanyan.character;

import com.sanyan.character.dto.AiCharacterDto;

/**
 * Character 域对内 API 契约。
 *
 * <p>chat / llm / 其他业务调用方需要 character 数据时走这里，不允许直接 import character-core/internal。
 */
public interface CharacterApi {
    /** 按 id 查角色；不存在返回 null。 */
    AiCharacterDto findById(Long characterId);
}
```

- [ ] **Step 4: 创建 AiCharacterDto record**

> ⚠️ 字段以 Step 1 Read 出来的 AiCharacterEntity 为准。下面是占位写法（按当前 AiCharacterEntity 的字段：id / name / basePrompt / personaConfig），实际写时按 Read 结果调整。

```java
package com.sanyan.character.dto;

import java.util.Map;

/**
 * AiCharacter 跨模块 DTO。字段是其他业务实际需要的最小集合。
 *
 * <p>personaConfig 是 JSONB 字段，跨模块仍以 Map 形式暴露；具体业务字段（stage_overrides 等）
 * 由调用方自己解析或后续在本 record 加专用子结构。
 */
public record AiCharacterDto(
        Long id,
        String name,
        String basePrompt,
        Map<String, Object> personaConfig) {}
```

- [ ] **Step 5: 创建 package-info.java**

```java
/**
 * sanyan-character-api：Character 域对内契约（同 user-api 的约束）。
 */
package com.sanyan.character;
```

- [ ] **Step 6: 父 pom 加 character-api 模块声明 + dependencyManagement**

修改 `pom.xml`，在 user-api/core 后面补：

```xml
        <module>business_packages/sanyan-character-api</module>
```

dependencyManagement：

```xml
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-character-api</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-character-core</artifactId><version>${project.version}</version></dependency>
```

- [ ] **Step 7: 验证 character-api 独立可编译**

```bash
mvn -pl business_packages/sanyan-character-api compile
```

预期：`BUILD SUCCESS`

- [ ] **Step 8: 不 commit，跟 Task 2.2 一起**

---

### Task 2.2: 新建 sanyan-character-core 模块 + CharacterApiImpl + 改调用方

**Files:**
- Create: `business_packages/sanyan-character-core/pom.xml`
- Move: `business_packages/sanyan-business/src/main/java/com/sanyan/character/internal/**` → `business_packages/sanyan-character-core/src/main/java/com/sanyan/character/internal/`
- Create: `business_packages/sanyan-character-core/src/main/java/com/sanyan/character/api/CharacterApiImpl.java`
- Create: `business_packages/sanyan-character-core/src/test/java/com/sanyan/character/CharacterApplicationContextIT.java`
- Modify: `business_packages/sanyan-business/pom.xml`（加 sanyan-character-api 依赖）
- Modify: `business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageService.java`（替换 character.internal.* import）
- Modify: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/AiService.java`（替换 character.internal.* import）
- Modify: `bootstrap/pom.xml`（加 sanyan-character-core 依赖）
- Modify: `pom.xml`（父 pom 加 character-core module 声明）

- [ ] **Step 1: 父 pom 加 character-core 模块声明**

修改 `pom.xml`：

```xml
        <module>business_packages/sanyan-character-api</module>
        <module>business_packages/sanyan-character-core</module>
```

- [ ] **Step 2: 创建 character-core/pom.xml**

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
    <artifactId>sanyan-character-core</artifactId>
    <name>sanyan-character-core</name>
    <packaging>jar</packaging>
    <dependencies>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-character-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-error</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-web</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
        <dependency><groupId>org.postgresql</groupId><artifactId>postgresql</artifactId><scope>runtime</scope></dependency>
        <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId></dependency>

        <!-- 测试 -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>com.h2database</groupId><artifactId>h2</artifactId><scope>test</scope></dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: 物理迁移 character 包**

```bash
mkdir -p business_packages/sanyan-character-core/src/main/java/com/sanyan/character
git mv business_packages/sanyan-business/src/main/java/com/sanyan/character/internal \
       business_packages/sanyan-character-core/src/main/java/com/sanyan/character/
```

预期：`git status` 显示 3 个文件 renamed

- [ ] **Step 4: 写 CharacterApiImpl**

`business_packages/sanyan-character-core/src/main/java/com/sanyan/character/api/CharacterApiImpl.java`:

```java
package com.sanyan.character.api;

import com.sanyan.character.CharacterApi;
import com.sanyan.character.dto.AiCharacterDto;
import com.sanyan.character.internal.AiCharacterEntity;
import com.sanyan.character.internal.AiCharacterRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class CharacterApiImpl implements CharacterApi {

    private final AiCharacterRepository repository;

    @Override
    public AiCharacterDto findById(Long characterId) {
        return repository.findById(characterId)
                .map(this::toDto)
                .orElse(null);
    }

    private AiCharacterDto toDto(AiCharacterEntity e) {
        return new AiCharacterDto(
                e.getId(),
                e.getName(),
                e.getBasePrompt(),
                e.getPersonaConfig()
        );
    }
}
```

> ⚠️ 字段名以实际 AiCharacterEntity 为准。如果字段名不同（如 `getPrompt()` 而非 `getBasePrompt()`），按实际改。

- [ ] **Step 5: 写 CharacterApplicationContextIT**

`business_packages/sanyan-character-core/src/test/java/com/sanyan/character/CharacterApplicationContextIT.java`:

```java
package com.sanyan.character;

import com.sanyan.character.api.CharacterApiImpl;
import com.sanyan.character.internal.AiCharacterRepository;
import com.sanyan.common.test.TestApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.AutoConfigureTestDatabase;
import org.springframework.boot.test.context.SpringBootTest;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(classes = TestApplication.class)
@AutoConfigureTestDatabase
class CharacterApplicationContextIT {

    @Autowired private CharacterApi characterApi;
    @Autowired private AiCharacterRepository repository;

    @Test
    void contextLoads_characterApiBeanInjected() {
        assertThat(characterApi).isNotNull().isInstanceOf(CharacterApiImpl.class);
        assertThat(repository).isNotNull();
    }
}
```

- [ ] **Step 6: 改 sanyan-business/pom.xml 加 character-api 依赖**

在 `sanyan-business/pom.xml` 的 `<dependencies>` 段加：

```xml
        <!-- S3 Phase 2 过渡：chat 和 llm 包仍在 sanyan-business 内，要调 character 走 character-api -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-character-api</artifactId></dependency>
```

- [ ] **Step 7: 改 MessageService（chat 包内）import character → CharacterApi**

Read `business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageService.java`，找出所有使用 `AiCharacterEntity` / `AiCharacterRepository` / `CharacterErrCode` 的位置，逐处替换：

| 旧 | 新 |
|---|---|
| `import com.sanyan.character.internal.AiCharacterEntity;` | `import com.sanyan.character.dto.AiCharacterDto;` |
| `import com.sanyan.character.internal.AiCharacterRepository;` | `import com.sanyan.character.CharacterApi;` |
| `import com.sanyan.character.internal.CharacterErrCode;` | _删除_（错误码用 `BusinessException` 的 code 比较，见 Step 8） |
| `private final AiCharacterRepository characterRepository;` | `private final CharacterApi characterApi;` |
| `AiCharacterEntity` 类型 | `AiCharacterDto` 类型 |
| `characterRepository.findById(id).orElseThrow(...)` | `AiCharacterDto c = characterApi.findById(id); if (c == null) throw new BusinessException(...);` |

> ⚠️ MessageService 里如果原代码用 `entity.getName()` 等 getter，DTO 是 record 改成 `dto.name()`。

- [ ] **Step 8: 处理 CharacterErrCode 跨模块引用**

如果 MessageService 原代码抛 `BusinessException(CharacterErrCode.CHARACTER_NOT_FOUND)`，因为现在 CharacterErrCode 在 character-core/internal/ 不可见，需要：

**方案 A（推荐）**：让 character-core 自己提供"找不到时抛异常"的方法，调用方直接拿结果不操心错误码：

修改 `CharacterApi` 加一个 method：

```java
/** 按 id 查角色；不存在抛 BusinessException(CHARACTER_NOT_FOUND=3001)。 */
AiCharacterDto getById(Long characterId);
```

`CharacterApiImpl` 实现：

```java
@Override
public AiCharacterDto getById(Long characterId) {
    return repository.findById(characterId)
            .map(this::toDto)
            .orElseThrow(() -> new BusinessException(CharacterErrCode.CHARACTER_NOT_FOUND));
}
```

调用方（MessageService）改成 `AiCharacterDto c = characterApi.getById(id);`。

- [ ] **Step 9: 改 AiService（llm 包内）同样处理**

Read + 改 `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/AiService.java`，按 Step 7-8 同样规则替换 character.internal.* 引用。

- [ ] **Step 10: 改 bootstrap pom 加 character-core 依赖**

```xml
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-character-core</artifactId></dependency>
```

- [ ] **Step 11: grep 确认 character.internal 在外部模块全部清零**

```bash
rg "com\.sanyan\.character\.internal\." business_packages/sanyan-business/ business_packages/sanyan-memory-core/
```

预期：空（除了 sanyan-character-core 本模块）

- [ ] **Step 12: 跑全量 verify**

```bash
mvn verify
```

预期：`BUILD SUCCESS` + 全部测试通过

- [ ] **Step 13: Commit Phase 2**

```bash
git add -A
git commit -m "refactor(modules): 拆 sanyan-character-api/core（Phase 2 / 7）

- 新建 sanyan-character-api（CharacterApi 接口 + AiCharacterDto record）
- 新建 sanyan-character-core（迁 internal/ 全部 + CharacterApiImpl + 冒烟 IT）
- chat/llm（仍在 sanyan-business）改走 CharacterApi.getById/findById，不再 import character.internal
- CharacterErrCode 留 character-core/internal/，跨模块错误码识别走 BusinessException.getErrCode().getCode()
- 业务逻辑零变化"
```

---

## Phase 3 · sanyan-llm-api/core（不含 embedding）

### Task 3.1: 新建 sanyan-llm-api 模块

**Files:**
- Create: `business_packages/sanyan-llm-api/pom.xml`
- Create: `business_packages/sanyan-llm-api/src/main/java/com/sanyan/llm/LlmApi.java`
- Create: `business_packages/sanyan-llm-api/src/main/java/com/sanyan/llm/LlmTaskType.java`（从 LLMTaskType 改名 + 上提）
- Create: `business_packages/sanyan-llm-api/src/main/java/com/sanyan/llm/dto/ChatMessage.java`
- Create: `business_packages/sanyan-llm-api/src/main/java/com/sanyan/llm/package-info.java`
- Modify: `pom.xml`（父 pom 加 module 声明 + dependencyManagement）

- [ ] **Step 1: Read 现有 LLMProviderRouter / LLMProvider / LLMTaskType 了解契约**

```bash
cat business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LLMProviderRouter.java
cat business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LLMProvider.java
cat business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LLMTaskType.java
```

确认：调用方传 `LLMTaskType` + `List<Map<String,String>> messages`，返回 `String` AI 文本。

- [ ] **Step 2: 创建 pom.xml**

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
    <artifactId>sanyan-llm-api</artifactId>
    <name>sanyan-llm-api</name>
    <packaging>jar</packaging>
    <dependencies>
        <!-- 零依赖 -->
    </dependencies>
</project>
```

- [ ] **Step 3: 创建 LlmTaskType enum（取代原 LLMTaskType）**

```java
package com.sanyan.llm;

/**
 * LLM 任务类型，决定走哪个 provider。
 *
 * <ul>
 *   <li>{@link #USER_FACING}：用户对话，质量优先（豆包 doubao-seed-character）</li>
 *   <li>{@link #BACKGROUND}：后台任务（摘要 / 档案抽取），便宜优先（DeepSeek V4-Flash）</li>
 * </ul>
 */
public enum LlmTaskType {
    USER_FACING,
    BACKGROUND
}
```

- [ ] **Step 4: 创建 ChatMessage record**

```java
package com.sanyan.llm.dto;

/**
 * OpenAI 兼容的 chat message。role 为 "system" / "user" / "assistant"。
 *
 * <p>各业务方（chat 域、memory 域的 summary/profile）独立拼装 List&lt;ChatMessage&gt;，
 * 然后传给 LlmApi.chat。
 */
public record ChatMessage(String role, String content) {

    public static ChatMessage system(String content) {
        return new ChatMessage("system", content);
    }

    public static ChatMessage user(String content) {
        return new ChatMessage("user", content);
    }

    public static ChatMessage assistant(String content) {
        return new ChatMessage("assistant", content);
    }
}
```

- [ ] **Step 5: 创建 LlmApi 接口**

```java
package com.sanyan.llm;

import com.sanyan.llm.dto.ChatMessage;

import java.util.List;

/**
 * LLM 域对内 API 契约。
 *
 * <p>由 LlmProviderRouter 内部按 {@link LlmTaskType} 路由到具体 provider（豆包 / DeepSeek）。
 *
 * <p>错误语义：上游失败由 LlmApi 实现抛 BusinessException，调用方按需 catch 降级。
 */
public interface LlmApi {
    /**
     * 调 LLM 进行对话推理。
     *
     * @param taskType 决定 provider（USER_FACING 走豆包，BACKGROUND 走 DeepSeek）
     * @param messages OpenAI 兼容 chat messages（system + user + assistant 序列）
     * @return AI 回复的文本
     */
    String chat(LlmTaskType taskType, List<ChatMessage> messages);
}
```

- [ ] **Step 6: 创建 package-info.java**

```java
/** sanyan-llm-api：LLM 域对内契约。 */
package com.sanyan.llm;
```

- [ ] **Step 7: 父 pom 注册 module + dependencyManagement**

```xml
        <module>business_packages/sanyan-llm-api</module>
```

```xml
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-llm-api</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-llm-core</artifactId><version>${project.version}</version></dependency>
```

- [ ] **Step 8: 跑 compile 验证**

```bash
mvn -pl business_packages/sanyan-llm-api compile
```

预期：`BUILD SUCCESS`

- [ ] **Step 9: 不 commit，跟 Task 3.2 一起**

---

### Task 3.2: 新建 sanyan-llm-core 模块（含 LlmApiImpl + 改 LlmTaskType 引用）

**Files:**
- Create: `business_packages/sanyan-llm-core/pom.xml`
- Move: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/{DoubaoAdapter,DeepSeekAdapter,LLMProvider,LLMProviderRouter,LlmErrCode}.java` → `business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/`
- **保留在 sanyan-business**（暂时）: `AiService.java`, `PromptBuilder.java`, `EmbeddingProvider.java`, `SiliconFlowEmbeddingProvider.java`, `LLMTaskType.java`（Phase 5 chat 拆开时再迁 AiService/PromptBuilder；Phase 4 拆 embedding；LLMTaskType 本 Task 删，引用方改用 com.sanyan.llm.LlmTaskType）
- Create: `business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/api/LlmApiImpl.java`
- Create: `business_packages/sanyan-llm-core/src/test/java/com/sanyan/llm/LlmApplicationContextIT.java`
- Modify: 所有引用 `com.sanyan.llm.internal.LLMTaskType` 和 `LLMProviderRouter` 的地方改成 `LlmApi` / `LlmTaskType`（来自 -api）
- Modify: `business_packages/sanyan-memory-core/pom.xml`（加 sanyan-llm-api 依赖）
- Modify: `business_packages/sanyan-business/pom.xml`（加 sanyan-llm-api 依赖）
- Modify: `bootstrap/pom.xml`（加 sanyan-llm-core 依赖）

- [ ] **Step 1: 父 pom 加 llm-core 模块声明**

```xml
        <module>business_packages/sanyan-llm-api</module>
        <module>business_packages/sanyan-llm-core</module>
```

- [ ] **Step 2: 创建 llm-core/pom.xml**

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
    <artifactId>sanyan-llm-core</artifactId>
    <name>sanyan-llm-core</name>
    <packaging>jar</packaging>
    <dependencies>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-llm-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-error</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-web</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
        <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId></dependency>

        <!-- 测试 -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>com.squareup.okhttp3</groupId><artifactId>mockwebserver</artifactId><scope>test</scope></dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

确认父 pom 的 dependencyManagement 是否有 `mockwebserver` 版本，没有的话本 Task 暂跳过新增测试，等 Phase 验证 OK 后再补；如有，直接复用。

- [ ] **Step 3: 物理迁移 5 个 internal 类**

```bash
mkdir -p business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal

git mv business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/DoubaoAdapter.java \
       business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/

git mv business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/DeepSeekAdapter.java \
       business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/

git mv business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LLMProvider.java \
       business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/

git mv business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LLMProviderRouter.java \
       business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/

git mv business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LlmErrCode.java \
       business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/
```

同样迁对应测试：

```bash
mkdir -p business_packages/sanyan-llm-core/src/test/java/com/sanyan/llm/internal
git mv business_packages/sanyan-business/src/test/java/com/sanyan/llm/internal/DoubaoAdapterTest.java \
       business_packages/sanyan-llm-core/src/test/java/com/sanyan/llm/internal/
git mv business_packages/sanyan-business/src/test/java/com/sanyan/llm/internal/DeepSeekAdapterTest.java \
       business_packages/sanyan-llm-core/src/test/java/com/sanyan/llm/internal/ 2>/dev/null || true
git mv business_packages/sanyan-business/src/test/java/com/sanyan/llm/internal/LLMProviderRouterTest.java \
       business_packages/sanyan-llm-core/src/test/java/com/sanyan/llm/internal/
```

- [ ] **Step 4: 删旧的 LLMTaskType + 让所有引用方用 com.sanyan.llm.LlmTaskType**

```bash
# 先找所有引用
rg "com\.sanyan\.llm\.internal\.LLMTaskType" -l

# 把这些文件里的 import 改成 com.sanyan.llm.LlmTaskType；
# 把使用处的 LLMTaskType 改成 LlmTaskType（大小写有差异，原来是大写 LLM，新规约小写）
```

每个引用方手动 Edit。预期受影响的文件包括：
- `business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageService.java`（如果调 LLM）
- `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/AiService.java`（仍在 business 内）
- `business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/summary/MemorySummaryService.java`
- `business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/profile/MemoryProfileRefreshService.java`
- 新搬过来的 `DoubaoAdapter` / `DeepSeekAdapter` / `LLMProvider` / `LLMProviderRouter`

注意：搬到 llm-core 的 5 个 internal 类内部也用 LLMTaskType——它们改用 `com.sanyan.llm.LlmTaskType`。

最后删 `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LLMTaskType.java`（已迁移到 llm-api）。

- [ ] **Step 5: 同样把 LLMProvider 接口的类名是否大小写**

如果 `LLMProvider` 内部用了 `LLMTaskType.values()` 或 `supports(LLMTaskType)`，把签名改成 `supports(LlmTaskType)`。这是接口签名变化，5 个实现类（DoubaoAdapter / DeepSeekAdapter 等）都要同步改。

- [ ] **Step 6: 写 LlmApiImpl（委托 LLMProviderRouter）**

`business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/api/LlmApiImpl.java`:

```java
package com.sanyan.llm.api;

import com.sanyan.llm.LlmApi;
import com.sanyan.llm.LlmTaskType;
import com.sanyan.llm.dto.ChatMessage;
import com.sanyan.llm.internal.LLMProviderRouter;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class LlmApiImpl implements LlmApi {

    private final LLMProviderRouter router;

    @Override
    public String chat(LlmTaskType taskType, List<ChatMessage> messages) {
        // 把 ChatMessage record 转 Map<role, content>——原 router/provider 用 Map 表示消息
        List<Map<String, String>> raw = messages.stream()
                .map(m -> Map.of("role", m.role(), "content", m.content()))
                .toList();
        return router.route(taskType).chat(raw);
    }
}
```

注意：实际 `LLMProviderRouter` 的方法签名以现有代码为准。如果它原本就是 `routeAndChat(LlmTaskType, List<Map>)` 一站式，直接调即可。

- [ ] **Step 7: 写 LlmApplicationContextIT**

```java
package com.sanyan.llm;

import com.sanyan.common.test.TestApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.AutoConfigureTestDatabase;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(classes = TestApplication.class)
@AutoConfigureTestDatabase
class LlmApplicationContextIT {

    @Autowired private LlmApi llmApi;

    @Test
    void contextLoads_llmApiBeanInjected() {
        assertThat(llmApi).isNotNull();
    }
}
```

- [ ] **Step 8: 改 memory-core 的 import**

```bash
rg "com\.sanyan\.llm\.internal\." business_packages/sanyan-memory-core/ -l
```

预期受影响：
- `MemorySummaryService.java`
- `MemoryProfileRefreshService.java`
- `MemoryRagSearchService.java`（用 EmbeddingProvider，Phase 4 才处理；本 Step 仅处理 LLM 相关的）

把 `import com.sanyan.llm.internal.LLMProviderRouter;` 改成 `import com.sanyan.llm.LlmApi;`，使用处替换。

⚠️ **PromptBuilder 的处理**：memory-core 现在用 `com.sanyan.llm.internal.PromptBuilder` 拼装 messages。Phase 3 不动 PromptBuilder 本身，因为它要等到 Phase 5 chat 拆出来时再决定归属（chat 自己用的 PromptBuilder 留 chat-core；memory 自己的拼装在本 Phase 直接 inline 写成 ChatMessage 列表）。

**本 Step 的临时处理**：memory-core 仍 import `com.sanyan.llm.internal.PromptBuilder`，但因为 PromptBuilder 还在 sanyan-business 内（暂未迁），sanyan-memory-core/pom.xml 已经有 `sanyan-business` 依赖（Plan 2 妥协），import 仍能解析。**memory-core 改 import 只针对 LLMProviderRouter → LlmApi 这一项；PromptBuilder 留到 Phase 5 处理**。

具体方法：

| 旧 | 新 |
|---|---|
| `import com.sanyan.llm.internal.LLMProviderRouter;` | `import com.sanyan.llm.LlmApi;` |
| `import com.sanyan.llm.internal.LLMTaskType;` | `import com.sanyan.llm.LlmTaskType;` |
| `private final LLMProviderRouter llmRouter;` | `private final LlmApi llmApi;` |
| `llmRouter.route(LLMTaskType.BACKGROUND).chat(messages)` | `llmApi.chat(LlmTaskType.BACKGROUND, chatMessages)` ← 但 chatMessages 是 List&lt;ChatMessage&gt; 不是 List&lt;Map&gt;，需要先转换 |

如果 PromptBuilder 当前返回 `List<Map<String,String>>`，包一个临时 adapter：

```java
// 临时桥：在 memory-core 改 import 完成前，先把 List<Map> 转成 List<ChatMessage>
List<ChatMessage> chatMessages = rawMessages.stream()
        .map(m -> new ChatMessage(m.get("role"), m.get("content")))
        .toList();
String reply = llmApi.chat(LlmTaskType.BACKGROUND, chatMessages);
```

Phase 5 PromptBuilder 处理时再清理。

- [ ] **Step 9: 改 memory-core/pom.xml 加 sanyan-llm-api 依赖**

```xml
        <!-- S3 Phase 3：调 LlmApi（chat 推理） -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-llm-api</artifactId></dependency>
```

- [ ] **Step 10: 改 sanyan-business 内 chat 包的 LLM 相关 import**

`business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageService.java` 如果调 `AiService`（不调 LLMProviderRouter），AiService 还在 sanyan-business 内（Phase 5 才拆），所以这一步可能没改动。

但 `AiService` 自己 import `LLMProviderRouter` 要改 import 路径（router 已迁到 sanyan-llm-core）。**这种跨模块 internal 引用违反"不依赖其他 -core/internal"原则**，但 AiService 还在 sanyan-business 里临时存在，需要让 sanyan-business 引 sanyan-llm-core？

> ⚠️ 决策：**让 sanyan-business 临时引 sanyan-llm-core 是个临时违规，会被 Phase 7 ArchUnit 抓到——但 Phase 5 之前 AiService 还在 business 内，必须如此**。把 sanyan-business 当成"AiService 的临时收容所"，不要在它内部 import sanyan-llm-core/internal，而是改成调 LlmApi：

`AiService` 改用 `LlmApi`：

```java
// 旧
private final LLMProviderRouter llmRouter;
String reply = llmRouter.route(LLMTaskType.USER_FACING).chat(messages);

// 新
private final LlmApi llmApi;
String reply = llmApi.chat(LlmTaskType.USER_FACING, chatMessages);
```

让 sanyan-business/pom.xml 加 sanyan-llm-api 依赖（不引 sanyan-llm-core）：

```xml
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-llm-api</artifactId></dependency>
```

- [ ] **Step 11: bootstrap/pom.xml 加 sanyan-llm-core 依赖**

```xml
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-llm-core</artifactId></dependency>
```

- [ ] **Step 12: grep 确认 llm.internal 外部引用清零（除本模块外）**

```bash
rg "com\.sanyan\.llm\.internal\." business_packages/sanyan-business/ business_packages/sanyan-memory-core/
```

预期：只有 sanyan-business 内 AiService / PromptBuilder / EmbeddingProvider / SiliconFlowEmbeddingProvider 等还没迁的类自己内部 import 同包，**外部模块零命中**。

- [ ] **Step 13: 跑全量 verify**

```bash
mvn verify
```

预期：`BUILD SUCCESS`

- [ ] **Step 14: Commit Phase 3**

```bash
git add -A
git commit -m "refactor(modules): 拆 sanyan-llm-api/core（Phase 3 / 7）

- 新建 sanyan-llm-api（LlmApi + LlmTaskType + ChatMessage record）
- 新建 sanyan-llm-core（迁 LLMProviderRouter / DoubaoAdapter / DeepSeekAdapter / LLMProvider / LlmErrCode）
- memory-core 改 import：LLMProviderRouter → LlmApi
- AiService 仍在 sanyan-business（Phase 5 拆 chat 时再迁），也改走 LlmApi 不直接引 router
- LLMTaskType → LlmTaskType（命名规范化）
- PromptBuilder + EmbeddingProvider + SiliconFlowEmbeddingProvider 留 sanyan-business（Phase 4/5 处理）
- 业务逻辑零变化"
```

---

## Phase 4 · sanyan-embedding-api/core

### Task 4.1: 新建 sanyan-embedding-api 模块

**Files:**
- Create: `business_packages/sanyan-embedding-api/pom.xml`
- Create: `business_packages/sanyan-embedding-api/src/main/java/com/sanyan/embedding/EmbeddingApi.java`
- Create: `business_packages/sanyan-embedding-api/src/main/java/com/sanyan/embedding/package-info.java`
- Modify: `pom.xml`（父 pom 加 module 声明 + dependencyManagement）

- [ ] **Step 1: 创建 pom.xml**

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
    <artifactId>sanyan-embedding-api</artifactId>
    <name>sanyan-embedding-api</name>
    <packaging>jar</packaging>
    <dependencies>
        <!-- 零依赖 -->
    </dependencies>
</project>
```

- [ ] **Step 2: 创建 EmbeddingApi 接口**

```java
package com.sanyan.embedding;

import java.util.List;

/**
 * Embedding 域对内 API 契约。
 *
 * <p>实现委托给硅基流动 SiliconFlowEmbeddingProvider（BAAI/bge-m3，1024 维）。
 * 调用方拿到 List&lt;float[]&gt; 后自己持久化或检索。
 */
public interface EmbeddingApi {
    /**
     * 把文本批量转向量。
     *
     * @param texts 文本列表，每条不空
     * @return 与入参一一对应的 1024 维向量
     * @throws com.sanyan.common.error.BusinessException 上游 4xx 不重试 / 5xx 重试耗尽 / 网络失败
     */
    List<float[]> embed(List<String> texts);
}
```

- [ ] **Step 3: 创建 package-info.java**

```java
/** sanyan-embedding-api：Embedding 域对内契约。 */
package com.sanyan.embedding;
```

- [ ] **Step 4: 父 pom 注册**

```xml
        <module>business_packages/sanyan-embedding-api</module>
```

```xml
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-embedding-api</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-embedding-core</artifactId><version>${project.version}</version></dependency>
```

- [ ] **Step 5: compile 验证**

```bash
mvn -pl business_packages/sanyan-embedding-api compile
```

预期：`BUILD SUCCESS`

- [ ] **Step 6: 不 commit，跟 Task 4.2 一起**

---

### Task 4.2: 新建 sanyan-embedding-core 模块（迁 SiliconFlowEmbeddingProvider + 拆 EmbeddingErrCode）

**Files:**
- Create: `business_packages/sanyan-embedding-core/pom.xml`
- Move: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/SiliconFlowEmbeddingProvider.java` → `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/`
- Move: `business_packages/sanyan-business/src/test/java/com/sanyan/llm/internal/SiliconFlowEmbeddingProviderTest.java` → `business_packages/sanyan-embedding-core/src/test/java/com/sanyan/embedding/internal/`
- Delete: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/EmbeddingProvider.java`（接口已被 EmbeddingApi 取代）
- Create: `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/EmbeddingErrCode.java`
- Create: `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/api/EmbeddingApiImpl.java`
- Create: `business_packages/sanyan-embedding-core/src/test/java/com/sanyan/embedding/EmbeddingApplicationContextIT.java`
- Modify: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/LlmErrCode.java`（删 EMBEDDING_SERVICE_UNAVAILABLE 4004）
- Modify: `business_packages/sanyan-memory-core/`（改 EmbeddingProvider → EmbeddingApi import + 把 LlmErrCode.EMBEDDING_SERVICE_UNAVAILABLE 改成 EmbeddingErrCode.EMBEDDING_SERVICE_UNAVAILABLE）
- Modify: `business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/MemoryErrCode.java`（保留 5002 不动）
- Modify: 调用方对 LlmErrCode.EMBEDDING_SERVICE_UNAVAILABLE 的引用迁到 EmbeddingErrCode
- Modify: `business_packages/sanyan-memory-core/pom.xml`（加 sanyan-embedding-api 依赖）
- Modify: `bootstrap/pom.xml`（加 sanyan-embedding-core 依赖）
- Modify: `docs/superpowers/specs/2026-05-17-s3-business-modular-split-design.md`：无需改
- Modify: `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`（启用 6xxx 区间）

- [ ] **Step 1: 父 pom 加 embedding-core 模块声明**

```xml
        <module>business_packages/sanyan-embedding-api</module>
        <module>business_packages/sanyan-embedding-core</module>
```

- [ ] **Step 2: 创建 embedding-core/pom.xml**

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
    <artifactId>sanyan-embedding-core</artifactId>
    <name>sanyan-embedding-core</name>
    <packaging>jar</packaging>
    <dependencies>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-embedding-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-error</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-web</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter</artifactId></dependency>
        <dependency><groupId>org.springframework</groupId><artifactId>spring-web</artifactId></dependency>
        <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId></dependency>

        <!-- 测试 -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>com.squareup.okhttp3</groupId><artifactId>mockwebserver</artifactId><scope>test</scope></dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: 物理迁移 SiliconFlowEmbeddingProvider + 测试**

```bash
mkdir -p business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal
mkdir -p business_packages/sanyan-embedding-core/src/test/java/com/sanyan/embedding/internal

git mv business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/SiliconFlowEmbeddingProvider.java \
       business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/

git mv business_packages/sanyan-business/src/test/java/com/sanyan/llm/internal/SiliconFlowEmbeddingProviderTest.java \
       business_packages/sanyan-embedding-core/src/test/java/com/sanyan/embedding/internal/
```

- [ ] **Step 4: 改 SiliconFlowEmbeddingProvider package + import**

把 `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/SiliconFlowEmbeddingProvider.java` 的：

| 旧 | 新 |
|---|---|
| `package com.sanyan.llm.internal;` | `package com.sanyan.embedding.internal;` |
| `implements EmbeddingProvider` | `// 暂时去掉 implements；EmbeddingApiImpl 会包装它` |
| `import com.sanyan.llm.internal.LlmErrCode;` | `import com.sanyan.embedding.internal.EmbeddingErrCode;` |
| `LlmErrCode.EMBEDDING_SERVICE_UNAVAILABLE` | `EmbeddingErrCode.EMBEDDING_SERVICE_UNAVAILABLE` |
| 不再实现 `EmbeddingProvider`，但保留 `public List<float[]> embed(List<String> texts)` 签名供 ApiImpl 调用 |

测试文件同样改 package。

- [ ] **Step 5: 创建 EmbeddingErrCode（在 6xxx 区间）**

`business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/EmbeddingErrCode.java`:

```java
package com.sanyan.embedding.internal;

import com.sanyan.common.error.ErrCode;
import lombok.AllArgsConstructor;
import lombok.Getter;

/**
 * Embedding 域错误码。占用 6xxx 区间（详见 ERROR_CODE_REGISTRY.md）。
 *
 * <p>历史：S3 Phase 4 之前，embedding 失败用 LlmErrCode.EMBEDDING_SERVICE_UNAVAILABLE (4004)。
 * 拆 embedding 独立域后，错误码迁到本 enum，新 code 6001。
 * MemoryErrCode 自己的 5002 EMBEDDING_SERVICE_UNAVAILABLE（业务层降级视角）保留不变。
 */
@Getter
@AllArgsConstructor
public enum EmbeddingErrCode implements ErrCode {
    // 协议级失败：4xx 直接抛 / 5xx 重试耗尽 / 网络异常
    EMBEDDING_SERVICE_UNAVAILABLE(6001, "Embedding 服务不可用");

    private final int code;
    private final String defaultMessage;
}
```

- [ ] **Step 6: 写 EmbeddingApiImpl**

`business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/api/EmbeddingApiImpl.java`:

```java
package com.sanyan.embedding.api;

import com.sanyan.embedding.EmbeddingApi;
import com.sanyan.embedding.internal.SiliconFlowEmbeddingProvider;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class EmbeddingApiImpl implements EmbeddingApi {

    private final SiliconFlowEmbeddingProvider provider;

    @Override
    public List<float[]> embed(List<String> texts) {
        return provider.embed(texts);
    }
}
```

- [ ] **Step 7: 删旧 EmbeddingProvider 接口（已被 EmbeddingApi 取代）**

```bash
rm business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/EmbeddingProvider.java
```

- [ ] **Step 8: 改 memory-core 引用**

```bash
rg "EmbeddingProvider" business_packages/sanyan-memory-core/ -l
```

逐文件改：

| 旧 | 新 |
|---|---|
| `import com.sanyan.llm.internal.EmbeddingProvider;` | `import com.sanyan.embedding.EmbeddingApi;` |
| `private final EmbeddingProvider embeddingProvider;` | `private final EmbeddingApi embeddingApi;` |
| `embeddingProvider.embed(...)` | `embeddingApi.embed(...)` |

如果 memory-core 内有 catch `LlmErrCode.EMBEDDING_SERVICE_UNAVAILABLE`，加一条 catch `EmbeddingErrCode.EMBEDDING_SERVICE_UNAVAILABLE`（code 6001）。但更简单的做法：catch `BusinessException` 后看 `code == 6001 || code == 4004 || code == 5002` 走相同降级——这种做法保留向后兼容，避免一次切换打破所有 fallback 测试。

实际上，因为 LlmErrCode.EMBEDDING_SERVICE_UNAVAILABLE 本 Phase 要删（Step 9），最干净的写法是 catch `BusinessException` 全部走降级，不管 errCode 是 6001 还是 5002——这是 MemoryRagSearchService 现有逻辑。

- [ ] **Step 9: 改 LlmErrCode 删 EMBEDDING_SERVICE_UNAVAILABLE 4004**

把 `business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/LlmErrCode.java` 中：

```java
EMBEDDING_SERVICE_UNAVAILABLE(4004, "Embedding 服务不可用"),
```

删掉（已迁到 EmbeddingErrCode 6001）。

- [ ] **Step 10: 改 memory-core/pom.xml 加 sanyan-embedding-api 依赖**

```xml
        <!-- S3 Phase 4：调 EmbeddingApi（向量化） -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-embedding-api</artifactId></dependency>
```

- [ ] **Step 11: 写 EmbeddingApplicationContextIT**

```java
package com.sanyan.embedding;

import com.sanyan.common.test.TestApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.AutoConfigureTestDatabase;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(classes = TestApplication.class)
@AutoConfigureTestDatabase
class EmbeddingApplicationContextIT {

    @Autowired private EmbeddingApi embeddingApi;

    @Test
    void contextLoads_embeddingApiBeanInjected() {
        assertThat(embeddingApi).isNotNull();
    }
}
```

- [ ] **Step 12: bootstrap/pom.xml 加 sanyan-embedding-core**

```xml
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-embedding-core</artifactId></dependency>
```

- [ ] **Step 13: ERROR_CODE_REGISTRY.md 更新**

修改 `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`：

把 6000-6999 行从"_（保留）_"改成 "embedding" + 表名 + 文件位置。

把 LlmErrCode 表里删 4004 那一行。

新增 EmbeddingErrCode 区段：

```markdown
### EmbeddingErrCode（6000-6999）

| Code | 常量 | 文案 |
|---|---|---|
| 6001 | `EMBEDDING_SERVICE_UNAVAILABLE` | Embedding 服务不可用（硅基流动 API 4xx / 5xx 重试耗尽 / 网络异常） |
```

历史变更追加一行：

```markdown
| 2026-05-17 | S3 Phase 4 启用 6xxx 区间（embedding 独立域）；LlmErrCode.EMBEDDING_SERVICE_UNAVAILABLE 4004 迁到 EmbeddingErrCode 6001 |
```

- [ ] **Step 14: 跑全量 verify**

```bash
mvn verify
```

预期：`BUILD SUCCESS` + `MemoryRagSearchServiceTest` 两条 fallback case 仍 PASS（因为它们 catch BusinessException 不看具体 code）

- [ ] **Step 15: Commit Phase 4**

```bash
git add -A
git commit -m "refactor(modules): 拆 sanyan-embedding-api/core（Phase 4 / 7）

- 新建 sanyan-embedding-api（EmbeddingApi 单接口）
- 新建 sanyan-embedding-core（迁 SiliconFlowEmbeddingProvider + 新 EmbeddingErrCode 6001 + EmbeddingApiImpl）
- 删旧 EmbeddingProvider 接口（已被 EmbeddingApi 取代）
- LlmErrCode 删 EMBEDDING_SERVICE_UNAVAILABLE 4004 迁到 EmbeddingErrCode 6001
- memory-core 改 EmbeddingProvider → EmbeddingApi
- ERROR_CODE_REGISTRY.md 启用 6xxx 区间 + 4004 退役
- 业务逻辑零变化"
```

---

## Phase 5 · sanyan-chat-api/core（最复杂节点）

### Task 5.1: 新建 sanyan-chat-api 模块

**Files:**
- Create: `business_packages/sanyan-chat-api/pom.xml`
- Create: `business_packages/sanyan-chat-api/src/main/java/com/sanyan/chat/ChatApi.java`
- Create: `business_packages/sanyan-chat-api/src/main/java/com/sanyan/chat/SenderType.java`（上提）
- Create: `business_packages/sanyan-chat-api/src/main/java/com/sanyan/chat/dto/MessageDto.java`
- Create: `business_packages/sanyan-chat-api/src/main/java/com/sanyan/chat/event/MessagePersistedEvent.java`（迁移）
- Create: `business_packages/sanyan-chat-api/src/main/java/com/sanyan/chat/package-info.java`
- Modify: `pom.xml`（父 pom 加 module 声明 + dependencyManagement）

- [ ] **Step 1: Read MessageEntity + 现有 SenderType + MessagePersistedEvent，了解字段**

```bash
cat business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/MessageEntity.java
cat business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/SenderType.java
cat business_packages/sanyan-business/src/main/java/com/sanyan/chat/event/MessagePersistedEvent.java
```

记录所有字段，DTO + event record 用。

- [ ] **Step 2: 创建 pom.xml**

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
    <artifactId>sanyan-chat-api</artifactId>
    <name>sanyan-chat-api</name>
    <packaging>jar</packaging>
    <dependencies>
        <!-- 零依赖 -->
    </dependencies>
</project>
```

- [ ] **Step 3: 创建 SenderType（上提到顶层 com.sanyan.chat）**

复制 `business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/SenderType.java` 的内容，改 package 为 `com.sanyan.chat`，放到 `business_packages/sanyan-chat-api/src/main/java/com/sanyan/chat/SenderType.java`。

- [ ] **Step 4: 创建 MessageDto record**

```java
package com.sanyan.chat.dto;

import com.sanyan.chat.SenderType;

import java.time.Instant;

/**
 * Message 跨模块 DTO。memory-core 等业务通过 ChatApi 拿这种结构而不是 MessageEntity。
 *
 * <p>字段对应 MessageEntity（id / userId / characterId / senderType / content / createdAt）。
 * audioUrl 等附加字段如有跨模块场景，按需扩展。
 */
public record MessageDto(
        Long id,
        Long userId,
        Long characterId,
        SenderType senderType,
        String content,
        Instant createdAt) {}
```

> ⚠️ 字段名以实际 MessageEntity 为准。

- [ ] **Step 5: 创建/迁移 MessagePersistedEvent**

把 `business_packages/sanyan-business/src/main/java/com/sanyan/chat/event/MessagePersistedEvent.java` 内容复制到 `business_packages/sanyan-chat-api/src/main/java/com/sanyan/chat/event/MessagePersistedEvent.java`，改 package 为 `com.sanyan.chat.event`。

如果原 event 引用了 MessageEntity，改成只携带必要字段（id / userId / characterId / senderType）或携带 MessageDto。

- [ ] **Step 6: 创建 ChatApi 接口**

```java
package com.sanyan.chat;

import com.sanyan.chat.dto.MessageDto;
import org.springframework.data.domain.Pageable;

import java.util.List;

/**
 * Chat 域对内 API 契约。memory-core 等业务方通过本接口拿消息数据。
 *
 * <p>历史：S3 Phase 5 之前 memory-core 直接 import MessageEntity/Repository——
 * 违反 java-backend §3.3。Phase 5 拆 chat 域后，所有跨模块查询走本接口。
 */
public interface ChatApi {
    /** 按 id 查消息；不存在返回 null。 */
    MessageDto findById(Long messageId);

    /**
     * 查某用户跟某 character 的最近消息（按 createdAt DESC）。
     * 由 chat-core 内部用 Pageable 实现。
     */
    List<MessageDto> listRecent(Long userId, Long characterId, Pageable pageable);

    /** 统计某用户跟某 character 的消息总数。 */
    long countByUserAndCharacter(Long userId, Long characterId);
}
```

> ⚠️ 方法清单以 memory-core 实际用了 MessageRepository 哪些方法为准。grep MessageRepository.find 看一下实际接口。

- [ ] **Step 7: 创建 package-info.java**

```java
/** sanyan-chat-api：Chat 域对内契约。 */
package com.sanyan.chat;
```

- [ ] **Step 8: 父 pom 注册**

```xml
        <module>business_packages/sanyan-chat-api</module>
```

```xml
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-chat-api</artifactId><version>${project.version}</version></dependency>
            <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-chat-core</artifactId><version>${project.version}</version></dependency>
```

注意 `Pageable` 是 Spring Data API。如果 ChatApi 用它意味着 chat-api 需要依赖 spring-data-commons。**决策**：让 ChatApi 不暴露 Pageable，改用普通 `int limit`：

```java
List<MessageDto> listRecent(Long userId, Long characterId, int limit);
```

实现侧（ChatApiImpl）自己包成 `PageRequest.of(0, limit)`。这样保持 chat-api 零依赖。

修改 Step 6 ChatApi 接口去掉 Pageable 引用。

- [ ] **Step 9: compile 验证**

```bash
mvn -pl business_packages/sanyan-chat-api compile
```

- [ ] **Step 10: 不 commit，跟 Task 5.2 一起**

---

### Task 5.2: 新建 sanyan-chat-core 模块（迁 chat 包全部 + 迁 AiService/PromptBuilder + 改 memory-core import）

这是最复杂的 Task。建议执行时拆成 2 个子代理 session：
- Session 1：建 sanyan-chat-core 模块结构 + 物理迁移所有 chat 包 + 写 ChatApiImpl
- Session 2：改 memory-core 的 5 处 import（用 ChatApi + MessageDto）

**Files:**
- Create: `business_packages/sanyan-chat-core/pom.xml`
- Move: 全部 `business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal/**` + `web/**` + `ws/**` → `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/`
- Move: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/AiService.java` → `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/`
- Move: `business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/PromptBuilder.java` → `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/`
- Move: 对应测试文件
- Create: `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/api/ChatApiImpl.java`
- Create: `business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/ChatApplicationContextIT.java`
- Modify: memory-core 5 个文件（替换 chat.internal.* import）
- Modify: memory-core 自己写 PromptBuilder（summary / profile 各一份），不再用原 llm 的 PromptBuilder
- Modify: `business_packages/sanyan-memory-core/pom.xml`（加 sanyan-chat-api 依赖 + 删 sanyan-business 依赖）
- Modify: `bootstrap/pom.xml`（加 sanyan-chat-core 依赖）
- Delete: 旧 chat.internal.SenderType / event/MessagePersistedEvent（已上提到 chat-api）

- [ ] **Step 1: 父 pom 加 chat-core 模块声明**

```xml
        <module>business_packages/sanyan-chat-api</module>
        <module>business_packages/sanyan-chat-core</module>
```

- [ ] **Step 2: 创建 chat-core/pom.xml**

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
    <artifactId>sanyan-chat-core</artifactId>
    <name>sanyan-chat-core</name>
    <packaging>jar</packaging>
    <dependencies>
        <!-- 本域 -api -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-chat-api</artifactId></dependency>
        <!-- 调其他业务 -api -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-user-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-character-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-llm-api</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-memory-api</artifactId></dependency>
        <!-- foundation -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-error</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-web</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-auth</artifactId></dependency>
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-ws</artifactId></dependency>
        <!-- Spring -->
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-websocket</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-validation</artifactId></dependency>
        <dependency><groupId>org.postgresql</groupId><artifactId>postgresql</artifactId><scope>runtime</scope></dependency>
        <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId></dependency>

        <!-- 测试 -->
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-common-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>com.h2database</groupId><artifactId>h2</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.awaitility</groupId><artifactId>awaitility</artifactId><scope>test</scope></dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: 物理迁移 chat 包**

```bash
mkdir -p business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat

git mv business_packages/sanyan-business/src/main/java/com/sanyan/chat/internal \
       business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/

git mv business_packages/sanyan-business/src/main/java/com/sanyan/chat/web \
       business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/

git mv business_packages/sanyan-business/src/main/java/com/sanyan/chat/ws \
       business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/

git mv business_packages/sanyan-business/src/main/java/com/sanyan/chat/event \
       business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/
```

⚠️ `event/` 目录的 MessagePersistedEvent 已在 chat-api 创建了，**重复的**——本步搬过来后会冲突。处理：先 `rm` 删 sanyan-business 下的 event 包再搬：

```bash
rm -rf business_packages/sanyan-business/src/main/java/com/sanyan/chat/event/
# event 已在 chat-api 创建
```

或者 `git mv` 后再 `git rm` 删除 chat-core 多余的 event（用 chat-api 那份）。

- [ ] **Step 4: 删 chat-core/internal/SenderType.java（已上提到 chat-api）**

```bash
rm business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/SenderType.java
```

- [ ] **Step 5: 迁 AiService + PromptBuilder（从 llm 包搬到 chat-core/internal/）**

```bash
git mv business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/AiService.java \
       business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/

git mv business_packages/sanyan-business/src/main/java/com/sanyan/llm/internal/PromptBuilder.java \
       business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/
```

测试同理（如果有）：

```bash
git mv business_packages/sanyan-business/src/test/java/com/sanyan/llm/internal/AiServiceTest.java \
       business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/ 2>/dev/null || true
```

- [ ] **Step 6: 改 AiService + PromptBuilder + 其他 chat 包内类的 package + import**

把 `AiService.java` 和 `PromptBuilder.java`：
- `package com.sanyan.llm.internal;` → `package com.sanyan.chat.internal;`
- 所有 `import com.sanyan.chat.internal.SenderType;` → `import com.sanyan.chat.SenderType;`（上提到 chat-api 顶层）
- 所有 `import com.sanyan.chat.event.MessagePersistedEvent;` 不变（在 chat-api 同名包）

把 chat-core 内全部其他类（ChatWebSocketHandler / MessageController / MessageService 等）的 `import com.sanyan.chat.internal.SenderType;` 改成 `import com.sanyan.chat.SenderType;`。

- [ ] **Step 7: 写 ChatApiImpl**

`business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/api/ChatApiImpl.java`:

```java
package com.sanyan.chat.api;

import com.sanyan.chat.ChatApi;
import com.sanyan.chat.SenderType;
import com.sanyan.chat.dto.MessageDto;
import com.sanyan.chat.internal.MessageEntity;
import com.sanyan.chat.internal.MessageRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class ChatApiImpl implements ChatApi {

    private final MessageRepository repository;

    @Override
    public MessageDto findById(Long messageId) {
        return repository.findById(messageId).map(this::toDto).orElse(null);
    }

    @Override
    public List<MessageDto> listRecent(Long userId, Long characterId, int limit) {
        return repository
                .findByUserIdAndCharacterIdOrderByCreatedAtDesc(
                        userId, characterId, PageRequest.of(0, limit))
                .stream().map(this::toDto).toList();
    }

    @Override
    public long countByUserAndCharacter(Long userId, Long characterId) {
        return repository.countByUserIdAndCharacterId(userId, characterId);
    }

    private MessageDto toDto(MessageEntity e) {
        return new MessageDto(
                e.getId(),
                e.getUserId(),
                e.getCharacterId(),
                e.getSenderType(),
                e.getContent(),
                e.getCreatedAt()
        );
    }
}
```

> ⚠️ MessageRepository 的实际方法签名以 Read 结果为准。如果方法不叫 `findByUserIdAndCharacterIdOrderByCreatedAtDesc`，按实际改。

- [ ] **Step 8: 写 ChatApplicationContextIT**

```java
package com.sanyan.chat;

import com.sanyan.common.test.TestApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.AutoConfigureTestDatabase;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(classes = TestApplication.class)
@AutoConfigureTestDatabase
class ChatApplicationContextIT {

    @Autowired private ChatApi chatApi;

    @Test
    void contextLoads_chatApiBeanInjected() {
        assertThat(chatApi).isNotNull();
    }
}
```

- [ ] **Step 9: 改 memory-core 的 5 处 import（按文件逐个改）**

5 个文件清单（从 spec 第 5 段引）：
- `MemorySummaryService.java`
- `SummaryScheduler.java`
- `MemoryProfileRefreshService.java`
- `MessageEmbeddingIndexListener.java`
- `UserMessageProfileRefreshListener.java`
- `MemoryChunkBuilder.java`

每个文件按以下规则改：

| 旧 | 新 |
|---|---|
| `import com.sanyan.chat.internal.MessageEntity;` | `import com.sanyan.chat.dto.MessageDto;` |
| `import com.sanyan.chat.internal.MessageRepository;` | `import com.sanyan.chat.ChatApi;` |
| `import com.sanyan.chat.internal.SenderType;` | `import com.sanyan.chat.SenderType;` |
| `private final MessageRepository messageRepository;` | `private final ChatApi chatApi;` |
| `messageRepository.findById(...)` | `chatApi.findById(...)` |
| `messageRepository.findByUserId...(...)` | `chatApi.listRecent(userId, characterId, limit)` |
| 用 `MessageEntity` 类型变量 | 用 `MessageDto` 类型 |
| `entity.getContent()` 等 getter | `dto.content()` 等 record accessor |

- [ ] **Step 10: memory-core 自己写 PromptBuilder（不再依赖原 chat PromptBuilder）**

memory-core 中 `MemorySummaryService` 和 `MemoryProfileRefreshService` 之前用 `com.sanyan.llm.internal.PromptBuilder`（已迁到 chat-core/internal/）。但 memory-core 不能引 chat-core/internal。

方案：memory-core 自己写 `SummaryPromptBuilder` + `ProfilePromptBuilder` 在 `business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/`。

参考原 PromptBuilder 内部，对 summary / profile 任务的拼装规则各搬一份过来。

> ⚠️ 这是本 Task 最大的代码量。需要分别 Read 原 PromptBuilder 的 buildSummaryPrompt() / buildProfileExtractPrompt()（或类似方法）写到 memory-core 自己的 class 里。Phase 3 Step 8 已经为这一步打过桥（临时 List<Map> 转 List<ChatMessage>），现在 PromptBuilder 自己拥有，直接产出 `List<ChatMessage>` 传给 LlmApi。

- [ ] **Step 11: 改 memory-core/pom.xml**

加 sanyan-chat-api 依赖：

```xml
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-chat-api</artifactId></dependency>
```

**删 sanyan-business 依赖**（Plan 2 妥协的整包依赖现在不再需要）：

```xml
        <!-- 删掉这一段 -->
        <!--
        <dependency>
            <groupId>com.sanyan</groupId>
            <artifactId>sanyan-business</artifactId>
        </dependency>
        -->
```

- [ ] **Step 12: bootstrap/pom.xml 加 sanyan-chat-core**

```xml
        <dependency><groupId>com.sanyan</groupId><artifactId>sanyan-chat-core</artifactId></dependency>
```

- [ ] **Step 13: grep 确认外部模块对 chat.internal 引用清零**

```bash
rg "com\.sanyan\.chat\.internal\." business_packages/sanyan-memory-core/ business_packages/sanyan-business/
```

预期：空

- [ ] **Step 14: 跑全量 verify**

```bash
mvn verify
```

预期：`BUILD SUCCESS`，含 Plan2EndToEndIT / MemoryRagSearchServiceIT 全部 PASS

- [ ] **Step 15: Commit Phase 5**

```bash
git add -A
git commit -m "refactor(modules): 拆 sanyan-chat-api/core（Phase 5 / 7）

- 新建 sanyan-chat-api（ChatApi 接口 + MessageDto record + SenderType 上提 + event/MessagePersistedEvent）
- 新建 sanyan-chat-core（迁 chat 全部 + AiService + PromptBuilder 从 llm 迁过来）
- memory-core 改 5 处 import：MessageEntity/Repository/SenderType → ChatApi/MessageDto
- memory-core 自己写 SummaryPromptBuilder + ProfilePromptBuilder（不再用 chat 的 PromptBuilder）
- memory-core/pom.xml 删 sanyan-business 整包依赖（Plan 2 妥协结束）
- 业务逻辑零变化"
```

---

## Phase 6 · 删 sanyan-business 空壳 + 配置类下沉 foundation

### Task 6.1: 把 config 类下沉到 foundation 再删 sanyan-business 模块

**Files:**
- Move: `business_packages/sanyan-business/src/main/java/com/sanyan/config/WebMvcConfig.java` → `foundation_packages/sanyan-common-web/src/main/java/com/sanyan/common/web/config/WebMvcConfig.java`
- Move: `business_packages/sanyan-business/src/main/java/com/sanyan/config/WebSocketConfig.java` → `foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/config/WebSocketConfig.java`
- Delete: `business_packages/sanyan-business/` 整个目录
- Modify: `pom.xml`（删 sanyan-business module 声明 + dependencyManagement）
- Modify: `bootstrap/pom.xml`（删 sanyan-business 依赖）

- [ ] **Step 1: 看 sanyan-business 残留**

```bash
find business_packages/sanyan-business/src -type f
```

预期：只剩 `config/WebMvcConfig.java` + `config/WebSocketConfig.java`（外加 pom.xml）

如果有别的，先评估是否漏迁。

- [ ] **Step 2: 物理迁移 WebMvcConfig**

```bash
mkdir -p foundation_packages/sanyan-common-web/src/main/java/com/sanyan/common/web/config

git mv business_packages/sanyan-business/src/main/java/com/sanyan/config/WebMvcConfig.java \
       foundation_packages/sanyan-common-web/src/main/java/com/sanyan/common/web/config/
```

改 package：`package com.sanyan.config;` → `package com.sanyan.common.web.config;`

如果 WebMvcConfig 引用了 sanyan-business 内的拦截器，那个拦截器可能在 sanyan-common-auth（如果是 token 拦截器）；按实际情况调整 import。

- [ ] **Step 3: 物理迁移 WebSocketConfig**

```bash
mkdir -p foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/config

git mv business_packages/sanyan-business/src/main/java/com/sanyan/config/WebSocketConfig.java \
       foundation_packages/sanyan-common-ws/src/main/java/com/sanyan/common/ws/config/
```

改 package：`package com.sanyan.config;` → `package com.sanyan.common.ws.config;`

如果 WebSocketConfig 注册的是 `chat.ws.ChatWebSocketHandler`（在 chat-core 内），common-ws 不能依赖 chat-core——这是循环依赖。处理：

**方案**：让 common-ws 提供 `WebSocketRegistrar` 接口（无依赖），chat-core 的 ChatWebSocketHandler 实现这个接口；WebSocketConfig 用 `@Autowired List<WebSocketRegistrar>` 拿所有实现注册。**这一步实施复杂，如果绕不开就让 WebSocketConfig 留在 chat-core/web/，不下沉**。

更现实的方案：先把 WebSocketConfig 留在 chat-core/web/，不下沉，只下沉 WebMvcConfig。这样 spec 第 8 节 DoD 调整一下（仅 WebMvcConfig 下沉，WebSocketConfig 留 chat-core/web/）。

- [ ] **Step 4: 删 sanyan-business 整个目录**

```bash
git rm -r business_packages/sanyan-business/
```

- [ ] **Step 5: 父 pom 删 sanyan-business module 声明 + dependencyManagement**

修改 `pom.xml`：删 `<module>business_packages/sanyan-business</module>` 行 + `<artifactId>sanyan-business</artifactId>` 行。

- [ ] **Step 6: bootstrap/pom.xml 删 sanyan-business 依赖**

如果还有 `<dependency>sanyan-business</dependency>`，删之。bootstrap 现在应该依赖 6 个 -core。

- [ ] **Step 7: grep 全工程对 sanyan-business 的引用确认清零**

```bash
rg "sanyan-business" --type pom
rg "sanyan\.config" --type java
```

预期：空（或仅在 ERROR_CODE_REGISTRY.md 等文档历史变更里出现）

- [ ] **Step 8: 跑全量 verify**

```bash
mvn verify
```

预期：`BUILD SUCCESS`

- [ ] **Step 9: Commit Phase 6**

```bash
git add -A
git commit -m "refactor(modules): 删 sanyan-business 空壳 + 配置类下沉 foundation（Phase 6 / 7）

- WebMvcConfig 迁到 sanyan-common-web/config/
- WebSocketConfig 决策：保留在 sanyan-chat-core/web/（避免 common-ws → chat-core 循环依赖）
- 删 sanyan-business 整个 maven 模块
- 父 pom 删 sanyan-business module 声明 + dependencyManagement
- bootstrap 不再依赖 sanyan-business
- 业务逻辑零变化"
```

---

## Phase 7 · ArchUnit 升级 + Enforcer 加守护 + 文档同步

### Task 7.1: 升级 ArchUnit 规则

**Files:**
- Modify: `bootstrap/src/test/java/com/sanyan/ArchitectureTest.java`

- [ ] **Step 1: 写新规则（暂时让旧规则共存）**

修改 `bootstrap/src/test/java/com/sanyan/ArchitectureTest.java`，在末尾新增：

```java
    // S3 Phase 7 新规则：任何 -core 不依赖其他 -core 的 internal/
    @ArchTest
    static final ArchRule core_internal_must_not_be_imported_across_business_domains =
        noClasses().that().resideInAnyPackage(
                "com.sanyan.user..",
                "com.sanyan.character..",
                "com.sanyan.chat..",
                "com.sanyan.llm..",
                "com.sanyan.embedding..",
                "com.sanyan.memory..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "com.sanyan.user.internal..",
                "com.sanyan.character.internal..",
                "com.sanyan.chat.internal..",
                "com.sanyan.llm.internal..",
                "com.sanyan.embedding.internal..",
                "com.sanyan.memory.internal..")
            .andShould().notHaveSimpleNameEndingWith("Test")
            .andShould().notHaveSimpleNameEndingWith("IT");
```

> ⚠️ 这个规则会限制"任何业务包不能跨包 import 别人的 internal/"——但允许"本包内的 internal/ 被本包内其他子包用"，需要 ArchUnit 的更细规则。如果上面规则误伤本包内调用，改成针对**每一对**业务的单独规则：

```java
@ArchTest
static final ArchRule memory_must_not_depend_on_chat_internal =
    noClasses().that().resideInAPackage("com.sanyan.memory..")
        .should().dependOnClassesThat().resideInAPackage("com.sanyan.chat.internal..");

@ArchTest
static final ArchRule chat_must_not_depend_on_user_internal =
    noClasses().that().resideInAPackage("com.sanyan.chat..")
        .should().dependOnClassesThat().resideInAPackage("com.sanyan.user.internal..");

// ... 6 个域两两组合写规则，共 30 条
```

更优雅做法：用 ArchUnit 的 layered architecture API：

```java
@ArchTest
static final ArchRule layered_business_architecture =
    layeredArchitecture()
        .consideringAllDependencies()
        .layer("UserApi").definedBy("com.sanyan.user..")
        .layer("CharacterApi").definedBy("com.sanyan.character..")
        // ...
        .whereLayer("UserInternal").mayOnlyBeAccessedByLayers("UserApi")
        // ...
```

实施时按实际情况选最简形式。

- [ ] **Step 2: 删旧规则（业务域不依赖 web/ws）**

把 `chat_must_not_depend_on_other_domain_web_or_ws` / `user_must_not_depend_on_other_domain_web_or_ws` 等老规则删掉——因为新模块结构下，这些是 Maven 边界物理隔离了。

- [ ] **Step 3: 跑 ArchitectureTest 验证新规则通过**

```bash
mvn -pl bootstrap test -Dtest=ArchitectureTest
```

预期：`Tests run: N, Failures: 0`

- [ ] **Step 4: 不 commit，跟 Task 7.2 / 7.3 一起**

---

### Task 7.2: 父 pom 加 Enforcer 规则（任何 -core 不依赖另一个 -core）

**Files:**
- Modify: `pom.xml`

- [ ] **Step 1: 父 pom 加 banned-dependencies 规则**

在父 `pom.xml` 的 `<plugin>` `<artifactId>maven-enforcer-plugin</artifactId>` 段加 execution：

```xml
                    <execution>
                        <id>enforce-no-core-to-core-dep</id>
                        <goals><goal>enforce</goal></goals>
                        <configuration>
                            <rules>
                                <bannedDependencies>
                                    <excludes>
                                        <exclude>com.sanyan:*-core</exclude>
                                    </excludes>
                                    <includes>
                                        <include>com.sanyan:${project.artifactId}</include>
                                    </includes>
                                </bannedDependencies>
                            </rules>
                        </configuration>
                    </execution>
```

> ⚠️ 这个规则的语义：在每个模块 build 时检查它的依赖图，**不允许引** `com.sanyan:*-core`，**除非**那个 `*-core` 就是自己（`${project.artifactId}` 自动展开成本模块名，自己引自己当然允许）。

bootstrap 模块例外：它依赖所有 -core 是正常的。解决方案：在 bootstrap/pom.xml 里加 plugin 跳过规则：

```xml
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-enforcer-plugin</artifactId>
                <executions>
                    <execution>
                        <id>enforce-no-core-to-core-dep</id>
                        <phase>none</phase>  <!-- bootstrap 跳过本规则 -->
                    </execution>
                </executions>
            </plugin>
```

- [ ] **Step 2: 跑 mvn validate 验证规则有效**

```bash
mvn validate
```

预期：`BUILD SUCCESS`。如果有违规，会在 validate 阶段直接报错。

- [ ] **Step 3: 不 commit，跟 Task 7.3 一起**

---

### Task 7.3: ERROR_CODE_REGISTRY.md 同步位置字段 + spec doc 标注完成

**Files:**
- Modify: `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`

- [ ] **Step 1: 更新 ERROR_CODE_REGISTRY.md 每个 ErrCode 的位置字段**

把表头里的"位置"列改成新模块路径：

| 区间 | 模块 | 类名 | 位置 |
|---|---|---|---|
| 400-499 | 通用 | `CommonErrCode` | `foundation_packages/sanyan-common-error/` |
| 1000-1999 | user | `UserErrCode` | `business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/` |
| 2000-2999 | chat | `ChatErrCode` | `business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/` |
| 3000-3999 | character | `CharacterErrCode` | `business_packages/sanyan-character-core/src/main/java/com/sanyan/character/internal/` |
| 4000-4999 | llm | `LlmErrCode` | `business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/` |
| 5000-5999 | memory | `MemoryErrCode` | `business_packages/sanyan-memory-core/src/main/java/com/sanyan/memory/internal/` |
| 6000-6999 | embedding | `EmbeddingErrCode` | `business_packages/sanyan-embedding-core/src/main/java/com/sanyan/embedding/internal/` |
| 7000-9999 | _（保留）_ | — | 给未来新业务模块 |

历史变更追加：

```markdown
| 2026-05-17 | S3 Phase 7：sanyan-business 单体拆完，ErrCode 位置全部更新到新 -core 模块路径 |
```

- [ ] **Step 2: 跑全量 verify 最后一次**

```bash
mvn verify
```

预期：`BUILD SUCCESS` + 所有测试通过 + ArchUnit + Enforcer 双重守护

- [ ] **Step 3: 本机 dogfood 4 个场景**

```bash
./scripts/dogfood/run_dogfood.sh
```

预期：profile / throttle / summary / rag 4 个场景全部 PASS

- [ ] **Step 4: Commit Phase 7**

```bash
git add -A
git commit -m "refactor(modules): ArchUnit + Enforcer 守护 + 文档同步（Phase 7 / 7）

- ArchUnit 新增：任何 -core 不依赖其他 -core 包的 internal/
- ArchUnit 删除：旧的'业务域不依赖 web/ws' 规则（已被 Maven 边界物理隔离）
- 父 pom Enforcer 加 bannedDependencies：任何 *-core 不依赖另一个 *-core
- bootstrap 例外跳过该 Enforcer 规则
- ERROR_CODE_REGISTRY.md 更新每个 ErrCode 的位置到新模块路径
- 本机 dogfood 4 场景全过
- S3 拆模块全部完成（Phase 1-7 共 7 个 commit）"
```

- [ ] **Step 5: push 到远程并 merge 到 master**

```bash
git push origin s3-modular-split

# 切到 dev 合并
git checkout dev
git merge --no-ff s3-modular-split -m "Merge branch 's3-modular-split' into dev：S3 sanyan-business 拆解 6 对 -api/-core"
git push origin dev

# 同步父 repo 子模块指针（在父 repo 里）
cd /Users/aventador/code/3yan
git checkout dev
git add server
git commit -m "chore: 更新 server 子模块引用（S3 sanyan-business 拆解完成）"
git push origin dev
```

⚠️ 是否 merge 到 master 由用户决定，本 Plan 不主动 push 到 master。

---

## Self-Review

### 1. Spec 覆盖

| Spec 节 | 对应 Plan Task | 状态 |
|---|---|---|
| §3.1 12 模块清单 | Phase 1-5 全部 Task | ✅ |
| §3.2 依赖白名单 | Phase 1-5 每个 Task 的 pom 都按白名单限定 | ✅ |
| §3.3 LlmApi 单接口 | Phase 3 Task 3.1 + Phase 4 拆出 embedding | ✅ |
| §3.4 PromptBuilder 不进 -api | Phase 3 Step 8 注 + Phase 5 Step 10 memory 自己写 | ✅ |
| §3.5 ErrCode 留 internal | 全部 Phase 的 ErrCode 都按此规则 | ✅ |
| §4 7 Phase | Phase 0-7 | ✅ |
| §5 子代理 3 步 | 本 Plan 顶部"子代理执行模式" + 各 Task 独立可派 | ✅ |
| §6 重构 TDD 节奏 | Task 0.1 / Task 1.2 等都按 Red→Green 写测试 | ✅ |
| §7 风险预案 | 各 Task 关键步骤都有 grep 验证 + 全量 mvn verify | ✅ |
| §8 DoD | Phase 7 Task 7.3 完成 dogfood + verify 全绿 + ArchUnit + Enforcer 守护 | ✅ |

### 2. Placeholder 扫

`grep -n "TBD\|TODO\|XXX\|按实际改\|按需扩展"` 全文检索本 plan 后，找到几处 "按实际改"（如字段名以 Read 结果为准）——这些不是 placeholder，是叮嘱实现子代理"先 Read 现状再下手"，合理保留。

无 TBD / 未填段。

### 3. 类型一致性

- `LlmTaskType` enum 在 Phase 3 创建于 `com.sanyan.llm` 顶层，所有 Phase 后续引用都用这个名（非 `LLMTaskType`）✅
- `ChatMessage` record 在 Phase 3 创建于 `com.sanyan.llm.dto`，Phase 5 memory-core 也用同一类 ✅
- `MessageDto` record 在 Phase 5 创建于 `com.sanyan.chat.dto`，memory-core 改 import 后用同一类 ✅
- `SenderType` enum 在 Phase 5 上提到 `com.sanyan.chat`（不再是 internal），memory-core 和 chat-core 内的 ws DTO 都引同一份 ✅

### 4. Type mismatches 检查

ChatApi 第 6 步 Pageable → 第 8 步改成 int limit 自洽 ✅

### Plan 不存在的虚假 method

- LLMProviderRouter 的 `route(LlmTaskType)` 方法是否真存在？需 Read 验证；不存在则 Task 3.2 Step 6 写 ApiImpl 时按实际方法签名调整
- MessageRepository 的 `findByUserIdAndCharacterIdOrderByCreatedAtDesc` 方法签名？同样需 Read 验证
- LlmErrCode 的 `EMBEDDING_SERVICE_UNAVAILABLE` 在硅基切换时已经存在；Phase 4 Step 9 删之合理

实施时**实现子代理必须先 Read 相关现有类的实际接口，再下手改/写 ApiImpl**。

---

## Execution Handoff

Plan 完成并保存到 `docs/superpowers/plans/2026-05-17-s3-business-modular-split-plan.md`。

7 个 Phase × 平均 2-3 Task = 共 ~15 个 Task。建议拆 3 个 session：
- **Session 1**：Phase 0-3（Task 0.1 / 1.1 / 1.2 / 2.1 / 2.2 / 3.1 / 3.2）= ~7 Task
- **Session 2**：Phase 4-5（Task 4.1 / 4.2 / 5.1 / 5.2）= ~4 Task
- **Session 3**：Phase 6-7（Task 6.1 / 7.1 / 7.2 / 7.3）= ~4 Task

按 superpowers 规范，每个 Task 派 3 个子代理依次执行（实现 → spec 审 → code 审）。
