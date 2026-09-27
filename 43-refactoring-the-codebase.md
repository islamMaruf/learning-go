# Chapter 43: Refactoring the Codebase — Clean Code and the Single Responsibility Principle

> **Goal of this chapter:** Our project works, but `main.go` has become a 200-line grab bag: data types, storage, validation, CORS, JSON helpers, and handlers all mixed together. You'll learn to **spot code smells**, apply the **Single Responsibility Principle (SRP)**, and **refactor** the project into focused packages (`models`, `database`, `handlers`, `middleware`, `util`), **without changing its behavior**, using the tests we already have as a safety net.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 3 hours  **Prerequisite:** [Chapters 7, 10, 42](42-the-preflight-request-and-the-options-method.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What is refactoring?](#2-what-is-refactoring)
3. [Sniffing out the problems](#3-sniffing-out-the-problems)
4. [The Single Responsibility Principle](#4-the-single-responsibility-principle)
5. [The target structure](#5-the-target-structure)
6. [Step 1: response helpers (`util`)](#6-step-1-response-helpers)
7. [Step 2: CORS becomes middleware](#7-step-2-cors-becomes-middleware)
8. [Step 3: the data layer (`models`, `database`)](#8-step-3-the-data-layer)
9. [Step 4: handlers](#9-step-4-handlers)
10. [Step 5: `main` just wires things](#10-step-5-main-just-wires-things)
11. [The tests keep passing](#11-the-tests-keep-passing)
12. [Before vs. after](#12-before-vs-after)
13. [Package dependency rules](#13-package-dependency-rules)
14. [What's still not great](#14-whats-still-not-great)
15. [A first look at SOLID](#15-a-first-look-at-solid)
16. [Common mistakes](#16-common-mistakes)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. What you will learn

- What **refactoring** is, and why you must never refactor *without tests*
- How to recognize **code smells**: duplication, long functions, mixed responsibilities, hidden global state
- The **Single Responsibility Principle** applied to packages, files, and functions
- How to split a Go project into packages with **one-way dependencies**
- How **middleware** removes repeated code from handlers
- How a `newRouter()` function makes the whole app testable

---

## 2. What is refactoring?

> **Refactoring** means restructuring existing code to make it cleaner **without changing what it does**.

```
BEFORE: works, but messy      ──refactor──►      AFTER: works exactly the same, but clean
         (behavior B)                                       (behavior B)
```

That "without changing behavior" clause is the whole point. It means:

1. **You need a way to prove behavior didn't change:** automated tests. We wrote tests in Chapters 40–42 precisely for this moment. **Never refactor untested code without adding tests first.**
2. **You refactor in small steps**, running the tests after each. If something breaks, you know which step did it.
3. **You don't add features while refactoring.** Two hats: one for changing structure, one for changing behavior. Never wear both at once.

Why bother, if it already works?

```
Messy code:   6 months later you need a new feature.
              Half a day reading and fearing the code; one hour of real work.
Clean code:   30 minutes to find where the change belongs; one hour of real work.
```

Code is **read far more than it is written**. Structure is a gift to your future self and your teammates: easier to understand, change, debug, and test.

---

## 3. Sniffing out the problems

Here's what our Chapter 42 `main.go` mixed together. Each item is a **code smell**: a hint that structure could be better.

| Smell | Where it shows up | Why it hurts |
|-------|-------------------|--------------|
| **Too many responsibilities in one file** | `main.go` holds types, storage, validation, CORS, JSON parsing, error mapping, handlers, *and* startup | Any change touches the same file; merge conflicts; hard to find things |
| **Mixed levels of abstraction** | `createProduct` handles HTTP details (size limits, status codes), parsing, validation, *and* storage under a lock | Can't reuse or test the parts separately |
| **Cross-cutting concern inside a handler** | `productsHandler` begins with `applyCORS`: every future handler must remember to call it | Forget once → a route that mysteriously fails in the browser |
| **Duplication waiting to happen** | Adding `DELETE`/`PUT` handlers means copying JSON-writing and error code again | Bugs fixed in one copy stay in the others |
| **Hidden global state** | `productList`, `nextID`, `mu` are package-level variables touched from anywhere | Hard to test in isolation, easy to misuse, tight coupling |
| **Naming and layout don't reveal intent** | Everything is `package main` | A newcomer can't see the architecture from the file tree |

Before touching anything, write the *smell list* like this. It turns a vague "this is messy" into concrete, fixable items.

---

## 4. The Single Responsibility Principle

Chapter 7 introduced **SRP**, the "S" of SOLID:

> *A unit of code (function, type, file, package) should have **one job**, and therefore **one reason to change**.*

How to check any unit: **describe it in one sentence without "and".**

| Unit | Description | SRP? |
|------|-------------|------|
| `main.go` (before) | "Defines the product type **and** stores products **and** validates input **and** sets CORS headers **and** serves HTTP..." | ❌ |
| `util.SendData` | "Writes a JSON response with a status code." | ✅ |
| `middleware.CORS` | "Adds CORS headers and answers preflight requests." | ✅ |
| `database` package | "Stores and retrieves products." | ✅ |
| `handlers.CreateProduct` | "Turns an HTTP request into a new product and an HTTP response." | ✅ |
| `main` | "Builds the router and starts the server." | ✅ |

> **Reasons to change** is the sharper way to think about it. If the *CORS policy* changes, only `middleware/cors.go` should need editing. If we switch the *database*, only `database/`. If the *JSON shape* changes, `models`/`handlers`. When one requirement forces edits in five places, responsibilities are tangled.

---

## 5. The target structure

```
ecommerce/
├── go.mod
├── main.go                    ← wiring: build the router, start the server
├── main_test.go               ← black-box tests through the whole router
├── models/
│   └── product.go             ← the Product data type
├── database/
│   └── products.go            ← storage (in memory for now; a real DB later)
├── handlers/
│   └── product.go             ← HTTP handlers for /products (request → response)
├── middleware/
│   └── cors.go                ← CORS policy as reusable middleware
└── util/
    └── response.go            ← JSON response helpers + safe JSON request decoding
```

The rule of thumb: **one folder = one responsibility**, named for *what it does*, so the tree itself documents the architecture.

Package names are short, lowercase, and singular-or-plural by convention (`models`, `handlers`, `middleware`); the important part is that each has a clear purpose.

---

## 6. Step 1: response helpers

We used `writeJSON` and `writeError` in Chapters 41–42. They belong in a **shared, dependency-free** package because *everything* that talks HTTP needs them.

We also move the strict, safe JSON-decoding logic out of `createProduct`: parsing a request body is a general job, not specific to products.

```go
// file: util/response.go
package util

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"strings"
)

// SendData writes v as a JSON response with the given status code.
func SendData(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

// SendError writes {"error": message} with the given status code.
func SendError(w http.ResponseWriter, status int, message string) {
	SendData(w, status, map[string]string{"error": message})
}

// DecodeJSON reads exactly one JSON object from the request body into dst.
// It limits the body to 1 MiB and rejects unknown fields. On failure it writes
// an error response itself and returns false; the caller should simply return.
func DecodeJSON(w http.ResponseWriter, r *http.Request, dst any) bool {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)

	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()

	if err := dec.Decode(dst); err != nil {
		status, msg := describeJSONError(err)
		SendError(w, status, msg)
		return false
	}
	if err := dec.Decode(&struct{}{}); !errors.Is(err, io.EOF) {
		SendError(w, http.StatusBadRequest, "request body must contain a single JSON object")
		return false
	}
	return true
}

func describeJSONError(err error) (status int, message string) {
	var syntaxErr *json.SyntaxError
	var typeErr *json.UnmarshalTypeError
	var tooLarge *http.MaxBytesError

	switch {
	case errors.Is(err, io.EOF):
		return http.StatusBadRequest, "request body must not be empty"
	case errors.Is(err, io.ErrUnexpectedEOF):
		return http.StatusBadRequest, "malformed JSON: the body ended unexpectedly"
	case errors.As(err, &syntaxErr):
		return http.StatusBadRequest, fmt.Sprintf("malformed JSON at position %d", syntaxErr.Offset)
	case errors.As(err, &typeErr):
		return http.StatusBadRequest, fmt.Sprintf("field %q must be of type %s", typeErr.Field, typeErr.Type)
	case errors.As(err, &tooLarge):
		return http.StatusRequestEntityTooLarge, "request body too large"
	case strings.HasPrefix(err.Error(), "json: unknown field"):
		return http.StatusBadRequest, strings.TrimPrefix(err.Error(), "json: ")
	default:
		return http.StatusBadRequest, "invalid JSON"
	}
}
```

Notes:

- Exported names (`SendData`, `SendError`, `DecodeJSON`) start with capitals because *other packages* call them (Chapter 10); `describeJSONError` is an internal detail and stays lowercase. A package's exported names are its **public API**: keep it small.
- `DecodeJSON` returns a `bool` and **writes the error response itself**, so each handler needs just `if !util.DecodeJSON(w, r, &req) { return }`.
- `any` is Go's spelling of `interface{}` (Chapter 51).

---

## 7. Step 2: CORS becomes middleware

In Chapter 42, `productsHandler` began by calling `applyCORS`. That's a **cross-cutting concern**: it applies to *every* route, not just products. Repeating a call in every handler is error-prone.

**Middleware** = a function that **wraps a handler** to add behavior before and/or after it, then calls the next handler:

```
request ──► [ CORS middleware ] ──► [ mux → your handler ] ──► response
```

The signature is always `func(http.Handler) http.Handler`:

```go
// file: middleware/cors.go
package middleware

import "net/http"

// allowedOrigins is the allow-list of front-end origins that may call the API from a browser.
var allowedOrigins = map[string]bool{
	"http://localhost:5173": true, // Vite/React dev server
	"http://localhost:3000": true,
}

const (
	allowedMethods = "GET, POST, PUT, PATCH, DELETE, OPTIONS"
	allowedHeaders = "Content-Type, Authorization"
)

// CORS adds CORS headers for allowed origins and answers preflight requests itself.
func CORS(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		origin := r.Header.Get("Origin")

		if origin != "" && allowedOrigins[origin] {
			h := w.Header()
			h.Set("Access-Control-Allow-Origin", origin)
			h.Add("Vary", "Origin")

			isPreflight := r.Method == http.MethodOptions && r.Header.Get("Access-Control-Request-Method") != ""
			if isPreflight {
				h.Set("Access-Control-Allow-Methods", allowedMethods)
				h.Set("Access-Control-Allow-Headers", allowedHeaders)
				h.Set("Access-Control-Max-Age", "600")
				w.WriteHeader(http.StatusNoContent)
				return // preflight fully answered: do NOT call the next handler
			}
		}

		next.ServeHTTP(w, r) // continue down the chain
	})
}
```

How to read it (this shape is the key to all middleware, revisited in Chapters 45–46):

```go
func CORS(next http.Handler) http.Handler {   // takes the handler to wrap...
	return http.HandlerFunc(func(w, r) {      // ...returns a NEW handler
		// (1) work BEFORE the wrapped handler
		next.ServeHTTP(w, r)                  // (2) call the wrapped handler
		// (3) work AFTER it (none here)
	})
}
```

It's a **higher-order function** (Chapter 17) that returns a **closure** (Chapter 20) capturing `next`. The middleware may also **short-circuit**: for a preflight it answers and returns *without* calling `next`.

Now *no handler ever mentions CORS*. Also, CORS headers are added even for `404` responses from the mux and any future route: exactly the completeness Chapter 42's mistake #4 warned about.

> `http.HandlerFunc(f)` converts a plain function with the handler signature into a value that implements the `http.Handler` interface (it has a `ServeHTTP` method that just calls `f`). Chapter 51 explains the trick.

---

## 8. Step 3: the data layer

### The model

```go
// file: models/product.go
package models

// Product is the data model of the shop.
type Product struct {
	ID          int     `json:"id"`
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}
```

A **model** is a plain data type shared by several layers. It has no behavior tied to HTTP or storage.

### The storage

Everything about *keeping* products (the slice, the ID counter, the lock) moves into one package. The rest of the app only sees three functions, so the storage *inside* can change later (a real database in Chapter 52) without touching handlers, provided those function signatures stay.

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
```

The mutex is now an **implementation detail** of this package (unexported, lowercase). No other package can forget to lock, because they can't touch the variables at all. That's **encapsulation** (Chapter 10) doing real work.

---

## 9. Step 4: handlers

A **handler** should be thin: translate *HTTP → domain action → HTTP*. It shouldn't know how storage locks or how JSON errors are worded.

```go
// file: handlers/product.go
package handlers

import (
	"ecommerce/database"
	"ecommerce/models"
	"ecommerce/util"
	"fmt"
	"net/http"
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

// Products dispatches /products by HTTP method.
// (Chapter 44 replaces this with Go 1.22 method-aware routes.)
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
```

Compare `CreateProduct` to the Chapter 42 version: it went from ~35 lines of mixed concerns to ~20 lines that read like a checklist: **decode → validate → store → respond**. Each step is one call.

---

## 10. Step 5: `main` just wires things

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
// Keeping this separate from main() lets tests exercise the whole app without starting a server.
func newRouter() http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("/products", handlers.Products)

	return middleware.CORS(mux) // wrap the whole router in the CORS middleware
}

func main() {
	fmt.Println("Server running on http://localhost:8080")
	if err := http.ListenAndServe(":8080", newRouter()); err != nil {
		fmt.Println("Error starting the server:", err)
	}
}
```

`main` shrank to *"build the app, run it"*. `newRouter` composes the pieces: routes registered on a mux, wrapped by middleware. Adding a route later is one line here plus a handler in `handlers/`.

---

## 11. The tests keep passing

Our safety net is the same set of behaviors we verified in Chapter 42: preflight handling, CORS headers, create/list, validation errors, concurrency. The only change is the *entry point*: instead of calling `productsHandler` directly, tests now go through **`newRouter()`**, the same handler the real server uses (a truer "end-to-end" test):

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

func TestCreateProduct(t *testing.T) {
	database.ResetProducts()
	rec := do(http.MethodPost, "/products", `{"title":"Mango","description":"d","price":200,"imageUrl":"u"}`, nil)
	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d (%s)", rec.Code, rec.Body.String())
	}
	if loc := rec.Header().Get("Location"); loc != "/products/4" {
		t.Errorf("Location = %q", loc)
	}

	var list []product
	json.NewDecoder(do(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	if len(list) != 4 {
		t.Errorf("expected 4 products after create, got %d", len(list))
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
		{"two objects", `{"title":"x","price":1}{"title":"y","price":2}`, http.StatusBadRequest},
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

func TestMethodNotAllowed(t *testing.T) {
	rec := do(http.MethodDelete, "/products", "", nil)
	if rec.Code != http.StatusMethodNotAllowed {
		t.Fatalf("expected 405, got %d", rec.Code)
	}
	if got := rec.Header().Get("Allow"); got != "GET, POST, OPTIONS" {
		t.Errorf("Allow = %q", got)
	}
}

func TestPreflightFromAllowedOrigin(t *testing.T) {
	rec := do(http.MethodOptions, "/products", "", map[string]string{
		"Origin":                         "http://localhost:5173",
		"Access-Control-Request-Method":  "POST",
		"Access-Control-Request-Headers": "content-type,authorization",
	})
	if rec.Code != http.StatusNoContent {
		t.Fatalf("expected 204, got %d", rec.Code)
	}
	h := rec.Header()
	if h.Get("Access-Control-Allow-Origin") != "http://localhost:5173" {
		t.Errorf("Allow-Origin = %q", h.Get("Access-Control-Allow-Origin"))
	}
	if !strings.Contains(h.Get("Access-Control-Allow-Headers"), "Authorization") {
		t.Errorf("Allow-Headers = %q", h.Get("Access-Control-Allow-Headers"))
	}
	if rec.Body.Len() != 0 {
		t.Errorf("preflight must have no body")
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

	ok := do(http.MethodPost, "/products", `{"title":"Kiwi","price":30}`, origin)
	bad := do(http.MethodPost, "/products", `{"title":""}`, origin)
	missing := do(http.MethodGet, "/no-such-route", "", origin) // a 404 produced by the mux itself

	for name, rec := range map[string]*httptest.ResponseRecorder{"created": ok, "invalid": bad, "404": missing} {
		if rec.Header().Get("Access-Control-Allow-Origin") != "http://localhost:5173" {
			t.Errorf("%s response is missing the CORS header", name)
		}
	}
	if missing.Code != http.StatusNotFound {
		t.Errorf("expected 404, got %d", missing.Code)
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
	seen := map[int]bool{}
	for _, p := range list {
		if seen[p.ID] {
			t.Fatalf("duplicate ID %d", p.ID)
		}
		seen[p.ID] = true
	}
}
```

```bash
go vet ./... && go test -race ./...
```

```
ok  	ecommerce	1.016s
?   	ecommerce/database	[no test files]
?   	ecommerce/handlers	[no test files]
?   	ecommerce/middleware	[no test files]
?   	ecommerce/models	[no test files]
?   	ecommerce/util	[no test files]
```

Green. The refactoring changed the *structure* and left the *behavior* alone. Notice the new test `TestCORSHeaderOnRealAndErrorResponses` exercises something the old design could not guarantee: **the mux's own 404** now carries the CORS header, because the middleware wraps the whole router.

You can also confirm with `curl` that the running server behaves identically to Chapter 42's (preflight `204`, create `201`, etc.).

> ✅ **Good practice:** a refactoring commit contains *no behavior change* and *no test change* (apart from entry points). If you must change a test to make the refactor pass, you probably changed behavior.

---

## 12. Before vs. after

| | Chapter 42 | Chapter 43 |
|--|-----------|-----------|
| Files | 1 (`main.go`, ~200 lines) | 6 small files across 5 focused packages |
| `createProduct` | ~35 lines, five concerns | ~20 lines: decode → validate → store → respond |
| CORS | Called manually inside a handler | **Middleware**: automatic, everywhere, including 404s |
| Locking | Visible to all code in `main` | Hidden inside `database` |
| JSON helpers | In `main` | In `util`, reusable |
| Adding a route | Edit the giant `main.go` | New handler file + one line in `main.go` |
| Testing | Handlers only | Whole app via `newRouter()` |
| Reading the code | Read everything to understand anything | Read the tree: it tells you the architecture |

Lines of code barely changed; **cognitive load** dropped enormously.

---

## 13. Package dependency rules

Go forbids **import cycles** (Chapter 10), and good design goes further: dependencies should form a clear, **one-directional** graph.

```
              main
             /    \
     middleware   handlers
                  /  |   \
            database util  models
               │
             models
```

- `main` depends on `handlers` and `middleware` (it wires them).
- `handlers` depends on `database`, `util`, `models`.
- `database` depends on `models`.
- `util` and `middleware` depend on **nothing in our project**: they're leaves. Self-contained utilities like these are the easiest to reuse and test.
- **Nothing depends on `main`** (it can't be imported anyway).

Arrows always point **toward more stable, lower-level code**. If you ever want `database` to import `handlers`, stop: that's an upward dependency, a sign that something is in the wrong place.

---

## 14. What's still not great

Refactoring is iterative. We improved a lot, but honest self-review finds:

1. **Global state.** `database` keeps package-level variables; `handlers` call it directly. Handlers are **tightly coupled** to the concrete storage: to test a handler with fake storage, or to swap in PostgreSQL, we must edit the handler. → **Chapter 50** ("removing tight coupling") and **51** (interfaces) fix this with **dependency injection**.
2. **Routing by hand.** The `switch r.Method` in `Products` is boilerplate. → **Chapter 44** (Go 1.22 method-aware routing).
3. **Middleware is a one-off.** We have one middleware and no clean way to *compose* several (logging, auth, recovery). → **Chapters 45–46**.
4. **Configuration is hard-coded** (port, allowed origins). → **Chapter 47**.
5. **No authentication.** Anyone can create products. → **Chapters 48–49**.

Good engineers refactor *as they go* in small steps, not in a giant rewrite at the end.

---

## 15. A first look at SOLID

SRP is the "S" in **SOLID**, five principles of object-oriented (and Go) design. Preview:

| Letter | Principle | One-line meaning | Where we use it |
|--------|-----------|------------------|-----------------|
| **S** | Single Responsibility | One job, one reason to change | This chapter |
| **O** | Open/Closed | Open for extension, closed for modification: add features by adding code, not editing tested code | Middleware chains (Ch. 45–46) |
| **L** | Liskov Substitution | Any implementation of an interface can be swapped in without surprises | Interfaces (Ch. 51) |
| **I** | Interface Segregation | Prefer small, focused interfaces | Ch. 51 |
| **D** | Dependency Inversion | Depend on abstractions (interfaces), not concrete details | Ch. 50–51 |

You've already *used* O with middleware: to add CORS we didn't edit any handler; we wrapped the router.

---

## 16. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Refactoring without tests | You can't prove nothing broke | Write/keep tests first |
| 2 | Big-bang refactor (everything at once) | Broken build for hours; can't find the cause | Small steps; run tests after each |
| 3 | Changing behavior "while I'm here" | Bugs hidden inside a "refactor" | Separate commits: refactor vs. feature |
| 4 | Package named `utils`/`common`/`helpers` stuffed with unrelated things | A new junk drawer | Name by purpose (`util` here is small and focused on HTTP responses; split it when it grows) |
| 5 | Circular imports | `import cycle not allowed` | Keep dependencies one-directional; extract shared types |
| 6 | Exporting everything | Huge public API, hard to change | Export only what other packages need |
| 7 | Over-engineering (interfaces and layers for a 50-line app) | More code, no benefit | Refactor toward pain you actually feel |
| 8 | Renaming without updating imports/tests | Compile errors | Let the compiler/`gopls` guide; `go build ./...` |
| 9 | Middleware forgetting to call `next` (or calling it after writing a response) | Requests hang or double-write | Short-circuit deliberately and `return` |
| 10 | Splitting by *type* (`controllers`, `models`) forever | Features scattered across many folders | Later chapters group by *feature/domain* (Ch. 50, 59–61) |

---

## 17. Exercises

### Exercise 1: Spot the smells
List four smells in this function and say which SRP violation each represents.

```go
func createUser(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "*")
	body, _ := io.ReadAll(r.Body)
	var u User
	json.Unmarshal(body, &u)
	if u.Email == "" { w.WriteHeader(400); return }
	db.Exec("INSERT INTO users ...", u.Email)
	smtp.SendMail(..., "Welcome!")
	w.WriteHeader(201)
	fmt.Fprintf(w, `{"id": %d}`, id)
}
```

<details><summary>Solution</summary>

(1) CORS inside a handler (cross-cutting concern → middleware). (2) Ignored errors from `ReadAll`/`Unmarshal` and hand-parsing the body (should use a shared decoder). (3) Validation, database access, and email sending in one function (mixed responsibilities → separate layers/services). (4) Building JSON with `Fprintf` (use a response helper/encoder), plus no `Content-Type` and inconsistent error shape.
</details>

### Exercise 2: Move a route
Add a `GET /health` handler returning `{"status":"ok"}`. Which files change? Which *don't*?

<details><summary>Solution</summary>

Add `handlers/health.go` with `func Health(w, r) { util.SendData(w, 200, map[string]string{"status": "ok"}) }` and one line in `newRouter`: `mux.HandleFunc("/health", handlers.Health)`. `database`, `models`, `util`, `middleware` stay untouched, thanks to SRP. (CORS automatically applies.)
</details>

### Exercise 3: A logging middleware
Write `middleware.Logger(next http.Handler) http.Handler` that prints `METHOD PATH` for each request, and wrap the router with it (`Logger(CORS(mux))`). (Chapter 45 does this properly.)

<details><summary>Solution</summary>

```go
package middleware

import (
	"log"
	"net/http"
)

func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Printf("%s %s", r.Method, r.URL.Path)
		next.ServeHTTP(w, r)
	})
}
// newRouter: return middleware.Logger(middleware.CORS(mux))
```
</details>

### Exercise 4: Dependency check
`database` needs to log a message. A teammate suggests importing `handlers` for its logging helper. Why is that a bad idea, and what's better?

<details><summary>Solution</summary>

`handlers` already imports `database`, so `database` importing `handlers` creates an **import cycle** (compile error) and points the dependency the wrong way (low-level → high-level). Put the logging helper in a leaf package (e.g., `util` or a `logging` package) that both can import, or use the standard `log` package.
</details>

### Exercise 5: Add a `Category`
Add an optional `category` field end-to-end. List each file you touch, and argue that each change is small and local.

<details><summary>Solution</summary>

`models/product.go` (field + tag), `handlers/product.go` (add to `createProductRequest`, default to `"general"`, pass to `models.Product`). `database`, `util`, `middleware`, `main` need **no** changes (the database stores whole `models.Product` values). Two files, a few lines.
</details>

### Exercise 6: A unit test for `util`
Write a test for `util.DecodeJSON` using `httptest` that verifies an unknown field yields a `400`.

<details><summary>Solution</summary>

```go
package util

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestDecodeJSONRejectsUnknownFields(t *testing.T) {
	req := httptest.NewRequest(http.MethodPost, "/", strings.NewReader(`{"a":1,"b":2}`))
	rec := httptest.NewRecorder()

	var dst struct{ A int `json:"a"` }
	if ok := DecodeJSON(rec, req, &dst); ok {
		t.Fatal("expected DecodeJSON to fail")
	}
	if rec.Code != http.StatusBadRequest {
		t.Errorf("expected 400, got %d", rec.Code)
	}
}
```
Being able to test `DecodeJSON` in isolation is a payoff of extracting it.
</details>

### Exercise 7 (challenge): Draw your own tree
Sketch where you'd put `PUT /products/{id}` and `DELETE /products/{id}` (handlers, storage functions, any new helper), without writing the code.

<details><summary>Solution</summary>

`handlers/product.go`: `UpdateProduct`, `DeleteProduct` (decode/validate/respond). `database/products.go`: `UpdateProduct(id, p) (models.Product, bool)` and `DeleteProduct(id) bool` (under `mu.Lock()`, returning "not found" as `false`). `util`: maybe a helper to parse the `{id}` path value. `main.go`: two route lines (Chapter 44's method+path patterns).
</details>

---

## 18. Quiz

1. What is refactoring, and what must not change?
2. Why should refactoring be backed by tests?
3. State the Single Responsibility Principle in one sentence.
4. What's a "cross-cutting concern", and how does middleware address it?
5. What is the signature of a middleware function?
6. Why is `newRouter()` useful for testing?
7. Which packages in our tree are "leaves"?

<details><summary>Answers</summary>

1. Restructuring code to improve it; behavior must not change.
2. Tests prove behavior stayed the same after each step.
3. A unit of code should have one job/one reason to change.
4. Something that applies to many routes (CORS, logging, auth); middleware wraps handlers to apply it once, in one place.
5. `func(http.Handler) http.Handler`.
6. It returns the whole app as a handler, so tests can exercise it without opening a port.
7. `util` and `middleware` (and `models`) depend on nothing else in the project.
</details>

---

## 19. Summary

- **Refactoring** improves structure **without changing behavior**. Do it in **small steps** with **tests** as the safety net.
- Identify **code smells** (mixed responsibilities, duplication, cross-cutting logic in handlers, hidden global state) before changing anything.
- **SRP:** one job, one reason to change: describe each unit in a sentence without "and".
- New structure: `models` (data), `database` (storage), `handlers` (HTTP ↔ actions), `middleware` (cross-cutting), `util` (shared helpers), `main` (wiring).
- **Middleware** `func(http.Handler) http.Handler` wraps a router or handler to add behavior everywhere, including 404s.
- `newRouter()` builds the whole app for both `main` and tests; **dependencies flow one way**.
- Remaining smells (global state, hand-written routing, single middleware, hard-coded config) are the agenda of the next chapters.

### ➡️ What's next?

[Chapter 44](44-advanced-routing-go-1-22-and-the-middleware-idea.md) upgrades routing with **Go 1.22's method-aware patterns and path parameters** (`GET /products/{id}`), and starts turning our one-off CORS wrapper into a proper middleware pipeline.
