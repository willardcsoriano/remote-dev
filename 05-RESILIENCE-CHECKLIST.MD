# Resilience Checklist

The remote-first model introduces single points of failure that did not exist in a local development setup: the remote host, the network between laptop and remote, and the primary editing tool. This document is a deliberate, actionable checklist for surviving these failures — to be worked through *before* a failure occurs, not after.

The checklist is divided into three tiers, in order of importance. Tier 1 is non-negotiable; without it, the workflow has no safety margin. Tier 2 reduces the impact of common failures. Tier 3 covers rarer but recoverable scenarios.

---

## Tier 1 — Non-negotiable

### ☐ 1.1 Plain SSH works from the laptop terminal

Independent of any IDE, the laptop must be able to open an SSH session from a vanilla terminal:

```bash
ssh user@remote
```

If this fails, the entire remote-first model is broken and every other failure becomes catastrophic. Verify monthly. Ensure:

- The SSH private key exists on the laptop and is loaded into `ssh-agent`
- The public key is in `~/.ssh/authorized_keys` on the remote
- `~/.ssh/config` on the laptop has a working `Host` block for the remote
- Connectivity to the remote on port 22 (or the configured SSH port) is not blocked by the local network

When the IDE breaks, a working plain SSH connection is the foundation on which every fallback is built.

### ☐ 1.2 A non-IDE editor exists on the remote

At least one editor must be installed on the remote that does not depend on VSCode, JetBrains, or any other host-side IDE software. Recommended minimum:

```bash
sudo apt install micro
```

`micro` is the lowest-friction option: arrow-key navigation, standard keyboard shortcuts, no learning curve. Alternatives are `nano` (preinstalled but minimal), `helix` (more capable, slight learning curve), or `vim` / `neovim` (most capable, steep learning curve).

When VSCode or JetBrains becomes unavailable, the path forward is `ssh remote → micro file.py` (or equivalent). Without this, basic edits become impossible during an IDE outage.

### ☐ 1.3 An always-available remote shell with persistent sessions

`tmux` or `screen` on the remote allows shell sessions to survive SSH disconnects. The hardened-baseline script in `debian-baseline` installs `tmux` by default.

```bash
# on the remote
tmux new -s main          # start or attach to session "main"
tmux attach -t main       # reattach later

# inside tmux, common shortcuts (default prefix: Ctrl+b):
# Ctrl+b c    new window
# Ctrl+b ,    rename window
# Ctrl+b n/p  next/previous window
# Ctrl+b %    split pane vertically
# Ctrl+b "    split pane horizontally
# Ctrl+b d    detach (session keeps running)
```

The contract: when SSH disconnects, the work inside `tmux` continues. Reattaching from a new SSH session (potentially hours later, from a different network or device) restores the exact terminal state.

For laptop-side SSH session resilience, consider `mosh` as an alternative SSH client (see [02-SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD) §10).

### ☐ 1.4 SSH key passphrase set, key file backed up

The SSH private key on the laptop should be encrypted with a passphrase. Even if the laptop is stolen, the passphrase prevents immediate use of the key.

```bash
# Add or change passphrase on an existing key
ssh-keygen -p -f ~/.ssh/id_ed25519
```

The key file itself must be backed up to a location not on the laptop — a hardware security key, a password manager attachment, a separate offline backup. Losing the key without a backup means losing access to every system the key authenticates against; rotating to a new key on a dozen remote hosts during a crisis is a difficult position to be in.

### ☐ 1.5 A recent VPS snapshot or backup

The remote host's state must be recoverable. Most VPS providers offer snapshot backups for a few dollars per month; for Hetzner, DigitalOcean, Linode, OVH, and most others, this is a single button in the management UI. Frequency: weekly minimum, daily if changes are significant.

Alternatively, periodic `borgbackup` or `restic` backups to a second host (or to object storage) provide equivalent protection. The mechanism is less important than the contract: *the remote host's contents are reproducible without manual reconstruction*.

When the VPS is lost (provider failure, accidental deletion, compromise requiring a clean rebuild), the snapshot restores everything in minutes. Without it, days of reconstruction work follow.

---

## Tier 2 — Strong recommendation

### ☐ 2.1 Browser-served IDE installed on the remote

Install `code-server` (or `OpenVSCode Server`) on the remote, bound to `127.0.0.1:8080`, configured with password authentication, accessible via SSH tunnel.

```bash
# on the remote, one-time
curl -fsSL https://code-server.dev/install.sh | sh
# configure password in ~/.config/code-server/config.yaml
systemctl --user enable --now code-server@$USER
```

```sshconfig
# in laptop ~/.ssh/config
Host remote-code
    HostName <vps-ip>
    User <user>
    LocalForward 8080 localhost:8080
```

When the laptop's VSCode application is broken, `ssh -N remote-code` + browser to `http://localhost:8080` restores an IDE within seconds. The first-time install takes thirty minutes; the recovery time when needed is thirty seconds.

### ☐ 2.2 A second terminal application

The laptop's default terminal (Terminal.app on macOS, GNOME Terminal on Linux, Windows Terminal on Windows) can occasionally misbehave, fail to update, or refuse to launch. A second terminal application — installed in advance — guarantees that "open a terminal" is never blocked by a single piece of software.

| Operating system | Suggested alternatives |
|---|---|
| macOS | iTerm2, Alacritty, WezTerm, Kitty |
| Linux | Alacritty, Kitty, WezTerm, Konsole, xterm |
| Windows | Windows Terminal, ConEmu, Alacritty, WSL terminal |

Installation is unattended; the alternative sits dormant until needed.

### ☐ 2.3 Familiarity with `sshfs`

Practice mounting the remote home directory once, today, so the procedure is known when needed. See [02-SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD) §8 for the details.

```bash
mkdir ~/remote
sshfs user@remote:/home/user ~/remote
ls ~/remote
fusermount -u ~/remote     # or `umount ~/remote` on macOS
```

`sshfs` becomes the foot in the door when interactive tools fail. The laptop's file manager, text editor, or any local application can open files on the remote — no SSH or IDE needed.

### ☐ 2.4 Claude Code CLI (or equivalent) installed on the remote

For an AI-agent fallback when interactive editing is impractical:

```bash
npm install -g @anthropic-ai/claude-code
```

A single `claude` invocation in any terminal opens a conversational interface to the entire codebase. For tasks that involve "make this change across many files," or "investigate why this is broken," the agent often resolves them faster than manual editing — and works without an IDE.

### ☐ 2.5 Documented `~/.ssh/config` on the laptop

The `~/.ssh/config` file on the laptop is canonical infrastructure. Lose it, lose the ability to reach hosts by their friendly names. Mitigations:

- Keep `~/.ssh/config` in a separate git repository (the `dotfiles` pattern), committed and pushed
- Or back up to a password manager attachment
- Or replicate to a secure file in cloud storage

The contents of `~/.ssh/config` are not secret (no keys, no passwords), but they encode the topology of personal infrastructure. Rebuilding from memory takes hours; restoring from backup takes seconds.

### ☐ 2.6 Browser bookmarks for tunneled services

Maintain a browser bookmark folder containing `http://localhost:<port>` URLs for every tunneled service in use (Cockpit, Netdata, code-server, databases, dashboards). When a tunnel is opened, the bookmark goes to the right place without typing.

Browsers sync bookmarks across devices (Firefox Sync, Chrome Sync), so this works across laptop replacements.

### ☐ 2.7 Documented runbook for "VSCode is broken"

A written, short procedure for the scenario "VSCode crashes / will not reinstall / extension breaks". Should fit on one screen, perhaps in a `runbook.md` in this repository. Example:

```
1. Open laptop terminal (default or backup).
2. Run: ssh -N remote-code
3. Open browser → http://localhost:8080
4. Authenticate with code-server password (in password manager).
5. Resume work in code-server until VSCode is restored.
```

Five lines. When the situation actually arises, no decisions need to be made — the runbook is followed.

---

## Tier 3 — Belt-and-braces

### ☐ 3.1 Hardware security key

A hardware key (YubiKey, Titan, Solo) supporting SSH ed25519-sk keys provides phishing-resistant authentication. Even an attacker with the laptop and its password cannot SSH without the physical key.

Setup:

```bash
ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_sk
```

The private key file is meaningless without the hardware token; the corresponding public key goes into `~/.ssh/authorized_keys` on the remote. Touching the hardware key authorises each SSH connection.

Trade-off: requires the physical token; losing it means using a backup key (one is usually registered). Worth it for higher-stakes infrastructure.

### ☐ 3.2 Secondary remote host

A second VPS, perhaps from a different provider, configured similarly to the primary. Snapshot-restored from the primary periodically, or kept synchronised via `rsync`. Use cases:

- Primary VPS provider has an outage
- Primary VPS is destroyed and recovery from snapshot fails
- Provider account is suspended

For most personal-use scenarios this is over-engineered. For revenue-generating or otherwise critical workflows, it is justified.

### ☐ 3.3 Local development environment as fallback

For the workflows that *can* run on a laptop alone (small projects, scripting, light data work), maintain the ability to switch to a fully local environment in an emergency. This usually means:

- A current local version of the project's runtime (Node, Python, etc.)
- A local copy of recently committed code
- Local Docker capability

Most of the time this capability is unused; in the scenario where the remote is genuinely lost and no replacement is yet provisioned, having a local fallback prevents a multi-day work stoppage.

### ☐ 3.4 Mobile access via SSH app

Install Termius, Blink Shell, or another mobile SSH app on a phone. Configure with the same `~/.ssh/config`. The phone becomes an emergency thin client — slow, awkward, but capable of `ssh remote && tmux attach` to investigate, kill a runaway process, or perform a quick fix from any location.

### ☐ 3.5 Tested restore procedure

A backup that has never been restored is a hypothesis. Periodically (annually at minimum), perform a test restore:

- Spin up a fresh VPS from the most recent snapshot
- Verify SSH works
- Verify the editor environment works
- Verify a recent project builds and runs

If this fails, the backup strategy needs revision *before* it is needed for real.

---

## Quick-reference summary card

| Tier | Item | Status |
|---|---|---|
| 1 | Plain SSH from terminal works | ☐ |
| 1 | Non-IDE editor on remote (`micro`/`helix`/`nano`) | ☐ |
| 1 | `tmux` on remote, sessions used by default | ☐ |
| 1 | SSH key has passphrase + backed up | ☐ |
| 1 | Recent VPS snapshot | ☐ |
| 2 | `code-server` installed and tunnelable | ☐ |
| 2 | Second terminal application on laptop | ☐ |
| 2 | `sshfs` installed and verified | ☐ |
| 2 | Claude Code CLI on remote | ☐ |
| 2 | `~/.ssh/config` backed up | ☐ |
| 2 | Browser bookmarks for tunneled services | ☐ |
| 2 | Written runbook for VSCode failure | ☐ |
| 3 | Hardware security key | ☐ |
| 3 | Secondary remote host | ☐ |
| 3 | Local development environment | ☐ |
| 3 | Mobile SSH app | ☐ |
| 3 | Annual restore test | ☐ |

A reasonable target: all of Tier 1 within the first week of remote-first adoption; all of Tier 2 within the first month; Tier 3 selectively, based on the work's importance.

---

## Further reading

- [02-SSH-SUPERPOWERS.MD](02-SSH-SUPERPOWERS.MD) — the SSH features that underpin most fallbacks
- [04-EDITORS-WITHOUT-VSCODE.MD](04-EDITORS-WITHOUT-VSCODE.MD) — depth on editor alternatives
- [06-SSH-CONFIG-RECIPES.MD](06-SSH-CONFIG-RECIPES.MD) — managing the `~/.ssh/config` file referenced throughout

External:

- *Site Reliability Engineering*, Beyer et al. (Google) — Chapter 6 on monitoring and Chapter 9 on simplicity, both relevant to maintainable personal infrastructure
- Hetzner / DigitalOcean / Linode documentation on snapshots — specific to whatever VPS provider is in use
