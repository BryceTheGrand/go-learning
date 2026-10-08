# Zoo Service: building a production-shaped Go web API

This is a remake of the official Go tutorial [Designing an API with Gin](https://go.dev/doc/tutorial/web-service-gin). The official tutorial builds a jazz record store in a single `main.go` with an in-memory list as its database. It teaches gin and JSON handling, but on purpose it skips a lot:

- no database
- no project structure (everything is one file)
- no auth, no roles, no real business logic
- no tests, no logging, no shutdown handling

This tutorial rebuilds the same idea as something closer to what you would actually ship: a **zoo management system** backed by PostgreSQL. Zookeepers log in with JWT bearer tokens. Keepers care for animals. Admins manage accounts. Business rules like "an animal has one primary keeper" and "feeding writes a log row and updates the animal in one transaction" are enforced in Go code and in the database.

What you will build, roughly in shape (the stages make it appear incrementally):

```
.
├── cmd/
│   ├── apiserver/main.go        # composition root: wire everything, run, shut down cleanly
│   └── migrate/main.go          # applies SQL migrations with goose
├── internal/
│   ├── animals/                 # animal domain: handler, service, repository, dto
│   ├── zookeepers/              # zookeeper domain: handler, service, repository, dto
│   ├── platform/                # config, database pool, auth, error types
│   └── server/                  # router: mount the domains
├── migrations/                  # plain SQL files, applied in order
├── docker-compose.yml           # PostgreSQL for development
└── Makefile
```

Two honest disclaimers up front, because you will meet them online:

1. **There is no official Go project layout.** [golang-standards/project-layout](https://github.com/golang-standards/project-layout) is a popular community convention, not a standard defined by the Go team. For a genuinely small app, a single `main.go` + `go.mod` is completely fine, and the project-layout repo itself says so. What *is* backed by the official Go guidance ("Organizing a Go module") is the two parts of it we adopt, and they are the two parts you will see in nearly every serious Go server:
   - `cmd/` - each executable binary gets a directory with a thin `main.go` whose only job is to wire things up.
   - `internal/` - the compiler itself enforces that code under `internal/` cannot be imported by another module, which means you can freely refactor your private code without breaking strangers.

2. **We start flat and grow into the structure.** Stage 1 looks just like the official tutorial (one file, in-memory data). We take on dependencies, a database, and auth while things are still flat; only when the file count genuinely hurts do we reorganize (Stage 5). You will feel the problem the layout solves before you learn the layout. This is deliberate: starting with the whole skeleton on day one teaches you folder names, not reasons.

## What you need before starting

- Go 1.27+ (`go version` prints something like `go version go1.27.1 darwin/arm64`)
- Docker with the compose plugin (`docker compose version` prints a version)
- A terminal and `curl` (your Mac has both)
- Basic programming experience in any language. No Go knowledge assumed.

## How to follow this tutorial

- Every stage ends with a **Verify** step: run the server, fire `curl` commands, and compare what you see against the expected output. Do not skip these. They are the checkpoint for the next stage.
- Code is written in stages; later files show the *complete new version* of a file, not a diff, unless the change is tiny. When a file does not change it is not shown again.
- All dependencies and versions used here are mainstream de-facto tools in the Go ecosystem: [gin](https://github.com/gin-gonic/gin) (most-used Go web framework, same one the official tutorial uses), [pgx](https://github.com/jackc/pgx) (the community-recommended PostgreSQL driver, used instead of the more generic `database/sql`), [goose](https://github.com/pressly/goose) (only for tracking and applying the SQL files we write ourselves, used as a library inside `cmd/migrate`), [golang-jwt/jwt/v5](https://github.com/golang-jwt/jwt) and Go's built-in bcrypt for auth.
- If a verify step fails, the fix is almost always in the "Gotchas" callouts close to where you are.

| Stage | What you build | Go concepts |
|---|---|---|
| 1 | Flat API: health check + in-memory animals (the official tutorial, ported) | modules, packages, gin basics, struct tags |
| 2 | Docker Compose Postgres, config, connection pool, migrations | two binaries, constructors, fail-fast config, pgxpool |
| 3 | Zookeeper CRUD against the database | repository/service/handler layering, `ShouldBindJSON` |
| 4 | bcrypt passwords + JWT login + auth middleware | middleware, closures, `defer`, custom JWT claims |
| 5 | **Restructure** into `cmd/` + `internal/` domains | import paths, `internal/`, exported vs lowercase, dependency injection |
| 6 | Animals against the database, joins and filtering | LEFT JOIN, NULLs and pointer fields, query filters |
| 7 | Business logic: assign a keeper, feed an animal, feed history | `context.Context` (the real one), transactions |
| 8 | Roles (admin vs keeper), seeded admin, workload summary | middleware factories, aggregate SQL |
| 9 | One error pipeline + structured logging with `slog` | error types, `errors.As`, wrapping with `%w` |
| 10 | Graceful shutdown | signals, server timeouts (the only goroutine material here) |
| 11 | Tests: table-driven services via fakes, handler tests | interfaces, `httptest`, table-driven tests |
| 12 | Makefile, README, and the honest layout recap | - |

---

## Stage 1: The official tutorial, ported to a zoo (flat, in-memory)

Everything in this stage lives in one file, exactly like the official tutorial. Two differences: our domain is animals, and we add a health-check endpoint you will rely on for the rest of the tutorial.

### 1.1 Create the module

Your project directory must contain only this tutorial (in this repo, `docs/tutorial.md`) when you start. You may find a leftover empty `go.mod` and `main.go` here; a `go.mod` that exists makes `go mod init` fail, so remove leftovers first:

```bash
rm -f go.mod main.go
go mod init zoo
```

`go mod init` creates `go.mod`, which declares this directory as a **module** named `zoo`. A module is Go's unit of versioned, importable code; the module path (`zoo` here, often a full URL like `github.com/you/zoo` in real projects) becomes the prefix of every import path inside the module. We use the short name `zoo` so import paths stay readable: code from Stage 5 will import `zoo/internal/animals`.

### 1.2 Install gin

```bash
go get -u github.com/gin-gonic/gin
```

This adds the gin dependency to `go.mod` and pins its exact version in `go.sum`, the lockfile. Run `go get` once more after any import changes if an editor complains that a package is unresolved.

### 1.3 The first main.go

Create `main.go`:

```go
package main

import (
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
)

// animal is the data we serve. The `json:"..."` tags set the field names
// gin uses when serializing to JSON: without them you get Go's title-cased
// names (ID, Name, Species), which is not what JSON APIs usually do.
type animal struct {
	ID       int64   `json:"id"`
	Name     string  `json:"name"`
	Species  string  `json:"species"`
	Enclosure string `json:"enclosure"`
}

// In-memory data, exactly like the official tutorial's slice of albums.
// Nothing here is persistent yet: every restart forgets your animals.
var animals = []animal{
	{ID: 1, Name: "Tembo", Species: "African bush elephant", Enclosure: "Savanna"},
	{ID: 2, Name: "Suki", Species: "Sumatran tiger", Enclosure: "Jungle"},
	{ID: 3, Name: "Biscuit", Species: "Red panda", Enclosure: "Forest"},
}

func main() {
	router := gin.Default()

	router.GET("/healthz", getHealth)
	router.GET("/api/v1/animals", getAnimals)
	router.GET("/api/v1/animals/:id", getAnimalByID)
	router.POST("/api/v1/animals", postAnimal)

	router.Run("localhost:8080")
}

func getHealth(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{"status": "ok"})
}

func getAnimals(c *gin.Context) {
	c.JSON(http.StatusOK, animals)
}

func getAnimalByID(c *gin.Context) {
	// :id in the route is a path parameter. Gin puts the actual value in
	// the request's context under the name of the parameter.
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "id must be a number"})
		return
	}

	for _, a := range animals {
		if a.ID == id {
			c.JSON(http.StatusOK, a)
			return
		}
	}

	c.JSON(http.StatusNotFound, gin.H{"error": "animal not found"})
}

func postAnimal(c *gin.Context) {
	var newAnimal animal
	if err := c.ShouldBindJSON(&newAnimal); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	newAnimal.ID = int64(len(animals) + 1)
	animals = append(animals, newAnimal)

	c.JSON(http.StatusCreated, newAnimal)
}
```

Things this code teaches, mapped to where you met them in the official tutorial:

- **`gin.Context`** is not Go's standard `context.Context` (that arrives in Stage 7, and the distinction matters). It carries the request and response plus helpers: `JSON` serializes, `Param` reads path parameters, `ShouldBindJSON` parses a request body into a struct.
- **Passing function names, not calls**: `router.GET("/healthz", getHealth)` registers the handler; it does not call it.
- **`gin.H`** is a cheat for "JSON object"; it is just `map[string]interface{}`.
- **Status codes are constants**: `http.StatusOK`, `http.StatusCreated`, `http.StatusBadRequest`, `http.StatusNotFound` - Go's `net/http` package, which gin builds on.

### 1.4 Run and verify

```bash
go run .
```

In a second terminal:

```bash
curl http://localhost:8080/healthz
# {"status":"ok"}

curl http://localhost:8080/api/v1/animals
# [{"id":1,"name":"Tembo","species":"African bush elephant","enclosure":"Savanna"}, ...]

curl http://localhost:8080/api/v1/animals/2
# {"id":2,"name":"Suki","species":"Sumatran tiger","enclosure":"Jungle"}

curl -X POST http://localhost:8080/api/v1/animals \
  -H "Content-Type: application/json" \
  -d '{"name":"Zuri","species":"Reticulated giraffe","enclosure":"Savanna"}'
# {"id":4,"name":"Zuri","species":"Reticulated giraffe","enclosure":"Savanna"}
```

> **Gotcha, learn it once now:** without `-H "Content-Type: application/json"`, `curl` sends `application/x-www-form-urlencoded`, and `ShouldBindJSON` fails or - worse in some handler shapes - parses into an empty struct. Every mutating request in this tutorial sends that header. If binding ever "silently fails", this is the first thing to check.

Stop the server with Ctrl+C. Your animals are gone, which is the entire motivation for Stage 2.

---

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

`internal/platform/config/config.go`:

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

One thing worth saying now because it shapes everything: **constructors that return `(value, error)` are the standard shape of Go error handling.** The convention is `error last` in every return list. The platform packages follow it everywhere. `Load` returns an error only because Stage 4 will make a missing JWT secret a real failure; today it cannot fail, but changing a function's return type later is a bigger edit than having it from the start.

### 2.3 The connection pool

`internal/platform/database/pool.go`:

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

`%w` is Go's error-wrapping verb: the returned error contains the original one (Stage 9 exploits this with `errors.As`). "Ping at startup" is a habit worth copying from production code: catch a misconfigured database URL when the process starts, not when the first request happens.

### 2.4 The migration command

Migrations are plain SQL files. goose is a tiny library that tracks which files have been applied (in a table it manages for you) and applies the rest in order. We do not use the goose CLI; we wrap it in a second binary of our own.

`cmd/migrate/main.go`:

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

Expected:

```
         List of relations
 Schema |    Name     | Type  | Owner
--------+-------------+-------+-------
 public | goose_db_version | table | zoo   <- goose's own bookkeeping table
 public | zookeepers       | table | zoo
```

Run `go run ./cmd/migrate` a second time: it applies nothing and reports it (`goose: no migrations to run. current version: 1` in current goose). That is the mark of a healthy migration tool: idempotent at the command level. The API server (`go run .`) still behaves exactly like Stage 1 - animals in memory, nothing database-backed. Check `http://localhost:8080/healthz` responds as before.

---

## Stage 3: The zookeeper domain, layered against the database

The API server is still one directory of files in `package main`, but zookeeper accounts now live in Postgres and touch three layers:

- **repository** - the only place with SQL. Knows the table, returns domain structs.
- **service** - business rules (this stage: valid username, unique username). Calls the repository.
- **handler** - decodes HTTP, asks the service, writes a status code. Calls the service.

Splitting now, while the rules are trivial, makes Stages 4-8 trivial too. In Stage 5 this trio moves to `internal/zookeepers/` and becomes its own package unchanged.

The three layers map to three new files. Create all three now.

`zookeepers_repository.go`:

```go
package main

import (
	"context"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

// zookeeper is the internal domain type: everything the database stores,
// including things (the password hash) that must never leave this domain.
type zookeeper struct {
	ID           int64
	Username     string
	PasswordHash string
	Role         string
	CreatedAt    time.Time
	UpdatedAt    time.Time
}

type zookeeperRepository struct {
	pool *pgxpool.Pool
}

func newZookeeperRepository(pool *pgxpool.Pool) *zookeeperRepository {
	return &zookeeperRepository{pool: pool}
}

func (r *zookeeperRepository) Create(ctx context.Context, zk zookeeper) (zookeeper, error) {
	row := r.pool.QueryRow(ctx,
		`INSERT INTO zookeepers (username, password_hash, role)
		 VALUES ($1, $2, $3)
		 RETURNING id, username, password_hash, role, created_at, updated_at`,
		zk.Username, zk.PasswordHash, zk.Role,
	)

	var created zookeeper
	err := row.Scan(&created.ID, &created.Username, &created.PasswordHash, &created.Role,
		&created.CreatedAt, &created.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return created, nil
}

func (r *zookeeperRepository) Get(ctx context.Context, id int64) (zookeeper, error) {
	var zk zookeeper
	err := r.pool.QueryRow(ctx,
		`SELECT id, username, password_hash, role, created_at, updated_at
		 FROM zookeepers WHERE id = $1`, id).
		Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role, &zk.CreatedAt, &zk.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return zk, nil
}

func (r *zookeeperRepository) List(ctx context.Context) ([]zookeeper, error) {
	rows, err := r.pool.Query(ctx,
		`SELECT id, username, password_hash, role, created_at, updated_at
		 FROM zookeepers ORDER BY id`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	zks := []zookeeper{}
	for rows.Next() {
		var zk zookeeper
		if err := rows.Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role,
			&zk.CreatedAt, &zk.UpdatedAt); err != nil {
			return nil, err
		}
		zks = append(zks, zk)
	}
	if err := rows.Err(); err != nil {
		return nil, err
	}
	return zks, nil
}

func (r *zookeeperRepository) Update(ctx context.Context, id int64, zku zookeeperUpdate) (zookeeper, error) {
	// COALESCE($n, col) means "use the new value if it was sent, else keep
	// the column as it is" - the standard SQL shape for partial updates.
	var zk zookeeper
	err := r.pool.QueryRow(ctx,
		`UPDATE zookeepers
		 SET username     = COALESCE($1, username),
		     password_hash = COALESCE($2, password_hash),
		     role         = COALESCE($3, role),
		     updated_at   = now()
		 WHERE id = $4
		 RETURNING id, username, password_hash, role, created_at, updated_at`,
		zku.Username, zku.PasswordHash, zku.Role, id).
		Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role, &zk.CreatedAt, &zk.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return zk, nil
}

func (r *zookeeperRepository) Delete(ctx context.Context, id int64) error {
	tag, err := r.pool.Exec(ctx, `DELETE FROM zookeepers WHERE id = $1`, id)
	if err != nil {
		return err
	}
	// DELETE does not error when nothing matched; check the row count.
	if tag.RowsAffected() == 0 {
		return errZookeeperNotFound
	}
	return nil
}
```

`zookeepers_service.go`:

```go
package main

import (
	"context"
	"errors"
)

// Sentinel errors: value-level errors the service decides on. Handlers map
// them to status codes without knowing SQL exists.
var (
	errZookeeperNotFound   = errors.New("zookeeper not found")
	errZookeeperDuplicate  = errors.New("username already taken")
	errZookeeperInvalid    = errors.New("username, password and role must be non-empty")
)

// zookeeperUpdate carries optional changes for PUT. nil means "not sent";
// a non-nil empty string means "set to empty string". Pointers are how Go
// distinguishes absent from empty in JSON.
type zookeeperUpdate struct {
	Username     *string
	PasswordHash *string
	Role         *string
}

type zookeeperService struct {
	repo *zookeeperRepository
}

func newZookeeperService(repo *zookeeperRepository) *zookeeperService {
	return &zookeeperService{repo: repo}
}

func (s *zookeeperService) Create(ctx context.Context, username, password, role string) (zookeeper, error) {
	if role == "" {
		role = "keeper"
	}
	if username == "" || password == "" || (role != "admin" && role != "keeper") {
		return zookeeper{}, errZookeeperInvalid
	}

	// Stage 3 shortcut: the raw password is stored. Stage 4 makes this real.
	zk, err := s.repo.Create(ctx, zookeeper{
		Username:     username,
		PasswordHash: password,
		Role:         role,
	})
	if err != nil {
		if isUniqueViolation(err) {
			return zookeeper{}, errZookeeperDuplicate
		}
		return zookeeper{}, err
	}
	return zk, nil
}

func (s *zookeeperService) Get(ctx context.Context, id int64) (zookeeper, error) {
	return s.repo.Get(ctx, id)
}

func (s *zookeeperService) List(ctx context.Context) ([]zookeeper, error) {
	return s.repo.List(ctx)
}

func (s *zookeeperService) Update(ctx context.Context, id int64, zku zookeeperUpdate) (zookeeper, error) {
	zk, err := s.repo.Update(ctx, id, zku)
	if err != nil {
		if isUniqueViolation(err) {
			return zookeeper{}, errZookeeperDuplicate
		}
		return zookeeper{}, err
	}
	return zk, nil
}

func (s *zookeeperService) Delete(ctx context.Context, id int64) error {
	return s.repo.Delete(ctx, id)
}
```

The uniqueness check needs one small helper, `isUniqueViolation`, which uses `errors.As` to find Postgres's error type. Put it at the bottom of `zookeepers_service.go`:

```go
// isUniqueViolation reports whether err is Postgres's unique-constraint
// violation (SQLSTATE 23505). errors.As unwraps the chain to find the
// concrete *pgconn.PgError inside.
func isUniqueViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23505"
}
```

which changes the import block of the file to:

```go
import (
	"context"
	"errors"

	"github.com/jackc/pgx/v5/pgconn"
)
```

`zookeepers_handler.go`:

```go
package main

import (
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
)

// zookeeperResponse is what the outside world sees. Note what is missing:
// PasswordHash. Responses are a separate shape from domain structs, and the
// password hash is the reason.
type zookeeperResponse struct {
	ID        int64     `json:"id"`
	Username  string    `json:"username"`
	Role      string    `json:"role"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

func (z zookeeper) toResponse() zookeeperResponse {
	return zookeeperResponse{
		ID:        z.ID,
		Username:  z.Username,
		Role:      z.Role,
		CreatedAt: z.CreatedAt,
		UpdatedAt: z.UpdatedAt,
	}
}

// parseID reads the :id path parameter, writing a 400 and returning false
// if it is not an integer. Small repetition killers like this are the first
// sign your "flat" files need Stage 5.
func parseID(c *gin.Context) (int64, bool) {
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "id must be an integer"})
		return 0, false
	}
	return id, true
}

type zookeeperHandler struct {
	svc *zookeeperService
}

func newZookeeperHandler(svc *zookeeperService) *zookeeperHandler {
	return &zookeeperHandler{svc: svc}
}

func (h *zookeeperHandler) create(c *gin.Context) {
	var req struct {
		Username string `json:"username" binding:"required"`
		Password string `json:"password" binding:"required"`
		Role     string `json:"role"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	zk, err := h.svc.Create(c.Request.Context(), req.Username, req.Password, req.Role)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			c.JSON(http.StatusConflict, gin.H{"error": "username already taken"})
			return
		}
		if errors.Is(err, errZookeeperInvalid) {
			c.JSON(http.StatusBadRequest, gin.H{"error": "username and password are required"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusCreated, zk.toResponse())
}

func (h *zookeeperHandler) list(c *gin.Context) {
	zks, err := h.svc.List(c.Request.Context())
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	resp := make([]zookeeperResponse, len(zks))
	for i, zk := range zks {
		resp[i] = zk.toResponse()
	}
	c.JSON(http.StatusOK, resp)
}

func (h *zookeeperHandler) get(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	zk, err := h.svc.Get(c.Request.Context(), id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, zk.toResponse())
}

func (h *zookeeperHandler) update(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	var req struct {
		Username *string `json:"username"`
		Password *string `json:"password"`
		Role     *string `json:"role"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	zku := zookeeperUpdate{Username: req.Username, Role: req.Role}
	if req.Password != nil {
		// Stage 3 stores it raw; Stage 4 hashes here.
		raw := *req.Password
		zku.PasswordHash = &raw
	}

	zk, err := h.svc.Update(c.Request.Context(), id, zku)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			c.JSON(http.StatusConflict, gin.H{"error": "username already taken"})
			return
		}
		if errors.Is(err, pgx.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, zk.toResponse())
}

func (h *zookeeperHandler) delete(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	if err := h.svc.Delete(c.Request.Context(), id); err != nil {
		if errors.Is(err, errZookeeperNotFound) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.Status(http.StatusNoContent)
}
```

The file's final import list (update your imports to match; the listing shows
the exact set):

```go
import (
	"errors"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/jackc/pgx/v5"
)
```

> **Wait, the handler imports pgx?** `errors.Is(err, pgx.ErrNoRows)` in the handler means the HTTP layer knows about the database driver. It works and it is honest about a middle stage, but Stage 9 replaces this: the service will translate "no rows" into its own error and the handler will stop importing pgx entirely. Note the flaw now and appreciate the fix later.

### 3.1 Wire the routes: replace main.go

Replace `main.go` completely:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"strconv"

	"github.com/gin-gonic/gin"

	"zoo/internal/platform/config"
	"zoo/internal/platform/database"
)

func main() {
	if err := run(); err != nil {
		slog.Error("startup failed", "error", err)
		os.Exit(1)
	}
}

func run() error {
	ctx := context.Background()

	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("load config: %w", err)
	}

	pool, err := database.NewPool(ctx, cfg.DatabaseURL)
	if err != nil {
		return fmt.Errorf("open database: %w", err)
	}
	defer pool.Close()

	zkRepo := newZookeeperRepository(pool)
	zkSvc := newZookeeperService(zkRepo)
	zkHandler := newZookeeperHandler(zkSvc)

	router := gin.Default()

	router.GET("/healthz", healthHandler(pool))

	zk := router.Group("/api/v1/zookeepers")
	{
		zk.POST("", zkHandler.create)
		zk.GET("", zkHandler.list)
		zk.GET("/:id", zkHandler.get)
		zk.PUT("/:id", zkHandler.update)
		zk.DELETE("/:id", zkHandler.delete)
	}

	router.GET("/api/v1/animals", getAnimals)
	router.GET("/api/v1/animals/:id", getAnimalByID)
	router.POST("/api/v1/animals", postAnimal)

	if err := router.Run("localhost:" + cfg.Port); err != nil {
		return fmt.Errorf("run server: %w", err)
	}
	return nil
}

// healthHandler is a function *returning* a handler function, a closure over
// the pool. This is the same shape auth middleware uses in Stage 4.
func healthHandler(pool *pgxpool.Pool) gin.HandlerFunc {
	return func(c *gin.Context) {
		if err := pool.Ping(c.Request.Context()); err != nil {
			c.JSON(http.StatusServiceUnavailable, gin.H{"status": "degraded", "db": "down"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"status": "ok", "db": "up"})
	}
}
```

which means `main.go` needs one more import than shown in the block above (add to the top-level import list):

```go
	"github.com/jackc/pgx/v5/pgxpool"
```

The Stage 1 animal handler functions (`getAnimals`, `getAnimalByID`, `postAnimal`) and the `animal` struct / `animals` slice **stay in `main.go`, unchanged below `run`**. Your root directory now has four `.go` files, all `package main`: `main.go`, `zookeepers_repository.go`, `zookeepers_service.go`, `zookeepers_handler.go`. That flat-and-growing feeling is the setup for Stage 5.

### 3.2 Verify: a real CRUD round-trip

```bash
go run .

# create (watch the Content-Type header; see the Stage 1 gotcha)
curl -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"giraffe-lane","role":"admin"}'
# {"id":1,"username":"maya","role":"admin","created_at":"2026-...","updated_at":"2026-..."}
# notice: no password anywhere in the response

# the other two
curl -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"sam","password":"elephant-road"}'
# {"id":2,"username":"sam","role":"keeper",...}

curl http://localhost:8080/api/v1/zookeepers
# [ ...maya..., ...sam... ]

curl http://localhost:8080/api/v1/zookeepers/1
# {"id":1,"username":"maya",...}

# update: change only the role, keep everything else
curl -X PUT http://localhost:8080/api/v1/zookeepers/2 \
  -H "Content-Type: application/json" \
  -d '{"role":"admin"}'
# {"id":2,"username":"sam","role":"admin",...}

# missing animal... er, zookeeper
curl http://localhost:8080/api/v1/zookeepers/99
# {"error":"zookeeper not found"} with status 404

# delete is 204: success, deliberately no body
curl -X PUT http://localhost:8080/api/v1/zookeepers/2 \
  -H "Content-Type: application/json" \
  -d '{"role":"keeper"}' > /dev/null   # reset sam to keeper for Stage 4
curl -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"temp","password":"x"}' > /dev/null
curl -X DELETE http://localhost:8080/api/v1/zookeepers/3 -i
# HTTP/1.1 204 No Content

# duplicate username: conflict, not a crash. Deliberately LAST in this
# script: a *failed* INSERT still consumes an identity value, so if you
# ran this attempt right after maya, sam would have been id 3 and temp
# id 4 and every later expected id would quietly lie. (Failed inserts
# burning sequence values is normal identity behavior; Stage 4.6 shows
# the TRUNCATE ... RESTART IDENTITY reset.)
curl -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"x"}'
# {"error":"username already taken"} with status 409
```

In the database:

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo \
  -c 'SELECT id, username, password_hash, role FROM zookeepers ORDER BY id'
```

You will see sam's and maya's **passwords sitting in the password_hash column as plain text**. That is Stage 3's deliberate, temporary flaw. Stage 4's first move is fixing exactly this, and the fix (bcrypt) does not change a single SQL statement: the layers did their job.

---

## Stage 4: Real passwords and login: bcrypt + JWT + middleware

Stage 3 leaves one deliberate disaster: raw passwords sitting in `password_hash`. Fix it now. This stage delivers the "auth of a standard form" part of the tutorial: `POST /api/v1/login` exchanges username/password for a JWT; zookeeper reads require a valid bearer token.

The auth packages need two new dependencies (bcrypt lives in Go's extended-standard-library repo, jwt in its own):

```bash
go get github.com/golang-jwt/jwt/v5 golang.org/x/crypto/bcrypt
go mod tidy
```

### 4.1 Passwords: platform/auth/password.go

```go
package auth

import (
	"fmt"

	"golang.org/x/crypto/bcrypt"
)

// HashPassword stores a password as a bcrypt hash. Note what is not here:
// a salt. bcrypt generates and embeds a random salt in every hash; hashing
// "hunter2" twice produces two different hashes, and that is correct.
//
// Worth knowing in passing: bcrypt only reads the first 72 bytes of its
// input (a historical limit the format cannot escape). x/crypto used to
// silently truncate; the current version instead rejects passwords longer
// than 72 bytes with ErrPasswordTooLong, so HashPassword surfaces that as
// an error rather than hashing a prefix of the real password. Modern
// designs (argon2, scrypt) do not have this limit; bcrypt remains the
// standard choice for its ubiquity and maturity.
func HashPassword(password string) (string, error) {
	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return "", fmt.Errorf("hash password: %w", err)
	}
	return string(hash), nil
}

// CheckPassword reports whether password hashes to hash. It uses constant
// time comparison internally; never compare hashes yourself with ==.
func CheckPassword(hash, password string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

### 4.2 Tokens: platform/auth/token.go

```go
package auth

import (
	"errors"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// Claims is our payload: who the caller is, plus enough to authorize them.
// ZookeeperID goes in so handlers never need a database lookup to know
// who is calling; that is a trade (staleness), noted in Stage 8.
type Claims struct {
	ZookeeperID int64  `json:"zookeeper_id"`
	Username    string `json:"username"`
	Role        string `json:"role"`
	jwt.RegisteredClaims
}

// ErrInvalidToken is the single error VerifyToken surfaces; the middleware
// maps everything (bad signature, expired, malformed) to one 401.
var ErrInvalidToken = errors.New("invalid token")

// IssueToken signs a Claims set with an HMAC (HS256) using secret.
// Production systems often move to asymmetric signing (RS256/Ed25519) so
// other services can verify tokens without holding the secret; HS256 with
// a moderate TTL is the standard single-API starting point.
func IssueToken(secret []byte, ttl time.Duration, id int64, username, role string) (string, error) {
	now := time.Now().UTC()
	claims := Claims{
		ZookeeperID: id,
		Username:    username,
		Role:        role,
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   username,
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(ttl)),
		},
	}

	signed, err := jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString(secret)
	if err != nil {
		return "", fmt.Errorf("sign token: %w", err)
	}
	return signed, nil
}

// VerifyToken parses raw, checks the signature was made by our secret, and
// validates standard fields like expiry. The keyFunc double-checks the
// algorithm: without it, an attacker can hand us a token we will happily
// verify against the wrong key (the classic JWT confusable-algorithm bug).
func VerifyToken(secret []byte, raw string) (Claims, error) {
	claims := Claims{}
	token, err := jwt.ParseWithClaims(raw, &claims, func(_ *jwt.Token) (any, error) {
		return secret, nil
	}, jwt.WithValidMethods([]string{jwt.SigningMethodHS256.Alg()}))
	if err != nil || !token.Valid {
		return Claims{}, ErrInvalidToken
	}
	return claims, nil
}
```

### 4.3 Auth middleware: platform/auth/middleware.go

Gin middleware is an ordinary function with the `gin.HandlerFunc` signature that runs before (and optionally after) the handler. Configuration is usually provided via a closure: `authMiddleware(secret)` *returns* a handler that captured `secret`. This is the pattern to recognize across all Go web frameworks.

```go
package auth

import (
	"net/http"
	"strings"

	"github.com/gin-gonic/gin"
)

// claimsKey is the context key under which AuthMiddleware stores the
// verified claims for handlers.
const claimsKey = "zoo.claims"

// AuthMiddleware requires a valid bearer token and stores the claims.
func AuthMiddleware(secret []byte) gin.HandlerFunc {
	return func(c *gin.Context) {
		raw, ok := strings.CutPrefix(c.GetHeader("Authorization"), "Bearer ")
		if !ok {
			// AbortWithStatusJSON stops the chain of remaining handlers.
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
				"error": "missing or malformed Authorization header",
			})
			return
		}

		claims, err := VerifyToken(secret, raw)
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
			return
		}

		c.Set(claimsKey, claims)
		c.Next() // run the actual handler now
	}
}

// ClaimsFrom returns the identity of the caller, when the request passed
// through AuthMiddleware.
func ClaimsFrom(c *gin.Context) (Claims, bool) {
	v, ok := c.Get(claimsKey)
	if !ok {
		return Claims{}, false
	}
	claims, ok := v.(Claims)
	return claims, ok
}
```

### 4.4 The three edits to the zookeeper domain

**1. `config.go` grows the JWT settings.** The complete new version:

```go
package config

import (
	"errors"
	"os"
	"time"
)

type Config struct {
	Port        string
	DatabaseURL string
	JWTSecret   string
	TokenTTL    time.Duration
}

func Load() (Config, error) {
	cfg := Config{
		Port:        "8080",
		DatabaseURL: "postgres://zoo:zoo@localhost:5433/zoo?sslmode=disable",
		TokenTTL:    24 * time.Hour,
	}

	if v := os.Getenv("PORT"); v != "" {
		cfg.Port = v
	}
	if v := os.Getenv("DATABASE_URL"); v != "" {
		cfg.DatabaseURL = v
	}

	cfg.JWTSecret = os.Getenv("JWT_SECRET")
	if cfg.JWTSecret == "" {
		return Config{}, errors.New("JWT_SECRET is required: set it to a long random string")
	}

	return cfg, nil
}
```

Failing to start without a secret is the point: an app that silently serves with a weak default is worse than one that refuses to boot. From now on, run the server with a secret:

```bash
export JWT_SECRET="a-long-random-string-for-dev-only-not-a-real-secret"
go run .
```

**2. `zookeepers_service.go`: hash on the way in, verify on the way through.** Three concrete edits:

- The import block becomes:

```go
import (
	"context"
	"errors"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"

	"zoo/internal/platform/auth"
)

var (
	errZookeeperNotFound  = errors.New("zookeeper not found")
	errZookeeperDuplicate = errors.New("username already taken")
	errZookeeperInvalid   = errors.New("username, password and role must be valid")
	errInvalidCredentials = errors.New("invalid credentials")
)
```

- In `Create`, replace the line that stores the raw password with a real hash:

```go
	hash, err := auth.HashPassword(password)
	if err != nil {
		return zookeeper{}, err
	}

	zk, err := s.repo.Create(ctx, zookeeper{
		Username:     username,
		PasswordHash: hash,
		Role:         role,
	})
```

- Add `Authenticate`, the method login calls, at the bottom of the service (before `isUniqueViolation`):

```go
// Authenticate verifies a login attempt. Both a wrong username and a wrong
// password return the same error: a distinct "no such user" message would
// tell an attacker which half of their guess is right.
func (s *zookeeperService) Authenticate(ctx context.Context, username, password string) (zookeeper, error) {
	zk, err := s.repo.GetByUsername(ctx, username)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return zookeeper{}, errInvalidCredentials
		}
		return zookeeper{}, err
	}

	if !auth.CheckPassword(zk.PasswordHash, password) {
		return zookeeper{}, errInvalidCredentials
	}
	return zk, nil
}
```

- And note: the same `Update` path now stores `COALESCE` results over an
  already-hashed value; if a PUT sends a new password, hash it in `Update`
  too:

```go
func (s *zookeeperService) Update(ctx context.Context, id int64, zku zookeeperUpdate) (zookeeper, error) {
	if zku.PasswordHash != nil {
		hash, err := auth.HashPassword(*zku.PasswordHash)
		if err != nil {
			return zookeeper{}, err
		}
		zku.PasswordHash = &hash
	}

	zk, err := s.repo.Update(ctx, id, zku)
	// ... the rest is unchanged
```

**3. `zookeepers_repository.go`: one new method.** Add next to `Get`:

```go
func (r *zookeeperRepository) GetByUsername(ctx context.Context, username string) (zookeeper, error) {
	var zk zookeeper
	err := r.pool.QueryRow(ctx,
		`SELECT id, username, password_hash, role, created_at, updated_at
		 FROM zookeepers WHERE username = $1`, username).
		Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role, &zk.CreatedAt, &zk.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return zk, nil
}
```

**4. `zookeepers_handler.go`: login endpoint + issued tokens.** Three edits:

- The handler struct and constructor change to carry the token config:

```go
type zookeeperHandler struct {
	svc         *zookeeperService
	tokenSecret []byte
	tokenTTL    time.Duration
}

func newZookeeperHandler(svc *zookeeperService, secret string, ttl time.Duration) *zookeeperHandler {
	return &zookeeperHandler{svc: svc, tokenSecret: []byte(secret), tokenTTL: ttl}
}
```

- Add the login method:

```go
func (h *zookeeperHandler) login(c *gin.Context) {
	var req struct {
		Username string `json:"username" binding:"required"`
		Password string `json:"password" binding:"required"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	zk, err := h.svc.Authenticate(c.Request.Context(), req.Username, req.Password)
	if err != nil {
		if errors.Is(err, errInvalidCredentials) {
			// Same message for unknown username and wrong password.
			c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	token, err := auth.IssueToken(h.tokenSecret, h.tokenTTL, zk.ID, zk.Username, zk.Role)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, gin.H{"token": token, "zookeeper": zk.toResponse()})
}
```

(and add `"zoo/internal/platform/auth"` to the file's import block).

### 4.5 Wire it: changes to main.go

Inside `run`, replace the zookeeper wiring block and the zookeeper routes group with:

```go
	zkRepo := newZookeeperRepository(pool)
	zkSvc := newZookeeperService(zkRepo)
	zkHandler := newZookeeperHandler(zkSvc, cfg.JWTSecret, cfg.TokenTTL)
```

and:

```go
	zk := router.Group("/api/v1/zookeepers")
	{
		zk.POST("", zkHandler.create)

		// Authenticated reads: this group's middleware runs before
		// each of its routes, after any group-level middleware from
		// the parent. Unauthenticated creation is Stage 8's problem.
		authed := zk.Group("", auth.AuthMiddleware([]byte(cfg.JWTSecret)))
		{
			authed.GET("", zkHandler.list)
			authed.GET("/:id", zkHandler.get)
			authed.PUT("/:id", zkHandler.update)
			authed.DELETE("/:id", zkHandler.delete)
		}
	}

	router.POST("/api/v1/login", zkHandler.login)
```

and extend `main.go`'s imports with `"zoo/internal/platform/auth"`.

### 4.6 Verify: the login flow

Bcrypt-hashed passwords do not match raw text, so any row you created in Stage 3 is now dead weight. Delete everything and start clean (the schema is fine; the rows are stale). Use `TRUNCATE ... RESTART IDENTITY`, not `DELETE`: a plain `DELETE` frees the rows but does not rewind the identity counter, so your next users would get ids 4 and 5 and every later stage's example ids would silently lie:

```bash
docker exec -it $(docker compose ps -q db) \
  psql -U zoo -d zoo -c 'TRUNCATE zookeepers RESTART IDENTITY'
```

Restart the server with JWT_SECRET exported (see above), then:

```bash
# create two zookeepers (open for now; Stage 8 closes this)
curl -s -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"giraffe-lane","role":"admin"}' > /dev/null
curl -s -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"sam","password":"elephant-road"}' > /dev/null

# no token -> 401
curl -i http://localhost:8080/api/v1/zookeepers
# HTTP/1.1 401 Unauthorized {"error":"missing or malformed Authorization header"}

# wrong password -> 401, same message as unknown username
curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"wrong"}'
# {"error":"invalid credentials"}

# successful login -> token
curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"giraffe-lane"}'
# {"token":"eyJhbGciOi...","zookeeper":{"id":1,"username":"maya","role":"admin","created_at":...}}

# store the token for later stages:
export TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"giraffe-lane"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')

# token unlocks reads
curl -i http://localhost:8080/api/v1/zookeepers -H "Authorization: Bearer $TOKEN"
# HTTP/1.1 200 OK [ ... ]
```

In the database, `password_hash` now starts with `$2a$...`: that is bcrypt.

---

## Stage 5: Restructure into `cmd/` + `internal/` (the layout stage)

Look at your root directory: four `.go` files in `package main`, plus `main.go` containing wiring, health, *and* animals. Still comfortable. Now imagine adding animals against the database (Stage 6), business routes (Stage 7), roles, tests. Every new idea would bloat the same flat directory, and any new engineer would have to read all of it.

Time to move. `go` makes this mechanical and safe - the compiler catches every broken import. Do not hand-edit; use the commands in order.

### 5.1 The moves

```bash
mkdir -p cmd/apiserver internal/animals internal/server internal/zookeepers

# zookeeper domain moves as a unit, one directory per domain - and the
# flat file names (a legacy of "one package main directory") lose their
# prefix, since the directory is now the namespace:
mv zookeepers_repository.go internal/zookeepers/repository.go
mv zookeepers_service.go   internal/zookeepers/service.go
mv zookeepers_handler.go   internal/zookeepers/handler.go

rm main.go
touch internal/server/router.go internal/animals/handler.go cmd/apiserver/main.go
```

Every moved file changes its package clause from `package main` to `package zookeepers` - that is what being *in* the directory means, not a separate importable name. The animals handlers from `main.go` become `internal/animals/handler.go` (full listing below), and the wiring in `main.go` splits across `internal/server/router.go` and `cmd/apiserver/main.go`.

### 5.2 The renames: flat names become exported names

Inside a package, names the package exports must be capitalized (Go's rule: `Foo` is exported, `foo` is package-private). The repository/service/handler structs are now used *by other packages* (`server`, `cmd/apiserver`), so they gain capitals and lose the redundant package prefix (nobody writes `zookeepers.ZookeeperRepository`; inside the package that stutter is noise).

| Was (package main) | Becomes (package zookeepers) |
|---|---|
| `zookeeper` (type) | `Zookeeper` |
| `zookeeperResponse` (type) | `Response` (moves to new `dto.go`) |
| `.toResponse()` (method) | `.Response()` |
| `zookeeperUpdate` (type) | `UpdateInput` |
| `zookeeperRepository` / `newZookeeperRepository` | `dbRepository` / `NewRepository` (the struct itself gets the lowercase "who am I" name; Stage 11's seam interface takes the `Repository` name) |
| `zookeeperService` / `newZookeeperService` | `Service` / `NewService` |
| `zookeeperHandler` / `newZookeeperHandler` | `Handler` / `NewHandler` |
| `errZookeeperNotFound` | `ErrNotFound` |
| `errZookeeperDuplicate` | `ErrDuplicateUsername` |
| `errZookeeperInvalid` | `ErrInvalidInput` |
| `errInvalidCredentials` | `ErrInvalidCredentials` (unchanged) |
| handler methods `create`, `list`, `get`, `update`, `delete`, `login`, `workload` | exported `Create`, `List`, `Get`, `Update`, `Delete`, `Login`, `Workload` (route mounting uses them, so they must be exported from the package) |
| `zoo/internal/platform/*` imports | unchanged |

(The `login` method already exists by this point - Stage 4 added it - so export it in the same pass; `Workload` arrives in Stage 8 and is exported from day one, since anything another package mounts must be.)

Apply them mechanically: in each of the three moved files, `package main` becomes `package zookeepers`, then every occurrence in the left column becomes the right one. Scope warning if you reach for sed/perl: the table renames **Go identifiers, not string contents** - `zookeeper not found`, the `"zookeeper"` JSON key in login's response, and similar literals must stay lowercase exactly as written, or the API's response shapes change under you. The helper `parseID` and `isUniqueViolation` stay lowercase: only the package needs them. (The platform files from Stage 2-4 were always under `internal/` - `internal/platform/...` - and are untouched by this stage; the story "flat files grow into internal domains" applies only to the `package main` files.)

The DTO split: create `internal/zookeepers/dto.go`:

```go
package zookeepers

import "time"

// Response is the wire shape of a zookeeper: everything the outside world
// is allowed to see. The password hash exists only inside this package.
type Response struct {
	ID        int64     `json:"id"`
	Username  string    `json:"username"`
	Role      string    `json:"role"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}
```

and delete the `zookeeperResponse` type and `toResponse` method from `handler.go`; add to `Zookeeper` in `repository.go` (or wherever you keep the domain struct - keep `Zookeeper` and its method together):

```go
// Response projects the domain type to its wire shape.
func (z Zookeeper) Response() Response {
	return Response{
		ID:        z.ID,
		Username:  z.Username,
		Role:      z.Role,
		CreatedAt: z.CreatedAt,
		UpdatedAt: z.UpdatedAt,
	}
}
```

The handlers now build `zk.Response()` instead of `zk.toResponse()`, and the list loop writes `resp[i] = zk.Response()`.

### 5.3 internal/animals: same shape, still in memory

The animals routes from Stage 1 become a real package, still backed by the in-memory slice (Stage 6 makes it database-backed; notice how much of this file survives that change).

`internal/animals/handler.go`:

```go
package animals

import (
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
)

// Animal is the domain type. LastFedAt and the primary keeper relationship
// arrive in Stage 6, when the data can actually outlive a process.
type Animal struct {
	ID        int64  `json:"id"`
	Name      string `json:"name"`
	Species   string `json:"species"`
	Enclosure string `json:"enclosure"`
}

// Handler serves animal routes. It holds the in-memory store; Stage 6
// empties this struct except for the repository it will depend on.
type Handler struct {
	animals []Animal
}

func NewHandler() *Handler {
	return &Handler{animals: []Animal{
		{ID: 1, Name: "Tembo", Species: "African bush elephant", Enclosure: "Savanna"},
		{ID: 2, Name: "Suki", Species: "Sumatran tiger", Enclosure: "Jungle"},
		{ID: 3, Name: "Biscuit", Species: "Red panda", Enclosure: "Forest"},
	}}
}

func (h *Handler) List(c *gin.Context) {
	c.JSON(http.StatusOK, h.animals)
}

func (h *Handler) Get(c *gin.Context) {
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "id must be an integer"})
		return
	}

	for _, a := range h.animals {
		if a.ID == id {
			c.JSON(http.StatusOK, a)
			return
		}
	}

	c.JSON(http.StatusNotFound, gin.H{"error": "animal not found"})
}

func (h *Handler) Create(c *gin.Context) {
	var newAnimal Animal
	if err := c.ShouldBindJSON(&newAnimal); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	newAnimal.ID = int64(len(h.animals) + 1)
	h.animals = append(h.animals, newAnimal)

	c.JSON(http.StatusCreated, newAnimal)
}
```

### 5.4 internal/server: the router

`main.go`'s route-wiring half becomes the server package. `cmd/apiserver/main.go`'s half is in 5.5.

`internal/server/router.go`:

```go
package server

import (
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/jackc/pgx/v5/pgxpool"

	"zoo/internal/animals"
	"zoo/internal/platform/auth"
	"zoo/internal/zookeepers"
)

// NewRouter builds the gin engine and mounts every domain's routes. It is
// the single map of the whole API; the domains know nothing about it.
func NewRouter(pool *pgxpool.Pool, secret string, zk *zookeepers.Handler, an *animals.Handler) *gin.Engine {
	router := gin.Default()

	router.GET("/healthz", healthHandler(pool))

	zkGroup := router.Group("/api/v1/zookeepers")
	{
		zkGroup.POST("", zk.Create)

		authed := zkGroup.Group("", auth.AuthMiddleware([]byte(secret)))
		{
			authed.GET("", zk.List)
			authed.GET("/:id", zk.Get)
			authed.PUT("/:id", zk.Update)
			authed.DELETE("/:id", zk.Delete)
		}
	}

	router.POST("/api/v1/login", zk.Login)

	anGroup := router.Group("/api/v1/animals")
	{
		anGroup.GET("", an.List)
		anGroup.GET("/:id", an.Get)
		anGroup.POST("", an.Create)
	}

	return router
}

func healthHandler(pool *pgxpool.Pool) gin.HandlerFunc {
	return func(c *gin.Context) {
		if err := pool.Ping(c.Request.Context()); err != nil {
			c.JSON(http.StatusServiceUnavailable, gin.H{"status": "degraded", "db": "down"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"status": "ok", "db": "up"})
	}
}
```

This file is where the handler *methods* got their capitalized names (`zk.Create`...): gin takes `func(c *gin.Context)` values, and method values like `zk.Create` bind the receiver - that is the idiomatic way to point gin at a method.

### 5.5 cmd/apiserver/main.go: the composition root

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"os"

	"zoo/internal/animals"
	"zoo/internal/platform/config"
	"zoo/internal/platform/database"
	"zoo/internal/server"
	"zoo/internal/zookeepers"
)

// Composition root: the only place in the codebase that knows how all the
// pieces are made and connected. Everything else receives its dependencies
// through constructors - plain function calls, no framework.
func main() {
	if err := run(); err != nil {
		slog.Error("startup failed", "error", err)
		os.Exit(1)
	}
}

func run() error {
	ctx := context.Background()

	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("load config: %w", err)
	}

	pool, err := database.NewPool(ctx, cfg.DatabaseURL)
	if err != nil {
		return fmt.Errorf("open database: %w", err)
	}
	defer pool.Close()

	zkHandler := zookeepers.NewHandler(
		zookeepers.NewService(zookeepers.NewRepository(pool)),
		cfg.JWTSecret, cfg.TokenTTL,
	)
	anHandler := animals.NewHandler()

	router := server.NewRouter(pool, cfg.JWTSecret, zkHandler, anHandler)

	if err := router.Run("localhost:" + cfg.Port); err != nil {
		return fmt.Errorf("run server: %w", err)
	}
	return nil
}
```

That chain - `NewRepository(pool)` into `NewService(repo)` into `NewHandler(svc, ...)` - is **dependency injection**. When you hear "DI framework" (wire, fx, dig), the idea is automating what this file does by hand; the Go convention is to simply do it here, in one place. Notice the line `zookeepers.NewService(zookeepers.NewRepository(pool))`: the service takes the repository as a parameter typed by the constructor's return (a pointer into the zookeepers package). Stage 11 loosens exactly this parameter to an interface, which is what makes the service testable with a fake.

`internal/` enforcement, concretely: try to import `zoo/internal/zookeepers` from a *different* module someday and the compiler refuses, by rule, regardless of build flags. That guarantee is why the layout works: your SQL, your secrets, your domain rules are all physically unreachable from outside.

### 5.6 The gate: build, then the regression script

```bash
go build ./...
go vet ./...
```

Both must be silent. If not, the compiler is your rename checklist - it reports the exact unresolved identifier each time.

Then run the full regression: this stage's success criterion is **the Stage 3.2 and Stage 4.6 curl flows behave exactly as they do in Stages 3 and 4** - the scripts themselves need one mechanical tweak (see below), but nothing that would betray a structural change. Fresh database first (your old rows predate bcrypt):

```bash
docker compose down -v
docker compose up -d
export JWT_SECRET="a-long-random-string-for-dev-only-not-a-real-secret"
go run ./cmd/migrate
go run ./cmd/apiserver
```

(`JWT_SECRET` must be exported before the migrate call: since Stage 4, `config.Load` requires it, and the migrate command loads that same config.)

Play both flows as written in Stages 3.2 and 4.6 - one known drift, explained: since Stage 4, zookeeper reads, PUT and DELETE sit behind the authed group, so the Stage 3.2 curls that ran open now need `-H "Authorization: Bearer $TOKEN"` (as Stage 4.6 shows) and POST is still open. Every status code and body shape must otherwise match what Stages 3 and 4 showed. If anything behaves differently beyond adding the token header, something in the moves went wrong - fix before Stage 6, which builds on this skeleton.

Current tree for reference:

```
.
├── cmd/apiserver/main.go
├── cmd/migrate/main.go
├── internal/
│   ├── animals/handler.go
│   ├── platform/config/config.go
│   ├── platform/database/pool.go
│   ├── platform/auth/{password,token,middleware}.go
│   ├── server/router.go
│   └── zookeepers/{repository,service,handler,dto}.go
├── migrations/00001_create_zookeepers.sql
├── docker-compose.yml
└── go.mod
```

---

## Stage 6: Animals against the database, with joins and filters

Now the animals get the same treatment: a migration, then a repository/service/handler trio in `internal/animals/` - replacing the in-memory handler from Stage 5. The new concepts are all on the SQL side: LEFT JOIN for the keeper relationship, `NULL` handling via pointer fields, and the `?keeper_id=` filter.

### 6.1 Migration: animals and the feed log

`migrations/00002_create_animals_and_feed_log.sql` (one file, both directions annotated, same convention as 00001):

```sql
-- +goose Up
CREATE TABLE animals (
    id                bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name              text NOT NULL,
    species           text NOT NULL,
    enclosure         text NOT NULL,
    primary_keeper_id bigint REFERENCES zookeepers(id) ON DELETE SET NULL,
    last_fed_at       timestamptz,
    created_at        timestamptz NOT NULL DEFAULT now(),
    updated_at        timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE feed_log (
    id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    animal_id bigint NOT NULL REFERENCES animals(id) ON DELETE CASCADE,
    keeper_id bigint NOT NULL REFERENCES zookeepers(id) ON DELETE RESTRICT,
    fed_at    timestamptz NOT NULL DEFAULT now(),
    note      text
);

CREATE INDEX feed_log_animal_idx ON feed_log (animal_id, fed_at DESC);

-- +goose Down
DROP TABLE feed_log;
DROP TABLE animals;
```

Three deliberate reference choices, all visible business decisions:

- `animals.primary_keeper_id ... ON DELETE SET NULL`: deleting a zookeeper means their animals become *unassigned*, not broken. NULLs are how "nobody is responsible yet" is represented.
- `feed_log.animal_id ... ON DELETE CASCADE`: deleting an animal deletes what happened to it (arguable; the wrap-up exercise suggests archiving instead).
- `feed_log.keeper_id ... ON DELETE RESTRICT`: you may not delete a zookeeper who still has feed history. Deleting maya mid-tutorial would corrupt auditability, so Postgres refuses - and Stage 8's delete path shows the 409 that results.

Run it (`JWT_SECRET` stays required by `config.Load`, which the migrate command sources too - keep it exported in this shell):

```bash
go run ./cmd/migrate
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo -c '\dt'
```

### 6.2 The animals package, database-backed

Four files now, and this shape is the template every future domain copies.

`internal/animals/dto.go`:

```go
package animals

import "time"

// keeperRef is the nested identity in Response: enough to display, never
// the full zookeeper (that belongs to the zookeepers domain).
type keeperRef struct {
	ID       int64  `json:"id"`
	Username string `json:"username"`
}

// Response is the wire shape. Optional values are pointers so they
// serialize as null and so a LEFT JOIN can scan NULL into them safely.
type Response struct {
	ID            int64      `json:"id"`
	Name          string     `json:"name"`
	Species       string     `json:"species"`
	Enclosure     string     `json:"enclosure"`
	LastFedAt     *time.Time `json:"last_fed_at"`
	CreatedAt     time.Time  `json:"created_at"`
	UpdatedAt     time.Time  `json:"updated_at"`
	PrimaryKeeper *keeperRef `json:"primary_keeper"`
}
```

> **NULLs are the most common Go+Postgres panic.** Scanning a database NULL into a non-pointer Go field panics ("converting NULL to string is unsupported"). Column-by-column, the fix is exactly one of: make the field a pointer (`*time.Time`), use `COALESCE(col, fallback)` in SQL, or use pgx's `pgtype` wrappers. This codebase standardizes on pointer fields for optional values, as above; you will see the trade repeated in the repository.

`internal/animals/repository.go`:

```go
package animals

import (
	"context"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

// Animal is the domain type. PrimaryKeeperID and LastFedAt are optional:
// pointer fields mirror nullable columns one to one, and the handler's
// Response projection relies on that (`nil` pointer serializes as null).
type Animal struct {
	ID              int64
	Name            string
	Species         string
	Enclosure       string
	PrimaryKeeperID *int64
	LastFedAt       *time.Time
	CreatedAt       time.Time
	UpdatedAt       time.Time
}

// updateInput carries optional PUT changes; nil means "not sent".
type updateInput struct {
	Name      *string
	Species   *string
	Enclosure *string
}

// dbRepository is the real, SQL-talking implementation. The
// lowercase "db" prefix is deliberate: Stage 11 introduces the service's
// interface and takes the name `Repository` for it, and the concrete
// struct keeps a name no caller ever needs to spell.
type dbRepository struct {
	pool *pgxpool.Pool
}

func NewRepository(pool *pgxpool.Pool) *dbRepository {
	return &dbRepository{pool: pool}
}

const selectColumns = `id, name, species, enclosure, primary_keeper_id, last_fed_at, created_at, updated_at`

func (r *dbRepository) Create(ctx context.Context, a Animal) (Animal, error) {
	row := r.pool.QueryRow(ctx,
		`INSERT INTO animals (name, species, enclosure)
		 VALUES ($1, $2, $3)
		 RETURNING `+selectColumns,
		a.Name, a.Species, a.Enclosure,
	)

	var created Animal
	err := row.Scan(&created.ID, &created.Name, &created.Species, &created.Enclosure,
		&created.PrimaryKeeperID, &created.LastFedAt, &created.CreatedAt, &created.UpdatedAt)
	if err != nil {
		return Animal{}, err
	}
	return created, nil
}

// List returns animals, optionally filtered to one primary keeper.
func (r *dbRepository) List(ctx context.Context, keeperID *int64) ([]Animal, error) {
	query := `SELECT ` + selectColumns + ` FROM animals`
	args := []any{}

	if keeperID != nil {
		query += ` WHERE primary_keeper_id = $1`
		args = append(args, *keeperID)
	}
	query += ` ORDER BY id`

	rows, err := r.pool.Query(ctx, query, args...)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	animals := []Animal{}
	for rows.Next() {
		a, err := scanAnimal(rows)
		if err != nil {
			return nil, err
		}
		animals = append(animals, a)
	}
	if err := rows.Err(); err != nil {
		return nil, err
	}
	return animals, nil
}

// Get returns one animal with its primary keeper's username joined in.
// The separate keeperUsername output makes the LEFT JOIN's nullability
// explicit; both NULL-able outputs are pointers.
//
// The column list is qualified (a.id, not id) because the query joins two
// tables that both have id and created_at; unqualified names would be
// ambiguous there ("column reference \"id\" is ambiguous"). Create and
// Update can keep the short list because their RETURNING only sees the
// animals table.
func (r *dbRepository) Get(ctx context.Context, id int64) (Animal, string, error) {
	var a Animal
	var keeperUsername *string
	err := r.pool.QueryRow(ctx,
		`SELECT a.id, a.name, a.species, a.enclosure, a.primary_keeper_id,
		        a.last_fed_at, a.created_at, a.updated_at, k.username
		 FROM animals a
		 LEFT JOIN zookeepers k ON k.id = a.primary_keeper_id
		 WHERE a.id = $1`, id).
		Scan(&a.ID, &a.Name, &a.Species, &a.Enclosure,
			&a.PrimaryKeeperID, &a.LastFedAt, &a.CreatedAt, &a.UpdatedAt,
			&keeperUsername)
	if err != nil {
		return Animal{}, "", err
	}
	return a, deref(keeperUsername), nil
}

func (r *dbRepository) Update(ctx context.Context, id int64, in updateInput) (Animal, error) {
	var updated Animal
	err := r.pool.QueryRow(ctx,
		`UPDATE animals
		 SET name       = COALESCE($1, name),
		     species    = COALESCE($2, species),
		     enclosure  = COALESCE($3, enclosure),
		     updated_at = now()
		 WHERE id = $4
		 RETURNING `+selectColumns,
		in.Name, in.Species, in.Enclosure, id).
		Scan(&updated.ID, &updated.Name, &updated.Species, &updated.Enclosure,
			&updated.PrimaryKeeperID, &updated.LastFedAt, &updated.CreatedAt, &updated.UpdatedAt)
	if err != nil {
		return Animal{}, err
	}
	return updated, nil
}

func (r *dbRepository) Delete(ctx context.Context, id int64) error {
	tag, err := r.pool.Exec(ctx, `DELETE FROM animals WHERE id = $1`, id)
	if err != nil {
		return err
	}
	if tag.RowsAffected() == 0 {
		return errAnimalNotFound
	}
	return nil
}

// scanAnimal scans the 8 animal columns of any row source. pgx.Rows
// implements the same Next/Scan interface as QueryRow, so one helper
// serves both List's loop and Get's single row without duplicating
// the Scan field order (which is a real maintenance hazard).
func scanAnimal(row pgxScanner) (Animal, error) {
	var a Animal
	if err := row.Scan(&a.ID, &a.Name, &a.Species, &a.Enclosure,
		&a.PrimaryKeeperID, &a.LastFedAt, &a.CreatedAt, &a.UpdatedAt); err != nil {
		return Animal{}, err
	}
	return a, nil
}
```

Two helper definitions go in the same file (or a tiny `internal/animals/util.go` if you prefer; the tutorial keeps them in `repository.go`):

```go
// pgxScanner is the smallest interface both pgx.Rows and pgx.Row satisfy
// for scanning. Defining interfaces this small, where you use them, is
// idiomatic Go (see Stage 11 for the same idea at the service boundary).
type pgxScanner interface {
	Scan(dest ...any) error
}

func deref(s *string) string {
	if s == nil {
		return ""
	}
	return *s
}
```

and near the top of `repository.go`, with the domain errors (service-side sentinels arrive in 6.3; this one lives with the repository for now, and Stage 9 formalizes the error pipeline):

```go
var errAnimalNotFound = errors.New("animal not found")
```

with `"errors"` added to that file's imports.

`internal/animals/service.go`:

```go
package animals

import "context"

// Service holds animal business rules. In Stage 6 it only delegates; the
// rules that justify this layer arrive in Stage 7 (assignment) and
// Stage 9 (feed history). A thin layer that is cheap to have is much less
// expensive than bolting one on later.
type Service struct {
	repo *dbRepository
}

func NewService(repo *dbRepository) *Service {
	return &Service{repo: repo}
}

func (s *Service) List(ctx context.Context, keeperID *int64) ([]Animal, error) {
	return s.repo.List(ctx, keeperID)
}

func (s *Service) Get(ctx context.Context, id int64) (Animal, string, error) {
	return s.repo.Get(ctx, id)
}

func (s *Service) Create(ctx context.Context, name, species, enclosure string) (Animal, error) {
	if name == "" || species == "" || enclosure == "" {
		return Animal{}, errAnimalInvalid
	}
	return s.repo.Create(ctx, Animal{Name: name, Species: species, Enclosure: enclosure})
}

func (s *Service) Update(ctx context.Context, id int64, in updateInput) (Animal, error) {
	return s.repo.Update(ctx, id, in)
}

func (s *Service) Delete(ctx context.Context, id int64) error {
	return s.repo.Delete(ctx, id)
}
```

`errAnimalInvalid` joins `errAnimalNotFound` in `repository.go`'s error block (Stage 9 moves both to where they are decided - but they exist so the handler can branch on them now):

```go
var (
	errAnimalNotFound = errors.New("animal not found")
	errAnimalInvalid  = errors.New("name, species and enclosure must be non-empty")
)
```

`internal/animals/handler.go` (complete new version; the in-memory handler is replaced):

```go
package animals

import (
	"errors"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/jackc/pgx/v5"
)

type Handler struct {
	svc *Service
}

func NewHandler(svc *Service) *Handler {
	return &Handler{svc: svc}
}

// parseID is the same helper the zookeepers package has, re-declared here.
// Two copies of a five-line helper is the correct amount of sharing for a
// two-domain project; factor into a shared package when it hurts more
// (Stage 9's error sweep shows what doing that properly looks like).
func parseID(c *gin.Context) (int64, bool) {
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "id must be an integer"})
		return 0, false
	}
	return id, true
}

func (h *Handler) List(c *gin.Context) {
	var keeperID *int64
	if raw := c.Query("keeper_id"); raw != "" {
		id, err := strconv.ParseInt(raw, 10, 64)
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": "keeper_id must be an integer"})
			return
		}
		keeperID = &id
	}

	animalsList, err := h.svc.List(c.Request.Context(), keeperID)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	resp := make([]Response, len(animalsList))
	for i, a := range animalsList {
		resp[i] = a.toResponse("") // list results skip the join
	}
	c.JSON(http.StatusOK, resp)
}

func (h *Handler) Get(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	a, keeperUsername, err := h.svc.Get(c.Request.Context(), id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "animal not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, a.toResponse(keeperUsername))
}

func (h *Handler) Create(c *gin.Context) {
	var req struct {
		Name      string `json:"name" binding:"required"`
		Species   string `json:"species" binding:"required"`
		Enclosure string `json:"enclosure" binding:"required"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	a, err := h.svc.Create(c.Request.Context(), req.Name, req.Species, req.Enclosure)
	if err != nil {
		if errors.Is(err, errAnimalInvalid) {
			c.JSON(http.StatusBadRequest, gin.H{"error": "name, species and enclosure are required"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusCreated, a.toResponse(""))
}

func (h *Handler) Update(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	var req struct {
		Name      *string `json:"name"`
		Species   *string `json:"species"`
		Enclosure *string `json:"enclosure"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	a, err := h.svc.Update(c.Request.Context(), id, updateInput{
		Name:      req.Name,
		Species:   req.Species,
		Enclosure: req.Enclosure,
	})
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "animal not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, a.toResponse(""))
}

func (h *Handler) Delete(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	if err := h.svc.Delete(c.Request.Context(), id); err != nil {
		if errors.Is(err, errAnimalNotFound) {
			c.JSON(http.StatusNotFound, gin.H{"error": "animal not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.Status(http.StatusNoContent)
}
```

The projection method joins the pieces; put it in `dto.go`:

```go
// toResponse builds the wire shape, attaching the primary keeper when the
// join found one. An empty username and nil PrimaryKeeperID both mean
// "unassigned", and both map to a null primary_keeper in JSON.
func (a Animal) toResponse(keeperUsername string) Response {
	resp := Response{
		ID:            a.ID,
		Name:          a.Name,
		Species:       a.Species,
		Enclosure:     a.Enclosure,
		LastFedAt:     a.LastFedAt,
		CreatedAt:     a.CreatedAt,
		UpdatedAt:     a.UpdatedAt,
	}
	if a.PrimaryKeeperID != nil && keeperUsername != "" {
		resp.PrimaryKeeper = &keeperRef{ID: *a.PrimaryKeeperID, Username: keeperUsername}
	}
	return resp
}
```

(Note the deliberate asymmetry with the zookeepers package, whose response is `Zookeeper.Response()`: animals need the joined username handed in, so an extra parameter earns its keep. Naming things by their actual shape beats forcing uniformity.)

### 6.3 Rewire

`cmd/apiserver/main.go`: replace the two animals lines

```go
	anHandler := animals.NewHandler()
```

with

```go
	anHandler := animals.NewHandler(
		animals.NewService(animals.NewRepository(pool)),
	)
```

`internal/server/router.go`: the animals group gains routes for `Update`/`Delete` and moves behind authentication, matching the zookeepers domain (mutations stay un-gated by role until Stage 8; that tightening is deliberate). The handler methods `an.Update` and `an.Delete` are new in Stage 6:

```go
	anGroup := router.Group("/api/v1/animals", auth.AuthMiddleware([]byte(secret)))
	{
		anGroup.GET("", an.List)
		anGroup.GET("/:id", an.Get)
		anGroup.POST("", an.Create)
		anGroup.PUT("/:id", an.Update)
		anGroup.DELETE("/:id", an.Delete)
	}
```

`router.go` already imports `auth`. (Variable naming note: a local variable named `animals` would legally shadow the imported `animals` package name - the compiler only objects once code in that scope tries to reference the package. Naming the group `anGroup` avoids the ambiguity entirely, and Stage 8's rewrite keeps that name.)

### 6.4 Verify: relationships live in the database

```bash
go build ./... && go vet ./...   # gate: silent
go run ./cmd/apiserver

# animals now start empty; create with maya's token (export TOKEN as in 4.6)
curl -s -X POST http://localhost:8080/api/v1/animals \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"Tembo","species":"African bush elephant","enclosure":"Savanna"}'
# {"id":1,"name":"Tembo",...,"last_fed_at":null,"primary_keeper":null}

# unauthenticated reads are now closed
curl -i http://localhost:8080/api/v1/animals
# HTTP/1.1 401

# assign maya as Tembo's primary keeper via SQL (the API for this is Stage 7)
docker exec -it $(docker compose ps -q db) \
  psql -U zoo -d zoo -c "UPDATE animals SET primary_keeper_id = 1 WHERE id = 1"

curl -s http://localhost:8080/api/v1/animals/1 -H "Authorization: Bearer $TOKEN"
# {"id":1,"primary_keeper":{"id":1,"username":"maya"},...}

# filter: only those maya keeps
curl -s "http://localhost:8080/api/v1/animals?keeper_id=1" -H "Authorization: Bearer $TOKEN"

# delete the keeper; the animal survives unassigned (ON DELETE SET NULL)
curl -s -X DELETE http://localhost:8080/api/v1/zookeepers/1 -H "Authorization: Bearer $TOKEN" -i
# HTTP/1.1 204
curl -s http://localhost:8080/api/v1/animals/1 -H "Authorization: Bearer $TOKEN"
# {"id":1,"primary_keeper":null,...}   <- the orphan case
```

That last transition - delete a zookeeper, watch the animal's keeper turn to `null` instead of crashing - is the LEFT JOIN and the `ON DELETE SET NULL` doing their job end-to-end.

---

## Stage 7: Business logic: assign a keeper, feed an animal, feed history

CRUD is bookkeeping; the routes in this stage are why the zookeeper domain and the animals domain exist at all:

- `PUT /api/v1/animals/:id/keeper` - assign (or clear) an animal's primary keeper, admin-side.
- `POST /api/v1/animals/:id/feed` - record a feeding; writes a feed-log row *and* updates the animal's `last_fed_at` in one transaction.
- `GET /api/v1/animals/:id/feed` - the feeding history (latest 20).

### 7.1 The one Go context to know: `context.Context`

Every database call your code makes takes a `context.Context` as its first parameter. It is Go's standard signal channel for "the caller stopped caring": when a client hangs up mid-request, gin closes the request's context, and every query that request started is cancelled by pgx itself. That is the payoff - cancellation you get for free by threading one value through.

Where does it come from? **From gin, once per request, in the handler: `c.Request.Context()`.** `gin.Context` - what handlers receive - is *gin's* per-request scratch pad (params, JSON, middleware state). It happens to be named the same thing as Go's cancellation context, and confusing the two is the single most common gin mistake. Rule: gin's context never leaves your handler; Go's `context.Context` is what you pass into `service` and `repository` calls.

### 7.2 Repository additions (internal/animals/repository.go)

Append to `repository.go`. New first, since assignment depends on one type guard:

```go
var (
	errAnimalNotFound  = errors.New("animal not found")
	errAnimalInvalid   = errors.New("name, species and enclosure must be non-empty")
	errKeeperNotFound  = errors.New("keeper not found")
	errKeeperHasFeeds  = errors.New("keeper has feed history and cannot be deleted")
)
```

(If you made this `var` block earlier with just the first two errors, this version replaces it; keep one `var` block per package for the errors.)

```go
// AssignKeeper sets or clears an animal's primary keeper. keeperID nil
// means "unassign". It returns the updated animal plus the (possibly
// empty) name of the keeper, so the caller can render the same response
// as Get without a second query.
func (r *dbRepository) AssignKeeper(ctx context.Context, animalID int64, keeperID *int64) (Animal, string, error) {
	if keeperID != nil {
		// Business rule, enforced before the write: only real zookeepers
		// can be assigned. (This reaches across domains in raw SQL; the
		// alternatives - asking the zookeepers service, or duplicating
		// the check - are discussed in the prose below.)
		var exists bool
		err := r.pool.QueryRow(ctx,
			`SELECT EXISTS (SELECT 1 FROM zookeepers WHERE id = $1)`, *keeperID).
			Scan(&exists)
		if err != nil {
			return Animal{}, "", err
		}
		if !exists {
			return Animal{}, "", errKeeperNotFound
		}
	}

	_, err := r.pool.Exec(ctx,
		`UPDATE animals
		 SET primary_keeper_id = $1, updated_at = now()
		 WHERE id = $2`,
		keeperID, animalID)
	if err != nil {
		return Animal{}, "", err
	}

	// Re-read via Get, which does the keeper join for us.
	return r.Get(ctx, animalID)
}

// FeedEntry is the domain shape of one feed_log row.
type FeedEntry struct {
	ID       int64
	AnimalID int64
	KeeperID int64
	FedAt    time.Time
	Note     *string
}

// Feed records a feeding transactionally: exactly either (feed row +
// updated last_fed_at) or nothing. The database transaction is the
// correctness tool here - without it, a crash between the two statements
// would log a feed and never stamp the animal.
func (r *dbRepository) Feed(ctx context.Context, animalID int64, keeperID int64, note string) (FeedEntry, error) {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return FeedEntry{}, err
	}
	// Standard Go transaction pattern: defer Rollback, Commit at the end.
	// Rollback after a Commit is a no-op, so this defer never hurts.
	defer tx.Rollback(ctx)

	var entry FeedEntry
	err = tx.QueryRow(ctx,
		`INSERT INTO feed_log (animal_id, keeper_id, note)
		 VALUES ($1, $2, $3)
		 RETURNING id, animal_id, keeper_id, fed_at, note`,
		animalID, keeperID, nullableString(note)).
		Scan(&entry.ID, &entry.AnimalID, &entry.KeeperID, &entry.FedAt, &entry.Note)
	if err != nil {
		return FeedEntry{}, err
	}

	_, err = tx.Exec(ctx,
		`UPDATE animals SET last_fed_at = $1, updated_at = now() WHERE id = $2`,
		entry.FedAt, animalID)
	if err != nil {
		return FeedEntry{}, err
	}

	if err := tx.Commit(ctx); err != nil {
		return FeedEntry{}, err
	}
	return entry, nil
}

// FeedHistory returns the 20 most recent log rows for one animal, joined
// to their keepers for display names.
func (r *dbRepository) FeedHistory(ctx context.Context, animalID int64) ([]FeedEntry, []string, error) {
	rows, err := r.pool.Query(ctx,
		`SELECT f.id, f.animal_id, f.keeper_id, f.fed_at, f.note, k.username
		 FROM feed_log f
		 JOIN zookeepers k ON k.id = f.keeper_id
		 WHERE f.animal_id = $1
		 ORDER BY f.fed_at DESC, f.id DESC
		 LIMIT 20`, animalID)
	if err != nil {
		return nil, nil, err
	}
	defer rows.Close()

	entries := []FeedEntry{}
	usernames := []string{}
	for rows.Next() {
		var e FeedEntry
		var keeperName string
		if err := rows.Scan(&e.ID, &e.AnimalID, &e.KeeperID, &e.FedAt, &e.Note, &keeperName); err != nil {
			return nil, nil, err
		}
		entries = append(entries, e)
		usernames = append(usernames, keeperName)
	}
	if err := rows.Err(); err != nil {
		return nil, nil, err
	}
	return entries, usernames, nil
}

// nullableString maps the Go empty string to SQL NULL, so an omitted
// feed note does not become "" in the log.
func nullableString(s string) any {
	if s == "" {
		return nil
	}
	return s
}
```

Add `"time"` to the import block if your edition of the file does not have it (the Stage 6 listing did).

Two design notes worth their prose:

- **`AssignKeeper` reading the zookeepers table directly.** Within one database, the honest choices are: (a) cross-domain SQL (done here, fast, couples the two domains' schema), (b) calling the zookeepers service (clean layering, more wiring), (c) trusting the FK and surfacing the violation as an error (fewest round trips, worse error message). Real teams pick per case; (a) with a comment is a perfectly defensible default, and the FK is still the final guard.
- **`defer tx.Rollback(ctx)`.** The Go transaction pattern you will see everywhere: begin; defer rollback; do work; return `tx.Commit(ctx)` (or map its error). If any early return happens first, the deferred rollback cleans up; after a successful commit, rollback is a documented no-op.

### 7.3 Service additions (internal/animals/service.go)

The rules: assign only real keepers (repository does that), feed only real animals with a real keeper identity (from the token, not the request - trust the caller's identity, never a body field), history for real animals.

These methods branch on `errors.Is` against `pgx.ErrNoRows` and on the FK helper defined below, so `internal/animals/service.go`'s import block becomes (it was only `context` since Stage 6):

```go
import (
	"context"
	"errors"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"
)
```

```go
func (s *Service) AssignKeeper(ctx context.Context, animalID int64, keeperID *int64) (Animal, string, error) {
	a, keeperUsername, err := s.repo.AssignKeeper(ctx, animalID, keeperID)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return Animal{}, "", errAnimalNotFound
		}
		return Animal{}, "", err
	}
	return a, keeperUsername, nil
}

// Feed records one feeding by the authenticated caller. keeperID comes
// from the token's claims, not the request body: identity travels in the
// token and is verified by the middleware, so the body never gets to say
// someone else did it.
func (s *Service) Feed(ctx context.Context, animalID int64, keeperID int64, note string) (FeedEntry, error) {
	// Unknown animal is 404 - the existence check is explicit, so the
	// foreign-key violation below can only ever mean "keeper vanished"
	// (a token outliving a deleted account; see the prose after this).
	if _, _, err := s.repo.Get(ctx, animalID); err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return FeedEntry{}, errAnimalNotFound
		}
		return FeedEntry{}, err
	}

	entry, err := s.repo.Feed(ctx, animalID, keeperID, note)
	if err != nil {
		if isFKViolation(err) {
			return FeedEntry{}, errKeeperNotFound
		}
		return FeedEntry{}, err
	}
	return entry, nil
}

func (s *Service) FeedHistory(ctx context.Context, animalID int64) ([]FeedEntry, []string, error) {
	// Unknown animal is 404, not an empty history.
	if _, _, err := s.repo.Get(ctx, animalID); err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return nil, nil, errAnimalNotFound
		}
		return nil, nil, err
	}
	return s.repo.FeedHistory(ctx, animalID)
}
```

Extend the file's imports and add the FK helper next to `isUniqueViolation` (which lives in the zookeepers package; this is its animals-domain twin - if the repetition starts to itch, that discomfort is exactly Stage 9's opening sentence):

```go
// isFKViolation reports whether err is Postgres's foreign-key violation
// (SQLSTATE 23503). Used to turn "no such keeper/animal, caught by the
// constraint" into a domain-appropriate 404.
func isFKViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23503"
}
```

### 7.4 Handler additions (internal/animals/handler.go)

Three methods on the existing `Handler`, plus their error-mapping helper. The file's import block gains `"zoo/internal/platform/auth"` (first use in this package). Note the `claims, ok := auth.ClaimsFrom(c)` shape: the middleware ran, so `ok` should always be true; the `if` is defensive against wiring mistakes (a route mounted outside the authed group), and it fails loudly rather than panicking.

```go
// AssignKeeper: PUT /api/v1/animals/:id/keeper, body {"keeper_id": 7}
// or {"keeper_id": null} to unassign.
func (h *Handler) AssignKeeper(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	var req struct {
		KeeperID *int64 `json:"keeper_id"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	// The middleware ran, so claims exist; a role check using them
	// arrives in Stage 8.
	_, ok = auth.ClaimsFrom(c)
	if !ok {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "no auth in context"})
		return
	}

	a, keeperUsername, err := h.svc.AssignKeeper(c.Request.Context(), id, req.KeeperID)
	if err != nil {
		h.writeError(c, err)
		return
	}

	c.JSON(http.StatusOK, a.toResponse(keeperUsername))
}

// Feed: POST /api/v1/animals/:id/feed, optional body {"note": "..."}
func (h *Handler) Feed(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	var req struct {
		Note string `json:"note"`
	}
	// An empty body must not fail: note is optional. ShouldBindJSON with
	// io.EOF (curl with no -d at all) is not a client error here.
	if c.Request.ContentLength > 0 {
		if err := c.ShouldBindJSON(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
	}

	claims, ok := auth.ClaimsFrom(c)
	if !ok {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "no auth in context"})
		return
	}

	entry, err := h.svc.Feed(c.Request.Context(), id, claims.ZookeeperID, req.Note)
	if err != nil {
		h.writeError(c, err)
		return
	}

	c.JSON(http.StatusCreated, gin.H{
		"id":        entry.ID,
		"animal_id": entry.AnimalID,
		"keeper_id": entry.KeeperID,
		"fed_at":    entry.FedAt,
		"note":      entry.Note,
	})
}

// FeedHistory: GET /api/v1/animals/:id/feed -> latest 20
func (h *Handler) FeedHistory(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	entries, usernames, err := h.svc.FeedHistory(c.Request.Context(), id)
	if err != nil {
		h.writeError(c, err)
		return
	}

	resp := make([]gin.H, len(entries))
	for i, e := range entries {
		resp[i] = gin.H{
			"id":        e.ID,
			"animal_id": e.AnimalID,
			"fed_at":    e.FedAt,
			"note":      e.Note,
			"keeper":    gin.H{"id": e.KeeperID, "username": usernames[i]},
		}
	}
	c.JSON(http.StatusOK, resp)
}
```

`writeError` maps the domain sentinels this domain has decided on, one place instead of five:

```go
// writeError centralizes sentinel -> status-code mapping for this domain.
// Stage 9 replaces it with the shared error pipeline; if you compare the
// two when you get there, notice this is the exact code being factored out.
func (h *Handler) writeError(c *gin.Context, err error) {
	switch {
	case errors.Is(err, errAnimalNotFound) || errors.Is(err, errKeeperNotFound):
		c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
	case errors.Is(err, errAnimalInvalid):
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	default:
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
	}
}
```

### 7.5 Mount the routes (internal/server/router.go)

Replace the animals group with:

```go
	anGroup := router.Group("/api/v1/animals", auth.AuthMiddleware([]byte(secret)))
	{
		anGroup.GET("", an.List)
		anGroup.GET("/:id", an.Get)
		anGroup.POST("", an.Create)
		anGroup.PUT("/:id", an.Update)
		anGroup.DELETE("/:id", an.Delete)
		anGroup.PUT("/:id/keeper", an.AssignKeeper)
		anGroup.POST("/:id/feed", an.Feed)
		anGroup.GET("/:id/feed", an.FeedHistory)
	}
```

### 7.6 Verify: the business rules, exercised

Stage 6's script deleted maya. Log in as sam (still there, username `sam`, password `elephant-road`) and keep that as `$TOKEN`:

```bash
go build ./... && go vet ./... && go run ./cmd/apiserver

export TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"sam","password":"elephant-road"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')

# sam is zookeeper id 2 (maya was 1). Assign sam to animal 1:
curl -s -X PUT http://localhost:8080/api/v1/animals/1/keeper \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"keeper_id":2}'
# {"id":1,...,"primary_keeper":{"id":2,"username":"sam"},...}

# unknown keeper -> 404, not a crash, not a silent no-op
curl -s -X PUT http://localhost:8080/api/v1/animals/1/keeper \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"keeper_id":999}'
# {"error":"keeper not found"}

# sam feeds Tembo
curl -s -X POST http://localhost:8080/api/v1/animals/1/feed \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"note":"morning hay and mineral block"}'
# {"id":1,"animal_id":1,"keeper_id":2,"fed_at":"2026-...","note":"morning hay and mineral block"}

# feed again with no body at all: optional note must not break
curl -s -X POST http://localhost:8080/api/v1/animals/1/feed -H "Authorization: Bearer $TOKEN"
# {"id":2,...,"note":null}

# last_fed_at is stamped (the transaction's second statement)
curl -s http://localhost:8080/api/v1/animals/1 -H "Authorization: Bearer $TOKEN"
# ... "last_fed_at":"2026-..." (matches the newest fed_at)

# history
curl -s http://localhost:8080/api/v1/animals/1/feed -H "Authorization: Bearer $TOKEN"
# [ {"id":2,...,"note":null}, {"id":1,...,"note":"morning hay and mineral block"} ]
# both entries carry "keeper":{"id":2,"username":"sam"}

# unknown animal's feed -> 404
curl -s http://localhost:8080/api/v1/animals/42/feed -H "Authorization: Bearer $TOKEN"
# {"error":"animal not found"}

# unassign: null
curl -s -X PUT http://localhost:8080/api/v1/animals/1/keeper \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"keeper_id":null}'
# primary_keeper back to null
```

> **A subtlety you just watched happen:** maya was deleted in Stage 6, yet a request bearing maya's token was still *accepted* by the server until she fed an animal and Postgres rejected the `keeper_id`. JWTs are self-contained: the server verifies the signature, not the account's existence. The feed endpoint returned "keeper not found" only because the FK caught it. Real systems add revocation (a checked token deny-list, very short expiry + refresh tokens, or a DB check per request); the wrap-up lists it as the first exercise. Know this trade exists before one bites you.

If you are feeling destructive: create a keeper, feed an animal as them, then delete them via `DELETE /api/v1/zookeepers/:id` - Postgres's RESTRICT turns the delete into an error the API reports as 500 today; remember the case, Stage 9 gives it its real status code (409 Conflict).

---

## Stage 8: Roles: admin gate, a seeded admin, and the workload summary

Until now any signed-in zookeeper could create accounts, rewrite animals, and reassign keepers. This stage closes that: writes are admin-only, and a fresh environment always has exactly one guaranteed admin, supplied by a seed migration - which finally closes the chicken-and-egg from Stage 4 ("who can create the first admin?").

### 8.1 The role gate: platform/auth/middleware.go

Append to `middleware.go`:

```go
// RequireRole returns middleware that admits only callers whose claims say
// role. It is the same closure-over-a-value pattern as AuthMiddleware: a
// factory with a parameter, so one function serves every role the domain
// ever grows. AuthMiddleware must run first (RequireRole reads the claims
// it stores), which Stage 8.4 handles by middleware ordering.
func RequireRole(role string) gin.HandlerFunc {
	return func(c *gin.Context) {
		claims, ok := ClaimsFrom(c)
		if !ok {
			// Wrong wiring (no auth middleware before this one), not a
			// client problem.
			c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "no auth in context"})
			return
		}

		if claims.Role != role {
			c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "requires role " + role})
			return
		}

		c.Next()
	}
}
```

One trade to say out loud: **claims can go stale.** The role inside a JWT is frozen at issue time. Promote a keeper to admin and their old token still says "keeper" until it expires (they must log in again); this is the reason Stage 7's comments said "no auth in context" defensively. A token issued to a *deleted* zookeeper still passes signature verification too (Stage 7's subtlety). Short TTLs are the standard mitigation; revocation is the exercise.

### 8.2 The seeded admin: migrations/00003_seed_admin.sql

You cannot type a bcrypt hash by hand; this one was generated with the snippet at the end of the section (hash of `zoo-admin-password`). The migration is idempotent via `ON CONFLICT`, so re-running against a database that already has an `admin` row is safe:

`migrations/00003_seed_admin.sql`:

```sql
-- +goose Up
-- Seed the documented bootstrap admin. In dev, log in with
--   username: admin       password: zoo-admin-password
-- and rotate this password per environment. (In this tutorial we leave
-- it fixed so the verify steps are reproducible.)
INSERT INTO zookeepers (username, password_hash, role)
VALUES ('admin', '$2a$10$fPKVrL96DdYq2Hllr6CmE.3PVqs7KPcS1.x9gUJQvYh6E8TESVpTy', 'admin')
ON CONFLICT (username) DO NOTHING;

-- +goose Down
DELETE FROM zookeepers WHERE username = 'admin';
```

If you were following along from scratch and wondering what the production move is: generate the hash in Go (a temporary `main.go` with four lines around `bcrypt.GenerateFromPassword`, as the comment in Stage 4's `password.go` suggests), paste it into the migration, and delete the tool. Committing the *plaintext* is the thing to never do; the hash is safe to commit, since deriving the plaintext from it is exactly what bcrypt is designed to prevent. One footgun to know while generating: the current bcrypt rejects inputs over 72 bytes with `ErrPasswordTooLong` rather than hashing a prefix, so over-long test passwords fail at hash time instead of quietly truncating.

```bash
go run ./cmd/migrate      # as ever: same shell must have JWT_SECRET exported
```

### 8.3 Workload summary: the aggregate

`GET /api/v1/zookeepers/:id/workload` answers "what is this keeper doing?" with one SQL round trip. This is also your first look at **scalar subqueries**: parentheses-wrapped queries that each produce one value, usable anywhere a value can go.

Add to `internal/zookeepers/repository.go`:

```go
// Workload returns one keeper's load: primary animals and feeds in the
// last 7 days, in a single round trip via scalar subqueries. Note the
// time math: now() is evaluated *in Postgres* with the database's clock,
// so the window is consistent no matter which host called it.
func (r *dbRepository) Workload(ctx context.Context, id int64) (animals int64, feeds int64, username string, err error) {
	err = r.pool.QueryRow(ctx,
		`SELECT username,
		       (SELECT count(*) FROM animals    WHERE primary_keeper_id = $1),
		       (SELECT count(*) FROM feed_log
		                             WHERE keeper_id = $1
		                             AND fed_at > now() - interval '7 days')
		FROM zookeepers WHERE id = $1`,
		id).
		Scan(&username, &animals, &feeds)
	return animals, feeds, username, err
}
```

Named result values (`animals int64, ...` in the signature) let the `Scan` targets line up with the returns without extra locals; Go allows either style, and both appear in real code.

Append to `internal/zookeepers/service.go` (and to `dto.go`, the projection types):

```go
// dto.go
// zookeeperRef is identity-only: enough for a summary, deliberately not
// the full Response (dates and role are noise in this context).
type zookeeperRef struct {
	ID       int64  `json:"id"`
	Username string `json:"username"`
}

// service.go
// Workload is the wire shape of the aggregate. The service decides what
// the reader needs (an identity, not a full zookeeper record) - that is
// read-shaping, and it is why this layer exists.
type Workload struct {
	Zookeeper      zookeeperRef `json:"zookeeper"`
	AnimalCount    int64        `json:"animal_count"`
	FeedsLast7Days int64        `json:"feeds_last_7_days"`
}

func (s *Service) Workload(ctx context.Context, id int64) (Workload, error) {
	animalCount, feeds, username, err := s.repo.Workload(ctx, id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return Workload{}, ErrNotFound
		}
		return Workload{}, err
	}

	return Workload{
		Zookeeper:      zookeeperRef{ID: id, Username: username},
		AnimalCount:    animalCount,
		FeedsLast7Days: feeds,
	}, nil
}
```

and add to `internal/zookeepers/handler.go`:

```go
func (h *Handler) Workload(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	wl, err := h.svc.Workload(c.Request.Context(), id)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, wl)
}
```

### 8.4 Mounting the gate: internal/server/router.go

The zookeeper group becomes (complete replacement):

```go
	authMiddleware := auth.AuthMiddleware([]byte(secret))
	requireAdmin := auth.RequireRole("admin")

	zkGroup := router.Group("/api/v1/zookeepers")
	{
		// Bootstrap: the seeded admin is the only creator of further
		// admins. Stage 3-7's open creation is closed. This route is
		// where you can see the ordering rule at its clearest: a route
		// outside the authed group must carry authMiddleware itself,
		// before requireAdmin.
		zkGroup.POST("", authMiddleware, requireAdmin, zk.Create)

		authed := zkGroup.Group("", authMiddleware)
		{
			authed.GET("", zk.List)
			authed.GET("/:id", zk.Get)
			// Inside the authed group, authMiddleware already ran;
			// only the extra check is added per route.
			authed.PUT("/:id", requireAdmin, zk.Update)
			authed.DELETE("/:id", requireAdmin, zk.Delete)
			authed.GET("/:id/workload", zk.Workload)
		}
	}
```

And the animals group (complete replacement):

```go
	anGroup := router.Group("/api/v1/animals", authMiddleware)
	{
		anGroup.GET("", an.List)
		anGroup.GET("/:id", an.Get)
		anGroup.POST("", requireAdmin, an.Create)
		anGroup.PUT("/:id", requireAdmin, an.Update)
		anGroup.DELETE("/:id", requireAdmin, an.Delete)
		// Reassigning keepers is admin business; feeding is everyone's.
		anGroup.PUT("/:id/keeper", requireAdmin, an.AssignKeeper)
		anGroup.POST("/:id/feed", an.Feed)
		anGroup.GET("/:id/feed", an.FeedHistory)
	}
```

Gin runs a route's middleware left to right; group middleware runs before route middleware. That is why `authed.PUT("/:id", requireAdmin, zk.Update)` needs no repeated `authMiddleware`: the group already carries it, and the route's list is *additional* on top ("group's first, then the route's"). The same reason `anGroup.POST("", requireAdmin, an.Create)` works - `anGroup` was built with `authMiddleware` in its constructor.

### 8.5 Verify: 403s where they belong

```bash
go build ./... && go vet ./... && go run ./cmd/apiserver

# login as the seeded admin
export ADMIN_TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"zoo-admin-password"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')

# sam is a keeper
export TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"sam","password":"elephant-road"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')

# sam may read...
curl -s http://localhost:8080/api/v1/zookeepers -H "Authorization: Bearer $TOKEN" | head -c 120
# [{"id":2,"username":"sam",...},{"id":3,"username":"admin",...}]  (ORDER BY id)

# ...but not create
curl -s -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"username":"newbie","password":"x2y3z4"}'
# {"error":"requires role admin"}                 (403)

# the admin can
curl -s -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"username":"newbie","password":"x2y3z4"}'
# {"id":4,"username":"newbie",...}   (sam=2 from Stage 4.6's TRUNCATE-restart; the seed took 3)

# same shape on animals: keeper attempts a create
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8080/api/v1/animals \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"Rex","species":"Cocker spaniel","enclosure":"Dog-gardens"}'
# 403. (Yes, Rex is a dog. Nobody said the zoo is strictly professional.)

curl -s -X POST http://localhost:8080/api/v1/animals \
  -H "Content-Type: application/json" -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"name":"Rex","species":"Cocker spaniel","enclosure":"Dog-gardens"}'
# {"id":2,...}

# feeding stays open to keepers:
curl -s -X POST http://localhost:8080/api/v1/animals/1/feed \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"note":"dinner"}' | head -c 60
# {"id":3,"animal_id":1,"keeper_id":2  ... (sam's feeds so far: 1 and 2)

# workload, aggregated in one query. Make sam Tembo's primary keeper again
# (Stage 7's last verify step unassigned it):
curl -s -X PUT http://localhost:8080/api/v1/animals/1/keeper \
  -H "Content-Type: application/json" -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"keeper_id":2}' > /dev/null

curl -s http://localhost:8080/api/v1/zookeepers/2/workload -H "Authorization: Bearer $ADMIN_TOKEN"
# {"zookeeper":{"id":2,"username":"sam"},"animal_count":1,"feeds_last_7_days":3}

curl -s http://localhost:8080/api/v1/zookeepers/99/workload -H "Authorization: Bearer $ADMIN_TOKEN"
# {"error":"zookeeper not found"}
```

---

## Stage 9: One error pipeline and structured logging

Count the places in your handler files that decide an HTTP status from an error: Stage 3 introduced one per method, Stage 7 added `writeError`. All of them do the same three steps, in slightly different shapes: find a *domain* error under the surface, look up its status, write JSON. That is duplication with consequences - two handlers can drift (one 404s, one 500s) for the same condition.

This stage factors the sweep into one pipeline. Also: replace gin's default console logger with `log/slog`, Go's built-in structured logging.

(A note on what you will see while it lands: parts of the file - handlers, middleware, verify outputs - carry the old flat `{"error":"..."}` envelope in one listing, the new envelope in the next. During Stage 9 itself the server is briefly inconsistent across routes; 9.5's verify is the first point at which everything below it is swept. Any earlier stage's expected output you re-check mid-stage may still show the flat shape - that is the point of a sweep.)

### 9.1 The error type: internal/platform/httperrors/errors.go

```go
package httperrors

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// AppError is an error that knows its HTTP meaning. Domains create these
// (package-level, so they compare with errors.Is); handlers do not think
// about status codes at all any more - they call Respond.
//
// Errors in Go are values that implement `error`; because AppError's
// method has a pointer receiver, *AppError is an error and errors.As can
// find it at the end of a wrapped chain (Stage 2's %w notes come home).
type AppError struct {
	Status  int    `json:"-"`
	Code    string `json:"code"`
	Message string `json:"message"`
}

func (e *AppError) Error() string {
	return e.Code + ": " + e.Message
}

// New builds an AppError. Code names the condition for machines
// ("not_found"), Message is the human-readable detail.
func New(status int, code, message string) *AppError {
	return &AppError{Status: status, Code: code, Message: message}
}

func BadRequest(message string) *AppError  { return New(http.StatusBadRequest, "invalid_request", message) }
func NotFound(message string) *AppError    { return New(http.StatusNotFound, "not_found", message) }
func Conflict(code, message string) *AppError {
	return New(http.StatusConflict, code, message)
}

// Respond writes err as the API's error envelope, once, in one place:
//
//	{"error": {"code": "not_found", "message": "animal 42 not found"}}
//
// The envelope (object with code and message) is worth the extra nesting
// over a bare "error": clients get a machine-readable code, the JSON
// shape never changes again, and adding a "details" field later is a
// change in one file.
func Respond(c *gin.Context, err error) {
	var appErr *AppError
	if errors.As(err, &appErr) {
		// The struct marshals itself using the json tags above ("-" hides
		// Status; the client has no business seeing it as a field).
		c.JSON(appErr.Status, gin.H{"error": appErr})
		return
	}

	// Anything we did not classify: never leak internals to the client,
	// but log the full error server-side (see requestLogger in Stage 9.3).
	slog.Error("unclassified error", "error", err)
	c.JSON(http.StatusInternalServerError, gin.H{"error": gin.H{"code": "internal", "message": "internal error"}})
}

// Abort is Respond for middleware: writes the envelope AND stops the
// chain. A plain Respond inside middleware would write "denied..." and
// then let the real handler run anyway; Abort is the one that must be
// used where the request is being rejected before the handler.
func Abort(c *gin.Context, err error) {
	Respond(c, err)
	c.Abort()
}
```

with the correct import block:

```go
import (
	"errors"
	"log/slog"
	"net/http"

	"github.com/gin-gonic/gin"
)
```

### 9.2 Domain errors become the new type

`internal/zookeepers/service.go` - the `var` block becomes (complete replacement; `errInvalidCredentials` included; sentinel variables keep their `Err` names so `errors.Is` calls elsewhere still work):

```go
var (
	ErrNotFound           = httperrors.New(http.StatusNotFound, "not_found", "zookeeper not found")
	ErrDuplicateUsername  = httperrors.New(http.StatusConflict, "duplicate_username", "username already taken")
	ErrInvalidInput       = httperrors.New(http.StatusBadRequest, "invalid_input", "username, password and role must be valid")
	ErrInvalidCredentials = httperrors.New(http.StatusUnauthorized, "invalid_credentials", "invalid credentials")
	ErrKeeperInUse        = httperrors.Conflict("keeper_in_use", "zookeeper has feed history and cannot be deleted")
)
```

which means `internal/zookeepers/service.go`'s import block gains two entries (with `"net/http"` going into the stdlib group and `"zoo/internal/platform/httperrors"` into the third):

```go
import (
	"context"
	"errors"
	"net/http"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"

	"zoo/internal/platform/httperrors"
)
```

`internal/animals/repository.go`:

```go
var (
	ErrNotFound       = httperrors.New(http.StatusNotFound, "not_found", "animal not found")
	ErrInvalidInput   = httperrors.New(http.StatusBadRequest, "invalid_input", "name, species and enclosure must be non-empty")
	ErrKeeperNotFound = httperrors.New(http.StatusNotFound, "not_found", "keeper not found")
)
```

(`internal/animals/repository.go` gains the same two entries in its import block: `"net/http"` and `"zoo/internal/platform/httperrors"`. It also **loses** `"errors"`: the error block no longer constructs with `errors.New`, and the compiler rejects the unused import. The zookeepers package's `service.go` keeps `errors` - its `isUniqueViolation`/`isFKViolation` helpers still use `errors.As`.)

(Animals' error block lives in `repository.go` from Stage 6; that was the informal arrangement - this is the moment it reads oddly enough to notice it lives with the repository and nothing else in that layer returns it. It stays; the wrap-up flags the alternative, `errors/`-package-per-domain.)

Two renames ripple: both domains had `errAnimalNotFound`/`errKeeperHasFeeds`-era names; every remaining use in each package updates to the `Err` names above. Specifically: `errAnimalNotFound` -> `ErrNotFound`, `errAnimalInvalid` -> `ErrInvalidInput`, `errKeeperNotFound` -> `ErrKeeperNotFound` in the animals package; `errKeeperHasFeeds` was defined but unused - it is replaced by `ErrKeeperInUse` (the naming honesty: the service decides, not the FK by itself).

And `zookeepers_service.go`'s `Delete` gains the 409 promise Stage 7 made. Since the FK helper `isFKViolation` currently lives only in the animals package (private), zookeepers grows its own copy right next to `isUniqueViolation`:

```go
// isFKViolation: same helper as in the animals package (Stage 7.3), now
// needed here too. Two private copies across two domains - the amount of
// sharing that starts to argue for a shared platform helper; an exercise
// in the wrap-up.
func isFKViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23503"
}

func (s *Service) Delete(ctx context.Context, id int64) error {
	err := s.repo.Delete(ctx, id)
	if err != nil && isFKViolation(err) {
		// feed_log.keeper_id is ON DELETE RESTRICT: the database's
		// auditability rule surfaces here as a proper conflict.
		return ErrKeeperInUse
	}
	return err
}
```

**Also required in this stage, or 404s become 500s:** the service methods that were pass-throughs since their stages never translate `pgx.ErrNoRows`; now that handlers no longer check it for them, both services' single-row reads and updates need it. The sweep in 9.3 collapses the *handlers'* checks - these service changes are the other half.

In `zookeepers/service.go`, `Get` and `Update` change from pass-throughs to (their complete new bodies; `Update` keeps its duplicate check):

```go
func (s *Service) Get(ctx context.Context, id int64) (Zookeeper, error) {
	zk, err := s.repo.Get(ctx, id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return Zookeeper{}, ErrNotFound
		}
		return Zookeeper{}, err
	}
	return zk, nil
}

func (s *Service) Update(ctx context.Context, id int64, in UpdateInput) (Zookeeper, error) {
	zk, err := s.repo.Update(ctx, id, in)
	if err != nil {
		if isUniqueViolation(err) {
			return Zookeeper{}, ErrDuplicateUsername
		}
		if errors.Is(err, pgx.ErrNoRows) {
			return Zookeeper{}, ErrNotFound
		}
		return Zookeeper{}, err
	}
	return zk, nil
}
```

(`service.go`'s import block gains nothing - the Stage 4 version already imports pgx. If you skipped Stage 4's pgx import, add it now.)

In `animals/service.go`, the same treatment for `Get` and `Update`:

```go
func (s *Service) Get(ctx context.Context, id int64) (Animal, string, error) {
	a, keeperUsername, err := s.repo.Get(ctx, id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return Animal{}, "", ErrNotFound
		}
		return Animal{}, "", err
	}
	return a, keeperUsername, nil
}

func (s *Service) Update(ctx context.Context, id int64, in updateInput) (Animal, error) {
	a, err := s.repo.Update(ctx, id, in)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return Animal{}, ErrNotFound
		}
		return Animal{}, err
	}
	return a, nil
}
```

### 9.3 The handler sweep

Every handler error branch collapses to one call. This file shows the complete zookeepers handler after the sweep - it is the shorter, simpler file; treat it as the exemplar and apply the same rules to the animals handler (whose per-method replacements are listed right after):

`internal/zookeepers/handler.go` (complete):

```go
package zookeepers

import (
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"

	"zoo/internal/platform/auth"
	"zoo/internal/platform/httperrors"
)

type Handler struct {
	svc         *Service
	tokenSecret []byte
	tokenTTL    time.Duration
}

func NewHandler(svc *Service, secret string, ttl time.Duration) *Handler {
	return &Handler{svc: svc, tokenSecret: []byte(secret), tokenTTL: ttl}
}

func parseID(c *gin.Context) (int64, bool) {
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		httperrors.Respond(c, httperrors.BadRequest("id must be an integer"))
		return 0, false
	}
	return id, true
}

func (h *Handler) Create(c *gin.Context) {
	var req struct {
		Username string `json:"username" binding:"required"`
		Password string `json:"password" binding:"required"`
		Role     string `json:"role"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		httperrors.Respond(c, httperrors.BadRequest("could not parse request body"))
		return
	}

	zk, err := h.svc.Create(c.Request.Context(), req.Username, req.Password, req.Role)
	if err != nil {
		httperrors.Respond(c, err)
		return
	}

	c.JSON(http.StatusCreated, zk.Response())
}

func (h *Handler) List(c *gin.Context) {
	zks, err := h.svc.List(c.Request.Context())
	if err != nil {
		httperrors.Respond(c, err)
		return
	}

	resp := make([]Response, len(zks))
	for i, zk := range zks {
		resp[i] = zk.Response()
	}
	c.JSON(http.StatusOK, resp)
}

func (h *Handler) Get(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	zk, err := h.svc.Get(c.Request.Context(), id)
	if err != nil {
		httperrors.Respond(c, err)
		return
	}

	c.JSON(http.StatusOK, zk.Response())
}

func (h *Handler) Update(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	var req struct {
		Username *string `json:"username"`
		Password *string `json:"password"`
		Role     *string `json:"role"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		httperrors.Respond(c, httperrors.BadRequest("could not parse request body"))
		return
	}

	in := UpdateInput{Username: req.Username, Role: req.Role}
	if req.Password != nil {
		raw := *req.Password
		in.PasswordHash = &raw
	}

	zk, err := h.svc.Update(c.Request.Context(), id, in)
	if err != nil {
		httperrors.Respond(c, err)
		return
	}

	c.JSON(http.StatusOK, zk.Response())
}

func (h *Handler) Delete(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	if err := h.svc.Delete(c.Request.Context(), id); err != nil {
		httperrors.Respond(c, err)
		return
	}

	c.Status(http.StatusNoContent)
}

func (h *Handler) Login(c *gin.Context) {
	var req struct {
		Username string `json:"username" binding:"required"`
		Password string `json:"password" binding:"required"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		httperrors.Respond(c, httperrors.BadRequest("could not parse request body"))
		return
	}

	zk, err := h.svc.Authenticate(c.Request.Context(), req.Username, req.Password)
	if err != nil {
		httperrors.Respond(c, err)
		return
	}

	token, err := auth.IssueToken(h.tokenSecret, h.tokenTTL, zk.ID, zk.Username, zk.Role)
	if err != nil {
		httperrors.Respond(c, err)
		return
	}

	c.JSON(http.StatusOK, gin.H{"token": token, "zookeeper": zk.Response()})
}

func (h *Handler) Workload(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	wl, err := h.svc.Workload(c.Request.Context(), id)
	if err != nil {
		httperrors.Respond(c, err)
		return
	}

	c.JSON(http.StatusOK, wl)
}
```

Compare with the Stage 3 file. No `errors`, no `pgx`, no status logic beyond success codes; `parseID` stays because it is input decoding, not error mapping. The service layer decides what "wrong" means; `httperrors.Respond` turns it into bytes.

Rules for the animals package (the same transformation, condensed):

- Every `if err != nil { <mapping> }` block in `handler.go` becomes `httperrors.Respond(c, err)` + `return`.
- Delete the now-obsolete `writeError` method and the sentinel-based branches inside the per-method handlers; the file's `errors` and `pgx` imports go with them (keep `strconv`, `net/http`, gin, auth).
- Each `c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})` from a failed `ShouldBindJSON` becomes `httperrors.Respond(c, httperrors.BadRequest("could not parse request body"))` - do not echo raw binding errors to clients (they can leak internal shape); Stage 9's logging shows where that detail belongs instead.
- In `service.go`'s `AssignKeeper`/`Feed`/`FeedHistory`, the `errors.Is(err, pgx.ErrNoRows)` branches return `ErrNotFound`; FK violation in `Feed` returns `ErrKeeperNotFound`.
- `internal/animals/handler.go` keeps its own `parseID` (Stage 6's duplication note, still accurate), now calling `httperrors.Respond`.

`service.go` in each package may still import `pgx` for these `ErrNoRows` checks: the boundary that changed is the handler's. If your style itches, note the alternative shape (repository translating `pgx.ErrNoRows` to domain errors) as a Stage 12 exercise.

**The middleware get the envelope too.** The auth layer still writes the old flat shape (`{"error": "invalid token"}`, `{"error": "requires role admin"}`), and Stage 9's promise is that *every* failure has the envelope - middleware failures are the most-hit failures of all. In `internal/platform/auth/middleware.go`, each rejection becomes an `AppError` plus `Abort` instead of `AbortWithStatusJSON` + literal:

```go
// AuthMiddleware, rejection paths:
		raw, ok := strings.CutPrefix(c.GetHeader("Authorization"), "Bearer ")
		if !ok {
			httperrors.Abort(c, httperrors.New(http.StatusUnauthorized,
				"missing_token", "missing or malformed Authorization header"))
			return
		}

		claims, err := VerifyToken(secret, raw)
		if err != nil {
			httperrors.Abort(c, httperrors.New(http.StatusUnauthorized,
				"invalid_token", "invalid token"))
			return
		}
```

```go
// RequireRole, rejection path (the "no auth in context" branch stays a
// plain 500 for now; Respond's fallback is exactly right for it):
		if claims.Role != role {
			httperrors.Abort(c, httperrors.Forbidden("requires role "+role))
			return
		}
```

which requires one more helper in `httperrors` (a tiny `forbidden` sibling of the others):

```go
func Forbidden(message string) *AppError { return New(http.StatusForbidden, "forbidden", message) }
```

`middleware.go`'s import block gains `"zoo/internal/platform/httperrors"`. To finish the sweep, the two defensive "no auth in context" branches in `internal/animals/handler.go` (the `if !ok` guards after `ClaimsFrom`) change from `c.JSON(http.StatusInternalServerError, gin.H{"error": "no auth in context"})` to:

```go
			httperrors.Respond(c, httperrors.New(http.StatusInternalServerError,
				"internal", "no auth in context"))
```

with `"zoo/internal/platform/httperrors"` added to that file's imports (the `animals` handler stops importing `gin.H`-shaped errors entirely at this point).

### 9.4 Structured logging and the router (internal/server/router.go)

In `NewRouter`, replace the line that builds the engine (`router := gin.Default()`) with `router := gin.New()` and follow it with the two additions shown below; the `healthHandler` mount and every route registration below it stay exactly as they are:

```go
func NewRouter(pool *pgxpool.Pool, secret string, zk *zookeepers.Handler, an *animals.Handler) *gin.Engine {
	router := gin.New()
	router.Use(gin.Recovery())
	router.Use(requestLogger())
	...
```

and add, above `NewRouter`:

```go
// requestLogger logs one structured line per request once it completes.
// The keys are stable (a schema to search on); handlers never log requests
// themselves.
func requestLogger() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()

		c.Next()

		slog.Info("http",
			"method", c.Request.Method,
			"path", c.FullPath(),
			"status", c.Writer.Status(),
			"duration", time.Since(start).Round(time.Millisecond),
		)
	}
}
```

with `"log/slog"` and `"time"` added to `router.go`'s imports. Note what changed in behavior: `gin.New()` + `Recovery()` + this logger is exactly what `gin.Default()` did, minus the console noise, with the format under your control.

### 9.5 Verify: the envelope everywhere

```bash
go build ./... && go vet ./... && go run ./cmd/apiserver

export ADMIN_TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"zoo-admin-password"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')

# not found, as an envelope (note: ErrNotFound's message is the static
# "animal not found"; putting the id in the message would mean building
# the AppError per call - Stage 12's exercises touch this trade):
curl -s http://localhost:8080/api/v1/animals/42 -H "Authorization: Bearer $ADMIN_TOKEN"
# {"error":{"code":"not_found","message":"animal not found"}}

# duplicate username, conflict envelope:
curl -s -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"username":"sam","password":"whatever1"}'
# {"error":{"code":"duplicate_username","message":"username already taken"}}

# the Stage 7 409 promise: newbie (id 4, created in Stage 8.5) feeds
# animal 1, which makes them undeletable by the RESTRICT rule
export NEWBIE=$(curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"newbie","password":"x2y3z4"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')
curl -s -X POST http://localhost:8080/api/v1/animals/1/feed \
  -H "Content-Type: application/json" -H "Authorization: Bearer $NEWBIE" \
  -d '{"note":"first shift"}' > /dev/null
curl -s -X DELETE http://localhost:8080/api/v1/zookeepers/4 -H "Authorization: Bearer $ADMIN_TOKEN"
# {"error":{"code":"keeper_in_use","message":"zookeeper has feed history and cannot be deleted"}}   (409)
# (id 4, not 3: id 3 is the seeded admin, who has no feed history and
# would be deletable - and destroying the tutorial's bootstrap admin
# mid-verify is not what anyone wants.)

# every failure route from earlier stages now returns the envelope; the
# request log (in the server's terminal) shows lines like:
# 2026/10/08 10:15:02 INFO http method=GET path=/api/v1/animals/:id status=404 duration=0s
```

(`time.Since(...).Round(time.Millisecond)` rounds sub-millisecond requests to `0s`, so most lines you see locally will read exactly like that one. The rounding is there so the log lines stay short, not to hide slowness - real latency shows up when it is milliseconds or more.)

(Note the exact log framing: `slog.Info` with the package's default handler prints the human-framed `date INFO msg key=value` form above, not the logfmt-structured `time=... level=...` form you may have seen in other projects - that comes from an explicitly installed `slog.NewTextHandler`, which the Stage 12 exercises make a natural next step. The keys are the structured part either way.)

---

## Stage 10: Graceful shutdown

Ctrl+C today kills the process wherever it is: mid-query, mid-response, mid-transaction. Graceful shutdown means: stop accepting new requests, finish the in-flight ones, close the pool, exit. This is the tutorial's only goroutine material, and it stays small on purpose.

The only genuinely concurrent pieces of Go you need here:

- **`go f(x)`** starts `f` running concurrently and returns immediately.
- **A channel** is how those concurrent pieces hand values to each other or wait for each other. Channels used here: the context's internal done-channel (no literals needed) and one buffered channel to hold the server's exit status.

`cmd/apiserver/main.go` (complete new version):

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"zoo/internal/animals"
	"zoo/internal/platform/config"
	"zoo/internal/platform/database"
	"zoo/internal/server"
	"zoo/internal/zookeepers"
)

func main() {
	if err := run(); err != nil {
		slog.Error("startup failed", "error", err)
		os.Exit(1)
	}
}

func run() error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("load config: %w", err)
	}

	pool, err := database.NewPool(ctx, cfg.DatabaseURL)
	if err != nil {
		return fmt.Errorf("open database: %w", err)
	}
	defer pool.Close()

	zkHandler := zookeepers.NewHandler(
		zookeepers.NewService(zookeepers.NewRepository(pool)),
		cfg.JWTSecret, cfg.TokenTTL,
	)
	anHandler := animals.NewHandler(animals.NewService(animals.NewRepository(pool)))

	router := server.NewRouter(pool, cfg.JWTSecret, zkHandler, anHandler)

	// The http.Server carries the timeouts that a bare gin Run never set
	// for you. Slowloris-proofing your read paths is this easy; skip it
	// and a single hung client pins a goroutine per open connection.
	srv := &http.Server{
		Addr:         "localhost:" + cfg.Port,
		Handler:      router,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// ListenAndServe blocks until the server stops; run it concurrently.
	errCh := make(chan error, 1)
	go func() {
		errCh <- srv.ListenAndServe()
	}()

	slog.Info("api server listening", "addr", srv.Addr)

	// Blocked here until SIGINT/SIGTERM (or the server errored early).
	// When the signal arrives, NotifyContext cancels ctx.
	select {
	case err := <-errCh:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			return fmt.Errorf("listen: %w", err)
		}
	case <-ctx.Done():
		slog.Info("signal received, shutting down")
	}

	// Finish in-flight requests, with a deadline that stops a hung one
	// from keeping the process alive forever.
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("shutdown: %w", err)
	}

	slog.Info("shutdown complete")
	return nil
}
```

Note the shape of the changes around it: `server.NewRouter` is unchanged (it returns the engine; the `http.Server` wraps it), and the pool closes *after* `Shutdown` because deferred calls run after the function's body - the ordering is `defer pool.Close()` running last, which is exactly right.

### 10.1 Verify: clean exits, both kinds

```bash
go build ./... && go run ./cmd/apiserver
# time=... msg="api server listening" addr=localhost:8080

# in another terminal, request while shutting down: kill the server
# (find the pid you just stopped it with; simplest: Ctrl+C in terminal 1)
# then watch terminal 1 for:
# time=... msg="signal received, shutting down"
# time=... msg="shutdown complete"
```

To try SIGTERM (the signal process managers actually send), background a *built binary*: `go run` wraps the compiled child in its own wrapper process, and killing the wrapper leaves the real server listening, which muddies the experiment.

```bash
go build -o /tmp/zoo-apiserver ./cmd/apiserver
/tmp/zoo-apiserver &
sleep 2 && kill -TERM %1        # %1 is the background job in this shell
# ... "signal received, shutting down" ... "shutdown complete"
# and the process exits 0, not from a panic
```

---

## Stage 11: Tests: fakes at the service seam, handlers over `httptest`

Go's testing toolkit is stdlib-only: `go test`, the `testing` package, `t.Run` for subtests, and table-driven tests as the standard pattern. The layering from Stages 5-9 pays off here: the service is testable without a database, the handler is testable without the whole router, and the repository gets exercised in verify scripts exactly because it is the thin, SQL-bearing layer we do not unit-test.

### 11.1 The seam: service takes an interface (internal/zookeepers/service.go)

To fake the repository for tests, the service must depend on *behavior*, not on the concrete `*dbRepository`. Move the type into `service.go` (complete new header for the file):

```go
package zookeepers

import (
	"context"
	"errors"
	"net/http"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"

	"zoo/internal/platform/auth"
	"zoo/internal/platform/httperrors"
)

// Repository is the interface the service consumes, defined in the file
// that consumes it (Go convention, unlike Java-style "the implementation
// defines the interface"). It is satisfied by *dbRepository, by test fakes,
// and by nothing else anyone needs to build.
type Repository interface {
	Create(ctx context.Context, zk Zookeeper) (Zookeeper, error)
	Get(ctx context.Context, id int64) (Zookeeper, error)
	GetByUsername(ctx context.Context, username string) (Zookeeper, error)
	List(ctx context.Context) ([]Zookeeper, error)
	Update(ctx context.Context, id int64, in UpdateInput) (Zookeeper, error)
	Delete(ctx context.Context, id int64) error
	Workload(ctx context.Context, id int64) (animals int64, feeds int64, username string, err error)
}

type Service struct {
	repo Repository
}

func NewService(repo Repository) *Service {
	return &Service{repo: repo}
}
```

The rest of the file is untouched - the header above already carries the one real change, the `Service` struct's field going from `*dbRepository` to `Repository`. `cmd/apiserver/main.go` still compiles unchanged: `*dbRepository` satisfies the interface without a cast, because the method sets match exactly. The unexported name on the concrete type works because callers never name it - they receive it from `NewRepository` and hand it to `NewService` in the same expression.

### 11.2 The fake: internal/zookeepers/service_test.go

```go
package zookeepers_test

import (
	"context"
	"errors"
	"strings"
	"testing"

	"zoo/internal/platform/httperrors"
	"zoo/internal/zookeepers"
)

// fakeRepository implements zookeepers.Repository with canned behavior.
// Each field is a lever a test case sets; only the methods the real tests
// exercise are meaningfully implemented.
type fakeRepository struct {
	created    []zookeepers.Zookeeper
	createErr  error

	byUsername map[string]zookeepers.Zookeeper // canned GetByUsername
}

func (f *fakeRepository) Create(_ context.Context, kz zookeepers.Zookeeper) (zookeepers.Zookeeper, error) {
	if f.createErr != nil {
		return zookeepers.Zookeeper{}, f.createErr
	}
	kz.ID = int64(len(f.created) + 1)
	f.created = append(f.created, kz)
	return kz, nil
}

func (f *fakeRepository) Get(_ context.Context, _ int64) (zookeepers.Zookeeper, error) {
	return zookeepers.Zookeeper{}, httperrors.NotFound("not used by these tests")
}

func (f *fakeRepository) GetByUsername(_ context.Context, username string) (zookeepers.Zookeeper, error) {
	if f.byUsername == nil {
		return zookeepers.Zookeeper{}, errors.New("not found in fake")
	}
	if kz, ok := f.byUsername[username]; ok {
		return kz, nil
	}
	return zookeepers.Zookeeper{}, errors.New("not found in fake")
}

func (f *fakeRepository) List(_ context.Context) ([]zookeepers.Zookeeper, error) {
	return nil, nil
}

func (f *fakeRepository) Update(_ context.Context, _ int64, _ zookeepers.UpdateInput) (zookeepers.Zookeeper, error) {
	return zookeepers.Zookeeper{}, nil
}

func (f *fakeRepository) Delete(_ context.Context, _ int64) error { return nil }

func (f *fakeRepository) Workload(_ context.Context, _ int64) (animals int64, feeds int64, username string, err error) {
	return 0, 0, "", nil
}

// Table-driven test: the table rows are behavior-under-test inputs, the
// single test body asserts them. Table-driven testing is the standard
// pattern for this shape of verification in Go: add a case, not a test.
func TestServiceCreate(t *testing.T) {
	tests := []struct {
		name       string
		username   string
		password   string
		role       string
		repoErr    error
		wantErr    error
		wantRole   string
		wantStored bool
	}{
		{
			name:       "empty role defaults to keeper",
			username:   "maya", password: "pw1", role: "",
			wantRole:   "keeper", wantStored: true,
		},
		{
			name:       "explicit admin stored",
			username:   "maya", password: "pw1", role: "admin",
			wantRole:   "admin", wantStored: true,
		},
		{
			name:       "empty password rejected",
			username:   "maya", password: "", role: "",
			wantErr:    zookeepers.ErrInvalidInput,
		},
		{
			name:       "bad role rejected",
			username:   "maya", password: "pw1", role: "visitor",
			wantErr:    zookeepers.ErrInvalidInput,
		},
		{
			name:       "duplicate from database surfaces as conflict",
			username:   "maya", password: "pw1", role: "",
			repoErr:    duplicateErr(),
			wantErr:    zookeepers.ErrDuplicateUsername,
		},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			repo := &fakeRepository{createErr: tc.repoErr}
			svc := zookeepers.NewService(repo)

			zk, err := svc.Create(context.Background(), tc.username, tc.password, tc.role)

			if tc.wantErr != nil {
				if !errors.Is(err, tc.wantErr) {
					t.Fatalf("Create() error = %v, want %v", err, tc.wantErr)
				}
				return
			}

			if err != nil {
				t.Fatalf("Create() unexpected error: %v", err)
			}
			if len(repo.created) != 1 || (tc.wantStored && !hasPasswordHash(repo.created[0])) {
				t.Fatalf("Create() stored = %+v, want one hashed row", repo.created)
			}
			if zk.Role != tc.wantRole {
				t.Fatalf("Create() role = %q, want %q", zk.Role, tc.wantRole)
			}
		})
	}
}

// hasPasswordHash confirms the service hashed the password: the stored
// value must look like a bcrypt hash, never the raw password.
func hasPasswordHash(zk zookeepers.Zookeeper) bool {
	return strings.HasPrefix(zk.PasswordHash, "$2a$")
}

func duplicateErr() error {
	// Note what this does NOT do: a real unique violation is a
	// *pgconn.PgError; the fake hands the sentinel straight back and the
	// service passes it through untouched. The pg-SQLSTATE mapping
	// (isUniqueViolation) is exercised by the Stage 3.2 curl scripts,
	// not here - fakes should not pretend to be Postgres.
	return zookeepers.ErrDuplicateUsername
}
```

Two things to learn from the failures your table should catch: the default-role case guards Stage 3's `if role == ""` behavior, so nobody "simplifies" it away; the duplicate case proves the pg error translation is reachable through the service call path (the fake trips the branch by handing the service the very error the real repository's unique-violation branch would produce - the translation itself is exercised by the curl scripts, and Stage 12's exercises suggest a repository-level test to pin it).

Note what a fake is allowed to be: it implements only what these tests call (`Get` even returns a stub error, because no test exercises `Get` here), and the compiler enforces that it stays a truthful `Repository`.

### 11.3 Handler tests over httptest: internal/zookeepers/handler_test.go

```go
package zookeepers_test

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/gin-gonic/gin"

	"zoo/internal/platform/auth"
	"zoo/internal/zookeepers"
)

// testSecret is shared by signToken, the middleware, and the handler. Real
// signed JWTs - stubbing middleware or faking the header would test the
// wiring, not the pipeline (a Stage-4-shaped mistake, learned by walking
// the token path in code).
const testSecret = "test-secret-for-ci-only"

func signToken(t *testing.T, id int64, username, role string) string {
	t.Helper()
	token, err := auth.IssueToken([]byte(testSecret), time.Hour, id, username, role)
	if err != nil {
		t.Fatalf("IssueToken: %v", err)
	}
	return token
}

// newTestEngine mirrors the routes the router mounts for this domain,
// gated the same way (list behind auth, create behind auth + admin,
// login open). If server/router.go's gating changes, keep this aligned;
// the wrap-up suggests a route-table test as the exercise that keeps
// them honest mechanically.
func newTestEngine(t *testing.T, svc *zookeepers.Service) *gin.Engine {
	t.Helper()
	gin.SetMode(gin.TestMode)

	router := gin.New()
	authMiddleware := auth.AuthMiddleware([]byte(testSecret))
	requireAdmin := auth.RequireRole("admin")
	h := zookeepers.NewHandler(svc, testSecret, time.Hour)

	group := router.Group("/api/v1/zookeepers")
	{
		group.POST("", authMiddleware, requireAdmin, h.Create)

		authed := group.Group("", authMiddleware)
		{
			authed.GET("", h.List)
			authed.GET("/:id", h.Get)
		}
	}

	// Mirroring the real router: login mounts at the API root, not inside
	// the zookeepers group.
	router.POST("/api/v1/login", h.Login)

	return router
}

func TestGetZookeeperAuth(t *testing.T) {
	tests := []struct {
		name       string
		authHeader string
		wantStatus int
	}{
		{
			name:       "missing token is 401",
			wantStatus: http.StatusUnauthorized,
		},
		{
			name:       "garbage token is 401",
			authHeader: "Bearer not-a-token",
			wantStatus: http.StatusUnauthorized,
		},
		{
			name:       "valid token gets in",
			authHeader: bearer(signToken(t, 1, "maya", "keeper")),
			wantStatus: http.StatusOK,
		},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			repo := &fakeRepository{}
			svc := zookeepers.NewService(repo)
			engine := newTestEngine(t, svc)

			req := httptest.NewRequest(http.MethodGet, "/api/v1/zookeepers", nil)
			if tc.authHeader != "" {
				req.Header.Set("Authorization", tc.authHeader)
			}
			rec := httptest.NewRecorder()
			engine.ServeHTTP(rec, req)

			if rec.Code != tc.wantStatus {
				t.Fatalf("status = %d, want %d; body = %q", rec.Code, tc.wantStatus, rec.Body.String())
			}
		})
	}
}

func TestCreateRequiresAdminRole(t *testing.T) {
	repo := &fakeRepository{byUsername: map[string]zookeepers.Zookeeper{
		"maya": {ID: 1, Username: "maya", PasswordHash: mustHash(t, "pw1"), Role: "keeper"},
	}}
	svc := zookeepers.NewService(repo)
	engine := newTestEngine(t, svc)

	req := httptest.NewRequest(http.MethodPost, "/api/v1/zookeepers",
		strings.NewReader(`{"username":"someone","password":"pw2"}`))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Authorization", bearer(signToken(t, 1, "maya", "keeper")))
	rec := httptest.NewRecorder()
	engine.ServeHTTP(rec, req)

	if rec.Code != http.StatusForbidden {
		t.Fatalf("status = %d, want %d (keeper cannot create); body = %q",
			rec.Code, http.StatusForbidden, rec.Body.String())
	}

	// the same request as admin succeeds, and the response never leaks the hash
	req = httptest.NewRequest(http.MethodPost, "/api/v1/zookeepers",
		strings.NewReader(`{"username":"someone","password":"pw2"}`))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Authorization", bearer(signToken(t, 2, "admin", "admin")))
	rec = httptest.NewRecorder()
	engine.ServeHTTP(rec, req)

	if rec.Code != http.StatusCreated {
		t.Fatalf("status = %d, want %d; body = %q", rec.Code, http.StatusCreated, rec.Body.String())
	}
	if strings.Contains(rec.Body.String(), "pw2") || strings.Contains(rec.Body.String(), "$2a") {
		t.Fatalf("password or hash leaked in response: %q", rec.Body.String())
	}
}

func bearer(token string) string { return "Bearer " + token }

// mustHash builds a real bcrypt hash for fixtures; a fake row that has
// never been hashed would fail Authenticate for the wrong reason.
func mustHash(t *testing.T, password string) string {
	t.Helper()
	hash, err := auth.HashPassword(password)
	if err != nil {
		t.Fatalf("HashPassword: %v", err)
	}
	return hash
}

// decodeJSON parses a handler response body in the tests that check shapes
// beyond status codes; unused in this file's assertions, which pin codes.
func decodeJSON(body string) (map[string]any, error) {
	var out map[string]any
	if err := json.Unmarshal([]byte(body), &out); err != nil {
		return nil, err
	}
	return out, nil
}
```

Token *expiry* specifically is unit-tested at the token boundary rather than through routing (a test token would otherwise have to be built with a negative TTL to be interesting; the routing tests pin status codes, the token tests pin the behavior):

`internal/platform/auth/token_test.go`:

```go
package auth_test

import (
	"strings"
	"testing"
	"time"

	"zoo/internal/platform/auth"
)

func TestIssueAndVerify(t *testing.T) {
	tests := []struct {
		name    string
		ttl     time.Duration
		wantErr error
	}{
		{name: "fresh token verifies", ttl: time.Minute},
		{name: "expired token does not verify", ttl: -time.Minute, wantErr: auth.ErrInvalidToken},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			token, err := auth.IssueToken([]byte("secret-for-tests"), tc.ttl, 7, "sam", "keeper")
			if err != nil {
				t.Fatalf("IssueToken: %v", err)
			}
			if !strings.Contains(token, ".") {
				t.Fatalf("token does not look like a JWT: %q", token)
			}

			_, err = auth.VerifyToken([]byte("secret-for-tests"), token)
			if tc.wantErr != nil && err == nil {
				t.Fatalf("VerifyToken should have failed for ttl %v", tc.ttl)
			}
			if tc.wantErr == nil && err != nil {
				t.Fatalf("VerifyToken: %v", err)
			}
		})
	}
}
```

### 11.4 Verify: `go test ./...`

```bash
go test ./...
# ok      zoo/internal/platform/auth      0.4s
# ok      zoo/internal/zookeepers         0.6s
# ?       zoo/cmd/apiserver               [no test files]
# ?       zoo/cmd/migrate                 [no test files]
# ?       zoo/internal/animals            [no test files]
# ?       zoo/internal/platform/config    [no test files]
# ?       zoo/internal/platform/database  [no test files]
# ?       zoo/internal/platform/httperrors [no test files]
# ?       zoo/internal/server             [no test files]
```

Add `-v` to see the subtests (`t.Run` rows print as `TestServiceCreate/duplicate_username_means_409`, `TestIssueAndVerify/expired_token_does_not_verify`, and so on). For one layer of confidence when you are not sure a test guards anything, break the code and re-run, e.g. temporarily remove `if role == "" { role = "keeper" }`; the "empty role defaults to keeper" case must fail. If it does not, the test is decoration. (Undo the break.)

---

## Stage 12: Makefile, README, and the honest recap

### 12.1 Makefile

`Makefile` (make is already on your Mac; targets use tabs, not spaces - a Makefile rite of passage):

```makefile
.PHONY: up down migrate run test

up:
	docker compose up -d

down:
	docker compose down -v

migrate:
	go run ./cmd/migrate

run:
	go run ./cmd/apiserver

test:
	go test ./...
```

With `JWT_SECRET` exported (Stage 4), the whole development loop is now: `make up`, `make migrate`, `make run`, and in another terminal `make test`. One asymmetry worth noticing rather than papering over: `make migrate` also sources `config.Load()`, which requires `JWT_SECRET` - a migration command needs no token-signing key, but it loads the config wholesale. The clean fix (per-binary config subsets, or a migrate-only loader) is left as an instinct check; for this tutorial, keeping `JWT_SECRET` exported before any `make` target is the honest note. Short muscle-memory commands matter: the less a workflow costs, the more often you run it.

### 12.2 The README someone else would need

`README.md` (this is also this repository's actual README):

```markdown
# Zoo Service

A Zoo management API: a remake of the Go official "Designing an API with
Gin" tutorial, production-shaped. Read `docs/tutorial.md` and build it
yourself, stage by stage.

## Stack

Go 1.27, gin, pgx (PostgreSQL), goose (SQL migrations), golang-jwt/v5 + bcrypt,
Docker Compose for the database.

## Quick start

1. export JWT_SECRET="a-long-random-string-for-dev-only-not-a-real-secret"
2. make up
3. make migrate
4. make run
5. Log in: POST /api/v1/login {"username":"admin","password":"zoo-admin-password"}
   (the seeded admin from migration 00003; dev credential, rotate per environment)

Full API documentation: the route table at the bottom of docs/tutorial.md.

## Layout

cmd/ - the two binaries (apiserver, migrate). internal/ - the private code:
one directory per domain (animals, zookeepers), plus platform (config,
database, auth, httperrors) and server (the router). See docs/tutorial.md
Stage 5 for why.
```

### 12.3 The honest recap of the layout choice

What you built, evaluated against the conventions you set out to learn:

- **What is near-universal**: `cmd/` (thin binary roots) and `internal/` (compiler-enforced privacy). The official Go module guidance says server projects should use exactly this combination.
- **What is community convention (project-layout)**: domain grouping (`internal/<domain>/{handler,service,repository}`), `platform/` for shared infrastructure, the migrations directory, the composition root pattern. Nothing official blesses these; lots of production codebases do something recognizable. The alternative is *layered* grouping (`internal/handlers`, `internal/services`, `internal/repositories` top-level). Try to name the code smell before reading: layered grouping puts one feature's change in three directories and makes each directory a mixed bag of unrelated concerns. Domain grouping localizes change; platform grouping exists precisely so domains do not import gin *and* pgx *and* goose knowledge into each other. Neither is dogma: small tools justifiably stay flat (Stage 1 was fine!), and monorepos with dozens of domains invent their own conventions.
- **What was skipped and why**: `pkg/` (nothing here is meant for import by other modules - and if it were, the guidance is to make it its own module), `api/` (OpenAPI specs; a fine exercise), `build/`, `scripts/`, `tools/`, `web/` (no CI story, no assets, an API-only tutorial). The project-layout repo is explicit that you should take what you need and delete the rest; that is what we did.

### 12.4 The finished API

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | /healthz | none | app + DB status |
| POST | /api/v1/login | none | returns token + zookeeper |
| POST | /api/v1/zookeepers | admin | create (role optional, defaults keeper) |
| GET | /api/v1/zookeepers | bearer | list |
| GET | /api/v1/zookeepers/:id | bearer | one |
| PUT | /api/v1/zookeepers/:id | admin | partial update (COALESCE) |
| DELETE | /api/v1/zookeepers/:id | admin | 204; 409 keeper_in_use if feed history exists |
| GET | /api/v1/zookeepers/:id/workload | bearer | scalar-subquery aggregate |
| POST | /api/v1/animals | admin | create |
| GET | /api/v1/animals | bearer | list; ?keeper_id= filter |
| GET | /api/v1/animals/:id | bearer | one, with primary_keeper join |
| PUT | /api/v1/animals/:id | admin | partial update |
| DELETE | /api/v1/animals/:id | admin | 204; cascades feed_log |
| PUT | /api/v1/animals/:id/keeper | admin | {keeper_id: 7 or null} |
| POST | /api/v1/animals/:id/feed | bearer | {note?} -> 201, transactional |
| GET | /api/v1/animals/:id/feed | bearer | latest 20 |

Errors everywhere are `{"error": {"code": ..., "message": ...}}`.

### 12.5 Where to go next (each is a real exercise, in rough order of difficulty)

1. **Token revocation or refresh**: the deleted-zookeeper-still-valids subtlety from Stage 7. Shorten the TTL and add `GET /api/v1/refresh`, or check account existence per request in `AuthMiddleware`.
2. **Pagination**: `?limit=`/`?cursor=` on both list routes; a cursor over `id` is the teaching version (offset pagination reads wrong at scale).
3. **Many-to-many care**: a `care_assignments` join table (many keepers per animal), which complicates every read; do it with the FK set you have now.
4. **Repository-level tests against a real Postgres** (dockerized, ephemeral database per run): this is where `isUniqueViolation` gets its automated test.
5. **A route-table parity test**: the `newTestEngine` in 11.3 mirrors the router by hand, with a comment asking to be kept honest. Replace the comment with a test: enumerate the method/path/middleware triples from `server/router.go` and assert the test engine mounts the same set, so gating drifts fail a build instead of a code review.
6. **Factor the shared helpers**: `isUniqueViolation` and `isFKViolation` exist as private twins in two packages, and Stage 5 promised not to share until it hurts. It now hurts: move them into `internal/platform/pgerrors/` (or similar) and delete the copies.
7. **Where errors live, take two**: the per-domain sentinels were consolidated in Stage 9, but repository files still carry domain errors (animals). Try the alternative shape - an `errors.go` per domain package, or the "repository translates `pgx.ErrNoRows` at the boundary and services never see raw pg errors" convention - and feel the trade in diff size.
8. **feed_log growth**: the table grows forever by design (auditability). Two real-world alternatives to try: a monthly rollup table fed by a scheduled job, or partitioning `feed_log` by `fed_at` (Postgres native partitioning); each changes `FeedHistory` in instructive ways.
9. **UUIDs**: change IDs to `uuid` columns; you will meet `pgtype.UUID` scanning and must decide `id::text` projections - Stage 2's callout becomes concrete.
10. **Dockerize the server itself** (multi-stage build + compose service for `apiserver`), and add `POST /api/v1/animals/:id/photo` backed by object storage.
11. **OpenAPI**: an `api/` directory with a spec generated or hand-written against the route table in 12.4.

If you only do one, do the pagination: it exercises every layer you built, touches SQL ordering, DTO shapes, and tests, and it is the first thing a reviewer will ask this API for.

---

You did the whole thing: an HTTP API with a real relational core, auth people actually use, one domain per directory, typed errors, structured logs, clean shutdown, and tests at the seam where they matter. The official tutorial you started from had one file and a slice. Look at the diff in your own head - that delta is "knowing Go" - and if you want more Go after this: [Effective Go](https://go.dev/doc/effective_go), the [Go module docs](https://go.dev/doc/modules/) on things like versioning and publishing, and reading real code (pgx's own source reads beautifully) will do the rest.