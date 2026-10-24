# Sub-Store 内置算子参数规范字典

Sub-Store 支持在订阅（Subscription）或集合（Collection）的 `process` 数组中声明算子流水线。算子严格按声明顺序自上而下链式执行。算子名称（`type` 字段）严格区分大小写。

> [!IMPORTANT]
> **核心规范：前端 Web UI 表单双向绑定与全量 Schema 严格兼容原则**
> Sub-Store 系统架构中存在一个极易忽视的致命陷阱：**后端执行引擎对入参高度宽容，但前端 Web 控制台组件严格依赖特定的响应式表单字段**。
> - **核心陷阱机制**：若通过脚本或 API 仅传入简写或降级字段（例如 Script Operator 传 `url`/`path` 而非 `mode` + `content`，或 Regex Filter 缺少 `action` 与 `keep` 的双向联动），在客户端通过 `/download/...` 拉取节点并执行流水线时完全能够正常执行；但只要用户在浏览器 Web 控制台中打开该集合查看，前端表单控件因缺失响应式驱动字段，单选框会失焦脱节、输入框呈现空白。当用户在网页端随意修改其他配置并点击“保存”时，前端会将这组空白表单回传并覆写后端，导致原算子配置彻底丢失！
> - **全算子防御原则**：该问题不仅限于 Script Operator，而是普遍存在于 Regex Filter、Rename、Flag、Sort、Deduplicate 等所有算子中。因此，所有算子与对象构造必须**一律采用与前端 UI 严格对齐的全量字段 Schema**，确保“**后端执行 100% 生效 + 前端 Web UI 完美可视与安全交互编辑**”。

---

## 1. 常用内置算子规范与前端 Web UI 控件映射字典

| 算子名称 (`type`) | 前端 Web 控件对应形态 | 必填/关键参数 (`args`) | 类型 | 前端双向绑定规范与联动说明 |
|---|---|---|---|---|
| **Script Operator**（兼容 `Script`） | 单选切换（Remote Link / Local Content）+ 远程地址输入框 / Monaco 代码编辑器 | `mode` | string | **单选模式驱动字段**：`"link"` 激活 Remote Link；`"script"` 激活 Local Content。缺失时单选框脱节失焦。 |
| | | `content` | string | **内容双向绑定字段**：当 `mode: "link"` 时填入脚本 URL（如 `/api/file/cleaner.js` 或外部链接）；当 `mode: "script"` 时填入完整 JavaScript 源码。若误传为 `url` 或 `path`，前端输入框显示为空白。 |
| **Regex Filter** | 正则表达式输入框（Pattern）+ 动作单选组（Action: Keep 保留 / Drop 丢弃） | `pattern` | string | 正则表达式匹配规则（支持 `(?i)` 忽略大小写标识）。 |
| | | `action` | string | **动作标识字符串**：`"keep"`（保留匹配节点）或 `"drop"`（剔除匹配节点）。 |
| | | `keep` | boolean | **动作布尔联动值**：保留填 `true`，剔除填 `false`。必须与 `action` 同步提供，防止不同 UI 版本单选状态错位或反转。 |
| **Rename** | 匹配表达式输入框（Pattern）+ 替换模板输入框（Replace） | `pattern` | string | 正则匹配表达式（支持捕获组）。 |
| | | `replace` | string | 替换目标字符串（支持捕获组引用如 `$1`）。 |
| **Flag** | 处理模式下拉/单选框（Mode: Add / Remove） | `mode` | string | 国旗处理模式：`"add"`（补全国旗 Emoji）或 `"remove"`（移除国旗）。 |
| **Sort** | 排序字段下拉框（Field）+ 排序方向下拉框（Order） | `field` | string | 排序依据字段：`"name"`（节点名称）或 `"latency"`（延迟）。 |
| | | `order` | string | 排序方向：`"asc"`（升序）或 `"desc"`（降序）。 |
| **Deduplicate** | 去重字段下拉框（Field） | `field` | string | 去重判定字段：`"name"`（按节点名去重）或 `"server"`（按服务器地址去重）。 |

> [!NOTE]
> 为避免排版解析异常，在编写文档与提示词时切勿在 Markdown 表格中滥用 HTML 标签。上述算子展示名称为 Sub-Store Web 控制台标准名称。

---

## 2. 算子 JSON 结构与拼装范例（严格兼容前端 Web UI）

### 2.1 挂载执行自定义 JavaScript 脚本 (`Script Operator`)

前端 UI 依赖 `args.mode` 与 `args.content` 双向绑定渲染表单：
- `mode: "link"`：前端单选框选中 **Remote Link**，并将 `content` 填入 URL 链接输入框；
- `mode: "script"`：前端单选框选中 **Local Content**，并将 `content` 渲染在 Monaco 代码编辑器中；
- 算子名称在前端 UI 展示为 `Script Operator`（后端向下兼容 `Script`，推荐统一写作 `Script Operator`）。

#### 范式 A：引用虚拟文件系统或远程脚本（Remote Link 推荐）
```json
{
  "type": "Script Operator",
  "args": {
    "mode": "link",
    "content": "/api/file/optimize-net.js",
    "arguments": {}
  }
}
```

#### 范式 B：内嵌 JavaScript 源码（Local Content）
```json
{
  "type": "Script Operator",
  "args": {
    "mode": "script",
    "content": "function operator(proxies = [], targetPlatform) {\n  return proxies.map(p => {\n    p.udp = true;\n    p.tfo = true;\n    return p;\n  });\n}",
    "arguments": {}
  }
}
```

> [!WARNING]
> **避坑警示**：切勿使用 `args: { "url": "/api/file/optimize-net.js" }` 或 `args: { "path": "..." }`！虽然后端在处理订阅时能兼容读取，但前端 UI 无法识别，会显示为空白输入框；一旦用户在 Web UI 点击保存，该算子将被空值覆盖清空。

---

### 2.2 正则保留目标地区节点 (`Regex Filter` - keep)
仅保留包含香港、日本、新加坡、美国关键词的节点。注意 `action` 与 `keep` 必须联动传入：
```json
{
  "type": "Regex Filter",
  "args": {
    "pattern": "(?i)港|hk|日|jp|新|sg|美|us",
    "action": "keep",
    "keep": true
  }
}
```

---

### 2.3 正则剔除垃圾与广告节点 (`Regex Filter` - drop)
剔除包含官网、到期、通知、重置等说明性质的非可用节点。注意 `action` 为 `"drop"` 且 `keep` 为 `false`：
```json
{
  "type": "Regex Filter",
  "args": {
    "pattern": "(?i)官网|到期|剩余|通知|重置|群|公告",
    "action": "drop",
    "keep": false
  }
}
```

> [!WARNING]
> **避坑警示**：若仅传 `action` 或仅传 `keep`，不同版本的前端 Web 控制台可能无法正确高亮选中的单选钮，导致在界面上再次编辑时状态发生意外翻转。

---

### 2.4 正则替换批量重命名 (`Rename`)
为所有节点统一增加 `[专线]` 前缀：
```json
{
  "type": "Rename",
  "args": {
    "pattern": "^(.*)$",
    "replace": "[专线] $1"
  }
}
```

---

### 2.5 自动补全国旗 Emoji (`Flag`)
根据节点名称自动识别国家/地区并追加国旗 Emoji：
```json
{
  "type": "Flag",
  "args": {
    "mode": "add"
  }
}
```

---

### 2.6 节点排序 (`Sort`)
按节点名称升序排列：
```json
{
  "type": "Sort",
  "args": {
    "field": "name",
    "order": "asc"
  }
}
```

---

### 2.7 节点去重 (`Deduplicate`)
按节点名称唯一化排重：
```json
{
  "type": "Deduplicate",
  "args": {
    "field": "name"
  }
}
```

---

## 3. 前端 Web UI 双向绑定机制与避坑防范

### 3.1 为什么“后端宽容 + 前端严格”是最隐蔽的故障源
Sub-Store 的后端使用 Node.js 编写，各算子在执行时采用了宽松的参数降级逻辑：
- 在执行脚本时，后端逻辑大致为：`const scriptUrl = args.content || args.url || args.path;`
- 在执行正则过滤时，后端逻辑大致为：`const isKeep = args.keep !== undefined ? args.keep : (args.action === 'keep');`

因此，即便第三方脚本提交了残缺或非标准的 JSON，测试接口 `/download/collection/...` 时节点依然能被正确清洗，容易让人误以为配置完全正确。

然而，前端 Web UI 是基于 Vue 的响应式表单开发，各表单控件严格绑定特定的属性名：
1. **表单初始化**：Web 控制台加载集合时，根据 `args.mode` 决定勾选哪个单选框（Remote Link 还是 Local Content）；根据 `args.content` 决定在文本框中填充什么内容。若没有 `mode`，两个单选框都可能处于未选中的异常状态；若填的是 `url`，输入框显示为空字符串。
2. **表单回写损坏**：用户在 Web 控制台中调整了集合的其他选项（例如加了一个去重算子），点击保存时，Vue 表单将直接采集当前界面上的所有控件值（此时 Script Operator 控件输出的是空字符串）并打包发送 `POST /api/collections`，瞬间抹掉之前的脚本地址！

---

## 4. 常见问题与 UI 修复：一键修正已有的 Script Operator 与算子配置

当发现已有集合在 Web 控制台中显示异常时，可通过以下方案一键修复已存在的配置：

### 4.1 单集合一键修正（curl + jq 管道）

```bash
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
```

### 4.2 验证修复结果

```bash
curl -s "${SUB_STORE_URL}/api/collection/${COLL_NAME}" | jq .process
```

---

## 5. 流水线执行生命周期与最佳实践

算子流水线通常建议按以下次序装配：
1. **Script Operator 预处理**：先执行自定义脚本，完成底层属性修正、ALPN 注入与网络优化。
2. **Regex Filter 剔除垃圾**：剔除公告与不可用节点，降低后续算子开销。
3. **Flag 补全国旗**：规范化地域 Emoji 标识。
4. **Rename 重命名**：执行统一的前缀或格式修正。
5. **Deduplicate 排重**：去除可能存在的同名或同服务器节点。
6. **Sort 排序**：最终按照标准顺序呈现。
