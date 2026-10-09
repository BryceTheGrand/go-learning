# Zoo Service: building a production-shaped Go web API

This is a remake of the official Go tutorial [Designing an API with Gin](https://go.dev/doc/tutorial/web-service-gin). The official tutorial builds a jazz record store in a single `main.go` with an in-memory list as its database. It teaches gin and JSON handling, but on purpose it skips a lot:

- no database
- no project structure (everything is one file)
- no auth, no roles, no real business logic
- no tests, no logging, no shutdown handling

This tutorial rebuilds the same idea as something closer to what you would actually ship: a **zoo management system** backed by PostgreSQL. Zookeepers log in with JWT bearer tokens. Keepers care for animals. Admins manage accounts. Business rules like "an animal has one primary keeper" and "feeding writes a log row and updates the animal in one transaction" are enforced in Go code and in the database.

Along the way it swaps the official tutorial's gin for the stack you will meet in most production Go services: **[chi](https://github.com/go-chi/chi) for routing, [cobra](https://github.com/spf13/cobra) and [viper](https://github.com/spf13/viper) for the command line and configuration, and `log/slog` for logging**. The single most important consequence of that swap is that your handlers are ordinary `func(http.ResponseWriter, *http.Request)` functions, not functions shaped by a framework - Stage 1 makes the case for it, and by the end of the tutorial the only thing tying the HTTP layer to chi is the route table.

What you will build, roughly in shape (the stages make it appear incrementally):

```
.
|-- cmd/
|   `-- zoo/main.go             # the thin entry point: cli.Execute()
|-- internal/
|   |-- cli/                    # cobra commands: serve, migrate
|   |-- animals/                # animal domain: handler, service, repository, dto
|   |-- zookeepers/             # zookeeper domain: handler, service, repository, dto
|   |-- platform/               # config (viper), database pool, auth, httpx, logging
|   `-- server/                 # router: mount the domains
|-- migrations/                 # plain SQL files, applied in order
|-- docker-compose.yml          # PostgreSQL for development
`-- Makefile
```

Two honest disclaimers up front, because you will meet them online:

1. **There is no official Go project layout.** [golang-standards/project-layout](https://github.com/golang-standards/project-layout) is a popular community convention, not a standard defined by the Go team. For a genuinely small app, a single `main.go` + `go.mod` is completely fine, and the project-layout repo itself says so. What *is* backed by the official Go guidance ("Organizing a Go module") is the two parts of it we adopt, and they are the two parts you will see in nearly every serious Go server:
   - `cmd/` - each executable binary gets a directory with a thin `main.go` whose only job is to wire things up.
   - `internal/` - the compiler itself enforces that code under `internal/` cannot be imported by another module, which means you can freely refactor your private code without breaking strangers.

2. **We start flat and grow into the structure.** [Stage 1](stages/01-official-port-flat.md) looks just like the official tutorial (one file, in-memory data). We take on dependencies, a database, and auth while things are still flat; only when the file count genuinely hurts do we reorganize ([Stage 5](stages/05-restructure-internal.md)). You will feel the problem the layout solves before you learn the layout. This is deliberate: starting with the whole skeleton on day one teaches you folder names, not reasons.

## What you need before starting

- Go 1.22+ (`go version` prints something like `go version go1.22.2 linux/amd64`)
- Docker with the compose plugin (`docker compose version` prints a version)
- A terminal and `curl`
- Basic programming experience in any language. No Go knowledge assumed.

Every dependency in this tutorial is pinned to an exact version, so the output you see matches what is printed here. The versions are also all buildable on Go 1.22; drop the `@version` suffix from a `go get` if you would rather track the newest releases.

## How to follow this tutorial

- Every stage ends with a **Verify** step: run the server, fire `curl` commands, and compare what you see against the expected output. Do not skip these. They are the checkpoint for the next stage.
- Code arrives in digestible pieces: a function or type in its own block, followed by a line-by-line explanation, and then, at the end of the section, the *complete file* as a reference - so you can confirm your assembled version. No Go knowledge is assumed, and anything new is explained when it first appears.
- By the end, the dependencies are: [chi](https://github.com/go-chi/chi) (a router, and a thin one - see Stage 1 for why we are not using a framework), [cobra](https://github.com/spf13/cobra) and [viper](https://github.com/spf13/viper) (the command line and configuration pair you will meet in `kubectl`, `hugo`, and most Go services), Go's own `log/slog` for structured logging, [pgx](https://github.com/jackc/pgx) (the community-recommended PostgreSQL driver, used instead of the more generic `database/sql`), [goose](https://github.com/pressly/goose) (only for tracking and applying the SQL files we write ourselves, used as a library inside our own `migrate` command), [golang-jwt/jwt/v5](https://github.com/golang-jwt/jwt) and Go's built-in bcrypt for auth.
- If a verify step fails, the fix is almost always in the "Gotchas" callouts close to where you are.

## The stages

Each stage is its own document. Follow them in order; every stage ends with a **Verify** checkpoint, and each document closes with links to the previous and next stage.

| # | Stage (click to open) | What you build | Go concepts introduced |
|---|---|---|---|
| 1 | [The official tutorial, ported to a zoo](stages/01-official-port-flat.md) | Flat API: health check + in-memory animals | modules, packages, structs, slices, functions, chi basics, stdlib HTTP handlers, struct tags, `encoding/json` |
| 2 | [A real database](stages/02-postgres-config-pool-migrations.md) | Docker Compose Postgres, config, connection pool, logging, a CLI, migrations | viper config, cobra commands, `RunE` and the main/run split, `defer`, slog, pgxpool |
| 3 | [The zookeeper domain](stages/03-zookeeper-domain-crud.md) | Zookeeper CRUD against the database | repository/service/handler layering, request decoding, closures |
| 4 | [Real passwords and login](stages/04-auth-bcrypt-jwt.md) | bcrypt passwords + JWT + auth middleware | middleware, closures, `defer`, custom JWT claims, context keys |
| 5 | [Restructure into `cmd/` + `internal/`](stages/05-restructure-internal.md) | The layout every serious Go server has, and the server joins the CLI | import paths, `internal/`, exported vs lowercase, dependency injection, chi route groups |
| 6 | [Animals against the database](stages/06-animals-database.md) | Joins, `NULL` handling, filters | LEFT JOIN, pointer fields for nullable columns |
| 7 | [Business logic](stages/07-business-logic.md) | Assign a keeper, feed an animal, feed history | `context.Context` (the real one), transactions |
| 8 | [Roles and workload](stages/08-roles-workload.md) | Admin gate, seeded admin, workload summary | middleware factories, per-route middleware, aggregate SQL |
| 9 | [One error pipeline](stages/09-error-pipeline-logging.md) | Typed errors + a structured request log | error types, `errors.As`, wrapping with `%w`, `slog`, response envelopes |
| 10 | [Graceful shutdown](stages/10-graceful-shutdown.md) | Clean exits on Ctrl+C and SIGTERM | signals, server timeouts, the one goroutine you need |
| 11 | [Tests](stages/11-tests.md) | Fakes at the service seam, handler tests | interfaces, `httptest`, table-driven tests |
| 12 | [Makefile, README, recap](stages/12-makefile-recap.md) | The workflow plus honest tradeoffs and exercises | - |
| 13 | [UUID primary keys](stages/13-uuid-primary-keys.md) | Swap every id from `bigint` to `uuid`: the migration under live foreign keys, the sweep through the code | `pgtype.UUID`, `gen_random_uuid`, expand-and-contract key change, JSON and `Scan` for ids |

## Bonus material

Three optional sections that take the finished service further into ERP territory. They are not part of the linear path and nothing depends on them, but they are where the interesting database work is: constraints that make a class of bug impossible rather than detectable, and the first rule that spans more than one tenant.

| # | Section (click to open) | What you build |
|---|---|---|
| 14 | [Bonus: multiple zoos and enclosures](stages/14-bonus-multi-zoo-enclosures.md) | Row-level tenancy, `zoo_id` on animals, enclosures with environments and capacity, row level security, an occupancy view |
| 15 | [Bonus: shifts across zoos, and paying for them](stages/15-bonus-scheduling-payroll.md) | Rosters as `tstzrange` with an `EXCLUDE` constraint, recurring shift templates, feed-only-while-on-shift, time-versioned pay rates, an idempotent payroll run |
| 16 | [Bonus: transfers, and where to go next](stages/16-bonus-transfers-and-next-steps.md) | Animal transfer as a state machine, guarded transitions, transactional capacity re-checks, roles growing into scoped permissions, an audit trail |

Every migration and query in Stage 13 and the bonus sections was executed against Postgres 17 before being written down, including the ones that are supposed to fail, so the `ERROR` messages shown are the ones you will actually see.
