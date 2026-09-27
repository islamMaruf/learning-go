# Chapter 57: Database Configuration — DSNs, TLS, Timeouts, Precedence, and Docker Compose

> **Goal of this chapter:** Make the database connection **safe and easy to configure in every environment**: laptop, CI, staging, production. You'll learn to build a connection string from separate settings (with correct escaping), refuse insecure TLS settings in production, add **connect** and **statement timeouts**, understand **environment-variable precedence** (and debug the classic "my `.env` is being ignored" problem), and run PostgreSQL with **Docker Compose**, including automatic schema loading and a health check.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 5 hours  **Prerequisite:** [Chapters 48 and 52](52-connecting-to-postgresql.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The problem: one app, many environments](#2-the-problem-one-app-many-environments)
3. [`DATABASE_URL` or separate variables?](#3-database_url-or-separate-variables)
4. [Building a DSN safely](#4-building-a-dsn-safely)
5. [TLS: `sslmode` explained](#5-tls-sslmode-explained)
6. [Timeouts: connect and statement](#6-timeouts-connect-and-statement)
7. [The implementation](#7-the-implementation)
8. [Tests](#8-tests)
9. [Environment variable precedence (and "my `.env` is ignored")](#9-environment-variable-precedence)
10. [Docker Compose for local PostgreSQL](#10-docker-compose-for-local-postgresql)
11. [Secrets management](#11-secrets-management)
12. [Common mistakes](#12-common-mistakes)
13. [Interview questions](#13-interview-questions)
14. [Exercises](#14-exercises)
15. [Quiz](#15-quiz)
16. [Summary](#16-summary)

---

## 1. What you will learn

- The **twelve-factor** rule: configuration lives in the environment, not in code
- Two configuration styles (one URL vs. separate parts) and how to support both
- Why you must **URL-encode** passwords, and how `net/url` does it for you
- The six PostgreSQL **`sslmode`** values, what each protects against, and which are acceptable in production
- **Connect timeouts** and **statement timeouts**: how a stuck query is cut off by the database itself
- How `godotenv` and the real environment interact, and how to **debug precedence**
- Running Postgres with **Docker Compose**: named volumes, health checks, init scripts
- Where secrets should (and should not) live

---

## 2. The problem: one app, many environments

The same binary runs in very different places:

| Environment | Database | TLS | Password comes from |
|-------------|----------|-----|---------------------|
| Your laptop | Docker container on `localhost` | not needed | `.env` file |
| CI | throwaway container | not needed | CI variable |
| Staging | managed cloud database | required | secret manager |
| Production | managed cloud database | **required + verified** | secret manager |

The **twelve-factor app** methodology (a widely adopted checklist for deployable apps) says: *store config in the environment*. Code is identical everywhere; only environment variables differ. That gives us three requirements for the configuration code:

1. **No hard-coded values.** (An early version of many apps has `"postgres://postgres:password@localhost/db"` in a `.go` file. Everyone who reads the repository, forever, now knows that password, and production can't use it.)
2. **Fail fast and clearly** when something is missing or unsafe, at startup, with all problems listed (Chapter 48).
3. **Safe defaults**: convenient for development, but the *dangerous* settings must be impossible to forget in production.

---

## 3. `DATABASE_URL` or separate variables?

Two conventions exist, and real platforms use both:

```
# One URL (Heroku, Render, Railway, Fly.io, most managed services hand you this)
DATABASE_URL=postgres://app:s3cret@db.internal:5432/ecommerce?sslmode=verify-full

# Separate parts (Docker Compose files, Kubernetes secrets, people who dislike long strings)
DB_HOST=db.internal
DB_PORT=5432
DB_USER=app
DB_PASSWORD=s3cret
DB_NAME=ecommerce
DB_SSLMODE=verify-full
```

| Style | Pros | Cons |
|-------|------|------|
| **URL** | one value to copy/paste; what platforms provide | password must be percent-encoded by hand; harder to override one part (say, only the host) |
| **Parts** | each value is readable, and the app does the escaping | more variables to keep consistent |

We support **both** with a simple precedence rule: **if `DATABASE_URL` is set it wins; otherwise the parts are required** and assembled into a URL. One code path downstream (`db.Connect` always receives a URL).

---

## 4. Building a DSN safely

The tempting way to assemble a URL is `fmt.Sprintf`:

```go
// ❌ Fragile: breaks when the password contains @ / : ? # %
dsn := fmt.Sprintf("postgres://%s:%s@%s:%s/%s", user, pass, host, port, name)
```

A password such as `p@ss/w:rd#1` contains characters that have *meaning* in URLs. Real output of what happens when you paste it in unescaped, and what `net/url` does instead:

```go
package main

import (
	"fmt"
	"net"
	"net/url"

	"github.com/jackc/pgx/v5"
)

func main() {
	pass := "p@ss/w:rd#1"

	naive := fmt.Sprintf("postgres://app:%s@db.internal:5432/shop", pass)
	cfg, err := pgx.ParseConfig(naive)
	fmt.Println("naive  :", naive)
	fmt.Printf("         -> err=%v | user=%q password=%q host=%q\n", err, cfg.User, cfg.Password, cfg.Host)

	u := url.URL{
		Scheme:   "postgres",
		User:     url.UserPassword("app", pass),
		Host:     net.JoinHostPort("db.internal", "5432"),
		Path:     "/shop",
		RawQuery: "sslmode=require",
	}
	cfg, err = pgx.ParseConfig(u.String())
	fmt.Println("escaped:", u.String())
	fmt.Printf("         -> err=%v | user=%q password=%q host=%q\n", err, cfg.User, cfg.Password, cfg.Host)
}
```

Real output (run it yourself after the check below):

```
naive  : postgres://app:p@ss/w:rd#1@db.internal:5432/shop
         -> err=<nil> | user="app" password="p" host="ss"
escaped: postgres://app:p%40ss%2Fw%3Ard%231@db.internal:5432/shop?sslmode=require
         -> err=<nil> | user="app" password="p@ss/w:rd#1" host="db.internal"
```

The naive URL is silently misparsed (**no error at all**): the parser decides the password is just `p` and the *host* is `ss` (look at the output). The connection would then fail with a confusing "cannot resolve host" or, worse, an authentication error for the wrong password. With `url.UserPassword(...)`, each special character is **percent-encoded** (`@`→`%40`, `/`→`%2F`, `:`→`%3A`, `#`→`%23`) and the password round-trips exactly. Two more details:

- **`net.JoinHostPort`** adds brackets for IPv6 addresses (`[::1]:5432`), which manual `host + ":" + port` gets wrong.
- Even the *error messages* of `pgx` **redact the password** (`xxxxx`); good driver behavior, though never rely on it: our own code doesn't include URLs in errors.

Rule: **never build URLs with string formatting; use `net/url`.**

---

## 5. TLS: `sslmode` explained

By default a database connection is **plain text**: anyone on the network path can read queries, results, and (during authentication) sometimes password material. **TLS** (the successor to "SSL"; PostgreSQL still calls the setting `sslmode`) encrypts it. The modes are a ladder:

| `sslmode` | Encrypts? | Verifies the server's certificate? | Protects against | Use |
|-----------|-----------|------------------------------------|------------------|-----|
| `disable` | ❌ | ❌ | nothing | **local development only** |
| `allow` | only if the server insists | ❌ | almost nothing | avoid |
| `prefer` | tries TLS, **falls back to plain** | ❌ | passive eavesdropping *sometimes* | avoid: an attacker can force the fallback |
| `require` | ✅ | ❌ | eavesdropping | acceptable when you control the network |
| `verify-ca` | ✅ | ✅ signed by a trusted CA | eavesdropping + fake servers | good |
| `verify-full` | ✅ | ✅ CA **and** hostname matches | eavesdropping + man-in-the-middle | **best: use in production** |

Why "verify" matters: encryption alone means "nobody can *read* this", but if you can't tell **who** you're talking to, an attacker can sit in the middle, present their own certificate, and decrypt everything (a **man-in-the-middle attack**). Certificate verification proves the server is the one you meant.

Our rules, enforced in config:

- Development/test: anything goes (default `disable`), because the container on `localhost` has no TLS.
- **Production: `require`, `verify-ca` or `verify-full` only, and it must be *explicit*** (a URL without `sslmode` would silently use the driver default `prefer`, which we've just classified as unacceptable).

You can see the difference on your own machine. The stock Postgres container has TLS switched off, so asking for it fails; a real, verified error is *better* than silently sending data in clear text:

```
tls error: server refused TLS connection
```

(The full log line is shown in §9.)

---

## 6. Timeouts: connect and statement

Without timeouts, one bad moment (a network partition, a runaway query) can stall requests forever and pile up goroutines and connections until the server dies. Set them at three layers:

| Layer | Setting | Guards against | Where |
|-------|---------|----------------|-------|
| **Connect timeout** | how long to wait to *establish* a connection | black-holed/unreachable host | driver config (`ConnectTimeout`) |
| **Statement timeout** | how long the *database* lets one statement run before **cancelling it** | runaway queries, missing indexes on big tables | PostgreSQL runtime parameter `statement_timeout` |
| **Request context** | how long the *caller* is willing to wait | slow clients, slow everything | `context.WithTimeout` per request |

The statement timeout is a **safety net enforced by the database itself**: even if Go code forgets a context, PostgreSQL kills the query and reports SQLSTATE `57014` (`query_canceled`). It protects the database from a single bad query starving everyone else.

We apply both connect and statement timeouts in `db.Connect`, driven by configuration (`DB_STATEMENT_TIMEOUT`, default `10s`). Note the layering: statement timeout ≤ request timeout ≤ server `WriteTimeout` (30 s in Chapter 47): the innermost limit should fire first, so the database returns a clean error instead of the connection being torn down.

---

## 7. The implementation

### Configuration

New and changed settings (everything else from Chapter 52 stays):

| Variable | Default | Rule |
|----------|---------|------|
| `DATABASE_URL` | none | if set, used as is (validated) |
| `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` | none | **required** when `DATABASE_URL` is unset |
| `DB_PORT` | `5432` | 1–65535 |
| `DB_SSLMODE` | `disable` | one of the six modes |
| `DB_STATEMENT_TIMEOUT` | `10s` | positive duration |
| *(production)* | | `sslmode` must be `require`, `verify-ca`, or `verify-full` |

```go
// file: config/config.go
package config

import (
	"errors"
	"fmt"
	"io/fs"
	"net"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/joho/godotenv"
)

// Config holds every setting the application needs. It is loaded once at start-up and then read-only.
type Config struct {
	Env            string        // "development", "test" or "production"
	Port           string        // TCP port to listen on, e.g. "8080"
	AllowedOrigins []string      // browser origins allowed by CORS
	LogFormat      string        // "text" or "json"
	JWTSecret      string        // key used to sign tokens (keep secret!)
	JWTTTL         time.Duration // how long an issued token stays valid

	DatabaseURL        string        // postgres://user:password@host:port/dbname (contains a secret!)
	DBMaxOpenConns     int           // pool: most connections open at once
	DBMaxIdleConns     int           // pool: connections kept ready while idle
	DBConnMaxLifetime  time.Duration // pool: recycle connections older than this
	DBStatementTimeout time.Duration // the database cancels any statement running longer than this
}

// String redacts the secrets so a Config can be logged safely.
func (c *Config) String() string {
	return fmt.Sprintf("Config{Env:%s Port:%s Origins:%v LogFormat:%s JWTSecret:*** JWTTTL:%s DatabaseURL:*** DBMaxOpenConns:%d DBMaxIdleConns:%d DBConnMaxLifetime:%s DBStatementTimeout:%s}",
		c.Env, c.Port, c.AllowedOrigins, c.LogFormat, c.JWTTTL, c.DBMaxOpenConns, c.DBMaxIdleConns, c.DBConnMaxLifetime, c.DBStatementTimeout)
}

// DatabaseTarget describes where the database is (host:port/name) without any credentials,
// so it is safe to log and helps to see WHICH database the app is really using.
func (c *Config) DatabaseTarget() string {
	u, err := url.Parse(c.DatabaseURL)
	if err != nil {
		return "(unparseable)"
	}
	return u.Host + u.Path
}

const minSecretLength = 32

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
		JWTSecret:      get("JWT_SECRET", ""), // required: no default
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

	// The token lifetime: a duration like "15m" or "24h".
	ttlText := get("JWT_TTL", "24h")
	ttl, err := time.ParseDuration(ttlText)
	if err != nil || ttl <= 0 {
		errs = append(errs, fmt.Errorf("JWT_TTL must be a positive duration like 15m or 24h, got %q", ttlText))
	}
	cfg.JWTTTL = ttl

	// The secret: required, long enough, and (in production) not the example value.
	switch {
	case cfg.JWTSecret == "":
		errs = append(errs, errors.New("JWT_SECRET is required (generate one with: head -c 48 /dev/urandom | base64)"))
	case len(cfg.JWTSecret) < minSecretLength:
		errs = append(errs, fmt.Errorf("JWT_SECRET must be at least %d characters", minSecretLength))
	case cfg.Env == "production" && strings.HasPrefix(cfg.JWTSecret, "change-me"):
		errs = append(errs, errors.New("JWT_SECRET is still the example value; set a real secret in production"))
	}

	// The database.
	dbURL, dbErrs := resolveDatabaseURL(get, cfg.Env)
	cfg.DatabaseURL = dbURL
	errs = append(errs, dbErrs...)

	cfg.DBMaxOpenConns = 10
	if n, err := strconv.Atoi(get("DB_MAX_OPEN_CONNS", "10")); err != nil || n < 1 {
		errs = append(errs, errors.New("DB_MAX_OPEN_CONNS must be a whole number of at least 1"))
	} else {
		cfg.DBMaxOpenConns = n
	}

	cfg.DBMaxIdleConns = 5
	if n, err := strconv.Atoi(get("DB_MAX_IDLE_CONNS", "5")); err != nil || n < 0 {
		errs = append(errs, errors.New("DB_MAX_IDLE_CONNS must be a whole number of at least 0"))
	} else {
		cfg.DBMaxIdleConns = n
	}
	if cfg.DBMaxIdleConns > cfg.DBMaxOpenConns {
		errs = append(errs, errors.New("DB_MAX_IDLE_CONNS must not exceed DB_MAX_OPEN_CONNS"))
	}

	lifeText := get("DB_CONN_MAX_LIFETIME", "30m")
	life, err := time.ParseDuration(lifeText)
	if err != nil || life <= 0 {
		errs = append(errs, fmt.Errorf("DB_CONN_MAX_LIFETIME must be a positive duration like 30m, got %q", lifeText))
	}
	cfg.DBConnMaxLifetime = life

	stmtText := get("DB_STATEMENT_TIMEOUT", "10s")
	stmt, err := time.ParseDuration(stmtText)
	if err != nil || stmt <= 0 {
		errs = append(errs, fmt.Errorf("DB_STATEMENT_TIMEOUT must be a positive duration like 10s, got %q", stmtText))
	}
	cfg.DBStatementTimeout = stmt

	if err := errors.Join(errs...); err != nil {
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

// sslModes are the values PostgreSQL accepts for sslmode.
var sslModes = map[string]bool{
	"disable": true, "allow": true, "prefer": true, "require": true, "verify-ca": true, "verify-full": true,
}

// secureSSLModes are the only acceptable modes in production: they always encrypt.
var secureSSLModes = map[string]bool{"require": true, "verify-ca": true, "verify-full": true}

// resolveDatabaseURL returns the database URL: DATABASE_URL if set, otherwise one assembled from
// DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME and DB_SSLMODE. Error messages never contain the
// URL or the password, because both are secrets.
func resolveDatabaseURL(get func(key, def string) string, env string) (string, []error) {
	raw := get("DATABASE_URL", "")

	if raw == "" {
		host, user, pass, name := get("DB_HOST", ""), get("DB_USER", ""), get("DB_PASSWORD", ""), get("DB_NAME", "")
		if host == "" && user == "" && pass == "" && name == "" {
			return "", []error{errors.New("DATABASE_URL is required (or set DB_HOST, DB_USER, DB_PASSWORD and DB_NAME), e.g. postgres://user:password@localhost:5432/ecommerce")}
		}

		var errs []error
		for _, req := range []struct{ key, value string }{{"DB_HOST", host}, {"DB_USER", user}, {"DB_PASSWORD", pass}, {"DB_NAME", name}} {
			if req.value == "" {
				errs = append(errs, fmt.Errorf("%s is required when DATABASE_URL is not set", req.key))
			}
		}
		port := get("DB_PORT", "5432")
		if n, err := strconv.Atoi(port); err != nil || n < 1 || n > 65535 {
			errs = append(errs, fmt.Errorf("DB_PORT must be a number between 1 and 65535, got %q", port))
		}
		sslmode := get("DB_SSLMODE", "disable")
		if len(errs) > 0 {
			if !sslModes[sslmode] {
				errs = append(errs, fmt.Errorf("DB_SSLMODE must be one of disable, allow, prefer, require, verify-ca, verify-full, got %q", sslmode))
			}
			return "", errs
		}

		// net/url percent-encodes special characters in the password; net.JoinHostPort handles IPv6.
		u := url.URL{
			Scheme:   "postgres",
			User:     url.UserPassword(user, pass),
			Host:     net.JoinHostPort(host, port),
			Path:     "/" + name,
			RawQuery: url.Values{"sslmode": {sslmode}}.Encode(),
		}
		raw = u.String()
	}

	if err := validateDatabaseURL(raw, env); err != nil {
		return raw, []error{err}
	}
	return raw, nil
}

// validateDatabaseURL checks the shape and TLS mode of the URL without ever repeating it:
// url.Parse errors quote the whole input, which would leak the password into logs.
func validateDatabaseURL(raw, env string) error {
	u, err := url.Parse(raw)
	if err != nil || (u.Scheme != "postgres" && u.Scheme != "postgresql") || u.Hostname() == "" {
		return errors.New("DATABASE_URL must look like postgres://user:password@host:5432/dbname (special characters in the password must be percent-encoded)")
	}

	sslmode := u.Query().Get("sslmode")
	if sslmode != "" && !sslModes[sslmode] {
		return errors.New("the database sslmode must be one of disable, allow, prefer, require, verify-ca, verify-full")
	}
	if env == "production" && !secureSSLModes[sslmode] {
		return errors.New("in production the database connection must be encrypted: set sslmode=require, verify-ca or verify-full (verify-full is best)")
	}
	return nil
}
```

Read the resolver as a small state machine: **URL given → validate it. Otherwise → all four parts or a list of exactly which ones are missing → assemble with `net/url` → validate the result with the *same* function.** Both paths converge on one validation, so the production TLS rule can't be bypassed by choosing the other style.

### The connection code

`db.Connect` now uses `pgx` directly to parse the URL, set the timeouts, and hand the configured connector to `database/sql`:

```go
// file: infra/db/db.go
package db

import (
	"context"
	"fmt"
	"strconv"
	"time"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/stdlib"
	"github.com/jmoiron/sqlx"
)

const (
	// pingTimeout bounds how long start-up waits for the database to answer.
	pingTimeout = 5 * time.Second
	// connectTimeout bounds every attempt to open a new connection (also later, when the pool reconnects).
	connectTimeout = 5 * time.Second
)

// Options describes how to connect and how the connection pool should behave.
type Options struct {
	URL              string        // postgres://user:password@host:port/dbname
	MaxOpenConns     int           // hard cap on connections
	MaxIdleConns     int           // idle connections kept ready
	ConnMaxLifetime  time.Duration // connections older than this are recycled
	StatementTimeout time.Duration // the database cancels statements longer than this (0 = no limit)
}

// Connect creates a connection pool and verifies that the database is reachable.
// On failure it returns an error (which never contains the password) and no pool.
// The caller owns the returned pool and must Close it.
func Connect(ctx context.Context, opts Options) (*sqlx.DB, error) {
	cfg, err := pgx.ParseConfig(opts.URL) // its errors have the password masked
	if err != nil {
		return nil, fmt.Errorf("invalid database settings: %w", err)
	}
	cfg.ConnectTimeout = connectTimeout
	if opts.StatementTimeout > 0 {
		// A runtime parameter is applied by PostgreSQL to every connection in the pool, in milliseconds.
		cfg.RuntimeParams["statement_timeout"] = strconv.FormatInt(opts.StatementTimeout.Milliseconds(), 10)
	}

	conn := sqlx.NewDb(stdlib.OpenDB(*cfg), "pgx") // lazy: does not contact the server yet
	conn.SetMaxOpenConns(opts.MaxOpenConns)
	conn.SetMaxIdleConns(opts.MaxIdleConns)
	conn.SetConnMaxLifetime(opts.ConnMaxLifetime)

	// Now actually talk to the server, but never wait forever.
	ctx, cancel := context.WithTimeout(ctx, pingTimeout)
	defer cancel()
	if err := conn.PingContext(ctx); err != nil {
		conn.Close() // don't leak the half-built pool
		return nil, fmt.Errorf("cannot reach database: %w", err)
	}
	return conn, nil
}
```

What changed and why:

- **`pgx.ParseConfig`** replaces `sqlx.Open("pgx", url)`: we now hold a structured `Config` we can adjust before opening. `stdlib.OpenDB(*cfg)` turns it into a standard `*sql.DB`, and `sqlx.NewDb` wraps that (same type as before, so no other file changes).
- **`RuntimeParams`** are PostgreSQL settings sent when each connection starts; `statement_timeout` is in **milliseconds**.
- **`ConnectTimeout`** applies to *every* new connection the pool opens, not just the first. If the database vanishes and returns later, reconnection attempts also have a bound. (We verified: with a black-hole address and a 1 s setting, the attempt failed after exactly 1 s with `dial error: timeout`.)
- The blank driver import is gone (importing `stdlib` by name registers the `"pgx"` driver as a side effect too).

### `main` uses it, and logs *where* it is connecting

```go
// file: main.go
package main

import (
	"context"
	"ecommerce/cmd"
	"ecommerce/config"
	"ecommerce/infra/db"
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

	if err := run(cfg, logger); err != nil {
		logger.Error("server failed", "err", err)
		os.Exit(1)
	}
}

// run connects to the database and serves until shutdown. Deferred cleanup works because run returns normally.
func run(cfg *config.Config, logger *slog.Logger) error {
	// Log WHERE we are connecting (never the credentials) BEFORE trying: when the "wrong" database is
	// used, or the connection fails, this line is the first thing to check.
	logger.Info("connecting to database", "target", cfg.DatabaseTarget())

	conn, err := db.Connect(context.Background(), db.Options{
		URL:              cfg.DatabaseURL,
		MaxOpenConns:     cfg.DBMaxOpenConns,
		MaxIdleConns:     cfg.DBMaxIdleConns,
		ConnMaxLifetime:  cfg.DBConnMaxLifetime,
		StatementTimeout: cfg.DBStatementTimeout,
	})
	if err != nil {
		return err
	}
	defer conn.Close()
	logger.Info("database connected", "max_open", cfg.DBMaxOpenConns, "statement_timeout", cfg.DBStatementTimeout)

	return cmd.Serve(cfg, logger, conn)
}

// newLogger builds a logger in the format the configuration asks for.
func newLogger(cfg *config.Config) *slog.Logger {
	if cfg.LogFormat == "json" {
		return slog.New(slog.NewJSONHandler(os.Stdout, nil))
	}
	return slog.New(slog.NewTextHandler(os.Stdout, nil))
}
```

And the documented example configuration, now showing both styles:

```env
# file: .env.example
# Copy this file to ".env" and adjust. Real .env files are NEVER committed to Git.
APP_ENV=development
HTTP_PORT=8080
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:3000
LOG_FORMAT=text

# REQUIRED. At least 32 characters. Generate one with:  head -c 48 /dev/urandom | base64
JWT_SECRET=change-me-please-use-a-long-random-string-at-least-32-chars
# How long a login token stays valid (e.g. 15m, 24h).
JWT_TTL=24h

# ---- Database: choose ONE style. DATABASE_URL wins if both are set. ----
# Style 1: a single URL (percent-encode special characters in the password!)
DATABASE_URL=postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable
# Style 2: separate parts (the app escapes the password for you)
# DB_HOST=127.0.0.1
# DB_PORT=15432
# DB_USER=postgres
# DB_PASSWORD=devpass
# DB_NAME=ecommerce
# DB_SSLMODE=disable          # production needs require, verify-ca or verify-full

# Pool and timeouts (see Chapters 52 and 57).
DB_MAX_OPEN_CONNS=10
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME=30m
DB_STATEMENT_TIMEOUT=10s
```

---

## 8. Tests

The Chapter 52 config tests gain the new field in `TestDefaults`, and `TestOverrides` (which selects `production`) must now supply a secure `sslmode`, because production rejects `disable`. Everything else is unchanged:

```go
// file: config/config_test.go
package config

import (
	"reflect"
	"strings"
	"testing"
	"time"
)

const (
	goodSecret = "0123456789abcdef0123456789abcdef0123" // 36 characters
	goodDBURL  = "postgres://app:s3cret-pw@localhost:5432/ecommerce?sslmode=disable"
)

func lookupFrom(env map[string]string) func(string) (string, bool) {
	return func(key string) (string, bool) {
		v, ok := env[key]
		return v, ok
	}
}

// valid returns env plus the two required settings (a JWT secret and a database URL).
func valid(env map[string]string) map[string]string {
	out := map[string]string{"JWT_SECRET": goodSecret, "DATABASE_URL": goodDBURL}
	for k, v := range env {
		out[k] = v
	}
	return out
}

func TestDefaults(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(valid(nil)))
	if err != nil {
		t.Fatal(err)
	}
	want := &Config{
		Env:                "development",
		Port:               "8080",
		AllowedOrigins:     []string{"http://localhost:5173"},
		LogFormat:          "text",
		JWTSecret:          goodSecret,
		JWTTTL:             24 * time.Hour,
		DatabaseURL:        goodDBURL,
		DBMaxOpenConns:     10,
		DBMaxIdleConns:     5,
		DBConnMaxLifetime:  30 * time.Minute,
		DBStatementTimeout: 10 * time.Second,
	}
	if !reflect.DeepEqual(cfg, want) {
		t.Errorf("got %+v, want %+v", cfg, want)
	}
}

func TestOverrides(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(valid(map[string]string{
		"APP_ENV":              "production",
		"HTTP_PORT":            "9090",
		"ALLOWED_ORIGINS":      "https://shop.example.com, https://admin.example.com/",
		"LOG_FORMAT":           "json",
		"JWT_TTL":              "15m",
		"DATABASE_URL":         "postgres://app:s3cret-pw@db.internal:5432/ecommerce?sslmode=verify-full",
		"DB_MAX_OPEN_CONNS":    "25",
		"DB_MAX_IDLE_CONNS":    "25",
		"DB_CONN_MAX_LIFETIME": "1h",
		"DB_STATEMENT_TIMEOUT": "3s",
	})))
	if err != nil {
		t.Fatal(err)
	}
	if cfg.Env != "production" || cfg.Port != "9090" || cfg.LogFormat != "json" || cfg.JWTTTL != 15*time.Minute {
		t.Errorf("unexpected config %+v", cfg)
	}
	if len(cfg.AllowedOrigins) != 2 {
		t.Errorf("expected 2 origins, got %v", cfg.AllowedOrigins)
	}
	if cfg.DBMaxOpenConns != 25 || cfg.DBMaxIdleConns != 25 || cfg.DBConnMaxLifetime != time.Hour || cfg.DBStatementTimeout != 3*time.Second {
		t.Errorf("unexpected pool settings %+v", cfg)
	}
}

func TestEmptyValuesFallBackToDefaults(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(valid(map[string]string{"HTTP_PORT": "   ", "LOG_FORMAT": "", "DB_MAX_OPEN_CONNS": ""})))
	if err != nil {
		t.Fatal(err)
	}
	if cfg.Port != "8080" || cfg.LogFormat != "text" || cfg.DBMaxOpenConns != 10 {
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
		{"unknown environment", map[string]string{"APP_ENV": "staging"}, "APP_ENV"},
		{"unknown log format", map[string]string{"LOG_FORMAT": "xml"}, "LOG_FORMAT"},
		{"origin without scheme", map[string]string{"ALLOWED_ORIGINS": "localhost:5173"}, "ALLOWED_ORIGINS"},
		{"origin with a path", map[string]string{"ALLOWED_ORIGINS": "https://shop.example.com/app"}, "ALLOWED_ORIGINS"},
		{"ttl not a duration", map[string]string{"JWT_TTL": "tomorrow"}, "JWT_TTL"},
		{"ttl zero", map[string]string{"JWT_TTL": "0s"}, "JWT_TTL"},
		{"ttl negative", map[string]string{"JWT_TTL": "-5m"}, "JWT_TTL"},
		{"max open is zero", map[string]string{"DB_MAX_OPEN_CONNS": "0"}, "DB_MAX_OPEN_CONNS"},
		{"max open is text", map[string]string{"DB_MAX_OPEN_CONNS": "many"}, "DB_MAX_OPEN_CONNS"},
		{"max idle negative", map[string]string{"DB_MAX_IDLE_CONNS": "-1"}, "DB_MAX_IDLE_CONNS"},
		{"idle exceeds open", map[string]string{"DB_MAX_OPEN_CONNS": "4", "DB_MAX_IDLE_CONNS": "8"}, "must not exceed"},
		{"lifetime not a duration", map[string]string{"DB_CONN_MAX_LIFETIME": "forever"}, "DB_CONN_MAX_LIFETIME"},
		{"lifetime zero", map[string]string{"DB_CONN_MAX_LIFETIME": "0s"}, "DB_CONN_MAX_LIFETIME"},
		{"statement timeout not a duration", map[string]string{"DB_STATEMENT_TIMEOUT": "soon"}, "DB_STATEMENT_TIMEOUT"},
		{"statement timeout zero", map[string]string{"DB_STATEMENT_TIMEOUT": "0s"}, "DB_STATEMENT_TIMEOUT"},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			cfg, err := fromLookup(lookupFrom(valid(tc.env)))
			if err == nil {
				t.Fatalf("expected an error, got config %+v", cfg)
			}
			if !strings.Contains(err.Error(), tc.want) {
				t.Errorf("error %q should mention %q", err, tc.want)
			}
		})
	}
}

func TestJWTSecretRules(t *testing.T) {
	tests := []struct {
		name string
		env  map[string]string
	}{
		{"missing", map[string]string{"DATABASE_URL": goodDBURL}},
		{"empty", map[string]string{"DATABASE_URL": goodDBURL, "JWT_SECRET": ""}},
		{"too short", map[string]string{"DATABASE_URL": goodDBURL, "JWT_SECRET": "short"}},
		{"example value in production", map[string]string{
			"DATABASE_URL": goodDBURL,
			"APP_ENV":      "production",
			"JWT_SECRET":   "change-me-please-use-a-long-random-string-at-least-32-chars",
		}},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			_, err := fromLookup(lookupFrom(tc.env))
			if err == nil || !strings.Contains(err.Error(), "JWT_SECRET") {
				t.Errorf("expected a JWT_SECRET error, got %v", err)
			}
		})
	}
}

func TestExampleSecretIsAcceptedInDevelopment(t *testing.T) {
	_, err := fromLookup(lookupFrom(valid(map[string]string{
		"JWT_SECRET": "change-me-please-use-a-long-random-string-at-least-32-chars",
	})))
	if err != nil {
		t.Errorf("the example .env must work out of the box in development: %v", err)
	}
}

func TestDatabaseURLRules(t *testing.T) {
	tests := []struct {
		name string
		url  string
		ok   bool
	}{
		{"postgres scheme", "postgres://u:p@localhost:5432/db", true},
		{"postgresql scheme", "postgresql://u:p@db.internal/db?sslmode=verify-full", true},
		{"no password is fine (e.g. peer auth)", "postgres://u@localhost/db", true},
		{"missing entirely", "", false},
		{"wrong scheme", "mysql://u:p@localhost:3306/db", false},
		{"no host", "postgres:///db", false},
		{"not a url", "just some text", false},
		{"unescaped special characters", "postgres://u:p ss@host:x/db", false},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			_, err := fromLookup(lookupFrom(valid(map[string]string{"DATABASE_URL": tc.url})))
			if tc.ok && err != nil {
				t.Errorf("expected %q to be accepted, got %v", tc.url, err)
			}
			if !tc.ok {
				if err == nil || !strings.Contains(err.Error(), "DATABASE_URL") {
					t.Errorf("expected a DATABASE_URL error for %q, got %v", tc.url, err)
				}
			}
		})
	}
}

func TestDatabaseErrorsNeverEchoThePassword(t *testing.T) {
	for _, bad := range []string{
		"mysql://app:hunter2@localhost/db",
		"postgres://app:hunter2@:x/db",
		"postgres://app:hun ter2@host/db",
	} {
		_, err := fromLookup(lookupFrom(valid(map[string]string{"DATABASE_URL": bad})))
		if err == nil {
			t.Fatalf("%q should be rejected", bad)
		}
		if strings.Contains(err.Error(), "hunter2") || strings.Contains(err.Error(), "hun ter2") {
			t.Errorf("the error leaked the password: %q", err)
		}
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
	for _, key := range []string{"HTTP_PORT", "APP_ENV", "LOG_FORMAT", "JWT_SECRET", "DATABASE_URL"} {
		if !strings.Contains(err.Error(), key) {
			t.Errorf("combined error should mention %s: %q", key, err)
		}
	}
}

func TestStringRedactsSecrets(t *testing.T) {
	cfg := &Config{JWTSecret: goodSecret, DatabaseURL: goodDBURL, Port: "8080"}
	s := cfg.String()
	if strings.Contains(s, goodSecret) {
		t.Error("String() leaked the JWT secret")
	}
	if strings.Contains(s, "s3cret-pw") || strings.Contains(s, "postgres://") {
		t.Error("String() leaked the database URL")
	}
}
```

New tests for the parts style, the URL escaping (verified through the real driver's parser), TLS rules, and the safe-to-log target:

```go
// file: config/database_test.go
package config

import (
	"strings"
	"testing"

	"github.com/jackc/pgx/v5"
)

// parts returns a valid parts-style environment (no DATABASE_URL) plus overrides.
func parts(overrides map[string]string) map[string]string {
	out := map[string]string{
		"JWT_SECRET":  goodSecret,
		"DB_HOST":     "db.internal",
		"DB_USER":     "app",
		"DB_PASSWORD": "s3cret-pw",
		"DB_NAME":     "ecommerce",
	}
	for k, v := range overrides {
		out[k] = v
	}
	return out
}

func TestDatabaseURLIsBuiltFromParts(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(parts(map[string]string{"DB_PORT": "15432", "DB_SSLMODE": "require"})))
	if err != nil {
		t.Fatal(err)
	}
	if want := "postgres://app:s3cret-pw@db.internal:15432/ecommerce?sslmode=require"; cfg.DatabaseURL != want {
		t.Errorf("got %q, want %q", cfg.DatabaseURL, want)
	}
}

func TestPartsDefaults(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(parts(nil)))
	if err != nil {
		t.Fatal(err)
	}
	if !strings.Contains(cfg.DatabaseURL, "@db.internal:5432/") || !strings.HasSuffix(cfg.DatabaseURL, "sslmode=disable") {
		t.Errorf("expected port 5432 and sslmode=disable by default, got %q", cfg.DatabaseURL)
	}
}

func TestSpecialCharactersInThePasswordSurviveTheRoundTrip(t *testing.T) {
	for _, password := range []string{"p@ss/w:rd#1", "100%sure", "a b c", "üñí€ödé", "?&=+;", "[::1]"} {
		cfg, err := fromLookup(lookupFrom(parts(map[string]string{"DB_PASSWORD": password})))
		if err != nil {
			t.Fatalf("%q: %v", password, err)
		}
		// Ask the real driver to parse our URL: the password and host must come back intact.
		parsed, err := pgx.ParseConfig(cfg.DatabaseURL)
		if err != nil {
			t.Fatalf("%q: the driver cannot parse %q: %v", password, cfg.DatabaseURL, err)
		}
		if parsed.Password != password || parsed.Host != "db.internal" || parsed.User != "app" || parsed.Database != "ecommerce" {
			t.Errorf("%q did not round-trip: %+v", password, parsed)
		}
	}
}

func TestIPv6HostsAreBracketed(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(parts(map[string]string{"DB_HOST": "::1"})))
	if err != nil {
		t.Fatal(err)
	}
	parsed, err := pgx.ParseConfig(cfg.DatabaseURL)
	if err != nil || parsed.Host != "::1" || parsed.Port != 5432 {
		t.Errorf("IPv6 host not handled: %q -> %+v (%v)", cfg.DatabaseURL, parsed, err)
	}
}

func TestMissingPartsAreAllReported(t *testing.T) {
	_, err := fromLookup(lookupFrom(map[string]string{
		"JWT_SECRET": goodSecret,
		"DB_HOST":    "db.internal", // the others are missing
	}))
	if err == nil {
		t.Fatal("expected an error")
	}
	for _, key := range []string{"DB_USER", "DB_PASSWORD", "DB_NAME"} {
		if !strings.Contains(err.Error(), key) {
			t.Errorf("the error should name %s: %q", key, err)
		}
	}
}

func TestDatabaseURLWinsOverParts(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(parts(map[string]string{"DATABASE_URL": goodDBURL})))
	if err != nil {
		t.Fatal(err)
	}
	if cfg.DatabaseURL != goodDBURL {
		t.Errorf("DATABASE_URL must take precedence, got %q", cfg.DatabaseURL)
	}
}

func TestBadPortAndSSLModeInPartsMode(t *testing.T) {
	for name, env := range map[string]map[string]string{
		"port":    {"DB_PORT": "postgres"},
		"sslmode": {"DB_SSLMODE": "sometimes"},
	} {
		if _, err := fromLookup(lookupFrom(parts(env))); err == nil {
			t.Errorf("%s: expected an error", name)
		}
	}
}

func TestProductionRequiresEncryptedDatabaseConnections(t *testing.T) {
	prod := func(env map[string]string) map[string]string {
		env["APP_ENV"] = "production"
		env["JWT_SECRET"] = goodSecret
		return env
	}

	tests := []struct {
		name string
		env  map[string]string
		ok   bool
	}{
		{"url without sslmode", prod(map[string]string{"DATABASE_URL": "postgres://u:p@db/x"}), false},
		{"url with disable", prod(map[string]string{"DATABASE_URL": "postgres://u:p@db/x?sslmode=disable"}), false},
		{"url with prefer", prod(map[string]string{"DATABASE_URL": "postgres://u:p@db/x?sslmode=prefer"}), false},
		{"url with require", prod(map[string]string{"DATABASE_URL": "postgres://u:p@db/x?sslmode=require"}), true},
		{"url with verify-full", prod(map[string]string{"DATABASE_URL": "postgres://u:p@db/x?sslmode=verify-full"}), true},
		{"parts with the default mode", prod(parts(nil)), false},
		{"parts with verify-ca", prod(parts(map[string]string{"DB_SSLMODE": "verify-ca"})), true},
		{"unknown mode in a url", prod(map[string]string{"DATABASE_URL": "postgres://u:p@db/x?sslmode=yes"}), false},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			_, err := fromLookup(lookupFrom(tc.env))
			if tc.ok && err != nil {
				t.Errorf("should be accepted: %v", err)
			}
			if !tc.ok && (err == nil || !strings.Contains(err.Error(), "sslmode")) {
				t.Errorf("should be rejected with an sslmode message, got %v", err)
			}
		})
	}
}

func TestDevelopmentMayUseUnencryptedConnections(t *testing.T) {
	if _, err := fromLookup(lookupFrom(parts(nil))); err != nil {
		t.Errorf("development must default to something that works with a local container: %v", err)
	}
}

func TestDatabaseTargetHasNoCredentials(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(parts(map[string]string{"DB_PASSWORD": "hunter2", "DB_PORT": "15432"})))
	if err != nil {
		t.Fatal(err)
	}
	target := cfg.DatabaseTarget()
	if target != "db.internal:15432/ecommerce" {
		t.Errorf("target = %q", target)
	}
	if strings.Contains(target, "hunter2") || strings.Contains(target, "app") {
		t.Errorf("the target must not include credentials: %q", target)
	}
}
```

And the timeout behavior itself, against a real database (skipped without `TEST_DATABASE_URL`), plus a test that needs none:

```go
// file: infra/db/options_test.go
package db

import (
	"context"
	"errors"
	"strings"
	"testing"
	"time"

	"github.com/jackc/pgx/v5/pgconn"
)

func TestMalformedURLErrorDoesNotLeakThePassword(t *testing.T) {
	conn, err := Connect(context.Background(), Options{URL: "postgres://app:top-secret-pw@bad host:x/db"})
	if err == nil {
		conn.Close()
		t.Fatal("expected an error")
	}
	if strings.Contains(err.Error(), "top-secret-pw") {
		t.Errorf("the error leaked the password: %v", err)
	}
}

func TestStatementTimeoutIsAppliedToEveryConnection(t *testing.T) {
	o := opts(testURL(t))
	o.StatementTimeout = 1500 * time.Millisecond
	conn, err := Connect(context.Background(), o)
	if err != nil {
		t.Fatal(err)
	}
	defer conn.Close()

	var got string
	if err := conn.GetContext(context.Background(), &got, "SHOW statement_timeout"); err != nil {
		t.Fatal(err)
	}
	if got != "1500ms" {
		t.Errorf("statement_timeout = %q, want 1500ms", got)
	}
}

func TestTheDatabaseCancelsRunawayStatements(t *testing.T) {
	o := opts(testURL(t))
	o.StatementTimeout = 200 * time.Millisecond
	conn, err := Connect(context.Background(), o)
	if err != nil {
		t.Fatal(err)
	}
	defer conn.Close()

	start := time.Now()
	// The Go context has NO deadline: only the database's own limit can stop this 5-second query.
	_, err = conn.ExecContext(context.Background(), "SELECT pg_sleep(5)")
	elapsed := time.Since(start)

	var pgErr *pgconn.PgError
	if !errors.As(err, &pgErr) || pgErr.Code != "57014" {
		t.Fatalf("expected SQLSTATE 57014 (query_canceled), got %v", err)
	}
	if elapsed > 2*time.Second {
		t.Errorf("the statement should have been cut off after ~200ms, took %s", elapsed)
	}
}

func TestWithoutAStatementTimeoutNothingIsCancelled(t *testing.T) {
	conn, err := Connect(context.Background(), opts(testURL(t))) // StatementTimeout is zero: no limit
	if err != nil {
		t.Fatal(err)
	}
	defer conn.Close()

	if _, err := conn.ExecContext(context.Background(), "SELECT pg_sleep(0.3)"); err != nil {
		t.Errorf("a short query must succeed without a limit: %v", err)
	}
}
```

Run:

```bash
go vet ./... && go test -race ./...
TEST_DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable' go test -race -count=1 ./config/ ./infra/db/ -v -run 'Timeout|Cancels|Malformed|Production|RoundTrip'
```

Look at what these tests protect: the password-with-special-characters case (a bug that only appears when some user picks a "strong" password), the **production TLS guard** (one typo away from sending customer data in clear text), and the **statement timeout** (proving the database, not Go, stops the runaway query).

---

## 9. Environment variable precedence

`config.Load` calls `godotenv.Load()`, which reads `.env` and sets each variable **only if it isn't already set in the real environment**. So the priority order is:

```
1. Real environment variables (export FOO=..., docker -e, Kubernetes env, CI settings)   ← highest
2. Values from the .env file                                                             ← fills gaps only
3. Defaults in code                                                                      ← lowest
```

That's the right design (a deployment platform must be able to override a file in the image), but it causes the most common configuration mystery: **"I edited `.env` and nothing changed."** Usually a variable with the same name is *already exported in your shell* (from another project, a profile script, or an earlier experiment), and it silently beats your file.

Real reproduction. The `.env` file points at the working database (port `15432`), but the shell has an old export pointing at port `1`:

```bash
export DATABASE_URL='postgres://postgres:devpass@127.0.0.1:1/ecommerce?sslmode=disable'   # stale, from an old session
cat .env | grep DATABASE_URL          # …:15432/… (what you think is used)
go run .
```
```
time=2026-09-26T16:09:51.803+06:00 level=INFO msg="connecting to database" target=127.0.0.1:1/ecommerce
time=2026-09-26T16:09:51.804+06:00 level=ERROR msg="server failed" err="cannot reach database: failed to connect to `user=postgres database=ecommerce`: 127.0.0.1:1 (127.0.0.1): dial error: dial tcp 127.0.0.1:1: connect: connection refused"
```

The `target=127.0.0.1:1/ecommerce` field of the first log line is why we log `DatabaseTarget()` *before* connecting: it shows **which database was really chosen**, without exposing credentials, even when the connection then fails. To debug:

```bash
env | grep -E '^(DATABASE_URL|DB_|APP_ENV|HTTP_PORT|JWT_)'   # what is set in THIS shell (careful: it may print secrets on screen)
unset DATABASE_URL                                           # remove the stale export
```

How to avoid the trap:

- Prefer running with a **clean environment for tests**: `env -i PATH="$PATH" HOME="$HOME" go run .` starts with *no* inherited variables (only what `.env` provides).
- Use project-specific prefixes (`SHOP_DATABASE_URL`) if your machine hosts many projects using the generic names.
- If you *want* the file to win (rare; mostly for demos), `godotenv.Overload()` overrides the environment. Don't do that in production images.
- Document the order (as above) in your README. Future-you will forget.

Also remember `.env` is a **development convenience**. In production nobody should be editing files on the server: variables come from the platform (Compose `environment:`, Kubernetes `env`/`Secret`, cloud service settings).

### Seeing the TLS ladder in practice

The stock container has TLS disabled, so requiring it fails, exactly what `sslmode=require` is *for*:

```bash
DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=require' go run .
```
```
time=2026-09-26T16:09:51.808+06:00 level=INFO msg="connecting to database" target=127.0.0.1:15432/ecommerce
time=2026-09-26T16:09:51.809+06:00 level=ERROR msg="server failed" err="cannot reach database: failed to connect to `user=postgres database=ecommerce`: 127.0.0.1:15432 (127.0.0.1): tls error: server refused TLS connection"
```

And production refuses an insecure setting before it ever connects (only `APP_ENV=production` differs):

```bash
APP_ENV=production DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable' go run .
```
```
configuration error:
in production the database connection must be encrypted: set sslmode=require, verify-ca or verify-full (verify-full is best)
```

---

## 10. Docker Compose for local PostgreSQL

`docker run …` (Chapter 52) is fine once. A **Compose file** records the whole setup in a file you commit, so a teammate runs one command. Features we use: a **named volume** (data survives container removal), a **health check** (`docker compose up --wait` blocks until Postgres accepts connections), **init scripts** (files in `/docker-entrypoint-initdb.d` run automatically the first time the database is created, so our schema appears on its own), and **variable substitution** with defaults (`${DB_PORT:-5432}`).

```yaml
# file: compose.yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-devpass}
      POSTGRES_DB: ${DB_NAME:-ecommerce}
    ports:
      - "127.0.0.1:${DB_PORT:-5432}:5432"      # localhost only: not reachable from the network
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./sql/001_create_users.sql:/docker-entrypoint-initdb.d/001_create_users.sql:ro
      - ./sql/002_create_products.sql:/docker-entrypoint-initdb.d/002_create_products.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 2s
      timeout: 3s
      retries: 15
    restart: unless-stopped

volumes:
  pgdata:
```

Points to understand:

- **`${DB_PASSWORD:-devpass}`** means "use `DB_PASSWORD` from the environment (or a `.env` file next to `compose.yaml`), else `devpass`". Notice that our *application* uses the same variable names (`DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_PORT`), so **one `.env` file configures both the database container and the app** with no duplication.
- **`127.0.0.1:` in the port mapping** publishes the port only on your loopback interface. Writing just `"5432:5432"` exposes the database to every network your machine is on (a very common way development databases end up on the internet).
- **`$${POSTGRES_USER}`**: the doubled `$` stops *Compose* from substituting it, so the shell **inside** the container expands it.
- **Init scripts run only when the data directory is empty** (first start). After that, changing them has no effect until you recreate the volume (`docker compose down -v`). That's a common puzzle; Chapter 58's migrations solve it properly by applying pending changes on every start.
- `restart: unless-stopped` brings the database back after a reboot.

Use it (the `-p shopdemo` project name and port `15433` are what we used so as not to touch other containers on the machine; you can omit both):

```bash
DB_PORT=15433 docker compose -p shopdemo up -d --wait
```
```
 Network shopdemo_default  Created
 Container shopdemo-db-1  Created
 Container shopdemo-db-1  Started
 Container shopdemo-db-1  Waiting
 Container shopdemo-db-1  Healthy
```

Check that the schema was created by the init scripts and run the application against it, using the **parts** style this time:

```bash
docker compose -p shopdemo exec db psql -U postgres -d ecommerce -c '\dt'
DATABASE_URL= DB_HOST=127.0.0.1 DB_PORT=15433 DB_USER=postgres DB_PASSWORD=devpass DB_NAME=ecommerce go run .   # empty DATABASE_URL= so the parts are used
```
```
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | products | table | postgres
 public | users    | table | postgres
(2 rows)

time=… level=INFO msg="connecting to database" target=127.0.0.1:15433/ecommerce
time=… level=INFO msg="database connected" max_open=10 statement_timeout=10s
time=… level=INFO msg="server starting" addr=:18080 env=development
```

Tidy up (`-v` also deletes the named volume: the data):

```bash
docker compose -p shopdemo down -v
```

**What about running the *app* in Compose too?** You would add a second service built from a `Dockerfile` (multi-stage: compile in a Go image, copy the static binary into a tiny runtime image) with `depends_on: db: condition: service_healthy` and `environment: DB_HOST: db` (inside the Compose network, the database is reachable by its **service name** `db`, not `localhost`). That's Exercise 6. During development it's usually more comfortable to run the app on the host (fast rebuilds, debugger) and only the *dependencies* in Compose, exactly as we've done.

---

## 11. Secrets management

A **secret** is any value that grants access: database passwords, JWT signing keys, API keys. Where they live, from worst to best:

| Approach | Verdict |
|----------|---------|
| Hard-coded in source | ❌ never. Leaks to everyone with repo access, and **stays in Git history forever** even after deletion |
| Committed `.env` | ❌ same as above |
| `.env` file, **git-ignored**, with a committed `.env.example` (no real secrets) | ✅ fine for local development |
| Environment variables injected by the platform (Compose, Kubernetes, CI) | ✅ standard for deployments |
| A **secret manager** (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, Doppler, 1Password CLI…) feeding the platform | ✅✅ best: access control, audit trail, rotation |

Habits that matter:

- **Different secrets per environment.** A leaked staging password must not open production.
- **Least privilege.** The app's database user needs `SELECT/INSERT/UPDATE/DELETE` on its tables, **not** superuser rights. Create a dedicated role (`CREATE ROLE shop_app LOGIN PASSWORD '…'; GRANT …`). We used the `postgres` superuser in these chapters purely for convenience: fine locally, **never in production**.
- **Rotate** secrets (change them periodically and immediately after any suspected leak). Short `DB_CONN_MAX_LIFETIME` helps: connections using an old credential get recycled.
- **Never log secrets** (our `Config.String()` redacts them), **never include them in error messages**, and never put them in URLs users see or in command-line arguments (visible in `ps`).
- If you ever commit a secret by mistake: **rotate it immediately**; deleting the commit isn't enough because clones and caches exist.

---

## 12. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Building the URL with `fmt.Sprintf` | Passwords with `@ / : # %` break the URL | `net/url` (`url.UserPassword`) |
| 2 | Hard-coding connection details | Secrets in Git; can't change per environment | Environment variables |
| 3 | `sslmode=disable` (or missing) in production | Traffic and credentials in clear text | Config rejects it; use `verify-full` |
| 4 | `sslmode=prefer`/`allow` | Attacker can force a downgrade | `require` at least; `verify-full` ideally |
| 5 | No statement/connect timeouts | One slow query or dead host stalls everything | `statement_timeout`, `ConnectTimeout`, contexts |
| 6 | Stale exported variable overrides `.env` | "My change is ignored" | Log the target, `env \| grep`, `unset`, prefixes |
| 7 | Editing init scripts and expecting a rerun | Nothing happens (data dir isn't empty) | `docker compose down -v` (or migrations) |
| 8 | Publishing `5432:5432` on all interfaces | Database exposed to the network | `127.0.0.1:5432:5432` |
| 9 | Using the superuser in production | A SQL-injection bug becomes total compromise | Dedicated least-privilege role |
| 10 | Same password everywhere | One leak opens every environment | Distinct secrets, rotated |
| 11 | Including the URL in an error or log | Password in log aggregators | Redact; log only host and database |
| 12 | Committing `.env` | Permanent leak | `.gitignore` + `.env.example` |
| 13 | Timeouts inconsistent across layers (statement > request > server) | Confusing failures, killed connections | Innermost limit smallest |
| 14 | `docker compose down -v` on data you wanted | Data destroyed | Know that `-v` deletes volumes |

---

## 13. Interview questions

**Q1. Why store configuration in the environment?**
The same build runs everywhere with different settings; no secrets in code; easy overrides by the platform (twelve-factor).

**Q2. Why not `fmt.Sprintf` a database URL?**
Special characters in the password/username change how the URL parses; `net/url` percent-encodes them correctly (and handles IPv6 hosts).

**Q3. Difference between `sslmode=require` and `verify-full`?**
`require` encrypts but doesn't check who the server is (vulnerable to man-in-the-middle); `verify-full` also validates the certificate chain and hostname.

**Q4. What is `statement_timeout` and why set it on the server side?**
A PostgreSQL parameter that cancels any statement running longer than the limit (SQLSTATE 57014). It protects the database even when client code forgets a context deadline.

**Q5. Which wins: an environment variable or the same name in `.env`?**
The real environment variable; `godotenv.Load` only fills in missing ones. Use `Overload` to invert (rarely wise).

**Q6. How do you make an app fail safely in production?**
Validate at startup and refuse insecure configurations (unencrypted DB, example secrets), reporting all problems at once.

**Q7. Why bind the Postgres port to `127.0.0.1` in Compose?**
Otherwise the database listens on every interface and can be reached from the network.

**Q8. Why do init scripts in `docker-entrypoint-initdb.d` sometimes not run?**
They run only when the data directory is empty (first initialization); an existing volume skips them.

**Q9. Why should the app not connect as the `postgres` superuser?**
Least privilege: if the app is compromised (or has an injection bug), the damage is limited to what its role may do.

---

## 14. Exercises

### Exercise 1: Reproduce and fix the precedence trap
Export a wrong `DATABASE_URL` in your shell, keep the right one in `.env`, and start the app. Read the `target=` log line. Fix it three different ways.

<details><summary>Solution</summary>

`unset DATABASE_URL`; or run with `env -u DATABASE_URL go run .`; or start with a clean environment (`env -i PATH="$PATH" HOME="$HOME" go run .`). (Bonus: rename your project's variables with a prefix.)
</details>

### Exercise 2: Passwords from hell
Add a test using the password `"'; DROP TABLE users; --"` and one containing a newline and a NUL byte. What happens with each in `url.UserPassword` → `pgx.ParseConfig`? Should the config code reject any?

<details><summary>Solution</summary>

Quotes and semicolons are fine (they're just percent-encoded/data; injection isn't a URL concern). Control characters such as a newline are encoded as `%0A` and round-trip, but a NUL (`\x00`) can't appear in PostgreSQL strings and will fail at authentication. Rejecting control characters in `DB_PASSWORD` early gives a clearer error than a cryptic auth failure.
</details>

### Exercise 3: `verify-full` for real
Generate a self-signed certificate with `openssl`, start Postgres with `-c ssl=on -c ssl_cert_file=… -c ssl_key_file=…`, connect with `sslmode=verify-full&sslrootcert=ca.pem`, and observe what fails when the hostname doesn't match the certificate.

<details><summary>Solution</summary>

`verify-full` errors with a message like `x509: certificate is valid for db.internal, not localhost` when the certificate's Subject Alternative Name doesn't include the host you dialed; `verify-ca` would accept it. Fix by issuing the certificate for the right name (or dialing by that name).
</details>

### Exercise 4: Wire the timeout into HTTP
Add an `http.TimeoutHandler` (or a per-request `context.WithTimeout` middleware) of 8 s in front of the API, keeping `DB_STATEMENT_TIMEOUT` at 10 s. Is this ordering right according to the layering rule? Fix the defaults.

<details><summary>Solution</summary>

No: the request limit (8 s) would fire *before* the statement limit (10 s), so the client gets a generic timeout and the query keeps running until the database kills it. Make the statement timeout the smallest (e.g., 5 s statement < 8 s request < 30 s `WriteTimeout`).
</details>

### Exercise 5: A least-privilege role
Write SQL creating role `shop_app` (login, password from a variable) that can read/write only `users` and `products`, and cannot create or drop tables. Connect the app as that role. What breaks when you run schema files with it?

<details><summary>Solution</summary>

```sql
CREATE ROLE shop_app LOGIN PASSWORD 'change-me';
GRANT USAGE ON SCHEMA public TO shop_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON users, products TO shop_app;
-- identity columns need their sequences:
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO shop_app;
```
Running `CREATE TABLE` with that role fails (`permission denied`): schema changes are done by a separate, more privileged "migration" role (Chapter 58), while the running app holds only data access.
</details>

### Exercise 6: Put the app in Compose
Write a multi-stage `Dockerfile` (build with the Go image, run a static binary in a minimal image, as a **non-root** user) and an `app` service with `depends_on` + health check. Which host name does `DB_HOST` use inside Compose?

<details><summary>Solution</summary>

```dockerfile
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/ecommerce .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/ecommerce /ecommerce
USER nonroot
ENTRYPOINT ["/ecommerce"]
```
```yaml
  app:
    build: .
    environment: { DB_HOST: db, DB_PORT: "5432", DB_USER: postgres, DB_PASSWORD: devpass, DB_NAME: ecommerce, JWT_SECRET: "..." }
    depends_on: { db: { condition: service_healthy } }
    ports: ["127.0.0.1:8080:8080"]
```
Inside the Compose network the host is the **service name**, `db`, and the port is the container's own `5432` (not the published host port).
</details>

---

## 15. Quiz

1. Which wins: an exported variable or the same variable in `.env`?
2. Why is `sslmode=prefer` unacceptable in production?
3. Which layer cancels a runaway query even if Go forgets a context?
4. What does `url.UserPassword` do that `Sprintf` does not?
5. When do Compose init scripts run?
6. Why bind the database port to `127.0.0.1`?
7. What's the correct ordering of statement, request, and server write timeouts?
8. Why should the running app use a least-privilege role?

<details><summary>Answers</summary>

1. The exported (real environment) variable.
2. It falls back to plain text, so an attacker can force a downgrade; it also never verifies the server.
3. The database, via `statement_timeout` (SQLSTATE 57014).
4. It percent-encodes special characters so the password survives parsing.
5. Only when the data directory is empty (first initialization).
6. So only local processes can reach it, not the whole network.
7. Statement (smallest) < request < server write timeout.
8. To limit the damage from a compromise or an injection bug to what the app genuinely needs.
</details>

---

## 16. Summary

- **Configuration comes from the environment**, validated at startup with every problem reported (Chapter 48), and never containing secrets in code.
- Support **`DATABASE_URL`** *or* **separate parts**; both feed one validated URL. Build URLs with **`net/url`** (escaping, IPv6), never `Sprintf`.
- **TLS ladder:** `disable` < `allow` < `prefer` < `require` < `verify-ca` < `verify-full`. **Production must encrypt (and ideally verify)**; the config refuses anything else, *including a missing `sslmode`*.
- **Timeouts at every layer:** connect timeout (driver), **statement timeout** (enforced by PostgreSQL), request context (caller); the innermost fires first.
- **Precedence:** real environment > `.env` > defaults. "My `.env` is ignored" nearly always means a stale exported variable: log `DatabaseTarget()` (credentials-free) to see the truth.
- **Docker Compose** gives a one-command database with a named volume, health check, `127.0.0.1` binding, init scripts, and variables shared with the app.
- **Secrets:** never in Git, distinct per environment, least-privilege roles, rotated, never logged.

### ➡️ What's next?

[Chapter 58](58-database-migrations.md) replaces the "run this SQL file by hand" step with **database migrations**: versioned up/down scripts tracked in the database itself, applied automatically and safely at startup or from a command.
