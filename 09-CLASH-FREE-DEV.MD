# Clash-Free Dev

## Overview

When several projects share one remote dev box, they collide in five places: ports, network exposure, databases, browser state and leftover processes. Only the first is obvious. This document gives one rule for each, so any project's `make dev` starts cleanly no matter what else is running, and a laptop that sleeps and wakes costs nothing more than rerunning it. The core idea is a **registered base port per project with a free-port fallback**, wired through environment variables to everything that needs the port, and bound to loopback only. The browser-state rule is the subtle one: cookies ignore ports and storage does not, so two apps on `localhost` can sign each other out, and an app that drifts to a new port seems to lose its local data. A copyable launcher script and a checklist for new projects close the document.

---

## The five clashes

| Clash | What goes wrong | Rule |
|---|---|---|
| **Ports** | Two dev servers want 5173 or 3000; the second fails or silently takes another port. | Each project registers its own base port; the launcher steps up to the next free port only as a fallback, and says so loudly. |
| **Exposure** | A dev server bound to all interfaces is on the box's public address; only the firewall stands between it and the internet. | Dev servers bind `127.0.0.1`. Editor port forwarding and SSH tunnels connect from the box itself, so nothing needs more. |
| **Databases** | Two projects both publish Postgres on 5432, or share a container and its data. | One Compose project per repo, each publishing its database on its own loopback port. |
| **Browser state** | Cookies are per host, not per port: two apps with the same session-cookie name on `localhost` overwrite each other's login. Storage (localStorage, IndexedDB) is per origin, port included: an app that drifts from :5280 to :5281 seems to lose its local data. | Prefix auth cookies per app. Keep the web port stable (the registry's job), and treat a fallback port as a warning, not a mode. |
| **Leftover processes** | A `make dev` from an earlier session still holds the port, so the new one drifts. The usual cause is an editor terminal killed hard (laptop sleep, the VS Code server reaping it): the kernel detaches the terminal without a reliable SIGHUP, and the whole dev tree runs on as orphans. | The launcher stops everything it started when its terminal disappears, so a dead session leaves nothing behind. If a holder remains anyway, the launcher names the port and how to find it (`lsof -i :<port>`). |

## Ports: register a base, fall back loudly

Framework defaults (Vite's 5173, Express's 3000, Django's 8000, `http-alt` 8080) are conventions, not reservations, which is exactly why they clash. Instead:

1. **Keep a registry** of your projects and their base ports, in one private note. Give each project a block of ten for its browser-facing port (5280-5289, 5290-5299, …), clear of the framework defaults, so the usual case is a stable URL.
2. **Internal ports may share a base.** An API reached only through the web server's proxy (`/api` → API) is never a browser origin, so every project can start its API search at 3000 and let it drift freely.
3. **One source of truth.** The launcher picks the ports and passes them as environment variables (`WEB_PORT`, `API_PORT`) to everything that needs them: the dev server, its proxy target, and anything that derives an origin from the port (auth callback URLs, trusted origins). Never write the port a second time anywhere.
4. **Fail loudly, not silently.** Run the dev server with `strictPort` (or the framework's equivalent), so a race between "port is free" and "server binds it" errors instead of quietly drifting again.

## Exposure: loopback only

Bind every dev server to `127.0.0.1`. Write the address out, not `localhost`: Node may resolve `localhost` to IPv6 `::1` first, and a proxy aiming at `localhost` can then miss a server on `127.0.0.1`. With Vite, that is `server.host: "127.0.0.1"` and a proxy target of `http://127.0.0.1:${API_PORT}`. With `@hono/node-server`, that is `serve({ fetch, port, hostname: "127.0.0.1" })` in development. Production containers keep their own binding, because the reverse proxy reaches them over the container network.

On a dev-only box the firewall then needs nothing but SSH (22) open. See [SECURITY.MD](08-SECURITY.MD) and [WHAT-TO-TUNNEL.MD](03-WHAT-TO-TUNNEL.MD).

## Databases: one Compose project each

- Compose names a project after its directory by default, which already separates containers, networks and volumes per repo. Set `name:` at the top of `compose.yaml` only if two checkouts share a directory name.
- Publish the database on loopback and on the project's registered port: `"127.0.0.1:5433:5432"`. The container still listens on 5432 inside; only the host side changes.
- Rootless Docker publishes ports without touching the firewall, so the loopback address in the mapping is the control that matters. See [DESKTOP-GUI-WORKFLOWS.MD](07-DESKTOP-GUI-WORKFLOWS.MD) for reaching it from a laptop GUI.

## Browser state: prefix cookies, keep the origin

- **Cookies.** Give every app's session cookie its own name. Libraries default to a shared prefix (Better Auth uses `better-auth.*`; others use `session`, `connect.sid`, `next-auth.*`). In Better Auth that is `advanced: { cookiePrefix: "<app>" }`; most libraries have an equivalent.
- **Storage.** localStorage and IndexedDB belong to the exact origin, port included. Local-first apps are the most exposed: their data lives there. A stable registered port is what keeps it reachable.

## The launcher

A dependency-free Node script, run by `make dev` or `pnpm dev`. It picks ports, warns on fallback, passes ports through the environment, forwards Ctrl-C to the dev servers, and stops the whole tree quietly when its terminal dies. Adjust the four constants and the `spawn` command to the project.

```js
// scripts/dev.mjs: clash-free dev launcher. See remote-dev 09-CLASH-FREE-DEV.MD.
import net from "node:net"
import { spawn } from "node:child_process"
import { readFileSync, readdirSync } from "node:fs"

const NAME = "myapp"     // shown in the banner
const WEB_BASE = Number(process.env.WEB_PORT) || 5290 // the registered base port
const API_BASE = Number(process.env.API_PORT) || 3000 // internal, shared base
const SCAN = 100

/** True only if nothing is bound to `port` on any interface. */
function isFree(port) {
  return new Promise((resolve) => {
    const srv = net.createServer()
    srv.once("error", () => resolve(false))
    srv.once("listening", () => srv.close(() => resolve(true)))
    srv.listen(port, "0.0.0.0")
  })
}

/** First free port at or above `base`, skipping any in `taken`. */
async function freePortFrom(base, taken = new Set()) {
  for (let port = base; port < base + SCAN; port += 1) {
    if (taken.has(port)) continue
    if (await isFree(port)) return port
  }
  throw new Error(`No free port found in ${base}..${base + SCAN - 1}`)
}

const webPort = await freePortFrom(WEB_BASE)
const apiPort = await freePortFrom(API_BASE, new Set([webPort]))

console.log(`\n  ▸ ${NAME} dev\n    web   http://localhost:${webPort}\n    api   http://localhost:${apiPort}  (internal)\n`)

// A different web port is a different browser origin: localStorage and
// IndexedDB from the usual port are not visible on this one (cookies are per
// host, so they carry over).
if (webPort !== WEB_BASE) {
  console.warn(
    `  ⚠ Port ${WEB_BASE} is taken, so web is on ${webPort}: a different browser origin.\n` +
      `      lsof -i :${WEB_BASE}    # see what holds it, stop it, rerun\n`,
  )
}

const child = spawn("pnpm", ["-r", "--parallel", "dev"], {
  stdio: "inherit",
  env: { ...process.env, WEB_PORT: String(webPort), API_PORT: String(apiPort) },
})
process.on("SIGINT", () => child.kill("SIGINT"))
process.on("SIGTERM", () => child.kill("SIGTERM"))
process.on("SIGHUP", () => stopTree())
// While stopping, stopTree() owns the exit, so stragglers still get SIGKILL.
child.on("exit", (code) => {
  if (!stopping) process.exit(code ?? 0)
})

// A dev server must not outlive its terminal. A terminal killed hard is
// detached without a reliable SIGHUP, so watch for it to disappear and stop
// everything we started. Linux-only (/proc); elsewhere this is a no-op.
const ttyAtStart = procStat("self")?.tty
if (ttyAtStart && ttyAtStart !== "0") {
  setInterval(() => {
    if (procStat("self")?.tty === "0") stopTree()
  }, 1000).unref()
}

/** ppid and tty_nr from /proc/<pid>/stat, or null off Linux / once the pid is gone. */
function procStat(pid) {
  try {
    const stat = readFileSync(`/proc/${pid}/stat`, "utf8")
    // Fields after the parenthesised command name, which may itself hold spaces.
    const fields = stat.slice(stat.lastIndexOf(")") + 2).split(" ")
    return { ppid: Number(fields[1]), tty: fields[4] }
  } catch {
    return null
  }
}

/** Every live descendant of `root`, found by walking /proc's parent links. */
function descendants(root) {
  const children = new Map()
  for (const entry of readdirSync("/proc")) {
    if (!/^\d+$/.test(entry)) continue
    const stat = procStat(entry)
    if (!stat) continue
    if (!children.has(stat.ppid)) children.set(stat.ppid, [])
    children.get(stat.ppid).push(Number(entry))
  }
  const found = []
  const queue = [root]
  while (queue.length) {
    for (const pid of children.get(queue.shift()) ?? []) {
      found.push(pid)
      queue.push(pid)
    }
  }
  return found
}

/** Stop every process this launcher started, then exit. Quiet: the terminal may be gone. */
let stopping = false
function stopTree() {
  if (stopping) return
  stopping = true
  const signalAll = (signal) => {
    for (const pid of descendants(process.pid)) {
      try {
        process.kill(pid, signal)
      } catch {
        // Already exited.
      }
    }
  }
  signalAll("SIGTERM")
  setTimeout(() => {
    signalAll("SIGKILL")
    process.exit(0)
  }, 2000)
}
```

Probing `0.0.0.0` detects a holder on any interface, including one bound only to loopback. The scan is a convenience; `strictPort` in the dev server is the guarantee.

The teardown signals only the launcher's own descendants, found through `/proc`, never its process group, which can include whatever invoked it. It is armed only when the launcher starts with a terminal, so CI and other non-interactive runs never trip it.

## Checklist for a new project

1. Claim the next free block in the registry and note the database port.
2. Add the launcher as `scripts/dev.mjs` with that base; point `make dev` (and `pnpm dev`) at it.
3. Dev server: `host: "127.0.0.1"`, `strictPort: true`, port from `WEB_PORT`, proxy to `http://127.0.0.1:${API_PORT}`.
4. API: listen on `127.0.0.1` outside production; port from `API_PORT`; derive auth origins from `WEB_PORT`.
5. Database: `"127.0.0.1:<registered>:5432"` in `compose.yaml`.
6. Auth: set a per-app cookie prefix.

## Verify

```bash
ss -ltn | grep -E ':(52[89][0-9]|30[0-9][0-9]) '   # dev listeners: every one should be 127.0.0.1
lsof -i :5280                                       # who holds a port
docker compose ls                                   # one Compose project per repo
```

Anything listening on `0.0.0.0` or `[::]` other than SSH is a dev server that skipped step 3 or 4.
