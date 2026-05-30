# 第三方登录（Apple + 微信）· 服务端 Implementation Plan（Plan B-Server）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development。Steps 用 checkbox（`- [ ]`）。

**Goal:** 服务端支持 Sign in with Apple + 微信登录：校验第三方凭证 → 已绑则登录、未绑则发 bindTicket → 绑手机号（C 策略：新号建/已有号需本人证明，不静默合并）→ 发本系统 JWT。

**Architecture:** 独立 `user_auth_identity` 身份表（一账号挂多身份）；两步式 `/oauth/challenge`→`/oauth/login`→`/oauth/bind-phone`；Apple 用 Nimbus JWKS 验 identityToken，微信用 RestClient 换 openid/unionid；bindTicket 独立密钥+typ=BIND+jti 一次性。落在 `sanyan-user-core` + `common-auth`/`common-web`。

**Tech Stack:** Java 21 + Spring Boot 3.5（jjwt 0.12.5、Nimbus JOSE+JWT[新增]、RestClient、Postgres+Flyway、Redis/KvCache、JUnit5+Mockito+Testcontainers）。

**对应 spec:** `docs/superpowers/specs/2026-05-31-third-party-login-sms-design.md`。依赖 **Plan A（腾讯云真短信，已合并）** —— bind-phone 复用 `SmsCodeSendService.verifyCode`。本计划是 **Plan B 的 server 子计划**；Flutter 子计划另写，二者通过本计划 Phase 4 定义的接口契约（spec §6）解耦、可并行。

**前置约束（注入每个实现子代理）：** server submodule master 上逐 task 提交、**不 push**；commit message 中文无 AI 署名；TDD 先 RED 后 GREEN；每 task 跑 `mvn -pl business_packages/sanyan-user-core test`（动 foundation 层的 task 加跑对应 foundation 包 + 一次 `mvn -pl bootstrap -am ... ` IT）；安全要求 S1-S10 见 spec §9，相关 task 内已落实。

---

## File Structure

**foundation 层：**
- `sanyan-common-auth/.../JwtUtil.java` — 改：generateToken 加 `typ=ACCESS`+`jti`；parseUserId 断言 typ==ACCESS。
- `sanyan-common-auth/.../TokenType.java` — 新建常量（ACCESS/BIND，单点定义）。
- `sanyan-common-auth/.../BindTicketUtil.java` — 新建：独立 HMAC 密钥签发/解析 bindTicket（typ=BIND、jti、10min）。
- `sanyan-common-web/.../GlobalExceptionHandler.java` — 改：加 `DataIntegrityViolationException` → 冲突码。
- 父 `pom.xml` + `sanyan-common-auth/pom.xml` — 加 nimbus-jose-jwt 依赖。

**user-core：**
- `internal/UserEntity.java` — phone/password 改 nullable。
- `internal/UserAuthIdentityEntity.java` + `UserAuthIdentityRepository.java` — 新建。
- `internal/UserLoginService.java` — 改：密码比对前判 password==null。
- `internal/oauth/Provider.java` — 新建 enum {APPLE, WECHAT}（与 DB CHECK 单点对齐）。
- `internal/oauth/ThirdPartyAuthVerifier.java` + `AppleTokenVerifier.java` + `WechatCodeVerifier.java` — 新建。
- `internal/oauth/OauthLoginService.java` + `OauthBindService.java` — 新建。
- `internal/oauth/OauthProperties.java` + `OauthConfig.java` — 新建（apple.allowed-aud、wechat.appid/secret、bind-ticket.secret/ttl）。
- `web/AuthController.java` — 改：加 `/oauth/challenge`、`/oauth/login`、`/oauth/bind-phone`。
- `web/OauthReq.java`（OauthLoginReq/OauthBindPhoneReq/响应 DTO）— 新建。
- `internal/UserErrCode.java` — 加 oauth 错误码。
- `bootstrap/.../db/migration/V12__create_user_auth_identity.sql` — 新建。

---

## Phase 1 — 数据模型

## Task 1：V12 migration + UserEntity 可空 + UserAuthIdentity 实体/Repo

**Files:**
- Create: `bootstrap/src/main/resources/db/migration/V12__create_user_auth_identity.sql`
- Modify: `business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/UserEntity.java`
- Create: `.../internal/UserAuthIdentityEntity.java`、`.../internal/UserAuthIdentityRepository.java`
- Create: `.../internal/oauth/Provider.java`
- Test: `.../internal/UserAuthIdentityRepositoryIT.java`

- [ ] **Step 1：写 V12 migration**

`V12__create_user_auth_identity.sql`：
```sql
-- 第三方登录身份表：一个 user 可挂多条身份（APPLE/WECHAT）
CREATE TABLE user_auth_identity (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT      NOT NULL REFERENCES users(id),
    provider    VARCHAR(16) NOT NULL CHECK (provider IN ('APPLE', 'WECHAT')),
    external_id VARCHAR(128) NOT NULL CHECK (length(trim(external_id)) > 0),
    raw_openid  VARCHAR(128),
    created_at  TIMESTAMP   NOT NULL DEFAULT now(),
    CONSTRAINT uk_provider_external UNIQUE (provider, external_id),
    CONSTRAINT uk_provider_user     UNIQUE (provider, user_id)
);
CREATE INDEX idx_uai_user ON user_auth_identity(user_id);

-- 第三方账号无手机号/密码：放开 NOT NULL
ALTER TABLE users ALTER COLUMN phone DROP NOT NULL;
ALTER TABLE users ALTER COLUMN password DROP NOT NULL;
```
> 落地前先 `psql` 或看现有 migration 确认 users 表名（是 `users` 还是 `user`——V3 等 migration 里确认）、phone/password 列名。本文件按 `users` 写，若实际是 `user` 则改表名。

- [ ] **Step 2：UserEntity phone/password 改可空**

`UserEntity.java`：`@Column(nullable = false, unique = true, length = 20) phone` → 去掉 `nullable = false`（保留 unique + length；unique 允许多 null）；`@Column(nullable = false) password` → 去掉 `nullable = false`。

- [ ] **Step 3：Provider enum**

`internal/oauth/Provider.java`：
```java
package com.sanyan.user.internal.oauth;
/** 第三方登录平台。值与 DB user_auth_identity.provider CHECK 单点对齐。 */
public enum Provider { APPLE, WECHAT }
```

- [ ] **Step 4：UserAuthIdentityEntity + Repository**

```java
package com.sanyan.user.internal;

import com.sanyan.user.internal.oauth.Provider;
import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import java.time.LocalDateTime;

@Entity @Data
@Table(name = "user_auth_identity",
       uniqueConstraints = {
         @UniqueConstraint(name = "uk_provider_external", columnNames = {"provider", "external_id"}),
         @UniqueConstraint(name = "uk_provider_user", columnNames = {"provider", "user_id"})
       })
public class UserAuthIdentityEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "user_id", nullable = false)
    private Long userId;
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 16)
    private Provider provider;
    @Column(name = "external_id", nullable = false, length = 128)
    private String externalId;
    @Column(name = "raw_openid", length = 128)
    private String rawOpenid;
    @CreationTimestamp
    private LocalDateTime createdAt;
}
```
```java
package com.sanyan.user.internal;

import com.sanyan.user.internal.oauth.Provider;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UserAuthIdentityRepository extends JpaRepository<UserAuthIdentityEntity, Long> {
    Optional<UserAuthIdentityEntity> findByProviderAndExternalId(Provider provider, String externalId);
}
```

- [ ] **Step 5：@DataJpaTest 验唯一约束 + 查询（先 RED）**

`UserAuthIdentityRepositoryIT.java`（参照现有 `@DataJpaTest` IT 风格，用 PostgresTestcontainerSupport——先 `rg -l "DataJpaTest|PostgresTestcontainerSupport" business_packages/sanyan-user-core/src/test` 看现有写法对齐）：
```java
// 关键用例：
// 1. save 后 findByProviderAndExternalId(APPLE, "sub123") 返回该行
// 2. 同 (provider, external_id) 再 save 第二条 → DataIntegrityViolationException
// 3. 同一 user_id 同 provider 第二条 → DataIntegrityViolationException（uk_provider_user）
```
（按现有 IT 基类写完整断言；若现有无 @DataJpaTest 基类，用 @SpringBootTest + PostgresTestcontainerSupport 跑 repo。）

- [ ] **Step 6：跑测试 RED → GREEN**

```bash
export JAVA_HOME="$(/usr/libexec/java_home)"
mvn -pl bootstrap -am install -DskipTests -q
mvn -pl business_packages/sanyan-user-core test -Dtest=UserAuthIdentityRepositoryIT 2>&1 | tail -20
```
RED：表不存在 / 实体未建。实现后 GREEN（migration 经 flyway 在 testcontainer 建表）。

- [ ] **Step 7：proactive 等回归 + 提交**

`mvn -pl business_packages/sanyan-user-core test`（确认 UserEntity 改可空不破坏现有 user 测试）。
```bash
git add bootstrap/src/main/resources/db/migration/V12__create_user_auth_identity.sql \
        business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/UserEntity.java \
        business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/UserAuthIdentityEntity.java \
        business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/UserAuthIdentityRepository.java \
        business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/oauth/Provider.java \
        business_packages/sanyan-user-core/src/test/java/com/sanyan/user/internal/UserAuthIdentityRepositoryIT.java
git commit -m "feat(oauth): user_auth_identity 身份表 + user phone/password 可空（第三方登录数据模型）"
```

## Task 2：UserLoginService null-password 防护

**Files:** Modify `.../internal/UserLoginService.java` + 其 Test。

- [ ] **Step 1：写失败测试** —— `login_throwsWhenAccountHasNoPassword`：mock `userRepository.findByPhone` 返回一个 password=null 的 UserEntity（第三方建的号），调 login(phone,任意密码) → 抛 BusinessException（不是 NPE、不是 BCrypt 隐式 false）。先看现有 UserLoginServiceTest 风格。
- [ ] **Step 2：RED**（现 login 直接拿 password 给 passwordEncoder.matches，password=null 会 NPE 或行为不明）。
- [ ] **Step 3：实现** —— login 在 `passwordEncoder.matches` 前加 `if (user.getPassword() == null) throw new BusinessException(UserErrCode.LOGIN_PASSWORD_NOT_SET);`（新错误码见 Task 8 错误码清单，或本 task 顺带加；文案"该账号未设置密码，请用第三方登录"）。
- [ ] **Step 4：GREEN + 全包回归 + 提交** `feat(oauth): UserLoginService 防护无密码账号（第三方建号）`。

---

## Phase 2 — common 层基础（token 隔离 + 异常映射 + 依赖）

## Task 3：JwtUtil 加 typ=ACCESS + jti，parseUserId 断言 typ

**Files:** Create `sanyan-common-auth/.../TokenType.java`；Modify `JwtUtil.java` + `JwtUtilTest.java`。

- [ ] **Step 1：TokenType 常量**
```java
package com.sanyan.common.auth;
/** JWT 用途隔离常量（登录 token vs bindTicket），单点定义。 */
public final class TokenType {
    private TokenType() {}
    public static final String CLAIM = "typ";
    public static final String ACCESS = "ACCESS";
    public static final String BIND = "BIND";
}
```

- [ ] **Step 2：写失败测试**（JwtUtilTest 加）：
  - `generateToken_carriesAccessTypAndJti`：生成 token 后用 jjwt 解析，断言 `typ`==ACCESS、`jti` 非空。
  - `parseUserId_rejectsNonAccessToken`：手工签一个 typ=BIND 的 token（同 key），`parseUserId` 抛异常（不返回 userId）。
  - 现有 `parseUserId` 正常用例保持绿。

- [ ] **Step 3：RED**（typ/jti 未加、parseUserId 不校验 typ）。

- [ ] **Step 4：实现**
  - `generateToken`：`.subject(...).claim(TokenType.CLAIM, TokenType.ACCESS).id(UUID.randomUUID().toString())`（jjwt 0.12 `.id()` = jti）`.issuedAt/.expiration/.signWith`。
  - `parseUserId`：解析 claims 后，`if (!TokenType.ACCESS.equals(claims.get(TokenType.CLAIM))) throw new BusinessException(CommonErrCode.TOKEN_INVALID);`（用现有 token 失效码；确认 CommonErrCode 有 TOKEN_EXPIRED/INVALID 类，无则用最接近的）再 `Long.parseLong(subject)`。
  > 兼容性：现有已签发的旧 token 无 typ claim → parseUserId 会拒。**这会让所有存量登录态失效**（用户需重登）。spec S9 接受"敏感变更后 token 失效"。在 commit body 注明此影响；若需平滑，可临时允许 typ 缺省视为 ACCESS（加注释 TODO 一个发布周期后移除）——**实现者按此做平滑：`typ == null || ACCESS 才放行`，注释说明过渡期**。

- [ ] **Step 5：GREEN + common-auth 全包 + 提交** `feat(auth): JwtUtil 登录 token 加 typ=ACCESS+jti，parseUserId 校验 typ（防 bindTicket 冒用）`。

## Task 4：BindTicketUtil（独立密钥 + typ=BIND + jti + 10min）

**Files:** Create `sanyan-common-auth/.../BindTicketUtil.java` + Test。

- [ ] **Step 1：写失败测试** `BindTicketUtilTest`：
  - `issue_then_parse_roundTrips`：issue(provider="APPLE", externalId="sub1", rawOpenid=null) → parse 拿回 provider/externalId + 一个 jti。
  - `parse_rejectsExpired`：用极短 TTL 或伪造过期 ticket → parse 抛。
  - `parse_rejectsWrongTyp`：用 JwtUtil 的 ACCESS token 喂给 BindTicketUtil.parse → 抛（typ != BIND）。
  - `parse_rejectsTamperedSignature` / 用不同密钥签的 → 抛。

- [ ] **Step 2：RED** → **Step 3：实现**
```java
package com.sanyan.common.auth;
// 独立 HMAC 密钥（sanyan.oauth.bind-ticket.secret，≥32B，≠ jwt.secret），typ=BIND，jti，TTL 10min。
// jjwt 0.12 API：Jwts.builder().claim("provider",..).claim("externalId",..).claim("rawOpenid",..)
//   .claim(TokenType.CLAIM, TokenType.BIND).id(jti).issuedAt.expiration.signWith(bindKey)
// parse：verifyWith(bindKey).parseSignedClaims；断言 typ==BIND；返回一个 record BindTicketPayload(provider, externalId, rawOpenid, jti)。
```
构造注入 `@Value("${sanyan.oauth.bind-ticket.secret}")` + `@Value("${sanyan.oauth.bind-ticket.ttl-minutes:10}")`。提供 `String issue(BindTicketPayload p)` 和 `BindTicketPayload parse(String ticket)`。`BindTicketPayload` 是 record（含 jti）。

- [ ] **Step 4：GREEN + 提交** `feat(auth): BindTicketUtil 独立密钥短期 bindTicket（typ=BIND+jti，10min）`。

## Task 5：GlobalExceptionHandler 加 DataIntegrityViolationException

**Files:** Modify `sanyan-common-web/.../GlobalExceptionHandler.java` + Test。

- [ ] **Step 1：失败测试**（GlobalExceptionHandlerTest，或 @WebMvcTest 触发）：抛 `DataIntegrityViolationException` 的接口 → 返回 BaseResp.failed(冲突码) 而非 500。
- [ ] **Step 2：RED**（现落 onUnknown → 500）。
- [ ] **Step 3：实现**：加 `@ExceptionHandler(DataIntegrityViolationException.class)` → `BaseResp.failed(CommonErrCode.CONFLICT.getCode(), "数据冲突，请重试")`（CommonErrCode 加 CONFLICT 码，若无；或复用最接近的通用冲突码——先看 CommonErrCode 现有码）。log.warn 记录。
- [ ] **Step 4：GREEN + 提交** `feat(web): GlobalExceptionHandler 处理 DataIntegrityViolation→冲突码（替代 500）`。

## Task 6：引入 Nimbus JOSE+JWT 依赖

**Files:** Modify 父 `pom.xml` + `sanyan-common-auth/pom.xml`。

- [ ] **Step 1：父 pom** properties + dependencyManagement 加 `com.nimbusds:nimbus-jose-jwt`（用 Maven Central 最新稳定版，写计划时约 `9.40+`，落地确认真实版本）。
- [ ] **Step 2：common-auth pom** 声明（不带 version）。
- [ ] **Step 3：`mvn -pl foundation_packages/sanyan-common-auth -am dependency:resolve -q` 确认拉到** + 确认关键类 `com.nimbusds.jwt.SignedJWT`、`com.nimbusds.jose.jwk.source.RemoteJWKSet`、`com.nimbusds.jose.proc.JWSVerificationKeySelector`、`com.nimbusds.jwt.proc.DefaultJWTProcessor` 可见（解 jar 看路径，记下供 Task 7）。
- [ ] **Step 4：提交** `build(oauth): 引入 nimbus-jose-jwt（Apple identityToken JWKS 验签）`。

---

## Phase 3 — 第三方凭证校验器

## Task 7：AppleTokenVerifier（Nimbus JWKS + S1 全 claim 校验）

**Files:** Create `.../internal/oauth/ThirdPartyAuthVerifier.java`（接口）、`AppleTokenVerifier.java`；Modify OauthProperties（apple.allowed-aud）；Test `AppleTokenVerifierTest`。

- [ ] **Step 1：接口 + Properties 雏形**
```java
// ThirdPartyAuthVerifier：record VerifiedIdentity(String externalId, String rawOpenid)
// AppleTokenVerifier.verify(String identityToken, String expectedNonce) -> VerifiedIdentity(sub, null)
```
OauthProperties 加 `apple.allowedAud: List<String>`（bundleId 白名单）。

- [ ] **Step 2：写失败测试**（用一对自生成 RSA key 模拟 Apple，注入可重写的 JWKS source）：
  - `verify_ok`：合法 token（iss=https://appleid.apple.com、aud∈白名单、未过期、nonce 的 SHA256 匹配、RS256）→ 返回 sub。
  - 各失败分支**逐个**：alg=none/HS256 拒；aud 不在白名单拒；iss 错拒；过期拒；nonce 不匹配拒；签名验不过拒。
  > 为可测：AppleTokenVerifier 把"JWKS 来源/processor"抽成 protected 方法或构造注入 `JWKSource`，测试注入本地公钥；S1 校验逻辑（alg 白名单 RS256、iss、aud、exp、nonce SHA256）在 verify 里。

- [ ] **Step 3：RED** → **Step 4：实现（S1 固定顺序）**
  用 Nimbus `DefaultJWTProcessor` + `RemoteJWKSet`（`https://appleid.apple.com/auth/keys`，单例缓存）+ `JWSVerificationKeySelector(RS256, jwkSource)`（**只允许 RS256**）；processor.process 验签拿 claims；再断言 `iss==https://appleid.apple.com`、`aud ∈ allowedAud`、exp（Nimbus 默认验）、`sha256Hex(expectedNonce) == claims.nonce`。全过返回 `VerifiedIdentity(sub, null)`，否则抛 `BusinessException(UserErrCode.OAUTH_VERIFY_FAILED)`。RemoteJWKSet 做成单例 Bean（@Configuration 提供），构造注入。

- [ ] **Step 5：GREEN + 提交** `feat(oauth): AppleTokenVerifier 用 Nimbus JWKS 验 identityToken（S1：alg白名单/iss/aud/exp/nonce）`。

## Task 8：WechatCodeVerifier（RestClient 换 openid/unionid + S2）+ oauth 错误码

**Files:** Create `WechatCodeVerifier.java`；Modify OauthProperties（wechat.appid/secret）、UserErrCode（oauth 码）+ ERROR_CODE_REGISTRY；Test `WechatCodeVerifierTest`。

- [ ] **Step 1：UserErrCode 加 oauth 码**（接现有最大 1008 顺延，同步登记表）：
  `OAUTH_VERIFY_FAILED(1009,"第三方登录校验失败")`、`WECHAT_UNIONID_MISSING(1010,"微信账号缺少 unionid，无法登录")`、`BIND_TICKET_INVALID(1011,"绑定会话无效或已过期")`、`BIND_TICKET_USED(1012,"绑定会话已使用")`、`NEED_MERGE_AUTH(1013,"该手机号已注册，请验证账号本人")`、`LOGIN_PASSWORD_NOT_SET(1014,"该账号未设密码，请用第三方登录")`。
  （Task 2 若已加 LOGIN_PASSWORD_NOT_SET 则此处不重复；按实际顺延。）

- [ ] **Step 2：写失败测试**（mock RestClient 或抽 protected 方法返回微信响应 JSON）：
  - `verify_ok`：微信返回 {openid, unionid} 无 errcode → 返回 VerifiedIdentity(unionid, openid)。
  - `verify_rejectsWhenErrcode`：返回含 errcode → 抛 OAUTH_VERIFY_FAILED。
  - `verify_rejectsWhenUnionidMissing`：返回有 openid 无 unionid → 抛 WECHAT_UNIONID_MISSING（**严禁 fallback openid**）。

- [ ] **Step 3：RED** → **Step 4：实现（S2）**
  用 `HttpClientFactory.newClient("https://api.weixin.qq.com", ...)` GET `/sns/oauth2/access_token?appid=&secret=&code=&grant_type=authorization_code`；解析响应：有 errcode 抛；`unionid` 空（StringUtils.hasText）抛 WECHAT_UNIONID_MISSING；否则 `VerifiedIdentity(unionid, openid)`。`external_id=unionid`，`raw_openid=openid`。access_token 用完即弃不入库。code 一次性消费（KvCache.setIfAbsent("wechat:code:"+code) 占位，已存在拒）放在调用方 OauthLoginService 或此处——放此处更内聚。

- [ ] **Step 5：GREEN + 提交** `feat(oauth): WechatCodeVerifier 换 openid/unionid（S2：只信服务端结果、unionid 必需、code 一次性）+ oauth 错误码`。

---

## Phase 4 — 编排 + 接口（契约层，Flutter 子计划依赖此）

## Task 9：OauthProperties + OauthConfig + nonce challenge 接口

**Files:** Create `OauthProperties.java`/`OauthConfig.java`（若 Task 7/8 已建则补全字段）；Modify AuthController（/oauth/challenge）；Create web/OauthReq DTO；application.yml + deploy.sh env。

- [ ] **Step 1：OauthProperties 全字段** `@ConfigurationProperties("sanyan.oauth")`：`apple.allowedAud(List)`、`wechat.appid/secret`、`bindTicket.secret/ttlMinutes`。OauthConfig `@EnableConfigurationProperties`（对齐项目模式）。application.yml `sanyan.oauth` 段 + env 占位；deploy.sh 补 env key（待凭证）。
- [ ] **Step 2：/oauth/challenge**（先写 @WebMvcTest 失败测试）：`POST /api/auth/oauth/challenge` → 服务端生成一次性 nonce（`UUID`），存 KvCache `oauth:nonce:{nonce}`=1 TTL 5min，返回 `{nonce}`。
- [ ] **Step 3：RED→实现→GREEN + 提交** `feat(oauth): OauthProperties/Config + /oauth/challenge 下发一次性 nonce（S8 防重放）`。

## Task 10：OauthLoginService + /oauth/login

**Files:** Create `OauthLoginService.java`；Modify AuthController（/oauth/login）、OauthReq；Test（Service 单测 + Controller IT）。

- [ ] **Step 1：DTO** `OauthLoginReq{ @NotNull Provider provider; @NotBlank String credential; String nonce }`；响应 `OauthLoginData{ Long userId; String token; String nickname; String avatar; boolean needBind; String bindTicket }`（命中登录填 user 字段、needBind=false；未绑则 needBind=true+bindTicket）。
- [ ] **Step 2：写失败测试** OauthLoginServiceTest（mock verifier/repo/jwt/bindTicketUtil）：
  - `login_existingIdentity_returnsLoginData`：verifier 返回 externalId，identityRepo.findByProviderAndExternalId 命中 → 取 user → JWT → needBind=false。
  - `login_newIdentity_returnsBindTicket`：未命中 → 签 bindTicket → needBind=true。
  - Apple 路径校验消费 nonce（mock KvCache.getAndDelete("oauth:nonce:"+nonce) 非空才放行；空抛）。
- [ ] **Step 3：RED→实现→GREEN**：编排——按 provider 选 verifier（Apple 先校验消费 nonce）→ VerifiedIdentity → findByProviderAndExternalId 命中发 JWT、未命中签 bindTicket（externalId/rawOpenid 进 ticket，服务端校验后填，不透传客户端原值）。`@Transactional(readOnly=true)` 或无事务（只读+可能不写）。
- [ ] **Step 4：Controller IT**（@WebMvcTest mock OauthLoginService）验 /oauth/login 两种返回。
- [ ] **Step 5：提交** `feat(oauth): OauthLoginService + /oauth/login（命中登录/未绑发 bindTicket，Apple 消费 nonce）`。

## Task 11：OauthBindService + /oauth/bind-phone（10 步固定顺序 + 合并安全 S4/S6/S7）

**Files:** Create `OauthBindService.java`；Modify AuthController（/oauth/bind-phone）、OauthReq；Test（Service 单测 + Controller IT）。

- [ ] **Step 1：DTO** `OauthBindPhoneReq{ @NotBlank String bindTicket; @NotBlank @Pattern(^1\d{10}$) String phone; @NotBlank String code; String password }`；响应复用 OauthLoginData（成功填 token；已有账号需本人证明时 needMerge）。新增响应位 `boolean needMerge`（或复用 needBind 区分——建议独立 needMerge）。
- [ ] **Step 2：写失败测试** OauthBindServiceTest（mock bindTicketUtil/KvCache/smsCodeSendService/userRepo/identityRepo/jwt/passwordEncoder），覆盖固定 10 步关键分支：
  - `bind_newPhone_createsUserAndIdentity_returnsJwt`：phone 无账号 → 建 user(phone,无密码)+identity → JWT。
  - `bind_existingPhoneWithPassword_requiresPassword`：phone 有账号且有密码、未传 password → 返回 needMerge（不静默合并）；传对的 password → 校验通过挂 identity。
  - `bind_rejectsUsedTicket`：jti 已消费（KvCache.setIfAbsent 返回 false）→ 抛 BIND_TICKET_USED（在验短信/写库前）。
  - `bind_rejectsBadSmsCode`：verifyCode false → 失败（不建号）。
  - `bind_rejectsInvalidTicket`：bindTicketUtil.parse 抛 → BIND_TICKET_INVALID。
- [ ] **Step 3：RED→实现（严格按 spec §5.3 的 10 步固定顺序）**
  ①parse bindTicket（独立密钥，BindTicketUtil）②typ==BIND（parse 内已校验）③exp/iat≤10min（parse 内）④jti 消费锁 `KvCache.setIfAbsent("oauth:bindticket:used:"+jti,"1",10min)` 失败→BIND_TICKET_USED ⑤ticket 内 provider/externalId 合法 ⑥SMS 可用性（provider 配置就绪，stub 期由 SmsCodeSendService 决定，本期真短信已接，直接用）⑦`smsCodeSendService.verifyCode(phone,code)` 原子验 false→失败 ⑧按 phone 查账号：无→建；有→有密码必须 `passwordEncoder.matches(password,...)` 否则 needMerge ⑨同 `@Transactional` insert user(如需)+identity，**insert-or-recover**：catch DataIntegrityViolationException 重查 (provider,externalId) 命中则当登录降级 ⑩签 ACCESS JWT 返回。建号 nickname 兜底不依赖 phone.substring（phone 此时有值，但用统一兜底如"用户"+随机/"微信用户"）。
- [ ] **Step 4：Controller IT** 验 /oauth/bind-phone（成功 / needMerge / 票据失效）。
- [ ] **Step 5：提交** `feat(oauth): OauthBindService + /oauth/bind-phone（10步顺序+合并安全 S4/jti一次性 S6/唯一约束兜底 S7）`。

---

## Phase 5 — 端到端 IT（mock 外部，不连真凭证）

## Task 12：oauth 全链路 SpringBoot IT（Testcontainer + mock verifier）

**Files:** Create `bootstrap/.../OauthFlowIT.java`。

- [ ] **Step 1：写 IT**（@SpringBootTest + PostgresTestcontainerSupport，参照 ProactiveFlowE2EIT/AuthControllerIT；@MockBean AppleTokenVerifier/WechatCodeVerifier 避免连真 Apple/微信，或注入 stub 返回固定 externalId）：
  - 全新身份 login→needBind→challenge nonce→bind-phone(新手机号 stub 短信码)→拿 JWT→该 JWT 能过 parseUserId。
  - 同身份再 login → 直接命中登录（needBind=false）。
  - 已有手机号账号 + 第三方绑定 → needMerge（无密码）/密码正确则合并。
- [ ] **Step 2：跑通 + 提交** `test(oauth): 第三方登录全链路 IT（mock 外部凭证，testcontainer）`。

---

## Self-Review

**1. Spec 覆盖：** 数据模型(T1)✓ user 可空(T1)✓ null-password 防护(T2)✓ JwtUtil typ/jti(T3)✓ BindTicketUtil(T4)✓ DataIntegrity 异常(T5)✓ Nimbus(T6)✓ Apple S1(T7)✓ 微信 S2(T8)✓ nonce S8(T9)✓ login(T10)✓ bind-phone 10步+S4/S6/S7(T11)✓ 全链路 IT(T12)✓。S9（JWT 吊销）部分由 T3 typ 校验覆盖，完整 jti 吊销列表本期 defer（spec §9 中危，记 follow-up）。S10 命名空间靠 T7 aud 白名单 + T8 单 appid + Provider 常量(T1)。
**2. 占位扫描：** Nimbus/版本号、users 表名、CommonErrCode 现有冲突/token 码、现有 IT 基类——均标"落地确认"，非逻辑空缺。
**3. 类型一致：** Provider(T1) 贯穿 entity/verifier/DTO；VerifiedIdentity(T7) 被 Apple/Wechat verifier 共用；BindTicketPayload(T4) 被 T10 签/T11 解；OauthLoginData(T10) 被 T11 复用。
**follow-up（不阻塞本计划）：** S9 完整 jti 吊销列表；真机 e2e 待凭证；手机号日志脱敏（Plan A 遗留）。
