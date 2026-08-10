# Remote Access — Tailscale + Caddy + Cloudflared

Remote access uses a **dual-channel** approach with zero inbound ports:

1. **Tailscale Mesh VPN** — WireGuard-based, for direct device access
2. **Cloudflared tunnel** — outbound-only tunnel for public web access

No ports are forwarded on the U30Air or Pi5. All remote access is outbound.

## Channel 1: Tailscale Mesh VPN

### Overview

| Property | Value |
|----------|-------|
| Interface | tailscale0 |
| Tunnel IP | 100.x.x.x/32 |
| IPv6 | <your-tailscale-ipv6> |
| MagicDNS | edgegateway.<your-tailnet>.ts.net |
| Tailnet | <your-tailnet-name> |
| Relay | hkg (Hong Kong) |
| Version | 1.98.4+ |

### Peers

| Hostname | OS | Tunnel IP | Online |
|----------|----|-----------|--------|
| EdgeGateway (Pi5) | Linux | 100.x.x.x | always |
| Device-A | Windows | 100.x.x.x | yes |
| Device-B | Android | 100.x.x.x | no |
| Device-C | Android | 100.x.x.x | no |

### Capabilities

The Tailscale node has: funnel, https, ssh, file-sharing, is-admin, is-owner

### Installation (Pi5)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=10.42.0.0/24
```

Note: Advertising the IoT subnet (10.42.0.0/24) allows remote Tailscale peers
to reach IoT devices directly. Enable in Tailscale admin console: Machines →
Edit route settings → Approve 10.42.0.0/24.

### Verification

```bash
tailscale status
tailscale ping device-a.<your-tailnet>.ts.net
```

## Channel 2: Caddy Reverse Proxy

Caddy runs on the Pi5, bound to the Tailscale interface (100.x.x.x). It
provides HTTPS reverse proxy for internal services.

### Service Configuration

Two systemd services exist:
1. `caddy.service` — active, running (standard Caddy, uses local certs)
2. `caddy-cf.service` — activating/auto-restart (Caddy with Cloudflare DNS plugin)

The active service is `caddy.service`. The `caddy-cf.service` is for domains
using Cloudflare DNS-01 challenge but may be in a restart loop if the CF API
token is invalid.

### systemd override

```ini
# /etc/systemd/system/caddy.service.d/override.conf
[Unit]
After=network-online.target tailscaled.service
Wants=tailscaled.service

[Service]
EnvironmentFile=/etc/caddy/cf_token
ExecStart=
ExecStart=/usr/local/bin/caddy-cf run --environ --config /etc/caddy/Caddyfile
ExecReload=
ExecReload=/usr/local/bin/caddy-cf reload --config /etc/caddy/Caddyfile --force
Restart=on-failure
RestartSec=5
```

Key: Caddy starts AFTER tailscaled.service, ensuring the tailscale0 interface
exists before Caddy tries to bind to 100.x.x.x.

### Caddyfile

```
# Cloudflare DNS-01 TLS (public trusted certs)
ha.<your-domain>.com {
    bind 100.x.x.x
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
    reverse_proxy 127.0.0.1:8123
}

# Local self-signed certs (Tailscale internal only)
panel.<your-domain>.com {
    bind 100.x.x.x
    tls /etc/caddy/certs/agent.crt /etc/caddy/certs/agent.key
    reverse_proxy 127.0.0.1:5689
}

zte.<your-domain>.com {
    bind 100.x.x.x
    tls /etc/caddy/certs/agent.crt /etc/caddy/certs/agent.key
    reverse_proxy 192.168.0.1:2333
}
```

### Route Summary

| Domain | TLS | Backend | Service |
|--------|-----|---------|---------|
| ha.<your-domain>.com | CF DNS-01 | 127.0.0.1:8123 | Home Assistant |
| panel.<your-domain>.com | Local cert | 127.0.0.1:5689 | 1Panel |
| zte.<your-domain>.com | Local cert | 192.168.0.1:2333 | U30Air UFI-TOOLS |

Key design:
- `bind 100.x.x.x` - Caddy ONLY listens on Tailscale interface, not 0.0.0.0
- Add more routes for personal services (media, AI, etc.) as needed

### CF API Token

Stored in `/etc/caddy/cf_token`:
```
CF_API_TOKEN=your_cloudflare_api_token
```

Permissions: `chmod 600 /etc/caddy/cf_token`

### How to Replicate

```bash
# Install Caddy with Cloudflare DNS plugin
# Option A: Use xcaddy
xcaddy build --with github.com/caddy-dns/cloudflare
# Option B: Use pre-built caddy-cf binary

# Create cert directory
mkdir -p /etc/caddy/certs
# Generate self-signed cert for Tailscale-only domains
openssl req -x509 -newkey rsa:4096 -keyout /etc/caddy/certs/agent.key \
  -out /etc/caddy/certs/agent.crt -days 3650 -nodes \
  -subj "/CN=*.<your-domain>.com"

# Create CF token file
echo "CF_API_TOKEN=your_token" > /etc/caddy/cf_token
chmod 600 /etc/caddy/cf_token

# Write Caddyfile (see above)
# Create systemd override (see above)

# Enable and start
systemctl daemon-reload
systemctl enable caddy
systemctl start caddy
```

## Channel 3: Cloudflared Tunnel

Cloudflared provides public web access without opening any inbound ports. The
tunnel is **outbound-only** — Cloudflared connects to Cloudflare edge, not the
other way around.

### Container

```yaml
# /opt/1panel/apps/cloudflared/Cloudflared/docker-compose.yml
networks:
    1panel-network:
        external: true
services:
    cloudflared:
        command: tunnel --no-autoupdate run --token ${token}
        environment:
          - TUNNEL_EDGE_IP_VERSION=4
        container_name: ${CONTAINER_NAME}
        image: cloudflare/cloudflared:2026.7.3
        network_mode: host
        restart: always
```

### Config

```yaml
# /opt/1panel/apps/cloudflared/Cloudflared/config.yml
protocol: http2

ingress:
  - hostname: "*"
    service: http://localhost:4000   # Public-facing service (e.g., SearXNG)
```

Key settings:
- `protocol: http2` — forces HTTP/2 (not QUIC). See pitfalls.md for why.
- `TUNNEL_EDGE_IP_VERSION=4` — IPv4 only
- `network_mode: host` — uses host networking (can reach localhost services)
- Token mode — tunnel is configured in Cloudflare dashboard, not locally

### How It Works

```
Public user -> Cloudflare edge -> Cloudflared tunnel (outbound from Pi5)
  -> http://localhost:4000 (your public-facing service)
```

The tunnel is established by Cloudflared connecting OUT to Cloudflare's edge
servers. No inbound connections needed. Cloudflare routes public traffic
through the established tunnel.

### Clash Interference (Critical)

Cloudflared connects to Cloudflare edge on TCP:7844. Clash TProxy on U30Air
intercepts this TCP traffic and routes through proxy nodes, which block HTTP/2
long connections.

**Fix**: Add Clash direct rule:
```
DOMAIN-SUFFIX,argotunnel.com,DIRECT
```

This makes cloudflared's TCP connections bypass the proxy and go direct.

Alternative fixes (if Clash config cannot be modified):
- Use `--edge` flag with hardcoded CF edge IPs
- Use QUIC protocol (UDP:7844 bypasses Clash TProxy)
- But `protocol: http2` forces HTTP/2, so QUIC fallback is disabled

See `references/pitfalls.md` for full diagnosis.

### How to Replicate

1. Create a Cloudflare tunnel in the CF dashboard (Zero Trust → Networks → Tunnels)
2. Get the tunnel token
3. Deploy the Docker container (see compose above) with the token
4. Configure ingress rules in the CF dashboard (or config.yml for local config)
5. Add Clash direct rule for argotunnel.com on U30Air

## Security Model

| Access Method | Encryption | Authentication | Scope |
|---------------|------------|----------------|-------|
| Tailscale VPN | WireGuard | Device key + ACL | All Pi5 services |
| Caddy (Tailscale) | TLS (CF/local) | Tailscale identity | HTTP services |
| Cloudflared | TLS (CF edge) | CF Access (optional) | Public web services |
| SSH | SSH key | Key-based | Pi5 shell |
| ADB (U30Air) | None | Open (port 5555) | U30Air shell |

No service listens on 0.0.0.0 except:
- dnsmasq on 10.42.0.1:53 (IoT DNS)
- Docker host-network containers (bound to specific ports)
- SSH on 0.0.0.0:22 (should be restricted to Tailscale)

All public-facing access goes through Cloudflare (Cloudflared tunnel), which
provides DDoS protection and optional Access policies.
