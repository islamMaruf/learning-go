# Chapter 50: Removing Tight Coupling - Clean Architecture

## Table of Contents
- [Introduction](#introduction)
- [Current Problems](#current-problems)
- [Feature-Based Organization](#feature-based-organization)
- [Creating Handler Structs](#creating-handler-structs)
- [Understanding Dependency Injection](#understanding-dependency-injection)
- [Implementing User Handlers](#implementing-user-handlers)
- [Implementing Product Handlers](#implementing-product-handlers)
- [Creating the Server Layer](#creating-the-server-layer)
- [Removing Config Tight Coupling](#removing-config-tight-coupling)
- [Creating Middleware Layer](#creating-middleware-layer)
- [Testing the Refactored Code](#testing-the-refactored-code)
- [Summary](#summary)
- [Practice Questions](#practice-questions)

---

## Introduction

In Chapter 49, we built authentication middleware. But our project structure has serious problems:
- All handlers in one folder (messy!)
- Routes scattered everywhere
- Config accessed from everywhere (tight coupling)
- No clear separation of concerns

Today, we'll **refactor** our entire project using:
1. **Feature-based organization** - group related code
2. **Dependency injection** - manage dependencies cleanly
3. **Loose coupling** - remove unnecessary dependencies

This is how **real production applications** are structured!

---

## Current Problems

### Problem 1: Mixed Handlers

Current structure:
```
handler/
├── create_user.go       ← User feature
├── login.go             ← User feature
├── create_product.go    ← Product feature
├── get_product.go       ← Product feature
├── update_product.go    ← Product feature
├── delete_product.go    ← Product feature
└── get_products.go      ← Product feature
```

**Problems:**
- All features mixed together
- Hard to find specific handlers
- Difficult to maintain as project grows

### Problem 2: Single Routes File

```go
// cmd/routes.go - Everything in one file!
mux.HandleFunc("POST /api/users", handler.CreateUser)
mux.HandleFunc("POST /api/users/login", handler.Login)
mux.HandleFunc("GET /api/products", handler.GetProducts)
mux.HandleFunc("POST /api/products", handler.CreateProduct)
// ... 100 more routes?
```

What if we have **1000 routes**? All in one file? **Not maintainable!**

### Problem 3: Tight Coupling with Config

```go
// middleware/authenticate_jwt.go
func AuthenticateJWT(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        cfg := config.GetConfig()  // ← Loads from .env EVERY request!
        // ...
    })
}
```

**Every** handler, middleware, and route accesses config directly. This is called **tight coupling**.

---

## Feature-Based Organization

### The Better Structure

```
handler/
├── user/
│   ├── handler.go       ← Handler struct
│   ├── routes.go        ← User routes
│   ├── create_user.go   ← Create user handler
│   └── login.go         ← Login handler
├── product/
│   ├── handler.go       ← Handler struct
│   ├── routes.go        ← Product routes
│   ├── create.go        ← Create product handler
│   ├── get.go           ← Get products handler
│   ├── update.go        ← Update product handler
│   └── delete.go        ← Delete product handler
└── review/
    ├── handler.go       ← Handler struct
    ├── routes.go        ← Review routes
    └── get_reviews.go   ← Get reviews handler
```

**Benefits:**
- Each feature has its own folder
- Easy to find code
- Can add new features without affecting others
- Clear separation of concerns

Think of it like organizing a library:
- **Fiction** → one section
- **Science** → another section
- **History** → another section

Not all books mixed together!

---

## Creating Handler Structs

### The Pattern

Each feature has a **handler struct** that groups related handlers together.

### User Handler Struct

**File: `handler/user/handler.go`**

```go
package user

import (
    "net/http"
)

// Handler groups all user-related handlers
type Handler struct {
    // Properties can be added here if needed
    // For now, keeping it empty
}

// NewHandler creates a new user handler
func NewHandler() *Handler {
    return &Handler{}
}

// CreateUser handles user creation
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    // Implementation from create_user.go
}

// Login handles user authentication
func (h *Handler) Login(w http.ResponseWriter, r *http.Request) {
    // Implementation from login.go
}
```

### Why a Struct?

**Without struct (old way):**
```go
func CreateUser(w, r) { ... }
func Login(w, r) { ... }
```

These are just floating functions. No relationship between them.

**With struct (new way):**
```go
type Handler struct {}

func (h *Handler) CreateUser(w, r) { ... }
func (h *Handler) Login(w, r) { ... }
```

Now they're **methods** of Handler. They're related!

```
Handler
  ├─ CreateUser()
  └─ Login()
```

---

## Understanding Dependency Injection

### The Problem

Imagine a car:
```
Car needs:
- Engine
- Wheels
- Steering

Car depends on these components
```

**Bad approach (tight coupling):**
```go
type Car struct {}

func (c *Car) Drive() {
    engine := CreateEngineDirectly()  // ← Car creates its own engine
    wheels := CreateWheelsDirectly()  // ← Car creates its own wheels
    // ...
}
```

**Problem:** Car is **tightly coupled** to specific engine and wheel implementations.

**Good approach (dependency injection):**
```go
type Car struct {
    Engine Engine  // ← Dependencies as properties
    Wheels Wheels
}

func NewCar(engine Engine, wheels Wheels) *Car {
    return &Car{
        Engine: engine,
        Wheels: wheels,
    }
}
```

**Benefit:** Car doesn't create dependencies. They're **injected** from outside!

### In Our Project

**Server depends on:**
- User handlers
- Product handlers
- Review handlers
- Config

**Instead of creating them inside Server, we inject them:**

```go
type Server struct {
    UserHandler    *user.Handler
    ProductHandler *product.Handler
    Config         *config.Config  // Dependencies
}

func NewServer(
    userHandler *user.Handler,
    productHandler *product.Handler,
    cfg *config.Config,
) *Server {
    return &Server{
        UserHandler:    userHandler,  // Injected
        ProductHandler: productHandler,
        Config:         cfg,
    }
}
```

---

## Implementing User Handlers

### Step 1: Create Handler Struct

**File: `handler/user/handler.go`**

```go
package user

type Handler struct {
    // Empty for now
}

func NewHandler() *Handler {
    return &Handler{}
}
```

### Step 2: Move Handlers to User Folder

Move these files to `handler/user/`:
- `create_user.go`
- `login.go`

**Update package name in both files:**
```go
package user  // Changed from: package handler
```

### Step 3: Convert to Receiver Functions

**File: `handler/user/create_user.go`**

```go
package user

import (
    "encoding/json"
    "net/http"
    "yourproject/database"
)

// CreateUser handles user registration
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var newUser database.User
    err := json.NewDecoder(r.Body).Decode(&newUser)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid request data"))
        return
    }
    
    createdUser := newUser.Store()
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(createdUser)
}
```

**File: `handler/user/login.go`**

```go
package user

import (
    "encoding/json"
    "net/http"
    "yourproject/config"
    "yourproject/database"
    "yourproject/utils"
)

// Login handles user authentication
func (h *Handler) Login(w http.ResponseWriter, r *http.Request) {
    var loginReq LoginRequest
    err := json.NewDecoder(r.Body).Decode(&loginReq)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid request data"))
        return
    }
    
    user := database.Find(loginReq.Email, loginReq.Password)
    if user == nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid credentials"))
        return
    }
    
    cfg := config.GetConfig()
    accessToken, err := utils.CreateJWT(
        cfg.JWTSecret,
        utils.Payload{
            Subject:   user.ID,
            FirstName: user.FirstName,
            LastName:  user.LastName,
            Email:     user.Email,
        },
    )
    
    if err != nil {
        http.Error(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "access_token": accessToken,
    })
}
```

### Step 4: Create User Routes

**File: `handler/user/routes.go`**

```go
package user

import (
    "net/http"
    "yourproject/middleware"
)

// RegisterRoutes registers all user routes
func (h *Handler) RegisterRoutes(
    mux *http.ServeMux,
    manager *middleware.Manager,
) {
    // Public routes
    mux.HandleFunc("POST /api/users", h.CreateUser)
    mux.HandleFunc("POST /api/users/login", h.Login)
}
```

**Key points:**
- `RegisterRoutes` is a **receiver function** of Handler
- Takes `mux` and `manager` as parameters
- Registers all user-related routes

---

## Implementing Product Handlers

### Step 1: Create Handler Struct

**File: `handler/product/handler.go`**

```go
package product

type Handler struct {
    // Empty for now
}

func NewHandler() *Handler {
    return &Handler{}
}
```

### Step 2: Move Product Handlers

Move these files to `handler/product/`:
- `create_product.go`
- `get_product.go`
- `get_products.go`
- `update_product.go`
- `delete_product.go`

**Update package in all files:**
```go
package product  // Changed from: package handler
```

### Step 3: Convert to Receiver Functions

**File: `handler/product/create_product.go`**

```go
package product

import (
    "encoding/json"
    "net/http"
    "yourproject/database"
)

// CreateProduct handles product creation
func (h *Handler) CreateProduct(w http.ResponseWriter, r *http.Request) {
    var newProduct database.Product
    err := json.NewDecoder(r.Body).Decode(&newProduct)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid request data"))
        return
    }
    
    createdProduct := newProduct.Store()
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(createdProduct)
}
```

Apply same pattern to other product handlers.

### Step 4: Create Product Routes

**File: `handler/product/routes.go`**

```go
package product

import (
    "net/http"
    "yourproject/middleware"
)

// RegisterRoutes registers all product routes
func (h *Handler) RegisterRoutes(
    mux *http.ServeMux,
    manager *middleware.Manager,
) {
    // Public routes
    mux.HandleFunc("GET /api/products", h.GetProducts)
    mux.HandleFunc("GET /api/products/{id}", h.GetProduct)
    
    // Protected routes
    mux.Handle("POST /api/products",
        manager.With(
            middleware.AuthenticateJWT,
            http.HandlerFunc(h.CreateProduct),
        ),
    )
    
    mux.Handle("PUT /api/products/{id}",
        manager.With(
            middleware.AuthenticateJWT,
            http.HandlerFunc(h.UpdateProduct),
        ),
    )
    
    mux.Handle("DELETE /api/products/{id}",
        manager.With(
            middleware.AuthenticateJWT,
            http.HandlerFunc(h.DeleteProduct),
        ),
    )
}
```

---

## Creating the Server Layer

### Server Struct with Dependencies

**File: `cmd/server/server.go`**

```go
package server

import (
    "fmt"
    "net/http"
    "yourproject/config"
    "yourproject/handler/product"
    "yourproject/handler/user"
    "yourproject/middleware"
)

// Server holds all dependencies
type Server struct {
    Config         *config.Config
    UserHandler    *user.Handler
    ProductHandler *product.Handler
}

// NewServer creates a new server with injected dependencies
func NewServer(
    cfg *config.Config,
    userHandler *user.Handler,
    productHandler *product.Handler,
) *Server {
    return &Server{
        Config:         cfg,
        UserHandler:    userHandler,
        ProductHandler: productHandler,
    }
}

// Start starts the HTTP server
func (s *Server) Start() {
    // Create middleware manager
    manager := middleware.NewManager()
    
    // Create mux
    mux := http.NewServeMux()
    
    // Register all routes
    s.UserHandler.RegisterRoutes(mux, manager)
    s.ProductHandler.RegisterRoutes(mux, manager)
    
    // Apply global middleware
    handler := middleware.Logger(mux)
    handler = middleware.CorsWithPreflight(handler)
    
    // Start server
    addr := fmt.Sprintf(":%d", s.Config.HTTPPort)
    fmt.Printf("Server running on port %d\n", s.Config.HTTPPort)
    http.ListenAndServe(addr, handler)
}
```

### Understanding the Flow

```
Server
  ├─ Config (dependency)
  ├─ UserHandler (dependency)
  └─ ProductHandler (dependency)

Server.Start()
  ├─ Create middleware manager
  ├─ Create mux
  ├─ UserHandler.RegisterRoutes(mux, manager)
  ├─ ProductHandler.RegisterRoutes(mux, manager)
  └─ Start HTTP server
```

### Update Main File

**File: `cmd/serve.go`**

```go
package cmd

import (
    "yourproject/cmd/server"
    "yourproject/config"
    "yourproject/handler/product"
    "yourproject/handler/user"
)

func Serve() {
    // Load config once
    config.LoadConfig()
    cfg := config.GetConfig()
    
    // Create handlers
    userHandler := user.NewHandler()
    productHandler := product.NewHandler()
    
    // Create server with dependencies injected
    srv := server.NewServer(
        cfg,
        userHandler,
        productHandler,
    )
    
    // Start server
    srv.Start()
}
```

### Dependency Injection in Action

```
main.go
  └─ cmd.Serve()
       ├─ config.LoadConfig()          ← Load once
       ├─ user.NewHandler()            ← Create user handler
       ├─ product.NewHandler()         ← Create product handler
       ├─ server.NewServer(...)        ← Inject dependencies
       └─ srv.Start()                  ← Start server
             ├─ userHandler.RegisterRoutes()
             └─ productHandler.RegisterRoutes()
```

**Benefits:**
1. **Config loaded once** - not on every request
2. **Clear dependencies** - easy to see what Server needs
3. **Easy to test** - can inject mock handlers
4. **Flexible** - can swap implementations

---

## Removing Config Tight Coupling

### The Problem

Currently, **everyone** accesses config:

```go
// In middleware
cfg := config.GetConfig()

// In handler
cfg := config.GetConfig()

// In routes
cfg := config.GetConfig()
```

This is like everyone in a relationship:
- Server dates Config
- Middleware dates Config
- Handler dates Config

**Problem:** Config has too many dependencies! This makes it hard to change.

### The Solution: Dependency Injection

**Only one entity should load config:**

```
cmd/serve.go
  └─ Loads config once
       └─ Passes to whoever needs it
```

### Create Middleware Layer

**File: `middleware/middleware.go`**

```go
package middleware

import (
    "yourproject/config"
)

// Middlewares holds middleware dependencies
type Middlewares struct {
    Config *config.Config
}

// NewMiddlewares creates middleware layer with dependencies
func NewMiddlewares(cfg *config.Config) *Middlewares {
    return &Middlewares{
        Config: cfg,
    }
}
```

### Update AuthenticateJWT

**File: `middleware/authenticate_jwt.go`**

```go
package middleware

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
    "net/http"
    "strings"
)

// AuthenticateJWT validates JWT tokens
func (m *Middlewares) AuthenticateJWT(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Get Authorization header
        header := r.Header.Get("Authorization")
        if header == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        // Split Bearer and token
        headerParts := strings.Split(header, " ")
        if len(headerParts) != 2 {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        accessToken := headerParts[1]
        
        // Split token into parts
        tokenParts := strings.Split(accessToken, ".")
        if len(tokenParts) != 3 {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        jwtHeader := tokenParts[0]
        jwtPayload := tokenParts[1]
        jwtSignature := tokenParts[2]
        
        // Create message
        message := jwtHeader + "." + jwtPayload
        
        // Use injected config (not global access!)
        secretKey := []byte(m.Config.JWTSecret)
        
        // Generate new signature
        h := hmac.New(sha256.New, secretKey)
        h.Write([]byte(message))
        hash := h.Sum(nil)
        
        // Base64 encode
        enc := base64.URLEncoding.WithPadding(base64.NoPadding)
        newSignature := enc.EncodeToString(hash)
        
        // Compare signatures
        if newSignature != jwtSignature {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        // Valid - call next handler
        next.ServeHTTP(w, r)
    })
}
```

**Key change:**
```go
// Old way (tight coupling):
cfg := config.GetConfig()
secretKey := cfg.JWTSecret

// New way (dependency injection):
secretKey := m.Config.JWTSecret  // Uses injected config
```

### Update Server to Use Middleware Layer

**File: `cmd/server/server.go`**

```go
func (s *Server) Start() {
    // Create middleware layer with config
    middlewares := middleware.NewMiddlewares(s.Config)
    
    // Create manager
    manager := middleware.NewManager()
    
    mux := http.NewServeMux()
    
    // Register routes (pass middlewares for auth)
    s.UserHandler.RegisterRoutes(mux, manager, middlewares)
    s.ProductHandler.RegisterRoutes(mux, manager, middlewares)
    
    // Apply global middleware
    handler := middleware.Logger(mux)
    handler = middleware.CorsWithPreflight(handler)
    
    addr := fmt.Sprintf(":%d", s.Config.HTTPPort)
    fmt.Printf("Server running on port %d\n", s.Config.HTTPPort)
    http.ListenAndServe(addr, handler)
}
```

### Update Routes to Use Middleware Layer

**File: `handler/product/routes.go`**

```go
func (h *Handler) RegisterRoutes(
    mux *http.ServeMux,
    manager *middleware.Manager,
    middlewares *middleware.Middlewares,  // Added
) {
    // Public routes
    mux.HandleFunc("GET /api/products", h.GetProducts)
    mux.HandleFunc("GET /api/products/{id}", h.GetProduct)
    
    // Protected routes (use middlewares.AuthenticateJWT)
    mux.Handle("POST /api/products",
        manager.With(
            middlewares.AuthenticateJWT,  // From injected middleware layer
            http.HandlerFunc(h.CreateProduct),
        ),
    )
    
    // ... other protected routes
}
```

---

## Testing the Refactored Code

### Test 1: Server Starts

```bash
go run main.go
```

Output:
```
Server running on port 4000
```

✓ Server starts successfully!

### Test 2: Get Products (Public)

```
GET http://localhost:4000/api/products
```

Response:
```json
[
  {"id": 1, "name": "Product 1"},
  {"id": 2, "name": "Product 2"}
]
```

✓ Public routes work!

### Test 3: Create Product (Protected)

**Without token:**
```
POST http://localhost:4000/api/products
Body: {"name": "New Product"}
```

Response:
```
401 Unauthorized
```

✓ Auth middleware blocks unauthenticated requests!

**With token:**
```
POST http://localhost:4000/api/products
Headers: Authorization: Bearer <valid_token>
Body: {"name": "New Product"}
```

Response:
```json
{
  "id": 3,
  "name": "New Product"
}
```

✓ Protected routes work with valid JWT!

---

## Summary

### What We Accomplished

**1. Feature-Based Organization**
```
handler/
├── user/          ← All user code together
├── product/       ← All product code together
└── review/        ← All review code together
```

**2. Dependency Injection**
```go
// Dependencies clearly defined
type Server struct {
    Config         *config.Config
    UserHandler    *user.Handler
    ProductHandler *product.Handler
}

// Dependencies injected at creation
srv := server.NewServer(cfg, userHandler, productHandler)
```

**3. Loose Coupling**
```go
// Old: Everyone accesses config directly (tight coupling)
cfg := config.GetConfig()

// New: Config injected where needed (loose coupling)
middlewares := middleware.NewMiddlewares(cfg)
```

### Benefits

**Maintainability:**
- Easy to find code (organized by feature)
- Easy to add new features
- Changes don't affect unrelated code

**Testability:**
- Can inject mock dependencies
- Can test features in isolation

**Performance:**
- Config loaded once (not on every request)
- No repeated file I/O

**Scalability:**
- Can add unlimited features
- Each feature independent

### The Relationship Analogy

**Tight Coupling (Bad):**
```
Server dates Config
Middleware dates Config
Handler dates Config
Routes date Config

Config has 10 relationships!
→ Insecure, unstable, hard to change
```

**Loose Coupling (Good):**
```
Only cmd/serve.go dates Config
Everyone else gets what they need through injection

Config has 1 relationship!
→ Secure, stable, easy to change
```

Just like in real life:
- **One committed relationship** = healthy, stable
- **Many relationships** = insecure, unstable

Same principle in code!

---

## Practice Questions

### Question 1: Adding a New Feature

**Question:**
Add a new "Review" feature with the following requirements:
- `GET /api/reviews` - Get all reviews
- `POST /api/reviews` - Create review (requires authentication)
- Each review has: ID, ProductID, UserID, Rating, Comment

Implement the complete feature following the patterns we learned.

<details>
<summary>Click to see answer</summary>

**Answer:**

**Step 1: Create Review Handler Struct**

**File: `handler/review/handler.go`**
```go
package review

type Handler struct {
    // Empty for now
}

func NewHandler() *Handler {
    return &Handler{}
}
```

**Step 2: Create Review Model**

**File: `database/review.go`**
```go
package database

type Review struct {
    ID        int    `json:"id"`
    ProductID int    `json:"product_id"`
    UserID    int    `json:"user_id"`
    Rating    int    `json:"rating"`
    Comment   string `json:"comment"`
}

var reviews []Review

func (r *Review) Store() Review {
    if r.ID != 0 {
        return *r
    }
    
    r.ID = len(reviews) + 1
    reviews = append(reviews, *r)
    return *r
}

func GetAllReviews() []Review {
    return reviews
}
```

**Step 3: Create Handlers**

**File: `handler/review/get_reviews.go`**
```go
package review

import (
    "encoding/json"
    "net/http"
    "yourproject/database"
)

func (h *Handler) GetReviews(w http.ResponseWriter, r *http.Request) {
    reviews := database.GetAllReviews()
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(reviews)
}
```

**File: `handler/review/create_review.go`**
```go
package review

import (
    "encoding/json"
    "net/http"
    "yourproject/database"
)

func (h *Handler) CreateReview(w http.ResponseWriter, r *http.Request) {
    var newReview database.Review
    err := json.NewDecoder(r.Body).Decode(&newReview)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid request data"))
        return
    }
    
    // Validate rating
    if newReview.Rating < 1 || newReview.Rating > 5 {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Rating must be between 1 and 5"))
        return
    }
    
    createdReview := newReview.Store()
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(createdReview)
}
```

**Step 4: Register Routes**

**File: `handler/review/routes.go`**
```go
package review

import (
    "net/http"
    "yourproject/middleware"
)

func (h *Handler) RegisterRoutes(
    mux *http.ServeMux,
    manager *middleware.Manager,
    middlewares *middleware.Middlewares,
) {
    // Public route
    mux.HandleFunc("GET /api/reviews", h.GetReviews)
    
    // Protected route
    mux.Handle("POST /api/reviews",
        manager.With(
            middlewares.AuthenticateJWT,
            http.HandlerFunc(h.CreateReview),
        ),
    )
}
```

**Step 5: Add to Server**

**File: `cmd/server/server.go`**
```go
import (
    "yourproject/handler/review"  // Add import
)

type Server struct {
    Config         *config.Config
    UserHandler    *user.Handler
    ProductHandler *product.Handler
    ReviewHandler  *review.Handler  // Add dependency
}

func NewServer(
    cfg *config.Config,
    userHandler *user.Handler,
    productHandler *product.Handler,
    reviewHandler *review.Handler,  // Inject
) *Server {
    return &Server{
        Config:         cfg,
        UserHandler:    userHandler,
        ProductHandler: productHandler,
        ReviewHandler:  reviewHandler,  // Store
    }
}

func (s *Server) Start() {
    middlewares := middleware.NewMiddlewares(s.Config)
    manager := middleware.NewManager()
    mux := http.NewServeMux()
    
    s.UserHandler.RegisterRoutes(mux, manager, middlewares)
    s.ProductHandler.RegisterRoutes(mux, manager, middlewares)
    s.ReviewHandler.RegisterRoutes(mux, manager, middlewares)  // Register
    
    // ... rest of Start()
}
```

**Step 6: Update cmd/serve.go**

**File: `cmd/serve.go`**
```go
import (
    "yourproject/handler/review"  // Add import
)

func Serve() {
    config.LoadConfig()
    cfg := config.GetConfig()
    
    userHandler := user.NewHandler()
    productHandler := product.NewHandler()
    reviewHandler := review.NewHandler()  // Create
    
    srv := server.NewServer(
        cfg,
        userHandler,
        productHandler,
        reviewHandler,  // Inject
    )
    
    srv.Start()
}
```

**Test:**
```
GET http://localhost:4000/api/reviews
→ 200 OK: []

POST http://localhost:4000/api/reviews
Headers: Authorization: Bearer <token>
Body: {
  "product_id": 1,
  "user_id": 1,
  "rating": 5,
  "comment": "Great product!"
}
→ 201 Created
```

✓ Complete feature added following clean architecture!

</details>

---

### Question 2: Understanding Dependency Injection

**Question:**
Explain the difference between these two approaches. Which is better and why?

**Approach A:**
```go
type Server struct {}

func (s *Server) Start() {
    cfg := config.GetConfig()
    userHandler := user.NewHandler()
    productHandler := product.NewHandler()
    // Use them...
}
```

**Approach B:**
```go
type Server struct {
    Config         *config.Config
    UserHandler    *user.Handler
    ProductHandler *product.Handler
}

func NewServer(cfg *config.Config, uh *user.Handler, ph *product.Handler) *Server {
    return &Server{
        Config: cfg,
        UserHandler: uh,
        ProductHandler: ph,
    }
}
```

<details>
<summary>Click to see answer</summary>

**Answer:**

**Approach A: No Dependency Injection (Bad)**

```go
type Server struct {}

func (s *Server) Start() {
    cfg := config.GetConfig()        // ← Creates inside
    userHandler := user.NewHandler()  // ← Creates inside
    // ...
}
```

**Problems:**

1. **Tight Coupling**
   - Server is tightly coupled to config package
   - Server is tightly coupled to user package
   - Can't easily change implementations

2. **Hard to Test**
   ```go
   // How to test Server.Start()?
   // Can't inject mock config or handlers!
   server := &Server{}
   server.Start()  // ← Uses real config, real handlers
   ```

3. **Hidden Dependencies**
   - Looking at Server struct, you can't see what it needs
   - Dependencies created internally (hidden)

4. **Less Flexible**
   - Can't use different config sources
   - Can't swap handler implementations
   - Everything hardcoded

**Approach B: Dependency Injection (Good)**

```go
type Server struct {
    Config         *config.Config      // ← Dependencies visible
    UserHandler    *user.Handler
    ProductHandler *product.Handler
}

func NewServer(
    cfg *config.Config,
    uh *user.Handler,
    ph *product.Handler,
) *Server {
    return &Server{
        Config: cfg,         // ← Injected from outside
        UserHandler: uh,
        ProductHandler: ph,
    }
}
```

**Benefits:**

1. **Loose Coupling**
   - Server doesn't create dependencies
   - Dependencies provided from outside
   - Easy to change implementations

2. **Easy to Test**
   ```go
   // Can inject mocks!
   mockConfig := &config.Config{HTTPPort: 8080}
   mockUserHandler := &MockUserHandler{}
   
   server := NewServer(mockConfig, mockUserHandler, ...)
   server.Start()  // ← Uses mocks, not real implementations
   ```

3. **Clear Dependencies**
   - Look at Server struct → see all dependencies
   - Look at NewServer → see what's required
   - No hidden surprises

4. **Flexible**
   ```go
   // Can use different configs
   testConfig := &config.Config{...}
   prodConfig := &config.Config{...}
   
   // Can use different handlers
   realHandler := user.NewHandler()
   mockHandler := &MockHandler{}
   ```

### Comparison Table

| Aspect | Approach A | Approach B |
|--------|-----------|-----------|
| **Coupling** | Tight | Loose |
| **Testability** | Hard | Easy |
| **Flexibility** | Low | High |
| **Dependencies** | Hidden | Visible |
| **Maintenance** | Difficult | Easy |

### Real-World Analogy

**Approach A (Bad):**
```
Restaurant that grows its own vegetables:
- Restaurant cooks food
- Restaurant also farms vegetables
- Restaurant also raises chickens

Problems:
- Too many responsibilities
- Can't use different suppliers
- Hard to test just the cooking
```

**Approach B (Good):**
```
Restaurant that buys from suppliers:
- Supplier A provides vegetables
- Supplier B provides chicken
- Restaurant just cooks

Benefits:
- Clear responsibilities
- Can change suppliers easily
- Can test cooking separately
```

### Conclusion

**Approach B (Dependency Injection) is better because:**
1. **Testable** - can inject mocks
2. **Flexible** - can swap implementations
3. **Clear** - dependencies are visible
4. **Maintainable** - loose coupling

This is a **fundamental principle** of clean architecture!

</details>

---

### Question 3: Config Loading Performance

**Question:**
In our old code, `config.GetConfig()` was called in every middleware execution. If we have 1000 requests per second, how many times does it load the `.env` file? Calculate the performance impact and explain why dependency injection solves this.

<details>
<summary>Click to see answer</summary>

**Answer:**

### Old Approach (Performance Problem)

**Code:**
```go
func AuthenticateJWT(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        cfg := config.GetConfig()  // ← Called on EVERY request!
        // ...
    })
}

func GetConfig() *Config {
    if cfg == nil {
        LoadConfig()  // ← Reads .env file!
    }
    return cfg
}
```

**Performance Calculation:**

**Scenario: 1000 requests/second**

```
Per request:
- AuthenticateJWT middleware called
- config.GetConfig() called
- If cfg is nil, LoadConfig() reads .env file

Worst case (no caching):
1000 requests/sec × 1 file read = 1000 file reads/sec

File I/O time: ~0.1ms per read
Total I/O time: 1000 × 0.1ms = 100ms/sec = 10% of CPU time wasted!
```

**Even with caching:**
```
Still calls GetConfig() 1000 times/sec
Still checks if cfg == nil 1000 times/sec
Unnecessary function calls add overhead
```

### New Approach (Dependency Injection)

**Code:**
```go
// Load config ONCE at startup
func Serve() {
    config.LoadConfig()  // ← Called ONCE
    cfg := config.GetConfig()
    
    middlewares := middleware.NewMiddlewares(cfg)  // ← Inject
    // ...
}

// Use injected config
func (m *Middlewares) AuthenticateJWT(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        secretKey := m.Config.JWTSecret  // ← Direct access, no function call
        // ...
    })
}
```

**Performance:**

```
Startup:
- config.LoadConfig() called ONCE
- .env file read ONCE
- Config stored in memory

Per request:
- m.Config.JWTSecret direct memory access
- No function calls
- No file I/O

1000 requests/sec × 0 file reads = 0 file reads/sec
```

### Performance Impact

**Old approach:**
```
Requests/sec: 1000
File reads: up to 1000
Function calls: 1000 (GetConfig)
Memory allocations: potentially 1000
CPU wasted: ~10%
```

**New approach:**
```
Requests/sec: 1000
File reads: 0 (already loaded)
Function calls: 0 (direct access)
Memory allocations: 0 (reuse same config)
CPU wasted: ~0%
```

### Benchmark Comparison

```go
// Old approach
func BenchmarkOldAuth(b *testing.B) {
    for i := 0; i < b.N; i++ {
        cfg := config.GetConfig()  // Function call every time
        _ = cfg.JWTSecret
    }
}
// Result: ~100ns per operation

// New approach
func BenchmarkNewAuth(b *testing.B) {
    cfg := config.GetConfig()  // Once before loop
    for i := 0; i < b.N; i++ {
        _ = cfg.JWTSecret  // Direct access
    }
}
// Result: ~5ns per operation (20x faster!)
```

### Why Dependency Injection Solves This

**1. Load Once**
```go
// At startup
config.LoadConfig()  // ← File I/O once
cfg := config.GetConfig()
```

**2. Inject Everywhere**
```go
// Pass to everyone who needs it
middlewares := middleware.NewMiddlewares(cfg)
server := server.NewServer(cfg, ...)
```

**3. Direct Access**
```go
// No function calls, just memory access
secretKey := m.Config.JWTSecret  // Fast!
```

### Memory Layout

**Old approach:**
```
Every request:
Request → GetConfig() → Check if nil → Return pointer
         ↑
         Function call overhead
```

**New approach:**
```
Every request:
Request → m.Config.JWTSecret
         ↑
         Direct memory access (single pointer dereference)
```

### Scaling Impact

**At 10,000 requests/second:**

| Metric | Old Approach | New Approach |
|--------|-------------|--------------|
| File reads/sec | up to 10,000 | 0 |
| Function calls/sec | 10,000 | 0 |
| CPU overhead | ~100ms/sec | ~0ms/sec |
| Latency added | +0.1ms/request | +0.005ms/request |

**At 100,000 requests/second:**
- Old: Completely breaks down (too much I/O)
- New: Still fast (just memory access)

### Conclusion

Dependency injection solves performance issues by:
1. **Loading once** - eliminate repeated I/O
2. **Injecting early** - pass references, not creating new
3. **Direct access** - no function call overhead
4. **Scalable** - works at any request volume

This is why **production systems** always use dependency injection!

</details>

---

**Next Chapter Preview:**

In Chapter 51, we'll add:
1. **Database Integration** - PostgreSQL instead of in-memory storage
2. **Database Migrations** - Schema management
3. **Connection Pooling** - Efficient database connections
4. **Repository Pattern** - Clean data access layer

Our architecture is ready for a real database! 🚀
