# Sub-Store 全局设置与 HTTP 配置规范

Sub-Store 提供全局设置接口（`/api/settings`）与备份同步端点（`/api/sync/artifacts`），用于集中管理出站 HTTP 请求行为、缓存更新策略、GitHub Gist 备份同步参数、默认导出平台以及全局节点处理偏好。

本规范严格面向已有运行中的 Sub-Store 实例，纯粹通过 HTTP REST API 进行读取与热更新配置，零环境侵入，不涉及任何环境部署、Docker 安装或宿主机配置。

---

## 一、设置接口概览与增量修改铁律

Sub-Store 通过 `/api/settings` 端点暴露运行时的全量配置读写能力。所有配置项写入后即时生效，无需重启任何服务或进程。

### 1. 端点规范

| 请求方法 | 端点路径 | 功能说明 |
|---|---|---|
| `GET` | `/api/settings` | 获取当前 Sub-Store 实例的全量设置对象 |
| `POST` | `/api/settings` | 回写更新设置对象（必须回写包含既有字段的完整配置） |

> [!NOTE]
> 独立服务器（Standalone）模式下，若配置了私密路径前缀，调用时需携带安全前缀：`${SUB_STORE_URL}/api/settings`（例如 `https://sub.example.com/<SECRET_PATH>/api/settings`）；代理 MITM 模式下通过虚拟域名 `http://sub.store/api/settings` 经由客户端代理接管。

### 2. 设置核心铁律：先拉取后更新，强制增量修改

> [!CAUTION]
> **严禁盲目 POST 部分字段覆盖**：
> Sub-Store 的设置端点接收配置对象。若直接提交仅包含少量修改字段的 JSON，极易导致未包含的其他关键配置项（如已配置的 `gistToken` 备份凭证、`proxy` 出站代理、自定义 `userAgent` 等）被冲掉清空或重置为默认值。
> 
> **所有设置修改必须严格遵循标准三步法（Read -> Mutate -> Write-back）**：
> 1. **步骤一（Read 先拉取）**：调用 `GET /api/settings` 获取当前实例的全量配置 JSON；
> 2. **步骤二（Mutate 内存合并）**：在内存或管道流中对目标字段进行针对性修改，严格完整保留其他所有既有字段；
> 3. **步骤三（Write-back 完整回写）**：调用 `POST /api/settings` 将合并后的完整配置回写保存。

### 3. 获取全量设置 (`GET /api/settings`)

通过 GET 请求拉取当前全量配置 JSON：

```bash
curl -s "${SUB_STORE_URL}/api/settings"
```

典型响应结构体示例（完整对齐前端 Web 控制台四个配置区块）：

```json
{
  "proxy": "http://127.0.0.1:7890",
  "ua": "ClashMeta/v1.18.0",
  "userAgent": "ClashMeta/v1.18.0",
  "flowUserAgent": "ClashMeta/v1.18.0",
  "timeout": 8000,
  "githubApiTimeout": 10000,
  "concurrency": 10,
  "concurrencyWaitTime": 0,
  "cache": true,
  "cacheTTL": 3600,
  "autoUpdate": true,
  "autoUpdateInterval": 24,
  "gistToken": "<GITHUB_TOKEN>",
  "gistId": "<GIST_ID>",
  "gistApi": "https://api.github.com",
  "gistUsername": "<GITHUB_USERNAME>",
  "target": "clash-meta",
  "removeEmoji": false,
  "sortMethod": "name",
  "sortOrder": "asc"
}
```

### 4. 标准增量修改模板（curl + jq 管道流）

通过管道流一步完成“先 GET 拉取 -> jq 增量字段合并 -> POST 回写”：

```bash
# 安全增量更新示例：微调超时与并发，其他既有配置（包括 Gist 凭证）100% 完整保留
curl -s "${SUB_STORE_URL}/api/settings" | jq '
  .timeout = 12000 |
  .concurrency = 5
' | curl -s -X POST "${SUB_STORE_URL}/api/settings" \
  -H "Content-Type: application/json" \
  --data-binary @-
```

### 5. 标准增量修改模板（极简 Python 脚本，用完即走）

```python
import os, json, urllib.request

base_url = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")
settings_url = f"{base_url}/api/settings"

# 1. Read: 先拉取全量配置
req_get = urllib.request.Request(settings_url, method="GET")
with urllib.request.urlopen(req_get) as resp:
    settings = json.loads(resp.read().decode("utf-8"))

# 2. Mutate: 增量更新目标字段，保留所有既有字段
settings["timeout"] = 12000
settings["concurrency"] = 5
settings["proxy"] = "http://127.0.0.1:7890"

# 3. Write-back: 完整回写
body = json.dumps(settings).encode("utf-8")
req_post = urllib.request.Request(settings_url, data=body, headers={"Content-Type": "application/json"}, method="POST")
with urllib.request.urlopen(req_post) as resp:
    print(f"设置增量更新成功 [{resp.status}]")
```

---

## 二、请求配置 (Request Configuration)

请求配置严格对齐前端 Web 控制台的 `Request Configuration` 区块，用于精确控制 Sub-Store 在拉取外部机场订阅、GitHub Gist 以及外部脚本文件时的出站 HTTP 客户端行为。

合理的请求配置可避免因网络波动、上游 WAF 拦截、并发突发或证书校验问题导致的拉取失败。

### 1. 关键设置项字典

| 前端配置项名称 | 对应字段名 | 数据类型 | 默认值 | 作用与规范说明 |
|---|---|---|---|---|
| **Default Proxy/Policy** | `proxy` | string | 空字符串 | 默认出站代理策略（如 `http://127.0.0.1:7890`、`socks5://127.0.0.1:7890` 或指定代理节点/策略组名称）。Sub-Store 将通过该代理中继拉取外部上游资源，适合在受限网络环境下拉取境外订阅资源。 |
| **Default User-Agent** | `ua` / `userAgent` | string | 空字符串 | 拉取订阅时的默认 UA。设置如 `ClashMeta/v1.18.0`、`Surge/5.0.0` 或浏览器 UA，防止因未指定 UA 或默认请求头被上游机场 WAF 判定为异常爬虫并拦截。 |
| **Default Flow User-Agent** | `flowUserAgent` | string | 空字符串 | 拉取流量与用量信息时的特定 UA。部分机场仅向特定客户端 UA（如 Quantumult X 或 Clash）返回 `subscription-userinfo` 响应头，此选项专门用于用量信息查询。 |
| **Default Timeout** | `timeout` | number | `8000` | 默认请求超时时间（毫秒，默认 `8000`）。当机场订阅节点量庞大或服务器网络延迟较高时，建议调整至 `12000` - `20000` 毫秒，防止连接过早截断。 |
| **GitHub API Request Timeout** | `githubApiTimeout` | number | `10000` | GitHub API 请求超时时间（毫秒，默认 `10000`）。专门控制与 GitHub API 交互（如 Gist 备份同步或拉取 GitHub 资源）时的网络超时等待时长。 |
| **Backend Request Concurrency** | `concurrency` | number | `10` | 后端请求并发数（默认 `10`）。后端同时拉取外部多个上游订阅或外部资源时的并发工作协程/请求上限。 |
| **Backend Request Concurrency Wait Time** | `concurrencyWaitTime` | number | `0` | 并发请求等待间隔（毫秒，默认 `0`）。各并发请求批次之间的缓冲等待时长，用于防止瞬间向同一上游发起突发密集请求而触发频率限制（Rate Limit）。 |

### 2. 增量配置范例

#### 范例 1：增量配置抗拦截出站客户端与并发控制
先拉取现有配置，在内存中针对性微调 UA、超时与并发参数，再完整回写：

```bash
curl -s "${SUB_STORE_URL}/api/settings" | jq '
  .ua = "ClashMeta/v1.18.0" |
  .userAgent = "ClashMeta/v1.18.0" |
  .flowUserAgent = "Quantumult%20X/1.0.30" |
  .timeout = 15000 |
  .githubApiTimeout = 15000 |
  .concurrency = 5 |
  .concurrencyWaitTime = 200
' | curl -s -X POST "${SUB_STORE_URL}/api/settings" \
  -H "Content-Type: application/json" \
  --data-binary @-
```

#### 范例 2：增量配置出站代理中继
先拉取现有配置，仅增量更新 `proxy` 代理中继地址，完整保留其他字段：

```bash
curl -s "${SUB_STORE_URL}/api/settings" | jq '
  .proxy = "http://127.0.0.1:7890"
' | curl -s -X POST "${SUB_STORE_URL}/api/settings" \
  -H "Content-Type: application/json" \
  --data-binary @-
```

---

## 三、缓存配置 (Cache Configuration)

缓存配置严格对齐前端 Web 控制台的 `Cache Configuration` 区块，用于管理 Sub-Store 对外部订阅源与文件资源的本地缓存生命周期及自动刷新策略。

### 1. 缓存机制与核心原理

- **订阅与文件缓存机制**：
  Sub-Store 为降低对上游机场服务器的高频网络请求并显著缩短客户端拉取配置时的响应耗时，会在内部维护上游响应结果的缓存。
- **TTL 缓存时长与失效策略**：
  数据在缓存中的存活周期受 TTL 限制。在有效时间内，下游客户端多次请求同一订阅或集合时直接复用缓存并执行本地流水线渲染；缓存过期后由后端并发拉取机制向上游重新获取。
- **后台自动定时刷新（Auto Update）**：
  配置 `autoUpdate` 与 `autoUpdateInterval`，在后台周期性定时触发全量或增量拉取刷新，确保即使长期未主动访问，缓存中的节点数据依然保持有效与最新。
- **缓存穿透与即时刷新**：
  在客户端请求 `/download/...` 端点时，可通过 URL 查询参数添加 `?cache=false` 或 `?purge=true` 绕过当前缓存，强制后端向上游发起真实网络请求并刷新缓存。

### 2. 关键设置项字典

| 配置项 | 数据类型 | 默认值 | 作用与规范说明 |
|---|---|---|---|
| `cache` | boolean | `true` | 是否全局启用订阅与文件缓存机制。 |
| `cacheTTL` | number | `3600` | 缓存存活时长（秒），在 TTL 内直接复用本地缓存。 |
| `autoUpdate` | boolean | `true` | 是否启用订阅后台定时自动刷新机制。 |
| `autoUpdateInterval` | number | `24` | 自动更新触发的时间周期（单位为小时），定期拉取上游订阅并更新本地缓存。 |

### 3. 增量配置范例

先拉取现有配置，增量更新缓存存活时长与自动刷新周期，再完整回写：

```bash
curl -s "${SUB_STORE_URL}/api/settings" | jq '
  .cache = true |
  .cacheTTL = 7200 |
  .autoUpdate = true |
  .autoUpdateInterval = 12
' | curl -s -X POST "${SUB_STORE_URL}/api/settings" \
  -H "Content-Type: application/json" \
  --data-binary @-
```

---

## 四、Gist 备份与同步配置 (Gist Configuration)

Gist 配置严格对齐前端 Web 控制台的 `Gist Configuration` 区块，支持将全量配置数据（自建节点、远程订阅、聚合集合、算子流水线及自定义 JavaScript 脚本）持久化备份至 GitHub Gist 或兼容的 Gist 服务端点。

### 1. 核心配置字段字典

| 配置项 | 数据类型 | 必需 | 默认值 | 说明与示例 |
|---|---|---|---|---|
| `gistToken` | string | 是 | 无 | GitHub 个人访问令牌（PAT），必须具备 `gist` 作用域读写权限。示例占位符：`<GITHUB_TOKEN>`。 |
| `gistId` | string | 是 | 无 | 目标 GitHub Gist 仓库的唯一 Hash 标识符。可在 GitHub 预先创建 secret gist 并提取其 ID。示例占位符：`<GIST_ID>`。 |
| `gistApi` | string | 否 | `https://api.github.com` | 自定义 Gist API 端点。支持配置自建反代、EdgeGist 或 Cloudflare Workers 反代端点，用于境内或受限网络环境。 |
| `gistUsername` | string | 否 | 无 | 绑定的 GitHub 用户名。示例占位符：`<GITHUB_USERNAME>`。 |

增量配置 Gist 凭证范例（严格基于三步法）：

```bash
curl -s "${SUB_STORE_URL}/api/settings" | jq '
  .gistToken = "<GITHUB_TOKEN>" |
  .gistId = "<GIST_ID>" |
  .gistApi = "https://api.github.com" |
  .gistUsername = "<GITHUB_USERNAME>"
' | curl -s -X POST "${SUB_STORE_URL}/api/settings" \
  -H "Content-Type: application/json" \
  --data-binary @-
```

### 2. Gist 备份触发端点 (`/api/sync/artifacts`)

配置完成 Gist 凭证后，Sub-Store 提供专用的备份与配置同步触发接口 `/api/sync/artifacts`。调用该接口立即触发后端将本地所有实体对象打包并同步至目标 Gist。

| 属性 | 规范说明 |
|---|---|
| **端点路径** | `/api/sync/artifacts` |
| **请求方法** | `GET` 或 `POST` |
| **认证与鉴权** | 继承 Sub-Store 实例的前缀路径隔离或客户端 MITM 权限 |
| **响应状态** | `200 OK` 表示同步成功并已持久化至 Gist |

#### 2.1 curl 命令行即时触发

```bash
# 标准 POST 触发全量同步
curl -s -X POST "${SUB_STORE_URL}/api/sync/artifacts"

# 带状态码校验与超时的触发脚本
mkdir -p /tmp/sub-store-copilot
HTTP_STATUS=$(curl -s -o /tmp/sub-store-copilot/sync-result.json -w "%{http_code}" \
  --max-time 15 \
  "${SUB_STORE_URL}/api/sync/artifacts")

if [ "$HTTP_STATUS" = "200" ]; then
  echo "Gist 备份同步成功"
  cat /tmp/sub-store-copilot/sync-result.json
else
  echo "Gist 备份同步失败，HTTP 状态码: ${HTTP_STATUS}"
  cat /tmp/sub-store-copilot/sync-result.json
fi
```

#### 2.2 极简 Python 触发脚本（零外部依赖，用完即走）

```python
import os
import urllib.request
import urllib.error

sub_store_url = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")
sync_endpoint = f"{sub_store_url}/api/sync/artifacts"

req = urllib.request.Request(sync_endpoint, method="POST")
try:
    with urllib.request.urlopen(req, timeout=15) as resp:
        body = resp.read().decode("utf-8")
        print(f"Gist 同步成功 [{resp.status}]: {body}")
except urllib.error.HTTPError as e:
    err_body = e.read().decode("utf-8", errors="ignore")
    print(f"Gist 同步失败 [{e.code}]: {err_body}")
except urllib.error.URLError as e:
    print(f"网络连接错误: {e.reason}")
```

### 3. 控制台同步操作心智模型

- **Upload（打包推送）**：
  将当前 Sub-Store 中的所有配置（自建 Manual 节点、Remote 订阅、聚合集合、算子流水线及虚拟 JS 脚本）全量打包并推送到目标 Gist。若远程已存在内容，将以本地状态覆盖远程 Gist。
- **Download（拉取恢复）**：
  从目标 Gist 拉取历史配置包就地反序列化，覆盖还原当前的订阅、集合与自定义脚本，适用于新实例初始化、配置迁移或灾备恢复。

### 4. 常见同步故障排查

| 状态码 / 现象 | 根因分析 | 解决措施 |
|---|---|---|
| `401 Bad credentials` | GitHub Token 失效、已过期或缺少 `gist` 作用域 | 重新生成具备 `gist` scope 的 GitHub PAT，更新 `gistToken` 设置 |
| `404 Not Found` | Gist ID 错误，或目标 Gist 在 GitHub 尚未创建 | 在 GitHub 网页上手动创建一个 Secret Gist，获取其 Hash ID 填入 `gistId` |
| `504 Gateway Timeout` | Sub-Store 服务所在网络无法直连 `api.github.com` | 在 settings 中配置 `proxy` 出站代理，或将 `gistApi` 设为可用的反向代理端点 |

---

## 五、其他常规配置 (Other Configuration)

其他配置严格对齐前端 Web 控制台的 `Other Configuration` 区块，用于控制 Sub-Store 的默认导出平台、节点展示偏好以及默认排序规则。

### 1. 关键设置项字典

| 配置项 | 数据类型 | 默认值 | 说明与取值范围 |
|---|---|---|---|
| `target` | string | `clash` | 默认下载平台格式。当客户端访问 `/download/...` 端点且未在 URL 参数显式声明 `?target=` 时，默认以此格式输出。常见取值：`clash-meta`（Mihomo）、`clash`、`surge`、`singbox`、`loon`、`v2ray`、`shadowrocket`。 |
| `removeEmoji` | boolean | `false` | 全局是否剔除节点名称中的 Emoji 图标。设为 `true` 后，所有订阅在下载导出时将自动过滤掉国家旗帜及各类 Emoji 字符。 |
| `sortMethod` | string | `name` | 节点默认排序依据字段（如 `name` 按节点名称排序）。 |
| `sortOrder` | string | `asc` | 节点默认排序方向：`asc`（升序）或 `desc`（降序）。 |

### 2. 增量配置范例

先拉取现有配置，增量更新默认平台格式、Emoji 剔除开关与排序方向，完整回写：

```bash
curl -s "${SUB_STORE_URL}/api/settings" | jq '
  .target = "singbox" |
  .removeEmoji = true |
  .sortMethod = "name" |
  .sortOrder = "asc"
' | curl -s -X POST "${SUB_STORE_URL}/api/settings" \
  -H "Content-Type: application/json" \
  --data-binary @-
```

---

## 六、操作方式与纯 HTTP 准则

本规范的所有能力均严格遵循纯 HTTP 操作准则：

1. **纯粹通过 HTTP 端点交互**：
   - 读取设置：`GET /api/settings`
   - 修改设置：`POST /api/settings`
   - 触发备份：`POST /api/sync/artifacts`
2. **强制先拉取后更新的增量模式**：
   - 更新任何配置项时，必须严格执行“先 GET 拉取 -> 内存合并 -> POST 回写”三步法。
   - 绝对严禁直接发送部分字段进行盲目覆盖，彻底防止冲掉 Gist 凭证或网络出站配置。
3. **零环境侵入与零部署**：
   - 纯粹面向已有运行中的 Sub-Store 实例。
   - 绝对不包含任何环境部署、Docker 安装、容器生命周期管理、系统守护进程配置或宿主机环境变量注入。
