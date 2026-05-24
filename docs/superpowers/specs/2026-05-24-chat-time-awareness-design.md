# 聊天时间感知设计 spec

**日期**：2026-05-24
**作者**：Aventador + Claude
**状态**：待实现

---

## 1. 背景与问题

### 1.1 用户场景

用户跟 AI（小婉）聊天，**跨天后**继续对话时，AI 会错误地把过去的对话当作"刚刚发生"。

**实测 bug**（来自 2026-05-23 dogfood）：
- 用户前天聊了机器人故事
- 今天再问"科幻新思路是啥"
- AI 回复："就是我们刚聊的那个机器人的故事呀"
- 期望：AI 应该说"前天聊的那个机器人故事"或类似

### 1.2 根因

LLM 在 system prompt 里只有「当前时间」，对话历史（OpenAI 兼容消息数组里的 `user` / `assistant` 消息）**不携带任何时间信息**。LLM 默认假设这些消息都是"刚才"发生的。

### 1.3 历史尝试与回滚

2026-05-23 做过两轮尝试，均已 revert：

**v1（commit `f16e111`）**：给每条历史消息 content 加 `[5月23日 01:15]` 前缀。
- ❌ **失败原因**：豆包模仿性强，把 `[5月23日 01:15]` 这个格式复读到自己回复里（截图：AI 回 `[5月23日 01:15] 怎么又问一遍呀？`）

**v2（commit `28d2d8c`）**：改成相邻消息 `createdAt` 间隔 >= 60 分钟时，在两条之间插入一条 system role 的 `（时间跳跃：5月23日 00:18）` 标记。
- ❌ **失败原因**：思路有缺陷——summary（长期记忆的浓缩文本）跨多天，没法标单一时间。整体方案被 revert。

**本次方案**：聚焦"短期窗口（最近 32 条消息）的时间感知"，summary 完全不动——这绕开了 v2 的死结。

---

## 2. 范围

### 2.1 In Scope

- 短期窗口（最近 32 条 `MessageEntity`）里**每条**消息让 LLM 能查到准确发送时间（精到分钟）
- LLM 据此使用准确时间词（刚才/今天上午/昨天/前天/上周）
- 时间信息不能被 LLM 复读到回复里

### 2.2 Out of Scope

- **summary**（`memoryContext.text()`）不带时间——保持现状，summary 是浓缩文本，跨多天天然没法标单一时间
- **RAG 召回片段**不带时间——同上
- 当前时间 vs 最后一条消息时间的 gap 提示——不显式做，靠 LLM 自己根据「当前时间」+「消息时间」推断
- 多轮跨度的高级时间表达（"很久之前"、"那段时间"）不刻意优化

### 2.3 精度边界

- 时间精度：**分钟级**（`M月d日 HH:mm`，不带年份不带星期）
- 范围：短期窗口最多覆盖几天到几周（32 条消息），年份冗余

---

## 3. 设计

### 3.1 接入点

**只改 2 个文件**：

| 文件 | 职责 |
|---|---|
| `sanyan-chat-core/internal/PromptBuilder.java` | 拼装时每条历史消息前插一条 system role 的时间标签 |
| `sanyan-chat-core/internal/AiService.java` | system prompt 加一段时间感知引导词 |

**不动**：
- `MessageEntity` / `MessageRepository`
- `MemoryApi` / `CharacterApi` 及其实现
- summary / RAG / 长期记忆相关代码
- `AiService.chat` 主流程

### 3.2 PromptBuilder 改造

**新增常量与方法**（放在 `PromptBuilder` 类内部，单一调用点不下沉）：

```java
static final String MESSAGE_TIME_PREFIX = "[消息时间：";
static final String MESSAGE_TIME_SUFFIX = "]";

private static final DateTimeFormatter MESSAGE_TIME_FORMATTER =
        DateTimeFormatter.ofPattern("M月d日 HH:mm", Locale.CHINESE);

private static String formatMessageTime(LocalDateTime time) {
    return MESSAGE_TIME_PREFIX + time.format(MESSAGE_TIME_FORMATTER) + MESSAGE_TIME_SUFFIX;
}
```

**`build` 方法 for 循环改造**（替换现有 line 87-91 的循环）：

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

**输出形态示例**：

```
[system] 人设 + 当前时间 + 时间感知引导
[system] 关系阶段
[system] 长期记忆
[system] [消息时间：5月22日 14:30]
[user]   你今天吃什么？
[system] [消息时间：5月22日 14:32]
[assistant] 我吃了沙拉。
[system] [消息时间：5月24日 10:00]
[user]   刚才聊的吃什么是几号？
```

**关键决策**：

| 项 | 选择 | 理由 |
|---|---|---|
| 格式 | `M月d日 HH:mm`（无年份、无星期） | 短期窗口最多覆盖几周，年份冗余；星期 token 密度低 |
| `createdAt == null` 时 | 跳过该条时间标签，消息本身正常加 | 跟 v1 兜底策略一致；测试 fixture 没填时间不影响主链路 |
| 标签位置 | 消息**前** | 与人类习惯一致，先时间后内容 |
| 是否合并相邻标签 | 不合并 | 每条独立标，逻辑简单；token 节省不显著 |
| 是否抽工具类 | 不抽 | 单一调用点，按全局规则等出现第二个调用方再下沉 |

### 3.3 AiService 改造

**`assembleSystemPrompt` 改造**：

```java
public String assembleSystemPrompt(String characterPrompt, String time) {
    return characterPrompt + "\n\n[当前时间] " + time + "\n\n" + TIME_AWARENESS_GUIDE;
}

private static final String TIME_AWARENESS_GUIDE = """
        [时间感知]
        对话历史中，每条消息前会有一条 system 角色的"[消息时间：M月D日 HH:mm]"标签，\
        表明该条消息的发送时刻。这是系统元数据，**不要复述到你的回复里**，也不要模仿这个格式。
        回复时请基于「当前时间」与「消息时间」的差距，使用准确的时间词（刚才/今天上午/昨天/前天/上周等），\
        不要把几天前的对话说成"刚才聊的"。
        """;
```

**关键决策**：

| 项 | 选择 | 理由 |
|---|---|---|
| 引导段位置 | 拼在 `[当前时间]` 之后、人设之外 | 与「当前时间」同主题，紧邻语义连贯 |
| 强调"不要复述/模仿" | 显式加粗 + 明示禁止 | v1 教训：豆包模仿性强必须主动压制 |
| 给具体时间词示例 | 给 | 不给的话 LLM 可能用"约 48 小时前"生硬表达 |
| 给反例 | 给（"不要把几天前说成刚才聊的"） | 反例约束力比正例强 |
| 是否放入 `xiaowan-system.md` 人设文件 | 不放 | 时间感知是系统对话机制，性质与角色定义不同；放人设里污染角色定义、未来新角色每份都得改 |

---

## 4. 风险与缓解

### 4.1 风险 1：连续 system 块影响 LLM 回复连贯性

**风险**：方案下短期窗口里 `user` / `assistant` 之间穿插了大量 system role 消息（32 条 → 64 条），OpenAI 协议合法，但豆包训练 corpus 里这种结构密度可能不高，回复连贯性可能下降。

**缓解**：
- dogfood 端到端验证（§5.3）主观评估回复连贯性
- 如发现明显下降，备选方案：退到 v2 思路（只在间隔 >= N 分钟时插标签）—— 牺牲细粒度精度换连贯性

### 4.2 风险 2：豆包仍然复读时间标签

**风险**：虽然 v2 验证了 system role 标记不被复读，但 v2 是"间隔大才插"，本次是"每条都插"，密度高 100 倍，是否仍然不复读未实测。

**缓解**：
- system prompt 引导词明确写"不要复述"+"不要模仿格式"
- dogfood 验证（§5.3 验证 2）

### 4.3 风险 3：token 成本上涨

**风险**：消息数从 32 翻到 64，token 增加。

**估算**：每条时间标签约 15 tokens，32 条 → 增加约 480 tokens / 次。豆包当前主对话上下文通常几千 tokens，增加 5-10% 可接受。

**缓解**：不做（成本可接受）。

---

## 5. 测试

### 5.1 PromptBuilderTest（新增 3 + 修现有）

**新增**：

| 用例 | 场景 | 关键断言 |
|---|---|---|
| `build_shouldInsertTimeLabelBeforeEachMessage` | 2 条消息都有 `createdAt` | 每条历史消息前一条 system role 的 `[消息时间：...]` |
| `build_shouldSkipTimeLabelWhenCreatedAtIsNull` | 一条有 `createdAt` 一条 null | 有 `createdAt` 的前面有标签，null 那条前面没标签，但消息本身仍加入 |
| `build_timeLabelShouldUseCorrectFormat` | 一条消息 `createdAt = 2026-05-22T14:30` | 标签 content 精确为 `[消息时间：5月22日 14:30]` |

**修改**：现有用例对消息数 / system 段顺序的断言要相应调整。

### 5.2 AiServiceTest（修 1）

修改 `assembleSystemPrompt` 用例，断言输出包含：
- `[时间感知]`
- `不要复述`
- 至少一个示例时间词（`刚才` / `今天上午` / `昨天` / `前天` / `上周`）

### 5.3 dogfood 端到端验证（手工，部署前）

测试环境跑：
1. 用 dogfood 用户跟 AI 聊一段话（5+ 条消息）
2. SQL 修改某条历史消息的 `createdAt` 改成 2 天前
3. 发新消息追问"刚才聊的 X 是啥"
4. **验证 1**：AI 回复不包含"刚才/刚聊"，应该说"前天聊的"或类似
5. **验证 2**：AI 回复不包含 `[消息时间：` 字样（复读检查）
6. **验证 3**：连续聊 5-10 轮，主观感受回复连贯性没有明显下降

dogfood 不自动化的理由：需要真实 LLM 调用 + "复读" / "连贯性"在 mock 测试里没法验证。

### 5.4 测试运行范围

按 CLAUDE.md「Task 测试粒度规范」：
- 改动局限在 `sanyan-chat-core` 单包，不动 foundation 也不跨模块
- 跑 `mvn test -pl business_packages/sanyan-chat-core` 即可
- 不需要全量回归

---

## 6. 完成定义

- [ ] PromptBuilder 改造完成，新增 3 个单测 + 修改原有用例全部通过
- [ ] AiService 改造完成，修改的引导词断言通过
- [ ] `sanyan-chat-core` 包 `mvn test` 全绿
- [ ] dogfood 端到端三条验证（§5.3）通过
- [ ] commit message 用中文描述改动（参考 v1/v2 commit 风格）
