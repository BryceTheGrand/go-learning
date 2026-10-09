## Stage 12: Makefile, README, and the honest recap

### 12.1 Makefile

`Makefile` (make is already on your machine; targets use tabs, not spaces - a Makefile rite of passage):

```makefile
.PHONY: up down migrate run test

up:
	docker compose up -d

down:
	docker compose down -v

migrate:
	go run ./cmd/zoo migrate

run:
	go run ./cmd/zoo serve

test:
	go test ./...
```

With `JWT_SECRET` exported (Stage 4), the whole development loop is now: `make up`, `make migrate`, `make run`, and in another terminal `make test`. One asymmetry worth noticing rather than papering over: `make migrate` also sources `config.Load()`, which requires `JWT_SECRET` - a migration command needs no token-signing key, but it loads the config wholesale. The clean fix (per-binary config subsets, or a migrate-only loader) is left as an instinct check; for this tutorial, keeping `JWT_SECRET` exported before any `make` target is the honest note. Short muscle-memory commands matter: the less a workflow costs, the more often you run it. One wrinkle to expect on `make run` and Ctrl+C: the shutdown lines appear as they should, and then `make` reports `make: *** [Makefile:13: run] Error 1`, because the signal that stopped your server also reached the `make` process wrapping it. (That is what a real terminal's Ctrl+C produces. If you signal the process group programmatically instead - from a script or a test harness - `make` may exit 0 with no line at all, so do not treat the absence of this message as a problem.) Nothing is wrong with the program - exit status 130 from a SIGINT is what the shell expects - but if that line bothers you, run the binary directly (`go build -o /tmp/zoo ./cmd/zoo && /tmp/zoo serve`) rather than through `make`.

### 12.2 The README someone else would need

`README.md` (this is also this repository's actual README):

```markdown
# Zoo Service

A Zoo management API: a remake of the official Go tutorial "Designing an API
with Gin", production-shaped, built on the standard Go server stack rather
than a web framework. Read `docs/tutorial.md` (the overview and table of
contents) and build it yourself, stage by stage - each stage lives in its own
file under `docs/stages/`.

## Stack

Go 1.22+, chi (router), cobra + viper (CLI and config), slog for structured
logging, pgx (PostgreSQL), goose (SQL migrations), golang-jwt/v5 + bcrypt,
Docker Compose for the database.

## Quick start

1. `export JWT_SECRET="a-long-random-string-for-dev-only-not-a-real-secret"`
2. `make up`
3. `make migrate`
4. `make run`
5. Log in: `POST /api/v1/login` with `{"username":"admin","password":"zoo-admin-password"}`
   (the seeded admin from migration 00003; dev credential, rotate per environment)

The single binary is `cmd/zoo`, with two subcommands: `zoo serve` runs the API,
`zoo migrate` applies the SQL migrations. Both read configuration from
environment variables (override anything with a `zoo.yaml` in the working
directory, or point at one with `--config`).

Full API documentation: the route table at the bottom of `docs/stages/12-makefile-recap.md`.

## Layout

`cmd/zoo` is the one binary. `internal/` holds the private code: one directory
per domain (animals, zookeepers), plus `cli` (the cobra commands), `server`
(the router), and `platform` (config, database, auth, httpx, logging). See
`docs/tutorial.md` Stage 5 for why.
```

### 12.3 The honest recap of the layout choice

What you built, evaluated against the conventions you set out to learn:

- **What is near-universal**: `cmd/` (thin binary roots) and `internal/` (compiler-enforced privacy). The official Go module guidance says server projects should use exactly this combination.
- **What is community convention (project-layout)**: domain grouping (`internal/<domain>/{handler,service,repository}`), `platform/` for shared infrastructure, the migrations directory, the composition root pattern. The CLI shape is convention too: `cmd/<binary>` holding a thin `main` that hands off to subcommands built with cobra, with configuration layered as defaults over an optional file over environment variables through viper, is the de facto standard for Go services - it is what `kubectl`, `hugo` and `gh` all do. Nothing official blesses any of it; lots of production codebases do something recognizable. The alternative is *layered* grouping (`internal/handlers`, `internal/services`, `internal/repositories` top-level). Try to name the code smell before reading: layered grouping puts one feature's change in three directories and makes each directory a mixed bag of unrelated concerns. Domain grouping localizes change; platform grouping exists precisely so domains do not import chi *and* pgx *and* goose knowledge into each other. Neither is dogma: small tools justifiably stay flat (Stage 1 was fine!), and monorepos with dozens of domains invent their own conventions.
- **What was skipped and why**: `pkg/` (nothing here is meant for import by other modules - and if it were, the guidance is to make it its own module), `api/` (OpenAPI specs; a fine exercise), `build/`, `scripts/`, `tools/`, `web/` (no CI story, no assets, an API-only tutorial). The project-layout repo is explicit that you should take what you need and delete the rest; that is what we did.

### 12.4 The finished API

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | /healthz | none | app + DB status |
| POST | /api/v1/login | none | returns token + zookeeper |
| POST | /api/v1/zookeepers | admin | create (role optional, defaults keeper) |
| GET | /api/v1/zookeepers | bearer | list |
| GET | /api/v1/zookeepers/{id} | bearer | one |
| PUT | /api/v1/zookeepers/{id} | admin | partial update (COALESCE) |
| DELETE | /api/v1/zookeepers/{id} | admin | 204; 409 keeper_in_use if feed history exists |
| GET | /api/v1/zookeepers/{id}/workload | bearer | scalar-subquery aggregate |
| POST | /api/v1/animals | admin | create |
| GET | /api/v1/animals | bearer | list; ?keeper_id= filter |
| GET | /api/v1/animals/{id} | bearer | one, with primary_keeper join |
| PUT | /api/v1/animals/{id} | admin | partial update |
| DELETE | /api/v1/animals/{id} | admin | 204; cascades feed_log |
| PUT | /api/v1/animals/{id}/keeper | admin | {keeper_id: 7 or null} |
| POST | /api/v1/animals/{id}/feed | bearer | {note?} -> 201, transactional |
| GET | /api/v1/animals/{id}/feed | bearer | latest 20 |

Errors everywhere are `{"error": {"code": ..., "message": ...}}`.

### 12.5 Where to go next (each is a real exercise, in rough order of difficulty)

1. **Token revocation or refresh**: the deleted-zookeeper-still-valids subtlety from Stage 7. Shorten the TTL and add `GET /api/v1/refresh`, or check account existence per request in `AuthMiddleware`.
2. **Pagination**: `?limit=`/`?cursor=` on both list routes; a cursor over `id` is the teaching version (offset pagination reads wrong at scale).
3. **Many-to-many care**: a `care_assignments` join table (many keepers per animal), which complicates every read; do it with the FK set you have now.
4. **Repository-level tests against a real Postgres** (dockerized, ephemeral database per run): this is where `isUniqueViolation` gets its automated test.
5. **A route-table parity test**: the `newTestRouter` in 11.3 mirrors the router by hand, with a comment asking to be kept honest. Replace the comment with a test: enumerate the method/path/middleware triples from `server.NewRouter` and assert the test router mounts the same set, so gating drifts fail a build instead of a code review.
6. **Test the CLI itself**: `internal/cli` is importable code precisely so commands can be tested (Stage 2.5). Build `newRootCmd()` (it is unexported, so the test file belongs in `package cli`, not `cli_test`), point `--config` at a temporary `zoo.yaml`, execute the command, and assert the parsed config - the one seam where viper's defaults-over-file-over-env layering is worth pinning.
7. **Factor the shared helpers**: `isUniqueViolation` and `isFKViolation` exist as private twins in two packages, and Stage 5 promised not to share until it hurts. It now hurts: move them into `internal/platform/pgerrors/` (or similar) and delete the copies.
8. **Where errors live, take two**: the per-domain sentinels were consolidated in Stage 9, but repository files still carry domain errors (animals). Try the alternative shape - an `errors.go` per domain package, or the "repository translates `pgx.ErrNoRows` at the boundary and services never see raw pg errors" convention - and feel the trade in diff size.
9. **feed_log growth**: the table grows forever by design (auditability). Two real-world alternatives to try: a monthly rollup table fed by a scheduled job, or partitioning `feed_log` by `fed_at` (Postgres native partitioning); each changes `FeedHistory` in instructive ways.
10. **UUIDs**: change IDs to `uuid` columns; you will meet `pgtype.UUID` scanning and must decide `id::text` projections - Stage 2's callout becomes concrete.
11. **Dockerize the server itself** (multi-stage build + compose service for the `zoo` binary), and add `POST /api/v1/animals/{id}/photo` backed by object storage.
12. **OpenAPI**: an `api/` directory with a spec generated or hand-written against the route table in 12.4.

If you only do one, do the pagination: it exercises every layer you built, touches SQL ordering, DTO shapes, and tests, and it is the first thing a reviewer will ask this API for.

---

You did the whole thing: an HTTP API with a real relational core, auth people actually use, one domain per directory, typed errors, structured logs, clean shutdown, and tests at the seam where they matter. The official tutorial you started from had one file and a slice. Look at the diff in your own head - that delta is "knowing Go" - and if you want more Go after this: [Effective Go](https://go.dev/doc/effective_go), the [Go module docs](https://go.dev/doc/modules/) on things like versioning and publishing, and reading real code (pgx's own source reads beautifully) will do the rest.

---

[Stage 11](11-tests.md)  |  [Overview](../tutorial.md)
