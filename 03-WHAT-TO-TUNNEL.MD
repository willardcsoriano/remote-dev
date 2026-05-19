# What to Tunnel

An inventory of services worth running on a remote host and reaching from the laptop through SSH. The mechanism is the same for all of them — a `LocalForward` directive in `~/.ssh/config`, or a one-off `ssh -L` invocation — so this document focuses on *what* is worth running, and *why*.

Each entry lists the typical port, a short description, and notable considerations.

---

## 1. Administrative interfaces

### Cockpit

| Property | Value |
|---|---|
| Default port | 9090 |
| Protocol | HTTPS (self-signed cert) |
| Auth | PAM (system user + password) |

Browser-based Linux system administration: services, users, logs, storage, firewall. See the companion notes in the `mds` repository for detailed coverage.

### Netdata

| Property | Value |
|---|---|
| Default port | 19999 |
| Protocol | HTTP |
| Auth | None on the local dashboard |

Real-time system observability. Per-second resolution, hundreds of pre-built collectors, built-in alerting. Read-only.

### Other server management UIs

- **Webmin** (port 10000) — older, comprehensive system management
- **Glances** (port 61208) — terminal-style metrics with optional web UI

---

## 2. Container management

### Portainer

| Property | Value |
|---|---|
| Default port | 9000 (HTTP) / 9443 (HTTPS) |
| Auth | Local user database |

Web UI for Docker and Docker Swarm: list/start/stop containers, manage volumes, networks, images, exec into running containers, view logs. The most popular general-purpose Docker UI.

### Dockge

| Property | Value |
|---|---|
| Default port | 5001 |
| Auth | Local user database |

Compose-focused alternative to Portainer. Manages `docker-compose.yml` stacks. Lighter weight, friendlier to compose-native workflows.

### Yacht

| Property | Value |
|---|---|
| Default port | 8000 |

Compose-centric, app-store-style UI for Docker. Useful for homelab deployments with many self-hosted apps.

### Lazydocker / ctop

Terminal UIs, not browser-based. Run on the remote host's shell (via SSH); no port forwarding needed. Cover the same ground as Portainer for users who prefer terminals.

---

## 3. Databases

### PostgreSQL

| Property | Value |
|---|---|
| Default port | 5432 |
| Auth | Database user + password |

Forward the port and connect with a local GUI client (DBeaver, TablePlus, pgAdmin, DataGrip) as if the database were running on the laptop.

### MySQL / MariaDB

| Property | Value |
|---|---|
| Default port | 3306 |
| Auth | Database user + password |

Same pattern. MySQL Workbench, DBeaver, TablePlus all work.

### Redis

| Property | Value |
|---|---|
| Default port | 6379 |
| Auth | Optional password (`requirepass`) |

Forward and connect with `redis-cli` from the laptop, or a GUI (RedisInsight, Medis).

### MongoDB

| Property | Value |
|---|---|
| Default port | 27017 |
| Auth | Optional username/password |

Compass (official GUI) or MongoDB for VSCode (extension) work over the forwarded port.

### Elasticsearch / OpenSearch

| Property | Value |
|---|---|
| Default port | 9200 (HTTP API) / 9300 (transport) |
| Auth | Optional security plugin |

Forward 9200 and query directly with `curl`, or use Kibana/OpenSearch Dashboards on a separate port (5601).

### SQLite

Not network-accessible. SQLite files are best accessed via sshfs or downloaded directly with `rsync`/`scp`, then opened with a local GUI (DB Browser for SQLite, DBeaver).

---

## 4. Web admin tools for databases

| Tool | Port | For |
|---|---|---|
| pgAdmin | 80 (default container) | PostgreSQL |
| phpMyAdmin | 80 | MySQL/MariaDB |
| Mongo Express | 8081 | MongoDB |
| Redis Commander | 8081 | Redis |
| Adminer | 8080 | Many SQL databases |

Often run as containers alongside the database itself, on the same Docker network.

---

## 5. Development infrastructure

### Source-control servers

| Tool | Port | Notes |
|---|---|---|
| Gitea | 3000 | Lightweight, single-binary GitHub-like |
| Forgejo | 3000 | Community fork of Gitea |
| GitLab CE | 80, 22, 5005 | Heavyweight; full-featured |

Use cases: a private repository host for personal projects, a CI runner integration target, an internal artefact distribution point.

### CI/CD runners

| Tool | Port | Notes |
|---|---|---|
| Drone CI | 80 | Pipeline-as-code, container-native |
| Woodpecker CI | 8000 | Drone fork |
| Jenkins | 8080 | Mature, plugin-rich |
| Concourse | 8080 | Pipeline-focused, opinionated |
| Buildkite agents | (agent only) | Hosted control plane |

### Container registries

| Tool | Port | Notes |
|---|---|---|
| Harbor | 80 / 443 | Production-grade with vulnerability scanning |
| distribution/distribution | 5000 | The reference Docker registry, minimal |

### Artefact / package registries

| Tool | Port | Notes |
|---|---|---|
| Verdaccio | 4873 | Private npm registry |
| Sonatype Nexus | 8081 | Multi-format (Maven, npm, Docker, more) |
| JFrog Artifactory CE | 8081 / 8082 | Multi-format |

### Mail capture for development

| Tool | Port | Purpose |
|---|---|---|
| Mailpit | 8025 (UI), 1025 (SMTP) | Catches outgoing mail from your app; web UI for inspection |
| MailHog | 8025 (UI), 1025 (SMTP) | Predecessor of Mailpit |
| Inbucket | 9000 | Similar pattern with API |

Essential for development against any app that sends email — the app sends to the local SMTP capture, mail never goes anywhere real.

---

## 6. Observability

### Metrics

| Tool | Port | Purpose |
|---|---|---|
| Prometheus | 9090 | Pull-based metrics database |
| VictoriaMetrics | 8428 | Prometheus-compatible, more efficient at scale |
| InfluxDB | 8086 | Time-series database |

### Visualisation

| Tool | Port | Purpose |
|---|---|---|
| Grafana | 3000 | Dashboarding for Prometheus/Loki/InfluxDB and many other sources |

### Logs

| Tool | Port | Purpose |
|---|---|---|
| Loki | 3100 | Log aggregation (Grafana ecosystem) |
| OpenSearch / Elasticsearch | 9200 | Full-text log indexing |

### Traces

| Tool | Port | Purpose |
|---|---|---|
| Jaeger | 16686 | Distributed tracing UI |
| Tempo | 3200 | Grafana-ecosystem tracing |

### Uptime

| Tool | Port | Purpose |
|---|---|---|
| Uptime Kuma | 3001 | Self-hosted uptime monitor (Pingdom-equivalent) |
| Gatus | 8080 | Health-check dashboard |

---

## 7. Self-hosted productivity

These are commonly run on a personal VPS for individual use; the SSH-tunnel access pattern keeps them off the public internet while remaining available to the operator from anywhere.

### Password and secret management

| Tool | Port | Purpose |
|---|---|---|
| Vaultwarden | 80 | Bitwarden-compatible server |
| Passbolt | 80/443 | Team-focused password manager |
| HashiCorp Vault | 8200 | Secrets management, more developer-focused |

### Notes and knowledge

| Tool | Port | Purpose |
|---|---|---|
| Trilium | 8080 | Hierarchical personal knowledge base |
| BookStack | 80 | Wiki with books/chapters/pages structure |
| Outline | 3000 | Notion-like collaborative wiki |
| AppFlowy | 8000 | Notion alternative |
| Affine | 3010 | Notion/Miro alternative |

### Files and sync

| Tool | Port | Purpose |
|---|---|---|
| Nextcloud | 80/443 | Google Drive replacement, comprehensive |
| Seafile | 8000 | File-sync-focused, lighter than Nextcloud |
| Syncthing | 8384 | Peer-to-peer file sync |

### Reading and reference

| Tool | Port | Purpose |
|---|---|---|
| Wallabag | 80 | Read-later (Pocket replacement) |
| Linkwarden | 3000 | Bookmark manager with snapshots |
| FreshRSS | 80 | RSS reader |
| Calibre-Web | 8083 | Ebook server and reader |
| Kavita | 5000 | Comics, manga, books |
| Paperless-ngx | 8000 | Document scanning, OCR, full-text search of personal documents |

### Media

| Tool | Port | Purpose |
|---|---|---|
| Jellyfin | 8096 | Video/music server (free Plex alternative) |
| Plex | 32400 | Polished media server (proprietary) |
| Audiobookshelf | 13378 | Audiobook and podcast server |
| Navidrome | 4533 | Music server (Subsonic-compatible) |
| PhotoPrism | 2342 | AI-tagged personal photo library |
| Immich | 2283 | Google Photos replacement with face/object recognition |

---

## 8. AI tooling

A category that has grown significantly recently. Many AI tools follow the local-or-remote pattern — a backend doing the heavy lifting, a web UI for interaction.

### LLM frontends

| Tool | Port | Purpose |
|---|---|---|
| Open WebUI | 8080 | Ollama frontend; OpenAI-compatible UI |
| LibreChat | 3080 | Multi-provider chat (OpenAI, Anthropic, etc.) |
| Chatbox | (desktop) | Native client, mentioned for completeness |
| AnythingLLM | 3001 | Document-aware chat, vector store integration |

### Local model runtimes

| Tool | Port | Purpose |
|---|---|---|
| Ollama | 11434 | LLM serving (Llama, Mistral, others) |
| llama.cpp server | 8080 | Direct GGUF model serving |
| vLLM | 8000 | Production-grade LLM inference |

### Image generation

| Tool | Port | Purpose |
|---|---|---|
| Stable Diffusion WebUI (AUTOMATIC1111) | 7860 | Comprehensive UI for Stable Diffusion |
| ComfyUI | 8188 | Node-graph UI for image generation pipelines |
| Fooocus | 7865 | Streamlined SD interface |
| InvokeAI | 9090 | Polished SD interface |

Note that image-generation models require a capable GPU; most small VPSes are not appropriate hosts. Mentioned for completeness.

### Voice and audio

| Tool | Port | Purpose |
|---|---|---|
| Whisper.cpp server | 8080 | Speech-to-text |
| Coqui TTS | 5002 | Text-to-speech |

### Workflow

| Tool | Port | Purpose |
|---|---|---|
| n8n | 5678 | Workflow automation, low-code |
| Activepieces | 80 | Workflow automation, open-source Zapier alternative |
| Flowise | 3000 | Drag-and-drop LLM-pipeline builder |

---

## 9. Browser-served editors and IDEs

| Tool | Port | Purpose |
|---|---|---|
| code-server | 8080 | VSCode in a browser |
| OpenVSCode Server | 3000 | Microsoft-published alternative |
| JupyterLab | 8888 | Notebook-style interactive coding |
| RStudio Server | 8787 | R IDE |
| Theia | 3000 | Eclipse Foundation's VSCode-like IDE |

See [04-EDITORS-WITHOUT-VSCODE.MD](04-EDITORS-WITHOUT-VSCODE.MD) for installation and use.

---

## 10. Communication and collaboration

Useful when the developer needs to host a communication service rather than rely on third-party platforms.

| Tool | Ports | Purpose |
|---|---|---|
| Matrix (Synapse) | 8008, 8448 | Federated chat |
| Rocket.Chat | 3000 | Slack alternative |
| Mattermost | 8065 | Slack alternative |
| Jitsi Meet | 80/443 | Video conferencing |
| BigBlueButton | 80/443/3478 | Video conferencing for education |
| Etherpad | 9001 | Real-time collaborative text editing |
| Owncast | 8080 | Live video streaming (Twitch alternative) |

---

## 11. Dashboards

Aggregating UIs that surface multiple services in one place.

| Tool | Port | Purpose |
|---|---|---|
| Homepage | 3000 | Service dashboard with widgets |
| Dashy | 4000 | Customisable homepage |
| Heimdall | 80 | Simple service launcher |
| Homarr | 7575 | Service dashboard with rich integrations |

Run on the remote, tunnel to the laptop, use it as the bookmark page of all the other tools above.

---

## 12. Pattern summary

The unifying observation is that **any service that listens on a TCP port can be tunneled**. The combination of:

- A capable VPS hosting many containerised services
- A firewall blocking all public access except SSH (port 22)
- A laptop with `~/.ssh/config` aliases per service
- A browser, native client, or terminal program reaching `localhost:<port>`

…produces a model where the developer has hundreds of internal-only services available, accessible from anywhere, with no public attack surface beyond SSH. Adding a new service is a `docker-compose up` on the remote plus a `Host` block in `~/.ssh/config`. The whole architecture composes endlessly.

---

## Further reading

- [02-SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD) — the mechanism enabling all of the above
- [06-SSH-CONFIG-RECIPES.MD](06-SSH-CONFIG-RECIPES.MD) — managing dozens of host aliases sustainably
- [07-DESKTOP-GUI-WORKFLOWS.MD](07-DESKTOP-GUI-WORKFLOWS.MD) — native laptop applications targeting these services

External:

- [awesome-selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted) — exhaustive catalogue of self-hostable software
- [Awesome Docker](https://github.com/veggiemonk/awesome-docker) — broader Docker ecosystem
- [LinuxServer.io](https://www.linuxserver.io/) — well-maintained container images for many of the tools listed
