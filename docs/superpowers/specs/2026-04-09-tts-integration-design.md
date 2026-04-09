# 豆包语音合成 2.0 接入设计

## 概述

在三言 App 后端接入火山引擎豆包语音合成 2.0（Seed-TTS 2.0），AI 回复支持语音模式。语音模式开启时，AI 的文字回复会经过 TTS 合成为音频文件，上传腾讯云 COS，音频 URL 随消息推送给客户端。

## 核心决策

| 决策项 | 选择 | 理由 |
|--------|------|------|
| TTS 调用时机 | 同步生成（方案 A） | TTS 耗时可并入打字延迟窗口，用户无感 |
| 音频存储 | 腾讯云 COS | 用户有免费额度，自带 CDN，不占服务器资源 |
| TTS 接口 | HTTP 一次性合成 | 语音附件场景不需要流式，实现简单 |
| 音色 | zh_female_vv_uranus_bigtts（Vivi 2.0） | 用户选定 |
| 控制方式 | 配置变量 tts.enabled | true 走语音，false 走文字 |

## 消息体设计

语音消息和文字消息共用同一个 Message 实体（已有 contentType 和 mediaUrl 字段）：

```
文字消息：
  contentType = "text"
  content     = "原始文本（含动作描述）"
  mediaUrl    = null

语音消息：
  contentType = "voice"
  content     = "纯净文本（去掉动作描述）"  ← 始终保留，供未来音频转文字展示
  mediaUrl    = "https://3yan-1258800826.cos.ap-beijing.myqcloud.com/voice/{convId}/{msgId}.mp3"
```

## 整体流程

```
用户发消息 → WebSocketHandler
    ↓
MessageService.handleUserMessage()
    ↓
AiService.chat() → AI 文字回复（含动作描述）
    ↓
判断 tts.enabled:
    ├── true:
    │   ├── 1. TextProcessor.extract() → 纯净文本 + 动作描述列表
    │   ├── 2. TtsService.synthesize(纯净文本, 动作描述) → 音频字节数组
    │   ├── 3. CosService.upload(音频, convId, msgId) → COS URL
    │   ├── 4. 保存消息: contentType="voice", content=纯净文本, mediaUrl=URL
    │   └── 5. WS 推送
    └── false:
        ├── 保存消息: contentType="text", content=原始文本
        └── WS 推送
```

## 新增模块

### 1. TextProcessor（工具类）

从 AI 回复中提取中文括号内的动作描述：

```
输入: "这个嘛，是我自己想出来的啦~（双手背在身后，身体轻轻扭动了一下）我会根据对话内容来添加动作描述"
输出:
  cleanText = "这个嘛，是我自己想出来的啦~我会根据对话内容来添加动作描述"
  actions   = ["双手背在身后，身体轻轻扭动了一下"]
```

实现：正则匹配 `（[^）]+）`，提取后从原文删除，去除多余空格。

### 2. TtsService

调用火山引擎 TTS API 生成音频。

- 接口：`https://openspeech.bytedance.com/api/v1/tts`（HTTP 一次性合成）
- 认证：App ID + Access Token（Bearer Token 方式）
- 请求参数：
  - app.appid: 7231248180
  - app.token: Access Token
  - app.cluster: volcano_tts
  - audio.voice_type: zh_female_vv_uranus_bigtts
  - audio.encoding: mp3
  - request.text: 纯净文本
  - request.text_type: plain
- 语音标签：将动作描述转为 TTS 2.0 的语音标签格式，影响语气和情感
- 返回：音频二进制数据（base64 编码）

### 3. CosService

上传音频文件到腾讯云 COS。

- Bucket: 3yan-1258800826
- Region: ap-beijing
- 路径规则: voice/{conversationId}/{messageId}.mp3
- 权限: Bucket 设为公有读私有写，客户端直接用 URL 播放
- 依赖: 腾讯云 COS Java SDK (cos_api)

## 配置

application-dev.yml 新增：

```yaml
sanyan:
  tts:
    enabled: false
    app-id: 7231248180
    access-token: ${TTS_ACCESS_TOKEN:VuO0W4r1DVtvxl_7bEfuE3Zbr3iBVuL9}
    voice-type: zh_female_vv_uranus_bigtts
    cluster: volcano_tts
  cos:
    secret-id: ${COS_SECRET_ID:AKIDMcBZkjW9Zt0ta5SccOw1sXyDHD2g7LOG}
    secret-key: ${COS_SECRET_KEY:lKhqQo270f8OTO7dz99wVcXEj7IJBsQm}
    bucket: 3yan-1258800826
    region: ap-beijing
```

## MessageService 改动

在 `handleUserMessage()` 方法中，AI 回复拿到后：

1. 检查 `tts.enabled`
2. 如果 true：调 TextProcessor → TtsService → CosService
3. 设置 `aiMsg.setContentType("voice")` / `aiMsg.setMediaUrl(cosUrl)` / `aiMsg.setContent(cleanText)`
4. 如果 false：保持现有逻辑不变

## MessageData DTO 改动

确认 `mediaUrl` 字段已暴露给客户端（当前 MessageData 没有 mediaUrl）：

```java
@Data
public class MessageData {
    private Long id;
    private Long conversationId;
    private String senderType;
    private String contentType;
    private String content;
    private String mediaUrl;      // 新增
    private String source;
    private LocalDateTime createdAt;
}
```

## 客户端适配

客户端 Message 模型和消息气泡需要支持 voice 类型：
- 检测 contentType == "voice" 且 mediaUrl 不为空
- 展示语音播放按钮（替代文字气泡）
- content 字段暂不展示，后续可做"语音转文字"功能时使用

## 打字延迟调整

当前打字延迟在 AI 回复保存之后计算。语音模式下 TTS + 上传已经有 2-3 秒耗时，可以减少或取消额外的打字延迟，避免用户等太久。

逻辑：如果 contentType == "voice"，打字延迟 = max(0, 原延迟 - TTS耗时)。

## 未来扩展

- **实时语音聊天**：独立的全双工流式通信流程，复用现有 WebSocket 连接，扩展消息类型即可，不影响现有语音附件逻辑
- **音频转文字展示**：content 字段已保留纯净文本，直接展示即可
- **多音色切换**：voice-type 改为角色级别配置，不同 AI 角色用不同声音
- **声音复刻**：使用火山引擎声音复刻 API 定制小婉专属声音

## 依赖

后端 Maven 新增：
- `com.qcloud:cos_api` — 腾讯云 COS SDK
- 无需额外 TTS SDK，直接 HTTP 调用火山引擎 API
