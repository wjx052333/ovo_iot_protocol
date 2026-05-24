# 事件同步方案

## Changelog

| 版本 | 日期 | 说明 |
|------|------|------|
| v0.5 | 2026-05-24 | 新增 App OSS STS 临时凭证接口（`GET /api/oss/credentials`）：App 可自主生成预签名 GET URL，无需逐个请求 Backend；存储路径迁移至 `events/{user_id}/{device_id}/...`；新增 AWS S3 支持。 |
| v0.4 | 2026-05-22 | 改为设备主动申请预签名 URL 方案：设备录像完成后发 `up/req_upload_url`，Backend 生成预签名 PUT URL 并下发 `UploadVideo` 命令，设备 HTTP PUT 上传后发 `up/event`。去除 STS、OssCredential、req_oss_cred，所有云提供商通用。 |
| v0.3 | 2026-05-22 | 引入 STS 自驱上传：Backend 定期向设备推送 OssCredential（STS 临时凭证），设备录像完成后**自主**上传到 OSS，上传成功后发 `up/event`。 |
| v0.2 | 2026-05-22 | 架构调整为三端同步（设备↔Backend↔App）：Backend 成为事件权威数据源，App 不再直接向设备查询。 |
| v0.1 | — | 初版：App 直接向设备发 QueryEvent 拉事件列表。 |

---

## 背景：三端同步问题

系统中存在三个角色：**设备**、**Backend**、**App**。一个完整的录像事件需要经历以下阶段：

```
设备录像完成
    │
    ▼ on_recording_done 触发
设备发布 up/req_upload_url（EventMsg）
    │
    ▼
Backend ExHook 收到：计算 oss_key，生成预签名 PUT URL，下发 UploadVideo 命令
    │
    ▼
设备 HTTP PUT 上传到 OSS（通用预签名 URL，无需云 SDK）
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
| OSS 上传凭证 | Backend 按需生成预签名 PUT URL 下发给设备 |

---

## 一、设备 → Backend 同步

### 1.1 实时路径（设备在线）

1. 录像完成 → `on_recording_done` 触发
2. 设备发布 `up/req_upload_url`（EventMsg，包含 video_id、start_time 等元数据）
3. Backend ExHook 收到：
   - 计算 `oss_key = {prefix}{device_id}/{video_start}_{video_id}.mp4`
   - 调用 `OssPresigner::presign_put(oss_key)` 生成预签名 PUT URL（TTL 由 `OSS_PUT_URL_TTL_SEC` 配置）
   - 下发 `UploadVideo` 命令（video_id + upload_url）
4. 设备收到 `UploadVideo` → HTTP PUT 到预签名 URL
5. **上传成功** → 设备发布 `up/event`（EventMsg，QoS 1）+ `up/status`
6. Backend ExHook 收到 `up/event`：
   - `INSERT IGNORE` 到 `device_events`（幂等）
   - 计算 `oss_key`，`UPDATE device_events SET oss_key = ...`
7. App 收到 `up/event` 后可立即展示新事件（视频已可播放）

**上传失败**：设备发布 `UploadVideoResp(FAIL)`，Backend 记录日志；事件元数据通过 `up/req_upload_url` 不会进 DB（需等 `up/event` 或 QueryEvent 补拉）。

### 1.2 离线恢复路径（设备断线重连后批量同步）

**目的**：同步离线期间产生的事件**元数据**（不触发上传，视频可通过 UploadVideo 兜底）。

**触发时机**：设备 MQTT 重连成功后立即发布 `StatusReport`，携带 `last_event_time`（本地最新完成录像的 `start_time`）。

**Backend 行为**：
1. 收到 `StatusReport.last_event_time > known_last_event_time` → 发 `QueryEvent`
2. 设备按游标分页（每页 ≤ 128 条）逐页响应 `QueryEventResp`
3. Backend 每收到一页 → `INSERT IGNORE` 全部事件元数据（含 oss_key）
4. 重复直到 `next_cursor_id == 0`，更新 `known_last_event_time`

### 1.3 `oss_key` 管理

`oss_key` 是 Backend 按公式 `{prefix}{device_id}/{video_start}_{video_id}.mp4` 算出的确定性路径，只要有事件元数据就能计算，不依赖设备上报。以下三个时机均会写入：

| 时机 | 操作 |
|------|------|
| 收到 `up/event` | INSERT IGNORE 元数据 + UPDATE oss_key |
| 收到 `QueryEventResp` | INSERT IGNORE 元数据（含 oss_key） |
| 收到 `UploadVideoResp OK` | UPDATE oss_key（兜底路径） |

> **注意**：`oss_key` 表示"视频**应当**在 OSS 上的路径"，而非"视频**已经**存在于 OSS"。如果设备离线期间录像但因无预签名 URL 而未能上传，Backend 通过 QueryEvent 补拉后仍会写入 oss_key，但对应文件并不存在于 OSS。此时 App 调用 `GET /api/events/{id}/video-url` 会拿到一个预签名 GET URL，访问该 URL 时 OSS 返回 404——这是预期行为。App 应将此 404 展示为"视频暂不可用"。如需补传，由 Backend 发 UploadVideo 命令（需先为设备生成新的预签名 PUT URL）。

### 1.4 UploadVideo 兜底路径（backward compat）

适用场景：离线期间产生的视频需补传，或任意需要重新上传的场景。

- Backend 发 `UploadVideo`（含预签名 PUT URL）
- 设备上传后：**上传成功发 `up/event`**（与实时路径一致）；失败发 `UploadVideoResp(FAIL)`
- Backend 收到 `up/event` → 写入 oss_key
- App 通过 MQTT 订阅或 REST 拉取感知

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

### 2.4 App 获取 OSS 临时凭证（自主生成预签名 URL）

> 适用场景：App 需要频繁访问大量视频/缩略图时，避免每次都请求 Backend 生成预签名 URL。

**接口**：`GET /api/oss/credentials`（需 JWT 鉴权）

**响应（200 OK）**：

```json
{
  "provider":          "aliyun",
  "access_key_id":     "STS.xxx",
  "access_key_secret": "yyy",
  "security_token":    "zzz",
  "expiry_time":       1748100000,
  "bucket":            "my-bucket",
  "endpoint":          "oss-cn-hangzhou.aliyuncs.com",
  "region":            "cn-hangzhou",
  "key_prefix":        "events/{user_id}/"
}
```

- `expiry_time`：Unix 秒，凭证过期时间（TTL 12 小时）
- `key_prefix`：调用者所有事件文件的路径前缀，格式 `events/{user_id}/`
- `provider`：`"aliyun"` / `"tencent"` / `"aws"`，决定客户端使用哪种签名算法

**凭证刷新策略**：

```
if (now() + 600 > expiry_time) → 重新调用 GET /api/oss/credentials
```

即在过期前 10 分钟刷新，防止凭证在使用途中过期。

**App 本地生成预签名 GET URL 步骤**：

1. 调用 `GET /api/oss/credentials` 拿到凭证（本地缓存，按上述策略刷新）
2. 由 `key_prefix` + `{device_id}/` + `{ts}_{id}.mp4`（或 `.jpg`）拼出完整 OSS key
3. 使用对应云 SDK 或本地签名算法，用 `(access_key_id, access_key_secret, security_token)` 生成预签名 GET URL
4. 直接访问 OSS，无需再调 Backend

凭证权限仅限该用户前缀下文件的 `GetObject` / `HeadObject`，不可写入。

---

## 三、各场景覆盖

| 场景 | 设备→Backend | App 感知新事件 |
|------|-------------|---------------|
| 三端全在线 | `up/req_upload_url` → `UploadVideo` → 上传 → `up/event` | 直接订阅 `up/event`（秒级） |
| App 离线，其余在线 | Backend 写 DB，oss_key 已设 | App 上线后 REST 拉增量 |
| 设备离线，其余在线 | 重连后 StatusReport → QueryEvent 补拉元数据；重新上线后走正常路径 | Backend 收到 `up/event` 后写 oss_key；App 收到 `up/event` 实时感知 |
| 设备+App 均离线 | 同上 | App 上线后 REST 拉增量 |
| 设备离线期间有录像但未上传 | QueryEvent 补拉元数据+oss_key；视频待补传 | REST 拉取，`video_uploaded=false`（oss_key 已写，但文件不在 OSS）直到补传完成 |

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

- `device.proto`：`StatusReport.last_event_time`、`QueryEvent/QueryEventResp` 分页游标、`UploadVideo/UploadVideoResp`
- `VideoStore`：`import_file()`、`latest_event_time()`、`set_on_recording_done()`、`get_meta()`
- `MqttClient`：`on_recording_done` → 发 `up/req_upload_url`；`UploadVideo` 成功 → 发 `up/event`
- `IotDevice`：`set_video_store()` 注册 VideoStore 到 MqttClient
- `OssPresigner`：支持 Aliyun、Tencent COS、AWS S3 预签名 PUT/GET URL
- `StsService`：Aliyun / Tencent / AWS STS AssumeRole，返回 12h 只读临时凭证
- `HttpServer`：`GET /api/oss/credentials`（JWT 鉴权，返回 STS 临时凭证 + key_prefix）
- `ExhookServer`：`up/req_upload_url` 处理（presign + 下发 UploadVideo）、`up/event` 收到后写 oss_key、QueryEvent 补拉
- `MySQLClient`：`list_device_events()`、`get_event_by_video_id()`、`insert_device_events()`、`update_event_oss_key()`
- `HttpServer`：`GET /api/devices/{id}/events`、`GET /api/events/{id}/video-url`

### 待完成

- [ ] App：订阅 `bird_mini/+/up/event`，收到后立即展示（视频已可播放）
- [ ] App：首次加载 / 离线恢复时调 `GET /api/devices/{id}/events` 拉增量
- [ ] 上传失败重传：Backend 可在收到 `UploadVideoResp(FAIL)` 后重新生成预签名 URL 并重发 `UploadVideo`
