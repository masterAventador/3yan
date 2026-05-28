# Plan 4 主动消息系统 dogfood 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 给 `dogfood_test.py` 加 `--plan4` 模式，4 分钟内端到端跑完 a/b/c/d 四个主动消息场景，覆盖触发器 → 调度 → 门控 → 生成 → 投递 → DB 状态全链路。

**Architecture:** 一行生产代码改（`RecallTrigger.fixedDelay` 改 `fixedDelayString` 让间隔可配置）+ 新建 `plan4_env_override.conf` 缩短 cron/threshold/scan interval + dogfood_test.py 加 4 个 scenario function + run_dogfood.sh 加 `--plan4` flag 走 apply/rollback env + 重启路径（与现有 `--plan3` 平行）。

**Tech Stack:** Java 21 + Spring Boot 3.x（生产代码），Python 3.13 + asyncio + websockets + PyJWT（dogfood 脚本），bash（wrapper）。

**Spec：** `docs/superpowers/specs/2026-05-29-plan-4-dogfood-design.md`

---

## File Structure

**生产代码改动（2 个文件）：**
- `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/RecallTrigger.java` — `@Scheduled` 注解改可配置
- `server/bootstrap/src/main/resources/application.yml` — 加 `sanyan.proactive.recall.scan-interval-ms` 默认值

**测试新增（1 个文件）：**
- `server/bootstrap/src/test/java/com/sanyan/bootstrap/RecallTriggerScheduledIntervalIT.java` — 验 SpEL 解析

**新建 dogfood 文件（1 个）：**
- `server/scripts/dogfood/plan4_env_override.conf`

**修改 dogfood 文件（2 个）：**
- `server/scripts/dogfood/dogfood_test.py` — 加 helpers + 4 scenarios + argparse 注册
- `server/scripts/dogfood/run_dogfood.sh` — 加 `--plan4` flag

---

## Task 1：RecallTrigger fixedDelay 改可配置

**Files:**
- Modify: `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/RecallTrigger.java`
- Modify: `server/bootstrap/src/main/resources/application.yml`
- Create: `server/bootstrap/src/test/java/com/sanyan/bootstrap/RecallTriggerScheduledIntervalIT.java`

- [ ] **Step 1：写失败的 IT**

文件 `server/bootstrap/src/test/java/com/sanyan/bootstrap/RecallTriggerScheduledIntervalIT.java`：

```java
package com.sanyan.bootstrap;

import com.sanyan.common.test.PostgresTestcontainerSupport;
import com.sanyan.proactive.trigger.RecallTrigger;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.core.env.Environment;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase.Replace.NONE;

/**
 * 防止 {@code sanyan.proactive.recall.scan-interval-ms} 配置 key 被 typo 后 SpEL
 * 解析失败回退到默认（或抛异常）—— 该 key 是 dogfood plan4 env override 的核心入口。
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureTestDatabase(replace = NONE)
class RecallTriggerScheduledIntervalIT extends PostgresTestcontainerSupport {

    @DynamicPropertySource
    static void overrideProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", PostgresTestcontainerSupport::jdbcUrl);
        registry.add("spring.datasource.username", PostgresTestcontainerSupport::username);
        registry.add("spring.datasource.password", PostgresTestcontainerSupport::password);
        registry.add("spring.datasource.driver-class-name", () -> "org.postgresql.Driver");
        registry.add("spring.flyway.enabled", () -> "true");
        registry.add("spring.flyway.locations", () -> "classpath:db/migration");
        registry.add("spring.autoconfigure.exclude", () ->
                "org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,"
                        + "org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration");
        registry.add("sanyan.jwt.secret", () ->
                "test-secret-key-at-least-256-bits-long-for-hmac-sha-testing");
        registry.add("sanyan.jwt.expiration-days", () -> "1");
        registry.add("sanyan.doubao.api-key", () -> "test");
        registry.add("sanyan.doubao.model", () -> "test");
        registry.add("sanyan.doubao.base-url", () -> "http://localhost:9999");
        registry.add("sanyan.cos.secret-id", () -> "test");
        registry.add("sanyan.cos.secret-key", () -> "test");
        registry.add("sanyan.cos.bucket", () -> "test-bucket");
        registry.add("sanyan.cos.region", () -> "ap-guangzhou");
        registry.add("sanyan.tts.enabled", () -> "false");
        registry.add("sanyan.tts.app-id", () -> "test");
        registry.add("sanyan.tts.access-token", () -> "test");
    }

    @Autowired
    private Environment environment;

    @Test
    void scan_interval_ms_property_should_resolve_to_positive_value() {
        String raw = environment.getProperty("sanyan.proactive.recall.scan-interval-ms");
        assertThat(raw)
                .as("yml 必须显式定义 sanyan.proactive.recall.scan-interval-ms，"
                        + "否则 dogfood plan4 env override 改不到 RecallTrigger 扫描间隔")
                .isNotNull();
        assertThat(Long.parseLong(raw))
                .as("默认扫描间隔应为正整数 ms")
                .isPositive();
    }

    @Test
    void recall_trigger_scan_method_should_use_fixed_delay_string() throws Exception {
        Method scan = RecallTrigger.class.getDeclaredMethod("scan");
        Scheduled annotation = scan.getAnnotation(Scheduled.class);
        assertThat(annotation).as("RecallTrigger.scan 必须有 @Scheduled 注解").isNotNull();
        assertThat(annotation.fixedDelayString())
                .as("RecallTrigger.scan 必须用 fixedDelayString 走 property，"
                        + "硬编码 fixedDelay 会让 dogfood env override 失效")
                .contains("sanyan.proactive.recall.scan-interval-ms");
    }
}
```

- [ ] **Step 2：跑测试确认 RED**

```
export JAVA_HOME="$(/usr/libexec/java_home)"
mvn -pl bootstrap -am install -DskipTests -q
mvn -pl bootstrap failsafe:integration-test failsafe:verify \
    -Dit.test='RecallTriggerScheduledIntervalIT' -DfailIfNoTests=false 2>&1 | tail -15
```

预期：两个测试都 FAIL，第一个因为 yml 没定义该 property，第二个因为注解还是 `fixedDelay` 不是 `fixedDelayString`。

- [ ] **Step 3：改 RecallTrigger 注解**

文件 `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/RecallTrigger.java` 找到这一行：

```java
@Scheduled(fixedDelay = 15 * 60 * 1000)   // 15min
```

改为：

```java
@Scheduled(fixedDelayString = "${sanyan.proactive.recall.scan-interval-ms:900000}")
```

注释直接删掉（`900000` 默认值已自解释，且本身 SpEL 表达式含义清晰）。

- [ ] **Step 4：加 yml 默认值**

文件 `server/bootstrap/src/main/resources/application.yml` 找到 `sanyan.proactive.recall` 段（应已存在 `thresholds-hours`），加 `scan-interval-ms`：

```yaml
sanyan:
  proactive:
    recall:
      scan-interval-ms: 900000        # 15min；可被 SANYAN_PROACTIVE_RECALL_SCANINTERVALMS env 覆盖
      thresholds-hours: [24, 72, 168]
```

如果原来 `recall:` 下只有 `thresholds-hours`，新增 `scan-interval-ms` 这一行即可，**两个 key 顺序无要求**。

- [ ] **Step 5：跑测试确认 GREEN**

```
mvn -pl bootstrap -am install -DskipTests -q
mvn -pl bootstrap failsafe:integration-test failsafe:verify \
    -Dit.test='RecallTriggerScheduledIntervalIT' -DfailIfNoTests=false 2>&1 | tail -15
```

预期：两个测试都 PASS。

- [ ] **Step 6：proactive-core 全包测试回归**

```
mvn -pl business_packages/sanyan-proactive-core test 2>&1 | grep -E "Tests run|BUILD" | tail -5
```

预期：`Tests run: 65, Failures: 0, Errors: 0, Skipped: 0` + `BUILD SUCCESS`（数字可能因新单测略增）。

- [ ] **Step 7：ProactiveFlowE2EIT 回归**

```
mvn -pl bootstrap failsafe:integration-test failsafe:verify \
    -Dit.test='ProactiveFlowE2EIT' -DfailIfNoTests=false 2>&1 | grep -E "Tests run|BUILD" | tail -5
```

预期：`Tests run: 2, Failures: 0` + `BUILD SUCCESS`。

- [ ] **Step 8：提交**

```bash
cd /Users/aventador/code/3yan/server
git add business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/RecallTrigger.java \
        bootstrap/src/main/resources/application.yml \
        bootstrap/src/test/java/com/sanyan/bootstrap/RecallTriggerScheduledIntervalIT.java
git commit -m "$(cat <<'EOF'
refactor(proactive): RecallTrigger 扫描间隔改可配置（默认值不变）

@Scheduled(fixedDelay = 15 * 60 * 1000) → fixedDelayString = "${sanyan.proactive.recall.scan-interval-ms:900000}"
application.yml 同步加 sanyan.proactive.recall.scan-interval-ms: 900000 显式默认值。

线上行为零变化（默认值 900000ms = 原硬编码 15min）。变更目的：让 plan4 dogfood
能通过 env override 把扫描间隔缩到秒级，否则 RecallTrigger 路径无法在测试时间窗内验证。

加 RecallTriggerScheduledIntervalIT 防 typo：① yml property 必须解析到正整数 ms
② @Scheduled.fixedDelayString 必须包含 property key。
EOF
)"
```

---

## Task 2：plan4_env_override.conf 配置文件

**Files:**
- Create: `server/scripts/dogfood/plan4_env_override.conf`

- [ ] **Step 1：新建文件**

文件 `server/scripts/dogfood/plan4_env_override.conf` 完整内容：

```
# Plan 4 主动消息系统 dogfood 阈值覆盖配置
# 由 run_dogfood.sh --plan4 追加到 /etc/3yan/3yan-server.env，测试结束后自动回滚。
#
# Spring relaxed-binding：环境变量大写下划线对应 yml kebab-case
# （如 SANYAN_PROACTIVE_GREETING_MORNINGCRON ↔ sanyan.proactive.greeting.morning-cron）。
#
# 原值 → 覆盖后：
#   morning-cron / night-cron: 0 0 8 / 0 30 22 * * *  →  */10 / */15 * * * * *  （每 10s/15s）
#   recall.scan-interval-ms:   900000               →  5000   （15min → 5s）
#   recall.thresholds-hours:   [24, 72, 168]        →  1,2,3  （24h/3d/7d → 1h/2h/3h；
#                                                              ProactiveProperties.thresholdsHours
#                                                              是 List<Integer>，必须整数）
#   quiet-hours.start / end:   23 / 8               →  0 / 0  （关免打扰，避免被 23-8 区间拦）
#   scatter-window-minutes:    30                   →  0      （早晚安不再随机延后）

SANYAN_PROACTIVE_GREETING_MORNINGCRON=*/10 * * * * *
SANYAN_PROACTIVE_GREETING_NIGHTCRON=*/15 * * * * *
SANYAN_PROACTIVE_RECALL_SCANINTERVALMS=5000
SANYAN_PROACTIVE_RECALL_THRESHOLDSHOURS=1,2,3
SANYAN_PROACTIVE_QUIETHOURS_START=0
SANYAN_PROACTIVE_QUIETHOURS_END=0
SANYAN_PROACTIVE_SCATTERWINDOWMINUTES=0
```

- [ ] **Step 2：人工对照 plan3_env_override.conf 风格**

打开 `server/scripts/dogfood/plan3_env_override.conf` 对照，确认本文件采用相同的注释风格（设计逻辑写在文件头注释里）。

- [ ] **Step 3：提交**

```bash
cd /Users/aventador/code/3yan/server
git add scripts/dogfood/plan4_env_override.conf
git commit -m "feat(dogfood): plan4_env_override.conf 缩短 cron/threshold/scan interval

由 run_dogfood.sh --plan4 在测试前追加进 /etc/3yan/3yan-server.env，
测试结束 trap rollback。覆盖项：早晚安 cron → */10/15s、recall 扫描 15min → 5s、
recall thresholds 24h/3d/7d → 1h/2h/3h、关 quiet-hours、关 scatter window。"
```

---

## Task 3：dogfood_test.py 加 plan4 helpers

**Files:**
- Modify: `server/scripts/dogfood/dogfood_test.py`

helpers 集中加在文件中部（常量区之后、scenario 函数之前），用 `# ---- Plan 4 helpers ----` 分隔。

- [ ] **Step 1：找到插入位置**

打开 `server/scripts/dogfood/dogfood_test.py`，找到 `# ---- 常量 -----` 段结束位置（约第 110 行附近），或现有 helper 函数（如 `mint_jwt`、`validate_dogfood_user_id`）旁边。在所有 scenario 函数（`scenario_xxx` 或 `run_xxx`）之前插入新代码块。

- [ ] **Step 2：加 DB 轮询 helper：等 memory_item / events_pending 出现**

```python
# ---- Plan 4 helpers ----

def wait_for_memory_item(db: DbHandle, user_id: int, kind: str,
                          timeout_s: float = 30.0,
                          poll_interval_s: float = 0.5) -> Optional[int]:
    """轮询 memory_item，等指定 kind 的 PENDING 行出现，返回 itemId 或 None（超时）。"""
    deadline = time.monotonic() + timeout_s
    while time.monotonic() < deadline:
        rows = db.query(
            f"SELECT id FROM memory_item "
            f"WHERE user_id = {user_id} AND kind = '{kind}' "
            f"AND status = 'PENDING' ORDER BY id DESC LIMIT 1"
        )
        if rows:
            return int(rows[0][0])
        time.sleep(poll_interval_s)
    return None


def wait_for_events_pending(db: DbHandle, user_id: int, event_type: str,
                             status: str = "scheduled",
                             timeout_s: float = 30.0,
                             poll_interval_s: float = 0.5) -> Optional[int]:
    """轮询 events_pending，等指定 event_type 的 status 行出现，返回 eventId 或 None（超时）。"""
    deadline = time.monotonic() + timeout_s
    while time.monotonic() < deadline:
        rows = db.query(
            f"SELECT id FROM events_pending "
            f"WHERE user_id = {user_id} AND event_type = '{event_type}' "
            f"AND status = '{status}' ORDER BY id DESC LIMIT 1"
        )
        if rows:
            return int(rows[0][0])
        time.sleep(poll_interval_s)
    return None
```

- [ ] **Step 3：加 scheduled_at 提前 helper（c/d 场景用）**

```python
def nudge_event_scheduled_at_now(db: DbHandle, event_id: int) -> None:
    """把 events_pending.scheduled_at 改成 NOW()，让 scheduler 下一轮（≤5s）立即领取。"""
    db.execute(f"UPDATE events_pending SET scheduled_at = NOW() WHERE id = {event_id}")
```

- [ ] **Step 4：加 last_active Redis 注入 helper（b 场景用）**

```python
def set_last_active_hours_ago(user_id: int, hours_ago: float) -> None:
    """直接在 Redis 写 user:last_active:<uid>，模拟用户已离线指定小时数。

    格式必须与 RecallTrigger 读取格式一致：ISO-8601 含时区（如 2026-05-29T12:34:56Z）。
    """
    ts = datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(hours=hours_ago)
    iso = ts.strftime("%Y-%m-%dT%H:%M:%SZ")
    redis_cmd("SET", f"user:last_active:{user_id}", iso)
```

> 注意：`redis_cmd` 应已存在于 dogfood_test.py（plan2/plan3 用过）。若不存在，先确认 helper 命名后照搬。

- [ ] **Step 5：加 relationship plant helper（c/d/a/b 都用）**

```python
def plant_relationship_at_stage(db: DbHandle, user_id: int, character_id: int,
                                 stage: int, intimacy: int) -> None:
    """直接插一行 relationships，绕过 IntimacyService 阶梯逻辑。

    参数 intimacy 必须与 stage 区间一致（dogfood 不靠它做业务推进，只为表非空约束）。
    """
    db.execute(
        f"INSERT INTO relationships (user_id, character_id, current_stage, "
        f"current_intimacy, last_message_at) "
        f"VALUES ({user_id}, {character_id}, {stage}, {intimacy}, NOW()) "
        f"ON CONFLICT (user_id, character_id) DO UPDATE SET "
        f"current_stage = EXCLUDED.current_stage, "
        f"current_intimacy = EXCLUDED.current_intimacy"
    )
```

> 落地前必须先用 `psql -c "\d relationships"` 在 new 服务器上确认表结构（字段名 / 唯一约束）。若 schema 与上面假设不符，需调整 INSERT 语句。**实现者请在 Step 5 验证后再写 commit。**

- [ ] **Step 6：加主动消息 WS 收取 helper**

```python
async def collect_proactive_ws_push(token: str, timeout_s: float = 60.0) -> Optional[dict]:
    """开一个全新 WS 连接，听 timeout_s 秒内第一帧 type=newMessage、message.role=AI 的帧。

    用法：场景里先用现有 chat() 把对话回复收完并关闭对应 WS，再调本 helper 等主动推送。
    这样能干净分离"对话回复" vs"主动推送"——后者必然出现在 chat() 已结束后的独立连接里。

    返回 dict（已 json.loads）或 None（超时）。
    """
    url = WS_URL_TEMPLATE.format(token=token)
    deadline = time.monotonic() + timeout_s
    async with websockets.connect(url) as ws:
        while True:
            remaining = deadline - time.monotonic()
            if remaining <= 0:
                return None
            try:
                raw = await asyncio.wait_for(ws.recv(), timeout=remaining)
            except asyncio.TimeoutError:
                return None
            try:
                frame = json.loads(raw)
            except json.JSONDecodeError:
                continue
            if frame.get("type") != "newMessage":
                continue
            msg = frame.get("message") or {}
            role = (msg.get("role") or "").upper()
            if role == "AI":
                return frame
```

- [ ] **Step 7：加 plan4 数据清理 helper**

```python
def cleanup_plan4_user_data(db: DbHandle, user_id: int) -> None:
    """plan4 scenario 间隔离：清掉 user 的 events_pending / memory_item / relationships
    + Redis last_active。保留 users 行。"""
    db.execute(f"DELETE FROM events_pending WHERE user_id = {user_id}")
    db.execute(f"DELETE FROM memory_item WHERE user_id = {user_id}")
    db.execute(f"DELETE FROM relationship_milestones WHERE user_id = {user_id}")
    db.execute(f"DELETE FROM intimacy_logs WHERE user_id = {user_id}")
    db.execute(f"DELETE FROM relationships WHERE user_id = {user_id}")
    db.execute(f"DELETE FROM memory_summaries WHERE user_id = {user_id}")
    db.execute(f"DELETE FROM memory_profiles WHERE user_id = {user_id}")
    db.execute(f"DELETE FROM chat_embeddings WHERE user_id = {user_id}")
    db.execute(f"DELETE FROM message WHERE user_id = {user_id}")
    redis_cmd("DEL", f"user:last_active:{user_id}")
    redis_cmd("DEL", f"ws:online:{user_id}")
```

> 如 dogfood_test.py 已有等价的 `purge_user_data` / `purge_all_user_data`，**复用该函数**，不要重复造。在 Step 7 实施前用 `rg -n "def purge" scripts/dogfood/dogfood_test.py` 确认现有函数签名，能复用就 import 不写新版本。

- [ ] **Step 8：本地 import / 编译检查**

helpers 用到的 import：`datetime` / `time` / `asyncio` / `json` / `Optional` / `websockets`。检查文件头 import 是否齐全，缺则补。

```
python3 -c "import ast; ast.parse(open('server/scripts/dogfood/dogfood_test.py').read())"
```

预期：无输出（语法 OK）。

- [ ] **Step 9：提交**

```bash
cd /Users/aventador/code/3yan/server
git add scripts/dogfood/dogfood_test.py
git commit -m "feat(dogfood): plan4 helpers (memory_item/events_pending 轮询、scheduled_at 提前、last_active 注入、主动推送 WS 收取、relationship plant、清理)"
```

---

## Task 4：scenario plan4_c_event_followup

**Files:**
- Modify: `server/scripts/dogfood/dogfood_test.py`

- [ ] **Step 1：实现 scenario 函数**

加在已有 scenario 函数旁（保持 `scenario_xxx` 命名风格；若现有是 `run_xxx` 则跟随）：

```python
async def scenario_plan4_c_event_followup(
        token: str, db: DbHandle, user_id: int, character_id: int,
        log: Logger,
) -> ScenarioResult:
    """plan4 c 事件追问：发"我后天下午有面试" → memory_item PLAN_EVENT 出现 →
    events_pending c_event_followup SCHEDULED 出现（★ 验 fallbackExecution=true bug fix）→
    UPDATE scheduled_at=NOW() → 等 WS 收到主动追问 → events_pending=sent / memory_item=done。
    """
    scenario_name = "plan4_c_event_followup"
    cleanup_plan4_user_data(db, user_id)
    plant_relationship_at_stage(db, user_id, character_id, stage=2, intimacy=50)

    # 1. 发用户消息触发抽取
    plant_msg = "我后天下午有个面试，挺重要的"
    replies = await chat(token, [plant_msg], log)
    if not replies or not replies[0]:
        return ScenarioResult(scenario_name, "FAIL", "plant: AI 没回复，主对话链路异常")

    # 2. 等 memory_item PLAN_EVENT 落库
    item_id = wait_for_memory_item(db, user_id, kind="PLAN_EVENT", timeout_s=30)
    if item_id is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            "30s 内 memory_item 没出现 PLAN_EVENT 行，实时抽取上游断了（不是排期问题）",
        )

    # 3. ★ 等 events_pending c_event_followup（验 fallbackExecution=true bug fix）
    event_id = wait_for_events_pending(
        db, user_id, event_type="c_event_followup", status="scheduled", timeout_s=10
    )
    if event_id is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            "memory_item 已落库但 events_pending 10s 内未出现 c_event_followup 行——"
            "MemoryItemScheduledListener 事件传播断了，"
            "怀疑 @TransactionalEventListener(AFTER_COMMIT) 无 fallbackExecution",
        )

    # 4. 提前 scheduled_at 让 scheduler 立即领取（避免等到事件当天 9:00）
    nudge_event_scheduled_at_now(db, event_id)

    # 5. 等主动推送
    push = await collect_proactive_ws_push(token, timeout_s=60)
    if push is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            "60s 内未收到主动推送 WS 帧，下游链路（调度→门控→生成→投递）有断点，"
            f"events_pending.id={event_id} 状态请查表",
        )

    # 6. 终态断言：events_pending=sent / memory_item=done（dispatcher best-effort 回标）
    deadline = time.monotonic() + 10
    while time.monotonic() < deadline:
        ep_status = db.query(
            f"SELECT status FROM events_pending WHERE id = {event_id}"
        )
        mi_status = db.query(
            f"SELECT status FROM memory_item WHERE id = {item_id}"
        )
        if ep_status and ep_status[0][0] == "sent" \
                and mi_status and mi_status[0][0] == "DONE":
            return ScenarioResult(scenario_name, "PASS",
                                  f"主动追问送达：'{push['message'].get('content', '')[:30]}...'")
        time.sleep(0.5)

    return ScenarioResult(
        scenario_name, "FAIL",
        f"WS 已收到推送但 10s 内 DB 终态未回写（events_pending.id={event_id}, "
        f"memory_item.id={item_id}）",
    )
```

- [ ] **Step 2：注册到 SCENARIOS dict**

文件中应有类似 `SCENARIOS = { ... }` 的注册表（plan2/plan3 风格）。加一行：

```python
"plan4_c_event_followup": scenario_plan4_c_event_followup,
```

> 实施者必须先 `rg -n "SCENARIOS = " scripts/dogfood/dogfood_test.py` 找到准确变量名 + 注册形式（可能是 dict / 列表）。如果注册需要元数据（如 plan3 mode flag），按现有 plan3 注册行的模板照写。

- [ ] **Step 3：语法检查**

```
python3 -c "import ast; ast.parse(open('server/scripts/dogfood/dogfood_test.py').read())"
```

预期：无输出。

- [ ] **Step 4：提交**

```bash
cd /Users/aventador/code/3yan/server
git add scripts/dogfood/dogfood_test.py
git commit -m "feat(dogfood): scenario plan4_c_event_followup 全链路（PLAN_EVENT→events_pending→WS推送→终态）"
```

> 端到端跑通验证留到 Task 9 之后（届时 run_dogfood.sh --plan4 已就绪）。

---

## Task 5：scenario plan4_d_emotion_care

**Files:**
- Modify: `server/scripts/dogfood/dogfood_test.py`

- [ ] **Step 1：实现 scenario 函数**

```python
async def scenario_plan4_d_emotion_care(
        token: str, db: DbHandle, user_id: int, character_id: int,
        log: Logger,
) -> ScenarioResult:
    """plan4 d 情绪关怀：发"今天压力好大睡不着" → memory_item EMOTION →
    events_pending d_emotion_care → 提前 scheduled_at → 收主动关怀推送 → 终态回写。
    """
    scenario_name = "plan4_d_emotion_care"
    cleanup_plan4_user_data(db, user_id)
    plant_relationship_at_stage(db, user_id, character_id, stage=2, intimacy=50)

    plant_msg = "今天压力好大，最近一直睡不着"
    replies = await chat(token, [plant_msg], log)
    if not replies or not replies[0]:
        return ScenarioResult(scenario_name, "FAIL", "plant: AI 没回复，主对话链路异常")

    item_id = wait_for_memory_item(db, user_id, kind="EMOTION", timeout_s=30)
    if item_id is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            "30s 内 memory_item 没出现 EMOTION 行，实时抽取没识别情绪",
        )

    event_id = wait_for_events_pending(
        db, user_id, event_type="d_emotion_care", status="scheduled", timeout_s=10
    )
    if event_id is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            "memory_item EMOTION 已落库但 events_pending 10s 内未出现 d_emotion_care 行——"
            "事件传播或 listener 路径断了",
        )

    nudge_event_scheduled_at_now(db, event_id)

    push = await collect_proactive_ws_push(token, timeout_s=60)
    if push is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            f"60s 内未收到主动关怀推送，events_pending.id={event_id} 状态请查表",
        )

    deadline = time.monotonic() + 10
    while time.monotonic() < deadline:
        ep_status = db.query(
            f"SELECT status FROM events_pending WHERE id = {event_id}"
        )
        mi_status = db.query(
            f"SELECT status FROM memory_item WHERE id = {item_id}"
        )
        if ep_status and ep_status[0][0] == "sent" \
                and mi_status and mi_status[0][0] == "DONE":
            return ScenarioResult(scenario_name, "PASS",
                                  f"情绪关怀送达：'{push['message'].get('content', '')[:30]}...'")
        time.sleep(0.5)

    return ScenarioResult(
        scenario_name, "FAIL",
        f"WS 已收到推送但 10s 内 DB 终态未回写（events_pending.id={event_id}, "
        f"memory_item.id={item_id}）",
    )
```

- [ ] **Step 2：注册到 SCENARIOS dict**

```python
"plan4_d_emotion_care": scenario_plan4_d_emotion_care,
```

- [ ] **Step 3：语法检查**

```
python3 -c "import ast; ast.parse(open('server/scripts/dogfood/dogfood_test.py').read())"
```

- [ ] **Step 4：提交**

```bash
git add scripts/dogfood/dogfood_test.py
git commit -m "feat(dogfood): scenario plan4_d_emotion_care 全链路（EMOTION→events_pending→WS推送→终态）"
```

---

## Task 6：scenario plan4_a_greeting

**Files:**
- Modify: `server/scripts/dogfood/dogfood_test.py`

- [ ] **Step 1：实现 scenario 函数**

```python
async def scenario_plan4_a_greeting(
        token: str, db: DbHandle, user_id: int, character_id: int,
        log: Logger,
) -> ScenarioResult:
    """plan4 a 早晚安：plant stage=1 朋友档 → 等 cron tick（override 后每 10-15s 一次）→
    events_pending a_greeting 出现 → 等 WS 收到主动问候 → events_pending=sent。

    依赖 env override：morning-cron=*/10、night-cron=*/15，scatter=0。
    """
    scenario_name = "plan4_a_greeting"
    cleanup_plan4_user_data(db, user_id)
    # stage=1 朋友档晚安开着；早安要 stage≥2。stage=1 保守覆盖更多档位的 trigger 判定。
    plant_relationship_at_stage(db, user_id, character_id, stage=1, intimacy=25)

    # 等 cron 触发 events_pending a_greeting 出现（cron 每 15s 一次最坏 + 一点 scatter）
    event_id = wait_for_events_pending(
        db, user_id, event_type="a_greeting", status="scheduled", timeout_s=30
    )
    if event_id is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            "30s 内 events_pending 未出现 a_greeting 行——cron 未触发或 "
            "GreetingDailyTrigger 因 stage 判定跳过（请确认 plant 的 stage 在场景开关内）",
        )

    # 等主动推送（scheduled_at 就是 NOW，无需 nudge）
    push = await collect_proactive_ws_push(token, timeout_s=60)
    if push is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            f"60s 内未收到主动问候推送，events_pending.id={event_id}",
        )

    # 终态：events_pending=sent
    deadline = time.monotonic() + 10
    while time.monotonic() < deadline:
        rows = db.query(
            f"SELECT count(*) FROM events_pending "
            f"WHERE user_id = {user_id} AND event_type = 'a_greeting' "
            f"AND status = 'sent'"
        )
        if rows and int(rows[0][0]) >= 1:
            return ScenarioResult(
                scenario_name, "PASS",
                f"主动问候送达：'{push['message'].get('content', '')[:30]}...'"
            )
        time.sleep(0.5)

    return ScenarioResult(
        scenario_name, "FAIL",
        "WS 收到推送但 10s 内未发现 a_greeting/sent 行",
    )
```

- [ ] **Step 2：注册到 SCENARIOS dict**

```python
"plan4_a_greeting": scenario_plan4_a_greeting,
```

- [ ] **Step 3：语法检查**

```
python3 -c "import ast; ast.parse(open('server/scripts/dogfood/dogfood_test.py').read())"
```

- [ ] **Step 4：提交**

```bash
git add scripts/dogfood/dogfood_test.py
git commit -m "feat(dogfood): scenario plan4_a_greeting 走真实 cron 触发路径"
```

---

## Task 7：scenario plan4_b_recall

**Files:**
- Modify: `server/scripts/dogfood/dogfood_test.py`

- [ ] **Step 1：实现 scenario 函数**

```python
async def scenario_plan4_b_recall(
        token: str, db: DbHandle, user_id: int, character_id: int,
        log: Logger,
) -> ScenarioResult:
    """plan4 b 失联召回：plant stage=0 + Redis last_active 设到 4h 前（命中 level-2 阈值 3h）
    → 等 RecallTrigger 扫（override 后 5s 一次）→ events_pending b_recall + payload
    escalation_level=2 → WS 收到占有欲档召回 → events_pending=sent。
    """
    scenario_name = "plan4_b_recall"
    cleanup_plan4_user_data(db, user_id)
    # 陌生人档也保留召回（spec §6.1），用 stage=0 覆盖最广
    plant_relationship_at_stage(db, user_id, character_id, stage=0, intimacy=5)

    # last_active 设到 4h 前（override 后阈值 1,2,3 → 命中 level-2）
    set_last_active_hours_ago(user_id, hours_ago=4)

    event_id = wait_for_events_pending(
        db, user_id, event_type="b_recall", status="scheduled", timeout_s=30
    )
    if event_id is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            "30s 内 events_pending 未出现 b_recall 行——RecallTrigger 未扫到或"
            "threshold 判断错（请确认 env override 已 apply）",
        )

    # 验 payload.escalation_level=2
    payload_rows = db.query(
        f"SELECT payload->>'escalationLevel' FROM events_pending WHERE id = {event_id}"
    )
    if not payload_rows or payload_rows[0][0] != "2":
        # 注意：payload key 实际名可能是 escalation_level / escalationLevel，按 RecallTrigger 实现确认
        log.warn(f"escalation_level 不是预期 2：{payload_rows}")

    push = await collect_proactive_ws_push(token, timeout_s=60)
    if push is None:
        return ScenarioResult(
            scenario_name, "FAIL",
            f"60s 内未收到失联召回推送，events_pending.id={event_id}",
        )

    deadline = time.monotonic() + 10
    while time.monotonic() < deadline:
        rows = db.query(
            f"SELECT count(*) FROM events_pending "
            f"WHERE user_id = {user_id} AND event_type = 'b_recall' "
            f"AND status = 'sent'"
        )
        if rows and int(rows[0][0]) >= 1:
            return ScenarioResult(
                scenario_name, "PASS",
                f"失联召回送达：'{push['message'].get('content', '')[:30]}...'"
            )
        time.sleep(0.5)

    return ScenarioResult(
        scenario_name, "FAIL",
        "WS 收到推送但 10s 内未发现 b_recall/sent 行",
    )
```

> ⚠️ payload key 名（`escalationLevel` vs `escalation_level`）以 `RecallTrigger.java` + `PayloadSupport.java` 的实际实现为准。实施者必须先 grep 确认后修改 payload-key 校验。若 key 不对，PASS 条件依然成立（核心断言是 status=sent + WS 帧到达），只是中间的 log.warn 不准。

- [ ] **Step 2：注册到 SCENARIOS dict**

```python
"plan4_b_recall": scenario_plan4_b_recall,
```

- [ ] **Step 3：语法检查**

```
python3 -c "import ast; ast.parse(open('server/scripts/dogfood/dogfood_test.py').read())"
```

- [ ] **Step 4：提交**

```bash
git add scripts/dogfood/dogfood_test.py
git commit -m "feat(dogfood): scenario plan4_b_recall 走真实 RecallTrigger 扫描路径"
```

---

## Task 8：argparse + all-plan4 dispatch

**Files:**
- Modify: `server/scripts/dogfood/dogfood_test.py`

- [ ] **Step 1：找到 argparse / scenarios dispatch**

```
rg -n "argparse|add_argument.*scenario|--scenario" scripts/dogfood/dogfood_test.py | head -15
```

- [ ] **Step 2：加 all-plan4 处理**

参照现有 `--scenario=all` / `--scenario=all-plan3` 的处理逻辑，新增 `all-plan4` 分支：

```python
PLAN4_SCENARIOS = [
    "plan4_c_event_followup",
    "plan4_d_emotion_care",
    "plan4_a_greeting",
    "plan4_b_recall",
]

# 在 main 解析 args 后：
if args.scenario == "all-plan4":
    scenarios_to_run = PLAN4_SCENARIOS
elif args.scenario == "all":
    scenarios_to_run = list(PLAN2_SCENARIOS)   # 原 plan2 默认行为不变
elif args.scenario == "all-plan3":
    scenarios_to_run = list(PLAN3_SCENARIOS)
elif args.scenario in SCENARIOS:
    scenarios_to_run = [args.scenario]
else:
    print(f"未知 scenario: {args.scenario}", file=sys.stderr)
    sys.exit(2)
```

> 上面 `PLAN2_SCENARIOS` / `PLAN3_SCENARIOS` 等变量名以现有文件实际命名为准。实施者根据 Step 1 结果调整。

- [ ] **Step 3：在 argparse choices 里加 plan4 scenario 名字**

如现有 `choices=[...]` 里枚举了所有 scenario 名，把 4 个 plan4 名字 + `all-plan4` 都加进去。

- [ ] **Step 4：语法检查 + 单 scenario dry-run**

```
python3 -c "import ast; ast.parse(open('server/scripts/dogfood/dogfood_test.py').read())"
python3 scripts/dogfood/dogfood_test.py --scenario=plan4_c_event_followup --help 2>&1 | head -5
```

预期：第一个无输出；第二个不报"未知 scenario"（仅显示帮助）。

- [ ] **Step 5：提交**

```bash
git add scripts/dogfood/dogfood_test.py
git commit -m "feat(dogfood): --scenario=all-plan4 默认跑 c→d→a→b 顺序"
```

---

## Task 9：run_dogfood.sh --plan4 flag

**Files:**
- Modify: `server/scripts/dogfood/run_dogfood.sh`

- [ ] **Step 1：仿照 --plan3 加 PLAN4_MODE 解析**

找到现有 plan3 解析段（约第 30-50 行）：

```bash
PLAN3_MODE=false
PASSTHROUGH_ARGS=()
for arg in "$@"; do
    if [[ "$arg" == "--plan3" ]]; then
        PLAN3_MODE=true
    elif [[ "$arg" == "--scenario=all-plan3" ]]; then
        PLAN3_MODE=true
        PASSTHROUGH_ARGS+=("$arg")
    else
        PASSTHROUGH_ARGS+=("$arg")
    fi
done
```

紧挨着加 PLAN4 解析（独立 for 循环或合并进同一个循环，按风格选）：

```bash
PLAN4_MODE=false
PLAN4_PASSTHROUGH=()
for arg in "${PASSTHROUGH_ARGS[@]:-}"; do
    if [[ "$arg" == "--plan4" ]]; then
        PLAN4_MODE=true
    elif [[ "$arg" == "--scenario=all-plan4" ]]; then
        PLAN4_MODE=true
        PLAN4_PASSTHROUGH+=("$arg")
    elif [[ "$arg" == --scenario=plan4_* ]]; then
        PLAN4_MODE=true
        PLAN4_PASSTHROUGH+=("$arg")
    else
        PLAN4_PASSTHROUGH+=("$arg")
    fi
done
PASSTHROUGH_ARGS=("${PLAN4_PASSTHROUGH[@]}")

if $PLAN4_MODE; then
    HAS_PLAN4_SCENARIO=false
    for arg in "${PASSTHROUGH_ARGS[@]:-}"; do
        if [[ "$arg" == --scenario=* ]]; then
            HAS_PLAN4_SCENARIO=true
            break
        fi
    done
    if ! $HAS_PLAN4_SCENARIO; then
        PASSTHROUGH_ARGS+=("--scenario=all-plan4")
    fi
fi
```

- [ ] **Step 2：加 apply/rollback plan4 env 函数**

照搬 plan3 的 `apply_plan3_env` / `rollback_plan3_env` 结构。新加：

```bash
LOCAL_PLAN4_OVERRIDE="$SCRIPT_DIR/plan4_env_override.conf"
PLAN4_ENV_APPLIED=false

apply_plan4_env() {
    if ! $PLAN4_MODE; then
        return
    fi
    if [[ ! -f "$LOCAL_PLAN4_OVERRIDE" ]]; then
        echo "找不到 $LOCAL_PLAN4_OVERRIDE" >&2
        exit 1
    fi

    echo "==> [plan4 apply] 备份 $REMOTE_ENV + 追加 override + 重启服务..."
    ssh "$SERVER" "sudo cp $REMOTE_ENV ${REMOTE_ENV}.bak.plan4"
    scp -q "$LOCAL_PLAN4_OVERRIDE" "$SERVER:/tmp/plan4_env_override.conf"
    ssh "$SERVER" "sudo bash -c 'echo \"\" >> $REMOTE_ENV && \
                                  echo \"# === plan4 dogfood override (auto, 测试结束自动 rollback) ===\" >> $REMOTE_ENV && \
                                  cat /tmp/plan4_env_override.conf >> $REMOTE_ENV && \
                                  rm /tmp/plan4_env_override.conf && \
                                  systemctl restart 3yan-server'"
    PLAN4_ENV_APPLIED=true

    echo "==> [plan4 apply] 等待服务启动（最多 30s）..."
    ssh "$SERVER" "
        for i in \$(seq 1 30); do
            if systemctl is-active --quiet 3yan-server; then
                exit 0
            fi
            sleep 1
        done
        exit 1
    " || { echo "  [error] 服务启动超时" >&2; exit 1; }
    echo "==> [plan4 apply] 完成"
}

rollback_plan4_env() {
    if ! $PLAN4_ENV_APPLIED; then
        return
    fi

    echo "==> [plan4 rollback] 恢复 $REMOTE_ENV → 重启服务..."

    # 双保险：① 备份覆盖 ② 显式 sed 删 plan4 override 块
    ssh "$SERVER" "sudo cp ${REMOTE_ENV}.bak.plan4 $REMOTE_ENV && \
                   sudo sed -i '/# === plan4 dogfood override/,\$d' $REMOTE_ENV && \
                   sudo rm -f ${REMOTE_ENV}.bak.plan4 && \
                   systemctl restart 3yan-server"

    REMAINING="$(ssh "$SERVER" "sudo grep -c 'plan4 dogfood override' $REMOTE_ENV || true")"
    if [[ "$REMAINING" != "0" ]]; then
        echo "  [error] env 文件仍残留 plan4 override 标记 ${REMAINING} 行" >&2
        exit 1
    fi

    echo "==> [plan4 rollback] 等待服务启动（最多 30s）..."
    ssh "$SERVER" "
        for i in \$(seq 1 30); do
            if systemctl is-active --quiet 3yan-server; then
                exit 0
            fi
            sleep 1
        done
        exit 1
    " || { echo "  [error] rollback 后服务启动超时" >&2; exit 1; }
    echo "==> [plan4 rollback] 完成"
}
```

- [ ] **Step 3：把 rollback_plan4_env 接进 cleanup_all + trap**

找到现有 `cleanup_all()`：

```bash
cleanup_all() {
    rollback_plan3_env
    cleanup_remote_script
}
trap cleanup_all EXIT
```

改为：

```bash
cleanup_all() {
    rollback_plan4_env
    rollback_plan3_env
    cleanup_remote_script
}
trap cleanup_all EXIT
```

（先 rollback plan4 再 plan3：plan4 套在 plan3 内层，先解外层；如果两者互斥可以并列。本期 plan3 / plan4 互斥使用，trap 内顺序无所谓但先 plan4 直觉上更对）

- [ ] **Step 4：在主流程加 apply_plan4_env 调用**

找到现有 `apply_plan3_env` 调用位置（在上传脚本之后、ssh 跑 python 之前）：

```bash
if $PLAN3_MODE; then
    apply_plan3_env
fi
```

并列加：

```bash
if $PLAN4_MODE; then
    apply_plan4_env
fi
```

- [ ] **Step 5：用法注释更新**

文件头注释加：

```bash
#   ./run_dogfood.sh --plan4                     # 跑 plan4 全部 4 个场景
#   ./run_dogfood.sh --plan4 --scenario=plan4_c_event_followup
#   ./run_dogfood.sh --scenario=all-plan4        # 同 --plan4
```

- [ ] **Step 6：shellcheck 静态检查**

```
shellcheck server/scripts/dogfood/run_dogfood.sh || true
```

修复 shellcheck 报的 ERROR 级问题（WARN 级如果已有同类未修可忽略，与现有风格保持一致）。

- [ ] **Step 7：本地 dry-run（不真重启）**

```
bash -n server/scripts/dogfood/run_dogfood.sh
```

预期：无语法错误。

- [ ] **Step 8：提交**

```bash
cd /Users/aventador/code/3yan/server
git add scripts/dogfood/run_dogfood.sh
git commit -m "feat(dogfood): run_dogfood.sh --plan4 flag (apply/rollback env + 重启 + trap)"
```

---

## Task 10：实跑全套 final gate（含 bug-catching capability 验证）

**Files:** 无新代码，纯端到端验证 + 文档化。

- [ ] **Step 1：跑全套**

```bash
cd /Users/aventador/code/3yan/server/scripts/dogfood
./run_dogfood.sh --plan4 --user-id=905
```

预期输出（每个 scenario 一行）：

```
[plan4_c_event_followup] PASS - 主动追问送达: '...'
[plan4_d_emotion_care]   PASS - 情绪关怀送达: '...'
[plan4_a_greeting]       PASS - 主动问候送达: '...'
[plan4_b_recall]         PASS - 失联召回送达: '...'
```

整体耗时 ≤ 5 分钟（apply 30s + 4 scenario ~3min + rollback 30s + 一些 buffer）。

任一 scenario FAIL → 回到 Task 4-7 排查对应实现 + 重跑。

- [ ] **Step 2：bug-catching capability 验证（需用户授权部署）**

**⚠️ 本步骤涉及临时把生产代码回滚到有 bug 状态再恢复，必须先问用户："要不要现在验证 dogfood 抓 bug 能力？需要部署有 bug 的版本→跑 plan4_c 看必须 FAIL→再部署回 fix 版本。"** 用户同意后再做。

如同意：

a. 临时编辑 `business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/MemoryItemScheduledListener.java` 去掉 `fallbackExecution = true`：

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onMemoryItemScheduled(MemoryItemScheduledEvent event) {
```

**不 commit。**

b. 本地部署（不推 git）：

```
export JAVA_HOME="$(/usr/libexec/java_home)"
cd server && ./deploy.sh
```

c. 跑 plan4_c：

```
./run_dogfood.sh --plan4 --scenario=plan4_c_event_followup --user-id=905
```

预期：FAIL，错误信息含"events_pending 10s 内未出现 c_event_followup 行"。

d. 恢复代码 + 重新部署：

```
cd server && git checkout business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/MemoryItemScheduledListener.java
./deploy.sh
```

e. 再跑一次验证恢复：

```
./run_dogfood.sh --plan4 --scenario=plan4_c_event_followup --user-id=905
```

预期：PASS。

- [ ] **Step 3：把 Step 1 + Step 2 的输出贴到 PR description / commit body 作为 acceptance 记录**

不需要新 commit，PR 模板里有 acceptance 段落即可。

---

## Self-Review（实施者请在 Task 1 开始前自查一遍）

**1. Spec 覆盖：**
- spec §3 改动文件清单 6 项 ↔ Task 1（前 3 项：RecallTrigger.java + application.yml + 新 IT）+ Task 2（plan4_env_override.conf）+ Task 3-8（dogfood_test.py 各步）+ Task 9（run_dogfood.sh） ✓
- spec §4 env override 内容 ↔ Task 2 完整列出 ✓
- spec §5 四个 scenario 流程 ↔ Task 4-7 ✓
- spec §6 RecallTrigger 改动 + IT ↔ Task 1 ✓
- spec §7 TDD + acceptance ↔ Task 1 (RecallTrigger TDD) + Task 10 (acceptance) ✓
- spec §9 risk: payload key 名以实现为准 ↔ Task 7 注释明确要求实施者 grep 确认 ✓

**2. Placeholder 扫描：**
- 无 "TBD / TODO / fill in" 关键词
- 每个 step 含完整代码 / 命令 / 预期输出
- 无 "see Task X" 占位（重复代码已展开）

**3. 类型 / 命名一致性：**
- `wait_for_memory_item` / `wait_for_events_pending` / `nudge_event_scheduled_at_now` / `set_last_active_hours_ago` / `plant_relationship_at_stage` / `collect_proactive_ws_push` / `cleanup_plan4_user_data` 在 Task 3 定义、Task 4-7 调用，签名一致
- `scenario_plan4_*` 命名一致，4 个 scenario 函数返回 `ScenarioResult`
- env override key 在 Task 2 conf 文件 ↔ Task 1 yml property ↔ Task 1 IT 断言三处使用同一字符串 `sanyan.proactive.recall.scan-interval-ms`

**4. Scope：**
本 plan 是一个独立可执行单元，不需要再拆。

---

## Execution Handoff

Plan 完成并准备好提交到 `docs/superpowers/plans/2026-05-29-plan-4-dogfood.md`。

**两种执行方式：**

1. **Subagent-Driven（推荐）** — 每个 Task 派一个全新子代理，Task 间两轮 review（spec 审查 + 代码质量审查），符合 CLAUDE.md 的 superpowers 规范
2. **Inline Execution** — 当前会话直接连跑，checkpoint 在关键节点停下

按 CLAUDE.md「Superpowers 流程规范」+「子代理 prompt 注入规则」，每个实现子代理的 prompt 必须显式注入：(a) 完整 TDD 流程要求（Red-Green-Refactor 三步、必须看到 RED 才写实现）(b) 本 Task 涉及的具体测试命令（如 Task 1 的 `mvn -pl bootstrap failsafe:integration-test ...`）(c) spec 文档路径（`docs/superpowers/specs/2026-05-29-plan-4-dogfood-design.md`）。
