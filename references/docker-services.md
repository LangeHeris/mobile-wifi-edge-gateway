# Docker Services - Container Layout & Networking

The Pi5 runs Docker containers as part of the edge gateway. This document
covers only containers that are **part of the network architecture** -
services that handle routing, DNS, remote access, or IoT control. Other
personal containers (media, AI, databases, etc.) are deployment-specific
and omitted here.

## Architecture-Relevant Containers Only

This document lists only containers that are **part of the network architecture**:
DNS, remote access, and IoT control. Other personal containers (media, AI,
databases, dashboards, etc.) are deployment-specific - add your own as needed.

## Container Inventory

### Host Network

Host network containers share the Pi5's network namespace. They bind directly
to host ports. Use this for services that need multicast, mDNS, or low-latency
network access.

| Container | Image | Port | Purpose |
|-----------|-------|------|---------|
| AdGuardHome | adguard/adguardhome | 2000, 8053 | DNS ad blocking (upstream for dnsmasq) |
| Cloudflared | cloudflare/cloudflared | - | Outbound tunnel for public access |

Home Assistant (port 8123, host network) is optional - include it if you need
IoT automation with mDNS discovery on the 10.42.0.0/24 subnet.

## Service Details

### AdGuard Home (ports 2000, 8053)
- Network: host
- DNS ad blocking, port 8053 (not 53, which is dnsmasq)
- Config: /opt/1panel/apps/adguardhome/ADguardHome/data/conf/AdGuardHome.yaml
- See references/dns-chain.md for full DNS architecture

### Cloudflared
- Network: host
- Outbound tunnel to Cloudflare edge
- Config: /opt/1panel/apps/cloudflared/Cloudflared/config.yml
- See references/remote-access.md for tunnel setup

### Home Assistant (port 8123, optional)
- Network: host (needed for mDNS IoT discovery)
- Controls IoT devices on 10.42.0.0/24 via local LAN (no cloud)
- Include only if IoT automation is desired

## Network Modes

```
Docker networks:
  host          - architecture-relevant containers (direct host networking)
  bridge        - isolated containers with port mapping
  docker0       - default bridge (172.17.0.0/16), typically unused
```

Host network is used when:
- Service needs mDNS/multicast (Home Assistant IoT discovery)
- Service needs to bind to specific host interface (AdGuard on 8053)
- Service needs raw network access (Cloudflared outbound tunnel)

Bridge network is used when:
- Service is isolated and only needs port mapping (Homepage dashboard)

## Restart Policies

All containers use `restart: always`. This ensures services survive reboots
and crashes automatically.

## Management

Containers can be managed via 1Panel (port 5689), which provides:
- Web UI for container management
- Docker compose file management
- Firewall management (nftables chains: 1PANEL_BASIC, 1PANEL_FORWARD)

### 1Panel app directory structure

```
/opt/1panel/apps/
├── adguardhome/ADguardHome/
│   ├── data/conf/    - AdGuardHome.yaml
│   └── data/work/    - AdGuard runtime data
├── cloudflared/Cloudflared/
│   ├── config.yml    - tunnel config
│   └── docker-compose.yml
├── homeassistant/
└── ...
```

## s6-overlay Supervision

If running Hermes Agent on the Pi5, its container uses s6-overlay for process
supervision:
- Auto-restarts crashed processes
- Ordered startup/shutdown
- The gateway process is supervised by s6

## Verification

```bash
# List all containers
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

# Check container health
docker ps --filter "health=healthy" --format "{{.Names}}: healthy"
docker ps --filter "health=unhealthy" --format "{{.Names}}: UNHEALTHY"

# Check network modes
docker inspect $(docker ps -q) --format "{{.Name}}: {{.HostConfig.NetworkMode}}"

# Check specific service
docker logs --tail 20 HomeAssistant
docker logs --tail 20 AdGuardHome
```

## How to Replicate

1. Install Docker: `curl -fsSL https://get.docker.com | sh`
2. (Optional) Install 1Panel: `curl -sSL https://1panel.ca/install.sh | bash`
3. Deploy AdGuard Home (see references/dns-chain.md)
4. Deploy Home Assistant (host network for mDNS)
5. Deploy Cloudflared (see references/remote-access.md)
6. Set all restart policies to `always`
7. Add other personal containers as needed (media, AI, databases, etc.)
