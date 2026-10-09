# Zoo Service

A Zoo management API: a remake of the official Go tutorial "Designing an API
with Gin", production-shaped, built on the standard Go server stack rather
than a web framework. Read `docs/tutorial.md` (the overview and table of
contents) and build it yourself, stage by stage - each stage lives in its own
file under `docs/stages/`.

## Stack

Go 1.22+, chi (router), cobra + viper (CLI and config), slog for structured
logging, pgx (PostgreSQL), goose (SQL migrations), golang-jwt/v5 + bcrypt,
Docker Compose for the database.

## Quick start

1. `export JWT_SECRET="a-long-random-string-for-dev-only-not-a-real-secret"`
2. `make up`
3. `make migrate`
4. `make run`
5. Log in: `POST /api/v1/login` with `{"username":"admin","password":"zoo-admin-password"}`
   (the seeded admin from migration 00003; dev credential, rotate per environment)

The single binary is `cmd/zoo`, with two subcommands: `zoo serve` runs the API,
`zoo migrate` applies the SQL migrations. Both read configuration from
environment variables (override anything with a `zoo.yaml` in the working
directory, or point at one with `--config`).

Full API documentation: the route table at the bottom of `docs/stages/12-makefile-recap.md`.

Stage 13 swaps every id for a UUID. Beyond it are three optional bonus sections
(14-16) covering multiple zoos, enclosure capacity and environments, staff shifts
and payroll, and animal transfers - see the "Bonus material" table in
`docs/tutorial.md`.

## Layout

`cmd/zoo` is the one binary. `internal/` holds the private code: one directory
per domain (animals, zookeepers), plus `cli` (the cobra commands), `server`
(the router), and `platform` (config, database, auth, httpx, logging). See
`docs/stages/05-restructure-internal.md` for why.
