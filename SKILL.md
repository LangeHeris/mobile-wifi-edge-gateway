---
name: mobile-wifi-edge-gateway
description: Use when deploying a 5G mobile WiFi + Pi5 software router edge gateway, configuring IoT network isolation, or setting up dual-tunnel remote access with Tailscale + Cloudflared.
---

# Mobile-WiFi Edge Gateway - 随身WiFi核心组网方案

## What This Is

A reproducible edge gateway architecture where a 5G mobile WiFi hotspot (ZTE
U30Air) serves as the WAN uplink, and a Raspberry Pi 5 acts as a software router
/ edge compute node. The Pi5 creates an isolated IoT network segment and
provides remote access via dual tunnels (Tailscale + Cloudflared).

This is not a single-device config - it's a **cross-device architecture** with
specific data flows, DNS chains, and security boundaries. All configs below are
real, verified configurations from a production deployment.

## Architecture Overview

```
Internet (5G NR)
    │
    ▼
┌──────────────────────────────────────────┐
│ U30Air 5G 随身WiFi (192.168.0.1/24)      │
│ • mihomo TProxy 透明路由 (port 7893)  │
│ • PicoClaw 网关                             │
│ • UFI-TOOLS 管理面板 (port 2333)          │
│ • 电池管理 (充电截止90%, 低电保护)         │
│ • Samba 文件共享                           │
│ • ADB 远程调试 (port 5555)                │
└──────────┬───────────────────────────────┘
           │ WiFi 5GHz (<your-wifi-ssid>, WPA2)
           │ wlan1 = <your-pi5-ip>/24 (DHCP client)
           ▼
┌──────────────────────────────────────────┐
│ EdgeGateway (Pi 5, <your-pi5-ip>)          │
│ • 路由核心: NAT + IP转发 + DHCP           │
│ • wlan0 AP = <your-iot-ssid> (10.42.0.1/24)     │
│ • Docker containers (host + bridge)         │
│ • AdGuard Home DNS 广告过滤               │
│ • Tailscale Mesh VPN (100.x.x.x/32)    │
│ • Caddy 反向代理 (绑tailscale0)            │
│ • Cloudflared 出站隧道                    │
└──────┬───────────────────────────────────┘
       │ WiFi 2.4GHz (<your-iot-ssid>, WPA2-CCMP)
       │ 10.42.0.0/24 (IoT 隔离网段)
       ▼
   IoT 设备群 (4台: 小米/智米/CUCO)
   Home Assistant 本地直控, 无公网依赖
```

## Key Design Decisions

1. **U30Air as WAN, not just hotspot** - mihomo (Clash Meta) TProxy on U30Air
   handles all proxy routing transparently. Pi5 traffic flows through U30Air
   then mihomo then proxy nodes then Internet.

2. **Dual WiFi on Pi5** - wlan1 (USB external, 5GHz) connects to U30Air as
   uplink client; wlan0 (onboard, 2.4GHz) serves as AP for IoT devices. eth0 is
   unused (no wired connection available).

3. **NetworkManager, not hostapd** - The Pi5 uses NetworkManager shared mode
   for the AP, which internally runs dnsmasq + iptables. The old hostapd config
   files exist but are inactive (systemd service disabled). Do NOT mix the two.

4. **IoT isolation** - IoT devices on 10.42.0.0/24, separated from the
   192.168.0.0/24 uplink network. Home Assistant controls locally, zero cloud
   dependency.

5. **Zero public ports** - Remote access exclusively via Tailscale Mesh VPN
   (WireGuard) and Cloudflared outbound tunnel. No inbound ports forwarded.

6. **DNS chain** - IoT clients then dnsmasq (NM-managed, port 53) then AdGuard
   Home (port 8053) then DoH upstreams. U30Air uses mihomo internal DNS with
   fake-IP.

## Prerequisites

### Hardware
- Raspberry Pi 5 (4GB+ RAM, NVMe SSD recommended)
- USB WiFi adapter (5GHz, 802.11ac) for wlan1 - verified: MediaTek MT7612U (0e8d:7612, driver mt76x2u). Alternatives: rtl8812au / rtl8821cu
- ZTE U30Air 5G随身WiFi (or similar 5G MiFi with mihomo support)
- MicroSD / NVMe for Pi5 OS

### Software (Pi5)
- Debian 13+ / Raspberry Pi OS (64-bit)
- NetworkManager (nmcli)
- Docker + Docker Compose
- Tailscale
- Caddy (with Cloudflare DNS plugin) or standard Caddy + local certs
- AdGuard Home (Docker container)

### Software (U30Air)
- minikano/ufi_tools custom stack (mihomo, PicoClaw, UFI-TOOLS)
- See references/u30air-integration.md for full software stack details

## Deployment Steps

### Phase 1: Pi5 Base System
1. Flash Debian 13 / Pi OS to NVMe/SD
2. Enable SSH, set hostname
3. Install Docker: `curl -fsSL https://get.docker.com | sh`
4. Install Tailscale: `curl -fsSL https://tailscale.com/install.sh | sh`
5. Install NetworkManager (if not preinstalled)

### Phase 2: Network Configuration
1. Configure wlan1 as WiFi client (uplink to U30Air)
2. Configure wlan0 as AP (IoT network)
3. Enable IP forwarding
4. Configure NAT/masquerade
See references/pi5-configuration.md

### Phase 3: U30Air Setup
1. Install minikano/ufi_tools stack
2. Configure mihomo (TProxy mode)
3. Set up boot scripts
4. Configure battery management
See references/u30air-integration.md and the zte-router-api skill

### Phase 4: DNS Layer
1. Deploy AdGuard Home (Docker, port 8053)
2. Configure dnsmasq upstream to AdGuard
3. Set up DoH upstreams
See references/dns-chain.md

### Phase 5: Docker Services
1. Deploy AdGuard Home and Cloudflared containers
2. (Optional) Deploy Home Assistant for IoT automation
3. Configure host vs bridge networking
4. Set up restart policies
See references/docker-services.md

### Phase 6: Remote Access
1. Join Tailscale network
2. Configure Caddy reverse proxy (bind to tailscale0)
3. Set up Cloudflared tunnel
See references/remote-access.md

### Phase 7: Verification
1. Verify routing: `traceroute 8.8.8.8` from IoT device
2. Verify DNS: `nslookup baidu.com 10.42.0.1` from IoT device
3. Verify remote access via Tailscale
4. Check all Docker containers healthy
See references/pitfalls.md for common issues

## Reference Files

| File | Content |
|------|---------|
| references/network-architecture.md | Full topology, data flow, IP addressing |
| references/pi5-configuration.md | NM configs, hostapd, dnsmasq, iptables, sysctl |
| references/u30air-integration.md | U30Air software stack, mihomo, boot scripts, battery |
| references/dns-chain.md | AdGuard + dnsmasq + mihomo DNS + pitfalls |
| references/remote-access.md | Tailscale + Caddy + Cloudflared |
| references/docker-services.md | Architecture-relevant containers, networking |
| references/adb-remote-access.md | ADB setup, usage, shell limitations |
| references/pitfalls.md | Known issues, dead loops, mihomo interference |

## Related Skills

- zte-router-api - U30Air device API, ADB access, goform endpoints, band locking
- raspberry-pi-network - Pi WiFi management (nmcli/wpa_supplicant)
- caddy-reverse-proxy - Caddy config on EdgeGateway
- adguard-home - AdGuard Home management
