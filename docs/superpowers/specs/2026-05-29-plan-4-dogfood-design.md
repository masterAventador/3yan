# Plan 4 主动消息系统 dogfood 设计 spec

**日期**：2026-05-29
**作者**：Aventador + Claude
**状态**：待实现
**前置依赖**：Plan 4 主动消息系统已上线（`docs/superpowers/specs/2026-05-27-plan-4-proactive-messaging-design.md`），且 `MemoryItemScheduledListener` 的 `fallbackExecution=true` bug 修复已部署（commit `2ccba6d`）。

---

## 1. 背景与目标

### 1.1 痛点

Plan 4 主动消息系统涉及多个时间敏感触发器：

| 场景 | 触发时点 |
|---|---|
| a 早晚安 | cron 8:00 / 22:30 |
| b 失联召回 | 每 15min 扫 last_active 24h / 3d / 7d 阈值 |
| c 事件追问 | `salient_at` = 事件当天 9:00 |
| d 情绪关怀 | `salient_at` = 观察日次日 9:00 |

真实手测要等小时级到天级时长。2026-05-28 修 `MemoryItemScheduledListener` AFTER_COMMIT 静默丢弃事件 bug 后，没有一个工具能在分钟级内验证整条链路。

### 1.2 现状

`server/scripts/dogfood/dogfood_test.py` 已覆盖 Plan 2（profile / throttle / summary / rag）和 Plan 3（10 个 stage/亲密度场景），用 `--plan3` flag 套 env override + 服务重启 + trap rollback 模式。**Plan 4 完全空白**。

### 1.3 目标

新增 `--plan4` 模式，4 分钟内端到端跑完 a/b/c/d 四个场景，覆盖：实时记忆抽取 → trigger 排期 → 调度器领取 → 频率门控 → Generator 调 LLM → DeliveryService 投递 → DB 状态回写 → WS 客户端收到推送。

抓 bug 能力的底线：**未修 fallbackExecution 的旧代码上跑 plan4_c → 必须超时失败**。

---

## 2. 范围与边界

### 2.1 本期做

- 4 个 scenario（a/b/c/d）完整覆盖触发器 → 投递全链路
- 一个 env override 配置文件，apply/rollback 各重启一次服务
- 复用现有 dogfood 基础设施（DbHandle / mint_jwt / WS chat / purge / ScenarioResult / run_dogfood.sh 框架）
- 一行生产代码改：`RecallTrigger.@Scheduled` 硬编码 `fixedDelay` 改为可配置 `fixedDelayString`，默认值不变

### 2.2 本期不做

- ❌ 多用户 / 并发场景测试（单用户串行够覆盖现有功能）
- ❌ L3 推送通道测试（线上 APNs 等证书，本期无意义）
- ❌ 4 层投递的 L2 ACK 超时分支（设计上 L1 在线即满足，L2 是兜底；ACK 超时分支用单测覆盖）
- ❌ 频率门控的"上限触顶"分支测试（Plan 4 spec §6 已用单测覆盖，dogfood 测放行路径即可）
- ❌ PROMISE 类记忆条目（spec §12 本期仅留存不排期）

---

## 3. 改动文件清单

| 文件 | 操作 | 内容 |
|---|---|---|
| `server/business_packages/sanyan-proactive-core/src/main/java/com/sanyan/proactive/trigger/RecallTrigger.java` | 修改 | `@Scheduled(fixedDelay = 15 * 60 * 1000)` → `@Scheduled(fixedDelayString = "${sanyan.proactive.recall.scan-interval-ms:900000}")` |
| `server/bootstrap/src/main/resources/application.yml` | 修改 | `sanyan.proactive.recall` 下新增 `scan-interval-ms: 900000` 显式默认值（与硬编码原值等价） |
| `server/business_packages/sanyan-proactive-core/src/test/java/.../RecallTriggerTest.java` | 修改 | 加单测：context 启动后 SpEL 解析 `${sanyan.proactive.recall.scan-interval-ms}` 得到非零值（防 typo） |
| `server/scripts/dogfood/plan4_env_override.conf` | 新建 | 缩短 cron / threshold / scan interval / 关 quiet-hours / scatter=0 |
| `server/scripts/dogfood/run_dogfood.sh` | 修改 | 加 `--plan4` flag（与 `--plan3` 平行的 apply/rollback env + 重启分支） |
| `server/scripts/dogfood/dogfood_test.py` | 修改 | 加 4 个 scenario function + `--scenario=all-plan4` + `--scenario=plan4_xxx` 单跑入口 + plan4 元数据注册 |

---

## 4. env override 配置（plan4_env_override.conf）

```
# Plan 4 dogfood env override
# 由 run_dogfood.sh --plan4 追加到 /etc/3yan/3yan-server.env，测试结束后自动回滚。
# Spring relaxed-binding：环境变量大写下划线对应 yml kebab-case。

# a 早晚安：cron 改成每 10s / 15s 触发一次
SANYAN_PROACTIVE_GREETING_MORNINGCRON=*/10 * * * * *
SANYAN_PROACTIVE_GREETING_NIGHTCRON=*/15 * * * * *

# b 失联召回：扫描间隔 15min → 5s；阈值 24h/72h/168h → 1h/2h/3h
# ProactiveProperties.recall.thresholdsHours 字段类型是 List<Integer>，
# 不能用小数。配合 last_active 设到 4h 前可直接命中 level-2 占有欲档。
SANYAN_PROACTIVE_RECALL_SCANINTERVALMS=5000
SANYAN_PROACTIVE_RECALL_THRESHOLDSHOURS=1,2,3

# 关免打扰：测试可能在 23:00-08:00 之间跑，不能被 quiet-hours 拦
SANYAN_PROACTIVE_QUIETHOURS_START=0
SANYAN_PROACTIVE_QUIETHOURS_END=0

# 关 scatter：早晚安默认 0~30min 随机延后，测试要确定性
SANYAN_PROACTIVE_SCATTERWINDOWMINUTES=0
```

**回滚机制**：与 plan3 一致，run_dogfood.sh 启动前 `cp .env .env.bak.plan4`，trap 退出时 `cp .env.bak.plan4 .env` + 显式 `sed` 删除 plan4 override 块（双保险防 bak 文件污染）+ 重启 + 验证 env 文件无残留 override 标记。

---

## 5. 四个 scenario 详细流程

### 5.1 共同前置

- 测试用户 `user_id = 905`（dogfood 保留区 900-919；plan2/plan3 占 901-904，905 是下一个空位）
- run_dogfood.sh 在 apply env 后、跑 scenario 前 `purge_user_data(905)` 清旧数据，trap 退出再清一次
- 每个 scenario 函数内部按需 plant 关系数据（如 c/d 需 stage≥0，a/b 需 stage≥1 / stage≥0），结束按需 cleanup
- WS 连接：每个 scenario 各自开 / 关一个连接，参考现有 plan3 模式

### 5.2 plan4_c_event_followup（最重要，验本周修的 bug）

```
1. plant：通过 ChatApi 或直接 DB 插一条 relationships(user_id=905, character_id=1, stage=2)
   （stage=2 暧昧档保证 c_event_followup 场景开关开着）
2. WS 连接 905 → 发"我后天下午有个面试"
3. 轮询 memory_item ≤30s：出现 PLAN_EVENT 行 user_id=905 → 拿到 itemId
   （这一步验"实时抽取 LLM 调用 + handleNew 写库"）
4. 轮询 events_pending ≤10s：出现 c_event_followup SCHEDULED 行 user_id=905
   （★ 这一步验 MemoryItemScheduledListener 在无事务发布场景下 fallbackExecution=true 生效）
5. UPDATE events_pending SET scheduled_at = NOW() WHERE id = <上一步拿到的 eventId>
6. 等 WS push ≤60s：收到一条 role=ai 的主动消息（与普通对话回复区分：payload 中 fromProactive=true 或类似标识，需对照 chat-api 实际契约）
7. 断言：events_pending.status = sent；memory_item.status = done
8. cleanup：purge_user_data(905)
```

### 5.3 plan4_d_emotion_care

与 c 完全对称，差异：

- step 2 发"今天压力好大睡不着"
- step 3 等 memory_item 出现 EMOTION 行（而非 PLAN_EVENT）
- step 4 等 events_pending 出现 d_emotion_care
- 其余完全一致

### 5.4 plan4_a_greeting

```
1. plant：relationships(user_id=905, character_id=1, stage=1)
   （stage=1 朋友档晚安开着；早安要 stage≥2，cron 撞早安会跳过，测晚安更稳）
2. WS 连接 905
3. 等下一次 cron tick（≤15s for night-cron */15 * * * * *）
4. 轮询 events_pending ≤30s：出现 a_greeting 行 user_id=905
   （cron 也可能多次触发产生多条，dedup 由调度器/dispatcher 处理，断言 anyMatch 即可）
5. 等 WS push ≤60s：收到一条 role=ai 主动消息
6. 断言：events_pending 至少一行 status=sent
7. cleanup
```

### 5.5 plan4_b_recall

```
1. plant：relationships(user_id=905, character_id=1, stage=0)
   （陌生人也保留召回，spec §6.1）
2. redis-cli SET user:last_active:905 "<4h 前的 ISO 时间戳>"
   （超过 level-2 阈值 3h，命中 level-2 占有欲档；override 后 thresholdsHours=[1,2,3]）
3. WS 连接 905
4. 等下一次 RecallTrigger 扫（≤5s after override）+ dispatcher 处理
5. 轮询 events_pending ≤30s：出现 b_recall 行 user_id=905 + payload escalation_level=2
6. 等 WS push ≤60s
7. 断言：events_pending.status = sent
8. cleanup（含清 Redis last_active）
```

### 5.6 执行顺序

`run_dogfood.sh --plan4 --scenario=all-plan4` 默认顺序：

```
plan4_c → plan4_d → plan4_a → plan4_b
```

理由：c/d 走真实抽取路径（不依赖 cron tick / Redis 状态，确定性最强，先验主链路）；a/b 依赖时间窗口，放后面避免抢占 c/d 的 dispatcher 资源。

---

## 6. RecallTrigger 生产代码改动

### 6.1 改动

```java
// 改前
@Scheduled(fixedDelay = 15 * 60 * 1000)   // 15min
public void scan() { ... }

// 改后
@Scheduled(fixedDelayString = "${sanyan.proactive.recall.scan-interval-ms:900000}")
public void scan() { ... }
```

`application.yml` 加显式默认：

```yaml
sanyan:
  proactive:
    recall:
      scan-interval-ms: 900000   # 15min
      thresholds-hours: [24, 72, 168]
```

### 6.2 影响面

- 生产默认值 900000ms = 15min，与原硬编码完全等价，**线上行为零变化**
- `ProactiveProperties` 不需要新加字段（`fixedDelayString` 直接走 SpEL 读 environment property，不绑定到 @ConfigurationProperties bean）
- 新加 `RecallTriggerScheduledIntervalIT`：起 SpringBoot context，反射读 `RecallTrigger.scan` 上的 `@Scheduled.fixedDelayString` 解析后非零，防后续误 typo 配置 key

### 6.3 不做的事

- ❌ 不把 `scanIntervalMs` 加进 `ProactiveProperties`（没有业务代码会读这个值，只有 Spring scheduling 框架内部用）
- ❌ 不动 thresholds-hours / cron 等其他已经在 ProactiveProperties 里的配置

---

## 7. TDD 验证策略

### 7.1 RecallTrigger 改动

- RED：先写 `RecallTriggerScheduledIntervalIT`，断言 SpEL `${sanyan.proactive.recall.scan-interval-ms}` 解析到非空非零 → 当前 yml 无此 key 必然失败（因为还没加默认）
- GREEN：注解改 `fixedDelayString` + yml 加默认值 → 测试通过
- 回归：`RecallTriggerTest` 现有单测全过；mvn `proactive-core` 全包测试全过

### 7.2 scenario 函数本身

不写单测（与现有 plan2 / plan3 scenario 一致：scenario 本身就是测试代码）。但要满足两条 acceptance：

1. **正向**：当前 master 上跑 `--plan4 --scenario=all-plan4` → 4 个场景全 PASS
2. **bug 抓取能力**：把 `MemoryItemScheduledListener` 注解临时去掉 `fallbackExecution=true` 重新部署 → 跑 `--plan4 --scenario=plan4_c_event_followup` → 必须超时 FAIL，错误信息指向 events_pending 未出现 c_event_followup 行（**证明这套 dogfood 能复现修复前的 bug**）；测完恢复 `fallbackExecution=true`

第二条由实现者跑一遍记录到 commit message / PR description，不进 CI。

### 7.3 测试粒度

按 CLAUDE.md「Superpowers Task 测试粒度规范」分级：

- RecallTrigger 改动：跑 `proactive-core` 包 + `bootstrap` IT 即可（不下沉 foundation，不跨多业务模块）
- dogfood 脚本改动：纯 Python 脚本，没有 Java 测试，但要本地实跑 `--plan4 --scenario=all-plan4` 通过为 final gate

---

## 8. 跨域依赖与边界

无新增跨域依赖。Plan 4 现有的：

- proactive-core 调 chat-api / character-api / memory-api（已有）
- chat-core 调 push-api（已有）

dogfood 脚本走外部入口：HTTP（plant 关系）+ WS（发消息 / 收推送）+ PG 直查（断言状态）+ Redis 直操（设 last_active）。无 Java 模块依赖变化。

---

## 9. 风险与缓解

| 风险 | 缓解 |
|---|---|
| env override apply/rollback 之间服务重启失败导致线上跑短间隔 cron 资源消耗 | run_dogfood.sh trap 包 `set -euo pipefail`，失败立即报警；rollback 后必 verify env 文件无残留 |
| cron 每 10s/15s 触发会在 events_pending 产生重复 a_greeting 行 | scenario 断言只用 anyMatch，不在乎数量；cleanup 阶段一次 DELETE 全清 |
| 多个 scenario 串行跑时，前一个的 events_pending 残留可能干扰后一个 | 每个 scenario start 前 + end 后都 purge_user_data(905)；plant 阶段先 DELETE FROM events_pending WHERE user_id=905 |
| WS 收推送和真实对话回复无法区分 | 实现前先在 chat-api 找 `deliverProactiveMessage` 落库时的 message 字段差异（如 `metadata` / `from_proactive` 标识）；找不到则 PR 时加最小标识，单独章节追加 |
| RecallTrigger fixedDelayString 默认值与原硬编码不一致引起线上行为漂移 | 单测断言解析值 == 900000；CI 防 typo |

---

## 10. 不在本期边界但需记 TODO

- WS 推送区分 proactive vs 普通对话回复的 message 字段约定（如已存在则用，未存在另起 issue）
- dogfood 报告生成（plan3 现有的 ScenarioResult 串成 markdown summary，plan4 复用即可，无新工作）
- 多用户 / 并发 / 失败重试场景（边界 §2.2 已声明）
