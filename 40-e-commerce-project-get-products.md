# Chapter 40: E-commerce Project — Your First Real Endpoint (`GET /products`)

> **Goal of this chapter:** Start the course project: an **e-commerce backend API** written in Go. You'll create the project, define a `Product` type, keep a small in-memory product list, and build your first real REST endpoint, `GET /products`, which returns JSON. You'll test it with `curl`, learn how the `ResponseWriter` and `Request` objects work, and write your first automated handler test.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 3 hours  **Prerequisite:** [Chapters 21, 37, 38](38-os-or-go-server-the-complete-journey-of-a-request.md)

> 🧭 **How the project chapters work.** Chapters 40–63 build *one* project step by step. Every code block that starts with a `// file: path` comment is a real file of the project at that stage: type them (or copy them) into your project folder. The code in each chapter was compiled and run to produce the outputs shown.

---

## 📚 Table of Contents

1. [What you will build and learn](#1-what-you-will-build-and-learn)
2. [The big picture](#2-the-big-picture)
3. [REST, HTTP methods, and status codes: the 2-minute recap](#3-recap)
4. [Step 1: set up the project](#4-step-1-set-up-the-project)
5. [Step 2: the `Product` type](#5-step-2-the-product-type)
6. [Step 3: some data](#6-step-3-some-data)
7. [Step 4: the handler](#7-step-4-the-handler)
8. [Step 5: the router and the server](#8-step-5-the-router-and-the-server)
9. [The complete code](#9-the-complete-code)
10. [Run and test it](#10-run-and-test-it)
11. [The `ResponseWriter` and `Request`, in depth](#11-responsewriter-and-request-in-depth)
12. [JSON encoding in depth](#12-json-encoding-in-depth)
13. [Writing your first handler test](#13-writing-your-first-handler-test)
14. [A word about browsers and CORS](#14-a-word-about-browsers-and-cors)
15. [Common mistakes](#15-common-mistakes)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will build and learn

**Project:** a backend for an online shop. Over the next chapters it will gain: creating products (POST), routing with path parameters, middleware (logging, CORS, recovery), configuration, JWT authentication, a PostgreSQL database, clean architecture, pagination, and finally concurrency techniques.

**In this chapter:**

- Design and expose a **REST endpoint**
- Model data with a **struct** and JSON **tags**
- Return **JSON** with the right **headers** and **status codes**
- Reject wrong HTTP methods properly (`405 Method Not Allowed`)
- Test the endpoint with `curl` and with Go's `httptest`

---

## 2. The big picture

```
┌────────────────┐     GET /products        ┌────────────────────┐
│   Frontend     │ ───────────────────────► │      Backend       │
│ (React app,    │                          │   (Go, this        │
│  mobile app,   │ ◄─────────────────────── │    project)        │
│  curl, ...)    │   200 OK + JSON list     │                    │
└────────────────┘   of products            └────────────────────┘
```

> The original version of this course used a ready-made React front end to give a "feel" for a real app. **You don't need any front-end knowledge** to be a backend developer, and everything here can be exercised with `curl` or an API client such as Postman, Insomnia, or Bruno. When a front end on another web address talks to your API, the browser adds the CORS rules (section 14 and the next chapters).

---

## 3. Recap

You met these in [Chapter 37](37-into-backend-development.md); here's the minimum you need today.

| HTTP method | Meaning | Body? |
|-------------|---------|-------|
| **GET** | Read data | no |
| **POST** | Create data | yes |
| **PUT** / **PATCH** | Replace / partially update | yes |
| **DELETE** | Remove | no |

| Status | Meaning |
|--------|---------|
| `200 OK` | Success |
| `201 Created` | Something was created |
| `400 Bad Request` | The client sent something invalid |
| `404 Not Found` | No such resource/route |
| `405 Method Not Allowed` | The route exists, but not for this HTTP method |
| `500 Internal Server Error` | Our bug |

Our first endpoint: **`GET /products` → `200` + a JSON array of products**.

---

## 4. Step 1: set up the project

```bash
mkdir ecommerce
cd ecommerce
go mod init ecommerce
```

This creates `go.mod` (Chapter 10):

```
module ecommerce

go 1.22
```

The module name `ecommerce` is the project's identity; import paths for its packages will start with it (`ecommerce/handlers`, ...). Use your editor with the Go extension for formatting and error checking. We need **Go 1.22 or newer**: Chapter 44 uses its new routing features.

---

## 5. Step 2: the `Product` type

A product has an ID, a title, a description, a price, and an image URL. In Go we model it with a **struct** (Chapter 21):

```go
type Product struct {
	ID          int     `json:"id"`
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}
```

Two things to understand:

**1. Field names start with a capital letter: they're *exported*.** The `encoding/json` package lives in a *different* package from ours, so it can only see exported fields (Chapter 10). A lowercase field like `title` would be **silently skipped** in the JSON output.

**2. The backticks part is a *struct tag*.** `` `json:"imageUrl"` `` tells `encoding/json`: "when converting this field to/from JSON, use the key `imageUrl`". Without tags, Go would use the field name as-is: `ImageURL`, but JSON APIs conventionally use `camelCase` (`imageUrl`) or `snake_case`. Tags let Go code follow Go style and the JSON follow the API style.

| Go field | Tag | JSON key |
|----------|-----|----------|
| `ID` | `json:"id"` | `"id"` |
| `Title` | `json:"title"` | `"title"` |
| `ImageURL` | `json:"imageUrl"` | `"imageUrl"` |

(Go style says initialisms like `ID`, `URL` stay uppercase: `ImageURL`, not `ImageUrl`.)

---

## 6. Step 3: some data

There's no database yet (that's Part 11), so we keep the products in a **package-level slice** (Chapter 25) that we fill once at start-up. We use the `init` function (Chapter 14) as a teaching example; in a moment we'll see a simpler alternative.

```go
var productList []Product

func init() {
	productList = []Product{
		{ID: 1, Title: "Orange", Description: "Orange is juicy and full of vitamin C.", Price: 100, ImageURL: "https://example.com/orange.jpg"},
		{ID: 2, Title: "Apple", Description: "A crunchy apple a day...", Price: 40, ImageURL: "https://example.com/apple.jpg"},
		{ID: 3, Title: "Banana", Description: "Great for a quick snack.", Price: 5, ImageURL: "https://example.com/banana.jpg"},
	}
}
```

> 💡 **Simplification.** Because the data is just a literal, you could skip `init` entirely and write `var productList = []Product{ {...}, {...} }`. We use `init` here to practice; later chapters replace this whole thing with a database.

> ⚠️ **Global mutable state** (Chapter 8) is fine for a throw-away in-memory store, but it will bite us in Chapter 41 (concurrent requests writing to it) and we'll fix that properly in Chapters 50–56.

---

## 7. Step 4: the handler

A **handler** is a function with the signature `func(w http.ResponseWriter, r *http.Request)` (Chapter 38). Ours:

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		w.Header().Set("Allow", http.MethodGet)
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(productList)
}
```

Walkthrough:

| Line | What it does |
|------|--------------|
| `r.Method != http.MethodGet` | Is this *not* a GET request? (`http.MethodGet` is the constant `"GET"`; use the constants rather than string literals.) |
| `w.Header().Set("Allow", "GET")` | Tell the client which methods this route *does* support. A correct `405` response includes an `Allow` header |
| `http.Error(w, msg, code)` | Sets the status code, sets `Content-Type: text/plain`, and writes the message: a convenient shortcut for errors |
| `return` | **Stop.** Forgetting `return` after an error response is a classic bug: the rest of the handler would keep running and write more output |
| `Content-Type: application/json` | Tells the client the body is JSON so it parses it correctly |
| `w.WriteHeader(http.StatusOK)` | Sends the status line and headers. (If you never call it, `200` is used automatically on your first `Write`.) |
| `json.NewEncoder(w).Encode(productList)` | Converts the slice to JSON text and writes it straight into the response |

### Order matters: headers → status → body

An HTTP response is sent as **status line → headers → body**. So in code:

```
1. w.Header().Set(...)      ← must come first
2. w.WriteHeader(status)    ← after this, headers are sent and can no longer change
3. w.Write / Encode(...)    ← the body (the first Write implies WriteHeader(200) if you skipped step 2)
```

Setting a header *after* writing the body has no effect.

---

## 8. Step 5: the router and the server

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/products", getProducts)

	fmt.Println("Server running on http://localhost:8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		fmt.Println("Error starting the server:", err)
	}
}
```

- `http.NewServeMux()`: creates the **router** (Chapter 38).
- `mux.HandleFunc("/products", getProducts)`: registers the **route**.
- `http.ListenAndServe(":8080", mux)`: opens port 8080 and serves forever (blocks).

---

## 9. The complete code

```go
// file: main.go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
)

// Product is the data model of the shop.
type Product struct {
	ID          int     `json:"id"`
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}

// productList is our temporary in-memory "database".
var productList []Product

func init() {
	productList = []Product{
		{ID: 1, Title: "Orange", Description: "Orange is juicy and full of vitamin C.", Price: 100, ImageURL: "https://example.com/orange.jpg"},
		{ID: 2, Title: "Apple", Description: "A crunchy apple a day...", Price: 40, ImageURL: "https://example.com/apple.jpg"},
		{ID: 3, Title: "Banana", Description: "Great for a quick snack.", Price: 5, ImageURL: "https://example.com/banana.jpg"},
	}
}

// getProducts handles GET /products.
func getProducts(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		w.Header().Set("Allow", http.MethodGet)
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(productList)
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/products", getProducts)

	fmt.Println("Server running on http://localhost:8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		fmt.Println("Error starting the server:", err)
	}
}
```

Execution flow:

```
go run .
   │
   ├─ package initialization: init() fills productList
   ├─ main(): create mux, register "/products", ListenAndServe blocks
   │
   └─ for each incoming request (in its own goroutine, Chapter 36):
         mux matches the path → getProducts(w, r)
              ├─ wrong method?  → 405 and return
              └─ otherwise      → 200 + JSON
```

---

## 10. Run and test it

```bash
go run .
```

```
Server running on http://localhost:8080
```

In a second terminal (real output):

```bash
curl -i http://localhost:8080/products
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Date: Sat, 26 Sep 2026 06:01:12 GMT
Content-Length: 380

[{"id":1,"title":"Orange","description":"Orange is juicy and full of vitamin C.","price":100,"imageUrl":"https://example.com/orange.jpg"},{"id":2,"title":"Apple","description":"A crunchy apple a day...","price":40,"imageUrl":"https://example.com/apple.jpg"},{"id":3,"title":"Banana","description":"Great for a quick snack.","price":5,"imageUrl":"https://example.com/banana.jpg"}]
```

Ugly? Pipe it through a JSON pretty-printer:

```bash
curl -s http://localhost:8080/products | python3 -m json.tool
# or, if installed:  curl -s http://localhost:8080/products | jq
```

Try the error paths:

```bash
curl -i -X POST http://localhost:8080/products     # wrong method
```

```
HTTP/1.1 405 Method Not Allowed
Allow: GET
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff

method not allowed
```

```bash
curl -i http://localhost:8080/nothing              # unknown route
```

```
HTTP/1.1 404 Not Found

404 page not found
```

### Testing with an API client (Postman, Insomnia, Bruno)

GUI clients let you build requests by clicking: choose the method (`GET`), enter `http://localhost:8080/products`, press **Send**, and see the status code, headers, and pretty-printed JSON. Try changing the method to `POST` and watch the `405`. They're great once requests get complicated (bodies, tokens); `curl` is great for quick checks and scripts.

> ⚠️ **"address already in use"?** Something else holds port 8080 (an old copy of your server, or another program). See Chapter 38, section 16: find it with `ss -ltnp | grep 8080` or `lsof -i :8080`, or change the port.

---

## 11. The `ResponseWriter` and `Request`, in depth

### `w http.ResponseWriter`: your response

It's an **interface** with three key methods:

| Method | Purpose |
|--------|---------|
| `Header() http.Header` | The response headers (a `map[string][]string`) to set *before* writing |
| `WriteHeader(statusCode int)` | Send the status line + headers |
| `Write([]byte) (int, error)` | Write body bytes (implies `WriteHeader(200)` the first time) |

Because it implements `io.Writer`, anything that writes to a writer works with it: `fmt.Fprintf(w, ...)`, `json.NewEncoder(w)`, `io.Copy(w, file)`, `template.Execute(w, data)`.

### `r *http.Request`: everything about the incoming request

| Field/method | Gives you | Example |
|--------------|-----------|---------|
| `r.Method` | HTTP method | `"GET"` |
| `r.URL.Path` | Path | `"/products"` |
| `r.URL.Query()` | Query parameters | `?page=2` → `Get("page") == "2"` |
| `r.Header` | Request headers | `r.Header.Get("Authorization")` |
| `r.Body` | The request body (an `io.ReadCloser`) | for POST/PUT/PATCH (next chapter) |
| `r.Context()` | Cancellation/deadline context | cancelled if the client disconnects |
| `r.RemoteAddr` | Client address | `"[::1]:43566"` |
| `r.PathValue("id")` | Path parameter (Go 1.22+) | Chapter 44 |

Since `r` is a **pointer** (`*http.Request`), the handler and the server share the same request object (Chapter 24). The request itself is treated as read-only by convention.

---

## 12. JSON encoding in depth

### `json.NewEncoder(w).Encode(v)` vs. `json.Marshal(v)`

Two ways to turn a Go value into JSON:

```go
// 1. Encoder: streams JSON directly into the writer; adds a trailing newline
json.NewEncoder(w).Encode(productList)

// 2. Marshal: returns the JSON bytes; you write them yourself
data, err := json.Marshal(productList)
if err != nil {
	http.Error(w, "could not encode products", http.StatusInternalServerError)
	return
}
w.Write(data)
```

`Marshal` lets you handle an encoding error *before* anything is written (you can still send a clean `500`). With `Encoder`, once you've started writing, a mid-stream failure can't change the status code anymore. For simple, safe data both are fine.

### What `encoding/json` does with each type

| Go | JSON |
|----|------|
| `int`, `float64`, ... | number |
| `string` | string (escaped) |
| `bool` | `true` / `false` |
| slice / array | array (`nil` slice → `null`, empty slice → `[]`) |
| struct | object (exported fields only) |
| `map[string]T` | object |
| pointer | the pointed-to value, or `null` if `nil` |
| `time.Time` | RFC 3339 string |

### Useful tag options

```go
type Example struct {
	Public   string  `json:"public"`
	Optional string  `json:"optional,omitempty"` // leave the key out if empty (""/0/false/nil)
	Secret   string  `json:"-"`                  // never include
	Price    float64 `json:"price,string"`       // encode the number as a JSON string
}
```

### `nil` vs. empty slice (a real API gotcha)

```go
var a []Product           // nil slice
b := []Product{}          // empty slice
// json: a → null,  b → []
```

Clients usually expect `[]` for "no products". If your list can be empty, initialize it (`make([]Product, 0)` or a literal), so the API returns `[]` instead of `null`.

### Pretty output while developing

```go
enc := json.NewEncoder(w)
enc.SetIndent("", "  ")
enc.Encode(productList)
```

Don't indent in production: it wastes bytes (clients can pretty-print themselves).

---

## 13. Writing your first handler test

Go can test an HTTP handler **without starting a server**: `net/http/httptest` gives you a fake request and a recording response.

```go
// file: main_test.go
package main

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestGetProducts(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/products", nil)
	rec := httptest.NewRecorder()

	getProducts(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("expected status 200, got %d", rec.Code)
	}
	if ct := rec.Header().Get("Content-Type"); ct != "application/json" {
		t.Fatalf("expected JSON content type, got %q", ct)
	}

	var products []Product
	if err := json.NewDecoder(rec.Body).Decode(&products); err != nil {
		t.Fatalf("response is not valid JSON: %v", err)
	}
	if len(products) != 3 {
		t.Fatalf("expected 3 products, got %d", len(products))
	}
	if products[0].Title != "Orange" {
		t.Errorf("expected first product Orange, got %q", products[0].Title)
	}
}

func TestGetProductsRejectsPOST(t *testing.T) {
	req := httptest.NewRequest(http.MethodPost, "/products", nil)
	rec := httptest.NewRecorder()

	getProducts(rec, req)

	if rec.Code != http.StatusMethodNotAllowed {
		t.Fatalf("expected 405, got %d", rec.Code)
	}
	if allow := rec.Header().Get("Allow"); allow != "GET" {
		t.Errorf("expected Allow: GET, got %q", allow)
	}
}
```

Run:

```bash
go test ./...
```

```
ok  	ecommerce	0.003s
```

(`-v` shows each test; `-race` enables the race detector; `-cover` shows coverage.) Tests like this run in milliseconds and catch regressions as the project grows.

- `httptest.NewRequest` builds a request (no network).
- `httptest.NewRecorder` is a `ResponseWriter` that records the status, headers, and body for you to inspect.
- We call the handler *directly* like any function: another benefit of handlers being plain functions.

---

## 14. A word about browsers and CORS

`curl` will happily talk to your server from anywhere. **Browsers won't**, at least not from a web page served at a *different origin*. If a React app at `http://localhost:5173` runs `fetch("http://localhost:8080/products")`, the browser blocks the response unless the server includes permission headers (`Access-Control-Allow-Origin: ...`). That protection is **CORS** (Cross-Origin Resource Sharing).

The browser console error looks like:

```
Access to fetch at 'http://localhost:8080/products' from origin 'http://localhost:5173'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present...
```

It's a *browser* rule, not a server failure: the request usually reaches your server fine. Fixing it is a **server-side header** job, and we'll do it in the next two chapters ([41](41-e-commerce-project-post-products.md) and [42](42-the-preflight-request-and-the-options-method.md)).

---

## 15. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Lowercase struct fields (`title string`) | JSON has empty `{}` objects | Export fields: `Title` |
| 2 | Forgetting the JSON tags | Keys come out as `ID`, `Title`, ... | Add `` `json:"id"` `` tags |
| 3 | No `return` after `http.Error` | Extra output appended; "superfluous WriteHeader" warnings | Always `return` |
| 4 | Setting headers after writing the body | Header silently ignored | Set headers first |
| 5 | Forgetting `Content-Type: application/json` | Clients may treat the body as text | Set it |
| 6 | Wrong status for a wrong method (`400`/`404`) | Misleading clients | `405` + `Allow` header |
| 7 | Returning `null` for an empty list | Front ends crash on `.map` | Initialize with `[]Product{}` |
| 8 | Ignoring the error from `ListenAndServe` | Server dies silently (port in use) | Print/log the error |
| 9 | Not restarting the server after edits | Old behavior | Stop and `go run .` again (or use a reloader like `air`) |
| 10 | Testing only with a browser | Can't see status/headers, can't send other methods | Use `curl` or an API client |

---

## 16. Exercises

### Exercise 1: Add a product
Add a fourth product ("Mango", price 200) to `productList`. Restart and verify with `curl`.

<details><summary>Solution</summary>

Add `{ID: 4, Title: "Mango", Description: "The king of fruits.", Price: 200, ImageURL: "https://example.com/mango.jpg"}` to the slice in `init`, restart, and `curl -s localhost:8080/products | jq length` should print `4`. (Update the test's expected count too.)
</details>

### Exercise 2: Hide a field
Add a `CostPrice float64` field that must *never* appear in JSON. What tag do you use?

<details><summary>Solution</summary>

```go
CostPrice float64 `json:"-"`
```
The `-` tag excludes the field from encoding and decoding.
</details>

### Exercise 3: Optional field
Add `Discount *float64` that appears only when set. Which tag option, and why a pointer?

<details><summary>Solution</summary>

```go
Discount *float64 `json:"discount,omitempty"`
```
`omitempty` omits the key when the value is empty; for a plain `float64`, `0` would count as empty (indistinguishable from "no discount"), whereas a pointer distinguishes `nil` (absent) from `0` (a real 0% value).
</details>

### Exercise 4: Health check
Add `GET /health` returning `{"status":"ok"}` with status 200.

<details><summary>Solution</summary>

```go
func health(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}
// in main: mux.HandleFunc("/health", health)
```
</details>

### Exercise 5: Use `Marshal`
Rewrite `getProducts` to use `json.Marshal` so that an encoding error produces a clean `500` before any body is written.

<details><summary>Solution</summary>

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		w.Header().Set("Allow", http.MethodGet)
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}
	data, err := json.Marshal(productList)
	if err != nil {
		http.Error(w, "internal server error", http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	w.Write(data)
}
```
</details>

### Exercise 6: Empty list
Make `productList` empty and check the JSON. Do you get `[]` or `null`? Fix it so it's always `[]`.

<details><summary>Solution</summary>

`var productList []Product` (nil) encodes as `null`. Use `productList = []Product{}` (or `make([]Product, 0)`) to get `[]`. You can also normalize in the handler: `if productList == nil { productList = []Product{} }`.
</details>

### Exercise 7 (challenge): Filter by query
Support `GET /products?maxPrice=50`, returning only products whose price is at most 50 (use `r.URL.Query().Get`, `strconv.ParseFloat`, and answer `400` for a bad number).

<details><summary>Solution</summary>

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
	// ...method check...
	result := productList
	if raw := r.URL.Query().Get("maxPrice"); raw != "" {
		max, err := strconv.ParseFloat(raw, 64)
		if err != nil {
			http.Error(w, "maxPrice must be a number", http.StatusBadRequest)
			return
		}
		result = make([]Product, 0)
		for _, p := range productList {
			if p.Price <= max {
				result = append(result, p)
			}
		}
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(result)
}
```
(Import `strconv`.)
</details>

---

## 17. Quiz

1. Why must struct fields be exported for JSON encoding?
2. What does the tag `` `json:"imageUrl"` `` do?
3. Which status code does a wrong HTTP method deserve, and which header goes with it?
4. In what order must you set headers, the status, and the body?
5. What is `httptest.NewRecorder` for?
6. Why can `curl` reach your API but a browser page sometimes can't?

<details><summary>Answers</summary>

1. `encoding/json` is in another package and can only access exported (capitalized) fields.
2. Sets the JSON key name for that field.
3. `405 Method Not Allowed`, with an `Allow` header listing supported methods.
4. Headers → `WriteHeader(status)` → body.
5. It's a fake `ResponseWriter` that records the response so tests can inspect status, headers, and body.
6. Browsers enforce CORS for cross-origin requests; `curl` doesn't.
</details>

---

## 18. Summary

- The project: an **e-commerce API**. First endpoint: **`GET /products` → JSON list**.
- Model data with a **struct** with **exported fields** and **`json` tags**.
- A handler is `func(w http.ResponseWriter, r *http.Request)`: check `r.Method`, set **headers**, write the **status**, then the **body** (in that order), and `return` after errors.
- Use `json.NewEncoder(w).Encode(v)` (or `json.Marshal`) to produce JSON; beware `null` vs `[]`.
- Return `405 Method Not Allowed` (+ `Allow`) for the wrong method.
- Test manually with `curl`/API clients and automatically with `httptest`.
- Browsers add the **CORS** rules for cross-origin calls: coming next.

### ➡️ What's next?

[Chapter 41](41-e-commerce-project-post-products.md) adds `POST /products` (creating data from a JSON request body), meets **CORS** face to face, and covers error handling for bad input.
