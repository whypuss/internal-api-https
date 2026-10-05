# internal-api-https

> 為局域網內網 AI API（如 Sub2API、Ollama、LiteLLM）提供穩定、防超時中斷的 HTTPS 穿透與本地 TLS 終止方案，專門適配 Mirasim、OpenCode、Claude Code 等嚴格要求 HTTPS 與長流式（SSE）推理的 Agent 客戶端。

---

## 📖 背景與解決的痛點

當我們把 **Mirasim** 等 AI Agent 客戶端連接到局域網內的自建 AI 中轉服務器（例如 **Sub2API** `http://192.168.x.x:8090`）時，經常會遇到以下問題：

1. **客戶端強制要求 HTTPS / 合法域名**：Mirasim 內部嚴格校驗 URL 格式，不允許直接填寫 `http://` 內網 IP。
2. **Cloudflare 邊緣 100 秒硬性超時（HTTP 524）**：Agent 在做多步驟任務、思考（Thinking）或生成超長代碼時，一旦單次請求超過 100 秒未完全輸出，Cloudflare CDN 會強制掐斷連線。
3. **SSE 流式長連接斷線**：未優化的 Tunnel 默認超時過短，在模型推理靜默期間容易被當作空閒連接踢掉。
4. **VPN / 代理網卡衝突（ECONNRESET / 1033 報錯）**：本機開啟 TUN/虛擬網卡代理（如 Karing、Clash、Sing-Box）時，DNS Fake-IP 劫持了 Cloudflare 邊緣節點，導致 TLS 握手被重置。
5. **缺少開機自啟與守護**：手動開終端跑指令，一旦重開機或休眠喚醒就會失效。

本倉庫提供了兩套經過實戰驗證的架構方案：
- **方案 A（Cloudflare Tunnel 長連接優化 + macOS 守護）**：免維護證書、支持外網或跨網段訪問。
- **方案 B（本地 Caddy 反代終止）**：流量完全在內網千兆消化，徹底打破 Cloudflare 100 秒超時限制。

---

## 🚀 部署方案

### 方案 A：Cloudflare Tunnel 優化配置（含 macOS 開機自啟）

#### 1. Ingress 配置文件 (`cloudflared.example.yml`)
放置於 `~/.cloudflared/<tunnel-name>.yml`：

```yaml
tunnel: <你的_TUNNEL_UUID>
credentials-file: /Users/<用戶名>/.cloudflared/<你的_TUNNEL_UUID>.json

ingress:
  - hostname: mira.yourdomain.com
    service: http://192.168.31.55:8090
    originRequest:
      connectTimeout: 30s
      keepAliveTimeout: 10m        # 保持後端連接 10 分鐘，防 SSE 間隔斷開
      keepAliveConnections: 100    # 支援更高並發子請求
      tcpKeepAlive: 30s            # 30秒 TCP 心跳包，防路由器 NAT 超時清除
      http2Origin: false           # 避免與上游 chunked SSE 產生協議衝突
      disableChunkedEncoding: false
  - service: http_status:404
```

#### 2. 綁定 DNS 路由
```bash
cloudflared tunnel route dns <你的_TUNNEL_UUID> mira.yourdomain.com
```

#### 3. macOS 系統級持久化守護 (`launchagent.example.plist`)
複製到 `~/Library/LaunchAgents/com.cloudflare.<tunnel-name>.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.cloudflare.<tunnel-name></string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/zsh</string>
        <string>-c</string>
        <string>exec /opt/homebrew/bin/cloudflared tunnel --protocol http2 --config /Users/<用戶名>/.cloudflared/<tunnel-name>.yml run</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>NetworkState</key>
        <true/>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/<用戶名>/.cloudflared/<tunnel-name>.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/<用戶名>/.cloudflared/<tunnel-name>.log</string>
</dict>
</plist>
```

載入開機自啟：
```bash
launchctl load -w ~/Library/LaunchAgents/com.cloudflare.<tunnel-name>.plist
```

---

### 方案 B：本地 Caddy 反代（徹底消除 100 秒超時）

如果任務經常需要超過 100 秒的思考時間，建議直接在局域網/本地進行 HTTPS 反代：

#### 1. Caddyfile 配置 (`Caddyfile.example`)
```caddy
{
    http_port 8080
    https_port 443
}

mira.yourdomain.com {
    tls internal
    reverse_proxy 192.168.31.55:8090 {
        flush_interval -1
        header_up Host {host}
        header_up X-Real-IP {remote_host}
    }
}
```

#### 2. 啟動並信任證書
```bash
sudo caddy start --config ./Caddyfile
sudo caddy trust
```

#### 3. 本地 Hosts 劫持直連
在 `/etc/hosts` 中加入：
```text
127.0.0.1 mira.yourdomain.com
```

---

## 🛠️ 排錯與診斷手冊

| 錯誤代碼 / 現象 | 核心原因 | 解決方法 |
| :--- | :--- | :--- |
| **ECONNRESET / 1033** | VPN 軟件（如 Karing）劫持了 Cloudflare IP 握手失敗 | 在 VPN 規則中將該域名設置為「直連 (DIRECT)」，或重啟 Tunnel |
| **HTTP 530** | 本地 `cloudflared` 進程中斷，Cloudflare 找不到後端 | 檢查本機 Tunnel 進程，確保其正常連上 edge 節點 |
| **HTTP 524** | Cloudflare 雲端 CDN 代理的 100 秒硬性超時限制 | 使用「方案 B 本地 Caddy 反代」，不走公網 Cloudflare 邊緣 |
| **HTTP 502** | Tunnel 無法連接局域網目標主機/端口 | 檢查局域網 IP 是否正確、防火牆是否放行、sub2api 是否正常監聽 |

---

## 📄 License
MIT
