# 三言 (3yan) — AI 陪伴聊天 App 设计文档

## 1. 产品概述

三言是一款 AI 陪伴聊天应用。AI 拥有独立人格，能像真人朋友一样与用户聊天，并主动发起对话。区别于传统 AI 助手的被动应答模式，三言的 AI 会模拟真实人物场景，在合适的时间主动找用户聊天。

### 核心特点

- AI 有完整人设（性格、说话风格、背景故事）
- AI 主动发消息（定时关怀、事件触发、情境模拟）
- 永久记忆（AI 记住用户聊过的所有重要信息）
- 拟人体验（不流式输出，模拟真人打字延迟）

### 技术栈

- **客户端**：Flutter（iOS + Android）
- **后端**：Java Spring Boot（单体架构）
- **AI 模型**：豆包 doubao-seed-character（火山引擎）
- **数据库**：MySQL + Redis
- **对象存储**：阿里云/腾讯云 OSS（语音文件，后期）
- **推送服务**：极光/个推
- **短信服务**：阿里云/腾讯云短信

## 2. 系统架构

### 整体架构

```
Flutter App (iOS + Android)
    │
    ├── WebSocket ──── Spring Boot 单体服务
    │                      │
    └── REST API ─────────┘
                           │
                ┌──────────┼──────────┐
                │          │          │
           豆包 API    推送服务     MySQL
        (seed-character) (极光/个推)   │
                                    Redis
                                      │
                                    OSS (后期)
```

### 通信方式

**WebSocket — 消息通道：**
- 用户发消息给 AI
- AI 回复（完整消息一次性推送，不流式输出）
- AI 主动消息（用户在线时）
- 业务层心跳保活（ping/pong）
- 离线消息同步

**REST API — 业务接口：**
- 注册 / 登录
- 用户信息管理
- AI 角色信息获取
- 历史消息分页拉取
- 推送 token 上报

### 关键设计决策

- **单体架构**：一个 Spring Boot 应用包含所有模块，部署一个 jar 包。用户量起来后可拆分。
- **WebSocket 为主，推送为辅**：在线时所有消息走 WebSocket，离线时 AI 主动消息通过推送通知送达，用户打开 App 后从服务端同步未读消息。
- **不流式输出**：AI 回复生成完毕后一次性发送，配合"正在输入..."状态和随机打字延迟，模拟真人聊天体验。
- **WSS 加密传输**：生产环境使用 WSS（WebSocket over TLS），Nginx 反向代理配置 SSL 证书。
- **WebSocket 鉴权**：连接时通过 URL 参数传递 JWT token（`wss://xxx/ws?token=xxx`），服务端在握手阶段验证 token，验证失败拒绝连接。

## 3. 数据模型

### user — 用户表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| phone | varchar | 手机号，唯一索引 |
| password | varchar | BCrypt 加密 |
| nickname | varchar | 昵称 |
| avatar | varchar | 头像 URL |
| created_at | datetime | 创建时间 |
| updated_at | datetime | 更新时间 |

### user_token — 登录凭证表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| user_id | bigint | 关联用户 |
| token | varchar | JWT token |
| device_type | varchar | ios / android |
| push_token | varchar | 推送 token |
| expired_at | datetime | 过期时间 |

### ai_character — AI 角色表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| name | varchar | 角色名称 |
| avatar | varchar | 角色头像 URL |
| system_prompt | text | 人设 prompt（性格、说话风格、背景故事） |
| greeting | text | 首次见面的开场白 |
| proactive_config | json | 主动消息配置（频率、时段、风格） |
| type | varchar | preset / custom（预设 / 用户自定义） |
| created_by | bigint | 创建者 user_id，预设角色为 null |
| created_at | datetime | 创建时间 |

### conversation — 会话表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| user_id | bigint | 用户 |
| character_id | bigint | AI 角色 |
| last_message_at | datetime | 最后消息时间 |
| unread_count | int | 未读消息数 |

user_id + character_id 联合唯一索引。一个用户 + 一个角色 = 一个会话。

### message — 消息表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| conversation_id | bigint | 所属会话 |
| sender_type | varchar | user / ai |
| content_type | varchar | text / voice（后期） |
| content | text | 文字内容 |
| media_url | varchar | 语音文件 URL（后期） |
| source | varchar | reply / proactive（回复 / 主动） |
| created_at | datetime | 发送时间 |

### memory_profile — 用户画像表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| conversation_id | bigint | 所属会话，唯一索引 |
| content | text | 自由文本格式的用户画像 |
| updated_at | datetime | 最后更新时间 |

每个会话一条记录。由 AI 在每轮对话结束后自动维护，AI 自行决定保留哪些历史信息、更新哪些新信息。不使用 KV 覆盖模式，避免丢失有价值的历史信息。

### memory_summary — 对话摘要表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| conversation_id | bigint | 所属会话 |
| summary | text | 这轮对话的关键摘要 |
| message_range | varchar | 覆盖的消息 ID 范围 |
| created_at | datetime | 生成时间 |

每轮对话结束后生成一条摘要。最近 N 条摘要在每次调用模型时带给豆包。

### 记忆隔离

记忆绑定在会话上。同一个用户跟不同 AI 角色的记忆是独立的，互不干扰。

## 4. 消息流程

### 用户发消息 → AI 回复

1. 用户通过 WebSocket 发送消息
2. 服务端将用户消息存入 MySQL
3. 服务端通过 WebSocket 推送"正在输入..."状态给客户端
4. 服务端组装 prompt：角色人设 + 当前时间 + 用户画像 + 最近对话摘要 + 最近 N 条消息原文
5. 调用豆包 API，等待完整回复
6. 根据回复文字长度计算随机打字延迟（20 字约 2-3 秒，50 字约 4-6 秒，上限 8 秒）
7. 延迟结束后，将 AI 回复存入 MySQL
8. 通过 WebSocket 推送完整消息给客户端

### 时间感知

每次调用豆包时，在 system prompt 中注入当前时间（如"当前时间：2026年4月7日 周一 14:30"）。确保 AI 不会因为上下文中有"晚安"就以为现在还是深夜，避免时间错乱。

### AI 主动发消息

触发 → 筛选目标用户 → 调豆包生成内容 → 存入 message 表（source = proactive）→ 根据在线状态投递：
- 在线：WebSocket 推送（先 typing + 延迟 + 完整消息）
- 离线：推送通知

### 离线消息同步

WebSocket 连接建立时，客户端发送 sync 消息带上本地最后一条消息的 ID，服务端返回之后的所有未读消息（分页）。推送通知只是提醒，不承载消息内容，点开后从服务端拉取。

### WebSocket 协议格式

```json
// 客户端 → 服务端：发消息
{
  "type": "send_message",
  "conversation_id": 1001,
  "content_type": "text",
  "content": "今天好累啊",
  "client_msg_id": "uuid-xxx"
}

// 服务端 → 客户端：收到确认
{ "type": "ack", "client_msg_id": "uuid-xxx", "server_msg_id": 50001 }

// 服务端 → 客户端：正在输入
{ "type": "typing", "conversation_id": 1001 }

// 服务端 → 客户端：新消息（AI 回复 / AI 主动消息）
{
  "type": "new_message",
  "conversation_id": 1001,
  "message": {
    "id": 50002,
    "sender_type": "ai",
    "content_type": "text",
    "content": "辛苦啦～今天怎么了，跟我说说？",
    "source": "reply",
    "created_at": "2026-04-07T20:30:15Z"
  }
}

// 客户端 → 服务端：同步未读消息
{ "type": "sync", "last_msg_id": 50000 }

// 服务端 → 客户端：同步结果
{ "type": "sync_result", "messages": [...], "has_more": false }

// 心跳
{ "type": "ping" }  →  { "type": "pong" }
```

**WebSocket 连接地址**：`wss://{domain}/ws?token={jwt_token}`

消息格式使用 JSON。后期如需优化性能可切换为 protobuf，通过 WebSocket 连接握手参数 `?format=json|protobuf` 协商格式，支持新老客户端共存。

## 5. 记忆系统

### 三层记忆架构

**第 1 层：用户画像 (memory_profile)**
- 自由文本格式，由 AI 自行维护
- 包含用户的姓名、职业、兴趣爱好、重要的人、近期关心的事等
- AI 自行决定保留历史信息还是更新（如"曾是程序员，2026年转行做设计师"）
- 每次调用模型时都放入 system prompt
- 体积小，永久保留

**第 2 层：对话摘要 (memory_summary)**
- 每轮对话结束后，由豆包生成一句话总结
- 如："用户聊到下周要参加朋友婚礼，决定送餐具"
- 每次调用模型时带最近 10-20 条摘要
- 让 AI 有"最近这段时间聊过什么"的记忆

**第 3 层：当前对话上下文 (message 原文)**
- 最近 20 条消息原文
- 滑动窗口，保证当前聊天连贯
- 超出窗口的消息由第 2 层摘要覆盖

### 发给豆包的 Prompt 结构

```
[System Prompt]
你是"小晚"，性格温柔体贴...（角色人设）
当前时间：2026年4月7日 周一 14:30

你对当前用户的了解：
小明，曾经是程序员，2026年3月转行做了UI设计师...（用户画像）

[近期对话摘要]
[3天前] 用户聊到下周要参加朋友婚礼，决定送餐具
[2天前] 用户说加班到很晚，心情不好
[昨天] 用户分享了周末去爬山的计划

[当前对话上下文]
用户：今天又加班了
小晚：又加班啊...你们公司最近是不是项目赶进度？
...（最近 20 条消息原文）

[用户最新消息]
用户：算了不说这个了，你今天干嘛了
```

### 记忆更新时机

- **触发条件**：用户最后一条消息起，10 分钟内没有新消息，视为一轮对话结束
- **更新方式**：后台异步调用豆包两次
  - 一次提取/更新用户画像：将当前画像 + 这轮对话发给豆包，让 AI 生成更新后的画像
  - 一次生成对话摘要：将这轮对话发给豆包，生成一句话总结
- **用户无感知**：异步执行，不阻塞聊天

### Token 预算控制

用户画像 + 摘要 + 上下文加起来不能超过模型上下文窗口。组装 prompt 时动态控制：画像太长就精简，摘要太多就减少条数，上下文太多就缩短窗口。

## 6. AI 主动消息系统

### 三种触发机制

**1. 定时关怀**
- Spring @Scheduled 定时任务
- 按角色 proactive_config 配置的时段执行（如 8-9 点、12-13 点、21-22 点）
- 每个用户在时间窗口内随机偏移 0-60 分钟，避免机械感
- 内容生成：人设 + 用户画像 + 最近摘要 + 当前时间 → 豆包生成自然问候

**2. 事件触发**
- 定时任务每 30 分钟扫描用户活跃状态
- 条件示例：超过 6 小时未聊天、超过 24 小时未打开 App、连续 3 天活跃度下降
- 同一事件类型不重复触发（用户未回复上一条主动消息时暂停同类型发送）

**3. 情境模拟**
- 每天在 9:00-22:00 随机时间触发 1-2 次
- 不指定话题，让 AI 基于角色人设自由发挥"分享生活"
- 如："刚在路上看到一只橘猫，超级胖，让我想起你说你以前养过猫"

### 防骚扰策略

- 每天总主动消息上限（默认 3 条）
- 两条主动消息最小间隔 2 小时
- 用户未回复上一条主动消息时暂停发送
- 默认活跃时段 8:00-22:00，深夜不发消息
- 后期可让用户自定义免打扰时段

### proactive_config 结构

```json
{
  "max_daily": 3,
  "min_interval_hours": 2,
  "active_hours": [8, 22],
  "greeting": {
    "enabled": true,
    "slots": ["08:00-09:00", "12:00-13:00", "21:00-22:00"]
  },
  "event_trigger": {
    "enabled": true,
    "idle_hours_threshold": 6
  },
  "situational": {
    "enabled": true,
    "daily_count": [1, 2]
  }
}
```

## 7. REST API 接口

### 用户模块

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/auth/sms/send | 发送短信验证码 |
| POST | /api/auth/register | 手机号 + 验证码注册（同时设置密码） |
| POST | /api/auth/login | 手机号 + 密码登录，返回 JWT |
| POST | /api/auth/password/reset | 验证码重置密码 |
| GET | /api/user/profile | 获取用户信息 |
| PUT | /api/user/profile | 修改昵称、头像 |

### 会话 & 消息模块

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/conversations | 会话列表（含未读数、最后一条消息预览） |
| GET | /api/conversations/{id}/messages | 历史消息分页拉取（游标分页） |
| POST | /api/conversations/{id}/read | 标记已读 |

### AI 角色模块

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/characters | 角色列表 |
| GET | /api/characters/{id} | 角色详情 |

### 推送模块

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/device/push-token | 上报推送 token |

### WebSocket 消息类型

| 方向 | type | 说明 |
|------|------|------|
| C → S | send_message | 用户发消息 |
| C → S | sync | 连接时同步未读消息 |
| C → S | ping | 心跳 |
| S → C | ack | 消息发送确认 |
| S → C | typing | AI 正在输入 |
| S → C | new_message | 新消息 |
| S → C | sync_result | 同步结果 |
| S → C | pong | 心跳回应 |

## 8. 功能路线图

### MVP — 核心体验跑通（当前阶段）

- 手机号注册登录（短信验证码 + 密码）
- 一个预设 AI 角色，文字聊天
- WebSocket 实时通信（WSS 加密）
- AI 主动发消息（三种触发机制）
- 三层记忆系统（画像 + 摘要 + 上下文）
- 推送通知（离线消息送达）
- 业务层心跳（ping/pong）

### V2 — 多角色 & 语音

- 多个预设角色可选
- 语音消息（语音条，火山引擎 ASR + TTS）
- 用户自定义角色（手动填写性格、背景）
- 用户免打扰时段设置

### V3 — 深度个性化

- 上传聊天记录蒸馏角色（参考 ex-skill 思路，从聊天记录中提取 5 层性格结构）
- 音频复刻（豆包语音克隆模型，生成专属声音）
- 实时语音通话
- AI 发图片 / 表情包

### V4 — 商业化 & 社区

- 会员订阅 / 按量付费
- 角色市场（用户分享/售卖自定义角色）
- 微信登录
