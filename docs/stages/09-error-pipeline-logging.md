## Stage 9: One error pipeline and structured logging

Count the places in your handler files that decide an HTTP status from an error: Stage 3 introduced one per method, Stage 7 added `writeError`. All of them do the same three steps, in slightly different shapes: find a *domain* error under the surface, look up its status, write JSON. That is duplication with consequences - two handlers can drift (one 404s, one 500s) for the same condition.

This stage factors the sweep into one pipeline, and finishes the logging story with a structured request log.

(A note on what you will see while it lands: parts of the file - handlers, middleware, verify outputs - carry the old flat `{"error":"..."}` envelope in one listing, the new envelope in the next. During Stage 9 itself the server is briefly inconsistent across routes; 9.5's verify is the first point at which everything below it is swept. Any earlier stage's expected output you re-check mid-stage may still show the flat shape - that is the point of a sweep.)

### 9.1 The error type: internal/platform/httpx/errors.go

Stage 5 gave `httpx` the mechanical half of speaking HTTP (`JSON`, `Decode`). This is the half that knows what an error *means*: a second file in the same package, imported as `httpx` by the handlers that already import it.

```go
package httpx

import (
	"errors"
	"log/slog"
	"net/http"
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

func BadRequest(message string) *AppError {
	return New(http.StatusBadRequest, "invalid_request", message)
}

func NotFound(message string) *AppError {
	return New(http.StatusNotFound, "not_found", message)
}

func Forbidden(message string) *AppError {
	return New(http.StatusForbidden, "forbidden", message)
}

func Conflict(code, message string) *AppError {
	return New(http.StatusConflict, code, message)
}

// Respond writes err as the API's error envelope, once, in one place:
//
//	{"error": {"code": "not_found", "message": "animal not found"}}
//
// The message is static, by the way - it does not name the id that was
// missing. That is a deliberate trade (one shared AppError value for every
// 404) and 9.5 comes back to it.
//
// The envelope (an object with code and message) is worth the extra nesting
// over a bare "error": clients get a machine-readable code, the JSON shape
// never changes again, and adding a "details" field later is a change in one
// file.
func Respond(w http.ResponseWriter, err error) {
	var appErr *AppError
	if !errors.As(err, &appErr) {
		// Anything we did not classify: never leak internals to the client,
		// but log the full error server-side (see requestLogger in 9.4).
		slog.Error("unclassified error", "error", err)
		JSON(w, http.StatusInternalServerError, errorEnvelope{
			Error: &AppError{Code: "internal", Message: "internal error"},
		})
		return
	}

	// The struct marshals itself using the json tags above ("-" hides
	// Status; the client has no business seeing it as a field).
	JSON(w, appErr.Status, errorEnvelope{Error: appErr})
}

// errorEnvelope is the wire shape of every error response.
type errorEnvelope struct {
	Error *AppError `json:"error"`
}
```

- **`errors.As(err, &appErr)`** is the "find the typed error in the chain" tool from Stage 3's `isUniqueViolation`, now doing the real work: whatever layer produced or wrapped the failure, `Respond` unwraps to the `*AppError` underneath and uses its status. A plain `errors.New` with no `AppError` underneath is, by definition, a bug or an unhandled case, and gets the 500 path with the details in the log.
- **Why `*AppError` and not a value.** `Error()` has a pointer receiver, so only `*AppError` satisfies the `error` interface - which is exactly what makes `errors.As` able to find it, and what lets `Respond` read the `Status` field off the found value. The json tags then do the encoding, with `json:"-"` hiding the status from the body while it stays readable by Go code.
- **`errorEnvelope` is unexported** because nothing outside this file constructs a response body by hand any more. Handlers call `Respond` with an error; this type is the shape that comes out. (Stage 4's login handler did once build `map[string]any{"token": ..., "zookeeper": ...}` by hand - that is a *success* shape with two keys, and it stays a map; the envelope is specifically the error contract.)

### 9.2 Domain errors become the new type

`internal/zookeepers/service.go` - the `var` block becomes (complete replacement; `errInvalidCredentials` included; sentinel variables keep their `Err` names so `errors.Is` calls elsewhere still work):

```go
var (
	ErrNotFound           = httpx.New(http.StatusNotFound, "not_found", "zookeeper not found")
	ErrDuplicateUsername  = httpx.New(http.StatusConflict, "duplicate_username", "username already taken")
	ErrInvalidInput       = httpx.New(http.StatusBadRequest, "invalid_input", "username, password and role must be valid")
	ErrInvalidCredentials = httpx.New(http.StatusUnauthorized, "invalid_credentials", "invalid credentials")
	ErrKeeperInUse        = httpx.Conflict("keeper_in_use", "zookeeper has feed history and cannot be deleted")
)
```

which means `internal/zookeepers/service.go`'s import block gains two entries (with `"net/http"` going into the stdlib group and `"zoo/internal/platform/httpx"` into the third):

```go
import (
	"context"
	"errors"
	"net/http"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"

	"zoo/internal/platform/httpx"
)
```

`internal/animals/repository.go`:

```go
var (
	ErrNotFound       = httpx.New(http.StatusNotFound, "not_found", "animal not found")
	ErrInvalidInput   = httpx.New(http.StatusBadRequest, "invalid_input", "name, species and enclosure must be non-empty")
	ErrKeeperNotFound = httpx.New(http.StatusNotFound, "not_found", "keeper not found")
)
```

(`internal/animals/repository.go` gains the same two entries in its import block: `"net/http"` and `"zoo/internal/platform/httpx"`. It also **loses** `"errors"`: the error block no longer constructs with `errors.New`, and the compiler rejects the unused import. The zookeepers package's `service.go` keeps `errors` - its `isUniqueViolation`/`isFKViolation` helpers still use `errors.As`.)

(Animals' error block lives in `repository.go` from Stage 6; that was the informal arrangement - this is the moment it reads oddly enough to notice it lives with the repository and nothing else in that layer returns it. It stays; the wrap-up flags the alternative, `errors/`-package-per-domain.)

Two renames ripple: both domains had `errAnimalNotFound`/`errKeeperHasFeeds`-era names; every remaining use in each package updates to the `Err` names above. Specifically: `errAnimalNotFound` -> `ErrNotFound`, `errAnimalInvalid` -> `ErrInvalidInput`, `errKeeperNotFound` -> `ErrKeeperNotFound` in the animals package; `errKeeperHasFeeds` was defined but unused - it is replaced by `ErrKeeperInUse` (the naming honesty: the service decides, not the FK by itself).

And `internal/zookeepers/service.go`'s `Delete` gains the 409 promise Stage 7 made. Since the FK helper `isFKViolation` currently lives only in the animals package (private), zookeepers grows its own copy right next to `isUniqueViolation`:

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

	"github.com/go-chi/chi/v5"

	"zoo/internal/platform/auth"
	"zoo/internal/platform/httpx"
)

type Handler struct {
	svc         *Service
	tokenSecret []byte
	tokenTTL    time.Duration
}

func NewHandler(svc *Service, secret string, ttl time.Duration) *Handler {
	return &Handler{svc: svc, tokenSecret: []byte(secret), tokenTTL: ttl}
}

func parseID(w http.ResponseWriter, r *http.Request) (int64, bool) {
	id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		httpx.Respond(w, httpx.BadRequest("id must be an integer"))
		return 0, false
	}
	return id, true
}

func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Username string `json:"username"`
		Password string `json:"password"`
		Role     string `json:"role"`
	}
	if err := httpx.Decode(r, &req); err != nil {
		httpx.Respond(w, httpx.BadRequest("could not parse request body"))
		return
	}

	zk, err := h.svc.Create(r.Context(), req.Username, req.Password, req.Role)
	if err != nil {
		httpx.Respond(w, err)
		return
	}

	httpx.JSON(w, http.StatusCreated, zk.Response())
}

func (h *Handler) List(w http.ResponseWriter, r *http.Request) {
	zks, err := h.svc.List(r.Context())
	if err != nil {
		httpx.Respond(w, err)
		return
	}

	resp := make([]Response, len(zks))
	for i, zk := range zks {
		resp[i] = zk.Response()
	}
	httpx.JSON(w, http.StatusOK, resp)
}

func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	zk, err := h.svc.Get(r.Context(), id)
	if err != nil {
		httpx.Respond(w, err)
		return
	}

	httpx.JSON(w, http.StatusOK, zk.Response())
}

func (h *Handler) Update(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	var req struct {
		Username *string `json:"username"`
		Password *string `json:"password"`
		Role     *string `json:"role"`
	}
	if err := httpx.Decode(r, &req); err != nil {
		httpx.Respond(w, httpx.BadRequest("could not parse request body"))
		return
	}

	in := UpdateInput{Username: req.Username, Role: req.Role}
	if req.Password != nil {
		raw := *req.Password
		in.PasswordHash = &raw
	}

	zk, err := h.svc.Update(r.Context(), id, in)
	if err != nil {
		httpx.Respond(w, err)
		return
	}

	httpx.JSON(w, http.StatusOK, zk.Response())
}

func (h *Handler) Delete(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	if err := h.svc.Delete(r.Context(), id); err != nil {
		httpx.Respond(w, err)
		return
	}

	w.WriteHeader(http.StatusNoContent)
}

func (h *Handler) Login(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Username string `json:"username"`
		Password string `json:"password"`
	}
	if err := httpx.Decode(r, &req); err != nil {
		httpx.Respond(w, httpx.BadRequest("could not parse request body"))
		return
	}

	zk, err := h.svc.Authenticate(r.Context(), req.Username, req.Password)
	if err != nil {
		httpx.Respond(w, err)
		return
	}

	token, err := auth.IssueToken(h.tokenSecret, h.tokenTTL, zk.ID, zk.Username, zk.Role)
	if err != nil {
		httpx.Respond(w, err)
		return
	}

	httpx.JSON(w, http.StatusOK, map[string]any{"token": token, "zookeeper": zk.Response()})
}

func (h *Handler) Workload(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	wl, err := h.svc.Workload(r.Context(), id)
	if err != nil {
		httpx.Respond(w, err)
		return
	}

	httpx.JSON(w, http.StatusOK, wl)
}
```

Compare with the Stage 3 file. No `errors`, no `pgx`, no status logic beyond success codes; `parseID` stays because it is input decoding, not error mapping. The service layer decides what "wrong" means; `httpx.Respond` turns it into bytes.

Rules for the animals package (the same transformation, condensed):

- Every `if err != nil { <mapping> }` block in `handler.go` that reports a *service* error becomes `httpx.Respond(w, err)` + `return`. That qualifier matters, and it is the one place this rule will bite you if you apply it blindly. The errors that arrive from the service are `*AppError` values and describe themselves; the errors that arrive from *decoding input* are not. `parseID` in both domains and the `keeper_id` query-parameter branch in the animals handler are looking at a `*strconv.NumError`, so passing that to `httpx.Respond` would fall through to the unclassified branch and answer `500 {"error":{"code":"internal",...}}` where the API promises a 400. Those three sites keep naming their own error instead:
  ```go
  httpx.Respond(w, httpx.BadRequest("id must be an integer"))
  ```
  (the zookeepers handler's `parseID`, printed in full two sections up, already has that shape, and 9.5 asserts the 400 it produces). The dividing line is worth remembering generally: **errors the domain decided on describe themselves; errors the input produced have to be classified by the layer that knows what the input means.**
- Delete the now-obsolete `writeError` method and the sentinel-based branches inside the per-method handlers; the file's `errors` and `pgx` imports go with them (keep `strconv`, `net/http`, chi, `auth`, and add `httpx`).
- Every failed `httpx.Decode` writes the fixed `httpx.BadRequest("could not parse request body")`, never the decoder's own error. The handlers this touches are the zookeeper `Create`, `Update` and `Login` (Stages 3-4, where the flat-era versions echoed `err.Error()`) and the animal `Create` and `Update` (Stage 6). Do not echo raw decode errors to clients - they leak internal shape - and 9.4's logging is where that detail belongs instead.
- In `service.go`'s `AssignKeeper`/`Feed`/`FeedHistory`, the `errors.Is(err, pgx.ErrNoRows)` branches return `ErrNotFound`; FK violation in `Feed` returns `ErrKeeperNotFound`.
- `internal/animals/handler.go` keeps its own `parseID` (Stage 6's duplication note, still accurate), now calling `httpx.Respond`.

`service.go` in each package may still import `pgx` for these `ErrNoRows` checks: the boundary that changed is the handler's. If your style itches, note the alternative shape (repository translating `pgx.ErrNoRows` to domain errors) as a Stage 12 exercise.

**The middleware get the envelope too.** The auth layer still writes the old flat shape (`{"error": "invalid token"}`, `{"error": "requires role admin"}`), and Stage 9's promise is that *every* failure has the envelope - middleware failures are the most-hit failures of all. In `internal/platform/auth/middleware.go`, each rejection becomes an `AppError`:

```go
// AuthMiddleware, rejection paths:
		raw, ok := strings.CutPrefix(r.Header.Get("Authorization"), "Bearer ")
		if !ok {
			httpx.Respond(w, httpx.New(http.StatusUnauthorized,
				"missing_token", "missing or malformed Authorization header"))
			return
		}

		claims, err := VerifyToken(secret, raw)
		if err != nil {
			httpx.Respond(w, httpx.New(http.StatusUnauthorized, "invalid_token", "invalid token"))
			return
		}
```

```go
// RequireRole, rejection paths. The "no auth in context" case keeps a 500 -
// it means this middleware was mounted without AuthMiddleware in front of
// it, which is a wiring bug, not a client problem - but it is still an
// envelope, because Stage 9's promise is that no response is exempt.
		claims, ok := ClaimsFrom(r.Context())
		if !ok {
			httpx.Respond(w, httpx.New(http.StatusInternalServerError, "internal", "no auth in context"))
			return
		}

		if claims.Role != role {
			httpx.Respond(w, httpx.Forbidden("requires role "+role))
			return
		}
```

**And note what is not there.** The gin version of this middleware needed an `Abort` helper alongside `Respond`, because writing a response in gin middleware did not stop the chain - the handler downstream still ran unless you remembered to abort. chi middleware has no such trap: a middleware *is* the chain, and returning without calling `next.ServeHTTP` is the only way to reject. The `Abort` helper and the whole class of "responded but forgot to abort" bugs simply have nowhere to live. Deleting it is a real simplification, not a translation.

`middleware.go`'s import block gains `"zoo/internal/platform/httpx"` **and loses `"encoding/json"` and `"log/slog"`**, which were there only for `writeError`. Delete `writeError` itself - every call site in the file is now a `httpx.Respond` - and update the comment Stage 8 added to `RequireRole`, which pointed forward to exactly this cleanup and is now describing a function that no longer exists. That settles the promise Stage 4's comment made about it ("the duplication is the honest cost of the flat layout ... Stage 9 replaces the flat shape entirely, at which point `writeError` is deleted rather than fixed"). This is also the moment to notice what `writeError` cost: four lines, two imports and a second JSON writer, all because `package main` is not importable. Stage 5's `internal/platform/httpx` is what made deleting it possible.

That leaves exactly one flat body in the codebase, and it is the same wiring-bug guard one layer up: the two defensive "no auth in context" branches in `internal/animals/handler.go` (the `if !ok` guards after `ClaimsFrom`) become envelopes too, for consistency's sake:

```go
			httpx.Respond(w, httpx.New(http.StatusInternalServerError, "internal", "no auth in context"))
```

with `"zoo/internal/platform/httpx"` added to that file's imports (the `animals` handler stops writing `map[string]string{"error": ...}` entirely at this point).

### 9.4 The request log and the unmatched routes

Logging itself arrived in Stage 2: `internal/platform/logging.Setup` installs a real `slog` handler with a level from `LOG_LEVEL` and a format from `LOG_FORMAT`, so every call in the project is already structured and already configurable. What is missing is a line *per request*, and an envelope on the two failures chi produces before any handler runs.

`internal/server/router.go`, in `NewRouter`:

```go
func NewRouter(pool *pgxpool.Pool, secret string, zk *zookeepers.Handler, an *animals.Handler) *chi.Mux {
	router := chi.NewRouter()
	router.Use(middleware.Recoverer)
	router.Use(requestLogger)

	// Every failure gets the envelope - including the two the router
	// produces itself, before any handler is involved.
	router.NotFound(func(w http.ResponseWriter, r *http.Request) {
		httpx.Respond(w, httpx.NotFound("no such route"))
	})
	router.MethodNotAllowed(func(w http.ResponseWriter, r *http.Request) {
		httpx.Respond(w, httpx.New(http.StatusMethodNotAllowed,
			"method_not_allowed", "method not allowed"))
	})

	...
}
```

and, above `NewRouter`:

```go
// requestLogger logs one structured line per request once it completes.
// The keys are stable (a schema to search on); handlers never log requests
// themselves.
func requestLogger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		ww := middleware.NewWrapResponseWriter(w, r.ProtoMajor)
		next.ServeHTTP(ww, r)

		slog.Info("http",
			"method", r.Method,
			"path", r.URL.Path,
			"route", chi.RouteContext(r.Context()).RoutePattern(),
			"status", ww.Status(),
			"duration", time.Since(start).Round(time.Millisecond),
		)
	})
}
```

with `"log/slog"`, `"time"`, `"github.com/go-chi/chi/v5/middleware"` and `"zoo/internal/platform/httpx"` added to `router.go`'s imports.

**`router.Use(requestLogger)`, with no parentheses**, and this is worth a second because it is the mistake everyone makes once. `requestLogger` *is* a middleware already - it takes the next handler and returns a wrapper - so chi is handed the function itself. Stage 4's `auth.AuthMiddleware([]byte(secret))` looked different because that one is a *factory*: you call it to get the middleware. Writing `router.Use(requestLogger())` calls a function that needs an argument with none, and the compiler says so (`not enough arguments in call to requestLogger`). The reliable way to tell the two apart is the signature: a middleware's only parameter is `next http.Handler`, and a factory's is whatever you are configuring.

Three things are worth naming here:

- **`router.Use(middleware.Recoverer)`** is a panic guard from chi's own `middleware` package, and it is the direct replacement for what gin's `Default()` router installed for you. chi deliberately ships no logger and no recovery enabled by default, which is why both lines are explicit here rather than implied by a constructor's name.
- **`middleware.NewWrapResponseWriter(w, r.ProtoMajor)`** is the piece you cannot write yourself without effort: it is a `http.ResponseWriter` that records what was written to it - the status code and the byte count - before passing everything through. That is the only way to log a status code from *outside* the handler, since `net/http` does not tell you what status a handler chose. The second argument is the HTTP protocol version, which only matters for a byte-counting nicety.
- **`chi.RouteContext(r.Context()).RoutePattern()`** gives the *matched route pattern* (`/api/v1/animals/{id}`) rather than the concrete path (`/api/v1/animals/42`). Logging the pattern is what makes a log searchable by endpoint instead of by argument; if you prefer seeing the actual URL, `r.URL.Path` is sitting right there in the same call.
- **`router.NotFound` / `router.MethodNotAllowed`** are chi's hooks for the two responses it can generate with no handler involved. Without them, a request to `/api/v1/nope` gets chi's built-in plain-text `404 page not found`, which would quietly break the "every failure has the envelope" promise the moment a client typo'd a path.

### 9.5 Verify: the envelope everywhere

```bash
go build ./... && go vet ./... && go run ./cmd/zoo serve

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

# the two failures no handler ever sees, now in the same shape:
curl -s http://localhost:8080/api/v1/nope -H "Authorization: Bearer $ADMIN_TOKEN"
# {"error":{"code":"not_found","message":"no such route"}}
curl -s -X PUT http://localhost:8080/api/v1/animals -H "Authorization: Bearer $ADMIN_TOKEN"
# {"error":{"code":"method_not_allowed","message":"method not allowed"}}

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

# a classified failure on the way in: parseID's own AppError, so the status
# and code come from the error rather than from the handler
curl -s http://localhost:8080/api/v1/animals/not-a-number -H "Authorization: Bearer $ADMIN_TOKEN"
# {"error":{"code":"invalid_request","message":"id must be an integer"}}

# The branch for errors that match nothing is harder to trigger on purpose -
# it needs a bug, not a bad request. The way to see it is to break something:
# make the service return a plain errors.New instead of an AppError and watch
# the response become {"error":{"code":"internal","message":"internal error"}}
# while the server's own terminal gets the full error from
# slog.Error("unclassified error", ...). That split - detail in the log,
# nothing useful to the client - is the whole point of the fallback.
```

The server's terminal now shows one structured line per request:

```
time=2026-10-09T10:15:02.483+01:00 level=INFO msg=http method=GET path=/api/v1/animals/42 route=/api/v1/animals/{id} status=404 duration=0s
```

`time.Since(...).Round(time.Millisecond)` rounds sub-millisecond requests to `0s`, so most lines you see locally will read exactly like that one. In the JSON sample the same field reads `"duration":0`, and it is worth knowing that the two renderings differ in more than punctuation: `slog` writes a `time.Duration` as its integer nanosecond count in JSON, so a two-millisecond request logs `"duration":2000000` where the text handler prints `duration=2ms`. Same value, same key, different unit - which is exactly the sort of thing a log pipeline will trip over if nobody wrote it down. The rounding is there so the log lines stay short, not to hide slowness - real latency shows up when it is milliseconds or more.

The framing of those lines comes from `internal/platform/logging.Setup` (Stage 2.4), which installed an `slog.NewTextHandler` with `Level` taken from `LOG_LEVEL`. Run the server with `LOG_FORMAT=json` and the same line arrives as one JSON object:

```bash
LOG_FORMAT=json go run ./cmd/zoo serve
# {"time":"2026-10-09T10:15:02.483+01:00","level":"INFO","msg":"http","method":"GET","path":"/api/v1/animals/42","route":"/api/v1/animals/{id}","status":404,"duration":0}
```

That switch cost one line of code in Stage 2 and no changes anywhere else in the project, which is the entire argument for structured logging.

---

[Stage 8](08-roles-workload.md)  |  [Overview](../tutorial.md)  |  [Stage 10](10-graceful-shutdown.md)
