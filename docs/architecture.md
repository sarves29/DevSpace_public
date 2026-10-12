# Architecture

## Layout

```
server/
  index.js          express + ws bootstrap, WebSocket routing
  auth/             WebAuthn ceremonies, signed sessions, device enrolment, middleware
  claude/           claude CLI sessions, history, scheduled jobs, token ledger, usage
  terminal/         node-pty over WebSocket, ring-buffer replay
  files/            sandboxed file CRUD, streamed ZIP writer
  git/              git + gh wrappers (argv arrays only)
  projects/         project registry
  system/           remote-desktop bridge, host watchdog reader
  notify/           Web Push, Telegram
  db/               JSON store, backups
public/             single-page app — vanilla JS, no build step
deploy/             systemd unit + nginx vhost template
scripts/            installer helpers, watchdog, staging, deploy verifier
qa/                 test harnesses
docs/               operator documentation
```

## How a Claude chat works

1. The browser opens `/ws/claude` (session cookie checked on upgrade).
2. The server spawns `claude --input-format stream-json --output-format stream-json`
   in the project's directory, with the chosen model and permission mode.
3. Each JSON event from the CLI is forwarded to the client, which renders text,
   tool calls, diffs, images, subagents and usage as they stream.
4. Permission requests travel over the CLI's control protocol; the UI (or a push
   notification) answers them.
5. Changing model or mode restarts the CLI with `--resume`, so context is kept.

## Process model

| Thing | Lifetime | Cap | Reaped when |
|---|---|---|---|
| PTY | Independent of the socket | 8 | Detached and idle past its TTL |
| Claude CLI | Independent of the socket | 12 | Detached and idle — never while a permission prompt is pending |
| HTTP session | 30 days | — | Expiry or explicit revoke |

At capacity the stalest **detached** process is evicted; an attached one never is.

## Connection resilience

Phones fail in three different ways, so there are three independent mechanisms:

1. **Server → client ping** every 30 s kills dead sockets, so detach-and-buffer
   starts within a minute of a phone locking.
2. **Client → server ping** every 25 s, plus an immediate reconnect on
   `visibilitychange` / `online` — unlocking reattaches in about a second.
3. **Exponential-backoff reconnect**, falling back to `--resume` for chats and a
   fresh shell for terminals if the process is truly gone.

When a second tab adopts a session, the first is told `attached_elsewhere` and
stops reconnecting — otherwise two tabs fight over it forever.

## Storage

No database. A single JSON document (credentials, sessions, projects, push
subscriptions, settings) is loaded at boot and written atomically on change,
with nightly backups. Audit events go to an append-only JSON-lines log with
in-process rotation. Per-device UI state lives in `localStorage`; anything that
would hurt to lose on a PWA reinstall (e.g. quick prompts) lives server-side.

## Deployment

- systemd unit bound to loopback, behind nginx TLS.
- One-button deploy = fast-forward pull → dependency install → restart, executed
  by a root-owned path unit outside the portal's own cgroup.
- A staging instance on its own port and worktree deploys any committed ref.
- A verifier compares the commit the server booted from with `HEAD` and flags
  server edits made after boot.
- Installer works on any box: wizard on first install, silent upgrade after,
  `--dry-run` without root, nginx vhost generated from a template.
