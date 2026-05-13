# P3 sanyan_util 基础包补全 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建 `foundation_packages/sanyan_util` 包，下沉 `message_bubble` 内 inline 的 ISO 8601 → HH:mm 格式化逻辑为 `DateUtil.formatHHmm`。

**Architecture:** 按 `~/.claude/rules/flutter-foundation-layer.md` §8 创建 `sanyan_util` 包；DateUtil 用 `abstract class` + static 方法（CLAUDE.md static 优先原则）；business 包通过 path 依赖引入。

**Tech Stack:** Flutter SDK 3.11.4 / Dart `>=3.0.0` / flutter_test。

**Spec:** `docs/superpowers/specs/2026-05-14-p3-foundation-util-design.md`

---

## 文件结构

### 新建文件

```
app/foundation_packages/sanyan_util/
├── pubspec.yaml                                    ← 包声明
├── lib/
│   ├── sanyan_util.dart                            ← barrel: export 'src/date_util.dart'
│   └── src/
│       └── date_util.dart                          ← DateUtil 类
└── test/
    ├── sanyan_util_suite.dart                      ← 测试聚合（rule §10 强制）
    └── date_util_test.dart                         ← 6 个 test
```

### 修改文件

- `app/business_packages/sanyan_chat/pubspec.yaml` — `dependencies:` 段加 `sanyan_util` path 依赖
- `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart` — `_Timestamp.build()` 用 `DateUtil.formatHHmm` 替换 inline 逻辑

### 工作目录约定

所有命令默认在 `/Users/aventador/code/3yan/app/` 下执行。

---

## Task 1: 建 sanyan_util 包骨架

**Files:**
- Create: `foundation_packages/sanyan_util/pubspec.yaml`
- Create: `foundation_packages/sanyan_util/lib/sanyan_util.dart`
- Create: `foundation_packages/sanyan_util/test/sanyan_util_suite.dart`
- Create: 目录 `foundation_packages/sanyan_util/lib/src/`

- [ ] **Step 1: 创建目录结构**

```bash
cd /Users/aventador/code/3yan/app
mkdir -p foundation_packages/sanyan_util/lib/src
mkdir -p foundation_packages/sanyan_util/test
```

- [ ] **Step 2: 创建 `pubspec.yaml`**

文件 `foundation_packages/sanyan_util/pubspec.yaml`:

```yaml
name: sanyan_util
description: 三言 - 通用工具类
publish_to: 'none'

environment:
  sdk: ^3.11.4

dependencies:
  flutter:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0
```

- [ ] **Step 3: 创建 barrel `lib/sanyan_util.dart`**

```dart
export 'src/date_util.dart';
```

注：此时 `src/date_util.dart` 还不存在，barrel 暂时是"指向未来的引用"。Task 2 创建实际文件。

- [ ] **Step 4: 创建测试聚合 `test/sanyan_util_suite.dart`**

```dart
import 'date_util_test.dart' as date_util;

void main() {
  date_util.main();
}
```

同样，`date_util_test.dart` Task 2 创建。

- [ ] **Step 5: 跑 `flutter pub get` 验证 pubspec 解析正常**

```bash
cd foundation_packages/sanyan_util
flutter pub get
```

Expected: `Got dependencies!` 输出，无 error。如果有 error 检查 pubspec yaml 格式。

- [ ] **Step 6: 此 task 不 commit（pub get 产物 + 不完整 barrel）**

Task 2 一起 commit，避免半成品 commit。

---

## Task 2: TDD DateUtil.formatHHmm

**Files:**
- Create: `foundation_packages/sanyan_util/test/date_util_test.dart`
- Create: `foundation_packages/sanyan_util/lib/src/date_util.dart`

- [ ] **Step 1: 写 6 个 failing test**

文件 `foundation_packages/sanyan_util/test/date_util_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_util/sanyan_util.dart';

void main() {
  group('DateUtil.formatHHmm', () {
    test('null returns empty string', () {
      expect(DateUtil.formatHHmm(null), '');
    });

    test('empty string returns empty string', () {
      expect(DateUtil.formatHHmm(''), '');
    });

    test('invalid string returns empty string', () {
      expect(DateUtil.formatHHmm('not-a-date'), '');
    });

    test('valid local ISO 8601 (no timezone) returns HH:mm verbatim', () {
      // 无时区后缀的 ISO 8601，Dart parse 默认按本地时区，toLocal() 不变换 → 跨时区稳定
      expect(DateUtil.formatHHmm('2026-05-14T14:05:30'), '14:05');
    });

    test('single-digit hour and minute are zero-padded', () {
      expect(DateUtil.formatHHmm('2026-05-14T03:07:00'), '03:07');
    });

    test('UTC ISO 8601 (Z suffix) is converted to local timezone', () {
      // 用相对断言避免硬编码本地时区：parse + toLocal 后必然是 HH:mm 格式
      final result = DateUtil.formatHHmm('2026-05-14T14:05:30Z');
      expect(result, matches(RegExp(r'^\d{2}:\d{2}$')));
    });
  });
}
```

- [ ] **Step 2: 跑测试看 RED**

```bash
cd /Users/aventador/code/3yan/app/foundation_packages/sanyan_util
flutter test
```

Expected: 编译失败。错误信息含 `Target of URI doesn't exist: 'src/date_util.dart'` 或 `Undefined name 'DateUtil'`。这是预期 RED。

- [ ] **Step 3: 创建 `lib/src/date_util.dart` 写最小实现**

```dart
/// 日期/时间格式化工具。
///
/// 按 ~/.claude/CLAUDE.md "static 优先原则"：abstract class + 全 static 方法。
abstract class DateUtil {
  /// 把 ISO 8601 字符串格式化为本地时区 HH:mm。
  ///
  /// - null / 空串 / 解析失败 → 返回 ""
  /// - 合法 ISO 8601 → 转本地时区后取 hour:minute，左补 0 至 2 位
  static String formatHHmm(String? iso8601) {
    if (iso8601 == null || iso8601.isEmpty) return '';
    try {
      final dt = DateTime.parse(iso8601).toLocal();
      final hh = dt.hour.toString().padLeft(2, '0');
      final mm = dt.minute.toString().padLeft(2, '0');
      return '$hh:$mm';
    } catch (_) {
      return '';
    }
  }
}
```

- [ ] **Step 4: 跑测试看 GREEN**

```bash
flutter test
```

Expected:
```
00:00 +6: All tests passed!
```

如有失败，根据 error 调实现。

- [ ] **Step 5: 跑测试聚合入口确认它工作**

```bash
flutter test test/sanyan_util_suite.dart
```

Expected: 同上 6 tests passed（suite 等同于直接跑 date_util_test）。

- [ ] **Step 6: commit Task 1 + 2 合并**

```bash
cd /Users/aventador/code/3yan/app
git add foundation_packages/sanyan_util/
git commit -m "feat(util): 新建 sanyan_util 包 + DateUtil.formatHHmm（TDD 6 tests pass）"
```

---

## Task 3: message_bubble 切换使用 DateUtil + 删 inline 逻辑

**Files:**
- Modify: `app/business_packages/sanyan_chat/pubspec.yaml`
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart`

- [ ] **Step 1: 给 sanyan_chat 加 sanyan_util 依赖**

文件 `business_packages/sanyan_chat/pubspec.yaml` 的 `dependencies:` 段（在 `sanyan_common_ui` 那条之后）增加：

```yaml
  sanyan_util:
    path: ../../foundation_packages/sanyan_util
```

完整 `dependencies:` 段改后样子：

```yaml
dependencies:
  flutter:
    sdk: flutter
  get: ^4.6.6
  uuid: ^4.3.3
  sanyan_network:
    path: ../../foundation_packages/sanyan_network
  sanyan_user:
    path: ../../foundation_packages/sanyan_user
  sanyan_common_ui:
    path: ../../foundation_packages/sanyan_common_ui
  sanyan_util:
    path: ../../foundation_packages/sanyan_util
```

- [ ] **Step 2: 跑 `flutter pub get` 验证依赖解析**

```bash
cd /Users/aventador/code/3yan/app/business_packages/sanyan_chat
flutter pub get
```

Expected: `Got dependencies!` 无 error。

- [ ] **Step 3: 修改 message_bubble.dart 替换 inline 逻辑**

文件 `business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart`:

在文件顶部 import 段加：

```dart
import 'package:sanyan_util/sanyan_util.dart';
```

然后找到 `class _Timestamp` 的 `build` 方法（当前约 line 134-142），改前是：

```dart
class _Timestamp extends StatelessWidget {
  final String createdAt;
  const _Timestamp({required this.createdAt});
  @override
  Widget build(BuildContext context) {
    String t = '';
    try {
      final dt = DateTime.parse(createdAt).toLocal();
      t = '${dt.hour.toString().padLeft(2, '0')}:${dt.minute.toString().padLeft(2, '0')}';
    } catch (_) {}
    return Text(t, style: const TextStyle(fontSize: 11, color: Colors.grey));
  }
}
```

改后：

```dart
class _Timestamp extends StatelessWidget {
  final String createdAt;
  const _Timestamp({required this.createdAt});
  @override
  Widget build(BuildContext context) {
    return Text(
      DateUtil.formatHHmm(createdAt),
      style: const TextStyle(fontSize: 11, color: Colors.grey),
    );
  }
}
```

变化：
- 删除 `String t = ''; try { ... } catch (_) {}` 5 行 inline 逻辑
- `Text(t, ...)` 改成 `Text(DateUtil.formatHHmm(createdAt), ...)`

- [ ] **Step 4: 跑 sanyan_chat 现有测试看是否仍通过**

```bash
cd /Users/aventador/code/3yan/app/business_packages/sanyan_chat
flutter test 2>&1 | tail -10
```

Expected: 如果 sanyan_chat 包下 `test/` 目录有测试文件就跑；如果是空（之前 plan-1 final review 提到 `business_packages/sanyan_chat/test/` 为空），命令输出 `No tests were found.`，这是 OK 的（P5 才补 sanyan_chat 测试）。

不应有编译 error。

- [ ] **Step 5: 跑 app 主工程编译确认无破坏**

```bash
cd /Users/aventador/code/3yan/app
flutter analyze 2>&1 | tail -10
```

Expected: `No issues found!` 或仅有 pre-existing warning。**不应**新增 error。

- [ ] **Step 6: commit**

```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_chat/pubspec.yaml \
        business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart
git commit -m "refactor(chat): message_bubble 时分格式化下沉到 DateUtil（删 inline 逻辑）"
```

---

## Task 4: 手测 app + push + 合并 master

**Files:** （无代码改动，仅集成验证 + git 操作）

- [ ] **Step 1: 启动 app dev 模式（如果需要全量验证）**

```bash
cd /Users/aventador/code/3yan/app
flutter run -d <你的模拟器或真机 device-id>
```

可选：如果只想编译验证不实际跑 UI，跳过此 step。

- [ ] **Step 2: 手测气泡时间显示**

在 app 上：
1. 进入聊天页（已登录 + 已有历史消息）
2. 观察每条消息气泡下方的时分时间显示
3. 确认显示**跟改动前一致**（格式 `HH:mm`，本地时区正确）

如果显示不正常（如全是空串、或时区不对），回退到 Task 3 检查 import / 类名。

- [ ] **Step 3: push 当前 dev 分支**

```bash
cd /Users/aventador/code/3yan/app
git push
```

- [ ] **Step 4: 合并 app 子模块 dev → master**

```bash
git checkout master
git pull --ff-only
git merge --no-ff dev -m "merge: dev → master（P3 sanyan_util 包补全 + DateUtil 下沉）"
git push
git checkout dev
```

- [ ] **Step 5: 主仓库更新 app 子模块引用 + 合并 master**

```bash
cd /Users/aventador/code/3yan
git checkout dev
git add app
git commit -m "chore: 更新 app 子模块引用（P3 sanyan_util 包补全）"
git push

git checkout master
git pull --ff-only
git merge --no-ff dev -m "merge: dev → master（P3 sanyan_util 包补全）"
git push
git checkout dev
```

---

## 验收总清单

执行完所有 task 后逐项 check（对应 spec §9）：

- [ ] `app/foundation_packages/sanyan_util/` 目录存在 + pubspec.yaml + barrel + DateUtil + 6 个 test + suite
- [ ] `flutter test` 在 sanyan_util 下 6 个 test 全过
- [ ] `app/business_packages/sanyan_chat/pubspec.yaml` 含 sanyan_util 依赖（path: `../../foundation_packages/sanyan_util`）
- [ ] `message_bubble.dart._Timestamp.build()` 不再有 `DateTime.parse` + `padLeft` inline 逻辑，改用 `DateUtil.formatHHmm(createdAt)`
- [ ] `flutter analyze` 在 app 主工程下无新增 error
- [ ] app 实际跑起来后气泡时间显示正常（手测）
- [ ] app 子模块 + 主仓库都 push 完 + 合并到各自的 master

---

## Self-Review 备注

**1. Spec 覆盖**：
- spec §2 包结构 → Task 1（pubspec/barrel/suite/目录）
- spec §3 DateUtil API → Task 2
- spec §4 使用方改动 → Task 3
- spec §5 测试设计 → Task 2 Step 1
- spec §6 实施路径 3 个 task → Plan 也是 3 个 task（外加 Task 4 验收 + 合并，是 plan 标准收尾不算新内容）
- spec §9 验收 → "验收总清单" 段

**2. Placeholder 扫描**：所有代码 block 都是完整内容；命令都给了 expected 输出；无 "TBD" / "..." / "implement later"。✅

**3. Type 一致性**：
- `DateUtil` 类名 + `formatHHmm(String?)` 签名贯穿 Task 2 实现、Task 2 测试、Task 3 使用方调用——全部一致。
- import path `package:sanyan_util/sanyan_util.dart` 在 Task 2 测试和 Task 3 使用方都是同一个写法。

---

## Execution Handoff

**Plan complete and saved to `docs/superpowers/plans/2026-05-14-p3-foundation-util.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — 每个 task 派一个 fresh subagent + 两段审查（spec + 代码质量），主会话保持干净。

**2. Inline Execution** — 当前会话里逐 task 执行，checkpoint 时停下给你 review。

P3 工作量小（3-4 个 task / 30-60 分钟），两种执行方式都 OK。subagent-driven 上下文更干净，inline 更直接。

**哪种？**
