# Chapter 41: E-commerce Project — `POST /products` (Creating Data), JSON Decoding, and CORS

> **Goal of this chapter:** Let clients **create** products. You'll read a **JSON request body**, decode it into a Go value, validate it, store it, and respond with `201 Created`. Along the way you'll meet the big topics of real APIs: **input validation**, **error responses**, **request size limits**, **concurrency safety** (two clients creating products at the same moment), and your first encounter with **CORS**, the browser rule that trips up every new backend developer.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 3–4 hours  **Prerequisite:** [Chapter 40](40-e-commerce-project-get-products.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What does POST do?](#2-what-does-post-do)
3. [GET vs. POST](#3-get-vs-post)
4. [The request body](#4-the-request-body)
5. [JSON decoding](#5-json-decoding)
6. [Separate the input type from the model](#6-separate-the-input-type-from-the-model)
7. [Validation](#7-validation)
8. [Status codes for POST](#8-status-codes-for-post)
9. [Concurrency: protect the shared list](#9-concurrency-protect-the-shared-list)
10. [The complete code](#10-the-complete-code)
11. [Testing with `curl`](#11-testing-with-curl)
12. [Automated tests](#12-automated-tests)
13. [CORS: the browser's rule](#13-cors-the-browsers-rule)
14. [Error-handling best practices](#14-error-handling-best-practices)
15. [Common mistakes](#15-common-mistakes)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- What `POST` means and how it differs from `GET`
- How to read and **decode** a JSON request body (`json.NewDecoder`)
- Why to use a **separate input struct** (never trust client-supplied IDs)
- How to **validate** input and answer with `400`/`422` and a JSON error message
- How to limit body size and reject unknown fields
- Why shared in-memory state needs a **mutex** (a first look; details in Chapters 67–68)
- What **CORS** is, and the difference between a *simple request* and a *preflighted* one

---

## 2. What does POST do?

`POST` **sends data to the server so it can create something** (or trigger an action). In REST terms: *"add a new resource to this collection."*

```
POST /products
Content-Type: application/json

{"title": "Mango", "description": "King of fruits", "price": 200, "imageUrl": "https://example.com/mango.jpg"}
```

Expected response: **`201 Created`**, with the new product (including its server-assigned `id`) in the body:

```
HTTP/1.1 201 Created
Location: /products/4
Content-Type: application/json

{"id":4,"title":"Mango", ...}
```

---

## 3. GET vs. POST

| | `GET /products` | `POST /products` |
|--|-----------------|------------------|
| Purpose | **Read** the collection | **Create** a new member |
| Request body | none | **JSON** describing the new item |
| Changes server data? | **No** (safe) | **Yes** |
| Repeating it | Same result (idempotent) | Creates *another* product each time |
| Success status | `200 OK` | `201 Created` |
| Caching | Allowed | Not cached |

Both use the **same URL** (`/products`): the *method* selects the behavior. Our current router registers a *path* only, so for now one handler inspects the method and dispatches. [Chapter 44](44-advanced-routing-go-1-22-and-the-middleware-idea.md) shows Go 1.22's cleaner `mux.HandleFunc("POST /products", ...)`.

---

## 4. The request body

The body of a request is a **stream of bytes** that the client sends after the headers. In Go it's `r.Body`, an `io.ReadCloser` that you read *once*, in order.

Important properties:

- **You read it once.** After reading, it's consumed (unless you saved the bytes).
- The server **doesn't need to close** `r.Body` (the `net/http` server does), but you must not assume the whole body is in memory: it streams.
- **Bodies can be huge or malicious.** Always **limit** how much you're willing to read (section 7): `http.MaxBytesReader`.
- The `Content-Type` header says how to interpret it: `application/json`, `application/x-www-form-urlencoded`, `multipart/form-data` (file uploads), ...

---

## 5. JSON decoding

Decoding is the reverse of encoding (Chapter 40): JSON text → Go value.

```go
var req createProductRequest
if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
	// malformed JSON, wrong types, empty body, ...
}
```

Two important details:

**1. Pass a *pointer*** (`&req`). `Decode` must *write into* your variable (Chapter 24), just like `Scanln(&x)`.

**2. What can go wrong?**

| Situation | The error you get |
|-----------|-------------------|
| Empty body | `io.EOF` |
| Broken JSON: `{"title": ` | `*json.SyntaxError` |
| Wrong type: `"price": "cheap"` | `*json.UnmarshalTypeError` (names the field) |
| Extra fields (with `DisallowUnknownFields`) | `json: unknown field "id"` |
| Body too large (with `MaxBytesReader`) | `*http.MaxBytesError` |

### `Decoder` vs. `Unmarshal`

```go
// Decoder: streams from an io.Reader (ideal for request bodies)
json.NewDecoder(r.Body).Decode(&v)

// Unmarshal: needs all the bytes in memory first
data, _ := io.ReadAll(r.Body)
json.Unmarshal(data, &v)
```

For request bodies, `Decoder` is the idiom. It also lets us configure strictness:

```go
dec := json.NewDecoder(r.Body)
dec.DisallowUnknownFields() // reject typos like {"titel": "..."} instead of silently ignoring them
```

### Case-insensitive matching (a surprise!)

By default `encoding/json` matches keys **case-insensitively**: `"TITLE"`, `"Title"`, and `"title"` all fill `Title`. It's usually harmless, but don't be surprised by it.

---

## 6. Separate the input type from the model

Should clients be allowed to send an `id` when creating a product? **No.** The **server** assigns IDs. If we decoded straight into `Product`, a client could send `{"id": 1, ...}` and try to overwrite product 1 or collide with existing IDs.

The fix is a dedicated **request type** (often called a DTO, *data transfer object*) containing *only* what the client may provide:

```go
type createProductRequest struct {
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}
```

Then the server builds the real `Product`, filling in the `ID` itself. With `DisallowUnknownFields`, a client who sends `"id": 99` gets a clear `400`, not silent acceptance.

> 🔐 **Security principle: never trust the client.** Whitelist the fields you accept (a DTO does exactly that) instead of decoding into your internal model. This prevents **mass assignment** vulnerabilities, where a client sets fields (`isAdmin`, `price`, `id`) they shouldn't control.

---

## 7. Validation

Decoding only proves the JSON is *well-formed and type-compatible*. It says nothing about whether the values make sense. Validate before storing:

```go
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
```

Rules of thumb:

- Validate **on the server**, always, even if the front end validates too (a client can be bypassed).
- Trim whitespace where sensible; enforce length limits; check ranges.
- Return **all** the problems if you can, or at least the first with a clear, specific message.
- Return `400 Bad Request` for malformed input, and `422 Unprocessable Entity` for well-formed but semantically invalid input (both are common; be consistent).

### Limit the body size

```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // at most 1 MiB
```

Without this, a client could stream gigabytes and exhaust your memory. `MaxBytesReader` makes the read fail with an error once the limit is exceeded (and tells the server to close the connection).

---

## 8. Status codes for POST

| Situation | Status | Body |
|-----------|--------|------|
| Created | `201 Created` | the created resource (with its new `id`) + `Location: /products/4` header |
| Malformed JSON / unknown field / wrong types / empty body | `400 Bad Request` | error message |
| Valid JSON, invalid values (`price: -5`) | `422 Unprocessable Entity` | error message |
| Body too large | `413 Request Entity Too Large` | error message |
| Wrong HTTP method | `405 Method Not Allowed` + `Allow` | error message |
| Our bug / storage failure | `500 Internal Server Error` | generic message (log details) |

The `Location` header points at the new resource's URL. Clients (and tools) use it to find the created item.

---

## 9. Concurrency: protect the shared list

Chapter 36 taught that `net/http` runs **each request in its own goroutine**. Our `productList` is a shared package-level variable. What if two clients `POST` at the same time?

```
goroutine A:  productList = append(productList, a)
goroutine B:  productList = append(productList, b)   ← at the same instant
```

Both read the old slice header, both append, both write back: **one product is lost**, or worse, memory is corrupted. That's a **data race** (Chapter 32/36; full treatment in Chapters 67–68).

The fix for now: a **mutex** (mutual-exclusion lock) from the `sync` package:

```go
var (
	mu          sync.RWMutex // guards productList and nextID
	productList []Product
	nextID      int
)
```

- `mu.Lock()` / `mu.Unlock()`: *exclusive* access for **writers**: only one goroutine at a time.
- `mu.RLock()` / `mu.RUnlock()`: *shared* access for **readers**: many readers at once, but never during a write. (`RWMutex` suits read-heavy data like a product list.)

Pattern: lock, do the minimal work, unlock (using `defer` so it can't be forgotten, Chapter 34):

```go
mu.Lock()
defer mu.Unlock()
// ...modify productList / nextID...
```

We'll test for races with `go test -race` in section 12.

---

## 10. The complete code

We keep everything in `main.go` for now, and reorganize in Chapter 43.

```go
// file: main.go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"strings"
	"sync"
)

// Product is the data model of the shop.
type Product struct {
	ID          int     `json:"id"`
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}

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

// The temporary in-memory "database".
var (
	mu          sync.RWMutex // guards productList and nextID
	productList []Product
	nextID      int
)

func resetProducts() {
	mu.Lock()
	defer mu.Unlock()
	productList = []Product{
		{ID: 1, Title: "Orange", Description: "Orange is juicy and full of vitamin C.", Price: 100, ImageURL: "https://example.com/orange.jpg"},
		{ID: 2, Title: "Apple", Description: "A crunchy apple a day...", Price: 40, ImageURL: "https://example.com/apple.jpg"},
		{ID: 3, Title: "Banana", Description: "Great for a quick snack.", Price: 5, ImageURL: "https://example.com/banana.jpg"},
	}
	nextID = 4
}

func init() { resetProducts() }

// ---- small response helpers -------------------------------------------------

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

func writeError(w http.ResponseWriter, status int, message string) {
	writeJSON(w, status, map[string]string{"error": message})
}

// enableCORS lets browser pages from other origins read our responses.
func enableCORS(w http.ResponseWriter) {
	w.Header().Set("Access-Control-Allow-Origin", "*")
}

// describeJSONError turns a decoding error into a message that is safe and helpful for clients.
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

// ---- handlers ---------------------------------------------------------------

// productsHandler dispatches /products by HTTP method.
func productsHandler(w http.ResponseWriter, r *http.Request) {
	enableCORS(w)

	switch r.Method {
	case http.MethodGet:
		getProducts(w, r)
	case http.MethodPost:
		createProduct(w, r)
	default:
		w.Header().Set("Allow", "GET, POST")
		writeError(w, http.StatusMethodNotAllowed, "method not allowed")
	}
}

func getProducts(w http.ResponseWriter, r *http.Request) {
	mu.RLock()
	products := make([]Product, len(productList)) // copy, so we don't encode while others write
	copy(products, productList)
	mu.RUnlock()

	writeJSON(w, http.StatusOK, products)
}

func createProduct(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // at most 1 MiB

	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()

	var req createProductRequest
	if err := dec.Decode(&req); err != nil {
		status, msg := describeJSONError(err)
		writeError(w, status, msg)
		return
	}
	// The body must contain exactly one JSON value, not {...}{...} or trailing junk.
	if err := dec.Decode(&struct{}{}); !errors.Is(err, io.EOF) {
		writeError(w, http.StatusBadRequest, "request body must contain a single JSON object")
		return
	}

	if msg := req.validate(); msg != "" {
		writeError(w, http.StatusUnprocessableEntity, msg)
		return
	}

	mu.Lock()
	product := Product{
		ID:          nextID,
		Title:       strings.TrimSpace(req.Title),
		Description: req.Description,
		Price:       req.Price,
		ImageURL:    req.ImageURL,
	}
	nextID++
	productList = append(productList, product)
	mu.Unlock()

	w.Header().Set("Location", fmt.Sprintf("/products/%d", product.ID))
	writeJSON(w, http.StatusCreated, product)
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/products", productsHandler)

	fmt.Println("Server running on http://localhost:8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		fmt.Println("Error starting the server:", err)
	}
}
```

Read it top to bottom as a story:

1. **Types:** `Product` (the model) and `createProductRequest` (what clients may send) + its `validate` method (Chapter 22).
2. **Storage:** a mutex-protected slice and an ID counter.
3. **Helpers:** `writeJSON`, `writeError` (so every response has a consistent shape), `enableCORS`, `describeJSONError`.
4. **Handlers:** `productsHandler` dispatches by method → `getProducts` / `createProduct`.
5. **`createProduct`:** limit → decode strictly → check single value → validate → store under lock → `201` + `Location`.

A few details worth noting:

- `errors.Is(err, io.EOF)` / `errors.As(err, &target)` are the standard ways to inspect errors (they unwrap wrapped errors).
- `getProducts` **copies** the slice while holding the read lock, then encodes *after* unlocking: we never hold a lock while doing slow network I/O.
- `Status 405` handling now returns the same JSON error shape as everything else.
- `enableCORS` is called for every response (see section 13).

---

## 11. Testing with `curl`

Start the server (`go run .`). Create a product (real output):

```bash
curl -i -X POST http://localhost:8080/products \
  -H "Content-Type: application/json" \
  -d '{"title":"Mango","description":"King of fruits","price":200,"imageUrl":"https://example.com/mango.jpg"}'
```

```
HTTP/1.1 201 Created
Access-Control-Allow-Origin: *
Content-Type: application/json
Location: /products/4
Date: Sat, 26 Sep 2026 06:12:31 GMT
Content-Length: 111

{"id":4,"title":"Mango","description":"King of fruits","price":200,"imageUrl":"https://example.com/mango.jpg"}
```

Now `GET /products` returns four products, the new one last. (Restarting the server resets the list: it's only in memory.)

The error cases:

```bash
# Truncated JSON
curl -s -X POST localhost:8080/products -d '{"title": '
# {"error":"malformed JSON: the body ended unexpectedly"}

# Syntax error (unquoted key)
curl -s -X POST localhost:8080/products -d '{title: "x"}'
# {"error":"malformed JSON at position 2"}

# Empty body
curl -s -X POST localhost:8080/products
# {"error":"request body must not be empty"}

# Missing/empty title → 422
curl -s -X POST localhost:8080/products -d '{"title":"","price":5}'
# {"error":"title is required"}

# Negative price → 422
curl -s -X POST localhost:8080/products -d '{"title":"Kiwi","price":-3}'
# {"error":"price must be greater than 0"}

# Wrong type → 400
curl -s -X POST localhost:8080/products -d '{"title":"Kiwi","price":"cheap"}'
# {"error":"field \"price\" must be of type float64"}

# Client tries to set the ID → 400 (unknown field)
curl -s -X POST localhost:8080/products -d '{"id":1,"title":"Hacked","price":1}'
# {"error":"unknown field \"id\""}

# DELETE isn't supported on /products yet → 405
curl -i -X DELETE localhost:8080/products
# HTTP/1.1 405 Method Not Allowed
# Allow: GET, POST
```

Every failure gives the client a **clear status code and a JSON error body**, the hallmark of a well-behaved API.

---

## 12. Automated tests

```go
// file: main_test.go
package main

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"strings"
	"sync"
	"testing"
)

func doRequest(method, target, body string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()
	productsHandler(rec, req)
	return rec
}

func TestGetProducts(t *testing.T) {
	resetProducts()
	rec := doRequest(http.MethodGet, "/products", "")
	if rec.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rec.Code)
	}
	var products []Product
	if err := json.NewDecoder(rec.Body).Decode(&products); err != nil {
		t.Fatal(err)
	}
	if len(products) != 3 {
		t.Fatalf("expected 3 products, got %d", len(products))
	}
	if rec.Header().Get("Access-Control-Allow-Origin") != "*" {
		t.Error("expected the CORS header")
	}
}

func TestCreateProduct(t *testing.T) {
	resetProducts()
	rec := doRequest(http.MethodPost, "/products", `{"title":"Mango","description":"d","price":200,"imageUrl":"u"}`)
	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d (%s)", rec.Code, rec.Body.String())
	}
	if loc := rec.Header().Get("Location"); loc != "/products/4" {
		t.Errorf("expected Location /products/4, got %q", loc)
	}
	var p Product
	json.NewDecoder(rec.Body).Decode(&p)
	if p.ID != 4 || p.Title != "Mango" {
		t.Errorf("unexpected product %+v", p)
	}

	// it must now appear in the list
	var list []Product
	json.NewDecoder(doRequest(http.MethodGet, "/products", "").Body).Decode(&list)
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
		{"malformed json", `{"title": `, http.StatusBadRequest},
		{"wrong type", `{"title":"x","price":"cheap"}`, http.StatusBadRequest},
		{"unknown field", `{"id":9,"title":"x","price":1}`, http.StatusBadRequest},
		{"two objects", `{"title":"x","price":1}{"title":"y","price":2}`, http.StatusBadRequest},
		{"missing title", `{"price":5}`, http.StatusUnprocessableEntity},
		{"negative price", `{"title":"x","price":-1}`, http.StatusUnprocessableEntity},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			resetProducts()
			rec := doRequest(http.MethodPost, "/products", tc.body)
			if rec.Code != tc.want {
				t.Errorf("expected %d, got %d (%s)", tc.want, rec.Code, rec.Body.String())
			}
		})
	}
}

func TestMethodNotAllowed(t *testing.T) {
	rec := doRequest(http.MethodDelete, "/products", "")
	if rec.Code != http.StatusMethodNotAllowed {
		t.Fatalf("expected 405, got %d", rec.Code)
	}
	if rec.Header().Get("Allow") != "GET, POST" {
		t.Errorf("unexpected Allow header %q", rec.Header().Get("Allow"))
	}
}

// Many goroutines creating products at once must not lose or duplicate any.
func TestConcurrentCreates(t *testing.T) {
	resetProducts()
	const n = 50
	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			doRequest(http.MethodPost, "/products", `{"title":"Concurrent","price":1}`)
		}()
	}
	wg.Wait()

	var list []Product
	json.NewDecoder(doRequest(http.MethodGet, "/products", "").Body).Decode(&list)
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

Run with the **race detector**:

```bash
go test -race ./...
```

```
ok  	ecommerce	1.036s
```

The table-driven test (a slice of `{name, body, want}` cases with `t.Run`) is the idiomatic Go way to cover many inputs compactly. And `TestConcurrentCreates` proves the mutex works: try **removing** `mu.Lock()` from `createProduct` and running `go test -race`; the race detector will report a `DATA RACE` and the test will fail, which is a vivid demonstration of Chapter 67.

---

## 13. CORS: the browser's rule

### The problem

Web browsers enforce the **same-origin policy**: a page loaded from one *origin* can't freely read responses from a *different* origin. An **origin** = **scheme + host + port**:

| URL | Origin |
|-----|--------|
| `http://localhost:5173/app` | `http://localhost:5173` |
| `http://localhost:8080/products` | `http://localhost:8080` ← **different port = different origin** |
| `https://shop.example.com` | `https://shop.example.com` |

A React app served at `localhost:5173` calling your API at `localhost:8080` is a **cross-origin** request. The browser sends it, but **refuses to give the response to the page's JavaScript** unless the server says it's allowed. The console shows:

```
Access to fetch at 'http://localhost:8080/products' from origin 'http://localhost:5173'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

### CORS = Cross-Origin Resource Sharing

CORS is the mechanism by which a **server opts in** to being read across origins, using response headers:

```
Access-Control-Allow-Origin: *                      (any origin may read)
Access-Control-Allow-Origin: http://localhost:5173  (only this origin)
```

That's what our `enableCORS` does (with `*` for development; in production allow only your real front-end origin).

Verify with `curl` by pretending to be a browser page (`Origin` header):

```bash
curl -si -H "Origin: http://localhost:5173" http://localhost:8080/products | grep -i access-control
# Access-Control-Allow-Origin: *
```

### Important facts

- **CORS is enforced by the browser, not the server.** `curl`, Postman, and other servers ignore it. The server merely *declares* its policy in headers.
- The request usually **still reaches your server** and may still *run* (a `POST` might create a product), even if the browser then hides the response. So CORS is **not a security mechanism for your API**; you need authentication and authorization (Chapters 48–49).
- **`*` and credentials:** you can't use `*` when the browser sends cookies/credentials; you must echo a specific origin.

### Simple vs. preflighted requests

Browsers split cross-origin requests into two kinds:

| **Simple request** | **Preflighted request** |
|--------------------|-------------------------|
| Method `GET`, `HEAD`, or `POST` | Any other method (`PUT`, `PATCH`, `DELETE`) |
| Only "safe" headers | Custom headers (e.g., `Authorization`) |
| `Content-Type` only `text/plain`, `application/x-www-form-urlencoded`, `multipart/form-data` | `Content-Type: application/json` |
| Sent directly | The browser first sends an **`OPTIONS`** "preflight" asking permission, and only then the real request |

Notice: **a `POST` with `Content-Type: application/json` (exactly what our create endpoint expects) is NOT a simple request.** So while our `GET /products` now works from a browser page thanks to `Access-Control-Allow-Origin`, a browser-side `fetch(..., {method: "POST", headers: {"Content-Type": "application/json"}})` will first send an `OPTIONS /products` request, and our server currently answers it with `405`, so the browser blocks the POST:

```
Access to fetch at 'http://localhost:8080/products' from origin 'http://localhost:5173' has been blocked by CORS policy:
Response to preflight request doesn't pass access control check: It does not have HTTP ok status.
```

Everything you tested with `curl` worked because `curl` isn't a browser. **Handling the preflight `OPTIONS` request is the next chapter.**

---

## 14. Error-handling best practices

1. **Use the right status code** (400 vs. 422 vs. 404 vs. 500). Clients branch on it.
2. **Return a consistent error shape**, e.g., `{"error": "message"}` (later: also a machine-readable `code`).
3. **Be specific but safe.** Tell the client *what's wrong with their input*. Never leak internals: no stack traces, SQL errors, or file paths in responses. Log those on the server instead.
4. **Validate early, fail fast**, before touching any storage.
5. **Limit resources:** body size now, timeouts later (`http.Server{ReadTimeout: ...}`, Chapter 47).
6. **Never trust the client:** whitelist fields (DTO), validate values, authenticate and authorize.
7. **Don't `panic` on bad input.** Bad input is an *expected* condition, not a bug.
8. **Log server-side errors** with context (request ID, path) so you can debug (Chapters 45–46).

---

## 15. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | `Decode(req)` instead of `Decode(&req)` | `json: Unmarshal(non-pointer ...)` error | Pass a pointer |
| 2 | Decoding into the `Product` model directly | Clients can set `id` (mass assignment) | Use a request DTO |
| 3 | Not limiting the body size | Memory exhaustion attack | `http.MaxBytesReader` |
| 4 | Ignoring the decode error | Silent zero-value products | Check `err`, respond `400` |
| 5 | No validation | `{"title":"","price":-100}` accepted | Validate, respond `422` |
| 6 | Shared slice without a lock | Lost updates, `DATA RACE` | `sync.Mutex`/`RWMutex` |
| 7 | Holding a lock while writing to the network | Slow clients block everyone | Copy under lock, encode after unlocking |
| 8 | Returning `200` for creation | Clients can't tell it was created | `201 Created` + `Location` |
| 9 | Forgetting `Content-Type: application/json` on the request (curl) | Works, but some frameworks reject; be explicit | `-H "Content-Type: application/json"` |
| 10 | Thinking CORS protects the API | Anyone can still call it with `curl` | Use authentication/authorization |
| 11 | Sending the CORS header only on success paths | Errors look like CORS failures in the browser | Set it for *all* responses (middleware, Chapter 43) |
| 12 | Enabling `*` origins in production | Any site can read responses | Allow specific origins |

---

## 16. Exercises

### Exercise 1: Explain the difference
In one paragraph, explain why we decode into `createProductRequest` rather than `Product`.

<details><summary>Solution</summary>

`Product` includes fields the server controls (`ID`), and decoding straight into it would let clients set them (mass assignment). A dedicated input type whitelists exactly what a client may provide, and, with `DisallowUnknownFields`, rejects anything else. It also lets the input shape evolve independently of the internal model.
</details>

### Exercise 2: Add a category
Add an optional `category` field to product creation. Default to `"general"` when omitted. Which struct(s) change?

<details><summary>Solution</summary>

Add `Category string `json:"category"`` to both `Product` and `createProductRequest`. In `createProduct`, use `category := strings.TrimSpace(req.Category); if category == "" { category = "general" }`. Update the tests' expectations if they compare entire products.
</details>

### Exercise 3: Duplicate titles
Reject a new product whose title (case-insensitive) already exists, with `409 Conflict`. Where must the check happen relative to the lock, and why?

<details><summary>Solution</summary>

The check-then-insert must both happen **inside the same `mu.Lock()` critical section**; otherwise two concurrent requests could both pass the check and both insert (a race, "check-then-act"):

```go
mu.Lock()
defer mu.Unlock()
for _, p := range productList {
	if strings.EqualFold(p.Title, title) {
		writeError(w, http.StatusConflict, "a product with this title already exists")
		return
	}
}
// ... append ...
```
(With `defer`, the response is written while holding the lock, which is acceptable for this tiny in-memory case; with a database, uniqueness would be a database constraint, Chapter 55.)
</details>

### Exercise 4: Maximum sizes
Reject `description` values longer than 1000 characters (422). Add a table-driven test case.

<details><summary>Solution</summary>

Add `case len(r.Description) > 1000: return "description must be at most 1000 characters"` to `validate`, and a test case `{"long description", `{"title":"x","price":1,"description":"` + strings.Repeat("a", 1001) + `"}`, 422}`.
</details>

### Exercise 5: Observe CORS
Serve a tiny page from another port (e.g., `python3 -m http.server 5173`) containing `fetch("http://localhost:8080/products").then(r => r.json()).then(console.log)`. What happens **without** `enableCORS`? With it? Then try a JSON `POST` from the page and read the console error.

<details><summary>Solution</summary>

Without the header the browser console shows the "blocked by CORS policy" error for the `GET`. With `Access-Control-Allow-Origin: *`, the GET works. The JSON `POST` triggers a preflight `OPTIONS` that our server answers with `405`, so the console reports the preflight failure: exactly what Chapter 42 fixes.
</details>

### Exercise 6: Return the created list count
Add a `X-Total-Count` response header to `GET /products` containing `len(products)`. Check with `curl -i`.

<details><summary>Solution</summary>

In `getProducts`: `w.Header().Set("X-Total-Count", strconv.Itoa(len(products)))` before `writeJSON` (import `strconv`).
</details>

### Exercise 7 (challenge): A race you can see
Comment out the lock in `createProduct` (`mu.Lock()`/`mu.Unlock()`), run `go test -race -run Concurrent`. Read the report. Which two goroutines/operations does it say raced?

<details><summary>Solution</summary>

The race detector prints `WARNING: DATA RACE` with two stacks: e.g. a *write* at `nextID++` (or `append(productList, ...)`) in one goroutine and a *read/write* of the same variable in another, both inside `createProduct`. Restore the lock and the report disappears.
</details>

---

## 17. Quiz

1. What status code and header signal a successfully created resource?
2. Why pass `&req` to `Decode`?
3. What does `DisallowUnknownFields` protect against?
4. Why is `http.MaxBytesReader` important?
5. Why do we need a mutex around `productList`?
6. What is an "origin"?
7. Why is a JSON `POST` from a browser not a "simple" CORS request?

<details><summary>Answers</summary>

1. `201 Created` with a `Location` header.
2. `Decode` must write into your variable, so it needs its address.
3. Typos and mass-assignment attempts (extra fields are rejected instead of ignored).
4. It prevents clients from sending huge bodies that exhaust memory.
5. Requests run in separate goroutines; concurrent appends to a shared slice race.
6. Scheme + host + port.
7. `Content-Type: application/json` isn't one of the "simple" content types, so the browser sends a preflight `OPTIONS` first.
</details>

---

## 18. Summary

- **`POST /products`** creates a product: read the JSON body with `json.NewDecoder(r.Body).Decode(&req)`, validate, store, and reply **`201 Created`** with the new resource and a **`Location`** header.
- Decode into a **request DTO** (no `id`) and use **`DisallowUnknownFields`**: never trust client-supplied fields.
- Limit input with **`http.MaxBytesReader`**; inspect errors with **`errors.Is/As`**; respond with **`400`** (malformed) / **`422`** (invalid values) / **`405`** (wrong method) using a **consistent JSON error shape**.
- Handlers run **concurrently** → protect shared state with **`sync.RWMutex`** (Chapters 67–68); prove it with `go test -race`.
- **CORS** is a browser policy; servers opt in with `Access-Control-Allow-Origin`. JSON `POST`s from browsers are **preflighted**.
- Test with `curl` and table-driven `httptest` tests.

### ➡️ What's next?

[Chapter 42](42-the-preflight-request-and-the-options-method.md) tackles the **preflight `OPTIONS` request**, the browser's "security guard", so browser front ends can finally send JSON `POST`s (and custom headers like `Authorization`).
