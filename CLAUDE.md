# CLAUDE.md

Personal productivity app: tasks plus *scalable activities* (daily minimum target, a
unit, and an append-only log of what was actually done). Single user, two browsers.

- [`docs/decisions.md`](docs/decisions.md) — architecture, data model, v1 scope, and
  the rejected alternatives. **Read before proposing any architectural change.**
- [`docs/setup-aws.md`](docs/setup-aws.md) — AWS account, IAM, and instance launch.
- [`docs/setup-server.md`](docs/setup-server.md) — Docker + Tailscale runbook.

Module: `github.com/AbdelrahmanAmr2205/pace` · Go 1.26

---

## How to work in this repo

**This project exists partly so the owner learns Go. Do not write the interesting
parts for him.** Speed is not the goal; a codebase he understands line by line is.

**Calibrate to his actual level.** He has written HTTP handlers and table-driven tests
in Go before, through university work and Boot.dev. Do not explain `func`, `:=`,
slices, or what a test table is. What is genuinely new here is that **he owns the
product**: every previous project handed him the requirements one step at a time, and
this is the first application he is building to actually use. So the support that
helps is design-level — package boundaries, where logic belongs, idiomatic error
handling, trade-offs between two workable schemas — plus honest pushback on feature
and scope decisions. Treat his product choices as decisions to be stress-tested, not
instructions to be executed silently.

### Claude writes

Structure and wiring: package layout, `go.mod`, `main`, routing, middleware, template
loading, config, migrations, Dockerfile, `compose.yaml`, Makefile — **and all tests.**

### The owner writes

Business logic. Concretely: day-boundary and timezone resolution, day rollups,
target snapshotting, schedule matching, validation rules, and the non-trivial SQL.

### The handoff protocol

For anything on his side, produce the *shape* and stop:

```go
// ResolveDay maps an instant to the local calendar day it belongs to, honouring
// settings.DayStartsAt — 01:30 with a 04:00 boundary belongs to the previous day.
// See TestResolveDay for the exact cases this must satisfy.
func ResolveDay(t time.Time, s Settings) (string, error) {
	panic("TODO: implement — see TestResolveDay")
}
```

Then say, in the response: what the function must do, which test file specifies it,
and how to run just those tests. Do **not** fill in the body unless he explicitly asks
("just write it", "fill it in", "I'm stuck — show me").

**Tests are the spec.** Write them first, make sure they fail for the right reason,
and hand them over. Never weaken a test to make an implementation pass.

**First of a kind is a worked example — only for patterns that are actually new to
him.** A repository method over `database/sql`, a migration runner, template
composition, the timer's start/stop flow: write one complete idiomatic version and
explain it, then hand over the rest of that kind. Plain HTTP handlers and table-driven
tests do not need this treatment; hand those over directly.

**Explain Go-specific choices as they are made**: why an interface here, why a value
receiver, why `errors.Is` over a type switch, why this belongs in its own package.
A paragraph in the response, not a wall of comments in the code.

---

## Non-negotiables

These are decided. Reopen them via `docs/decisions.md`, not mid-task.

- **Progress entries are append-only.** Totals are always `SUM(amount)`. Never add a
  mutable running-total column. **Corrections append a reversing row** (negative
  `amount`, `reverses_entry_id` set) — never an UPDATE, never a DELETE, never a
  `voided_at` flag that would force totals to start filtering.
- **Amounts are integers** in the activity's base unit (minutes, pages, reps, metres).
  No floats — hitting a minimum must never be a float comparison.
- **`activity_days.target_snapshot` is written once and never updated.** Changing an
  activity's target must not rewrite history. Rows are materialised by the day-boundary
  job, on startup catch-up, and before any target/schedule mutation — **not lazily when
  a page is rendered**, which loses schedule history. See `decisions.md` §4.
- **Elapsed time and amount are different columns.** `duration_min` is wall-clock
  minutes; `amount` is in the activity's base unit. They coincide only when
  `unit_kind='time'`, for which the base unit is always minutes. A timer is valid on any
  activity — do not propose restricting it to time-based ones.
- **`foreign_keys` must be ON for every connection.** SQLite defaults it OFF per
  connection, and `modernc.org/sqlite` silently ignores mattn-style DSN parameters. Set
  it via `_pragma=foreign_keys(ON)` in the DSN and keep the test that asserts it.
- **`import _ "time/tzdata"`** must stay in the binary. The image is `FROM scratch`
  with no `/usr/share/zoneinfo`, and the day boundary depends on IANA timezones.
- **Hijri dates are display-only.** Never stored, never a key, never a query filter.
  `day` is always a Gregorian `YYYY-MM-DD`.
- **At most one timer runs at a time**, enforced by the single-row `active_timer`
  table. Elapsed time is always derived from `started_at` server-side, never counted
  on the client.
- **Publish containers to `127.0.0.1` only.** Docker writes its own iptables rules and
  a plainly published port can reach the internet regardless of the cloud firewall.
- **No JavaScript, no CSS framework, no web framework, no ORM** in v1. Stdlib
  `net/http` and `html/template`, hand-written SQL, one hand-written stylesheet.
- **CSRF middleware guards every unsafe method.** Tailscale stops the internet opening
  connections; it does not stop another site making *your* browser open one, and network
  position is the only authority this app has. Reject mismatched `Origin` on
  POST/PUT/DELETE.
- **Nothing provider-specific.** The app moves from AWS to Oracle Cloud within six
  months (the AWS account *closes* at the end of the free plan); it must stay a static
  binary plus one SQLite file.

## Scope discipline

v1 has no priorities, task descriptions, subtasks, separate due-vs-scheduled dates,
recurring tasks, weekly views, reminders, notifications, countdown timers, streaks or
analytics. See `docs/decisions.md` §5 for the full list and the test for whether a new
request earns its way in (structural things early, scalar fields when missed). If a
request implies something on the out-list, say so and ask rather than quietly adding it.

**Habits are not an entity.** They are activities, grouped by area and rendered with
`display_style='compact'`. Do not propose a third top-level type.

The feature that matters most is **speed of logging**. Free numeric entry is the
primary input; shortcut buttons are optional extras. If a change makes recording an
amount slower, it is a regression even if it adds something.

---

## Commands

```bash
go test ./...                    # everything
go test ./internal/core -run TestResolveDay -v
go vet ./... && gofmt -l .       # must be clean before a commit
go run ./cmd/pace                # local dev, binds 127.0.0.1:8080
docker build -t pace:dev .
```

## Conventions

- Standard layout: `cmd/pace/`, `internal/…`. Nothing exported that need not be.
- Wrap errors with context (`fmt.Errorf("load settings: %w", err)`); sentinel errors
  for conditions callers branch on. No panics outside `main` and TODO stubs.
- Table-driven tests, `t.Run` subtests, `httptest` for handlers. Test behaviour
  through exported functions, not internals.
- SQL lives in the data layer only; no queries in handlers.
- Migrations are append-only numbered files. Never edit one that has run.
- All timestamps stored UTC RFC 3339. `day` is a local `YYYY-MM-DD` string.

## Git

### Branching

Work happens on a branch, never directly on `main`. One branch per unit of work —
usually a roadmap step from `decisions.md` §7. Name it `<type>/<short-kebab-summary>`,
using the same type vocabulary as the commit subjects below so a branch and its commits
agree:

`feat/activities-crud` · `fix/day-boundary-dst` · `docs/backup-runbook` ·
`refactor/split-data-layer` · `test/timer-rollover` · `chore/bump-compose`

The cycle:

```bash
git switch -c feat/activities-crud    # branch off main
# ... work until tests pass, then he reviews the diff here ...
git switch main
git merge --no-ff feat/activities-crud
git push
git branch -d feat/activities-crud
```

**`--no-ff` is deliberate.** It keeps each branch visible as one unit, so
`git log --first-parent main` reads back as the list of steps actually completed rather
than a flat stream of commits. A fast-forward merge erases that boundary permanently.

Rebasing onto `main` before merging is normally a no-op here, since nothing else moves
`main` — don't bother unless it genuinely moved. **Never rewrite history that has been
pushed.**

### Tags

Annotated tags mark milestones, not merges:

```bash
git tag -a v0.1.0 -m "First real page served from real data"
git push --tags
```

`v0.1.0` when the app first serves a real page from real data. `v1.0.0` at
`decisions.md` §7 step 6 — used daily for two weeks with nothing changed. The point is
being able to go back and see what it looked like when it started earning its keep.

### Commit messages

Conventional Commits:

```
<type>(<scope>): <subject>

<body — why, not what>
```

- **Type**: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `build`, `perf`.
- **Scope**: the area touched — `activities`, `tasks`, `areas`, `progress`, `timer`,
  `today`, `history`, `export`, `db`, `http`, `hijri`, `settings`, `docker`, `deploy`.
  Omit only when the change is genuinely global.
- **Subject**: imperative mood ("add", not "added" or "adds"), lowercase, no full stop,
  50 characters or fewer.
- **Body**: wrapped at 72 characters, and where the actual reasoning lives.

```
feat(timer): record duration separately from amount
fix(db): enable foreign keys on every pooled connection
docs(decisions): explain why activity_days is materialised eagerly
```

**The format constrains the first line only.** The body still carries a real
explanation — the trade-off taken, the alternative rejected, the failure mode being
prevented. A tidy conventional subject over an empty body is a downgrade from what this
repo already has, not an improvement.

Commits predating this convention do not follow it. Leave them alone.

### When to commit

**Never commit, merge or push unless he asks.** He reviews the diff first, in this
session. When a step is complete and its tests pass, say so, summarise what changed, and
leave the work in the tree — do not create the commit yourself and do not stage it in
anticipation. Each of commit, merge and push is a separate request; asking for one is
not asking for the next.

`go vet ./... && gofmt -l .` must be clean before proposing a commit. If the suite is red
or the work is half finished, say so rather than quietly recording it.
