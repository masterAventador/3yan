# TTS 接入实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 后端接入豆包语音合成 2.0，AI 回复支持语音模式，语音文件上传腾讯云 COS，客户端支持语音消息播放。

**Architecture:** 在现有 AI 回复流程中插入 TTS 环节。AI 文字回复 → TextProcessor 提取动作描述 → TtsService 合成音频 → CosService 上传 COS → 消息带 mediaUrl 推送客户端。通过配置变量 tts.enabled 控制开关。

**Tech Stack:** Spring Boot 3.2.5 / Java 17 / 火山引擎 TTS HTTP API / 腾讯云 COS Java SDK / Flutter (audioplayers)

---

### Task 1: TextProcessor — 动作描述提取工具类

**Files:**
- Create: `server/src/main/java/com/sanyan/util/TextProcessor.java`
- Test: `server/src/test/java/com/sanyan/util/TextProcessorTest.java`

- [ ] **Step 1: 写失败的测试**

```java
package com.sanyan.util;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TextProcessorTest {

    @Test
    void extractActions_withSingleAction() {
        var result = TextProcessor.extract("你好呀（歪头微笑）今天怎么样？");
        assertThat(result.cleanText()).isEqualTo("你好呀今天怎么样？");
        assertThat(result.actions()).containsExactly("歪头微笑");
    }

    @Test
    void extractActions_withMultipleActions() {
        var result = TextProcessor.extract("嗯（点头）我知道了（双手抱胸）");
        assertThat(result.cleanText()).isEqualTo("嗯我知道了");
        assertThat(result.actions()).containsExactly("点头", "双手抱胸");
    }

    @Test
    void extractActions_withNoActions() {
        var result = TextProcessor.extract("普通的回复没有动作");
        assertThat(result.cleanText()).isEqualTo("普通的回复没有动作");
        assertThat(result.actions()).isEmpty();
    }

    @Test
    void extractActions_withEnglishParentheses_shouldNotExtract() {
        var result = TextProcessor.extract("这是(英文括号)不提取");
        assertThat(result.cleanText()).isEqualTo("这是(英文括号)不提取");
        assertThat(result.actions()).isEmpty();
    }

    @Test
    void extractActions_withEmptyInput() {
        var result = TextProcessor.extract("");
        assertThat(result.cleanText()).isEmpty();
        assertThat(result.actions()).isEmpty();
    }

    @Test
    void extractActions_actionAtStartAndEnd() {
        var result = TextProcessor.extract("（害羞地低头）谢谢你呀（开心地跳起来）");
        assertThat(result.cleanText()).isEqualTo("谢谢你呀");
        assertThat(result.actions()).containsExactly("害羞地低头", "开心地跳起来");
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=TextProcessorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 编译失败，TextProcessor 不存在

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.util;

import java.util.ArrayList;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class TextProcessor {

    private static final Pattern ACTION_PATTERN = Pattern.compile("（([^）]+)）");

    public record ExtractResult(String cleanText, List<String> actions) {}

    public static ExtractResult extract(String text) {
        if (text == null || text.isEmpty()) {
            return new ExtractResult("", List.of());
        }

        List<String> actions = new ArrayList<>();
        Matcher matcher = ACTION_PATTERN.matcher(text);
        while (matcher.find()) {
            actions.add(matcher.group(1));
        }

        String cleanText = ACTION_PATTERN.matcher(text).replaceAll("");
        return new ExtractResult(cleanText, actions);
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=TextProcessorTest`
Expected: 6 tests PASS

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add src/main/java/com/sanyan/util/TextProcessor.java src/test/java/com/sanyan/util/TextProcessorTest.java
git commit -m "feat: TextProcessor 动作描述提取工具类"
```

---

### Task 2: CosService — 腾讯云 COS 上传

**Files:**
- Modify: `server/pom.xml` (添加 COS SDK 依赖)
- Modify: `server/src/main/resources/application-dev.yml` (添加 COS 配置)
- Create: `server/src/main/java/com/sanyan/service/CosService.java`
- Test: `server/src/test/java/com/sanyan/service/CosServiceTest.java`

- [ ] **Step 1: 添加 Maven 依赖**

在 `server/pom.xml` 的 `<dependencies>` 中添加：

```xml
<!-- Tencent Cloud COS -->
<dependency>
    <groupId>com.qcloud</groupId>
    <artifactId>cos_api</artifactId>
    <version>5.6.227</version>
</dependency>
```

- [ ] **Step 2: 添加配置**

在 `server/src/main/resources/application-dev.yml` 的 `sanyan:` 下添加：

```yaml
  tts:
    enabled: false
    app-id: "7231248180"
    access-token: ${TTS_ACCESS_TOKEN:VuO0W4r1DVtvxl_7bEfuE3Zbr3iBVuL9}
    voice-type: zh_female_vv_uranus_bigtts
    cluster: volcano_tts
  cos:
    secret-id: ${COS_SECRET_ID:AKIDMcBZkjW9Zt0ta5SccOw1sXyDHD2g7LOG}
    secret-key: ${COS_SECRET_KEY:lKhqQo270f8OTO7dz99wVcXEj7IJBsQm}
    bucket: 3yan-1258800826
    region: ap-beijing
```

- [ ] **Step 3: 写失败的测试**

```java
package com.sanyan.service;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class CosServiceTest {

    @Test
    void buildCosUrl_returnsCorrectFormat() {
        String url = CosService.buildCosUrl("3yan-1258800826", "ap-beijing", "voice/1/100.mp3");
        assertThat(url).isEqualTo("https://3yan-1258800826.cos.ap-beijing.myqcloud.com/voice/1/100.mp3");
    }

    @Test
    void buildObjectKey_returnsCorrectPath() {
        String key = CosService.buildObjectKey(42L, 100L);
        assertThat(key).isEqualTo("voice/42/100.mp3");
    }
}
```

- [ ] **Step 4: 运行测试确认失败**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=CosServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 编译失败，CosService 不存在

- [ ] **Step 5: 写最小实现**

```java
package com.sanyan.service;

import com.qcloud.cos.COSClient;
import com.qcloud.cos.ClientConfig;
import com.qcloud.cos.auth.BasicCOSCredentials;
import com.qcloud.cos.model.ObjectMetadata;
import com.qcloud.cos.region.Region;
import jakarta.annotation.PostConstruct;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.io.ByteArrayInputStream;

@Slf4j
@Service
public class CosService {

    @Value("${sanyan.cos.secret-id}")
    private String secretId;

    @Value("${sanyan.cos.secret-key}")
    private String secretKey;

    @Value("${sanyan.cos.bucket}")
    private String bucket;

    @Value("${sanyan.cos.region}")
    private String region;

    private COSClient cosClient;

    @PostConstruct
    public void init() {
        var cred = new BasicCOSCredentials(secretId, secretKey);
        var config = new ClientConfig(new Region(region));
        cosClient = new COSClient(cred, config);
        log.info("COS 客户端初始化完成: bucket={}, region={}", bucket, region);
    }

    public String upload(byte[] data, Long conversationId, Long messageId) {
        String key = buildObjectKey(conversationId, messageId);
        try {
            var metadata = new ObjectMetadata();
            metadata.setContentLength(data.length);
            metadata.setContentType("audio/mpeg");
            cosClient.putObject(bucket, key, new ByteArrayInputStream(data), metadata);
            String url = buildCosUrl(bucket, region, key);
            log.info("语音上传 COS 成功: key={}, size={}bytes, url={}", key, data.length, url);
            return url;
        } catch (Exception e) {
            log.error("COS 上传失败: key={}", key, e);
            throw new RuntimeException("语音文件上传失败", e);
        }
    }

    public static String buildObjectKey(Long conversationId, Long messageId) {
        return "voice/" + conversationId + "/" + messageId + ".mp3";
    }

    public static String buildCosUrl(String bucket, String region, String key) {
        return "https://" + bucket + ".cos." + region + ".myqcloud.com/" + key;
    }
}
```

- [ ] **Step 6: 运行测试确认通过**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=CosServiceTest`
Expected: 2 tests PASS

- [ ] **Step 7: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add pom.xml src/main/resources/application-dev.yml src/main/java/com/sanyan/service/CosService.java src/test/java/com/sanyan/service/CosServiceTest.java
git commit -m "feat: CosService 腾讯云 COS 上传服务"
```

---

### Task 3: TtsService — 火山引擎 TTS 调用

**Files:**
- Create: `server/src/main/java/com/sanyan/service/TtsService.java`
- Test: `server/src/test/java/com/sanyan/service/TtsServiceTest.java`

- [ ] **Step 1: 写失败的测试**

```java
package com.sanyan.service;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TtsServiceTest {

    @Test
    void buildRequestBody_withoutActions() {
        String body = TtsService.buildRequestBody(
                "7231248180", "volcano_tts",
                "zh_female_vv_uranus_bigtts", "你好呀", null);
        assertThat(body).contains("\"appid\":\"7231248180\"");
        assertThat(body).contains("\"voice_type\":\"zh_female_vv_uranus_bigtts\"");
        assertThat(body).contains("\"text\":\"你好呀\"");
        assertThat(body).contains("\"encoding\":\"mp3\"");
    }

    @Test
    void buildRequestBody_withActions() {
        String body = TtsService.buildRequestBody(
                "7231248180", "volcano_tts",
                "zh_female_vv_uranus_bigtts", "你好呀",
                java.util.List.of("害羞地低头"));
        assertThat(body).contains("你好呀");
        // 动作描述应该以某种形式影响请求（语音标签或 SSML）
        assertThat(body).contains("害羞");
    }
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=TtsServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 编译失败，TtsService 不存在

- [ ] **Step 3: 写最小实现**

```java
package com.sanyan.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class TtsService {

    private static final String TTS_URL = "https://openspeech.bytedance.com/api/v1/tts";

    private final RestTemplate restTemplate;
    private final ObjectMapper objectMapper;

    @Value("${sanyan.tts.enabled:false}")
    private boolean enabled;

    @Value("${sanyan.tts.app-id:}")
    private String appId;

    @Value("${sanyan.tts.access-token:}")
    private String accessToken;

    @Value("${sanyan.tts.voice-type:zh_female_vv_uranus_bigtts}")
    private String voiceType;

    @Value("${sanyan.tts.cluster:volcano_tts}")
    private String cluster;

    public boolean isEnabled() {
        return enabled;
    }

    /**
     * 合成语音，返回 MP3 音频字节数组
     */
    public byte[] synthesize(String text, List<String> actions) {
        if (text == null || text.isBlank()) {
            return null;
        }

        try {
            String requestBody = buildRequestBody(appId, cluster, voiceType, text, actions);

            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            headers.set("Authorization", "Bearer;" + accessToken);

            HttpEntity<String> entity = new HttpEntity<>(requestBody, headers);

            log.info("TTS 请求: textLength={}, actions={}", text.length(),
                    actions != null ? actions.size() : 0);
            long start = System.currentTimeMillis();

            ResponseEntity<String> response = restTemplate.exchange(
                    TTS_URL, HttpMethod.POST, entity, String.class);

            long elapsed = System.currentTimeMillis() - start;
            log.info("TTS 响应: status={}, 耗时={}ms", response.getStatusCode(), elapsed);

            @SuppressWarnings("unchecked")
            Map<String, Object> responseMap = objectMapper.readValue(response.getBody(), Map.class);
            String audioBase64 = (String) responseMap.get("data");
            if (audioBase64 == null) {
                log.error("TTS 返回无音频数据: {}", response.getBody());
                return null;
            }

            return Base64.getDecoder().decode(audioBase64);

        } catch (Exception e) {
            log.error("TTS 合成失败", e);
            return null;
        }
    }

    /**
     * 构建 TTS 请求体 JSON
     */
    public static String buildRequestBody(String appId, String cluster,
                                           String voiceType, String text,
                                           List<String> actions) {
        // 如果有动作描述，将其作为语音标签前缀插入文本
        String finalText = text;
        if (actions != null && !actions.isEmpty()) {
            String actionHint = "[" + String.join("，", actions) + "]";
            finalText = actionHint + text;
        }

        Map<String, Object> body = new LinkedHashMap<>();

        Map<String, Object> app = new LinkedHashMap<>();
        app.put("appid", appId);
        app.put("token", "access_token");
        app.put("cluster", cluster);
        body.put("app", app);

        Map<String, String> user = Map.of("uid", "sanyan_server");
        body.put("user", user);

        Map<String, Object> audio = new LinkedHashMap<>();
        audio.put("voice_type", voiceType);
        audio.put("encoding", "mp3");
        audio.put("speed_ratio", 1.0);
        audio.put("volume_ratio", 1.0);
        audio.put("pitch_ratio", 1.0);
        body.put("audio", audio);

        Map<String, Object> request = new LinkedHashMap<>();
        request.put("reqid", UUID.randomUUID().toString());
        request.put("text", finalText);
        request.put("text_type", "plain");
        request.put("operation", "query");
        body.put("request", request);

        try {
            return new ObjectMapper().writeValueAsString(body);
        } catch (Exception e) {
            throw new RuntimeException("构建 TTS 请求体失败", e);
        }
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=TtsServiceTest`
Expected: 2 tests PASS

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add src/main/java/com/sanyan/service/TtsService.java src/test/java/com/sanyan/service/TtsServiceTest.java
git commit -m "feat: TtsService 火山引擎语音合成服务"
```

---

### Task 4: MessageService + MessageData 改动 — 接入 TTS 流程

**Files:**
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java`
- Modify: `server/src/main/java/com/sanyan/dto/data/MessageData.java`
- Test: `server/src/test/java/com/sanyan/service/MessageServiceTtsTest.java`

- [ ] **Step 1: MessageData 添加 mediaUrl 字段**

在 `server/src/main/java/com/sanyan/dto/data/MessageData.java` 中添加：

```java
private String mediaUrl;
```

- [ ] **Step 2: MessageService.toData() 添加 mediaUrl 映射**

在 `server/src/main/java/com/sanyan/service/MessageService.java` 的 `toData()` 方法中添加一行：

```java
d.setMediaUrl(msg.getMediaUrl());
```

- [ ] **Step 3: 写失败的测试**

```java
package com.sanyan.service;

import com.sanyan.util.TextProcessor;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class MessageServiceTtsTest {

    @Test
    void ttsFlow_extractsActionsAndCleansText() {
        String aiReply = "当然可以呀（开心地拍手）我们来聊聊吧！";
        var result = TextProcessor.extract(aiReply);

        assertThat(result.cleanText()).isEqualTo("当然可以呀我们来聊聊吧！");
        assertThat(result.actions()).containsExactly("开心地拍手");
    }

    @Test
    void ttsFlow_noActions_textUnchanged() {
        String aiReply = "好的，我知道了~";
        var result = TextProcessor.extract(aiReply);

        assertThat(result.cleanText()).isEqualTo("好的，我知道了~");
        assertThat(result.actions()).isEmpty();
    }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=MessageServiceTtsTest`
Expected: 2 tests PASS

- [ ] **Step 5: 修改 handleUserMessage() 接入 TTS**

在 `MessageService.java` 中注入 `TtsService` 和 `CosService`，修改 `handleUserMessage()` 方法，在 AI 回复保存前插入 TTS 逻辑：

```java
// 已有代码: String aiReply = aiService.chat(character, conversationId);

// === TTS 逻辑开始 ===
String contentType = "text";
String messageContent = aiReply;
String mediaUrl = null;

if (ttsService.isEnabled()) {
    var extracted = TextProcessor.extract(aiReply);
    messageContent = extracted.cleanText();

    byte[] audioData = ttsService.synthesize(extracted.cleanText(), extracted.actions());
    if (audioData != null) {
        // 先保存消息拿到 ID，再上传 COS
        Message aiMsg = new Message();
        aiMsg.setConversationId(conversationId);
        aiMsg.setSenderType("ai");
        aiMsg.setContentType("voice");
        aiMsg.setContent(messageContent);
        aiMsg.setSource("reply");
        messageRepository.save(aiMsg);

        mediaUrl = cosService.upload(audioData, conversationId, aiMsg.getId());
        aiMsg.setMediaUrl(mediaUrl);
        messageRepository.save(aiMsg);

        // 更新会话、Redis 等后续逻辑...
        conv.setLastMessageAt(LocalDateTime.now());
        conv.setUnreadCount(conv.getUnreadCount() + 1);
        conversationRepository.save(conv);

        String roundKey = "conv:round:" + conversationId;
        String roundTsKey = "conv:round:" + conversationId + ":ts";
        redisTemplate.opsForList().rightPush(roundKey, String.valueOf(userMsg.getId()));
        redisTemplate.opsForList().rightPush(roundKey, String.valueOf(aiMsg.getId()));
        redisTemplate.opsForValue().set(roundTsKey, String.valueOf(System.currentTimeMillis()));

        log.info("语音消息生成完成: convId={}, msgId={}, mediaUrl={}", conversationId, aiMsg.getId(), mediaUrl);
        return aiMsg;
    }
    // audioData 为 null 时降级为文字消息
    log.warn("TTS 合成失败，降级为文字消息: convId={}", conversationId);
}

// 原有文字消息逻辑（TTS 关闭或降级时执行）
Message aiMsg = new Message();
aiMsg.setConversationId(conversationId);
aiMsg.setSenderType("ai");
aiMsg.setContentType(contentType);
aiMsg.setContent(messageContent);
aiMsg.setSource("reply");
messageRepository.save(aiMsg);
// === TTS 逻辑结束 ===
```

- [ ] **Step 6: 运行全量测试**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test`
Expected: 全部 PASS

- [ ] **Step 7: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add src/main/java/com/sanyan/service/MessageService.java src/main/java/com/sanyan/dto/data/MessageData.java src/test/java/com/sanyan/service/MessageServiceTtsTest.java
git commit -m "feat: MessageService 接入 TTS 流程 + MessageData 添加 mediaUrl"
```

---

### Task 5: WebSocketHandler 打字延迟调整

**Files:**
- Modify: `server/src/main/java/com/sanyan/websocket/WebSocketHandler.java`

- [ ] **Step 1: 修改 handleSendMessage 中的延迟逻辑**

在 `WebSocketHandler.java` 的 `handleSendMessage()` 方法中，修改打字延迟计算：

```java
// 原来：
// long delay = messageService.calculateTypingDelay(aiMsg.getContent());

// 改为：语音消息已有 TTS 耗时，减少打字延迟
long delay;
if ("voice".equals(aiMsg.getContentType())) {
    delay = 500; // 语音消息只保留极短延迟
} else {
    delay = messageService.calculateTypingDelay(aiMsg.getContent());
}
```

- [ ] **Step 2: 运行全量测试**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test`
Expected: 全部 PASS

- [ ] **Step 3: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add src/main/java/com/sanyan/websocket/WebSocketHandler.java
git commit -m "feat: 语音消息减少打字延迟"
```

---

### Task 6: 客户端 — Message 模型 + 语音气泡

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/models/message.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart`
- Modify: `app/pubspec.yaml` (添加 audioplayers 依赖)

- [ ] **Step 1: Message 模型添加 mediaUrl**

在 `app/business_packages/sanyan_chat/lib/src/models/message.dart` 中：

```dart
class Message {
  final int id;
  final int conversationId;
  final String senderType;
  final String contentType;
  final String content;
  final String? mediaUrl;  // 新增
  final String source;
  final String createdAt;
  final String? clientMsgId;

  Message({
    required this.id,
    required this.conversationId,
    required this.senderType,
    required this.contentType,
    required this.content,
    this.mediaUrl,            // 新增
    required this.source,
    required this.createdAt,
    this.clientMsgId,
  });

  bool get isFromAi => senderType == 'ai';
  bool get isProactive => source == 'proactive';
  bool get isVoice => contentType == 'voice' && mediaUrl != null;  // 新增

  factory Message.fromJson(Map<String, dynamic> json) => Message(
    id: json['id'] ?? 0,
    conversationId: json['conversationId'] ?? 0,
    senderType: json['senderType'] ?? '',
    contentType: json['contentType'] ?? 'text',
    content: json['content'] ?? '',
    mediaUrl: json['mediaUrl'],    // 新增
    source: json['source'] ?? 'reply',
    createdAt: json['createdAt'] ?? '',
  );

  Map<String, dynamic> toJson() => {
    'id': id,
    'conversationId': conversationId,
    'senderType': senderType,
    'contentType': contentType,
    'content': content,
    'mediaUrl': mediaUrl,           // 新增
    'source': source,
    'createdAt': createdAt,
  };
}
```

- [ ] **Step 2: 添加 audioplayers 依赖**

在 `app/pubspec.yaml` 的 dependencies 中添加：

```yaml
  audioplayers: ^6.1.0
```

Run: `cd /Users/aventador/sourceCode/3yan/app && fvm flutter pub get`

- [ ] **Step 3: 修改 MessageBubble 支持语音类型**

在 `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart` 中：

```dart
import 'package:flutter/material.dart';
import 'package:audioplayers/audioplayers.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';
import '../../models/message.dart';

class MessageBubble extends StatefulWidget {
  final Message message;
  const MessageBubble({super.key, required this.message});

  @override
  State<MessageBubble> createState() => _MessageBubbleState();
}

class _MessageBubbleState extends State<MessageBubble> {
  final AudioPlayer _player = AudioPlayer();
  bool _isPlaying = false;

  @override
  void dispose() {
    _player.dispose();
    super.dispose();
  }

  void _togglePlay() async {
    if (_isPlaying) {
      await _player.stop();
      setState(() => _isPlaying = false);
    } else {
      setState(() => _isPlaying = true);
      await _player.play(UrlSource(widget.message.mediaUrl!));
      _player.onPlayerComplete.listen((_) {
        if (mounted) setState(() => _isPlaying = false);
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    final isUser = widget.message.senderType == 'user';
    final isVoice = widget.message.isVoice && !isUser;

    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      child: Row(
        mainAxisAlignment: isUser ? MainAxisAlignment.end : MainAxisAlignment.start,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          if (!isUser) ...[
            Container(
              width: 36,
              height: 36,
              decoration: const BoxDecoration(
                shape: BoxShape.circle,
                gradient: AppColors.brandGradient,
              ),
              child: const Icon(Icons.favorite, color: Colors.white, size: 18),
            ),
            const SizedBox(width: 10),
          ],
          Flexible(
            child: isVoice ? _buildVoiceBubble() : _buildTextBubble(isUser),
          ),
          if (isUser) const SizedBox(width: 10),
        ],
      ),
    );
  }

  Widget _buildVoiceBubble() {
    return GestureDetector(
      onTap: _togglePlay,
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
        decoration: BoxDecoration(
          color: AppColors.aiBubble,
          borderRadius: const BorderRadius.only(
            topLeft: Radius.circular(4),
            topRight: Radius.circular(16),
            bottomLeft: Radius.circular(16),
            bottomRight: Radius.circular(16),
          ),
        ),
        child: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              _isPlaying ? Icons.pause_circle_filled : Icons.play_circle_filled,
              color: AppColors.brandButton,
              size: 32,
            ),
            const SizedBox(width: 10),
            Icon(
              Icons.graphic_eq,
              color: _isPlaying ? AppColors.brandButton : AppColors.textSecondary,
              size: 20,
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildTextBubble(bool isUser) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 12),
      decoration: BoxDecoration(
        gradient: isUser ? AppColors.buttonGradient : null,
        color: isUser ? null : AppColors.aiBubble,
        borderRadius: BorderRadius.only(
          topLeft: Radius.circular(isUser ? 16 : 4),
          topRight: Radius.circular(isUser ? 4 : 16),
          bottomLeft: const Radius.circular(16),
          bottomRight: const Radius.circular(16),
        ),
      ),
      child: Text(
        widget.message.content,
        style: TextStyle(
          color: isUser ? Colors.white : AppColors.textPrimary,
          fontSize: 15,
          height: 1.5,
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: 验证编译通过**

Run: `cd /Users/aventador/sourceCode/3yan/app && fvm flutter analyze`
Expected: No issues found

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add business_packages/sanyan_chat/lib/src/models/message.dart business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart pubspec.yaml pubspec.lock
git commit -m "feat: 客户端支持语音消息播放"
```

---

### Task 7: 端到端验证 — 开启 TTS 测试

**Files:** 无新增，修改配置验证

- [ ] **Step 1: 开启 TTS**

将 `server/src/main/resources/application-dev.yml` 中 `sanyan.tts.enabled` 改为 `true`。

- [ ] **Step 2: 本地启动后端**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn spring-boot:run`
Expected: 启动日志出现 "COS 客户端初始化完成"

- [ ] **Step 3: 客户端发送消息测试**

在模拟器上发一条消息给小婉，观察：
1. 后端日志出现 "TTS 请求" 和 "TTS 响应"
2. 后端日志出现 "语音上传 COS 成功"
3. 客户端收到语音消息气泡（播放按钮）
4. 点击播放能听到语音

- [ ] **Step 4: 验证降级**

将 `tts.enabled` 改回 `false`，重启后端，发消息确认回到文字模式。

- [ ] **Step 5: 部署到服务器**

```bash
cd /Users/aventador/sourceCode/3yan/server
mvn clean package -DskipTests
scp target/sanyan-server-0.1.0.jar beastify:/opt/sanyan/
ssh beastify "systemctl restart sanyan"
```

- [ ] **Step 6: 提交最终配置**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add src/main/resources/application-dev.yml
git commit -m "feat: TTS 配置完成，默认关闭"
```
