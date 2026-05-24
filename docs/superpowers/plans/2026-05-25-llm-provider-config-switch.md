# LLM Provider 配置化切换 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task.

**Goal:** 把 LLM provider 选择从代码硬编码改成 yml 配置驱动，所有环境默认 DeepSeek 接管 USER_FACING + BACKGROUND（解决 2026-05-25 dogfood 发现的豆包 character 模型把 system role 时间标签当 assistant 回复输出的 bug）。豆包代码保留待用。

**Architecture:** 每个 Adapter 加 `@Value("${sanyan.llm.<provider>.task-types:default}")` 配置，`supports()` 比对配置值 vs `taskType.name()`。LLMProviderRouter 不动（已有 0 匹配抛错 / 多匹配 warn 的逻辑刚好覆盖配置态）。

**Tech Stack:** Java 17 + Spring Boot 3 + JUnit 5 + Mockito + ReflectionTestUtils。改动局限在 `business_packages/sanyan-llm-core` 单包 + `bootstrap/src/main/resources/application*.yml`。

**Spec 上下文：** 见 `docs/superpowers/specs/2026-05-24-chat-time-awareness-design.md` §"风险与缓解"——dogfood 验证 2 失败触发本次方案调整。

---

## File Structure

| 文件 | 操作 | 职责 |
|---|---|---|
| `server/business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/DoubaoAdapter.java` | 修改 | `supports()` 改为读 `@Value("${sanyan.llm.doubao.task-types:}")` 字段（默认空） |
| `server/business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/DeepSeekAdapter.java` | 修改 | `supports()` 改为读 `@Value("${sanyan.llm.deepseek.task-types:USER_FACING,BACKGROUND}")` 字段（默认两个都接） |
| `server/business_packages/sanyan-llm-core/src/test/java/com/sanyan/llm/internal/DoubaoAdapterTest.java` | 修改 | 改原 `supports_shouldReturnTrueForUserFacingOnly` + 新增 task-types 解析用例 |
| `server/business_packages/sanyan-llm-core/src/test/java/com/sanyan/llm/internal/DeepSeekAdapterTest.java` | 修改 | 同上对称 |
| `server/bootstrap/src/main/resources/application.yml`（如果存在） | 修改 / 创建 | 显式声明默认值：doubao 空、deepseek 两个都接 |
| `server/bootstrap/src/main/resources/application-prod.yml`（如果有） | 检查 | 不需要 override（默认值就是生产想要的） |

---

## Task 1: 配置化 supports() + Adapter 测试

**Files:**
- Modify: `DoubaoAdapter.java`, `DeepSeekAdapter.java`
- Test: `DoubaoAdapterTest.java`, `DeepSeekAdapterTest.java`

- [ ] **Step 1.1: 写失败测试 — DoubaoAdapter supports 配置驱动**

修改 `DoubaoAdapterTest.java` 原有用例 `supports_shouldReturnTrueForUserFacingOnly`，**删掉它**（与新方案不兼容），改为下面 4 个用例：

```java
    @Test
    void supports_shouldReturnFalseForAllWhenTaskTypesIsEmpty() {
        ReflectionTestUtils.setField(adapter, "taskTypes", "");
        assertThat(adapter.supports(LlmTaskType.USER_FACING)).isFalse();
        assertThat(adapter.supports(LlmTaskType.BACKGROUND)).isFalse();
    }

    @Test
    void supports_shouldReturnTrueOnlyForConfiguredTaskType() {
        ReflectionTestUtils.setField(adapter, "taskTypes", "USER_FACING");
        assertThat(adapter.supports(LlmTaskType.USER_FACING)).isTrue();
        assertThat(adapter.supports(LlmTaskType.BACKGROUND)).isFalse();
    }

    @Test
    void supports_shouldReturnTrueForAllConfiguredTaskTypes() {
        ReflectionTestUtils.setField(adapter, "taskTypes", "USER_FACING,BACKGROUND");
        assertThat(adapter.supports(LlmTaskType.USER_FACING)).isTrue();
        assertThat(adapter.supports(LlmTaskType.BACKGROUND)).isTrue();
    }

    @Test
    void supports_shouldTolerateWhitespaceAndCaseInTaskTypesConfig() {
        ReflectionTestUtils.setField(adapter, "taskTypes", " user_facing ,  BACKGROUND ");
        assertThat(adapter.supports(LlmTaskType.USER_FACING)).isTrue();
        assertThat(adapter.supports(LlmTaskType.BACKGROUND)).isTrue();
    }
```

如果 `ReflectionTestUtils` 还没 import，加 `import org.springframework.test.util.ReflectionTestUtils;`。

对 `DeepSeekAdapterTest.java` 做完全对称的修改（删原 `supports_shouldReturnTrueForBackgroundOnly`，加 4 个新用例）。

- [ ] **Step 1.2: 运行测试，确认新用例失败**

```bash
cd /Users/aventador/code/3yan/server
mvn test -pl business_packages/sanyan-llm-core -Dtest='DoubaoAdapterTest,DeepSeekAdapterTest'
```

Expected: 8 个新用例全部失败（`taskTypes` 字段不存在），其他用例正常。

- [ ] **Step 1.3: 实现 — DoubaoAdapter supports 配置化**

修改 `DoubaoAdapter.java`：

1. 加 imports：
```java
import java.util.Arrays;
import java.util.Set;
import java.util.stream.Collectors;
```

2. **不动构造器签名**，在类内（建议放 `restClient` field 下方、构造器上方）加一个 field-level `@Value` 注入：

```java
    /**
     * 配置驱动的 task type 列表（comma-separated，case-insensitive，trim 空格）。
     * 默认空 = supports() 全返回 false = 豆包不接任何任务（2026-05-25 切 DeepSeek 后的默认行为）。
     * <p>Field-level @Value 而非构造器参数：
     * <ul>
     *   <li>不动现有构造器签名（最小侵入）</li>
     *   <li>非 final，便于 ReflectionTestUtils.setField 在测试里动态切值</li>
     * </ul>
     */
    @Value("${sanyan.llm.doubao.task-types:}")
    private String taskTypes;
```

3. 完全重写 `supports` 方法（替换当前 line 111-114）：

```java
    @Override
    public boolean supports(LlmTaskType taskType) {
        if (taskTypes == null || taskTypes.isBlank()) {
            return false;
        }
        Set<String> configured = Arrays.stream(taskTypes.split(","))
                .map(s -> s.trim().toUpperCase())
                .filter(s -> !s.isEmpty())
                .collect(Collectors.toSet());
        return configured.contains(taskType.name());
    }
```

- [ ] **Step 1.4: 实现 — DeepSeekAdapter supports 配置化**

对 `DeepSeekAdapter.java` 做完全对称的修改：

1. 加同样的 3 个 imports（Arrays / Set / Collectors）
2. **不动构造器签名**，加 field-level `@Value`，**注意 default 是 `USER_FACING,BACKGROUND`**（与 doubao 默认空形成互斥的默认行为，"所有环境都用 DeepSeek"）：

```java
    @Value("${sanyan.llm.deepseek.task-types:USER_FACING,BACKGROUND}")
    private String taskTypes;
```

3. 重写 `supports()` 与 DoubaoAdapter 完全一样的实现（同样的 4 行解析 + contains）

**关键差异**：默认值字符串不同（doubao 空 vs deepseek `USER_FACING,BACKGROUND`）。

- [ ] **Step 1.5: 运行测试，确认 supports 用例全过**

```bash
cd /Users/aventador/code/3yan/server
mvn test -pl business_packages/sanyan-llm-core -Dtest='DoubaoAdapterTest,DeepSeekAdapterTest'
```

Expected: 两个 test 类所有用例（含 8 个新加的 supports）全过。

如果有非 supports 用例挂，重点检查 setUp 里是否需要给 `taskTypes` field 赋初值（推荐 setUp 里给 doubao 设 `"USER_FACING"`、deepseek 设 `"BACKGROUND,USER_FACING"`，避免后续断言全 false）。

- [ ] **Step 1.6: 跑 sanyan-llm-core 全包测试**

```bash
cd /Users/aventador/code/3yan/server
mvn test -pl business_packages/sanyan-llm-core
```

Expected: 全包全过。重点确认 `LLMProviderRouterTest`（mock 的 LLMProvider 不受改动影响）+ `LlmApplicationContextIT`（Spring 装配 + 默认配置 → doubao supports 全 false, deepseek 接两个 → USER_FACING + BACKGROUND 都能路由）。

- [ ] **Step 1.7: Commit**

```bash
cd /Users/aventador/code/3yan/server
git add business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/DoubaoAdapter.java \
        business_packages/sanyan-llm-core/src/main/java/com/sanyan/llm/internal/DeepSeekAdapter.java \
        business_packages/sanyan-llm-core/src/test/java/com/sanyan/llm/internal/DoubaoAdapterTest.java \
        business_packages/sanyan-llm-core/src/test/java/com/sanyan/llm/internal/DeepSeekAdapterTest.java
git commit -m "$(cat <<'EOF'
refactor(llm): Adapter.supports() 改成 yml 配置驱动

把 LLM provider 的 task type 路由从代码硬编码改成 @Value 注入的
sanyan.llm.<provider>.task-types 配置项（comma-separated 列表，
case-insensitive 解析 + 自动 trim 空格）。默认值：
- doubao: 空（不接任何任务）
- deepseek: USER_FACING,BACKGROUND（接所有任务）

动机：2026-05-25 dogfood 发现豆包 character 模型在多 system 块密集
prompt 结构下会把 system role 时间标签当 assistant 回复输出（20% 复
读率），切 DeepSeek 后 57/57 气泡 0 复读。本次把 provider 选择配置
化，切换不再需要改代码 + 重新打包，灰度/回滚都靠改 yml + 重启。

豆包 Adapter 代码保留待用——未来豆包修了这个问题或换 character v2
模型可以直接 yml 切回。

测试：每个 Adapter 加 4 个 supports 解析用例（空 / 单 type / 多 type /
空格大小写容错），sanyan-llm-core 全包测试全过。
EOF
)"
```

---

## Task 2: application.yml 加默认配置

**Files:**
- Modify or create: `server/bootstrap/src/main/resources/application.yml`

- [ ] **Step 2.1: 检查当前 application.yml 是否存在 sanyan.llm 配置块**

```bash
cd /Users/aventador/code/3yan/server
find bootstrap/src/main/resources -name "application*.yml" | xargs grep -l "sanyan" 2>/dev/null
```

如果文件存在，Read 它确认是否已有 `sanyan.llm` 块。

- [ ] **Step 2.2: 在 application.yml 显式声明默认 task-types**

在 `bootstrap/src/main/resources/application.yml` 里的 `sanyan` block 下添加（如果已存在 `sanyan.llm` 段，合并）：

```yaml
sanyan:
  # ... 其他已有配置 ...
  llm:
    doubao:
      task-types: ""        # 默认豆包不接任何任务（dogfood 验证：character 模型在多 system 密集结构下会复读 system 内容）
    deepseek:
      task-types: USER_FACING,BACKGROUND  # 默认 DeepSeek 接所有任务
```

**注意**：虽然 `@Value` 已有默认值，**仍然在 yml 显式声明**——这样 ops 改配置时一眼能看到当前设置，不用翻 Java 代码找 default value。

- [ ] **Step 2.3: 检查 application-prod.yml 等环境特定配置**

```bash
cd /Users/aventador/code/3yan/server
ls bootstrap/src/main/resources/application-*.yml 2>/dev/null
```

如果有 `application-prod.yml` 等环境 profile 文件，**检查是否需要 override**：本次方案是"所有环境都用 DeepSeek"，所以 prod 不需要 override。如果 prod 文件里已有 `sanyan.llm.*.task-types` 配置（不应该有），移除。

- [ ] **Step 2.4: 跑 LlmApplicationContextIT 验证 Spring 装配**

```bash
cd /Users/aventador/code/3yan/server
mvn verify -pl business_packages/sanyan-llm-core -Dit.test=LlmApplicationContextIT
```

Expected: BUILD SUCCESS。

- [ ] **Step 2.5: Commit**

```bash
cd /Users/aventador/code/3yan/server
git add bootstrap/src/main/resources/application.yml
git commit -m "$(cat <<'EOF'
config(llm): application.yml 显式声明 task-types 默认值

虽然 @Value 已有 default value，仍在 yml 里显式声明 ——
- doubao.task-types: ""（豆包不接任何任务）
- deepseek.task-types: USER_FACING,BACKGROUND（DeepSeek 接所有）

ops 修改配置时一眼能看到当前生效值，不用翻 Java 代码找 default。
EOF
)"
```

---

## Task 3: 全包测试 + 子模块 push + 主仓库 pointer 更新 + 部署

**Files:**
- Modify: `/Users/aventador/code/3yan/server`（pointer 更新）

- [ ] **Step 3.1: 跑 sanyan-llm-core 全包 mvn verify**

```bash
cd /Users/aventador/code/3yan/server
mvn verify -pl business_packages/sanyan-llm-core
```

Expected: 单测 + IT 全过。

- [ ] **Step 3.2: 跑 sanyan-chat-core 回归（确认 PromptBuilder/AiService 不受影响）**

```bash
cd /Users/aventador/code/3yan/server
mvn test -pl business_packages/sanyan-chat-core
```

Expected: 44/44 通过。

- [ ] **Step 3.3: push server 子模块**

```bash
cd /Users/aventador/code/3yan/server
git push
```

- [ ] **Step 3.4: 主仓库更新 pointer 并 commit / push**

```bash
cd /Users/aventador/code/3yan
git add server
git commit -m "$(cat <<'EOF'
chore: 更新 server 子模块引用（LLM provider 配置化 + 默认走 DeepSeek）
EOF
)"
git push
```

- [ ] **Step 3.5: 部署到 new**

```bash
cd /Users/aventador/code/3yan/server
./deploy.sh
```

Expected: ✅ 部署成功。

---

## Task 4: 部署后 dogfood 回归验证

**Files:** 无文件改动，仅运行已有 dogfood 脚本。

- [ ] **Step 4.1: scp 多场景 dogfood 脚本到 new**

```bash
scp /Users/aventador/code/3yan/server/scripts/dogfood/dogfood_test.py \
    /Users/aventador/.claude/jobs/ca722a57/time_awareness_dogfood.py \
    new:/tmp/
```

- [ ] **Step 4.2: 在 new 上跑 3 场景 dogfood**

```bash
ssh new "cd /tmp && python3 time_awareness_dogfood.py"
```

Expected: 与上次实验结果一致——
- 阶段 A 所有气泡 0 复读（57/57 干净）
- 验证 1 / 2 全过 3 场景
- 验证 3 至少 1 场景命中

如果**任一场景出现 phase_a leak ≥ 1**，停下来——说明配置切换没生效（可能 yml 加载失败、profile 不对、或者 @Value default 没注入）。排查方向：
1. `ssh new "journalctl -u 3yan-server --since '5 minutes ago' | grep -E 'doubao|deepseek|task-types'"` 看启动日志
2. 验证 jar 真正部署上去：`ssh new "ls -la /opt/3yan/3yan-server/target/"` 看 mtime
3. 跑 `curl -s http://localhost:8080/actuator/configprops` 看 spring 加载的实际配置（如果 actuator 开启）

- [ ] **Step 4.3: 清理 dogfood 数据 + 临时脚本**

```bash
ssh new "PGPASSWORD=\$(grep '^PG_PASSWORD=' /etc/3yan/3yan-server.env | cut -d= -f2-) psql -h localhost -U \$(grep '^PG_USER=' /etc/3yan/3yan-server.env | cut -d= -f2-) -d sanyan -c 'DELETE FROM chat_embeddings WHERE user_id IN (920,921,922); DELETE FROM memory_summaries WHERE user_id IN (920,921,922); DELETE FROM memory_profiles WHERE user_id IN (920,921,922); DELETE FROM message WHERE user_id IN (920,921,922); DELETE FROM users WHERE id IN (920,921,922);' && rm /tmp/dogfood_test.py /tmp/time_awareness_dogfood.py"
```

---

## Self-Review

**Plan 覆盖：**
- ✅ supports() 配置化 → Task 1
- ✅ application.yml 默认值显式 → Task 2
- ✅ 子模块 push + 主仓库 pointer + 部署 → Task 3
- ✅ dogfood 回归 + 清理 → Task 4

**类型一致性：**
- `taskTypes` field name 在 DoubaoAdapter、DeepSeekAdapter、测试 ReflectionTestUtils 全一致
- 配置 key `sanyan.llm.<provider>.task-types` 在 Java `@Value` 和 yml 全对齐

**无占位符：** 所有代码块完整可执行。

**关键决策回顾：**
- **`taskTypes` 字段为何不 final**：要支持 `ReflectionTestUtils.setField` 动态改值跑多个测试用例。原有其他字段（如 model、apiKey）是 final 也不动；只新加 field 非 final，最小侵入。
- **default 值不对称**（doubao 空，deepseek 接两种）：默认行为就是"所有环境用 DeepSeek"，不需要 prod override。豆包将来要回归时，改 yml 互换即可。
- **case-insensitive + 空格 trim 解析**：防 ops 手抖 yml 写 `user_facing` 或带空格，给容错。

**预期 dogfood 结果：**
应该跟上次实验一致——57 气泡 0 复读 + 3 场景验证 1/2 全过。如果不一致说明配置没真正生效，按 Step 4.2 的排查路径走。
