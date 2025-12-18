# Chapter 43: Refactor The Codebase (Clean Code!)

## Table of Contents
- [Introduction](#introduction)
- [Why Refactoring Matters](#why-refactoring-matters)
- [Current Code Problems](#current-code-problems)
- [Single Responsibility Principle](#single-responsibility-principle)
- [Refactoring Step 1: Extract CORS Handler](#refactoring-step-1-extract-cors-handler)
- [Refactoring Step 2: Extract Preflight Handler](#refactoring-step-2-extract-preflight-handler)
- [Refactoring Step 3: Extract Response Sender](#refactoring-step-3-extract-response-sender)
- [Before vs After Comparison](#before-vs-after-comparison)
- [Testing After Refactoring](#testing-after-refactoring)
- [SOLID Principles Introduction](#solid-principles-introduction)
- [Practice Questions](#practice-questions)
- [Summary](#summary)
- [What's Next](#whats-next)

---

## Introduction

**Previous Classes:**
- Chapter 40: GET Request
- Chapter 41: POST Request  
- Chapter 42: OPTIONS Method & Preflight

**Today's Mission:** Clean up our messy code! 🧹

Right now our code works, but it's **TERRIBLE**:
- CORS handling repeated everywhere
- Same code copy-pasted multiple times
- Functions doing too many things
- Hard to maintain and understand

**Today we'll learn:**
- How to identify code problems
- Single Responsibility Principle (SRP)
- Extracting reusable functions
- Making code clean and maintainable

---

## Why Refactoring Matters

### What is Refactoring?

**Refactoring** = Improving code structure without changing behavior

```
┌────────────────────────────────────────┐
│         Refactoring Definition         │
├────────────────────────────────────────┤
│                                        │
│  Before: Messy code that works ✅      │
│  After:  Clean code that works ✅      │
│                                        │
│  Behavior: SAME                        │
│  Structure: BETTER                     │
│                                        │
└────────────────────────────────────────┘
```

### Why Clean Code Matters

**Scenario: Real Company Project**

```
Bad Code:
  - You write messy code
  - 6 months later: Need to add feature
  - You don't understand your own code! 😱
  - Takes 2 days to understand
  - Takes 1 hour to add feature
  - Total: 2+ days wasted

Clean Code:
  - You write clean code
  - 6 months later: Need to add feature
  - Code is clear and organized ✅
  - Takes 30 minutes to understand
  - Takes 1 hour to add feature  
  - Total: 1.5 hours
```

**Clean Code Benefits:**

1. **Easy to understand** (for you and team)
2. **Easy to modify** (add features quickly)
3. **Easy to debug** (find bugs faster)
4. **Easy to test** (write tests easily)
5. **Professional** (get hired at top companies!)

### Real-World Impact

```
Company with messy code:
  → Slow development
  → Many bugs
  → Developers frustrated
  → Company loses money 💸

Company with clean code:
  → Fast development
  → Fewer bugs
  → Developers happy
  → Company makes money 💰
```

---

## Current Code Problems

### Problem 1: CORS Repeated Everywhere

**In getProducts:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Habib")
    w.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
    
    // ... rest of code
}
```

**In createProduct:**
```go
func createProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
    w.Header().Set("Access-Control-Allow-Methods", "POST")
    
    // ... rest of code
}
```

**Problem:** Same CORS code repeated! 😱

### Problem 2: OPTIONS Handling Repeated

**In getProducts:**
```go
if r.Method == http.MethodOptions {
    w.WriteHeader(http.StatusOK)
    return
}
```

**In createProduct:**
```go
if r.Method == http.MethodOptions {
    w.WriteHeader(http.StatusOK)
    return
}
```

**Problem:** Same preflight handling repeated! 😱

### Problem 3: Functions Do Too Much

**getProducts function:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // 1. Handle CORS
    // 2. Handle OPTIONS
    // 3. Validate method
    // 4. Encode response
    // 5. Send response
}
```

**Problems:**
- Function has 5 responsibilities! (Should have 1!)
- Hard to test individual parts
- Hard to reuse logic

### Problem 4: Duplicate Response Logic

```go
// In getProducts:
encoder := json.NewEncoder(w)
encoder.Encode(productList)

// In createProduct:
encoder := json.NewEncoder(w)
encoder.Encode(newProduct)
```

**Problem:** Same encoding pattern repeated!

### Code Smell Summary

```
┌────────────────────────────────────────┐
│           Code Smells 🤢               │
├────────────────────────────────────────┤
│  ❌ Copy-paste code                    │
│  ❌ Functions doing multiple things    │
│  ❌ Same logic repeated                │
│  ❌ Hard to understand                 │
│  ❌ Hard to maintain                   │
└────────────────────────────────────────┘
```

---

## Single Responsibility Principle

### What is SRP?

**Single Responsibility Principle (SRP):**
> "A function should have ONE and ONLY ONE reason to change"

**Translation:** Each function should do ONE thing only!

### From SOLID Principles

**SOLID** is a famous software engineering philosophy:

```
S - Single Responsibility Principle
O - Open/Closed Principle
L - Liskov Substitution Principle
I - Interface Segregation Principle
D - Dependency Inversion Principle
```

**Today we focus on "S" - Single Responsibility**

### SRP Examples

**❌ BAD (Multiple Responsibilities):**

```go
func processOrder(order Order) {
    // Responsibility 1: Validate order
    if order.Total <= 0 {
        return
    }
    
    // Responsibility 2: Save to database
    db.Save(order)
    
    // Responsibility 3: Send email
    sendEmail(order.Email, "Order confirmed")
    
    // Responsibility 4: Update inventory
    inventory.Decrease(order.ProductID)
}
```

**✅ GOOD (Single Responsibility):**

```go
func validateOrder(order Order) error {
    if order.Total <= 0 {
        return errors.New("invalid total")
    }
    return nil
}

func saveOrder(order Order) error {
    return db.Save(order)
}

func sendOrderEmail(email string) error {
    return sendEmail(email, "Order confirmed")
}

func updateInventory(productID int) error {
    return inventory.Decrease(productID)
}

func processOrder(order Order) error {
    if err := validateOrder(order); err != nil {
        return err
    }
    if err := saveOrder(order); err != nil {
        return err
    }
    if err := sendOrderEmail(order.Email); err != nil {
        return err
    }
    return updateInventory(order.ProductID)
}
```

**Benefits:**
- Each function has ONE clear purpose
- Easy to test individually
- Easy to modify without breaking others
- Easy to understand

### Visual Analogy

```
❌ Bad: One Person Does Everything
┌────────────────────────────────┐
│  Chef:                         │
│    - Cooks food                │
│    - Takes orders              │
│    - Cleans tables             │
│    - Handles money             │
│    - Washes dishes             │
│                                │
│  Result: Overwhelmed! 😰       │
└────────────────────────────────┘

✅ Good: Each Person Has One Job
┌────────────────────────────────┐
│  Chef: Cooks food              │
│  Waiter: Takes orders          │
│  Cleaner: Cleans tables        │
│  Cashier: Handles money        │
│  Dishwasher: Washes dishes     │
│                                │
│  Result: Efficient! 😊         │
└────────────────────────────────┘
```

---

## Refactoring Step 1: Extract CORS Handler

### Problem Code

**CORS headers repeated in every handler:**

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Habib")
    // ...
}

func createProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
    // ...
}
```

### Solution: Create handleCors Function

```go
// Responsibility: Handle CORS headers
func handleCors(w http.ResponseWriter) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Habib")
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
}
```

**Single Responsibility:** This function ONLY handles CORS headers. Nothing else!

### Using handleCors

**Before:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Habib")
    
    // ... rest of code
}
```

**After:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)  // One line! 🎉
    
    // ... rest of code
}
```

### Benefits

```
Before:
  - 4 lines of CORS code in getProducts
  - 4 lines of CORS code in createProduct
  - If we need to change CORS: Change in 2 places!
  - Total: 8 lines

After:
  - 1 line in getProducts (handleCors call)
  - 1 line in createProduct (handleCors call)
  - If we need to change CORS: Change in 1 place!
  - Total: 2 lines + 1 function
  
Saved: 5 lines, easier maintenance!
```

---

## Refactoring Step 2: Extract Preflight Handler

### Problem Code

**OPTIONS handling repeated:**

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // ... rest
}

func createProduct(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // ... rest
}
```

### Solution: Create handlePreflightRequest Function

```go
// Responsibility: Handle OPTIONS preflight requests
func handlePreflightRequest(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
}
```

**Wait!** This doesn't work! Why?

### The Return Problem

```go
func handlePreflightRequest(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return  // This only returns from THIS function!
    }
}
```

**Problem:** `return` only exits `handlePreflightRequest`, not the calling function!

### Solution: Boolean Return

```go
// Returns true if it was OPTIONS (caller should stop)
// Returns false if not OPTIONS (caller should continue)
func handlePreflightRequest(w http.ResponseWriter, r *http.Request) bool {
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return true  // Was OPTIONS, caller should stop
    }
    return false  // Not OPTIONS, caller should continue
}
```

### Using handlePreflightRequest

**Before:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // ... rest of code
}
```

**After (Option 1 - Check return value):**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    
    if handlePreflightRequest(w, r) {
        return  // Was OPTIONS, stop here
    }
    
    // ... rest of code
}
```

**After (Option 2 - Simple call):**

If we structure our code carefully, we can simplify:

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    handlePreflightRequest(w, r)
    
    // Validate GET method
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET allowed", 400)
        return
    }
    
    // ... rest of code
}
```

**Note:** In the transcript, the instructor just calls it without checking return, which works if preflight check comes before method validation.

---

## Refactoring Step 3: Extract Response Sender

### Problem Code

**Encoding pattern repeated:**

```go
// In getProducts:
w.WriteHeader(http.StatusOK)
encoder := json.NewEncoder(w)
encoder.Encode(productList)

// In createProduct:
w.WriteHeader(http.StatusCreated)
encoder := json.NewEncoder(w)
encoder.Encode(newProduct)
```

### Solution: Create sendData Function

```go
// Responsibility: Send JSON response
func sendData(w http.ResponseWriter, data interface{}, statusCode int) {
    w.WriteHeader(statusCode)
    encoder := json.NewEncoder(w)
    encoder.Encode(data)
}
```

**Key Features:**

1. **`interface{}`**: Accepts ANY data type
   - Can send string
   - Can send int
   - Can send struct
   - Can send slice
   - Can send anything!

2. **`statusCode int`**: Flexible status codes
   - 200 for GET success
   - 201 for POST success
   - Any status code we want

### Understanding `interface{}`

**What is `interface{}`?**

```go
interface{}  // Empty interface = any type
```

**Why it works:**

```go
// Can pass string
sendData(w, "Hello", 200)

// Can pass int
sendData(w, 42, 200)

// Can pass struct
product := Product{ID: 1, Title: "Orange"}
sendData(w, product, 200)

// Can pass slice
products := []Product{...}
sendData(w, products, 200)
```

**Analogy:**

```
interface{} is like a universal container

Just like:
  - Universal power adapter works with any plug
  - Universal remote works with any device
  
interface{} accepts any data type!
```

### Using sendData

**Before:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    handlePreflightRequest(w, r)
    
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    
    // 3 lines to send response
    w.WriteHeader(http.StatusOK)
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}
```

**After:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    handlePreflightRequest(w, r)
    
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    
    // 1 line to send response! 🎉
    sendData(w, productList, http.StatusOK)
}
```

**Benefits:**

```
Before: 3 lines to send response
After:  1 line to send response
Saved:  2 lines per handler
Consistency: All responses sent the same way
```

---

## Before vs After Comparison

### Complete Code Comparison

**BEFORE (Messy Code):**

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Product struct {
    ID          int     `json:"id"`
    Title       string  `json:"title"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    ImageURL    string  `json:"image"`
}

var productList []Product

func init() {
    productList = []Product{
        {ID: 1, Title: "Orange", Description: "Fresh", Price: 100, ImageURL: "url1"},
        {ID: 2, Title: "Apple", Description: "Red", Price: 40, ImageURL: "url2"},
    }
}

// Messy handler - does too much!
func getProducts(w http.ResponseWriter, r *http.Request) {
    // CORS handling
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Habib")
    w.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
    
    // OPTIONS handling
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // Method validation
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    
    // Response sending
    w.WriteHeader(http.StatusOK)
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}

// Messy handler - repeats same logic!
func createProduct(w http.ResponseWriter, r *http.Request) {
    // CORS handling (REPEATED!)
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
    w.Header().Set("Access-Control-Allow-Methods", "POST")
    
    // OPTIONS handling (REPEATED!)
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // Method validation
    if r.Method != http.MethodPost {
        http.Error(w, "Only POST", 400)
        return
    }
    
    // Decode request
    var newProduct Product
    decoder := json.NewDecoder(r.Body)
    decoder.Decode(&newProduct)
    
    // Set ID and save
    newProduct.ID = len(productList) + 1
    productList = append(productList, newProduct)
    
    // Response sending (REPEATED PATTERN!)
    w.WriteHeader(http.StatusCreated)
    encoder := json.NewEncoder(w)
    encoder.Encode(newProduct)
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/products", getProducts)
    mux.HandleFunc("/products/create", createProduct)
    http.ListenAndServe(":8080", mux)
}
```

**Problems:**
- 100+ lines
- CORS repeated 2 times
- OPTIONS handling repeated 2 times
- Response encoding repeated 2 times
- Hard to maintain

---

**AFTER (Clean Code):**

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Product struct {
    ID          int     `json:"id"`
    Title       string  `json:"title"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    ImageURL    string  `json:"image"`
}

var productList []Product

func init() {
    productList = []Product{
        {ID: 1, Title: "Orange", Description: "Fresh", Price: 100, ImageURL: "url1"},
        {ID: 2, Title: "Apple", Description: "Red", Price: 40, ImageURL: "url2"},
    }
}

// ============================================
// Reusable Helper Functions
// ============================================

// Responsibility: Handle CORS headers
func handleCors(w http.ResponseWriter) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Habib")
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
}

// Responsibility: Handle OPTIONS preflight
func handlePreflightRequest(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
}

// Responsibility: Send JSON response
func sendData(w http.ResponseWriter, data interface{}, statusCode int) {
    w.WriteHeader(statusCode)
    encoder := json.NewEncoder(w)
    encoder.Encode(data)
}

// ============================================
// Clean Handler Functions
// ============================================

// Responsibility: Get product list
func getProducts(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    handlePreflightRequest(w, r)
    
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    
    sendData(w, productList, http.StatusOK)
}

// Responsibility: Create new product
func createProduct(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    handlePreflightRequest(w, r)
    
    if r.Method != http.MethodPost {
        http.Error(w, "Only POST", 400)
        return
    }
    
    var newProduct Product
    decoder := json.NewDecoder(r.Body)
    decoder.Decode(&newProduct)
    
    newProduct.ID = len(productList) + 1
    productList = append(productList, newProduct)
    
    sendData(w, newProduct, http.StatusCreated)
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/products", getProducts)
    mux.HandleFunc("/products/create", createProduct)
    http.ListenAndServe(":8080", mux)
}
```

**Improvements:**
- Still ~100 lines, but much cleaner!
- CORS: 1 function, called 2 times
- OPTIONS: 1 function, called 2 times
- Response: 1 function, called 2 times
- Easy to maintain
- Easy to understand
- Easy to add new endpoints

### Visual Comparison

```
┌────────────────────────────────────────────────────────┐
│                  Before Refactoring                    │
├────────────────────────────────────────────────────────┤
│                                                        │
│  getProducts:                                          │
│    - CORS handling (4 lines)                           │
│    - OPTIONS handling (3 lines)                        │
│    - Method validation (3 lines)                       │
│    - Response sending (3 lines)                        │
│    Total: 13 lines                                     │
│                                                        │
│  createProduct:                                        │
│    - CORS handling (4 lines) ← REPEATED                │
│    - OPTIONS handling (3 lines) ← REPEATED             │
│    - Method validation (3 lines)                       │
│    - Decoding (3 lines)                                │
│    - Business logic (2 lines)                          │
│    - Response sending (3 lines) ← REPEATED             │
│    Total: 18 lines                                     │
│                                                        │
│  Problems: Lots of repetition! 😰                      │
│                                                        │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│                  After Refactoring                     │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Helper Functions:                                     │
│    - handleCors() (4 lines)                            │
│    - handlePreflightRequest() (4 lines)                │
│    - sendData() (4 lines)                              │
│                                                        │
│  getProducts:                                          │
│    - handleCors(w) (1 line)                            │
│    - handlePreflightRequest(w, r) (1 line)             │
│    - Method validation (3 lines)                       │
│    - sendData() (1 line)                               │
│    Total: 6 lines ✅                                   │
│                                                        │
│  createProduct:                                        │
│    - handleCors(w) (1 line)                            │
│    - handlePreflightRequest(w, r) (1 line)             │
│    - Method validation (3 lines)                       │
│    - Decoding (3 lines)                                │
│    - Business logic (2 lines)                          │
│    - sendData() (1 line)                               │
│    Total: 11 lines ✅                                  │
│                                                        │
│  Benefits: Clean, reusable, maintainable! 😊           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Testing After Refactoring

**Critical Rule:** After refactoring, ALWAYS test!

### Why Test?

```
You refactored code, but:
  - Does it still work?
  - Did you break something?
  - Are all features working?

Only way to know: TEST!
```

### Testing Process

**Step 1: Start Server**

```bash
go run main.go
```

**Expected output:**
```
Server listening on :8080...
```

**Step 2: Test GET Request**

**Using browser:**
```
http://localhost:8080/products
```

**Expected response:**
```json
[
  {
    "id": 1,
    "title": "Orange",
    "description": "Fresh",
    "price": 100,
    "image": "url1"
  },
  {
    "id": 2,
    "title": "Apple",
    "description": "Red",
    "price": 40,
    "image": "url2"
  }
]
```

**✅ Working!**

**Step 3: Test POST Request**

**Using Postman or curl:**

```bash
curl -X POST http://localhost:8080/products/create \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Banana",
    "description": "Yellow",
    "price": 5,
    "image": "url3"
  }'
```

**Expected response:**
```json
{
  "id": 3,
  "title": "Banana",
  "description": "Yellow",
  "price": 5,
  "image": "url3"
}
```

**✅ Working!**

**Step 4: Test Preflight**

**Open browser DevTools → Network tab**

Make request from frontend with custom header:
```javascript
fetch('http://localhost:8080/products', {
  headers: { 'Habib': 'Hello' }
})
```

**Check Network tab:**
```
OPTIONS /products → 200 OK ✅
GET /products → 200 OK ✅
```

**✅ Working!**

### Test Results

```
┌────────────────────────────────────┐
│         Test Results               │
├────────────────────────────────────┤
│  ✅ GET /products → Works          │
│  ✅ POST /products/create → Works  │
│  ✅ OPTIONS preflight → Works      │
│  ✅ CORS headers → Present         │
│  ✅ No errors in console           │
│                                    │
│  Conclusion: REFACTORING SUCCESS!  │
└────────────────────────────────────┘
```

---

## SOLID Principles Introduction

We just learned the **"S"** in SOLID! Let's understand the full picture.

### What is SOLID?

**SOLID** = 5 principles for writing clean, maintainable code

```
┌────────────────────────────────────────────┐
│              SOLID Principles              │
├────────────────────────────────────────────┤
│                                            │
│  S - Single Responsibility Principle       │
│      (Each function does ONE thing)        │
│                                            │
│  O - Open/Closed Principle                 │
│      (Open for extension, closed for mod)  │
│                                            │
│  L - Liskov Substitution Principle         │
│      (Subtypes must be substitutable)      │
│                                            │
│  I - Interface Segregation Principle       │
│      (Many specific interfaces > 1 big)    │
│                                            │
│  D - Dependency Inversion Principle        │
│      (Depend on abstractions, not details) │
│                                            │
└────────────────────────────────────────────┘
```

### S - Single Responsibility Principle (Today!)

**Definition:** A function/class should have ONE responsibility

**Example from our code:**

```go
// ✅ Good: Each function has ONE job
handleCors(w)                    // Job: Set CORS headers
handlePreflightRequest(w, r)     // Job: Handle OPTIONS
sendData(w, data, 200)           // Job: Send JSON response
```

**Why it matters:**
- Easy to understand
- Easy to test
- Easy to modify
- Easy to reuse

### O - Open/Closed Principle (Preview)

**Definition:** Open for extension, closed for modification

**Example:**
```go
// Instead of modifying existing function:
func getProducts() { ... }  // Don't touch this!

// Add new function (extension):
func getProductByID() { ... }  // Add this!
```

### Other Principles (Future Chapters)

We'll learn L, I, D in later chapters when we need them!

---

## Practice Questions

### Question 1: Identify Code Smell
**Q:** Look at this code. What's wrong? How would you refactor it?

```go
func processUser(user User) {
    // Validate
    if user.Email == "" {
        fmt.Println("Invalid email")
        return
    }
    
    // Save to database
    db.Save(user)
    
    // Send email
    sendEmail(user.Email, "Welcome!")
    
    // Log activity
    log.Println("User created:", user.ID)
}
```

<details>
<summary><b>Answer</b></summary>

**Problems (Code Smells):**

1. **Multiple Responsibilities** - Function does 4 things:
   - Validates user
   - Saves to database
   - Sends email
   - Logs activity

2. **Hard to Test** - Can't test validation without database
3. **Hard to Modify** - Changing email logic affects validation
4. **Violates SRP** - Single Responsibility Principle

**Refactored Version:**

```go
// ✅ Each function has ONE responsibility

func validateUser(user User) error {
    if user.Email == "" {
        return errors.New("invalid email")
    }
    return nil
}

func saveUser(user User) error {
    return db.Save(user)
}

func sendWelcomeEmail(email string) error {
    return sendEmail(email, "Welcome!")
}

func logUserCreation(userID int) {
    log.Println("User created:", userID)
}

func processUser(user User) error {
    // Orchestrate the steps
    if err := validateUser(user); err != nil {
        return err
    }
    
    if err := saveUser(user); err != nil {
        return err
    }
    
    if err := sendWelcomeEmail(user.Email); err != nil {
        return err
    }
    
    logUserCreation(user.ID)
    return nil
}
```

**Benefits:**
- Each function testable independently
- Can mock database for testing
- Can change email without touching validation
- Clear, readable, maintainable
</details>

---

### Question 2: Create Reusable Function
**Q:** This code is repeated in 3 handlers. Extract it to a reusable function:

```go
func handler1(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        http.Error(w, "Method not allowed", 405)
        return
    }
    // ... rest
}

func handler2(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", 405)
        return
    }
    // ... rest
}

func handler3(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodDelete {
        http.Error(w, "Method not allowed", 405)
        return
    }
    // ... rest
}
```

<details>
<summary><b>Answer</b></summary>

**Solution: Extract Method Validation Function**

```go
// Reusable function to validate HTTP method
func validateMethod(w http.ResponseWriter, r *http.Request, allowedMethod string) bool {
    if r.Method != allowedMethod {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return false  // Validation failed
    }
    return true  // Validation passed
}

// Using in handlers
func handler1(w http.ResponseWriter, r *http.Request) {
    if !validateMethod(w, r, http.MethodGet) {
        return  // Stop if validation failed
    }
    // ... rest of code
}

func handler2(w http.ResponseWriter, r *http.Request) {
    if !validateMethod(w, r, http.MethodPost) {
        return
    }
    // ... rest of code
}

func handler3(w http.ResponseWriter, r *http.Request) {
    if !validateMethod(w, r, http.MethodDelete) {
        return
    }
    // ... rest of code
}
```

**Even Better: Multiple Methods**

```go
// Support validating multiple methods
func validateMethod(w http.ResponseWriter, r *http.Request, allowedMethods ...string) bool {
    for _, method := range allowedMethods {
        if r.Method == method {
            return true  // Method is allowed
        }
    }
    
    http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    return false
}

// Usage: Accept GET or POST
func handler4(w http.ResponseWriter, r *http.Request) {
    if !validateMethod(w, r, http.MethodGet, http.MethodPost) {
        return
    }
    // ... handle both GET and POST
}
```

**Benefits:**
- DRY (Don't Repeat Yourself)
- Consistent error messages
- Easy to modify validation logic
- Flexible for multiple methods
</details>

---

### Question 3: Single Responsibility Check
**Q:** Does this function follow Single Responsibility Principle? Why or why not?

```go
func createProduct(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    handlePreflightRequest(w, r)
    
    if r.Method != http.MethodPost {
        http.Error(w, "Only POST", 400)
        return
    }
    
    var newProduct Product
    decoder := json.NewDecoder(r.Body)
    decoder.Decode(&newProduct)
    
    newProduct.ID = len(productList) + 1
    productList = append(productList, newProduct)
    
    sendData(w, newProduct, http.StatusCreated)
}
```

<details>
<summary><b>Answer</b></summary>

**Answer: NO, it does NOT follow SRP perfectly (but it's better than before!)**

**Current Responsibilities:**

1. ✅ Handle CORS (delegated to `handleCors`)
2. ✅ Handle preflight (delegated to `handlePreflightRequest`)
3. ❌ Validate method (done inline)
4. ❌ Decode request (done inline)
5. ❌ Generate ID (done inline)
6. ❌ Save product (done inline)
7. ✅ Send response (delegated to `sendData`)

**Technically, this function has 5 responsibilities!**

**Perfect SRP Version:**

```go
// Each responsibility gets its own function

func decodeProduct(r *http.Request) (Product, error) {
    var product Product
    decoder := json.NewDecoder(r.Body)
    err := decoder.Decode(&product)
    return product, err
}

func generateProductID() int {
    return len(productList) + 1
}

func saveProduct(product Product) {
    productList = append(productList, product)
}

func createProduct(w http.ResponseWriter, r *http.Request) {
    // Orchestration only!
    handleCors(w)
    handlePreflightRequest(w, r)
    
    if r.Method != http.MethodPost {
        http.Error(w, "Only POST", 400)
        return
    }
    
    newProduct, err := decodeProduct(r)
    if err != nil {
        http.Error(w, "Invalid JSON", 400)
        return
    }
    
    newProduct.ID = generateProductID()
    saveProduct(newProduct)
    
    sendData(w, newProduct, http.StatusCreated)
}
```

**Trade-off:**

Perfect SRP:
- More functions
- Easier to test
- Easier to modify
- Might be overkill for small projects

Pragmatic SRP (our version):
- Fewer functions
- Still maintainable
- Good enough for most projects
- Balance between clean and practical

**Conclusion:** Our refactored code is MUCH better than before, even if not 100% perfect SRP. In real projects, find the balance that works for your team!
</details>

---

### Question 4: interface{} Understanding
**Q:** Explain what `interface{}` means and why we use it in `sendData`. Give 3 examples of different data types you can pass.

<details>
<summary><b>Answer</b></summary>

**What is `interface{}`?**

`interface{}` (empty interface) is a special type in Go that can hold **ANY value of ANY type**.

**Definition:**
```go
interface{}  // No methods required = any type accepted
```

**Why We Use It:**

In our `sendData` function:
```go
func sendData(w http.ResponseWriter, data interface{}, statusCode int)
```

We use `interface{}` because we want to send **different types** of data:
- Sometimes a single product (struct)
- Sometimes a list of products (slice)
- Sometimes an error message (string)
- Sometimes a number (int)

**Example 1: Sending Struct**
```go
product := Product{
    ID:    1,
    Title: "Orange",
    Price: 100,
}

sendData(w, product, 200)
// JSON: {"id":1,"title":"Orange","price":100}
```

**Example 2: Sending Slice**
```go
products := []Product{
    {ID: 1, Title: "Orange", Price: 100},
    {ID: 2, Title: "Apple", Price: 40},
}

sendData(w, products, 200)
// JSON: [{"id":1,...},{"id":2,...}]
```

**Example 3: Sending Map**
```go
response := map[string]string{
    "message": "Product created successfully",
    "status":  "success",
}

sendData(w, response, 201)
// JSON: {"message":"Product created...","status":"success"}
```

**Example 4: Sending String** (though not ideal for JSON)
```go
message := "Hello World"
sendData(w, message, 200)
// JSON: "Hello World"
```

**Example 5: Sending Number**
```go
count := len(productList)
sendData(w, count, 200)
// JSON: 3
```

**How Does It Work?**

`json.Encoder` uses reflection to inspect the actual type at runtime:

```go
encoder := json.NewEncoder(w)
encoder.Encode(data)  // data is interface{}

// Internally:
// - If data is Product → encode as object
// - If data is []Product → encode as array
// - If data is string → encode as string
// - etc.
```

**Visual Analogy:**

```
interface{} is like a universal box:

┌─────────────────────────────┐
│  Universal Box (interface{})│
├─────────────────────────────┤
│                             │
│  Can hold:                  │
│  📦 Product                 │
│  📦 []Product               │
│  📦 map[string]string       │
│  📦 string                  │
│  📦 int                     │
│  📦 ANYTHING!               │
│                             │
└─────────────────────────────┘
```

**Important Notes:**

1. **Type Safety Lost:**
```go
func sendData(data interface{}) {
    // Can't call data.Title - compiler doesn't know it's Product!
    // Would need type assertion: product := data.(Product)
}
```

2. **Runtime vs Compile Time:**
```go
// Compiler accepts anything:
sendData(w, "string", 200)    // ✅ Compiles
sendData(w, 12345, 200)       // ✅ Compiles
sendData(w, product, 200)     // ✅ Compiles

// Errors only at runtime if json.Encode fails
```

3. **Alternative (Generics in Go 1.18+):**
```go
// Modern Go with generics:
func sendData[T any](w http.ResponseWriter, data T, statusCode int) {
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(data)
}
```

**Conclusion:** `interface{}` gives us flexibility to send any data type through a single function, making our code reusable and clean!
</details>

---

### Question 5: Complete Refactoring Challenge
**Q:** Refactor this messy code following what you learned today:

```go
func updateProduct(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    if r.Method != http.MethodPut {
        http.Error(w, "Only PUT", 400)
        return
    }
    
    var updatedProduct Product
    decoder := json.NewDecoder(r.Body)
    decoder.Decode(&updatedProduct)
    
    // Find and update
    for i, p := range productList {
        if p.ID == updatedProduct.ID {
            productList[i] = updatedProduct
            break
        }
    }
    
    w.WriteHeader(http.StatusOK)
    encoder := json.NewEncoder(w)
    encoder.Encode(updatedProduct)
}
```

<details>
<summary><b>Answer</b></summary>

**Refactored Version:**

```go
// Reuse existing helper functions:
// - handleCors(w)
// - handlePreflightRequest(w, r)
// - sendData(w, data, statusCode)

// New helper function for finding product
func findProductIndex(id int) int {
    for i, p := range productList {
        if p.ID == id {
            return i
        }
    }
    return -1  // Not found
}

// Clean updateProduct function
func updateProduct(w http.ResponseWriter, r *http.Request) {
    // 1. Handle CORS
    handleCors(w)
    
    // 2. Handle preflight
    handlePreflightRequest(w, r)
    
    // 3. Validate method
    if r.Method != http.MethodPut {
        http.Error(w, "Only PUT allowed", http.StatusMethodNotAllowed)
        return
    }
    
    // 4. Decode product
    var updatedProduct Product
    decoder := json.NewDecoder(r.Body)
    err := decoder.Decode(&updatedProduct)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    // 5. Find product
    index := findProductIndex(updatedProduct.ID)
    if index == -1 {
        http.Error(w, "Product not found", http.StatusNotFound)
        return
    }
    
    // 6. Update product
    productList[index] = updatedProduct
    
    // 7. Send response
    sendData(w, updatedProduct, http.StatusOK)
}
```

**Even Better: Extract More Functions**

```go
func decodeProductUpdate(r *http.Request) (Product, error) {
    var product Product
    decoder := json.NewDecoder(r.Body)
    err := decoder.Decode(&product)
    return product, err
}

func updateProductInList(product Product) error {
    index := findProductIndex(product.ID)
    if index == -1 {
        return errors.New("product not found")
    }
    productList[index] = product
    return nil
}

func updateProduct(w http.ResponseWriter, r *http.Request) {
    handleCors(w)
    handlePreflightRequest(w, r)
    
    if r.Method != http.MethodPut {
        http.Error(w, "Only PUT allowed", http.StatusMethodNotAllowed)
        return
    }
    
    updatedProduct, err := decodeProductUpdate(r)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    
    err = updateProductInList(updatedProduct)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }
    
    sendData(w, updatedProduct, http.StatusOK)
}
```

**Benefits of Refactored Code:**

1. **Reused existing helpers:**
   - `handleCors(w)`
   - `handlePreflightRequest(w, r)`
   - `sendData(w, data, statusCode)`

2. **New helper with single responsibility:**
   - `findProductIndex(id)` - only finds product

3. **Better error handling:**
   - Check decode error
   - Check if product exists
   - Proper status codes

4. **Cleaner main function:**
   - Each step clear
   - Easy to follow logic
   - Easy to modify

5. **Testable:**
   - Can test `findProductIndex` separately
   - Can test `decodeProductUpdate` separately
   - Can test `updateProductInList` separately

**Comparison:**

```
Before:
  - 20 lines in one function
  - CORS, OPTIONS, decoding, updating, encoding all mixed
  - Hard to test
  - Hard to modify

After:
  - Main function: 15 lines (clear orchestration)
  - Helper functions: Each with single purpose
  - Easy to test
  - Easy to modify
  - Professional quality! 🎉
```
</details>

---

## Summary

**What We Learned Today:**

1. ✅ **Why Refactoring Matters**
   - Improves code maintainability
   - Makes code easier to understand
   - Reduces bugs
   - Professional skill

2. ✅ **Single Responsibility Principle (SRP)**
   - Each function does ONE thing
   - Part of SOLID principles
   - Makes code testable and maintainable

3. ✅ **Three Key Refactorings:**
   - Extracted `handleCors()` - handles CORS headers
   - Extracted `handlePreflightRequest()` - handles OPTIONS
   - Extracted `sendData()` - sends JSON responses

4. ✅ **Practical Benefits:**
   - Reduced code duplication
   - Easier to modify
   - Easier to add new endpoints
   - Professional code quality

5. ✅ **Testing After Refactoring:**
   - Always test after refactoring!
   - Verify nothing broke
   - Test all endpoints

**Key Takeaways:**

```
┌────────────────────────────────────────┐
│         Refactoring Rules              │
├────────────────────────────────────────┤
│  1. Don't change behavior              │
│  2. Extract repeated code              │
│  3. One function = One responsibility  │
│  4. Test after refactoring             │
│  5. Make code readable                 │
└────────────────────────────────────────┘
```

---

## What's Next?

**Chapter 44: Project Structure & Package Organization**

Next class we'll learn:
- **Package organization** (handlers/, models/, middleware/)
- **Clean architecture** patterns
- **Separating concerns** (not everything in main.go!)
- **Project structure** for large applications

**Why This Matters:**

Right now everything is in `main.go` - this is fine for learning, but **terrible for real projects**!

```
Current (Bad for Real Projects):
main.go (300+ lines) 😱

Next (Good for Real Projects):
├── main.go (entry point only)
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

**What You'll Learn:**

1. **Go packages** - how to organize code
2. **Import/export** - public vs private
3. **Project structure** - industry standards
4. **Clean architecture** - professional patterns

This is **crucial** for getting hired! Companies want developers who can organize code properly!

---

**Congratulations! You just learned professional refactoring!** 🎉

Most junior developers never learn this - they just keep writing messy code. But you now understand:
- How to identify code smells
- How to apply SRP
- How to make code clean and maintainable

**You're becoming a professional developer!** Keep going! 🚀

---

**Remember:**
> "Any fool can write code that a computer can understand. Good programmers write code that humans can understand." - Martin Fowler

---
