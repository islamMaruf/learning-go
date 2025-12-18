# Chapter 56: CRUD Operations in Go - Complete Implementation

## Table of Contents
- [Introduction](#introduction)
- [From In-Memory to Database](#from-in-memory-to-database)
- [Creating Products Table](#creating-products-table)
- [Repository Pattern Recap](#repository-pattern-recap)
- [Implementing CREATE Operation](#implementing-create-operation)
- [Implementing READ Operations](#implementing-read-operations)
  - [Get Single Product](#get-single-product)
  - [List All Products](#list-all-products)
- [Implementing UPDATE Operation](#implementing-update-operation)
- [Implementing DELETE Operation](#implementing-delete-operation)
- [Understanding sqlx Methods](#understanding-sqlx-methods)
- [Integrating with Server](#integrating-with-server)
- [Testing with Postman](#testing-with-postman)
- [Why Repository Pattern is Powerful](#why-repository-pattern-is-powerful)
- [Summary](#summary)
- [Practice Questions](#practice-questions)

---

## Introduction

In Chapter 55, we learned SQL CRUD operations in pgAdmin. But that's not useful for our application—**we need CRUD operations in Go!**

**In this chapter:**
- ✓ Implement CREATE, READ, UPDATE, DELETE in Go
- ✓ Use sqlx library for database operations
- ✓ Replace in-memory storage with PostgreSQL
- ✓ Keep handler layer unchanged (Repository Pattern magic!)

**Why CRUD matters:**
```
Junior Engineer Job Requirements:
✓ CRUD operations
✓ REST API
✓ Database integration

If you master this chapter:
→ 80% chance of getting hired as junior developer!
→ You already know MORE than required
```

**What we'll build:**
```go
// Product Repository with real database
type ProductRepository struct {
    DB *sqlx.DB  // Database connection
}

// CRUD Operations
Create(product Product) (*Product, error)
Get(id int) (*Product, error)
List() ([]*Product, error)
Update(product Product) (*Product, error)
Delete(id int) error
```

Let's implement real database operations!

---

## From In-Memory to Database

### Current Problem (In-Memory)

**File: `database/product.go` (Before)**

```go
package database

type ProductRepository struct {
    // No database connection
}

var products []Product  // In-memory storage

func (r *ProductRepository) Create(product Product) Product {
    product.ID = len(products) + 1
    products = append(products, product)  // Lost on restart!
    return product
}
```

**Problems:**
- ❌ Data lost when server restarts
- ❌ No persistence
- ❌ Can't scale across multiple servers
- ❌ Not production-ready

### Solution (Database)

**File: `database/product.go` (After)**

```go
package database

import "github.com/jmoiron/sqlx"

type ProductRepository struct {
    DB *sqlx.DB  // Database connection
}

func NewProductRepository(db *sqlx.DB) *ProductRepository {
    return &ProductRepository{DB: db}
}

// No more in-memory array!
// Data persists in PostgreSQL
```

**Benefits:**
- ✓ Data persists forever
- ✓ Survives server restarts
- ✓ Scalable
- ✓ Production-ready

---

## Creating Products Table

### Table Schema

**File: `infra/db/queries/create_products_table.sql`**

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    price DOUBLE PRECISION NOT NULL,
    image_url TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Create index for faster queries
CREATE INDEX idx_products_title ON products(title);
```

**Why BIGSERIAL for products?**
```
Products lifecycle:
- Products added daily
- Products deleted when out of stock
- Products re-added
- Over years: billions of IDs possible

SERIAL (2.1B) might not be enough
BIGSERIAL (9.2 quintillion) will NEVER run out!
```

**Execute in pgAdmin:**
```
Right-click ecommerce database
→ Query Tool
→ Paste SQL
→ Execute (F5)
→ Refresh tables to see 'products'
```

### Product Struct with DB Tags

**File: `database/product.go`**

```go
package database

import "time"

type Product struct {
    ID          int64     `json:"id" db:"id"`
    Title       string    `json:"title" db:"title"`
    Description string    `json:"description" db:"description"`
    Price       float64   `json:"price" db:"price"`
    ImageURL    string    `json:"image_url" db:"image_url"`
    CreatedAt   time.Time `json:"created_at" db:"created_at"`
    UpdatedAt   time.Time `json:"updated_at" db:"updated_at"`
}
```

**Why `db` tags?**

```
Without db tags:
Go field: Title
Database column: title
MISMATCH! Won't work!

With db tags:
Go field: Title → db:"title" → Database column: title
MATCH! Works perfectly!
```

**Tag format:**
```go
`json:"api_name" db:"database_column_name"`
  ↓                ↓
  For API         For Database
```

---

## Repository Pattern Recap

### Why Repository Pattern?

**Visual representation:**

```
Before (without repository):
Handler → Direct SQL queries
Every handler has database code
Change database = Change EVERYWHERE!

After (with repository):
Handler → Repository → Database
Handlers don't know about database
Change database = Change ONLY repository!
```

### Repository Structure

```go
type ProductRepository struct {
    DB *sqlx.DB  // Injected dependency
}

// Constructor (Dependency Injection)
func NewProductRepository(db *sqlx.DB) *ProductRepository {
    return &ProductRepository{DB: db}
}
```

**Key concept:**
- Repository gets database connection through constructor
- Handlers don't need to know about database
- All database logic in one place

---

## Implementing CREATE Operation

### SQL INSERT Query

**What we want to do:**
```sql
INSERT INTO products (title, description, price, image_url)
VALUES ('Laptop', 'High-performance laptop', 999.99, 'http://...')
RETURNING id;
```

**Why `RETURNING id`?**
- PostgreSQL auto-generates ID (BIGSERIAL)
- We need to return the new ID to the client
- Client needs ID for future operations

### Create Method Implementation

**File: `database/product.go`**

```go
func (r *ProductRepository) Create(p Product) (*Product, error) {
    // SQL query with named parameters
    query := `
        INSERT INTO products (title, description, price, image_url)
        VALUES (:title, :description, :price, :image_url)
        RETURNING id
    `
    
    // Execute query with NamedQuery (maps struct fields to :placeholders)
    row, err := r.DB.NamedQuery(query, &p)
    if err != nil {
        return nil, err
    }
    defer row.Close()
    
    // Scan the returned ID
    if row.Next() {
        err = row.Scan(&p.ID)
        if err != nil {
            return nil, err
        }
    }
    
    return &p, nil
}
```

### Understanding NamedQuery

**Named parameters (colon syntax):**

```go
query := `
    INSERT INTO products (title, description, price, image_url)
    VALUES (:title, :description, :price, :image_url)
`
```

**How it works:**

```
:title       → Looks for field with db:"title" tag
:description → Looks for field with db:"description" tag
:price       → Looks for field with db:"price" tag
:image_url   → Looks for field with db:"image_url" tag

sqlx automatically maps struct fields to named parameters!
```

**Why better than positional parameters:**

```go
// ❌ Positional (easy to mess up order)
query := "INSERT INTO products VALUES ($1, $2, $3, $4)"
db.Exec(query, title, description, price, imageURL)
// What if you swap arguments? DISASTER!

// ✓ Named (clear and order-independent)
query := "INSERT INTO products VALUES (:title, :description, :price, :image_url)"
db.NamedQuery(query, product)
// Fields matched by name, order doesn't matter!
```

### RETURNING Clause

**Why use RETURNING?**

```sql
INSERT INTO products (title, price)
VALUES ('Laptop', 999.99)
RETURNING id;
```

**What happens:**
1. PostgreSQL inserts row
2. Generates ID (e.g., 1, 2, 3...)
3. Returns the generated ID
4. We capture it with `row.Scan(&p.ID)`

**Without RETURNING:**
```go
// Would need extra query:
db.Exec("INSERT INTO products ...")
var id int
db.QueryRow("SELECT lastval()").Scan(&id)  // Extra query!
```

**With RETURNING:**
```go
// One query does both:
row := db.NamedQuery("INSERT ... RETURNING id", product)
row.Scan(&p.ID)  // Done!
```

---

## Implementing READ Operations

### Get Single Product

**SQL SELECT query:**
```sql
SELECT id, title, description, price, image_url, created_at, updated_at
FROM products
WHERE id = $1;
```

**Implementation:**

**File: `database/product.go`**

```go
func (r *ProductRepository) Get(id int) (*Product, error) {
    var product Product
    
    query := `
        SELECT id, title, description, price, image_url, created_at, updated_at
        FROM products
        WHERE id = $1
    `
    
    // Get method: Query + Scan in one step
    err := r.DB.Get(&product, query, id)
    if err != nil {
        // Check if no rows found
        if err == sql.ErrNoRows {
            return nil, nil  // Product not found (not an error)
        }
        return nil, err  // Real error
    }
    
    return &product, nil
}
```

### Understanding DB.Get()

**What `Get` does:**

```go
err := r.DB.Get(&product, query, id)
```

**Steps:**
1. Execute query with parameters
2. Scan first row into struct
3. Return error if query fails or no rows

**Automatic mapping:**

```
Database columns → db tags → Struct fields

id          → db:"id"          → product.ID
title       → db:"title"       → product.Title
description → db:"description" → product.Description
price       → db:"price"       → product.Price
```

**Why pointer `&product`?**
```go
var product Product  // Empty struct

err := r.DB.Get(&product, query, id)
                  ↑
                  Pass address so Get can modify it
```

### List All Products

**SQL SELECT all:**
```sql
SELECT id, title, description, price, image_url, created_at, updated_at
FROM products;
```

**Implementation:**

```go
func (r *ProductRepository) List() ([]*Product, error) {
    var products []*Product  // Slice of pointers
    
    query := `
        SELECT id, title, description, price, image_url, created_at, updated_at
        FROM products
    `
    
    // Select method: Query multiple rows
    err := r.DB.Select(&products, query)
    if err != nil {
        return nil, err
    }
    
    return products, nil
}
```

### Pointer vs Value in Slices

**Why `[]*Product` not `[]Product`?**

```go
// ❌ Value slice (copies entire product)
func List() ([]Product, error) {
    var products []Product
    // Each Product struct copied into slice
    // If Product is 1 KB, 1000 products = 1 MB copied!
}

// ✓ Pointer slice (copies only addresses)
func List() ([]*Product, error) {
    var products []*Product
    // Only pointers copied (8 bytes each)
    // 1000 products = 8 KB copied
}
```

**Memory comparison:**

```
Product struct size: 200 bytes (example)
1000 products:

[]Product:
1000 × 200 bytes = 200 KB in memory
When returning: Another 200 KB copied
Total: 400 KB

[]*Product:
1000 × 8 bytes = 8 KB in memory (pointers)
When returning: Another 8 KB copied (pointers)
Total: 16 KB

Savings: 384 KB! 24x smaller!
```

**Slice contains:**
```go
[]*Product

Slice header (24 bytes):
- Pointer to array (8 bytes)
- Length (8 bytes)
- Capacity (8 bytes)

→ Copying slice header is cheap!
→ Original data stays in place
```

**When NOT to use pointers:**
```go
// Small structs (< 64 bytes): Value is fine
type Point struct {
    X, Y int
}
var points []Point  // OK, Point is small

// Large structs (> 64 bytes): Use pointers
type Product struct {
    // Many fields...
}
var products []*Product  // Better!
```

---

## Implementing UPDATE Operation

### SQL UPDATE Query

**What we want to do:**
```sql
UPDATE products
SET title = $1, description = $2, price = $3, image_url = $4
WHERE id = $5
RETURNING id;
```

**Implementation:**

**File: `database/product.go`**

```go
func (r *ProductRepository) Update(p Product) (*Product, error) {
    query := `
        UPDATE products
        SET title = $1,
            description = $2,
            price = $3,
            image_url = $4,
            updated_at = CURRENT_TIMESTAMP
        WHERE id = $5
        RETURNING id
    `
    
    // QueryRow for UPDATE with RETURNING
    row := r.DB.QueryRow(
        query,
        p.Title,
        p.Description,
        p.Price,
        p.ImageURL,
        p.ID,
    )
    
    // Get the returned ID (just for confirmation)
    var id int64
    err := row.Scan(&id)
    if err != nil {
        return nil, err
    }
    
    return &p, nil
}
```

### Positional Parameters ($1, $2...)

**Why positional here instead of named?**

```go
// Named parameters work with structs
db.NamedQuery(query, &product)  // Maps :field to struct.field

// Positional parameters for manual control
db.QueryRow(query, value1, value2, value3)  // $1, $2, $3
```

**Positional parameter mapping:**

```sql
UPDATE products
SET title = $1,        -- p.Title
    description = $2,  -- p.Description
    price = $3,        -- p.Price
    image_url = $4,    -- p.ImageURL
WHERE id = $5          -- p.ID
```

**Order matters!**
```go
// Must match query order exactly:
db.QueryRow(query, 
    p.Title,        // $1
    p.Description,  // $2
    p.Price,        // $3
    p.ImageURL,     // $4
    p.ID,           // $5
)
```

### Alternative: Named Parameters for UPDATE

**You can also use NamedQuery:**

```go
func (r *ProductRepository) Update(p Product) (*Product, error) {
    query := `
        UPDATE products
        SET title = :title,
            description = :description,
            price = :price,
            image_url = :image_url,
            updated_at = CURRENT_TIMESTAMP
        WHERE id = :id
        RETURNING id
    `
    
    row, err := r.DB.NamedQuery(query, &p)
    if err != nil {
        return nil, err
    }
    defer row.Close()
    
    if row.Next() {
        var id int64
        row.Scan(&id)
    }
    
    return &p, nil
}
```

**Both approaches work!** Choose what you prefer.

---

## Implementing DELETE Operation

### SQL DELETE Query

**What we want to do:**
```sql
DELETE FROM products
WHERE id = $1;
```

**Implementation:**

**File: `database/product.go`**

```go
func (r *ProductRepository) Delete(id int) error {
    query := `
        DELETE FROM products
        WHERE id = $1
    `
    
    // Exec for DELETE (no rows returned)
    _, err := r.DB.Exec(query, id)
    if err != nil {
        return err
    }
    
    return nil
}
```

### Understanding DB.Exec()

**What `Exec` does:**

```go
result, err := r.DB.Exec(query, id)
```

**Returns:**
- `result`: Contains rows affected count
- `err`: Error if query fails

**When to use Exec:**
- DELETE queries (no data returned)
- UPDATE without RETURNING
- INSERT without RETURNING

**Checking rows affected:**

```go
result, err := r.DB.Exec(query, id)
if err != nil {
    return err
}

rowsAffected, _ := result.RowsAffected()
if rowsAffected == 0 {
    return errors.New("product not found")
}
```

### Soft Delete Alternative

**Hard delete (permanent):**
```go
func (r *ProductRepository) Delete(id int) error {
    query := "DELETE FROM products WHERE id = $1"
    _, err := r.DB.Exec(query, id)
    return err
}
```

**Soft delete (recoverable):**
```go
func (r *ProductRepository) Delete(id int) error {
    query := `
        UPDATE products
        SET is_deleted = TRUE, deleted_at = NOW()
        WHERE id = $1
    `
    _, err := r.DB.Exec(query, id)
    return err
}

// List only active products
func (r *ProductRepository) List() ([]*Product, error) {
    query := `
        SELECT * FROM products
        WHERE is_deleted = FALSE
        ORDER BY created_at DESC
    `
    // ... rest of implementation
}
```

**Soft delete is preferred in production!**

---

## Understanding sqlx Methods

### Method Summary

| Method | Use Case | Returns | Example |
|--------|----------|---------|---------|
| **QueryRow** | Single row | `*sql.Row` | INSERT RETURNING, UPDATE RETURNING |
| **NamedQuery** | Named params | `*sqlx.Rows` | INSERT with struct |
| **Get** | Single row to struct | `error` | SELECT single product |
| **Select** | Multiple rows to slice | `error` | SELECT all products |
| **Exec** | No data returned | `sql.Result` | DELETE, UPDATE |

### When to Use Each Method

**1. QueryRow - Single row expected**

```go
// Use for: INSERT/UPDATE with RETURNING
row := db.QueryRow("INSERT INTO products ... RETURNING id", values...)
row.Scan(&id)
```

**2. NamedQuery - Insert with struct**

```go
// Use for: INSERT with struct (named parameters)
row, err := db.NamedQuery("INSERT INTO products ... VALUES (:title, :price)", &product)
```

**3. Get - Single row into struct**

```go
// Use for: SELECT single row
var product Product
err := db.Get(&product, "SELECT * FROM products WHERE id = $1", id)
```

**4. Select - Multiple rows into slice**

```go
// Use for: SELECT multiple rows
var products []*Product
err := db.Select(&products, "SELECT * FROM products")
```

**5. Exec - No rows returned**

```go
// Use for: DELETE, UPDATE without RETURNING
_, err := db.Exec("DELETE FROM products WHERE id = $1", id)
```

### Visual Decision Tree

```
Need to return data?
├─ YES → Data returned
│  ├─ Single row?
│  │  ├─ YES → Use QueryRow or Get
│  │  └─ NO  → Use Select
│  └─ Using struct fields?
│     ├─ YES → Use NamedQuery
│     └─ NO  → Use QueryRow
│
└─ NO → No data returned
   └─ Use Exec
```

---

## Integrating with Server

### Update cmd/serve.go

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
    productRepo := database.NewProductRepository(dbConnection)  // ← New!
    
    // Create handlers with repositories
    userHandler := user.NewHandler(userRepo)
    productHandler := product.NewHandler(productRepo)  // ← Inject repository
    
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
productHandler := product.NewHandler()

// New: With database
productRepo := database.NewProductRepository(dbConnection)
productHandler := product.NewHandler(productRepo)
```

### Update Product Handler

**File: `handler/product/handler.go`**

```go
package product

import (
    "yourproject/database"
)

type Handler struct {
    ProductRepo *database.ProductRepository  // Inject repository
}

func NewHandler(productRepo *database.ProductRepository) *Handler {
    return &Handler{
        ProductRepo: productRepo,
    }
}
```

**Handlers remain unchanged!**

```go
// handler/product/create_product.go
func (h *Handler) CreateProduct(w http.ResponseWriter, r *http.Request) {
    var newProduct database.Product
    json.NewDecoder(r.Body).Decode(&newProduct)
    
    // Repository handles database operations
    createdProduct, err := h.ProductRepo.Create(newProduct)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(createdProduct)
}
```

**Notice:** Handler code doesn't change! This is the power of Repository Pattern!

---

## Testing with Postman

### Start Server

```bash
go run main.go
```

**Expected output:**
```
Successfully connected to PostgreSQL!
Server running on port 4000
```

### Create Product

**Request:**
```
POST http://localhost:4000/api/products
Content-Type: application/json
Authorization: Bearer <your_jwt_token>

{
  "title": "Laptop",
  "description": "High-performance laptop for developers",
  "price": 999.99,
  "image_url": "https://example.com/laptop.jpg"
}
```

**Response:**
```json
{
  "id": 1,
  "title": "Laptop",
  "description": "High-performance laptop for developers",
  "price": 999.99,
  "image_url": "https://example.com/laptop.jpg",
  "created_at": "2024-01-20T10:30:00Z",
  "updated_at": "2024-01-20T10:30:00Z"
}
```

**Create more products:**
```json
{
  "title": "Mouse",
  "description": "Wireless gaming mouse",
  "price": 49.99,
  "image_url": "https://example.com/mouse.jpg"
}
```

```json
{
  "title": "Keyboard",
  "description": "Mechanical RGB keyboard",
  "price": 149.99,
  "image_url": "https://example.com/keyboard.jpg"
}
```

### Get Single Product

**Request:**
```
GET http://localhost:4000/api/products/1
```

**Response:**
```json
{
  "id": 1,
  "title": "Laptop",
  "description": "High-performance laptop for developers",
  "price": 999.99,
  "image_url": "https://example.com/laptop.jpg",
  "created_at": "2024-01-20T10:30:00Z",
  "updated_at": "2024-01-20T10:30:00Z"
}
```

### List All Products

**Request:**
```
GET http://localhost:4000/api/products
```

**Response:**
```json
[
  {
    "id": 1,
    "title": "Laptop",
    "price": 999.99,
    ...
  },
  {
    "id": 2,
    "title": "Mouse",
    "price": 49.99,
    ...
  },
  {
    "id": 3,
    "title": "Keyboard",
    "price": 149.99,
    ...
  }
]
```

### Update Product

**Request:**
```
PUT http://localhost:4000/api/products/2
Content-Type: application/json
Authorization: Bearer <your_jwt_token>

{
  "title": "Gaming Mouse",
  "description": "RGB gaming mouse with 16000 DPI",
  "price": 79.99,
  "image_url": "https://example.com/gaming-mouse.jpg"
}
```

**Response:**
```json
{
  "id": 2,
  "title": "Gaming Mouse",
  "description": "RGB gaming mouse with 16000 DPI",
  "price": 79.99,
  "image_url": "https://example.com/gaming-mouse.jpg",
  "updated_at": "2024-01-20T11:45:00Z"
}
```

### Delete Product

**Request:**
```
DELETE http://localhost:4000/api/products/1
Authorization: Bearer <your_jwt_token>
```

**Response:**
```json
{
  "message": "Product deleted successfully"
}
```

**Verify deletion:**
```
GET http://localhost:4000/api/products
```

**Result:** Product 1 no longer in list

### Verify in Database

**Check in pgAdmin:**
```sql
SELECT * FROM products;
```

**Result shows all operations persisted:**
```
 id | title        | description           | price  | created_at          
----+--------------+-----------------------+--------+---------------------
  2 | Gaming Mouse | RGB gaming mouse...   | 79.99  | 2024-01-20 10:32:00
  3 | Keyboard     | Mechanical RGB...     | 149.99 | 2024-01-20 10:33:00
```

✓ Product 1 deleted
✓ Product 2 updated with new price
✓ All changes persisted in database!

---

## Why Repository Pattern is Powerful

### One Place to Change

**Scenario:** Switch from PostgreSQL to MongoDB

**Without Repository Pattern:**
```
Need to change:
✓ CreateProduct handler (20 lines)
✓ GetProduct handler (15 lines)
✓ ListProducts handler (25 lines)
✓ UpdateProduct handler (30 lines)
✓ DeleteProduct handler (10 lines)

Total: 100+ lines across 5 files
Risk: Breaking handlers, inconsistencies
```

**With Repository Pattern:**
```
Need to change:
✓ ProductRepository only (80 lines)

Total: 80 lines in 1 file
Risk: Minimal, handlers unchanged
```

### Complete Isolation

**Visual representation:**

```
Handler Layer (Unchanged)
↓
Repository Interface (Contract)
↓
Repository Implementation (Changed)
↓
Database (PostgreSQL → MongoDB)
```

**Handlers don't know:**
- What database you use
- How queries are written
- Connection details
- Nothing about persistence!

**Repository handles everything:**
- Database connection
- Query construction
- Error handling
- Data mapping

### Real-World Example

**You implemented:**
```go
// In serve.go - only this changed:
productRepo := database.NewProductRepository(dbConnection)
productHandler := product.NewHandler(productRepo)
```

**Handlers stayed exactly the same:**
```go
// handler/product/create_product.go - NO CHANGES!
func (h *Handler) CreateProduct(w http.ResponseWriter, r *http.Request) {
    // Same code as before
    createdProduct, err := h.ProductRepo.Create(newProduct)
    // Works with database now!
}
```

**This is the power of Repository Pattern!** 🚀

---

## Summary

### What We Accomplished

**1. Created Products Table**
```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    price DOUBLE PRECISION NOT NULL,
    image_url TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**2. Implemented CRUD in Repository**
```go
Create(product Product) (*Product, error)  // NamedQuery
Get(id int) (*Product, error)              // Get
List() ([]*Product, error)                 // Select
Update(product Product) (*Product, error)  // QueryRow
Delete(id int) error                       // Exec
```

**3. Used Correct sqlx Methods**
```go
NamedQuery() // INSERT with struct (named params)
Get()        // SELECT single row into struct
Select()     // SELECT multiple rows into slice
QueryRow()   // INSERT/UPDATE with RETURNING
Exec()       // DELETE, UPDATE without RETURNING
```

**4. Integrated with Server**
```go
productRepo := database.NewProductRepository(dbConnection)
productHandler := product.NewHandler(productRepo)
```

**5. Tested Successfully**
- ✓ Create products → Persisted in database
- ✓ Get product → Retrieved from database
- ✓ List products → All products returned
- ✓ Update product → Changes saved
- ✓ Delete product → Removed from database

### Key Concepts

**Repository Pattern Benefits:**
1. **Isolation** - Database changes don't affect handlers
2. **Testability** - Mock repository for testing
3. **Maintainability** - All database code in one place
4. **Scalability** - Easy to switch databases

**sqlx Method Selection:**

| Operation | Method | Why |
|-----------|--------|-----|
| INSERT with struct | NamedQuery | Named parameters |
| SELECT single | Get | Auto-scans into struct |
| SELECT multiple | Select | Auto-scans into slice |
| UPDATE with RETURNING | QueryRow | Returns single row |
| DELETE | Exec | No data returned |

**Pointer vs Value:**
```go
[]*Product  // ✓ Good for large structs (save memory)
[]Product   // ❌ Bad for large structs (copies data)
```

### Before vs After

**Before (In-Memory):**
```go
var products []Product  // Lost on restart

func Create(p Product) Product {
    products = append(products, p)
    return p
}
```

**After (Database):**
```go
func (r *ProductRepository) Create(p Product) (*Product, error) {
    query := "INSERT INTO products ... RETURNING id"
    // Data persists forever in PostgreSQL
}
```

### Job-Ready Skills

**You now know:**
- ✓ CRUD operations in Go
- ✓ sqlx library usage
- ✓ Repository pattern
- ✓ Database integration
- ✓ REST API with persistence

**This qualifies you for junior developer positions!** 🎉

---

## Practice Questions

### Question 1: Add Soft Delete

**Question:**
Modify the Product table and repository to support soft delete instead of hard delete. Products should be marked as deleted but not actually removed from the database.

Requirements:
1. Add `is_deleted` and `deleted_at` columns to products table
2. Update Delete method to soft delete
3. Update List method to exclude deleted products
4. Add Restore method to un-delete products

<details>
<summary>Click to see answer</summary>

**Answer:**

**Step 1: Alter table**

```sql
-- Add soft delete columns
ALTER TABLE products
ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE,
ADD COLUMN deleted_at TIMESTAMPTZ;

-- Create index for faster queries
CREATE INDEX idx_products_is_deleted ON products(is_deleted);
```

**Step 2: Update Product struct**

**File: `database/product.go`**

```go
type Product struct {
    ID          int64      `json:"id" db:"id"`
    Title       string     `json:"title" db:"title"`
    Description string     `json:"description" db:"description"`
    Price       float64    `json:"price" db:"price"`
    ImageURL    string     `json:"image_url" db:"image_url"`
    IsDeleted   bool       `json:"is_deleted" db:"is_deleted"`
    DeletedAt   *time.Time `json:"deleted_at,omitempty" db:"deleted_at"`
    CreatedAt   time.Time  `json:"created_at" db:"created_at"`
    UpdatedAt   time.Time  `json:"updated_at" db:"updated_at"`
}
```

**Step 3: Update Delete method (soft delete)**

```go
func (r *ProductRepository) Delete(id int) error {
    query := `
        UPDATE products
        SET is_deleted = TRUE,
            deleted_at = CURRENT_TIMESTAMP
        WHERE id = $1 AND is_deleted = FALSE
    `
    
    result, err := r.DB.Exec(query, id)
    if err != nil {
        return err
    }
    
    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return errors.New("product not found or already deleted")
    }
    
    return nil
}
```

**Step 4: Update List method (exclude deleted)**

```go
func (r *ProductRepository) List() ([]*Product, error) {
    var products []*Product
    
    query := `
        SELECT id, title, description, price, image_url,
               is_deleted, deleted_at, created_at, updated_at
        FROM products
        WHERE is_deleted = FALSE
        ORDER BY created_at DESC
    `
    
    err := r.DB.Select(&products, query)
    if err != nil {
        return nil, err
    }
    
    return products, nil
}
```

**Step 5: Add Restore method**

```go
func (r *ProductRepository) Restore(id int) error {
    query := `
        UPDATE products
        SET is_deleted = FALSE,
            deleted_at = NULL
        WHERE id = $1 AND is_deleted = TRUE
    `
    
    result, err := r.DB.Exec(query, id)
    if err != nil {
        return err
    }
    
    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return errors.New("product not found or not deleted")
    }
    
    return nil
}
```

**Step 6: Add ListDeleted method (for admin)**

```go
func (r *ProductRepository) ListDeleted() ([]*Product, error) {
    var products []*Product
    
    query := `
        SELECT id, title, description, price, image_url,
               is_deleted, deleted_at, created_at, updated_at
        FROM products
        WHERE is_deleted = TRUE
        ORDER BY deleted_at DESC
    `
    
    err := r.DB.Select(&products, query)
    if err != nil {
        return nil, err
    }
    
    return products, nil
}
```

**Step 7: Create handler for restore**

**File: `handler/product/restore_product.go`**

```go
package product

import (
    "encoding/json"
    "net/http"
    "strconv"
)

func (h *Handler) RestoreProduct(w http.ResponseWriter, r *http.Request) {
    // Get ID from URL
    idStr := r.PathValue("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "Invalid product ID", http.StatusBadRequest)
        return
    }
    
    // Restore product
    err = h.ProductRepo.Restore(id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "message": "Product restored successfully",
    })
}
```

**Step 8: Register restore route**

**File: `handler/product/routes.go`**

```go
func (h *Handler) RegisterRoutes(
    mux *http.ServeMux,
    manager *middleware.Manager,
    middlewares *middleware.Middlewares,
) {
    // ... existing routes
    
    // Restore deleted product (admin only)
    mux.HandleFunc(
        "POST /api/products/{id}/restore",
        manager.With(h.RestoreProduct, middlewares.Auth, middlewares.AdminOnly),
    )
    
    // List deleted products (admin only)
    mux.HandleFunc(
        "GET /api/products/deleted",
        manager.With(h.ListDeletedProducts, middlewares.Auth, middlewares.AdminOnly),
    )
}
```

**Test soft delete:**

```bash
# Delete product
DELETE http://localhost:4000/api/products/1

# Verify not in list
GET http://localhost:4000/api/products
# Product 1 not shown

# Check in database (still there!)
SELECT * FROM products WHERE id = 1;
# is_deleted = true, deleted_at = timestamp

# Restore product
POST http://localhost:4000/api/products/1/restore

# Verify back in list
GET http://localhost:4000/api/products
# Product 1 shown again!
```

**Benefits of soft delete:**
- ✓ Can recover accidentally deleted products
- ✓ Maintain referential integrity (orders still reference products)
- ✓ Audit trail (know when products were deleted)
- ✓ Analytics (track deleted products)

</details>

---

### Question 2: Add Pagination

**Question:**
Add pagination support to the List method. Allow clients to request a specific page of products with a configurable page size.

Requirements:
1. Add pagination parameters (page, pageSize)
2. Return total count of products
3. Calculate and return pagination metadata
4. Add query optimization with LIMIT and OFFSET

<details>
<summary>Click to see answer</summary>

**Answer:**

**Step 1: Create pagination types**

**File: `database/pagination.go`**

```go
package database

type PaginationParams struct {
    Page     int `json:"page"`
    PageSize int `json:"page_size"`
}

type PaginationMetadata struct {
    CurrentPage  int `json:"current_page"`
    PageSize     int `json:"page_size"`
    TotalPages   int `json:"total_pages"`
    TotalRecords int `json:"total_records"`
    HasNext      bool `json:"has_next"`
    HasPrev      bool `json:"has_prev"`
}

type PaginatedProducts struct {
    Products   []*Product          `json:"products"`
    Pagination PaginationMetadata  `json:"pagination"`
}
```

**Step 2: Update List method with pagination**

**File: `database/product.go`**

```go
func (r *ProductRepository) List(params PaginationParams) (*PaginatedProducts, error) {
    // Default values
    if params.Page < 1 {
        params.Page = 1
    }
    if params.PageSize < 1 || params.PageSize > 100 {
        params.PageSize = 10  // Default page size
    }
    
    // Calculate offset
    offset := (params.Page - 1) * params.PageSize
    
    // Get total count
    var totalCount int
    countQuery := `
        SELECT COUNT(*)
        FROM products
        WHERE is_deleted = FALSE
    `
    err := r.DB.Get(&totalCount, countQuery)
    if err != nil {
        return nil, err
    }
    
    // Get paginated products
    var products []*Product
    query := `
        SELECT id, title, description, price, image_url,
               created_at, updated_at
        FROM products
        WHERE is_deleted = FALSE
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
    `
    
    err = r.DB.Select(&products, query, params.PageSize, offset)
    if err != nil {
        return nil, err
    }
    
    // Calculate pagination metadata
    totalPages := (totalCount + params.PageSize - 1) / params.PageSize
    
    metadata := PaginationMetadata{
        CurrentPage:  params.Page,
        PageSize:     params.PageSize,
        TotalPages:   totalPages,
        TotalRecords: totalCount,
        HasNext:      params.Page < totalPages,
        HasPrev:      params.Page > 1,
    }
    
    return &PaginatedProducts{
        Products:   products,
        Pagination: metadata,
    }, nil
}
```

**Step 3: Update handler to accept pagination**

**File: `handler/product/list_products.go`**

```go
package product

import (
    "encoding/json"
    "net/http"
    "strconv"
    
    "yourproject/database"
)

func (h *Handler) ListProducts(w http.ResponseWriter, r *http.Request) {
    // Parse query parameters
    pageStr := r.URL.Query().Get("page")
    pageSizeStr := r.URL.Query().Get("page_size")
    
    page, _ := strconv.Atoi(pageStr)
    pageSize, _ := strconv.Atoi(pageSizeStr)
    
    params := database.PaginationParams{
        Page:     page,
        PageSize: pageSize,
    }
    
    // Get paginated products
    result, err := h.ProductRepo.List(params)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(result)
}
```

**Test pagination:**

```bash
# Page 1, 10 products per page
GET http://localhost:4000/api/products?page=1&page_size=10
```

**Response:**
```json
{
  "products": [
    { "id": 10, "title": "Product 10", ... },
    { "id": 9, "title": "Product 9", ... },
    ...
    { "id": 1, "title": "Product 1", ... }
  ],
  "pagination": {
    "current_page": 1,
    "page_size": 10,
    "total_pages": 5,
    "total_records": 50,
    "has_next": true,
    "has_prev": false
  }
}
```

```bash
# Page 2, 10 products per page
GET http://localhost:4000/api/products?page=2&page_size=10
```

**Response:**
```json
{
  "products": [
    { "id": 20, "title": "Product 20", ... },
    ...
    { "id": 11, "title": "Product 11", ... }
  ],
  "pagination": {
    "current_page": 2,
    "page_size": 10,
    "total_pages": 5,
    "total_records": 50,
    "has_next": true,
    "has_prev": true
  }
}
```

**Query optimization:**

```sql
-- Without index (slow)
SELECT * FROM products ORDER BY created_at DESC LIMIT 10 OFFSET 100;
-- Scans all rows up to offset

-- With index (fast)
CREATE INDEX idx_products_created_at ON products(created_at DESC);
-- Uses index to jump directly to offset
```

**Performance comparison:**

```
1 million products, page 1000 (offset 10,000):

Without index:
- Scans 10,000+ rows
- Time: ~500ms

With index:
- Jumps directly using index
- Time: ~5ms

100x faster!
```

</details>

---

### Question 3: Add Search Functionality

**Question:**
Implement search functionality that allows users to search products by title or description. Support case-insensitive partial matching.

Requirements:
1. Add Search method to repository
2. Support searching by title or description
3. Case-insensitive search
4. Combine search with pagination
5. Use full-text search for better performance

<details>
<summary>Click to see answer</summary>

**Answer:**

**Step 1: Basic search with ILIKE**

**File: `database/product.go`**

```go
func (r *ProductRepository) Search(searchTerm string, params PaginationParams) (*PaginatedProducts, error) {
    // Default pagination
    if params.Page < 1 {
        params.Page = 1
    }
    if params.PageSize < 1 || params.PageSize > 100 {
        params.PageSize = 10
    }
    
    offset := (params.Page - 1) * params.PageSize
    
    // Prepare search pattern
    searchPattern := "%" + searchTerm + "%"
    
    // Get total count
    var totalCount int
    countQuery := `
        SELECT COUNT(*)
        FROM products
        WHERE is_deleted = FALSE
        AND (title ILIKE $1 OR description ILIKE $1)
    `
    err := r.DB.Get(&totalCount, countQuery, searchPattern)
    if err != nil {
        return nil, err
    }
    
    // Get products
    var products []*Product
    query := `
        SELECT id, title, description, price, image_url,
               created_at, updated_at
        FROM products
        WHERE is_deleted = FALSE
        AND (title ILIKE $1 OR description ILIKE $1)
        ORDER BY created_at DESC
        LIMIT $2 OFFSET $3
    `
    
    err = r.DB.Select(&products, query, searchPattern, params.PageSize, offset)
    if err != nil {
        return nil, err
    }
    
    // Pagination metadata
    totalPages := (totalCount + params.PageSize - 1) / params.PageSize
    
    metadata := PaginationMetadata{
        CurrentPage:  params.Page,
        PageSize:     params.PageSize,
        TotalPages:   totalPages,
        TotalRecords: totalCount,
        HasNext:      params.Page < totalPages,
        HasPrev:      params.Page > 1,
    }
    
    return &PaginatedProducts{
        Products:   products,
        Pagination: metadata,
    }, nil
}
```

**Step 2: Improved search with full-text search**

```sql
-- Create full-text search index
ALTER TABLE products
ADD COLUMN search_vector tsvector;

-- Create function to update search vector
CREATE OR REPLACE FUNCTION products_search_vector_update() RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER products_search_vector_trigger
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION products_search_vector_update();

-- Update existing rows
UPDATE products SET search_vector = 
    setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(description, '')), 'B');

-- Create GIN index for fast full-text search
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
```

**Step 3: Use full-text search in repository**

```go
func (r *ProductRepository) SearchFullText(searchTerm string, params PaginationParams) (*PaginatedProducts, error) {
    // Default pagination
    if params.Page < 1 {
        params.Page = 1
    }
    if params.PageSize < 1 || params.PageSize > 100 {
        params.PageSize = 10
    }
    
    offset := (params.Page - 1) * params.PageSize
    
    // Prepare search query
    searchQuery := plainto_tsquery('english', searchTerm)
    
    // Get total count
    var totalCount int
    countQuery := `
        SELECT COUNT(*)
        FROM products
        WHERE is_deleted = FALSE
        AND search_vector @@ plainto_tsquery('english', $1)
    `
    err := r.DB.Get(&totalCount, countQuery, searchTerm)
    if err != nil {
        return nil, err
    }
    
    // Get products with relevance ranking
    var products []*Product
    query := `
        SELECT 
            id, title, description, price, image_url,
            created_at, updated_at,
            ts_rank(search_vector, plainto_tsquery('english', $1)) AS rank
        FROM products
        WHERE is_deleted = FALSE
        AND search_vector @@ plainto_tsquery('english', $1)
        ORDER BY rank DESC, created_at DESC
        LIMIT $2 OFFSET $3
    `
    
    err = r.DB.Select(&products, query, searchTerm, params.PageSize, offset)
    if err != nil {
        return nil, err
    }
    
    // Pagination metadata
    totalPages := (totalCount + params.PageSize - 1) / params.PageSize
    
    metadata := PaginationMetadata{
        CurrentPage:  params.Page,
        PageSize:     params.PageSize,
        TotalPages:   totalPages,
        TotalRecords: totalCount,
        HasNext:      params.Page < totalPages,
        HasPrev:      params.Page > 1,
    }
    
    return &PaginatedProducts{
        Products:   products,
        Pagination: metadata,
    }, nil
}
```

**Step 4: Create search handler**

**File: `handler/product/search_products.go`**

```go
package product

import (
    "encoding/json"
    "net/http"
    "strconv"
    
    "yourproject/database"
)

func (h *Handler) SearchProducts(w http.ResponseWriter, r *http.Request) {
    // Get search term
    searchTerm := r.URL.Query().Get("q")
    if searchTerm == "" {
        http.Error(w, "Search term required", http.StatusBadRequest)
        return
    }
    
    // Get pagination params
    pageStr := r.URL.Query().Get("page")
    pageSizeStr := r.URL.Query().Get("page_size")
    
    page, _ := strconv.Atoi(pageStr)
    pageSize, _ := strconv.Atoi(pageSizeStr)
    
    params := database.PaginationParams{
        Page:     page,
        PageSize: pageSize,
    }
    
    // Search products
    result, err := h.ProductRepo.SearchFullText(searchTerm, params)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(result)
}
```

**Step 5: Register search route**

```go
// handler/product/routes.go
mux.HandleFunc("GET /api/products/search", h.SearchProducts)
```

**Test search:**

```bash
# Search for "laptop"
GET http://localhost:4000/api/products/search?q=laptop&page=1&page_size=10
```

**Response:**
```json
{
  "products": [
    {
      "id": 1,
      "title": "Gaming Laptop",
      "description": "High-performance laptop for gaming",
      "price": 1499.99
    },
    {
      "id": 5,
      "title": "Business Laptop",
      "description": "Professional laptop for business users",
      "price": 899.99
    }
  ],
  "pagination": {
    "current_page": 1,
    "page_size": 10,
    "total_pages": 1,
    "total_records": 2,
    "has_next": false,
    "has_prev": false
  }
}
```

**Performance comparison:**

```
1 million products, search "laptop":

ILIKE search:
SELECT * FROM products WHERE title ILIKE '%laptop%'
- Full table scan
- Time: ~2000ms

Full-text search with GIN index:
SELECT * FROM products WHERE search_vector @@ plainto_tsquery('laptop')
- Uses GIN index
- Time: ~20ms

100x faster!
```

**Advanced features:**

```go
// Fuzzy search (typo tolerance)
searchQuery := "laptop:*"  // Matches "laptops", "laptop's"

// Search with operators
searchQuery := "gaming & laptop"  // Must have both words
searchQuery := "gaming | laptop"  // Either word
searchQuery := "gaming & !cheap"  // Gaming but not cheap

// Phrase search
searchQuery := "high performance laptop"  // Exact phrase
```

</details>

---

**Next Chapter Preview:**

In Chapter 57, we'll cover:
1. **Advanced queries** - JOIN operations, GROUP BY, aggregations
2. **Transactions** - ACID properties, BEGIN/COMMIT/ROLLBACK
3. **Error handling** - Database-specific errors, recovery
4. **Query optimization** - Indexes, EXPLAIN ANALYZE
5. **Database migrations** - Schema versioning with golang-migrate

We've mastered CRUD—now let's become **database experts**! 🚀
