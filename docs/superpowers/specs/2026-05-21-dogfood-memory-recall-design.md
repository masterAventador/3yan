# Dogfood 记忆能力黑盒召回测试

**日期**：2026-05-21
**作者**：brainstorm session
**状态**：设计已确认，待写实现 plan
**目标读者**：实现这个 spec 的工程师/agent，以及未来维护 dogfood 的人

---

## 1. 背景与问题

`server/scripts/dogfood/dogfood_test.py` 现有 4 个 plan2 场景 `profile / throttle / summary / rag`，全部是**白盒断言**——只查数据库表（`memory_profiles` / `memory_summaries` / `chat_embeddings`）是否落库，不验证 AI 在后续对话里是否真的能召回这些记忆并答出来。

也就是说：现在的 dogfood 只能告诉你"记忆机制有没有落地"，无法告诉你"AI 在用户视角下是否真的记得"。这两件事并非等价。落库正确但 prompt 注入失败、注入了但 LLM 答非所问、关键事实被概括稀释——这些 bug 现有 dogfood 一个也抓不到。

本设计追加 3 个**黑盒端到端**场景，在真实链路上发若干条消息把目标事实挤出短期窗口（32 条），再问 AI，检查回复是否命中预期关键词。

---

## 2. 关键技术约束

实现前必须理解的链路常量（来自 `MemoryConstants.java`、`AiService.java`）：

| 常量 | 值 | 影响 |
|---|---|---|
| `SHORT_TERM_WINDOW_SIZE` | 32 | 每次 AI 回复时把最近 32 条 message（user+ai 混合，**不是只 user**）拼进 prompt。**事实必须被推到这 32 条之外**才算考验长期记忆。 |
| `SUMMARY_TRIGGER_THRESHOLD` | 30 | 累积 30 条新消息触发后台 summary。 |
| `RAG_CHUNK_MIN_SIZE / MAX_SIZE` | 5 / 10 | 每段 RAG chunk 大小，要让 plant 至少占 1 个 chunk。 |
| `RAG_MIN_COS_SIM` | 0.6 | RAG 召回最低 cos sim 阈值；probe 跟 plant 的语义距离要 ≥ 此值才会进 top-K。 |
| `PROFILE_EXTRACT_THROTTLE_MINUTES` | 5 | profile 抽取节流；**首次抽取不受节流影响**，dogfood 测试时每个新 user_id 都是首次。 |

`AiService.chat`（行 110-141）每次调用 LLM 前的固定 4 步：拉短期窗口 32 条 → 调 `MemoryApi.getRelevantContext` 拿 profile+summary+RAG 三层整合 → `PromptBuilder` 拼装 → `LlmApi.chat`。

---

## 3. 设计概览

### 3.1 新增 3 个场景

接在现有 plan2 4 场景之后，纳入 `SCENARIO_ORDER` 默认集：

```python
SCENARIO_ORDER = [
    "profile", "throttle", "summary", "rag",          # 现有：白盒（机制落库）
    "memory_recall_profile",                          # 新：黑盒（AI 真召回）
    "memory_recall_summary",
    "memory_recall_rag",
]

SCENARIO_USER_IDS = {
    ...
    "memory_recall_profile": 905,
    "memory_recall_summary": 906,
    "memory_recall_rag": 907,
    ...
}
```

**白盒先跑、黑盒后跑** 的串行排序刻意保留：白盒挂了说明上游断了，黑盒必然挂；这种顺序帮排查时从浅到深定位故障点。

### 3.2 共享骨架函数

```python
WaitFn = Callable[[DbHandle, int, Logger], Awaitable[bool]]

async def run_memory_recall(
    scenario_name: str,
    plant_messages: list[str],
    post_plant_wait_fn: Optional[WaitFn],
    distract_count: int,
    post_distract_wait_fn: Optional[WaitFn],
    probe_message: str,
    expected_keywords: list[str],
    token: str,
    db: DbHandle,
    user_id: int,
    character_id: int,
    log: Logger,
) -> ScenarioResult:
    """6 步固定流程：clean → plant → wait1 → distract → wait2 → probe，关键词命中即 PASS。"""
```

6 步：

1. `clean_test_data(db, user_id, log)`（沿用现有：删 message / memory_summaries / memory_profiles / chat_embeddings / redis throttle / RAG queue）
2. **Plant**：逐条 `send_message_via_ws(plant_messages[i])`，每条等 AI 回完（沿用现有 ws helper）
3. **Wait1（plant 后）**：若 `post_plant_wait_fn` 非 None 调它，超时直接 FAIL（"上游机制断了"）。profile / RAG 场景用，summary 场景传 None
4. **Distract**：从 `RECALL_DISTRACT_MESSAGES` 池取前 `distract_count` 条循环发，每条等 AI 回完
5. **Wait2（distract 后）**：若 `post_distract_wait_fn` 非 None 调它，超时 FAIL。summary 场景用（distract 跑完才满足 SUMMARY_TRIGGER_THRESHOLD），profile / RAG 场景传 None
6. **Probe**：`send_message_via_ws(probe_message)` 拿 reply.text → `any(kw in reply.text for kw in expected_keywords)` → PASS/FAIL

3 个场景的差异只是塞给骨架的 4 个参数（plant 内容 / wait 函数 / probe 文本 / 关键词），骨架本身一份代码。

### 3.3 三场景具体内容

#### 3.3.1 `memory_recall_profile`（验证人设抽取）

- **plant_messages（5 条）**：
  1. `"我叫王莎莎，今年27岁，在杭州做 Java 后端"`
  2. `"我老家是四川绵阳的，去年才来杭州工作"`
  3. `"我特别怕辣，川菜里只敢吃糖醋排骨那种"`
  4. `"周末喜欢打羽毛球，水平业余但能赢我同事"`
  5. `"我有只叫汤圆的英短猫，今年3岁了"`
- **post_plant_wait_fn**：`_wait_profile_landed`（30s 轮询 `memory_profiles WHERE user_id=905`）
- **distract_count**：35
- **post_distract_wait_fn**：`None`
- **probe_message**：`"对了 你还记得我老家是哪里的吗？"`
- **expected_keywords**：`["绵阳", "四川", "川"]`
- **为什么不会假阳性**：probe 之前 35 条 distract 把 plant 推到第 41 条之外（远超 32 条短期窗口），AI 拿到"绵阳"的唯一渠道 = `memory_profiles.summary_text` 注入 prompt。

#### 3.3.2 `memory_recall_summary`（验证滚动摘要）

- **plant_messages（3 条）**：
  1. `"今天去公司楼下新开的那家寿司店吃饭"`
  2. `"结果点的三文鱼刺身吃完拉肚子拉了一下午"`
  3. `"下次绝对不去那家了"`
- **post_plant_wait_fn**：`None`（plant 仅 3 条不够触发 summary，wait 到 distract 后）
- **distract_count**：35
- **post_distract_wait_fn**：`_wait_summary_landed`（30s 轮询 `memory_summaries WHERE user_id=906` 出现新行）
- **probe_message**：`"前阵子我说过哪家店让我吃坏肚子来着？"`
- **expected_keywords**：`["寿司", "刺身", "三文鱼"]`

**SUMMARY_TRIGGER_THRESHOLD=30 触发保证**：clean 把 `message` 表清空（计数器从 0 起），plant 3 + distract 35 = 38 条 user msg + ~38 条 ai msg，user_id=906 名下累积 ~76 条新消息，远超 30 触发线。

#### 3.3.3 `memory_recall_rag`（验证语义召回）

- **plant_messages（5 条）**：
  1. `"下周三我要去成都出差，跟一个甲方碰需求"`
  2. `"项目是给某个银行做风控系统对接"`
  3. `"本来不想去，但项目经理逼我"`
  4. `"在成都待 3 天，周五晚上飞回来"`
  5. `"酒店订的是春熙路那边"`
- **post_plant_wait_fn**：`_wait_rag_chunk_landed`（30s 轮询 `chat_embeddings WHERE user_id=907` ≥ 1 行）
- **distract_count**：35
- **post_distract_wait_fn**：`None`
- **probe_message**：`"提醒一下，我下周哪天要出差？"`（**不直接问"成都"** 避免污染，改问"哪天"间接逼 AI 召回上下文）
- **expected_keywords**：`["周三", "星期三", "下周三"]`
- **RAG 必经路径**：probe 文本跟 plant 第 1 条语义距离极近，cos sim 大概率 > 0.6 阈值，进 RAG top-K。

### 3.4 公共数据

#### `RECALL_DISTRACT_MESSAGES`（模块顶层常量，35 条纯闲聊填充）

每条都是日常话题切换，**绝不涉及个人身份/具体事件/任何场景关键词**——避免 AI 在填充阶段的回复巧合提到"绵阳/汤圆/寿司/成都"等词污染后续召回判定。

完整 35 条：

```python
RECALL_DISTRACT_MESSAGES = [
    "今天天气怎么样啊", "你平时几点睡觉", "最近有什么好看的剧推荐吗",
    "周末一般都干嘛", "你喜欢吃什么口味的菜", "运动一般做什么",
    "工作上忙吗", "今天心情怎么样", "你对秋天感觉如何",
    "咖啡还是茶", "早睡早起做得到吗", "节假日喜欢出去玩还是宅家",
    "你养过宠物吗", "听音乐喜欢什么类型", "周末有出去的打算吗",
    "你怎么看待加班这件事", "最近压力大吗", "买东西看重价格还是品牌",
    "对未来三年有什么规划", "聊聊你最近读的书吧", "你怕冷还是怕热",
    "更喜欢城市还是乡村", "你早上吃早饭吗", "对健身房有什么看法",
    "拖延症严重吗", "聊天和打电话你更喜欢哪个", "周末睡懒觉到几点",
    "对相亲怎么看", "你怎么放松", "晚上一般几点入睡",
    "你做饭吗", "对短视频上瘾吗", "下班后做什么",
    "对极简生活怎么看", "你怎么看待独居",
]
assert len(RECALL_DISTRACT_MESSAGES) == 35
```

#### 等待函数

```python
async def _wait_profile_landed(db: DbHandle, user_id: int, log: Logger) -> bool:
    """轮询 memory_profiles，30s 内出现行返回 True；超时返回 False。"""

async def _wait_summary_landed(db: DbHandle, user_id: int, log: Logger) -> bool:
    """同上，memory_summaries 表。"""

async def _wait_rag_chunk_landed(db: DbHandle, user_id: int, log: Logger) -> bool:
    """同上，chat_embeddings 表，至少 1 行即返回。"""
```

3 个 wait 函数都用 1s 轮询间隔、30s 总超时（对齐现有 `PROFILE_REFRESH_WAIT_SECONDS` / `SUMMARY_REFRESH_WAIT_SECONDS` / `RAG_INDEX_WAIT_SECONDS`）。

---

## 4. 错误处理 & FAIL 报告

骨架内 4 类失败的精确分类，每类直接定位到具体环节：

| 失败类型 | 触发条件 | FAIL detail |
|---|---|---|
| **plant 链路坏** | 5 条 plant 全发完但任何一条 `len(reply.bubbles) == 0` | `"plant phase: 第 X 条 AI 没回复气泡，主对话链路异常"` |
| **上游机制断** | `_wait_xxx_landed` 30s 超时 | `"profile/summary/rag 未在 30s 内落库（DB 行数: profiles=N, summaries=M, embeddings=K），记忆上游断了，不是召回问题"` |
| **distract phase 异常** | 35 条填充中任一条 WS 超时 / AI 没回复 | `"distract phase: 第 X 条无回复，跑测中断（已发送 X 条）"` |
| **召回失败（真正的回归）** | probe 后 AI 回复里没命中任何 expected_keyword | 见下方模板 |

**召回失败 detail 模板**（最重要，决定排查效率）：

```
召回失败: AI 回复未命中期望关键词
  AI 回复全文: "{reply.text}"
  期望关键词（任一命中即可）: {expected_keywords}
  上下文统计:
    - plant 之后总消息数: {plant_count + distract_count + 1 + ai_replies_count}
    - DB 状态: profiles={n_profile} summaries={n_summary} embeddings={n_emb}
    - profile.summary_text 长度: {len_chars}（None=无 profile）
    - 最新 summary 创建时间: {summary_created_at}
```

让 FAIL 时一眼看出是「机制落库了但召回没命中」还是「机制压根没落库」。

### 时间预算

每场景 ≈ plant (~30s) + wait (30s) + distract (~250s, 35 条 × 7s) + probe (~10s) ≈ **6 分钟**。
3 场景串行总 ≈ **20 分钟**，落在可接受 dogfood 时长内。

### 异常兜底

沿用现有 `_run_one_scenario` 的 try/except wrap：任何场景内未捕获异常 → `ScenarioResult(name, "FAIL", f"scenario 抛异常: {e!r}")`。骨架不重复 wrap。

---

## 5. 跟现有 4 场景的关系

现有 `profile / throttle / summary / rag` **保留不动**，继续跑在 `--scenario=all` 默认集。它们检查"机制落库"，是新场景的前置依赖检查——白盒先跑、黑盒后跑，故障从浅到深定位。

单独跑新场景：`./run_dogfood.sh --scenario=memory_recall_summary`（沿用现有 argparse 单 scenario 模式）。

---

## 6. 测试策略（dogfood 自身代码）

按 TDD 铁律，骨架 `run_memory_recall` 是新加的"业务逻辑"，必须有单元测试覆盖。

**新建 `server/scripts/dogfood/test_run_memory_recall.py`**，至少 6 条用例：

| 测试 | 验证 |
|---|---|
| `test_骨架按顺序调用 plant→wait→distract→probe` | mock 的 send 函数，assert 5 步调用顺序正确 |
| `test_wait_xxx_landed 超时返回 FAIL_上游断` | mock DB 始终返回空，assert FAIL detail 含"未在 30s 内落库" |
| `test_keyword_match_任一命中即 PASS` | reply="...绵阳..." + kw=["绵阳","四川"] → PASS |
| `test_keyword_全部未命中_返回 FAIL_召回失败` | reply="...北京..." + kw=["绵阳","四川"] → FAIL，detail 含 reply 全文 |
| `test_RECALL_DISTRACT_MESSAGES 长度=35` | 保护填充池数量 |
| `test_RECALL_DISTRACT_MESSAGES 不含场景关键词` | 用 expected_keywords 全集（绵阳/汤圆/寿司/成都/...）扫 35 条，必须 0 命中 |

**集成验证**：`python3 -m pytest server/scripts/dogfood/test_run_memory_recall.py` 全过 → 上 server `./run_dogfood.sh --scenario=memory_recall_summary` 端到端真跑确认。

---

## 7. 不纳入 CI

20 分钟跑测对 CI 太重。定位为**手动 dogfood**：发版前 + 改了 `MemoryApi` / `PromptBuilder` / `AiService` 后必跑。

---

## 8. 不做的事（YAGNI）

- 不改现有 4 个白盒场景的任何代码
- 不改 plan3 缩短阈值 override 机制
- 不加 LLM-as-judge 备用断言（关键词足够；未来若命中率不稳再加）
- 不并发跑 3 个场景（现有 SCENARIO_ORDER 串行模型够用）
- 不参数化 `distract_count`（35 写死，未来场景需要不同深度再开参数）
