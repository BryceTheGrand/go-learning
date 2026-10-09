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
//
// The flat error bodies come from writeError, the local helper Stage 4 added
// to this file; Stage 9 replaces both call sites with the shared response
// envelope and deletes the helper.
func RequireRole(role string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims, ok := ClaimsFrom(r.Context())
			if !ok {
				// Wrong wiring (no auth middleware before this one), not a
				// client problem.
				writeError(w, http.StatusInternalServerError, "no auth in context")
				return
			}

			if claims.Role != role {
				writeError(w, http.StatusForbidden, "requires role "+role)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}
```

The shape has changed from the gin version, and every change is a simplification.

- **The signature is chi's middleware type**, `func(http.Handler) http.Handler`. So the factory-inside-a-factory reads like this: `RequireRole("admin")` runs the outer function once and returns a *middleware*; chi calls that middleware once per route that names it, and the middleware returns a handler wrapping `next`. It is the exact same factory shape as `AuthMiddleware`, one level of indirection deeper than you might expect - but the reason is the same, and it is what lets `requireAdmin` be built once and mounted many times (8.4).
- **The claims come from `r.Context()`**, by way of `ClaimsFrom` - the standard `context.Context` that travels with the request, not a gin-specific store. Stage 4 stored them there with `context.WithValue`; only the argument the reader passes changed, from gin's context to the request's.
- **Rejection is a bare `return`.** A chi middleware that does not call `next.ServeHTTP` simply never passes the request on. That *is* the mechanism gin's `Abort` helper existed to simulate, and its absence here is the point: there is no state to set, and no way to respond and then accidentally continue anyway.
- **The bodies go through `writeError`**, the four-line helper Stage 4 put in this same file, so they are still the flat `{"error": "..."}` shape - `httpx.Respond` does not exist yet. Stage 9 adds it to the `httpx` package, folds these two rejections and every other failure in the API into one shared envelope, and deletes `writeError` along with the two imports only it uses. Note the status codes are already correct (403 and 500): only the spelling of the body changes later.

One trade to say out loud: **claims can go stale.** The role inside a JWT is frozen at issue time. Promote a keeper to admin and their old token still says "keeper" until it expires (they must log in again); this is the reason Stage 7's comments said "no auth in context" defensively. A token issued to a *deleted* zookeeper still passes signature verification too (Stage 7's subtlety). Short TTLs are the standard mitigation; revocation is the exercise.

### 8.2 The seeded admin: migrations/00003_seed_admin.sql

You cannot type a bcrypt hash by hand. The one below is the bcrypt hash of `zoo-admin-password`, produced by a throwaway four-line `main.go` that called `bcrypt.GenerateFromPassword` the same way Stage 4's `password.go` does - the plaintext is the thing you never commit, and the hash is safe, because recovering the plaintext from it is exactly what bcrypt is for. The migration is idempotent via `ON CONFLICT`, so re-running it against a database that already has an `admin` row is safe:

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
go run ./cmd/zoo migrate    # as ever: same shell must have JWT_SECRET exported
```

### 8.3 Workload summary: the aggregate

`GET /api/v1/zookeepers/{id}/workload` answers "what is this keeper doing?" with one SQL round trip. This is also your first look at **scalar subqueries**: parentheses-wrapped queries that each produce one value, usable anywhere a value can go.

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
func (h *Handler) Workload(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	wl, err := h.svc.Workload(r.Context(), id)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			httpx.JSON(w, http.StatusNotFound, map[string]string{"error": "zookeeper not found"})
			return
		}
		httpx.JSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	httpx.JSON(w, http.StatusOK, wl)
}
```

The handler is the same three-step shape as every other one in this file: `parseID(w, r)` turns the path parameter into an `int64`, the service call gets `r.Context()`, and the error branch writes the flat body this stage is still using. (`parseID` itself is Stage 5's helper: `chi.URLParam(r, "id")` plus `strconv.ParseInt`, writing `{"error": "id must be an integer"}` on a bad segment.) Stage 9 replaces the two `httpx.JSON` error lines with a single `httpx.Respond(w, err)`, because by then `ErrNotFound` will carry its own status.

### 8.4 Mounting the gate: internal/server/router.go

Two middleware values are now needed at several points, so `NewRouter` builds them once at the top and names them, and the two groups are rewritten around them.

The zookeeper group becomes (complete replacement):

```go
	authMiddleware := auth.AuthMiddleware([]byte(secret))
	requireAdmin := auth.RequireRole("admin")

	router.Route("/zookeepers", func(router chi.Router) {
		// Bootstrap: the seeded admin is the only creator of further
		// admins. Stage 3-7's open creation is closed. Both middlewares
		// ride on this one route: authMiddleware first (it stores the
		// claims), then the role check that reads them.
		router.Group(func(router chi.Router) {
			router.Use(authMiddleware, requireAdmin)
			router.Post("/", zk.Create)
		})

		// Authenticated reads: this group's middleware runs for every
		// route declared inside it.
		router.Group(func(router chi.Router) {
			router.Use(authMiddleware)
			router.Get("/", zk.List)
			router.Get("/{id}", zk.Get)
			// Inside the authed group, authMiddleware has already run;
			// only the extra check is mounted alongside these routes.
			router.Group(func(router chi.Router) {
				router.Use(requireAdmin)
				router.Put("/{id}", zk.Update)
				router.Delete("/{id}", zk.Delete)
			})
			router.Get("/{id}/workload", zk.Workload)
		})
	})
```

And the animals group (complete replacement):

```go
	router.Route("/animals", func(router chi.Router) {
		// Feeding stays open to every signed-in keeper; the mutating
		// routes each add requireAdmin to the group's stack for that
		// one route.
		router.Group(func(router chi.Router) {
			router.Use(authMiddleware)
			router.Get("/", an.List)
			router.Get("/{id}", an.Get)
			router.With(requireAdmin).Post("/", an.Create)
			router.With(requireAdmin).Put("/{id}", an.Update)
			router.With(requireAdmin).Delete("/{id}", an.Delete)
			// Reassigning keepers is admin business; feeding is everyone's.
			router.With(requireAdmin).Put("/{id}/keeper", an.AssignKeeper)
			router.Post("/{id}/feed", an.Feed)
			router.Get("/{id}/feed", an.FeedHistory)
		})
	})
```

**How chi orders middleware**, since two mechanisms appear in that tree. A middleware registered with `router.Use` inside a group runs for *every* route declared in that group, and groups nest, so the chain is built outside-in: the outer group's middleware runs first, then the inner group's. A single route can add more on top of its group's stack with `router.With(...)`, which returns a router carrying the group's middleware *plus* the ones you name - so `router.With(requireAdmin).Put("/{id}", an.Update)` runs the group's `authMiddleware` first and `requireAdmin` after it. Where several middlewares are named in one call (`router.Use(authMiddleware, requireAdmin)`), they run left to right.

That order is what makes `router.With(requireAdmin).Put("/{id}", an.Update)` need no repeated auth: the enclosing group already runs `authMiddleware`, and `With`'s list is *additional* on top of it ("group's first, then the route's"). The ordering is not merely tidy - `requireAdmin` reads the claims `authMiddleware` stored, so auth has to be the earlier one in every spelling above. The nested `router.Group(func(router chi.Router) { router.Use(requireAdmin); ... })` on `Update`/`Delete` is the same idea once more, with no prefix at all: a group whose only job is to put `requireAdmin` in front of the two routes inside it.

One chi spelling note, because it is the gin-ism most likely to be mistyped: chi's route methods (`Get`, `Post`, `Put`, `Delete`, ...) take a path and a handler and nothing more - there is no `router.Put("/{id}", requireAdmin, h)` form at all. Per-route middleware is always spelled `With`; whole-group middleware is always `Use`.

One Go reading note on the first two lines: `authMiddleware := auth.AuthMiddleware(...)` and `requireAdmin := auth.RequireRole("admin")` assign the *returned closures* to local variables, so the factory functions run exactly once at wiring time and each closure is reused for every route that names it. Storing a function in a variable is unremarkable in Go - functions are values, Stage 1 taught - but naming these values (`requireAdmin` instead of `auth.RequireRole("admin")` at every mount) is what makes the route table read like a policy document.

### 8.5 Verify: 403s where they belong

```bash
go build ./... && go vet ./... && go run ./cmd/zoo serve

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
  -d '{"note":"dinner"}'
# {"animal_id":1,"fed_at":"2026-...","id":3,"keeper_id":2,"note":"dinner"}   (sam's feeds so far: 1 and 2)

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

[Stage 7](07-business-logic.md)  |  [Overview](../tutorial.md)  |  [Stage 9](09-error-pipeline-logging.md)
