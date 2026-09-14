# Pace — Design Decisions

Status: accepted for v1
Date: 2026-09-12

A personal productivity app: ordinary tasks plus *scalable activities* — recurring
things with a daily minimum target, a unit, and a log of how much was actually done.
Single user (me), two clients (laptop browser, phone browser).

This document records what was decided and, more importantly, **what was rejected and
why**. Revisit it when a decision starts to hurt; don't quietly drift from it.

---

## 1. The two questions that determined everything

**Q: How often do I need to write data while my laptop is unreachable?**
A: Often. The laptop is frequently closed, and having to open it is enough friction
that I would skip logging. → The phone must work on its own.

**Q: Is learning a frontend framework a goal?**
A: No. Go is the goal — I want to build a real application in Go. Frontend can come
in a later version if it is ever needed.

Everything below follows from those two answers.

---

## 2. Architecture

**One Go server with one SQLite database, hosted on a small always-on VM. Both the
laptop and the phone are thin clients rendering server-generated HTML.**

```
   Laptop browser  ─┐
                    ├─ HTTPS ─→  Go binary (net/http + html/template)
   Phone browser   ─┘                     │
                                          └─→ pace.db  (SQLite, WAL)
```

### Why this shape

There is exactly one copy of the data, so there is **no synchronization subsystem at
all** — no second database, no outbox, no change log, no tombstones, no conflict
policy, no merge tests. That is an entire category of work and an entire category of
bugs deleted, and for a single-user app it buys nothing to keep it.

It also puts ~100% of the code in Go, which is the point of the project.

### Rejected: local-first PWA with IndexedDB on each device + a sync server

The original plan (React + TypeScript + IndexedDB + Dexie + service worker + a Go
sync endpoint). Rejected because:

- It puts ~90% of the work in the stack I am explicitly *not* trying to learn, and
  reduces the Go component to a few hundred lines of JSON exchange.
- Two databases means merge logic, tombstones, and ordering tests — the hardest part
  of the whole project — in service of a requirement (device-local data) that is not
  actually a priority for me.
- IndexedDB in a browser profile is a mediocre source of truth for data I would be
  sad to lose (storage eviction, cleared site data, profile loss).

### Rejected: Go sync server running on the laptop, phone syncing over the LAN

Rejected because it keeps every bit of the sync complexity *and* gives up
reachability — which is precisely the constraint that matters most to me. It is the
worst of both options. It also has a blocker that is easy to miss: service workers
and PWA install require a secure context, and `http://192.168.x.x` is not one (only
`localhost` is exempt), so "install the PWA from the laptop's IP and use it offline"
does not work as described without setting up TLS anyway.

### Rejected: peer-to-peer sync between phone and laptop

Interesting distributed-systems project, wrong first project. Deferred indefinitely.

### Accepted cost of this decision

**No internet means no app.** This is a real regression from the "fully offline"
answer I gave earlier, and it is accepted deliberately: a reachable server solves the
laptop-is-closed problem, which is the friction I actually hit, whereas phone-with-no-
signal is rare for me.

Mitigation path if it does start to hurt, in order of effort:

1. A web app manifest + a tiny service worker that caches the Today page for
   read-only viewing while offline.
2. An offline write queue for progress entries only. **This is cheap by construction:**
   progress entries are append-only (§4), so queuing and replaying them needs no
   conflict resolution whatsoever — just retry until the POST succeeds, keyed by a
   client-generated id for idempotency. Task edits would stay online-only.

Do not build either until the lack is actually annoying in daily use.

---

## 3. Stack

| Layer | Choice | Notes |
|---|---|---|
| Language | Go | The point of the project. |
| HTTP | stdlib `net/http` | Go 1.22+ routing patterns are enough. No framework. |
| Templates | stdlib `html/template` | Server-rendered. No build step, no bundler. |
| Database | SQLite via `modernc.org/sqlite` | Pure Go — no cgo, so cross-compilation stays trivial. WAL mode; see §4 for the required connection pragmas. |
| DB access | `database/sql` + hand-written SQL | Learning value. Consider `sqlc` later if the SQL gets repetitive. |
| JS | None in v1 | Plain HTML forms. A full page load per interaction is fast enough. |
| CSS | One hand-written mobile-first stylesheet | No framework. |
| Timezones | `import _ "time/tzdata"` | Embeds the IANA database in the binary. Required — see below. |
| Access | Tailscale (see §6) | Reachable from anywhere; no public exposure; no auth code. |
| Packaging | Docker, multi-stage → `FROM scratch` | Makes the planned host migration a near-no-op. |
| Deploy | `docker compose up -d` on the VM | Image from GHCR. `restart: unless-stopped`. |

### Decided: use Docker

Not because a static Go binary needs it — it does not — but because **the host will
change in about three months** (§6). Containerising turns that migration into "install
Docker, copy one file, `docker compose up -d`" instead of re-deriving a systemd unit,
a service user, file permissions and a Go toolchain on a new distro. I already know
Docker, so the cost is zero.

Constraints that keep it from becoming a liability:

- **Multi-stage build ending in `FROM scratch`.** `modernc.org/sqlite` is pure Go, so
  there is no cgo and no libc to link against. The final image is one static binary
  plus embedded templates and CSS — a few MB, nothing to patch, no base-image CVEs.
- **`import _ "time/tzdata"` is mandatory.** A `scratch` image has no
  `/usr/share/zoneinfo`, so `time.LoadLocation` would fail at runtime — and the whole
  day-boundary design (§4) depends on resolving an IANA timezone. Embedding costs
  ~450 KB and removes the failure mode entirely. Test it in the built image, not just
  on the laptop.
- **The database lives on a bind mount**, `/srv/pace/data:/data`, never inside the
  container's writable layer. This is the one genuine footgun: an image rebuild or a
  stray `docker compose down -v` against a misconfigured volume destroys the data.
  Treat the host path as the real artefact and the container as disposable.
- **Bind to `127.0.0.1:8080` on the host**, i.e. `ports: ["127.0.0.1:8080:8080"]`.
  Docker otherwise punches its own rules into iptables and can expose a published port
  to the internet regardless of the cloud firewall. Tailscale reaches it from there.
- **Tailscale runs on the host, not in a container.** Sidecar setups exist; they add
  moving parts for no benefit here.

Image distribution: GHCR (free for personal use, and real practice). Build and push
from the laptop, pull on the VM. If I would rather not depend on a registry at all,
`docker save pace:latest | gzip | ssh ec2-user@pace 'gunzip | sudo docker load'` works
and needs no account.

**Rejected: ECR.** It would tie image storage to the provider I am explicitly planning
to leave. Same reasoning as the section below.

**Rejected: ECS / Fargate / any managed orchestration.** One container on one box.
Fargate is also not free, which defeats the point.

### Rejected: PostgreSQL

For one user and one process, SQLite is strictly simpler and fast enough by orders of
magnitude. The one honest argument for Postgres is career-relevant practice, and it is
not worth the operational burden here. SQL is kept in a thin data layer so porting
later is bounded work, not a rewrite.

### Rejected: HTMX / any JS in v1

The interaction that matters most is "tap +25m." A form POST with a redirect does that
in one round trip on a local-ish connection. Add HTMX only if a page reload becomes
visibly annoying.

### Rejected: AWS-specific services (Lambda, DynamoDB, Cognito, API Gateway)

The app must stay portable — a single binary plus a SQLite file that runs anywhere.
Do not design around any one provider's managed services.

---

## 4. Data model

Design rules that are not negotiable:

1. **Progress entries are append-only.** The day's total is always
   `SUM(amount)`, never a mutable counter. This is what makes history trustworthy and
   what makes a future offline queue conflict-free. **Mistakes are corrected by
   appending a reversing row, never by editing or deleting one** — see "Undo without
   mutating history" below.
2. **Amounts are integers in the activity's base unit** (minutes, pages, reps,
   metres). No floats — "did I hit the minimum" should never be a floating-point
   comparison. Display formatting is a presentation concern.
3. **Each day's target is snapshotted.** Changing an activity's minimum from 30 to 60
   next month must not retroactively rewrite whether past days succeeded.
4. **`day` is separate from `recorded_at`.** I will forget to log and backfill
   yesterday at 8am. Timestamps are stored UTC (RFC 3339); `day` is a local calendar
   date resolved through the settings timezone and day boundary.

```sql
-- Ordinary tasks. Deliberately minimal; see §5.
-- User-defined life areas: Health, Religion, Study, ... The organising axis for
-- both tasks and activities, and what keeps the Today screen from becoming a wall.
CREATE TABLE areas (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL,
    color       TEXT,                      -- for section headers; presentation only
    sort_order  INTEGER NOT NULL DEFAULT 0,
    archived_at TEXT,
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL
);

CREATE TABLE tasks (
    id            TEXT PRIMARY KEY,        -- UUID, generated server-side
    title         TEXT NOT NULL,
    area_id       TEXT REFERENCES areas(id),
    scheduled_for TEXT,                    -- local 'YYYY-MM-DD'; drives the Today screen
    done_at       TEXT,                    -- UTC RFC3339; NULL = not done
    duration_min  INTEGER CHECK (duration_min IS NULL OR duration_min > 0),
                                           -- optional; how long it actually took
    created_at    TEXT NOT NULL,
    updated_at    TEXT NOT NULL
);

-- A reusable activity definition.
CREATE TABLE activities (
    id             TEXT PRIMARY KEY,
    name           TEXT NOT NULL,
    unit           TEXT NOT NULL,          -- 'minutes' | 'pages' | 'reps' | 'metres' | ...
    unit_kind      TEXT NOT NULL CHECK (unit_kind IN ('time','quantity')),
                                           -- 'time' => the base unit is ALWAYS minutes
    minimum_target INTEGER NOT NULL CHECK (minimum_target > 0),  -- in base units
    schedule       TEXT NOT NULL,          -- weekday bitmask or 'daily'
    area_id        TEXT REFERENCES areas(id),
    display_style  TEXT NOT NULL DEFAULT 'card'
                   CHECK (display_style IN ('card','compact')),  -- see §5
    quick_amounts  TEXT,                   -- OPTIONAL '25,10' → extra +25 / +10 buttons
                                           -- alongside the free-entry field. Often NULL.
    sort_order     INTEGER NOT NULL DEFAULT 0,
    archived_at    TEXT,
    created_at     TEXT NOT NULL,
    updated_at     TEXT NOT NULL
);

-- One row per (activity, day) it was scheduled for. Materialised at the day
-- boundary; NEVER updated once written. This is the target snapshot.
CREATE TABLE activity_days (
    id              TEXT PRIMARY KEY,
    activity_id     TEXT NOT NULL REFERENCES activities(id),
    day             TEXT NOT NULL,         -- local 'YYYY-MM-DD'
    target_snapshot INTEGER NOT NULL CHECK (target_snapshot > 0),
    created_at      TEXT NOT NULL,
    UNIQUE (activity_id, day)
);

-- Append-only log of work actually done. Corrections are new rows; nothing here is
-- ever UPDATEd or DELETEd.
CREATE TABLE progress_entries (
    id                TEXT PRIMARY KEY,    -- client-suppliable, for idempotent retries
    activity_id       TEXT NOT NULL REFERENCES activities(id),
    day               TEXT NOT NULL,       -- local 'YYYY-MM-DD'; may be backdated
    amount            INTEGER NOT NULL CHECK (amount <> 0),
                                           -- base units; negative ONLY on a reversal
    duration_min      INTEGER CHECK (duration_min IS NULL OR duration_min > 0),
                                           -- wall-clock minutes, written by the timer
                                           -- stop flow. Equals amount when
                                           -- unit_kind='time'. NULL if hand-typed.
    note              TEXT,
    reverses_entry_id TEXT UNIQUE REFERENCES progress_entries(id),
                                           -- set only on a correction row. UNIQUE
                                           -- permits many NULLs, so this reads as
                                           -- "an entry may be reversed at most once"
    recorded_at       TEXT NOT NULL        -- UTC RFC3339, when it was entered
);

CREATE INDEX idx_progress_activity_day ON progress_entries (activity_id, day);
CREATE INDEX idx_progress_day          ON progress_entries (day);

-- Single-row settings table.
CREATE TABLE settings (
    id                INTEGER PRIMARY KEY CHECK (id = 1),
    timezone          TEXT NOT NULL,       -- IANA name
    day_starts_at     TEXT NOT NULL DEFAULT '04:00',
    week_starts_on    INTEGER NOT NULL DEFAULT 6   -- 0=Sun .. 6=Sat
                      CHECK (week_starts_on BETWEEN 0 AND 6),
    hijri_offset_days INTEGER NOT NULL DEFAULT 0   -- -1 / 0 / +1, display only
                      CHECK (hijri_offset_days BETWEEN -2 AND 2)
);

-- At most one timer runs at a time; the row exists only while it is running.
-- Elapsed time is derived from started_at on stop, so closing the browser,
-- switching devices or a phone killing the tab cannot lose it.
CREATE TABLE active_timer (
    id          INTEGER PRIMARY KEY CHECK (id = 1),
    task_id     TEXT REFERENCES tasks(id),
    activity_id TEXT REFERENCES activities(id),
    started_at  TEXT NOT NULL,             -- UTC RFC3339
    CHECK ((task_id IS NULL) <> (activity_id IS NULL))
);
```

### SQLite runtime settings

The schema above is only half the story. **SQLite defaults `foreign_keys` to OFF, per
connection** — so without this, every `REFERENCES` clause above is decorative and a
typo'd `activity_id` inserts happily. `database/sql` pools connections, so the settings
have to be attached to the DSN, where the driver applies them to every connection it
opens, not run once at startup.

```
file:/data/pace.db?_pragma=journal_mode(WAL)&_pragma=foreign_keys(ON)&_pragma=busy_timeout(5000)&_pragma=synchronous(NORMAL)
```

- **`_pragma=name(value)` is `modernc.org/sqlite`'s syntax.** The mattn-style
  `?_foreign_keys=on&_journal_mode=WAL` spelling that most blog posts use is **silently
  ignored** by this driver — no error, foreign keys just stay off. Worth a test that
  asserts `PRAGMA foreign_keys` reads back as 1.
- `busy_timeout` makes a writer wait for a lock instead of failing instantly.
- `synchronous=NORMAL` is the correct pairing with WAL: safe against process crashes,
  and only at risk from an OS-level crash mid-write.
- **`db.SetMaxOpenConns(1)`.** With one user there is no throughput to lose, and it
  removes `SQLITE_BUSY` as a category rather than handling it. Revisit only if a page
  ever feels slow because of it.

### Habits are not a new entity

A habit is already a scalable activity. "Five prayers in the mosque" is an activity
with `unit='prayers'` and `minimum_target=5`; zikr is an activity with a repetition
count; "one Boot.dev lesson a day" is an activity with `minimum_target=1`. Adding a
third top-level entity alongside Task and Activity would duplicate the CRUD, the
rollups, the history and the Today rendering, and would leave a permanent "which
bucket does this go in?" question at the moment of capture — which is exactly when
friction is most expensive.

**The real problem is clutter, and clutter is a presentation problem.** It is solved
by two cheap things:

- `areas` — the Today screen groups by area, and a finished section collapses to a
  single line (`Religion ✓ 9/9`).
- `activities.display_style` — `'compact'` renders a one-line checklist row instead of
  a full card. Small targets and routine items use it. It is a display hint and
  nothing else; no logic branches on it.

Whether a prayer set is one activity with a target of 5 or five separate activities
each with a target of 1 is **runtime configuration, not schema** — both work today,
and the second gives per-prayer history at the cost of five rows. Decide it when
creating the activities, and change it freely afterwards.

**Because this app now tracks religious obligations, the no-retroactive-debt rule in
§5 stops being a nicety.** A tracker that accumulates visible guilt about missed
prayers is worse than no tracker: the failure mode is avoidance, and the thing avoided
is not just the app. Show today, show history plainly, and never show a deficit.

### How `activity_days` gets filled

One function, `MaterialiseDay(day)`: insert a row for every non-archived activity
scheduled on that day, snapshotting the current `minimum_target`, skipping days before
the activity was created, and **never overwriting an existing row**. It has exactly
three call sites:

1. **A ticker at each local day boundary.** The normal path.
2. **Server startup**, catching up every day between the last materialised day and
   today. Covers restarts, deploys and downtime.
3. **Immediately before any target or schedule change.** A belt-and-braces guarantee
   that today's row exists before the thing that would change it.

**Why not fill lazily when a day is rendered.** The earlier design created rows on
view, which had two failure modes. The mild one: look at a day a week later and it gets
today's target. The serious one is about *schedules* — if an activity was scheduled on
Sundays and I later drop Sunday, then look back at a past Sunday, no row is created and
the system has silently forgotten that Sunday was ever expected. History gains a hole
rather than a wrong number, and progress entries can end up on a day with no
corresponding `activity_days` row at all.

**This is now correct, not merely better.** A target can only change while the server
is running, and while the server is running the boundary ticker fires. So the startup
catch-up after downtime writes exactly the target those days would have been given at
the time — there is no window in which a change goes unrecorded. The only way to defeat
it is to edit the database by hand while the process is stopped.

Cost: roughly `activities × 365` rows a year. At any plausible number of activities
that is a few thousand rows, which SQLite does not notice.

### Undo without mutating history

Append-only is the right default, but an append-only log with no correction mechanism
is not usable software. Tapping `+25` twice, or typing `250` for `25`, will happen —
and the faster capture gets, the more often it will. A capture flow you are afraid of
is a slow capture flow, which defeats the whole point.

**Undo is an insert, not a delete.** A correction row carries `reverses_entry_id`
pointing at the original and an `amount` that is exactly its negation. Both
non-negotiables survive untouched: nothing is ever mutated, and the day total stays a
plain `SUM(amount)` with no filtering.

Rules, all enforced server-side and tested:

- The reversal is **generated by the server**, never supplied by the client: it copies
  the original's `activity_id` and `day` and negates its `amount`.
- An entry can be reversed at most once — enforced by `UNIQUE (reverses_entry_id)`.
- **A reversal cannot itself be reversed.** Otherwise "undo the undo" becomes a way to
  walk a total anywhere.
- History renders a reversed pair as one struck-through line, not as two rows of noise.

### Timers, duration and amount

The naive design — "stopping a timer writes the elapsed minutes as a progress entry" —
contradicts the rule that `amount` is in the activity's base unit. A timer on
*Read book* (`unit='pages'`) cannot produce a valid entry that way.

The fix is not to ban timers on those activities. It is to notice that **elapsed time
and amount are two different facts**, and to store both:

- `progress_entries.duration_min` — wall-clock minutes, written by the timer.
- `progress_entries.amount` — the thing being counted, in the activity's base unit.

For `unit_kind='time'` they hold the same number; for everything else they do not.
That is why **time activities always store minutes as their base unit** — `unit` may
display as hours, but the stored integer is minutes, which keeps the comparison against
`minimum_target` exact.

The payoff is that "how much time did I spend on Health this month" becomes one query
over `progress_entries.duration_min` plus `tasks.duration_min`, across every activity
regardless of what it counts.

**Stopping a timer always opens a confirmation form**, with the elapsed minutes
pre-filled and editable:

| Timer on | The form asks for | Written to |
|---|---|---|
| Activity, `unit_kind='time'` | duration (= amount) | one `progress_entry` |
| Activity, `unit_kind='quantity'` | duration, pre-filled; **and** amount | one `progress_entry` |
| Task | duration | `tasks.duration_min` |

The extra tap is deliberate. Timers get forgotten — left running overnight, a timer
would otherwise append 840 minutes to an append-only log, and the only remedy would be
a reversal. The edit step is what keeps garbage out in the first place. It is also not
the hot path: quick capture means typing an amount on the Today screen, which is
untouched by this.

Two more rules: stopping a task's timer records the duration but does **not** mark the
task done — those are separate acts. And a timer running across a day boundary books to
the day it **started**, since that is when the work happened; the ordinary backdating
control can move it.

---

## 5. v1 scope

### In

1. **Areas** — user-defined, CRUD, assignable to tasks and activities. The organising
   axis for everything else, which is why it is in v1 rather than added later.
2. **Tasks** — title, area, optional scheduled date, done/not done, optional duration.
   Create, edit, complete, delete. Scheduling for a future date is core, not an extra.
3. **Activities** — name, area, unit, minimum target, weekly schedule, display style.
4. **Progress logging** — **free numeric entry is the primary input**; optional
   per-activity shortcut buttons where the amount really is repetitive. Optional note,
   and backdating to a previous day.
5. **Timer** — start on any task or activity. Stopping opens a short form with the
   elapsed minutes pre-filled and editable, then writes a progress entry (or a task
   duration). Server-side start time, no live ticking display. Full rules in §4,
   "Timers, duration and amount".
6. **Today screen** — grouped by area: tasks scheduled today, each scheduled activity
   with `done / target`, and any running timer. Gregorian and Hijri date in the header.
   This is the home screen and where I will spend ~all my time.
7. **History** — per-activity view of the last N days: target, actual, entries.
8. **Export** — `GET /export` returns the whole database as JSON. This is portability
   and human-readable review, **not the backup**. The snapshot described in §6 is what
   you actually restore from.
9. **CSRF rejection.** Middleware that refuses any state-changing request whose
   `Origin` is not this app's own. See below.

### Explicitly out

Priorities, task descriptions, subtasks, separate due-vs-scheduled dates, recurring
*tasks* (activities cover recurrence), weekly views and weekly targets, reminders and
notifications, countdown/Pomodoro timers, a live ticking timer display, streaks and
gamification, analytics, calendar views, drag-and-drop, multi-user, collaboration, any
offline support, any sync, any JS framework.

Two deferred things have had their *storage* added early because the column is free
and the migration later is not: `settings.week_starts_on` (no week view yet) and
`tasks.duration_min` (no time-per-area reporting yet).

**Rule:** add a field back the first time I actually miss it, not in anticipation.
Every field I add is a field I have to fill in at 11pm.

**Note on the above rule vs. the v1 growth.** Areas, the timer and Hijri dates were
added after the first draft of this document, which is a real expansion. It is
allowed because they are not speculative — they come from how the owner already knows
he will use the app — and because two of them are *structural*: a taxonomy and a date
semantic are painful to retrofit across every screen and query, whereas a scalar field
is cheap to add whenever. That is the test to apply to the next request too.

### Why CSRF protection, when there is no login

Tailscale means nothing on the internet can *open a connection* to the app. It does not
mean nothing on the internet can persuade **my own browser** to open one. Any site I
visit on a device with Tailscale running can auto-submit a form at
`https://pace.<tailnet>.ts.net/progress` and the request will arrive looking entirely
legitimate.

There is no session cookie to steal here, which is exactly why this matters: the only
ambient authority the app recognises is *being on the tailnet*, and the browser carries
that authority into every tab. A cross-site POST inherits it for free.

The fix is about ten lines and needs no tokens, no sessions and no state: **reject any
`POST`/`PUT`/`DELETE` whose `Origin` header does not match this app's own origin**, with
`Sec-Fetch-Site: same-origin` as a secondary signal. The expected origin comes from
config so that `localhost:8080` still works in development.

This goes in the skeleton (step 1), not deferred alongside authentication. It is not
authentication, it costs nothing, and retrofitting it across every handler later is
strictly more work than writing one middleware now.

### Two product decisions worth making now

- **Quick capture is the feature.** If logging 25 minutes takes more than ~5 seconds
  end to end, I will stop using this app and the project is wasted. The Today screen
  and the quick-amount buttons exist to serve that number. Add an Android home-screen
  shortcut (a web app manifest gives a standalone icon without needing a service
  worker).
- **No retroactive debt, ever.** A missed minimum does not accumulate into anything.
  The failure mode for trackers like this is becoming a guilt machine I avoid opening,
  and I would rather lose the motivational pressure than the habit of logging.

### Hijri dates: display only, never stored

The header shows the Hijri date alongside the Gregorian one. Two rules make this cheap
instead of contagious:

- **Nothing is ever stored or keyed by a Hijri date.** `day` stays a Gregorian
  `YYYY-MM-DD` everywhere. Hijri is a render-time formatting concern, full stop. The
  moment a Hijri string becomes a primary key or a query filter, every ambiguity below
  infects the data model permanently.
- **Pick one calculated base**, via a small pure-Go library (no cgo — the image is
  `FROM scratch`), pinned to an exact version and verified against known date pairs in
  a test.

### Target: the Egyptian calendar

**The dates should match Egypt, not Saudi Arabia.** This is a real distinction, not a
detail: Umm al-Qura is the Saudi civil calendar and is what most libraries implement
by default, while Egypt's Hijri dates come from Dar al-Ifta using its own criterion.
The two commonly differ by a day, and the difference is not constant — it varies month
to month, since each is deciding a month *start*.

No library can be relied on to produce the Egyptian calendar automatically, and there
is no stable machine-readable feed of the announcements. So the design is deliberately
modest:

1. A calculated base (Umm al-Qura or a tabular calendar — whichever has a maintained,
   pure-Go implementation; pin it and test it).
2. `settings.hijri_offset_days` (-1 / 0 / +1) as the alignment control, reachable in
   one tap from the header rather than buried in Settings, because it will occasionally
   need nudging.

**Known limitation, accepted:** a single global offset is a blunt instrument for a
discrepancy that shifts month to month. If it turns out to drift often enough to
annoy, the upgrade is a small `hijri_month_overrides` table recording the announced
Gregorian start date of a given Hijri month, falling back to the calculated base for
months with no override. That is the correct model — a month start is announced once
and applies to the whole month — but it is not worth building until the simple offset
has actually proven insufficient.

This stays purely cosmetic either way: because nothing is stored or keyed by a Hijri
date, being a day off is a wrong label on a screen, never wrong data.

---

## 6. Access, hosting, TLS

### Access: Tailscale first

Tailscale is a mesh VPN built on WireGuard. Every device I enrol joins one private
network (a *tailnet*), gets a stable address and a DNS name, and can reach the others
directly from anywhere — home wifi, cellular, a café — without any inbound port being
open to the internet. Step-by-step setup lives in
[`docs/setup-server.md`](setup-server.md).

What it buys this project, concretely:

- **Reachability.** The phone can reach the EC2 instance over cellular, which is the
  entire reason the architecture changed in §1.
- **No authentication code in v1.** Nothing outside my tailnet can open a TCP
  connection to the app at all, so there is no login to write, no session handling, no
  password reset, nothing to get wrong.
- **Real HTTPS, no domain needed.** `tailscale serve` terminates TLS with a valid
  certificate for the machine's `*.ts.net` name. That gives a secure context, which
  add-to-home-screen wants, and avoids browser warnings.
- **The EC2 security group can allow zero inbound.** Tailscale only needs outbound.

**Understood limitation:** this is network-level access control, not application-level
authentication. Anyone holding my unlocked phone is inside the tailnet and therefore
inside the app. For a personal task tracker that is the correct trade; it would not be
for anything sensitive, and it is the reason app-level auth stays on the roadmap.

Other trade-offs accepted: Tailscale must be running on the phone, and it is a
dependency on a third party's coordination service (the data path is peer-to-peer, but
enrolling a new device needs their control plane to be up).

**Two footguns, recorded so I do not rediscover them:** node keys expire by default
(commonly 180 days) and an expired server silently drops off the tailnet — disable key
expiry on the server node. And `tailscale funnel` is the *opposite* of
`tailscale serve`: it publishes to the whole internet. Never run it here.

**Later, deliberately, as a learning milestone:** single-password auth (Argon2id hash
in an env var, signed HttpOnly session cookie) plus Caddy for automatic Let's Encrypt,
if I want the app publicly reachable or want the exercise. Not urgent, and not to be
rushed — this is the one part of the app where a mistake exposes my data.

### Hosting: AWS now, Oracle Cloud later — deliberately

The requirement is tiny: one always-on VM, 512 MB RAM, one container, one SQLite file.

**Phase 1 — AWS EC2 (up to six months).** The current Free account plan gives $100 of
credits on signup, up to $100 more for completing activities, and **ends after six
months or when the credits run out, whichever comes first**. Do not plan against a
remembered number — the Billing console shows the real balance and date.

Starting here on purpose: hands-on EC2, security groups, EBS and IAM is worth something
independent of this app, and a known expiry date forces the migration to happen instead
of becoming a someday task.

**The expiry is harsher than it sounds, and this is the part to internalise.** When a
Free account plan ends, *the account closes automatically* and access to resources and
data goes with it. AWS holds the content for 90 days before deleting it permanently, and
the only way to get it back is to upgrade to a paid plan within that window. So the
deadline is not "start paying" — it is "the instance and its EBS volume become
unreachable." Two consequences, both already required for other reasons: the database
must exist somewhere other than that box, and the calendar reminder from
[`setup-aws.md`](setup-aws.md) §1 is load-bearing rather than tidy.

Verified 2026-09-14 against AWS's billing documentation; re-check rather than trusting
this paragraph in six months.

**Phase 2 — Oracle Cloud Always Free.** The Ampere A1 (ARM) allowance is 1,500 OCPU
hours and 9,000 GB hours per month — **2 OCPUs and 12 GB of memory** run continuously,
split across one instance or two. Smaller than the figure that circulates in older
write-ups, and still many times more than this app will use. It does not expire.

Two things to plan around:

- **A1 capacity is frequently unavailable** in popular regions. Pick the region on what
  you can actually launch, not on latency.
- **Idle-resource reclamation is real and still current.** Oracle deems a compute
  instance idle if, over a 7-day window, 95th-percentile CPU is under 20%, network is
  under 20%, *and* — for A1 shapes specifically — memory is under 20%. A personal
  to-do app is the textbook case. Reclaimed means stopped and, for Always Free
  resources, recoverable only up to a point. The usual fix is upgrading the tenancy to
  Pay As You Go, which keeps the Always Free resources free while exempting them from
  reclamation.

  Verified 2026-09-14 against Oracle's technical documentation. A second opinion during
  review claimed this policy had been withdrawn; it has not — the marketing FAQ reads
  more softly than the docs. **Do not let the only copy of the data live somewhere that
  can be reclaimed**, whichever way the policy goes.

**Make the migration boring in advance:**

- Target **arm64 on both**. Oracle's A1 is ARM; on AWS, pick a `t4g` (Graviton)
  instance so the architecture matches. Then the image is byte-identical across the
  move. If the free tier forces x86 instead, it is a one-word change
  (`GOARCH=amd64` → `arm64`) — which is exactly why nothing provider-specific is
  allowed into the design.
- Keep the whole deployment as: a `compose.yaml`, a bind-mounted `pace.db`, and a
  Tailscale install. Migration = new instance, install Docker + Tailscale, `scp` the
  database, `docker compose up -d`, rename the old tailnet node and retire it.
- **Do the migration once while the AWS account is still free**, not on the last day.
  A rehearsed move is an afternoon; a rushed one loses data.

### Backups

- **The Go process takes its own snapshot**, running `VACUUM INTO` on a timer to a
  timestamped file under the bind-mounted `/data`; keep 7 days on the box.

  This is not a stylistic preference. The runtime image is `FROM scratch`, so there is
  **no `sqlite3` binary inside it** — any backup plan phrased as a shell command has no
  execution path at all without adding a sidecar container or abandoning the scratch
  base. `VACUUM INTO` is one `db.Exec` away in Go, is safe against a live database, and
  produces a single compact file. Do not reintroduce `sqlite3 .backup` anywhere.
- **Copy the newest one off the box**: `scp`/`rsync` to the laptop on a timer, over
  Tailscale. Provider-neutral, free, and it is what survives the AWS account closing.
  Optional while on AWS: also push to S3 for the practice, but keep it to one line in a
  script so the migration does not drag it along.
- `GET /export` is the human-readable JSON export, not the backup.
- **Do a restore drill once**, early. An untested backup is not a backup.

---

## 7. Build order

Each step should end with something I can actually use.

0. **Server reachable** — AWS account, EC2 instance, Docker, Tailscale,
   `tailscale serve`, and a hello-world container. Open it from the phone *before
   writing any application code*. Runbooks: [`docs/setup-aws.md`](setup-aws.md), then
   [`docs/setup-server.md`](setup-server.md).
1. **Skeleton** — `net/http` server, SQLite open with the §4 connection pragmas +
   migrations, CSRF middleware, health endpoint, settings row. Containerise it and
   confirm `time.LoadLocation` works inside the `scratch` image.
2. **Areas, then tasks** — areas first (small, and everything else references them),
   then task CRUD, server-rendered. This establishes the handler and template patterns
   reused everywhere after.
3. **Activities and progress** — the data model from §4, including target snapshots and
   backdating.
4. **Today screen** — grouped by area, with Hijri in the header and the timer
   start/stop flow. Then tune it until logging is genuinely a few seconds.
5. **History + export + backups** — including the restore drill.
6. **Use it for two weeks. Change nothing.** Keep a list of what is actually missing or
   annoying. That list, not this document, decides v2.

### Testing

Test the things where the bugs will actually be:

- Day-boundary and timezone resolution (the 1am case, DST transitions).
- Day rollups: sum of entries vs. `target_snapshot`, including zero-entry scheduled days.
- Backdating: an entry written today against yesterday's `day`.
- Target-change history: change a minimum, confirm past days are unaffected.
- **Materialisation**: the boundary ticker; startup catch-up across several missed days;
  never overwriting an existing row; no rows before an activity existed; archived
  activities skipped.
- **Schedule-change history**: drop a weekday from a schedule, then confirm past days
  that were scheduled still report as scheduled.
- **Reversal**: the total returns to its prior value; an entry cannot be reversed twice;
  a reversal cannot be reversed; the reversal's `day` and `activity_id` match the
  original's.
- **Connection pragmas**: `PRAGMA foreign_keys` reads back as 1 on a pooled connection,
  and a bad foreign key is actually rejected.
- Timer: stop on a time activity, on a quantity activity, and on a task; the edited
  duration is what gets written; a timer running across a day boundary books to the day
  it started; starting a second timer while one runs is rejected.
- **CSRF middleware**: a same-origin POST passes, a foreign `Origin` is rejected, a
  missing `Origin` on an unsafe method is rejected, and GET is unaffected.
- Hijri conversion against known date pairs, including the offset setting.

Do **not** write sync-ordering tests. There is no sync.

---

## 8. Honest note on motivation

Notion could do most of this in an afternoon, and the earlier claim that building this
is "easier than configuring Notion" is not true. The real reasons to build it are: I
want to write a real Go application, I want a tool shaped exactly like how I work, and
I will keep touching this codebase for months.

Those are good reasons. But they mean the project is judged on two things — whether I
still open the app daily in a month, and whether I understand every line of it. Not on
how many features it has.
