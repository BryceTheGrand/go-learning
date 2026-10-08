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

- **`if _, _, err := s.repo.Get(...)`**: the blank identifier at work inside a multi-value unpack - Get returns three values, the caller wants *neither* result, only the error. Writing `_, _, err := ...` discards both results positionally; `if err := s.repo.Get(...)` alone would not compile, because Go does not let you silently drop multi-value results.
- **`nil, nil, err`**: three return slots, the error last, `nil` in both result positions (nil for the slice, empty string... no - for the `string` third slot the fail-return is `""`, shown above in `AssignKeeper`; `FeedHistory`'s two slice slots take `nil`). Each fail-return must name *every* slot; that is the price of the explicit convention, paid again here.

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

- **`switch {` with no condition** is Go's truth-test switch: each `case` is a boolean expression, evaluated top to bottom, first match wins. It reads like the if/else-if chain it replaces but keeps the classic switch shape, and it is the standard Go way to write multi-branch dispatch on error identity. The `default` case is the unmatched fall-through (the 500), which is why Stage 9 can promise "everything unknown becomes one status code unless a case says otherwise".
- Note the deliberate contrast with Stage 3's handlers: those wrote one `if errors.Is(...)` per handler; this file has three error-mapping methods per domain and the fan-out pays for a mapping function. When a file's error mapping starts repeating itself, extracting it like this is the move - and the sentence says so, because Stage 9 is about to do exactly that.

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

[Stage 6](06-animals-database.md)  ·  [Overview](../tutorial.md)  ·  [Stage 8](08-roles-workload.md)
