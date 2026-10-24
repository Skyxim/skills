# Sub-Store 聚合集合接口规范 (`/api/collections`)

Sub-Store 集合（Collection）用于将多个独立的订阅（包括自建 Manual 节点与远程机场订阅）聚合为一个统一的节点池，并在导出前按序挂载算子流水线（Process Pipeline）进行统一的清洗、重命名、排序和去重。

所有端点均基于当前模式的前缀 URL `{SUB_STORE_URL}`。

---

## 1. 端点概览

| 请求方法 | 端点路径 | 功能描述 |
|---|---|---|
| `GET` | `/api/collections` | 获取全量集合列表 |
| `GET` | `/api/collection/:name` | 获取指定名称的单个集合详情 |
| `POST` | `/api/collections` | 创建或全量覆写集合配置与算子流水线 |
| `DELETE` | `/api/collection/:name` | 删除指定名称的集合 |

---

## 2. 请求字段规范字典

在调用 `POST /api/collections` 创建或更新集合时，请求体为 JSON 对象：

| 字段名 | 类型 | 必需 | 默认值 | 说明 |
|---|---|---|---|---|
| `name` | string | 是 | 无 | 集合唯一名称 |
| `type` | string | 否 | `collection` | 固定为 `collection` |
| `subscriptions` | array of string | 是 | 无 | 聚合引用的订阅名称列表（对应已存在的订阅 `name`） |
| `process` | array of object | 否 | `[]` | 算子流水线配置列表（按声明顺序自上而下链式执行） |

> [!IMPORTANT]
> **前端 Web UI 表单双向绑定约束**：
> 在 `process` 数组中声明各个算子时，必须严格按照 [内置算子参数规范字典](api-operators.md) 传入全量字段（例如 `Script Operator` 必须包含 `mode: "link"` 与 `content`；`Regex Filter` 必须包含 `action` 与 `keep` 联动）。切勿仅满足后端下载执行而传入简写或模糊字段，防止用户在 Web 控制台编辑时表单控件脱节留白并在保存时覆写损毁配置。

---

## 3. 请求与响应示例

### 3.1 获取所有集合
- **Method**: `GET`
- **URL**: `{SUB_STORE_URL}/api/collections`
- **Response (200 OK)**: 包含所有集合配置对象的 JSON 数组。

### 3.2 创建多源聚合集合并装配流水线
下面示例将自建节点 `My-Manual-Nodes` 与机场订阅 `Airport-Main` 进行聚合，并挂载定制脚本、正则过滤、国旗补全、排序与去重算子：

```bash
curl -s -X POST "${SUB_STORE_URL}/api/collections" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "All-Nodes",
    "type": "collection",
    "subscriptions": [
      "Airport-Main",
      "My-Manual-Nodes"
    ],
    "process": [
      {
        "type": "Script Operator",
        "args": {
          "mode": "link",
          "content": "/api/file/cleaner.js",
          "arguments": {}
        }
      },
      {
        "type": "Regex Filter",
        "args": {
          "pattern": "(?i)官网|到期|剩余|通知|重置",
          "action": "drop",
          "keep": false
        }
      },
      {
        "type": "Flag",
        "args": {
          "mode": "add"
        }
      },
      {
        "type": "Sort",
        "args": {
          "field": "name",
          "order": "asc"
        }
      },
      {
        "type": "Deduplicate",
        "args": {
          "field": "name"
        }
      }
    ]
  }'
```

### 3.3 删除指定集合
```bash
curl -s -X DELETE "${SUB_STORE_URL}/api/collection/All-Nodes"
```

---

## 4. 增量更新集合流水线规范（严禁破坏性覆盖）

Sub-Store 的 `POST /api/collections` 接口以 `name` 为唯一标识。如果同名集合已存在，该接口会**以请求体中的内容做全量替换**。

> [!CAUTION]
> **全量破坏性覆盖的严重后果**：
> 1. **算子清空丢失**：若用户要求“加一个脚本算子”或“修改正则过滤词”，若直接提交仅包含该算子的 `process` 数组，该集合中原有的 Quick Setting、Flag 国旗、Sort 排序、Deduplicate 去重等其他算子将被彻底抹除！
> 2. **订阅源关联丢失**：若在提交对象中漏传了 `subscriptions` 数组，集合将失去所有绑定的节点来源，下游拉取时变为空列表！
> 3. **元数据丢失**：原有集合若配置了 `remark`、`tag` 或其他高级字段，未带上的字段均会丢失。

### 4.1 增量修改标准三步法（3-Step Incremental Pattern）
所有的算子与集合调整必须严格遵循以下三步：
1. **Read (读)**：`GET /api/collection/:name` 获取当前完整集合配置对象；
2. **Mutate (增量改)**：在内存或 JSON 结构中仅针对目标项做局部的修改或追加，完整保留 `subscriptions` 列表与既有所有算子；
3. **Write-back (回写)**：将合并后的全量 JSON 对象 `POST /api/collections` 保存生效。

### 4.2 常用增量操作标准范例

#### 范例 A：安全向现有集合追加新算子（curl + jq 管道流）
```bash
COLL_NAME="All-Nodes"
# 待追加的算子对象（必须符合前端 UI 全量 Schema）
NEW_OP='{
  "type": "Script Operator",
  "args": {
    "mode": "link",
    "content": "/api/file/optimize-net.js",
    "arguments": {}
  }
}'

curl -s "${SUB_STORE_URL}/api/collection/${COLL_NAME}" | jq --argjson op "$NEW_OP" '
  .process = (.process // []) + [$op]
' | curl -s -X POST "${SUB_STORE_URL}/api/collections" \
  -H "Content-Type: application/json" \
  --data-binary @-
```

#### 范例 B：安全修改现有集合中的指定算子参数（curl + jq 管道流）
仅修改 `Regex Filter` 的正则匹配表达式，不触碰任何其他算子：
```bash
COLL_NAME="All-Nodes"

curl -s "${SUB_STORE_URL}/api/collection/${COLL_NAME}" | jq '
  .process = (.process | map(
    if .type == "Regex Filter" then
      .args.pattern = "(?i)官网|到期|剩余|通知|重置|活动|公告"
    else . end
  ))
' | curl -s -X POST "${SUB_STORE_URL}/api/collections" \
  -H "Content-Type: application/json" \
  --data-binary @-
```

#### 范例 C：Python 增量更新指定集合算子（保留全量元数据）
在现场手写临时 Python 脚本时，使用字典解包合并保证零字段丢失：
```python
import os, json, urllib.request

BASE_URL = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")
COLL_NAME = "All-Nodes"

# 1. Read: 读取当前完整对象
req = urllib.request.Request(f"{BASE_URL}/api/collection/{COLL_NAME}")
with urllib.request.urlopen(req) as resp:
    current_col = json.loads(resp.read().decode("utf-8"))

# 2. Mutate: 增量追加或更新算子，严格保留 subscriptions 与既有算子
process = current_col.get("process", [])
target_op = {
    "type": "Script Operator",
    "args": {"mode": "link", "content": "/api/file/optimize-net.js", "arguments": {}}
}

# 检查是否已存在同名脚本算子；若存在则更新，不存在则前置追加
existing_idx = next(
    (i for i, op in enumerate(process) 
     if op.get("type") == "Script Operator" and op.get("args", {}).get("content") == target_op["args"]["content"]),
    None
)
if existing_idx is not None:
    process[existing_idx] = target_op
else:
    process.insert(0, target_op)

current_col["process"] = process

# 3. Write-back: 回写完整对象
save_req = urllib.request.Request(
    f"{BASE_URL}/api/collections",
    data=json.dumps(current_col).encode("utf-8"),
    headers={"Content-Type": "application/json"},
    method="POST"
)
with urllib.request.urlopen(save_req) as resp:
    print(f"增量更新集合 [{COLL_NAME}] 成功 (HTTP {resp.status})")
```
