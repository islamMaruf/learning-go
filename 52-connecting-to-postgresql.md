# Chapter 52: Connecting to PostgreSQL — Infrastructure, `database/sql`, and Connection Pools

> **Goal of this chapter:** Give the application a real database. You'll learn what **infrastructure** is, how Go talks to databases (`database/sql`, drivers, `sqlx`, and the alternatives), how to run **PostgreSQL** locally with Docker, what a **connection string (DSN)** is, and how a **connection pool** works. Then you'll build the project's `infra/db` package: a `Connect` function that opens a pool, verifies it with a timeout, never leaks the password, and fails fast, plus **liveness and readiness endpoints** and an **integration test**.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 4 hours  **Prerequisite:** [Chapters 48 and 51](51-interfaces-and-design-patterns.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why a database at all?](#2-why-a-database-at-all)
3. [Infrastructure and the `infra` folder](#3-infrastructure-and-the-infra-folder)
4. [How Go talks to databases](#4-how-go-talks-to-databases)
5. [Choosing a library](#5-choosing-a-library)
6. [Running PostgreSQL with Docker](#6-running-postgresql-with-docker)
7. [The connection string (DSN)](#7-the-connection-string-dsn)
8. [`sql.Open` does not connect!](#8-sqlopen-does-not-connect)
9. [Connection pools](#9-connection-pools)
10. [Configuration for the database](#10-configuration-for-the-database)
11. [The `infra/db` package](#11-the-infradb-package)
12. [Startup: fail fast](#12-startup-fail-fast)
13. [Liveness and readiness endpoints](#13-liveness-and-readiness-endpoints)
14. [Tests](#14-tests)
15. [Running it for real](#15-running-it-for-real)
16. [Common mistakes](#16-common-mistakes)
17. [Interview questions](#17-interview-questions)
18. [Exercises](#18-exercises)
19. [Quiz](#19-quiz)
20. [Summary](#20-summary)

---

## 1. What you will learn

- What **infrastructure** means and why we keep it in its own folder
- The architecture of `database/sql`: the **standard interface**, **drivers**, and **pools**
- The trade-offs between `database/sql`, `sqlx`, `pgx`, GORM, and `sqlc`
- How to run a **throwaway PostgreSQL** with Docker
- The anatomy of a **DSN** (`postgres://user:pass@host:5432/db?sslmode=…`) and how to keep passwords **out of logs and error messages**
- Why `sql.Open` is **lazy**, and why you must `Ping` with a **timeout**
- **Pool tuning**: `MaxOpenConns`, `MaxIdleConns`, `ConnMaxLifetime`, and what happens when the pool is exhausted
- **Fail-fast startup**, `/healthz` (liveness) vs. `/readyz` (readiness)
- Writing an **integration test** that is skipped when no database is available

---

## 2. Why a database at all?

Until now products and users lived in **Go slices in memory**. Two big problems:

| Problem | Consequence |
|---------|-------------|
| **Memory is volatile** | Restart the server and every registered user disappears |
| **One process only** | Run two copies behind a load balancer and each has *different* data |

A **database** is a separate program dedicated to storing data **durably** (on disk, surviving crashes), **concurrently** (many clients), and **queryably** (find things fast with indexes), with **transactions** (all-or-nothing changes; Chapter 60).

We use **PostgreSQL** ("Postgres"): a free, open-source relational database, famous for correctness, rich features (JSON, full-text search, extensions), and reliability. A *relational* database stores data in **tables** (rows and columns) and you talk to it in **SQL** (Structured Query Language).

```
                        network (TCP :5432)
 ┌──────────────┐     ┌────────────────────┐     ┌──────────────┐
 │  Go server   │ ──► │  PostgreSQL server │ ──► │  disk files  │
 │ (many        │ ◄── │  (a separate       │ ◄── │ (durable)    │
 │  requests)   │     │   process)         │     └──────────────┘
 └──────────────┘     └────────────────────┘
```

Our Go program is a **client**; Postgres is a **server** that could run on the same machine (development) or a different one (production).

---

## 3. Infrastructure and the `infra` folder

**Infrastructure** = the *external services* your application depends on but doesn't implement itself:

| Service | Purpose | Examples |
|---------|---------|----------|
| Database | durable structured data | PostgreSQL, MySQL |
| Cache | fast temporary data | Redis, Memcached |
| Message queue / stream | async work, events | RabbitMQ, Kafka, NATS |
| Object storage | files, images | S3, MinIO |
| Search | full-text search | Elasticsearch, OpenSearch |

Code that *talks to* these services goes in one place, so the rest of the app doesn't scatter connection details everywhere:

```
ecommerce/
├── cmd/            ← start-up & wiring
├── config/         ← settings
├── infra/
│   └── db/         ← NEW: everything about connecting to PostgreSQL
│       ├── db.go
│       └── db_test.go
├── health/         ← NEW: /healthz and /readyz
├── database/       ← the in-memory stores (replaced by real ones in Chapter 53)
├── product/  user/ ← features (depend on interfaces, Chapter 51)
└── ...
```

Two rules keep this healthy:

1. **`infra/...` packages know nothing about business features.** They create connections and nothing more.
2. **Features never import `infra/...`.** They depend on the small interfaces from Chapter 51; only the composition root (`cmd`) connects the two.

---

## 4. How Go talks to databases

Go's standard library has a package **`database/sql`** that provides a **generic interface**: the same code for PostgreSQL, MySQL, SQLite… The actual protocol lives in a separate, pluggable **driver**.

```
   your code
       │  uses
       ▼
 ┌──────────────┐
 │ database/sql │   standard API: Query, Exec, transactions, POOL of connections
 └──────┬───────┘
        │ driver interface (database/sql/driver)
        ▼
 ┌──────────────┐
 │    driver    │   e.g. pgx (PostgreSQL), go-sql-driver/mysql, go-sqlite3
 └──────┬───────┘
        │ PostgreSQL wire protocol over TCP
        ▼
   PostgreSQL server
```

A driver **registers itself** by name inside its `init()` function (Chapter 14). You import it with a **blank identifier** purely for that side effect:

```go
import _ "github.com/jackc/pgx/v5/stdlib" // registers the driver under the name "pgx"
```

`_` means "import for side effects only; I won't call anything in this package directly". Without the import, `sql.Open("pgx", …)` fails with `unknown driver "pgx" (forgotten import?)`.

Important design fact: **`*sql.DB` is not a single connection. It is a pool of connections**, safe for concurrent use by many goroutines. You create **one** for the whole program and share it.

---

## 5. Choosing a library

| Option | What it is | Pros | Cons |
|--------|------------|------|------|
| **`database/sql` + driver** | Standard library API | No dependencies beyond the driver; full control | Verbose row scanning (`rows.Scan(&a,&b,…)`) |
| **`sqlx`** | Thin extension of `database/sql` | Scan rows straight into structs; named parameters; same `*sql.DB` underneath | Still write SQL by hand |
| **`pgx` (native)** | High-performance PostgreSQL-only toolkit | Fastest; Postgres features (COPY, LISTEN/NOTIFY, batching) | Not portable to other databases |
| **GORM / ent** | ORMs (Object-Relational Mappers) | Little SQL to write | Magic, hidden queries, harder to optimize |
| **`sqlc`** | Generates type-safe Go from your SQL files | Compile-time checked SQL, no runtime magic | Extra build step |

**This course's choice:** `sqlx` on top of **`pgx`'s `database/sql` driver**.

- You **learn SQL and `database/sql`**, skills that transfer to every language and library.
- `sqlx` removes the boilerplate (`db.Select(&products, query)`).
- `pgx` is the actively maintained, modern PostgreSQL driver. (The older `github.com/lib/pq` is in maintenance mode; you'll still see it in older tutorials, and it works the same way.)

Install (run in your project folder):

```bash
go get github.com/jmoiron/sqlx
go get github.com/jackc/pgx/v5
```

> **Pinned `go` line reminder:** as in Chapter 48, adding a dependency may raise the `go` line in `go.mod` (here `pgx` requires a recent Go). That's normal; the toolchain tells you.

---

## 6. Running PostgreSQL with Docker

You *could* install Postgres directly, but **Docker** gives you a clean, disposable database in one command, identical on every machine, and removable without trace.

```bash
docker run -d --name shop-db \
  -e POSTGRES_PASSWORD=devpass \
  -e POSTGRES_DB=ecommerce \
  -p 127.0.0.1:5432:5432 \
  postgres:16-alpine
```

| Piece | Meaning |
|-------|---------|
| `-d` | run in the background (detached) |
| `--name shop-db` | a name so you can refer to it later |
| `-e POSTGRES_PASSWORD=devpass` | password for the default `postgres` superuser (**development only!**) |
| `-e POSTGRES_DB=ecommerce` | create an empty database called `ecommerce` on first start |
| `-p 127.0.0.1:5432:5432` | publish container port 5432 on **localhost only** (`127.0.0.1`), so other machines on your network can't reach your dev database |
| `postgres:16-alpine` | the image (Postgres 16 on the small Alpine Linux) |

> **Port already in use?** If you already run something on `5432`, change the *left* number: `-p 127.0.0.1:15432:5432`, and use `15432` in your connection string. The examples in this chapter use `15432` for exactly that reason.

Check it's ready, and try a query with the bundled `psql` client:

```bash
docker exec shop-db pg_isready -U postgres
docker exec shop-db psql -U postgres -d ecommerce -c "SELECT version();"
```

Real output:

```
/var/run/postgresql:5432 - accepting connections
                                       version
--------------------------------------------------------------------------------------
 PostgreSQL 16.15 on x86_64-pc-linux-musl, compiled by gcc (Alpine 15.2.0) 15.2.0, 64-bit
(1 row)
```

Handy lifecycle commands:

```bash
docker stop shop-db      # stop (data kept inside the container)
docker start shop-db     # start again
docker rm -f shop-db     # delete the container AND its data
```

> Data lives in the container's filesystem. `docker rm` destroys it. That's ideal for learning; for data you want to keep, mount a volume (`-v shop-data:/var/lib/postgresql/data`).

---

## 7. The connection string (DSN)

A **DSN** (Data Source Name) tells the driver where and how to connect. For PostgreSQL the URL form is:

```
postgres://USER:PASSWORD@HOST:PORT/DATABASE?option=value&option=value
└──┬───┘   └─┬┘ └───┬───┘ └─┬┘ └─┬┘ └──┬───┘ └────────┬─────────┘
 scheme    user  password host port  database   query parameters
```

Example for our container:

```
postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable
```

| Part | Notes |
|------|-------|
| `postgres` / `postgresql` | both accepted as the scheme |
| `HOST` | `127.0.0.1` or `localhost` for local; a DNS name in production |
| `sslmode` | `disable` (no TLS: **local development only**), `require`, `verify-full` (encrypted **and** certificate-verified: use in production) |
| Special characters in the password | must be **percent-encoded** (`p@ss/word` → `p%40ss%2Fword`), or the URL breaks |

### Where does it live?

In **configuration** (Chapter 47/48): an environment variable named **`DATABASE_URL`** (the widely used convention). Never hard-code it and never commit real credentials.

### 🔐 Passwords must never leak

The DSN contains a **secret**. It must never appear in logs, error messages, or HTTP responses. We enforce this in two ways: our config validation never echoes the value back (Go's `url.Parse` errors *include the whole URL*!), and `Config.String()` prints only `DatabaseURL:***`. The `pgx` driver already redacts passwords in its own error messages; the test in §14 proves it.

---

## 8. `sql.Open` does not connect!

The single most surprising fact about `database/sql`:

> **`sql.Open` only validates its arguments and creates the pool object. It does *not* connect.** Connections are opened lazily, on first use.

Here is a real experiment: one correct URL, one with the wrong port, and one that is nonsense; **`Open` succeeds for all of them**:

```go
package main

import (
	"context"
	"fmt"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
	"github.com/jmoiron/sqlx"
)

func main() {
	good, err := sqlx.Open("pgx", "postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable")
	fmt.Println("open good:", err)

	dead, err := sqlx.Open("pgx", "postgres://postgres:devpass@127.0.0.1:1/ecommerce?sslmode=disable")
	fmt.Println("open dead:", err) // nil! nothing was contacted

	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	fmt.Println("ping good:", good.PingContext(ctx))
	fmt.Println("ping dead:", dead.PingContext(ctx))
}
```

Real output:

```
open good: <nil>
open dead: <nil>
ping good: <nil>
ping dead: failed to connect to `user=postgres database=ecommerce`: 127.0.0.1:1 (127.0.0.1): dial error: dial tcp 127.0.0.1:1: connect: connection refused
```

Therefore: **after `Open`, always `PingContext` with a timeout** to verify the database is actually reachable *now*, before you start serving traffic. `Ping` borrows a connection from the pool (opening one if needed) and does a cheap round trip.

Note the error text: `failed to connect to user=postgres database=ecommerce` shows the user and database, **never the password**.

---

## 9. Connection pools

Opening a database connection is **expensive**: a TCP handshake, (usually) TLS, and authentication, often 5–50 ms. Doing that per request would be crippling. A **pool** keeps a set of connections open and **lends** them out:

```
  goroutine A ──┐                      ┌─ conn 1 ─┐
  goroutine B ──┼──►  *sql.DB  ───────►├─ conn 2 ─┼──► PostgreSQL
  goroutine C ──┤     (the pool)       ├─ conn 3 ─┤
  goroutine D ──┘                      └──────────┘
                     every query: borrow → run → return
```

Each query (or transaction) **borrows** a connection, runs, and **returns** it. If all are busy, the caller **waits** (blocks) until one is free or its `context` expires.

### The four knobs

| Setting | Method | Meaning | Default |
|---------|--------|---------|---------|
| Max open | `SetMaxOpenConns(n)` | hard cap on connections **in use + idle** | **unlimited** ⚠️ |
| Max idle | `SetMaxIdleConns(n)` | how many idle connections to keep ready | 2 |
| Max lifetime | `SetConnMaxLifetime(d)` | recycle a connection after this age (helps with load balancers, failovers, password rotation) | forever |
| Max idle time | `SetConnMaxIdleTime(d)` | close connections idle longer than this | forever |

**Why care about "unlimited"?** PostgreSQL itself allows only ~100 connections by default (`max_connections`), and each is a whole OS process on the server side. A traffic spike with an unlimited pool opens hundreds of connections and Postgres starts refusing with `too many clients already`. **Always set `MaxOpenConns`.**

A sensible starting point for a single app instance: `MaxOpenConns = 10–25`, `MaxIdleConns = MaxOpenConns/2` (or equal), `ConnMaxLifetime = 30m`. Bigger is *not* better: past a small number more connections just fight for the same CPU and disk. (Multiply by the number of app instances when checking against Postgres's limit.)

### See the pool in action

This experiment fires **40 concurrent queries** that each take 100 ms (`pg_sleep(0.1)`) through pools of different sizes:

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
	"github.com/jmoiron/sqlx"
)

func run(maxOpen int) {
	db, _ := sqlx.Open("pgx", "postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable")
	defer db.Close()
	db.SetMaxOpenConns(maxOpen)

	start := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < 40; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			db.ExecContext(context.Background(), "SELECT pg_sleep(0.1)")
		}()
	}
	wg.Wait()

	s := db.Stats()
	fmt.Printf("MaxOpen=%-2d took %4dms  still_open=%d waited=%d\n",
		maxOpen, time.Since(start).Milliseconds(), s.OpenConnections, s.WaitCount)
}

func main() {
	run(1)
	run(5)
	run(20)
}
```

Real output (times vary slightly per machine):

```
MaxOpen=1  took 4066ms  still_open=1 waited=39
MaxOpen=5  took  816ms  still_open=2 waited=35
MaxOpen=20 took  236ms  still_open=2 waited=20
```

Reading the numbers: with **1** connection the 40 queries run one after another (40 × 0.1 s ≈ 4 s); with **5** they run in 8 waves (≈ 0.8 s); with **20**, in 2 waves (≈ 0.2 s). `WaitCount` is how many callers had to **queue for a connection**. `still_open` is measured *after* the work finished: the pool had opened up to 5 or 20 connections during the burst, but the default `MaxIdleConns` is only **2**, so the surplus were closed as they were returned. That's why idle sizing matters: if it's too low, a bursty app keeps paying to reopen connections. `db.Stats()` is your window into pool health (in production, export it as metrics; a steadily growing `WaitDuration` means the pool is too small or queries are too slow).

Corollary: **a query that hangs holds a connection.** That's why every database call takes a `context.Context` with a deadline (Chapter 51's interfaces already do), and why you must always **close rows** and finish transactions (Chapters 53 and 60), or you "leak" connections until the pool is empty and the whole app freezes.

---

## 10. Configuration for the database

Add four settings to `Config` (Chapter 48's pattern: defaults, validation, all errors reported together):

| Variable | Default | Rule |
|----------|---------|------|
| `DATABASE_URL` | none, **required** | `postgres://` or `postgresql://` URL with a host |
| `DB_MAX_OPEN_CONNS` | `10` | integer ≥ 1 |
| `DB_MAX_IDLE_CONNS` | `5` | integer ≥ 0, and ≤ max open |
| `DB_CONN_MAX_LIFETIME` | `30m` | positive duration |

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

	DatabaseURL       string        // postgres://user:password@host:port/dbname (contains a secret!)
	DBMaxOpenConns    int           // pool: most connections open at once
	DBMaxIdleConns    int           // pool: connections kept ready while idle
	DBConnMaxLifetime time.Duration // pool: recycle connections older than this
}

// String redacts the secrets so a Config can be logged safely.
func (c *Config) String() string {
	return fmt.Sprintf("Config{Env:%s Port:%s Origins:%v LogFormat:%s JWTSecret:*** JWTTTL:%s DatabaseURL:*** DBMaxOpenConns:%d DBMaxIdleConns:%d DBConnMaxLifetime:%s}",
		c.Env, c.Port, c.AllowedOrigins, c.LogFormat, c.JWTTTL, c.DBMaxOpenConns, c.DBMaxIdleConns, c.DBConnMaxLifetime)
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
		JWTSecret:      get("JWT_SECRET", ""),   // required: no default
		DatabaseURL:    get("DATABASE_URL", ""), // required: no default
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

	// The database. NOTE: error messages never include the URL itself, because it contains a password.
	if cfg.DatabaseURL == "" {
		errs = append(errs, errors.New("DATABASE_URL is required, e.g. postgres://user:password@localhost:5432/ecommerce"))
	} else if err := validateDatabaseURL(cfg.DatabaseURL); err != nil {
		errs = append(errs, err)
	}

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

// validateDatabaseURL checks the shape of the URL without ever repeating it:
// url.Parse errors quote the whole input, which would leak the password into logs.
func validateDatabaseURL(raw string) error {
	u, err := url.Parse(raw)
	if err != nil || (u.Scheme != "postgres" && u.Scheme != "postgresql") || u.Hostname() == "" {
		return errors.New("DATABASE_URL must look like postgres://user:password@host:5432/dbname (special characters in the password must be percent-encoded)")
	}
	return nil
}
```

Design notes:

- **Why not echo `%q` of the URL like other errors do?** Because a *malformed* password URL is still a password. Compare `HTTP_PORT` errors (harmless to echo) with `DATABASE_URL` errors (never echo). A test below proves it.
- `DB_MAX_IDLE_CONNS > DB_MAX_OPEN_CONNS` would be silently clamped by `database/sql`; we'd rather tell you your config is contradictory.
- `.env.example` gains the new variables:

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

# REQUIRED. sslmode=disable is for LOCAL development only; use sslmode=verify-full in production.
DATABASE_URL=postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable
# Connection pool (see Chapter 52).
DB_MAX_OPEN_CONNS=10
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME=30m
```

---

## 11. The `infra/db` package

```go
// file: infra/db/db.go
package db

import (
	"context"
	"fmt"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib" // registers the "pgx" driver with database/sql
	"github.com/jmoiron/sqlx"
)

// pingTimeout bounds how long start-up waits for the database to answer.
const pingTimeout = 5 * time.Second

// Options describes how to connect and how the connection pool should behave.
type Options struct {
	URL             string        // postgres://user:password@host:port/dbname
	MaxOpenConns    int           // hard cap on connections
	MaxIdleConns    int           // idle connections kept ready
	ConnMaxLifetime time.Duration // connections older than this are recycled
}

// Connect creates a connection pool and verifies that the database is reachable.
// On failure it returns an error (which never contains the password) and no pool.
// The caller owns the returned pool and must Close it.
func Connect(ctx context.Context, opts Options) (*sqlx.DB, error) {
	conn, err := sqlx.Open("pgx", opts.URL) // lazy: does not contact the server yet
	if err != nil {
		return nil, fmt.Errorf("open database: %w", err)
	}

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

Walkthrough:

1. **`sqlx.Open("pgx", url)`**: creates the pool; validates nothing about reachability.
2. **Pool settings** applied *before* any connection exists.
3. **`context.WithTimeout` + `PingContext`**: the deadline turns "hang forever on a black-holed network" into an error after 5 seconds.
4. **`conn.Close()` on failure**: the pool object has background goroutines; a failed `Connect` must not leak them.
5. **`%w` wrapping** preserves the underlying error for `errors.Is/As` (e.g., a caller can detect `context.DeadlineExceeded`).
6. The function **returns `*sqlx.DB`** (a concrete type), following "accept interfaces, return structs" (Chapter 51). Consumers will declare what they need as small interfaces.

---

## 12. Startup: fail fast

If the database is unreachable at startup, it is **better to crash immediately with a clear message** than to start serving requests that will all fail. A supervisor (Docker restart policy, Kubernetes) will retry.

`main` now opens the pool before the server, and closes it after the server has shut down. We put the work in a `run` function, because `os.Exit` skips deferred calls: with `defer conn.Close()` in `main` itself, calling `os.Exit(1)` afterwards would silently skip the cleanup.

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
	conn, err := db.Connect(context.Background(), db.Options{
		URL:             cfg.DatabaseURL,
		MaxOpenConns:    cfg.DBMaxOpenConns,
		MaxIdleConns:    cfg.DBMaxIdleConns,
		ConnMaxLifetime: cfg.DBConnMaxLifetime,
	})
	if err != nil {
		return err
	}
	defer conn.Close()
	logger.Info("database connected", "max_open", cfg.DBMaxOpenConns, "max_idle", cfg.DBMaxIdleConns)

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

`cmd.Serve` and `buildHandler` receive the pool and hand it onward:

```go
// file: cmd/serve.go
package cmd

import (
	"context"
	"ecommerce/config"
	"errors"
	"fmt"
	"log/slog"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/jmoiron/sqlx"
)

const shutdownTimeout = 10 * time.Second

// Serve starts the HTTP server and blocks until it stops.
// It shuts down gracefully when the process receives SIGINT (Ctrl+C) or SIGTERM.
func Serve(cfg *config.Config, logger *slog.Logger, conn *sqlx.DB) error {
	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           buildHandler(cfg, logger, conn),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      30 * time.Second,
		IdleTimeout:       60 * time.Second,
		ErrorLog:          slog.NewLogLogger(logger.Handler(), slog.LevelError),
	}

	ln, err := net.Listen("tcp", srv.Addr)
	if err != nil {
		return fmt.Errorf("cannot listen on %s: %w", srv.Addr, err)
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
	case err := <-serveErr:
		if errors.Is(err, http.ErrServerClosed) {
			return nil
		}
		return err

	case <-ctx.Done():
		logger.Info("shutting down", "timeout", timeout)

		shutdownCtx, cancel := context.WithTimeout(context.Background(), timeout)
		defer cancel()

		if err := srv.Shutdown(shutdownCtx); err != nil {
			return fmt.Errorf("graceful shutdown failed: %w", err)
		}
		<-serveErr
		logger.Info("server stopped cleanly")
		return nil
	}
}
```

```go
// file: cmd/wire.go
package cmd

import (
	"ecommerce/config"
	"ecommerce/database"
	"ecommerce/health"
	"ecommerce/product"
	"ecommerce/rest"
	"ecommerce/user"
	"log/slog"
	"net/http"

	"github.com/jmoiron/sqlx"
)

// buildHandler is the composition root: it creates every component, hands each one
// exactly the dependencies it needs, and returns the finished HTTP handler.
func buildHandler(cfg *config.Config, logger *slog.Logger, conn *sqlx.DB) http.Handler {
	// storage: still in memory until Chapter 53 swaps in PostgreSQL-backed stores
	productStore := database.NewProductStore(database.SampleProducts()...)
	userStore := database.NewUserStore()

	productHandler := product.NewHandler(productStore, logger)
	userHandler := user.NewHandler(userStore, []byte(cfg.JWTSecret), cfg.JWTTTL, logger)
	healthHandler := health.NewHandler(conn, logger) // *sqlx.DB satisfies health.Pinger

	return rest.NewServer(cfg, logger, productHandler, userHandler, healthHandler).Handler()
}
```

---

## 13. Liveness and readiness endpoints

Orchestrators (Docker, Kubernetes, load balancers) need to ask your app two different questions:

| Probe | Question | On failure | Our path |
|-------|----------|------------|----------|
| **Liveness** | "Is the process alive and not wedged?" | restart it | `GET /healthz` |
| **Readiness** | "Can it serve traffic **right now** (dependencies OK)?" | stop routing traffic to it (don't restart!) | `GET /readyz` |

Liveness must **not** check the database: if Postgres is down, restarting your app won't help and may worsen an outage. Readiness *should* check it.

The `health` package depends only on a **tiny interface** that `*sqlx.DB` (and any test fake) satisfies:

```go
// file: health/handler.go
package health

import (
	"context"
	"ecommerce/util"
	"log/slog"
	"net/http"
	"time"
)

// readyTimeout keeps a readiness probe fast: a slow database counts as "not ready".
const readyTimeout = 2 * time.Second

// Pinger is what the health checks need from the database (Chapter 51: define small interfaces at the point of use).
type Pinger interface {
	PingContext(ctx context.Context) error
}

// Handler serves the health endpoints.
type Handler struct {
	db     Pinger
	logger *slog.Logger
}

// NewHandler creates a Handler.
func NewHandler(db Pinger, logger *slog.Logger) *Handler {
	return &Handler{db: db, logger: logger}
}

// Routes registers the endpoints on mux. They are deliberately public (no authentication).
func (h *Handler) Routes(mux *http.ServeMux) {
	mux.HandleFunc("GET /healthz", h.Live)
	mux.HandleFunc("GET /readyz", h.Ready)
}

// Live answers "is the process running?". It touches no dependencies.
func (h *Handler) Live(w http.ResponseWriter, r *http.Request) {
	util.SendData(w, http.StatusOK, map[string]string{"status": "ok"})
}

// Ready answers "can this instance serve traffic?" by pinging the database.
func (h *Handler) Ready(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), readyTimeout)
	defer cancel()

	if err := h.db.PingContext(ctx); err != nil {
		h.logger.WarnContext(ctx, "readiness check failed", "err", err)
		util.SendError(w, http.StatusServiceUnavailable, "database unavailable")
		return
	}
	util.SendData(w, http.StatusOK, map[string]string{"status": "ready"})
}
```

`503 Service Unavailable` is the right status: "I'm running but can't serve right now". The response says only `database unavailable`; the detailed cause goes to the log (the same rule as Chapter 51).

`rest.Server` learns about the new handler:

```go
// file: rest/server.go
package rest

import (
	"ecommerce/config"
	"ecommerce/health"
	"ecommerce/middleware"
	"ecommerce/product"
	"ecommerce/user"
	"log/slog"
	"net/http"
)

// Server assembles the HTTP application from its already-constructed parts.
type Server struct {
	cfg      *config.Config
	logger   *slog.Logger
	products *product.Handler
	users    *user.Handler
	health   *health.Handler
}

// NewServer connects the feature handlers to the shared configuration and logger.
func NewServer(cfg *config.Config, logger *slog.Logger, products *product.Handler, users *user.Handler, health *health.Handler) *Server {
	return &Server{cfg: cfg, logger: logger, products: products, users: users, health: health}
}

// Handler returns the complete HTTP handler: every feature's routes wrapped in the global middleware.
func (s *Server) Handler() http.Handler {
	mux := http.NewServeMux()

	authn := middleware.Authenticate([]byte(s.cfg.JWTSecret)) // route-level middleware, given to features
	s.products.Routes(mux, authn)
	s.users.Routes(mux, authn)
	s.health.Routes(mux)

	global := middleware.NewStack(
		middleware.RequestID,
		middleware.Logger(s.logger),
		middleware.CORS(s.cfg.AllowedOrigins),
		middleware.Recover(s.logger),
		middleware.JSONErrors,
	)
	return global.Then(mux)
}
```

---

## 14. Tests

### Config tests

The `withSecret` helper becomes `valid` and supplies both required settings; new tests cover the database rules, including that **the password never appears in an error**.

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
		Env:               "development",
		Port:              "8080",
		AllowedOrigins:    []string{"http://localhost:5173"},
		LogFormat:         "text",
		JWTSecret:         goodSecret,
		JWTTTL:            24 * time.Hour,
		DatabaseURL:       goodDBURL,
		DBMaxOpenConns:    10,
		DBMaxIdleConns:    5,
		DBConnMaxLifetime: 30 * time.Minute,
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
		"DB_MAX_OPEN_CONNS":    "25",
		"DB_MAX_IDLE_CONNS":    "25",
		"DB_CONN_MAX_LIFETIME": "1h",
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
	if cfg.DBMaxOpenConns != 25 || cfg.DBMaxIdleConns != 25 || cfg.DBConnMaxLifetime != time.Hour {
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

### Health tests, with a fake `Pinger`

Chapter 51's payoff again: no database needed to test what happens when the database is down or slow.

```go
// file: health/handler_test.go
package health

import (
	"bytes"
	"context"
	"errors"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

type fakePinger struct {
	err   error
	delay time.Duration
}

func (f fakePinger) PingContext(ctx context.Context) error {
	if f.delay > 0 {
		select {
		case <-time.After(f.delay):
		case <-ctx.Done():
			return ctx.Err() // honor the deadline like a real driver does
		}
	}
	return f.err
}

func newMux(p Pinger) (http.Handler, *bytes.Buffer) {
	var logs bytes.Buffer
	mux := http.NewServeMux()
	NewHandler(p, slog.New(slog.NewTextHandler(&logs, nil))).Routes(mux)
	return mux, &logs
}

func get(h http.Handler, path string) *httptest.ResponseRecorder {
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest(http.MethodGet, path, nil))
	return rec
}

func TestLivenessNeverTouchesTheDatabase(t *testing.T) {
	mux, _ := newMux(fakePinger{err: errors.New("database is on fire")})
	rec := get(mux, "/healthz")
	if rec.Code != http.StatusOK {
		t.Fatalf("liveness must not depend on the database, got %d", rec.Code)
	}
}

func TestReadyWhenTheDatabaseAnswers(t *testing.T) {
	mux, _ := newMux(fakePinger{})
	rec := get(mux, "/readyz")
	if rec.Code != http.StatusOK || !strings.Contains(rec.Body.String(), "ready") {
		t.Errorf("got %d %s", rec.Code, rec.Body.String())
	}
}

func TestNotReadyWhenTheDatabaseFails(t *testing.T) {
	mux, logs := newMux(fakePinger{err: errors.New("dial tcp 10.0.0.5:5432: connection refused")})
	rec := get(mux, "/readyz")

	if rec.Code != http.StatusServiceUnavailable {
		t.Fatalf("expected 503, got %d", rec.Code)
	}
	if strings.Contains(rec.Body.String(), "10.0.0.5") {
		t.Errorf("internal address leaked to the client: %s", rec.Body.String())
	}
	if !strings.Contains(logs.String(), "connection refused") {
		t.Errorf("the cause must be logged, got %q", logs.String())
	}
}

func TestSlowDatabaseCountsAsNotReady(t *testing.T) {
	mux, _ := newMux(fakePinger{delay: 5 * time.Second})

	start := time.Now()
	rec := get(mux, "/readyz")

	if rec.Code != http.StatusServiceUnavailable {
		t.Errorf("expected 503, got %d", rec.Code)
	}
	if elapsed := time.Since(start); elapsed > 3*time.Second {
		t.Errorf("the probe must give up after readyTimeout, took %s", elapsed)
	}
}
```

### The `rest` test helper

`rest/app_test.go` only needs the extra constructor argument (a fake that always answers):

```go
// file: rest/app_test.go
package rest

import (
	"context"
	"ecommerce/auth"
	"ecommerce/config"
	"ecommerce/database"
	"ecommerce/health"
	"ecommerce/product"
	"ecommerce/user"
	"encoding/json"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

var quietLogger = slog.New(slog.NewTextHandler(io.Discard, nil))

// alwaysUp is a health.Pinger for a database that always answers.
type alwaysUp struct{}

func (alwaysUp) PingContext(context.Context) error { return nil }

// testApp is a complete, private instance of the application.
type testApp struct {
	t        *testing.T
	cfg      *config.Config
	handler  http.Handler
	products *database.ProductStore
	users    *database.UserStore
}

func newApp(t *testing.T) *testApp {
	t.Helper()
	cfg := &config.Config{
		Env:            "test",
		Port:           "0",
		AllowedOrigins: []string{"http://localhost:5173"},
		LogFormat:      "text",
		JWTSecret:      "0123456789abcdef0123456789abcdef0123",
		JWTTTL:         time.Hour,
	}
	products := database.NewProductStore(database.SampleProducts()...)
	users := database.NewUserStore()

	server := NewServer(cfg, quietLogger,
		product.NewHandler(products, quietLogger),
		user.NewHandler(users, []byte(cfg.JWTSecret), cfg.JWTTTL, quietLogger),
		health.NewHandler(alwaysUp{}, quietLogger),
	)
	return &testApp{t: t, cfg: cfg, handler: server.Handler(), products: products, users: users}
}

func (a *testApp) do(method, target, body string, headers map[string]string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	if body != "" {
		req.Header.Set("Content-Type", "application/json")
	}
	for k, v := range headers {
		req.Header.Set(k, v)
	}
	rec := httptest.NewRecorder()
	a.handler.ServeHTTP(rec, req)
	return rec
}

// bearer returns headers carrying a valid token for the given user ID.
func (a *testApp) bearer(userID int) map[string]string {
	a.t.Helper()
	token, err := auth.CreateToken([]byte(a.cfg.JWTSecret), userID, time.Hour, time.Now())
	if err != nil {
		a.t.Fatal(err)
	}
	return map[string]string{"Authorization": "Bearer " + token}
}

func (a *testApp) count() int {
	var list []struct{ ID int }
	json.NewDecoder(a.do(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	return len(list)
}
```

### The integration test: a real database, when one is available

A **unit test** uses fakes and runs everywhere. An **integration test** uses the real thing, so it needs the database and should be **skipped, not failed**, when none is configured. We key it on an environment variable.

```go
// file: infra/db/db_test.go
package db

import (
	"context"
	"net/url"
	"os"
	"strings"
	"testing"
	"time"
)

// testURL returns the database to test against, or skips the test.
func testURL(t *testing.T) string {
	t.Helper()
	u := os.Getenv("TEST_DATABASE_URL")
	if u == "" {
		t.Skip("set TEST_DATABASE_URL to run database integration tests")
	}
	return u
}

func opts(u string) Options {
	return Options{URL: u, MaxOpenConns: 4, MaxIdleConns: 2, ConnMaxLifetime: time.Minute}
}

func TestConnectAndQuery(t *testing.T) {
	conn, err := Connect(context.Background(), opts(testURL(t)))
	if err != nil {
		t.Fatal(err)
	}
	defer conn.Close()

	var one int
	if err := conn.GetContext(context.Background(), &one, "SELECT 1"); err != nil || one != 1 {
		t.Fatalf("SELECT 1 gave %d, %v", one, err)
	}
}

func TestPoolSettingsAreApplied(t *testing.T) {
	conn, err := Connect(context.Background(), opts(testURL(t)))
	if err != nil {
		t.Fatal(err)
	}
	defer conn.Close()

	if got := conn.Stats().MaxOpenConnections; got != 4 {
		t.Errorf("MaxOpenConnections = %d, want 4", got)
	}
}

func TestWrongPasswordFailsWithoutLeakingIt(t *testing.T) {
	u, err := url.Parse(testURL(t))
	if err != nil || u.User == nil {
		t.Skip("TEST_DATABASE_URL has no user to alter")
	}
	u.User = url.UserPassword(u.User.Username(), "definitely-not-the-password")

	conn, err := Connect(context.Background(), opts(u.String()))
	if err == nil {
		conn.Close()
		t.Fatal("expected authentication to fail")
	}
	if strings.Contains(err.Error(), "definitely-not-the-password") {
		t.Errorf("the error leaked the password: %v", err)
	}
}

// This one needs no database: nothing listens on port 1.
func TestConnectFailsCleanlyWhenUnreachable(t *testing.T) {
	start := time.Now()
	conn, err := Connect(context.Background(), opts("postgres://u:secret-pw@127.0.0.1:1/db?sslmode=disable"))
	if err == nil {
		conn.Close()
		t.Fatal("expected an error")
	}
	if conn != nil {
		t.Error("no pool should be returned on failure")
	}
	if strings.Contains(err.Error(), "secret-pw") {
		t.Errorf("the error leaked the password: %v", err)
	}
	if time.Since(start) > pingTimeout+time.Second {
		t.Errorf("Connect must respect its timeout, took %s", time.Since(start))
	}
}

func TestConnectHonorsACancelledContext(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	cancel()

	conn, err := Connect(ctx, opts("postgres://u:pw@127.0.0.1:1/db?sslmode=disable"))
	if err == nil {
		conn.Close()
		t.Fatal("expected an error for a cancelled context")
	}
}
```

Run everything, including the integration tests against your container:

```bash
go vet ./... && go test -race ./...                       # integration tests are skipped
TEST_DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable' \
  go test -race -v ./infra/db/                            # now they run
```

---

## 15. Running it for real

Copy `.env.example` to `.env` (or export the variables), then start the app:

```bash
go run .
```

Real output:

```
time=2026-09-26T12:31:58.941+06:00 level=INFO msg="database connected" max_open=10 max_idle=5
time=2026-09-26T12:31:58.993+06:00 level=INFO msg="server starting" addr=:18080 env=development
```

Set `HTTP_PORT=18080` if 8080 is busy on your machine (the outputs below used 18080). Try the probes (`-s` silences progress; the output is trimmed to the status line and body):

```bash
curl -si http://localhost:18080/healthz
curl -si http://localhost:18080/readyz
```

```
HTTP/1.1 200 OK
Content-Type: application/json
{"status":"ok"}

HTTP/1.1 200 OK
{"status":"ready"}
```

Now the interesting part: **stop the database while the server runs** (`docker stop shop-db`) and probe again:

```
/healthz  →  HTTP/1.1 200 OK                       {"status":"ok"}
/readyz   →  HTTP/1.1 503 Service Unavailable      {"error":"database unavailable"}
```

The server stays up and **liveness stays 200** (restarting the app wouldn't help), while **readiness turns 503** so a load balancer would stop sending it traffic. Start the database again (`docker start shop-db`) and `/readyz` returns `200` again after a moment, by itself (in our run, on the first probe 3 seconds later): the pool simply opens fresh connections; no restart is required. That resilience is why we use a pool rather than one long-lived connection.

Finally, a **wrong password** at startup:

```
time=2026-09-26T12:32:04.398+06:00 level=ERROR msg="server failed" err="cannot reach database: failed to connect to `user=postgres database=ecommerce`: 127.0.0.1:15432 (127.0.0.1): failed SASL auth: FATAL: password authentication failed for user \"postgres\" (SQLSTATE 28P01)"
$ echo $?
1
```

Clear message, non-zero exit, and **no password in the output**.

---

## 16. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Assuming `sql.Open` connects | App starts "fine", every request fails | `PingContext` right after `Open` |
| 2 | Forgetting the blank driver import | `unknown driver "pgx" (forgotten import?)` | `import _ "github.com/jackc/pgx/v5/stdlib"` |
| 3 | No `SetMaxOpenConns` | Traffic spike exhausts Postgres (`too many clients`) | Always cap the pool |
| 4 | A new `sql.Open` per request | Connection storm; pool benefits lost | One shared `*sql.DB` per database |
| 5 | Logging or echoing the DSN | Password leak | Redact; never include the URL in errors |
| 6 | `os.Exit` right after `defer conn.Close()` in the same function | Deferred cleanup silently skipped | Do the work in a function that returns |
| 7 | `sslmode=disable` in production | Credentials and data cross the network in plain text | `verify-full` with proper certificates |
| 8 | Unescaped special characters in the password | Bad-URL errors or connecting as the wrong user | Percent-encode (`@` → `%40`) |
| 9 | Ping with no timeout | Startup hangs indefinitely on a black-holed network | `context.WithTimeout` |
| 10 | Liveness probe that pings the database | A database outage makes the orchestrator restart-loop healthy apps | Liveness = process only; readiness = dependencies |
| 11 | Exposing the database port publicly (`-p 5432:5432` on a server) | Anyone can attack it | Bind to `127.0.0.1` or a private network |
| 12 | Committing `.env` | Secrets in Git history forever | `.gitignore` + `.env.example` |
| 13 | Integration tests that **fail** without a database | Contributors can't run tests offline | Skip unless `TEST_DATABASE_URL` is set |

---

## 17. Interview questions

**Q1. What does `sql.Open` do?**
It validates the driver name and creates a lazy connection pool. It doesn't connect; use `Ping`.

**Q2. Is `*sql.DB` a connection?**
No. It's a concurrency-safe **pool** of connections; create one per database and share it.

**Q3. Why import a driver with `_`?**
The driver registers itself in `init()`; the blank import triggers that side effect.

**Q4. What can go wrong with an unbounded pool?**
The number of connections can exceed the server's `max_connections`, causing errors and memory pressure. Always set `MaxOpenConns`.

**Q5. Why do we need `ConnMaxLifetime`?**
To recycle connections so that failovers, credential rotation, and idle-connection killers in load balancers don't leave the pool full of dead connections.

**Q6. Liveness vs. readiness?**
Liveness: "is the process healthy?" (failure ⇒ restart). Readiness: "can it serve requests now?" (failure ⇒ remove from rotation). Readiness may check dependencies; liveness should not.

**Q7. How do you keep DB credentials out of logs?**
Store the DSN only in configuration; never include it in error messages or `String()` output; rely on drivers that redact; test it.

**Q8. sqlx vs. GORM vs. sqlc?**
`sqlx`: thin helpers over hand-written SQL. GORM: full ORM that generates SQL. `sqlc`: generates Go from SQL for compile-time safety. Trade-off: control and transparency vs. convenience.

**Q9. Why fail fast at startup if the DB is down?**
A clear crash beats serving errors; supervisors can restart/retry, and the failure is visible immediately.

---

## 18. Exercises

### Exercise 1: Explore with `psql`
Using `docker exec -it shop-db psql -U postgres -d ecommerce`, run `\l` (list databases), `\conninfo`, and `SELECT current_user, current_database();`. What does `\q` do?

<details><summary>Solution</summary>

`\l` lists databases, `\conninfo` shows how you're connected, the `SELECT` prints `postgres | ecommerce`, and `\q` quits `psql`.
</details>

### Exercise 2: Break the DSN on purpose
Try each: wrong port, wrong database name, wrong password, `sslmode=require` against the non-TLS container. Record each error message. Which ones include your password?

<details><summary>Solution</summary>

Wrong port → `connection refused`; wrong database → `database "x" does not exist (SQLSTATE 3D000)`; wrong password → `password authentication failed (SQLSTATE 28P01)`; `sslmode=require` → `server refused TLS connection`. None includes the password, because the driver redacts it.
</details>

### Exercise 3: Watch the pool
Copy the §9 experiment and add `SetMaxIdleConns(0)`. Print `Stats().OpenConnections` and `Idle` after the run. Explain the difference from the default.

<details><summary>Solution</summary>

With `MaxIdleConns(0)` every connection is closed as soon as it's returned, so `Idle` is 0 and later queries must reconnect (slower). The default keeps up to 2 idle connections ready for reuse.
</details>

### Exercise 4: Add a startup retry
In `db.Connect`, retry the ping up to 5 times with a 1-second pause (respecting `ctx`) so the app tolerates starting a few seconds before the database (common with Docker Compose). Which errors should *not* be retried?

<details><summary>Solution</summary>

```go
var err error
for attempt := 1; attempt <= 5; attempt++ {
	pctx, cancel := context.WithTimeout(ctx, pingTimeout)
	err = conn.PingContext(pctx)
	cancel()
	if err == nil {
		return conn, nil
	}
	select {
	case <-time.After(time.Second):
	case <-ctx.Done():
		conn.Close()
		return nil, ctx.Err()
	}
}
conn.Close()
return nil, fmt.Errorf("cannot reach database after 5 attempts: %w", err)
```
Don't retry *permanent* failures such as a wrong password (`SQLSTATE 28P01`) or an unknown database: retrying can't fix them and, for passwords, may trigger lockouts. Retry only connection-level/transient errors.
</details>

### Exercise 5: Prometheus-style stats
Add a `GET /debug/dbstats` endpoint returning `db.Stats()` as JSON (open, in-use, idle, wait count). Should it be public?

<details><summary>Solution</summary>

Give `health.Handler` (or a new handler) a `Stats() sql.DBStats` method source; marshal it. It should **not** be public: it reveals internals. Protect it with the auth middleware, bind it to an internal port, or expose the numbers as metrics (Chapter 68) instead.
</details>

### Exercise 6 (challenge): Environment-aware TLS
Make config reject `sslmode=disable` when `APP_ENV=production`. Write the table test first.

<details><summary>Solution</summary>

In `validateDatabaseURL`, parse `u.Query().Get("sslmode")`; in `fromLookup`, `if cfg.Env == "production" && sslmode is "disable" or "allow" or "prefer" { errs = append(..., errors.New("DATABASE_URL must use sslmode=require or verify-full in production")) }`. Add table cases for `production+disable → error`, `production+verify-full → ok`, `development+disable → ok`.
</details>

---

## 19. Quiz

1. What is a driver, and how does it get registered?
2. Does `sql.Open` check that the database is reachable?
3. What does `SetMaxOpenConns(1)` do to 40 concurrent queries?
4. Why does `Connect` call `conn.Close()` when the ping fails?
5. What HTTP status should a failing readiness probe return?
6. Why must the DSN never appear in an error message?
7. Why should integration tests skip rather than fail without a database?

<details><summary>Answers</summary>

1. The implementation of the wire protocol for one database; it registers itself by name in `init()` via a blank import.
2. No. It's lazy; use `PingContext`.
3. They run strictly one after another (the rest wait for the single connection).
4. The pool has background resources; leaving it open leaks them.
5. `503 Service Unavailable`.
6. It contains the password; logs and responses are widely readable.
7. So anyone can run `go test ./...` offline; CI opts in by setting the variable.
</details>

---

## 20. Summary

- **Infrastructure** (databases, caches, queues) lives in `infra/...`; features depend on small interfaces, and only the composition root connects them.
- Go's **`database/sql`** = standard API + pluggable **driver** (`pgx`) + a built-in **connection pool**; **`sqlx`** adds struct scanning on top. `*sql.DB` is a **pool**, created once and shared.
- **`sql.Open` is lazy**: always `PingContext` with a **timeout**, and **fail fast** at startup.
- **Tune the pool**: cap `MaxOpenConns`, keep some idle, recycle with `ConnMaxLifetime`; watch `db.Stats()`.
- The **DSN is a secret**: keep it in config, redact it in `String()`, never echo it in errors (even `url.Parse` errors quote it!).
- **`/healthz`** (liveness, no dependencies) and **`/readyz`** (readiness, pings the DB with a timeout, `503` on failure) let orchestrators manage the app; failures are logged, not leaked.
- **Integration tests** hit the real database and **skip** when `TEST_DATABASE_URL` is unset; fakes cover the rest.

### ➡️ What's next?

[Chapter 53](53-users-in-postgresql.md) starts **using** the connection: creating tables, writing SQL with `sqlx`, and implementing `product.Store` on top of PostgreSQL.
