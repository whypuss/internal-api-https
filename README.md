# internal-api-https

> 專門為局域網內網 AI API（如 Sub2API、Ollama）提供穩定 HTTPS 穿透與持久化守護，解決 Mirasim、OpenCode 等 Agent 客戶端調用內網服務時的 HTTPS 要求、長流式（SSE）斷流與重開機失效問題。

---

## 🏗️ 標準架構鏈路

```text
Mirasim 客戶端 (Mac)
      │
      │ 1. 發送 HTTPS 請求 (mira.yourdomain.com)
      ▼
Cloudflare 邊緣節點 (全球 CDN 轉發)
      │
      │ 2. 經由 Cloudflare Tunnel 穿透隧道
      ▼
本機 cloudflared 守護進程 (LaunchAgent 常駐)
      │
      │ 3. 內網 HTTP 直轉 (保持 10 分鐘長連接 & 心跳)
      ▼
局域網 Sub2API 服務 (192.168.31.55:8090)
```

---

## 🚀 完整配置與落地步驟

### 1. 配置 Cloudflare Tunnel (`cloudflared.example.yml`)
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
      keepAliveConnections: 100    # 支持更多並發子請求
      tcpKeepAlive: 30s            # 30秒 TCP 心跳包，防路由器 NAT 超時清除
      http2Origin: false           # 避免與上游 chunked SSE 產生協議衝突
      disableChunkedEncoding: false
  - service: http_status:404
```

### 2. 綁定 DNS 路由
```bash
cloudflared tunnel route dns <你的_TUNNEL_UUID> mira.yourdomain.com
```

### 3. macOS 系統級持久化守護 (`launchagent.example.plist`)
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

啟用開機自啟：
```bash
launchctl load -w ~/Library/LaunchAgents/com.cloudflare.<tunnel-name>.plist
```

---

## 🔍 驗證與排錯清單

### 驗證連通性
```bash
curl -I https://mira.yourdomain.com/v1
```
返回 `HTTP/2 200 OK` 即代表鏈路正常。

### 常見問題速查
| 錯誤代碼 / 現象 | 核心原因 | 解決方法 |
| :--- | :--- | :--- |
| **ECONNRESET / 1033** | 本地 VPN (如 Karing) 劫持了 Cloudflare 節點 IP | 將該域名在 VPN 分流規則中設為「直連」，或重啟 Tunnel |
| **HTTP 530** | 本地 `cloudflared` 守護進程掉線 | 執行 `launchctl kickstart -k gui/$(id -u)/com.cloudflare.<tunnel-name>` 重啟 |
| **HTTP 502** | Tunnel 無法連接內網服務 | 檢查後端 IP/端口是否放行且正常運行 |
| **HTTP 524** | 單次請求思考/生成時間超過 100 秒 | 拆分提示詞或適當限制單次輸出長度 |

---

## 📄 License
MIT
