# 设备事件同步方案

## 背景

设备通过 `bird_mini/{device_id}/up/event` 实时推送录像事件（运动检测等）给 APP。但该方案存在两类丢失场景：

- **APP 离线**：推送时 APP 未订阅，消息被 broker 丢弃（QoS 1 不持久化）
- **设备离线**：录像期间断网，重连后 APP 不知晓新事件

需要一套可靠同步机制，使 APP 在任意时刻上线后都能获取完整事件列表。

---

## 方案：StatusReport 携带 last_event_time

### 核心思路

`up/event` 保留为**实时推送通道**（advisory），不承担可靠性保证。

可靠性由 **QueryEvent** 承担：APP 在需要时主动向设备查询指定时间范围内的事件列表，设备从 SQLite 响应。

触发 QueryEvent 的时机由 `StatusReport.last_event_time` 驱动：设备在每次心跳/状态上报时，携带本地最新完成录像的 unix 时间戳。APP 将收到的值与本地已知值对比，若更大则发起 QueryEvent 同步。

### 协议变更

在 `device.proto` 的 `StatusReport` 中新增字段：

```protobuf
message StatusReport {
  string device_id      = 1;
  bool   streaming      = 2;
  string room_id        = 3;
  int32  signal_dbm     = 4;
  int64  last_event_time = 5;  // 最新完成录像的 start_time（unix 秒），0 表示无录像
}
```

`last_event_time = 0` 表示设备暂无录像记录，APP 不触发查询。

### 设备行为

1. **心跳/StatusReport**：每次发布时，从 `VideoStore::latest_event_time()` 读取最新 `start_time`，填入 `last_event_time`。
2. **录像完成时**：立即发布一次 StatusReport（不等下次心跳），使 APP 能快速感知新事件。
3. **断网重连后**：MQTT 连接建立后立即发布一次 StatusReport，使离线期间的积累事件尽快同步。

### APP 行为

1. 本地维护 `known_last_event_time`（持久化，初始为 0）。
2. 收到任意 StatusReport 时：
   - 若 `last_event_time > known_last_event_time`：发送 QueryEvent 查询 `(known_last_event_time, now)`
   - 收到 QueryEventResp 后：更新 `known_last_event_time = last_event_time`，合并事件到本地列表
3. APP 上线后主动拉取一次当前设备的 StatusReport（或等待设备下次心跳，通常 ≤30s）。

### 双离线场景覆盖

| 场景 | 行为 |
|------|------|
| APP 在线，设备在线 | `up/event` 实时推送；StatusReport 也触发 QueryEvent（幂等，返回同一批数据） |
| APP 离线，设备在线 | APP 上线后收到 StatusReport，触发 QueryEvent 补拉 |
| APP 在线，设备离线 | 设备重连后立即发 StatusReport，触发 QueryEvent |
| 双方同时离线 | 双方均上线后，设备发 StatusReport，APP 触发 QueryEvent |

### up/event 的定位

- **不保证送达**：QoS 1，但 APP 可能不在线
- **用途**：在线状态下减少 QueryEvent 延迟（APP 可直接从推送中更新本地列表，跳过轮询）
- **不是唯一来源**：任何情况下 QueryEvent 都是权威数据源

---

## 实现清单

### device.proto

```diff
 message StatusReport {
   string device_id      = 1;
   bool   streaming      = 2;
   string room_id        = 3;
   int32  signal_dbm     = 4;
+  int64  last_event_time = 5;
 }
```

### device.options

```diff
+device.StatusReport.last_event_time  // 无需选项，int64 原生支持
```

无需额外 options，`int64` 直接映射。

### VideoStore 新增接口

```cpp
// 返回最新完成录像的 start_time（unix 秒），无记录时返回 0
int64_t latest_event_time();
```

实现：`SELECT MAX(start_time) FROM events` 或缓存在内存中。

### mqtt_client.cpp

- `publishStatusReport()` 中填充 `sr.last_event_time`
- `connectAndSubscribe()` 成功后立即调用 `publishStatusReport()`

### iot_device.cpp

- `set_video_store()` 中，在 `on_recording_done` 回调里额外调用 `publishStatusReport()`（录像完成立即上报）

---

## 测试用例覆盖

### 单元测试（test_video_store）

| 用例 | 验证点 |
|------|--------|
| `LatestEventTime_empty` | 无录像时返回 0 |
| `LatestEventTime_single` | 一条录像后返回其 start_time |
| `LatestEventTime_multiple` | 多条录像时返回最大值 |
| `LatestEventTime_after_finish` | `finish_recording` 后值更新 |

### 集成测试（test_event_sync，新建）

使用 mock MQTT broker（或 spy 方式拦截 publish 调用）：

| 用例 | 场景 | 验证点 |
|------|------|--------|
| `StatusReport_carries_last_event_time` | 有录像时发布 StatusReport | `last_event_time` == VideoStore 返回值 |
| `StatusReport_zero_when_no_recording` | 无录像时发布 StatusReport | `last_event_time == 0` |
| `StatusReport_published_on_recording_done` | 完成录像后 | 立即收到新 StatusReport，`last_event_time` 更新 |
| `StatusReport_published_on_reconnect` | 模拟断线重连 | 重连后立即收到 StatusReport |
| `QueryEvent_triggered_by_last_event_time` | APP 收到更大的 `last_event_time` | 发出 QueryEvent，收到正确 EventMsg 列表 |
| `QueryEvent_not_triggered_when_equal` | APP 收到相同 `last_event_time` | 不发出 QueryEvent |
| `App_offline_recovery` | APP 断线重连后收到 StatusReport | 触发 QueryEvent，补拉正确 |

### mock_peer.py 扩展（端到端）

在现有 mock_peer.py 基础上新增：

- `subscribe_to_status_report(device_id)` → 收到 StatusReport 时回调，解析 `last_event_time`
- `trigger_query_event_if_newer(known_time)` → 发送 QueryEvent，返回 EventMsg 列表
- 端到端测试：运行 `test_event_sync_e2e.py`，验证 APP-offline / device-offline 两个场景下事件最终一致
