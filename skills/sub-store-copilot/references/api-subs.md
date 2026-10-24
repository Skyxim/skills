# Sub-Store 订阅管理接口规范 (`/api/subs`)

Sub-Store 订阅（Subscription）用于录入和维护代理节点源。订阅分为两大类型：
1. **自建节点（Manual 模式）**：内联裸节点 URI 列表（vmess, vless, ss, trojan, hysteria2 等）。
2. **机场订阅（Remote 模式）**：远程机场 HTTP/HTTPS 订阅链接。

所有端点均基于当前模式的前缀 URL `{SUB_STORE_URL}`。

---

## 1. 端点概览

| 请求方法 | 端点路径 | 功能描述 |
|---|---|---|
| `GET` | `/api/subs` | 获取全量订阅列表（包含 Manual 与 Remote） |
| `GET` | `/api/sub/:name` | 获取指定名称的单个订阅详情 |
| `POST` | `/api/subs` | 创建或全量覆写订阅配置 |
| `DELETE` | `/api/sub/:name` | 删除指定名称的订阅 |

---

## 2. 请求字段规范字典

在调用 `POST /api/subs` 创建或覆写订阅时，请求体为 JSON 对象，包含以下字段：

| 字段名 | 类型 | 必需 | 默认值 | 说明 |
|---|---|---|---|---|
| `name` | string | 是 | 无 | 订阅唯一标识名称 |
| `type` | string | 否 | `subscription` | 订阅类型：自建多协议节点填 `manual`，远程机场填 `subscription` |
| `source` | string | 否 | `remote` | 数据源位置：自建本地填 `local`，远程填 `remote` |
| `url` | string | 条件必需 | 空字符串 | 远程机场订阅链接；当 `source` 为 `local` 时必须留空 |
| `content` | string | 条件必需 | 空字符串 | 自建节点链接纯文本（换行符 `\n` 分隔）；当 `source` 为 `remote` 时留空 |
| `enabled` | boolean | 否 | `true` | 是否启用该订阅 |
| `insecure` | boolean | 否 | `false` | 拉取远程订阅时是否允许跳过 TLS 证书合法性校验 |
| `process` | array of object | 否 | `[]` | 针对此订阅专享绑定的算子流数组 |

> [!IMPORTANT]
> **前端 Web UI 表单双向绑定约束**：
> 在调用 `POST /api/subs` 录入自建 Manual 节点或远程机场订阅时，必须携带与前端 UI 表单完全对应的规范字段（Manual 节点显式声明 `source: "local"`, `type: "manual"`, `url: ""`；Remote 机场订阅显式声明 `source: "remote"`, `type: "subscription"`, `content: ""`）。若省略 `source` 或类型字段，前端 Web 控制台在切换订阅编辑视图时可能会出现表单形态渲染错乱或模式未识别。若订阅绑定了 `process` 算子流水线，算子构造同样必须严格遵循 [内置算子参数规范字典](api-operators.md)。

---

## 3. 请求与响应示例

### 3.1 获取所有订阅
- **Method**: `GET`
- **URL**: `{SUB_STORE_URL}/api/subs`
- **Response (200 OK)**:
```json
[
  {
    "name": "My-Manual-Nodes",
    "type": "manual",
    "source": "local",
    "url": "",
    "content": "trojan://password@1.2.3.4:443#%E6%97%A5%E6%9C%AC-VPS\nss://YWVzLTEyOC1nY206cGFzc3dvcmQ@2.2.2.2:8388#%E9%A6%99%E6%B8%AF-01",
    "enabled": true,
    "process": []
  },
  {
    "name": "Airport-Main",
    "type": "subscription",
    "source": "remote",
    "url": "https://api.airport.com/link?token=<SECRET>",
    "enabled": true,
    "insecure": false,
    "process": []
  }
]
```

### 3.2 创建自建 Manual 节点订阅
```bash
curl -s -X POST "${SUB_STORE_URL}/api/subs" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My-Manual-Nodes",
    "type": "manual",
    "source": "local",
    "url": "",
    "content": "trojan://pass@1.1.1.1:443#JP-Node\nss://YWVzLTEyOC1nY206cGFzc3dvcmQ@2.2.2.2:8388#HK-Node\nhysteria2://user:pass@3.3.3.3:443/?sni=example.com#US-Hysteria",
    "enabled": true
  }'
```

### 3.3 创建远程机场订阅
```bash
curl -s -X POST "${SUB_STORE_URL}/api/subs" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Airport-Main",
    "type": "subscription",
    "source": "remote",
    "url": "https://airport.example.com/sub?token=<SECRET>",
    "enabled": true,
    "insecure": false
  }'
```

### 3.4 删除指定订阅
```bash
curl -s -X DELETE "${SUB_STORE_URL}/api/sub/Airport-Main"
```

---

## 4. 增量追加节点与增量更新规范（严禁破坏性覆盖）

Sub-Store 的 `POST /api/subs` 接口以 `name` 为主键进行整包覆写。如果指定名称的订阅已存在，请求体中的 JSON 将**全量替换**该订阅对象。

> [!CAUTION]
> **全量破坏性覆盖的严重后果**：
> 在自建 Manual 订阅中，所有节点 URI（vmess, vless, ss, trojan, hysteria2 等）以换行符 `\n` 拼接存储在 `content` 字符串中。
> 若用户要求“追加一个新节点”，如果直接构造仅包含新节点的 JSON 对象 POST，**原有订阅中的所有其他已有节点将瞬间被全部清空抹去**！

### 4.1 增量修改标准三步法（3-Step Incremental Pattern）
向自建节点订阅追加新节点或更新配置时，必须严格执行三步法：
1. **Read (读)**：`GET /api/sub/:name` 获取当前订阅完整 JSON 对象（包含已有 `content` 及其他元数据）；
2. **Mutate (增量改)**：将新节点 URI 用换行符 `\n` 拼接在已有 `content` 末尾，或进行去重合并；保留所有既有字段（如 `source: "local"`, `type: "manual"`, `url: ""` 及绑定的 `process` 算子）；
3. **Write-back (回写)**：将合并后的完整 JSON 对象 `POST /api/subs` 写回保存（或在仅修改内容时通过 `PATCH /api/sub/:name` 提交更新字段）。

### 4.2 常用增量操作标准范例

#### 范例 A：安全向已有订阅追加新节点（curl + jq 管道流）
```bash
SUB_NAME="My-Manual-Nodes"
NEW_NODE="hysteria2://user:pass@4.4.4.4:443/?sni=example.com#%E6%96%B0%E8%8A%82%E7%82%B9-02"

curl -s "${SUB_STORE_URL}/api/sub/${SUB_NAME}" | jq --arg n "$NEW_NODE" '
  .content = (if .content and (.content | length > 0) then (.content + "\n" + $n) else $n end)
' | curl -s -X POST "${SUB_STORE_URL}/api/subs" \
  -H "Content-Type: application/json" \
  --data-binary @-
```

#### 范例 B：Python 增量追加节点并执行物理指纹排重（零第三方依赖）
在现场编写临时 Python 脚本时，防止向自建订阅中重复添加相同节点：
```python
import os, json, re, hashlib, urllib.request

BASE_URL = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")
SUB_NAME = "My-Manual-Nodes"
NEW_NODES = [
    "trojan://pass@1.1.1.1:443#%E6%97%A5%E6%9C%AC-01",
    "hysteria2://user:pass@6.6.6.6:443/?sni=new.vps.com#%E6%96%B0%E5%8A%A0%E5%9D%A1-03"
]

def node_fp(uri):
    m = re.match(r"^([a-z0-9+-]+)://([^@]+@)?([^/?#:]+)(?::(\d+))?", uri, re.I)
    return hashlib.md5(f"{m.group(1)}://{m.group(2) or ''}{m.group(3)}:{m.group(4) or ''}".lower().encode()).hexdigest() if m else uri

# 1. Read: 获取现有订阅
req = urllib.request.Request(f"{BASE_URL}/api/sub/{SUB_NAME}")
with urllib.request.urlopen(req) as resp:
    sub_data = json.loads(resp.read().decode("utf-8"))

existing_nodes = [l.strip() for l in sub_data.get("content", "").splitlines() if l.strip()]
existing_fps = {node_fp(n) for n in existing_nodes}

# 2. Mutate: 增量并入新节点（仅追加指纹不存在的新节点）
added_count = 0
for n in NEW_NODES:
    fp = node_fp(n)
    if fp not in existing_fps:
        existing_nodes.append(n)
        existing_fps.add(fp)
        added_count += 1

sub_data["content"] = "\n".join(existing_nodes)

# 3. Write-back: 回写保存完整对象
save_req = urllib.request.Request(
    f"{BASE_URL}/api/subs",
    data=json.dumps(sub_data).encode("utf-8"),
    headers={"Content-Type": "application/json"},
    method="POST"
)
with urllib.request.urlopen(save_req) as resp:
    print(f"增量追加完成! 新增 {added_count} 个节点，当前总计 {len(existing_nodes)} 个节点 (HTTP {resp.status})")
```
