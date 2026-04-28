# 基于 WebRTC 音视频流的本地视频点播方案

## Changelog

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-04-28 | v1.0 | 初始版本 |

---

## 1. 概述

本方案在现有 P2P WebRTC 直播通道的基础上，新增**本地录像点播（VOD）**能力，使 APP 可以通过 WebRTC 链路回放设备本地存储的历史视频片段。

整体分三个环节：

1. **事件上报**：设备发生运动检测等事件时，通过 MQTT 将录像片段元信息上报到云端。
2. **事件查询**：APP 通过 MQTT 指令查询指定时间段内的录像列表。
3. **点播回放**：APP 发起 WebRTC 连接，在 Offer 中指定目标文件；连接建立后可实时发送播放控制信令（暂停、跳转、切换、停止）。

所有新增能力均在已有 `device.proto` 和 `webrtc.proto` 基础上扩展，不引入新的传输通道。

---

## 2. 事件上报

### 触发时机

设备端在事件发生后（如移动检测触发录像完成）立即上报一条 `EventMsg`。

### MQTT Topic

```
bird_mini/{device_id}/up/event
```

- **发布方**：设备
- **订阅方**：后端
- **QoS**：1（保证至少一次送达）
- **Payload**：`device.EventMsg`（Protobuf 二进制）

### Payload 定义（`device.proto`）

```protobuf
enum EventType {
  EVENT_TYPE_UND    = 0;
  EVENT_TYPE_MOVING = 1;  // 移动检测
}

message VideoMsg {
  int64   id         = 1;  // 设备本地录像文件唯一 ID
  float   length     = 2;  // 时长（秒）
  int64   start_time = 3;  // Unix 时间戳（秒）
  int64   end_time   = 4;  // Unix 时间戳（秒）
}

message EventMsg {
  EventType type      = 1;
  int64     timestamp = 2;  // 事件触发时间（Unix 秒）
  VideoMsg  video     = 3;
}
```

后端收到后可持久化到数据库，`video.id` 是后续点播请求中用于定位文件的唯一标识。

---

## 3. 事件查询

APP 通过 MQTT 向设备下发 `QueryEvent` 指令，获取指定时间范围内的录像列表。

### 下行指令

```
Topic：bird_mini/{device_id}/down/cmd
Payload：device.Command{ query_event: QueryEvent{ start_time, end_time } }
```

### 上行响应

```
Topic：bird_mini/{device_id}/up/cmd_resp（已有）
Payload：device.CommandResponse{ query_event: QueryEventResp{ repeated EventMsg msg_list } }
```

`msg_list` 按 `video.start_time` 升序排列，APP 拿到列表后可展示录像时间线。

---

## 4. 点播回放

### 4.1 整体流程

```
APP                                      设备
 |                                        |
 |── GET /api/ice_config ───────────────→ |  (拿 TURN 凭据，复用直播接口)
 |                                        |
 |── MQTT QueryEvent ──────────────────→ |
 |←─ QueryEventResp (录像列表) ──────── |
 |                                        |
 |   用户选择某条录像（video.id）         |
 |                                        |
 |── MQTT Offer (含 WebRTCVideo.id) ──→ |
 |←─ MQTT Answer ────────────────────── |
 |←→ ICE Candidate 交换 ─────────────── |
 |                                        |
 |   ICE 连通，设备开始推指定录像 RTP    |
 |                                        |
 |── SIGNAL_TYPE_SEEK (position=30.0) ─→|  (跳转到第 30 秒)
 |── SIGNAL_TYPE_PAUSE ───────────────→ |  (暂停)
 |── SIGNAL_TYPE_PLAY (video.id=N+1) ─→|  (切换到下一个文件)
 |── SIGNAL_TYPE_STOP ────────────────→ |  (结束点播，关闭 PeerConnection)
```

### 4.2 发起点播（WebRTC Offer）

点播发起方式与直播 Offer **相同的信令通道**，区别仅在于 `WebRTCOffer.video` 字段是否携带：

| 场景 | `WebRTCOffer.video` |
|------|----------------------|
| 直播预览 | 不携带（为空） |
| 录像点播 | 携带，填 `video.id` |

```protobuf
// webrtc.proto
message WebRTCVideo {
  int64 id = 1;  // 对应 VideoMsg.id，设备本地录像文件唯一标识
}

message WebRTCOffer {
  string     sdp        = 1;
  IceConfig  ice_config = 2;
  WebRTCVideo video     = 3;  // 携带此字段 → 点播模式，否则为直播模式
}
```

**APP 发送 Offer 步骤：**

1. 从 `QueryEventResp` 中取出目标 `VideoMsg`，记录 `video.id`。
2. 调用 `createOffer()`，生成本地 SDP。
3. 构造 `WebrtcSignal`：
   - `type = SIGNAL_TYPE_OFFER`
   - `offer.sdp = <本地 SDP>`
   - `offer.ice_config = <刚从 /api/ice_config 拿到的配置>`
   - `offer.video.id = <目标录像 id>`
4. Publish 到 `bird_mini/{device_id}/down/webrtc`。

**设备收到 Offer 后：**

- 检测 `offer.video.id != 0`，进入点播模式，定位本地文件。
- 创建 Answer 并回复，ICE 协商完成后，从指定文件开始推流，而非摄像头实时流。

### 4.3 播放控制信令

ICE 连通后，APP 通过 `SIGNAL_TYPE_PLAY / PAUSE / SEEK / STOP` 实时控制回放，Payload 统一使用 `WebRTCPlayOrSeek`：

```protobuf
// webrtc.proto
message WebRTCPlayOrSeek {
  WebRTCVideo video    = 1;  // PLAY 时填写，切换到指定文件；SEEK 时留空
  float       position = 2;  // SEEK 时填写（秒）；PLAY 时留空（从头开始）
}
```

| 信令类型 | `video` 字段 | `position` 字段 | 语义 |
|----------|-------------|----------------|------|
| `SIGNAL_TYPE_PLAY` | 填目标 `video.id` | 0 | 切换播放另一段录像，从头开始 |
| `SIGNAL_TYPE_SEEK` | 留空 | 目标时间（秒） | 当前录像跳转到指定位置 |
| `SIGNAL_TYPE_PAUSE` | — | — | 暂停推流（PeerConnection 保持，仅停止发送 RTP） |
| `SIGNAL_TYPE_STOP` | — | — | 结束点播，等价于 BYE，设备关闭 PeerConnection |

控制信令发布到：`bird_mini/{device_id}/down/webrtc`，`app_client_id` 与 Offer 时相同。

### 4.4 设备端推流行为约束

- **点播期间**设备不得同时向 LiveKit 推摄像头流，避免带宽竞争。若当前有直播会话，应在响应 Offer 时返回 `SIGNAL_TYPE_REJECT` 或 `SIGNAL_TYPE_ERROR`。
- 推流格式与直播相同：**H.264 视频 + Opus 音频**，无需 APP 端额外适配。
- 文件播放到末尾时，设备主动发送 `SIGNAL_TYPE_BYE`（或通过 Data Channel 通知，暂定前者）。
- `SEEK` 操作后，设备应发送 IDR 帧（I 帧）作为第一个 RTP 包，确保 APP 解码器能立即恢复画面。

---

## 5. 错误处理

| 场景 | 设备行为 |
|------|---------|
| 指定 `video.id` 文件不存在 | 回复 `SIGNAL_TYPE_ERROR`，`error_code = 404` |
| 正在直播，无法点播 | 回复 `SIGNAL_TYPE_REJECT` |
| `SEEK` 位置超出文件时长 | 跳转到末尾，随即发送 `SIGNAL_TYPE_BYE` |
| `PLAY` 切换时目标文件不存在 | 发送 `SIGNAL_TYPE_ERROR`，`error_code = 404`，保持连接 |

---

## 6. MQTT Topic 汇总

| Topic | 发布方 | 订阅方 | Payload | 说明 |
|-------|--------|--------|---------|------|
| `bird_mini/{device_id}/up/event` | 设备 | 后端 | `device.EventMsg` | 事件录像上报 |
| `bird_mini/{device_id}/down/cmd` | APP/后端 | 设备 | `device.Command` (QueryEvent) | 查询录像列表 |
| `bird_mini/{device_id}/up/cmd_resp` | 设备 | APP/后端 | `device.CommandResponse` | 录像列表响应 |
| `bird_mini/{device_id}/down/webrtc` | APP | 设备 | `webrtc.WebrtcSignal` | Offer / 播控信令 |
| `bird_mini/{device_id}/up/webrtc/{app_client_id}` | 设备 | APP | `webrtc.WebrtcSignal` | Answer / Candidate |

---

## 7. 与直播通道的复用关系

点播方案**复用**了以下现有能力，无需新增接口：

- `/api/ice_config` — TURN 凭据下发
- WebRTC 信令 Topic（`down/webrtc` / `up/webrtc/{id}`）
- ICE 打洞 + TURN 中继机制
- H.264 + Opus RTP 推流管线

**新增**的内容：

- MQTT Topic `up/event`（设备上报）
- `device.Command.query_event` + `QueryEventResp`（已加入 proto）
- `WebRTCOffer.video` 字段语义（Offer 携带 `video.id` 即触发点播模式）
- `WebRTCPlayOrSeek` 播控信令（`PLAY / PAUSE / SEEK / STOP`）
