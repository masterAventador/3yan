# ASR 接入实现计划（语音消息第二阶段）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 接入豆包大模型录音文件识别极速版 API，让小婉真正理解用户发送的语音消息内容并正常回复；同时清理项目中 senderType / contentType 字符串硬编码。

**Architecture:** 在 MessageService 的语音消息分支中插入 ASR 调用：保存用户消息 → 调 ASR 转写 → 回填 content 字段 → 走正常 aiService.chat() 流程。ASR 失败时降级到现有的 chatVoiceAck。客户端业务零改动。

**Tech Stack:** Spring Boot 3.2.5 / Java 17 / 火山引擎豆包极速版 ASR（volc.bigasr.auc_turbo）/ Flutter

---

## File Structure

### 服务端新增
| 文件 | 职责 |
|------|------|
| `server/src/main/java/com/sanyan/dto/ws/SenderType.java` | 发送者类型常量 |
| `server/src/main/java/com/sanyan/service/AsrService.java` | 豆包 ASR 极速版调用 |
| `server/src/test/java/com/sanyan/service/AsrServiceTest.java` | AsrService 单元测试 |

### 服务端修改
| 文件 | 改动 |
|------|------|
| `server/src/main/resources/application-dev.yml` | 新增 sanyan.asr.* 配置 |
| `server/src/main/java/com/sanyan/service/MessageService.java` | 注入 AsrService + 语音分支接入 ASR + 清理硬编码 |
| `server/src/main/java/com/sanyan/service/AiService.java` | senderType 硬编码 → SenderType 常量 |
| `server/src/main/java/com/sanyan/service/MemoryService.java` | senderType 硬编码 → SenderType 常量 |
| `server/src/main/java/com/sanyan/service/ProactiveService.java` | senderType 硬编码 → SenderType 常量 |

### 客户端新增
| 文件 | 职责 |
|------|------|
| `app/foundation_packages/sanyan_network/lib/src/sender_type.dart` | 发送者类型常量 |

### 客户端修改
| 文件 | 改动 |
|------|------|
| `app/foundation_packages/sanyan_network/lib/sanyan_network.dart` | 导出 sender_type.dart |
| `app/business_packages/sanyan_chat/lib/src/api/chat_api.dart` | 'voice' → ContentType.voice |
| `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart` | senderType / contentType 常量化 |
| `app/business_packages/sanyan_chat/lib/src/models/message.dart` | senderType / contentType 常量化 |
| `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart` | senderType 常量化 |
| `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart` | senderType 常量化 |

---

### Task 0: 服务端硬编码清理

**Files:**
- Create: `server/src/main/java/com/sanyan/dto/ws/SenderType.java`
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java`
- Modify: `server/src/main/java/com/sanyan/service/AiService.java`
- Modify: `server/src/main/java/com/sanyan/service/MemoryService.java`
- Modify: `server/src/main/java/com/sanyan/service/ProactiveService.java`

- [ ] **Step 1: 创建 SenderType 常量类**

创建 `server/src/main/java/com/sanyan/dto/ws/SenderType.java`：

```java
package com.sanyan.dto.ws;

public final class SenderType {
    public static final String USER = "user";
    public static final String AI = "ai";

    private SenderType() {}
}
```

- [ ] **Step 2: 替换 MessageService.java 中的硬编码**

修改 `server/src/main/java/com/sanyan/service/MessageService.java`：

在 imports 中添加：
```java
import com.sanyan.dto.ws.SenderType;
```

然后找到以下位置逐一替换：

第 48 行：
```java
userMsg.setSenderType("user");
```
改为：
```java
userMsg.setSenderType(SenderType.USER);
```

第 62 行：
```java
if ("voice".equals(contentType)) {
```
改为：
```java
if (MessageContentType.VOICE.equals(contentType)) {
```

（注意：`MessageContentType` 已经在文件里有 import，如果没有就加上 `import com.sanyan.dto.ws.MessageContentType;`）

第 82 行：
```java
aiMsg.setSenderType("ai");
```
改为：
```java
aiMsg.setSenderType(SenderType.AI);
```

第 116 行：
```java
aiMsg.setSenderType("ai");
```
改为：
```java
aiMsg.setSenderType(SenderType.AI);
```

- [ ] **Step 3: 替换 AiService.java 中的硬编码**

修改 `server/src/main/java/com/sanyan/service/AiService.java`：

imports 添加：
```java
import com.sanyan.dto.ws.SenderType;
```

第 132 行找到：
```java
String role = "user".equals(msg.getSenderType()) ? "user" : "assistant";
```
改为：
```java
String role = SenderType.USER.equals(msg.getSenderType()) ? "user" : "assistant";
```

（注意：这里右侧的 `"user"` 和 `"assistant"` 是豆包 Chat API 的 role 字段值，**不要改**，它们是 OpenAI 兼容协议的规范值，不属于我们的 senderType）

- [ ] **Step 4: 替换 MemoryService.java 中的硬编码**

修改 `server/src/main/java/com/sanyan/service/MemoryService.java`：

imports 添加：
```java
import com.sanyan.dto.ws.SenderType;
```

第 92 行和第 109 行找到：
```java
String role = "user".equals(msg.getSenderType()) ? "用户" : "AI";
```
改为：
```java
String role = SenderType.USER.equals(msg.getSenderType()) ? "用户" : "AI";
```

- [ ] **Step 5: 替换 ProactiveService.java 中的硬编码**

修改 `server/src/main/java/com/sanyan/service/ProactiveService.java`：

imports 添加：
```java
import com.sanyan.dto.ws.SenderType;
```

第 116 行找到：
```java
proactiveMsg.setSenderType("ai");
```
改为：
```java
proactiveMsg.setSenderType(SenderType.AI);
```

第 189 行找到：
```java
return "ai".equals(last.getSenderType()) && "proactive".equals(last.getSource());
```
改为：
```java
return SenderType.AI.equals(last.getSenderType()) && "proactive".equals(last.getSource());
```

（"proactive" 是 source 字段，不是 senderType，暂时不动）

- [ ] **Step 6: 运行全量测试确认没有回归**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test`
Expected: 全部测试通过（新旧测试都不应该因为常量替换失败）

- [ ] **Step 7: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add -A
git commit -m "refactor: 服务端 senderType / contentType 硬编码统一用常量类"
```

---

### Task 1: 客户端硬编码清理

**Files:**
- Create: `app/foundation_packages/sanyan_network/lib/src/sender_type.dart`
- Modify: `app/foundation_packages/sanyan_network/lib/sanyan_network.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/api/chat_api.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/models/message.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart`

- [ ] **Step 1: 创建 SenderType 常量类**

创建 `app/foundation_packages/sanyan_network/lib/src/sender_type.dart`：

```dart
abstract class SenderType {
  static const String user = 'user';
  static const String ai = 'ai';
}
```

- [ ] **Step 2: 导出 SenderType**

修改 `app/foundation_packages/sanyan_network/lib/sanyan_network.dart`，在已有的 exports 之后添加：

```dart
export 'src/sender_type.dart';
```

- [ ] **Step 3: 替换 chat_api.dart 中的硬编码**

修改 `app/business_packages/sanyan_chat/lib/src/api/chat_api.dart`：

第 53 行（在 `FormData.fromMap` 里）找到：
```dart
'type': 'voice',
```
改为：
```dart
'type': ContentType.voice,
```

（`ContentType` 应该已经通过 `sanyan_network` import 在用了，如果没有就加 `import 'package:sanyan_network/sanyan_network.dart';`）

- [ ] **Step 4: 替换 chat_controller.dart 中的硬编码**

修改 `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`：

确保顶部有 import：
```dart
import 'package:sanyan_network/sanyan_network.dart';
```

第 89-90 行找到：
```dart
senderType: 'user',
contentType: 'text',
```
改为：
```dart
senderType: SenderType.user,
contentType: ContentType.text,
```

第 108 行找到：
```dart
senderType: 'user',
```
改为：
```dart
senderType: SenderType.user,
```

- [ ] **Step 5: 替换 message.dart 中的硬编码**

修改 `app/business_packages/sanyan_chat/lib/src/models/message.dart`：

第 33 行找到：
```dart
bool get isFromAi => senderType == 'ai';
```
改为：
```dart
bool get isFromAi => senderType == SenderType.ai;
```

第 43 行找到：
```dart
contentType: json['contentType'] ?? 'text',
```
改为：
```dart
contentType: json['contentType'] ?? ContentType.text,
```

（SenderType 和 ContentType 已通过 `import 'package:sanyan_network/sanyan_network.dart';` 可用，如果文件顶部没有这个 import 就加上）

- [ ] **Step 6: 替换 message_bubble.dart 中的硬编码**

修改 `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart`：

第 15 行找到：
```dart
final isUser = message.senderType == 'user';
```
改为：
```dart
final isUser = message.senderType == SenderType.user;
```

确保顶部有 `import 'package:sanyan_network/sanyan_network.dart';`（应该已有，因为用了 ContentType）

- [ ] **Step 7: 替换 voice_bubble.dart 中的硬编码**

修改 `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart`：

第 51 行找到：
```dart
final isUser = widget.message.senderType == 'user';
```
改为：
```dart
final isUser = widget.message.senderType == SenderType.user;
```

确保顶部有 `import 'package:sanyan_network/sanyan_network.dart';`

- [ ] **Step 8: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 9: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "refactor: 客户端 senderType / contentType 硬编码统一用常量类"
```

---

### Task 2: AsrService 实现 + 单元测试

**Files:**
- Create: `server/src/main/java/com/sanyan/service/AsrService.java`
- Create: `server/src/test/java/com/sanyan/service/AsrServiceTest.java`
- Modify: `server/src/main/resources/application-dev.yml`

- [ ] **Step 1: 添加配置**

修改 `server/src/main/resources/application-dev.yml`，在已有的 `sanyan:` 节点下（与 `tts:` 和 `cos:` 同级）添加：

```yaml
  asr:
    enabled: true
    app-id: "6383804695"
    access-token: ${ASR_ACCESS_TOKEN:BgkVhS2NOZfEP9TbhuWdREWzbZEjNLHi}
```

- [ ] **Step 2: 写 AsrService 的失败测试**

创建 `server/src/test/java/com/sanyan/service/AsrServiceTest.java`：

```java
package com.sanyan.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.util.ReflectionTestUtils;
import org.springframework.web.client.RestClientException;
import org.springframework.web.client.RestTemplate;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class AsrServiceTest {

    private RestTemplate restTemplate;
    private ObjectMapper objectMapper;
    private AsrService asrService;

    @BeforeEach
    void setUp() {
        restTemplate = mock(RestTemplate.class);
        objectMapper = new ObjectMapper();
        asrService = new AsrService(restTemplate, objectMapper);
        ReflectionTestUtils.setField(asrService, "enabled", true);
        ReflectionTestUtils.setField(asrService, "appId", "6383804695");
        ReflectionTestUtils.setField(asrService, "accessToken", "test-token");
    }

    @Test
    void transcribe_success_returnsText() {
        HttpHeaders respHeaders = new HttpHeaders();
        respHeaders.add("X-Api-Status-Code", "20000000");
        String body = "{\"result\":{\"text\":\"今天心情不好\"}}";
        ResponseEntity<String> response = new ResponseEntity<>(body, respHeaders, HttpStatus.OK);

        when(restTemplate.exchange(
                eq("https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash"),
                eq(HttpMethod.POST),
                any(HttpEntity.class),
                eq(String.class)
        )).thenReturn(response);

        String result = asrService.transcribe("https://example.com/audio.m4a");

        assertThat(result).isEqualTo("今天心情不好");
    }

    @Test
    void transcribe_silenceAudio_returnsEmptyString() {
        HttpHeaders respHeaders = new HttpHeaders();
        respHeaders.add("X-Api-Status-Code", "20000003");
        ResponseEntity<String> response = new ResponseEntity<>("{}", respHeaders, HttpStatus.OK);

        when(restTemplate.exchange(
                eq("https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash"),
                eq(HttpMethod.POST),
                any(HttpEntity.class),
                eq(String.class)
        )).thenReturn(response);

        String result = asrService.transcribe("https://example.com/audio.m4a");

        assertThat(result).isEmpty();
    }

    @Test
    void transcribe_errorCode_retriesOnceThenReturnsNull() {
        HttpHeaders respHeaders = new HttpHeaders();
        respHeaders.add("X-Api-Status-Code", "45000151");
        ResponseEntity<String> response = new ResponseEntity<>("{}", respHeaders, HttpStatus.OK);

        when(restTemplate.exchange(
                eq("https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash"),
                eq(HttpMethod.POST),
                any(HttpEntity.class),
                eq(String.class)
        )).thenReturn(response);

        String result = asrService.transcribe("https://example.com/audio.m4a");

        assertThat(result).isNull();
        // 重试 1 次 = 共调用 2 次
        verify(restTemplate, times(2)).exchange(
                any(String.class), any(HttpMethod.class), any(HttpEntity.class), eq(String.class));
    }

    @Test
    void transcribe_networkException_retriesOnceThenReturnsNull() {
        when(restTemplate.exchange(
                eq("https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash"),
                eq(HttpMethod.POST),
                any(HttpEntity.class),
                eq(String.class)
        )).thenThrow(new RestClientException("network error"));

        String result = asrService.transcribe("https://example.com/audio.m4a");

        assertThat(result).isNull();
        verify(restTemplate, times(2)).exchange(
                any(String.class), any(HttpMethod.class), any(HttpEntity.class), eq(String.class));
    }

    @Test
    void transcribe_firstCallFailsSecondSucceeds_returnsText() {
        HttpHeaders successHeaders = new HttpHeaders();
        successHeaders.add("X-Api-Status-Code", "20000000");
        ResponseEntity<String> successResponse = new ResponseEntity<>(
                "{\"result\":{\"text\":\"重试后成功\"}}",
                successHeaders,
                HttpStatus.OK);

        List<ResponseEntity<String>> responses = new ArrayList<>();
        responses.add(null); // placeholder for exception

        when(restTemplate.exchange(
                eq("https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash"),
                eq(HttpMethod.POST),
                any(HttpEntity.class),
                eq(String.class)
        ))
                .thenThrow(new RestClientException("first call fails"))
                .thenReturn(successResponse);

        String result = asrService.transcribe("https://example.com/audio.m4a");

        assertThat(result).isEqualTo("重试后成功");
    }

    @Test
    void isEnabled_returnsConfigValue() {
        assertThat(asrService.isEnabled()).isTrue();

        ReflectionTestUtils.setField(asrService, "enabled", false);
        assertThat(asrService.isEnabled()).isFalse();
    }
}
```

- [ ] **Step 3: 运行测试确认失败**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=AsrServiceTest`
Expected: 编译失败，AsrService 类不存在

- [ ] **Step 4: 写 AsrService**

创建 `server/src/main/java/com/sanyan/service/AsrService.java`：

```java
package com.sanyan.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.UUID;

/**
 * 豆包大模型录音文件识别极速版（volc.bigasr.auc_turbo）调用。
 * 一次 HTTP 请求返回识别结果，不需要轮询。
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AsrService {

    private static final String ASR_URL =
            "https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash";
    private static final String RESOURCE_ID = "volc.bigasr.auc_turbo";

    // 状态码
    private static final String CODE_SUCCESS = "20000000";
    private static final String CODE_SILENCE = "20000003";

    // 重试次数（首次 + 重试）
    private static final int MAX_ATTEMPTS = 2;

    private final RestTemplate restTemplate;
    private final ObjectMapper objectMapper;

    @Value("${sanyan.asr.enabled:false}")
    private boolean enabled;

    @Value("${sanyan.asr.app-id:}")
    private String appId;

    @Value("${sanyan.asr.access-token:}")
    private String accessToken;

    public boolean isEnabled() {
        return enabled;
    }

    /**
     * 转写音频为文字
     *
     * @param audioUrl 公网可访问的音频 URL
     * @return 转写文字；静音返回空字符串；失败返回 null
     */
    public String transcribe(String audioUrl) {
        for (int attempt = 0; attempt < MAX_ATTEMPTS; attempt++) {
            try {
                String result = doTranscribe(audioUrl);
                if (result != null) {
                    return result;
                }
                log.warn("ASR 调用返回空结果，attempt={}", attempt);
            } catch (Exception e) {
                log.warn("ASR 调用异常, attempt={}", attempt, e);
            }
        }
        return null;
    }

    /**
     * 执行一次 ASR 请求
     *
     * @return 识别文字；静音返回 ""；失败或错误状态码返回 null
     */
    private String doTranscribe(String audioUrl) throws Exception {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("X-Api-App-Key", appId);
        headers.set("X-Api-Access-Key", accessToken);
        headers.set("X-Api-Resource-Id", RESOURCE_ID);
        headers.set("X-Api-Request-Id", UUID.randomUUID().toString());
        headers.set("X-Api-Sequence", "-1");

        Map<String, Object> body = new LinkedHashMap<>();
        body.put("user", Map.of("uid", "sanyan_server"));
        body.put("audio", Map.of("url", audioUrl));

        Map<String, Object> request = new LinkedHashMap<>();
        request.put("model_name", "bigmodel");
        request.put("enable_itn", true);
        request.put("enable_punc", true);
        body.put("request", request);

        String requestBody = objectMapper.writeValueAsString(body);
        HttpEntity<String> entity = new HttpEntity<>(requestBody, headers);

        log.info("ASR 请求: audioUrl={}", audioUrl);
        long start = System.currentTimeMillis();

        ResponseEntity<String> response = restTemplate.exchange(
                ASR_URL, HttpMethod.POST, entity, String.class);

        long elapsed = System.currentTimeMillis() - start;
        String statusCode = response.getHeaders().getFirst("X-Api-Status-Code");
        String logId = response.getHeaders().getFirst("X-Tt-Logid");
        log.info("ASR 响应: statusCode={}, logId={}, 耗时={}ms", statusCode, logId, elapsed);

        if (CODE_SUCCESS.equals(statusCode)) {
            // 解析 result.text
            @SuppressWarnings("unchecked")
            Map<String, Object> responseMap =
                    objectMapper.readValue(response.getBody(), Map.class);
            @SuppressWarnings("unchecked")
            Map<String, Object> result = (Map<String, Object>) responseMap.get("result");
            if (result == null) {
                log.error("ASR 响应缺少 result 字段: {}", response.getBody());
                return null;
            }
            String text = (String) result.get("text");
            return text != null ? text : "";
        }

        if (CODE_SILENCE.equals(statusCode)) {
            log.info("ASR 识别到静音音频");
            return "";
        }

        log.error("ASR 返回错误状态码: {}", statusCode);
        return null;
    }
}
```

- [ ] **Step 5: 运行测试确认通过**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=AsrServiceTest`
Expected: 6 tests PASS

- [ ] **Step 6: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add -A
git commit -m "feat: AsrService 豆包极速版 ASR 接入"
```

---

### Task 3: MessageService 接入 ASR 流程

**Files:**
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java`

- [ ] **Step 1: 注入 AsrService**

修改 `server/src/main/java/com/sanyan/service/MessageService.java`，在类字段区添加：

```java
private final AsrService asrService;
```

（由于 `@RequiredArgsConstructor`，构造器会自动注入）

- [ ] **Step 2: 修改语音消息分支**

在 `handleUserMessage()` 方法中，找到 Task 0 已经修改过的这段：

```java
String aiReply;
if (MessageContentType.VOICE.equals(contentType)) {
    aiReply = aiService.chatVoiceAck(character, conversationId);
} else {
    aiReply = aiService.chat(character, conversationId);
}
```

改为：

```java
String aiReply;
if (MessageContentType.VOICE.equals(contentType)) {
    // 尝试 ASR 转写
    String transcribedText = asrService.isEnabled()
            ? asrService.transcribe(mediaUrl)
            : null;

    if (transcribedText != null && !transcribedText.isBlank()) {
        // ASR 成功：回填 content 字段，走正常对话
        userMsg.setContent(transcribedText);
        messageRepository.save(userMsg);
        log.info("ASR 转写成功: convId={}, msgId={}, text={}",
                conversationId, userMsg.getId(), transcribedText);
        aiReply = aiService.chat(character, conversationId);
    } else {
        // ASR 失败或静音：降级到 chatVoiceAck
        log.info("ASR 失败或静音，降级为 chatVoiceAck: convId={}", conversationId);
        aiReply = aiService.chatVoiceAck(character, conversationId);
    }
} else {
    aiReply = aiService.chat(character, conversationId);
}
```

- [ ] **Step 3: 运行全量测试确认无回归**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test`
Expected: 全部测试通过

- [ ] **Step 4: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add -A
git commit -m "feat: MessageService 语音分支接入 ASR 转写流程"
```

---

### Task 4: 服务端部署并端到端验证

**Files:** 无代码改动

- [ ] **Step 1: 打包**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn clean package -DskipTests -q`
Expected: BUILD SUCCESS，生成 `target/sanyan-server-0.1.0.jar`

- [ ] **Step 2: 上传到服务器**

Run: `scp target/sanyan-server-0.1.0.jar beastify:/opt/sanyan/`

- [ ] **Step 3: 重启服务**

Run: `ssh beastify "systemctl restart sanyan"`

- [ ] **Step 4: 检查启动日志**

Run:
```bash
ssh beastify "journalctl -u sanyan --since '1 min ago' --no-pager" 2>&1 | grep -E "Started SanyanApplication|ERROR" | tail -5
```
Expected: 看到 `Started SanyanApplication`，没有 ERROR

- [ ] **Step 5: 推送代码**

```bash
cd /Users/aventador/sourceCode/3yan/server
git push origin dev
```

- [ ] **Step 6: 手动端到端验证（用户在 App 上操作）**

**用户需要完成以下验证**：
1. 在 App 上发一条语音消息说"今天心情有点不好"
2. 服务器日志应该依次出现：
   - `ASR 请求: audioUrl=https://3yan-...`
   - `ASR 响应: statusCode=20000000, logId=..., 耗时=XXXms`
   - `ASR 转写成功: convId=X, msgId=X, text=今天心情有点不好。`
3. 小婉的回复应该与"心情不好"相关的安慰话，而不是"没听清"的撒娇话
4. 数据库：`SELECT content FROM message WHERE id = XXX` 应该返回转写的文字

**如果 ASR 失败（状态码 45000151 音频格式不正确）**：
- 日志会显示 `ASR 返回错误状态码: 45000151`
- 小婉会走降级回复
- 需要执行 Task 5（音频格式降级）

- [ ] **Step 7: 验证失败场景（可选）**

1. 临时把 `application-dev.yml` 的 `sanyan.asr.enabled` 改为 `false`，重启服务
2. 发语音 → 应该直接走 `chatVoiceAck` 降级回复
3. 日志应该有 `ASR 失败或静音，降级为 chatVoiceAck`
4. 改回 `true` 重启恢复

---

### Task 5: m4a 兼容性兜底 — 客户端改录 WAV（条件执行）

**前置条件**：仅在 Task 4 验证时发现 ASR 返回 `45000151`（音频格式不正确）才执行此任务。如果 m4a 能正常识别，跳过本 Task。

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/voice_recorder.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/voice_cache_manager.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/api/chat_api.dart`

- [ ] **Step 1: 修改 VoiceRecorder 为 WAV 编码**

修改 `app/business_packages/sanyan_chat/lib/src/chat/voice_recorder.dart`：

找到 `_recorder.start(...)` 调用中的 `RecordConfig`：

```dart
await _recorder.start(
  const RecordConfig(
    encoder: AudioEncoder.aacLc,
    sampleRate: 24000,
    numChannels: 1,
  ),
  path: _currentFilePath!,
);
```

改为：

```dart
await _recorder.start(
  const RecordConfig(
    encoder: AudioEncoder.wav,
    sampleRate: 16000,
    numChannels: 1,
  ),
  path: _currentFilePath!,
);
```

**说明**：
- `AudioEncoder.wav` 是无损 PCM 编码
- 采样率降到 16000 Hz（语音识别标准采样率，足够清晰，文件更小）
- 单声道（`numChannels: 1`）

- [ ] **Step 2: 修改缓存文件扩展名**

修改 `app/business_packages/sanyan_chat/lib/src/chat/voice_cache_manager.dart`：

找到：
```dart
static Future<String> newVoiceFilePath(String uuid) async {
  final dir = await getCacheDir();
  return '${dir.path}/$uuid.m4a';
}
```

改为：
```dart
static Future<String> newVoiceFilePath(String uuid) async {
  final dir = await getCacheDir();
  return '${dir.path}/$uuid.wav';
}
```

- [ ] **Step 3: 修改上传时的 filename 和 Content-Type**

修改 `app/business_packages/sanyan_chat/lib/src/api/chat_api.dart`：

找到 `uploadVoice` 方法里构建 `FormData` 的位置：

```dart
final formData = FormData.fromMap({
  'file': await MultipartFile.fromFile(localFilePath, filename: 'voice.m4a'),
  'type': ContentType.voice,
  'duration': duration,
});
```

改为：

```dart
final formData = FormData.fromMap({
  'file': await MultipartFile.fromFile(localFilePath, filename: 'voice.wav'),
  'type': ContentType.voice,
  'duration': duration,
});
```

- [ ] **Step 4: 服务端 MediaService 兼容 WAV content-type**

修改 `server/src/main/java/com/sanyan/service/MediaService.java`，在 `uploadVoice()` 中，找到：

```java
String url = cosService.upload(file.getBytes(), key, "audio/mp4");
```

改为（同时兼容 m4a 和 wav）：

```java
String contentType = key.endsWith(".wav") ? "audio/wav" : "audio/mp4";
String url = cosService.upload(file.getBytes(), key, contentType);
```

同时 `CosService.buildUserVoiceKey` 也需要支持 wav 后缀。修改 `server/src/main/java/com/sanyan/service/CosService.java`：

找到：
```java
public static String buildUserVoiceKey(Long userId, String uuid) {
    return "voice/user/" + userId + "/" + System.currentTimeMillis() + "_" + uuid + ".m4a";
}
```

改为：
```java
public static String buildUserVoiceKey(Long userId, String uuid, String extension) {
    return "voice/user/" + userId + "/" + System.currentTimeMillis() + "_" + uuid + "." + extension;
}
```

然后在 `MediaService.uploadVoice()` 里根据上传的文件原始名判断扩展名：

```java
String originalFilename = file.getOriginalFilename();
String extension = "m4a"; // 默认
if (originalFilename != null && originalFilename.toLowerCase().endsWith(".wav")) {
    extension = "wav";
}
String key = CosService.buildUserVoiceKey(userId, uuid, extension);
String contentType = "wav".equals(extension) ? "audio/wav" : "audio/mp4";
String url = cosService.upload(file.getBytes(), key, contentType);
```

- [ ] **Step 5: 编译验证**

客户端：
```bash
cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze
```
Expected: No issues found

服务端：
```bash
cd /Users/aventador/sourceCode/3yan/server && mvn test
```
Expected: 全部测试通过（可能 `CosServiceTest.buildObjectKeyForUserVoice_returnsCorrectPath` 会因为签名变化失败，需要更新测试的方法调用：`CosService.buildUserVoiceKey(1L, "abc123", "m4a")`）

如果 CosServiceTest 失败，修复测试：

找到：
```java
@Test
void buildObjectKeyForUserVoice_returnsCorrectPath() {
    String key = CosService.buildUserVoiceKey(1L, "abc123");
    assertThat(key).startsWith("voice/user/1/");
    assertThat(key).endsWith("_abc123.m4a");
}
```

改为：
```java
@Test
void buildObjectKeyForUserVoice_returnsCorrectPath() {
    String key = CosService.buildUserVoiceKey(1L, "abc123", "m4a");
    assertThat(key).startsWith("voice/user/1/");
    assertThat(key).endsWith("_abc123.m4a");
}

@Test
void buildObjectKeyForUserVoice_withWavExtension() {
    String key = CosService.buildUserVoiceKey(1L, "abc123", "wav");
    assertThat(key).endsWith("_abc123.wav");
}
```

再次运行 `mvn test` 确保通过。

- [ ] **Step 6: 提交**

服务端：
```bash
cd /Users/aventador/sourceCode/3yan/server
git add -A
git commit -m "fix: MediaService 支持 WAV 格式上传"
```

客户端：
```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "fix: VoiceRecorder 从 AAC/m4a 改为 WAV 格式，兼容 ASR 极速版"
```

- [ ] **Step 7: 打包并部署服务端**

```bash
cd /Users/aventador/sourceCode/3yan/server && mvn clean package -DskipTests -q
scp target/sanyan-server-0.1.0.jar beastify:/opt/sanyan/
ssh beastify "systemctl restart sanyan"
```

- [ ] **Step 8: 推送代码**

```bash
cd /Users/aventador/sourceCode/3yan/server && git push origin dev
cd /Users/aventador/sourceCode/3yan/app && git push origin dev
```

- [ ] **Step 9: 重新端到端验证**

跟 Task 4 Step 6 一样，发一条语音验证 ASR 能正常识别。
