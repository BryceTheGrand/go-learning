## Bonus 14: Multiple zoos, and enclosures that mean something

This is **bonus material**. It is not part of the linear path through Stages 1-13, though it is the one bonus the other two are built on: Bonus 15's shifts are scoped to a zoo, and Bonus 16's transfers move an animal between two of them. It assumes you have a working Stage 13 project and a database you are willing to change.

Three things make it worth building anyway. It is the second time a schema change has to land on a table that already holds data - Stage 13 converted the primary keys under live foreign keys, and this stage adds the tenant columns to `animals` the same way (the expand and contract pattern). It is the first business rule that a `CHECK` constraint genuinely cannot express, so you have to reach for a transaction and a row lock. And it is the first time the same table serves more than one customer, which is where "we forgot a `WHERE` clause" becomes a data breach rather than a bug.

Every migration and every query below was run against Postgres 17 before being written down, including the failure cases.

### 14.1 What "multiple zoos" actually forces

There are three ways to host several zoos in one service, and they are not equivalent:

| Shape | Isolation | Cost |
|---|---|---|
| One database per zoo | Strongest: a connection can only reach its own zoo | A migration must be run N times; cross-zoo reporting needs a data warehouse |
| One schema per zoo in one database (`zoo_north.animals`) | Strong: `search_path` scopes everything | Migrations still run N times; connection pooling gets fiddly |
| One set of tables, every row carries `zoo_id` | Weakest: correctness depends on every query filtering | One migration, one pool, trivial cross-zoo reporting |

Real ERP systems pick per table, not per system: master data (species, environments, pay rates) is shared, operational data (animals, enclosures, shifts) is tenant-scoped. That is what we will do, and 14.5 is about repairing the weakness of the third row rather than pretending it does not exist.

Two names before the schema: a **tenant** is the customer whose data must not be visible to another customer; **row-level tenancy** is the third shape above, where the tenant is a column rather than a separate database.

### 14.2 The schema: zoos, environments, species, enclosures

`migrations/00005_multi_zoo_and_enclosures.sql`:

```sql
-- +goose Up
CREATE TABLE zoos (
    id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
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
    id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    zoo_id      uuid NOT NULL REFERENCES zoos(id) ON DELETE CASCADE,
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

INSERT INTO zoos (id, name) VALUES
    ('00000000-0000-4000-8000-000000000001', 'North Zoo'),
    ('00000000-0000-4000-8000-000000000002', 'South Zoo');

-- The enclosures those two zoos actually have. Note what is missing: North
-- Zoo has no forest yard, which 14.3 makes visible.
INSERT INTO enclosures (id, zoo_id, name, environment, capacity) VALUES
    ('00000000-0000-4000-8000-000000000001', '00000000-0000-4000-8000-000000000001', 'Savanna Yard',  'savanna', 3),
    ('00000000-0000-4000-8000-000000000002', '00000000-0000-4000-8000-000000000001', 'Jungle House',  'jungle',  1),
    ('00000000-0000-4000-8000-000000000003', '00000000-0000-4000-8000-000000000002', 'South Savanna', 'savanna', 2);

-- +goose Down
DROP TABLE enclosures, species, environments, zoos;
```

Five decisions are visible in that SQL, and each is the kind you should be able to defend in a review:

- **`environments` is a table, not a `CHECK (environment IN (...))`.** A `CHECK` constraint is code: adding a biome means a migration, on every environment, at a coordinated moment. A lookup table makes it an `INSERT`. The rule of thumb is that a `CHECK` is right for values that are *structural* (a role is `admin` or `keeper`; there is no fifth role) and wrong for values that are *data* (the list of biomes will grow).
- **`capacity` carries a `CHECK`.** `CHECK (capacity > 0)` rejects a capacity of zero or below, which is meaningless. It is worth internalising what a constraint is *not*: it is not a spell-checker (a typo in a column name is a compile error in Go and a `\d enclosures` away in SQL) and, as 14.4 shows, it can never see more than one row at a time.
- **`enclosures.UNIQUE (zoo_id, name)`**, not `UNIQUE (name)` globally: two zoos may legitimately both have a "Savanna Yard". The uniqueness you want is per tenant.
- **`ON DELETE CASCADE` from `zoos` to `enclosures`** matches the `feed_log` cascade debate from Stage 6: deleting a whole zoo is an administrative act, and silently orphaning its rows would be worse than removing them.
- **The species mapping is keyed by name (`text`)**, mirroring the existing `animals.species` column, so the two agree on what a species *is* by construction rather than by a translation step. In a system you were designing from scratch you would give species a `uuid` id and have `animals` reference it - Stage 13 makes the case for UUIDs on identifiers that cross a trust or a system boundary. But a species name crosses neither: it is a stable natural key that is never guessed and never merged, which is exactly the exception 13.1 carves out, so the text key here is the right call and not merely the small one. Text keys are used here to keep the migration small, and the trade is spelled out rather than hidden.

### 14.3 Moving existing animals: the expand and contract pattern

`animals` already has rows, an `enclosure` text column, and a deployed version of the API reading it. Renaming a column out from under a running service is how you get an outage, so the migration happens in two deploys.

Your database has three animals in it at this point, and they come from three different stages: Tembo, kept by sam (Stages 6 and 8), Rex (Stage 8), and Zuri, the giraffe 13.6's verify created to watch a UUID come back from a POST. Stage 1's in-memory list had a different three (Tembo, Suki, Biscuit), and this walkthrough wants one of *those* names back: a name with no matching enclosure is what makes the `NULL` case below visible. So the fixture replaces the roster outright rather than adding to it.

Stage 13 handed the real rows `gen_random_uuid()` ids - random, and different in every database. A random UUID cannot be retyped from one prompt to the next, so the fixture pins fixed, obviously-synthetic ids (the `00000000-0000-4000-8000-0000000000NN` shape Stage 13's test helper builds) and every result below is reproducible against them. This is bonus material on a database you are willing to change, so rebuild them:

```sql
DELETE FROM animals;   -- the feed_log rows point at them, so they cascade away too
INSERT INTO animals (id, name, species, enclosure) VALUES
    ('00000000-0000-4000-8000-000000000001', 'Tembo',   'African bush elephant', 'Savanna Yard'),
    ('00000000-0000-4000-8000-000000000002', 'Suki',    'Sumatran tiger',        'Jungle House'),
    ('00000000-0000-4000-8000-000000000003', 'Biscuit', 'Red panda',             'Forest Yard');
```

> **Order matters here: run that `psql` block *before* you write and run `00006`.** The fixture writes the old `enclosure` text column, and the migration's backfill is what fills `enclosure_id` from it - so the pinned rows have to be in place for the backfill to see them. Migrate first and you get the opposite: `00006` backfills whatever rows existed then, and the fixture inserts three fresh rows *afterwards* with `zoo_id` and `enclosure_id` left `NULL`, so every result below reads `(NULL)` and none of it matches. If that has already happened, the fix is idempotent and safe to run in either order: run the `DELETE`/`INSERT` fixture again and then re-run the two backfill `UPDATE`s by hand (`goose` records version 6 as applied and will not repeat them for you). Because both `UPDATE`s are guarded by `WHERE ... IS NULL`, re-running them touches only the rows the fixture just added and changes nothing that was already filled.

**Expand** - additive and nullable, so the old code keeps working while the new code ships. This is a new migration file, `migrations/00006_animals_tenancy.sql`:

```sql
-- +goose Up
ALTER TABLE animals ADD COLUMN zoo_id       uuid REFERENCES zoos(id);
ALTER TABLE animals ADD COLUMN enclosure_id uuid REFERENCES enclosures(id);

-- +goose Down
ALTER TABLE animals DROP COLUMN enclosure_id;
ALTER TABLE animals DROP COLUMN zoo_id;
```

**Backfill** - fill the new columns from the old ones, as far as the data allows. The `UPDATE`s go in the same file under the same `Up` (they are one deploy with the columns that need them):

```sql
UPDATE animals SET zoo_id = '00000000-0000-4000-8000-000000000001' WHERE zoo_id IS NULL;

UPDATE animals SET enclosure_id = (
    SELECT e.id FROM enclosures e
    WHERE e.zoo_id = animals.zoo_id AND e.name = animals.enclosure
) WHERE enclosure_id IS NULL;
```

The second lookup goes through `enclosures.name`, and that is the whole design of the backfill: **the legacy text column is a name, so the new column is filled by matching that name.** The `UNIQUE (zoo_id, name)` constraint from 14.2 is what makes it a scalar subquery rather than a `LIMIT 1` guess, and it is the same natural key the API has always been taking in `"enclosure": "Savanna Yard"` - so the value that goes into the new column is a direct function of the value that was in the old one, and a migration is a data move rather than a place to re-decide where animals belong.

Run that against the three rows above and the backfill places two of them:

```
                  id                  |  name   |        species        |                zoo_id                |             enclosure_id
--------------------------------------+---------+-----------------------+--------------------------------------+--------------------------------------
 00000000-0000-4000-8000-000000000001 | Tembo   | African bush elephant | 00000000-0000-4000-8000-000000000001 | 00000000-0000-4000-8000-000000000001
 00000000-0000-4000-8000-000000000002 | Suki    | Sumatran tiger        | 00000000-0000-4000-8000-000000000001 | 00000000-0000-4000-8000-000000000002
 00000000-0000-4000-8000-000000000003 | Biscuit | Red panda             | 00000000-0000-4000-8000-000000000001 |
(3 rows)
```

Biscuit keeps a `NULL`, because no enclosure at North Zoo is called `Forest Yard` - North Zoo has a savanna yard and a jungle yard and no forest anything. **That is the backfill doing its job.** The alternatives are all worse, and each for a different reason. Inventing an enclosure, or defaulting everything into the first one, would have silently placed a red panda in a savanna and nobody would have found out until an inspector did. Less obviously, *re-deriving* the answer - picking the enclosure whose environment matches the animal's species, rather than the one the record actually names - looks careful and is not: it would quietly move an animal that had been mis-housed, and it has no answer at all when a zoo has two enclosures of the same environment, which is exactly the case a `LIMIT 1` hides. A `NULL` here is the honest outcome and a real, actionable fact: this record names a yard that does not exist, and North Zoo has an animal it cannot house. Whether it *should* be living somewhere else is a question for a human, and 14.4 is where the rule that keeps it from happening again gets written.

Which means the API has to be honest about it too. `animals.enclosure_id` stays nullable, and the response grows the enclosure without pretending it is always there:

```go
type enclosureRef struct {
	ID          pgtype.UUID `json:"id"`
	Name        string      `json:"name"`
	Environment string      `json:"environment"`
}
```

`Response` then changes in exactly one line - the `Enclosure string` from Stage 6 is **replaced**, not added to (note `ID` is a `pgtype.UUID` now; this struct is post-Stage-13):

```go
type Response struct {
	ID            pgtype.UUID   `json:"id"`
	Name          string        `json:"name"`
	Species       string        `json:"species"`
	Enclosure     *enclosureRef `json:"enclosure"`   // was: string
	LastFedAt     *time.Time    `json:"last_fed_at"`
	CreatedAt     time.Time     `json:"created_at"`
	UpdatedAt     time.Time     `json:"updated_at"`
	PrimaryKeeper *keeperRef    `json:"primary_keeper"`
}
```

> **Leave the old `Enclosure string` field in place and you get a silent wrong answer.** A second field named `Enclosure` is a compile error, so what you will actually do is rename one of them - and then the two fields share the tag `json:"enclosure"`. `encoding/json` resolves same-depth tag conflicts by dropping *every* field involved, so both disappear: no compile error, no runtime error, no failing test, just an `enclosure` key that quietly stops appearing in every animal response. (Verified: a struct with `Enclosure string \`json:"enclosure"\`` and `Ref *enclosureRef \`json:"enclosure"\`` marshals to `{"id":1}`, with no error.) The compiler cannot help you here; the only way to catch it is to read the JSON body, which is why the verify steps compare the body rather than the status code.

That struct change is only half of it - the field is `*enclosureRef`, so something has to fill it, and the read path is where the old free-text column actually disappears. It is the same move Stage 6 made for `primary_keeper`, applied to a second nullable relationship:

- `Animal` loses `Enclosure string` and gains `EnclosureID pgtype.UUID` (plus `*enclosureRef`'s own fields, or a nested struct you scan into - either works, and the tutorial's `keeperRef` shape is the one to copy).
- `selectColumns` stops naming the text column and the query grows `LEFT JOIN enclosures e ON e.id = a.enclosure_id`, selecting `e.id, e.name, e.environment` alongside the animal's own columns.
- `toResponse` maps the three scanned values into `*enclosureRef` - and the pointer stays `nil` when the join found nothing, which is what serialises as `null` for Biscuit. A query that forgot the join, or an `INNER JOIN`, would either lose the field or drop Biscuit's row entirely; those are the two failure modes to watch when you rewire it.

None of that is new code you have not written before; it is Stage 6's nullable-join pattern with a different table on the right-hand side. The one place it differs is that the column being replaced (`enclosure`) is a plain string today, so the sweep is mechanical: rename the field, join the table, map the struct.

**The write path is the half that bites, and it is the reason this is a two-deploy pattern rather than a rename.** `animals.enclosure` is still `NOT NULL` - the contract below has not run, and running it early is exactly the mistake the pattern exists to prevent - so `Create` and `Update` must keep writing it while they start writing the new columns too:

```go
-- Create, during the expand window: the old column and the new ones are both
-- written. The old one keeps the running (old) version of the service working;
-- the new ones are what everything reads after the contract deploy.
INSERT INTO animals (name, species, enclosure, zoo_id, enclosure_id)
VALUES ($1, $2, $3, $4,
        (SELECT e.id FROM enclosures e
          WHERE e.zoo_id = $4 AND e.name = $3))
```

The subquery is the load-bearing part, and the rule it encodes is the one to carry out of this section: **during the expand window the new column is derived from the old one, not supplied alongside it.** `$3` - the free text the API has always accepted (`{"name":..., "species":..., "enclosure":"Savanna Yard"}`) - is the source of truth while the old service is still running and reading it, and `enclosure_id` is looked up from it by exactly the lookup the backfill above uses: the enclosure of that name, inside this zoo. That is deliberate, and it is the one shape that keeps the migration honest. Resolve the id from anywhere else - a second field on the request, a `SELECT` the caller does not control, or a fallback to a previous value with `COALESCE` when the name does not match - and the two columns can disagree: write `"Jungle House"` into the text column and a `Savanna Yard` id into the new one and nothing complains, because nothing compares them. The disagreement only becomes visible after the contract deploy, when the column everyone reads is the one that was wrong.

Two consequences fall out of the rule:

- **`Update` recomputes it whenever `enclosure` changes.** The id is a function of the text, so a change to the input has to move the derived value with it. This is the same statement as the one above, run through `UPDATE ... FROM`, and it is not optional for the same reason: a stale id is the same divergence, just one edit later. (`species` does not enter the lookup, which is correct - editing what an animal *is* is not a move - but it does mean a species change alone can leave it sitting in a yard that no longer suits it; the note below is about who catches that.)
- **The derivation disappears at the contract deploy, and its direction inverts.** Once `enclosure` is dropped, the API takes an enclosure id - the caller names the enclosure directly - and there is nothing left to look up. That is what makes the second deploy worth doing rather than a formality: it is the moment the free-text column stops being the truth.

A miss on the lookup is not an error: the subquery returns no row and `enclosure_id` stays `NULL`. What produces a miss during the expand window is a name this zoo has no enclosure for - a typo, or a yard that has since been deleted - and a `NULL` there is the same honest answer the backfill gave Biscuit, for the same reason: the record names a yard that does not exist, and inventing one to keep the column full is how a data problem becomes a silent one. The API reports `"enclosure": null` on exactly the rows where that happened, and 14.7's `psql` verify shows you the same rows with an empty `enclosure_id` column underneath.

One thing this lookup does **not** buy you is worth stating plainly, because the version of it you would write by instinct does. The backfill matches a name, so it carries no opinion about environment: `{"species": "African bush elephant", "enclosure": "Jungle House"}` resolves to a real enclosure id and is written without complaint. The environment check that would have caught it by construction is not here - which is the whole reason 14.4 exists, and why its `ErrWrongEnvironment` is doing real work from the moment it lands rather than serving as a redundant second check.

Get this wrong in either direction and the failure is quiet rather than loud. Omit `enclosure` and the insert is refused (`null value in column "enclosure" of relation "animals" violates not-null constraint`) - loud, and you fix it in a minute. Omit `zoo_id` and `enclosure_id` instead and the row is created and *invisible*: `enclosure_id` scans back as `NULL` so the API reports `"enclosure": null`, and with `zoo_id` NULL the row matches none of 14.5's tenant-scoped `WHERE zoo_id = $n` queries - and under the row-level policy it is invisible to every role that is subject to it (your `zoo` owner role is not, which is exactly why the policy needs the `FORCE ROW LEVEL SECURITY` note 14.5 ends on). Nothing errors. That asymmetry - one path shouts, the other whispers - is why the expand-and-contract write is worth doing deliberately rather than by leaving `Create` alone and hoping.

The same question lands on `zoo_id`'s value: an animal has to belong to *some* zoo at creation, and the honest answer is the authenticated caller's zoo (the same claim 14.5 says it cannot source for you yet). Until that exists, `Create` writing the single-zoo constant is the interim that keeps the fixture working, and it is the piece of this section most worth not shipping.

**One consequence of the `RETURNING` shape that bites the write path specifically.** Stage 6's `Create` and `Update` both `RETURNING selectColumns` - a single-table clause that cannot join - so the row that comes back carries `enclosure_id` but no `enclosures` fields to fill `*enclosureRef` with. Map it straight through and every `POST`/`PUT` replies with `"enclosure": null` even for a name the lookup just resolved, which is exactly the kind of "the write worked but the response lies" bug that a read path joined correctly will hide from you. The fix is a read-back: after the write, call the joined read with the new id (or put the write in a CTE and select the joined shape from it). It costs one extra query on the write path, where you are already paying for a transaction, and it is the same trade 6.2's `Get`-carries-the-join decision made - the response is a *view* of the row, and only the joined query produces that view.

**Contract** - later, in a separate deploy, once you are certain nothing reads the old column:

```sql
ALTER TABLE animals ALTER COLUMN zoo_id SET NOT NULL;
ALTER TABLE animals DROP COLUMN enclosure;
```

The gap between expand and contract is measured in deploys, not minutes: you can only drop `enclosure` after the *oldest* running version of your service has stopped reading it. That is why the pattern exists, and why these two `ALTER`s are deliberately not in `00006` with the first two.

**Do not add this as a migration file yet.** goose applies every file it finds, in number order, so a contract sitting in `migrations/` gets applied on the next `zoo migrate` - which is the opposite of "a later deploy". When you are actually ready for the second deploy, *that* is when the two statements above become a migration file of their own, numbered after everything that exists in `migrations/` by then (and shipped alongside the Go sweep that stops selecting `animals.enclosure`). The rest of this bonus keeps adding files - 14.5 and 14.6, then Bonus 15's and 16's - so by the time you would write the contract it is no longer `00007`; the number is whatever is free on that day, and the point is only that it is a *later* file than the expand. Until then these statements are here so you can see where the story ends, not so you can run them.

### 14.4 Capacity: the rule a CHECK constraint cannot express

"An enclosure may not hold more animals than its capacity" is a rule about *many rows*, and a `CHECK` constraint only ever sees one row. Postgres offers three ways out, and it is worth knowing which one you are choosing:

1. **A trigger** (`BEFORE INSERT OR UPDATE ... FOR EACH ROW`) that counts and raises. It works, it is invisible to anyone reading the Go code, and it makes every write slower. Good for rules that must hold no matter who writes.
2. **An `EXCLUDE` constraint** (Bonus 15.2 uses one for shifts) that expresses the rule as a *conflict between rows*. Brilliant when the rule is "no two of these may overlap"; there is no clean way to spell "no more than n of these" with it.
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
func (r *dbRepository) PlaceAnimal(ctx context.Context, zooID, animalID, enclosureID pgtype.UUID) error {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx)

	// Lock order matters and is not local to this function: Bonus 16.4's
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

- **`FOR UPDATE`** takes a row-level lock, held until the transaction ends. A second `PlaceAnimal` for the same enclosure blocks on that line rather than failing: measured on Postgres 17, a request arriving while another transaction held the lock simply waited it out and then proceeded (the exact wait depends on your machine, but the shape does not - it queues, it does not error). That serialisation is what a capacity check needs, and it is why the *enclosure* is locked and not merely the animal: two animals may move into the same enclosure concurrently, but they must not both be the one that fills it. Note that both rows are locked, animal first, and in a fixed order - 16.4 takes the same pair, and a stated order is the only thing that keeps them from deadlocking.
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

> **These repository functions are shown, not wired.** `PlaceAnimal` compiles and does exactly what is described, but this section gives it no handler and no route, so nothing in 14.7 exercises it over HTTP - the verify drives the capacity rule with plain SQL instead, which is what you want to watch anyway (the rule is the database's, and seeing it hold without any Go in the loop is the point). Adding `PUT /api/v1/animals/{id}/enclosure` is a real exercise: the handler is three lines, but the tenant scoping of 14.5 has to be settled first, which is the next section. Read the Go here as the shape a correct implementation takes, not as a file to paste and forget.

> **A note on where this rule belongs.** It lives in the repository because it needs the transaction, but it is a *business* rule, and a reviewer is entitled to ask why it is not in the service. The honest answer: the service could own the rule and pass the repository a "check this for me" callback, but that buys indirection, not clarity. What matters is that exactly one place enforces it - two places enforcing it with different SQL is how tenants end up with three elephants in a two-elephant yard.

### 14.5 Not leaking another zoo's rows

Here is the failure mode, stated plainly: **every query that touches animals now needs `WHERE zoo_id = $n`, and the day someone writes one without it, North Zoo reads South Zoo's animals.** It will pass review, because the query is otherwise correct. It will pass tests, because a test that only has one zoo's data cannot detect it.

There are two defences, and the right answer is usually both.

**Defence one: make the unscoped query impossible to write.** Change the repository's method set so that no method can list animals without a zoo:

```go
// Every read takes the zoo it is allowed to see. Note what does not exist:
// there is no ListAnimals(ctx) with no zoo parameter, so there is no call
// site that can accidentally read across tenants.
func (r *dbRepository) List(ctx context.Context, zooID pgtype.UUID, keeperID pgtype.UUID) ([]Animal, error)
func (r *dbRepository) Get(ctx context.Context, zooID, id pgtype.UUID) (Animal, string, error)
```

The compiler then enforces what a code review only hopes for. This is the single highest-value change in this whole bonus section, and it costs nothing but a signature.

Note the same rule as 14.4's Go: this is the shape the method set should *take*, not a change to paste in today. Every handler and every test currently calls `List(ctx, keeperID)` and `Get(ctx, id)`, and neither call site will compile against the new signatures until `zooID` has a source. So make them one commit, not two - change the signature in the same change that gives the claim somewhere to come from, or you will spend an afternoon threading a value you have not yet decided how to obtain. If you want the enforcement now and the claim later, the honest interim is a separate method (`ListForZoo`) added alongside, with the old one deleted when the routes move over; that is more code, but it keeps the tree compiling the whole way.

> **Where does `zooID` come from?** That is the question this signature deliberately raises, and this section does not answer it for you, because the answer is a design decision rather than a line of code. The tenant has to be something the *caller proves*, not something they ask for: a `?zoo_id=` query parameter is a request to read another zoo, not a credential, and anyone can send any value. The shape that works is to make it part of the identity you already trust - a `zoo_id` claim in the JWT (so a zookeeper belongs to a zoo, set at login), or a membership row looked up from the authenticated keeper - and then pass it from the middleware into the handler the same way `Claims.ZookeeperID` already travels (Stage 4's context key). Wiring that is a genuine piece of work across the auth package and every animal route, which is why it is named here as the gap it is rather than hidden behind a snippet that pretends to close it. RLS (below) is the backstop for when it is wired wrong.

**Defence two: row level security, so even a mistake cannot leak.** Postgres can filter rows by policy, per connecting role, with no help from your SQL (`migrations/00007_row_level_security.sql`):

```sql
-- +goose Up
CREATE ROLE zoo_app LOGIN PASSWORD 'change-me';
GRANT USAGE ON SCHEMA public TO zoo_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO zoo_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO zoo_app;

ALTER TABLE animals ENABLE ROW LEVEL SECURITY;

CREATE POLICY animals_by_zoo ON animals
    USING (zoo_id = nullif(current_setting('app.zoo_id', true), '')::uuid);

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

- With `app.zoo_id` set to South Zoo's id (`00000000-0000-4000-8000-000000000002`), `SELECT * FROM animals` returns only South Zoo's rows - none, in this fixture; with North Zoo's id (`00000000-0000-4000-8000-000000000001`), all three. The policy is doing the filtering, and the query has no `WHERE zoo_id` at all.
- With the tenant **never set**, the same query returns **zero rows**: `nullif` yields NULL, `NULL = anything` is NULL rather than true, and the policy fails closed. A request that forgot to set a tenant sees nothing rather than everything.

And the trap, which cost a real debugging session before it was written down: `current_setting('app.zoo_id', true)` returns **NULL** when the setting was never set, but returns **the empty string** after a `RESET`, because the setting now exists with an empty value. `''::uuid` is a hard error - `invalid input syntax for type uuid: ""` - which surfaces as a 500 rather than an empty result. The `nullif(..., '')` above is not defensive padding; it is the fix, and it is exactly the sort of thing that only shows up when a connection is reused by a later request. Which is always, with a pool.

> **What RLS costs you.** The policy is attached to the *role*, so the role that owns the tables bypasses it by default - `ALTER TABLE ... FORCE ROW LEVEL SECURITY` changes that, for a plain owner. It cannot change it for a superuser: superusers and roles with `BYPASSRLS` bypass row security unconditionally, `FORCE` or not. That matters here, because the official Postgres image creates `POSTGRES_USER` - this tutorial's `zoo` - as a superuser with `BYPASSRLS` (verify it: `SELECT rolsuper, rolbypassrls FROM pg_roles WHERE rolname = 'zoo'`). So `FORCE` is a no-op for the connection you have been using all tutorial, and parity between your owner and your application role is the thing `FORCE` buys you *elsewhere*, not here. Every new table needs its own policy or it is unprotected, which is a list that drifts. And the tenant has to be set on the same connection that runs the query, which with a pooled connection means inside a transaction (`set_config(..., true)` is transaction-local) rather than once at startup. None of that is a reason not to do it; all of it is a reason to do defence one first and RLS second, and to make sure the role your *application* connects as is not the superuser the container bootstraps with.

### 14.6 Reporting: occupancy without a counting query per request

"Which enclosures are full, and which have space?" is a question an operations screen asks every few seconds, and it is a `GROUP BY` over the whole `animals` table. Materialise it (`migrations/00008_zoo_occupancy_view.sql`; the `Up` is the two statements, the `Down` drops the view):

```sql
-- +goose Up
CREATE MATERIALIZED VIEW zoo_occupancy AS
SELECT z.id AS zoo_id, z.name AS zoo,
       e.id AS enclosure_id, e.name AS enclosure, e.capacity,
       count(a.id) AS occupants
FROM zoos z
JOIN enclosures e ON e.zoo_id = z.id
LEFT JOIN animals a ON a.enclosure_id = e.id
GROUP BY z.id, z.name, e.id, e.name, e.capacity;

CREATE UNIQUE INDEX zoo_occupancy_enclosure_idx ON zoo_occupancy (enclosure_id);

-- +goose Down
DROP MATERIALIZED VIEW zoo_occupancy;
```

The unique index is not optional decoration. Without it, `REFRESH MATERIALIZED VIEW CONCURRENTLY zoo_occupancy` fails with `cannot refresh materialized view ... concurrently` and you are left with the plain `REFRESH`, which takes an `ACCESS EXCLUSIVE` lock and blocks every reader for the duration. With the index, the concurrent refresh is allowed and readers are never blocked. Verified on Postgres 17 in both directions, because the error message is good but the reason is not obvious.

A materialised view is a cache with a manual invalidation button, so decide *who* presses it: a `REFRESH ... CONCURRENTLY` at the end of every feed and transfer transaction is fine at this scale and wrong at a bigger one, where the answer becomes an incremental view (`REFRESH ... CONCURRENTLY` on a schedule) or a summary table updated by the same transaction that changed it.

### 14.7 Verify

Everything below is `psql` rather than `curl`, because every claim in this section is about the database's behaviour, and the database is where you should check it.

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo
```

```sql
-- the schema exists and the animals were backfilled
\d enclosures
SELECT id, name, species, zoo_id, enclosure_id FROM animals ORDER BY id;
--  00000000-0000-4000-8000-000000000001 | Tembo   | African bush elephant | 00000000-0000-4000-8000-000000000001 | 00000000-0000-4000-8000-000000000001
--  00000000-0000-4000-8000-000000000002 | Suki    | Sumatran tiger        | 00000000-0000-4000-8000-000000000001 | 00000000-0000-4000-8000-000000000002
--  00000000-0000-4000-8000-000000000003 | Biscuit | Red panda             | 00000000-0000-4000-8000-000000000001 |       (NULL)
-- Biscuit's row names "Forest Yard"; North Zoo has no enclosure by that name,
-- so the backfill leaves him NULL rather than guessing.

-- capacity is a rule about many rows, so plain SQL happily breaks it: the
-- jungle yard (id 00000000-0000-4000-8000-000000000002) holds one animal and
-- its capacity is 1
UPDATE animals SET enclosure_id = '00000000-0000-4000-8000-000000000002' WHERE id = '00000000-0000-4000-8000-000000000001';
SELECT count(*) FROM animals WHERE enclosure_id = '00000000-0000-4000-8000-000000000002';   -- 2, over capacity
-- plain SQL walked past the rule and made an elephant share a one-animal yard.
-- The check that refuses it is 14.4's PlaceAnimal, inside its transaction - and
-- this is exactly why it has to be there: no constraint can see the row you are
-- about to add alongside the ones already present.
UPDATE animals SET enclosure_id = '00000000-0000-4000-8000-000000000001' WHERE id = '00000000-0000-4000-8000-000000000001';      -- back where the backfill put it

-- tenancy. This part only means anything as a role that does NOT own the
-- table, because a table's owner bypasses its own policies (14.5 says so).
SET ROLE zoo_app;                -- the role 14.5's migration created
SET app.zoo_id = '00000000-0000-4000-8000-000000000001';
SELECT count(*) FROM animals;    -- 3: North Zoo's animals
RESET app.zoo_id;
SELECT count(*) FROM animals;    -- 0: no tenant set, so nothing is visible
RESET ROLE;

-- and the report
REFRESH MATERIALIZED VIEW CONCURRENTLY zoo_occupancy;
SELECT * FROM zoo_occupancy ORDER BY zoo_id, enclosure_id;
--  00000000-0000-4000-8000-000000000001 | North Zoo | 00000000-0000-4000-8000-000000000001 | Savanna Yard  | 3 | 1
--  00000000-0000-4000-8000-000000000001 | North Zoo | 00000000-0000-4000-8000-000000000002 | Jungle House  | 1 | 1
--  00000000-0000-4000-8000-000000000002 | South Zoo | 00000000-0000-4000-8000-000000000003 | South Savanna | 2 | 0
```

Two things in that scroll are worth more than the SQL they sit next to.

**`SET ROLE zoo_app` is not decoration.** Run those same two `SELECT`s as the owner and every one of them "passes" while proving nothing: the owner sees all three rows with a tenant set and all three with none, because row level security does not apply to a table's owner by default. That is the difference between a green test and a meaningful one, and it is why the verification names the role explicitly.

**The assertion worth keeping** is the second one: as `zoo_app`, with `app.zoo_id` unset, `SELECT count(*) FROM animals` must return **0**. If it ever returns the whole table, one of two things has happened - the policy was dropped, or someone connected as a role that bypasses it - and the count alone will not tell you which. Do not reach for `ALTER TABLE animals FORCE ROW LEVEL SECURITY` expecting to close the second case: it makes a plain table owner subject to the policy, and it does nothing at all for a superuser or a `BYPASSRLS` role, which is what `zoo` is. Reproduced against this tutorial's own compose file - `FORCE` on, tenant unset, `zoo` still counts all three animals, while `zoo_app` counts zero. Parity between those two needs a non-superuser owner, which is a change to who your application connects as, not a statement you can add to a migration. That is the shape of the lesson: RLS is a fence around the roles you point at it, and picking the roles is the part that decides whether it holds.

---

[Stage 13](13-uuid-primary-keys.md)  |  [Overview](../tutorial.md)  |  [Bonus 15](15-bonus-scheduling-payroll.md)
