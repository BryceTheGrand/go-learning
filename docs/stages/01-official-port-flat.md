## Stage 1: The official tutorial, ported to a zoo (flat, in-memory)

Everything in this stage lives in one file, exactly like the official tutorial. Two differences: our domain is animals, and we add a health-check endpoint you will rely on for the rest of the tutorial.

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

### 1.2 Install gin

```bash
go get -u github.com/gin-gonic/gin
```

This downloads gin and adds it to `go.mod` and `go.sum` (the lockfile listing exact versions plus checksums). `-u` means "use the latest version of this package and its own dependencies". Run `go get` once more after any import changes if an editor complains that a package is unresolved.

### 1.3 The first main.go, one piece at a time

Go programs are built from **packages** (files that compile together and share their names) and functions. So that the first file does not arrive as a wall of code, we will build `main.go` top to bottom, one declaration at a time, and print the complete file at the end of this section.

#### The package clause and the imports

Every Go file starts with these two things:

```go
package main

import (
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
)
```

What each line means:

- **`package main`** names the package this file belongs to. A package is the compilation and naming unit: every file in the same directory must declare the same package name, and they can refer to each other's identifiers without imports. `main` is the one special name: a package named `main` must contain a `func main()`, and that function is what a built binary executes. Every other package in this project gets a real domain name (Stage 5).
- **`import "..."`** says "this file uses code from that package", and the quoted string is the *import path*. The two blank-line groups are pure convention: standard-library paths (`net/http`, `strconv`) on top, third-party below. Both are real packages from real places - `net/http` ships with Go, `strconv` too; `github.com/gin-gonic/gin` is the gin dependency you installed in 1.2, and the path matches its repository. When this file says `http.StatusOK` or `gin.Default()`, the compiler resolves those names through these imports.
- Import paths and package names are different things: the path is where the code lives, the name is what you prefix identifiers with. `net/http`'s package *name* is `http`, which is why you write `http.StatusOK`, never `net/http.StatusOK`.

#### The data: a struct type

Go is a statically typed language - every value has a type known at compile time - and a **struct** is how you group related values under one name. Conceptually you know this from every language: it is a record. In Go it looks like this:

```go
// animal is the data we serve. The `json:"..."` tags set the field names
// gin uses when serializing to JSON: without them you get Go's title-cased
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
- **The backtick strings after each field are *struct tags*** - metadata you can attach to fields. They are ordinary constant strings; Go's compiler stores them, and libraries (here, gin's JSON encoder, which uses `encoding/json` under the hood) read them with reflection. The tag `json:"id"` says "when this struct becomes JSON, write this field under the key `id`". Without tags you would get the Go field names (`ID`, `Name`) - which is rarely the JSON convention APIs use, hence tags on every field. In this tutorial every type that touches the network has tags; it is a habit worth copying.
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
	router := gin.Default()

	router.GET("/healthz", getHealth)
	router.GET("/api/v1/animals", getAnimals)
	router.GET("/api/v1/animals/:id", getAnimalByID)
	router.POST("/api/v1/animals", postAnimal)

	router.Run("localhost:8080")
}
```

- **`func main()`** in `package main` is precisely what runs when you do `go run .`. Nothing else in Go "calls" it; the runtime does.
- **`router := gin.Default()`** creates gin's HTTP router (URL-to-handler table) pre-loaded with two standard behaviors: logging each request and recovering from handler panics. `:=` is Go's short declaration: declare *and* initialize in one line, with the type inferred from the right-hand side. You will use `:=` for almost everything.
- **Function values, a core Go idea.** `router.GET("/healthz", getHealth)` passes the *function itself*, not a call of it: no `()` after `getHealth`. Functions are values in Go; gin stores yours in a table and calls each one when a matching request arrives. The rule of thumb is worth one conscious thought now: **passing a function omits the `()`, calling one includes it.**
- **`"/api/v1/animals/:id"`**: the `:id` is a **path parameter**. Gin matches any segment there and hands you the value via `c.Param("id")` in that handler. Two gin routing rules worth knowing early: a static segment and a wildcard may coexist at the same position (`/animals/:id` next to `/animals/latest` is fine), but two *different* wildcard names at the same position (`/animals/:id` and `/animals/:name`) conflict - gin panics at startup with a loud wildcard-conflict message. Keeping one wildcard name per position is why every `:id` route in this tutorial shares that name.
- **`router.Run("localhost:8080")`** blocks forever serving on port 8080. It returns an error if the port is taken; Stage 3's version grows the error handling.

#### The first handler: getHealth

A **handler** is any function that takes one `*gin.Context`. The `*` means "pointer to": gin passes a pointer to its request-scoped object rather than a copy, because the same object carries the request in *and* the response out. Every handler below has exactly this shape; only the body changes.

```go
func getHealth(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{"status": "ok"})
}
```

- **`c *gin.Context`**: not Go's standard `context.Context` (that arrives in Stage 7, and the distinction matters). It carries the request and response plus helpers: `JSON` serializes a value and writes it with a status code, `Param` reads path parameters, `ShouldBindJSON` parses a request body into a struct.
- **`gin.H{"status": "ok"}`**: `gin.H` is a cheat for "JSON object" - it is literally `map[string]interface{}`, a map with string keys. `{...}` is a map literal. Handy for one-off shapes; real response types (from Stage 3 on) are structs with tags.
- **`http.StatusOK`**: status codes are constants from Go's `net/http` package, which gin builds on: `200 OK`, `201 Created`, `400 Bad Request`, `404 Not Found` all have their names here. Prefer them over raw numbers; they are greppable and self-documenting.
- The JSON you get: `{"status":"ok"}`. No key order guarantees; Go's map serialization sorts keys.

#### getAnimals: returning the list

```go
func getAnimals(c *gin.Context) {
	c.JSON(http.StatusOK, animals)
}
```

One line on purpose. `c.JSON` can serialize anything JSON-able - a struct, a map, a slice of either. Passing the whole slice produces a JSON array. This is the official tutorial's album list, verbatim.

#### getAnimalByID: path parameters, errors, and the range loop

```go
func getAnimalByID(c *gin.Context) {
	// :id in the route is a path parameter. Gin puts the actual value in
	// the request's context under the name of the parameter.
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "id must be a number"})
		return
	}

	for _, a := range animals {
		if a.ID == id {
			c.JSON(http.StatusOK, a)
			return
		}
	}

	c.JSON(http.StatusNotFound, gin.H{"error": "animal not found"})
}
```

This is the most instructive handler of the file. Go-specific things, in order:

- **`strconv.ParseInt(c.Param("id"), 10, 64)`** turns the URL's text (say `"2"`) into an `int64`. Arguments: the string, the base (10), the bit size (64). The URL gives you *strings*; converting explicitly at the boundary is Go's norm, and `strconv` is the standard package for it.
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

#### postAnimal: binding a request body, appending

```go
func postAnimal(c *gin.Context) {
	var newAnimal animal
	if err := c.ShouldBindJSON(&newAnimal); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	newAnimal.ID = int64(len(animals) + 1)
	animals = append(animals, newAnimal)

	c.JSON(http.StatusCreated, newAnimal)
}
```

- **`var newAnimal animal`** declares the variable with its type's **zero value** (empty strings, ID 0). Go will not let you *use* it before it is set - well, it will not stop a partially-filled struct, but declaring-then-filling is idiomatic.
- **`c.ShouldBindJSON(&newAnimal)`** - note the **`&`**: you pass a pointer to your variable so gin can *mutate it in place*. The parse fills the struct through the pointer. This pass-by-pointer-when-the-callee-writes is the standard Go pattern for "fill in my struct"; you will pass `&req` to binding functions in every stage of this tutorial. (If you passed the struct without `&`, gin would fill a *copy* and your variable would stay empty. The error convention catches this class of bug only at runtime - the handler would 400 or, in some shapes, store an empty animal.)
- **The multi-value `if`**: `if err := c.ShouldBindJSON(&newAnimal); err != nil` declares `err` *inside the if statement*, scoped to it and its body. Error checks in Go overwhelmingly use this scoping so that `err` can never leak with a stale value.
- **`err.Error()`**: an error is an interface with one method, `Error() string`, which yields a human-readable message. Calling it embeds that message in the response.
- **`int64(len(animals) + 1)`**: `len` gives the current count; `int64(...)` converts the resulting `int` to the struct's field type. Go never silently converts between integer types (there is no implicit widening) - the explicit conversion is required, and small like this becomes reflex.
- **`animals = append(animals, newAnimal)`**: `append` grows a slice and *returns the grown slice*, which must be assigned back. `append` alone (discarding the result) does nothing useful.
- **`http.StatusCreated`** (201) for a successful create is the REST convention this tutorial keeps for every POST.

That is the whole file, in five pieces. Assembled it looks like this:

#### The complete main.go

```go
package main

import (
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
)

// animal is the data we serve. The `json:"..."` tags set the field names
// gin uses when serializing to JSON: without them you get Go's title-cased
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
	router := gin.Default()

	router.GET("/healthz", getHealth)
	router.GET("/api/v1/animals", getAnimals)
	router.GET("/api/v1/animals/:id", getAnimalByID)
	router.POST("/api/v1/animals", postAnimal)

	router.Run("localhost:8080")
}

func getHealth(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{"status": "ok"})
}

func getAnimals(c *gin.Context) {
	c.JSON(http.StatusOK, animals)
}

func getAnimalByID(c *gin.Context) {
	// :id in the route is a path parameter. Gin puts the actual value in
	// the request's context under the name of the parameter.
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "id must be a number"})
		return
	}

	for _, a := range animals {
		if a.ID == id {
			c.JSON(http.StatusOK, a)
			return
		}
	}

	c.JSON(http.StatusNotFound, gin.H{"error": "animal not found"})
}

func postAnimal(c *gin.Context) {
	var newAnimal animal
	if err := c.ShouldBindJSON(&newAnimal); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	newAnimal.ID = int64(len(animals) + 1)
	animals = append(animals, newAnimal)

	c.JSON(http.StatusCreated, newAnimal)
}
```

Cross-check your version against this. It differs from what the pieces showed only in nothing - assembly is exactly what the pieces were.

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

> **Gotcha, learn it once now:** without `-H "Content-Type: application/json"`, `curl` sends `application/x-www-form-urlencoded`, and `ShouldBindJSON` fails or - worse in some handler shapes - parses into an empty struct. Every mutating request in this tutorial sends that header. If binding ever "silently fails", this is the first thing to check.

Stop the server with Ctrl+C. Your animals are gone, which is the entire motivation for Stage 2.

---

[Stage 0](../tutorial.md)  ·  [Overview](../tutorial.md)  ·  [Stage 2](02-postgres-config-pool-migrations.md)
