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

The Go testing vocabulary in that file, since it is the first test file of the tutorial:

- **File and package naming**: test files end in `_test.go` and are *compiled only when you run `go test`* - never into the production binary. The package here is **`zookeepers_test`** (not `zookeepers`): Go's *external test package*, which imports the domain the way a stranger would (`zookeepers.Zookeeper`) and therefore tests the public surface rather than the innards. You could also test in `package zookeepers`; the external form is the stricter, more common choice.
- **`_ context.Context`**: a function *parameter* named `_` - the blank identifier works in parameter lists too, meaning "this position exists for the interface, the fake ignores the value". You saw `_` for discarded returns (Stage 1's range); this is its parameter-list use.
- **`func TestServiceCreate(t *testing.T)`**: any function named `TestXxx` taking `*testing.T` is discovered by `go test` and run as a test. `*testing.T` is the test's report handle; `t.Fatalf` marks the test failed and *stops it there* (versus `t.Errorf`, which marks failed but continues).
- **The table-drive pattern itself**: a slice literal of anonymous structs - one element per case, with `name` first - then `for _, tc := range tests { t.Run(tc.name, ...) }`. `t.Run` creates a *named subtest* for each row (`t.Run` takes a name and a function value; the function receives that row's own `t`), and its output names show up separately: `TestServiceCreate/empty_role_defaults_to_keeper`. Add-a-case-not-a-test is the whole point: the loop body never changes as the table grows.
- **`%v`, `%q`, `%+v`** in the failure messages: format verbs - `%v` prints a value in default form, `%q` prints a string *quoted* (so you can see empty vs whitespace exactly), `%+v` prints a struct with field names. These show up in nearly every Go test you will read.

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

Two remaining new words in the httptest file, worth their lines:

- **`t.Helper()`**: declares that the calling function (`signToken`, `mustHash`, `newTestEngine`) is *assistance code*, not a test. On failure, the reported line number points at the actual test, not at inside the helper - small feature, large debugging payoff once helpers multiply.
- **The `httptest` loop**: `httptest.NewRequest` builds a real `*http.Request` with no network in sight; `httptest.NewRecorder` gives you a writer object that *records* status and body instead of sending bytes anywhere; and `engine.ServeHTTP(rec, req)` runs the request through the whole routing + middleware + handler stack synchronously, in-process. You assert on `rec.Code` and `rec.Body`. This is why handler tests are fast and hermetic: the router is real, the network is simulated. Note that the negative-TTL case (`ttl: -time.Minute`) works because `IssueToken` stamps `ExpiresAt` in the past, and verification sees it - no clock mocking, no sleeps.

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

Add `-v` to see the subtests (`t.Run` rows print as `TestServiceCreate/duplicate_from_database_surfaces_as_conflict`, `TestIssueAndVerify/expired_token_does_not_verify`, and so on - spaces in case names become underscores). For one layer of confidence when you are not sure a test guards anything, break the code and re-run, e.g. temporarily remove `if role == "" { role = "keeper" }`; the "empty role defaults to keeper" case must fail. If it does not, the test is decoration. (Undo the break.)

---

[Stage 10](10-graceful-shutdown.md)  ·  [Overview](../tutorial.md)  ·  [Stage 12](12-makefile-recap.md)
