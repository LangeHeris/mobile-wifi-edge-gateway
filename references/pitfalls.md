# Pitfalls & Known Issues

Real problems encountered during deployment and operation. Each has a
diagnosed root cause and verified fix.

## 1. DNS Dead Loop (iptables REDIRECT + AdGuard)

### Symptom
After stopping Mihomo proxy, all LAN DNS queries timeout. AdGuard logs show
DoH upstream `context deadline exceeded`. Direct queries to public DNS
(114.114.114.114, 223.5.5.5) return Mihomo fake-IP (198.18.0.x).

### Root Cause
iptables PREROUTING REDIRECT rules created a dead loop:
```
AdGuard → queries upstream DNS 223.5.5.5:53
        → iptables PREROUTING REDIRECT → AdGuard:8053
        → returns cached Mihomo fake-IP
        → or DoH bootstrap loop (also hijacked)
```

The rules had no interface filter, so they matched ALL interfaces including
loopback and the AdGuard's own upstream queries.

### Fix
Delete the unfiltered rules. If per-interface redirect is needed, use `-i`:
```bash
# Delete bad rules
iptables -t nat -D PREROUTING -p udp --dport 53 -j REDIRECT --to-port 8053
iptables -t nat -D PREROUTING -p tcp --dport 53 -j REDIRECT --to-port 8053
iptables-save > /etc/iptables/rules.v4

# Safe version (if needed — only for wlan0/IoT traffic)
iptables -t nat -A PREROUTING -i wlan0 -p udp --dport 53 -j REDIRECT --to-port 8053
```

### Prevention
- ALWAYS use `-i <interface>` with iptables REDIRECT rules
- Check with `-C` before adding to avoid duplicates
- With NetworkManager shared mode, this redirect is NOT needed at all

## 2. Cloudflared + Clash TProxy Interference

### Symptom
Cloudflared HTTP/2 connections to Cloudflare edge (TCP:7844) fail. Tunnel
auto-degrades to QUIC (UDP:7844). When `protocol: http2` is forced, tunnel
fails fatally.

### Root Cause
Clash TProxy on U30Air intercepts ALL outbound TCP traffic, including
cloudflared's TCP:7844 connections. Proxy nodes block or don't support HTTP/2
long connections.

Additionally, Clash fake-IP DNS hijacks `argotunnel.com` resolution:
- AdGuard resolves argotunnel.com → real CF edge IP
- But if DNS goes through Clash, it returns fake-IP (198.18.0.10/198.18.0.11)
- TLS handshake fails with EOF

### Fix (Recommended)
Add Clash direct rule for argotunnel.com:
```
DOMAIN-SUFFIX,argotunnel.com,DIRECT
```
This makes cloudflared TCP connections bypass the proxy.

### Alternative Fixes
1. Configure cloudflared with `--edge 198.41.192.1:7844,198.41.192.2:7844`
   (hardcode real CF edge IPs, skip DNS)
2. Use `edge-ip-version: 4` in config.yml
3. Switch to QUIC protocol (UDP bypasses Clash TProxy) — but may be unreliable
4. Replace cloudflared with Tailscale Funnel

### Note
198.18.0.10 and 198.18.0.11 are Clash fake-IP addresses, NOT real Cloudflare
edge IPs. Real CF edge IPs are in the 198.41.192.x range.

## 3. U30Air DHCP DNS Read-Only

### Symptom
Cannot set custom DNS for devices connecting to U30Air WiFi. All goformId
attempts to change `dhcp_dns1`/`dhcp_dns2` fail silently.

### Root Cause
ZTE U30Air firmware makes DHCP DNS settings read-only. Confirmed by testing
all known goformId endpoints: DHCP_SETTING, SET_NETWORK, BASIC_SETTING, SET_NV.

### Fix (Workaround)
On Pi5, configure NetworkManager to ignore DHCP DNS:
```ini
# wlan1 connection (<your-wifi-ssid> 1)
ipv4.dns = 127.0.0.1
ipv4.ignore-auto-dns = yes
```
This makes Pi5 use local AdGuard Home instead of U30Air's DNS.

For direct U30Air clients (phones/tablets on <your-wifi-ssid>), they will use Clash
DNS (fake-IP). This is acceptable for most use cases but ad blocking won't
work for these devices.

## 4. NetworkManager vs hostapd Conflict

### Symptom
WiFi AP on wlan0 doesn't work, or hostapd fails to start, or IP address
disappears after reboot.

### Root Cause
Both NetworkManager and hostapd try to manage wlan0. If hostapd.service is
enabled, it conflicts with NM's AP mode.

### Fix
Disable hostapd completely:
```bash
systemctl stop hostapd
systemctl disable hostapd
```

Remove the systemd override that adds IP address:
```bash
rm /etc/systemd/system/hostapd.service.d/ipfix.conf
```

Use NetworkManager's `shared` mode exclusively for the AP.

### Legacy Files to Clean Up
- `/etc/hostapd/hostapd.conf` — wrong interface (wlan1 instead of wlan0)
- `/etc/hostapd/hostapd-wlan0ap.conf` — duplicate with same issue
- `/etc/default/hostapd` — points to wrong config file
- `/etc/dhcpcd.conf` — different subnet (10.3.141.1 vs current 10.42.0.1)

These files are harmless if hostapd is disabled, but should be removed to
avoid confusion during future troubleshooting.

## 5. caddy-cf Service Restart Loop

### Symptom
`caddy-cf.service` is in `activating (auto-restart)` state, constantly
failing and restarting.

### Root Cause
The caddy-cf binary requires a valid Cloudflare API token for DNS-01
challenge. If `/etc/caddy/cf_token` contains an expired or invalid token,
caddy-cf exits with status 1.

### Fix
1. Verify CF API token: `curl -X GET "https://api.cloudflare.com/client/v4/user/tokens/verify" -H "Authorization: Bearer YOUR_TOKEN"`
2. Update token: `echo "CF_API_TOKEN=new_token" > /etc/caddy/cf_token`
3. Restart: `systemctl restart caddy-cf`

Note: The standard `caddy.service` (without CF DNS plugin) runs fine
independently for domains using local certs.

## 6. DeepSeek API Empty tool_calls Rejection

### Symptom
Hermes Agent gateway exits unexpectedly. s6-supervise restarts it.

### Root Cause
DeepSeek API rejects empty `tool_calls: []` arrays with HTTP 400:
```
Invalid 'messages[53].tool_calls': empty array.
Expected an array with minimum length 1, but got an empty array instead.
```
OpenAI silently accepts empty arrays, but DeepSeek has stricter validation.

### Fix
This is a gateway-level issue. The Hermes Agent gateway should strip empty
`tool_calls` arrays before sending to DeepSeek. s6-supervise auto-recovery
prevents prolonged downtime.

## 7. Clash fake-ip DNS Hijacking

### Symptom
DNS queries from Pi5 return 198.18.0.x addresses instead of real IPs,
causing connection failures for services that bypass Clash.

### Root Cause
Clash DNS on U30Air uses fake-IP mode (198.18.0.0/16). If Pi5 DNS queries
reach Clash DNS (port 1053 on U30Air), they get fake-IP responses.

### Fix
Pi5 wlan1 NetworkManager connection has:
```ini
ipv4.dns = 127.0.0.1
ipv4.ignore-auto-dns = yes
```
This ensures Pi5 uses local AdGuard Home, not U30Air's Clash DNS.

## 8. Tailscale MTU Issues

### Symptom
Tailscale connections work for small packets but fail for large data
transfers (e.g., SSH terminal works but file transfer hangs).

### Root Cause
Tailscale uses MTU 1280 (WireGuard overhead). Some path MTU discovery
packets are dropped by Clash TProxy, causing fragmentation issues.

### Fix
Usually works fine with default settings. If issues occur, check:
```bash
# Verify MTU
ip link show tailscale0
# Should show mtu 1280

# Test connectivity
tailscale ping <peer>
tailscale status
```

## 9. Docker Bridge Network Duplication

### Symptom
Two 1panel-network bridges exist (172.18.0.0/16 and 172.19.0.0/16), causing
container communication issues.

### Root Cause
1Panel may create multiple bridge networks during installation/updates. Docker
assigns different subnet ranges to each.

### Fix
Check which network your containers are on:
```bash
docker network ls
docker inspect <container> --format "{{.NetworkSettings.Networks}}"
```

If containers need to communicate, ensure they're on the same bridge, or
use the host network for services that need cross-container access.

## 10. 5G Signal Degradation

### Symptom
5G signal drops, RSRP worsens, throughput decreases.

### Fix
- Lock to specific band (n41 recommended for most carriers in China)
- Use UFI-TOOLS panel (port 2333) → Lock Bands
- Check signal: RSRP should be > -100 dBm, SINR > 10 dB
- Reposition U30Air for better signal (near window, away from metal)

Current production values: RSRP -84 dBm, SINR 17 dB (excellent).

## Verification Checklist

After deployment, verify:

```bash
# 1. Routing
traceroute 8.8.8.8                          # From IoT device
ip route                                    # On Pi5

# 2. DNS
dig @10.42.0.1 baidu.com                    # From IoT device
dig @127.0.0.1 -p 8053 google.com           # AdGuard direct

# 3. NAT/Masquerade
sudo nft list ruleset | grep masquerade     # Check rules
sudo iptables -L FORWARD -n                 # Check FORWARD chain

# 4. Docker
docker ps --format "{{.Names}}: {{.Status}}" # All containers up

# 5. Remote access
tailscale status                             # All peers visible
curl -k https://agent.<your-domain>.com      # Via Tailscale

# 6. No dead-loop rules
sudo iptables -t nat -L PREROUTING -n | grep 8053  # Should be empty

# 7. hostapd not running
systemctl status hostapd                     # inactive (dead)
pgrep hostapd                                # no output

# 8. IP forwarding
cat /proc/sys/net/ipv4/ip_forward            # 1
```
