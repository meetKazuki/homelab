# ADR-0002: Komodo for Docker-level GitOps (L3 — Application Deployment)

## Status

Accepted — 2026-06-25. Implementation deferred to Sprint 2.

## Context

The homelab's automation pipeline was deliberately split into three distinct layers during planning, rather than reaching for one tool to cover all of them — a structure chosen specifically to avoid the kind of over-engineering this project has pushed back on before (e.g. the earlier rejection of Docker Swarm mode):

| Layer | Job | Produces |
|---|---|---|
| L1 — Provisioning | Create new VMs/CTs on Proxmox from a definition | A booted host: networking, base OS, SSH key, hostname |
| L2 — Configuration | Bring a booted host to "ready for workloads" | Users, packages, sshd, Docker, Tailscale, base directory layout |
| L3 — Deployment | Reconcile running Docker stacks against what's declared in Git | Containers matching `compose.yml` definitions in the monorepo |

This ADR concerns **L3** only. L1 (OpenTofu, conditional, deferred to Sprint 7) and L2 (Ansible, Sprint 3) are separate decisions.

The homelab's actual topology at the time of this decision: no Docker Swarm, no Kubernetes — multiple independent Docker hosts (`docker-prod-01`, `proxy-prod-01`, `telemetry-prod-01`, `garage-prod-01`, `coolify-prod-01`, `plane-prod-01`, plus `nas-01` and `core-01` as Docker-capable but non-uniform hosts), each running its own Compose stacks under `/opt/homelab/`.

## Decision

Use **Komodo** as the L3 deployment tool: Komodo Core on a dedicated host, **Periphery** agents on every Docker-capable host, stacks defined in the monorepo (one directory per stack under `stacks/`), with the target host expressed as a tag.

## Reasoning

- **Topology fit.** Komodo's model — a central core plus lightweight per-host agents, each agent simply executing Compose operations on its own host — matches a multi-host, non-clustered Docker fleet exactly. It doesn't assume Swarm or Kubernetes underneath it, unlike several alternatives in this space.
- **Verified current relevance, not assumed.** Given how quickly self-hosted GitOps tooling turns over, this wasn't taken on stale knowledge — a live check at decision time confirmed Komodo is the currently dominant tool for this specific shape of setup (multi-host Docker, no orchestrator), with multiple recent independent write-ups of topologies closely resembling this one.
- **Integrates with what's already built**, rather than introducing parallel infrastructure:
  - Komodo has native ntfy support, slotting directly into the existing two-tier alerting setup (`homelab-critical` / `homelab-alerts`) with no new alerting plumbing required.
  - Komodo's own configuration ("Resource Syncs") is itself expressed as TOML in Git — meaning Komodo's own setup becomes version-controlled in the same monorepo it manages, rather than living as unmanaged state in a UI.
  - Pairs naturally with Renovate for automated image-version update PRs, extending the GitOps loop to dependency management without extra tooling.
- **Avoids the bootstrapping circularity becoming a long-term problem.** Forgejo itself runs as a Docker stack and, logically, should eventually be managed the same way every other stack is. It cannot be from day one — the tool that would manage it doesn't exist yet at the point Forgejo is first stood up. This is accepted as a known, temporary gap (see Consequences), resolved as soon as Komodo exists.

## Consequences

- **Sprint 1 ships without Komodo.** Forgejo is deployed and operated manually (`docker compose` by hand on `forgejo-prod-01`), not yet GitOps-managed. This is intentional, not an oversight — see the Sprint 1 handoff's risk register.
- **Sprint 2 resolves the gap**, in order: Komodo Core deployed on its own host (`komodo-prod-01`, proposed on `pve-02` for cluster balance), Periphery agents rolled out to every Docker host, then Forgejo's own stack is the first thing adopted into Komodo once Komodo exists — closing the circularity rather than leaving it open indefinitely.
- **The first proof-of-loop migration** (recommended: `uptime-kuma` on `core-01`) is chosen deliberately as low-risk and immediately visible if something breaks, and because it forces the open question of whether Periphery runs cleanly on ARM (`core-01` is a Raspberry Pi) on day one rather than discovering it later under higher stakes.
- Stack directories live under `stacks/<host>/`, matching the existing physical layout convention (`/opt/homelab/` per host) rather than being organized by service — this means a Komodo Resource Sync can target an entire host's stacks by directory/tag cleanly.

## Alternatives considered

- **Kubernetes (k3s or similar) + Flux/ArgoCD, or Docker Swarm-based GitOps** — rejected on the same grounds Swarm mode itself was rejected earlier in this project: these solve a clustering/orchestration problem this homelab does not have, at a complexity cost disproportionate to the actual topology (independent Docker hosts, not a cluster).
- Other Docker-Compose-oriented tools (e.g. Portainer's own stack-from-Git features, Dockge) were not formally evaluated against Komodo in the planning conversation — Komodo was identified directly via a live check of current (2026) tooling for this specific topology and adopted on that basis. Noting this honestly rather than implying a comparison that didn't happen: if Komodo's fit turns out to be wrong in practice during Sprint 2, those are the most likely tools worth a real side-by-side look before reaching for something heavier.
