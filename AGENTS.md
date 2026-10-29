# AGENTS.md

This file documents conventions for AI coding assistants working on the
`rw_mt5` repository.

## Repository

- This is the **`rw_mt5`** repository.
- It holds the MetaTrader 5 (MT5) expert advisor deployment pipeline,
  container definitions for KasmVNC/Wine-based MT5 instances, MariaDB
  schema and monitoring, and the shared WireGuard mesh infrastructure
  used to reach all Docker VMs and hosts.
- Key directories:
  - `scripts/mt5/experts/` — MT5 expert advisor pipeline (DLL + MQL5)
  - `scripts/mt5/experts/deploy/` — deploy, build, rollback scripts
  - `scripts/mt5/experts/dll/` — C DLL source (`mt5apidll.c`)
  - `scripts/mt5/experts/mql5/` — MQL5 expert source (`mt5_apidll_expert.mq5`)
  - `scripts/mt5/experts/sql/` — SQL migrations
  - `scripts/mt5/` — MT5 keepalive monitoring, management scripts
  - `scripts/vm104/mt5-actr-test-vm104/` — KasmVNC+Wine MT5 container
  - `scripts/host/` — host-side helpers (routing, firewall, NPM, fail2ban)
- **Never** store private keys, pre-shared keys, MT5 broker passwords, or
  user passwords in any `.md` file. Public keys and non-secret identifiers
  are acceptable.

## AI Assistant Instructions

- Always read existing files before editing them.
- Check `git status` and `git log --oneline -5` before making changes.
- Do not commit unless explicitly asked.
- Use conventional commit message format (e.g. "feat: add ...",
  "fix: correct ...").

---

# MT5 Operations & Configuration

## MT5 Graceful Stop/Start Policy

**Rule:** Send `WM_CLOSE` to the MT5 window via `wmctrl`. This is identical
to clicking File → Exit or the X button — MT5 saves workspace settings,
chart templates, window layout, and exits cleanly.

```sh
# Priority 1: WM_CLOSE via window manager (preserves all settings)
docker exec <container> sh -c '
  export DISPLAY=:1
  WMID=$(wmctrl -l 2>/dev/null | grep -i "ActivTrades\|MetaTrader" | head -1 | awk "{print \$1}")
  if [ -n "$WMID" ]; then
    wmctrl -i -c "$WMID"
  fi
'
# Wait up to 15s for graceful exit
for i in 1 2 3 4 5; do sleep 3;
  docker exec <container> sh -c "ps aux | grep -v grep | grep -q terminal64" || break
done

# Priority 2: SIGTERM to terminal64.exe (Wine translates to WM_QUIT)
docker exec <container> sh -c '
  PID=$(ps aux | grep "[t]erminal64\.exe" | awk "{print \$2}")
  [ -n "$PID" ] && kill -TERM "$PID"
'

# Fallback: wineserver -k (last resort after 15s timeout)
docker exec <container> sh -c "export WINEPREFIX=\$WINEPREFIX && wineserver -k"
```

**Never use SIGKILL (`kill -9`).** It destroys chart bindings and requires
manual expert re-attach from the KasmVNC desktop.

## Start MT5

```sh
docker exec -d <container> sh -c \
  "export WINEPREFIX=/home/kasm-user/.wine DISPLAY=:1 && \
   /opt/wine-staging/bin/wine \"$MT5_DIR/terminal64.exe\" >/dev/null 2>&1"
```

After a graceful stop + start, MT5 restores saved chart bindings and the
expert initializes automatically. Keepalive data appears in MariaDB ~5 s
after the expert starts.

## Re-attach Expert After Binding Loss

If chart bindings were lost:
1. Open `http://192.168.54.111:10190/` (KasmVNC desktop)
2. In MT5 Navigator (Ctrl+N), expand **Expert Advisors** → **custom**
3. Drag `mt5_apidll_expert.ex5` onto each chart
4. Check the Experts log (View → Experts) for "DB:" messages

## MT5 Timestamp & Timezone

**Timezone: UTC.** Both the Wine container and MariaDB use UTC:

- **MT5 container:** container base (jammy) uses UTC. Expert Log timestamps
  are in UTC as MT5 inherits the system time.
- **MariaDB (`mt5-mariadb`):** `TIMESTAMP` columns (`created_at`,
  `last_seen`) are stored as UTC. When queried, they are displayed in the
  connection's timezone (defaults to UTC if `@@time_zone = '+00:00'`).

All time-sensitive columns use `TIMESTAMP` (not `DATETIME`):

| Table | Column | Type | Note |
|---|---|---|---|
| `mt5_experts` | `created_at` | TIMESTAMP | Set on INSERT via `DEFAULT current_timestamp()` in UTC |
| `expert_keepalive` | `last_seen` | TIMESTAMP | Updated via `ON DUPLICATE KEY UPDATE last_seen=NOW()`. `NOW()` returns the MariaDB server's UTC time. |

Verify server/session timezone: `SELECT @@global.time_zone, @@session.time_zone;`
(`'+00:00'` means UTC).

## MT5 Kuma Monitoring

Each Kuma instance (NBG and VIE-1) monitors MT5 experts through **Push
monitors**. A systemd timer on each hub runs every 60 seconds, queries
`expert_keepalive` in MariaDB on vm104, and pushes to the local Kuma's push
API.

| Monitor name pattern | What it tracks |
|---|---|
| `mt5_expert_keepalive` | **Aggregate** – UP if at least one expert has reported within 120s |
| `mt5_{hostname}_{magic}` | **Per-expert** – UP only if that specific instance reported within 120s |

### Adding a New Expert

After attaching an expert to a chart in MT5, run on **both** hubs:

```bash
sudo /usr/local/sbin/mt5-keepalive-setup.sh nbg
sudo /usr/local/sbin/mt5-keepalive-setup.sh vie-1
```

The script reads all `(hostname, magic)` pairs from `expert_keepalive`,
creates a Kuma push monitor + token file for any new ones. The systemd
timer picks it up within 60 seconds.

### Systemd Timer

- Service: `mt5-keepalive-push.service` (oneshot)
- Timer: `mt5-keepalive-push.timer` (every 60s)
- Script: `/usr/local/sbin/mt5-keepalive-push.sh`
- Tokens: `/usr/local/etc/mt5-keepalive-tokens/*.token`

### Scripts

| Script | Purpose |
|---|---|
| `scripts/mt5/mt5-keepalive-push.sh` | Push script, installed on both hubs |
| `scripts/mt5/mt5-keepalive-push.service` | Systemd service unit |
| `scripts/mt5/mt5-keepalive-push.timer` | Systemd timer (60s) |
| `scripts/mt5/mt5-keepalive-setup.sh` | Sync Kuma monitors with MariaDB experts |

---

# MT5 Expert Advisor DevOps Pipeline

Located in `scripts/mt5/experts/`.

## Architecture

| Component | Path | Description |
|---|---|---|
| C DLL | `dll/mt5apidll.c` | Native DLL loaded by MT5 expert; `GetComputerNameW` (15-char limit), `db_connect` with 3s timeout, `db_keepalive` upsert. Loads `libmariadb.dll` dynamically via `LoadLibrary`/`GetProcAddress`. Handle store avoids 64-bit pointer truncation. Credentials from `.env`. |
| MQL5 Expert | `mql5/mt5_apidll_expert.mq5` | Multi-instance expert: worker modes (keepalive\|trading\|account\|symbol\|history), unique magic per chart, `Log()` helper tags every line with `[hostname\|mode\|magic\|chartid]`, uses `StringToCharArray` + `uchar&[]` for DLL params, MariaDB keepalive every tick/timer. |
| SQL Migrations | `sql/` | Schema changes for `mt5_data` database. |
| Deploy Scripts | `deploy/` | Master pipeline, build-only, rollback, shared lib. |

## Pipeline

### deploy/deploy.sh (master pipeline)

1. **Stop MT5 gracefully** (`wmctrl WM_CLOSE` → SIGTERM → `wineserver -k`)
2. **Move old files to backup** (archives in `archive/{env}/{ts}/`)
3. **Compile DLL** (`gcc -shared -o mt5apidll.dll mt5apidll.c ...`)
4. **Compile MQL5** (MetaEditor64.exe /compile under Xvfb)
5. **Create versioned archive** (`archive/{env}/{build}/` with DLL, .ex5,
   sources, VERSION metadata; update latest symlink)
6. **Copy to container** via auto-forward (SSH over WireGuard mesh from
   any Kandev sandbox)
7. **Start MT5** (graceful start)
8. **Apply SQL** migrations
9. **Sync Kuma MT5 monitors** on both hubs

### deploy/build.sh (compile-only)

Creates `archive/{env}/{build}/` with DLL, .ex5, sources, VERSION metadata,
updates latest symlink.

### deploy/rollback.sh

List available builds and switch between versions.

### deploy/lib.sh (shared)

Functions: `host_to_ip` (16 hosts), versioning, auto-forward via direct SSH
from sandbox over WireGuard mesh.

## Naming Convention

**Hostname:** max 15 characters (Windows `GetComputerNameW` limit). MT5
container hostnames follow the pattern `MT5ATRS01P104_1` (broker code +
site + identifier).

## Container Access for Deployment

The MT5 container (`mt5-actr-s01-p104-1`) has SSH access to all mesh hosts
using the shared `kandev` ed25519 key:

- **Key:** `/home/kasm-user/.ssh/kandev`
- **Config:** `/home/kasm-user/.ssh/config` — mesh hosts via tunnel IPs,
  proxies via LAN IPs, `Host *` with KEX algorithms avoiding post-quantum
  key exchange (PQ ML-KEM packets exceed the 1428-byte path MTU behind
  owrt002's DSL/cable NAT). Force ECDH-only: `curve25519-sha256`,
  `curve25519-sha256@libssh.org`, `ecdh-sha2-nistp256/384/521`.
- **mesh-test.sh:** `/home/kasm-user/scripts/check-hosts/mesh-test.sh`
  runs with auto-detected sandbox tag.

**Post-quantum SSH key-exchange / MTU fix (2026-10-17):** The vm104 host
and its containers sit behind owrt002's DSL/cable NAT (`84.115.210.109`),
which has a path MTU of ~1428 bytes — too small for ~1250-byte ML-KEM
post-quantum key-exchange packets (`mlkem768x25519-sha256`, default in
OpenSSH 10.x). SSH to any host running OpenSSH 10.x hung at
`expecting SSH2_MSG_KEX_ECDH_REPLY`. Fix: force ECDH-only KEX algorithms.
Applied in: `mesh-test.sh` `SSH_OPTS`, container `~/.ssh/config`, vm104
host `~/.ssh/config`.

---

# KasmVNC MT5 Container

## Test Instance: `mt5-actr-test-vm104` (deployed 2026-10-16)

First KasmVNC test container on vm104 (`192.168.54.111`):

| Attribute | Value |
|---|---|
| **Image** | `kasmweb/core-ubuntu-jammy:1.15.0` (later: `kasmweb/core-ubuntu-jammy:1.15.0` + Wine 11.20 Staging) |
| **Container name** | `mt5-actr-test-vm104` → renamed `mt5-actr-s01-p104-1` |
| **Host port** | `192.168.54.111:10190` → `6901` (KasmVNC HTTPS) |
| **Desktop** | XFCE4 (via `START_XFCE4=1`) |
| **Wine** | Upgraded from 6.0.3 to **Wine 11.20 Staging** (WineHQ PPA). Removed `WINEARCH=win64` (11.20 WoW64 unified). Symlink `/usr/bin/wine64 → /opt/wine-staging/bin/wine`. |
| **Dependencies** | `corefonts`, `vcrun2022`, `gdiplus` via `winetricks`. `wine-mono` 11.3.0. `binfmt-support` + binfmt registration (`MZ` magic → `wine64`). |
| **Compose dir** | `/home/ubuntu/docker/mt5-actr-s01-p104-1/` on vm104 |
| **Repo copy** | `scripts/vm104/mt5-actr-s01-p104-1/` |

### NPM Hosts (both hubs, Authelia two-factor + Basic `mesh-admin`)

- `mt5-actr-s01-p104-1.gw-1-nbg-1-de-netcup.rwnix.net` (NBG row 50, list 1)
- `mt5-actr-s01-p104-1.gw-1-vie-1-at-netcup.rwnix.net` (VIE-1 row 62, list 8)

### Known Limitations (historical, most fixed)

- Wine 6.0.3 → upgraded to **11.20 Staging** (latest as of 2026-10-17)
- No activation/reg file — broker ini, `.mq5` experts, socket bridge pending
- `.env` contains VNC password (mode 600, never committed)
- MT5 auto-start on container boot via `/dockerstartup/custom_startup.sh`

### Helper Script

Inside the container: `/home/kasm-user/install-mt5.sh` (versioned as
`scripts/vm104/mt5-actr-s01-p104-1/install-mt5.sh`). Run from KasmVNC
desktop terminal:
```bash
bash /home/kasm-user/install-mt5.sh
```

### MT5 Auto-Start

MT5 launches automatically via `/dockerstartup/custom_startup.sh` (baked
into the Dockerfile). The script waits for the XFCE desktop via
`/usr/bin/desktop_ready`, checks if MT5 is installed, and launches
`terminal64.exe` with Wine. No manual intervention needed after container
restart.

### Persistent User Settings

Desktop color/theme/XFCE settings stored in the named volume
`mt5-actr-s01-p104-1-config` (`/home/kasm-user/`). On `--volumes` teardown
they reset to defaults. Key files:

| File | Contains |
|---|---|
| `.config/dconf/user` | GTK theme, color scheme (binary dconf DB) |
| `.config/gtk-3.0/` | GTK3 theme overrides (`settings.ini`) |
| `.config/xfce4/xfconf/xfce-perchannel-xml/` | XFCE panel, window manager, display settings |

Backup/restore procedure documented in `pve103-pve104-inventory.md`.

## Wine MT5 Container Internals

| Item | Detail |
|---|---|
| Wine version | 11.20 Staging (upgraded from 6.0.3 via WineHQ PPA) |
| Wine prefix | `win64` at `/home/kasm-user/.wine` (persistent named volume) |
| Architecture | `amd64` + `i386` as foreign arch — full x86/x64 |
| Winbind | Installed |
| winetricks | 20260125-next at `/usr/local/bin/winetricks` |
| Dependencies | `corefonts`, `vcrun2022`, `gdiplus` |
| binfmt | `MZ` magic → `wine64`; `.exe` files with +x auto-launch |
| Mono | 11.3.0 (for Wine 11.20 compatibility) |
| Gecko | 32+64 bit installed |

### KasmVNC Display

KasmVNC manages the full display lifecycle internally — no manual `:99`
Xvfb coordination. Each container runs one `vncserver` on its own port
(6901). Default config: `new_session_disconnects_existing_exclusive_session:
false` — two browser tabs see the same session.

### Idle Timeout

NPM default `proxy_read_timeout` is 240s → KasmVNC sessions disconnect
after 4 min idle. The global `server_proxy.conf` has `proxy_read_timeout
86400s;` on both hubs.

## MT5 MariaDB

MariaDB 11.5 container for MT5 expert data, port 3309 on vm104.

```yaml
services:
  mt5-mariadb:
    image: mariadb:11.5@sha256:...
    ports:
      - "192.168.54.111:3309:3306"
    volumes:
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
      - mariadb-data:/var/lib/mysql
    environment:
      MARIADB_ROOT_PASSWORD: <from .env>
```

Schema: `mt5_data` database with `mt5_experts` and `expert_keepalive`
tables. Deploy by running `scripts/mt5/experts/sql/` migrations.

---

# Mesh Network Infrastructure

## WireGuard Mesh Topology

The WireGuard mesh connects remote sites through two hub hosts:

| Hub | Hostname | Public IP | WG Subnet | WG Port |
|---|---|---|---|---|
| NBG | `gw-1-nbg-1-de-netcup.rwnix.net` | `152.53.118.212` | `10.1.1.0/24` | 51820 |
| VIE-1 | `gw-1-vie-1-at-netcup.rwnix.net` | `152.53.35.177` | `10.2.1.0/24` | 51820 |

Both hubs run wg-easy (container-managed WireGuard), `WG-EASY-HOST`
DOCKER-USER chain for mesh traffic isolation, and reciprocal
`wg_interhub` clients for direct hub-to-hub management routing
(`10.1.1.17` NBG, `10.2.1.17` VIE-1 via `wg-interhub-policy.service`).

### Site Routers (OpenWrt)

All 8 commissioned routers per hub (16 total) connect as WireGuard peers:

| Router | NBG IP | VIE-1 IP | Site LAN |
|---|---|---|---|
| owrt001 | 10.1.1.3 | 10.2.1.3 | 192.168.1.0/24 |
| owrt002 | 10.1.1.2 | 10.2.1.2 | 192.168.15/53/54.0/24 |
| owrt003 | 10.1.1.4 | 10.2.1.4 | 192.168.16.0/24 |
| owrt011 | 10.1.1.5 | 10.2.1.5 | 192.168.14/51/52/101/102.0/24 |
| owrt018 | 10.1.1.6 | 10.2.1.6 | 192.168.18.0/24 |
| owrt031 | 10.1.1.7 | 10.2.1.7 | 192.168.12/141.0/24 |
| owrt032 | 10.1.1.8 | 10.2.1.8 | 192.168.11.0/24 |
| owrt033 | 10.1.1.9 | 10.2.1.9 | 192.168.13.0/24 |

All routers use MTU 1320 (consistent across all 16 OpenWrt routers, both
hub hosts wg0, and all wg-easy peers). `tcp_mtu_probing` = 0 on all.

### Gateway VMs (OpenWrt on Proxmox)

| VM | Host | Public IP | NBG IP | VIE-1 IP | Site LAN |
|---|---|---|---|---|---|
| owrt-fsn-1 | fsn-1 | 144.76.14.154 | 10.1.1.12 | 10.2.1.12 | 192.168.111.0/24 |
| owrt-hel-1 | hel-1 | 65.109.122.220 | 10.1.1.11 | 10.2.1.11 | 192.168.112.0/24 |
| owrt-gra-1 | gra-1 | 51.68.65.24 | 10.1.1.13 | 10.2.1.13 | 192.168.13.0/24 |
| owrt-lim-1 | lim-1 | 51.68.177.41 | 10.1.1.14 | 10.2.1.14 | 192.168.14.0/24 |
| owrt-eri-1 | eri-1 | 57.128.154.34 | 10.1.1.10 | 10.2.1.10 | 192.168.10.0/24 |
| owrt-waw-1 | waw-1 | 51.83.180.112 | 10.1.1.15 | 10.2.1.15 | 192.168.15.0/24 |
| owrt-pve103 | — | (behind owrt002) | 192.168.15.53 | 192.168.15.53 | 192.168.53.0/24 |
| owrt-pve104 | — | (behind owrt002) | 192.168.15.54 | 192.168.15.54 | 192.168.54.0/24 |

## Docker VMs Hosting MT5 Containers

### docker-1-pve-104 (vm104, `192.168.54.111`)

Primary MT5 container host. 26 running containers as of 2026-10-17.

| Container | Image | Purpose | Ports |
|---|---|---|---|
| `mt5-actr-s01-p104-1` | `mt5-actr-s01-p104-1:2.0.0` | KasmVNC+Wine+ActivTrades MT5 | `:10190` |
| `mt5-mariadb` | `mariadb:11.5` | MariaDB for MT5 expert data | `:3309` |
| `kandev104` | `kandev:v0.96.0-ssh` | Kandev sandbox | `:38431`, `127.0.0.1:2222` |

### docker-1-pve-103 (vm103, `192.168.53.111`)

11 running containers.

### Legacy MT5 Containers (eri-1, waw-1)

| VM | Container | Broker | Web port | Status |
|---|---|---|---|---|
| eri-1 docker-1 (192.168.113.11) | `mt5-1-activtrades-1` | ActivTrades | 10101→3001 | Up |
| eri-1 docker-1 | `mt5-1-fxpro-1` | FxPro | 20101→3001 | Up |
| eri-1 docker-1 | `mt5-1-xm-1` | XM | 30101→3001 | Up |
| eri-1 docker-2 (192.168.113.12) | `mt5-1-fxpro-1` | FxPro | 20101→3001 | Up |
| waw-1 docker-1 (192.168.114.11) | `mt5-1-activtrades-1` | ActivTrades | 10101→3001 | Up |

All legacy containers use `lscr.io/linuxserver/webtop:ubuntu-xfce` with
Selkies streaming (Xvfb `:99`). The KasmVNC migration replaces these.

### Key Docker VM Routes

All behind owrt002 (br-lan, `10.1.1.2`):

| Subnet | Purpose | Key Hosts |
|---|---|---|
| `192.168.15.0/24` | owrt002 br-lan | pve103 (`.7`), pve104 (`.6`) |
| `192.168.53.0/24` | owrt053 LAN | docker-1-pve-103 (`.111`) |
| `192.168.54.0/24` | owrt054 LAN | docker-1-pve-104 (`.111`) |

All Docker VMs reachable from both Kandev sandboxes directly via the
WireGuard mesh (no ProxyJump needed since 2026-09-23).

## OpenWrt Safe-Change Policy

When changing MT5-related routing on OpenWrt routers, follow this policy:

1. **Routing = WireGuard allowed_ips only.** Route a remote subnet only by
   adding it to the owning peer's `allowed_ips` (with `route_allowed_ips
   '1'`). No static routes, `ip route add` in rc.local, or helper scripts.
2. **One owner per prefix.** A subnet may appear in exactly one peer per
   WireGuard interface. Duplicates cause silent blackholes.
3. **Never merge or drop an existing WireGuard interface.** Only edit peer
   `allowed_ips` for routing.
4. **Permanent setup only** — every change must be in uci so it survives
   reboot.
5. **Always use rpcd auto-rollback** (`ubus call uci apply
   {"rollback":true,"timeout":120}`). Confirm only after checking from
   **outside**.
6. **Known pitfalls:**
   - `netifd reload` starts a **second netifd daemon** — use
     `/etc/init.d/network reload` instead.
   - Hotplug helpers still run if renamed `.bak` — move them **out** of
     `/etc/hotplug.d/`.
   - WireGuard `route_allowed_ips` can override local interface routes
     after reboot if the own LAN is in a peer's allowed_ips.
7. **Out-of-band:** VMs use `qm guest exec` on the PVE host. Native
   routers have no OOB — only the rollback timer protects them.

## MT5 Container SSH Access

The MT5 container can reach all mesh hosts for deployment:

- **Key:** `/home/kasm-user/.ssh/kandev` (fingerprint
  `SHA256:/1xfmrDI0Y4EjdD…`)
- **Config:** `/home/kasm-user/.ssh/config` — tunnel IPs for hub hosts,
  LAN IPs for proxies, `Host *` with ECDH-only KEX.
- **Post-quantum KEX fix:** OpenSSH 10.x defaults to
  `mlkem768x25519-sha256` (~1250 B packets). owrt002's DSL path MTU is
  ~1428 B — SSH hangs. Force ECDH-only algorithms in `~/.ssh/config` and
  `mesh-test.sh`.

---

# SSH Access & Credentials

## Shared Key

All hosts use the same shared **`kandev`** ed25519 key
(`SHA256:/1xfmrDI0Y4EjdD…`). The private key is installed on both hub
sandboxes (`~/.ssh/kandev`) and inside the MT5 container
(`/home/kasm-user/.ssh/kandev`). **Root key login is disabled** on both
hubs (root `authorized_keys` empty); always use the `ubuntu` user and
`sudo` on hub hosts.

## Access Matrix

| Host | IP (reachable from sandboxes) | Login user | SSH key | Reachable from |
|---|---|---|---|---|
| **nbg-1** | `152.53.118.212` (public), `10.1.1.1` (mesh) | `ubuntu` / `kandev@…:2222` | `kandev` key | Both sandboxes, public IP |
| **vie-1** | `152.53.35.177` (public), `10.2.1.1` (mesh) | `ubuntu` / `kandev@…:2222` | `kandev` key | Both sandboxes, public IP |
| **Kandev sandbox (NBG)** | `172.26.0.2` (bridge), `127.0.0.1:2222` (host) | `kandev` / `root` | `kandev` key | nbg-1 loopback, both sandboxes via `:2222` |
| **Kandev sandbox (VIE-1)** | `172.26.0.2` (bridge), `127.0.0.1:2222` (host) | `kandev` / `root` | `kandev` key | vie-1 loopback, both sandboxes via `:2222` |
| **owrt002** | `10.1.1.2` (NBG) / `10.2.1.2` (VIE-1) | `root` | `kandev` key | Both sandboxes (direct, mesh) |
| **owrt001, 003, 011, 018, 031, 032, 033** | `10.1.1.x` (NBG) / `10.2.1.x` (VIE-1) | `root` | `kandev` key | Owning hub's sandbox (direct) |
| **pve103** | `192.168.15.7` | `root` | `kandev` key | Both sandboxes (via owrt002) |
| **pve104** | `192.168.15.6` | `root` | `kandev` key | Both sandboxes (via owrt002) |
| **vm103** (docker-103) | `192.168.53.111` | `ubuntu` | `kandev` key | Both sandboxes (via owrt002) |
| **vm104** (docker-104) | `192.168.54.111` | `ubuntu` | `kandev` key | Both sandboxes (via owrt002) |
| **MT5 container** (vm104) | `192.168.54.111:10190` (KasmVNC web) | `kasm-user` (VNC), `root` (container ssh) | `kandev` key | Both sandboxes (via vm104), KasmVNC web UI |
| **owrt-fsn-1, hel-1, gra-1, lim-1, eri-1, waw-1** | `10.1.1.x` (NBG) / `10.2.1.x` (VIE-1) | `root` | `kandev` key | Both sandboxes (direct, mesh) |
| **fsn-1, hel-1** (Proxmox) | `144.76.14.155` / `65.109.122.228` (public) | `root` | `kandev` key | Both sandboxes (public IP) |
| **gra-1, lim-1, eri-1, waw-1** (Proxmox) | Various public IPs | `root` | `kandev` key | Both sandboxes (public IP) |
| **gpu-1** | `194.182.188.115` | `ubuntu` / `root` | `kandev` key | Both sandboxes (public IP) |
| **gpu-2** | `92.39.59.7` | `ubuntu` / `root` | `kandev` key | Both sandboxes (public IP) |
| **ber-1, ber-2** (IONOS) | `217.154.145.10` / `85.215.213.123` | `root` | `kandev` key | Both sandboxes (public IP) |

## SSH Aliases (sandbox `~/.ssh/config`)

Both sandboxes carry short aliases for every host. The same alias name
resolves to the correct IP per sandbox:

| Alias | Resolves to (NBG sandbox) | Resolves to (VIE-1 sandbox) |
|---|---|---|
| `nbg-1` | `10.1.1.1` (mesh) | `10.1.1.1` (mesh via interhub) |
| `vie-1` | `10.2.1.1` (mesh via interhub) | `10.2.1.1` (mesh) |
| `owrt002` | `10.1.1.2` | `10.2.1.2` |
| `pve103` | `192.168.15.7` | `192.168.15.7` |
| `pve104` | `192.168.15.6` | `192.168.15.6` |
| `vm103` | `192.168.53.111` | `192.168.53.111` |
| `vm104` | `192.168.54.111` | `192.168.54.111` |
| `gpu-1` | `194.182.188.115` | `194.182.188.115` |
| `gpu-2` | `92.39.59.7` | `92.39.59.7` |

## Access Paths

**From a hub sandbox (NBG or VIE-1):** all mesh hosts are reachable
directly — no ProxyJump needed. The sandbox egress is MASQUERADED by
Docker to the host's `wg0` IP (`10.1.1.1` NBG / `10.2.1.1` VIE-1).

**From the MT5 container on vm104:** SSH access to all mesh hosts works
with the same `kandev` key. Public IPs time out from vm104 (behind
owrt002's NAT); reach hosts via their mesh tunnel IPs instead.

```sh
# From any sandbox, reach any mesh host by alias:
ssh owrt002                     # -> root@10.x.1.2 (per sandbox)
ssh ubuntu@vm104                # -> docker-1-pve-104
ssh root@pve103                 # -> Proxmox host behind owrt002

# Reach the MT5 container's host:
ssh ubuntu@vm104
# Then on vm104:
docker exec -it mt5-actr-s01-p104-1 bash

# Direct container SSH (from sandbox, two-hop):
ssh -J vm104 -p 2222 kandev@127.0.0.1   # -> kandev104 sandbox

# From the MT5 container itself (deploy pipeline):
./deploy.sh                     # auto-forwards over mesh
```

**To reach the KasmVNC desktop:**
Open `https://mt5-actr-s01-p104-1.<hub FQDN>` in a browser (Basic
`mesh-admin` first, then Authelia two-factor), or connect directly inside
the mesh at `http://192.168.54.111:10190/`.

## Access Limitations

- **`a1ars021`** is explicitly excluded — never connected, owner-excluded.
- **Public `2222/tcp`** on both hubs is whitelist-scoped (298 sources),
  filtered in DOCKER-USER (`KANDEV-SSH-WL`), not by ufw.
- **Direct VIE-1 route to `10.1.1.0/24`** cannot be assumed without the
  reciprocal `wg_interhub` client.
- **Root key login** is disabled on nbg-1 and vie-1 (empty
  `authorized_keys`). Always use the `ubuntu` user + `sudo` on hub hosts.
- **OpenWrt routers** use dropbear; the `kandev` key is authorized for
  `root`. Password authentication is disabled on all.
- **Public Proxmox hosts** (fsn-1, hel-1, gra-1, lim-1, eri-1, waw-1):
  SSH on port 22, whitelisted to 298 sources; password auth disabled.

---

# Container & Service Management

## Docker Service Policy

1. **No `latest` tag.** Use `tag@sha256:<digest>`.
2. **Start only via Docker Compose** in `/home/ubuntu/docker/<service>/`.
   Log limits `json-file` 10m x 3, restart `unless-stopped`.
3. **No published host ports** for web services (SEC-21: Docker DNAT
   bypasses ufw). Attach to a dedicated bridge; NPM proxies to the fixed
   bridge IP.
4. **Exposure only through NPM**, gated by Authelia two-factor. Add an
   NPM Basic list (`mesh-admin`) when the service has no login of its own.
   MT5 container ports are LAN-bound by design (`192.168.54.111:10190`).
5. **Secrets** only in a 600 `.env` on the host; never in md files or repo.
6. **Exposure check** before and after: `docker ps` ports, `ss -tlnp`,
   `ufw status` count unchanged, probe the public hostname from the other
   hub.

## UFW / Firewall Notes

- Both hubs filter Docker-published ports in `DOCKER-USER`, not ufw input
  rules (SEC-21).
- `WG-EASY-HOST` chain in `DOCKER-USER` isolates wg0 mesh traffic.
- Public ports: only 22/443/2222/51820 TCP/UDP are open (whitelisted: 19
  fixed IPs + 279 provider CIDRs). Port 80 is explicitly denied.
- WireGuard MTU is 1320 across all peers.

## NPM Add Proxy Host

For new MT5 NPM hosts, use the helper on both hubs:
```bash
sudo /usr/local/sbin/npm-add-proxy-host <prefix> <ip> <port> [template]
```

Clones a template NPM row, adds to Authelia two-factor, writes backups,
regenerates configs, restarts authelia.

## Mesh Test

`scripts/check-hosts/mesh-test.sh` (on both Kandev sandboxes) covers ping
and key-only SSH to all commissioned peers. Run from either sandbox after
any MT5 container or routing change.

---

# MT5 Broker Installers

| Broker | Installer | MT5 Dir | Notes |
|---|---|---|---|
| ActivTrades | `activtrades5setup.exe` (7.2 MB) | `MetaTrader 5 - ActivTrades` | No silent flags (`/auto`/`/S` ignored) |
| FxPro | `fxpro5setup.exe` (7.2 MB) | `MetaTrader 5 - FxPro` | Requires KasmVNC GUI |
| XM | `xm5setup.exe` (5.0 MB) | `MetaTrader 5 - XM` | Requires KasmVNC GUI |

All installers work correctly under Wine 11.20 Staging.

---

# MT5 Container Image Versioning Policy

## Tag Convention

Use **semantic versioning** (`MAJOR.MINOR.PATCH`) for Docker image tags:

| Change | Bump | Example |
|---|---|---|
| Dockerfile change, new base image, Wine upgrade | `MAJOR` | `1.0.0` → `2.0.0` |
| New DLL version, new MQL5 expert features | `MINOR` | `1.0.0` → `1.1.0` |
| Config fix, dependency update, no functional change | `PATCH` | `1.0.0` → `1.0.1` |

## Rules

1. **No `:latest` tag.** Every image must have an explicit version tag. Remove `:latest` after retagging:
   ```sh
   docker tag <image>:<old-ver> <image>:<new-ver>
   docker rmi <image>:latest   # if it existed
   ```
2. **Update compose files.** After retagging, update `image:` in the container's `docker-compose.yml`.
3. **Clone preserves version.** `mt5-clone-container.sh` carries over the source image's tag. If the source has `:latest` or no tag, it defaults to `1.0.0`.
4. **Build archives** (`archive/{env}/{build}/`) in the deploy pipeline use the build number (not the Docker tag) for versioning — those are independent.

## Version Tracking in MariaDB

The deployed DLL version is tracked in `expert_keepalive.dll_version` and `mt5_experts.dll_version`. The deploy pipeline cross-checks this against the built DLL version after every deploy.
