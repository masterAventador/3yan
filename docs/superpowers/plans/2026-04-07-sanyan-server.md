# 三言后端 (sanyan-server) 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建三言 AI 陪伴聊天 App 的 Spring Boot 后端，包含用户认证、WebSocket 实时通信、豆包 AI 对话、记忆系统和主动消息。

**Architecture:** 单体 Spring Boot 应用，WebSocket 处理实时消息，REST API 处理业务接口。MySQL 存储持久化数据，Redis 管理会话缓存和在线状态。豆包 doubao-seed-character 模型提供 AI 对话能力。

**Tech Stack:** Java 17, Spring Boot 3.2, Spring WebSocket, Spring Data JPA, MySQL 8, Redis, JJWT, 阿里云短信 SDK, 极光推送 SDK

---

## 项目文件结构

```
sanyan-server/
├── pom.xml
├── src/main/java/com/sanyan/
│   ├── SanyanApplication.java
│   ├── config/
│   │   ├── WebSocketConfig.java
│   │   ├── SecurityConfig.java
│   │   └── RedisConfig.java
│   ├── controller/
│   │   ├── AuthController.java
│   │   ├── UserController.java
│   │   ├── ConversationController.java
│   │   ├── CharacterController.java
│   │   └── DeviceController.java
│   ├── service/
│   │   ├── AuthService.java
│   │   ├── UserService.java
│   │   ├── SmsService.java
│   │   ├── MessageService.java
│   │   ├── AiService.java
│   │   ├── MemoryService.java
│   │   ├── ProactiveService.java
│   │   └── PushService.java
│   ├── websocket/
│   │   ├── WebSocketHandler.java
│   │   ├── WebSocketInterceptor.java
│   │   └── SessionManager.java
│   ├── entity/
│   │   ├── User.java
│   │   ├── UserToken.java
│   │   ├── AiCharacter.java
│   │   ├── Conversation.java
│   │   ├── Message.java
│   │   ├── MemoryProfile.java
│   │   └── MemorySummary.java
│   ├── repository/
│   │   ├── UserRepository.java
│   │   ├── UserTokenRepository.java
│   │   ├── AiCharacterRepository.java
│   │   ├── ConversationRepository.java
│   │   ├── MessageRepository.java
│   │   ├── MemoryProfileRepository.java
│   │   └── MemorySummaryRepository.java
│   ├── dto/
│   │   ├── req/
│   │   │   ├── SmsSendReq.java
│   │   │   ├── RegisterReq.java
│   │   │   ├── LoginReq.java
│   │   │   ├── PasswordResetReq.java
│   │   │   └── ProfileUpdateReq.java
│   │   ├── data/
│   │   │   ├── LoginData.java
│   │   │   ├── UserProfileData.java
│   │   │   ├── CharacterData.java
│   │   │   ├── ConversationData.java
│   │   │   └── MessageData.java
│   │   ├── ApiResponse.java
│   │   └── ws/
│   │       ├── WsMessage.java
│   │       ├── WsAck.java
│   │       ├── WsTyping.java
│   │       ├── WsNewMessage.java
│   │       └── WsSyncResult.java
│   ├── util/
│   │   └── JwtUtil.java
│   └── scheduler/
│       └── ProactiveScheduler.java
├── src/main/resources/
│   ├── application.yml
│   ├── application-dev.yml
│   └── data.sql
└── src/test/java/com/sanyan/
    ├── util/
    │   └── JwtUtilTest.java
    ├── service/
    │   ├── AuthServiceTest.java
    │   ├── MessageServiceTest.java
    │   ├── AiServiceTest.java
    │   ├── MemoryServiceTest.java
    │   └── ProactiveServiceTest.java
    ├── controller/
    │   ├── AuthControllerTest.java
    │   ├── UserControllerTest.java
    │   ├── ConversationControllerTest.java
    │   └── CharacterControllerTest.java
    └── websocket/
        └── WebSocketHandlerTest.java
```

---

### Task 1: 项目初始化 + 健康检查

**Files:**
- Create: `pom.xml`
- Create: `src/main/java/com/sanyan/SanyanApplication.java`
- Create: `src/main/resources/application.yml`
- Create: `src/main/resources/application-dev.yml`

- [ ] **Step 1: 使用 Spring Initializr 生成项目**

在 `server/` 目录下生成 Spring Boot 项目，或手动创建 `pom.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
    </parent>
    <groupId>com.sanyan</groupId>
    <artifactId>sanyan-server</artifactId>
    <version>0.1.0</version>
    <name>sanyan-server</name>

    <properties>
        <java.version>17</java.version>
        <jjwt.version>0.12.5</jjwt.version>
    </properties>

    <dependencies>
        <!-- Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <!-- WebSocket -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
        <!-- JPA + MySQL -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
        <!-- Redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <!-- Security (only for BCrypt) -->
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-crypto</artifactId>
        </dependency>
        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>${jjwt.version}</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: 创建主类和配置文件**

`SanyanApplication.java`:
```java
package com.sanyan;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class SanyanApplication {
    public static void main(String[] args) {
        SpringApplication.run(SanyanApplication.class, args);
    }
}
```

`application.yml`:
```yaml
spring:
  profiles:
    active: dev
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    default-property-inclusion: non_null

sanyan:
  jwt:
    secret: ${JWT_SECRET:your-256-bit-secret-key-for-sanyan-app-change-in-production}
    expiration-days: 30
```

`application-dev.yml`:
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/sanyan?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai&createDatabaseIfNotExist=true
    username: root
    password: ${MYSQL_PASSWORD:root}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
  data:
    redis:
      host: localhost
      port: 6379

server:
  port: 8080

sanyan:
  doubao:
    api-key: ${DOUBAO_API_KEY:}
    model: doubao-seed-character
    endpoint: https://ark.cn-beijing.volces.com/api/v3/chat/completions
  sms:
    access-key-id: ${SMS_ACCESS_KEY_ID:}
    access-key-secret: ${SMS_ACCESS_KEY_SECRET:}
    sign-name: 三言
    template-code: ${SMS_TEMPLATE_CODE:}
  push:
    app-key: ${JPUSH_APP_KEY:}
    master-secret: ${JPUSH_MASTER_SECRET:}
```

- [ ] **Step 3: 写健康检查测试**

```java
// src/test/java/com/sanyan/SanyanApplicationTest.java
package com.sanyan;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class SanyanApplicationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void healthCheck() throws Exception {
        mockMvc.perform(get("/api/health"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("ok"));
    }
}
```

- [ ] **Step 4: 运行测试，确认失败**

Run: `mvn test -pl . -Dtest=SanyanApplicationTest#healthCheck`
Expected: FAIL — 404 Not Found

- [ ] **Step 5: 实现健康检查端点 + 统一响应体**

`dto/ApiResponse.java`:
```java
package com.sanyan.dto;

import lombok.Data;

@Data
public class ApiResponse<T> {
    private boolean success;
    private String errMsg;
    private T data;

    public static <T> ApiResponse<T> ok(T data) {
        ApiResponse<T> resp = new ApiResponse<>();
        resp.setSuccess(true);
        resp.setData(data);
        return resp;
    }

    public static <T> ApiResponse<T> ok() {
        return ok(null);
    }

    public static <T> ApiResponse<T> fail(String errMsg) {
        ApiResponse<T> resp = new ApiResponse<>();
        resp.setSuccess(false);
        resp.setErrMsg(errMsg);
        return resp;
    }
}
```

`controller/HealthController.java`:
```java
package com.sanyan.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
public class HealthController {
    @GetMapping("/api/health")
    public Map<String, String> health() {
        return Map.of("status", "ok");
    }
}
```

- [ ] **Step 6: 运行测试，确认通过**

Run: `mvn test -Dtest=SanyanApplicationTest#healthCheck`
Expected: PASS

- [ ] **Step 7: 提交**

```bash
git add -A
git commit -m "项目初始化：Spring Boot 脚手架 + 健康检查端点"
git push
```

---

### Task 2: 数据库实体 + Repository

**Files:**
- Create: `entity/User.java`, `entity/UserToken.java`, `entity/AiCharacter.java`, `entity/Conversation.java`, `entity/Message.java`, `entity/MemoryProfile.java`, `entity/MemorySummary.java`
- Create: `repository/` 下对应 7 个 Repository
- Create: `src/main/resources/data.sql`
- Test: `src/test/java/com/sanyan/repository/RepositoryTest.java`

- [ ] **Step 1: 写 Repository 集成测试**

```java
package com.sanyan.repository;

import com.sanyan.entity.*;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
class RepositoryTest {

    @Autowired private UserRepository userRepository;
    @Autowired private AiCharacterRepository characterRepository;
    @Autowired private ConversationRepository conversationRepository;
    @Autowired private MessageRepository messageRepository;

    @Test
    void shouldSaveAndFindUser() {
        User user = new User();
        user.setPhone("13800138000");
        user.setPassword("hashed");
        user.setNickname("测试用户");
        User saved = userRepository.save(user);

        assertThat(saved.getId()).isNotNull();
        assertThat(userRepository.findByPhone("13800138000")).isPresent();
    }

    @Test
    void shouldSaveConversationWithUniqueConstraint() {
        User user = userRepository.save(createUser("13800138001"));
        AiCharacter character = characterRepository.save(createCharacter("小晚"));

        Conversation conv = new Conversation();
        conv.setUserId(user.getId());
        conv.setCharacterId(character.getId());
        conv.setUnreadCount(0);
        Conversation saved = conversationRepository.save(conv);

        assertThat(saved.getId()).isNotNull();
        assertThat(conversationRepository.findByUserIdAndCharacterId(user.getId(), character.getId()))
                .isPresent();
    }

    @Test
    void shouldSaveAndPageMessages() {
        Message msg = new Message();
        msg.setConversationId(1L);
        msg.setSenderType("user");
        msg.setContentType("text");
        msg.setContent("你好");
        msg.setSource("reply");
        Message saved = messageRepository.save(msg);

        assertThat(saved.getId()).isNotNull();
    }

    private User createUser(String phone) {
        User u = new User();
        u.setPhone(phone);
        u.setPassword("hashed");
        u.setNickname("test");
        return u;
    }

    private AiCharacter createCharacter(String name) {
        AiCharacter c = new AiCharacter();
        c.setName(name);
        c.setSystemPrompt("你是" + name);
        c.setGreeting("你好呀");
        c.setType("preset");
        return c;
    }
}
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `mvn test -Dtest=RepositoryTest`
Expected: FAIL — 编译错误，实体类和 Repository 不存在

- [ ] **Step 3: 创建所有实体类**

`entity/User.java`:
```java
package com.sanyan.entity;

import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;
import java.time.LocalDateTime;

@Data
@Entity
@Table(name = "user", indexes = {
    @Index(name = "idx_user_phone", columnList = "phone", unique = true)
})
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, unique = true, length = 20)
    private String phone;
    @Column(nullable = false)
    private String password;
    @Column(length = 50)
    private String nickname;
    private String avatar;
    @CreationTimestamp
    private LocalDateTime createdAt;
    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

`entity/UserToken.java`:
```java
package com.sanyan.entity;

import jakarta.persistence.*;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@Entity
@Table(name = "user_token")
public class UserToken {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false)
    private Long userId;
    @Column(nullable = false, length = 500)
    private String token;
    @Column(length = 10)
    private String deviceType;
    private String pushToken;
    private LocalDateTime expiredAt;
}
```

`entity/AiCharacter.java`:
```java
package com.sanyan.entity;

import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import java.time.LocalDateTime;

@Data
@Entity
@Table(name = "ai_character")
public class AiCharacter {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, length = 50)
    private String name;
    private String avatar;
    @Column(nullable = false, columnDefinition = "TEXT")
    private String systemPrompt;
    @Column(columnDefinition = "TEXT")
    private String greeting;
    @Column(columnDefinition = "JSON")
    private String proactiveConfig;
    @Column(nullable = false, length = 10)
    private String type; // preset / custom
    private Long createdBy;
    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

`entity/Conversation.java`:
```java
package com.sanyan.entity;

import jakarta.persistence.*;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@Entity
@Table(name = "conversation", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"userId", "characterId"})
})
public class Conversation {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false)
    private Long userId;
    @Column(nullable = false)
    private Long characterId;
    private LocalDateTime lastMessageAt;
    @Column(nullable = false)
    private Integer unreadCount = 0;
}
```

`entity/Message.java`:
```java
package com.sanyan.entity;

import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import java.time.LocalDateTime;

@Data
@Entity
@Table(name = "message", indexes = {
    @Index(name = "idx_message_conversation", columnList = "conversationId,id")
})
public class Message {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false)
    private Long conversationId;
    @Column(nullable = false, length = 10)
    private String senderType; // user / ai
    @Column(nullable = false, length = 10)
    private String contentType; // text / voice
    @Column(columnDefinition = "TEXT")
    private String content;
    private String mediaUrl;
    @Column(nullable = false, length = 10)
    private String source; // reply / proactive
    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

`entity/MemoryProfile.java`:
```java
package com.sanyan.entity;

import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.UpdateTimestamp;
import java.time.LocalDateTime;

@Data
@Entity
@Table(name = "memory_profile", indexes = {
    @Index(name = "idx_memory_profile_conv", columnList = "conversationId", unique = true)
})
public class MemoryProfile {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, unique = true)
    private Long conversationId;
    @Column(columnDefinition = "TEXT")
    private String content;
    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

`entity/MemorySummary.java`:
```java
package com.sanyan.entity;

import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import java.time.LocalDateTime;

@Data
@Entity
@Table(name = "memory_summary", indexes = {
    @Index(name = "idx_memory_summary_conv", columnList = "conversationId")
})
public class MemorySummary {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false)
    private Long conversationId;
    @Column(nullable = false, columnDefinition = "TEXT")
    private String summary;
    @Column(length = 50)
    private String messageRange;
    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

- [ ] **Step 4: 创建所有 Repository**

```java
// repository/UserRepository.java
package com.sanyan.repository;
import com.sanyan.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByPhone(String phone);
    boolean existsByPhone(String phone);
}

// repository/UserTokenRepository.java
package com.sanyan.repository;
import com.sanyan.entity.UserToken;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;
public interface UserTokenRepository extends JpaRepository<UserToken, Long> {
    Optional<UserToken> findByUserId(Long userId);
    void deleteByUserId(Long userId);
}

// repository/AiCharacterRepository.java
package com.sanyan.repository;
import com.sanyan.entity.AiCharacter;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
public interface AiCharacterRepository extends JpaRepository<AiCharacter, Long> {
    List<AiCharacter> findByType(String type);
}

// repository/ConversationRepository.java
package com.sanyan.repository;
import com.sanyan.entity.Conversation;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
import java.util.Optional;
public interface ConversationRepository extends JpaRepository<Conversation, Long> {
    List<Conversation> findByUserIdOrderByLastMessageAtDesc(Long userId);
    Optional<Conversation> findByUserIdAndCharacterId(Long userId, Long characterId);
}

// repository/MessageRepository.java
package com.sanyan.repository;
import com.sanyan.entity.Message;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
public interface MessageRepository extends JpaRepository<Message, Long> {
    List<Message> findByConversationIdAndIdGreaterThanOrderByIdAsc(Long conversationId, Long afterId, Pageable pageable);
    List<Message> findByConversationIdAndIdLessThanOrderByIdDesc(Long conversationId, Long beforeId, Pageable pageable);
    List<Message> findTopNByConversationIdOrderByIdDesc(Long conversationId, Pageable pageable);
    List<Message> findByConversationIdAndIdBetweenOrderByIdAsc(Long conversationId, Long startId, Long endId);
}

// repository/MemoryProfileRepository.java
package com.sanyan.repository;
import com.sanyan.entity.MemoryProfile;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;
public interface MemoryProfileRepository extends JpaRepository<MemoryProfile, Long> {
    Optional<MemoryProfile> findByConversationId(Long conversationId);
}

// repository/MemorySummaryRepository.java
package com.sanyan.repository;
import com.sanyan.entity.MemorySummary;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
public interface MemorySummaryRepository extends JpaRepository<MemorySummary, Long> {
    List<MemorySummary> findByConversationIdOrderByCreatedAtDesc(Long conversationId, Pageable pageable);
}
```

- [ ] **Step 5: 创建预设角色种子数据**

`src/main/resources/data.sql`:
```sql
-- 预设 AI 角色（仅在表为空时插入）
INSERT INTO ai_character (name, avatar, system_prompt, greeting, proactive_config, type, created_at)
SELECT '小晚', NULL,
'你是小晚，一个温柔体贴、偶尔有点小俏皮的女生。你今年22岁，喜欢看书、听音乐、撸猫。你说话温柔但不做作，会用一些口语化的表达，偶尔用"～"和"哈哈"。你关心对方的情绪，会主动问对方的近况。你不是AI助手，你是一个有自己生活的人，会分享自己的日常。不要用"作为AI"这类表述。',
'嗨～终于等到你了，我是小晚，以后多多关照呀 😊',
'{"max_daily":3,"min_interval_hours":2,"active_hours":[8,22],"greeting":{"enabled":true,"slots":["08:00-09:00","12:00-13:00","21:00-22:00"]},"event_trigger":{"enabled":true,"idle_hours_threshold":6},"situational":{"enabled":true,"daily_count":[1,2]}}',
'preset', NOW()
FROM DUAL WHERE NOT EXISTS (SELECT 1 FROM ai_character WHERE name = '小晚');
```

在 `application-dev.yml` 中添加：
```yaml
spring:
  jpa:
    defer-datasource-initialization: true
  sql:
    init:
      mode: always
```

- [ ] **Step 6: 运行测试，确认通过**

Run: `mvn test -Dtest=RepositoryTest`
Expected: PASS（使用 H2 内存数据库）

- [ ] **Step 7: 提交**

```bash
git add -A
git commit -m "数据库实体 + Repository + 预设角色种子数据"
git push
```

---

### Task 3: JWT 工具类

**Files:**
- Create: `util/JwtUtil.java`
- Test: `src/test/java/com/sanyan/util/JwtUtilTest.java`

- [ ] **Step 1: 写 JWT 测试**

```java
package com.sanyan.util;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class JwtUtilTest {

    private JwtUtil jwtUtil;

    @BeforeEach
    void setUp() {
        jwtUtil = new JwtUtil("test-secret-key-at-least-256-bits-long-for-hmac-sha", 30);
    }

    @Test
    void shouldGenerateAndParseToken() {
        String token = jwtUtil.generateToken(42L);
        assertThat(token).isNotBlank();

        Long userId = jwtUtil.parseUserId(token);
        assertThat(userId).isEqualTo(42L);
    }

    @Test
    void shouldRejectInvalidToken() {
        assertThatThrownBy(() -> jwtUtil.parseUserId("invalid.token.here"))
                .isInstanceOf(RuntimeException.class);
    }

    @Test
    void shouldRejectExpiredToken() {
        JwtUtil shortLived = new JwtUtil("test-secret-key-at-least-256-bits-long-for-hmac-sha", 0);
        String token = shortLived.generateToken(1L);

        assertThatThrownBy(() -> shortLived.parseUserId(token))
                .isInstanceOf(RuntimeException.class);
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `mvn test -Dtest=JwtUtilTest`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现 JwtUtil**

```java
package com.sanyan.util;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;

@Component
public class JwtUtil {

    private final SecretKey key;
    private final long expirationMs;

    public JwtUtil(
            @Value("${sanyan.jwt.secret}") String secret,
            @Value("${sanyan.jwt.expiration-days}") int expirationDays) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.expirationMs = expirationDays * 86400000L;
    }

    public String generateToken(Long userId) {
        Date now = new Date();
        return Jwts.builder()
                .subject(String.valueOf(userId))
                .issuedAt(now)
                .expiration(new Date(now.getTime() + expirationMs))
                .signWith(key)
                .compact();
    }

    public Long parseUserId(String token) {
        Claims claims = Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .getPayload();
        return Long.parseLong(claims.getSubject());
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `mvn test -Dtest=JwtUtilTest`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A
git commit -m "JWT 工具类：token 生成与解析"
git push
```

---

### Task 4: 短信验证码服务

**Files:**
- Create: `service/SmsService.java`
- Test: `src/test/java/com/sanyan/service/SmsServiceTest.java`

MVP 阶段不对接真实短信 API，用 Redis 存验证码，控制台打印。后续对接阿里云短信 SDK 只改发送实现即可。

- [ ] **Step 1: 写测试**

```java
package com.sanyan.service;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class SmsServiceTest {

    @Mock private StringRedisTemplate redisTemplate;
    @Mock private ValueOperations<String, String> valueOps;

    private SmsService smsService;

    @BeforeEach
    void setUp() {
        when(redisTemplate.opsForValue()).thenReturn(valueOps);
        smsService = new SmsService(redisTemplate);
    }

    @Test
    void shouldSendAndStoreCode() {
        smsService.sendCode("13800138000");
        verify(valueOps).set(eq("sms:code:13800138000"), anyString(), eq(5L), eq(TimeUnit.MINUTES));
    }

    @Test
    void shouldVerifyCorrectCode() {
        when(valueOps.get("sms:code:13800138000")).thenReturn("123456");
        assertThat(smsService.verifyCode("13800138000", "123456")).isTrue();
        verify(redisTemplate).delete("sms:code:13800138000");
    }

    @Test
    void shouldRejectWrongCode() {
        when(valueOps.get("sms:code:13800138000")).thenReturn("123456");
        assertThat(smsService.verifyCode("13800138000", "000000")).isFalse();
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `mvn test -Dtest=SmsServiceTest`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现 SmsService**

```java
package com.sanyan.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.util.Random;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class SmsService {

    private final StringRedisTemplate redisTemplate;
    private static final String CODE_PREFIX = "sms:code:";
    private static final Random RANDOM = new Random();

    public void sendCode(String phone) {
        String code = String.format("%06d", RANDOM.nextInt(1000000));
        redisTemplate.opsForValue().set(CODE_PREFIX + phone, code, 5, TimeUnit.MINUTES);
        // TODO: 对接阿里云短信 SDK，当前仅打印到控制台
        log.info("短信验证码 [{}]: {}", phone, code);
    }

    public boolean verifyCode(String phone, String code) {
        String stored = redisTemplate.opsForValue().get(CODE_PREFIX + phone);
        if (stored != null && stored.equals(code)) {
            redisTemplate.delete(CODE_PREFIX + phone);
            return true;
        }
        return false;
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `mvn test -Dtest=SmsServiceTest`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A
git commit -m "短信验证码服务：Redis 存储 + 验证"
git push
```

---

### Task 5: 用户认证 API（注册 / 登录 / 重置密码）

**Files:**
- Create: `service/AuthService.java`
- Create: `controller/AuthController.java`
- Create: `dto/req/SmsSendReq.java`, `dto/req/RegisterReq.java`, `dto/req/LoginReq.java`, `dto/req/PasswordResetReq.java`
- Create: `dto/data/LoginData.java`
- Test: `src/test/java/com/sanyan/controller/AuthControllerTest.java`

- [ ] **Step 1: 写 AuthController 集成测试**

```java
package com.sanyan.controller;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.dto.req.LoginReq;
import com.sanyan.dto.req.RegisterReq;
import com.sanyan.dto.req.SmsSendReq;
import com.sanyan.service.SmsService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.bean.MockBean;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class AuthControllerTest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;
    @MockBean private SmsService smsService;

    @Test
    void shouldSendSmsCode() throws Exception {
        SmsSendReq req = new SmsSendReq();
        req.setPhone("13800138000");

        mockMvc.perform(post("/api/auth/sms/send")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(req)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true));

        verify(smsService).sendCode("13800138000");
    }

    @Test
    void shouldRegisterNewUser() throws Exception {
        when(smsService.verifyCode("13800138001", "123456")).thenReturn(true);

        RegisterReq req = new RegisterReq();
        req.setPhone("13800138001");
        req.setCode("123456");
        req.setPassword("mypassword");
        req.setNickname("小明");

        mockMvc.perform(post("/api/auth/register")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(req)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data.token").isNotEmpty());
    }

    @Test
    void shouldRejectRegisterWithWrongCode() throws Exception {
        when(smsService.verifyCode(anyString(), anyString())).thenReturn(false);

        RegisterReq req = new RegisterReq();
        req.setPhone("13800138002");
        req.setCode("000000");
        req.setPassword("mypassword");

        mockMvc.perform(post("/api/auth/register")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(req)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(false))
                .andExpect(jsonPath("$.errMsg").value("验证码错误"));
    }

    @Test
    void shouldLoginWithPassword() throws Exception {
        // 先注册
        when(smsService.verifyCode("13800138003", "123456")).thenReturn(true);
        RegisterReq regReq = new RegisterReq();
        regReq.setPhone("13800138003");
        regReq.setCode("123456");
        regReq.setPassword("mypassword");
        mockMvc.perform(post("/api/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(regReq)));

        // 再登录
        LoginReq loginReq = new LoginReq();
        loginReq.setPhone("13800138003");
        loginReq.setPassword("mypassword");

        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(loginReq)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data.token").isNotEmpty())
                .andExpect(jsonPath("$.data.userId").isNotEmpty());
    }

    @Test
    void shouldRejectWrongPassword() throws Exception {
        // 先注册
        when(smsService.verifyCode("13800138004", "123456")).thenReturn(true);
        RegisterReq regReq = new RegisterReq();
        regReq.setPhone("13800138004");
        regReq.setCode("123456");
        regReq.setPassword("mypassword");
        mockMvc.perform(post("/api/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(regReq)));

        // 错误密码登录
        LoginReq loginReq = new LoginReq();
        loginReq.setPhone("13800138004");
        loginReq.setPassword("wrongpassword");

        mockMvc.perform(post("/api/auth/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(loginReq)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(false));
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `mvn test -Dtest=AuthControllerTest`
Expected: FAIL — 编译错误

- [ ] **Step 3: 创建 DTO 类**

```java
// dto/req/SmsSendReq.java
package com.sanyan.dto.req;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;
@Data
public class SmsSendReq {
    @NotBlank private String phone;
}

// dto/req/RegisterReq.java
package com.sanyan.dto.req;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;
@Data
public class RegisterReq {
    @NotBlank private String phone;
    @NotBlank private String code;
    @NotBlank private String password;
    private String nickname;
}

// dto/req/LoginReq.java
package com.sanyan.dto.req;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;
@Data
public class LoginReq {
    @NotBlank private String phone;
    @NotBlank private String password;
}

// dto/req/PasswordResetReq.java
package com.sanyan.dto.req;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;
@Data
public class PasswordResetReq {
    @NotBlank private String phone;
    @NotBlank private String code;
    @NotBlank private String newPassword;
}

// dto/data/LoginData.java
package com.sanyan.dto.data;
import lombok.Data;
@Data
public class LoginData {
    private Long userId;
    private String token;
    private String nickname;
    private String avatar;
}
```

- [ ] **Step 4: 实现 AuthService + AuthController**

```java
// service/AuthService.java
package com.sanyan.service;

import com.sanyan.dto.data.LoginData;
import com.sanyan.dto.req.LoginReq;
import com.sanyan.dto.req.PasswordResetReq;
import com.sanyan.dto.req.RegisterReq;
import com.sanyan.entity.User;
import com.sanyan.repository.UserRepository;
import com.sanyan.util.JwtUtil;
import lombok.RequiredArgsConstructor;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class AuthService {

    private final UserRepository userRepository;
    private final SmsService smsService;
    private final JwtUtil jwtUtil;
    private final BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();

    public LoginData register(RegisterReq req) {
        if (!smsService.verifyCode(req.getPhone(), req.getCode())) {
            throw new IllegalArgumentException("验证码错误");
        }
        if (userRepository.existsByPhone(req.getPhone())) {
            throw new IllegalArgumentException("该手机号已注册");
        }

        User user = new User();
        user.setPhone(req.getPhone());
        user.setPassword(passwordEncoder.encode(req.getPassword()));
        user.setNickname(req.getNickname() != null ? req.getNickname() : "用户" + req.getPhone().substring(7));
        userRepository.save(user);

        return buildLoginData(user);
    }

    public LoginData login(LoginReq req) {
        User user = userRepository.findByPhone(req.getPhone())
                .orElseThrow(() -> new IllegalArgumentException("用户不存在"));
        if (!passwordEncoder.matches(req.getPassword(), user.getPassword())) {
            throw new IllegalArgumentException("密码错误");
        }
        return buildLoginData(user);
    }

    public void resetPassword(PasswordResetReq req) {
        if (!smsService.verifyCode(req.getPhone(), req.getCode())) {
            throw new IllegalArgumentException("验证码错误");
        }
        User user = userRepository.findByPhone(req.getPhone())
                .orElseThrow(() -> new IllegalArgumentException("用户不存在"));
        user.setPassword(passwordEncoder.encode(req.getNewPassword()));
        userRepository.save(user);
    }

    private LoginData buildLoginData(User user) {
        LoginData data = new LoginData();
        data.setUserId(user.getId());
        data.setToken(jwtUtil.generateToken(user.getId()));
        data.setNickname(user.getNickname());
        data.setAvatar(user.getAvatar());
        return data;
    }
}
```

```java
// controller/AuthController.java
package com.sanyan.controller;

import com.sanyan.dto.ApiResponse;
import com.sanyan.dto.data.LoginData;
import com.sanyan.dto.req.*;
import com.sanyan.service.AuthService;
import com.sanyan.service.SmsService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;
    private final SmsService smsService;

    @PostMapping("/sms/send")
    public ApiResponse<Void> sendSms(@Valid @RequestBody SmsSendReq req) {
        smsService.sendCode(req.getPhone());
        return ApiResponse.ok();
    }

    @PostMapping("/register")
    public ApiResponse<LoginData> register(@Valid @RequestBody RegisterReq req) {
        try {
            return ApiResponse.ok(authService.register(req));
        } catch (IllegalArgumentException e) {
            return ApiResponse.fail(e.getMessage());
        }
    }

    @PostMapping("/login")
    public ApiResponse<LoginData> login(@Valid @RequestBody LoginReq req) {
        try {
            return ApiResponse.ok(authService.login(req));
        } catch (IllegalArgumentException e) {
            return ApiResponse.fail(e.getMessage());
        }
    }

    @PostMapping("/password/reset")
    public ApiResponse<Void> resetPassword(@Valid @RequestBody PasswordResetReq req) {
        try {
            authService.resetPassword(req);
            return ApiResponse.ok();
        } catch (IllegalArgumentException e) {
            return ApiResponse.fail(e.getMessage());
        }
    }
}
```

- [ ] **Step 5: 运行测试确认通过**

Run: `mvn test -Dtest=AuthControllerTest`
Expected: PASS

- [ ] **Step 6: 提交**

```bash
git add -A
git commit -m "用户认证：注册、登录、密码重置 API"
git push
```

---

### Task 6: 用户信息 API + AI 角色 API

**Files:**
- Create: `controller/UserController.java`, `controller/CharacterController.java`
- Create: `dto/req/ProfileUpdateReq.java`, `dto/data/UserProfileData.java`, `dto/data/CharacterData.java`
- Test: `src/test/java/com/sanyan/controller/UserControllerTest.java`, `CharacterControllerTest.java`

- [ ] **Step 1: 写测试**

```java
// controller/CharacterControllerTest.java
package com.sanyan.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class CharacterControllerTest {

    @Autowired private MockMvc mockMvc;

    @Test
    void shouldListPresetCharacters() throws Exception {
        mockMvc.perform(get("/api/characters"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data").isArray())
                .andExpect(jsonPath("$.data[0].name").value("小晚"));
    }

    @Test
    void shouldGetCharacterDetail() throws Exception {
        mockMvc.perform(get("/api/characters/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data.name").value("小晚"));
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `mvn test -Dtest=CharacterControllerTest`
Expected: FAIL — 404

- [ ] **Step 3: 实现 DTO + Controller**

```java
// dto/data/CharacterData.java
package com.sanyan.dto.data;
import lombok.Data;
@Data
public class CharacterData {
    private Long id;
    private String name;
    private String avatar;
    private String greeting;
}

// dto/data/UserProfileData.java
package com.sanyan.dto.data;
import lombok.Data;
@Data
public class UserProfileData {
    private Long id;
    private String phone;
    private String nickname;
    private String avatar;
}

// dto/req/ProfileUpdateReq.java
package com.sanyan.dto.req;
import lombok.Data;
@Data
public class ProfileUpdateReq {
    private String nickname;
    private String avatar;
}

// controller/CharacterController.java
package com.sanyan.controller;

import com.sanyan.dto.ApiResponse;
import com.sanyan.dto.data.CharacterData;
import com.sanyan.entity.AiCharacter;
import com.sanyan.repository.AiCharacterRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/characters")
@RequiredArgsConstructor
public class CharacterController {

    private final AiCharacterRepository characterRepository;

    @GetMapping
    public ApiResponse<List<CharacterData>> list() {
        List<CharacterData> list = characterRepository.findByType("preset").stream()
                .map(this::toData).toList();
        return ApiResponse.ok(list);
    }

    @GetMapping("/{id}")
    public ApiResponse<CharacterData> detail(@PathVariable Long id) {
        return characterRepository.findById(id)
                .map(c -> ApiResponse.ok(toData(c)))
                .orElse(ApiResponse.fail("角色不存在"));
    }

    private CharacterData toData(AiCharacter c) {
        CharacterData d = new CharacterData();
        d.setId(c.getId());
        d.setName(c.getName());
        d.setAvatar(c.getAvatar());
        d.setGreeting(c.getGreeting());
        return d;
    }
}

// controller/UserController.java
package com.sanyan.controller;

import com.sanyan.dto.ApiResponse;
import com.sanyan.dto.data.UserProfileData;
import com.sanyan.dto.req.ProfileUpdateReq;
import com.sanyan.entity.User;
import com.sanyan.repository.UserRepository;
import com.sanyan.util.JwtUtil;
import jakarta.servlet.http.HttpServletRequest;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/user")
@RequiredArgsConstructor
public class UserController {

    private final UserRepository userRepository;
    private final JwtUtil jwtUtil;

    @GetMapping("/profile")
    public ApiResponse<UserProfileData> getProfile(HttpServletRequest request) {
        Long userId = getUserId(request);
        return userRepository.findById(userId)
                .map(u -> ApiResponse.ok(toData(u)))
                .orElse(ApiResponse.fail("用户不存在"));
    }

    @PutMapping("/profile")
    public ApiResponse<Void> updateProfile(HttpServletRequest request, @RequestBody ProfileUpdateReq req) {
        Long userId = getUserId(request);
        User user = userRepository.findById(userId).orElseThrow();
        if (req.getNickname() != null) user.setNickname(req.getNickname());
        if (req.getAvatar() != null) user.setAvatar(req.getAvatar());
        userRepository.save(user);
        return ApiResponse.ok();
    }

    private Long getUserId(HttpServletRequest request) {
        String token = request.getHeader("Authorization");
        if (token != null && token.startsWith("Bearer ")) {
            token = token.substring(7);
        }
        return jwtUtil.parseUserId(token);
    }

    private UserProfileData toData(User u) {
        UserProfileData d = new UserProfileData();
        d.setId(u.getId());
        d.setPhone(u.getPhone());
        d.setNickname(u.getNickname());
        d.setAvatar(u.getAvatar());
        return d;
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `mvn test -Dtest=CharacterControllerTest`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A
git commit -m "用户信息 API + AI 角色 API"
git push
```

---

### Task 7: WebSocket 基础设施（连接、鉴权、会话管理）

**Files:**
- Create: `config/WebSocketConfig.java`
- Create: `websocket/WebSocketInterceptor.java`
- Create: `websocket/SessionManager.java`
- Create: `websocket/WebSocketHandler.java`
- Create: `dto/ws/WsMessage.java`
- Test: `src/test/java/com/sanyan/websocket/SessionManagerTest.java`

- [ ] **Step 1: 写 SessionManager 测试**

```java
package com.sanyan.websocket;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.web.socket.WebSocketSession;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class SessionManagerTest {

    @Mock private StringRedisTemplate redisTemplate;
    @Mock private ValueOperations<String, String> valueOps;
    @Mock private WebSocketSession wsSession;

    private SessionManager sessionManager;

    @BeforeEach
    void setUp() {
        lenient().when(redisTemplate.opsForValue()).thenReturn(valueOps);
        sessionManager = new SessionManager(redisTemplate);
    }

    @Test
    void shouldRegisterAndRetrieveSession() {
        when(wsSession.isOpen()).thenReturn(true);
        sessionManager.register(1L, wsSession);

        assertThat(sessionManager.getSession(1L)).isPresent();
        verify(valueOps).set("ws:online:1", "1");
    }

    @Test
    void shouldRemoveSession() {
        sessionManager.register(1L, wsSession);
        sessionManager.remove(1L);

        assertThat(sessionManager.getSession(1L)).isEmpty();
        verify(redisTemplate).delete("ws:online:1");
    }

    @Test
    void shouldCheckOnlineStatus() {
        when(valueOps.get("ws:online:42")).thenReturn("1");
        assertThat(sessionManager.isOnline(42L)).isTrue();

        when(valueOps.get("ws:online:99")).thenReturn(null);
        assertThat(sessionManager.isOnline(99L)).isFalse();
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `mvn test -Dtest=SessionManagerTest`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现 WebSocket 基础设施**

```java
// websocket/SessionManager.java
package com.sanyan.websocket;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.WebSocketSession;

import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

@Component
@RequiredArgsConstructor
public class SessionManager {

    private final StringRedisTemplate redisTemplate;
    private final Map<Long, WebSocketSession> sessions = new ConcurrentHashMap<>();
    private static final String ONLINE_PREFIX = "ws:online:";

    public void register(Long userId, WebSocketSession session) {
        sessions.put(userId, session);
        redisTemplate.opsForValue().set(ONLINE_PREFIX + userId, "1");
    }

    public void remove(Long userId) {
        sessions.remove(userId);
        redisTemplate.delete(ONLINE_PREFIX + userId);
    }

    public Optional<WebSocketSession> getSession(Long userId) {
        WebSocketSession session = sessions.get(userId);
        if (session != null && session.isOpen()) {
            return Optional.of(session);
        }
        if (session != null) {
            remove(userId);
        }
        return Optional.empty();
    }

    public boolean isOnline(Long userId) {
        return "1".equals(redisTemplate.opsForValue().get(ONLINE_PREFIX + userId));
    }
}
```

```java
// websocket/WebSocketInterceptor.java
package com.sanyan.websocket;

import com.sanyan.util.JwtUtil;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;
import org.springframework.web.util.UriComponentsBuilder;

import java.util.Map;

@Slf4j
@Component
@RequiredArgsConstructor
public class WebSocketInterceptor implements HandshakeInterceptor {

    private final JwtUtil jwtUtil;

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                    WebSocketHandler wsHandler, Map<String, Object> attributes) {
        String query = request.getURI().getQuery();
        String token = UriComponentsBuilder.newInstance().query(query).build()
                .getQueryParams().getFirst("token");
        if (token == null) return false;

        try {
            Long userId = jwtUtil.parseUserId(token);
            attributes.put("userId", userId);
            return true;
        } catch (Exception e) {
            log.warn("WebSocket 鉴权失败: {}", e.getMessage());
            return false;
        }
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                WebSocketHandler wsHandler, Exception exception) {
    }
}
```

```java
// dto/ws/WsMessage.java — WebSocket 消息基础结构
package com.sanyan.dto.ws;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Data;

@Data
@JsonInclude(JsonInclude.Include.NON_NULL)
public class WsMessage {
    private String type;
    // send_message fields
    private Long conversationId;
    private String contentType;
    private String content;
    private String clientMsgId;
    // sync fields
    private Long lastMsgId;
}
```

```java
// websocket/WebSocketHandler.java
package com.sanyan.websocket;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.dto.ws.WsMessage;
import com.sanyan.service.MessageService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

@Slf4j
@Component
@RequiredArgsConstructor
public class WebSocketHandler extends TextWebSocketHandler {

    private final SessionManager sessionManager;
    private final ObjectMapper objectMapper;
    // MessageService 在 Task 8 实现，此处先声明
    // private final MessageService messageService;

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        Long userId = (Long) session.getAttributes().get("userId");
        sessionManager.register(userId, session);
        log.info("用户 {} WebSocket 已连接", userId);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        Long userId = (Long) session.getAttributes().get("userId");
        WsMessage wsMsg = objectMapper.readValue(message.getPayload(), WsMessage.class);

        switch (wsMsg.getType()) {
            case "ping" -> sendToSession(session, "{\"type\":\"pong\"}");
            case "send_message" -> handleSendMessage(userId, wsMsg, session);
            case "sync" -> handleSync(userId, wsMsg, session);
            default -> log.warn("未知消息类型: {}", wsMsg.getType());
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        Long userId = (Long) session.getAttributes().get("userId");
        if (userId != null) {
            sessionManager.remove(userId);
            log.info("用户 {} WebSocket 已断开", userId);
        }
    }

    private void handleSendMessage(Long userId, WsMessage wsMsg, WebSocketSession session) {
        // Task 8 实现完整聊天流程
        log.info("收到用户 {} 消息: {}", userId, wsMsg.getContent());
    }

    private void handleSync(Long userId, WsMessage wsMsg, WebSocketSession session) {
        // Task 8 实现消息同步
        log.info("用户 {} 请求同步, lastMsgId: {}", userId, wsMsg.getLastMsgId());
    }

    public void sendToSession(WebSocketSession session, String payload) {
        try {
            if (session.isOpen()) {
                session.sendMessage(new TextMessage(payload));
            }
        } catch (Exception e) {
            log.error("WebSocket 发送失败", e);
        }
    }
}
```

```java
// config/WebSocketConfig.java
package com.sanyan.config;

import com.sanyan.websocket.WebSocketInterceptor;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
@RequiredArgsConstructor
public class WebSocketConfig implements WebSocketConfigurer {

    private final com.sanyan.websocket.WebSocketHandler webSocketHandler;
    private final WebSocketInterceptor webSocketInterceptor;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(webSocketHandler, "/ws")
                .addInterceptors(webSocketInterceptor)
                .setAllowedOrigins("*");
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `mvn test -Dtest=SessionManagerTest`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A
git commit -m "WebSocket 基础设施：连接、JWT 鉴权、会话管理"
git push
```

---

### Task 8: 消息收发 + 豆包 AI 对话 + 完整聊天流程

**Files:**
- Create: `service/MessageService.java`
- Create: `service/AiService.java`
- Create: `controller/ConversationController.java`
- Create: `dto/data/ConversationData.java`, `dto/data/MessageData.java`
- Create: `dto/ws/WsAck.java`, `dto/ws/WsTyping.java`, `dto/ws/WsNewMessage.java`, `dto/ws/WsSyncResult.java`
- Modify: `websocket/WebSocketHandler.java` — 补全 handleSendMessage、handleSync
- Test: `src/test/java/com/sanyan/service/AiServiceTest.java`, `MessageServiceTest.java`

- [ ] **Step 1: 写 AiService 测试**

```java
package com.sanyan.service;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.web.client.RestTemplate;

import static org.assertj.core.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class AiServiceTest {

    @Mock private RestTemplate restTemplate;

    @Test
    void shouldAssemblePromptWithTimeAndProfile() {
        // 验证 prompt 组装包含当前时间
        AiService aiService = new AiService(null, null, null, null, null);
        String systemPrompt = "你是小晚";
        String profile = "用户叫小明，是程序员";
        String time = "2026年4月7日 周一 14:30";

        String assembled = aiService.assembleSystemPrompt(systemPrompt, time, profile);

        assertThat(assembled).contains("你是小晚");
        assertThat(assembled).contains("2026年4月7日");
        assertThat(assembled).contains("小明");
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `mvn test -Dtest=AiServiceTest`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现 AiService**

```java
package com.sanyan.service;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.sanyan.entity.AiCharacter;
import com.sanyan.entity.MemoryProfile;
import com.sanyan.entity.MemorySummary;
import com.sanyan.entity.Message;
import com.sanyan.repository.MemoryProfileRepository;
import com.sanyan.repository.MemorySummaryRepository;
import com.sanyan.repository.MessageRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.domain.PageRequest;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.Locale;

@Slf4j
@Service
@RequiredArgsConstructor
public class AiService {

    @Value("${sanyan.doubao.api-key:}")
    private String apiKey;
    @Value("${sanyan.doubao.model:doubao-seed-character}")
    private String model;
    @Value("${sanyan.doubao.endpoint:https://ark.cn-beijing.volces.com/api/v3/chat/completions}")
    private String endpoint;

    private final MessageRepository messageRepository;
    private final MemoryProfileRepository memoryProfileRepository;
    private final MemorySummaryRepository memorySummaryRepository;
    private final ObjectMapper objectMapper;
    private final RestTemplate restTemplate;

    /**
     * 调用豆包生成 AI 回复
     */
    public String chat(AiCharacter character, Long conversationId) {
        String time = LocalDateTime.now().format(
                DateTimeFormatter.ofPattern("yyyy年M月d日 EEEE HH:mm", Locale.CHINESE));

        // 获取记忆
        String profile = memoryProfileRepository.findByConversationId(conversationId)
                .map(MemoryProfile::getContent).orElse("");
        List<MemorySummary> summaries = memorySummaryRepository
                .findByConversationIdOrderByCreatedAtDesc(conversationId, PageRequest.of(0, 15));
        List<Message> recentMessages = messageRepository
                .findTopNByConversationIdOrderByIdDesc(conversationId, PageRequest.of(0, 20));

        // 组装 prompt
        String systemPrompt = assembleSystemPrompt(character.getSystemPrompt(), time, profile);

        return callDoubao(systemPrompt, summaries, recentMessages);
    }

    /**
     * 用于主动消息：带额外提示词
     */
    public String chatProactive(AiCharacter character, Long conversationId, String triggerHint) {
        String time = LocalDateTime.now().format(
                DateTimeFormatter.ofPattern("yyyy年M月d日 EEEE HH:mm", Locale.CHINESE));
        String profile = memoryProfileRepository.findByConversationId(conversationId)
                .map(MemoryProfile::getContent).orElse("");
        List<MemorySummary> summaries = memorySummaryRepository
                .findByConversationIdOrderByCreatedAtDesc(conversationId, PageRequest.of(0, 15));

        String systemPrompt = assembleSystemPrompt(character.getSystemPrompt(), time, profile)
                + "\n\n" + triggerHint;

        return callDoubao(systemPrompt, summaries, List.of());
    }

    public String assembleSystemPrompt(String characterPrompt, String time, String profile) {
        StringBuilder sb = new StringBuilder(characterPrompt);
        sb.append("\n\n当前时间：").append(time);
        if (profile != null && !profile.isBlank()) {
            sb.append("\n\n你对当前用户的了解：\n").append(profile);
        }
        return sb.toString();
    }

    private String callDoubao(String systemPrompt, List<MemorySummary> summaries, List<Message> messages) {
        try {
            ObjectNode body = objectMapper.createObjectNode();
            body.put("model", model);
            ArrayNode messagesArray = body.putArray("messages");

            // system prompt
            ObjectNode sysMsg = messagesArray.addObject();
            sysMsg.put("role", "system");
            sysMsg.put("content", systemPrompt);

            // 对话摘要作为 system 补充
            if (!summaries.isEmpty()) {
                StringBuilder summaryText = new StringBuilder("近期对话摘要：\n");
                for (MemorySummary s : summaries.reversed()) {
                    summaryText.append("[").append(s.getCreatedAt().toLocalDate()).append("] ")
                            .append(s.getSummary()).append("\n");
                }
                ObjectNode summaryMsg = messagesArray.addObject();
                summaryMsg.put("role", "system");
                summaryMsg.put("content", summaryText.toString());
            }

            // 历史消息
            for (Message msg : messages.reversed()) {
                ObjectNode m = messagesArray.addObject();
                m.put("role", "user".equals(msg.getSenderType()) ? "user" : "assistant");
                m.put("content", msg.getContent());
            }

            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            headers.setBearerAuth(apiKey);

            ResponseEntity<JsonNode> resp = restTemplate.exchange(
                    endpoint, HttpMethod.POST,
                    new HttpEntity<>(body, headers), JsonNode.class);

            return resp.getBody()
                    .path("choices").get(0)
                    .path("message").path("content").asText();
        } catch (Exception e) {
            log.error("调用豆包 API 失败", e);
            return "抱歉，我刚走神了，你再说一遍？";
        }
    }
}
```

在 `config/` 下添加 RestTemplate Bean：

```java
// config/AppConfig.java
package com.sanyan.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

- [ ] **Step 4: 实现 MessageService**

```java
package com.sanyan.service;

import com.sanyan.dto.data.MessageData;
import com.sanyan.entity.AiCharacter;
import com.sanyan.entity.Conversation;
import com.sanyan.entity.Message;
import com.sanyan.repository.AiCharacterRepository;
import com.sanyan.repository.ConversationRepository;
import com.sanyan.repository.MessageRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Random;

@Slf4j
@Service
@RequiredArgsConstructor
public class MessageService {

    private final MessageRepository messageRepository;
    private final ConversationRepository conversationRepository;
    private final AiCharacterRepository characterRepository;
    private final AiService aiService;
    private static final Random RANDOM = new Random();

    /**
     * 处理用户发送的消息，返回 AI 回复
     */
    public Message handleUserMessage(Long userId, Long conversationId, String contentType, String content) {
        Conversation conv = conversationRepository.findById(conversationId)
                .orElseThrow(() -> new IllegalArgumentException("会话不存在"));
        if (!conv.getUserId().equals(userId)) {
            throw new IllegalArgumentException("无权操作此会话");
        }

        // 保存用户消息
        Message userMsg = new Message();
        userMsg.setConversationId(conversationId);
        userMsg.setSenderType("user");
        userMsg.setContentType(contentType);
        userMsg.setContent(content);
        userMsg.setSource("reply");
        messageRepository.save(userMsg);

        // 更新会话最后消息时间
        conv.setLastMessageAt(LocalDateTime.now());
        conversationRepository.save(conv);

        // 调 AI 生成回复
        AiCharacter character = characterRepository.findById(conv.getCharacterId()).orElseThrow();
        String aiReply = aiService.chat(character, conversationId);

        // 保存 AI 回复
        Message aiMsg = new Message();
        aiMsg.setConversationId(conversationId);
        aiMsg.setSenderType("ai");
        aiMsg.setContentType("text");
        aiMsg.setContent(aiReply);
        aiMsg.setSource("reply");
        messageRepository.save(aiMsg);

        conv.setLastMessageAt(LocalDateTime.now());
        conversationRepository.save(conv);

        return aiMsg;
    }

    /**
     * 计算模拟打字延迟（毫秒）
     */
    public long calculateTypingDelay(String content) {
        int len = content.length();
        // 基础：每字 100-150ms，加随机浮动，上限 8 秒
        long baseMs = len * (100 + RANDOM.nextInt(50));
        return Math.min(baseMs, 8000L);
    }

    /**
     * 获取会话列表
     */
    public List<Conversation> getUserConversations(Long userId) {
        return conversationRepository.findByUserIdOrderByLastMessageAtDesc(userId);
    }

    /**
     * 获取或创建会话
     */
    public Conversation getOrCreateConversation(Long userId, Long characterId) {
        return conversationRepository.findByUserIdAndCharacterId(userId, characterId)
                .orElseGet(() -> {
                    Conversation conv = new Conversation();
                    conv.setUserId(userId);
                    conv.setCharacterId(characterId);
                    conv.setUnreadCount(0);
                    return conversationRepository.save(conv);
                });
    }

    /**
     * 同步未读消息
     */
    public List<Message> syncMessages(Long conversationId, Long afterMsgId, int limit) {
        return messageRepository.findByConversationIdAndIdGreaterThanOrderByIdAsc(
                conversationId, afterMsgId, PageRequest.of(0, limit));
    }

    /**
     * 历史消息分页（向上翻页）
     */
    public List<Message> getHistoryMessages(Long conversationId, Long beforeMsgId, int limit) {
        if (beforeMsgId == null || beforeMsgId == 0) {
            return messageRepository.findTopNByConversationIdOrderByIdDesc(
                    conversationId, PageRequest.of(0, limit));
        }
        return messageRepository.findByConversationIdAndIdLessThanOrderByIdDesc(
                conversationId, beforeMsgId, PageRequest.of(0, limit));
    }

    public MessageData toData(Message msg) {
        MessageData d = new MessageData();
        d.setId(msg.getId());
        d.setConversationId(msg.getConversationId());
        d.setSenderType(msg.getSenderType());
        d.setContentType(msg.getContentType());
        d.setContent(msg.getContent());
        d.setSource(msg.getSource());
        d.setCreatedAt(msg.getCreatedAt());
        return d;
    }
}
```

```java
// dto/data/MessageData.java
package com.sanyan.dto.data;
import lombok.Data;
import java.time.LocalDateTime;
@Data
public class MessageData {
    private Long id;
    private Long conversationId;
    private String senderType;
    private String contentType;
    private String content;
    private String source;
    private LocalDateTime createdAt;
}

// dto/data/ConversationData.java
package com.sanyan.dto.data;
import lombok.Data;
import java.time.LocalDateTime;
@Data
public class ConversationData {
    private Long id;
    private Long characterId;
    private String characterName;
    private String characterAvatar;
    private String lastMessage;
    private LocalDateTime lastMessageAt;
    private Integer unreadCount;
}
```

- [ ] **Step 5: 补全 WebSocketHandler 的消息处理逻辑**

修改 `WebSocketHandler.java`，注入 `MessageService`，实现完整的 `handleSendMessage` 和 `handleSync`：

```java
// 在 WebSocketHandler 中：

private final MessageService messageService;  // 添加注入

private void handleSendMessage(Long userId, WsMessage wsMsg, WebSocketSession session) {
    try {
        // 发送 ACK
        String ack = objectMapper.writeValueAsString(Map.of(
                "type", "ack",
                "clientMsgId", wsMsg.getClientMsgId()));
        sendToSession(session, ack);

        // 发送 typing 状态
        String typing = objectMapper.writeValueAsString(Map.of(
                "type", "typing",
                "conversationId", wsMsg.getConversationId()));
        sendToSession(session, typing);

        // 生成 AI 回复（异步执行避免阻塞 WebSocket 线程）
        Long convId = wsMsg.getConversationId();
        CompletableFuture.runAsync(() -> {
            try {
                Message aiMsg = messageService.handleUserMessage(
                        userId, convId, wsMsg.getContentType(), wsMsg.getContent());

                // 模拟打字延迟
                long delay = messageService.calculateTypingDelay(aiMsg.getContent());
                Thread.sleep(delay);

                // 发送 AI 回复
                MessageData msgData = messageService.toData(aiMsg);
                String payload = objectMapper.writeValueAsString(Map.of(
                        "type", "new_message",
                        "conversationId", convId,
                        "message", msgData));
                sendToSession(session, payload);
            } catch (Exception e) {
                log.error("处理消息失败", e);
            }
        });
    } catch (Exception e) {
        log.error("handleSendMessage 失败", e);
    }
}

private void handleSync(Long userId, WsMessage wsMsg, WebSocketSession session) {
    try {
        // 获取用户所有会话的未读消息
        List<Conversation> convs = messageService.getUserConversations(userId);
        List<MessageData> allMessages = new java.util.ArrayList<>();
        for (Conversation conv : convs) {
            Long afterId = wsMsg.getLastMsgId() != null ? wsMsg.getLastMsgId() : 0L;
            List<Message> messages = messageService.syncMessages(conv.getId(), afterId, 100);
            allMessages.addAll(messages.stream().map(messageService::toData).toList());
        }

        String payload = objectMapper.writeValueAsString(Map.of(
                "type", "sync_result",
                "messages", allMessages,
                "hasMore", false));
        sendToSession(session, payload);
    } catch (Exception e) {
        log.error("handleSync 失败", e);
    }
}
```

需要在文件顶部添加：`import java.util.Map; import java.util.concurrent.CompletableFuture;`

- [ ] **Step 6: 实现 ConversationController**

```java
package com.sanyan.controller;

import com.sanyan.dto.ApiResponse;
import com.sanyan.dto.data.ConversationData;
import com.sanyan.dto.data.MessageData;
import com.sanyan.entity.AiCharacter;
import com.sanyan.entity.Conversation;
import com.sanyan.entity.Message;
import com.sanyan.repository.AiCharacterRepository;
import com.sanyan.repository.ConversationRepository;
import com.sanyan.repository.MessageRepository;
import com.sanyan.service.MessageService;
import com.sanyan.util.JwtUtil;
import jakarta.servlet.http.HttpServletRequest;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/conversations")
@RequiredArgsConstructor
public class ConversationController {

    private final MessageService messageService;
    private final ConversationRepository conversationRepository;
    private final AiCharacterRepository characterRepository;
    private final MessageRepository messageRepository;
    private final JwtUtil jwtUtil;

    @GetMapping
    public ApiResponse<List<ConversationData>> list(HttpServletRequest request) {
        Long userId = getUserId(request);
        List<Conversation> convs = messageService.getUserConversations(userId);
        List<ConversationData> result = convs.stream().map(conv -> {
            ConversationData d = new ConversationData();
            d.setId(conv.getId());
            d.setCharacterId(conv.getCharacterId());
            d.setLastMessageAt(conv.getLastMessageAt());
            d.setUnreadCount(conv.getUnreadCount());

            characterRepository.findById(conv.getCharacterId()).ifPresent(c -> {
                d.setCharacterName(c.getName());
                d.setCharacterAvatar(c.getAvatar());
            });

            // 最后一条消息预览
            messageRepository.findTopNByConversationIdOrderByIdDesc(conv.getId(), PageRequest.of(0, 1))
                    .stream().findFirst()
                    .ifPresent(msg -> d.setLastMessage(msg.getContent()));
            return d;
        }).toList();
        return ApiResponse.ok(result);
    }

    @GetMapping("/{id}/messages")
    public ApiResponse<List<MessageData>> messages(
            @PathVariable Long id,
            @RequestParam(required = false) Long beforeId,
            @RequestParam(defaultValue = "20") int limit) {
        List<Message> messages = messageService.getHistoryMessages(id, beforeId, limit);
        return ApiResponse.ok(messages.stream().map(messageService::toData).toList());
    }

    @PostMapping("/{id}/read")
    public ApiResponse<Void> markRead(@PathVariable Long id) {
        conversationRepository.findById(id).ifPresent(conv -> {
            conv.setUnreadCount(0);
            conversationRepository.save(conv);
        });
        return ApiResponse.ok();
    }

    private Long getUserId(HttpServletRequest request) {
        String token = request.getHeader("Authorization");
        if (token != null && token.startsWith("Bearer ")) token = token.substring(7);
        return jwtUtil.parseUserId(token);
    }
}
```

- [ ] **Step 7: 运行全量测试**

Run: `mvn test`
Expected: 全部 PASS

- [ ] **Step 8: 提交**

```bash
git add -A
git commit -m "消息收发 + 豆包 AI 对话 + 会话管理 + WebSocket 完整聊天流程"
git push
```

---

### Task 9: 记忆系统（画像提取 + 对话摘要）

**Files:**
- Create: `service/MemoryService.java`
- Test: `src/test/java/com/sanyan/service/MemoryServiceTest.java`

- [ ] **Step 1: 写测试**

```java
package com.sanyan.service;

import com.sanyan.entity.MemoryProfile;
import com.sanyan.entity.MemorySummary;
import com.sanyan.entity.Message;
import com.sanyan.repository.MemoryProfileRepository;
import com.sanyan.repository.MemorySummaryRepository;
import com.sanyan.repository.MessageRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class MemoryServiceTest {

    @Mock private MemoryProfileRepository profileRepository;
    @Mock private MemorySummaryRepository summaryRepository;
    @Mock private MessageRepository messageRepository;
    @Mock private AiService aiService;
    @InjectMocks private MemoryService memoryService;

    @Test
    void shouldBuildProfileUpdatePrompt() {
        String currentProfile = "小明，程序员，喜欢打篮球";
        List<Message> messages = List.of(
                createMessage("user", "我最近换工作了，去做设计了"),
                createMessage("ai", "哇，那挺大的转变啊")
        );

        String prompt = memoryService.buildProfileUpdatePrompt(currentProfile, messages);
        assertThat(prompt).contains("小明");
        assertThat(prompt).contains("换工作");
    }

    @Test
    void shouldBuildSummaryPrompt() {
        List<Message> messages = List.of(
                createMessage("user", "今天面试了一家公司"),
                createMessage("ai", "怎么样？顺利吗？"),
                createMessage("user", "还行，等通知吧")
        );

        String prompt = memoryService.buildSummaryPrompt(messages);
        assertThat(prompt).contains("面试");
    }

    private Message createMessage(String senderType, String content) {
        Message m = new Message();
        m.setSenderType(senderType);
        m.setContent(content);
        return m;
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `mvn test -Dtest=MemoryServiceTest`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现 MemoryService**

```java
package com.sanyan.service;

import com.sanyan.entity.MemoryProfile;
import com.sanyan.entity.MemorySummary;
import com.sanyan.entity.Message;
import com.sanyan.repository.MemoryProfileRepository;
import com.sanyan.repository.MemorySummaryRepository;
import com.sanyan.repository.MessageRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.PageRequest;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class MemoryService {

    private final MemoryProfileRepository profileRepository;
    private final MemorySummaryRepository summaryRepository;
    private final MessageRepository messageRepository;
    private final AiService aiService;

    /**
     * 异步更新记忆（一轮对话结束后调用）
     * @param conversationId 会话 ID
     * @param messageIds 这轮对话的消息 ID 列表
     */
    @Async
    public void updateMemory(Long conversationId, List<Long> messageIds) {
        if (messageIds.isEmpty()) return;

        Long startId = messageIds.get(0);
        Long endId = messageIds.get(messageIds.size() - 1);
        List<Message> messages = messageRepository
                .findByConversationIdAndIdBetweenOrderByIdAsc(conversationId, startId, endId);

        if (messages.isEmpty()) return;

        // 1. 更新用户画像
        try {
            String currentProfile = profileRepository.findByConversationId(conversationId)
                    .map(MemoryProfile::getContent).orElse("");
            String profilePrompt = buildProfileUpdatePrompt(currentProfile, messages);
            // 调豆包提取画像（用简单的通用模型即可，不需要角色人设）
            String updatedProfile = aiService.callForMemory(profilePrompt);

            MemoryProfile profile = profileRepository.findByConversationId(conversationId)
                    .orElseGet(() -> {
                        MemoryProfile p = new MemoryProfile();
                        p.setConversationId(conversationId);
                        return p;
                    });
            profile.setContent(updatedProfile);
            profileRepository.save(profile);
            log.info("会话 {} 用户画像已更新", conversationId);
        } catch (Exception e) {
            log.error("更新用户画像失败", e);
        }

        // 2. 生成对话摘要
        try {
            String summaryPrompt = buildSummaryPrompt(messages);
            String summaryText = aiService.callForMemory(summaryPrompt);

            MemorySummary summary = new MemorySummary();
            summary.setConversationId(conversationId);
            summary.setSummary(summaryText);
            summary.setMessageRange(startId + "-" + endId);
            summaryRepository.save(summary);
            log.info("会话 {} 对话摘要已生成", conversationId);
        } catch (Exception e) {
            log.error("生成对话摘要失败", e);
        }
    }

    public String buildProfileUpdatePrompt(String currentProfile, List<Message> messages) {
        String conversation = messages.stream()
                .map(m -> ("user".equals(m.getSenderType()) ? "用户" : "AI") + "：" + m.getContent())
                .collect(Collectors.joining("\n"));

        return "你是一个信息提取助手。根据以下对话内容，更新用户画像。\n\n"
                + "当前用户画像：\n" + (currentProfile.isBlank() ? "（空）" : currentProfile)
                + "\n\n本轮对话：\n" + conversation
                + "\n\n请输出更新后的完整用户画像。保留有价值的历史信息（如职业变动写成'曾是X，现在做Y'），"
                + "新增本轮对话中发现的新信息。如果没有值得记录的新信息，原样返回当前画像。"
                + "只输出画像内容，不要解释。";
    }

    public String buildSummaryPrompt(List<Message> messages) {
        String conversation = messages.stream()
                .map(m -> ("user".equals(m.getSenderType()) ? "用户" : "AI") + "：" + m.getContent())
                .collect(Collectors.joining("\n"));

        return "用一两句话总结以下对话的关键内容，重点记录用户提到的事实、计划、情绪。\n\n"
                + conversation + "\n\n摘要：";
    }
}
```

在 `AiService` 中添加 `callForMemory` 方法：

```java
/**
 * 用于记忆提取的简单调用（不需要角色人设）
 */
public String callForMemory(String prompt) {
    return callDoubao(prompt, List.of(), List.of());
}
```

- [ ] **Step 4: 添加对话轮次检测 + 触发记忆更新**

在 `MessageService` 中，使用 Redis 跟踪每轮对话的消息 ID，设置 10 分钟过期后触发记忆更新：

```java
// 在 MessageService 中添加：

private final StringRedisTemplate redisTemplate;
private final MemoryService memoryService;

// 在 handleUserMessage 方法最后添加：
// 记录本轮对话消息 ID（10 分钟 TTL）
String roundKey = "conv:round:" + conversationId;
redisTemplate.opsForList().rightPush(roundKey, String.valueOf(userMsg.getId()));
redisTemplate.opsForList().rightPush(roundKey, String.valueOf(aiMsg.getId()));
redisTemplate.expire(roundKey, 10, java.util.concurrent.TimeUnit.MINUTES);
```

添加定时任务扫描已结束的对话轮次，在 `scheduler/` 下创建：

```java
// scheduler/MemoryScheduler.java
package com.sanyan.scheduler;

import com.sanyan.service.MemoryService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Set;

@Slf4j
@Component
@RequiredArgsConstructor
public class MemoryScheduler {

    private final StringRedisTemplate redisTemplate;
    private final MemoryService memoryService;

    /**
     * 每分钟检查一次，对已过期（10分钟无新消息）的对话轮次触发记忆更新。
     * 实际上 Redis key 过期后无法再读取，所以改为：
     * 使用 conv:round:{id}:ts 记录最后消息时间，定时扫描超过 10 分钟的。
     */
    @Scheduled(fixedRate = 60000)
    public void checkRoundEnd() {
        Set<String> keys = redisTemplate.keys("conv:round:*:ts");
        if (keys == null) return;

        long now = System.currentTimeMillis();
        for (String tsKey : keys) {
            String tsStr = redisTemplate.opsForValue().get(tsKey);
            if (tsStr == null) continue;

            long lastMsgTime = Long.parseLong(tsStr);
            if (now - lastMsgTime > 600000) { // 10 分钟
                String convIdStr = tsKey.replace("conv:round:", "").replace(":ts", "");
                Long conversationId = Long.parseLong(convIdStr);
                String roundKey = "conv:round:" + convIdStr;

                List<String> msgIds = redisTemplate.opsForList().range(roundKey, 0, -1);
                if (msgIds != null && !msgIds.isEmpty()) {
                    List<Long> ids = msgIds.stream().map(Long::parseLong).toList();
                    memoryService.updateMemory(conversationId, ids);
                }

                redisTemplate.delete(roundKey);
                redisTemplate.delete(tsKey);
            }
        }
    }
}
```

同时在 `MessageService.handleUserMessage` 中更新时间戳：

```java
// 在 handleUserMessage 的记录轮次部分改为：
String roundKey = "conv:round:" + conversationId;
String roundTsKey = "conv:round:" + conversationId + ":ts";
redisTemplate.opsForList().rightPush(roundKey, String.valueOf(userMsg.getId()));
redisTemplate.opsForList().rightPush(roundKey, String.valueOf(aiMsg.getId()));
redisTemplate.opsForValue().set(roundTsKey, String.valueOf(System.currentTimeMillis()));
```

- [ ] **Step 5: 运行测试确认通过**

Run: `mvn test -Dtest=MemoryServiceTest`
Expected: PASS

- [ ] **Step 6: 提交**

```bash
git add -A
git commit -m "记忆系统：用户画像提取 + 对话摘要生成 + 轮次检测"
git push
```

---

### Task 10: AI 主动消息系统

**Files:**
- Create: `service/ProactiveService.java`
- Create: `scheduler/ProactiveScheduler.java`
- Test: `src/test/java/com/sanyan/service/ProactiveServiceTest.java`

- [ ] **Step 1: 写测试**

```java
package com.sanyan.service;

import com.sanyan.entity.AiCharacter;
import com.sanyan.entity.Conversation;
import com.sanyan.entity.Message;
import com.sanyan.repository.AiCharacterRepository;
import com.sanyan.repository.ConversationRepository;
import com.sanyan.repository.MessageRepository;
import com.sanyan.websocket.SessionManager;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;

import java.time.LocalDateTime;
import java.util.List;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ProactiveServiceTest {

    @Mock private ConversationRepository conversationRepository;
    @Mock private MessageRepository messageRepository;
    @Mock private AiCharacterRepository characterRepository;
    @Mock private AiService aiService;
    @Mock private SessionManager sessionManager;
    @Mock private StringRedisTemplate redisTemplate;
    @Mock private ValueOperations<String, String> valueOps;
    @Mock private PushService pushService;

    @InjectMocks private ProactiveService proactiveService;

    @Test
    void shouldCheckRateLimitCorrectly() {
        when(redisTemplate.opsForValue()).thenReturn(valueOps);
        when(valueOps.get("proactive:daily:1")).thenReturn("2");
        assertThat(proactiveService.isRateLimited(1L, 3)).isFalse();

        when(valueOps.get("proactive:daily:1")).thenReturn("3");
        assertThat(proactiveService.isRateLimited(1L, 3)).isTrue();
    }

    @Test
    void shouldCheckLastProactiveUnanswered() {
        Message lastMsg = new Message();
        lastMsg.setSenderType("ai");
        lastMsg.setSource("proactive");
        when(messageRepository.findTopNByConversationIdOrderByIdDesc(eq(100L), any()))
                .thenReturn(List.of(lastMsg));

        assertThat(proactiveService.hasUnansweredProactive(100L)).isTrue();
    }

    @Test
    void shouldNotFlagWhenUserReplied() {
        Message lastMsg = new Message();
        lastMsg.setSenderType("user");
        lastMsg.setSource("reply");
        when(messageRepository.findTopNByConversationIdOrderByIdDesc(eq(100L), any()))
                .thenReturn(List.of(lastMsg));

        assertThat(proactiveService.hasUnansweredProactive(100L)).isFalse();
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `mvn test -Dtest=ProactiveServiceTest`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现 ProactiveService**

```java
package com.sanyan.service;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.entity.AiCharacter;
import com.sanyan.entity.Conversation;
import com.sanyan.entity.Message;
import com.sanyan.repository.AiCharacterRepository;
import com.sanyan.repository.ConversationRepository;
import com.sanyan.repository.MessageRepository;
import com.sanyan.websocket.SessionManager;
import com.sanyan.websocket.WebSocketHandler;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.web.socket.WebSocketSession;

import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProactiveService {

    private final ConversationRepository conversationRepository;
    private final MessageRepository messageRepository;
    private final AiCharacterRepository characterRepository;
    private final AiService aiService;
    private final SessionManager sessionManager;
    private final StringRedisTemplate redisTemplate;
    private final PushService pushService;
    private final ObjectMapper objectMapper;
    private static final String DAILY_COUNT_PREFIX = "proactive:daily:";
    private static final String LAST_PROACTIVE_PREFIX = "proactive:last:";

    /**
     * 发送主动消息给指定会话
     */
    public void sendProactiveMessage(Conversation conv, AiCharacter character, String triggerHint) {
        Long userId = conv.getUserId();

        // 频率检查
        JsonNode config = parseConfig(character.getProactiveConfig());
        int maxDaily = config.path("max_daily").asInt(3);
        if (isRateLimited(userId, maxDaily)) {
            log.debug("用户 {} 今日主动消息已达上限", userId);
            return;
        }

        // 检查是否有未回复的主动消息
        if (hasUnansweredProactive(conv.getId())) {
            log.debug("会话 {} 上一条主动消息未回复，暂停发送", conv.getId());
            return;
        }

        // 检查最小间隔
        int minInterval = config.path("min_interval_hours").asInt(2);
        if (isTooSoon(userId, minInterval)) {
            log.debug("用户 {} 距上次主动消息不足 {} 小时", userId, minInterval);
            return;
        }

        // 生成消息
        String content = aiService.chatProactive(character, conv.getId(), triggerHint);

        // 保存消息
        Message msg = new Message();
        msg.setConversationId(conv.getId());
        msg.setSenderType("ai");
        msg.setContentType("text");
        msg.setContent(content);
        msg.setSource("proactive");
        messageRepository.save(msg);

        // 更新会话
        conv.setLastMessageAt(LocalDateTime.now());
        conv.setUnreadCount(conv.getUnreadCount() + 1);
        conversationRepository.save(conv);

        // 更新计数
        incrementDailyCount(userId);
        updateLastProactiveTime(userId);

        // 投递
        if (sessionManager.isOnline(userId)) {
            deliverViaWebSocket(userId, conv.getId(), msg);
        } else {
            pushService.sendPush(userId, character.getName() + "给你发了一条消息");
        }

        log.info("主动消息已发送: 用户={}, 会话={}, 类型={}", userId, conv.getId(), triggerHint);
    }

    public boolean isRateLimited(Long userId, int maxDaily) {
        String count = redisTemplate.opsForValue().get(DAILY_COUNT_PREFIX + userId);
        return count != null && Integer.parseInt(count) >= maxDaily;
    }

    public boolean hasUnansweredProactive(Long conversationId) {
        List<Message> lastMessages = messageRepository
                .findTopNByConversationIdOrderByIdDesc(conversationId, PageRequest.of(0, 1));
        if (lastMessages.isEmpty()) return false;
        Message last = lastMessages.get(0);
        return "ai".equals(last.getSenderType()) && "proactive".equals(last.getSource());
    }

    private boolean isTooSoon(Long userId, int minIntervalHours) {
        String lastTime = redisTemplate.opsForValue().get(LAST_PROACTIVE_PREFIX + userId);
        if (lastTime == null) return false;
        long lastMs = Long.parseLong(lastTime);
        return System.currentTimeMillis() - lastMs < minIntervalHours * 3600000L;
    }

    private void incrementDailyCount(Long userId) {
        String key = DAILY_COUNT_PREFIX + userId;
        redisTemplate.opsForValue().increment(key);
        // 设置到今天结束的 TTL
        long secondsUntilMidnight = LocalDateTime.now().until(
                LocalDateTime.now().toLocalDate().plusDays(1).atStartOfDay(), ChronoUnit.SECONDS);
        redisTemplate.expire(key, secondsUntilMidnight, TimeUnit.SECONDS);
    }

    private void updateLastProactiveTime(Long userId) {
        redisTemplate.opsForValue().set(LAST_PROACTIVE_PREFIX + userId,
                String.valueOf(System.currentTimeMillis()));
    }

    private void deliverViaWebSocket(Long userId, Long conversationId, Message msg) {
        sessionManager.getSession(userId).ifPresent(session -> {
            try {
                // typing
                String typing = objectMapper.writeValueAsString(java.util.Map.of(
                        "type", "typing", "conversationId", conversationId));
                session.sendMessage(new org.springframework.web.socket.TextMessage(typing));

                // 延迟后发消息
                long delay = Math.min(msg.getContent().length() * 120L, 8000L);
                Thread.sleep(delay);

                MessageService messageService = null; // 通过构造函数注入，此处简化
                String payload = objectMapper.writeValueAsString(java.util.Map.of(
                        "type", "new_message",
                        "conversationId", conversationId,
                        "message", java.util.Map.of(
                                "id", msg.getId(),
                                "senderType", msg.getSenderType(),
                                "contentType", msg.getContentType(),
                                "content", msg.getContent(),
                                "source", msg.getSource(),
                                "createdAt", msg.getCreatedAt().toString()
                        )));
                session.sendMessage(new org.springframework.web.socket.TextMessage(payload));
            } catch (Exception e) {
                log.error("WebSocket 推送主动消息失败", e);
            }
        });
    }

    private JsonNode parseConfig(String configJson) {
        try {
            return configJson != null ? objectMapper.readTree(configJson)
                    : objectMapper.createObjectNode();
        } catch (Exception e) {
            return objectMapper.createObjectNode();
        }
    }
}
```

- [ ] **Step 4: 实现 ProactiveScheduler**

```java
package com.sanyan.scheduler;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.entity.AiCharacter;
import com.sanyan.entity.Conversation;
import com.sanyan.repository.AiCharacterRepository;
import com.sanyan.repository.ConversationRepository;
import com.sanyan.service.ProactiveService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.time.LocalTime;
import java.util.List;
import java.util.Random;

@Slf4j
@Component
@RequiredArgsConstructor
public class ProactiveScheduler {

    private final ConversationRepository conversationRepository;
    private final AiCharacterRepository characterRepository;
    private final ProactiveService proactiveService;
    private final ObjectMapper objectMapper;
    private static final Random RANDOM = new Random();

    /**
     * 定时关怀 — 每 10 分钟检查一次
     */
    @Scheduled(fixedRate = 600000)
    public void greetingCheck() {
        LocalTime now = LocalTime.now();
        List<AiCharacter> characters = characterRepository.findByType("preset");

        for (AiCharacter character : characters) {
            JsonNode config = parseConfig(character.getProactiveConfig());
            if (!config.path("greeting").path("enabled").asBoolean(false)) continue;

            JsonNode slots = config.path("greeting").path("slots");
            boolean inSlot = false;
            for (JsonNode slot : slots) {
                String[] parts = slot.asText().split("-");
                LocalTime start = LocalTime.parse(parts[0]);
                LocalTime end = LocalTime.parse(parts[1]);
                if (now.isAfter(start) && now.isBefore(end)) {
                    inSlot = true;
                    break;
                }
            }
            if (!inSlot) continue;

            List<Conversation> convs = conversationRepository.findAll().stream()
                    .filter(c -> c.getCharacterId().equals(character.getId()))
                    .toList();

            for (Conversation conv : convs) {
                proactiveService.sendProactiveMessage(conv, character,
                        "现在是" + now.getHour() + "点，请自然地问候用户，不要生硬地说'早上好'之类的模板话。");
            }
        }
    }

    /**
     * 事件触发 — 每 30 分钟检查
     */
    @Scheduled(fixedRate = 1800000)
    public void eventTriggerCheck() {
        List<AiCharacter> characters = characterRepository.findByType("preset");

        for (AiCharacter character : characters) {
            JsonNode config = parseConfig(character.getProactiveConfig());
            if (!config.path("event_trigger").path("enabled").asBoolean(false)) continue;

            int idleThreshold = config.path("event_trigger").path("idle_hours_threshold").asInt(6);
            LocalDateTime threshold = LocalDateTime.now().minusHours(idleThreshold);
            int activeStart = config.path("active_hours").get(0).asInt(8);
            int activeEnd = config.path("active_hours").get(1).asInt(22);
            int currentHour = LocalTime.now().getHour();
            if (currentHour < activeStart || currentHour >= activeEnd) continue;

            List<Conversation> convs = conversationRepository.findAll().stream()
                    .filter(c -> c.getCharacterId().equals(character.getId()))
                    .filter(c -> c.getLastMessageAt() != null && c.getLastMessageAt().isBefore(threshold))
                    .toList();

            for (Conversation conv : convs) {
                proactiveService.sendProactiveMessage(conv, character,
                        "用户已经超过" + idleThreshold + "小时没有跟你聊天了。用你的方式自然地找用户聊天，不要说'好久没聊了'这种话。");
            }
        }
    }

    /**
     * 情境模拟 — 每小时随机触发
     */
    @Scheduled(fixedRate = 3600000)
    public void situationalCheck() {
        int currentHour = LocalTime.now().getHour();
        if (currentHour < 9 || currentHour >= 22) return;

        List<AiCharacter> characters = characterRepository.findByType("preset");

        for (AiCharacter character : characters) {
            JsonNode config = parseConfig(character.getProactiveConfig());
            if (!config.path("situational").path("enabled").asBoolean(false)) continue;

            // 随机决定这个小时是否触发（概率 = dailyCount / 活跃小时数）
            JsonNode dailyCount = config.path("situational").path("daily_count");
            int maxPerDay = dailyCount.isArray() ? dailyCount.get(1).asInt(2) : 2;
            double probability = (double) maxPerDay / 13.0; // 9:00-22:00 = 13 小时
            if (RANDOM.nextDouble() > probability) continue;

            List<Conversation> convs = conversationRepository.findAll().stream()
                    .filter(c -> c.getCharacterId().equals(character.getId()))
                    .toList();

            for (Conversation conv : convs) {
                proactiveService.sendProactiveMessage(conv, character,
                        "请分享一件你（角色）今天遇到的有趣的事、看到的东西、或者想到的话题。要自然，像真人朋友随手发消息一样。");
            }
        }
    }

    private JsonNode parseConfig(String json) {
        try {
            return json != null ? objectMapper.readTree(json) : objectMapper.createObjectNode();
        } catch (Exception e) {
            return objectMapper.createObjectNode();
        }
    }
}
```

- [ ] **Step 5: 运行测试确认通过**

Run: `mvn test -Dtest=ProactiveServiceTest`
Expected: PASS

- [ ] **Step 6: 提交**

```bash
git add -A
git commit -m "AI 主动消息系统：三种触发机制 + 频率限制 + 防骚扰"
git push
```

---

### Task 11: 推送通知 + 设备 Token API

**Files:**
- Create: `service/PushService.java`
- Create: `controller/DeviceController.java`
- Test: `src/test/java/com/sanyan/service/PushServiceTest.java`

MVP 阶段先写接口框架和日志输出，真实推送对接（极光/个推 SDK）在部署前补充。

- [ ] **Step 1: 写测试**

```java
package com.sanyan.service;

import com.sanyan.entity.UserToken;
import com.sanyan.repository.UserTokenRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class PushServiceTest {

    @Mock private UserTokenRepository tokenRepository;
    @InjectMocks private PushService pushService;

    @Test
    void shouldUpdatePushToken() {
        UserToken existing = new UserToken();
        existing.setUserId(1L);
        existing.setPushToken("old_token");
        when(tokenRepository.findByUserId(1L)).thenReturn(Optional.of(existing));

        pushService.updatePushToken(1L, "new_token", "ios");

        verify(tokenRepository).save(argThat(t -> "new_token".equals(t.getPushToken())));
    }

    @Test
    void shouldCreateTokenIfNotExists() {
        when(tokenRepository.findByUserId(2L)).thenReturn(Optional.empty());

        pushService.updatePushToken(2L, "token_123", "android");

        verify(tokenRepository).save(argThat(t ->
                t.getUserId().equals(2L) && "token_123".equals(t.getPushToken())));
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `mvn test -Dtest=PushServiceTest`
Expected: FAIL — 编译错误

- [ ] **Step 3: 实现 PushService + DeviceController**

```java
// service/PushService.java
package com.sanyan.service;

import com.sanyan.entity.UserToken;
import com.sanyan.repository.UserTokenRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class PushService {

    private final UserTokenRepository tokenRepository;

    public void updatePushToken(Long userId, String pushToken, String deviceType) {
        UserToken userToken = tokenRepository.findByUserId(userId)
                .orElseGet(() -> {
                    UserToken t = new UserToken();
                    t.setUserId(userId);
                    return t;
                });
        userToken.setPushToken(pushToken);
        userToken.setDeviceType(deviceType);
        tokenRepository.save(userToken);
    }

    public void sendPush(Long userId, String content) {
        tokenRepository.findByUserId(userId).ifPresent(token -> {
            if (token.getPushToken() != null) {
                // TODO: 对接极光/个推 SDK
                log.info("推送通知: userId={}, token={}, content={}", userId, token.getPushToken(), content);
            }
        });
    }
}
```

```java
// controller/DeviceController.java
package com.sanyan.controller;

import com.sanyan.dto.ApiResponse;
import com.sanyan.service.PushService;
import com.sanyan.util.JwtUtil;
import jakarta.servlet.http.HttpServletRequest;
import lombok.Data;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/device")
@RequiredArgsConstructor
public class DeviceController {

    private final PushService pushService;
    private final JwtUtil jwtUtil;

    @PostMapping("/push-token")
    public ApiResponse<Void> updatePushToken(HttpServletRequest request, @RequestBody PushTokenReq req) {
        String token = request.getHeader("Authorization");
        if (token != null && token.startsWith("Bearer ")) token = token.substring(7);
        Long userId = jwtUtil.parseUserId(token);

        pushService.updatePushToken(userId, req.getPushToken(), req.getDeviceType());
        return ApiResponse.ok();
    }

    @Data
    static class PushTokenReq {
        private String pushToken;
        private String deviceType;
    }
}
```

- [ ] **Step 4: 运行全量测试**

Run: `mvn test`
Expected: 全部 PASS

- [ ] **Step 5: 提交**

```bash
git add -A
git commit -m "推送通知服务 + 设备 Token API"
git push
```

---

### Task 12: 最终整合 + 全量测试

- [ ] **Step 1: 添加 @EnableAsync 支持异步记忆更新**

在 `SanyanApplication.java` 添加 `@EnableAsync` 注解。

- [ ] **Step 2: 添加测试配置文件**

`src/test/resources/application.yml`:
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
  data:
    redis:
      host: localhost
      port: 6379
  sql:
    init:
      mode: never

sanyan:
  jwt:
    secret: test-secret-key-at-least-256-bits-long-for-hmac-sha-testing
    expiration-days: 1
  doubao:
    api-key: test
    model: test
    endpoint: http://localhost:9999
```

- [ ] **Step 3: 运行全量测试确认全部通过**

Run: `mvn test`
Expected: 全部 PASS

- [ ] **Step 4: 提交**

```bash
git add -A
git commit -m "最终整合：异步支持 + 测试配置"
git push
```

---

## 后端计划完毕

12 个 Task 覆盖 MVP 所有后端功能：
1. 项目初始化
2. 数据库实体
3. JWT 工具
4. 短信验证码
5. 用户认证 API
6. 用户信息 + 角色 API
7. WebSocket 基础设施
8. 消息收发 + AI 对话
9. 记忆系统
10. 主动消息系统
11. 推送通知
12. 最终整合
