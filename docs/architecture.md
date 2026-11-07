# Architecture

## Shape

```
Wearable API ──► Python sync ──┐
                               ├──► health.db (SQLite) ──► Next.js app ──► browser
Manual entry (UI) ─────────────┘                             (server components)
```

One file is the whole database. No ORM, no off-the-shelf migrations framework,
no hosted anything. The app talks to SQLite through `better-sqlite3` in a server
component; nothing about the data model reaches the client except the rendered
values.

**Four npm dependencies**, total: `next`, `react`, `react-dom`,
`better-sqlite3`. Charts are hand-rolled inline SVG. Adding a charting library
would have been a fifth dependency, a client bundle, and a loss of control over
exactly how a gap in the data renders — which, given "missing is not zero" is a
core rule, was the deciding factor.

---

## Data model

39 tables, 22 views, 43 indexes, 124 numbered migrations.

The tables fall into three groups, and the boundaries between them are enforced
rather than conventional.

**Logged by hand.** Daily subjective entries (appetite, mood, notes), food
entries pointing at a food bank, lifting sets pointing at a movement table,
weigh-ins, medication doses, pen stock, programme sessions.

**Synced from the wearable.** Daily summaries, sleep, activities, and a sync
log. These are **never hand-edited** — the sync owns them, writes are UPSERTs on
a natural key, and the sync log carries one row per endpoint per date so a
partial failure is visible instead of silent.

**Computed, never stored.** Day nutrition totals, computed-vs-stated intake with
a label saying which, sleep with derived percentages, lifting progression with
top set and percentage change, weekly alcohol with the floor rule applied, and
sync gaps per endpoint.

That third group is the important one. **No derived figure is stored.** A daily
protein total is a view over food entries, not a column someone remembered to
update. The cost is query complexity; the benefit is that a figure cannot drift
out of agreement with the rows it came from.

### Foreign keys are a per-connection setting

`PRAGMA foreign_keys` is not stored in a SQLite file. It defaults to **off**, per
connection. A bare `sqlite3` invocation will therefore silently write orphan
rows into a schema full of carefully declared references.

Every path into the database sets it: the app's connection module, and a shell
wrapper that is the only sanctioned way to run ad-hoc SQL. This is the kind of
detail that is invisible until a report is wrong and the cause is a row pointing
at a food that no longer exists.

---

## Sync

Scheduled four times a day. Three invocation modes: a recent window, an explicit
date range for backfill, and a stale-only mode that no-ops unless the newest
data is over an hour old — which is what the dashboard itself calls on load.

**Idempotent by construction.** Every write is an UPSERT on the natural key, so
re-running any range is safe. Backfilling a month and re-running it twice
produces the same database.

Lessons encoded in the sync:

- **One date per API call.** Batched date requests fail *silently* — returning a
  response that looks fine and contains nothing.
- **Historical dates time out.** Retried once, then the gap is recorded as a gap
  rather than being retried indefinitely or quietly dropped.
- **A sleep record's date is the wake-up morning**, not the evening it started.
  Getting this wrong shifts an entire series by one day.
- **Timestamps arrive as epoch milliseconds UTC** and are converted before
  storage. Everything is stored UTC and displayed in local time.
- **Rate limiting is IP-based, not credential-based.** Hitting the login
  endpoint too often returns a 429 that clears on its own. A cached session
  token avoids re-authenticating at all, which is the actual fix.

The sync is Python because the wearable client library is. It deliberately omits
one popular auth dependency, which is deprecated and pulls OpenTelemetry in
behind it — unacceptable in a project whose whole premise is that no health data
leaves the machine.

---

## The view layer as an API

Views are where the domain rules live, so that no caller can bypass them.

The weekly alcohol view applies the floor rule itself and reports whether the
total is a floor. The intake view labels its own source as computed or stated.
The sleep view derives percentages. The sync-gap view surfaces failures per
endpoint.

A page cannot accidentally render a total that ignores a flagged night, because
the view does not expose one.

---

## Scoring and the calendar

Each day scores sleep, intake and activity 0–100. A day where all three cross
their own excellence threshold is marked specially — and that verdict is
computed **once**, in one module, shared by the calendar and by every page that
shows a single day.

That sounds obvious. It was not the first implementation: the same verdict was
originally derived in two places, which meant a day could render as excellent on
one page and ordinary on another. The weights are explicit and documented, so
the answer to "what is the cheapest way to improve today's score" is a real
calculation rather than a guess.

---

## Tests without a framework

132 test files — 119 run by the Node built-in test runner, 13 by the Python
virtualenv that owns the sync. No Jest, no Vitest, no added test dependency.

Two of them are worth describing, because they test things that are usually left
untested:

**A sliced-out sync block against synthetic payloads.** The real sync needs
credentials and is rate-limited, so it cannot be exercised in a test. The zone
calculation is therefore extracted and run against hand-built payloads —
including the null-versus-measured-zero case, which is the actual bug class.

**Every SQL statement embedded in prose.** The project's documentation and
agent instructions contain dozens of example queries. Nothing executes
documentation, so a renamed column silently invalidates them until someone gets
a wrong answer. A test extracts every embedded `SELECT` and executes it
read-only, asserting only that it parses and runs — never what it returns.

That test opens the database read-only but **not** immutable, because immutable
mode ignores the write-ahead log and reports recently created objects as
missing — a false failure that is worse than no test.

Both tests were verified by reintroducing the bug they cover and confirming they
fail. A test that has never failed has not been tested.

---

## Production deployment

Docker on a home server, reached over a private tailnet. No published host
ports; a userspace VPN sidecar container is the only ingress.

The controller fetches, builds, runs the full test suite, takes a verified
backup, replaces the running app, and recovers the previous image if the new one
fails to start. A two-minute timer polls for a new approved commit, so shipping
is a push rather than a session.

**Schema changes are gated by an explicit manifest.** A migration does not run
because it exists; it runs because its hash is listed as approved. The deploy
and the sync share a lock, so neither can run while the other holds the
database.

**The smoke test performs a real authenticated write** against a disposable copy
before a build is promoted. This is deliberate: an earlier iteration verified
deployments with a `GET`, which passed happily through a bug where every write
returned 401 — because a `fetch` POST does not replay HTTP Basic credentials the
way a page navigation does. A read-only check proves nothing about writes.
