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
| Database | SQLite via `modernc.org/sqlite` | Pure Go — no cgo, so cross-compilation stays trivial. WAL mode. |
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
`docker save pace:latest | gzip | ssh pace 'gunzip | docker load'` works and needs no
account.

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
   what makes a future offline queue conflict-free.
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
    duration_min  INTEGER,                 -- optional; how long it actually took
    created_at    TEXT NOT NULL,
    updated_at    TEXT NOT NULL
);

-- A reusable activity definition.
CREATE TABLE activities (
    id             TEXT PRIMARY KEY,
    name           TEXT NOT NULL,
    unit           TEXT NOT NULL,          -- 'minutes' | 'pages' | 'reps' | 'metres' | ...
    unit_kind      TEXT NOT NULL,          -- 'time' | 'quantity'  (drives display only)
    minimum_target INTEGER NOT NULL,       -- current target, in base units
    schedule       TEXT NOT NULL,          -- weekday bitmask or 'daily'
    area_id        TEXT REFERENCES areas(id),
    display_style  TEXT NOT NULL DEFAULT 'card',  -- 'card' | 'compact'; see §5
    quick_amounts  TEXT,                   -- OPTIONAL '25,10' → extra +25 / +10 buttons
                                           -- alongside the free-entry field. Often NULL.
    sort_order     INTEGER NOT NULL DEFAULT 0,
    archived_at    TEXT,
    created_at     TEXT NOT NULL,
    updated_at     TEXT NOT NULL
);

-- One row per (activity, day) it was scheduled for. Created lazily; NEVER updated
-- once written. This is the target snapshot.
CREATE TABLE activity_days (
    id              TEXT PRIMARY KEY,
    activity_id     TEXT NOT NULL REFERENCES activities(id),
    day             TEXT NOT NULL,         -- local 'YYYY-MM-DD'
    target_snapshot INTEGER NOT NULL,
    created_at      TEXT NOT NULL,
    UNIQUE (activity_id, day)
);

-- Append-only log of work actually done.
CREATE TABLE progress_entries (
    id          TEXT PRIMARY KEY,          -- client-suppliable, for idempotent retries
    activity_id TEXT NOT NULL REFERENCES activities(id),
    day         TEXT NOT NULL,             -- local 'YYYY-MM-DD'; may be backdated
    amount      INTEGER NOT NULL,          -- base units; positive
    note        TEXT,
    recorded_at TEXT NOT NULL              -- UTC RFC3339, when it was entered
);

CREATE INDEX idx_progress_activity_day ON progress_entries (activity_id, day);
CREATE INDEX idx_progress_day          ON progress_entries (day);

-- Single-row settings table.
CREATE TABLE settings (
    id                INTEGER PRIMARY KEY CHECK (id = 1),
    timezone          TEXT NOT NULL,       -- IANA name
    day_starts_at     TEXT NOT NULL DEFAULT '04:00',
    week_starts_on    INTEGER NOT NULL DEFAULT 6,  -- 0=Sun .. 6=Sat
    hijri_offset_days INTEGER NOT NULL DEFAULT 0   -- -1 / 0 / +1, display only
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

When any day is rendered, upsert an `activity_days` row for every non-archived
activity scheduled on that day, snapshotting the current `minimum_target` — and never
overwrite an existing row. That covers both "scheduled but I did nothing" and "target
changed later."

**Known limitation, accepted:** if I don't open the app for a week, those days get
their rows created whenever I eventually look at them, snapshotting the *then-current*
target. Fine for a personal app. If it ever matters, a nightly job can materialise the
day ahead of time.

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
5. **Timer** — start on a task or activity, stop to write a progress entry (or a task
   duration) of the elapsed minutes. Server-side start time, no live ticking display.
6. **Today screen** — grouped by area: tasks scheduled today, each scheduled activity
   with `done / target`, and any running timer. Gregorian and Hijri date in the header.
   This is the home screen and where I will spend ~all my time.
7. **History** — per-activity view of the last N days: target, actual, entries.
8. **Export** — `GET /export` returns the whole database as JSON. My user-facing backup.

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
- **Use the tabular Umm al-Qura calculation**, via a small pure-Go library (no cgo —
  the image is `FROM scratch`), pinned and verified against known dates in a test.

Astronomical calculation and local moon sighting disagree, and sighting varies by
country, so the displayed date can legitimately be a day off. That is what
`settings.hijri_offset_days` (-1 / 0 / +1) is for: a one-line user correction rather
than an attempt to be authoritative about something the app cannot know.

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

**Phase 1 — AWS EC2 (~3–4 months).** Checked: the free window is roughly three months,
not perpetual. Starting here anyway, on purpose, because getting hands-on with EC2,
security groups, EBS and IAM is worth something independent of this app, and a known
expiry date is a feature — it forces the migration to actually happen instead of
becoming a someday task.

**Phase 2 — Oracle Cloud Always Free.** The Ampere A1 allowance (ARM) is far larger
than this app will ever need and does not expire. Two things to verify before
committing, because they are the usual complaints:

- A1 capacity is frequently unavailable in popular regions; pick the region on
  availability, not on latency.
- Always-Free tenancies have historically had **idle-resource reclamation** — an
  instance below low CPU/network/memory thresholds over a week can be reclaimed. A
  personal to-do app is the definition of idle by those metrics. The standard fix is
  upgrading the account to Pay As You Go, which stops reclamation while keeping the
  Always Free resources free. Confirm the current policy before relying on it, and do
  not let the only copy of the data live somewhere that can be reclaimed.

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

- Nightly `VACUUM INTO` a timestamped file; keep 7 days on the box.
- Copy the newest one off the box. Primary: `scp` to the laptop on a timer — provider-
  neutral, free, and survives the account expiring. Optional while on AWS: also push to
  S3 for the practice, but keep it to one line in a script so the migration does not
  drag it along.
- `GET /export` as the manual, human-readable backup.
- **Do a restore drill once**, early. An untested backup is not a backup.

---

## 7. Build order

Each step should end with something I can actually use.

0. **Server reachable** — EC2 instance, Docker, Tailscale, `tailscale serve`, and a
   hello-world container. Open it from the phone over cellular *before writing any
   application code*. Runbook: [`docs/setup-server.md`](setup-server.md).
1. **Skeleton** — `net/http` server, SQLite open + migrations, health endpoint,
   settings row. Containerise it and confirm `time.LoadLocation` works inside the
   `scratch` image.
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
- Timer: stop writes the correct elapsed minutes; a timer left running across a day
  boundary; starting a second timer while one runs is rejected.
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
