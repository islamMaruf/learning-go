# Chapter 61: DDD Into Code (Part 2) - Product Domain

## Table of Contents
- [Introduction](#introduction)
- [Refactoring Product Domain](#refactoring-product-domain)
- [Creating Product Repository](#creating-product-repository)
- [Defining Port Interfaces](#defining-port-interfaces)
- [Implementing Product Service](#implementing-product-service)
- [Updating Product Handler](#updating-product-handler)
- [Wiring Everything Together](#wiring-everything-together)
- [Testing Product Domain](#testing-product-domain)
- [Why This Works Better](#why-this-works-better)
- [When to Split Files](#when-to-split-files)
- [Industry Standards Achieved](#industry-standards-achieved)
- [Preparing for Unit Testing](#preparing-for-unit-testing)
- [Engineer vs Developer](#engineer-vs-developer)
- [Summary](#summary)
- [What's Next](#whats-next)

---

## Introduction

In Chapter 60, we refactored the User domain. Now it's time to apply the same DDD pattern to the Product domain!

**What we'll do:**
```
Before:
ProductHandler → ProductRepository → Database
(Tight coupling, parent accessing child directly)

After:
ProductHandler → ProductService → ProductRepository → Database
(Loose coupling, dependencies through interfaces)
```

**Same pattern, different domain:**
- User domain ✓ (Done in Chapter 60)
- Product domain ← (This chapter!)
- Future domains (Order, Payment, etc.)

**The goal:**
> **"One file changed = One file affected. NOT 10,000 files!"**

Let's transform the Product domain!

---

## Refactoring Product Domain

### Step 1: Update Product Entity in Domain

**File: `domain/product.go`** (Already exists from Chapter 60)

```go
package domain

import "time"

// Product entity represents a product in the system
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

**Why in domain package?**

```
Product entity is shared across:
- Product domain service
- Product repository
- Product handler
- Maybe Order domain (needs product info)

Shared entity = Lives in domain/ package
Domain-specific logic = Lives in domain/product/ package
```

### Step 2: Create Product Domain Structure

**Directory structure:**

```
domain/
├── user.go                # User entity
├── product.go             # Product entity
├── user/
│   ├── port.go           # User interfaces
│   └── service.go        # User business logic
└── product/
    ├── port.go           # Product interfaces ← Create this
    └── service.go        # Product business logic ← Create this
```

---

## Creating Product Repository

### Update Repository to Use Domain Package

**File: `database/product.go`**

**Before (Bad):**

```go
package database

// Local Product struct (WRONG!)
type Product struct {
    ID          int64
    Title       string
    Description string
    Price       float64
}

type ProductRepository struct {
    DB *sqlx.DB
}

func (r *ProductRepository) Create(product *Product) (*Product, error) {
    // Uses local Product
}
```

**After (Good):**

```go
package database

import (
    "github.com/jmoiron/sqlx"
    "yourproject/domain"
    productDomain "yourproject/domain/product"
)

// ProductRepository implements productDomain.Repository
type ProductRepository struct {
    DB *sqlx.DB
}

// NewProductRepository returns interface, not concrete struct
func NewProductRepository(db *sqlx.DB) productDomain.Repository {
    return &ProductRepository{DB: db}
}

// Create implements productDomain.Repository.Create
func (r *ProductRepository) Create(product *domain.Product) (*domain.Product, error) {
    query := `
        INSERT INTO products (title, description, price, image_url)
        VALUES (:title, :description, :price, :image_url)
        RETURNING id, created_at, updated_at
    `
    
    rows, err := r.DB.NamedQuery(query, product)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    if rows.Next() {
        err = rows.Scan(&product.ID, &product.CreatedAt, &product.UpdatedAt)
        if err != nil {
            return nil, err
        }
    }
    
    return product, nil
}

// Get implements productDomain.Repository.Get
func (r *ProductRepository) Get(id int64) (*domain.Product, error) {
    var product domain.Product
    
    query := "SELECT * FROM products WHERE id = $1"
    err := r.DB.Get(&product, query, id)
    
    if err == sql.ErrNoRows {
        return nil, nil
    }
    
    if err != nil {
        return nil, err
    }
    
    return &product, nil
}

// List implements productDomain.Repository.List
func (r *ProductRepository) List() ([]*domain.Product, error) {
    var products []*domain.Product
    
    query := "SELECT * FROM products ORDER BY created_at DESC"
    err := r.DB.Select(&products, query)
    
    if err != nil {
        return nil, err
    }
    
    return products, nil
}

// Update implements productDomain.Repository.Update
func (r *ProductRepository) Update(product *domain.Product) (*domain.Product, error) {
    query := `
        UPDATE products 
        SET title = :title, 
            description = :description, 
            price = :price, 
            image_url = :image_url,
            updated_at = NOW()
        WHERE id = :id
        RETURNING updated_at
    `
    
    rows, err := r.DB.NamedQuery(query, product)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    if rows.Next() {
        err = rows.Scan(&product.UpdatedAt)
        if err != nil {
            return nil, err
        }
    }
    
    return product, nil
}

// Delete implements productDomain.Repository.Delete
func (r *ProductRepository) Delete(id int64) error {
    query := "DELETE FROM products WHERE id = $1"
    _, err := r.DB.Exec(query, id)
    return err
}
```

**Key changes:**

```go
// 1. Import domain package
import "yourproject/domain"

// 2. Use domain.Product, not local Product
func (r *ProductRepository) Create(product *domain.Product) (*domain.Product, error) {
    //                                       ↑
    //                              From domain package
}

// 3. Return interface from constructor
func NewProductRepository(db *sqlx.DB) productDomain.Repository {
    //                                   ↑
    //                           Interface, not struct!
    return &ProductRepository{DB: db}
}

// 4. Implements productDomain.Repository interface
// (Will define this next in port.go)
```

---

## Defining Port Interfaces

### Create Product Port.go

**File: `domain/product/port.go`**

```go
package product

import "yourproject/domain"

// Repository defines what service needs from data layer
type Repository interface {
    Create(product *domain.Product) (*domain.Product, error)
    Get(id int64) (*domain.Product, error)
    List() ([]*domain.Product, error)
    Update(product *domain.Product) (*domain.Product, error)
    Delete(id int64) error
}

// Service defines what handlers need from product domain
type Service interface {
    Create(product *domain.Product) (*domain.Product, error)
    Get(id int64) (*domain.Product, error)
    List() ([]*domain.Product, error)
    Update(product *domain.Product) (*domain.Product, error)
    Delete(id int64) error
}
```

**Interface breakdown:**

```
Repository interface:
┌─────────────────────────────────────────┐
│ Create(product) → product, error        │
│ Get(id) → product, error                │
│ List() → []product, error               │
│ Update(product) → product, error        │
│ Delete(id) → error                      │
└─────────────────────────────────────────┘
    ↑
    │ (Service depends on this)
    │
Service interface:
┌─────────────────────────────────────────┐
│ Create(product) → product, error        │
│ Get(id) → product, error                │
│ List() → []product, error               │
│ Update(product) → product, error        │
│ Delete(id) → error                      │
└─────────────────────────────────────────┘
    ↑
    │ (Handler depends on this)
    │
Handler uses Service interface
```

**Why same signatures?**

```go
// Service and Repository have same signatures here
// BUT they won't always match!

Service might add:
- Business rules validation
- Permission checks
- Logging
- Caching
- Events

That's why we keep them separate!

Example future change:
Service.Create might become:
Create(product, userID, permissions) → product, error

Repository.Create stays:
Create(product) → product, error

Handler unchanged! (Uses Service interface)
```

---

## Implementing Product Service

### Create Product Service.go

**File: `domain/product/service.go`**

```go
package product

import "yourproject/domain"

// service implements Service interface
type service struct {
    productRepo Repository
}

// NewService creates new product service
func NewService(repo Repository) Service {
    return &service{
        productRepo: repo,
    }
}

// Create implements Service.Create
func (s *service) Create(product *domain.Product) (*domain.Product, error) {
    // Business logic can go here
    // For now, delegate to repository
    return s.productRepo.Create(product)
}

// Get implements Service.Get
func (s *service) Get(id int64) (*domain.Product, error) {
    return s.productRepo.Get(id)
}

// List implements Service.List
func (s *service) List() ([]*domain.Product, error) {
    return s.productRepo.List()
}

// Update implements Service.Update
func (s *service) Update(product *domain.Product) (*domain.Product, error) {
    return s.productRepo.Update(product)
}

// Delete implements Service.Delete
func (s *service) Delete(id int64) error {
    return s.productRepo.Delete(id)
}
```

**Pattern explanation:**

```go
// 1. Private struct (lowercase)
type service struct {
    productRepo Repository  // Depends on interface, not concrete type
}

// 2. Public constructor returning interface
func NewService(repo Repository) Service {
    //            ↑                   ↑
    //      Takes interface    Returns interface
    return &service{
        productRepo: repo,
    }
}

// 3. Methods implement Service interface
func (s *service) Create(product *domain.Product) (*domain.Product, error) {
    // Currently just delegates to repository
    // But can add business logic later!
    return s.productRepo.Create(product)
}
```

**Why "just delegate" to repository?**

```
Right now: Simple delegation
service.Create → repository.Create

Future: Add business logic
service.Create → 
    1. Validate product data
    2. Check user permissions
    3. Call repository.Create
    4. Log the creation
    5. Send event notification
    6. Update cache
    7. Return result

Repository stays unchanged!
Handler stays unchanged!
Only service.go changes!

This is the POWER of DDD! 💪
```

**Instructor's shortcut explanation:**

> **"You might say: 'Why create service if it just delegates?' Because in the future, we'll add business logic here! I'm using shortcuts now because I know where this is going. It's like chatting with girls—if you chat, you learn shortcuts naturally. If you don't chat, you won't get shortcuts!" 😄**

---

## Updating Product Handler

### Create Handler Port.go

**File: `handler/product/port.go`** (New file!)

```go
package product

import (
    "yourproject/domain"
    productDomain "yourproject/domain/product"
)

// Service defines what handler needs from product domain
type Service interface {
    productDomain.Service  // Embed domain's Service interface
}
```

**Why embed?**

```go
// Instead of duplicating:
type Service interface {
    Create(product *domain.Product) (*domain.Product, error)
    Get(id int64) (*domain.Product, error)
    List() ([]*domain.Product, error)
    Update(product *domain.Product) (*domain.Product, error)
    Delete(id int64) error
}

// Just embed:
type Service interface {
    productDomain.Service  // Gets all methods automatically!
}

Benefits:
✓ Single source of truth
✓ Domain defines interface
✓ Handler automatically gets updates
✓ No duplication
```

### Update Product Handler

**File: `handler/product/handler.go`**

**Before (Bad):**

```go
package product

import "yourproject/database"

type Handler struct {
    ProductRepo *database.ProductRepository  // ❌ Direct child access!
}

func (h *Handler) Create(c *gin.Context) {
    var product database.Product
    // ... bind data
    
    createdProduct, err := h.ProductRepo.Create(&product)  // ❌ Direct repo access
    // ...
}
```

**After (Good):**

```go
package product

import (
    "net/http"
    "strconv"
    
    "github.com/gin-gonic/gin"
    "yourproject/domain"
    productDomain "yourproject/domain/product"
)

type Handler struct {
    productService productDomain.Service  // ✓ Access through interface!
}

func NewHandler(service productDomain.Service) *Handler {
    return &Handler{
        productService: service,
    }
}

// Create handles product creation
func (h *Handler) Create(c *gin.Context) {
    var product domain.Product
    
    if err := c.ShouldBindJSON(&product); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // Use service, not repository!
    createdProduct, err := h.productService.Create(&product)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusCreated, createdProduct)
}

// Get handles getting a single product
func (h *Handler) Get(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid product ID"})
        return
    }
    
    product, err := h.productService.Get(id)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    if product == nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "Product not found"})
        return
    }
    
    c.JSON(http.StatusOK, product)
}

// List handles listing all products
func (h *Handler) List(c *gin.Context) {
    products, err := h.productService.List()
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, products)
}

// Update handles product updates
func (h *Handler) Update(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid product ID"})
        return
    }
    
    var product domain.Product
    if err := c.ShouldBindJSON(&product); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    product.ID = id
    
    updatedProduct, err := h.productService.Update(&product)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, updatedProduct)
}

// Delete handles product deletion
func (h *Handler) Delete(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid product ID"})
        return
    }
    
    err = h.productService.Delete(id)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{"message": "Product deleted successfully"})
}
```

**Key changes:**

```go
// 1. Rename import to avoid conflict
import (
    productDomain "yourproject/domain/product"
    //  ↑
    //  Handler package is "product"
    //  Domain package is also "product"
    //  Rename to avoid conflict!
)

// 2. Depend on Service interface, not Repository
type Handler struct {
    productService productDomain.Service  // Interface!
}

// 3. Use domain.Product everywhere
var product domain.Product  // From shared domain package

// 4. Call service methods
h.productService.Create(&product)  // Through interface!
```

---

## Wiring Everything Together

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
    userDomain "yourproject/domain/user"
    productDomain "yourproject/domain/product"
    "yourproject/handler/user"
    "yourproject/handler/product"
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
    
    // 3. Run migrations
    err = db.MigrateDB(dbConnection.DB, "./migrations")
    if err != nil {
        fmt.Println("Failed to run migrations:", err)
        os.Exit(1)
    }
    
    fmt.Println("Successfully migrated database!")
    
    // 4. Create repositories
    userRepo := database.NewUserRepository(dbConnection)
    productRepo := database.NewProductRepository(dbConnection)
    
    // 5. Create domain services
    userService := userDomain.NewService(userRepo)
    productService := productDomain.NewService(productRepo)
    
    // 6. Create handlers
    userHandler := user.NewHandler(userService)
    productHandler := product.NewHandler(productService)
    
    // 7. Start server
    srv := server.NewServer(cfg, userHandler, productHandler)
    srv.Start()
    
    fmt.Printf("Server running on port %s\n", cfg.Server.Port)
}
```

**Dependency injection flow:**

```
Database Connection
    ↓
Repositories (implement domain Repository interfaces)
    ├── UserRepository
    └── ProductRepository
    ↓
Domain Services (implement domain Service interfaces)
    ├── UserService (depends on UserRepository interface)
    └── ProductService (depends on ProductRepository interface)
    ↓
Handlers (depend on domain Service interfaces)
    ├── UserHandler (depends on UserService interface)
    └── ProductHandler (depends on ProductService interface)
    ↓
Server (wires handlers to routes)
    ↓
Start listening on port 4000
```

**Type magic:**

```go
// Step 4: Create repository
productRepo := database.NewProductRepository(dbConnection)
// Declared type: productDomain.Repository (interface)
// Actual type: *database.ProductRepository (struct)

// Step 5: Create service
productService := productDomain.NewService(productRepo)
//                                         ↑
//                              Passes Repository interface
// Declared type: productDomain.Service (interface)
// Actual type: *product.service (struct)

// Step 6: Create handler
productHandler := product.NewHandler(productService)
//                                   ↑
//                           Passes Service interface
// Type: *product.Handler (struct)
// Contains: productDomain.Service (interface field)
```

---

## Testing Product Domain

### Run the Application

```bash
go run main.go
```

**Expected output:**

```
Successfully connected to PostgreSQL!
Successfully migrated database!
Server running on port 4000
```

### Test Product Creation

**Request:**

```bash
POST http://localhost:4000/api/products
Content-Type: application/json

{
  "title": "MacBook Pro",
  "description": "Powerful laptop for developers",
  "price": 2499.99,
  "image_url": "https://example.com/macbook.jpg"
}
```

**Response:**

```json
{
  "id": 1,
  "title": "MacBook Pro",
  "description": "Powerful laptop for developers",
  "price": 2499.99,
  "image_url": "https://example.com/macbook.jpg",
  "created_at": "2024-01-20T10:00:00Z",
  "updated_at": "2024-01-20T10:00:00Z"
}
```

### Test Product Listing

**Request:**

```bash
GET http://localhost:4000/api/products
```

**Response:**

```json
[
  {
    "id": 1,
    "title": "MacBook Pro",
    "description": "Powerful laptop for developers",
    "price": 2499.99,
    "image_url": "https://example.com/macbook.jpg",
    "created_at": "2024-01-20T10:00:00Z"
  }
]
```

### Test Get Single Product

**Request:**

```bash
GET http://localhost:4000/api/products/1
```

**Response:**

```json
{
  "id": 1,
  "title": "MacBook Pro",
  "description": "Powerful laptop for developers",
  "price": 2499.99,
  "image_url": "https://example.com/macbook.jpg",
  "created_at": "2024-01-20T10:00:00Z"
}
```

### Test Product Update

**Request:**

```bash
PUT http://localhost:4000/api/products/1
Content-Type: application/json

{
  "title": "MacBook Pro M3",
  "description": "Latest M3 chip for ultimate performance",
  "price": 2999.99,
  "image_url": "https://example.com/macbook-m3.jpg"
}
```

**Response:**

```json
{
  "id": 1,
  "title": "MacBook Pro M3",
  "description": "Latest M3 chip for ultimate performance",
  "price": 2999.99,
  "image_url": "https://example.com/macbook-m3.jpg",
  "updated_at": "2024-01-20T11:00:00Z"
}
```

### Test Product Deletion

**Request:**

```bash
DELETE http://localhost:4000/api/products/1
```

**Response:**

```json
{
  "message": "Product deleted successfully"
}
```

✓ **Everything works perfectly!**

---

## Why This Works Better

### Change Impact Analysis

**Scenario: Change database from PostgreSQL to MongoDB**

**Without DDD:**

```
Change database/product.go:
    ↓
Breaks handler/product/handler.go (imports database package)
    ↓
Breaks router (handler signature changed)
    ↓
Breaks cmd/serve.go (initialization changed)
    ↓
Breaks tests (everything changed)

Files to modify: 50+ files! 😱
```

**With DDD:**

```
Change database/product.go:
    ↓
domain/product/service.go still works (uses Repository interface)
    ↓
handler/product/handler.go still works (uses Service interface)
    ↓
router still works (handler unchanged)
    ↓
cmd/serve.go still works (interfaces unchanged)

Files to modify: 1 file (database/product.go)! 🎉
```

### Example: Switch to MongoDB

**File: `database/product_mongo.go`** (New implementation)

```go
package database

import (
    "context"
    
    "go.mongodb.org/mongo-driver/mongo"
    "yourproject/domain"
    productDomain "yourproject/domain/product"
)

// ProductMongoRepository implements productDomain.Repository
type ProductMongoRepository struct {
    Collection *mongo.Collection
}

// NewProductMongoRepository creates MongoDB repository
func NewProductMongoRepository(db *mongo.Database) productDomain.Repository {
    return &ProductMongoRepository{
        Collection: db.Collection("products"),
    }
}

// Create implements productDomain.Repository.Create
func (r *ProductMongoRepository) Create(product *domain.Product) (*domain.Product, error) {
    // MongoDB implementation
    result, err := r.Collection.InsertOne(context.Background(), product)
    if err != nil {
        return nil, err
    }
    
    product.ID = result.InsertedID.(int64)
    return product, nil
}

// ... other methods
```

**Change in cmd/serve.go:**

```go
// Before: PostgreSQL
productRepo := database.NewProductRepository(dbConnection)

// After: MongoDB (ONE LINE CHANGE!)
productRepo := database.NewProductMongoRepository(mongoDatabase)

// Everything else stays exactly the same! 🎉
productService := productDomain.NewService(productRepo)
productHandler := product.NewHandler(productService)
```

**Files affected: 2**
- `database/product_mongo.go` (new file)
- `cmd/serve.go` (1 line changed)

**Files unaffected:**
- ✓ `domain/product/service.go`
- ✓ `domain/product/port.go`
- ✓ `handler/product/handler.go`
- ✓ All routes
- ✓ All tests

**This is the POWER of interfaces!** 💪

---

## When to Split Files

### The 100-200 Line Rule

**Instructor's advice:**

> **"If a file exceeds 100-200 lines, discuss with your team about splitting it. But generally, keeping related code in one file is better."**

### Example: Split service.go

**If `service.go` gets too large (200+ lines):**

**Before:**

```
domain/product/
├── port.go
└── service.go (300 lines - TOO BIG!)
```

**After:**

```
domain/product/
├── port.go
├── service.go       (Constructor + dependencies)
├── create.go        (Create business logic)
├── get.go          (Get business logic)
├── list.go         (List business logic)
├── update.go       (Update business logic)
└── delete.go       (Delete business logic)
```

### Split Example

**File: `domain/product/service.go`** (After split)

```go
package product

import "yourproject/domain"

// service implements Service interface
type service struct {
    productRepo Repository
}

// NewService creates new product service
func NewService(repo Repository) Service {
    return &service{
        productRepo: repo,
    }
}
```

**File: `domain/product/create.go`** (New file)

```go
package product

import "yourproject/domain"

// Create implements Service.Create
func (s *service) Create(product *domain.Product) (*domain.Product, error) {
    // Validation
    if product.Title == "" {
        return nil, errors.New("title is required")
    }
    
    if product.Price <= 0 {
        return nil, errors.New("price must be positive")
    }
    
    // Business logic
    product.Title = strings.TrimSpace(product.Title)
    product.Description = strings.TrimSpace(product.Description)
    
    // Create in repository
    createdProduct, err := s.productRepo.Create(product)
    if err != nil {
        return nil, err
    }
    
    // Log creation
    log.Printf("Product created: ID=%d, Title=%s", createdProduct.ID, createdProduct.Title)
    
    // Send event
    // publishProductCreatedEvent(createdProduct)
    
    return createdProduct, nil
}
```

**File: `domain/product/get.go`** (New file)

```go
package product

import "yourproject/domain"

// Get implements Service.Get
func (s *service) Get(id int64) (*domain.Product, error) {
    if id <= 0 {
        return nil, errors.New("invalid product ID")
    }
    
    product, err := s.productRepo.Get(id)
    if err != nil {
        return nil, err
    }
    
    if product == nil {
        return nil, nil
    }
    
    // Add to recently viewed
    // addToRecentlyViewed(id)
    
    return product, nil
}
```

**File: `domain/product/delete.go`** (New file)

```go
package product

// Delete implements Service.Delete
func (s *service) Delete(id int64) error {
    if id <= 0 {
        return errors.New("invalid product ID")
    }
    
    // Check if product exists
    product, err := s.productRepo.Get(id)
    if err != nil {
        return err
    }
    
    if product == nil {
        return errors.New("product not found")
    }
    
    // Check if product is used in orders
    // hasOrders, err := s.orderRepo.HasOrdersForProduct(id)
    // if hasOrders {
    //     return errors.New("cannot delete product with existing orders")
    // }
    
    // Delete
    err = s.productRepo.Delete(id)
    if err != nil {
        return err
    }
    
    // Log deletion
    log.Printf("Product deleted: ID=%d, Title=%s", product.ID, product.Title)
    
    return nil
}
```

**Benefits of splitting:**

```
Each file has ONE responsibility:
✓ service.go    → Dependencies and constructor
✓ create.go     → Create logic only
✓ get.go        → Get logic only
✓ list.go       → List logic only
✓ update.go     → Update logic only
✓ delete.go     → Delete logic only

Benefits:
✓ Easy to find code
✓ Easy to review changes
✓ Multiple developers can work simultaneously
✓ Clear file purpose
✓ No merge conflicts
```

---

## Industry Standards Achieved

### The Instructor's Challenge

**Instructor's passionate speech:**

> **"This structure I've created is the BEST standard used by the TOP 5 companies in Bangladesh! Now no one can tell you: 'You don't have an industry-standard project.'"**
>
> **"You now have the FATHER of industry standards! Put your hand on your chest and challenge anyone to bring their 'expert' to me. If they say this is not industry-standard, bring them to me!"**
>
> **"Why can you challenge them? Because you have LOGIC: One file changed = One file affected. It doesn't break 10,000 other files!"**

### What Makes This Industry Standard?

**1. Separation of concerns:**
```
✓ Domain entities in domain/
✓ Business logic in domain/*/service.go
✓ Data access in database/
✓ HTTP handling in handler/
✓ Configuration in config/
```

**2. Dependency inversion:**
```
✓ High-level modules don't depend on low-level modules
✓ Both depend on abstractions (interfaces)
✓ Handler → Service interface (not concrete service)
✓ Service → Repository interface (not concrete repository)
```

**3. Single responsibility:**
```
✓ Handler: HTTP request/response
✓ Service: Business logic
✓ Repository: Data access
✓ Each layer has ONE job
```

**4. Open/closed principle:**
```
✓ Open for extension (add new implementations)
✓ Closed for modification (existing code unchanged)
✓ Example: Add MongoDB without changing service/handler
```

**5. Interface segregation:**
```
✓ Small, focused interfaces
✓ Clients depend only on methods they use
✓ No fat interfaces with unused methods
```

**6. Testability:**
```
✓ Mock interfaces for unit testing
✓ No concrete dependencies
✓ Test each layer in isolation
```

### Compare with "Bokkol and Mukkul" Code

**Instructor's rant (and he's RIGHT!):**

> **"If someone writes code where changing one file breaks 10,000 files, that person is NOT an engineer. Give them two slaps and show them this video! Then they'll watch my course, I'll earn money, and I'll donate it to poor students. A beautiful cycle!" 😄**

**Bad code (98% of developers):**

```go
// Everything mixed together
func CreateOrder(c *gin.Context) {
    // User validation (should be in User domain)
    if user.Email == "" {
        // ...
    }
    
    // Product check (should be in Product domain)
    if product.Stock < 1 {
        // ...
    }
    
    // Payment processing (should be in Payment domain)
    if !processPayment() {
        // ...
    }
    
    // Inventory update (should be in Inventory domain)
    updateStock()
    
    // Notification (should be in Notification domain)
    sendEmail()
    
    // Order creation (finally!)
    createOrder()
}

// Change one thing = Everything breaks! 😱
```

**Good code (2% of engineers):**

```go
// Clean separation
func (h *OrderHandler) Create(c *gin.Context) {
    order := parseRequest(c)
    
    // Each domain handles its own logic
    createdOrder, err := h.orderService.Create(order)
    if err != nil {
        handleError(c, err)
        return
    }
    
    sendResponse(c, createdOrder)
}

// orderService coordinates domains
func (s *orderService) Create(order *domain.Order) (*domain.Order, error) {
    // User domain validates user
    user, err := s.userService.ValidateUser(order.UserID)
    
    // Product domain checks stock
    product, err := s.productService.CheckStock(order.ProductID)
    
    // Payment domain processes payment
    payment, err := s.paymentService.Process(order.Amount)
    
    // Inventory domain updates stock
    err = s.inventoryService.Deduct(order.ProductID, order.Quantity)
    
    // Notification domain sends notifications
    s.notificationService.NotifyUser(user.Email)
    
    // Order domain creates order
    return s.orderRepo.Create(order)
}

// Change one domain = One domain affected! 🎉
```

---

## Preparing for Unit Testing

### Why This Structure Makes Testing Easy

**Instructor's hint:**

> **"I haven't told you yet, but unit testing is a BEAUTIFUL thing! We'll cover it in the next 2-3 classes. Then you'll understand why we prefer this coding style!"**

### Testing Without DDD (Nightmare)

```go
// Handler directly uses repository
type Handler struct {
    ProductRepo *database.ProductRepository
}

// How to test this?
func TestCreate(t *testing.T) {
    // Need real database connection! 😱
    db := setupRealDatabase()
    
    repo := database.NewProductRepository(db)
    handler := Handler{ProductRepo: repo}
    
    // Test requires actual database
    // Slow, brittle, hard to maintain
}
```

### Testing With DDD (Beautiful)

```go
// Handler uses service interface
type Handler struct {
    productService Service
}

// Easy to test!
func TestCreate(t *testing.T) {
    // Create mock service
    mockService := &MockProductService{
        CreateFunc: func(product *domain.Product) (*domain.Product, error) {
            return &domain.Product{
                ID:    1,
                Title: product.Title,
            }, nil
        },
    }
    
    handler := NewHandler(mockService)
    
    // Test without database!
    // Fast, reliable, easy to maintain! 🎉
}

// Mock implementation
type MockProductService struct {
    CreateFunc func(*domain.Product) (*domain.Product, error)
}

func (m *MockProductService) Create(product *domain.Product) (*domain.Product, error) {
    return m.CreateFunc(product)
}
```

**Benefits:**

```
✓ No database required
✓ Test runs in milliseconds
✓ Test specific scenarios easily
✓ No test data cleanup
✓ Parallel test execution
✓ CI/CD friendly
```

---

## Engineer vs Developer

### The Harsh Truth

**Instructor's observation:**

> **"You'll see in the industry: 98% are developers, only 2% are genuine engineers."**

### Developer vs Engineer

**Developer (98%):**

```
Characteristics:
❌ Writes code that "works"
❌ One change breaks 10,000 files
❌ No unit tests
❌ Tight coupling everywhere
❌ Copy-paste programming
❌ No design patterns
❌ "It works on my machine!"
❌ Fear of changing old code

Result:
- Technical debt accumulates
- Bugs multiply
- Maintenance nightmare
- Company loses money
- No salary increase for anyone!
```

**Engineer (2%):**

```
Characteristics:
✓ Writes code that LASTS
✓ One change affects ONE file
✓ Comprehensive unit tests
✓ Loose coupling (interfaces)
✓ Understands design patterns
✓ Applies SOLID principles
✓ Code works everywhere!
✓ Confidently refactors code

Result:
- Technical debt prevented
- Bugs isolated and fixed quickly
- Easy maintenance
- Company makes profit
- Salary increases for everyone! 🎉
```

### How to Become an Engineer

**1. Learn principles:**
```
✓ SOLID principles
✓ Design patterns
✓ Domain-Driven Design
✓ Clean Architecture
```

**2. Write testable code:**
```
✓ Small, focused functions
✓ Dependency injection
✓ Interface-based programming
✓ Unit tests for everything
```

**3. Think about impact:**
```
✓ Before writing code: "What if this changes?"
✓ During writing: "How will I test this?"
✓ After writing: "Does this affect other files?"
```

**4. Challenge yourself:**
```
✓ Can I change the database without breaking handlers?
✓ Can I add a feature without modifying existing code?
✓ Can I test this without external dependencies?
```

**5. Learn from seniors:**
```
✓ If your senior writes bad code, teach them!
✓ Show them this project structure
✓ Explain: "One change = One file"
✓ Help them improve slowly
```

---

## Summary

### What We Accomplished

**1. Refactored Product domain:**
```
✓ Created domain/product/port.go
✓ Created domain/product/service.go
✓ Updated database/product.go to implement Repository interface
✓ Updated handler/product/handler.go to use Service interface
✓ Wired everything in cmd/serve.go
```

**2. Applied DDD pattern:**
```
✓ Parent (Handler) accesses child (Repository) through interfaces
✓ Dependencies defined in port.go
✓ Business logic in service.go
✓ Data access in repository
✓ HTTP logic in handler
```

**3. Achieved industry standards:**
```
✓ Separation of concerns
✓ Dependency inversion
✓ Single responsibility
✓ Open/closed principle
✓ Interface segregation
✓ High testability
```

### DDD Complete!

**Domains refactored:**
```
✓ User domain (Chapter 60)
✓ Product domain (Chapter 61)
```

**Structure achieved:**

```
yourproject/
├── domain/
│   ├── user.go              # User entity
│   ├── product.go           # Product entity
│   ├── user/
│   │   ├── port.go         # User interfaces
│   │   └── service.go      # User business logic
│   └── product/
│       ├── port.go         # Product interfaces
│       └── service.go      # Product business logic
├── database/
│   ├── user.go             # User repository (implements domain/user.Repository)
│   └── product.go          # Product repository (implements domain/product.Repository)
├── handler/
│   ├── user/
│   │   ├── port.go         # Handler interfaces
│   │   └── handler.go      # User HTTP handlers (uses domain/user.Service)
│   └── product/
│       ├── port.go         # Handler interfaces
│       └── handler.go      # Product HTTP handlers (uses domain/product.Service)
└── cmd/
    └── serve.go            # Dependency injection
```

### Key Principles Recap

**1. Parent never accesses child directly:**
```
Handler → Service interface (not service struct)
Service → Repository interface (not repository struct)
```

**2. Child can access parent:**
```
Repository → domain.Product entity ✓
Service → domain.Product entity ✓
```

**3. Dependencies through interfaces:**
```
All dependencies defined in port.go
Implementations injected via constructors
```

**4. One change = One file:**
```
Change repository → Service unchanged
Change service → Handler unchanged
Change handler → Router unchanged
```

---

## What's Next

**In Chapter 62, we'll dive into:**

**1. Unit Testing Fundamentals**
```
- What is unit testing?
- Why unit tests matter
- Testing pyramid
- Test-driven development (TDD)
```

**2. Testing with Mocks**
```
- Creating mock implementations
- Testing handlers with mock services
- Testing services with mock repositories
- Assertion libraries
```

**3. Table-Driven Tests**
```
- Go testing patterns
- Multiple test cases
- Edge cases and error scenarios
- Test coverage
```

**4. Integration Tests**
```
- Testing with real database
- Docker for test databases
- Test fixtures and cleanup
- CI/CD integration
```

**You'll finally understand:**

> **"Why we write code this way! Testing will make everything crystal clear!"**

**Get ready to become part of the 2% engineers!** 💪

---

**Key Takeaways:**

> **"This is the FATHER of industry standards! One file changed = One file affected. Anyone who writes code where one change breaks 10,000 files deserves two slaps and a lesson in DDD!"**

**Remember:**
- Product domain now follows DDD ✓
- Handler → Service → Repository (through interfaces) ✓
- Easy to test (coming next!) ✓
- Easy to extend ✓
- Easy to maintain ✓
- Industry standard achieved! ✓

**See you in Chapter 62 where we learn the BEAUTIFUL world of unit testing!** 🎉
