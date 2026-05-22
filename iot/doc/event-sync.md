# 事件同步方案

## Changelog

| 版本 | 日期 | 说明 |
|------|------|------|
| v0.3 | 2026-05-22 | 引入 STS 自驱上传：Backend 定期向设备推送 OssCredential（STS 临时凭证），设备录像完成后**自主**上传到 OSS，上传成功后发 `up/event`；Backend 收到 `up/event` 即写入 `oss_key`。去除 Backend 主动触发 `UploadVideo` 的路径（仅保留为 backward compat 兜底）；去除 `notify/event_ready`（`up/event` 本身即"视频已就绪"信号）。 |
| v0.2 | 2026-05-22 | 架构调整为三端同步（设备↔Backend↔App）：Backend 成为事件权威数据源，App 不再直接向设备查询，改为向 Backend 拉列表（`GET /api/devices/{id}/events`）。 |
| v0.1 | — | 初版：App 直接向设备发 QueryEvent 拉事件列表，依赖 StatusReport.last_event_time 触发。 |

---

## 背景：三端同步问题

系统中存在三个角色：**设备**、**Backend**、**App**。一个完整的录像事件需要经历以下阶段：

```
设备录像完成
    │
    ▼
设备自主上传到 OSS（STS 临时凭证，无需 Backend 介入）
    │
    ▼ 上传成功
设备发布 up/event（EventMsg, QoS 1）── 此刻视频已在 OSS
    │
    ├──────────────────────────────────► App（直接订阅，实时感知）
    │
    ▼
Backend ExHook 收到 up/event：
  - INSERT IGNORE → device_events
  - 计算 oss_key，UPDATE device_events
```

三端各自需要解决的问题：

| 问题 | 负责方 |
|------|--------|
| 新事件实时通知 App | 设备直发 `up/event`（上传完成后，可靠） |
| 设备离线期间事件不丢 | Backend 主动向设备 QueryEvent 补拉元数据 |
| App 离线期间事件不丢 | App 上线后向 Backend REST 拉增量 |
| STS 凭证管理 | Backend 定期推送 OssCredential，设备存储并自主使用 |

---

## 一、设备 → Backend 同步

### 1.1 STS 凭证下发

**触发时机**：
1. 设备 MQTT 连接建立时（`OnClientConnected`），Backend 立即推送一次
2. 设备主动上行 `up/req_oss_cred`（无凭证或凭证即将过期时）
3. Backend 后台线程每 60s 扫描一次，凭证剩余有效期 < `STS_REFRESH_SEC`（默认 300s）时刷新

**下行指令**：`down/cmd`（`Command.oss_credential`，QoS 1）

```proto
message OssCredential {
  string access_key_id     = 1;
  string access_key_secret = 2;
  string security_token    = 3;  // STS session token
  int64  expiry_time       = 4;  // unix seconds
  string bucket            = 5;
  string region            = 6;
  string key_prefix        = 7;  // e.g. "events/"
}
```

设备收到后存入内存（`MqttClient::m_oss_creds_`），ACK 一条 `OssCredentialResp`。

### 1.2 实时路径（设备在线 + 有 STS 凭证）

1. 录像完成 → `on_recording_done` 回调触发
2. `_self_upload`：读取内存中的 STS 凭证，调用 `mhc_oss_upload()`（带 `x-oss-security-token` 头）
3. **上传成功** → 设备发布 `up/event`（`device.EventMsg`，QoS 1）+ `up/status`（更新 `last_event_time`）
4. Backend ExHook 收到 `up/event`：
   - `INSERT IGNORE` 到 `device_events`（幂等）
   - 计算 `oss_key = {key_prefix}{device_id}/{video_start}_{video_id}.mp4`
   - `UPDATE device_events SET oss_key = ...`
5. App 收到 `up/event` 后可立即展示新事件（视频已可播放）

**失败处理**：上传失败则不发 `up/event`，设备记 warn 日志。Backend 后续 QueryEvent 同步可补拉元数据，视频可通过 `UploadVideo` 命令兜底上传（见 §1.4）。

**凭证不足**：若 `expiry_time ≤ now + 30s`，跳过自主上传，设备发 `up/req_oss_cred` 请求新凭证，待凭证到达后视频仍需通过 `UploadVideo` 兜底（凭证过期期间的录像无法自动补传，属已知限制）。

### 1.3 离线恢复路径（设备断线重连后批量同步）

**目的**：同步离线期间产生的事件**元数据**（不触发上传，上传由设备自驱）。

**触发时机**：设备 MQTT 重连成功后立即发布 `StatusReport`，携带 `last_event_time`（本地最新完成录像的 `start_time`）。

**Backend 行为**：
1. 收到 `StatusReport.last_event_time > known_last_event_time` → 发 `QueryEvent`
2. 设备按游标分页（每页 ≤ 128 条）逐页响应 `QueryEventResp`
3. Backend 每收到一页 → `INSERT IGNORE` 全部事件元数据
4. 重复直到 `next_cursor_id == 0`，更新 `known_last_event_time`

**`oss_key` 管理**：`oss_key` 是 Backend 按公式 `{prefix}{device_id}/{video_start}_{video_id}.mp4` 算出的确定性路径，只要有事件元数据就能计算，不依赖设备上报。以下三个时机均会写入：

| 时机 | 操作 |
|------|------|
| 收到 `up/event` | INSERT IGNORE 元数据 + UPDATE oss_key |
| 收到 `QueryEventResp` | INSERT IGNORE 元数据（含 oss_key） |
| 收到 `UploadVideoResp OK` | UPDATE oss_key |

> **注意**：`oss_key` 表示"视频**应当**在 OSS 上的路径"，而非"视频**已经**存在于 OSS"。如果设备离线期间录像但因无 STS 凭证而未能上传，Backend 通过 QueryEvent 补拉后仍会写入 oss_key，但对应文件并不存在于 OSS。此时 App 调用 `GET /api/events/{id}/video-url` 会拿到一个预签名 GET URL，访问该 URL 时 OSS 返回 404——这是预期行为，说明视频尚未上传。App 应将此 404 展示为"视频暂不可用"，而非通用错误。如需补传，由 Backend 发 UploadVideo 命令触发设备重传。

### 1.4 UploadVideo 兜底路径（backward compat）

适用场景：设备无 STS 凭证时产生的视频、或 STS 上传失败需人工补传时。

- Backend 发 `UploadVideo`（含预签名 PUT URL）
- 设备上传后发 `UploadVideoResp`
- Backend 收到 `CMD_RESULT_OK` → 计算并写入 `oss_key`
- **此路径不产生 `up/event`**，App 需通过 REST 拉取感知

---

## 二、App ↔ Backend 同步

### 2.1 App 获取事件列表（REST 拉取）

**接口**：`GET /api/devices/{device_id}/events?after_id=0&limit=50`

- 需要 JWT 鉴权，校验调用者拥有该设备
- 游标分页，`after_id` 为上次返回的最后一条 `id`，`limit` 最大 200
- 响应字段：

```json
{
  "events": [
    {
      "id": 123,
      "event_type": 1,
      "event_ts": 1716000000,
      "video_start": 1716000000,
      "video_end": 1716000030,
      "video_length": 30.0,
      "video_uploaded": true
    }
  ],
  "next_cursor_id": 124
}
```

- `video_uploaded: true` 表示 `oss_key` 已写入，可调 `/api/events/{id}/video-url` 获取播放地址

**App 首次加载**：从 `after_id=0` 开始分页拉完；本地缓存最大 `id` 作为下次 `after_id`。

**App 从离线恢复**：以本地缓存的最大 `id` 作为 `after_id`，拉取增量即可。

### 2.2 App 实时感知新事件（MQTT 直订）

App 直接订阅 `bird_mini/{device_id}/up/event`（或通配 `bird_mini/+/up/event`）。

- `up/event` 发布时，视频**已上传到 OSS**，`video_uploaded` 即为 true
- App 收到后可立即展示并调 `/api/events/{id}/video-url` 获取播放地址
- QoS 1；App 离线期间丢失的消息通过 REST 拉增量补回

### 2.3 App 获取视频播放地址

```
GET /api/events/{id}/video-url
```

返回带 TTL 的预签名 GET URL（默认 1 小时），有本地缓存（剩余 > 5 分钟时复用）。

---

## 三、各场景覆盖

| 场景 | 设备→Backend | App 感知新事件 |
|------|-------------|---------------|
| 三端全在线 | `up/event` 上传后实时 | 直接订阅 `up/event`（秒级） |
| App 离线，其余在线 | Backend 写 DB，oss_key 已设 | App 上线后 REST 拉增量 |
| 设备离线，其余在线 | 重连后 StatusReport → QueryEvent 补拉元数据；Backend 重新推送 OssCredential；设备上传后发 `up/event` | Backend 收到 `up/event` 后写 oss_key；App 收到 `up/event` 实时感知 |
| 设备+App 均离线 | 同上 | App 上线后 REST 拉增量 |
| STS 凭证过期/不足 | 视频元数据通过 QueryEvent 补拉；视频可通过 UploadVideo 兜底 | REST 拉取，`video_uploaded=false` 直到兜底上传完成 |

---

## 四、App 订阅 Topic 清单

| Topic | 方向 | QoS | 说明 |
|-------|------|-----|------|
| `bird_mini/{device_id}/up/event` | 设备→App | 1 | 视频已上传，可直接播放；是新事件的权威信号 |
| `bird_mini/{device_id}/up/status` | 设备→App | 0 | StatusReport，含 `last_event_time` |
| `bird_mini/{device_id}/up/heartbeat` | 设备→App | 0 | 心跳，判断设备在线状态 |
| `bird_mini/{device_id}/up/cmd_response` | 设备→App | 1 | 指令响应（QueryEventResp 等） |
| `bird_mini/{device_id}/up/webrtc/{app_client_id}` | 设备→App | 0 | P2P WebRTC 信令 |

---

## 五、实现清单

### 已完成

- `device.proto`：`StatusReport.last_event_time`、`QueryEvent/QueryEventResp` 分页游标、`OssCredential/OssCredentialResp`
- `device.options`：OssCredential 字段 max_size
- `VideoStore`：`import_file()`、`latest_event_time()`、`set_on_recording_done()`
- `MqttClient`：STS 凭证存储、`OssCredential` cmd 处理（case 11）、`_self_upload()`（上传成功后发 `up/event` + StatusReport）、`request_oss_credential()`
- `IotDevice`：`set_video_store()` 注册 `on_recording_done` → `_self_upload`
- `StsClient`：Aliyun STS AssumeRole（HMAC-SHA1 签名）
- `ExhookServer`：OssCredential 定期推送（后台线程）、`OnClientConnected` 触发推送、`up/req_oss_cred` 处理、`up/event` 收到后写 `oss_key`
- `mhc_oss_upload`：支持 `security_token` 参数（`x-oss-security-token` 头）
- `MySQLClient`：`list_device_events()`、`get_event_by_video_id()`、`insert_device_events()`、`update_event_oss_key()`
- `HttpServer`：`GET /api/devices/{id}/events`

### 待完成

- [ ] App：订阅 `bird_mini/+/up/event`，收到后立即展示（视频已可播放）
- [ ] App：首次加载 / 离线恢复时调 `GET /api/devices/{id}/events` 拉增量
- [ ] 设备端：凭证不足时自动调 `request_oss_credential()` 并排队待传视频（当前限制：凭证过期期间的录像需人工触发 UploadVideo 兜底）
