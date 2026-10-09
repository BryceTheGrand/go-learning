## Bonus 15: Shifts across zoos, and paying for them

Bonus material again: not part of the linear path, and nothing the core tutorial relies on. It picks up from Bonus 14 (multiple zoos exist and animals belong to one), and it is the part of an ERP that people find hardest to model well - *time*, and *money*. Bonus 16 does lean on one thing here: the two keeper rows 15.2 pins, which its permission seed references. Everything else in this section is optional in the strict sense.

The reason both are hard is the same. Both are continuous quantities that the business talks about in discrete, human terms ("Wednesday mornings", "twenty pounds an hour"), and most of the damage in this area comes from storing the human description instead of the underlying fact.

Every query below was run against Postgres 17, including the ones that are supposed to fail.

### 15.1 Why a shift is a range, not a pair of columns

The obvious schema for "keepers work certain days at certain times" is:

```sql
-- the shape to avoid. Named shifts_naive rather than shifts so that running it
-- does not leave a table for 15.2's real migration to collide with.
CREATE TABLE shifts_naive (
    zookeeper_id uuid NOT NULL,
    zoo_id       uuid NOT NULL,
    shift_date   date   NOT NULL,
    starts_at    time   NOT NULL,
    ends_at      time   NOT NULL
);
```

It stores the *description* of a shift, and it makes the two questions you will actually ask expensive and error-prone:

- "Is Sam double-booked?" becomes a comparison of `date` plus overlapping `time` ranges, in application code, for every pair of shifts. The database cannot help, because "these two things overlap" is not something a `CHECK` constraint or a unique index can express over four separate columns.
- "Who is on duty at 3pm?" becomes `WHERE shift_date = $1 AND starts_at <= $2 AND ends_at > $2`, which is correct only if you remembered that the comparison is half-open in one place and closed in another. Night shifts crossing midnight make `shift_date` a lie.

Postgres has a type for the underlying fact - **an interval in time** - and a whole family of operators over it:

```sql
SELECT tstzrange('2026-10-12 08:00+01', '2026-10-12 16:00+01');   -- ["2026-10-12 07:00+00","2026-10-12 15:00+00")
SELECT tstzrange('2026-10-12 08:00+01', '2026-10-12 16:00+01') @> '2026-10-12 12:00+01'::timestamptz;  -- t: contains that instant
SELECT tstzrange('2026-10-12 08:00+01', '2026-10-12 16:00+01')
    && tstzrange('2026-10-12 15:00+01', '2026-10-12 20:00+01');                          -- t: they overlap
SELECT tstzrange('2026-10-12 08:00+01', '2026-10-12 16:00+01')
    && tstzrange('2026-10-12 16:00+01', '2026-10-12 20:00+01');                          -- f: touching is not overlapping
```

The `::timestamptz` on the second line is not decoration. Without it, that untyped literal has nothing to tell Postgres what type it should be, so it is read as a *range* and the query dies with `ERROR: malformed range literal: "2026-10-12 12:00+01"`. `@>` will happily compare a range to a range; making it compare to an instant is your job, spelled out.

Three operators to remember: `@>` is *contains*, `&&` is *overlaps*, and `*` is *intersection* (Bonus 15.6 uses that one to clamp a shift to a pay period). Ranges are **half-open by default** - `[start, end)`, lower bound included, upper bound excluded - which is exactly why the third line above is `f`: a shift ending at 16:00 and the next starting at 16:00 do not overlap, so back-to-back shifts are legal without anyone writing a special case.

### 15.2 The constraint that makes double-booking impossible

`migrations/00009_staff_scheduling.sql`:

```sql
-- +goose Up
-- btree_gist lets a GiST index mix an ordinary equality column (uuid here)
-- with a range column. Without it, the exclusion constraint below cannot be
-- created at all.
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE shifts (
    id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    zookeeper_id uuid NOT NULL REFERENCES zookeepers(id) ON DELETE CASCADE,
    zoo_id       uuid NOT NULL REFERENCES zoos(id) ON DELETE CASCADE,
    during       tstzrange NOT NULL,
    CONSTRAINT shifts_during_not_empty CHECK (NOT isempty(during))
);

-- The one line this whole section exists for.
ALTER TABLE shifts
    ADD CONSTRAINT shifts_no_overlap
    EXCLUDE USING gist (zookeeper_id WITH =, during WITH &&);

-- +goose Down
DROP TABLE shifts;
```

An **exclusion constraint** says "no two rows may satisfy this predicate". Read it as: for any two rows, if `zookeeper_id` is equal *and* `during` overlaps, the insert is refused. The `WITH =` and `WITH &&` are the operators to test, per column, and `USING gist` is the index type that can answer "is there any row that overlaps this?" - which is why `btree_gist` is needed to mix an equality test into a GiST index. The equality column is a `uuid` here rather than a `bigint`, and it makes no difference: `btree_gist` has carried a `uuid` operator class since long before Postgres 17, so `zookeeper_id WITH =` on a UUID works exactly as it did on an integer.

Before any shifts, pin the two keepers the examples below name. Stage 13 gave every zookeeper a random id, so there is nothing to paste forward from a result here - the section needs fixed ids the way Bonus 14 needed fixed zoo ids, and it gets them the same way:

```sql
-- The two keepers this section works with. On a database that has run Bonus 14,
-- feed_log is empty (14.3 rebuilt the animals), so no keeper is referenced and
-- these rows can be re-pinned freely: maya was deleted back in Stage 6, and sam
-- exists from Stage 8 under a generated id.
--
-- maya's hash is a placeholder and she does not log in again in this tutorial.
-- sam's is a real bcrypt hash of his Stage 7 password ('elephant-road'), so
-- re-pinning his id does not quietly break that login. A bcrypt hash is
-- self-contained - it carries its own salt and the cost factor in the prefix -
-- so pasting a fixed one verifies exactly like a freshly generated one.
DELETE FROM zookeepers WHERE username IN ('maya', 'sam');
INSERT INTO zookeepers (id, username, password_hash, role) VALUES
    ('00000000-0000-4000-8000-000000000001', 'maya', 'x', 'keeper'),
    ('00000000-0000-4000-8000-000000000002', 'sam',
     '$2a$10$n3nDW/JSOi.MBDLous6Lte0uGXDPsBuqWZvQmZODEKWJSoEG9xwV.', 'keeper');
```

> **Read the first row as a deliberate shortcut, not the pattern.** Writing `'x'` into `password_hash` produces a row that can never log in, which is fine for a fixture whose only job is to have an id to point a `zookeeper_id` at - and wrong anywhere else. It is also the exact opposite of Stage 4's rule, so it is worth saying why it is allowed here: this row is never handed a password and never authenticates, and the alternative (a real hash for a character who was deleted six stages ago) would be noise. sam's row gets the real hash for the opposite reason - Stage 7 documents his credentials as usable, and a fixture that silently invalidated them would be a trap for anyone who tries them later.

What that buys, verified rather than assumed:

```
-- maya, North Zoo, Mon 12 Oct 08:00-16:00
INSERT INTO shifts (zookeeper_id, zoo_id, during) VALUES
    ('00000000-0000-4000-8000-000000000001', '00000000-0000-4000-8000-000000000001', tstzrange('2026-10-12 08:00+01', '2026-10-12 16:00+01'));

-- maya again, at the OTHER zoo, 15:00-20:00: refused
INSERT INTO shifts (zookeeper_id, zoo_id, during) VALUES
    ('00000000-0000-4000-8000-000000000001', '00000000-0000-4000-8000-000000000002', tstzrange('2026-10-12 15:00+01', '2026-10-12 20:00+01'));
ERROR:  conflicting key value violates exclusion constraint "shifts_no_overlap"
DETAIL:  Key (zookeeper_id, during)=(00000000-0000-4000-8000-000000000001, ["2026-10-12 14:00:00+00","2026-10-12 19:00:00+00")) conflicts
         with existing key (zookeeper_id, during)=(00000000-0000-4000-8000-000000000001, ["2026-10-12 07:00:00+00","2026-10-12 15:00:00+00")).

-- maya, back-to-back 16:00-20:00: accepted
INSERT INTO shifts (zookeeper_id, zoo_id, during) VALUES
    ('00000000-0000-4000-8000-000000000001', '00000000-0000-4000-8000-000000000002', tstzrange('2026-10-12 16:00+01', '2026-10-12 20:00+01'));
INSERT 0 1
```

Notice what the refused insert tells you: the constraint does not care that the second shift is at a *different zoo*. Physical impossibility is the rule - one person cannot be in two places - and the constraint states it once, for all zoos, permanently. That is precisely the kind of invariant that should live in the database rather than in a "check for overlaps" service method that someone will forget to call from the import script.

### 15.3 From patterns to shifts

Nobody wants to insert three years of Wednesdays by hand, so the pattern is stored once and *materialised* into concrete shifts.

`migrations/00010_shift_templates.sql`:

```sql
-- +goose Up
CREATE TABLE shift_templates (
    id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    zookeeper_id uuid NOT NULL REFERENCES zookeepers(id) ON DELETE CASCADE,
    zoo_id       uuid NOT NULL REFERENCES zoos(id) ON DELETE CASCADE,
    iso_dow      int NOT NULL CHECK (iso_dow BETWEEN 1 AND 7),   -- 1 = Monday
    starts_at    time NOT NULL,
    ends_at      time NOT NULL,
    CONSTRAINT shift_templates_ordered CHECK (ends_at > starts_at)
);

-- +goose Down
DROP TABLE shift_templates;
```

and the generator, which turns "every Wednesday, 08:00 to 16:00" into rows:

```sql
INSERT INTO shifts (zookeeper_id, zoo_id, during)
SELECT t.zookeeper_id, t.zoo_id,
       tstzrange((d::date + t.starts_at) AT TIME ZONE 'Europe/London',
                 (d::date + t.ends_at)   AT TIME ZONE 'Europe/London')
FROM shift_templates t
CROSS JOIN generate_series($1::date, $2::date, interval '1 day') AS d
WHERE EXTRACT(isodow FROM d) = t.iso_dow
ON CONFLICT DO NOTHING;
```

Three things are worth noticing, and the first one is the reason the whole range approach pays off.

- **`AT TIME ZONE` is where the timezone decision gets made, once.** A shift template says "08:00 local", and the instant that corresponds to changes twice a year. Materialising three weeks either side of a clock change produces this, which is not a bug:

  ```
   zookeeper_id                         | zoo_id                               | during
  --------------------------------------+--------------------------------------+-----------------------------------------------------
   00000000-0000-4000-8000-000000000002 | 00000000-0000-4000-8000-000000000001 | ["2026-10-21 07:00:00+00","2026-10-21 15:00:00+00")
   00000000-0000-4000-8000-000000000002 | 00000000-0000-4000-8000-000000000001 | ["2026-10-28 08:00:00+00","2026-10-28 16:00:00+00")
   00000000-0000-4000-8000-000000000002 | 00000000-0000-4000-8000-000000000001 | ["2026-11-04 08:00:00+00","2026-11-04 16:00:00+00")
  ```

  British Summer Time ends on 25 October 2026, so the 21st is 07:00 UTC and the 28th is 08:00 UTC - both are 08:00 *to the person standing in the zoo*, which is the thing the contract and the payroll both mean. Had the column been `timestamp without time zone`, all three rows would read 08:00 and the third would silently be an hour wrong for anyone who looked at it from another zone. **Store instants; do the human interpretation at the edges.** This is Stage 2's `timestamptz` advice collecting its dividend three stages later.
- **The generator is idempotent, and that trailing `ON CONFLICT DO NOTHING` is the entire reason.** Drop it and the second run raises `ERROR: conflicting key value violates exclusion constraint "shifts_no_overlap"` and, being a single statement, aborts as a whole rather than inserting the rows that did not conflict. Keep it and the second and third runs report `INSERT 0 0` with the roster unchanged. Three things about that clause are worth knowing, because none of them is guessable. It works on an *exclusion* constraint even though you cannot name one: `ON CONFLICT (zookeeper_id) DO NOTHING` fails with `there is no unique or exclusion constraint matching the ON CONFLICT specification`, while the targetless form quietly swallows the violation. That bluntness is the second thing: it will swallow a conflict for *any* reason, including a duplicate you did not anticipate, which is fine for a generator you re-run and wrong for a user-facing insert. And the third is the alternative you are trading away - catching SQLSTATE `23P01` in Go and treating it as "already generated" is more code, but it tells you which rows were already there, which is what you need the day you have to reconcile a roster rather than just fill one.
- **Recurrence rules get deep, so keep them shallow or buy them.** "Every second Tuesday except bank holidays, and never on the same day as the other North Zoo keeper" is a rule, not a table. The moment a template needs exceptions, exclusions, or month-relative dates, you are re-implementing RFC 5545 (`RRULE`), and the mature answers are a recurrence library or a third-party calendar rather than a cleverer `generate_series`. What this table is good for is what most rosters actually need: a regular weekly pattern, materialised ahead, with individual shifts edited afterwards.

### 15.4 A rule that spans both domains: you may only feed while on shift

Now the scheduling data can enforce a rule the earlier stages could not express: **a keeper may only record a feeding for an animal while on shift at that animal's zoo.** This is the first rule in the tutorial that joins all three domains - the token says who you are, the roster says whether you are working, and the animal says where you are working.

The whole check is one `EXISTS`, and it belongs inside the same transaction as the insert (Stage 7.2's `Feed`):

```sql
SELECT EXISTS (
    SELECT 1
    FROM shifts s
    JOIN animals a ON a.zoo_id = s.zoo_id
    WHERE s.zookeeper_id = $1
      AND a.id = $2
      AND s.during @> now()
);
```

The data that makes each row meaningful is not in the tutorial yet: the shifts the earlier sections insert are pinned to dates in October 2026 rather than to `now()`, so none of them covers the moment you run this, and none of the tutorial's animals is at South Zoo. So a shift *and* an animal are both needed, and both are temporary:

```sql
-- One shift each, covering now(), at the zoo each keeper belongs to.
INSERT INTO shifts (zookeeper_id, zoo_id, during) VALUES
    ('00000000-0000-4000-8000-000000000001', '00000000-0000-4000-8000-000000000002',
     tstzrange(now() - interval '1 hour', now() + interval '1 hour')),
    ('00000000-0000-4000-8000-000000000002', '00000000-0000-4000-8000-000000000001',
     tstzrange(now() - interval '1 hour', now() + interval '1 hour'));

-- And a South Zoo animal to ask about. Disposable, and deliberately not one of
-- ...0001-...0003: this rule only reads a zoo, so borrowing Tembo would mean
-- moving him and moving him back. `enclosure` is still NOT NULL until Bonus
-- 14.3's contract runs, so it has to be written even though nothing reads it.
INSERT INTO animals (id, name, species, enclosure, zoo_id) VALUES
    ('00000000-0000-4000-8000-0000000000ff', 'Temp', 'Red panda', 'Forest Yard',
     '00000000-0000-4000-8000-000000000002');
```

With those in place, all four combinations behave as stated (the animal id is the one you just inserted, `...00ff`; the North Zoo animal is Tembo, `...0001`):

| Caller | Animal's zoo | Shift covering `now()` | Result |
|---|---|---|---|
| maya (00000000-0000-4000-8000-000000000001) | South Zoo (`...00ff`) | yes, South Zoo | `t` |
| sam (00000000-0000-4000-8000-000000000002) | South Zoo (`...00ff`) | none | `f` |
| sam (00000000-0000-4000-8000-000000000002) | North Zoo (Tembo, `...0001`) | yes, North Zoo | `t` |
| anyone | any | shift at a *different* zoo | `f` |

That last row is the reason the join is on `a.zoo_id = s.zoo_id` rather than a plain `s.zoo_id = $zooId`: a keeper on shift at North Zoo feeding an animal at South Zoo is exactly the case a naive check misses.

**Delete the temporary animal when you are done, and the two shifts with it.** The animal is the one that matters: `...00ff` is a row Bonus 16's recap does not list, it belongs to South Zoo, and every later count (an RLS check re-run, an occupancy query) will include it. The shifts are only untidy - 15.7 opens with `DELETE FROM shifts` - but this rule reads "on shift *now*", so a leftover shift covering `now()` changes what that section's `WHERE s.during @> now()` returns:

```sql
DELETE FROM shifts  WHERE zookeeper_id IN ('00000000-0000-4000-8000-000000000001',
                                           '00000000-0000-4000-8000-000000000002')
                     AND during @> now();
DELETE FROM animals WHERE id = '00000000-0000-4000-8000-0000000000ff';
```

The complementary query - the operations screen - is just as short, and needs no `date` arithmetic at all:

```sql
SELECT s.zookeeper_id, z.name AS zoo, lower(s.during) AS since, upper(s.during) AS until
FROM shifts s
JOIN zoos z ON z.id = s.zoo_id
WHERE s.during @> now()
ORDER BY s.zookeeper_id;
```

> **Where this rule should live, honestly.** The shift check inside `Feed` couples the animals domain to the scheduling domain - the same trade Stage 7.2 discussed when `AssignKeeper` reached across into `zookeepers`. The alternative is an `OnShift(ctx, keeperID, animalID, at) (bool, error)` method on a scheduling service, injected into the animals service - note the second parameter is the *animal*, matching the `EXISTS` above: the join resolves the animal's zoo, so the caller hands over the id it already has and never looks the zoo up itself. That method is cleaner in the diagram, and it adds a dependency and an interface to maintain. This bonus ships no Go for it - there is no `internal/scheduling/` package in the tutorial, and the deliverable here is the `EXISTS` predicate itself, which you can verify in `psql` exactly as shown. If you do build the service, it belongs in `internal/scheduling/`, one domain package with its own repository and service, mirroring how `internal/zookeepers/` owns its table. Both are defensible; what is not defensible is enforcing it in the *handler*, because the import script and the future mobile client will not go through the handler.
>
> One binding trap if you do take that shape. The `EXISTS` above writes `s.during @> now()`, but a parameterised version must write `s.during @> $3::timestamptz` - bind the *element* and cast it, never bind the range. Let the cast go and Postgres resolves the bare `$3` against `@>` to the **range** overload, so it infers the parameter's type as `tstzrange` (confirm it yourself with `PREPARE` and `pg_prepared_statements`), and pgx then refuses a `time.Time` on the way out: `failed to encode args[2]: unable to encode time.Date(...) into binary format for tstzrange (OID 3910): cannot find encode plan` (the index is `2` because `at` is the third argument here, after the keeper and the animal). It fails client-side, before the query reaches the server, which is why the message names an OID rather than an operator - the same family of surprise as Stage 13's stale `int64` against a `uuid` column. 15.1 made the point for literals; parameters need it too.

> **Read the table above as the rule, not as a switch to flip today.** Every keeper the core tutorial creates - sam, admin, newbie - has no `shifts` row, so the moment you wire this `EXISTS` into `Feed` and restart, **every feeding the API has been accepting since Stage 7 starts being refused**. That is the rule working exactly as written and still a nasty surprise if you did not expect it. So this tutorial stops at the predicate: build it, verify it against the table below, and wire it into `Feed` only once there is a roster worth enforcing against. Concretely, that means the standalone `OnShift` method is the path to take here, and `Feed` is left exactly as Stage 7 wrote it - the `EXISTS` is real, tested code that no route calls yet, the same shape as the `PlaceAnimal` method in 14.4 and the transfer functions in 16.4. If you would rather see the gate actually fire end-to-end, insert a shift covering `now()` for the keeper your token belongs to first (the 15.7 block below does exactly that) and wire the check in behind it; just know you are turning on a gate over an empty roster, and that the tutorial's own keepers predate the roster - a real system would never have that gap, because the shifts would have existed before the gate.

### 15.5 Money: rates that change, stored as integers

Two rules before any schema, both of which have cost somebody a production incident:

- **Never `float` or `double` a monetary amount.** Binary floating point cannot represent 0.10 exactly, and `sum` over ten thousand payslips accumulates the error into a figure that does not reconcile. Store **integer minor units** - pence, cents - in a `bigint`, and format at the display edge.
- **A rate is not a property of the person; it is a property of the person *at a time*.** A raise does not overwrite the old rate, because the shifts worked in March must still be paid at March's rate. That is a **time-versioned** (or "temporal") table, and it looks like this (`migrations/00011_pay_rates.sql`):

```sql
-- +goose Up
CREATE TABLE pay_rates (
    zookeeper_id   uuid NOT NULL REFERENCES zookeepers(id) ON DELETE CASCADE,
    hourly_cents   int NOT NULL CHECK (hourly_cents > 0),
    currency       text NOT NULL DEFAULT 'GBP',
    effective_from timestamptz NOT NULL,
    PRIMARY KEY (zookeeper_id, effective_from)
);

-- +goose Down
DROP TABLE pay_rates;
```

The primary key does the work: one rate per keeper per instant, and the *current* rate is the newest row that has already started:

```sql
SELECT hourly_cents FROM pay_rates
WHERE zookeeper_id = $1 AND effective_from <= now()
ORDER BY effective_from DESC
LIMIT 1;
```

Adding a raise is an `INSERT`, never an `UPDATE`. There is no column to overwrite, so there is no way to lose the history - which is the entire point, and the same reasoning that makes `feed_log` append-only in Stage 6. (If you want to know not just what the rate *was* but what you *believed* it was, you need `effective_to`, or full bitemporal columns: `valid_from`, `valid_to`, `recorded_at`. That is a genuinely deeper design, and worth reading about before you need it rather than after.)

### 15.6 The payroll run

A payslip is a claim about a period, and the run that produces it must be **idempotent** - safe to execute twice, because it will be executed twice. The constraint that makes it so is a unique key on the period (`migrations/00012_payslips.sql`):

```sql
-- +goose Up
CREATE TABLE payslips (
    id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    zookeeper_id uuid NOT NULL REFERENCES zookeepers(id),
    period       tstzrange NOT NULL,
    minutes      int NOT NULL CHECK (minutes >= 0),
    gross_cents  bigint NOT NULL CHECK (gross_cents >= 0),
    created_at   timestamptz NOT NULL DEFAULT now(),
    UNIQUE (zookeeper_id, period),
    CONSTRAINT payslips_period_not_empty CHECK (NOT isempty(period))
);

-- +goose Down
DROP TABLE payslips;
```

and the run itself, which is the densest query in this tutorial and repays being read line by line:

```sql
WITH period AS (SELECT tstzrange($1, $2) AS p)
INSERT INTO payslips (zookeeper_id, period, minutes, gross_cents)
SELECT s.zookeeper_id,
       p.p,
       sum(EXTRACT(epoch FROM upper(s.during * p.p) - lower(s.during * p.p)) / 60)::int,
       sum(EXTRACT(epoch FROM upper(s.during * p.p) - lower(s.during * p.p)) / 3600 * r.hourly_cents)::bigint
FROM shifts s
CROSS JOIN period p
JOIN LATERAL (
    SELECT hourly_cents
    FROM pay_rates pr
    WHERE pr.zookeeper_id = s.zookeeper_id
      AND pr.effective_from <= lower(s.during)      -- the rate in force when the shift STARTED
    ORDER BY pr.effective_from DESC
    LIMIT 1
) r ON true
WHERE s.during && p.p
GROUP BY s.zookeeper_id, p.p
ON CONFLICT (zookeeper_id, period) DO NOTHING;
```

- **`JOIN LATERAL ... LIMIT 1`** is "run this subquery once per shift row, and let it see that row's columns". It is how you resolve the rate *in force at the start of this particular shift*, which is the correct rule: a raise at noon does not retroactively reprice the morning you already worked. Verified with a raise at 12:00 between two shifts on the same day - 8 hours at 2000 and 4 hours at 2500 came out as exactly `26000` cents and `720` minutes.
- **`s.during * p.p`** is range *intersection*, and it is what makes a period boundary honest. Without it, a shift running 15:00 to 19:00 would be paid as four hours even when the pay period *begins* at 16:00. With it, the counted interval is `["16:00","19:00")` and the minutes are 180. (The other boundary is the mirror image and worth checking you understand: a period that *ends* at 16:00 clips the same shift to `["15:00","16:00")`, which is 60 minutes. Get the direction wrong and a night shift is paid for hours that belong to the next period.) This matters every single time a pay period ends mid-shift, which for anyone working nights is most of them.
- **`EXTRACT(epoch FROM interval)`** gives seconds; the division and `::int` / `::bigint` casts are explicit because Postgres will otherwise hand you `numeric` with a long tail of decimals, and rounding money is a decision you should have to write down. Where you round is a *policy* question - this query truncates the aggregate, a real payroll rounds each line and carries the residue in a ledger - so the tutorial shows the shape and says out loud that the policy is yours.
- **`ON CONFLICT ... DO NOTHING`** makes the run safe to repeat: the second execution over the same period inserts `INSERT 0 0`. Note that this is the correct idempotency key only because `UNIQUE (zookeeper_id, period)` exists; a range column can carry a unique constraint (ranges have a btree operator class, and equal ranges compare equal), which is not obvious and is worth checking rather than assuming.

One more rule that has nothing to do with SQL and everything to do with payroll: **a finalised payslip is immutable.** Corrections are new rows (an adjustment against a later period), never edits to history. If your payroll table can be `UPDATE`d after the money has left the building, you have an audit problem, not a schema problem - and the fix is a permissions one (`REVOKE UPDATE ON payslips`) backed by the append-only habit this tutorial has followed since `feed_log`.

> **What this is not.** Real payroll is regulated: tax codes, statutory deductions, pension contributions, minimum-wage floors, retroactive corrections, and a filing obligation at the end. This section shows how to compute *hours worked at the right rate*, which is the part that belongs in your domain model; everything after that belongs behind a payroll provider's API, and the right move is to export to one rather than to write it. Knowing where the boundary is, is itself the lesson.

### 15.7 Verify

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo
```

```sql
-- Start from an empty roster. 15.2 already inserted shifts at exactly these
-- times, and re-running those inserts would trip the exclusion constraint
-- before we want it to.
DELETE FROM shifts;

-- double booking is refused by the database, and it does not care which zoo
INSERT INTO shifts (zookeeper_id, zoo_id, during)
VALUES ('00000000-0000-4000-8000-000000000001', '00000000-0000-4000-8000-000000000001', tstzrange('2026-10-12 08:00+01', '2026-10-12 16:00+01'));
INSERT INTO shifts (zookeeper_id, zoo_id, during)
VALUES ('00000000-0000-4000-8000-000000000001', '00000000-0000-4000-8000-000000000002', tstzrange('2026-10-12 15:00+01', '2026-10-12 20:00+01'));   -- ERROR 23P01

-- back-to-back is not a conflict
INSERT INTO shifts (zookeeper_id, zoo_id, during)
VALUES ('00000000-0000-4000-8000-000000000001', '00000000-0000-4000-8000-000000000002', tstzrange('2026-10-12 16:00+01', '2026-10-12 20:00+01'));   -- INSERT 0 1

-- A shift covering now, so the next query finds something. It has to be
-- built from now(): the tutorial's shifts are all pinned to dates in October
-- 2026, which is part of why its example ids are stable and its clock is not.
-- 15.7 opens by deleting the roster again, so this one is not left behind.
INSERT INTO shifts (zookeeper_id, zoo_id, during)
VALUES ('00000000-0000-4000-8000-000000000001', '00000000-0000-4000-8000-000000000002', tstzrange(now() - interval '1 hour', now() + interval '1 hour'));

-- who is on duty right now
SELECT s.zookeeper_id, z.name, s.during FROM shifts s
JOIN zoos z ON z.id = s.zoo_id WHERE s.during @> now();

-- the same shift list, materialised from a weekly pattern, survives a clock change
INSERT INTO shift_templates (zookeeper_id, zoo_id, iso_dow, starts_at, ends_at)
VALUES ('00000000-0000-4000-8000-000000000002', '00000000-0000-4000-8000-000000000001', 3, '08:00', '16:00');
INSERT INTO shifts (zookeeper_id, zoo_id, during)
SELECT t.zookeeper_id, t.zoo_id,
       tstzrange((d::date + t.starts_at) AT TIME ZONE 'Europe/London',
                 (d::date + t.ends_at)   AT TIME ZONE 'Europe/London')
FROM shift_templates t
CROSS JOIN generate_series('2026-10-19'::date, '2026-11-08'::date, interval '1 day') AS d
WHERE EXTRACT(isodow FROM d) = t.iso_dow
ON CONFLICT DO NOTHING;    -- 15.3's idempotence clause: the generated shifts are safe to re-run
SELECT during FROM shifts WHERE zookeeper_id = '00000000-0000-4000-8000-000000000002' ORDER BY during;
-- the 21st and the 28th are both 08:00 in London, and an hour apart in UTC

-- Note the split in what "safe to repeat" covers. The generator above is idempotent, so
-- re-running the whole block never duplicates a shift. The template INSERT on the line
-- before it is not: `shift_templates` has no unique key on the pattern, so a second run
-- leaves a second identical template and the generator then sees six candidates instead
-- of three. The three extra rows all collide with shifts that already exist and are
-- dropped, so the output is unchanged - but the table quietly grows, which is the sort of
-- thing that only shows up as a wrong count months later. Re-runnable fixtures earn their
-- keep by being re-runnable from the top: 15.7 already opens with `DELETE FROM shifts`,
-- so add the matching `DELETE FROM shift_templates;` above this insert if you mean to
-- run the block more than once.

-- the payroll run, self-contained this time: 15.6's version takes $1 and $2
-- as the period, and psql cannot bind placeholders, so the dates are written
-- out. The shift list is the two October shifts inserted above.
INSERT INTO pay_rates (zookeeper_id, hourly_cents, effective_from) VALUES
    ('00000000-0000-4000-8000-000000000001', 2000, '2026-01-01 00:00Z'),
    ('00000000-0000-4000-8000-000000000001', 2500, '2026-10-12 12:00Z');

INSERT INTO payslips (zookeeper_id, period, minutes, gross_cents)
SELECT s.zookeeper_id, p.p,
       sum(EXTRACT(epoch FROM upper(s.during * p.p) - lower(s.during * p.p)) / 60)::int,
       sum(EXTRACT(epoch FROM upper(s.during * p.p) - lower(s.during * p.p)) / 3600 * r.hourly_cents)::bigint
FROM shifts s
CROSS JOIN (SELECT tstzrange('2026-10-12 00:00Z', '2026-10-13 00:00Z') AS p) p
JOIN LATERAL (
    SELECT hourly_cents FROM pay_rates pr
    WHERE pr.zookeeper_id = s.zookeeper_id
      AND pr.effective_from <= lower(s.during)
    ORDER BY pr.effective_from DESC
    LIMIT 1
) r ON true
WHERE s.during && p.p
GROUP BY s.zookeeper_id, p.p
ON CONFLICT (zookeeper_id, period) DO NOTHING;

SELECT zookeeper_id, minutes, gross_cents FROM payslips;
--  00000000-0000-4000-8000-000000000001 | 720 | 26000      <- 8h at 2000 plus 4h at 2500
-- Run that INSERT again and it reports INSERT 0 0: the payslip is already there.
```

The habit worth keeping from this section: when a business rule is about *time*, look for the type that already models it before reaching for columns that describe it. `tstzrange` plus an exclusion constraint replaced an overlap check that would have been written in application code, tested inconsistently, and bypassed by the first script that inserted shifts directly.

---

[Bonus 14](14-bonus-multi-zoo-enclosures.md)  |  [Overview](../tutorial.md)  |  [Bonus 16](16-bonus-transfers-and-next-steps.md)
