# SSH Superpowers

The SSH protocol is best known as a remote-shell mechanism, but its capabilities extend substantially beyond opening a login session. This document catalogues the features that make SSH the connective tissue of remote-first development, and the ancillary tools (sshfs, rsync) that depend on it.

For a foundational treatment of SSH itself — key pairs, authentication, server configuration — refer to introductory material on the protocol. This document assumes that background and focuses on the *uses* of SSH within a remote-first workflow.

## 1. The five core capabilities

| Capability | Flag / Tool | Typical use |
|---|---|---|
| Remote shell | `ssh` (default) | Log in and run commands |
| Local port forwarding | `-L` | Pull a service on the remote host to the laptop's localhost |
| Remote port forwarding | `-R` | Expose a laptop service to the remote host |
| Dynamic port forwarding | `-D` | Use the remote host as a SOCKS proxy |
| Jump host | `-J` / `ProxyJump` | Reach a host that is only accessible through an intermediary |

To these, two SSH-derived tools must be added:

| Tool | Purpose |
|---|---|
| **sshfs** | Mount a remote filesystem on the laptop as if it were local |
| **rsync over SSH** | Efficient, resumable file synchronisation |

A third capability — **agent forwarding** — is treated separately because its use is more nuanced.

## 2. Local port forwarding (`-L`)

The most commonly used SSH feature beyond the shell. Pulls a service listening on the remote host (or reachable from it) to a port on the laptop.

```bash
ssh -L <local-port>:<dest-host>:<dest-port> user@remote
```

The destination is from the remote host's perspective. If `<dest-host>` is `localhost`, the remote forwards to itself; if it is another hostname, the remote forwards to that other host on the destination port.

Examples:

```bash
# Reach a database running on the remote host
ssh -L 5432:localhost:5432 user@remote
# now `psql -h localhost -p 5432` on the laptop connects to PostgreSQL on the remote

# Reach a database running on a third host, through a bastion
ssh -L 5432:db.internal:5432 user@bastion
# the laptop's localhost:5432 → bastion → db.internal:5432

# Multiple forwards in one SSH session
ssh -L 5432:localhost:5432 -L 6379:localhost:6379 user@remote

# Hold the tunnel without an interactive shell
ssh -N -L 5432:localhost:5432 user@remote

# Background the tunnel after authentication
ssh -fN -L 5432:localhost:5432 user@remote
```

The most common point of confusion: the `<dest-host>` is **the remote's localhost, not the laptop's**. When in doubt, mentally label the addresses: "from where" → "to where, from the SSH server's perspective".

For the production-grade pattern using `~/.ssh/config`, see [06-SSH-CONFIG-RECIPES.MD](06-SSH-CONFIG-RECIPES.MD).

## 3. Remote port forwarding (`-R`)

The mirror image of `-L`. A port on the **remote** host becomes connected to a service on the **laptop**.

```bash
ssh -R <remote-port>:<dest-host>:<dest-port> user@remote
```

Examples:

```bash
# Expose the laptop's dev server to the remote host
ssh -R 8000:localhost:8000 user@remote
# now `curl http://localhost:8000` from the remote reaches the laptop's port 8000

# Expose a laptop database to the remote
ssh -R 5432:localhost:5432 user@remote
```

Use cases:

- **Webhook testing.** A service running on the remote calls back to a development server running on the laptop. Without `-R`, the laptop is unreachable from the remote (NAT, no public IP). With `-R`, the remote can reach the laptop through the SSH tunnel.
- **Reverse-tunnel deployments.** Maintain persistent outbound connections from machines behind NAT, allowing inbound access for management purposes — the foundation of many "phone home" architectures.
- **Sharing a laptop service through a public VPS.** Combined with `GatewayPorts yes` on the remote `sshd_config`, the forwarded port can be exposed to clients beyond the remote host itself. This is essentially how ngrok works at the protocol level.

### Security note

`-R` requires careful configuration. By default, the remote port is bound only to `127.0.0.1` on the remote host. Exposing it to the public requires `GatewayPorts yes` server-side, which should be enabled only deliberately. Without this option, attempts to bind to a non-loopback address are silently rebound to loopback.

## 4. Dynamic port forwarding / SOCKS proxy (`-D`)

Establishes a SOCKS5 proxy server on the laptop, with the remote host as the egress point. All traffic configured to use the proxy exits from the remote host.

```bash
ssh -D 1080 user@remote
```

A SOCKS-aware client (Firefox is the easiest; Chrome requires OS-level proxy configuration) is then pointed at `localhost:1080`.

Use cases:

- **Appear from the remote's IP address.** Useful for testing geo-aware services, regional content variations, or verifying how a site looks from the perspective of a different network.
- **Bypass restrictive networks.** Conferences, public WiFi, and some corporate networks block or interfere with traffic. A SOCKS tunnel exits via the remote, sidestepping local restrictions.
- **Diagnose region-specific bugs.** Reproduce an issue that only manifests for clients on a particular network path.
- **Encrypt traffic on untrusted networks.** All TCP traffic through the SOCKS proxy is wrapped in SSH; eavesdroppers on the local network see only encrypted SSH packets.

### Limitations

- **UDP is not forwarded.** SSH SOCKS proxying handles TCP only. DNS lookups need to be performed by the SOCKS-aware client itself (e.g., Firefox setting `network.proxy.socks_remote_dns = true`) or they leak to the laptop's DNS resolver.
- **HTTP/3 (QUIC) traffic** does not flow through SOCKS proxies; modern browsers may attempt QUIC connections that bypass the proxy. Disable QUIC if comprehensive routing is required.

## 5. Connection multiplexing

A property of the SSH client rather than a flag, but transformative for usability. Multiplexing reuses an existing SSH connection for subsequent sessions to the same host, saving the cost of repeated TCP and SSH handshakes.

Configuration in `~/.ssh/config`:

```sshconfig
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

The first SSH connection to a host opens a Unix-domain socket at the `ControlPath` location. Subsequent connections find the socket, attach to the existing master connection, and skip the handshake. `ControlPersist 10m` keeps the master alive for ten minutes after the last attached session exits.

Effect on workflow: opening a second terminal to the same host is essentially instantaneous; an `scp` after an `ssh` reuses the same TCP connection. For workflows that spawn many short-lived SSH connections (e.g., editor "auto-save" pushing files, repeated invocations of `gh` against a tunneled API), multiplexing is the difference between sluggish and snappy.

## 6. Jump hosts (`-J` / `ProxyJump`)

Reach a host through one or more intermediaries. Common in environments where production hosts are not directly reachable from developer workstations; access flows through a hardened bastion.

```bash
ssh -J alice@bastion alice@internal-host
# multi-hop
ssh -J alice@hop1,alice@hop2 alice@target
```

In `~/.ssh/config`:

```sshconfig
Host bastion
    HostName bastion.example.com
    User alice

Host internal-host
    HostName 10.0.0.50
    User alice
    ProxyJump bastion
```

After this, `ssh internal-host` is a single command that transparently routes through the bastion. Port forwarding, scp, sshfs — all work across the jump.

Internally, `ProxyJump` is implemented as `ProxyCommand ssh -W %h:%p bastion`. The bastion's role is purely a TCP relay; it never holds the user's authentication state.

## 7. Agent forwarding

The SSH agent (`ssh-agent`) holds decrypted private keys in memory on the laptop, eliminating repeated passphrase prompts. Agent forwarding allows a remote host to use the laptop's loaded keys for outgoing SSH operations (typically `git pull`, `git push`).

```bash
# laptop, once per session
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# connect with agent forwarding
ssh -A user@remote
```

Or persistently in `~/.ssh/config`:

```sshconfig
Host remote
    ForwardAgent yes
```

On the remote, `git push` against a GitHub repository succeeds without the key ever being copied to the remote. The remote's `ssh` client opens a Unix-domain socket bridged through SSH back to the laptop's agent; signing operations are performed by the agent on the laptop.

### Security caveats

- **A root user on the remote can use the forwarded agent socket for the duration of the SSH session.** Any of the keys in the agent can be used to authenticate to other systems. Agent forwarding to an untrusted host is therefore equivalent to handing the agent's contents to the host's administrators for the duration of the session.
- **Forwarded keys do not leave the laptop.** The agent on the laptop is the only signer. The forwarded *socket* gives the remote the ability to ask the agent to sign challenges, but the keys themselves stay local.
- **Use selectively.** Enable forwarding only for hosts you control or trust fully. For routine work on a personal VPS, this is acceptable; for shared infrastructure, prefer alternatives such as per-host deploy keys or short-lived OpenID Connect tokens.

## 8. sshfs — mount a remote filesystem locally

`sshfs` is a FUSE-based filesystem driver that exposes a remote directory as a local mount point. Reads and writes are transparently proxied over SSH.

Installation:

```bash
# Linux
sudo apt install sshfs

# macOS
brew install macfuse sshfs

# Windows: WinFsp + sshfs-win (https://github.com/winfsp/sshfs-win)
```

Usage:

```bash
mkdir ~/vps-home
sshfs willard@vps1:/home/willard ~/vps-home

# the remote home directory is now accessible at ~/vps-home
ls ~/vps-home
cat ~/vps-home/README.md
# the laptop's file manager, image viewer, text editor — all can open these as if they were local

# unmount when done
fusermount -u ~/vps-home    # Linux
umount ~/vps-home           # macOS
```

Useful options:

```bash
# Performance: cache, larger reads
sshfs user@host:/path ~/mnt \
    -o cache=yes \
    -o kernel_cache \
    -o compression=yes \
    -o reconnect \
    -o ServerAliveInterval=15
```

Use cases:

- Open PDFs from the remote in the laptop's PDF reader
- Edit a few configuration files quickly without copying them down
- Browse remote photos in the laptop's image viewer
- Drag and drop files between local and remote directly in the file manager
- Run a local desktop application against remote data (e.g., LibreOffice opening a file that lives on the VPS)

Performance is reasonable for small files and unsuited to high-throughput workloads (large media editing, builds against many small files). For those cases, `rsync` the working set down explicitly.

## 9. rsync over SSH

`rsync` is a file synchronisation tool; it can use SSH as its transport. Unlike `scp`, it copies only the differences between source and destination, supports resuming interrupted transfers, preserves metadata, and offers powerful filtering.

```bash
# Upload a folder to the remote
rsync -avz ~/project/ user@remote:/home/user/project/

# Download a folder from the remote
rsync -avz user@remote:/home/user/project/ ~/project/

# Dry run (show what would happen without doing it)
rsync -avzn user@remote:/home/user/project/ ~/project/

# Delete files in destination that are no longer in source (mirror)
rsync -avz --delete user@remote:/path/ ~/local/

# Exclude patterns
rsync -avz --exclude='node_modules' --exclude='.git' ~/project/ user@remote:/path/
```

Flag glossary:

| Flag | Meaning |
|---|---|
| `-a` | "Archive": preserves permissions, timestamps, symlinks, ownership where possible |
| `-v` | Verbose: list each file transferred |
| `-z` | Compress data during transfer |
| `-n` | Dry run: report without modifying |
| `--delete` | Remove files in destination not present in source |
| `--exclude` | Skip files matching the pattern |
| `-P` | Show progress and allow resume on partial files |

For backups, deployments, or moving large working sets, `rsync` is invariably the right tool. For everyday file editing across the boundary, sshfs or the IDE's remote-edit features are usually more convenient.

## 10. mosh — survivable shell

`mosh` (Mobile Shell) is an alternative to SSH for interactive use. It provides a more resilient interactive experience over flaky or intermittent connections.

Differences from SSH:

- **Survives connection drops.** A laptop suspended and reopened, a network change from WiFi to LTE, a temporary loss of connectivity — none of these terminate a `mosh` session.
- **Local echo.** Mosh predicts the response to keystrokes and displays them immediately; subjective latency is closer to local typing.
- **UDP-based.** Mosh uses a UDP-based protocol (after an initial SSH-mediated handshake) rather than persistent TCP. This survives NAT timeouts that would otherwise drop a TCP-based SSH session.

Installation:

```bash
sudo apt install mosh    # on both laptop and remote

mosh user@remote
```

Limitations:

- Does **not** support port forwarding, X11 forwarding, or agent forwarding. Mosh is purely for the shell; for tunnels, use a parallel `ssh` session.
- Requires UDP port range (60000-61000 by default) to be reachable on the remote host. May require additional firewall configuration.

For developers who work over flaky networks (mobile tethering, train WiFi, conference networks), `mosh` is often the better default for the shell session, with a separate `ssh` session held open for tunnels.

## 11. autossh — keep tunnels alive

For long-running background tunnels that must survive temporary network interruptions, `autossh` wraps the `ssh` command and reconnects automatically if the connection drops.

```bash
sudo apt install autossh

autossh -M 0 -fN -L 9090:localhost:9090 user@remote
```

The `-M 0` flag disables `autossh`'s own monitoring channel; reconnection is then driven by `ServerAliveInterval` and `ServerAliveCountMax` (typically set in `~/.ssh/config`). This is the contemporary best practice — the older monitoring approach used additional ports and was harder to debug.

Combined with `~/.ssh/config` entries that include `ServerAliveInterval 60` and `ExitOnForwardFailure yes`, `autossh` provides genuinely robust persistent tunnels.

## 12. Putting it together

A typical day's use of these features:

```bash
# Connect to the development VPS
ssh vps1                    # alias in ~/.ssh/config

# In a second terminal: tunnels for admin UIs
ssh -N vps1-admin           # forwards Cockpit and Netdata ports

# In a third terminal: tunnel for a database GUI
ssh -N vps1-postgres        # forwards PostgreSQL port

# Mount the remote home for quick file access
sshfs willard@vps1:/home/willard ~/vps     # in a file manager

# Sync down a build artifact
rsync -avz vps1:/home/willard/project/dist/ ~/Downloads/dist/
```

All five SSH operations use the same authentication, share the multiplexed master connection (assuming `ControlMaster auto`), and exit cleanly when the master is torn down.

The conceptual unit is not "SSH" but "**the SSH session as the foundation of everything else**". Editors, file managers, database GUIs, browsers, monitoring dashboards all become local because of it.

---

## Further reading

- [03-WHAT-TO-TUNNEL.MD](03-WHAT-TO-TUNNEL.MD) — what services are worth reaching via these mechanisms
- [06-SSH-CONFIG-RECIPES.MD](06-SSH-CONFIG-RECIPES.MD) — production-grade `~/.ssh/config` patterns
- [07-DESKTOP-GUI-WORKFLOWS.MD](07-DESKTOP-GUI-WORKFLOWS.MD) — laptop applications that integrate via these tunnels
- [08-SECURITY.MD](08-SECURITY.MD) — threat model for SSH-mediated access

External:

- `man ssh`, `man ssh_config`, `man sshd_config` — authoritative reference
- The OpenSSH manual: <https://man.openbsd.org/ssh>
- `man rsync` — comprehensive, dense, worth slow reading
- Mosh project: <https://mosh.org/>
