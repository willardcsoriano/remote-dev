# remote-dev

Patterns and references for **remote-first development**: using a small, ordinary laptop as a thin client for a more capable remote server, via SSH and the tunneling primitives layered on top of it.

The architecture is older than personal computing: a modest local device handles display and input, while compute, storage, and tooling live on a remote machine. The combination becomes more useful than either part alone — a $300 laptop and a $20/mo VPS approach the productivity of a high-end workstation.

This repository is a working reference. The documents below can be read in order as a self-contained primer, or accessed independently as targeted references.

## Contents

| # | Document | Subject |
|---|---|---|
| 01 | [THIN-CLIENT-MODEL.MD](01-THIN-CLIENT-MODEL.MD) | Architectural pattern: where compute lives, what flows over the network, historical context, trade-offs |
| 02 | [SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD) | The full feature set of SSH: local/remote/dynamic forwarding, sshfs, rsync, ProxyJump, agent forwarding, connection multiplexing |
| 03 | [WHAT-TO-TUNNEL.MD](03-WHAT-TO-TUNNEL.MD) | A taxonomy of services worth reaching via tunnel: databases, dev infrastructure, self-hosted apps, observability, AI tooling |
| 04 | [EDITORS-WITHOUT-VSCODE.MD](04-EDITORS-WITHOUT-VSCODE.MD) | Alternatives and fallbacks: code-server, OpenVSCode Server, Helix, Neovim, micro, Claude Code CLI |
| 05 | [RESILIENCE-CHECKLIST.MD](05-RESILIENCE-CHECKLIST.MD) | Concrete steps to take *before* a primary tool fails, so the workflow survives |
| 06 | [SSH-CONFIG-RECIPES.MD](06-SSH-CONFIG-RECIPES.MD) | Practical `~/.ssh/config` patterns: per-service aliases, wildcards, multiplexing, multi-identity setups |
| 07 | [DESKTOP-GUI-WORKFLOWS.MD](07-DESKTOP-GUI-WORKFLOWS.MD) | Native laptop applications talking to remote services: database GUIs, sshfs mounts, API clients, monitoring dashboards |
| 08 | [SECURITY.MD](08-SECURITY.MD) | Threat model, key hygiene, tunnel exposure, agent-forwarding caveats, defense-in-depth |

## How to use this repository

If new to remote-first development, read in order from 01 to 08. The first three documents establish the model; 04 and 05 are practical safety; 06 and 07 are recipes; 08 grounds the security posture.

If looking for a specific recipe, jump directly. Each document is self-contained, with cross-references where context helps.

## Audience

Developers comfortable with the shell, a basic understanding of SSH, and access to at least one remote Linux host (VPS, lab server, homelab). No assumption of expertise in networking, systems administration, or any particular language.

## Scope and non-goals

In scope:

- Architectures and workflows that use a remote Linux host as the primary development environment
- SSH and SSH-derived protocols (sshfs, rsync over SSH, etc.) as the connective tissue
- Editor and IDE alternatives that work in this model
- Native laptop applications that integrate via tunnels
- Security considerations specific to this architecture

Out of scope:

- Detailed coverage of any single tool (editor, IDE, database) beyond its relevance to remote workflows
- Cloud-provider-specific managed services (Codespaces, Cloud9, etc.) — mentioned but not central
- Container orchestration (Kubernetes, Nomad) — separate concern
- Network protocols below SSH (TCP, IP) — assumed knowledge

## Related repositories

- [debian-baseline](https://github.com/willardcsoriano/debian-baseline) — companion hardening script for the remote host
- Personal reference notes are kept in a private `mds` repository; published material relevant to broader audience lives here

## License

These documents are released as educational reference material. Adapt, fork, redistribute freely. No warranty.
