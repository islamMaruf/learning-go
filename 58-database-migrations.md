# Chapter 58: Database Migrations — Versioned, Repeatable, Safe Schema Changes

> **Goal of this chapter:** Stop applying SQL files by hand. You'll learn what **migrations** are and why every serious project uses them, write **versioned Up/Down migration files**, embed them in the binary, and run them with **goose** through a small `infra/migrate` package, using a `migrate` command (`up`, `down`, `status`) and an optional `serve -auto-migrate` flag. You'll see **transactional DDL** roll back a half-applied migration, how a **database lock** keeps four app instances from migrating at once, how to **adopt migrations on a database that already has tables**, and the rules for changing a live schema without downtime.

**Difficulty:** 🔴 Advanced  **Estimated time:** 7 hours  **Prerequisite:** [Chapters 52, 56 and 57](57-database-configuration.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The problem with "run this SQL file"](#2-the-problem-with-run-this-sql-file)
3. [What is a migration?](#3-what-is-a-migration)
4. [Choosing a tool](#4-choosing-a-tool)
5. [The migration files](#5-the-migration-files)
6. [The `migrate` package](#6-the-migrate-package)
7. [The command line: `migrate` and `serve`](#7-the-command-line-migrate-and-serve)
8. [Tests](#8-tests)
9. [Watching it work](#9-watching-it-work)
10. [Adopting migrations on an existing database](#10-adopting-migrations-on-an-existing-database)
11. [Rules for changing a live schema](#11-rules-for-changing-a-live-schema)
12. [When should migrations run?](#12-when-should-migrations-run)
13. [Common mistakes](#13-common-mistakes)
14. [Interview questions](#14-interview-questions)
15. [Exercises](#15-exercises)
16. [Quiz](#16-quiz)
17. [Summary](#17-summary)

---

## 1. What you will learn

- What a **migration** is, and how the database records which ones have been applied
- **Up** and **Down** scripts, sequential **version numbers**, and why an applied migration is **immutable**
- `//go:embed` for shipping SQL inside the binary
- The `goose` library: `Provider`, `Up`, `Down`, `Status`, `-- +goose NO TRANSACTION`
- **Transactional DDL** in PostgreSQL: a failed migration leaves no half-built schema
- **Advisory locks**: several instances starting at once, only one migrates
- Building a small CLI with subcommands and the `flag` package
- Zero-downtime changes: **expand → migrate → contract**, `CREATE INDEX CONCURRENTLY`
- Testing migrations, including a full **Up → Down → Up** round trip

---

## 2. The problem with "run this SQL file"

So far, schema changes meant `docker exec … psql < sql/002_create_products.sql`. That works for one developer and two tables. Now imagine a real team:

| Situation | What goes wrong with manual SQL |
|-----------|--------------------------------|
| A teammate pulls your branch that adds a column | Their database doesn't have it; the app crashes with `column "x" does not exist`. Which file do they run? Have they already run it? |
| Staging, production, three developer laptops, CI | Five databases, each at a *different* schema version, nobody sure which |
| You need to undo a bad change | There is no undo; you write reverse SQL at 2 a.m. under pressure |
| Two people add `003_…` at the same time | Conflicting numbers, or silently different orders on different machines |
| A script fails halfway | Half the tables exist; re-running fails with "already exists" |
| A new hire joins | "Run these 14 files in *this* order, skipping #7" |

We need what we already have for code: **version control for the database schema**: an ordered history of changes, a record of which ones each database has applied, and one command that brings *any* database up to date.

---

## 3. What is a migration?

A **migration** is a small, numbered script that moves the schema from version *N−1* to version *N* (**Up**), plus the reverse (**Down**). The database itself stores its current version in a bookkeeping table, so the tool knows which migrations are still **pending**.

```
 migrations/                                      database
 ┌─────────────────────────────────────┐          ┌──────────────────────────────┐
 │ 00001_create_users.sql        Up/Down│          │ goose_db_version             │
 │ 00002_create_products.sql     Up/Down│  ──────► │  1  applied  2026-09-26 10:01│
 │ 00003_index_products.sql      Up/Down│  migrate │  2  applied  2026-09-26 10:01│
 └─────────────────────────────────────┘    up    │  3  applied  2026-09-26 10:01│
                                                  └──────────────────────────────┘
        "what SHOULD exist"                            "what HAS been applied"
```

`migrate up` compares the two lists and applies only the missing ones, **in order**. Run it on an empty database and you get the full schema; run it on a database that's one version behind and you get just the newest change; run it on an up-to-date database and nothing happens. The same command works everywhere. That property, **idempotent convergence to the latest version**, is the whole point.

Terminology:

| Term | Meaning |
|------|---------|
| **Up** | apply the change (create table, add column) |
| **Down** | reverse it (drop table, drop column). Used in development and emergencies |
| **Version** | the migration's number (`1`, `2`, `3`…); the database stores the highest applied |
| **Pending** | in the code but not yet applied to this database |
| **Baseline** | the first migration(s): the schema as it stood when you began using migrations |
| **DDL / DML** | structure statements (`CREATE`, `ALTER`, `DROP`) vs. data statements (`INSERT`, `UPDATE`) |

---

## 4. Choosing a tool

Go has several good migration tools:

| Tool | Notes |
|------|-------|
| **`pressly/goose`** | SQL *and* Go migrations, embeddable, a clean `Provider` API, session locking; used here |
| `golang-migrate` | very popular; separate up/down files (`001_x.up.sql`, `001_x.down.sql`); many database drivers |
| `rubenv/sql-migrate` | similar annotations to goose; simpler |
| Atlas, Flyway, Liquibase | declarative / JVM-based / cross-language tools, common in polyglot companies |

The concepts are identical across all of them. Learn one and the others take an hour. Install the dependency:

```bash
go get github.com/pressly/goose/v3
```

(Like `pgx` in Chapter 52, this may raise the `go` line in `go.mod`; that's expected.)

---

## 5. The migration files

Convention: `NNNNN_description.sql`, five-digit **sequential** numbers, lower-case snake-case description. Each file has two sections marked by `-- +goose` comments (these are *annotations* the tool reads; to PostgreSQL they're just SQL comments).

```sql
-- file: migrations/00001_create_users.sql
-- +goose Up
CREATE TABLE users (
    id            BIGINT      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email         TEXT        NOT NULL CHECK (email <> ''),
    password_hash TEXT        NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Emails are unique regardless of capitalization: Asha@x.com and asha@x.com are the same account.
CREATE UNIQUE INDEX users_email_lower_key ON users (lower(email));

-- +goose Down
DROP TABLE users;
```

```sql
-- file: migrations/00002_create_products.sql
-- +goose Up
CREATE TABLE products (
    id          BIGINT        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title       TEXT          NOT NULL CHECK (char_length(btrim(title)) BETWEEN 1 AND 100),
    description TEXT          NOT NULL DEFAULT '',
    price       NUMERIC(12,2) NOT NULL CHECK (price > 0),
    image_url   TEXT          NOT NULL DEFAULT '',
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT now()
);

-- +goose Down
DROP TABLE products;
```

These are exactly the tables from Chapters 53 and 56, now versioned. A third migration shows how the schema *evolves*. Suppose a dashboard will list "newest products first", so we want an index on `created_at`. Building an index on a big table takes a lock that blocks writes, so PostgreSQL offers `CREATE INDEX CONCURRENTLY`, which builds it **without blocking**, at the price that it **cannot run inside a transaction**. goose needs to be told that:

```sql
-- file: migrations/00003_index_products_created_at.sql
-- +goose NO TRANSACTION
-- +goose Up
CREATE INDEX CONCURRENTLY IF NOT EXISTS products_created_at_idx ON products (created_at);

-- +goose Down
DROP INDEX CONCURRENTLY IF EXISTS products_created_at_idx;
```

By default goose wraps **each migration in a transaction**; this is what makes failures safe (§9). `-- +goose NO TRANSACTION` opts out for statements PostgreSQL refuses to run in one (`CREATE INDEX CONCURRENTLY`, `ALTER TYPE … ADD VALUE` in older versions, `VACUUM`). With no transaction, put **one statement per file** so that a failure can't leave a half-applied mix. `IF NOT EXISTS` / `IF EXISTS` make the statement safe to re-run after an interrupted attempt.

### Embedding

The SQL travels **inside the binary**, so deployment is still one file, and a binary can never run against migrations that don't match it:

```go
// file: migrations/migrations.go
// Package migrations embeds the SQL migration files so they ship inside the binary.
package migrations

import "embed"

// FS contains every NNNNN_name.sql file in this directory.
//
//go:embed *.sql
var FS embed.FS
```

(`//go:embed *.sql` with the directive on its own comment line right above the variable, exactly as in Chapter 53. The doc comment above it is separate.)

---

## 6. The `migrate` package

A thin wrapper hides the third-party library behind our own small API (so the rest of the app doesn't import goose, and swapping tools later is a one-package change):

```go
// file: infra/migrate/migrate.go
// Package migrate applies versioned SQL migrations to PostgreSQL.
package migrate

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"io/fs"
	"time"

	"github.com/pressly/goose/v3"
	"github.com/pressly/goose/v3/lock"
)

// Migrator applies the migrations found in a file system to one database.
type Migrator struct {
	provider *goose.Provider
}

// Migration describes one migration and whether this database has applied it.
type Migration struct {
	Version   int64
	Name      string
	Applied   bool
	AppliedAt time.Time // zero if not applied
}

// New creates a Migrator for db using the *.sql files in fsys.
//
// Every operation takes a PostgreSQL advisory lock first, so several application instances
// starting at the same moment cannot run migrations concurrently: one migrates, the rest wait,
// then find nothing left to do. The lock needs a spare connection, so the pool must allow at least two.
func New(db *sql.DB, fsys fs.FS) (*Migrator, error) {
	// Retry the lock every second (the default is every 5) for up to 5 minutes: another instance may be mid-migration.
	locker, err := lock.NewPostgresSessionLocker(lock.WithLockTimeout(1, 300))
	if err != nil {
		return nil, fmt.Errorf("create migration lock: %w", err)
	}
	p, err := goose.NewProvider(goose.DialectPostgres, db, fsys, goose.WithSessionLocker(locker))
	if err != nil {
		return nil, fmt.Errorf("load migrations: %w", err)
	}
	return &Migrator{provider: p}, nil
}

// Up applies every pending migration, in order, and returns their file names.
// It stops at the first failure; earlier migrations stay applied, and a failed one is rolled back.
func (m *Migrator) Up(ctx context.Context) ([]string, error) {
	results, err := m.provider.Up(ctx)

	// When a migration fails, goose returns a *PartialError that lists the ones that DID succeed before it.
	var partial *goose.PartialError
	if errors.As(err, &partial) {
		results = partial.Applied
	}

	names := make([]string, 0, len(results))
	for _, r := range results {
		names = append(names, r.Source.Path)
	}
	if err != nil {
		return names, fmt.Errorf("migrate up: %w", err)
	}
	return names, nil
}

// Down reverts the most recently applied migration and returns its file name ("" if there was nothing to revert).
func (m *Migrator) Down(ctx context.Context) (string, error) {
	res, err := m.provider.Down(ctx)
	if err != nil {
		return "", fmt.Errorf("migrate down: %w", err)
	}
	return res.Source.Path, nil
}

// DownAll reverts every applied migration, newest first. It destroys data: use it in tests and development only.
func (m *Migrator) DownAll(ctx context.Context) error {
	if _, err := m.provider.DownTo(ctx, 0); err != nil {
		return fmt.Errorf("migrate down to 0: %w", err)
	}
	return nil
}

// Status lists every known migration in version order with its applied state.
func (m *Migrator) Status(ctx context.Context) ([]Migration, error) {
	statuses, err := m.provider.Status(ctx)
	if err != nil {
		return nil, fmt.Errorf("migration status: %w", err)
	}
	out := make([]Migration, 0, len(statuses))
	for _, s := range statuses {
		out = append(out, Migration{
			Version:   s.Source.Version,
			Name:      s.Source.Path,
			Applied:   s.State == goose.StateApplied,
			AppliedAt: s.AppliedAt,
		})
	}
	return out, nil
}

// There is deliberately no Close method: goose's Provider.Close would close the *sql.DB we were given,
// but the caller owns that pool (and other code is still using it).
```

**Ownership note:** the `Migrator` has no `Close` method on purpose. goose's `Provider.Close()` closes the `*sql.DB` it was given; calling it would silently kill the application's shared pool. (A first version of this wrapper did expose `Close`, and the integration tests failed everywhere with `sql: database is closed`: a reminder that whoever *opens* a resource should be the one to close it.)

What goose does inside `Up`, step by step:

1. Takes the **advisory lock** (a PostgreSQL feature: a named, session-level lock any client can request; others requesting it wait).
2. Creates the bookkeeping table **`goose_db_version`** if it doesn't exist.
3. Lists applied versions from that table and the migrations from the file system; **verifies there are no gaps** (an applied version missing from the files is an error) unless told otherwise.
4. For each pending migration, in order: **BEGIN**, run the `Up` statements, **insert a row into `goose_db_version`**, **COMMIT**. (The bookkeeping row is written in the *same* transaction as the schema change: they succeed or fail together.)
5. Releases the lock.

---

## 7. The command line: `migrate` and `serve`

`main` becomes a tiny dispatcher: `go run . migrate up`, `go run . migrate status`, `go run . serve` (the default with no arguments), and `go run . serve -auto-migrate`. The standard **`flag`** package parses the options.

```go
// file: main.go
package main

import (
	"context"
	"ecommerce/cmd"
	"ecommerce/config"
	"ecommerce/infra/db"
	"ecommerce/infra/migrate"
	"ecommerce/migrations"
	"errors"
	"flag"
	"fmt"
	"io"
	"log/slog"
	"os"

	"github.com/jmoiron/sqlx"
)

const usage = `usage:
  ecommerce [serve] [-auto-migrate]   start the HTTP server (optionally applying pending migrations first)
  ecommerce migrate up                apply all pending migrations
  ecommerce migrate down              revert the most recent migration
  ecommerce migrate status            list migrations and whether they are applied
`

func main() {
	cfg, err := config.Load()
	if err != nil {
		fmt.Fprintln(os.Stderr, "configuration error:\n"+err.Error())
		os.Exit(1) // non-zero exit code: a supervisor/CI sees that startup failed
	}

	logger := newLogger(cfg)

	if err := dispatch(cfg, logger, os.Args[1:], os.Stdout); err != nil {
		logger.Error("command failed", "err", err)
		os.Exit(1)
	}
}

// errUsage marks a mistake in the command line (as opposed to a runtime failure).
var errUsage = errors.New("invalid command line")

// dispatch runs the subcommand named by args (default: serve).
func dispatch(cfg *config.Config, logger *slog.Logger, args []string, out io.Writer) error {
	name := "serve"
	if len(args) > 0 && args[0] != "" && args[0][0] != '-' { // "-auto-migrate" alone means "serve -auto-migrate"
		name, args = args[0], args[1:]
	}

	switch name {
	case "serve":
		return serve(cfg, logger, args, out)
	case "migrate":
		return migrateCommand(cfg, logger, args, out)
	default:
		fmt.Fprint(out, usage)
		return fmt.Errorf("%w: unknown command %q", errUsage, name)
	}
}

// connect opens the database pool described by the configuration.
func connect(ctx context.Context, cfg *config.Config, logger *slog.Logger) (*sqlx.DB, error) {
	// Log WHERE we are connecting (never the credentials) BEFORE trying: when the "wrong" database is
	// used, or the connection fails, this line is the first thing to check.
	logger.Info("connecting to database", "target", cfg.DatabaseTarget())

	return db.Connect(ctx, db.Options{
		URL:              cfg.DatabaseURL,
		MaxOpenConns:     cfg.DBMaxOpenConns,
		MaxIdleConns:     cfg.DBMaxIdleConns,
		ConnMaxLifetime:  cfg.DBConnMaxLifetime,
		StatementTimeout: cfg.DBStatementTimeout,
	})
}

// serve connects to the database and serves HTTP until shutdown.
func serve(cfg *config.Config, logger *slog.Logger, args []string, out io.Writer) error {
	fs := flag.NewFlagSet("serve", flag.ContinueOnError)
	fs.SetOutput(out) // flag prints its own error and usage text here
	autoMigrate := fs.Bool("auto-migrate", false, "apply pending migrations before starting")
	if err := fs.Parse(args); err != nil {
		return fmt.Errorf("%w: %v", errUsage, err)
	}

	ctx := context.Background()
	conn, err := connect(ctx, cfg, logger)
	if err != nil {
		return err
	}
	defer conn.Close()

	if *autoMigrate {
		if err := applyMigrations(ctx, conn, logger); err != nil {
			return err
		}
	}

	logger.Info("database ready", "max_open", cfg.DBMaxOpenConns, "statement_timeout", cfg.DBStatementTimeout)
	return cmd.Serve(cfg, logger, conn)
}

// applyMigrations brings the database up to the latest version and logs what it did.
func applyMigrations(ctx context.Context, conn *sqlx.DB, logger *slog.Logger) error {
	m, err := migrate.New(conn.DB, migrations.FS)
	if err != nil {
		return err
	}

	applied, err := m.Up(ctx)
	if err != nil {
		return err
	}
	logger.Info("migrations applied", "count", len(applied), "files", applied)
	return nil
}

// migrateCommand implements "migrate up|down|status".
func migrateCommand(cfg *config.Config, logger *slog.Logger, args []string, out io.Writer) error {
	// Check the command line BEFORE touching the database: a typo must not need a connection to be reported.
	if len(args) != 1 || (args[0] != "up" && args[0] != "down" && args[0] != "status") {
		fmt.Fprint(out, usage)
		return fmt.Errorf("%w: migrate needs exactly one of up, down, status", errUsage)
	}

	ctx := context.Background()
	conn, err := connect(ctx, cfg, logger)
	if err != nil {
		return err
	}
	defer conn.Close()

	m, err := migrate.New(conn.DB, migrations.FS)
	if err != nil {
		return err
	}

	switch args[0] {
	case "up":
		applied, err := m.Up(ctx)
		if err != nil {
			return err
		}
		if len(applied) == 0 {
			fmt.Fprintln(out, "already up to date")
		}
		for _, name := range applied {
			fmt.Fprintln(out, "applied ", name)
		}
	case "down":
		name, err := m.Down(ctx)
		if err != nil {
			return err
		}
		fmt.Fprintln(out, "reverted", name)
	case "status":
		list, err := m.Status(ctx)
		if err != nil {
			return err
		}
		for _, mig := range list {
			state := "pending"
			if mig.Applied {
				state = "applied " + mig.AppliedAt.Local().Format("2006-01-02 15:04:05")
			}
			fmt.Fprintf(out, "%-45s %s\n", mig.Name, state)
		}
	}
	return nil
}

// newLogger builds a logger in the format the configuration asks for.
func newLogger(cfg *config.Config) *slog.Logger {
	if cfg.LogFormat == "json" {
		return slog.New(slog.NewJSONHandler(os.Stdout, nil))
	}
	return slog.New(slog.NewTextHandler(os.Stdout, nil))
}
```

Things worth studying in `main.go`:

- **`flag.NewFlagSet`** gives each subcommand its own flags (`serve -auto-migrate`). `flag.ContinueOnError` returns an error instead of calling `os.Exit`, and `SetOutput(out)` sends its messages where we choose: both keep the function testable.
- **Validate the command line before connecting.** An early draft of this code opened the database first; the test in §8 (`migrate sideways` with no database available) exposed it: a typo produced a confusing connection error instead of the usage text.
- **`dispatch` takes `args` and an `io.Writer`** instead of reading `os.Args`/`os.Stdout` directly. That's the same "inject the dependency" idea as everywhere else, and it makes the command line testable (§8).
- **`connect`** is shared by both subcommands, so the logging of the target and the timeouts are consistent.
- `defer conn.Close()` works because the functions *return* (no `os.Exit` inside them; `main` does that after everything unwound).
- The *migrate* command needs the whole `Config` to load (including `JWT_SECRET`), although it only needs the database settings. Splitting configuration into per-command parts is a good refactoring exercise (Exercise 6).

Compose no longer needs to load the schema through init scripts (the app does that now), so those mounts disappear:

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
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 2s
      timeout: 3s
      retries: 15
    restart: unless-stopped

volumes:
  pgdata:
```

The two hand-run schema files are obsolete; the migrations replace them (the demo data file stays, as it is *data*, not schema):

<!-- delete: sql/001_create_users.sql -->
<!-- delete: sql/002_create_products.sql -->

---

## 8. Tests

### Test-support: an isolated database with the real migrations

Chapter 53's test helper applied SQL files by hand. Now that a `Migrator` exists, the helper can build test databases **exactly the way production does**: which also means every store test quietly verifies the migrations. It moves into a reusable (non-`_test`) package so that several packages' tests can share it, like `net/http/httptest`:

```go
// file: infra/db/dbtest/dbtest.go
// Package dbtest creates isolated PostgreSQL schemas for integration tests.
package dbtest

import (
	"context"
	"ecommerce/infra/db"
	"ecommerce/infra/migrate"
	"ecommerce/migrations"
	"fmt"
	"net/url"
	"os"
	"sync/atomic"
	"testing"
	"time"

	"github.com/jmoiron/sqlx"
)

var schemaCounter atomic.Int64

// DB is a connection pool whose search_path is a private, freshly created schema.
type DB struct {
	*sqlx.DB        // a pool on the isolated schema
	DSN      string // reconnect with this to get a second, independent pool
}

func opts(u string) db.Options {
	return db.Options{URL: u, MaxOpenConns: 8, MaxIdleConns: 4, ConnMaxLifetime: time.Minute}
}

// Empty returns a pool on a brand-new, empty schema; the schema is dropped when the test ends.
// The test is skipped when TEST_DATABASE_URL is not set.
func Empty(t *testing.T) *DB {
	t.Helper()
	base := os.Getenv("TEST_DATABASE_URL")
	if base == "" {
		t.Skip("set TEST_DATABASE_URL to run PostgreSQL integration tests")
	}
	ctx := context.Background()

	schema := fmt.Sprintf("t_%d_%d", os.Getpid(), schemaCounter.Add(1))

	admin, err := db.Connect(ctx, opts(base))
	if err != nil {
		t.Fatal(err)
	}
	if _, err := admin.ExecContext(ctx, "CREATE SCHEMA "+schema); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() {
		admin.ExecContext(ctx, "DROP SCHEMA "+schema+" CASCADE")
		admin.Close()
	})

	u, err := url.Parse(base)
	if err != nil {
		t.Fatal(err)
	}
	q := u.Query()
	q.Set("search_path", schema) // every connection in this pool sees only our schema
	u.RawQuery = q.Encode()

	conn, err := db.Connect(ctx, opts(u.String()))
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { conn.Close() })
	return &DB{DB: conn, DSN: u.String()}
}

// Migrated is like Empty, but with every real migration already applied.
func Migrated(t *testing.T) *DB {
	t.Helper()
	d := Empty(t)

	m, err := migrate.New(d.DB.DB, migrations.FS)
	if err != nil {
		t.Fatal(err)
	}
	if _, err := m.Up(context.Background()); err != nil {
		t.Fatalf("applying migrations: %v", err)
	}
	return d
}

// Reconnect opens a second, independent pool on the same schema.
func (d *DB) Reconnect(t *testing.T) *sqlx.DB {
	t.Helper()
	conn, err := db.Connect(context.Background(), opts(d.DSN))
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { conn.Close() })
	return conn
}
```

The `postgres` package's old helper becomes a thin adapter, so none of the existing store tests change at all:

```go
// file: postgres/testdb_test.go
package postgres

import (
	"ecommerce/infra/db/dbtest"
	"testing"

	"github.com/jmoiron/sqlx"
)

// testDB keeps the names the store tests already use.
type testDB struct {
	*sqlx.DB
	inner *dbtest.DB
}

// newTestDB returns a pool on a fresh schema with all migrations applied.
func newTestDB(t *testing.T) *testDB {
	t.Helper()
	d := dbtest.Migrated(t)
	return &testDB{DB: d.DB, inner: d}
}

// reconnect opens a second, independent pool on the same schema.
func (d *testDB) reconnect(t *testing.T) *sqlx.DB { return d.inner.Reconnect(t) }
```

### Do the files themselves make sense? (no database needed)

A unit test that guards the *conventions*: numbering has no gaps or duplicates, and every file has both directions:

```go
// file: migrations/migrations_test.go
package migrations

import (
	"io/fs"
	"regexp"
	"strconv"
	"strings"
	"testing"
)

var nameRE = regexp.MustCompile(`^(\d{5})_[a-z0-9_]+\.sql$`)

func TestFilesFollowTheConventions(t *testing.T) {
	entries, err := fs.ReadDir(FS, ".")
	if err != nil || len(entries) == 0 {
		t.Fatalf("no migrations embedded (%v)", err)
	}

	for i, e := range entries { // ReadDir returns the entries sorted by name
		m := nameRE.FindStringSubmatch(e.Name())
		if m == nil {
			t.Errorf("%s: name must look like 00001_short_description.sql", e.Name())
			continue
		}
		if n, _ := strconv.Atoi(m[1]); n != i+1 {
			t.Errorf("%s: expected version %05d (versions must be sequential with no gaps)", e.Name(), i+1)
		}

		body, _ := fs.ReadFile(FS, e.Name())
		text := string(body)
		up, down := strings.Index(text, "-- +goose Up"), strings.Index(text, "-- +goose Down")
		if up < 0 || down < 0 || up > down {
			t.Errorf("%s: needs \"-- +goose Up\" followed by \"-- +goose Down\"", e.Name())
		}
	}
}
```

### Do they work? (integration tests, real PostgreSQL)

These use an **external test package** (`migrate_test`), which lets them import `dbtest` (which itself imports `migrate`) without an import cycle:

```go
// file: infra/migrate/migrate_test.go
package migrate_test

import (
	"context"
	"ecommerce/infra/db/dbtest"
	"ecommerce/infra/migrate"
	"ecommerce/migrations"
	"sync"
	"testing"
	"testing/fstest"

	"github.com/jmoiron/sqlx"
)

var ctx = context.Background()

func newMigrator(t *testing.T, d *dbtest.DB) *migrate.Migrator {
	t.Helper()
	m, err := migrate.New(d.DB.DB, migrations.FS)
	if err != nil {
		t.Fatal(err)
	}
	return m
}

// tables lists the tables in the test's private schema (which includes goose's bookkeeping table).
func tables(t *testing.T, db *sqlx.DB) map[string]bool {
	t.Helper()
	var names []string
	if err := db.Select(&names, "SELECT table_name FROM information_schema.tables WHERE table_schema = current_schema()"); err != nil {
		t.Fatal(err)
	}
	out := map[string]bool{}
	for _, n := range names {
		out[n] = true
	}
	return out
}

func TestUpBuildsTheWholeSchemaAndIsIdempotent(t *testing.T) {
	d := dbtest.Empty(t)
	m := newMigrator(t, d)

	applied, err := m.Up(ctx)
	if err != nil {
		t.Fatal(err)
	}
	if len(applied) != 3 {
		t.Errorf("a new database should apply all 3 migrations, applied %v", applied)
	}
	if got := tables(t, d.DB); !got["users"] || !got["products"] {
		t.Errorf("expected users and products, got %v", got)
	}

	again, err := m.Up(ctx)
	if err != nil || len(again) != 0 {
		t.Errorf("a second Up must do nothing, got %v (%v)", again, err)
	}
}

func TestStatusReflectsProgress(t *testing.T) {
	d := dbtest.Empty(t)
	m := newMigrator(t, d)

	before, err := m.Status(ctx)
	if err != nil || len(before) != 3 {
		t.Fatalf("status = %v (%v)", before, err)
	}
	for _, s := range before {
		if s.Applied {
			t.Errorf("nothing should be applied yet: %+v", s)
		}
	}

	m.Up(ctx)
	after, _ := m.Status(ctx)
	for _, s := range after {
		if !s.Applied || s.AppliedAt.IsZero() {
			t.Errorf("everything should be applied now: %+v", s)
		}
	}
}

func TestDownRevertsOnlyTheNewestMigration(t *testing.T) {
	d := dbtest.Empty(t)
	m := newMigrator(t, d)
	m.Up(ctx)

	name, err := m.Down(ctx)
	if err != nil || name != "00003_index_products_created_at.sql" {
		t.Fatalf("Down reverted %q (%v)", name, err)
	}
	if got := tables(t, d.DB); !got["users"] || !got["products"] {
		t.Errorf("only the index should be gone, tables = %v", got)
	}

	var indexes int
	d.DB.Get(&indexes, "SELECT count(*) FROM pg_indexes WHERE schemaname = current_schema() AND indexname = 'products_created_at_idx'")
	if indexes != 0 {
		t.Error("the index should have been dropped by the Down section")
	}

	// and it can be applied again
	if again, err := m.Up(ctx); err != nil || len(again) != 1 {
		t.Errorf("re-applying: %v (%v)", again, err)
	}
}

func TestFullRoundTripUpDownUp(t *testing.T) {
	d := dbtest.Empty(t)
	m := newMigrator(t, d)

	if _, err := m.Up(ctx); err != nil {
		t.Fatal(err)
	}
	if err := m.DownAll(ctx); err != nil {
		t.Fatal(err)
	}
	if got := tables(t, d.DB); got["users"] || got["products"] {
		t.Errorf("Down must remove everything it created, got %v", got)
	}
	if _, err := m.Up(ctx); err != nil {
		t.Errorf("Up after a full Down must work: %v", err)
	}
}

func TestConcurrentInstancesApplyEachMigrationOnce(t *testing.T) {
	d := dbtest.Empty(t)

	// Four "application instances" start at the same moment, each with its own Migrator.
	const instances = 4
	var wg sync.WaitGroup
	errs := make(chan error, instances)
	for i := 0; i < instances; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			m, err := migrate.New(d.DB.DB, migrations.FS)
			if err != nil {
				errs <- err
				return
			}
			_, err = m.Up(ctx)
			errs <- err
		}()
	}
	wg.Wait()
	close(errs)
	for err := range errs {
		if err != nil {
			t.Errorf("an instance failed to migrate: %v", err)
		}
	}

	var applied int
	d.DB.Get(&applied, "SELECT count(*) FROM goose_db_version WHERE is_applied AND version_id > 0")
	if applied != 3 {
		t.Errorf("each migration must be recorded exactly once, got %d rows", applied)
	}
}

func TestAFailedMigrationIsRolledBackCompletely(t *testing.T) {
	d := dbtest.Empty(t)

	broken := fstest.MapFS{
		"00001_good.sql": {Data: []byte("-- +goose Up\nCREATE TABLE fine (id int);\n-- +goose Down\nDROP TABLE fine;\n")},
		// creates a table, THEN fails: PostgreSQL DDL is transactional, so the table must not survive
		"00002_bad.sql": {Data: []byte("-- +goose Up\nCREATE TABLE half_done (id int);\nSELECT * FROM table_that_does_not_exist;\n-- +goose Down\nDROP TABLE half_done;\n")},
	}
	m, err := migrate.New(d.DB.DB, broken)
	if err != nil {
		t.Fatal(err)
	}

	applied, err := m.Up(ctx)
	if err == nil {
		t.Fatal("the broken migration should have failed")
	}
	if len(applied) != 1 {
		t.Errorf("only the first migration should have been applied, got %v", applied)
	}

	got := tables(t, d.DB)
	if !got["fine"] {
		t.Error("the earlier, successful migration must stay applied")
	}
	if got["half_done"] {
		t.Error("a failed migration must leave nothing behind (transactional DDL)")
	}

	status, _ := m.Status(ctx)
	if !status[0].Applied || status[1].Applied {
		t.Errorf("version 2 must still be pending: %+v", status)
	}
}
```

### The command line

`dispatch` was written to be testable. No database is needed to test the parts that don't touch one:

```go
// file: main_test.go
package main

import (
	"bytes"
	"ecommerce/config"
	"errors"
	"io"
	"log/slog"
	"strings"
	"testing"
)

func TestUnknownCommandsPrintUsageAndFail(t *testing.T) {
	logger := slog.New(slog.NewTextHandler(io.Discard, nil))
	cfg := &config.Config{}

	for _, args := range [][]string{
		{"frobnicate"},
		{"migrate"},                // needs an action
		{"migrate", "sideways"},    // unknown action
		{"migrate", "up", "extra"}, // too many arguments
		{"serve", "-no-such-flag"},
	} {
		var out bytes.Buffer
		err := dispatch(cfg, logger, args, &out)
		if !errors.Is(err, errUsage) {
			t.Errorf("%v: expected a usage error, got %v", args, err)
		}
		if len(args) > 0 && args[0] != "serve" && !strings.Contains(out.String(), "usage:") {
			t.Errorf("%v: the usage text should have been printed, got %q", args, out.String())
		}
	}
}
```

Run all of it (the second command runs the integration tests against your database):

```bash
go vet ./... && go test -race ./...
TEST_DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable' go test -race -count=1 ./...
```

Look at the last two tests: **`TestConcurrentInstancesApplyEachMigrationOnce`** proves the lock works (four simultaneous `Up`s, three rows in the version table, no errors), and **`TestAFailedMigrationIsRolledBackCompletely`** proves the safety property: a migration that creates a table *and then fails* leaves **no trace**, because PostgreSQL DDL is transactional (MySQL, for example, cannot roll back `CREATE TABLE`, which is one reason migration tooling has more ceremony there).

---

## 9. Watching it work

Start from an **empty** database (a fresh container, or `DROP SCHEMA public CASCADE; CREATE SCHEMA public;` on a scratch one) and use the command line. (Log lines are shown without their `time=…` prefix; the `connecting to database` line comes from the `connect` helper.)

```bash
go run . migrate status
```
```
level=INFO msg="connecting to database" target=127.0.0.1:15432/ecommerce
00001_create_users.sql                        pending
00002_create_products.sql                     pending
00003_index_products_created_at.sql           pending
```

All three are pending. Apply them:

```bash
go run . migrate up
```
```
level=INFO msg="connecting to database" target=127.0.0.1:15432/ecommerce
applied  00001_create_users.sql
applied  00002_create_products.sql
applied  00003_index_products_created_at.sql
```

Run it again:

```bash
go run . migrate up
go run . migrate status
```
```
level=INFO msg="connecting to database" target=127.0.0.1:15432/ecommerce
already up to date
00001_create_users.sql                        applied 2026-09-26 16:17:03
00002_create_products.sql                     applied 2026-09-26 16:17:03
00003_index_products_created_at.sql           applied 2026-09-26 16:17:03
```

Nothing to do the second time: **idempotent**. The bookkeeping table is plain SQL you can read:

```bash
docker exec shop-db psql -U postgres -d ecommerce -c "SELECT id, version_id, is_applied, tstamp FROM goose_db_version ORDER BY id"
```
```
 id | version_id | is_applied |           tstamp
----+------------+------------+----------------------------
  1 |          0 | t          | 2026-09-26 10:17:03.420594
  2 |          1 | t          | 2026-09-26 10:17:03.441283
  3 |          2 | t          | 2026-09-26 10:17:03.449024
  4 |          3 | t          | 2026-09-26 10:17:03.457346
(4 rows)
```

Row `version_id = 0` is goose's own "database initialized" marker. Now undo the newest and re-apply it:

```bash
go run . migrate down
go run . migrate status
go run . migrate up
```
```
reverted 00003_index_products_created_at.sql
00001_create_users.sql                        applied 2026-09-26 16:17:03
00002_create_products.sql                     applied 2026-09-26 16:17:03
00003_index_products_created_at.sql           pending
```

And the combined workflow: **migrate as the server starts**:

```bash
go run . serve -auto-migrate
```
```
level=INFO msg="connecting to database" target=127.0.0.1:15432/ecommerce
level=INFO msg="migrations applied" count=1 files=[00003_index_products_created_at.sql]
level=INFO msg="database ready" max_open=10 statement_timeout=10s
level=INFO msg="server starting" addr=:18080 env=development
```

A misspelled command prints the usage and exits non-zero:

```bash
go run . migrate sideways
```
```
usage:
  ecommerce [serve] [-auto-migrate]   start the HTTP server (optionally applying pending migrations first)
  ecommerce migrate up                apply all pending migrations
  ecommerce migrate down              revert the most recent migration
  ecommerce migrate status            list migrations and whether they are applied
level=ERROR msg="command failed" err="invalid command line: migrate needs exactly one of up, down, status"
```

---

## 10. Adopting migrations on an existing database

You likely have tables already: the ones you created by hand in Chapters 53 and 56. Run `migrate up` there and the baseline migration collides with reality:

```
level=ERROR msg="command failed" err="migrate up: partial migration error (type:sql,version:1): ERROR: relation \"users\" already exists (SQLSTATE 42P07)"
```

`00001` tries to `CREATE TABLE users`, which already exists, so it fails (and, being transactional, changes nothing). Two ways out:

**A. Development database with nothing precious: start clean.**

```bash
docker exec shop-db psql -U postgres -d ecommerce -c 'DROP TABLE products, users'
go run . migrate up
```

**B. A database with real data (the general technique, called *baselining*):** tell the tool "versions 1 and 2 are already there" without running them, then let it apply anything newer. Because the baseline files describe the schema *exactly as it exists*, this is safe:

```bash
docker exec shop-db psql -U postgres -d ecommerce -c \
  "INSERT INTO goose_db_version (version_id, is_applied) VALUES (1, true), (2, true)"
go run . migrate status
go run . migrate up
```
```
INSERT 0 2
00001_create_users.sql                        applied 2026-09-26 16:17:15
00002_create_products.sql                     applied 2026-09-26 16:17:15
00003_index_products_created_at.sql           pending
applied  00003_index_products_created_at.sql
```

Only the genuinely new migration (`00003`) ran. Before baselining a production database, **diff the real schema against what the baseline migration would create** (`pg_dump --schema-only` compared with the files); if they differ, the "already applied" claim is false and future migrations will misbehave.

---

## 11. Rules for changing a live schema

Schema changes on a database that serves traffic are where outages are born. The rules experienced teams follow:

### 1. Never edit a migration that has been applied anywhere shared

An applied migration is **history**. If you change `00002` after production ran it, production and new databases silently diverge. Fix mistakes with a **new** migration (`00004_fix_…`). The only exception is a migration that has never left your own laptop.

### 2. One purpose per migration; keep them small

Small migrations are easy to review, quick to run, and simple to revert. Several unrelated changes in one file means a failure blocks everything.

### 3. Prefer forward-only in production; treat `Down` as a development tool

A `Down` that drops a column *destroys that column's data*. In production the usual "undo" is a **new forward migration** (and restoring from backup for disasters). Still write `Down` scripts: they document intent, power the Up→Down→Up test, and speed up development.

### 4. Make changes backward-compatible: expand → migrate → contract

During a deployment, **old and new versions of the app run at the same time** against the same database. A change that breaks the old version causes errors mid-deploy. Split risky changes into phases across releases:

```
Goal: rename products.title to products.name

 Release 1 (expand):   ADD COLUMN name;  app writes BOTH title and name, reads title
 Release 2 (migrate):  backfill name = title for old rows (in batches);  app reads name
 Release 3 (contract): stop writing title;  later: DROP COLUMN title
```

A one-shot `ALTER TABLE … RENAME COLUMN` would break every running old instance the instant it commits.

### 5. Know which statements are dangerous on big tables

| Statement | Risk | Safer approach |
|-----------|------|----------------|
| `CREATE INDEX` | blocks writes while building | `CREATE INDEX CONCURRENTLY` (no transaction) |
| `ALTER TABLE … ADD COLUMN … DEFAULT <constant>` | since PostgreSQL 11 it's instant | fine; but a *volatile* default (`now()`, `random()`) rewrites the table |
| `ALTER TABLE … ADD COLUMN … NOT NULL` without default | fails if rows exist | add nullable → backfill → `SET NOT NULL` |
| `ALTER TABLE … ALTER COLUMN TYPE` | rewrites the whole table under a lock | add new column, copy, swap |
| Adding a `FOREIGN KEY` / `CHECK` | validates every row under a lock | `ADD CONSTRAINT … NOT VALID`, then `VALIDATE CONSTRAINT` (lighter lock) |
| Long `UPDATE` backfill in one statement | long lock, huge transaction | batches of a few thousand rows |
| Any DDL waiting behind a long-running query | *it* blocks every query queued behind it | set `lock_timeout` so the migration gives up rather than stalls the app |

### 6. Set a `lock_timeout` for risky changes

```sql
-- +goose Up
SET LOCAL lock_timeout = '3s';
ALTER TABLE products ADD COLUMN category TEXT NOT NULL DEFAULT 'general';
```

If the table is busy and the lock can't be had within 3 seconds, the migration **fails fast** (and can be retried) instead of queueing up and freezing all traffic to that table.

### 7. Separate schema from data

Changes to *structure* and *bulk data changes* belong in separate migrations. Data migrations that are large should be batched jobs, not one statement.

### 8. Test them

Run migrations in CI against a real PostgreSQL (like `dbtest.Migrated`), including **Up → Down → Up**. Better still, restore a copy of production (anonymized) and rehearse the migration there to learn how long it takes.

---

## 12. When should migrations run?

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **Manual / pipeline step** | a deployment job runs `ecommerce migrate up` *before* the new version starts | explicit; the app can run with a least-privilege role; a failed migration blocks the deploy | one more thing to orchestrate |
| **On startup** (`serve -auto-migrate`) | each instance migrates first; the advisory lock serializes them | zero orchestration; great for development and small apps | the app's DB user needs DDL rights; slow migration delays startup and can fail the health check; every instance races for the lock |
| **Separate job/container** (Kubernetes Job, init container) | a one-shot process | clean separation of privileges | more infrastructure |

Our recommendation: **`-auto-migrate` in development** and small deployments; a **separate `migrate up` step** in serious production (which also lets the running app use the least-privilege role from Chapter 57 with no DDL rights).

---

## 13. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Editing an already-applied migration | Environments silently diverge | Add a new migration |
| 2 | No `Down` section | Can't revert; round-trip test impossible | Always write both |
| 3 | Version-number clashes between branches | Confusing/misordered history | Rebase and renumber before merging (or use timestamp versions) |
| 4 | `CREATE INDEX` (not `CONCURRENTLY`) on a large live table | Writes blocked during the build | `CONCURRENTLY` + `NO TRANSACTION` |
| 5 | Several statements in a `NO TRANSACTION` migration | A failure leaves a half-applied change | One statement per file, idempotent (`IF NOT EXISTS`) |
| 6 | Dropping/renaming a column the running app still uses | Errors during deploys | Expand → migrate → contract |
| 7 | Data backfill in one giant `UPDATE` | Long locks, bloat | Batches |
| 8 | Forgetting a migration exists in the SQL folder but isn't embedded | Tests pass, prod misses it | `//go:embed *.sql` + the conventions test |
| 9 | Running `migrate down` in production casually | Data destroyed | Forward-only; restore from backup |
| 10 | Running migrations from many instances without a lock | Duplicate/conflicting DDL | Advisory lock (goose `WithSessionLocker`) |
| 11 | A pool of size 1 with the session locker | Deadlock: the lock holds the only connection | Allow ≥ 2 connections |
| 12 | Giving the app superuser rights so it can auto-migrate | An injection bug becomes a total compromise | Migrations as a separate, privileged step |
| 13 | Never testing migrations from an *old* version | "Works on empty DB, fails on production" | Test upgrading a database restored from production |
| 14 | Baselining without comparing schemas | Baseline lies about reality | Diff `pg_dump --schema-only` first |

---

## 14. Interview questions

**Q1. What is a database migration?**
A versioned, ordered script that changes the schema from one version to the next, with a reverse; a bookkeeping table records which versions a database has applied.

**Q2. Why never edit an applied migration?**
Databases that already ran it won't rerun it, so they'd differ from ones created later. History must be immutable; fix forward with a new migration.

**Q3. What does "transactional DDL" give you?**
PostgreSQL runs schema changes inside transactions, so a failing migration is fully rolled back: no half-created tables.

**Q4. How do you prevent two instances from migrating simultaneously?**
A database-level lock (PostgreSQL advisory lock); goose's session locker makes one instance run migrations while the others wait, then find nothing pending.

**Q5. Explain expand → migrate → contract.**
A zero-downtime pattern: first add the new structure (compatible with old code), then backfill and switch the app over, and only then remove the old structure.

**Q6. Why `CREATE INDEX CONCURRENTLY`, and what's the catch?**
It builds the index without blocking writes, but can't run inside a transaction, so the migration must opt out of the transaction (and stay idempotent).

**Q7. How do you introduce migrations to an existing database?**
Write the baseline migration to match the current schema, verify by diffing, then record it as applied without executing it (or start from a clean database in development).

**Q8. Should production run `Down` migrations?**
Rarely: they often destroy data. Prefer a forward migration or a restore from backup.

**Q9. Why embed migrations in the binary?**
The deployed code and its expected schema can't get out of sync; no separate files to ship.

---

## 15. Exercises

### Exercise 1: Add a migration
Add `category TEXT NOT NULL DEFAULT 'general'` to `products` as `00004`. Write Up and Down, run `migrate up`, and check the column with `\d products`. What does the `conventions` test catch if you name it `00005_…` by mistake?

<details><summary>Solution</summary>

```sql
-- +goose Up
ALTER TABLE products ADD COLUMN category TEXT NOT NULL DEFAULT 'general';

-- +goose Down
ALTER TABLE products DROP COLUMN category;
```
Naming it `00005_…` (skipping `00004`) fails `TestFilesFollowTheConventions` ("expected version 00004"), and the round-trip integration tests would also change their expected count of 3.
</details>

### Exercise 2: Break one on purpose
Add a migration whose Up contains a valid `CREATE TABLE` followed by a syntax error. Run `migrate up`. What state is the database in? What does `migrate status` say?

<details><summary>Solution</summary>

The whole migration is rolled back (no table), earlier ones stay applied, and status shows the broken one as `pending`. Fix the file (it never succeeded anywhere, so editing it is allowed) and run `up` again.
</details>

### Exercise 3: Expand → migrate → contract
Plan (as three migrations plus code changes) renaming `products.title` to `products.name` with zero downtime. Write the SQL for each phase, including a batched backfill.

<details><summary>Solution</summary>

1. `ALTER TABLE products ADD COLUMN name TEXT;` (app now writes both columns).
2. Batched backfill, repeated until 0 rows: `UPDATE products SET name = title WHERE id IN (SELECT id FROM products WHERE name IS NULL LIMIT 5000);` then `ALTER TABLE products ALTER COLUMN name SET NOT NULL` (after adding `CHECK (name IS NOT NULL) NOT VALID` + `VALIDATE` if the table is huge). App reads `name`.
3. After every instance uses only `name`: `ALTER TABLE products DROP COLUMN title;`.
</details>

### Exercise 4: Add a lock timeout
Modify `00004` to fail fast if it can't get its lock in 2 seconds. Demonstrate it: hold a lock in one `psql` session (`BEGIN; LOCK TABLE products;`) and run the migration in another.

<details><summary>Solution</summary>

Add `SET LOCAL lock_timeout = '2s';` as the first statement of the Up section. While the other session holds the lock, `migrate up` fails after ~2 s with `canceling statement due to lock timeout (SQLSTATE 55P03)` instead of hanging (and blocking everyone else queued behind it).
</details>

### Exercise 5: A Go migration
Write migration `00005` as a *Go* migration (goose supports them) that lower-cases all existing user emails. Why would you use Go rather than SQL here, and when is SQL (`UPDATE users SET email = lower(email)`) simply better?

<details><summary>Solution</summary>

Go migrations (registered with `goose.WithGoMigrations` / `goose.NewGoMigration`) suit logic SQL can't express (calling a hashing function, an external system, complex per-row transformation). For a lower-casing, plain SQL is better: it's declarative, faster, and needs no code. Prefer SQL unless you can't.
</details>

### Exercise 6: Split the configuration
`migrate` needs only the database settings, but `config.Load` demands `JWT_SECRET` too. Refactor: `config.LoadDatabase()` returns just the database part; `serve` uses the full `Load`. What tests do you add?

<details><summary>Solution</summary>

Extract the database block into `loadDatabase(lookup)` returning a `DatabaseConfig` struct that `Config` embeds; `LoadDatabase` calls `godotenv.Load()` then `loadDatabase(os.LookupEnv)`. Tests: `LoadDatabase` succeeds with only database variables; `Load` still fails without `JWT_SECRET`; and every existing database rule (production TLS, redaction) applies to both.
</details>

### Exercise 7 (challenge): Detect drift
Write a test that compares the schema produced by `Migrated` against a checked-in `schema.golden.sql` (from `pg_dump --schema-only`), so an accidental edit to an applied migration is caught in CI.

<details><summary>Solution</summary>

In an integration test, run `pg_dump --schema-only --no-owner -n <schema>` (via `exec.Command`) against the test schema, normalize the schema name and comments, and compare with the golden file; provide a `-update` flag to regenerate it deliberately. Any unreviewed change to old migrations now shows up as a diff.
</details>

---

## 16. Quiz

1. What does `migrate up` do on a database that is already up to date?
2. Why is an applied migration immutable?
3. Which PostgreSQL feature makes a failed migration leave no half-built tables?
4. Why must `CREATE INDEX CONCURRENTLY` use `-- +goose NO TRANSACTION`?
5. What stops four instances from migrating at the same time?
6. What is baselining?
7. Why is `Down` risky in production?
8. What are the three phases of a zero-downtime rename?

<details><summary>Answers</summary>

1. Nothing; it reports "already up to date".
2. Other databases have already run it; editing it would make environments diverge. Fix forward with a new migration.
3. Transactional DDL.
4. PostgreSQL refuses to run it inside a transaction block.
5. A database advisory lock taken by the session locker.
6. Marking migrations as already applied on a database whose schema already matches them.
7. It can drop columns/tables and destroy data.
8. Expand (add new, write both), migrate (backfill, switch reads), contract (drop old).
</details>

---

## 17. Summary

- A **migration** is a numbered Up/Down script; the database's bookkeeping table (`goose_db_version`) records what's applied, so **`migrate up` brings any database to the latest version** and does nothing if it already is.
- We embed `migrations/*.sql` with `//go:embed`, wrap **goose** behind our own `infra/migrate` package, and expose `migrate up|down|status` and `serve -auto-migrate` through a small, testable `dispatch` built on `flag`.
- PostgreSQL's **transactional DDL** makes failed migrations all-or-nothing; goose writes the bookkeeping row in the same transaction; an **advisory lock** serializes concurrent instances. All three are proven by tests.
- The `dbtest` helper builds **isolated schemas with the real migrations**, so every store test also exercises them; a convention test guards numbering and Up/Down markers.
- **Rules:** never edit applied migrations; keep them small; write `Down` but prefer forward fixes in production; **expand → migrate → contract** for zero-downtime; `CONCURRENTLY` and `lock_timeout` for big tables; baseline carefully.
- Run migrations **automatically in development**, as an **explicit privileged step** in production.

### ➡️ What's next?

**Part 12: architecture.** [Chapter 59](59-domain-driven-design.md) introduces **Domain-Driven Design**: organizing code around the business domain (users, products, orders) so the codebase stays understandable as it grows, and the following chapters reshape our `user` and `product` packages accordingly.
