---
name: sub-store-copilot
description: 交互式管理 Sub-Store 节点订阅与多源集合，编排算子流水线（过滤/重命名/国旗/排序/去重），现场定制编写与挂载 JavaScript 节点处理脚本，并导出纯净节点订阅或 Proxy Provider 链接。严格遵循 Sub-Store 前端 Web UI 双向绑定规范构造全量 Schema，彻底防止“后端执行生效但前端界面单选脱节/输入框空白/配置被 UI 覆写丢失”的问题。
---

# Sub-Store 节点管理、算子流水线与脚本定制指南

本 Skill 专为**交互带操（Copilot）**设计，面向已有运行中的 Sub-Store 实例，**纯粹基于 HTTP REST API 进行访问与操作，零环境侵入，不涉及 Docker、宿主机安装或外部环境变量配置**，协助用户完成**节点生命周期管理**、**多源聚合**、**附加算子流水线装配**与**现场编写 JavaScript 节点处理脚本**。

> [!NOTE]
> 本 Skill 专注于产出清洗、规范、优化后的纯净节点集或 Proxy Provider 订阅，不涉及生成包含完整 DNS 与复杂分流规则的客户端整套配置文档。
> **拒绝死板流程**：实际使用中多为临时单次修改。所有能力均为原子化设计，可随时独立单次触发，亦可按需自由组合。

---

## 一、核心原则与架构认知

### 1. 核心定位：纯粹基于 HTTP REST API 交互（零环境侵入）

- **面向已有实例**：本 Skill 纯粹通过标准 HTTP REST API（如 `/api/subs`、`/api/collections`、`/api/file`、`/api/settings`、`/api/sync/artifacts`）与运行中的 Sub-Store 实例通信，执行节点清洗、流水线编排与全局参数微调。
- **零环境侵入保证**：绝对不包含任何环境部署、Docker 安装、容器生命周期管理、系统守护进程配置或宿主机环境变量注入。
- **即时生效与平台中立**：所有配置与数据通过 HTTP 请求动态读取与热写回，无需重启服务或接触底层宿主机环境。

### 2. 两大运行形态与架构拓扑

Sub-Store 在实际网络中呈现为两种互斥的运行形态：

#### 形态一：独立服务器形态（Standalone Server）
Sub-Store 作为一个独立的 HTTP 进程或服务运行（例如在局域网服务器、VPS、本地独立服务端口如 `127.0.0.1:3000` 或指定域名），暴露标准 HTTP/HTTPS REST API 端点。
- **通信特征**：基于标准 TCP/IP 网络直接通信，无需外部代理客户端介入。
- **安全路径前缀（Prefix URL）**：Sub-Store 原生不包含多用户密码认证，在独立服务器形态中常通过环境变量 `SUB_STORE_FRONTEND_BACKEND_PATH` 配置不可预测的高熵随机路径前缀（如 `/<SECRET_PATH>`）。所有 API 与下载请求必须携带该路径前缀（如 `https://sub.example.com/<SECRET_PATH>/api/...`），未携带或路径不符者直接返回 404 或静态伪装页，从而阻断公网扫描与未授权访问。

#### 形态二：代理 MITM 下的脚本劫持形态（Proxy MITM Script Hijack）
Sub-Store 托管在规则分流代理客户端（如 Surge、Loon、Quantumult X、Shadowrocket 等）内置的 JavaScript 脚本引擎中，通过客户端的中间人解密（MITM）和 URL 重写/脚本规则劫持特定虚拟域名（如 `http://sub.store`）。
- **通信特征**：系统中不存在监听物理端口的独立服务器进程。所有发往虚拟域名的 HTTP 请求**必须被代理客户端捕获并交由内部脚本引擎拦截处理**。
- **流量注入约束**：若在未开启代理接管的环境下直接请求 `sub.store`，系统网络栈会向公网 DNS 发起查询并必然收到 `NXDOMAIN` 失败，同时造成 DNS 泄漏。因此，管理调用必须确保经由代理客户端（如系统 TUN/增强模式接管，或显式配置代理端口 `http_proxy`）。若通过 HTTPS 访问，还需允许客户端生成的自签名证书。

```text
======================= 形态一：独立服务器形态 (Standalone Server) =======================

+------------------------------+     HTTP / HTTPS REST API     +------------------------------+
| 管理端 (curl / Python /      | ============================> | Sub-Store 独立服务器         |
| Copilot / 客户端拉取)        | <============================ | (独立 HTTP 服务 / 反代端点) |
+------------------------------+                               +------------------------------+
                                                                              |
                                                                              | 直连拉取
                                                                              v
                                                               +------------------------------+
                                                               | 外部机场与自建节点上游源     |
                                                               +------------------------------+

================= 形态二：代理 MITM 下的脚本劫持形态 (Proxy MITM Script Hijack) =================

+------------------------------+
| 管理端 (curl / Python /      |
| Copilot / 浏览器)            |
+------------------------------+
               |
               | 1. 发起请求: http://sub.store/api/... (必须经由代理客户端捕获接管)
               v
+--------------------------------------------------------------+
| 代理客户端 (Surge / Loon / Quantumult X / Shadowrocket)      |
|                                                              |
|   +--------------------------------------------------------+ |
|   | 流量接管层 (TUN 增强模式 / 操作系统代理 / 本地代理端口) | |
|   +--------------------------------------------------------+ |
|                                |                             |
|                                | 2. 匹配虚拟域名 sub.store   |
|                                v                             |
|   +--------------------------------------------------------+ |
|   | 内置 MITM 拦截与 JS 脚本引擎 (执行 Sub-Store 模块)     | |
|   +--------------------------------------------------------+ |
+--------------------------------------------------------------+
               |
               | 3. 直连或代理分流拉取
               v
+------------------------------+
| 外部机场与自建节点上游源     |
+------------------------------+
```

### 3. 核心铁律：前端 Web UI 表单严格兼容原则（双向绑定全量 Schema）

Sub-Store 存在一个经典陷阱：**后端执行引擎对入参高度宽容，但前端 Web 控制台组件严格依赖特定的响应式表单字段**。
- **核心陷阱分析**：若仅满足后端调用，传入简化或模糊字段（例如 Script Operator 传 `url`/`path` 而非 `mode` + `content`，或 Regex Filter 缺少 `action` 与 `keep` 的双向联动），后端在下载和执行节点清洗时能够向下兼容生效；但在用户访问 Web 控制台打开该集合时，前端 Vue 表单因缺少关键驱动字段，单选框会失焦脱节、输入框呈现空白。当用户后续在网页上编辑并点击保存时，前端会将这组空白表单回传并覆写后端，导致配置损坏或丢失！
- **全局原则（不只是 Script Operator）**：此问题贯穿于 Sub-Store 的所有核心对象与算子，包括 Script Operator（必须包含 `mode: "link"|"script"` 与 `content`）、Regex Filter（必须联动补齐 `action: "keep"|"drop"` 与 `keep: true|false`）、Rename、Flag、Sort、Deduplicate 以及 Manual/Remote 订阅源。编写任何 API 请求与脚本时，必须一律采用与前端 UI 严格对齐的全量字段格式，确保“**后端执行 100% 生效 + 前端 Web UI 完美可视与安全交互编辑**”。

### 4. 核心铁律：增量修改原则（严禁破坏性覆盖）

在 Sub-Store 的实际运维中，盲目“全量覆盖”是最危险的高危操作：
- **追加节点危险**：若用户要求“加一个节点”，若直接构造新对象 POST 会把原有订阅里的其他节点全部清空抹去；
- **微调算子危险**：若用户要求“调整某个算子参数”或“加个脚本算子”，若直接提交包含单个算子的 `process` 数组，会把该集合已有的 Quick Setting、Flag、Regex Filter、Sort 等其他所有既有算子全部冲掉丢失；
- **微调设置危险**：若直接向 `/api/settings` POST 仅包含少量字段的 JSON，会导致未提交的 `gistToken` 备份凭证、出站代理 `proxy` 或超时设置等被冲掉丢失或重置；
- **丢失元数据**：若未保留集合或订阅原有的 `subscriptions`、`remark`、`tag`、`ignoreFailedRemoteSub` 等元数据，全量覆写会导致多源关联与高级设置被清空。

**增量修改标准三步法（3-Step Incremental Pattern）**：
无论是节点维护、算子调整还是全局配置微调，必须严格遵循标准三步法：
1. **Read (先读/先拉取)**：通过 `GET` 接口（`/api/sub/...`、`/api/collection/...` 或 `/api/settings`）拉取目标的当前全量 JSON 对象；
2. **Mutate (增量改/内存合并)**：在内存或管道流中进行局部针对性操作（如向 `content` 追加新节点行、向 `process` 数组追加新算子、更新特定算子 `args`、或修改特定的 settings 键值），严格保留其他所有属性、既有算子与凭证；
3. **Write-back (完整回写)**：将合并后的完整对象通过 `POST` 写回保存（或在支持的端点使用 `PATCH`）。

---

## 二、步骤一：识别运行形态与端点探测

在对 Sub-Store 进行任何操作前，首先需要探测并确认当前 Sub-Store 的运行形态与可用的基准端点。

### 1. 端点识别判定逻辑
- **独立服务器判定**：
  - 若已配置环境变量 `SUB_STORE_PREFIX_URL`，直接使用该 URL（支持携带 `/<SECRET_PATH>` 安全前缀）。
  - 若未显式配置环境变量，优先尝试探测本地回环无前缀独立服务（`http://127.0.0.1:3000`）。
- **代理 MITM 脚本劫持判定**：
  - 尝试直连或通过代理端口请求虚拟域名 `http://sub.store/api/subs` 或 `https://sub.store/api/subs`。
  - 若处于代理接管环境（如 TUN 增强模式或本地代理端口转发），探测成功后将基准端点绑定至 `http://sub.store`。

### 2. 自动化端点探测脚本
执行以下探测脚本快速锁定当前可用端点与通信参数：

```bash
if [ -n "$SUB_STORE_PREFIX_URL" ]; then
  SUB_STORE_URL="${SUB_STORE_PREFIX_URL%/}"
  echo "使用指定的独立服务器端点 (Prefix URL): <SECRET_PATH>"
elif curl -s --max-time 1.5 "http://127.0.0.1:3000/api/subs" > /dev/null 2>&1; then
  SUB_STORE_URL="http://127.0.0.1:3000"
  echo "已探测到本地独立服务器: $SUB_STORE_URL"
elif curl -s --max-time 1.5 "http://sub.store/api/subs" > /dev/null 2>&1; then
  SUB_STORE_URL="http://sub.store"
  echo "已探测到代理客户端 MITM 虚拟域名 (直连接管 HTTP): $SUB_STORE_URL"
elif curl -k -s --max-time 1.5 "https://sub.store/api/subs" > /dev/null 2>&1; then
  SUB_STORE_URL="https://sub.store"
  echo "已探测到代理客户端 MITM 虚拟域名 (直连接管 HTTPS): $SUB_STORE_URL"
elif curl -x "http://127.0.0.1:6152" -s --max-time 1.5 "http://sub.store/api/subs" > /dev/null 2>&1; then
  SUB_STORE_URL="http://sub.store"
  export http_proxy="http://127.0.0.1:6152"
  export https_proxy="http://127.0.0.1:6152"
  echo "已通过本地代理端口 (6152) 接入代理客户端 MITM 虚拟域名: $SUB_STORE_URL"
elif curl -x "http://127.0.0.1:7890" -s --max-time 1.5 "http://sub.store/api/subs" > /dev/null 2>&1; then
  SUB_STORE_URL="http://sub.store"
  export http_proxy="http://127.0.0.1:7890"
  export https_proxy="http://127.0.0.1:7890"
  echo "已通过本地代理端口 (7890) 接入代理客户端 MITM 虚拟域名: $SUB_STORE_URL"
else
  echo "未探测到可用 Sub-Store 端点。请根据运行形态处理："
  echo "1. 独立服务器：请执行 export SUB_STORE_PREFIX_URL=\"https://domain.com/<SECRET_PATH>\""
  echo "2. 代理 MITM 劫持：请确保代理客户端（Surge/Loon 等）已启动且当前环境流量已交由代理捕获"
fi
```

---

## 三、原子操作工具箱（Atomic Actions）

所有功能均为独立原子能力，随时针对具体需求单次调用。

### 1. 节点与订阅管理（增/删/改/查）

随时对单条自建节点或机场订阅进行录入、查询与增量微调：

- **查询全部现有订阅与节点**：
  ```bash
  curl -s "${SUB_STORE_URL}/api/subs" | jq .
  ```
- **首次录入自建节点订阅（Manual 本地模式）**：
  节点链接（vmess, vless, ss, trojan, hysteria2 等）以换行符分隔放入 `content`，`url` 留空：
  ```bash
  curl -s -X POST "${SUB_STORE_URL}/api/subs" \
    -H "Content-Type: application/json" \
    -d '{
      "name": "My-Manual-Nodes",
      "type": "manual",
      "source": "local",
      "url": "",
      "enabled": true,
      "content": "trojan://password@1.1.1.1:443#%E6%97%A5%E6%9C%AC-01\nhysteria2://user:pass@3.3.3.3:443/?sni=example.com#%E7%BE%8E%E5%9B%BD-01"
    }'
  ```
- **增量追加自建节点（严禁直接覆盖清空原有节点）**：
  追加新节点链接时，必须先读取已有 `content` 并使用换行符 `\n` 拼接在末尾，再写回保存：
  ```bash
  SUB_NAME="My-Manual-Nodes"
  NEW_NODE="hysteria2://user:pass@4.4.4.4:443/?sni=example.com#%E6%96%B0%E8%8A%82%E7%82%B9-02"

  curl -s "${SUB_STORE_URL}/api/sub/${SUB_NAME}" | jq --arg n "$NEW_NODE" '
    .content = (if .content and (.content | length > 0) then (.content + "\n" + $n) else $n end)
  ' | curl -s -X POST "${SUB_STORE_URL}/api/subs" \
    -H "Content-Type: application/json" \
    --data-binary @-
  ```
- **接入/更新远程机场订阅（Remote URL 模式）**：
  ```bash
  curl -s -X POST "${SUB_STORE_URL}/api/subs" \
    -H "Content-Type: application/json" \
    -d '{
      "name": "Airport-Main",
      "type": "subscription",
      "source": "remote",
      "url": "https://airport.example.com/sub?token=<SECRET>",
      "enabled": true
    }'
  ```

### 2. 现场编写 JS 脚本并一键推送

针对特定清洗、字段覆写或属性修补需求，现场手写标准 JavaScript 脚本并推送至虚拟文件存储。

> [!IMPORTANT]
> **临时脚本存放与能力命名规范**：
> - **统一存储目录**：所有现场临时编写的 JavaScript 处理脚本（以及临时 Python 辅助脚本）**统一存放于 `/tmp/sub-store-copilot/` 目录下**，严禁在主工作区随处新建文件，保持项目环境整洁。
> - **文件命名格式**：`/tmp/sub-store-copilot/<capability-name>.js`。
> - **自解释命名原则**：`<capability-name>` **必须具备清晰的自解释语义，准确表明脚本的具体能力**（例如 `optimize-udp-tfo.js`、`clean-ads-expire.js`、`inject-singbox-tags.js`、`rename-region-flags.js` 等），严禁使用 `test.js`、`temp.js`、`1.js` 等无意义命名。
> - **标准工作流程**：在本地 `/tmp/sub-store-copilot/<capability-name>.js` 编写并校验 -> 使用 `curl --data-binary` 二进制上传至 Sub-Store 虚拟存储池 `/api/file/<capability-name>.js` -> 在算子流水线中引用。

- **脚本函数规范**：必须导出 `operator(proxies, targetPlatform, context)` 函数：
  ```javascript
  function operator(proxies = [], targetPlatform) {
    return proxies.map(p => {
      p.udp = true;
      p.tfo = true;
      if (p.port === 443 || p.tls === true) p["skip-cert-verify"] = true;
      if (p.type === "hysteria2") p.alpn = p.alpn || ["h3"];
      return p;
    });
  }
  ```
- **标准工作流：本地编写并一键推送**（推荐模式，工作区零污染）：
  ```bash
  # 1. 创建规范临时目录并写入自解释命名的处理脚本
  mkdir -p /tmp/sub-store-copilot
  cat << 'EOF' > /tmp/sub-store-copilot/optimize-udp-tfo.js
  function operator(proxies = [], targetPlatform) {
    return proxies.map(p => {
      p.udp = true;
      p.tfo = true;
      if (p.port === 443 || p.tls === true) p["skip-cert-verify"] = true;
      if (p.type === "hysteria2") p.alpn = p.alpn || ["h3"];
      return p;
    });
  }
  EOF

  # 2. 二进制上传推送至虚拟文件存储
  curl -s -X POST "${SUB_STORE_URL}/api/file/optimize-udp-tfo.js" \
    -H "Content-Type: text/javascript" \
    --data-binary @/tmp/sub-store-copilot/optimize-udp-tfo.js
  ```
- **快捷管道流上传**（无需中间文件，单行即推）：
  ```bash
  curl -s -X POST "${SUB_STORE_URL}/api/file/optimize-udp-tfo.js" \
    -H "Content-Type: text/javascript" \
    --data-binary @- << 'EOF'
  function operator(proxies = [], targetPlatform) {
    return proxies.map(p => {
      p.udp = true;
      p.tfo = true;
      return p;
    });
  }
  EOF
  ```

### 3. 算子流水线调整与参数微调

创建聚合集合或对现有流水线进行增量调整（Script Operator, Regex Filter, Rename, Flag, Sort, Deduplicate）：

> [!CAUTION]
> **严禁盲目提交单算子数组**：对现有集合追加或调整算子时，若直接 POST 仅包含单个新算子的 `process` 数组，原有集合中配置的所有其他算子（Quick Setting、Flag、Regex、Sort 等）和订阅关联将被彻底冲掉！必须在既有 `process` 上增量追加或精准 patch。

- **初次创建多源聚合集合与初始流水线**：
  ```bash
  curl -s -X POST "${SUB_STORE_URL}/api/collections" \
    -H "Content-Type: application/json" \
    -d '{
      "name": "Clean-Nodes",
      "type": "collection",
      "subscriptions": ["Airport-Main", "My-Manual-Nodes"],
      "process": [
        { "type": "Script Operator", "args": { "mode": "link", "content": "/api/file/optimize-udp-tfo.js", "arguments": {} } },
        { "type": "Regex Filter", "args": { "pattern": "(?i)官网|到期|剩余|通知", "action": "drop", "keep": false } },
        { "type": "Flag", "args": { "mode": "add" } },
        { "type": "Sort", "args": { "field": "name", "order": "asc" } },
        { "type": "Deduplicate", "args": { "field": "name" } }
      ]
    }'
  ```

- **增量追加算子至现有集合（安全保留既有算子与订阅源元数据）**：
  先读现有配置，向 `process` 数组末尾追加新算子并回写：
  ```bash
  COLL_NAME="Clean-Nodes"
  NEW_OP='{"type": "Script Operator", "args": {"mode": "link", "content": "/api/file/optimize-udp-tfo.js", "arguments": {}}}'

  curl -s "${SUB_STORE_URL}/api/collection/${COLL_NAME}" | jq --argjson op "$NEW_OP" '
    .process = (.process // []) + [$op]
  ' | curl -s -X POST "${SUB_STORE_URL}/api/collections" \
    -H "Content-Type: application/json" \
    --data-binary @-
  ```

### 4. 节点导出与清洗预览

随时拉取指定订阅或集合，快速查看清洗后效果或直接作为客户端 Proxy Provider：

```bash
# 1. 预览 Mihomo (Clash.Meta) YAML 纯节点列表（适合 proxy-provider 对接）
curl -s "${SUB_STORE_URL}/download/collection/Clean-Nodes?target=clash-meta" | head -n 30

# 2. 预览 Sing-Box JSON outbounds
curl -s "${SUB_STORE_URL}/download/collection/Clean-Nodes?target=singbox" | head -n 30

# 3. 预览 Surge [Proxy] 节点清单
curl -s "${SUB_STORE_URL}/download/collection/Clean-Nodes?target=surge" | head -n 30

# 4. 获取通用原始 URI / Base64 列表
curl -s "${SUB_STORE_URL}/download/collection/Clean-Nodes"
```

外部客户端作为 Proxy Provider 使用范例：
```yaml
proxy-providers:
  Clean-Nodes-Provider:
    type: http
    url: "${SUB_STORE_URL}/download/collection/Clean-Nodes?target=clash-meta"
    path: ./providers/clean-nodes.yaml
    interval: 3600
    health-check:
      enable: true
      url: http://www.gstatic.com/generate_204
      interval: 300
```

### 5. 全局配置管理与 Gist 备份同步（Settings & Sync）

通过纯 HTTP REST 接口即时读取或微调运行中实例的全局请求配置、缓存参数与平台偏好，并按需触发 Gist 备份同步：

- **读取当前全量设置**：
  ```bash
  curl -s "${SUB_STORE_URL}/api/settings" | jq .
  ```
- **增量更新请求配置与出站代理（严格保留既有 Gist 凭证）**：
  先读取配置，在内存中修改目标字段后回写，避免直接覆写导致敏感项丢失：
  ```bash
  curl -s "${SUB_STORE_URL}/api/settings" | jq '
    .proxy = "http://127.0.0.1:7890" |
    .timeout = 15000 |
    .concurrency = 5 |
    .concurrencyWaitTime = 200
  ' | curl -s -X POST "${SUB_STORE_URL}/api/settings" \
    -H "Content-Type: application/json" \
    --data-binary @-
  ```
- **curl 一键触发 Gist 备份同步**：
  ```bash
  # 调用备份端点立即触发全量配置上传（支持 GET 或 POST）
  curl -s -X POST "${SUB_STORE_URL}/api/sync/artifacts"
  ```
- **带状态检查的安全触发命令**：
  ```bash
  mkdir -p /tmp/sub-store-copilot
  HTTP_CODE=$(curl -s -o /tmp/sub-store-copilot/sync.log -w "%{http_code}" --max-time 15 "${SUB_STORE_URL}/api/sync/artifacts")
  if [ "$HTTP_CODE" = "200" ]; then
    echo "Gist 备份同步成功"
  else
    echo "Gist 备份同步失败 (HTTP $HTTP_CODE):"
    cat /tmp/sub-store-copilot/sync.log
  fi
  ```
- **极简 Python 触发脚本（零依赖，用完即走）**：
  ```python
  import os, urllib.request

  url = f"{os.environ.get('SUB_STORE_PREFIX_URL', 'http://127.0.0.1:3000').rstrip('/')}/api/sync/artifacts"
  try:
      with urllib.request.urlopen(urllib.request.Request(url, method="POST"), timeout=15) as r:
          print(f"备份同步完成 [{r.status}]")
  except Exception as e:
      print(f"备份同步异常: {e}")
  ```
- **参数检查与前置要求**：
  - **前缀路径检查**：独立服务器配置了安全前缀时，请求路径必须带有 `/<SECRET_PATH>/api/sync/artifacts`；代理 MITM 模式下需经代理客户端转发捕获。
  - **GitHub Token 作用域**：Token 必须具备 `gist` 读写权限（示例中使用 `<GITHUB_TOKEN>` 脱敏占位符，严禁泄露真实值）。
  - **Gist 关联检查**：确保 Web 控制台 Settings 中已正确配置目标 `<GIST_ID>` 与用户名。

---

## 四、常见组合场景（Recipes，仅供参考，绝非强制）

### 场景 1：临时改一次 —— 微调现有订阅的正则过滤或改名规则
用户仅需调整关键词或替换前缀，无需触动整个订阅体系：
1. **遵循增量三步法（先读 -> 增量改 -> 完整回写）**：
   定位集合中现有的 `Regex Filter` 算子并更新 `args.pattern`，严格保留其他所有既有算子与 `subscriptions` 引用：
   ```bash
   COLL_NAME="Clean-Nodes"
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
2. 立即生效，集合中原有的 Script Operator、Flag、Sort 等算子及所有订阅源关联丝毫不受影响。

### 场景 2：临时写一次 —— 现场写个专有 JS 修复特定节点属性并生效
遇到特殊节点配置缺陷（如特定节点缺少 ALPN、SNI 不匹配或需要指定指纹）：
1. 现场编写针对性 JS 修复脚本，写入规范临时目录并上传至虚拟存储：
   ```bash
   mkdir -p /tmp/sub-store-copilot
   cat << 'EOF' > /tmp/sub-store-copilot/fix-sni-domain.js
   function operator(proxies = []) {
     return proxies.map(p => {
       if (p.name && p.name.includes("特殊")) p.sni = "custom.sni.com";
       return p;
     });
   }
   EOF
   curl -s -X POST "${SUB_STORE_URL}/api/file/fix-sni-domain.js" \
     -H "Content-Type: text/javascript" \
     --data-binary @/tmp/sub-store-copilot/fix-sni-domain.js
   ```
2. **增量前置插入算子（绝不冲掉既有流水线）**：
   读取现有集合，将 `Script Operator` 算子插入至 `process` 数组前端，回写更新：
   ```bash
   NEW_OP='{"type": "Script Operator", "args": {"mode": "link", "content": "/api/file/fix-sni-domain.js", "arguments": {}}}'
   curl -s "${SUB_STORE_URL}/api/collection/Clean-Nodes" | jq --argjson op "$NEW_OP" '
     .process = [$op] + (.process // [])
   ' | curl -s -X POST "${SUB_STORE_URL}/api/collections" \
     -H "Content-Type: application/json" \
     --data-binary @-
   ```
3. 调用 `${SUB_STORE_URL}/download/collection/Clean-Nodes?target=clash-meta` 即时校验修补效果。

### 场景 3：临时补一次 —— 向现有订阅追加新自建节点
用户临时新搭了一台 VPS 或临时拿到两个节点链接需要塞进现有订阅：
1. **增量追加原则**：严禁直接 POST 仅包含新节点的对象（否则已有节点全灭）。
2. 使用增量管道流：读取现有配置 -> 将新节点行拼接在 `content` 末尾 -> 回写保存：
   ```bash
   SUB_NAME="My-Manual-Nodes"
   NEW_NODE="hysteria2://user:pass@4.4.4.4:443/?sni=new.vps.com#%E6%96%B0%E8%8A%82%E7%82%B9"

  curl -s "${SUB_STORE_URL}/api/sub/${SUB_NAME}" | jq --arg n "$NEW_NODE" '
    .content = (if .content and (.content | length > 0) then (.content + "\n" + $n) else $n end)
  ' | curl -s -X POST "${SUB_STORE_URL}/api/subs" \
    -H "Content-Type: application/json" \
    --data-binary @-
   ```
3. 下游所有引用该订阅的集合自动同步更新，现有流水线与既有节点完整保留。

### 场景 4：典型完整组合 —— 从零录入节点、配置算子与聚合
从零初始化时的完整组合参考：
1. 运行端点探测脚本确认通信环境（独立服务器或代理 MITM 虚拟域名）。
2. 分别录入自建 Manual 节点与机场 Remote 订阅。
3. 推送网络优化脚本，配置正则过滤 + 国旗 + 排序算子流聚合为新集合。
4. 交付 `/download/collection/<name>?target=clash-meta` 供客户端 Proxy Provider 引用。

### 场景 5：临时配一次 —— HTTP 动态微调出站代理、请求超时与并发控制
遇到外部机场拉取缓慢、需要设置中继代理或防止并发过高被上游限流：
1. **遵循增量修改三步法（先读 -> 增量改 -> 完整回写）**：
   通过 `GET /api/settings` 读取现有设置，在内存中精准更新 `proxy`、`timeout`、`concurrency` 与 `concurrencyWaitTime`，严格保留已有的 `gistToken` 等关键凭证：
   ```bash
   curl -s "${SUB_STORE_URL}/api/settings" | jq '
     .proxy = "http://127.0.0.1:7890" |
     .timeout = 15000 |
     .concurrency = 5 |
     .concurrencyWaitTime = 200
   ' | curl -s -X POST "${SUB_STORE_URL}/api/settings" \
     -H "Content-Type: application/json" \
     --data-binary @-
   ```
2. 立即热生效，Sub-Store 后续拉取外部机场订阅与外部脚本时自动中继本地代理并以 5 并发平滑请求。无需重启任何服务。

---

## 五、复杂操作：按需编写临时 Python 脚本（用完即走）

在管理 Sub-Store 时，坚决**拒绝维护笨重僵化的永久全功能 CLI**，而是根据操作复杂度灵活切换工具：
- **简单单次操作**：优先使用 `curl` + `jq`（最轻量、无外部切换，单行即可完成查询、单订阅添加或文件上传）。
- **复杂批量与深度转换**：当涉及多源脏数据解析（如 Base64 嵌套提取与指纹去重）、多集合批量注入算子流水线、节点增量 Diff 比对或本地 JS 脚本静态预检推送时，**现场手写轻量、临时、用完即走的 Python 脚本**。

> [!TIP]
> **临时脚本规范与零依赖原则**：
> - **统一存储目录**：所有现场编写的临时 Python 脚本与 JS 处理脚本统一存放在 `/tmp/sub-store-copilot/` 目录下，严禁在主工作区随处新建文件。
> - **自解释命名格式**：文件命名为 `/tmp/sub-store-copilot/<capability-name>.py`，`<capability-name>` 必须准确表明脚本具体能力（如 `clean-and-dedup-nodes.py`、`batch-inject-pipeline.py`、`diff-sync-nodes.py` 等），严禁 `test.py`、`temp.py` 等模糊命名。
> - **零依赖保证**：临时脚本仅使用 Python 3 标准库（`urllib.request`, `json`, `os`, `re`, `base64` 等），无须 `pip install`，在 `/tmp/sub-store-copilot/` 随写随跑，跑完即抛。

### 10 行极简通信模板（开箱即用）
```python
import os, json, urllib.request

BASE_URL = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")

def api(path, method="GET", data=None):
    headers = {"Content-Type": "application/json"} if data is not None else {}
    body = json.dumps(data).encode("utf-8") if data is not None else None
    req = urllib.request.Request(f"{BASE_URL}{path}", data=body, headers=headers, method=method)
    with urllib.request.urlopen(req) as resp:
        res = resp.read().decode("utf-8")
        return json.loads(res) if res and resp.headers.get_content_type() == "application/json" else res
```

更多五大高频场景（多源清洗去重、批量流水线注入、增量 Diff 对比、本地 JS 语法预检批量上传、Web UI 表单失联全量巡检修复）的完整实战范式，请参阅：
- [复杂操作临时 Python 脚本编写指南](references/complex-python-ops.md)

---

## 六、核心对象与常用算子速查

### 1. 核心对象结构速查

| 对象类型 | 端点 | 核心必需字段 | 关键说明 |
|---|---|---|---|
| **自建节点** | `/api/subs` | `name`, `type: "manual"`, `source: "local"`, `content` | `url` 留空，节点全量写在 `content` 换行分隔 |
| **远程订阅** | `/api/subs` | `name`, `type: "subscription"`, `source: "remote"`, `url` | `content` 留空，填入机场 HTTP 订阅链接 |
| **聚合集合** | `/api/collections` | `name`, `type: "collection"`, `subscriptions`, `process` | 聚合多个订阅，按序链式执行 `process` 算子流水线 |
| **处理脚本** | `/api/file/:name` | 原始 JavaScript 文本 | 现场定制清洗，在算子流中通过 `Script Operator` 算子挂载 |

### 2. 常用算子（Operators）快速拼装

算子 `type` 严格区分大小写，声明在 `process` 数组中自上而下链式执行：

```json
// 1. 挂载执行自定义 JS 脚本（Remote Link 模式，UI 与后端完全兼容）
{ "type": "Script Operator", "args": { "mode": "link", "content": "/api/file/optimize-udp-tfo.js", "arguments": {} } }

// 2. 正则保留特定地区（如港日新美）
{ "type": "Regex Filter", "args": { "pattern": "(?i)港|hk|日|jp|新|sg|美|us", "action": "keep", "keep": true } }

// 3. 正则剔除垃圾与广告节点
{ "type": "Regex Filter", "args": { "pattern": "(?i)官网|到期|剩余|通知|重置", "action": "drop", "keep": false } }

// 4. 正则替换重命名（统一加前缀）
{ "type": "Rename", "args": { "pattern": "^(.*)$", "replace": "[专线] $1" } }

// 5. 自动补全国旗 Emoji
{ "type": "Flag", "args": { "mode": "add" } }

// 6. 节点按名称升序排序
{ "type": "Sort", "args": { "field": "name", "order": "asc" } }

// 7. 按节点名称去重
{ "type": "Deduplicate", "args": { "field": "name" } }
```

### 3. 增量操作避坑警示（Prevent Destructive Overwrites）

| 运维操作场景 | 破坏性覆盖高危做法（严禁） | 增量式安全操作规范（标准） |
|---|---|---|
| **追加新自建节点** | 构造仅包含新节点的 JSON POST 提交，导致**已有节点全部丢失** | 先 GET 现有订阅，在现有 `content` 末尾换行追加新链接后再回写 |
| **新增处理算子** | 提交仅含单个新算子的 `process` 数组，导致**已有所有算子被冲掉** | 先 GET 现有集合，向 `process` 数组 append/prepend 该算子后再回写 |
| **调整算子参数** | 重构新集合对象覆写，导致 `subscriptions` 关联与高级选项被清空 | 先 GET 现有集合，遍历定位目标算子修改对应 `args`，保留所有其他属性回写 |
| **修改全局配置 (/api/settings)** | 提交仅包含待修改字段的 JSON POST，导致 **Gist Token、出站代理等既有配置被冲掉清空** | 严格遵循先拉取后更新：先 GET 拉取全量 settings，在内存或 jq 中增量合并修改目标字段后再完整回写 |
| **临时脚本编写** | 随手在根目录写 `test.js`/`temp.py` 污染工作区 | 统一在 `/tmp/sub-store-copilot/` 编写自解释命名文件，用完即走 |

---

## 七、常见问题与 UI 修复：一键修正已有的 Script Operator 与算子配置

当之前通过第三方脚本或非标准 API 创建了集合，常会出现**“节点清洗与客户端拉取完全正常，但在 Web 控制台打开集合时，Script Operator 的 Remote Link 单选框未勾选、输入框空白，或者正则过滤器单选脱节”**的现象。若在 Web 界面误点保存，会导致后端配置被空表单覆写丢失。

可使用以下方案进行一键排查与全量无损修复：

### 1. 单集合即时修复（curl + jq 管道流）

读取指定集合配置，将所有非标准算子字段自动映射为前端 Web UI 严格绑定的全量格式，并直接覆写生效：

```bash
# 1. 设定目标集合名称并执行一键转换覆写
COLL_NAME="Clean-Nodes"
curl -s "${SUB_STORE_URL}/api/collection/${COLL_NAME}" | jq '
  .process = (.process | map(
    if .type == "Script Operator" or .type == "Script" then
      .type = "Script Operator" |
      .args = {
        mode: (.args.mode // (if .args.content and (.args.content | contains("\n")) then "script" else "link" end)),
        content: (.args.content // .args.url // .args.path // "")
      }
    elif .type == "Regex Filter" then
      .args.action = (.args.action // (if .args.keep == false then "drop" else "keep" end)) |
      .args.keep = (if .args.keep != null then .args.keep elif .args.action == "drop" then false else true end)
    else . end
  ))
' | curl -s -X POST "${SUB_STORE_URL}/api/collections" \
  -H "Content-Type: application/json" \
  --data-binary @-

# 2. 验证修复后的 process 结构
curl -s "${SUB_STORE_URL}/api/collection/${COLL_NAME}" | jq .process
```

### 2. 单集合精准覆盖修复（curl 直接更新）

若已知集合需要挂载的脚本与规则，可直接发送全量标准 JSON 更新：

```bash
curl -s -X POST "${SUB_STORE_URL}/api/collections" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Clean-Nodes",
    "type": "collection",
    "subscriptions": ["Airport-Main", "My-Manual-Nodes"],
    "process": [
      {
        "type": "Script Operator",
        "args": {
          "mode": "link",
          "content": "/api/file/optimize-udp-tfo.js",
          "arguments": {}
        }
      },
      {
        "type": "Regex Filter",
        "args": {
          "pattern": "(?i)官网|到期|剩余|通知",
          "action": "drop",
          "keep": false
        }
      }
    ]
  }'
```

### 3. 全量集合一键巡检与修复（临时 Python 脚本，用完即走）

若管理多个集合，可在 `/tmp/sub-store-copilot/fix-ui-schema-doctor.py` 现场手写并执行以下极简修复脚本，自动扫描全部集合并批量纠正所有不兼容的算子格式：

```python
import os, json, urllib.request

BASE_URL = os.environ.get("SUB_STORE_PREFIX_URL", "http://127.0.0.1:3000").rstrip("/")

def api(path, data=None):
    headers = {"Content-Type": "application/json"} if data else {}
    body = json.dumps(data).encode("utf-8") if data else None
    req = urllib.request.Request(f"{BASE_URL}{path}", data=body, headers=headers, method="POST" if data else "GET")
    with urllib.request.urlopen(req) as resp:
        res = resp.read().decode("utf-8")
        return json.loads(res) if res else {}

collections = api("/api/collections")
print(f"开始巡检 {len(collections)} 个集合...")

for col in collections:
    name = col.get("name")
    process = col.get("process", [])
    changed = False

    for op in process:
        op_type = op.get("type", "")
        args = op.setdefault("args", {})
        
        # 1. 修复 Script Operator / Script 算子
        if op_type in ("Script Operator", "Script"):
            op["type"] = "Script Operator"
            if "mode" not in args or "content" not in args:
                content = args.get("content") or args.get("url") or args.get("path") or ""
                mode = args.get("mode") or ("script" if "\n" in content else "link")
                op["args"] = {"mode": mode, "content": content}
                changed = True
                
        # 2. 修复 Regex Filter 算子联动
        elif op_type == "Regex Filter":
            action = args.get("action")
            keep = args.get("keep")
            if action is None and keep is not None:
                args["action"] = "keep" if keep else "drop"
                changed = True
            elif keep is None and action is not None:
                args["keep"] = (action == "keep")
                changed = True

    if changed:
        api("/api/collections", data=col)
        print(f"已修复集合 [{name}] 中的非标准算子并写回 Sub-Store！")

print("全量集合巡检与修复完成。")
```

---

## 八、完整技术手册索引

如需查阅特定功能模块的深入规范与字典，请参考以下独立手册：

### 运行形态与网络架构
- [运行形态与服务器探测指南](references/mode.md)：独立服务器与代理 MITM 脚本劫持两大运行形态概览、端点探测优先级与自动化探测脚本。

### API 与算子数据字典
- [订阅管理接口规范](references/api-subs.md)：`/api/subs` 接口规范，自建 Manual 节点内联与远程 Remote 机场订阅字段与请求体。
- [聚合集合接口规范](references/api-collections.md)：`/api/collections` 接口规范，多源聚合与装配算子流水线配置。
- [内置算子参数规范字典](references/api-operators.md)：Regex Filter、Rename、Flag、Sort、Deduplicate、Script Operator 算子完整参数规范与链式执行原理。
- [虚拟文件存储接口规范](references/api-files.md)：`/api/file/:name` 与 `/api/files` 接口规范，JS 脚本管理与 curl 管道上传。
- [节点导出与下载接口规范](references/api-download.md)：`/download/...` 端点规范、target 参数格式与外部客户端 Proxy Provider 对接范例。
- [全局设置与 HTTP 配置规范](references/api-settings.md)：对齐前端 Web 控制台四大配置模块（Request Configuration / Cache Configuration / Gist Configuration / Other Configuration），通过 `GET /api/settings` 与 `POST /api/settings` 进行纯 HTTP 读写，以及通过 `POST /api/sync/artifacts` 触发备份同步。

### 复杂批量操作与临时脚本
- [复杂操作临时 Python 脚本编写指南](references/complex-python-ops.md)：复杂批量与深度转换场景下现场编写轻量、临时、用完即走（Ephemeral）Python 脚本心智模型、不到 20 行极简标准库连接范式及五大高频场景（混乱文本清洗去重、多集合批量流水线注入、增量 Diff 同步、本地脚本语法预检一键推送、Web UI 表单失联全量巡检修复）实战模板。

### JavaScript 节点处理脚本
- [节点对象规范字典 (Proxy Schema)](references/script-proxy-schema.md)：Sub-Store 内部 `proxy` 对象的全量属性字典（通用字段与各协议专有字段）及函数签名。
- [JavaScript 节点处理脚本实战代码库](references/script-cookbook.md)：高频实战脚本（网络参数优化、地区标准重命名、Sing-Box tag 注入清洗、物理指纹哈希去重、垃圾节点剔除、平台 TLS 指纹自适应）。
