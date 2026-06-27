# Naming Convention v1.7 — Patch Notes

> **Applies to:** `naming-convention-v1.6.md`
> **Date:** 2026-06-23
> **Status:** Patch only — apply these changes to the canonical v1.6 file. This document is not a full replacement; I don't have the complete v1.6 text in this session to safely regenerate the whole file from memory, and fabricating sections I can't verify would risk introducing errors into your canonical reference. Merge these into the existing doc by hand.

---

## 1. Correction to §3 (VM/CT Rename Table)

**Change:** VMID 105's entry currently reads `s3-prod-01` (or similar tool-abstracted placeholder) as the proposed name. Update to:

| VMID | Name | Type | Node |
|---|---|---|---|
| 105 | `garage-prod-01` | CT | pve-02 |

**Reasoning (add as a footnote/exception, same category as `pbs`/`coolify`):** Deliberate deviation — ties the name to the specific tool (Garage) rather than the abstracted role, consistent with the existing exception pattern already documented for `pbs` and `coolify`. Confirmed via `pve-workload-phase2-status-report.md` §1.

---

## 2. New Section — S3 Bucket & Key Conventions

*(Insert as a new subsection, suggested placement: after the existing service-account naming section, since it follows the same least-privilege philosophy.)*

### 2.1 Bucket Naming

Pattern: `<consumer>-<purpose>`, lowercase, hyphenated. Garage/S3 enforces DNS-label rules (3–63 chars, no underscores) — same constraint already applied to hostnames.

**Closed vocabulary for `<purpose>`:**

| Code | Meaning | Operational implication |
|---|---|---|
| `data` | Primary data the service can't function without | Needs its own backup coverage |
| `media` | User-generated/uploaded binary objects | Grows unbounded; candidate for lifecycle rules |
| `backups` | This bucket *is* a backup target for something else | Write-mostly; should be reachable by the fewest possible code paths |
| `cache` | Ephemeral, safe to lose, regenerable | Never needs backing up |

**Examples in use:** `sure-data`, `beszel-media`, `beszel-backups`.

### 2.2 Access Keys

- One Garage key per **bucket**, not per consumer — even when one consumer owns multiple buckets with different purpose codes (e.g. Beszel's `media` and `backups` buckets get separate keys, not a shared one). Rationale: a `backups` bucket should stay reachable by the fewest possible code paths; sharing a key with a `media` bucket widens that blast radius for no benefit.
- Key name matches the bucket it's scoped to: key `sure-data` → bucket `sure-data`, key `beszel-backups` → bucket `beszel-backups`.
- Never issue a shared/admin-equivalent key across multiple buckets as a shortcut.

### 2.3 Network Exposure

| Port | Purpose | Tier |
|---|---|---|
| 3900 | S3 API | `.lan` by default (e.g. `garage-api.lan`); `ts.kazuki.uk` only if a specific consumer needs remote access |
| 3903 | Admin API | Loopback-only on the Garage host. Never routed, never published to the LAN — admin actions go via SSH tunnel |
| 3909 (or equivalent UI port) | Web UI | `ts.kazuki.uk`, Tailscale-only, basicAuth retained as defense-in-depth |

No current case exists for Garage's S3 or Admin API to be internet-facing via Cloudflare. Confirmed working LAN record: `garage-api.lan` (AdGuard, points at `garage-prod-01`'s IP, port 3900).

---

## 3. New Hardware — Network Layer (gap in original §2 hardware table)

The original hardware table didn't capture the router/firewall or the wireless AP. Add:

| Hostname | Role | Notes |
|---|---|---|
| `fw-01` | Router/firewall (OPNsense) | Was previously informally referred to as `nexus-viii` — not a personal device, does not qualify for the personal-device exemption; rename applied |
| `ap-01` | Wireless access point (Cisco AIR-CAP3602I-A-K9) | Previously identified by its literal hardware model name — the exact anti-pattern §1 warns against |

---

## 4. Confirmed Since Initial Draft (2026-06-23, later same session)

The following were flagged above as unconfirmed and have since been verified directly by the system owner:

- `nexus-garage` system account on `garage-prod-01` — disabled/replaced with `svc-garage`. Confirmed.
- `garage-ui` basicAuth credential — rotated off the `nexus-garage-admin` username. Confirmed.
- Old `garage`/`bucket.kazuki.uk` Traefik router block and its Cloudflare DNS record — removed. Confirmed.

No remaining open items from tonight's Garage rebuild. Safe to treat §1–3 of this patch, plus this confirmation, as settled fact for merge into v1.6 → v1.7.
