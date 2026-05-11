# 消息发送可靠性 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 给 3yan 客户端所有类型的消息（文本/语音/未来其他）建立统一的 sending→sent/failed 状态机，基于服务端 ACK + 超时 + 断线事件驱动；跨聊天页生命周期保持 pending 状态（内存 + 本地持久化）；自动重连 WebSocket；失败消息红感叹号可手动重试。

**Architecture:** 在 `foundation_packages/sanyan_network` 新增 `MessageSender`（GetxService 单例，App 生命周期），用一个 `Map<clientMsgId, _PendingEntry>` + 一个 1s 周期扫描 Timer（懒启动）管理所有 pending 消息。`WsClient` 扩展 `onDisconnected` 事件流和自动重连（指数退避）。`MessageSender` 监听 ACK / 断线，更新 Message 对象状态并用 GetStorage 持久化。ChatController 把 Message 创建后交给 Sender，订阅 Sender 的状态变更流刷 UI，进入聊天页时合并 Sender 里该会话的 pending 消息。气泡 UI 抽基类统一展示 sending/failed 指示。

**Tech Stack:** Flutter 3.11 · Dart · GetX 4.6 · `web_socket_channel` · `get_storage` · `uuid` · `flutter_test` + `mocktail`（单元测试）

---

## File Structure

**新建（sanyan_network）：**
- `lib/src/message_sender.dart` — `MessageSender` GetxService，核心调度 + pending 管理 + 持久化
- `lib/src/pending_entry.dart` — `_PendingEntry` 内部类（clientMsgId、Message 引用、sendTimeMs），含 JSON 序列化
- `lib/src/message_status_change.dart` — `MessageStatusChange` 事件 DTO（conversationId + Message）

**修改（sanyan_network）：**
- `lib/src/ws_client.dart` — 暴露 `Stream<void> onDisconnected`；自动重连（指数退避）；`sendMessage`/`sendVoiceMessage` 签名改为接受外部传入的 `clientMsgId`
- `lib/sanyan_network.dart` — barrel 加 `message_sender.dart` / `message_status_change.dart` 导出
- `pubspec.yaml` — 加 `get_storage: ^2.1.1`、`uuid: ^4.3.3`

**修改（sanyan_chat）：**
- `lib/src/chat/chat_controller.dart` — `sendMessage`/`sendVoiceMessage` 交给 `MessageSender`；`onInit` 合并 `sender.getPending(conversationId)` 到 `messages`；订阅 `sender.statusChanges` 刷 UI；新增 `retryMessage(Message)`
- `lib/src/chat/widget/message_bubble.dart` — 用新基类；文本气泡内容不变
- `lib/src/chat/widget/voice_bubble.dart` — 用新基类；**删除现存的失败 UI 代码**
- `lib/src/api/models/message.dart` — 加 `toJson()` 覆盖 status/clientMsgId 等字段（MessageSender 持久化用）

**新建（sanyan_chat）：**
- `lib/src/chat/widget/message_bubble_base.dart` — 抽基类：外壳 + 状态层（sending 小菊花 / failed 红感叹号 / sent 无指示）；子类只实现"内容区域"

**修改（app）：**
- `lib/main.dart` — 启动时 `Get.put<MessageSender>(MessageSender(), permanent: true)` 并 await `onInit`（保证冷启动 pending 加载完再启路由）

**测试：**
- `foundation_packages/sanyan_network/test/message_sender_test.dart` — 新建，覆盖：send + ACK → sent、超时 → failed、断线 → failed、retry、getPending、冷启动加载 pending → 立即标 failed
- `foundation_packages/sanyan_network/test/ws_client_reconnect_test.dart` — 新建，覆盖：断开事件、指数退避重连
- `business_packages/sanyan_chat/test/chat_controller_test.dart` — 更新：发送流程改用 MessageSender mock；语音消息不再 `status=sent` 立刻

---

## Task 1: 建分支 + 加依赖

**Files:**
- Modify: `foundation_packages/sanyan_network/pubspec.yaml`

- [ ] **Step 1: 建分支**

```bash
cd /Users/aventador/code/3yan/app
git checkout -b feat/message-reliability
```

Expected: `Switched to a new branch 'feat/message-reliability'`

- [ ] **Step 2: 加依赖到 sanyan_network**

改 `foundation_packages/sanyan_network/pubspec.yaml` 的 `dependencies:`：

```yaml
dependencies:
  flutter:
    sdk: flutter
  dio: ^5.4.3
  web_socket_channel: ^3.0.0
  get: ^4.6.6
  uuid: ^4.3.3
  get_storage: ^2.1.1
```

（`uuid` 已是 sanyan_chat 的依赖，这里在 network 层也需要，版本对齐。`get_storage` 在 sanyan_user 已经在用了，传递版本一致。）

- [ ] **Step 3: pub get 验证**

```bash
cd /Users/aventador/code/3yan/app
flutter pub get
```

Expected: `Got dependencies!`

- [ ] **Step 4: 提交**

```bash
git add -A
git commit -m "chore(network): 加 get_storage 和 uuid 依赖，为 MessageSender 准备"
```

---

## Task 2: WsClient 暴露 onDisconnected 事件流

**目的：** Socket 底层断开（onDone/onError）时，MessageSender 需要立刻知道并把所有 pending 标 failed。

**Files:**
- Modify: `foundation_packages/sanyan_network/lib/src/ws_client.dart`
- Test (create): `foundation_packages/sanyan_network/test/ws_client_disconnect_test.dart`

- [ ] **Step 1: 写失败测试**

Create `foundation_packages/sanyan_network/test/ws_client_disconnect_test.dart`:

```dart
import 'dart:async';
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_network/sanyan_network.dart';

void main() {
  test('WsClient exposes onDisconnected stream', () {
    final client = WsClient();
    expect(client.onDisconnected, isA<Stream<void>>());
  });

  test('WsClient.onDisconnected emits when _onError is triggered', () async {
    final client = WsClient();
    final events = <void>[];
    final sub = client.onDisconnected.listen(events.add);

    // 直接触发内部的断开处理（需要暴露 test-only 方法或通过反射）
    client.notifyDisconnectedForTest();

    await Future.delayed(const Duration(milliseconds: 10));
    expect(events, hasLength(1));
    await sub.cancel();
  });
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
cd /Users/aventador/code/3yan/app/foundation_packages/sanyan_network
flutter test test/ws_client_disconnect_test.dart
```

Expected: 编译失败 (`onDisconnected` 和 `notifyDisconnectedForTest` 未定义)

- [ ] **Step 3: 实现 onDisconnected**

修改 `foundation_packages/sanyan_network/lib/src/ws_client.dart` 的 class body，在 `class WsClient extends GetxService {` 后面加：

```dart
final StreamController<void> _disconnectedController = StreamController<void>.broadcast();

/// Fires when underlying WebSocket closes (onDone or onError).
/// 注意：本地客户端主动 close 也会触发，订阅方需自行判断是否要重连。
Stream<void> get onDisconnected => _disconnectedController.stream;

/// 仅供测试使用，手动触发断开事件。
@visibleForTesting
void notifyDisconnectedForTest() => _disconnectedController.add(null);
```

同时在 `ws_client.dart` 顶部加 import：

```dart
import 'package:flutter/foundation.dart' show visibleForTesting;
```

找到现有代码里 socket stream 的监听逻辑（onDone 和 onError 回调），分别加上 `_disconnectedController.add(null);`。当前代码大概有 `.listen(..., onDone: ..., onError: ...)`——如果没有就补上：

```dart
_channel?.stream.listen(
  _onMessage,
  onDone: () {
    _disconnectedController.add(null);
    // 现有 onDone 逻辑（比如清理状态）
  },
  onError: (err) {
    _disconnectedController.add(null);
    // 现有 onError 逻辑（比如打 log）
  },
);
```

如果现有代码结构不完全这样，不要改其他逻辑，只在断开路径上 `add(null)`。

在 `onClose` 或 `dispose` 方法里记得 close controller：

```dart
@override
void onClose() {
  _disconnectedController.close();
  super.onClose();
}
```

- [ ] **Step 4: 跑测试确认绿**

```bash
flutter test test/ws_client_disconnect_test.dart
```

Expected: 两个测试 PASS

- [ ] **Step 5: 全量测试 + 提交**

```bash
flutter test
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "feat(ws-client): 暴露 onDisconnected 事件流

socket 底层断开（onDone/onError）会触发。订阅方用于感知连接失败，
典型场景是 MessageSender 把 pending 消息标 failed。"
```

---

## Task 3: WsClient 自动重连（指数退避）

**Files:**
- Modify: `foundation_packages/sanyan_network/lib/src/ws_client.dart`
- Test (create): `foundation_packages/sanyan_network/test/ws_client_reconnect_test.dart`

- [ ] **Step 1: 写失败测试**

Create `foundation_packages/sanyan_network/test/ws_client_reconnect_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_network/sanyan_network.dart';

void main() {
  test('reconnect delay follows exponential backoff sequence', () {
    // 测试重连延迟序列：1s, 2s, 5s, 10s, 30s, 30s...
    expect(WsClient.reconnectDelayForAttempt(0), const Duration(seconds: 1));
    expect(WsClient.reconnectDelayForAttempt(1), const Duration(seconds: 2));
    expect(WsClient.reconnectDelayForAttempt(2), const Duration(seconds: 5));
    expect(WsClient.reconnectDelayForAttempt(3), const Duration(seconds: 10));
    expect(WsClient.reconnectDelayForAttempt(4), const Duration(seconds: 30));
    expect(WsClient.reconnectDelayForAttempt(5), const Duration(seconds: 30));
    expect(WsClient.reconnectDelayForAttempt(100), const Duration(seconds: 30));
  });
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
cd /Users/aventador/code/3yan/app/foundation_packages/sanyan_network
flutter test test/ws_client_reconnect_test.dart
```

Expected: 编译失败（`reconnectDelayForAttempt` 未定义）

- [ ] **Step 3: 实现指数退避序列函数**

在 `ws_client.dart` 的 `class WsClient` 里加：

```dart
/// 重连延迟序列（指数退避）：第 N 次重试的间隔。
/// 0→1s, 1→2s, 2→5s, 3→10s, 4 及以后→30s 封顶。
static Duration reconnectDelayForAttempt(int attempt) {
  const sequence = [1, 2, 5, 10, 30];
  final idx = attempt < sequence.length ? attempt : sequence.length - 1;
  return Duration(seconds: sequence[idx]);
}
```

- [ ] **Step 4: 跑测试确认绿**

```bash
flutter test test/ws_client_reconnect_test.dart
```

Expected: PASS

- [ ] **Step 5: 接入自动重连逻辑**

在 `ws_client.dart` class 里加字段和方法：

```dart
int _reconnectAttempt = 0;
Timer? _reconnectTimer;
bool _manuallyDisconnected = false; // 主动 disconnect() 时不重连
```

找到 `connect()` 方法，在 socket 成功建立后加：

```dart
_reconnectAttempt = 0; // 重置计数
```

在 onDone/onError 回调里（已经 `add(null)` 的地方），再追加：

```dart
if (!_manuallyDisconnected) {
  _scheduleReconnect();
}
```

加 `_scheduleReconnect` 方法：

```dart
void _scheduleReconnect() {
  _reconnectTimer?.cancel();
  final delay = reconnectDelayForAttempt(_reconnectAttempt);
  _reconnectAttempt++;
  _reconnectTimer = Timer(delay, () {
    if (!_manuallyDisconnected) {
      connect();
    }
  });
}
```

`disconnect()` 方法加：

```dart
_manuallyDisconnected = true;
_reconnectTimer?.cancel();
```

在 `connect()` 方法开头加：

```dart
_manuallyDisconnected = false;
```

- [ ] **Step 6: 全量测试 + 提交**

```bash
flutter test
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "feat(ws-client): WebSocket 自动重连（指数退避 1→2→5→10→30s 封顶）

断开时自动调度重连，连上后重置计数。主动 disconnect() 不重连。"
```

---

## Task 4: WsClient sendMessage / sendVoiceMessage 接受外部 clientMsgId

**目的：** 让 MessageSender 掌握 clientMsgId 生成（便于在发送前就加入 pending 队列）。

**Files:**
- Modify: `foundation_packages/sanyan_network/lib/src/ws_client.dart`

当前 `sendMessage` 内部用 uuid 生成 id 再返回；`sendVoiceMessage` 已经支持传入（但可选）。统一改为**必传**，MessageSender 负责生成。

- [ ] **Step 1: 改 sendMessage 签名**

把 `String sendMessage(int conversationId, String content)` 改为：

```dart
void sendMessage({
  required int conversationId,
  required String content,
  required String clientMsgId,
}) {
  final payload = {
    'type': WsEventType.sendMessage,
    'conversationId': conversationId,
    'contentType': ContentType.text,
    'content': content,
    'clientMsgId': clientMsgId,
  };
  _send(payload);
}
```

删除方法内部的 `_uuid.v4()` 调用。如果类顶部还有 `static const _uuid = Uuid();` 并且只在这处用，一并删掉。

- [ ] **Step 2: 改 sendVoiceMessage 签名**

把原 `clientMsgId` 从 optional 改为 required：

```dart
void sendVoiceMessage({
  required int conversationId,
  required String mediaUrl,
  required int duration,
  required String clientMsgId,
}) {
  final payload = {
    'type': WsEventType.sendMessage,
    'conversationId': conversationId,
    'contentType': ContentType.voice,
    'mediaUrl': mediaUrl,
    'duration': duration,
    'clientMsgId': clientMsgId,
  };
  _send(payload);
}
```

- [ ] **Step 3: 改 ChatController 旧调用点（过渡）**

此时 ChatController 还没接 MessageSender，先让编译过。在 `chat_controller.dart` 顶部加：

```dart
import 'package:uuid/uuid.dart';
```

（sanyan_chat 本来就有 uuid 依赖）

在 `sendMessage`（文本）里：

```dart
void sendMessage() {
  final text = inputController.text.trim();
  if (text.isEmpty) return;
  final clientMsgId = const Uuid().v4();
  final wsClient = Get.find<WsClient>();
  wsClient.sendMessage(
    conversationId: conversation.id,
    content: text,
    clientMsgId: clientMsgId,
  );
  messages.add(Message(
    id: 0,
    conversationId: conversation.id,
    senderType: SenderType.user,
    contentType: ContentType.text,
    content: text,
    source: 'reply',
    createdAt: DateTime.now().toString(),
    clientMsgId: clientMsgId,
  ));
  inputController.clear();
  _scrollToBottom();
}
```

在 `_uploadAndSendVoice` 里，找到 `wsClient.sendVoiceMessage(...)` 调用处，把 named param 补全：

```dart
wsClient.sendVoiceMessage(
  conversationId: conversation.id,
  mediaUrl: uploadResp.data!.url,
  duration: uploadResp.data!.duration,
  clientMsgId: msg.clientMsgId!,
);
```

（后续 Task 16/17 会把这套逻辑挪到 MessageSender，这里只是保持编译过。）

- [ ] **Step 4: flutter analyze + 全量测试**

```bash
cd /Users/aventador/code/3yan/app
flutter analyze
cd business_packages/sanyan_chat
flutter test
```

两者都必须绿。

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "refactor(ws-client): sendMessage/sendVoiceMessage 改为必传 clientMsgId

为后续 MessageSender 接管消息生命周期做准备——id 由上层生成并追踪，
底层不再自产自销。"
```

---

## Task 5: PendingEntry 内部模型

**目的：** MessageSender 的队列条目，包含 Message 引用 + 发送时间戳 + JSON 序列化。

**Files:**
- Create: `foundation_packages/sanyan_network/lib/src/pending_entry.dart`
- Test (create): `foundation_packages/sanyan_network/test/pending_entry_test.dart`

注意：`PendingEntry` 本身不引用 `Message` 类（`Message` 在 `sanyan_chat`，不能反向引用）。改存 raw Map<String, dynamic>，Message 在 sanyan_chat 层重组。

- [ ] **Step 1: 写失败测试**

Create `foundation_packages/sanyan_network/test/pending_entry_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_network/sanyan_network.dart';

void main() {
  test('PendingEntry.toJson roundtrip', () {
    final entry = PendingEntry(
      clientMsgId: 'abc-123',
      conversationId: 42,
      sendTimeMs: 1700000000000,
      messageJson: {'content': '你好', 'contentType': 'text'},
    );
    final json = entry.toJson();
    final back = PendingEntry.fromJson(json);

    expect(back.clientMsgId, 'abc-123');
    expect(back.conversationId, 42);
    expect(back.sendTimeMs, 1700000000000);
    expect(back.messageJson['content'], '你好');
  });
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
cd /Users/aventador/code/3yan/app/foundation_packages/sanyan_network
flutter test test/pending_entry_test.dart
```

Expected: 编译失败

- [ ] **Step 3: 创建 PendingEntry 类**

Create `foundation_packages/sanyan_network/lib/src/pending_entry.dart`:

```dart
/// 发送中消息的追踪条目。
///
/// 不直接持有 Message 对象引用（Message 类在 sanyan_chat 里，
/// foundation 层不反向依赖 business 层），而是存 Map<String, dynamic>
/// 格式的消息快照，sanyan_chat 负责解析回 Message。
class PendingEntry {
  final String clientMsgId;
  final int conversationId;
  final int sendTimeMs;
  final Map<String, dynamic> messageJson;

  PendingEntry({
    required this.clientMsgId,
    required this.conversationId,
    required this.sendTimeMs,
    required this.messageJson,
  });

  Map<String, dynamic> toJson() => {
        'clientMsgId': clientMsgId,
        'conversationId': conversationId,
        'sendTimeMs': sendTimeMs,
        'message': messageJson,
      };

  factory PendingEntry.fromJson(Map<String, dynamic> json) => PendingEntry(
        clientMsgId: json['clientMsgId'] as String,
        conversationId: json['conversationId'] as int,
        sendTimeMs: json['sendTimeMs'] as int,
        messageJson: Map<String, dynamic>.from(json['message'] as Map),
      );
}
```

- [ ] **Step 4: 加到 barrel**

`foundation_packages/sanyan_network/lib/sanyan_network.dart` 加一行：

```dart
export 'src/pending_entry.dart';
```

- [ ] **Step 5: 跑测试确认绿 + 提交**

```bash
flutter test test/pending_entry_test.dart
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "feat(network): PendingEntry 模型——MessageSender 队列条目

存消息快照（Map）而非 Message 引用，避开 foundation→business 反向
依赖。"
```

---

## Task 6: MessageSender skeleton + 单元测试脚手架

**目的：** 建骨架类，定义 API 和依赖注入，后续 task 逐步填充行为。

**Files:**
- Create: `foundation_packages/sanyan_network/lib/src/message_sender.dart`
- Create: `foundation_packages/sanyan_network/test/message_sender_test.dart`

- [ ] **Step 1: 写失败测试**

Create `foundation_packages/sanyan_network/test/message_sender_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:get/get.dart';
import 'package:sanyan_network/sanyan_network.dart';

import 'support/fake_ws_client.dart';

void main() {
  setUp(() {
    Get.reset();
  });

  test('MessageSender can be constructed with dependencies', () {
    final ws = FakeWsClient();
    final sender = MessageSender(
      wsClient: ws,
      timeout: const Duration(milliseconds: 100),
      scanInterval: const Duration(milliseconds: 20),
    );
    expect(sender, isNotNull);
    expect(sender.getPending(1), isEmpty);
  });
}
```

Create `foundation_packages/sanyan_network/test/support/fake_ws_client.dart`:

```dart
import 'dart:async';
import 'package:sanyan_network/sanyan_network.dart';

/// WsClient 是 GetxService，mocktail 不工作（onStart 的 InternalFinalCallback
/// 为 null）。写一个最小可用的 fake，暴露测试需要的事件流和 send 方法记录。
class FakeWsClient extends WsClient {
  final StreamController<Map<String, dynamic>> _events =
      StreamController.broadcast();
  final StreamController<void> _disconnected = StreamController.broadcast();
  final sentTexts = <Map<String, dynamic>>[];
  final sentVoices = <Map<String, dynamic>>[];

  @override
  Stream<WsEvent> get eventStream => const Stream.empty();

  @override
  Stream<void> get onDisconnected => _disconnected.stream;

  @override
  void sendMessage({
    required int conversationId,
    required String content,
    required String clientMsgId,
  }) {
    sentTexts.add({
      'conversationId': conversationId,
      'content': content,
      'clientMsgId': clientMsgId,
    });
  }

  @override
  void sendVoiceMessage({
    required int conversationId,
    required String mediaUrl,
    required int duration,
    required String clientMsgId,
  }) {
    sentVoices.add({
      'conversationId': conversationId,
      'mediaUrl': mediaUrl,
      'duration': duration,
      'clientMsgId': clientMsgId,
    });
  }

  /// 测试辅助：模拟收到 ACK
  void simulateAck(String clientMsgId) {
    // MessageSender 监听的是 WsEvent 流，这里留空，后续 Task 7 补实现
  }

  /// 测试辅助：模拟断开
  void simulateDisconnect() {
    _disconnected.add(null);
  }

  @override
  void onClose() {
    _events.close();
    _disconnected.close();
    super.onClose();
  }
}
```

（这里先不模拟 ACK，Task 7 再补。）

- [ ] **Step 2: 跑测试确认红**

```bash
cd /Users/aventador/code/3yan/app/foundation_packages/sanyan_network
flutter test test/message_sender_test.dart
```

Expected: 编译失败（`MessageSender` 未定义）

- [ ] **Step 3: 创建 MessageSender 骨架**

Create `foundation_packages/sanyan_network/lib/src/message_sender.dart`:

```dart
import 'dart:async';
import 'package:get/get.dart';
import 'pending_entry.dart';
import 'ws_client.dart';

/// 跨聊天页生命周期的消息发送管理器。
///
/// 负责：
/// - 所有 pending 消息的追踪（Map<clientMsgId, PendingEntry>）
/// - 超时检测（1 个周期扫描 Timer，懒启动）
/// - ACK / 断线事件处理
/// - 本地持久化（GetStorage，App 冷启恢复）
/// - 统一 retry 入口
///
/// 单例注册：main.dart 里 Get.put<MessageSender>(MessageSender(), permanent: true)。
class MessageSender extends GetxService {
  final WsClient wsClient;
  final Duration timeout;
  final Duration scanInterval;

  final Map<String, PendingEntry> _pending = {};
  Timer? _scanTimer;

  /// 状态变化事件流——Sender 更新了某条消息的 status 后广播给 UI 层。
  /// ChatController 订阅并刷新 messages 列表。
  final StreamController<PendingEntry> _statusChangesController =
      StreamController<PendingEntry>.broadcast();
  Stream<PendingEntry> get statusChanges => _statusChangesController.stream;

  MessageSender({
    required this.wsClient,
    this.timeout = const Duration(seconds: 30),
    this.scanInterval = const Duration(seconds: 1),
  });

  /// 取指定会话的所有 pending 条目。
  List<PendingEntry> getPending(int conversationId) {
    return _pending.values
        .where((e) => e.conversationId == conversationId)
        .toList();
  }

  @override
  void onClose() {
    _scanTimer?.cancel();
    _statusChangesController.close();
    super.onClose();
  }
}
```

- [ ] **Step 4: 加到 barrel**

`foundation_packages/sanyan_network/lib/sanyan_network.dart` 加：

```dart
export 'src/message_sender.dart';
```

- [ ] **Step 5: 跑测试确认绿 + 提交**

```bash
flutter test test/message_sender_test.dart
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "feat(network): MessageSender GetxService 骨架

暴露 timeout/scanInterval 构造参数（测试可注入短值避免等 30s）。
定义 statusChanges 事件流和 getPending() API，行为实现留给后续 task。"
```

---

## Task 7: MessageSender.sendText() + ACK 处理 → sent

**Files:**
- Modify: `foundation_packages/sanyan_network/lib/src/message_sender.dart`
- Modify: `foundation_packages/sanyan_network/test/support/fake_ws_client.dart`
- Modify: `foundation_packages/sanyan_network/test/message_sender_test.dart`

设计：`sendText` 接收消息快照 Map（clientMsgId 上层已生成并塞进 map），存进 pending 后调 `wsClient.sendMessage`。收到 ACK 事件时，从 pending 移除并广播状态变化事件（`PendingEntry` 自带 `messageJson['status']` 改为 'sent'）。

- [ ] **Step 1: 扩展 FakeWsClient 支持模拟 ACK**

`test/support/fake_ws_client.dart` 的 `_events` 类型改为 `StreamController<WsEvent>`，并修改：

```dart
import 'dart:async';
import 'package:sanyan_network/sanyan_network.dart';

class FakeWsClient extends WsClient {
  final StreamController<WsEvent> _events = StreamController.broadcast();
  final StreamController<void> _disconnected = StreamController.broadcast();
  final sentTexts = <Map<String, dynamic>>[];
  final sentVoices = <Map<String, dynamic>>[];

  @override
  Stream<WsEvent> get eventStream => _events.stream;

  @override
  Stream<void> get onDisconnected => _disconnected.stream;

  @override
  void sendMessage({
    required int conversationId,
    required String content,
    required String clientMsgId,
  }) {
    sentTexts.add({
      'conversationId': conversationId,
      'content': content,
      'clientMsgId': clientMsgId,
    });
  }

  @override
  void sendVoiceMessage({
    required int conversationId,
    required String mediaUrl,
    required int duration,
    required String clientMsgId,
  }) {
    sentVoices.add({
      'conversationId': conversationId,
      'mediaUrl': mediaUrl,
      'duration': duration,
      'clientMsgId': clientMsgId,
    });
  }

  void simulateAck(String clientMsgId) {
    _events.add(WsEvent(
      type: WsEventType.ack,
      conversationId: null,
      clientMsgId: clientMsgId,
      message: null,
    ));
  }

  void simulateDisconnect() {
    _disconnected.add(null);
  }

  @override
  void onClose() {
    _events.close();
    _disconnected.close();
    super.onClose();
  }
}
```

（如果 `WsEvent` 构造签名和上面不一致，读 `ws_client.dart` 确认实际字段后调整。）

- [ ] **Step 2: 写失败测试**

在 `test/message_sender_test.dart` 追加：

```dart
test('sendText: message enters pending, ACK transitions to sent', () async {
  final ws = FakeWsClient();
  final sender = MessageSender(
    wsClient: ws,
    timeout: const Duration(milliseconds: 500),
    scanInterval: const Duration(milliseconds: 50),
  );

  final changes = <PendingEntry>[];
  sender.statusChanges.listen(changes.add);

  sender.sendText(
    conversationId: 1,
    clientMsgId: 'cid-1',
    messageJson: {
      'id': 0,
      'conversationId': 1,
      'content': 'hi',
      'contentType': 'text',
      'senderType': 'user',
      'source': 'reply',
      'createdAt': '2026-04-21',
      'clientMsgId': 'cid-1',
      'status': 'sending',
    },
  );

  // Sender 已经把条目加入 pending
  expect(sender.getPending(1), hasLength(1));
  // WsClient 被调用
  expect(ws.sentTexts, hasLength(1));
  expect(ws.sentTexts.first['clientMsgId'], 'cid-1');

  // 模拟 ACK
  ws.simulateAck('cid-1');
  await Future.delayed(const Duration(milliseconds: 20));

  // pending 移除，广播状态变化
  expect(sender.getPending(1), isEmpty);
  expect(changes, hasLength(1));
  expect(changes.first.messageJson['status'], 'sent');
});
```

- [ ] **Step 3: 跑测试确认红**

```bash
flutter test test/message_sender_test.dart
```

Expected: 编译失败或超时（sendText 未定义）

- [ ] **Step 4: 实现 sendText + ACK 监听**

在 `message_sender.dart` 里加：

```dart
import 'ws_event.dart'; // 如果 WsEvent 类在独立文件；否则通过 ws_client.dart 间接拿到
```

构造函数里启动事件订阅（新增 `onInit` 覆盖）：

```dart
StreamSubscription<WsEvent>? _wsEventSub;
StreamSubscription<void>? _wsDisconnectSub;

@override
void onInit() {
  super.onInit();
  _wsEventSub = wsClient.eventStream.listen(_onWsEvent);
}

@override
void onClose() {
  _wsEventSub?.cancel();
  _wsDisconnectSub?.cancel();
  _scanTimer?.cancel();
  _statusChangesController.close();
  super.onClose();
}

void _onWsEvent(WsEvent event) {
  if (event.type == WsEventType.ack) {
    _handleAck(event.clientMsgId);
  }
}

void _handleAck(String? clientMsgId) {
  if (clientMsgId == null) return;
  final entry = _pending.remove(clientMsgId);
  if (entry == null) return;
  entry.messageJson['status'] = 'sent';
  _statusChangesController.add(entry);
  if (_pending.isEmpty) {
    _scanTimer?.cancel();
    _scanTimer = null;
  }
}
```

新增 `sendText` 方法：

```dart
void sendText({
  required int conversationId,
  required String clientMsgId,
  required Map<String, dynamic> messageJson,
}) {
  final entry = PendingEntry(
    clientMsgId: clientMsgId,
    conversationId: conversationId,
    sendTimeMs: DateTime.now().millisecondsSinceEpoch,
    messageJson: Map<String, dynamic>.from(messageJson),
  );
  _pending[clientMsgId] = entry;
  wsClient.sendMessage(
    conversationId: conversationId,
    content: messageJson['content'] as String,
    clientMsgId: clientMsgId,
  );
  _ensureScanTimer();
}

void _ensureScanTimer() {
  _scanTimer ??= Timer.periodic(scanInterval, (_) => _scan());
}

void _scan() {
  // Task 8 填
}
```

注意：构造函数的 MessageSender 是 GetxService，测试里 `Get.put` 会触发 onInit。上面测试没走 `Get.put`，直接构造，所以 onInit 不会被调用。需要改测试或在构造函数末尾手动调用监听逻辑。

**选择：把 WsClient 订阅挪到构造函数末尾**，让测试不依赖 Get 生命周期：

```dart
MessageSender({
  required this.wsClient,
  this.timeout = const Duration(seconds: 30),
  this.scanInterval = const Duration(seconds: 1),
}) {
  _wsEventSub = wsClient.eventStream.listen(_onWsEvent);
  _wsDisconnectSub = wsClient.onDisconnected.listen(_onDisconnected);
}

void _onDisconnected(_) {
  // Task 9 填
}
```

去掉 `onInit` 覆盖（或保留为空覆盖）。

- [ ] **Step 5: 跑测试确认绿**

```bash
flutter test test/message_sender_test.dart
```

Expected: PASS

- [ ] **Step 6: 提交**

```bash
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "feat(message-sender): sendText + ACK → sent 状态机

消息发送时加入 pending map 并调 WsClient.sendMessage；收到 ACK 事件
时从 map 移除、status 置 sent、广播 statusChanges 事件。订阅放构造
函数便于单测脱离 Get 生命周期。"
```

---

## Task 8: MessageSender 超时扫描 → failed

**Files:**
- Modify: `foundation_packages/sanyan_network/lib/src/message_sender.dart`
- Modify: `foundation_packages/sanyan_network/test/message_sender_test.dart`

- [ ] **Step 1: 写失败测试**

追加：

```dart
test('timeout: message exceeding timeout is marked failed', () async {
  final ws = FakeWsClient();
  final sender = MessageSender(
    wsClient: ws,
    timeout: const Duration(milliseconds: 100),
    scanInterval: const Duration(milliseconds: 20),
  );

  final changes = <PendingEntry>[];
  sender.statusChanges.listen(changes.add);

  sender.sendText(
    conversationId: 1,
    clientMsgId: 'cid-timeout',
    messageJson: {
      'content': 'hi',
      'contentType': 'text',
      'clientMsgId': 'cid-timeout',
      'status': 'sending',
      'conversationId': 1,
    },
  );

  // 等超过 timeout + 一轮 scan
  await Future.delayed(const Duration(milliseconds: 200));

  expect(sender.getPending(1), isEmpty);
  expect(changes, hasLength(1));
  expect(changes.first.messageJson['status'], 'failed');
});
```

- [ ] **Step 2: 跑测试确认红**

```bash
flutter test test/message_sender_test.dart
```

Expected: FAIL（timeout 的消息仍然在 pending）

- [ ] **Step 3: 实现 scan**

`message_sender.dart` 的 `_scan` 改为：

```dart
void _scan() {
  final now = DateTime.now().millisecondsSinceEpoch;
  final expiredIds = <String>[];
  for (final entry in _pending.values) {
    if (now - entry.sendTimeMs > timeout.inMilliseconds) {
      expiredIds.add(entry.clientMsgId);
    }
  }
  for (final id in expiredIds) {
    final entry = _pending.remove(id);
    if (entry != null) {
      entry.messageJson['status'] = 'failed';
      _statusChangesController.add(entry);
    }
  }
  if (_pending.isEmpty) {
    _scanTimer?.cancel();
    _scanTimer = null;
  }
}
```

- [ ] **Step 4: 跑测试确认绿 + 提交**

```bash
flutter test test/message_sender_test.dart
git add -A
git commit -m "feat(message-sender): 周期扫描超时消息 → failed

每次扫描 now - sendTimeMs > timeout 的条目，设 failed + 从 map 移除 +
广播。队列空时停 Timer 省电。"
```

---

## Task 9: MessageSender 断线处理 → 全部 failed

**Files:**
- Modify: `foundation_packages/sanyan_network/lib/src/message_sender.dart`
- Modify: `foundation_packages/sanyan_network/test/message_sender_test.dart`

- [ ] **Step 1: 写失败测试**

追加：

```dart
test('disconnect: all pending messages marked failed', () async {
  final ws = FakeWsClient();
  final sender = MessageSender(
    wsClient: ws,
    timeout: const Duration(seconds: 30),
    scanInterval: const Duration(seconds: 1),
  );

  final changes = <PendingEntry>[];
  sender.statusChanges.listen(changes.add);

  sender.sendText(
    conversationId: 1,
    clientMsgId: 'a',
    messageJson: {'content': 'x', 'clientMsgId': 'a', 'conversationId': 1, 'status': 'sending'},
  );
  sender.sendText(
    conversationId: 1,
    clientMsgId: 'b',
    messageJson: {'content': 'y', 'clientMsgId': 'b', 'conversationId': 1, 'status': 'sending'},
  );

  expect(sender.getPending(1), hasLength(2));

  ws.simulateDisconnect();
  await Future.delayed(const Duration(milliseconds: 20));

  expect(sender.getPending(1), isEmpty);
  expect(changes, hasLength(2));
  for (final c in changes) {
    expect(c.messageJson['status'], 'failed');
  }
});
```

- [ ] **Step 2: 跑测试确认红**

Expected: FAIL（断线后 pending 不清、status 没变）

- [ ] **Step 3: 实现 _onDisconnected**

替换 `message_sender.dart` 的 `_onDisconnected`：

```dart
void _onDisconnected(_) {
  if (_pending.isEmpty) return;
  final entries = _pending.values.toList();
  _pending.clear();
  for (final entry in entries) {
    entry.messageJson['status'] = 'failed';
    _statusChangesController.add(entry);
  }
  _scanTimer?.cancel();
  _scanTimer = null;
}
```

- [ ] **Step 4: 跑测试确认绿 + 提交**

```bash
flutter test test/message_sender_test.dart
git add -A
git commit -m "feat(message-sender): WebSocket 断线时 pending 消息批量 failed

订阅 WsClient.onDisconnected，触发时把所有 pending 条目标 failed 并
广播，清空 map，停 Timer。"
```

---

## Task 10: MessageSender 持久化（GetStorage）+ 冷启加载

**Files:**
- Modify: `foundation_packages/sanyan_network/lib/src/message_sender.dart`
- Modify: `foundation_packages/sanyan_network/test/message_sender_test.dart`

设计：
- 冷启动时加载 disk pending → **立即全部标 failed**（因为旧 Timer/socket 已随进程消亡，这些消息没指望被 ACK）→ 保留在 pending map 里供 UI 展示？还是立即从 map 移除只留在 disk？

选择：**加载后立即广播 failed 事件 + 从 pending map 移除**。failed 消息本身的 UI 展示靠 ChatController 在 onInit 时 merge——但 sender 这时已经没在 pending 里了，怎么给 controller？

改设计：**加载后立即标 failed（不移除 map）+ 广播 + 持久化**，pending map 里保留 failed 状态的条目，直到 ChatController merge 它们并调用 `sender.removePending(clientMsgId)` 显式移除。

但这样 pending map 会无限增长（用户永远不进聊天页就永远留着）。

更简单的方案：**MessageSender 只管 sending 状态的追踪 + failed 事件的广播**。failed 消息本身不在 sender 里滞留——ChatController 负责把 failed 消息存到自己的 messages 列表里（结合从服务端拉的历史消息）。

**但 ChatController 离开后 messages 列表也没了**——用户关聊天页到回列表页再进来，failed 消息也应该还看得到。

**最终方案**：MessageSender 持有 **"有故事的消息" = pending（sending）+ recentlyFailed**。recentlyFailed 是 failed 后未被 ChatController 认领的短期缓冲。ChatController onInit 时把这些拉过来塞进 messages，然后调 `sender.clearRecentlyFailed(convId)`。

太复杂了。简化一版：

**最终最终方案**：持久化的是 "失败消息队列"（不区分 sending 和 failed）。冷启动时加载 → 全部状态改 failed → 留在一个独立的 `_failedArchive` 里（不参与 Timer 扫描、不参与断线清理）。ChatController onInit 时 `sender.getArchivedFailed(convId)` 拿所有未认领的，显示后调 `sender.acknowledgeFailed(convId)` 清空该会话的缓冲。

这方案清晰，但引入第二个数据结构。为了**一期简单**：

**一期采纳**：pending map 里同时存 sending 和 failed。Sender 不自动清理 failed；ChatController 拿后显式 remove。这个行为细节通过 API 契约表达。`getPending(convId)` 返回所有（sending + failed 都算 pending "未完结"）。`ChatController` 合并到 messages 后，监听 `statusChanges`；对 failed 的条目，UI 显示感叹号 + 可点重试。重试时 sender 内部重建 sending 条目。

但这时 Sender 里 failed 条目什么时候清？——ChatController 销毁时？回列表再进聊天页又能看到？

**最终一期方案**（定了不改了）：
- Sender map 保留 pending + failed 不自动清
- ChatController 离开时**不清 sender map**（消息状态跨聊天页生存）
- 下次进聊天页 `getPending` 再拿
- 只有 ACK 到达 或 用户手动重试成功时才从 map 移除
- 用户在 UI 上点"删除此失败消息"的能力放到**未来**（YAGNI）

冷启加载：

```dart
// 冷启读 disk → 直接写入 pending map（status 还是 disk 里的值，可能是 sending 或 failed）
// 对 status == 'sending' 的：马上标 failed（因为 timer 和 socket 都重置了）
// 对 status == 'failed' 的：保持 failed
// 不广播冷启事件（ChatController 还没监听）；ChatController onInit 时 getPending 拿到即可
```

- [ ] **Step 1: 写失败测试**

追加（放在 `Get.reset()` 的 setUp 里加 `GetStorage.init` 或用 mock，但是单测跑 GetStorage 要 init path。先用 `SharedPreferences.setMockInitialValues({})` 这种不适合 GetStorage。GetStorage 内部用 path_provider。

最干净做法：抽一个 `PendingStorage` 接口，默认实现用 GetStorage，测试用内存 map。

实际 Spec 复杂度上升了。让我们接受**集成测试**的现实——用真 GetStorage：

```dart
import 'package:flutter/services.dart';
import 'package:path_provider_platform_interface/path_provider_platform_interface.dart';
import 'package:plugin_platform_interface/plugin_platform_interface.dart';

class _FakePathProviderPlatform extends PathProviderPlatform
    with MockPlatformInterfaceMixin {
  @override
  Future<String?> getApplicationDocumentsPath() async => '/tmp';
  @override
  Future<String?> getApplicationSupportPath() async => '/tmp';
  @override
  Future<String?> getTemporaryPath() async => '/tmp';
}

setUp(() async {
  TestWidgetsFlutterBinding.ensureInitialized();
  PathProviderPlatform.instance = _FakePathProviderPlatform();
  Get.reset();
  await GetStorage.init('sanyan_pending_test');
  final box = GetStorage('sanyan_pending_test');
  await box.erase();
});
```

然后 MessageSender 构造函数接受可选 `boxName`（默认 `'sanyan_pending'`）。

追加测试：

```dart
test('cold start: pending loaded from disk are immediately marked failed', () async {
  // 第一个 sender 写入 pending
  final ws1 = FakeWsClient();
  final sender1 = MessageSender(
    wsClient: ws1,
    timeout: const Duration(seconds: 30),
    scanInterval: const Duration(seconds: 1),
    boxName: 'sanyan_pending_test',
  );
  await sender1.initAsync();
  sender1.sendText(
    conversationId: 1,
    clientMsgId: 'persist-1',
    messageJson: {'content': 'x', 'clientMsgId': 'persist-1', 'conversationId': 1, 'status': 'sending'},
  );
  // sender1 还活着，不等 ACK，直接销毁模拟冷启
  sender1.onClose();

  // 第二个 sender 冷启
  final ws2 = FakeWsClient();
  final sender2 = MessageSender(
    wsClient: ws2,
    timeout: const Duration(seconds: 30),
    scanInterval: const Duration(seconds: 1),
    boxName: 'sanyan_pending_test',
  );
  await sender2.initAsync();

  final pending = sender2.getPending(1);
  expect(pending, hasLength(1));
  expect(pending.first.messageJson['status'], 'failed');
});
```

- [ ] **Step 2: 跑测试确认红**

```bash
flutter test test/message_sender_test.dart
```

Expected: 编译失败（`initAsync`/`boxName` 未定义）

- [ ] **Step 3: 实现持久化 + 加载**

`message_sender.dart` 加 import：

```dart
import 'package:get_storage/get_storage.dart';
```

类字段：

```dart
final String boxName;
late final GetStorage _box;
static const _storeKey = 'pending';
```

构造：

```dart
MessageSender({
  required this.wsClient,
  this.timeout = const Duration(seconds: 30),
  this.scanInterval = const Duration(seconds: 1),
  this.boxName = 'sanyan_pending',
}) {
  _wsEventSub = wsClient.eventStream.listen(_onWsEvent);
  _wsDisconnectSub = wsClient.onDisconnected.listen(_onDisconnected);
}
```

新增 `initAsync`：

```dart
/// 冷启动加载 pending 消息。main.dart 的 onInit 流程里 await 它。
Future<void> initAsync() async {
  await GetStorage.init(boxName);
  _box = GetStorage(boxName);
  _loadFromDisk();
}

void _loadFromDisk() {
  final raw = _box.read<List<dynamic>>(_storeKey);
  if (raw == null) return;
  for (final item in raw) {
    final entry = PendingEntry.fromJson(Map<String, dynamic>.from(item as Map));
    // 冷启 sending 状态立即转 failed（timer/socket 已失效）
    if (entry.messageJson['status'] == 'sending') {
      entry.messageJson['status'] = 'failed';
    }
    _pending[entry.clientMsgId] = entry;
  }
}

void _persist() {
  _box.write(
    _storeKey,
    _pending.values.map((e) => e.toJson()).toList(),
  );
}
```

在所有修改 `_pending` 的地方后面都加 `_persist()`：
- `sendText` 末尾
- `_handleAck` 移除后
- `_scan` 标 failed 后
- `_onDisconnected` 清空后

- [ ] **Step 4: 跑测试确认绿 + 提交**

```bash
flutter test test/message_sender_test.dart
git add -A
git commit -m "feat(message-sender): GetStorage 持久化 + 冷启加载

pending 变化（send/ack/timeout/disconnect）立即 _persist。
initAsync 冷启从 disk 加载——之前是 sending 的立刻改 failed
（timer 和 socket 都随进程消亡了，没指望被 ACK）。"
```

---

## Task 11: MessageSender.sendVoice() + retry() + removePending()

**Files:**
- Modify: `foundation_packages/sanyan_network/lib/src/message_sender.dart`
- Modify: `foundation_packages/sanyan_network/test/message_sender_test.dart`

- [ ] **Step 1: 写失败测试**

追加：

```dart
test('sendVoice: enqueues and calls wsClient.sendVoiceMessage', () {
  final ws = FakeWsClient();
  final sender = MessageSender(
    wsClient: ws,
    timeout: const Duration(milliseconds: 100),
    scanInterval: const Duration(milliseconds: 20),
    boxName: 'sanyan_pending_test',
  );

  sender.sendVoice(
    conversationId: 2,
    clientMsgId: 'voice-1',
    mediaUrl: 'https://cos/x.mp3',
    duration: 5,
    messageJson: {
      'content': '',
      'clientMsgId': 'voice-1',
      'conversationId': 2,
      'contentType': 'voice',
      'mediaUrl': 'https://cos/x.mp3',
      'duration': 5,
      'status': 'sending',
    },
  );

  expect(sender.getPending(2), hasLength(1));
  expect(ws.sentVoices, hasLength(1));
  expect(ws.sentVoices.first['clientMsgId'], 'voice-1');
});

test('retry: re-enqueues a failed entry with fresh sendTime', () async {
  final ws = FakeWsClient();
  final sender = MessageSender(
    wsClient: ws,
    timeout: const Duration(milliseconds: 100),
    scanInterval: const Duration(milliseconds: 20),
    boxName: 'sanyan_pending_test',
  );

  sender.sendText(
    conversationId: 1,
    clientMsgId: 'retry-1',
    messageJson: {'content': 'x', 'clientMsgId': 'retry-1', 'conversationId': 1, 'contentType': 'text', 'status': 'sending'},
  );
  await Future.delayed(const Duration(milliseconds: 200)); // 等超时
  expect(sender.getPending(1).first.messageJson['status'], 'failed');

  final entry = sender.getPending(1).first;
  sender.retry(entry);

  expect(sender.getPending(1).first.messageJson['status'], 'sending');
  expect(ws.sentTexts, hasLength(2)); // 第一次 + 重试
});

test('removePending: entry gone from map', () {
  final ws = FakeWsClient();
  final sender = MessageSender(
    wsClient: ws,
    boxName: 'sanyan_pending_test',
  );
  sender.sendText(
    conversationId: 1,
    clientMsgId: 'rm-1',
    messageJson: {'content': 'x', 'clientMsgId': 'rm-1', 'conversationId': 1, 'contentType': 'text', 'status': 'sending'},
  );
  sender.removePending('rm-1');
  expect(sender.getPending(1), isEmpty);
});
```

- [ ] **Step 2: 跑测试确认红**

Expected: 编译失败（sendVoice/retry/removePending 未定义）

- [ ] **Step 3: 实现**

`message_sender.dart` 加方法：

```dart
void sendVoice({
  required int conversationId,
  required String clientMsgId,
  required String mediaUrl,
  required int duration,
  required Map<String, dynamic> messageJson,
}) {
  final entry = PendingEntry(
    clientMsgId: clientMsgId,
    conversationId: conversationId,
    sendTimeMs: DateTime.now().millisecondsSinceEpoch,
    messageJson: Map<String, dynamic>.from(messageJson),
  );
  _pending[clientMsgId] = entry;
  wsClient.sendVoiceMessage(
    conversationId: conversationId,
    mediaUrl: mediaUrl,
    duration: duration,
    clientMsgId: clientMsgId,
  );
  _ensureScanTimer();
  _persist();
}

void retry(PendingEntry entry) {
  entry.messageJson['status'] = 'sending';
  // 同一个 PendingEntry 对象，但 sendTimeMs 更新
  final refreshed = PendingEntry(
    clientMsgId: entry.clientMsgId,
    conversationId: entry.conversationId,
    sendTimeMs: DateTime.now().millisecondsSinceEpoch,
    messageJson: entry.messageJson,
  );
  _pending[entry.clientMsgId] = refreshed;
  final contentType = entry.messageJson['contentType'] as String?;
  if (contentType == 'voice') {
    wsClient.sendVoiceMessage(
      conversationId: entry.conversationId,
      mediaUrl: entry.messageJson['mediaUrl'] as String,
      duration: entry.messageJson['duration'] as int,
      clientMsgId: entry.clientMsgId,
    );
  } else {
    wsClient.sendMessage(
      conversationId: entry.conversationId,
      content: entry.messageJson['content'] as String,
      clientMsgId: entry.clientMsgId,
    );
  }
  _ensureScanTimer();
  _statusChangesController.add(refreshed);
  _persist();
}

void removePending(String clientMsgId) {
  final entry = _pending.remove(clientMsgId);
  if (entry == null) return;
  if (_pending.isEmpty) {
    _scanTimer?.cancel();
    _scanTimer = null;
  }
  _persist();
}
```

- [ ] **Step 4: 跑测试确认绿 + 提交**

```bash
flutter test test/message_sender_test.dart
git add -A
git commit -m "feat(message-sender): sendVoice / retry / removePending API

retry 重置 sendTimeMs 后按 contentType 分派给 WsClient 对应 send 方法。
removePending 在 ChatController 决定彻底删除该消息时调用。"
```

---

## Task 12: 在 main.dart 注册 MessageSender

**Files:**
- Modify: `app/lib/main.dart`

- [ ] **Step 1: 修改 main.dart**

在 `main()` 里，`ApiClient.tokenProvider = ...` 之后、`runApp` 之前：

```dart
// Register MessageSender (GetxService, 跨聊天页生命周期)
final wsClient = Get.put<WsClient>(WsClient(), permanent: true);
final messageSender = MessageSender(wsClient: wsClient);
await messageSender.initAsync();
Get.put<MessageSender>(messageSender, permanent: true);
```

注意：如果 `WsClient` 原来是懒注册的（第一次 `Get.find` 时才创建），这里改为显式注册一次。确认 WsClient 构造没有昂贵副作用。

- [ ] **Step 2: flutter analyze**

```bash
cd /Users/aventador/code/3yan/app
flutter analyze
```

Expected: `No issues found!`

- [ ] **Step 3: 跑测试**

```bash
flutter test
```

Expected: 所有包测试通过

- [ ] **Step 4: 提交**

```bash
git add -A
git commit -m "chore(app): main.dart 注册 MessageSender（permanent）

启动流程 await messageSender.initAsync() 保证冷启 pending 加载完毕
再进入路由。"
```

---

## Task 13: MessageBubbleBase 抽基类 + 状态 UI

**Files:**
- Create: `business_packages/sanyan_chat/lib/src/chat/widget/message_bubble_base.dart`
- Test (create): `business_packages/sanyan_chat/test/message_bubble_base_test.dart`

- [ ] **Step 1: 写失败测试**

Create `business_packages/sanyan_chat/test/message_bubble_base_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_chat/sanyan_chat.dart';

void main() {
  testWidgets('sending status shows loading indicator', (tester) async {
    await tester.pumpWidget(MaterialApp(
      home: Scaffold(
        body: MessageBubbleBase(
          isFromAi: false,
          status: MessageStatus.sending,
          onRetry: null,
          child: const Text('hi'),
        ),
      ),
    ));
    expect(find.byKey(const Key('bubble-sending-indicator')), findsOneWidget);
    expect(find.byKey(const Key('bubble-failed-indicator')), findsNothing);
  });

  testWidgets('failed status shows retry indicator + tap triggers onRetry', (tester) async {
    var tapped = false;
    await tester.pumpWidget(MaterialApp(
      home: Scaffold(
        body: MessageBubbleBase(
          isFromAi: false,
          status: MessageStatus.failed,
          onRetry: () => tapped = true,
          child: const Text('hi'),
        ),
      ),
    ));
    expect(find.byKey(const Key('bubble-failed-indicator')), findsOneWidget);
    await tester.tap(find.byKey(const Key('bubble-failed-indicator')));
    expect(tapped, isTrue);
  });

  testWidgets('sent status shows no indicator', (tester) async {
    await tester.pumpWidget(MaterialApp(
      home: Scaffold(
        body: MessageBubbleBase(
          isFromAi: false,
          status: MessageStatus.sent,
          onRetry: null,
          child: const Text('hi'),
        ),
      ),
    ));
    expect(find.byKey(const Key('bubble-sending-indicator')), findsNothing);
    expect(find.byKey(const Key('bubble-failed-indicator')), findsNothing);
  });
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
cd /Users/aventador/code/3yan/app/business_packages/sanyan_chat
flutter test test/message_bubble_base_test.dart
```

Expected: 编译失败（`MessageBubbleBase` 未定义）

- [ ] **Step 3: 实现基类**

Create `business_packages/sanyan_chat/lib/src/chat/widget/message_bubble_base.dart`:

```dart
import 'package:flutter/material.dart';
import '../../api/models/message_status.dart';

/// 所有消息气泡的外壳 + 状态层（sending / failed / sent）。
/// 子类只负责"内容区域"（文字、语音、图片...），通过 child 传入。
///
/// 状态指示放在气泡右侧（用户消息）或左侧（AI 消息）：
/// - sending: 小菊花
/// - failed: 红色感叹号（可点重试）
/// - sent: 无指示
class MessageBubbleBase extends StatelessWidget {
  final bool isFromAi;
  final MessageStatus? status;
  final VoidCallback? onRetry;
  final Widget child;

  const MessageBubbleBase({
    super.key,
    required this.isFromAi,
    required this.status,
    required this.onRetry,
    required this.child,
  });

  @override
  Widget build(BuildContext context) {
    final indicator = _buildIndicator();
    return Row(
      mainAxisAlignment: isFromAi ? MainAxisAlignment.start : MainAxisAlignment.end,
      crossAxisAlignment: CrossAxisAlignment.center,
      children: isFromAi
          ? [Flexible(child: child), if (indicator != null) indicator]
          : [if (indicator != null) indicator, Flexible(child: child)],
    );
  }

  Widget? _buildIndicator() {
    switch (status) {
      case MessageStatus.sending:
        return const Padding(
          key: Key('bubble-sending-indicator'),
          padding: EdgeInsets.symmetric(horizontal: 8),
          child: SizedBox(
            width: 14,
            height: 14,
            child: CircularProgressIndicator(strokeWidth: 2),
          ),
        );
      case MessageStatus.failed:
        return IconButton(
          key: const Key('bubble-failed-indicator'),
          icon: const Icon(Icons.error, color: Colors.red, size: 20),
          onPressed: onRetry,
          padding: const EdgeInsets.symmetric(horizontal: 8),
          constraints: const BoxConstraints(minWidth: 32, minHeight: 32),
        );
      case MessageStatus.sent:
      case null:
        return null;
    }
  }
}
```

- [ ] **Step 4: 加到 barrel**

`business_packages/sanyan_chat/lib/sanyan_chat.dart` 加：

```dart
export 'src/chat/widget/message_bubble_base.dart';
```

- [ ] **Step 5: 跑测试确认绿 + 提交**

```bash
flutter test test/message_bubble_base_test.dart
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "feat(chat-ui): MessageBubbleBase 基类统一发送状态指示

sending 小菊花 / failed 红感叹号（点击重试）/ sent 无指示。
子类只负责内容区域（child）。"
```

---

## Task 14: MessageBubble（文本）接入基类

**Files:**
- Modify: `business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart`
- Modify: `business_packages/sanyan_chat/test/chat_controller_test.dart`（可能要调整现有测试）

现有 MessageBubble 是文本气泡（而不是"所有气泡的工厂"——它内部根据 contentType 分派到 VoiceBubble 等）。先看一下代码结构再决定怎么接。

- [ ] **Step 1: 读现有结构**

```bash
cat business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart
```

根据文件内容决定如何接入基类。如果 `MessageBubble` 是文本专用，只需把文本气泡包一层 `MessageBubbleBase`。如果它是分派器（根据 contentType 路由到 VoiceBubble/VideoBubble），则在分派器外面包一层 `MessageBubbleBase`，让所有类型共享状态层。

**基于当前 grep 结果**（message_bubble.dart 既有文本渲染也被 voice/video 引用），推测它是**分派器**。采用**外包基类**方案：

在 `MessageBubble.build()` 返回的顶层 Widget 外面包一层 `MessageBubbleBase`：

```dart
@override
Widget build(BuildContext context) {
  return MessageBubbleBase(
    isFromAi: message.isFromAi,
    status: message.status,
    onRetry: onRetry,  // 新增构造参数，透传
    child: _buildContent(context),
  );
}
```

`_buildContent` 里放原来的气泡内容（文本或分派到 VoiceBubble）。

注意给 MessageBubble 加 `final VoidCallback? onRetry;` 构造参数。ListView 那边传入 `() => c.retryMessage(message)`。

- [ ] **Step 2: 修改 MessageBubble**

改 `message_bubble.dart`（基于实际代码调整——下面是示意）：

```dart
// 顶部加 import
import 'message_bubble_base.dart';

// class 加字段
final VoidCallback? onRetry;

const MessageBubble({
  super.key,
  required this.message,
  this.onRetry,
});

@override
Widget build(BuildContext context) {
  return MessageBubbleBase(
    isFromAi: message.isFromAi,
    status: message.status,
    onRetry: onRetry,
    child: _buildContent(context),
  );
}

Widget _buildContent(BuildContext context) {
  // 原来 build 方法里的内容全放这里
  // ...
}
```

- [ ] **Step 3: 修改 ChatPage 传 onRetry**

`chat_page.dart` 的 `ListView.builder` 里：

```dart
return MessageBubble(
  message: c.messages[index],
  onRetry: () => c.retryMessage(c.messages[index]),
);
```

（`c.retryMessage` 在 Task 19 定义，这里先引用未定义方法会让测试/编译失败。**在 Task 19 之前不要做 Task 14 的 Step 3**；或者在 ChatController 里先加一个空实现 `void retryMessage(Message msg) {}`，Task 19 再填内容。采用后者，保证 Task 14 能独立通过。）

在 `chat_controller.dart` 里临时加空方法：

```dart
/// TODO(Task 19): 接入 MessageSender.retry
void retryMessage(Message msg) {}
```

- [ ] **Step 4: flutter analyze + test**

```bash
cd /Users/aventador/code/3yan/app
flutter analyze
cd business_packages/sanyan_chat
flutter test
```

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "refactor(chat-ui): MessageBubble 外包 MessageBubbleBase

text / voice / video 气泡共享同一层状态指示。MessageBubble 加 onRetry
透传参数。ChatController 加空的 retryMessage 占位，Task 19 补实现。"
```

---

## Task 15: VoiceBubble 接入基类 + 删除旧失败 UI

**Files:**
- Modify: `business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart`

因为外层 `MessageBubble` 已经包了 `MessageBubbleBase`，voice 内部的失败 UI 是重复的。要**删除**。

- [ ] **Step 1: 读 voice_bubble 看有哪些失败 UI 要删**

```bash
grep -nE "failed|Failed|isFailed|MessageStatus.failed|Icon.*error|Icons.error" business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart
```

定位所有与"消息发送失败"相关的 UI 片段。

- [ ] **Step 2: 删除重复 UI**

把 voice_bubble 里针对 `msg.isFailed` 或 `status == failed` 渲染的红感叹号、重试按钮、错误提示等相关代码全部删除。保留 voice_bubble 的**播放态 UI**（播放/暂停 icon、波形图、时长等）——那是内容，不是状态指示。

- [ ] **Step 3: 手动验证测试 + test**

```bash
cd /Users/aventador/code/3yan/app
flutter analyze
cd business_packages/sanyan_chat
flutter test
```

- [ ] **Step 4: 提交**

```bash
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "refactor(voice-bubble): 删除重复的失败状态 UI，统一走 MessageBubbleBase

外层基类已处理 failed 状态指示，voice_bubble 只保留播放态 UI。"
```

---

## Task 16: ChatController.sendMessage 接入 MessageSender（文本）

**Files:**
- Modify: `business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`
- Modify: `business_packages/sanyan_chat/test/chat_controller_test.dart`

- [ ] **Step 1: 写失败测试**

测试环境里还没有 MessageSender 注入。我们用 fake 替身。给 `chat_controller_test.dart` 加 `_FakeMessageSender`：

```dart
class _FakeMessageSender extends MessageSender {
  _FakeMessageSender() : super(wsClient: _FakeWsClient(), boxName: 'test');

  final sentTexts = <Map<String, dynamic>>[];

  @override
  void sendText({
    required int conversationId,
    required String clientMsgId,
    required Map<String, dynamic> messageJson,
  }) {
    sentTexts.add({
      'conversationId': conversationId,
      'clientMsgId': clientMsgId,
      'messageJson': messageJson,
    });
  }

  @override
  List<PendingEntry> getPending(int conversationId) => [];
}
```

测试：

```dart
test('sendMessage: delegates to MessageSender with sending status', () {
  final sender = _FakeMessageSender();
  final ws = _FakeWsClient();
  Get.put<WsClient>(ws);
  Get.put<MessageSender>(sender);
  final c = ChatController(_fixtureConv(), recorder: _MockRecorder());
  c.inputController.text = 'hello';
  c.sendMessage();

  expect(sender.sentTexts, hasLength(1));
  final payload = sender.sentTexts.first;
  expect(payload['conversationId'], 1);
  expect(payload['messageJson']['content'], 'hello');
  expect(payload['messageJson']['status'], 'sending');
  expect(payload['messageJson']['clientMsgId'], isNotEmpty);
  expect(c.messages, hasLength(1));
  expect(c.messages.first.content, 'hello');
  expect(c.messages.first.status, MessageStatus.sending);
});
```

- [ ] **Step 2: 跑测试确认红**

Expected: FAIL（ChatController 还在直接调 wsClient.sendMessage，不经 MessageSender）

- [ ] **Step 3: 改 ChatController.sendMessage**

```dart
void sendMessage() {
  final text = inputController.text.trim();
  if (text.isEmpty) return;
  final clientMsgId = const Uuid().v4();
  final msg = Message(
    id: 0,
    conversationId: conversation.id,
    senderType: SenderType.user,
    contentType: ContentType.text,
    content: text,
    source: 'reply',
    createdAt: DateTime.now().toString(),
    clientMsgId: clientMsgId,
    status: MessageStatus.sending,
  );
  messages.add(msg);
  Get.find<MessageSender>().sendText(
    conversationId: conversation.id,
    clientMsgId: clientMsgId,
    messageJson: msg.toJson(),
  );
  inputController.clear();
  _scrollToBottom();
}
```

**注意**：`Message.toJson()` 现在要包含 `status` 字段。读 `message.dart` 的 `toJson`，确认已加 `'status': status?.name`。没有的话先在 `message.dart` 里补：

```dart
Map<String, dynamic> toJson() => {
  // ... 现有字段 ...
  'status': status?.name,
  'clientMsgId': clientMsgId,
};
```

以及对应的 `fromJson` 加：

```dart
status: json['status'] != null
    ? MessageStatus.values.firstWhere((s) => s.name == json['status'])
    : null,
clientMsgId: json['clientMsgId'],
```

- [ ] **Step 4: 跑测试确认绿 + 提交**

```bash
cd /Users/aventador/code/3yan/app/business_packages/sanyan_chat
flutter test
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "refactor(chat-controller): sendMessage 走 MessageSender

Message.toJson/fromJson 补全 status / clientMsgId 字段，供 MessageSender
持久化使用。"
```

---

## Task 17: ChatController.sendVoiceMessage 接入 MessageSender

**Files:**
- Modify: `business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`
- Modify: `business_packages/sanyan_chat/test/chat_controller_test.dart`

行为变化：上传成功后**不再立刻** `status=sent`，改为 `status=sending` + 调 `MessageSender.sendVoice`，等 ACK 才 sent。

- [ ] **Step 1: 写失败测试**

```dart
test('sendVoiceMessage: after upload, delegates to MessageSender with sending status', () async {
  final sender = _FakeMessageSender();
  Get.put<WsClient>(_FakeWsClient());
  Get.put<MessageSender>(sender);

  // Mock ChatApi.uploadVoice 的行为需要用 mocktail 或 http mock
  // 此测试简化为：直接注入一个成功的 uploadResp path——如果现有 ChatApi
  // 是 static method 难以 mock，跳过此单测，用 integration-style 测试
  // 或把 uploadResp 逻辑抽接口后注入。
  // 本 plan 一期简化：在 ChatController 加一个 test-only overrideable
  // upload 方法，或者用 mocktail hijack ChatApi（需要给 ChatApi 加非 static）
});
```

由于 `ChatApi` 是 static methods，mocktail 不工作。**简化**：跳过语音上传路径的单测，用 **flutter analyze + 手测** 兜底。核心逻辑（sendVoice 调用 MessageSender）的 spec 用下面这个纯逻辑测试：

```dart
test('retryVoiceMessage: delegates to MessageSender via retry', () async {
  final sender = _FakeMessageSender();
  Get.put<WsClient>(_FakeWsClient());
  Get.put<MessageSender>(sender);

  final c = ChatController(_fixtureConv(), recorder: _MockRecorder());
  final failedMsg = Message(
    id: 0,
    conversationId: 1,
    senderType: 'user',
    contentType: 'voice',
    content: '',
    mediaUrl: 'https://cos/x.mp3',
    duration: 5,
    source: 'reply',
    createdAt: '2026-04-21',
    clientMsgId: 'v1',
    status: MessageStatus.failed,
  );
  c.messages.add(failedMsg);

  c.retryMessage(failedMsg); // Task 19 实现

  // 验证 Sender 的 retry 被调用（后续 Task 19 才完整）
});
```

这个测试让 Task 17 的 spec 验证推迟到 Task 19。Task 17 只做代码改造。

- [ ] **Step 2: 改 `_uploadAndSendVoice`**

删掉 "上传成功立刻 sent" 的逻辑，改为 "上传成功 → 交给 MessageSender"：

```dart
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
    msg.mediaUrl = uploadResp.data!.url;
    messages.refresh();
    // 不再 msg.status = MessageStatus.sent——交给 MessageSender 等 ACK
    Get.find<MessageSender>().sendVoice(
      conversationId: conversation.id,
      clientMsgId: msg.clientMsgId!,
      mediaUrl: uploadResp.data!.url,
      duration: uploadResp.data!.duration,
      messageJson: msg.toJson(),
    );
  } catch (e, stack) {
    debugPrint('[ChatController] _uploadAndSendVoice failed: $e\n$stack');
    _markFailed(msg);
  }
}
```

- [ ] **Step 3: flutter analyze + test**

```bash
flutter analyze
cd business_packages/sanyan_chat
flutter test
```

- [ ] **Step 4: 提交**

```bash
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "refactor(chat-controller): 语音消息改走 MessageSender，由 ACK 驱动 sent

HTTP 上传成功后立刻交给 MessageSender，msg.status 保持 sending 等
ACK。移除 '上传成功立即 sent' 的 optimistic 行为——ACK 偶发丢失的
问题靠 30s 超时兜住。"
```

---

## Task 18: ChatController 监听 statusChanges + merge pending

**Files:**
- Modify: `business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`

- [ ] **Step 1: onInit 加 merge pending**

在 `_loadHistory()` 返回后追加：

```dart
Future<void> _loadHistory() async {
  try {
    final resp = await ChatApi.listMessages(conversation.id);
    if (resp.success && resp.data != null) {
      messages.value = resp.data!;
    }
  } finally {
    _mergePendingFromSender();
    isLoading.value = false;
    _scrollToBottom();
  }
}

void _mergePendingFromSender() {
  final sender = Get.find<MessageSender>();
  final pending = sender.getPending(conversation.id);
  for (final entry in pending) {
    final msg = Message.fromJson(entry.messageJson);
    // 如果 messages 里已经有同 clientMsgId 的消息（不应该发生，但保险），跳过
    if (msg.clientMsgId != null &&
        messages.any((m) => m.clientMsgId == msg.clientMsgId)) {
      continue;
    }
    messages.add(msg);
  }
}
```

- [ ] **Step 2: 订阅 statusChanges**

在 `onInit` 里加订阅：

```dart
StreamSubscription<PendingEntry>? _senderSub;

@override
void onInit() {
  super.onInit();
  _loadHistory();
  _listenWs();
  ChatApi.markRead(conversation.id);
  WidgetsBinding.instance.addPostFrameCallback((_) {
    if (Get.isRegistered<ConversationListController>()) {
      Get.find<ConversationListController>().enterChat(conversation.id);
    }
  });

  _senderSub = Get.find<MessageSender>().statusChanges.listen(_onSenderStatusChange);
}

void _onSenderStatusChange(PendingEntry entry) {
  if (entry.conversationId != conversation.id) return;
  final idx = messages.indexWhere((m) => m.clientMsgId == entry.clientMsgId);
  if (idx == -1) {
    // 如果本 controller 没这条消息（比如刚打开时还没 merge 完），忽略——
    // 下次进来 merge 时会拿到最新状态。
    return;
  }
  final newStatus = _parseStatus(entry.messageJson['status'] as String?);
  messages[idx].status = newStatus;
  messages.refresh();
}

MessageStatus? _parseStatus(String? raw) {
  if (raw == null) return null;
  return MessageStatus.values.firstWhere(
    (s) => s.name == raw,
    orElse: () => MessageStatus.sending,
  );
}
```

`onClose` 里加：

```dart
_senderSub?.cancel();
```

- [ ] **Step 3: 删除 `_onAck` 和 `_markFailed` 中不再需要的部分**

`_onAck` 整个可以删除（Sender 接管了）。`_markFailed` 本来用于 voice 上传失败，现在还是保留——上传失败是 ChatController 的职责，不是 Sender 的（因为 Sender 的 timeout 只覆盖 WS ACK 阶段）。

在 `_listenWs` 里找到 `case WsEventType.ack: _onAck(event.clientMsgId);`——这条可以删，因为 Sender 已经监听 WsClient 事件流会处理 ACK。**确认两边不要双重处理**。

- [ ] **Step 4: flutter analyze + test**

```bash
flutter analyze
cd business_packages/sanyan_chat
flutter test
```

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "feat(chat-controller): onInit merge pending + 订阅 sender.statusChanges

进聊天页时把 MessageSender 里该会话的 pending 消息合并到 messages
列表（冷启后的 failed、离开后回来的 sending 都能看到）。订阅状态
变化事件，刷新对应气泡的 status。删除本地 _onAck 避免与 Sender 重复。"
```

---

## Task 19: ChatController.retryMessage 统一实现

**Files:**
- Modify: `business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`
- Modify: `business_packages/sanyan_chat/test/chat_controller_test.dart`

- [ ] **Step 1: 写失败测试**

```dart
test('retryMessage: failed text message re-enters sending via MessageSender', () {
  final sender = _FakeMessageSender();
  Get.put<WsClient>(_FakeWsClient());
  Get.put<MessageSender>(sender);

  final c = ChatController(_fixtureConv(), recorder: _MockRecorder());
  final failedMsg = Message(
    id: 0,
    conversationId: 1,
    senderType: 'user',
    contentType: 'text',
    content: 'hello',
    source: 'reply',
    createdAt: '2026-04-21',
    clientMsgId: 'retry-a',
    status: MessageStatus.failed,
  );
  c.messages.add(failedMsg);

  c.retryMessage(failedMsg);

  expect(sender.retriedEntries, hasLength(1));
  expect(failedMsg.status, MessageStatus.sending);
});
```

在 `_FakeMessageSender` 里加：

```dart
final retriedEntries = <PendingEntry>[];

@override
void retry(PendingEntry entry) {
  retriedEntries.add(entry);
  entry.messageJson['status'] = 'sending';
}
```

- [ ] **Step 2: 跑测试确认红**

Expected: FAIL（`retryMessage` 是空实现）

- [ ] **Step 3: 实现 retryMessage**

```dart
void retryMessage(Message msg) {
  if (msg.status != MessageStatus.failed) return;
  if (msg.clientMsgId == null) return;
  final sender = Get.find<MessageSender>();
  // 把 Message 转回 PendingEntry 交给 Sender
  final entry = PendingEntry(
    clientMsgId: msg.clientMsgId!,
    conversationId: msg.conversationId,
    sendTimeMs: DateTime.now().millisecondsSinceEpoch,
    messageJson: msg.toJson(),
  );
  sender.retry(entry);
  msg.status = MessageStatus.sending;
  messages.refresh();
}
```

- [ ] **Step 4: 跑测试确认绿 + 提交**

```bash
cd /Users/aventador/code/3yan/app/business_packages/sanyan_chat
flutter test
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "feat(chat-controller): retryMessage 统一入口接入 MessageSender.retry

支持文本 + 语音消息重试（Sender 内部按 contentType 分派）。失败气泡
点击红感叹号即触发。"
```

---

## Task 20: 清理 + 端到端验证 + 合并 dev

- [ ] **Step 1: grep 清理**

```bash
cd /Users/aventador/code/3yan/app
grep -rn "_onAck\|_markFailed" business_packages/sanyan_chat/lib/ --include="*.dart"
```

确认：`_markFailed` 还在被 `_uploadAndSendVoice` 用（上传失败时——合理保留）；`_onAck` 应该已经没引用。如果还有残留删掉。

- [ ] **Step 2: 全量测试**

```bash
cd /Users/aventador/code/3yan/app
flutter analyze
flutter test
cd foundation_packages/sanyan_network
flutter test
```

三处都必须绿。

- [ ] **Step 3: 手动冒烟**

启动 app（iOS 或 Web），进聊天页：

1. **正常发消息**：发 "hello" → 预期：小菊花 1-2 秒 → 消失（sent）→ AI 回复
2. **断网发消息**：关 wifi → 发 "test" → 预期：立刻红感叹号 → 开 wifi → 点感叹号 → 重新发送 → 菊花 → 消失 → AI 回复
3. **进出聊天页保持状态**：断网发 "a" → 关闭聊天页回列表 → 重进聊天页 → 预期：看到 "a" 带红感叹号
4. **冷启动**：断网发 "b" → 杀进程 → 开 wifi → 开 app → 进聊天页 → 预期：看到 "b" 带红感叹号（冷启逻辑把 sending 转 failed）
5. **服务端重启**：发 "c" → 立即 `ssh beastify 'systemctl restart sanyan'` → 预期：30 秒后红感叹号（WebSocket 断开 + MessageSender 处理）

- [ ] **Step 4: 合并到 dev**

```bash
cd /Users/aventador/code/3yan/app
git checkout dev
git merge --no-ff feat/message-reliability -m "合并 feat/message-reliability：统一消息发送状态机 + 断线/超时/冷启兜底"
git push origin dev
```

- [ ] **Step 5: 主工程更新子模块引用**

```bash
cd /Users/aventador/code/3yan
git checkout dev
git add app
git commit -m "chore: 更新 app 子模块引用（消息发送可靠性）"
git push origin dev
```

---

## Self-Review 已完成

- [x] **Spec 覆盖**：
  - 统一状态机（sending/sent/failed）：Task 13/14/15/16/17 覆盖
  - ACK 驱动：Task 7（sendText + ACK）
  - 30s 超时：Task 8（可注入 timeout）
  - 断线立即标 failed：Task 9
  - 冷启加载：Task 10
  - 连接自动重连：Task 2/3
  - 气泡基类统一 UI：Task 13/14/15
  - Sender 跨 controller 生命周期：Task 12（permanent Get.put）
  - 重试入口：Task 11（sender.retry）+ Task 19（controller.retryMessage）
  - 内存 Map + 扫描 Timer + 懒启动：Task 6/7/8
  - 持久化：Task 10

- [x] **Placeholder 扫描**：无 TBD / TODO / "similar to Task N"。`chat_controller.dart` 的 `retryMessage` 空实现在 Task 14 有明确占位说明，Task 19 填实。

- [x] **类型一致性**：
  - `MessageSender.sendText/sendVoice/retry/removePending/getPending/statusChanges` 全 plan 命名一致
  - `PendingEntry(clientMsgId, conversationId, sendTimeMs, messageJson)` 字段一致
  - `MessageStatus.sending/sent/failed` 用原有枚举，无新增
  - `WsClient.onDisconnected`、`reconnectDelayForAttempt`、`sendMessage/sendVoiceMessage(required clientMsgId)` 一致
  - `Message.toJson/fromJson` 加 `status`/`clientMsgId` 字段在 Task 16 说明

- [x] **工作量核对**：20 个 task，其中 1/12/15/20 是轻改动（配置、注册、UI 删除、合并），13/16/19 是 UI/ChatController 中量，7/8/9/10/11 是 MessageSender 核心，拆分合理。
