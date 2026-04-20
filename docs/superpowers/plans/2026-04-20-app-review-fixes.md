# App Code Review 修复 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 app 端 2026-04-20 code review 报告中的违反全局规范问题——把 `ChatPage` 从 StatefulWidget + setState 重构为 StatelessWidget + GetX，所有业务/录音状态迁到 `ChatController`；顺带清掉魔法数字和吞异常。

**Architecture:** 在 app submodule 的新分支 `fix/review-findings` 上工作。核心是把 `VoiceRecorder`、`inputMode`、`isRecording`、`isCancelling`、`recordStartPosition` 从 `_ChatPageState` 全部迁到 `ChatController`（作为 `Rx`/字段）；录音启动/取消/停止的业务逻辑也从 Widget 事件处理函数迁到 Controller 方法，Widget 只负责透传。

**Tech Stack:** Flutter 3.11 · GetX 4.6 · Dart · flutter_test（单元测试）· mocktail（mock VoiceRecorder）

---

## File Structure

**修改（app/business_packages/sanyan_chat/lib/src/chat/）：**
- `chat_page.dart` — 全文重写为 StatelessWidget，构造函数接收 `Conversation`
- `chat_controller.dart` — 接管 VoiceRecorder + 3 个 Rx 状态 + 录音处理方法 + `toggleInputMode`；`_uploadAndSendVoice` 的 `catch` 加日志
- `voice_recorder.dart` — 抽接口 `IVoiceRecorder`（便于 mock），`VoiceRecorder` 实现它；或在 ChatController 里允许通过构造注入，不动原类
- `widget/chat_input_mode.dart` — 无改动
- `widget/chat_input_bar.dart` — 无改动（接口已经是回调透传，符合规范）

**修改（app/）：**
- `lib/main.dart` — GetPage 路由工厂改为 `() => ChatPage(conversation: Get.arguments as Conversation)`

**修改（app/business_packages/sanyan_chat/）：**
- `pubspec.yaml` — 加 mocktail dev dep
- `test/chat_controller_test.dart` — 补充 toggleInputMode / 录音状态流转的单元测试

---

## Task 1: 建 app 分支

- [ ] **Step 1:**

```bash
cd /Users/aventador/code/3yan/app
git checkout -b fix/review-findings
git status
```

Expected: 在新分支、干净。

---

## Task 2: 给 VoiceRecorder 抽接口（便于 mock 测试）

**目的：** 让 ChatController 能注入 mock 录音器，新增的录音业务逻辑可在无音频硬件环境下测试。

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/voice_recorder.dart`
- Test: `app/business_packages/sanyan_chat/test/voice_recorder_interface_test.dart`

- [ ] **Step 1: 写失败测试**

Create `app/business_packages/sanyan_chat/test/voice_recorder_interface_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_chat/sanyan_chat.dart';

void main() {
  test('VoiceRecorder implements IVoiceRecorder', () {
    final r = VoiceRecorder();
    expect(r, isA<IVoiceRecorder>());
  });
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
cd /Users/aventador/code/3yan/app/business_packages/sanyan_chat
flutter test test/voice_recorder_interface_test.dart
```

Expected: FAIL（`IVoiceRecorder` 未定义）

- [ ] **Step 3: 抽接口**

在 `voice_recorder.dart` 顶部 `class RecordingResult` 之后新增：

```dart
abstract class IVoiceRecorder {
  Future<bool> isPermissionGranted();
  Future<bool> requestPermission();
  Future<bool> start({void Function()? onMaxDurationReached});
  Future<RecordingResult?> stop();
  Future<void> cancel();
  int get currentDurationSeconds;
  void dispose();
}
```

`VoiceRecorder class` 改为：

```dart
class VoiceRecorder implements IVoiceRecorder {
  // ... 其余不变 ...
}
```

并通过 `lib/sanyan_chat.dart` 导出 `IVoiceRecorder`（确认该 barrel file 已有 `export 'src/chat/voice_recorder.dart';`，否则补上）。

- [ ] **Step 4: 跑测试确认绿 + 提交**

```bash
flutter test test/voice_recorder_interface_test.dart
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "refactor(chat): 抽 IVoiceRecorder 接口，便于测试注入 mock"
```

---

## Task 3: ChatController 接管 inputMode + `toggleInputMode`

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`
- Modify: `app/business_packages/sanyan_chat/pubspec.yaml`（加 mocktail dev）
- Test: `app/business_packages/sanyan_chat/test/chat_controller_test.dart`

- [ ] **Step 1: 加 mocktail 依赖**

`app/business_packages/sanyan_chat/pubspec.yaml` 的 `dev_dependencies:` 追加：

```yaml
  mocktail: ^1.0.4
```

```bash
cd /Users/aventador/code/3yan/app/business_packages/sanyan_chat
flutter pub get
```

- [ ] **Step 2: 写失败测试（toggleInputMode）**

在 `test/chat_controller_test.dart` 文件末尾追加：

```dart
import 'package:get/get.dart';
import 'package:mocktail/mocktail.dart';
import 'package:sanyan_network/sanyan_network.dart';

class _MockWsClient extends Mock implements WsClient {}
class _MockRecorder extends Mock implements IVoiceRecorder {}

void _resetGet() {
  Get.reset();
}

Conversation _fixtureConv() => Conversation(
      id: 1,
      characterId: 100,
      characterName: 'A',
      unreadCount: 0,
    );

void main() {
  // ... 原有测试保留 ...

  group('ChatController.inputMode', () {
    tearDown(_resetGet);

    test('initial inputMode reads from storage', () {
      Get.put<WsClient>(_MockWsClient());
      final c = ChatController(
        _fixtureConv(),
        recorder: _MockRecorder(),
      );
      expect(c.inputMode.value, isA<ChatInputMode>());
    });

    test('toggleInputMode flips between keyboard and voice', () {
      Get.put<WsClient>(_MockWsClient());
      final c = ChatController(
        _fixtureConv(),
        recorder: _MockRecorder(),
      );
      final before = c.inputMode.value;
      c.toggleInputMode();
      expect(c.inputMode.value, isNot(before));
      c.toggleInputMode();
      expect(c.inputMode.value, before);
    });
  });
}
```

- [ ] **Step 3: 跑测试确认红**

```bash
flutter test test/chat_controller_test.dart
```

Expected: FAIL（ChatController 构造不接受 `recorder` 参数；`inputMode` / `toggleInputMode` 未定义）

- [ ] **Step 4: 实现**

`chat_controller.dart` 修改：

1. 顶部 import 加：
   ```dart
   import 'voice_recorder.dart';
   import 'widget/chat_input_mode.dart';
   import 'package:sanyan_user/sanyan_user.dart';
   ```

2. 类声明字段区新增：
   ```dart
   final IVoiceRecorder recorder;
   late final Rx<ChatInputMode> inputMode;
   ```

3. 构造函数：
   ```dart
   ChatController(this.conversation, {IVoiceRecorder? recorder})
       : recorder = recorder ?? VoiceRecorder() {
     inputMode = ChatInputMode.fromStorage(LocalStorage.lastInputMode).obs;
   }
   ```

4. 新增方法：
   ```dart
   void toggleInputMode() {
     inputMode.value = inputMode.value == ChatInputMode.keyboard
         ? ChatInputMode.voice
         : ChatInputMode.keyboard;
     LocalStorage.lastInputMode = inputMode.value.storageValue;
   }
   ```

5. `onClose` 里加：
   ```dart
   recorder.dispose();
   ```

- [ ] **Step 5: 跑测试确认绿**

```bash
flutter test test/chat_controller_test.dart
```

- [ ] **Step 6: 提交**

```bash
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "refactor(chat): inputMode 状态和切换逻辑迁到 ChatController"
```

---

## Task 4: ChatController 接管录音状态 + 业务方法

**目的：** 把 `_ChatPageState` 里的 `_isRecording`、`_isCancelling`、`_recordStartPosition`、`_onRecordStart/Move/End/Cancel` 全部迁到 `ChatController`。

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`
- Test: `app/business_packages/sanyan_chat/test/chat_controller_test.dart`

- [ ] **Step 1: 写失败测试（录音状态流转）**

在 `test/chat_controller_test.dart` 追加：

```dart
import 'package:flutter/widgets.dart';

void main() {
  // ... 原有测试保留 ...

  group('ChatController.recording', () {
    late _MockWsClient ws;
    late _MockRecorder recorder;
    late ChatController c;

    setUp(() {
      ws = _MockWsClient();
      recorder = _MockRecorder();
      Get.put<WsClient>(ws);
      c = ChatController(_fixtureConv(), recorder: recorder);
    });

    tearDown(_resetGet);

    test('onRecordStart: no permission -> requests and does NOT start', () async {
      when(() => recorder.isPermissionGranted()).thenAnswer((_) async => false);
      when(() => recorder.requestPermission()).thenAnswer((_) async => true);

      await c.onRecordStart(const Offset(0, 0));

      expect(c.isRecording.value, false);
      verifyNever(() => recorder.start(onMaxDurationReached: any(named: 'onMaxDurationReached')));
    });

    test('onRecordStart: permission granted -> starts and sets isRecording', () async {
      when(() => recorder.isPermissionGranted()).thenAnswer((_) async => true);
      when(() => recorder.start(
            onMaxDurationReached: any(named: 'onMaxDurationReached'),
          )).thenAnswer((_) async => true);

      await c.onRecordStart(const Offset(0, 100));

      expect(c.isRecording.value, true);
      expect(c.isCancelling.value, false);
    });

    test('onRecordMove: dy > threshold flips isCancelling to true', () async {
      when(() => recorder.isPermissionGranted()).thenAnswer((_) async => true);
      when(() => recorder.start(
            onMaxDurationReached: any(named: 'onMaxDurationReached'),
          )).thenAnswer((_) async => true);
      await c.onRecordStart(const Offset(100, 200));

      c.onRecordMove(const Offset(100, 100)); // dy = 200 - 100 = 100 > 80

      expect(c.isCancelling.value, true);
    });

    test('onRecordEnd: cancelling=true -> cancels recording, does not send', () async {
      when(() => recorder.isPermissionGranted()).thenAnswer((_) async => true);
      when(() => recorder.start(
            onMaxDurationReached: any(named: 'onMaxDurationReached'),
          )).thenAnswer((_) async => true);
      when(() => recorder.cancel()).thenAnswer((_) async {});
      await c.onRecordStart(const Offset(100, 300));
      c.onRecordMove(const Offset(100, 200)); // isCancelling -> true

      await c.onRecordEnd();

      expect(c.isRecording.value, false);
      verify(() => recorder.cancel()).called(1);
      verifyNever(() => recorder.stop());
    });

    test('onRecordEnd: cancelling=false -> stops and returns RecordingResult', () async {
      when(() => recorder.isPermissionGranted()).thenAnswer((_) async => true);
      when(() => recorder.start(
            onMaxDurationReached: any(named: 'onMaxDurationReached'),
          )).thenAnswer((_) async => true);
      when(() => recorder.stop())
          .thenAnswer((_) async => RecordingResult('/tmp/a.m4a', 3));
      await c.onRecordStart(const Offset(0, 0));

      await c.onRecordEnd();

      expect(c.isRecording.value, false);
      verify(() => recorder.stop()).called(1);
    });
  });
}
```

- [ ] **Step 2: 跑测试确认红**

```bash
flutter test test/chat_controller_test.dart
```

Expected: FAIL（`isRecording`/`isCancelling`/`onRecordStart`/`onRecordMove`/`onRecordEnd` 未定义）

- [ ] **Step 3: 实现 ChatController 录音能力**

`chat_controller.dart` 新增字段：

```dart
final isRecording = false.obs;
final isCancelling = false.obs;
Offset? _recordStartPosition;

/// 滑动取消阈值：手指上滑超过此像素数视为取消录音。
static const double _cancelSwipeThreshold = 80.0;

/// 回调：需要弹提示给用户（UI 层通过 registerToastHandler 注入）。
void Function(String message)? _toastHandler;
void registerToastHandler(void Function(String) handler) {
  _toastHandler = handler;
}
void _showToast(String msg) => _toastHandler?.call(msg);
```

新增方法：

```dart
/// 进聊天页后台跑一次 start+cancel，触发 iOS AAC codec 首次初始化。
/// 之后 codec 被缓存，真正按住说话的启动时间能从 ~1s 降到百毫秒。
/// 只在已有权限时预热，避免静默触发系统权限弹窗。
Future<void> warmupRecorder() async {
  if (!await recorder.isPermissionGranted()) return;
  final started = await recorder.start();
  if (started) {
    await recorder.cancel();
  }
}

Future<void> onRecordStart(Offset globalPosition) async {
  // 权限已授予 → 直接开始录音。
  // 权限未授予 → 弹系统弹窗询问，弹窗会打断长按手势，此时不应该继续开始录音
  //             （否则用户手指早已离开按钮，录音会卡死），而是提示用户再次按住。
  if (!await recorder.isPermissionGranted()) {
    final granted = await recorder.requestPermission();
    _showToast(granted ? '麦克风权限已获取，请再次按住说话' : '需要麦克风权限才能发送语音');
    return;
  }
  final started = await recorder.start(onMaxDurationReached: () {
    if (isRecording.value) onRecordEnd();
  });
  if (!started) {
    _showToast('录音启动失败');
    return;
  }
  isRecording.value = true;
  isCancelling.value = false;
  _recordStartPosition = globalPosition;
}

void onRecordMove(Offset globalPosition) {
  if (!isRecording.value || _recordStartPosition == null) return;
  final dy = _recordStartPosition!.dy - globalPosition.dy;
  final cancelling = dy > _cancelSwipeThreshold;
  if (cancelling != isCancelling.value) {
    isCancelling.value = cancelling;
  }
}

Future<void> onRecordEnd() async {
  if (!isRecording.value) return;
  final cancelling = isCancelling.value;
  isRecording.value = false;
  isCancelling.value = false;
  _recordStartPosition = null;

  if (cancelling) {
    await recorder.cancel();
    return;
  }
  final result = await recorder.stop();
  if (result == null) {
    _showToast('说话时间太短');
    return;
  }
  sendVoiceMessage(result.filePath, result.durationSeconds);
}

Future<void> onRecordCancel() async {
  if (!isRecording.value) return;
  isRecording.value = false;
  isCancelling.value = false;
  _recordStartPosition = null;
  await recorder.cancel();
}
```

- [ ] **Step 4: 跑测试确认绿**

```bash
flutter test test/chat_controller_test.dart
```

Expected: 全部 PASS

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/code/3yan/app
git add -A
git commit -m "refactor(chat): 录音状态和业务方法迁到 ChatController

isRecording/isCancelling 改为 Rx；onRecordStart/Move/End/Cancel、
warmupRecorder 都从 Widget 搬到 Controller；滑动取消阈值抽成
_cancelSwipeThreshold=80.0 常量。"
```

---

## Task 5: ChatPage 重写为 StatelessWidget

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_page.dart`
- Modify: `app/lib/main.dart`

- [ ] **Step 1: 重写 chat_page.dart**

全文替换为：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';
import '../models/conversation.dart';
import 'chat_controller.dart';
import 'widget/chat_input_bar.dart';
import 'widget/chat_input_mode.dart';
import 'widget/message_bubble.dart';
import 'widget/typing_indicator.dart';
import 'widget/voice_record_overlay.dart';

class ChatPage extends StatelessWidget {
  final Conversation conversation;
  const ChatPage({super.key, required this.conversation});

  @override
  Widget build(BuildContext context) {
    final c = Get.put(ChatController(conversation));
    c.registerToastHandler((msg) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(msg), duration: const Duration(seconds: 2)),
      );
    });
    WidgetsBinding.instance.addPostFrameCallback((_) => c.warmupRecorder());

    return Scaffold(
      backgroundColor: AuraColors.surface,
      appBar: AppBar(
        title: Text(
          conversation.characterName ?? '',
          style: const TextStyle(
            fontSize: 18,
            fontWeight: FontWeight.bold,
            color: AuraColors.primary,
          ),
        ),
        flexibleSpace: ClipRect(
          child: BackdropFilter(
            filter: ImageFilter.blur(
              sigmaX: AuraColors.glassBlur,
              sigmaY: AuraColors.glassBlur,
            ),
            child: Container(color: AuraColors.surface.withValues(alpha: 0.6)),
          ),
        ),
        actions: [
          IconButton(
            onPressed: () {},
            icon: const Icon(Icons.search, color: AuraColors.primary),
          ),
          IconButton(
            onPressed: () {},
            icon: const Icon(Icons.more_vert, color: AuraColors.primary),
          ),
        ],
        bottom: const PreferredSize(
          preferredSize: Size.fromHeight(1),
          child: Divider(height: 1, thickness: 1, color: Color(0xFFD4FBFB)),
        ),
      ),
      body: Stack(
        children: [
          SafeArea(
            top: false,
            child: Column(
              children: [
                Expanded(
                  child: Obx(() {
                    if (c.isLoading.value && c.messages.isEmpty) {
                      return const Center(
                        child: CircularProgressIndicator(color: AuraColors.primary),
                      );
                    }
                    final showTyping = c.isAiTyping.value;
                    final itemCount = c.messages.length + (showTyping ? 1 : 0);
                    return ListView.builder(
                      controller: c.scrollController,
                      padding: const EdgeInsets.symmetric(vertical: 16),
                      itemCount: itemCount,
                      itemBuilder: (context, index) {
                        if (showTyping && index == c.messages.length) {
                          return const TypingIndicator();
                        }
                        return MessageBubble(message: c.messages[index]);
                      },
                    );
                  }),
                ),
                Obx(() => ChatInputBar(
                      controller: c.inputController,
                      mode: c.inputMode.value,
                      onToggleMode: c.toggleInputMode,
                      onSendText: c.sendMessage,
                      isRecording: c.isRecording.value,
                      onRecordStart: c.onRecordStart,
                      onRecordMove: c.onRecordMove,
                      onRecordEnd: c.onRecordEnd,
                      onRecordCancel: c.onRecordCancel,
                    )),
              ],
            ),
          ),
          Obx(() => c.isRecording.value
              ? VoiceRecordOverlay(isCancelling: c.isCancelling.value)
              : const SizedBox.shrink()),
        ],
      ),
    );
  }
}
```

- [ ] **Step 2: 路由工厂改用构造传参（唯一允许从 Get.arguments 取值的位置）**

`app/lib/main.dart:44` 改为：

```dart
GetPage(
  name: AppRoutes.chat,
  page: () => ChatPage(conversation: Get.arguments as Conversation),
),
```

- [ ] **Step 3: flutter analyze + 跑全量测试**

```bash
cd /Users/aventador/code/3yan/app
flutter analyze
flutter test
```

Expected: `No issues found!` + 全部 test pass

- [ ] **Step 4: 手动 smoke 测试（按全局规则要求 UI 改动必须跑一次）**

```bash
flutter run  # 启动 iOS 模拟器
```

人工验证：登录 → 进聊天页 → 切键盘/语音输入 → 长按录音 → 上滑取消 → 松开发送。录音业务行为应与重构前完全一致。

- [ ] **Step 5: 清理 dead code**

确认 `ChatPage` 文件里已经没有 `_ChatPageState`、`_toggleMode`、`_onRecordStart` 等旧代码（Write 操作已覆盖整个文件）。Grep 确认没有遗留：

```bash
cd /Users/aventador/code/3yan/app
grep -rn "_ChatPageState\|_isRecording\|_isCancelling" business_packages/sanyan_chat/lib/
```

Expected: 无输出。

- [ ] **Step 6: 提交**

```bash
git add -A
git commit -m "refactor(chat): ChatPage 改 StatelessWidget，符合全局规范

- 消除 setState，状态全走 Rx + Obx
- 构造函数接收 conversation（不再在 initState 读 Get.arguments）
- Widget 事件全部回调透传给 ChatController
- 路由工厂统一从 Get.arguments 取 conversation 注入构造函数
  （全局规范允许的唯一例外位置）"
```

---

## Task 6: `_uploadAndSendVoice` 吞异常加日志

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_controller.dart`

这条是低风险的小改（原 158-160 行），一步到位。

- [ ] **Step 1: 改 catch 分支**

```dart
} catch (e, stack) {
  // ignore: avoid_print
  print('[ChatController] _uploadAndSendVoice failed: $e\n$stack');
  _markFailed(msg);
}
```

（如果项目有统一 Logger 工具，替换 print 为对应调用。先 grep 确认：）

```bash
grep -rn "class .*Logger\|debugPrint" /Users/aventador/code/3yan/app/foundation_packages/
```

如有 Logger，改为 `Logger.e(...)`；否则暂用 print（后续统一）。

- [ ] **Step 2: 跑测试 + 提交**

```bash
cd /Users/aventador/code/3yan/app
flutter analyze
flutter test
git add -A
git commit -m "fix(chat): 语音上传异常打日志而不是静默失败"
```

---

## Task 7: 合并到 dev + 更新主工程子模块引用

- [ ] **Step 1: app 合并**

```bash
cd /Users/aventador/code/3yan/app
git checkout dev
git merge --no-ff fix/review-findings -m "merge: app code review 修复"
git push origin dev
```

- [ ] **Step 2: 主工程更新 app 子模块引用（如果 server plan 的 Task 10 还没合主工程，这里一起合）**

```bash
cd /Users/aventador/code/3yan
git checkout dev
git merge --no-ff fix/review-findings -m "merge: code review 修复合入主工程" || true
git add app server
git commit -m "chore: 更新 app/server 子模块引用（code review 修复）"
git push origin dev
```

---

## Self-Review 已完成

- [x] Spec 覆盖：ChatPage StatefulWidget（Task 5）、录音业务在 Widget（Task 4）、Get.arguments 在 initState（Task 5）、魔法数字 80（Task 4 `_cancelSwipeThreshold`）、`_uploadAndSendVoice` 吞异常（Task 6）
- [x] 接口名一致：`onRecordStart/Move/End/Cancel`、`toggleInputMode`、`warmupRecorder`、`registerToastHandler` 跨 test / controller / widget 引用一致
- [x] `isRecording` / `isCancelling` / `inputMode` 全部 Rx，build 里用 Obx 包
- [x] 不包含：魔法数字 80 的 `showToast`/`Snackbar` 的替换（规范未要求）、HomePage 的 Rx 在 build 内重建问题（可后续）、message_bubble 的 Icon 替换为图片资源（需要设计图，留后续）
