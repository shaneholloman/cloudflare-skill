# Cloudflare Tunnel Expert

Expert skill for Cloudflare Tunnel (formerly Argo Tunnel) - secure, outbound-only connections between infrastructure and Cloudflare's global network.

## Core Concepts

### Architecture
- **Tunnel**: Persistent object routing traffic to DNS records
- **Connector**: `cloudflared` process establishing outbound connection to Cloudflare
- **Tunnel UUID**: Format `<UUID>.cfargotunnel.com` (e.g., `6ff42ae2-765d-4adf-8112-31c55c1551ef.cfargotunnel.com`)
- **Outbound-only**: No inbound ports required; all connections initiated from origin

### Tunnel Types
1. **Quick Tunnel**: Temporary, no config (`.trycloudflare.com` subdomain)
2. **Remotely-managed**: Config stored on Cloudflare (recommended)
3. **Locally-managed**: Config stored in local `cloudflared` directory

### Connection Models
1. **Published Applications**: Expose local services via public hostname
2. **Private Networks**: Connect networks (requires WARP client for access)

## Installation

### macOS
```bash
brew install cloudflared
```

### Linux
```bash
# Debian/Ubuntu
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg -i cloudflared-linux-amd64.deb

# RHEL/CentOS
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-x86_64.rpm
rpm -i cloudflared-linux-x86_64.rpm
```

### Docker
```bash
docker pull cloudflare/cloudflared:latest
```

## Quick Tunnel (No Config)

```bash
# Create temporary public URL for localhost:8080
cloudflared tunnel --url http://localhost:8080

# Output: https://random-subdomain.trycloudflare.com
```

## Locally-Managed Tunnel Setup

### 1. Authenticate
```bash
cloudflared tunnel login
# Opens browser to authenticate and downloads cert.pem to ~/.cloudflared/
```

### 2. Create Tunnel
```bash
cloudflared tunnel create my-tunnel
# Creates:
# - Tunnel UUID
# - Credentials file: ~/.cloudflared/<UUID>.json
```

### 3. Configure Tunnel

**Create config file**: `~/.cloudflared/config.yml`

#### Single Service
```yaml
tunnel: 6ff42ae2-765d-4adf-8112-31c55c1551ef
credentials-file: /root/.cloudflared/6ff42ae2-765d-4adf-8112-31c55c1551ef.json

ingress:
  - hostname: app.example.com
    service: http://localhost:8000
  - service: http_status:404  # Catch-all required
```

#### Multiple Services
```yaml
tunnel: 6ff42ae2-765d-4adf-8112-31c55c1551ef
credentials-file: /root/.cloudflared/6ff42ae2-765d-4adf-8112-31c55c1551ef.json

ingress:
  # HTTP service
  - hostname: app.example.com
    service: http://localhost:8000
  
  # HTTPS with self-signed cert
  - hostname: api.example.com
    service: https://localhost:8443
    originRequest:
      noTLSVerify: true
  
  # Static assets with path matching
  - hostname: static.example.com
    path: \.(jpg|png|css|js)$
    service: https://localhost:8001
  
  # Wildcard subdomain
  - hostname: "*.example.com"
    service: https://localhost:8002
  
  # SSH service
  - hostname: ssh.example.com
    service: ssh://localhost:22
  
  # RDP service
  - hostname: rdp.example.com
    service: rdp://localhost:3389
  
  # TCP service
  - hostname: tcp.example.com
    service: tcp://localhost:2222
  
  # Unix socket
  - hostname: socket.example.com
    service: unix:/home/production/app.sock
  
  # Test server
  - hostname: test.example.com
    service: hello_world
  
  # Catch-all
  - service: http_status:404
```

#### Private Network
```yaml
tunnel: 6ff42ae2-765d-4adf-8112-31c55c1551ef
credentials-file: /root/.cloudflared/6ff42ae2-765d-4adf-8112-31c55c1551ef.json

warp-routing:
  enabled: true
```

### 4. Route DNS
```bash
# Create CNAME record pointing to tunnel
cloudflared tunnel route dns my-tunnel app.example.com

# Or manually create CNAME:
# app.example.com -> <UUID>.cfargotunnel.com
```

### 5. Run Tunnel
```bash
# Run with config file
cloudflared tunnel run my-tunnel

# Run as service (Linux)
cloudflared service install
systemctl start cloudflared
systemctl enable cloudflared

# Run as service (macOS)
sudo cloudflared service install
sudo launchctl start com.cloudflare.cloudflared
```

## Origin Configuration Parameters

### Connection Settings
```yaml
originRequest:
  connectTimeout: 30s          # TCP connection timeout (default: 30s)
  tlsTimeout: 10s              # TLS handshake timeout (default: 10s)
  tcpKeepAlive: 30s            # TCP keepalive interval (default: 30s)
  noHappyEyeballs: false       # Disable RFC 6555 (default: false)
  keepAliveTimeout: 90s        # HTTP keepalive timeout (default: 90s)
  keepAliveConnections: 100    # Max idle connections (default: 100)
```

### TLS Settings
```yaml
originRequest:
  noTLSVerify: true            # Disable cert verification
  originServerName: "app.internal"  # Override SNI
  caPool: /path/to/ca.pem      # Custom CA cert
```

### HTTP Settings
```yaml
originRequest:
  disableChunkedEncoding: true  # Disable chunked transfer encoding
  httpHostHeader: "app.internal"  # Override Host header
  http2Origin: true             # Use HTTP/2 to origin
```

### Proxy Settings
```yaml
originRequest:
  proxyAddress: "socks5://proxy:1080"  # SOCKS5 proxy
  proxyPort: 1080
  proxyType: socks5
```

## Supported Service Types

| Protocol | Service Format | Use Case | Client Requirement |
|----------|----------------|----------|-------------------|
| HTTP | `http://localhost:8000` | Web apps | Browser |
| HTTPS | `https://localhost:8443` | Secure web apps | Browser |
| TCP | `tcp://localhost:2222` | Arbitrary TCP | `cloudflared access tcp` |
| SSH | `ssh://localhost:22` | SSH access | `cloudflared access ssh` |
| RDP | `rdp://localhost:3389` | Remote desktop | `cloudflared access rdp` |
| SMB | `smb://localhost:445` | File sharing | `cloudflared access` |
| UNIX | `unix:/path/to/socket` | Unix socket | Browser |
| UNIX+TLS | `unix+tls:/path/to/socket` | Secure Unix socket | Browser |
| HTTP_STATUS | `http_status:404` | Static response | N/A |
| HELLO_WORLD | `hello_world` | Test server | Browser |
| BASTION | `bastion` | Jump host | `cloudflared` |

## Ingress Rule Matching

Rules evaluated **top to bottom**. First match wins.

### Matching Logic
```yaml
ingress:
  # Exact hostname + path regex
  - hostname: static.example.com
    path: \.(jpg|png|css|js)$
    service: https://localhost:8001
  
  # Wildcard hostname (*.example.com matches alpha.example.com, beta.example.com)
  - hostname: "*.example.com"
    service: https://localhost:8002
  
  # Hostname only (all paths)
  - hostname: example.com
    service: https://localhost:8000
  
  # Path only (all hostnames)
  - path: /api/.*
    service: http://localhost:9000
  
  # Catch-all (required as last rule)
  - service: http_status:404
```

### Validation
```bash
# Validate config
cloudflared tunnel ingress validate

# Test rule matching
cloudflared tunnel ingress rule https://foo.example.com
# Output: Matched rule #3
```

## CLI Commands

### Tunnel Management
```bash
# List tunnels
cloudflared tunnel list

# Get tunnel info
cloudflared tunnel info my-tunnel

# Delete tunnel
cloudflared tunnel delete my-tunnel

# Cleanup connections
cloudflared tunnel cleanup my-tunnel
```

### DNS Routing
```bash
# Create DNS record
cloudflared tunnel route dns <TUNNEL> <HOSTNAME>

# List routes
cloudflared tunnel route list

# Delete route
cloudflared tunnel route delete <ROUTE_ID>
```

### Private Network Routes
```bash
# Route private IP/CIDR
cloudflared tunnel route ip add 10.0.0.0/8 my-tunnel

# Route specific IP
cloudflared tunnel route ip add 192.168.1.100/32 my-tunnel

# List IP routes
cloudflared tunnel route ip list

# Delete IP route
cloudflared tunnel route ip delete <ROUTE_ID>
```

## Docker Patterns

### Quick Tunnel
```dockerfile
FROM cloudflare/cloudflared:latest
CMD ["tunnel", "--url", "http://app:8080"]
```

### Named Tunnel
```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run --token ${TUNNEL_TOKEN}
    restart: unless-stopped
    networks:
      - app

  app:
    image: myapp:latest
    networks:
      - app

networks:
  app:
```

### With Config File
```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    volumes:
      - ./config.yml:/etc/cloudflared/config.yml:ro
      - ./credentials.json:/etc/cloudflared/credentials.json:ro
    command: tunnel --config /etc/cloudflared/config.yml run
    restart: unless-stopped
```

## Remotely-Managed Tunnels (Dashboard/API)

### Via Dashboard
1. Navigate to **Zero Trust** > **Networks** > **Tunnels**
2. Click **Create a tunnel**
3. Choose **Cloudflared**
4. Name tunnel, click **Save**
5. Copy install command (includes token)
6. Run on origin:
```bash
cloudflared service install <TOKEN>
```

### Via API (Unofficial - use Cloudflare API)
```bash
# Create tunnel
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/tunnels" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "my-tunnel",
    "tunnel_secret": "<base64-secret>"
  }'

# List tunnels
curl -X GET "https://api.cloudflare.com/client/v4/accounts/{account_id}/tunnels" \
  -H "Authorization: Bearer ${CF_API_TOKEN}"

# Update tunnel config
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/{account_id}/tunnels/{tunnel_id}/configurations" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{
    "config": {
      "ingress": [
        {"hostname": "app.example.com", "service": "http://localhost:8000"},
        {"service": "http_status:404"}
      ]
    }
  }'
```

## High Availability

### Multiple Replicas
```yaml
# Same config on multiple machines
# cloudflared automatically load balances
tunnel: 6ff42ae2-765d-4adf-8112-31c55c1551ef
credentials-file: /root/.cloudflared/6ff42ae2-765d-4adf-8112-31c55c1551ef.json

ingress:
  - hostname: app.example.com
    service: http://localhost:8000
  - service: http_status:404
```

Run on multiple servers:
```bash
# Server 1
cloudflared tunnel run my-tunnel

# Server 2 (same config)
cloudflared tunnel run my-tunnel

# Cloudflare load balances between replicas
```

### Zero-Downtime Config Updates
1. Start replica with new config
2. Wait for replica to connect
3. Stop old instance
4. Long-lived connections (WebSocket, SSH, UDP) will drop

## Common Use Cases

### 1. Web Application
```yaml
tunnel: ${TUNNEL_ID}
credentials-file: /etc/cloudflared/credentials.json

ingress:
  - hostname: myapp.example.com
    service: http://localhost:3000
  - service: http_status:404
```

### 2. SSH Access
```yaml
tunnel: ${TUNNEL_ID}
credentials-file: /etc/cloudflared/credentials.json

ingress:
  - hostname: ssh.example.com
    service: ssh://localhost:22
  - service: http_status:404
```

Client connects via:
```bash
cloudflared access ssh --hostname ssh.example.com
```

### 3. Development Environment
```bash
# Quick tunnel for local dev
cloudflared tunnel --url http://localhost:3000
```

### 4. Kubernetes Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
      - name: cloudflared
        image: cloudflare/cloudflared:latest
        args:
        - tunnel
        - --no-autoupdate
        - run
        - --token
        - $(TUNNEL_TOKEN)
        env:
        - name: TUNNEL_TOKEN
          valueFrom:
            secretKeyRef:
              name: tunnel-credentials
              key: token
```

### 5. Private Network Access
```yaml
# Enable WARP routing
tunnel: ${TUNNEL_ID}
credentials-file: /etc/cloudflared/credentials.json

warp-routing:
  enabled: true
```

```bash
# Route private CIDR
cloudflared tunnel route ip add 10.0.0.0/16 my-tunnel
```

Users with WARP client can access `10.0.0.0/16`.

### 6. gRPC Service
```yaml
tunnel: ${TUNNEL_ID}
credentials-file: /etc/cloudflared/credentials.json

ingress:
  - hostname: grpc.example.com
    service: http://localhost:50051
    originRequest:
      http2Origin: true
  - service: http_status:404
```

### 7. Multiple Environments
```yaml
tunnel: ${TUNNEL_ID}
credentials-file: /etc/cloudflared/credentials.json

ingress:
  - hostname: prod.example.com
    service: http://localhost:8001
  - hostname: staging.example.com
    service: http://localhost:8002
  - hostname: dev.example.com
    service: http://localhost:8003
  - service: http_status:404
```

## Troubleshooting

### Common Issues

#### 1. Error 1016 (Origin DNS Error)
```bash
# Check tunnel status
cloudflared tunnel info my-tunnel

# Verify tunnel is running
ps aux | grep cloudflared

# Check logs
journalctl -u cloudflared -n 100
```

#### 2. Certificate Errors
```yaml
originRequest:
  noTLSVerify: true  # Bypass cert validation (dev only)
  caPool: /path/to/ca.pem  # Use custom CA
```

#### 3. Connection Timeouts
```yaml
originRequest:
  connectTimeout: 60s
  tlsTimeout: 20s
  keepAliveTimeout: 120s
```

#### 4. Tunnel Not Starting
```bash
# Validate config
cloudflared tunnel ingress validate

# Check credentials file exists
ls -la ~/.cloudflared/*.json

# Verify tunnel exists
cloudflared tunnel list
```

### Debug Mode
```bash
# Run with debug logging
cloudflared tunnel --loglevel debug run my-tunnel

# Check specific rule
cloudflared tunnel ingress rule https://app.example.com
```

### Logs
```bash
# Linux systemd
journalctl -u cloudflared -f

# macOS
tail -f /Library/Logs/com.cloudflare.cloudflared.err.log

# Docker
docker logs -f <container_id>
```

## Best Practices

### Security
1. Use remotely-managed tunnels for prod (centralized control)
2. Enable Access policies for sensitive services
3. Rotate tunnel credentials regularly
4. Use `noTLSVerify: false` (verify certs)
5. Restrict `bastion` service type

### Performance
1. Run multiple replicas for HA
2. Place `cloudflared` close to origin (same network)
3. Use HTTP/2 for gRPC: `http2Origin: true`
4. Tune keepalive settings for long-lived connections
5. Monitor connection counts

### Configuration
1. Use environment variables for secrets
2. Version control config files
3. Validate config before deploying: `cloudflared tunnel ingress validate`
4. Test rules: `cloudflared tunnel ingress rule <URL>`
5. Document ingress rule order (first match wins)

### Operations
1. Monitor tunnel health via Cloudflare dashboard
2. Set up alerts for tunnel disconnections
3. Implement graceful shutdown for config updates
4. Keep `cloudflared` updated (versions supported for 1 year)
5. Use `--no-autoupdate` in prod; control updates manually

## Configuration File Locations

### Default Paths
```
~/.cloudflared/config.yml          # User config
/etc/cloudflared/config.yml        # System-wide config (Linux)
~/.cloudflared/cert.pem            # Auth cert (locally-managed)
~/.cloudflared/<UUID>.json         # Tunnel credentials
```

### Override
```bash
cloudflared tunnel --config /path/to/config.yml run my-tunnel
```

## Environment Variables

```bash
TUNNEL_TOKEN=<token>                    # Remotely-managed tunnel token
TUNNEL_ORIGIN_CERT=/path/to/cert.pem   # Override cert path
NO_AUTOUPDATE=true                      # Disable auto-updates
TUNNEL_LOGLEVEL=debug                   # Log level
```

## Migration Strategies

### From Ngrok
```yaml
# Ngrok: ngrok http 8000
# Cloudflare Tunnel:
ingress:
  - hostname: app.example.com
    service: http://localhost:8000
  - service: http_status:404
```

### From VPN
```yaml
# Replace VPN with private network routing
warp-routing:
  enabled: true
```

```bash
cloudflared tunnel route ip add 10.0.0.0/8 my-tunnel
```

Users install WARP client instead of VPN client.

## Key Differences from Other Tools

| Feature | Cloudflare Tunnel | Ngrok | VPN | Reverse Proxy |
|---------|-------------------|-------|-----|---------------|
| Public IP required | ❌ | ❌ | ✅ | ✅ |
| Outbound-only | ✅ | ✅ | ❌ | ❌ |
| Custom domains | ✅ | ✅ (paid) | N/A | ✅ |
| Built-in DDoS protection | ✅ | ❌ | ❌ | ❌ |
| Private network routing | ✅ | ❌ | ✅ | ❌ |
| Zero Trust integration | ✅ | ❌ | ❌ | ❌ |
| Free tier | ✅ | ✅ (limited) | N/A | N/A |

## Rate Limits & Quotas

- **Free tier**: Unlimited tunnels, unlimited traffic
- **Tunnel limit**: 1000 replicas per tunnel
- **DNS records**: Standard Cloudflare limits apply
- **Connection duration**: No hard limit (hours to days)

## Reference Architecture

```
┌─────────────┐
│  Internet   │
└──────┬──────┘
       │
       │ HTTPS/TCP/etc
       │
┌──────▼──────────────────────┐
│   Cloudflare Edge Network   │
│  (DDoS, WAF, Caching, etc)  │
└──────┬──────────────────────┘
       │
       │ Outbound-only tunnel
       │ (no inbound ports)
       │
┌──────▼──────┐     ┌─────────────┐
│ cloudflared ├────►│   Origin    │
│  (Replica 1)│     │  Service(s) │
└─────────────┘     └─────────────┘
       │
┌──────▼──────┐
│ cloudflared │
│  (Replica 2)│
└─────────────┘
```

## Official Resources

- **Docs**: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- **GitHub**: https://github.com/cloudflare/cloudflared
- **Docker Hub**: https://hub.docker.com/r/cloudflare/cloudflared
- **Release Notes**: https://github.com/cloudflare/cloudflared/blob/master/RELEASE_NOTES
