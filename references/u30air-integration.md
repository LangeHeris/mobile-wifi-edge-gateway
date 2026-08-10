# U30Air Integration — 5G WAN Gateway + Clash TProxy

The U30Air is not just a WiFi hotspot — it's the **WAN gateway** with Clash Meta
transparent proxy handling all outbound traffic. This document covers the U30Air
side of the architecture. For device API, ADB, and band locking, see the
`zte-router-api` skill.

## Device Summary

| Property | Value |
|----------|-------|
| Model | ZTE U30 Air |
| Firmware | U30AirV1.0.0B17 |
| OS | Android 13 (SDK 33) |
| Architecture | arm64-v8a, 8-core |
| SELinux | Permissive (key enabler) |
| Root | No su binary — boot scripts run as root via SELinux Permissive |
| Build | user/release-keys (not debuggable) |
| WiFi | 5GHz + 2.4GHz dual-band AP |
| 5G | NR SA, Band n41, 100MHz |
| Power | USB-C, battery (charge cap 90%) |

## Root Mechanism (Important)

The U30Air does NOT have traditional root (no su, no Magisk, no BusyBox).
Instead:

1. `getenforce` = Permissive
2. `ro.secure=1`, `ro.debuggable=0` — ADB cannot `adb root`
3. ADB shell runs as `uid=2000(shell)`
4. Boot scripts in `/sdcard/ufi_tools_boot.sh` execute as **root** (PID 1 child)
   because init runs them with elevated privileges under Permissive SELinux

This means: you cannot get a root shell via ADB interactively, but all
boot-time scripts run as root. The Clash.Core process also runs as root.

## Network Role

The U30Air serves as:
1. **5G WAN modem** — connects to carrier 5G NR network
2. **WiFi AP** — broadcasts <your-wifi-ssid> (192.168.0.0/24) for Pi5 uplink
3. **Transparent proxy** — Clash Meta TProxy intercepts all TCP/UDP
4. **DNS resolver** — Clash DNS with fake-IP (198.18.0.0/16)
5. **DHCP server** — assigns IPs to wlan1 clients (192.168.0.x)

### U30Air Network Config
- LAN IP: 192.168.0.1/24
- WiFi SSID: <your-wifi-ssid> (5GHz + 2.4GHz)
- DHCP: 192.168.0.x pool
- WAN: 10.x.x.x (carrier-grade NAT)
- Public IP: dynamic (via 5G carrier)

### Limitation: DHCP DNS is read-only
The U30Air firmware does NOT allow customizing DHCP-assigned DNS via API or web
UI. The `dhcp_dns1`/`dhcp_dns2` goform fields are read-only. All goformId
attempts (DHCP_SETTING, SET_NETWORK, BASIC_SETTING, SET_NV) fail silently.

Workaround: Pi5's wlan1 NM connection has `ipv4.ignore-auto-dns=yes` and
`ipv4.dns=127.0.0.1`, so it ignores the U30Air's DHCP DNS and uses local
AdGuard Home instead.

## Clash Meta (mihomo) Configuration

NOT ShellCrash. This is a standalone Clash Meta deployment from the
minikano/ufi_tools ecosystem.

### Directory Layout
```
/data/clash/
├── Proxy/
│   ├── Clash.Core          # 50MB arm64 binary (mihomo kernel)
│   ├── config.yaml         # 217KB, owned by u0_a95 (f50_sms app)
│   ├── GeoIP.dat           # 17MB
│   ├── GeoSite.dat         # 4MB
│   ├── cache.db            # 128KB
│   ├── WebUI/
│   │   ├── metacubexd/     # MetaCubeXD dashboard
│   │   └── dashboard/      # Zashboard dashboard
│   ├── proxies/            # proxy node configs
│   ├── rule_providers/     # rule provider configs
│   └── ruleset/            # rule sets
├── Scripts/
│   ├── Clash.Service       # start/stop wrapper
│   ├── Clash.Inotify       # config file change watcher
│   ├── main.sh             # 14KB - core logic
│   ├── ck_cfg.sh           # config check
│   └── mosdns.sh           # mosdns start/stop
└── Tools/
    ├── mosdns_arm64        # 28MB - DNS splitter
    ├── yq_linux_arm64      # 10MB - YAML processor
    ├── gojq                # 4MB - JSON query
    └── imosdns/            # mosdns config + geo lists
```

### Ports & Modes

| Setting | Value |
|---------|-------|
| Mode | TProxy |
| TProxy port | 7893 |
| Mixed proxy port | 7890 |
| DNS port | 1053 |
| Dashboard port | 7788 |
| TUN device | tun0 |
| FakeIP range | 198.18.0.0/16 |

### How TProxy Works

Clash TProxy intercepts ALL outbound TCP/UDP traffic on the U30Air at the
netfilter level. Traffic flow:

```
Device on U30Air LAN (e.g., Pi5 at <your-pi5-ip>)
  → U30Air receives packet
  → iptables TPROXY rule redirects to Clash port 7893
  → Clash matches rules (domain/IP/geoip)
  → If proxy: forwards through proxy node
  → If direct: sends directly to 5G modem
  → 5G NR → Internet
```

This means ALL traffic from Pi5 (and IoT devices behind Pi5) goes through
Clash's routing rules on the U30Air. No per-device proxy configuration needed.

### Config Subscription

`config.yaml` first line can be a URL → auto-downloads subscription.
If second line is `subconverter`, routes through subconverter API.

Subscription download uses a bundled curl:
`/data/data/com.minikano.f50_sms/files/curl`

### Clash.Service Script

```sh
#!/system/bin/sh
Module_dir="/data/clash"
sdcard_dir="/sdcard"
. /data/clash/Scripts/main.sh
case "$1" in
    start) echo "正在准备启动猫猫插件..."; start ;;
    stop) echo "正在准备关闭猫猫插件..."; stop ;;
    *) echo "只能使用start|stop两个参数控制内核启动或停止"; exit 1 ;;
esac
```

### Inotify Watcher

`Clash.Inotify` watches `/data/clash/Clash` directory for config changes.
When config.yaml is modified, it triggers automatic reload.

Runtime artifacts: `/data/local/tmp/inotify_pipe` (named pipe),
`/data/local/tmp/boot_once.flag`

## Boot Sequence

The boot script `/sdcard/ufi_tools_boot.sh` runs as root on every boot:

```sh
# 1. (Optional) Start your gateway/service if running on U30Air
# cd /data/picoclaw && nohup env HOME=/data/picoclaw \
#   SSL_CERT_DIR=/system/etc/security/cacerts \
#   /data/picoclaw/picoclaw gateway > /dev/null 2>&1 &

# 2. Start Clash Meta
/data/clash/Scripts/Clash.Service start

# 3. Start inotify watcher for Clash config changes
inotifyd /data/clash/Scripts/Clash.Inotify "/data/clash/Clash" >> /dev/null &

# 4. Start battery charge control script
/system/bin/sh /sdcard/kano_charge_control.sh &

# 5. Override Samba root password
echo -e "<your_samba_password>\n<your_samba_password>" | /system/bin/smbpasswd -s root > /dev/null 2>&1

# 6. Start low battery protection script
/system/bin/sh /sdcard/kano_low_battery.sh &
```

Order matters: start your gateway first (if any), then Clash (network), then
supporting services.

## Battery Management

### Charge Control (kano_charge_control.sh)

Config (`/sdcard/kano_charge_control_config.conf`):
```
max_charge=90
start_charge=1
```
Stops charging at 90%, resumes at 1% (effectively always charges when plugged
in but caps at 90% to preserve battery lifespan).

### Low Battery Protection (kano_low_battery.sh)

Config (`/sdcard/kano_low_battery_config.conf`):
```json
{
  "t": [
    {"level": 10, "action": "hotspot_on", "enabled": true},
    {"level": 5, "action": "hotspot_off", "enabled": true}
  ]
}
```
- At 10% battery: ensures hotspot is ON (so Pi5 can stay connected)
- At 5% battery: turns hotspot OFF (to conserve remaining battery)

This ensures the network stays up as long as possible during power loss, then
gracefully shuts down the WiFi to preserve emergency battery.

## Samba File Sharing

- Config: `/data/samba/etc/smb.conf`
- Password file: `/data/samba/etc/smbpasswd`
- Root password overridden at boot to `<your_samba_password>`
- Shares:
  - `/data/SAMBA_SHARE/机内存储` (fuse mount of internal storage)
  - `/data/SAMBA_SHARE/外部存储` (tmpfs, volatile)
- `unlock_samba.sh` on /sdcard: removes immutable flag + deletes smb.conf for
  reset

## mosdns (DNS Splitter)

- Binary: `/data/clash/Tools/mosdns_arm64` (28MB)
- Config: `/data/clash/Tools/imosdns/config_custom.yaml`
- Geo lists: `geosite_cn.txt`, `geosite_geolocation-!cn.txt`, `geoip_cn.txt`
- Controlled via `/data/clash/Scripts/mosdns.sh`
- Default: `mosdns_enable="false"` (disabled unless explicitly enabled)

When enabled, mosdns splits DNS queries: Chinese domains go to domestic DNS,
foreign domains go to Clash DNS. Disabled by default because Clash DNS
fake-IP mode handles routing adequately.

## UFI-TOOLS Keep-Alive

- Lock dir: `/data/local/tmp/ufi_keep_alive.lock/`
- PID file: `/data/local/tmp/ufi_keep_alive.lock/pid`
- Log: `/sdcard/ufi_keep_alive.log`
- Ensures boot scripts stay running; restarts if killed

## UFI-TOOLS Web Panel (Port 2333)

Management dashboard at `http://192.168.0.1:2333/`

**Project**: https://github.com/kanoqwq/UFI-TOOLS (minikano/ufi_tools)

UFI-TOOLS 是 minikano 开发的随身WiFi管理面板，提供 Clash 代理管理、TTYD 终端、
频段锁定、信号监控等功能。通过 boot 脚本以 root 权限自动启动。

| Setting | Value |
|---------|-------|
| Version | UFI-TOOLS v4.1.1 |
| Languages | 中文 / English / 日本語 / Tiếng Việt |
| Auth | Token or password (same as Samba: <your_samba_password>) |

### Login (for browser automation)

The panel uses hidden `.modal` elements. Login procedure:

1. Navigate to `http://192.168.0.1:2333/`
2. Show login modal via JS console:
   ```js
   document.querySelectorAll('.modal')[0].style.display = 'block';
   ```
3. Fill `input#TOKEN` and `input#PWDINPUT` with password (<your_samba_password>)
4. Click "Confirm" button
5. Page reloads with data and management sections

Post-login features: Clash dashboard (metacubexd iframe), TTYD terminal,
band locking (15 bands), signal parameters, device properties, real-time
monitoring.

## Open Ports (U30Air)

| Port | Service |
|------|---------|
| 80 | ZTE web UI (goform API) |
| 8080 | ZTE web UI (duplicate instance) |
| 2333 | UFI-TOOLS web panel (minikano) |
| 5555 | ADB daemon |
| 7788 | Clash dashboard (metacubexd/zashboard) |
| 7890 | Clash mixed proxy |
| 7893 | Clash TProxy |
| 1053 | Clash DNS |
| 53 | dnsmasq DNS (loopback + LAN) |
| 445 | Samba (SMB) |

## Clash + Cloudflared Interference (Known Issue)

Cloudflared on Pi5 connects to Cloudflare edge on TCP:7844. Clash TProxy on
U30Air intercepts this TCP traffic and routes through proxy nodes, which block
HTTP/2 long connections.

**Symptoms**: cloudflared HTTP/2 connection fails, auto-degrades to QUIC
(UDP:7844) which works.

**Fix**: Add Clash direct rule for argotunnel.com:
```
DOMAIN-SUFFIX,argotunnel.com,DIRECT
```

See `references/pitfalls.md` for full details.

## How to Replicate (U30Air Side)

1. Obtain a ZTE U30Air (or compatible 5G MiFi that supports minikano/ufi_tools)
2. Enable ADB: Settings → Developer options → USB debugging
3. Connect via ADB: `adb connect 192.168.0.1:5555`
4. Install minikano/ufi_tools suite: https://github.com/kanoqwq/UFI-TOOLS
5. Place boot script at `/sdcard/ufi_tools_boot.sh`
6. Configure Clash subscription URL in `/data/clash/Proxy/config.yaml`
7. Set battery config in `/sdcard/kano_charge_control_config.conf`
8. Set low battery config in `/sdcard/kano_low_battery_config.conf`
9. Reboot device — boot scripts run automatically as root

For detailed ADB access, goform API, and band locking, see the `zte-router-api`
skill which has complete documentation and scripts.
