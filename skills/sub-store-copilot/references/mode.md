# Sub-Store 运行形态与服务器探测指南

Sub-Store 主要存在两种运行形态：独立服务器（Standalone Server）与代理 MITM 脚本劫持（Proxy MITM Script Hijack）。在发起管理调用或自动化操作前，核心步骤是探测并锁定可用的服务基准端点。

---

## 一、两大运行形态概览

| 运行形态 | 运行载体 | 访问端点特征 | 核心注意点 |
|---|---|---|---|
| **独立服务器 (Standalone Server)** | 独立 HTTP 服务进程、VPS 或本地端口 | `http://127.0.0.1:3000` 或 `https://sub.example.com/<SECRET_PATH>` | 原生无密码认证；公网部署通常配置高熵随机前缀 `/<SECRET_PATH>`，未携带或前缀不匹配时直接返回 404。 |
| **代理 MITM 脚本劫持 (Proxy MITM Hijack)** | 托管于 Surge / Loon / Quantumult X / Shadowrocket 脚本引擎 | `http://sub.store` 或 `https://sub.store` | 虚拟域名在公网 DNS 无解析，请求必须经由宿主代理客户端接管（TUN/系统代理或指定本地代理端口如 6152 / 7890）。 |

---

## 二、服务器端点探测策略与核心逻辑

### 1. 探测优先级与候选列表

自动化运维或脚本探测时，按以下优先级逐级回退：

1. **环境变量显式注入（最高优先级）**：
   - 读取环境变量 `SUB_STORE_URL` 或 `SUB_STORE_PREFIX_URL`。若用户已显式配置，直接以该地址作为基准端点（末尾自动去除斜杠）。
2. **本地独立服务器默认端口**：
   - 探测本地回环地址 `http://127.0.0.1:3000`（无安全前缀的标准本地独立部署）。
3. **代理客户端 MITM 虚拟域名**：
   - 探测虚拟域名 `http://sub.store`。
   - 优先尝试直连（当前环境处于 TUN 增强模式或已配置代理环境变量）。
   - 若直连不可达，依次遍历常见代理端口（如 `127.0.0.1:6152`、`127.0.0.1:7890`、`127.0.0.1:1087` 等）探测。

### 2. 健康检查路径

探测时可通过请求以下路径验证服务可用性：
- `/api/subs`：返回订阅列表 JSON。返回 HTTP 200 即代表服务就绪且 API 正常响应。
- `/api/utils/env`：返回当前 Sub-Store 运行环境与内置配置信息。
- 根路径 `/`：返回 Web 控制台页面（无前缀独立服务），若配置了私密前缀且未命中时会返回 404。

---

## 三、自动化端点探测脚本

### 1. Bash 探测与环境变量导出函数

以下 Bash 函数自动按优先级执行探测。探测成功时自动导出 `SUB_STORE_URL`（并在需要代理接管时导出 `http_proxy` 与 `https_proxy`）：

```bash
detect_sub_store() {
  local candidates=()
  local proxy_ports=(6152 7890 1087 10808)

  # 1. 优先使用环境变量指定的目标
  if [ -n "${SUB_STORE_URL:-}" ]; then
    candidates+=("${SUB_STORE_URL%/}")
  elif [ -n "${SUB_STORE_PREFIX_URL:-}" ]; then
    candidates+=("${SUB_STORE_PREFIX_URL%/}")
  fi

  # 2. 本地独立服务器默认端点
  candidates+=("http://127.0.0.1:3000")

  # 3. 探测直连或环境变量已生效的候选端点
  for target in "${candidates[@]}"; do
    if curl -s -f --max-time 1.5 "${target}/api/subs" > /dev/null 2>&1; then
      export SUB_STORE_URL="${target}"
      echo "成功连接至 Sub-Store 端点: ${SUB_STORE_URL}"
      return 0
    fi
  done

  # 4. 探测代理客户端 MITM 虚拟域名 (直连 / TUN 接管模式)
  if curl -s -f --max-time 1.5 "http://sub.store/api/subs" > /dev/null 2>&1; then
    export SUB_STORE_URL="http://sub.store"
    echo "成功连接至 Sub-Store 代理 MITM 端点 (直连/TUN): ${SUB_STORE_URL}"
    return 0
  fi

  # 5. 遍历常见代理端口探测 MITM 虚拟域名
  for port in "${proxy_ports[@]}"; do
    if curl -x "http://127.0.0.1:${port}" -s -f --max-time 1.5 "http://sub.store/api/subs" > /dev/null 2>&1; then
      export SUB_STORE_URL="http://sub.store"
      export http_proxy="http://127.0.0.1:${port}"
      export https_proxy="http://127.0.0.1:${port}"
      echo "成功通过本地代理端口 (127.0.0.1:${port}) 连接至 Sub-Store MITM 端点: ${SUB_STORE_URL}"
      return 0
    fi
  done

  echo "未探测到可用的 Sub-Store 实例。请确认独立服务已启动或代理客户端已开启 MITM 模块。" >&2
  return 1
}
```

### 2. Python 自动化探测实现（零第三方依赖）

在临时 Python 运维脚本中，可直接内置以下函数快速定位服务基准地址：

```python
import os
import urllib.request

def detect_sub_store():
    # 1. 优先检查环境变量
    env_url = os.environ.get("SUB_STORE_URL") or os.environ.get("SUB_STORE_PREFIX_URL")
    candidates = [env_url.rstrip("/")] if env_url else []
    
    # 2. 本地独立服务器与直连虚拟域名
    candidates.extend(["http://127.0.0.1:3000", "http://sub.store"])
    
    for url in candidates:
        try:
            req = urllib.request.Request(f"{url}/api/subs", method="GET")
            with urllib.request.urlopen(req, timeout=1.5) as resp:
                if resp.status == 200:
                    return url, None
        except Exception:
            continue

    # 3. 遍历常见本地代理端口探测虚拟域名
    proxy_ports = [6152, 7890, 1087, 10808]
    for port in proxy_ports:
        proxy_url = f"http://127.0.0.1:{port}"
        try:
            handler = urllib.request.ProxyHandler({"http": proxy_url, "https": proxy_url})
            opener = urllib.request.build_opener(handler)
            req = urllib.request.Request("http://sub.store/api/subs", method="GET")
            with opener.open(req, timeout=1.5) as resp:
                if resp.status == 200:
                    return "http://sub.store", proxy_url
        except Exception:
            continue

    return None, None

# 使用示例
if __name__ == "__main__":
    base_url, proxy = detect_sub_store()
    if base_url:
        print(f"Sub-Store 可用端点: {base_url} (代理: {proxy or '直连'})")
    else:
        print("未检测到可用的 Sub-Store 服务端点")
```
