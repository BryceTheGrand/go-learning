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
- Delete the now-obsolete `writeError` method and the sentinel-based branches inside the per-method handlers; the file's `errors` and `pgx` imports go with them (keep `strconv`, `net/http`, gin, `auth`, and add `httperrors`).
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

[Stage 8](08-roles-workload.md)  ·  [Overview](../tutorial.md)  ·  [Stage 10](10-graceful-shutdown.md)
