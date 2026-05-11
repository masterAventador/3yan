# 主动消息空内容 bug 修复 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 AI 主动消息发空内容（msg id=92, 2026-04-16 04:53:50）的根因 + 防止凌晨触发 + 防御性兜底。

**Architecture:**
- **根因（Task 1）**：`chatProactive` 把"触发指令"塞在 system prompt 里、然后把最近 20 条消息历史也喂给豆包，导致 messages 数组以 assistant 结尾。模型懵了，只输出强制要求的 `[emotion:xxx:n]` 标签，被 `TextProcessor` 剥光后内容为空。
- **重构方向**：`chatProactive` **不传消息历史**，只用人设 + 用户画像 + 长期记忆摘要 + 时间，把触发指令作为唯一的 user turn 明确告知"这是系统指令、不是用户的话"。
- **配套修复**：(Task 2) 配置键对齐 + 加 `active_hours` 时段过滤；(Task 3) `ProactiveService` 在内容清洗后再判一次空。

**Tech Stack:** Spring Boot 3.2.5 + Java 17 + JUnit 5 + Mockito + AssertJ + Lombok。

---

## File Structure

| 文件 | 类型 | 责任 |
|---|---|---|
| `server/src/main/java/com/sanyan/service/AiService.java` | 修改 | `chatProactive` 重构：不传历史，加 synthetic user turn；新增 `buildProactiveMessages` 纯方法便于测试 |
| `server/src/test/java/com/sanyan/service/AiServiceTest.java` | 修改 | 新增 `buildProactiveMessages` 的测试 |
| `server/src/main/java/com/sanyan/scheduler/ProactiveScheduler.java` | 修改 | 读嵌套 JSON 键、加 `active_hours` 时段过滤、抽 `isWithinActiveHours` 纯方法 |
| `server/src/test/java/com/sanyan/scheduler/ProactiveSchedulerTest.java` | 创建 | 测嵌套键解析、`active_hours` 过滤逻辑 |
| `server/src/main/resources/data.sql` | 修改 | `situational.daily_count` → `situational.trigger_rate`（语义对齐 scheduler 实际能用的字段） |
| `server/src/main/java/com/sanyan/service/ProactiveService.java` | 修改 | 抽 `isSendableContent` 纯方法；在 `cleanContent.isBlank()` 时跳过保存 + warn |
| `server/src/test/java/com/sanyan/service/ProactiveServiceTest.java` | 修改 | 新增 `isSendableContent` 的测试 |

---

## Task 1: AiService.chatProactive 重构（去历史 + synthetic user turn）

**Files:**
- Modify: `server/src/main/java/com/sanyan/service/AiService.java:85-98`（`chatProactive` 方法）
- Modify: `server/src/main/java/com/sanyan/service/AiService.java:116-167`（`callDoubao` 方法）— 拆成 `buildChatMessages` + `callDoubaoRaw`
- Test: `server/src/test/java/com/sanyan/service/AiServiceTest.java`

### 设计

新增一个**包私有静态方法**便于纯单元测试：

```java
static List<Map<String,String>> buildProactiveMessages(
    String characterPrompt,
    String time,
    String profile,                    // 用户画像，可能为 null
    List<MemorySummary> summaries,    // 长期记忆，可能为空
    String triggerHint                 // 触发场景描述
)
```

返回的 messages 数组结构：
1. `{role: system, content: characterPrompt + "\n\n[当前时间] " + time + 画像 + 长期记忆}`
2. `{role: user, content: 包装后的 triggerHint}`（明确告知"这是系统指令、不是用户的话"）

**不**包含任何 `recentMessages`。

`chatProactive` 改成：拿数据 → 调 `buildProactiveMessages` → 调 `callDoubaoRaw(messages)`。

`callDoubao` 拆成两个方法：
- `buildChatMessages(systemPrompt, summaries, recentMessages)` — 旧的内部组装逻辑（继续给 `chat` / `chatVoiceAck` 用）
- `callDoubaoRaw(List<Map<String,String>> messages)` — 真正的 HTTP 调用部分

### Steps

- [ ] **Step 1.1: 写失败测试 — buildProactiveMessages 不包含历史消息**

文件 `server/src/test/java/com/sanyan/service/AiServiceTest.java`，在 class 内追加：

```java
@Test
void buildProactiveMessages_shouldContainSystemAndUserTurnOnly_noHistory() {
    String characterPrompt = "你是小婉，一个温柔的女生";
    String time = "2026年4月16日 周四 04:53";
    String profile = "用户叫小明，是程序员，最近在装环境";
    List<MemorySummary> summaries = List.of();
    String triggerHint = "用户已经有一段时间没有说话了，主动关心一下";

    List<Map<String, String>> messages = AiService.buildProactiveMessages(
        characterPrompt, time, profile, summaries, triggerHint);

    assertThat(messages).hasSize(2);
    assertThat(messages.get(0).get("role")).isEqualTo("system");
    assertThat(messages.get(0).get("content")).contains("你是小婉");
    assertThat(messages.get(0).get("content")).contains("2026年4月16日");
    assertThat(messages.get(0).get("content")).contains("小明");
    assertThat(messages.get(1).get("role")).isEqualTo("user");
    assertThat(messages.get(1).get("content")).contains("系统指令");
    assertThat(messages.get(1).get("content")).contains("用户已经有一段时间没有说话了");
}
```

需要的 import 在文件顶部追加：
```java
import com.sanyan.entity.MemorySummary;
import java.util.List;
import java.util.Map;
```

- [ ] **Step 1.2: 跑测试确认失败**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=AiServiceTest#buildProactiveMessages_shouldContainSystemAndUserTurnOnly_noHistory -q
```

期望：编译失败，"cannot find symbol: method buildProactiveMessages"。

- [ ] **Step 1.3: 写最小实现让测试通过**

在 `AiService.java` 类内任意位置（建议放在 `chatProactive` 方法上方）追加：

```java
/**
 * Build messages array for proactive trigger.
 * NO conversation history is passed — proactive should be a fresh initiation,
 * not a continuation of past dialogue.
 */
static List<Map<String, String>> buildProactiveMessages(
        String characterPrompt,
        String time,
        String profile,
        List<MemorySummary> summaries,
        String triggerHint) {

    StringBuilder systemContent = new StringBuilder(characterPrompt);
    systemContent.append("\n\n[当前时间] ").append(time);
    if (profile != null && !profile.isBlank()) {
        systemContent.append("\n\n[你对用户的印象] ").append(profile);
    }
    if (summaries != null && !summaries.isEmpty()) {
        systemContent.append("\n\n[你们以前聊过的事]");
        for (MemorySummary s : summaries) {
            systemContent.append("\n- ").append(s.getSummary());
        }
    }

    String userContent = "【这不是用户的话，是给你的系统指令】\n"
            + triggerHint + "\n"
            + "注意：\n"
            + "- 这是你主动发起的消息，不要写成在回复用户上一句\n"
            + "- 不要直接复述以前聊过的事，那是你的记忆，不是要拿来重复\n"
            + "- 考虑当前时间，措辞要符合场景\n"
            + "- 直接输出消息内容，不要带任何解释或前缀";

    return List.of(
            Map.of("role", "system", "content", systemContent.toString()),
            Map.of("role", "user", "content", userContent)
    );
}
```

- [ ] **Step 1.4: 跑测试确认通过**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=AiServiceTest#buildProactiveMessages_shouldContainSystemAndUserTurnOnly_noHistory -q
```

期望：BUILD SUCCESS，1 test passed。

- [ ] **Step 1.5: 写失败测试 — chatProactive 走新流程（不再调用 messageRepository 取历史）**

在 `AiServiceTest.java` 文件内追加：

```java
@Test
void chatProactive_shouldNotLoadRecentMessages() throws Exception {
    // 用 Mockito mock 依赖
    var messageRepo = mock(com.sanyan.repository.MessageRepository.class);
    var memoryProfileRepo = mock(com.sanyan.repository.MemoryProfileRepository.class);
    var memorySummaryRepo = mock(com.sanyan.repository.MemorySummaryRepository.class);
    var objectMapper = new com.fasterxml.jackson.databind.ObjectMapper();
    var restTemplate = mock(org.springframework.web.client.RestTemplate.class);

    AiService aiService = new AiService(messageRepo, memoryProfileRepo, memorySummaryRepo, objectMapper, restTemplate);

    when(memoryProfileRepo.findByConversationId(1L)).thenReturn(java.util.Optional.empty());
    when(memorySummaryRepo.findByConversationIdOrderByCreatedAtDesc(eq(1L), any())).thenReturn(List.of());

    // mock RestTemplate to return a canned doubao response
    String fakeResp = "{\"choices\":[{\"message\":{\"content\":\"嗨，最近怎么样？[emotion:happy:2]\"}}]}";
    when(restTemplate.exchange(anyString(), any(), any(), eq(String.class)))
            .thenReturn(new org.springframework.http.ResponseEntity<>(fakeResp, org.springframework.http.HttpStatus.OK));

    com.sanyan.entity.AiCharacter character = new com.sanyan.entity.AiCharacter();
    character.setSystemPrompt("你是小婉");
    aiService.chatProactive(character, 1L, "主动打招呼");

    // 关键断言：messageRepository.findByConversationIdOrderByIdDesc 不应被调用
    verify(messageRepo, never()).findByConversationIdOrderByIdDesc(anyLong(), any());
}
```

需要的 import 追加：
```java
import static org.mockito.Mockito.*;
import static org.mockito.ArgumentMatchers.*;
```

- [ ] **Step 1.6: 跑测试确认失败**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=AiServiceTest#chatProactive_shouldNotLoadRecentMessages -q
```

期望：测试失败 — `messageRepo.findByConversationIdOrderByIdDesc` 被调用了至少 1 次（旧实现里的 line 93-94 还在）。

- [ ] **Step 1.7: 重构 chatProactive 使用新流程**

修改 `AiService.java` 的 `chatProactive` 方法（替换原 85-98 行）：

```java
public String chatProactive(AiCharacter character, Long conversationId, String triggerHint) {
    String time = formatCurrentTime();
    String profile = getProfile(conversationId);

    List<MemorySummary> summaries = memorySummaryRepository
            .findByConversationIdOrderByCreatedAtDesc(conversationId, PageRequest.of(0, 5));

    List<Map<String, String>> messages = buildProactiveMessages(
            character.getSystemPrompt(), time, profile, summaries, triggerHint);

    return callDoubaoRaw(messages);
}
```

然后把原 `callDoubao` 方法（116-167 行）拆成两半。在 `callDoubao` 上方插入新方法 `callDoubaoRaw`：

```java
public String callDoubaoRaw(List<Map<String, String>> chatMessages) {
    try {
        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("model", model);
        requestBody.put("messages", chatMessages);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(apiKey);

        String body = objectMapper.writeValueAsString(requestBody);
        HttpEntity<String> entity = new HttpEntity<>(body, headers);

        log.info("豆包 API 请求: model={}, messagesCount={}", model, chatMessages.size());
        long start = System.currentTimeMillis();
        ResponseEntity<String> response = restTemplate.exchange(endpoint, HttpMethod.POST, entity, String.class);
        long elapsed = System.currentTimeMillis() - start;
        log.info("豆包 API 响应: status={}, 耗时={}ms", response.getStatusCode(), elapsed);

        @SuppressWarnings("unchecked")
        Map<String, Object> responseMap = objectMapper.readValue(response.getBody(), Map.class);
        @SuppressWarnings("unchecked")
        List<Map<String, Object>> choices = (List<Map<String, Object>>) responseMap.get("choices");
        @SuppressWarnings("unchecked")
        Map<String, String> message = (Map<String, String>) choices.get(0).get("message");
        return message.get("content");

    } catch (Exception e) {
        log.error("豆包 API 调用失败", e);
        return "抱歉，我现在有点走神了，等下再聊吧~";
    }
}
```

并把原 `callDoubao` 方法体改成委托：

```java
public String callDoubao(String systemPrompt, List<MemorySummary> summaries, List<Message> messages) {
    List<Map<String, String>> chatMessages = new ArrayList<>();
    chatMessages.add(Map.of("role", "system", "content", systemPrompt));

    if (summaries != null && !summaries.isEmpty()) {
        StringBuilder summaryText = new StringBuilder("[历史记忆摘要]\n");
        for (MemorySummary s : summaries) {
            summaryText.append("- ").append(s.getSummary()).append("\n");
        }
        chatMessages.add(Map.of("role", "system", "content", summaryText.toString()));
    }

    if (messages != null) {
        for (Message msg : messages) {
            String role = SenderType.USER.equals(msg.getSenderType()) ? "user" : "assistant";
            chatMessages.add(Map.of("role", role, "content", msg.getContent()));
        }
    }

    return callDoubaoRaw(chatMessages);
}
```

- [ ] **Step 1.8: 跑全部 AiServiceTest 确认都通过**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=AiServiceTest -q
```

期望：所有测试通过（包括原有 `shouldAssemblePromptWithTimeAndProfile`）。

- [ ] **Step 1.9: 跑全量后端测试确认无回归**

```bash
cd /Users/aventador/code/3yan/server
mvn test -q
```

期望：BUILD SUCCESS，所有测试通过。

- [ ] **Step 1.10: Commit**

```bash
cd /Users/aventador/code/3yan/server
git add src/main/java/com/sanyan/service/AiService.java src/test/java/com/sanyan/service/AiServiceTest.java
git commit -m "fix: 主动消息提示词重构，去掉对话历史 + 触发指令作为 user turn"
```

---

## Task 2: ProactiveScheduler 配置键对齐 + active_hours 过滤

**Files:**
- Modify: `server/src/main/java/com/sanyan/scheduler/ProactiveScheduler.java`
- Create: `server/src/test/java/com/sanyan/scheduler/ProactiveSchedulerTest.java`
- Modify: `server/src/main/resources/data.sql`

### 设计

`data.sql` 现在用嵌套 JSON：
```json
{
  "max_daily": 3,
  "min_interval_hours": 2,
  "active_hours": [8, 22],
  "greeting": {"enabled": true, "slots": ["08:00-09:00", "12:00-13:00", "21:00-22:00"]},
  "event_trigger": {"enabled": true, "idle_hours_threshold": 6},
  "situational": {"enabled": true, "daily_count": [1, 2]}
}
```

但代码读的是扁平键（`greeting_slots`、`idle_trigger_hours`、`situational_trigger_rate`），全部读不到。

**改 scheduler**：
1. 改用 `cfg.path("greeting").path("slots")` 等嵌套读取
2. 增加 `cfg.path("X").path("enabled")` 开关检查（false → 跳过该类型）
3. 增加 `active_hours` 时段过滤：抽 `boolean isWithinActiveHours(LocalTime now, JsonNode cfg)` 纯方法，所有 3 个 check 都先调一遍
4. `situational` 改读 `situational.trigger_rate`（与 `data.sql` 同步改成 `trigger_rate: 0.2`，统一用概率语义）

### Steps

- [ ] **Step 2.1: 写失败测试 — isWithinActiveHours 在范围内返回 true、外返回 false**

新建文件 `server/src/test/java/com/sanyan/scheduler/ProactiveSchedulerTest.java`：

```java
package com.sanyan.scheduler;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;

import java.time.LocalTime;

import static org.assertj.core.api.Assertions.assertThat;

class ProactiveSchedulerTest {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void isWithinActiveHours_shouldReturnTrue_whenCurrentHourInRange() throws Exception {
        JsonNode cfg = objectMapper.readTree("{\"active_hours\":[8,22]}");
        assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(10, 0), cfg)).isTrue();
        assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(8, 0), cfg)).isTrue();
        assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(22, 0), cfg)).isTrue();
    }

    @Test
    void isWithinActiveHours_shouldReturnFalse_whenCurrentHourOutsideRange() throws Exception {
        JsonNode cfg = objectMapper.readTree("{\"active_hours\":[8,22]}");
        assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(4, 53), cfg)).isFalse();
        assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(7, 59), cfg)).isFalse();
        assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(23, 0), cfg)).isFalse();
    }

    @Test
    void isWithinActiveHours_shouldReturnTrue_whenConfigMissing() throws Exception {
        // 没配 active_hours 时默认放行（不主动限制）
        JsonNode cfg = objectMapper.readTree("{}");
        assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(4, 0), cfg)).isTrue();
    }
}
```

- [ ] **Step 2.2: 跑测试确认失败**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=ProactiveSchedulerTest -q
```

期望：编译失败 — `cannot find symbol: method isWithinActiveHours`。

- [ ] **Step 2.3: 实现 isWithinActiveHours**

在 `ProactiveScheduler.java` 类内（建议放在 `isInTimeSlot` 方法附近）追加：

```java
/**
 * Check if the current time is within the active hours range configured.
 * If active_hours not configured, returns true (no time restriction).
 *
 * Config example: {"active_hours": [8, 22]} means hours 8:00-22:59 inclusive.
 */
static boolean isWithinActiveHours(LocalTime now, JsonNode cfg) {
    if (cfg == null || !cfg.has("active_hours")) return true;
    JsonNode range = cfg.get("active_hours");
    if (!range.isArray() || range.size() != 2) return true;
    int startHour = range.get(0).asInt();
    int endHour = range.get(1).asInt();
    int curHour = now.getHour();
    return curHour >= startHour && curHour <= endHour;
}
```

- [ ] **Step 2.4: 跑测试确认通过**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=ProactiveSchedulerTest -q
```

期望：3 tests passed。

- [ ] **Step 2.5: 在 3 个 @Scheduled 方法的入口加上 active_hours 过滤 + 改读嵌套键**

修改 `ProactiveScheduler.java`：

**`greetingCheck()` 方法**（替换原 41-70 行的方法体）：

```java
@Scheduled(fixedRate = 600_000)
public void greetingCheck() {
    LocalTime now = LocalTime.now();
    List<AiCharacter> characters = characterRepository.findAll();

    for (AiCharacter character : characters) {
        JsonNode cfg = parseConfig(character);
        if (cfg == null) continue;
        if (!isWithinActiveHours(now, cfg)) continue;

        JsonNode greetingNode = cfg.path("greeting");
        if (!greetingNode.path("enabled").asBoolean(true)) continue;
        JsonNode slots = greetingNode.path("slots");
        if (!slots.isArray() || slots.isEmpty()) continue;

        boolean inSlot = false;
        for (JsonNode slotNode : slots) {
            if (isInTimeSlot(now, slotNode.asText())) {
                inSlot = true;
                break;
            }
        }
        if (!inSlot) continue;

        List<Conversation> conversations = conversationRepository.findAll();
        for (Conversation conv : conversations) {
            if (!conv.getCharacterId().equals(character.getId())) continue;
            try {
                proactiveService.sendProactiveMessage(conv, character,
                        "现在是打招呼的好时机，用温暖自然的方式主动问候用户，关心他/她的近况");
            } catch (Exception e) {
                log.error("问候触发失败, convId={}", conv.getId(), e);
            }
        }
    }
}
```

**`eventTriggerCheck()` 方法**（替换原 76-103 行的方法体）：

```java
@Scheduled(fixedRate = 1_800_000)
public void eventTriggerCheck() {
    LocalTime now = LocalTime.now();
    List<AiCharacter> characters = characterRepository.findAll();

    for (AiCharacter character : characters) {
        JsonNode cfg = parseConfig(character);
        if (cfg == null) continue;
        if (!isWithinActiveHours(now, cfg)) continue;

        JsonNode eventNode = cfg.path("event_trigger");
        if (!eventNode.path("enabled").asBoolean(true)) continue;
        int idleHours = eventNode.path("idle_hours_threshold").asInt(8);

        LocalDateTime threshold = LocalDateTime.now().minusHours(idleHours);
        List<Conversation> conversations = conversationRepository.findAll();

        for (Conversation conv : conversations) {
            if (!conv.getCharacterId().equals(character.getId())) continue;
            if (conv.getLastMessageAt() == null) continue;
            if (conv.getLastMessageAt().isAfter(threshold)) continue;

            try {
                proactiveService.sendProactiveMessage(conv, character,
                        "用户已经有一段时间没有说话了，主动关心一下他/她，语气要自然温暖，不要显得刻意");
            } catch (Exception e) {
                log.error("空闲触发失败, convId={}", conv.getId(), e);
            }
        }
    }
}
```

**`situationalCheck()` 方法**（替换原 109-134 行的方法体）：

```java
@Scheduled(fixedRate = 3_600_000)
public void situationalCheck() {
    LocalTime now = LocalTime.now();
    List<AiCharacter> characters = characterRepository.findAll();

    for (AiCharacter character : characters) {
        JsonNode cfg = parseConfig(character);
        if (cfg == null) continue;
        if (!isWithinActiveHours(now, cfg)) continue;

        JsonNode situNode = cfg.path("situational");
        if (!situNode.path("enabled").asBoolean(true)) continue;
        double triggerRate = situNode.path("trigger_rate").asDouble(0.2);

        if (ThreadLocalRandom.current().nextDouble() > triggerRate) continue;

        List<Conversation> conversations = conversationRepository.findAll();
        for (Conversation conv : conversations) {
            if (!conv.getCharacterId().equals(character.getId())) continue;
            try {
                proactiveService.sendProactiveMessage(conv, character,
                        "随机分享一个生活感悟、有趣的事情或者问用户一个轻松的问题，让对话更有温度");
            } catch (Exception e) {
                log.error("情景触发失败, convId={}", conv.getId(), e);
            }
        }
    }
}
```

- [ ] **Step 2.6: 同步更新 data.sql 的 situational 字段**

修改 `server/src/main/resources/data.sql` 第 6 行（`proactive_config` JSON 的 situational 部分）：

把：
```
"situational":{"enabled":true,"daily_count":[1,2]}
```
改为：
```
"situational":{"enabled":true,"trigger_rate":0.2}
```

完整改动后的 proactive_config 应该是：
```json
{"max_daily":3,"min_interval_hours":2,"active_hours":[8,22],"greeting":{"enabled":true,"slots":["08:00-09:00","12:00-13:00","21:00-22:00"]},"event_trigger":{"enabled":true,"idle_hours_threshold":6},"situational":{"enabled":true,"trigger_rate":0.2}}
```

- [ ] **Step 2.7: 写失败测试 — eventTriggerCheck 在凌晨 4 点不应触发**

在 `ProactiveSchedulerTest.java` 内追加（需要把单元测试改成能 mock 当前时间的形式，但因为 `LocalTime.now()` 静态难 mock，这里走"行为级断言"策略，验证 active_hours 配置存在时 helper 返回 false 就够了，已经在 Step 2.1 覆盖；这里增加一个集成校验：解析 `data.sql` 的实际 JSON，断言它包含 `active_hours` 且 `isWithinActiveHours(LocalTime.of(4, 53), ...)` 返回 false）：

```java
@Test
void dataSqlConfig_shouldExcludeEarlyMorning() throws Exception {
    String configJson = "{\"max_daily\":3,\"min_interval_hours\":2,\"active_hours\":[8,22],"
            + "\"greeting\":{\"enabled\":true,\"slots\":[\"08:00-09:00\",\"12:00-13:00\",\"21:00-22:00\"]},"
            + "\"event_trigger\":{\"enabled\":true,\"idle_hours_threshold\":6},"
            + "\"situational\":{\"enabled\":true,\"trigger_rate\":0.2}}";
    JsonNode cfg = objectMapper.readTree(configJson);

    // 凌晨 4:53 应该被 active_hours 挡住
    assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(4, 53), cfg)).isFalse();
    // 早上 8 点应该放行
    assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(8, 0), cfg)).isTrue();
    // 22 点应该放行
    assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(22, 0), cfg)).isTrue();
    // 23 点应该被挡住
    assertThat(ProactiveScheduler.isWithinActiveHours(LocalTime.of(23, 0), cfg)).isFalse();
}
```

- [ ] **Step 2.8: 跑测试确认通过**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=ProactiveSchedulerTest -q
```

期望：4 tests passed。

- [ ] **Step 2.9: 跑全量后端测试确认无回归**

```bash
cd /Users/aventador/code/3yan/server
mvn test -q
```

期望：BUILD SUCCESS。

- [ ] **Step 2.10: Commit**

```bash
cd /Users/aventador/code/3yan/server
git add src/main/java/com/sanyan/scheduler/ProactiveScheduler.java \
        src/test/java/com/sanyan/scheduler/ProactiveSchedulerTest.java \
        src/main/resources/data.sql
git commit -m "fix: 主动消息调度器读嵌套配置键 + 加 active_hours 时段过滤"
```

---

## Task 3: ProactiveService 兜底（清洗后空内容跳过 + 隔离 AI 兜底文案）

**Files:**
- Modify: `server/src/main/java/com/sanyan/service/AiService.java`（抽兜底文案为常量，让 chatProactive 在拿到兜底时返回 null）
- Modify: `server/src/main/java/com/sanyan/service/ProactiveService.java`（在 `sendProactiveMessage` 内 `cleanContent` 计算后增加 blank 检查）
- Modify: `server/src/test/java/com/sanyan/service/AiServiceTest.java`
- Modify: `server/src/test/java/com/sanyan/service/ProactiveServiceTest.java`

### 设计

**两层兜底，分别针对两类失败场景：**

**(A) AI 接口挂了 → chatProactive 返回 null（让 ProactiveService 现有 `aiReply == null` 分支接住）**

`AiService.callDoubaoRaw` 的 catch 里返回的是固定字符串"抱歉，我现在有点走神了，等下再聊吧~"。这串文案在 `chat`/`chatVoiceAck` 链路里是合理兜底（用户主动问话，AI 回一句"走神了"说得通），但在 `chatProactive` 链路里就成了"AI 凭空跳出来说我走神了" — 用户体验灾难。

修法：
1. 把兜底文案抽成 `AiService` 的 `static final String AI_FALLBACK_MESSAGE` 常量
2. `callDoubaoRaw` 的 catch 用这个常量
3. `chatProactive` 比对返回值是否等于这个常量，若是则返回 null（ProactiveService 既有的 null 检查会兜住）

**(B) 模型返回内容清洗后为空 → ProactiveService 跳过保存**

抽一个**包私有静态方法**：
```java
static boolean isSendableContent(String cleanContent)
```
返回 `cleanContent != null && !cleanContent.isBlank()`。

在 `sendProactiveMessage` 内 `String cleanContent = extracted.cleanText();` 之后立即调用：若返回 false → log warn + return（不落库、不投递）。

### Steps

#### Part A: 隔离 AI 兜底文案

- [ ] **Step 3.A1: 写失败测试 — chatProactive 在 callDoubao 兜底时返回 null**

在 `AiServiceTest.java` 类内追加：

```java
@Test
void chatProactive_shouldReturnNull_whenDoubaoFallsBack() throws Exception {
    var messageRepo = mock(com.sanyan.repository.MessageRepository.class);
    var memoryProfileRepo = mock(com.sanyan.repository.MemoryProfileRepository.class);
    var memorySummaryRepo = mock(com.sanyan.repository.MemorySummaryRepository.class);
    var objectMapper = new com.fasterxml.jackson.databind.ObjectMapper();
    var restTemplate = mock(org.springframework.web.client.RestTemplate.class);

    AiService aiService = new AiService(messageRepo, memoryProfileRepo, memorySummaryRepo, objectMapper, restTemplate);

    when(memoryProfileRepo.findByConversationId(1L)).thenReturn(java.util.Optional.empty());
    when(memorySummaryRepo.findByConversationIdOrderByCreatedAtDesc(eq(1L), any())).thenReturn(List.of());

    // 模拟豆包接口 5xx → callDoubaoRaw 走 catch 返回兜底
    when(restTemplate.exchange(anyString(), any(), any(), eq(String.class)))
            .thenThrow(new org.springframework.web.client.RestClientException("simulated 5xx"));

    com.sanyan.entity.AiCharacter character = new com.sanyan.entity.AiCharacter();
    character.setSystemPrompt("你是小婉");
    String result = aiService.chatProactive(character, 1L, "主动打招呼");

    assertThat(result).isNull();
}

@Test
void chat_shouldReturnFallbackString_whenDoubaoFallsBack() throws Exception {
    // 验证 chat 链路（用户主动发消息）保持原行为：失败时返回兜底文案
    var messageRepo = mock(com.sanyan.repository.MessageRepository.class);
    var memoryProfileRepo = mock(com.sanyan.repository.MemoryProfileRepository.class);
    var memorySummaryRepo = mock(com.sanyan.repository.MemorySummaryRepository.class);
    var objectMapper = new com.fasterxml.jackson.databind.ObjectMapper();
    var restTemplate = mock(org.springframework.web.client.RestTemplate.class);

    AiService aiService = new AiService(messageRepo, memoryProfileRepo, memorySummaryRepo, objectMapper, restTemplate);

    when(memoryProfileRepo.findByConversationId(1L)).thenReturn(java.util.Optional.empty());
    when(memorySummaryRepo.findByConversationIdOrderByCreatedAtDesc(eq(1L), any())).thenReturn(List.of());
    when(messageRepo.findByConversationIdOrderByIdDesc(eq(1L), any())).thenReturn(List.of());
    when(restTemplate.exchange(anyString(), any(), any(), eq(String.class)))
            .thenThrow(new org.springframework.web.client.RestClientException("simulated 5xx"));

    com.sanyan.entity.AiCharacter character = new com.sanyan.entity.AiCharacter();
    character.setSystemPrompt("你是小婉");
    String result = aiService.chat(character, 1L);

    assertThat(result).isEqualTo(AiService.AI_FALLBACK_MESSAGE);
}
```

- [ ] **Step 3.A2: 跑测试确认失败**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=AiServiceTest#chatProactive_shouldReturnNull_whenDoubaoFallsBack -q
mvn test -Dtest=AiServiceTest#chat_shouldReturnFallbackString_whenDoubaoFallsBack -q
```

期望：第一个失败 — `chatProactive` 当前返回兜底字符串而非 null。第二个编译失败 — `AI_FALLBACK_MESSAGE` 常量不存在。

- [ ] **Step 3.A3: 在 AiService 抽出兜底文案常量、改造 chatProactive**

修改 `AiService.java`：

1. 在类的字段区（建议放在 `@Value` 字段下方）追加：

```java
/**
 * Fallback message returned when doubao API call fails.
 * Used by chat/chatVoiceAck (user-triggered) as a graceful degradation.
 * For chatProactive (AI-initiated), this string triggers a null return — see chatProactive().
 */
static final String AI_FALLBACK_MESSAGE = "抱歉，我现在有点走神了，等下再聊吧~";
```

2. 修改 `callDoubaoRaw` 的 catch 块，使用常量替代字面量：

```java
} catch (Exception e) {
    log.error("豆包 API 调用失败", e);
    return AI_FALLBACK_MESSAGE;
}
```

3. 修改 `chatProactive` 方法，在调用后判定是否兜底，是则返回 null：

```java
public String chatProactive(AiCharacter character, Long conversationId, String triggerHint) {
    String time = formatCurrentTime();
    String profile = getProfile(conversationId);

    List<MemorySummary> summaries = memorySummaryRepository
            .findByConversationIdOrderByCreatedAtDesc(conversationId, PageRequest.of(0, 5));

    List<Map<String, String>> messages = buildProactiveMessages(
            character.getSystemPrompt(), time, profile, summaries, triggerHint);

    String result = callDoubaoRaw(messages);
    if (AI_FALLBACK_MESSAGE.equals(result)) {
        log.warn("豆包 API 失败，主动消息放弃。conversationId={}", conversationId);
        return null;
    }
    return result;
}
```

- [ ] **Step 3.A4: 跑测试确认通过**

```bash
mvn test -Dtest=AiServiceTest -q
```

期望：所有 AiServiceTest 通过（含原有 3 个 + 新增 2 个 = 5 个）。

#### Part B: ProactiveService 清洗后兜底

- [ ] **Step 3.B1: 写失败测试 — isSendableContent 对空白返回 false、对非空返回 true**

在 `ProactiveServiceTest.java` 内追加：

```java
@Test
void isSendableContent_shouldReturnFalseForBlank() {
    assertThat(ProactiveService.isSendableContent(null)).isFalse();
    assertThat(ProactiveService.isSendableContent("")).isFalse();
    assertThat(ProactiveService.isSendableContent("   ")).isFalse();
    assertThat(ProactiveService.isSendableContent("\n")).isFalse();
}

@Test
void isSendableContent_shouldReturnTrueForActualContent() {
    assertThat(ProactiveService.isSendableContent("嗨，最近怎么样？")).isTrue();
    assertThat(ProactiveService.isSendableContent("a")).isTrue();
}
```

- [ ] **Step 3.B2: 跑测试确认失败**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=ProactiveServiceTest -q
```

期望：编译失败 — `cannot find symbol: method isSendableContent`。

- [ ] **Step 3.B3: 实现 isSendableContent**

在 `ProactiveService.java` 类内任意位置（建议放在 `isRateLimited` 方法上方）追加：

```java
/**
 * Check if the cleaned content (after TextProcessor.extract) is sendable.
 * Blank content can result from AI replying with only emotion/action tags.
 */
static boolean isSendableContent(String cleanContent) {
    return cleanContent != null && !cleanContent.isBlank();
}
```

- [ ] **Step 3.B4: 跑测试确认通过**

```bash
cd /Users/aventador/code/3yan/server
mvn test -Dtest=ProactiveServiceTest -q
```

期望：所有测试通过（含原有 + 新增 2 个）。

- [ ] **Step 3.B5: 在 sendProactiveMessage 内调用兜底**

修改 `ProactiveService.java` 的 `sendProactiveMessage` 方法。在原 110-112 行：
```java
// Extract emotion tags and action descriptions
var extracted = TextProcessor.extract(aiReply);
String cleanContent = extracted.cleanText();
```

紧接着追加（在 `// Save message to DB` 之前）：

```java
// Defensive: if cleaned content is blank (AI replied only with tags), skip
if (!isSendableContent(cleanContent)) {
    log.warn("主动消息清洗后内容为空，跳过保存。conversationId={}, aiReply={}",
            conversationId, aiReply);
    return;
}
```

- [ ] **Step 3.B6: 跑全量后端测试确认无回归**

```bash
cd /Users/aventador/code/3yan/server
mvn test -q
```

期望：BUILD SUCCESS。

- [ ] **Step 3.B7: Commit**

```bash
cd /Users/aventador/code/3yan/server
git add src/main/java/com/sanyan/service/AiService.java \
        src/main/java/com/sanyan/service/ProactiveService.java \
        src/test/java/com/sanyan/service/AiServiceTest.java \
        src/test/java/com/sanyan/service/ProactiveServiceTest.java
git commit -m "fix: 主动消息隔离 AI 兜底文案 + 清洗后空内容跳过保存"
```

---

## 收尾验证

- [ ] **Step F.1: 全量测试**

```bash
cd /Users/aventador/code/3yan/server
mvn test -q
```

期望：BUILD SUCCESS，全部测试通过。

- [ ] **Step F.2: 检查 git log**

```bash
cd /Users/aventador/code/3yan/server
git log --oneline -5
```

期望看到 3 个新 commit（Task 1/2/3 各一个）。

- [ ] **Step F.3: 部署到测试环境前提醒**

⚠️ 注意：`data.sql` 的 INSERT 用 `WHERE NOT EXISTS`，**不会**更新已有的"小婉"角色行。要让线上的 proactive_config 也变成新格式，需要手动跑：

```sql
UPDATE ai_character SET proactive_config = '{"max_daily":3,"min_interval_hours":2,"active_hours":[8,22],"greeting":{"enabled":true,"slots":["08:00-09:00","12:00-13:00","21:00-22:00"]},"event_trigger":{"enabled":true,"idle_hours_threshold":6},"situational":{"enabled":true,"trigger_rate":0.2}}' WHERE name = '小婉';
```

把这条 SQL 在用户确认后由用户执行（或下一个会话由助手 ssh 进 beastify 跑）。
