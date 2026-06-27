# Runbooks

Per-host operational notes — environment specifics, known quirks, and recovery procedures for each piece of infrastructure.

| Host | Runbook | Status |
|---|---|---|
| `nas-01` | [nas-01.md](nas-01.md) | Present |
| `core-01` | [core-01.md](core-01.md) | Present |
| `pve-01` / `pve-02` (cluster) | — | **Missing.** Cluster setup (node rename, `kazuki` account creation, storage ID conventions) was completed but never written up as a dedicated runbook. Tracked as an open gap, not silently dropped — see the Sprint 1 status report. To be produced in its own pass, verified against live cluster state rather than reconstructed from memory or chat history. |

These were copied in as-is from existing working documents (Sprint 1 scope is migration, not rewriting). Future updates should happen in place, here, going forward.
