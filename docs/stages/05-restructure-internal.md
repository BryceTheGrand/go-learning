## Stage 5: Restructure into `cmd/` + `internal/` (the layout stage)

Look at your root directory: four `.go` files in `package main`, plus `main.go` containing wiring, health, *and* animals. Still comfortable. Now imagine adding animals against the database (Stage 6), business routes (Stage 7), roles, tests. Every new idea would bloat the same flat directory, and any new engineer would have to read all of it.

Time to move. `go` makes this mechanical and safe - the compiler catches every broken import. Do not hand-edit; use the commands in order.

### 5.1 The moves

```bash
mkdir -p internal/animals internal/server internal/zookeepers internal/platform/httpx

# zookeeper domain moves as a unit, one directory per domain - and the
# flat file names (a legacy of "one package main directory") lose their
# prefix, since the directory is now the namespace:
mv zookeepers_repository.go internal/zookeepers/repository.go
mv zookeepers_service.go   internal/zookeepers/service.go
mv zookeepers_handler.go   internal/zookeepers/handler.go

rm main.go
touch internal/animals/handler.go internal/platform/httpx/httpx.go \
      internal/server/router.go internal/cli/serve.go
```

Every moved file changes its package clause from `package main` to `package zookeepers` - that is what being *in* the directory means, not a separate importable name. The animals handlers from `main.go` become `internal/animals/handler.go` (full listing below); the two HTTP helpers in `main.go` become `internal/platform/httpx/httpx.go`; and the wiring in `main.go` splits across `internal/server/router.go` and `internal/cli/serve.go`.

That last one is the visible structural payoff of Stage 2's decision to build a CLI: the flat `main.go` that has been the API server since Stage 1 does not become *another* binary. It becomes the `serve` subcommand of the binary that already exists, and the root of the repository goes back to holding nothing but docs and configuration.

### 5.2 The renames: flat names become exported names

Inside a package, names the package exports must be capitalized (Go's rule: `Foo` is exported, `foo` is package-private). The repository/service/handler structs are now used *by other packages* (`server`, `cli`), so they gain capitals and lose the redundant package prefix (nobody writes `zookeepers.ZookeeperRepository`; inside the package that stutter is noise).

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
| handler methods `create`, `list`, `get`, `update`, `delete`, `login` | exported `Create`, `List`, `Get`, `Update`, `Delete`, `Login` (route mounting uses them, so they must be exported from the package) |
| `writeJSON(w, status, v)` | `httpx.JSON(w, status, v)` |
| `decodeJSON(r, v)` | `httpx.Decode(r, v)` |
| `zoo/internal/platform/*` imports | unchanged |

(The `login` method already exists by this point - Stage 4 added it - so export it in the same pass; anything another package mounts must be exported. `Workload` is deliberately not in that list: it does not exist yet. Stage 8 adds it already exported, as a matter of course, because by then the package has callers outside itself.)

Apply them mechanically: in each of the three moved files, `package main` becomes `package zookeepers`, then every occurrence in the left column becomes the right one. Scope warning if you reach for sed/perl: the table renames **Go identifiers, not string contents** - `zookeeper not found`, the `"zookeeper"` JSON key in login's response, and similar literals must stay lowercase exactly as written, or the API's response shapes change under you. The helpers `parseID` and `isUniqueViolation` stay lowercase: only the package needs them. (The platform files from Stage 2-4 were always under `internal/` - `internal/platform/...` - and are untouched by this stage; the story "flat files grow into internal domains" applies only to the `package main` files.)

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

The animals routes from Stage 1 become a real package, still backed by the in-memory slice (Stage 6 makes it database-backed; notice how much of this file survives that change). The code is Stage 1's, rehoused, so the pieces are brief - the only new idea is *being a package*, plus the handler signature you already know.

```go
package animals

import (
	"net/http"
	"strconv"

	"github.com/go-chi/chi/v5"

	"zoo/internal/platform/httpx"
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

func (h *Handler) List(w http.ResponseWriter, r *http.Request) {
	httpx.JSON(w, http.StatusOK, h.animals)
}

func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "id must be an integer"})
		return
	}

	for _, a := range h.animals {
		if a.ID == id {
			httpx.JSON(w, http.StatusOK, a)
			return
		}
	}

	httpx.JSON(w, http.StatusNotFound, map[string]string{"error": "animal not found"})
}

func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
	var newAnimal Animal
	if err := httpx.Decode(r, &newAnimal); err != nil {
		httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "could not parse request body"})
		return
	}

	newAnimal.ID = int64(len(h.animals) + 1)
	h.animals = append(h.animals, newAnimal)

	httpx.JSON(w, http.StatusCreated, newAnimal)
}
```

- The old package-level `var animals` slice has become a **field** on the handler (`h.animals`). That is the honest move this stage keeps making: global state becomes struct state, owned by one object, injectable. The seed data moved into the constructor literal for the same reason.
- The slice *still* has the Stage 1 data-race footnote - it is still shared mutable memory behind concurrent requests. It dies in Stage 6; no one writes locks for a store that is about to be deleted.
- `Get` and `Create` have the same bodies as Stage 1's handlers, with `w` and `r` in place of gin's context; if you want the line-by-line of either, Stage 1.3 still has it.

### 5.4 internal/platform/httpx: the two helpers become a package

Stage 1 wrote `writeJSON` at the bottom of `main.go`, and Stage 3 added `decodeJSON` beside it once the third handler needed one. Now three packages need them, which is the definition of "this is infrastructure".

`internal/platform/httpx/httpx.go`:

```go
package httpx

import (
	"encoding/json"
	"log/slog"
	"net/http"
)

// JSON writes v as the response body with the given status code.
func JSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.Error("write json response", "error", err)
	}
}

// Decode reads a JSON request body into v.
func Decode(r *http.Request, v any) error {
	return json.NewDecoder(r.Body).Decode(v)
}
```

- The names lost their `w`/`r` prefixes and the package supplies the meaning: `httpx.JSON(w, ...)` reads the same as `writeJSON(w, ...)` did, in fewer characters, and the `httpx.` prefix says exactly which layer this is.
- Note what is still **not** here: nothing about status codes chosen from errors, nothing about envelopes, nothing about auth. Those arrive in Stage 9, in a second file in this same package. What lives here is only the mechanical part of speaking HTTP: bytes out, bytes in.
- `Decode` is one line, and that is the honest size of it. Extracting a one-line helper is only worth it when the *name* is the value - here it is, because `httpx.Decode(r, &req)` tells a reader they are looking at a request body without making them parse `json.NewDecoder(r.Body)`.

### 5.5 internal/server: the router

`main.go`'s route-wiring half becomes the server package. The composition root is in 5.6.

`internal/server/router.go`:

```go
package server

import (
	"net/http"

	"github.com/go-chi/chi/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"zoo/internal/animals"
	"zoo/internal/platform/auth"
	"zoo/internal/platform/httpx"
	"zoo/internal/zookeepers"
)

// NewRouter builds the chi router and mounts every domain's routes. It is
// the single map of the whole API; the domains know nothing about it.
func NewRouter(pool *pgxpool.Pool, secret string, zk *zookeepers.Handler, an *animals.Handler) *chi.Mux {
	router := chi.NewRouter()

	router.Get("/healthz", healthHandler(pool))

	router.Route("/api/v1", func(router chi.Router) {
		router.Post("/login", zk.Login)

		router.Route("/zookeepers", func(router chi.Router) {
			router.Post("/", zk.Create)

			// Authenticated reads: this group's middleware runs before
			// each of its routes. Unauthenticated creation is Stage 8's
			// problem.
			router.Group(func(router chi.Router) {
				router.Use(auth.AuthMiddleware([]byte(secret)))
				router.Get("/", zk.List)
				router.Get("/{id}", zk.Get)
				router.Put("/{id}", zk.Update)
				router.Delete("/{id}", zk.Delete)
			})
		})

		router.Route("/animals", func(router chi.Router) {
			router.Get("/", an.List)
			router.Get("/{id}", an.Get)
			router.Post("/", an.Create)
		})
	})

	return router
}

func healthHandler(pool *pgxpool.Pool) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if err := pool.Ping(r.Context()); err != nil {
			httpx.JSON(w, http.StatusServiceUnavailable,
				map[string]string{"status": "degraded", "db": "down"})
			return
		}
		httpx.JSON(w, http.StatusOK, map[string]string{"status": "ok", "db": "up"})
	}
}
```

This file is where the handler *methods* got their capitalized names (`zk.Create`...): chi takes `http.HandlerFunc` values, and method values like `zk.Create` bind the receiver - that is the idiomatic way to point a router at a method.

Three parts worth their own moment before the full listing. First, the imports:

```go
import (
	"net/http"

	"github.com/go-chi/chi/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"zoo/internal/animals"
	"zoo/internal/platform/auth"
	"zoo/internal/platform/httpx"
	"zoo/internal/zookeepers"
)
```

This is the first file that imports several *own-domain* packages at once, and it shows off what the module path bought you: `zoo/internal/animals` imports as package `animals`, so the body says `animals.Handler` - the directory name is the package name, and each grouping convention (stdlib, third-party, this module) is doing its job. Note the router imports both domains plus `auth`, `httpx` and `pgxpool`: the router needs to *know about* everything it mounts, which is why the domains themselves know nothing about it (the comment below says this; the import list is the proof).

Second, the signature of `NewRouter(pool, secret, zk, an)` - every non-domain input the routes depend on arrives as a parameter: the pool for the health check, the secret for the middleware factory. That is deliberate: the router stays a pure assembler, constructing nothing but the router.

Third, the routing style, which is the gin-ism you will notice most:

- **`router.Route("/zookeepers", func(router chi.Router) { ... })`** is a *sub-router*: every route declared inside gets the prefix, and the closure receives a router scoped to it. Gin's `router.Group("/api/v1/zookeepers")` plus a `{...}` block did the same job; chi makes the scope explicit by handing you the object inside the function.
- **`router.Group(func(router chi.Router) { ... })`** is a group that adds *no* prefix and only middleware. Inside it, `router.Use(...)` runs for every route in the block and no others. That is the mechanism behind "the reads are authenticated and the create is not", with no second prefix to invent - gin needed `zkGroup.Group("", middleware)` for the same trick.
- **`router.Use` order is the execution order**, and a route declared outside a group (like `router.Post("/", zk.Create)` above) never sees that group's middleware. Stage 8 adds per-route middleware on top of this, and the ordering rule it follows is the same one.
- **`chi.Router`** rather than `*chi.Mux` in the closure signatures: `chi.Router` is the *interface* the router satisfies (registration methods plus `http.Handler`), which is what a sub-router hands you. Accept the interface in function parameters, as Go style prefers, and you never have to think about it again.

### 5.6 internal/cli/serve.go: the composition root

The flat `main.go` is gone. Its body survives almost word for word, as a subcommand.

```go
package cli

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"

	"github.com/spf13/cobra"

	"zoo/internal/animals"
	"zoo/internal/platform/config"
	"zoo/internal/platform/database"
	"zoo/internal/server"
	"zoo/internal/zookeepers"
)

func newServeCmd(a *app) *cobra.Command {
	return &cobra.Command{
		Use:   "serve",
		Short: "Run the API server",
		RunE: func(cmd *cobra.Command, _ []string) error {
			return runServer(cmd.Context(), a.cfg)
		},
	}
}

// runServer is the composition root: the only place in the codebase that knows
// how all the pieces are made and connected. Everything else receives its
// dependencies through constructors - plain function calls, no framework.
func runServer(ctx context.Context, cfg config.Config) error {
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

	slog.Info("api server listening", "addr", "localhost:"+cfg.Port)
	if err := http.ListenAndServe("localhost:"+cfg.Port, router); err != nil {
		return fmt.Errorf("run server: %w", err)
	}
	return nil
}
```

and `internal/cli/root.go` gains one line, in `newRootCmd`:

```go
	root.AddCommand(newServeCmd(a), newMigrateCmd(a))
```

That chain - `NewRepository(pool)` into `NewService(repo)` into `NewHandler(svc, ...)` - is **dependency injection**. When you hear "DI framework" (wire, fx, dig), the idea is automating what this file does by hand; the Go convention is to simply do it here, in one place. Notice the line `zookeepers.NewService(zookeepers.NewRepository(pool))`: the service takes the repository as a parameter typed by the constructor's return (a pointer into the zookeepers package). Stage 11 loosens exactly this parameter to an interface, which is what makes the service testable with a fake.

Two details of the move that are easy to get wrong:

- **`a.cfg` instead of `config.Load()`.** The configuration was already parsed in the root command's `PersistentPreRunE` (Stage 2.5), so `runServer` receives it as an argument and never calls `Load` itself. This is why `app` exists.
- **`cmd.Context()` instead of `context.Background()`.** The flat `main.go` had no parent context to inherit, so it started from `context.Background()`. A cobra command does: `cmd.Context()` is created by cobra for the life of the command and is the natural place for Stage 10 to hang the signal handling.

`internal/` enforcement, concretely: try to import `zoo/internal/zookeepers` from a *different* module someday and the compiler refuses, by rule, regardless of build flags. That guarantee is why the layout works: your SQL, your secrets, your domain rules are all physically unreachable from outside.

### 5.7 The gate: build, then the regression script

```bash
go build ./...
go vet ./...
go run ./cmd/zoo --help     # serve and migrate are both listed
```

Both build commands must be silent. If not, the compiler is your rename checklist - it reports the exact unresolved identifier each time.

A note on what `go build ./...` does and does not do, because it is easy to expect a binary: when the pattern matches more than one package, Go compiles them and **discards** the result - it is a type-check, not a build. So that command is silent and writes nothing, here or later, even though `cmd/zoo` is now the module's only `main` package. To actually get a binary you name the package: `go build ./cmd/zoo` writes `./zoo` into the current directory (which is why Stage 10's shutdown check uses `go build -o /tmp/zoo ./cmd/zoo` instead), and `go build -o /dev/null ./cmd/zoo` compiles without leaving anything behind. If you do leave a file called `zoo` in the project root, Stage 2.2 is where you read why that name was worth being careful about; since the config file is now named explicitly rather than searched for, it is untidy rather than dangerous, but the two notes are about the same three letters.

Then run the full regression: this stage's success criterion is **the Stage 3.2 and Stage 4.6 curl flows behave exactly as they do in Stages 3 and 4**, with one mechanical change to every command (see below), but nothing that would betray a structural change. Fresh database first (your old rows predate bcrypt):

```bash
docker compose down -v
docker compose up -d
export JWT_SECRET="a-long-random-string-for-dev-only-not-a-real-secret"
go run ./cmd/zoo migrate
go run ./cmd/zoo serve
```

(`JWT_SECRET` must be exported before the migrate call: since Stage 4, `config.Load` requires it, and the migrate command loads that same config through the root command's `PersistentPreRunE`.)

The one mechanical change: every `go run .` becomes `go run ./cmd/zoo serve`. Everything else - ports, paths, headers, bodies, status codes - is identical. The known drift from Stage 4 stands: zookeeper reads, PUT and DELETE sit behind the authed group, so the Stage 3.2 curls that ran open now need `-H "Authorization: Bearer $TOKEN"`, and POST is still open. If anything behaves differently beyond that, something in the moves went wrong - fix before Stage 6, which builds on this skeleton.

> **Replay them in order - 3.2 first, then 4.6 - and it matters more than it looks.** Stage 4.6 opens with a `TRUNCATE zookeepers RESTART IDENTITY`, which rewinds the id counter; Stage 3.2's script creates and deletes rows and deliberately ends with a duplicate-username attempt that fails. Run them in the written order and the seeded admin lands on id 3. (Migration 00003 does not exist yet - Stage 8 adds it - so that is a promise about where this is going rather than a claim about your tree right now; Stage 8.5 and Stage 9.5 are the sections that depend on it, when they say `newbie (id 4, created in Stage 8.5)`.) Run 4.6 first, or run 3.2 without 4.6's truncate, and the identity counter has drifted: the seed takes a different id, every id in those later examples is off by the same amount, and the mismatch will look like a bug in the code rather than in the order you ran two shell scripts. The general lesson is worth more than this instance - **example ids in a tutorial are a contract with the state of your database**, and the only reason `curl` examples are worth printing with ids at all is that identity columns are deterministic when you start from an empty table.

Current tree for reference:

```
.
|-- cmd/zoo/main.go
|-- internal/
|   |-- animals/handler.go
|   |-- cli/{root,serve,migrate}.go
|   |-- platform/config/config.go
|   |-- platform/database/pool.go
|   |-- platform/httpx/httpx.go
|   |-- platform/logging/logging.go
|   |-- platform/auth/{password,token,middleware}.go
|   |-- server/router.go
|   `-- zookeepers/{repository,service,handler,dto}.go
|-- migrations/00001_create_zookeepers.sql
|-- docker-compose.yml
`-- go.mod
```

---

[Stage 4](04-auth-bcrypt-jwt.md)  |  [Overview](../tutorial.md)  |  [Stage 6](06-animals-database.md)
