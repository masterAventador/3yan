# Dogfood 记忆能力黑盒召回测试 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 给 dogfood 加 3 个端到端记忆召回测试场景（memory_recall_profile / memory_recall_summary / memory_recall_rag），验证 profile/summary/RAG 三种记忆机制在真实链路上能被 AI 召回。

**Architecture:** 在 `server/scripts/dogfood/dogfood_test.py` 加共享骨架 `run_memory_recall` + 3 个场景函数 + 35 条纯闲聊填充常量 + 公共 wait helper（参数化表名）。新建 `test_run_memory_recall.py` 用 pytest + monkeypatch 单测骨架核心逻辑（mock chat / clean / db）。3 场景串行接在现有 plan2 4 场景之后。

**Tech Stack:** Python 3 + asyncio + websockets（已有）+ pytest + pytest-asyncio（新加 dev 依赖）+ unittest.mock

**Spec 参考：** `docs/superpowers/specs/2026-05-21-dogfood-memory-recall-design.md`

---

## 前置准备

本地装 dev 依赖（dogfood_test.py 是 server 上跑的脚本，但单测在本地跑）：

```bash
pip install pytest pytest-asyncio PyJWT websockets
```

PyJWT / websockets dogfood_test.py 顶层 import 时需要，pytest / pytest-asyncio 跑单测需要。

---

### Task 1: 加 `RECALL_DISTRACT_MESSAGES` 常量 + 保护单测

**Files:**
- Create: `server/scripts/dogfood/test_run_memory_recall.py`
- Modify: `server/scripts/dogfood/dogfood_test.py`（在第 99 行 `INTIMACY_WAIT_SECONDS = 10.0` 之后插入新 section）

- [ ] **Step 1: 创建 test 文件，写针对常量的 2 条失败测试**

写入 `server/scripts/dogfood/test_run_memory_recall.py`：

```python
"""dogfood_test.py 内部 helper 的单元测试。

运行：cd server/scripts/dogfood && pytest test_run_memory_recall.py -v
"""

import sys
import pathlib

sys.path.insert(0, str(pathlib.Path(__file__).parent))
import dogfood_test as dt  # noqa: E402


# ---------------- RECALL_DISTRACT_MESSAGES 保护测试 ----------------

def test_distract_pool_长度恰好35():
    """填充池数量必须固定 35，骨架据此推 plant 出短期窗口 32 条。"""
    assert len(dt.RECALL_DISTRACT_MESSAGES) == 35


def test_distract_pool_不含场景关键词避免污染():
    """
    填充消息绝不能巧合提到任何场景的 expected_keywords，
    否则 AI 在 distract 阶段回应时若顺嘴提到，会假阳性 PASS 召回测试。
    """
    forbidden = [
        # profile 场景关键词
        "绵阳", "四川", "川", "杭州", "王莎莎", "汤圆",
        # summary 场景关键词
        "寿司", "刺身", "三文鱼",
        # rag 场景关键词
        "成都", "周三", "星期三", "出差", "春熙路",
    ]
    for msg in dt.RECALL_DISTRACT_MESSAGES:
        for word in forbidden:
            assert word not in msg, f"填充消息 '{msg}' 含场景关键词 '{word}'"
```

- [ ] **Step 2: 跑测试确认 FAIL**

```bash
cd server/scripts/dogfood && pytest test_run_memory_recall.py -v
```

Expected: 2 测试 FAIL，错误 `AttributeError: module 'dogfood_test' has no attribute 'RECALL_DISTRACT_MESSAGES'`

- [ ] **Step 3: 在 dogfood_test.py 加 RECALL_DISTRACT_MESSAGES 常量**

在 `server/scripts/dogfood/dogfood_test.py` 第 99 行（`INTIMACY_WAIT_SECONDS = 10.0`）之后、`# ----------------------------- log -----------------------------` 之前插入：

```python


# ----------------------------- 记忆召回测试常量 -----------------------------

# 召回测试用的"无关填充消息"池——35 条纯闲聊，绝不含场景关键词，
# 用来在 plant 之后把 plant 推出 SHORT_TERM_WINDOW_SIZE=32 短期窗口。
# 任何修改必须同步跑 test_distract_pool_不含场景关键词避免污染 防止污染。
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
```

- [ ] **Step 4: 跑测试确认 PASS**

```bash
cd server/scripts/dogfood && pytest test_run_memory_recall.py -v
```

Expected: 2 PASS

- [ ] **Step 5: Commit**

```bash
git add server/scripts/dogfood/dogfood_test.py server/scripts/dogfood/test_run_memory_recall.py
git commit -m "feat(dogfood): 加 RECALL_DISTRACT_MESSAGES 填充消息池 + 保护单测"
```

---

### Task 2: 加 `_wait_table_has_row` helper + 3 个 wait 包装函数

**Files:**
- Modify: `server/scripts/dogfood/dogfood_test.py`（新增 4 个 async 函数，放在 `clean_test_data` 之后）
- Modify: `server/scripts/dogfood/test_run_memory_recall.py`（追加 wait helper 测试）

骨架函数会调用 `post_plant_wait_fn` / `post_distract_wait_fn`，本任务把 3 个 wait 函数和共享的轮询 helper 准备好。

- [ ] **Step 1: 在 test 文件追加 wait helper 失败测试**

在 `test_run_memory_recall.py` 末尾追加：

```python
import asyncio
import pytest
from unittest.mock import MagicMock


# ---------------- _wait_table_has_row 单测 ----------------

@pytest.mark.asyncio
async def test_wait_table_has_row_首次查到行立即返回True(monkeypatch):
    """db.query 第一次就返回非空 → wait 立即 True，不 sleep。"""
    db = MagicMock()
    db.query.return_value = [["1"]]  # COUNT(*) = 1

    # 拦截 asyncio.sleep 防止真等
    sleep_calls = []
    async def fake_sleep(d):
        sleep_calls.append(d)
    monkeypatch.setattr(dt.asyncio, "sleep", fake_sleep)

    result = await dt._wait_table_has_row(db, "memory_profiles", 901, dt.Logger(False))

    assert result is True
    assert sleep_calls == []  # 首次命中，不该 sleep
    db.query.assert_called_once()


@pytest.mark.asyncio
async def test_wait_table_has_row_30s超时返回False(monkeypatch):
    """db.query 始终空 + 模拟 time 流逝 30s → 返回 False。"""
    db = MagicMock()
    db.query.return_value = [["0"]]  # COUNT(*) = 0

    # 用 fake monotonic 模拟时间流逝：每次调用 +5s
    counter = {"n": 0}
    def fake_monotonic():
        counter["n"] += 1
        return counter["n"] * 5.0  # 0, 5, 10, 15, ... 30s 后超时

    async def fake_sleep(d):
        pass  # 不真等

    monkeypatch.setattr(dt.time, "monotonic", fake_monotonic)
    monkeypatch.setattr(dt.asyncio, "sleep", fake_sleep)

    result = await dt._wait_table_has_row(db, "memory_profiles", 901, dt.Logger(False), timeout=30.0)

    assert result is False


@pytest.mark.asyncio
async def test_wait_3个包装函数指向正确的表名(monkeypatch):
    """_wait_profile_landed / _wait_summary_landed / _wait_rag_chunk_landed 应查对应表。"""
    called_tables = []

    async def fake_wait(db, table, user_id, log, timeout=30.0):
        called_tables.append(table)
        return True

    monkeypatch.setattr(dt, "_wait_table_has_row", fake_wait)

    db = MagicMock()
    log = dt.Logger(False)
    await dt._wait_profile_landed(db, 901, log)
    await dt._wait_summary_landed(db, 901, log)
    await dt._wait_rag_chunk_landed(db, 901, log)

    assert called_tables == ["memory_profiles", "memory_summaries", "chat_embeddings"]
```

- [ ] **Step 2: 跑测试确认 FAIL**

```bash
cd server/scripts/dogfood && pytest test_run_memory_recall.py -v
```

Expected: 3 新增测试 FAIL，错误 `AttributeError: module 'dogfood_test' has no attribute '_wait_table_has_row'`

- [ ] **Step 3: 在 dogfood_test.py 实现 4 个 async 函数**

在 `clean_test_data` 函数后面（第 338 行 `log.debug("[clean] 完成")` 之后）插入：

```python


# ----------------------------- 记忆召回 wait helper -----------------------------

async def _wait_table_has_row(
    db: DbHandle, table: str, user_id: int, log: Logger, timeout: float = 30.0,
) -> bool:
    """轮询 `SELECT COUNT(*) FROM <table> WHERE user_id = ?` 直到 > 0 或 timeout。

    1s 轮询间隔。首次查到立即返回 True，整个轮询超时返回 False。
    """
    start = time.monotonic()
    while True:
        rows = db.query(f"SELECT COUNT(*) FROM {table} WHERE user_id = {user_id}")
        if rows and int(rows[0][0]) > 0:
            log.debug(f"[wait] {table} user_id={user_id} 出现行")
            return True
        if time.monotonic() - start >= timeout:
            log.warn(f"[wait] {table} user_id={user_id} 超时 {timeout}s 未出现行")
            return False
        await asyncio.sleep(1)


async def _wait_profile_landed(db: DbHandle, user_id: int, log: Logger) -> bool:
    return await _wait_table_has_row(db, "memory_profiles", user_id, log)


async def _wait_summary_landed(db: DbHandle, user_id: int, log: Logger) -> bool:
    return await _wait_table_has_row(db, "memory_summaries", user_id, log)


async def _wait_rag_chunk_landed(db: DbHandle, user_id: int, log: Logger) -> bool:
    return await _wait_table_has_row(db, "chat_embeddings", user_id, log)
```

- [ ] **Step 4: 跑测试确认 PASS**

```bash
cd server/scripts/dogfood && pytest test_run_memory_recall.py -v
```

Expected: 5 PASS（Task 1 的 2 条 + Task 2 的 3 条）

- [ ] **Step 5: Commit**

```bash
git add server/scripts/dogfood/dogfood_test.py server/scripts/dogfood/test_run_memory_recall.py
git commit -m "feat(dogfood): 加 _wait_table_has_row helper + profile/summary/rag wait 包装"
```

---

### Task 3: 加 `run_memory_recall` 骨架函数 + 4 条单测

**Files:**
- Modify: `server/scripts/dogfood/dogfood_test.py`（新增 `run_memory_recall` async 函数 + 一个内部 helper `_match_keywords`，放在 wait helper 后）
- Modify: `server/scripts/dogfood/test_run_memory_recall.py`（追加 4 条骨架测试）

骨架按 spec §3.2 的 6 步执行：clean → plant → wait1 → distract → wait2 → probe → 关键词匹配。

- [ ] **Step 1: 在 test 文件追加 4 条失败测试**

在 `test_run_memory_recall.py` 末尾追加：

```python


# ---------------- run_memory_recall 骨架测试 ----------------

def _fake_wsreply(text: str) -> dt.WsReply:
    """构造一个带 1 个 AI 气泡的 fake WsReply。"""
    r = dt.WsReply()
    r.new_messages.append({"id": 1, "senderType": "AI", "content": text, "createdAt": "x"})
    return r


def _fake_empty_wsreply() -> dt.WsReply:
    """AI 没回任何气泡的 WsReply（链路异常）。"""
    return dt.WsReply()


@pytest.mark.asyncio
async def test_run_memory_recall_6步按序执行(monkeypatch):
    """骨架按 clean → plant_chat → wait1 → distract_chat → wait2 → probe_chat 顺序执行。"""
    call_order = []

    def fake_clean(db, user_id, log):
        call_order.append("clean")

    async def fake_chat(token, contents, log, wait_between=0.5):
        if contents == ["plant1"]:
            call_order.append("plant_chat")
        elif contents == ["d1", "d2"]:
            call_order.append("distract_chat")
        elif contents == ["probe"]:
            call_order.append("probe_chat")
        return [_fake_wsreply("AI 回 含 关键词") for _ in contents]

    async def fake_wait1(db, user_id, log):
        call_order.append("wait1")
        return True

    async def fake_wait2(db, user_id, log):
        call_order.append("wait2")
        return True

    monkeypatch.setattr(dt, "clean_test_data", fake_clean)
    monkeypatch.setattr(dt, "chat", fake_chat)
    monkeypatch.setattr(dt, "RECALL_DISTRACT_MESSAGES", ["d1", "d2", "d3"])  # 缩小让 distract_count=2 取前 2 条

    result = await dt.run_memory_recall(
        scenario_name="test_scenario",
        plant_messages=["plant1"],
        post_plant_wait_fn=fake_wait1,
        distract_count=2,
        post_distract_wait_fn=fake_wait2,
        probe_message="probe",
        expected_keywords=["关键词"],
        token="t",
        db=MagicMock(),
        user_id=900,
        character_id=1,
        log=dt.Logger(False),
    )

    assert call_order == ["clean", "plant_chat", "wait1", "distract_chat", "wait2", "probe_chat"]
    assert result.status == "PASS"


@pytest.mark.asyncio
async def test_run_memory_recall_关键词命中PASS(monkeypatch):
    """probe AI 回复中含任一 expected_keyword → PASS。"""
    monkeypatch.setattr(dt, "clean_test_data", lambda *a, **kw: None)

    async def fake_chat(token, contents, log, wait_between=0.5):
        return [_fake_wsreply("我记得你说过绵阳的事") for _ in contents]

    monkeypatch.setattr(dt, "chat", fake_chat)
    monkeypatch.setattr(dt, "RECALL_DISTRACT_MESSAGES", ["d"])

    result = await dt.run_memory_recall(
        scenario_name="test",
        plant_messages=["plant"],
        post_plant_wait_fn=None,
        distract_count=1,
        post_distract_wait_fn=None,
        probe_message="probe",
        expected_keywords=["绵阳", "四川"],  # "绵阳" 命中
        token="t", db=MagicMock(), user_id=900, character_id=1, log=dt.Logger(False),
    )

    assert result.status == "PASS"
    assert "绵阳" in result.detail


@pytest.mark.asyncio
async def test_run_memory_recall_关键词全未命中FAIL(monkeypatch):
    """probe AI 回复完全不含任何 expected_keyword → FAIL，detail 含 reply 全文。"""
    monkeypatch.setattr(dt, "clean_test_data", lambda *a, **kw: None)

    async def fake_chat(token, contents, log, wait_between=0.5):
        return [_fake_wsreply("我不太记得了，你说说看") for _ in contents]

    monkeypatch.setattr(dt, "chat", fake_chat)
    monkeypatch.setattr(dt, "RECALL_DISTRACT_MESSAGES", ["d"])

    result = await dt.run_memory_recall(
        scenario_name="test",
        plant_messages=["plant"],
        post_plant_wait_fn=None,
        distract_count=1,
        post_distract_wait_fn=None,
        probe_message="probe",
        expected_keywords=["绵阳", "四川"],
        token="t", db=MagicMock(), user_id=900, character_id=1, log=dt.Logger(False),
    )

    assert result.status == "FAIL"
    assert "我不太记得了" in result.detail  # AI 回复全文出现在 detail
    assert "绵阳" in result.detail  # expected keywords 列表也在 detail


@pytest.mark.asyncio
async def test_run_memory_recall_wait_fn返回False立即FAIL(monkeypatch):
    """post_plant_wait_fn 返回 False（落库超时）→ 骨架直接 FAIL，不进 distract。"""
    monkeypatch.setattr(dt, "clean_test_data", lambda *a, **kw: None)

    chat_call_count = {"n": 0}

    async def fake_chat(token, contents, log, wait_between=0.5):
        chat_call_count["n"] += 1
        return [_fake_wsreply("OK") for _ in contents]

    async def fake_wait_timeout(db, user_id, log):
        return False  # 落库失败

    monkeypatch.setattr(dt, "chat", fake_chat)

    result = await dt.run_memory_recall(
        scenario_name="test",
        plant_messages=["plant"],
        post_plant_wait_fn=fake_wait_timeout,
        distract_count=1,
        post_distract_wait_fn=None,
        probe_message="probe",
        expected_keywords=["OK"],
        token="t", db=MagicMock(), user_id=900, character_id=1, log=dt.Logger(False),
    )

    assert result.status == "FAIL"
    assert "未在 30s 内落库" in result.detail or "上游" in result.detail
    assert chat_call_count["n"] == 1  # 只发了 plant，没进 distract / probe
```

- [ ] **Step 2: 跑测试确认 FAIL**

```bash
cd server/scripts/dogfood && pytest test_run_memory_recall.py -v
```

Expected: 4 新增 FAIL，错误 `AttributeError: module 'dogfood_test' has no attribute 'run_memory_recall'`

- [ ] **Step 3: 在 dogfood_test.py 实现 `run_memory_recall` 骨架**

在 `_wait_rag_chunk_landed` 函数后插入：

```python


# ----------------------------- 记忆召回测试骨架 -----------------------------

WaitFn = Callable[[DbHandle, int, Logger], "asyncio.Future[bool]"]


async def run_memory_recall(
    scenario_name: str,
    plant_messages: list[str],
    post_plant_wait_fn: Optional[Callable],
    distract_count: int,
    post_distract_wait_fn: Optional[Callable],
    probe_message: str,
    expected_keywords: list[str],
    token: str,
    db: DbHandle,
    user_id: int,
    character_id: int,
    log: Logger,
) -> ScenarioResult:
    """端到端记忆召回测试骨架。6 步：clean → plant → wait1 → distract → wait2 → probe。

    spec: docs/superpowers/specs/2026-05-21-dogfood-memory-recall-design.md
    """
    # Step 1: clean
    clean_test_data(db, user_id, log)

    # Step 2: plant
    log.info(f"    [plant] 发 {len(plant_messages)} 条事实植入消息")
    plant_replies = await chat(token, plant_messages, log)
    for i, r in enumerate(plant_replies):
        if not r.new_messages:
            return ScenarioResult(
                scenario_name, "FAIL",
                f"plant phase: 第 {i+1} 条 AI 没回复气泡，主对话链路异常",
            )

    # Step 3: wait1（plant 后）
    if post_plant_wait_fn is not None:
        log.info(f"    [wait1] 等记忆机制落库（30s 上限）")
        ok = await post_plant_wait_fn(db, user_id, log)
        if not ok:
            n_profile = _count_rows(db, "memory_profiles", user_id)
            n_summary = _count_rows(db, "memory_summaries", user_id)
            n_emb = _count_rows(db, "chat_embeddings", user_id)
            return ScenarioResult(
                scenario_name, "FAIL",
                f"上游机制未在 30s 内落库（DB 行数: profiles={n_profile}, "
                f"summaries={n_summary}, embeddings={n_emb}），记忆上游断了，不是召回问题",
            )

    # Step 4: distract
    distract_msgs = RECALL_DISTRACT_MESSAGES[:distract_count]
    log.info(f"    [distract] 发 {len(distract_msgs)} 条无关消息推 plant 出短期窗口")
    distract_replies = await chat(token, distract_msgs, log)
    for i, r in enumerate(distract_replies):
        if not r.new_messages:
            return ScenarioResult(
                scenario_name, "FAIL",
                f"distract phase: 第 {i+1} 条无回复，跑测中断（已发送 {i+1} 条）",
            )

    # Step 5: wait2（distract 后，summary 场景用）
    if post_distract_wait_fn is not None:
        log.info(f"    [wait2] 等 distract 触发的机制落库（30s 上限）")
        ok = await post_distract_wait_fn(db, user_id, log)
        if not ok:
            n_summary = _count_rows(db, "memory_summaries", user_id)
            return ScenarioResult(
                scenario_name, "FAIL",
                f"上游机制（distract 后）未在 30s 内落库（DB 行数: summaries={n_summary}）",
            )

    # Step 6: probe
    log.info(f"    [probe] 发提问消息，检查 AI 是否召回")
    probe_replies = await chat(token, [probe_message], log)
    if not probe_replies or not probe_replies[0].new_messages:
        return ScenarioResult(
            scenario_name, "FAIL",
            "probe phase: AI 没回复气泡，无法判断召回",
        )

    reply_text = probe_replies[0].ai_concat
    hit = next((kw for kw in expected_keywords if kw in reply_text), None)
    if hit:
        return ScenarioResult(
            scenario_name, "PASS",
            f"AI 回复命中关键词 '{hit}'；回复全文: {reply_text!r}",
        )

    # FAIL: 召回失败，打详细上下文
    n_profile = _count_rows(db, "memory_profiles", user_id)
    n_summary = _count_rows(db, "memory_summaries", user_id)
    n_emb = _count_rows(db, "chat_embeddings", user_id)
    return ScenarioResult(
        scenario_name, "FAIL",
        f"召回失败: AI 回复未命中期望关键词\n"
        f"  AI 回复全文: {reply_text!r}\n"
        f"  期望关键词（任一命中即可）: {expected_keywords}\n"
        f"  上下文统计: DB 行数 profiles={n_profile} summaries={n_summary} embeddings={n_emb}",
    )


def _count_rows(db: DbHandle, table: str, user_id: int) -> int:
    """辅助：查表里 user_id 对应行数，错误时返回 -1（不阻断 FAIL detail 生成）。"""
    try:
        rows = db.query(f"SELECT COUNT(*) FROM {table} WHERE user_id = {user_id}")
        return int(rows[0][0]) if rows else 0
    except Exception:
        return -1
```

注意：`Callable` / `Optional` 已在文件顶部 `from typing import Any, Callable, Optional`（行 38），无需补 import。

- [ ] **Step 4: 跑测试确认 PASS**

```bash
cd server/scripts/dogfood && pytest test_run_memory_recall.py -v
```

Expected: 9 PASS（Task 1 的 2 + Task 2 的 3 + Task 3 的 4）

- [ ] **Step 5: Commit**

```bash
git add server/scripts/dogfood/dogfood_test.py server/scripts/dogfood/test_run_memory_recall.py
git commit -m "feat(dogfood): 加 run_memory_recall 骨架函数 + 4 条单测覆盖 6 步流程"
```

---

### Task 4: 加 3 个场景函数 + 注册到 SCENARIO_REGISTRY / SCENARIO_ORDER / SCENARIO_USER_IDS

**Files:**
- Modify: `server/scripts/dogfood/dogfood_test.py`（新增 3 个 `run_memory_recall_*` 函数 + 修改 `SCENARIO_REGISTRY` (行 1798) / `SCENARIO_ORDER` (行 1818) / `SCENARIO_USER_IDS` (行 1838)）

3 个场景函数都是 thin wrapper，调用骨架并塞 spec §3.3 定义的参数。命名沿用现有 `run_<scenario_name>` 规范（不是 `scenario_*`），匹配 `SCENARIO_REGISTRY` 已有条目（如 `"profile": run_profile`）。本任务不需要单测（thin wrapper 无逻辑），由 Task 5 端到端验证。

- [ ] **Step 1: 在 dogfood_test.py 加 3 个场景函数**

在 `SCENARIO_REGISTRY` 定义之前（行 ~1796）、紧接最后一个现有 plan3 场景函数 `run_plan3_stage_prompt` 之后插入：

```python


# ----------------------------- 记忆召回 3 个场景 -----------------------------

async def run_memory_recall_profile(
    token: str, db: DbHandle, user_id: int, character_id: int, log: Logger,
) -> ScenarioResult:
    """验证 profile 抽取：发暴露身份的消息 → 等 profile 落库 → distract 35 条挤出窗口 → 问老家。"""
    return await run_memory_recall(
        scenario_name="memory_recall_profile",
        plant_messages=[
            "我叫王莎莎，今年27岁，在杭州做 Java 后端",
            "我老家是四川绵阳的，去年才来杭州工作",
            "我特别怕辣，川菜里只敢吃糖醋排骨那种",
            "周末喜欢打羽毛球，水平业余但能赢我同事",
            "我有只叫汤圆的英短猫，今年3岁了",
        ],
        post_plant_wait_fn=_wait_profile_landed,
        distract_count=35,
        post_distract_wait_fn=None,
        probe_message="对了 你还记得我老家是哪里的吗？",
        expected_keywords=["绵阳", "四川", "川"],
        token=token, db=db, user_id=user_id, character_id=character_id, log=log,
    )


async def run_memory_recall_summary(
    token: str, db: DbHandle, user_id: int, character_id: int, log: Logger,
) -> ScenarioResult:
    """验证 summary：发吃坏肚子事件 → distract 35 条触发 summary → 等落库 → 问哪家店。"""
    return await run_memory_recall(
        scenario_name="memory_recall_summary",
        plant_messages=[
            "今天去公司楼下新开的那家寿司店吃饭",
            "结果点的三文鱼刺身吃完拉肚子拉了一下午",
            "下次绝对不去那家了",
        ],
        post_plant_wait_fn=None,
        distract_count=35,
        post_distract_wait_fn=_wait_summary_landed,
        probe_message="前阵子我说过哪家店让我吃坏肚子来着？",
        expected_keywords=["寿司", "刺身", "三文鱼"],
        token=token, db=db, user_id=user_id, character_id=character_id, log=log,
    )


async def run_memory_recall_rag(
    token: str, db: DbHandle, user_id: int, character_id: int, log: Logger,
) -> ScenarioResult:
    """验证 RAG 语义召回：发出差事件 → 等 chunk 入库 → distract 35 条 → 间接问哪天出差。"""
    return await run_memory_recall(
        scenario_name="memory_recall_rag",
        plant_messages=[
            "下周三我要去成都出差，跟一个甲方碰需求",
            "项目是给某个银行做风控系统对接",
            "本来不想去，但项目经理逼我",
            "在成都待 3 天，周五晚上飞回来",
            "酒店订的是春熙路那边",
        ],
        post_plant_wait_fn=_wait_rag_chunk_landed,
        distract_count=35,
        post_distract_wait_fn=None,
        probe_message="提醒一下，我下周哪天要出差？",
        expected_keywords=["周三", "星期三", "下周三"],
        token=token, db=db, user_id=user_id, character_id=character_id, log=log,
    )
```

- [ ] **Step 2: 修改 SCENARIO_REGISTRY（行 1798）加 3 行**

把 `SCENARIO_REGISTRY` 里 plan2 段改成：

```python
SCENARIO_REGISTRY: dict[str, Callable] = {
    # ---- Plan 2 场景（保持不变）----
    "profile": run_profile,
    "throttle": run_throttle,
    "summary": run_summary,
    "rag": run_rag,
    # ---- 记忆召回（plan2 端到端增强）----
    "memory_recall_profile": run_memory_recall_profile,
    "memory_recall_summary": run_memory_recall_summary,
    "memory_recall_rag": run_memory_recall_rag,
    # ---- Plan 3 场景（10 个）----
    "plan3_baseline": run_plan3_baseline,
    ... (其余 plan3 行保持不变)
}
```

- [ ] **Step 3: 修改 SCENARIO_ORDER（行 1818）**

把：

```python
SCENARIO_ORDER = ["profile", "throttle", "summary", "rag"]
```

改成：

```python
SCENARIO_ORDER = [
    "profile", "throttle", "summary", "rag",
    "memory_recall_profile",
    "memory_recall_summary",
    "memory_recall_rag",
]
```

- [ ] **Step 4: 修改 SCENARIO_USER_IDS（行 1838-1855）加 3 行**

把 Plan 2 区块改成：

```python
SCENARIO_USER_IDS = {
    # Plan 2
    "profile": 901,
    "throttle": 902,
    "summary": 903,
    "rag": 904,
    # 记忆召回（plan2 端到端增强）
    "memory_recall_profile": 905,
    "memory_recall_summary": 906,
    "memory_recall_rag": 907,
    # Plan 3（910-919）
    "plan3_baseline": 910,
    ... (其余 plan3 行保持不变)
}
```

- [ ] **Step 5: 跑现有单测确认没破坏 + 检查 Python 语法**

```bash
cd server/scripts/dogfood && pytest test_run_memory_recall.py -v && python3 -c "import dogfood_test; print('imports OK')"
```

Expected: 9 PASS + `imports OK`（验证 dogfood_test.py 整文件 Python 语法正确，import 3 个新场景函数也都能 resolve）

- [ ] **Step 6: Commit**

```bash
git add server/scripts/dogfood/dogfood_test.py
git commit -m "feat(dogfood): 加 3 个记忆召回场景并注册到 SCENARIO_REGISTRY/SCENARIO_ORDER"
```

---

### Task 5: 端到端真跑验证 + 收尾

3 个场景共 ~20 分钟，在 server 上真跑确认整体可用。这是手动验证步骤，不写单测。

- [ ] **Step 1: 单跑 memory_recall_summary（最有把握的场景）做 smoke**

```bash
cd /Users/aventador/code/3yan && ./server/scripts/dogfood/run_dogfood.sh --scenario=memory_recall_summary -v
```

Expected: 整个场景跑完（~7 分钟）→ 最后输出 `[memory_recall_summary] PASS`，detail 含命中关键词「寿司/刺身/三文鱼」之一。

如果 FAIL：按 detail 分类排查：
- "plant phase: AI 没回复" → 服务器主链路问题，跟本功能无关
- "上游机制（distract 后）未在 30s 内落库" → summary 触发本身坏了，去查 SummaryScheduler 日志
- "召回失败: ..." → AI 拿到 summary 了但回复没提到关键词，看 AI 回复全文 + 看 prompt 注入日志

- [ ] **Step 2: 跑 memory_recall_profile**

```bash
./server/scripts/dogfood/run_dogfood.sh --scenario=memory_recall_profile -v
```

Expected: PASS，detail 命中「绵阳/四川/川」之一。

- [ ] **Step 3: 跑 memory_recall_rag**

```bash
./server/scripts/dogfood/run_dogfood.sh --scenario=memory_recall_rag -v
```

Expected: PASS，detail 命中「周三/星期三/下周三」之一。

⚠️ RAG 是 3 个场景里最不稳的——cos sim 阈值 0.6 + LLM 回答方式不确定。如果偶尔 FAIL（比如 AI 回复用了"中间那天"而没说"周三"），可以补关键词列表（如加"中"或微调 probe 文本让 AI 必须正面回答日期）。

- [ ] **Step 4: 完整 all 跑一次 verify 没破坏现有场景**

```bash
./server/scripts/dogfood/run_dogfood.sh --scenario=all
```

Expected: 7 个场景按顺序 PASS（profile / throttle / summary / rag / memory_recall_profile / memory_recall_summary / memory_recall_rag）。总时长 ~25 分钟。

- [ ] **Step 5: 如有微调需求（关键词补充 / probe 措辞）做最后一次改动 commit**

```bash
git add server/scripts/dogfood/dogfood_test.py
git commit -m "tune(dogfood): 端到端验证后微调召回关键词/probe 措辞"
```

如果不需要任何微调，跳过此 step。

- [ ] **Step 6: 推到 origin**

```bash
git push origin master
```

---

## 整体完成定义

- 4 个 commit（Task 1-4）+ 可选 1 个微调 commit（Task 5 step 5）
- `pytest server/scripts/dogfood/test_run_memory_recall.py` 9 PASS
- `./run_dogfood.sh --scenario=all` 7 场景全 PASS
- 推到 origin/master
