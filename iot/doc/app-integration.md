# 手机 APP 接入方案

## 1. 注册 & 登录

### 注册
```http
POST /api/register
Content-Type: application/json

{"username": "user123", "password": "secure_pass"}
```
**返回：**
```json
{"access_token": "eyJ0eXAiOiJKV1Qi..."}
```

### 登录
```http
POST /api/login
Content-Type: application/json

{"username": "user123", "password": "secure_pass"}
```
**返回：**
```json
{"access_token": "eyJ0eXAiOiJKV1Qi..."}
```

**Token 格式：** RS256 JWT，有效期 24 小时，Payload 含 `sub`（用户名）和 `exp`。

所有需要鉴权的接口均在 Header 中携带：
```
Authorization: Bearer <access_token>
```

---

## 2. 设备管理

### 获取我绑定的设备
```http
GET /api/my/devices
Authorization: Bearer <token>
```

---

## 3. 视频通话（LiveKit 方案）

这是**主要通话方式**：通过后端分发 LiveKit Token，APP 使用 LiveKit Client SDK 接入。

### 发起通话
```http
POST /api/devices/{device_id}/call
Authorization: Bearer <token>
```
**返回：**
```json
{
  "livekit_url": "wss://livekit.example.com",
  "token": "<livekit_jwt_hs256>",
  "room_id": "room-dev-001"
}
```

后端同时会通过 MQTT 向设备下发 `JoinRoom` 命令，设备自动连接同一个 LiveKit 房间并开始推流。

**APP 侧接入：**
```
拿到 livekit_url + token
→ LiveKit Client SDK: room.connect(livekit_url, token)
→ 订阅 device 发布的 video/audio track
→ 视频渲染
```

### 挂断
```http
POST /api/devices/{device_id}/hangup
Authorization: Bearer <token>
```

---

## 4. P2P WebRTC 视频通话（直连方案）

适用于低延迟需求或绕过 LiveKit 的场景，信令通过 **MQTT + Protobuf** 传输。

### 整体流程

```
APP → [GET /api/ice_config]  → 获取 TURN 临时凭据
APP → 创建本地 PeerConnection
APP → createOffer()
APP → MQTT 发送 Offer   ──→ 设备收到，createAnswer()
                            设备 → MQTT 回复 Answer
APP ← 收到 Answer
双向交换 ICE Candidate
ICE 连通后，设备 RTP 流直接推给 APP
```

### MQTT Topic 规则

| 方向 | Topic | 说明 |
|------|-------|------|
| APP → 设备 | `bird_mini/{device_id}/down/webrtc` | Offer / Candidate / Bye |
| 设备 → APP | `bird_mini/{device_id}/up/webrtc/{app_client_id}` | Answer / Candidate |

`app_client_id` 是 APP 自己生成的 UUID，用于标识本次会话。

### 获取 TURN 配置
```http
GET /api/ice_config
Authorization: Bearer <token>
```
**返回：**
```json
{
  "ice_servers": [
    {
      "urls": ["turn:coturn.host:3479", "turns:coturn.host:5349?transport=tcp"],
      "username": "1713456789:user123",
      "credential": "base64_hmac"
    }
  ],
  "ttl_seconds": 300
}
```
凭据有效期 **5 分钟**，发起 P2P 前实时获取。

### Protobuf 消息格式（`webrtc.proto`）

ovo_iot_protocol/iot/protocol/webrtc.proto

### 详细步骤

**① APP 发送 Offer**
- `app_client_id`：本次会话 UUID（APP 自生成）
- `type`：`SIGNAL_TYPE_OFFER`
- `sdp`：本地 createOffer() 的结果
- `ice_config`：刚从 `/api/ice_config` 拿到的完整配置
- **发布到：** `bird_mini/{device_id}/down/webrtc`

**② 设备回复 Answer**
- `type`：`SIGNAL_TYPE_ANSWER`
- `sdp`：设备 createAnswer() 结果
- **发布到：** `bird_mini/{device_id}/up/webrtc/{app_client_id}`
- APP 订阅该 topic，收到后 `setRemoteDescription(answer)`

**③ 双向交换 ICE Candidate**
- 双方各自在收集到 candidate 后，`type = SIGNAL_TYPE_CANDIDATE` 发送给对方
- APP → 设备：`bird_mini/{device_id}/down/webrtc`
- 设备 → APP：`bird_mini/{device_id}/up/webrtc/{app_client_id}`

**④ 连通后**
- 设备推送 **H.264 视频** + **Opus 音频** RTP 流
- ICE 状态到 Connected，媒体开始流入

**⑤ 结束通话**
- APP 发送 `SIGNAL_TYPE_BYE` 到 `bird_mini/{device_id}/down/webrtc`
- 设备收到后关闭 PeerConnection

---

## 5. REST API 一览

| 方法 | 端点 | 鉴权 | 说明 |
|------|------|------|------|
| POST | `/api/register` | 无 | 注册 |
| POST | `/api/login` | 无 | 登录 |
| GET | `/api/my/devices` | Bearer | 我的设备 |
| POST | `/api/devices/{id}/call` | Bearer | 发起通话 |
| POST | `/api/devices/{id}/hangup` | Bearer | 挂断 |
| GET | `/api/devices/{id}/join` | Bearer | 加入已有房间 |
| GET | `/api/ice_config` | Bearer | 获取 TURN 凭据（P2P 用）|

---

## 6. 推荐接入路径

**使用 LiveKit 方案**（第3节）：实现最简单，使用官方 LiveKit Client SDK，无需自己做 WebRTC 信令，稳定性好。

**P2P 方案**（第4节）适用于：局域网直连、超低延迟需求、或不想经过 LiveKit 中继的场景，需要自己实现 WebRTC + MQTT 信令逻辑。
