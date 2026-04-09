# 三言 App UI 全面改版 — Stitch Ethereal Editorial 风格

## 概述

按照 Stitch 设计稿（`stitch/` 目录）1:1 还原三言 App 的 Flutter UI。从暖阳橘主题切换为 Ethereal Editorial 风格：薄荷绿色系、毛玻璃效果、Manrope 字体、无边框分割、渐变交互元素。

## 设计来源

| 页面 | Stitch 截图 | Stitch 代码 |
|------|-------------|-------------|
| 登录 | stitch/login_screen_with_logo/screen.png | stitch/login_screen_with_logo/code.html |
| 注册 | stitch/signup_screen_with_logo/screen.png | stitch/signup_screen_with_logo/code.html |
| 消息列表 | stitch/messages/screen.png | stitch/messages/code.html |
| 聊天 | stitch/chat_detail/screen.png | stitch/chat_detail/code.html |
| 设置 | stitch/settings/screen.png | stitch/settings/code.html |
| 联系人 | stitch/contacts/screen.png | stitch/contacts/code.html |
| 品牌规范 | stitch/brand_identity/screen.png | stitch/aura_mint/DESIGN.md |

## 一、设计系统

### 1.1 色板

```dart
// 品牌色
primary       = #006B64   // 品牌主色/导航文字/图标
primaryFixed  = #73F1E4   // 薄荷渐变起点/选中态背景
primaryDim    = #005E57   // 深色变体
secondary     = #006595   // 蓝色渐变终点
secondaryFixedDim = #AED9FF // 天蓝渐变终点

// 表面层级（从深到浅）
surface                = #E2FFFF  // 全局背景
surfaceDim             = #A1E6E7  // 暗表面
surfaceContainerHighest = #B0EEEE // 输入框默认背景
surfaceContainerHigh   = #BCF2F3  // 高层容器
surfaceContainer       = #C8F7F7  // 标准容器
surfaceContainerLow    = #D4FBFB  // 低层容器/hover 态
surfaceContainerLowest = #FFFFFF  // 卡片/白色元素

// 文字
onSurface        = #00393A  // 主文字
onSurfaceVariant = #2A6869  // 次要文字/标签
outlineVariant   = #80BCBD  // Ghost 边框
outline          = #488485  // 边框

// 其他
tertiary          = #6E4DA7  // 紫色强调
tertiaryContainer = #C9AAFF  // 紫色容器
error             = #AC3434  // 错误
```

### 1.2 渐变

```dart
// 主按钮/发送按钮/选中态
mintAzureGradient = LinearGradient(#73F1E4 → #AED9FF)

// 用户聊天气泡
userBubbleGradient = LinearGradient(#006B64 → #006595)  // primary → secondary

// 登录页背景
loginBgGradient = RadialGradient(center: topLeft, #73F1E4 0%, #E2FFFF 40%, #AED9FF 100%)
```

### 1.3 毛玻璃效果

```dart
// 登录卡片、底部导航、顶栏
glassPanel:
  color: Colors.white.withOpacity(0.4)
  backdropFilter: ImageFilter.blur(sigmaX: 24, sigmaY: 24)
  border: Border.all(color: Colors.white.withOpacity(0.2))
  borderRadius: 16
  shadow: BoxShadow(color: Color(0x0F00393A), blurRadius: 40, offset: Offset(0, 20))
```

### 1.4 字体

```dart
// 标题 — Manrope
headline: fontFamily: 'Manrope', fontWeight: w700/w800
  - App 标题: 24px, extraBold
  - 页面标题: 20px, bold
  - 聊天名称: 18px, bold
  - 会话列表名称: 16px, bold

// 正文 — Manrope
body: fontFamily: 'Manrope', fontWeight: w400/w600
  - 消息内容: 15px, regular
  - 会话预览: 14px, regular
  - 副标题: 14px, medium

// 标签 — Inter
label: fontFamily: 'Inter', fontWeight: w500/w600
  - 输入框标签: 10px, bold, uppercase, letterSpacing: 1.5
  - 时间戳: 10px, medium
  - 底部导航: 10px, semibold, uppercase
```

### 1.5 圆角

```dart
cardRadius    = 12  // 卡片/输入框
buttonRadius  = 9999 // 按钮（full）
bubbleRadius  = 16   // 消息气泡
avatarRadius  = 9999 // 头像（圆形）
navBarRadius  = 24   // 底部导航顶部圆角
```

### 1.6 核心规则

- **无边框分割**：不用 Divider，靠背景色层级和间距区分区域
- **无纯黑文字**：主文字用 #00393A，不用 #000000
- **阴影用品牌色**：BoxShadow 颜色基于 #00393A 低透明度，不用纯黑
- **渐变优于纯色**：按钮、选中态、用户气泡都用渐变

## 二、页面设计

### 2.1 登录页

**布局**：全屏径向渐变背景 + 居中毛玻璃卡片

**结构**：
```
[渐变背景]
  [毛玻璃卡片, 圆角16, padding 32]
    [Logo 图片, 96px]
    [副标题: "走进更温暖的连接方式", Manrope medium, #2A6869]
    [间距 48]
    [标签 "手机号", Inter 10px, bold, uppercase, #2A6869]
    [输入框: #B0EEEE/50%, 圆角12, 高56, 左侧图标]
    [间距 24]
    [标签 "密码", Inter 10px, bold, uppercase, #2A6869]
    [输入框: 同上, type=password, 左侧锁图标]
    [间距 32]
    [登录按钮: 薄荷→天蓝渐变, 圆角full, 高56, Manrope bold, 带右箭头]
    [间距 40]
    [分隔线: #80BCBD/10%]
    [忘记密码? #2A6869]
    [没有账号？立即注册, "立即注册" 用 #006B64 bold]
```

### 2.2 注册页

**布局**：同登录页（渐变背景 + 毛玻璃卡片）

**结构**：
```
[渐变背景]
  [毛玻璃卡片]
    [Logo 图片, 64px]
    [标题: "加入三言", Manrope 28px extraBold, #00393A]
    [副标题: "填写信息创建你的账号", #2A6869]
    [间距 32]
    [昵称输入框]
    [手机号输入框]
    [验证码输入框 + 获取验证码按钮]
    [密码输入框]
    [间距 24]
    [注册按钮: 薄荷→天蓝渐变, 全圆角]
    [间距 16]
    [OR CONTINUE WITH 分割线]
    [第三方登录图标行]
    [已有账号？返回登录]
```

### 2.3 消息列表页（主页）

**布局**：surface 背景 + 顶栏 + 滚动内容 + FAB + 底部导航

**顶栏**（毛玻璃 sticky）：
```
[毛玻璃背景, #E2FFFF/60% + blur]
  [头像 40px] [三言, Manrope extraBold 24px, 渐变文字 primary→secondary] [搜索图标]
[分割线: #D4FBFB, 1px]
```

**内容区**：
```
[搜索框: #B0EEEE 背景, 圆角12, 高56, 搜索图标]
[间距 40]
[Recent Activity 标题, Manrope bold 20px]
[横向滚动头像列表: 64px 圆形, 渐变边框]
[间距 48]
[Messages 标题, Manrope bold 20px]
[会话卡片列表]:
  未读: 白色底(#FFF) + 阴影 + 右侧渐变未读数
  已读: 无背景
  每项: [头像56px + 在线点] [名称 bold + 时间 10px] [预览文字 14px]
```

**FAB**：右下角，primary→secondary 渐变圆形，编辑图标

**底部导航**：
```
[毛玻璃背景, 顶部圆角24]
  [Messages(选中): 渐变背景圆角pill, 填充图标]
  [Contacts: #2A6869 文字/图标]
  [Status: #2A6869]
  [Settings: #2A6869]
  标签: Manrope 10px, semibold, uppercase
```

### 2.4 聊天页

**顶栏**（毛玻璃）：
```
[返回箭头 primary] [头像 40px + 渐变边框] [名称 Manrope bold 18px + "Online" 状态] [搜索 + 更多]
```

**消息区**：
```
[日期指示器: 居中, #B0EEEE/40% pill, Inter 11px]

AI 消息:
  [白色底 #FFFFFF, 圆角16px, 左上角方角(tl=0), padding 16]
  [文字 15px Manrope, #00393A]
  [时间戳在下方左侧, 10px Inter, #2A6869/70%]
  [阴影: 0 4px 12px rgba(0,57,58,0.03)]

用户消息:
  [primary→secondary 渐变, 圆角16px, 右上角方角(tr=0)]
  [文字 15px, 白色]
  [时间戳在下方右侧 + 已读图标]

正在输入: 三个弹跳圆点(primary/40%) + "正在输入" Inter 10px
```

**输入栏**（毛玻璃固定底部）：
```
[#E2FFFF/80% + blur]
  [圆角full 容器: 白色/80% + ghost 边框]
    [➕ 按钮 primary] [输入框 透明背景] [表情按钮] [发送按钮: 薄荷→天蓝渐变圆形]
```

### 2.5 联系人页

```
[顶栏同消息列表]
[标题: "Contacts", Manrope extraBold 28px]
[副标题: "xx people in your orbit"]
[搜索框]
[横向 Recent 头像]
[按字母分组列表]:
  字母标题: Manrope bold, #2A6869
  每项: [头像 48px] [名称 bold] [状态描述 14px #2A6869/70%]
```

### 2.6 设置页

```
[顶栏同消息列表]
[用户卡片: 头像 + 名称 + 邮箱]
[分组]:
  Account:
    - Personal Information (描述文字)
    - Password & Security
  Appearance:
    - Dark Mode (开关)
    - Chat Wallpaper
  Notifications
  Privacy:
    - Last Seen & Online
    - Blocked Contacts
  Data:
    - Messages (描述)
    - Terms of Service
[Logout 按钮: outline 样式, primary 色]
```

## 三、需要改动的文件

### 新增
- `app/foundation_packages/sanyan_common_ui/lib/src/aura_theme.dart` — 新设计系统
- `app/foundation_packages/sanyan_common_ui/lib/src/widgets/glass_panel.dart` — 毛玻璃组件
- `app/foundation_packages/sanyan_common_ui/lib/src/widgets/aura_input.dart` — 输入框组件
- `app/foundation_packages/sanyan_common_ui/lib/src/widgets/aura_button.dart` — 渐变按钮
- `app/foundation_packages/sanyan_common_ui/lib/src/widgets/aura_nav_bar.dart` — 底部导航
- `app/business_packages/sanyan_chat/lib/src/contacts/contacts_page.dart` — 联系人页
- `app/business_packages/sanyan_chat/lib/src/settings/settings_page.dart` — 设置页
- Logo 图片资源 — 从 Stitch 截图中裁切或重绘

### 修改
- `app/foundation_packages/sanyan_common_ui/lib/sanyan_common_ui.dart` — 导出新文件
- `app/lib/main.dart` — 切换到 AuraTheme + 添加新路由
- `app/foundation_packages/sanyan_user/lib/src/auth/login_page.dart` — 完全重写
- `app/foundation_packages/sanyan_user/lib/src/auth/register_page.dart` — 完全重写（如果在 sanyan_auth 包里则改那个）
- `app/business_packages/sanyan_chat/lib/src/home/home_page.dart` — 完全重写
- `app/business_packages/sanyan_chat/lib/src/chat/chat_page.dart` — 完全重写
- `app/business_packages/sanyan_chat/lib/src/chat/widget/message_bubble.dart` — 完全重写
- `app/business_packages/sanyan_chat/lib/src/chat/widget/chat_input_bar.dart` — 完全重写
- `app/pubspec.yaml` — 添加 google_fonts 依赖（Manrope）

### 删除
- `app/foundation_packages/sanyan_common_ui/lib/src/theme.dart` — 旧暖阳橘主题（被 aura_theme.dart 替代）

## 四、字体依赖

使用 `google_fonts` 包加载 Manrope 和 Inter，不需要本地字体文件。

## 五、Logo 处理

从 Stitch 的 login_screen_with_logo 截图中可以看到 logo 是一个薄荷绿旋转花瓣图标。方案：
1. 从 Stitch 截图中裁切 logo 图片作为临时资源
2. 后续用矢量图（SVG）替换

## 六、不在本次范围内

- Contacts 和 Settings 的业务逻辑（只做 UI 壳子）
- Dark Mode 实现（Settings 里有开关但不实现）
- Status 页面内容（占位页面）
- 第三方登录功能
- 聊天页发图片功能
