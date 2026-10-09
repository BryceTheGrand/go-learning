## Bonus 15: Moving animals between zoos, and where to go next

The last bonus section, and the one that ties the others together: once there is more than one zoo (Bonus 13), a roster (Bonus 14), and capacity that matters, you inevitably need to *move an animal from one to another* - and that turns out to be the first thing in this tutorial that is a **workflow** rather than a request.

It closes with the three things that a system at this point always grows next (permissions that outgrow a `role` column, an audit trail, and a sense of when to stop), and then hands you back to Stage 12's exercise list.

As before, every query was executed against Postgres 17, including the ones that fail.

### 15.1 A transfer is not an UPDATE

The tempting implementation is one statement:

```sql
-- do not do this. It is one statement, and it works.
UPDATE animals SET zoo_id = 2, enclosure_id = 3 WHERE id = 1;

-- put it back, so the rest of this section starts from where 13 left off
UPDATE animals SET zoo_id = 1, enclosure_id = 1 WHERE id = 1;
```

Everything wrong with it is invisible in the SQL - including the fact that it succeeds - which is why it is worth enumerating:

- **Nobody approved it.** Moving an animal between sites is a regulated act (transport, health certificates, welfare checks). It needs a request, a decision, and a record of who made it.
- **The destination's capacity was never checked.** Bonus 13.4 built the guard; a bare `UPDATE` walks straight past it.
- **There is no history.** Six months later, "when did Tembo move, and who authorised it?" has no answer, because the row that would have held the answer was overwritten.
- **It cannot be half-done.** A real transfer has states - requested, approved, in transit, completed - and each transition is a different person's job.

So the thing being modelled is not the animal's location. It is the *request*, and the animal's location is a side effect of the request reaching its final state.

### 15.2 The schema

`migrations/00009_animal_transfers.sql`:

```sql
-- +goose Up
CREATE TABLE transfers (
    id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    animal_id       bigint NOT NULL REFERENCES animals(id) ON DELETE CASCADE,
    from_zoo_id     bigint NOT NULL REFERENCES zoos(id),
    to_zoo_id       bigint NOT NULL REFERENCES zoos(id),
    to_enclosure_id bigint REFERENCES enclosures(id),
    status          text NOT NULL DEFAULT 'requested'
                    CHECK (status IN ('requested', 'approved', 'completed', 'cancelled')),
    requested_by    bigint NOT NULL REFERENCES zookeepers(id),
    requested_at    timestamptz NOT NULL DEFAULT now(),
    decided_by      bigint REFERENCES zookeepers(id),
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

- **`status` is text with a `CHECK`, not an enum type.** Either is defensible. A `CHECK` is a one-line change to extend; a Postgres `enum` is a type whose values are ordered and which cannot be dropped from without a migration. For a state machine that will grow a state or two, the `CHECK` is the smaller commitment. (Bonus 13.2 makes the opposite call for `environments` and explains why: that one is *data*, this one is *structure*.)
- **The partial unique index is how you say "at most one open X per Y".** `WHERE status IN ('requested','approved')` means the index only contains the rows that are still in flight, so it refuses a second *open* transfer while leaving the closed ones alone. Verified both ways: a second request for the same animal fails with `duplicate key value violates unique constraint "transfers_one_open_per_animal"` (`23505`), and once the first is cancelled, a new request for the same animal inserts cleanly. This single index replaces a "check for an existing transfer" method that would race with itself.
- **`CONSTRAINT transfers_different_zoos CHECK (from_zoo_id <> to_zoo_id)`** is a rule that costs nothing to state and is embarrassing to get wrong. A "transfer" from a zoo to itself is a move between enclosures, which is Bonus 13.4's job.

### 15.3 Transitions as guarded UPDATEs

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

and then checks `tag.RowsAffected()` - Stage 3.2's `Exec` plus row count, doing real work for the first time. Verified behaviour:

```
-- jumping straight from requested to completed
UPDATE transfers SET status='completed' WHERE id=3 AND status='approved';   -- UPDATE 0

-- the legal transition
UPDATE transfers SET status='approved'  WHERE id=3 AND status='requested';  -- UPDATE 1
UPDATE transfers SET status='completed' WHERE id=3 AND status='approved';   -- UPDATE 1

-- replaying it
UPDATE transfers SET status='completed' WHERE id=3 AND status='approved';   -- UPDATE 0
```

**`UPDATE 0` is not an error; it is the answer.** Zero rows affected means "the precondition did not hold", and the caller maps that to `409 Conflict` ("this transfer has already been decided") exactly as Stage 9 maps its other sentinels. Two properties fall out for free, and both matter at scale:

- **Concurrency-safety without a lock.** Two simultaneous completions cannot both succeed, because the second one's `WHERE` no longer matches.
- **Retry-safety.** A client that times out and retries the completion gets `UPDATE 0`, which the API can translate into "already done" rather than "conflict". Deciding which of those two it means is a policy choice - but note that you *can* decide, because the row is still there and still says `completed`, which is the whole reason for keeping history rather than deleting it.

### 15.4 Completing a transfer: one transaction, re-checked from scratch

Approval and completion are separated by hours or weeks, and the world moves in between: the animal may have been moved by hand, the destination enclosure may have filled up. So completion re-validates everything the approval assumed, in one transaction:

```go
// CompleteTransfer moves the animal and closes the request atomically. The
// approval decision is not trusted as a statement of current fact: the
// animal is locked, its zoo re-read, and the destination's capacity
// re-checked, because any of those can have changed since it was approved.
func (r *dbRepository) CompleteTransfer(ctx context.Context, transferID int64) error {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx)

	var animalID, fromZooID, toZooID int64
	var toEnclosureID *int64
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
	var currentZooID int64
	if err := tx.QueryRow(ctx,
		`SELECT zoo_id FROM animals WHERE id = $1 FOR UPDATE`, animalID).
		Scan(&currentZooID); err != nil {
		return err
	}
	if currentZooID != fromZooID {
		return ErrAnimalMoved
	}

	if toEnclosureID != nil {
		if err := r.reserveSlot(ctx, tx, *toEnclosureID, toZooID); err != nil {
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

The `reserveSlot` call inside is Bonus 13.4's capacity guard lifted out into a method of its own, so that moving an animal *within* a zoo and moving it *between* zoos share one implementation:

```go
// reserveSlot is the capacity and environment guard from 13.4, taking the
// enclosure first and the zoo second - the same order PlaceAnimal uses, and
// deliberately the same order in every caller, because a pair of arguments
// that swaps meaning between call sites is how the zoo id ends up being used
// as the enclosure id.
func (r *dbRepository) reserveSlot(ctx context.Context, tx pgx.Tx, enclosureID, zooID int64) error {
	// SELECT environment, capacity FROM enclosures
	//  WHERE id = $1 AND zoo_id = $2 FOR UPDATE
	// then count the other occupants (id <> $2), then check the species'
	// environment against the enclosure's, returning ErrEnclosureNotFound,
	// ErrWrongEnvironment or ErrEnclosureFull. The body is 13.4's guard
	// moved, not rewritten; only the signature is new.
}
```

Four decisions in that function are the ones to carry away:

- **Re-read, do not trust.** The transfer row said the animal was at `from_zoo_id` when it was approved. This code checks that it still is, and returns `ErrAnimalMoved` if not, rather than blindly applying a stale decision.
- **Lock ordering.** The animal is locked before the enclosure, and 13.4's `PlaceAnimal` locks the same two rows in the same order - which is the point, since a transfer and a placement can be in flight at once. Two transactions that grabbed those rows in opposite orders would deadlock; Postgres detects that and kills one, which is safe but ugly and hard to reproduce on demand. A stated order is the cheap fix, and it only works if it is written down in both places, which is exactly what the two functions now do - and exactly the sort of thing that drifts, because nothing in the language enforces a comment.
- **One implementation of the capacity rule.** `reserveSlot` is extracted the moment it has a second caller. Duplicating it would not be a DRY violation so much as a correctness one: two copies of a capacity check drift, and the one that drifts is the one nobody is testing.
- **The animal's `feed_log` history does not move with it**, and should not: `feed_log.keeper_id` references the keeper who fed it, whoever they were and wherever they worked. The animal's past is a fact about the animal, not about the zoo that currently houses it. This is the kind of question ("do we partition history by tenant?") that deserves an explicit answer, and "no, history stays with the animal" is a defensible one.

### 15.5 When a role column stops being enough

`zookeepers.role` has been a `text` column with two legal values since Stage 2, and by now it is straining:

- Adding a `vet` role means changing a `CHECK`, and it cannot express "a vet at one zoo but not another".
- `RequireRole("admin")` is a string comparison in middleware, so the set of things an admin can do is spread across the route table rather than declared in one place.
- There is no way to grant one person one extra power - "Sam may run payroll but not edit animals" - without inventing a role for the combination.

The standard answer is four tables and a function (`migrations/00010_roles_and_permissions.sql`):

```sql
CREATE TABLE roles          (name text PRIMARY KEY, description text NOT NULL);
CREATE TABLE permissions    (name text PRIMARY KEY, description text NOT NULL);
CREATE TABLE role_permissions (
    role       text NOT NULL REFERENCES roles(name) ON DELETE CASCADE,
    permission text NOT NULL REFERENCES permissions(name) ON DELETE CASCADE,
    PRIMARY KEY (role, permission)
);

CREATE TABLE zookeeper_roles (
    zookeeper_id bigint NOT NULL REFERENCES zookeepers(id) ON DELETE CASCADE,
    role         text NOT NULL REFERENCES roles(name),
    zoo_id       bigint REFERENCES zoos(id) ON DELETE CASCADE,   -- NULL = every zoo
    UNIQUE NULLS NOT DISTINCT (zookeeper_id, role, zoo_id)
);

CREATE FUNCTION has_permission(zk bigint, perm text, zoo bigint)
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
    (1, 'vet', 2), (1, 'keeper', NULL);
```

- **`NULLS NOT DISTINCT`** (Postgres 15 and later) is the quiet hero. A `UNIQUE` constraint normally treats two `NULL`s as different, so `(1, 'keeper', NULL)` could be inserted twice and mean "keeper everywhere" twice. `NULLS NOT DISTINCT` makes the nulls compare equal, so the natural key behaves the way you expect. Verified: maya as `vet` scoped to South Zoo, and as `keeper` unscoped, resolves correctly across all four combinations - vet at South (`t`), vet at North (`f`), keeper anywhere (`t`), payroll admin (`f`).
- **`zoo_id IS NULL` means "every zoo"**, which is how the existing global roles keep working during the migration. The `OR` in the function is a little blunt, but it is a documented and testable convention rather than an accident.
- **The middleware stops comparing strings.** `RequireRole("admin")` becomes `RequirePermission("payroll:run")`, and the route table starts reading as the policy document Stage 8 wanted it to be. The `role` claim in the JWT becomes a hint at best: **permissions change far more often than roles did**, which sharpens Stage 8's staleness warning considerably. A revoked permission inside a 24-hour token is a 24-hour window; this is the strongest argument in the tutorial for short token lifetimes plus refresh, and it is where you should spend the effort the deleted-zookeeper problem originally asked for.

### 15.6 An audit trail, and who the database thinks you are

Regulated systems need "who changed this row, and when", and Postgres will write it for you (`migrations/00011_audit_log.sql`):

```sql
CREATE TABLE audit_log (
    id       bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    at       timestamptz NOT NULL DEFAULT now(),
    actor_id bigint,
    table_name text NOT NULL,
    row_id   bigint,
    action   text NOT NULL
);

CREATE FUNCTION audit_row() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO audit_log (actor_id, table_name, row_id, action)
    VALUES (nullif(current_setting('app.actor_id', true), '')::bigint,
            TG_TABLE_NAME, COALESCE(NEW.id, OLD.id), TG_OP);
    RETURN COALESCE(NEW, OLD);
END $$;

CREATE TRIGGER animals_audit
    AFTER INSERT OR UPDATE OR DELETE ON animals
    FOR EACH ROW EXECUTE FUNCTION audit_row();
```

The subtle part is `actor_id`. The database has no idea who your HTTP caller was unless you tell it, and the mechanism is the same session-local setting Bonus 13.5 used for the tenant:

```sql
BEGIN;
SELECT set_config('app.actor_id', '1', true);   -- transaction-local
UPDATE animals SET enclosure_id = NULL WHERE id = 3;
COMMIT;
```

Verified: the update above writes an audit row with `actor_id = 1`, and an update *outside* any declared actor still writes the row with `actor_id` NULL. That asymmetry is deliberate and worth keeping - **audit fails open on attribution and closed on omission.** You always learn that something changed; you sometimes learn who, and a NULL actor in the log is a signal that some code path forgot to declare one, which is exactly the signal you want.

Two honest caveats. First, trigger-based auditing writes one row per statement effect, so a bulk `UPDATE` touching ten thousand animals writes ten thousand audit rows inside one transaction - fine here, a throughput problem in a bigger system, where you move to logical decoding or an outbox table. Second, if a `DELETE` cascades (Stage 6's `feed_log` cascade) the trigger on the *child* table does not fire unless you install it there too, so "why did these feed rows vanish?" is answerable only if you thought about it in advance.

### 15.7 Where to go next

Stage 12.5 already lists eleven exercises for the core API. The bonus sections add a second list, in roughly the order that a real system grows them:

1. **Enclosure moves inside a zoo** (13.4) before transfers between zoos (15.4): the second is the first plus a workflow, and the capacity guard is shared.
2. **Tenancy scoping in the repository signatures** (13.5), because it is a same-day change that makes a whole class of bug unwritable, and it gets more expensive to do every week you wait.
3. **Row level security** (13.5) once the signatures are scoped, as the second line of defence rather than the first.
4. **Shifts and the roster** (14.2), because the exclusion constraint is the single highest-value line of SQL in the bonus material - it makes an entire category of scheduling bug impossible rather than merely tested.
5. **The payroll export** (14.6), not a payroll implementation. Compute hours at the right rate; hand the money to someone whose job it is.
6. **Permissions** (15.5) when a third role appears, and not before. Two roles and a `CHECK` is a perfectly good design until the day it is not.
7. **Auditing** (15.6) when someone asks a question the application logs cannot answer.

Two closing cautions, both learned the expensive way by people who did not read this far:

- **Every one of these features multiplies the tenancy surface.** A shift, a payslip, an audit row, and a permission grant all belong to somebody, and "somebody" is now a zoo as well as a person. The repository-scoping discipline in 13.5 is not a one-off; it is a rule that every new table has to obey, and the moment it is relaxed for one convenient query is the moment the system stops being multi-tenant in fact rather than in intent.
- **The database is doing more work than you think.** Exclusion constraints, `FOR UPDATE` row locks, triggers, and `REFRESH MATERIALIZED VIEW CONCURRENTLY` are all real locking and real I/O. None of them is a reason to avoid the feature - they are the reason to measure before you deploy it, and to read `EXPLAIN (ANALYZE, BUFFERS)` on anything that runs per request rather than per report.

### 15.8 Verify

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo
```

```sql
-- one open transfer per animal
INSERT INTO transfers (animal_id, from_zoo_id, to_zoo_id, to_enclosure_id, requested_by)
VALUES (1, 1, 2, 3, 1);
INSERT INTO transfers (animal_id, from_zoo_id, to_zoo_id, to_enclosure_id, requested_by)
VALUES (1, 1, 2, 3, 1);                    -- ERROR 23505: one already open

-- A cancelled one does not block a new one. Note the id this row gets: the
-- failed insert above consumed identity value 2 even though it wrote no row
-- (Stage 3.2 warned about exactly this), so it is 3, and it is 3 that every
-- statement below has to name.
UPDATE transfers SET status='cancelled', decided_by=1, decided_at=now() WHERE id=1;
INSERT INTO transfers (animal_id, from_zoo_id, to_zoo_id, to_enclosure_id, requested_by)
VALUES (1, 1, 2, 3, 1);                    -- INSERT 0 1, id 3

-- transitions are guarded, and replaying one is a no-op
UPDATE transfers SET status='completed', completed_at=now() WHERE id=3 AND status='approved';  -- UPDATE 0
UPDATE transfers SET status='approved',  decided_by=1, decided_at=now() WHERE id=3 AND status='requested';  -- UPDATE 1
UPDATE transfers SET status='completed', completed_at=now() WHERE id=3 AND status='approved';  -- UPDATE 1
UPDATE transfers SET status='completed', completed_at=now() WHERE id=3 AND status='approved';  -- UPDATE 0

-- permissions are scoped to a zoo
SELECT has_permission(1, 'animal:medicate', 2) AS vet_at_south,
       has_permission(1, 'animal:medicate', 1) AS vet_at_north;

-- the audit trail knows the actor, when the actor was declared
BEGIN;
SELECT set_config('app.actor_id', '1', true);
UPDATE animals SET enclosure_id = NULL WHERE id = 3;
COMMIT;
SELECT id, actor_id, table_name, row_id, action FROM audit_log ORDER BY id;
```

If you build only one thing from all three bonus sections, build the one that makes a class of bug impossible rather than the one that makes it detectable. `EXCLUDE USING gist` for shifts, the partial unique index for open transfers, and `FOR UPDATE` for capacity are all the same idea wearing three hats: **put the invariant where it cannot be bypassed.**

---

[Bonus 14](14-bonus-scheduling-payroll.md)  |  [Overview](../tutorial.md)
