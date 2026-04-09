# Stitch UI Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将三言 App 的 Flutter UI 从暖阳橘主题完全切换为 Stitch Ethereal Editorial 风格（薄荷绿色系、毛玻璃、Manrope 字体）。

**Architecture:** 先替换设计系统（颜色/字体/组件），再逐页重写 UI。每个 Task 独立可编译。共用组件抽到 sanyan_common_ui，页面在各自的 business_packages 中。

**Tech Stack:** Flutter 3.41.6 (FVM) / Dart / GetX / google_fonts / audioplayers

---

## File Structure

### 新增文件
| 文件 | 职责 |
|------|------|
| `foundation_packages/sanyan_common_ui/lib/src/aura_colors.dart` | Ethereal Editorial 完整色板 |
| `foundation_packages/sanyan_common_ui/lib/src/aura_theme.dart` | 全局 ThemeData |
| `foundation_packages/sanyan_common_ui/lib/src/widgets/glass_panel.dart` | 毛玻璃容器组件 |
| `foundation_packages/sanyan_common_ui/lib/src/widgets/aura_input.dart` | 输入框组件 |
| `foundation_packages/sanyan_common_ui/lib/src/widgets/aura_button.dart` | 渐变按钮组件 |
| `foundation_packages/sanyan_common_ui/lib/src/widgets/aura_nav_bar.dart` | 底部导航栏组件 |
| `business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart` | 语音消息气泡 |
| `business_packages/sanyan_chat/lib/src/chat/widget/video_bubble.dart` | 视频消息气泡 |
| `business_packages/sanyan_chat/lib/src/contacts/contacts_page.dart` | 联系人页 |
| `business_packages/sanyan_chat/lib/src/settings/settings_page.dart` | 设置页 |
| `business_packages/sanyan_chat/lib/src/status/status_page.dart` | 状态页（占位） |
| `assets/images/logo.png` | Logo 图片（已放入） |

### 修改文件
| 文件 | 改动 |
|------|------|
| `pubspec.yaml` | 添加 google_fonts + 声明 assets |
| `foundation_packages/sanyan_common_ui/lib/sanyan_common_ui.dart` | 导出新文件 |
| `foundation_packages/sanyan_routes/lib/src/routes.dart` | 添加 contacts/settings/status 路由 |
| `lib/main.dart` | 切换主题 + 添加新路由 |
| `business_packages/sanyan_auth/lib/src/auth/login_page.dart` | 完全重写 |
| `business_packages/sanyan_auth/lib/src/auth/register_page.dart` | 完全重写 |
| `business_packages/sanyan_chat/lib/src/home/home_page.dart` | 完全重写 |
| `business_packages/sanyan_chat/lib/src/chat/chat_page.dart` | 完全重写 |
| `business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart` | 完全重写 |
| `business_packages/sanyan_chat/lib/src/chat/widget/chat_input_bar.dart` | 完全重写 |
| `business_packages/sanyan_chat/lib/sanyan_chat.dart` | 导出新页面 |

### 删除文件
| 文件 | 原因 |
|------|------|
| `foundation_packages/sanyan_common_ui/lib/src/theme.dart` | 被 aura_colors.dart + aura_theme.dart 替代 |

---

### Task 1: 设计系统 — 色板 + 主题 + 依赖

**Files:**
- Create: `app/foundation_packages/sanyan_common_ui/lib/src/aura_colors.dart`
- Create: `app/foundation_packages/sanyan_common_ui/lib/src/aura_theme.dart`
- Delete: `app/foundation_packages/sanyan_common_ui/lib/src/theme.dart`
- Modify: `app/foundation_packages/sanyan_common_ui/lib/sanyan_common_ui.dart`
- Modify: `app/pubspec.yaml`
- Modify: `app/lib/main.dart`

- [ ] **Step 1: 添加 google_fonts 依赖和 assets 声明**

在 `app/pubspec.yaml` 的 dependencies 中添加：
```yaml
  google_fonts: ^6.2.1
```

在 flutter 节下添加 assets：
```yaml
flutter:
  uses-material-design: true
  assets:
    - assets/images/
```

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter pub get`

- [ ] **Step 2: 创建 AuraColors**

创建 `app/foundation_packages/sanyan_common_ui/lib/src/aura_colors.dart`：

```dart
import 'package:flutter/material.dart';

class AuraColors {
  // Brand
  static const Color primary = Color(0xFF006B64);
  static const Color primaryFixed = Color(0xFF73F1E4);
  static const Color primaryDim = Color(0xFF005E57);
  static const Color primaryContainer = Color(0xFF73F1E4);
  static const Color secondary = Color(0xFF006595);
  static const Color secondaryFixedDim = Color(0xFFAED9FF);

  // Surface hierarchy
  static const Color surface = Color(0xFFE2FFFF);
  static const Color surfaceDim = Color(0xFFA1E6E7);
  static const Color surfaceContainerHighest = Color(0xFFB0EEEE);
  static const Color surfaceContainerHigh = Color(0xFFBCF2F3);
  static const Color surfaceContainer = Color(0xFFC8F7F7);
  static const Color surfaceContainerLow = Color(0xFFD4FBFB);
  static const Color surfaceContainerLowest = Color(0xFFFFFFFF);

  // Text
  static const Color onSurface = Color(0xFF00393A);
  static const Color onSurfaceVariant = Color(0xFF2A6869);
  static const Color onPrimary = Color(0xFFE2FFFA);

  // Borders
  static const Color outlineVariant = Color(0xFF80BCBD);
  static const Color outline = Color(0xFF488485);

  // Accent
  static const Color tertiary = Color(0xFF6E4DA7);
  static const Color tertiaryContainer = Color(0xFFC9AAFF);
  static const Color error = Color(0xFFAC3434);

  // Gradients
  static const LinearGradient mintAzureGradient = LinearGradient(
    colors: [primaryFixed, secondaryFixedDim],
  );

  static const LinearGradient userBubbleGradient = LinearGradient(
    begin: Alignment.topLeft,
    end: Alignment.bottomRight,
    colors: [primary, secondary],
  );

  // Glass panel
  static const double glassBlur = 24.0;
  static Color glassColor = Colors.white.withValues(alpha: 0.4);
  static Color glassBorder = Colors.white.withValues(alpha: 0.2);
  static BoxShadow glassShadow = BoxShadow(
    color: const Color(0xFF00393A).withValues(alpha: 0.06),
    blurRadius: 40,
    offset: const Offset(0, 20),
  );
}
```

- [ ] **Step 3: 创建 AuraTheme**

创建 `app/foundation_packages/sanyan_common_ui/lib/src/aura_theme.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'aura_colors.dart';

class AuraTheme {
  static ThemeData get light => ThemeData(
        useMaterial3: true,
        scaffoldBackgroundColor: AuraColors.surface,
        fontFamily: GoogleFonts.manrope().fontFamily,
        appBarTheme: AppBarTheme(
          backgroundColor: Colors.transparent,
          foregroundColor: AuraColors.primary,
          elevation: 0,
          scrolledUnderElevation: 0,
          centerTitle: false,
          titleTextStyle: GoogleFonts.manrope(
            fontSize: 18,
            fontWeight: FontWeight.w700,
            color: AuraColors.primary,
          ),
          iconTheme: const IconThemeData(
            color: AuraColors.primary,
            size: 24,
          ),
        ),
        colorScheme: ColorScheme.fromSeed(
          seedColor: AuraColors.primary,
          surface: AuraColors.surface,
          onSurface: AuraColors.onSurface,
          primary: AuraColors.primary,
          secondary: AuraColors.secondary,
        ),
      );
}
```

- [ ] **Step 4: 更新导出文件**

替换 `app/foundation_packages/sanyan_common_ui/lib/sanyan_common_ui.dart`：

```dart
export 'src/constants.dart';
export 'src/aura_colors.dart';
export 'src/aura_theme.dart';
```

- [ ] **Step 5: 删除旧主题**

删除 `app/foundation_packages/sanyan_common_ui/lib/src/theme.dart`

- [ ] **Step 6: 更新 main.dart**

在 `app/lib/main.dart` 中：
- 将 `import 'package:sanyan_common_ui/sanyan_common_ui.dart'` 保留
- 将 `theme: AppTheme.light` 改为 `theme: AuraTheme.light`

- [ ] **Step 7: 修复所有 AppTheme/AppColors 引用**

全局搜索替换：
- `AppColors.` → `AuraColors.`（在所有引用旧色的文件中）
- `AppTheme.light` → `AuraTheme.light`

暂时让编译通过，后续 Task 会重写这些页面。对于临时引用不到的颜色（如 brandGradient、brandButton 等），在 AuraColors 中添加兼容别名：

```dart
// 兼容旧代码（后续 Task 重写页面后删除）
static const LinearGradient brandGradient = mintAzureGradient;
static const LinearGradient buttonGradient = mintAzureGradient;
static const Color brandButton = primary;
static const Color brandStart = primaryFixed;
static const Color brandEnd = secondary;
static const Color accent = primary;
static const Color textPrimary = onSurface;
static const Color textSecondary = onSurfaceVariant;
static const Color textPlaceholder = outlineVariant;
static const Color background = surfaceContainerLowest;
static const Color inputFill = surfaceContainerHighest;
static const Color divider = surfaceContainerLow;
static const Color border = outlineVariant;
static const Color aiBubble = surfaceContainerLowest;
```

- [ ] **Step 8: 验证编译**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 9: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: Ethereal Editorial 设计系统 — AuraColors + AuraTheme + google_fonts"
```

---

### Task 2: 共用组件 — GlassPanel + AuraInput + AuraButton + AuraNavBar

**Files:**
- Create: `app/foundation_packages/sanyan_common_ui/lib/src/widgets/glass_panel.dart`
- Create: `app/foundation_packages/sanyan_common_ui/lib/src/widgets/aura_input.dart`
- Create: `app/foundation_packages/sanyan_common_ui/lib/src/widgets/aura_button.dart`
- Create: `app/foundation_packages/sanyan_common_ui/lib/src/widgets/aura_nav_bar.dart`
- Modify: `app/foundation_packages/sanyan_common_ui/lib/sanyan_common_ui.dart`

- [ ] **Step 1: 创建 GlassPanel**

`app/foundation_packages/sanyan_common_ui/lib/src/widgets/glass_panel.dart`：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import '../aura_colors.dart';

class GlassPanel extends StatelessWidget {
  final Widget child;
  final double borderRadius;
  final EdgeInsetsGeometry? padding;

  const GlassPanel({
    super.key,
    required this.child,
    this.borderRadius = 16,
    this.padding,
  });

  @override
  Widget build(BuildContext context) {
    return ClipRRect(
      borderRadius: BorderRadius.circular(borderRadius),
      child: BackdropFilter(
        filter: ImageFilter.blur(
          sigmaX: AuraColors.glassBlur,
          sigmaY: AuraColors.glassBlur,
        ),
        child: Container(
          padding: padding,
          decoration: BoxDecoration(
            color: AuraColors.glassColor,
            borderRadius: BorderRadius.circular(borderRadius),
            border: Border.all(color: AuraColors.glassBorder),
            boxShadow: [AuraColors.glassShadow],
          ),
          child: child,
        ),
      ),
    );
  }
}
```

- [ ] **Step 2: 创建 AuraInput**

`app/foundation_packages/sanyan_common_ui/lib/src/widgets/aura_input.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import '../aura_colors.dart';

class AuraInput extends StatelessWidget {
  final TextEditingController? controller;
  final String? label;
  final String hint;
  final IconData? prefixIcon;
  final bool obscureText;
  final TextInputType? keyboardType;
  final Widget? suffix;
  final ValueChanged<String>? onSubmitted;

  const AuraInput({
    super.key,
    this.controller,
    this.label,
    required this.hint,
    this.prefixIcon,
    this.obscureText = false,
    this.keyboardType,
    this.suffix,
    this.onSubmitted,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        if (label != null) ...[
          Padding(
            padding: const EdgeInsets.only(left: 4, bottom: 8),
            child: Text(
              label!.toUpperCase(),
              style: GoogleFonts.inter(
                fontSize: 10,
                fontWeight: FontWeight.w700,
                color: AuraColors.onSurfaceVariant,
                letterSpacing: 1.5,
              ),
            ),
          ),
        ],
        Container(
          height: 56,
          decoration: BoxDecoration(
            color: AuraColors.surfaceContainerHighest.withValues(alpha: 0.5),
            borderRadius: BorderRadius.circular(12),
          ),
          child: Row(
            children: [
              if (prefixIcon != null) ...[
                const SizedBox(width: 16),
                Icon(prefixIcon, color: AuraColors.onSurfaceVariant, size: 20),
                const SizedBox(width: 12),
              ] else
                const SizedBox(width: 16),
              Expanded(
                child: TextField(
                  controller: controller,
                  obscureText: obscureText,
                  keyboardType: keyboardType,
                  onSubmitted: onSubmitted,
                  style: GoogleFonts.manrope(
                    fontSize: 15,
                    color: AuraColors.onSurface,
                  ),
                  decoration: InputDecoration(
                    hintText: hint,
                    hintStyle: GoogleFonts.manrope(
                      fontSize: 15,
                      color: AuraColors.onSurfaceVariant.withValues(alpha: 0.4),
                    ),
                    border: InputBorder.none,
                    contentPadding: EdgeInsets.zero,
                    isDense: true,
                  ),
                ),
              ),
              if (suffix != null) suffix!,
              const SizedBox(width: 16),
            ],
          ),
        ),
      ],
    );
  }
}
```

- [ ] **Step 3: 创建 AuraButton**

`app/foundation_packages/sanyan_common_ui/lib/src/widgets/aura_button.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import '../aura_colors.dart';

class AuraButton extends StatelessWidget {
  final String text;
  final VoidCallback? onTap;
  final bool isLoading;
  final IconData? trailingIcon;

  const AuraButton({
    super.key,
    required this.text,
    this.onTap,
    this.isLoading = false,
    this.trailingIcon,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: isLoading ? null : onTap,
      child: Container(
        height: 56,
        decoration: BoxDecoration(
          gradient: AuraColors.mintAzureGradient,
          borderRadius: BorderRadius.circular(9999),
          boxShadow: [
            BoxShadow(
              color: AuraColors.primaryFixed.withValues(alpha: 0.3),
              blurRadius: 25,
              offset: const Offset(0, 10),
            ),
          ],
        ),
        child: Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            if (isLoading)
              const SizedBox(
                width: 20,
                height: 20,
                child: CircularProgressIndicator(
                  strokeWidth: 2,
                  color: Color(0xFF00393A),
                ),
              )
            else ...[
              Text(
                text,
                style: GoogleFonts.manrope(
                  fontSize: 16,
                  fontWeight: FontWeight.w700,
                  color: const Color(0xFF004366),
                ),
              ),
              if (trailingIcon != null) ...[
                const SizedBox(width: 8),
                Icon(trailingIcon, color: const Color(0xFF004366), size: 20),
              ],
            ],
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: 创建 AuraNavBar**

`app/foundation_packages/sanyan_common_ui/lib/src/widgets/aura_nav_bar.dart`：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import '../aura_colors.dart';

class AuraNavBar extends StatelessWidget {
  final int currentIndex;
  final ValueChanged<int> onTap;

  const AuraNavBar({
    super.key,
    required this.currentIndex,
    required this.onTap,
  });

  static const _items = [
    _NavItem(Icons.chat_bubble, 'Messages'),
    _NavItem(Icons.group, 'Contacts'),
    _NavItem(Icons.flare, 'Status'),
    _NavItem(Icons.settings, 'Settings'),
  ];

  @override
  Widget build(BuildContext context) {
    return ClipRRect(
      borderRadius: const BorderRadius.vertical(top: Radius.circular(24)),
      child: BackdropFilter(
        filter: ImageFilter.blur(sigmaX: 24, sigmaY: 24),
        child: Container(
          decoration: BoxDecoration(
            color: AuraColors.surface.withValues(alpha: 0.8),
            borderRadius: const BorderRadius.vertical(top: Radius.circular(24)),
            boxShadow: [
              BoxShadow(
                color: const Color(0xFF00393A).withValues(alpha: 0.04),
                blurRadius: 30,
                offset: const Offset(0, -10),
              ),
            ],
          ),
          child: SafeArea(
            top: false,
            child: Padding(
              padding: const EdgeInsets.fromLTRB(16, 8, 16, 8),
              child: Row(
                mainAxisAlignment: MainAxisAlignment.spaceAround,
                children: List.generate(_items.length, (i) {
                  final item = _items[i];
                  final active = i == currentIndex;
                  return GestureDetector(
                    onTap: () => onTap(i),
                    behavior: HitTestBehavior.opaque,
                    child: _buildTab(item, active),
                  );
                }),
              ),
            ),
          ),
        ),
      ),
    );
  }

  Widget _buildTab(_NavItem item, bool active) {
    if (active) {
      return Container(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
        decoration: BoxDecoration(
          gradient: AuraColors.mintAzureGradient,
          borderRadius: BorderRadius.circular(9999),
        ),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(item.icon, size: 22, color: AuraColors.onSurface),
            const SizedBox(height: 2),
            Text(
              item.label.toUpperCase(),
              style: GoogleFonts.manrope(
                fontSize: 10,
                fontWeight: FontWeight.w600,
                letterSpacing: 0.5,
                color: AuraColors.onSurface,
              ),
            ),
          ],
        ),
      );
    }

    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 10),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(item.icon, size: 22, color: AuraColors.onSurfaceVariant),
          const SizedBox(height: 2),
          Text(
            item.label.toUpperCase(),
            style: GoogleFonts.manrope(
              fontSize: 10,
              fontWeight: FontWeight.w600,
              letterSpacing: 0.5,
              color: AuraColors.onSurfaceVariant,
            ),
          ),
        ],
      ),
    );
  }
}

class _NavItem {
  final IconData icon;
  final String label;
  const _NavItem(this.icon, this.label);
}
```

- [ ] **Step 5: 更新导出**

更新 `app/foundation_packages/sanyan_common_ui/lib/sanyan_common_ui.dart`：

```dart
export 'src/constants.dart';
export 'src/aura_colors.dart';
export 'src/aura_theme.dart';
export 'src/widgets/glass_panel.dart';
export 'src/widgets/aura_input.dart';
export 'src/widgets/aura_button.dart';
export 'src/widgets/aura_nav_bar.dart';
```

- [ ] **Step 6: 验证编译**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`
Expected: No issues found

- [ ] **Step 7: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "feat: 共用组件 — GlassPanel + AuraInput + AuraButton + AuraNavBar"
```

---

### Task 3: 登录页重写

**Files:**
- Modify: `app/business_packages/sanyan_auth/lib/src/auth/login_page.dart`

- [ ] **Step 1: 重写登录页**

完全替换 `app/business_packages/sanyan_auth/lib/src/auth/login_page.dart`：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';
import 'package:sanyan_routes/sanyan_routes.dart';
import 'login_controller.dart';

class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    final c = Get.put(LoginController());
    return Scaffold(
      backgroundColor: Colors.transparent,
      body: Container(
        decoration: const BoxDecoration(
          gradient: RadialGradient(
            center: Alignment(-0.8, -0.6),
            radius: 1.5,
            colors: [
              AuraColors.primaryFixed,
              AuraColors.surface,
              AuraColors.secondaryFixedDim,
            ],
            stops: [0.0, 0.4, 1.0],
          ),
        ),
        child: SafeArea(
          child: Center(
            child: SingleChildScrollView(
              padding: const EdgeInsets.symmetric(horizontal: 24),
              child: GlassPanel(
                padding: const EdgeInsets.all(32),
                child: Column(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    // Logo
                    Image.asset('assets/images/logo.png', height: 80),
                    const SizedBox(height: 16),
                    Text(
                      '走进更温暖的连接方式',
                      style: GoogleFonts.manrope(
                        fontSize: 14,
                        fontWeight: FontWeight.w500,
                        color: AuraColors.onSurfaceVariant,
                      ),
                    ),
                    const SizedBox(height: 40),

                    // Phone input
                    AuraInput(
                      controller: c.phoneController,
                      label: '手机号',
                      hint: '请输入手机号',
                      prefixIcon: Icons.alternate_email,
                      keyboardType: TextInputType.phone,
                    ),
                    const SizedBox(height: 20),

                    // Password input
                    Obx(() => AuraInput(
                          controller: c.passwordController,
                          label: '密码',
                          hint: '请输入密码',
                          prefixIcon: Icons.lock_outline,
                          obscureText: c.obscurePassword.value,
                          onSubmitted: (_) => c.login(),
                          suffix: GestureDetector(
                            onTap: () => c.obscurePassword.toggle(),
                            child: Icon(
                              c.obscurePassword.value
                                  ? Icons.visibility_off
                                  : Icons.visibility,
                              color: AuraColors.onSurfaceVariant,
                              size: 20,
                            ),
                          ),
                        )),
                    const SizedBox(height: 28),

                    // Login button
                    Obx(() => AuraButton(
                          text: '登录',
                          trailingIcon: Icons.arrow_forward,
                          isLoading: c.isLoading.value,
                          onTap: c.login,
                        )),
                    const SizedBox(height: 32),

                    // Divider
                    Container(
                      height: 1,
                      color: AuraColors.outlineVariant.withValues(alpha: 0.1),
                    ),
                    const SizedBox(height: 24),

                    // Links
                    Text(
                      '忘记密码？',
                      style: GoogleFonts.manrope(
                        fontSize: 14,
                        fontWeight: FontWeight.w500,
                        color: AuraColors.onSurfaceVariant,
                      ),
                    ),
                    const SizedBox(height: 12),
                    Row(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        Text(
                          '还没有账号？',
                          style: GoogleFonts.manrope(
                            fontSize: 14,
                            color: AuraColors.onSurfaceVariant.withValues(alpha: 0.6),
                          ),
                        ),
                        GestureDetector(
                          onTap: () => Get.toNamed(AppRoutes.register),
                          child: Text(
                            '立即注册',
                            style: GoogleFonts.manrope(
                              fontSize: 14,
                              fontWeight: FontWeight.w700,
                              color: AuraColors.primary,
                            ),
                          ),
                        ),
                      ],
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
}
```

- [ ] **Step 2: 验证编译**

Run: `cd /Users/aventador/sourceCode/3yan/app && /Users/aventador/fvm/versions/3.41.6/bin/flutter analyze`

- [ ] **Step 3: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add business_packages/sanyan_auth/lib/src/auth/login_page.dart
git commit -m "feat: 登录页 Ethereal Editorial 风格重写"
```

---

### Task 4: 注册页重写

**Files:**
- Modify: `app/business_packages/sanyan_auth/lib/src/auth/register_page.dart`

- [ ] **Step 1: 重写注册页**

完全替换 `app/business_packages/sanyan_auth/lib/src/auth/register_page.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';
import 'register_controller.dart';

class RegisterPage extends StatelessWidget {
  const RegisterPage({super.key});

  @override
  Widget build(BuildContext context) {
    final c = Get.put(RegisterController());
    return Scaffold(
      backgroundColor: Colors.transparent,
      body: Container(
        decoration: const BoxDecoration(
          gradient: RadialGradient(
            center: Alignment(-0.8, -0.6),
            radius: 1.5,
            colors: [
              AuraColors.primaryFixed,
              AuraColors.surface,
              AuraColors.secondaryFixedDim,
            ],
            stops: [0.0, 0.4, 1.0],
          ),
        ),
        child: SafeArea(
          child: Center(
            child: SingleChildScrollView(
              padding: const EdgeInsets.symmetric(horizontal: 24),
              child: GlassPanel(
                padding: const EdgeInsets.all(32),
                child: Column(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    // Logo
                    Image.asset('assets/images/logo.png', height: 56),
                    const SizedBox(height: 16),
                    Text(
                      '加入三言',
                      style: GoogleFonts.manrope(
                        fontSize: 28,
                        fontWeight: FontWeight.w800,
                        color: AuraColors.onSurface,
                      ),
                    ),
                    const SizedBox(height: 4),
                    Text(
                      '填写信息创建你的账号',
                      style: GoogleFonts.manrope(
                        fontSize: 14,
                        color: AuraColors.onSurfaceVariant,
                      ),
                    ),
                    const SizedBox(height: 28),

                    // Nickname
                    AuraInput(
                      controller: c.nicknameController,
                      label: '昵称',
                      hint: '给自己取个名字',
                      prefixIcon: Icons.person_outline,
                    ),
                    const SizedBox(height: 16),

                    // Phone
                    AuraInput(
                      controller: c.phoneController,
                      label: '手机号',
                      hint: '请输入手机号',
                      prefixIcon: Icons.phone_outlined,
                      keyboardType: TextInputType.phone,
                    ),
                    const SizedBox(height: 16),

                    // Code
                    Row(
                      crossAxisAlignment: CrossAxisAlignment.end,
                      children: [
                        Expanded(
                          child: AuraInput(
                            controller: c.codeController,
                            label: '验证码',
                            hint: '请输入验证码',
                            prefixIcon: Icons.sms_outlined,
                            keyboardType: TextInputType.number,
                          ),
                        ),
                        const SizedBox(width: 12),
                        Obx(() => GestureDetector(
                              onTap: c.countdown.value > 0 ? null : c.sendSms,
                              child: Container(
                                height: 56,
                                padding: const EdgeInsets.symmetric(horizontal: 16),
                                decoration: BoxDecoration(
                                  color: AuraColors.surfaceContainerLowest,
                                  borderRadius: BorderRadius.circular(12),
                                  border: Border.all(
                                    color: AuraColors.outlineVariant.withValues(alpha: 0.3),
                                  ),
                                ),
                                alignment: Alignment.center,
                                child: Text(
                                  c.countdown.value > 0
                                      ? '${c.countdown.value}s'
                                      : '获取验证码',
                                  style: GoogleFonts.inter(
                                    fontSize: 13,
                                    fontWeight: FontWeight.w600,
                                    color: c.countdown.value > 0
                                        ? AuraColors.onSurfaceVariant
                                        : AuraColors.primary,
                                  ),
                                ),
                              ),
                            )),
                      ],
                    ),
                    const SizedBox(height: 16),

                    // Password
                    AuraInput(
                      controller: c.passwordController,
                      label: '密码',
                      hint: '设置密码（至少6位）',
                      prefixIcon: Icons.lock_outline,
                      obscureText: true,
                    ),
                    const SizedBox(height: 24),

                    // Register button
                    Obx(() => AuraButton(
                          text: '注册',
                          isLoading: c.isLoading.value,
                          onTap: c.register,
                        )),
                    const SizedBox(height: 24),

                    // Login link
                    Row(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        Text(
                          '已有账号？',
                          style: GoogleFonts.manrope(
                            fontSize: 14,
                            color: AuraColors.onSurfaceVariant.withValues(alpha: 0.6),
                          ),
                        ),
                        GestureDetector(
                          onTap: () => Get.back(),
                          child: Text(
                            '返回登录',
                            style: GoogleFonts.manrope(
                              fontSize: 14,
                              fontWeight: FontWeight.w700,
                              color: AuraColors.primary,
                            ),
                          ),
                        ),
                      ],
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
}
```

- [ ] **Step 2: 验证编译 + 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
/Users/aventador/fvm/versions/3.41.6/bin/flutter analyze
git add business_packages/sanyan_auth/lib/src/auth/register_page.dart
git commit -m "feat: 注册页 Ethereal Editorial 风格重写"
```

---

### Task 5: 消息列表页（首页）重写 + 路由 + 主框架

**Files:**
- Modify: `app/foundation_packages/sanyan_routes/lib/src/routes.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/contacts/contacts_page.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/settings/settings_page.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/status/status_page.dart`
- Modify: `app/business_packages/sanyan_chat/lib/sanyan_chat.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/home/home_page.dart`
- Modify: `app/lib/main.dart`

- [ ] **Step 1: 添加路由**

更新 `app/foundation_packages/sanyan_routes/lib/src/routes.dart`：

```dart
class AppRoutes {
  static const splash = '/splash';
  static const login = '/login';
  static const register = '/register';
  static const home = '/home';
  static const chat = '/chat';
  static const contacts = '/contacts';
  static const settings = '/settings';
  static const status = '/status';
}
```

- [ ] **Step 2: 创建占位页面**

`app/business_packages/sanyan_chat/lib/src/contacts/contacts_page.dart`：
```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';

class ContactsPage extends StatelessWidget {
  const ContactsPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                'Contacts',
                style: GoogleFonts.manrope(
                  fontSize: 28,
                  fontWeight: FontWeight.w800,
                  color: AuraColors.onSurface,
                ),
              ),
              const SizedBox(height: 8),
              Text(
                '敬请期待',
                style: GoogleFonts.manrope(
                  fontSize: 14,
                  color: AuraColors.onSurfaceVariant,
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

`app/business_packages/sanyan_chat/lib/src/settings/settings_page.dart`：
```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';

class SettingsPage extends StatelessWidget {
  const SettingsPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                'Settings',
                style: GoogleFonts.manrope(
                  fontSize: 28,
                  fontWeight: FontWeight.w800,
                  color: AuraColors.onSurface,
                ),
              ),
              const SizedBox(height: 8),
              Text(
                '敬请期待',
                style: GoogleFonts.manrope(
                  fontSize: 14,
                  color: AuraColors.onSurfaceVariant,
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

`app/business_packages/sanyan_chat/lib/src/status/status_page.dart`：
```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';

class StatusPage extends StatelessWidget {
  const StatusPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                'Status',
                style: GoogleFonts.manrope(
                  fontSize: 28,
                  fontWeight: FontWeight.w800,
                  color: AuraColors.onSurface,
                ),
              ),
              const SizedBox(height: 8),
              Text(
                '敬请期待',
                style: GoogleFonts.manrope(
                  fontSize: 14,
                  color: AuraColors.onSurfaceVariant,
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

- [ ] **Step 3: 更新 sanyan_chat 导出**

在 `app/business_packages/sanyan_chat/lib/sanyan_chat.dart` 中添加导出：
```dart
export 'src/contacts/contacts_page.dart';
export 'src/settings/settings_page.dart';
export 'src/status/status_page.dart';
```

- [ ] **Step 4: 重写消息列表页**

完全替换 `app/business_packages/sanyan_chat/lib/src/home/home_page.dart`：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';
import 'package:sanyan_routes/sanyan_routes.dart';
import '../contacts/contacts_page.dart';
import '../settings/settings_page.dart';
import '../status/status_page.dart';
import '../models/conversation.dart';
import 'home_controller.dart';

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    final c = Get.put(HomeController());
    final currentTab = 0.obs;

    final pages = [
      _MessagesTab(c: c),
      const ContactsPage(),
      const StatusPage(),
      const SettingsPage(),
    ];

    return Scaffold(
      body: Obx(() => IndexedStack(
            index: currentTab.value,
            children: pages,
          )),
      bottomNavigationBar: Obx(() => AuraNavBar(
            currentIndex: currentTab.value,
            onTap: (i) => currentTab.value = i,
          )),
    );
  }
}

class _MessagesTab extends StatelessWidget {
  final HomeController c;
  const _MessagesTab({required this.c});

  @override
  Widget build(BuildContext context) {
    return SafeArea(
      bottom: false,
      child: Column(
        children: [
          // Top bar
          _buildTopBar(),
          // Content
          Expanded(
            child: Obx(() {
              if (c.isLoading.value && c.conversations.isEmpty) {
                return const Center(
                  child: CircularProgressIndicator(color: AuraColors.primary),
                );
              }
              if (c.conversations.isEmpty) {
                return Center(
                  child: Text(
                    '还没有对话，开始聊天吧',
                    style: GoogleFonts.manrope(color: AuraColors.onSurfaceVariant),
                  ),
                );
              }
              return RefreshIndicator(
                color: AuraColors.primary,
                onRefresh: c.loadConversations,
                child: ListView(
                  padding: const EdgeInsets.fromLTRB(24, 0, 24, 100),
                  children: [
                    const SizedBox(height: 24),
                    // Search
                    Container(
                      height: 56,
                      decoration: BoxDecoration(
                        color: AuraColors.surfaceContainerHighest,
                        borderRadius: BorderRadius.circular(12),
                      ),
                      child: Row(
                        children: [
                          const SizedBox(width: 20),
                          const Icon(Icons.search, color: AuraColors.onSurfaceVariant),
                          const SizedBox(width: 12),
                          Text(
                            '搜索对话...',
                            style: GoogleFonts.manrope(
                              color: AuraColors.onSurfaceVariant.withValues(alpha: 0.5),
                            ),
                          ),
                        ],
                      ),
                    ),
                    const SizedBox(height: 32),
                    // Messages title
                    Text(
                      'Messages',
                      style: GoogleFonts.manrope(
                        fontSize: 20,
                        fontWeight: FontWeight.w700,
                        color: AuraColors.onSurface,
                      ),
                    ),
                    const SizedBox(height: 16),
                    // Conversation list
                    ...c.conversations.map((conv) => _ConversationItem(conversation: conv)),
                  ],
                ),
              );
            }),
          ),
        ],
      ),
    );
  }

  Widget _buildTopBar() {
    return Padding(
      padding: const EdgeInsets.fromLTRB(24, 12, 24, 0),
      child: Row(
        children: [
          // Avatar placeholder
          Container(
            width: 40,
            height: 40,
            decoration: BoxDecoration(
              shape: BoxShape.circle,
              border: Border.all(color: AuraColors.primaryContainer, width: 2),
            ),
            child: ClipOval(
              child: Image.asset('assets/images/logo.png', fit: BoxFit.cover),
            ),
          ),
          const SizedBox(width: 12),
          ShaderMask(
            shaderCallback: (bounds) => const LinearGradient(
              colors: [AuraColors.primary, AuraColors.secondary],
            ).createShader(bounds),
            child: Text(
              '三言',
              style: GoogleFonts.manrope(
                fontSize: 24,
                fontWeight: FontWeight.w800,
                color: Colors.white,
              ),
            ),
          ),
          const Spacer(),
          const Icon(Icons.search, color: AuraColors.primary),
        ],
      ),
    );
  }
}

class _ConversationItem extends StatelessWidget {
  final Conversation conversation;
  const _ConversationItem({required this.conversation});

  @override
  Widget build(BuildContext context) {
    final hasUnread = conversation.unreadCount > 0;
    return GestureDetector(
      onTap: () => Get.toNamed(AppRoutes.chat, arguments: conversation),
      child: Container(
        padding: const EdgeInsets.all(20),
        margin: const EdgeInsets.only(bottom: 8),
        decoration: BoxDecoration(
          color: hasUnread ? AuraColors.surfaceContainerLowest : null,
          borderRadius: BorderRadius.circular(12),
          boxShadow: hasUnread
              ? [BoxShadow(
                  color: const Color(0xFF00393A).withValues(alpha: 0.04),
                  blurRadius: 40,
                  offset: const Offset(0, 20),
                )]
              : null,
        ),
        child: Row(
          children: [
            // Avatar
            Stack(
              clipBehavior: Clip.none,
              children: [
                Container(
                  width: 56,
                  height: 56,
                  decoration: const BoxDecoration(
                    shape: BoxShape.circle,
                    gradient: AuraColors.mintAzureGradient,
                  ),
                  child: const Icon(Icons.favorite, color: Colors.white, size: 24),
                ),
                Positioned(
                  right: 0,
                  bottom: 0,
                  child: Container(
                    width: 16,
                    height: 16,
                    decoration: BoxDecoration(
                      color: AuraColors.primaryFixed,
                      shape: BoxShape.circle,
                      border: Border.all(
                        color: hasUnread
                            ? AuraColors.surfaceContainerLowest
                            : AuraColors.surface,
                        width: 2,
                      ),
                    ),
                  ),
                ),
              ],
            ),
            const SizedBox(width: 16),
            // Info
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceBetween,
                    children: [
                      Text(
                        conversation.characterName ?? '未知',
                        style: GoogleFonts.manrope(
                          fontSize: 16,
                          fontWeight: FontWeight.w700,
                          color: hasUnread
                              ? AuraColors.onSurface
                              : AuraColors.onSurfaceVariant.withValues(alpha: 0.8),
                        ),
                      ),
                      Text(
                        _formatTime(conversation.lastMessageAt),
                        style: GoogleFonts.inter(
                          fontSize: 10,
                          fontWeight: hasUnread ? FontWeight.w600 : FontWeight.w500,
                          color: hasUnread
                              ? AuraColors.primary
                              : AuraColors.onSurfaceVariant.withValues(alpha: 0.6),
                          letterSpacing: 0.5,
                        ),
                      ),
                    ],
                  ),
                  const SizedBox(height: 4),
                  Row(
                    children: [
                      Expanded(
                        child: Text(
                          conversation.lastMessage ?? '',
                          maxLines: 1,
                          overflow: TextOverflow.ellipsis,
                          style: GoogleFonts.manrope(
                            fontSize: 14,
                            fontWeight: hasUnread ? FontWeight.w600 : FontWeight.w400,
                            color: hasUnread
                                ? AuraColors.onSurface
                                : AuraColors.onSurfaceVariant.withValues(alpha: 0.7),
                          ),
                        ),
                      ),
                      if (hasUnread) ...[
                        const SizedBox(width: 8),
                        Container(
                          width: 24,
                          height: 24,
                          decoration: const BoxDecoration(
                            gradient: AuraColors.mintAzureGradient,
                            shape: BoxShape.circle,
                          ),
                          alignment: Alignment.center,
                          child: Text(
                            '${conversation.unreadCount}',
                            style: GoogleFonts.inter(
                              fontSize: 10,
                              fontWeight: FontWeight.w700,
                              color: const Color(0xFF00443F),
                            ),
                          ),
                        ),
                      ],
                    ],
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }

  String _formatTime(String? time) {
    if (time == null) return '';
    try {
      final dt = DateTime.parse(time);
      final now = DateTime.now();
      final diff = now.difference(dt);
      if (diff.inMinutes < 1) return '刚刚';
      if (diff.inHours < 1) return '${diff.inMinutes}分钟前';
      if (diff.inDays < 1) return '${diff.inHours}小时前';
      if (diff.inDays < 7) return '${diff.inDays}天前';
      return '${dt.month}/${dt.day}';
    } catch (_) {
      return '';
    }
  }
}
```

- [ ] **Step 5: 更新 main.dart 路由**

在 `app/lib/main.dart` 中确保引入新页面并添加路由。

- [ ] **Step 6: 验证编译 + 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
/Users/aventador/fvm/versions/3.41.6/bin/flutter analyze
git add -A
git commit -m "feat: 消息列表页 + 4 tab 导航 + 占位页面"
```

---

### Task 6: 聊天页 + 消息气泡 + 输入栏重写

**Files:**
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/chat_page.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart`
- Modify: `app/business_packages/sanyan_chat/lib/src/chat/widget/chat_input_bar.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart`
- Create: `app/business_packages/sanyan_chat/lib/src/chat/widget/video_bubble.dart`

- [ ] **Step 1: 重写聊天页**

完全替换 `app/business_packages/sanyan_chat/lib/src/chat/chat_page.dart`：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';
import '../models/conversation.dart';
import 'chat_controller.dart';
import 'widget/chat_input_bar.dart';
import 'widget/message_bubble.dart';

class ChatPage extends StatelessWidget {
  ChatPage({super.key});

  final Conversation conversation = Get.arguments as Conversation;
  late final ChatController c = Get.put(ChatController(conversation));

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Container(
        decoration: const BoxDecoration(
          color: AuraColors.surface,
          image: DecorationImage(
            image: AssetImage('assets/images/logo.png'),
            opacity: 0.02,
            fit: BoxFit.none,
          ),
        ),
        child: SafeArea(
          child: Column(
            children: [
              _buildTopBar(),
              Container(height: 1, color: AuraColors.surfaceContainerLow),
              Expanded(
                child: Obx(() {
                  if (c.isLoading.value && c.messages.isEmpty) {
                    return const Center(
                      child: CircularProgressIndicator(color: AuraColors.primary),
                    );
                  }
                  return ListView.builder(
                    controller: c.scrollController,
                    padding: const EdgeInsets.symmetric(vertical: 16, horizontal: 4),
                    itemCount: c.messages.length,
                    itemBuilder: (context, index) =>
                        MessageBubble(message: c.messages[index]),
                  );
                }),
              ),
              ChatInputBar(
                controller: c.inputController,
                onSend: c.sendMessage,
              ),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildTopBar() {
    return ClipRRect(
      child: BackdropFilter(
        filter: ImageFilter.blur(sigmaX: 24, sigmaY: 24),
        child: Container(
          color: AuraColors.surface.withValues(alpha: 0.6),
          padding: const EdgeInsets.fromLTRB(16, 8, 16, 8),
          child: Row(
            children: [
              GestureDetector(
                onTap: () => Get.back(),
                child: const Icon(Icons.arrow_back, color: AuraColors.primary, size: 24),
              ),
              const SizedBox(width: 12),
              Container(
                width: 40,
                height: 40,
                decoration: BoxDecoration(
                  shape: BoxShape.circle,
                  border: Border.all(color: AuraColors.primaryContainer, width: 2),
                  gradient: AuraColors.mintAzureGradient,
                ),
                child: const Icon(Icons.favorite, color: Colors.white, size: 18),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      conversation.characterName ?? '',
                      style: GoogleFonts.manrope(
                        fontSize: 18,
                        fontWeight: FontWeight.w700,
                        color: AuraColors.primary,
                      ),
                    ),
                    Obx(() => c.isAiTyping.value
                        ? Row(
                            children: [
                              Container(
                                width: 6,
                                height: 6,
                                decoration: const BoxDecoration(
                                  color: AuraColors.primaryFixed,
                                  shape: BoxShape.circle,
                                ),
                              ),
                              const SizedBox(width: 4),
                              Text(
                                'ONLINE',
                                style: GoogleFonts.inter(
                                  fontSize: 10,
                                  fontWeight: FontWeight.w500,
                                  color: AuraColors.onSurfaceVariant,
                                  letterSpacing: 1.5,
                                ),
                              ),
                            ],
                          )
                        : const SizedBox.shrink()),
                  ],
                ),
              ),
              const Icon(Icons.search, color: AuraColors.primary),
              const SizedBox(width: 16),
              const Icon(Icons.more_vert, color: AuraColors.primary),
            ],
          ),
        ),
      ),
    );
  }
}
```

- [ ] **Step 2: 重写消息气泡**

完全替换 `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';
import '../../models/message.dart';
import 'voice_bubble.dart';

class MessageBubble extends StatelessWidget {
  final Message message;
  const MessageBubble({super.key, required this.message});

  @override
  Widget build(BuildContext context) {
    final isUser = message.senderType == 'user';
    final isVoice = message.isVoice;

    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 6),
      child: Column(
        crossAxisAlignment:
            isUser ? CrossAxisAlignment.end : CrossAxisAlignment.start,
        children: [
          Container(
            constraints: BoxConstraints(
              maxWidth: MediaQuery.of(context).size.width * 0.75,
            ),
            child: isVoice && !isUser
                ? VoiceBubble(message: message, isUser: false)
                : isVoice && isUser
                    ? VoiceBubble(message: message, isUser: true)
                    : _buildTextBubble(isUser),
          ),
          const SizedBox(height: 4),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 4),
            child: Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                Text(
                  _formatTime(message.createdAt),
                  style: GoogleFonts.inter(
                    fontSize: 10,
                    color: AuraColors.onSurfaceVariant.withValues(alpha: 0.7),
                  ),
                ),
                if (isUser) ...[
                  const SizedBox(width: 4),
                  Icon(Icons.done_all, size: 12, color: AuraColors.primary),
                ],
              ],
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildTextBubble(bool isUser) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        gradient: isUser ? AuraColors.userBubbleGradient : null,
        color: isUser ? null : AuraColors.surfaceContainerLowest,
        borderRadius: BorderRadius.only(
          topLeft: Radius.circular(isUser ? 16 : 0),
          topRight: Radius.circular(isUser ? 0 : 16),
          bottomLeft: const Radius.circular(16),
          bottomRight: const Radius.circular(16),
        ),
        boxShadow: [
          BoxShadow(
            color: isUser
                ? AuraColors.primary.withValues(alpha: 0.1)
                : const Color(0xFF00393A).withValues(alpha: 0.03),
            blurRadius: 12,
            offset: const Offset(0, 4),
          ),
        ],
      ),
      child: Text(
        message.content,
        style: GoogleFonts.manrope(
          fontSize: 15,
          height: 1.5,
          color: isUser ? Colors.white : AuraColors.onSurface,
        ),
      ),
    );
  }

  String _formatTime(String time) {
    try {
      final dt = DateTime.parse(time);
      return '${dt.hour.toString().padLeft(2, '0')}:${dt.minute.toString().padLeft(2, '0')}';
    } catch (_) {
      return '';
    }
  }
}
```

- [ ] **Step 3: 创建语音气泡**

创建 `app/business_packages/sanyan_chat/lib/src/chat/widget/voice_bubble.dart`：

```dart
import 'package:flutter/material.dart';
import 'package:audioplayers/audioplayers.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';
import '../../models/message.dart';

class VoiceBubble extends StatefulWidget {
  final Message message;
  final bool isUser;
  const VoiceBubble({super.key, required this.message, required this.isUser});

  @override
  State<VoiceBubble> createState() => _VoiceBubbleState();
}

class _VoiceBubbleState extends State<VoiceBubble> {
  final AudioPlayer _player = AudioPlayer();
  bool _isPlaying = false;

  @override
  void dispose() {
    _player.dispose();
    super.dispose();
  }

  void _togglePlay() async {
    if (widget.message.mediaUrl == null) return;
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
    final isUser = widget.isUser;
    final playColor = isUser ? Colors.white : AuraColors.primary;
    final barColor = isUser
        ? Colors.white.withValues(alpha: 0.4)
        : AuraColors.primary.withValues(alpha: 0.4);
    final barActiveColor = isUser ? Colors.white : AuraColors.primary;

    return GestureDetector(
      onTap: _togglePlay,
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 10),
        decoration: BoxDecoration(
          gradient: isUser ? AuraColors.mintAzureGradient : null,
          color: isUser ? null : AuraColors.surfaceContainerLowest,
          borderRadius: BorderRadius.only(
            topLeft: Radius.circular(isUser ? 24 : 0),
            topRight: Radius.circular(isUser ? 0 : 24),
            bottomLeft: const Radius.circular(24),
            bottomRight: const Radius.circular(24),
          ),
          boxShadow: [
            BoxShadow(
              color: AuraColors.primary.withValues(alpha: 0.1),
              blurRadius: 12,
              offset: const Offset(0, 4),
            ),
          ],
        ),
        child: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            // Play button
            Container(
              width: 40,
              height: 40,
              decoration: BoxDecoration(
                color: isUser
                    ? Colors.white.withValues(alpha: 0.9)
                    : AuraColors.surfaceContainerHighest,
                shape: BoxShape.circle,
              ),
              child: Icon(
                _isPlaying ? Icons.pause : Icons.play_arrow,
                color: isUser ? AuraColors.primary : AuraColors.primary,
                size: 24,
              ),
            ),
            const SizedBox(width: 10),
            // Waveform bars
            Row(
              children: List.generate(15, (i) {
                final heights = [8, 16, 12, 20, 24, 16, 12, 20, 8, 16, 12, 20, 16, 8, 16];
                return Container(
                  width: 2,
                  height: heights[i].toDouble(),
                  margin: const EdgeInsets.symmetric(horizontal: 1.5),
                  decoration: BoxDecoration(
                    color: _isPlaying ? barActiveColor : barColor,
                    borderRadius: BorderRadius.circular(1),
                  ),
                );
              }),
            ),
            const SizedBox(width: 10),
            // Duration
            Text(
              '0:12',
              style: GoogleFonts.inter(
                fontSize: 11,
                fontWeight: FontWeight.w600,
                color: isUser
                    ? AuraColors.onSurface.withValues(alpha: 0.7)
                    : AuraColors.onSurfaceVariant,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: 创建视频气泡（UI 壳子）**

创建 `app/business_packages/sanyan_chat/lib/src/chat/widget/video_bubble.dart`：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';

class VideoBubble extends StatelessWidget {
  final bool isUser;
  final String? thumbnailUrl;

  const VideoBubble({super.key, required this.isUser, this.thumbnailUrl});

  @override
  Widget build(BuildContext context) {
    return ClipRRect(
      borderRadius: BorderRadius.only(
        topLeft: Radius.circular(isUser ? 32 : 0),
        topRight: Radius.circular(isUser ? 0 : 32),
        bottomLeft: const Radius.circular(32),
        bottomRight: const Radius.circular(32),
      ),
      child: SizedBox(
        width: 220,
        height: 275,
        child: Stack(
          fit: StackFit.expand,
          children: [
            // Thumbnail placeholder
            Container(color: AuraColors.surfaceContainerHigh),
            // Glass play button
            Center(
              child: ClipOval(
                child: BackdropFilter(
                  filter: ImageFilter.blur(sigmaX: 12, sigmaY: 12),
                  child: Container(
                    width: 64,
                    height: 64,
                    decoration: BoxDecoration(
                      color: Colors.white.withValues(alpha: 0.2),
                      shape: BoxShape.circle,
                      border: Border.all(
                        color: Colors.white.withValues(alpha: 0.4),
                      ),
                    ),
                    child: const Icon(
                      Icons.play_arrow,
                      color: Colors.white,
                      size: 36,
                    ),
                  ),
                ),
              ),
            ),
            // Progress bar
            Positioned(
              bottom: 0,
              left: 0,
              right: 0,
              child: Container(
                height: 4,
                color: Colors.white.withValues(alpha: 0.2),
                child: FractionallySizedBox(
                  alignment: Alignment.centerLeft,
                  widthFactor: 0.33,
                  child: Container(
                    decoration: BoxDecoration(
                      color: AuraColors.primaryFixed,
                      boxShadow: [
                        BoxShadow(
                          color: AuraColors.primaryFixed.withValues(alpha: 0.8),
                          blurRadius: 8,
                        ),
                      ],
                    ),
                  ),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 5: 重写输入栏**

完全替换 `app/business_packages/sanyan_chat/lib/src/chat/widget/chat_input_bar.dart`：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';
import 'package:sanyan_common_ui/sanyan_common_ui.dart';

class ChatInputBar extends StatelessWidget {
  final TextEditingController controller;
  final VoidCallback onSend;
  const ChatInputBar({super.key, required this.controller, required this.onSend});

  @override
  Widget build(BuildContext context) {
    return ClipRect(
      child: BackdropFilter(
        filter: ImageFilter.blur(sigmaX: 24, sigmaY: 24),
        child: Container(
          color: AuraColors.surface.withValues(alpha: 0.8),
          padding: const EdgeInsets.fromLTRB(16, 12, 16, 12),
          child: SafeArea(
            top: false,
            child: Container(
              decoration: BoxDecoration(
                color: AuraColors.surfaceContainerLowest.withValues(alpha: 0.8),
                borderRadius: BorderRadius.circular(9999),
                border: Border.all(
                  color: AuraColors.outlineVariant.withValues(alpha: 0.1),
                ),
                boxShadow: [
                  BoxShadow(
                    color: AuraColors.onSurface.withValues(alpha: 0.05),
                    blurRadius: 20,
                  ),
                ],
              ),
              padding: const EdgeInsets.symmetric(horizontal: 4, vertical: 4),
              child: Row(
                children: [
                  // Add button
                  IconButton(
                    onPressed: () {},
                    icon: const Icon(Icons.add_circle, color: AuraColors.primary),
                  ),
                  // Input
                  Expanded(
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
                          color: AuraColors.onSurfaceVariant.withValues(alpha: 0.5),
                        ),
                        border: InputBorder.none,
                        contentPadding: const EdgeInsets.symmetric(horizontal: 4),
                        isDense: true,
                      ),
                      textInputAction: TextInputAction.send,
                      onSubmitted: (_) => onSend(),
                    ),
                  ),
                  // Emoji
                  IconButton(
                    onPressed: () {},
                    icon: const Icon(
                      Icons.sentiment_satisfied,
                      color: AuraColors.onSurfaceVariant,
                    ),
                  ),
                  // Send
                  GestureDetector(
                    onTap: onSend,
                    child: Container(
                      width: 44,
                      height: 44,
                      decoration: BoxDecoration(
                        gradient: AuraColors.mintAzureGradient,
                        shape: BoxShape.circle,
                        boxShadow: [
                          BoxShadow(
                            color: AuraColors.primaryFixed.withValues(alpha: 0.3),
                            blurRadius: 8,
                          ),
                        ],
                      ),
                      child: const Icon(Icons.send, color: Color(0xFF00393A), size: 20),
                    ),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

- [ ] **Step 6: 验证编译 + 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
/Users/aventador/fvm/versions/3.41.6/bin/flutter analyze
git add -A
git commit -m "feat: 聊天页 + 消息气泡 + 语音气泡 + 视频气泡 + 输入栏 Ethereal 重写"
```

---

### Task 7: 清理 + 全量测试 + 删除旧兼容代码

**Files:**
- Modify: `app/foundation_packages/sanyan_common_ui/lib/src/aura_colors.dart` (删除兼容别名)
- Modify: `app/integration_test/app_test.dart` (适配新 UI)

- [ ] **Step 1: 删除 AuraColors 兼容别名**

从 `aura_colors.dart` 中删除 Step 7 添加的所有 `// 兼容旧代码` 注释下的别名字段。

- [ ] **Step 2: 全局搜索确认无残留引用**

Run: `cd /Users/aventador/sourceCode/3yan/app && grep -r "AppColors\|AppTheme\|brandGradient\|brandButton\|brandStart\|brandEnd\|textPrimary\|textSecondary\|textPlaceholder\|inputFill\|aiBubble" --include="*.dart" lib/ business_packages/ foundation_packages/ | grep -v "aura_colors.dart"`

Expected: 无输出（所有旧引用已被替换）

- [ ] **Step 3: 更新 E2E 测试**

根据新 UI 元素更新 `app/integration_test/app_test.dart` 中的 finder（登录按钮文案、页面标识等）。

- [ ] **Step 4: 验证编译 + 运行测试**

```bash
cd /Users/aventador/sourceCode/3yan/app
/Users/aventador/fvm/versions/3.41.6/bin/flutter analyze
/Users/aventador/fvm/versions/3.41.6/bin/flutter test
```

- [ ] **Step 5: 提交**

```bash
cd /Users/aventador/sourceCode/3yan/app
git add -A
git commit -m "chore: 清理旧主题兼容代码 + 更新 E2E 测试"
```
