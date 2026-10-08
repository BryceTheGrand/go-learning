## Stage 2: A real database: Postgres, config, pool, migrations

The server still serves in-memory animals. This stage wires the plumbing a database-backed app needs: a containerized Postgres, configuration, a pooled connection, and a small migration tool. Stage 3 will make zookeepers the first thing actually stored.

### 2.1 Run Postgres with Docker Compose

`docker-compose.yml` in the project root:

```yaml
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: zoo
      POSTGRES_USER: zoo
      POSTGRES_PASSWORD: zoo
    ports:
      - "5433:5432"   # host 5433 -> container 5432
    volumes:
      - zoo_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zoo -d zoo"]
      interval: 2s
      timeout: 2s
      retries: 15

volumes:
  zoo_pgdata:
```

Notes that will save you from mysterious failures:

- Host port **5433**, not 5432: if anything (a local Postgres install, another project) is squatting on 5432, mapping to it fails or - sneakier - connects you to the *other* database and nothing in this tutorial's config would explain the row counts. We pin a boring non-standard port instead.
- `volumes: zoo_pgdata` keeps data between `docker compose down`. Your schema survives Ctrl+C; wipe it with `docker compose down -v` (the `-v` also removes the volume).
- The healthcheck matters: Postgres accepting TCP connections does not mean it finished initializing on first boot. Other stages rely on the database being genuinely ready, and `pg_isready` waits for that.

```bash
docker compose up -d
docker compose ps        # db "healthy" within a few seconds
```

### 2.2 Configuration from environment variables

This file also introduces a directory you have not met: `internal/platform/`. `platform` is this project's name for *infrastructure the domains all lean on* - configuration, database, later auth and error plumbing. It is lowercase (`internal`) so the compiler keeps outsiders away; the reason `internal` means that arrives properly in Stage 5.

As in Stage 1, we build the file piece by piece and give you the whole file at the end.

#### The type

`internal/platform/config/config.go`:

```go
package config

import "os"

// Config is the process's configuration, parsed once at startup.
type Config struct {
	Port        string
	DatabaseURL string
}
```

- **`package config`**: first non-`main` package of the project. Import path `zoo/internal/platform/config`, package name `config` - other files will say `config.Load()`, the directory name matching the package name by convention.
- **`Config struct`**: a struct again, like Stage 1's `animal`, but exported (capital `C`) because *other packages* will build and consume it. Its fields are exported too - a configuration type is exactly the thing you want everyone to be able to read. Note there are no JSON tags: nothing here is ever serialized; tags are wire-shape concerns only.
- Note what the fields' *types* tell you: both strings. A URL is text; there is no integer type in Play-Your-Cards-Right with "5433" vs `5433`. (Stage 4 adds a duration type, `time.Duration`, and shows that trade again.)

#### Load: parse-and-return

```go
// Load reads configuration from environment variables. It follows one rule:
// sensible defaults for development, hard failure when a value is required
// and missing. Stage 4 adds the JWT secret here.
func Load() (Config, error) {
	cfg := Config{
		Port:        "8080",
		DatabaseURL: "postgres://zoo:zoo@localhost:5433/zoo?sslmode=disable",
	}

	if v := os.Getenv("PORT"); v != "" {
		cfg.Port = v
	}
	if v := os.Getenv("DATABASE_URL"); v != "" {
		cfg.DatabaseURL = v
	}

	return cfg, nil
}
```

- **`func Load() (Config, error)`**: the first *multi-value return* in the tutorial. Stage 1 taught the error convention from the caller's side (`x, err := f(...)`); this is the callee's side: **result first, error last**. Everything that can fail in this tutorial follows it.
- **`cfg := Config{...}`** is a struct composite literal, same construct as the three animals - just assigned and then *mutated* below, which is why the literal fields are the defaults and environment variables override them.
- **`if v := os.Getenv("PORT"); v != ""`** is the scoped-declaration `if` again (Stage 1's postAnimal): `v` exists only inside. `os.Getenv` returns `""` for absent variables in Go, never panics - check-for-empty is the standard idiom.
- **`return cfg, nil`**: the happy return. `nil` is the everything-is-fine error.
- Design note worth internalizing: **configuration is parsed once, at startup, into a plain struct** - no `config.Get("database_url")` scattered through the code. Callers receive the struct and pass what they need. This is a plain-Go substitute for config frameworks, and honestly it is what most Go servers do.
- Why does a function that cannot fail today return `(Config, error)`? Because Stage 4's JWT secret *will* be required and missing-secret must be a startup crash. Changing a function's return signature later means editing every caller; carrying the error slot costs nothing now. Go convention is to design returns for the steady state.

The complete file:

```go
package config

import "os"

// Config is the process's configuration, parsed once at startup.
type Config struct {
	Port        string
	DatabaseURL string
}

// Load reads configuration from environment variables. It follows one rule:
// sensible defaults for development, hard failure when a value is required
// and missing. Stage 4 adds the JWT secret here.
func Load() (Config, error) {
	cfg := Config{
		Port:        "8080",
		DatabaseURL: "postgres://zoo:zoo@localhost:5433/zoo?sslmode=disable",
	}

	if v := os.Getenv("PORT"); v != "" {
		cfg.Port = v
	}
	if v := os.Getenv("DATABASE_URL"); v != "" {
		cfg.DatabaseURL = v
	}

	return cfg, nil
}
```

One thing worth saying now because it shapes everything: **constructors that return `(value, error)` are the standard shape of Go error handling.** The convention is `error last` in every return list. The platform packages follow it everywhere.

### 2.3 The connection pool

`internal/platform/database/pool.go` - a package with one function. Its purpose is a decision disguised as plumbing: *one process, one pool*. Handlers do not each open connections; the pool owns them.

#### The signature: pointers appear for the first time

```go
package database

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// NewPool opens a pgx connection pool and verifies the database is reachable.
// A pool is safe for many concurrent users; a single pgx connection is not.
// Your handlers get a pool and let it manage who talks to Postgres when.
func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
	pool, err := pgxpool.New(ctx, databaseURL)
	if err != nil {
		return nil, fmt.Errorf("parse database url: %w", err)
	}

	if err := pool.Ping(ctx); err != nil {
		pool.Close()
		return nil, fmt.Errorf("connect to database: %w", err)
	}

	return pool, nil
}
```

- **`ctx context.Context` as the first parameter**: you met "a thing called context" nowhere yet, and this tutorial will not fully explain it until Stage 7. For now, accept the convention: functions that touch the outside world (database, network) take a `context.Context` first. It carries cancellation and deadlines; `context.Background()` is the "no parent" context, what `main` packages start with.
- **`*pgxpool.Pool`** is the first pointer type in a return position. Rules of thumb, now and for the rest of the tutorial: structs come back as pointers (*pgxpool.Pool, and from Stage 5 every domain handler/service), because you are handing over *the* thing, not a copy; a `nil` pointer is the "nothing" a failed constructor returns here (`return nil, fmt.Errorf(...)`).
- **`pool.Ping(ctx)`** is the fail-fast: a URL may parse fine and the database still be down. Catching that at process start, not at first request, is a production habit worth copying. (The `pool.Close()` before the error return releases the half-open pool - a constructor cleans up its own mess.)
- **`fmt.Errorf("parse database url: %w", err)`**: `%w` is Go's error-*wrapping* verb. The error you produce *contains* the one you were handed, so a caller can later ask "what was underneath?" (`errors.As`/`errors.Is`, exploited properly in Stage 9). The prefix text is your addition: it answers the "failed *where*?" question. Wrap at every layer, briefly; that is idiomatic Go, and the layering of messages in Stage 9's responses to failures will trace straight down to these words.

The complete file:

```go
package database

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// NewPool opens a pgx connection pool and verifies the database is reachable.
// A pool is safe for many concurrent users; a single pgx connection is not.
// Your handlers get a pool and let it manage who talks to Postgres when.
func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
	pool, err := pgxpool.New(ctx, databaseURL)
	if err != nil {
		return nil, fmt.Errorf("parse database url: %w", err)
	}

	if err := pool.Ping(ctx); err != nil {
		pool.Close()
		return nil, fmt.Errorf("connect to database: %w", err)
	}

	return pool, nil
}
```

### 2.4 The migration command

Migrations are plain SQL files. goose is a tiny library that tracks which files have been applied (in a table it manages for you) and applies the rest in order. We do not use the goose CLI; we wrap it in a second binary of our own. This file teaches two habits: the `main`/`run` split, and `defer`.

`cmd/migrate/main.go`:

#### main()

```go
package main

import (
	"context"
	"fmt"
	"os"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib"
	"github.com/pressly/goose/v3"

	"zoo/internal/platform/config"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, "migrate:", err)
		os.Exit(1)
	}
}
```

- **Why `main` is three lines.** `os.Exit(1)` terminates the process immediately, printing nothing itself - so a command should do its real work in `run()` (a *plain function*, returnable-with-error), and `main` should only: call it, print the error, exit nonzero. This is the standard Go CLI shape: logic lives in functions that *return* errors; `main` is the one place allowed to exit the process.

#### run(), part 1: context, config, pool, and the first defer

`run()` is one function. It is shown in two parts because each half has its own explanation, and each part is an exact excerpt of the final file - so where a block below stops mid-function, that is deliberate: the function is not finished, and the block has no closing brace:

```go
func run() error {
	ctx := context.Background()

	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("load config: %w", err)
	}

	pool, err := pgxpool.New(ctx, cfg.DatabaseURL)
	if err != nil {
		return fmt.Errorf("connect: %w", err)
	}
	defer pool.Close()
```

- **`context.Background()`**: the root context - "cancel me never". A command-line tool's whole life can hang off it; Stage 7 shows contexts that get cancelled per HTTP request (`c.Request.Context()`).
- **`defer pool.Close()`** is the first `defer` in the tutorial. It schedules the close call to run **when the surrounding function returns** - however it returns: the happy path, or any of the `return fmt.Errorf(...)` exits above and below. That is the value: one line, placed right after the resource is acquired, that can never be forgotten on the error paths. `defer`s run LIFO (last deferred, first to run); Stage 7 leans on this in its transaction rollback pattern.
- Part 1 stops at `defer pool.Close()`. There is no `return` and no closing brace yet, because `run` is only half-written at this point; part 2 picks up on the very next line.

#### run(), part 2: the goose run

This is the rest of that same `run()`, continuing on the line directly after `defer pool.Close()` above. Its final `}` is the brace that closes `run`:

```go
	// goose wants a database/sql-style handle; the pgxstdlib adapter
	// gives us one that is backed by the very same pool we already open.
	db := stdlib.OpenDBFromPool(pool)
	defer db.Close()

	if err := goose.SetDialect("postgres"); err != nil {
		return fmt.Errorf("set dialect: %w", err)
	}

	if err := goose.Up(db, "migrations"); err != nil {
		return fmt.Errorf("run migrations: %w", err)
	}

	return nil
}
```

- **`stdlib.OpenDBFromPool(pool)`** is an adapter: goose (like many libraries) speaks the old standard `database/sql` interface, and pgx's `stdlib` package hands it a `database/sql`-shaped view of your *existing* pool. Two types, one connection budget. `defer db.Close()` closes the adapter's view, not the pool (the pgx docs are explicit about that), so the first `defer` still owns the real resource. Note the teardown order that falls out of this: `defer`s run LIFO, so `db.Close()` runs first and `pool.Close()` second - the borrower is released before the thing it borrows from. That is the order you want, and it came for free from acquiring `db` after `pool`.
- A related trap worth filing away now: `defer`s fire when the *enclosing function returns*, and `os.Exit` does not run them. Cleanup deferred directly in `main` before an `os.Exit(1)` would be silently skipped. This file sidesteps that because `main` only calls `run()`; by the time `os.Exit(1)` runs, `run()` has already returned and its defers have already fired.
- **`goose.Up(db, "migrations")`** applies every not-yet-applied migration file in the directory, in filename order, and records them. `SetDialect` first tells it the vendor (it needs to know what a "now()" looks like, among other things).

The complete file:

```go
package main

import (
	"context"
	"fmt"
	"os"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib"
	"github.com/pressly/goose/v3"

	"zoo/internal/platform/config"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, "migrate:", err)
		os.Exit(1)
	}
}

func run() error {
	ctx := context.Background()

	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("load config: %w", err)
	}

	pool, err := pgxpool.New(ctx, cfg.DatabaseURL)
	if err != nil {
		return fmt.Errorf("connect: %w", err)
	}
	defer pool.Close()

	// goose wants a database/sql-style handle; the pgxstdlib adapter
	// gives us one that is backed by the very same pool we already open.
	db := stdlib.OpenDBFromPool(pool)
	defer db.Close()

	if err := goose.SetDialect("postgres"); err != nil {
		return fmt.Errorf("set dialect: %w", err)
	}

	if err := goose.Up(db, "migrations"); err != nil {
		return fmt.Errorf("run migrations: %w", err)
	}

	return nil
}
```

Two observations:

- This is the module's **second binary**. `go run .` runs whichever `package main` is in the current directory (that is the API server); `go run ./cmd/migrate` runs the migration command. One module, several programs; that is exactly why `cmd/` exists.
- It reads SQL files from the `migrations/` directory **relative to where you run it**, so always run it from the project root.

One version note for the record: `goose.SetDialect` + `goose.Up(db, dir)` is goose's legacy API - the current docs point to a newer `Provider` interface instead. We keep the legacy pair here because it is still fully supported, appears in every goose example you will find, and is the smallest thing that does the job; a note in the goose docs marks the newer API for when you want stricter store handling.

### 2.5 First migration: zookeepers

goose has its own file convention (a different one from golang-migrate): **one file per version**, with `-- +goose Up` and `-- +goose Down` annotations inside the file marking which statements go which direction. Create this file:

`migrations/00001_create_zookeepers.sql`:

```sql
-- +goose Up
CREATE TABLE zookeepers (
    id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username      text UNIQUE NOT NULL,
    password_hash text NOT NULL,
    role          text NOT NULL DEFAULT 'keeper' CHECK (role IN ('admin', 'keeper')),
    created_at    timestamptz NOT NULL DEFAULT now(),
    updated_at    timestamptz NOT NULL DEFAULT now()
);

-- +goose Down
DROP TABLE zookeepers;
```

Notes:

- File names are `<number>_<snake_case_description>.sql`. There must be **every version exactly once** - a duplicate version number across two files makes goose refuse to run (and so would two files with the same number in different sections). goose applies them in number order and refuses to re-run one that is already applied. `rm -rf migrations/00001*` would not be enough to "unapply" one on a database that has it; use `docker compose down -v` to start over from empty.
- The annotations are goose's "which half is this?" markers, not SQL comments to be ignored: everything under `-- +goose Up` runs on apply, everything under `-- +goose Down` runs on rollback. A version file must contain both (or explicitly be up/down-only, which we will not need).
- `GENERATED ALWAYS AS IDENTITY` is the modern replacement for `SERIAL`: Postgres assigns monotonically increasing integers. We use plain integers for IDs throughout this tutorial: they make `curl` examples copy-pasteable. Real-world systems frequently use UUIDs; a wrap-up note covers that swap (and its non-obvious pgx scanning caveats).
- `timestamptz` is "timestamp with time zone" and is what you should always pick for time columns. It stores an instant; serialization details show up in Stage 7.
- `password_hash` is a lie in Stage 3 and becomes true in Stage 4. We keep the column fixed from the start so the schema is stable; the code does not hash until Stage 4, and that dishonesty is part of the lesson.

Add the new dependencies (then tidy: `go get` records modules, `go mod tidy` pins the full go.sum closure of everything the code actually imports - without it the build can fail with "missing go.sum entry" for a dependency of a dependency):

```bash
go get github.com/jackc/pgx/v5 github.com/pressly/goose/v3
go mod tidy
```

### 2.6 Verify: a ready database

```bash
docker compose up -d          # already up, keeps schema
go run ./cmd/migrate
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo -c '\dt'
```

That last line is dense on a first meeting with Docker and with Postgres, so here it is pulled apart, left to right.

- **`docker exec`** runs a *new* command inside a container that is already running. A container is not a machine you log into; it is a process (here, the Postgres server), and `exec` starts a second process beside it, sharing the same filesystem namespace and network. Contrast `docker run`, which starts a *new* container. `exec` only works while the container's main process is alive - if the db container is stopped, there is nothing to exec into.
- **`-it`** is two flags: `-i` keeps STDIN open (so you can type into the command), `-t` allocates a pseudo-terminal (so the command believes it is attached to a real terminal, with a prompt and line editing). Together they make an interactive session possible. For a one-shot `-c` query like this one, `-it` is not actually needed - dropping it gives identical output - but it is the habitual form, and you will keep reusing this exact command later without `-c`, where it matters.
- **`$(docker compose ps -q db)`** is shell *command substitution*, not Docker syntax: the shell runs the inner command and splices its output into the outer one as text. `docker compose ps` lists the containers in this Compose project; `-q` ("quiet") prints only container IDs instead of a table; the trailing `db` filters to the service named `db` in `docker-compose.yml`. So the substitution means "whatever ID this project's `db` container currently has" - which is why the command keeps working after a `docker compose down && up -d` cycle hands the container a different ID. Compose also has a shorthand that skips the substitution entirely:

  ```bash
  docker compose exec db psql -U zoo -d zoo -c '\dt'
  ```

  `docker compose exec <service>` resolves the service for you. Both forms behave identically; this tutorial spells out the ID form because it makes clear *which* container is being entered.
- **`psql`** is Postgres's command-line client: the program that connects to a server and sends it SQL. It is not Docker-specific and not something we are inventing - it ships inside the `postgres` image, which is the only reason `exec` can run it. If you install `psql` on your host instead, you would connect over TCP with `-h localhost -p <the host port mapped in docker-compose.yml>` and, per the auth rule below, a password.
- **`-U zoo`** is the database *role* to connect as; **`-d zoo`** is the *database* to connect to. They match `POSTGRES_USER` and `POSTGRES_DB` in the compose file. Those are genuinely different things that happen to share the name `zoo` here: one is a login identity, the other is a named collection of tables. `-U` is required - without it psql defaults to your operating-system username, and no such role exists in this database.
- **Why no password is asked.** Inside the container psql connects over the local Unix socket (no `-h` means "use the socket"), and the image's `pg_hba.conf` contains `local all all trust`: socket connections are admitted with no password. Network connections follow a different rule, `host all all all scram-sha-256`, which is why the app's `DATABASE_URL` carries a password and this command does not.
- **`-c '\dt'`** means "run this one command string, then exit" instead of dropping into an interactive prompt. The string is a **meta-command**: psql shorthand beginning with a backslash, not SQL. `\dt` is "describe tables" - list them. Two constraints: a `-c` argument must be *either* pure SQL *or* a single backslash command, never a mixture; and bare `\dt` lists only objects **visible in your schema search path**, which for this connection is `public`. A table in some other schema would exist but not appear. `\dt *.*` lists every schema including Postgres's own catalogs - 111 rows on the container we checked, nearly all of them internals you will never touch.
- **The exit status propagates**: if the command inside fails, `docker exec` exits nonzero as well, so this shape is safe to use in scripts. (`docker exec <id> false` exits 1.)

Drop the `-c` and the command becomes the interactive shell, which is where you will want to spend real time poking around:

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo
```

Expected output from the `\dt` command:

```
             List of relations
 Schema |       Name       | Type  | Owner
--------+------------------+-------+-------
 public | goose_db_version | table | zoo
 public | zookeepers       | table | zoo
(2 rows)
```

`goose_db_version` is goose's own bookkeeping table - the record of which migration versions have been applied. It is why the tool is idempotent: on the next run it reads this table, sees version 1, and does nothing.

Run `go run ./cmd/migrate` a second time: it applies nothing and reports it (`goose: no migrations to run. current version: 1` in current goose). That is the mark of a healthy migration tool: idempotent at the command level. The API server (`go run .`) still behaves exactly like Stage 1 - animals in memory, nothing database-backed. Check `http://localhost:8080/healthz` responds as before.

---

[Stage 1](01-official-port-flat.md)  ·  [Overview](../tutorial.md)  ·  [Stage 3](03-zookeeper-domain-crud.md)
