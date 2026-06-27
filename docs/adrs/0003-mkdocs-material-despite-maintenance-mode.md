# ADR-0003: MkDocs Material for the living wiki, despite its maintenance-mode status

## Status

Accepted — 2026-06-25. Implementation deferred to a later sprint (wiki generation depends on the GitOps monorepo existing first — this sprint builds that foundation).

## Context

The project's top priority, ahead of GitOps itself, is a living wiki of the homelab — ideally generated from the same Git repo that drives infrastructure, rather than hand-maintained documentation that drifts from reality. Once the monorepo (`docs/`, `stacks/`, etc.) exists as the declarative source of truth, the wiki becomes a *view over that repo* rather than a separate maintenance burden.

**Material for MkDocs** was the front-runner for the wiki's tooling and aesthetic, on the strength of its visual quality — reference sites in this style (FastAPI's docs, Pydantic's docs, and squidfunk's own demo) were reviewed directly and confirmed as matching the desired look and feel (the Pydantic.dev site specifically was named as the target aesthetic).

However, Material for MkDocs entered **maintenance mode in November 2025** — its primary maintainer stepped back from active feature development, with the project continuing to receive only essential upkeep rather than ongoing investment. This is a real consideration for a tool meant to be the long-term face of a project explicitly intended to be shared and maintained publicly, not a minor footnote.

## Decision

Proceed with **Material for MkDocs**, with the maintenance-mode status explicitly acknowledged and documented (this ADR), rather than either ignoring it or avoiding the tool on that basis alone.

## Reasoning

- **The aesthetic requirement was specific and confirmed.** The Pydantic.dev-style look was the actual target, verified by direct visual review of reference sites rather than assumed. Maintenance mode doesn't change how the tool looks or behaves today — it changes the trajectory of future feature development, which is a different and more bounded risk.
- **Maintenance mode is not abandonment.** Essential upkeep (security fixes, compatibility with new MkDocs core releases) is still expected to continue. The risk being accepted here is "this won't gain major new features going forward," not "this will break or become unsupported soon."
- **The alternative (Zensical, MkDocs Material's prospective successor/replacement effort) was not yet mature enough to commit to at decision time.** Choosing an unproven successor over a working, well-understood tool purely to avoid a maintenance-status label would trade a known, bounded risk for an unknown one — the wrong direction for a foundational documentation choice.
- **The decision is explicitly revisitable, not locked in.** The call made was: ship with Material for MkDocs now, revisit if either Zensical matures into a clear successor, or if Material's maintenance situation degrades faster than the "essential upkeep only" expectation (e.g. security issues going unpatched). This ADR exists partly so that re-evaluation trigger is written down rather than relying on memory.

## Consequences

- The wiki, once built, depends on a tool in maintenance mode. This is an accepted, documented risk — not an oversight.
- A watch item exists: periodically check whether Zensical (or any other successor effort) has reached a state where migration is worth it. This should be revisited at a natural checkpoint (e.g. when the wiki generation work actually begins) rather than left purely implicit.
- Because the wiki will be generated from the monorepo (wiki as a *view*, not hand-authored content), the actual migration cost of switching documentation generators later is lower than it would be for a hand-maintained docs site: the source content (Markdown in `docs/`) is generator-agnostic, and only the build/theme layer would need to change.

## Alternatives considered

- **Zensical** — the apparent intended successor to Material for MkDocs. Not chosen for this era of the project because it was not yet mature enough to commit to as a foundational dependency at decision time. Revisit candidate, explicitly.
- **Docusaurus, VitePress, and other JS-ecosystem doc generators** — not pursued because the aesthetic target (Pydantic/FastAPI-style) is itself built on MkDocs Material, and replicating that specific look in a different generator would be redundant effort for no real gain.
- **Hand-maintained wiki (no generator, e.g. a plain Git wiki or static pages)** — rejected from the outset of planning, for the same reason GitHub-only hosting was rejected in ADR-0001: it reintroduces the documentation-drift problem this entire project is structured to eliminate.
