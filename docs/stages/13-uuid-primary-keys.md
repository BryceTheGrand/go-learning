## Stage 13: UUID primary keys

Through Stage 12 every id in this service was a `bigint GENERATED ALWAYS AS IDENTITY`, and Stage 2 said out loud why: small integers make `curl` examples copy-pasteable. That choice bought twelve stages of readable output. It also cost something, and Stage 2 promised to come back to it. This is that stage.

Nothing built so far was wrong. Sequential ids are a perfectly good default, and plenty of production systems run on them forever. But an id that anyone can count is an id that leaks: `GET /api/v1/animals/1`, then `2`, then `3` walks your entire table, tells the caller exactly how many animals you have, and lets them enumerate every keeper by guessing. The fix is to stop the identifier from being a number at all, and the identifier most systems reach for is a **UUID**.

There are two ways to get UUIDs into an API, and they solve different problems:

| Shape | What changes | What it costs |
|---|---|---|
| **Swap the primary key** to `uuid` (this stage) | The id *is* a UUID everywhere: columns, FKs, JSON, the JWT, the URL | 16 bytes per key, worse B-tree locality for random UUIDs, ids stop being readable |
| **Keep the `bigint` key, add a `public_id uuid`** | A second, opaque column is what the API exposes; the key stays an integer | Two identifiers per row; every lookup joins on `public_id`; internals still leak on any route that forgets |

We take the first, because the whole point is that there is no integer id left to leak, and because the migration it forces is the most instructive thing in this stage: changing a primary key that foreign keys already point at. The second shape is a real and often better answer for a large existing system - a callout in 13.3 says when to prefer it.

Two honesty notes before the code, both of which Stage 2 owed you:

- **This is the stage where the copy-pasteable examples stop being copy-pasteable.** A UUID is not something you can retype from one line to the next, so from here the examples show you the id that came *back* and expect you to paste it forward. That is what working with UUIDs is actually like, and pretending otherwise would be the tutorial lying to you.
- **The swap is not free at the storage layer.** A random (version 4) UUID has no ordering, so inserting a stream of them scatters writes across the B-tree instead of appending to the right edge - the classic reason "UUIDs made my inserts slower" shows up in production. Postgres 18 added `uuidv7()`, a *time-ordered* UUID that keeps the locality while staying globally unique. We are pinned to Postgres 17 here, whose only built-in generator is `gen_random_uuid()` (version 4), so that is what this stage uses; 13.1 explains when to reach for v7 instead.

### 13.1 What a UUID buys, and what it costs

The case for UUIDs is easiest to remember as four problems a number cannot solve:

- **Enumeration.** Identifiers that are small and dense invite walking. A UUID makes "try every id" impossible, which matters the moment a route is missing an authorization check (a `404` for a real-but-forbidden row and a `404` for a nonexistent one should look identical, and with UUIDs they at least cannot be *found* by counting).
- **Merging and sharding.** Two databases that both minted `bigint` ids will collide the day you consolidate them. UUIDs are generated independently with no coordination, so rows from any number of sources concatenate cleanly.
- **Client-generated ids.** A mobile client that has been offline can decide its own row's id, send it, and retry safely: the create is idempotent because the id already exists. With an identity column the id is the server's to assign and a retried offline create duplicates.
- **No sequence contention.** Identity columns are a single point of write serialization, and on a busy table the sequence is a lock everyone waits on.

The costs are just as real and belong in the same breath:

- **16 bytes, not 8.** Every index that includes the key grows, and every foreign key carries it.
- **Random v4 hurts insert locality**, as above. If write throughput on a hot table is the concern, use v7 (time-ordered) - generate it in Go, or move to Postgres 18 where `uuidv7()` is built in - and keep UUIDs for everything else.
- **Ids stop being debuggable.** `animal 42` is a sentence; `animal 9e351958-...` is not. You will want something human-friendly (a slug, a name, a short display code) beside the UUID for the console and the logs.
- **Ordering by `id` stops meaning "in the order they were created".** Several of the tutorial's example queries use `ORDER BY id`; with random UUIDs that ordering is arbitrary. Order by a `created_at` column instead, or use v7 if you liked that ids sorted.

The rule of thumb that falls out: **use a UUID when the identifier crosses a trust boundary or a system boundary, and do not spend the cost where it does not.** The `species` and `environments` lookup tables in the bonus material keep their `text` keys for exactly this reason - "savanna" is a stable natural key that is never guessed and never merged, so a UUID would be ceremony.

### 13.2 The type you will actually hold: `pgtype.UUID`

You do not need a new dependency for this. pgx v5 already ships `pgtype.UUID`, which is the single type you will use from the HTTP layer down to the driver:

```go
type UUID struct {
	Bytes [16]byte
	Valid bool
}
```

`Valid` is the null flag, so a nullable foreign key is a `pgtype.UUID` whose `Valid` is false - and the JSON encoding falls out for free. `pgtype.UUID` implements `MarshalJSON` and `UnmarshalJSON`: a valid value serializes as the canonical quoted string, an invalid one as `null`. It also scans a `uuid` column directly, binds as a query argument, and implements the `database/sql` `Scanner`/`Valuer` pair. Four things worth verifying once, all confirmed against pgx v5.7.4 and Postgres 17:

```
scan+json : {"id":"9e351958-c891-42df-8093-aa96ec9484e9","name":"Tembo","keeper_id":"8f28...230a"}
unmarshal : 9e351958-c891-42df-8093-aa96ec9484e9 keeper valid: false   (a null body field)
bad parse : cannot parse UUID not-a-uuid
missing   : true                                                       (unknown uuid -> pgx.ErrNoRows)
```

So the DTO is the same shape it always was, with the type changed:

```go
type Response struct {
	ID            pgtype.UUID `json:"id"`
	Name          string      `json:"name"`
	PrimaryKeeper *keeperRef  `json:"primary_keeper"`
}
```

**Nullability: this is where the type pays for itself.** Since Stage 6 the convention for an optional column has been a pointer (`*int64`, `*time.Time`), because scanning a SQL `NULL` into a non-pointer fails at runtime. `pgtype.UUID` carries its own null flag, so the optional keeper is a plain `pgtype.UUID` and needs no pointer - the pointer convention existed to carry nullness, and this type already does. That is a deliberate exception to the Stage 6 rule, not an oversight:

```go
type Animal struct {
	ID              pgtype.UUID
	Name            string
	Species         string
	Enclosure       string
	PrimaryKeeperID pgtype.UUID // Valid == false when the column is NULL
	LastFedAt       *time.Time
	CreatedAt       time.Time
	UpdatedAt       time.Time
}
```

**Parsing an id out of the URL is the one place the code genuinely changes shape.** Stage 3's `parseID` was `strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)`; a UUID parses with the type's own `Scan`, which accepts the canonical string and rejects everything else:

```go
// parseUUID reads the {id} path parameter into a pgtype.UUID. A UUID that
// will not parse is a client error (400), the same as a non-numeric id was
// in Stage 3 - the only change is what "well-formed" means.
func parseUUID(w http.ResponseWriter, r *http.Request) (pgtype.UUID, bool) {
	var id pgtype.UUID
	if err := id.Scan(chi.URLParam(r, "id")); err != nil {
		httpx.Respond(w, httpx.BadRequest("invalid id: must be a uuid"))
		return pgtype.UUID{}, false
	}
	return id, true
}
```

> **A note on `id::text`.** A query that wants the id as a Go string - a response built as a `map[string]any`, or a log field - has to say so: `SELECT id::text FROM ...`. Without the cast, pgx hands you a `pgtype.UUID` and `Scan` into a `string` fails with a type error. This is the "`id::text` projection" decision Stage 2's callout promised, and the rule is simply: scan into `pgtype.UUID` unless you have a specific reason to want text, and when you want text, cast in the SQL rather than formatting in Go.

> **If you would rather have `uuid.Parse` and `uuid.New` in your domain code**, the community standard is [`github.com/google/uuid`](https://github.com/google/uuid). pgx v5 will *not* scan a column into a `uuid.UUID` on its own - its codec only knows the `pgtype` interfaces - so you either register an adapter that teaches the type map about `google/uuid` (a small third-party module, `vgarvardt/pgx-google-uuid`, wired in through `pgxpool.Config.AfterConnect`), or convert at the seam: `pgtype.UUID{Bytes: id, Valid: true}` on the way out, `uuid.UUID(pg.Bytes)` on the way back. That conversion *is* the "non-obvious pgx scanning caveat" Stage 2 promised; it is not a bug, it is the boundary between pgx's own UUID type and everyone else's. This tutorial stays on `pgtype.UUID` to keep the dependency list where Stage 2 left it, and because one id type end to end is fewer places to get it wrong.

### 13.3 The migration: changing a key under live foreign keys

Here is the problem the rest of the stage builds on. `animals.id` is a primary key, and `feed_log.animal_id` is a foreign key pointing at it. You cannot simply `ALTER TABLE animals ALTER COLUMN id TYPE uuid` - the child column type must match the parent's, and the parent's other rows have to keep resolving to the same child rows while you convert. There is no `ALTER` for "change this key to a new value." You have to move the data.

The pattern is the **expand and contract** one, and this is the first time the tutorial needs it: add the new alongside the old, copy the values across, then drop the old. It runs in three phases, and all three are below. In a production system you would split phase 1-2 (deploy it, let the old code keep serving on the `bigint` columns) from phase 3 (deploy the code that reads UUIDs, *then* drop the old columns) - the whole reason the pattern exists is that you can only drop a column once the oldest running version of your service has stopped reading it. We run all three in one migration because the tutorial has one deploy and three rows of data; the separation is a deploy, not a syntax.

`migrations/00004_uuid_primary_keys.sql`:

```sql
-- +goose Up
-- Phase 1 (expand): a uuid column beside every bigint key. The NOT NULL
-- DEFAULT is what fills the existing rows: gen_random_uuid() is volatile, so
-- Postgres evaluates it per row and rewrites the table once.
ALTER TABLE zookeepers ADD COLUMN new_id        uuid NOT NULL DEFAULT gen_random_uuid();
ALTER TABLE animals    ADD COLUMN new_id        uuid NOT NULL DEFAULT gen_random_uuid();
ALTER TABLE animals    ADD COLUMN new_keeper_id uuid;
ALTER TABLE feed_log   ADD COLUMN new_id        uuid NOT NULL DEFAULT gen_random_uuid();
ALTER TABLE feed_log   ADD COLUMN new_animal_id uuid;
ALTER TABLE feed_log   ADD COLUMN new_keeper_id uuid;

-- Phase 2 (backfill): the child uuid columns are filled by joining on the
-- old integer keys. This is the only step that needs to know both worlds.
UPDATE animals  a SET new_keeper_id = z.new_id FROM zookeepers z WHERE z.id = a.primary_keeper_id;
UPDATE feed_log f SET new_animal_id = a.new_id FROM animals    a WHERE a.id = f.animal_id;
UPDATE feed_log f SET new_keeper_id = z.new_id FROM zookeepers z WHERE z.id = f.keeper_id;

-- Phase 3 (contract): drop every constraint that names the old columns,
-- promote the uuid columns, then drop the integer columns. FKs first, in
-- dependency order, or the drop of a referenced PK is refused.
ALTER TABLE animals  DROP CONSTRAINT animals_primary_keeper_id_fkey;
ALTER TABLE feed_log DROP CONSTRAINT feed_log_animal_id_fkey;
ALTER TABLE feed_log DROP CONSTRAINT feed_log_keeper_id_fkey;
ALTER TABLE zookeepers DROP CONSTRAINT zookeepers_pkey;
ALTER TABLE animals    DROP CONSTRAINT animals_pkey;
ALTER TABLE feed_log   DROP CONSTRAINT feed_log_pkey;

ALTER TABLE animals  DROP COLUMN primary_keeper_id;
ALTER TABLE feed_log DROP COLUMN animal_id;
ALTER TABLE feed_log DROP COLUMN keeper_id;
ALTER TABLE zookeepers DROP COLUMN id;
ALTER TABLE animals    DROP COLUMN id;
ALTER TABLE feed_log   DROP COLUMN id;

-- The staging columns take the real names only now, once nothing else can
-- be confused for the key.
ALTER TABLE zookeepers RENAME COLUMN new_id        TO id;
ALTER TABLE animals    RENAME COLUMN new_id        TO id;
ALTER TABLE animals    RENAME COLUMN new_keeper_id TO primary_keeper_id;
ALTER TABLE feed_log   RENAME COLUMN new_id        TO id;
ALTER TABLE feed_log   RENAME COLUMN new_animal_id TO animal_id;
ALTER TABLE feed_log   RENAME COLUMN new_keeper_id TO keeper_id;

ALTER TABLE zookeepers ADD PRIMARY KEY (id);
ALTER TABLE animals    ADD PRIMARY KEY (id);
ALTER TABLE feed_log   ADD PRIMARY KEY (id);

-- A dropped column takes its constraints with it, and that includes NOT NULL.
-- Re-state it, or the UUID foreign keys are quietly nullable where the bigint
-- ones were not - a bug that only shows up when a NULL slips in months later.
ALTER TABLE feed_log ALTER COLUMN animal_id SET NOT NULL;
ALTER TABLE feed_log ALTER COLUMN keeper_id SET NOT NULL;

ALTER TABLE animals  ADD CONSTRAINT animals_primary_keeper_id_fkey
    FOREIGN KEY (primary_keeper_id) REFERENCES zookeepers(id) ON DELETE SET NULL;
ALTER TABLE feed_log ADD CONSTRAINT feed_log_animal_id_fkey
    FOREIGN KEY (animal_id) REFERENCES animals(id) ON DELETE CASCADE;
ALTER TABLE feed_log ADD CONSTRAINT feed_log_keeper_id_fkey
    FOREIGN KEY (keeper_id) REFERENCES zookeepers(id) ON DELETE RESTRICT;

-- The index went down with the column it named; put it back.
CREATE INDEX feed_log_animal_idx ON feed_log (animal_id, fed_at DESC);

-- +goose Down
-- The mirror image, and the caveat that makes a key change different from
-- most migrations: the integer ids are gone, so this rebuilds bigint identity
-- columns and hands out *fresh* numbers. The relationships survive, because
-- they are re-derived by joining through the UUIDs before those are dropped,
-- but the old numbers do not come back. Anything outside the database that
-- stored an id - a bookmark, a cached URL - now points at nothing.
ALTER TABLE zookeepers ADD COLUMN old_id bigint GENERATED ALWAYS AS IDENTITY;
ALTER TABLE animals    ADD COLUMN old_id bigint GENERATED ALWAYS AS IDENTITY;
ALTER TABLE animals    ADD COLUMN old_keeper_id bigint;
ALTER TABLE feed_log   ADD COLUMN old_id bigint GENERATED ALWAYS AS IDENTITY;
ALTER TABLE feed_log   ADD COLUMN old_animal_id bigint;
ALTER TABLE feed_log   ADD COLUMN old_keeper_id bigint;

UPDATE animals  a SET old_keeper_id = z.old_id FROM zookeepers z WHERE z.id = a.primary_keeper_id;
UPDATE feed_log f SET old_animal_id = a.old_id FROM animals    a WHERE a.id = f.animal_id;
UPDATE feed_log f SET old_keeper_id = z.old_id FROM zookeepers z WHERE z.id = f.keeper_id;

ALTER TABLE animals  DROP CONSTRAINT animals_primary_keeper_id_fkey;
ALTER TABLE feed_log DROP CONSTRAINT feed_log_animal_id_fkey;
ALTER TABLE feed_log DROP CONSTRAINT feed_log_keeper_id_fkey;
ALTER TABLE zookeepers DROP CONSTRAINT zookeepers_pkey;
ALTER TABLE animals    DROP CONSTRAINT animals_pkey;
ALTER TABLE feed_log   DROP CONSTRAINT feed_log_pkey;

ALTER TABLE animals  DROP COLUMN primary_keeper_id;
ALTER TABLE feed_log DROP COLUMN animal_id;
ALTER TABLE feed_log DROP COLUMN keeper_id;
ALTER TABLE zookeepers DROP COLUMN id;
ALTER TABLE animals    DROP COLUMN id;
ALTER TABLE feed_log   DROP COLUMN id;

ALTER TABLE zookeepers RENAME COLUMN old_id        TO id;
ALTER TABLE animals    RENAME COLUMN old_id        TO id;
ALTER TABLE animals    RENAME COLUMN old_keeper_id TO primary_keeper_id;
ALTER TABLE feed_log   RENAME COLUMN old_id        TO id;
ALTER TABLE feed_log   RENAME COLUMN old_animal_id TO animal_id;
ALTER TABLE feed_log   RENAME COLUMN old_keeper_id TO keeper_id;

ALTER TABLE zookeepers ADD PRIMARY KEY (id);
ALTER TABLE animals    ADD PRIMARY KEY (id);
ALTER TABLE feed_log   ADD PRIMARY KEY (id);
ALTER TABLE feed_log   ALTER COLUMN animal_id SET NOT NULL;
ALTER TABLE feed_log   ALTER COLUMN keeper_id SET NOT NULL;

ALTER TABLE animals  ADD CONSTRAINT animals_primary_keeper_id_fkey
    FOREIGN KEY (primary_keeper_id) REFERENCES zookeepers(id) ON DELETE SET NULL;
ALTER TABLE feed_log ADD CONSTRAINT feed_log_animal_id_fkey
    FOREIGN KEY (animal_id) REFERENCES animals(id) ON DELETE CASCADE;
ALTER TABLE feed_log ADD CONSTRAINT feed_log_keeper_id_fkey
    FOREIGN KEY (keeper_id) REFERENCES zookeepers(id) ON DELETE RESTRICT;

CREATE INDEX feed_log_animal_idx ON feed_log (animal_id, fed_at DESC);
```

Run it, and the data is intact - the joins survived the move:

```
docker exec $(docker compose ps -q db) psql -U zoo -d zoo -c '\d animals'
```

```
 name              | text                     | not null
 species           | text                     | not null
 enclosure         | text                     | not null
 last_fed_at       | timestamp with time zone |
 created_at        | timestamp with time zone | not null default now()
 updated_at        | timestamp with time zone | not null default now()
 id                | uuid                     | not null default gen_random_uuid()
 primary_keeper_id | uuid                     |
Indexes:
    "animals_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "animals_primary_keeper_id_fkey" FOREIGN KEY (primary_keeper_id)
        REFERENCES zookeepers(id) ON DELETE SET NULL
```

> **The order above is odd on purpose, and reading it is worth ten seconds.** Postgres stores columns in creation order and `ADD COLUMN` always appends, so the two uuid columns arrive *after* `updated_at`; `RENAME COLUMN` moves a name, not a position. That is why `id` is not first any more. Nothing depends on column order in Postgres - only `SELECT *` can even see it - so it is cosmetic, and the alternative (a table rebuild to "fix" it) costs a full rewrite for no behaviour change. In engines where order *is* meaningful, this is exactly the step that would force the rebuild; here it is a reminder that the expand-and-contract pattern leaves fingerprints.

and the relationships are exactly what they were, now spelled in UUIDs:

```
SELECT a.name, a.species, z.username AS keeper
FROM animals a LEFT JOIN zookeepers z ON z.id = a.primary_keeper_id ORDER BY a.name;

  name  |        species        | keeper
--------+-----------------------+--------
 Rex   | Cocker spaniel        |
 Tembo | African bush elephant | sam
```

Those are the two animals your database has held since Stage 8: Tembo, kept by sam since you reassigned him, and Rex, who has no keeper. The join is unchanged; only the types it is joining on have changed, and the values it prints are the ones it printed before.

The foreign keys still *enforce*, which is the part that matters and the part you should not take on faith. Deleting a keeper who has feed history still trips `RESTRICT`. Verified by trying to delete sam, who fed Tembo in Stage 8 and is also Tembo's primary keeper:

```
ERROR:  update or delete on table "zookeepers" violates foreign key constraint
        "feed_log_keeper_id_fkey" on table "feed_log"
DETAIL:  Key (id)=(<sam's id>) is still referenced from table "feed_log".
```

(`<sam's id>` is whatever `SELECT id FROM zookeepers WHERE username = 'sam'` returns for you; the point is which constraint fires, not the value.) The `SET NULL` path is the same constraint read the other way, and you can exercise it with a keeper you make for the purpose: create one, make it some animal's primary keeper without feeding anything as it, then delete it - `animals.primary_keeper_id` becomes NULL rather than blocking the delete.

Four things in that migration are the ones worth carrying away:

- **Order is not cosmetic.** Foreign keys are dropped before the primary keys they reference, and the staging columns are renamed only after the old ones are gone. Get the order wrong and Postgres refuses with `cannot drop constraint ... because other objects depend on it`, which is the database protecting you from a half-migrated schema.
- **A dropped column silently drops its `NOT NULL`.** This is the trap in the whole exercise: `feed_log.animal_id` was `NOT NULL`, and dropping the column to replace it means the replacement is nullable unless you say otherwise. The `SET NOT NULL` lines above are not tidiness; without them the migration "succeeds" and leaves a subtler schema than it found.
- **`DEFAULT gen_random_uuid()` is doing the backfill** for the rows that need a wholly new value, while the `UPDATE ... FROM` statements are doing it for the rows whose relationship must be preserved. You need both, and mixing them up (assigning the child a fresh UUID instead of copying the parent's) would break every join while looking fine.
- **`gen_random_uuid()` needs no extension** on Postgres 13 or later; it is in core. If you are on an older server you are reaching for `pgcrypto`, which is a migration of its own.

> **When you would *not* swap the key.** In a large system with the key already woven through dozens of tables and services, changing the primary key is a lot of risk for the benefit, and the better move is the second row of the table at the top: leave `bigint` keys alone and add `public_id uuid NOT NULL UNIQUE DEFAULT gen_random_uuid()` to the entity tables, expose *that* in the API and the URLs, and keep the integer key internal and hidden. It is purely additive (no FK dance, no dropped columns), it is reversible, and it achieves the security goal - a caller cannot enumerate what they cannot see. The cost is a second identifier and a rule ("look up by `public_id`, never by `id`") that a reviewer has to remember. Swapping the key earns its risk when the schema is young, when you are consolidating databases anyway, or when you want *one* identifier rather than two.

### 13.4 The sweep through the code

The migration changes the database; the compiler now walks you through the code. This is the payoff of typing the id consistently: `go build ./...` fails at every site that still assumes `int64` in a *signature*, and there is no guessing which ones those are.

There is exactly one place it will not fail, and it is worth knowing before you meet it. Stage 6's `?keeper_id=` filter is an id that never appears in a signature that the compiler cares about: `List(ctx, keeperID *int64)` still compiles after the column becomes a `uuid`, `pgx` takes the Go value without a murmur, and the error does not arrive until the query runs:

```
failed to encode args[0]: unable to encode 2 into binary format for uuid (OID 2950): cannot find encode plan
```

(The `2` is the `?keeper_id=` value you sent - Stage 6's filter binds `*keeperID`, the dereferenced `int64`, so that is what pgx names in the message.)

That is not a SQL error at all - it never reaches Postgres. Postgres infers the parameter's type from the column it is compared against, so `$1` is typed `uuid`, and pgx then looks for a way to encode an `*int64` as one and finds none. It surfaces as a `500` from a filter that used to work. (Write the filter's condition with a literal instead of a parameter - `WHERE primary_keeper_id = 1` - and you get the SQL-level complaint you were probably expecting: `operator does not exist: uuid = integer`. Either way the compiler stays quiet, which is the point.) The compiler is not the whole sweep; runtime is the other half, and this filter is the one case in the tutorial where they disagree.

The changes, by file:

| File | From | To |
|---|---|---|
| `internal/zookeepers/repository.go` | `ID int64`, `Get(ctx, id int64)`, scans into `int64` | `ID pgtype.UUID`, `id pgtype.UUID`, scans into `pgtype.UUID` |
| `internal/zookeepers/service.go` | `Repository` interface with `int64` ids | the same signatures with `pgtype.UUID` |
| `internal/zookeepers/handler.go` | `parseID` returning `(int64, bool)` | `parseUUID` returning `(pgtype.UUID, bool)` |
| `internal/zookeepers/dto.go` | `Response{ID int64}` | `Response{ID pgtype.UUID}` |
| `internal/animals/repository.go` | `Animal.ID int64`, `PrimaryKeeperID *int64`, `Get/Update/Delete(ctx, id int64)` | `pgtype.UUID` for both; the pointer goes away |
| `internal/animals/repository.go` + `service.go` + `handler.go` | the `List` filter: `keeperID *int64`, `primary_keeper_id = $1` | `keeperID *pgtype.UUID`; **the compiler will not find this one** (see above) - the handler's `keeper_id must be an integer` guard becomes a UUID parse returning the same 400 |
| `internal/animals/service.go` | `int64` ids in every method | `pgtype.UUID` |
| `internal/animals/handler.go` | `parseID`, `AssignKeeper` body `*int64` | `parseUUID`, body `pgtype.UUID` |
| `internal/animals/dto.go` | `keeperRef{ID int64}`, `Response{ID int64}` | `pgtype.UUID` |
| `internal/platform/auth/token.go` | `Claims{ZookeeperID int64}`, `IssueToken(..., id int64, ...)` | `pgtype.UUID` (it serializes into the token as a string) |
| `internal/platform/auth/middleware.go` | `ClaimsFrom` returns `Claims` with `int64` id | unchanged shape, new type |
| `internal/zookeepers/service_test.go`, `handler_test.go`, `auth/token_test.go` | fakes that mint `int64(len+1)` | fakes that mint a deterministic UUID |

Three of those deserve a closer look.

**The repository scan is the same statement with a different target.** Nothing about the SQL changes (the column is still `id`); only the Go type it lands in, and the `pgx.ErrNoRows` translation stays where Stage 9 put it - in the *service*, not here:

```go
// repository: the driver's error goes straight up, as Stage 9 arranged.
func (r *dbRepository) Get(ctx context.Context, id pgtype.UUID) (Zookeeper, error) {
	var zk Zookeeper
	err := r.pool.QueryRow(ctx,
		`SELECT id, username, password_hash, role, created_at, updated_at
		 FROM zookeepers WHERE id = $1`, id).
		Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role, &zk.CreatedAt, &zk.UpdatedAt)
	if err != nil {
		return Zookeeper{}, err
	}
	return zk, nil
}
```

```go
// service: unchanged from Stage 9 - the type is the only thing that moved.
func (s *Service) Get(ctx context.Context, id pgtype.UUID) (Zookeeper, error) {
	zk, err := s.repo.Get(ctx, id)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return Zookeeper{}, ErrNotFound
		}
		return Zookeeper{}, err
	}
	return zk, nil
}
```

If you are tempted to put the `ErrNoRows` check back into the repository while you are in the file anyway, resist: Stage 9's whole point was that exactly one layer translates driver errors into domain ones, and moving it back means the service's check silently stops matching.

**The JWT carries the id as a string, and that is fine.** `Claims.ZookeeperID` becomes a `pgtype.UUID`; when the token is marshalled, `pgtype.UUID.MarshalJSON` writes it as a quoted string, and `VerifyToken` reads it back through `UnmarshalJSON`. Nothing else about Stage 4 changes - the id is still in the token so handlers need no lookup, and the staleness trade from Stage 8 is unchanged. The one thing to watch: the claim is now a 36-character string in the token, which makes tokens slightly larger, which nobody will notice until tokens are very large.

```go
type Claims struct {
	ZookeeperID pgtype.UUID `json:"zookeeper_id"`
	Username    string      `json:"username"`
	Role        string      `json:"role"`
	jwt.RegisteredClaims
}

func IssueToken(secret []byte, ttl time.Duration, id pgtype.UUID, username, role string) (string, error) {
	// ... unchanged except the id type
}
```

**Tests need a UUID they can write down.** A fake that used to do `kz.ID = int64(len(f.created) + 1)` cannot invent a UUID the same way, but it can build a deterministic, obviously-fake one, which is better than random anyway because a failing assertion prints something readable:

```go
// testID builds a stable, obviously-synthetic UUID from a single byte, so a
// test can say "the second zookeeper" and the failure output shows it as
// 00000000-0000-4000-8000-000000000002 instead of a wall of hex. The version
// and variant nibbles are set (0x4 and 0x8) so the value is a well-formed v4
// UUID rather than a 16-byte blob that merely happens to fit the column - the
// same shape the bonus sections pin their fixtures to.
func testID(n byte) pgtype.UUID {
	return pgtype.UUID{Bytes: [16]byte{6: 0x40, 8: 0x80, 15: n}, Valid: true}
}
```

With that, `signToken(t, testID(1), "maya", "keeper")` replaces `signToken(t, 1, ...)` and the rest of Stage 11's tests read exactly as before.

> **What does *not* change.** The route table, the middleware wiring, the error envelope, the transaction pattern in `Feed`, and every SQL statement's structure are all identical - the id is still `$1` and still `WHERE id = $1`. That is the dividend of having one id type threaded through every layer since Stage 3: the type change is a compile error at exactly the right places and silence everywhere else. If your `go build` output is not a tidy list of every id site, something upstream is typed too loosely.

### 13.5 An id from the client is a lookup key, not a scope

There is one discipline this stage should make permanent, because it is the reason to care about enumeration in the first place: **an id that arrives from the client is never authorization.** It is a key you look a row up by, inside a scope you have already decided the caller is allowed to see.

The `{id}` in `/api/v1/animals/{id}` says *which* animal, not *whose* it is. If multi-tenant data ever lands in this schema - the same animal id space shared by more than one zoo - the safe query is `WHERE id = $1 AND zoo_id = $2` with `$2` taken from the authenticated caller, never a bare `WHERE id = $1`. A UUID makes the guessable ids hard to guess, which raises the cost of a mistake; it does not make the mistake safe. The two defences are independent, and you want both: unguessable ids so a caller cannot find a row they should not see, and a scope in the query so that even if they learn an id it does not matter.

This is worth writing down here because the bonus material, when it introduces multiple zoos, leans on exactly this rule - and the reader who internalises it now will find that section obvious rather than difficult.

### 13.6 Verify

Restart the server and walk a create-then-read, which is the flow that changed shape. Watch what an id looks like now.

```bash
curl -s -X POST http://localhost:8080/api/v1/animals \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"name":"Zuri","species":"Reticulated giraffe","enclosure":"Savanna Yard"}'
# {"id":"5f1c2a9e-...-...","name":"Zuri","species":"Reticulated giraffe",...}
```

Take that id, paste it into the next call - there is no "next integer" any more:

```bash
curl -s http://localhost:8080/api/v1/animals/5f1c2a9e-...-... -H "Authorization: Bearer $TOKEN"
# {"id":"5f1c2a9e-...-...","name":"Zuri",...}
```

And the two failure modes, which should be a `400` and a `404` respectively:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/api/v1/animals/not-a-uuid -H "Authorization: Bearer $TOKEN"
# 400  - the path parameter does not parse as a UUID
curl -s -o /dev/null -w '%{http_code}\n' \
  http://localhost:8080/api/v1/animals/00000000-0000-0000-0000-000000000000 -H "Authorization: Bearer $TOKEN"
# 404  - well-formed UUID, no such row
```

The database side, confirming the key really is a UUID and the child columns kept their `NOT NULL`:

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo
```

```sql
SELECT data_type FROM information_schema.columns
 WHERE table_name='animals' AND column_name='id';          -- uuid

SELECT is_nullable FROM information_schema.columns
 WHERE table_name='feed_log' AND column_name='animal_id';  -- NO

SELECT a.name, z.username FROM animals a
 JOIN zookeepers z ON z.id = a.primary_keeper_id ORDER BY a.name;   -- joins still resolve
```

If those three come back `uuid`, `NO`, and the same keeper pairing as before the migration, the swap is done. Two things the compiler cannot check for you and you should:

- **`ORDER BY id` anywhere in your code now sorts arbitrarily.** The tutorial orders zookeepers and animals by `id` in a few list queries (Stage 3's list, Stage 6's filter); with v4 UUIDs that ordering is no longer creation order. Switch those to `ORDER BY created_at` (add a tiebreaker on `id` if you want stability), or move to v7 UUIDs, which sort by creation time.
- **Anything that stored an id in a place the compiler does not see** - a config file, a fixture, a row in a table you did not migrate - still holds an integer. Grep your repo for the old numbers; the ones that matter are the ones outside the Go type system.

---

[Stage 12](12-makefile-recap.md)  |  [Overview](../tutorial.md)  |  [Bonus 14](14-bonus-multi-zoo-enclosures.md)
