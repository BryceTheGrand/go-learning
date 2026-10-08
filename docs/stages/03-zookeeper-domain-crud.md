## Stage 3: The zookeeper domain, layered against the database

The API server is still one directory of files in `package main`, but zookeeper accounts now live in Postgres and touch three layers:

- **repository** - the only place with SQL. Knows the table, returns domain structs.
- **service** - business rules (this stage: valid username, unique username). Calls the repository.
- **handler** - decodes HTTP, asks the service, writes a status code. Calls the service.

Why three layers for what is still "insert a row"? Because it is cheap now, while the rules are trivial, and because the layer *seams* are what Stages 4-9 bolt rules onto. The test in Stage 11 exists only because of this split. In Stage 5 the trio moves to `internal/zookeepers/` and is otherwise unchanged.

As in Stage 1, each file is built piece by piece with explanations, and every section ends with the complete file so you can check your assembly.

**Where these files go:** all three new files live in the **project root**, in the same directory as `main.go`, and all three declare `package main` like it does. Nothing moves into a subdirectory in this stage. The root ends up holding four `.go` files - `main.go`, `zookeepers_repository.go`, `zookeepers_service.go`, `zookeepers_handler.go` - and Stage 5 is the stage that finally splits them into `internal/` packages. Create each file as you reach its section.

#### The repository: zookeepers_repository.go

`zookeepers_repository.go`, in the project root:

The repository is the only file that knows SQL exists. Everything it does: turn SQL rows into Go structs, and structs into rows.

##### Package, import, and the domain type

```go
package main

import (
	"context"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

// zookeeper is the internal domain type: everything the database stores,
// including things (the password hash) that must never leave this domain.
type zookeeper struct {
	ID           int64
	Username     string
	PasswordHash string
	Role         string
	CreatedAt    time.Time
	UpdatedAt    time.Time
}
```

- Two new things in the import block: **`time`** (stdlib) for timestamps, and **`pgxpool`**, the pool constructor from Stage 2.2/2.3 - the repository receives the pool, it does not open one itself.
- **`time.Time`** is Go's instant-in-time type. pgx scans Postgres `timestamptz` columns into it directly; when this struct is JSON-serialized the default format is RFC 3339 (`"2026-10-08T10:15:02Z"`), which is why every later curl shows `2026-...` strings.
- **No JSON tags on this type**, by design: this is the *internal* type. What the API sends out is a separate struct (see the handler below). The password hash lives here and here only.
- All fields exported: they cross function boundaries but stay inside one package for now; capitalization gets *meaningful* in Stage 5 when packages split.

##### The repository struct and its constructor

```go
type zookeeperRepository struct {
	pool *pgxpool.Pool
}

func newZookeeperRepository(pool *pgxpool.Pool) *zookeeperRepository {
	return &zookeeperRepository{pool: pool}
}
```

- The repository needs only one thing: the pool. Holding it as a field means every method below can say `r.pool` without passing anything around.
- **The constructor**: a plain function named `new<Thing>` that allocates and returns it. `&zookeeperRepository{pool: pool}` - the `&` takes the address of the fresh struct literal, returning `*zookeeperRepository`. Go has no `new` keyword rituals for this; a constructor is a plain function, and Go convention is that *any* type you hand across function boundaries is a pointer, built like this.
- **Methods** are the new syntax just below it: `func (r *zookeeperRepository) Create(...)`. The parenthesized name before the function name is the **receiver** - "this method belongs to `zookeeperRepository`, and inside it the instance is called `r`". It is how every Go type gets behavior attached: there are no classes; a struct plus its methods is the unit. The receiver is a *pointer* (`*zookeeperRepository`) when the method wants the real struct (methods on a value receiver work on a copy). Convention: pick one receiver type per type and stay consistent - pointer if any method needs to mutate or if the struct is handed around, which is the case for everything in this tutorial.

#### Repository.Create

```go
func (r *zookeeperRepository) Create(ctx context.Context, zk zookeeper) (zookeeper, error) {
	row := r.pool.QueryRow(ctx,
		`INSERT INTO zookeepers (username, password_hash, role)
		 VALUES ($1, $2, $3)
		 RETURNING id, username, password_hash, role, created_at, updated_at`,
		zk.Username, zk.PasswordHash, zk.Role,
	)

	var created zookeeper
	err := row.Scan(&created.ID, &created.Username, &created.PasswordHash, &created.Role,
		&created.CreatedAt, &created.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return created, nil
}
```

- **`QueryRow`** runs one statement that should return exactly one row and hands you a *handle to scan it*. Arguments in order: the context, the SQL, then one value per `$n` placeholder. **`$1, $2, ...`, not string concatenation** - placeholders are how SQL stays injection-proof, and pgx converts Go values to Postgres types for you.
- Backtick strings are Go's *raw string literals*: no escaping, newlines included. SQL is the canonical use; the indentation actually goes into the SQL string, and Postgres does not care.
- **`RETURNING ...`** is a Postgres feature worth knowing: the INSERT/UPDATE statement also returns the row as stored after the change - including the database-assigned `id` and the `created_at` the database set. Instead of insert-then-select (two round trips, racy), you get the full row back in one.
- **`row.Scan(&created.ID, ...)`** copies the one row's columns into your variables, in order - column order in the SQL must match argument order in Scan, and that correspondence is *the* maintenance hazard of every repository (miss one, and your `ID` field gets `Username`). pgx handles converting Postgres types to Go types: `bigint` into `int64`, `timestamptz` into `time.Time`, `text` into `string`.
- **`return zookeeper{}, err`** on failure: the *zero value* `zookeeper{}` plus the error. The error convention from Stage 1 (result first, error last) applies; you owe the caller a real error and may not hand back a half-filled struct.

#### Repository.Get

```go
func (r *zookeeperRepository) Get(ctx context.Context, id int64) (zookeeper, error) {
	var zk zookeeper
	err := r.pool.QueryRow(ctx,
		`SELECT id, username, password_hash, role, created_at, updated_at
		 FROM zookeepers WHERE id = $1`, id).
		Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role, &zk.CreatedAt, &zk.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return zk, nil
}
```

- Same shape as Create: `QueryRow` + `Scan`, placeholders for the WHERE clause. Note the chain style: `.Scan` on the line after `QueryRow(...)` - the method call was long enough that the tutorial breaks it after the comma; purely style.
- Here is a fact the later stages build on: when **no row matches**, `Scan` returns **`pgx.ErrNoRows`**. "Not found" is a *normal error return*, not a panic and not `nil`. Stage 3's handler checks for it explicitly with `errors.Is` (see the pgx-notes callout after the handler); Stage 9 moves the check into the service.
- The `SELECT` column list is spelled out instead of `SELECT *`: two reasons real code does this. It makes the Scan-order contract visible, and adding a column later cannot silently change row shape in ways that desync the Scan.

#### Repository.List: the multi-row loop

```go
func (r *zookeeperRepository) List(ctx context.Context) ([]zookeeper, error) {
	rows, err := r.pool.Query(ctx,
		`SELECT id, username, password_hash, role, created_at, updated_at
		 FROM zookeepers ORDER BY id`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	zks := []zookeeper{}
	for rows.Next() {
		var zk zookeeper
		if err := rows.Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role,
			&zk.CreatedAt, &zk.UpdatedAt); err != nil {
			return nil, err
		}
		zks = append(zks, zk)
	}
	if err := rows.Err(); err != nil {
		return nil, err
	}
	return zks, nil
}
```

The multi-row dance is different from `QueryRow`, and it is the shape every list endpoint in every Go service uses:

- **`Query`** (plural) instead of `QueryRow`: returns a live `rows` iterator over potentially many rows.
- **`defer rows.Close()`** - the Stage 2 `defer` habit, now for a resource that came from a *successful* call. Every defer-able resource in Go follows the same rule: acquire, defer the release, then work.
- **`for rows.Next()`** - range-free iteration, advancing row by row; the loop ends when rows are exhausted. Inside: declare one struct per row, Scan, append - the exact three moves as Create.
- **`rows.Err()` after the loop**: subtle but not optional - `Next()` returning false can be caused by a *mid-iteration network error*, not just exhaustion, and only the final `Err()` reveals it. Omitting it means "query died halfway" masquerades as "empty result". Every pgx or `database/sql` list loop you will read ends this way.
- **`zks := []zookeeper{}`** initializes an empty (non-nil) slice so a found-nothing result serializes as `[]` rather than `null`. Small deliberate choice; JSON clients care.

#### Repository.Update: partial updates with COALESCE

```go
func (r *zookeeperRepository) Update(ctx context.Context, id int64, zku zookeeperUpdate) (zookeeper, error) {
	// COALESCE($n, col) means "use the new value if it was sent, else keep
	// the column as it is" - the standard SQL shape for partial updates.
	var zk zookeeper
	err := r.pool.QueryRow(ctx,
		`UPDATE zookeepers
		 SET username     = COALESCE($1, username),
		     password_hash = COALESCE($2, password_hash),
		     role         = COALESCE($3, role),
		     updated_at   = now()
		 WHERE id = $4
		 RETURNING id, username, password_hash, role, created_at, updated_at`,
		zku.Username, zku.PasswordHash, zku.Role, id).
		Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role, &zk.CreatedAt, &zk.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return zk, nil
}
```

- The `zookeeperUpdate` type it takes is defined in the service file below, and it is the answer to a question every PUT endpoint must answer: **if the client sends `{"role":"admin"}` only, what happens to the username?** This SQL's answer: untouched. The trick is `COALESCE($n, col)`: if the parameter is NULL (Go sent a nil pointer), keep the current column; if the parameter has a value, use it. So "field not sent" and "field sent" both arrive in one round trip.
- The nil-ness travels through pgx automatically: a Go `*string` that is nil becomes SQL NULL; a non-nil pointer is followed and its string sent. This is the first structural use of pointers-for-nullability you saw in the error convention: "absent" in JSON needs a nullable Go type, and Go's nullable type is a pointer.
- Like Create, it is `QueryRow` + `RETURNING ...` + `Scan` because SQL gives the post-change row right back.
- **No row at id?** `UPDATE ... WHERE id = $4` matching nothing is *not* an error - it is `ErrNoRows` from the `RETURNING`-scan. Same not-found story as Get, and both are handled in the handler below.

#### Repository.Delete: Exec and RowsAffected

```go
func (r *zookeeperRepository) Delete(ctx context.Context, id int64) error {
	tag, err := r.pool.Exec(ctx, `DELETE FROM zookeepers WHERE id = $1`, id)
	if err != nil {
		return err
	}
	// DELETE does not error when nothing matched; check the row count.
	if tag.RowsAffected() == 0 {
		return errZookeeperNotFound
	}
	return nil
}
```

- **`Exec`** is the third and last pgx call shape: for statements whose interesting output is *how many rows*, not rows themselves (`DELETE`, and Stage 6's UPDATE). It returns a command tag; `tag.RowsAffected()` is the count. Zero means id 99 pointed at nothing - and *that* has to be turned into a "not found" error by you, because SQL did not consider it an error.
- `return nil` is the entire success path: a DELETE that deleted exactly one row.

##### The complete zookeepers_repository.go

```go
package main

import (
	"context"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

// zookeeper is the internal domain type: everything the database stores,
// including things (the password hash) that must never leave this domain.
type zookeeper struct {
	ID           int64
	Username     string
	PasswordHash string
	Role         string
	CreatedAt    time.Time
	UpdatedAt    time.Time
}

type zookeeperRepository struct {
	pool *pgxpool.Pool
}

func newZookeeperRepository(pool *pgxpool.Pool) *zookeeperRepository {
	return &zookeeperRepository{pool: pool}
}

func (r *zookeeperRepository) Create(ctx context.Context, zk zookeeper) (zookeeper, error) {
	row := r.pool.QueryRow(ctx,
		`INSERT INTO zookeepers (username, password_hash, role)
		 VALUES ($1, $2, $3)
		 RETURNING id, username, password_hash, role, created_at, updated_at`,
		zk.Username, zk.PasswordHash, zk.Role,
	)

	var created zookeeper
	err := row.Scan(&created.ID, &created.Username, &created.PasswordHash, &created.Role,
		&created.CreatedAt, &created.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return created, nil
}

func (r *zookeeperRepository) Get(ctx context.Context, id int64) (zookeeper, error) {
	var zk zookeeper
	err := r.pool.QueryRow(ctx,
		`SELECT id, username, password_hash, role, created_at, updated_at
		 FROM zookeepers WHERE id = $1`, id).
		Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role, &zk.CreatedAt, &zk.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return zk, nil
}

func (r *zookeeperRepository) List(ctx context.Context) ([]zookeeper, error) {
	rows, err := r.pool.Query(ctx,
		`SELECT id, username, password_hash, role, created_at, updated_at
		 FROM zookeepers ORDER BY id`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	zks := []zookeeper{}
	for rows.Next() {
		var zk zookeeper
		if err := rows.Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role,
			&zk.CreatedAt, &zk.UpdatedAt); err != nil {
			return nil, err
		}
		zks = append(zks, zk)
	}
	if err := rows.Err(); err != nil {
		return nil, err
	}
	return zks, nil
}

func (r *zookeeperRepository) Update(ctx context.Context, id int64, zku zookeeperUpdate) (zookeeper, error) {
	// COALESCE($n, col) means "use the new value if it was sent, else keep
	// the column as it is" - the standard SQL shape for partial updates.
	var zk zookeeper
	err := r.pool.QueryRow(ctx,
		`UPDATE zookeepers
		 SET username     = COALESCE($1, username),
		     password_hash = COALESCE($2, password_hash),
		     role         = COALESCE($3, role),
		     updated_at   = now()
		 WHERE id = $4
		 RETURNING id, username, password_hash, role, created_at, updated_at`,
		zku.Username, zku.PasswordHash, zku.Role, id).
		Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role, &zk.CreatedAt, &zk.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return zk, nil
}

func (r *zookeeperRepository) Delete(ctx context.Context, id int64) error {
	tag, err := r.pool.Exec(ctx, `DELETE FROM zookeepers WHERE id = $1`, id)
	if err != nil {
		return err
	}
	// DELETE does not error when nothing matched; check the row count.
	if tag.RowsAffected() == 0 {
		return errZookeeperNotFound
	}
	return nil
}
```

#### The service: zookeepers_service.go

`zookeepers_service.go`, in the project root (beside `main.go` and `zookeepers_repository.go`):

The service is the business-rules layer. It decides what counts as valid, what a database error *means* in domain terms, and what callers may not do. It never writes SQL, and it never knows about HTTP.

##### Sentinel errors

```go
package main

import (
	"context"
	"errors"
)

// Sentinel errors: value-level errors the service decides on. Handlers map
// them to status codes without knowing SQL exists.
var (
	errZookeeperNotFound  = errors.New("zookeeper not found")
	errZookeeperDuplicate = errors.New("username already taken")
	errZookeeperInvalid   = errors.New("username, password and role must be non-empty")
)
```

- **Sentinel errors** are Go's name for errors you define once as values and compare against. `errors.New("...")` creates an error value; the `var (...)` block groups declarations, same as `import (...)`. Nothing special - an error *is* a value, so naming one and reusing it (in returns, in `errors.Is` comparisons by handlers) gives the rest of the program a stable vocabulary of failures.
- Names are lowercase (`err...`): they are private - no other package needs to know them (Stage 9 will make some of them public with capital `Err` when handlers in another package need to recognize them).
- The convention this tutorial follows throughout: **decide errors where the business rule lives (service), translate to statuses where HTTP lives (handler)**.

##### The update-input type: pointers as "absent"

```go
// zookeeperUpdate carries optional changes for PUT. nil means "not sent";
// a non-nil empty string means "set to empty string". Pointers are how Go
// distinguishes absent from empty in JSON.
type zookeeperUpdate struct {
	Username     *string
	PasswordHash *string
	Role         *string
}
```

This is one of the most useful little structs in Go web development, so it gets its own explanation. The handler will decode `{"username":"sam"}` - role absent - and needs to tell the database "username: change, role: leave alone". A plain `string` cannot say "leave alone" (its zero value is `""`, which is *some* value); a `*string` can: `nil` pointer = not sent, non-nil pointer = sent (even to empty string). The handler fills these pointers, the repository passes them to `COALESCE` above, and nil-ness becomes SQL NULL exactly as pgx sends it. This is also the shape Stage 6's animals reuse.

##### The service struct, constructor, and Create

```go
type zookeeperService struct {
	repo *zookeeperRepository
}

func newZookeeperService(repo *zookeeperRepository) *zookeeperService {
	return &zookeeperService{repo: repo}
}

func (s *zookeeperService) Create(ctx context.Context, username, password, role string) (zookeeper, error) {
	if role == "" {
		role = "keeper"
	}
	if username == "" || password == "" || (role != "admin" && role != "keeper") {
		return zookeeper{}, errZookeeperInvalid
	}

	// Stage 3 shortcut: the raw password is stored. Stage 4 makes this real.
	zk, err := s.repo.Create(ctx, zookeeper{
		Username:     username,
		PasswordHash: password,
		Role:         role,
	})
	if err != nil {
		if isUniqueViolation(err) {
			return zookeeper{}, errZookeeperDuplicate
		}
		return zookeeper{}, err
	}
	return zk, nil
}
```

- The constructor holds the repository - exactly the pattern of the repository holding the pool, one layer up.
- `username, password, role string` - several parameters may share a type; Go allows `a, b string`.
- **Business rules, in order**: default the role, then reject empties (the `||`-chain is plain boolean logic; note Go has no truthiness - `if username` does not work, emptiness is spelled `== ""`).
- **`return zookeeper{}, errZookeeperInvalid`**: returning the *sentinel error itself* rather than a message. The handler will recognize it with `errors.Is` below. This is the vocabulary-at-the-seam idea: the service speaks in domain failures, the handler speaks in status codes.
- **`isUniqueViolation(err)`** translates "Postgres said duplicate key" into "username already taken". A unique-constraint violation arrives as a *pgx-typed* error, not this sentinel; the check is the helper at the bottom of this file. This translation layer (database facts -> domain meaning) is the service's whole reason to exist.

#### The other service methods

```go
func (s *zookeeperService) Get(ctx context.Context, id int64) (zookeeper, error) {
	return s.repo.Get(ctx, id)
}

func (s *zookeeperService) List(ctx context.Context) ([]zookeeper, error) {
	return s.repo.List(ctx)
}

func (s *zookeeperService) Update(ctx context.Context, id int64, zku zookeeperUpdate) (zookeeper, error) {
	zk, err := s.repo.Update(ctx, id, zku)
	if err != nil {
		if isUniqueViolation(err) {
			return zookeeper{}, errZookeeperDuplicate
		}
		return zookeeper{}, err
	}
	return zk, nil
}

func (s *zookeeperService) Delete(ctx context.Context, id int64) error {
	return s.repo.Delete(ctx, id)
}
```

Get, List and Delete are straight delegation (no rules *yet* - Stages 4, 7, 8, 9 change that). Update duplicates Create's database-error translation, and that duplication is honest: the SQL facts are the same, the rule "unique means taken" applies to both. (Stage 9 will also fold not-found translation into this layer.)

##### The helper: isUniqueViolation

```go
// isUniqueViolation reports whether err is Postgres's unique-constraint
// violation (SQLSTATE 23505). errors.As unwraps the chain to find the
// concrete *pgconn.PgError inside.
func isUniqueViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23505"
}
```

Two ideas in five lines:

- **`errors.As`** asks "is there a `*pgconn.PgError` anywhere inside this error (possibly wrapped through several layers)?" - if so, it fills `pgErr` with that value and returns true. This is the standard tool for "extract the concrete error type" and works because Stage 2's `%w` wrapping kept the chain intact.
- **`23505`** is the SQLSTATE code for unique violation - Postgres's machine-readable error taxonomy, stable across versions, which is why matching on the code (rather than the message text) is the professional move. Stage 7 adds its sibling, the FK-violation code 23503.

which changes the import block of the file to:

```go
import (
	"context"
	"errors"

	"github.com/jackc/pgx/v5/pgconn"
)
```

##### The complete zookeepers_service.go

```go
package main

import (
	"context"
	"errors"

	"github.com/jackc/pgx/v5/pgconn"
)

// Sentinel errors: value-level errors the service decides on. Handlers map
// them to status codes without knowing SQL exists.
var (
	errZookeeperNotFound  = errors.New("zookeeper not found")
	errZookeeperDuplicate = errors.New("username already taken")
	errZookeeperInvalid   = errors.New("username, password and role must be non-empty")
)

// zookeeperUpdate carries optional changes for PUT. nil means "not sent";
// a non-nil empty string means "set to empty string". Pointers are how Go
// distinguishes absent from empty in JSON.
type zookeeperUpdate struct {
	Username     *string
	PasswordHash *string
	Role         *string
}

type zookeeperService struct {
	repo *zookeeperRepository
}

func newZookeeperService(repo *zookeeperRepository) *zookeeperService {
	return &zookeeperService{repo: repo}
}

func (s *zookeeperService) Create(ctx context.Context, username, password, role string) (zookeeper, error) {
	if role == "" {
		role = "keeper"
	}
	if username == "" || password == "" || (role != "admin" && role != "keeper") {
		return zookeeper{}, errZookeeperInvalid
	}

	// Stage 3 shortcut: the raw password is stored. Stage 4 makes this real.
	zk, err := s.repo.Create(ctx, zookeeper{
		Username:     username,
		PasswordHash: password,
		Role:         role,
	})
	if err != nil {
		if isUniqueViolation(err) {
			return zookeeper{}, errZookeeperDuplicate
		}
		return zookeeper{}, err
	}
	return zk, nil
}

func (s *zookeeperService) Get(ctx context.Context, id int64) (zookeeper, error) {
	return s.repo.Get(ctx, id)
}

func (s *zookeeperService) List(ctx context.Context) ([]zookeeper, error) {
	return s.repo.List(ctx)
}

func (s *zookeeperService) Update(ctx context.Context, id int64, zku zookeeperUpdate) (zookeeper, error) {
	zk, err := s.repo.Update(ctx, id, zku)
	if err != nil {
		if isUniqueViolation(err) {
			return zookeeper{}, errZookeeperDuplicate
		}
		return zookeeper{}, err
	}
	return zk, nil
}

func (s *zookeeperService) Delete(ctx context.Context, id int64) error {
	return s.repo.Delete(ctx, id)
}

// isUniqueViolation reports whether err is Postgres's unique-constraint
// violation (SQLSTATE 23505). errors.As unwraps the chain to find the
// concrete *pgconn.PgError inside.
func isUniqueViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23505"
}
```

#### The handler: zookeepers_handler.go

`zookeepers_handler.go`, in the project root (the third and last new file of this stage):

The handler is the HTTP edge: it decodes requests, writes responses, and translates service errors to statuses. New Go concepts land here: a method with a value receiver, response DTOs, and `errors.Is`.

##### The response type and the first value receiver

```go
package main

import (
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
)

// zookeeperResponse is what the outside world sees. Note what is missing:
// PasswordHash. Responses are a separate shape from domain structs, and the
// password hash is the reason.
type zookeeperResponse struct {
	ID        int64     `json:"id"`
	Username  string    `json:"username"`
	Role      string    `json:"role"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

func (z zookeeper) toResponse() zookeeperResponse {
	return zookeeperResponse{
		ID:        z.ID,
		Username:  z.Username,
		Role:      z.Role,
		CreatedAt: z.CreatedAt,
		UpdatedAt: z.UpdatedAt,
	}
}
```

- **The DTO concept**: a data-transfer shape, separate from the domain type. One struct per concern; the security property ("no hash ever leaves the domain") then falls out of "the response type has no such field" - no discipline required, no accidental `zk` leak possible... you cannot send what the shape does not contain.
- `CreatedAt time.Time` has a JSON tag but not a custom format: `time.Time` marshals as RFC 3339 by default.
- **`func (z zookeeper) toResponse()`** is a method with a **value receiver** - the first one (all earlier methods were pointer receivers like `func (r *zookeeperRepository)`). Reading it: a method on the `zookeeper` type itself, receiver named `z`, returning a fresh response. Value receiver here because the method only *reads* and returns something new - no mutation, no sharing needed. The rule of thumb: reads-only on small structs may be value receivers; anything else is pointer. (Being consistent inside a type matters more than picking perfectly on the first try.)
- Note the receiver being a *different type's* method than the file's name suggests: Go does not care where in the package the method is declared; it attaches to the type. This file is the response/HTTP edge, and `zookeeper.toResponse` lives here because it is used here (Stage 5's split will move it out with a comment).

##### parseID: the (value, ok) helper

```go
// parseID reads the :id path parameter, writing a 400 and returning false
// if it is not an integer. Small repetition killers like this are the first
// sign your "flat" files need Stage 5.
func parseID(c *gin.Context) (int64, bool) {
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "id must be an integer"})
		return 0, false
	}
	return id, true
}
```

- Nothing new in the mechanics (ParseInt from Stage 1, scoped `if`). What is new is the **`(int64, bool)` return shape**: "the value, and whether producing it succeeded". It is Go's second standard return-pair (the first being `(value, error)`); this one is for helpers that already *handled* the failure themselves (wrote the 400) and only need to tell their caller "do not continue". The caller pattern below is `id, ok := parseID(c); if !ok { return }`.
- Note what it saves: every `:id` handler in this and every later file calls it once instead of four repeated lines.

##### The handler struct and create

```go
type zookeeperHandler struct {
	svc *zookeeperService
}

func newZookeeperHandler(svc *zookeeperService) *zookeeperHandler {
	return &zookeeperHandler{svc: svc}
}

func (h *zookeeperHandler) create(c *gin.Context) {
	var req struct {
		Username string `json:"username" binding:"required"`
		Password string `json:"password" binding:"required"`
		Role     string `json:"role"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	zk, err := h.svc.Create(c.Request.Context(), req.Username, req.Password, req.Role)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			c.JSON(http.StatusConflict, gin.H{"error": "username already taken"})
			return
		}
		if errors.Is(err, errZookeeperInvalid) {
			c.JSON(http.StatusBadRequest, gin.H{"error": "username and password are required"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusCreated, zk.toResponse())
}
```

- **The anonymous request struct**: `var req struct {...}` - a struct type declared inline, used once. Go allows (and web handlers love) this: fields + JSON tags + no name. `binding:"required"` extends those tags with a gin-specific one: "reject the request if the field is absent", checked by `ShouldBindJSON`. Three tags in one backtick string are the standard spelling.
- **`c.Request.Context()`** - here is the context plumbing, finally visible: gin's context wraps the standard `*http.Request`, which owns a real `context.Context` carrying request cancellation. Handlers pass *that* into services and repositories. (Stage 7 explains what it enables; for now, copy the habit: never invent a context inside a handler, take it from `c.Request`.)
- **`errors.Is(err, errZookeeperDuplicate)`**: the reader for sentinel errors. `==` would also work for a directly-returned sentinel, but `errors.Is` is the norm because it understands wrapped errors (`%w`). It is how the handler recognizes the service's vocabulary without sharing any types beyond the sentinels.
- **Status mapping, in full**: 409 Conflict for "username exists" (the REST-canonical code for resource-state conflicts), 400 for malformed input, 500 for everything else. The 500 case: the error reached the handler but matched nothing known - the handler must not leak it to the client ("internal error" only), and real logging of it arrives in Stage 9.
- Response bodies stay flat (`gin.H{"error": "..."}`); Stage 9 gives them a consistent envelope.

#### list, get, update, delete

```go
func (h *zookeeperHandler) list(c *gin.Context) {
	zks, err := h.svc.List(c.Request.Context())
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	resp := make([]zookeeperResponse, len(zks))
	for i, zk := range zks {
		resp[i] = zk.toResponse()
	}
	c.JSON(http.StatusOK, resp)
}
```

- **`make([]T, n)`**: second slice constructor (Stage 1 had a literal, Stage 3's List had `[]zookeeper{}`). `make([]T, n)` allocates the slice with n elements at once - here the response array, filled index by index with `resp[i] = ...`. This projection loop (domain -> response, one line per element) is a shape you will write forever.

```go
func (h *zookeeperHandler) get(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	zk, err := h.svc.Get(c.Request.Context(), id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, zk.toResponse())
}
```

- The `parseID` gate pattern: ask, bail if not ok. Then the get-then-map. Note `pgx.ErrNoRows` appears *in the handler*: the service passed the database's "no rows" error straight through, so recognizing it needs the driver's own sentinel.

```go
func (h *zookeeperHandler) update(c *gin.Context) {
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
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	zku := zookeeperUpdate{Username: req.Username, Role: req.Role}
	if req.Password != nil {
		// Stage 3 stores it raw; Stage 4 hashes here.
		raw := *req.Password
		zku.PasswordHash = &raw
	}

	zk, err := h.svc.Update(c.Request.Context(), id, zku)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			c.JSON(http.StatusConflict, gin.H{"error": "username already taken"})
			return
		}
		if errors.Is(err, pgx.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, zk.toResponse())
}
```

- The request struct's fields are `*string` - same pointers-as-absent trick, now on the decode side. JSON `{"role":"admin"}` fills `Role`; `{"role":""}` fills an empty-string pointer; absent leaves `nil`. That is exactly the vocabulary `zookeeperUpdate` and `COALESCE` speak.
- `if req.Password != nil { raw := *req.Password; zku.PasswordHash = &raw }` - two pointer moves in two lines: `*req.Password` *dereferences* (reads the string a pointer points at), and `&raw` takes the address of the local so the update struct gets its own pointer. Why copy-then-point, instead of `zku.PasswordHash = req.Password`? Clarity of ownership, mostly, and a habit that will pay off in Stage 4: the *handler* is about to transform this value (hash it) before anyone stores it; keeping the raw input in a named local makes the transform natural. (Direct assignment would also compile.)

```go
func (h *zookeeperHandler) delete(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	if err := h.svc.Delete(c.Request.Context(), id); err != nil {
		if errors.Is(err, errZookeeperNotFound) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.Status(http.StatusNoContent)
}
```

- **`c.Status(http.StatusNoContent)`** writes the 204 with no body - REST's "deleted, nothing to say". `err != nil` inside the `if` - this is the error convention compressed into the condition itself.

The file's final import list (update your imports to match; the listing shows
the exact set):

```go
import (
	"errors"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/jackc/pgx/v5"
)
```

> **Wait, the handler imports pgx?** `errors.Is(err, pgx.ErrNoRows)` in the handler means the HTTP layer knows about the database driver. It works and it is honest about a middle stage, but Stage 9 replaces this: the service will translate "no rows" into its own error and the handler will stop importing pgx entirely. Note the flaw now and appreciate the fix later.

##### The complete zookeepers_handler.go

```go
package main

import (
	"errors"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/jackc/pgx/v5"
)

// zookeeperResponse is what the outside world sees. Note what is missing:
// PasswordHash. Responses are a separate shape from domain structs, and the
// password hash is the reason.
type zookeeperResponse struct {
	ID        int64     `json:"id"`
	Username  string    `json:"username"`
	Role      string    `json:"role"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

func (z zookeeper) toResponse() zookeeperResponse {
	return zookeeperResponse{
		ID:        z.ID,
		Username:  z.Username,
		Role:      z.Role,
		CreatedAt: z.CreatedAt,
		UpdatedAt: z.UpdatedAt,
	}
}

// parseID reads the :id path parameter, writing a 400 and returning false
// if it is not an integer. Small repetition killers like this are the first
// sign your "flat" files need Stage 5.
func parseID(c *gin.Context) (int64, bool) {
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "id must be an integer"})
		return 0, false
	}
	return id, true
}

type zookeeperHandler struct {
	svc *zookeeperService
}

func newZookeeperHandler(svc *zookeeperService) *zookeeperHandler {
	return &zookeeperHandler{svc: svc}
}

func (h *zookeeperHandler) create(c *gin.Context) {
	var req struct {
		Username string `json:"username" binding:"required"`
		Password string `json:"password" binding:"required"`
		Role     string `json:"role"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	zk, err := h.svc.Create(c.Request.Context(), req.Username, req.Password, req.Role)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			c.JSON(http.StatusConflict, gin.H{"error": "username already taken"})
			return
		}
		if errors.Is(err, errZookeeperInvalid) {
			c.JSON(http.StatusBadRequest, gin.H{"error": "username and password are required"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusCreated, zk.toResponse())
}

func (h *zookeeperHandler) list(c *gin.Context) {
	zks, err := h.svc.List(c.Request.Context())
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	resp := make([]zookeeperResponse, len(zks))
	for i, zk := range zks {
		resp[i] = zk.toResponse()
	}
	c.JSON(http.StatusOK, resp)
}

func (h *zookeeperHandler) get(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	zk, err := h.svc.Get(c.Request.Context(), id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, zk.toResponse())
}

func (h *zookeeperHandler) update(c *gin.Context) {
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
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	zku := zookeeperUpdate{Username: req.Username, Role: req.Role}
	if req.Password != nil {
		// Stage 3 stores it raw; Stage 4 hashes here.
		raw := *req.Password
		zku.PasswordHash = &raw
	}

	zk, err := h.svc.Update(c.Request.Context(), id, zku)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			c.JSON(http.StatusConflict, gin.H{"error": "username already taken"})
			return
		}
		if errors.Is(err, pgx.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.JSON(http.StatusOK, zk.toResponse())
}

func (h *zookeeperHandler) delete(c *gin.Context) {
	id, ok := parseID(c)
	if !ok {
		return
	}

	if err := h.svc.Delete(c.Request.Context(), id); err != nil {
		if errors.Is(err, errZookeeperNotFound) {
			c.JSON(http.StatusNotFound, gin.H{"error": "zookeeper not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}

	c.Status(http.StatusNoContent)
}
```

### 3.1 Wire the routes: replace main.go

Replace `main.go` completely. Most of it is Stage 2's pattern now applied in sequence; the new piece is `healthHandler`, a function *returning* a function:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/jackc/pgx/v5/pgxpool"

	"zoo/internal/platform/config"
	"zoo/internal/platform/database"
)

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

	zkRepo := newZookeeperRepository(pool)
	zkSvc := newZookeeperService(zkRepo)
	zkHandler := newZookeeperHandler(zkSvc)

	router := gin.Default()

	router.GET("/healthz", healthHandler(pool))

	zk := router.Group("/api/v1/zookeepers")
	{
		zk.POST("", zkHandler.create)
		zk.GET("", zkHandler.list)
		zk.GET("/:id", zkHandler.get)
		zk.PUT("/:id", zkHandler.update)
		zk.DELETE("/:id", zkHandler.delete)
	}

	router.GET("/api/v1/animals", getAnimals)
	router.GET("/api/v1/animals/:id", getAnimalByID)
	router.POST("/api/v1/animals", postAnimal)

	if err := router.Run("localhost:" + cfg.Port); err != nil {
		return fmt.Errorf("run server: %w", err)
	}
	return nil
}

// healthHandler is a function *returning* a handler function, a closure over
// the pool. This is the same shape auth middleware uses in Stage 4.
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

New and worth pausing on:

- **The wiring block**: `zkRepo := newZookeeperRepository(pool)` then the service, then the handler - one line each, arguments exactly one previous object. **This is dependency injection**: each layer receives what it needs at construction, bottom-up. Stage 5 moves these three lines' shape into `cmd/apiserver/main.go` and reuses them verbatim; Stage 11 makes the middle parameter an interface so tests can substitute.
- **`router.Group("/api/v1/zookeepers")`** creates a sub-router with the prefix all five routes share; the `{...}` block around the calls is Go's ordinary block statement (parentheses-shaped clarity for grouping - it compiles identically without the braces, but nearly all route trees in gin code use it).
- **`healthHandler(pool)`** - look carefully: it is *called* here (it takes the pool - gin never knows the pool or the ping) and what *it returns* is the func that gets registered. The returned function "closes over" `pool`: the pool variable outlives this call because the closure keeps using it. A **closure** is a function plus the variables it captured; this is the exact mechanism Stage 4's auth middleware uses, so the shape (`func(pool) gin.HandlerFunc { return func(c *gin.Context) {...} }`) is worth memorizing now.
- **`slog.Error("startup failed", "error", err)`**: `log/slog` is Go's structured logging - key/value pairs, machine-greppable. It appears (in main only) a stage before Stage 9 formalizes it; the `error, err` pairing is Go's convention for `key, value ...` variadic call sites. (The `animals` handlers below in the same file are unchanged from Stage 1; not shown again here - they appear under `run` exactly as in Stage 1.3 and Stage 1's complete listing.)

The Stage 1 animal handler functions (`getAnimals`, `getAnimalByID`, `postAnimal`) and the `animal` struct / `animals` slice **stay in `main.go`, unchanged below `run`**. Your root directory now has four `.go` files, all `package main`: `main.go`, `zookeepers_repository.go`, `zookeepers_service.go`, `zookeepers_handler.go`. That flat-and-growing feeling is the setup for Stage 5.

### 3.2 Verify: a real CRUD round-trip

```bash
go run .

# create (watch the Content-Type header; see the Stage 1 gotcha)
curl -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"giraffe-lane","role":"admin"}'
# {"id":1,"username":"maya","role":"admin","created_at":"2026-...","updated_at":"2026-..."}
# notice: no password anywhere in the response

# the other two
curl -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"sam","password":"elephant-road"}'
# {"id":2,"username":"sam","role":"keeper",...}

curl http://localhost:8080/api/v1/zookeepers
# [ ...maya..., ...sam... ]

curl http://localhost:8080/api/v1/zookeepers/1
# {"id":1,"username":"maya",...}

# update: change only the role, keep everything else
curl -X PUT http://localhost:8080/api/v1/zookeepers/2 \
  -H "Content-Type: application/json" \
  -d '{"role":"admin"}'
# {"id":2,"username":"sam","role":"admin",...}

# missing animal... er, zookeeper
curl http://localhost:8080/api/v1/zookeepers/99
# {"error":"zookeeper not found"} with status 404

# delete is 204: success, deliberately no body
curl -X PUT http://localhost:8080/api/v1/zookeepers/2 \
  -H "Content-Type: application/json" \
  -d '{"role":"keeper"}' > /dev/null   # reset sam to keeper for Stage 4
curl -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"temp","password":"x"}' > /dev/null
curl -X DELETE http://localhost:8080/api/v1/zookeepers/3 -i
# HTTP/1.1 204 No Content

# duplicate username: conflict, not a crash. Deliberately LAST in this
# script: a *failed* INSERT still consumes an identity value, so if you
# ran this attempt right after maya, sam would have been id 3 and temp
# id 4 and every later expected id would quietly lie. (Failed inserts
# burning sequence values is normal identity behavior; Stage 4.6 shows
# the TRUNCATE ... RESTART IDENTITY reset.)
curl -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"x"}'
# {"error":"username already taken"} with status 409
```

In the database:

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo \
  -c 'SELECT id, username, password_hash, role FROM zookeepers ORDER BY id'
```

You will see sam's and maya's **passwords sitting in the password_hash column as plain text**. That is Stage 3's deliberate, temporary flaw. Stage 4's first move is fixing exactly this, and the fix (bcrypt) does not change a single SQL statement: the layers did their job.

---

[Stage 2](02-postgres-config-pool-migrations.md)  ·  [Overview](../tutorial.md)  ·  [Stage 4](04-auth-bcrypt-jwt.md)