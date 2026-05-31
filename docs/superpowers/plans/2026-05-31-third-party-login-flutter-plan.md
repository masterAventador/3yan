# 第三方登录 Flutter 客户端 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在三言 Flutter app（GetX 架构）里接入 Apple / 微信第三方登录，对接已就绪的服务端契约（`/oauth/challenge`、`/oauth/login`、`/oauth/bind-phone`），含强制绑手机号与账号合并安全。

**Architecture:** 分层对接——`sanyan_network`/`sanyan_user`（foundation）放 API 契约（req/data 模型、AuthApi 方法、ApiResponse 修复）；`sanyan_auth`（business）放 SDK 抽象、编排服务、控制器、登录页/绑手机页 UI。所有 SDK 调用、网关、登录成功副作用都封在**可注入接口**后面，纯逻辑（sha256、响应解析、结果映射）抽成纯函数，使控制器分支逻辑可用 fake 单测；原生配置（iOS entitlements / Info.plist / Android WXEntryActivity）作为凭证门控的配置任务收尾。

**Tech Stack:** Flutter 3.41.6（fvm）、GetX 4.6、Dio（sanyan_network ApiClient 单例）、GetStorage（LocalStorage）、`sign_in_with_apple`、`fluwx`（4.x）、`crypto`（sha256）、`mocktail`（测试 mock）。

**服务端契约（已实现，本计划对接，勿改服务端）：**
- `POST /api/auth/oauth/challenge` 入参 `{}` → 返回 `{nonce}`（一次性，Apple 用，S8 防重放）。
- `POST /api/auth/oauth/login` 入参 `{provider, credential, nonce?}`，`provider` 取大写枚举 `"APPLE"`/`"WECHAT"`；Apple：`credential`=identityToken、`nonce`=challenge 拿到的**原始 nonce**（服务端比对 token 的 nonce claim == sha256(原始nonce)）；微信：`credential`=code、`nonce` 可空。返回：命中 `{userId, token, nickname?, avatar?}`；未绑 `{needBind:true, bindTicket}`。
- `POST /api/auth/oauth/bind-phone` 入参 `{bindTicket, phone, code, password?}` → 成功 `{userId, token}`；已有账号需本人证明 `{needMergeAuth:true}`（success=true 数据态，**该 bindTicket 已失效，不可用同 ticket 重试**，见 spec §6）；传错密码 → success=false / code `1014`。
- 错误码：`BIND_TICKET_USED=1013`、`NEED_MERGE_AUTH=1014`、`OAUTH_VERIFY_FAILED=1010`、`SMS_CODE_INVALID` 等（见服务端 `ERROR_CODE_REGISTRY.md`）。

**关键既有惯例（实现时务必遵循；现有代码有违规处，按规则的正确写法写新代码）：**
- req 模型：`foundation_packages/sanyan_user/lib/src/api/req/`，手写 `extends BaseReq` + `path`/`method`/`toJson()`，**不用 freezed/json_serializable**。
- AuthApi：`abstract class AuthApi`（`auth_api.dart`）+ 全 static 方法 + `static final _client = ApiClient();`，调 `_client.send(Req(), fromData: (d) => ...)`。
- ApiClient（`sanyan_network`）单例、**永不抛异常**（HTTP 错误兜成 `ApiResponse(success:false, ...)`），拦截器自动注入 `Bearer ${LocalStorage.token}`。
- 登录态：`LocalStorage`（GetStorage）的 static `token`/`userId`；写进去即对后续请求/WS 生效。
- 路由：常量集中在 `foundation_packages/sanyan_routes/lib/src/routes.dart` 的 `AppRoutes`；注册集中在 `app/lib/main.dart` 的 `GetMaterialApp.getPages`。
- 登录成功五步链（**现内联且在 LoginController 与 RegisterController 重复**）：写 `LocalStorage.token` → 写 `LocalStorage.userId` → `AuthApi.registerPushTokenAfterLogin()` → `Get.find<WsClient>().connect()` → `Get.offAllNamed(AppRoutes.chat)`。本计划 Task 8 抽成可复用的 `LoginSuccessHandler`。
- 验证码倒计时：参考 `business_packages/sanyan_auth/lib/src/auth/register_controller.dart` 的 `_startCountdown()`（60s `Timer.periodic`）+ `register_page.dart` 的 `Obx` 按钮。
- UI 设计系统：`sanyan_common_ui` 的 `GlassPanel`/`AuraInput`/`AuraButton`/`AuraColors`。
- 测试：`fvm flutter test`（仓库根 `.fvmrc` → 3.41.6）；包级 suite 聚合（如 `sanyan_user/test/sanyan_user_suite.dart`）；新增测试登记进对应 suite。req 测试模板见 `sanyan_user/test/api/register_device_token_req_test.dart`，纯类测试模板见 `sanyan_common_ui/test/widgets/slide_drawer_controller_test.dart`。
- **新代码遵循规则正确写法**：禁止在 `build` 里 `Get.put`（用 `Bindings` 或 controller 在页面外注入）；GetX controller 属性引用名用 `c`；工具类用 `abstract class` + static。

**测试 mock 选型：** 用 **mocktail**（无 codegen，符合项目规则）。能用手写 fake（`class FakeXxx implements Xxx`）的优先手写 fake，只在需要"验证调用次数/参数"时用 mocktail 的 `Mock` + `verify`。

---

## 文件结构总览

**新建：**
- `foundation_packages/sanyan_user/lib/src/api/data/oauth_login_data.dart` — `OauthLoginData` 响应模型（login + bind-phone 共用）
- `foundation_packages/sanyan_user/lib/src/api/req/oauth_challenge_req.dart`
- `foundation_packages/sanyan_user/lib/src/api/req/oauth_login_req.dart`
- `foundation_packages/sanyan_user/lib/src/api/req/oauth_bind_phone_req.dart`
- `business_packages/sanyan_auth/lib/src/oauth/oauth_crypto.dart` — sha256Hex 纯函数
- `business_packages/sanyan_auth/lib/src/oauth/sdk_auth_result.dart` — sealed 结果类型
- `business_packages/sanyan_auth/lib/src/oauth/apple_auth_provider.dart` — 抽象 + 真实现
- `business_packages/sanyan_auth/lib/src/oauth/wechat_auth_provider.dart` — 抽象 + 真实现
- `business_packages/sanyan_auth/lib/src/oauth/oauth_gateway.dart` — 抽象 + 真实现（包 AuthApi static）
- `business_packages/sanyan_auth/lib/src/auth/login_success_handler.dart` — 抽象 + 真实现
- `business_packages/sanyan_auth/lib/src/oauth/oauth_outcome.dart` — 编排结果类型 + 纯映射
- `business_packages/sanyan_auth/lib/src/oauth/oauth_login_controller.dart`
- `business_packages/sanyan_auth/lib/src/oauth/bind_phone_controller.dart`
- `business_packages/sanyan_auth/lib/src/oauth/bind_phone_page.dart`
- iOS：`app/ios/Runner/Runner.entitlements`；Android：`app/android/app/src/main/kotlin/.../wxapi/WXEntryActivity.kt`
- 对应 `test/` 测试文件（每 Task 列出）

**修改：**
- `foundation_packages/sanyan_network/lib/src/api_response.dart` — 加 `code`/`message`
- `foundation_packages/sanyan_user/lib/src/api/auth_api.dart` — 加 3 个 oauth 方法
- `foundation_packages/sanyan_user/lib/sanyan_user.dart` — export 新增 data 模型
- `business_packages/sanyan_auth/lib/sanyan_auth.dart` — barrel export 新增公开类
- `business_packages/sanyan_auth/lib/src/auth/login_controller.dart` + `register_controller.dart` — 改用 LoginSuccessHandler
- `business_packages/sanyan_auth/lib/src/auth/login_page.dart` — 加第三方登录按钮区
- `business_packages/sanyan_auth/pubspec.yaml` — 加 SDK/crypto/mocktail 依赖
- `foundation_packages/sanyan_routes/lib/src/routes.dart` — 加 `bindPhone` 常量
- `app/lib/main.dart` — getPages 注册 bindPhone + Get.put 注入 oauth 依赖 + fluwx registerApi
- iOS `app/ios/Runner/Info.plist`、`AppDelegate.swift`、`project.pbxproj`(entitlements 引用)；Android `AndroidManifest.xml`

---

## Phase A — API 契约层（sanyan_network / sanyan_user，纯逻辑强 TDD）

### Task 1: 修复 ApiResponse 暴露 code/message

**背景：** 现有 `ApiResponse.fromJson` 读 `json['errMsg']`，但服务端 `BaseResp` 发的是 `message`/`code` → `errMsg` 恒为 null，业务错误文案全丢。修复使 needMergeAuth/错误码场景能展示文案，且修好所有现存 `Get.snackbar(resp.errMsg ?? ...)`。

**Files:**
- Modify: `foundation_packages/sanyan_network/lib/src/api_response.dart`
- Test: `foundation_packages/sanyan_network/test/api_response_test.dart`（新建）
- Modify: `foundation_packages/sanyan_network/test/sanyan_network_suite.dart`（若存在 suite，登记新测试；不存在则跳过）

- [ ] **Step 1: 先 Read 现有 api_response.dart**，确认现字段（`success`/`errMsg`/`data`）与 `fromJson` 实现，再改。

- [ ] **Step 2: 写失败测试**

`foundation_packages/sanyan_network/test/api_response_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_network/sanyan_network.dart';

void main() {
  group('ApiResponse.fromJson', () {
    test('reads code and message from server BaseResp', () {
      final r = ApiResponse.fromJson(
        {'success': false, 'code': 1014, 'message': '该手机号已注册', 'data': null},
        (d) => d as Map<String, dynamic>?,
      );
      expect(r.success, isFalse);
      expect(r.code, 1014);
      expect(r.message, '该手机号已注册');
    });

    test('errMsg falls back to message for backward compatibility', () {
      final r = ApiResponse.fromJson(
        {'success': false, 'message': '验证码错误', 'data': null},
        (d) => d as Map<String, dynamic>?,
      );
      expect(r.errMsg, '验证码错误');
    });

    test('success response carries data and null code/message', () {
      final r = ApiResponse.fromJson(
        {'success': true, 'data': {'token': 't', 'userId': 1}},
        (d) => d as Map<String, dynamic>?,
      );
      expect(r.success, isTrue);
      expect(r.data, {'token': 't', 'userId': 1});
      expect(r.code, isNull);
    });
  });
}
```

- [ ] **Step 3: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test foundation_packages/sanyan_network/test/api_response_test.dart`
Expected: FAIL（`code`/`message` getter 不存在，编译错误）

- [ ] **Step 4: 改 ApiResponse**

在 `ApiResponse<T>` 加 `final int? code;` 与 `final String? message;` 字段，构造函数加可选命名参数。`errMsg` 改为：保留字段但 `fromJson` 里 `errMsg = json['errMsg'] ?? json['message']`（兼容旧字段，回退到 message）。`fromJson` 增加 `code: json['code'] as int?`、`message: json['message'] as String?`。保持 HTTP 兜底构造（`ApiResponse(success:false, errMsg: friendly)`）可用——给 `code`/`message` 默认 null。

- [ ] **Step 5: 运行确认通过**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test foundation_packages/sanyan_network/test/api_response_test.dart`
Expected: PASS (3 tests)

- [ ] **Step 6: 回归 sanyan_network 包测试**（确认没破坏现有 WsEvent 等测试）

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test foundation_packages/sanyan_network`
Expected: 全 PASS

- [ ] **Step 7: Commit**

```bash
cd /Users/aventador/code/3yan/app
git add foundation_packages/sanyan_network/lib/src/api_response.dart foundation_packages/sanyan_network/test/api_response_test.dart
git commit -m "fix(network): ApiResponse 暴露后端 code/message，errMsg 回退 message（修业务错误文案丢失）"
```

---

### Task 2: OauthLoginData 响应模型

**背景：** login 与 bind-phone 共用一个响应数据模型，承载命中登录（token/userId）、未绑（needBind/bindTicket）、需合并（needMergeAuth）三态。

**Files:**
- Create: `foundation_packages/sanyan_user/lib/src/api/data/oauth_login_data.dart`
- Modify: `foundation_packages/sanyan_user/lib/sanyan_user.dart`（export）
- Test: `foundation_packages/sanyan_user/test/api/oauth_login_data_test.dart`
- Modify: `foundation_packages/sanyan_user/test/sanyan_user_suite.dart`（登记）

- [ ] **Step 1: 写失败测试**

`foundation_packages/sanyan_user/test/api/oauth_login_data_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_user/sanyan_user.dart';

void main() {
  group('OauthLoginData.fromJson', () {
    test('logged-in: token + userId, not needBind/needMerge', () {
      final d = OauthLoginData.fromJson({'token': 'jwt', 'userId': 42, 'nickname': '小婉'});
      expect(d.token, 'jwt');
      expect(d.userId, 42);
      expect(d.needBind, isFalse);
      expect(d.needMergeAuth, isFalse);
      expect(d.bindTicket, isNull);
      expect(d.nickname, '小婉');
    });

    test('needBind: carries bindTicket, no token', () {
      final d = OauthLoginData.fromJson({'needBind': true, 'bindTicket': 'bt-123'});
      expect(d.needBind, isTrue);
      expect(d.bindTicket, 'bt-123');
      expect(d.token, isNull);
    });

    test('needMergeAuth: flag true, no token', () {
      final d = OauthLoginData.fromJson({'needMergeAuth': true});
      expect(d.needMergeAuth, isTrue);
      expect(d.token, isNull);
    });

    test('missing flags default to false', () {
      final d = OauthLoginData.fromJson({'token': 't', 'userId': 1});
      expect(d.needBind, isFalse);
      expect(d.needMergeAuth, isFalse);
    });
  });
}
```

- [ ] **Step 2: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test foundation_packages/sanyan_user/test/api/oauth_login_data_test.dart`
Expected: FAIL（类不存在）

- [ ] **Step 3: 实现 OauthLoginData**

`foundation_packages/sanyan_user/lib/src/api/data/oauth_login_data.dart`:
```dart
/// 第三方登录 / 绑手机 共用响应模型。
/// 三态互斥：命中登录(token+userId) | 未绑(needBind+bindTicket) | 需合并(needMergeAuth)。
class OauthLoginData {
  final String? token;
  final int? userId;
  final bool needBind;
  final String? bindTicket;
  final bool needMergeAuth;
  final String? nickname;
  final String? avatar;

  const OauthLoginData({
    this.token,
    this.userId,
    this.needBind = false,
    this.bindTicket,
    this.needMergeAuth = false,
    this.nickname,
    this.avatar,
  });

  factory OauthLoginData.fromJson(Map<String, dynamic> json) => OauthLoginData(
        token: json['token'] as String?,
        userId: json['userId'] as int?,
        needBind: json['needBind'] as bool? ?? false,
        bindTicket: json['bindTicket'] as String?,
        needMergeAuth: json['needMergeAuth'] as bool? ?? false,
        nickname: json['nickname'] as String?,
        avatar: json['avatar'] as String?,
      );

  bool get loggedIn => token != null && userId != null;
}
```

- [ ] **Step 4: export**

在 `foundation_packages/sanyan_user/lib/sanyan_user.dart` 加：
```dart
export 'src/api/data/oauth_login_data.dart';
```

- [ ] **Step 5: 登记 suite**

在 `foundation_packages/sanyan_user/test/sanyan_user_suite.dart` 仿现有写法加：
```dart
import 'api/oauth_login_data_test.dart' as oauth_login_data_test;
// main() 内：
oauth_login_data_test.main();
```

- [ ] **Step 6: 运行确认通过**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test foundation_packages/sanyan_user/test/api/oauth_login_data_test.dart`
Expected: PASS (4 tests)

- [ ] **Step 7: Commit**

```bash
cd /Users/aventador/code/3yan/app
git add foundation_packages/sanyan_user/lib/src/api/data/oauth_login_data.dart foundation_packages/sanyan_user/lib/sanyan_user.dart foundation_packages/sanyan_user/test/api/oauth_login_data_test.dart foundation_packages/sanyan_user/test/sanyan_user_suite.dart
git commit -m "feat(user): OauthLoginData 响应模型（命中/未绑/需合并三态）"
```

---

### Task 3: Oauth req 模型 + AuthApi 方法

**Files:**
- Create: `foundation_packages/sanyan_user/lib/src/api/req/oauth_challenge_req.dart`
- Create: `foundation_packages/sanyan_user/lib/src/api/req/oauth_login_req.dart`
- Create: `foundation_packages/sanyan_user/lib/src/api/req/oauth_bind_phone_req.dart`
- Modify: `foundation_packages/sanyan_user/lib/src/api/auth_api.dart`
- Test: `foundation_packages/sanyan_user/test/api/oauth_req_test.dart`
- Modify: `foundation_packages/sanyan_user/test/sanyan_user_suite.dart`

- [ ] **Step 1: 先 Read** `register_device_token_req.dart` 与 `auth_api.dart`，确认 `BaseReq` 接口与 `_client.send` 签名。

- [ ] **Step 2: 写失败测试**

`foundation_packages/sanyan_user/test/api/oauth_req_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_user/src/api/req/oauth_challenge_req.dart';
import 'package:sanyan_user/src/api/req/oauth_login_req.dart';
import 'package:sanyan_user/src/api/req/oauth_bind_phone_req.dart';

void main() {
  test('OauthChallengeReq path/method/body', () {
    final r = OauthChallengeReq();
    expect(r.path, '/api/auth/oauth/challenge');
    expect(r.method, 'POST');
    expect(r.toJson(), {});
  });

  test('OauthLoginReq carries provider/credential/nonce', () {
    final r = OauthLoginReq(provider: 'APPLE', credential: 'idtoken', nonce: 'raw-nonce');
    expect(r.path, '/api/auth/oauth/login');
    expect(r.method, 'POST');
    expect(r.toJson(), {'provider': 'APPLE', 'credential': 'idtoken', 'nonce': 'raw-nonce'});
  });

  test('OauthLoginReq omits null nonce (wechat)', () {
    final r = OauthLoginReq(provider: 'WECHAT', credential: 'code', nonce: null);
    expect(r.toJson(), {'provider': 'WECHAT', 'credential': 'code'});
  });

  test('OauthBindPhoneReq carries ticket/phone/code, optional password', () {
    final r = OauthBindPhoneReq(bindTicket: 'bt', phone: '13800000000', code: '1234', password: 'pw');
    expect(r.path, '/api/auth/oauth/bind-phone');
    expect(r.method, 'POST');
    expect(r.toJson(), {'bindTicket': 'bt', 'phone': '13800000000', 'code': '1234', 'password': 'pw'});
  });

  test('OauthBindPhoneReq omits null password', () {
    final r = OauthBindPhoneReq(bindTicket: 'bt', phone: '13800000000', code: '1234', password: null);
    expect(r.toJson(), {'bindTicket': 'bt', 'phone': '13800000000', 'code': '1234'});
  });
}
```

- [ ] **Step 3: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test foundation_packages/sanyan_user/test/api/oauth_req_test.dart`
Expected: FAIL（类不存在）

- [ ] **Step 4: 实现 3 个 req（沿用 BaseReq 模式，省略 null 字段）**

`oauth_challenge_req.dart`:
```dart
import 'package:sanyan_network/sanyan_network.dart';

class OauthChallengeReq extends BaseReq {
  @override
  String get path => '/api/auth/oauth/challenge';
  @override
  String get method => 'POST';
  @override
  Map<String, dynamic> toJson() => {};
}
```

`oauth_login_req.dart`:
```dart
import 'package:sanyan_network/sanyan_network.dart';

class OauthLoginReq extends BaseReq {
  final String provider; // 'APPLE' | 'WECHAT'
  final String credential;
  final String? nonce;
  OauthLoginReq({required this.provider, required this.credential, this.nonce});
  @override
  String get path => '/api/auth/oauth/login';
  @override
  String get method => 'POST';
  @override
  Map<String, dynamic> toJson() => {
        'provider': provider,
        'credential': credential,
        if (nonce != null) 'nonce': nonce,
      };
}
```

`oauth_bind_phone_req.dart`:
```dart
import 'package:sanyan_network/sanyan_network.dart';

class OauthBindPhoneReq extends BaseReq {
  final String bindTicket;
  final String phone;
  final String code;
  final String? password;
  OauthBindPhoneReq({required this.bindTicket, required this.phone, required this.code, this.password});
  @override
  String get path => '/api/auth/oauth/bind-phone';
  @override
  String get method => 'POST';
  @override
  Map<String, dynamic> toJson() => {
        'bindTicket': bindTicket,
        'phone': phone,
        'code': code,
        if (password != null) 'password': password,
      };
}
```

- [ ] **Step 5: 加 AuthApi 方法**

在 `auth_api.dart` 顶部 import 3 个 req 与 `data/oauth_login_data.dart`，在 `abstract class AuthApi` 内加：
```dart
static Future<ApiResponse<String?>> oauthChallenge() =>
    _client.send(OauthChallengeReq(), fromData: (d) => (d as Map<String, dynamic>)['nonce'] as String?);

static Future<ApiResponse<OauthLoginData>> oauthLogin({
  required String provider,
  required String credential,
  String? nonce,
}) =>
    _client.send(
      OauthLoginReq(provider: provider, credential: credential, nonce: nonce),
      fromData: (d) => OauthLoginData.fromJson(d as Map<String, dynamic>),
    );

static Future<ApiResponse<OauthLoginData>> oauthBindPhone({
  required String bindTicket,
  required String phone,
  required String code,
  String? password,
}) =>
    _client.send(
      OauthBindPhoneReq(bindTicket: bindTicket, phone: phone, code: code, password: password),
      fromData: (d) => OauthLoginData.fromJson(d as Map<String, dynamic>),
    );
```
> 注：`oauthChallenge` 的 `fromData` 假设 `data` 是 `{nonce}` 对象。若服务端 challenge 直接返回字符串 nonce 作为 data，改为 `(d) => d as String?`。实现时按服务端实际响应（看 `OauthChallengeService` / IT 断言）确认，二选一。

- [ ] **Step 6: 运行确认通过**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test foundation_packages/sanyan_user/test/api/oauth_req_test.dart`
Expected: PASS (5 tests)

- [ ] **Step 7: 登记 suite + 跑全包**

在 `sanyan_user_suite.dart` 加 `import 'api/oauth_req_test.dart' as oauth_req_test;` + `oauth_req_test.main();`。
Run: `cd /Users/aventador/code/3yan/app && fvm flutter test foundation_packages/sanyan_user`
Expected: 全 PASS

- [ ] **Step 8: Commit**

```bash
cd /Users/aventador/code/3yan/app
git add foundation_packages/sanyan_user/lib/src/api/req/oauth_challenge_req.dart foundation_packages/sanyan_user/lib/src/api/req/oauth_login_req.dart foundation_packages/sanyan_user/lib/src/api/req/oauth_bind_phone_req.dart foundation_packages/sanyan_user/lib/src/api/auth_api.dart foundation_packages/sanyan_user/test/api/oauth_req_test.dart foundation_packages/sanyan_user/test/sanyan_user_suite.dart
git commit -m "feat(user): oauth 三接口 req 模型 + AuthApi.oauthChallenge/oauthLogin/oauthBindPhone"
```

---

## Phase B — SDK 抽象 / 网关 / 纯函数（sanyan_auth）

### Task 4: 加依赖 + sha256Hex 纯函数

**背景：** Apple 登录要求 client 把 challenge 拿到的原始 nonce 做 SHA256 传给 Apple SDK（Apple 把它写进 identityToken 的 nonce claim），原始 nonce 传服务端。先加依赖 + 落 sha256 纯函数。

**Files:**
- Modify: `business_packages/sanyan_auth/pubspec.yaml`
- Create: `business_packages/sanyan_auth/lib/src/oauth/oauth_crypto.dart`
- Test: `business_packages/sanyan_auth/test/oauth/oauth_crypto_test.dart`
- Create/Modify: `business_packages/sanyan_auth/test/sanyan_auth_suite.dart`（若无 suite 则新建）

- [ ] **Step 1: 加依赖**

在 `business_packages/sanyan_auth/pubspec.yaml` 的 `dependencies` 加（版本以 `fvm flutter pub get` 解析为准，下列为目标下限）：
```yaml
  sign_in_with_apple: ^6.1.0
  fluwx: ^4.5.5
  crypto: ^3.0.3
```
`dev_dependencies` 加：
```yaml
  mocktail: ^1.0.4
```
Run: `cd /Users/aventador/code/3yan/app/business_packages/sanyan_auth && fvm flutter pub get`
Expected: 解析成功，无版本冲突（若 fluwx/sign_in_with_apple 与现有 SDK 约束冲突，记录冲突信息 BLOCKED 上报）。

- [ ] **Step 2: 写失败测试**

`business_packages/sanyan_auth/test/oauth/oauth_crypto_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_auth/src/oauth/oauth_crypto.dart';

void main() {
  test('sha256Hex matches known vector', () {
    // sha256("abc") = ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
    expect(sha256Hex('abc'), 'ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad');
  });

  test('sha256Hex of empty string', () {
    expect(sha256Hex(''), 'e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855');
  });
}
```

- [ ] **Step 3: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/oauth_crypto_test.dart`
Expected: FAIL（函数不存在）

- [ ] **Step 4: 实现 sha256Hex**

`business_packages/sanyan_auth/lib/src/oauth/oauth_crypto.dart`:
```dart
import 'dart:convert';
import 'package:crypto/crypto.dart';

/// 对原始 nonce 做 SHA-256 并返回小写 hex。
/// Apple Sign In 要求传 sha256(nonce)，原始 nonce 发服务端比对 token 的 nonce claim。
String sha256Hex(String input) => sha256.convert(utf8.encode(input)).toString();
```

- [ ] **Step 5: 运行确认通过 + 登记 suite**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/oauth_crypto_test.dart`
Expected: PASS (2 tests)
若 `business_packages/sanyan_auth/test/sanyan_auth_suite.dart` 不存在，新建：
```dart
import 'oauth/oauth_crypto_test.dart' as oauth_crypto_test;
void main() {
  oauth_crypto_test.main();
}
```

- [ ] **Step 6: Commit**

```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_auth/pubspec.yaml business_packages/sanyan_auth/lib/src/oauth/oauth_crypto.dart business_packages/sanyan_auth/test/oauth/oauth_crypto_test.dart business_packages/sanyan_auth/test/sanyan_auth_suite.dart pubspec.lock business_packages/sanyan_auth/pubspec.lock
git commit -m "build(auth): 引入 sign_in_with_apple/fluwx/crypto/mocktail + sha256Hex 纯函数"
```

---

### Task 5: SdkAuthResult + AppleAuthProvider

**背景：** SDK 调用封到接口后面，结果用 sealed 类型表达 成功(凭证) / 用户取消 / 失败(文案)，让控制器分支可测、且把不可单测的平台调用隔离在薄实现里。

**Files:**
- Create: `business_packages/sanyan_auth/lib/src/oauth/sdk_auth_result.dart`
- Create: `business_packages/sanyan_auth/lib/src/oauth/apple_auth_provider.dart`
- Test: `business_packages/sanyan_auth/test/oauth/apple_auth_provider_test.dart`
- Modify: suite

- [ ] **Step 1: 写失败测试**（测可测的部分：result 工厂 + provider 把"插件抛取消异常"映射成 cancelled、"插件返回 token"映射成 success；插件调用用可重写的 protected 方法做 seam）

`business_packages/sanyan_auth/test/oauth/apple_auth_provider_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sign_in_with_apple/sign_in_with_apple.dart';
import 'package:sanyan_auth/src/oauth/sdk_auth_result.dart';
import 'package:sanyan_auth/src/oauth/apple_auth_provider.dart';

class _FakeApple extends AppleAuthProviderImpl {
  final Object? throwThis;
  final String? returnToken;
  _FakeApple({this.throwThis, this.returnToken});
  @override
  Future<String?> rawIdentityToken(String nonceSha256) async {
    if (throwThis != null) throw throwThis!;
    return returnToken;
  }
}

void main() {
  test('returns success credential when plugin yields identityToken', () async {
    final p = _FakeApple(returnToken: 'idtoken-xyz');
    final r = await p.obtainCredential('sha');
    expect(r, isA<SdkAuthSuccess>());
    expect((r as SdkAuthSuccess).credential, 'idtoken-xyz');
  });

  test('maps canceled exception to SdkAuthCancelled', () async {
    final p = _FakeApple(
      throwThis: const SignInWithAppleAuthorizationException(
        code: AuthorizationErrorCode.canceled, message: 'user canceled'),
    );
    final r = await p.obtainCredential('sha');
    expect(r, isA<SdkAuthCancelled>());
  });

  test('maps null token to SdkAuthFailure', () async {
    final p = _FakeApple(returnToken: null);
    final r = await p.obtainCredential('sha');
    expect(r, isA<SdkAuthFailure>());
  });

  test('maps other error to SdkAuthFailure', () async {
    final p = _FakeApple(throwThis: Exception('boom'));
    final r = await p.obtainCredential('sha');
    expect(r, isA<SdkAuthFailure>());
  });
}
```

- [ ] **Step 2: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/apple_auth_provider_test.dart`
Expected: FAIL（类不存在）

- [ ] **Step 3: 实现 sealed 结果 + provider**

`sdk_auth_result.dart`:
```dart
/// 第三方 SDK 取凭证的三态结果。
sealed class SdkAuthResult {
  const SdkAuthResult();
}

class SdkAuthSuccess extends SdkAuthResult {
  final String credential; // Apple=identityToken / WeChat=code
  const SdkAuthSuccess(this.credential);
}

class SdkAuthCancelled extends SdkAuthResult {
  const SdkAuthCancelled();
}

class SdkAuthFailure extends SdkAuthResult {
  final String message;
  const SdkAuthFailure(this.message);
}
```

`apple_auth_provider.dart`:
```dart
import 'package:sign_in_with_apple/sign_in_with_apple.dart';
import 'sdk_auth_result.dart';

/// Apple 登录凭证获取抽象（可注入 / 可 fake）。
abstract class AppleAuthProvider {
  /// [nonceSha256] = sha256Hex(challenge 拿到的原始 nonce)。
  Future<SdkAuthResult> obtainCredential(String nonceSha256);
}

class AppleAuthProviderImpl implements AppleAuthProvider {
  /// 薄平台调用 seam——测试覆写它，避免依赖真插件。
  Future<String?> rawIdentityToken(String nonceSha256) async {
    final cred = await SignInWithApple.getAppleIDCredential(
      scopes: const [AppleIDAuthorizationScopes.email, AppleIDAuthorizationScopes.fullName],
      nonce: nonceSha256,
    );
    return cred.identityToken;
  }

  @override
  Future<SdkAuthResult> obtainCredential(String nonceSha256) async {
    try {
      final token = await rawIdentityToken(nonceSha256);
      if (token == null) return const SdkAuthFailure('Apple 未返回凭证');
      return SdkAuthSuccess(token);
    } on SignInWithAppleAuthorizationException catch (e) {
      if (e.code == AuthorizationErrorCode.canceled) return const SdkAuthCancelled();
      return SdkAuthFailure('Apple 登录失败: ${e.code.name}');
    } catch (_) {
      return const SdkAuthFailure('Apple 登录失败');
    }
  }
}
```

- [ ] **Step 4: 运行确认通过**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/apple_auth_provider_test.dart`
Expected: PASS (4 tests)

- [ ] **Step 5: 登记 suite + commit**

suite 加 `sdk`/apple 测试 import+调用。
```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_auth/lib/src/oauth/sdk_auth_result.dart business_packages/sanyan_auth/lib/src/oauth/apple_auth_provider.dart business_packages/sanyan_auth/test/oauth/apple_auth_provider_test.dart business_packages/sanyan_auth/test/sanyan_auth_suite.dart
git commit -m "feat(auth): SdkAuthResult 三态 + AppleAuthProvider（取消/失败/成功映射，平台调用可 seam）"
```

---

### Task 6: WechatAuthProvider（fluwx Completer 桥接）

**背景：** fluwx 的 code 经 `addSubscriber` 异步回调返回，不是 `authBy` 的返回值。provider 用 `Completer` 把"发起 authBy + 等订阅回调"桥接成一个 `Future<SdkAuthResult>`，并在拿到结果后取消订阅。

**Files:**
- Create: `business_packages/sanyan_auth/lib/src/oauth/wechat_auth_provider.dart`
- Test: `business_packages/sanyan_auth/test/oauth/wechat_auth_provider_test.dart`
- Modify: suite

- [ ] **Step 1: 写失败测试**（fake 出"发起 + 投递回调"两个 seam，验证 success/cancel/fail 映射）

`business_packages/sanyan_auth/test/oauth/wechat_auth_provider_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_auth/src/oauth/sdk_auth_result.dart';
import 'package:sanyan_auth/src/oauth/wechat_auth_provider.dart';

/// 用可控 fake 覆写"发起授权"与"是否安装"，并手动投递回调结果。
class _FakeWechat extends WechatAuthProviderImpl {
  final bool installed;
  final WechatAuthCallbackResult deliver;
  _FakeWechat({required this.installed, required this.deliver});
  @override
  Future<bool> isInstalled() async => installed;
  @override
  Future<void> startAuth(void Function(WechatAuthCallbackResult) onResult) async {
    onResult(deliver); // 立即投递，模拟微信回调
  }
}

void main() {
  test('success delivers code', () async {
    final p = _FakeWechat(installed: true,
        deliver: const WechatAuthCallbackResult(successful: true, code: 'wxcode'));
    final r = await p.obtainCredential('');
    expect(r, isA<SdkAuthSuccess>());
    expect((r as SdkAuthSuccess).credential, 'wxcode');
  });

  test('not installed → failure', () async {
    final p = _FakeWechat(installed: false,
        deliver: const WechatAuthCallbackResult(successful: true, code: 'x'));
    final r = await p.obtainCredential('');
    expect(r, isA<SdkAuthFailure>());
  });

  test('user denied (errCode -4 / -2) → cancelled', () async {
    final p = _FakeWechat(installed: true,
        deliver: const WechatAuthCallbackResult(successful: false, errCode: -2));
    final r = await p.obtainCredential('');
    expect(r, isA<SdkAuthCancelled>());
  });

  test('failed with other errCode → failure', () async {
    final p = _FakeWechat(installed: true,
        deliver: const WechatAuthCallbackResult(successful: false, errCode: -1));
    final r = await p.obtainCredential('');
    expect(r, isA<SdkAuthFailure>());
  });
}
```

- [ ] **Step 2: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/wechat_auth_provider_test.dart`
Expected: FAIL（类不存在）

- [ ] **Step 3: 实现 WechatAuthProvider**

`wechat_auth_provider.dart`:
```dart
import 'dart:async';
import 'package:fluwx/fluwx.dart';
import 'sdk_auth_result.dart';

/// 桥接微信回调用的中间结果（把 fluwx 的 WeChatAuthResponse 归一化，便于 fake）。
class WechatAuthCallbackResult {
  final bool successful;
  final String? code;
  final int? errCode;
  const WechatAuthCallbackResult({required this.successful, this.code, this.errCode});
}

abstract class WechatAuthProvider {
  Future<SdkAuthResult> obtainCredential(String unusedNonce);
}

class WechatAuthProviderImpl implements WechatAuthProvider {
  final Fluwx _fluwx = Fluwx();

  /// seam：是否安装微信
  Future<bool> isInstalled() => _fluwx.isWeChatInstalled;

  /// seam：发起授权并把回调归一化后交给 [onResult]（只触发一次）。
  Future<void> startAuth(void Function(WechatAuthCallbackResult) onResult) async {
    late final dynamic cancelable;
    cancelable = _fluwx.addSubscriber((response) {
      if (response is WeChatAuthResponse) {
        onResult(WechatAuthCallbackResult(
          successful: response.isSuccessful,
          code: response.code,
          errCode: response.errCode,
        ));
        cancelable.cancel();
      }
    });
    await _fluwx.authBy(which: NormalAuth(scope: 'snsapi_userinfo', state: 'sanyan_login'));
  }

  @override
  Future<SdkAuthResult> obtainCredential(String unusedNonce) async {
    if (!await isInstalled()) return const SdkAuthFailure('未安装微信');
    final completer = Completer<SdkAuthResult>();
    await startAuth((r) {
      if (completer.isCompleted) return;
      if (r.successful && r.code != null) {
        completer.complete(SdkAuthSuccess(r.code!));
      } else if (r.errCode == -2 || r.errCode == -4) {
        // -2 用户取消 / -4 用户拒绝授权
        completer.complete(const SdkAuthCancelled());
      } else {
        completer.complete(SdkAuthFailure('微信登录失败 (${r.errCode})'));
      }
    });
    return completer.future;
  }
}
```
> 实现注意：`WeChatAuthResponse` 的字段名（`isSuccessful`/`code`/`errCode`）以装好的 fluwx 版本为准（已按 4.x 文档写）；若版本字段名不同，只调整 `startAuth` 里的归一化，不动 `obtainCredential` 逻辑。

- [ ] **Step 4: 运行确认通过 + 登记 suite + commit**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/wechat_auth_provider_test.dart`
Expected: PASS (4 tests)
```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_auth/lib/src/oauth/wechat_auth_provider.dart business_packages/sanyan_auth/test/oauth/wechat_auth_provider_test.dart business_packages/sanyan_auth/test/sanyan_auth_suite.dart
git commit -m "feat(auth): WechatAuthProvider（fluwx Completer 桥接，未装/取消/失败/成功映射）"
```

---

### Task 7: OauthGateway（包 AuthApi static 的可注入网关）

**背景：** AuthApi 是 static 不可 mock。为让控制器分支逻辑可测，引入薄网关接口（仅作 DI 测试 seam），真实现委托给 static AuthApi。

**Files:**
- Create: `business_packages/sanyan_auth/lib/src/oauth/oauth_gateway.dart`
- Test: `business_packages/sanyan_auth/test/oauth/oauth_gateway_test.dart`（仅验证接口契约/真实现可构造；委托逻辑无分支，做轻测试）
- Modify: suite

- [ ] **Step 1: 写失败测试**（断言抽象方法签名 + 真实现实例化不抛）

`business_packages/sanyan_auth/test/oauth/oauth_gateway_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_auth/src/oauth/oauth_gateway.dart';

void main() {
  test('OauthGatewayImpl is constructable and is an OauthGateway', () {
    final g = OauthGatewayImpl();
    expect(g, isA<OauthGateway>());
  });
}
```

- [ ] **Step 2: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/oauth_gateway_test.dart`
Expected: FAIL（类不存在）

- [ ] **Step 3: 实现网关**

`oauth_gateway.dart`:
```dart
import 'package:sanyan_network/sanyan_network.dart';
import 'package:sanyan_user/sanyan_user.dart';

/// OAuth 后端调用网关（DI 测试 seam）。控制器依赖本接口，单测用 fake 注入。
abstract class OauthGateway {
  Future<ApiResponse<String?>> challenge();
  Future<ApiResponse<OauthLoginData>> login({required String provider, required String credential, String? nonce});
  Future<ApiResponse<OauthLoginData>> bindPhone({required String bindTicket, required String phone, required String code, String? password});
}

class OauthGatewayImpl implements OauthGateway {
  @override
  Future<ApiResponse<String?>> challenge() => AuthApi.oauthChallenge();
  @override
  Future<ApiResponse<OauthLoginData>> login({required String provider, required String credential, String? nonce}) =>
      AuthApi.oauthLogin(provider: provider, credential: credential, nonce: nonce);
  @override
  Future<ApiResponse<OauthLoginData>> bindPhone({required String bindTicket, required String phone, required String code, String? password}) =>
      AuthApi.oauthBindPhone(bindTicket: bindTicket, phone: phone, code: code, password: password);
}
```

- [ ] **Step 4: 运行确认通过 + 登记 suite + commit**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/oauth_gateway_test.dart`
Expected: PASS (1 test)
```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_auth/lib/src/oauth/oauth_gateway.dart business_packages/sanyan_auth/test/oauth/oauth_gateway_test.dart business_packages/sanyan_auth/test/sanyan_auth_suite.dart
git commit -m "feat(auth): OauthGateway 网关接口（包 AuthApi static，作控制器测试 seam）"
```

---

## Phase C — 登录成功复用 + 控制器（sanyan_auth 业务逻辑，fake 单测）

### Task 8: LoginSuccessHandler 抽取 + 现有控制器改用

**背景：** 登录成功五步链现内联在 LoginController 且被 RegisterController 复制。抽成可注入 `LoginSuccessHandler`（DRY + 让 oauth 控制器复用 + 可在测试中验证调用），并把现有两个控制器改用它。

**Files:**
- Create: `business_packages/sanyan_auth/lib/src/auth/login_success_handler.dart`
- Modify: `business_packages/sanyan_auth/lib/src/auth/login_controller.dart`
- Modify: `business_packages/sanyan_auth/lib/src/auth/register_controller.dart`
- Test: `business_packages/sanyan_auth/test/auth/login_success_handler_test.dart`
- Modify: suite

- [ ] **Step 1: 先 Read** `login_controller.dart` 与 `register_controller.dart`，确认登录成功段确切代码。

- [ ] **Step 2: 写失败测试**（验证 handler 把 token/userId 写入 LocalStorage；用真 LocalStorage + GetStorage 内存初始化；导航与 WS 副作用用真实现的可覆写 seam 验证调用）

`business_packages/sanyan_auth/test/auth/login_success_handler_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:get_storage/get_storage.dart';
import 'package:sanyan_user/sanyan_user.dart';
import 'package:sanyan_auth/src/auth/login_success_handler.dart';

class _SpyHandler extends LoginSuccessHandlerImpl {
  int connectCount = 0;
  int navCount = 0;
  int pushCount = 0;
  @override
  void connectWs() => connectCount++;
  @override
  void navigateToChat() => navCount++;
  @override
  void registerPushToken() => pushCount++;
}

void main() {
  setUpAll(() async {
    await GetStorage.init();
  });

  test('handle writes token+userId and triggers push/ws/nav once', () {
    final h = _SpyHandler();
    h.handle(token: 'jwt-1', userId: 7);
    expect(LocalStorage.token, 'jwt-1');
    expect(LocalStorage.userId, 7);
    expect(h.pushCount, 1);
    expect(h.connectCount, 1);
    expect(h.navCount, 1);
  });
}
```

- [ ] **Step 3: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/auth/login_success_handler_test.dart`
Expected: FAIL（类不存在）

- [ ] **Step 4: 实现 LoginSuccessHandler**

`login_success_handler.dart`:
```dart
import 'package:get/get.dart';
import 'package:sanyan_network/sanyan_network.dart';
import 'package:sanyan_routes/sanyan_routes.dart';
import 'package:sanyan_user/sanyan_user.dart';

/// 登录成功统一处理（手机号登录/注册/第三方登录三处复用）。
abstract class LoginSuccessHandler {
  void handle({required String token, required int userId});
}

class LoginSuccessHandlerImpl implements LoginSuccessHandler {
  // 副作用 seam（测试可覆写）
  void registerPushToken() => AuthApi.registerPushTokenAfterLogin();
  void connectWs() => Get.find<WsClient>().connect();
  void navigateToChat() => Get.offAllNamed(AppRoutes.chat);

  @override
  void handle({required String token, required int userId}) {
    LocalStorage.token = token;
    LocalStorage.userId = userId;
    registerPushToken();
    connectWs();
    navigateToChat();
  }
}
```
> 注：`AuthApi.registerPushTokenAfterLogin()` 沿用现有方法名（recon 确认存在）。`WsClient`/`AppRoutes` import 路径以现有包 barrel 为准。

- [ ] **Step 5: 改 LoginController 与 RegisterController 复用 handler**

两个控制器各持一个 `LoginSuccessHandler`（默认 `LoginSuccessHandlerImpl()`，可注入 fake）。把原内联的 "写 token→写 userId→push→connect→offAllNamed" 五行替换为：
```dart
_successHandler.handle(token: resp.data!['token'] as String, userId: resp.data!['userId'] as int);
```
LoginController/RegisterController 顶部加字段（用可选构造参数注入，默认真实现）：
```dart
final LoginSuccessHandler _successHandler;
LoginController({LoginSuccessHandler? successHandler})
    : _successHandler = successHandler ?? LoginSuccessHandlerImpl();
```
RegisterController 同理。**删除两处内联的重复登录成功代码**（删除规范：连重复块一起删）。

- [ ] **Step 6: 运行确认通过 + 回归 auth 包**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth`
Expected: 新 handler 测试 PASS；若 auth 包此前无测试，至少新测试 PASS、`fvm flutter analyze` 无 error。
Run: `cd /Users/aventador/code/3yan/app && fvm flutter analyze business_packages/sanyan_auth`
Expected: No issues（或仅既有 warning）。

- [ ] **Step 7: Commit**

```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_auth/lib/src/auth/login_success_handler.dart business_packages/sanyan_auth/lib/src/auth/login_controller.dart business_packages/sanyan_auth/lib/src/auth/register_controller.dart business_packages/sanyan_auth/test/auth/login_success_handler_test.dart business_packages/sanyan_auth/test/sanyan_auth_suite.dart
git commit -m "refactor(auth): 抽取 LoginSuccessHandler，登录/注册复用（消除登录成功链重复）"
```

---

### Task 9: OauthLoginController（编排 + 分支）

**背景：** 编排 Apple/微信登录：Apple 先 challenge 拿 nonce → sha256 → provider 取凭证 → oauthLogin；微信直接 provider 取 code → oauthLogin。按响应分支：命中→handler.handle；needBind→跳 bindPhone 页带 ticket；needMergeAuth→snackbar 引导；取消→静默；失败→snackbar。

**Files:**
- Create: `business_packages/sanyan_auth/lib/src/oauth/oauth_outcome.dart`
- Create: `business_packages/sanyan_auth/lib/src/oauth/oauth_login_controller.dart`
- Test: `business_packages/sanyan_auth/test/oauth/oauth_login_controller_test.dart`
- Modify: suite

- [ ] **Step 1: 写失败测试**（fake provider + fake gateway + fake handler；导航/snackbar 用控制器的可覆写 seam 计数；覆盖 5 个分支 + Apple challenge 失败）

`business_packages/sanyan_auth/test/oauth/oauth_login_controller_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_network/sanyan_network.dart';
import 'package:sanyan_user/sanyan_user.dart';
import 'package:sanyan_auth/src/oauth/sdk_auth_result.dart';
import 'package:sanyan_auth/src/oauth/apple_auth_provider.dart';
import 'package:sanyan_auth/src/oauth/wechat_auth_provider.dart';
import 'package:sanyan_auth/src/oauth/oauth_gateway.dart';
import 'package:sanyan_auth/src/auth/login_success_handler.dart';
import 'package:sanyan_auth/src/oauth/oauth_login_controller.dart';

class _FakeApple implements AppleAuthProvider {
  final SdkAuthResult result;
  _FakeApple(this.result);
  @override
  Future<SdkAuthResult> obtainCredential(String n) async => result;
}
class _FakeWechat implements WechatAuthProvider {
  final SdkAuthResult result;
  _FakeWechat(this.result);
  @override
  Future<SdkAuthResult> obtainCredential(String n) async => result;
}
class _FakeGateway implements OauthGateway {
  ApiResponse<String?> challengeResp;
  ApiResponse<OauthLoginData> loginResp;
  _FakeGateway({required this.challengeResp, required this.loginResp});
  @override
  Future<ApiResponse<String?>> challenge() async => challengeResp;
  @override
  Future<ApiResponse<OauthLoginData>> login({required String provider, required String credential, String? nonce}) async => loginResp;
  @override
  Future<ApiResponse<OauthLoginData>> bindPhone({required String bindTicket, required String phone, required String code, String? password}) async =>
      throw UnimplementedError();
}
class _SpyHandler implements LoginSuccessHandler {
  int calls = 0; String? token; int? userId;
  @override
  void handle({required String token, required int userId}) { calls++; this.token = token; this.userId = userId; }
}

/// 覆写导航/提示 seam 以计数
class _TestController extends OauthLoginController {
  _TestController({required super.apple, required super.wechat, required super.gateway, required super.successHandler});
  String? lastBindTicket; int navBind = 0; List<String> snacks = [];
  @override
  void goToBindPhone(String bindTicket) { navBind++; lastBindTicket = bindTicket; }
  @override
  void showError(String msg) => snacks.add(msg);
}

_TestController build({required SdkAuthResult appleResult, required ApiResponse<String?> challengeResp, required ApiResponse<OauthLoginData> loginResp, _SpyHandler? handler}) {
  return _TestController(
    apple: _FakeApple(appleResult),
    wechat: _FakeWechat(appleResult),
    gateway: _FakeGateway(challengeResp: challengeResp, loginResp: loginResp),
    successHandler: handler ?? _SpyHandler(),
  );
}

void main() {
  final okChallenge = ApiResponse<String?>(success: true, data: 'raw-nonce');

  test('apple hit → successHandler.handle called', () async {
    final h = _SpyHandler();
    final c = build(
      appleResult: const SdkAuthSuccess('idtoken'),
      challengeResp: okChallenge,
      loginResp: ApiResponse(success: true, data: const OauthLoginData(token: 'jwt', userId: 9)),
      handler: h,
    );
    await c.loginWithApple();
    expect(h.calls, 1);
    expect(h.token, 'jwt');
    expect(h.userId, 9);
  });

  test('apple needBind → navigate to bindPhone with ticket', () async {
    final c = build(
      appleResult: const SdkAuthSuccess('idtoken'),
      challengeResp: okChallenge,
      loginResp: ApiResponse(success: true, data: const OauthLoginData(needBind: true, bindTicket: 'bt-9')),
    );
    await c.loginWithApple();
    expect(c.navBind, 1);
    expect(c.lastBindTicket, 'bt-9');
  });

  test('apple needMergeAuth → error snackbar, no nav', () async {
    final c = build(
      appleResult: const SdkAuthSuccess('idtoken'),
      challengeResp: okChallenge,
      loginResp: ApiResponse(success: true, data: const OauthLoginData(needMergeAuth: true)),
    );
    await c.loginWithApple();
    expect(c.navBind, 0);
    expect(c.snacks, isNotEmpty);
  });

  test('user cancelled → silent (no nav, no snackbar)', () async {
    final c = build(
      appleResult: const SdkAuthCancelled(),
      challengeResp: okChallenge,
      loginResp: ApiResponse(success: true, data: const OauthLoginData(token: 'x', userId: 1)),
    );
    await c.loginWithApple();
    expect(c.navBind, 0);
    expect(c.snacks, isEmpty);
  });

  test('sdk failure → error snackbar', () async {
    final c = build(
      appleResult: const SdkAuthFailure('Apple 登录失败'),
      challengeResp: okChallenge,
      loginResp: ApiResponse(success: true, data: const OauthLoginData(token: 'x', userId: 1)),
    );
    await c.loginWithApple();
    expect(c.snacks, isNotEmpty);
  });

  test('challenge failure → error snackbar, sdk not called path', () async {
    final c = build(
      appleResult: const SdkAuthSuccess('idtoken'),
      challengeResp: ApiResponse<String?>(success: false, message: 'challenge 失败'),
      loginResp: ApiResponse(success: true, data: const OauthLoginData(token: 'x', userId: 1)),
    );
    await c.loginWithApple();
    expect(c.snacks, isNotEmpty);
  });

  test('login api failure → error snackbar', () async {
    final c = build(
      appleResult: const SdkAuthSuccess('idtoken'),
      challengeResp: okChallenge,
      loginResp: ApiResponse(success: false, code: 1010, message: '第三方校验失败'),
    );
    await c.loginWithApple();
    expect(c.snacks, isNotEmpty);
  });
}
```

- [ ] **Step 2: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/oauth_login_controller_test.dart`
Expected: FAIL（类不存在）

- [ ] **Step 3: 实现 outcome 纯类型（可选）+ 控制器**

`oauth_outcome.dart`（轻量，给可读分支用，可不单测）：
```dart
enum OauthOutcomeKind { loggedIn, needBind, needMerge, cancelled, failed }
```

`oauth_login_controller.dart`:
```dart
import 'package:get/get.dart';
import 'package:sanyan_network/sanyan_network.dart';
import 'package:sanyan_routes/sanyan_routes.dart';
import 'package:sanyan_user/sanyan_user.dart';
import '../auth/login_success_handler.dart';
import 'apple_auth_provider.dart';
import 'wechat_auth_provider.dart';
import 'oauth_gateway.dart';
import 'oauth_crypto.dart';
import 'sdk_auth_result.dart';
import 'bind_phone_page.dart';

class OauthLoginController extends GetxController {
  final AppleAuthProvider apple;
  final WechatAuthProvider wechat;
  final OauthGateway gateway;
  final LoginSuccessHandler successHandler;
  final loading = false.obs;

  OauthLoginController({
    required this.apple,
    required this.wechat,
    required this.gateway,
    required this.successHandler,
  });

  Future<void> loginWithApple() async {
    if (loading.value) return;
    loading.value = true;
    try {
      final ch = await gateway.challenge();
      if (!ch.success || ch.data == null) {
        showError(ch.errMsg ?? '获取登录凭据失败');
        return;
      }
      final rawNonce = ch.data!;
      final sdk = await apple.obtainCredential(sha256Hex(rawNonce));
      await _afterSdk(sdk, provider: 'APPLE', nonce: rawNonce);
    } finally {
      loading.value = false;
    }
  }

  Future<void> loginWithWechat() async {
    if (loading.value) return;
    loading.value = true;
    try {
      final sdk = await wechat.obtainCredential('');
      await _afterSdk(sdk, provider: 'WECHAT', nonce: null);
    } finally {
      loading.value = false;
    }
  }

  Future<void> _afterSdk(SdkAuthResult sdk, {required String provider, String? nonce}) async {
    switch (sdk) {
      case SdkAuthCancelled():
        return; // 静默
      case SdkAuthFailure(message: final m):
        showError(m);
        return;
      case SdkAuthSuccess(credential: final cred):
        final resp = await gateway.login(provider: provider, credential: cred, nonce: nonce);
        if (!resp.success || resp.data == null) {
          showError(resp.errMsg ?? '登录失败');
          return;
        }
        final d = resp.data!;
        if (d.needBind && d.bindTicket != null) {
          goToBindPhone(d.bindTicket!);
        } else if (d.needMergeAuth) {
          showError('该手机号已注册，请用该手机号登录后在设置里绑定');
        } else if (d.loggedIn) {
          successHandler.handle(token: d.token!, userId: d.userId!);
        } else {
          showError('登录失败');
        }
    }
  }

  // 导航/提示 seam（测试覆写）
  void goToBindPhone(String bindTicket) => Get.to(() => BindPhonePage(bindTicket: bindTicket));
  void showError(String msg) => Get.snackbar('登录失败', msg);
}
```
> 注：本文件 import 了 Task 12 才创建的 `bind_phone_page.dart`。为保证本 Task 可独立编译通过，**先在 Task 9 创建 `bind_phone_page.dart` 的最小占位**（一个接收 `bindTicket` 的 StatelessWidget 空壳），Task 12 再补全 UI。或调整 Task 顺序：先做 Task 12 的页面骨架。实现者择一，保证每步可编译。

- [ ] **Step 4: 运行确认通过**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/oauth_login_controller_test.dart`
Expected: PASS (7 tests)

- [ ] **Step 5: 登记 suite + commit**

```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_auth/lib/src/oauth/oauth_outcome.dart business_packages/sanyan_auth/lib/src/oauth/oauth_login_controller.dart business_packages/sanyan_auth/lib/src/oauth/bind_phone_page.dart business_packages/sanyan_auth/test/oauth/oauth_login_controller_test.dart business_packages/sanyan_auth/test/sanyan_auth_suite.dart
git commit -m "feat(auth): OauthLoginController 编排 Apple/微信登录 + 五分支处理（命中/绑定/合并/取消/失败）"
```

---

### Task 10: BindPhoneController（绑手机 + 倒计时 + 合并安全提示）

**背景：** 绑手机页控制器：手机号 + 验证码（复用 60s 倒计时）+ 可选密码（合并已有账号时）。提交 → oauthBindPhone → 命中→handler.handle；needMergeAuth→提示需本人证明；失败（如 1013 票据失效 / 1014 密码错）→snackbar。

**Files:**
- Create: `business_packages/sanyan_auth/lib/src/oauth/bind_phone_controller.dart`
- Test: `business_packages/sanyan_auth/test/oauth/bind_phone_controller_test.dart`
- Modify: suite

- [ ] **Step 1: 写失败测试**（fake gateway + fake handler + 覆写 seam；覆盖 提交成功/needMerge/失败/手机号校验/发码倒计时启动）

`business_packages/sanyan_auth/test/oauth/bind_phone_controller_test.dart`:
```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sanyan_network/sanyan_network.dart';
import 'package:sanyan_user/sanyan_user.dart';
import 'package:sanyan_auth/src/oauth/oauth_gateway.dart';
import 'package:sanyan_auth/src/auth/login_success_handler.dart';
import 'package:sanyan_auth/src/oauth/bind_phone_controller.dart';

class _FakeGateway implements OauthGateway {
  ApiResponse<OauthLoginData> bindResp;
  bool smsOk;
  int bindCalls = 0;
  _FakeGateway({required this.bindResp, this.smsOk = true});
  @override
  Future<ApiResponse<String?>> challenge() async => throw UnimplementedError();
  @override
  Future<ApiResponse<OauthLoginData>> login({required String provider, required String credential, String? nonce}) async => throw UnimplementedError();
  @override
  Future<ApiResponse<OauthLoginData>> bindPhone({required String bindTicket, required String phone, required String code, String? password}) async {
    bindCalls++;
    return bindResp;
  }
}
class _SpyHandler implements LoginSuccessHandler {
  int calls = 0;
  @override
  void handle({required String token, required int userId}) => calls++;
}
class _TestController extends BindPhoneController {
  _TestController({required super.bindTicket, required super.gateway, required super.successHandler});
  List<String> errs = []; List<String> infos = []; int smsSends = 0;
  @override
  void showError(String m) => errs.add(m);
  @override
  void showInfo(String m) => infos.add(m);
  @override
  Future<bool> sendSmsApi(String phone) async { smsSends++; return true; }
}

void main() {
  _TestController make(ApiResponse<OauthLoginData> resp, {_SpyHandler? h}) => _TestController(
    bindTicket: 'bt', gateway: _FakeGateway(bindResp: resp), successHandler: h ?? _SpyHandler());

  test('submit success → handler.handle', () async {
    final h = _SpyHandler();
    final c = make(ApiResponse(success: true, data: const OauthLoginData(token: 'jwt', userId: 5)), h: h);
    c.phone.value = '13800000000'; c.code.value = '1234';
    await c.submit();
    expect(h.calls, 1);
  });

  test('submit needMergeAuth → info prompt, no handler', () async {
    final h = _SpyHandler();
    final c = make(ApiResponse(success: true, data: const OauthLoginData(needMergeAuth: true)), h: h);
    c.phone.value = '13800000000'; c.code.value = '1234';
    await c.submit();
    expect(h.calls, 0);
    expect(c.infos, isNotEmpty);
  });

  test('submit failure (ticket used) → error', () async {
    final c = make(ApiResponse(success: false, code: 1013, message: '绑定会话已使用'));
    c.phone.value = '13800000000'; c.code.value = '1234';
    await c.submit();
    expect(c.errs, isNotEmpty);
  });

  test('submit blocked when phone invalid', () async {
    final c = make(ApiResponse(success: true, data: const OauthLoginData(token: 't', userId: 1)));
    c.phone.value = '123'; c.code.value = '1234';
    await c.submit();
    expect(c.errs, isNotEmpty); // 手机号格式错
  });

  test('sendSms starts 60s countdown', () async {
    final c = make(ApiResponse(success: true, data: const OauthLoginData(token: 't', userId: 1)));
    c.phone.value = '13800000000';
    await c.sendSms();
    expect(c.countdown.value, 60);
    expect(c.smsSends, 1);
  });

  test('sendSms blocked when phone invalid', () async {
    final c = make(ApiResponse(success: true, data: const OauthLoginData(token: 't', userId: 1)));
    c.phone.value = 'bad';
    await c.sendSms();
    expect(c.countdown.value, 0);
    expect(c.smsSends, 0);
  });
}
```

- [ ] **Step 2: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/bind_phone_controller_test.dart`
Expected: FAIL（类不存在）

- [ ] **Step 3: 实现 BindPhoneController**

`bind_phone_controller.dart`:
```dart
import 'dart:async';
import 'package:get/get.dart';
import 'package:sanyan_user/sanyan_user.dart';
import '../auth/login_success_handler.dart';
import 'oauth_gateway.dart';

class BindPhoneController extends GetxController {
  final String bindTicket;
  final OauthGateway gateway;
  final LoginSuccessHandler successHandler;

  final phone = ''.obs;
  final code = ''.obs;
  final password = ''.obs;
  final countdown = 0.obs;
  final submitting = false.obs;
  Timer? _timer;

  BindPhoneController({required this.bindTicket, required this.gateway, required this.successHandler});

  static final RegExp _phoneReg = RegExp(r'^1\d{10}$');

  Future<void> sendSms() async {
    if (countdown.value > 0) return;
    if (!_phoneReg.hasMatch(phone.value)) {
      showError('请输入正确的手机号');
      return;
    }
    final ok = await sendSmsApi(phone.value);
    if (ok) _startCountdown();
  }

  Future<void> submit() async {
    if (submitting.value) return;
    if (!_phoneReg.hasMatch(phone.value)) {
      showError('请输入正确的手机号');
      return;
    }
    if (code.value.isEmpty) {
      showError('请输入验证码');
      return;
    }
    submitting.value = true;
    try {
      final resp = await gateway.bindPhone(
        bindTicket: bindTicket,
        phone: phone.value,
        code: code.value,
        password: password.value.isEmpty ? null : password.value,
      );
      if (!resp.success || resp.data == null) {
        showError(resp.errMsg ?? '绑定失败');
        return;
      }
      final d = resp.data!;
      if (d.needMergeAuth) {
        showInfo('该手机号已注册，请输入该账号密码以验证本人，或用该手机号登录后在设置里绑定');
      } else if (d.loggedIn) {
        successHandler.handle(token: d.token!, userId: d.userId!);
      } else {
        showError('绑定失败');
      }
    } finally {
      submitting.value = false;
    }
  }

  void _startCountdown() {
    countdown.value = 60;
    _timer?.cancel();
    _timer = Timer.periodic(const Duration(seconds: 1), (t) {
      if (countdown.value <= 0) { t.cancel(); return; }
      countdown.value--;
    });
  }

  @override
  void onClose() {
    _timer?.cancel();
    super.onClose();
  }

  // seam（测试覆写）
  Future<bool> sendSmsApi(String phone) async {
    final resp = await AuthApi.sendSms(phone);
    if (!resp.success) showError(resp.errMsg ?? '验证码发送失败');
    return resp.success;
  }
  void showError(String msg) => Get.snackbar('提示', msg);
  void showInfo(String msg) => Get.snackbar('需要验证', msg);
}
```
> 注：`AuthApi.sendSms(phone)` 沿用现有方法（recon 确认 `auth_api.dart:10`）。

- [ ] **Step 4: 运行确认通过 + 登记 suite + commit**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/bind_phone_controller_test.dart`
Expected: PASS (6 tests)
```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_auth/lib/src/oauth/bind_phone_controller.dart business_packages/sanyan_auth/test/oauth/bind_phone_controller_test.dart business_packages/sanyan_auth/test/sanyan_auth_suite.dart
git commit -m "feat(auth): BindPhoneController（绑手机+60s倒计时+合并安全提示+票据失效处理）"
```

---

## Phase D — UI + 路由 + 装配

### Task 11: 登录页加第三方登录按钮区

**背景：** 在 `login_page.dart` 的"立即注册"链接上方加 分割线 + Apple/微信登录按钮，点击触发 `OauthLoginController`。沿用 `GlassPanel`/`AuraButton`/`AuraColors` 设计系统。

**Files:**
- Modify: `business_packages/sanyan_auth/lib/src/auth/login_page.dart`
- Test: `business_packages/sanyan_auth/test/auth/login_page_oauth_test.dart`

- [ ] **Step 1: 先 Read** `login_page.dart` 确认现有结构与 `OauthLoginController` 的注入方式（用 `Get.find<OauthLoginController>()`，在 Task 13 的 main.dart 里 `Get.put`；测试里用 `Get.put` 注入 fake/spy 控制器）。

- [ ] **Step 2: 写失败 widget 测试**（验证两个按钮存在 + 点击触发控制器方法；用一个记录调用的 spy 控制器经 Get 注入）

`business_packages/sanyan_auth/test/auth/login_page_oauth_test.dart`:
```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:get/get.dart';
import 'package:sanyan_auth/src/oauth/oauth_login_controller.dart';
import 'package:sanyan_auth/src/auth/login_page.dart';

class _SpyOauth extends OauthLoginController {
  _SpyOauth() : super(apple: _na(), wechat: _na2(), gateway: _na3(), successHandler: _na4());
  int apple = 0; int wechat = 0;
  @override
  Future<void> loginWithApple() async => apple++;
  @override
  Future<void> loginWithWechat() async => wechat++;
}
// 下面 4 个占位仅为构造 super，spy 已覆写不会真用到它们
// 用 throw 占位即可，因覆写后不触达
dynamic _na() => throw UnimplementedError();
// ... 实际写时：让 spy 不调用 super 的依赖，用最简单 Fake 实现替代（见 Task 9 的 Fake 模式）

void main() {
  testWidgets('login page shows Apple & WeChat buttons and taps route to controller', (tester) async {
    final spy = _SpyOauth();
    Get.put<OauthLoginController>(spy);
    await tester.pumpWidget(const GetMaterialApp(home: LoginPage()));
    await tester.pumpAndSettle();

    expect(find.byKey(const Key('btn_oauth_apple')), findsOneWidget);
    expect(find.byKey(const Key('btn_oauth_wechat')), findsOneWidget);

    await tester.tap(find.byKey(const Key('btn_oauth_apple')));
    await tester.pump();
    expect(spy.apple, 1);

    await tester.tap(find.byKey(const Key('btn_oauth_wechat')));
    await tester.pump();
    expect(spy.wechat, 1);

    Get.delete<OauthLoginController>();
  });
}
```
> 实现者注意：上面 `_na()` 占位是伪代码，落地时改用 Task 9 测试里的 `_FakeApple`/`_FakeWechat`/`_FakeGateway`/`_SpyHandler`（提取成 `test/oauth/fakes.dart` 复用），构造 `_SpyOauth` 时传入这些 Fake，spy 覆写两个 login 方法即可。

- [ ] **Step 3: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/auth/login_page_oauth_test.dart`
Expected: FAIL（按钮 key 不存在）

- [ ] **Step 4: 改 login_page.dart 加按钮区**

在登录 `AuraButton` 之后、"立即注册" 之前，插入：
```dart
const SizedBox(height: 16),
Row(children: const [
  Expanded(child: Divider()),
  Padding(padding: EdgeInsets.symmetric(horizontal: 8), child: Text('其他登录方式')),
  Expanded(child: Divider()),
]),
const SizedBox(height: 16),
Builder(builder: (_) {
  final oauth = Get.find<OauthLoginController>();
  return Column(children: [
    AuraButton(
      key: const Key('btn_oauth_apple'),
      onPressed: oauth.loginWithApple,
      child: const Text('使用 Apple 登录'),
    ),
    const SizedBox(height: 12),
    AuraButton(
      key: const Key('btn_oauth_wechat'),
      onPressed: oauth.loginWithWechat,
      child: const Text('使用微信登录'),
    ),
  ]);
}),
```
> `AuraButton` 的确切参数名（`onPressed`/`child` vs `text`/`onTap`）以现有定义为准，先 Read `sanyan_common_ui` 的 AuraButton。Apple 按钮按平台展示策略（仅 iOS 显示）二期再做，本期都展示。

- [ ] **Step 5: 运行确认通过 + analyze + commit**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/auth/login_page_oauth_test.dart`
Expected: PASS (1 test)
Run: `cd /Users/aventador/code/3yan/app && fvm flutter analyze business_packages/sanyan_auth`
Expected: No error
```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_auth/lib/src/auth/login_page.dart business_packages/sanyan_auth/test/auth/login_page_oauth_test.dart business_packages/sanyan_auth/test/oauth/fakes.dart
git commit -m "feat(auth): 登录页加 Apple/微信登录按钮区，点击触发 OauthLoginController"
```

---

### Task 12: BindPhonePage UI + 路由注册

**背景：** 绑手机页：手机号输入、验证码输入 + 倒计时按钮（复用注册页模式）、可选密码输入（合并已有账号用）、绑定按钮。`OauthLoginController.goToBindPhone` 已用 `Get.to(() => BindPhonePage(bindTicket: ...))` 构造跳转（不走 Get.arguments，符合 spec）。

**Files:**
- Modify/Complete: `business_packages/sanyan_auth/lib/src/oauth/bind_phone_page.dart`（Task 9 已建占位，这里补全）
- Modify: `foundation_packages/sanyan_routes/lib/src/routes.dart`（加 `bindPhone` 常量，备深链用）
- Test: `business_packages/sanyan_auth/test/oauth/bind_phone_page_test.dart`

- [ ] **Step 1: 写失败 widget 测试**（页面渲染 3 个输入 + 倒计时按钮 + 绑定按钮；点验证码按钮触发 controller.sendSms；点绑定触发 controller.submit）

`business_packages/sanyan_auth/test/oauth/bind_phone_page_test.dart`:
```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:get/get.dart';
import 'package:sanyan_auth/src/oauth/bind_phone_page.dart';

void main() {
  testWidgets('renders inputs and bind button', (tester) async {
    await tester.pumpWidget(const GetMaterialApp(home: BindPhonePage(bindTicket: 'bt')));
    await tester.pumpAndSettle();
    expect(find.byKey(const Key('input_bind_phone')), findsOneWidget);
    expect(find.byKey(const Key('input_bind_code')), findsOneWidget);
    expect(find.byKey(const Key('btn_send_sms')), findsOneWidget);
    expect(find.byKey(const Key('btn_bind_submit')), findsOneWidget);
  });
}
```

- [ ] **Step 2: 运行确认失败**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/bind_phone_page_test.dart`
Expected: FAIL（占位页面无这些 key）

- [ ] **Step 3: 实现 BindPhonePage**

`bind_phone_page.dart`（补全占位）：页面内 `Get.put(BindPhoneController(bindTicket: bindTicket, gateway: Get.find<OauthGateway>(), successHandler: Get.find<LoginSuccessHandler>()))`——**但禁止在 build 里 Get.put**（项目规则）。改用页面级 `Binding` 或在 `initState`（StatefulWidget）里注入。推荐 StatefulWidget：`initState` 里 `Get.put(BindPhoneController(...), tag: bindTicket)`，`dispose` 里 `Get.delete`。UI 用 `GlassPanel` + `AuraInput`（手机号 key `input_bind_phone`、验证码 key `input_bind_code`、密码 `input_bind_password`）+ 验证码 `Obx` 按钮（key `btn_send_sms`，`countdown>0` 显示秒数禁用）+ 绑定 `AuraButton`（key `btn_bind_submit`，`onPressed: c.submit`）。输入框 `onChanged` 写入 `c.phone.value` 等。
```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:sanyan_user/sanyan_user.dart';
import '../auth/login_success_handler.dart';
import 'oauth_gateway.dart';
import 'bind_phone_controller.dart';

class BindPhonePage extends StatefulWidget {
  final String bindTicket;
  const BindPhonePage({super.key, required this.bindTicket});
  @override
  State<BindPhonePage> createState() => _BindPhonePageState();
}

class _BindPhonePageState extends State<BindPhonePage> {
  late final BindPhoneController c;
  @override
  void initState() {
    super.initState();
    c = Get.put(
      BindPhoneController(
        bindTicket: widget.bindTicket,
        gateway: Get.find<OauthGateway>(),
        successHandler: Get.find<LoginSuccessHandler>(),
      ),
      tag: widget.bindTicket,
    );
  }
  @override
  void dispose() {
    Get.delete<BindPhoneController>(tag: widget.bindTicket);
    super.dispose();
  }
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('绑定手机号')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(children: [
          TextField(
            key: const Key('input_bind_phone'),
            keyboardType: TextInputType.phone,
            decoration: const InputDecoration(labelText: '手机号'),
            onChanged: (v) => c.phone.value = v,
          ),
          Row(children: [
            Expanded(child: TextField(
              key: const Key('input_bind_code'),
              decoration: const InputDecoration(labelText: '验证码'),
              onChanged: (v) => c.code.value = v,
            )),
            Obx(() => TextButton(
              key: const Key('btn_send_sms'),
              onPressed: c.countdown.value > 0 ? null : c.sendSms,
              child: Text(c.countdown.value > 0 ? '${c.countdown.value}s' : '获取验证码'),
            )),
          ]),
          TextField(
            key: const Key('input_bind_password'),
            obscureText: true,
            decoration: const InputDecoration(labelText: '密码（已有账号时填写）'),
            onChanged: (v) => c.password.value = v,
          ),
          const SizedBox(height: 24),
          Obx(() => ElevatedButton(
            key: const Key('btn_bind_submit'),
            onPressed: c.submitting.value ? null : c.submit,
            child: const Text('绑定并登录'),
          )),
        ]),
      ),
    );
  }
}
```
> UI 可用 `GlassPanel`/`AuraInput`/`AuraButton` 美化（与登录页一致），上面用原生组件保证测试稳定；实现者可替换为设计系统组件但保留相同 `Key`。

- [ ] **Step 4: 加路由常量**

`foundation_packages/sanyan_routes/lib/src/routes.dart` 的 `AppRoutes` 加：
```dart
static const String bindPhone = '/bind-phone';
```
（本期跳转用 `Get.to(()=>BindPhonePage(...))` 构造传参；路由常量备深链/未来用。）

- [ ] **Step 5: 运行确认通过 + commit**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter test business_packages/sanyan_auth/test/oauth/bind_phone_page_test.dart`
Expected: PASS (1 test)
```bash
cd /Users/aventador/code/3yan/app
git add business_packages/sanyan_auth/lib/src/oauth/bind_phone_page.dart foundation_packages/sanyan_routes/lib/src/routes.dart business_packages/sanyan_auth/test/oauth/bind_phone_page_test.dart
git commit -m "feat(auth): BindPhonePage 绑手机页 UI（手机号/验证码倒计时/密码/绑定）+ bindPhone 路由常量"
```

---

### Task 13: main.dart 装配 + fluwx 初始化 + barrel 导出

**背景：** 在 app 入口注入 oauth 依赖（providers/gateway/handler/controller），初始化 fluwx，导出公开类。装配是集成胶水，无业务分支——做编译 + analyze + 启动冒烟验证，不强求单测（属胶水代码）。

**Files:**
- Modify: `app/lib/main.dart`
- Modify: `business_packages/sanyan_auth/lib/sanyan_auth.dart`（barrel export OauthLoginController/BindPhonePage/各 provider/gateway/handler）
- Test: 无新单测（集成胶水）；靠 `fvm flutter analyze` + `fvm flutter test`（全量）+ 启动冒烟

- [ ] **Step 1: barrel 导出**

`business_packages/sanyan_auth/lib/sanyan_auth.dart` 加：
```dart
export 'src/oauth/oauth_login_controller.dart';
export 'src/oauth/bind_phone_page.dart';
export 'src/oauth/apple_auth_provider.dart';
export 'src/oauth/wechat_auth_provider.dart';
export 'src/oauth/oauth_gateway.dart';
export 'src/auth/login_success_handler.dart';
```

- [ ] **Step 2: main.dart 注入依赖**（在现有 `runApp`/Get 初始化处，`WsClient` 已 permanent put 之后）

```dart
// oauth 依赖装配
Get.put<OauthGateway>(OauthGatewayImpl(), permanent: true);
Get.put<LoginSuccessHandler>(LoginSuccessHandlerImpl(), permanent: true);
Get.put<OauthLoginController>(
  OauthLoginController(
    apple: AppleAuthProviderImpl(),
    wechat: WechatAuthProviderImpl(),
    gateway: Get.find<OauthGateway>(),
    successHandler: Get.find<LoginSuccessHandler>(),
  ),
  permanent: true,
);
```

- [ ] **Step 3: 初始化 fluwx**（在 `main()` 的 `WidgetsFlutterBinding.ensureInitialized()` 之后，需要 await）

```dart
final fluwx = Fluwx();
await fluwx.registerApi(
  appId: const String.fromEnvironment('WECHAT_APPID', defaultValue: ''),
  universalLink: const String.fromEnvironment('WECHAT_UNIVERSAL_LINK', defaultValue: ''),
);
```
> `WECHAT_APPID` / `WECHAT_UNIVERSAL_LINK` 由 `--dart-define` 注入（凭证门控；空值时微信登录不可用但 app 正常启动，Apple 不受影响）。import `package:fluwx/fluwx.dart`。

- [ ] **Step 4: 编译 + analyze + 全量测试**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter analyze`
Expected: No error（解决所有 import/类型问题）
Run: `cd /Users/aventador/code/3yan/app && fvm flutter test`
Expected: 全部 PASS（含前面所有 Task 的测试）

- [ ] **Step 5: Commit**

```bash
cd /Users/aventador/code/3yan/app
git add app/lib/main.dart business_packages/sanyan_auth/lib/sanyan_auth.dart
git commit -m "feat(app): 装配 oauth 依赖（gateway/handler/controller）+ 初始化 fluwx + barrel 导出"
```

---

## Phase E — 原生配置（凭证门控，TDD 例外：配置文件）

> 本阶段是 spec Phase 5 真机 e2e 的前置。`{{...}}` 标记处为**用户提供的凭证/标识**（非计划占位逻辑），实现者保留标记并在 commit message / 注释里写明"待填"，由用户在真机联调前填入。

### Task 14: iOS 原生配置（Apple Sign In + 微信回调）

**Files:**
- Create: `app/ios/Runner/Runner.entitlements`
- Modify: `app/ios/Runner/Info.plist`
- Modify: `app/ios/Runner/AppDelegate.swift`
- Modify: `app/ios/Runner.xcodeproj/project.pbxproj`（关联 entitlements + 开启 Sign In with Apple capability）

- [ ] **Step 1: 建 entitlements**

`app/ios/Runner/Runner.entitlements`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.developer.applesignin</key>
  <array><string>Default</string></array>
</dict>
</plist>
```

- [ ] **Step 2: Info.plist 加微信 URL Scheme + 查询白名单**（先 Read 现有 Info.plist，在 `<dict>` 内合并，勿覆盖现有 NSMicrophoneUsageDescription 等）

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLName</key><string>weixin</string>
    <key>CFBundleURLSchemes</key><array><string>{{WECHAT_APPID}}</string></array>
  </dict>
</array>
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>weixin</string>
  <string>weixinULAPI</string>
</array>
```

- [ ] **Step 3: AppDelegate 处理微信回调**（fluwx 4.x 多数情况自动处理 open url；若需手动，按 fluwx 文档在 `application(_:open:options:)` 与 `application(_:continue:restorationHandler:)` 转发。先按"fluwx 自动注册"跑，真机不回调再加手动转发）

- [ ] **Step 4: Xcode 工程关联 entitlements**

在 `project.pbxproj` 的 Debug/Release build settings 加 `CODE_SIGN_ENTITLEMENTS = Runner/Runner.entitlements;`，并在 Signing & Capabilities 添加 "Sign In with Apple"（需 Apple Developer 账号配置 App ID 能力——属上线 gate）。

- [ ] **Step 5: 验证编译**（无凭证也应能 pod install + 编译过；微信功能待凭证）

Run: `cd /Users/aventador/code/3yan/app/ios && pod install`
Expected: 成功（sign_in_with_apple/fluwx pods 装上）
> 真机 build 需开发者账号 + bundleId 配 Apple Sign In 能力，属凭证门控，本 Task 只保证配置文件结构正确 + pod install 通过。

- [ ] **Step 6: Commit**

```bash
cd /Users/aventador/code/3yan/app
git add ios/Runner/Runner.entitlements ios/Runner/Info.plist ios/Runner/AppDelegate.swift ios/Runner.xcodeproj/project.pbxproj ios/Podfile.lock
git commit -m "feat(ios): Apple Sign In entitlements + 微信 URL Scheme/查询白名单（WECHAT_APPID 待填）"
```

### Task 15: Android 原生配置（微信回调）

**Files:**
- Create: `app/android/app/src/main/kotlin/com/sanyan/sanyan_app/wxapi/WXEntryActivity.kt`
- Modify: `app/android/app/src/main/AndroidManifest.xml`

- [ ] **Step 1: 建 WXEntryActivity**（fluwx 要求包路径 `${applicationId}.wxapi.WXEntryActivity`，applicationId = `com.sanyan.sanyan_app`）

`app/android/app/src/main/kotlin/com/sanyan/sanyan_app/wxapi/WXEntryActivity.kt`:
```kotlin
package com.sanyan.sanyan_app.wxapi

import com.jarvanmo.fluwx.wxapi.FluwxWXEntryActivity

class WXEntryActivity : FluwxWXEntryActivity()
```
> 类全名以装好的 fluwx 版本提供的基类为准（4.x 为 `com.jarvanmo.fluwx.wxapi.FluwxWXEntryActivity`）；若包名不同，按 fluwx 文档调整 import。

- [ ] **Step 2: AndroidManifest 注册 + queries**（先 Read 现有 manifest，在 `<application>` 内加 activity，在根加 `<queries>`）

```xml
<activity
    android:name=".wxapi.WXEntryActivity"
    android:exported="true"
    android:taskAffinity="com.sanyan.sanyan_app"
    android:launchMode="singleTask" />
```
根元素内加：
```xml
<queries>
  <package android:name="com.tencent.mm" />
</queries>
```

- [ ] **Step 3: 验证编译**

Run: `cd /Users/aventador/code/3yan/app && fvm flutter build apk --debug` （或 `assembleDebug`）
Expected: 编译成功（微信功能待 appid + 签名，但编译应过）
> 若无法本机 build APK（环境/SDK），至少 `fvm flutter analyze` + gradle sync 不报结构错。

- [ ] **Step 4: Commit**

```bash
cd /Users/aventador/code/3yan/app
git add android/app/src/main/kotlin/com/sanyan/sanyan_app/wxapi/WXEntryActivity.kt android/app/src/main/AndroidManifest.xml
git commit -m "feat(android): 微信回调 WXEntryActivity + manifest queries（appid/签名待真机联调）"
```

---

## 最终验收（全部 Task 完成后）

- [ ] **全量测试**：`cd /Users/aventador/code/3yan/app && fvm flutter test`，全绿。
- [ ] **静态分析**：`fvm flutter analyze`，无 error。
- [ ] **派最终代码审查子代理**（superpowers:code-reviewer）整体审 Flutter oauth 改动：分支完整性、与服务端契约一致、规则合规（无 build 里 Get.put、复用 LoginSuccessHandler、工具类 static）、SDK 抽象隔离是否干净、原生配置凭证门控点是否标清。
- [ ] **凭证门控提醒**：真机 e2e 前需用户提供并填入：Apple App ID 开启 Sign In 能力 + Team ID；微信开放平台 AppID（填 iOS Info.plist `{{WECHAT_APPID}}`、`--dart-define=WECHAT_APPID=` + `WECHAT_UNIVERSAL_LINK=`）+ Android 应用签名。这些属 spec Phase 5。

---

## Self-Review（计划对照 spec 的自查）

**1. Spec 覆盖**（spec §Flutter 客户端组件 行140-142 + Phase 4）：
- `oauth_login_req.dart`(`OAuthLoginReq{provider,credential,nonce}` + Data) → Task 2/3 ✅
- `oauth_bind_phone_req.dart`(`OAuthBindPhoneReq{bindTicket,phone,code,password?}`) → Task 3 ✅
- `LoginController.loginWithApple()/loginWithWeChat()` → Task 9（`OauthLoginController`，独立控制器而非塞进 LoginController，更内聚）✅
- `_handleLoginSuccess(resp)` 抽出复用（写 token/userId + WsClient.connect + 跳 chat）→ Task 8（`LoginSuccessHandler`）✅
- needBind 跳绑手机号页带 bindTicket（构造函数传入，不走 Get.arguments）→ Task 9 `goToBindPhone` + Task 12 `BindPhonePage(bindTicket:)` ✅
- 微信 subscriber 在 onInit/onClose 注册注销 → Task 6 `WechatAuthProviderImpl` 在回调拿到结果后 `cancelable.cancel()`（一次性订阅，等价语义）✅
- needMergeAuth 处理 → Task 9/10 ✅
- Apple nonce：challenge 拿原始 nonce → sha256 传 SDK → 原始 nonce 传服务端 → Task 4/9 ✅
- 原生配置（Apple entitlements / 微信 scheme+WXEntryActivity）→ Task 14/15 ✅

**2. 占位扫描**：原生配置的 `{{WECHAT_APPID}}` 是**外部凭证注入点**（非逻辑占位），已显式标注门控且结构完整；Dart 侧无 TODO/占位。`oauthChallenge` 的 fromData 给了二选一明确判定依据（看服务端响应）。AuraButton 参数名给了"先 Read 确认"的明确动作。✅

**3. 类型一致性**：`OauthLoginData`（token/userId/needBind/bindTicket/needMergeAuth/loggedIn）跨 Task 2/9/10 一致；`SdkAuthResult`(Success/Cancelled/Failure) 跨 Task 5/6/9 一致；`OauthGateway`(challenge/login/bindPhone) 跨 Task 7/9/10 一致；`LoginSuccessHandler.handle({token,userId})` 跨 Task 8/9/10 一致；`provider` 值统一大写 `APPLE`/`WECHAT`。✅

**已知跨 Task 依赖**：Task 9 的 OauthLoginController import Task 12 的 BindPhonePage —— 已在 Task 9 注明"先建 bind_phone_page.dart 最小占位，Task 12 补全"，保证每步可编译。

---

## 实现完成记录（2026-06-01）

T1-T15 全部完成，每个 task 走 subagent-driven 双审（spec + 代码质量）。共 14 个 commit（68375c7..81baaca）+ C1 修复 1 个（bbf795a），在 app 子模块 master。Dart 侧 118 测试全绿、全量 analyze 干净。

**T15 结论修正**：fluwx 4.x/5.x 的 AndroidManifest 已通过 `<activity-alias ${applicationId}.wxapi.WXEntryActivity>` 自动提供微信回调入口 + `<package com.tencent.mm>` queries，**Android 侧无需手写任何原生代码**（计划原列的 WXEntryActivity.kt + manifest 改动是冗余、会导致 merged manifest 组件重复，已撤销）。iOS 仍需 T14 的 entitlements + Info.plist。

**C1（已解决）**：fluwx 4.6.3 与项目 Kotlin 2.2.20 不兼容（`ImagesIO.kt` 用了 Kotlin 2.x 移除的 `String.toLowerCase(Locale)` → Android 编译失败）。升级 fluwx 4.6.3→5.7.5 修复（5.x 对我们的 auth 路径完全向后兼容，零代码改动，仅版本号）；已实测 `:fluwx:compileDebugKotlin` 在 Kotlin 2.2.20 下编过。

**端到端契约已核**：客户端 OauthLoginData/req/AuthApi/Gateway 与服务端 `/oauth/{challenge,login,bind-phone}` 逐项对齐；needBind/needMergeAuth 三态服务端均返回非空 data 对象，客户端解析正常。

## Phase 5 Follow-up（真机联调前处理）

- **凭证填充（上线 gate）**：iOS Info.plist 的 `{{WECHAT_APPID}}` 占位、`--dart-define=WECHAT_APPID=` + `WECHAT_UNIVERSAL_LINK=`；Apple Developer 后台给 App ID 开 Sign In with Apple capability；微信开放平台填 iOS bundleId(`com.sanyan.sanyanApp`)/Android applicationId(`com.sanyan.sanyan_app`)+签名 MD5。
- **M2（二期）**：Apple 登录按钮在非 iOS 平台也显示，应加 `Platform.isIOS && SignInWithApple.isAvailable()` gate 仅 iOS 显示。
- **M3（二期）**：微信 `obtainCredential` 超时后 fluwx 订阅未取消（控制器层 `.timeout` 只解 UI 挂死）。二期把 cancelable 提到 provider 可取消，超时时通知 provider 注销订阅。
- **真机回调链路**：微信→WXEntryActivity→Flutter 回传 code、Apple Sign In 真机授权，需真机 + 凭证验证（单测用 fake 隔离了平台层，未覆盖）。
- **本机环境**：开发机 `JAVA_HOME` 指向失效的 openjdk@17 路径，Android 构建需修环境变量或用 Android Studio JBR。
