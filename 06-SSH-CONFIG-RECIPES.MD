# SSH Config Recipes

The `~/.ssh/config` file is the foundation of a maintainable remote-first workflow. Without it, every SSH operation requires retyping the host, user, port, key file, and any forwarding directives — both tedious and error-prone. With it, complex topologies collapse to short, memorable aliases.

This document is a recipe collection: concrete patterns that compose into a usable infrastructure-as-config setup on the laptop.

For the underlying SSH features being configured, see [02-SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD).

---

## 1. The basics

`~/.ssh/config` lives on the **laptop** (the SSH client), not on the remote host. The file has no global header; it consists of `Host` blocks, each defining an alias and the parameters used to connect to it.

```sshconfig
Host alias
    HostName 203.0.113.10
    User alice
    IdentityFile ~/.ssh/id_ed25519
    Port 22
```

`ssh alias` now connects as `alice@203.0.113.10` on port 22 using the named key. Every common SSH directive can be set here; full reference in `man ssh_config`.

### File permissions

SSH refuses to use config or key files with overly permissive permissions:

```bash
chmod 700 ~/.ssh/
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

### Verifying a config block

To see how SSH would interpret an alias without actually connecting:

```bash
ssh -G alias
```

This expands all matching directives from the file (including inheritance, see §3) and prints the resolved configuration. Use it to debug "why isn't my forward applied?" or "which key would SSH offer?"

---

## 2. The per-service alias pattern

The recommended pattern is **one alias per service**, with shared connection settings inherited from a wildcard parent block. The rationale is least-privilege: forwarding only the port for the service currently in use minimises the laptop's local exposure surface.

```sshconfig
# Inherited by every "vps1" and "vps1-*" alias
Host vps1 vps1-*
    HostName 203.0.113.10
    User alice
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
    RequestTTY no

# Specific service aliases — minimal blocks, just the forwards
Host vps1-cockpit
    LocalForward 9090 localhost:9090

Host vps1-netdata
    LocalForward 19999 localhost:19999

Host vps1-postgres
    LocalForward 5432 localhost:5432

Host vps1-code
    LocalForward 8080 localhost:8080

Host vps1-grafana
    LocalForward 3000 localhost:3000
```

Use:

```bash
ssh -N vps1-cockpit      # open the Cockpit tunnel
ssh -N vps1-postgres     # open the database tunnel
ssh vps1                 # interactive shell (the bare name still works for SSH login)
```

### Why this pattern works

- **No duplication.** Hostname, user, key, keepalive are defined once. Adding a new service is a two-line block.
- **Wildcard inheritance.** `Host vps1 vps1-*` matches the base alias and every namespaced sub-alias.
- **Connection multiplexing.** `ControlMaster auto` causes the first connection to a host to open a master socket; subsequent connections reuse it. Opening five tunnels costs one SSH handshake.
- **Persistence across short reconnects.** `ControlPersist 10m` keeps the master alive for ten minutes after the last session exits.

---

## 3. Inheritance and matching

`Host` blocks are processed top-to-bottom; the first directive seen for a given key wins. This makes ordering significant when wildcards are involved.

```sshconfig
# This works:
Host vps1
    HostName 203.0.113.10

Host *
    User alice
    ServerAliveInterval 60

# Block A's HostName wins because it appears first.
# Block B's User and ServerAliveInterval also apply because Block A didn't set them.
```

```sshconfig
# This does NOT do what one might expect:
Host *
    User alice

Host vps1
    User bob

# Connecting to vps1 results in User=alice, not bob,
# because the wildcard block's User directive is processed first.
```

The defensive habit: place specific hosts above wildcards.

### Match by pattern, not exact name

`Host` blocks accept multiple patterns separated by spaces and support `*` and `?` wildcards:

```sshconfig
Host prod-*
    User deploy
    IdentityFile ~/.ssh/prod_key

Host staging-* dev-*
    User developer
    IdentityFile ~/.ssh/dev_key

Host *.internal.example.com
    ProxyJump bastion
```

### The default `Host *` block at the end

A common pattern: end the file with a `Host *` block for true defaults applied to every connection:

```sshconfig
Host *
    AddKeysToAgent yes
    UseKeychain yes              # macOS-specific
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
    HashKnownHosts yes
```

The `Host *` block applies to all hosts but is overridden by any earlier specific block. Ordering matters: place this last.

---

## 4. The `Include` directive

For larger setups, `~/.ssh/config` can be split into multiple files:

```sshconfig
# in ~/.ssh/config
Include ~/.ssh/config.d/*.conf
Include ~/.ssh/work.conf
Include ~/.ssh/personal.conf
```

Useful when:

- Multiple roles (work / personal / open-source) have different sets of hosts
- A tool (some VPN clients, some IDEs) auto-generates a config snippet that should be tracked separately
- Hosts should be groupable as files for version control

Each included file uses the same syntax as the main config. Files can themselves contain `Include` directives (with sensible depth limits to prevent loops).

---

## 5. Multi-identity / multiple-key patterns

A common scenario: the same laptop authenticates to GitHub with one key, to a work GitLab with another, and to personal VPS hosts with a third. SSH matches keys based on the `Host` block:

```sshconfig
Host github.com
    User git
    IdentityFile ~/.ssh/github_ed25519
    IdentitiesOnly yes

Host gitlab.work.example.com
    User git
    IdentityFile ~/.ssh/work_ed25519
    IdentitiesOnly yes

Host vps1 vps1-*
    HostName 203.0.113.10
    User alice
    IdentityFile ~/.ssh/personal_ed25519
    IdentitiesOnly yes
```

The crucial directive is **`IdentitiesOnly yes`**. Without it, SSH offers *every* loaded key from `ssh-agent` in turn until one succeeds; many remote hosts disconnect after a few wrong-key offers ("too many authentication failures") before the right one is tried. `IdentitiesOnly yes` restricts the offer to the explicitly named `IdentityFile`.

---

## 6. Jump hosts (`ProxyJump`)

For hosts only reachable through a bastion:

```sshconfig
Host bastion
    HostName bastion.example.com
    User alice
    IdentityFile ~/.ssh/bastion_key

Host db-prod
    HostName 10.0.0.50
    User postgres-admin
    IdentityFile ~/.ssh/prod_key
    ProxyJump bastion
```

`ssh db-prod` transparently routes through the bastion. Port forwarding, sshfs, rsync — all work across the jump. The bastion never holds the user's authentication state.

For multi-hop:

```sshconfig
Host distant-host
    HostName 192.168.99.99
    User alice
    ProxyJump hop1,hop2,hop3
```

Each hop must itself be defined as a `Host` block; SSH chains the `ProxyJump` recursively.

---

## 7. `ssh-agent` interaction

The agent holds decrypted private keys in memory. To load a key:

```bash
ssh-add ~/.ssh/id_ed25519
```

To make this automatic when a key is first used:

```sshconfig
Host *
    AddKeysToAgent yes
    UseKeychain yes              # macOS only; integrates with Keychain.app
```

`AddKeysToAgent yes` causes a passphrase prompt on first use, after which the decrypted key is held in the agent. `UseKeychain yes` (macOS) stores the passphrase in the system keychain so subsequent logins are passphrase-free.

To inspect loaded keys:

```bash
ssh-add -l
```

To clear all keys (e.g., before stepping away from the laptop):

```bash
ssh-add -D
```

---

## 8. Selective agent forwarding

Agent forwarding lets a remote host use the laptop's loaded keys for outgoing SSH operations (e.g., `git pull` on the remote, signing with the laptop's key). It is convenient and risky in equal measure.

Best practice: enable only for trusted hosts.

```sshconfig
Host my-trusted-vps
    HostName 203.0.113.10
    User alice
    ForwardAgent yes

Host any-other-host
    HostName ...
    # ForwardAgent yes is NOT set; agent stays on the laptop
```

For untrusted or shared hosts, do not forward the agent. A root user on the remote can use the forwarded socket for the duration of the SSH session to authenticate to anything any of the loaded keys can reach.

---

## 9. Per-host port and protocol overrides

Hosts on non-standard SSH ports:

```sshconfig
Host special
    HostName 198.51.100.42
    User admin
    Port 2222
```

Forcing protocol or cipher (rare; usually defaults are fine):

```sshconfig
Host legacy-server
    HostName 10.10.10.10
    User root
    KexAlgorithms +diffie-hellman-group14-sha1
    HostKeyAlgorithms +ssh-rsa
```

The `+` prefix adds to the default set rather than replacing it. Use only when interoperating with a stubborn old host.

---

## 10. Compression and performance tuning

For high-latency or low-bandwidth connections, compression and tuned cipher selection help:

```sshconfig
Host slow-link-*
    Compression yes
    Ciphers chacha20-poly1305@openssh.com,aes128-gcm@openssh.com
```

ChaCha20-Poly1305 is faster on ARM (most phones, some laptops) and avoids the throughput cap that AES-CBC variants sometimes hit without hardware acceleration.

For long-running tunnels over flaky networks:

```sshconfig
Host flaky-*
    ServerAliveInterval 30
    ServerAliveCountMax 3
    ExitOnForwardFailure yes
    TCPKeepAlive no
```

`ServerAliveInterval 30` sends a keepalive every 30 seconds; `ServerAliveCountMax 3` disconnects after three missed responses (i.e., after 90 seconds of unresponsiveness). `TCPKeepAlive no` prevents reliance on OS-level TCP keepalives, which are coarser.

---

## 11. The X11 forwarding directive

For remote GUI applications that should render on the laptop's display:

```sshconfig
Host gui-host
    HostName ...
    ForwardX11 yes
    ForwardX11Trusted no
```

X11 forwarding is now rarely the right answer (browser-served IDEs and Wayland have replaced most of its use cases), but it remains the simplest way to launch a single remote GUI application on the local display.

---

## 12. A complete example

Combining the above into a realistic `~/.ssh/config`:

```sshconfig
##############################################################
# Personal VPS — vps1
##############################################################
Host vps1 vps1-*
    HostName 203.0.113.10
    User alice
    IdentityFile ~/.ssh/personal_ed25519
    IdentitiesOnly yes
    ForwardAgent yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
    RequestTTY no

Host vps1-cockpit
    LocalForward 9090 localhost:9090

Host vps1-netdata
    LocalForward 19999 localhost:19999

Host vps1-code
    LocalForward 8080 localhost:8080

Host vps1-postgres
    LocalForward 5432 localhost:5432

##############################################################
# Work — through bastion
##############################################################
Host work-bastion
    HostName bastion.work.example.com
    User alice
    IdentityFile ~/.ssh/work_ed25519
    IdentitiesOnly yes

Host work-*
    User developer
    IdentityFile ~/.ssh/work_ed25519
    IdentitiesOnly yes
    ProxyJump work-bastion

Host work-staging
    HostName 10.50.10.20

Host work-prod-db
    HostName 10.50.20.30
    LocalForward 5432 localhost:5432

##############################################################
# Git providers
##############################################################
Host github.com
    User git
    IdentityFile ~/.ssh/github_ed25519
    IdentitiesOnly yes

Host gitlab.work.example.com
    User git
    IdentityFile ~/.ssh/work_ed25519
    IdentitiesOnly yes

##############################################################
# Global defaults — placed LAST
##############################################################
Host *
    AddKeysToAgent yes
    UseKeychain yes
    HashKnownHosts yes
    VisualHostKey yes
    StrictHostKeyChecking ask
    UserKnownHostsFile ~/.ssh/known_hosts
```

This is enough for a developer with one personal VPS, work infrastructure behind a bastion, and three Git providers. It expands naturally: a new VPS adds a sibling `Host vps2 vps2-*` block; a new service on `vps1` adds a two-line per-service block.

---

## 13. Anti-patterns to avoid

| Anti-pattern | Why bad |
|---|---|
| Putting `Host *` at the top of the file | Wildcard rules override what's below them |
| Omitting `IdentitiesOnly yes` with multiple keys loaded | "Too many authentication failures" |
| Hardcoding passwords in any form | SSH doesn't read passwords from config; if attempted, key auth is bypassed silently |
| Setting `StrictHostKeyChecking no` globally | Defeats man-in-the-middle protection |
| Sharing `~/.ssh/config` to git as-is | The file references private key paths; safe enough, but `IdentityFile` paths leak local layout |
| Forwarding agent to every host by default | Any compromised remote can use the agent |
| Mixing `Match` and `Host` blocks without testing | The interaction is subtle; verify with `ssh -G` |

---

## 14. The `Match` directive (advanced)

For conditional configuration based on properties beyond hostname:

```sshconfig
Match host vps1 user alice
    IdentityFile ~/.ssh/alice_key

Match host vps1 user bob
    IdentityFile ~/.ssh/bob_key
```

`Match` is more flexible than `Host` but less commonly needed. Use `Host` patterns first; reach for `Match` only when the conditional logic genuinely requires it.

---

## 15. Versioning the config

The dotfiles pattern — checking `~/.ssh/config` into git — is widely used. Considerations:

- The file contains no secrets, but it does encode the structure of personal infrastructure
- Public repositories should redact specific IPs, internal hostnames, and bastion identities
- Private repositories (or encrypted secrets in a manager) are appropriate
- Hardware-specific paths (`~/.ssh/id_xyz`) make the file laptop-specific; use generic names where possible

Templating tools (chezmoi, yadm, GNU Stow) help maintain a single source of truth across multiple laptops.

---

## Further reading

- [02-SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD) — features being configured here
- [05-RESILIENCE-CHECKLIST.MD](05-RESILIENCE-CHECKLIST.MD) — including the `~/.ssh/config` in backup/restore
- [08-SECURITY.MD](08-SECURITY.MD) — security implications of forwarding and key management

External:

- `man ssh_config` — authoritative reference
- *Pro Git*, Chacon and Straub — Chapter 4 covers SSH key setup for git
- [drduh/config](https://github.com/drduh/config) and [drduh/YubiKey-Guide](https://github.com/drduh/YubiKey-Guide) — opinionated security-focused examples
