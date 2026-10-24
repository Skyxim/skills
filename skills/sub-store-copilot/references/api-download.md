# Sub-Store 节点导出与下载接口规范 (`/download/...`)

Sub-Store 提供动态节点导出接口，支持将单订阅（Sub）或聚合集合（Collection）经过算子流水线实时清洗、重构后，以各种目标客户端原生配置格式输出。导出的 URL 可直接交付给外部客户端作为 Proxy Provider 订阅链接。

所有端点均基于当前模式的前缀 URL `{SUB_STORE_URL}`。

---

## 1. 导出端点路径说明

| 请求方法 | 端点路径 | 说明 |
|---|---|---|
| `GET` | `/download/collection/:name` | 导出指定聚合集合（Collection）清洗后的节点数据 |
| `GET` | `/download/sub/:name` | 导出指定单订阅（Sub）清洗后的节点数据 |
| `GET` | `/download/:name` | 兼容别名：导出指定集合或订阅的节点数据 |

---

## 2. target 目标平台参数规范

通过在 URL 后追加 `?target={target}` 查询参数，Sub-Store 会自动将内部 Proxy 对象转换为对应客户端的原生格式：

| `target` 参数值 | 适用客户端 | 输出格式与特征 |
|---|---|---|
| `clash-meta` 或 `clash` | Mihomo (Clash.Meta) / Clash Premium | YAML 纯节点列表格式，专为 `proxy-providers` 设计 |
| `singbox` | Sing-Box | JSON 格式，输出包含所有节点的 `outbounds` 列表 |
| `surge` | Surge 4 / 5 | Surge 原生纯文本 `[Proxy]` 节点配置段 |
| `loon` | Loon | Loon 原生节点列表格式 |
| `qx` | Quantumult X | Quantumult X 原生节点列表格式 |
| `shadowrocket` | Shadowrocket (小火箭) | 小火箭原生节点列表或混合链接 |
| *(省略 target)* | 通用客户端 | 默认返回标准 Base64 编码节点或 URI 原始协议节点文本 |

---

## 3. 命令行预览与下载示例

```bash
# 1. 导出并预览 Mihomo (Clash.Meta) 格式节点（前 30 行）
curl -s "${SUB_STORE_URL}/download/collection/All-Nodes?target=clash-meta" | head -n 30

# 2. 导出并预览 Sing-Box 格式 outbounds JSON 节点
curl -s "${SUB_STORE_URL}/download/collection/All-Nodes?target=singbox" | head -n 30

# 3. 导出 Surge 格式节点
curl -s "${SUB_STORE_URL}/download/collection/All-Nodes?target=surge" | head -n 30

# 4. 获取原始 Base64 / URI 链接
curl -s "${SUB_STORE_URL}/download/collection/All-Nodes"
```

---

## 4. 外部客户端 Proxy Provider 对接范例

### 4.1 Mihomo (Clash.Meta) 配置接入范例

在 Mihomo 主配置文件的 `proxy-providers` 中直接引用下载链接：

```yaml
proxy-providers:
  All-Nodes-Provider:
    type: http
    url: "${SUB_STORE_URL}/download/collection/All-Nodes?target=clash-meta"
    path: ./providers/all-nodes.yaml
    interval: 3600
    health-check:
      enable: true
      url: http://www.gstatic.com/generate_204
      interval: 300

proxy-groups:
  - name: "AUTO-SELECT"
    type: url-test
    use:
      - All-Nodes-Provider
    url: http://www.gstatic.com/generate_204
    interval: 300
```

### 4.2 Sing-Box 出站集对接范例

在 Sing-Box 配置中，可配合自定义脚本或作为外部 provider 引入导出的 outbounds JSON：

```json
{
  "outbounds": [
    {
      "type": "selector",
      "tag": "select-node",
      "outbounds": [
        "auto-node"
      ]
    }
  ]
}
```
可通过本地脚本或定时任务同步拉取 `${SUB_STORE_URL}/download/collection/All-Nodes?target=singbox` 并载入配置。
