## Stage 3: The zookeeper domain, layered against the database

The API server is still one directory of files in `package main`, but zookeeper accounts now live in Postgres and touch three layers:

- **repository** - the only place with SQL. Knows the table, returns domain structs.
- **service** - business rules (this stage: valid username, unique username). Calls the repository.
- **handler** - decodes HTTP, asks the service, writes a status code. Calls the service.

Why three layers for what is still "insert a row"? Because it is cheap now, while the rules are trivial, and because the layer *seams* are what Stages 4-9 bolt rules onto. The test in Stage 11 exists only because of this split. In Stage 5 the trio moves to `internal/zookeepers/` and is otherwise unchanged.

As in Stage 1, each file is built piece by piece with explanations, and every section ends with the complete file so you can check your assembly.

**Where these files go:** all three new files live in the **project root**, in the same directory as `main.go`, and all three declare `package main` like it does. Nothing moves into a subdirectory in this stage. The root ends up holding four `.go` files - `main.go`, `zookeepers_repository.go`, `zookeepers_service.go`, `zookeepers_handler.go` - and Stage 5 is the stage that finally splits them into `internal/` packages. Create each file as you reach its section.

### The repository: zookeepers_repository.go

`zookeepers_repository.go`, in the project root:

The repository is the only file that knows SQL exists. Everything it does: turn SQL rows into Go structs, and structs into rows.

#### Package, import, and the domain type

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

#### The repository struct and its constructor

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

### Repository.Create

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

### Repository.Get

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
- **`var zk zookeeper`: why `var` here and `:=` everywhere else?** `var` only *declares*, initializing the variable to the type's **zero value** - here a `zookeeper` with every field empty (no initializer needed). `:=` *declares and assigns in one step*, so it needs a value on the right. The choice reduces to "do I have a value yet?". Here we do not: `Scan` writes into the struct's fields through the pointers two lines down, so `var` is the honest spelling. (`zk := zookeeper{}` would compile and behave identically; it just reads as "here is a value" when the value does not exist until `Scan` runs.) Two rules that settle the rest of the codebase: `var` is legal at package scope and `:=` is not (which is why the service's sentinel errors below use a `var (...)` block), and `var x int64` is how you state a type that the initializer would not have chosen (`x := 0` gives an `int`).
- Here is a fact the later stages build on: when **no row matches**, `Scan` returns **`pgx.ErrNoRows`**. "Not found" is a *normal error return*, not a panic and not `nil`. Stage 3's handler checks for it explicitly with `errors.Is` (see the pgx-notes callout after the handler); Stage 9 moves the check into the service.
- The `SELECT` column list is spelled out instead of `SELECT *`: two reasons real code does this. It makes the Scan-order contract visible, and adding a column later cannot silently change row shape in ways that desync the Scan.

### Repository.List: the multi-row loop

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
- **`zks := []zookeeper{}`: why the literal and not `var zks []zookeeper`?** Because those two lines are not the same, and the difference is exactly what this function returns. `var zks []zookeeper` **creates no slice at all** - it is the nil slice: a zero-length slice header with no backing array. `[]zookeeper{}` is an empty but *non-nil* slice. Both report `len 0 cap 0`, and every slice operation treats them interchangeably (`range` iterates zero times; `append` works and allocates its array on first use; `copy` copies nothing). They diverge in exactly two places: `x == nil` (and `reflect.DeepEqual`), and JSON serialization, where nil becomes `null` and empty becomes `[]`.

  That second one is the whole reason for the literal here, because this slice goes straight out as the HTTP response body. With no zookeepers in the table, `var` would answer `null` while the literal answers `[]` - and a client doing `const zks = await res.json(); zks.map(...)` blows up on the first. Note *when* it bites, since it is easy to conclude the choice is academic: only when the result set is **empty**. If any row exists, `append` returns a non-nil slice either way and both spellings produce `[ ... ]`. The no-rows case is precisely what a first-time user hits against a fresh database.
- Worth knowing that the mainstream advice points the other way: Google's Go style guide prefers `var t []string` for a local slice, because nil and empty being indistinguishable is normally a *feature* - an API should not force callers to care. That is right for a slice that stays inside the program. A slice that becomes a JSON response body is the standard exception, and it is why this line is a literal while `var zk zookeeper` two functions up is not.
- **House rule for the rest of this tutorial**, so you never have to re-derive it: use `var xs []T` as the default, and reach for the `[]T{}` literal only when the slice is going out as a JSON array. The rule is about *where the value goes*, not about it being a list. (Maps are the exception to the exception: a nil map is readable but writing to it panics with "assignment to entry in nil map", so any map you assign into must be `make`d or literal-constructed, and nil maps also serialize as `null`.)

### Repository.Update: partial updates with COALESCE

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

### Repository.Delete: Exec and RowsAffected

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

#### The complete zookeepers_repository.go

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

### The service: zookeepers_service.go

`zookeepers_service.go`, in the project root (beside `main.go` and `zookeepers_repository.go`):

The service is the business-rules layer. It decides what counts as valid, what a database error *means* in domain terms, and what callers may not do. It never writes SQL, and it never knows about HTTP.

#### Sentinel errors

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

#### The update-input type: pointers as "absent"

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

#### The service struct, constructor, and Create

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

### The other service methods

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

#### The helper: isUniqueViolation

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

#### The complete zookeepers_service.go

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

### The handler: zookeepers_handler.go

`zookeepers_handler.go`, in the project root (the third and last new file of this stage):

The handler is the HTTP edge: it decodes requests, writes responses, and translates service errors to statuses. New Go concepts land here: a method with a value receiver, response DTOs, and `errors.Is`.

#### The response type and the first value receiver

```go
package main

import (
	"net/http"
	"strconv"
	"time"

	"github.com/go-chi/chi/v5"
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
- The import list is the first thing the stack change rewrites: `gin` is gone, and `github.com/go-chi/chi/v5` takes its place. Nothing else in this file's imports survives untouched from the gin version except `net/http`, `strconv` and `time` - and `net/http` is now load-bearing in a way it was not before, because the handler signatures themselves are built from it.

#### parseID: the (value, ok) helper

```go
// parseID reads the {id} path parameter, writing a 400 and returning false
// if it is not an integer. Small repetition killers like this are the first
// sign your "flat" files need Stage 5.
func parseID(w http.ResponseWriter, r *http.Request) (int64, bool) {
	id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "id must be an integer"})
		return 0, false
	}
	return id, true
}
```

- Nothing new in the mechanics (ParseInt from Stage 1, scoped `if`). What is new is the **`(int64, bool)` return shape**: "the value, and whether producing it succeeded". It is Go's second standard return-pair (the first being `(value, error)`); this one is for helpers that already *handled* the failure themselves (wrote the 400) and only need to tell their caller "do not continue". The caller pattern below is `id, ok := parseID(w, r); if !ok { return }`.
- **It now takes `(w, r)`** and hands both back to the caller's control flow through its return values. With gin the helper needed only `c`, because `c` carried the response writer, the request, the params and the JSON helper all at once - one framework object standing in for three stdlib ones. With chi, `w` and `r` are separate again, and this signature is what "the handler is an ordinary `net/http` handler" looks like one level down.
- **`chi.URLParam(r, "id")`** is the replacement for `c.Param("id")`; the value still lives in the request's context (Stage 1 taught that), and this function is how you read it without touching the context API directly.
- **`writeJSON(w, ...)`** rather than a framework's `c.JSON`: the helper from Stage 1, defined at the bottom of `main.go`, available to every file in `package main` for as long as the flat era lasts.
- Note what it saves: every `{id}` handler in this and every later file calls it once instead of four repeated lines.

#### The handler struct and create

```go
type zookeeperHandler struct {
	svc *zookeeperService
}

func newZookeeperHandler(svc *zookeeperService) *zookeeperHandler {
	return &zookeeperHandler{svc: svc}
}

func (h *zookeeperHandler) create(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Username string `json:"username"`
		Password string `json:"password"`
		Role     string `json:"role"`
	}
	if err := decodeJSON(r, &req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": err.Error()})
		return
	}

	zk, err := h.svc.Create(r.Context(), req.Username, req.Password, req.Role)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			writeJSON(w, http.StatusConflict, map[string]string{"error": "username already taken"})
			return
		}
		if errors.Is(err, errZookeeperInvalid) {
			writeJSON(w, http.StatusBadRequest, map[string]string{"error": "username and password are required"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	writeJSON(w, http.StatusCreated, zk.toResponse())
}
```

- **The signature**: `func (h *zookeeperHandler) create(w http.ResponseWriter, r *http.Request)`. A method value on a receiver, with the standard library's handler signature - exactly what chi registers. Nothing about `h` changes; only the shape of the two parameters does, and they are the same two the Stage 1 handlers already took.
- **The anonymous request struct**: `var req struct {...}` - a struct type declared inline, used once. Go allows (and web handlers love) this: fields + JSON tags + no name. In the gin version `Username` and `Password` also carried `binding:"required"`, a gin-specific tag that the binder inspected *while decoding* and used to reject the request when the field was absent. chi has no binder, so the tag is deleted outright - and nothing is lost, because "a zookeeper must have a username and a password" is a business rule, and it already lives one layer down: `Create` rejects an empty username or password with `errZookeeperInvalid`, which the branch below turns into the 400. Deleting the tag did not delete the check, it moved the check to the layer that owns it.
- **`decodeJSON(r, &req)`** replaces `c.ShouldBindJSON(&req)`. It is `main.go`'s body-decoding helper, added in 3.1 with the rest of that file's rewrite; note the `&` - same pass-the-pointer-so-the-callee-can-write pattern as Stage 1's `json.NewDecoder(r.Body).Decode(&newAnimal)`. Everything gin's binder did beyond decoding (the tag validation) is now the service's job, as above.
- **`r.Context()`** - here is the context plumbing, finally visible: `*http.Request` carries a real `context.Context` for the request's lifetime, and `r.Context()` hands it to you. gin's `c.Request` was the same `*http.Request`; `r` simply *is* it now, with no wrapper in between. Handlers pass that context into services and repositories, and Stage 4 will hang the caller's identity off it. (Stage 7 explains what it enables; for now, copy the habit: never invent a context inside a handler, take it from the request.)
- **`errors.Is(err, errZookeeperDuplicate)`**: the reader for sentinel errors. `==` would also work for a directly-returned sentinel, but `errors.Is` is the norm because it understands wrapped errors (`%w`). It is how the handler recognizes the service's vocabulary without sharing any types beyond the sentinels.
- **Status mapping, in full**: 409 Conflict for "username exists" (the REST-canonical code for resource-state conflicts), 400 for malformed input, 500 for everything else. The 500 case: the error reached the handler but matched nothing known - the handler must not leak it to the client ("internal error" only), and real logging of it arrives in Stage 9.
- Response bodies stay flat (`map[string]string{"error": "..."}`, which is what `gin.H{...}` always was underneath - a map literal with one key); Stage 9 gives them a consistent envelope.

### list, get, update, delete

```go
func (h *zookeeperHandler) list(w http.ResponseWriter, r *http.Request) {
	zks, err := h.svc.List(r.Context())
	if err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	resp := make([]zookeeperResponse, len(zks))
	for i, zk := range zks {
		resp[i] = zk.toResponse()
	}
	writeJSON(w, http.StatusOK, resp)
}
```

- **`make([]T, n)`**: second slice constructor (Stage 1 had a literal, Stage 3's List had `[]zookeeper{}`). `make([]T, n)` allocates the slice with n elements at once - here the response array, filled index by index with `resp[i] = ...`. This projection loop (domain -> response, one line per element) is a shape you will write forever.
- Note there is no `r` use in this body. An unused *parameter* is fine in Go; only unused local variables are errors, and the signature is fixed by what chi registers.

```go
func (h *zookeeperHandler) get(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	zk, err := h.svc.Get(r.Context(), id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			writeJSON(w, http.StatusNotFound, map[string]string{"error": "zookeeper not found"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	writeJSON(w, http.StatusOK, zk.toResponse())
}
```

- The `parseID` gate pattern: ask, bail if not ok. Then the get-then-map. Note `pgx.ErrNoRows` appears *in the handler*: the service passed the database's "no rows" error straight through, so recognizing it needs the driver's own sentinel.

```go
func (h *zookeeperHandler) update(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	var req struct {
		Username *string `json:"username"`
		Password *string `json:"password"`
		Role     *string `json:"role"`
	}
	if err := decodeJSON(r, &req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": err.Error()})
		return
	}

	zku := zookeeperUpdate{Username: req.Username, Role: req.Role}
	if req.Password != nil {
		// Stage 3 stores it raw; Stage 4 hashes here.
		raw := *req.Password
		zku.PasswordHash = &raw
	}

	zk, err := h.svc.Update(r.Context(), id, zku)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			writeJSON(w, http.StatusConflict, map[string]string{"error": "username already taken"})
			return
		}
		if errors.Is(err, pgx.ErrNoRows) {
			writeJSON(w, http.StatusNotFound, map[string]string{"error": "zookeeper not found"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	writeJSON(w, http.StatusOK, zk.toResponse())
}
```

- The request struct's fields are `*string` - same pointers-as-absent trick, now on the decode side. JSON `{"role":"admin"}` fills `Role`; `{"role":""}` fills an empty-string pointer; absent leaves `nil`. That is exactly the vocabulary `zookeeperUpdate` and `COALESCE` speak.
- Note that this struct's tags never carried `binding:"required"` in any version of the tutorial, and now that all three fields are optional *by design* (absent means "leave alone"), no tag ever could have described them. The gin version's binder could not have expressed "optional, but present-means-set" either; the pointer fields and COALESCE are the mechanism, and they were always the real answer.
- `if req.Password != nil { raw := *req.Password; zku.PasswordHash = &raw }` - two pointer moves in two lines: `*req.Password` *dereferences* (reads the string a pointer points at), and `&raw` takes the address of the local so the update struct gets its own pointer. Why copy-then-point, instead of `zku.PasswordHash = req.Password`? Clarity of ownership, mostly, and a habit that will pay off in Stage 4: the *handler* is about to transform this value (hash it) before anyone stores it; keeping the raw input in a named local makes the transform natural. (Direct assignment would also compile.)

```go
func (h *zookeeperHandler) delete(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	if err := h.svc.Delete(r.Context(), id); err != nil {
		if errors.Is(err, errZookeeperNotFound) {
			writeJSON(w, http.StatusNotFound, map[string]string{"error": "zookeeper not found"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	w.WriteHeader(http.StatusNoContent)
}
```

- **`w.WriteHeader(http.StatusNoContent)`** writes the 204 with no body - REST's "deleted, nothing to say". This is the one place a handler writes a status *without* going through `writeJSON`, and deliberately: the helper encodes a value after the status line, and a 204 must not carry one. `c.Status(...)` was gin's spelling of the same idea; the stdlib spelling is `WriteHeader`.
- `err != nil` inside the `if` - this is the error convention compressed into the condition itself.

The file's final import list (update your imports to match; the listing shows
the exact set):

```go
import (
	"errors"
	"net/http"
	"strconv"
	"time"

	"github.com/go-chi/chi/v5"
	"github.com/jackc/pgx/v5"
)
```

> **Wait, the handler imports pgx?** `errors.Is(err, pgx.ErrNoRows)` in the handler means the HTTP layer knows about the database driver. It works and it is honest about a middle stage, but Stage 9 replaces this: the service will translate "no rows" into its own error and the handler will stop importing pgx entirely. Note the flaw now and appreciate the fix later.

#### The complete zookeepers_handler.go

```go
package main

import (
	"errors"
	"net/http"
	"strconv"
	"time"

	"github.com/go-chi/chi/v5"
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

// parseID reads the {id} path parameter, writing a 400 and returning false
// if it is not an integer. Small repetition killers like this are the first
// sign your "flat" files need Stage 5.
func parseID(w http.ResponseWriter, r *http.Request) (int64, bool) {
	id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "id must be an integer"})
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

func (h *zookeeperHandler) create(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Username string `json:"username"`
		Password string `json:"password"`
		Role     string `json:"role"`
	}
	if err := decodeJSON(r, &req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": err.Error()})
		return
	}

	zk, err := h.svc.Create(r.Context(), req.Username, req.Password, req.Role)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			writeJSON(w, http.StatusConflict, map[string]string{"error": "username already taken"})
			return
		}
		if errors.Is(err, errZookeeperInvalid) {
			writeJSON(w, http.StatusBadRequest, map[string]string{"error": "username and password are required"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	writeJSON(w, http.StatusCreated, zk.toResponse())
}

func (h *zookeeperHandler) list(w http.ResponseWriter, r *http.Request) {
	zks, err := h.svc.List(r.Context())
	if err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	resp := make([]zookeeperResponse, len(zks))
	for i, zk := range zks {
		resp[i] = zk.toResponse()
	}
	writeJSON(w, http.StatusOK, resp)
}

func (h *zookeeperHandler) get(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	zk, err := h.svc.Get(r.Context(), id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			writeJSON(w, http.StatusNotFound, map[string]string{"error": "zookeeper not found"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	writeJSON(w, http.StatusOK, zk.toResponse())
}

func (h *zookeeperHandler) update(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	var req struct {
		Username *string `json:"username"`
		Password *string `json:"password"`
		Role     *string `json:"role"`
	}
	if err := decodeJSON(r, &req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": err.Error()})
		return
	}

	zku := zookeeperUpdate{Username: req.Username, Role: req.Role}
	if req.Password != nil {
		// Stage 3 stores it raw; Stage 4 hashes here.
		raw := *req.Password
		zku.PasswordHash = &raw
	}

	zk, err := h.svc.Update(r.Context(), id, zku)
	if err != nil {
		if errors.Is(err, errZookeeperDuplicate) {
			writeJSON(w, http.StatusConflict, map[string]string{"error": "username already taken"})
			return
		}
		if errors.Is(err, pgx.ErrNoRows) {
			writeJSON(w, http.StatusNotFound, map[string]string{"error": "zookeeper not found"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	writeJSON(w, http.StatusOK, zk.toResponse())
}

func (h *zookeeperHandler) delete(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	if err := h.svc.Delete(r.Context(), id); err != nil {
		if errors.Is(err, errZookeeperNotFound) {
			writeJSON(w, http.StatusNotFound, map[string]string{"error": "zookeeper not found"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	w.WriteHeader(http.StatusNoContent)
}
```

### 3.1 Wire the routes: replace main.go

Replace `main.go` completely. Most of it is Stage 2's pieces assembled in sequence - configuration, logging, the pool - plus the wiring for the new layers. The new piece is `healthHandler`, a function *returning* a function:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"strconv"

	"github.com/go-chi/chi/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"zoo/internal/platform/config"
	"zoo/internal/platform/database"
	"zoo/internal/platform/logging"
)

// animal is the data we serve. The `json:"..."` tags set the field names
// encoding/json uses when serializing: without them you get Go's title-cased
// names (ID, Name, Species), which is not what JSON APIs usually do.
type animal struct {
	ID        int64  `json:"id"`
	Name      string `json:"name"`
	Species   string `json:"species"`
	Enclosure string `json:"enclosure"`
}

// In-memory data, exactly like the official tutorial's slice of albums.
// Nothing here is persistent yet: every restart forgets your animals.
var animals = []animal{
	{ID: 1, Name: "Tembo", Species: "African bush elephant", Enclosure: "Savanna"},
	{ID: 2, Name: "Suki", Species: "Sumatran tiger", Enclosure: "Jungle"},
	{ID: 3, Name: "Biscuit", Species: "Red panda", Enclosure: "Forest"},
}

func main() {
	if err := run(); err != nil {
		slog.Error("startup failed", "error", err)
		os.Exit(1)
	}
}

func run() error {
	ctx := context.Background()

	cfg, err := config.Load("")
	if err != nil {
		return fmt.Errorf("load config: %w", err)
	}
	// One call installs Stage 2's slog handler, so every log line below (and
	// writeJSON's) is structured and respects LOG_LEVEL and LOG_FORMAT.
	logging.Setup(cfg.LogLevel, cfg.LogFormat)

	pool, err := database.NewPool(ctx, cfg.DatabaseURL)
	if err != nil {
		return fmt.Errorf("open database: %w", err)
	}
	defer pool.Close()

	zkRepo := newZookeeperRepository(pool)
	zkSvc := newZookeeperService(zkRepo)
	zkHandler := newZookeeperHandler(zkSvc)

	router := chi.NewRouter()

	router.Get("/healthz", healthHandler(pool))

	router.Route("/api/v1", func(router chi.Router) {
		router.Route("/zookeepers", func(router chi.Router) {
			// All five routes are open for now; Stage 4 adds login and
			// gates the reads.
			router.Post("/", zkHandler.create)
			router.Get("/", zkHandler.list)
			router.Get("/{id}", zkHandler.get)
			router.Put("/{id}", zkHandler.update)
			router.Delete("/{id}", zkHandler.delete)
		})

		router.Get("/animals", getAnimals)
		router.Get("/animals/{id}", getAnimalByID)
		router.Post("/animals", postAnimal)
	})

	slog.Info("api server listening", "addr", "localhost:"+cfg.Port)
	if err := http.ListenAndServe("localhost:"+cfg.Port, router); err != nil {
		return fmt.Errorf("run server: %w", err)
	}
	return nil
}

// healthHandler is a function *returning* a handler function, a closure over
// the pool. This is the same shape auth middleware uses in Stage 4.
func healthHandler(pool *pgxpool.Pool) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if err := pool.Ping(r.Context()); err != nil {
			writeJSON(w, http.StatusServiceUnavailable, map[string]string{"status": "degraded", "db": "down"})
			return
		}
		writeJSON(w, http.StatusOK, map[string]string{"status": "ok", "db": "up"})
	}
}

func getAnimals(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, animals)
}

func getAnimalByID(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "id must be an integer"})
		return
	}

	for _, a := range animals {
		if a.ID == id {
			writeJSON(w, http.StatusOK, a)
			return
		}
	}

	writeJSON(w, http.StatusNotFound, map[string]string{"error": "animal not found"})
}

func postAnimal(w http.ResponseWriter, r *http.Request) {
	var newAnimal animal
	if err := decodeJSON(r, &newAnimal); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "could not parse request body"})
		return
	}

	newAnimal.ID = int64(len(animals) + 1)
	animals = append(animals, newAnimal)

	writeJSON(w, http.StatusCreated, newAnimal)
}

// writeJSON is the whole of "return JSON": set the content type, write the
// status, encode the body. Stage 5 moves it into internal/platform/httpx.
func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.Error("write json response", "error", err)
	}
}

// decodeJSON reads a JSON request body into v. Stage 5 moves it into
// internal/platform/httpx as httpx.Decode.
func decodeJSON(r *http.Request, v any) error {
	return json.NewDecoder(r.Body).Decode(v)
}
```

New and worth pausing on:

- **The wiring block**: `zkRepo := newZookeeperRepository(pool)` then the service, then the handler - one line each, arguments exactly one previous object. **This is dependency injection**: each layer receives what it needs at construction, bottom-up. Stage 5 moves this wiring into `internal/cli/serve.go` and the route table into `internal/server/router.go`, where the same three lines survive almost verbatim; Stage 11 makes the middle parameter an interface so tests can substitute.
- **`config.Load("")`**: configuration is now parsed through Stage 2's viper-backed loader, once, at startup, into a plain struct. The argument is the *explicit* config file path - what Stage 2's CLI fills from the `--config` flag. This flat `main` has no flags, so it passes the empty string, which `Load` reads as "no explicit file: use `./zoo.yaml` if it is present, and do not fail if it is not" (Stage 2.2's asymmetry). The error is non-nil only for a bad explicit path or an unparseable level, so the check here is mostly for the future Stage 4 makes real, when a missing `JWT_SECRET` becomes a startup failure.
- **`logging.Setup(cfg.LogLevel, cfg.LogFormat)`**: one line, and every `slog` call in this program - including the one inside `writeJSON` - now goes through a real handler with a level from `LOG_LEVEL` and a format from `LOG_FORMAT`. It must run *after* `config.Load` (it needs the parsed level) and before anything logs. Note that this means the startup failure path above it still logs through `slog`'s default handler; there is no way around that, and it is exactly why `main` only ever prints one thing.
- **`router := chi.NewRouter()`** and the nested `router.Route(...)` calls: each `Route(pattern, fn)` is a *sub-router* - every route declared inside gets the prefix, and the closure receives a router scoped to it. gin spelled the same idea `router.Group("/api/v1/zookeepers")` plus a `{...}` block; chi makes the scope explicit by handing you the object inside the function. The path parameters inside are `{id}`, chi's spelling (and the standard library's), not the old `:id`.
- **`healthHandler(pool)`** - look carefully: it is *called* here (it takes the pool - chi never knows the pool or the ping) and what *it returns* is the func that gets registered. The returned function "closes over" `pool`: the pool variable outlives this call because the closure keeps using it. A **closure** is a function plus the variables it captured; this is the exact mechanism Stage 4's auth middleware uses, so the shape (`func(pool *pgxpool.Pool) http.HandlerFunc { return func(w http.ResponseWriter, r *http.Request) {...} }`) is worth memorizing now. Note the return type is `http.HandlerFunc`, from the standard library - gin's `gin.HandlerFunc` was a different named type with a different parameter, and the only reason this line changed is that.
- **`decodeJSON(r, &req)` is new here.** Stage 1's `postAnimal` wrote `json.NewDecoder(r.Body).Decode(&newAnimal)` by hand, and this stage's `create` and `update` need the same two lines - the third call site is where repetition stops being noise and becomes a name. So the helper joins `writeJSON` at the bottom of the file; the animals handler now calls it too, which is the only change to the Stage 1 code in this file. Both helpers are exactly what Stage 5 moves into `internal/platform/httpx` (as `httpx.JSON` and `httpx.Decode`), which is the other reason to give them names now: they are about to become infrastructure.
- **`slog.Error("startup failed", "error", err)`**: the whole program's errors funnel up through `run`'s `error` return, and `main` is the single place that decides a nonzero exit. Nothing here logs twice and nothing calls `os.Exit` besides this one line - the same `main`/`run` split Stage 2.5 put behind cobra's `RunE`, kept for as long as this file exists.

The Stage 1 animal pieces - the `animal` type, the `animals` slice, and the three handlers - stay in `main.go`, unchanged below `run` apart from `postAnimal` decoding through the shared `decodeJSON`. They are reproduced in full in the listing above so it is a genuine, compiling file rather than a sketch with a hole in it. Your root directory now has four `.go` files, all `package main`: `main.go`, `zookeepers_repository.go`, `zookeepers_service.go`, `zookeepers_handler.go`. That flat-and-growing feeling is the setup for Stage 5.

### 3.2 Verify: a real CRUD round-trip

```bash
go run .

# create (Content-Type stays the habit even though the decoder ignores it;
# see the Stage 1 gotcha)
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
docker exec $(docker compose ps -q db) psql -U zoo -d zoo \
  -c 'SELECT id, username, password_hash, role FROM zookeepers ORDER BY id'
```

You will see sam's and maya's **passwords sitting in the password_hash column as plain text**. That is Stage 3's deliberate, temporary flaw. Stage 4's first move is fixing exactly this, and the fix (bcrypt) does not change a single SQL statement: the layers did their job.

---

[Stage 2](02-postgres-config-pool-migrations.md)  |  [Overview](../tutorial.md)  |  [Stage 4](04-auth-bcrypt-jwt.md)
