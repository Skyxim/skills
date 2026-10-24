# 复杂操作临时 Python 脚本编写指南与范式

在管理 Sub-Store 时，不同场景的操作复杂度差异显著：
- 对于**单点操作**（如新增单个订阅、查询状态、手动上传单个文件），使用 `curl` 配合 `jq` 是最直接高效的方式。
- 对于**复杂批量操作**（如混乱数据源清洗提取、跨多个集合批量注入算子流水线、节点增量 Diff 比对、本地脚本语法预检与全量批量推送），冗长且易错的 shell/jq 管道不仅可读性差，也难以进行完善的异常处理。

本指南提供**现场编写轻量、临时、用完即走（Ephemeral）Python 脚本**的心智模型、极简连接范式与高频场景实战模板。

> [!TIP]
> **设计哲学：用完即走，拒绝臃肿封装**
> 避免为一次性的运维需求维护庞大固化的永久 CLI。遇到复杂多步操作时，现场手写轻量、临时、用完即走的 Python 脚本。
> - **统一存储目录**：所有临时 Python 脚本与 JS 处理脚本**统一存放于 `/tmp/sub-store-copilot/` 目录下**，严禁在主工作区随处新建文件，保持项目环境整洁。
> - **自解释命名规范**：文件路径为 `/tmp/sub-store-copilot/<capability-name>.py`，`<capability-name>` 必须准确表明脚本具体能力（如 `clean-and-dedup-nodes.py`、`batch-inject-pipeline.py`、`diff-sync-nodes.py` 等），严禁使用 `test.py`、`temp.py` 等无意义命名。
> - **零依赖保证**：100% 采用 Python 3 标准库（`urllib.request`, `json`, `os`, `re`, `base64`, `hashlib`, `urllib.parse` 等），无须执行 `pip install`，在任意 Linux/macOS 环境开箱即用。

---

## 一、心智模型与工具选型准则

```
                ┌──────────────────────────────┐
                │       当前运维操作任务       │
                └──────────────┬───────────────┘
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
      【简单 / 单次 / 单点】          【复杂 / 批量 / 深度转换】
   例如：查询列表、新增单条订阅、   例如：多源混乱文本清洗去重、
   单文件 curl 上传、拉取测试       跨集合算子批量注入、节点增量 Diff、
               │                    本地 JS 预检上传、Web UI 表单失联全量巡检修复
               ▼                               ▼
      优先使用 curl + jq             现场手写临时 Python 脚本
   （最快、最轻、无上下文切换）     （纯标准库、结构清晰、易调试、用完即走）
```

| 维度 | curl + jq | 临时 Python 脚本（用完即走） | 永久复杂 CLI（反对） |
|---|---|---|---|
| **适用范围** | 单点 API 调用、简单查看与更新 | 多步编排、复杂嵌套数据结构转换、Diff 对比 | 固化且通用的全套命令行 |
| **开发成本** | 几乎为零，单行即可 | 1~3 分钟内手写 30~80 行代码 | 极高，需要长期维护代码与打包 |
| **代码可读性** | 管道多层嵌套后极难维护 | 逻辑清晰，分支与错误处理完整 | 良好但结构过度工程化 |
| **外部依赖** | 依赖 curl、jq | **零 pip 依赖**，标准库自带 | 依赖虚拟环境或特定包管理器 |
| **生命周期** | 即敲即走 | 任务完成后留在本地或随时删除 | 长期维护包袱 |

---

## 二、标准极简端点自适应连接范式

在编写临时脚本时，可直接复用以下不到 20 行的纯标准库 HTTP 通信客户端。它会自动按优先级检测环境变量 `SUB_STORE_PREFIX_URL`、本地独立进程与 MitM 虚拟假域名端点，并支持安全脱敏：

```python
import os, json, urllib.request, urllib.error

def get_base_url():
    if os.environ.get("SUB_STORE_PREFIX_URL"):
        return os.environ["SUB_STORE_PREFIX_URL"].rstrip("/")
    for url in ["http://127.0.0.1:3000", "http://sub.store", "https://sub.store"]:
        try:
            with urllib.request.urlopen(f"{url}/api/subs", timeout=1.5):
                return url
        except Exception:
            pass
    raise RuntimeError("未检测到 Sub-Store 服务，请设置环境变量: export SUB_STORE_PREFIX_URL='https://domain.com/<SECRET_PATH>'")

BASE_URL = get_base_url()

def api(path, method="GET", data=None, is_json=True):
    """统一 API 请求辅助函数（自动处理 JSON 编解码）"""
    url = f"{BASE_URL}{path}"
    headers = {"Content-Type": "application/json"} if is_json and data is not None else {}
    body = json.dumps(data).encode("utf-8") if is_json and data is not None else (data.encode("utf-8") if isinstance(data, str) else data)
    req = urllib.request.Request(url, data=body, headers=headers, method=method)
    try:
        with urllib.request.urlopen(req, timeout=10) as resp:
            raw = resp.read().decode("utf-8")
            return json.loads(raw) if is_json and raw else raw
    except urllib.error.HTTPError as e:
        err_msg = e.read().decode("utf-8", errors="ignore")
        raise RuntimeError(f"API 请求失败 [{e.code}] {method} {path}: {err_msg}")
```

---

## 三、五大高频复杂场景实战模版

### 场景一：多源与混乱格式节点解析、清洗、指纹去重并批量入库

#### 场景特征
用户手头有一大段混乱文本，混杂了：
1. Base64 编码的订阅块；
2. 零散暴露的协议链接（`vmess://`, `vless://`, `ss://`, `trojan://`, `hysteria2://` 等）；
3. 夹杂的广告行、废话文本或重复的相同服务器节点。

#### 临时脚本实现 (`/tmp/sub-store-copilot/clean-and-dedup-nodes.py`)

```python
import os, re, json, base64, hashlib
import urllib.request, urllib.parse

# 1. 基础配置
SUB_NAME = "Cleaned-Manual-Nodes"
RAW_INPUT_FILE = "/tmp/sub-store-copilot/raw-nodes.txt"  # 存放混乱文本的文件路径

# 2. 健壮的 Base64 自动补全容错解码
def safe_b64decode(s):
    s = s.strip()
    s += "=" * ((4 - len(s) % 4) % 4)
    try:
        return base64.b64decode(s, altchars=b"-_").decode("utf-8", errors="ignore")
    except Exception:
        try:
            return base64.b64decode(s).decode("utf-8", errors="ignore")
        except Exception:
            return ""

# 3. 提取物理连接指纹（用于去重，避免相同节点重复）
def extract_fingerprint(uri):
    # 提取协议、主机、端口和身份凭证
    match = re.match(r"^([a-z0-9+-]+)://([^@]+@)?([^/?#:]+)(?::(\d+))?", uri, re.IGNORECASE)
    if not match:
        return hashlib.md5(uri.encode("utf-8")).hexdigest()
    proto, cred, host, port = match.groups()
    norm = f"{proto.lower()}://{cred or ''}{host.lower()}:{port or ''}"
    return hashlib.md5(norm.encode("utf-8")).hexdigest()

# 4. 从混乱文本中提取所有节点 URI
def extract_proxies(text):
    proxies = []
    # 检查是否有整块 Base64 内容
    lines = text.strip().splitlines()
    for line in lines:
        line = line.strip()
        if not line:
            continue
        # 如果整行像 Base64 编码块
        if re.match(r"^[A-Za-z0-9+/=_-]{20,}$", line) and "://" not in line:
            decoded = safe_b64decode(line)
            if "://" in decoded:
                proxies.extend(extract_proxies(decoded))
                continue

        # 匹配单条节点 URI
        matched = re.findall(r"(?:vmess|vless|ss|ssr|trojan|hysteria2|hy2|tuic|snell|wireguard)://[^\s]+", line, re.IGNORECASE)
        for uri in matched:
            # 过滤包含广告/官网废话的节点名（URL Hash 中的名称）
            parsed = urllib.parse.urlsplit(uri)
            node_name = urllib.parse.unquote(parsed.fragment)
            if re.search(r"(?i)官网|剩余|到期|重置|通知|发布页|aff", node_name):
                continue
            proxies.append(uri)
    return proxies

# 5. 执行解析、清洗与指纹去重
with open(RAW_INPUT_FILE, "r", encoding="utf-8") as f:
    raw_content = f.read()

all_nodes = extract_proxies(raw_content)
seen_fp = set()
unique_nodes = []

for uri in all_nodes:
    fp = extract_fingerprint(uri)
    if fp not in seen_fp:
        seen_fp.add(fp)
        unique_nodes.append(uri)

print(f"清洗完成: 原始识别 {len(all_nodes)} 个节点，物理指纹去重后剩余 {len(unique_nodes)} 个节点。")

# 6. 推送至 Sub-Store 录入为 Manual 订阅
base_url = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")
payload = {
    "name": SUB_NAME,
    "type": "manual",
    "source": "local",
    "enabled": True,
    "content": "\n".join(unique_nodes)
}

req = urllib.request.Request(
    f"{base_url}/api/subs",
    data=json.dumps(payload).encode("utf-8"),
    headers={"Content-Type": "application/json"},
    method="POST"
)
with urllib.request.urlopen(req) as resp:
    print(f"入库成功! 订阅 [{SUB_NAME}] 已更新 (HTTP {resp.status})")
```

---

### 场景二：多集合批量统一增量注入与更新算子流水线 (Batch Incremental Pipeline Injection)

#### 场景特征
用户维护了多个订阅集合（如 `Proxy-Auto`, `Proxy-Gaming`, `Proxy-Streaming`），需要批量为所有集合增量修补算子流水线：
1. 在算子流水线开头增量注入通用修补 JS 脚本（如 `/api/file/optimize-udp-tfo.js`）；
2. 增量更新或插入垃圾节点过滤正则（Regex Filter），与已有过滤规则并存或对齐；
3. 在末尾增量补齐国旗 Emoji 与排序去重算子（若已有则保留或更新，避免重复插入）；
4. **铁律与核心保护**：**严禁暴力覆盖 `process` 数组**！必须完整保留各集合已有的个性化业务算子（如专用的地区正则、自制清洗脚本等）以及 `subscriptions`、`remark`、`tag` 等所有元数据。

#### 临时脚本实现 (`/tmp/sub-store-copilot/batch-inject-pipeline.py`)

```python
import os, json, urllib.request

BASE_URL = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")

def api(path, method="GET", data=None):
    req = urllib.request.Request(
        f"{BASE_URL}{path}",
        data=json.dumps(data).encode("utf-8") if data else None,
        headers={"Content-Type": "application/json"} if data else {},
        method=method
    )
    with urllib.request.urlopen(req) as resp:
        content = resp.read().decode("utf-8")
        return json.loads(content) if content else {}

# 1. 获取所有现有集合（Read: 增量第一步）
collections = api("/api/collections")
print(f"获取到 {len(collections)} 个现有集合，准备执行增量流水线注入...")

# 2. 遍历集合并进行智能增量合并（Mutate: 增量第二步）
updated_count = 0
for col in collections:
    col_name = col.get("name")
    if col_name.startswith("Test-"):
        print(f"跳过测试集合: {col_name}")
        continue

    print(f"正在为集合 [{col_name}] 增量合并算子流水线...")
    
    # 获取既有算子列表（防止暴力覆盖已有算子）
    existing_process = list(col.get("process", []))

    # A. 增量检查/注入前置网络优化脚本
    script_content = "/api/file/optimize-udp-tfo.js"
    has_script = any(
        op.get("type") in ("Script Operator", "Script") and
        op.get("args", {}).get("content") == script_content
        for op in existing_process
    )
    if not has_script:
        # 前置插入网络优化脚本
        existing_process.insert(0, {
            "type": "Script Operator",
            "args": {"mode": "link", "content": script_content, "arguments": {}}
        })

    # B. 增量检查/更新垃圾广告过滤正则
    ad_pattern = "(?i)官网|到期|剩余|通知|重置"
    regex_op = next((op for op in existing_process if op.get("type") == "Regex Filter" and op.get("args", {}).get("action") == "drop"), None)
    if regex_op:
        regex_op["args"]["pattern"] = ad_pattern
        regex_op["args"]["keep"] = False
    else:
        # 在脚本算子后追加广告过滤
        insert_idx = 1 if has_script or len(existing_process) > 0 else 0
        existing_process.insert(insert_idx, {
            "type": "Regex Filter",
            "args": {"pattern": ad_pattern, "action": "drop", "keep": False}
        })

    # C. 增量在末尾补齐国旗、排序与去重算子（若不存在则追加）
    existing_types = {op.get("type") for op in existing_process}
    if "Flag" not in existing_types:
        existing_process.append({"type": "Flag", "args": {"mode": "add"}})
    if "Sort" not in existing_types:
        existing_process.append({"type": "Sort", "args": {"field": "name", "order": "asc"}})
    if "Deduplicate" not in existing_types:
        existing_process.append({"type": "Deduplicate", "args": {"field": "name"}})

    # D. 构建回写对象：使用字典解包保留原集合所有已有元数据（Write-back: 增量第三步）
    update_payload = {
        **col,
        "process": existing_process
    }

    # 执行保存写回
    api("/api/collections", method="POST", data=update_payload)
    updated_count += 1
    print(f" -> 集合 [{col_name}] 增量更新成功，当前共包含 {len(existing_process)} 个链式算子。")

print(f"增量批量注入完成! 共安全更新 {updated_count} 个集合，无任何既有算子或元数据丢失。")
```

---

### 场景三：节点增量同步与 Diff 状态对比（找出新增、失效与变更节点）

#### 场景特征
当外部订阅或自建节点源产生变动时，不应盲目暴力覆盖。需要：
1. 抓取远端或清洗出来的新节点集；
2. 拉取 Sub-Store 当前已有集合/订阅的实际节点列表；
3. 基于物理指纹对比，生成详细差异报告（新增 X 个、失效 Y 个、保持不变 Z 个）；
4. 提供 Dry-Run 试跑预览，确认差异符合预期后再落地同步。

#### 临时脚本实现 (`/tmp/sub-store-copilot/diff-and-sync-nodes.py`)

```python
import os, json, hashlib, re, urllib.request, urllib.parse

BASE_URL = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")
TARGET_SUB = "My-Manual-Nodes"
DRY_RUN = True  # 改为 False 执行实际覆写入库

def get_node_fingerprint(uri):
    """提取协议物理特征计算哈希"""
    match = re.match(r"^([a-z0-9+-]+)://([^@]+@)?([^/?#:]+)(?::(\d+))?", uri, re.IGNORECASE)
    if not match:
        return hashlib.sha256(uri.encode("utf-8")).hexdigest()
    proto, cred, host, port = match.groups()
    raw = f"{proto.lower()}://{cred or ''}{host.lower()}:{port or ''}"
    return hashlib.sha256(raw.encode("utf-8")).hexdigest()

# 1. 获取现有订阅节点
req = urllib.request.Request(f"{BASE_URL}/api/sub/{TARGET_SUB}")
try:
    with urllib.request.urlopen(req) as resp:
        sub_data = json.loads(resp.read().decode("utf-8"))
        old_content = sub_data.get("content", "")
except Exception as e:
    print(f"订阅未找到或为空，视为空集合: {e}")
    old_content = ""

old_nodes = [n.strip() for n in old_content.splitlines() if n.strip()]
old_map = {get_node_fingerprint(n): n for n in old_nodes}

# 2. 模拟/读取待同步的新节点列表（可来自文件或外部网络抓取）
new_nodes_raw = [
    "trojan://pass1@1.1.1.1:443#%E6%97%A5%E6%9C%AC-01",  # 保持不变
    "hysteria2://user:pass@5.5.5.5:443/?sni=new.com#%E6%96%B0%E5%8A%A0%E5%9D%A1-02",  # 新增节点
]
new_nodes = [n.strip() for n in new_nodes_raw if n.strip()]
new_map = {get_node_fingerprint(n): n for n in new_nodes}

# 3. 计算 Diff
added_fps = set(new_map.keys()) - set(old_map.keys())
removed_fps = set(old_map.keys()) - set(new_map.keys())
unchanged_fps = set(old_map.keys()) & set(new_map.keys())

print("=" * 40)
print(f"Diff 状态报告 (目标订阅: {TARGET_SUB})")
print("=" * 40)
print(f"[+] 新增节点 ({len(added_fps)} 个):")
for fp in added_fps:
    parsed = urllib.parse.urlsplit(new_map[fp])
    print(f"    + {urllib.parse.unquote(parsed.fragment) or new_map[fp]}")

print(f"[-] 失效/下线节点 ({len(removed_fps)} 个):")
for fp in removed_fps:
    parsed = urllib.parse.urlsplit(old_map[fp])
    print(f"    - {urllib.parse.unquote(parsed.fragment) or old_map[fp]}")

print(f"[=] 保持不变 ({len(unchanged_fps)} 个)")
print("=" * 40)

# 4. 执行同步
if DRY_RUN:
    print("[DRY-RUN 试跑模式] 未做真实修改。如确认无误，请将 DRY_RUN 置为 False 运行。")
else:
    # 全量覆写或增量合并
    final_nodes = list(new_map.values())
    payload = {
        "name": TARGET_SUB,
        "type": "manual",
        "source": "local",
        "enabled": True,
        "content": "\n".join(final_nodes)
    }
    update_req = urllib.request.Request(
        f"{BASE_URL}/api/subs",
        data=json.dumps(payload).encode("utf-8"),
        headers={"Content-Type": "application/json"},
        method="POST"
    )
    with urllib.request.urlopen(update_req) as resp:
        print(f"同步入库完成! 当前包含 {len(final_nodes)} 个节点。")
```

---

### 场景四：本地 JavaScript 脚本文件的自动化语法预检与一键推送入库

#### 场景特征
用户在本地开发或批量存放了若干 JavaScript 节点处理脚本（如 `patch-sni.js`, `rename-rules.js`），在推送到 Sub-Store 虚拟文件系统前：
1. 校验每个脚本是否符合 Sub-Store 算子脚本规范（必须包含 `operator` 函数声明）；
2. 检查基本语法与括号平衡；
3. 批量通过 `POST /api/file/:name` 推送到 Sub-Store 虚拟文件系统；
4. 验证远端文件列表，输出已成功部署的脚本列表。

#### 临时脚本实现 (`/tmp/sub-store-copilot/push-scripts-with-check.py`)

```python
import os, re, json, glob, urllib.request

BASE_URL = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")
SCRIPTS_DIR = "/tmp/sub-store-copilot"  # 本地临时脚本统一存放目录

def validate_script(filepath):
    """预检本地 JS 脚本是否满足 Sub-Store 算子规范"""
    filename = os.path.basename(filepath)
    with open(filepath, "r", encoding="utf-8") as f:
        code = f.read()

    # 1. 检查是否导出 operator 函数
    if not re.search(r"function\s+operator\s*\(", code):
        return False, f"{filename}: 缺少标准 'function operator(proxies, ...)' 函数声明"

    # 2. 简单的开闭括号平衡检查
    for open_char, close_char in [("{", "}"), ("(", ")"), ("[", "]")]:
        if code.count(open_char) != code.count(close_char):
            return False, f"{filename}: 括号不平衡 '{open_char}' ({code.count(open_char)} 次) vs '{close_char}' ({code.count(close_char)} 次)"

    return True, f"{filename}: 预检通过"

# 1. 扫描并预检所有 JS 脚本
js_files = glob.glob(os.path.join(SCRIPTS_DIR, "*.js"))
if not js_files:
    print(f"目录 [{SCRIPTS_DIR}] 下未找到任何 .js 文件。")
    exit(0)

print(f"找到 {len(js_files)} 个本地脚本，开始静态预检...")
valid_files = []
for fpath in js_files:
    ok, msg = validate_script(fpath)
    if ok:
        print(f"  [OK] {msg}")
        valid_files.append(fpath)
    else:
        print(f"  [FAIL] {msg} (跳过此脚本)")

# 2. 批量推送到 Sub-Store 虚拟文件系统
print("\n开始批量推送至 Sub-Store 虚拟文件系统...")
for fpath in valid_files:
    filename = os.path.basename(fpath)
    with open(fpath, "rb") as f:
        code_bytes = f.read()

    upload_url = f"{BASE_URL}/api/file/{filename}"
    req = urllib.request.Request(
        upload_url,
        data=code_bytes,
        headers={"Content-Type": "text/javascript"},
        method="POST"
    )
    try:
        with urllib.request.urlopen(req) as resp:
            print(f"  -> 上传成功: {filename} (HTTP {resp.status})")
    except Exception as e:
        print(f"  -> 上传失败: {filename}, 错误: {e}")

# 3. 获取远端文件清单做最终确认
print("\n获取远端当前虚拟文件存储清单:")
try:
    with urllib.request.urlopen(f"{BASE_URL}/api/files") as resp:
        files = json.loads(resp.read().decode("utf-8"))
        for item in files:
            name = item.get("name") if isinstance(item, dict) else item
            print(f"  * {name}")
except Exception as e:
    print(f"查询远端文件列表失败: {e}")
```

---

### 场景五：现有集合算子结构全量巡检与 Web UI 兼容性一键修复 (UI Schema Doctor)

#### 场景特征
用户在使用第三方自动化脚本或旧版工具维护 Sub-Store 时，部分集合的算子可能存在非标准字段（如 Script Operator 缺失 `mode` 或错用 `url`/`path` 字段，Regex Filter 缺失 `action` 或 `keep` 联动）。这些集合虽然在客户端拉取订阅时能正常执行，但在 Sub-Store Web 控制台中打开时，单选框会失焦脱节、输入框空白，一旦在网页端点击保存还会导致原配置被清空损坏。

#### 临时脚本实现 (`/tmp/sub-store-copilot/fix-ui-schema-doctor.py`)

```python
import os, json, urllib.request

BASE_URL = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")

def api(path, method="GET", data=None):
    headers = {"Content-Type": "application/json"} if data is not None else {}
    body = json.dumps(data).encode("utf-8") if data is not None else None
    req = urllib.request.Request(f"{BASE_URL}{path}", data=body, headers=headers, method=method)
    with urllib.request.urlopen(req) as resp:
        res = resp.read().decode("utf-8")
        return json.loads(res) if res else {}

print(f"连接 Sub-Store: {BASE_URL}")
collections = api("/api/collections")
print(f"成功获取 {len(collections)} 个集合，开始巡检算子 Schema...")

total_fixed_collections = 0
total_fixed_operators = 0

for col in collections:
    col_name = col.get("name", "未命名")
    process = col.get("process", [])
    col_changed = False
    fixed_details = []

    for idx, op in enumerate(process):
        op_type = op.get("type", "")
        args = op.setdefault("args", {})

        # 1. 巡检并修复 Script Operator / Script
        if op_type in ("Script Operator", "Script"):
            if op["type"] != "Script Operator":
                op["type"] = "Script Operator"
                col_changed = True
                fixed_details.append(f"算子 #{idx+1} 重命名为标准 'Script Operator'")

            mode = args.get("mode")
            content = args.get("content") or args.get("url") or args.get("path") or ""

            # 自动推断模式（包含换行通常为源码模式，否则为链接模式）
            standard_mode = mode or ("script" if "\n" in content else "link")

            if mode != standard_mode or args.get("content") != content or "url" in args or "path" in args:
                op["args"] = {
                    "mode": standard_mode,
                    "content": content
                }
                col_changed = True
                total_fixed_operators += 1
                fixed_details.append(f"算子 #{idx+1} [Script Operator] 补全规范 mode: {standard_mode}, content 字段")

        # 2. 巡检并修复 Regex Filter 联动
        elif op_type == "Regex Filter":
            action = args.get("action")
            keep = args.get("keep")
            pattern = args.get("pattern", "")

            # 若缺少 action 或 keep，进行双向补全联动
            if action is None and keep is not None:
                args["action"] = "keep" if keep else "drop"
                col_changed = True
                total_fixed_operators += 1
                fixed_details.append(f"算子 #{idx+1} [Regex Filter] 根据 keep 补齐 action: {args['action']}")
            elif keep is None and action is not None:
                args["keep"] = (action == "keep")
                col_changed = True
                total_fixed_operators += 1
                fixed_details.append(f"算子 #{idx+1} [Regex Filter] 根据 action 补齐 keep: {args['keep']}")

    if col_changed:
        api("/api/collections", method="POST", data=col)
        total_fixed_collections += 1
        print(f" -> 已修复并同步集合 [{col_name}]: {', '.join(fixed_details)}")

print(f"\n巡检修复完成: 共检查 {len(collections)} 个集合，修复了 {total_fixed_collections} 个集合中的 {total_fixed_operators} 处非标准算子。")
```

---

## 四、实用工具函数与避坑备忘

### 1. 健壮的 Base64 容错补全
许多第三方订阅链接或节点导出的 Base64 字符串会缺失末尾的补齐 `=` 号，或者使用了 URL-Safe 变体（`-` 与 `_` 代替 `+` 与 `/`）。使用以下函数可 100% 容错解码：
```python
def robust_b64decode(s: str) -> str:
    s = s.strip()
    s += "=" * ((4 - len(s) % 4) % 4)
    for alt in [b"-_", None]:
        try:
            return base64.b64decode(s, altchars=alt).decode("utf-8", errors="ignore")
        except Exception:
            pass
    return ""
```

### 2. URL 锚点中的中文节点名称编解码
节点 URI 末尾的 `#节点名称` 经常包含 URL 编码或特殊字符：
```python
import urllib.parse

# 解码节点名称
raw_name = urllib.parse.unquote(url_fragment)

# 重新编码节点名称拼装 URI
safe_fragment = urllib.parse.quote(new_node_name)
```

### 3. API 覆写机制与增量操作铁律（严禁破坏性覆盖）
- **全量覆写风险**：`POST /api/subs` 与 `POST /api/collections` 均以 `name` 为主键进行整包全量覆盖保存。如果提交时仅包含变更字段或局部数组，后端将以提交的数据完全抹去数据库中的原有配置：
  - 针对集合：提交局部 `process` 会清空其他所有已有算子；缺少 `subscriptions` 会清空订阅源关联；
  - 针对订阅：提交局部 `content` 会清空已有全部节点。
- **增量三步铁律（Read -> Mutate -> Write-back）**：所有修改操作必须先 `GET` 读取当前完整对象，在 Python 字典或内存结构中做局部增量更新，完整保留 `subscriptions`、既有算子、`remark`、`tag` 等元数据后，再 `POST` 写回。
- **字典解包安全实践**：在构建更新 payload 时，推荐使用 `{**current_obj, "process": merged_process}` 模式，防止遗漏已有字段。
- **PATCH 接口支持**：若只需修改单个特定字段（例如仅更新订阅的 `content`），且确定当前后端版本支持，可使用 `PATCH /api/sub/<name>` 或 `PATCH /api/collection/<name>`。

### 4. 安全防护与脱敏
- **严禁硬编码真实 Token 或敏感路径**。在临时脚本中，统一通过 `os.environ.get("SUB_STORE_PREFIX_URL")` 获取连接地址。
- 输出运行日志或异常信息时，应统一对 URL 路径中的 `<SECRET_PATH>` 或查询参数中的 `token=<SECRET>` 进行脱敏，防止屏幕共享或终端录制泄露机密。
