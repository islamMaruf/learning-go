# Chapter 53: Users in PostgreSQL — Tables, `INSERT`, `SELECT`, and a Real Store

> **Goal of this chapter:** Make user accounts **permanent**. You'll write your first SQL (`CREATE TABLE`, constraints, indexes, `INSERT … RETURNING`, `SELECT … WHERE`), learn **why you must never build SQL with string concatenation** (with a live SQL-injection demonstration), and implement `user.Store` (the interface from Chapter 51) on PostgreSQL with `sqlx`: embedded `.sql` files, `db:` struct tags, translating database errors (`sql.ErrNoRows`, unique violations) into the app's sentinel errors, and a **shared contract test** that proves the in-memory and PostgreSQL stores behave identically.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 6 hours  **Prerequisite:** [Chapters 51 and 52](52-connecting-to-postgresql.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Tables, rows, and columns](#2-tables-rows-and-columns)
3. [Designing the `users` table](#3-designing-the-users-table)
4. [Creating the table](#4-creating-the-table)
5. [INSERT and SELECT by hand](#5-insert-and-select-by-hand)
6. [SQL injection: why we never concatenate](#6-sql-injection-why-we-never-concatenate)
7. [Where to keep SQL: embedded files](#7-where-to-keep-sql-embedded-files)
8. [Reading rows into structs: `db` tags](#8-reading-rows-into-structs-db-tags)
9. [Translating database errors](#9-translating-database-errors)
10. [The PostgreSQL `UserStore`](#10-the-postgresql-userstore)
11. [One test suite for every implementation](#11-one-test-suite-for-every-implementation)
12. [Wiring it in and trying it](#12-wiring-it-in-and-trying-it)
13. [Common mistakes](#13-common-mistakes)
14. [Interview questions](#14-interview-questions)
15. [Exercises](#15-exercises)
16. [Quiz](#16-quiz)
17. [Summary](#17-summary)

---

## 1. What you will learn

- The vocabulary of **relational databases**: table, row, column, primary key, constraint, index
- Reading and writing **SQL**: `CREATE TABLE`, `INSERT … RETURNING`, `SELECT … WHERE`
- **Parameterized queries** (`$1`, `$2`) and why they defeat **SQL injection**
- `//go:embed` to compile `.sql` files into the binary
- `sqlx`: `GetContext`, `ExecContext`, struct scanning with `db` tags
- Mapping **driver errors → domain errors** (`sql.ErrNoRows` → `models.ErrNotFound`, SQLSTATE `23505` → `models.ErrEmailTaken`)
- **Contract tests**: one suite run against several implementations of the same interface
- **Schema-per-test** isolation for integration tests

---

## 2. Tables, rows, and columns

A relational database stores data in **tables**, like strictly typed spreadsheets:

```
 users  (table)
 ┌────┬──────────────────┬───────────────┬───────────────────────────┐
 │ id │ email            │ password_hash │ created_at                │  ← columns (each has a TYPE)
 ├────┼──────────────────┼───────────────┼───────────────────────────┤
 │  1 │ asha@example.com │ $2a$10$N9qo…  │ 2026-09-26 06:32:10+00    │  ← row (one user)
 │  2 │ ravi@example.com │ $2a$10$eImi…  │ 2026-09-26 06:35:44+00    │  ← row
 └────┴──────────────────┴───────────────┴───────────────────────────┘
```

| Term | Meaning |
|------|---------|
| **Table** | a collection of rows with the same shape (like a Go slice of one struct type) |
| **Row** | one record (one struct value) |
| **Column** | one named, typed field |
| **Primary key** | the column(s) that uniquely identify a row |
| **Constraint** | a rule the database *enforces* (not null, unique, …) |
| **Index** | a lookup structure that makes searches fast |
| **Schema** | (1) the definition of your tables; (2) also a namespace inside a database (we'll use this meaning for tests) |

The mapping to Go is direct: **table ≈ struct type, row ≈ struct value, column ≈ field**.

**Why constraints in the database when Go already validates?** Because the database is the **last line of defense**. Bugs, a second app, a manual `UPDATE` at 3 a.m., or a race between two requests can all bypass your Go code. A `UNIQUE` constraint can't be raced past; a Go `if emailExists()` check can (two requests check at the same instant, both see "no", both insert). **Enforce invariants where they can't be bypassed.**

---

## 3. Designing the `users` table

```sql
-- file: sql/001_create_users.sql
CREATE TABLE users (
    id            BIGINT      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email         TEXT        NOT NULL CHECK (email <> ''),
    password_hash TEXT        NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Emails are unique regardless of capitalization: Asha@x.com and asha@x.com are the same account.
CREATE UNIQUE INDEX users_email_lower_key ON users (lower(email));
```

Line by line:

| Piece | Meaning |
|-------|---------|
| `id BIGINT` | a 64-bit integer (≈ Go's `int64`; on 64-bit machines Go's `int` is the same size) |
| `GENERATED ALWAYS AS IDENTITY` | the database assigns 1, 2, 3, … automatically; you may not supply your own. (The older spelling is `SERIAL`; identity columns are the modern, standard-compliant way.) |
| `PRIMARY KEY` | means **unique + not null**, and creates an index on `id` |
| `email TEXT NOT NULL` | text of any length that must be present. (In PostgreSQL `TEXT` and `VARCHAR(n)` perform identically; use `TEXT` unless you need a hard length limit.) |
| `CHECK (email <> '')` | a custom rule: must not be an empty string (`<>` means "not equal") |
| `password_hash TEXT NOT NULL` | the bcrypt hash from Chapter 48, never the password itself |
| `created_at TIMESTAMPTZ` | a timestamp **with time zone**: stored as an absolute instant (UTC internally). Always prefer it over plain `TIMESTAMP`. |
| `DEFAULT now()` | if an insert omits the column, the database fills in the current time |
| `CREATE UNIQUE INDEX … (lower(email))` | an **expression index**: uniqueness is checked on the *lowercased* email. This is the actual guarantee behind "one account per email" |

> **Column naming:** SQL convention is `snake_case` (`password_hash`), Go's is `CamelCase` (`PasswordHash`). Struct tags (§8) bridge them.

> **Why not store `password_hash` as `BYTEA`?** bcrypt hashes are printable ASCII (`$2a$10$…`), so `TEXT` is simplest. Chapter 54 explores the type system properly.

---

## 4. Creating the table

Save the block above as `sql/001_create_users.sql` and run it with `psql` inside the container from Chapter 52 (`-i` passes our file in through standard input):

```bash
docker exec -i shop-db psql -U postgres -d ecommerce -v ON_ERROR_STOP=1 < sql/001_create_users.sql
```

Real output:

```
CREATE TABLE
CREATE INDEX
```

Inspect what you made (`\d` = "describe"):

```bash
docker exec shop-db psql -U postgres -d ecommerce -c '\d users'
```

```
                                   Table "public.users"
    Column     |           Type           | Collation | Nullable |           Default
---------------+--------------------------+-----------+----------+------------------------------
 id            | bigint                   |           | not null | generated always as identity
 email         | text                     |           | not null |
 password_hash | text                     |           | not null |
 created_at    | timestamp with time zone |           | not null | now()
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_email_lower_key" UNIQUE, btree (lower(email))
Check constraints:
    "users_email_check" CHECK (email <> ''::text)
```

Run it a second time and you get `ERROR:  relation "users" already exists`, because SQL files like this aren't repeatable. **Migrations** (Chapter 58) solve that properly by tracking which changes were applied. For now, this manual step is enough to learn the queries.

---

## 5. INSERT and SELECT by hand

Try the two statements you'll wrap in Go, straight in `psql`:

```bash
docker exec -i shop-db psql -U postgres -d ecommerce <<'SQL'
INSERT INTO users (email, password_hash)
VALUES ('asha@example.com', 'not-a-real-hash')
RETURNING id, email, created_at;

SELECT id, email FROM users WHERE email = 'asha@example.com';

-- the unique index at work (note the different capitalization):
INSERT INTO users (email, password_hash) VALUES ('ASHA@example.com', 'x');
SQL
```

Real output:

```
 id |      email       |          created_at
----+------------------+-------------------------------
  1 | asha@example.com | 2026-09-26 06:36:20.637012+00
(1 row)

INSERT 0 1
 id |      email       
----+------------------
  1 | asha@example.com
(1 row)

ERROR:  duplicate key value violates unique constraint "users_email_lower_key"
DETAIL:  Key (lower(email))=(asha@example.com) already exists.
```

Three things to notice:

1. **`INSERT … RETURNING`** hands back the row the database actually stored (with the generated `id` and `created_at`) in the same round trip. Without it you'd need a second query (and would have to guess the ID). It's the idiomatic way to get generated values in PostgreSQL.
2. `INSERT 0 1` is the *command tag*: it inserted one row.
3. The **duplicate** was rejected by the database itself, with a machine-readable failure we can detect from Go (§9).

Clean up before the tests (`TRUNCATE` empties the table; `RESTART IDENTITY` resets the counter):

```bash
docker exec shop-db psql -U postgres -d ecommerce -c 'TRUNCATE users RESTART IDENTITY'
```

---

## 6. SQL injection: why we never concatenate

The most important security lesson in database programming. Suppose a beginner writes login like this, building the SQL **by pasting user input into a string**:

```go
// ⚠️ VULNERABLE: never write this
query := "SELECT id, email FROM users WHERE email = '" + email + "'"
```

For a normal email it works. But the input is **attacker-controlled**. What if the "email" is:

```
' OR '1'='1
```

The pasted-together SQL becomes:

```sql
SELECT id, email FROM users WHERE email = '' OR '1'='1'
```

`'1'='1'` is always true, so the `WHERE` matches **every row**. Here is a real experiment against our table (with two users inserted):

```go
package main

import (
	"context"
	"fmt"

	_ "github.com/jackc/pgx/v5/stdlib"
	"github.com/jmoiron/sqlx"
)

func main() {
	db, _ := sqlx.Open("pgx", "postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable")
	defer db.Close()
	ctx := context.Background()

	attack := "' OR '1'='1"

	// ❌ Vulnerable: input pasted into the SQL text
	bad := "SELECT email FROM users WHERE email = '" + attack + "'"
	var leaked []string
	err := db.SelectContext(ctx, &leaked, bad)
	fmt.Println("query sent:", bad)
	fmt.Println("vulnerable returned:", leaked, err)

	// ✅ Safe: the input travels separately, as data
	var safe []string
	err = db.SelectContext(ctx, &safe, "SELECT email FROM users WHERE email = $1", attack)
	fmt.Println("parameterized returned:", safe, err)
}
```

Real output:

```
query sent: SELECT email FROM users WHERE email = '' OR '1'='1'
vulnerable returned: [asha@example.com ravi@example.com] <nil>
parameterized returned: [] <nil>
```

The vulnerable query **leaked every user**. Worse variants add `; DROP TABLE users; --` (delete the table) or `UNION SELECT` (read other tables). This attack class has topped vulnerability lists for over two decades.

### The fix: parameters (placeholders)

```go
db.GetContext(ctx, &u, "SELECT ... WHERE email = $1", email)
```

`$1`, `$2`, … are **placeholders**. The SQL text and the values are sent to the server **separately**: the database parses the SQL *first*, with holes, and the values only ever fill those holes *as data*. They can never be interpreted as SQL, whatever they contain. In the experiment, the whole string `' OR '1'='1` was looked up as an (unusual) email address, and matched nothing.

Rules:

- **Values** (anything a user or another system can influence) → always **placeholders**.
- Placeholders work only for *values*, not for identifiers (table or column names) or keywords. If you ever need a dynamic column name (say, sorting by a user-chosen field: Chapter 63), validate it against a **fixed allow-list** in Go and only then insert it into the SQL text.
- `fmt.Sprintf` into SQL is never acceptable for values. Ever.

Placeholder syntax varies by database: PostgreSQL uses `$1, $2`; MySQL and SQLite use `?`.

---

## 7. Where to keep SQL: embedded files

SQL as long Go strings is awkward: no syntax highlighting, no way to paste it into `psql` to test, and diffs are noisy. Go's **`embed`** package lets you keep queries in real `.sql` files and compile them into the binary (so deployment is still one file):

```sql
-- file: postgres/queries/users_insert.sql
INSERT INTO users (email, password_hash)
VALUES ($1, $2)
RETURNING id, email, password_hash, created_at
```

```sql
-- file: postgres/queries/users_by_email.sql
SELECT id, email, password_hash, created_at
FROM users
WHERE lower(email) = lower($1)
```

```sql
-- file: postgres/queries/users_by_id.sql
SELECT id, email, password_hash, created_at
FROM users
WHERE id = $1
```

```go
import _ "embed" // required for //go:embed on string variables

//go:embed queries/users_insert.sql
var insertUserSQL string // the file's contents, baked in at compile time
```

How `//go:embed` works:

- It's a **compiler directive** (a comment beginning with `//go:` and no space after `//`). It must sit directly above a package-level `var`.
- The variable can be a `string`, `[]byte`, or an `embed.FS` for many files.
- Paths are **relative to the Go file** and can't go up (`..`) or point outside the package directory: that's why the SQL lives inside the `postgres` package folder.
- A missing file is a **compile error**, not a runtime surprise.

`SELECT *` is avoided on purpose: name your columns. `SELECT *` breaks silently (or wastefully) when someone adds a column, and can leak future sensitive columns.

---

## 8. Reading rows into structs: `db` tags

`database/sql` makes you scan column by column: `rows.Scan(&u.ID, &u.Email, …)`, in exactly the right order. `sqlx` matches **column names to struct fields** instead, using **struct tags** (the same mechanism as `json:"…"` in Chapters 21 and 41):

```go
// file: models/user.go
package models

import (
	"strings"
	"time"
)

// User is a registered account.
type User struct {
	ID           int       `json:"id"        db:"id"`
	Email        string    `json:"email"     db:"email"`
	PasswordHash string    `json:"-"         db:"password_hash"` // `json:"-"`: never serialized to clients
	CreatedAt    time.Time `json:"createdAt" db:"created_at"`
}

// NormalizeEmail is the canonical form of an email address: trimmed and lowercased.
// Every store uses it, so all implementations agree on what "the same email" means.
func NormalizeEmail(email string) string {
	return strings.ToLower(strings.TrimSpace(email))
}
```

Rules for `db` tags:

- `db:"column_name"` maps that column to the field. Without a tag, `sqlx` lower-cases the field name (`PasswordHash` → `passwordhash`, which does **not** match `password_hash`: hence the tags).
- Fields must be **exported** (capitalized), or `sqlx` cannot set them.
- A column with **no matching field** is an error (`missing destination name …`), which is *good*: the query and struct can't drift apart silently. (Opposite: struct fields with no column are just left at their zero value.)
- `db:"-"` ignores a field.

**A design trade-off.** Putting `db` and `json` tags on the same `models.User` is pragmatic and common, but it couples the model to two representations. Larger systems keep a private `userRow` struct with `db` tags inside the storage package and convert; that's worth it when the DB shape and the API shape start to diverge (Chapter 60 revisits this).

---

## 9. Translating database errors

The rest of the app speaks in **domain errors** (`models.ErrNotFound`, `models.ErrEmailTaken`), which the handlers turn into 404 and 409. The storage layer's job is to **translate**: convert driver-specific failures into those sentinels, and wrap everything else with context.

### "No rows"

`GetContext` returns **`sql.ErrNoRows`** when the query matched nothing:

```go
err := db.GetContext(ctx, &u, queryByEmail, email)
if errors.Is(err, sql.ErrNoRows) {
	return models.User{}, models.ErrNotFound
}
```

### Unique violation

PostgreSQL reports failures with a five-character **SQLSTATE** code. `23505` = `unique_violation`. The `pgx` driver returns them as `*pgconn.PgError`, which we extract with `errors.As` (Chapter 41):

```go
import "github.com/jackc/pgx/v5/pgconn"

func isUniqueViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23505"
}
```

Common SQLSTATEs worth knowing:

| Code | Name | Typical cause |
|------|------|---------------|
| `23505` | unique_violation | duplicate email |
| `23503` | foreign_key_violation | referencing a row that doesn't exist |
| `23502` | not_null_violation | missing required column |
| `23514` | check_violation | a `CHECK` rule failed |
| `40001` | serialization_failure | transaction conflict; retry (Chapter 60) |
| `57014` | query_canceled | context cancelled / statement timeout |

**Why detect the error instead of checking first ("does this email exist?")?** Check-then-insert has a **race** (two requests can both pass the check). Insert-and-handle-the-violation is atomic: the database decides. That's what the unique index is for.

**Never return raw driver errors to handlers.** If `product.Handler` had to import `pgconn` to interpret errors, swapping databases would ripple through the whole app. The sentinel errors keep the storage technology private to the storage package.

---

## 10. The PostgreSQL `UserStore`

```go
// file: postgres/user_store.go
package postgres

import (
	"context"
	"database/sql"
	"ecommerce/models"
	_ "embed" // needed for //go:embed
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5/pgconn"
	"github.com/jmoiron/sqlx"
)

//go:embed queries/users_insert.sql
var insertUserSQL string

//go:embed queries/users_by_email.sql
var userByEmailSQL string

//go:embed queries/users_by_id.sql
var userByIDSQL string

// UserStore keeps users in PostgreSQL. It is safe for concurrent use: *sqlx.DB is a connection pool.
type UserStore struct {
	db *sqlx.DB
}

// NewUserStore creates a store on top of an existing connection pool (see infra/db).
func NewUserStore(db *sqlx.DB) *UserStore { return &UserStore{db: db} }

// Create stores a new user, or returns models.ErrEmailTaken if the email is already registered.
func (s *UserStore) Create(ctx context.Context, email, passwordHash string) (models.User, error) {
	var u models.User
	err := s.db.GetContext(ctx, &u, insertUserSQL, models.NormalizeEmail(email), passwordHash)
	if err != nil {
		if isUniqueViolation(err) {
			return models.User{}, models.ErrEmailTaken
		}
		return models.User{}, fmt.Errorf("insert user: %w", err)
	}
	u.CreatedAt = u.CreatedAt.UTC()
	return u, nil
}

// FindByEmail looks a user up by (case-insensitive) email, or returns models.ErrNotFound.
func (s *UserStore) FindByEmail(ctx context.Context, email string) (models.User, error) {
	return s.find(ctx, "find user by email", userByEmailSQL, models.NormalizeEmail(email))
}

// FindByID looks a user up by ID, or returns models.ErrNotFound.
func (s *UserStore) FindByID(ctx context.Context, id int) (models.User, error) {
	return s.find(ctx, "find user by id", userByIDSQL, id)
}

// find runs a single-row user query and translates "no rows" into models.ErrNotFound.
func (s *UserStore) find(ctx context.Context, what, query string, arg any) (models.User, error) {
	var u models.User
	if err := s.db.GetContext(ctx, &u, query, arg); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return models.User{}, models.ErrNotFound
		}
		return models.User{}, fmt.Errorf("%s: %w", what, err)
	}
	u.CreatedAt = u.CreatedAt.UTC()
	return u, nil
}

// isUniqueViolation reports whether err is PostgreSQL's unique_violation (SQLSTATE 23505).
func isUniqueViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23505"
}
```

What to notice:

- **`GetContext`** = "run this query, expect **exactly one row**, scan it into a struct". `SelectContext` is the many-rows version (Chapter 55–56); `ExecContext` is for statements that return no rows.
- **Every call takes `ctx`**: if the HTTP client disconnects or a deadline passes, the query is cancelled and the connection freed.
- The store **normalizes emails itself** (`models.NormalizeEmail`), so callers can't forget. The lowercase in `users_by_email.sql` (`lower(email) = lower($1)`) is belt-and-braces and matches the expression index.
- **Errors are wrapped with `%w`** and a short description (`insert user: …`), so logs say *what* was being attempted while `errors.Is/As` still see the original.
- **`.UTC()`**: the driver returns times in the machine's local zone; normalizing to UTC makes results identical across machines and implementations.
- The type has **no reference to the `user` package**. It satisfies `user.Store` implicitly (Chapter 51). A compile-time assertion lives in the *test* file so production code stays decoupled.

### The in-memory store, aligned

The in-memory store stays (it's the fast fake used by the handler tests). It now shares `models.NormalizeEmail`, so both implementations define "same email" identically:

```go
// file: database/user_store.go
package database

import (
	"context"
	"ecommerce/models"
	"sync"
	"time"
)

// UserStore keeps users in memory. It is safe for concurrent use.
// Production uses the PostgreSQL store; this one backs fast tests and demos.
type UserStore struct {
	mu     sync.RWMutex
	users  []models.User
	nextID int
}

// NewUserStore creates an empty user store.
func NewUserStore() *UserStore { return &UserStore{nextID: 1} }

// Create stores a new user, or returns models.ErrEmailTaken. The duplicate check and the insert
// share one lock, so two simultaneous registrations of the same email cannot both succeed.
func (s *UserStore) Create(_ context.Context, email, passwordHash string) (models.User, error) {
	email = models.NormalizeEmail(email)

	s.mu.Lock()
	defer s.mu.Unlock()

	for _, u := range s.users {
		if u.Email == email {
			return models.User{}, models.ErrEmailTaken
		}
	}
	u := models.User{ID: s.nextID, Email: email, PasswordHash: passwordHash, CreatedAt: time.Now().UTC()}
	s.nextID++
	s.users = append(s.users, u)
	return u, nil
}

// FindByEmail looks a user up by (case-insensitive) email, or returns models.ErrNotFound.
func (s *UserStore) FindByEmail(_ context.Context, email string) (models.User, error) {
	email = models.NormalizeEmail(email)

	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, u := range s.users {
		if u.Email == email {
			return u, nil
		}
	}
	return models.User{}, models.ErrNotFound
}

// FindByID looks a user up by ID, or returns models.ErrNotFound.
func (s *UserStore) FindByID(_ context.Context, id int) (models.User, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, u := range s.users {
		if u.ID == id {
			return u, nil
		}
	}
	return models.User{}, models.ErrNotFound
}
```

---

## 11. One test suite for every implementation

We now have two implementations of `user.Store`. How do we know they *behave the same*? Write the behavior once, as a **contract test**, and run it against each implementation:

```go
// file: user/storetest/storetest.go
// Package storetest holds a behavior suite that every user.Store implementation must pass.
package storetest

import (
	"context"
	"ecommerce/models"
	"ecommerce/user"
	"errors"
	"sync"
	"testing"
	"time"
)

// Factory returns a fresh, empty store for one test.
type Factory func(t *testing.T) user.Store

// Run executes the contract against stores made by newStore.
func Run(t *testing.T, newStore Factory) {
	ctx := context.Background()

	t.Run("create returns the stored user", func(t *testing.T) {
		s := newStore(t)
		u, err := s.Create(ctx, "  Asha@Example.COM ", "hash-1")
		if err != nil {
			t.Fatal(err)
		}
		if u.ID < 1 {
			t.Errorf("expected a generated ID, got %d", u.ID)
		}
		if u.Email != "asha@example.com" {
			t.Errorf("email must be normalized, got %q", u.Email)
		}
		if u.PasswordHash != "hash-1" {
			t.Errorf("hash = %q", u.PasswordHash)
		}
		if age := time.Since(u.CreatedAt); age < -time.Second || age > time.Minute {
			t.Errorf("CreatedAt %v is not about now", u.CreatedAt)
		}
		if u.CreatedAt.Location() != time.UTC {
			t.Errorf("CreatedAt must be in UTC, got %v", u.CreatedAt.Location())
		}
	})

	t.Run("find by email ignores case and spaces", func(t *testing.T) {
		s := newStore(t)
		want, _ := s.Create(ctx, "asha@example.com", "h")

		got, err := s.FindByEmail(ctx, "  ASHA@Example.com")
		if err != nil {
			t.Fatal(err)
		}
		if got.ID != want.ID || got.PasswordHash != "h" {
			t.Errorf("got %+v, want %+v", got, want)
		}
	})

	t.Run("find by id", func(t *testing.T) {
		s := newStore(t)
		want, _ := s.Create(ctx, "asha@example.com", "h")

		got, err := s.FindByID(ctx, want.ID)
		if err != nil || got.Email != "asha@example.com" {
			t.Errorf("got %+v, %v", got, err)
		}
	})

	t.Run("missing users are ErrNotFound", func(t *testing.T) {
		s := newStore(t)
		if _, err := s.FindByEmail(ctx, "ghost@example.com"); !errors.Is(err, models.ErrNotFound) {
			t.Errorf("FindByEmail: got %v", err)
		}
		if _, err := s.FindByID(ctx, 12345); !errors.Is(err, models.ErrNotFound) {
			t.Errorf("FindByID: got %v", err)
		}
	})

	t.Run("duplicate emails are ErrEmailTaken", func(t *testing.T) {
		s := newStore(t)
		if _, err := s.Create(ctx, "asha@example.com", "h"); err != nil {
			t.Fatal(err)
		}
		for _, again := range []string{"asha@example.com", "ASHA@EXAMPLE.COM", "  Asha@Example.com  "} {
			if _, err := s.Create(ctx, again, "h2"); !errors.Is(err, models.ErrEmailTaken) {
				t.Errorf("Create(%q): got %v, want ErrEmailTaken", again, err)
			}
		}
	})

	t.Run("IDs are distinct", func(t *testing.T) {
		s := newStore(t)
		a, _ := s.Create(ctx, "a@example.com", "h")
		b, _ := s.Create(ctx, "b@example.com", "h")
		if a.ID == b.ID || b.ID < a.ID {
			t.Errorf("IDs must be distinct and increasing: %d then %d", a.ID, b.ID)
		}
	})

	t.Run("hostile input is stored as plain data", func(t *testing.T) {
		s := newStore(t)
		nasty := "x'); drop table users; --@example.com" // already lower-case, so it round-trips unchanged
		if _, err := s.Create(ctx, nasty, "h"); err != nil {
			t.Fatal(err)
		}
		got, err := s.FindByEmail(ctx, nasty)
		if err != nil || got.Email != nasty {
			t.Errorf("got %+v, %v", got, err)
		}
		if _, err := s.Create(ctx, "after@example.com", "h"); err != nil {
			t.Errorf("the store must still work afterwards: %v", err)
		}
	})

	t.Run("concurrent duplicate registrations: exactly one wins", func(t *testing.T) {
		s := newStore(t)

		const attempts = 30
		var wg sync.WaitGroup
		var mu sync.Mutex
		wins := 0

		for i := 0; i < attempts; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()
				_, err := s.Create(ctx, "race@example.com", "h")
				switch {
				case err == nil:
					mu.Lock()
					wins++
					mu.Unlock()
				case !errors.Is(err, models.ErrEmailTaken):
					t.Errorf("unexpected error: %v", err)
				}
			}()
		}
		wg.Wait()

		if wins != 1 {
			t.Fatalf("exactly one registration must succeed, got %d", wins)
		}
	})
}
```

Run it against the in-memory implementation:

```go
// file: database/user_store_test.go
package database

import (
	"ecommerce/user"
	"ecommerce/user/storetest"
	"testing"
)

var _ user.Store = (*UserStore)(nil)

func TestUserStoreContract(t *testing.T) {
	storetest.Run(t, func(t *testing.T) user.Store { return NewUserStore() })
}
```

The old user tests in `database/stores_test.go` are now covered by the contract, so that file keeps only the product tests:

```go
// file: database/stores_test.go
package database

import (
	"context"
	"ecommerce/models"
	"errors"
	"sync"
	"testing"
)

var ctx = context.Background()

func TestProductStoreIDsAreUniqueAndSequential(t *testing.T) {
	s := NewProductStore()
	const n = 100

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			s.Create(ctx, SampleProducts()[0])
		}()
	}
	wg.Wait()

	list, _ := s.List(ctx)
	if len(list) != n {
		t.Fatalf("expected %d products, got %d", n, len(list))
	}
	seen := map[int]bool{}
	for _, p := range list {
		if p.ID < 1 || p.ID > n || seen[p.ID] {
			t.Fatalf("bad or duplicate ID %d", p.ID)
		}
		seen[p.ID] = true
	}
}

func TestProductGetReportsNotFound(t *testing.T) {
	s := NewProductStore(SampleProducts()...)
	if _, err := s.Get(ctx, 2); err != nil {
		t.Errorf("product 2 exists: %v", err)
	}
	if _, err := s.Get(ctx, 99); !errors.Is(err, models.ErrNotFound) {
		t.Errorf("expected ErrNotFound, got %v", err)
	}
}

func TestListReturnsACopy(t *testing.T) {
	s := NewProductStore(SampleProducts()...)
	list, _ := s.List(ctx)
	list[0].Title = "tampered"

	if got, _ := s.Get(ctx, 1); got.Title == "tampered" {
		t.Error("callers must not be able to modify the store's internal slice")
	}
}
```

### Isolated databases for the PostgreSQL run

Integration tests that share one table would interfere (leftover rows, parallel tests). The trick: give **each test its own PostgreSQL schema** (a namespace inside the database), point the connection at it with `search_path`, create the table there, and `DROP SCHEMA … CASCADE` afterwards:

```go
// file: postgres/testdb_test.go
package postgres

import (
	"context"
	"ecommerce/infra/db"
	"fmt"
	"net/url"
	"os"
	"sync/atomic"
	"testing"
	"time"

	"github.com/jmoiron/sqlx"
)

var schemaCounter atomic.Int64

// testDB is an isolated schema plus the means to open more pools on it.
type testDB struct {
	*sqlx.DB        // a pool on the isolated schema
	dsn      string // reconnect with this to get a second, independent pool
}

func opts(u string) db.Options {
	return db.Options{URL: u, MaxOpenConns: 8, MaxIdleConns: 4, ConnMaxLifetime: time.Minute}
}

// newTestDB returns a pool whose search_path is a brand-new, empty schema containing only the
// users table. The schema is dropped when the test ends. The test is skipped without TEST_DATABASE_URL.
func newTestDB(t *testing.T) *testDB {
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

	ddl, err := os.ReadFile("../sql/001_create_users.sql")
	if err != nil {
		t.Fatal(err)
	}
	if _, err := conn.ExecContext(ctx, string(ddl)); err != nil {
		t.Fatalf("applying schema: %v", err)
	}
	return &testDB{DB: conn, dsn: u.String()}
}

// reconnect opens a second, independent pool on the same schema.
func (d *testDB) reconnect(t *testing.T) *sqlx.DB {
	t.Helper()
	conn, err := db.Connect(context.Background(), opts(d.dsn))
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { conn.Close() })
	return conn
}
```

`schema` in the helper is built from our own process ID and a counter, never from outside input, which is why concatenating it into `CREATE SCHEMA` is acceptable here (identifiers can't be placeholders). The helper returns a small `testDB` wrapper that also remembers the DSN, so a test can open a *second, independent* pool on the same schema (used to prove persistence below).

The PostgreSQL tests: the shared contract, plus behaviors specific to a real database:

```go
// file: postgres/user_store_test.go
package postgres

import (
	"context"
	"ecommerce/models"
	"ecommerce/user"
	"ecommerce/user/storetest"
	"errors"
	"testing"
)

var _ user.Store = (*UserStore)(nil) // the real store satisfies the interface

func TestUserStoreContract(t *testing.T) {
	storetest.Run(t, func(t *testing.T) user.Store { return NewUserStore(newTestDB(t).DB) })
}

func TestPasswordHashIsPersisted(t *testing.T) {
	d := newTestDB(t)
	NewUserStore(d.DB).Create(context.Background(), "asha@example.com", "$2a$10$abcdef")

	var stored string
	if err := d.Get(&stored, "SELECT password_hash FROM users"); err != nil || stored != "$2a$10$abcdef" {
		t.Errorf("stored hash = %q (%v)", stored, err)
	}
}

func TestDatabaseRejectsWhatTheStoreWouldNot(t *testing.T) {
	d := newTestDB(t)

	// bypass the Go code entirely: the constraints still protect the data
	insert := func(email string) error {
		_, err := d.Exec("INSERT INTO users (email, password_hash) VALUES ($1, 'h')", email)
		return err
	}
	if err := insert("dup@example.com"); err != nil {
		t.Fatal(err)
	}
	if err := insert("DUP@example.com"); err == nil {
		t.Error("the unique index on lower(email) must reject a differently-cased duplicate")
	}
	if err := insert(""); err == nil {
		t.Error("the CHECK constraint must reject an empty email")
	}
}

func TestCancelledContextStopsTheQuery(t *testing.T) {
	s := NewUserStore(newTestDB(t).DB)
	ctx, cancel := context.WithCancel(context.Background())
	cancel()

	if _, err := s.FindByID(ctx, 1); err == nil || errors.Is(err, models.ErrNotFound) {
		t.Errorf("a cancelled context must give an error that is not ErrNotFound, got %v", err)
	}
}

func TestDataOutlivesThePool(t *testing.T) {
	// Persistence in miniature: close every connection, open new ones, and the user is still there.
	d := newTestDB(t)
	created, err := NewUserStore(d.DB).Create(context.Background(), "durable@example.com", "h")
	if err != nil {
		t.Fatal(err)
	}
	fresh := d.reconnect(t) // a second, independent pool on the same schema

	d.DB.Close() // the first pool is gone, along with all its connections
	got, err := NewUserStore(fresh).FindByID(context.Background(), created.ID)
	if err != nil || got.Email != "durable@example.com" {
		t.Errorf("after the first pool closed: %+v, %v", got, err)
	}
}
```

Run everything:

```bash
go vet ./... && go test -race ./...            # PostgreSQL tests skip themselves
TEST_DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable' \
  go test -race -count=1 -v ./postgres/ ./database/ -run 'Contract|Persist|Reject|Cancelled|Outlives'
```

The same eight contract scenarios run against **both** implementations; PostgreSQL-only tests check the extras.

---

## 12. Wiring it in and trying it

Only the composition root changes. The users now live in PostgreSQL; products stay in memory until Chapter 56:

```go
// file: cmd/wire.go
package cmd

import (
	"ecommerce/config"
	"ecommerce/database"
	"ecommerce/health"
	"ecommerce/postgres"
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
	// storage: chosen here and nowhere else
	productStore := database.NewProductStore(database.SampleProducts()...) // still in memory (Chapter 56)
	userStore := postgres.NewUserStore(conn)                               // PostgreSQL

	productHandler := product.NewHandler(productStore, logger)
	userHandler := user.NewHandler(userStore, []byte(cfg.JWTSecret), cfg.JWTTTL, logger)
	healthHandler := health.NewHandler(conn, logger)

	return rest.NewServer(cfg, logger, productHandler, userHandler, healthHandler).Handler()
}
```

Notice what did **not** change: `user.Handler`, the middleware, the routes, every handler test. That is the payoff of Chapter 51's interface: replacing the storage was **one line**.

Start the server (with the environment from Chapter 52; we used `HTTP_PORT=18080`) and use the API. The output below is trimmed to the status line and body:

```bash
curl -s -X POST localhost:18080/users -d '{"email":"Asha@Example.com","password":"correct-horse-battery"}'
```

```
HTTP/1.1 201 Created
{"id":1,"email":"asha@example.com","createdAt":"2026-09-26T06:37:24.922112Z"}
```

Register the same email again (different capitalization), then log in and call `/me`:

```
# same email, different capitalization
HTTP/1.1 409 Conflict
{"error":"email already registered"}

# login (the token is shortened here), then GET /me with it
HTTP/1.1 200 OK
{"id":1,"email":"asha@example.com","createdAt":"2026-09-26T06:37:24.922112Z"}

# wrong password
HTTP/1.1 401 Unauthorized
{"error":"invalid email or password"}
```

Now the proof of **persistence**: stop the server (Ctrl+C), start it again, and log in with the same account. Then look at the row itself:

```bash
docker exec shop-db psql -U postgres -d ecommerce -c "SELECT id, email, left(password_hash, 7) AS hash_start, created_at FROM users"
```

```
 id |      email       | hash_start |          created_at
----+------------------+------------+-------------------------------
  1 | asha@example.com | $2a$10$    | 2026-09-26 06:37:24.922112+00
(1 row)
```

The account survived the restart, and the database holds only a **bcrypt hash** (`$2a$10$`…), never the password.

---

## 13. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Building SQL with `+` or `fmt.Sprintf` | **SQL injection** | Placeholders `$1, $2…` |
| 2 | Check-then-insert for uniqueness | Race: duplicates slip through | A `UNIQUE` index; handle SQLSTATE `23505` |
| 3 | Returning `sql.ErrNoRows` or `pgconn` errors to handlers | Leaky abstraction; every layer knows the database | Translate to domain errors in the store |
| 4 | `SELECT *` | Breaks on schema changes; can leak columns | Name the columns |
| 5 | Column/tag mismatch (`password_hash` vs field name) | `missing destination name` errors | `db:"password_hash"` tags |
| 6 | Forgetting `RETURNING` and then guessing the ID | Extra query or wrong ID | `INSERT … RETURNING id, …` |
| 7 | `TIMESTAMP` instead of `TIMESTAMPTZ` | Ambiguous times across zones/servers | `TIMESTAMPTZ`, and `.UTC()` in Go |
| 8 | Ignoring `ctx` in DB calls (`db.Get` not `GetContext`) | Cancelled requests keep queries running | Always the `…Context` variants |
| 9 | Not normalizing emails (case, spaces) | `Asha@x.com` and `asha@x.com` become two accounts | Normalize in one place + a `lower(email)` unique index |
| 10 | Testing against a shared, dirty table | Flaky tests | Schema per test (or transactions rolled back) |
| 11 | Re-running a hand-run SQL script | `relation already exists` | Migrations (Chapter 58) |
| 12 | Treating "email taken" as a 500 | Confusing errors for users | Map `ErrEmailTaken` → 409 (already done in Chapter 51) |
| 13 | Storing plaintext passwords | Catastrophic breach | bcrypt hashes (Chapter 48) |

---

## 14. Interview questions

**Q1. How do you prevent SQL injection in Go?**
Use parameterized queries (`$1`, `?`) so values are sent separately from the SQL text; never concatenate input into SQL; allow-list any dynamic identifiers.

**Q2. `Get` vs `Select` vs `Exec` in sqlx?**
`Get`: one row into a struct or scalar (`sql.ErrNoRows` if none). `Select`: many rows into a slice. `Exec`: statements without result rows (returns rows affected).

**Q3. How do you enforce unique emails?**
A unique index in the database (here on `lower(email)`), and translate the unique-violation error (SQLSTATE 23505) into a domain error. Application-level checks alone are racy.

**Q4. What does `RETURNING` do?**
Makes `INSERT/UPDATE/DELETE` return columns of the affected rows in the same round trip, ideal for generated IDs and defaults.

**Q5. Why translate `sql.ErrNoRows`?**
So callers depend on the app's own errors, not on the storage technology; swapping databases (or using a fake) changes nothing upstream.

**Q6. What is `//go:embed`?**
A compiler directive that includes files' contents in the binary as a `string`, `[]byte`, or `embed.FS`.

**Q7. What is a contract test?**
A single behavior suite run against every implementation of an interface, ensuring they are interchangeable.

**Q8. Why `TIMESTAMPTZ`?**
It stores an absolute instant; `TIMESTAMP` (without zone) stores just a wall-clock reading that's ambiguous across zones.

---

## 15. Exercises

### Exercise 1: More columns
Add `full_name TEXT NOT NULL DEFAULT ''` to `users`. What must change in SQL, the model, the store, and the handler? Why is a `DEFAULT` helpful when adding a NOT NULL column to a table that already has rows?

<details><summary>Solution</summary>

`ALTER TABLE users ADD COLUMN full_name TEXT NOT NULL DEFAULT ''`; add `FullName string `json:"fullName" db:"full_name"`` to `models.User`; list `full_name` in all three query files' column lists (and `INSERT` as a third parameter `$3`); accept it in the register request. Existing rows need *some* value when the new NOT NULL column appears; the `DEFAULT` supplies it (without a default, the `ALTER` would fail on a non-empty table). In production this schema change is exactly what Chapter 58's migrations manage.
</details>

### Exercise 2: Break the injection defenses on purpose
In a scratch program, write a vulnerable `findByEmail` using `fmt.Sprintf`, then log in as anyone using `' OR '1'='1`. Then port it to `$1` and confirm the attack fails.

<details><summary>Solution</summary>

See §6: the vulnerable version returns all rows; the parameterized version returns none because the entire input is compared as one string.
</details>

### Exercise 3: `ExistsByEmail`
Add `ExistsByEmail(ctx, email) (bool, error)` using `SELECT EXISTS(SELECT 1 FROM users WHERE lower(email) = lower($1))`. Should registration call it before `Create`? Why or why not?

<details><summary>Solution</summary>

```go
var exists bool
err := s.db.GetContext(ctx, &exists, `SELECT EXISTS(SELECT 1 FROM users WHERE lower(email) = lower($1))`, models.NormalizeEmail(email))
```
Useful for a friendly "email available?" UI hint, but registration must **not rely on it**: two requests can both see `false`. The unique index + handling `23505` is the only race-free guarantee.
</details>

### Exercise 4: Add it to the contract
Add a contract scenario "create then find returns the same `CreatedAt` (to the second)" and run it against both stores. What subtlety does PostgreSQL introduce?

<details><summary>Solution</summary>

Compare with `Truncate(time.Second)`/`WithinDuration`: PostgreSQL stores microsecond precision while Go's `time.Now()` has nanoseconds, so equality on the raw value can differ in the sub-microsecond digits in the in-memory store vs. Postgres. Contract tests should assert what callers can rely on.
</details>

### Exercise 5: Case-preserving emails
Suppose the product owner wants to *display* the email as typed (`Asha@Example.com`) but still treat it case-insensitively. What changes?

<details><summary>Solution</summary>

Stop lower-casing before storing (keep only `TrimSpace`); keep the unique index on `lower(email)`; lookups already use `lower(email) = lower($1)`. The in-memory store and contract tests must then compare case-insensitively too, showing why a shared contract is valuable.
</details>

### Exercise 6: Explain the query plan
In `psql`, run `EXPLAIN SELECT id FROM users WHERE lower(email) = 'asha@example.com'` after inserting a few thousand rows (`INSERT … SELECT generate_series(...)`). Does it use `users_email_lower_key`? What if the query used `email = '…'` instead?

<details><summary>Solution</summary>

`lower(email) = …` can use the expression index (an *Index Scan* on `users_email_lower_key`, for enough rows). Plain `email = …` cannot, because the index stores lowercased values, so PostgreSQL falls back to a sequential scan. Indexes only help queries that use the *same expression*. (Chapter 55 covers `EXPLAIN` in depth.)
</details>

---

## 16. Quiz

1. What is `RETURNING` for?
2. Why can't placeholders be used for table names?
3. Which error code means "duplicate key" in PostgreSQL?
4. What does `sqlx`'s `GetContext` return when no row matches?
5. Why translate driver errors inside the store?
6. What must be true of a struct field for `sqlx` to fill it?
7. Why does the contract test create a *fresh* store per scenario?
8. Why is a unique **index** better than an `if exists` check in Go?

<details><summary>Answers</summary>

1. Getting the stored row (generated ID, defaults) back from the `INSERT` in the same round trip.
2. Placeholders carry *values*; identifiers are part of the SQL structure, which is parsed before values are bound. Use an allow-list instead.
3. `23505` (`unique_violation`).
4. `sql.ErrNoRows`.
5. So the rest of the app depends on domain errors, not on a particular database or driver.
6. It must be exported and mapped to a column (matching name or a `db` tag).
7. So scenarios are independent: no leftover rows, any order, parallel-safe.
8. The database enforces it atomically, even against concurrent requests and other clients; a Go check-then-insert has a race window.
</details>

---

## 17. Summary

- A relational database stores **tables of typed rows**. Enforce invariants (`NOT NULL`, `UNIQUE`, `CHECK`) **in the database**, where nothing can bypass them.
- `users` has an identity primary key, `TIMESTAMPTZ` creation time, and a unique index on `lower(email)`.
- **Always parameterize** (`$1, $2…`); string-built SQL is injectable: we demonstrated leaking every user with `' OR '1'='1`.
- Keep queries in **embedded `.sql` files**, use `INSERT … RETURNING`, name columns, and map them to fields with **`db` tags**.
- The store **translates errors**: `sql.ErrNoRows → ErrNotFound`, SQLSTATE `23505 → ErrEmailTaken`, everything else wrapped with `%w`. Handlers stay database-agnostic.
- A **contract test** runs the same scenarios against both implementations; **schema-per-test** isolates integration tests, which skip without `TEST_DATABASE_URL`.
- Swapping in PostgreSQL was a **one-line change** in the composition root, the payoff of Chapter 51's interfaces.

### ➡️ What's next?

[Chapter 54](54-postgresql-data-types.md) is a guided tour of **PostgreSQL's data types** (numbers, money, text, time, JSON, UUID, arrays) so you can design the products table properly, and choose the right type for every column.
