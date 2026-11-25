# Vantage

A self-hosted health dashboard. One SQLite file, a Next.js app over the top, and
a sync that pulls wearable data in on a timer.

It replaced a sprawl of chat logs, one-off charts and a folder of daily text
files with a single source of truth for weight, medication, training, sleep and
food — and then kept going, into the part most trackers skip: explaining *why* a
number moved.

**This repository is a showcase.** It holds screenshots and design notes. The
source is private, because the database it runs against is medical.

---

## Screenshots

### Expedition

![Expedition — the climb from start weight to goal, with the next camp, the day's quests and overnight conditions](screenshots/overview.png)

The whole journey as one climb: where you are, the next camp, and the summit.
The projected arrival date is fitted from the actual pace, not the goal date
repeated back -- so it moves when the rate does.

The reading carries its own age. `11D OLD` next to a weigh-in says the number
is stale rather than quietly showing an old figure as if it were today's.

### Weight

![Weight — tape measurements over time, drawn on a body outline](screenshots/weight.png)

Sizes follow the tape, not the scale. The outline draws the current shape
solid and the starting one dashed, with the changes exaggerated 3x so half an
inch is actually visible -- and it says so, rather than letting anyone read
the drawing as life-size.

Each row carries its reading count. `2 readings` is a warning that a trend
line through two points is a line through two points.

### Trail Mix

![Trail Mix — targets as flags to raise, ceilings as a pack to fill](screenshots/food.png)

Two kinds of number, two different objects, so they can never be confused.
Targets are **flags**: raised or not, and the flag's height is the progress.
Ceilings are a **pack**: it fills up, and something bursting out of it is
over.

![The flags and the pack, close up](screenshots/food-targets.png)

The same colour cannot mean "good" in one place and "careful" in another,
because the shapes are doing the work the colour used to. Each column says
its own verdict underneath -- `Reached +17 g`, `19 g short` -- so nothing has
to be decoded.

![Trail Mix — the day's stops, and every item logged](screenshots/food-log.png)

Below that, the day as a route: what was eaten at each stop, then every item
with the macro you pick highlighted across all of them at once.

### Climber

![Climber — which muscles have been worked, and what to train next](screenshots/training.png)

The body map warms where the last 28 days of lifts landed, so the gaps are
visible rather than inferred from a table. `10 / 24` movements trained, 14
never done.

`READY TO GO UP` is deliberately conservative: a load moves only after two
clean sessions at it, because finishing the reps is not the same as owning
the weight.

### Running

![Running — the programme as laps around camp](screenshots/running.png)

A nine-week programme as 27 laps. Done, next, outstanding and ahead are four
different states, and a run that was skipped stays visibly outstanding rather
than quietly vanishing.

The heart-rate band in the header scales with the programme week -- the walk
intervals disappear as the runs go continuous, so a fixed band would flag
every later session as too hot.

<!-- Sleep is the one page still to shoot. Medication is deliberately NOT
     screenshotted and has no block here: every other page can be shown with
     invented figures, but that page's content IS the sensitive part -- dose
     history, injection schedule and supply projection -- so it is described
     in the text and not pictured. -->

All screenshots use a demo profile whose figures are invented.

---

## What it does

| Page | What it shows |
|---|---|
| Expedition | The climb from start weight to goal: where you are, the next camp, the day's prompts, and last night's conditions |
| Trail Mix | Targets as flags and ceilings as a pack, the day's stops, every item logged, barcode scanning, a searchable food bank |
| Weight | Tape measurements over time on a body outline, and the readings behind each one |
| Climber | A body map of what the last 28 days of lifts worked, per-movement loads, and a load ladder that waits for two clean sessions |
| Running | A nine-week programme as laps around camp, with HR zones against a target that scales with programme week |
| Camp | Sleep score, HRV, sleep stress, REM, body battery, and a derived bedtime recommendation |
| Medication | Dose calendar, pharmacokinetic decay curve, supply projection, remaining stock derived rather than stored |
| Mood, Vista, Badges, Guide | Mood scores and tags, the coach's written notes, earned badges, and how the whole thing works |

Multi-profile, PIN-gated, with a demo profile whose figures are invented — which
is what the screenshots above show.

---

## The interesting part

Most of the engineering here is not the UI. It is the set of rules the code
enforces to stop it producing a confident wrong answer. Every one of them exists
because an earlier version got it wrong.

**Missing is not zero.** A gap renders as "no data" in its own colour, never as
`0` or a dash. The sync log records failures per endpoint, so a successful daily
fetch cannot mask a failed sleep fetch on the same date.

**Never infer an input from its own output.** Alcohol intake is not reverse
engineered from sleep data, even though the overnight signature is obvious,
because deriving the input from the output makes the correlation circular. Where
the signature is present and the field is empty, the night is *flagged* and the
value left null — and any weekly total containing a flagged night reports itself
as a floor rather than a total.

**Stated figures never mix with computed ones.** Imported days carry
hand-written totals; itemised days compute from a food table. A view column says
which, and the UI labels the difference, because averaging the two silently
would make the series meaningless.

**An open day understates intake.** Food gets forgotten, never invented. A day
counts as closed only when the user says so, and open days are excluded from
every average.

**Two different questions, two different code paths.** "Is this reading
unusual?" and "what moved it?" live in the same module and are deliberately not
wired together. A move can be entirely explainable while sitting inside normal
variation — so gating the explanation on the anomaly flag would show nothing on
exactly the readings a person actually asks about.

**A statistical band is calibrated on comparable readings only.** Fitting
variance across a regime change mixed two populations — a short initial rapid
phase and a later settled one — and the first swamped the second, flagging an
old baseline reading while staying silent on a genuine outlier. Bounded to the
settled period, the residuals centre on zero. Below a minimum sample there is
*no band at all*, rather than a fallback to the contaminated one.

**Stated physiology is never presented as fitted.** The figures that rank
competing explanations for a weight move are published constants, not
regressions on one person's data. They render with a `~`, the UI says in its own
words that they rank rather than measure, and the note explaining what sample
size would be needed to fit them properly is itself tracked in the database.

**Some causes are deliberately absent.** Three plausible explanations for a
weight change are not offered at all, each for a stated reason: one has no
timestamp to reason from, one cannot be evidenced from the data on file, and one
would need body composition measurements the project does not have. An
explanation that cannot be supported is worse than no explanation.

**Know what the sensor actually measures.** Resistance training does not appear
in wearable data at all, because it never holds heart rate in the relevant
zones — so it is read from the lifting log, and cardio volume from intensity
minutes, never from activity count. Intensity minutes themselves come from the
vendor's activity-and-cadence method, not heart-rate zones, and conflating the
two produced a wrong analysis once already. The least trustworthy column in the
sleep data varies by more than an order of magnitude between nights; it is
displayed for completeness and never scored.

**Never guess a match.** An exact or single-candidate food match logs silently.
Multiple candidates ask. An unknown food is added to the bank first. The one
thing it will not do is pick the likeliest row and continue.

---

## Architecture

Further notes:

- [Architecture](docs/architecture.md) — data model, sync design, the view layer
- [Design decisions](docs/decisions.md) — what was built, rejected, and reversed
- [Security posture](docs/security.md) — threat model for medical data on a work machine

In short: Next.js App Router with server components, `better-sqlite3`, four npm
dependencies, and charts hand-rolled as inline SVG rather than pulling in a
charting library. A Python virtualenv handles the wearable sync. The production
deployment is Docker and a userspace Tailscale sidecar on a home server, with a
two-minute poller, verified backups and a smoke test that performs a real
authenticated write against a disposable copy of the database before promoting
a build.

Roughly 86,000 lines across the app, domain logic, 124 numbered migrations and
132 test files.

---

## Why it exists

Commercial trackers are built to be *encouraging*. That makes them bad at the
one job that matters when you are making decisions from the data: telling you
when the data does not support a conclusion.

This one is built the other way round. It will tell you a week's alcohol total
is a floor rather than a total, that an average excludes four days because they
were never closed, and that it has no opinion on a reading because there are not
enough comparable ones yet. Those answers are less satisfying and considerably
more useful.
