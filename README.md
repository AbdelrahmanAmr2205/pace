# pace

A personal productivity app: ordinary tasks, plus **scalable activities** — recurring
things that have a daily minimum target, a unit, and a log of how much was actually
done. "Study algorithms, at least 30 minutes" is an activity; "30 + 20 minutes logged
today" is its progress; "finish the database assignment" is just a task.

Built for one user (me), on two clients (laptop and phone). It is not trying to be
Todoist or Notion.

**Status: design done, no code yet.**

## Shape

One Go server, one SQLite database, server-rendered HTML. Both devices are thin
clients, so there is exactly one copy of the data and no synchronisation layer at all.

```
   Laptop browser  ─┐
                    ├─ HTTPS via Tailscale ─→  Go binary (net/http + html/template)
   Phone browser   ─┘                                    │
                                                         └─→ pace.db (SQLite, WAL)
```

- **Go**, stdlib `net/http` and `html/template`. No framework.
- **SQLite** via `modernc.org/sqlite` — pure Go, no cgo.
- **No JavaScript** in v1. Plain HTML forms, no build step.
- **Docker**, multi-stage to a `FROM scratch` image.
- **Tailscale** for access: reachable from anywhere, nothing exposed to the public
  internet, real HTTPS, and no authentication code in v1.
- Hosted on AWS EC2 to start, migrating to Oracle Cloud Always Free when the free
  window ends.

## Design principles

- **Progress entries are append-only.** A day's total is always `SUM(amount)`, never a
  mutable counter.
- **Amounts are integers** in the activity's base unit. "Did I hit the minimum" is
  never a floating-point comparison.
- **Each day's target is snapshotted**, so raising a minimum next month does not
  rewrite whether past days succeeded.
- **Quick capture is the feature.** If logging 25 minutes takes more than a few
  seconds, the app has failed regardless of what else it does.
- **No retroactive debt.** A missed minimum does not accumulate into anything. This is
  not meant to be a guilt machine.

## Docs

- [`docs/decisions.md`](docs/decisions.md) — what was decided, what was rejected, and
  why. Read this before changing the architecture.
- [`docs/setup-aws.md`](docs/setup-aws.md) — AWS account from scratch: root hardening,
  budgets, least-privilege identities, and launching the instance with no open ports.
- [`docs/setup-server.md`](docs/setup-server.md) — Docker + Tailscale runbook.

## Roadmap

0. Server reachable — hello-world container opening on the phone
1. Skeleton — server, SQLite, migrations, settings
2. Tasks
3. Activities and progress entries
4. Today screen
5. History, export, backups
6. Use it for two weeks before changing anything

## Licence

MIT — see [LICENSE](LICENSE).
