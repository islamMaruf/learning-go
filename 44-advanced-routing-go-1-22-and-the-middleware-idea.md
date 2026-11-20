# Chapter 44: Advanced Routing (Go 1.22+) and the Middleware Idea

> **Goal of this chapter:** Replace hand-written `switch r.Method` code with Go 1.22's **method-aware, wildcard-capable routing patterns** such as `GET /products/{id}`. You'll learn the pattern syntax and precedence rules, get automatic `405 Method Not Allowed` responses for free, read **path parameters** with `r.PathValue`, add a `GET /products/{id}` endpoint, understand why web frameworks used to be necessary (and when they still are), and see why **middleware** is the right home for CORS and other cross-cutting behavior.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 3 hours  **Prerequisite:** [Chapter 43](43-refactoring-the-codebase.md)

> ⚠️ **Requires Go 1.22 or newer** (`go version`), and `go 1.22` (or higher) in your `go.mod`. Older versions silently ignore method patterns and `{wildcards}` and treat them as literal text, a very confusing failure mode.

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The problem with the old routing](#2-the-problem-with-the-old-routing)
3. [Go 1.22 patterns](#3-go-122-patterns)
4. [Wildcards and `r.PathValue`](#4-wildcards-and-rpathvalue)
5. [Which pattern wins? Precedence and conflicts](#5-which-pattern-wins)
6. [Automatic `405`, `HEAD`, and `404`](#6-automatic-405-head-and-404)
7. [Why frameworks existed](#7-why-frameworks-existed)
8. [Upgrading our project](#8-upgrading-our-project)
9. [Trying it out](#9-trying-it-out)
10. [The `OPTIONS` problem, and why middleware solves it](#10-the-options-problem)
11. [Middleware, revisited: chains, order, and stopping](#11-middleware-revisited)
12. [Tests](#12-tests)
13. [Custom JSON for `404` and `405`](#13-custom-json-for-404-and-405)
14. [Common mistakes](#14-common-mistakes)
15. [Exercises](#15-exercises)
16. [Quiz](#16-quiz)
17. [Summary](#17-summary)

---

## 1. What you will learn

- The Go 1.22 **pattern syntax**: `[METHOD ][HOST]/path` with `{name}`, `{name...}`, and `{$}`
- How the mux chooses among overlapping patterns, and how conflicts are reported
- Reading **path parameters** with `r.PathValue("id")` and validating them
- Getting `405 Method Not Allowed` (with an `Allow` header) automatically
- Why CORS preflights and method routing interact, and how middleware fixes it
- The **chain pattern**: how wrapped handlers execute and how a middleware can *stop* a request

---

## 2. The problem with the old routing

Before Go 1.22, `http.ServeMux` matched **only the path**. To support several methods on one path, every handler had to repeat method-checking boilerplate (our Chapter 43 `Products` dispatcher):

```go
func Products(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		GetProducts(w, r)
	case http.MethodPost:
		CreateProduct(w, r)
	default:
		w.Header().Set("Allow", "GET, POST, OPTIONS")
		util.SendError(w, http.StatusMethodNotAllowed, "method not allowed")
	}
}
```

And there was **no built-in way to capture a piece of the URL** such as the `42` in `/products/42`. You had to slice strings by hand:

```go
id := strings.TrimPrefix(r.URL.Path, "/products/")   // "42" (and hope it's valid)
```

Every project either did that or pulled in a routing library (section 7).

**Problems:**

1. Method checks repeated in every handler (or a dispatcher per path)
2. Hand-parsed path parameters: fragile
3. Handlers doing routing work, so they do too many things (SRP violation)
4. Every new route means editing the dispatcher

---

## 3. Go 1.22 patterns

A route pattern is now:

```
[METHOD ][HOST]/[PATH]
```

The method and host are optional; the space separates the method from the rest.

```go
mux.HandleFunc("GET /products", handlers.GetProducts)
mux.HandleFunc("POST /products", handlers.CreateProduct)
mux.HandleFunc("GET /products/{id}", handlers.GetProduct)
```

| Pattern | Matches |
|---------|---------|
| `/products` | **Any method**, exactly `/products` |
| `GET /products` | `GET` (and `HEAD`) requests to exactly `/products` |
| `POST /products` | `POST` to exactly `/products` |
| `GET /products/{id}` | `GET`/`HEAD` to `/products/` + **one** path segment (captured as `id`) |
| `GET /files/{path...}` | `GET` to `/files/` + **the rest of the path** (any number of segments) |
| `GET /{$}` | `GET` to **exactly** `/` (and nothing else: see below) |
| `GET example.com/docs` | `GET` only when the request's `Host` is `example.com` |
| `/static/` | Any method; the **whole subtree** `/static/...` (a trailing slash means "prefix") |

Rules to remember:

- **Method names are case-sensitive and must be uppercase** (`GET`, not `get`).
- A pattern **without** a method matches **all** methods.
- `GET` implies `HEAD`; you don't register `HEAD` separately.
- **`/` alone is a catch-all** matching every path that nothing else matched; use **`/{$}`** for *only* the home page.
- A trailing slash on a pattern means "this path and everything below it"; without it, only the exact path.

---

## 4. Wildcards and `r.PathValue`

A **wildcard** is a `{name}` segment. It matches exactly **one path segment** (no slashes), and the matched text is available in the handler via `r.PathValue("name")`.

```go
func GetProduct(w http.ResponseWriter, r *http.Request) {
	idText := r.PathValue("id")      // e.g. "42" for /products/42
	id, err := strconv.Atoi(idText)  // the value is ALWAYS a string; convert and validate it
	...
}
```

Real examples, verified against a router with these patterns:

```go
mux.HandleFunc("GET /{$}", home)
mux.HandleFunc("GET /items/{id}", item)
mux.HandleFunc("GET /files/{path...}", file)
mux.HandleFunc("GET /users/{uid}/orders/{oid}", order)
```

| Request | Result |
|---------|--------|
| `GET /` | `home` |
| `GET /x` | `404` (`/{$}` matches *only* `/`) |
| `GET /items/5` | `item` (`id = "5"`) |
| `GET /items/5/x` | `404` (a `{id}` wildcard is one segment) |
| `GET /files/a/b/c.txt` | `file` (`path = "a/b/c.txt"`: the `...` swallows the rest) |
| `GET /users/7/orders/99` | `order` (`uid = "7"`, `oid = "99"`) |
| `GET /items//5` | `301` redirect to `/items/5` (the mux cleans doubled slashes) |

**Important:** path values are **untrusted user input**, exactly like query parameters and JSON bodies. Always validate: is `id` a positive integer? Is the string too long? The mux only guarantees *one segment*, not that it makes sense.

Wildcards must occupy a *whole segment* (`/items/{id}` is valid; `/items/a{id}` is not), and a `{name...}` wildcard can only be the last segment. Wildcard names in a pattern must be unique.

---

## 5. Which pattern wins?

Several patterns can match the same request. Go picks the **most specific** one, defined precisely:

> Pattern A is **more specific** than B if A matches a strict subset of the requests B matches.

```go
mux.HandleFunc("GET /items/{id}", byID)   // matches /items/anything
mux.HandleFunc("GET /items/new", newForm) // matches only /items/new  → more specific
```

`GET /items/new` → `newForm` (the literal beats the wildcard). `GET /items/5` → `byID`. No conflict: registration order doesn't matter.

### Conflicts panic at start-up

If two patterns match overlapping sets and **neither is more specific**, registering the second **panics immediately**, which is *good*: you find the ambiguity when the server starts, not in production. Real messages:

**Identical matching sets** (wildcard names don't matter):

```go
mux.HandleFunc("GET /items/{id}", a)
mux.HandleFunc("GET /items/{name}", b)
```

```
panic: pattern "GET /items/{name}" (registered at main.go:33) conflicts with pattern "GET /items/{id}" (registered at main.go:32):
GET /items/{name} matches the same requests as GET /items/{id}
```

**Overlapping but neither more specific:**

```go
mux.HandleFunc("GET /a/{x}/b", one)
mux.HandleFunc("GET /a/b/{y}", two)
```

```
panic: pattern "GET /a/b/{y}" ... conflicts with pattern "GET /a/{x}/b":
GET /a/b/{y} and GET /a/{x}/b both match some paths, like "/a/b/b".
But neither is more specific than the other.
GET /a/b/{y} matches "/a/b/y", but GET /a/{x}/b doesn't.
GET /a/{x}/b matches "/a/x/b", but GET /a/b/{y} doesn't.
```

Read these carefully; the mux even gives you an example path that's ambiguous.

**Method and host** also participate: a pattern with a method is more specific than one without (`GET /x` beats `/x` for GET requests); one with a host beats one without.

---

## 6. Automatic `405`, `HEAD`, and `404`

Registering method-specific patterns gives you correct HTTP behavior **for free**:

- Path exists, but not for this method → **`405 Method Not Allowed`** with an **`Allow`** header listing the supported methods.
- Path doesn't exist at all → **`404 Not Found`**.
- `GET` patterns also answer `HEAD`.

Real output against our upgraded server (routes: `GET /products`, `POST /products`, `GET /products/{id}`):

```bash
curl -i -X DELETE http://localhost:8080/products
```

```
HTTP/1.1 405 Method Not Allowed
Allow: GET, HEAD, POST
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Content-Length: 19

Method Not Allowed
```

```bash
curl -i -X POST http://localhost:8080/products/1
```

```
HTTP/1.1 405 Method Not Allowed
Allow: GET, HEAD
...
```

`Allow` is computed from the registered patterns: `/products` supports `GET, HEAD, POST`; `/products/{id}` only `GET, HEAD`. We wrote **zero** code for this.

`HEAD /products` works too: it returns the headers (with `Content-Type`) and no body.

The default `404`/`405` bodies are **plain text**, not our JSON error shape (section 13 discusses that).

---

## 7. Why frameworks existed

Before 1.22, real projects almost always adopted a third-party router or framework because the standard mux lacked:

| Missing feature | What people used |
|-----------------|------------------|
| Method routing (`GET /x` vs `POST /x`) | gorilla/mux, chi, httprouter |
| Path parameters (`/users/:id`) | gorilla/mux, chi, gin, echo |
| Route groups and middleware chains | chi, gin, echo |
| Request binding/validation helpers | gin, echo |

Popular ones: **chi** (small, idiomatic, `net/http`-compatible), **gin**, **echo**, **fiber**, **gorilla/mux** (archived for a while, since revived).

**Do you still need one?** For many services, **no**: the standard library now covers method routing, path parameters, and (via the middleware pattern) composition. Advantages of staying with `net/http`:

- No dependency to audit/upgrade; stable API (Go compatibility promise)
- Everything you learn transfers to every framework (they're built on `http.Handler`)
- Fewer moving parts to explain and debug

Frameworks still shine for: route *groups* with per-group middleware, built-in request binding/validation, OpenAPI tooling, or when a team already knows them. **`chi` is the natural upgrade path** because it uses the same `http.Handler` and `func(http.Handler) http.Handler` middleware we're writing.

---

## 8. Upgrading our project

Changes from Chapter 43:

1. `database`: add `GetProduct(id)`.
2. `handlers`: **delete the `Products` dispatcher** (routing is no longer the handler's job), add `GetProduct`.
3. `main.go`: register three method-aware routes.
4. Tests: the old `405` test now expects the mux's automatic `405`; new tests for the by-ID endpoint.

```go
// file: database/products.go
package database

import (
	"ecommerce/models"
	"sync"
)

var (
	mu       sync.RWMutex // guards products and nextID
	products []models.Product
	nextID   int
)

func init() { ResetProducts() }

// ResetProducts restores the seed data. (Useful for tests.)
func ResetProducts() {
	mu.Lock()
	defer mu.Unlock()
	products = []models.Product{
		{ID: 1, Title: "Orange", Description: "Orange is juicy and full of vitamin C.", Price: 100, ImageURL: "https://example.com/orange.jpg"},
		{ID: 2, Title: "Apple", Description: "A crunchy apple a day...", Price: 40, ImageURL: "https://example.com/apple.jpg"},
		{ID: 3, Title: "Banana", Description: "Great for a quick snack.", Price: 5, ImageURL: "https://example.com/banana.jpg"},
	}
	nextID = 4
}

// ListProducts returns a copy of all products.
func ListProducts() []models.Product {
	mu.RLock()
	defer mu.RUnlock()
	out := make([]models.Product, len(products))
	copy(out, products)
	return out
}

// CreateProduct assigns the next ID to p, stores it, and returns the stored product.
func CreateProduct(p models.Product) models.Product {
	mu.Lock()
	defer mu.Unlock()
	p.ID = nextID
	nextID++
	products = append(products, p)
	return p
}

// GetProduct returns the product with the given ID.
func GetProduct(id int) (models.Product, bool) {
	mu.RLock()
	defer mu.RUnlock()
	for _, p := range products {
		if p.ID == id {
			return p, true
		}
	}
	return models.Product{}, false
}
```

```go
// file: handlers/product.go
package handlers

import (
	"ecommerce/database"
	"ecommerce/models"
	"ecommerce/util"
	"fmt"
	"net/http"
	"strconv"
	"strings"
)

// createProductRequest is what a client may send to create a product.
// It deliberately has no ID: the server assigns it.
type createProductRequest struct {
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}

func (r createProductRequest) validate() string {
	switch {
	case strings.TrimSpace(r.Title) == "":
		return "title is required"
	case len(r.Title) > 100:
		return "title must be at most 100 characters"
	case r.Price <= 0:
		return "price must be greater than 0"
	}
	return ""
}

// GetProducts handles GET /products.
func GetProducts(w http.ResponseWriter, r *http.Request) {
	util.SendData(w, http.StatusOK, database.ListProducts())
}

// CreateProduct handles POST /products.
func CreateProduct(w http.ResponseWriter, r *http.Request) {
	var req createProductRequest
	if !util.DecodeJSON(w, r, &req) {
		return // DecodeJSON already wrote the error response
	}
	if msg := req.validate(); msg != "" {
		util.SendError(w, http.StatusUnprocessableEntity, msg)
		return
	}

	product := database.CreateProduct(models.Product{
		Title:       strings.TrimSpace(req.Title),
		Description: req.Description,
		Price:       req.Price,
		ImageURL:    req.ImageURL,
	})

	w.Header().Set("Location", fmt.Sprintf("/products/%d", product.ID))
	util.SendData(w, http.StatusCreated, product)
}

// GetProduct handles GET /products/{id}.
func GetProduct(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id < 1 {
		util.SendError(w, http.StatusBadRequest, "id must be a positive integer")
		return
	}

	product, found := database.GetProduct(id)
	if !found {
		util.SendError(w, http.StatusNotFound, "product not found")
		return
	}
	util.SendData(w, http.StatusOK, product)
}
```

```go
// file: main.go
package main

import (
	"ecommerce/handlers"
	"ecommerce/middleware"
	"fmt"
	"net/http"
)

// newRouter builds the complete HTTP handler for the app.
func newRouter() http.Handler {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /products", handlers.GetProducts)
	mux.HandleFunc("POST /products", handlers.CreateProduct)
	mux.HandleFunc("GET /products/{id}", handlers.GetProduct)

	return middleware.CORS(mux)
}

func main() {
	fmt.Println("Server running on http://localhost:8080")
	if err := http.ListenAndServe(":8080", newRouter()); err != nil {
		fmt.Println("Error starting the server:", err)
	}
}
```

Notice how `handlers.GetProducts` shrank to one line: it no longer checks the method (the mux already guaranteed it). **Routing is now declared in one place** (`newRouter`), and each handler does its one job: exactly the SRP goal from Chapter 43.

---

## 9. Trying it out

Start the server and try (real output from the built project):

```bash
curl -si http://localhost:8080/products/2
```

```
HTTP/1.1 200 OK
Content-Type: application/json
...
{"id":2,"title":"Apple","description":"A crunchy apple a day...","price":40,"imageUrl":"https://example.com/apple.jpg"}
```

```bash
curl -s http://localhost:8080/products/abc     # {"error":"id must be a positive integer"}      (400)
curl -s http://localhost:8080/products/99      # {"error":"product not found"}                  (404)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/products/          # 404 (no trailing-slash route)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/nothing            # 404
```

Everything behaves as the design says. In an API client (Postman, Insomnia, Bruno), notice how the request builder shows `{{id}}`-style variables for path parameters: they map directly onto `{id}`.

---

## 10. The `OPTIONS` problem

Method-specific patterns raise a question: **what happens to a browser's preflight `OPTIONS` request?** We registered no `OPTIONS` route. The mux would answer `405` (plain, without CORS headers), and the browser would refuse to send the real request, the Chapter 42 failure all over again.

**The naive fix** would be to register `OPTIONS` for every route:

```go
mux.HandleFunc("OPTIONS /products", preflight)
mux.HandleFunc("OPTIONS /products/{id}", preflight)
// ... and one for every new route, forever
```

That's exactly the repetition we're trying to eliminate: forget one, and that route breaks in browsers.

**The middleware fix** (already in place since Chapter 43): our `middleware.CORS` **wraps the entire mux** and answers preflight requests *before the mux ever sees them*:

```
browser ── OPTIONS /products/7 ──►  CORS middleware  ── (is it a preflight from an allowed origin?) ── yes ──► 204 (mux never involved)
browser ── GET /products/7     ──►  CORS middleware  ──(not a preflight)──►  mux  ──►  handler
```

Real output from the built project confirms both paths:

```bash
# A plain OPTIONS with no Origin is NOT a preflight → passes through to the mux → automatic 405
curl -si -X OPTIONS http://localhost:8080/products | head -2
# HTTP/1.1 405 Method Not Allowed
# Allow: GET, HEAD, POST

# A real preflight from an allowed origin → answered by the middleware
curl -si -X OPTIONS http://localhost:8080/products \
  -H "Origin: http://localhost:5173" -H "Access-Control-Request-Method: POST" | head -3
# HTTP/1.1 204 No Content
# Access-Control-Allow-Headers: Content-Type, Authorization
# Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
```

This is the payoff of middleware: **a rule that applies to every route lives in one place.** Add a hundred routes and CORS still works for all of them.

---

## 11. Middleware, revisited

### Definition

> **Middleware** is code that sits *between* the server and your handlers, running for every request that passes through it. It can inspect or modify the request, short-circuit with its own response, call the next handler, and inspect or modify the response on the way back.

In Go: **`func(http.Handler) http.Handler`**.

```
                    ┌───────────── middleware chain ─────────────┐
 request ──► Logger ──► CORS ──► Auth ──► mux ──► handler
                                                    │
 response ◄── Logger ◄── CORS ◄── Auth ◄── mux ◄────┘
```

**Analogy: airport security.** Every passenger (request) passes through checkpoints (middleware): passport check, baggage scan, boarding-pass check, *before* reaching the gate (handler). Each checkpoint can let you through or turn you back, and the same people work for every flight (route).

### The chain pattern

Because a middleware takes a handler and returns a handler, you can nest them:

```go
handler := Logger(CORS(Auth(mux)))
```

Execution order for the *request* is **outside → inside** (`Logger` first, then `CORS`, then `Auth`, then the mux). The *response* travels back **inside → outside**. Each middleware can do work **before** `next.ServeHTTP(w, r)` (request phase) and **after** it (response phase):

```go
func Timing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()             // BEFORE: request phase
		next.ServeHTTP(w, r)            // hand over to the rest of the chain
		log.Println(time.Since(start))  // AFTER: response phase
	})
}
```

### Why order matters

```go
handler := Auth(CORS(mux))   // ❌ Auth runs first
handler := CORS(Auth(mux))   // ✅ CORS runs first
```

If `Auth` is outermost, then an unauthenticated **preflight `OPTIONS`** (which browsers send *without* credentials!) hits `Auth` first and gets `401`, so the browser blocks every request, even valid ones. CORS must be **outside** auth so it can answer preflights and decorate error responses. General guidance (outermost → innermost): *recover from panics → logging/request-ID → CORS → rate limiting → authentication → routing → handler.* Chapters 45–46 build several of these.

### Stopping the chain

A middleware doesn't have to call `next`. Our CORS middleware **stops** the chain for preflights by writing a response and `return`ing:

```go
if isPreflight {
	w.WriteHeader(http.StatusNoContent)
	return // next.ServeHTTP is never called
}
```

Authentication middleware (Chapter 49) does the same for missing tokens: respond `401` and `return`.

### Middleware benefits

| Benefit | Explanation |
|---------|-------------|
| **DRY** | Cross-cutting logic written once |
| **SRP** | Handlers only do their own job |
| **Consistency** | Every route gets the same behavior |
| **Composability** | Mix and match by wrapping |
| **Open/Closed** | Add behavior without editing handlers |
| **Testability** | Test a middleware alone with a dummy `next` |

---

## 12. Tests

Update `main_test.go` for the new routing (the `405` now comes from the mux) and add tests for the by-ID endpoint. Everything else from Chapter 43 is unchanged.

```go
// file: main_test.go
package main

import (
	"ecommerce/database"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"strings"
	"sync"
	"testing"
)

type product struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
}

func do(method, target, body string, headers map[string]string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	if body != "" {
		req.Header.Set("Content-Type", "application/json")
	}
	for k, v := range headers {
		req.Header.Set(k, v)
	}
	rec := httptest.NewRecorder()
	newRouter().ServeHTTP(rec, req)
	return rec
}

func TestListProducts(t *testing.T) {
	database.ResetProducts()
	rec := do(http.MethodGet, "/products", "", nil)
	if rec.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rec.Code)
	}
	var list []product
	if err := json.NewDecoder(rec.Body).Decode(&list); err != nil {
		t.Fatal(err)
	}
	if len(list) != 3 {
		t.Fatalf("expected 3 products, got %d", len(list))
	}
}

func TestGetProductByID(t *testing.T) {
	database.ResetProducts()

	tests := []struct {
		name string
		path string
		want int
	}{
		{"found", "/products/2", http.StatusOK},
		{"not found", "/products/99", http.StatusNotFound},
		{"not a number", "/products/abc", http.StatusBadRequest},
		{"zero", "/products/0", http.StatusBadRequest},
		{"negative", "/products/-4", http.StatusBadRequest},
		{"too many segments", "/products/2/extra", http.StatusNotFound},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			if rec := do(http.MethodGet, tc.path, "", nil); rec.Code != tc.want {
				t.Errorf("GET %s: expected %d, got %d (%s)", tc.path, tc.want, rec.Code, rec.Body.String())
			}
		})
	}

	var p product
	json.NewDecoder(do(http.MethodGet, "/products/2", "", nil).Body).Decode(&p)
	if p.ID != 2 || p.Title != "Apple" {
		t.Errorf("unexpected product %+v", p)
	}
}

func TestCreateProduct(t *testing.T) {
	database.ResetProducts()
	rec := do(http.MethodPost, "/products", `{"title":"Mango","description":"d","price":200,"imageUrl":"u"}`, nil)
	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d (%s)", rec.Code, rec.Body.String())
	}
	if loc := rec.Header().Get("Location"); loc != "/products/4" {
		t.Errorf("Location = %q", loc)
	}
	// the Location header must actually work
	if got := do(http.MethodGet, rec.Header().Get("Location"), "", nil); got.Code != http.StatusOK {
		t.Errorf("GET %s = %d", rec.Header().Get("Location"), got.Code)
	}
}

func TestCreateProductErrors(t *testing.T) {
	tests := []struct {
		name string
		body string
		want int
	}{
		{"empty body", ``, http.StatusBadRequest},
		{"truncated json", `{"title": `, http.StatusBadRequest},
		{"wrong type", `{"title":"x","price":"cheap"}`, http.StatusBadRequest},
		{"unknown field", `{"id":9,"title":"x","price":1}`, http.StatusBadRequest},
		{"missing title", `{"price":5}`, http.StatusUnprocessableEntity},
		{"negative price", `{"title":"x","price":-1}`, http.StatusUnprocessableEntity},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			database.ResetProducts()
			if rec := do(http.MethodPost, "/products", tc.body, nil); rec.Code != tc.want {
				t.Errorf("expected %d, got %d (%s)", tc.want, rec.Code, rec.Body.String())
			}
		})
	}
}

func TestAutomaticMethodNotAllowed(t *testing.T) {
	tests := []struct {
		method, path, allow string
	}{
		{http.MethodDelete, "/products", "GET, HEAD, POST"},
		{http.MethodPut, "/products", "GET, HEAD, POST"},
		{http.MethodPost, "/products/1", "GET, HEAD"},
	}
	for _, tc := range tests {
		rec := do(tc.method, tc.path, "", nil)
		if rec.Code != http.StatusMethodNotAllowed {
			t.Errorf("%s %s: expected 405, got %d", tc.method, tc.path, rec.Code)
		}
		if got := rec.Header().Get("Allow"); got != tc.allow {
			t.Errorf("%s %s: Allow = %q, want %q", tc.method, tc.path, got, tc.allow)
		}
	}
}

func TestHeadWorksOnGetRoutes(t *testing.T) {
	rec := do(http.MethodHead, "/products", "", nil)
	if rec.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rec.Code)
	}
}

func TestPreflightFromAllowedOrigin(t *testing.T) {
	// preflights are answered by the middleware, even for routes with path parameters
	for _, path := range []string{"/products", "/products/7"} {
		rec := do(http.MethodOptions, path, "", map[string]string{
			"Origin":                         "http://localhost:5173",
			"Access-Control-Request-Method":  "POST",
			"Access-Control-Request-Headers": "content-type,authorization",
		})
		if rec.Code != http.StatusNoContent {
			t.Fatalf("%s: expected 204, got %d", path, rec.Code)
		}
		h := rec.Header()
		if h.Get("Access-Control-Allow-Origin") != "http://localhost:5173" {
			t.Errorf("%s: Allow-Origin = %q", path, h.Get("Access-Control-Allow-Origin"))
		}
		if !strings.Contains(h.Get("Access-Control-Allow-Headers"), "Authorization") {
			t.Errorf("%s: Allow-Headers = %q", path, h.Get("Access-Control-Allow-Headers"))
		}
		if rec.Body.Len() != 0 {
			t.Errorf("%s: preflight must have no body", path)
		}
	}
}

func TestPlainOptionsIsNotAPreflight(t *testing.T) {
	// no Origin header → not a browser preflight → goes to the mux → 405
	rec := do(http.MethodOptions, "/products", "", nil)
	if rec.Code != http.StatusMethodNotAllowed {
		t.Errorf("expected 405, got %d", rec.Code)
	}
}

func TestPreflightFromUnknownOrigin(t *testing.T) {
	rec := do(http.MethodOptions, "/products", "", map[string]string{
		"Origin":                        "https://evil.example",
		"Access-Control-Request-Method": "POST",
	})
	if rec.Header().Get("Access-Control-Allow-Origin") != "" {
		t.Errorf("unknown origins must not be allowed")
	}
}

func TestCORSHeaderOnRealAndErrorResponses(t *testing.T) {
	database.ResetProducts()
	origin := map[string]string{"Origin": "http://localhost:5173"}

	responses := map[string]*httptest.ResponseRecorder{
		"created": do(http.MethodPost, "/products", `{"title":"Kiwi","price":30}`, origin),
		"invalid": do(http.MethodPost, "/products", `{"title":""}`, origin),
		"404":     do(http.MethodGet, "/no-such-route", "", origin),
		"405":     do(http.MethodDelete, "/products", "", origin),
	}
	for name, rec := range responses {
		if rec.Header().Get("Access-Control-Allow-Origin") != "http://localhost:5173" {
			t.Errorf("%s response is missing the CORS header", name)
		}
	}
}

func TestConcurrentCreates(t *testing.T) {
	database.ResetProducts()
	const n = 50
	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			do(http.MethodPost, "/products", `{"title":"Concurrent","price":1}`, nil)
		}()
	}
	wg.Wait()

	var list []product
	json.NewDecoder(do(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	if len(list) != 3+n {
		t.Fatalf("expected %d products, got %d", 3+n, len(list))
	}
}
```

```bash
go test -race ./...
```

```
ok  	ecommerce	1.0s
```

---

## 13. Custom JSON for `404` and `405`

The mux's automatic responses are **plain text** (`404 page not found`, `Method Not Allowed`), while everything else in our API speaks JSON (`{"error": ...}`). Clients that always `JSON.parse` the body would choke on those.

You have a few options:

1. **Accept it** for now (many APIs do), noting that status codes are what clients should branch on.
2. **A response-rewriting middleware** that wraps the `ResponseWriter`, detects the mux's plain-text `404`/`405` and replaces them with JSON. It needs a `ResponseWriter` wrapper: a technique Chapter 46 introduces for logging status codes.
3. **A catch-all pattern**: registering `"/"` gives a handler for unmatched paths, but it also matches *every method on every path*, so it would **swallow the automatic `405`s** (a `DELETE /products` would hit the catch-all instead). Use it only if you implement the method logic yourself.

We'll revisit (2) when we build the response-writer wrapper. It's a good example of a trade-off: the standard mux is deliberately minimal.

---

## 14. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Go < 1.22 or `go 1.21` in `go.mod` | Patterns like `GET /x/{id}` never match (treated literally) | Upgrade Go; set `go 1.22` in `go.mod` |
| 2 | Lowercase method (`"get /x"`) | Never matches / behaves oddly | Uppercase: `"GET /x"` |
| 3 | Forgetting the space (`"GET/products"`) | Pattern is a *path* `GET/products` | `"GET /products"` |
| 4 | Two ambiguous patterns | **Panic at start-up** | Make one more specific or restructure |
| 5 | Wildcard mistakes: `/items/{id}/` vs `/items/{id}` | Unexpected 404s / subtree matching | A trailing slash means "subtree"; be exact |
| 6 | Trusting `r.PathValue` | Injection, crashes on non-numeric ids | Validate and convert (`strconv.Atoi`) |
| 7 | Registering `"/"` and expecting only the home page | Catches everything | Use `"/{$}"` |
| 8 | Registering `HEAD` separately for `GET` routes | Panic (conflict) or redundancy | `GET` already covers `HEAD` |
| 9 | Method checks left inside handlers | Redundant code, inconsistent errors | Remove them: the mux enforces methods |
| 10 | Putting `Auth` outside `CORS` | Browser preflights get `401` | `CORS` outermost |
| 11 | Expecting `r.PathValue` in a handler registered with a pattern *without* that wildcard | Returns `""` | Name must match the pattern |
| 12 | Forgetting the mux's `404/405` are plain text | JSON clients fail to parse | See section 13 |

---

## 15. Exercises

### Exercise 1: Read the patterns
For the patterns below, which handler serves each request? `GET /a`, `POST /a`, `GET /a/1`, `GET /a/new`, `DELETE /a/new`.

```go
mux.HandleFunc("GET /a", h1)
mux.HandleFunc("/a/{id}", h2)
mux.HandleFunc("GET /a/new", h3)
```

<details><summary>Solution</summary>

`GET /a` → h1. `POST /a` → 405 (pattern `GET /a` exists but not for POST; `/a/{id}` doesn't match `/a`). `GET /a/1` → h2. `GET /a/new` → h3 (more specific than h2 for GET). `DELETE /a/new` → h2 (`/a/{id}` matches any method and the id `new`).
</details>

### Exercise 2: Add `PUT` and `DELETE`
Register `PUT /products/{id}` and `DELETE /products/{id}` routes with handlers that respond `501 Not Implemented` for now. Check `curl -i -X POST /products/1` to see how `Allow` changes.

<details><summary>Solution</summary>

```go
mux.HandleFunc("PUT /products/{id}", handlers.NotImplemented)
mux.HandleFunc("DELETE /products/{id}", handlers.NotImplemented)
```
```go
func NotImplemented(w http.ResponseWriter, r *http.Request) {
	util.SendError(w, http.StatusNotImplemented, "not implemented yet")
}
```
`Allow` for `/products/{id}` becomes `DELETE, GET, HEAD, PUT`.
</details>

### Exercise 3: Product images
Add `GET /products/{id}/image` that redirects (302) to the product's `imageUrl`. What happens for unknown ids?

<details><summary>Solution</summary>

```go
mux.HandleFunc("GET /products/{id}/image", handlers.ProductImage)

func ProductImage(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id < 1 {
		util.SendError(w, http.StatusBadRequest, "id must be a positive integer")
		return
	}
	p, ok := database.GetProduct(id)
	if !ok {
		util.SendError(w, http.StatusNotFound, "product not found")
		return
	}
	http.Redirect(w, r, p.ImageURL, http.StatusFound)
}
```
</details>

### Exercise 4: Conflict hunt
Which of these pairs conflict? (a) `GET /x/{a}` & `GET /x/{b}`, (b) `GET /x/{a}` & `POST /x/{a}`, (c) `/x/{a}` & `GET /x/{a}`, (d) `GET /x/{a}/y` & `GET /x/z/{b}`.

<details><summary>Solution</summary>

(a) conflict (same matching set). (b) no conflict (different methods). (c) no conflict (the method-specific pattern is more specific for GET). (d) conflict (both match `/x/z/y`; neither is more specific).
</details>

### Exercise 5: Middleware order
You have `Logger`, `CORS`, `Auth`, `Recover`. Write the wrapping expression in the best order and justify it.

<details><summary>Solution</summary>

`Recover(Logger(CORS(Auth(mux))))`: `Recover` outermost so a panic anywhere (even in other middleware) becomes a `500` instead of a dropped connection; `Logger` next so every request is logged, including failures; `CORS` before `Auth` so preflights (no credentials) succeed and error responses carry CORS headers; `Auth` last, closest to the routes.
</details>

### Exercise 6: A `Timing` middleware
Write middleware that adds an `X-Response-Time` header. (Hint: headers must be set *before* the body is written; think about whether that's possible in the "after" phase.)

<details><summary>Solution</summary>

You can't set a header *after* `next.ServeHTTP` has already written the response. To include the duration you need to wrap the `ResponseWriter` and set the header just before the status/body are first written (Chapter 46's technique). A simpler middleware can *log* the duration after `next` returns.
</details>

### Exercise 7 (challenge): A JSON 404 for unknown routes
Implement a middleware that converts the mux's plain-text `404` into `{"error":"route not found"}` **without** affecting the automatic `405`s. Which technique from section 13 does it need?

<details><summary>Solution</summary>

Wrap the `ResponseWriter` in a struct that embeds `http.ResponseWriter` and overrides `WriteHeader`/`Write`: if the status is `404` and the handler hasn't set a JSON `Content-Type`, replace the body with the JSON error (and set the right headers). The wrapper is applied by middleware around the mux, and `405`s pass through untouched.
</details>

---

## 16. Quiz

1. What does the pattern `POST /products` match?
2. What's the difference between `/{$}` and `/`?
3. How do you read the `id` in `GET /products/{id}`?
4. What happens if you register two ambiguous patterns?
5. What does the mux automatically return for the wrong method?
6. Why must `CORS` wrap the mux instead of being a route handler?
7. In `Logger(CORS(mux))`, which runs first on the request? On the response?

<details><summary>Answers</summary>

1. A `POST` request to exactly `/products`.
2. `/{$}` matches only the exact path `/`; `/` matches every path that no other pattern matches.
3. `r.PathValue("id")` (a string; validate and convert).
4. A panic at registration (server start-up), with a descriptive message.
5. `405 Method Not Allowed` with an `Allow` header.
6. Preflight `OPTIONS` requests must be answered for every route (and error responses need CORS headers); wrapping the mux does that once.
7. `Logger` first on the request; on the response, `CORS` finishes first, then `Logger`.
</details>

---

## 17. Summary

- **Go 1.22 patterns:** `"[METHOD ][HOST]/path"` with wildcards `{name}` (one segment), `{name...}` (rest), and `{$}` (exact end). Read values with **`r.PathValue("name")`**: always a string; **validate it**.
- The mux chooses the **most specific** matching pattern; **ambiguous patterns panic at start-up**.
- Method patterns give **automatic `405` + `Allow`**, `HEAD` for every `GET`, and `404` for unknown paths (plain text; see section 13).
- Frameworks (chi, gin, echo) filled the gaps before 1.22; today the standard library often suffices, and `chi` is the natural step up (same `http.Handler` model).
- **Middleware** (`func(http.Handler) http.Handler`) wraps the whole router so cross-cutting rules (CORS/preflight) apply to every route once. **Order matters** (`CORS` before `Auth`), and a middleware can **stop** the chain by responding without calling `next`.
- Our router now: `GET /products`, `POST /products`, `GET /products/{id}`, wrapped by CORS, with handlers stripped of routing code.

### ➡️ What's next?

[Chapter 45](45-building-your-first-real-middleware.md) builds a second middleware, a **request logger**, and starts building a small toolkit to *chain* several middleware together.
