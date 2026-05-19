# Editors Without VSCode

VSCode is the dominant tool for remote-first development as of 2026, but it is not the only one — and a working setup should not depend on a single tool. This document surveys alternatives and fallbacks across three categories: browser-served IDEs, terminal editors, and AI-agent-driven workflows. The goal is **resilience** as much as exploration: when the primary editor becomes unavailable, work should continue.

## 1. Categories of alternative

| Category | Examples | When useful |
|---|---|---|
| Browser-served IDEs | code-server, OpenVSCode Server, Theia | VSCode itself is broken; want IDE feel from any device |
| Terminal editors (modern) | Helix, Neovim, micro | SSH-only access; speed; permanence |
| Terminal editors (traditional) | Vim, Emacs | Available almost everywhere; deeper power for those who invest |
| AI-agent terminals | Claude Code CLI, Aider | Conversational editing without a UI |
| Minimal fallback | nano (preinstalled) | Five-second emergency edits |

A robust setup has multiple options across these categories. The recommended posture (covered in [05-RESILIENCE-CHECKLIST.MD](05-RESILIENCE-CHECKLIST.MD)) is: VSCode as primary, code-server as browser fallback, micro or Helix as terminal fallback, nano as last resort.

---

## 2. code-server — VSCode in a browser

The closest functional substitute for VSCode in remote-first workflows. Maintained as an open-source project by Coder Inc., it runs the same VSCode codebase as the desktop client, served over HTTP.

### Installation (on the remote host)

```bash
curl -fsSL https://code-server.dev/install.sh | sh
```

The installer is idempotent and works on Debian, Ubuntu, Fedora, RHEL, Arch, and Alpine. On systemd-based systems it installs a per-user service.

### Configuration

```bash
# Initial password (or PAM-based auth)
mkdir -p ~/.config/code-server
cat > ~/.config/code-server/config.yaml <<'EOF'
bind-addr: 127.0.0.1:8080
auth: password
password: <set-a-strong-password>
cert: false
EOF
```

`bind-addr: 127.0.0.1:8080` restricts the listener to localhost on the remote — access only via SSH tunnel, never directly from the internet. `cert: false` disables self-signed-cert generation (the SSH tunnel provides confidentiality).

### Starting

```bash
systemctl --user enable --now code-server@$USER
# or, ad-hoc
code-server
```

### Access from the laptop

```bash
ssh -N -L 8080:localhost:8080 user@remote
# then open http://localhost:8080 in the laptop browser
```

For a permanent alias, add to `~/.ssh/config` on the laptop:

```sshconfig
Host vps1-code
    HostName <vps-ip>
    User <user>
    LocalForward 8080 localhost:8080
```

Then `ssh -N vps1-code` opens the tunnel.

### Extensions

code-server uses Open VSX Registry by default (an open-source mirror of the VSCode Marketplace, which is restricted by Microsoft licensing). Most popular extensions are available there. To install a specific extension that exists only in Microsoft's marketplace, manually download the `.vsix` and install it via the UI.

### Trade-offs vs native VSCode Remote-SSH

| Aspect | code-server | Native VSCode Remote-SSH |
|---|---|---|
| Where the UI runs | Server-side, in browser | Client-side, native app |
| Latency | Network round-trip per keystroke | Lower for UI interactions |
| Resource use on laptop | Browser tab only | Electron app |
| Resource use on remote | Higher (rendering + IDE backend) | Lower (just IDE backend) |
| Extensions | Open VSX (mostly compatible) | Full Microsoft marketplace |
| Required setup per remote | One-time install | One-time install |
| Multi-device convenience | Excellent (any browser) | Requires VSCode locally |

For most developers, native Remote-SSH is the preferred everyday tool; code-server is the better choice when device independence matters or when the laptop cannot install VSCode for any reason.

---

## 3. OpenVSCode Server

Microsoft-published variant of the same idea. Source code at `gitpod-io/openvscode-server`. Functionally similar to code-server; differences are in licensing, default ports, and minor UI tweaks. Comparable as a fallback.

```bash
# Run as a container
docker run -d \
    -p 127.0.0.1:3000:3000 \
    -v "$(pwd):/home/workspace:cached" \
    gitpod/openvscode-server
```

---

## 4. Theia

Eclipse Foundation's VSCode-compatible IDE platform. Used as the basis for several commercial cloud IDEs. Less polished than code-server for individual use, but interesting if customisation or embedding into other products is a concern.

---

## 5. Terminal editors — modern

### Helix

The most approachable of the modern terminal editors. Written in Rust, single-binary install, sensible defaults out of the box. Vim-influenced keybindings but with on-screen prompts that show available actions — substantially easier to learn than Vim itself.

Installation:

```bash
sudo apt install helix              # may be available in newer distros
# or via cargo
cargo install helix-term --locked
# or via release tarball from https://helix-editor.com
```

First impressions:

- Press `:` for command mode (like vim)
- Press `space` to bring up the "leader menu" listing available actions
- Press `Esc` to return to normal mode
- Press `i` for insert mode

Without further configuration, Helix provides syntax highlighting, LSP integration for most major languages, tree-sitter-based navigation, multi-cursor editing, and a file picker. Configuration is minimal TOML in `~/.config/helix/config.toml`.

Worth investing a weekend in. Productive within hours; expert within months.

### Neovim

Vim's modern, actively developed fork. The most flexible and powerful terminal editor available, but with the steepest learning curve. Production-grade plugins, asynchronous architecture, Lua scripting, first-class LSP.

The barrier to a productive Neovim setup is configuration. Two paths:

1. **Use a pre-built distribution** — LazyVim, AstroNvim, NvChad, LunarVim. Install in one command, get IDE-class features immediately. Trade-off: opinionated, sometimes hard to debug.
2. **Use kickstart.nvim** — a minimal single-file config that demonstrates how to assemble Neovim from primitives. Trade-off: requires reading Lua; rewards understanding.

```bash
# Install Neovim
sudo apt install neovim
# Then choose: LazyVim install or kickstart.nvim
```

For someone who already uses Vim, Neovim is a strict upgrade. For a developer new to terminal editors, **Helix is the better starting point**. Neovim becomes interesting after a few months of comfortable terminal-editor usage.

### micro

The friendliest terminal editor possible: arrow keys for movement, `Ctrl+S` to save, `Ctrl+Q` to quit, normal selection with `Shift+arrows`, normal copy/paste with `Ctrl+C`/`Ctrl+V`. No modes. No surprise keybindings.

Installation:

```bash
sudo apt install micro
# or
curl https://getmic.ro | bash
```

micro provides syntax highlighting, multi-cursor editing, plugin support, and a sensible default theme. It is, deliberately, a more capable nano — not an attempt to compete with Vim or VSCode.

Use case: the safe, no-investment fallback. Five minutes to install; effective immediately; nothing to learn. Recommended for any developer's remote toolkit as a non-Vim terminal editor of last resort.

---

## 6. Terminal editors — traditional

### Vim

The oldest of the actively used terminal editors. Available on essentially every Unix system. Modal editing (separate modes for navigation, insertion, command entry), keyboard-centric, deeply scriptable.

Vim's value proposition is *muscle-memory leverage*: once internalised, editing in Vim is faster than mouse-driven editing because hands never leave the home row. Reaching that state requires sustained investment — weeks to be productive, months to be efficient, years to plateau.

Worth learning if (a) terminal-only editing is a frequent part of work, or (b) the user values the universality of Vim's availability. Not worth learning purely as a fallback — the time investment is large, and lighter alternatives (Helix, micro) cover the fallback case adequately.

### Emacs

The other ancient terminal editor. Functionally a Lisp interpreter that happens to default to editing text. Famously customisable; the OS-within-an-OS comparison is not entirely a joke.

Practical Emacs use is dominated by frameworks (Doom Emacs, Spacemacs) that pre-configure it for modern development. The investment curve is similar to Vim's; the resulting capability is at least as high.

Emacs and Vim users tend to be in distinct camps; the choice between them is usually settled early and seldom revisited. Recommended only if a specific tool in the Emacs ecosystem (org-mode, Magit, projectile) is independently attractive.

---

## 7. AI-agent terminals

A category that did not exist meaningfully five years ago. AI coding agents reachable via the shell turn the terminal into a conversational IDE.

### Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
claude
```

Runs in any terminal, including over SSH. Reads and writes files, executes shell commands, navigates the codebase, performs multi-step refactors with explanation. The interaction model is closer to pairing with a colleague than to operating a tool.

Particular fit for remote-first workflows: Claude Code lives on the same host as the project, with full filesystem and shell access; no client-server data crossing is required.

### Aider

```bash
pip install aider-chat
aider --model claude-3-5-sonnet-20241022
```

Similar agent-style tool, BYO API key, supports multiple model providers. Strong git integration (auto-commits each change, easy to revert).

### Differences from a regular editor

These tools are not drop-in replacements for editors — they augment, not substitute. Used together with an editor (Vim, code-server, VSCode), they remove the friction of "I know what I want to do, but writing it out is tedious." For routine editing, an actual editor remains faster.

---

## 8. The emergency fallback: nano

Every Debian and Ubuntu system has `nano` preinstalled. It is the minimum viable text editor:

- Arrow keys move the cursor
- `Ctrl+O` saves
- `Ctrl+X` exits
- `Ctrl+K` cuts a line, `Ctrl+U` pastes

It has no syntax highlighting (without configuration), no plugins, no LSP. It is not pleasant for serious editing. It is reliable: it always works, requires no learning, and survives any context.

Use case: editing `/etc/sudoers` once, fixing a config file when nothing else is available, performing the kind of edit that a Vim user would do with `vipw` and a non-Vim user would do with the file manager. Always available, never the right choice for prolonged work.

---

## 9. JetBrains IDE Remote Development

For developers already using IntelliJ IDEA, PyCharm, GoLand, RubyMine, or WebStorm, **JetBrains Gateway** provides a remote-development experience comparable to VSCode Remote-SSH.

- The IDE backend runs on the remote host
- The frontend runs locally as a thin JetBrains Client (separate from the IDE itself)
- Source indexing, refactoring, debugging — all happen remote-side

JetBrains' implementation differs from VSCode's in that the local client is a dedicated thin client, not the full IDE. Performance is generally good; the model is mature as of 2026.

Available via the JetBrains Toolbox application.

Trade-off: the JetBrains IDEs are paid for non-open-source projects, though Community Editions exist for several (IntelliJ IDEA Community, PyCharm Community).

---

## 10. Browser-based IDEs from the major cloud providers

Mentioned for completeness; not the focus of this repository, which is concerned with self-hosted remote workflows.

| Service | Notes |
|---|---|
| GitHub Codespaces | VSCode in browser, container per repository, paid beyond free tier |
| Gitpod | Multi-provider; can be self-hosted |
| Coder | Self-hosted variant of the same pattern, enterprise-focused |
| AWS Cloud9 | Deprecated for new accounts; still available for existing users |
| Replit | Educational and rapid-prototyping focus |

Codespaces is the practical default for GitHub-centric work; for the self-hosted equivalent on personal infrastructure, code-server is the correspondent.

---

## 11. A recommended composite setup

For a developer running remote-first development with this repository's posture:

| Role | Tool |
|---|---|
| Primary editor | VSCode with Remote-SSH (or JetBrains Gateway, if preferred) |
| Browser fallback | code-server installed on the remote, port 8080 reachable via tunnel |
| Terminal editor — learn this one | Helix (most accessible modern option) |
| Terminal editor — last resort | nano (already there) |
| AI agent | Claude Code CLI in the integrated terminal |
| Mobile / tablet access | code-server via tunnel from a mobile SSH app |

Installation cost: ~30 minutes (code-server install, Helix install, alias configuration). Ongoing maintenance: minimal. Disaster-recovery time when VSCode breaks: 30 seconds (open the browser, navigate to the tunneled code-server URL).

---

## Further reading

- [05-RESILIENCE-CHECKLIST.MD](05-RESILIENCE-CHECKLIST.MD) — actionable steps to take before the primary editor fails
- [06-SSH-CONFIG-RECIPES.MD](06-SSH-CONFIG-RECIPES.MD) — how to add tunnel aliases for browser-served IDEs

External:

- code-server: <https://coder.com/docs/code-server>
- Helix: <https://helix-editor.com/>
- Neovim: <https://neovim.io/> and <https://github.com/nvim-lua/kickstart.nvim>
- micro: <https://micro-editor.github.io/>
- JetBrains Gateway: <https://www.jetbrains.com/remote-development/gateway/>
