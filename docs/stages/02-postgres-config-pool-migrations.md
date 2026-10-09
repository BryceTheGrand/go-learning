## Stage 2: A real database: Postgres, config, pool, migrations

The server still serves in-memory animals. This stage wires the plumbing a database-backed app needs: a containerized Postgres, configuration, a pooled connection, structured logging, a command line, and a small migration tool. Stage 3 will make zookeepers the first thing actually stored.

It is also the stage where the project stops being one file. You will meet four new directories - `cmd/`, `internal/cli/`, `internal/platform/`, and `migrations/` - and two tools that show up in nearly every production Go service: **viper** for configuration and **cobra** for the command line.

### 2.1 Run Postgres with Docker Compose

`docker-compose.yml` in the project root:

```yaml
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: zoo
      POSTGRES_USER: zoo
      POSTGRES_PASSWORD: zoo
    ports:
      - "5433:5432"   # host 5433 -> container 5432
    volumes:
      - zoo_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zoo -d zoo"]
      interval: 2s
      timeout: 2s
      retries: 15

volumes:
  zoo_pgdata:
```

Notes that will save you from mysterious failures:

- Host port **5433**, not 5432: if anything (a local Postgres install, another project) is squatting on 5432, mapping to it fails or - sneakier - connects you to the *other* database and nothing in this tutorial's config would explain the row counts. We pin a boring non-standard port instead.
- `volumes: zoo_pgdata` keeps data between `docker compose down`. Your schema survives Ctrl+C; wipe it with `docker compose down -v` (the `-v` also removes the volume).
- The healthcheck matters: Postgres accepting TCP connections does not mean it finished initializing on first boot. Other stages rely on the database being genuinely ready, and `pg_isready` waits for that.

```bash
docker compose up -d
docker compose ps        # db "healthy" within a few seconds
```

### 2.2 Configuration with viper

This file also introduces a directory you have not met: `internal/platform/`. `platform` is this project's name for *infrastructure the domains all lean on* - configuration, database, logging, later auth and error plumbing. It is lowercase (`internal`) so the compiler keeps outsiders away; the reason `internal` means that arrives properly in Stage 5.

The dependency first:

```bash
go get github.com/spf13/cobra@v1.10.2 github.com/spf13/viper@v1.20.1
```

(Viper arrives here with cobra even though we do not write a command until 2.5; they are two halves of the same "CLI" story, and `internal/cli` uses both. Notice there is no `go mod tidy` on that line yet. The reason is worth knowing before it bites you: `tidy` **removes** requirements that nothing in the module imports, and at this moment nothing imports cobra - the files that will are created in 2.5. The tidy call comes at the end of 2.5, when there is something to keep.)

As in Stage 1, we build the file piece by piece and give you the whole file at the end.

#### The type

`internal/platform/config/config.go`:

```go
package config

import "log/slog"

// Config is the process's configuration, parsed once at startup.
type Config struct {
	Port        string
	DatabaseURL string
	LogLevel    slog.Level
	LogFormat   string
}
```

- **`package config`**: first non-`main` package of the project. Import path `zoo/internal/platform/config`, package name `config` - other files will say `config.Load()`, the directory name matching the package name by convention.
- **`Config struct`**: a struct again, like Stage 1's `animal`, but exported (capital `C`) because *other packages* will build and consume it. Its fields are exported too - a configuration type is exactly the thing you want everyone to be able to read. Note there are no JSON tags: nothing here is ever serialized; tags are wire-shape concerns only.
- Note what the fields' *types* tell you: the two strings are text - a URL is text, there is no integer type in Go with "5433" vs `5433` - `LogLevel` is `slog.Level`, the logger's own level type rather than a string, so the rest of the program never has to parse it; and Stage 4 adds a `time.Duration` for the token lifetime and shows the same trade again.

#### Load: defaults, a config file, environment variables

```go
package config

import (
	"fmt"
	"log/slog"
	"os"

	"github.com/spf13/viper"
)

// Load builds the configuration with viper: defaults first, then an optional
// YAML config file, then environment variables, which win. An explicit
// configFile path that does not exist is an error; a missing ./zoo.yaml is not.
func Load(configFile string) (Config, error) {
	v := viper.New()
	v.SetDefault("port", "8080")
	v.SetDefault("database_url", "postgres://zoo:zoo@localhost:5433/zoo?sslmode=disable")
	v.SetDefault("log_level", "info")
	v.SetDefault("log_format", "text")

	const defaultConfigFile = "zoo.yaml"

	if configFile == "" && fileExists(defaultConfigFile) {
		configFile = defaultConfigFile
	}
	if configFile != "" {
		v.SetConfigFile(configFile)
		if err := v.ReadInConfig(); err != nil {
			return Config{}, fmt.Errorf("read config file: %w", err)
		}
	}

	v.AutomaticEnv()

	var level slog.Level
	if err := level.UnmarshalText([]byte(v.GetString("log_level"))); err != nil {
		return Config{}, fmt.Errorf("invalid log_level: %w", err)
	}

	return Config{
		Port:        v.GetString("port"),
		DatabaseURL: v.GetString("database_url"),
		LogLevel:    level,
		LogFormat:   v.GetString("log_format"),
	}, nil
}

// fileExists reports whether path names a regular file.
func fileExists(path string) bool {
	info, err := os.Stat(path)
	return err == nil && !info.IsDir()
}
```

- **`func Load(configFile string) (Config, error)`**: the first *multi-value return* in the tutorial. Stage 1 taught the error convention from the caller's side (`x, err := f(...)`); this is the callee's side: **result first, error last**. Everything that can fail in this tutorial follows it.
- **What viper actually buys you.** The Stage 1 era of Go config was `os.Getenv("PORT")` scattered through the code. Viper replaces that with a *layered lookup*: a default, overridden by a config file, overridden by an environment variable. The `SetDefault` calls are that first layer, and they mean every setting has a working development value without a single line of "if it is empty, use this" boilerplate at the call site.
- **The config file is looked up explicitly rather than searched for.** If `--config` names one it must exist; otherwise `zoo.yaml` in the working directory is loaded if, and only if, it is there. That `fileExists` check is doing real work, and the reason is worth the two lines. The obvious alternative is viper's built-in search - `v.SetConfigName("zoo")` plus `v.AddConfigPath(".")` - and it has a nasty edge: the search also matches a file named exactly `zoo`, and `go build ./cmd/zoo` produces precisely that in the project root. The next run then tries to parse the compiled binary as YAML and dies with:
  ```
  ERROR command failed error="read config file: While parsing config: yaml: control characters are not allowed"
  ```
  which says nothing at all about the cause. Naming the file you actually want, and checking for it, removes the whole class of problem - and as a bonus there is no `viper.ConfigFileNotFoundError` branch to write, because a missing file never reaches viper. (That typed error, plus `errors.As` to test for it, is what the search-based version needs; Stage 3's database code uses the same `errors.As` to dig a `*pgconn.PgError` out of an error chain.)
- **`v.AutomaticEnv()`**: after this line, any key viper has seen resolves against the environment too, with the key upper-cased. So `port` reads `PORT`, `database_url` reads `DATABASE_URL`, `log_level` reads `LOG_LEVEL`. That mapping is viper's convention and it is why the key names here are snake_case: they *are* the environment variable names.
- **`v.GetDuration("token_ttl")`** (Stage 4) is the other half of the argument for viper: environment variables are always strings, and viper coerces "24h" into a `time.Duration` for you, so `"90m"` or `"1h30m"` work without a parsing call at every use site.
- **`level.UnmarshalText`**: `slog.Level` knows how to parse "debug", "info", "warn" and "error" (case-insensitively). It is a value receiver method on a type that also satisfies the `encoding.TextUnmarshaler` interface - the standard-library way for a type to declare "here is how I parse myself from text".
- **`return Config{}, err`** on failure: the *zero value* `Config{}` plus the error, never a half-filled struct. The error convention from Stage 1 applies to returns as well as calls.
- Design note worth internalizing: **configuration is parsed once, at startup, into a plain struct** - no `config.Get("database_url")` scattered through the code. Callers receive the struct and pass what they need. Viper is used *inside this one file*; nothing else in the project ever imports it.
- Why does a function that cannot fail today return `(Config, error)`? Because Stage 4's JWT secret *will* be required and missing-secret must be a startup crash. Changing a function's return signature later means editing every caller; carrying the error slot costs nothing now. Go convention is to design returns for the steady state.

The complete file:

```go
package config

import (
	"fmt"
	"log/slog"
	"os"

	"github.com/spf13/viper"
)

// Config is the process's configuration, parsed once at startup.
type Config struct {
	Port        string
	DatabaseURL string
	LogLevel    slog.Level
	LogFormat   string
}

// Load builds the configuration with viper: defaults first, then an optional
// YAML config file, then environment variables, which win. An explicit
// configFile path that does not exist is an error; a missing ./zoo.yaml is not.
func Load(configFile string) (Config, error) {
	v := viper.New()
	v.SetDefault("port", "8080")
	v.SetDefault("database_url", "postgres://zoo:zoo@localhost:5433/zoo?sslmode=disable")
	v.SetDefault("log_level", "info")
	v.SetDefault("log_format", "text")

	const defaultConfigFile = "zoo.yaml"

	if configFile == "" && fileExists(defaultConfigFile) {
		configFile = defaultConfigFile
	}
	if configFile != "" {
		v.SetConfigFile(configFile)
		if err := v.ReadInConfig(); err != nil {
			return Config{}, fmt.Errorf("read config file: %w", err)
		}
	}

	v.AutomaticEnv()

	var level slog.Level
	if err := level.UnmarshalText([]byte(v.GetString("log_level"))); err != nil {
		return Config{}, fmt.Errorf("invalid log_level: %w", err)
	}

	return Config{
		Port:        v.GetString("port"),
		DatabaseURL: v.GetString("database_url"),
		LogLevel:    level,
		LogFormat:   v.GetString("log_format"),
	}, nil
}

// fileExists reports whether path names a regular file.
func fileExists(path string) bool {
	info, err := os.Stat(path)
	return err == nil && !info.IsDir()
}
```

One thing worth saying now because it shapes everything: **constructors that return `(value, error)` are the standard shape of Go error handling.** The convention is `error last` in every return list. The platform packages follow it everywhere.

### 2.3 The connection pool

`internal/platform/database/pool.go` - a package with one function. Its purpose is a decision disguised as plumbing: *one process, one pool*. Handlers do not each open connections; the pool owns them.

```bash
go get github.com/jackc/pgx/v5@v5.7.4 github.com/pressly/goose/v3@v3.24.1
```

(Same story as 2.2: `go get` records the module, and the `go mod tidy` that prunes and pins the full `go.sum` closure waits until 2.5, because nothing imports goose yet either.)

#### The signature: pointers appear for the first time

```go
package database

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// NewPool opens a pgx connection pool and verifies the database is reachable.
// A pool is safe for many concurrent users; a single pgx connection is not.
// Your handlers get a pool and let it manage who talks to Postgres when.
func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
	pool, err := pgxpool.New(ctx, databaseURL)
	if err != nil {
		return nil, fmt.Errorf("parse database url: %w", err)
	}

	if err := pool.Ping(ctx); err != nil {
		pool.Close()
		return nil, fmt.Errorf("connect to database: %w", err)
	}

	return pool, nil
}
```

- **`ctx context.Context` as the first parameter**: you met "a thing called context" nowhere yet, and this tutorial will not fully explain it until Stage 7. For now, accept the convention: functions that touch the outside world (database, network) take a `context.Context` first. It carries cancellation and deadlines; `context.Background()` is the "no parent" context, what `main` packages start with.
- **`*pgxpool.Pool`** is the first pointer type in a return position. Rules of thumb, now and for the rest of the tutorial: structs come back as pointers (*pgxpool.Pool, and from Stage 5 every domain handler/service), because you are handing over *the* thing, not a copy; a `nil` pointer is the "nothing" a failed constructor returns here (`return nil, fmt.Errorf(...)`).
- **`pool.Ping(ctx)`** is the fail-fast: a URL may parse fine and the database still be down. Catching that at process start, not at first request, is a production habit worth copying. (The `pool.Close()` before the error return releases the half-open pool - a constructor cleans up its own mess.)
- **`fmt.Errorf("parse database url: %w", err)`**: `%w` is Go's error-*wrapping* verb. The error you produce *contains* the one you were handed, so a caller can later ask "what was underneath?" (`errors.As`/`errors.Is`, exploited properly in Stage 9). The prefix text is your addition: it answers the "failed *where*?" question. Wrap at every layer, briefly; that is idiomatic Go, and the layering of messages in Stage 9's responses to failures will trace straight down to these words.

The complete file:

```go
package database

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// NewPool opens a pgx connection pool and verifies the database is reachable.
// A pool is safe for many concurrent users; a single pgx connection is not.
// Your handlers get a pool and let it manage who talks to Postgres when.
func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
	pool, err := pgxpool.New(ctx, databaseURL)
	if err != nil {
		return nil, fmt.Errorf("parse database url: %w", err)
	}

	if err := pool.Ping(ctx); err != nil {
		pool.Close()
		return nil, fmt.Errorf("connect to database: %w", err)
	}

	return pool, nil
}
```

### 2.4 Structured logging with slog

Stage 1 already logs with `slog`, but it logs through the *default* handler, which is not something you would ship: the level is fixed at INFO and there is no way to ask for JSON. One small package fixes that, and every later stage logs through it.

`internal/platform/logging/logging.go`:

```go
package logging

import (
	"log/slog"
	"os"
)

// Setup installs the process-wide slog logger. Call it once, at startup.
func Setup(level slog.Level, format string) {
	opts := &slog.HandlerOptions{Level: level}
	var handler slog.Handler
	if format == "json" {
		handler = slog.NewJSONHandler(os.Stdout, opts)
	} else {
		handler = slog.NewTextHandler(os.Stdout, opts)
	}
	slog.SetDefault(slog.New(handler))
}
```

- **What `slog` is.** Go's structured logger, standard library since Go 1.21. A log *record* is a message plus key/value pairs, and a *handler* decides what to do with it. That split is the design: `slog.Info("http", "status", 404)` is the same call whether the handler writes `status=404` for a human reading a terminal or `{"status":404}` for a log pipeline - only the handler changes.
- **`slog.SetDefault(...)`** is the one piece of global state this tutorial deliberately keeps. The alternative is threading a `*slog.Logger` through every constructor in the project, which is a real pattern (and the wrap-up mentions it); the standard library's own answer is that the default logger covers the common case, and one `Setup` call at startup is how you configure it.
- **`slog.HandlerOptions{Level: level}`** is where the `LOG_LEVEL` environment variable finally lands: a `debug` level makes `slog.Debug` calls visible, `warn` hides everything below it.
- **Two handlers, one switch.** `NewTextHandler` writes the logfmt-ish `time=... level=INFO msg=http method=GET` form you saw in Stage 1; `NewJSONHandler` writes one JSON object per line, which is what a log collector wants. `LOG_FORMAT=json` selects it. Both are standard library; no logging dependency appears in `go.mod`.

### 2.5 The command line with cobra

So far the only way to run anything has been `go run .`, which works exactly as long as there is exactly one thing to run. Migrations are now a second thing, and they need arguments of their own, so this is where the project grows a real command line. The tool for it is [cobra](https://github.com/spf13/cobra) - the library behind `kubectl`, `hugo`, `gh`, and a great many Go services.

The shape we are building toward is one binary with subcommands:

```bash
go run ./cmd/zoo serve      # the API server (arrives in Stage 5)
go run ./cmd/zoo migrate    # apply SQL migrations
```

At this stage only `migrate` exists; the API server is still the flat `main.go` at the repo root and joins the CLI in Stage 5, when it stops being flat anyway.

#### The entry point: cmd/zoo/main.go

```go
package main

import "zoo/internal/cli"

func main() {
	cli.Execute()
}
```

- **`cmd/zoo/`** is the first `cmd/` directory of the project, and it exists for exactly one reason: each executable gets its own directory with a thin `main.go`. The directory name becomes the binary name - `go build ./cmd/zoo` produces `zoo`. There is nothing clever in this file, and that is the point; all of the program lives in importable packages.
- **`zoo/internal/cli`** is where the commands live. Real projects vary here (`cmd/` with `package main` and several files is the other common shape), but putting the commands under `internal/` makes them ordinary importable code: a test can call `newRootCmd()` and execute it, which is precisely what `internal/cli` gets away with because it is not inside `package main`.

#### The root command: internal/cli/root.go

```go
package cli

import (
	"log/slog"
	"os"

	"github.com/spf13/cobra"

	"zoo/internal/platform/config"
	"zoo/internal/platform/logging"
)

// app carries what every subcommand needs: the --config flag's value and the
// configuration parsed from it.
type app struct {
	configFile string
	cfg        config.Config
}

// Execute runs the root command and turns a command failure into a non-zero
// exit. main called this; nothing else in the program exits the process.
func Execute() {
	if err := newRootCmd().Execute(); err != nil {
		slog.Error("command failed", "error", err)
		os.Exit(1)
	}
}

func newRootCmd() *cobra.Command {
	a := &app{}

	root := &cobra.Command{
		Use:           "zoo",
		Short:         "Zoo management API",
		SilenceUsage:  true,
		SilenceErrors: true,
		PersistentPreRunE: func(_ *cobra.Command, _ []string) error {
			cfg, err := config.Load(a.configFile)
			if err != nil {
				return err
			}
			logging.Setup(cfg.LogLevel, cfg.LogFormat)
			a.cfg = cfg
			return nil
		},
	}

	root.PersistentFlags().StringVar(&a.configFile, "config", "", "path to a YAML config file (optional)")

	root.AddCommand(newMigrateCmd(a))
	return root
}
```

- **`&cobra.Command{...}`** is a struct literal (Stage 1 met those) with fields that describe the command: `Use` is the one-line invocation shown in help, `Short` the description beside it. `Execute()` parses `os.Args`, finds the matching subcommand, and runs it.
- **`RunE` and the main/run split.** cobra's convention is that a command does its real work in `RunE`, which returns an `error`, and that *something upstream* decides what a non-zero exit means. That is exactly the `main`/`run` split this tutorial has been teaching, except the framework now owns half of it: `Execute()` in this file is the only place that calls `os.Exit`. Look for `RunE` in any cobra project and you have found where the work starts.
- **`SilenceUsage: true, SilenceErrors: true`**: by default cobra prints the full usage text after any error, which is noise when the error is "the database is down" rather than "you typed the flag wrong". Silencing both means cobra neither dumps usage nor prints the error itself, leaving the reporting to `Execute()` above. Flip `SilenceUsage` back off and you will see the difference immediately.
- **`PersistentPreRunE`** runs before every subcommand's `RunE` (persistent = inherited by children). This is the natural home for "read configuration, configure logging, once": every command needs it, and doing it here means `runMigrations` receives an already-configured `slog` and an already-parsed `config.Config`, and never thinks about either.
- **`root.PersistentFlags().StringVar(&a.configFile, ...)`**: a *persistent* flag is inherited by subcommands, so `zoo migrate --config prod.yaml` and `zoo --config prod.yaml migrate` both work. `StringVar` binds the flag to a variable directly, and the third argument is the help text.
- **`AddCommand(newMigrateCmd(a))`**: the root command holds a tree of subcommands. Each `new<X>Cmd` function *returns* a command - the same factory shape as Stage 1's `writeJSON` helper and, later, Stage 4's middleware; this codebase consistently prefers a function that builds and returns a value over one that mutates a shared global.
- **The `app` struct** is the one piece of shared state: `a.configFile` is written by the flag, `a.cfg` by `PersistentPreRunE`, and each command reads them in its `RunE`. Since `newRootCmd` passes the same `a` pointer to every subcommand, this is how a cobra program threads configuration without globals.

#### The migrate subcommand: internal/cli/migrate.go

Migrations are plain SQL files. goose is a tiny library that tracks which files have been applied (in a table it manages for you) and applies the rest in order. We do not use the goose CLI; we wrap it in a subcommand of our own. This file teaches two habits: the `RunE`-returns-an-error split you just met, and `defer`.

```go
package cli

import (
	"context"
	"fmt"
	"log/slog"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib"
	"github.com/pressly/goose/v3"
	"github.com/spf13/cobra"
)

func newMigrateCmd(a *app) *cobra.Command {
	return &cobra.Command{
		Use:   "migrate",
		Short: "Apply the SQL migrations in ./migrations",
		RunE: func(cmd *cobra.Command, _ []string) error {
			return runMigrations(cmd.Context(), a.cfg.DatabaseURL)
		},
	}
}

func runMigrations(ctx context.Context, databaseURL string) error {
	pool, err := pgxpool.New(ctx, databaseURL)
	if err != nil {
		return fmt.Errorf("connect: %w", err)
	}
	defer pool.Close()

	// goose wants a database/sql-style handle; the pgx stdlib adapter gives us
	// one backed by the very same pool we already open.
	db := stdlib.OpenDBFromPool(pool)
	defer db.Close()

	if err := goose.SetDialect("postgres"); err != nil {
		return fmt.Errorf("set dialect: %w", err)
	}

	if err := goose.Up(db, "migrations"); err != nil {
		return fmt.Errorf("run migrations: %w", err)
	}

	slog.Info("migrations applied", "dir", "migrations")
	return nil
}
```

- **`RunE: func(cmd *cobra.Command, _ []string) error { ... }`**: the two parameters cobra hands every command are the command itself (from which `cmd.Context()` comes) and the positional arguments after the flags, which this command takes none of - hence the `_`. Note the body is one line: all the work is in a plain function below it, taking exactly the values it needs. That is the same reason Stage 1 kept handlers thin.
- **`slog.Info("migrations applied", "dir", "migrations")`** is this command's own line, and it is here mostly so you can *see* the configuration from 2.4 take effect: this one line is the difference between the readable `time=... msg="migrations applied" dir=migrations` form and `{"msg":"migrations applied","dir":"migrations",...}` under `LOG_FORMAT=json`. goose's own progress lines come out in the same format, because goose logs through `slog` when it is available - which is a quiet argument for the standard library's logger being the default choice rather than one library among many.
- **`cmd.Context()`** is a `context.Context` cobra carries for the life of the command, and it is what a real `run` function should hang its work off. Stage 10 gets a signal-cancelled context in here; for now it is the "cancel me never" root context.
- **`defer pool.Close()`** is the first `defer` in the tutorial. It schedules the close call to run **when the surrounding function returns** - however it returns: the happy path, or any of the `return fmt.Errorf(...)` exits above and below. That is the value: one line, placed right after the resource is acquired, that can never be forgotten on the error paths. `defer`s run LIFO (last deferred, first to run); Stage 7 leans on this in its transaction rollback pattern.
- **`stdlib.OpenDBFromPool(pool)`** is an adapter: goose (like many libraries) speaks the old standard `database/sql` interface, and pgx's `stdlib` package hands it a `database/sql`-shaped view of your *existing* pool. Two types, one connection budget. `defer db.Close()` closes the adapter's view, not the pool (the pgx docs are explicit about that), so the first `defer` still owns the real resource. Note the teardown order that falls out of this: `defer`s run LIFO, so `db.Close()` runs first and `pool.Close()` second - the borrower is released before the thing it borrows from. That is the order you want, and it came for free from acquiring `db` after `pool`.
- A related trap worth filing away now: `defer`s fire when the *enclosing function returns*, and `os.Exit` does not run them. The one `os.Exit(1)` in this program lives in `Execute()`, and by the time it runs, every `run` function has already returned and its defers have already fired. Deferring cleanup directly in `main` would be the mistake.
- **`goose.Up(db, "migrations")`** applies every not-yet-applied migration file in the directory, in filename order, and records them. `SetDialect` first tells it the vendor (it needs to know what a "now()" looks like, among other things).

The complete file:

```go
package cli

import (
	"context"
	"fmt"
	"log/slog"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib"
	"github.com/pressly/goose/v3"
	"github.com/spf13/cobra"
)

func newMigrateCmd(a *app) *cobra.Command {
	return &cobra.Command{
		Use:   "migrate",
		Short: "Apply the SQL migrations in ./migrations",
		RunE: func(cmd *cobra.Command, _ []string) error {
			return runMigrations(cmd.Context(), a.cfg.DatabaseURL)
		},
	}
}

func runMigrations(ctx context.Context, databaseURL string) error {
	pool, err := pgxpool.New(ctx, databaseURL)
	if err != nil {
		return fmt.Errorf("connect: %w", err)
	}
	defer pool.Close()

	// goose wants a database/sql-style handle; the pgx stdlib adapter gives us
	// one backed by the very same pool we already open.
	db := stdlib.OpenDBFromPool(pool)
	defer db.Close()

	if err := goose.SetDialect("postgres"); err != nil {
		return fmt.Errorf("set dialect: %w", err)
	}

	if err := goose.Up(db, "migrations"); err != nil {
		return fmt.Errorf("run migrations: %w", err)
	}

	slog.Info("migrations applied", "dir", "migrations")
	return nil
}
```

Now that something actually imports cobra, goose and viper, run the tidy the earlier sections deferred:

```bash
go mod tidy
go build ./...
```

`go get` records *a* version for a module you named; `go mod tidy` reconciles `go.mod` and `go.sum` with what the code actually imports - adding the transitive dependencies `go.sum` needs (without them the build fails with "missing go.sum entry" for a dependency of a dependency) and removing anything nothing uses. Which is exactly the pruning that would have deleted cobra and goose if you had run it back in 2.2 and 2.3, before these files existed. Two related habits worth keeping: run `tidy` after adding or removing imports rather than after every `go get`, and never let a *bare* `go get` or `tidy` re-resolve a package you pinned with `@vX.Y.Z` - it will happily take a newer release, and newer releases of these particular libraries require a newer Go toolchain than the one this tutorial targets.

`go build ./...` should be silent. If it reports `no required module provides package github.com/pressly/goose/v3`, the tidy was run too early, and re-running the `go get ...@version` line from 2.3 restores it.

Two observations:

- This is the module's **second program**, and the first one that is not the API server. One module, several commands; that is exactly why `cmd/` exists.
- It reads SQL files from the `migrations/` directory **relative to where you run it**, so always run it from the project root.

One version note for the record: `goose.SetDialect` + `goose.Up(db, dir)` is goose's legacy API - the current docs point to a newer `Provider` interface instead. We keep the legacy pair here because it is still fully supported, appears in every goose example you will find, and is the smallest thing that does the job; a note in the goose docs marks the newer API for when you want stricter store handling.

### 2.6 First migration: zookeepers

goose has its own file convention (a different one from golang-migrate): **one file per version**, with `-- +goose Up` and `-- +goose Down` annotations inside the file marking which statements go which direction. Create this file:

`migrations/00001_create_zookeepers.sql`:

```sql
-- +goose Up
CREATE TABLE zookeepers (
    id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username      text UNIQUE NOT NULL,
    password_hash text NOT NULL,
    role          text NOT NULL DEFAULT 'keeper' CHECK (role IN ('admin', 'keeper')),
    created_at    timestamptz NOT NULL DEFAULT now(),
    updated_at    timestamptz NOT NULL DEFAULT now()
);

-- +goose Down
DROP TABLE zookeepers;
```

Notes:

- File names are `<number>_<snake_case_description>.sql`. **Each version number must appear exactly once** across the whole directory: two files numbered 00001 make goose refuse to run rather than guess which one you meant. goose applies them in number order and refuses to re-run one that is already applied. `rm -rf migrations/00001*` would not be enough to "unapply" one on a database that has it; use `docker compose down -v` to start over from empty.
- The annotations are goose's "which half is this?" markers, not SQL comments to be ignored: everything under `-- +goose Up` runs on apply, everything under `-- +goose Down` runs on rollback. A version file must contain both (or explicitly be up/down-only, which we will not need).
- `GENERATED ALWAYS AS IDENTITY` is the modern replacement for `SERIAL`: Postgres assigns monotonically increasing integers. We use plain integers for IDs throughout this tutorial: they make `curl` examples copy-pasteable. Real-world systems frequently use UUIDs; a wrap-up note covers that swap (and its non-obvious pgx scanning caveats).
- `timestamptz` is "timestamp with time zone" and is what you should always pick for time columns. It stores an instant; serialization details show up in Stage 7.
- `password_hash` is a lie in Stage 3 and becomes true in Stage 4. We keep the column fixed from the start so the schema is stable; the code does not hash until Stage 4, and that dishonesty is part of the lesson.

### 2.7 Verify: a ready database and a working command

```bash
docker compose up -d          # already up, keeps schema
go run ./cmd/zoo migrate
docker exec $(docker compose ps -q db) psql -U zoo -d zoo -c '\dt'
```

A quick tour of the command line you just built, before the database:

```bash
go run ./cmd/zoo            # the root command's help (cobra also adds "completion" and "help")
go run ./cmd/zoo migrate -h # the subcommand's own help
go run ./cmd/zoo migrate --config nope.yaml
# 2026/10/09 10:15:02 ERROR command failed error="read config file: open nope.yaml: no such file or directory"
go run ./cmd/zoo migrate
# time=2026-10-09T10:15:02.123+01:00 level=INFO msg="OK   00001_create_zookeepers.sql (5.96ms)"
# time=2026-10-09T10:15:02.124+01:00 level=INFO msg="goose: successfully migrated database to version: 1"
# time=2026-10-09T10:15:02.125+01:00 level=INFO msg="migrations applied" dir=migrations
LOG_FORMAT=json go run ./cmd/zoo migrate
# {"time":"2026-10-09T10:15:02.456+01:00","level":"INFO","msg":"goose: no migrations to run. current version: 1"}
# {"time":"2026-10-09T10:15:02.457+01:00","level":"INFO","msg":"migrations applied","dir":"migrations"}
```

Everything after `RunE` returns is cobra's and slog's doing: the flag parsing, the help text, the `Use`/`Short` strings, the exit code, the log format. The tutorial will not explain them again. Note the two different time formats in that output: the first line is the *default* `slog` handler, which is what `Execute()` still has when `config.Load` fails before `logging.Setup` has run, and the rest are the `TextHandler` installed by 2.4, which writes RFC 3339.

That `docker exec` line is dense on a first meeting with Docker and with Postgres, so here it is pulled apart, left to right.

- **`docker exec`** runs a *new* command inside a container that is already running. A container is not a machine you log into; it is a process (here, the Postgres server), and `exec` starts a second process beside it, sharing the same filesystem namespace and network. Contrast `docker run`, which starts a *new* container. `exec` only works while the container's main process is alive - if the db container is stopped, there is nothing to exec into.
- **Why the one-shot commands do not say `-it`.** Those are two flags: `-i` keeps STDIN open (so you can type into the command) and `-t` allocates a pseudo-terminal (so the command believes it is attached to a real terminal, with a prompt and line editing). Together they make an interactive session possible - and they make it *mandatory*: Docker refuses to attach a TTY when stdin is a pipe, so a scripted or piped run of `docker exec -it ...` fails with `cannot attach stdin to a TTY-enabled container because stdin is not a terminal`. Every `-c` query in this tutorial is therefore spelled without them, which is also what makes those lines safe to drop into a script. The interactive session a few lines down is the one place `-it` earns its keep.
- **`$(docker compose ps -q db)`** is shell *command substitution*, not Docker syntax: the shell runs the inner command and splices its output into the outer one as text. `docker compose ps` lists the containers in this Compose project; `-q` ("quiet") prints only container IDs instead of a table; the trailing `db` filters to the service named `db` in `docker-compose.yml`. So the substitution means "whatever ID this project's `db` container currently has" - which is why the command keeps working after a `docker compose down && up -d` cycle hands the container a different ID. Compose also has a shorthand that skips the substitution entirely:

  ```bash
  docker compose exec db psql -U zoo -d zoo -c '\dt'
  ```

  `docker compose exec <service>` resolves the service for you. Both forms behave identically; this tutorial spells out the ID form because it makes clear *which* container is being entered.
- **`psql`** is Postgres's command-line client: the program that connects to a server and sends it SQL. It is not Docker-specific and not something we are inventing - it ships inside the `postgres` image, which is the only reason `exec` can run it. If you install `psql` on your host instead, you would connect over TCP with `-h localhost -p <the host port mapped in docker-compose.yml>` and, per the auth rule below, a password.
- **`-U zoo`** is the database *role* to connect as; **`-d zoo`** is the *database* to connect to. They match `POSTGRES_USER` and `POSTGRES_DB` in the compose file. Those are genuinely different things that happen to share the name `zoo` here: one is a login identity, the other is a named collection of tables. `-U` is required - without it psql defaults to your operating-system username, and no such role exists in this database.
- **Why no password is asked.** Inside the container psql connects over the local Unix socket (no `-h` means "use the socket"), and the image's `pg_hba.conf` contains `local all all trust`: socket connections are admitted with no password. Network connections follow a different rule, `host all all all scram-sha-256`, which is why the app's `DATABASE_URL` carries a password and this command does not.
- **`-c '\dt'`** means "run this one command string, then exit" instead of dropping into an interactive prompt. The string is a **meta-command**: psql shorthand beginning with a backslash, not SQL. `\dt` is "describe tables" - list them. Two constraints: a `-c` argument must be *either* pure SQL *or* a single backslash command, never a mixture; and bare `\dt` lists only objects **visible in your schema search path**, which for this connection is `public`. A table in some other schema would exist but not appear. `\dt *.*` lists every schema including Postgres's own catalogs - a hundred-odd rows on the container we checked, nearly all of them internals you will never touch. Your own count will differ by however many tables you have created, which is worth knowing before you use a number like that as a check.
- **The exit status propagates**: if the command inside fails, `docker exec` exits nonzero as well, so this shape is safe to use in scripts. (`docker exec <id> false` exits 1.)

Drop the `-c` and the command becomes the interactive shell, which is where you will want to spend real time poking around:

```bash
docker exec -it $(docker compose ps -q db) psql -U zoo -d zoo
```

Expected output from the `\dt` command:

```
             List of relations
 Schema |       Name       | Type  | Owner
--------+------------------+-------+-------
 public | goose_db_version | table | zoo
 public | zookeepers       | table | zoo
(2 rows)
```

`goose_db_version` is goose's own bookkeeping table - the record of which migration versions have been applied. It is why the tool is idempotent: on the next run it reads this table, sees version 1, and does nothing.

Run `go run ./cmd/zoo migrate` a second time: it applies nothing and reports it (`goose: no migrations to run. current version: 1` in current goose). That is the mark of a healthy migration tool: idempotent at the command level. The API server (`go run .`) still behaves exactly like Stage 1 - animals in memory, nothing database-backed. Check `http://localhost:8080/healthz` responds as before.

---

[Stage 1](01-official-port-flat.md)  |  [Overview](../tutorial.md)  |  [Stage 3](03-zookeeper-domain-crud.md)
