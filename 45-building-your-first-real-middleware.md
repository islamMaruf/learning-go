# Chapter 45: Building Your First Real Middleware — A Request Logger

> **Goal of this chapter:** Build a production-style **logging middleware** that records every request: method, path, **status code**, response size, duration, and client address. Along the way you'll learn the `log` and `log/slog` packages, why plain middleware can't see the status code (and the **`ResponseWriter` wrapper** trick that fixes it), how to pick log levels from status codes, how to make a middleware configurable and testable, and how to keep all of it working with `http.ResponseController`.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 3 hours  **Prerequisite:** [Chapters 43–44](44-advanced-routing-go-1-22-and-the-middleware-idea.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why log requests?](#2-why-log-requests)
3. [Logging in Go: `log` and `log/slog`](#3-logging-in-go)
4. [Version 1: the simplest logger middleware](#4-version-1-the-simplest-logger)
5. [The problem: middleware can't see the status code](#5-the-problem-you-cant-see-the-status-code)
6. [The `ResponseWriter` wrapper](#6-the-responsewriter-wrapper)
7. [Version 2: a complete logger](#7-version-2-a-complete-logger)
8. [Wiring it into the app](#8-wiring-it-into-the-app)
9. [See it in action](#9-see-it-in-action)
10. [Testing the middleware](#10-testing-the-middleware)
11. [Design notes: what to log, what never to log](#11-design-notes)
12. [Wrapper pitfalls: `Flush`, `Hijack`, and `ResponseController`](#12-wrapper-pitfalls)
13. [Common mistakes](#13-common-mistakes)
14. [Exercises](#14-exercises)
15. [Quiz](#15-quiz)
16. [Summary](#16-summary)

---

## 1. What you will learn

- Why every backend needs **access logs**
- The `log` package vs. **structured logging** with `log/slog` (Go 1.21+)
- How to write a middleware that measures **duration**
- Why a plain middleware **cannot see the response status**, and how to fix it by **wrapping `http.ResponseWriter`**
- How to choose **log levels** (INFO/WARN/ERROR) from status codes
- How to inject a logger so the middleware is **testable**
- What *not* to log (secrets, personal data)

---

## 2. Why log requests?

Once your API is live you can no longer watch it run on your screen. Logs are how you answer:

| Question | Log line that answers it |
|----------|--------------------------|
| "Is the server even receiving requests?" | any line |
| "Why did this user get an error?" | `status=500 path=/products/7` |
| "Which endpoint is slow?" | `duration=2.3s path=/products` |
| "Who is hammering us?" | `remote=203.0.113.9` repeated thousands of times |
| "What happened just before the crash?" | the last lines before it |
| "Did my deploy break something?" | sudden rise in `status=500` |

An **access log** with one line per request is the single most useful piece of operational data. And because it applies to *every* request, it's the perfect job for **middleware** (Chapter 44): written once, applied everywhere.

---

## 3. Logging in Go

### 3.1 The classic `log` package

```go
package main

import "log"

func main() {
	log.Println("server starting")
	log.Printf("listening on %s", ":8080")
}
```

Output (with a timestamp prefix by default):

```
2026/09/26 07:01:12 server starting
2026/09/26 07:01:12 listening on :8080
```

Simple and fine for small programs. Weaknesses: unstructured free text (hard for machines to search/filter), no levels (info/warn/error), and `log.Fatal` exits the program.

### 3.2 Structured logging with `log/slog`

Since **Go 1.21**, the standard library has `log/slog`: **structured, leveled** logging. Each record has a *message* plus *key/value attributes*:

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

	logger.Info("request", "method", "GET", "path", "/products", "status", 200)
	logger.Warn("slow request", "path", "/products", "ms", 1200)
	logger.Error("database down", "err", "connection refused")
}
```

Output (text format):

```
time=2026-09-26T07:01:12.345+06:00 level=INFO msg=request method=GET path=/products status=200
time=2026-09-26T07:01:12.345+06:00 level=WARN msg="slow request" path=/products ms=1200
time=2026-09-26T07:01:12.345+06:00 level=ERROR msg="database down" err="connection refused"
```

With `slog.NewJSONHandler` you get one JSON object per line:

```json
{"time":"2026-09-26T07:01:12.345+06:00","level":"INFO","msg":"request","method":"GET","path":"/products","status":200}
```

Why structured logs win: log aggregation tools (Loki, Elasticsearch, Datadog, CloudWatch) can filter `status>=500` or group by `path` because the fields are *data*, not prose.

Key ideas:

| Concept | Meaning |
|---------|---------|
| `slog.Logger` | The thing you log with (`Info`, `Warn`, `Error`, `Debug`) |
| **Handler** | Decides the *format* (text/JSON) and *destination* (stdout/file) and minimum level |
| **Attributes** | Alternating `key, value` pairs after the message (or `slog.String("k","v")` for type safety) |
| `logger.With(...)` | Returns a logger that adds fixed attributes to every record |
| `slog.Default()` | The global logger (also what `log.Println` now feeds into) |

**Convention:** log to **stdout/stderr** and let the platform (Docker, Kubernetes, systemd) collect it. Don't manage log files inside your app.

---

## 4. Version 1: the simplest logger

A first attempt, printing the method, path and how long the request took:

```go
func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()               // before: note the time

		next.ServeHTTP(w, r)              // run the rest of the chain

		log.Printf("%s %s took %v", r.Method, r.URL.Path, time.Since(start)) // after: report
	})
}
```

This already demonstrates the **before/after** middleware structure (Chapter 44):

```
start := now          ← BEFORE (request phase)
next.ServeHTTP(w, r)  ← the rest of the chain runs here (could take milliseconds or seconds)
log(duration)         ← AFTER  (response phase)
```

Output for a `GET /products`:

```
2026/09/26 07:05:01 GET /products took 51.3µs
```

Good start, but incomplete: the most important field for debugging, **the status code**, is missing.

---

## 5. The problem: you can't see the status code

Why not simply read the status after `next.ServeHTTP`? Because **`http.ResponseWriter` has no getter**:

```go
type ResponseWriter interface {
	Header() http.Header
	Write([]byte) (int, error)
	WriteHeader(statusCode int)
}
```

The status is *written into the wire*, not stored where you can read it back. Our middleware hands `w` to the next handler, which calls `w.WriteHeader(404)`, and the middleware never learns what number was used.

```
Logger ──w──► CORS ──w──► mux ──w──► handler:  w.WriteHeader(404)
   ▲                                                    │
   └────────  "what status was it?"  ✗  ───────────────┘
```

**The solution: hand the next handler a *different* `ResponseWriter`, one that secretly records what happens, and then forwards everything to the real one.** This is the **decorator** (wrapper) pattern, and it works because `http.ResponseWriter` is an *interface* (Chapter 51 preview): anything with those three methods can stand in for the real writer.

---

## 6. The `ResponseWriter` wrapper

```go
// statusRecorder wraps a ResponseWriter to remember the status code and body size.
type statusRecorder struct {
	http.ResponseWriter     // embedded: all other methods forward automatically
	status int              // 0 until WriteHeader/Write is called
	bytes  int
}

func (s *statusRecorder) WriteHeader(code int) {
	if s.status == 0 {
		s.status = code
	}
	s.ResponseWriter.WriteHeader(code) // forward to the real writer
}

func (s *statusRecorder) Write(b []byte) (int, error) {
	if s.status == 0 {
		s.status = http.StatusOK // a Write without WriteHeader implies 200
	}
	n, err := s.ResponseWriter.Write(b)
	s.bytes += n
	return n, err
}

// Unwrap lets http.ResponseController reach the real writer (Flush, Hijack, deadlines...).
func (s *statusRecorder) Unwrap() http.ResponseWriter { return s.ResponseWriter }
```

How it works:

1. **Embedding** `http.ResponseWriter` (Chapter 21/22) means `statusRecorder` *automatically has* `Header()` (forwarded to the embedded real writer), plus we **override** `WriteHeader` and `Write`.
2. In our overrides we **record**, then **forward** to the real writer.
3. If the handler never calls `WriteHeader`, Go sends `200` on the first `Write`, so we mimic that (`status = 200`).
4. If the handler writes **nothing at all** (e.g., an empty `200`), `status` stays `0`, and the middleware treats `0` as `200` when reporting.
5. `Unwrap` is the modern hook (section 12) so features like `Flush` still work through the wrapper.

Now the logger passes **the recorder** down the chain instead of `w`:

```go
rec := &statusRecorder{ResponseWriter: w}
next.ServeHTTP(rec, r)          // downstream code writes into rec
status := rec.status            // ...and we can read it back!
```

---

## 7. Version 2: a complete logger

```go
// file: middleware/logger.go
package middleware

import (
	"log/slog"
	"net/http"
	"time"
)

// statusRecorder wraps a ResponseWriter to remember the status code and body size.
type statusRecorder struct {
	http.ResponseWriter
	status int
	bytes  int
}

func (s *statusRecorder) WriteHeader(code int) {
	if s.status == 0 {
		s.status = code
	}
	s.ResponseWriter.WriteHeader(code)
}

func (s *statusRecorder) Write(b []byte) (int, error) {
	if s.status == 0 {
		s.status = http.StatusOK
	}
	n, err := s.ResponseWriter.Write(b)
	s.bytes += n
	return n, err
}

// Unwrap lets http.ResponseController reach the real writer (Flush, deadlines, ...).
func (s *statusRecorder) Unwrap() http.ResponseWriter { return s.ResponseWriter }

// Logger returns a middleware that writes one structured log record per request.
// The logger is injected (not global) so tests and different environments can supply their own.
func Logger(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			rec := &statusRecorder{ResponseWriter: w}

			next.ServeHTTP(rec, r)

			status := rec.status
			if status == 0 {
				status = http.StatusOK // handler wrote nothing: net/http sends 200
			}

			// pick a level from the outcome
			level := slog.LevelInfo
			switch {
			case status >= 500:
				level = slog.LevelError
			case status >= 400:
				level = slog.LevelWarn
			}

			logger.LogAttrs(r.Context(), level, "request",
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.Int("status", status),
				slog.Int("bytes", rec.bytes),
				slog.Duration("duration", time.Since(start)),
				slog.String("remote", r.RemoteAddr),
			)
		})
	}
}
```

Notice the **shape** of `Logger`: it's a *function that returns a middleware* (a "middleware factory"):

```go
func Logger(logger *slog.Logger) func(http.Handler) http.Handler
//          └─ configuration ─┘   └───────── the actual middleware ─────────┘
```

That's the standard way to give a middleware settings: the outer function captures the configuration in a closure (Chapter 20); the returned function is the `func(http.Handler) http.Handler` you already know. `CORS` didn't need config yet, so it stayed a plain middleware. (Chapter 47 will make CORS configurable in the same way.)

Other design choices:

- **Log after** the handler returns, so we know status and duration. (A request that crashes the whole process never gets logged, which is why "Recover" middleware, Chapter 46, matters.)
- `logger.LogAttrs` with typed `slog.String/Int/Duration` avoids allocations and is the recommended fast path.
- The **level** follows the status: `5xx → ERROR`, `4xx → WARN`, else `INFO`. In production you can then alert on errors and filter out routine traffic.
- `r.RemoteAddr` is the immediate peer. Behind a reverse proxy it's the *proxy's* address; you'd read `X-Forwarded-For` (only from a trusted proxy!).

---

## 8. Wiring it into the app

`newRouter` now receives the logger, so `main` decides *where logs go*, while tests can silence them:

```go
// file: main.go
package main

import (
	"ecommerce/handlers"
	"ecommerce/middleware"
	"fmt"
	"log/slog"
	"net/http"
	"os"
)

// newRouter builds the complete HTTP handler for the app.
func newRouter(logger *slog.Logger) http.Handler {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /products", handlers.GetProducts)
	mux.HandleFunc("POST /products", handlers.CreateProduct)
	mux.HandleFunc("GET /products/{id}", handlers.GetProduct)

	// Outermost first: Logger sees every request, including ones CORS answers itself.
	return middleware.Logger(logger)(middleware.CORS(mux))
}

func main() {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

	fmt.Println("Server running on http://localhost:8080")
	if err := http.ListenAndServe(":8080", newRouter(logger)); err != nil {
		logger.Error("server stopped", "err", err)
		os.Exit(1)
	}
}
```

Placement: `Logger` is **outside** `CORS`, so preflight requests (answered by CORS without ever reaching the mux) are logged too. If `Logger` were inside CORS, preflights would be invisible.

Because `newRouter` now takes an argument, the existing tests need one small change: pass a logger that discards output. In the test helper `do`, replace the `newRouter()` call with:

```go
newRouter(slog.New(slog.NewTextHandler(io.Discard, nil))).ServeHTTP(rec, req)
```

(and add `"io"` and `"log/slog"` to the test file's imports). We'll show the complete updated test file in section 10.

---

## 9. See it in action

Run the server and send a few requests:

```bash
curl -s localhost:8080/products > /dev/null
curl -s localhost:8080/products/2 > /dev/null
curl -s localhost:8080/products/99 > /dev/null
curl -s -X POST localhost:8080/products -d '{"title":"","price":5}' > /dev/null
curl -s -X DELETE localhost:8080/products > /dev/null
curl -s localhost:8080/nothing > /dev/null
curl -s -X OPTIONS localhost:8080/products -H 'Origin: http://localhost:5173' -H 'Access-Control-Request-Method: POST' > /dev/null
```

Real log output from the built project:

```
time=2026-09-26T11:57:56.333+06:00 level=INFO msg=request method=GET path=/products status=200 bytes=380 duration=55.186µs remote=[::1]:59440
time=2026-09-26T11:57:56.337+06:00 level=INFO msg=request method=GET path=/products/2 status=200 bytes=120 duration=28.203µs remote=[::1]:59448
time=2026-09-26T11:57:56.343+06:00 level=WARN msg=request method=GET path=/products/99 status=404 bytes=30 duration=32.515µs remote=[::1]:59460
time=2026-09-26T11:57:56.348+06:00 level=WARN msg=request method=POST path=/products status=422 bytes=30 duration=54.031µs remote=[::1]:59472
time=2026-09-26T11:57:56.353+06:00 level=WARN msg=request method=DELETE path=/products status=405 bytes=19 duration=31.854µs remote=[::1]:59486
time=2026-09-26T11:57:56.358+06:00 level=WARN msg=request method=GET path=/nothing status=404 bytes=19 duration=19.624µs remote=[::1]:59498
time=2026-09-26T11:57:56.362+06:00 level=INFO msg=request method=OPTIONS path=/products status=204 bytes=0 duration=5.167µs remote=[::1]:59508
```

Everything you'd want at a glance: successes (`INFO`), client errors (`WARN`, with the exact status), and the preflight (`204`, 0 bytes). (`[::1]` is the IPv6 loopback address; `curl` chose IPv6 for `localhost` here.)

**JSON logs for production:** swap the handler in `main`:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
```

```json
{"time":"2026-09-26T05:56:26.7+06:00","level":"INFO","msg":"request","method":"GET","path":"/products","status":200,"bytes":380,"duration":95600,"remote":"127.0.0.1:56392"}
```

(`duration` in JSON is in nanoseconds by default.)

Even better, choose the format from configuration (Chapter 47): human-friendly text in development, JSON in production.

---

## 10. Testing the middleware

Because the logger is **injected**, we can capture its output in a buffer and assert on it: no global state, no printing to the console.

```go
// file: middleware/logger_test.go
package middleware

import (
	"bytes"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

// newTestLogger returns a logger writing to buf, without timestamps so output is stable.
func newTestLogger(buf *bytes.Buffer) *slog.Logger {
	return slog.New(slog.NewTextHandler(buf, &slog.HandlerOptions{
		ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
			if a.Key == slog.TimeKey {
				return slog.Attr{} // drop the time
			}
			return a
		},
	}))
}

func runLogger(t *testing.T, handler http.HandlerFunc, method, path string) string {
	t.Helper()
	var buf bytes.Buffer
	h := Logger(newTestLogger(&buf))(handler)

	req := httptest.NewRequest(method, path, nil)
	req.RemoteAddr = "10.0.0.7:1234"
	h.ServeHTTP(httptest.NewRecorder(), req)
	return buf.String()
}

func TestLoggerRecordsStatusAndSize(t *testing.T) {
	out := runLogger(t, func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusCreated)
		w.Write([]byte("hello"))
	}, http.MethodPost, "/things")

	for _, want := range []string{"level=INFO", "method=POST", "path=/things", "status=201", "bytes=5", "remote=10.0.0.7:1234"} {
		if !strings.Contains(out, want) {
			t.Errorf("log %q does not contain %q", out, want)
		}
	}
}

func TestLoggerDefaultsToStatus200WhenOnlyWriting(t *testing.T) {
	out := runLogger(t, func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("ok")) // no explicit WriteHeader
	}, http.MethodGet, "/")
	if !strings.Contains(out, "status=200") {
		t.Errorf("expected status=200, got %q", out)
	}
}

func TestLoggerDefaultsToStatus200WhenNothingWritten(t *testing.T) {
	out := runLogger(t, func(w http.ResponseWriter, r *http.Request) {}, http.MethodGet, "/")
	if !strings.Contains(out, "status=200") || !strings.Contains(out, "bytes=0") {
		t.Errorf("unexpected log %q", out)
	}
}

func TestLoggerLevelsFollowStatus(t *testing.T) {
	tests := []struct {
		status int
		level  string
	}{
		{http.StatusOK, "level=INFO"},
		{http.StatusNotFound, "level=WARN"},
		{http.StatusInternalServerError, "level=ERROR"},
	}
	for _, tc := range tests {
		out := runLogger(t, func(w http.ResponseWriter, r *http.Request) {
			w.WriteHeader(tc.status)
		}, http.MethodGet, "/")
		if !strings.Contains(out, tc.level) {
			t.Errorf("status %d: expected %s in %q", tc.status, tc.level, out)
		}
	}
}

func TestLoggerPassesTheRequestThrough(t *testing.T) {
	var buf bytes.Buffer
	called := false
	h := Logger(newTestLogger(&buf))(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		called = true
		w.Header().Set("X-Test", "yes")
	}))

	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest(http.MethodGet, "/", nil))

	if !called {
		t.Error("the wrapped handler was not called")
	}
	if rec.Header().Get("X-Test") != "yes" {
		t.Error("headers set downstream must reach the real ResponseWriter")
	}
}
```

Also update the application-level test's helper (the only change to `main_test.go`), so it compiles with the new `newRouter(logger)` signature. Here's the complete file:

```go
// file: main_test.go
package main

import (
	"ecommerce/database"
	"encoding/json"
	"io"
	"log/slog"
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

var quietLogger = slog.New(slog.NewTextHandler(io.Discard, nil))

func do(method, target, body string, headers map[string]string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	if body != "" {
		req.Header.Set("Content-Type", "application/json")
	}
	for k, v := range headers {
		req.Header.Set(k, v)
	}
	rec := httptest.NewRecorder()
	newRouter(quietLogger).ServeHTTP(rec, req)
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
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			if rec := do(http.MethodGet, tc.path, "", nil); rec.Code != tc.want {
				t.Errorf("GET %s: expected %d, got %d", tc.path, tc.want, rec.Code)
			}
		})
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
}

func TestAutomaticMethodNotAllowed(t *testing.T) {
	rec := do(http.MethodDelete, "/products", "", nil)
	if rec.Code != http.StatusMethodNotAllowed {
		t.Fatalf("expected 405, got %d", rec.Code)
	}
	if got := rec.Header().Get("Allow"); got != "GET, HEAD, POST" {
		t.Errorf("Allow = %q", got)
	}
}

func TestPreflightFromAllowedOrigin(t *testing.T) {
	rec := do(http.MethodOptions, "/products/7", "", map[string]string{
		"Origin":                         "http://localhost:5173",
		"Access-Control-Request-Method":  "POST",
		"Access-Control-Request-Headers": "content-type,authorization",
	})
	if rec.Code != http.StatusNoContent {
		t.Fatalf("expected 204, got %d", rec.Code)
	}
	if rec.Header().Get("Access-Control-Allow-Origin") != "http://localhost:5173" {
		t.Errorf("missing Allow-Origin")
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

Run:

```bash
go test -race ./...
```

```
ok  	ecommerce	1.0s
ok  	ecommerce/middleware	1.0s
```

Note `TestConcurrentCreates` now also exercises the logger from many goroutines at once. `slog` loggers are safe for concurrent use.

---

## 11. Design notes

**What to log per request:** timestamp, method, path, status, duration, response size, client address, and a **request ID** (Chapter 46) to correlate lines belonging to one request.

**What NOT to log:**

- ❌ **Passwords, tokens, API keys, credit card numbers**: never, in any form.
- ❌ Full **request bodies** by default (they may contain personal data).
- ❌ The `Authorization` header or cookies.
- ⚠️ **Query strings**: they can contain secrets (`?token=...`). We log `r.URL.Path`, not `r.URL.String()` or `RequestURI`, for exactly that reason. If you need query parameters, allow-list the safe ones.
- ⚠️ Personal data (emails, names) may be regulated (GDPR). Log identifiers, not identities.

**Volume and cost:** an access log line per request is cheap, but at thousands of requests per second it adds up. Log at `INFO` for everything in development; consider sampling or higher-level filtering (or logging health checks at `DEBUG`) in high-traffic production.

**Log levels:** `DEBUG` (very detailed; usually off in production), `INFO` (normal events), `WARN` (something odd but handled: 4xx here), `ERROR` (something failed: 5xx). Set the minimum level in the handler options: `&slog.HandlerOptions{Level: slog.LevelInfo}`.

**Correlation:** as soon as you have concurrent requests, log lines interleave. A **request ID** on every line lets you reconstruct one request's story: we add it in the next chapter.

---

## 12. Wrapper pitfalls

Wrapping `http.ResponseWriter` is powerful but has a subtle catch: the *real* writer may implement **extra optional interfaces**, and our wrapper (which only exposes the three basic methods) **hides** them.

| Optional interface | Used for |
|--------------------|----------|
| `http.Flusher` (`Flush()`) | Streaming responses (server-sent events, chunked output) |
| `http.Hijacker` (`Hijack()`) | Taking over the raw connection (WebSockets) |
| `http.Pusher` | HTTP/2 push (rarely used now) |
| `io.ReaderFrom` | Efficient file copying (`sendfile`) |

If a handler does `w.(http.Flusher).Flush()` on our wrapper, the assertion **fails** because `statusRecorder` doesn't have a `Flush` method: streaming silently breaks whenever the Logger middleware is on.

**The modern fix (Go 1.20+): `http.ResponseController`.** It finds the capability by following `Unwrap() http.ResponseWriter` chains, which is exactly why we added `Unwrap` to `statusRecorder`:

```go
// in a streaming handler:
rc := http.NewResponseController(w)
fmt.Fprint(w, "chunk 1\n")
rc.Flush()                          // works through the wrapper thanks to Unwrap()
rc.SetWriteDeadline(time.Now().Add(30 * time.Second))
```

So: **always give your wrappers an `Unwrap` method**, and have handlers use `http.NewResponseController(w)` instead of type assertions.

---

## 13. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Reading the status without a wrapper | Can't; the status is write-only | Wrap `ResponseWriter` |
| 2 | Passing `w` (not the wrapper) to `next.ServeHTTP` | Recorder stays empty; logs always say 200 | Pass `rec` |
| 3 | Forgetting the implicit `200` on `Write` | Status `0` logged | Default to `200` |
| 4 | Not calling the real `WriteHeader`/`Write` in the wrapper | Response never reaches the client | Always forward |
| 5 | Wrapper hides `Flusher`/`Hijacker` | Streaming/WebSockets break | Add `Unwrap()` and use `ResponseController` |
| 6 | Logging before `next.ServeHTTP` | No status/duration | Log after |
| 7 | Logging secrets (`Authorization`, query tokens, bodies) | Security incident | Log allow-listed fields only |
| 8 | Global logger hard-wired in the middleware | Untestable, inflexible | Inject `*slog.Logger` |
| 9 | Using `log.Fatal` inside request code | Kills the whole server | Return errors / `logger.Error` |
| 10 | `Logger` inside `CORS` (wrong order) | Preflight requests unlogged | Put `Logger` outermost |
| 11 | Using `r.URL.String()` | Query-string secrets in logs | Log `r.URL.Path` |
| 12 | Panics skipping the log line | Crashed requests unlogged | Add `Recover` middleware (Chapter 46) |

---

## 14. Exercises

### Exercise 1: Add the query string safely
Add a `query` attribute with only the `page` and `limit` parameters (if present).

<details><summary>Solution</summary>

```go
q := r.URL.Query()
var parts []string
for _, key := range []string{"page", "limit"} {
	if v := q.Get(key); v != "" {
		parts = append(parts, key+"="+v)
	}
}
// ... in LogAttrs: slog.String("query", strings.Join(parts, "&"))
```
Allow-listing keys avoids logging tokens or other secrets.
</details>

### Exercise 2: Slow request warning
Log at `WARN` (even for 2xx) when a request takes longer than 500 ms, adding `slow=true`.

<details><summary>Solution</summary>

```go
elapsed := time.Since(start)
if level == slog.LevelInfo && elapsed > 500*time.Millisecond {
	level = slog.LevelWarn
}
// add slog.Bool("slow", elapsed > 500*time.Millisecond)
```
</details>

### Exercise 3: JSON logs
Switch the app to JSON logging when the environment variable `LOG_FORMAT=json` is set.

<details><summary>Solution</summary>

```go
var handler slog.Handler
if os.Getenv("LOG_FORMAT") == "json" {
	handler = slog.NewJSONHandler(os.Stdout, nil)
} else {
	handler = slog.NewTextHandler(os.Stdout, nil)
}
logger := slog.New(handler)
```
(Chapter 47 moves this into proper configuration.)
</details>

### Exercise 4: Test a wrapper edge case
Write a test proving that calling `WriteHeader` twice records only the **first** status (as `net/http` itself does).

<details><summary>Solution</summary>

```go
func TestRecorderKeepsFirstStatus(t *testing.T) {
	rec := &statusRecorder{ResponseWriter: httptest.NewRecorder()}
	rec.WriteHeader(http.StatusNotFound)
	rec.WriteHeader(http.StatusInternalServerError) // ignored by net/http; we must not record it
	if rec.status != http.StatusNotFound {
		t.Errorf("expected 404, got %d", rec.status)
	}
}
```
</details>

### Exercise 5: Health checks at DEBUG
Log requests to `/health` at `DEBUG` instead of `INFO` to keep load balancers' pings out of normal logs.

<details><summary>Solution</summary>

```go
if r.URL.Path == "/health" && level == slog.LevelInfo {
	level = slog.LevelDebug
}
```
With the default handler level (`INFO`), `DEBUG` records are dropped.
</details>

### Exercise 6: Support streaming
Write a `/stream` handler that sends three chunks one second apart using `http.NewResponseController(w).Flush()`. Verify with `curl -N localhost:8080/stream` that chunks arrive one at a time *with the Logger enabled*. What happens if you remove `Unwrap` from `statusRecorder`?

<details><summary>Solution</summary>

```go
func Stream(w http.ResponseWriter, r *http.Request) {
	rc := http.NewResponseController(w)
	for i := 1; i <= 3; i++ {
		fmt.Fprintf(w, "chunk %d\n", i)
		rc.Flush()
		time.Sleep(time.Second)
	}
}
```
With `Unwrap`, chunks arrive immediately, one per second. Without it, `Flush` returns `http.ErrNotSupported`, the chunks are buffered, and `curl` sees everything at the end (the wrapper hid the `Flusher`).
</details>

### Exercise 7 (challenge): Count bytes accurately
`http.ResponseWriter` might implement `io.ReaderFrom` (used by `io.Copy` for files). Our recorder doesn't implement it, so `io.Copy(w, file)` falls back to plain `Write` calls. Is our byte count still right? Why or why not?

<details><summary>Solution</summary>

Yes: since `statusRecorder` lacks `ReadFrom`, `io.Copy` uses `Write` in a loop, and each `Write` goes through our counter, so the count is correct (it just misses the `sendfile` optimization). If we *did* forward `ReadFrom` to the underlying writer, we'd have to count the bytes it returns ourselves.
</details>

---

## 15. Quiz

1. Why can't a middleware read the response status directly?
2. What's the trick to capture it?
3. What does `slog` add over `log`?
4. Why inject the `*slog.Logger` instead of using a global one?
5. Which log level would you use for a `404`? For a `500`?
6. Why does `statusRecorder` implement `Unwrap`?
7. Why log `r.URL.Path` rather than `r.URL.String()`?

<details><summary>Answers</summary>

1. `http.ResponseWriter` has setters (`WriteHeader`) but no getter.
2. Pass the downstream handlers a wrapper that records `WriteHeader`/`Write` calls and forwards them.
3. Structured, leveled, key/value logging with pluggable handlers (text/JSON).
4. Tests and different environments can supply their own logger (e.g., a buffer).
5. `WARN` for 4xx, `ERROR` for 5xx (in this design).
6. So `http.ResponseController` can reach the real writer for `Flush`, `Hijack`, deadlines.
7. Query strings may contain secrets.
</details>

---

## 16. Summary

- Every production service needs **access logs**: one structured line per request. Middleware is the right place.
- Use **`log/slog`** for **structured, leveled** logs; inject the logger; log to stdout; JSON in production.
- A middleware **cannot read the status** from `http.ResponseWriter`; wrap it in a **`statusRecorder`** that records `WriteHeader`/`Write` and forwards to the real writer. Default to **200** when nothing was written explicitly.
- Log **after** `next.ServeHTTP` (so status and duration are known); choose the **level from the status** (5xx = ERROR, 4xx = WARN).
- A **middleware factory** (`func Logger(cfg) func(http.Handler) http.Handler`) is how middleware gets configuration through a closure.
- Give wrappers an **`Unwrap`** method and use `http.NewResponseController` so streaming and other optional interfaces keep working.
- **Never log secrets**; prefer `r.URL.Path`; add a request ID for correlation (next chapter).

### ➡️ What's next?

[Chapter 46](46-advanced-middleware.md) goes deeper: the **request pipeline**. You'll build a tidy way to **chain** many middleware, add **panic recovery**, **request IDs** (with `context`), and a **JSON `404`**, and visualize exactly how a request flows in and out of the stack.
