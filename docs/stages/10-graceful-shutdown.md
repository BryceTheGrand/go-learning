## Stage 10: Graceful shutdown

Ctrl+C today kills the process wherever it is: mid-query, mid-response, mid-transaction. Graceful shutdown means: stop accepting new requests, finish the in-flight ones, close the pool, exit. This is the tutorial's only goroutine material, and it stays small on purpose.

Five concurrency ideas, all introduced here, none anywhere else in this tutorial:

- **`go f(x)`** starts `f` running concurrently and returns immediately. The function runs on a **goroutine** - Go's lightweight thread, scheduled by the runtime onto OS threads for you. There is no thread pool to manage, no class to extend; `go` in front of a call is the whole mechanism.
- **A channel** is how those concurrent pieces hand values to each other or wait for each other: a typed pipe where one side sends (`ch <- v`) and the other receives (`v := <-ch`). A receive *blocks* until a value arrives, which makes channels both a message queue and a synchronization point.
- **`make(chan error, 1)`** creates a channel of `error` values. The second argument is the *buffer size*: how many values the channel can hold before sends block. Buffered here means the goroutine can deliver its result and finish even if nobody has read it yet.
- **`select`** waits on several channel operations at once, taking whichever is *ready first*. It is Go's `poll`/`epoll` for your own code: "sleep until one of these things happens".
- **`signal.NotifyContext`**: a stdlib helper that turns OS signals (Ctrl+C is `SIGINT`; containers are stopped with `SIGTERM`) into cancellation of a context - wiring the kernel's world to the `context.Context` world Stage 7 introduced.

`cmd/apiserver/main.go` is replaced wholesale, and since it is the concurrency climax, it arrives one piece at a time.

#### main(): unchanged

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"zoo/internal/animals"
	"zoo/internal/platform/config"
	"zoo/internal/platform/database"
	"zoo/internal/server"
	"zoo/internal/zookeepers"
)

func main() {
	if err := run(); err != nil {
		slog.Error("startup failed", "error", err)
		os.Exit(1)
	}
}
```

Same `main`/`run` split Stage 2 taught. New imports in the list: `os/signal`, `syscall`, `net/http`, and `time` - all four earn their keep below.

#### Context from signals: the run() opener

```go
func run() error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("load config: %w", err)
	}

	pool, err := database.NewPool(ctx, cfg.DatabaseURL)
	if err != nil {
		return fmt.Errorf("open database: %w", err)
	}
	defer pool.Close()
```

- **`signal.NotifyContext(ctx, signals...)`** returns two values: a *derived* context (cancelled when one of the named signals arrives) and a `stop` function that unregisters the handler. **`ctx, stop := ...`** - the same two-value shape as `(value, error)`, but the second slot is a function instead of an error; Go results are heterogeneous.
- **`defer stop()`**: releasing the signal handler at function exit is the tidy form; this run() returns only when the server is done, so the placement is mostly ceremony (the process is exiting anyway) but costs nothing and matches the library's documented usage.
- The config/pool block is Stage 2 verbatim, with one real change: the pool is opened with the *signal-connected* context, so a shutdown signal even cancels in-flight startup work.

#### Wiring and the http.Server

```go
	zkHandler := zookeepers.NewHandler(
		zookeepers.NewService(zookeepers.NewRepository(pool)),
		cfg.JWTSecret, cfg.TokenTTL,
	)
	anHandler := animals.NewHandler(animals.NewService(animals.NewRepository(pool)))

	router := server.NewRouter(pool, cfg.JWTSecret, zkHandler, anHandler)

	// The http.Server carries the timeouts that a bare gin Run never set
	// for you. Slowloris-proofing your read paths is this easy; skip it
	// and a single hung client pins a goroutine per open connection.
	srv := &http.Server{
		Addr:         "localhost:" + cfg.Port,
		Handler:      router,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}
```

- Wiring unchanged from Stage 5.
- **`&http.Server{...}`**: plain struct literal (with `&` for the pointer, as constructors do). gin's `router.Run(...)` hid one of these from you - it builds a zero-timeout server, which is why this stage replaces it with an explicit one. The three timeouts are the standard defensive set: `ReadTimeout` caps how long a client may take to *send* a request (protection against slowloris-style slow-senders), `WriteTimeout` caps the response write, `IdleTimeout` caps how long an idle keep-alive connection may linger. Without any of these, one client that opens connections and never talks ties up a goroutine per connection, forever.
- **`Handler: router`**: any `*gin.Engine` is itself an `http.Handler` (a one-method interface from `net/http`: `ServeHTTP(w, r)`). This is the seam where gin plugs into the standard library - everything gin does is inside that one method call.

#### The goroutine, the channel, and the select

```go
	// ListenAndServe blocks until the server stops; run it concurrently.
	errCh := make(chan error, 1)
	go func() {
		errCh <- srv.ListenAndServe()
	}()

	slog.Info("api server listening", "addr", srv.Addr)

	// Blocked here until SIGINT/SIGTERM (or the server errored early).
	// When the signal arrives, NotifyContext cancels ctx.
	select {
	case err := <-errCh:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			return fmt.Errorf("listen: %w", err)
		}
	case <-ctx.Done():
		slog.Info("signal received, shutting down")
	}
```

This is the concurrency heart; read it in order:

- **`go func() { errCh <- srv.ListenAndServe() }()`** - the goroutine runs an *anonymous function* (a function literal, like the keyFunc in Stage 4's token.go) and the trailing `()` invokes it concurrently. `ListenAndServe` blocks until the server stops, which is why it must not run on the main path: run() still has work to do after (the shutdown).
- **`errCh <- ...`** sends the blocking call's eventual result down the channel. Buffer of 1: the goroutine completes its send even if run() has already moved on to the shutdown path below.
- **The two-arm `select`** blocks until exactly one arm is ready:
  - `<-errCh`: the server stopped *on its own* (port taken at startup, etc.). `http.ErrServerClosed` is the sentinel `ListenAndServe` returns after a well-behaved `Shutdown` - it is a normal-ish outcome, filtered out with the now-familiar `errors.Is`. Everything else is a real listen failure and fails the startup.
  - `<-ctx.Done()`: every cancellable context carries an internal done-channel, and `ctx.Done()` is how you read it - this receive unblocks exactly when Ctrl+C (or SIGTERM) has arrived and `NotifyContext` cancelled the context. Note the arm receives *a struct-like signal, not a value* - `case <-ctx.Done():` discards it with unary receive.
- This select *is* the graceful-shutdown design in miniature: whichever happens first - the server dying, or the operator asking it to die - run() finds out here, in one place.

#### The shutdown itself

```go
	// Finish in-flight requests, with a deadline that stops a hung one
	// from keeping the process alive forever.
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("shutdown: %w", err)
	}

	slog.Info("shutdown complete")
	return nil
}
```

- **`context.WithTimeout(parent, d)`**: derives a context that cancels itself after a duration. The parent here is deliberately `context.Background()` - this timeout starts *now*, during shutdown; it must not inherit a state the parent could cancel it from. The two-value return is the Stage 10 `ctx, stop` shape again; `defer cancel()` releases the timer's resources whether or not the timeout actually fired (the documented cleanup for every `WithTimeout`/`WithCancel`).
- **`srv.Shutdown(shutdownCtx)`**: stop accepting; wait for in-flight requests to finish; return. Its context is the deadline: if some request is still running when the 10 seconds lapse, `Shutdown` gives up and reports an error (context deadline exceeded) rather than hanging forever. Hung-request insurance, spelled in code.
- After `Shutdown` returns, `run` returns normally, and `defer pool.Close()` (declared way up top) runs last - so the pool closes *after* the server finished. The defer ordering quietly got the shutdown sequence right; no explicit ordering code was needed.

#### The complete main.go

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"zoo/internal/animals"
	"zoo/internal/platform/config"
	"zoo/internal/platform/database"
	"zoo/internal/server"
	"zoo/internal/zookeepers"
)

func main() {
	if err := run(); err != nil {
		slog.Error("startup failed", "error", err)
		os.Exit(1)
	}
}

func run() error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("load config: %w", err)
	}

	pool, err := database.NewPool(ctx, cfg.DatabaseURL)
	if err != nil {
		return fmt.Errorf("open database: %w", err)
	}
	defer pool.Close()

	zkHandler := zookeepers.NewHandler(
		zookeepers.NewService(zookeepers.NewRepository(pool)),
		cfg.JWTSecret, cfg.TokenTTL,
	)
	anHandler := animals.NewHandler(animals.NewService(animals.NewRepository(pool)))

	router := server.NewRouter(pool, cfg.JWTSecret, zkHandler, anHandler)

	// The http.Server carries the timeouts that a bare gin Run never set
	// for you. Slowloris-proofing your read paths is this easy; skip it
	// and a single hung client pins a goroutine per open connection.
	srv := &http.Server{
		Addr:         "localhost:" + cfg.Port,
		Handler:      router,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// ListenAndServe blocks until the server stops; run it concurrently.
	errCh := make(chan error, 1)
	go func() {
		errCh <- srv.ListenAndServe()
	}()

	slog.Info("api server listening", "addr", srv.Addr)

	// Blocked here until SIGINT/SIGTERM (or the server errored early).
	// When the signal arrives, NotifyContext cancels ctx.
	select {
	case err := <-errCh:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			return fmt.Errorf("listen: %w", err)
		}
	case <-ctx.Done():
		slog.Info("signal received, shutting down")
	}

	// Finish in-flight requests, with a deadline that stops a hung one
	// from keeping the process alive forever.
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("shutdown: %w", err)
	}

	slog.Info("shutdown complete")
	return nil
}
```

Note the shape of the changes around it: `server.NewRouter` is unchanged (it returns the engine; the `http.Server` wraps it), and the pool closes *after* `Shutdown` because deferred calls run after the function's body - the ordering is `defer pool.Close()` running last, which is exactly right.

### 10.1 Verify: clean exits, both kinds

```bash
go build ./... && go run ./cmd/apiserver
# time=... msg="api server listening" addr=localhost:8080

# in another terminal, request while shutting down: kill the server
# (find the pid you just stopped it with; simplest: Ctrl+C in terminal 1)
# then watch terminal 1 for:
# time=... msg="signal received, shutting down"
# time=... msg="shutdown complete"
```

To try SIGTERM (the signal process managers actually send), background a *built binary*: `go run` wraps the compiled child in its own wrapper process, and killing the wrapper leaves the real server listening, which muddies the experiment.

```bash
go build -o /tmp/zoo-apiserver ./cmd/apiserver
/tmp/zoo-apiserver &
sleep 2 && kill -TERM %1        # %1 is the background job in this shell
# ... "signal received, shutting down" ... "shutdown complete"
# and the process exits 0, not from a panic
```

---

[Stage 9](09-error-pipeline-logging.md)  ·  [Overview](../tutorial.md)  ·  [Stage 11](11-tests.md)