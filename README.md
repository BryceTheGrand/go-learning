# Zoo Service

A Zoo management API: a remake of the official Go tutorial "Designing an API
with Gin", production-shaped. Read `docs/tutorial.md` (the overview and
table of contents) and build it yourself, stage by stage - each stage lives
in its own file under `docs/stages/`.

## Stack

Go 1.27, gin, pgx (PostgreSQL), goose (SQL migrations), golang-jwt/v5 + bcrypt,
Docker Compose for the database.

## Quick start

1. `export JWT_SECRET="a-long-random-string-for-dev-only-not-a-real-secret"`
2. `make up`
3. `make migrate`
4. `make run`
5. Log in: `POST /api/v1/login` with `{"username":"admin","password":"zoo-admin-password"}`
   (the seeded admin from migration 00003; dev credential, rotate per environment)

Full API documentation: the route table at the bottom of `docs/stages/12-makefile-recap.md`.

## Layout

`cmd/` holds the two binaries (apiserver, migrate). `internal/` holds the
private code: one directory per domain (animals, zookeepers), plus `platform`
(config, database, auth, httperrors) and `server` (the router). See
`docs/tutorial.md` Stage 5 for why.