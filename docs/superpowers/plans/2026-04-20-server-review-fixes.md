# Server Code Review 修复 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 server 端 2026-04-20 code review 报告中的 P1/P2 问题（安全、数据一致性、性能、可观测性、运维安全），每项改动都先写失败测试再实现。

**Architecture:** 全部在 `server` git submodule 的新分支 `fix/review-findings` 上工作。每个修复独立成 task，每个 task 独立 commit。测试用 Mockito/JUnit 5（service 层）+ `@DataJpaTest`（repository 层）+ `@SpringBootTest` + `MockMvc`（controller 层）。

**Tech Stack:** Spring Boot 3.2.5 · JPA/Hibernate · MySQL · Redis · JUnit 5 · Mockito · AssertJ

**范围外（本次不做）：**
- 密钥轮换（用户声明故意保留）
- SessionManager 改造为 Redis Pub/Sub（保持单机假设，本 plan 仅加文档）
- SMS SDK 接入（无阿里云 AccessKey）

---

## File Structure

**新建：** 无

**修改（server/src/main/java/com/sanyan/）：**
- `service/MessageService.java` — 事务、权限校验、senderType 修正、N+1 修复、fallback 标记
- `controller/ConversationController.java` — 简化（service 已兜底）
- `entity/Message.java` — 加 `fallbackReason` 字段、`createdAt` 索引
- `entity/Conversation.java` — 加 `lastMessageAt` 索引
- `dto/data/MessageData.java` — 透出 `fallbackReason`
- `websocket/WebSocketHandler.java` — 异步任务失败时推 error 事件给客户端
- `websocket/SessionManager.java` — 加单机假设文档 + TODO
- `dto/ws/WsEventType.java` — 新增 `ERROR` 事件类型
- `resources/application.yml` — 生产 profile 关闭 `show-sql`、`ddl-auto` 限制

**修改（server/src/test/java/com/sanyan/）：**
- `service/MessageServicePermissionTest.java`（新建）
- `service/MessageServiceTransactionalTest.java`（新建）
- `service/MessageServiceSenderTypeTest.java`（新建）
- `service/MessageServiceConversationListTest.java`（新建）
- `service/MessageServiceFallbackReasonTest.java`（新建）
- `websocket/WebSocketHandlerErrorTest.java`（新建）

---

## Task 1: 建分支

**Files:** 无代码改动，仅 git 操作。

- [ ] **Step 1: 在 server 子模块建新分支**

```bash
cd /Users/aventador/code/3yan/server
git checkout -b fix/review-findings
git status  # 确认在新分支，工作区干净
```

Expected: `On branch fix/review-findings` + `nothing to commit, working tree clean`

- [ ] **Step 2: 在主工程建同名分支**

```bash
cd /Users/aventador/code/3yan
git checkout -b fix/review-findings
```

---

## Task 2: Service 层兜底权限校验（getHistoryMessages）

**问题：** `MessageService.getHistoryMessages` 不校验 userId，只靠 Controller 层把关。任何代码路径绕过 Controller 都会泄露他人对话。

**Files:**
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java`
- Test: `server/src/test/java/com/sanyan/service/MessageServicePermissionTest.java`

- [ ] **Step 1: 写失败测试**

Create `server/src/test/java/com/sanyan/service/MessageServicePermissionTest.java`:

```java
package com.sanyan.service;

import com.sanyan.entity.Conversation;
import com.sanyan.repository.AiCharacterRepository;
import com.sanyan.repository.ConversationRepository;
import com.sanyan.repository.MessageRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.redis.core.StringRedisTemplate;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class MessageServicePermissionTest {

    @Mock ConversationRepository conversationRepository;
    @Mock MessageRepository messageRepository;
    @Mock AiCharacterRepository characterRepository;
    @Mock AiService aiService;
    @Mock TtsService ttsService;
    @Mock CosService cosService;
    @Mock AsrService asrService;
    @Mock StringRedisTemplate redisTemplate;
    @InjectMocks MessageService service;

    @Test
    void getHistoryMessages_whenUserNotOwner_throwsForbidden() {
        Conversation conv = new Conversation();
        conv.setId(10L);
        conv.setUserId(1L);
        when(conversationRepository.findById(10L)).thenReturn(Optional.of(conv));

        assertThatThrownBy(() -> service.getHistoryMessages(2L, 10L, null, 20))
                .isInstanceOf(SecurityException.class)
                .hasMessageContaining("无权访问");
    }

    @Test
    void getHistoryMessages_whenConversationMissing_throwsNotFound() {
        when(conversationRepository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.getHistoryMessages(1L, 99L, null, 20))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("会话不存在");
    }
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
cd /Users/aventador/code/3yan/server
./mvnw test -Dtest=MessageServicePermissionTest
```

Expected: FAIL（编译失败：`getHistoryMessages` 现在签名只有 `(Long, Long, int)`，没有 userId 参数）

- [ ] **Step 3: 改签名 + 加权限校验**

在 `MessageService.java` 修改 `getHistoryMessages`（原 207-217 行附近）：

```java
public List<Message> getHistoryMessages(Long userId, Long conversationId, Long beforeMsgId, int limit) {
    Conversation conv = conversationRepository.findById(conversationId)
            .orElseThrow(() -> new IllegalArgumentException("会话不存在"));
    if (!conv.getUserId().equals(userId)) {
        throw new SecurityException("无权访问该会话");
    }
    List<Message> messages;
    if (beforeMsgId == null || beforeMsgId <= 0) {
        messages = messageRepository.findByConversationIdOrderByIdDesc(conversationId, PageRequest.of(0, limit));
    } else {
        messages = messageRepository.findByConversationIdAndIdLessThanOrderByIdDesc(
                conversationId, beforeMsgId, PageRequest.of(0, limit));
    }
    Collections.reverse(messages);
    return messages;
}
```

同时把 `syncMessages` 也加上 userId 校验：

```java
public List<Message> syncMessages(Long userId, Long conversationId, Long afterMsgId, int limit) {
    Conversation conv = conversationRepository.findById(conversationId)
            .orElseThrow(() -> new IllegalArgumentException("会话不存在"));
    if (!conv.getUserId().equals(userId)) {
        throw new SecurityException("无权访问该会话");
    }
    if (afterMsgId == null || afterMsgId <= 0) {
        List<Message> messages = messageRepository.findByConversationIdOrderByIdDesc(
                conversationId, PageRequest.of(0, limit));
        Collections.reverse(messages);
        return messages;
    }
    return messageRepository.findByConversationIdAndIdGreaterThanOrderByIdAsc(
            conversationId, afterMsgId, PageRequest.of(0, limit));
}
```

- [ ] **Step 4: 更新 caller 传 userId**

`ConversationController.java` 的 `messages` 方法：

```java
@GetMapping("/{id}/messages")
public ApiResponse<List<MessageData>> messages(
        @PathVariable Long id,
        @RequestParam(required = false) Long beforeId,
        @RequestParam(defaultValue = "20") int limit,
        HttpServletRequest request) {
    Long userId = getUserId(request);
    try {
        List<Message> messages = messageService.getHistoryMessages(userId, id, beforeId, limit);
        List<MessageData> data = messages.stream().map(messageService::toData).toList();
        return ApiResponse.ok(data);
    } catch (IllegalArgumentException | SecurityException e) {
        return ApiResponse.fail(e.getMessage());
    }
}
```

`WebSocketHandler.handleSync` 调用处（100-107 行）改为：

```java
List<Message> messages = messageService.syncMessages(
        userId, conv.getId(), wsMsg.getLastMsgId(), 50);
```

- [ ] **Step 5: 跑测试确认绿**

```bash
./mvnw test -Dtest=MessageServicePermissionTest
```

Expected: PASS

- [ ] **Step 6: 全量测试**

```bash
./mvnw test
```

Expected: BUILD SUCCESS

- [ ] **Step 7: 提交**

```bash
git add -A
git commit -m "fix: service 层兜底校验会话所有权

getHistoryMessages/syncMessages 原本只靠 Controller 层做权限校验，
Service 层对任意 userId 都放行。补上 ownership 校验，避免未来新增
调用路径绕过。"
```

---

## Task 3: `handleUserMessage` 加事务

**问题：** `handleUserMessage` 内部多次 save（user msg、AI msg、conversation），中途失败会产出半成品记录。

**Files:**
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java`
- Test: `server/src/test/java/com/sanyan/service/MessageServiceTransactionalTest.java`

- [ ] **Step 1: 写失败测试（验证 @Transactional 已声明）**

Create `server/src/test/java/com/sanyan/service/MessageServiceTransactionalTest.java`:

```java
package com.sanyan.service;

import org.junit.jupiter.api.Test;
import org.springframework.transaction.annotation.Transactional;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;

class MessageServiceTransactionalTest {

    @Test
    void handleUserMessage_isAnnotatedWithTransactional() throws NoSuchMethodException {
        Method m = MessageService.class.getMethod(
                "handleUserMessage", Long.class, Long.class, String.class, String.class, String.class, Integer.class);
        Transactional tx = m.getAnnotation(Transactional.class);
        assertThat(tx).as("handleUserMessage 必须带 @Transactional 保护多次 save 的原子性").isNotNull();
    }
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
./mvnw test -Dtest=MessageServiceTransactionalTest
```

Expected: FAIL（`tx` 为 null）

- [ ] **Step 3: 加注解**

在 `MessageService.java` 的 `handleUserMessage` 方法上加 `@Transactional`，并在文件顶部补 import：

```java
import org.springframework.transaction.annotation.Transactional;
```

方法签名改为：

```java
@Transactional
public Message handleUserMessage(Long userId, Long conversationId, String contentType, String content,
                                  String mediaUrl, Integer duration) {
```

- [ ] **Step 4: 跑测试确认绿**

```bash
./mvnw test -Dtest=MessageServiceTransactionalTest
```

Expected: PASS

- [ ] **Step 5: 全量测试 + 提交**

```bash
./mvnw test
git add -A
git commit -m "fix: handleUserMessage 加 @Transactional

handleUserMessage 内部多次 save（user msg、AI msg、conversation），
中途失败（如 COS 上传、TTS 合成抛出）会留下不一致数据。加事务保证原子。"
```

---

## Task 4: 统一 senderType 常量

**问题：** `MessageService.java:134` 硬编码 `"ai"`，其余地方用 `SenderType.AI`。

**Files:**
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java:134`
- Test: `server/src/test/java/com/sanyan/service/MessageServiceSenderTypeTest.java`

- [ ] **Step 1: 写失败测试**

Create `server/src/test/java/com/sanyan/service/MessageServiceSenderTypeTest.java`:

```java
package com.sanyan.service;

import com.sanyan.dto.ws.SenderType;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class MessageServiceSenderTypeTest {

    @Test
    void source_noHardcodedAiString() throws IOException {
        String source = Files.readString(Path.of(
                "src/main/java/com/sanyan/service/MessageService.java"));
        // 允许 javadoc 和常量引用里包含 "ai"；禁止 setSenderType("ai") 这种硬编码
        assertThat(source)
                .as("应使用 SenderType.AI 常量而不是硬编码 \"ai\"")
                .doesNotContain("setSenderType(\"ai\")");
    }

    @Test
    void senderType_constantMatches() {
        // 防止常量值被意外改动，兜底回归
        assertThat(SenderType.AI).isEqualTo("ai");
        assertThat(SenderType.USER).isEqualTo("user");
    }
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
./mvnw test -Dtest=MessageServiceSenderTypeTest
```

Expected: FAIL（source 包含 `setSenderType("ai")`）

- [ ] **Step 3: 替换**

`MessageService.java:134`：

```java
aiMsg.setSenderType(SenderType.AI);
```

- [ ] **Step 4: 跑测试确认绿 + 提交**

```bash
./mvnw test -Dtest=MessageServiceSenderTypeTest
./mvnw test
git add -A
git commit -m "fix: 统一使用 SenderType.AI 常量替换硬编码字符串"
```

---

## Task 5: 修复对话列表 N+1 查询

**问题：** `toConversationData` 对每个 conversation 额外查 1 次 Character + 1 次 last message。20 条 = 41 次 SQL。

**方案：** 批量查询。先把所有 `characterId` 和 `conversationId` 收集起来，一次性查 Character Map、用自定义 JPQL 查每个会话的最后一条消息。

**Files:**
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java`
- Modify: `server/src/main/java/com/sanyan/repository/MessageRepository.java`
- Modify: `server/src/main/java/com/sanyan/repository/AiCharacterRepository.java`
- Test: `server/src/test/java/com/sanyan/service/MessageServiceConversationListTest.java`

- [ ] **Step 1: 写失败测试**

Create `server/src/test/java/com/sanyan/service/MessageServiceConversationListTest.java`:

```java
package com.sanyan.service;

import com.sanyan.entity.AiCharacter;
import com.sanyan.entity.Conversation;
import com.sanyan.entity.Message;
import com.sanyan.repository.AiCharacterRepository;
import com.sanyan.repository.ConversationRepository;
import com.sanyan.repository.MessageRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.redis.core.StringRedisTemplate;

import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class MessageServiceConversationListTest {

    @Mock ConversationRepository conversationRepository;
    @Mock MessageRepository messageRepository;
    @Mock AiCharacterRepository characterRepository;
    @Mock AiService aiService;
    @Mock TtsService ttsService;
    @Mock CosService cosService;
    @Mock AsrService asrService;
    @Mock StringRedisTemplate redisTemplate;
    @InjectMocks MessageService service;

    @Test
    void getUserConversations_usesBatchQueries_notNPlusOne() {
        Conversation c1 = conv(1L, 100L);
        Conversation c2 = conv(2L, 100L); // 同角色
        Conversation c3 = conv(3L, 200L);
        when(conversationRepository.findByUserIdOrderByLastMessageAtDesc(7L))
                .thenReturn(List.of(c1, c2, c3));
        when(characterRepository.findAllById(any()))
                .thenReturn(List.of(character(100L, "A"), character(200L, "B")));
        when(messageRepository.findLastMessagePerConversation(any()))
                .thenReturn(List.of(
                        lastMsg(1L, "hi1"),
                        lastMsg(2L, "hi2"),
                        lastMsg(3L, "hi3")
                ));

        var result = service.getUserConversations(7L);

        assertThat(result).hasSize(3);
        // character lookup 只执行一次批量查询，不按 id 逐条查
        verify(characterRepository, times(1)).findAllById(any());
        verify(characterRepository, never()).findById(anyLong());
        // last message 只查一次批量
        verify(messageRepository, times(1)).findLastMessagePerConversation(any());
        verify(messageRepository, never()).findByConversationIdOrderByIdDesc(anyLong(), any());
    }

    private Conversation conv(long id, long charId) {
        Conversation c = new Conversation();
        c.setId(id);
        c.setUserId(7L);
        c.setCharacterId(charId);
        c.setUnreadCount(0);
        return c;
    }

    private AiCharacter character(long id, String name) {
        AiCharacter a = new AiCharacter();
        a.setId(id);
        a.setName(name);
        return a;
    }

    private Message lastMsg(long convId, String content) {
        Message m = new Message();
        m.setConversationId(convId);
        m.setContent(content);
        return m;
    }
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
./mvnw test -Dtest=MessageServiceConversationListTest
```

Expected: FAIL（`findLastMessagePerConversation` 未定义）

- [ ] **Step 3: 加 Repository 方法**

`MessageRepository.java` 新增：

```java
import com.sanyan.dto.LastMessageProjection;
import org.springframework.data.jpa.repository.Query;
import java.util.Collection;

@Query("""
    SELECT m FROM Message m
    WHERE m.conversationId IN :convIds
      AND m.id IN (
        SELECT MAX(m2.id) FROM Message m2
        WHERE m2.conversationId IN :convIds
        GROUP BY m2.conversationId
      )
""")
List<Message> findLastMessagePerConversation(@Param("convIds") Collection<Long> convIds);
```

（注意：`@Param` import `org.springframework.data.repository.query.Param`。）

- [ ] **Step 4: 改 `toConversationData` 的调用方式**

把 `getUserConversations` 改为一次性批量查：

```java
public List<ConversationData> getUserConversations(Long userId) {
    List<Conversation> conversations = conversationRepository.findByUserIdOrderByLastMessageAtDesc(userId);
    if (conversations.isEmpty()) return List.of();

    Set<Long> charIds = conversations.stream()
            .map(Conversation::getCharacterId).collect(Collectors.toSet());
    Set<Long> convIds = conversations.stream()
            .map(Conversation::getId).collect(Collectors.toSet());

    Map<Long, AiCharacter> charMap = characterRepository.findAllById(charIds).stream()
            .collect(Collectors.toMap(AiCharacter::getId, c -> c));
    Map<Long, Message> lastMsgMap = messageRepository.findLastMessagePerConversation(convIds).stream()
            .collect(Collectors.toMap(Message::getConversationId, m -> m));

    return conversations.stream()
            .map(conv -> toConversationData(conv, charMap.get(conv.getCharacterId()),
                    lastMsgMap.get(conv.getId())))
            .toList();
}

private ConversationData toConversationData(Conversation conv, AiCharacter character, Message lastMessage) {
    ConversationData d = new ConversationData();
    d.setId(conv.getId());
    d.setCharacterId(conv.getCharacterId());
    d.setLastMessageAt(conv.getLastMessageAt());
    d.setUnreadCount(conv.getUnreadCount());
    if (character != null) {
        d.setCharacterName(character.getName());
        d.setCharacterAvatar(character.getAvatar());
    }
    if (lastMessage != null) {
        d.setLastMessage(lastMessage.getContent());
    }
    return d;
}
```

文件顶部补 import：

```java
import com.sanyan.entity.AiCharacter;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;
```

- [ ] **Step 5: 跑测试确认绿 + 全量测试**

```bash
./mvnw test -Dtest=MessageServiceConversationListTest
./mvnw test
```

- [ ] **Step 6: 提交**

```bash
git add -A
git commit -m "perf: 消除对话列表 N+1 查询

原先 toConversationData 对每个 Conversation 额外发 1 次 character
查询和 1 次 last message 查询。20 个对话 = 41 次 SQL。改为两次批量
IN 查询 + 内存 map 组装。"
```

---

## Task 6: WebSocketHandler 异步失败推 error 给客户端

**问题：** `handleSendMessage` 的 `CompletableFuture.runAsync` 失败后只 log.error，客户端已收 ACK 但其实服务端炸了，用户永远等不到 AI 回复。

**Files:**
- Modify: `server/src/main/java/com/sanyan/websocket/WebSocketHandler.java`
- Modify: `server/src/main/java/com/sanyan/dto/ws/WsEventType.java`
- Create: `server/src/main/java/com/sanyan/dto/ws/WsError.java`
- Test: `server/src/test/java/com/sanyan/websocket/WebSocketHandlerErrorTest.java`

- [ ] **Step 1: 写失败测试**

Create `server/src/test/java/com/sanyan/websocket/WebSocketHandlerErrorTest.java`:

```java
package com.sanyan.websocket;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.sanyan.dto.ws.WsEventType;
import com.sanyan.dto.ws.WsMessage;
import com.sanyan.service.MessageService;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class WebSocketHandlerErrorTest {

    @Mock SessionManager sessionManager;
    @Mock MessageService messageService;
    @Mock WebSocketSession session;
    ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void handleSendMessage_onAsyncFailure_sendsErrorFrame() throws Exception {
        Map<String, Object> attrs = new HashMap<>();
        attrs.put("userId", 1L);
        when(session.getAttributes()).thenReturn(attrs);
        when(session.isOpen()).thenReturn(true);
        when(messageService.handleUserMessage(any(), any(), any(), any(), any(), any()))
                .thenThrow(new RuntimeException("AI 服务炸了"));

        WebSocketHandler handler = new WebSocketHandler(sessionManager, objectMapper, messageService);

        WsMessage incoming = new WsMessage();
        incoming.setType(WsEventType.SEND_MESSAGE);
        incoming.setConversationId(10L);
        incoming.setClientMsgId("cid-xyz");
        incoming.setContent("hi");
        incoming.setContentType("text");
        handler.handleTextMessage(session, new TextMessage(objectMapper.writeValueAsString(incoming)));

        ArgumentCaptor<TextMessage> captor = ArgumentCaptor.forClass(TextMessage.class);
        await().atMost(2, TimeUnit.SECONDS).untilAsserted(() -> {
            verify(session, atLeastOnce()).sendMessage(captor.capture());
            assertThat(captor.getAllValues().stream().anyMatch(
                    m -> m.getPayload().contains("\"type\":\"" + WsEventType.ERROR + "\"")
                      && m.getPayload().contains("cid-xyz"))).isTrue();
        });
    }
}
```

- [ ] **Step 2: 加 awaitility 依赖到 pom.xml（如已存在可跳过）**

```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <scope>test</scope>
</dependency>
```

Spring Boot parent 已管理版本。

- [ ] **Step 3: 跑测试确认红**

```bash
./mvnw test -Dtest=WebSocketHandlerErrorTest
```

Expected: FAIL（WsEventType.ERROR 未定义）

- [ ] **Step 4: 新增事件类型和 DTO**

`WsEventType.java` 加常量 `public static final String ERROR = "error";`

Create `server/src/main/java/com/sanyan/dto/ws/WsError.java`:

```java
package com.sanyan.dto.ws;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class WsError {
    private final String type = WsEventType.ERROR;
    private String clientMsgId;
    private Long conversationId;
    private String message;
}
```

- [ ] **Step 5: WebSocketHandler catch 分支发 error**

`WebSocketHandler.java` 的 `handleSendMessage` 的 catch 块改为：

```java
} catch (Exception e) {
    log.error("处理用户消息失败, userId={}", userId, e);
    WsError err = new WsError(wsMsg.getClientMsgId(), wsMsg.getConversationId(),
            "消息处理失败，请稍后重试");
    sendObject(session, err);
}
```

- [ ] **Step 6: 跑测试确认绿 + 全量测试**

```bash
./mvnw test -Dtest=WebSocketHandlerErrorTest
./mvnw test
```

- [ ] **Step 7: 提交**

```bash
git add -A
git commit -m "fix: WebSocket 异步处理失败时推送 error 事件给客户端

原来客户端已收 ACK，服务端 CompletableFuture 里抛异常只打 log，
用户永远等不到 AI 回复也不知道出了什么事。补一个 error 事件，
客户端可据此提示重试。"
```

---

## Task 7: 加 ASR/TTS 降级标记

**问题：** ASR 失败降级到 `chatVoiceAck`、TTS 失败降级到文本，用户体验突然变差且不知道原因。

**方案：** `Message` 实体加 `fallbackReason` 字段（`asr_failed` / `tts_failed` / null），DTO 透出，客户端自行决定是否展示。

**Files:**
- Modify: `server/src/main/java/com/sanyan/entity/Message.java`
- Modify: `server/src/main/java/com/sanyan/dto/data/MessageData.java`
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java`
- Test: `server/src/test/java/com/sanyan/service/MessageServiceFallbackReasonTest.java`

- [ ] **Step 1: 写失败测试**

Create `server/src/test/java/com/sanyan/service/MessageServiceFallbackReasonTest.java`:

```java
package com.sanyan.service;

import com.sanyan.dto.ws.MessageContentType;
import com.sanyan.entity.AiCharacter;
import com.sanyan.entity.Conversation;
import com.sanyan.entity.Message;
import com.sanyan.repository.AiCharacterRepository;
import com.sanyan.repository.ConversationRepository;
import com.sanyan.repository.MessageRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.redis.core.ListOperations;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class MessageServiceFallbackReasonTest {

    @Mock ConversationRepository conversationRepository;
    @Mock MessageRepository messageRepository;
    @Mock AiCharacterRepository characterRepository;
    @Mock AiService aiService;
    @Mock TtsService ttsService;
    @Mock CosService cosService;
    @Mock AsrService asrService;
    @Mock StringRedisTemplate redisTemplate;
    @Mock ValueOperations<String, String> valueOps;
    @Mock ListOperations<String, String> listOps;
    @InjectMocks MessageService service;

    @Test
    void asrFailed_marksMessageAsFallback() {
        setupCommonMocks();
        when(asrService.isEnabled()).thenReturn(true);
        when(asrService.transcribe(anyString())).thenReturn(null); // ASR 失败
        when(aiService.chatVoiceAck(any(), anyLong())).thenReturn("嗯嗯");
        when(ttsService.isEnabled()).thenReturn(false);

        service.handleUserMessage(1L, 10L, MessageContentType.VOICE, "", "http://v.mp3", 3);

        ArgumentCaptor<Message> cap = ArgumentCaptor.forClass(Message.class);
        verify(messageRepository, atLeast(2)).save(cap.capture());
        Message ai = cap.getAllValues().stream()
                .filter(m -> "ai".equals(m.getSenderType())).findFirst().orElseThrow();
        assertThat(ai.getFallbackReason()).isEqualTo("asr_failed");
    }

    @Test
    void ttsFailed_marksMessageAsFallback() {
        setupCommonMocks();
        when(ttsService.isEnabled()).thenReturn(true);
        when(ttsService.synthesize(anyString(), any())).thenReturn(null); // TTS 失败
        when(aiService.chat(any(), anyLong())).thenReturn("好的");

        service.handleUserMessage(1L, 10L, MessageContentType.TEXT, "hi", null, null);

        ArgumentCaptor<Message> cap = ArgumentCaptor.forClass(Message.class);
        verify(messageRepository, atLeast(2)).save(cap.capture());
        Message ai = cap.getAllValues().stream()
                .filter(m -> "ai".equals(m.getSenderType())).findFirst().orElseThrow();
        assertThat(ai.getFallbackReason()).isEqualTo("tts_failed");
    }

    private void setupCommonMocks() {
        Conversation conv = new Conversation();
        conv.setId(10L);
        conv.setUserId(1L);
        conv.setCharacterId(100L);
        conv.setUnreadCount(0);
        when(conversationRepository.findById(10L)).thenReturn(Optional.of(conv));
        when(characterRepository.findById(100L)).thenReturn(Optional.of(new AiCharacter()));
        when(redisTemplate.opsForValue()).thenReturn(valueOps);
        when(redisTemplate.opsForList()).thenReturn(listOps);
    }
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
./mvnw test -Dtest=MessageServiceFallbackReasonTest
```

Expected: FAIL（`getFallbackReason` 不存在）

- [ ] **Step 3: 给 Message 实体加字段**

`Message.java` 在 `ttsStyle` 之后加：

```java
// 降级原因：asr_failed（ASR 识别失败）、tts_failed（TTS 合成失败）、null（正常）。
// 客户端可据此展示"由于网络问题采用文字回复"等提示。
@Column(length = 32)
private String fallbackReason;
```

`MessageData.java` 加：

```java
private String fallbackReason;
```

`toData` 方法补：

```java
d.setFallbackReason(msg.getFallbackReason());
```

- [ ] **Step 4: 在降级分支设置 fallbackReason**

`MessageService.handleUserMessage` 里：

ASR 失败分支（原 78-81 行附近）：`aiReply = aiService.chatVoiceAck(...)` 后把 `aiReply` 的保存逻辑后面加上（在 TTS 分支外，即 TEXT 流程里）：

在 voice TTS 失败或禁用走 TEXT 流程时，如果本次是 ASR 失败降级，设 `aiMsg.setFallbackReason("asr_failed")`；如果是 TTS 失败降级（原来有 TTS 但 `audioData == null`），设 `aiMsg.setFallbackReason("tts_failed")`。

具体实现：方法开头声明 `String fallbackReason = null;`，在 ASR 失败分支设 `fallbackReason = "asr_failed";`，在 TTS `audioData == null` 的 `log.warn` 之后设 `fallbackReason = "tts_failed";`，最后创建 text 分支 `aiMsg` 时调用 `aiMsg.setFallbackReason(fallbackReason);`。

完整 handleUserMessage 关键片段：

```java
@Transactional
public Message handleUserMessage(Long userId, Long conversationId, String contentType, String content,
                                  String mediaUrl, Integer duration) {
    Conversation conv = conversationRepository.findById(conversationId)
            .orElseThrow(() -> new IllegalArgumentException("会话不存在"));
    if (!conv.getUserId().equals(userId)) {
        throw new SecurityException("无权访问该会话");
    }

    // ... 保存 userMsg ...

    AiCharacter character = characterRepository.findById(conv.getCharacterId())
            .orElseThrow(() -> new IllegalArgumentException("角色不存在"));

    String aiReply;
    String fallbackReason = null;
    if (MessageContentType.VOICE.equals(contentType)) {
        String transcribedText = asrService.isEnabled()
                ? asrService.transcribe(mediaUrl) : null;
        if (transcribedText != null && !transcribedText.isBlank()) {
            userMsg.setContent(transcribedText);
            messageRepository.save(userMsg);
            aiReply = aiService.chat(character, conversationId);
        } else {
            log.info("ASR 失败或静音，降级为 chatVoiceAck: convId={}", conversationId);
            aiReply = aiService.chatVoiceAck(character, conversationId);
            fallbackReason = "asr_failed";
        }
    } else {
        aiReply = aiService.chat(character, conversationId);
    }

    if (ttsService.isEnabled()) {
        var extracted = TextProcessor.extract(aiReply);
        byte[] audioData = ttsService.synthesize(extracted.cleanText(), extracted.ttsStyle());
        if (audioData != null) {
            Message aiMsg = new Message();
            aiMsg.setConversationId(conversationId);
            aiMsg.setSenderType(SenderType.AI);
            aiMsg.setContentType(MessageContentType.VOICE);
            aiMsg.setContent(extracted.cleanText());
            aiMsg.setTtsStyle(extracted.ttsStyle());
            aiMsg.setSource("reply");
            aiMsg.setDuration(estimateVoiceDuration(extracted.cleanText()));
            aiMsg.setFallbackReason(fallbackReason);
            messageRepository.save(aiMsg);
            // ... COS 上传、update conversation、Redis tracking（同原逻辑）...
            return aiMsg;
        }
        log.warn("TTS 合成失败，降级为文字消息: convId={}", conversationId);
        fallbackReason = "tts_failed";
    }

    // Text path
    var extracted = TextProcessor.extract(aiReply);
    Message aiMsg = new Message();
    aiMsg.setConversationId(conversationId);
    aiMsg.setSenderType(SenderType.AI);
    aiMsg.setContentType(MessageContentType.TEXT);
    aiMsg.setContent(extracted.cleanText());
    aiMsg.setSource("reply");
    aiMsg.setFallbackReason(fallbackReason);
    messageRepository.save(aiMsg);
    // ... update conversation、Redis tracking ...
    return aiMsg;
}
```

（原本 Voice 成功路径没有降级，所以 `fallbackReason == null`；ASR 失败且 TTS 启用时，aiMsg 为 VOICE 但 fallbackReason=asr_failed——客户端收到语音但知道是降级的 voice ack，行为合理。）

- [ ] **Step 5: 跑测试确认绿 + 全量测试**

```bash
./mvnw test -Dtest=MessageServiceFallbackReasonTest
./mvnw test
```

- [ ] **Step 6: 提交**

```bash
git add -A
git commit -m "feat: AI 消息带降级原因字段

新增 Message.fallbackReason（asr_failed / tts_failed），在 DTO 透出。
客户端收到带标记的消息可展示\"由于网络问题降级\"等提示，
避免用户以为功能坏了。"
```

---

## Task 8: 加索引

**问题：** `Message.createdAt`、`Conversation.lastMessageAt` 无索引。

**Files:**
- Modify: `server/src/main/java/com/sanyan/entity/Message.java`
- Modify: `server/src/main/java/com/sanyan/entity/Conversation.java`
- Test: `server/src/test/java/com/sanyan/entity/EntityIndexTest.java`（新建）

- [ ] **Step 1: 写失败测试**

Create `server/src/test/java/com/sanyan/entity/EntityIndexTest.java`:

```java
package com.sanyan.entity;

import jakarta.persistence.Index;
import jakarta.persistence.Table;
import org.junit.jupiter.api.Test;

import java.util.Arrays;

import static org.assertj.core.api.Assertions.assertThat;

class EntityIndexTest {

    @Test
    void message_hasCreatedAtIndex() {
        Table table = Message.class.getAnnotation(Table.class);
        assertThat(Arrays.stream(table.indexes()).map(Index::columnList))
                .anyMatch(c -> c.contains("createdAt"));
    }

    @Test
    void conversation_hasLastMessageAtIndex() {
        Table table = Conversation.class.getAnnotation(Table.class);
        assertThat(Arrays.stream(table.indexes()).map(Index::columnList))
                .anyMatch(c -> c.contains("lastMessageAt") || c.contains("last_message_at"));
    }
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
./mvnw test -Dtest=EntityIndexTest
```

- [ ] **Step 3: 加索引**

`Message.java` 的 `@Table` 改为：

```java
@Table(name = "message", indexes = {
    @Index(name = "idx_message_conversation", columnList = "conversationId,id"),
    @Index(name = "idx_message_created_at", columnList = "createdAt")
})
```

`Conversation.java` 的 `@Table` 改为：

```java
@Table(name = "conversation",
    uniqueConstraints = {
        @UniqueConstraint(columnNames = {"user_id", "character_id"})
    },
    indexes = {
        @Index(name = "idx_conversation_character_id", columnList = "character_id"),
        @Index(name = "idx_conversation_user_last_msg", columnList = "user_id,last_message_at")
    }
)
```

- [ ] **Step 4: 跑测试确认绿 + 全量测试 + 提交**

```bash
./mvnw test -Dtest=EntityIndexTest
./mvnw test
git add -A
git commit -m "perf: 给 Message.createdAt、Conversation 的用户+时间组合加索引

消息列表按时间排序、历史消息时间范围查询都需要 createdAt 索引；
getUserConversations 按 userId+lastMessageAt DESC 查询用组合索引
比 character 索引更匹配主查询模式。"
```

---

## Task 9: 运维配置收紧 + SessionManager 单机前提文档化

**问题：**
1. `application-dev.yml` 的 `ddl-auto: update` 和 `show-sql: true` 如果被误用到生产会出事。
2. `SessionManager` 内存 Map + Redis 的组合只在单实例部署下可靠，代码里没说明。

**Files:**
- Modify: `server/src/main/resources/application.yml`（新增 prod 默认）
- Modify: `server/src/main/java/com/sanyan/websocket/SessionManager.java`

这一组是文档/配置性改动，TDD 不适用（属于 CLAUDE.md 的 Configuration files 例外），直接改 + 人工核验。

- [ ] **Step 1: application.yml 根配置加默认生产态**

`application.yml` 末尾追加（最终版）：

```yaml
spring:
  profiles:
    active: dev
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    default-property-inclusion: non_null
  jpa:
    # 默认只校验结构，生产不允许 ddl-auto=update
    hibernate:
      ddl-auto: validate
    show-sql: false
```

`application-dev.yml` 保留 `ddl-auto: update` 和 `show-sql: true`（因为 dev profile 会 override 默认）。确认 dev profile 行为未变。

- [ ] **Step 2: SessionManager 加文档**

`SessionManager.java` 类级注释改为：

```java
/**
 * WebSocket 在线用户管理。
 *
 * <p><b>⚠️ 单实例部署前提：</b>真正的 {@link WebSocketSession} 只存在
 * 本进程的 {@link #sessions} 内存 Map。多实例部署时，{@link #isOnline}
 * 虽然依赖 Redis 跨实例可见，但 {@link #getSession} 只能在当前进程命中，
 * 无法把消息推给别的实例持有的连接（会造成主动消息丢失）。
 *
 * <p>TODO（多实例化改造）：
 * <ul>
 *   <li>方案 A：Redis Pub/Sub — 每个实例订阅 {@code ws:push:{userId}} 频道，
 *       需要下行消息时对应实例把自己持有的 session 写入。</li>
 *   <li>方案 B：在 Redis 记录 userId → instanceId 映射，通过 HTTP/RPC 路由到
 *       正确实例。</li>
 * </ul>
 */
@Component
@RequiredArgsConstructor
public class SessionManager {
    // ... 原代码不变 ...
}
```

- [ ] **Step 3: 全量测试 + 提交**

```bash
./mvnw test
git add -A
git commit -m "chore: 运维加固 + SessionManager 单机前提文档化

- application.yml 默认 ddl-auto: validate、show-sql: false，
  避免 dev profile 失效时生产暴露。
- SessionManager 加 Javadoc 说明单机部署前提和多实例改造方案。"
```

---

## Task 10: 合并到 dev

- [ ] **Step 1: 把 fix/review-findings 合回 dev**

```bash
cd /Users/aventador/code/3yan/server
git checkout dev
git merge --no-ff fix/review-findings -m "merge: server code review 修复"
git push origin dev
```

- [ ] **Step 2: 主工程更新子模块引用**

```bash
cd /Users/aventador/code/3yan
git add server
git commit -m "chore: 更新 server 子模块引用（code review 修复）"
# 主工程分支合并在 App plan Task 10 之后一起做
```

---

## Self-Review 已完成

- [x] Spec 覆盖：P1 权限（Task 2）、事务（Task 3）、senderType（Task 4）、N+1（Task 5）、WebSocket 吞异常（Task 6）、降级标记（Task 7）、索引（Task 8）、生产配置+SessionManager 文档（Task 9）
- [x] 无 placeholder，所有代码和测试都写全
- [x] 类型一致：`fallbackReason` 字段命名跨 entity/DTO/test 一致；`SenderType.AI` 使用一致
- [x] 排除的：密钥轮换（用户保留）、SessionManager Redis Pub/Sub（大改留给后续）、SMS SDK（无凭证）
