# 聊天时间感知 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 AI 跨天对话时不再把过去的对话说成"刚才聊的"——短期窗口（最近 32 条消息）每条都通过 system role 标签注入精确时间，AI 据此用准确时间词（刚才/今天/昨天/前天/上周）。

**Architecture:** 只动 2 个文件 —— `PromptBuilder.build` 拼装时在每条历史消息前插一条 system role 的 `[消息时间：M月d日 HH:mm]` 标签；`AiService.assembleSystemPrompt` 加一段 `[时间感知]` 引导词告诉 LLM 标签是元数据、不要复述。summary 不动，绕开 v2 的死结。

**Tech Stack:** Java 17 + Spring Boot 3 + JUnit 5 + Mockito + AssertJ。所有改动在 `business_packages/sanyan-chat-core` 单包内。

**关键约束（注入到每个实现子代理的 prompt 里）：**
- 遵守 TDD 铁律：先写失败测试 → 运行确认失败 → 写最小实现 → 运行确认通过 → commit
- 子代理类型：实现用 `general-purpose`，spec 审查用 `general-purpose`，代码质量审查用 `superpowers:code-reviewer`
- 测试粒度：本 plan 改动局限在 `sanyan-chat-core` 单包，不动 foundation 也不跨模块。每个 Task 跑 `mvn test -pl business_packages/sanyan-chat-core` 即可，不需要全量回归

**Spec 引用：** `docs/superpowers/specs/2026-05-24-chat-time-awareness-design.md`

---

## File Structure

| 文件 | 操作 | 职责 |
|---|---|---|
| `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/PromptBuilder.java` | 修改 | 加 `formatMessageTime` 方法 + 修改 `build` for 循环 |
| `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/PromptBuilderTest.java` | 修改 | 新增 3 个时间标签相关用例（现有用例 0 修改） |
| `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/AiService.java` | 修改 | 加 `TIME_AWARENESS_GUIDE` 常量 + 修改 `assembleSystemPrompt` |
| `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/AiServiceTest.java` | 修改 | 新增 1 个 `assembleSystemPrompt` 直接测试用例 |

**关于"现有用例 0 修改"：**
现有 PromptBuilderTest 里 `userMessage(content)` / `aiMessage(content)` 这两个 helper 不设置 `createdAt`，且现有用例里直接 `new MessageEntity()` 的地方也没设 `createdAt`。因为本次设计是「`createdAt == null` 时跳过时间标签」，所以所有现有用例的消息总数 / 顺序断言**完全不受影响**。AiServiceTest 同理。

---

## Task 1: PromptBuilder 时间标签插入

**Files:**
- Modify: `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/PromptBuilder.java`
- Test: `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/PromptBuilderTest.java`

- [ ] **Step 1.1：写第一个失败测试 — 每条消息前都插时间标签**

在 `PromptBuilderTest.java` 的 `import` 区追加：

```java
import java.time.LocalDateTime;
```

在文件末尾（`aiMessage` helper 之前）追加新用例：

```java
    // ---------- 2026-05-24 时间感知：每条消息前插 system role 时间标签 ----------

    @Test
    void build_shouldInsertTimeLabelBeforeEachMessage() {
        MessageEntity m1 = userMessageAt("早上好", LocalDateTime.of(2026, 5, 22, 14, 30));
        MessageEntity m2 = aiMessageAt("早呀", LocalDateTime.of(2026, 5, 22, 14, 32));

        List<Map<String, String>> result =
                builder.build("sys", null, null, List.of(m1, m2));

        // 1 system + (1 time-label + 1 user) + (1 time-label + 1 assistant) = 5
        // 注意：标签字符串里的冒号是全角"："，与下面 PromptBuilder 实现里的 MESSAGE_TIME_PREFIX 对齐
        assertThat(result).hasSize(5);
        assertThat(result.get(0)).containsEntry("role", "system").containsEntry("content", "sys");
        assertThat(result.get(1))
                .containsEntry("role", "system")
                .containsEntry("content", "[消息时间：5月22日 14:30]");
        assertThat(result.get(2)).containsEntry("role", "user").containsEntry("content", "早上好");
        assertThat(result.get(3))
                .containsEntry("role", "system")
                .containsEntry("content", "[消息时间：5月22日 14:32]");
        assertThat(result.get(4)).containsEntry("role", "assistant").containsEntry("content", "早呀");
    }

    @Test
    void build_shouldSkipTimeLabelWhenCreatedAtIsNull() {
        MessageEntity withTime = userMessageAt("有时间", LocalDateTime.of(2026, 5, 22, 14, 30));
        MessageEntity noTime = userMessage("没时间");  // helper 不设 createdAt → null

        List<Map<String, String>> result =
                builder.build("sys", null, null, List.of(withTime, noTime));

        // 1 system + (1 time-label + 1 user) + (1 user, 无 label) = 4
        assertThat(result).hasSize(4);
        assertThat(result.get(0)).containsEntry("role", "system").containsEntry("content", "sys");
        assertThat(result.get(1))
                .containsEntry("role", "system")
                .containsEntry("content", "[消息时间：5月22日 14:30]");
        assertThat(result.get(2)).containsEntry("role", "user").containsEntry("content", "有时间");
        assertThat(result.get(3)).containsEntry("role", "user").containsEntry("content", "没时间");
    }

    @Test
    void build_timeLabelShouldUseCorrectFormat() {
        // 验证格式：M月d日 HH:mm（无年份、无星期、无前导 0 的月份和日期）
        MessageEntity msg = userMessageAt("x", LocalDateTime.of(2026, 5, 22, 14, 30));

        List<Map<String, String>> result =
                builder.build("sys", null, null, List.of(msg));

        assertThat(result.get(1))
                .containsEntry("role", "system")
                .containsEntry("content", "[消息时间：5月22日 14:30]");
    }

    private static MessageEntity userMessageAt(String content, LocalDateTime createdAt) {
        MessageEntity m = userMessage(content);
        m.setCreatedAt(createdAt);
        return m;
    }

    private static MessageEntity aiMessageAt(String content, LocalDateTime createdAt) {
        MessageEntity m = aiMessage(content);
        m.setCreatedAt(createdAt);
        return m;
    }
```

**注意**：`[消息时间:5月22日 14:30]`.replace(":", "：") 是为了在断言里用清晰的代码表达全角冒号。实际期望字符串就是 `[消息时间：5月22日 14:30]`（全角冒号，与下面实现里的 `MESSAGE_TIME_PREFIX = "[消息时间："` 对齐）。

- [ ] **Step 1.2：运行测试，确认 3 个新用例失败**

```bash
cd /Users/aventador/code/3yan/server
mvn test -pl business_packages/sanyan-chat-core -Dtest=PromptBuilderTest
```

Expected: 3 个新用例失败（断言消息总数错误、找不到 time-label system 段），其余 16 个原有用例全部通过。

如果原有用例也失败，停下来——说明 fixture 假设有误，需要排查。

- [ ] **Step 1.3：实现最小代码让测试通过 — 修改 PromptBuilder**

在 `PromptBuilder.java` 里 `import` 区追加：

```java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Locale;
```

在 `MEMORY_PREFIX` 常量之后追加：

```java
    /** 消息时间标签的前缀（全角冒号，跟现有 [当前时间] 风格一致）。 */
    static final String MESSAGE_TIME_PREFIX = "[消息时间：";

    /** 消息时间标签的后缀。 */
    static final String MESSAGE_TIME_SUFFIX = "]";

    /**
     * 消息时间格式：M月d日 HH:mm（无年份、无星期）。
     * <p>短期窗口最多覆盖几天到几周（32 条消息），年份冗余；星期 token 密度低，砍掉。
     */
    private static final DateTimeFormatter MESSAGE_TIME_FORMATTER =
            DateTimeFormatter.ofPattern("M月d日 HH:mm", Locale.CHINESE);

    private static String formatMessageTime(LocalDateTime time) {
        return MESSAGE_TIME_PREFIX + time.format(MESSAGE_TIME_FORMATTER) + MESSAGE_TIME_SUFFIX;
    }
```

修改 `build` 方法里的 for 循环（当前 line 87-91）：

```java
        for (MessageEntity msg : limited) {
            if (msg.getCreatedAt() != null) {
                messages.add(Map.of("role", "system", "content", formatMessageTime(msg.getCreatedAt())));
            }
            String role = SenderType.USER.equals(msg.getSenderType()) ? "user" : "assistant";
            String content = msg.getContent() == null ? "" : msg.getContent();
            messages.add(Map.of("role", role, "content", content));
        }
```

- [ ] **Step 1.4：运行测试，确认全部通过**

```bash
cd /Users/aventador/code/3yan/server
mvn test -pl business_packages/sanyan-chat-core -Dtest=PromptBuilderTest
```

Expected: 19 个用例全部通过（原 16 个 + 新 3 个）。

如果原有用例出现失败，重点检查 `build_shouldEmitFullPromptStructureWithCharacterAndMemoryAndHistory` / `build_shouldIncludeAllMessagesWhenExactlyAtWindowSize` 这些断言消息数的用例——理论上 fixture 没设 createdAt 不会受影响，挂了说明实现没正确判 null。

- [ ] **Step 1.5：Commit**

```bash
cd /Users/aventador/code/3yan/server
git add business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/PromptBuilder.java \
        business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/PromptBuilderTest.java
git commit -m "$(cat <<'EOF'
feat(chat): PromptBuilder 每条消息前插 system role 时间标签

跨天对话 AI 把过去说成"刚才聊的"的修复 v3——短期窗口里每条消息前插
一条 system role 的 [消息时间：M月d日 HH:mm] 标签。createdAt 为 null 时
跳过该条标签，消息本身正常加。

格式选择：无年份（短期窗口覆盖不到跨年）、无星期（token 密度低）。
LLM 角度：system role 标签训练上不会被复读到 user-facing 回复里，比
v1 直接在 message content 里加 prefix 安全。引导词在 AiService 那边
另一个 commit 加。

PromptBuilderTest 新增 3 用例（每条前都插标签 / createdAt null 跳过 /
格式正确），现有 16 用例 fixture 不带 createdAt，0 修改全过。
EOF
)"
```

---

## Task 2: AiService 时间感知引导词

**Files:**
- Modify: `server/business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/AiService.java`
- Test: `server/business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/AiServiceTest.java`

- [ ] **Step 2.1：写失败测试 — assembleSystemPrompt 输出含时间感知引导**

在 `AiServiceTest.java` 文件末尾（`productionSystemPromptResource_containsRelationshipBoundaries` 之前或之后）追加新用例：

```java
    // ---------- 2026-05-24 时间感知：assembleSystemPrompt 引导词 ----------

    @Test
    void assembleSystemPrompt_shouldIncludeTimeAwarenessGuide() {
        String result = service.assembleSystemPrompt("你是小婉", "2026年5月24日 周日 10:00");

        // 当前时间段保持原样
        assertThat(result)
                .contains("你是小婉")
                .contains("[当前时间] 2026年5月24日 周日 10:00");

        // 新加的时间感知引导段
        assertThat(result)
                .as("必须含 [时间感知] 段标题")
                .contains("[时间感知]");
        assertThat(result)
                .as("必须显式禁止 LLM 复述时间标签格式")
                .contains("不要复述");
        assertThat(result)
                .as("必须给出至少一个准确时间词示例，避免 LLM 用生硬表达")
                .containsAnyOf("刚才", "今天", "昨天", "前天", "上周");
        assertThat(result)
                .as("必须告诉 LLM 标签格式以便识别为系统元数据")
                .contains("[消息时间：");
    }
```

- [ ] **Step 2.2：运行测试，确认新用例失败**

```bash
cd /Users/aventador/code/3yan/server
mvn test -pl business_packages/sanyan-chat-core -Dtest=AiServiceTest
```

Expected: 新用例 `assembleSystemPrompt_shouldIncludeTimeAwarenessGuide` 失败（assembleSystemPrompt 返回不含 `[时间感知]` 等内容）；其余原有用例全部通过。

- [ ] **Step 2.3：实现最小代码 — 修改 AiService.assembleSystemPrompt**

在 `AiService.java` 类内（建议放在 `systemPromptTemplate` 字段之后、`@PostConstruct` 方法之前）追加常量：

```java
    /**
     * 时间感知引导段，拼在 system prompt 的「[当前时间]」之后。
     * <p>2026-05-24 v3 时间感知方案：PromptBuilder 在每条历史消息前插一条 system role 的
     * 「[消息时间：M月d日 HH:mm]」标签；本引导段告诉 LLM 标签是元数据、不要复述、
     * 并基于「当前时间」与「消息时间」的差距使用准确时间词。
     */
    private static final String TIME_AWARENESS_GUIDE = """
            [时间感知]
            对话历史中，每条消息前会有一条 system 角色的"[消息时间：M月d日 HH:mm]"标签，\
            表明该条消息的发送时刻。这是系统元数据，**不要复述到你的回复里**，也不要模仿这个格式。
            回复时请基于「当前时间」与「消息时间」的差距，使用准确的时间词（刚才/今天上午/昨天/前天/上周等），\
            不要把几天前的对话说成"刚才聊的"。""";
```

修改 `assembleSystemPrompt` 方法（当前 line 143-145）：

```java
    public String assembleSystemPrompt(String characterPrompt, String time) {
        return characterPrompt + "\n\n[当前时间] " + time + "\n\n" + TIME_AWARENESS_GUIDE;
    }
```

- [ ] **Step 2.4：运行测试，确认全部通过**

```bash
cd /Users/aventador/code/3yan/server
mvn test -pl business_packages/sanyan-chat-core -Dtest=AiServiceTest
```

Expected: 全部用例通过。重点确认：
- 新用例 `assembleSystemPrompt_shouldIncludeTimeAwarenessGuide` 通过
- 现有 `chat_shouldPassSystemPromptLoadedFromResourceToRouter` 通过（它用 `.contains()` 断言，新加内容不会让它挂）

- [ ] **Step 2.5：Commit**

```bash
cd /Users/aventador/code/3yan/server
git add business_packages/sanyan-chat-core/src/main/java/com/sanyan/chat/internal/AiService.java \
        business_packages/sanyan-chat-core/src/test/java/com/sanyan/chat/internal/AiServiceTest.java
git commit -m "$(cat <<'EOF'
feat(chat): system prompt 加 [时间感知] 引导段，禁止 LLM 复述时间标签

配合上一 commit PromptBuilder 的时间标签注入，本 commit 在
assembleSystemPrompt 的「[当前时间]」之后追加 [时间感知] 引导段：
- 告诉 LLM 历史消息前的 system 标签是元数据，不要复述/模仿
- 给出准确时间词示例（刚才/今天上午/昨天/前天/上周）
- 给反例（不要把几天前说成"刚才聊的"）

新增 1 个直接测 assembleSystemPrompt 的用例，断言引导段标题 +
禁止复述 + 时间词示例 + 标签格式描述。现有用例 0 修改全过。
EOF
)"
```

---

## Task 3: 全包测试 + 子模块引用更新

**Files:**
- Modify: `/Users/aventador/code/3yan/server`（subref pointer，主仓库视角）

- [ ] **Step 3.1：跑 sanyan-chat-core 全包测试做最终自检**

```bash
cd /Users/aventador/code/3yan/server
mvn test -pl business_packages/sanyan-chat-core
```

Expected: 整个 sanyan-chat-core 包测试全过（包括 PromptBuilderTest 19 个 + AiServiceTest 15 个 + 其他类的所有测试）。

如果有失败，重点看是不是引入了 import 缺失 / lint 错误。

- [ ] **Step 3.2：跑 sanyan-chat-core 包的 mvn verify 跑集成测试（如果有 *IT.java）**

```bash
cd /Users/aventador/code/3yan/server
mvn verify -pl business_packages/sanyan-chat-core
```

Expected: 单测 + 集成测试都通过。如果包里没有 *IT.java 则只跑单测。

- [ ] **Step 3.3：push server 子模块**

```bash
cd /Users/aventador/code/3yan/server
git push
```

Expected: 2 个新 commit（Task 1 + Task 2）成功推送到子模块远端。

- [ ] **Step 3.4：主仓库更新 server 子模块引用并 commit / push**

```bash
cd /Users/aventador/code/3yan
git add server
git commit -m "$(cat <<'EOF'
chore: 更新 server 子模块引用（v3 时间感知 — 每条消息前插 system role 时间标签）
EOF
)"
git push
```

Expected: 主仓库 master 多一个 commit 更新 server pointer，并推到 origin。

---

## Task 4: dogfood 端到端验证（手工，部署测试环境前）

> **本 task 不走 TDD subagent 流程**——是手工验证步骤，建议用户本人执行（涉及部署、真实 LLM 调用、主观感受评估）。

**前置：** Task 3 已完成、代码已 push 到 server master，但**还未部署到测试环境**。本 task 完成后才决定是否部署。

- [ ] **Step 4.1：部署到测试环境**

按用户项目惯例的部署流程（CLAUDE.md「部署规范」明确说**只有用户明确说"部署"时**才能执行；如果用户没明确指令，停在 Task 3 等用户指令）。

如果用户授权部署：

```bash
ssh new
# 在服务器上按项目部署脚本部署 server 到测试环境
```

- [ ] **Step 4.2：用 dogfood 用户跟 AI 聊 5+ 条消息**

通过 app 或直接通过 API 用 dogfood 账号跟小婉聊一段话，发 5 条以上消息让短期窗口里有一定量内容。

- [ ] **Step 4.3：SQL 修改某条消息的 createdAt 改成 2 天前**

在测试环境数据库里执行：

```sql
-- 把刚才聊过的某条消息 createdAt 改到 2 天前（具体 user_id / message id 按实际填）
UPDATE message
SET created_at = NOW() - INTERVAL '2 days'
WHERE id = <某条消息的 id>;
```

- [ ] **Step 4.4：发新消息追问，验证 3 条**

发新消息："刚才聊的 X 是啥"（X 是上一步改了时间的那条消息内容里的关键词）。

观察 AI 回复：

- **验证 1（核心）**：AI 回复**不包含**"刚才"、"刚聊"、"刚刚" 这类词，应该说"前天聊的"、"前天提到的"或类似准确时间表达
- **验证 2（复读检查）**：AI 回复**不包含** `[消息时间：` 字样、也不包含全角方括号包时间的格式
- **验证 3（连贯性）**：跟 AI 再连续聊 5-10 轮，主观感受回复连贯性、人设感、对话流畅度**没有明显下降**（对比上线前）

- [ ] **Step 4.5：根据验证结果决定下一步**

- 三条都通过 → 收工，方案落地
- 验证 1 失败（仍说"刚才"） → 检查时间标签是否真的进了 prompt（看服务端日志），可能需要加强引导词措辞
- 验证 2 失败（复读时间标签） → 退到备选方案：把时间标签密度降低（v2 思路：间隔大才插）。需要重新 brainstorming
- 验证 3 失败（连贯性下降明显） → 同上，密度太高把豆包搞晕了，需要 brainstorm 折中方案

---

## Self-Review

**Spec 覆盖：**
- ✅ §3.1 接入点（PromptBuilder + AiService 两处）→ Task 1 + Task 2
- ✅ §3.2 PromptBuilder 改造（常量 + 方法 + for 循环）→ Task 1.3
- ✅ §3.3 AiService 改造（TIME_AWARENESS_GUIDE 常量 + assembleSystemPrompt）→ Task 2.3
- ✅ §5.1 PromptBuilderTest 新增 3 用例 → Task 1.1
- ✅ §5.2 AiServiceTest 改/加用例 → Task 2.1（plan 里改成"新增"而不是"修改"，因为发现现有用例 fixture 不带 createdAt 不会受影响）
- ✅ §5.3 dogfood 三条验证 → Task 4.4
- ✅ §5.4 测试运行范围（只跑 sanyan-chat-core 单包）→ 每个 Task 测试命令都加 `-pl business_packages/sanyan-chat-core`
- ✅ §6 完成定义 → Task 3.1 + Task 4

**类型一致性：**
- `MESSAGE_TIME_PREFIX` / `MESSAGE_TIME_SUFFIX` / `MESSAGE_TIME_FORMATTER` / `formatMessageTime` 在 spec、Task 1.3、Task 2.3 引用一致
- `TIME_AWARENESS_GUIDE` 在 spec §3.3 和 Task 2.3 中名字、可见性（private static final）一致
- 全角冒号 `：` 在「[消息时间：」「[当前时间] 」「[时间感知]」全文一致

**无占位符：** 所有代码块都是完整可运行的代码，没有 TBD / TODO / "类似 Task N"。每个测试和实现的具体代码都写出来了。

**风险点回顾：**
- Spec §4.1 / §4.2 / §4.3 三个风险点的缓解都映射到 Task 4 dogfood 验证
- Task 4.5 列出了"验证失败时的退路"，避免发现问题时不知所措

---

## 备选方案（仅在 Task 4 验证失败时启用）

如果 §4.4 验证 2（复读）或验证 3（连贯性）失败，备选方案：

**降级到 v2 思路** — 把"每条都插标签"改成"相邻消息 `createdAt` 间隔 >= 60 分钟才插标签"：
- 修改 `PromptBuilder.build` 的 for 循环，加上 `lastCreatedAt` 跟踪和 `Duration.between(...).toMinutes() >= 60` 判断
- 标签内容从「[消息时间：...]」改成「（时间跳跃：...）」（v2 措辞）
- 引导词相应调整

这是「精度 vs 风险」的折中：放弃"每条都能精确查时间"的目标，换回 LLM 连贯性 + 更低复读概率。

**触发标准：** 仅在 Task 4.4 验证明确失败时使用，不要 Task 1/2 期间提前 fallback。
