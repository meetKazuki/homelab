# NAS Standardisation Runbook — `tnas-f4-424` → `nas-01`

> **Status:** Draft for execution. Built from verified diagnostics (TOS 7.0.0706), not assumptions from the original handoff.
> **Deviations from `handoff-nas-standardisation.md` are called out explicitly** — see §0.

---

## 0. Deviations from the original handoff (read this first)

The handoff assumed a generic Linux layout. The actual box is TOS 7.0.0706 (Debian-based, real kernel, but TOS manages users/SSH/apps through its own layer on top). Confirmed facts that change the plan:

| Handoff assumption | Reality | Resolution |
|---|---|---|
| `/opt/stacks/` for Docker stacks | Root partition (`/dev/md9`) is only 7.5GB; real storage is `/Volume1` (3.7TB) | Use `/Volume1/@apps/homelab/` (per your call — "homelab" not "stacks") |
| `kazuki` becomes primary admin via rename | Current admin `nexus-tnas` is **UID 0** (root under an alias) — not a normal sudo user | Create a **new**, separate unprivileged `kazuki` account. Do not touch `nexus-tnas`'s identity. |
| `svc-backup`, `svc-media` as plain `useradd --system` | TOS users may need to exist in TOS's own user DB (GUI) for SSH/PAM to recognize them reliably, and TOS has a habit of overwriting hand-edited configs on its own service toggles | Create via TOS GUI where possible; verify survival after a GUI touch + reboot before considering it done |
| `kazuki` verified via live SSH login test | **There is no SSH daemon running on this NAS at all, and never has been.** What we'd been calling "SSH access" the entire time was actually TOS's browser-based web terminal (confirmed: `$SSH_CONNECTION`/`$SSH_TTY` empty, `w`/`who` show 0 logged-in TTY sessions, no `sshd` process exists, no host keys were ever generated, `/run/sshd` didn't exist). The VLAN/OPNsense angle was a red herring — investigated and ruled out along the way, but the real explanation is simpler: nothing was ever listening on port 9222. | `kazuki`'s actual, working access method is the **TOS web GUI and its built-in web terminal** — same as `nexus-tnas` has always used. Every place this runbook says "SSH in as `kazuki`" should be read as "log in via TOS web GUI/terminal as `kazuki`." sshd-related edits (`AllowUsers`, `PermitRootLogin`, host keys, `/run/sshd`) were explored and partially fixed along the way, but are **not required for `kazuki`'s day-to-day access** and that work is now deprioritized — left in a "config corrected, daemon not started" state rather than fully brought up, since there's no current need to run sshd at all for this purpose. The one place a real, running sshd genuinely matters is `svc-backup` (Step 4) — Backrest connects from the Docker VM over actual SSH, not a browser, so sshd needs to be properly started and working for that specific case, scoped narrowly rather than as a general access mechanism. |
| Plex migrates into a compose stack | Plex is a native TOS app with a dedicated system user (`plex`, UID 10001) using HW transcode groups (`video`, `render`) | Plex stays as-is; documented as an exception, not migrated |
| All three services get `*.kazuki.uk` DNS records | Jellyfin/Plex are behind Traefik (which backends to the NAS) and keep their existing `kazuki.uk` records unchanged. slskd is **not** proxied and has no DNS record at all — it's accessed the same way as the rest of the arr stack (raw IP:port), and should stay that way. | No DNS changes needed for Jellyfin/Plex. The handoff's draft `slskd.kazuki.uk` record is dropped — see Step 8 for the corrected reasoning. |
| `nexus-pve` share renamed for branding cleanup | Contains the Proxmox VM backup target, not just media — mixed-use, higher blast radius | **Deferred**, per your call. Out of scope for this thread. |
| Handoff's arr-stack table lists both "Jellyseerr" and "Seerr" as separate services | Seerr is the unified successor project to Jellyseerr/Overseerr (post-merge) — confirmed you're only running Seerr | **Jellyseerr dropped from the service list entirely** — stale leftover from before the upstream project merge, not a separate thing to migrate. |
| Stack directory holding Gluetun + qBittorrent named for the dependency owner (`gluetun/`) | User finds it easier to reason about qBittorrent and slskd as "the two downloaders," with Gluetun as a supporting/background dependency rather than a peer-level service | Directory named `qBittorrent/` instead — flagged as a deliberate departure from the "name for what depends on what" convention used elsewhere, not an inconsistency to fix later. |
| `media`/`downloads` ownership assumed safe to standardize freely | Files under `/Volume1/nexus-pve/` (`media`, `downloads`, plus the Proxmox-only `dump`/`images`/`template`) were owned by an orphaned UID `1003` with no matching `/etc/passwd` entry. Initially looked like it might be a deliberate Proxmox NFS UID-mapping requirement (Proxmox does actively mount this exact share per `/etc/pve/storage.cfg`'s `nfs: tnas-f4-424` entry) — investigated via the NAS's actual `/etc/exports`, which showed `no_root_squash` and `sec=sys` for all Proxmox/Docker-VM clients, meaning NFS passes through whatever UID the client presents rather than enforcing `1003`. Concluded `1003` was incidental (leftover from whatever process wrote those files previously), not a live cross-system contract. | `media` and `downloads` — pure media-serving paths, never touched by Proxmox — `chown -R`'d to `svc-media:allusers` (1006:997), verified complete via `find ... ! -user svc-media` returning empty. **`dump`, `images`, `template`, and the share root deliberately left at `1003`/`nexus-tnas`** — Proxmox-only territory, out of scope for this thread, consistent with the already-deferred `nexus-pve` rename decision. |
| `sudo` on this box behaves like normal Linux `sudo` | **Critical, box-wide finding**: `sudo`-as-`kazuki` resolves to `root` at **UID 9999** — a separate, neutered placeholder account distinct from the box's actual root, `nexus-tnas` at **UID 0**. `sudo whoami` returns `root` either way, masking the distinction. This means `sudo` from any non-`nexus-tnas` account **cannot** successfully chown/modify anything owned by `nexus-tnas` (UID 0), even though it appears to have full root capabilities (`capsh --print` shows `cap_chown` present in the bounding set — the failure isn't a missing capability, it's that UID 9999 genuinely isn't the owning UID). Confirmed via repeated, reproducible `chown: Operation not permitted` failures despite `sudo whoami` → `root`. | **Any ownership-changing operation on files owned by `nexus-tnas`/UID 0 must be run by logging in as `nexus-tnas` directly** (via TOS web terminal, which doesn't depend on SSH lockout state, or via SSH if `PermitRootLogin` is deliberately and temporarily enabled) — never via `sudo` from `kazuki` or any other account. This is a standing fact about this box, not a one-time bug, and applies to all future steps. |

---

## 1. Pre-flight: what you already have (verified, not guessed)

```
Hostname:        NeXus-TNAS-F424   → target: nas-01
TOS version:     7.0.0706
Kernel:          6.12.63+ (real Debian-based Linux, not a sandboxed appliance)
Docker:          29.4.0, Docker Compose v5.1.2
Docker root:     /Volume1/@apps/DockerEngine/
Storage:         /Volume1 — 3.7TB BTRFS, 57% used
Running containers: jellyfin, slskd, glances, beszel-agent, portainer-agent
Admin account:   nexus-tnas (UID 0 / GID 0 — this IS root, not a normal admin)
Existing root:   UID 9999, /usr/sbin/nologin (TOS's neutralized placeholder)
Plex:            native TOS app, system user `plex` (UID 10001), groups: video, render
SSH:             port 9222 configured in sshd_config, but NO daemon actually running (no host keys, no /run/sshd, no sshd process) — "SSH access" was always the TOS web terminal, not real SSH

```

Current bind mounts (messy, inconsistent — this is part of what we're fixing):

| Container | Mount | Path |
|---|---|---|
| jellyfin | media | `/Volume1/nexus-pve/media` |
| jellyfin | config | `/home/nexus-tnas/Documents/homelab/jellyfin/config` |
| slskd | downloads | `/Volume1/nexus-pve/downloads` |
| slskd | media | `/Volume1/nexus-pve/media` |
| slskd | config | `/home/nexus-tnas/Documents/homelab/slskd/config` |

Configs live scattered in the admin's home directory. We're consolidating these into `/Volume1/@apps/homelab/<stack>/`.

---

## 2. Execution order

Each step has a **verify** sub-step. Do not proceed to the next numbered step until the verify passes. Report back at each checkpoint marked **STOP**.

### Step 1 — Create the `kazuki` account (before touching anything else) — ✅ COMPLETE

**Done via TOS GUI**, so TOS's own user DB and the Unix layer stay in sync:

1. TOS Control Panel → Access Permissions → Users → Create. ✅
2. Username: `kazuki`. ✅
3. Added to the `admin` group (confirmed via Create User flow, group assignment screen showed `admin` ticked). ✅
4. Confirmed at the Unix level via `id kazuki` / `grep kazuki /etc/passwd /etc/group` — UID 1005, primary group `allusers` (997), secondary `admin` (998). ✅

**Access method: TOS web GUI and its built-in browser terminal — not SSH.** Mid-runbook we discovered there is no SSH daemon running on this NAS, and never has been; what looked like an "SSH session" the whole time was TOS's web terminal widget (confirmed via empty `$SSH_CONNECTION`/`$SSH_TTY`, `w`/`who` showing 0 TTY logins, and no `sshd` process existing at all). `nexus-tnas` has only ever used this same web-based method too — so `kazuki` is now equally functional via the same path, and that's a complete, working result.

An SSH key was generated and staged on disk (`/home/kazuki/.ssh/authorized_keys`, correct ownership `kazuki:allusers`, perms `700`/`600`) in case sshd is ever brought up for this account specifically — but this is **not required** for `kazuki` to be considered done, since the web GUI/terminal is the actual access method going forward (per explicit decision). Treat the SSH key as inert, pre-staged groundwork, not a blocker.

**This step is complete.** No further action needed.

---

### Step 2 — `nexus-tnas` (root) access — deferred, no longer urgent

**Original goal:** close `PermitRootLogin yes` as an active risk now that `kazuki` exists as an alternative admin path.

**Revised, given what we now know:** since there is no running sshd at all, `PermitRootLogin yes` in `/etc/ssh/sshd_config` is currently inert — it only matters if/when sshd is actually started and reachable. The risk this step was meant to close doesn't exist in practice right now. `nexus-tnas`'s actual access (the web GUI/terminal) is unaffected by anything in `sshd_config`.

**What's already been written to the config file** (not yet live, since sshd isn't running to reload):
```
AllowUsers kazuki
PermitRootLogin no
```
This is harmless to leave in place as-is. **No action needed right now.** If sshd is ever deliberately brought up in the future (e.g. scope expands beyond just `svc-backup`), revisit this file at that time to confirm it still reads the way you want — don't assume it's been "applied" just because it's written, since nothing has reloaded it.

`nexus-tnas`'s username stays as-is (per earlier decision) regardless of SSH status.

---

### Step 3 — Rename the host

1. TOS Control Panel → general/network settings → change hostname to `nas-01`.
2. Confirm via `hostname` and `uname -a` after the change takes effect (may require a reboot — TOS-dependent).

**STOP — verify**, then check for breakage per the convention's own migration notes (§8 of `naming-convention-v1.md`):
- SSH known_hosts on your client(s) will need updating for the new name (key itself shouldn't change, just the alias).
- If Tailscale is running on this NAS, rename the device in Tailscale admin to match.
- If anything currently references `tnas-f4-424` or `NeXus-TNAS-F424` by name (scripts, monitoring, PBS labels) — grep for it. You mentioned Proxmox storage IDs (`nexus-tnas-pbs`, `tnas-f4-424`) are out of scope for this thread, so leave those, but flag anywhere *this* box's own scripts/configs reference the old name.

---

### Step 4 — Service accounts: `svc-backup` and `svc-media`

Two accounts, created the same GUI-first way as `kazuki`.

**`svc-media`** — ✅ **COMPLETE.** No SSH/sshd involvement at all; purely a file-ownership account for container permissions (Step 6).
1. Created via TOS GUI (Privileges → User → +). Not added to `admin`.
2. UID 1006, GID 997 (`allusers`), shell already defaulted to `/usr/sbin/nologin` by TOS — no extra lock step needed.
3. **UID:GID = `1006:997` — this is what Step 6 needs.**

**`svc-backup`** (for Backrest, Option A — remote SSH from the Docker VM's Backrest container) — ⚠️ **STAGED, NOT YET VERIFIED. See blocker at the end of this section before treating this as done.**

1. Created via TOS GUI, not added to `admin`. UID 1007, GID 997 (`allusers`), shell already `/usr/sbin/nologin` by TOS default.
2. Share permissions left unset for now (per decision) — `/Volume1/@apps/homelab/` doesn't exist yet (that's Step 5), so Read Only access there will be granted once it exists.
3. **sshd bring-up** — this NAS had never run sshd before this thread (no host keys, no `/run/sshd`, no process). What we actually found and did, in order:
   - Host keys generated: `ssh-keygen -A`.
   - `/run/sshd` created manually (`mkdir -p /run/sshd && chmod 0755`) — **this is tmpfs and will not survive a reboot** unless something recreates it at boot. Untested across a reboot so far — flag as a follow-up check.
   - `sshd -t` passed once both of the above existed.
   - The daemon is **not** systemd-managed despite this being a broadly Debian-based system — it's a SysV init script (`/etc/init.d/ssh`), which itself shells out to `systemctl` under the unit name `ssh.service` (not `sshd.service`, which is why our first attempt at `systemctl reload sshd` failed with "unit not found").
   - The daemon was actually **started** by enabling "Allow SSH access with username and password" in TOS's GUI (Control Panel → Terminal & SNMP). **Important, confirmed behavior:** this GUI toggle rewrites `sshd_config`'s `AllowUsers` line back to TOS's own default (`nexus-tnas` only), discarding any manual edits. This happened once already, silently dropping both `kazuki` and `svc-backup` from the allowlist. **Any future touch of this GUI panel will likely do it again — re-check `AllowUsers` every single time after using it.**
   - Current working state: `AllowUsers nexus-tnas kazuki svc-backup`, confirmed listening on `0.0.0.0:9222` and `[::]:9222`.
4. SSH key: generated **on the Docker VM**, inside Backrest's own persistent volume (not the VM host's general filesystem) — `/home/nexus-vm/homelab/backrest/data/ssh/svc-backup_nas01` (passphrase-less, appropriate for an automation key with locked-down `600` perms; matches the path Backrest's container sees internally as `/data/ssh/svc-backup_nas01`). Public key copied into `svc-backup`'s `authorized_keys` on the NAS.
5. **Home directory permissions fix required and applied**: `/home/svc-backup` was created `drwxrwxrwx`, owned by `nexus-tnas` — this is a TOS default for new user home dirs that breaks `StrictModes` (silently rejects key auth, falls back to password, no error). Fixed: `chown svc-backup:allusers /home/svc-backup && chmod 750 /home/svc-backup`. **`kazuki`'s home directory has the same `drwxrwxrwx` pattern and was never fixed** — flagged as an open follow-up, not yet resolved.
6. Tested key auth from inside the Backrest container itself (`docker exec backrest sh -c "ssh -i /data/ssh/svc-backup_nas01 ..."`) — **key was accepted** (no password prompt, `StrictModes` passed after the home-dir fix), but the connection returned **"This account is currently not available"** — expected, since the shell is `/usr/sbin/nologin`, which blocks all command execution including SFTP.
7. Confirmed via Backrest's own "Add Repository" UI that it expects an `sftp:user@host:/repo-path` URI — restic's SSH backend is SFTP-only, never executes a remote shell. So the fix is to grant SFTP specifically, not a general shell.
8. Added to `/etc/ssh/sshd_config` (must be the last block in the file):
   ```
   Match User svc-backup
       ForceCommand internal-sftp
       ChrootDirectory /Volume1/@apps/homelab
       AllowTcpForwarding no
       X11Forwarding no
   ```
   `sshd -t` passed, reload via `/etc/init.d/ssh reload` succeeded, `AllowUsers` re-confirmed intact afterward.

**Update (post-Step 5):** attempted the real test. Connection and authentication succeed — `pwd` correctly returns `/`, confirming the chroot is active and `ChrootDirectory`'s ownership requirements (verified clean: `/Volume1`, `/Volume1/@apps`, `/Volume1/@apps/homelab` all `root`-owned, `755`) are satisfied. **But `ls` inside the SFTP session returns nothing, and `cd jellyfin` fails with "No such file or directory"** — even though `jellyfin`/`slskd`/`media`/`qBittorrent` all visibly exist when checked via `sudo -u svc-backup ls` from outside the chroot. **Root cause not yet found — investigation paused mid-stream, not resolved.** Leading candidates, not yet checked:
- A casing mismatch noticed but not confirmed: the `sudo -u svc-backup ls` output showed `qbittorrent` (lowercase) where the directory was deliberately created as `qBittorrent` — could be a display artifact or could be a genuine second, separately-created directory; unconfirmed.
- Possible traversal/execute-bit permission gap specific to a true separate SSH session vs. `sudo -u` impersonation (these can behave differently under some PAM/NSS setups) — a `namei -l` walk of the full path from `svc-backup`'s actual login context was queued but not run.
- Possible NFS/filesystem caching quirk specific to the `internal-sftp` subprocess.

**This is a real, open blocker — Backrest is not usable yet.** Do not mark this done. Resume investigation before actually relying on this backup path; in the meantime, treat the NAS as unbacked-up via this mechanism.
4. Grant `svc-backup` Read Only share permissions on `/Volume1/@apps/homelab/` via TOS's share permission system once it exists (Control Panel → Shared Folder → permissions) — per the convention's least-privilege rule, this account should not have write access outside the chroot/SFTP path Backrest actually needs.

---

### Step 5 — Build the new stacks directory structure

```
/Volume1/@apps/homelab/
├── jellyfin/            # Jellyfin + Jellystat together — Jellystat's only
│   │                    # real dependency is Jellyfin (polls its API for
│   │                    # stats), so it lives alongside it rather than
│   │                    # in a standalone directory or in media/
│   └── compose.yml
├── slskd/
│   └── compose.yml
├── media/               # Sonarr, Radarr, Lidarr, Prowlarr, Bazarr, Whisparr,
│   │                    # Aurral (real dependency: Lidarr specifically),
│   │                    # Seerr (bridges Jellyfin + arr stack, grouped here
│   │                    # since fulfillment runs through Sonarr/Radarr)
│   │                    # (NOT qBittorrent — see qBittorrent/ below)
│   │                    # (Jellyseerr dropped — stale; Seerr is its successor)
│   ├── compose.yml
│   ├── .env
└── qBittorrent/         # Gluetun + qBittorrent together, deliberately
    └── compose.yml      # qBittorrent uses network_mode: service:gluetun,
                          # so if Gluetun is down, qBittorrent has no network
                          # at all (correct — prevents VPN-bypass leaks)
                          # Directory named for qBittorrent rather than
                          # Gluetun — deliberate choice (user finds it
                          # easier to reason about qBittorrent + slskd as
                          # "the two downloaders," with Gluetun as
                          # qBittorrent's supporting dependency rather than
                          # a peer-level service). Container naming follows:
                          # qBittorrent-gluetun, qBittorrent-qbittorrent.
```

**Architecture note on `qBittorrent/`:** originally the handoff treated qBittorrent as part of the `media` stack with Gluetun as a separate dependency. Revised here, deliberately: qBittorrent's `compose.yml` lives in the `qBittorrent/` directory as part of the *same* compose project, using `network_mode: "service:gluetun"`. This means qBittorrent's container shares Gluetun's network namespace entirely — if Gluetun isn't running, qBittorrent has no network path at all, which is the correct fail-closed behavior for a VPN-gated download client (the alternative — qBittorrent in a separate stack reaching Gluetun over the LAN — could fail open if misconfigured, exposing real IP traffic). This also avoids cross-compose-project dependency ordering issues. Container naming will be `qBittorrent-gluetun` and `qBittorrent-qbittorrent` (project name prefix, slightly redundant for the second but acceptable per the convention's own naming rules). **Naming note:** the directory is named for qBittorrent rather than Gluetun, against the general "name the directory for what most things in it depend on" principle — a deliberate choice, not an oversight (see §0).

**Architecture note on `jellyfin/` and `media/` groupings:** rather than bundling every arr-adjacent tool into one bucket by name-association, each was checked against its actual runtime dependency. Aurral writes through Lidarr, not directly to the library — so it sits with the rest of the arr stack in `media/`. Jellystat only ever talks to Jellyfin's API — so it sits in `jellyfin/`, not `media/`, despite the superficial "stats/automation tool" resemblance to the arr stack. Seerr is the genuine exception — it reads from Jellyfin for library state and writes to Sonarr/Radarr for fulfillment, so it has a real foot in both camps; it's placed in `media/` since its fulfillment path is what actually drives downloads, but this is a judgment call, not a clean dependency match like the other two.

Note: **`plex/` is intentionally absent** — it's a native TOS app, not a compose stack (see §0).

For each existing container (jellyfin, slskd), this means:
1. Write a `compose.yml` that reproduces the current container config (image, ports, mounts), placed at `/Volume1/@apps/homelab/<service>/compose.yml`.
2. **Config migration: clean cutover, confirmed.** Move existing config dirs from `/home/nexus-tnas/Documents/homelab/<service>/config` into the new structure now, rather than leaving config split across two locations.
3. Also fix the media/downloads paths — see Step 5a below.

**Step 5a — `nexus-pve` share path**: since renaming that share is deferred, the new compose files will still bind-mount from `/Volume1/nexus-pve/media` and `/Volume1/nexus-pve/downloads`. That's fine — the compose-file/stack-directory naming is independent of the underlying share name. Just don't let the deferred rename block this work.

**Migration order, confirmed: stop-then-rebuild (not alongside-then-cutover).** This means brief downtime on jellyfin/slskd while we rebuild under the new structure — accepted tradeoff for simplicity over running two copies against the same media library simultaneously.

**✅ COMPLETE — executed, both services confirmed working.** Actual sequence, differing in a few places from the original plan above:

1. Directory structure created at `/Volume1/@apps/homelab/{jellyfin,slskd,media,qBittorrent}` (4 directories, not 5 — `jellystat` folded into `jellyfin/`, see architecture note above).
2. **Discovered mid-migration**: `nexus-pve/media` and `nexus-pve/downloads` (and the Proxmox-only `dump`/`images`/`template`) were owned by an orphaned UID `1003`. Investigated thoroughly given the stakes (Proxmox does mount this exact share per `/etc/pve/storage.cfg`) — confirmed via the NAS's actual `/etc/exports` (`no_root_squash`, `sec=sys`) that NFS doesn't enforce this UID, so it was safe to fix. `chown -R 1006:997` applied to `media` and `downloads` only; `dump`/`images`/`template`/share-root deliberately left alone (Proxmox's territory, out of scope). Verified complete via `find ... ! -user svc-media` returning empty — see §0 for the full writeup.
3. Existing `docker stop jellyfin slskd` — confirmed via `docker ps -a`, both `Exited`.
4. Rollback snapshots captured pre-stop: `docker inspect jellyfin/slskd > /tmp/*.json`.
5. **User moved entire `jellyfin/` and `slskd/` folders** (not just `config`) from `/home/nexus-tnas/Documents/homelab/` into the new structure — clean cut, old location fully removed. This surfaced pre-existing `jellyfin.yml`/`slskd.yml`/`.env` files alongside `config` that we hadn't known about — turned out to be the real, working prior compose definitions, not stale leftovers. Reused rather than rebuilt from scratch.
6. **Found both services were running with `PUID=0`/`PGID=0` (root)** in their original compose files — direct conflict with Step 6's goal. Fixed in the same pass rather than as a separate later step: both files rewritten with `PUID=1006`/`PGID=997` (svc-media), renamed `jellyfin.yml`/`slskd.yml` → `compose.yml` per convention, added `restart: unless-stopped`, preserved the Intel OpenCL GPU passthrough (`DOCKER_MODS`, `/dev/dri` device) untouched.
7. **Jellyfin's first start failed**: `System.UnauthorizedAccessException` — config directory had only been partially chowned (top-level dir updated, nested files/dirs like `.aspnet`/`.cache` still `1003`). Real lesson here: **a single-level `ls`/`chown` is not enough evidence of a clean recursive chown on this box — always verify with `find ... ! -user X` returning empty before trusting it.** Fixed with `chown -R 1006:997` on the full config tree, re-verified, restarted — clean startup, confirmed working by actually streaming media.
8. **slskd's config ownership was fixed *before* first start this time** (lesson from #7 applied immediately) — came up clean on the first attempt, no errors, shares loaded, connected to Soulseek.
9. **Soulseek username changed**: original config logged in as `nexus-tnas` (branding leakage onto the public Soulseek network, same pattern as other internal-naming cleanup in this thread). Since Soulseek usernames can't be renamed in place — confirmed via research, changing the username creates a brand-new account, abandoning any reputation/history tied to the old one — user explicitly chose to start fresh rather than keep `nexus-tnas`. New identity: `kazuki-nas`. Updated `SLSKD_USERNAME`/`SLSKD_PASSWORD` in `.env` (new password, not reused), restarted, confirmed login as `kazuki-nas` in logs.

**Key lesson carried forward for Step 7 (arr stack)**: check and fix container-process UID/GID and full-recursive config ownership *before* first start, not after — this cost real debugging time twice in this step alone.

---

### Step 6 — Apply `svc-media` ownership to media-serving containers

**✅ COMPLETE for jellyfin and slskd** — done as part of Step 5 rather than as a separate later pass, since we were already rewriting both compose files at that point (discovered both were running as root, `PUID=0`/`PGID=0`, and fixed it in the same edit). Both now run as `1006:997`, confirmed via successful startup and verified file ownership on disk (`find ... ! -user svc-media` returning empty after the full recursive chown).

**Still pending for Step 7's services** — Sonarr, Radarr, Lidarr, Prowlarr, Bazarr, Whisparr, Seerr, Aurral, qBittorrent, Gluetun, and Jellystat all need the same `PUID=1006`/`PGID=997` treatment when their compose files are written, plus the same lesson applied proactively: **set correct UID/GID and verify full-recursive directory ownership *before* first start**, not after — don't repeat the debugging cycle Step 5 went through twice.

---

### Step 7 — Migrate the arr stack + Gluetun from the Docker VM

**Source compose files obtained and reviewed** (real `arrq-stack/compose.yml` and `gluetun/compose.yml` from the Docker VM, not reconstructed from guesswork) — several findings that changed the plan from the original handoff:

- **Whisparr confirmed dead** — exists as a leftover directory on the Docker VM but was never actually deployed (user confirmed). Not migrated.
- **A separate `jellyseerr/` directory also exists** on the Docker VM, root-owned, distinct from `seerr/` — user will delete it independently; not part of this migration (consistent with §0's earlier finding that Seerr supersedes Jellyseerr).
- **qBittorrent uses `network_mode: "container:gluetun"`** (not `service:gluetun` as initially guessed) — functionally equivalent, just the actual syntax in use.
- **A genuine, working third-party integration exists**: `DOCKER_MODS=ghcr.io/t-anc/gsp-qbittorent-gluetun-sync-port-mod:main` on qBittorrent, paired with Gluetun's own `config.toml` exposing a `GET /v1/portforward` API route specifically for this mod (port-forwarding sync). Preserved as-is in the new qBittorrent/Gluetun compose file (see below) — not simplified away.
- **`private`/`proxy` Docker networks (both `external: true`) don't carry over to the NAS** — confirmed they're Docker-VM-local, created ad hoc by the user early on without firm intent ("configured them when I had no idea what I was doing"). Resolved: a new NAS-local `media-internal` network handles inter-service traffic; Seerr and Aurral (the only two with Traefik labels) drop their Docker-network-based Traefik integration entirely — Traefik (still on the Docker VM) needs new dynamic-config entries pointing at the NAS by IP:port instead (`192.168.50.163:5055` for Seerr, `192.168.50.163:3001` for Aurral), same pattern already used for Jellyfin/Plex. This is Traefik-side work on the Docker VM, flagged inline in the compose file, tracked as a follow-up — not yet done.
- **External NFS-backed Docker volumes (`tnas-f4-424-media`, `tnas-f4-424-downloads`) confirmed via `docker volume inspect`** to be NFS mounts pointing at the NAS itself (`192.168.50.163:/Volume1/nexus-pve/media` and `.../downloads`, `nfsvers=3,nolock,soft`). Since containers now run locally on the NAS, these become simple bind mounts directly to `/Volume1/nexus-pve/media` and `/Volume1/nexus-pve/downloads` — no Docker volume objects needed at all on the NAS side, and no NFS round-trip.
- **Aurral and slskd cannot safely share one Soulseek account** — both would maintain live connections simultaneously, and Soulseek's protocol generally disconnects one session when a second login under the same username occurs. User chose to give Aurral its **own**, separate, brand-new Soulseek identity (`kazuki-aurral`, password set directly on the NAS, not passed through this conversation) rather than share `kazuki-nas` (slskd's identity) or risk instability.

**Compose file written**: `/Volume1/@apps/homelab/media/compose.yml` — Seerr, Aurral, flaresolverr, Prowlarr, Radarr, Sonarr, Bazarr, Lidarr. `PUID`/`PGID` set to `1006`/`997` (svc-media) directly in the file from the start this time, applying Step 5's lesson proactively rather than discovering the root-ownership problem after a failed start. `TZ` hardcoded to `Africa/Lagos`, matching jellyfin/slskd. qBittorrent and Gluetun deliberately excluded — they go in the separate `qBittorrent/` directory (see below), not here.

**Config migration — this took far longer than expected, and surfaced several real, reusable lessons:**

1. Pre-created the seven destination subdirectories (`media/{seerr,aurral,prowlarr,radarr,sonarr,bazarr,lidarr}`) before transferring — necessary because `rsync` won't create missing *parent* directories on the fly, only the final target.
2. **First transfer attempt used one `rsync` call with seven source paths and a single destination directory** — this is a real rsync footgun: multiple sources + one destination flattens everything into that destination directly, rather than nesting each source under its own subfolder. Caused a partial, mixed-together write (a few Prowlarr files landed loose inside `media/` itself) before permission errors stopped it. No actual data loss — verified the destination was empty afterward, rsync's temp-file-then-rename pattern meant nothing partially-written ever got committed. **Lesson: always run one `rsync` call per source/destination pair when migrating multiple service configs, never combine them in one multi-source call.**
3. **`rsync` defaulted to port 22**; this NAS has always run sshd on port 9222 — every transfer needs `-e "ssh -p 9222"` explicitly.
4. **Ownership chain problems, twice over** — `kazuki` initially couldn't write into `media/` at all (directory owned by `nexus-tnas`/root, `755`, no group-write for `kazuki`'s actual group). Attempted to fix via `sudo chown` as `kazuki`, which **failed even though `sudo whoami` correctly returned `root`** — root-caused to the box's two-roots problem (see §0): `sudo` resolves to UID 9999 (`root`, the TOS-neutralized placeholder), not UID 0 (`nexus-tnas`, the actual file owner) — so `sudo`-as-`kazuki` can never successfully chown anything `nexus-tnas` owns, regardless of capabilities. **Lesson, now confirmed firmly: any chown/ownership fix on this box must be run as `nexus-tnas` directly, never via `sudo` from another account.**
5. **Attempting to fix this via `nexus-tnas` SSH login surfaced a second, unrelated lockout**: `pam_tally2` (an older PAM module, distinct from `faillock`, which exists in this box's PAM config but has no CLI tool installed) locked `nexus-tnas` after repeated failed password attempts. Root cause of *those* original failures turned out to be `PermitRootLogin no` (set deliberately in Step 2) silently rejecting `nexus-tnas`'s password before PAM ever got a fair chance to check it — `nexus-tnas` is UID 0, so root-login policy applies regardless of username allowlisting. The lockout messages were a red herring layered on top of the real cause.
6. **Resolved via a deliberate, temporary, fully-reverted exception**: set `PermitRootLogin yes` just long enough to run the one needed chown as `nexus-tnas`, confirmed it worked (`ls -la` showing `kazuki` as new owner), then immediately reverted to `PermitRootLogin no` and confirmed the revert via `grep`. **This is the correct, intentional pattern for any future case where `nexus-tnas` genuinely needs to SSH in — narrow window, explicit revert, confirmed both ways, never left open "to be safe."**
7. **A NAS reboot happened mid-troubleshooting** (user's choice, to clear the stuck `pam_tally2` tally file, which lives on tmpfs at `/tmp/log/tallylog` and has no CLI reset tool on this box) — this also served as the first real test of whether `/run/sshd` (also tmpfs, manually created back in Step 4) would survive a reboot. **It did** — sshd came back up on its own, listening correctly on port 9222, no manual intervention needed. Jellyfin and slskd also restarted cleanly on their own (`restart: unless-stopped` working as intended). This resolves the previously-open follow-up about `/run/sshd`'s reboot survival — confirmed fine, at least once.
8. **`AllowUsers` reverted to TOS's GUI default (`nexus-tnas` only) again after the reboot** — the same clobbering behavior documented back in Step 4, now confirmed to also trigger on reboot, not just on manual GUI panel touches. This caused a fresh, *different* set of `kazuki` login failures (rejected before authentication, logged as `not allowed because not listed in AllowUsers`) that superficially looked like more password problems but were actually a third, unrelated cause layered on top of the previous two. Fixed by re-adding `kazuki` and `svc-backup` to the allowlist and reloading. **Lesson, now doubly confirmed: re-check `AllowUsers` after *any* reboot or GUI Terminal & SNMP panel interaction, not just after deliberate edits to that panel.**
9. **A final, smaller ownership gap**: directories created via `mkdir -p` (the pre-created destination subfolders in step 1 above) inherited the creating user's ownership (`nexus-tnas`, since created from that session), not the parent directory's ownership (`kazuki`, set earlier) — `mkdir` does not inherit parent ownership, only permission-bit defaults via umask. Required one more `chown -R kazuki:allusers` pass before transfers could succeed.
10. All seven config transfers completed successfully once the above were resolved. Final step: `chown -R 1006:997 /Volume1/@apps/homelab/media/` to bring everything to `svc-media` ownership ahead of first start (proactively this time, not reactively) — verified complete via `find ... ! -user svc-media` / `! -group allusers` both returning empty.

**Still pending for the `media` stack**: Traefik dynamic-config updates on the Docker VM for Seerr/Aurral; decommissioning the old containers on the Docker VM once confirmed fully working on the NAS.

**Update — first start completed.** `compose.yml` and `.env` (Aurral's dedicated `kazuki-aurral` Soulseek credentials) were never actually written to the NAS during the earlier review/discussion — caught before any accidental overwrite of stale Docker-VM files, then created fresh via heredoc, same reliable method as jellyfin/slskd. Brought up via `docker compose up -d`:
- Seven of eight services (Aurral, flaresolverr, Prowlarr, Radarr, Sonarr, Bazarr, Lidarr) started cleanly first try — confirms Step 5/6's lesson (fix ownership before first start) genuinely paid off this time.
- **Seerr crash-looped on `EACCES` writing its own log file**, despite its config directory being correctly `svc-media`-owned on disk. Root cause: **Seerr's image (`ghcr.io/seerr-team/seerr`) does not honor `PUID`/`PGID` at all** — confirmed via `docker exec seerr id` showing `uid=1000(node)`, a fixed user baked into the image, unrelated to our env vars. This is a real, generalizable gotcha: `PUID`/`PGID` is a LinuxServer.io convention, not a universal Docker standard — every other service in this stack happens to be a LinuxServer.io image (hence why they all worked), but **any non-LinuxServer.io image needs to be checked for its actual UID model before assuming `PUID`/`PGID` will work.** Fixed by adding an explicit `user: "1006:997"` override to Seerr's service definition in `compose.yml`, forcing the container to run as `svc-media` regardless of its built-in default — confirmed via `docker exec seerr id` afterward showing the correct UID, and a clean restart.
- Radarr logged a (harmless, fully expected) warning about being unable to reach qBittorrent while qBittorrent didn't yet exist — confirmed resolved once qBittorrent came online (see below).

**`qBittorrent/` directory (Gluetun + qBittorrent) — ✅ COMPLETE.**

- Source compose file confirmed reusing `network_mode: "service:gluetun"` (functionally equivalent to `container:gluetun`, both valid for same-file services) and **deliberately reusing the existing VPN credentials/WireGuard key and Gluetun `config.toml` API keys** rather than rotating them — user's explicit choice, since moving hosts doesn't require a new VPN identity.
- **First compose draft (user-reconstructed) still referenced the stale `tnas-f4-424-downloads` named volume** — caught before deployment: this would have silently created a brand-new, empty Docker volume on the NAS rather than using the real downloads data, since the name wouldn't resolve to anything existing. Fixed by switching to a direct bind mount (`/Volume1/nexus-pve/downloads:/downloads`), consistent with the rest of the migration.
- **A `sed`-based in-place edit accidentally mangled the file's `volumes:` block** — turned a top-level volume declaration into a malformed extra "service" entry. Caught by reviewing the full file output before running anything, not assumed correct. Fixed with a second targeted edit restoring a proper top-level `volumes:` block.
- **Config transfer from the Docker VM hit the by-now-familiar `kazuki`-can't-write-into-a-`nexus-tnas`-owned-directory wall** — same fix as every other case this thread: `chown -R kazuki:allusers` on the destination first (run as `nexus-tnas` directly, not via `sudo`), then retry.
- **qBittorrent's config transfer itself mostly succeeded but failed on ~16 log files** (`qbittorrent.log` and rotated backups) with `[sender] ... Permission denied` — this one's different from every other error today: it failed reading from the **source** (Docker VM), not writing to the NAS, almost certainly because those specific log files were root-owned on the Docker VM and unreadable by the transferring user. Correctly judged as safe to skip — log files are disposable, no functional state lost (torrents, settings, RSS feeds, GeoDB, and BT_backup/.fastresume state all transferred successfully).
- Final `chown -R 1006:997` applied to the whole `qBittorrent/` directory before first start, applying Step 5/6's lesson proactively for the third time this step.
- **First start: clean.** No permission errors, no crash loops — both LinuxServer.io images (`qbittorrent`) and Gluetun's own image handled `PUID`/`PGID`/ownership without incident.
- **VPN and port-forwarding integration confirmed genuinely working end-to-end**: Gluetun's logs show it successfully obtaining forwarded ports from the VPN provider; the `gsp-qbittorent-gluetun-sync-port-mod` sidecar logs show it detecting each port change and pushing updates into qBittorrent's API successfully. One recurring, expected, non-blocking log line — `ERROR [port forwarding] refreshing port mapping ... UDP external port requested as X but received Y` — is a known ProtonVPN/Gluetun quirk (the provider doesn't always grant the exact port requested on renewal) that Gluetun handles gracefully by accepting whatever port it's actually given; not a migration-caused issue.
- Radarr's earlier qBittorrent-connectivity warning confirmed resolved once qBittorrent came online.

**Step 7's originally-scoped work is functionally complete**: all ten services (Seerr, Aurral, flaresolverr, Prowlarr, Radarr, Sonarr, Bazarr, Lidarr, Gluetun, qBittorrent) running on the NAS, correctly owned, VPN-routed download client confirmed working with port-forwarding intact.

**Jellystat — ✅ COMPLETE.** Added to the `jellyfin/` stack (per Step 5's architecture decision — its only dependency is Jellyfin, not the arr stack), alongside a fresh Postgres instance.

- **User explicitly chose a fresh start over a Postgres dump/restore** from the old Docker-VM installation — all historical watch-stats/play-count data was deliberately abandoned rather than migrated; Jellystat rebuilds its history going forward from Jellyfin's API only.
- Compose file built from the real source, with the same class of fixes as every other service this step: dropped the `jellystat`/`proxy` external Docker networks (replaced with a single `default` network shared by Jellystat and Postgres, since they only need to reach each other); switched named volumes (`jellystat-data`, `postgres-data`) to bind mounts; hardcoded `POSTGRES_IP=postgres` (resolves via Docker's internal DNS) rather than relying on an `.env`-supplied IP; **added `POSTGRES_DB` to Postgres's own environment block** — missing from the original, and the one change that actually mattered functionally: without it, Postgres creates its own default `postgres` database on first boot rather than the one Jellystat expects, breaking the connection silently.
- **User's manually-reconstructed compose file reverted several of these fixes twice** before landing correctly — same pattern as qBittorrent's compose file earlier in this step. Resolved by having the user run the exact heredoc verbatim rather than hand-merge, with one exception: `.env` was confirmed to correctly supply `TZ`, `POSTGRES_IP=postgres`, and `POSTGRES_SSL_ENABLED=false` directly, so the final deployed file uses `${...}` references for those three rather than hardcoded values — functionally equivalent, confirmed working.
- First start: Postgres came up cleanly, **no permission issues** despite not being a LinuxServer.io image (unlike Seerr) — its default internal user handled the `1006:997`-owned bind mount without complaint.
- Two transient, expected first-boot log messages (`database "postgres" already exists`, `relation "app_config" does not exist`) — both are normal noise from Postgres's own default-database creation attempt and Jellystat's schema-not-yet-initialized state on a brand-new empty database, not real errors. Resolved on their own within seconds.
- `jellystat.kazuki.uk` added to Traefik's dynamic config (same pattern as Seerr/Aurral, pointing at `192.168.50.163:3000`) — confirmed live and reachable.

**This closes out all of Step 7's originally-scoped work**, plus Jellystat (handled separately per the architecture decision, but tracked here since it shares the same migration patterns and lessons).

**Still pending, deliberately not yet done**:
- Traefik dynamic-config updates on the Docker VM for Seerr (`192.168.50.163:5055`) and Aurral (`192.168.50.163:3001`).
- Decommissioning the old `arrq-stack` and `gluetun` containers/stacks on the Docker VM — keep them stopped-but-present as rollback until the NAS-side versions have had some real runtime to prove out (Prowlarr indexer health, actual download completion through the new qBittorrent, etc.), rather than deleting immediately.
- Cleanup of stale Docker-VM-side leftovers the user flagged for independent deletion (`jellyseerr/`, `whisparr/` directories) — not part of this runbook's scope, user's own follow-up.

---

### Step 8 — DNS updates

Resolved, after walking through the actual access pattern for each service (corrects a mistake in the original handoff's draft table). **Note: the internal zone is now `internal.kazuki.uk`, not `int.kazuki.uk`** — user preference, applied as a standing convention update in `naming-convention-v1.2.md`.

| Record | Resolution | Notes |
|---|---|---|
| `nas-01.internal.kazuki.uk` | AdGuard A record → NAS LAN IP | Host-level record, per §6.2. No SSL needed — this is a friendly name for SSH/management access to the host itself, not a web service with an HTTPS endpoint. Cloudflare-issued certs (used for `*.kazuki.uk` service records like `jellyfin.kazuki.uk`) are a separate concern entirely, tied to specific web services behind Traefik, not host-level records. |
| `jellyfin.kazuki.uk` | AdGuard → Traefik; Traefik's router config forwards to NAS IP on the backend | Traefik is already the front door for this — DNS just needs to keep pointing at Traefik, not be changed to point at the NAS directly. No DNS change needed here as part of this thread, just confirm it still resolves correctly post-rename. |
| `plex.kazuki.uk` | Same as Jellyfin — Traefik in front, NAS as backend | Same — no DNS change needed, just confirm post-rename. |
| `slskd.kazuki.uk` | **No `kazuki.uk` record at all.** Corrects the handoff's draft, which incorrectly proposed this as a service record. | slskd is access-tier-equivalent to the rest of the arr stack (Sonarr, Radarr, etc.), which are accessed by raw IP:port with no DNS layer at all today. Superseded by the new `.lan` tier below — `slskd.lan` is the friendlier option, not a `kazuki.uk` record. |

**§6.4 — `.lan` tier (new, v1.4):** a separate, deliberate naming layer for LAN-only management UIs — flat, unnested, no SSL, plain A records in AdGuard. Chosen over `.local` specifically due to mDNS/Bonjour reservation conflicts (RFC 6762) — `.local` is actively auto-resolved by most modern OSes by default, and manually configuring it risks real, non-deterministic naming collisions.

**Action items for this step:**
1. **Add/confirm `nas-01.internal.kazuki.uk` in AdGuard** — pointing at the NAS's LAN IP. Not yet confirmed done as of this check.
2. After Step 3's rename, verify Traefik's router config for Jellyfin/Plex still correctly targets the NAS (it may reference the NAS by old hostname/IP — check it doesn't break).
3. **Add `.lan` A records in AdGuard** for: `nas-01.lan`, `qbittorrent.lan`, `sonarr.lan`, `radarr.lan`, `prowlarr.lan`, `bazarr.lan`, `lidarr.lan`, `slskd.lan` — all pointing at `192.168.50.163`. Port must still be specified when accessing (e.g. `http://sonarr.lan:8989`) — DNS resolves the hostname only, not the port. Not yet done as of this point in the thread.

---

## 3. Things intentionally left out of this runbook

- `nexus-pve` share rename — deferred per your decision, contains PVE backup data.
- Renaming `nexus-tnas` to `root` — rejected per your decision, UID-0 risk; access is neutralized instead.
- Any Proxmox-side work (node renames, storage ID renames) — explicitly out of scope per the handoff.
- Docker VM SSH username fix (`nexus-vm` → `kazuki`) — separate VM, separate thread.

---

## 4. Open questions before we go further

1. ~~Whether to move existing jellyfin/slskd config directories now or defer~~ — resolved: moved, clean cutover, done (Step 5).
2. Backrest's restic repository target — local NAS disk, Google Drive, or both? Handoff doesn't specify for the NAS-specific backup scope. Moot until item 7 below is resolved, since Backrest can't reach the NAS at all yet.
3. ~~Traefik config for DNS~~ — resolved: Jellyfin/Plex keep existing Traefik-fronted records unchanged, slskd gets no DNS record (see Step 8).
4. **`kazuki`'s home directory still has the `drwxrwxrwx`/wrong-owner permissions bug** that was found and fixed for `svc-backup`. Not yet applied to `kazuki`. Low urgency since `kazuki` falls back to password auth successfully and that's the accepted access pattern for now — but if key-based SSH for `kazuki` ever becomes something you actually want working, this needs the same fix: `chown kazuki:allusers /home/kazuki && chmod 750 /home/kazuki`.
5. ~~`/run/sshd` is tmpfs and untested across a reboot~~ — **resolved, tested**: a real reboot occurred during Step 7 troubleshooting; sshd came back up cleanly on its own, no manual `mkdir -p /run/sshd` needed. Confirmed fine, at least once — worth a second confirmation on a future reboot before fully trusting it, but no longer a complete unknown.
6. **Step 5's directory creation needs to satisfy `ChrootDirectory` requirements for `svc-backup`'s SFTP access** — root ownership, not group/world-writable. Confirmed fine — `/Volume1`, `/Volume1/@apps`, `/Volume1/@apps/homelab` all checked clean. Doesn't resolve item 7 below, though.
7. **`svc-backup`'s SFTP access is still genuinely broken, unresolved.** Connects and authenticates fine, chroot is active (`pwd` returns `/`), but `ls`/`cd` inside the session can't see or enter any of the real directories even though they exist and are readable via `sudo -u svc-backup ls` from outside the chroot. Investigation was paused mid-stream during Step 4/5 and never resumed — genuinely the oldest open item at this point. **Backrest cannot back up the NAS until this is fixed.** Worth picking up before declaring the NAS thread complete.
8. ~~Traefik dynamic config on the Docker VM needs new entries for Seerr/Aurral~~ — **resolved**: user added `seerr`/`aurral` routers and `seerr-service`/`aurral-service` load-balancer entries to Traefik's dynamic config file, following the exact existing pattern (matches Jellyfin/Plex's structure — `crowdsec` middleware, Cloudflare cert resolver, plain IP:port backend at `192.168.50.163:5055` and `192.168.50.163:3001` respectively). `seerr.kazuki.uk` and `aurral.kazuki.uk` now route correctly.
9. **`AllowUsers` clobbering is confirmed to trigger on reboot, not just manual GUI panel touches** — hit twice now (once after the deliberate Step 7 reboot, implicitly each time the Terminal & SNMP panel was touched). Re-check this specific line after *any* reboot or Terminal & SNMP interaction, indefinitely — standing operational habit, not a one-time gotcha.
10. ~~Old `arrq-stack`/`gluetun` containers on the Docker VM are still present~~ — **resolved**: user confirmed decommissioning is complete.
11. **`jellyseerr/` and `whisparr/` leftover directories on the Docker VM** — user's own cleanup task, independent of this runbook, not yet done as of last check.
