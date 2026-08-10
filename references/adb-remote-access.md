# ADB Remote Access (Pi5 -> U30Air)

U30Air exposes ADB on port 5555. Pi5 (Hermes container) can directly shell
into U30Air for proxy logs, process inspection, file reads, and config checks.

## Setup (one-time, persists in /tmp)

No sudo needed -- just `apt-get download` + `dpkg-deb -x`:

```bash
# 1. Download adb binary + shared libs (arm64)
mkdir -p /tmp/adb-extracted /tmp/adb-deps
cd /tmp/adb-deps && apt-get download adb
dpkg-deb -x adb_*.deb /tmp/adb-extracted
apt-get download android-libbase android-libboringssl android-libcutils \
  android-liblog android-libutils android-libziparchive libprotobuf32t64
for deb in *.deb; do dpkg-deb -x "$deb" /tmp/adb-deps; done

# 2. Create wrapper at <hermes-install-dir>/bin/adb-remote
cat > <hermes-install-dir>/bin/adb-remote << 'EOF'
#!/bin/bash
ADB_BIN="/tmp/adb-extracted/usr/lib/android-sdk/platform-tools/adb"
LIB_DIR="/tmp/adb-deps/usr/lib/aarch64-linux-gnu/android"
LIB_DIR2="/tmp/adb-deps/usr/lib/aarch64-linux-gnu"
export LD_LIBRARY_PATH="${LIB_DIR}:${LIB_DIR2}:${LD_LIBRARY_PATH}"
exec "$ADB_BIN" "$@"
EOF
chmod +x <hermes-install-dir>/bin/adb-remote
```

Note: /tmp is volatile -- rerun step 1 after Pi5 reboot. The wrapper at
<hermes-install-dir>/bin/adb-remote persists.

## Usage

```bash
# Connect
adb-remote connect 192.168.0.1:5555
adb-remote devices                         # verify connected

# Common operations
adb-remote shell "cat /sdcard/mihomo.log"    # proxy logs
adb-remote shell "ps -A | grep mihomo"       # proxy process
adb-remote shell "pm list packages -3"       # 3rd-party APKs
adb-remote shell "getprop ro.product.model"  # Device info
adb-remote shell "cat /data/clash/Proxy/config.yaml" # proxy config
adb-remote shell "ls /sdcard/"               # File listing
```

From Hermes execute_code (terminal triggers null-char bug on adb paths):
```python
import subprocess
result = subprocess.run(
    ["<hermes-install-dir>/bin/adb-remote", "shell", "cat", "/sdcard/mihomo.log"],
    capture_output=True, text=True, timeout=15
)
print(result.stdout)
```

## ADB Shell Limitations

- Shell runs as `uid=2000(shell)`, NOT root
- No `su`, no Magisk -- but SELinux=Permissive lets boot scripts run as root
- Proxy process (mihomo/Clash.Core) runs as root (started by boot script)
- Config files owned by `u0_a95` may be unreadable from shell
- Workaround: read via /proc/PID/ or proxy REST API (port 7788, secret in config)

For full ADB protocol details, raw-socket fallback, and more commands,
see the `zte-router-api` skill (ADB Access section).
