# Sub-Store JavaScript 节点对象规范字典 (Proxy Schema)

Sub-Store 在解析各种代理协议订阅（vmess, vless, ss, trojan, hysteria2 等）后，会在运行时统一将其标准化为 JavaScript 对象（`proxy`）。在自定义脚本的 `operator` 函数中，开发者可以直接读取并修改这些属性字段。

---

## 1. 脚本入口函数签名规范

所有用于 Sub-Store 流水线的 JavaScript 脚本必须导出名为 `operator` 的全局函数（支持同步函数或 `async` 异步函数）：

```javascript
/**
 * Sub-Store 节点处理入口函数
 * @param {Array<Object>} proxies 待处理的节点对象数组
 * @param {string} targetPlatform 当前编译的目标平台（如 clash-meta, singbox, surge 等）
 * @param {Object} context 运行时上下文对象（如 context.source, context.target）
 * @returns {Array<Object> | Promise<Array<Object>>} 返回修改后或过滤后的节点数组
 */
function operator(proxies = [], targetPlatform, context) {
  // 原地修改对象属性或通过 filter / map 返回新数组
  return proxies;
}
```

### 核心参数说明
- `proxies`: 节点对象数组，每个元素为一个代表特定代理协议的内部对象。
- `targetPlatform`: 客户端编译目标标识符（例如 `clash-meta`, `singbox`, `surge`, `loon`, `qx`, `shadowrocket` 等）。可用于在单一脚本中实现多平台自适应逻辑。
- `context`: 运行时上下文信息（如数据源元数据等）。

---

## 2. 通用基础属性字段

所有协议节点对象均继承或包含的基础字段：

| 字段名称 | 类型 | 说明与规范 |
|---|---|---|
| `name` | string | 节点显示名称（唯一标识） |
| `type` | string | 协议类型：`ss`, `ssr`, `vmess`, `vless`, `trojan`, `hysteria`, `hysteria2`, `tuic`, `wireguard`, `http`, `socks5` |
| `server` | string | 远程服务器域名或 IPv4/IPv6 地址 |
| `port` | number | 远程服务器端口（1 ~ 65535） |
| `udp` | boolean | 是否开启 UDP 转发支持（`true` / `false`） |
| `tfo` | boolean | 是否开启 TCP Fast Open（`true` / `false`） |
| `skip-cert-verify` | boolean | 是否跳过 TLS 证书合法性校验（`true` / `false`） |
| `tls` | boolean | 是否开启 TLS 加密通道 |
| `sni` | string | TLS SNI（Server Name Indication）服务器名称指示 |
| `alpn` | array of string | ALPN 协商协议列表，例如 `["h2", "http/1.1"]` 或 `["h3"]` |
| `ip` | string | 显式指定服务器连接的静态 IP 地址（绕过 DNS 解析） |
| `interface-name` | string | 指定出站绑定的物理网络接口名称 |
| `routing-mark` | number | Linux 路由标记（fwmark） |

---

## 3. 协议专有属性字段字典

### 3.1 Shadowsocks (`type: "ss"`)
- `cipher`: 加密算法（如 `aes-128-gcm`, `aes-256-gcm`, `chacha20-ietf-poly1305`, `2022-blake3-aes-128-gcm`）。
- `password`: 连接密码。
- `plugin`: 混淆插件类型（如 `obfs`, `v2ray-plugin`）。
- `plugin-opts`: 插件配置对象（如 `{ mode: "websocket", host: "example.com" }`）。

### 3.2 VMess (`type: "vmess"`)
- `uuid`: 用户唯一标识 UUID。
- `alterId`: 额外 ID（现代节点通常为 `0`）。
- `cipher`: 加密算法（默认 `auto`，或 `zero`, `aes-128-gcm`）。
- `network`: 传输层协议：`tcp`, `ws`, `grpc`, `h2`, `http`。
- `ws-opts`: WebSocket 传输参数对象：
  - `path`: 请求路径（如 `/ray`）。
  - `headers`: 自定义请求头对象（如 `{ Host: "cdn.example.com" }`）。
  - `max-early-data`: 早期数据长度。
  - `early-data-header-name`: 早期数据头部名称。
- `grpc-opts`: gRPC 传输参数对象：
  - `grpc-service-name`: gRPC 服务名称。
- `h2-opts`: HTTP/2 传输参数：
  - `host`: 域名数组。
  - `path`: 路径。

### 3.3 VLESS (`type: "vless"`)
- `uuid`: 用户唯一标识 UUID。
- `flow`: 流控算法（如 `xtls-rprx-vision`）。
- `network`: 传输协议：`tcp`, `ws`, `grpc`。
- `reality-opts`: REALITY 协议配置对象：
  - `public-key`: REALITY 服务端公钥。
  - `short-id`: REALITY 简短标识 ID。
- `client-fingerprint`: 客户端 TLS 指纹（如 `chrome`, `firefox`, `safari`）。
- `packet-encoding`: 数据包编码（如 `xudp`）。

### 3.4 Trojan (`type: "trojan"`)
- `password`: Trojan 认证密码。
- `sni`: 域名 SNI。
- `alpn`: ALPN 协议数组。
- `network`: 传输方式（`tcp`, `ws`, `grpc`）。
- `ws-opts` / `grpc-opts`: 对应传输层配置。

### 3.5 Hysteria / Hysteria2 (`type: "hysteria"` / `type: "hysteria2"`)
- `auth`: Hysteria2 认证密钥或密码。
- `up` / `down`: 上行/下行速率限制（字符串或数字，如 `"50 Mbps"` 或 `50`）。
- `obfs`: 混淆类型（如 `salamander`）。
- `obfs-password`: 混淆密钥。
- `sni`: 证书验证 SNI。
- `alpn`: ALPN 列表（通常包含 `h3`）。
- `ca` / `ca-str`: 自定义 CA 证书文件路径或证书 Base64/文本。

### 3.6 TUIC (`type: "tuic"`)
- `uuid`: 用户 UUID。
- `password`: 连接密码。
- `congestion_controller`: 拥塞控制算法（`bbr`, `cubic`, `new_reno`）。
- `udp_relay_mode`: UDP 转发模式（`native`, `quic`）。
- `zero_rtt_handshake`: 是否开启 0-RTT（布尔值）。

### 3.7 WireGuard (`type: "wireguard"`)
- `ip` / `ipv6`: 本地分配的客户端虚拟 IP 地址。
- `private-key`: 客户端私钥。
- `public-key`: 对端服务端公钥。
- `preshared-key`: 预共享密钥（可选）。
- `mtu`: 最大传输单元（如 `1420`）。
- `reserved`: 保留字节数组（如 `[0,0,0]`）。

### 3.8 HTTP / SOCKS5 (`type: "http"` / `type: "socks5"`)
- `username`: 代理认证用户名。
- `password`: 代理认证密码。
- `tls`: 是否通过 TLS 加密代理连接。
