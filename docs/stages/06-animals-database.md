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
go run ./cmd/zoo migrate
docker exec $(docker compose ps -q db) psql -U zoo -d zoo -c '\dt'
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

> **NULLs are the most common Go+Postgres runtime failure.** Scanning a database NULL into a non-pointer Go field is unsupported; pgx v5 reports it as a scan error at query time (the error names the type that cannot receive NULL), not at compile time - which is exactly the kind that slips through quick tests. Column-by-column, the fix is exactly one of: make the field a pointer (`*time.Time`), use `COALESCE(col, fallback)` in SQL, or use pgx's `pgtype` wrappers. This codebase standardizes on pointer fields for optional values, as above; you will see the trade repeated in the repository.

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

// scanAnimal scans the 8 animal columns of a multi-row source: one Scan
// field order for every caller that needs the plain animal shape (Create
// and Update inline-scan their RETURNING rows; Get's joined scan has 9
// columns and stays inline too). pgx.Rows and pgx.Row share the same
// Scan call, which is why this one helper serves List's loop.
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

Three new Go ideas hide in those helpers:

- **An interface**, declared for the first time: `type pgxScanner interface { ... }` is a set of method signatures, and *any type that has them* satisfies it - no registration, no "implements" keyword, Go compares shapes (this is called structural typing). `pgx.Row` (single row) and `pgx.Rows` (iterator) both have `Scan(dest ...any) error`, so both are usable as a `pgxScanner`. `Scan(dest ...any)`: the `...` makes Scan *variadic* - it accepts any number of arguments, collected into a slice of `any` internally. And `any` is simply an alias for `interface{}` - "a value of any type" - renamed in Go 1.18 for readability.
- **`deref`**: the nil-pointer guard as a one-liner helper: if the pointer is nil, hand back the zero value `""`. `Get` calls it on the join's nullable `keeperUsername`, so "this animal has no keeper" arrives at the projection as an empty string rather than a nil pointer that every caller would have to check. (`List` does not need it: it runs no join, and passes `""` to the projection directly.)
- The `List` method (in the full listing above) uses both ideas again: **`args := []any{}`** builds a slice of "any typed values" used as query parameters, and `r.pool.Query(ctx, query, args...)` passes that slice with `args...` - the *spread operator*: expand the slice into individual arguments, the inverse of variadic collection. `args` starts empty, grows only if the filter is present, and pgx matches `$1` placeholders to slice positions - which is how one query serves both filtered and unfiltered shapes without SQL string-concatenating user text.

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

	"github.com/go-chi/chi/v5"
	"github.com/jackc/pgx/v5"

	"zoo/internal/platform/httpx"
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
func parseID(w http.ResponseWriter, r *http.Request) (int64, bool) {
	id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "id must be an integer"})
		return 0, false
	}
	return id, true
}

func (h *Handler) List(w http.ResponseWriter, r *http.Request) {
	var keeperID *int64
	if raw := r.URL.Query().Get("keeper_id"); raw != "" {
		id, err := strconv.ParseInt(raw, 10, 64)
		if err != nil {
			httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "keeper_id must be an integer"})
			return
		}
		keeperID = &id
	}

	animalsList, err := h.svc.List(r.Context(), keeperID)
	if err != nil {
		httpx.JSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	resp := make([]Response, len(animalsList))
	for i, a := range animalsList {
		resp[i] = a.toResponse("") // list results skip the join
	}
	httpx.JSON(w, http.StatusOK, resp)
}

func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	a, keeperUsername, err := h.svc.Get(r.Context(), id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			httpx.JSON(w, http.StatusNotFound, map[string]string{"error": "animal not found"})
			return
		}
		httpx.JSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	httpx.JSON(w, http.StatusOK, a.toResponse(keeperUsername))
}

func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Name      string `json:"name"`
		Species   string `json:"species"`
		Enclosure string `json:"enclosure"`
	}
	if err := httpx.Decode(r, &req); err != nil {
		httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "could not parse request body"})
		return
	}

	a, err := h.svc.Create(r.Context(), req.Name, req.Species, req.Enclosure)
	if err != nil {
		if errors.Is(err, errAnimalInvalid) {
			httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "name, species and enclosure are required"})
			return
		}
		httpx.JSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	httpx.JSON(w, http.StatusCreated, a.toResponse(""))
}

func (h *Handler) Update(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	var req struct {
		Name      *string `json:"name"`
		Species   *string `json:"species"`
		Enclosure *string `json:"enclosure"`
	}
	if err := httpx.Decode(r, &req); err != nil {
		httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "could not parse request body"})
		return
	}

	a, err := h.svc.Update(r.Context(), id, updateInput{
		Name:      req.Name,
		Species:   req.Species,
		Enclosure: req.Enclosure,
	})
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			httpx.JSON(w, http.StatusNotFound, map[string]string{"error": "animal not found"})
			return
		}
		httpx.JSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	httpx.JSON(w, http.StatusOK, a.toResponse(""))
}

func (h *Handler) Delete(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	if err := h.svc.Delete(r.Context(), id); err != nil {
		if errors.Is(err, errAnimalNotFound) {
			httpx.JSON(w, http.StatusNotFound, map[string]string{"error": "animal not found"})
			return
		}
		httpx.JSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	w.WriteHeader(http.StatusNoContent)
}
```

The projection method joins the pieces; put it in `dto.go`:

```go
// toResponse builds the wire shape, attaching the primary keeper when the
// join found one. An empty username and nil PrimaryKeeperID both mean
// "unassigned", and both map to a null primary_keeper in JSON.
func (a Animal) toResponse(keeperUsername string) Response {
	resp := Response{
		ID:        a.ID,
		Name:      a.Name,
		Species:   a.Species,
		Enclosure: a.Enclosure,
		LastFedAt: a.LastFedAt,
		CreatedAt: a.CreatedAt,
		UpdatedAt: a.UpdatedAt,
	}
	if a.PrimaryKeeperID != nil && keeperUsername != "" {
		resp.PrimaryKeeper = &keeperRef{ID: *a.PrimaryKeeperID, Username: keeperUsername}
	}
	return resp
}
```

(Note the deliberate asymmetry with the zookeepers package, whose response is `Zookeeper.Response()`: animals need the joined username handed in, so an extra parameter earns its keep. Naming things by their actual shape beats forcing uniformity.)

The List handler also introduces the last unexplored request reader, explained in place:

```go
	var keeperID *int64
	if raw := r.URL.Query().Get("keeper_id"); raw != "" {
		id, err := strconv.ParseInt(raw, 10, 64)
		if err != nil {
			httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "keeper_id must be an integer"})
			return
		}
		keeperID = &id
	}
```

- **`r.URL.Query().Get("keeper_id")`**: reads a *URL query parameter* (`?keeper_id=1`), the third reader after `chi.URLParam` (path) and JSON decoding (body). `r.URL.Query()` parses the raw query string into a `url.Values` (a `map[string][]string`, because a key may repeat), and `.Get` takes the first value; it returns `""` when absent, which is exactly the "no filter" case.
- **`keeperID = &id`**: taking the address of a local, one more time - `id` is a scoped if-declaration, so the `&id` pointer escapes into `keeperID` and survives the if (Go keeps the variable alive as long as the pointer to it exists; this is safe by design, not a dangling pointer). The pointer's nil-or-not *is* the filter's on/off switch downstream.

### 6.3 Rewire

`internal/cli/serve.go`: replace the animals wiring line

```go
	anHandler := animals.NewHandler()
```

with

```go
	anHandler := animals.NewHandler(
		animals.NewService(animals.NewRepository(pool)),
	)
```

That is the same `NewRepository` -> `NewService` -> `NewHandler` chain the zookeepers domain already had - one more reason Stage 5's composition root is the right place for it.

`internal/server/router.go`: the animals group gains routes for `Update`/`Delete` and moves behind authentication. Where Stage 5's group was a bare `router.Route`, this nests the same no-prefix `router.Group` the zookeepers reads use, so `router.Use(auth.AuthMiddleware(...))` applies to every route in the block and nothing else. The handler methods `an.Update` and `an.Delete` are new in Stage 6:

```go
		router.Route("/animals", func(router chi.Router) {
			router.Group(func(router chi.Router) {
				router.Use(auth.AuthMiddleware([]byte(secret)))
				router.Get("/", an.List)
				router.Get("/{id}", an.Get)
				router.Post("/", an.Create)
				router.Put("/{id}", an.Update)
				router.Delete("/{id}", an.Delete)
			})
		})
```

Every animals route now needs a token, `Create` included. The zookeepers group still leaves its create open until Stage 8, and there is no reason to copy that gap here. Mutations stay un-gated by *role* until Stage 8; that tightening is deliberate.

`router.go` already imports `auth`.

### 6.4 Verify: relationships live in the database

```bash
go build ./... && go vet ./...   # gate: silent
go run ./cmd/zoo serve

# animals now start empty; create with maya's token (export TOKEN as in 4.6)
curl -s -X POST http://localhost:8080/api/v1/animals \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"Tembo","species":"African bush elephant","enclosure":"Savanna"}'
# {"id":1,"name":"Tembo",...,"last_fed_at":null,"primary_keeper":null}

# unauthenticated reads are now closed
curl -i http://localhost:8080/api/v1/animals
# HTTP/1.1 401

# assign maya as Tembo's primary keeper via SQL (the API for this is Stage 7)
docker exec $(docker compose ps -q db) \
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

[Stage 5](05-restructure-internal.md)  |  [Overview](../tutorial.md)  |  [Stage 7](07-business-logic.md)
