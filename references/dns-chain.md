# DNS Chain — AdGuard Home + dnsmasq + Clash DNS

The DNS architecture spans three layers across two devices. Getting this right
is critical — misconfiguration causes silent failures, dead loops, and traffic
routing issues.

## DNS Flow Overview

```
IoT Device (10.42.0.x)
  │
  │ DNS query to 10.42.0.1:53
  ▼
dnsmasq (Pi5, NM-managed, port 53 on wlan0)
  │
  │ upstream: 127.0.0.1#8053
  ▼
AdGuard Home (Pi5, Docker container, port 8053)
  │
  │ filters ads/trackers
  │ upstream: DoH (Aliyun/DNSPod/Cloudflare)
  ▼
DoH upstream → resolved IP

--- Separate chain on U30Air ---

U30Air application
  │
  │ DNS query to Clash DNS (port 1053)
  ▼
Clash Meta DNS (fake-IP mode, 198.18.0.0/16)
  │
  │ maps domain → fake-IP for routing
  ▼
Clash routing rules → proxy node or direct
```

## Layer 1: dnsmasq (Pi5, NetworkManager-managed)

NetworkManager runs its own dnsmasq instance for the wlan0 AP. This is NOT the
system dnsmasq service (which is disabled).

### Configuration

The dnsmasq process is started by NM with these parameters:
```
/usr/sbin/dnsmasq
  --conf-file=/dev/null           # No config file (NM controls everything)
  --no-hosts
  --keep-in-foreground
  --bind-interfaces
  --except-interface=lo
  --listen-address=10.42.0.1      # Only on wlan0
  --dhcp-range=10.42.0.10,10.42.0.254,3600  # 1-hour leases
  --dhcp-leasefile=/var/lib/NetworkManager/dnsmasq-wlan0.leases
  --conf-dir=/etc/NetworkManager/dnsmasq-shared.d
```

### Custom overrides

Create files in `/etc/NetworkManager/dnsmasq-shared.d/`:

```bash
# Point upstream to AdGuard Home
echo "server=127.0.0.1#8053" > /etc/NetworkManager/dnsmasq-shared.d/upstream.conf

# DHCP options
echo "dhcp-option=3,10.42.0.1" > /etc/NetworkManager/dnsmasq-shared.d/dhcp.conf
echo "dhcp-option=6,10.42.0.1" >> /etc/NetworkManager/dnsmasq-shared.d/dhcp.conf

# Domain
echo "domain-needed" >> /etc/NetworkManager/dnsmasq-shared.d/upstream.conf
echo "bogus-priv" >> /etc/NetworkManager/dnsmasq-shared.d/upstream.conf
```

Apply: `nmcli connection down "<your-iot-ssid>" && nmcli connection up "<your-iot-ssid>"`

### Legacy /etc/dnsmasq.conf (NOT USED)

The file `/etc/dnsmasq.conf` exists with configuration but is NOT loaded because
the systemd service is disabled. Do NOT rely on it.

```
# /etc/dnsmasq.conf (INACTIVE — do not use)
interface=wlan0
port=53
dhcp-range=10.42.0.100,10.42.0.200,255.255.255.0,12h
dhcp-option=3,10.42.0.1
dhcp-option=6,10.42.0.1
server=127.0.0.1#8053
except-interface=lo
bind-interfaces
domain-needed
bogus-priv
```

## Layer 2: AdGuard Home (Pi5, Docker)

### Container

```
Container: ADguardHome
Image: adguard/adguardhome:v0.107.78
Network: host
Mounts:
  /opt/1panel/apps/adguardhome/ADguardHome/data/conf → /opt/adguardhome/conf
  /opt/1panel/apps/adguardhome/ADguardHome/data/work → /opt/adguardhome/work
```

### Ports

AdGuard Home uses port 8053 for DNS (not 53, which is privileged and used by
dnsmasq on 10.42.0.1:53).

- Web UI: port 2000
- DNS: port 8053

### Initial Setup

If AdGuard Home is not yet initialized:

```bash
# Initialize via API (no browser needed)
curl -X POST http://localhost:2000/control/install/configure \
  -H "Content-Type: application/json" \
  -d '{
    "web": {"ip": "0.0.0.0", "port": 2000},
    "dns": {"ip": "0.0.0.0", "port": 8053},
    "username": "admin",
    "password": "YOUR_PASSWORD"
  }'
```

### Upstream DNS (DoH)

AdGuard Home should be configured with DoH (DNS over HTTPS) upstreams for
privacy and reliability:

Recommended upstreams (China-optimized):
- `https://dns.alidns.com/dns-query` (Aliyun)
- `https://doh.pub/dns-query` (DNSPod/Tencent)
- `https://dns.cloudflare.com/dns-query` (Cloudflare — for international)

Configure via AdGuard Home web UI: Settings → DNS Settings → Upstream DNS servers

### Bootstrap DNS

Set bootstrap DNS to plain IP addresses (no DoH, for initial resolution):
- `223.5.5.5` (Aliyun)
- `119.29.29.29` (DNSPod)
- `9.9.9.9` (Quad9)

## Layer 3: Clash DNS (U30Air)

Clash Meta on U30Air has its own DNS resolver with fake-IP mode.

### Configuration (in Clash config.yaml)

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - "*.lan"
    - "*.local"
    - "*.localhost"
  nameserver:
    - 223.5.5.5
    - 119.29.29.29
  fallback:
    - https://dns.cloudflare.com/dns-query
    - tls://8.8.8.8:853
```

### How fake-IP Works

1. Device queries domain → Clash DNS returns fake-IP (198.18.0.x)
2. Device connects to fake-IP
3. Clash intercepts connection, looks up original domain from fake-IP mapping
4. Clash routes based on domain rules (not IP)
5. This allows domain-based routing even for non-proxy traffic

### Impact on Pi5 (Verified)

Pi5's DNS chain (dnsmasq -> AdGuard -> DoH upstreams) resolves to **real IPs**,
completely independent of Clash fake-IP. Verified by querying baidu.com through
different DNS servers:

| DNS Server | Path | Result | Fake-IP? |
|---|---|---|---|
| 127.0.0.1:53 | dnsmasq -> AdGuard -> DoH | 124.237.177.164 etc. | No |
| <your-pi5-ip>:53 | Pi5 LAN (same as above) | 124.237.177.164 etc. | No |
| 10.42.0.1:53 | AP gateway (IoT path) | 124.237.177.164 etc. | No |
| 8.8.8.8:53 | Direct query -> Clash TProxy intercept | 198.18.1.55 | Yes |
| 1.1.1.1:53 | Direct query -> Clash TProxy intercept | 198.18.1.55 | Yes |

**Conclusion**: fake-IP only appears when directly querying external DNS
servers (8.8.8.8, 1.1.1.1) from Pi5. The TProxy on U30Air intercepts these
queries and returns fake-IPs. But Pi5's own DNS resolution (via AdGuard)
always gets real IPs because:

1. `ipv4.dns = 127.0.0.1` on wlan1 (ignores U30Air DHCP DNS)
2. `ipv4.ignore-auto-dns = yes` (prevents override)
3. AdGuard resolves via DoH to upstream resolvers, getting real IPs
4. Clash TProxy then handles routing based on the real destination IP

This means ad-blocking and DNS filtering work correctly on Pi5 and all IoT
devices connected to the AP. fake-IP is irrelevant to the Pi5 DNS chain.

## U30Air DHCP DNS Limitation (Critical)

The U30Air firmware does NOT allow customizing DHCP-assigned DNS:

- `dhcp_dns1` / `dhcp_dns2` goform fields are **read-only**
- All goformId attempts fail: DHCP_SETTING, SET_NETWORK, BASIC_SETTING, SET_NV
- No web UI option to change DHCP DNS

This means devices connecting directly to U30Air WiFi (<your-wifi-ssid>) get the
U30Air's default DNS (Clash DNS), NOT AdGuard Home.

**Workaround on Pi5**: NetworkManager wlan1 connection has:
```ini
ipv4.dns = 127.0.0.1
ipv4.ignore-auto-dns = yes
```
This makes Pi5 ignore the U30Air's DHCP DNS and use local AdGuard Home.

**For direct U30Air clients** (phones/tablets connected to <your-wifi-ssid>): they will
use Clash DNS (fake-IP). This is usually fine because Clash handles routing
correctly for direct clients. But ad blocking won't work for these devices.

## DNS Dead Loop Trap (Removed)

Previously, iptables PREROUTING REDIRECT rules were used to force all DNS (port
53) traffic to AdGuard Home port 8053:

```bash
# THESE WERE REMOVED — caused dead loop
iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-port 8053
iptables -t nat -A PREROUTING -p tcp --dport 53 -j REDIRECT --to-port 8053
```

**The dead loop**: AdGuard queries upstream DNS → upstream DNS query hits
PREROUTING → redirected back to AdGuard → infinite loop → DNS timeout.

**Fix**: These rules were deleted. If you need per-interface DNS redirect, use
`-i wlan0` to limit scope:

```bash
# Safe version (only for wlan0/IoT traffic)
iptables -t nat -A PREROUTING -i wlan0 -p udp --dport 53 -j REDIRECT --to-port 8053
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 53 -j REDIRECT --to-port 8053
```

But with NetworkManager shared mode, this is NOT needed — NM's dnsmasq already
handles DNS on wlan0:53 and forwards to AdGuard on 8053.

## Verification

```bash
# From Pi5
dig @10.42.0.1 baidu.com          # Should resolve via dnsmasq → AdGuard
dig @127.0.0.1 -p 8053 google.com # AdGuard directly
nslookup baidu.com 10.42.0.1      # From IoT device

# Check AdGuard is running
docker ps | grep -i adguard
curl -s http://localhost:2000/ | head -5

# Check dnsmasq
pgrep -a dnsmasq

# Check no dead-loop rules exist
sudo iptables -t nat -L PREROUTING -n | grep 8053
# Should return nothing (rules removed)
```
