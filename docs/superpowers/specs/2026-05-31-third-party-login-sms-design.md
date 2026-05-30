# 第三方登录（Apple + 微信）+ 腾讯云真实短信 设计 Spec

- 日期：2026-05-31
- 状态：设计已与用户确认，待用户审查 → 转 writing-plans
- 范围：全栈（服务端 sanyan-user-core / foundation 层 + Flutter app）

---

## 1. 背景与目标

当前认证体系（已调研确认）：
- 注册：仅手机号 + 短信验证码 + 密码（+ 可选昵称）。
- 登录：仅手机号 + 密码。
- 短信：**stub**——`SmsCodeSendService.sendCode()` 只 `log.info` 打验证码，未接任何服务商，无防刷/限频。
- `UserEntity` 只有 id / phone(唯一) / password / nickname / avatar，无第三方身份字段，无任何第三方/OAuth 脚手架。

目标：
1. 新增 **Sign in with Apple** + **微信移动应用登录** 两种第三方登录（全栈）。
2. 把短信 stub 替换为 **腾讯云真实短信**（含防刷/限频/原子校验）。
3. 不碰支付。

## 2. 范围与非目标

**范围内：**
- 服务端：第三方身份校验、账号模型改造、登录/绑定接口、腾讯云短信集成。
- 客户端（Flutter）：Apple / 微信登录按钮 + SDK 接入 + 绑手机号页 + 调用新接口。

**非目标（本期不做）：**
- 支付（Apple 内购 / 微信支付）。
- 第三方资料（昵称/头像）回填——昵称走兜底生成，二期再考虑拉取。
- 设置页里的"已登录态主动绑定/解绑第三方"完整 UI——本期数据模型与接口为其预留，UI 二期。
- Apple 端不接 Android（Android 隐藏 Apple 按钮）。

## 3. 关键决策（已与用户确认）

| 决策 | 选择 | 理由 |
|---|---|---|
| 实现范围 | 全栈（服务端 + Flutter） | 第三方登录天生客户端+服务端配合 |
| 账号绑定策略 | **C：第三方登录后强制绑手机号**，手机号为 join key | 自动合并同人账号，避免账号碎片化（伴侣 app 账号含关系/记忆数据） |
| 合并安全 | 新手机号→直接建号；**已有账号→不静默合并**（需原密码或已登录态绑定） | 防账号接管（安全审查 S4） |
| 数据模型 | 独立 `user_auth_identity` 身份表（非 user 表平铺字段） | 一账号多身份天然 1:N，加 provider 不改表 |
| 短信 | **腾讯云真实短信**（替换 stub），接口抽象 + 配置选择 provider | 用户已有腾讯云；dev 用 stub，prod 用腾讯云 fail-fast |
| 凭证 | 用户后续提供（Apple Team/bundleId、微信 AppID/Secret、腾讯云签名/模板/密钥） | 代码 + 单测（mock）先就绪，真 e2e 等凭证 |

## 4. 账号模型与数据模型

### 4.1 user 表改动（Flyway，见 4.3）
- `phone`：`NOT NULL` → **可空**（第三方建号到绑手机号之间可能暂无 phone；建号成功时必带确定 phone，见 S7.3）。
- `password`：`NOT NULL` → **可空**（第三方账号无密码，只通过 Apple/微信登录，不塞假密码）。

### 4.2 新增 user_auth_identity 表
```
user_auth_identity(
  id            BIGSERIAL PK,
  user_id       BIGINT NOT NULL  FK→user(id),
  provider      VARCHAR  NOT NULL  CHECK(provider IN ('APPLE','WECHAT')),
  external_id   VARCHAR  NOT NULL  CHECK(length(trim(external_id)) > 0),  -- Apple sub / 微信 unionid
  raw_openid    VARCHAR  NULL,     -- 微信 openid，仅审计，不参与命中查询
  created_at    TIMESTAMP NOT NULL DEFAULT now(),
  UNIQUE(provider, external_id),   -- 一个第三方身份只能绑一个账号
  UNIQUE(provider, user_id)        -- 一个账号每 provider 至多一条身份（防并发挂两个 sub）
)
```
- 命中查询**只用 `(provider, external_id)`**；`raw_openid` 禁止参与 OR / 兜底。
- `provider` 值 `APPLE`/`WECHAT` 在 SQL CHECK、Java enum、DTO 三处用**常量单点定义**。

### 4.3 Flyway
- `bootstrap/src/main/resources/db/migration/V12__create_user_auth_identity.sql`（确认是唯一生效 migration 目录 + 下一个版本号）：建表 + `ALTER TABLE user ALTER COLUMN phone/password DROP NOT NULL`。

## 5. 流程设计

### 5.0 短信 provider 抽象（Phase 0）
- `SmsSender` 接口 + 两实现：`StubSmsSender`（log 打码，dev/test）、`TencentSmsSender`（真发）。
- 由配置 `sanyan.sms.provider: stub|tencent` 选择（dev 默认 stub，prod 设 tencent）。
- prod 选 tencent 但密钥/签名/模板缺失 → **启动 fail-fast**，绝不退回 stub（防生产误配把验证码打日志）。
- `SmsCodeSendService` 依赖 `SmsSender` 发送；生成/校验逻辑见 S5（SecureRandom + GETDEL 原子校验 + 失败计数锁定 + 发送限频）。

### 5.1 POST /api/auth/oauth/challenge（防重放 nonce）
- 服务端签发一次性 nonce，存 KvCache（短 TTL，如 5min），返回给客户端。
- Apple 登录前先取 nonce，SDK 授权时带上（Apple 会把 nonce 的 SHA256 写进 identityToken）。

### 5.2 POST /api/auth/oauth/login（两步式第一步）
入参 `{provider, credential, nonce}`：
1. 按 provider 选 verifier 校验 credential（Apple: identityToken；微信: code）→ 得 `external_id`（+微信 raw_openid）。**校验链见 S1/S2，提取 external_id 是校验链最后一步。**
2. 按 `(provider, external_id)` 查 `user_auth_identity`：
   - **命中** → 取 user → 签登录 JWT（`typ=ACCESS`，带 jti）→ 返回 `LoginData`（直接进 app）。
   - **未命中** → 签发 `bindTicket`（独立密钥，`typ=BIND`，载 provider+external_id+raw_openid，jti，TTL 10min）→ 返回 `{needBind:true, bindTicket}`。

### 5.3 POST /api/auth/oauth/bind-phone（两步式第二步，含合并安全）
入参 `{bindTicket, phone, code[, password]}`。**固定执行顺序（实现强约束，任何乱序都重开漏洞）：**
1. 验 bindTicket 签名（独立密钥）
2. `typ == BIND`
3. `exp`/`iat` ≤ 10min
4. **jti 消费锁** `KvCache.setIfAbsent("oauth:bindticket:used:"+jti, "1", 10min)`，已用则拒（必须在验短信/查库/写库**之前**）
5. 校验 ticket 内 provider/external_id 合法
6. SMS 可用性 gate（provider=tencent 且已配置；否则该路径不可达，见 5.0）
7. **原子** `getAndDelete` 验短信码（失败计数，见 S5）
8. 按 phone 查账号：
   - **phone 无账号** → 同事务建 user(phone, 无密码) + insert identity
   - **phone 已有账号** → **不静默合并**：该账号有密码则必须校验入参 `password`（`passwordEncoder.matches`）证明本人；无密码或不愿走密码则返回 `needMerge` 要求已登录态绑定。验证通过才 insert identity 挂上去
9. insert identity 用 **insert-or-recover**（依赖双唯一约束兜底，catch `DataIntegrityViolationException` 重查命中分支降级，不抛 500）
10. 签登录 JWT（`typ=ACCESS`，带 jti）返回 `LoginData`

## 6. 接口契约

| 接口 | 请求 | 响应 |
|---|---|---|
| `POST /api/auth/oauth/challenge` | `{}` | `{nonce}` |
| `POST /api/auth/oauth/login` | `{provider, credential, nonce}` | 命中：`LoginData{token,userId}`；未命中：`{needBind:true, bindTicket}` |
| `POST /api/auth/oauth/bind-phone` | `{bindTicket, phone, code, password?}` | 成功：`LoginData{token,userId}`；已有账号需本人证明：`{needMerge:true}` |
| `POST /api/auth/sms/send`（已存在，增强） | `{phone}` | 成功——内部走 SmsSender；加限频/验证码防刷 |

错误码：`UserErrCode` 新增（排 1007+，同步 `ERROR_CODE_REGISTRY.md`）：`WECHAT_UNIONID_MISSING`、`BIND_TICKET_USED`、`BIND_TICKET_INVALID`、`OAUTH_VERIFY_FAILED`、`SMS_CODE_INVALID`、`SMS_RATE_LIMITED`、`NEED_MERGE_AUTH` 等。

## 7. 服务端组件与集成点（sanyan-user-core / foundation 层）

- `web/AuthController`：新增 3 个 oauth 接口；`/sms/send` 增强。
- `internal/OauthLoginService`（独立 `@Transactional`，不塞进 UserLoginService）：编排校验→查身份→登录或发 bindTicket。
- `internal/OauthBindService`：bind-phone 编排（按 5.3 固定顺序）。
- `internal/AppleTokenVerifier`：Nimbus JOSE+JWT，`RemoteJWKSet` 拉 `https://appleid.apple.com/auth/keys`（单例 Bean，按 kid 选 key + 轮换 + 缓存）。
- `internal/WechatCodeVerifier`：用 AppID+AppSecret 调 `sns/oauth2/access_token` 换 openid/unionid（出站 HTTP 复用统一 client）。
- `internal/UserAuthIdentityEntity` + `UserAuthIdentityRepository.findByProviderAndExternalId`。
- `internal/sms/SmsSender`（接口）+ `StubSmsSender` + `TencentSmsSender`（tencentcloud-sdk-java sms 模块）。
- common-auth：`BindTicketUtil`（**独立 HMAC 密钥**，不复用 JwtUtil 签发逻辑）；`JwtUtil` 改造——登录 token 带 `typ=ACCESS`+jti，`parseUserId` 在 parse sub 前断言 `typ==ACCESS`；token 类型常量单点定义。
- common-cache `KvCache`：新增 `getAndDelete`（Redis GETDEL / Lua），用于短信码原子校验、nonce/jti 消费。
- common-web `GlobalExceptionHandler`：新增 `@ExceptionHandler(DataIntegrityViolationException.class)` → 业务冲突码（现落 onUnknown 成 500）。
- `UserEntity` phone/password 可空；`UserLoginService.login` 密码比对前显式判 `password==null` 抛明确错误（不依赖 BCrypt 隐式 false）；oauth 建号路径不复用 `UserRegisterService` 的 `phone.substring(7)` 昵称兜底（phone 可能 null）。
- 生产强制改掉 `JWT_SECRET` 默认值（`bootstrap/application.yml`），检测到默认值拒绝启动/告警。

### 配置新增（application.yml + /etc/3yan/3yan-server.env）
- `sanyan.apple.allowed-aud`（bundleId 白名单）
- `sanyan.oauth.bind-ticket.secret`（≥32字节，≠ jwt.secret）、`.ttl-minutes=10`
- 微信 `sanyan.wechat.app-id` / `app-secret`
- 腾讯云 `sanyan.sms.provider` / `sms.tencent.secret-id` / `secret-key` / `sdk-app-id` / `sign-name` / `template-id`

## 8. 客户端（Flutter app）组件

- `pubspec.yaml`（sanyan_auth）：加 `sign_in_with_apple`、`fluwx`。
- `main.dart`：启动 `Fluwx().registerApi(appId, universalLink)`（appId/universalLink 抽常量，禁硬编码）。
- `foundation_packages/sanyan_user/lib/src/api/req/`：`oauth_login_req.dart`（`OAuthLoginReq{provider,credential,nonce}` + `OAuthLoginData{token?,userId?,needBind,bindTicket?}`）、`oauth_bind_phone_req.dart`（`OAuthBindPhoneReq{bindTicket,phone,code,password?}`）。
- `auth_api.dart`（abstract + static）：`oauthChallenge()` / `oauthLogin()` / `oauthBindPhone()`；`sendSms` 复用。
- `LoginController`：`loginWithApple()` / `loginWithWeChat()`，命中走抽出的 `_handleLoginSuccess(resp)`（写 token/userId + WsClient.connect + 跳 chat），needBind 跳绑手机号页带 bindTicket（构造函数传入，不走 Get.arguments）；微信 subscriber 在 onInit/onClose 注册注销。
- 新增 `bind_phone/bind_phone_page.dart` + `bind_phone_controller.dart`，UI 复用 register 的手机号+验证码+倒计时；`Get.to(() => BindPhonePage(...))`，不进 AppRoutes。
- 登录页：Apple 按钮（iOS && `SignInWithApple.isAvailable`，用 `SignInWithAppleButton`）、微信按钮（`isWeChatInstalled`）；用户取消静默处理。

### iOS / Android 原生配置（待用户提供凭证后填）
- iOS：Runner 开 Sign in with Apple capability（entitlements）；`Info.plist` `LSApplicationQueriesSchemes` 加 `weixin/weixinULAPI/wechat/weixinURLParamsAPI`，`CFBundleURLTypes` 加微信 appId scheme；Associated Domains + `apple-app-site-association`（微信 Universal Links）。
- Android：`包名.wxapi.WXEntryActivity`（exported、singleTask）；ProGuard/R8 keep 微信 SDK + wxapi。

## 9. 安全要求清单（必须实现，安全审查 S1–S10）

> 铁律：提取 external_id 永远是校验链最后一步；任一步失败立即终止，不进写库。

**致命级：**
- **S1 Apple identityToken 完整验签 + 验全 claim**（固定序）：alg 白名单只 RS256（拒 none/HS*）→ JWKS 按 kid 验签 → `iss==https://appleid.apple.com` → `aud∈白名单`（禁信 token 反推 aud）→ exp/iat（±60s）→ nonce（比对 /challenge 下发 nonce 的 SHA256）→ 全过取 `sub`，绝不取客户端另传 id。
- **S2 微信只信服务端换取结果**：credential 只收 code；身份只取服务端 AppSecret 换来的响应；换取前 `setIfAbsent("wechat:code:"+code)` 一次性占位防重放；校验响应无 errcode；**unionid 非空断言，为空抛 WECHAT_UNIONID_MISSING，严禁 fallback openid**；命中只用 (provider,external_id)。
- **S3 bindTicket 与登录 JWT 强隔离**：独立 HMAC 密钥；`typ` 双隔离（ACCESS/BIND，常量单点）；access 解析前断言 typ==ACCESS、bind 解析断言 typ==BIND；bindTicket 不写 sub=userId，external_id 由服务端校验后填入（不透传客户端）；独立 10min TTL（不继承 30天）。
- **S4 绑手机号合并前验证归属 + 不静默合并**：stub 期 bind-phone 整链 feature flag 关/白名单；合并到已有账号默认禁用，需原密码或已登录态绑定；合并/建号前 verifyCode 验短信。（本期接真短信后，gate 改为"SMS provider 已配置"。）

**高危级：**
- **S5 短信防爆破 + 原子校验**：SecureRandom 生成；verifyCode 原子比对+删除（GETDEL/Lua）；失败计数达 5 锁定 + 作废 code（失败也计数）；sendCode 同 phone 60s 一条 + 单日上限；`/sms/send` 按 phone+IP 限频 + 图形/滑块验证码。
- **S6 bindTicket 一次性消费**：带 jti，bind-phone 第一步 setIfAbsent 消费锁（在验短信/查库/写库前），已用拒 BIND_TICKET_USED。
- **S7 唯一约束防竞态**：双唯一 (provider,external_id)+(provider,user_id)；禁 check-then-act（existsByPhone TOCTOU）→ insert-or-recover catch DataIntegrityViolation 降级；建 user+建 identity 同一事务，必带确定 phone（无孤儿）；GlobalExceptionHandler 补冲突码映射。
- **S8 oauth/login 防重放**：/oauth/challenge 发一次性 nonce；Apple 路径校验消费 nonce；微信靠 code 一次性消费兜底；微信发起带 state 回调校验防 CSRF。

**中危级：**
- **S9 JWT 可吊销**：access token 带 jti + 服务端吊销列表/token-version；合并、绑/解绑、改密后强制该用户已签 token 失效。
- **S10 externalId 命名空间隔离**：靠 S1.4 强校验 aud + 微信固化单一 appid（换取响应校验 appid 一致）钉死命名空间；provider 常量三处单点。

## 10. 外部前置条件 / 待用户提供的凭证

**Apple：** Developer 后台 App ID 开 Sign in with Apple；Xcode 加 capability/entitlements；Bundle ID 一致；记录 Team ID、bundleId（服务端 aud）。**App Store 4.8：有微信登录就必须给 Apple 登录（iOS），按钮符合 HIG。**

**微信：** 开放平台**企业认证**（￥300，unionid 硬前提，个人拿不到）；创建移动应用 + 审核 + 开微信登录，拿 AppID/AppSecret（Secret 仅后端）；iOS Universal Links + Android 包名/签名 MD5（debug+release 各登记）；APP 备案；未上架接口 100 次/天限制。**上线 gate：unionid 必须能返回，拿不到直接拒登录。**

**腾讯云短信：** 企业资质（验证码类需企业实名）；短信签名 + 正文模板（各过运营商审核，验证码类数小时~1工作日）；拿 SDKAppID/签名/模板ID/SecretId/SecretKey。

**待用户提供清单：** Apple Team ID + bundleId；微信 AppID + AppSecret + Universal Link 域名 + Android 包名+签名MD5；腾讯云 SDKAppID + 签名 + 模板ID + SecretId/SecretKey。

## 11. 测试策略（TDD）

- **服务端单测（Mockito）**：`AppleTokenVerifier`（用已知 kid 的测试 JWT + mock JWKS，覆盖 alg/iss/aud/exp/nonce 各失败分支）、`WechatCodeVerifier`（mock 微信 HTTP 响应：成功/errcode/unionid 缺失）、`BindTicketUtil`（签发/解析/typ 隔离/过期）、`OauthLoginService`（命中登录 / 未命中发 ticket）、`OauthBindService`（新号建/已有账号需密码/jti 已用/短信失败/竞态降级）、`SmsCodeSendService`（SecureRandom、原子校验、失败计数、限频）、`TencentSmsSender`（mock 腾讯云 SDK）。
- **Controller IT（@WebMvcTest）**：3 个 oauth 接口 + /sms/send。
- **真实 e2e**：等用户提供凭证后，真机 + dogfood（参照 plan4 dogfood 模式）。
- **客户端**：req toJson / OAuthLoginData needBind 分支解析、LoginController（mock AuthApi/SDK）、BindPhoneController（mocktail）。
- 所有外部依赖（Apple JWKS、微信 HTTP、腾讯云 SDK）测试中 mock；不依赖真凭证即可全绿。

## 12. 分期与上线 gate

- **Phase 0**：腾讯云真短信（SmsSender 抽象 + 防刷/限频/原子校验）——bind-phone 的依赖，先做。
- **Phase 1**：数据模型（migration + entity + repo）+ user 可空改造 + 现有登录 null 防护。
- **Phase 2**：common-auth/cache/web 基础改造（BindTicketUtil、JwtUtil typ/jti、KvCache getAndDelete、GlobalExceptionHandler）。
- **Phase 3**：服务端 verifier + OauthLoginService + OauthBindService + 接口。
- **Phase 4**：Flutter 客户端（按钮 + SDK + 绑手机号页 + API）。
- **Phase 5**：凭证就绪后真机 e2e + dogfood。

**上线 gate：** ① 腾讯云签名/模板过审 + 密钥配置（prod fail-fast）；② 微信 unionid 可返回；③ Apple aud 白名单配置正确；④ JWT_SECRET 改掉默认值；⑤ S1–S8 全部实现并测试通过。
