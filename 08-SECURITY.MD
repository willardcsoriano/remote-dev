# Security in Remote-First Development

The remote-first development pattern collapses several traditional security boundaries: code, secrets, runtime, and infrastructure all live on a single remote host, reached through a single channel (SSH) by a single client (the laptop). This concentration is convenient and dangerous in equal measure. This document treats the threat model deliberately, identifies the principal risks, and prescribes mitigations.

The audience is a developer or solo operator; the security level targeted is **appropriate for self-hosted personal infrastructure**. Enterprises and high-stakes deployments require additional measures beyond the scope of this document.

---

## 1. The threat model

Three actors and three assets, with the channels between them:

**Actors:**

- The **operator** (the developer)
- The **VPS provider** (Hetzner, DigitalOcean, Linode, etc.)
- A **hostile third party** (malware on the laptop, network attacker, compromised dependency, etc.)

**Assets:**

- **Source code** — on the remote host's filesystem, in git
- **Secrets** — API keys, database credentials, signing keys, customer data
- **Compute and storage** — the VPS itself, used either by the operator or co-opted by an attacker for other purposes

**Channels:**

- **SSH connection** between laptop and remote (encrypted, authenticated)
- **Tunneled application connections** layered atop SSH (database protocols, HTTP, etc.)
- **Outbound connections from the remote** to package registries, the internet, third-party APIs

The principal risks, ranked roughly by likelihood-times-impact:

| Risk | Likelihood | Impact |
|---|---|---|
| Compromised laptop with loaded SSH agent | Medium | Total loss of access |
| Compromised package dependency on the remote | Medium | Code/secret exfiltration |
| VPS provider account compromise | Low | Total loss of access and data |
| Tunnel-side service vulnerability | Low | Pivot to remote-side code execution |
| SSH brute-force or MITM | Very low (with hardening) | Total loss |
| VPS provider operational malice | Very low | Data exposure |

---

## 2. The SSH key — the central artefact

The SSH private key on the laptop is the master credential. Possession of it (or a session with the agent loaded) yields the same access as the operator. Its protection deserves disproportionate attention.

### Key generation

Use ed25519, not RSA:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
```

ed25519 keys are smaller, faster, and as of 2026 considered strictly stronger than RSA at common sizes. ed25519 has no theoretical weaknesses; RSA-2048 is borderline.

### Passphrase

Always set a passphrase on the private key:

```bash
ssh-keygen -p -f ~/.ssh/id_ed25519
```

A passphrase encrypts the key file on disk. Without it, possessing the file is sufficient to use the key. With it, the file is meaningless without the passphrase.

The passphrase need not be re-entered every connection: `ssh-agent` holds the decrypted key in memory after one entry. On macOS, `UseKeychain yes` integrates the passphrase with the system keychain (set in `~/.ssh/config`).

### Key storage and backup

- The private key file lives in `~/.ssh/id_ed25519` on the laptop.
- It should be backed up to *exactly one* secondary location — a hardware token, a password manager, encrypted storage. More copies mean more attack surface.
- It must **never** be copied to a remote host. SSH authentication does not require the private key on the destination; the agent on the laptop signs challenges.

### Hardware-backed keys

For higher security, generate a key that requires a hardware token (YubiKey, Titan, Solo Key) to use:

```bash
ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_sk
```

The resulting private-key file is meaningless without the physical token; signing requires a touch on the hardware. This defends against laptop compromise: even if the attacker has the file, they cannot use it without the token.

Trade-off: requires the token to authenticate. Losing it means using a backup key (which should be registered in advance) or going through a manual recovery procedure. For personal-VPS work this is heavier than necessary; for shared/production work it is appropriate.

### Per-laptop keys

Multiple devices should each have their own key pair. Adding `device-laptop-2026.pub` to `authorized_keys` on the remote is preferable to copying the existing private key to the new device. Reasons:

- Lost or stolen device → revoke only that device's key
- Audit trails (with key fingerprints) show which device authenticated
- Different devices may have different hardware-backing capabilities

---

## 3. SSH server-side hardening

The remote host's SSH daemon (`sshd`) is the gateway. Misconfiguration here defeats most other security measures.

Mandatory:

- **`PasswordAuthentication no`** — disable password login entirely; key-only.
- **`PermitRootLogin no`** — administrative actions go through a sudo user, never direct root SSH.
- **`AllowUsers <username>`** — explicit allow-list of users permitted to authenticate.

Strongly recommended:

- **`MaxAuthTries 3`** — limit per-connection failed attempts.
- **`LoginGraceTime 30`** — drop unauthenticated connections after 30 seconds.
- **`AllowAgentForwarding`**, **`AllowTcpForwarding`** — set explicitly. The `debian-baseline` script sets `AllowTcpForwarding local`, permitting local-only forwards (`-L`) but not gateway forwards.
- **`fail2ban`** — automatically ban IPs after repeated SSH auth failures. Standard configuration: 5 attempts → 1-hour ban.

These settings, applied in `/etc/ssh/sshd_config`, raise the cost of an attack from "trivial brute force" to "extremely difficult."

The `debian-baseline` repository (referenced in this repository's README) configures all of these by default. For hosts not provisioned with that script, the equivalent settings should be applied manually.

### Two-factor SSH (PAM-based)

For hosts requiring stronger authentication, PAM modules can require a second factor in addition to the SSH key:

- `libpam-google-authenticator` — TOTP via an authenticator app
- `libpam-u2f` — hardware key (different from ed25519-sk in mechanism but similar in intent)

Configuration: install the PAM module, configure `/etc/pam.d/sshd` to require it, set `AuthenticationMethods publickey,keyboard-interactive:pam` in `sshd_config`. The cost is one additional prompt per SSH session.

For personal-VPS work this is rarely needed; for shared infrastructure it is appropriate.

---

## 4. Tunnel exposure

A common misconception: SSH tunnels are "secure" because SSH itself is encrypted. The encryption is fine; the exposure is more nuanced.

### Local exposure

A `-L 5432:localhost:5432` tunnel opens **port 5432 on the laptop**. By default, SSH binds the local end to `127.0.0.1` — only the laptop's own processes can reach it. This is normally what is wanted.

If `GatewayPorts yes` is set on the laptop's SSH client config (rare; not the default), the forwarded port becomes reachable from other machines on the laptop's network. This is occasionally useful for testing but generally re-exposes a service that was deliberately firewalled.

### Authentication relies on the destination service

The SSH tunnel provides confidentiality and integrity between laptop and remote. It does **not** authenticate the user to the destination service. If the destination service (Netdata, an unauthenticated database, a debug HTTP server) has no authentication of its own, anyone reaching the tunneled port at `localhost` reaches the service.

Implications:

- Anyone with SSH access to the operator's laptop, while the tunnel is open, can use the tunneled services.
- Multi-user systems with the laptop's account compromised expose all tunneled services.
- Browser tabs in private/incognito mode still reach `localhost`.

Mitigation: where possible, require authentication on the destination service. Cockpit uses PAM; databases require credentials; code-server requires a password. Netdata's lack of authentication is the notable exception in the default service set — a compromised laptop exposes it.

### Open ports during inactivity

A tunnel held open for hours (e.g., backgrounded with `-fN`) exposes the destination service for as long as it lives. Periodically close tunnels when not in use:

```bash
# from earlier shell-alias examples
adminui-off   # closes the tunnels
```

Or rely on `ControlPersist` rather than long-lived `-fN` processes: the master connection closes automatically after the persistence interval, taking forwarded ports with it.

---

## 5. Agent forwarding hazards

Agent forwarding (`ssh -A`) allows the remote to use the laptop's loaded keys for outgoing SSH operations. The keys do not leave the laptop, but the *capability to sign challenges with them* is exposed to the remote.

The hazard:

- A root user on the remote, during the forwarded session, can use the laptop's agent to authenticate to any other system the laptop's keys can reach.
- A compromised remote process can do the same.

Best practices:

- **Forward agent only to fully trusted hosts.** Your personal VPS, yes. A shared development machine with other users, no.
- **Explicit per-host enablement.** Do not set `ForwardAgent yes` in a `Host *` block; set it only where needed:
  ```sshconfig
  Host my-vps
      ForwardAgent yes
  Host *
      # default: no forwarding
  ```
- **Limit what the agent holds.** Add only the keys actually needed for the session. `ssh-add` adds; `ssh-add -D` clears.
- **Consider alternatives.** Per-host deploy keys (a small key authorised only for one specific operation, like a single git remote) are safer than forwarding the master key.

---

## 6. Outbound connections from the remote

The remote host makes outbound network connections constantly: package installs, container image pulls, git fetches, telemetry. Each represents a trust relationship with whatever the remote connects to.

### Package supply chain

`apt`, `npm`, `pip`, `gem`, and so on download and execute code from upstream registries. A poisoned package becomes code execution on the remote. The September 2025–onward "Shai-Hulud" wave of npm supply-chain attacks is the contemporary cautionary example.

Mitigations:

- **Defer recent versions.** pnpm's `minimumReleaseAge` (and equivalents in npm 11, yarn 4.10+) refuses to install package versions younger than a configured threshold (7 days is common). Most malicious versions are yanked within hours; the delay catches them.
- **Pin exact versions.** Use lockfiles consistently. Resist the urge to `npm install <package>` ad-hoc without consideration of what it pulls in transitively.
- **Audit before installing.** Tools like `socket.dev` or `snyk` flag suspicious package patterns.
- **Containerise.** A poisoned package inside a container with limited filesystem access is less catastrophic than the same package running unconstrained on the host.

### Outbound to telemetry / analytics

Some installed tools phone home: Netdata Cloud, language analytics, IDE telemetry, error reporters. Each outbound connection is a piece of operational metadata leaving the network. Most are benign; some warrant disabling explicitly.

The `debian-baseline` script disables Netdata telemetry by default. For other tools, opt-out is usually possible via environment variables or config.

---

## 7. Secrets management

Secrets on a remote host are intrinsically more centralised than secrets on a laptop, which is both an advantage (one place to manage) and a risk (one place to compromise).

Layered approach:

### Layer 1 — Environment variables

Twelve-factor-app style: secrets enter the application through environment variables, never in code or config files committed to version control.

```bash
# in ~/.bashrc or per-service systemd unit
export DATABASE_URL='postgres://...'
export ANTHROPIC_API_KEY='sk-ant-...'
```

The shell environment is per-user-per-session; processes started by other users (or root) do not see them. The `.bashrc` file should be `chmod 600`.

### Layer 2 — Service-level secrets

For docker-compose deployments, secrets in a `.env` file referenced by `docker-compose.yml`:

```
# .env (never committed; in .gitignore)
DATABASE_URL=postgres://...
JWT_SECRET=...
```

Docker reads the file at container start and injects values as environment variables into the containerised app.

### Layer 3 — Secret managers

For more robust setups, a dedicated secret manager:

- **HashiCorp Vault** — full-featured, self-hosted; substantial setup overhead
- **Bitwarden Secrets Manager / Vaultwarden** — lightweight, integrates with Bitwarden vault
- **infisical** — modern, open-source, developer-focused
- **doppler** — managed (paid)
- **OnePassword Connect / 1Password CLI** — if already using 1Password personally

A secret manager centralises rotation, audit logging, and access policy. For a single-user personal VPS this is usually over-engineered; for any infrastructure with more than one collaborator it pays off quickly.

### Layer 4 — Never-committed secrets in git

Use `.gitignore` for `.env`, `secrets.json`, `*.pem`, and similar. Tools like `git-secrets` and `gitleaks` scan commits before push to catch accidental commits.

If a secret is committed by accident: **rotate it immediately**. Removing the commit from git history is theatre; the secret has been exposed and must be considered compromised.

---

## 8. The remote host's own hardening

A defended remote host is a substantial topic in its own right; this section is a checklist of the *highest-leverage* items applicable to a personal VPS.

| Measure | Coverage |
|---|---|
| Automatic security updates (`unattended-upgrades`) | Most CVEs patched within days |
| Default-deny firewall (UFW) | Most network attack surface eliminated |
| `fail2ban` on SSH | Brute-force SSH banned automatically |
| Disabled root SSH, key-only auth | Account-level attacks neutralised |
| AppArmor or SELinux enforcement | Mandatory access control; compromised process is contained |
| File integrity monitoring (AIDE) | Tampering with system binaries detected |
| Rootkit scanning (rkhunter) | Common rootkit signatures detected |
| Audit logging (auditd) | Security-relevant events recorded |
| Regular Lynis audit | Known hardening gaps surfaced |

The companion `debian-baseline` repository applies these in a single idempotent script. For non-baselined hosts, the equivalent measures should be applied manually.

### VPS-provider account security

The provider's web console is a parallel admin channel that bypasses SSH entirely. If an attacker compromises that account, they can reset SSH keys, attach the disk to a different VPS, or simply delete the host. Provider account hygiene:

- Unique, strong password
- Two-factor authentication enabled
- Hardware-key 2FA where supported (Hetzner, DigitalOcean, AWS all support this)
- Regular audit of authorised access (some providers allow team members)

---

## 9. Backup and recovery

A backup is a security control: it limits the impact of ransomware, accidental destruction, and provider failure.

Minimum acceptable:

- **VPS provider snapshots** weekly. Most providers offer this for a few dollars/month.
- **External backup** of `~/.ssh/`, `~/.config/`, and any irreplaceable data — to a second host, encrypted object storage, or a password-manager attachment.
- **Periodic restore test** — annually at minimum. A backup that has never been restored is a hypothesis.

Tools:

- **borgbackup** — deduplicated, encrypted, efficient
- **restic** — modern equivalent with native cloud-storage support
- **rsync to a second host** — simpler but lacking encryption-at-rest

Encryption matters. If backups are stored on a third-party provider (S3, Wasabi, Backblaze B2), assume the provider can read them. Encrypt before upload.

---

## 10. Incident response (briefly)

If something goes wrong — suspected compromise, unexpected outbound traffic, unauthorised login — the response should be planned in advance, not improvised.

A short personal-VPS playbook:

1. **Snapshot the VPS immediately** (preserves evidence and rollback option).
2. **Revoke SSH access** at the provider level (disconnect public key from the host).
3. **Review logs**: `journalctl`, `/var/log/auth.log`, `auditctl` outputs.
4. **Check file integrity**: `aide --check` if AIDE is configured.
5. **Check rootkits**: `rkhunter --check`.
6. **Identify the entry vector** (compromised key, exposed service, supply-chain package).
7. **Rebuild from a known-good snapshot** *after* identifying and closing the entry vector.

A more elaborate playbook is appropriate for higher-stakes infrastructure; this is the personal-VPS minimum.

The companion `debian-baseline` repository installs the logging and integrity-monitoring tools referenced above; without them, forensic capability is limited.

---

## 11. Principles to internalise

The granular advice above reduces to a handful of principles:

1. **The SSH key is the master credential.** Treat it accordingly: passphrase, hardware backing, careful backup, per-device generation.
2. **Defaults are usually wrong for security.** Most services bind to all interfaces, require no authentication, and offer no transport encryption. Override defaults deliberately.
3. **Tunnels are not magic.** They provide confidentiality between two endpoints; what the tunneled service does with the bytes that arrive is the service's own concern.
4. **The supply chain is the attack surface.** Most modern compromises target packages, not protocols. Defer recent versions; pin exact versions; audit dependencies.
5. **Backups are a security measure.** A ransomware attack is survivable with backups, catastrophic without them.
6. **Centralisation is double-edged.** Putting everything on one VPS reduces management overhead and increases blast radius. Plan for both.
7. **Audit periodically.** A configuration that was correct two years ago may not be correct today. `lynis audit system` once per quarter.
8. **No silver bullets.** Defense in depth is the only viable strategy; any single control will eventually fail.

---

## Further reading

- [debian-baseline](https://github.com/willardcsoriano/debian-baseline) — companion hardening script
- [05-RESILIENCE-CHECKLIST.MD](05-RESILIENCE-CHECKLIST.MD) — the operational side of resilience, which intersects with security
- [02-SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD) — the SSH primitives, with security caveats noted

External:

- *Practical Cryptography for Developers*, Svetlin Nakov — accessible introduction to the primitives behind SSH
- *Security Engineering*, Ross Anderson — the canonical reference, freely available online
- *The Art of Software Security Assessment*, Dowd, McDonald, Schuh — practical vulnerability analysis
- NIST SP 800-63B — authentication guidelines
- Mozilla's OpenSSH guidelines: <https://infosec.mozilla.org/guidelines/openssh>
- The OWASP top-ten lists for web-application-specific concerns
