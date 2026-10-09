## Stage 7: Business logic: assign a keeper, feed an animal, feed history

CRUD is bookkeeping; the routes in this stage are why the zookeeper domain and the animals domain exist at all:

- `PUT /api/v1/animals/{id}/keeper` - assign (or clear) an animal's primary keeper, admin-side.
- `POST /api/v1/animals/{id}/feed` - record a feeding; writes a feed-log row *and* updates the animal's `last_fed_at` in one transaction.
- `GET /api/v1/animals/{id}/feed` - the feeding history (latest 20).

### 7.1 The one Go context to know: `context.Context`

Every database call your code makes takes a `context.Context` as its first parameter. It is Go's standard signal channel for "the caller stopped caring": when a client hangs up mid-request, the standard library's HTTP server cancels that request's context, and every query the request started is cancelled by pgx itself. That is the payoff - cancellation you get for free by threading one value through.

Where does it come from? **From `net/http`, once per request, in the handler: `r.Context()`.** Every `*http.Request` carries a `context.Context` of its own, created by the server when it accepted the connection and cancelled the moment the client disconnects or the handler returns. chi adds nothing here: it does not wrap the request in an object of its own. A chi handler receives the same `*http.Request` the standard library built, so `r.Context()` is Go's context, full stop.

That last sentence is worth dwelling on, because it is one of the things this port buys. The framework this tutorial replaced had two different things called "context": gin's `*gin.Context`, the per-request scratch pad handlers receive (params, JSON, middleware state), and Go's `context.Context`, the cancellation channel you pass into `service` and `repository` layers. They shared a name and nothing else, and confusing them was the single most common gin mistake. chi has no such object. There is exactly one context type in this codebase, it is the standard library's, and `r.Context()` is the only way to reach it.

Rule: the request's context never leaves your handler. You read `r.Context()` there and hand it to the `service` and `repository` calls; nothing below the handler ever sees an `*http.Request`.

### 7.2 Repository additions (internal/animals/repository.go)

Append to `repository.go`. New first, since assignment depends on one type guard:

```go
var (
	errAnimalNotFound = errors.New("animal not found")
	errAnimalInvalid  = errors.New("name, species and enclosure must be non-empty")
	errKeeperNotFound = errors.New("keeper not found")
	errKeeperHasFeeds = errors.New("keeper has feed history and cannot be deleted")
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

The import block already has `"time"` in it (Stage 6 listed it for the `Animal` struct's timestamp fields, and `FeedEntry.FedAt` is the same type), so this addition needs no new import.

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
- **`nil, nil, err`**: three return slots to fill, so a failure has to name all three - the error last, and a zero value in each result position. `FeedHistory` returns `([]FeedEntry, []string, error)`, so its fail-return is `nil, nil, err`; `AssignKeeper` returns `(Animal, string, error)`, so its fail-return is the *zero value* `Animal{}`, `""`, `err` (shown above). Getting this wrong in the other direction - returning a real value alongside an error - is the mistake the convention exists to prevent: a caller who checks the error must never also have to wonder whether the value means anything. Each fail-return names *every* slot, and that is the price of the explicit convention, paid again here.

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

Three methods on the existing `Handler`, plus their error-mapping helper. The file's import block gains `"zoo/internal/platform/auth"` (first use in this package). Note the `claims, ok := auth.ClaimsFrom(r.Context())` shape: the middleware ran, so `ok` should always be true; the `if` is defensive against wiring mistakes (a route mounted outside the authed group), and it fails loudly rather than panicking.

```go
// AssignKeeper: PUT /api/v1/animals/{id}/keeper, body {"keeper_id": 7}
// or {"keeper_id": null} to unassign.
func (h *Handler) AssignKeeper(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	var req struct {
		KeeperID *int64 `json:"keeper_id"`
	}
	if err := httpx.Decode(r, &req); err != nil {
		httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "could not parse request body"})
		return
	}

	// The middleware ran, so claims exist; a role check using them
	// arrives in Stage 8.
	_, ok = auth.ClaimsFrom(r.Context())
	if !ok {
		httpx.JSON(w, http.StatusInternalServerError, map[string]string{"error": "no auth in context"})
		return
	}

	a, keeperUsername, err := h.svc.AssignKeeper(r.Context(), id, req.KeeperID)
	if err != nil {
		h.writeError(w, err)
		return
	}

	httpx.JSON(w, http.StatusOK, a.toResponse(keeperUsername))
}

// Feed: POST /api/v1/animals/{id}/feed, optional body {"note": "..."}
func (h *Handler) Feed(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	var req struct {
		Note string `json:"note"`
	}
	// An empty body must not fail: note is optional. Decoding an empty
	// body (io.EOF - curl with no -d at all) is not a client error here.
	if r.ContentLength > 0 {
		if err := httpx.Decode(r, &req); err != nil {
			httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": "could not parse request body"})
			return
		}
	}

	claims, ok := auth.ClaimsFrom(r.Context())
	if !ok {
		httpx.JSON(w, http.StatusInternalServerError, map[string]string{"error": "no auth in context"})
		return
	}

	entry, err := h.svc.Feed(r.Context(), id, claims.ZookeeperID, req.Note)
	if err != nil {
		h.writeError(w, err)
		return
	}

	httpx.JSON(w, http.StatusCreated, map[string]any{
		"id":        entry.ID,
		"animal_id": entry.AnimalID,
		"keeper_id": entry.KeeperID,
		"fed_at":    entry.FedAt,
		"note":      entry.Note,
	})
}

// FeedHistory: GET /api/v1/animals/{id}/feed -> latest 20
func (h *Handler) FeedHistory(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	entries, usernames, err := h.svc.FeedHistory(r.Context(), id)
	if err != nil {
		h.writeError(w, err)
		return
	}

	resp := make([]map[string]any, len(entries))
	for i, e := range entries {
		resp[i] = map[string]any{
			"id":        e.ID,
			"animal_id": e.AnimalID,
			"fed_at":    e.FedAt,
			"note":      e.Note,
			"keeper":    map[string]any{"id": e.KeeperID, "username": usernames[i]},
		}
	}
	httpx.JSON(w, http.StatusOK, resp)
}
```

`writeError` maps the domain sentinels this domain has decided on, one place instead of five:

```go
// writeError centralizes sentinel -> status-code mapping for this domain.
// Stage 9 replaces it with the shared error pipeline; if you compare the
// two when you get there, notice this is the exact code being factored out.
func (h *Handler) writeError(w http.ResponseWriter, err error) {
	switch {
	case errors.Is(err, errAnimalNotFound) || errors.Is(err, errKeeperNotFound):
		httpx.JSON(w, http.StatusNotFound, map[string]string{"error": err.Error()})
	case errors.Is(err, errAnimalInvalid):
		httpx.JSON(w, http.StatusBadRequest, map[string]string{"error": err.Error()})
	default:
		httpx.JSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
	}
}
```

- **`switch {` with no condition** is Go's truth-test switch: each `case` is a boolean expression, evaluated top to bottom, first match wins. It reads like the if/else-if chain it replaces but keeps the classic switch shape, and it is the standard Go way to write multi-branch dispatch on error identity. The `default` case is the unmatched fall-through (the 500), which is why Stage 9 can promise "everything unknown becomes one status code unless a case says otherwise".
- Note the deliberate contrast with Stage 3's handlers: those wrote one `if errors.Is(...)` per handler; this file has three error-mapping methods per domain and the fan-out pays for a mapping function. When a file's error mapping starts repeating itself, extracting it like this is the move - and the sentence says so, because Stage 9 is about to do exactly that.

### 7.5 Mount the routes (internal/server/router.go)

Replace the animals group with:

```go
		router.Route("/animals", func(router chi.Router) {
			router.Group(func(router chi.Router) {
				router.Use(auth.AuthMiddleware([]byte(secret)))
				router.Get("/", an.List)
				router.Get("/{id}", an.Get)
				router.Post("/", an.Create)
				router.Put("/{id}", an.Update)
				router.Delete("/{id}", an.Delete)
				router.Put("/{id}/keeper", an.AssignKeeper)
				router.Post("/{id}/feed", an.Feed)
				router.Get("/{id}/feed", an.FeedHistory)
			})
		})
```

Stage 6's group already carried the auth middleware, so this is the same block with three more routes in it - the `/{id}/keeper` and `/{id}/feed` patterns slot in beside `/{id}` without any ordering concerns, because chi's matcher is a trie over the whole path, not a first-match-wins list.

### 7.6 Verify: the business rules, exercised

Stage 6's script deleted maya. Log in as sam (still there, username `sam`, password `elephant-road`) and keep that as `$TOKEN`:

```bash
go build ./... && go vet ./... && go run ./cmd/zoo serve

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
# {"animal_id":1,"fed_at":"2026-...","id":1,"keeper_id":2,"note":"morning hay and mineral block"}
# note the key order: this body is built from a map, and encoding/json sorts
# map keys alphabetically, so "animal_id" leads and "id" is third. Stage 1
# mentioned this; here is where it becomes visible. A struct response would
# keep the order you wrote instead.

# feed again with no body at all: optional note must not break
curl -s -X POST http://localhost:8080/api/v1/animals/1/feed -H "Authorization: Bearer $TOKEN"
# {"animal_id":1,"fed_at":"2026-...","id":2,"keeper_id":2,"note":null}

# last_fed_at is stamped (the transaction's second statement)
curl -s http://localhost:8080/api/v1/animals/1 -H "Authorization: Bearer $TOKEN"
# ... "last_fed_at":"2026-..." (matches the newest fed_at)

# history (newest first; each entry carries its keeper, keys again alphabetical)
curl -s http://localhost:8080/api/v1/animals/1/feed -H "Authorization: Bearer $TOKEN"
# [ {"animal_id":1,...,"id":2,"note":null}, {"animal_id":1,...,"id":1,"note":"morning hay and mineral block"} ]
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

> **A subtlety worth pausing on, because Stage 6 already demonstrated it:** maya was deleted there by a request carrying maya's own, still-valid token, and the API accepted it. JWTs are self-contained - the server verifies the signature, not the account's existence - so a token outlives the account it names. Nothing in this stage notices either; the "keeper not found" above comes from an explicit existence check, and if you try the destructive paragraph at the end of this section, the error you get is Postgres rejecting a `keeper_id` that no longer exists, not the middleware catching anything. Real systems add revocation (a checked token deny-list, very short expiry plus refresh tokens, or a DB check per request); the wrap-up lists it as the first exercise. Know this trade exists before one bites you.

If you are feeling destructive: create a keeper, feed an animal as them, then delete them via `DELETE /api/v1/zookeepers/{id}` - Postgres's RESTRICT turns the delete into an error the API reports as 500 today; remember the case, Stage 9 gives it its real status code (409 Conflict).

---

[Stage 6](06-animals-database.md)  |  [Overview](../tutorial.md)  |  [Stage 8](08-roles-workload.md)
