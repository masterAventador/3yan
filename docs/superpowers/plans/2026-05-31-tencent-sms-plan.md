# 腾讯云真实短信 + 短信安全加固 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把短信验证码从 stub（只打日志）替换为腾讯云真实发送，并补齐防刷/限频/原子校验，dev 仍用 stub、生产用腾讯云且缺密钥 fail-fast。

**Architecture:** 抽 `SmsSender` 接口 + `StubSmsSender`（dev/test）/ `TencentSmsSender`（prod）两实现，由 `sanyan.sms.provider` 配置选择激活哪个 Bean。`SmsCodeSendService` 依赖 `SmsSender` 发送，验证码生成改 `SecureRandom`、校验改 Redis 原子 `getAndDelete` + 失败计数锁定，发送加每手机号 60s/单日限频。

**Tech Stack:** Java 21 + Spring Boot 3.x，tencentcloud-sdk-java-sms，Redis（KvCache/StringRedisTemplate），JUnit5 + Mockito。

**对应 spec:** `docs/superpowers/specs/2026-05-31-third-party-login-sms-design.md`（§5.0、§9 S5）。这是第三方登录计划 B 的前置依赖（bind-phone 依赖真短信）。

**前置约束（注入子代理）：** 只提交不 push（等用户指令）；commit message 中文、无 AI 署名；每 task 跑 `mvn -pl business_packages/sanyan-user-core test`（动到 foundation 层的 task 加跑对应 foundation 包测试），TDD 先 RED 再 GREEN。

---

## File Structure（先锁定decomposition）

- `foundation_packages/sanyan-common-cache/src/main/java/com/sanyan/common/cache/KvCache.java` — 加 `getAndDelete`（原子 GETDEL）。
- `business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/sms/SmsSender.java` — 新建接口。
- `.../internal/sms/StubSmsSender.java` — 新建（日志实现，dev/test）。
- `.../internal/sms/TencentSmsSender.java` — 新建（腾讯云实现，prod）。
- `.../internal/sms/SmsProperties.java` — 新建 `@ConfigurationProperties("sanyan.sms")`。
- `.../internal/SmsCodeSendService.java` — 改造（注入 SmsSender、SecureRandom、原子校验、失败计数、限频）。
- `.../internal/UserErrCode.java` — 加短信相关错误码。
- `bootstrap/src/main/resources/application.yml` — 加 `sanyan.sms` 配置。
- `bootstrap/pom.xml` 或 `business_packages/sanyan-user-core/pom.xml` + 父 `pom.xml` — 加 tencentcloud-sdk-java-sms 依赖 + dependencyManagement 固定版本。
- 测试：各 `*Test.java` 同包路径。

---

## Task 1：KvCache 加原子 getAndDelete

**Files:**
- Modify: `foundation_packages/sanyan-common-cache/src/main/java/com/sanyan/common/cache/KvCache.java`
- Test: `foundation_packages/sanyan-common-cache/src/test/java/com/sanyan/common/cache/KvCacheTest.java`（无则新建）

- [ ] **Step 1：写失败测试**

先 `rg -n "class KvCacheTest" foundation_packages/sanyan-common-cache/src/test` 看是否已有测试类；有则加方法，无则新建下面文件：

```java
package com.sanyan.common.cache;

import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class KvCacheTest {

    @Test
    void getAndDelete_returnsValueAndConsumesKey() {
        StringRedisTemplate redis = mock(StringRedisTemplate.class);
        ValueOperations<String, String> ops = mock(ValueOperations.class);
        when(redis.opsForValue()).thenReturn(ops);
        when(ops.getAndDelete("k")).thenReturn("v");

        KvCache cache = new KvCache(redis);
        String got = cache.getAndDelete("k");

        assertThat(got).isEqualTo("v");
        verify(ops).getAndDelete("k");
    }

    @Test
    void getAndDelete_returnsNullWhenAbsent() {
        StringRedisTemplate redis = mock(StringRedisTemplate.class);
        ValueOperations<String, String> ops = mock(ValueOperations.class);
        when(redis.opsForValue()).thenReturn(ops);
        when(ops.getAndDelete("missing")).thenReturn(null);

        assertThat(new KvCache(redis).getAndDelete("missing")).isNull();
    }
}
```

- [ ] **Step 2：跑测试确认 RED**

Run: `mvn -pl foundation_packages/sanyan-common-cache test -Dtest=KvCacheTest`
Expected: 编译失败 / FAIL —— `getAndDelete` 方法不存在。

- [ ] **Step 3：实现 getAndDelete**

在 `KvCache` 加（紧挨 `get` 方法）：

```java
/**
 * 原子「取出并删除」（Redis GETDEL）。用于一次性消费的值（短信验证码、nonce、bindTicket jti），
 * 避免「get 后再 delete」非原子导致并发读到同一值都通过。
 *
 * @return key 的旧值；不存在返回 null。
 */
public String getAndDelete(String key) {
    return redis.opsForValue().getAndDelete(key);
}
```

- [ ] **Step 4：跑测试确认 GREEN**

Run: `mvn -pl foundation_packages/sanyan-common-cache test -Dtest=KvCacheTest`
Expected: PASS。

- [ ] **Step 5：提交**

```bash
cd /Users/aventador/code/3yan/server
git add foundation_packages/sanyan-common-cache/src/main/java/com/sanyan/common/cache/KvCache.java \
        foundation_packages/sanyan-common-cache/src/test/java/com/sanyan/common/cache/KvCacheTest.java
git commit -m "feat(cache): KvCache 加原子 getAndDelete（GETDEL）"
```

---

## Task 2：SmsProperties 配置类

**Files:**
- Create: `business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/sms/SmsProperties.java`
- Modify: `bootstrap/src/main/resources/application.yml`

- [ ] **Step 1：新建 SmsProperties**

```java
package com.sanyan.user.internal.sms;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

/** 短信配置。provider=stub(dev 默认)|tencent(prod)。 */
@Data
@Component
@ConfigurationProperties("sanyan.sms")
public class SmsProperties {
    /** stub | tencent */
    private String provider = "stub";
    private Tencent tencent = new Tencent();

    @Data
    public static class Tencent {
        private String secretId;
        private String secretKey;
        private String sdkAppId;
        private String signName;
        private String templateId;
        /** 地域，默认广州 */
        private String region = "ap-guangzhou";
    }
}
```

- [ ] **Step 2：application.yml 加配置**

在 `sanyan:` 段下（jwt 段附近）加：

```yaml
  sms:
    provider: ${SMS_PROVIDER:stub}        # dev 默认 stub；prod 在 env 设 tencent
    tencent:
      secret-id: ${SMS_TENCENT_SECRET_ID:}
      secret-key: ${SMS_TENCENT_SECRET_KEY:}
      sdk-app-id: ${SMS_TENCENT_SDK_APP_ID:}
      sign-name: ${SMS_TENCENT_SIGN_NAME:}
      template-id: ${SMS_TENCENT_TEMPLATE_ID:}
      region: ${SMS_TENCENT_REGION:ap-guangzhou}
```

- [ ] **Step 3：编译确认**

Run: `mvn -pl business_packages/sanyan-user-core -am compile -q`
Expected: 无错误（@ConfigurationProperties 仅需 spring-boot-configuration-processor 可选，无则忽略提示）。

- [ ] **Step 4：提交**

```bash
git add business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/sms/SmsProperties.java \
        bootstrap/src/main/resources/application.yml
git commit -m "feat(sms): SmsProperties 配置类 + application.yml sanyan.sms 段"
```

---

## Task 3：SmsSender 接口 + StubSmsSender

**Files:**
- Create: `.../internal/sms/SmsSender.java`
- Create: `.../internal/sms/StubSmsSender.java`
- Test: `.../internal/sms/StubSmsSenderTest.java`

- [ ] **Step 1：写失败测试**

```java
package com.sanyan.user.internal.sms;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThatCode;

class StubSmsSenderTest {
    @Test
    void send_doesNotThrow() {
        SmsSender sender = new StubSmsSender();
        assertThatCode(() -> sender.send("13800138000", "123456")).doesNotThrowAnyException();
    }
}
```

- [ ] **Step 2：跑测试确认 RED**

Run: `mvn -pl business_packages/sanyan-user-core test -Dtest=StubSmsSenderTest`
Expected: 编译失败（SmsSender/StubSmsSender 不存在）。

- [ ] **Step 3：实现接口 + Stub**

`SmsSender.java`：
```java
package com.sanyan.user.internal.sms;

/** 短信发送抽象。验证码内容由调用方生成，发送渠道由实现决定。 */
public interface SmsSender {
    /** 发送验证码短信；失败抛 BusinessException。 */
    void send(String phone, String code);
}
```

`StubSmsSender.java`：
```java
package com.sanyan.user.internal.sms;

import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

/** dev/test 用：只打日志不真发。由 sanyan.sms.provider=stub（默认）激活。 */
@Slf4j
@Component
@ConditionalOnProperty(name = "sanyan.sms.provider", havingValue = "stub", matchIfMissing = true)
public class StubSmsSender implements SmsSender {
    @Override
    public void send(String phone, String code) {
        log.info("[stub-sms] 验证码 [{}]: {}（未真发，dev/test 模式）", phone, code);
    }
}
```

- [ ] **Step 4：跑测试确认 GREEN**

Run: `mvn -pl business_packages/sanyan-user-core test -Dtest=StubSmsSenderTest`
Expected: PASS。

- [ ] **Step 5：提交**

```bash
git add business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/sms/SmsSender.java \
        business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/sms/StubSmsSender.java \
        business_packages/sanyan-user-core/src/test/java/com/sanyan/user/internal/sms/StubSmsSenderTest.java
git commit -m "feat(sms): SmsSender 接口 + StubSmsSender（日志实现，dev/test 默认）"
```

---

## Task 4：加 tencentcloud-sdk-java-sms 依赖

**Files:**
- Modify: `pom.xml`（父，dependencyManagement 固定版本）
- Modify: `business_packages/sanyan-user-core/pom.xml`（声明依赖）

- [ ] **Step 1：父 pom dependencyManagement 加版本**

先 `rg -n "<dependencyManagement>|<properties>" pom.xml` 找到位置。在 `<properties>` 加版本号（用最新稳定版，写计划时约 `3.1.1100`，实现时 `mvn versions:display-dependency-updates` 或查 Maven Central 确认最新）：
```xml
<tencentcloud-sms.version>3.1.1100</tencentcloud-sms.version>
```
在 `<dependencyManagement><dependencies>` 加：
```xml
<dependency>
  <groupId>com.tencentcloudapi</groupId>
  <artifactId>tencentcloud-sdk-java-sms</artifactId>
  <version>${tencentcloud-sms.version}</version>
</dependency>
```

- [ ] **Step 2：user-core pom 声明依赖**

`business_packages/sanyan-user-core/pom.xml` 的 `<dependencies>` 加（不带 version，走父管理）：
```xml
<dependency>
  <groupId>com.tencentcloudapi</groupId>
  <artifactId>tencentcloud-sdk-java-sms</artifactId>
</dependency>
```

- [ ] **Step 3：拉依赖确认**

Run: `mvn -pl business_packages/sanyan-user-core -am dependency:resolve -q 2>&1 | tail -5`
Expected: 成功下载，无 version 报错。

- [ ] **Step 4：提交**

```bash
git add pom.xml business_packages/sanyan-user-core/pom.xml
git commit -m "build(sms): 引入 tencentcloud-sdk-java-sms 依赖"
```

---

## Task 5：TencentSmsSender 实现

**Files:**
- Create: `.../internal/sms/TencentSmsSender.java`
- Test: `.../internal/sms/TencentSmsSenderTest.java`

- [ ] **Step 1：写失败测试**

腾讯云 `SmsClient.SendSms` 直接 new client 难 mock，故 TencentSmsSender 把「建 client」抽成可重写的 protected 方法，测试时注入 mock client。测试：

```java
package com.sanyan.user.internal.sms;

import com.sanyan.common.error.BusinessException;
import com.tencentcloudapi.sms.v20210111.SmsClient;
import com.tencentcloudapi.sms.v20210111.models.SendSmsRequest;
import com.tencentcloudapi.sms.v20210111.models.SendSmsResponse;
import com.tencentcloudapi.sms.v20210111.models.SendStatus;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class TencentSmsSenderTest {

    private SmsProperties props() {
        SmsProperties p = new SmsProperties();
        p.setProvider("tencent");
        SmsProperties.Tencent t = p.getTencent();
        t.setSecretId("sid"); t.setSecretKey("skey"); t.setSdkAppId("1400000000");
        t.setSignName("三言"); t.setTemplateId("123456");
        return p;
    }

    @Test
    void send_buildsRequestAndSucceedsWhenStatusOk() throws Exception {
        SmsClient client = mock(SmsClient.class);
        SendStatus ok = new SendStatus(); ok.setCode("Ok");
        SendSmsResponse resp = new SendSmsResponse(); resp.setSendStatusSet(new SendStatus[]{ok});
        when(client.SendSms(any(SendSmsRequest.class))).thenReturn(resp);

        TencentSmsSender sender = new TencentSmsSender(props()) {
            @Override protected SmsClient client() { return client; }
        };
        sender.send("13800138000", "123456");

        verify(client).SendSms(argThat(req ->
            req.getSmsSdkAppId().equals("1400000000")
            && req.getSignName().equals("三言")
            && req.getTemplateId().equals("123456")
            && req.getPhoneNumberSet()[0].equals("+8613800138000")
            && req.getTemplateParamSet()[0].equals("123456")));
    }

    @Test
    void send_throwsWhenStatusNotOk() throws Exception {
        SmsClient client = mock(SmsClient.class);
        SendStatus fail = new SendStatus(); fail.setCode("LimitExceeded"); fail.setMessage("超频");
        SendSmsResponse resp = new SendSmsResponse(); resp.setSendStatusSet(new SendStatus[]{fail});
        when(client.SendSms(any(SendSmsRequest.class))).thenReturn(resp);

        TencentSmsSender sender = new TencentSmsSender(props()) {
            @Override protected SmsClient client() { return client; }
        };
        assertThatThrownBy(() -> sender.send("13800138000", "123456"))
            .isInstanceOf(BusinessException.class);
    }
}
```

- [ ] **Step 2：跑测试确认 RED**

Run: `mvn -pl business_packages/sanyan-user-core test -Dtest=TencentSmsSenderTest`
Expected: 编译失败（TencentSmsSender 不存在）。

- [ ] **Step 3：实现 TencentSmsSender**

> 注：腾讯云 SDK 包名/类名以实际拉到的版本为准（`com.tencentcloudapi.sms.v20210111.*`、`com.tencentcloudapi.common.Credential`、`SendSmsRequest` 字段 `SmsSdkAppId/SignName/TemplateId/PhoneNumberSet/TemplateParamSet`）。手机号需 E.164 格式（国内加 `+86`）。

```java
package com.sanyan.user.internal.sms;

import com.sanyan.common.error.BusinessException;
import com.sanyan.user.internal.UserErrCode;
import com.tencentcloudapi.common.Credential;
import com.tencentcloudapi.sms.v20210111.SmsClient;
import com.tencentcloudapi.sms.v20210111.models.SendSmsRequest;
import com.tencentcloudapi.sms.v20210111.models.SendSmsResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

/** 腾讯云真实短信。由 sanyan.sms.provider=tencent 激活（prod）。 */
@Slf4j
@Component
@ConditionalOnProperty(name = "sanyan.sms.provider", havingValue = "tencent")
public class TencentSmsSender implements SmsSender {

    private final SmsProperties props;

    public TencentSmsSender(SmsProperties props) {
        this.props = props;
        SmsProperties.Tencent t = props.getTencent();
        // prod fail-fast：选了 tencent 但密钥/签名/模板缺失，启动即报错，绝不退回 stub
        if (isBlank(t.getSecretId()) || isBlank(t.getSecretKey()) || isBlank(t.getSdkAppId())
                || isBlank(t.getSignName()) || isBlank(t.getTemplateId())) {
            throw new IllegalStateException(
                "sanyan.sms.provider=tencent 但腾讯云短信配置不完整（secretId/secretKey/sdkAppId/signName/templateId 必填）");
        }
    }

    private static boolean isBlank(String s) { return s == null || s.isBlank(); }

    /** 抽出来便于测试注入 mock client。 */
    protected SmsClient client() {
        SmsProperties.Tencent t = props.getTencent();
        return new SmsClient(new Credential(t.getSecretId(), t.getSecretKey()), t.getRegion());
    }

    @Override
    public void send(String phone, String code) {
        SmsProperties.Tencent t = props.getTencent();
        SendSmsRequest req = new SendSmsRequest();
        req.setSmsSdkAppId(t.getSdkAppId());
        req.setSignName(t.getSignName());
        req.setTemplateId(t.getTemplateId());
        req.setPhoneNumberSet(new String[]{toE164(phone)});
        req.setTemplateParamSet(new String[]{code});
        try {
            SendSmsResponse resp = client().SendSms(req);
            if (resp.getSendStatusSet() == null || resp.getSendStatusSet().length == 0
                    || !"Ok".equals(resp.getSendStatusSet()[0].getCode())) {
                String msg = resp.getSendStatusSet() != null && resp.getSendStatusSet().length > 0
                        ? resp.getSendStatusSet()[0].getMessage() : "无返回";
                log.warn("腾讯云短信发送失败 phone={} msg={}", phone, msg);
                throw new BusinessException(UserErrCode.SMS_SEND_FAILED);
            }
        } catch (BusinessException e) {
            throw e;
        } catch (Exception e) {
            log.warn("腾讯云短信调用异常 phone={}", phone, e);
            throw new BusinessException(UserErrCode.SMS_SEND_FAILED);
        }
    }

    private static String toE164(String phone) {
        return phone.startsWith("+") ? phone : "+86" + phone;
    }
}
```

> 本 Task 依赖 Task 6 引入的 `UserErrCode.SMS_SEND_FAILED`。**先做 Task 6 的 Step「加错误码」再编译本类**，或本 Task 临时把错误码也加上（见 Task 6 错误码清单）。实现顺序建议：先把 Task 6 的 UserErrCode 改动做了，再回来编译 Task 5。

- [ ] **Step 4：跑测试确认 GREEN**

Run: `mvn -pl business_packages/sanyan-user-core test -Dtest=TencentSmsSenderTest`
Expected: PASS。

- [ ] **Step 5：提交**

```bash
git add business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/sms/TencentSmsSender.java \
        business_packages/sanyan-user-core/src/test/java/com/sanyan/user/internal/sms/TencentSmsSenderTest.java
git commit -m "feat(sms): TencentSmsSender 腾讯云真实发送（provider=tencent 激活，prod 缺配置 fail-fast）"
```

---

## Task 6：UserErrCode 加短信错误码

**Files:**
- Modify: `business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/UserErrCode.java`
- Modify: `foundation_packages/sanyan-common-error/ERROR_CODE_REGISTRY.md`（若存在；无则跳过并在 commit body 说明）

- [ ] **Step 1：先看现有码段**

Run: `rg -n "enum UserErrCode|[0-9]{4}," business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/UserErrCode.java`
确认现有最大 code，新增码接在后面（user 域 1000-1999）。

- [ ] **Step 2：加错误码**

在 `UserErrCode` enum 末尾加（code 取现有最大值 +1 起，下面 1010-1012 为示意，按实际顺延）：
```java
SMS_SEND_FAILED(1010, "短信发送失败，请稍后重试"),
SMS_CODE_INVALID(1011, "验证码错误或已过期"),
SMS_RATE_LIMITED(1012, "验证码发送过于频繁，请稍后再试"),
```

- [ ] **Step 3：编译确认**

Run: `mvn -pl business_packages/sanyan-user-core -am compile -q`
Expected: 无错误。

- [ ] **Step 4：同步 ERROR_CODE_REGISTRY.md（若有）**

`rg -l "ERROR_CODE_REGISTRY" foundation_packages/ docs/ 2>/dev/null`；找到则在 user 域段补这三条码。

- [ ] **Step 5：提交**

```bash
git add business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/UserErrCode.java
# 若改了 registry 一并 add
git commit -m "feat(sms): UserErrCode 加 SMS_SEND_FAILED/CODE_INVALID/RATE_LIMITED"
```

---

## Task 7：SmsCodeSendService 改造（SmsSender + SecureRandom + 原子校验 + 失败计数）

**Files:**
- Modify: `.../internal/SmsCodeSendService.java`
- Modify: `.../internal/SmsCodeSendServiceTest.java`（现有测试，构造签名变了）

- [ ] **Step 1：改失败测试（先看现有测试）**

`rg -n "new SmsCodeSendService|sendCode|verifyCode" business_packages/sanyan-user-core/src/test/java/com/sanyan/user/internal/SmsCodeSendServiceTest.java` 看现有用例。把构造改为注入 mock `SmsSender`，并新增原子校验/失败计数用例。完整替换为：

```java
package com.sanyan.user.internal;

import com.sanyan.common.cache.KvCache;
import com.sanyan.common.error.BusinessException;
import com.sanyan.user.internal.sms.SmsSender;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class SmsCodeSendServiceTest {

    private KvCache kvCache;
    private SmsSender smsSender;
    private SmsCodeSendService service;

    @BeforeEach
    void setup() {
        kvCache = mock(KvCache.class);
        smsSender = mock(SmsSender.class);
        service = new SmsCodeSendService(kvCache, smsSender);
    }

    @Test
    void sendCode_storesAndSendsSixDigit() {
        when(kvCache.setIfAbsent(startsWith("sms:rate:"), any(), any())).thenReturn(true);
        when(kvCache.increment(startsWith("sms:daily:"), any())).thenReturn(1L);

        service.sendCode("13800138000");

        ArgumentCaptor<String> codeCap = ArgumentCaptor.forClass(String.class);
        verify(smsSender).send(eq("13800138000"), codeCap.capture());
        assertThat(codeCap.getValue()).matches("\\d{6}");
        verify(kvCache).set(eq("sms:code:13800138000"), eq(codeCap.getValue()), any());
    }

    @Test
    void sendCode_throwsWhenRateLimited() {
        when(kvCache.setIfAbsent(startsWith("sms:rate:"), any(), any())).thenReturn(false); // 60s 内已发

        assertThatThrownBy(() -> service.sendCode("13800138000")).isInstanceOf(BusinessException.class);
        verify(smsSender, never()).send(any(), any());
    }

    @Test
    void verifyCode_atomicSuccess() {
        when(kvCache.getAndDelete("sms:code:13800138000")).thenReturn("123456");
        assertThat(service.verifyCode("13800138000", "123456")).isTrue();
        verify(kvCache).getAndDelete("sms:code:13800138000"); // 原子取删，不再 get+delete
    }

    @Test
    void verifyCode_wrongCode_countsFailureAndReturnsFalse() {
        when(kvCache.getAndDelete("sms:code:13800138000")).thenReturn("999999");
        when(kvCache.increment(startsWith("sms:fail:"), any())).thenReturn(1L);
        assertThat(service.verifyCode("13800138000", "123456")).isFalse();
        verify(kvCache).increment(startsWith("sms:fail:"), any()); // 失败计数
    }

    @Test
    void verifyCode_lockedAfterTooManyFailures() {
        when(kvCache.increment(startsWith("sms:fail:"), any())).thenReturn(6L); // 已超 5 次
        assertThatThrownBy(() -> service.verifyCode("13800138000", "123456"))
            .isInstanceOf(BusinessException.class);
        verify(kvCache, never()).getAndDelete(any()); // 锁定后不再校验
    }
}
```

- [ ] **Step 2：跑测试确认 RED**

Run: `mvn -pl business_packages/sanyan-user-core test -Dtest=SmsCodeSendServiceTest`
Expected: 编译失败 / FAIL（构造签名变了、新行为未实现）。

- [ ] **Step 3：改造 SmsCodeSendService**

```java
package com.sanyan.user.internal;

import com.sanyan.common.cache.KvCache;
import com.sanyan.common.error.BusinessException;
import com.sanyan.user.internal.sms.SmsSender;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.security.SecureRandom;
import java.time.Duration;

@Slf4j
@Service
@RequiredArgsConstructor
public class SmsCodeSendService {

    private final KvCache kvCache;
    private final SmsSender smsSender;

    private static final String CODE_PREFIX = "sms:code:";
    private static final String RATE_PREFIX = "sms:rate:";   // 60s 一条节流
    private static final String DAILY_PREFIX = "sms:daily:"; // 单日上限
    private static final String FAIL_PREFIX = "sms:fail:";   // 校验失败计数
    private static final Duration CODE_TTL = Duration.ofMinutes(5);
    private static final Duration RATE_TTL = Duration.ofSeconds(60);
    private static final Duration DAILY_TTL = Duration.ofHours(24);
    private static final Duration FAIL_TTL = Duration.ofMinutes(10);
    private static final int DAILY_CAP = 10;
    private static final int MAX_FAIL = 5;
    private static final SecureRandom RANDOM = new SecureRandom();

    public void sendCode(String phone) {
        // 60s 一条
        if (!kvCache.setIfAbsent(RATE_PREFIX + phone, "1", RATE_TTL)) {
            throw new BusinessException(UserErrCode.SMS_RATE_LIMITED);
        }
        // 单日上限
        long today = kvCache.increment(DAILY_PREFIX + phone, DAILY_TTL);
        if (today > DAILY_CAP) {
            throw new BusinessException(UserErrCode.SMS_RATE_LIMITED);
        }
        String code = String.format("%06d", RANDOM.nextInt(1_000_000));
        kvCache.set(CODE_PREFIX + phone, code, CODE_TTL);
        smsSender.send(phone, code); // 真发失败抛 BusinessException
    }

    public boolean verifyCode(String phone, String code) {
        // 失败次数锁定（先记 +1，达上限直接拒，避免无限重试）
        long fails = kvCache.increment(FAIL_PREFIX + phone, FAIL_TTL);
        if (fails > MAX_FAIL) {
            throw new BusinessException(UserErrCode.SMS_RATE_LIMITED);
        }
        String stored = kvCache.getAndDelete(CODE_PREFIX + phone); // 原子取删
        if (stored != null && stored.equals(code)) {
            kvCache.delete(FAIL_PREFIX + phone); // 成功清失败计数
            return true;
        }
        return false;
    }
}
```

> 注：`verifyCode` 把失败计数提前到校验前 +1，是为了「失败也计数」（原实现失败不消费 = 无限试）；成功路径清计数。`getAndDelete` 命中即消费，杜绝并发重放。

- [ ] **Step 4：跑测试确认 GREEN**

Run: `mvn -pl business_packages/sanyan-user-core test -Dtest=SmsCodeSendServiceTest`
Expected: PASS。

- [ ] **Step 5：proactive 等其他用到 verifyCode/sendCode 的地方回归**

Run: `mvn -pl business_packages/sanyan-user-core test 2>&1 | grep -E "Tests run|BUILD" | tail -3`
Expected: 全绿（UserRegisterService 用 verifyCode，构造未变，不受影响）。

- [ ] **Step 6：提交**

```bash
git add business_packages/sanyan-user-core/src/main/java/com/sanyan/user/internal/SmsCodeSendService.java \
        business_packages/sanyan-user-core/src/test/java/com/sanyan/user/internal/SmsCodeSendServiceTest.java
git commit -m "feat(sms): SmsCodeSendService 走 SmsSender + SecureRandom + 原子校验 + 失败计数 + 限频"
```

---

## Task 8：全量回归 + provider 切换冒烟

**Files:** 无新代码，验证 + 文档。

- [ ] **Step 1：user-core 全包测试**

Run: `mvn -pl business_packages/sanyan-user-core test 2>&1 | grep -E "Tests run|BUILD" | tail -3`
Expected: BUILD SUCCESS，Failures 0。

- [ ] **Step 2：常量/全量编译**

Run: `cd /Users/aventador/code/3yan/server && export JAVA_HOME="$(/usr/libexec/java_home)" && mvn -q -DskipTests package 2>&1 | tail -5`
Expected: BUILD SUCCESS（确认 tencent SDK 依赖打进 fat jar、@ConditionalOnProperty 不破坏启动）。

- [ ] **Step 3：provider 切换确认（不连真腾讯云）**

确认逻辑：dev（provider 缺省=stub）→ StubSmsSender 生效；若设 `SMS_PROVIDER=tencent` 但 5 个密钥缺失 → TencentSmsSender 构造抛 IllegalStateException 启动失败（fail-fast）。可用一个 `@SpringBootTest` 加 `@TestPropertySource(properties="sanyan.sms.provider=tencent")` 且不给密钥，断言上下文启动失败（可选，时间紧可仅人工 review 构造逻辑）。

- [ ] **Step 4（文档）：deploy.sh / env 清单更新提示**

在 `deploy.sh` 注释的「生产必需 key」清单补：`SMS_PROVIDER=tencent`、`SMS_TENCENT_SECRET_ID/SECRET_KEY/SDK_APP_ID/SIGN_NAME/TEMPLATE_ID`（待签名/模板过审后填）。提交：
```bash
git add deploy.sh
git commit -m "docs(deploy): env 清单补腾讯云短信 key（待签名模板过审后填）"
```

---

## Self-Review（实施前自查）

**1. Spec 覆盖（对 §5.0 + §9 S5）：** SmsSender 抽象 + provider 选择 + prod fail-fast（Task 2/3/5/8）✓；SecureRandom（Task 7）✓；getAndDelete 原子校验（Task 1/7）✓；失败计数锁定（Task 7）✓；发送 60s/单日限频（Task 7）✓。**未覆盖：`/sms/send` 的 phone+IP 维度限频 + 图形/滑块验证码**——图形验证码涉及客户端，属独立较大特性，本计划只做服务端 per-phone 限频；IP 维度限频 + 图形验证码列为后续单独任务（在计划末尾备注，不阻塞短信真发上线）。

**2. 占位扫描：** tencentcloud SDK 版本号 `3.1.1100` 标注了「实现时确认最新」；错误码 1010-1012 标注「按现有最大值顺延」；SDK 包名/字段标注「以实际版本为准」——这些是必要的实现时确认点，非逻辑占位。无 TODO/TBD 逻辑空缺。

**3. 类型一致性：** `SmsSender.send(String,String)` 在 Stub/Tencent/SmsCodeSendService 三处签名一致 ✓；`SmsCodeSendService(KvCache, SmsSender)` 构造在测试与实现一致 ✓；`KvCache.getAndDelete(String):String` 在 Task1 定义、Task7 使用一致 ✓；`UserErrCode.SMS_*` 在 Task6 定义、Task5/7 使用一致（Task5 依赖提示已注明先做 Task6 错误码）✓。

**后续单独任务（不在本计划）：** `/sms/send` phone+IP 维度限频 + 图形/滑块验证码（需客户端配合）。
