# homelab

The living source of truth for my homelab — infrastructure, configuration, deployed services, and the reasoning behind how it's all put together.

**Mirror:** [github.com/meetKazuki/homelab](https://github.com/meetKazuki/homelab) — read-only. All real work happens against the canonical repo on self-hosted Forgejo. See [Mirror policy](#mirror-policy) below.

## What this is

This repo is the GitOps foundation for a homelab that includes a Proxmox cluster, several Docker hosts, a NAS, and a Raspberry Pi running core services — moving from manually-configured infrastructure toward a fully declarative, version-controlled setup.

The long-term goal: every host, every stack, and every config decision lives here, and a wiki gets generated *from* this repo rather than maintained separately and left to drift.

## Structure
