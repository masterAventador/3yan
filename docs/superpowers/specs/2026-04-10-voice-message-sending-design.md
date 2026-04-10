# 语音消息发送功能设计（第一阶段）

## 概述

三言 App 客户端支持用户发送语音消息。第一阶段实现完整的录音 → 上传 → WebSocket 发送 → 对方展示 → AI 回复的链路，但 AI 暂不理解语音内容（第二阶段接入 ASR）。第一阶段的验收标准：用户自己能在聊天页发语音，气泡正常显示，能播放自己录的内容，小婉给出符合人设的"没听清"撒娇回复。

## 核心决策

| 决策项 | 选择 | 理由 |
|--------|------|------|
| 分阶段实施 | 分两步 | 每一步独立可验证，降低风险 |
| 录音交互 | 按住说话（微信式） | 符合国人习惯 |
| UI 模式 | 两种模式切换，记住用户偏好 | 跟微信一致 |
| 上传策略 | 本地乐观 UI + 并发上传 | 不阻塞多条消息 |
| 最长时长 | 60 秒 | 覆盖 99% 场景 |
| 录音格式 | m4a (AAC) | 压缩率高，两端原生支持 |
| 失败处理 | 红色感叹号 + 点击重试 | 用户有控制权 |
| 上传架构 | 上传 + 发送分两步 | 原子性好，易重试 |
| AI 回复（第一阶段） | 生成符合人设的"没听清"回复 | 既保持流畅，又为第二阶段铺垫 |

## 一、整体流程

```
┌──────────────────────────────────────────────────────────┐
│                      客户端                              │
├──────────────────────────────────────────────────────────┤
│ 1. 用户按住语音按钮                                      │
│ 2. record 包开始录音 → m4a 文件存到 app 缓存目录         │
│ 3. 松开 → 停止录音                                        │
│    - 上滑取消：删除本地文件，不发送                       │
│    - 正常松开：进入发送流程                               │
│ 4. 乐观 UI：立即在聊天页显示语音气泡（状态：sending）    │
│ 5. 独立 Future 并发上传，不阻塞其他消息                  │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│                      服务端                              │
├──────────────────────────────────────────────────────────┤
│ 1. POST /api/media/upload                                 │
│    - multipart file + duration + type                     │
│    - JWT 鉴权                                             │
│    - 上传到腾讯云 COS                                     │
│    - 路径：voice/user/{userId}/{timestamp}_{uuid}.m4a     │
│    - 返回 { url, duration }                               │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│                      客户端                              │
├──────────────────────────────────────────────────────────┤
│ 6. 拿到 url → 发 WS send_message                          │
│    - contentType: "voice"                                 │
│    - mediaUrl: url                                        │
│    - duration: 秒数                                       │
│    - clientMsgId: 本地生成                                │
│ 7. 收到 ACK → 气泡状态改为"sent"                          │
│ 8. 删除本地 m4a 文件                                      │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│                      服务端                              │
├──────────────────────────────────────────────────────────┤
│ 1. WebSocket 收到 voice 消息                              │
│ 2. 保存用户消息 (contentType=voice, mediaUrl, duration)   │
│ 3. 触发 AI 回复：                                         │
│    - 调豆包大模型，传入特殊 prompt 让小婉生成"没听清"回复 │
│    - AI 回复走现有 TTS 流程（文字或语音）                 │
└──────────────────────────────────────────────────────────┘
```

## 二、客户端实现

### 2.1 依赖

```yaml
record: ^5.1.2              # 录音
permission_handler: ^11.3.1 # 权限请求
path_provider: ^2.1.3       # 获取缓存目录（已有）
```

### 2.2 新增文件

```
business_packages/sanyan_chat/lib/src/chat/
  voice_recorder.dart              # 录音管理器
  widget/
    voice_record_overlay.dart      # 录音浮层 UI
    hold_to_speak_button.dart      # 按住说话长条按钮
```

### 2.3 修改文件

```
business_packages/sanyan_chat/lib/src/chat/widget/chat_input_bar.dart
  - 新增输入模式切换（keyboard/voice）
  - 布局：[麦克风/键盘图标] [输入框/长条按钮] [表情] [加号]

business_packages/sanyan_chat/lib/src/chat/chat_controller.dart
  - 新增 sendVoiceMessage() 方法
  - 新增 retryVoiceMessage() 方法

business_packages/sanyan_chat/lib/src/models/message.dart
  - 新增 status 字段（sending/sent/failed）
  - 新增 localFilePath 字段

business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart
  - 根据 status 显示不同状态（加载中、已发送、失败感叹号）

business_packages/sanyan_chat/lib/src/api/chat_api.dart
  - 新增 uploadVoice() 方法

foundation_packages/sanyan_network/lib/src/ws_client.dart
  - sendVoiceMessage() 新方法

foundation_packages/sanyan_user/lib/src/dao/local_storage.dart
  - 新增 lastInputMode getter/setter
```

### 2.4 ChatInputBar 改造

**模式**：
```dart
enum ChatInputMode { keyboard, voice }
```

**布局**：
```
[🎤/⌨️] [输入框 或 按住说话长条按钮] [😀 表情] [+ 加号]
```

- `ChatInputMode.keyboard`：左侧显示麦克风图标，中间是 TextField
- `ChatInputMode.voice`：左侧显示键盘图标，中间是 HoldToSpeakButton

**模式切换**：点击左侧图标切换。切换后立即保存到 `LocalStorage.lastInputMode`。

**初始模式**：从 `LocalStorage.lastInputMode` 读取，默认 keyboard。

### 2.5 录音浮层

按住说话按钮时，在输入栏上方悬浮显示：

```
┌─────────────────────────┐
│                         │
│         🎤              │  ← 麦克风图标
│                         │
│       正在录音          │
│        00:12            │  ← 实时秒数
│                         │
│    👆 松开发送          │
│    👆 上滑取消          │
└─────────────────────────┘
```

**手势逻辑**：
- 按下 → 请求录音权限 → 开始录音，显示浮层
- 手指在按钮上 → 正常录音状态
- 手指上滑到浮层区域 → 切换为"松开取消"红色提示
- 松开 → 根据手指位置判断发送或取消
- 超过 60 秒 → 自动停止录音并发送
- 录音时长 < 1 秒 → Toast "说话时间太短"，不发送，删除本地文件

### 2.6 消息状态管理

```dart
enum MessageStatus {
  sending,   // 正在上传/发送
  sent,      // 发送成功
  failed,    // 发送失败
}
```

在 `Message` 模型中新增：
```dart
MessageStatus? status;
String? localFilePath;
```

注意：只有正在发送中的消息会设置这两个字段。从服务端拿到的历史消息 `status` 默认为 `sent`。

### 2.7 发送流程（ChatController）

```dart
Future<void> sendVoiceMessage(String localPath, int duration) async {
  final clientMsgId = _uuid.v4();

  // 1. 乐观 UI
  final msg = Message(
    clientMsgId: clientMsgId,
    contentType: ContentType.voice,
    localFilePath: localPath,
    status: MessageStatus.sending,
    senderType: 'user',
    // ...
  );
  messages.add(msg);
  _scrollToBottom();

  // 2. 独立 Future 并发上传（不 await）
  _uploadAndSend(msg);
}

Future<void> _uploadAndSend(Message msg) async {
  try {
    // 上传文件
    final uploadResult = await ChatApi.uploadVoice(
      msg.localFilePath!,
      duration: msg.duration ?? 0,
    );
    if (!uploadResult.success) {
      _markFailed(msg);
      return;
    }

    // WebSocket 发送
    Get.find<WsClient>().sendVoiceMessage(
      conversationId: conversation.id,
      mediaUrl: uploadResult.data!.url,
      duration: uploadResult.data!.duration,
      clientMsgId: msg.clientMsgId!,
    );

    // ACK 由 _listenWs 收到后处理，更新 status 为 sent
    // 并删除本地文件
  } catch (e) {
    _markFailed(msg);
  }
}

Future<void> retryVoiceMessage(Message msg) async {
  msg.status = MessageStatus.sending;
  messages.refresh();
  await _uploadAndSend(msg);
}

void _markFailed(Message msg) {
  msg.status = MessageStatus.failed;
  messages.refresh();
}
```

### 2.8 本地文件生命周期

- **录音完成**：存到 `getApplicationCacheDirectory()/sanyan_voice/{uuid}.m4a`
- **发送成功**（收到 ACK）：删除对应本地文件
- **发送失败**：保留文件，等用户手动重试
- **App 启动时清理**：扫描 `sanyan_voice/` 目录，删除超过 7 天的残留文件

### 2.9 失败重试 UI

语音气泡 UI 检测 `status == failed` 时：
- 气泡右侧（用户消息）显示红色感叹号图标
- 点击感叹号 → 调用 `retryVoiceMessage(msg)` → 重新走一遍上传+发送流程

`status == sending` 时：
- 气泡右侧显示转圈 loading 指示器

## 三、服务端实现

### 3.1 新增文件

```
server/src/main/java/com/sanyan/
  controller/MediaController.java   # 媒体上传接口
  dto/data/MediaUploadData.java     # 上传响应数据
  service/MediaService.java         # 媒体上传业务逻辑
```

### 3.2 新增接口

```
POST /api/media/upload
  Header:
    Authorization: Bearer {jwt}
    Content-Type: multipart/form-data
  Body:
    - file: m4a 文件（必填）
    - type: "voice"（必填，为后续扩展图片/视频预留）
    - duration: 秒数（必填，客户端传，避免服务端解析音频）

  Response:
    {
      "success": true,
      "data": {
        "url": "https://3yan-1258800826.cos.ap-beijing.myqcloud.com/voice/user/1/1775000000_abc123.m4a",
        "duration": 12
      }
    }
```

**校验**：
- 文件大小 ≤ 5MB
- 文件类型必须是 audio/mp4 或 audio/m4a
- duration 必须 1 ≤ duration ≤ 60
- 必须有有效 JWT

### 3.3 MediaService

```java
public class MediaService {
    public MediaUploadData uploadVoice(Long userId, MultipartFile file, Integer duration) {
        // 1. 校验文件
        validateVoiceFile(file, duration);

        // 2. 生成 COS 路径
        String key = "voice/user/" + userId + "/"
                   + System.currentTimeMillis() + "_"
                   + UUID.randomUUID().toString().substring(0, 8)
                   + ".m4a";

        // 3. 上传到 COS
        String url = cosService.upload(file.getBytes(), key, "audio/mp4");

        return new MediaUploadData(url, duration);
    }
}
```

### 3.4 CosService 泛化

现有签名：
```java
public String upload(byte[] data, Long conversationId, Long messageId)
```

改为通用签名：
```java
public String upload(byte[] data, String key, String contentType)
```

并添加一个兼容方法给旧的 AI 语音合成用：
```java
public String uploadAiVoice(byte[] data, Long conversationId, Long messageId) {
    return upload(data, "voice/ai/" + conversationId + "/" + messageId + ".mp3", "audio/mpeg");
}
```

### 3.5 WebSocket 消息协议扩展

`WsMessage.java` 新增字段：
```java
private String mediaUrl;   // 媒体 URL（语音/视频等消息类型用）
private Integer duration;  // 时长（秒，语音消息用）
```

当服务端收到 `type=send_message, contentType=voice` 的消息：
1. 校验 `mediaUrl` 非空
2. 保存消息到数据库（带 mediaUrl 和 duration）
3. 触发 AI 回复

### 3.6 MessageService 改造

`handleUserMessage` 签名扩展：
```java
public Message handleUserMessage(
    Long userId, Long conversationId,
    String contentType, String content,
    String mediaUrl, Integer duration
);
```

保存用户消息时：
- `contentType == "text"`：content 存文字，mediaUrl null
- `contentType == "voice"`：content 为空字符串，mediaUrl 存 COS URL，duration 存秒数

AI 回复分支：
```java
String aiReply;
if ("voice".equals(contentType)) {
    // 第一阶段：生成符合人设的"没听清"回复
    aiReply = aiService.chatVoiceAck(character, conversationId);
} else {
    // 原有文字消息逻辑
    aiReply = aiService.chat(character, conversationId);
}

// 后续 TTS 合成、保存、WebSocket 推送等流程保持不变
```

### 3.7 AI 回复：chatVoiceAck

在 `AiService` 新增方法：
```java
public String chatVoiceAck(AiCharacter character, Long conversationId) {
    String basePrompt = assembleSystemPrompt(character.getSystemPrompt(),
                                              formatCurrentTime(),
                                              getProfile(conversationId));
    String voicePrompt = basePrompt + "\n\n[特殊场景] 用户刚才发了一条语音消息，"
        + "但你现在还无法听懂语音内容。请用符合你人设的方式回复，"
        + "表达你听到了但没听清，让用户再说一遍或者打字告诉你。"
        + "要自然、多变、撒娇，每次都不一样。注意还是要保留语音情感标签。";

    List<MemorySummary> summaries = ...; // 同 chat()
    List<Message> recentMessages = ...;  // 同 chat()

    return callDoubao(voicePrompt, summaries, recentMessages);
}
```

## 四、权限与错误处理

### 4.1 权限配置

**iOS** (`ios/Runner/Info.plist`)：
```xml
<key>NSMicrophoneUsageDescription</key>
<string>三言需要使用麦克风来录制语音消息</string>
```

**Android** (`android/app/src/main/AndroidManifest.xml`)：
```xml
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```

### 4.2 权限请求时机

用户第一次点击麦克风按钮进入语音模式时请求权限，不在 App 启动时请求。

用 `permission_handler` 统一处理：
```dart
final status = await Permission.microphone.request();
if (status.isDenied) {
  Toast.show('需要麦克风权限才能发送语音');
  return;
}
if (status.isPermanentlyDenied) {
  // 弹窗引导去设置
  _showPermissionDialog();
  return;
}
```

### 4.3 错误处理清单

| 错误场景 | 处理方式 |
|---------|---------|
| 麦克风权限被拒 | Toast 提示 + 引导设置 |
| 录音时长 < 1 秒 | Toast "说话时间太短"，不发送 |
| 录音失败（硬件异常） | Toast "录音失败，请重试" |
| 本地文件写入失败 | Toast "录音保存失败" |
| 上传文件超过 5MB | Toast "语音文件过大" |
| 上传网络超时 | 气泡变红色感叹号，可重试 |
| 上传服务端 5xx | 同上 |
| WebSocket 掉线 | 气泡变红色感叹号，WS 自动重连后可重试 |
| 服务端保存失败 | WS 返回错误 → 气泡变红色感叹号 |

### 4.4 超时策略

- 录音自动停止：60 秒
- 上传 HTTP 请求超时：30 秒
- 整条消息发送超时（按下到收到 ACK）：60 秒

### 4.5 缓存清理

App 启动时在 `main()` 中调用：
```dart
VoiceCacheManager.cleanupOldFiles();
```

实现：扫描 `getApplicationCacheDirectory()/sanyan_voice/`，删除修改时间超过 7 天的文件。

## 五、分阶段说明

### 第一阶段（本次实现）
- 客户端录音 + 上传 + 发送链路
- 服务端媒体上传接口
- 用户能发语音，对方能收到并播放
- AI 用"没听清"的预置回复

### 第二阶段（下次实现）
- 服务端接入豆包 ASR
- 语音消息进服务端 → ASR 转文字 → 喂给大模型 → 正常语义回复
- 本阶段已预留扩展点：`chatVoiceAck` 改成 `chatWithVoice` 即可

## 六、不在本次范围内

- 语音波形可视化（录音时的动画）
- 发送前试听/预览
- 语音转文字（第二阶段）
- 语音通话/实时对讲
- 断点续传
- 语音文件压缩优化
- CDN 加速
