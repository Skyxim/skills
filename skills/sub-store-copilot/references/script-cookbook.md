# Sub-Store JavaScript 节点处理脚本实战代码库

本手册提供了一系列开箱即用、经过严格验证的高频 JavaScript 节点处理脚本。所有脚本均符合 Sub-Store 的 `operator(proxies, targetPlatform, context)` 规范。

> [!IMPORTANT]
> **本地临时脚本存放与自解释命名规范**：
> - **统一存储目录**：所有现场定制编写的 JavaScript 脚本统一存放在 `/tmp/sub-store-copilot/` 目录下，严禁在主工作区随处新建文件，保持项目环境整洁。
> - **文件命名格式**：`/tmp/sub-store-copilot/<capability-name>.js`。
> - **自解释命名原则**：`<capability-name>` 必须具备清晰的自解释语义，准确表达脚本具体能力（如 `rename-region-flags.js`、`optimize-udp-tfo.js`、`inject-singbox-tags.js`、`dedup-node-fingerprint.js`、`clean-ads-expire.js`、`platform-tls-fingerprint.js` 等），严禁使用 `test.js`、`temp.js`、`1.js` 等无意义命名。
> - **标准工作流程**：在本地 `/tmp/sub-store-copilot/<capability-name>.js` 编写并校验 -> 使用 `curl --data-binary @/tmp/sub-store-copilot/<capability-name>.js` 二进制上传至 `/api/file/<capability-name>.js` -> 在算子流水线中引用。

---

## 1. 地区规范化识别与带国旗标准重命名 (`rename-region-flags.js`)

- **本地文件路径**：`/tmp/sub-store-copilot/rename-region-flags.js`
- **Sub-Store 存储端点**：`/api/file/rename-region-flags.js`

清洗机场节点名称中的广告与杂质（官网、TG群、通知、到期时间），识别国家/地区并统一规范为“国旗 Emoji + 地区中文 + 编号 + 倍率”的标准命名：

```javascript
function operator(proxies = []) {
  const REGEX_RATE = /(?:^|\s|[|\[(])([0-9]+(?:\.[0-9]+)?)\s*[xX倍](?:$|\s|[|\])])/;

  const REGIONS = [
    { flag: "🇭🇰", name: "香港", reg: /(?:香港|HK|Hong\s*Kong|HongKong)/i },
    { flag: "🇯🇵", name: "日本", reg: /(?:日本|JP|Japan|Tokyo|Osaka)/i },
    { flag: "🇸🇬", name: "新加坡", reg: /(?:新加坡|SG|Singapore)/i },
    { flag: "🇺🇸", name: "美国", reg: /(?:美国|US|USA|America|United\s*States)/i },
    { flag: "🇹🇼", name: "台湾", reg: /(?:台湾|TW|Taiwan)/i },
    { flag: "🇰🇷", name: "韩国", reg: /(?:韩国|KR|Korea|Seoul)/i },
    { flag: "🇬🇧", name: "英国", reg: /(?:英国|UK|GB|Britain|London)/i },
    { flag: "🇩🇪", name: "德国", reg: /(?:德国|DE|Germany|Frankfurt)/i }
  ];

  const counters = {};

  return proxies.map(p => {
    const raw = p.name || "";

    // 1. 提取倍率
    let rate = "1.0";
    const matchRate = raw.match(REGEX_RATE);
    if (matchRate) {
      rate = matchRate[1];
    }

    // 2. 匹配国家或地区
    let matchedRegion = null;
    for (const r of REGIONS) {
      if (r.reg.test(raw)) {
        matchedRegion = r;
        break;
      }
    }

    if (matchedRegion) {
      const key = matchedRegion.name;
      counters[key] = (counters[key] || 0) + 1;
      const indexStr = String(counters[key]).padStart(2, "0");
      p.name = `${matchedRegion.flag} ${matchedRegion.name} ${indexStr} [${rate}x]`;
    } else {
      // 无法识别地区时兜底剔除广告杂质
      p.name = raw
        .replace(/https?:\/\/\S+/gi, "")
        .replace(/(?:官网|网址|到期|剩余|流量|通知|群|TG)[^| -]*/gi, "")
        .trim();
    }
    return p;
  });
}
```

---

## 2. 网络参数优化与安全批量覆写 (`optimize-udp-tfo.js`)

- **本地文件路径**：`/tmp/sub-store-copilot/optimize-udp-tfo.js`
- **Sub-Store 存储端点**：`/api/file/optimize-udp-tfo.js`

批量为支持的节点开启 UDP 与 TCP Fast Open；针对 443 端口与 TLS 节点配置跳过证书校验；为 Hysteria2 / Trojan 智能补全 ALPN：

```javascript
function operator(proxies = []) {
  return proxies.map(proxy => {
    // 开启常用传输加速与 UDP
    proxy.udp = true;
    proxy.tfo = true;

    // 证书与 TLS 自适应优化
    if (proxy.port === 443 || proxy.tls === true || proxy.network === "ws") {
      proxy["skip-cert-verify"] = true;
    }

    // 协议专有 ALPN 注入
    if (proxy.type === "hysteria2") {
      proxy.alpn = proxy.alpn || ["h3"];
    } else if (proxy.type === "trojan" || proxy.type === "vless") {
      proxy.alpn = proxy.alpn || ["h2", "http/1.1"];
    }

    return proxy;
  });
}
```

---

## 3. Sing-Box 出站标签（tag）唯一化与字符清洗 (`inject-singbox-tags.js`)

- **本地文件路径**：`/tmp/sub-store-copilot/inject-singbox-tags.js`
- **Sub-Store 存储端点**：`/api/file/inject-singbox-tags.js`

Sing-Box 对 `outbounds` 中的 `tag` 要求严苛：**严禁重名**、**严禁空格与特殊字符**。本脚本保证所有节点生成合法、全局唯一的 tag：

```javascript
function operator(proxies = [], targetPlatform) {
  const tagCounts = new Map();

  return proxies.map((p, idx) => {
    let raw = p.name || `proxy-${idx}`;

    // 替换空格与路径斜杠为中划线，剔除不可见特殊符号
    let clean = raw
      .replace(/[\s\/\\]+/g, "-")
      .replace(/[^a-zA-Z0-9_\u4e00-\u9fa5\.\-]/g, "")
      .trim();

    if (!clean) clean = `node-${idx}`;

    // 计数排重
    const count = tagCounts.get(clean) || 0;
    tagCounts.set(clean, count + 1);

    const uniqueTag = count === 0 ? clean : `${clean}-${count + 1}`;
    p.name = uniqueTag;
    p.tag = uniqueTag;

    return p;
  });
}
```

---

## 4. 底层物理指纹哈希去重 (`dedup-node-fingerprint.js`)

- **本地文件路径**：`/tmp/sub-store-copilot/dedup-node-fingerprint.js`
- **Sub-Store 存储端点**：`/api/file/dedup-node-fingerprint.js`

单纯按名称去重容易误杀合法节点，或者遗漏名称不同但服务器 IP、端口及认证凭据完全相同的垃圾重复节点。本脚本基于 `server + port + credential` 构造物理指纹进行绝对去重：

```javascript
function operator(proxies = []) {
  const seen = new Set();

  return proxies.filter(p => {
    const credential = p.uuid || p.password || p.auth || "";
    const fingerprint = `${p.type}://${p.server}:${p.port}/${credential}`;

    if (seen.has(fingerprint)) {
      return false;
    }
    seen.add(fingerprint);
    return true;
  });
}
```

---

## 5. 垃圾通知与不可用节点过滤 (`clean-ads-expire.js`)

- **本地文件路径**：`/tmp/sub-store-copilot/clean-ads-expire.js`
- **Sub-Store 存储端点**：`/api/file/clean-ads-expire.js`

过滤掉机场用来发布公告、展示到期时间的假节点，以及端口非法的异常节点：

```javascript
function operator(proxies = []) {
  const JUNK_REGEX = /(?:到期|剩余|重置|官网|通知|维护|测试|traffic|expire|reset|notice|群|公告)/i;

  return proxies.filter(p => {
    const name = p.name || "";
    if (JUNK_REGEX.test(name)) return false;

    // 端口合法性校验
    const port = Number(p.port);
    if (!port || port <= 0 || port > 65535) return false;

    // 必须具备有效服务器地址
    if (!p.server || typeof p.server !== "string" || p.server.trim() === "") {
      return false;
    }

    return true;
  });
}
```

---

## 6. 目标平台自适应与 TLS 指纹注入 (`platform-tls-fingerprint.js`)

- **本地文件路径**：`/tmp/sub-store-copilot/platform-tls-fingerprint.js`
- **Sub-Store 存储端点**：`/api/file/platform-tls-fingerprint.js`

针对不同客户端编译目标自适应注入配置（例如仅为 Mihomo 注入 `client-fingerprint: chrome`）：

```javascript
function operator(proxies = [], targetPlatform) {
  const isMeta = targetPlatform && targetPlatform.toLowerCase().includes("meta");

  return proxies.map(p => {
    // 若目标是 Clash.Meta 且节点支持 TLS
    if (isMeta && (p.tls || p.type === "vless" || p.type === "trojan")) {
      p["client-fingerprint"] = "chrome";
    }

    // 统一剔除无效的空字段
    Object.keys(p).forEach(key => {
      if (p[key] === null || p[key] === undefined || p[key] === "") {
        delete p[key];
      }
    });

    return p;
  });
}
```

---

## 7. 脚本一键推送与调试

### 7.1 本地编写并推送脚本到 Sub-Store
```bash
# 1. 在规范临时目录写入自解释命名的处理脚本
mkdir -p /tmp/sub-store-copilot
cat << 'EOF' > /tmp/sub-store-copilot/clean-ads-expire.js
function operator(proxies = []) {
  return proxies.filter(p => !/(?:官网|到期|剩余|重置)/i.test(p.name || ""));
}
EOF

# 2. 通过二进制上传推送到 Sub-Store 虚拟文件存储
curl -s -X POST "${SUB_STORE_URL}/api/file/clean-ads-expire.js" \
  -H "Content-Type: text/javascript" \
  --data-binary @/tmp/sub-store-copilot/clean-ads-expire.js
```

### 7.2 实时预览清洗效果
```bash
# 预览 Mihomo 格式清洗节点
curl -s "${SUB_STORE_URL}/download/collection/All-Nodes?target=clash-meta" | head -n 30

# 预览 Sing-Box 格式清洗节点
curl -s "${SUB_STORE_URL}/download/collection/All-Nodes?target=singbox" | head -n 30
```
