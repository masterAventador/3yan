# 语音消息发送功能实现计划（第一阶段）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现用户在聊天页录音 → 上传到服务器 → 通过 WebSocket 发送语音消息 → 小婉用"没听清"的预置回复响应的完整链路。

**Architecture:** 客户端用 `record` 包录音，录音文件本地缓存 + 乐观 UI + 并发上传。服务端新增 `/api/media/upload` HTTP 接口上传到 COS，扩展 WebSocket `send_message` 支持 voice 类型，AI 回复分支用特殊 prompt 让小婉生成"没听清"的撒娇回复。

**Tech Stack:** Flutter 3.41.6 / GetX / record 5.x / permission_handler / path_provider / Spring Boot 3.2.5 / 腾讯云 COS

---

## 文件结构

### 服务端新增
| 文件 | 职责 |
|------|------|
| `server/src/main/java/com/sanyan/controller/MediaController.java` | 媒体上传 HTTP 接口 |
| `server/src/main/java/com/sanyan/dto/data/MediaUploadData.java` | 上传响应 DTO |
| `server/src/main/java/com/sanyan/service/MediaService.java` | 媒体上传业务逻辑 |
| `server/src/test/java/com/sanyan/service/MediaServiceTest.java` | MediaService 单元测试 |

### 服务端修改
| 文件 | 改动 |
|------|------|
| `server/src/main/java/com/sanyan/service/CosService.java` | 新增通用 upload(bytes, key, contentType) 方法 |
| `server/src/main/java/com/sanyan/dto/ws/WsMessage.java` | 新增 mediaUrl、duration 字段 |
| `server/src/main/java/com/sanyan/service/MessageService.java` | handleUserMessage 支持 mediaUrl、duration，新增语音消息处理分支 |
| `server/src/main/java/com/sanyan/websocket/WebSocketHandler.java` | 传递 mediaUrl 和 duration 给 MessageService |
| `server/src/main/java/com/sanyan/service/AiService.java` | 新增 chatVoiceAck 方法 |

### 客户端新增
| 文件 | 职责 |
|------|------|
| `app/business_packages/sanyan_chat/lib/src/chat/voice_recorder.dart` | 录音管理器（开始/停止/取消） |
| `app/business_packages/sanyan_chat/lib/src/chat/voice_cache_manager.dart` | 本地语音文件缓存清理 |
| `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_record_overlay.dart` | 录音浮层 UI |
| `app/business_packages/sanyan_chat/lib/src/chat/widget/hold_to_speak_button.dart` | 按住说话长条按钮 |
| `app/business_packages/sanyan_chat/lib/src/chat/widget/chat_input_mode.dart` | 输入模式枚举 |
| `app/business_packages/sanyan_chat/lib/src/api/req/upload_voice_req.dart` | 上传语音请求类 |
| `app/business_packages/sanyan_chat/lib/src/models/message_status.dart` | 消息发送状态枚举 |

### 客户端修改
| 文件 | 改动 |
|------|------|
| `app/pubspec.yaml` | 添加 record、permission_handler 依赖 |
| `app/business_packages/sanyan_chat/pubspec.yaml` | 添加依赖引用 |
| `app/foundation_packages/sanyan_user/lib/src/dao/local_storage.dart` | 新增 lastInputMode getter/setter |
| `app/business_packages/sanyan_chat/lib/src/models/message.dart` | 新增 status 和 localFilePath 字段 |
| `app/business_packages/sanyan_chat/lib/src/api/chat_api.dart` | 新增 uploadVoice 方法 |
| `app/foundation_packages/sanyan_network/lib/src/ws_client.dart` | 新增 sendVoiceMessage 方法 |
| `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart` | 新增 sendVoiceMessage、retryVoiceMessage、ACK 处理 |
| `app/business_packages/sanyan_chat/lib/src/chat/widget/chat_input_bar.dart` | 支持模式切换 |
| `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart` | 支持发送中/失败状态 |
| `app/business_packages/sanyan_chat/lib/src/chat/chat_page.dart` | 录音时叠加浮层 |
| `app/lib/main.dart` | App 启动时调用缓存清理 |
| `app/ios/Runner/Info.plist` | 麦克风权限声明 |
| `app/android/app/src/main/AndroidManifest.xml` | 录音权限声明 |

---

### Task 1: 服务端 CosService 泛化

**Files:**
- Modify: `server/src/main/java/com/sanyan/service/CosService.java`
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java` (调用点)
- Modify: `server/src/main/java/com/sanyan/service/ProactiveService.java` (调用点)
- Test: `server/src/test/java/com/sanyan/service/CosServiceTest.java`

- [ ] **Step 1: 写失败的测试**

在 `server/src/test/java/com/sanyan/service/CosServiceTest.java` 中添加新测试：

```java
@Test
void buildObjectKeyForAiVoice_returnsCorrectPath() {
    String key = CosService.buildAiVoiceKey(100L, 200L);
    assertThat(key).isEqualTo("voice/ai/100/200.mp3");
}

@Test
void buildObjectKeyForUserVoice_returnsCorrectPath() {
    String key = CosService.buildUserVoiceKey(1L, "abc123");
    assertThat(key).startsWith("voice/user/1/");
    assertThat(key).endsWith("_abc123.m4a");
}
```

- [ ] **Step 2: 运行测试确认失败**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=CosServiceTest`
Expected: 编译失败，buildAiVoiceKey 和 buildUserVoiceKey 方法不存在

- [ ] **Step 3: 泛化 CosService**

替换 `server/src/main/java/com/sanyan/service/CosService.java` 的上传方法和 key 构造方法：

```java
public String upload(byte[] data, String key, String contentType) {
    try {
        var metadata = new ObjectMetadata();
        metadata.setContentLength(data.length);
        metadata.setContentType(contentType);
        cosClient.putObject(bucket, key, new ByteArrayInputStream(data), metadata);
        String url = buildCosUrl(bucket, region, key);
        log.info("COS 上传成功: key={}, size={}bytes, contentType={}", key, data.length, contentType);
        return url;
    } catch (Exception e) {
        log.error("COS 上传失败: key={}", key, e);
        throw new RuntimeException("文件上传失败", e);
    }
}

/** AI 合成语音的 key */
public static String buildAiVoiceKey(Long conversationId, Long messageId) {
    return "voice/ai/" + conversationId + "/" + messageId + ".mp3";
}

/** 用户上传语音的 key */
public static String buildUserVoiceKey(Long userId, String uuid) {
    return "voice/user/" + userId + "/" + System.currentTimeMillis() + "_" + uuid + ".m4a";
}

public static String buildCosUrl(String bucket, String region, String key) {
    return "https://" + bucket + ".cos." + region + ".myqcloud.com/" + key;
}
```

删除旧的 `upload(byte[], Long, Long)` 方法和 `buildObjectKey(Long, Long)` 方法。

- [ ] **Step 4: 更新 MessageService 的调用点**

在 `server/src/main/java/com/sanyan/service/MessageService.java` 中找到原来 `cosService.upload(audioData, conversationId, aiMsg.getId())` 的调用，改为：

```java
String cosKey = CosService.buildAiVoiceKey(conversationId, aiMsg.getId());
String mediaUrl = cosService.upload(audioData, cosKey, "audio/mpeg");
```

- [ ] **Step 5: 更新 ProactiveService 的调用点**

在 `server/src/main/java/com/sanyan/service/ProactiveService.java` 里同样找到 `cosService.upload(audioData, conversationId, proactiveMsg.getId())`，改为：

```java
String cosKey = CosService.buildAiVoiceKey(conversationId, proactiveMsg.getId());
String mediaUrl = cosService.upload(audioData, cosKey, "audio/mpeg");
```

- [ ] **Step 6: 运行测试确认通过**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test`
Expected: 全部 PASS

- [ ] **Step 7: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add -A
git commit -m "refactor: CosService 泛化上传方法，支持多种媒体类型"
```

---

### Task 2: 服务端 MediaService + MediaController

**Files:**
- Create: `server/src/main/java/com/sanyan/dto/data/MediaUploadData.java`
- Create: `server/src/main/java/com/sanyan/service/MediaService.java`
- Create: `server/src/main/java/com/sanyan/service/MediaServiceTest.java`
- Create: `server/src/main/java/com/sanyan/controller/MediaController.java`

- [ ] **Step 1: 写 MediaUploadData DTO**

创建 `server/src/main/java/com/sanyan/dto/data/MediaUploadData.java`：

```java
package com.sanyan.dto.data;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class MediaUploadData {
    private String url;
    private Integer duration;
}
```

- [ ] **Step 2: 写 MediaService 的失败测试**

创建 `server/src/test/java/com/sanyan/service/MediaServiceTest.java`：

```java
package com.sanyan.service;

import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.mock.web.MockMultipartFile;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class MediaServiceTest {

    @Test
    void uploadVoice_rejectsFileTooLarge() {
        CosService mockCos = Mockito.mock(CosService.class);
        MediaService service = new MediaService(mockCos);
        byte[] bigFile = new byte[6 * 1024 * 1024]; // 6MB
        MockMultipartFile file = new MockMultipartFile("file", "test.m4a", "audio/mp4", bigFile);

        assertThatThrownBy(() -> service.uploadVoice(1L, file, 10))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("文件过大");
    }

    @Test
    void uploadVoice_rejectsDurationOutOfRange() {
        CosService mockCos = Mockito.mock(CosService.class);
        MediaService service = new MediaService(mockCos);
        MockMultipartFile file = new MockMultipartFile("file", "test.m4a", "audio/mp4", new byte[100]);

        assertThatThrownBy(() -> service.uploadVoice(1L, file, 0))
                .isInstanceOf(IllegalArgumentException.class);
        assertThatThrownBy(() -> service.uploadVoice(1L, file, 61))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void uploadVoice_callsCosAndReturnsData() throws Exception {
        CosService mockCos = Mockito.mock(CosService.class);
        Mockito.when(mockCos.upload(Mockito.any(), Mockito.anyString(), Mockito.eq("audio/mp4")))
                .thenReturn("https://3yan-1258800826.cos.ap-beijing.myqcloud.com/voice/user/1/xxx.m4a");

        MediaService service = new MediaService(mockCos);
        MockMultipartFile file = new MockMultipartFile("file", "test.m4a", "audio/mp4", new byte[100]);

        var result = service.uploadVoice(1L, file, 12);

        assertThat(result.getUrl()).contains("voice/user/1/");
        assertThat(result.getDuration()).isEqualTo(12);
    }
}
```

- [ ] **Step 3: 运行测试确认失败**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=MediaServiceTest`
Expected: 编译失败，MediaService 类不存在

- [ ] **Step 4: 写 MediaService**

创建 `server/src/main/java/com/sanyan/service/MediaService.java`：

```java
package com.sanyan.service;

import com.sanyan.dto.data.MediaUploadData;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class MediaService {

    private static final long MAX_VOICE_SIZE = 5 * 1024 * 1024; // 5MB
    private static final int MIN_DURATION = 1;
    private static final int MAX_DURATION = 60;

    private final CosService cosService;

    public MediaUploadData uploadVoice(Long userId, MultipartFile file, Integer duration) {
        // 校验
        if (file == null || file.isEmpty()) {
            throw new IllegalArgumentException("文件不能为空");
        }
        if (file.getSize() > MAX_VOICE_SIZE) {
            throw new IllegalArgumentException("文件过大，最大 5MB");
        }
        if (duration == null || duration < MIN_DURATION || duration > MAX_DURATION) {
            throw new IllegalArgumentException("时长必须在 " + MIN_DURATION + "-" + MAX_DURATION + " 秒之间");
        }

        // 上传
        String uuid = UUID.randomUUID().toString().substring(0, 8);
        String key = CosService.buildUserVoiceKey(userId, uuid);
        try {
            String url = cosService.upload(file.getBytes(), key, "audio/mp4");
            log.info("用户语音上传成功: userId={}, key={}, duration={}s", userId, key, duration);
            return new MediaUploadData(url, duration);
        } catch (Exception e) {
            log.error("用户语音上传失败: userId={}", userId, e);
            throw new RuntimeException("语音上传失败", e);
        }
    }
}
```

- [ ] **Step 5: 写 MediaController**

创建 `server/src/main/java/com/sanyan/controller/MediaController.java`：

```java
package com.sanyan.controller;

import com.sanyan.dto.ApiResponse;
import com.sanyan.dto.data.MediaUploadData;
import com.sanyan.service.MediaService;
import com.sanyan.util.JwtUtil;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

@Slf4j
@RestController
@RequestMapping("/api/media")
@RequiredArgsConstructor
public class MediaController {

    private final MediaService mediaService;
    private final JwtUtil jwtUtil;

    @PostMapping("/upload")
    public ApiResponse<MediaUploadData> upload(
            @RequestHeader("Authorization") String authHeader,
            @RequestParam("file") MultipartFile file,
            @RequestParam("type") String type,
            @RequestParam("duration") Integer duration) {

        Long userId = jwtUtil.extractUserId(authHeader.replace("Bearer ", ""));
        if (userId == null) {
            return ApiResponse.fail("未授权");
        }

        if (!"voice".equals(type)) {
            return ApiResponse.fail("不支持的媒体类型: " + type);
        }

        try {
            MediaUploadData data = mediaService.uploadVoice(userId, file, duration);
            return ApiResponse.ok(data);
        } catch (IllegalArgumentException e) {
            return ApiResponse.fail(e.getMessage());
        } catch (Exception e) {
            log.error("媒体上传失败", e);
            return ApiResponse.fail("上传失败");
        }
    }
}
```

- [ ] **Step 6: 运行测试确认通过**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test -pl . -Dtest=MediaServiceTest`
Expected: 3 tests PASS

- [ ] **Step 7: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add -A
git commit -m "feat: MediaController + MediaService 语音上传接口"
```

---

### Task 3: 服务端 WebSocket 协议扩展 + MessageService 支持语音

**Files:**
- Modify: `server/src/main/java/com/sanyan/dto/ws/WsMessage.java`
- Modify: `server/src/main/java/com/sanyan/service/MessageService.java`
- Modify: `server/src/main/java/com/sanyan/websocket/WebSocketHandler.java`
- Modify: `server/src/main/java/com/sanyan/service/AiService.java`

- [ ] **Step 1: 扩展 WsMessage**

修改 `server/src/main/java/com/sanyan/dto/ws/WsMessage.java`：

```java
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
    private String mediaUrl;   // 语音/视频等消息的媒体 URL
    private Integer duration;  // 语音/视频时长（秒）
    private String clientMsgId;
    // sync fields
    private Long lastMsgId;
}
```

- [ ] **Step 2: AiService 新增 chatVoiceAck 方法**

在 `server/src/main/java/com/sanyan/service/AiService.java` 中添加：

```java
/**
 * 用户发了语音但当前不支持 ASR，让 AI 生成"没听清"的撒娇回复。
 */
public String chatVoiceAck(AiCharacter character, Long conversationId) {
    String time = formatCurrentTime();
    String profile = getProfile(conversationId);
    String basePrompt = assembleSystemPrompt(character.getSystemPrompt(), time, profile);
    String voicePrompt = basePrompt
        + "\n\n[特殊场景] 用户刚才发了一条语音消息，但你现在还无法听懂语音内容。"
        + "请用符合你人设的方式回复，表达你听到了但没听清，让用户再说一遍或者打字告诉你。"
        + "要自然、多变、撒娇，每次都不一样。记得按规则输出语音情感标签。";

    List<MemorySummary> summaries = memorySummaryRepository
            .findByConversationIdOrderByCreatedAtDesc(conversationId, PageRequest.of(0, 5));
    List<Message> recentMessages = messageRepository
            .findByConversationIdOrderByIdDesc(conversationId, PageRequest.of(0, 20));
    Collections.reverse(recentMessages);

    return callDoubao(voicePrompt, summaries, recentMessages);
}
```

- [ ] **Step 3: 修改 MessageService.handleUserMessage 签名**

在 `server/src/main/java/com/sanyan/service/MessageService.java` 里：

原来的签名：
```java
public Message handleUserMessage(Long userId, Long conversationId, String contentType, String content) {
```

改为：
```java
public Message handleUserMessage(Long userId, Long conversationId, String contentType, String content,
                                  String mediaUrl, Integer duration) {
```

在保存用户消息的位置，原来：
```java
Message userMsg = new Message();
userMsg.setConversationId(conversationId);
userMsg.setSenderType("user");
userMsg.setContentType(contentType);
userMsg.setContent(content);
userMsg.setSource("reply");
messageRepository.save(userMsg);
```

改为：
```java
Message userMsg = new Message();
userMsg.setConversationId(conversationId);
userMsg.setSenderType("user");
userMsg.setContentType(contentType);
userMsg.setContent(content == null ? "" : content);
userMsg.setMediaUrl(mediaUrl);
userMsg.setDuration(duration);
userMsg.setSource("reply");
messageRepository.save(userMsg);
```

找到调用 `aiService.chat(character, conversationId)` 的地方，改为根据 contentType 分支：

```java
log.info("调用豆包 AI: convId={}, character={}, contentType={}",
        conversationId, character.getName(), contentType);
String aiReply;
if ("voice".equals(contentType)) {
    aiReply = aiService.chatVoiceAck(character, conversationId);
} else {
    aiReply = aiService.chat(character, conversationId);
}
log.info("豆包 AI 回复: convId={}, replyLength={}", conversationId, aiReply.length());
```

- [ ] **Step 4: Message 实体增加 duration 字段**

在 `server/src/main/java/com/sanyan/entity/Message.java` 中新增字段（如果还没有）：

```java
@Column
private Integer duration; // 语音/视频时长（秒）
```

- [ ] **Step 5: MessageData DTO 增加 duration**

在 `server/src/main/java/com/sanyan/dto/data/MessageData.java` 中新增：

```java
private Integer duration;
```

在 `MessageService.toData()` 中添加：

```java
d.setDuration(msg.getDuration());
```

- [ ] **Step 6: WebSocketHandler 传递新字段**

在 `server/src/main/java/com/sanyan/websocket/WebSocketHandler.java` 的 `handleSendMessage` 方法里，找到 `messageService.handleUserMessage(...)` 的调用：

原来：
```java
Message aiMsg = messageService.handleUserMessage(
        userId, wsMsg.getConversationId(), wsMsg.getContentType(), wsMsg.getContent());
```

改为：
```java
Message aiMsg = messageService.handleUserMessage(
        userId, wsMsg.getConversationId(), wsMsg.getContentType(), wsMsg.getContent(),
        wsMsg.getMediaUrl(), wsMsg.getDuration());
```

- [ ] **Step 7: 编译通过 + 全量测试**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn test`
Expected: 全部 PASS

- [ ] **Step 8: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/server
git add -A
git commit -m "feat: WebSocket 和 MessageService 支持语音消息 + AI chatVoiceAck"
```

---

### Task 4: 服务端部署并验证 HTTP 接口

**Files:** 无代码修改

- [ ] **Step 1: 打包**

Run: `cd /Users/aventador/sourceCode/3yan/server && mvn clean package -DskipTests -q`

- [ ] **Step 2: 上传到服务器**

Run: `scp target/sanyan-server-0.1.0.jar beastify:/opt/sanyan/`

- [ ] **Step 3: 重启服务**

Run: `ssh beastify "systemctl restart sanyan"`

- [ ] **Step 4: 检查启动日志**

Run: `ssh beastify "sleep 5 && journalctl -u sanyan --since '30 sec ago' --no-pager" | tail -20`
Expected: 看到 "Started SanyanApplication"

- [ ] **Step 5: 推送代码**

```bash
cd /Users/aventador/sourceCode/3yan/server
git push origin dev
```

---

### Task 5: 客户端依赖 + 权限配置

**Files:**
- Modify: `app/pubspec.yaml`
- Modify: `app/business_packages/sanyan_chat/pubspec.yaml`
- Modify: `app/ios/Runner/Info.plist`
- Modify: `app/android/app/src/main/AndroidManifest.xml`

- [ ] **Step 1: 添加 Flutter 依赖**

修改 `app/pubspec.yaml` 的 dependencies，添加：

```yaml
  record: ^5.1.2
  permission_handler: ^11.3.1
  path_provider: ^2.1.3
```

修改 `app/business_packages/sanyan_chat/pubspec.yaml`，添加：

```yaml
  record: ^5.1.2
  permission_handler: ^11.3.1
  path_provider: ^2.1.3
```

- [ ] **Step 2: 拉取依赖**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter pub get`

- [ ] **Step 3: iOS 权限配置**

在 `app/ios/Runner/Info.plist` 中添加（如果不存在）：

```xml
<key>NSMicrophoneUsageDescription</key>
<string>三言需要使用麦克风来录制语音消息</string>
```

- [ ] **Step 4: Android 权限配置**

在 `app/android/app/src/main/AndroidManifest.xml` 的 `<manifest>` 标签下添加：

```xml
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```

- [ ] **Step 5: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 6: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "chore: 添加录音相关依赖和权限配置"
```

---

### Task 6: LocalStorage + Message 模型扩展

**Files:**
- Modify: `app/foundation_packages/sanyan_user/lib/src/dao/local_storage.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/models/message_status.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/models/message.dart`

- [ ] **Step 1: 新增 MessageStatus 枚举**

创建 `app/business_packages/sanyan_chat/lib/src/models/message_status.dart`：

```dart
enum MessageStatus {
  sending,   // 正在上传/发送
  sent,      // 发送成功
  failed,    // 发送失败
}
```

- [ ] **Step 2: Message 模型新增字段**

修改 `app/business_packages/sanyan_chat/lib/src/models/message.dart`，替换整个 class：

```dart
import 'package:sanyan_network/sanyan_network.dart';
import 'message_status.dart';

class Message {
  final int id;
  final int conversationId;
  final String senderType;
  final String contentType;
  final String content;
  final String? mediaUrl;
  final int? duration; // 语音时长（秒）
  final String source;
  final String createdAt;
  final String? clientMsgId;
  MessageStatus? status;       // 仅本地发送中/失败的消息使用
  String? localFilePath;       // 仅本地发送中/失败的消息使用

  Message({
    required this.id,
    required this.conversationId,
    required this.senderType,
    required this.contentType,
    required this.content,
    this.mediaUrl,
    this.duration,
    required this.source,
    required this.createdAt,
    this.clientMsgId,
    this.status,
    this.localFilePath,
  });

  bool get isFromAi => senderType == 'ai';
  bool get isProactive => source == 'proactive';
  bool get isVoice => contentType == ContentType.voice && mediaUrl != null;
  bool get isSending => status == MessageStatus.sending;
  bool get isFailed => status == MessageStatus.failed;

  factory Message.fromJson(Map<String, dynamic> json) => Message(
    id: json['id'] ?? 0,
    conversationId: json['conversationId'] ?? 0,
    senderType: json['senderType'] ?? '',
    contentType: json['contentType'] ?? 'text',
    content: json['content'] ?? '',
    mediaUrl: json['mediaUrl'],
    duration: json['duration'],
    source: json['source'] ?? 'reply',
    createdAt: json['createdAt'] ?? '',
  );

  Map<String, dynamic> toJson() => {
    'id': id,
    'conversationId': conversationId,
    'senderType': senderType,
    'contentType': contentType,
    'content': content,
    'mediaUrl': mediaUrl,
    'duration': duration,
    'source': source,
    'createdAt': createdAt,
  };
}
```

- [ ] **Step 3: LocalStorage 新增 lastInputMode**

修改 `app/foundation_packages/sanyan_user/lib/src/dao/local_storage.dart`，在 clear 方法前添加：

```dart
static String? get lastInputMode => _box.read('lastInputMode');
static set lastInputMode(String? value) {
  if (value == null) {
    _box.remove('lastInputMode');
  } else {
    _box.write('lastInputMode', value);
  }
}
```

- [ ] **Step 4: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: Message 模型扩展 status/localFilePath + MessageStatus 枚举"
```

---

### Task 7: 客户端 VoiceRecorder 和 VoiceCacheManager

**Files:**
- Create: `app/business_packages/sanyan_chat/lib/src/chat/voice_recorder.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/chat/voice_cache_manager.dart`
- Modify: `app/lib/main.dart`

- [ ] **Step 1: 写 VoiceCacheManager**

创建 `app/business_packages/sanyan_chat/lib/src/chat/voice_cache_manager.dart`：

```dart
import 'dart:io';
import 'package:path_provider/path_provider.dart';

class VoiceCacheManager {
  static const _dirName = 'sanyan_voice';
  static const _maxAgeMs = 7 * 24 * 60 * 60 * 1000; // 7 天

  /// 获取语音缓存目录，自动创建
  static Future<Directory> getCacheDir() async {
    final cacheDir = await getApplicationCacheDirectory();
    final voiceDir = Directory('${cacheDir.path}/$_dirName');
    if (!await voiceDir.exists()) {
      await voiceDir.create(recursive: true);
    }
    return voiceDir;
  }

  /// 生成一个新的语音文件路径
  static Future<String> newVoiceFilePath(String uuid) async {
    final dir = await getCacheDir();
    return '${dir.path}/$uuid.m4a';
  }

  /// 删除单个文件
  static Future<void> deleteFile(String path) async {
    try {
      final file = File(path);
      if (await file.exists()) {
        await file.delete();
      }
    } catch (_) {}
  }

  /// 清理超过 7 天的旧文件（App 启动时调用）
  static Future<void> cleanupOldFiles() async {
    try {
      final dir = await getCacheDir();
      final now = DateTime.now().millisecondsSinceEpoch;
      await for (final entity in dir.list()) {
        if (entity is File) {
          final stat = await entity.stat();
          final ageMs = now - stat.modified.millisecondsSinceEpoch;
          if (ageMs > _maxAgeMs) {
            await entity.delete();
          }
        }
      }
    } catch (_) {}
  }
}
```

- [ ] **Step 2: 写 VoiceRecorder**

创建 `app/business_packages/sanyan_chat/lib/src/chat/voice_recorder.dart`：

```dart
import 'dart:async';
import 'package:record/record.dart';
import 'package:uuid/uuid.dart';
import 'package:permission_handler/permission_handler.dart';
import 'voice_cache_manager.dart';

class RecordingResult {
  final String filePath;
  final int durationSeconds;
  RecordingResult(this.filePath, this.durationSeconds);
}

class VoiceRecorder {
  static const _uuid = Uuid();
  static const maxDurationSeconds = 60;
  static const minDurationSeconds = 1;

  final _recorder = AudioRecorder();
  DateTime? _startTime;
  String? _currentFilePath;
  Timer? _maxDurationTimer;
  void Function()? _onMaxDurationReached;

  /// 检查并请求麦克风权限
  /// 返回 true 表示已授权
  Future<bool> ensurePermission() async {
    final status = await Permission.microphone.request();
    return status.isGranted;
  }

  /// 开始录音，返回 false 表示权限被拒或硬件失败
  Future<bool> start({void Function()? onMaxDurationReached}) async {
    if (!await ensurePermission()) return false;

    try {
      final uuid = _uuid.v4();
      _currentFilePath = await VoiceCacheManager.newVoiceFilePath(uuid);

      await _recorder.start(
        const RecordConfig(
          encoder: AudioEncoder.aacLc,
          sampleRate: 24000,
          numChannels: 1,
        ),
        path: _currentFilePath!,
      );

      _startTime = DateTime.now();
      _onMaxDurationReached = onMaxDurationReached;
      _maxDurationTimer = Timer(const Duration(seconds: maxDurationSeconds), () {
        _onMaxDurationReached?.call();
      });
      return true;
    } catch (_) {
      return false;
    }
  }

  /// 停止录音并返回结果，如果时长过短返回 null 并删除文件
  Future<RecordingResult?> stop() async {
    _maxDurationTimer?.cancel();
    try {
      final path = await _recorder.stop();
      if (path == null || _startTime == null) return null;

      final duration = DateTime.now().difference(_startTime!).inSeconds;
      _startTime = null;
      _currentFilePath = null;

      if (duration < minDurationSeconds) {
        await VoiceCacheManager.deleteFile(path);
        return null;
      }

      final clampedDuration = duration > maxDurationSeconds ? maxDurationSeconds : duration;
      return RecordingResult(path, clampedDuration);
    } catch (_) {
      return null;
    }
  }

  /// 取消录音，删除本地文件
  Future<void> cancel() async {
    _maxDurationTimer?.cancel();
    try {
      final path = await _recorder.stop();
      if (path != null) {
        await VoiceCacheManager.deleteFile(path);
      }
    } catch (_) {}
    _startTime = null;
    _currentFilePath = null;
  }

  /// 实时获取当前录音时长（秒）
  int get currentDurationSeconds {
    if (_startTime == null) return 0;
    return DateTime.now().difference(_startTime!).inSeconds;
  }

  void dispose() {
    _maxDurationTimer?.cancel();
    _recorder.dispose();
  }
}
```

- [ ] **Step 3: main.dart 启动时清理缓存**

修改 `app/lib/main.dart`，在 `main()` 函数中添加：

```dart
import 'package:sanyan_chat/sanyan_chat.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await LocalStorage.init();

  ApiClient.tokenProvider = () => LocalStorage.token;
  WsClient.tokenProvider = () => LocalStorage.token;

  // 清理 7 天前的语音缓存文件
  unawaited(VoiceCacheManager.cleanupOldFiles());

  runApp(const SanyanApp());
}
```

需要在 `app/business_packages/sanyan_chat/lib/sanyan_chat.dart` 中导出：

```dart
export 'src/chat/voice_cache_manager.dart';
export 'src/chat/voice_recorder.dart';
```

- [ ] **Step 4: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found（可能有 unawaited 相关警告，可忽略或导入 `dart:async`）

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: VoiceRecorder + VoiceCacheManager 录音管理器"
```

---

### Task 8: 客户端 ChatApi.uploadVoice + WsClient.sendVoiceMessage

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/api/chat_api.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/api/req/upload_voice_req.dart` (如果要用 BaseReq，否则跳过，直接在 chat_api 写)
- Modify: `app/foundation_packages/sanyan_network/lib/src/ws_client.dart`

- [ ] **Step 1: ChatApi 新增 uploadVoice 方法**

在 `app/business_packages/sanyan_chat/lib/src/api/chat_api.dart` 中添加（使用 Dio 的 MultipartFile）：

```dart
import 'package:dio/dio.dart';

// ... 已有 import 和类定义保持不变 ...

class ChatApi {
  // ... 已有方法 ...

  /// 上传语音文件到服务器，返回 COS URL 和 duration
  static Future<ApiResponse<VoiceUploadResult>> uploadVoice(
    String localFilePath, {
    required int duration,
  }) async {
    try {
      final formData = FormData.fromMap({
        'file': await MultipartFile.fromFile(localFilePath, filename: 'voice.m4a'),
        'type': 'voice',
        'duration': duration,
      });

      final resp = await ApiClient().postFormData(
        '/api/media/upload',
        formData: formData,
      );
      return ApiResponse.fromJson(
        resp,
        (data) => VoiceUploadResult.fromJson(data as Map<String, dynamic>),
      );
    } catch (e) {
      return ApiResponse.fail('上传失败: $e');
    }
  }
}

class VoiceUploadResult {
  final String url;
  final int duration;
  VoiceUploadResult({required this.url, required this.duration});

  factory VoiceUploadResult.fromJson(Map<String, dynamic> json) => VoiceUploadResult(
    url: json['url'] ?? '',
    duration: json['duration'] ?? 0,
  );
}
```

- [ ] **Step 2: ApiClient 添加 postFormData 方法**

在 `app/foundation_packages/sanyan_network/lib/src/api_client.dart` 中添加：

```dart
Future<dynamic> postFormData(String path, {required FormData formData}) async {
  final resp = await _dio.post(path, data: formData);
  return resp.data;
}
```

- [ ] **Step 3: WsClient 新增 sendVoiceMessage 方法**

在 `app/foundation_packages/sanyan_network/lib/src/ws_client.dart` 中，在 `sendMessage` 方法下面添加：

```dart
String sendVoiceMessage({
  required int conversationId,
  required String mediaUrl,
  required int duration,
  String? clientMsgId,
}) {
  final msgId = clientMsgId ?? _uuid.v4();
  _send({
    'type': WsEventType.sendMessage,
    'conversationId': conversationId,
    'contentType': ct.ContentType.voice,
    'content': '',
    'mediaUrl': mediaUrl,
    'duration': duration,
    'clientMsgId': msgId,
  });
  return msgId;
}
```

- [ ] **Step 4: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: ChatApi.uploadVoice + WsClient.sendVoiceMessage"
```

---

### Task 9: 客户端 ChatController 语音发送逻辑

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`

- [ ] **Step 1: 扩展 ChatController**

在 `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart` 中添加方法和修改 `_listenWs`。完整改动：

**在 import 区添加**：
```dart
import 'package:uuid/uuid.dart';
import 'voice_cache_manager.dart';
import '../models/message_status.dart';
```

**在类里添加**：
```dart
static const _uuid = Uuid();

/// 发送语音消息（乐观 UI + 并发上传）
Future<void> sendVoiceMessage(String localPath, int duration) async {
  final clientMsgId = _uuid.v4();

  final msg = Message(
    id: 0,
    conversationId: conversation.id,
    senderType: 'user',
    contentType: ContentType.voice,
    content: '',
    mediaUrl: null,
    duration: duration,
    source: 'reply',
    createdAt: DateTime.now().toString(),
    clientMsgId: clientMsgId,
    status: MessageStatus.sending,
    localFilePath: localPath,
  );
  messages.add(msg);
  _scrollToBottom();

  // 独立 Future 并发，不 await
  _uploadAndSendVoice(msg);
}

/// 重试发送失败的语音消息
Future<void> retryVoiceMessage(Message msg) async {
  if (msg.localFilePath == null) return;
  msg.status = MessageStatus.sending;
  messages.refresh();
  await _uploadAndSendVoice(msg);
}

Future<void> _uploadAndSendVoice(Message msg) async {
  try {
    final uploadResp = await ChatApi.uploadVoice(
      msg.localFilePath!,
      duration: msg.duration ?? 0,
    );
    if (!uploadResp.success || uploadResp.data == null) {
      _markFailed(msg);
      return;
    }

    final wsClient = Get.find<WsClient>();
    wsClient.sendVoiceMessage(
      conversationId: conversation.id,
      mediaUrl: uploadResp.data!.url,
      duration: uploadResp.data!.duration,
      clientMsgId: msg.clientMsgId,
    );
    // ACK 到达后由 _listenWs 更新 status 为 sent 并删除本地文件
  } catch (_) {
    _markFailed(msg);
  }
}

void _markFailed(Message msg) {
  msg.status = MessageStatus.failed;
  messages.refresh();
}
```

**修改 `_listenWs` 的 ack 分支**：

原来：
```dart
case WsEventType.ack:
  break;
```

改为：
```dart
case WsEventType.ack:
  _onAck(event.clientMsgId);
  break;
```

**在类里添加 `_onAck` 方法**：
```dart
void _onAck(String? clientMsgId) {
  if (clientMsgId == null) return;
  final idx = messages.indexWhere((m) => m.clientMsgId == clientMsgId);
  if (idx == -1) return;
  final msg = messages[idx];
  if (msg.status != MessageStatus.sending) return;

  msg.status = MessageStatus.sent;
  messages.refresh();

  // 删除本地缓存文件（语音消息）
  if (msg.localFilePath != null) {
    VoiceCacheManager.deleteFile(msg.localFilePath!);
  }
}
```

- [ ] **Step 2: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 3: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: ChatController 语音发送 + ACK 状态更新"
```

---

### Task 10: 客户端录音浮层 UI

**Files:**
- Create: `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_record_overlay.dart`

- [ ] **Step 1: 写 VoiceRecordOverlay Widget**

创建 `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_record_overlay.dart`：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';

class VoiceRecordOverlay extends StatefulWidget {
  final bool isCancelling; // 是否处于"上滑取消"状态

  const VoiceRecordOverlay({super.key, required this.isCancelling});

  @override
  State<VoiceRecordOverlay> createState() => _VoiceRecordOverlayState();
}

class _VoiceRecordOverlayState extends State<VoiceRecordOverlay>
    with TickerProviderStateMixin {
  late final AnimationController _pulseController;

  @override
  void initState() {
    super.initState();
    _pulseController = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    )..repeat(reverse: true);
  }

  @override
  void dispose() {
    _pulseController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Positioned.fill(
      child: BackdropFilter(
        filter: ImageFilter.blur(sigmaX: 24, sigmaY: 24),
        child: Container(
          decoration: BoxDecoration(
            gradient: RadialGradient(
              colors: [
                AuraColors.surfaceDim.withValues(alpha: 0.4),
                AuraColors.surface.withValues(alpha: 0.8),
              ],
            ),
          ),
          child: SafeArea(
            child: Padding(
              padding: const EdgeInsets.symmetric(horizontal: 32, vertical: 96),
              child: Column(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  _buildWaveform(),
                  _buildMicCircle(),
                  _buildHint(),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }

  Widget _buildWaveform() {
    return SizedBox(
      height: 64,
      child: Row(
        mainAxisAlignment: MainAxisAlignment.center,
        crossAxisAlignment: CrossAxisAlignment.center,
        children: List.generate(7, (i) {
          final delays = [0.1, 0.3, 0.5, 0.2, 0.4, 0.6, 0.1];
          final colors = [
            AuraColors.primary.withValues(alpha: 0.4),
            AuraColors.primary.withValues(alpha: 0.6),
            AuraColors.primary,
            AuraColors.secondary,
            AuraColors.primary,
            AuraColors.primary.withValues(alpha: 0.6),
            AuraColors.primary.withValues(alpha: 0.4),
          ];
          return Padding(
            padding: const EdgeInsets.symmetric(horizontal: 3),
            child: _AnimatedBar(
              delay: delays[i],
              color: colors[i],
            ),
          );
        }),
      ),
    );
  }

  Widget _buildMicCircle() {
    final color = widget.isCancelling
        ? Colors.red
        : null; // null 时用渐变
    return Stack(
      alignment: Alignment.center,
      children: [
        // 光晕
        AnimatedBuilder(
          animation: _pulseController,
          builder: (context, child) {
            return Container(
              width: 180,
              height: 180,
              decoration: BoxDecoration(
                shape: BoxShape.circle,
                color: (widget.isCancelling
                        ? Colors.red
                        : AuraColors.primary)
                    .withValues(alpha: 0.2 * _pulseController.value),
              ),
            );
          },
        ),
        // 主圆
        Container(
          width: 132,
          height: 132,
          decoration: BoxDecoration(
            shape: BoxShape.circle,
            color: color,
            gradient: color == null
                ? const LinearGradient(
                    begin: Alignment.topLeft,
                    end: Alignment.bottomRight,
                    colors: [AuraColors.primary, AuraColors.secondary],
                  )
                : null,
            boxShadow: [
              BoxShadow(
                color: Colors.black.withValues(alpha: 0.15),
                blurRadius: 30,
                offset: const Offset(0, 10),
              ),
            ],
          ),
          child: Icon(
            widget.isCancelling ? Icons.close : Icons.mic,
            color: Colors.white,
            size: 60,
          ),
        ),
      ],
    );
  }

  Widget _buildHint() {
    return Column(
      children: [
        Text(
          widget.isCancelling ? 'Release to cancel' : 'Release to send, Swipe up to cancel',
          style: GoogleFonts.manrope(
            fontSize: 16,
            fontWeight: FontWeight.w700,
            color: widget.isCancelling ? Colors.red : AuraColors.onSurface,
          ),
        ),
        const SizedBox(height: 4),
        Text(
          widget.isCancelling ? '松开取消' : '松开发送，上滑取消',
          style: GoogleFonts.inter(
            fontSize: 12,
            color: AuraColors.onSurfaceVariant.withValues(alpha: 0.6),
            letterSpacing: 2,
          ),
        ),
      ],
    );
  }
}

class _AnimatedBar extends StatefulWidget {
  final double delay;
  final Color color;
  const _AnimatedBar({required this.delay, required this.color});

  @override
  State<_AnimatedBar> createState() => _AnimatedBarState();
}

class _AnimatedBarState extends State<_AnimatedBar> with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1200),
    );
    _animation = Tween<double>(begin: 0.2, end: 1.0).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
    Future.delayed(Duration(milliseconds: (widget.delay * 1000).toInt()), () {
      if (mounted) _controller.repeat(reverse: true);
    });
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return Container(
          width: 3,
          height: 64 * _animation.value,
          decoration: BoxDecoration(
            color: widget.color,
            borderRadius: BorderRadius.circular(2),
          ),
        );
      },
    );
  }
}
```

- [ ] **Step 2: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 3: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: VoiceRecordOverlay 录音浮层 UI"
```

---

### Task 11: 客户端 HoldToSpeakButton + ChatInputBar 模式切换

**Files:**
- Create: `app/business_packages/sanyan_chat/lib/src/chat/widget/chat_input_mode.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/chat/widget/hold_to_speak_button.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/widget/chat_input_bar.dart`

- [ ] **Step 1: 创建 ChatInputMode 枚举**

创建 `app/business_packages/sanyan_chat/lib/src/chat/widget/chat_input_mode.dart`：

```dart
enum ChatInputMode {
  keyboard,
  voice;

  String get storageValue => name;

  static ChatInputMode fromStorage(String? value) {
    if (value == ChatInputMode.voice.name) return ChatInputMode.voice;
    return ChatInputMode.keyboard;
  }
}
```

- [ ] **Step 2: 创建 HoldToSpeakButton**

创建 `app/business_packages/sanyan_chat/lib/src/chat/widget/hold_to_speak_button.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';

class HoldToSpeakButton extends StatelessWidget {
  final bool isPressed;
  final void Function(Offset globalPosition) onPressStart;
  final void Function(Offset globalPosition) onPressMove;
  final VoidCallback onPressEnd;
  final VoidCallback onPressCancel;

  const HoldToSpeakButton({
    super.key,
    required this.isPressed,
    required this.onPressStart,
    required this.onPressMove,
    required this.onPressEnd,
    required this.onPressCancel,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      behavior: HitTestBehavior.opaque,
      onLongPressStart: (details) => onPressStart(details.globalPosition),
      onLongPressMoveUpdate: (details) => onPressMove(details.globalPosition),
      onLongPressEnd: (_) => onPressEnd(),
      onLongPressCancel: onPressCancel,
      child: Container(
        height: 40,
        decoration: BoxDecoration(
          color: isPressed
              ? AuraColors.primaryContainer
              : AuraColors.surfaceContainer.withValues(alpha: 0.5),
          borderRadius: BorderRadius.circular(9999),
        ),
        alignment: Alignment.center,
        child: Text(
          isPressed ? '正在录音...' : '按住说话',
          style: GoogleFonts.manrope(
            fontSize: 14,
            fontWeight: FontWeight.w500,
            color: isPressed
                ? AuraColors.primary
                : AuraColors.onSurfaceVariant,
          ),
        ),
      ),
    );
  }
}
```

- [ ] **Step 3: 改造 ChatInputBar 支持模式切换**

替换 `app/business_packages/sanyan_chat/lib/src/chat/widget/chat_input_bar.dart` 整个文件：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';
import 'chat_input_mode.dart';
import 'hold_to_speak_button.dart';

class ChatInputBar extends StatelessWidget {
  final TextEditingController controller;
  final ChatInputMode mode;
  final VoidCallback onToggleMode;
  final VoidCallback onSendText;
  final bool isRecording;
  final void Function(Offset) onRecordStart;
  final void Function(Offset) onRecordMove;
  final VoidCallback onRecordEnd;
  final VoidCallback onRecordCancel;

  const ChatInputBar({
    super.key,
    required this.controller,
    required this.mode,
    required this.onToggleMode,
    required this.onSendText,
    required this.isRecording,
    required this.onRecordStart,
    required this.onRecordMove,
    required this.onRecordEnd,
    required this.onRecordCancel,
  });

  @override
  Widget build(BuildContext context) {
    return ClipRect(
      child: BackdropFilter(
        filter: ImageFilter.blur(
          sigmaX: AuraColors.glassBlur,
          sigmaY: AuraColors.glassBlur,
        ),
        child: Container(
          color: AuraColors.surface.withValues(alpha: 0.8),
          child: SafeArea(
            top: false,
            child: Padding(
              padding: const EdgeInsets.fromLTRB(12, 10, 12, 10),
              child: Container(
                decoration: BoxDecoration(
                  color: AuraColors.surfaceContainerLowest.withValues(alpha: 0.8),
                  borderRadius: BorderRadius.circular(9999),
                  border: Border.all(
                    color: AuraColors.outlineVariant.withValues(alpha: 0.1),
                    width: 1,
                  ),
                ),
                padding: const EdgeInsets.symmetric(horizontal: 4, vertical: 4),
                child: Row(
                  children: [
                    // 左侧：模式切换按钮
                    IconButton(
                      onPressed: onToggleMode,
                      icon: Icon(
                        mode == ChatInputMode.keyboard
                            ? Icons.mic
                            : Icons.keyboard,
                      ),
                      color: AuraColors.primary,
                      iconSize: 24,
                      padding: EdgeInsets.zero,
                      constraints: const BoxConstraints(minWidth: 36, minHeight: 36),
                    ),
                    const SizedBox(width: 8),

                    // 中间：输入框 或 按住说话按钮
                    Expanded(
                      child: mode == ChatInputMode.keyboard
                          ? _buildTextField()
                          : HoldToSpeakButton(
                              isPressed: isRecording,
                              onPressStart: onRecordStart,
                              onPressMove: onRecordMove,
                              onPressEnd: onRecordEnd,
                              onPressCancel: onRecordCancel,
                            ),
                    ),
                    const SizedBox(width: 8),

                    // 表情
                    Icon(
                      Icons.sentiment_satisfied_outlined,
                      size: 22,
                      color: AuraColors.onSurfaceVariant,
                    ),
                    const SizedBox(width: 4),

                    // 加号
                    IconButton(
                      onPressed: () {},
                      icon: const Icon(Icons.add_circle_outline),
                      color: AuraColors.primary,
                      iconSize: 26,
                      padding: EdgeInsets.zero,
                      constraints: const BoxConstraints(minWidth: 36, minHeight: 36),
                    ),
                  ],
                ),
              ),
            ),
          ),
        ),
      ),
    );
  }

  Widget _buildTextField() {
    return Container(
      decoration: BoxDecoration(
        color: AuraColors.surfaceContainer.withValues(alpha: 0.5),
        borderRadius: BorderRadius.circular(9999),
      ),
      child: TextField(
        controller: controller,
        style: GoogleFonts.manrope(
          fontSize: 14,
          color: AuraColors.onSurface,
        ),
        decoration: InputDecoration(
          hintText: '输入消息...',
          hintStyle: GoogleFonts.manrope(
            fontSize: 14,
            color: AuraColors.outlineVariant,
          ),
          border: InputBorder.none,
          contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
          isDense: true,
        ),
        textInputAction: TextInputAction.send,
        onSubmitted: (_) => onSendText(),
      ),
    );
  }
}
```

- [ ] **Step 4: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: ChatInputBar 支持键盘/语音模式切换 + HoldToSpeakButton"
```

---

### Task 12: 客户端 ChatPage 集成录音交互

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_page.dart`

- [ ] **Step 1: 改造 ChatPage 为 StatefulWidget 并集成录音逻辑**

ChatPage 之前是 StatelessWidget（从 Get.arguments 拿 conversation），现在因为要管录音状态，改成 StatefulWidget。

查看当前 `chat_page.dart` 找到 ChatInputBar 的使用位置，替换整个 ChatPage 类为：

```dart
// 注意：保留原来文件顶部的 imports，新增以下 import：
// import 'voice_recorder.dart';
// import 'widget/chat_input_mode.dart';
// import 'widget/voice_record_overlay.dart';
// import 'package:sanyan_user/sanyan_user.dart'; // LocalStorage
// import 'package:fluttertoast/fluttertoast.dart'; // Toast

// 把原来的 StatelessWidget ChatPage 改成：

class ChatPage extends StatefulWidget {
  ChatPage({super.key});

  final Conversation conversation = Get.arguments as Conversation;
  late final ChatController c = Get.put(ChatController(conversation));

  @override
  State<ChatPage> createState() => _ChatPageState();
}

class _ChatPageState extends State<ChatPage> {
  late ChatInputMode _mode;
  final VoiceRecorder _recorder = VoiceRecorder();
  bool _isRecording = false;
  bool _isCancelling = false;
  Offset? _recordStartPosition;

  @override
  void initState() {
    super.initState();
    _mode = ChatInputMode.fromStorage(LocalStorage.lastInputMode);
  }

  @override
  void dispose() {
    _recorder.dispose();
    super.dispose();
  }

  void _toggleMode() {
    setState(() {
      _mode = _mode == ChatInputMode.keyboard
          ? ChatInputMode.voice
          : ChatInputMode.keyboard;
      LocalStorage.lastInputMode = _mode.storageValue;
    });
  }

  Future<void> _onRecordStart(Offset globalPosition) async {
    final granted = await _recorder.ensurePermission();
    if (!granted) {
      _showToast('需要麦克风权限才能发送语音');
      return;
    }

    final started = await _recorder.start(onMaxDurationReached: () {
      if (_isRecording) _onRecordEnd();
    });
    if (!started) {
      _showToast('录音启动失败');
      return;
    }
    setState(() {
      _isRecording = true;
      _isCancelling = false;
      _recordStartPosition = globalPosition;
    });
  }

  void _onRecordMove(Offset globalPosition) {
    if (!_isRecording || _recordStartPosition == null) return;
    final dy = _recordStartPosition!.dy - globalPosition.dy;
    final cancelling = dy > 80;
    if (cancelling != _isCancelling) {
      setState(() {
        _isCancelling = cancelling;
      });
    }
  }

  Future<void> _onRecordEnd() async {
    if (!_isRecording) return;

    final cancelling = _isCancelling;
    setState(() {
      _isRecording = false;
      _isCancelling = false;
      _recordStartPosition = null;
    });

    if (cancelling) {
      await _recorder.cancel();
      return;
    }

    final result = await _recorder.stop();
    if (result == null) {
      _showToast('说话时间太短');
      return;
    }

    widget.c.sendVoiceMessage(result.filePath, result.durationSeconds);
  }

  Future<void> _onRecordCancel() async {
    if (!_isRecording) return;
    setState(() {
      _isRecording = false;
      _isCancelling = false;
      _recordStartPosition = null;
    });
    await _recorder.cancel();
  }

  void _showToast(String message) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(message), duration: const Duration(seconds: 2)),
    );
  }

  @override
  Widget build(BuildContext context) {
    // ... 复制原来 ChatPage.build 的 Scaffold 内容，保留聊天列表等部分 ...
    // 关键改动：
    // 1. 输入栏用新 ChatInputBar，传递模式和回调
    // 2. 外层包 Stack 添加录音浮层

    return Scaffold(
      body: Stack(
        children: [
          // 保留原来的 Scaffold body 内容结构
          // ... 顶栏 + 消息列表 + 输入栏 ...
          Column(
            children: [
              // ... 顶栏 ...
              Expanded(
                child: Obx(() {
                  if (widget.c.isLoading.value && widget.c.messages.isEmpty) {
                    return const Center(child: CircularProgressIndicator(color: AuraColors.primary));
                  }
                  return ListView.builder(
                    controller: widget.c.scrollController,
                    padding: const EdgeInsets.symmetric(vertical: 16, horizontal: 4),
                    itemCount: widget.c.messages.length,
                    itemBuilder: (context, index) =>
                        MessageBubble(message: widget.c.messages[index]),
                  );
                }),
              ),
              // 新输入栏
              ChatInputBar(
                controller: widget.c.inputController,
                mode: _mode,
                onToggleMode: _toggleMode,
                onSendText: widget.c.sendMessage,
                isRecording: _isRecording,
                onRecordStart: _onRecordStart,
                onRecordMove: _onRecordMove,
                onRecordEnd: _onRecordEnd,
                onRecordCancel: _onRecordCancel,
              ),
            ],
          ),
          // 录音浮层
          if (_isRecording)
            VoiceRecordOverlay(isCancelling: _isCancelling),
        ],
      ),
    );
  }
}
```

读取原 `chat_page.dart` 中顶栏部分的代码，把原来的顶栏逻辑复制过来，组装到新的 Scaffold body 的 Column 顶部。

- [ ] **Step 2: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 3: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: ChatPage 集成录音交互 + 录音浮层 + 模式切换"
```

---

### Task 13: 客户端 VoiceBubble 支持发送状态

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart`

- [ ] **Step 1: 改造 VoiceBubble 显示 sending/failed 状态**

读取当前 `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart`，在构建气泡的 Row 里加上状态指示器。

在 `Row` 最右边（duration 文字之后）加上：

```dart
// 根据 message.status 显示状态
if (widget.message.isSending) ...[
  const SizedBox(width: 8),
  const SizedBox(
    width: 14,
    height: 14,
    child: CircularProgressIndicator(
      strokeWidth: 2,
      color: Colors.white70,
    ),
  ),
] else if (widget.message.isFailed) ...[
  const SizedBox(width: 8),
  GestureDetector(
    onTap: () {
      final c = Get.find<ChatController>();
      c.retryVoiceMessage(widget.message);
    },
    child: Container(
      width: 18,
      height: 18,
      decoration: const BoxDecoration(
        color: Colors.red,
        shape: BoxShape.circle,
      ),
      child: const Icon(Icons.priority_high, color: Colors.white, size: 12),
    ),
  ),
],
```

import 需要添加：
```dart
import 'package:get/get.dart';
import '../chat_controller.dart';
```

**注意**：如果 voice_bubble 是无状态的，而 `message.status` 是可变字段，Flutter 需要用 Obx 或 GetBuilder 触发重建。但因为 ChatController 里用 `messages.refresh()` 触发，整个 ListView 会重建，voice_bubble 的构造函数会被重新调用，所以直接读 `widget.message.isSending` 就够了。

- [ ] **Step 2: 编译验证**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 3: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: VoiceBubble 显示发送中/失败状态 + 失败重试"
```

---

### Task 14: 端到端验证

**Files:** 无代码修改

- [ ] **Step 1: 启动 Android 模拟器**

在 Android Studio 或用命令行启动模拟器。

- [ ] **Step 2: 运行 App**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter run`

- [ ] **Step 3: 手动验证**

在模拟器上：
1. 登录 App
2. 进入和小婉的聊天
3. 点击输入栏左侧麦克风按钮 → 应该切换到语音模式，中间显示"按住说话"
4. 按住"按住说话"按钮 → 应该请求麦克风权限（首次）
5. 授予权限后开始录音 → 浮层应该显示
6. 松开 → 语音气泡应该立即出现在聊天列表中（状态：发送中）
7. 等几秒 → 气泡状态变为已发送
8. 点击播放 → 应该能听到自己刚才录的声音
9. 小婉应该回复一条"没听清"类的消息

- [ ] **Step 4: 验证失败场景**

1. 按住录音 → 不到 1 秒松开 → 应该显示"说话时间太短"Toast
2. 按住录音 → 上滑 → 浮层变红色"松开取消" → 松开 → 不发送
3. 录音时连续按两次录两条 → 两条都应该并发上传，不互相阻塞

- [ ] **Step 5: 提交最终成果**

```bash
cd /Users/aventador/sourceCode/3yan/app
git push origin dev
```
