## Stage 1: The official tutorial, ported to a zoo (flat, in-memory)

Everything in this stage lives in one file, exactly like the official tutorial. Three differences: our domain is animals, we add a health-check endpoint you will rely on for the rest of the tutorial, and we use [chi](https://github.com/go-chi/chi) rather than gin.

That third one is worth a sentence before any code, because it shapes every stage after this one. Gin is a *framework*: it gives you its own `Context` type, its own binder, its own response helpers, and your handlers are gin-shaped functions that only gin can run. chi is a *router*: it matches URL patterns to handlers and does nothing else. A chi handler is an ordinary `func(http.ResponseWriter, *http.Request)` - the same type the standard library's own `net/http` server expects - and chi's middleware is the standard `func(http.Handler) http.Handler`. Swapping chi for the standard library's `http.ServeMux` later would change your routing lines and nothing else. That property is what people mean by platform-agnostic code, and it is why this tutorial reaches for chi.

### 1.1 Create the module

Your project directory must contain only this tutorial's docs when you start. If you find leftover files here - an empty `go.mod`, an empty `main.go` - remove them first: a `go.mod` that already exists makes `go mod init` fail.

```bash
rm -f go.mod main.go
go mod init zoo
```

`go mod init` creates `go.mod`, which declares this directory as a **module** named `zoo`. Two new words, so let us define both:

- A **module** is Go's unit of versioned, importable code: one directory tree with one `go.mod` at its root. When someone says "import my package", the compiler needs to know *which package* - and the module path is where that name starts.
- The **module path** (`zoo` here; usually a full URL like `github.com/you/zoo` in real projects) becomes the prefix of every import path inside the module. We use the short name `zoo` so import paths stay readable: Stage 5 will import `zoo/internal/animals`.

`go.mod` also records your Go version and (since modules, Go 1.11) every dependency with its pinned version. It is a file you mostly *read*; the tooling writes it.

### 1.2 Install chi

```bash
go get github.com/go-chi/chi/v5@v5.2.3
```

This downloads chi and adds it to `go.mod` and `go.sum` (the lockfile listing exact versions plus checksums). The `@v5.2.3` suffix pins an exact release; every dependency in this tutorial is pinned, so what you see printed here is what you get, and dropping the suffix would take whatever is newest instead.

One note on chi's import path: it ends in `/v5`. Go's **semantic import versioning** rule says that a module at major version 2 or higher puts the major version in its import path, so `github.com/go-chi/chi/v5` is the v5 line of chi, and its package name is still plain `chi` - you will write `chi.NewRouter()`, never `chi/v5.NewRouter()`.

### 1.3 The first main.go, one piece at a time

Go programs are built from **packages** (files that compile together and share their names) and functions. So that the first file does not arrive as a wall of code, we will build `main.go` top to bottom, one declaration at a time, and print the complete file at the end of this section.

#### The package clause and the imports

Every Go file starts with these two things:

```go
package main

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"strconv"

	"github.com/go-chi/chi/v5"
)
```

What each line means:

- **`package main`** names the package this file belongs to. A package is the compilation and naming unit: every file in the same directory must declare the same package name, and they can refer to each other's identifiers without imports. `main` is the one special name: a package named `main` must contain a `func main()`, and that function is what a built binary executes. Every other package in this project gets a real domain name (Stage 5).
- **`import "..."`** says "this file uses code from that package", and the quoted string is the *import path*. The two blank-line groups are pure convention: standard-library paths (`encoding/json`, `log/slog`, `net/http`, `strconv`) on top, third-party below. Both are real packages from real places - all four stdlib ones ship with Go, and `github.com/go-chi/chi/v5` is the dependency you installed in 1.2. When this file says `http.StatusOK` or `chi.NewRouter()`, the compiler resolves those names through these imports.
- Import paths and package names are different things: the path is where the code lives, the name is what you prefix identifiers with. `net/http`'s package *name* is `http`, which is why you write `http.StatusOK`, never `net/http.StatusOK`.
- **`encoding/json`** is here because with chi there is no binder: JSON encoding and decoding is something you do yourself, with the standard library. That sounds like more work and is in fact less magic; Stage 1's helpers below are the whole of it.
- **`log/slog`** is Go's structured logger, in the standard library since Go 1.21. Without configuration it prints a readable `time level msg key=value` line to stderr. Stage 2 installs a configured handler; for now the default is fine.

#### The data: a struct type

Go is a statically typed language - every value has a type known at compile time - and a **struct** is how you group related values under one name. Conceptually you know this from every language: it is a record. In Go it looks like this:

```go
// animal is the data we serve. The `json:"..."` tags set the field names
// encoding/json uses when serializing: without them you get Go's title-cased
// names (ID, Name, Species), which is not what JSON APIs usually do.
type animal struct {
	ID        int64  `json:"id"`
	Name      string `json:"name"`
	Species   string `json:"species"`
	Enclosure string `json:"enclosure"`
}
```

Line by line:

- **`type animal struct`** declares a new type named `animal`. Capitalization is the *only* access control Go has: names starting with an uppercase letter (like `ID`) are visible outside the declaring package; lowercase names (like `animal`) are private to it. Ours is lowercase because Stage 1's whole program is one package and nobody needs it from outside.
- **`ID int64`** is a field: name, then type. `int64` is a 64-bit integer. We deliberately choose the wide one because database ids (Stage 2 onward) are 64-bit in Postgres, and mixing integer widths is a classic Go annoyance.
- **`Name string`**: Go strings are UTF-8 text, plain and simple.
- **The backtick strings after each field are *struct tags*** - metadata you can attach to fields. They are ordinary constant strings; Go's compiler stores them, and libraries read them with reflection. Here it is `encoding/json`, which is part of the standard library, not part of chi. The tag `json:"id"` says "when this struct becomes JSON, write this field under the key `id`". Without tags you would get the Go field names (`ID`, `Name`) - which is rarely the JSON convention APIs use, hence tags on every field. In this tutorial every type that touches the network has tags; it is a habit worth copying.
- Note the fields themselves stay capitalized even though `animal` is lowercase. That is not cosmetic: Go's JSON encoder **ignores unexported fields entirely**. Lowercase struct type + uppercase fields is the standard shape for "my package's implementation detail, but serializable".

#### The data: a slice holding three animals

```go
// In-memory data, exactly like the official tutorial's slice of albums.
// Nothing here is persistent yet: every restart forgets your animals.
var animals = []animal{
	{ID: 1, Name: "Tembo", Species: "African bush elephant", Enclosure: "Savanna"},
	{ID: 2, Name: "Suki", Species: "Sumatran tiger", Enclosure: "Jungle"},
	{ID: 3, Name: "Biscuit", Species: "Red panda", Enclosure: "Forest"},
}
```

- **`var animals = ...`** declares a package-level variable: it exists for the whole program, and every function in the package can read (and write) it. That is usually something to be suspicious of in production code - which is exactly why Stage 2 throws it away.
- **`[]animal`** is a **slice**: Go's growable, array-backed list type. Square brackets with no size (unlike an array, `[4]animal`, whose length is part of its type) means "a sequence of animals". Slices are the every-day list in Go; the difference from arrays matters in Stage 6's database scans, but for now: `[]T` is the dynamic list of T, `len()` gives its length (used in 1.3's `postAnimal`), and `append` grows it.
- **`{ID: 1, Name: "Tembo", ...}`** is a *composite literal*: a value spelled with field names. Field names let you skip fields (unset ones get the type's **zero value** - `0`, `""`, `nil`...) and reorder lines. This is Go's object-literal, and you will see it constantly.
- Honest footnote, no code needed yet: several HTTP requests arriving at once would mutate this shared slice from several goroutines - a **data race**. The official tutorial has the same flaw. The database in Stage 2 makes the whole problem disappear, which is part of why "a server needs a real data source" is not just about persistence.

#### main(): the entry point

```go
func main() {
	router := chi.NewRouter()

	router.Get("/healthz", getHealth)
	router.Get("/api/v1/animals", getAnimals)
	router.Get("/api/v1/animals/{id}", getAnimalByID)
	router.Post("/api/v1/animals", postAnimal)

	slog.Info("api server listening", "addr", "localhost:8080")
	if err := http.ListenAndServe("localhost:8080", router); err != nil {
		slog.Error("server stopped", "error", err)
	}
}
```

- **`func main()`** in `package main` is precisely what runs when you do `go run .`. Nothing else in Go "calls" it; the runtime does.
- **`router := chi.NewRouter()`** creates chi's router: a table that maps URL patterns to handlers. `:=` is Go's short declaration: declare *and* initialize in one line, with the type inferred from the right-hand side. You will use `:=` for almost everything. What comes back is a `*chi.Mux`, which is nothing more than an `http.Handler` with registration methods bolted on - it has a `ServeHTTP(w, r)` method like every other HTTP handler in Go.
- **Function values, a core Go idea.** `router.Get("/healthz", getHealth)` passes the *function itself*, not a call of it: no `()` after `getHealth`. Functions are values in Go; chi stores yours in its table and calls each one when a matching request arrives. The rule of thumb is worth one conscious thought now: **passing a function omits the `()`, calling one includes it.**
- **`"/api/v1/animals/{id}"`**: the `{id}` is a **path parameter**. chi matches any single segment there and hands you the value via `chi.URLParam(r, "id")` in that handler. Note the braces: gin spells this `:id`, chi spells it `{id}`, and the standard library's `http.ServeMux` (since Go 1.22) also spells it `{id}` - so this is the spelling to carry in your head. chi panics at startup if two routes are genuinely ambiguous, which is a good moment to read the error rather than a mystery 404 later.
- **`http.ListenAndServe("localhost:8080", router)`** blocks forever serving on port 8080, and it is plain `net/http`, not a framework call. Its second argument is any `http.Handler`; `router` happens to be one. It returns an error if the port is taken - we log it. Stage 10 replaces this line with an `http.Server` you can shut down cleanly.
- **`slog.Info("api server listening", "addr", "localhost:8080")`**: key/value pairs, machine-greppable, and the argument list is variadic. Stage 2 gives `slog` a configured handler instead of the default.

#### The first handler: getHealth

A **handler** is any function with the signature `func(http.ResponseWriter, *http.Request)`. `http.ResponseWriter` is where you write the response; `*http.Request` is the incoming request. Neither is a chi type - both come from `net/http`, and chi did not invent them. Every handler below has exactly this shape; only the body changes.

```go
func getHealth(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
}
```

- **`w http.ResponseWriter`** is an interface: one method you care about (`Write([]byte) (int, error)`) plus a couple of helpers (`Header()`, `WriteHeader(status)`). Whatever you write is the response body. **`r *http.Request`** is a pointer because the request is shared, not copied, and the pointer is what carries the body, headers, URL, and the request's context.
- **`writeJSON(w, ...)`** is our own helper, defined at the bottom of the file. gin gave you `c.JSON` for this; with chi there is no such method, because chi is not in the business of writing bodies. Three lines of `encoding/json` replace it, and you can see exactly what they do (see the helper's own explanation below).
- **`map[string]string{"status": "ok"}`**: a map literal. In the official gin tutorial you would write `gin.H{"status": "ok"}`, and `gin.H` is literally a `map[string]any` - a map with string keys. Writing the map type out has the same effect with no framework in the picture. For one-off shapes a map is honest; real response types (from Stage 3 on) are structs with tags.
- **`http.StatusOK`**: status codes are constants from `net/http`: `200 OK`, `201 Created`, `400 Bad Request`, `404 Not Found` all have their names here. Prefer them over raw numbers; they are greppable and self-documenting.
- The JSON you get: `{"status":"ok"}`. No key order guarantees; Go's map serialization sorts keys.

#### getAnimals: returning the list

```go
func getAnimals(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, animals)
}
```

One line on purpose. `writeJSON` can serialize anything JSON-able - a struct, a map, a slice of either. Passing the whole slice produces a JSON array. This is the official tutorial's album list, verbatim. (The unused `r` parameter is fine: Go allows unused *parameters*, only unused *local variables* are an error. Handlers must have this exact signature to be registered at all.)

#### getAnimalByID: path parameters, errors, and the range loop

```go
func getAnimalByID(w http.ResponseWriter, r *http.Request) {
	// {id} in the route is a path parameter. chi stores the matched segment
	// in the request's context under the parameter's name.
	id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "id must be an integer"})
		return
	}

	for _, a := range animals {
		if a.ID == id {
			writeJSON(w, http.StatusOK, a)
			return
		}
	}

	writeJSON(w, http.StatusNotFound, map[string]string{"error": "animal not found"})
}
```

This is the most instructive handler of the file. Go-specific things, in order:

- **`chi.URLParam(r, "id")`** reads the segment chi matched against `{id}`. It is a plain function taking the request and the parameter name - the value lives in the request's context, not on some framework object. (`strconv.ParseInt` then turns the URL's text, say `"2"`, into an `int64`. Arguments: the string, the base (10), the bit size (64). The URL gives you *strings*; converting explicitly at the boundary is Go's norm, and `strconv` is the standard package for it.)
- **The error convention - the single most important pattern in this tutorial.** Go does not have exceptions for expected failures. Functions that can fail return **two values: result, error**. So almost every Go call looks like `x, err := f(...)`, immediately followed by:

  ```go
  if err != nil {
  	... handle it ...
  }
  ```

  `err != nil` is Go for "it failed". (`nil` is the zero value for interfaces and pointers - what a failed call returns instead of a real error.) You will write this shape hundreds of times. It looks repetitive next to exceptions; Go's answer is that *handling is explicit at every call site*, and nothing throws across ten stack frames without a line saying so.
- **`for _, a := range animals`** is Go's only loop. `range` walks the slice yielding index and element; `range` over a map yields key and value; and `range` over nothing but one variable (`for i := range x`) is the counting loop. The **`_` (blank identifier)** discards the index - assigning to `_` throws the value away, which is how Go spells "I only want the element here". Assigning to `_` comes up everywhere; you will meet its other use ("call and ignore the error") in Stage 7.
- **Early returns** instead of `else`: body writes and `return` inside the `if`, the not-found JSON at the bottom as the fall-through. Go style leans heavily on this - guard clauses and straight-line code - partly because of that every-call error check. There is no `try` nesting to escape.
- **`a.ID == id`**: comparing struct fields; `==` works on strings and numbers directly.

#### postAnimal: decoding a request body, appending

```go
func postAnimal(w http.ResponseWriter, r *http.Request) {
	var newAnimal animal
	if err := json.NewDecoder(r.Body).Decode(&newAnimal); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "could not parse request body"})
		return
	}

	newAnimal.ID = int64(len(animals) + 1)
	animals = append(animals, newAnimal)

	writeJSON(w, http.StatusCreated, newAnimal)
}
```

- **`var newAnimal animal`** declares the variable with its type's **zero value** (empty strings, ID 0). Go will not let you *use* it before it is set - well, it will not stop a partially-filled struct, but declaring-then-filling is idiomatic.
- **`json.NewDecoder(r.Body).Decode(&newAnimal)`** - note the **`&`**: you pass a pointer to your variable so the decoder can *mutate it in place*. The parse fills the struct through the pointer. This pass-by-pointer-when-the-callee-writes is the standard Go pattern for "fill in my struct"; you will pass `&req` to decode functions in every stage of this tutorial. (If you passed the struct without `&`, the decoder would fill a *copy* and your variable would stay empty.)
- **Why `NewDecoder(r.Body).Decode(...)` and not `json.Unmarshal`?** `Unmarshal` wants the whole body as a `[]byte`, which would mean reading it into memory first (`io.ReadAll`). A decoder streams straight from the request body. It is the standard spelling in HTTP handlers, and it is exactly what gin's `ShouldBindJSON` did under the hood.
- **The multi-value `if`**: `if err := json.NewDecoder(...).Decode(&newAnimal); err != nil` declares `err` *inside the if statement*, scoped to it and its body. Error checks in Go overwhelmingly use this scoping so that `err` can never leak with a stale value.
- Note what is *not* here any more: gin validated `binding:"required"` tags during binding and rejected the request for you. chi has no binder, so that job moves to whoever knows what a valid animal is - the service layer, from Stage 3 onward. At Stage 1 a missing field simply decodes to its zero value; `{"name":"Zuri"}` would create an animal with an empty species. That is a real hole, and closing it properly is one of the things the layers in Stage 3 exist for.
- **`int64(len(animals) + 1)`**: `len` gives the current count; `int64(...)` converts the resulting `int` to the struct's field type. Go never silently converts between integer types (there is no implicit widening) - the explicit conversion is required, and small like this becomes reflex.
- **`animals = append(animals, newAnimal)`**: `append` grows a slice and *returns the grown slice*, which must be assigned back. `append` alone (discarding the result) does nothing useful.
- **`http.StatusCreated`** (201) for a successful create is the REST convention this tutorial keeps for every POST.

#### writeJSON: the three lines gin was hiding

```go
// writeJSON is the whole of "return JSON": set the content type, write the
// status, encode the body. Stage 5 moves it into internal/platform/httpx.
func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.Error("write json response", "error", err)
	}
}
```

- **`w.Header().Set(...)` before `w.WriteHeader(...)`**: headers must be set before the status line goes out, because writing the status flushes them. Setting a header afterwards is a silent no-op - one of the classic `net/http` mistakes, and you can see it is impossible to make here because the three lines are in this order.
- **`w.WriteHeader(status)`**: without it, the first `Write` defaults to 200. Being explicit is how a 201 actually becomes a 201.
- **`json.NewEncoder(w).Encode(v)`** writes directly to the response writer, which is why there is no `[]byte` in sight. `Encode` appends a trailing newline to the JSON - harmless, and it is what gin did too.
- **`v any`**: `any` is an alias for `interface{}` - "a value of any type". A parameter of type `any` accepts a struct, a map, a slice, all of them.
- The error branch is why the helper takes no shortcut with `Marshal`: encoding can genuinely fail part-way through streaming (a value the encoder cannot represent). By then the status line is already sent, so the honest response is to log it, which is what we do.

That is the whole file, one declaration at a time. Assembled it looks like this:

#### The complete main.go

```go
package main

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"strconv"

	"github.com/go-chi/chi/v5"
)

// animal is the data we serve. The `json:"..."` tags set the field names
// encoding/json uses when serializing: without them you get Go's title-cased
// names (ID, Name, Species), which is not what JSON APIs usually do.
type animal struct {
	ID        int64  `json:"id"`
	Name      string `json:"name"`
	Species   string `json:"species"`
	Enclosure string `json:"enclosure"`
}

// In-memory data, exactly like the official tutorial's slice of albums.
// Nothing here is persistent yet: every restart forgets your animals.
var animals = []animal{
	{ID: 1, Name: "Tembo", Species: "African bush elephant", Enclosure: "Savanna"},
	{ID: 2, Name: "Suki", Species: "Sumatran tiger", Enclosure: "Jungle"},
	{ID: 3, Name: "Biscuit", Species: "Red panda", Enclosure: "Forest"},
}

func main() {
	router := chi.NewRouter()

	router.Get("/healthz", getHealth)
	router.Get("/api/v1/animals", getAnimals)
	router.Get("/api/v1/animals/{id}", getAnimalByID)
	router.Post("/api/v1/animals", postAnimal)

	slog.Info("api server listening", "addr", "localhost:8080")
	if err := http.ListenAndServe("localhost:8080", router); err != nil {
		slog.Error("server stopped", "error", err)
	}
}

func getHealth(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

func getAnimals(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, animals)
}

func getAnimalByID(w http.ResponseWriter, r *http.Request) {
	// {id} in the route is a path parameter. chi stores the matched segment
	// in the request's context under the parameter's name.
	id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "id must be an integer"})
		return
	}

	for _, a := range animals {
		if a.ID == id {
			writeJSON(w, http.StatusOK, a)
			return
		}
	}

	writeJSON(w, http.StatusNotFound, map[string]string{"error": "animal not found"})
}

func postAnimal(w http.ResponseWriter, r *http.Request) {
	var newAnimal animal
	if err := json.NewDecoder(r.Body).Decode(&newAnimal); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "could not parse request body"})
		return
	}

	newAnimal.ID = int64(len(animals) + 1)
	animals = append(animals, newAnimal)

	writeJSON(w, http.StatusCreated, newAnimal)
}

// writeJSON is the whole of "return JSON": set the content type, write the
// status, encode the body. Stage 5 moves it into internal/platform/httpx.
func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.Error("write json response", "error", err)
	}
}
```

Cross-check your version against this. There is nothing new in it: the pieces assembled above are exactly the file, in exactly that order.

### 1.4 Run and verify

```bash
go run .
```

In a second terminal:

```bash
curl http://localhost:8080/healthz
# {"status":"ok"}

curl http://localhost:8080/api/v1/animals
# [{"id":1,"name":"Tembo","species":"African bush elephant","enclosure":"Savanna"}, ...]

curl http://localhost:8080/api/v1/animals/2
# {"id":2,"name":"Suki","species":"Sumatran tiger","enclosure":"Jungle"}

curl -X POST http://localhost:8080/api/v1/animals \
  -H "Content-Type: application/json" \
  -d '{"name":"Zuri","species":"Reticulated giraffe","enclosure":"Savanna"}'
# {"id":4,"name":"Zuri","species":"Reticulated giraffe","enclosure":"Savanna"}
```

> **A gotcha that changed shape, worth learning once:** in the gin version of this tutorial the `Content-Type: application/json` header was load-bearing - gin's binder inspected it and refused bodies it did not recognize, so forgetting it produced a confusing "empty struct" bug. `encoding/json` does not look at `Content-Type` at all; it decodes whatever bytes arrive. This tutorial still sends the header on every mutating request, because that is what a correct HTTP client does and because servers, proxies, and future binders will care. Do not take its absence as proof that the decoder will be lenient forever.

Stop the server with Ctrl+C. Your animals are gone, which is the entire motivation for Stage 2.

---

[Overview](../tutorial.md)  |  [Stage 2](02-postgres-config-pool-migrations.md)
