# Chapter 42: The Preflight Request and the `OPTIONS` Method — The Browser's Security Guard

> **Goal of this chapter:** Finish what Chapter 41 started. Browsers won't send a JSON `POST` (or a `DELETE`, or a request carrying an `Authorization` header) to another origin until they've **asked permission** with an automatic `OPTIONS` request called a **preflight**. You'll learn why it exists, exactly what the browser sends and what the server must answer, and you'll implement a correct, configurable CORS policy (an allow-list of origins, methods, and headers), then verify it with a real browser.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 3 hours  **Prerequisite:** [Chapter 41](41-e-commerce-project-post-products.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The `OPTIONS` method](#2-the-options-method)
3. [What is a preflight request?](#3-what-is-a-preflight-request)
4. [Simple vs. preflighted requests](#4-simple-vs-preflighted-requests)
5. [Why preflight exists](#5-why-preflight-exists)
6. [Anatomy of a preflight exchange](#6-anatomy-of-a-preflight-exchange)
7. [The failure, reproduced](#7-the-failure-reproduced)
8. [Implementing preflight on the server](#8-implementing-preflight-on-the-server)
9. [The complete code](#9-the-complete-code)
10. [Custom headers and `Authorization`](#10-custom-headers-and-authorization)
11. [Testing the policy with `curl`](#11-testing-the-policy-with-curl)
12. [Testing with a real browser](#12-testing-with-a-real-browser)
13. [Automated tests](#13-automated-tests)
14. [Caching preflights: `Access-Control-Max-Age`](#14-caching-preflights)
15. [Security notes](#15-security-notes)
16. [Common mistakes](#16-common-mistakes)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. What you will learn

- What the **`OPTIONS`** HTTP method is
- What a **preflight request** is, *who* sends it, and *when*
- Which requests are "simple" and which trigger a preflight
- The exact **request and response headers** of a preflight
- How to implement a proper **CORS allow-list** in Go
- Why `Access-Control-Allow-Headers` must list `Authorization` (needed for JWT in Chapter 48)
- How to debug CORS errors

---

## 2. The `OPTIONS` method

`OPTIONS` is an HTTP method that asks a server: **"What are you willing to do for me on this URL?"** (which methods, which headers, etc.). It has **no side effects** (it's *safe*) and normally no body. A plain-HTTP use looks like this:

```bash
curl -i -X OPTIONS http://localhost:8080/products
```

A server might answer with an `Allow: GET, POST, OPTIONS` header. In practice, though, the overwhelmingly common use of `OPTIONS` today is **CORS preflight**, sent *automatically by browsers*, and that's what this chapter is about.

---

## 3. What is a preflight request?

> A **preflight request** is an `OPTIONS` request that the **browser sends by itself**, *before* the real request, to check whether the server permits a cross-origin call with that method and those headers.

```
Your JavaScript:   fetch("http://localhost:8080/products", { method: "POST", ... })
                        │
   The browser   ───────┤   (1) OPTIONS /products   "May I POST with a JSON body from origin :5173?"
   (automatic!)         │   ◄── (2) 204 No Content  "Yes: you may POST, with these headers"
                        │
                        └─► (3) POST /products      the REAL request (only if step 2 said yes)
```

- **You don't write this call.** The browser does; your JavaScript never sees the preflight.
- **The server must answer the `OPTIONS`** with the right headers. Your normal handler logic (`POST` → create a product) **doesn't run** for the preflight.
- If the preflight fails, the browser **never sends the real request** and reports a CORS error in the console.

> **Analogy: the security guard.** Before letting a visitor (the real request) into the building, the guard (browser) phones ahead: *"A visitor from origin X wants to do Y with items Z. Is that OK?"* Only if the building (server) says yes does the visitor go in.

---

## 4. Simple vs. preflighted requests

A cross-origin request is **simple** (no preflight) only if **all** of these are true:

| Requirement | Simple |
|-------------|--------|
| Method | `GET`, `HEAD`, or `POST` |
| Headers | Only "safelisted" ones (`Accept`, `Accept-Language`, `Content-Language`, `Content-Type`*) |
| `Content-Type` | One of `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain` |
| No event listeners on upload, no streams | ✔ |

Otherwise it is **preflighted**. Modern JSON APIs trigger it constantly:

| Request | Preflight? | Why |
|---------|-----------|-----|
| `GET /products` (no custom headers) | ❌ No | simple |
| `POST /products` with `Content-Type: application/json` | ✅ **Yes** | JSON content type isn't safelisted |
| `POST` with form-encoded body | ❌ No | simple content type |
| `PUT`, `PATCH`, `DELETE` | ✅ Yes | non-simple method |
| Any request with `Authorization` | ✅ Yes | custom/non-safelisted header |
| Any request with `X-Something` header | ✅ Yes | custom header |

**Key insight for our project:** almost everything a real front end does (JSON `POST`, `PUT`, `DELETE`, requests with a `Bearer` token) **will be preflighted**.

---

## 5. Why preflight exists

Why does the browser bother with this dance? **To protect old servers.**

Before CORS existed, browsers already allowed *some* cross-origin requests: a `<form method="POST">` or an `<img src>` could hit any site. Servers were built with that in mind: they assumed cross-origin requests could only be `GET`/`POST` with simple content types.

CORS added powerful new abilities: JSON bodies, `PUT`/`DELETE`, custom headers. Sending one of *those* to an old server that never anticipated them could be harmful (a malicious page could `DELETE` something on a legacy intranet server that never expected cross-origin deletes). So the browser **asks first**. A server that has never heard of CORS won't answer the preflight properly, and the dangerous request is never sent. Only a server that has *opted in* (by answering the preflight) receives the powerful request.

```
Old server, no CORS support:
   preflight OPTIONS ─►  405 / no CORS headers  ─►  browser: "nope" → dangerous request NEVER SENT ✓

Modern server, CORS-aware:
   preflight OPTIONS ─►  204 + Allow-* headers  ─►  browser: "OK" → real request sent ✓
```

---

## 6. Anatomy of a preflight exchange

### The browser's preflight request

```
OPTIONS /products HTTP/1.1
Host: localhost:8080
Origin: http://localhost:5173
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type,authorization
```

| Header | Meaning |
|--------|---------|
| `Origin` | The page's origin making the request |
| `Access-Control-Request-Method` | The method the **real** request will use |
| `Access-Control-Request-Headers` | The (non-safelisted) headers the real request will carry |

### The server's answer

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 600
Vary: Origin
```

| Header | Meaning |
|--------|---------|
| `Access-Control-Allow-Origin` | Which origin may proceed (`*` = any, or echo the specific origin) |
| `Access-Control-Allow-Methods` | Methods allowed cross-origin |
| `Access-Control-Allow-Headers` | Request headers allowed cross-origin |
| `Access-Control-Max-Age` | Seconds the browser may **cache** this permission (skips repeat preflights) |
| `Vary: Origin` | Tells caches that the response depends on the `Origin` header |
| Status | `204 No Content` (or `200`): any **2xx** counts as OK. There's no body needed |

The browser compares the request against those lists. If the method and every requested header are allowed → the real request goes out. Then the **actual response** must *also* carry `Access-Control-Allow-Origin`, or the browser won't hand it to your JavaScript.

```
BROWSER                                                     SERVER
  │ OPTIONS /products                                          │
  │ Origin: http://localhost:5173                              │
  │ Access-Control-Request-Method: POST                        │
  │ Access-Control-Request-Headers: content-type               │
  │ ──────────────────────────────────────────────────────────►│
  │                                                            │ (checks the origin/method/headers)
  │ 204 No Content                                             │
  │ Access-Control-Allow-Origin: http://localhost:5173         │
  │ Access-Control-Allow-Methods: GET, POST, ..., OPTIONS      │
  │ Access-Control-Allow-Headers: Content-Type, Authorization  │
  │ ◄──────────────────────────────────────────────────────────│
  │ (browser: allowed!)                                        │
  │ POST /products   Content-Type: application/json  {...}     │
  │ ──────────────────────────────────────────────────────────►│
  │ 201 Created  Access-Control-Allow-Origin: http://localhost:5173
  │ ◄──────────────────────────────────────────────────────────│
```

---

## 7. The failure, reproduced

With the Chapter 41 server, your `curl` tests pass but a browser `POST` fails. Reproduce the preflight yourself with `curl` by acting like the browser:

```bash
curl -i -X OPTIONS http://localhost:8080/products \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type"
```

Chapter 41's server (which only knew `GET`/`POST`) answers:

```
HTTP/1.1 405 Method Not Allowed
Access-Control-Allow-Origin: *
Allow: GET, POST
...
{"error":"method not allowed"}
```

`405` is not a 2xx status, so the preflight **fails**. It also lacks `Access-Control-Allow-Methods`/`-Headers`. The browser says:

```
Access to fetch at 'http://localhost:8080/products' from origin 'http://localhost:5173' has been
blocked by CORS policy: Response to preflight request doesn't pass access control check:
It does not have HTTP ok status.
```

---

## 8. Implementing preflight on the server

We'll replace the blunt `Access-Control-Allow-Origin: *` with a proper **policy**:

```go
var allowedOrigins = map[string]bool{
	"http://localhost:5173": true, // our React/Vite dev server
	"http://localhost:3000": true,
}
```

The logic for every request:

1. Read the `Origin` header.
2. If the origin **is on the allow-list**: reply with `Access-Control-Allow-Origin: <that origin>` (echo it) and `Vary: Origin`.
3. If the request is a **preflight** (`OPTIONS` + an `Access-Control-Request-Method` header): also send `Allow-Methods`, `Allow-Headers`, `Max-Age`, and answer **`204`** *without* running any business logic.
4. If the origin is **not** on the list: send **no CORS headers** at all. The browser will block it. (Non-browser clients aren't affected.)

Why *echo the origin* rather than `*`? (a) It's the only way to allow **credentials** later (cookies/`Authorization` from the browser with `credentials: "include"` can't use `*`), and (b) it lets you restrict to known front ends. `Vary: Origin` is essential because the response now *differs by origin*; without it, a shared cache could serve one origin's answer to another.

---

## 9. The complete code

Only `main.go` changes from Chapter 41 (plus new tests). Here are the parts that are new or changed; the rest (types, storage, `createProduct`, `getProducts`, helpers) is unchanged.

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

var (
	mu          sync.RWMutex
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

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

func writeError(w http.ResponseWriter, status int, message string) {
	writeJSON(w, status, map[string]string{"error": message})
}

// ---- CORS policy --------------------------------------------------------------

// allowedOrigins is the allow-list of front-end origins that may call this API from a browser.
var allowedOrigins = map[string]bool{
	"http://localhost:5173": true, // Vite/React dev server
	"http://localhost:3000": true,
}

const (
	allowedMethods = "GET, POST, PUT, PATCH, DELETE, OPTIONS"
	allowedHeaders = "Content-Type, Authorization"
)

// applyCORS adds the CORS headers for allowed origins.
// It reports whether the request was a preflight that has been fully answered.
func applyCORS(w http.ResponseWriter, r *http.Request) (handled bool) {
	origin := r.Header.Get("Origin")
	if origin == "" || !allowedOrigins[origin] {
		return false // not a cross-origin browser call, or an origin we don't trust: no CORS headers
	}

	h := w.Header()
	h.Set("Access-Control-Allow-Origin", origin) // echo the specific origin, not "*"
	h.Add("Vary", "Origin")

	isPreflight := r.Method == http.MethodOptions && r.Header.Get("Access-Control-Request-Method") != ""
	if isPreflight {
		h.Set("Access-Control-Allow-Methods", allowedMethods)
		h.Set("Access-Control-Allow-Headers", allowedHeaders)
		h.Set("Access-Control-Max-Age", "600") // browsers may cache this answer for 10 minutes
		w.WriteHeader(http.StatusNoContent)    // 204: success, no body
		return true
	}
	return false
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

// ---- handlers ---------------------------------------------------------------

func productsHandler(w http.ResponseWriter, r *http.Request) {
	if applyCORS(w, r) { // answers preflight requests completely
		return
	}

	switch r.Method {
	case http.MethodGet:
		getProducts(w, r)
	case http.MethodPost:
		createProduct(w, r)
	default:
		w.Header().Set("Allow", "GET, POST, OPTIONS")
		writeError(w, http.StatusMethodNotAllowed, "method not allowed")
	}
}

func getProducts(w http.ResponseWriter, r *http.Request) {
	mu.RLock()
	products := make([]Product, len(productList))
	copy(products, productList)
	mu.RUnlock()

	writeJSON(w, http.StatusOK, products)
}

func createProduct(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)

	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()

	var req createProductRequest
	if err := dec.Decode(&req); err != nil {
		status, msg := describeJSONError(err)
		writeError(w, status, msg)
		return
	}
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

What changed compared with Chapter 41:

| Change | Why |
|--------|-----|
| `enableCORS` (`*`) → **`applyCORS`** with an **allow-list** | Only trusted front ends get CORS headers |
| Handles **`OPTIONS` preflight** → `204` + `Allow-Methods/Headers/Max-Age` | Lets browsers send JSON `POST`s (and later `DELETE`, `Authorization`) |
| `Vary: Origin` | Correct caching when the response depends on the origin |
| CORS handled **before** method dispatch, for every response | Error responses also carry CORS headers, so the browser can show the real error instead of a misleading CORS one |

---

## 10. Custom headers and `Authorization`

Any header outside the browser's safelist must be **explicitly allowed** in `Access-Control-Allow-Headers`. In Chapter 48 the front end will send:

```
Authorization: Bearer eyJhbGciOi...
```

That header triggers a preflight with `Access-Control-Request-Headers: authorization`. If our response doesn't list `Authorization`, the browser refuses. That's why `allowedHeaders` already includes `Authorization` (and `Content-Type`).

Header names in `Access-Control-Request-Headers` are lowercase and comma-separated; matching is case-insensitive, so `Content-Type` in our list matches `content-type`.

If your front end adds a custom header, such as `X-Request-ID`, add it too:

```go
const allowedHeaders = "Content-Type, Authorization, X-Request-ID"
```

To let JavaScript **read** a custom *response* header (like `X-Total-Count` or `Location`), list it in `Access-Control-Expose-Headers`; browsers hide non-safelisted response headers from scripts by default.

Forgetting a header produces this classic error:

```
Request header field x-request-id is not allowed by Access-Control-Allow-Headers in preflight response.
```

---

## 11. Testing the policy with `curl`

Run the server (`go run .`) and pretend to be a browser at `http://localhost:5173`.

**1. A preflight from an allowed origin** (real output):

```bash
curl -i -X OPTIONS http://localhost:8080/products \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type,authorization"
```

```
HTTP/1.1 204 No Content
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Max-Age: 600
Vary: Origin
Date: Sat, 26 Sep 2026 06:30:10 GMT
```

**2. A preflight from an origin that isn't allowed:**

```bash
curl -i -X OPTIONS http://localhost:8080/products \
  -H "Origin: https://evil.example" \
  -H "Access-Control-Request-Method: POST"
```

```
HTTP/1.1 405 Method Not Allowed
Allow: GET, POST, OPTIONS
Content-Type: application/json
...
{"error":"method not allowed"}
```

No `Access-Control-*` headers, so a browser at that origin would block it.

**3. The real request from the allowed origin:**

```bash
curl -i -X POST http://localhost:8080/products \
  -H "Origin: http://localhost:5173" \
  -H "Content-Type: application/json" \
  -d '{"title":"Kiwi","price":30}'
```

```
HTTP/1.1 201 Created
Access-Control-Allow-Origin: http://localhost:5173
Content-Type: application/json
Location: /products/4
Vary: Origin
...
```

Note the real response **also** has `Access-Control-Allow-Origin`: required, or the browser won't expose it to JavaScript even though the preflight passed.

**4. No `Origin` at all** (plain `curl`, mobile app, another server): no CORS headers, and everything works normally. CORS only matters for browsers.

---

## 12. Testing with a real browser

`curl` simulates a browser, but let's be sure. Create a folder with a single `index.html` and serve it on a different port than the API:

```html
<!doctype html>
<meta charset="utf-8">
<title>CORS demo</title>
<pre id="out">running...</pre>
<script>
  const out = document.getElementById("out");
  async function main() {
    try {
      const res = await fetch("http://localhost:8080/products", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ title: "Kiwi", price: 30 }),
      });
      out.textContent = "status " + res.status + "\n" + JSON.stringify(await res.json(), null, 2);
    } catch (e) {
      out.textContent = "FAILED: " + e.message;
    }
  }
  main();
</script>
```

```bash
python3 -m http.server 5173     # serves index.html at http://localhost:5173
go run .                        # the API at :8080 (in another terminal)
```

Open `http://localhost:5173`. Then open **DevTools → Network** (and Console):

- With **Chapter 41's** server: two things in the Network tab: a red `OPTIONS` request (status `405`) and *no* `POST`. The Console shows the "Response to preflight request doesn't pass access control check" error.
- With **this chapter's** server: an `OPTIONS` request (status `204`), followed by the `POST` (status `201`) and the page shows the created product.

Watching the browser send the `OPTIONS` you never wrote makes the concept concrete. (If the port `5173` is taken, choose another and add that origin to `allowedOrigins`.)

> ✅ **Verified with a real browser** while writing this chapter. Against the Chapter 41 server the Network panel showed `OPTIONS /products → 405 Method Not Allowed` followed by `POST /products [FAILED: net::ERR_FAILED]`, and the console printed exactly the "Response to preflight request doesn't pass access control check: It does not have HTTP ok status" message above. Against this chapter's server it showed `OPTIONS → 204 No Content` then `POST → 201 Created`, and the page displayed the new product.
>
> One more thing we observed: after reloading the page, the browser sent **only the `POST`** (no second `OPTIONS`), because it had **cached the preflight result** for `Access-Control-Max-Age: 600` seconds (section 14). If you're debugging CORS and your changes seem to have no effect, hard-reload, disable caching in DevTools, or use a fresh browser profile.

---

## 13. Automated tests

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

func doRequest(method, target, body string, headers map[string]string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	if body != "" {
		req.Header.Set("Content-Type", "application/json")
	}
	for k, v := range headers {
		req.Header.Set(k, v)
	}
	rec := httptest.NewRecorder()
	productsHandler(rec, req)
	return rec
}

func TestPreflightFromAllowedOrigin(t *testing.T) {
	rec := doRequest(http.MethodOptions, "/products", "", map[string]string{
		"Origin":                         "http://localhost:5173",
		"Access-Control-Request-Method":  "POST",
		"Access-Control-Request-Headers": "content-type,authorization",
	})

	if rec.Code != http.StatusNoContent {
		t.Fatalf("expected 204, got %d", rec.Code)
	}
	h := rec.Header()
	if got := h.Get("Access-Control-Allow-Origin"); got != "http://localhost:5173" {
		t.Errorf("Allow-Origin = %q", got)
	}
	if !strings.Contains(h.Get("Access-Control-Allow-Methods"), "POST") {
		t.Errorf("Allow-Methods = %q", h.Get("Access-Control-Allow-Methods"))
	}
	if !strings.Contains(h.Get("Access-Control-Allow-Headers"), "Authorization") {
		t.Errorf("Allow-Headers = %q", h.Get("Access-Control-Allow-Headers"))
	}
	if h.Get("Vary") != "Origin" {
		t.Errorf("Vary = %q", h.Get("Vary"))
	}
	if rec.Body.Len() != 0 {
		t.Errorf("preflight must have no body, got %q", rec.Body.String())
	}
}

func TestPreflightFromUnknownOrigin(t *testing.T) {
	rec := doRequest(http.MethodOptions, "/products", "", map[string]string{
		"Origin":                        "https://evil.example",
		"Access-Control-Request-Method": "POST",
	})
	if got := rec.Header().Get("Access-Control-Allow-Origin"); got != "" {
		t.Errorf("must not allow unknown origins, got %q", got)
	}
	if rec.Code == http.StatusNoContent {
		t.Errorf("unknown origin must not get a successful preflight")
	}
}

func TestRealRequestCarriesCORSHeader(t *testing.T) {
	resetProducts()
	rec := doRequest(http.MethodPost, "/products", `{"title":"Kiwi","price":30}`,
		map[string]string{"Origin": "http://localhost:5173"})

	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d", rec.Code)
	}
	if got := rec.Header().Get("Access-Control-Allow-Origin"); got != "http://localhost:5173" {
		t.Errorf("Allow-Origin = %q", got)
	}
}

func TestNoOriginNoCORSHeaders(t *testing.T) {
	rec := doRequest(http.MethodGet, "/products", "", nil)
	if rec.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rec.Code)
	}
	if rec.Header().Get("Access-Control-Allow-Origin") != "" {
		t.Error("plain (non-browser) requests should get no CORS headers")
	}
}

func TestCreateAndList(t *testing.T) {
	resetProducts()
	rec := doRequest(http.MethodPost, "/products", `{"title":"Mango","price":200}`, nil)
	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d (%s)", rec.Code, rec.Body.String())
	}
	var list []Product
	json.NewDecoder(doRequest(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	if len(list) != 4 {
		t.Errorf("expected 4 products, got %d", len(list))
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
			resetProducts()
			if rec := doRequest(http.MethodPost, "/products", tc.body, nil); rec.Code != tc.want {
				t.Errorf("expected %d, got %d (%s)", tc.want, rec.Code, rec.Body.String())
			}
		})
	}
}

func TestConcurrentCreates(t *testing.T) {
	resetProducts()
	const n = 50
	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			doRequest(http.MethodPost, "/products", `{"title":"Concurrent","price":1}`, nil)
		}()
	}
	wg.Wait()

	var list []Product
	json.NewDecoder(doRequest(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	if len(list) != 3+n {
		t.Fatalf("expected %d products, got %d", 3+n, len(list))
	}
}
```

```bash
go test -race ./...
```

```
ok  	ecommerce	1.04s
```

---

## 14. Caching preflights

Preflights add a **round trip** before every non-simple request: double latency for a fresh call. `Access-Control-Max-Age: 600` tells the browser it may **reuse the permission for 10 minutes** for that URL + method + header combination, so repeat calls skip the preflight. Browsers cap this (Chrome: 2 hours, Firefox: 24 hours; historically Chrome capped at 10 minutes for some versions), so treat it as an optimization, not a guarantee.

---

## 15. Security notes

- **CORS is not authentication.** It only tells *browsers* which pages may read responses. Anyone can call your API directly with `curl`. Use tokens and permissions (Chapters 48–49).
- **A preflight passing doesn't mean the action is authorized.** Authorization is checked on the *real* request.
- **Avoid `Access-Control-Allow-Origin: *` for anything with credentials or sensitive data.** `*` cannot be combined with `Access-Control-Allow-Credentials: true`, and blindly *reflecting any* `Origin` header is equally dangerous (it's effectively `*` with credentials). Always **check against an allow-list**, as we do.
- **Keep the allow-list in configuration** (Chapter 47), not hard-coded, so production and development can differ.
- **`Vary: Origin`** prevents cache poisoning between origins.
- **Don't run business logic for `OPTIONS`.** The preflight must be *side-effect free*.
- **Errors should carry CORS headers too**, or the browser will hide the real failure behind a misleading CORS error.

---

## 16. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Not handling `OPTIONS` | `405` on preflight; "does not have HTTP ok status" | Answer preflights with `204` |
| 2 | Missing `Access-Control-Allow-Headers` entry (e.g., `Authorization`) | "Request header field ... is not allowed" | List every custom header |
| 3 | Missing `Access-Control-Allow-Methods` for `PUT`/`DELETE` | "Method PUT is not allowed by Access-Control-Allow-Methods" | Include all methods |
| 4 | CORS headers only on the preflight, not the real response | Preflight passes, request blocked | Add `Allow-Origin` to every response |
| 5 | `*` with credentials | "must not be the wildcard '*' when credentials mode is 'include'" | Echo a specific allowed origin |
| 6 | Reflecting any `Origin` without checking | Wide-open API | Use an allow-list |
| 7 | Forgetting `Vary: Origin` when echoing | Cache serves the wrong origin's headers | Add it |
| 8 | Running handler logic for `OPTIONS` | Duplicate side effects/creation | Return early for preflights |
| 9 | Preflight returns `3xx` redirect | Preflights can't follow redirects | Answer directly with `2xx` |
| 10 | Testing only with `curl`/Postman | "Works on my machine", fails in the browser | Test with a real browser page |
| 11 | Wrong origin string (trailing slash, `http` vs `https`, wrong port) | Origin not on the list | Origins have **no** path/trailing slash: `http://localhost:5173` |
| 12 | Thinking CORS fixes a server that's down | Network error, not CORS | Check that the server responds at all |

**Debugging checklist:** open DevTools → Network → find the red `OPTIONS` request → read its response headers and status → compare with `Access-Control-Request-*` request headers.

---

## 17. Exercises

### Exercise 1: Explain in your own words
Why doesn't the browser preflight a plain `GET` without custom headers?

<details><summary>Solution</summary>

A simple `GET` is something browsers were already able to send cross-origin before CORS (`<img>`, `<script>`, links), so old servers already had to cope with it. Only newer capabilities (other methods, JSON bodies, custom headers) risk harming servers that never anticipated them, so only those need a permission check.
</details>

### Exercise 2: Classify
Preflight or not? (a) `GET` with `Authorization: Bearer x`, (b) `POST` form-encoded, (c) `DELETE /products/4`, (d) `POST` JSON, (e) `GET` with no headers.

<details><summary>Solution</summary>

(a) preflight (custom header), (b) no, (c) preflight, (d) preflight, (e) no.
</details>

### Exercise 3: Read a preflight
A browser sends `Access-Control-Request-Method: PUT` and `Access-Control-Request-Headers: x-api-key, content-type`. What must the server include for the real request to be sent?

<details><summary>Solution</summary>

A 2xx status, `Access-Control-Allow-Origin` (matching the origin), `Access-Control-Allow-Methods` containing `PUT`, and `Access-Control-Allow-Headers` containing `X-API-Key` and `Content-Type`.
</details>

### Exercise 4: Configurable origins
Read the allowed origins from an environment variable (comma-separated), falling back to `http://localhost:5173`.

<details><summary>Solution</summary>

```go
func loadAllowedOrigins() map[string]bool {
	raw := os.Getenv("ALLOWED_ORIGINS")
	if raw == "" {
		raw = "http://localhost:5173"
	}
	m := map[string]bool{}
	for _, o := range strings.Split(raw, ",") {
		if o = strings.TrimSpace(o); o != "" {
			m[o] = true
		}
	}
	return m
}

var allowedOrigins = loadAllowedOrigins()
```
(Chapter 47 does this properly with a config package.)
</details>

### Exercise 5: Expose a header
Make the `Location` header readable by browser JavaScript on cross-origin `POST` responses.

<details><summary>Solution</summary>

Add `h.Set("Access-Control-Expose-Headers", "Location")` next to the `Allow-Origin` header in `applyCORS`.
</details>

### Exercise 6: Observe caching
Using DevTools, call the JSON `POST` twice from the demo page (add a button). With `Max-Age: 600`, how many `OPTIONS` requests do you see? Set `Max-Age: 0` and repeat.

<details><summary>Solution</summary>

With `Max-Age: 600`, the browser sends **one** preflight and reuses the permission for subsequent identical requests (within the cache lifetime). With `0`, it preflights every time. (DevTools may need "Disable cache" unchecked to see this.)
</details>

### Exercise 7 (challenge): Middleware version
Turn `applyCORS` into middleware, `func cors(next http.Handler) http.Handler`, applied to the whole mux instead of being called by one handler. What advantages do you get? (Chapters 43 and 45 do exactly this.)

<details><summary>Solution</summary>

```go
func cors(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if applyCORS(w, r) {
			return
		}
		next.ServeHTTP(w, r)
	})
}

// main: http.ListenAndServe(":8080", cors(mux))
```
Advantages: applies to **every** route automatically (including 404s from the mux), keeps handlers free of CORS code, and preflight requests are answered even for routes that don't list `OPTIONS`.
</details>

---

## 18. Quiz

1. Who sends a preflight request, and what method does it use?
2. Name the three request headers a preflight carries and what they mean.
3. What status code should a successful preflight response have?
4. Why must the real response also include `Access-Control-Allow-Origin`?
5. Why is an allow-list better than `*`?
6. Is CORS a substitute for authentication?

<details><summary>Answers</summary>

1. The browser, automatically, using `OPTIONS`.
2. `Origin` (the page's origin), `Access-Control-Request-Method` (real method), `Access-Control-Request-Headers` (non-safelisted headers).
3. Any 2xx; conventionally `204 No Content`.
4. The browser checks CORS again on the real response before exposing it to JavaScript.
5. It grants access only to trusted front ends and allows credentials safely.
6. No; it only governs which pages a *browser* lets read responses.
</details>

---

## 19. Summary

- **`OPTIONS`** asks what's allowed. Browsers use it automatically as a **preflight** before non-simple cross-origin requests: JSON `POST`, `PUT`/`PATCH`/`DELETE`, and anything with `Authorization` or custom headers.
- Preflight protects **legacy servers** from powerful new request types; only servers that answer correctly get the real request.
- The server answers with **`204`** plus **`Access-Control-Allow-Origin`, `-Allow-Methods`, `-Allow-Headers`, `-Max-Age`** and **`Vary: Origin`**, without running business logic.
- Use an **origin allow-list** (echo the matching origin) instead of `*`; put CORS headers on **all** responses, including errors.
- `Authorization` and any custom headers must be **listed in `Allow-Headers`**; use **`Expose-Headers`** so scripts can read response headers.
- CORS is enforced by **browsers only**: it is **not** authentication. Test with a real browser page, not just `curl`.

### ➡️ What's next?

[Chapter 43](43-refactoring-the-codebase.md) refactors the growing `main.go` into clean, single-purpose pieces (handlers, middleware, utilities), applying the **Single Responsibility Principle** from Chapter 7.
