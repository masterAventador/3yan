# 豆包 ASR 接入设计（语音消息第二阶段）

## 概述

让小婉真正"听懂"用户发送的语音消息。当前第一阶段小婉收到语音消息后只会用 `chatVoiceAck()` 回一句"没听清"的撒娇话；第二阶段接入豆包大模型录音文件识别**极速版**（auc_turbo），将用户语音转成文字后喂给正常对话流程，让小婉理解并回复相关内容。

顺便清理项目中所有 `senderType` 和 `contentType` 字符串硬编码，统一用常量类。

## 核心决策

| 决策项 | 选择 | 理由 |
|--------|------|------|
| ASR 调用时机 | 串行（保存消息 → ASR → AI 回复） | 让 AI 上下文里自然包含转写文字 |
| ASR 接口 | 极速版 `recognize/flash` | 一次请求返回，不需要轮询 |
| 失败策略 | 重试 1 次，仍失败降级到 `chatVoiceAck` | 既有容错又保留"没听清"的自然降级 |
| 转写文字存储 | 存到 `message.content` 字段 | 与 AI 语音消息对称，复用现有字段 |
| 音频格式 | 先试 m4a，失败再降级 WAV | 极速版大模型对格式容错性通常很高，0 客户端改动优先 |
| 硬编码清理 | ASR 实现前作为 Task 0 执行 | 新代码直接用常量类，避免引入新的硬编码 |

## 一、整体流程

```
用户发语音（已有）
    ↓
客户端录 m4a → 上传 COS → 拿 URL → WS send_message(voice, mediaUrl, duration)
    ↓
服务端 WebSocketHandler.handleSendMessage()
    ↓
MessageService.handleUserMessage(contentType=voice, mediaUrl, duration)
    ↓
保存用户消息（content 先空着）
    ↓
━━━━━ 新增：ASR 调用 ━━━━━
AsrService.transcribe(mediaUrl)
    ├─ POST /api/v3/auc/bigmodel/recognize/flash
    ├─ 响应 header X-Api-Status-Code
    │   ├─ 20000000 → 返回 result.text
    │   ├─ 20000003 → 静音，返回空字符串
    │   └─ 其他 → 失败
    └─ 失败时重试 1 次
    ↓
if (ASR 成功 && 文字非空) {
    userMsg.setContent(transcribedText)
    messageRepository.save(userMsg)  // 回填 content
    aiReply = aiService.chat(...)  // 正常对话（上下文 recentMessages 已含转写文字）
} else {
    aiReply = aiService.chatVoiceAck(...)  // 降级：没听清
}
    ↓
原有 TTS + COS 上传 + WebSocket 推送流程不变
```

**关键点**：
1. ASR 在保存用户消息之后、调 AI 之前执行
2. ASR 成功后**回填** user message 的 content 字段，这样 `aiService.chat()` 读 `recentMessages` 时能看到文字
3. 客户端**零改动**（先试 m4a，失败再考虑改录音格式）

## 二、服务端实现

### 2.1 Task 0：清理硬编码

在实现 ASR 前先清理项目中 `senderType` 和 `contentType` 的字符串硬编码。

**新增文件**：
- `server/src/main/java/com/sanyan/dto/ws/SenderType.java`
  ```java
  public class SenderType {
      public static final String USER = "user";
      public static final String AI = "ai";
  }
  ```

**修改文件**（替换所有 `"user"` / `"ai"` / `"voice"` / `"text"` 字面量为常量引用）：

| 文件 | 行号 | 改动 |
|------|------|------|
| `service/MessageService.java` | 48 | `setSenderType("user")` → `SenderType.USER` |
| `service/MessageService.java` | 62 | `"voice".equals(contentType)` → `MessageContentType.VOICE.equals(contentType)` |
| `service/MessageService.java` | 82 | `setSenderType("ai")` → `SenderType.AI` |
| `service/MessageService.java` | 116 | `setSenderType("ai")` → `SenderType.AI` |
| `service/AiService.java` | 132 | `"user".equals(...)` → `SenderType.USER.equals(...)` |
| `service/MemoryService.java` | 92, 109 | `"user".equals(...)` → `SenderType.USER.equals(...)` |
| `service/ProactiveService.java` | 116 | `setSenderType("ai")` → `SenderType.AI` |
| `service/ProactiveService.java` | 189 | `"ai".equals(...)` → `SenderType.AI.equals(...)` |

**客户端新增文件**：
- `app/foundation_packages/sanyan_network/lib/src/sender_type.dart`
  ```dart
  class SenderType {
    static const String user = 'user';
    static const String ai = 'ai';
  }
  ```
- 通过 `sanyan_network.dart` 导出

**客户端修改文件**：
| 文件 | 行号 | 改动 |
|------|------|------|
| `chat_api.dart` | 53 | `'type': 'voice'` → `'type': ContentType.voice` |
| `chat_controller.dart` | 89, 108 | `senderType: 'user'` → `senderType: SenderType.user` |
| `chat_controller.dart` | 90 | `contentType: 'text'` → `contentType: ContentType.text` |
| `models/message.dart` | 33 | `senderType == 'ai'` → `senderType == SenderType.ai` |
| `models/message.dart` | 43 | `json['contentType'] ?? 'text'` → `json['contentType'] ?? ContentType.text` |
| `widget/message_bubble.dart` | 15 | `senderType == 'user'` → `senderType == SenderType.user` |
| `widget/voice_bubble.dart` | 51 | `senderType == 'user'` → `senderType == SenderType.user` |

**说明**：字段类型保持 `String`（不改成 enum），避免影响 JPA 映射和 JSON 序列化。常量类只是防止硬编码散落。

### 2.2 AsrService

**新增文件**：`server/src/main/java/com/sanyan/service/AsrService.java`

**职责**：调用豆包 ASR 极速版 API，转写音频 URL 为文字。

**接口**：
- URL: `https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash`
- 认证 Header（旧版控制台）:
  - `X-Api-App-Key`: App ID
  - `X-Api-Access-Key`: Access Token
  - `X-Api-Resource-Id`: `volc.bigasr.auc_turbo`
  - `X-Api-Request-Id`: UUID
  - `X-Api-Sequence`: `-1`
- 请求体:
  ```json
  {
    "user": { "uid": "sanyan_server" },
    "audio": { "url": "COS_URL" },
    "request": {
      "model_name": "bigmodel",
      "enable_itn": true,
      "enable_punc": true
    }
  }
  ```

**响应处理**：
- 成功：Response Header `X-Api-Status-Code == 20000000`，Body JSON 的 `result.text` 是识别文字
- 静音：`X-Api-Status-Code == 20000003`，返回空字符串
- 失败：其他状态码，返回 null

**错误码对照**：
- `20000000`：成功
- `20000003`：静音音频
- `45000001`：请求参数无效
- `45000002`：空音频
- `45000151`：音频格式不正确
- `550XXXX`：服务内部错误
- `55000031`：服务器繁忙

**核心方法签名**：
```java
/**
 * 转写音频为文字
 * @param audioUrl 公网可访问的音频 URL
 * @return 转写文字；静音返回空字符串；失败返回 null
 */
public String transcribe(String audioUrl);

/**
 * 是否启用 ASR 服务
 */
public boolean isEnabled();
```

**重试策略**：失败时重试 1 次，仍失败返回 null。

**超时**：使用现有 `RestTemplate` 的默认超时（connect 5s、read 30s）。

### 2.3 配置新增

`server/src/main/resources/application-dev.yml` 新增：
```yaml
sanyan:
  asr:
    enabled: true
    app-id: "6383804695"
    access-token: ${ASR_ACCESS_TOKEN:BgkVhS2NOZfEP9TbhuWdREWzbZEjNLHi}
```

**注意**：ASR 的凭据跟 TTS 不同（不同 App ID），独立配置。

### 2.4 MessageService 改动

`MessageService.handleUserMessage()` 方法内，找到当前的语音分支：

```java
if (MessageContentType.VOICE.equals(contentType)) {
    aiReply = aiService.chatVoiceAck(character, conversationId);
} else {
    aiReply = aiService.chat(character, conversationId);
}
```

改为：

```java
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
        // ASR 失败或静音：降级
        log.info("ASR 失败或静音，降级为 chatVoiceAck: convId={}", conversationId);
        aiReply = aiService.chatVoiceAck(character, conversationId);
    }
} else {
    aiReply = aiService.chat(character, conversationId);
}
```

### 2.5 依赖注入

`MessageService` 已经通过 `@RequiredArgsConstructor` 注入了 TtsService、CosService 等，添加 `AsrService` 即可：

```java
private final AsrService asrService;
```

## 三、客户端改动

**第二阶段客户端零改动**。保持第一阶段的录音格式（m4a）和上传流程不变。

**例外**：Task 0 的硬编码清理会涉及客户端文件，但不涉及 ASR 业务逻辑。

## 四、音频格式兼容性兜底

**风险**：极速版 ASR 文档明确支持的音频格式是 WAV / MP3 / OGG OPUS，**没列 m4a**。客户端当前录的是 m4a。

**预期结果**：
- 最佳情况：极速版大模型对格式容错性高，m4a 能直接用
- 最坏情况：返回错误码 `45000151`（音频格式不正确）

**兜底方案**：如果实现后测试发现 m4a 不被支持，降级到录 WAV：
- 客户端 `VoiceRecorder` 将 `AudioEncoder.aacLc` 改为 `AudioEncoder.wav`
- 文件扩展名 `.m4a` 改为 `.wav`
- 上传 `MultipartFile` 的 filename 和 `Content-Type` 相应调整
- 注意：60 秒 WAV 文件约 2MB，是 m4a 的 6 倍，上传会慢一些

## 五、测试验证

**手动验证步骤**：
1. 发一条语音说"今天心情有点不好"
2. 服务端日志应该出现：
   - `ASR V3 响应: statusCode=20000000`
   - `ASR 转写成功: text=今天心情有点不好。`
3. 小婉的回复应该与"心情不好"相关（安慰话），而不是"没听清"的撒娇话
4. 数据库 `message` 表里这条语音消息的 `content` 字段应该存着转写文字

**失败场景验证**：
1. 发一条完全无声的语音 → 小婉应该走 `chatVoiceAck` 降级回复
2. 临时把 ASR 配置改错导致认证失败 → 小婉应该走 `chatVoiceAck` 降级回复
3. 检查日志里有无"ASR 失败或静音，降级为 chatVoiceAck"

## 六、不在本次范围内

- ASR 回调通知模式（本次用同步请求/响应）
- ASR 结果的句级/词级时间戳（API 返回了但我们不用）
- 多说话人分离（`enable_speaker_info` 不开启）
- 情感识别、性别识别（极速版不支持，也不需要）
- WAV 格式降级的客户端实现（作为 m4a 不兼容时的预案，不是本次必做）
- 客户端展示 ASR 转写文字（虽然存到了 content，但本次不做"语音气泡下方显示文字"的 UI）
