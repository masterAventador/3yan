# Plan 2 dogfood 第二关 checklist

> **USER-ONLY task**：subagent 只创建本模板，实际"用 3-7 天 + 填空"由用户本人执行。
> Plan 2 实施日期：2026-05-14 ~ 2026-05-15

---

## 测试场景

### 场景 1：第 3 天后小婉还记得我说过的事吗？

**操作**：
- Day 1：跟小婉聊 30+ 条对话，主动提一些可记忆的事实（例：宠物、工作、最近发生的事、喜好）
- Day 2、Day 3：继续日常聊天，**不主动重复 Day 1 的事实**
- Day 3+：直接问"你还记得我家那只猫吗？"或类似 trigger

**预期**：
- 小婉应能从 RAG 检索或 profile 抽取里调出 Day 1 提到的猫的具体细节（名字、品种等，能记多少看 LLM 能力）

**dogfood 结果**：
- [ ] 还记得：____________________________________________________
- [ ] 不记得：____________________________________________________
- 调用日志 / RAG 命中分数（看 `/var/log/sanyan-server.log` 关键字 `[RAG_SEARCH]` / `[MEMORY_CONTEXT]`）：____________

---

### 场景 2：profile 抽取的 occupation / preferences 准确率？

**操作**：
- 自然地透露你的职业、几个偏好（食物 / 电影 / 颜色 / 音乐）
- 等待 5 分钟以上（避免节流），再触发用户消息让 LLM 抽取一次
- 查看数据库：
  ```sql
  SELECT user_id, character_id, profile_jsonb FROM memory_profiles WHERE user_id = <你的id>;
  ```

**预期**：
- `basic_info.occupation` 字段填上你的职业
- `preferences.{foods,movies,colors,music}` 数组里含你提过的内容

**dogfood 结果**：
- 准确抽取的字段：____________________________________________________
- 错误 / 漏抽 / 误抽的字段：____________________________________________________
- 节流是否生效（5 分钟内多次发不重复触发）：____________

---

### 场景 3：RAG 检索的相关性？有"答非所问"的情况吗？

**操作**：
- 累计 50+ 条历史消息（覆盖多个主题）
- 问一个明确指向旧话题的问题（例："上次我们聊那只猫，叫什么来着？"）
- 用 `mvn` 测试或直接打开 `/var/log/sanyan-server.log` 看 RAG 检索返回什么

**预期**：
- 检索回的 Top 5 fragments 里至少 1-2 条是相关的（cos sim > 0.6）
- 不应有完全不相关的片段（如问猫但返回天气话题）

**dogfood 结果**：
- 检索相关性主观打分（0-10）：____________
- "答非所问"次数 / 总次数：____________
- 阈值 0.6 是否合理（太严过滤太多、太松返回噪音）：____________

---

### 场景 4：LLM 成本日均（DeepSeek V4-Flash 用量）

**操作**：
- 1 周后看 DeepSeek 平台账单或 API 调用日志
- 拆分用量：summary 生成 / profile 抽取 各占多少调用次数 + tokens

**预期**：
- 单用户日均成本应小于 0.5 元（参考：DeepSeek V4-Flash 输入 ¥0.5/M tokens、输出 ¥1/M tokens）

**dogfood 结果**：
- summary 调用次数 / token 累计：____________
- profile 抽取调用次数 / token 累计：____________
- 主对话（豆包）调用对比：____________
- 日均总成本：____________

---

## 系统稳定性观察

> **2026-05-17 变更**：自建 BGE-M3 服务（原部署在 old）已下线，向量化改用硅基流动 API（`POST https://api.siliconflow.cn/v1/embeddings`，模型 `BAAI/bge-m3`，免费）。原"old 服务器资源水位""embedding service 稳定性"两节已废弃，替换为下面的"硅基流动 API 用量"。

### 硅基流动 API 用量与稳定性
- [ ] 一周累计调用次数（看 https://cloud.siliconflow.cn/ 控制台用量统计）：____________
- [ ] 一周累计 token 用量（input-tokens 应该都是 0 费用）：____________
- [ ] 4xx 错误次数（`grep "SiliconFlow 4xx" /opt/3yan/3yan-server/server.log | wc -l`）：____________
- [ ] 5xx + 网络异常次数（`grep "SiliconFlow embedding attempt" /opt/3yan/3yan-server/server.log | wc -l`）：____________
- [ ] 平均 RAG 检索延迟（首段 embedding 单调用 + DB 检索，看 `[RAG_SEARCH]` 关键字）：____________
- [ ] 是否触发限流（L0 等级：2000 RPM / 500000 TPM）：____________

### Redis Streams 索引队列
- [ ] `redis-cli XLEN sanyan:memory:rag:index-queue` 长期 < 10（健康）还是堆积：____________
- [ ] dead-letter 日志条数 `grep RAG_INDEX_DLQ /var/log/sanyan-server.log | wc -l`：____________

---

## 一周后总体评分（dogfood 阶段结束填）

| 维度 | 评分 (0-10) | 备注 |
| --- | --- | --- |
| 长期记忆有用性（vs Plan 1 短期窗口） |  |  |
| Profile 抽取准确率 |  |  |
| RAG 检索召回质量 |  |  |
| 成本可控性 |  |  |
| 系统稳定性 |  |  |
| 是否达成 spec §"聊到第 3 天会不会失忆？" 目标 |  |  |

---

## Plan 3 输入（dogfood 反馈影响下一阶段）

- 需要改进的点：
- 需要废弃的功能：
- 优先级最高的下一步：
