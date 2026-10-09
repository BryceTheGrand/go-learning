## Bonus 16: Moving animals between zoos, and where to go next

The last bonus section, and the one that ties the others together: once there is more than one zoo (Bonus 14), a roster (Bonus 15), and capacity that matters, you inevitably need to *move an animal from one to another* - and that turns out to be the first thing in this tutorial that is a **workflow** rather than a request.

It closes with the three things that a system at this point always grows next (permissions that outgrow a `role` column, an audit trail, and a sense of when to stop), and then hands you back to Stage 12's exercise list.

As before, every query was executed against Postgres 17, including the ones that fail.

### 16.1 A transfer is not an UPDATE

Every id in this section is a uuid now (Stage 13), and a uuid is not something you retype from one line to the next. So the examples below are pinned to fixed, obviously-synthetic ids - and by the time you reach this section those rows are **already in your database**: Bonus 14's migration seeded the zoos and enclosures, Bonus 14.3 rebuilt the animals against those ids, and Bonus 15's keeper block pinned maya and sam. This is a recap of what is there, not a fixture to run:

| table | id | row |
|---|---|---|
| `zookeepers` | `...0001` | maya (keeper) |
| `zookeepers` | `...0002` | sam (keeper) |
| `zoos` | `...0001` | North Zoo |
| `zoos` | `...0002` | South Zoo |
| `enclosures` | `...0001` | North Zoo, Savanna Yard, savanna, capacity 3 |
| `enclosures` | `...0002` | North Zoo, Jungle House, jungle, capacity 1 |
| `enclosures` | `...0003` | South Zoo, South Savanna, savanna, capacity 2 |
| `animals` | `...0001` | Tembo, African bush elephant, North Zoo, Savanna Yard |
| `animals` | `...0002` | Suki, Sumatran tiger, North Zoo, Jungle House |
| `animals` | `...0003` | Biscuit, Red panda, North Zoo, no enclosure |

Confirm it rather than re-inserting - a second `INSERT` of these rows collides with the ones already there (and, on a database where Bonus 14.3's contract has not been run, omits the still-`NOT NULL` `animals.enclosure` column and fails outright):

```sql
SELECT a.id, a.name, a.zoo_id, a.enclosure_id
FROM animals a ORDER BY a.id;
```

(Starting from a scratch database instead of following in order? Bonus 14.3's rebuild and Bonus 15's keeper block are the two snippets that create the rows above; this section cannot run without them, because `transfers` references both `animals` and `zoos`.)

Now the tempting implementation, which is one statement:

```sql
-- do not do this. It is one statement, and it works.
UPDATE animals SET zoo_id = '00000000-0000-4000-8000-000000000002',
                   enclosure_id = '00000000-0000-4000-8000-000000000003'
 WHERE id = '00000000-0000-4000-8000-000000000001';

-- put it back, so the rest of this section starts from where 14 left off
UPDATE animals SET zoo_id = '00000000-0000-4000-8000-000000000001',
                   enclosure_id = '00000000-0000-4000-8000-000000000001'
 WHERE id = '00000000-0000-4000-8000-000000000001';
```

Everything wrong with it is invisible in the SQL - including the fact that it succeeds - which is why it is worth enumerating:

- **Nobody approved it.** Moving an animal between sites is a regulated act (transport, health certificates, welfare checks). It needs a request, a decision, and a record of who made it.
- **The destination's capacity was never checked.** Bonus 14.4 built the guard; a bare `UPDATE` walks straight past it.
- **There is no history.** Six months later, "when did Tembo move, and who authorised it?" has no answer, because the row that would have held the answer was overwritten.
- **It cannot be half-done.** A real transfer has states - requested, approved, in transit, completed - and each transition is a different person's job.

So the thing being modelled is not the animal's location. It is the *request*, and the animal's location is a side effect of the request reaching its final state.

### 16.2 The schema

`migrations/00013_animal_transfers.sql`:

```sql
-- +goose Up
CREATE TABLE transfers (
    id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    animal_id       uuid NOT NULL REFERENCES animals(id) ON DELETE CASCADE,
    from_zoo_id     uuid NOT NULL REFERENCES zoos(id),
    to_zoo_id       uuid NOT NULL REFERENCES zoos(id),
    to_enclosure_id uuid REFERENCES enclosures(id),
    status          text NOT NULL DEFAULT 'requested'
                    CHECK (status IN ('requested', 'approved', 'completed', 'cancelled')),
    requested_by    uuid NOT NULL REFERENCES zookeepers(id),
    requested_at    timestamptz NOT NULL DEFAULT now(),
    decided_by      uuid REFERENCES zookeepers(id),
    decided_at      timestamptz,
    completed_at    timestamptz,
    CONSTRAINT transfers_different_zoos CHECK (from_zoo_id <> to_zoo_id)
);

-- At most one transfer in flight per animal. The WHERE clause makes this a
-- PARTIAL index, so it constrains only the open rows: an animal may have any
-- number of completed transfers in its history.
CREATE UNIQUE INDEX transfers_one_open_per_animal
    ON transfers (animal_id)
    WHERE status IN ('requested', 'approved');

-- +goose Down
DROP TABLE transfers;
```

Three things there are worth naming, and the second is the trick you will reuse.

- **`status` is text with a `CHECK`, not an enum type.** Either is defensible. A `CHECK` is a one-line change to extend; a Postgres `enum` is a type whose values are ordered and which cannot be dropped from without a migration. For a state machine that will grow a state or two, the `CHECK` is the smaller commitment. (Bonus 14.2 makes the opposite call for `environments` and explains why: that one is *data*, this one is *structure*.)
- **The partial unique index is how you say "at most one open X per Y".** `WHERE status IN ('requested','approved')` means the index only contains the rows that are still in flight, so it refuses a second *open* transfer while leaving the closed ones alone. Verified both ways: a second request for the same animal fails with `duplicate key value violates unique constraint "transfers_one_open_per_animal"` (`23505`), and once the first is cancelled, a new request for the same animal inserts cleanly. This single index replaces a "check for an existing transfer" method that would race with itself.
- **`CONSTRAINT transfers_different_zoos CHECK (from_zoo_id <> to_zoo_id)`** is a rule that costs nothing to state and is embarrassing to get wrong. A "transfer" from a zoo to itself is a move between enclosures, which is Bonus 14.4's job.

### 16.3 Transitions as guarded UPDATEs

A state machine is easy to write badly. The bad version reads the row, checks the status in Go, and writes:

```go
// the race: two requests both read 'approved', both decide to proceed
if transfer.Status == "approved" {   // ... another request does the same, right here
	// update to completed
}
```

Between the read and the write, another request can do the same read. The good version puts the precondition in the `WHERE` clause, so the read and the write are one atomic statement:

```sql
UPDATE transfers
SET status = 'completed', completed_at = now()
WHERE id = $1 AND status = 'approved';
```

and then checks `tag.RowsAffected()` - Stage 3.2's `Exec` plus row count, doing real work for the first time. Verified behaviour, against the request on Tembo that 16.8 inserts (`00000000-0000-4000-8000-000000000006`, open and waiting on a decision):

```sql
-- jumping straight from requested to completed
UPDATE transfers SET status='completed' WHERE id='00000000-0000-4000-8000-000000000006' AND status='approved';   -- UPDATE 0

-- the legal transition
UPDATE transfers SET status='approved'  WHERE id='00000000-0000-4000-8000-000000000006' AND status='requested';  -- UPDATE 1
UPDATE transfers SET status='completed' WHERE id='00000000-0000-4000-8000-000000000006' AND status='approved';   -- UPDATE 1

-- replaying it
UPDATE transfers SET status='completed' WHERE id='00000000-0000-4000-8000-000000000006' AND status='approved';   -- UPDATE 0
```

**Read those four line counts as a transcript, not as a fixture you can re-run at will.** They are true at exactly one point: `...0006` freshly inserted and still `requested`, which is what 16.8's block produces in its middle, between the insert and its own copy of these same four statements. Run them anywhere else and every one is `UPDATE 0` - before 16.8 the row does not exist yet, and after 16.8's block has run it is already `completed`, so no `WHERE` matches. That is not flakiness; it is the row count telling the truth about the state machine, and it is the same property that makes the guard work. If you want a throwaway to poke at, insert your own `transfers` row with a literal id and drive that instead of borrowing `...0006`.

**`UPDATE 0` is not an error; it is the answer.** Zero rows affected means "the precondition did not hold", and the caller maps that to `409 Conflict` ("this transfer has already been decided") exactly as Stage 9 maps its other sentinels. Two properties fall out for free, and both matter at scale:

- **Concurrency-safety without a lock.** Two simultaneous completions cannot both succeed, because the second one's `WHERE` no longer matches.
- **Retry-safety.** A client that times out and retries the completion gets `UPDATE 0`, which the API can translate into "already done" rather than "conflict". Deciding which of those two it means is a policy choice - but note that you *can* decide, because the row is still there and still says `completed`, which is the whole reason for keeping history rather than deleting it.

### 16.4 Completing a transfer: one transaction, re-checked from scratch

Approval and completion are separated by hours or weeks, and the world moves in between: the animal may have been moved by hand, the destination enclosure may have filled up. So completion re-validates everything the approval assumed, in one transaction.

It refuses in two ways, and both are new sentinels declared in 14.4's vocabulary (the capacity and environment errors it also returns already exist, from that section):

```go
var (
	ErrTransferNotApprovable = httpx.Conflict("transfer_not_approvable", "this transfer is not awaiting completion")
	ErrAnimalMoved           = httpx.Conflict("animal_moved", "the animal is no longer in the zoo the approval named")
)
```

Both are `409`s rather than `500`s for the reason 14.4 gives: a refusal that the caller could have expected is a state of the world, not a bug, and `httpx.Respond` reads the status off the error so the handler needs no code for either. `ErrTransferNotApprovable` also covers the replay case - a second completion of the same transfer hits `UPDATE 0` and reports a conflict, which is what 16.3 said the row count was for.

```go
// CompleteTransfer moves the animal and closes the request atomically. The
// approval decision is not trusted as a statement of current fact: the
// animal is locked, its zoo re-read, and the destination's capacity
// re-checked, because any of those can have changed since it was approved.
func (r *dbRepository) CompleteTransfer(ctx context.Context, transferID pgtype.UUID) error {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx)

	var animalID, fromZooID, toZooID pgtype.UUID
	var toEnclosureID pgtype.UUID
	err = tx.QueryRow(ctx,
		`SELECT animal_id, from_zoo_id, to_zoo_id, to_enclosure_id
		 FROM transfers
		 WHERE id = $1 AND status = 'approved'`, transferID).
		Scan(&animalID, &fromZooID, &toZooID, &toEnclosureID)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return ErrTransferNotApprovable
		}
		return err
	}

	// Lock the animal first, then the enclosure: always in this order, so
	// two concurrent transfers cannot deadlock by taking them in opposite
	// orders. Lock ordering is a convention you have to choose and then keep.
	// The species comes along because reserveSlot needs it for the environment
	// check - read it under the lock you already hold rather than re-querying.
	var currentZooID pgtype.UUID
	var species string
	if err := tx.QueryRow(ctx,
		`SELECT zoo_id, species FROM animals WHERE id = $1 FOR UPDATE`, animalID).
		Scan(&currentZooID, &species); err != nil {
		return err
	}
	if currentZooID != fromZooID {
		return ErrAnimalMoved
	}

	if toEnclosureID.Valid {
		if err := r.reserveSlot(ctx, tx, toEnclosureID, toZooID, species, animalID); err != nil {
			return err
		}
	}

	if _, err := tx.Exec(ctx,
		`UPDATE animals
		 SET zoo_id = $1, enclosure_id = $2, updated_at = now()
		 WHERE id = $3`,
		toZooID, toEnclosureID, animalID); err != nil {
		return err
	}

	tag, err := tx.Exec(ctx,
		`UPDATE transfers
		 SET status = 'completed', completed_at = now()
		 WHERE id = $1 AND status = 'approved'`, transferID)
	if err != nil {
		return err
	}
	if tag.RowsAffected() != 1 {
		return ErrTransferNotApprovable
	}

	return tx.Commit(ctx)
}
```

> **Shown, not wired - and here is exactly what wiring it would take.** This is the third method in the bonuses (after 14.4's `PlaceAnimal` and 15.4's shift check) that is real, compiling code no route calls. What is missing is not a parameter but a route and a permission: a handler on `POST /api/v1/transfers/{id}/complete`, mounted behind `requireAdmin` beside the other mutations, that reads the transfer id from the path and calls `CompleteTransfer` with it. Note what is *not* in that sentence: the caller never supplies a zoo id or an enclosure id. Every value the move needs - the animal, the zoo it is leaving, the zoo and enclosure it is going to - is read from the `transfers` row that an earlier approval wrote, which is what makes the re-validation at the top of the function meaningful: the approval is the caller's input, and everything else is the database's current fact. A route that accepted `to_zoo_id` in the body would let the caller restate the destination at completion time, which is the one thing this design deliberately does not allow.

The `reserveSlot` call inside is Bonus 14.4's capacity guard lifted out into a method of its own, so that moving an animal *within* a zoo and moving it *between* zoos share one implementation:

```go
// reserveSlot is the capacity and environment guard from 14.4. The enclosure
// comes before the zoo in the signature for a reason worth stating: both are
// pgtype.UUID, so swapping them compiles, passes vet, and then checks capacity
// against the wrong tenant. Two ids of the same type sitting next to each
// other, always in the same order, is the cheap defence.
//
// It needs two values beyond the enclosure it is guarding: the species (to
// check the enclosure's environment) and the animal being moved (to exclude it
// from the occupant count, so re-placing an animal already in the enclosure
// does not count it twice). Both are read by the caller under the animal lock
// it already holds, so they arrive as arguments rather than being re-queried.
func (r *dbRepository) reserveSlot(
	ctx context.Context,
	tx pgx.Tx,
	enclosureID, zooID pgtype.UUID,
	species string,
	animalID pgtype.UUID,
) error {
	// SELECT environment, capacity FROM enclosures
	//  WHERE id = $1 AND zoo_id = $2 FOR UPDATE
	// then count the other occupants (id <> animalID), then check species'
	// environment against the enclosure's, returning ErrEnclosureNotFound,
	// ErrWrongEnvironment or ErrEnclosureFull. The body is 14.4's guard
	// moved, not rewritten; only the signature is new.
}
```

(14.4's inline version got the species and the animal id from the locks it was already holding, which is why an extracted function with only the enclosure and the zoo in its signature could not work: the two values the guard needs are not derivable from those. Passing them in is what keeps "one implementation of the capacity rule" true rather than nominal.)

Three of the types change here and nothing else: `transferID` is a `pgtype.UUID`, every id read out of the transfer is a `pgtype.UUID`, and the optional destination enclosure is a `pgtype.UUID` whose `Valid` is false when the column is `NULL` - Stage 13's rule, so no `*int64` and no pointer. The comparison `currentZooID != fromZooID` still reads the way it did, because `pgtype.UUID` is a plain comparable struct.

Four decisions in that function are the ones to carry away:

- **Re-read, do not trust.** The transfer row said the animal was at `from_zoo_id` when it was approved. This code checks that it still is, and returns `ErrAnimalMoved` if not, rather than blindly applying a stale decision.
- **Lock ordering.** The animal is locked before the enclosure, and 14.4's `PlaceAnimal` locks the same two rows in the same order - which is the point, since a transfer and a placement can be in flight at once. Two transactions that grabbed those rows in opposite orders would deadlock; Postgres detects that and kills one, which is safe but ugly and hard to reproduce on demand. A stated order is the cheap fix, and it only works if it is written down in both places, which is exactly what the two functions now do - and exactly the sort of thing that drifts, because nothing in the language enforces a comment.
- **One implementation of the capacity rule.** `reserveSlot` is extracted the moment it has a second caller. Duplicating it would not be a DRY violation so much as a correctness one: two copies of a capacity check drift, and the one that drifts is the one nobody is testing. So 14.4's `PlaceAnimal` loses its inline version in the same change - its `SELECT ... FOR UPDATE` on the enclosure, the occupant count, and the environment check all move into `reserveSlot`, and `PlaceAnimal` calls it exactly as `CompleteTransfer` does. Leaving both is the drift this bullet is warning about: the code compiles, both paths work, and the next fix lands in one of them.
- **The animal's `feed_log` history does not move with it**, and should not: `feed_log.keeper_id` references the keeper who fed it, whoever they were and wherever they worked. The animal's past is a fact about the animal, not about the zoo that currently houses it. This is the kind of question ("do we partition history by tenant?") that deserves an explicit answer, and "no, history stays with the animal" is a defensible one.

### 16.5 When a role column stops being enough

`zookeepers.role` has been a `text` column with two legal values since Stage 2, and by now it is straining:

- Adding a `vet` role means changing a `CHECK`, and it cannot express "a vet at one zoo but not another".
- `RequireRole("admin")` is a string comparison in middleware, so the set of things an admin can do is spread across the route table rather than declared in one place.
- There is no way to grant one person one extra power - "Sam may run payroll but not edit animals" - without inventing a role for the combination.

The standard answer is four tables and a function (`migrations/00014_roles_and_permissions.sql`). The lookup keys stay `text` - a role name is a stable natural key, never guessed and never merged (Stage 13.1 says why) - while the columns that point at an entity are now `uuid`:

```sql
-- +goose Up
CREATE TABLE roles          (name text PRIMARY KEY, description text NOT NULL);
CREATE TABLE permissions    (name text PRIMARY KEY, description text NOT NULL);
CREATE TABLE role_permissions (
    role       text NOT NULL REFERENCES roles(name) ON DELETE CASCADE,
    permission text NOT NULL REFERENCES permissions(name) ON DELETE CASCADE,
    PRIMARY KEY (role, permission)
);

CREATE TABLE zookeeper_roles (
    zookeeper_id uuid NOT NULL REFERENCES zookeepers(id) ON DELETE CASCADE,
    role         text NOT NULL REFERENCES roles(name),
    zoo_id       uuid REFERENCES zoos(id) ON DELETE CASCADE,   -- NULL = every zoo
    UNIQUE NULLS NOT DISTINCT (zookeeper_id, role, zoo_id)
);

-- +goose StatementBegin
CREATE FUNCTION has_permission(zk uuid, perm text, zoo uuid)
RETURNS boolean LANGUAGE sql STABLE AS $$
    SELECT EXISTS (
        SELECT 1
        FROM zookeeper_roles zr
        JOIN role_permissions rp ON rp.role = zr.role
        WHERE zr.zookeeper_id = zk
          AND rp.permission = perm
          AND (zr.zoo_id IS NULL OR zr.zoo_id = zoo)
    )
$$;
-- +goose StatementEnd

-- Enough seed data for the four combinations below to mean anything. Note
-- the two shapes of grant: maya is a vet at South Zoo only, and a keeper
-- everywhere (zoo_id NULL).
INSERT INTO roles (name, description) VALUES
    ('keeper', 'feeds and looks after animals'),
    ('vet',    'clinical authority'),
    ('admin',  'manages accounts and roster');
INSERT INTO permissions (name, description) VALUES
    ('animal:read',     'see animals'),
    ('animal:write',    'create and move animals'),
    ('animal:medicate', 'record treatment'),
    ('payroll:run',     'produce payslips');
INSERT INTO role_permissions (role, permission) VALUES
    ('keeper', 'animal:read'), ('keeper', 'animal:write'),
    ('vet',    'animal:read'), ('vet',    'animal:medicate'), ('vet', 'animal:write'),
    ('admin',  'animal:read'), ('admin',  'payroll:run');
INSERT INTO zookeeper_roles (zookeeper_id, role, zoo_id) VALUES
    ('00000000-0000-4000-8000-000000000001', 'vet',    '00000000-0000-4000-8000-000000000002'),
    ('00000000-0000-4000-8000-000000000001', 'keeper', NULL);

-- +goose Down
DROP FUNCTION has_permission(uuid, text, uuid);
DROP TABLE zookeeper_roles, role_permissions, permissions, roles;
```

- **`NULLS NOT DISTINCT`** (Postgres 15 and later) is the quiet hero. A `UNIQUE` constraint normally treats two `NULL`s as different, so `('00000000-0000-4000-8000-000000000001', 'keeper', NULL)` could be inserted twice and mean "keeper everywhere" twice. `NULLS NOT DISTINCT` makes the nulls compare equal, so the natural key behaves the way you expect. Verified: maya as `vet` scoped to South Zoo, and as `keeper` unscoped, resolves correctly across all four combinations - vet at South (`t`), vet at North (`f`), keeper anywhere (`t`), payroll admin (`f`).
- **`zoo_id IS NULL` means "every zoo"**, which is how the existing global roles keep working during the migration. The `OR` in the function is a little blunt, but it is a documented and testable convention rather than an accident.
- **The last `INSERT` points at `...0001`, and that row has to exist.** It is the keeper Bonus 15.2's block pinned as maya; if you skipped that block, this migration stops with `violates foreign key constraint "zookeeper_roles_zookeeper_id_fkey"` (`23503`) and nothing is left half-applied, because goose runs each file in one transaction. The constraint is doing the useful thing here - it is telling you which precondition you missed rather than storing a grant for a person who does not exist.
- **The middleware stops comparing strings.** `RequireRole("admin")` becomes `RequirePermission("payroll:run")`, and the route table starts reading as the policy document Stage 8 wanted it to be. The `role` claim in the JWT becomes a hint at best: **permissions change far more often than roles did**, which sharpens Stage 8's staleness warning considerably. A revoked permission inside a 24-hour token is a 24-hour window; this is the strongest argument in the tutorial for short token lifetimes plus refresh, and it is where you should spend the effort the deleted-zookeeper problem originally asked for.

> **This paragraph is the design, not a patch to apply today.** Two things would have to be settled before `RequirePermission` is more than a sketch. First, the route-to-permission mapping: nothing above says which permission guards `POST /api/v1/animals`, and writing that table is the actual work - it is also the point, since the mapping is what a reviewer should be reading. Second, where the check gets its answer: calling `has_permission` costs a database round trip on every request through the middleware, which is a real cost and the reason a cached or token-carried permission set is the usual next step. Do this when a third role genuinely appears (16.7 puts it sixth for that reason), and treat the function and its seed data - which 16.8 does verify - as the piece that is already done.

### 16.6 An audit trail, and who the database thinks you are

Regulated systems need "who changed this row, and when", and Postgres will write it for you (`migrations/00015_audit_log.sql`). The audited row's id is a uuid now, so `row_id` and `actor_id` are uuids too, and the actor is cast to match:

```sql
-- +goose Up
CREATE TABLE audit_log (
    id       uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    at       timestamptz NOT NULL DEFAULT now(),
    actor_id uuid,
    table_name text NOT NULL,
    row_id   uuid,
    action   text NOT NULL
);

-- +goose StatementBegin
CREATE FUNCTION audit_row() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO audit_log (actor_id, table_name, row_id, action)
    VALUES (nullif(current_setting('app.actor_id', true), '')::uuid,
            TG_TABLE_NAME, COALESCE(NEW.id, OLD.id), TG_OP);
    RETURN COALESCE(NEW, OLD);
END $$;
-- +goose StatementEnd

CREATE TRIGGER animals_audit
    AFTER INSERT OR UPDATE OR DELETE ON animals
    FOR EACH ROW EXECUTE FUNCTION audit_row();

-- The trigger above runs as whoever wrote the animal, and Bonus 14.5's GRANT was
-- a snapshot: `GRANT ... ON ALL TABLES IN SCHEMA public` covered the tables that
-- existed when it ran (00007), and audit_log was created nine migrations later.
-- Without this line the RLS role - the one 14.5 sets up as the production shape -
-- cannot write an animal at all, because the trigger's INSERT into audit_log is
-- refused. Grant it here, where the table is created, rather than trusting an
-- earlier blanket grant to reach a later table.
GRANT INSERT ON audit_log TO zoo_app;

-- +goose Down
DROP TRIGGER animals_audit ON animals;
DROP FUNCTION audit_row();
DROP TABLE audit_log;
```

> **Why the `StatementBegin`/`StatementEnd` fence, and why it is not optional here.** goose splits a migration file into statements on `;`, and the function body above contains two of them (`INSERT ... ;` and `RETURN ... ;`) that are *not* statement terminators. Without the fence, goose sends Postgres the first half of the function and the migration dies with `ERROR: unterminated dollar-quoted string ... (SQLSTATE 42601)` - the SQL is valid, but the runner is the one that has to parse it, and Stage 2.6 warned that it splits on semicolons. The fence tells goose to treat everything between the two annotations as one statement. `has_permission` in 16.5 is fenced defensively for the same reason: it happens to survive unfenced because its body has no interior semicolon, which is luck rather than design, and one added line would break it.

The subtle part is `actor_id`. The database has no idea who your HTTP caller was unless you tell it, and the mechanism is the same session-local setting Bonus 14.5 used for the tenant - the only difference is that the actor is now a uuid, so the setting is cast with `::uuid` rather than `::bigint`:

```sql
BEGIN;
SELECT set_config('app.actor_id', '00000000-0000-4000-8000-000000000001', true);   -- transaction-local
UPDATE animals SET enclosure_id = NULL WHERE id = '00000000-0000-4000-8000-000000000003';
COMMIT;
```

Verified: the update above writes an audit row with `actor_id = 00000000-0000-4000-8000-000000000001`, and an update *outside* any declared actor still writes the row with `actor_id` NULL. That asymmetry is deliberate and worth keeping - **audit fails open on attribution and closed on omission.** You always learn that something changed; you sometimes learn who, and a NULL actor in the log is a signal that some code path forgot to declare one, which is exactly the signal you want.

**The corollary lands immediately: the moment this migration is applied, your API starts writing NULL-actor rows.** Nothing in the Go sets `app.actor_id` - the trigger fires on `animals`, and every existing route that writes an animal (create, update, delete, `AssignKeeper`) now records a change with no actor attached. That is the trigger working, not a bug, and the verify below shows exactly this shape - it performs one update inside a declared actor and one outside, and the second lands with `actor_id` NULL. You can see the same thing from the API without touching `set_config` at all: `POST /animals`, then `SELECT actor_id, action, row_id FROM audit_log ORDER BY at DESC LIMIT 1` and note the NULL. Order by `at`, not by `id` - `audit_log.id` is a `gen_random_uuid()`, so `ORDER BY id DESC` is not "most recent", it is "whichever id sorts last", and it will happily hand you a row from an hour ago. Closing it is the same shape as 14.5's tenant: the middleware knows the caller's id already (`Claims.ZookeeperID`), so the fix is to have every animal write run inside a transaction that calls `set_config('app.actor_id', $1, true)` first - one helper at the repository boundary, not one call per handler. Until that exists, read the NULL rows as the backlog item they are rather than as evidence the trigger is broken.

Two honest caveats. First, trigger-based auditing writes one row per statement effect, so a bulk `UPDATE` touching ten thousand animals writes ten thousand audit rows inside one transaction - fine here, a throughput problem in a bigger system, where you move to logical decoding or an outbox table. Second, if a `DELETE` cascades (Stage 6's `feed_log` cascade) the trigger on the *child* table does not fire unless you install it there too, so "why did these feed rows vanish?" is answerable only if you thought about it in advance.

And a third, which is really about migrations rather than triggers. A trigger function runs with the privileges of *whoever caused it to fire*, so this one needs the writing role to be able to `INSERT` into `audit_log` - and it is easy to believe that is already true when it is not. Bonus 14.5 ran `GRANT ... ON ALL TABLES IN SCHEMA public TO zoo_app`, which reads like a blanket permission but is a snapshot: it covered the tables that existed at 00007, and `audit_log` is nine migrations younger. Apply this section on top of 14.5's role and the first animal write as `zoo_app` dies with `ERROR: permission denied for table audit_log` (`CONTEXT: PL/pgSQL function audit_row() line 3`) - the tenant policy and the application role are exactly the production shape 14.5 recommends, and the audit trigger is the thing that breaks it. Every `psql` verify in this tutorial connects as `zoo`, a superuser, so nothing here catches it; that is worth remembering on its own. The fix is the `GRANT INSERT ON audit_log TO zoo_app;` in the migration above, placed where the table is created rather than trusting an earlier grant to reach it. The alternative is to declare the function `SECURITY DEFINER`, so it executes as its owner and the app role needs no grant at all - which is the usual choice for an audit table, because it lets you *withhold* `INSERT` from the application and so keep the log append-only from the app's point of view, the trigger being the only writer. If you take that route, pin the function's `search_path` (`SECURITY DEFINER SET search_path = public, pg_temp`) or qualify every name inside it, because a definer function that resolves names through an attacker-writable schema is a well-known way to hand out the owner's privileges. The plain grant is the smaller thing to understand; the definer function is the smaller set of privileges. Either beats discovering it in production.

### 16.7 Where to go next

Stage 12.5 already lists eleven exercises for the core API. The bonus sections add a second list, in roughly the order that a real system grows them:

1. **Enclosure moves inside a zoo** (14.4) before transfers between zoos (16.4): the second is the first plus a workflow, and the capacity guard is shared.
2. **Tenancy scoping in the repository signatures** (14.5), because it is a same-day change that makes a whole class of bug unwritable, and it gets more expensive to do every week you wait.
3. **Row level security** (14.5) once the signatures are scoped, as the second line of defence rather than the first.
4. **Shifts and the roster** (15.2), because the exclusion constraint is the single highest-value line of SQL in the bonus material - it makes an entire category of scheduling bug impossible rather than merely tested.
5. **The payroll export** (15.6), not a payroll implementation. Compute hours at the right rate; hand the money to someone whose job it is.
6. **Permissions** (16.5) when a third role appears, and not before. Two roles and a `CHECK` is a perfectly good design until the day it is not.
7. **Auditing** (16.6) when someone asks a question the application logs cannot answer.

Two closing cautions, both learned the expensive way by people who did not read this far:

- **Every one of these features multiplies the tenancy surface.** A shift, a payslip, an audit row, and a permission grant all belong to somebody, and "somebody" is now a zoo as well as a person. The repository-scoping discipline in 14.5 is not a one-off; it is a rule that every new table has to obey, and the moment it is relaxed for one convenient query is the moment the system stops being multi-tenant in fact rather than in intent.
- **The database is doing more work than you think.** Exclusion constraints, `FOR UPDATE` row locks, triggers, and `REFRESH MATERIALIZED VIEW CONCURRENTLY` are all real locking and real I/O. None of them is a reason to avoid the feature - they are the reason to measure before you deploy it, and to read `EXPLAIN (ANALYZE, BUFFERS)` on anything that runs per request rather than per report.

### 16.8 Verify

The 16.1 fixture is already in place, so the rows below exist and the literals named are the ones that were inserted. First the transfers:

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo
```

```sql
-- This block writes pinned ids, so it is not re-runnable as it stands: a second
-- pass dies on `duplicate key value violates unique constraint "transfers_pkey"`
-- (23505) at the first INSERT. Clear its own rows first, the way 15.7 clears
-- `shifts` - the ids are known, so the cleanup is exact and touches nothing else.
DELETE FROM transfers WHERE id IN ('00000000-0000-4000-8000-000000000004',
                                   '00000000-0000-4000-8000-000000000005',
                                   '00000000-0000-4000-8000-000000000006');

-- one open transfer per animal
INSERT INTO transfers (id, animal_id, from_zoo_id, to_zoo_id, to_enclosure_id, requested_by)
VALUES ('00000000-0000-4000-8000-000000000004',
        '00000000-0000-4000-8000-000000000001',
        '00000000-0000-4000-8000-000000000001',
        '00000000-0000-4000-8000-000000000002',
        '00000000-0000-4000-8000-000000000003',
        '00000000-0000-4000-8000-000000000001');              -- INSERT 0 1
INSERT INTO transfers (id, animal_id, from_zoo_id, to_zoo_id, to_enclosure_id, requested_by)
VALUES ('00000000-0000-4000-8000-000000000005',
        '00000000-0000-4000-8000-000000000001',
        '00000000-0000-4000-8000-000000000001',
        '00000000-0000-4000-8000-000000000002',
        '00000000-0000-4000-8000-000000000003',
        '00000000-0000-4000-8000-000000000001');              -- ERROR 23505: one already open

-- A cancelled one does not block a new one. Note the id this row gets: it is
-- the literal written above, and it is deterministic precisely because there is
-- no sequence to advance - a `uuid ... DEFAULT gen_random_uuid()` has nothing to
-- burn, so the failed insert above consumed no value and this row simply carries
-- the id it was given. That literal is what every statement below names.
UPDATE transfers SET status='cancelled', decided_by='00000000-0000-4000-8000-000000000001', decided_at=now()
 WHERE id='00000000-0000-4000-8000-000000000004';              -- UPDATE 1
INSERT INTO transfers (id, animal_id, from_zoo_id, to_zoo_id, to_enclosure_id, requested_by)
VALUES ('00000000-0000-4000-8000-000000000006',
        '00000000-0000-4000-8000-000000000001',
        '00000000-0000-4000-8000-000000000001',
        '00000000-0000-4000-8000-000000000002',
        '00000000-0000-4000-8000-000000000003',
        '00000000-0000-4000-8000-000000000001');              -- INSERT 0 1

-- transitions are guarded, and replaying one is a no-op
UPDATE transfers SET status='completed', completed_at=now() WHERE id='00000000-0000-4000-8000-000000000006' AND status='approved';  -- UPDATE 0
UPDATE transfers SET status='approved',  decided_by='00000000-0000-4000-8000-000000000001', decided_at=now() WHERE id='00000000-0000-4000-8000-000000000006' AND status='requested';  -- UPDATE 1
UPDATE transfers SET status='completed', completed_at=now() WHERE id='00000000-0000-4000-8000-000000000006' AND status='approved';  -- UPDATE 1
UPDATE transfers SET status='completed', completed_at=now() WHERE id='00000000-0000-4000-8000-000000000006' AND status='approved';  -- UPDATE 0

-- permissions are scoped to a zoo
SELECT has_permission('00000000-0000-4000-8000-000000000001', 'animal:medicate', '00000000-0000-4000-8000-000000000002') AS vet_at_south,
       has_permission('00000000-0000-4000-8000-000000000001', 'animal:medicate', '00000000-0000-4000-8000-000000000001') AS vet_at_north;
--  t | f

-- the audit trail knows the actor, when the actor was declared. Start from an
-- empty log: 16.6's snippet already wrote a row.
DELETE FROM audit_log;

BEGIN;
SELECT set_config('app.actor_id', '00000000-0000-4000-8000-000000000001', true);
UPDATE animals SET enclosure_id = NULL WHERE id = '00000000-0000-4000-8000-000000000003';
COMMIT;

-- the same change with no actor declared: still recorded, with a NULL actor
UPDATE animals SET enclosure_id = NULL WHERE id = '00000000-0000-4000-8000-000000000003';

SELECT actor_id, table_name, row_id, action FROM audit_log ORDER BY actor_id;
--                actor_id               | table_name |                row_id                | action
-- --------------------------------------+------------+--------------------------------------+--------
--  00000000-0000-4000-8000-000000000001 | animals    | 00000000-0000-4000-8000-000000000003 | UPDATE
--                                        | animals    | 00000000-0000-4000-8000-000000000003 | UPDATE
```

The failed insert's exact complaint, worth reading rather than paraphrasing, names the index and the animal that already has a transfer in flight:

```
ERROR:  duplicate key value violates unique constraint "transfers_one_open_per_animal"
DETAIL:  Key (animal_id)=(00000000-0000-4000-8000-000000000001) already exists.
```

If you build only one thing from all three bonus sections, build the one that makes a class of bug impossible rather than the one that makes it detectable. `EXCLUDE USING gist` for shifts, the partial unique index for open transfers, and `FOR UPDATE` for capacity are all the same idea wearing three hats: **put the invariant where it cannot be bypassed.**

---

[Bonus 15](15-bonus-scheduling-payroll.md)  |  [Overview](../tutorial.md)
