# P3：前端 sanyan_util 基础包补全设计 spec

> Plan-1 收尾 sub-plan 系列的第 2 个（P2 已砍掉合并到 plan-4）。

## 1. 背景与目标

### 1.1 现状

按 `~/.claude/rules/flutter.md` §1，foundation_packages 强制必有 6 个基础包。当前 app 的 foundation_packages 只有 4 个：
- `sanyan_common_ui` / `sanyan_network` / `sanyan_routes` / `sanyan_user`

缺：
- ❌ `sanyan_util`（强制）
- ❌ `sanyan_bizkit`（强制）

业务包里目前散落的"应该下沉到 util"的代码：
- `business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart` 的 `_Timestamp.build()` 内联了 ISO 8601 字符串到 `HH:mm` 的格式化逻辑

### 1.2 目标

补齐 `sanyan_util` 包，把现有 inline 的日期格式化下沉为 `DateUtil.formatHHmm`。

### 1.3 非目标（明确不做）

- ❌ **不建 `sanyan_bizkit`**：当前只有 `sanyan_auth` + `sanyan_chat` 两个业务包，没有跨业务公共业务页面。建空壳违反 YAGNI。等真出现跨业务公共页面时再建（视为"规则的合理迭代场景"）。
- ❌ **不预留多类工具骨架**：只下沉现有重复代码，不提前建 PhoneUtil / StringUtil / CryptoUtil 等空壳类。未来需要时随手加。
- ❌ **不建 `sanyan_test_helpers`**：rule §9 说"按需建"，当前各包测试无重复 mock 类，不需要。

## 2. 包结构

```
foundation_packages/sanyan_util/
├── pubspec.yaml                       # name: sanyan_util，仅依赖 Flutter SDK
├── lib/
│   ├── sanyan_util.dart                # barrel
│   └── src/
│       └── date_util.dart              # DateUtil 类
└── test/
    ├── sanyan_util_suite.dart          # 测试聚合入口（rule §10 强制）
    └── date_util_test.dart
```

### 2.1 pubspec.yaml 内容

```yaml
name: sanyan_util
description: 三言 - 通用工具类
version: 0.1.0
publish_to: 'none'

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: ">=3.0.0"

dependencies:
  flutter:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
```

### 2.2 barrel `sanyan_util.dart`

```dart
export 'src/date_util.dart';
```

## 3. API 设计

### 3.1 DateUtil

```dart
/// 日期/时间格式化工具。
///
/// 按 ~/.claude/CLAUDE.md "static 优先原则"：abstract class + 全 static 方法，
/// 不可实例化。abstract 已阻止 `new`，不需要 private constructor。
abstract class DateUtil {
  /// 把 ISO 8601 字符串格式化为本地时区 HH:mm。
  ///
  /// - null/空串/解析失败 → 返回 ""（保持跟现有 message_bubble inline 实现的兜底行为一致）
  /// - 合法 ISO 8601 字符串 → 转本地时区后取小时分钟，左补 0 至 2 位
  ///
  /// 示例：
  /// - `DateUtil.formatHHmm('2026-05-14T22:05:30Z')` → 北京时区 → "06:05"（次日凌晨）
  /// - `DateUtil.formatHHmm('2026-05-14T14:05:30+08:00')` → 北京时区 → "14:05"
  /// - `DateUtil.formatHHmm(null)` → ""
  /// - `DateUtil.formatHHmm('garbage')` → ""
  static String formatHHmm(String? iso8601);
}
```

### 3.2 为何用 static 工具类而非 String 扩展

权衡过 `extension on String { String toHHmm() }`，决定**用 static 工具类**，理由：

| 维度 | static util | extension on String |
|---|---|---|
| 语义主体 | "格式化日期" 主体清晰 | "字符串.格式化为时分" 主体错（String 可以是任意内容） |
| namespace 污染 | 无 | 项目所有 String 都多 `.toHHmm()`，包括无意义场景 |
| 与规则一致 | rule §8 工具类 `<Role>Util` 命名直接对齐 | — |
| 与后端工具类一致 | 后端 `TextProcessor` / `TypingDelayCalculator` 也走 static utility class | — |

如未来要做 `Duration.toHumanReadable()` 这类**"主体即操作对象"**的扩展场景，再单独引入 extension，不影响本设计。

## 4. 使用方改动

### 4.1 sanyan_chat pubspec 加依赖

`business_packages/sanyan_chat/pubspec.yaml` 的 `dependencies:` 段加：

```yaml
sanyan_util:
  path: ../../foundation_packages/sanyan_util
```

### 4.2 message_bubble.dart 替换 inline 逻辑

`business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart` 的 `_Timestamp.build()`：

**改前**：
```dart
String t = '';
try {
  final dt = DateTime.parse(createdAt).toLocal();
  t = '${dt.hour.toString().padLeft(2, '0')}:${dt.minute.toString().padLeft(2, '0')}';
} catch (_) {}
return Text(t, ...);
```

**改后**：
```dart
import 'package:sanyan_util/sanyan_util.dart';
// ...
return Text(DateUtil.formatHHmm(createdAt), ...);
```

注意：`createdAt` 字段在现有代码里是 `String` 类型（来自 `Message.fromJson`），匹配 `DateUtil.formatHHmm(String? iso8601)` 签名。

## 5. 测试设计（TDD 必须）

`test/date_util_test.dart`：

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
      // 该 case 用相对断言避免硬编码本地时区：parse + toLocal 后必然是 HH:mm 格式
      final result = DateUtil.formatHHmm('2026-05-14T14:05:30Z');
      expect(result, matches(RegExp(r'^\d{2}:\d{2}$')));
    });
  });
}
```

测试聚合入口 `test/sanyan_util_suite.dart`：

```dart
import 'date_util_test.dart' as date_util;

void main() {
  date_util.main();
}
```

## 6. 实施路径（3 个 task）

| Task | 内容 | 验收 |
|---|---|---|
| **3.1** | 建 sanyan_util 包骨架（pubspec/barrel/空目录/suite 文件） | `flutter pub get` 在 sanyan_util 下成功 |
| **3.2** | TDD: 写 6 个 test → 跑 RED → 写 DateUtil.formatHHmm → 跑 GREEN → commit | `flutter test` 6 个全过 |
| **3.3** | sanyan_chat pubspec 加依赖；message_bubble 替换 inline 逻辑；手测 app 气泡时间显示正常；commit | app 跑起来气泡时间显示跟之前一致；sanyan_chat 测试如有未破 |

## 7. 风险

无重大风险。

| ID | 风险 | 缓解 |
|---|---|---|
| R1 | message_bubble 替换后时间显示不准 | TDD 5 个 case 覆盖了关键场景；手测 app 验证显示 |
| R2 | sanyan_util 依赖路径写错导致 pubspec 解析失败 | 借鉴现有 4 个 foundation 包的 pubspec 写法 |

## 8. 分支策略

直接在 app 子模块的 `dev` 分支做（P3 工作量小 + 风险低，无需独立 feature 分支）。3 个 task 各一个 commit；M3.3 完成 + 手测通过后合并到 master。

## 9. 验收标准

- [ ] `foundation_packages/sanyan_util/` 目录存在 + pubspec.yaml + barrel + DateUtil + 6 个 test + suite
- [ ] `flutter test` 6 个 test 全过
- [ ] sanyan_chat pubspec 含 sanyan_util 依赖
- [ ] message_bubble.dart 不再有 `DateTime.parse + padLeft` inline 逻辑，改用 `DateUtil.formatHHmm`
- [ ] app 跑起来后气泡时间显示正常（手测）
- [ ] 现有 sanyan_chat 测试如有未破

## 10. 范围外（后续 sub-plan）

- ❌ `sanyan_bizkit` 包：等真出现跨业务公共页面时再建
- ❌ `sanyan_test_helpers` 包：等多包出现重复 mock 类时再建
- ❌ DateUtil 其他方法（formatYMD / smartTimeDisplay 中文今天昨天等）：未来真有需求时加
- ❌ PhoneUtil / StringUtil 等其他工具类：同上
