## Bonus 13: Multiple zoos, and enclosures that mean something

This is **bonus material**. It is not part of the linear path through Stages 1-12, and nothing later depends on it. It assumes you have a working Stage 12 project and a database you are willing to change.

Three things make it worth building anyway. It is the first time a schema change has to happen to a table that already holds data (the expand and contract pattern). It is the first business rule that a `CHECK` constraint genuinely cannot express, so you have to reach for a transaction and a row lock. And it is the first time the same table serves more than one customer, which is where "we forgot a `WHERE` clause" becomes a data breach rather than a bug.

Every migration and every query below was run against Postgres 17 before being written down, including the failure cases.

### 13.1 What "multiple zoos" actually forces

There are three ways to host several zoos in one service, and they are not equivalent:

| Shape | Isolation | Cost |
|---|---|---|
| One database per zoo | Strongest: a connection can only reach its own zoo | A migration must be run N times; cross-zoo reporting needs a data warehouse |
| One schema per zoo in one database (`zoo_north.animals`) | Strong: `search_path` scopes everything | Migrations still run N times; connection pooling gets fiddly |
| One set of tables, every row carries `zoo_id` | Weakest: correctness depends on every query filtering | One migration, one pool, trivial cross-zoo reporting |

Real ERP systems pick per table, not per system: master data (species, environments, pay rates) is shared, operational data (animals, enclosures, shifts) is tenant-scoped. That is what we will do, and 13.5 is about repairing the weakness of the third row rather than pretending it does not exist.

Two names before the schema: a **tenant** is the customer whose data must not be visible to another customer; **row-level tenancy** is the third shape above, where the tenant is a column rather than a separate database.

### 13.2 The schema: zoos, environments, species, enclosures

`migrations/00004_multi_zoo_and_enclosures.sql`:

```sql
-- +goose Up
CREATE TABLE zoos (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name       text NOT NULL UNIQUE,
    created_at timestamptz NOT NULL DEFAULT now()
);

-- Environments are a lookup table rather than a CHECK constraint on
-- enclosures because they are data, not code: a new biome should be an
-- INSERT, not a migration.
CREATE TABLE environments (
    name        text PRIMARY KEY,
    description text NOT NULL
);

-- The species -> environment mapping is what makes "put the wrong animal in
-- the wrong enclosure" a rule the database can help with.
CREATE TABLE species (
    name        text PRIMARY KEY,
    environment text NOT NULL REFERENCES environments(name)
);

CREATE TABLE enclosures (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    zoo_id      bigint NOT NULL REFERENCES zoos(id) ON DELETE CASCADE,
    name        text NOT NULL,
    environment text NOT NULL REFERENCES environments(name),
    capacity    int NOT NULL CHECK (capacity > 0),
    UNIQUE (zoo_id, name)
);

INSERT INTO environments (name, description) VALUES
    ('savanna', 'open grassland, warm'),
    ('jungle',  'humid, dense cover'),
    ('forest',  'temperate woodland'),
    ('aquatic', 'water bodies');

INSERT INTO species (name, environment) VALUES
    ('African bush elephant', 'savanna'),
    ('Reticulated giraffe',   'savanna'),
    ('Sumatran tiger',        'jungle'),
    ('Red panda',             'forest');

INSERT INTO zoos (name) VALUES ('North Zoo'), ('South Zoo');

-- The enclosures those two zoos actually have. Note what is missing: North
-- Zoo has no forest yard, which 13.3 makes visible.
INSERT INTO enclosures (zoo_id, name, environment, capacity) VALUES
    (1, 'Savanna Yard',  'savanna', 3),
    (1, 'Jungle House',  'jungle',  1),
    (2, 'South Savanna', 'savanna', 2);

-- +goose Down
DROP TABLE enclosures, species, environments, zoos;
```

Five decisions are visible in that SQL, and each is the kind you should be able to defend in a review:

- **`environments` is a table, not a `CHECK (environment IN (...))`.** A `CHECK` constraint is code: adding a biome means a migration, on every environment, at a coordinated moment. A lookup table makes it an `INSERT`. The rule of thumb is that a `CHECK` is right for values that are *structural* (a role is `admin` or `keeper`; there is no fifth role) and wrong for values that are *data* (the list of biomes will grow).
- **`capacity` carries a `CHECK`.** `CHECK (capacity > 0)` rejects a capacity of zero or below, which is meaningless. It is worth internalising what a constraint is *not*: it is not a spell-checker (a typo in a column name is a compile error in Go and a `\d enclosures` away in SQL) and, as 13.4 shows, it can never see more than one row at a time.
- **`enclosures.UNIQUE (zoo_id, name)`**, not `UNIQUE (name)` globally: two zoos may legitimately both have a "Savanna Yard". The uniqueness you want is per tenant.
- **`ON DELETE CASCADE` from `zoos` to `enclosures`** matches the `feed_log` cascade debate from Stage 6: deleting a whole zoo is an administrative act, and silently orphaning its rows would be worse than removing them.
- **The species mapping is keyed by name (`text`)**, mirroring the existing `animals.species` column so the migration in 13.3 can join on it. In a system you were designing from scratch you would give species an integer id and have `animals` reference it; the wrap-up in Stage 12 makes the same point about UUIDs. Text keys are used here to keep the migration small, and the trade is spelled out rather than hidden.

### 13.3 Moving existing animals: the expand and contract pattern

`animals` already has rows, an `enclosure` text column, and a deployed version of the API reading it. Renaming a column out from under a running service is how you get an outage, so the migration happens in two deploys.

**Expand** - additive and nullable, so the old code keeps working while the new code ships:

```sql
-- +goose Up
ALTER TABLE animals ADD COLUMN zoo_id       bigint REFERENCES zoos(id);
ALTER TABLE animals ADD COLUMN enclosure_id bigint REFERENCES enclosures(id);
```

**Backfill** - fill the new columns from the old ones, as far as the data allows:

```sql
UPDATE animals SET zoo_id = 1 WHERE zoo_id IS NULL;

UPDATE animals SET enclosure_id = (
    SELECT e.id FROM enclosures e
    JOIN species s ON s.environment = e.environment
    WHERE e.zoo_id = animals.zoo_id AND s.name = animals.species
    ORDER BY e.id
    LIMIT 1
) WHERE enclosure_id IS NULL;
```

Run that against the tutorial's three animals and the backfill places two of them:

```
 id |  name   |        species        | zoo_id | enclosure_id
----+---------+-----------------------+--------+--------------
  1 | Tembo   | African bush elephant |      1 |            1
  2 | Suki    | Sumatran tiger        |      1 |            2
  3 | Biscuit | Red panda             |      1 |
```

Biscuit the red panda has no enclosure, because North Zoo has a savanna yard and a jungle yard and no forest. **That is the backfill doing its job.** The alternative - inventing an enclosure, or defaulting everything into the first one - would have silently placed a red panda in a savanna and nobody would have found out until an inspector did. A `NULL` here is a real, actionable fact: North Zoo has an animal it cannot house.

Which means the API has to be honest about it too. `animals.enclosure_id` stays nullable, and the response grows the enclosure without pretending it is always there:

```go
type enclosureRef struct {
	ID          int64  `json:"id"`
	Name        string `json:"name"`
	Environment string `json:"environment"`
}

type Response struct {
	...
	Enclosure *enclosureRef `json:"enclosure"`
}
```

**Contract** - later, in a separate deploy, once you are certain nothing reads the old column:

```sql
ALTER TABLE animals ALTER COLUMN zoo_id SET NOT NULL;
ALTER TABLE animals DROP COLUMN enclosure;
```

The gap between expand and contract is measured in deploys, not minutes: you can only drop `enclosure` after the *oldest* running version of your service has stopped reading it. That is why the pattern exists, and why the two `ALTER`s above are deliberately not in the same migration file as the first two.

### 13.4 Capacity: the rule a CHECK constraint cannot express

"An enclosure may not hold more animals than its capacity" is a rule about *many rows*, and a `CHECK` constraint only ever sees one row. Postgres offers three ways out, and it is worth knowing which one you are choosing:

1. **A trigger** (`BEFORE INSERT OR UPDATE ... FOR EACH ROW`) that counts and raises. It works, it is invisible to anyone reading the Go code, and it makes every write slower. Good for rules that must hold no matter who writes.
2. **An `EXCLUDE` constraint** (Bonus 14.2 uses one for shifts) that expresses the rule as a *conflict between rows*. Brilliant when the rule is "no two of these may overlap"; there is no clean way to spell "no more than n of these" with it.
3. **A transaction with a row lock**, checked in the repository. Visible in the code, easy to test, and the one we take here.

The reason a plain `SELECT count(*)` is not enough is a race:

```
request A: count = capacity - 1   ->  1 free slot
request B: count = capacity - 1   ->  1 free slot   (A has not written yet)
request A: INSERT ... COMMIT
request B: INSERT ... COMMIT      ->  capacity + 1 animals
```

Both requests were correct at the moment they looked. The fix is to make the second one wait:

```go
// PlaceAnimal moves an animal into an enclosure in its own zoo, refusing to
// exceed the enclosure's capacity. The check runs in one transaction, and it
// takes a row lock on the animal and then on the enclosure, in that order, so
// two requests racing for the last place queue up behind each other instead of
// both seeing a free slot.
func (r *dbRepository) PlaceAnimal(ctx context.Context, zooID, animalID, enclosureID int64) error {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx)

	// Lock order matters and is not local to this function: Bonus 15.4's
	// CompleteTransfer takes these same two locks, and takes them in the same
	// order. Two transactions grabbing the same two rows in opposite orders
	// deadlock; Postgres detects it and kills one, which is safe but noisy,
	// and a stated order is cheaper than the debugging.
	var species string
	if err := tx.QueryRow(ctx,
		`SELECT species FROM animals
		 WHERE id = $1 AND zoo_id = $2
		 FOR UPDATE`,
		animalID, zooID).
		Scan(&species); err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return ErrNotFound
		}
		return err
	}

	var environment string
	var capacity int
	if err := tx.QueryRow(ctx,
		`SELECT environment, capacity FROM enclosures
		 WHERE id = $1 AND zoo_id = $2
		 FOR UPDATE`,
		enclosureID, zooID).
		Scan(&environment, &capacity); err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return ErrEnclosureNotFound
		}
		return err
	}

	var compatible bool
	if err := tx.QueryRow(ctx,
		`SELECT EXISTS (SELECT 1 FROM species
		                 WHERE name = $1 AND environment = $2)`,
		species, environment).
		Scan(&compatible); err != nil {
		return err
	}
	if !compatible {
		return ErrWrongEnvironment
	}

	// Count the other occupants: the animal being moved may already be in
	// this enclosure, and counting it would make re-placing it look full.
	var occupants int
	if err := tx.QueryRow(ctx,
		`SELECT count(*) FROM animals WHERE enclosure_id = $1 AND id <> $2`,
		enclosureID, animalID).
		Scan(&occupants); err != nil {
		return err
	}
	if occupants >= capacity {
		return ErrEnclosureFull
	}

	if _, err := tx.Exec(ctx,
		`UPDATE animals SET enclosure_id = $1, updated_at = now() WHERE id = $2`,
		enclosureID, animalID); err != nil {
		return err
	}

	return tx.Commit(ctx)
}
```

Three things in that function are load-bearing:

- **`FOR UPDATE`** takes a row-level lock, held until the transaction ends. A second `PlaceAnimal` for the same enclosure blocks on that line rather than failing: measured on Postgres 17, a request arriving while another transaction held the lock simply waited it out and then proceeded (the exact wait depends on your machine, but the shape does not - it queues, it does not error). That serialisation is what a capacity check needs, and it is why the *enclosure* is locked and not merely the animal: two animals may move into the same enclosure concurrently, but they must not both be the one that fills it. Note that both rows are locked, animal first, and in a fixed order - 15.4 takes the same pair, and a stated order is the only thing that keeps them from deadlocking.
- **`WHERE id = $1 AND zoo_id = $2`** rather than `WHERE id = $1`. The zoo check is not decoration; it is what stops a request scoped to North Zoo from moving an animal into South Zoo's enclosure by guessing an id. The rule is worth writing down once and applying everywhere: **an id from the client is never a scope, it is a lookup key inside the scope you already have.**
- **`defer tx.Rollback(ctx)`** is Stage 7's transaction pattern, unchanged. Every early return above - not found, wrong environment, full - rolls back, and the lock is released.

The three errors returned here are new sentinels, declared in Stage 9's vocabulary rather than beside it:

```go
var (
	ErrEnclosureNotFound = httpx.NotFound("enclosure not found")
	ErrWrongEnvironment  = httpx.Conflict("wrong_environment", "that species cannot live in this enclosure")
	ErrEnclosureFull     = httpx.Conflict("enclosure_full", "enclosure is at capacity")
)
```

Declared that way, the handler calling `PlaceAnimal` needs no new code at all: `httpx.Respond(w, err)` reads the status off whichever one comes back, which is the whole point of Stage 9's pipeline. Declared as plain `errors.New` values instead, all three would surface as 500s - an error is only self-describing if you make it so, and that is the difference Stage 9 was for.

> **A note on where this rule belongs.** It lives in the repository because it needs the transaction, but it is a *business* rule, and a reviewer is entitled to ask why it is not in the service. The honest answer: the service could own the rule and pass the repository a "check this for me" callback, but that buys indirection, not clarity. What matters is that exactly one place enforces it - two places enforcing it with different SQL is how tenants end up with three elephants in a two-elephant yard.

### 13.5 Not leaking another zoo's rows

Here is the failure mode, stated plainly: **every query that touches animals now needs `WHERE zoo_id = $n`, and the day someone writes one without it, North Zoo reads South Zoo's animals.** It will pass review, because the query is otherwise correct. It will pass tests, because a test that only has one zoo's data cannot detect it.

There are two defences, and the right answer is usually both.

**Defence one: make the unscoped query impossible to write.** Change the repository's method set so that no method can list animals without a zoo:

```go
// Every read takes the zoo it is allowed to see. Note what does not exist:
// there is no ListAnimals(ctx) with no zoo parameter, so there is no call
// site that can accidentally read across tenants.
func (r *dbRepository) List(ctx context.Context, zooID int64, keeperID *int64) ([]Animal, error)
func (r *dbRepository) Get(ctx context.Context, zooID, id int64) (Animal, string, error)
```

The compiler then enforces what a code review only hopes for. This is the single highest-value change in this whole bonus section, and it costs nothing but a signature.

**Defence two: row level security, so even a mistake cannot leak.** Postgres can filter rows by policy, per connecting role, with no help from your SQL:

```sql
-- +goose Up
CREATE ROLE zoo_app LOGIN PASSWORD 'change-me';
GRANT USAGE ON SCHEMA public TO zoo_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO zoo_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO zoo_app;

ALTER TABLE animals ENABLE ROW LEVEL SECURITY;

CREATE POLICY animals_by_zoo ON animals
    USING (zoo_id = nullif(current_setting('app.zoo_id', true), '')::bigint);

-- +goose Down
DROP POLICY animals_by_zoo ON animals;
ALTER TABLE animals DISABLE ROW LEVEL SECURITY;
```

and the application sets the tenant for the duration of each transaction:

```go
tx, err := pool.Begin(ctx)
...
if _, err := tx.Exec(ctx, `SELECT set_config('app.zoo_id', $1, true)`, zooID); err != nil {
	return err
}
```

Verified behaviour, which is worth reading carefully because the second line is not what you would guess:

- With `app.zoo_id = '2'` set, `SELECT * FROM animals` returns only zoo 2's rows. With `'1'`, only zoo 1's. The policy is doing the filtering, and the query has no `WHERE zoo_id` at all.
- With the tenant **never set**, the same query returns **zero rows**: `nullif` yields NULL, `NULL = anything` is NULL rather than true, and the policy fails closed. A request that forgot to set a tenant sees nothing rather than everything.

And the trap, which cost a real debugging session before it was written down: `current_setting('app.zoo_id', true)` returns **NULL** when the setting was never set, but returns **the empty string** after a `RESET`, because the setting now exists with an empty value. `''::bigint` is a hard error - `invalid input syntax for type bigint: ""` - which surfaces as a 500 rather than an empty result. The `nullif(..., '')` above is not defensive padding; it is the fix, and it is exactly the sort of thing that only shows up when a connection is reused by a later request. Which is always, with a pool.

> **What RLS costs you.** The policy is attached to the *role*, so your migration role (which owns the tables) bypasses it by default - `ALTER TABLE ... FORCE ROW LEVEL SECURITY` changes that if you want parity. Every new table needs its own policy or it is unprotected, which is a list that drifts. And the tenant has to be set on the same connection that runs the query, which with a pooled connection means inside a transaction (`set_config(..., true)` is transaction-local) rather than once at startup. None of that is a reason not to do it; all of it is a reason to do defence one first and RLS second.

### 13.6 Reporting: occupancy without a counting query per request

"Which enclosures are full, and which have space?" is a question an operations screen asks every few seconds, and it is a `GROUP BY` over the whole `animals` table. Materialise it:

```sql
CREATE MATERIALIZED VIEW zoo_occupancy AS
SELECT z.id AS zoo_id, z.name AS zoo,
       e.id AS enclosure_id, e.name AS enclosure, e.capacity,
       count(a.id) AS occupants
FROM zoos z
JOIN enclosures e ON e.zoo_id = z.id
LEFT JOIN animals a ON a.enclosure_id = e.id
GROUP BY z.id, z.name, e.id, e.name, e.capacity;

CREATE UNIQUE INDEX zoo_occupancy_enclosure_idx ON zoo_occupancy (enclosure_id);
```

The unique index is not optional decoration. Without it, `REFRESH MATERIALIZED VIEW CONCURRENTLY zoo_occupancy` fails with `cannot refresh materialized view ... concurrently` and you are left with the plain `REFRESH`, which takes an `ACCESS EXCLUSIVE` lock and blocks every reader for the duration. With the index, the concurrent refresh is allowed and readers are never blocked. Verified on Postgres 17 in both directions, because the error message is good but the reason is not obvious.

A materialised view is a cache with a manual invalidation button, so decide *who* presses it: a `REFRESH ... CONCURRENTLY` at the end of every feed and transfer transaction is fine at this scale and wrong at a bigger one, where the answer becomes an incremental view (`REFRESH ... CONCURRENTLY` on a schedule) or a summary table updated by the same transaction that changed it.

### 13.7 Verify

Everything below is `psql` rather than `curl`, because every claim in this section is about the database's behaviour, and the database is where you should check it.

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo
```

```sql
-- the schema exists and the animals were backfilled
\d enclosures
SELECT id, name, species, zoo_id, enclosure_id FROM animals ORDER BY id;
--  1 | Tembo   | African bush elephant | 1 |            1
--  2 | Suki    | Sumatran tiger        | 1 |            2
--  3 | Biscuit | Red panda             | 1 |       (NULL)
-- Biscuit the red panda has no enclosure: North Zoo has no forest yard.

-- capacity is a rule about many rows, so plain SQL happily breaks it: the
-- jungle yard (id 2) holds one animal and its capacity is 1
UPDATE animals SET enclosure_id = 2 WHERE id = 1;
SELECT count(*) FROM animals WHERE enclosure_id = 2;   -- 2, over capacity
-- the API refuses that move; the repository's transaction is what refuses it
UPDATE animals SET enclosure_id = 1 WHERE id = 1;      -- back where the backfill put it

-- tenancy. This part only means anything as a role that does NOT own the
-- table, because a table's owner bypasses its own policies (13.5 says so).
SET ROLE zoo_app;                -- the role 13.5's migration created
SET app.zoo_id = '1';
SELECT count(*) FROM animals;    -- 3: North Zoo's animals
RESET app.zoo_id;
SELECT count(*) FROM animals;    -- 0: no tenant set, so nothing is visible
RESET ROLE;

-- and the report
REFRESH MATERIALIZED VIEW CONCURRENTLY zoo_occupancy;
SELECT * FROM zoo_occupancy ORDER BY zoo_id, enclosure_id;
--  1 | North Zoo | 1 | Savanna Yard  | 3 | 1
--  1 | North Zoo | 2 | Jungle House  | 1 | 1
--  2 | South Zoo | 3 | South Savanna | 2 | 0
```

Two things in that scroll are worth more than the SQL they sit next to.

**`SET ROLE zoo_app` is not decoration.** Run those same two `SELECT`s as the owner and every one of them "passes" while proving nothing: the owner sees all three rows with a tenant set and all three with none, because row level security does not apply to a table's owner by default. That is the difference between a green test and a meaningful one, and it is why the verification names the role explicitly.

**The assertion worth keeping** is the second one: as `zoo_app`, with `app.zoo_id` unset, `SELECT count(*) FROM animals` must return **0**. If it ever returns the whole table, one of two things has happened - the policy was dropped, or someone connected as the owner - and the count alone will not tell you which. Add `ALTER TABLE animals FORCE ROW LEVEL SECURITY` if you want the owner to be subject to its own policy too, and then the same assertion holds for every role.

---

[Stage 12](12-makefile-recap.md)  |  [Overview](../tutorial.md)  |  [Bonus 14](14-bonus-scheduling-payroll.md)
