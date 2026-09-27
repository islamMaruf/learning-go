# Chapter 47: A Real Project Structure — Configuration Management, Timeouts, and Graceful Shutdown

> **Goal of this chapter:** Stop hard-coding values. You'll move settings (port, allowed CORS origins, log format, environment name) out of the source code and into **environment variables** and a **`.env` file**, build a validated **`config` package**, reorganize the project into a conventional layout (`main.go`, `cmd/`, `config/`, `rest/`), configure the `http.Server` with **timeouts**, and add **graceful shutdown** so deployments never cut requests off mid-flight. You'll also practice string ↔ number **type conversion** with `strconv`.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 4 hours  **Prerequisite:** [Chapter 46](46-advanced-middleware.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The problem with hard-coded values](#2-the-problem-with-hard-coded-values)
3. [Environment variables](#3-environment-variables)
4. [The `.env` file and `godotenv`](#4-the-env-file-and-godotenv)
5. [Design: a `Config` value, not a global](#5-design-a-config-value-not-a-global)
6. [Type conversion: string ↔ number](#6-type-conversion-string--number)
7. [The `config` package](#7-the-config-package)
8. [The new project layout](#8-the-new-project-layout)
9. [`http.Server`: timeouts and graceful shutdown](#9-httpserver-timeouts-and-graceful-shutdown)
10. [The complete code](#10-the-complete-code)
11. [Tests](#11-tests)
12. [Running it](#12-running-it)
13. [Exit codes and failing fast](#13-exit-codes-and-failing-fast)
14. [Security and workflow notes](#14-security-and-workflow-notes)
15. [Common mistakes](#15-common-mistakes)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- Why configuration must live **outside** the code (the "twelve-factor" rule)
- How **environment variables** work, and `os.Getenv` vs. `os.LookupEnv`
- How to use a **`.env`** file in development with `godotenv`, and why it's never committed
- How to convert values with `strconv` (`Atoi`, `ParseInt`, `ParseBool`, `Itoa`, `FormatInt`) and `time.ParseDuration`
- How to build a **validated `Config`** that reports *all* problems at once
- How to structure a Go service: `main.go` → `cmd/` → `rest/` + `config/`
- Why servers need **timeouts**, and how to shut down **gracefully** on `SIGINT`/`SIGTERM`

---

## 2. The problem with hard-coded values

Our `main.go` currently contains:

```go
http.ListenAndServe(":8080", newRouter(logger))          // the port
var allowedOrigins = map[string]bool{"http://localhost:5173": true}  // CORS origins
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))  // log format
```

That's fine on your laptop. In real life the *same code* must run in several **environments**:

| Setting | Development | Production |
|---------|-------------|------------|
| Port | `8080` | `80` or whatever the platform assigns |
| Allowed origins | `http://localhost:5173` | `https://shop.example.com` |
| Log format | human-readable text | JSON (for log aggregators) |
| Database | local Postgres | managed cloud database |
| Secrets (JWT key, DB password) | dummy | real, and never in Git |

If those live in the code you must **edit and recompile** for every environment: error-prone, slow, and dangerous: a real password ends up in your Git history *forever*.

**The twelve-factor rule:** *Store config in the environment.* Code is the same everywhere; **configuration varies per deployment and is injected from outside.**

```
              same compiled binary
                      │
   ┌──────────────────┼──────────────────┐
 dev env vars      staging env vars    production env vars
 PORT=8080         PORT=8081           PORT=80
 LOG=text          LOG=json            LOG=json
```

**A test for "is this config?"** Could you open-source the code right now without leaking anything? If not, something belongs in configuration.

---

## 3. Environment variables

An **environment variable** is a named string value that the operating system attaches to a **process** (Chapter 28). Every process inherits its parent's environment at start-up. You can see yours:

```bash
env | head          # Linux/macOS: list them all
echo $HOME          # print one
```

Set one for a **single command** (Linux/macOS):

```bash
HTTP_PORT=9090 go run .
```

Windows PowerShell: `$env:HTTP_PORT = "9090"; go run .`  •  Windows cmd: `set HTTP_PORT=9090 && go run .`

Or `export HTTP_PORT=9090` to set it for the whole shell session.

### Reading them in Go

```go
port := os.Getenv("HTTP_PORT")            // "" if unset OR set to empty: can't tell which
value, ok := os.LookupEnv("HTTP_PORT")    // ok == false only when it is NOT set at all
```

- **`os.Getenv`** is convenient; it returns `""` for both "unset" and "set to empty".
- **`os.LookupEnv`** tells you whether it was set: use it when the difference matters (e.g., deliberately empty means "disable").

**Everything is a string.** Ports, booleans, durations, lists: all arrive as text and *you* convert them (section 6).

**Naming convention:** `UPPER_SNAKE_CASE`, optionally prefixed (`ECOMMERCE_HTTP_PORT`) to avoid clashes with other programs.

---

## 4. The `.env` file and `godotenv`

Typing `export ...` for six variables every morning is tedious. A **`.env` file** is a plain-text list of `KEY=value` pairs kept next to your project for **local development**:

```env
# file: .env.example
# Copy this file to ".env" and adjust. Real .env files are NEVER committed to Git.
APP_ENV=development
HTTP_PORT=8080
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:3000
LOG_FORMAT=text
```

The Go standard library doesn't read `.env` files; the small, popular package **`github.com/joho/godotenv`** does:

```bash
go get github.com/joho/godotenv
```

This adds a `require` line to `go.mod` and checksums to `go.sum` (Chapter 10):

```
require github.com/joho/godotenv v1.5.1
```

```go
import "github.com/joho/godotenv"

godotenv.Load() // reads ".env" and puts its pairs into the process environment
```

Two behaviors that matter:

1. **`Load` does not override variables that are already set.** So a real environment variable (from Docker, Kubernetes, your CI system, or `HTTP_PORT=9090 go run .`) always **wins** over `.env`. That gives you the right precedence: *real environment > .env file > code defaults.*
2. If `.env` **doesn't exist**, `Load` returns an error (an `fs.ErrNotExist`). In production there's usually no `.env` (values come from the platform), so we **ignore** "file not found" but report other errors (e.g., a malformed file).

### `.gitignore`

```text
# file: .gitignore
# Local configuration and secrets: never commit these
.env

# Build output
/ecommerce
*.exe
*.test
```

Commit **`.env.example`** (documentation of which variables exist, with safe dummy values) and **ignore `.env`** (your real local values). New teammates copy `.env.example` to `.env`.

> 🔐 If you ever commit a real secret, **assume it's compromised**: rotate it immediately. Deleting the file in a later commit doesn't remove it from history.

---

## 5. Design: a `Config` value, not a global

There are two common ways to expose configuration to the rest of the program:

**A. A package-level global** (simple; often seen in tutorials):

```go
package config

var Port int          // set once at start-up; read from anywhere as config.Port
func Load() { ... }
```

**B. A `Config` struct that you load once and pass to whoever needs it** (what we'll do):

```go
type Config struct { Port string; ... }
func Load() (*Config, error)

cfg, err := config.Load()
rest.NewRouter(logger, cfg)      // dependencies are explicit parameters
```

Why B?

| | Global | Passed `Config` |
|--|--------|-----------------|
| Dependencies visible in signatures? | ❌ hidden | ✅ explicit |
| Testing with different settings | Must mutate global state (tests interfere) | Just build a `Config` literal per test |
| Race conditions | Possible if anything writes it | None: treat it as read-only |
| Import cycles | Every package imports `config` | Only the packages that receive it |

Chapters 50–51 generalize this idea into **dependency injection**. For now: `main` loads the config, and everything downstream *receives* what it needs.

---

## 6. Type conversion: string ↔ number

Environment variables are strings, so configuration code is full of conversions. Chapter 2 introduced `strconv`; here's the complete toolbox.

### String → number

```go
port, err := strconv.Atoi("8080")                    // ASCII to int
n, err := strconv.ParseInt("ff", 16, 64)             // string, base (2,8,10,16, or 0=auto), bit size → int64
u, err := strconv.ParseUint("42", 10, 64)
f, err := strconv.ParseFloat("3.14", 64)
b, err := strconv.ParseBool("true")                  // accepts 1, t, T, TRUE, true, True, 0, f, F, FALSE, false, False
d, err := time.ParseDuration("1m30s")                // durations: "500ms", "10s", "2h45m"
```

`ParseInt(s, base, bitSize)`, the parameters:

| Parameter | Meaning |
|-----------|---------|
| `s` | The text |
| `base` | 2, 8, 10, 16; `0` = infer from prefix (`0x1F`, `0b101`, `0o17`) |
| `bitSize` | The integer size the result must fit: `0`=int, 8, 16, 32, 64. Values that don't fit produce an error |

**Every parse returns an `error`.** Always check it: user-supplied text can be anything: `"8O80"` (letter O), `""`, `"99999999999999999999"`.

```go
_, err := strconv.Atoi("8O80")
fmt.Println(err) // strconv.Atoi: parsing "8O80": invalid syntax
```

### Number → string

```go
s1 := strconv.Itoa(8080)                       // int → "8080"  (the everyday choice)
s2 := strconv.FormatInt(int64(8080), 10)       // any integer type → string, in the given base
s3 := fmt.Sprintf("%d", 8080)                  // most flexible; slightly slower
s4 := strconv.FormatInt(255, 2)                // "11111111" (binary)
```

A classic trap:

```go
n := 65
fmt.Println(string(n))          // "A"  ← a CHARACTER, not "65"! (go vet warns)
fmt.Println(strconv.Itoa(n))    // "65" ✓
```

| Task | Use |
|------|-----|
| `int` → text | `strconv.Itoa(n)` |
| `int64` (or other) → text | `strconv.FormatInt(n, 10)` |
| Anything → formatted text | `fmt.Sprintf` |
| Text → `int` | `strconv.Atoi(s)` |
| Text → other integer types | `strconv.ParseInt/ParseUint` + conversion |

---

## 7. The `config` package

Requirements for our loader:

1. **Defaults** for optional settings (so the app runs with zero configuration in development).
2. **Validation**: catch typos early (`HTTP_PORT=eighty`).
3. **Report every problem at once**, not one per restart. (`errors.Join`, Go 1.20+, bundles several errors into one.)
4. **Testable**: separate "how do I look up a variable?" from "how do I interpret the values?" by taking a `lookup` function.

```go
// file: config/config.go
package config

import (
	"errors"
	"fmt"
	"io/fs"
	"net/url"
	"os"
	"strconv"
	"strings"

	"github.com/joho/godotenv"
)

// Config holds every setting the application needs. It is loaded once at start-up and then read-only.
type Config struct {
	Env            string   // "development", "test" or "production"
	Port           string   // TCP port to listen on, e.g. "8080"
	AllowedOrigins []string // browser origins allowed by CORS
	LogFormat      string   // "text" or "json"
}

// Load reads configuration from the environment (and an optional .env file).
// Real environment variables take precedence over values in .env.
func Load() (*Config, error) {
	if err := godotenv.Load(); err != nil && !errors.Is(err, fs.ErrNotExist) {
		return nil, fmt.Errorf("reading .env: %w", err)
	}
	return fromLookup(os.LookupEnv)
}

// fromLookup builds a Config using lookup to read variables. Tests can pass a fake lookup.
func fromLookup(lookup func(string) (string, bool)) (*Config, error) {
	// get returns the variable's value, or def if it is unset or empty.
	get := func(key, def string) string {
		if v, ok := lookup(key); ok && strings.TrimSpace(v) != "" {
			return strings.TrimSpace(v)
		}
		return def
	}

	cfg := &Config{
		Env:            get("APP_ENV", "development"),
		Port:           get("HTTP_PORT", "8080"),
		AllowedOrigins: splitCSV(get("ALLOWED_ORIGINS", "http://localhost:5173")),
		LogFormat:      get("LOG_FORMAT", "text"),
	}

	var errs []error

	if n, err := strconv.Atoi(cfg.Port); err != nil || n < 1 || n > 65535 {
		errs = append(errs, fmt.Errorf("HTTP_PORT must be a number between 1 and 65535, got %q", cfg.Port))
	}

	switch cfg.Env {
	case "development", "test", "production":
	default:
		errs = append(errs, fmt.Errorf("APP_ENV must be development, test or production, got %q", cfg.Env))
	}

	switch cfg.LogFormat {
	case "text", "json":
	default:
		errs = append(errs, fmt.Errorf("LOG_FORMAT must be text or json, got %q", cfg.LogFormat))
	}

	for _, origin := range cfg.AllowedOrigins {
		if err := validateOrigin(origin); err != nil {
			errs = append(errs, err)
		}
	}

	if err := errors.Join(errs...); err != nil { // nil when errs is empty
		return nil, err
	}
	return cfg, nil
}

// splitCSV splits "a, b,c" into ["a" "b" "c"], dropping empty items.
func splitCSV(s string) []string {
	var out []string
	for _, part := range strings.Split(s, ",") {
		if part = strings.TrimSpace(part); part != "" {
			out = append(out, part)
		}
	}
	return out
}

// validateOrigin checks that an origin looks like scheme://host[:port] with no path.
func validateOrigin(origin string) error {
	u, err := url.Parse(origin)
	if err != nil || (u.Scheme != "http" && u.Scheme != "https") || u.Host == "" || (u.Path != "" && u.Path != "/") {
		return fmt.Errorf("ALLOWED_ORIGINS entry %q must look like https://host[:port] (no path)", origin)
	}
	return nil
}
```

Points worth noticing:

- The `get` closure (Chapter 20) captures `lookup` and implements the "unset **or empty** → default" rule in one place.
- We only expose **one** function that can fail (`Load`) and one that can't (`Config` fields are plain data).
- `errors.Join(errs...)` returns `nil` for an empty list, so `if err := errors.Join(...); err != nil` is the whole "did anything fail?" check. When there are failures, the resulting error message lists each on its own line.
- Validation rejects an origin with a **path** (`https://shop.example.com/app`): browsers send origins as `scheme://host:port`, so a path can never match: a silent misconfiguration this check turns into a loud one.
- Adding a setting later means: a field, a default, a validation: three obvious places.

---

## 8. The new project layout

As the app grows, *where things go* matters. We adopt a widely used Go layout:

```
ecommerce/
├── main.go                  ← entry point: load config → set up logger → run the server
├── go.mod / go.sum
├── .env.example             ← documented variables (committed)
├── .env                     ← your local values (NOT committed)
├── .gitignore
├── cmd/
│   └── serve.go             ← starts/stops the HTTP server (timeouts, graceful shutdown)
├── config/
│   └── config.go            ← reads and validates configuration
├── rest/
│   └── router.go            ← builds the HTTP handler: routes + middleware pipeline
├── handlers/                ← (as before) request handlers
├── middleware/              ← (as before) CORS, logger, recover, ...
├── database/, models/, util/   ← (as before)
```

Guidelines:

- **`main.go` stays tiny**: read config, build the logger, call `cmd.Serve`, exit with a code on failure.
- **`cmd/`**: things about *running* the program (serve, and later: `migrate`, `seed`). Big projects put one folder per executable under `cmd/`; ours is a single-binary project, so `cmd/serve.go` is a package named `cmd`.
- **`rest/`**: the HTTP layer's assembly: which routes exist, and which middleware wrap them. (It used to be `newRouter` inside `main`; moving it into a package lets both `cmd` and tests import it.)
- **`config/`**: configuration only; no other package reads environment variables directly. One place to look, one place to validate.

---

## 9. `http.Server`: timeouts and graceful shutdown

Until now we used `http.ListenAndServe(":8080", handler)`, a convenience with **no timeouts**. That's dangerous in production:

- A client that opens a connection and sends **one byte per minute** (a "Slowloris" attack) ties up your resources indefinitely.
- A stalled connection can keep a goroutine and file descriptor forever.

We use an explicit **`http.Server`**:

```go
srv := &http.Server{
	Addr:              ":" + cfg.Port,
	Handler:           handler,
	ReadHeaderTimeout: 5 * time.Second,   // time allowed to read the request headers
	ReadTimeout:       10 * time.Second,  // time to read the entire request (headers + body)
	WriteTimeout:      30 * time.Second,  // time to write the response
	IdleTimeout:       60 * time.Second,  // how long an idle keep-alive connection may stay open
}
```

| Field | Protects against |
|-------|------------------|
| `ReadHeaderTimeout` | Slow-header attacks (always set this one) |
| `ReadTimeout` | Slow request bodies |
| `WriteTimeout` | Slow-reading clients; runaway handlers (measured from end of header read) |
| `IdleTimeout` | Piles of idle keep-alive connections |

Choose values that fit your slowest *legitimate* request (file uploads need bigger `ReadTimeout`).

### Graceful shutdown

When you deploy a new version, the platform sends your process **`SIGTERM`** ("please stop"); pressing `Ctrl+C` sends **`SIGINT`**. If the program just dies, **requests in flight are cut off** mid-response: users see errors on every deploy.

**Graceful shutdown** means: *stop accepting new connections, let running requests finish (up to a limit), then exit.* Go supports it directly with `srv.Shutdown(ctx)`.

The recipe:

1. Run `srv.ListenAndServe()` in a **goroutine** (Chapter 36) so the main flow can wait for a signal.
2. `signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)` gives a context that's **cancelled when a signal arrives**.
3. On cancellation, call `srv.Shutdown(shutdownCtx)` with a **deadline** (say 10 s): it closes the listeners, waits for active requests, and returns.
4. Note: `ListenAndServe` returns `http.ErrServerClosed` after `Shutdown`: that's the *normal* outcome, not an error.

```
        SIGTERM
           │
           ▼
   ctx cancelled ──► srv.Shutdown(10s deadline)
                         │  stop accepting new connections
                         │  wait for in-flight requests to finish
                         ▼
                    ListenAndServe returns ErrServerClosed ──► exit 0
```

We'll implement this so it's **testable**: a function `run(ctx, srv, listener, timeout)` that takes a context (tests cancel it instead of sending real signals) and an already-open listener (tests use port `0` = "any free port").

---

## 10. The complete code

<!-- delete: main_test.go -->
<!-- delete: pipeline_test.go -->

The router assembly moves from `main.go` into `rest/router.go`, and CORS now takes its origins from configuration.

```go
// file: middleware/cors.go
package middleware

import (
	"net/http"
	"strings"
)

const (
	allowedMethods = "GET, POST, PUT, PATCH, DELETE, OPTIONS"
	allowedHeaders = "Content-Type, Authorization"
)

// CORS returns a middleware that allows browser calls from the given origins
// and answers preflight requests itself.
func CORS(allowedOrigins []string) Middleware {
	allowed := make(map[string]bool, len(allowedOrigins))
	for _, o := range allowedOrigins {
		allowed[strings.TrimRight(o, "/")] = true
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			origin := r.Header.Get("Origin")

			if origin != "" && allowed[origin] {
				h := w.Header()
				h.Set("Access-Control-Allow-Origin", origin)
				h.Add("Vary", "Origin")

				isPreflight := r.Method == http.MethodOptions && r.Header.Get("Access-Control-Request-Method") != ""
				if isPreflight {
					h.Set("Access-Control-Allow-Methods", allowedMethods)
					h.Set("Access-Control-Allow-Headers", allowedHeaders)
					h.Set("Access-Control-Max-Age", "600")
					w.WriteHeader(http.StatusNoContent)
					return
				}
			}

			next.ServeHTTP(w, r)
		})
	}
}
```

Notice `CORS` is now a **middleware factory** (like `Logger`): configuration in, middleware out. Its allow-list is built **once** (into a map) when the middleware is created, not on every request.

```go
// file: rest/router.go
package rest

import (
	"ecommerce/config"
	"ecommerce/handlers"
	"ecommerce/middleware"
	"log/slog"
	"net/http"
)

// NewRouter builds the complete HTTP handler for the app: routes wrapped in the middleware pipeline.
func NewRouter(logger *slog.Logger, cfg *config.Config) http.Handler {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /products", handlers.GetProducts)
	mux.HandleFunc("POST /products", handlers.CreateProduct)
	mux.HandleFunc("GET /products/{id}", handlers.GetProduct)

	global := middleware.NewStack(
		middleware.RequestID,
		middleware.Logger(logger),
		middleware.CORS(cfg.AllowedOrigins),
		middleware.Recover(logger),
		middleware.JSONErrors,
	)
	return global.Then(mux)
}
```

```go
// file: cmd/serve.go
package cmd

import (
	"context"
	"ecommerce/config"
	"ecommerce/rest"
	"errors"
	"fmt"
	"log/slog"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

const shutdownTimeout = 10 * time.Second

// Serve starts the HTTP server and blocks until it stops.
// It shuts down gracefully when the process receives SIGINT (Ctrl+C) or SIGTERM.
func Serve(cfg *config.Config, logger *slog.Logger) error {
	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           rest.NewRouter(logger, cfg),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      30 * time.Second,
		IdleTimeout:       60 * time.Second,
		ErrorLog:          slog.NewLogLogger(logger.Handler(), slog.LevelError),
	}

	ln, err := net.Listen("tcp", srv.Addr)
	if err != nil {
		return fmt.Errorf("cannot listen on %s: %w", srv.Addr, err) // e.g. "address already in use"
	}

	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	logger.Info("server starting", "addr", srv.Addr, "env", cfg.Env)
	return run(ctx, srv, ln, shutdownTimeout, logger)
}

// run serves on ln until ctx is cancelled, then shuts the server down gracefully.
func run(ctx context.Context, srv *http.Server, ln net.Listener, timeout time.Duration, logger *slog.Logger) error {
	serveErr := make(chan error, 1)
	go func() { serveErr <- srv.Serve(ln) }()

	select {
	case err := <-serveErr: // the server stopped on its own (e.g., a fatal error)
		if errors.Is(err, http.ErrServerClosed) {
			return nil
		}
		return err

	case <-ctx.Done(): // a shutdown signal arrived
		logger.Info("shutting down", "timeout", timeout)

		shutdownCtx, cancel := context.WithTimeout(context.Background(), timeout)
		defer cancel()

		if err := srv.Shutdown(shutdownCtx); err != nil {
			return fmt.Errorf("graceful shutdown failed: %w", err)
		}
		<-serveErr // wait for Serve to return (it yields http.ErrServerClosed)
		logger.Info("server stopped cleanly")
		return nil
	}
}
```

```go
// file: main.go
package main

import (
	"ecommerce/cmd"
	"ecommerce/config"
	"fmt"
	"log/slog"
	"os"
)

func main() {
	cfg, err := config.Load()
	if err != nil {
		fmt.Fprintln(os.Stderr, "configuration error:\n"+err.Error())
		os.Exit(1) // non-zero exit code: a supervisor/CI sees that startup failed
	}

	logger := newLogger(cfg)

	if err := cmd.Serve(cfg, logger); err != nil {
		logger.Error("server failed", "err", err)
		os.Exit(1)
	}
}

// newLogger builds a logger in the format the configuration asks for.
func newLogger(cfg *config.Config) *slog.Logger {
	if cfg.LogFormat == "json" {
		return slog.New(slog.NewJSONHandler(os.Stdout, nil))
	}
	return slog.New(slog.NewTextHandler(os.Stdout, nil))
}
```

Read `main` as the whole story of the program: *load config (fail fast) → build logger → serve → exit code.*

Also update the existing logger tests? No: only the code that called `middleware.CORS` as a value changed (`rest/router.go`). The middleware tests from Chapter 46 don't use CORS and keep passing.

---

## 11. Tests

### Config tests

Because `fromLookup` takes a function, tests supply a fake environment: no real env vars touched.

```go
// file: config/config_test.go
package config

import (
	"reflect"
	"strings"
	"testing"
)

func lookupFrom(env map[string]string) func(string) (string, bool) {
	return func(key string) (string, bool) {
		v, ok := env[key]
		return v, ok
	}
}

func TestDefaults(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(nil))
	if err != nil {
		t.Fatal(err)
	}
	want := &Config{
		Env:            "development",
		Port:           "8080",
		AllowedOrigins: []string{"http://localhost:5173"},
		LogFormat:      "text",
	}
	if !reflect.DeepEqual(cfg, want) {
		t.Errorf("got %+v, want %+v", cfg, want)
	}
}

func TestOverrides(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(map[string]string{
		"APP_ENV":         "production",
		"HTTP_PORT":       "9090",
		"ALLOWED_ORIGINS": "https://shop.example.com, https://admin.example.com/",
		"LOG_FORMAT":      "json",
	}))
	if err != nil {
		t.Fatal(err)
	}
	if cfg.Env != "production" || cfg.Port != "9090" || cfg.LogFormat != "json" {
		t.Errorf("unexpected config %+v", cfg)
	}
	if len(cfg.AllowedOrigins) != 2 {
		t.Errorf("expected 2 origins, got %v", cfg.AllowedOrigins)
	}
}

func TestEmptyValuesFallBackToDefaults(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(map[string]string{"HTTP_PORT": "   ", "LOG_FORMAT": ""}))
	if err != nil {
		t.Fatal(err)
	}
	if cfg.Port != "8080" || cfg.LogFormat != "text" {
		t.Errorf("blank values must use the defaults, got %+v", cfg)
	}
}

func TestInvalidValues(t *testing.T) {
	tests := []struct {
		name string
		env  map[string]string
		want string
	}{
		{"port is not a number", map[string]string{"HTTP_PORT": "eighty"}, "HTTP_PORT"},
		{"port too large", map[string]string{"HTTP_PORT": "70000"}, "HTTP_PORT"},
		{"port zero", map[string]string{"HTTP_PORT": "0"}, "HTTP_PORT"},
		{"unknown environment", map[string]string{"APP_ENV": "staging"}, "APP_ENV"},
		{"unknown log format", map[string]string{"LOG_FORMAT": "xml"}, "LOG_FORMAT"},
		{"origin without scheme", map[string]string{"ALLOWED_ORIGINS": "localhost:5173"}, "ALLOWED_ORIGINS"},
		{"origin with a path", map[string]string{"ALLOWED_ORIGINS": "https://shop.example.com/app"}, "ALLOWED_ORIGINS"},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			cfg, err := fromLookup(lookupFrom(tc.env))
			if err == nil {
				t.Fatalf("expected an error, got config %+v", cfg)
			}
			if !strings.Contains(err.Error(), tc.want) {
				t.Errorf("error %q should mention %q", err, tc.want)
			}
		})
	}
}

func TestReportsAllProblemsAtOnce(t *testing.T) {
	_, err := fromLookup(lookupFrom(map[string]string{
		"HTTP_PORT":  "abc",
		"APP_ENV":    "moon",
		"LOG_FORMAT": "yaml",
	}))
	if err == nil {
		t.Fatal("expected errors")
	}
	for _, key := range []string{"HTTP_PORT", "APP_ENV", "LOG_FORMAT"} {
		if !strings.Contains(err.Error(), key) {
			t.Errorf("combined error should mention %s: %q", key, err)
		}
	}
}
```

### Router tests (moved from `main`)

The whole-app tests now live next to the code they exercise, in package `rest`, and build their own `Config`, so no environment variables are involved:

```go
// file: rest/router_test.go
package rest

import (
	"ecommerce/config"
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

var testConfig = &config.Config{
	Env:            "test",
	Port:           "0",
	AllowedOrigins: []string{"http://localhost:5173"},
	LogFormat:      "text",
}

var quietLogger = slog.New(slog.NewTextHandler(io.Discard, nil))

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
	NewRouter(quietLogger, testConfig).ServeHTTP(rec, req)
	return rec
}

func TestListAndGet(t *testing.T) {
	database.ResetProducts()

	var list []product
	rec := do(http.MethodGet, "/products", "", nil)
	if rec.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rec.Code)
	}
	json.NewDecoder(rec.Body).Decode(&list)
	if len(list) != 3 {
		t.Fatalf("expected 3 products, got %d", len(list))
	}

	for path, want := range map[string]int{"/products/2": 200, "/products/99": 404, "/products/abc": 400} {
		if got := do(http.MethodGet, path, "", nil).Code; got != want {
			t.Errorf("GET %s: expected %d, got %d", path, want, got)
		}
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
	if rec := do(http.MethodPost, "/products", `{"title":"","price":5}`, nil); rec.Code != http.StatusUnprocessableEntity {
		t.Errorf("expected 422, got %d", rec.Code)
	}
}

func TestPipelineErrorsAreJSONWithCORSAndRequestID(t *testing.T) {
	origin := map[string]string{"Origin": "http://localhost:5173"}

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

	rec = do(http.MethodDelete, "/products", "", nil)
	if rec.Code != 405 || rec.Header().Get("Allow") != "GET, HEAD, POST" {
		t.Errorf("405: code=%d allow=%q", rec.Code, rec.Header().Get("Allow"))
	}
}

func TestCORSUsesConfiguredOrigins(t *testing.T) {
	preflight := func(origin string) *httptest.ResponseRecorder {
		return do(http.MethodOptions, "/products", "", map[string]string{
			"Origin":                        origin,
			"Access-Control-Request-Method": "POST",
		})
	}

	if rec := preflight("http://localhost:5173"); rec.Code != http.StatusNoContent ||
		rec.Header().Get("Access-Control-Allow-Origin") != "http://localhost:5173" {
		t.Errorf("configured origin rejected: code=%d", rec.Code)
	}
	if rec := preflight("https://evil.example"); rec.Header().Get("Access-Control-Allow-Origin") != "" {
		t.Error("unconfigured origin must not be allowed")
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

### Graceful shutdown test

We prove the important property, that **an in-flight request finishes even after shutdown begins**, without sending real signals:

```go
// file: cmd/serve_test.go
package cmd

import (
	"context"
	"io"
	"log/slog"
	"net"
	"net/http"
	"testing"
	"time"
)

func TestGracefulShutdownLetsInFlightRequestsFinish(t *testing.T) {
	started := make(chan struct{})
	mux := http.NewServeMux()
	mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
		close(started)
		time.Sleep(300 * time.Millisecond) // a request that takes a while
		w.Write([]byte("finished"))
	})

	srv := &http.Server{Handler: mux}
	ln, err := net.Listen("tcp", "127.0.0.1:0") // port 0: the OS picks a free port
	if err != nil {
		t.Fatal(err)
	}
	logger := slog.New(slog.NewTextHandler(io.Discard, nil))

	ctx, cancel := context.WithCancel(context.Background())
	runDone := make(chan error, 1)
	go func() { runDone <- run(ctx, srv, ln, 5*time.Second, logger) }()

	// start a slow request...
	type result struct {
		body string
		err  error
	}
	got := make(chan result, 1)
	go func() {
		resp, err := http.Get("http://" + ln.Addr().String() + "/slow")
		if err != nil {
			got <- result{err: err}
			return
		}
		defer resp.Body.Close()
		b, _ := io.ReadAll(resp.Body)
		got <- result{body: string(b)}
	}()

	<-started // the handler is running
	cancel()  // ...and ask the server to shut down while it is in flight

	res := <-got
	if res.err != nil || res.body != "finished" {
		t.Fatalf("in-flight request was cut off: body=%q err=%v", res.body, res.err)
	}
	if err := <-runDone; err != nil {
		t.Fatalf("run returned %v, want nil", err)
	}

	// after shutdown the port no longer accepts connections
	if _, err := http.Get("http://" + ln.Addr().String() + "/slow"); err == nil {
		t.Error("server still accepting connections after shutdown")
	}
}
```

Run everything:

```bash
go vet ./... && go test -race ./...
```

```
?   	ecommerce	[no test files]
ok  	ecommerce/cmd	0.3s
ok  	ecommerce/config	1.0s
?   	ecommerce/database	[no test files]
?   	ecommerce/handlers	[no test files]
ok  	ecommerce/middleware	1.0s
?   	ecommerce/models	[no test files]
ok  	ecommerce/rest	1.0s
?   	ecommerce/util	[no test files]
```

---

## 12. Running it

```bash
cp .env.example .env
go run .
```

```
time=... level=INFO msg="server starting" addr=:8080 env=development
```

**Environment variables override `.env`** (real precedence in action):

```bash
HTTP_PORT=9090 LOG_FORMAT=json go run .
```

```
{"time":"...","level":"INFO","msg":"server starting","addr":":9090","env":"development"}
```

**Invalid configuration fails fast with every problem listed** (real output):

```bash
HTTP_PORT=eighty APP_ENV=moon go run .
```

```
configuration error:
HTTP_PORT must be a number between 1 and 65535, got "eighty"
APP_ENV must be development, test or production, got "moon"
exit status 1
```

**Port already in use** (like Chapter 38's troubleshooting) gives a clear error and non-zero exit:

```
time=... level=ERROR msg="server failed" err="cannot listen on :8080: listen tcp :8080: bind: address already in use"
```

**Graceful shutdown:** press `Ctrl+C` (or `kill -TERM <pid>`) while running:

```
time=... level=INFO msg="shutting down" timeout=10s
time=... level=INFO msg="server stopped cleanly"
```

---

## 13. Exit codes and failing fast

Every process ends with an **exit code**, an integer the parent process (shell, Docker, Kubernetes, CI) reads:

| Code | Convention |
|------|------------|
| `0` | Success |
| non-zero (usually `1`) | Failure |

```bash
go run . ; echo "exit code: $?"      # $? is the last command's exit code
```

Go returns `0` when `main` returns normally, and `1` for an unrecovered panic. Use **`os.Exit(1)`** when you detect a fatal startup problem, so orchestration knows the launch **failed** and can alert, restart, or roll back a deploy.

**Fail fast:** validate configuration at start-up and **refuse to run** when it's wrong. A server that starts with a broken config and fails an hour later at 3 a.m. is much worse than one that refuses to start immediately.

**`os.Exit` skips deferred functions** (Chapter 34). That's why `main` does its real work in `cmd.Serve` (which can `defer stop()` and return an error) and only calls `os.Exit` at the very end.

---

## 14. Security and workflow notes

- **Never log secrets.** When you later add a JWT secret or database URL, do not print the whole `Config`. A `String()` method that redacts sensitive fields (Exercise 5) prevents accidents.
- **Defaults should be safe.** Defaults are for *development convenience* (localhost origins, text logs). Anything security-sensitive (JWT secret, DB password) should have **no default** and be **required** (Chapter 48 does exactly this).
- **`.env` is for development.** In production, set real environment variables through your platform's secret manager (Docker/Compose `environment:`, Kubernetes Secrets, cloud parameter stores).
- **Don't read env vars all over the code.** Only `config` calls `os.LookupEnv`; everything else receives a `*Config`.
- **Document every variable** in `.env.example` (and README).
- **Config is read-only after start-up.** Changing it while goroutines read it would be a data race (Chapter 67).

---

## 15. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Committing `.env` | Secrets leaked in Git history | `.gitignore` it; commit `.env.example`; rotate leaked secrets |
| 2 | Forgetting that env vars are **strings** | Type errors, `"false"` treated as truthy | Convert with `strconv`, validate |
| 3 | Ignoring `strconv` errors | `Port = 0` silently | Check every error |
| 4 | Using `os.Getenv` when "empty vs. unset" matters | Can't distinguish | `os.LookupEnv` |
| 5 | Treating `godotenv.Load` errors as fatal when `.env` is missing | Production won't start | Ignore `fs.ErrNotExist` only |
| 6 | Expecting `.env` to override real environment variables | Confusing precedence | It doesn't (by design); real env wins |
| 7 | `http.ListenAndServe` with no timeouts in production | Slowloris; leaked connections | Use `http.Server` with timeouts |
| 8 | `os.Exit` / `log.Fatal` inside library code or with pending defers | Skips cleanup | Return errors; exit only in `main` |
| 9 | Not draining connections on shutdown | Deploys cut requests off | `srv.Shutdown(ctx)` |
| 10 | Treating `http.ErrServerClosed` as a failure | False error on every clean stop | `errors.Is(err, http.ErrServerClosed)` |
| 11 | Global config mutated at run time | Data races; test interference | Load once, pass a read-only `*Config` |
| 12 | Trailing slash or path in an allowed origin | CORS never matches | Origins are `scheme://host[:port]` only |
| 13 | `string(65)` to build text from a number | `"A"` instead of `"65"` | `strconv.Itoa` |
| 14 | Hard-coding secrets "just for now" | They stay | Required env vars with no default |

---

## 16. Exercises

### Exercise 1: Add a setting
Add `READ_TIMEOUT` (a duration such as `10s`) to the config with default `10s`, validate that it parses and is positive, and use it in `http.Server`.

<details><summary>Solution</summary>

Add `ReadTimeout time.Duration` to `Config`. In `fromLookup`:
```go
rt, err := time.ParseDuration(get("READ_TIMEOUT", "10s"))
if err != nil || rt <= 0 {
	errs = append(errs, fmt.Errorf("READ_TIMEOUT must be a positive duration like 10s, got %q", get("READ_TIMEOUT", "10s")))
}
cfg.ReadTimeout = rt
```
and in `Serve`: `ReadTimeout: cfg.ReadTimeout`. Add a test with `READ_TIMEOUT=banana`.
</details>

### Exercise 2: Boolean flags
Add `DEBUG_ROUTES` (bool, default false) using `strconv.ParseBool`. What do `"1"`, `"true"`, `"yes"`, and `""` produce?

<details><summary>Solution</summary>

`ParseBool` accepts `1, t, T, TRUE, true, True, 0, f, F, FALSE, false, False`. So `"1"` and `"true"` → `true`; `"yes"` → **error** (not accepted), so validate and report it; `""` (unset/empty) → use the default `false` via the `get` helper before parsing.
</details>

### Exercise 3: Required secret
Add `JWT_SECRET` as **required** (no default) with a minimum length of 32 characters, and make `Load` fail if it's missing. (You'll use it in Chapter 48.) How do you keep existing tests passing?

<details><summary>Solution</summary>

```go
secret, ok := lookup("JWT_SECRET")
if !ok || len(strings.TrimSpace(secret)) < 32 {
	errs = append(errs, errors.New("JWT_SECRET is required and must be at least 32 characters"))
}
```
Update the config tests' fake environments (add a 32+ character `JWT_SECRET` to the "valid" cases, and add one test for the missing/short case). Tests for the router build their own `Config` literal and don't call `Load`.
</details>

### Exercise 4: Precedence experiment
Create `.env` with `HTTP_PORT=8081`, then run `HTTP_PORT=8082 go run .`. Which port is used? Then run `env -u HTTP_PORT go run .`. Explain.

<details><summary>Solution</summary>

`8082`: real environment variables beat `.env` (`godotenv.Load` doesn't override). With the variable unset, `.env`'s `8081` is used. If neither exists, the code default `8080`.
</details>

### Exercise 5: Redacting `String()`
Give `Config` a `String()` method that prints all fields but replaces `JWTSecret` with `***`. Why is this worth doing?

<details><summary>Solution</summary>

```go
func (c *Config) String() string {
	return fmt.Sprintf("Config{Env:%s Port:%s Origins:%v LogFormat:%s JWTSecret:***}",
		c.Env, c.Port, c.AllowedOrigins, c.LogFormat)
}
```
`fmt`/`slog` call `String()` when printing, so an accidental `logger.Info("config", "cfg", cfg)` can't leak the secret.
</details>

### Exercise 6: Shutdown timeout
A handler sleeps 15 s, but `shutdownTimeout` is 10 s. What does `Shutdown` return, and what happens to the request? How would you decide the right timeout?

<details><summary>Solution</summary>

`Shutdown` returns `context deadline exceeded` after 10 s; the slow request is still running (connections aren't force-closed by `Shutdown`), so the process exits and it's cut off (`run` returns "graceful shutdown failed"). Choose the timeout as slightly longer than your longest legitimate request, but shorter than your platform's kill grace period (Kubernetes defaults to 30 s before `SIGKILL`); you can call `srv.Close()` after a failed `Shutdown` to force-close.
</details>

### Exercise 7 (challenge): Health and readiness
Add `GET /healthz` (always `200 {"status":"ok"}`) for load balancers. During shutdown, ideally `/healthz` starts failing *before* the server stops accepting connections so the balancer drains traffic. Sketch how you'd do that with an `atomic.Bool`.

<details><summary>Solution</summary>

Keep a package-level (or injected) `var shuttingDown atomic.Bool`. The handler returns `503` when `shuttingDown.Load()` is true. In `run`, on `ctx.Done()` first call `shuttingDown.Store(true)`, sleep briefly (e.g., a few seconds) so the load balancer's next health check notices and stops routing new traffic, *then* call `srv.Shutdown`. `atomic.Bool` avoids a data race between the handler goroutines and the shutdown goroutine (Chapter 67).
</details>

---

## 17. Quiz

1. Why shouldn't configuration live in source code?
2. What's the difference between `os.Getenv` and `os.LookupEnv`?
3. In what order do real env vars, `.env`, and code defaults take precedence?
4. Why should `.env` be in `.gitignore`?
5. What does `errors.Join` return for an empty list?
6. Which timeout protects against slow-header attacks?
7. What does `http.ErrServerClosed` mean after `Shutdown`?
8. Why does `main` call `os.Exit(1)` for bad configuration?

<details><summary>Answers</summary>

1. It varies per environment and may be secret; code should be identical everywhere.
2. `LookupEnv` also reports whether the variable was set at all; `Getenv` returns `""` for both unset and empty.
3. Real environment variables > `.env` file > code defaults.
4. It contains local/secret values that must never enter version control.
5. `nil`.
6. `ReadHeaderTimeout`.
7. It's the expected result of a graceful shutdown, not a failure.
8. So the supervisor/CI sees a non-zero exit code and knows startup failed.
</details>

---

## 18. Summary

- **Configuration lives outside the code** (twelve-factor): environment variables for production, a `.env` file for development (never committed; commit `.env.example`).
- Precedence: **real env vars > `.env` > defaults**. `godotenv.Load` doesn't override existing variables; ignore only "file not found".
- Env vars are **strings**: convert and **validate** with `strconv` (`Atoi`, `ParseBool`, `Itoa`, …) and `time.ParseDuration`; collect **all** problems with `errors.Join`; **fail fast** with `os.Exit(1)`.
- Load a **`*Config` once** and **pass it** to what needs it: explicit dependencies, easy tests, no global mutation. Only the `config` package touches `os`.
- Layout: `main.go` (tiny) → `cmd/` (running the program) → `rest/` (routes + middleware) + `config/`.
- Use an explicit **`http.Server` with timeouts** (`ReadHeaderTimeout` at minimum) and **graceful shutdown** via `signal.NotifyContext` + `srv.Shutdown`, testable by cancelling a context.
- `CORS` became a **middleware factory** taking its allow-list from config.

### ➡️ What's next?

**Authentication.** [Chapter 48](48-authentication-with-jwt.md) adds users, password hashing, login, and **JWT tokens**, using the configuration system you just built for the signing secret.
