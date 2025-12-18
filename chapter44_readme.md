# Chapter 44: Advanced Routing & Middleware (Go 1.22+)

## Table of Contents
- [Introduction](#introduction)
- [The Problem With Old Routing](#the-problem-with-old-routing)
- [Go 1.22 Advanced Routing](#go-122-advanced-routing)
- [Old vs New Comparison](#old-vs-new-comparison)
- [Why Frameworks Existed](#why-frameworks-existed)
- [Implementing Advanced Routing](#implementing-advanced-routing)
- [The OPTIONS Problem](#the-options-problem)
- [What is Middleware](#what-is-middleware)
- [Creating CORS Middleware](#creating-cors-middleware)
- [How Middleware Works](#how-middleware-works)
- [Complete Implementation](#complete-implementation)
- [Practice Questions](#practice-questions)
- [Summary](#summary)
- [What's Next](#whats-next)

---

## Introduction

**Today's Topic:** Advanced Routing & Middleware in Go 1.22+

**What We've Done So Far:**
```go
mux.HandleFunc("/hello", helloHandler)
mux.HandleFunc("/products", getProducts)
mux.HandleFunc("/products/create", createProduct)
```

**The Problem:** Inside every handler, we check:
```go
if r.Method != http.MethodGet {
    http.Error(w, "Only GET allowed", 400)
    return
}
```

This is **painful**! Imagine doing this for 100 endpoints! 😱

**Today's Solution:**
- Advanced routing (method in URL pattern)
- Middleware (reusable functions)
- Clean, maintainable code

**Important Note About Frontend:**

Many students are asking: "I can't run the frontend!"

**Answer:** You **DON'T NEED** the frontend! 

- This is a **backend course**
- Frontend is just for visualization
- We'll use **Postman** for testing
- If Postman works → Everything works!

---

## The Problem With Old Routing

### Old Way (Before Go 1.22)

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // CORS handling
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    // OPTIONS handling
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // Method validation
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET allowed", 400)
        return
    }
    
    // Actual logic
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}
```

**Problems:**

1. **Repeated code** - CORS in every handler
2. **Method checking** - if r.Method != GET everywhere
3. **OPTIONS handling** - Repeated in every handler
4. **Messy** - Handler does too many things
5. **Hard to maintain** - Change CORS? Update 10 files!

### Visual Problem

```
Every Handler Has:
┌──────────────────────────────┐
│  1. Handle CORS              │
│  2. Handle OPTIONS           │
│  3. Check method             │
│  4. Actual business logic    │
└──────────────────────────────┘

Problem: 1, 2, 3 repeated everywhere! 😱
```

---

## Go 1.22 Advanced Routing

### Version Requirement

**CRITICAL:** You **MUST** have Go 1.22.0 or higher!

```bash
go version
# go version go1.22.2  ✅ Works
# go version go1.20.0  ❌ Won't work
```

**Check your version:**
```bash
go version
```

If below 1.22, upgrade Go!

### The New Way

**Old (HandleFunc):**
```go
mux.HandleFunc("/hello", helloHandler)

// Inside handler:
if r.Method != http.MethodGet {
    http.Error(w, "Only GET", 400)
    return
}
```

**New (Handle with method in pattern):**
```go
mux.Handle("GET /hello", http.HandlerFunc(helloHandler))

// No method checking needed inside handler! 🎉
```

### Key Differences

```
┌────────────────────────────────────────────────────┐
│              OLD vs NEW                            │
├────────────────────────────────────────────────────┤
│                                                    │
│  OLD (Before 1.22):                                │
│    mux.HandleFunc("/products", handler)            │
│    Pattern: Just the path                          │
│    Method: Check inside handler                    │
│                                                    │
│  NEW (Go 1.22+):                                   │
│    mux.Handle("GET /products", ...)                │
│    Pattern: METHOD + SPACE + PATH                  │
│    Method: Router handles it automatically         │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Pattern Format

**Format:** `"METHOD /path"`

```go
"GET /hello"           // GET requests only
"POST /create"         // POST requests only
"PUT /update"          // PUT requests only
"DELETE /delete"       // DELETE requests only
"PATCH /modify"        // PATCH requests only
"OPTIONS /preflight"   // OPTIONS requests only
```

**Important:** Method and path separated by **ONE SPACE**!

---

## Old vs New Comparison

### Complete Side-by-Side

**OLD (HandleFunc):**

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Product struct {
    ID    int    `json:"id"`
    Title string `json:"title"`
}

var productList []Product

func getProducts(w http.ResponseWriter, r *http.Request) {
    // CORS
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    // OPTIONS
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // Method check - REQUIRED!
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    
    // Business logic
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}

func main() {
    mux := http.NewServeMux()
    
    // Old way - HandleFunc
    mux.HandleFunc("/products", getProducts)
    
    http.ListenAndServe(":8080", mux)
}
```

**NEW (Handle with method):**

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Product struct {
    ID    int    `json:"id"`
    Title string `json:"title"`
}

var productList []Product

func getProducts(w http.ResponseWriter, r *http.Request) {
    // CORS (still needed, will fix with middleware later)
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    // OPTIONS (still needed, will fix with middleware later)
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // NO METHOD CHECK NEEDED! 🎉
    // Router already filtered by GET
    
    // Business logic
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}

func main() {
    mux := http.NewServeMux()
    
    // New way - Handle with method in pattern
    mux.Handle("GET /products", http.HandlerFunc(getProducts))
    
    http.ListenAndServe(":8080", mux)
}
```

### What Changed?

**1. Registration:**
```go
// Old
mux.HandleFunc("/products", getProducts)

// New
mux.Handle("GET /products", http.HandlerFunc(getProducts))
```

**2. Method Validation:**
```go
// Old - Manual check required
if r.Method != http.MethodGet {
    http.Error(w, "Only GET", 400)
    return
}

// New - Router handles it automatically
// No check needed! Router filters by GET
```

**3. What Happens with Wrong Method:**

```go
// Old routing:
POST /products → Handler called → Manual check → 400 error

// New routing:
POST /products → Router blocks → 405 Method Not Allowed
```

### Testing the Difference

**Request with correct method:**
```bash
curl -X GET http://localhost:8080/products
# ✅ Returns product list
```

**Request with wrong method:**
```bash
curl -X POST http://localhost:8080/products
# Old: 400 Bad Request (your custom message)
# New: 405 Method Not Allowed (automatic)
```

---

## Why Frameworks Existed

### The Pain Before Go 1.22

Before Go 1.22, routing was **painful**:

```go
// Every single handler needed:
if r.Method != http.MethodGet {
    http.Error(w, "Only GET", 400)
    return
}
```

**For 50 endpoints = 50 method checks!** 😱

### Popular Go Frameworks

Because of this pain, developers used frameworks:

**1. Gin**
```go
router := gin.Default()
router.GET("/products", getProducts)  // Clean!
```

**2. Chi**
```go
r := chi.NewRouter()
r.Get("/products", getProducts)  // Simple!
```

**3. Gorilla Mux**
```go
r := mux.NewRouter()
r.HandleFunc("/products", getProducts).Methods("GET")  // Clear!
```

### What are Frameworks?

**Framework** = Bundle of libraries working together

```
Framework = Library 1 + Library 2 + Library 3 + ...
```

**Examples:**
- Gin = Router + JSON helpers + Validation + More
- Chi = Router + Middleware + Context + More

### Do We Need Frameworks Now?

**Before Go 1.22:**
- ✅ Frameworks needed (routing was painful)
- Gin, Chi, Gorilla Mux very popular

**After Go 1.22:**
- ❌ Frameworks less necessary (routing is clean)
- Standard library is powerful enough!
- Can build without external dependencies

**Our Approach:**
- Use **standard library** (net/http)
- Learn **fundamentals**
- Understand **how things work**
- No magic, no black boxes!

---

## Implementing Advanced Routing

### Step 1: Update All Routes

**Old routes:**
```go
mux.HandleFunc("/hello", helloHandler)
mux.HandleFunc("/about", aboutHandler)
mux.HandleFunc("/products", getProducts)
mux.HandleFunc("/products/create", createProduct)
```

**New routes:**
```go
mux.Handle("GET /hello", http.HandlerFunc(helloHandler))
mux.Handle("GET /about", http.HandlerFunc(aboutHandler))
mux.Handle("GET /products", http.HandlerFunc(getProducts))
mux.Handle("POST /products/create", http.HandlerFunc(createProduct))
```

### Step 2: Remove Method Checks

**Before:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    handlePreflightRequest(w, r)
    
    // This check no longer needed! ❌
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    
    // Business logic
    sendData(w, productList, 200)
}
```

**After:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    handlePreflightRequest(w, r)
    
    // Method check removed! 🎉
    // Router already filtered by GET
    
    // Business logic
    sendData(w, productList, 200)
}
```

### Step 3: Update Main Function

**Complete main.go:**
```go
func main() {
    mux := http.NewServeMux()
    
    // GET routes
    mux.Handle("GET /hello", http.HandlerFunc(helloHandler))
    mux.Handle("GET /about", http.HandlerFunc(aboutHandler))
    mux.Handle("GET /products", http.HandlerFunc(getProducts))
    
    // POST routes
    mux.Handle("POST /products/create", http.HandlerFunc(createProduct))
    
    // Start server
    http.ListenAndServe(":8080", mux)
}
```

### Testing with Postman

**Test 1: Correct method**
```
GET http://localhost:8080/products
Response: 200 OK + product list ✅
```

**Test 2: Wrong method**
```
POST http://localhost:8080/products
Response: 405 Method Not Allowed ✅
```

**Test 3: POST to create**
```
POST http://localhost:8080/products/create
Body: {"title": "New Product"}
Response: 201 Created ✅
```

---

## The OPTIONS Problem

### The Issue

When we add method to pattern, OPTIONS breaks!

**Setup:**
```go
mux.Handle("GET /products", http.HandlerFunc(getProducts))
```

**What happens:**

```
1. Browser sends OPTIONS /products (preflight)
2. Router looks for "OPTIONS /products" route
3. Route not found!
4. 405 Method Not Allowed ❌
5. CORS fails! ❌
```

### The Painful Solution

Add OPTIONS for every route:

```go
// GET route
mux.Handle("GET /products", http.HandlerFunc(getProducts))

// OPTIONS route for preflight
mux.Handle("OPTIONS /products", http.HandlerFunc(getProducts))
```

**Inside handler:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    
    // Handle OPTIONS
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // Handle GET
    // ... business logic
}
```

### The Problem with This Approach

```
For every endpoint:
  - GET /products → OPTIONS /products
  - POST /create → OPTIONS /create
  - PUT /update → OPTIONS /update
  - DELETE /delete → OPTIONS /delete

For 20 endpoints = 40 routes! 😱
```

**This is insane!** We need a better solution...

### Enter: Middleware! 🎉

Instead of repeating OPTIONS everywhere, we use **middleware**!

---

## What is Middleware

### Definition

**Middleware** = Function that runs **BEFORE** your handler

```
Request → Middleware 1 → Middleware 2 → Handler → Response
```

### Visual Flow

```
┌─────────────────────────────────────────────────┐
│         Request Processing Flow                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  Client Request                                 │
│      ↓                                          │
│  Router matches route                           │
│      ↓                                          │
│  Middleware 1 (CORS)                            │
│      ↓                                          │
│  Middleware 2 (Authentication)                  │
│      ↓                                          │
│  Middleware 3 (Logging)                         │
│      ↓                                          │
│  Handler (Business Logic)                       │
│      ↓                                          │
│  Response                                       │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Real-World Analogy

**Restaurant:**

```
Customer → Greeter (Middleware 1)
        → Health check (Middleware 2)
        → Reservation check (Middleware 3)
        → Waiter serves food (Handler)
```

Each person does ONE job before passing to next!

### Why Middleware?

**Without Middleware:**
```go
func handler1(w, r) {
    // CORS
    // Auth
    // Logging
    // Business logic
}

func handler2(w, r) {
    // CORS (repeated!)
    // Auth (repeated!)
    // Logging (repeated!)
    // Business logic
}
```

**With Middleware:**
```go
func corsMiddleware(next) {
    // CORS only
    next()
}

func authMiddleware(next) {
    // Auth only
    next()
}

func handler(w, r) {
    // Only business logic!
}
```

### Middleware Benefits

```
✅ Code reuse (write once, use everywhere)
✅ Single Responsibility (each does one thing)
✅ Easy to test
✅ Easy to enable/disable
✅ Clean handlers (only business logic)
```

---

## Creating CORS Middleware

### The Goal

Move CORS handling from handlers to middleware:

**Before:**
```go
func getProducts(w, r) {
    // CORS here ❌
    w.Header().Set("Access-Control-Allow-Origin", "*")
    // ... business logic
}

func createProduct(w, r) {
    // CORS here too! ❌
    w.Header().Set("Access-Control-Allow-Origin", "*")
    // ... business logic
}
```

**After:**
```go
// CORS middleware (once!)
func corsMiddleware(next) {
    // CORS here ✅
}

func getProducts(w, r) {
    // Only business logic! ✅
}

func createProduct(w, r) {
    // Only business logic! ✅
}
```

### Step 1: Create Middleware Function

```go
func handleCorsMiddleware(next http.Handler) http.Handler {
    // This function returns a handler
    
    handleCors := func(w http.ResponseWriter, r *http.Request) {
        // Set CORS headers
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        w.Header().Set("Content-Type", "application/json")
        
        // Call next handler
        next.ServeHTTP(w, r)
    }
    
    // Return as http.Handler
    return http.HandlerFunc(handleCors)
}
```

### Understanding the Code

**Step-by-step breakdown:**

```go
// 1. Function signature
func handleCorsMiddleware(next http.Handler) http.Handler
//                         ↑                  ↑
//                   Takes handler      Returns handler

// 2. Inner function (does the work)
handleCors := func(w http.ResponseWriter, r *http.Request) {
    // Set CORS headers
    w.Header().Set("Access-Control-Allow-Origin", "*")
    // ...
    
    // Call next handler in chain
    next.ServeHTTP(w, r)
}

// 3. Wrap and return as handler
return http.HandlerFunc(handleCors)
```

### Key Concepts

**1. Function Expression:**
```go
handleCors := func(w, r) {
    // Function stored in variable
}
```

**2. Handler Interface:**
```go
type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}
```

**3. next.ServeHTTP(w, r):**
```go
// Calls the next handler in chain
next.ServeHTTP(w, r)
```

### Step 2: Use the Middleware

```go
func main() {
    mux := http.NewServeMux()
    
    // Wrap handler with middleware
    mux.Handle("GET /products", 
        handleCorsMiddleware(
            http.HandlerFunc(getProducts)
        ),
    )
}
```

### Visual Flow with Middleware

```
Request: GET /products
    ↓
Router matches "GET /products"
    ↓
handleCorsMiddleware called
    ↓ (sets CORS headers)
    ↓ calls next.ServeHTTP()
    ↓
getProducts handler called
    ↓ (business logic)
    ↓ returns response
    ↓
Response sent to client
```

---

## How Middleware Works

### The Chain Pattern

Middleware creates a **chain of handlers**:

```go
handler1 := middleware1(handler2)
handler2 := middleware2(handler3)
handler3 := actualHandler
```

**Execution order:**
```
Request → middleware1 → middleware2 → actualHandler → Response
```

### Detailed Example

```go
// Define handlers
func actualHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("3. Actual handler")
    w.Write([]byte("Response"))
}

// Middleware 1
func middleware1(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w, r) {
        fmt.Println("1. Middleware 1 - Before")
        next.ServeHTTP(w, r)
        fmt.Println("6. Middleware 1 - After")
    })
}

// Middleware 2
func middleware2(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w, r) {
        fmt.Println("2. Middleware 2 - Before")
        next.ServeHTTP(w, r)
        fmt.Println("5. Middleware 2 - After")
    })
}

// Usage
mux.Handle("GET /test",
    middleware1(
        middleware2(
            http.HandlerFunc(actualHandler)
        )
    )
)
```

**Output when request comes:**
```
1. Middleware 1 - Before
2. Middleware 2 - Before
3. Actual handler
4. Middleware 2 - After
5. Middleware 1 - After
```

### Why This Order?

```
Call Stack:

middleware1 {
    print "Before"
    middleware2 {
        print "Before"
        actualHandler {
            print "Handler"
        } ← Executes
        print "After"
    } ← Returns
    print "After"
} ← Returns
```

### Stopping the Chain

Middleware can **stop** the chain:

```go
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w, r) {
        // Check authentication
        token := r.Header.Get("Authorization")
        
        if token == "" {
            http.Error(w, "Unauthorized", 401)
            return  // Stop! Don't call next
        }
        
        // If authorized, continue
        next.ServeHTTP(w, r)
    })
}
```

**Flow with auth failure:**
```
Request → authMiddleware → 401 Error → Response
          (no next call)
```

---

## Complete Implementation

### Full Code with Middleware

```go
package main

import (
    "encoding/json"
    "net/http"
)

// Product struct
type Product struct {
    ID          int     `json:"id"`
    Title       string  `json:"title"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
}

var productList []Product

func init() {
    productList = []Product{
        {ID: 1, Title: "Orange", Description: "Fresh", Price: 100},
        {ID: 2, Title: "Apple", Description: "Red", Price: 40},
    }
}

// ============================================
// Middleware Functions
// ============================================

// CORS Middleware
func handleCorsMiddleware(next http.Handler) http.Handler {
    handleCors := func(w http.ResponseWriter, r *http.Request) {
        // Set CORS headers
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, Habib")
        w.Header().Set("Content-Type", "application/json")
        
        // Handle preflight
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusOK)
            return
        }
        
        // Call next handler
        next.ServeHTTP(w, r)
    }
    
    return http.HandlerFunc(handleCors)
}

// ============================================
// Handler Functions
// ============================================

// Get all products
func getProducts(w http.ResponseWriter, r *http.Request) {
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}

// Create product
func createProduct(w http.ResponseWriter, r *http.Request) {
    var newProduct Product
    decoder := json.NewDecoder(r.Body)
    decoder.Decode(&newProduct)
    
    newProduct.ID = len(productList) + 1
    productList = append(productList, newProduct)
    
    w.WriteHeader(http.StatusCreated)
    encoder := json.NewEncoder(w)
    encoder.Encode(newProduct)
}

// ============================================
// Main Function
// ============================================

func main() {
    mux := http.NewServeMux()
    
    // GET routes with middleware
    mux.Handle("GET /products",
        handleCorsMiddleware(
            http.HandlerFunc(getProducts),
        ),
    )
    
    // POST routes with middleware
    mux.Handle("POST /products/create",
        handleCorsMiddleware(
            http.HandlerFunc(createProduct),
        ),
    )
    
    // Start server
    http.ListenAndServe(":8080", mux)
}
```

### Benefits of This Approach

**1. Clean Handlers:**
```go
func getProducts(w, r) {
    // Only business logic!
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}
```

No CORS, no OPTIONS, no method checking!

**2. Reusable Middleware:**
```go
// Use same middleware for all routes
mux.Handle("GET /products", handleCorsMiddleware(...))
mux.Handle("POST /create", handleCorsMiddleware(...))
mux.Handle("PUT /update", handleCorsMiddleware(...))
```

**3. Easy to Modify:**
```go
// Change CORS? Update ONE place!
func handleCorsMiddleware(next http.Handler) http.Handler {
    // Change here affects all routes ✅
}
```

**4. Single Responsibility:**
```go
corsMiddleware  → Handles CORS only
authMiddleware  → Handles auth only
logMiddleware   → Handles logging only
handler         → Handles business logic only
```

### Testing

**Test 1: GET request**
```bash
curl -X GET http://localhost:8080/products

# Response:
# 200 OK
# [{"id":1,"title":"Orange",...}, ...]
```

**Test 2: POST request**
```bash
curl -X POST http://localhost:8080/products/create \
  -H "Content-Type: application/json" \
  -d '{"title":"Banana","description":"Yellow","price":5}'

# Response:
# 201 Created
# {"id":3,"title":"Banana",...}
```

**Test 3: OPTIONS (preflight)**
```bash
curl -X OPTIONS http://localhost:8080/products

# Response:
# 200 OK
# Headers include:
#   Access-Control-Allow-Origin: *
#   Access-Control-Allow-Methods: GET, POST, ...
```

**Test 4: Wrong method**
```bash
curl -X DELETE http://localhost:8080/products

# Response:
# 405 Method Not Allowed
```

---

## Practice Questions

### Question 1: Method in Pattern
**Q:** Explain the new routing pattern format in Go 1.22. What happens if you send a request with the wrong method?

<details>
<summary><b>Answer</b></summary>

**New Pattern Format (Go 1.22+):**

```go
mux.Handle("METHOD /path", handler)
```

**Format:** `"METHOD /path"` (method + space + path)

**Examples:**
```go
mux.Handle("GET /users", http.HandlerFunc(getUsers))
mux.Handle("POST /users", http.HandlerFunc(createUser))
mux.Handle("PUT /users/{id}", http.HandlerFunc(updateUser))
mux.Handle("DELETE /users/{id}", http.HandlerFunc(deleteUser))
mux.Handle("OPTIONS /users", http.HandlerFunc(optionsHandler))
```

**What Happens with Wrong Method:**

**Setup:**
```go
mux.Handle("GET /products", http.HandlerFunc(getProducts))
```

**Test 1: Correct method**
```
Request: GET /products
→ Router matches "GET /products"
→ Handler called ✅
→ Response: 200 OK
```

**Test 2: Wrong method**
```
Request: POST /products
→ Router looks for "POST /products"
→ Route not found
→ No handler called
→ Response: 405 Method Not Allowed ✅
```

**Comparison with Old Way:**

**Old (Before 1.22):**
```go
mux.HandleFunc("/products", getProducts)

func getProducts(w, r) {
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    // ...
}

// Request: POST /products
// → Handler IS called
// → Manual check fails
// → Returns 400 Bad Request
```

**New (Go 1.22+):**
```go
mux.Handle("GET /products", http.HandlerFunc(getProducts))

func getProducts(w, r) {
    // No method check needed!
    // ...
}

// Request: POST /products
// → Handler NOT called
// → Router blocks automatically
// → Returns 405 Method Not Allowed
```

**Benefits:**

1. **No manual checks** - Router handles it
2. **Correct HTTP status** - 405 instead of 400
3. **Cleaner handlers** - Only business logic
4. **Better performance** - Handler not called at all

**Important Notes:**

- Must have **Go 1.22.0+**
- Space required between method and path
- Method is **case-sensitive** (use `GET` not `get`)
- Can't have multiple methods in one pattern (no `"GET POST /path"`)

**For multiple methods on same path:**
```go
// Register separately
mux.Handle("GET /products", http.HandlerFunc(getProducts))
mux.Handle("POST /products", http.HandlerFunc(createProduct))
mux.Handle("OPTIONS /products", http.HandlerFunc(handleOptions))
```
</details>

---

### Question 2: Middleware Concept
**Q:** What is middleware? Explain how middleware works with a visual diagram and code example.

<details>
<summary><b>Answer</b></summary>

**Middleware Definition:**

**Middleware** = A function that wraps a handler to add functionality before/after the handler executes.

**Visual Flow:**

```
┌──────────────────────────────────────────────┐
│         Request Processing Flow              │
├──────────────────────────────────────────────┤
│                                              │
│  Client sends request                        │
│       ↓                                      │
│  ┌─────────────────────┐                    │
│  │  Middleware 1       │                    │
│  │  (CORS)             │                    │
│  │  - Before: Set headers                   │
│  │  - Call next         │                   │
│  │  - After: (optional) │                   │
│  └─────────────────────┘                    │
│       ↓                                      │
│  ┌─────────────────────┐                    │
│  │  Middleware 2       │                    │
│  │  (Auth)             │                    │
│  │  - Before: Check token                   │
│  │  - Call next         │                   │
│  │  - After: Log access │                   │
│  └─────────────────────┘                    │
│       ↓                                      │
│  ┌─────────────────────┐                    │
│  │  Handler            │                    │
│  │  (Business Logic)   │                    │
│  │  - Get data          │                   │
│  │  - Send response     │                   │
│  └─────────────────────┘                    │
│       ↓                                      │
│  Response to client                          │
│                                              │
└──────────────────────────────────────────────┘
```

**Code Example:**

```go
// Simple logging middleware
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // BEFORE handler
        fmt.Printf("[%s] %s\n", r.Method, r.URL.Path)
        
        // Call next handler
        next.ServeHTTP(w, r)
        
        // AFTER handler
        fmt.Println("Request completed")
    })
}

// Auth middleware
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // BEFORE handler
        token := r.Header.Get("Authorization")
        
        if token == "" {
            http.Error(w, "Unauthorized", 401)
            return  // Stop here! Don't call next
        }
        
        // If authorized, continue
        next.ServeHTTP(w, r)
        
        // AFTER handler
        fmt.Println("Authorized access granted")
    })
}

// Business logic handler
func getProducts(w http.ResponseWriter, r *http.Request) {
    products := []string{"Apple", "Orange", "Banana"}
    json.NewEncoder(w).Encode(products)
}

// Usage: Chain middlewares
func main() {
    mux := http.NewServeMux()
    
    mux.Handle("GET /products",
        loggingMiddleware(
            authMiddleware(
                http.HandlerFunc(getProducts),
            ),
        ),
    )
    
    http.ListenAndServe(":8080", mux)
}
```

**Execution Flow:**

```
Request: GET /products
Authorization: Bearer token123

1. loggingMiddleware (before)
   → Prints: "[GET] /products"
   
2. authMiddleware (before)
   → Checks token: ✅ Valid
   
3. getProducts handler
   → Returns product list
   
4. authMiddleware (after)
   → Prints: "Authorized access granted"
   
5. loggingMiddleware (after)
   → Prints: "Request completed"

Response: 200 OK + product list
```

**Stopping the Chain:**

```go
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w, r) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "Unauthorized", 401)
            return  // ← STOP! Don't call next
        }
        
        next.ServeHTTP(w, r)  // Continue only if authorized
    })
}
```

**Flow when auth fails:**
```
Request: GET /products (no token)

1. loggingMiddleware (before)
   → Prints: "[GET] /products"
   
2. authMiddleware (before)
   → Checks token: ❌ Missing
   → Returns 401 error
   → Does NOT call next.ServeHTTP()
   
(getProducts never called!)

3. loggingMiddleware (after)
   → Prints: "Request completed"

Response: 401 Unauthorized
```

**Benefits of Middleware:**

1. **Separation of concerns** - Each middleware does ONE thing
2. **Reusability** - Use same middleware for multiple routes
3. **Composability** - Combine middleware in any order
4. **Clean handlers** - Handler only has business logic
5. **Easy testing** - Test each middleware independently

**Real-World Examples:**

```go
// CORS middleware
corsMiddleware      → Sets CORS headers

// Auth middleware
authMiddleware      → Validates JWT tokens

// Logging middleware
loggingMiddleware   → Logs requests/responses

// Rate limiting
rateLimitMiddleware → Prevents abuse

// Compression
gzipMiddleware      → Compresses responses
```

**Complete Example with Multiple Middleware:**

```go
mux.Handle("GET /api/users",
    loggingMiddleware(
        corsMiddleware(
            authMiddleware(
                rateLimitMiddleware(
                    http.HandlerFunc(getUsers),
                ),
            ),
        ),
    ),
)

// Execution order:
// logging → cors → auth → rateLimit → handler
```
</details>

---

### Question 3: Create CORS Middleware
**Q:** Write a complete CORS middleware that handles preflight requests and sets all necessary headers.

<details>
<summary><b>Answer</b></summary>

**Complete CORS Middleware:**

```go
package main

import (
    "net/http"
)

// corsMiddleware handles CORS and preflight requests
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // ========================================
        // Set CORS Headers
        // ========================================
        
        // Allow all origins (for development)
        // In production, specify exact origin:
        // w.Header().Set("Access-Control-Allow-Origin", "https://myapp.com")
        w.Header().Set("Access-Control-Allow-Origin", "*")
        
        // Allow all HTTP methods
        w.Header().Set("Access-Control-Allow-Methods", 
            "GET, POST, PUT, PATCH, DELETE, OPTIONS")
        
        // Allow these headers in requests
        w.Header().Set("Access-Control-Allow-Headers", 
            "Content-Type, Authorization, X-Requested-With, X-User-ID")
        
        // Allow credentials (cookies, auth headers)
        // Note: Can't use with Allow-Origin: * in production
        w.Header().Set("Access-Control-Allow-Credentials", "true")
        
        // How long preflight can be cached (seconds)
        w.Header().Set("Access-Control-Max-Age", "3600")
        
        // Set response content type
        w.Header().Set("Content-Type", "application/json")
        
        // ========================================
        // Handle OPTIONS Preflight
        // ========================================
        
        if r.Method == http.MethodOptions {
            // Return 204 No Content for OPTIONS
            w.WriteHeader(http.StatusNoContent)
            return  // Stop here, don't call next handler
        }
        
        // ========================================
        // Continue to Next Handler
        // ========================================
        
        next.ServeHTTP(w, r)
    })
}

// Example usage
func main() {
    mux := http.NewServeMux()
    
    // Wrap all routes with CORS middleware
    mux.Handle("GET /users", 
        corsMiddleware(http.HandlerFunc(getUsers)))
    
    mux.Handle("POST /users", 
        corsMiddleware(http.HandlerFunc(createUser)))
    
    http.ListenAndServe(":8080", mux)
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte(`{"users": ["Alice", "Bob"]}`))
}

func createUser(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusCreated)
    w.Write([]byte(`{"id": 1, "name": "Charlie"}`))
}
```

**Production-Ready Version:**

```go
// corsConfig holds CORS configuration
type corsConfig struct {
    AllowedOrigins []string
    AllowedMethods []string
    AllowedHeaders []string
    AllowCredentials bool
    MaxAge int
}

// corsMiddleware with configuration
func corsMiddleware(config corsConfig) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")
            
            // Check if origin is allowed
            allowed := false
            for _, allowedOrigin := range config.AllowedOrigins {
                if allowedOrigin == "*" || allowedOrigin == origin {
                    allowed = true
                    w.Header().Set("Access-Control-Allow-Origin", allowedOrigin)
                    break
                }
            }
            
            if !allowed {
                http.Error(w, "Origin not allowed", http.StatusForbidden)
                return
            }
            
            // Set other CORS headers
            w.Header().Set("Access-Control-Allow-Methods", 
                strings.Join(config.AllowedMethods, ", "))
            
            w.Header().Set("Access-Control-Allow-Headers", 
                strings.Join(config.AllowedHeaders, ", "))
            
            if config.AllowCredentials {
                w.Header().Set("Access-Control-Allow-Credentials", "true")
            }
            
            if config.MaxAge > 0 {
                w.Header().Set("Access-Control-Max-Age", 
                    strconv.Itoa(config.MaxAge))
            }
            
            // Handle preflight
            if r.Method == http.MethodOptions {
                w.WriteHeader(http.StatusNoContent)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// Usage with configuration
func main() {
    config := corsConfig{
        AllowedOrigins: []string{
            "https://myapp.com",
            "https://staging.myapp.com",
        },
        AllowedMethods: []string{"GET", "POST", "PUT", "DELETE"},
        AllowedHeaders: []string{"Content-Type", "Authorization"},
        AllowCredentials: true,
        MaxAge: 3600,
    }
    
    cors := corsMiddleware(config)
    
    mux := http.NewServeMux()
    mux.Handle("GET /api/users", cors(http.HandlerFunc(getUsers)))
    
    http.ListenAndServe(":8080", mux)
}
```

**Testing:**

```bash
# Test preflight request
curl -X OPTIONS http://localhost:8080/users \
  -H "Origin: https://myapp.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type" \
  -v

# Response should include:
# HTTP/1.1 204 No Content
# Access-Control-Allow-Origin: *
# Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
# Access-Control-Allow-Headers: Content-Type, Authorization, ...
```

**Key Points:**

1. **Always handle OPTIONS** - Required for preflight
2. **Return 204 for OPTIONS** - Standard practice
3. **Don't call next on OPTIONS** - Preflight doesn't need handler
4. **Set all headers before checking method** - Browser needs them
5. **Use Max-Age** - Reduces preflight requests
</details>

---

### Question 4: Go Version Requirement
**Q:** Why is Go 1.22+ required for this chapter? What happens if you try to use the new routing syntax with an older version?

<details>
<summary><b>Answer</b></summary>

**Why Go 1.22+ Required:**

Go 1.22.0 introduced **enhanced routing patterns** that allow HTTP methods in the route pattern.

**What's New in Go 1.22:**

**1. Method in Pattern:**
```go
// Go 1.22+
mux.Handle("GET /users", handler)  ✅

// Before Go 1.22
mux.Handle("GET /users", handler)  ❌ Syntax error!
```

**2. Path Parameters (also new in 1.22):**
```go
// Go 1.22+
mux.Handle("GET /users/{id}", handler)  ✅

// Before Go 1.22
mux.Handle("GET /users/{id}", handler)  ❌ Doesn't work!
```

**What Happens with Older Go Version:**

**Test with Go 1.20:**

```go
// Code
package main

import "net/http"

func main() {
    mux := http.NewServeMux()
    mux.Handle("GET /hello", http.HandlerFunc(hello))
    http.ListenAndServe(":8080", mux)
}

func hello(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello"))
}
```

**Compile with Go 1.20:**
```bash
$ go version
go version go1.20.0 linux/amd64

$ go run main.go
# ❌ ERROR: Pattern doesn't work as expected!
# Router treats "GET /hello" as literal path
# Must access: http://localhost:8080/GET%20/hello
```

**The Problem:**

Before Go 1.22, the router treated the entire pattern as a **literal path**, including spaces and method names!

```
Go 1.20 interprets:
  Pattern: "GET /hello"
  As path: "/GET /hello" (literal string!)

Go 1.22+ interprets:
  Pattern: "GET /hello"
  As: Method=GET, Path=/hello (parsed!)
```

**Checking Your Version:**

```bash
go version

# Must see:
# go version go1.22.0 or higher ✅

# If you see:
# go version go1.20.x ❌
# go version go1.21.x ❌
```

**Upgrading Go:**

```bash
# Download from https://go.dev/dl/

# Or use version manager:
# gvm install go1.22.0
# gvm use go1.22.0

# Verify:
go version
```

**Minimum Versions:**

```
✅ go1.22.0 - Works
✅ go1.22.1 - Works
✅ go1.22.2 - Works
✅ go1.23.0 - Works (if exists)

❌ go1.21.x - Doesn't work
❌ go1.20.x - Doesn't work
❌ go1.19.x - Doesn't work
```

**Backward Compatibility:**

If you need to support Go < 1.22, use old syntax:

```go
// Works on ALL Go versions
mux.HandleFunc("/hello", func(w, r) {
    if r.Method != http.MethodGet {
        http.Error(w, "Method not allowed", 405)
        return
    }
    w.Write([]byte("Hello"))
})
```

**Why This Matters:**

1. **Team Consistency** - Everyone needs Go 1.22+
2. **Deployment** - Production servers need upgrade
3. **CI/CD** - Build pipelines need version update
4. **Libraries** - If building library, specify in go.mod

**In go.mod:**
```go
module myapp

go 1.22  // Minimum version required
```

**Summary:**

- **Required:** Go 1.22.0 or higher
- **Reason:** Enhanced routing patterns (method in pattern)
- **Check:** `go version`
- **Upgrade:** Download from go.dev
- **Alternative:** Use old HandleFunc syntax for older versions
</details>

---

### Question 5: Complete Refactoring
**Q:** Refactor this old-style code to use Go 1.22 routing and middleware:

```go
// Old code
mux.HandleFunc("/products", func(w, r) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    if r.Method == http.MethodOptions {
        w.WriteHeader(200)
        return
    }
    
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    
    json.NewEncoder(w).Encode(products)
})

mux.HandleFunc("/products/create", func(w, r) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    if r.Method == http.MethodOptions {
        w.WriteHeader(200)
        return
    }
    
    if r.Method != http.MethodPost {
        http.Error(w, "Only POST", 400)
        return
    }
    
    var product Product
    json.NewDecoder(r.Body).Decode(&product)
    products = append(products, product)
    
    w.WriteHeader(201)
    json.NewEncoder(w).Encode(product)
})
```

<details>
<summary><b>Answer</b></summary>

**Refactored Code (Go 1.22+ with Middleware):**

```go
package main

import (
    "encoding/json"
    "net/http"
)

// Product struct
type Product struct {
    ID    int    `json:"id"`
    Title string `json:"title"`
    Price float64 `json:"price"`
}

// Global product list
var products []Product

func init() {
    products = []Product{
        {ID: 1, Title: "Apple", Price: 1.50},
        {ID: 2, Title: "Orange", Price: 2.00},
    }
}

// ============================================
// Middleware
// ============================================

// CORS middleware - handles CORS and preflight
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Set CORS headers
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        w.Header().Set("Content-Type", "application/json")
        
        // Handle OPTIONS preflight
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusOK)
            return  // Don't call next handler for preflight
        }
        
        // Continue to next handler
        next.ServeHTTP(w, r)
    })
}

// ============================================
// Handlers (Clean - Only Business Logic!)
// ============================================

// Get all products
func getProducts(w http.ResponseWriter, r *http.Request) {
    // No CORS handling needed! ✅
    // No OPTIONS handling needed! ✅
    // No method checking needed! ✅
    
    // Only business logic!
    json.NewEncoder(w).Encode(products)
}

// Create product
func createProduct(w http.ResponseWriter, r *http.Request) {
    // No CORS handling needed! ✅
    // No OPTIONS handling needed! ✅
    // No method checking needed! ✅
    
    // Only business logic!
    var product Product
    err := json.NewDecoder(r.Body).Decode(&product)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    product.ID = len(products) + 1
    products = append(products, product)
    
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(product)
}

// ============================================
// Main - Setup Routes
// ============================================

func main() {
    mux := http.NewServeMux()
    
    // New routing style with method in pattern
    // Wrapped with CORS middleware
    
    mux.Handle("GET /products",
        corsMiddleware(http.HandlerFunc(getProducts)),
    )
    
    mux.Handle("POST /products/create",
        corsMiddleware(http.HandlerFunc(createProduct)),
    )
    
    // Start server
    println("Server running on :8080")
    http.ListenAndServe(":8080", mux)
}
```

**Key Improvements:**

**1. Clean Handlers:**
```go
// Before: 15+ lines with CORS, OPTIONS, method check
// After: 3 lines with only business logic! ✅
```

**2. Method in Pattern:**
```go
// Before: mux.HandleFunc("/products", handler)
//         + manual method check inside

// After: mux.Handle("GET /products", handler)
//        No method check needed! ✅
```

**3. Reusable Middleware:**
```go
// CORS code written ONCE
// Used for ALL routes ✅

corsMiddleware(handler1)
corsMiddleware(handler2)
corsMiddleware(handler3)
```

**4. No Repeated Code:**
```go
// Before:
//   CORS in getProducts ❌
//   CORS in createProduct ❌
//   CORS in updateProduct ❌

// After:
//   CORS in middleware ONCE ✅
//   Applied to all routes automatically
```

**Advanced: Multiple Middleware:**

```go
// Add logging middleware
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w, r) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

// Chain multiple middleware
mux.Handle("GET /products",
    loggingMiddleware(
        corsMiddleware(
            http.HandlerFunc(getProducts),
        ),
    ),
)

// Execution: logging → cors → handler
```

**Even Better: Middleware Helper:**

```go
// Helper to apply multiple middleware
func chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

// Usage
mux.Handle("GET /products",
    chain(
        http.HandlerFunc(getProducts),
        loggingMiddleware,
        corsMiddleware,
        authMiddleware,
    ),
)
```

**Summary of Changes:**

```
┌────────────────────────────────────────────────┐
│         Before → After                         │
├────────────────────────────────────────────────┤
│                                                │
│  HandleFunc → Handle                           │
│  "/path" → "METHOD /path"                      │
│  Manual method check → Automatic               │
│  CORS everywhere → CORS in middleware          │
│  OPTIONS everywhere → OPTIONS in middleware    │
│  Messy handlers → Clean handlers               │
│  Repeated code → Reusable middleware           │
│                                                │
└────────────────────────────────────────────────┘
```
</details>

---

## Summary

**What We Learned Today:**

1. ✅ **Go 1.22+ Advanced Routing**
   - Method in pattern: `"GET /path"`
   - Automatic method filtering
   - No manual method checks!

2. ✅ **Old vs New Comparison**
   - HandleFunc → Handle
   - Manual checks → Automatic
   - Cleaner code

3. ✅ **Why Frameworks Existed**
   - Gin, Chi, Gorilla Mux
   - Solved routing pain
   - Now standard library is enough!

4. ✅ **Middleware Concept**
   - Functions that wrap handlers
   - Execute before/after handler
   - Reusable across routes

5. ✅ **CORS Middleware**
   - Handle CORS once
   - Handle OPTIONS once
   - Apply to all routes

6. ✅ **Clean Architecture**
   - Handlers only have business logic
   - Middleware handles cross-cutting concerns
   - Single Responsibility Principle

**Key Takeaways:**

```
┌──────────────────────────────────────────┐
│         Modern Go API Structure          │
├──────────────────────────────────────────┤
│                                          │
│  Routes (Go 1.22+)                       │
│    ↓ "GET /path"                         │
│  Middleware                              │
│    ↓ CORS, Auth, Logging                 │
│  Handlers                                │
│    ↓ Business Logic Only                 │
│  Response                                │
│                                          │
└──────────────────────────────────────────┘
```

**Professional Code:**
- No repeated CORS
- No repeated OPTIONS
- No manual method checks
- Clean, maintainable, testable!

---

## What's Next?

**Chapter 45: Project Structure & Packages**

Next class:
- **Package organization** (handlers/, middleware/, models/)
- **File structure** for large projects
- **Separating concerns** properly
- **Import/Export** between packages
- **Clean architecture** patterns

**Why This Matters:**

Right now everything is in `main.go` - fine for learning, but **terrible for real projects**!

```
Current (Single File):
main.go (500+ lines) 😱

Next (Organized):
├── main.go (50 lines - entry point only)
├── handlers/
│   ├── product_handler.go
│   └── user_handler.go
├── middleware/
│   ├── cors.go
│   └── auth.go
├── models/
│   └── product.go
└── utils/
    └── response.go
```

**Professional structure for:**
- Large teams
- Scalable codebases
- Easy maintenance
- Clear organization

---

**Congratulations!** You now understand:
- Modern Go routing (1.22+)
- Middleware patterns
- Clean code architecture
- Why frameworks aren't always needed

**You're writing professional-level Go code!** 🎉

---

**Remember:**
> "With Go 1.22+, the standard library is powerful enough for most APIs. Learn the fundamentals before reaching for frameworks!"

---
