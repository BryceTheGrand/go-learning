## Stage 4: Real passwords and login: bcrypt + JWT + middleware

Stage 3 leaves one deliberate disaster: raw passwords sitting in `password_hash`. Fix it now. This stage delivers the "auth of a standard form" part of the tutorial: `POST /api/v1/login` exchanges username/password for a JWT; zookeeper reads require a valid bearer token.

The auth packages need two new dependencies (bcrypt lives in Go's extended-standard-library repo, jwt in its own), again pinned to exact versions:

```bash
go get github.com/golang-jwt/jwt/v5@v5.3.1 golang.org/x/crypto@v0.33.0
```

As in Stage 2.2, there is no `go mod tidy` on that line: nothing imports these two packages until the files below exist, and `tidy` would remove them again. The tidy call comes at the end of 4.2, once `password.go` and `token.go` have something to keep.

Files of this stage live in a new platform package, `internal/platform/auth/`: one directory, three files, all `package auth`. Since the reader of this stage has now written two packages, each file again arrives piece by piece with the explanations, complete file at the end.

### 4.1 Passwords: platform/auth/password.go

This file is two functions and a policy note: how passwords become stored strings, and how a login attempt is checked.

#### The package clause and what bcrypt is

```go
package auth

import (
	"fmt"

	"golang.org/x/crypto/bcrypt"
)
```

- A brand-new package name: `auth`. From the root of the module, its import path (used by the service file below) will be `zoo/internal/platform/auth`.
- **`golang.org/x/crypto`** is Go's "extended standard library": Google-maintained crypto packages that are stable and widely used but versioned outside the core stdlib for release-cycle reasons. bcrypt comes from there, under the sub-path `/bcrypt`. Import paths can point at a *subdirectory* of a repository - `x/crypto` is the repo, `/bcrypt` one package inside it.

#### HashPassword: string to hash

```go
// HashPassword stores a password as a bcrypt hash. Note what is not here:
// a salt. bcrypt generates and embeds a random salt in every hash; hashing
// "hunter2" twice produces two different hashes, and that is correct.
//
// Worth knowing in passing: bcrypt only reads the first 72 bytes of its
// input (a historical limit the format cannot escape). x/crypto used to
// silently truncate; the current version instead rejects passwords longer
// than 72 bytes with ErrPasswordTooLong, so HashPassword surfaces that as
// an error rather than hashing a prefix of the real password. Modern
// designs (argon2, scrypt) do not have this limit; bcrypt remains the
// standard choice for its ubiquity and maturity.
func HashPassword(password string) (string, error) {
	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return "", fmt.Errorf("hash password: %w", err)
	}
	return string(hash), nil
}
```

- **`[]byte(password)`**: a *type conversion* between `string` and `[]byte` (a byte slice). Notable fact: conversion of a string to a byte slice **copies the bytes** (`[]byte(s)` is not a view over `s`'s memory). Crypto APIs want byte slices; strings win the storage argument in Go; this conversion is the bridge, and it appears in both functions here.
- **`bcrypt.GenerateFromPassword(clearBytes, cost) -> ([]byte, error)`** does the actual hashing. `bcrypt.DefaultCost` is 10, meaning roughly 100ms worth of work - bcrypt deliberately makes the operation slow, so that mass guessing against a leaked database stays expensive. Higher cost = slower = more resistant, at the price of real latency; default is the sane choice for most services.
- The comment block is doing real work: **bcrypt embeds a random salt per hash** (no separate salt column exists in our schema, correctly), and the **72-byte input limit** is a property of the format you cannot engineer away. Note the current behavior chosen deliberately: reject overly long passwords rather than silently truncate them - a silent truncation means a user's password 73 is stored as its prefix 72, a subtle correctness bug with real security taste.

#### CheckPassword: hash plus try, without comparing yourself

```go
// CheckPassword reports whether password hashes to hash. It uses constant
// time comparison internally; never compare hashes yourself with ==.
func CheckPassword(hash, password string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

- **`CompareHashAndPassword`** re-hashes the candidate with the parameters recorded *inside* the stored hash string (`$2a$10$...`: version, cost, salt) and compares in constant time. Its error is `nil` exactly on a match - so the wrapper `... == nil` turns the (goes-without-saying) two-value API into the clean boolean the service wants. This is a very Go move for a helper: let a library function and `== nil` replace hand-rolled logic.
- **`bool`** is Go's boolean type, and notice what CheckPassword *chooses not to return*: no details of why authentication failed. Later, `Authenticate` in the service returns the same error for unknown username and wrong password - the security reason is spelled out there.

#### The complete password.go

```go
package auth

import (
	"fmt"

	"golang.org/x/crypto/bcrypt"
)

// HashPassword stores a password as a bcrypt hash. Note what is not here:
// a salt. bcrypt generates and embeds a random salt in every hash; hashing
// "hunter2" twice produces two different hashes, and that is correct.
//
// Worth knowing in passing: bcrypt only reads the first 72 bytes of its
// input (a historical limit the format cannot escape). x/crypto used to
// silently truncate; the current version instead rejects passwords longer
// than 72 bytes with ErrPasswordTooLong, so HashPassword surfaces that as
// an error rather than hashing a prefix of the real password. Modern
// designs (argon2, scrypt) do not have this limit; bcrypt remains the
// standard choice for its ubiquity and maturity.
func HashPassword(password string) (string, error) {
	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return "", fmt.Errorf("hash password: %w", err)
	}
	return string(hash), nil
}

// CheckPassword reports whether password hashes to hash. It uses constant
// time comparison internally; never compare hashes yourself with ==.
func CheckPassword(hash, password string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

This file is stack-independent: nothing in it knows whether the caller is gin, chi, or a command-line tool, and that is deliberate. Passwords are a *platform* concern, and the `internal/platform/` directory is where this project keeps exactly those.

### 4.2 Tokens: platform/auth/token.go

The JWT half. What a JWT *is* in one paragraph, then the file: a signed string with three base64 sections (header, payload, signature) split by dots - `eyJhbGciOi...` in every curl below. The payload is *readable by anyone* (base64, not encrypted) but *unforgeable* (the last section is a signature only your secret can produce). "Who are you" rides in the payload; "can you prove it" rides in the signature. Never put secrets like password hashes in there.

#### Claims: the payload as a struct

```go
package auth

import (
	"errors"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// Claims is our payload: who the caller is, plus enough to authorize them.
// ZookeeperID goes in so handlers never need a database lookup to know
// who is calling; that is a trade (staleness), noted in Stage 8.
type Claims struct {
	ZookeeperID int64  `json:"zookeeper_id"`
	Username    string `json:"username"`
	Role        string `json:"role"`
	jwt.RegisteredClaims
}
```

- Three ordinary fields with JSON tags, and then a line with no field name: **`jwt.RegisteredClaims`**. This is the third big Go struct trick, **embedding** (Stage 3 had tagged fields and pointer fields): naming a type without a field name *inlines* that type - Claims *has* everything RegisteredClaims has, and its field names (`Subject`, `ExpiresAt`, ...) are usable as if they were declared directly (`claims.Subject`, `claims.ExpiresAt`). When serialized to JSON the embedded struct's fields appear *flattened* at the same level - exactly how JWT libraries like their payload shapes. Used with restraint, embedding gives you "that standard set of fields, plus mine" without repetitive copy-paste; overusing it hurts readability, and much Go code avoids it outside cases like this where a base structure is genuinely shared.
- Why put `ZookeeperID` in the token at all: the middleware verifies the token once and *hands the parsed identity to handlers* (4.3 below); those handlers never pay a database round trip just to know whether the caller exists. The trade: a role change or a delete does not retroactively invalidate an already-issued token - Stage 8 spells that out.

#### The shared error and IssueToken

```go
// ErrInvalidToken is the single error VerifyToken surfaces; the middleware
// maps everything (bad signature, expired, malformed) to one 401.
var ErrInvalidToken = errors.New("invalid token")

// IssueToken signs a Claims set with an HMAC (HS256) using secret.
// Production systems often move to asymmetric signing (RS256/Ed25519) so
// other services can verify tokens without holding the secret; HS256 with
// a moderate TTL is the standard single-API starting point.
func IssueToken(secret []byte, ttl time.Duration, id int64, username, role string) (string, error) {
	now := time.Now().UTC()
	claims := Claims{
		ZookeeperID: id,
		Username:    username,
		Role:        role,
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   username,
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(ttl)),
		},
	}

	signed, err := jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString(secret)
	if err != nil {
		return "", fmt.Errorf("sign token: %w", err)
	}
	return signed, nil
}
```

- **`time.Duration`** appears as an argument type for the first time: Go's duration type. A duration *is* an int64 of nanoseconds whose readability comes from the constants: `24 * time.Hour`, `5 * time.Minute` - you never write raw nanoseconds. It arrives in `Config` in 4.4 as `TokenTTL`, read from a `TOKEN_TTL` environment variable that viper coerces from text into this type.
- **`now := time.Now().UTC()`** and **`now.Add(ttl)`**: instants in, instants out. Using UTC deliberately: tokens are parsed again somewhere else (same server, but the principle holds) and a UTC stamp has no offset to guess about. `jwt.NewNumericDate` converts to the JWT-standard "seconds since epoch" representation, and `time.Time` in `Claims` fields... is not what you see: the library wants *these* numeric dates for the registered fields.
- **`jwt.NewWithClaims(m, claims).SignedString(secret)`** chained: build a token object with an algorithm and payload, then sign it into the string the API returns. Note the shape of the failure branch - the Stage 3 lesson applies unchanged: `return "", fmt.Errorf(...)` with the `%w` wrap.
- HS256 ("HMAC with a secret shared by signer and verifier") versus RS256/Ed25519 ("sign with a private key, verify with a public one"): for one API holding one secret, HS256 is the mainstream, simplest, correct choice. That decision is written here so you *know it was a decision*: you will meet codebases that switched for cross-service reasons.

#### VerifyToken: the defensive half

```go
// VerifyToken parses raw, checks the signature was made by our secret, and
// validates standard fields like expiry. The keyFunc double-checks the
// algorithm: without it, an attacker can hand us a token we will happily
// verify against the wrong key (the classic JWT confusable-algorithm bug).
func VerifyToken(secret []byte, raw string) (Claims, error) {
	claims := Claims{}
	token, err := jwt.ParseWithClaims(raw, &claims, func(_ *jwt.Token) (any, error) {
		return secret, nil
	}, jwt.WithValidMethods([]string{jwt.SigningMethodHS256.Alg()}))
	if err != nil || !token.Valid {
		return Claims{}, ErrInvalidToken
	}
	return claims, nil
}
```

- **`jwt.ParseWithClaims(tokenString, &claims, keyFunc, options...)`**: parse and verify in one call. Note `&claims` - the pass-the-pointer-though pattern from Stage 1's binding: the library *writes into* your struct through the pointer. The `keyFunc` argument is a **function value passed inline** (Stage 1 taught function values as *named* things like `getAnimals`; here it is an anonymous function literal, written in place). Its job: answer "what key verifies this token?" - we return our shared secret. It receives the parsed token header as `_ *jwt.Token`, discarding it, because we ignore it - see the security point below.
- **`jwt.WithValidMethods([]string{...})`**: option pattern (a config-for-a-function-call, a widespread Go library idiom: `With...` functions that return options). It says "accept only tokens signed HS256". The security point is real and worth remembering: JWT headers announce their own algorithm; a verifier that trusts the announced algorithm accepts tokens made with someone else's key choice (the historical "alg confusion" attack). Declaring the allowed set turns that from attacker-chosen into configuration.
- **`if err != nil || !token.Valid { return Claims{}, ErrInvalidToken }`**: `||` short-circuiting and the single-sentinel decision. Whatever went wrong - expired, wrong signature, malformed - the *caller-visible* answer is one error, `ErrInvalidToken`. Why collapse: all of these mean "do not trust this request", and their differences are useless (and slightly dangerous) to reveal. The middleware below maps it to exactly one 401.
- The return value is `(Claims, error)` - the *claims*, not the token: consumers rarely want proof of verification, they want the identity that survived it.

#### The complete token.go

```go
package auth

import (
	"errors"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// Claims is our payload: who the caller is, plus enough to authorize them.
// ZookeeperID goes in so handlers never need a database lookup to know
// who is calling; that is a trade (staleness), noted in Stage 8.
type Claims struct {
	ZookeeperID int64  `json:"zookeeper_id"`
	Username    string `json:"username"`
	Role        string `json:"role"`
	jwt.RegisteredClaims
}

// ErrInvalidToken is the single error VerifyToken surfaces; the middleware
// maps everything (bad signature, expired, malformed) to one 401.
var ErrInvalidToken = errors.New("invalid token")

// IssueToken signs a Claims set with an HMAC (HS256) using secret.
// Production systems often move to asymmetric signing (RS256/Ed25519) so
// other services can verify tokens without holding the secret; HS256 with
// a moderate TTL is the standard single-API starting point.
func IssueToken(secret []byte, ttl time.Duration, id int64, username, role string) (string, error) {
	now := time.Now().UTC()
	claims := Claims{
		ZookeeperID: id,
		Username:    username,
		Role:        role,
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   username,
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(ttl)),
		},
	}

	signed, err := jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString(secret)
	if err != nil {
		return "", fmt.Errorf("sign token: %w", err)
	}
	return signed, nil
}

// VerifyToken parses raw, checks the signature was made by our secret, and
// validates standard fields like expiry. The keyFunc double-checks the
// algorithm: without it, an attacker can hand us a token we will happily
// verify against the wrong key (the classic JWT confusable-algorithm bug).
func VerifyToken(secret []byte, raw string) (Claims, error) {
	claims := Claims{}
	token, err := jwt.ParseWithClaims(raw, &claims, func(_ *jwt.Token) (any, error) {
		return secret, nil
	}, jwt.WithValidMethods([]string{jwt.SigningMethodHS256.Alg()}))
	if err != nil || !token.Valid {
		return Claims{}, ErrInvalidToken
	}
	return claims, nil
}
```

Like `password.go`, this file is stack-independent: it signs and verifies strings, and no HTTP type appears in it.

Both packages are now imported by real code, so the deferred tidy can run:

```bash
go mod tidy
go build ./...
```

### 4.3 Auth middleware: platform/auth/middleware.go

A chi middleware is an ordinary function with the signature `func(http.Handler) http.Handler`: it receives the *next* handler in the chain and returns a handler that wraps it - run its own logic, then decide whether to call next. That is the standard library's middleware type, and chi's `router.Use` and `router.Group` take values of exactly that type and nothing else. Configuration is provided via a closure, the same way `healthHandler(pool)` took its pool in Stage 3: `AuthMiddleware(secret)` *returns* a middleware that captured `secret`. This is the shape to recognize across all Go middleware.

#### The constant and the middleware itself

```go
package auth

import (
	"context"
	"encoding/json"
	"log/slog"
	"net/http"
	"strings"
)

// claimsKey is the unexported struct type AuthMiddleware uses as the request
// context key for the verified claims. gin's version of this file used a
// string key ("zoo.claims"); an unexported type is strictly better, because
// no other package can name auth.claimsKey, so no other package can ever
// collide with - or overwrite - the value we stash under it.
type claimsKey struct{}
```

- **The type replaces the string constant.** Under gin the key was `const claimsKey = "zoo.claims"` and it was *conventional*: any other package that wrote `c.Set("zoo.claims", ...)` would silently clobber us, and nothing but discipline prevented it. Because an unexported identifier is unreachable from outside its package, `claimsKey{}` is a key nobody else can construct. It costs one line and removes a whole class of collisions.
- `struct{}` is Go's **empty struct**: a type with no fields and, in this position, no data. `claimsKey{}` is the only value of its type, which is fine - context keys are compared by type-and-value, and all we need is a value no one else has.
- The type is *unexported* and used only as a value here; handlers never name it, they go through `ClaimsFrom` below. That is the point: the key is an implementation detail of this package.

```go
// AuthMiddleware requires a valid bearer token and stores the claims in the
// request context.
func AuthMiddleware(secret []byte) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			raw, ok := strings.CutPrefix(r.Header.Get("Authorization"), "Bearer ")
			if !ok {
				// No Abort call is needed: returning without calling next
				// *is* chi's rejection. See the note below.
				writeError(w, http.StatusUnauthorized, "missing or malformed Authorization header")
				return
			}

			claims, err := VerifyToken(secret, raw)
			if err != nil {
				writeError(w, http.StatusUnauthorized, "invalid token")
				return
			}

			ctx := context.WithValue(r.Context(), claimsKey{}, claims)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}
```

- **The signature**: `func AuthMiddleware(secret []byte) func(http.Handler) http.Handler` - a function that returns a *middleware*, which is itself a function that returns a handler. You met the outer half of this shape in Stage 3's `healthHandler(pool)`; here it earns its name, **a middleware factory**: call it once with the secret at wiring time, and the returned closure (a function bundled with the `secret` it captured) is what `router.Use` installs. That is also *why* wiring in 4.5 says `auth.AuthMiddleware([]byte(cfg.JWTSecret))` - with parentheses, calling the factory, not passing it. Three nested functions look odd the first time; read them inside out: `http.HandlerFunc` is the handler, the middle function turns a handler into a wrapped handler, the outer one closes over the secret.
- **`strings.CutPrefix(s, prefix)`**: a newer stdlib function that splits a string into (rest, had-prefix) - here, (the token's raw text, whether the `Authorization` header actually said `Bearer `). It is the tidy multi-value form of "hasPrefix, then trim". `c.GetHeader(...)` became `r.Header.Get(...)`: with chi there is no request wrapper, so the header read is the plain `net/http` one - the same method the standard library server would give you with no framework at all.
- **Rejection, and what happened to `Abort`.** The gin version wrote `c.AbortWithStatusJSON(...)` because a plain JSON write inside gin middleware did *not* stop the chain: the handler downstream still ran unless you remembered to mark the context aborted. Every gin codebase has the scar of "responded but forgot to abort". chi has no such trap, because there is nothing to abort: a middleware *is* the chain. It calls `next.ServeHTTP(w, r)` when the request may proceed, and it simply **returns** when it may not. This is worth saying out loud, because it is a real simplification and not a translation: deleting `Abort` deletes a class of bugs rather than moving them somewhere else.
- **`context.WithValue(r.Context(), claimsKey{}, claims)`** and **`next.ServeHTTP(w, r.WithContext(ctx))`** replace `c.Set`. The claims ride in the *request's* context - the very `context.Context` the handlers already take out with `r.Context()` (Stage 3), and the one the repositories hand to pgx. Note `r.WithContext(ctx)`: it returns a *shallow copy* of the request carrying the new context; requests are not mutated in place, which is what makes the copy necessary and what keeps one request's values from leaking into another's. That copy is what the rest of the chain sees.
- **The stash is gone.** gin's `c.Set`/`c.Get` was an in-memory map living exactly as long as the request, shared between middleware and handler. Go's request context is that same "lives exactly as long as the request" scope, with better typing and no framework object to thread around - which is why the pairing `WithValue`/`Value` replaces it.

#### ClaimsFrom: reading it back out, with a type assertion

```go
// ClaimsFrom returns the identity of the caller, when the request passed
// through AuthMiddleware.
func ClaimsFrom(ctx context.Context) (Claims, bool) {
	claims, ok := ctx.Value(claimsKey{}).(Claims)
	return claims, ok
}
```

- **`ctx.Value(claimsKey{})`** returns `any` - a context stores anything - so the result must be *narrowed* back to the type we put in. That is the **type assertion** `.(Claims)`: "I claim it is actually a `Claims`". Assertions come in two forms. One-value form (`claims := v.(Claims)`) *panics at runtime* if the value is not a Claims; the **comma-ok form** returns `(value, didItWork)` and never panics. In Go libraries you will read, comma-ok assertions are the norm at trust boundaries exactly like this one. Here the two facts collapse into a single line, because an absent key and a wrong type both mean "this request did not pass through the middleware" - exactly the question the caller is asking.
- **The design point in the name**: `ClaimsFrom` returns `(Claims, bool)`, the second standard return-pair from Stage 3's `parseID` (the `(value, ok)` shape, when the helper already handled the failure internally). Handlers will call it as `claims, ok := auth.ClaimsFrom(r.Context())`, check `ok`, and 401 when the request was not really authenticated - the "did middleware run?" question answered without global state. Note the argument: a plain `context.Context`, not a request and not a framework object, which is why this helper works identically inside a handler, a repository, or a test.
- `ClaimsFrom` is written here, in the middleware's file, though nothing in this stage calls it yet - Stage 7's handlers are the first. It belongs beside the code that stored the value, because the two must agree on the key.

#### The complete middleware.go

```go
package auth

import (
	"context"
	"encoding/json"
	"log/slog"
	"net/http"
	"strings"
)

// claimsKey is the unexported struct type AuthMiddleware uses as the request
// context key for the verified claims. gin's version of this file used a
// string key ("zoo.claims"); an unexported type is strictly better, because
// no other package can name auth.claimsKey, so no other package can ever
// collide with - or overwrite - the value we stash under it.
type claimsKey struct{}

// AuthMiddleware requires a valid bearer token and stores the claims in the
// request context.
func AuthMiddleware(secret []byte) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			raw, ok := strings.CutPrefix(r.Header.Get("Authorization"), "Bearer ")
			if !ok {
				// No Abort call is needed: returning without calling next
				// *is* chi's rejection. See the note below.
				writeError(w, http.StatusUnauthorized, "missing or malformed Authorization header")
				return
			}

			claims, err := VerifyToken(secret, raw)
			if err != nil {
				writeError(w, http.StatusUnauthorized, "invalid token")
				return
			}

			ctx := context.WithValue(r.Context(), claimsKey{}, claims)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// ClaimsFrom returns the identity of the caller, when the request passed
// through AuthMiddleware.
func ClaimsFrom(ctx context.Context) (Claims, bool) {
	claims, ok := ctx.Value(claimsKey{}).(Claims)
	return claims, ok
}

// writeError writes the flat {"error": "..."} body this stage uses. It is
// a stopgap: this package cannot reach the flat main package's writeJSON, so
// it borrows the same three lines. Stage 9 replaces it with the shared error
// pipeline (httpx.Respond) and deletes it.
func writeError(w http.ResponseWriter, status int, message string) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(map[string]string{"error": message}); err != nil {
		slog.Error("write error response", "error", err)
	}
}
```

**Why does this package write its own JSON?** Because it cannot see `main.go`. `writeJSON` and `decodeJSON` live in `package main` at the repository root, and a Go package can use only what it imports - `package auth` importing `package main` is not merely bad style, it is forbidden (a program package is not importable). So this file carries its own four-line copy, `writeError`, and the duplication is the honest cost of the flat layout: the helpers are in the wrong place, and this is the first file that notices. Stage 5 moves them to `internal/platform/httpx` where every package can see them, and Stage 9 replaces the flat shape entirely, at which point `writeError` is deleted rather than fixed.

### 4.4 The three edits to the zookeeper domain

**1. `config.go` grows the JWT settings.** The complete new version (the piecewise explanations of Stage 2 still apply; the new lines carry their own comments):

```go
package config

import (
	"errors"
	"fmt"
	"log/slog"
	"os"
	"time"

	"github.com/spf13/viper"
)

// Config is the process's configuration, parsed once at startup.
type Config struct {
	Port        string
	DatabaseURL string
	JWTSecret   string
	TokenTTL    time.Duration
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
	v.SetDefault("token_ttl", "24h")
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

	cfg := Config{
		Port:        v.GetString("port"),
		DatabaseURL: v.GetString("database_url"),
		JWTSecret:   v.GetString("jwt_secret"),
		TokenTTL:    v.GetDuration("token_ttl"),
		LogLevel:    level,
		LogFormat:   v.GetString("log_format"),
	}

	if cfg.JWTSecret == "" {
		return Config{}, errors.New("jwt_secret is required: set the JWT_SECRET environment variable")
	}

	return cfg, nil
}

// fileExists reports whether path names a regular file.
func fileExists(path string) bool {
	info, err := os.Stat(path)
	return err == nil && !info.IsDir()
}
```

New lines, spelled out:

- **Two fields join the struct**: `JWTSecret string` and `TokenTTL time.Duration`. This is the duration Stage 2.2 promised would arrive, and the moment the error slot in `Load`'s return stops being speculative - the "why does a function that cannot fail today return `(Config, error)`?" question now has its answer: because a missing secret must be a startup crash, and changing a return signature later means editing every caller.
- **`v.SetDefault("token_ttl", "24h")`** is a default for the new setting, in the same three-line layer as the others. Note there is deliberately **no** `v.SetDefault("jwt_secret", ...)`: a defaulted secret is precisely the "silently serves with a weak default" failure the check below exists to prevent, and a default would make that check unreachable. Defaults belong on settings that have a safe development value; this one does not.
- **`v.GetDuration("token_ttl")`** is the other half of the argument for viper, and Stage 2's note finally cashes in: environment variables are always strings, and viper coerces `"24h"` into a `time.Duration` for you, so `TOKEN_TTL=90m` or `TOKEN_TTL=1h30m` work with no parsing call at the use site. `v.GetString` could never have produced a `time.Duration`; this is what the typed getter is for.
- **`cfg.JWTSecret = v.GetString("jwt_secret")`** reads `JWT_SECRET` (viper upper-cases the key, Stage 2.2's convention). This is the first *required* setting in the tutorial, and the check below it is the whole "fail-fast config" chain from Stage 2 doing its job: missing -> `errors.New(...)` describing the fix -> `Load` fails -> `run` fails -> `main` prints and exits nonzero. An app that silently serves with a weak default is worse than one that refuses to boot.
- The check runs *after* the struct is built (`cfg := Config{...}`, not a `return Config{}, ...` literal assembled at the end). That ordering is not cosmetic: it lets the error message name the actual setting, and it keeps the success path a single `return cfg, nil`. The failure path still hands back the zero value `Config{}` plus the error, never a half-filled struct.
- **`errors.New` in config** (versus the sentinel pattern of the auth package): here it is constructed at the *only possible* place, its message is addressed to the operator, and nobody ever needs to test for it programmatically; a named constant would add ceremony without a consumer. Both are correct uses; the difference is whether anyone will ever *inspect* the error.

From now on, run the server with a secret:

```bash
export JWT_SECRET="a-long-random-string-for-dev-only-not-a-real-secret"
go run .
```

(`TOKEN_TTL` needs nothing: it defaults to `24h`. Set `TOKEN_TTL=5m` in an environment where you want to watch a token expire, and the login flow below starts rejecting with `invalid token` after five minutes - the single-sentinel design of `VerifyToken` means expiry is indistinguishable from any other rejection, which is the point.)

**2. `zookeepers_service.go`: hash on the way in, verify on the way through.** Three concrete edits:

- The import block becomes:

```go
import (
	"context"
	"errors"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"

	"zoo/internal/platform/auth"
)

var (
	errZookeeperNotFound  = errors.New("zookeeper not found")
	errZookeeperDuplicate = errors.New("username already taken")
	errZookeeperInvalid   = errors.New("username, password and role must be valid")
	errInvalidCredentials = errors.New("invalid credentials")
)
```

  New here: **the `auth` package import** (first import from inside `zoo/internal/...` - the platform is being used by a real domain) and the new sentinel `errInvalidCredentials`, one error for every failed login regardless of which half failed. Note also that `errZookeeperInvalid`'s message tightened from Stage 3's "must be non-empty" to "must be valid" - nothing in the code reads the message text (the handler writes its own status strings), so this is a wording cleanup, but keep your file matching what is shown here.

- In `Create`, replace the line that stores the raw password with a real hash:

```go
	hash, err := auth.HashPassword(password)
	if err != nil {
		return zookeeper{}, err
	}

	zk, err := s.repo.Create(ctx, zookeeper{
		Username:     username,
		PasswordHash: hash,
		Role:         role,
	})
```

  Note the mechanics: `hash, err :=` is a *reassignment* of `err` - Go's multi-variable `:=` requires at least one **new** variable on the left (`hash` is), and any second variable whose name already exists in the same scope is simply reused and assigned over. Both statements sit in the same function body, so there is one `err` here, reassigned twice; that is legal and utterly common, and it is why stale-`err` bugs are guarded against by always checking immediately after the assigning call. The *fresh-scope* behavior you have met already (`if err := rows.Err(); err != nil` in Stage 3's List loop, `if err := decodeJSON(r, &newAnimal); err != nil` in its handlers) belongs to `if`/`for`/`switch` blocks, not to plain statements. What is *not* in this snippet: any change to the repository - the SQL still says `INSERT INTO zookeepers (username, password_hash, ...)`. The service now hands it a hash instead of a raw password; from the database's point of view, nothing happened - which is exactly what "the layers did their job" means.

- Add `Authenticate`, the method login calls, at the bottom of the service (before `isUniqueViolation`):

```go
// Authenticate verifies a login attempt. Both a wrong username and a wrong
// password return the same error: a distinct "no such user" message would
// tell an attacker which half of their guess is right.
func (s *zookeeperService) Authenticate(ctx context.Context, username, password string) (zookeeper, error) {
	zk, err := s.repo.GetByUsername(ctx, username)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return zookeeper{}, errInvalidCredentials
		}
		return zookeeper{}, err
	}

	if !auth.CheckPassword(zk.PasswordHash, password) {
		return zookeeper{}, errInvalidCredentials
	}
	return zk, nil
}
```

  This method is *the* example of a security decision as business logic: username unknown (`pgx.ErrNoRows` -> `errInvalidCredentials`) and password wrong (`CheckPassword` false -> `errInvalidCredentials`) collapse into the identical error. The handler cannot do this translation - it does not know what `pgx.ErrNoRows` *means* here; the service does.

- And note: the same `Update` path now stores `COALESCE` results over an
  already-hashed value; if a PUT sends a new password, hash it in `Update`
  too:

```go
func (s *zookeeperService) Update(ctx context.Context, id int64, zku zookeeperUpdate) (zookeeper, error) {
	if zku.PasswordHash != nil {
		hash, err := auth.HashPassword(*zku.PasswordHash)
		if err != nil {
			return zookeeper{}, err
		}
		zku.PasswordHash = &hash
	}

	zk, err := s.repo.Update(ctx, id, zku)
	// ... the rest is unchanged
```

  Pointer mechanics on display: `if zku.PasswordHash != nil` means "a new password was sent" (`nil` = field absent, from Stage 3's rule); `*zku.PasswordHash` dereferences the pointer to get the actual string; hashing it produces a new string; `&hash` builds its pointer for the update struct, so what reaches `COALESCE` is the hash, never the clear text.

  Note what this does *not* touch: the handler. It still copies the password the request sent into `zookeeperUpdate` and stops there (the type is renamed `UpdateInput` in Stage 5, along with everything else that gains an exported name), which is now exactly right - the raw value travels one layer and is hashed by the layer that owns the rule. That leaves one loose end behind, though: delete the comment Stage 3 left on those two lines in `zookeepers_handler.go` (`// Stage 3 stores it raw; Stage 4 hashes here.`). It was a forward note when it was written and it is false now, and a comment that lies is worse than no comment.

**3. `zookeepers_repository.go`: one new method.** Add next to `Get`:

```go
func (r *zookeeperRepository) GetByUsername(ctx context.Context, username string) (zookeeper, error) {
	var zk zookeeper
	err := r.pool.QueryRow(ctx,
		`SELECT id, username, password_hash, role, created_at, updated_at
		 FROM zookeepers WHERE username = $1`, username).
		Scan(&zk.ID, &zk.Username, &zk.PasswordHash, &zk.Role, &zk.CreatedAt, &zk.UpdatedAt)
	if err != nil {
		return zookeeper{}, err
	}
	return zk, nil
}
```

Same `QueryRow` + `Scan` shape as `Get`, with the unique `username` column in the WHERE clause. Its caller in the service is `Authenticate`, which needs *the stored hash* - including the fact that the repository returns the full domain struct here (hash and all) is exactly why the repository's type is internal and never reaches an HTTP response.

**4. `zookeepers_handler.go`: login endpoint + issued tokens.** Three edits:

- The handler struct and constructor change to carry the token config:

```go
type zookeeperHandler struct {
	svc         *zookeeperService
	tokenSecret []byte
	tokenTTL    time.Duration
}

func newZookeeperHandler(svc *zookeeperService, secret string, ttl time.Duration) *zookeeperHandler {
	return &zookeeperHandler{svc: svc, tokenSecret: []byte(secret), tokenTTL: ttl}
}
```

  Two new constructor arguments, no new dependencies beyond that - the handler now issues tokens, so it carries the secret and the TTL as plain fields (both unexported: other packages do not reach into a handler). **`[]byte(secret)`** at construction time: convert once, at the boundary, rather than on every login request.

- Add the login method:

```go
func (h *zookeeperHandler) login(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Username string `json:"username"`
		Password string `json:"password"`
	}
	if err := decodeJSON(r, &req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": err.Error()})
		return
	}

	zk, err := h.svc.Authenticate(r.Context(), req.Username, req.Password)
	if err != nil {
		if errors.Is(err, errInvalidCredentials) {
			// Same message for unknown username and wrong password.
			writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "invalid credentials"})
			return
		}
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	token, err := auth.IssueToken(h.tokenSecret, h.tokenTTL, zk.ID, zk.Username, zk.Role)
	if err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
		return
	}

	writeJSON(w, http.StatusOK, map[string]any{"token": token, "zookeeper": zk.toResponse()})
}
```

  It is Stage 3's `create` shape again - handler signature, `decodeJSON`, `r.Context()`, `writeJSON`, flat error bodies - with the login rule in the middle. The one difference worth naming: this struct's `Username` and `Password` dropped their gin `binding:"required"` tags for the same reason Stage 3's did, and `Authenticate` covers what they used to check (an empty username cannot match a row, and `CheckPassword` never matches an empty password against a real hash), so the 401 branch handles it with no extra code.

  Note the response shape: a `map[string]any` with two keys - `"token"` and a nested `"zookeeper"` object built by the same `toResponse()` from Stage 3. (This is what `gin.H` always was: `gin.H` is literally `map[string]any`, and writing the type out needs no framework.) When responses start getting more shape than a map naturally has (Stage 9's envelope), a proper response struct replaces it; for a two-key, one-off shape the map is the honest tool.

(and add `"zoo/internal/platform/auth"` to the file's import block; the file already imports `github.com/go-chi/chi/v5` from Stage 3).

### 4.5 Wire it: changes to main.go

Inside `run`, replace the zookeeper wiring block with:

```go
	zkRepo := newZookeeperRepository(pool)
	zkSvc := newZookeeperService(zkRepo)
	zkHandler := newZookeeperHandler(zkSvc, cfg.JWTSecret, cfg.TokenTTL)
```

and the route block with (login joins the same `/api/v1` sub-router, so the whole block is shown again):

```go
	router.Route("/api/v1", func(router chi.Router) {
		router.Route("/zookeepers", func(router chi.Router) {
			router.Post("/", zkHandler.create)

			// Authenticated reads: this group's middleware runs for each of
			// its routes and no others. Unauthenticated creation is Stage 8's
			// problem.
			router.Group(func(router chi.Router) {
				router.Use(auth.AuthMiddleware([]byte(cfg.JWTSecret)))
				router.Get("/", zkHandler.list)
				router.Get("/{id}", zkHandler.get)
				router.Put("/{id}", zkHandler.update)
				router.Delete("/{id}", zkHandler.delete)
			})
		})

		router.Post("/login", zkHandler.login)

		router.Get("/animals", getAnimals)
		router.Get("/animals/{id}", getAnimalByID)
		router.Post("/animals", postAnimal)
	})
```

and extend `main.go`'s imports with `"zoo/internal/platform/auth"`.

- **`router.Group(func(router chi.Router) {...})`** is the replacement for `zk.Group("", middleware)`, and it is *smaller*: a chi group adds no prefix at all and is purely a middleware scope. `router.Use(...)` inside the block runs for every route in the block and no others, which is how one path prefix splits into protected and unprotected branches - `router.Post("/", ...)` sits above the group and never sees the middleware. (gin needed the empty-string prefix argument to say "same path, new middleware"; chi splits prefix from middleware by having two different methods, `Route` for a prefix and `Group` for middleware.)
- **`router.Use` order is execution order**, and middleware declared outside a group (like the plain `router.Post` above) never sees that group's middleware. Stage 8 stacks a second middleware on top of this one and follows the same rule; the ordering matters there, because the role check reads what the auth middleware stored.
- **`auth.AuthMiddleware([]byte(cfg.JWTSecret))`** - called, with its parentheses, exactly once at wiring: what `router.Use` receives is the returned middleware closure holding the secret. The `[]byte(...)` conversion happens here rather than per request, the same boundary-conversion habit the handler's constructor uses.
- Login is mounted on the `/api/v1` sub-router (no auth required - it is how tokens are obtained) and stays plain `POST /api/v1/login`, outside the zookeeper group, so nothing about the token check can accidentally apply to the route that mints tokens.

### 4.6 Verify: the login flow

Bcrypt-hashed passwords do not match raw text, so any row you created in Stage 3 is now dead weight. Delete everything and start clean (the schema is fine; the rows are stale). Use `TRUNCATE ... RESTART IDENTITY`, not `DELETE`: a plain `DELETE` frees the rows but does not rewind the identity counter, so your next users would get ids 4 and 5 and every later stage's example ids would silently lie:

```bash
docker exec $(docker compose ps -q db) \
  psql -U zoo -d zoo -c 'TRUNCATE zookeepers RESTART IDENTITY'
```

Restart the server with JWT_SECRET exported (see above), then:

```bash
# create two zookeepers (open for now; Stage 8 closes this)
curl -s -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"giraffe-lane","role":"admin"}' > /dev/null
curl -s -X POST http://localhost:8080/api/v1/zookeepers \
  -H "Content-Type: application/json" \
  -d '{"username":"sam","password":"elephant-road"}' > /dev/null

# no token -> 401
curl -i http://localhost:8080/api/v1/zookeepers
# HTTP/1.1 401 Unauthorized {"error":"missing or malformed Authorization header"}

# wrong password -> 401, same message as unknown username
curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"wrong"}'
# {"error":"invalid credentials"}

# successful login -> token
curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"giraffe-lane"}'
# {"token":"eyJhbGciOi...","zookeeper":{"id":1,"username":"maya","role":"admin","created_at":...}}

# store the token for later stages:
export TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/login \
  -H "Content-Type: application/json" \
  -d '{"username":"maya","password":"giraffe-lane"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')

# token unlocks reads
curl -i http://localhost:8080/api/v1/zookeepers -H "Authorization: Bearer $TOKEN"
# HTTP/1.1 200 OK [ ... ]
```

In the database, `password_hash` now starts with `$2a$...`: that is bcrypt.

---

[Stage 3](03-zookeeper-domain-crud.md)  |  [Overview](../tutorial.md)  |  [Stage 5](05-restructure-internal.md)
