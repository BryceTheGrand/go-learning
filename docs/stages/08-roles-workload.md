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

One Go reading note on the first two lines: `authMiddleware := auth.AuthMiddleware(...)` and `requireAdmin := auth.RequireRole("admin")` assign the *returned closures* to local variables, so the factory functions run exactly once at wiring time and each closure is reused for every route that names it. Storing a function in a variable is unremarkable in Go - functions are values, Stage 1 taught - but naming these values (`requireAdmin` instead of `auth.RequireRole("admin")` at every mount) is what makes the route table read like a policy document.

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

[Stage 7](07-business-logic.md)  ·  [Overview](../tutorial.md)  ·  [Stage 9](09-error-pipeline-logging.md)
