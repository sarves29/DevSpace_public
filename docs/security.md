# Security model

DevSpace exposes an unrestricted shell on a production server to the internet.
The shell is the product, so the boundary is everything *around* it: who can log
in, what the web API can touch, and what a compromised portal process could do.

## Authentication — passkeys only

- WebAuthn platform authenticators (Touch ID, Face ID, Android biometric, Windows Hello).
- **User verification is required.** A stolen, unlocked-but-not-yours phone is not enough.
- Signature-counter regression is treated as a cloned authenticator and alarmed.
- No password, no recovery email, no security question, no admin backdoor.
- The relying-party ID is the apex domain, so one passkey is valid on every
  subdomain and the portal can be renamed without re-enrolling devices.
- New devices are enrolled with a short-lived, single-use token minted on the
  server's CLI — and every enrolment fires a Telegram alert.

## Sessions

- HMAC-SHA256-signed cookies, `HttpOnly`, `Secure`, `SameSite`; the server
  refuses to boot without a strong secret.
- Every active session is listed with IP and last-seen time and can be revoked individually.
- Per-IP rate limits and lockout on repeated authentication failures.

## Filesystem sandbox

1. **Where projects may live** — a project root must sit under a configured set of
   base directories; `/` or `/etc` can never be registered.
2. **Staying inside a project** — every client-supplied path goes through one
   resolver that rejects NUL bytes, neutralises absolute paths and `..`, and
   `realpath`s both the root and the target. A symlink pointing out of a project
   is refused, not followed. String prefixes alone are a textual boundary, not a physical one.

## Serving untrusted content

Project files are attacker-influenced. Serving an `.html` or scripted `.svg`
inline on the portal's origin would be stored XSS against the operator's session, so:

- Inline only for a whitelist of image/PDF types.
- `X-Content-Type-Options: nosniff`, a `sandbox` Content-Security-Policy, `no-store`.
- Everything else is forced to a download.

## Command execution

Every subprocess uses `execFile` with an argv array — never a shell string,
never interpolation. Git calls put `--` before path lists, reject absolute or
`..` paths, disable credential prompts and carry a timeout.

## Privilege containment

- The service runs as an unprivileged user on loopback only.
- systemd hardening: `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem` — setuid
  binaries such as `sudo` do not work from the portal's terminal.
- The few privileged operations (restart, nginx test/reload) go through a
  root-owned wrapper with a **closed verb list** and a sudoers rule naming exact
  arguments. No password is stored anywhere. Nothing writable by the portal user
  is ever the target of a privileged rule.
- The remote-desktop bridge's target is fixed server-side to a loopback port;
  the browser can never name a host or port, so the portal cannot be turned into
  a proxy to other loopback services.
- Encrypted vaults are unlocked from the terminal only — no passphrase ever
  crosses HTTP.

## Detection

- Telegram alerts on every login, failure, enrolment, deploy and push.
- Append-only audit log.
- An independent host watchdog (cron, not the portal) checks disks, services,
  backups, suspicious executables and more; a stale verdict is reported as a
  fault, never as silence.
