# Security posture

The data is medical, the development machine is also a work machine, and the
production host sits on a home network. Each of those is a different threat
model and they are treated separately.

---

## Threat model

What this defends against:

- Another device or person on the home Wi-Fi reaching the dashboard
- The database being readable by another local account
- Health data leaking into a client-side JavaScript bundle
- Health data being served by an API endpoint that was added for convenience
- A screen-share or shoulder-surf exposing the app incidentally
- A third-party dependency phoning home with request data

What it explicitly does **not** defend against, stated rather than glossed:

- Corporate device management, endpoint monitoring or data-loss prevention.
  These see process names and file paths regardless of file permissions.
- The root user, who ignores permission bits entirely.
- Anyone able to read network traffic on the LAN, where plain HTTP is used.
- OS-level filename indexing, unless the data directory is manually excluded.

The last four are judgement calls for the owner, not code problems, and are
documented as such.

---

## Network

**Loopback by default.** Development binds `127.0.0.1` and must stay that way.
A LAN bind exists for phone access, is opt-in on every invocation, and is never
the default.

**The bind address and the auth gate are one unit.** The variable that tells the
proxy whether to require a password is derived from the *same constant* that is
passed as the bind address, in the same file. They cannot drift apart. Changing
one without the other is the catastrophic bug, so it is made unrepresentable
rather than documented as a caution.

**Fail closed.** In LAN mode an unset password returns 503 to every request
rather than serving unauthenticated content. The loopback exemption is tested
*before* that check, so a fresh clone with no configuration can still run
locally, while a LAN bind without a password refuses to serve at all.

**The loopback exemption matches only a literal loopback address.** Unset,
empty, misspelled, and `localhost` all keep the password. The safe default is
"ask". The check is never keyed off a request header, because headers are
attacker-controlled.

Password comparison is constant-time, hashing both sides first because the
underlying comparison throws on a length mismatch — which would itself be an
oracle.

**Production is a private tailnet**, with no published host ports and a
userspace VPN sidecar as the only ingress. That is the real answer; the LAN
password is a convenience for a phone on the same network.

---

## Application surface

**One API route exists**, and it triggers a sync. A `GET` returns timestamps.
**No route serves health data**, and adding one is prohibited — the rule is
written where someone would look before adding an endpoint.

Pages are server components. Query results are rendered server-side and the
client receives values, not a data layer.

**No public environment variables, ever.** Framework-prefixed public variables
are inlined into the client bundle at build time. Credentials are read from the
environment at runtime and never logged.

**Fonts are self-hosted.** Nothing is fetched from a third party at runtime, so
no external service receives a request correlated with using a medical app.

**Four npm dependencies.** The smallest supply chain that can do the job. The
sync's Python environment deliberately omits a popular auth library that is
deprecated and pulls in a telemetry stack behind it.

---

## At rest

The database and the credentials file are owner-read/write only; the containing
directory is owner-only, so the filenames are not even listable.

SQLite copies the database file's mode onto its write-ahead log and shared-memory
files, which was verified across two clients rather than assumed — a `600`
database with a `644` WAL beside it would defeat the whole exercise.

The credentials file is owner-managed: not read, not printed, not written to.

---

## Writes and sessions

Basic auth alone broke every write, and the failure was instructive. Reads
worked; writes returned 401; it presented as a dead button with no error.

The cause is that a `fetch` POST does not replay saved Basic credentials the way
a page navigation does. Basic now mints an HMAC-signed, `httpOnly`, `sameSite`
session cookie, which `fetch` does send.

The cookie is deliberately **not** flagged `secure`, because that flag means
HTTPS-only and would stop it being sent over plain HTTP on the LAN —
reinstating the original bug. The token is signed with a key derived from the
password, so rotating the password invalidates every existing session.

**Any change to auth is tested against a cookie-only POST.** A passing `GET`
proves nothing about writes. That is now a deployment smoke test performing a
real authenticated write against a disposable copy of the database before a
build is promoted.

---

## Operational separation

The development checkout and the live database are deliberately separated, and
the separation is enforced by a role marker inside the database itself rather
than by discipline.

The development copy is explicitly non-authoritative. Live reads and writes go
through a remote wrapper that holds the deployment lock for the query's
lifetime, takes SQL on standard input, accepts no arguments, and can be opened
read-only by a pragma for anything that only needs to read.

**If the remote path fails, the correct behaviour is to stop and say so** — never
to fall back to writing locally. A silent local write is the exact failure the
split exists to prevent, because nothing errors, the figure looks right, and the
real dashboard never sees it.

Schema changes are gated by an approved manifest: a migration runs because its
hash is listed, not because the file exists. The deploy and the sync share a
lock so neither can run while the other holds the database.

---

## Incidental exposure

No GUI, no notifications, no dock or menu-bar presence. Nothing is visible
during a screen share. The only background processes are a scheduled sync and
the server itself.

A discreet mode exists for using the dashboard in a shared space.
