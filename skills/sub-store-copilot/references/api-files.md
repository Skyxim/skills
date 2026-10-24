# Sub-Store 虚拟文件存储接口规范 (`/api/files` & `/api/file/:name`)

Sub-Store 内置轻量级虚拟文件存储（Virtual File Store），用于持久化保存用户定制的 JavaScript 节点清洗、属性覆写和协议定制脚本。

所有端点均基于当前模式的前缀 URL `{SUB_STORE_URL}`。

---

## 1. 端点概览

| 请求方法 | 端点路径 | 功能描述 |
|---|---|---|
| `GET` | `/api/files` | 获取系统已保存的自定义脚本与文件列表 |
| `GET` | `/api/file/:name` | 读取指定脚本的原始 JavaScript 纯文本代码 |
| `POST` | `/api/file/:name` | 上传、创建或全量覆写指定 JavaScript 脚本 |
| `DELETE` | `/api/file/:name` | 删除指定的脚本文件 |

---

## 2. 接口操作命令与示例

### 2.1 获取全量脚本文件列表
- **Method**: `GET`
- **URL**: `{SUB_STORE_URL}/api/files`
- **Response (200 OK)**:
```json
[
  "clean-ads-expire.js",
  "optimize-udp-tfo.js",
  "inject-singbox-tags.js"
]
```

### 2.2 上传/覆写 JavaScript 脚本 (`POST /api/file/:name`)
通过 HTTP 请求头 `Content-Type: text/javascript` 或 `application/javascript` 提交，请求体直接为原始 JavaScript 脚本代码。

> [!IMPORTANT]
> **本地脚本存放路径与能力命名规范**：
> - **统一存储目录**：所有现场定制编写的 JavaScript 脚本统一存放在 `/tmp/sub-store-copilot/` 目录下，严禁在主工作区随处落盘，保持项目环境整洁。
> - **文件命名格式**：`/tmp/sub-store-copilot/<capability-name>.js`。
> - **自解释命名原则**：`<capability-name>` 必须具备清晰的自解释语义，准确表达脚本具体能力（如 `optimize-udp-tfo.js`、`clean-ads-expire.js`、`inject-singbox-tags.js`、`rename-region-flags.js` 等），严禁使用 `test.js`、`temp.js`、`1.js` 等模糊命名。
> - **标准流程**：在本地 `/tmp/sub-store-copilot/<capability-name>.js` 编写并验证 -> 使用 `curl --data-binary @/tmp/sub-store-copilot/<capability-name>.js` 二进制干净上传至 `/api/file/<capability-name>.js` -> 算子流中引用。

#### 标准工作流：本地编写并二进制干净上传（推荐）
```bash
# 1. 现场创建临时目录并写入自解释命名的 JS 脚本
mkdir -p /tmp/sub-store-copilot
cat << 'EOF' > /tmp/sub-store-copilot/optimize-udp-tfo.js
function operator(proxies = [], targetPlatform) {
  return proxies.map(p => {
    p.udp = true;
    p.tfo = true;
    if (p.port === 443 || p.tls === true || p.network === "ws") {
      p["skip-cert-verify"] = true;
    }
    return p;
  });
}
EOF

# 2. 通过 --data-binary 二进制完整上传
curl -s -X POST "${SUB_STORE_URL}/api/file/optimize-udp-tfo.js" \
  -H "Content-Type: text/javascript" \
  --data-binary @/tmp/sub-store-copilot/optimize-udp-tfo.js
```

#### 快捷管道流上传（单行即推）
亦可使用 `curl` 与管道输入（here-doc）一键推送到 Sub-Store：

```bash
curl -s -X POST "${SUB_STORE_URL}/api/file/optimize-udp-tfo.js" \
  -H "Content-Type: text/javascript" \
  --data-binary @- << 'EOF'
function operator(proxies = [], targetPlatform) {
  return proxies.map(p => {
    p.udp = true;
    p.tfo = true;
    if (p.port === 443 || p.tls === true || p.network === "ws") {
      p["skip-cert-verify"] = true;
    }
    return p;
  });
}
EOF
```

### 2.3 读取脚本内容 (`GET /api/file/:name`)
- **Method**: `GET`
- **URL**: `{SUB_STORE_URL}/api/file/optimize-udp-tfo.js`
- **Response (200 OK)**: 返回该文件保存的完整原始 JavaScript 代码文本。

### 2.4 删除指定脚本 (`DELETE /api/file/:name`)
```bash
curl -s -X DELETE "${SUB_STORE_URL}/api/file/optimize-udp-tfo.js"
```

---

## 3. 在算子流水线中引用文件

上传文件后，在集合或订阅的 `process` 数组中通过 `Script Operator`（或 `Script`）算子声明引用。前端 UI 渲染依赖 `args.mode: "link"` 激活 Remote Link 模式，并将 `args.content` 绑定到链接输入框：

```json
{
  "type": "Script Operator",
  "args": {
    "mode": "link",
    "content": "/api/file/optimize-udp-tfo.js"
  }
}
```
