# Pi5 Software Router Configuration

All configs verified on a production Raspberry Pi 5 running Debian 13 (trixie).

## Hardware: USB WiFi Adapter

The external USB WiFi adapter used for wlan1 (uplink to U30Air):

| Field | Value |
|---|---|
| Product | 802.11ac WLAN |
| Chip | MediaTek MT7612U |
| Vendor ID | 0e8d |
| Product ID | 7612 |
| Driver | mt76x2u (kernel built-in) |
| Manufacturer | MediaTek Inc. |
| Band | Dual-band 802.11ac (5GHz preferred) |

Verify on your system:
```bash
cat /sys/bus/usb/devices/1-2/product      # "802.11ac WLAN"
cat /sys/bus/usb/devices/1-2/idVendor     # 0e8d
cat /sys/bus/usb/devices/1-2/idProduct    # 7612
cat /sys/class/net/wlan1/device/driver    # mt76x2u
```

Any 802.11ac USB adapter with mt76x2u / rtl8812au / rtl8821cu driver works
as the uplink interface. The Pi5 onboard WiFi (wlan0) serves as the AP.

## NetworkManager: The Core

NetworkManager (nmcli) manages both WiFi interfaces. This is the recommended
approach on Pi5 - do NOT use legacy hostapd/dhcpcd alongside NM.

### wlan1 - Uplink Client (to U30Air)

Connection name: `<your-wifi-ssid> 1`
UUID: `<your-connection-uuid>`

Key NM properties:
```ini
connection.type = 802-11-wireless
connection.interface-name = wlan1
connection.autoconnect = yes
connection.autoconnect-priority = 100
connection.autoconnect-retries = 0 (forever)

802-11-wireless.ssid = <your-wifi-ssid>
802-11-wireless.mode = infrastructure
802-11-wireless.band = (auto — connects on 5GHz)
802-11-wireless-security.key-mgmt = wpa-psk
802-11-wireless-security.auth-alg = open
802-11-wireless-security.psk = <your_wifi_password>

ipv4.method = auto
ipv4.dns = 127.0.0.1
ipv4.ignore-auto-dns = yes
ipv6.method = auto
```

Critical settings:
- `autoconnect-priority = 100` — ensures wlan1 connects first
- `autoconnect-retries = 0` — retry forever (U30Air may reboot)
- `ipv4.dns = 127.0.0.1` — use local AdGuard, NOT U30Air DHCP DNS
- `ipv4.ignore-auto-dns = yes` — critical: prevents U30Air from overriding DNS

Result: wlan1 gets <your-pi5-ip>/24 via DHCP, gateway 192.168.0.1, but DNS
points to localhost (AdGuard Home).

### wlan0 — AP for IoT (NetworkManager Shared Mode)

Connection name: `<your-iot-ssid>`
UUID: `<your-connection-uuid>`

Key NM properties:
```ini
connection.type = 802-11-wireless
connection.interface-name = wlan0
connection.autoconnect = yes

802-11-wireless.ssid = <your-iot-ssid>
802-11-wireless.mode = ap
802-11-wireless.band = bg
802-11-wireless.channel = 6
802-11-wireless-security.key-mgmt = wpa-psk
802-11-wireless-security.proto = rsn
802-11-wireless-security.pairwise = ccmp
802-11-wireless-security.group = ccmp
802-11-wireless-security.pmf = 1 (disable)

ipv4.method = shared
ipv4.addresses = 10.42.0.1/24
ipv4.gateway = (unset — no gateway for AP itself)
```

Critical settings:
- `mode = ap` — NetworkManager creates a WiFi access point
- `ipv4.method = shared` — NM automatically:
  1. Assigns 10.42.0.1/24 to wlan0
  2. Starts dnsmasq for DHCP (10.42.0.10-254) and DNS
  3. Enables iptables/nftables masquerade for 10.42.0.0/24
  4. Enables IP forwarding via sysctl
- `band = bg` + `channel = 6` — 2.4GHz only (IoT compatibility)
- `pairwise = ccmp` — WPA2-CCMP encryption (not TKIP)

### How to Replicate (nmcli commands)

```bash
# Create uplink connection (wlan1 → U30Air)
nmcli connection add type wifi ifname wlan1 con-name "<your-wifi-ssid>" \
  ssid "<your-wifi-ssid>" \
  wifi-sec.key-mgmt wpa-psk \
  wifi-sec.psk "YOUR_WIFI_PASSWORD" \
  ipv4.method auto \
  ipv4.dns "127.0.0.1" \
  ipv4.ignore-auto-dns yes \
  connection.autoconnect yes \
  connection.autoconnect-priority 100 \
  connection.autoconnect-retries 0

# Create AP connection (wlan0 → IoT)
nmcli connection add type wifi ifname wlan0 con-name "<your-iot-ssid>" \
  ssid "<your-iot-ssid>" \
  wifi.mode ap \
  wifi.band bg \
  wifi.channel 6 \
  wifi-sec.key-mgmt wpa-psk \
  wifi-sec.psk "YOUR_IOT_PASSWORD" \
  wifi-sec.proto rsn \
  wifi-sec.pairwise ccmp \
  wifi-sec.group ccmp \
  ipv4.method shared \
  ipv4.addresses "10.42.0.1/24"

# Activate both
nmcli connection up "<your-wifi-ssid>"
nmcli connection up "<your-iot-ssid>"
```

## dnsmasq (NM-managed, NOT systemd service)

The systemd `dnsmasq.service` is DISABLED. NetworkManager runs its own dnsmasq
instance internally.

### Active dnsmasq process

```
/usr/sbin/dnsmasq \
  --conf-file=/dev/null \
  --no-hosts \
  --keep-in-foreground \
  --bind-interfaces \
  --except-interface=lo \
  --clear-on-reload \
  --strict-order \
  --listen-address=10.42.0.1 \
  --dhcp-range=10.42.0.10,10.42.0.254,3600 \
  --dhcp-leasefile=/var/lib/NetworkManager/dnsmasq-wlan0.leases \
  --pid-file=/run/nm-dnsmasq-wlan0.pid \
  --conf-dir=/etc/NetworkManager/dnsmasq-shared.d
```

Key points:
- Listens on 10.42.0.1:53 (wlan0 only)
- DHCP range: 10.42.0.10 - 10.42.0.254 (1-hour lease)
- Config dir: `/etc/NetworkManager/dnsmasq-shared.d/`
- Do NOT edit /etc/dnsmasq.conf — it is NOT used by NM

### Custom dnsmasq overrides

Create files in `/etc/NetworkManager/dnsmasq-shared.d/` to add custom rules:

```bash
# Example: static DHCP lease for IoT device
echo "dhcp-host=aa:bb:cc:dd:ee:ff,10.42.0.50,iot-device-name" \
  > /etc/NetworkManager/dnsmasq-shared.d/iot-static.conf

# Example: point DNS upstream to AdGuard
echo "server=127.0.0.1#8053" \
  > /etc/NetworkManager/dnsmasq-shared.d/upstream.conf
```

Restart NM to apply: `nmcli connection down "<your-iot-ssid>" && nmcli connection up "<your-iot-ssid>"`

## Legacy Config Files (DO NOT USE — inactive)

These files exist from a previous RaspAP installation but are NOT active:

### /etc/hostapd/hostapd.conf (and hostapd-wlan0ap.conf)
```ini
interface=wlan1          # WRONG: should be wlan0
driver=nl80211
ssid=<your-iot-ssid>
hw_mode=g
channel=6
wmm_enabled=1
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=<your_password>
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP         # WRONG: should be CCMP only
rsn_pairwise=CCMP
```

Note: The file says `interface=wlan1` which is incorrect — it should be wlan0.
This config is NOT loaded because hostapd.service is disabled.

### /etc/default/hostapd
```ini
DAEMON_CONF="/etc/hostapd/hostapd-wlan0ap.conf"
```

This points to a different file but hostapd.service is inactive (dead).

### /etc/dhcpcd.conf
Contains a static IP config for wlan0 as 10.3.141.1/24 (different subnet than
current 10.42.0.1/24). This is NOT used — NetworkManager handles IP assignment.

### systemd override for hostapd
```
# /etc/systemd/system/hostapd.service.d/ipfix.conf
[Service]
ExecStartPost=/bin/bash -c "sleep 1 && ip addr add 10.42.0.1/24 dev wlan0 2>/dev/null || true"
```

This was a workaround to add the IP after hostapd started. No longer needed
since NM handles it. The file is harmless but should be removed to avoid
confusion.

### RaspAP residual
- `raspap-network-activity@wlan0.service` — still running but only monitors
  network activity. Does not manage the AP. Safe to leave or disable.

## IP Forwarding

### sysctl configuration
```bash
# /etc/sysctl.d/90_raspap.conf
net.ipv4.ip_forward=1
```

Verify:
```bash
cat /proc/sys/net/ipv4/ip_forward
# Output: 1
```

NetworkManager's `shared` mode also enables this automatically, but the sysctl
config ensures it persists across reboots even if NM doesn't set it.

## iptables / nftables

The system uses iptables-nft (iptables command wrapping nftables kernel API).

### NAT table (PREROUTING)
```
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0  ADDRTYPE match dst-type LOCAL
```

Only Docker's PREROUTING rule exists. No custom DNS redirect rules (the old
iptables REDIRECT 53→8053 rules were removed — see pitfalls.md).

### NAT table (POSTROUTING / masquerade)
Managed by nftables. Key masquerade rules:
```
ip saddr 172.17.0.0/16 oifname != "docker0" masquerade
ip saddr 172.19.0.0/16 oifname != "br-6b603b6c07cf" masquerade
ip saddr 172.18.0.0/16 oifname != "br-5ba548806a6f" masquerade
# Generic masquerade (NM-added for shared mode)
counter masquerade
```

### FORWARD chain
```
Chain FORWARD (policy DROP)
ACCEPT  all  --  10.42.0.0/24  0.0.0.0/0    (IoT → anywhere)
ACCEPT  all  --  0.0.0.0/0  0.0.0.0/0  ctstate RELATED,ESTABLISHED
DOCKER  rules...
```

The ACCEPT rule for 10.42.0.0/24 allows IoT devices to reach the Internet
through the Pi5. NetworkManager adds this automatically with shared mode.

## 1Panel Firewall

1Panel management panel (port 5689) adds its own nftables chains:
- `1PANEL_BASIC` — input filtering
- `1PANEL_FORWARD` — forward filtering
- `1PANEL_BASIC_AFTER` / `1PANEL_BASIC_BEFORE`

These coexist with Docker and NM rules. If you encounter connectivity issues,
check: `sudo nft list ruleset | grep 1PANEL`

## Verification Commands

```bash
# Check NM connections
nmcli connection show
nmcli device status

# Check wlan0 AP status
nmcli -f all connection show "<your-iot-ssid>" | grep -E "ssid|mode|band|channel|method|address"

# Check wlan1 client status
nmcli -f all connection show "<your-wifi-ssid> 1" | grep -E "ssid|mode|dns|autoconnect"
iw dev wlan1 link

# Check routing
ip route

# Check IP forwarding
sysctl net.ipv4.ip_forward
cat /proc/sys/net/ipv4/ip_forward

# Check dnsmasq
pgrep -a dnsmasq

# Check masquerade
sudo nft list ruleset | grep masquerade

# Check hostapd is NOT running
systemctl status hostapd
pgrep -a hostapd  # should return nothing

# Check interfaces
ip addr show | grep -E "^[0-9]+:|inet "
```
