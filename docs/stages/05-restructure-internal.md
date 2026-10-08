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

The animals routes from Stage 1 become a real package, still backed by the in-memory slice (Stage 6 makes it database-backed; notice how much of this file survives that change). The code is Stage 1's, rehoused, so the pieces are brief - the only new idea is *being a package*.

#### Package clause, type, constructor

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
```

- The old package-level `var animals` slice has become a **field** on the handler (`h.animals`). That is the honest move this stage keeps making: global state becomes struct state, owned by one object, injectable. The seed data moved into the constructor literal for the same reason.
- The slice *still* has the Stage 1 data-race footnote - it is still shared mutable memory behind concurrent requests. It dies in Stage 6; no one writes locks for a store that is about to be deleted.

#### The methods: Stage 1's handlers, rehoused

```go
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

These three are the Stage 1 handlers exactly, with `animals` becoming `h.animals` and the capitalized method names (5.2's rule). If you want the line-by-line of any of them, Stage 1.3 still has it.

#### The complete internal/animals/handler.go

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

Cross-check your assembled version against this; Stage 6 replaces every method body but keeps these names.

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

Two parts worth their own moment before the full listing. First, the imports:

```go
import (
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/jackc/pgx/v5/pgxpool"

	"zoo/internal/animals"
	"zoo/internal/platform/auth"
	"zoo/internal/zookeepers"
)
```

This is the first file that imports several *own-domain* packages at once, and it shows off what the module path bought you: `zoo/internal/animals` imports as package `animals`, so the body says `animals.Handler` - the directory name is the package name, and each grouping convention (stdlib, third-party, this module) is doing its job. Note the router imports both domains plus `auth` and `pgxpool`: the router needs to *know about* everything it mounts, which is why the domains themselves know nothing about it (the comment below says this; the import list is the proof).

Second, the signature of `NewRouter(pool, secret, zk, an)` - every non-domain input the routes depend on arrives as a parameter: the pool for the health check, the secret for the middleware factory. That is deliberate: the router stays a pure assembler, constructing nothing but the engine.

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

[Stage 4](04-auth-bcrypt-jwt.md)  ·  [Overview](../tutorial.md)  ·  [Stage 6](06-animals-database.md)
