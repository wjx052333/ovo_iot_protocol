# Agent 对接文档

> 面向 APP 研发同学。描述如何通过 MQTT 与后端 AI Agent 交互，包含文字对话、视觉告警推送和持久化监控任务。

## Changelog

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-05-19 | v1.0 | 初始版本 |

---

## 1. 概述

系统提供两套 Agent 交互通道：

| 通道 | 触发方 | 传输内容 | 适用场景 |
|------|--------|----------|----------|
| **App 文字通道** | APP 主动发起 | 文字 / 选项 / AI 回复 | 对话、查询、任务管理 |
| **设备语音通道** | 设备端发起 | 语音（Opus）| 设备侧对讲（非本文重点）|

本文重点描述 **App 文字通道**以及与之配套的**视觉告警推送**和**任务系统 REST API**。

所有 MQTT 消息使用 **Protobuf** 编码，proto 定义见同目录 `protocol/mqtt_agent.proto`。

MQTT Topic 前缀（本文以 `bird_mini` 为例）由后端统一配置，实际接入时以后端提供的值为准。

---

## 2. MQTT Topic 结构

### App 通道（无设备绑定）

| 方向 | Topic | 消息类型 | QoS |
|------|-------|----------|-----|
| APP → Agent | `bird_mini/{user_id}/up/agent_request` | `AgentRequest` | 1 |
| APP → Agent | `bird_mini/{user_id}/up/agent_msg` | `AppMsg` | 1 |
| Agent → APP | `bird_mini/{user_id}/down/agent_response` | `AgentResponse` | 1 |
| Agent → APP | `bird_mini/{user_id}/down/agent_msg` | `AgentMsg` | 1 |

`{user_id}` 即 JWT `sub`（登录用户名）。

### 设备语音通道（供参考）

| 方向 | Topic | 消息类型 |
|------|-------|----------|
| 设备 → Agent | `bird_mini/{user_id}/{device_id}/up/agent_request` | `AgentRequest` |
| 设备 → Agent | `bird_mini/{user_id}/{device_id}/up/opus` | `AudioFrame` |
| Agent → 设备 | `bird_mini/{user_id}/{device_id}/down/agent_response` | `AgentResponse` |
| Agent → 设备 | `bird_mini/{user_id}/{device_id}/down/opus` | `AudioFrame`（TTS） |

---

## 3. App 文字通道——会话流程

### 3.1 建立会话

发送 `AgentRequest(CHAT)` 到 `up/agent_request`，Agent 回复 `AgentResponse(OK)` 后会话正式建立。

```
APP                                      Agent
 │                                         │
 │── AgentRequest(CHAT) ─────────────────→ │
 │                                         │
 │←──────────────── AgentResponse(OK) ────│
 │                                         │
 │  （会话已建立，可发送文字）              │
```

如果 Agent 尚未为该用户启动（首次连接），`AgentResponse(OK)` 可能有 1~2 秒延迟（Agent 冷启动）；已有会话则立即回复。

### 3.2 发送文字消息

会话建立后，向 `up/agent_msg` 发送 `AppMsg`：

```protobuf
AppMsg {
  text: "帮我查一下门口摄像头今天的告警记录"
}
```

Agent 处理后回复一条或多条 `AgentMsg`（通过 `down/agent_msg`）：

```protobuf
AgentMsg {
  seq_id: 1
  state: AGENT_STATE_IDLE
  chat: AgentText {
    text: "今天门口摄像头共触发 3 次告警，分别是……"
    hints: [
      Hint { label: "详情", text: "查看完整告警记录" }
      Hint { label: "关闭", text: "忽略这些告警" }
    ]
  }
}
```

`hints` 是快捷回复按钮，APP 可在输入框上方展示；用户点击某个 hint 时，以 `AppMsg(text=hint.text)` 发回 Agent。

### 3.3 选项交互（PresentChoices）

Agent 有时会推送多选组，供用户从预设选项中选择：

```protobuf
AgentMsg {
  seq_id: 2
  state: AGENT_STATE_IDLE
  choices: MultiChoiceGroup {
    choice_groups: [
      ChoiceGroup {
        group_id: "action_001"
        question: "请选择要执行的操作"
        multi_select: false
        choices: [
          Choice { label: "立即录像", description: "保存最近 30 秒片段" }
          Choice { label: "推送通知", description: "发送手机通知" }
          Choice { label: "忽略",    description: "标记为误报" }
        ]
      }
    ]
  }
}
```

用户选择后，APP 发送 `ChoiceResponse`：

```protobuf
AppMsg {
  choice: ChoiceResponse {
    group_id: "action_001"
    label:    "立即录像"
    question: "请选择要执行的操作"  // 可选，回传原题目
  }
}
```

### 3.4 结束会话

```protobuf
// APP → Agent
AgentRequest { action: AGENT_ACTION_STOP }

// Agent → APP
AgentResponse { code: AGENT_RESPONSE_CODE_QUIT }
```

会话也会因 **30 分钟无消息**自动超时，Agent 发送 `AgentResponse(QUIT, reason="idle_timeout")`。

---

## 4. AgentState 说明

`AgentMsg.state` 描述 Agent 当前状态，APP 可据此更新 UI：

| 值 | 含义 | 建议 UI |
|----|------|---------|
| `AGENT_STATE_IDLE` | 回复完毕，等待用户输入 | 可发送输入框激活 |
| `AGENT_STATE_THINKING` | 正在调用 LLM | 显示思考动画 |
| `AGENT_STATE_SPEAKING` | 语音通道输出中 | 显示播放动画（App 通道一般不出现）|
| `AGENT_STATE_WAITING_INPUT` | 等待用户进一步输入 | 保持输入框激活 |

目前 App 通道回复均附带 `AGENT_STATE_IDLE`，后续会扩展。

---

## 5. 视觉告警推送

当摄像头触发告警事件时，后端自动唤醒 Agent 并向 APP 推送 `AgentMsg(alert=...)`，**无需 APP 预先发送 CHAT 建立会话**（Agent 被动唤醒）。

### 告警消息结构

```protobuf
AgentMsg {
  seq_id: 5
  state: AGENT_STATE_IDLE
  alert: AlertMsg {
    text:     "检测到门口有陌生人停留超过 30 秒，已自动录像。"
    alert_id: "12345"    // 对应后端 vision_events 表的 ID
    task_id:  "a1b2c3d4-..."  // 触发此告警的任务 ID（若非任务驱动则为空）
  }
}
```

### APP 处理建议

1. `alert_id` 非空时，在通知中心或对话列表展示为「告警卡片」而非普通文字气泡。
2. `task_id` 非空时，可关联展示对应监控任务的标题和配置；跳转到任务详情页时使用。
3. 收到告警后，APP 仍可继续发送文字消息与 Agent 互动（追问详情、下达指令等）。

---

## 6. 任务系统 REST API

监控任务（Task）是持久化的规则，满足条件时自动触发告警和执行动作。APP 通过 REST API 管理任务，MQTT 用于接收告警推送。

**注意：** 创建任务需要 **Pro 会员**，否则返回 `400 { "error": "pro_required" }`。

所有接口均需携带：
```
Authorization: Bearer <access_token>
```

### 6.1 任务 CRUD

#### 获取任务列表

```http
GET /api/tasks
GET /api/tasks?status=active
```

`status` 可选值：`active`（运行中）、`paused`（已暂停）；不传则返回所有非删除任务。

**返回：**
```json
{
  "tasks": [
    {
      "id": "a1b2c3d4-0000-4000-8000-000000000001",
      "title": "监控门口陌生人",
      "description": "有陌生人在门口停留超过 30 秒时告警",
      "device_id": "dev-001",
      "watch_targets": ["person"],
      "schedule": {
        "start": "08:00",
        "end": "22:00",
        "days": 127
      },
      "enabled_actions": ["notify_push", "record_clip"],
      "call_911_confirmed": false,
      "status": "active",
      "triggered_count": 3,
      "created_at": "2026-05-01T10:00:00",
      "updated_at": "2026-05-19T08:30:00"
    }
  ]
}
```

`schedule.days` 是位掩码，bit0=周一，bit6=周日，127 表示每天。

#### 创建任务

```http
POST /api/tasks
Content-Type: application/json

{
  "title": "监控门口陌生人",
  "description": "有陌生人在门口停留超过 30 秒时告警",
  "device_id": "dev-001",
  "watch_targets": ["person"],
  "schedule": {
    "start": "08:00",
    "end": "22:00",
    "days": 127
  },
  "enabled_actions": ["notify_push", "record_clip"],
  "call_911_confirmed": false
}
```

**返回 201：**
```json
{ "task": { ...同上... } }
```

**返回 400（非 Pro）：**
```json
{ "error": "pro_required" }
```

#### 获取单个任务

```http
GET /api/tasks/{task_id}
```

**返回 200：** `{ "task": {...} }`，**返回 404：** 任务不存在或无权限。

#### 更新任务

```http
PUT /api/tasks/{task_id}
Content-Type: application/json

{
  "title": "新标题",
  "watch_targets": ["person", "vehicle"]
}
```

只传需要修改的字段，**`status` 字段会被忽略**（状态变更用专用接口）。

**返回 200：** `{ "task": {...} }`

#### 删除任务（软删除）

```http
DELETE /api/tasks/{task_id}
```

**返回 200：** `{ "task": { "status": "deleted", ... } }`

### 6.2 暂停 / 恢复任务

```http
POST /api/tasks/{task_id}/pause
POST /api/tasks/{task_id}/resume
```

无需 body。**返回 200：** `{ "task": {...} }`

### 6.3 任务执行记录

```http
GET /api/tasks/{task_id}/executions
GET /api/tasks/{task_id}/executions?limit=20&before_ts=2026-05-19+12:00:00
```

游标翻页：将上一页返回的 `next_cursor` 作为下一次请求的 `before_ts`。

**返回 200：**
```json
{
  "executions": [
    {
      "id": "exec-uuid",
      "task_id": "a1b2c3d4-...",
      "event_id": "12345",
      "triggered_at": "2026-05-19 12:00:00",
      "vlm_description": "一名男性站在门口，手提包裹",
      "agent_reasoning": "检测到陌生人，触发 notify_push 和 record_clip",
      "actions_taken": ["notify_push", "record_clip"],
      "user_feedback": "",
      "created_at": "2026-05-19 12:00:01"
    }
  ],
  "next_cursor": "2026-05-19 11:50:00"
}
```

`next_cursor` 为空字符串时表示已到最后一页。

---

## 7. 完整会话示例

### 7.1 普通文字对话

```
APP                                       Agent
 │                                          │
 │─ AgentRequest(CHAT) ──────────────────→ │
 │←──── AgentResponse(OK) ────────────────│
 │                                          │
 │─ AppMsg("门口今天有人吗？") ────────────→│
 │←──── AgentMsg(chat="今天 08:32 有一名…")│
 │                                          │
 │─ AppMsg("帮我暂停门口监控任务") ─────────→│
 │←──── AgentMsg(chat="已暂停任务「监控…」")│
 │                                          │
 │─ AgentRequest(STOP) ──────────────────→ │
 │←──── AgentResponse(QUIT) ──────────────│
```

### 7.2 视觉告警自动推送

```
后端                    Agent                     APP
 │                        │                         │
 │─ POST /api/v1/vision_alert ──────────────────→  │
 │                        │                         │
 │                        │─ AgentMsg(alert=AlertMsg{
 │                        │    text="检测到陌生人…",
 │                        │    alert_id="12345",
 │                        │    task_id="a1b2c3d4-…"
 │                        │  }) ──────────────────→ │
 │                        │                         │
 │                        │          APP 展示告警卡片│
 │                        │                         │
 │                        │ ←─ AppMsg("这是谁？") ──│
 │                        │─ AgentMsg(chat="…") ──→ │
```

### 7.3 选项交互

```
APP                                       Agent
 │─ AppMsg("我要设置一个新监控任务") ──────→│
 │←── AgentMsg(choices=MultiChoiceGroup{  │
 │      question: "请选择监控对象"         │
 │      choices: ["人员","车辆","包裹"]    │
 │    }) ─────────────────────────────────│
 │                                         │
 │─ AppMsg(choice={label:"人员"}) ────────→│
 │←── AgentMsg(chat="好的，我将为您…") ────│
```

---

## 8. 错误处理

### AgentResponse 错误码

| code | 含义 | APP 建议处理 |
|------|------|-------------|
| `OK (1)` | 会话建立成功 | 显示"连接成功" |
| `QUIT (2)` | 会话结束 | 清理会话状态；`reason="idle_timeout"` 时可提示超时 |
| `NO_REQUEST (3)` | 发送数据前未建立会话 | 先发 CHAT 再重试 |
| `ERROR (4)` | 内部错误 | 提示用户重试 |

### REST API 常见错误

| HTTP 状态 | `error` 字段 | 说明 |
|-----------|-------------|------|
| 400 | `pro_required` | 当前账号非 Pro，无法创建任务 |
| 401 | — | Token 无效或过期，重新登录 |
| 403 | — | 无权访问该资源 |
| 404 | — | 资源不存在 |

---

## 9. 接入 Checklist

- [ ] MQTT 连接时 `client_id` 使用 `app_{user_id}_{8-hex}` 格式（见《APP 接入方案》第3节）
- [ ] 订阅 `bird_mini/{user_id}/down/agent_response`
- [ ] 订阅 `bird_mini/{user_id}/down/agent_msg`
- [ ] 发起对话：发送 `AgentRequest(CHAT)` → 等待 `AgentResponse(OK)` → 发送 `AppMsg`
- [ ] 处理 `AgentMsg.chat`（文字回复 + hints 快捷按钮）
- [ ] 处理 `AgentMsg.choices`（多选组，用户选择后发 `ChoiceResponse`）
- [ ] 处理 `AgentMsg.alert`（告警推送，`alert_id` + `task_id` 可跳转详情）
- [ ] 会话结束时发 `AgentRequest(STOP)` 或等待 `AgentResponse(QUIT)` 自动超时
- [ ] Pro 用户：任务 CRUD（`/api/tasks`）+ 执行记录翻页（`before_ts` 游标）
