# Chapter 58: Database Migrations

## Table of Contents
- [Introduction](#introduction)
- [What Are Database Migrations?](#what-are-database-migrations)
- [The Manual Problem](#the-manual-problem)
- [Setting Up Migrations](#setting-up-migrations)
- [Installing sql-migrate](#installing-sql-migrate)
- [Creating Migration Files](#creating-migration-files)
- [Writing UP and DOWN Migrations](#writing-up-and-down-migrations)
- [Implementing Migration Runner](#implementing-migration-runner)
- [Running Migrations on Startup](#running-migrations-on-startup)
- [Testing Migrations](#testing-migrations)
- [Common Issues and Fixes](#common-issues-and-fixes)
- [Summary](#summary)
- [Practice Questions](#practice-questions)

---

## Introduction

We've been creating database tables manually in pgAdmin—**this doesn't scale!**

**Current workflow:**
```
Need new table → Open pgAdmin → Write SQL → Execute → Done
Need to change table → Open pgAdmin → Write SQL → Execute → Done
Deploy to production → Open pgAdmin → Manually run all SQL → Pray it works
```

**Problems:**
- ❌ Manual work for every database change
- ❌ Easy to forget steps in production
- ❌ No version history of schema changes
- ❌ Team members don't know what changed
- ❌ Can't roll back changes easily

**Solution: Database Migrations!**

---

## What Are Database Migrations?

**Definition:** Migrations are version-controlled database schema changes.

**How it works:**

```
Migration Files:
├── 00001_create_users.up.sql     ← Create users table
├── 00001_create_users.down.sql   ← Drop users table
├── 00002_create_products.up.sql  ← Create products table
└── 00002_create_products.down.sql ← Drop products table

Run migrations → Tables created automatically
```

**Key concepts:**

```
UP migration:   Move schema forward (create, alter, add)
DOWN migration: Revert schema backward (drop, remove)

Version 0 → Version 1 (UP)   → Users table created
Version 1 → Version 0 (DOWN) → Users table dropped

Version 1 → Version 2 (UP)   → Products table created
Version 2 → Version 1 (DOWN) → Products table dropped
```

**Benefits:**

| Manual SQL | Migrations |
|------------|------------|
| No history | Version controlled |
| Manual execution | Automatic execution |
| Easy to forget | Impossible to forget |
| No rollback | Easy rollback |
| Team confusion | Team clarity |

---

## The Manual Problem

### What We've Been Doing

**File: `infra/db/queries/create_users_table.sql`**

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    is_banned BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**Process:**
1. Write SQL in file
2. Open pgAdmin
3. Copy-paste SQL
4. Execute manually
5. Hope you don't forget in production!

**Scenario: Add new column 10 days later**

```
Day 1:    Create users table (manually)
Day 10:   Need to add "phone_number" column
          
          Steps:
          1. Remember what tables exist
          2. Write ALTER TABLE SQL
          3. Open pgAdmin
          4. Execute on dev database
          5. Remember to run on staging
          6. Remember to run on production
          7. Hope nothing breaks!
```

**This is painful and error-prone!**

---

## Setting Up Migrations

### Create Migrations Folder

**Project structure:**

```
your-project/
├── cmd/
├── config/
├── database/
├── handler/
├── infra/
│   └── db/
│       ├── connection.go
│       └── migrate.go          ← New!
├── migrations/                  ← New folder!
│   ├── 00001_create_users.up.sql
│   ├── 00001_create_users.down.sql
│   ├── 00002_create_products.up.sql
│   └── 00002_create_products.down.sql
├── .env
├── go.mod
└── main.go
```

**Create folder:**

```bash
mkdir migrations
```

---

## Installing sql-migrate

### Why sql-migrate?

**Popular migration tools:**

1. **golang-migrate/migrate** (most popular)
   - Requires Go 1.23+
   - More complex setup
   - Industry standard

2. **rubenv/sql-migrate** (we'll use this)
   - Works with Go 1.22+
   - Simpler to use
   - Good for learning

### Installation

**Install package:**

```bash
go get github.com/rubenv/sql-migrate
```

**This adds to `go.mod`:**

```go
require (
    github.com/rubenv/sql-migrate v1.6.0
    // ... other dependencies
)
```

**Note about golang-migrate:**

```bash
# If you tried this first:
go get -u github.com/golang-migrate/migrate/v4

# Error: requires Go 1.23+
# We're using Go 1.22, so we use sql-migrate instead
```

---

## Creating Migration Files

### Naming Convention

**Format:** `{version}_{description}.{direction}.sql`

**Parts:**
- `version`: Sequential number (00001, 00002, 00003...)
- `description`: What the migration does
- `direction`: `up` or `down`
- `extension`: `.sql`

**Examples:**

```
✓ Good naming:
00001_create_users.up.sql
00001_create_users.down.sql
00002_create_products.up.sql
00002_create_products.down.sql
00003_add_phone_to_users.up.sql
00003_add_phone_to_users.down.sql

❌ Bad naming:
create_users.sql                  (no version, no direction)
1_users.up.sql                   (version should be 5 digits)
00001_CreateUsers.up.sql         (use snake_case)
```

### Create First Migration

**File: `migrations/00001_create_users.up.sql`**

```sql
-- +migrate Up
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    is_banned BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**File: `migrations/00001_create_users.down.sql`**

```sql
-- +migrate Down
DROP TABLE IF EXISTS users;
```

**Important annotations:**

```sql
-- +migrate Up
This tells sql-migrate: "This is an UP migration"

-- +migrate Down
This tells sql-migrate: "This is a DOWN migration"

Without these comments, sql-migrate won't recognize the files!
```

### Create Second Migration

**File: `migrations/00002_create_products.up.sql`**

```sql
-- +migrate Up
CREATE TABLE IF NOT EXISTS products (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    price DOUBLE PRECISION NOT NULL,
    image_url TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**File: `migrations/00002_create_products.down.sql`**

```sql
-- +migrate Down
DROP TABLE IF EXISTS products;
```

---

## Writing UP and DOWN Migrations

### UP Migrations: Moving Forward

**Purpose:** Apply changes to database (create, alter, add)

**Examples:**

```sql
-- Create table
-- +migrate Up
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);

-- Add column
-- +migrate Up
ALTER TABLE users
ADD COLUMN phone_number VARCHAR(20);

-- Create index
-- +migrate Up
CREATE INDEX idx_users_email ON users(email);

-- Add constraint
-- +migrate Up
ALTER TABLE orders
ADD CONSTRAINT fk_user_id
FOREIGN KEY (user_id) REFERENCES users(id);
```

**Why `IF NOT EXISTS`?**

```sql
-- ✓ Safe (won't fail if table exists)
CREATE TABLE IF NOT EXISTS users (...);

-- ❌ Unsafe (fails if table already exists)
CREATE TABLE users (...);

Scenario:
1. Run migrations → creates table
2. Run migrations again → sees IF NOT EXISTS, skips
3. No error!

Without IF NOT EXISTS:
1. Run migrations → creates table
2. Run migrations again → tries to create again
3. ERROR: table already exists
```

### DOWN Migrations: Rolling Back

**Purpose:** Undo changes from UP migration

**Rules:**

```
UP migration creates table   → DOWN migration drops table
UP migration adds column     → DOWN migration removes column
UP migration creates index   → DOWN migration drops index
UP migration adds constraint → DOWN migration drops constraint
```

**Examples:**

```sql
-- Drop table
-- +migrate Down
DROP TABLE IF EXISTS users;

-- Remove column
-- +migrate Down
ALTER TABLE users
DROP COLUMN phone_number;

-- Drop index
-- +migrate Down
DROP INDEX IF EXISTS idx_users_email;

-- Drop constraint
-- +migrate Down
ALTER TABLE orders
DROP CONSTRAINT fk_user_id;
```

**Why `IF EXISTS`?**

```sql
-- ✓ Safe (won't fail if table doesn't exist)
DROP TABLE IF EXISTS users;

-- ❌ Unsafe (fails if table doesn't exist)
DROP TABLE users;
```

### Complete Migration Pairs

**Migration 1: Create users**

```sql
-- 00001_create_users.up.sql
-- +migrate Up
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    is_banned BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 00001_create_users.down.sql
-- +migrate Down
DROP TABLE IF EXISTS users;
```

**Migration 2: Create products**

```sql
-- 00002_create_products.up.sql
-- +migrate Up
CREATE TABLE IF NOT EXISTS products (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    price DOUBLE PRECISION NOT NULL,
    image_url TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 00002_create_products.down.sql
-- +migrate Down
DROP TABLE IF EXISTS products;
```

---

## Implementing Migration Runner

### Create migrate.go

**File: `infra/db/migrate.go`**

```go
package db

import (
    "database/sql"
    "fmt"
    
    _ "github.com/lib/pq"  // PostgreSQL driver
    migrate "github.com/rubenv/sql-migrate"
)

func MigrateDB(db *sql.DB, migrationsDir string) error {
    // Configure migrations
    migrations := &migrate.FileMigrationSource{
        Dir: migrationsDir,
    }
    
    // Execute migrations
    n, err := migrate.Exec(db, "postgres", migrations, migrate.Up)
    if err != nil {
        return err
    }
    
    fmt.Printf("Successfully migrated %d migrations!\n", n)
    return nil
}
```

**Understanding the code:**

```go
// 1. Create migration source
migrations := &migrate.FileMigrationSource{
    Dir: migrationsDir,  // "./migrations"
}
// Tells sql-migrate where to find migration files

// 2. Execute migrations
n, err := migrate.Exec(
    db,              // Database connection
    "postgres",      // Database driver
    migrations,      // Migration source
    migrate.Up,      // Direction (Up or Down)
)
// Runs all pending UP migrations

// Returns:
// n   = number of migrations applied
// err = error if something went wrong
```

### Migration Directions

```go
// Apply all pending migrations
migrate.Exec(db, "postgres", migrations, migrate.Up)

// Rollback last migration
migrate.Exec(db, "postgres", migrations, migrate.Down)

// Apply specific number of migrations
migrate.ExecMax(db, "postgres", migrations, migrate.Up, 2)

// Rollback specific number
migrate.ExecMax(db, "postgres", migrations, migrate.Down, 1)
```

---

## Running Migrations on Startup

### Integrate with Serve Command

**File: `cmd/serve.go`**

```go
package cmd

import (
    "fmt"
    "os"
    
    "yourproject/cmd/server"
    "yourproject/config"
    "yourproject/database"
    "yourproject/handler/product"
    "yourproject/handler/user"
    "yourproject/infra/db"
)

func Serve() {
    // 1. Load config
    config.LoadConfig()
    cfg := config.GetConfig()
    
    // 2. Connect to database
    dbConnection, err := db.NewConnection(cfg.DB)
    if err != nil {
        fmt.Println("Failed to connect to database:", err)
        os.Exit(1)
    }
    defer dbConnection.Close()
    
    // 3. Run migrations (NEW!)
    err = db.MigrateDB(dbConnection.DB, "./migrations")
    if err != nil {
        fmt.Println("Failed to run migrations:", err)
        os.Exit(1)
    }
    
    // 4. Create repositories
    userRepo := database.NewUserRepository(dbConnection)
    productRepo := database.NewProductRepository(dbConnection)
    
    // 5. Create handlers
    userHandler := user.NewHandler(userRepo)
    productHandler := product.NewHandler(productRepo)
    
    // 6. Start server
    srv := server.NewServer(cfg, userHandler, productHandler)
    srv.Start()
}
```

**Important: dbConnection.DB**

```go
// sqlx.DB wrapper
dbConnection, err := db.NewConnection(cfg.DB)  // Returns *sqlx.DB

// Get underlying *sql.DB for migrations
err = db.MigrateDB(dbConnection.DB, "./migrations")
                              ↑
                              Extract *sql.DB from *sqlx.DB
```

**Migration directory path:**

```
"./migrations"

. = Current working directory (project root)
/ = Directory separator
migrations = Folder name

Full path: /path/to/project/migrations
```

---

## Testing Migrations

### First Run: Create Tables

**Step 1: Drop all tables (start fresh)**

```sql
-- In pgAdmin
DROP TABLE IF EXISTS users CASCADE;
DROP TABLE IF EXISTS products CASCADE;
```

**Step 2: Run application**

```bash
go run main.go
```

**Expected output:**

```
Successfully connected to PostgreSQL!
Successfully migrated 2 migrations!
Server running on port 4000
```

**Step 3: Check database**

```sql
SELECT * FROM users;
-- Empty table (structure created)

SELECT * FROM products;
-- Empty table (structure created)
```

**Step 4: Check migration tracking**

```sql
SELECT * FROM gorp_migrations;
```

**Result:**

```
 id |            applied_at           
----+---------------------------------
 00001_create_users.up.sql    | 2024-01-20 10:00:00
 00002_create_products.up.sql | 2024-01-20 10:00:01
```

**sql-migrate tracks which migrations ran!**

### Second Run: Skip Completed Migrations

**Run again:**

```bash
go run main.go
```

**Expected output:**

```
Successfully connected to PostgreSQL!
Successfully migrated 0 migrations!
Server running on port 4000
```

**0 migrations** because all migrations already ran!

### Test With Data

**Create user:**

```bash
POST http://localhost:4000/api/users/signup
Content-Type: application/json

{
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com",
  "password": "12345678"
}
```

**Check database:**

```sql
SELECT * FROM users;
```

**Result:**

```
 id | first_name | last_name | email              | ...
----+------------+-----------+--------------------+----
  1 | John       | Doe       | john@example.com   | ...
```

✓ Migrations working perfectly!

---

## Common Issues and Fixes

### Issue 1: Nil Pointer in Login

**Error:**

```
panic: runtime error: invalid memory address or nil pointer dereference
handler/user/login.go:34
```

**Code with bug:**

```go
func (h *Handler) Login(w http.ResponseWriter, r *http.Request) {
    // ... decode request
    
    user, err := h.UserRepo.FindByEmail(req.Email)
    if err != nil {
        // Handle error
    }
    
    // BUG: user might be nil!
    if user.ID == 0 {  // ← CRASH if user is nil
        // ...
    }
}
```

**Why it happens:**

```go
// When user not found:
user, err := h.UserRepo.FindByEmail("nonexistent@example.com")
// Returns: user = nil, err = nil

// Accessing nil pointer:
user.ID  // ← CRASH! Can't access field of nil pointer
```

**Fix: Check for nil first**

```go
func (h *Handler) Login(w http.ResponseWriter, r *http.Request) {
    // ... decode request
    
    user, err := h.UserRepo.FindByEmail(req.Email)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Check if user is nil (not found)
    if user == nil {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }
    
    // Now safe to access user.ID
    if !verifyPassword(user.Password, req.Password) {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }
    
    // Generate JWT...
}
```

### Issue 2: Pointer vs Value in Repository

**Bug in FindByEmail:**

```go
func (r *UserRepository) FindByEmail(email string) (*User, error) {
    var user User  // ← Value, not pointer
    
    query := "SELECT * FROM users WHERE email = $1"
    err := r.DB.Get(user, query, email)  // ← Should be &user
    
    return &user, err
}
```

**Fix: Use pointer**

```go
func (r *UserRepository) FindByEmail(email string) (*User, error) {
    var user User
    
    query := "SELECT * FROM users WHERE email = $1"
    err := r.DB.Get(&user, query, email)  // ← Pass address
    
    if err == sql.ErrNoRows {
        return nil, nil  // Not found, not an error
    }
    
    if err != nil {
        return nil, err  // Real error
    }
    
    return &user, nil
}
```

### Issue 3: Go Version Requirement

**Error with golang-migrate:**

```bash
go get github.com/golang-migrate/migrate/v4

Error: requires Go 1.23+
```

**Solution 1: Upgrade Go**

```bash
# Download Go 1.23 or 1.24
# Update go.mod
go 1.24
```

**Solution 2: Use sql-migrate (what we did)**

```bash
go get github.com/rubenv/sql-migrate
# Works with Go 1.22+
```

### Issue 4: Migration Files Not Found

**Error:**

```
Failed to run migrations: no migration files found
```

**Causes:**

```
1. Wrong directory path
   db.MigrateDB(dbConnection.DB, "migrations")  // ❌ Missing ./
   db.MigrateDB(dbConnection.DB, "./migrations") // ✓ Correct

2. Wrong file naming
   create_users.sql          // ❌ Missing direction
   create_users.up.sql       // ✓ Correct

3. Missing annotations
   CREATE TABLE users (...); // ❌ Missing -- +migrate Up
   -- +migrate Up            // ✓ Correct
   CREATE TABLE users (...);

4. Wrong directory structure
   migrations/sql/00001...   // ❌ Files in subdirectory
   migrations/00001...       // ✓ Files directly in migrations/
```

---

## Summary

### What We Accomplished

**1. Created migrations folder**
```
migrations/
├── 00001_create_users.up.sql
├── 00001_create_users.down.sql
├── 00002_create_products.up.sql
└── 00002_create_products.down.sql
```

**2. Installed sql-migrate**
```bash
go get github.com/rubenv/sql-migrate
```

**3. Implemented migration runner**
```go
func MigrateDB(db *sql.DB, migrationsDir string) error {
    migrations := &migrate.FileMigrationSource{Dir: migrationsDir}
    n, err := migrate.Exec(db, "postgres", migrations, migrate.Up)
    return err
}
```

**4. Run migrations on startup**
```go
// In cmd/serve.go
err = db.MigrateDB(dbConnection.DB, "./migrations")
```

**5. Removed manual SQL files**
```
Before: infra/db/queries/create_users_table.sql
After:  migrations/00001_create_users.up.sql (automatic!)
```

### Benefits Achieved

| Before Migrations | After Migrations |
|-------------------|------------------|
| Manual SQL execution | Automatic execution |
| No version history | Tracked in database |
| Easy to forget | Impossible to forget |
| No rollback | Easy rollback |
| Team confusion | Team clarity |
| Production errors | Production safety |

### Key Concepts

**1. UP migrations = Forward**
```sql
-- +migrate Up
CREATE TABLE users (...);
```

**2. DOWN migrations = Backward**
```sql
-- +migrate Down
DROP TABLE users;
```

**3. Version tracking**
```
sql-migrate creates gorp_migrations table
Tracks which migrations already ran
Prevents duplicate execution
```

**4. Idempotent migrations**
```sql
CREATE TABLE IF NOT EXISTS users (...);  -- Safe to run multiple times
DROP TABLE IF EXISTS users;              -- Safe to run multiple times
```

---

## Practice Questions

### Question 1: Add Phone Number Column

**Question:** Create a new migration to add `phone_number` column to users table. Make it optional (nullable) and add index for performance.

<details>
<summary>Click to see answer</summary>

**File: `migrations/00003_add_phone_to_users.up.sql`**

```sql
-- +migrate Up
ALTER TABLE users
ADD COLUMN phone_number VARCHAR(20);

CREATE INDEX idx_users_phone_number ON users(phone_number);
```

**File: `migrations/00003_add_phone_to_users.down.sql`**

```sql
-- +migrate Down
DROP INDEX IF EXISTS idx_users_phone_number;

ALTER TABLE users
DROP COLUMN phone_number;
```

**Test:**

```bash
# Run app
go run main.go
# Output: Successfully migrated 1 migrations!

# Check database
SELECT * FROM users;
# Now has phone_number column!

# Rollback (if needed)
# Modify MigrateDB to use migrate.Down
# Or manually: DROP INDEX ... ALTER TABLE ...
```

**Why this order?**

```sql
-- UP: Create column first, then index
1. ADD COLUMN phone_number     ← Column must exist
2. CREATE INDEX on phone_number ← Then create index

-- DOWN: Drop index first, then column
1. DROP INDEX idx_users_phone_number  ← Remove index first
2. DROP COLUMN phone_number           ← Then remove column

Reason: Can't create index on non-existent column
        Can't drop column that has index on it
```

</details>

---

### Question 2: Create Orders Table with Foreign Key

**Question:** Create orders table with foreign key to users table. Include proper rollback.

<details>
<summary>Click to see answer</summary>

**File: `migrations/00004_create_orders.up.sql`**

```sql
-- +migrate Up
CREATE TABLE IF NOT EXISTS orders (
    id BIGSERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1,
    total_price DOUBLE PRECISION NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT fk_user
        FOREIGN KEY (user_id)
        REFERENCES users(id)
        ON DELETE CASCADE,
    
    CONSTRAINT fk_product
        FOREIGN KEY (product_id)
        REFERENCES products(id)
        ON DELETE CASCADE
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_product_id ON orders(product_id);
CREATE INDEX idx_orders_status ON orders(status);
```

**File: `migrations/00004_create_orders.down.sql`**

```sql
-- +migrate Down
DROP TABLE IF EXISTS orders;
```

**Foreign key behavior:**

```sql
ON DELETE CASCADE
-- When user deleted → all their orders deleted
-- When product deleted → all orders with that product deleted

ON DELETE SET NULL
-- When user deleted → orders.user_id set to NULL
-- Useful if you want to keep order history

ON DELETE RESTRICT
-- Can't delete user if they have orders
-- Must delete orders first
```

**Test with data:**

```sql
-- Create order
INSERT INTO orders (user_id, product_id, quantity, total_price)
VALUES (1, 1, 2, 199.98);

-- Try to delete user (CASCADE will delete orders too)
DELETE FROM users WHERE id = 1;
-- Order automatically deleted!
```

</details>

---

### Question 3: Rollback Specific Migration

**Question:** Implement a function to rollback the last N migrations.

<details>
<summary>Click to see answer</summary>

**File: `infra/db/migrate.go`**

```go
package db

import (
    "database/sql"
    "fmt"
    
    migrate "github.com/rubenv/sql-migrate"
)

// MigrateUp runs all pending UP migrations
func MigrateUp(db *sql.DB, migrationsDir string) error {
    migrations := &migrate.FileMigrationSource{Dir: migrationsDir}
    n, err := migrate.Exec(db, "postgres", migrations, migrate.Up)
    if err != nil {
        return err
    }
    fmt.Printf("Applied %d migrations\n", n)
    return nil
}

// MigrateDown rolls back last N migrations
func MigrateDown(db *sql.DB, migrationsDir string, count int) error {
    migrations := &migrate.FileMigrationSource{Dir: migrationsDir}
    n, err := migrate.ExecMax(db, "postgres", migrations, migrate.Down, count)
    if err != nil {
        return err
    }
    fmt.Printf("Rolled back %d migrations\n", n)
    return nil
}

// MigrateStatus shows current migration status
func MigrateStatus(db *sql.DB, migrationsDir string) error {
    migrations := &migrate.FileMigrationSource{Dir: migrationsDir}
    records, err := migrate.GetMigrationRecords(db, "postgres")
    if err != nil {
        return err
    }
    
    fmt.Println("Applied migrations:")
    for _, record := range records {
        fmt.Printf("  - %s (applied at %v)\n", record.Id, record.AppliedAt)
    }
    
    return nil
}
```

**Add CLI command:**

```go
// cmd/migrate.go
package cmd

import (
    "fmt"
    "os"
    "strconv"
    
    "yourproject/config"
    "yourproject/infra/db"
)

func MigrateCommand(args []string) {
    if len(args) == 0 {
        fmt.Println("Usage: go run main.go migrate [up|down|status] [count]")
        os.Exit(1)
    }
    
    // Load config and connect
    config.LoadConfig()
    cfg := config.GetConfig()
    dbConnection, err := db.NewConnection(cfg.DB)
    if err != nil {
        fmt.Println("Failed to connect:", err)
        os.Exit(1)
    }
    defer dbConnection.Close()
    
    command := args[0]
    
    switch command {
    case "up":
        err = db.MigrateUp(dbConnection.DB, "./migrations")
        
    case "down":
        count := 1
        if len(args) > 1 {
            count, _ = strconv.Atoi(args[1])
        }
        err = db.MigrateDown(dbConnection.DB, "./migrations", count)
        
    case "status":
        err = db.MigrateStatus(dbConnection.DB, "./migrations")
        
    default:
        fmt.Println("Unknown command:", command)
        os.Exit(1)
    }
    
    if err != nil {
        fmt.Println("Migration error:", err)
        os.Exit(1)
    }
}
```

**Usage:**

```bash
# Apply all pending migrations
go run main.go migrate up

# Rollback last migration
go run main.go migrate down

# Rollback last 3 migrations
go run main.go migrate down 3

# Show migration status
go run main.go migrate status
```

**Example output:**

```bash
$ go run main.go migrate status
Applied migrations:
  - 00001_create_users.up.sql (applied at 2024-01-20 10:00:00)
  - 00002_create_products.up.sql (applied at 2024-01-20 10:00:01)
  - 00003_add_phone_to_users.up.sql (applied at 2024-01-20 10:05:00)

$ go run main.go migrate down 1
Rolled back 1 migrations

$ go run main.go migrate status
Applied migrations:
  - 00001_create_users.up.sql (applied at 2024-01-20 10:00:00)
  - 00002_create_products.up.sql (applied at 2024-01-20 10:00:01)
```

</details>

---

**Congratulations!** Your project now has professional database migrations! Your schema changes are version-controlled, automatic, and rollback-friendly! 🎉

**Next chapter preview:** We'll explore advanced database topics including transactions, connection pooling, and query optimization!
