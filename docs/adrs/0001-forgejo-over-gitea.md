# ADR-0001: Forgejo over Gitea for self-hosted Git

## Status

Accepted — 2026-06-25

## Context

Sprint 1 of the GitOps pipeline requires a self-hosted Git server to act as the canonical source of truth for the homelab monorepo, with a one-way push mirror to a public GitHub repository for visibility and disaster recovery.

The initial planning conversation defaulted to **Gitea** as the obvious lightweight self-hosted option — it's the longest-established project in this space, well documented, and a common recommendation in homelab communities historically.

Before committing, that assumption was checked against current (2026) community consensus rather than carried forward from stale general knowledge, since the self-hosted Git landscape has shifted meaningfully in the last few years.

## Decision

Use **Forgejo**, not Gitea.

## Reasoning

Forgejo forked from Gitea in 2022, originally as a community response to governance concerns. The two projects remain API-compatible — most Gitea documentation, integrations, and migration paths apply equally to Forgejo, and moving between them later (in either direction) is straightforward if this decision ever needs revisiting.

The deciding factors:

- **Governance.** Forgejo is stewarded by a non-profit (Forgejo e.V. / Codeberg e.V. lineage), with a stated commitment to staying fully FOSS. Gitea is backed by a commercial entity (Gitea Limited, doing business as CommitGo Inc. as of the 2026 landscape check). For a homelab project explicitly intended to be documented and shared publicly, aligning with the more clearly non-commercial governance model fits the project's own posture better.
- **2026 community consensus.** A live check of current recommendations (not relying on older training data) confirmed Forgejo has become the default suggestion for new self-hosted Git setups in 2026, specifically in topologies similar to this one — single-node, Docker-based, paired with a GitOps deployment layer downstream.
- **No material technical downside.** Feature parity is close enough, and API compatibility close enough, that this is not a capability trade-off. The repo skeleton, GitHub push-mirror configuration, and everything built downstream of this choice (SOPS, Komodo, the wiki generation pipeline) are identical regardless of which of the two is chosen.

## Consequences

- Forgejo is deployed as a single Docker container on a dedicated CT (`forgejo-prod-01`), SQLite-backed, per the Sprint 1 handoff.
- Documentation, scripts, and any future automation should reference Forgejo's API/CLI conventions specifically (`forgejo admin user ...` rather than `gitea admin user ...`, etc.) — they're close to identical to Gitea's but not always byte-for-byte.
- If Forgejo's governance or maintenance trajectory changes materially, the API compatibility with Gitea means a future migration is a bounded, well-understood piece of work rather than a rebuild.

## Alternatives considered

- **Gitea** — rejected on governance grounds, despite being a perfectly competent technical choice. Not rejected for any functional deficiency.
- **GitHub-only (no self-hosted leg)** — rejected from the outset of planning. The project's stated priority is a self-hosted, declarative source of truth for the homelab; depending on a third-party platform as the canonical store contradicts that goal. GitHub is retained as a one-way mirror for visibility and off-site redundancy, not as the source of truth.
