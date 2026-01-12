# Cloudflare Spectrum Skill

## Overview

Cloudflare Spectrum provides security and acceleration for ANY TCP or UDP-based application. It's a global Layer 4 (L4) reverse proxy running on Cloudflare's edge nodes that routes MQTT, email, file transfer, version control, games, and more through Cloudflare to mask origins and protect from DDoS attacks.

**Key Characteristics:**
- Works at Layer 4 (Transport Layer) - proxies TCP/UDP traffic
- Does NOT terminate connections at application layer, but DOES terminate TCP/UDP sockets at L4
- Passes L4 payloads as-is without modification
- Available on all paid plans (Pro/Business: selected protocols; Enterprise: all TCP/UDP)
- Provides DDoS protection, Argo Smart Routing, TLS termination, proxy protocol support

## Core Concepts

### Application Types

**TCP/UDP Applications:**
- Proxy directly to origin
- Use for game servers, SSH, database connections, IoT protocols (MQTT), custom TCP/UDP services

**HTTP/HTTPS Applications:**
- Route through Cloudflare's full pipeline (CDN, Workers, Bot Management, WAF)
- Use when you need additional Cloudflare features beyond L4 proxying

### Origin Types

1. **Direct IP Origins** (`origin_direct`): Point to specific IP:port
2. **DNS Origins** (`origin_dns`): Use CNAME to resolve origin hostname
3. **Load Balancer Origins**: Distribute traffic across multiple origins

### IP Assignment

- Each application gets unique IPv4 + IPv6 addresses (or IPv6-only)
- IPs are anycast from all Cloudflare data centers (except China)
- IPs may change - always use DNS name to look up current IPs
- Supports BYOIP (Bring Your Own IP) and Static IP (Enterprise, API-only)

## Configuration Patterns

### Basic TCP Application with Direct IP Origin

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/22",
    "dns": {
      "type": "CNAME",
      "name": "ssh.example.com"
    },
    "origin_direct": ["tcp://192.0.2.1:22"],
    "proxy_protocol": "off",
    "ip_firewall": true,
    "tls": "off",
    "edge_ips": {
      "type": "dynamic",
      "connectivity": "all"
    },
    "traffic_type": "direct",
    "argo_smart_routing": true
  }'
```

### TCP Application with CNAME Origin

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "dns": {
      "type": "CNAME",
      "name": "game.example.com"
    },
    "protocol": "tcp/27015",
    "proxy_protocol": "v1",
    "tls": "off",
    "origin_dns": {
      "name": "origin-game.example.com",
      "ttl": 1200
    },
    "origin_port": 27015
  }'
```

### UDP Application with Simple Proxy Protocol

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "udp/53",
    "dns": {
      "type": "CNAME",
      "name": "dns.example.com"
    },
    "origin_direct": ["udp://10.0.0.1:53"],
    "proxy_protocol": "simple",
    "edge_ips": {
      "type": "dynamic",
      "connectivity": "ipv4"
    }
  }'
```

### Port Ranges

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/1000-2000",
    "dns": {
      "type": "CNAME",
      "name": "range.example.com"
    },
    "origin_direct": ["tcp://192.0.2.1:3000-4000"]
  }'
```

**Port range rules:**
- Number of edge ports MUST match number of origin ports
- Connections map by offset: `edge_port + N` → `origin_port + N`
- Example: `range.example.com:1005` → `origin:3005`

### TLS Termination Modes

```bash
# Flexible: TLS client→edge, plaintext edge→origin
{
  "tls": "flexible"
}

# Full: TLS client→edge and edge→origin (no cert validation)
{
  "tls": "full"
}

# Full (Strict): TLS end-to-end with strict cert validation
{
  "tls": "full_strict"
}

# Off: Passthrough (origin cert presented to client)
{
  "tls": "off"
}
```

### Load Balancer Origin

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "dns": {
      "type": "CNAME",
      "name": "lb.example.com"
    },
    "protocol": "tcp/443",
    "origin_dns": {
      "name": "lb-origin.example.com"
    },
    "origin_port": 443
  }'
```

## TypeScript SDK Patterns

### Using Cloudflare TypeScript SDK

```typescript
import Cloudflare from 'cloudflare';

const client = new Cloudflare({
  apiToken: process.env.CLOUDFLARE_API_TOKEN,
});

// Create Spectrum application
const app = await client.spectrum.apps.create({
  zone_id: 'YOUR_ZONE_ID',
  protocol: 'tcp/22',
  dns: {
    type: 'CNAME',
    name: 'ssh.example.com',
  },
  origin_direct: ['tcp://192.0.2.1:22'],
  proxy_protocol: 'off',
  tls: 'off',
});

// List all Spectrum applications
const apps = await client.spectrum.apps.list({
  zone_id: 'YOUR_ZONE_ID',
  per_page: 50,
});

// Get specific application
const appConfig = await client.spectrum.apps.get(
  'APP_ID',
  { zone_id: 'YOUR_ZONE_ID' }
);

// Update application
const updated = await client.spectrum.apps.update(
  'APP_ID',
  {
    zone_id: 'YOUR_ZONE_ID',
    protocol: 'tcp/22',
    dns: {
      type: 'CNAME',
      name: 'ssh.example.com',
    },
    origin_direct: ['tcp://192.0.2.100:22'],
    argo_smart_routing: true,
  }
);

// Delete application
await client.spectrum.apps.delete(
  'APP_ID',
  { zone_id: 'YOUR_ZONE_ID' }
);
```

### Analytics Queries

```typescript
// Get current analytics (last minute)
const current = await client.spectrum.analytics.aggregates.currents.get({
  zone_id: 'YOUR_ZONE_ID',
});

// Get analytics by time
const bytimes = await client.spectrum.analytics.events.bytimes.list({
  zone_id: 'YOUR_ZONE_ID',
  time_start: '2024-01-01T00:00:00Z',
  time_end: '2024-01-01T23:59:59Z',
  dimensions: ['event', 'appID'],
  metrics: ['count', 'bytesIngress', 'bytesEgress'],
});

// Get summary analytics
const summary = await client.spectrum.analytics.events.summaries.list({
  zone_id: 'YOUR_ZONE_ID',
  time_start: '2024-01-01T00:00:00Z',
  time_end: '2024-01-01T23:59:59Z',
});
```

## Proxy Protocol Implementation

### Proxy Protocol v1 (TCP)

Enable to pass client IP to origin. Prepends plain-text header to each TCP connection:

```
PROXY TCP4 192.0.2.0 192.0.2.255 42300 443\r\n
PROXY TCP6 2001:db8:: 2001:db8:ffff:ffff:ffff:ffff:ffff:ffff 42300 443\r\n
```

**Configuration:**
```json
{
  "proxy_protocol": "v1"
}
```

**Origin must parse header:**
```javascript
// Node.js example
socket.once('data', (data) => {
  const proxyLine = data.toString().split('\r\n')[0];
  const [, protocol, clientIP, proxyIP, clientPort, proxyPort] = proxyLine.split(' ');
  console.log(`Client: ${clientIP}:${clientPort}`);
});
```

### Proxy Protocol v2 (TCP/UDP)

Binary format. For TCP: prepends to connection. For UDP: prepends to first datagram.

```json
{
  "proxy_protocol": "v2"
}
```

### Simple Proxy Protocol (UDP)

Cloudflare custom protocol for UDP. 38-byte fixed header prepended to EACH datagram.

```json
{
  "proxy_protocol": "simple"
}
```

**Header structure:**
```c
struct {
    uint16_t magic;          // 0x56EC
    uint8_t  client_addr[16]; // IPv6 or IPv4-mapped IPv6
    uint8_t  proxy_addr[16];
    uint16_t client_port;
    uint16_t proxy_port;
};
```

**Python parser example:**
```python
import struct
import socket

def parse_spp_header(datagram):
    magic, client_addr, proxy_addr, client_port, proxy_port = struct.unpack(
        '!H16s16sHH',
        datagram[:38]
    )
    
    if magic != 0x56EC:
        raise ValueError("Invalid SPP magic number")
    
    client_ip = socket.inet_ntop(socket.AF_INET6, client_addr)
    payload = datagram[38:]
    
    return {
        'client_ip': client_ip,
        'client_port': client_port,
        'payload': payload
    }
```

**CRITICAL:** Origin MUST prepend same header to response packets for validation.

## Common Use Cases

### 1. SSH Server Protection

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/22",
    "dns": {"type": "CNAME", "name": "ssh.example.com"},
    "origin_direct": ["tcp://10.0.1.5:22"],
    "ip_firewall": true,
    "argo_smart_routing": true
  }'
```

**Benefits:**
- Hide origin IP from attackers
- DDoS protection at L3/L4
- Argo reduces latency
- IP firewall for additional access control

### 2. Game Server (e.g., Minecraft)

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/25565",
    "dns": {"type": "CNAME", "name": "mc.example.com"},
    "origin_direct": ["tcp://192.168.1.10:25565"],
    "proxy_protocol": "v1",
    "argo_smart_routing": true
  }'
```

**Benefits:**
- Protection from DDoS attacks common in gaming
- Argo improves player experience with optimized routing
- Proxy protocol preserves player IPs for logging

### 3. MQTT Broker (IoT)

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/1883",
    "dns": {"type": "CNAME", "name": "mqtt.example.com"},
    "origin_dns": {
      "name": "mqtt-origin.example.com",
      "ttl": 600
    },
    "origin_port": 1883,
    "tls": "full"
  }'
```

**With TLS (port 8883):**
```json
{
  "protocol": "tcp/8883",
  "tls": "full_strict"
}
```

### 4. Database Access (PostgreSQL/MySQL)

```bash
# PostgreSQL
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/5432",
    "dns": {"type": "CNAME", "name": "db.example.com"},
    "origin_direct": ["tcp://10.0.2.50:5432"],
    "proxy_protocol": "v2",
    "tls": "full_strict",
    "ip_firewall": true
  }'
```

**Best practices:**
- Use `tls: "full_strict"` for encryption
- Enable `ip_firewall` and configure IP Access Rules
- Consider using `proxy_protocol: "v2"` if database supports it
- Lock down origin to only accept Cloudflare IPs

### 5. SMTP Server

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/587",
    "dns": {"type": "CNAME", "name": "mail.example.com"},
    "origin_direct": ["tcp://192.168.1.20:587"],
    "proxy_protocol": "v1",
    "tls": "flexible"
  }'
```

**Important SMTP considerations:**
- Enable Proxy Protocol (mail servers need client IP for SPF/DKIM checks)
- Spectrum apps have NO reverse DNS entries (may cause delivery issues)
- MX record IP mismatch may cause rejections
- Best for submission (port 587), not direct delivery (port 25)

### 6. VPN/WireGuard (UDP)

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "udp/51820",
    "dns": {"type": "CNAME", "name": "vpn.example.com"},
    "origin_direct": ["udp://10.0.3.5:51820"],
    "edge_ips": {
      "type": "dynamic",
      "connectivity": "all"
    }
  }'
```

### 7. Redis with TLS

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/6379",
    "dns": {"type": "CNAME", "name": "redis.example.com"},
    "origin_direct": ["tcp://10.0.4.10:6379"],
    "tls": "full",
    "ip_firewall": true,
    "argo_smart_routing": true
  }'
```

## Security Best Practices

### 1. Origin Protection

```bash
# After configuring Spectrum, lock down origin firewall to only accept Cloudflare IPs
# Download current Cloudflare IP ranges
curl https://www.cloudflare.com/ips-v4
curl https://www.cloudflare.com/ips-v6

# Configure firewall (iptables example)
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  iptables -A INPUT -p tcp --dport 22 -s $ip -j ACCEPT
done
iptables -A INPUT -p tcp --dport 22 -j DROP
```

### 2. IP Access Rules

Enable per-application, then configure in dashboard or API:

```bash
# Block country
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/firewall/access_rules/rules" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "mode": "block",
    "configuration": {
      "target": "country",
      "value": "CN"
    },
    "notes": "Block China for Spectrum app"
  }'

# Allow specific IP
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/firewall/access_rules/rules" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "mode": "whitelist",
    "configuration": {
      "target": "ip",
      "value": "203.0.113.50"
    }
  }'
```

**Enable in Spectrum app:**
```json
{
  "ip_firewall": true
}
```

### 3. Replace Origin IPs

After moving to Spectrum, change origin IPs to prevent direct attacks bypassing Cloudflare.

### 4. DDoS Protection

Spectrum provides automatic L3/L4 DDoS protection:
- SYN/SYN-ACK reflection attack mitigation (Cloudflare absorbs impact)
- SYN cookie challenges via Linux networking stack
- Drops invalid/out-of-state TCP packets
- Drops packets for unspecified protocols or ports
- Network-layer DDoS Attack Protection managed ruleset (enabled by default)

**No configuration needed** - protection is automatic.

## Architecture Patterns

### Pattern 1: Multi-Region Failover with Load Balancer

```bash
# 1. Create load balancer with health checks
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/load_balancers/pools" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "name": "game-servers",
    "origins": [
      {"name": "us-east", "address": "us-east.origin.example.com", "enabled": true},
      {"name": "eu-west", "address": "eu-west.origin.example.com", "enabled": true}
    ],
    "monitor": "MONITOR_ID"
  }'

# 2. Create Spectrum app pointing to LB
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/25565",
    "dns": {"type": "CNAME", "name": "game.example.com"},
    "origin_dns": {
      "name": "game-lb.example.com"
    },
    "origin_port": 25565,
    "argo_smart_routing": true
  }'
```

### Pattern 2: Blue-Green Deployment

```typescript
// Switch origin during deployment
const updateOrigin = async (appId: string, newOriginIP: string) => {
  await client.spectrum.apps.update(appId, {
    zone_id: ZONE_ID,
    origin_direct: [`tcp://${newOriginIP}:8080`],
  });
};

// Deploy workflow
await updateOrigin(APP_ID, 'green-env-ip'); // Switch to green
// Test green environment
await updateOrigin(APP_ID, 'blue-env-ip'); // Rollback if needed
```

### Pattern 3: BYOIP for Static IPs

```bash
# Enterprise only, API-only configuration
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/apps" \
  --request POST \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  --json '{
    "protocol": "tcp/22",
    "dns": {"type": "CNAME", "name": "ssh.example.com"},
    "origin_direct": ["tcp://192.0.2.1:22"],
    "edge_ips": {
      "type": "static",
      "ips": ["203.0.113.10", "2001:db8::1"]
    }
  }'
```

### Pattern 4: Infrastructure as Code with Terraform

```hcl
resource "cloudflare_spectrum_application" "ssh" {
  zone_id  = var.zone_id
  protocol = "tcp/22"

  dns {
    type = "CNAME"
    name = "ssh.example.com"
  }

  origin_direct = ["tcp://192.0.2.1:22"]
  
  proxy_protocol  = "off"
  tls             = "off"
  ip_firewall     = true
  argo_smart_routing = true

  edge_ips {
    type         = "dynamic"
    connectivity = "all"
  }
}

resource "cloudflare_spectrum_application" "minecraft" {
  zone_id  = var.zone_id
  protocol = "tcp/25565"

  dns {
    type = "CNAME"
    name = "mc.example.com"
  }

  origin_dns {
    name = "mc-origin.example.com"
  }

  origin_port = 25565
  proxy_protocol = "v1"
  argo_smart_routing = true
}
```

### Pattern 5: Pulumi Infrastructure

```typescript
import * as cloudflare from "@pulumi/cloudflare";

const sshApp = new cloudflare.SpectrumApplication("ssh-server", {
  zoneId: zoneId,
  protocol: "tcp/22",
  dns: {
    type: "CNAME",
    name: "ssh.example.com",
  },
  originDirect: ["tcp://192.0.2.1:22"],
  proxyProtocol: "off",
  tls: "off",
  ipFirewall: true,
  argoSmartRouting: true,
  edgeIps: {
    type: "dynamic",
    connectivity: "all",
  },
});

export const sshAppId = sshApp.id;
export const sshDns = sshApp.dns.name;
```

## Configuration Reference

### Required Fields

```typescript
type SpectrumAppRequired = {
  protocol: string;           // "tcp/22", "udp/53", "tcp/1000-2000"
  dns: {
    type: "CNAME" | "ADDRESS";
    name: string;             // "app.example.com"
  };
  // ONE of:
  origin_direct?: string[];   // ["tcp://192.0.2.1:22"]
  origin_dns?: {              // OR use DNS origin
    name: string;
    ttl?: number;
  };
  origin_port?: number | string; // With origin_dns: 22 or "1000-2000"
};
```

### Optional Fields

```typescript
type SpectrumAppOptional = {
  proxy_protocol?: "off" | "v1" | "v2" | "simple";
  tls?: "off" | "flexible" | "full" | "full_strict";
  ip_firewall?: boolean;
  argo_smart_routing?: boolean;
  traffic_type?: "direct" | "http" | "https";
  edge_ips?: {
    type: "dynamic" | "static";
    connectivity?: "all" | "ipv4" | "ipv6";
    ips?: string[];  // For static type
  };
};
```

### API Endpoints

```
GET    /zones/{zone_id}/spectrum/apps                    # List apps
POST   /zones/{zone_id}/spectrum/apps                    # Create app
GET    /zones/{zone_id}/spectrum/apps/{app_id}           # Get app
PUT    /zones/{zone_id}/spectrum/apps/{app_id}           # Update app
DELETE /zones/{zone_id}/spectrum/apps/{app_id}           # Delete app

GET    /zones/{zone_id}/spectrum/analytics/aggregate/current
GET    /zones/{zone_id}/spectrum/analytics/events/bytime
GET    /zones/{zone_id}/spectrum/analytics/events/summary
```

### Supported Protocols

**Pro/Business Plans:**
- Selected protocols only (check Cloudflare docs for current list)

**Enterprise Plans:**
- All TCP ports (1-65535)
- All UDP ports (1-65535)
- Port ranges
- Custom protocols

## Troubleshooting

### Issue: Connection timeouts

**Diagnosis:**
```bash
# Test connectivity to Spectrum app
nc -zv app.example.com 22

# Check DNS resolution
dig app.example.com

# Verify origin accepts Cloudflare IPs
curl -v telnet://origin-ip:port
```

**Solutions:**
- Verify origin firewall allows Cloudflare IPs
- Check origin service is running and listening on correct port
- Ensure DNS record is CNAME (not A/AAAA unless using BYOIP)

### Issue: Client IP showing Cloudflare IP

**Solution:** Enable Proxy Protocol

```json
{
  "proxy_protocol": "v1"  // TCP: v1 or v2; UDP: simple
}
```

Ensure origin application parses proxy protocol headers.

### Issue: TLS errors

**Diagnosis:**
```bash
openssl s_client -connect app.example.com:443 -showcerts
```

**Solutions:**
- `tls: "flexible"` - Origin doesn't need TLS cert
- `tls: "full"` - Origin needs ANY valid cert
- `tls: "full_strict"` - Origin needs valid cert for origin hostname
- `tls: "off"` - Passthrough (origin cert presented to client)

### Issue: UDP packets dropped

**Check:**
- Verify protocol matches application type
- For Simple Proxy Protocol: ensure origin prepends header to responses
- Confirm no stateful firewall blocking "invalid" UDP return packets

### Issue: Analytics not showing traffic

**Wait:** Analytics update with ~1 minute delay. Check:

```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/spectrum/analytics/aggregate/current" \
  --header "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Performance Optimization

### 1. Enable Argo Smart Routing

```json
{
  "argo_smart_routing": true
}
```

Reduces latency by routing traffic through less congested paths. Costs extra but provides 30% average latency reduction.

### 2. Use IPv6

```json
{
  "edge_ips": {
    "connectivity": "ipv6"
  }
}
```

Benefits:
- Larger address space
- Better routing in some regions
- Reduced NAT overhead

### 3. Optimize Origin

- Use connection pooling at origin
- Tune TCP window sizes
- Enable TCP Fast Open if supported
- Use load balancer for horizontal scaling

### 4. Port Range Optimization

When running multiple similar services, use port ranges to reduce number of Spectrum apps:

```json
{
  "protocol": "tcp/3000-3099",
  "origin_direct": ["tcp://192.0.2.1:4000-4099"]
}
```

## Monitoring and Analytics

### Key Metrics

```typescript
type SpectrumMetrics = {
  bytesIngress: number;   // Bytes received from clients
  bytesEgress: number;    // Bytes sent to clients
  count: number;          // Number of events/connections
  dataIngress: number;    // Data received
  dataEgress: number;     // Data sent
};
```

### Monitoring Setup

```typescript
// Poll current stats every minute
setInterval(async () => {
  const stats = await client.spectrum.analytics.aggregates.currents.get({
    zone_id: ZONE_ID,
  });
  
  stats.result.forEach(app => {
    console.log(`App ${app.appID}:`, {
      ingress: app.bytesIngress,
      egress: app.bytesEgress,
    });
  });
}, 60000);

// Query historical data
const historical = await client.spectrum.analytics.events.bytimes.list({
  zone_id: ZONE_ID,
  time_start: new Date(Date.now() - 24*60*60*1000).toISOString(),
  time_end: new Date().toISOString(),
  dimensions: ['appID', 'coloName'],
  metrics: ['bytesIngress', 'bytesEgress', 'count'],
  time_delta: 'hour',
});
```

## Limitations and Considerations

### Plan Restrictions

| Feature | Pro/Business | Enterprise |
|---------|--------------|------------|
| Protocols | Selected only | All TCP/UDP |
| Port ranges | ❌ | ✅ |
| Proxy Protocol | Limited | Full (v1, v2, simple) |
| BYOIP | ❌ | ✅ |
| Static IPs | ❌ | ✅ |
| TLS customization | Default only | Full control |

### Technical Limits

- No reverse DNS for Spectrum IPs (impacts SMTP)
- Cannot use for Magic WAN egress with Proxy Protocol
- IPs may change (use DNS, not hardcoded IPs)
- No application-layer inspection (use HTTP/HTTPS app type for WAF)
- China data centers excluded from anycast

### Cost Considerations

- Spectrum charges based on usage (GB transferred)
- Argo Smart Routing is additional cost
- BYOIP requires Enterprise plan
- Static IPs require Enterprise plan

## Migration Guide

### From Direct Connection to Spectrum

1. **Create Spectrum app** (keep origin running on current IP)
2. **Test** using Spectrum DNS name
3. **Update client configurations** to use Spectrum DNS
4. **Monitor traffic** shifting to Spectrum
5. **Replace origin IPs** once traffic fully migrated
6. **Tighten origin firewall** to only accept Cloudflare IPs

### From Another Proxy Service

1. **Document current config** (ports, protocols, TLS settings)
2. **Create equivalent Spectrum apps**
3. **Test with subset of traffic** (use DNS/load balancer)
4. **Compare performance and cost**
5. **Gradually shift traffic**
6. **Decommission old service**

## Advanced Topics

### Custom TCP Flags and Options

Spectrum passes most TCP options transparently:
- Window scaling
- Timestamps
- SACK
- MSS (Maximum Segment Size)

Some TCP options may be modified by Cloudflare for optimization.

### UDP Stateful Tracking

Spectrum maintains UDP "connection" state based on:
- 5-tuple (src IP, src port, dst IP, dst port, protocol)
- Timeout after inactivity (typically 30 seconds)

Each new 5-tuple creates new "connection" for analytics.

### TLS Certificate Management

For `tls: "full_strict"` mode:
- Origin must have valid cert for `origin_dns.name` or `origin_direct` hostname
- Cloudflare validates cert against standard CA bundle
- Use Cloudflare Origin CA certificates for easy setup

### Combining with Other Cloudflare Services

```typescript
// Spectrum for TCP/UDP + Workers for management API
const app = await client.spectrum.apps.create({
  zone_id: ZONE_ID,
  protocol: 'tcp/25565',
  dns: { type: 'CNAME', name: 'game.example.com' },
  origin_dns: { name: 'origin.example.com' },
  origin_port: 25565,
});

// Worker at api.example.com for app management
// Separate from Spectrum (Spectrum is L4, Workers are L7)
```

## Debugging Tools

### Testing Proxy Protocol

```bash
# Send test connection and capture proxy header
nc -l 8080 | xxd  # In one terminal (origin)
nc app.example.com 8080  # In another terminal (client)

# Should see PROXY header prepended:
# PROXY TCP4 203.0.113.5 198.51.100.10 54321 8080\r\n
```

### Packet Capture

```bash
# At origin
tcpdump -i eth0 -w spectrum.pcap port 8080
wireshark spectrum.pcap

# Look for:
# - TCP handshake completion
# - Proxy protocol header (if enabled)
# - Application data
```

### DNS Verification

```bash
# Verify Spectrum app DNS resolves to Cloudflare IPs
dig +short app.example.com

# Compare to Cloudflare IP ranges
curl https://www.cloudflare.com/ips-v4
curl https://www.cloudflare.com/ips-v6
```

---

## Quick Reference

```bash
# Create basic TCP app
POST /zones/{zone}/spectrum/apps
{"protocol": "tcp/PORT", "dns": {"type": "CNAME", "name": "HOST"}, "origin_direct": ["tcp://IP:PORT"]}

# Enable proxy protocol (get client IP)
{"proxy_protocol": "v1"}  # TCP
{"proxy_protocol": "simple"}  # UDP

# Enable TLS termination
{"tls": "full_strict"}

# Enable DDoS + routing optimization
{"argo_smart_routing": true, "ip_firewall": true}

# Port ranges
{"protocol": "tcp/1000-2000", "origin_direct": ["tcp://IP:3000-4000"]}
```

**Remember:**
- Spectrum is Layer 4 (TCP/UDP) - not HTTP/HTTPS application layer
- Always use DNS name (IPs may change)
- Enable Proxy Protocol if origin needs client IP
- Lock down origin firewall to Cloudflare IPs only
- Enterprise required for full features (port ranges, BYOIP, static IPs)
