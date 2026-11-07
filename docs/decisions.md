# Design decisions

Including the ones that were wrong. A list of only the good calls would be
marketing.

---

## Reversed

### A fixed target that should have been a function

Two pacing targets for the running programme were originally fixed numbers,
derived from the first two sessions of week one. They were then applied to every
later week.

This is wrong for a structural reason, not a tuning reason. Early sessions are
mostly walking, and walk intervals let heart rate recover between efforts. As
the walk ratio falls, the honest target moves. A band derived from walk-heavy
weeks flags perfectly correct later sessions as too hard.

Both targets are now functions of programme week, interpolating on walk ratio.
The lesson generalises: **a number measured under conditions that are themselves
changing is not a constant.**

### A prescription that treated the symptom

A movement was reported as feeling too heavy at a given weight. The system
prescribed cutting the range of motion.

The actual cause was posture and rest interval — performed seated rather than
standing, with the prescribed rest actually taken, the same weight was
comfortable. The prescription was withdrawn.

What went wrong was the order of inquiry: load was adjusted before the cheaper
variables were checked. The rule now is to check rest, position and session
order *before* concluding anything about load. In two separate cases the user's
own instinct beat the stored prescription, which is itself a finding worth
encoding.

### A measurement taken under the wrong conditions

A string of maximum-effort scores across several sessions read as a ceiling — the
user was at the limit at those loads. Loads had been static for months, which
seemed to confirm it.

Then the rest interval was corrected from roughly 40 seconds to the
evidence-based 2–3 minutes, and the same movement at the *same load* scored two
reps in reserve. One variable had changed.

Every one of those earlier scores was a valid measurement of a different thing:
performance under inadequate rest. The months of static loads were not a
plateau, they were an artefact. **Historic data collected under a since-corrected
protocol is not a baseline.** It has to be re-measured, not reinterpreted.

### An inference that should have been a question

The sync originally matched activities to planned programme sessions by date
proximity and marked them complete. On five separate occasions it closed a
session using a workout that had been done *instead of* the planned one.

Automatic matching was removed. Candidates are counted, the UI asks, and the
user's answer is what creates the link — with an explicit exclusion flag and a
reason when the answer is no.

The valuable part is what the unanswered state now means: an unlinked candidate
is a session still outstanding, which is **correct**. The earlier design
optimised for having no open questions, and bought that by recording things that
had not happened.

### Matching on an identifier instead of a logged answer

A supplement was modelled as a constant, inferred from the presence of a
particular item in the food log by its row id. Two separate failures followed: a
prediction that assumed the supplement during a period when it had not been
taken, and then — after a second qualifying product was introduced — a matcher
that went blind to it entirely because it keyed on the first product's id.

It is now an explicitly logged answer per occurrence, with three states: yes, a
recorded no, and never asked. **Null is not a no**, and every row from before the
question existed is null — so that period supports no conclusion in either
direction.

A later change let an unmentioned occurrence default to yes, at the user's
explicit request and after being told the cost. The cost is recorded alongside
the change: a yes no longer proves the answer was given, so any claim in the
other direction now needs explicit no rows, never silence.

---

## Rejected

### Deriving alcohol intake from sleep data

The overnight signature is unmistakable and the temptation is obvious. It was
rejected because the primary question the data exists to answer is how alcohol
affects sleep — and deriving the input from the output makes that correlation
circular and guarantees a positive result.

Flagging is the compromise: the night is marked, the value stays null, and
totals report themselves as floors.

### Predicting the next occurrence of an irregular event

Asked for, and declined. Two observed intervals do not support a prediction,
and the conventional default figure would have been materially wrong.
The module reports elapsed time and the real observed gaps with an explicit
confidence field, so a single interval can never be quoted as a norm.

The general principle: **two data points are not a prior.** The same reasoning
blocks predicting illness duration.

### Inferring an event from co-occurring symptoms

Two concurrent symptoms look like they could be automatically grouped into one
illness. They are not, because a headache during an unrelated flare-up is two
facts, and inferring an event would put every coincidence into the duration
statistics.

The event is created when the user confirms it. Once open, related symptoms join
automatically and the start date moves earlier if an earlier onset appears. The
end is derived from the last member resolving, never typed.

Which symptom types may join is explicit, and some are excluded on purpose
because they track medication rather than illness.

### Lowering a threshold until it fired

An anomaly detector did not flag the reading the user had actually asked about.
Lowering the sensitivity constant until it did would have tuned a threshold to
the single sample that happened to exist.

Instead, the two questions were separated — "is this unusual?" and "what moved
it?" — and only the second drives the UI. The threshold was left alone.

### A charting library

Five dependencies instead of four, a client bundle, and someone else's opinion
about how a gap in a series renders. Given that "missing is not zero" is a core
rule, that last point decided it. Charts are inline SVG.

Every chart carries a real hover readout, because a native SVG `<title>`
tooltip — which is what you get for free — is not one.

---

## Kept deliberately

### Imperfect historical data, left imperfect

A convention error meant a stretch of food entries used cooked rather than raw
weights for meat, overstating protein by roughly a quarter.

The entries were left alone and the convention was fixed going forward, with the
reason recorded in a migration. Rewriting the log would have replaced measured
data with retroactively computed data and made the series less trustworthy, not
more. The fix is the convention.

### Stated physiological constants, never fitted

The figures that rank competing explanations for a weight change are published
values, not regressions on one person's data. They render with a `~` and the UI
states in its own words that they rank rather than measure.

Fitting them properly would need several times the available readings *and*
enough days where the variables diverge rather than spike together. That
requirement is written down with a review date rather than quietly ignored.

### Three absent explanations

Dose timing carries no weight attribution, because the timestamp column is null
on every row — there is no hour to reason from. A second is never inferred at
all, because the event log it would come from records occurrences, not phases.
Muscle gain is never offered as an explanation, because with the available body
composition data it cannot be evidenced.

An explanation that cannot be supported is worse than no explanation, because it
will be believed.

### A write-only feature with no UI

One data-entry path has no interface and will not be given one. It was
considered, decided against, and the decision recorded — so it does not get
re-proposed every few months.

---

## Security decisions

### HTTP Basic is base64, not encryption

For LAN access the honest description is that it stops a housemate, a guest or a
stray device on the Wi-Fi, and does not stop anyone who can read network
traffic. That is an accepted trade for a home network, stated plainly rather
than oversold. A private tailnet is the actual upgrade, and is what production
uses.

### Fail closed, and keep the fail-closed direction

In LAN mode an unset password returns 503 to everyone rather than serving
unauthenticated. The exemption for the loopback bind matches only a literal
loopback address — unset, empty, misspelled and `localhost` all keep the
password.

The check is never keyed off a request header, because headers are
attacker-controlled.

### The bind address and the auth gate are one unit

The network bind and the variable that tells the proxy whether to demand a
password are derived from **the same constant**, in the same file, so they cannot
drift apart. Changing one without the other is precisely the bug that would
expose everything, so the code makes it unrepresentable.

### A session cookie, deliberately not `secure`

Basic auth alone broke every write: reads worked, writes returned 401, and it
surfaced as a dead button. A `fetch` POST does not replay saved Basic
credentials the way a navigation does.

Basic now mints an HMAC-signed session cookie, which `fetch` does send. It is
intentionally not flagged `secure`, because that flag means HTTPS-only and would
stop the cookie being sent over plain HTTP on the LAN — reinstating the original
bug. The token is signed with a key derived from the password, so rotating the
password invalidates every session.

### Stated limits

Some things are not fixable in code and are written down rather than papered
over: corporate device management sees process names and file paths regardless
of permissions, the root user ignores permission bits, and the OS indexes
filenames unless the data directory is explicitly excluded.

---

## Working method

Three rules that shaped the codebase more than any architectural choice.

**A fix is not done until the changed path has been run.** Reading the diff
back, a syntax check, or a page returning 200 are not verification. Where a live
run is genuinely impossible, the changed block is extracted and tested against a
synthetic input — always covering null against a measured zero.

**Real data can hide the bug.** A test written against the real series passed
with the bug deliberately reinstated, because the real data never exercised the
broken branch. Fixtures are built to hit the branch, not to look realistic.

**Write the commit message for someone reading it in six months.** The history
is the design record. "This machine was never the exercise we thought it was"
beats "fix component" — and when, ten weeks in, a whole category of training
turned out to have been logged against the wrong movement, that history is what
made the correction a twenty-minute job rather than an archaeology project.
