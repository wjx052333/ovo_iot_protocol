# Log Server — APP 对接文档

Log Server 负责收集两类日志：
- **APP 日志**：由 APP 主动上传（如崩溃现场、用户反馈时）
- **设备日志**：APP 触发后由设备自动上传，同步抓取服务端容器日志

每次上传会生成一个 **incident**，包含客户端日志和服务端容器快照，供开发者排查问题。

---

## 服务地址

| 环境 | 地址 |
|------|------|
| 生产 | `https://<部署域名>/log` |
| 本地 | `https://127.0.0.1:17880/log` |

> nginx 将 `/log/` 转发到 log_server 内部（`:8889`），APP 统一访问上表地址。

---

## 1. APP 上传日志

APP 主动上传日志文件（如用户点击"提交反馈"时）。

```http
POST /log/upload
Content-Type: multipart/form-data
```

**Form 字段：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `source` | string | 是 | 上传来源，如 `PreviewView`、`DevicesView`、`crash` |
| `description` | string | 否 | 问题描述 |
| `files[]` | file | 否 | 一个或多个日志文件 |

**请求示例（curl）：**
```bash
curl -X POST https://host/log/upload \
  -F "source=PreviewView" \
  -F "description=视频画面卡顿" \
  -F "files[]=@app.log" \
  -F "files[]=@crash.log"
```

**成功响应（200）：**
```json
{
  "incident_id": "PreviewView_20260419_102318",
  "stored_at": "/app/incident_logs/PreviewView_20260419_102318",
  "client_files": ["app.log", "crash.log"],
  "server_logs": ["docker-compose.log", "backend.log", "emqx.log", "livekit.log"]
}
```

**无需鉴权**，此接口公开。

---

## 2. 触发设备上传日志

APP 通过后端 REST API 通知设备上传日志。后端向设备下发 MQTT 命令，设备收到后自动将日志上传到 log_server，**同步抓取服务端容器日志**。

**① APP 调用后端触发接口**

```http
POST /api/devices/{device_id}/upload-logs?upload_url={log_server_upload_url}
Authorization: Bearer <access_token>
```

| 参数 | 位置 | 说明 |
|------|------|------|
| `device_id` | 路径 | 目标设备 ID |
| `upload_url` | query | 设备上传日志的目标 URL，填写 log_server 的上传地址 |
| `description` | query（可选）| 问题描述，转发给 log_server |

**upload_url 填写：**
```
https://<部署域名>/log/api/device-logs/upload
```

**请求示例：**
```bash
curl -X POST \
  "https://host/api/devices/dev-001/upload-logs?upload_url=https://host/log/api/device-logs/upload" \
  -H "Authorization: Bearer eyJ..."
```

**成功响应（200）：**
```json
{"ok": true}
```

**② 后台流程（APP 不需要处理）**

```
APP → POST /api/devices/{id}/upload-logs
       ↓
    backend 通过 MQTT 向设备下发 UploadLogs 命令（含 upload_url）
       ↓
    设备接收命令，POST 日志文件到 upload_url
       ↓
    log_server 保存设备日志 + 抓取容器日志 → 生成 incident
```

设备上传完成后，incident 出现在 `/log/incidents` 列表中，incident_id 格式为 `{device_id}_{时间戳}`。

---

## 3. 鉴权

查询 incident 的接口需要在 Header 中携带密码：

```
X-Token: <log_server_password>
```

密码通过以下接口验证获取：

```http
POST /log/auth
Content-Type: application/x-www-form-urlencoded

password=<密码>
```

**成功响应（200）：**
```json
{"ok": true}
```

**失败响应（401）：**
```json
{"detail": "密码错误"}
```

> 密码由后端管理员提供，建议存入 `sessionStorage`，重启 APP 后需重新验证。

---

## 4. 查询 Incident 列表

```http
GET /log/incidents
X-Token: <password>
```

**响应：**
```json
{
  "incidents": [
    "dev-001_20260419_142318",
    "PreviewView_20260419_102318",
    "crash_20260418_093045"
  ],
  "count": 3
}
```

列表按时间**倒序**排列（最新在前）。

**筛选设备 incident：** incident_id 以 `{device_id}_` 开头；APP incident 以 `source_` 开头。

---

## 5. 查询 Incident 详情

```http
GET /log/incidents/{incident_id}
X-Token: <password>
```

**响应（设备 incident）：**
```json
{
  "incident_id": "dev-001_20260419_142318",
  "meta": "device_id  : dev-001\ntimestamp  : 20260419_142318\ndescription: (无)",
  "client_files": [],
  "server_files": ["docker-compose.log", "backend.log", "emqx.log", "livekit.log", "coturn.log"]
}
```

> 设备 incident 无 `client` 子目录，设备日志直接在根目录（`iot.log` 等），通过下一节接口读取。

**响应（APP incident）：**
```json
{
  "incident_id": "PreviewView_20260419_102318",
  "meta": "incident_id  : PreviewView_20260419_102318\nsource       : PreviewView\n...",
  "client_files": ["app.log", "crash.log"],
  "server_files": ["docker-compose.log", "backend.log", "emqx.log", "livekit.log"]
}
```

---

## 6. 读取 Incident 文件内容

```http
GET /log/incidents/{incident_id}/files/{category}/{filename}
X-Token: <password>
```

| 参数 | 说明 |
|------|------|
| `category` | `client`（APP 上传）或 `server`（容器日志） |
| `filename` | 文件名，如 `app.log`、`backend.log` |

**响应：** `text/plain` 文件内容（UTF-8）

**示例：**
```bash
# 读取 APP 上传的 crash.log
GET /log/incidents/PreviewView_20260419_102318/files/client/crash.log

# 读取服务端 backend 日志
GET /log/incidents/PreviewView_20260419_102318/files/server/backend.log
```

---

## 7. API 一览

| 方法 | 路径 | 鉴权 | 说明 |
|------|------|------|------|
| POST | `/log/upload` | 无 | APP 上传日志 |
| POST | `/log/auth` | 无 | 验证密码 |
| GET | `/log/health` | 无 | 健康检查 |
| GET | `/log/incidents` | X-Token | 列出所有 incident |
| GET | `/log/incidents/{id}` | X-Token | 查看 incident 详情 |
| GET | `/log/incidents/{id}/files/{category}/{filename}` | X-Token | 读取文件内容 |
| POST | `/api/devices/{id}/upload-logs` | Bearer JWT | 触发设备上传（后端接口） |

---

## 8. 典型接入流程

### 用户反馈场景（APP 直接上传）

```
1. 用户点击"提交反馈"
2. APP 收集本地日志文件
3. POST /log/upload (source="feedback", description="用户描述", files[]=[app.log])
4. 返回 incident_id，提示用户"日志已上传"
```

### 设备异常排查场景

```
1. 运维发现设备 dev-001 异常
2. POST /api/devices/dev-001/upload-logs?upload_url=https://host/log/api/device-logs/upload
3. 等待约 10s，设备日志上传完成
4. GET /log/incidents → 找到最新的 dev-001_xxx incident
5. GET /log/incidents/dev-001_xxx → 查看包含哪些文件
6. GET /log/incidents/dev-001_xxx/files/server/backend.log → 查看后端日志
```
