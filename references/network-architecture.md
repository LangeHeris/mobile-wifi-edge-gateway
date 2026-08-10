# Network Architecture — Full Topology & Data Flow

## Physical Layer

```
Internet (5G NR, Band n41, 100MHz)
    │
    │ 5G NR radio
    ▼
┌─────────────────────────────────────────────────┐
│ ZTE U30Air (Android 13, arm64-v8a, 8-core)      │
│ Firmware: U30AirV1.0.0B17                        │
│ IP: 192.168.0.1/24 (WiFi AP mode)               │
│ WiFi: 5GHz + 2.4GHz dual-band                   │
│ Battery: USB-C powered, 73% (charge cap 90%)     │
│ SELinux: Permissive (enables boot-script root)   │
│                                                  │
│ Software stack (minikano/ufi_tools):             │
│   • Clash Meta (mihomo) - TProxy :7893, TUN      │
│   • UFI-TOOLS v4.1.1 - web panel :2333           │
│   • mosdns - DNS splitter (default off)           │
│   • f50_sms - SMS unlock APK                     │
│   • Samba - file share (root pw: <your_samba_password>)          │
│   • ADB daemon - :5555                           │
└──────────┬──────────────────────────────────────┘
           │
           │ WiFi 5GHz (SSID: <your-wifi-ssid>, WPA2-PSK)
           │ Channel: auto, VHT80
           │ Signal: -39 to -44 dBm (excellent)
           │ Link rate: 780 Mbps RX / 520-780 Mbps TX
           │
           ▼
┌─────────────────────────────────────────────────┐
│ Raspberry Pi 5 (EdgeGateway)                      │
│ Debian 13 (trixie), 8GB RAM, NVMe SSD            │
│                                                  │
│ wlan1 (USB MT76xx, 5GHz dual-stream)             │
│   Role: WiFi client (uplink to U30Air)            │
│   IP: <your-pi5-ip>/24 (DHCP from U30Air)        │
│   Gateway: 192.168.0.1                           │
│   DNS: 127.0.0.1 (ignore-auto-dns=yes)           │
│   Autoconnect priority: 100 (forever retry)      │
│                                                  │
│ wlan0 (onboard BCM, 2.4GHz)                      │
│   Role: WiFi AP (NetworkManager shared mode)      │
│   SSID: <your-iot-ssid>                                 │
│   IP: 10.42.0.1/24                               │
│   Channel: 6, hw_mode: g, WPA2-PSK/CCMP          │
│   DHCP: 10.42.0.10-254 (NM-managed dnsmasq)      │
│   Connected stations: 4 (IoT devices)             │
│                                                  │
│ tailscale0 (WireGuard)                           │
│   IP: 100.x.x.x/32                             │
│   MagicDNS: edgegateway.<your-tailnet>.ts.net                  │
│   Peers: Device-A (Win), Device-B, Device-C     │
│                                                  │
│ eth0: DOWN (unused, no wired connection)          │
│ docker0: 172.17.0.1/16 (bridge, DOWN)            │
│ br-5ba548806a6f: 172.18.0.1/16 (1panel-network)  │
│ br-6b603b6c07cf: 172.19.0.1/16 (1panel-network)  │
└──────────┬──────────────────────────────────────┘
           │
           │ WiFi 2.4GHz (<your-iot-ssid>, 10.42.0.0/24)
           │
           ▼
    IoT Devices (4 stations):
    • 小米电热毯 (MJ5) — unavailable
    • 智米空气净化器 (RMA3) — online, PM2.5=14
    • 小米风扇 (P43) — 12V supply
    • CUCO智能插座 ×2 (V3) — computer + water heater
    All controlled via Home Assistant (local LAN, no cloud)
```

## Logical Data Flow

### Outbound Traffic (IoT → Internet)

```
IoT device (10.42.0.x)
  → wlan0 AP (10.42.0.1)
  → Pi5 routing table (default via 192.168.0.1 dev wlan1)
  → NAT masquerade (10.42.0.0/24 → <your-pi5-ip>)
  → wlan1 (<your-pi5-ip>)
  → U30Air (192.168.0.1)
  → Clash TProxy (port 7893, tun0)
  → Proxy node (or direct, per Clash rules)
  → 5G NR → Internet
```

### DNS Resolution (IoT clients)

```
IoT device DNS query
  → 10.42.0.1:53 (dnsmasq, NM-managed)
  → upstream: 127.0.0.1#8053 (AdGuard Home on Pi5)
  → AdGuard filters (ad blocking)
  → DoH upstream (Aliyun/DNSPod/Cloudflare)
  → resolved IP returned

Note: U30Air DHCP cannot broadcast custom DNS (firmware read-only).
Pi5 wlan0 AP DHCP broadcasts 10.42.0.1 as DNS, so IoT devices
use Pi5 dnsmasq → AdGuard chain automatically.
```

### DNS Resolution (Pi5 itself)

```
Pi5 application DNS query
  → 127.0.0.1:53 (dnsmasq on wlan0)
  → 127.0.0.1#8053 (AdGuard Home)
  → DoH upstream

Note: wlan1 (uplink) has ignore-auto-dns=yes and dns=127.0.0.1
in NetworkManager, so U30Air's DHCP DNS is ignored.
```

### DNS Resolution (U30Air)

```
U30Air application DNS query
  → Clash DNS (port 1053, fake-IP mode)
  → fake-IP range: 198.18.0.0/16
  → proxy routing based on fake-IP mapping
```

### Remote Access Inbound

```
Remote user (Tailscale)
  → Tailscale Mesh VPN (WireGuard, encrypted)
  → tailscale0 (100.x.x.x)
  → Caddy reverse proxy (bind: 100.x.x.x, :80/:443)
  → backend services (127.0.0.1:PORT)

Remote user (Cloudflared, public domain)
  → Cloudflare edge → Cloudflared tunnel (outbound from Pi5)
  -> http://localhost:4000 (your public-facing service)
```

## IP Address Summary

| Device/Interface | IP | Subnet | Role |
|-----------------|-----|--------|------|
| U30Air | 192.168.0.1 | /24 | WiFi AP + 5G gateway |
| Pi5 wlan1 | <your-pi5-ip> | /24 | Uplink client |
| Pi5 wlan0 | 10.42.0.1 | /24 | IoT AP |
| Pi5 tailscale0 | 100.x.x.x | /32 | VPN tunnel |
| Pi5 docker0 | 172.17.0.1 | /16 | Docker default bridge |
| Pi5 br-5ba548806a6f | 172.18.0.1 | /16 | 1panel-network |
| Pi5 br-6b603b6c07cf | 172.19.0.1 | /16 | 1panel-network (2nd) |
| IoT devices | 10.42.0.10-254 | /24 | DHCP clients |
| U30Air (WAN) | 10.x.x.x | — | Carrier NAT (CGNAT) |
| Public IP | dynamic | — | Via 5G carrier |

## Routing Table (Pi5)

```
default via 192.168.0.1 dev wlan1 proto dhcp src <your-pi5-ip> metric 601
10.42.0.0/24 dev wlan0 proto kernel scope link src 10.42.0.1 metric 600
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.18.0.0/16 dev br-5ba548806a6f proto kernel scope link src 172.18.0.1
172.19.0.0/16 dev br-6b603b6c07cf proto kernel scope link src 172.19.0.1
192.168.0.0/24 dev wlan1 proto kernel scope link src <your-pi5-ip> metric 601
```

Key points:
- Default route goes through wlan1 → U30Air (192.168.0.1)
- 10.42.0.0/24 is local to wlan0 (IoT network)
- Docker bridges are isolated, no default route through them
- No static route for Tailscale (handled by tailscale0 interface)

## NAT / Masquerade Rules

The Pi5 uses nftables (via iptables-nft compatibility layer). Key masquerade
rules:

```
# IoT traffic masquerade (10.42.0.0/24 → wlan1 IP)
ip saddr 10.42.0.0/24 oifname "wlan1" masquerade  (implicit via NM shared)

# Docker bridge masquerade
ip saddr 172.17.0.0/16 oifname != "docker0" masquerade
ip saddr 172.18.0.0/16 oifname != "br-5ba548806a6f" masquerade
ip saddr 172.19.0.0/16 oifname != "br-6b603b6c07cf" masquerade

# FORWARD chain: ACCEPT from 10.42.0.0/24 (IoT → Internet)
ACCEPT  all  --  10.42.0.0/24  0.0.0.0/0
```

Note: NetworkManager's `shared` mode on wlan0 automatically configures:
- dnsmasq for DHCP/DNS on 10.42.0.1
- iptables/nftables masquerade for 10.42.0.0/24
- IP forwarding enabled via sysctl

## Firewall (1Panel managed)

1Panel adds its own nftables chains:
- `1PANEL_BASIC` — input filter
- `1PANEL_FORWARD` — forward filter
- `1PANEL_BASIC_AFTER` / `1PANEL_BASIC_BEFORE` — hooks

These coexist with Docker and NetworkManager rules. The FORWARD chain has
policy DROP but with ACCEPT rules for established connections and Docker traffic.
