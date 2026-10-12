# Engineering notes

A few problems from building DevSpace that were more interesting than they looked.

## Sessions that survive a phone locking

A mobile browser freezes JavaScript and drops sockets when the screen locks.
If the terminal or the AI process lived on the socket, every lock would kill
the work. Instead both are server-side processes with their own lifetime:
output is buffered (a 256 KB ring for terminals, an event history for chats)
and replayed when the client reattaches. Pings in both directions plus
reconnect-on-`visibilitychange` bring a phone back in about a second, and idle
processes are reaped — except a chat waiting on a permission prompt, which must
never be killed while the operator is deciding.

## Renaming the host without re-enrolling a single device

WebAuthn credentials are bound to the relying-party ID, and with passkeys-only
auth there is no fallback: change it carelessly and every device is locked out.
The RP ID was migrated once, to the **apex domain**, so every future subdomain
rename is free. The installer now defaults the target to whatever is already
configured — a run that cannot name a new target cannot invalidate anything.

## The hardening that was never on

systemd hardening directives had been in the unit file for weeks. Checking the
*running* process (`/proc/<pid>/status` → `NoNewPrivs`) showed they were not
applied — the deploy path never reinstalled the unit. Lesson: verify a security
control on the live process, not in the config file. A deploy verifier now
compares the booted commit with `HEAD` for the same reason.

## The watchdog that went blind

Turning on `PrivateTmp` gave the service its own `/tmp` and `/var/tmp`. The cron
watchdog kept running perfectly — but published its verdict into the *real*
`/var/tmp`, which the portal could no longer see, so the health panel went
blank. The state file moved into the app's data directory, and a verdict older
than three intervals is now reported as a fault rather than silence.

## Approving tool use from the lock screen

Permission prompts arrive as Web Push notifications with Approve/Deny buttons.
The service worker handling the click cannot read the page's CSRF token, so the
endpoint is instead guarded by a single-use random nonce that only ever travels
inside the encrypted push payload, plus the session cookie.

## No emoji in the UI

Emoji render differently — or not at all — across phone fonts; one arrow
character drew nothing on Android. Every pictograph comes from one inline-SVG
icon table, so the UI looks the same on every device.

## Several portals on one phone

Each server runs its own portal, and they are all installed as PWAs. A fixed
name and icon meant identical launcher icons with no way to tell servers apart.
The web manifest is now generated per install (name from settings or hostname)
and the icon ships in eight colourways.

## No build step, on purpose

The portal is the tool you use to repair the server. If fixing it required a
working Node toolchain and bundler, a broken build would lock you out of the
fix. So the frontend is plain ES modules served as written.
