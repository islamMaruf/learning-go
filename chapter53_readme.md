# Chapter 53: Database Operations - Create & Find Users

## Table of Contents
- [Introduction](#introduction)
- [Creating Users Table](#creating-users-table)
- [Understanding Table Schema](#understanding-table-schema)
- [Storing SQL Queries](#storing-sql-queries)
- [Modifying User Repository](#modifying-user-repository)
- [Implementing Create Operation](#implementing-create-operation)
- [Implementing Find Operation](#implementing-find-operation)
- [Database Struct Tags](#database-struct-tags)
- [Fixing SSL Mode Issue](#fixing-ssl-mode-issue)
- [Testing the Implementation](#testing-the-implementation)
- [Summary](#summary)
- [Practice Questions](#practice-questions)

---

## Introduction

In Chapter 52, we connected to PostgreSQL. Now we'll:
1. **Create tables** - Users table in PostgreSQL
2. **Write SQL queries** - INSERT and SELECT operations
3. **Replace in-memory storage** - Use real database
4. **Test with real data** - Create and find users

**No time to waste—let's code!**

---

## Creating Users Table

### Using pgAdmin Query Tool

**Step 1: Open Query Tool**

```
pgAdmin 4
└─ Servers
    └─ PostgreSQL 15
        └─ Databases
            └─ ecommerce
                └─ Right-click → Query Tool
```

**Step 2: Write CREATE TABLE SQL**

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    is_shop_owner BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Step 3: Execute Query**

Click **Execute/Refresh** button (F5) or click the ▶️ icon.

**Expected output:**
```
Query returned successfully in 44 msec.
```

**Step 4: Verify Table Created**

```
Tables
└─ Right-click → Refresh
    └─ users ← Should appear here
```

**View table structure:**
```
users
├─ Columns
│  ├─ id (integer) PK
│  ├─ first_name (varchar)
│  ├─ last_name (varchar)
│  ├─ email (varchar) UNIQUE
│  ├─ password (varchar)
│  ├─ is_shop_owner (boolean)
│  ├─ created_at (timestamp)
│  └─ updated_at (timestamp)
└─ Constraints
   ├─ PRIMARY KEY (id)
   └─ UNIQUE (email)
```

**View data (currently empty):**
```
Right-click users → View/Edit Data → All Rows
```

No data yet—we'll add it programmatically!

---

## Understanding Table Schema

### Column Definitions

**id - Serial Primary Key**
```sql
id SERIAL PRIMARY KEY
```

**Breakdown:**
- `SERIAL` = Auto-incrementing integer (1, 2, 3, 4, ...)
- `PRIMARY KEY` = Unique identifier for each row
- Automatically generated, never manually set

**first_name, last_name - Variable Character**
```sql
first_name VARCHAR(100) NOT NULL
last_name VARCHAR(100) NOT NULL
```

**Breakdown:**
- `VARCHAR(100)` = Variable length string, max 100 characters
- `NOT NULL` = Must provide value, cannot be empty
- Stores user's name

**email - Unique Constraint**
```sql
email VARCHAR(255) NOT NULL UNIQUE
```

**Breakdown:**
- `VARCHAR(255)` = Up to 255 characters
- `NOT NULL` = Required field
- `UNIQUE` = No two users can have same email
- Prevents duplicate registrations

**password - Encrypted Storage**
```sql
password VARCHAR(255) NOT NULL
```

**Breakdown:**
- `VARCHAR(255)` = Store hashed password
- `NOT NULL` = Required
- Will store bcrypt hash (not plain text!)

**is_shop_owner - Boolean Flag**
```sql
is_shop_owner BOOLEAN DEFAULT FALSE
```

**Breakdown:**
- `BOOLEAN` = true or false
- `DEFAULT FALSE` = If not specified, defaults to false
- Distinguishes regular users from shop owners

**Timestamps - Auto-managed**
```sql
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
```

**Breakdown:**
- `TIMESTAMP` = Date and time
- `DEFAULT CURRENT_TIMESTAMP` = Automatically set to current time
- `created_at` = Never changes (when user created)
- `updated_at` = Can be updated when user info changes

### Data Types Explained

| Type | Description | Example |
|------|-------------|---------|
| **SERIAL** | Auto-increment integer | 1, 2, 3, ... |
| **VARCHAR(n)** | Variable string, max n chars | "Habib", "user@example.com" |
| **BOOLEAN** | True or false | true, false |
| **TIMESTAMP** | Date and time | 2024-01-15 10:30:00 |
| **INTEGER** | Whole number | 42, -10, 0 |
| **TEXT** | Unlimited length string | Long descriptions |

### Constraints Explained

| Constraint | Purpose | Example |
|------------|---------|---------|
| **PRIMARY KEY** | Unique identifier | User ID |
| **NOT NULL** | Must have value | Email required |
| **UNIQUE** | No duplicates | Email must be unique |
| **DEFAULT** | Default value if not provided | FALSE for is_shop_owner |
| **FOREIGN KEY** | References another table | (Not used here yet) |

---

## Storing SQL Queries

### Create SQL Folder

**Why save SQL queries?**
- ✓ Reference later
- ✓ Documentation
- ✓ Easy to modify
- ✓ Share with team

**Create folder structure:**

```bash
mkdir -p infra/db/queries
```

**File: `infra/db/queries/01_create_users_table.sql`**

```sql
-- Create users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    is_shop_owner BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create index on email for faster lookups
CREATE INDEX idx_users_email ON users(email);
```

**Benefits:**
- Version control SQL schemas
- Easy to recreate database
- Document database structure
- Migration scripts

---

## Modifying User Repository

### Add Database Dependency

**File: `database/user.go`**

**Before (in-memory):**
```go
package database

type UserRepository struct {
    // No database connection
}

var users []User  // In-memory storage

func (r *UserRepository) Store(user User) User {
    user.ID = len(users) + 1
    users = append(users, user)  // Store in array
    return user
}
```

**After (with database):**
```go
package database

import (
    "github.com/jmoiron/sqlx"
)

type UserRepository struct {
    DB *sqlx.DB  // Database connection
}

// No more in-memory array!
// Data stored in PostgreSQL

func (r *UserRepository) Store(user User) (*User, error) {
    // Will use database INSERT
}
```

### Understanding the Change

**Visual comparison:**

```
Before:
UserRepository → []User array (in memory)
                 └─ Lost when app restarts

After:
UserRepository → PostgreSQL database
                 └─ Persists forever
```

**Dependency Injection:**

```go
type UserRepository struct {
    DB *sqlx.DB  // ← Injected dependency
}

// Inject database when creating repository
func NewUserRepository(db *sqlx.DB) *UserRepository {
    return &UserRepository{
        DB: db,
    }
}
```

---

## Implementing Create Operation

### SQL Insert Query

**What we want to do:**
```sql
INSERT INTO users (first_name, last_name, email, password, is_shop_owner)
VALUES ('Habib', 'Rahman', 'habib@example.com', 'hashed_password', true)
RETURNING id;
```

**Why `RETURNING id`?**
- PostgreSQL generates the ID (SERIAL)
- We need to return it to the user
- Response should include the new ID

### Store Method Implementation

**File: `database/user.go`**

```go
package database

import (
    "database/sql"
    "fmt"
    
    "github.com/jmoiron/sqlx"
)

type UserRepository struct {
    DB *sqlx.DB
}

func NewUserRepository(db *sqlx.DB) *UserRepository {
    return &UserRepository{DB: db}
}

// Store creates a new user in database
func (r *UserRepository) Store(user User) (*User, error) {
    // SQL query
    query := `
        INSERT INTO users (first_name, last_name, email, password, is_shop_owner)
        VALUES (:first_name, :last_name, :email, :password, :is_shop_owner)
        RETURNING id
    `
    
    // Execute query with named parameters
    rows, err := r.DB.NamedQuery(query, &user)
    if err != nil {
        fmt.Println("Error creating user:", err)
        return nil, err
    }
    defer rows.Close()
    
    // Get the generated ID
    if rows.Next() {
        var userID int
        err = rows.Scan(&userID)
        if err != nil {
            return nil, err
        }
        user.ID = userID
    }
    
    return &user, nil
}
```

### Understanding NamedQuery

**Named parameters with colons:**

```go
query := `
    INSERT INTO users (first_name, last_name, email)
    VALUES (:first_name, :last_name, :email)
`
```

**How it works:**

```
:first_name → Looks for field named "first_name" in struct
:last_name  → Looks for field named "last_name" in struct
:email      → Looks for field named "email" in struct
```

**sqlx automatically maps struct fields to parameters!**

**Why better than positional parameters:**

```go
// Positional (hard to read, easy to mess up order)
query := "INSERT INTO users VALUES ($1, $2, $3)"
db.Exec(query, firstName, lastName, email)

// Named (clear, order doesn't matter)
query := "INSERT INTO users VALUES (:first_name, :last_name, :email)"
db.NamedQuery(query, user)
```

---

## Implementing Find Operation

### SQL Select Query

**What we want to do:**
```sql
SELECT id, first_name, last_name, email, password, is_shop_owner, created_at, updated_at
FROM users
WHERE email = 'habib@example.com' AND password = 'hashed_password'
LIMIT 1;
```

### Find Method Implementation

**File: `database/user.go`**

```go
// Find searches for user by email and password
func (r *UserRepository) Find(email, password string) *User {
    var user User
    
    // SQL query
    query := `
        SELECT id, first_name, last_name, email, password, is_shop_owner, created_at, updated_at
        FROM users
        WHERE email = $1 AND password = $2
        LIMIT 1
    `
    
    // Execute query
    err := r.DB.Get(&user, query, email, password)
    if err != nil {
        if err == sql.ErrNoRows {
            // No user found with those credentials
            return nil
        }
        // Other error
        fmt.Println("Error finding user:", err)
        return nil
    }
    
    return &user
}
```

### Understanding Get Method

**`DB.Get()` is convenience method:**

```go
err := r.DB.Get(&user, query, email, password)
```

**What it does:**
1. Executes query with parameters
2. Scans result into struct
3. Returns error if no rows or other error

**Parameters:**
- `&user` - Destination (where to store result)
- `query` - SQL query
- `email, password` - Positional parameters ($1, $2)

**Positional parameters ($1, $2):**

```sql
WHERE email = $1 AND password = $2
              ↑                ↑
              First arg        Second arg
```

```go
r.DB.Get(&user, query, email, password)
                       ↑       ↑
                       $1      $2
```

### Handling No Results

```go
if err == sql.ErrNoRows {
    return nil  // User not found
}
```

**Why check for `ErrNoRows`?**
- "No rows found" is not really an error
- It just means wrong credentials
- Different from database connection error

---

## Database Struct Tags

### The Problem

**Without struct tags:**

```go
type User struct {
    FirstName string  // Go field name: FirstName
}
```

**Database column:**
```sql
first_name VARCHAR(100)  -- Database column: first_name
```

**Mismatch!** Go uses `FirstName`, database uses `first_name`.

### The Solution: Struct Tags

**Add `db` tags to match database columns:**

**File: `database/user.go`**

```go
type User struct {
    ID           int       `json:"id" db:"id"`
    FirstName    string    `json:"first_name" db:"first_name"`
    LastName     string    `json:"last_name" db:"last_name"`
    Email        string    `json:"email" db:"email"`
    Password     string    `json:"password" db:"password"`
    IsShopOwner  bool      `json:"is_shop_owner" db:"is_shop_owner"`
    CreatedAt    time.Time `json:"created_at" db:"created_at"`
    UpdatedAt    time.Time `json:"updated_at" db:"updated_at"`
}
```

**Two types of tags:**

```go
`json:"first_name" db:"first_name"`
  ↑                ↑
  JSON field name  Database column name
```

**Why both tags?**

```
Client (JSON)
    ↓
Handler
    ↓ (json tag)
Go Struct
    ↓ (db tag)
Database
```

### Tag Rules

**Format:**
```go
`db:"column_name"`
```

**Rules:**
1. Must match database column **exactly**
2. Case-sensitive
3. Underscores matter (`first_name` ≠ `firstname`)
4. If missing, sqlx uses field name as-is

**Example mismatches:**

```go
// ❌ Wrong - won't work
type User struct {
    FirstName string `db:"firstname"`  // Database has first_name
}

// ❌ Wrong - case matters
type User struct {
    FirstName string `db:"First_Name"`  // Database has first_name
}

// ✓ Correct
type User struct {
    FirstName string `db:"first_name"`  // Matches database
}
```

---

## Fixing SSL Mode Issue

### The Error

```
pq: SSL is not enabled on the server
```

**What this means:**
- Your PostgreSQL server doesn't have SSL enabled
- Connection string tries to use SSL by default
- Local development typically doesn't need SSL

### The Fix

**Update connection string:**

**File: `infra/db/connection.go`**

**Before:**
```go
func GetConnectionString() string {
    return fmt.Sprintf(
        "user=%s password=%s host=%s port=%s dbname=%s",
        user, password, host, port, dbname,
    )
}
```

**After:**
```go
func GetConnectionString() string {
    return fmt.Sprintf(
        "user=%s password=%s host=%s port=%s dbname=%s sslmode=disable",
        user, password, host, port, dbname,
    )
}
```

**What `sslmode=disable` means:**

```
sslmode=disable → Don't use SSL encryption

For local development:
✓ localhost → sslmode=disable (fine)
✗ Production → NEVER disable SSL!
```

### SSL Modes Explained

| Mode | Description | Use Case |
|------|-------------|----------|
| **disable** | No SSL | Local development only |
| **require** | SSL required | Production |
| **verify-ca** | Verify SSL certificate | Production (preferred) |
| **verify-full** | Verify cert + hostname | Production (most secure) |

**Production connection string:**

```go
// Production (DO use SSL!)
"user=prod password=xxx host=db.example.com port=5432 dbname=myapp sslmode=require"
```

### HTTP vs HTTPS Analogy

**Think of it like websites:**

```
Local development:
http://localhost:4000  ← No SSL (fine for local)

Production:
https://facebook.com   ← SSL (required for production)
https://google.com     ← SSL (secure)
```

**Same with databases:**

```
Local:
postgres://localhost:5432/ecommerce?sslmode=disable  ← No SSL

Production:
postgres://production-db.com:5432/ecommerce?sslmode=require  ← SSL
```

---

## Testing the Implementation

### Update cmd/serve.go

**Inject database into repository:**

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
    // Load config
    config.LoadConfig()
    cfg := config.GetConfig()
    
    // Connect to database
    dbConnection, err := db.NewConnection()
    if err != nil {
        fmt.Println("Failed to connect to database:", err)
        os.Exit(1)
    }
    defer dbConnection.Close()
    
    // Create repositories with database
    userRepo := database.NewUserRepository(dbConnection)
    
    // Create handlers with repositories
    userHandler := user.NewHandler(userRepo)  // Inject repository
    productHandler := product.NewHandler()
    
    // Create server
    srv := server.NewServer(
        cfg,
        userHandler,
        productHandler,
    )
    
    // Start server
    srv.Start()
}
```

**Key changes:**

```go
// Old: No database
userHandler := user.NewHandler()

// New: With database
userRepo := database.NewUserRepository(dbConnection)
userHandler := user.NewHandler(userRepo)
```

### Update User Handler

**Inject repository into handler:**

**File: `handler/user/handler.go`**

```go
package user

import (
    "yourproject/database"
)

type Handler struct {
    UserRepo *database.UserRepository  // Inject repository
}

func NewHandler(userRepo *database.UserRepository) *Handler {
    return &Handler{
        UserRepo: userRepo,
    }
}
```

**Update CreateUser handler:**

**File: `handler/user/create_user.go`**

```go
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var newUser database.User
    err := json.NewDecoder(r.Body).Decode(&newUser)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid request data"))
        return
    }
    
    // Store in database (not in-memory array!)
    createdUser, err := h.UserRepo.Store(newUser)
    if err != nil {
        fmt.Println("Error creating user:", err)
        http.Error(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(createdUser)
}
```

### Test Create User

**Start server:**
```bash
go run main.go
```

**Expected output:**
```
Successfully connected to PostgreSQL!
Server running on port 4000
```

**Create user via Postman:**

```
POST http://localhost:4000/api/users
Content-Type: application/json

{
  "first_name": "Habib",
  "last_name": "Rahman",
  "email": "habib@example.com",
  "password": "password123",
  "is_shop_owner": true
}
```

**Response:**
```json
{
  "id": 1,
  "first_name": "Habib",
  "last_name": "Rahman",
  "email": "habib@example.com",
  "password": "password123",
  "is_shop_owner": true,
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**Verify in database:**

```sql
SELECT * FROM users;
```

**Result:**
```
 id | first_name | last_name |        email        |   password   | is_shop_owner |      created_at       |      updated_at
----+------------+-----------+---------------------+--------------+---------------+-----------------------+-----------------------
  1 | Habib      | Rahman    | habib@example.com   | password123  | t             | 2024-01-15 10:30:00   | 2024-01-15 10:30:00
```

✓ User stored in database!

### Test Login (Find User)

**Login request:**

```
POST http://localhost:4000/api/users/login
Content-Type: application/json

{
  "email": "habib@example.com",
  "password": "password123"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

✓ User found in database!

### Persistence Test

**Restart server:**
```bash
# Stop server (Ctrl+C)
# Start again
go run main.go
```

**Try login again:**
```
POST http://localhost:4000/api/users/login
```

✓ Still works! Data persists across restarts!

**This is the power of a database:**
- Data survives server restarts
- No more in-memory arrays
- Real production-ready storage

---

## Summary

### What We Accomplished

**1. Created Database Table**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100),
    email VARCHAR(255) UNIQUE,
    -- ... more columns
);
```

**2. Saved SQL Queries**
```
infra/db/queries/
└─ 01_create_users_table.sql
```

**3. Modified Repository**
```go
type UserRepository struct {
    DB *sqlx.DB  // Added database dependency
}
```

**4. Implemented Create**
```go
func (r *UserRepository) Store(user User) (*User, error) {
    // INSERT INTO users ...
    // RETURNING id
}
```

**5. Implemented Find**
```go
func (r *UserRepository) Find(email, password string) *User {
    // SELECT * FROM users WHERE email = $1 AND password = $2
}
```

**6. Added Struct Tags**
```go
type User struct {
    FirstName string `json:"first_name" db:"first_name"`
    // Maps Go field to database column
}
```

**7. Fixed SSL Issue**
```go
"... sslmode=disable"  // For local development
```

**8. Tested Successfully**
```
✓ Create user → Stored in database
✓ Login → Found in database
✓ Restart server → Data persists
```

### Key Concepts

**Named Parameters:**
```go
INSERT INTO users VALUES (:first_name, :last_name)
                          ↑            ↑
                    Maps to struct field
```

**Struct Tags:**
```go
`db:"first_name"`  ← Database column name
```

**RETURNING Clause:**
```sql
INSERT ... RETURNING id  ← Get generated ID back
```

**Error Handling:**
```go
if err == sql.ErrNoRows {
    return nil  // Not found (not an error)
}
```

### Before vs After

**Before (In-Memory):**
```go
var users []User  // Lost on restart

func Store(user User) User {
    users = append(users, user)
    return user
}
```

**After (Database):**
```go
func (r *UserRepository) Store(user User) (*User, error) {
    // INSERT INTO users ...
    // Data persists forever
}
```

### Next Steps

In the next chapters, we'll:
1. **Hash passwords** - bcrypt for security
2. **Add products table** - Complete CRUD
3. **Implement UPDATE and DELETE** - Full operations
4. **Add migrations** - Database version control
5. **Error handling** - Better error messages

We now have **real database integration**—no more toy examples! 🚀

---

## Practice Questions

### Question 1: Add More User Fields

**Question:**
Add the following fields to the users table and update the code:
- `phone_number` (VARCHAR 20, nullable)
- `date_of_birth` (DATE, nullable)
- `country` (VARCHAR 100, default 'Bangladesh')

Implement the complete flow: create table, update struct, and test.

<details>
<summary>Click to see answer</summary>

**Answer:**

**Step 1: Drop and recreate table**

**File: `infra/db/queries/02_add_user_fields.sql`**

```sql
-- Drop existing table (careful in production!)
DROP TABLE IF EXISTS users;

-- Recreate with new fields
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    phone_number VARCHAR(20),  -- New field (nullable)
    date_of_birth DATE,        -- New field (nullable)
    country VARCHAR(100) DEFAULT 'Bangladesh',  -- New field with default
    is_shop_owner BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index on email
CREATE INDEX idx_users_email ON users(email);
```

**Execute in pgAdmin Query Tool.**

**Step 2: Update User struct**

**File: `database/user.go`**

```go
package database

import "time"

type User struct {
    ID           int        `json:"id" db:"id"`
    FirstName    string     `json:"first_name" db:"first_name"`
    LastName     string     `json:"last_name" db:"last_name"`
    Email        string     `json:"email" db:"email"`
    Password     string     `json:"password" db:"password"`
    PhoneNumber  *string    `json:"phone_number,omitempty" db:"phone_number"`  // Pointer for nullable
    DateOfBirth  *time.Time `json:"date_of_birth,omitempty" db:"date_of_birth"` // Pointer for nullable
    Country      string     `json:"country" db:"country"`
    IsShopOwner  bool       `json:"is_shop_owner" db:"is_shop_owner"`
    CreatedAt    time.Time  `json:"created_at" db:"created_at"`
    UpdatedAt    time.Time  `json:"updated_at" db:"updated_at"`
}
```

**Why pointers for nullable fields?**

```go
PhoneNumber *string  // Can be nil (NULL in database)

// Without pointer:
PhoneNumber string   // Would be "" (empty string), not NULL
```

**Step 3: Update Store query**

**File: `database/user.go`**

```go
func (r *UserRepository) Store(user User) (*User, error) {
    query := `
        INSERT INTO users (
            first_name, last_name, email, password,
            phone_number, date_of_birth, country, is_shop_owner
        )
        VALUES (
            :first_name, :last_name, :email, :password,
            :phone_number, :date_of_birth, :country, :is_shop_owner
        )
        RETURNING id
    `
    
    rows, err := r.DB.NamedQuery(query, &user)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    if rows.Next() {
        var userID int
        err = rows.Scan(&userID)
        if err != nil {
            return nil, err
        }
        user.ID = userID
    }
    
    return &user, nil
}
```

**Step 4: Update Find query**

```go
func (r *UserRepository) Find(email, password string) *User {
    var user User
    
    query := `
        SELECT 
            id, first_name, last_name, email, password,
            phone_number, date_of_birth, country,
            is_shop_owner, created_at, updated_at
        FROM users
        WHERE email = $1 AND password = $2
        LIMIT 1
    `
    
    err := r.DB.Get(&user, query, email, password)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil
        }
        fmt.Println("Error finding user:", err)
        return nil
    }
    
    return &user
}
```

**Step 5: Test with new fields**

```json
POST http://localhost:4000/api/users

{
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com",
  "password": "password123",
  "phone_number": "+8801712345678",
  "date_of_birth": "1990-05-15",
  "country": "USA",
  "is_shop_owner": false
}
```

**Response:**
```json
{
  "id": 2,
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com",
  "password": "password123",
  "phone_number": "+8801712345678",
  "date_of_birth": "1990-05-15T00:00:00Z",
  "country": "USA",
  "is_shop_owner": false,
  "created_at": "2024-01-15T11:00:00Z",
  "updated_at": "2024-01-15T11:00:00Z"
}
```

**Test nullable fields (omit phone and date of birth):**

```json
{
  "first_name": "Jane",
  "last_name": "Smith",
  "email": "jane@example.com",
  "password": "password456",
  "is_shop_owner": true
}
```

**Response (country gets default, others NULL):**
```json
{
  "id": 3,
  "first_name": "Jane",
  "last_name": "Smith",
  "email": "jane@example.com",
  "country": "Bangladesh",
  "is_shop_owner": true
}
```

✓ New fields work with nullable and default values!

</details>

---

### Question 2: Implement GetByID Method

**Question:**
Implement a `GetByID(id int) *User` method in UserRepository that retrieves a user by their ID. Handle the case where user doesn't exist.

<details>
<summary>Click to see answer</summary>

**Answer:**

**File: `database/user.go`**

```go
// GetByID retrieves user by ID
func (r *UserRepository) GetByID(id int) *User {
    var user User
    
    query := `
        SELECT 
            id, first_name, last_name, email, password,
            is_shop_owner, created_at, updated_at
        FROM users
        WHERE id = $1
    `
    
    err := r.DB.Get(&user, query, id)
    if err != nil {
        if err == sql.ErrNoRows {
            fmt.Printf("User with ID %d not found\n", id)
            return nil
        }
        fmt.Println("Error getting user by ID:", err)
        return nil
    }
    
    return &user
}
```

**Create handler:**

**File: `handler/user/get_user.go`**

```go
package user

import (
    "encoding/json"
    "net/http"
    "strconv"
)

func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    // Get ID from URL path
    idStr := r.PathValue("id")
    
    // Convert to integer
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    // Get user from database
    user := h.UserRepo.GetByID(id)
    if user == nil {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    // Don't send password in response!
    user.Password = ""
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

**Register route:**

**File: `handler/user/routes.go`**

```go
func (h *Handler) RegisterRoutes(
    mux *http.ServeMux,
    manager *middleware.Manager,
    middlewares *middleware.Middlewares,
) {
    // ... existing routes
    
    // New route: Get user by ID
    mux.HandleFunc("GET /api/users/{id}", h.GetUser)
}
```

**Test:**

```
GET http://localhost:4000/api/users/1
```

**Response:**
```json
{
  "id": 1,
  "first_name": "Habib",
  "last_name": "Rahman",
  "email": "habib@example.com",
  "password": "",
  "is_shop_owner": true,
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**Test non-existent user:**
```
GET http://localhost:4000/api/users/999
```

**Response:**
```
404 Not Found
User not found
```

✓ GetByID implemented with proper error handling!

</details>

---

### Question 3: Handle Duplicate Email Error

**Question:**
When trying to create a user with an email that already exists, PostgreSQL returns an error. Catch this error and return a user-friendly message instead of "Internal server error".

<details>
<summary>Click to see answer</summary>

**Answer:**

**Understanding the error:**

When inserting duplicate email:
```sql
INSERT INTO users (email, ...) VALUES ('existing@email.com', ...);
```

PostgreSQL returns:
```
pq: duplicate key value violates unique constraint "users_email_key"
```

**Solution: Check error type**

**File: `database/user.go`**

```go
package database

import (
    "database/sql"
    "fmt"
    "strings"
    
    "github.com/jmoiron/sqlx"
    "github.com/lib/pq"  // Import pq for error types
)

func (r *UserRepository) Store(user User) (*User, error) {
    query := `
        INSERT INTO users (first_name, last_name, email, password, is_shop_owner)
        VALUES (:first_name, :last_name, :email, :password, :is_shop_owner)
        RETURNING id
    `
    
    rows, err := r.DB.NamedQuery(query, &user)
    if err != nil {
        // Check if it's a PostgreSQL error
        if pqErr, ok := err.(*pq.Error); ok {
            // Check if it's unique constraint violation
            if pqErr.Code == "23505" {  // Unique violation error code
                // Check which field caused the error
                if strings.Contains(pqErr.Message, "email") {
                    return nil, fmt.Errorf("email already exists")
                }
                return nil, fmt.Errorf("duplicate entry")
            }
        }
        
        fmt.Println("Error creating user:", err)
        return nil, err
    }
    defer rows.Close()
    
    if rows.Next() {
        var userID int
        err = rows.Scan(&userID)
        if err != nil {
            return nil, err
        }
        user.ID = userID
    }
    
    return &user, nil
}
```

**PostgreSQL Error Codes:**

| Code | Error | Meaning |
|------|-------|---------|
| 23505 | unique_violation | Duplicate key |
| 23503 | foreign_key_violation | FK constraint failed |
| 23502 | not_null_violation | NULL in NOT NULL column |
| 42P01 | undefined_table | Table doesn't exist |

**Update handler:**

**File: `handler/user/create_user.go`**

```go
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var newUser database.User
    err := json.NewDecoder(r.Body).Decode(&newUser)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid request data"))
        return
    }
    
    // Store in database
    createdUser, err := h.UserRepo.Store(newUser)
    if err != nil {
        // Check for specific errors
        if strings.Contains(err.Error(), "email already exists") {
            w.WriteHeader(http.StatusConflict)
            w.Write([]byte("Email already registered"))
            return
        }
        
        // Generic error
        fmt.Println("Error creating user:", err)
        http.Error(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(createdUser)
}
```

**Test duplicate email:**

**First request:**
```json
POST http://localhost:4000/api/users

{
  "first_name": "Test",
  "last_name": "User",
  "email": "duplicate@test.com",
  "password": "password123"
}
```

**Response:** `201 Created`

**Second request (same email):**
```json
POST http://localhost:4000/api/users

{
  "first_name": "Another",
  "last_name": "User",
  "email": "duplicate@test.com",
  "password": "different"
}
```

**Response:**
```
409 Conflict
Email already registered
```

✓ User-friendly error message for duplicate email!

**Better error structure:**

```go
// Create custom error type
type DuplicateEmailError struct {
    Email string
}

func (e *DuplicateEmailError) Error() string {
    return fmt.Sprintf("email %s already exists", e.Email)
}

// In Store method:
if pqErr.Code == "23505" && strings.Contains(pqErr.Message, "email") {
    return nil, &DuplicateEmailError{Email: user.Email}
}

// In handler:
if _, ok := err.(*database.DuplicateEmailError); ok {
    w.WriteHeader(http.StatusConflict)
    json.NewEncoder(w).Encode(map[string]string{
        "error": "Email already registered",
        "field": "email",
    })
    return
}
```

This provides structured error responses for better client-side handling!

</details>

---

**Next Chapter Preview:**

In Chapter 54, we'll:
1. **Hash passwords** - bcrypt for security
2. **Update operations** - Edit user information
3. **Delete operations** - Remove users
4. **Products table** - Complete CRUD for products
5. **Transactions** - Ensure data consistency

Our database integration is working—now we'll make it **secure and complete**! 🚀
