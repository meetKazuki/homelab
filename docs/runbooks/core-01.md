# `core-01` Runbook

> **Node:** Raspberry Pi 4 Model B (formerly `nexus-pi` / `NeXus-Pi4`)
> **Status:** Live document — reflects actual verified state, not intended state
> **Last updated:** 2026-06-21
> **Predecessor:** `nas-01` standardisation (reference for patterns/pitfalls)
> **Naming convention:** `naming-convention-v1.4.md`

---

## 1. Hardware & OS

| Item | Value |
|---|---|
| Device | Raspberry Pi 4 Model B Rev 1.5 |
| RAM | 3.7Gi total (4GB model, confirmed) |
| OS | Debian GNU/Linux 13 (trixie) — genuine Raspberry Pi OS (`pi-gen` build, `raspberrypi-sys-mods`/`raspberrypi-net-mods` installed; `os-release` branding alone was misleading) |
| Kernel | `6.18.29+rpt-rpi-v8` (arm64) |
| Root filesystem | `/dev/sda2` — USB/SSD-attached, 117G total, ~93G free at audit time (not SD card; doc's space concern doesn't apply here) |
| Boot partition | `/dev/sda1` on `/boot/firmware`, 510M |
| Static IP | `192.168.50.3` (unchanged through this entire thread) |

## 2. Hostname

- **Current:** `core-01` (renamed from `NeXus-Pi4` — note original was mixed-case, not lowercase `nexus-pi` as the handoff doc assumed)
- **Persistence mechanism:** This box is provisioned via **cloud-init** (NoCloud datasource, `seedfrom: file:///boot/firmware`). The live hostname alone is not sufficient — `/boot/firmware/user-data` is the actual source of truth and gets re-applied by `update_hostname` (frequency: `ALWAYS`) on every boot.
- **Fix applied:** Edited `hostname: NeXus-Pi4` → `hostname: core-01` directly in `/boot/firmware/user-data`, in addition to live `hostnamectl set-hostname` and `/etc/hosts` edit.
- **Verified:** Survived a full reboot — `hostnamectl`, `/etc/hostname`, `/etc/hosts` all consistent post-reboot. DNS resolution via AdGuard independently confirmed working from an external client post-rename (`nslookup` against `192.168.50.3`).
- **Relevant cloud-init config:** `/etc/cloud/cloud.cfg.d/99_raspberry-pi.cfg` (NoCloud datasource config); `preserve_hostname: false` in main `cloud.cfg`.

## 3. User Accounts

| Account | UID | Type | Status | Access |
|---|---|---|---|---|
| `kazuki` | 1001 | Human, primary admin | **Active** | SSH key (copied from `nexus-pi`'s `authorized_keys`), `sudo NOPASSWD:ALL` |
| `svc-docker` | system | Service | **Active** | No login shell; group-owns `/opt/homelab/` content |
| `nexus-pi` | 1000 | Human, original admin | **Disabled** (not deleted) | `passwd -l` (locked) + shell set to `/usr/sbin/nologin`. Retained because its home directory still holds the archived HA config backup and the still-running `influxdb3-core`/`speedtest-tracker` data, pending Proxmox migration. |

**Design decisions:**
- Created `kazuki` fresh rather than renaming `nexus-pi` in place — mirrors the precedent already set on `nas-01` (original admin account retained as dormant fallback rather than renamed/removed), and avoided the single-point-of-failure risk of renaming the only working admin account on a DNS-critical box mid-process.
- Single `svc-docker` account rather than the handoff doc's proposed `svc-iot`/`svc-backup` split — four lightweight, heterogeneous containers didn't justify role separation.
- `kazuki` sudo is `NOPASSWD:ALL` by explicit choice (convenience prioritised over the extra confirmation step), unlike `nexus-pi`'s blanket `NOPASSWD:ALL` which was inherited rather than chosen — functionally identical, but now a deliberate decision either way.

## 4. Docker Stack Directory Structure

**Location:** `/opt/homelab/` (not `/home/kazuki/homelab/` — root partition has ample space, unlike the NAS's TOS constraint; also not `/opt/stacks/` per your naming preference, consistent with the NAS's `homelab` convention).

**Ownership:** `kazuki:svc-docker`, setgid (`g+s`) applied at the top level and each service subdirectory so newly created files inherit the `svc-docker` group automatically. Note: setgid only auto-propagates to *direct* children of a setgid directory — deep nested copies (e.g. via `cp -a`) still needed an explicit recursive `chown` to guarantee consistency.

```
/opt/homelab/
├── homeassistant/
│   ├── compose.yml
│   ├── .env
│   └── config/        (fresh — see Section 5)
├── ntfy/
│   ├── compose.yml
│   ├── .env
│   ├── Makefile        (pre-existing per-service pattern, discovered during migration — not in original handoff inventory)
│   └── config/
├── uptime-kuma/
│   ├── compose.yml
│   ├── .env
│   └── Makefile
│   (data lives in named Docker volume `uptime-kuma-data`, untouched by directory migration)
└── portainer-agent/
    ├── compose.yml
    ├── .env
    └── Makefile
```

**Note on `service-up.sh`:** Explicitly excluded from evaluation per instruction. Its likely replacement — per-service `Makefile`s — already exist and predate this thread; this appears to resolve the handoff doc's open question about `service-up.sh` vs. a Makefile pattern without further action needed.

## 5. Home Assistant — Rebuilt Fresh (Deliberate Data Loss)

**This was an explicit decision, not an accident or oversight.** The previous `homeassistant/config/` (143M, containing all automations, entity registry, history, and device pairings — Tuya, TV, power monitoring) was **archived, not migrated**:

```
/home/nexus-pi/homeassistant-OLD-config-backup/   ← archived, untouched
/opt/homelab/homeassistant/config/                ← fresh, empty at deployment
```

Compose file is otherwise unchanged from the original (host networking, `privileged: true`, `NET_ADMIN`/`NET_RAW`, USB/vhci device passthrough, dbus mount). `.env` carries over only `TZ=Africa/Lagos` — the old `.env`'s `PUID`/`PGID` values were unused dead config (the official HA image doesn't consume them) and were dropped.

**Consequence:** All device integrations (Tuya, TV, power monitoring) need re-pairing from scratch. This was understood and accepted at decision time.

## 6. Service Inventory (Final State)

| Service | Image | Port(s) | Location | Status |
|---|---|---|---|---|
| `homeassistant` | `ghcr.io/home-assistant/home-assistant:stable` | host networking | `/opt/homelab/homeassistant/` | Up, fresh install |
| `ntfy` | `binwiederhier/ntfy` | 8085 | `/opt/homelab/ntfy/` | Up, healthy, data migrated |
| `uptime-kuma` | `louislam/uptime-kuma:2` | 3001 | `/opt/homelab/uptime-kuma/` | Up, healthy, monitors confirmed intact post-migration |
| `portainer-agent` | `portainer/agent:2.33.5` | 9001 | `/opt/homelab/portainer-agent/` | Up |
| `AdGuard Home` | native binary (not Docker) | 53, web UI | `/opt/AdGuardHome/` | Up — see Section 7 |
| `influxdb3-core` | `influxdb:3-core` | 8181 | `/home/nexus-pi/homelab/influxdb3-core/` | **Still running on Pi** — migration to Proxmox deferred to Proxmox normalisation thread |
| `influxdb3-explorer` | `influxdata/influxdb3-ui:1.4.0` | 8888 | (undocumented in original handoff — discovered live via `docker ps -a`) | **Still running on Pi** — moves with `influxdb3-core` |
| `speedtest-tracker` | `lscr.io/linuxserver/speedtest-tracker:latest` | 8080/8443 | `/home/nexus-pi/homelab/speedtest-tracker/` | **Still running on Pi** — migration deferred |
| `beszel-agent` | native binary at `/opt/beszel-agent/` | — | n/a | Reports to a Beszel monitoring hub on Proxmox. Not in original handoff inventory; confirmed legitimate, no action taken |

**Confirmed:** No active Home Assistant integration dependency on InfluxDB — clean to migrate without a cutover sequencing concern.

## 7. AdGuard Home

| Item | Detail |
|---|---|
| Install type | Native binary, systemd-managed — **not Docker** |
| Location | `/opt/AdGuardHome/` |
| Version | `v0.107.77` |
| Config | `AdGuardHome.yaml` — `600`, root-owned |
| Data | `data/` — `700` |
| Backup | `agh-backup/` — `700` |
| Runs as | `root` (no `User=` in unit file) |
| Service | `systemd`, `enabled`, `Restart=always`/`RestartSec=10` |
| Update mechanism | Self-updating via built-in updater (uses `AdGuardHome.sig` for verification against GitHub releases) — not `apt`/`dpkg` |
| Permissions fix | Top-level directory found `777` at audit time, tightened to `755` (safe — service runs as root, unaffected by removing world-write) |
| DNS records | No changes made. `core-01.lan`/`core-01.internal.kazuki.uk` (from the original handoff doc's proposed table) were speculative entries not matching any real, currently-used convention — explicitly declined. `ntfy` already resolves externally via `ntfy.kazuki.uk` (outside AdGuard's scope). `homeassistant`/`uptime-kuma` are reached via Tailscale, no local DNS needed. |

## 8. Backup (Backrest / rclone SFTP)

**Status: Explicitly deferred, not verified in this thread.** The handoff doc flagged that the rename could break the existing rclone SFTP mount from the Proxmox backup VM (if it references the old hostname rather than the static IP). The static IP (`192.168.50.3`) did **not** change — only the hostname — so risk is lower than the doc anticipated, but this has **not been confirmed**. You've indicated you'll address this on your own timeline; it is not a blocker on this thread's completion, but it is a genuinely open item.

## 9. Open Items (Carried Forward)

- [ ] Verify/fix Backrest rclone SFTP mount post-rename (deferred by choice)
- [ ] Decommission `influxdb3-core`, `influxdb3-explorer`, `speedtest-tracker` on the Pi side — blocked on Proxmox normalisation thread deploying their replacements first
- [ ] Re-pair Home Assistant integrations (Tuya, TV, power monitoring) — consequence of the deliberate fresh install
