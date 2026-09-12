# CLAUDE.md

Personal productivity app: tasks plus *scalable activities* (daily minimum target, a
unit, and an append-only log of what was actually done). Single user, two browsers.

- [`docs/decisions.md`](docs/decisions.md) — architecture, data model, v1 scope, and
  the rejected alternatives. **Read before proposing any architectural change.**
- [`docs/setup-server.md`](docs/setup-server.md) — EC2 + Docker + Tailscale runbook.

Module: `github.com/AbdelrahmanAmr2205/pace` · Go 1.26

---

## How to work in this repo

**This project exists partly so the owner learns Go. Do not write the interesting
parts for him.** Speed is not the goal; a codebase he understands line by line is.

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

**First of a kind is a worked example.** For a new pattern — the first HTTP handler,
the first repository method, the first table-driven test — write one complete,
idiomatic version end to end and explain it. Then hand over the rest of that kind.

**Explain Go-specific choices as they are made**: why an interface here, why a value
receiver, why `errors.Is` over a type switch, why this belongs in its own package.
A paragraph in the response, not a wall of comments in the code.

---

## Non-negotiables

These are decided. Reopen them via `docs/decisions.md`, not mid-task.

- **Progress entries are append-only.** Totals are always `SUM(amount)`. Never add a
  mutable running-total column.
- **Amounts are integers** in the activity's base unit (minutes, pages, reps, metres).
  No floats — hitting a minimum must never be a float comparison.
- **`activity_days.target_snapshot` is written once and never updated.** Changing an
  activity's target must not rewrite history.
- **`import _ "time/tzdata"`** must stay in the binary. The image is `FROM scratch`
  with no `/usr/share/zoneinfo`, and the day boundary depends on IANA timezones.
- **Publish containers to `127.0.0.1` only.** Docker writes its own iptables rules and
  a plainly published port can reach the internet regardless of the cloud firewall.
- **No JavaScript, no CSS framework, no web framework, no ORM** in v1. Stdlib
  `net/http` and `html/template`, hand-written SQL, one hand-written stylesheet.
- **Nothing provider-specific.** The app moves from AWS to Oracle Cloud in ~3 months;
  it must stay a static binary plus one SQLite file.

## Scope discipline

v1 has no projects, priorities, task descriptions, subtasks, reminders, streaks, or
analytics. If a request implies one, say so and ask rather than quietly adding it.
Fields get added back the first time they are genuinely missed, not in anticipation.

The feature that matters most is **speed of logging**. If a change makes recording
25 minutes slower, it is a regression even if it adds something.

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

Commit when a roadmap step is complete and its tests pass. Real messages explaining
*why*, not a restatement of the diff. **Do not push** — that is his call. Do not
commit work-in-progress or a failing suite.
