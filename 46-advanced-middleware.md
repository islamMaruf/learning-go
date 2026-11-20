# Chapter 46: Advanced Middleware — Understanding the Request Pipeline

> **Goal of this chapter:** Turn our collection of one-off middleware into a real **request pipeline**. You'll build a tidy way to **compose** middleware (`Stack`), understand exactly how a request flows in and out of the layers, add three essential pieces of production middleware, **panic recovery**, **request IDs** (using Go's `context`), and **JSON errors for the mux's built-in `404`/`405`**, and learn how to decide the *order* of the pipeline (with the mistakes that go wrong if you get it backwards).

**Difficulty:** 🟠–🔴 Intermediate–Advanced  **Estimated time:** 4 hours  **Prerequisite:** [Chapters 44–45](45-building-your-first-real-middleware.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The request pipeline](#2-the-request-pipeline)
3. [How wrapping really works](#3-how-wrapping-really-works)
4. [Watching the onion: an execution-order experiment](#4-watching-the-onion)
5. [Composing middleware: `Chain` and `Stack`](#5-composing-middleware-chain-and-stack)
6. [Global vs. route-level middleware](#6-global-vs-route-level-middleware)
7. [Why not bind middleware to the mux directly?](#7-why-not-bind-middleware-to-the-mux-directly)
8. [Middleware #1: panic recovery](#8-middleware-1-panic-recovery)
9. [Middleware #2: request IDs and `context`](#9-middleware-2-request-ids-and-context)
10. [Middleware #3: JSON errors for the mux's `404`/`405`](#10-middleware-3-json-errors)
11. [Choosing the order](#11-choosing-the-order)
12. [The complete code](#12-the-complete-code)
13. [Tests](#13-tests)
14. [Seeing it work](#14-seeing-it-work)
15. [Common mistakes](#15-common-mistakes)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- The **pipeline / onion model**: request in through the layers, response out through the same layers in reverse
- How a middleware is *actually* just a function wrapping a function (and how to trace the wrapping)
- A **`Stack`** type to declare a pipeline readably and reuse parts of it
- **Recover** from panics so one buggy request can't produce an empty reply
- Attach a **request ID** to each request using **`context.Context`**, and show it in logs and response headers
- Turn the mux's plain-text `404`/`405` into **JSON** with a `ResponseWriter` wrapper
- How to reason about **middleware order**

---

## 2. The request pipeline

Every request that reaches your server passes through a **pipeline**: a sequence of layers, each with one job, ending at your handler. The response then travels back through the same layers in reverse.

```
                       REQUEST ─────────────────────────────────────────►

  client ──► RequestID ──► Logger ──► CORS ──► Recover ──► JSONErrors ──► mux ──► handler
                                                                                      │
  client ◄── RequestID ◄── Logger ◄── CORS ◄── Recover ◄── JSONErrors ◄── mux ◄───────┘

                       ◄───────────────────────────────────────── RESPONSE
```

Each layer can do work:

- **on the way in** (before calling `next`): inspect/modify the request, attach data, reject early;
- **on the way out** (after `next` returns): inspect/log/modify the response.

**Analogy: an onion.** The handler is the core; each middleware is a layer wrapped around it. A request must pass through every layer from the outside in; the response peels back out. Outer layers protect and observe everything inside.

---

## 3. How wrapping really works

Remember (Chapters 17 and 44): a middleware is `func(http.Handler) http.Handler`. Applying one **wraps** a handler and gives you a new handler:

```go
h := handler                   // the core
h = JSONErrors(h)              // layer 1 wraps the core
h = Recover(h)                 // layer 2 wraps layer 1
h = CORS(h)                    // layer 3 wraps layer 2
h = Logger(h)                  // layer 4 wraps layer 3
```

```
   Logger( CORS( Recover( JSONErrors( handler ) ) ) )
   └ outermost                                 └ innermost
```

The final `h` is what you give to `http.ListenAndServe`. When a request arrives, `h.ServeHTTP` runs `Logger`'s code, which calls its `next` (the CORS handler), whose code calls *its* `next`, ... down to the core, and then returns back up. It's just nested function calls, the same as Chapter 4's call stack: **the middleware that was applied last is the outermost, and runs first.**

That reversal is a classic source of confusion: if you wrap them one by one, the *last* one you wrap runs *first*. Nesting `Logger(CORS(...))` visually shows it, but a loop can't be read that way. That's why we'll build `Chain`/`Stack` helpers where **the first middleware you list is the first to run**.

---

## 4. Watching the onion

Let's *see* the order. Here's a tiny experiment (not part of the project) with three middleware that print when they're entered and exited:

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
)

func tag(name string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			fmt.Println("→ enter", name)
			next.ServeHTTP(w, r)
			fmt.Println("← exit ", name)
		})
	}
}

func main() {
	core := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("   ** handler runs **")
	})

	h := tag("A")(tag("B")(tag("C")(core)))
	h.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", "/", nil))
}
```

Real output:

```
→ enter A
→ enter B
→ enter C
   ** handler runs **
← exit  C
← exit  B
← exit  A
```

Enter in order `A, B, C`; exit in reverse `C, B, A`. **First in, last out.** That mirrors the stack frames of nested function calls exactly. Keep this picture in mind whenever you reason about middleware: it explains logging (log *after* to know the status), timing, recovery (`defer` in an outer layer catches inner panics), and why order matters.

---

## 5. Composing middleware: `Chain` and `Stack`

Nesting calls by hand (`A(B(C(core)))`) doesn't scale: it's hard to read and hard to edit. We want to *list* middleware in the order they should run, like a checklist.

### `Chain`

```go
// Middleware is the standard shape of a middleware.
type Middleware func(http.Handler) http.Handler

// Chain wraps h in the given middleware. The FIRST middleware listed is the OUTERMOST,
// so it sees the request first and the response last.
func Chain(h http.Handler, mws ...Middleware) http.Handler {
	for i := len(mws) - 1; i >= 0; i-- { // wrap from the inside out
		h = mws[i](h)
	}
	return h
}
```

Why loop **backwards**? To make the *first* listed middleware the outermost, it must be applied *last*. The loop applies the last-listed (innermost) first.

```
Chain(core, A, B, C)   ≡   A(B(C(core)))
```

### `Stack`: a reusable list

A route-specific pipeline often *extends* the global one (e.g., add authentication to a few routes). A tiny type makes that ergonomic:

```go
// Stack is an ordered, reusable list of middleware.
type Stack struct {
	mws []Middleware
}

// NewStack creates a Stack from the given middleware, outermost first.
func NewStack(mws ...Middleware) *Stack {
	return &Stack{mws: append([]Middleware(nil), mws...)}
}

// Append returns a NEW Stack with extra middleware added inside the existing ones.
// The original Stack is not modified.
func (s *Stack) Append(mws ...Middleware) *Stack {
	all := make([]Middleware, 0, len(s.mws)+len(mws))
	all = append(all, s.mws...)
	all = append(all, mws...)
	return &Stack{mws: all}
}

// Then wraps h with every middleware in the stack.
func (s *Stack) Then(h http.Handler) http.Handler { return Chain(h, s.mws...) }

// ThenFunc is Then for a plain handler function.
func (s *Stack) ThenFunc(f http.HandlerFunc) http.Handler { return s.Then(f) }
```

`Append` **copies** into a fresh slice: two stacks built from the same base can't accidentally share (and overwrite) the same backing array. That's the slice aliasing trap from Chapter 25 in the wild. (Try replacing the copy with `append(s.mws, mws...)` and building two derived stacks; with spare capacity, they'd clobber each other.)

The result is a *declarative* pipeline:

```go
global := middleware.NewStack(
	middleware.RequestID,
	middleware.Logger(logger),
	middleware.CORS,
	middleware.Recover(logger),
)
handler := global.Then(mux)
```

---

## 6. Global vs. route-level middleware

**Global middleware** wraps the entire router: it runs for every request, including unknown paths and preflights. CORS, logging, recovery, and request IDs are global.

**Route-level middleware** wraps *individual handlers* and runs only for those routes, *after* the mux has matched the route. Authentication for specific endpoints is the classic case (Chapter 49):

```go
authed := global.Append(middleware.RequireAuth)     // not written yet: Chapter 49

mux.Handle("GET /products", http.HandlerFunc(handlers.GetProducts))            // public
mux.Handle("POST /products", middleware.Chain(http.HandlerFunc(handlers.CreateProduct),
	middleware.RequireAuth))                                                    // protected
```

Notice `mux.Handle(pattern, handler)` (takes an `http.Handler`) rather than `HandleFunc` (takes a function). Wrapping produces an `http.Handler`, so use `Handle`.

| | Global | Route-level |
|--|--------|-------------|
| Wraps | the whole `mux` | one handler |
| Runs for | every request | only that route |
| Sees unmatched routes (404) | ✅ yes | ❌ no |
| Sees preflight `OPTIONS` | ✅ yes | ❌ no (mux may 405 first) |
| Good for | CORS, logging, recovery, request ID | auth, per-route limits, per-route caching |

---

## 7. Why not bind middleware to the mux directly?

A tempting shortcut is to register *every* route through a helper that wraps the handler:

```go
// ❌ tempting, but wrong for cross-cutting concerns like CORS
mux.Handle("GET /products", cors(handlers.GetProducts))
mux.Handle("POST /products", cors(handlers.CreateProduct))
```

Two problems: **you'll forget one**, and **it can never see requests the mux rejects itself**:

1. A browser's preflight `OPTIONS /products` reaches the mux, which finds no `OPTIONS /products` route and answers `405` *before any per-route middleware runs*. The preflight fails; the real request is blocked.
2. Unknown paths (`404`) and wrong methods (`405`) never get CORS headers, so the browser hides those errors behind a misleading "CORS policy" message.

The correct approach (which we've used since Chapter 43): **wrap the mux itself**, so global middleware sees *everything* first:

```go
http.ListenAndServe(":8080", global.Then(mux))   // ✅ wraps the router
```

Rule: **anything that must apply to all requests, including ones no route matches, wraps the mux. Anything specific to a route wraps that handler.**

---

## 8. Middleware #1: panic recovery

What happens if a handler panics (nil pointer, index out of range, explicit `panic`)? `net/http` recovers each connection's goroutine so *one* request's panic doesn't kill the server, but the client gets **an empty reply / dropped connection** and the default handler logs a stack trace to stderr. We can do much better: respond with a clean `500` JSON error and log the panic properly.

```go
// file: middleware/recover.go
package middleware

import (
	"ecommerce/util"
	"log/slog"
	"net/http"
	"runtime/debug"
)

// Recover turns a panic in any downstream handler into a logged error and a 500 response.
func Recover(logger *slog.Logger) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				rec := recover()
				if rec == nil {
					return
				}
				// http.ErrAbortHandler is how handlers deliberately abort a response; let net/http deal with it.
				if rec == http.ErrAbortHandler {
					panic(rec)
				}

				logger.ErrorContext(r.Context(), "panic recovered",
					"panic", rec,
					"method", r.Method,
					"path", r.URL.Path,
					"request_id", RequestIDFrom(r.Context()),
					"stack", string(debug.Stack()),
				)
				util.SendError(w, http.StatusInternalServerError, "internal server error")
			}()

			next.ServeHTTP(w, r)
		})
	}
}
```

How it works (this is Chapter 34 in action):

- `defer func() { recover() ... }()` runs when the function ends, **including by panic**. `recover()` stops the panic and returns its value (`nil` if there was none).
- We log the panic **with the stack** (`debug.Stack()`) for debugging, but send the client only a **generic message**: never leak internals (file paths, values) to clients.
- If the handler had *already started writing* the response before it panicked, the status is sent and we can't change it; `SendError` would append text to a half-written body. It's a best-effort fix; for critical streaming code, avoid panics.
- We re-panic on `http.ErrAbortHandler`, the documented sentinel used to abort a response silently.

**Placement:** `Recover` must be **outside** everything it should protect (handlers, and other middleware), but *inside* `Logger` (so the log shows the `500`) and inside `CORS` (so the `500` carries CORS headers and the browser can display it). See section 11.

> ⚠️ **Recovering isn't fixing.** A panic is a bug. Recovery keeps the server alive and gives clients a sane response; the alert on `level=ERROR msg="panic recovered"` is how you find and fix the bug.

---

## 9. Middleware #2: request IDs and `context`

When many requests interleave, log lines from different requests are shuffled together. A **request ID**, a unique string attached to each request, lets you filter all logs for one request (and quote it in a support ticket: "my request `9f1c…` failed").

We need to carry the ID *from* the middleware *to* wherever it's needed (the logger, handlers, later database calls) **without changing every function signature**. Go's answer is **`context.Context`**.

### What is `context`?

`context.Context` is a value that travels along with a request (and through goroutines and function calls) carrying:

- **cancellation** signals (e.g., the client disconnected → stop work),
- **deadlines/timeouts**,
- **request-scoped values** (like a request ID or the authenticated user).

Every `*http.Request` has one: `r.Context()`. To attach a value you derive a **new** context (contexts are immutable) and a new request that carries it:

```go
ctx := context.WithValue(r.Context(), myKey, "abc123")
r = r.WithContext(ctx)               // a shallow copy of the request with the new context
next.ServeHTTP(w, r)                 // pass the NEW request down the chain
```

### Context keys: use a private type

Keys must not collide between packages, so the convention is an **unexported type** for keys:

```go
type contextKey string
const requestIDKey contextKey = "request_id"
```

Because the type is unexported, no other package can construct the same key, so no accidental collisions (a plain string key like `"request_id"` could clash with someone else's).

### The middleware

```go
// file: middleware/requestid.go
package middleware

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"net/http"
)

type contextKey string

const requestIDKey contextKey = "request_id"

// RequestIDHeader is the header used to accept and return the request ID.
const RequestIDHeader = "X-Request-ID"

// RequestID gives every request a unique ID, stored in the request context and echoed in the response header.
// If the client (or an upstream proxy) sent a well-formed X-Request-ID, it is reused so IDs can be traced across services.
func RequestID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get(RequestIDHeader)
		if !validRequestID(id) {
			id = newRequestID()
		}

		w.Header().Set(RequestIDHeader, id)
		ctx := context.WithValue(r.Context(), requestIDKey, id)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// RequestIDFrom returns the request ID stored in ctx, or "" if there is none.
func RequestIDFrom(ctx context.Context) string {
	id, _ := ctx.Value(requestIDKey).(string)
	return id
}

func newRequestID() string {
	var b [8]byte
	if _, err := rand.Read(b[:]); err != nil {
		return "unknown"
	}
	return hex.EncodeToString(b[:]) // 16 hex characters
}

// validRequestID accepts short IDs made of safe characters, so a client can't inject
// newlines or huge strings into our logs (log injection).
func validRequestID(id string) bool {
	if id == "" || len(id) > 64 {
		return false
	}
	for _, c := range id {
		switch {
		case c >= 'a' && c <= 'z', c >= 'A' && c <= 'Z', c >= '0' && c <= '9', c == '-', c == '_', c == '.':
		default:
			return false
		}
	}
	return true
}
```

Key points:

- `crypto/rand` (not `math/rand`) gives unpredictable IDs. 8 random bytes → 16 hex characters: plenty for correlation.
- **Trust but verify** inbound IDs: reuse a client-provided ID only if it's short and made of safe characters. Otherwise a malicious client could put newlines in the header and forge log lines.
- We **echo** the ID in the response header so the client (or a support agent) can quote it.
- Handlers retrieve it anywhere with `middleware.RequestIDFrom(r.Context())`.

### Updating the logger

The logger (Chapter 45) now includes the ID as one more attribute. Only the `LogAttrs` call changes:

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
func Logger(logger *slog.Logger) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			rec := &statusRecorder{ResponseWriter: w}

			next.ServeHTTP(rec, r)

			status := rec.status
			if status == 0 {
				status = http.StatusOK
			}

			level := slog.LevelInfo
			switch {
			case status >= 500:
				level = slog.LevelError
			case status >= 400:
				level = slog.LevelWarn
			}

			logger.LogAttrs(r.Context(), level, "request",
				slog.String("request_id", RequestIDFrom(r.Context())),
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

(Changes from Chapter 45: the return type is now the named `Middleware` type, and there's a `request_id` attribute. `Logger` must be placed *inside* `RequestID` so the context already holds the ID when the logger runs.)

---

## 10. Middleware #3: JSON errors

In Chapter 44 we noticed the mux's automatic `404`/`405` responses are plain text (`404 page not found`), unlike the rest of our JSON API. We fix that with a `ResponseWriter` wrapper (the same technique as the logger):

```go
// file: middleware/jsonerrors.go
package middleware

import (
	"encoding/json"
	"net/http"
	"strings"
)

// jsonErrorWriter intercepts the mux's own plain-text 404/405 responses and replaces them with JSON.
// Responses produced by our handlers (already application/json) pass through untouched.
type jsonErrorWriter struct {
	http.ResponseWriter
	replaced bool
}

func (w *jsonErrorWriter) WriteHeader(code int) {
	isPlain := strings.HasPrefix(w.Header().Get("Content-Type"), "text/plain")
	if isPlain && (code == http.StatusNotFound || code == http.StatusMethodNotAllowed) {
		message := "route not found"
		if code == http.StatusMethodNotAllowed {
			message = "method not allowed"
		}
		w.replaced = true
		w.Header().Set("Content-Type", "application/json")
		w.Header().Del("Content-Length")
		w.ResponseWriter.WriteHeader(code)
		json.NewEncoder(w.ResponseWriter).Encode(map[string]string{"error": message})
		return
	}
	w.ResponseWriter.WriteHeader(code)
}

func (w *jsonErrorWriter) Write(b []byte) (int, error) {
	if w.replaced {
		return len(b), nil // swallow the mux's plain-text body: we already sent JSON
	}
	return w.ResponseWriter.Write(b)
}

func (w *jsonErrorWriter) Unwrap() http.ResponseWriter { return w.ResponseWriter }

// JSONErrors makes the router's built-in 404 and 405 responses use the API's JSON error format.
func JSONErrors(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		next.ServeHTTP(&jsonErrorWriter{ResponseWriter: w}, r)
	})
}
```

How it works: `http.NotFound`/`http.Error` set `Content-Type: text/plain` and then call `WriteHeader(404)`. Our wrapper spots that combination *at `WriteHeader` time*, rewrites the headers, writes the JSON itself, and then **discards** the plain-text body that follows. Any error our own handlers send (JSON) has `application/json` set, so it's left alone. The `Allow` header set by the mux for `405` is preserved.

---

## 11. Choosing the order

Recall the rule: **the first middleware in the list runs first on the request and last on the response.** Our pipeline:

```go
middleware.NewStack(
	middleware.RequestID,          // 1
	middleware.Logger(logger),     // 2
	middleware.CORS,               // 3
	middleware.Recover(logger),    // 4
	middleware.JSONErrors,         // 5
)
```

Reasoning for each position:

| # | Middleware | Why here |
|---|-----------|----------|
| 1 | **RequestID** | Must be outermost of the two that *use* the ID: everything after it (logger, recovery, handlers) can read the ID from the context |
| 2 | **Logger** | Outside the rest so it logs *every* request, including preflights answered by CORS and `500`s produced by `Recover`; it measures the total time |
| 3 | **CORS** | Before `Recover`/handlers so *every* response, including a recovered `500`, carries CORS headers; must not be behind auth (Chapter 49) or preflights fail |
| 4 | **Recover** | Inside `Logger`/`CORS` (so the `500` is logged and carries CORS headers) but outside `JSONErrors` and the router so any panic below is caught |
| 5 | **JSONErrors** | Closest to the mux, so it can see and rewrite the mux's own errors |

The mistakes that come from the wrong order (each one is a real bug people ship):

| Wrong order | Symptom |
|-------------|---------|
| `Auth` **outside** `CORS` | Browser preflights get `401`; every cross-origin call breaks |
| `Logger` **inside** `Recover` | A panicking request is never logged (the panic unwinds through the logger's "after" code) |
| `Recover` **outside** `CORS` | Recovered `500`s lack CORS headers → browser shows a misleading CORS error instead of the `500` |
| `RequestID` **inside** `Logger` | Log lines have an empty `request_id` |
| `Recover` **inside** `JSONErrors`' target (only around handlers) | Panics in other middleware aren't caught |

A helpful general recipe (outermost → innermost): **request ID → logging → CORS → panic recovery → rate limiting → authentication → routing → handler.**

---

## 12. The complete code

New/changed files this chapter: `middleware/chain.go` (new), `middleware/recover.go` (new), `middleware/requestid.go` (new), `middleware/jsonerrors.go` (new), `middleware/logger.go` (updated), `middleware/cors.go` (updated to the `Middleware` type: just its signature, since a plain function already satisfies it), and `main.go`.

```go
// file: middleware/chain.go
package middleware

import "net/http"

// Middleware is the standard shape of a middleware: it wraps a handler and returns a new one.
type Middleware func(http.Handler) http.Handler

// Chain wraps h in the given middleware. The FIRST middleware listed is the OUTERMOST,
// so it sees the request first and the response last.
func Chain(h http.Handler, mws ...Middleware) http.Handler {
	for i := len(mws) - 1; i >= 0; i-- {
		h = mws[i](h)
	}
	return h
}

// Stack is an ordered, reusable list of middleware.
type Stack struct {
	mws []Middleware
}

// NewStack creates a Stack from the given middleware, outermost first.
func NewStack(mws ...Middleware) *Stack {
	return &Stack{mws: append([]Middleware(nil), mws...)}
}

// Append returns a NEW Stack with extra middleware added inside the existing ones.
// The receiver is not modified.
func (s *Stack) Append(mws ...Middleware) *Stack {
	all := make([]Middleware, 0, len(s.mws)+len(mws))
	all = append(all, s.mws...)
	all = append(all, mws...)
	return &Stack{mws: all}
}

// Then wraps h with every middleware in the stack.
func (s *Stack) Then(h http.Handler) http.Handler { return Chain(h, s.mws...) }

// ThenFunc is Then for a plain handler function.
func (s *Stack) ThenFunc(f http.HandlerFunc) http.Handler { return s.Then(f) }
```

`middleware/cors.go` is unchanged from Chapter 43 (`func CORS(next http.Handler) http.Handler`); it already has the right shape to be used as a `Middleware` value.

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

	// The global pipeline, outermost first.
	global := middleware.NewStack(
		middleware.RequestID,
		middleware.Logger(logger),
		middleware.CORS,
		middleware.Recover(logger),
		middleware.JSONErrors,
	)
	return global.Then(mux)
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

---

## 13. Tests

Each middleware gets small, focused tests that use a dummy `next`, one of the payoffs of the `func(http.Handler) http.Handler` design.

```go
// file: middleware/pipeline_test.go
package middleware

import (
	"bytes"
	"encoding/json"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func quiet() (*slog.Logger, *bytes.Buffer) {
	var buf bytes.Buffer
	return slog.New(slog.NewTextHandler(&buf, nil)), &buf
}

// ---- Chain / Stack -----------------------------------------------------------

func recordingMiddleware(name string, calls *[]string) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			*calls = append(*calls, "enter "+name)
			next.ServeHTTP(w, r)
			*calls = append(*calls, "exit "+name)
		})
	}
}

func TestChainRunsFirstListedFirst(t *testing.T) {
	var calls []string
	core := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) { calls = append(calls, "handler") })

	h := Chain(core, recordingMiddleware("A", &calls), recordingMiddleware("B", &calls), recordingMiddleware("C", &calls))
	h.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", "/", nil))

	got := strings.Join(calls, ", ")
	want := "enter A, enter B, enter C, handler, exit C, exit B, exit A"
	if got != want {
		t.Errorf("order:\n got  %s\n want %s", got, want)
	}
}

func TestStackAppendDoesNotModifyOriginal(t *testing.T) {
	var calls []string
	base := NewStack(recordingMiddleware("base", &calls))
	extended := base.Append(recordingMiddleware("extra", &calls))
	other := base.Append(recordingMiddleware("other", &calls)) // a second stack from the same base

	core := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {})

	base.Then(core).ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", "/", nil))
	if strings.Contains(strings.Join(calls, ","), "extra") {
		t.Error("base stack was modified by Append")
	}

	calls = nil
	extended.Then(core).ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", "/", nil))
	if got := strings.Join(calls, ","); got != "enter base,enter extra,exit extra,exit base" {
		t.Errorf("extended stack ran %s", got)
	}

	calls = nil
	other.Then(core).ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", "/", nil))
	if strings.Contains(strings.Join(calls, ","), "extra") {
		t.Error("two stacks derived from one base interfere with each other")
	}
}

// ---- Recover -----------------------------------------------------------------

func TestRecoverTurnsPanicInto500JSON(t *testing.T) {
	logger, logs := quiet()
	h := Chain(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		panic("boom")
	}), Recover(logger))

	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest("GET", "/x", nil))

	if rec.Code != http.StatusInternalServerError {
		t.Fatalf("expected 500, got %d", rec.Code)
	}
	var body map[string]string
	if err := json.NewDecoder(rec.Body).Decode(&body); err != nil || body["error"] != "internal server error" {
		t.Errorf("unexpected body %q (%v)", rec.Body.String(), err)
	}
	if strings.Contains(rec.Body.String(), "boom") {
		t.Error("the panic message must not leak to the client")
	}
	if !strings.Contains(logs.String(), "panic recovered") || !strings.Contains(logs.String(), "boom") {
		t.Errorf("panic not logged: %q", logs.String())
	}
}

func TestRecoverLeavesNormalRequestsAlone(t *testing.T) {
	logger, logs := quiet()
	h := Chain(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusAccepted)
	}), Recover(logger))

	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))
	if rec.Code != http.StatusAccepted || logs.Len() != 0 {
		t.Errorf("code=%d logs=%q", rec.Code, logs.String())
	}
}

func TestRecoverRepanicsOnErrAbortHandler(t *testing.T) {
	logger, _ := quiet()
	h := Chain(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		panic(http.ErrAbortHandler)
	}), Recover(logger))

	defer func() {
		if r := recover(); r != http.ErrAbortHandler {
			t.Errorf("expected ErrAbortHandler to propagate, got %v", r)
		}
	}()
	h.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", "/", nil))
}

// ---- RequestID ----------------------------------------------------------------

func TestRequestIDIsGeneratedAndExposed(t *testing.T) {
	var seen string
	h := RequestID(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		seen = RequestIDFrom(r.Context())
	}))

	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))

	if len(seen) != 16 {
		t.Errorf("expected a 16-character ID in the context, got %q", seen)
	}
	if rec.Header().Get(RequestIDHeader) != seen {
		t.Errorf("response header %q != context %q", rec.Header().Get(RequestIDHeader), seen)
	}
}

func TestRequestIDReusesGoodClientID(t *testing.T) {
	var seen string
	h := RequestID(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) { seen = RequestIDFrom(r.Context()) }))

	req := httptest.NewRequest("GET", "/", nil)
	req.Header.Set(RequestIDHeader, "trace-abc_123.x")
	h.ServeHTTP(httptest.NewRecorder(), req)

	if seen != "trace-abc_123.x" {
		t.Errorf("expected the client's ID to be reused, got %q", seen)
	}
}

func TestRequestIDReplacesBadClientIDs(t *testing.T) {
	for _, bad := range []string{"has space", "evil\nINFO fake log line", strings.Repeat("a", 65), "semi;colon"} {
		var seen string
		h := RequestID(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) { seen = RequestIDFrom(r.Context()) }))

		req := httptest.NewRequest("GET", "/", nil)
		req.Header["X-Request-Id"] = []string{bad}
		h.ServeHTTP(httptest.NewRecorder(), req)

		if seen == bad || len(seen) != 16 {
			t.Errorf("bad ID %q was not replaced (got %q)", bad, seen)
		}
	}
}

func TestRequestIDAppearsInLogs(t *testing.T) {
	logger, logs := quiet()
	h := Chain(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {}), RequestID, Logger(logger))

	req := httptest.NewRequest("GET", "/x", nil)
	req.Header.Set(RequestIDHeader, "req-42")
	h.ServeHTTP(httptest.NewRecorder(), req)

	if !strings.Contains(logs.String(), "request_id=req-42") {
		t.Errorf("log line lacks the request ID: %q", logs.String())
	}
}

// ---- JSONErrors ---------------------------------------------------------------

func TestJSONErrorsRewritesMuxErrors(t *testing.T) {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /only", func(w http.ResponseWriter, r *http.Request) { w.Write([]byte("fine")) })
	h := JSONErrors(mux)

	// unknown route → JSON 404
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest("GET", "/missing", nil))
	if rec.Code != 404 || rec.Header().Get("Content-Type") != "application/json" || !strings.Contains(rec.Body.String(), `"route not found"`) {
		t.Errorf("404: code=%d ct=%q body=%q", rec.Code, rec.Header().Get("Content-Type"), rec.Body.String())
	}

	// wrong method → JSON 405, Allow header preserved
	rec = httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest("POST", "/only", nil))
	if rec.Code != 405 || !strings.Contains(rec.Body.String(), `"method not allowed"`) {
		t.Errorf("405: code=%d body=%q", rec.Code, rec.Body.String())
	}
	if rec.Header().Get("Allow") == "" {
		t.Error("the Allow header must survive")
	}

	// normal responses are untouched
	rec = httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest("GET", "/only", nil))
	if rec.Code != 200 || rec.Body.String() != "fine" {
		t.Errorf("200: code=%d body=%q", rec.Code, rec.Body.String())
	}
}

func TestJSONErrorsLeavesHandlerJSONAlone(t *testing.T) {
	h := JSONErrors(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusNotFound)
		w.Write([]byte(`{"error":"product not found"}`))
	}))

	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))
	if rec.Body.String() != `{"error":"product not found"}` {
		t.Errorf("handler's own JSON error was altered: %q", rec.Body.String())
	}
}
```

The updated `middleware/logger_test.go` from Chapter 45 needs one adjustment: its assertions still hold (the log line simply gains a `request_id=` field, empty when the middleware runs alone), so it passes unchanged. Add one integration test to `main_test.go` confirming the pipeline is assembled correctly:

```go
// file: pipeline_test.go
package main

import (
	"net/http"
	"strings"
	"testing"
)

func TestPipelineIntegration(t *testing.T) {
	origin := map[string]string{"Origin": "http://localhost:5173"}

	// unknown route: JSON 404, with CORS headers and a request ID
	rec := do(http.MethodGet, "/no-such-route", "", origin)
	if rec.Code != 404 || !strings.Contains(rec.Body.String(), `{"error":"route not found"}`) {
		t.Errorf("404: code=%d body=%q", rec.Code, rec.Body.String())
	}
	if rec.Header().Get("Access-Control-Allow-Origin") != "http://localhost:5173" {
		t.Error("404 is missing CORS headers")
	}
	if len(rec.Header().Get("X-Request-ID")) != 16 {
		t.Errorf("missing/odd X-Request-ID: %q", rec.Header().Get("X-Request-ID"))
	}

	// wrong method: JSON 405 with Allow
	rec = do(http.MethodDelete, "/products", "", nil)
	if rec.Code != 405 || !strings.Contains(rec.Body.String(), "method not allowed") || rec.Header().Get("Allow") != "GET, HEAD, POST" {
		t.Errorf("405: code=%d allow=%q body=%q", rec.Code, rec.Header().Get("Allow"), rec.Body.String())
	}

	// a client-supplied request ID round-trips
	rec = do(http.MethodGet, "/products", "", map[string]string{"X-Request-ID": "abc-123"})
	if rec.Header().Get("X-Request-ID") != "abc-123" {
		t.Errorf("request ID not echoed: %q", rec.Header().Get("X-Request-ID"))
	}
}
```

```bash
go test -race ./...
```

```
ok  	ecommerce	1.0s
ok  	ecommerce/middleware	1.0s
```

---

## 14. Seeing it work

**JSON errors and request IDs** (real output):

```bash
curl -si http://localhost:8080/nothing
```

```
HTTP/1.1 404 Not Found
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Request-Id: 5f0e5a7f9b2c41d3
Date: Sat, 26 Sep 2026 07:20:44 GMT
Content-Length: 28

{"error":"route not found"}
```

```bash
curl -si -X DELETE http://localhost:8080/products | head -6
```

```
HTTP/1.1 405 Method Not Allowed
Allow: GET, HEAD, POST
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Request-Id: 1a6c0e8b47d29f35
...
{"error":"method not allowed"}
```

**A client-supplied ID is reused** and appears in the logs:

```bash
curl -s -H "X-Request-ID: order-flow-7" http://localhost:8080/products/2 -o /dev/null -D - | grep -i x-request
# X-Request-Id: order-flow-7
```

```
time=... level=INFO msg=request request_id=order-flow-7 method=GET path=/products/2 status=200 bytes=120 duration=41µs remote=[::1]:5xxxx
```

**A recovered panic.** Temporarily add a route that panics to see `Recover` at work (don't keep this in the real project):

```go
mux.HandleFunc("GET /debug/panic", func(w http.ResponseWriter, r *http.Request) {
	var p *models.Product
	_ = p.Title // nil pointer dereference → panic
})
```

```bash
curl -si http://localhost:8080/debug/panic
```

```
HTTP/1.1 500 Internal Server Error
Content-Type: application/json
X-Request-Id: 8c3d0b6a2f7e1594
...
{"error":"internal server error"}
```

The server keeps running, the client gets a clean JSON `500`, and the log has the details, correlated by request ID:

```
time=... level=ERROR msg="panic recovered" panic="runtime error: invalid memory address or nil pointer dereference" method=GET path=/debug/panic request_id=8c3d0b6a2f7e1594 stack="goroutine 24 [running]:\nruntime/debug.Stack()..."
time=... level=ERROR msg=request request_id=8c3d0b6a2f7e1594 method=GET path=/debug/panic status=500 bytes=34 duration=180µs remote=[::1]:5xxxx
```

Two log lines, one request ID: the panic details (with a stack trace) and the access-log entry showing the `500`. That's why the ID matters.

---

## 15. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Expecting the last-wrapped middleware to run last | It runs **first** | Use `Chain`/`Stack` (first listed = outermost) |
| 2 | Not calling `next.ServeHTTP` (unintentionally) | Requests hang or return empty | Call it on every non-short-circuit path |
| 3 | Modifying the request without `r.WithContext`/clone | Shared-state bugs | Derive a new request |
| 4 | Passing the *old* request `r` to `next` after `WithContext` | Downstream can't see the new context | Pass `r.WithContext(ctx)` |
| 5 | String context keys (`"request_id"`) | Key collisions between packages | Private key type |
| 6 | Trusting client-provided IDs blindly | Log injection, huge headers | Validate length and charset |
| 7 | `Recover` inside `Logger` | Panics go unlogged | `Recover` inside `Logger` and `CORS`, outside the handlers |
| 8 | Leaking panic messages/stack to clients | Information disclosure | Log details; return a generic 500 |
| 9 | Global mutable state in middleware (e.g., a shared slice) | Data races | Per-request data in context; shared state behind a mutex |
| 10 | `append(s.mws, ...)` in `Append` | Aliased backing arrays: two derived stacks clobber each other | Copy into a fresh slice |
| 11 | Route-level CORS | Preflights hit the mux and 405 | Wrap the mux with global CORS |
| 12 | Doing slow work in a middleware for every request | Latency for everyone | Keep middleware cheap; move heavy work into specific handlers |

---

## 16. Exercises

### Exercise 1: Trace the order
Given `Chain(core, A, B, C)`, write the sequence of enter/exit events if `B` short-circuits (returns without calling `next`).

<details><summary>Solution</summary>

`enter A`, `enter B` (B responds and returns), `exit A`. `C` and `core` never run.
</details>

### Exercise 2: A timing header
Write `Timing` middleware that adds an `X-Response-Time` header. Use a `ResponseWriter` wrapper that sets the header in its `WriteHeader` (since headers can't be set after writing has begun).

<details><summary>Solution</summary>

```go
type timingWriter struct {
	http.ResponseWriter
	start time.Time
	wrote bool
}

func (t *timingWriter) WriteHeader(code int) {
	if !t.wrote {
		t.wrote = true
		t.Header().Set("X-Response-Time", time.Since(t.start).String())
	}
	t.ResponseWriter.WriteHeader(code)
}

func (t *timingWriter) Write(b []byte) (int, error) {
	if !t.wrote { // implicit 200
		t.WriteHeader(http.StatusOK)
	}
	return t.ResponseWriter.Write(b)
}

func (t *timingWriter) Unwrap() http.ResponseWriter { return t.ResponseWriter }

func Timing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		next.ServeHTTP(&timingWriter{ResponseWriter: w, start: time.Now()}, r)
	})
}
```
</details>

### Exercise 3: Secure headers
Write `SecureHeaders` middleware that sets `X-Content-Type-Options: nosniff` and `X-Frame-Options: DENY` on every response. Where in the stack should it go?

<details><summary>Solution</summary>

```go
func SecureHeaders(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h := w.Header()
		h.Set("X-Content-Type-Options", "nosniff")
		h.Set("X-Frame-Options", "DENY")
		next.ServeHTTP(w, r)
	})
}
```
Headers can be set *before* `next`, so it can sit almost anywhere; put it near the outside so even errors from inner layers have them (e.g., right after `RequestID`).
</details>

### Exercise 4: Panic in middleware
A bug in `CORS` itself causes a panic. With the order `RequestID, Logger, CORS, Recover, JSONErrors`, is it caught by `Recover`? What would you change so it is?

<details><summary>Solution</summary>

No: `CORS` is *outside* `Recover`, so its panic isn't caught by our `Recover` (`net/http` will still recover the goroutine, but the client gets a dropped connection). To catch it, move `Recover` outside `CORS`, at the cost that recovered `500`s then lack CORS headers, unless you *also* make `Recover` set CORS headers or add a second, outer, minimal `Recover`. It's a trade-off: keep middleware simple and well-tested to make panics in them very unlikely.
</details>

### Exercise 5: Use the ID in a handler
Add the request ID to the JSON body of `500` errors: `{"error":"internal server error","requestId":"..."}` so users can quote it. Which function do you change?

<details><summary>Solution</summary>

In `Recover`, build the body with the ID:
```go
util.SendData(w, http.StatusInternalServerError, map[string]string{
	"error":     "internal server error",
	"requestId": RequestIDFrom(r.Context()),
})
```
(Update the test accordingly.)
</details>

### Exercise 6: Route-level middleware
Write `Only(method string) Middleware` that responds `405` unless `r.Method == method`, and apply it to a single handler with `Chain`. Why isn't this necessary with Go 1.22 patterns?

<details><summary>Solution</summary>

```go
func Only(method string) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if r.Method != method {
				w.Header().Set("Allow", method)
				util.SendError(w, http.StatusMethodNotAllowed, "method not allowed")
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```
With method patterns (`"GET /x"`) the mux enforces methods itself, so this is redundant, an example of a good middleware becoming unnecessary when the platform improves.
</details>

### Exercise 7 (challenge): Rate limiting
Design (don't code) a `RateLimit` middleware allowing 10 requests per minute per client IP. Where does it go in the stack? What shared state does it need, and how do you keep it safe with concurrent requests?

<details><summary>Solution</summary>

Place it after `CORS`/`Recover` and before authentication (cheap rejection early; preflights aren't counted). It needs a map from client key (IP, or user ID after auth) to a counter/token-bucket with timestamps, protected by a `sync.Mutex` (or sharded/atomic counters), plus a background cleanup to evict stale entries so memory doesn't grow forever. Respond `429 Too Many Requests` with a `Retry-After` header. The standard library's `golang.org/x/time/rate` provides a ready-made token bucket.
</details>

---

## 17. Quiz

1. In `Chain(core, A, B, C)`, which middleware sees the request first?
2. Why does `Chain` loop from the last middleware to the first?
3. Why can't request-specific data live in a package-level variable?
4. What is a context key, and why use a private type?
5. Why should `CORS` come before `Recover` in the stack?
6. Why bind global middleware to the mux rather than to each route?
7. What does `recover()` return when there's no panic?

<details><summary>Answers</summary>

1. `A`.
2. To make the first-listed middleware the last one applied, and therefore the outermost.
3. Concurrent requests would overwrite each other's data; request-scoped values belong in the request's context.
4. The key for `context.WithValue`; a private type prevents collisions with other packages' keys.
5. So that a recovered `500` still carries CORS headers and the browser can display it.
6. Route-level middleware can't see requests the mux rejects itself (404, 405, preflights).
7. `nil`.
</details>

---

## 18. Summary

- A **request pipeline** is an onion of middleware: requests go **in** through the layers, responses come **out** in reverse. The **first-listed middleware is outermost**.
- `Chain(h, mws...)` and a small **`Stack`** type (with a copying `Append`) make pipelines readable and reusable; use `mux.Handle` for route-level wrapping.
- **Global middleware wraps the mux** (so it sees preflights and 404/405); route-level middleware wraps a single handler.
- **`Recover`** converts panics into logged errors and a generic JSON `500`, and must sit *inside* `Logger` and `CORS` but outside handlers.
- **`RequestID`** stores a unique ID in **`context.Context`** (private key type), echoes it in `X-Request-ID`, validates inbound IDs, and is used by the logger to correlate lines.
- **`JSONErrors`** rewrites the mux's plain-text `404`/`405` using a `ResponseWriter` wrapper.
- **Order matters:** *request ID → logging → CORS → recovery → (rate limit) → auth → routing → handler.* The wrong order produces classic bugs (401 on preflights, unlogged panics, CORS errors hiding 500s).

### ➡️ What's next?

**Part 9** begins. [Chapter 47](47-a-real-project-structure.md) tackles **configuration management and a real project structure**: reading settings from environment variables and `.env` files, a `cmd/` folder, timeouts, and graceful shutdown.
