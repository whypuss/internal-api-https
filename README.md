# internal-api-https

> Reliable, timeout-resistant HTTPS proxy and Cloudflare Tunnel configurations for internal LAN AI APIs (Sub2API, Ollama, LiteLLM) accessed by Agent clients (Mirasim, OpenCode, Claude Code).

---

## 📖 Background & Problems Solved

When connecting agent clients like **Mirasim** to local network AI proxies like **Sub2API** (`http://192.168.x.x:8090`), users frequently encounter:

1. **Client Rejection of Plain HTTP**: Mirasim strictly expects `https://` URLs with recognized hostnames.
2. **Cloudflare 100s Gateway Timeout (HTTP 524)**: Long multi-step tasks or complex reasoning exceed 100 seconds, causing public Cloudflare CDN proxies to terminate the connection.
3. **SSE Streaming Dropouts**: Unoptimized tunnel parameters drop silent chunks during long inference gaps.
4. **VPN / DNS Collisions (ECONNRESET / 1033)**: TUN-mode proxies (Karing, Clash, Sing-Box) hijack DNS queries to Fake-IPs and drop or reset Cloudflare edge handshakes.
5. **Process Lifecycle**: Missing daemonization causing tunnels to quit after user logout or reboot.

This repository provides reusable configurations and scripts for two proven patterns:
- **Pattern A (Public/Hybrid via Cloudflare Tunnel)**: Optimized for long-lived keep-alives and streaming with system launchd persistence.
- **Pattern B (Pure Local HTTPS via Caddy)**: Completely eliminates Cloudflare's 100s limit by terminating TLS locally and routing directly within the LAN.

---

## 🚀 Setup Patterns

### Pattern A: Optimized Cloudflare Tunnel (with macOS Persistence)

#### 1. Ingress Configuration (`config.example.yml`)
Place this into `~/.cloudflared/<tunnel-name>.yml`:

```yaml
tunnel: <YOUR_TUNNEL_UUID>
credentials-file: /Users/<USER>/.cloudflared/<YOUR_TUNNEL_UUID>.json

ingress:
  - hostname: mira.yourdomain.com
    service: http://192.168.31.55:8090
    originRequest:
      connectTimeout: 30s
      keepAliveTimeout: 10m        # Keeps upstream SSE connection open for 10 minutes
      keepAliveConnections: 100    # Increases concurrent connection capacity
      tcpKeepAlive: 30s            # Heartbeat packet prevents NAT timeout
      http2Origin: false           # Prevents protocol conflict with upstream chunked SSE
      disableChunkedEncoding: false
  - service: http_status:404
```

#### 2. Route DNS via CLI
```bash
cloudflared tunnel route dns <YOUR_TUNNEL_UUID> mira.yourdomain.com
```

#### 3. macOS LaunchAgent Daemon (`launchagent.example.plist`)
Copy to `~/Library/LaunchAgents/com.cloudflare.<tunnel-name>.plist`:

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
        <string>exec /opt/homebrew/bin/cloudflared tunnel --protocol http2 --config /Users/<USER>/.cloudflared/<tunnel-name>.yml run</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>NetworkState</key>
        <true/>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/<USER>/.cloudflared/<tunnel-name>.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/<USER>/.cloudflared/<tunnel-name>.log</string>
</dict>
</plist>
```

Activate persistence:
```bash
launchctl load -w ~/Library/LaunchAgents/com.cloudflare.<tunnel-name>.plist
```

---

### Pattern B: Local Zero-Timeout HTTPS (Caddy Termination)

To bypass Cloudflare CDN's 100s edge timeout entirely:

#### 1. Caddyfile (`Caddyfile.example`)
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

#### 2. Run Caddy & Trust Certificate
```bash
sudo caddy start --config ./Caddyfile
sudo caddy trust
```

#### 3. Route Domain to Localhost
In `/etc/hosts`:
```text
127.0.0.1 mira.yourdomain.com
```

---

## 🛠️ Diagnostics & Troubleshooting

| Error Code / Symptom | Root Cause | Solution |
| :--- | :--- | :--- |
| **ECONNRESET / 1033** | VPN / Proxy (e.g. Karing) DNS hijacking or handshake drop | Bypass the domain in VPN routing rules, or restart tunnel |
| **HTTP 530** | Cloudflare cannot connect to local tunnel daemon | Verify local `cloudflared` daemon is active and registered |
| **HTTP 524** | Cloudflare Edge 100-second proxy timeout | Use Pattern B (Local HTTPS reverse proxy) |
| **HTTP 502** | Tunnel unable to reach internal LAN target | Check LAN target IP/Port connectivity and network route |

---

## 📄 License
MIT
