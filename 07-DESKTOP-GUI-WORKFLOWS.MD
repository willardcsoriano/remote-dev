# Desktop GUI Workflows

The remote-first model is most often described in terms of terminals and browsers, but a frequently overlooked benefit is that **native desktop applications on the laptop can speak to remote services transparently** through SSH tunnels and SSH-derived mounts. The laptop becomes a polished GUI front-end to whatever runs on the remote.

This document covers the patterns and tools that make this work: database GUIs, file browsers, API clients, monitoring dashboards, and the integration glue (sshfs mounts, port forwards) that ties them to the remote.

For the underlying tunnel and mount mechanisms, see [02-SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD).

---

## 1. The general pattern

Almost every native desktop application that connects to a network service can be configured to use `localhost:<port>` as the target. With an SSH tunnel forwarding `<port>` to the remote, the application is none the wiser — it sees a service responding on the laptop and connects normally.

The recipe is uniform:

1. Open a tunnel: `ssh -N -L <port>:localhost:<port> remote` (or use a `~/.ssh/config` alias as in [06-SSH-CONFIG-RECIPES.MD](06-SSH-CONFIG-RECIPES.MD))
2. Configure the laptop application to connect to `localhost:<port>` (or `127.0.0.1:<port>`)
3. Authenticate as usual — the application's own auth, not SSH's, applies inside the tunnel

The benefit relative to a web-UI-only approach: native applications are usually more polished, faster, and richer than browser equivalents. The benefit relative to fully local development: the database, queue, or other backing service lives on the remote host where it belongs.

---

## 2. Database clients

The most common case. A relational or NoSQL database runs on the remote host (often as a Docker container), bound only to `localhost` there. A native database client on the laptop, connecting through a tunnel, provides the same experience as if the database were local.

### General-purpose tools (multi-database)

#### DBeaver

- Free, open-source community edition
- Supports PostgreSQL, MySQL, MariaDB, SQLite, MongoDB, Redis, SQL Server, Oracle, ClickHouse, and dozens more
- Native Java application, available on macOS, Windows, Linux
- Built-in SSH tunneling — can establish the tunnel itself if preferred (the connection-property panel includes an "SSH" tab)

DBeaver is the default recommendation for a developer working with multiple databases. The combination of breadth, polish, and price (free) is unmatched.

#### TablePlus

- Polished, native (macOS and Windows; Linux beta)
- Free tier with reasonable limits; paid tier for power features
- Supports PostgreSQL, MySQL, MariaDB, SQLite, MongoDB, Redis, Cassandra, MSSQL, more
- Snappy UI, fast for everyday work

For developers who value UI polish and work primarily with relational databases, TablePlus is a strong choice.

#### DataGrip

- JetBrains product; paid
- Most powerful SQL-focused database IDE
- Excellent SQL completion, refactoring, schema introspection
- Bundled with IntelliJ IDEA Ultimate

For SQL-heavy work, DataGrip's intelligence is worth the cost.

### Database-specific tools

| Database | Recommended native client |
|---|---|
| PostgreSQL | pgAdmin 4 (free; web UI also possible) |
| MySQL/MariaDB | MySQL Workbench (free, Oracle), HeidiSQL (Windows) |
| Redis | RedisInsight (free, Redis Labs), Medis |
| MongoDB | MongoDB Compass (free, MongoDB Inc.) |
| ClickHouse | DBeaver, DataGrip |
| Cassandra | DBeaver, DataGrip |

### Workflow example: PostgreSQL via DBeaver

1. PostgreSQL running on the remote, bound to `127.0.0.1:5432`
2. Laptop `~/.ssh/config`:
   ```sshconfig
   Host vps1-postgres
       LocalForward 5432 localhost:5432
   ```
3. `ssh -N vps1-postgres` from laptop terminal (or in background)
4. DBeaver → New Connection → PostgreSQL → Host: `localhost`, Port: `5432`, credentials as configured in the remote PostgreSQL
5. Connect. Query. Use the GUI exactly as if PostgreSQL were on the laptop.

The same pattern, with the same tools, scales to a dozen databases on different hosts.

---

## 3. File browsers via sshfs

Mount the remote filesystem at a local path, then use any laptop application that opens files.

Installation:

```bash
# Linux
sudo apt install sshfs

# macOS (Homebrew + macFUSE)
brew install --cask macfuse
brew install sshfs

# Windows: WinFsp + sshfs-win
# https://github.com/winfsp/sshfs-win
```

Usage:

```bash
mkdir ~/remote-vps
sshfs willard@vps1:/home/willard ~/remote-vps \
    -o reconnect \
    -o ServerAliveInterval=15 \
    -o ServerAliveCountMax=3

# unmount
fusermount -u ~/remote-vps     # Linux
umount ~/remote-vps            # macOS
```

After mounting, the laptop's file manager (Finder, Files, Explorer) shows the remote home directory as a normal folder. Any application that opens files via the standard file-open dialog can open files on the remote:

- Image viewer for remote screenshots
- PDF reader for remote documentation
- Music player for remote audio files
- Video player for remote video (streams over SSH; works for modest bitrates)
- Local text editor for quick edits to remote config files

### Performance notes

- **Small-file operations are good, large-file operations are slow-ish.** sshfs is fine for opening a 5 MB PDF; it is not appropriate for editing a 5 GB video. For bulk operations, prefer `rsync`.
- **Latency dominates.** A remote on the same continent: snappy. A remote across an ocean: noticeably laggy.
- **Caching helps.** The `cache=yes`, `kernel_cache`, and `compression=yes` options improve subjective responsiveness.

### Use case: local Markdown preview of remote files

Many laptop Markdown editors (Typora, Obsidian, Marktext) can open `.md` files from any filesystem. Mount the remote, open the `mds` repository in the editor, write and preview as if local. The repository commits and pushes still happen remote-side via a terminal.

---

## 4. API clients

Tools that send HTTP requests against an API. Native clients on the laptop, tunneled to a backend service on the remote.

| Tool | Notes |
|---|---|
| **Postman** | Native, mature, complex; team features in paid tier |
| **Insomnia** | Native, simpler than Postman; free tier |
| **Bruno** | Open-source, file-based (request collections in git) |
| **HTTPie Desktop** | New, polished native version of the HTTPie CLI |
| **RapidAPI Client** | Free, native |

Workflow: backend API runs on the remote at `localhost:3000`. Tunnel forwards that port. Postman / Bruno collections target `http://localhost:3000`. The remote backend is exercised exactly as a local one would be.

### CLI HTTP tools

For terminal-driven work:

- **curl** — universal, dense syntax
- **HTTPie** — friendly defaults, JSON-aware
- **xh** — Rust rewrite of HTTPie, faster

These run directly on the remote (over an SSH shell) for the lowest-latency API testing, or on the laptop against a tunneled endpoint.

---

## 5. Git GUIs

For developers who prefer visual diff, history, and branching tools over terminal git.

| Tool | Notes |
|---|---|
| **GitKraken** | Polished, paid for non-OSS use |
| **Sourcetree** | Free (Atlassian) |
| **Fork** | Free for personal use, native |
| **GitHub Desktop** | Free, GitHub-focused, simple |
| **Lazygit** | Terminal UI; runs on the remote, not the laptop |

For remote-first workflows, the simplest pattern: the working tree lives on the remote; the laptop git GUI accesses it via sshfs mount. Open the mounted path as the repository in the GUI.

Limitations:

- Performance is moderate (every `git status` walks the remote filesystem over sshfs)
- Some GUIs cache aggressively and may show stale state after a remote-side `git pull`
- Conflict resolution is awkward when both the laptop and the remote could edit the working tree

For most users, a terminal git on the remote (with `tig`, `lazygit`, or plain `git log --graph --oneline`) is more productive than a sshfs-mounted GUI.

---

## 6. Monitoring dashboards (already covered, briefly)

| Service | Port |
|---|---|
| Cockpit | 9090 |
| Netdata | 19999 |
| Grafana | 3000 |
| Portainer | 9000 |
| Uptime Kuma | 3001 |

These are browser-based; the laptop's browser (Firefox, Chrome, Brave, Safari, anything) is the GUI, the tunnel is the connectivity. No specialised native application required.

For Grafana specifically, the desktop is the browser; the configuration of the dashboards happens server-side. The same Grafana that monitors the VPS can dashboard databases, metrics from `Prometheus`, logs from `Loki`, traces from `Tempo` — all on the same host, all reached through one tunnel.

---

## 7. Container and image management

While the canonical Docker GUI experience is **Portainer** (browser-based, see [03-WHAT-TO-TUNNEL.MD](03-WHAT-TO-TUNNEL.MD) §2), some native options exist.

### Docker Desktop with remote context

Docker Desktop on the laptop can be configured to use a remote Docker daemon:

```bash
# on the laptop, register a context for the remote Docker
docker context create vps1 --docker "host=ssh://willard@vps1"
docker context use vps1

# from now on, `docker ps`, `docker logs`, etc. operate on the remote daemon
```

The laptop's Docker Desktop GUI also picks up the new context. Operations on containers, images, volumes, and networks happen remote-side via SSH; the laptop Docker daemon need not even be running.

Limitation: Docker Desktop is heavyweight; running it just to manage a remote Docker daemon is somewhat wasteful. For CLI-only use, the `docker context` mechanism alone is sufficient.

### Lazydocker

A terminal-UI Docker manager. Runs on the remote via SSH; no port forwarding needed. Fast, keyboard-driven, free.

---

## 8. SSH-aware text editors that aren't IDEs

Some text editors support remote files directly via SSH/SFTP without full IDE features:

- **Sublime Text** + SFTP plugin
- **BBEdit** (macOS) + SFTP
- **Notepad++** (Windows) + NppFTP

These open a remote file, save it back via SFTP, and require no sshfs mount. Lighter than a Remote-SSH IDE setup, less powerful for navigation across many files.

For occasional remote edits without committing to a full IDE workflow, these are appropriate.

---

## 9. Specialty integrations

### Wireshark capturing remote traffic

Run `tcpdump` on the remote, pipe its output over SSH to a local Wireshark instance:

```bash
ssh remote "sudo tcpdump -i any -U -w - 'not port 22'" | wireshark -k -i -
```

Wireshark displays the capture as it would for a local interface, but the actual traffic comes from the remote's network interfaces. Useful for diagnosing issues that don't reproduce on the laptop.

### Local browser DevTools against remote dev servers

The laptop browser's DevTools (Inspect, Network panel, console, performance profiler) work transparently against tunneled HTTP endpoints. The browser cannot distinguish a tunneled service from a local one. Use Chrome / Firefox / Edge DevTools as normal; the metrics, network inspection, and breakpoints all work.

### Local Postman testing of webhooks from the remote

Tunnel-and-listen pattern: the laptop is the webhook receiver, the remote (or a service it can reach) is the caller.

```bash
ssh -R 8000:localhost:8000 remote
# the remote can now POST to http://localhost:8000 to reach the laptop's port 8000
```

The laptop runs a webhook receiver (a small Node script, `webhook` Go binary, or Postman in mock mode). The remote service's webhook test sends to `localhost:8000` on its side; the request lands on the laptop. Equivalent to ngrok, but private to the SSH connection.

---

## 10. The integration philosophy

The pattern across all of these is the same: **the laptop is a polished GUI, the remote is the substantive work environment, SSH is the connective layer.** No part of the development workflow is inherently constrained to live in one place; tools can choose to live wherever they perform best, and SSH bridges them.

The cost of this composition is **one alias per service in `~/.ssh/config`** and the discipline to open the right tunnels at the start of a work session. The benefit is a workflow in which:

- The laptop is light, replaceable, and disposable
- The remote is durable, capable, and snapshot-backed
- Every category of tool (editor, database client, file manager, monitoring dashboard, API client) can use its best-of-breed implementation
- Nothing in the workflow depends on a single vendor's continued support of remote-development features

A workflow built on this composition is genuinely robust: any single tool can be replaced without touching the others.

---

## Further reading

- [02-SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD) — sshfs, port forwarding, agent mechanics
- [03-WHAT-TO-TUNNEL.MD](03-WHAT-TO-TUNNEL.MD) — services that pair with the GUI tools listed here
- [06-SSH-CONFIG-RECIPES.MD](06-SSH-CONFIG-RECIPES.MD) — managing many tunnel aliases
- [08-SECURITY.MD](08-SECURITY.MD) — implications of running native apps against tunneled services

External:

- DBeaver: <https://dbeaver.io/>
- TablePlus: <https://tableplus.com/>
- Insomnia: <https://insomnia.rest/>
- Bruno: <https://www.usebruno.com/>
- macFUSE + sshfs (macOS): <https://macfuse.github.io/>
- WinFsp + sshfs-win (Windows): <https://github.com/winfsp/sshfs-win>
