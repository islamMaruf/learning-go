# Chapter 63: Pagination - Query Parameters, Offset, and Maps

## Table of Contents
- [Introduction](#introduction)
- [Understanding Pagination Concept](#understanding-pagination-concept)
- [Query Parameters in URLs](#query-parameters-in-urls)
- [Understanding Maps in Go](#understanding-maps-in-go)
- [Extracting Query Parameters](#extracting-query-parameters)
- [Calculating Offset](#calculating-offset)
- [Implementing Pagination in Handler](#implementing-pagination-in-handler)
- [Updating Service Layer](#updating-service-layer)
- [Updating Repository Layer](#updating-repository-layer)
- [Implementing Count Method](#implementing-count-method)
- [Creating Pagination Response](#creating-pagination-response)
- [Testing Pagination](#testing-pagination)
- [Memory Usage Comparison](#memory-usage-comparison)
- [Summary](#summary)
- [What's Next](#whats-next)

---

## Introduction

In Chapter 62, we saw the **problem**—loading all data crashes servers, exhausts memory, and costs a fortune.

Now it's time for the **solution**: **Pagination!**

**What we'll implement:**
```
Before:
GET /api/products
Returns: ALL 300,000 products (disaster!)

After:
GET /api/products?page=1&limit=10
Returns: 10 products + pagination metadata (beautiful!)
```

**Topics covered:**
```
✓ Query parameters (?page=1&limit=10)
✓ Maps in Go (key-value pairs)
✓ Offset calculation (how to skip records)
✓ SQL LIMIT and OFFSET
✓ Pagination metadata response
✓ Domain-Driven Design integration
✓ Memory optimization results
```

**Note:** This is a long class (30+ minutes) because I'm teaching you REAL pagination implementation, not just theory!

Let's build it!

---

## Understanding Pagination Concept

### The Problem Recap

**Scenario:**
```
Database: 107 products
Request: Give me ALL products

Result:
- 107 products loaded into memory
- 10 MB data transferred
- Slow performance
- High memory usage
```

### The Solution: Pagination

**Concept:**
```
Database: 107 products
Request: Give me 10 products per page

Result:
- Page 1: Products 1-10
- Page 2: Products 11-20
- Page 3: Products 21-30
...
- Page 11: Products 101-107 (last 7)

Total pages: 11 (107 ÷ 10 = 10.7 → 11 pages)
```

### Page Calculation

**Formula:**
```
Total items: 107
Items per page (limit): 10

Total pages = CEIL(Total items / Limit)
Total pages = CEIL(107 / 10)
Total pages = CEIL(10.7)
Total pages = 11
```

**In Go (integer division):**
```go
totalPages := totalItems / limit  // 107 / 10 = 10 (integer division)

// Need to add 1 if there's a remainder
if totalItems % limit != 0 {
    totalPages++  // 10 + 1 = 11
}
```

### Which Products on Which Page?

**Page 1 (limit=10):**
```
Start: 1
End: 10
Products: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
```

**Page 2 (limit=10):**
```
Start: 11
End: 20
Products: 11, 12, 13, 14, 15, 16, 17, 18, 19, 20
```

**Page 4 (limit=10):**
```
Calculation:
Page 4 means we skip first 3 pages
3 pages × 10 items = 30 items skipped

Start: 31
End: 40
Products: 31, 32, 33, 34, 35, 36, 37, 38, 39, 40
```

**Formula:**
```
Start = (page - 1) × limit + 1
End = page × limit

Page 4, Limit 10:
Start = (4 - 1) × 10 + 1 = 31
End = 4 × 10 = 40
```

### Different Limit Example

**Page 3 (limit=20):**
```
First 2 pages: 2 × 20 = 40 items shown

Start: 41
End: 60
Products: 41, 42, 43, ..., 60

Calculation:
(3 - 1) × 20 + 1 = 41
3 × 20 = 60
```

---

## Query Parameters in URLs

### What are Query Parameters?

**Example URL:**
```
http://localhost:4000/api/products?page=4&limit=10
                                   ↑
                            Question mark starts query params
```

**Breaking it down:**
```
Base URL: http://localhost:4000/api/products
Query Parameters: page=4&limit=10

Parameter 1:
  Key: page
  Value: 4

Parameter 2:
  Key: limit
  Value: 10
```

**Multiple parameters separated by `&`:**
```
?page=1&limit=10&sort=price&order=asc

Parameters:
- page = 1
- limit = 10
- sort = price
- order = asc
```

### Query Params vs Path Params

**Path parameters:**
```
GET /api/products/5
                 ↑
            Product ID in URL path
```

**Query parameters:**
```
GET /api/products?page=2
                 ↑
            Parameters after ?
```

**When to use which:**

```
Path params:
✓ Identifying specific resource
✓ Required parameters
✓ RESTful resource IDs
Examples: /users/123, /products/456

Query params:
✓ Filtering, sorting, pagination
✓ Optional parameters
✓ Multiple options
Examples: ?page=1&limit=10&sort=name
```

### In Postman

**Params tab:**
```
In Postman, instead of manually typing:
?page=4&limit=10

Use the "Params" tab:
┌─────────┬────────┬──────────┐
│  Key    │ Value  │ ☑        │
├─────────┼────────┼──────────┤
│ page    │ 4      │ ☑ Enabled│
│ limit   │ 10     │ ☑ Enabled│
└─────────┴────────┴──────────┘

Postman auto-generates:
http://localhost:4000/api/products?page=4&limit=10
```

**Benefits:**
```
✓ No manual URL editing
✓ Easy enable/disable
✓ No syntax errors
✓ Visual organization
```

---

## Understanding Maps in Go

### What is a Map?

**Concept:**
> **A map is a key-value pair. You map one value to another value.**

**Simple example:**

```go
// Map: number → count
// How many times each number appears?

numbers := map[int]int{
    1: 1,  // 1 appears 1 time
    2: 5,  // 2 appears 5 times
    3: 2,  // 3 appears 2 times
}

fmt.Println(numbers[2])  // Output: 5
```

**What happened:**
```
We have a collection:
[1, 2, 2, 2, 2, 2, 3, 3]

Count frequency:
1 → appears 1 time
2 → appears 5 times
3 → appears 2 times

Map stores: number → count
```

### Map Syntax

**Declaration:**

```go
// Empty map
var m map[string]int

// Make map
m = make(map[string]int)

// Map literal
m := map[string]int{
    "apple":  5,
    "banana": 3,
}

// Generic syntax
map[KeyType]ValueType
```

**Examples:**

```go
// String → Int
ages := map[string]int{
    "Alice": 25,
    "Bob":   30,
}

// String → String
capitals := map[string]string{
    "Bangladesh": "Dhaka",
    "India":      "New Delhi",
}

// String → []String (string slice)
hobbies := map[string][]string{
    "Alice": {"reading", "swimming"},
    "Bob":   {"gaming", "cooking"},
}
```

### Accessing Map Values

**Getting values:**

```go
ages := map[string]int{
    "page":  1,
    "limit": 10,
}

// Access value
page := ages["page"]     // page = 1
limit := ages["limit"]   // limit = 10

fmt.Println(page)   // Output: 1
fmt.Println(limit)  // Output: 10
```

**Changing example:**

```go
params := map[string]string{
    "page":  "1",
    "limit": "10",
}

// Change page to limit
fmt.Println(params["page"])   // Output: "1"
fmt.Println(params["limit"])  // Output: "10"
```

### String to String Slice Map

**Why slice?**

```go
// Query param can have multiple values
?color=red&color=blue&color=green

// Map needs to store array:
colors := map[string][]string{
    "color": {"red", "blue", "green"},
}

// Or single value:
?page=1

page := map[string][]string{
    "page": {"1"},  // Still array, but one element
}
```

**This is exactly what query parameters return!**

```go
// Request: ?page=1&limit=10

queryParams := map[string][]string{
    "page":  {"1"},     // Array with one element
    "limit": {"10"},    // Array with one element
}
```

---

## Extracting Query Parameters

### Getting Query from Request

**In handler:**

```go
func (h *Handler) GetProducts(c *gin.Context) {
    // Get query parameters from URL
    queryParams := c.Request.URL.Query()
    
    // Type: url.Values (which is map[string][]string)
}
```

**What is `c.Request.URL.Query()`?**

```go
// c.Request: The HTTP request
// c.Request.URL: The URL of the request
// c.Request.URL.Query(): Parses query parameters

// Returns: url.Values
// Which is: map[string][]string
```

**Example:**

```
Request: GET /api/products?page=4&limit=10

c.Request.URL.Query() returns:
map[string][]string{
    "page":  {"4"},
    "limit": {"10"},
}
```

### The Get Method

**Problem: Array values**

```go
queryParams := c.Request.URL.Query()
// queryParams["page"] = {"4"}  // It's an array!

// We want just the string "4", not array
```

**Solution: Get() method**

```go
page := queryParams.Get("page")
// Returns: "4" (string, not array)

limit := queryParams.Get("limit")
// Returns: "10" (string, not array)
```

**What Get() does:**

```go
// Simplified implementation of Get()
func (v Values) Get(key string) string {
    values := v[key]  // Get array
    
    if len(values) == 0 {
        return ""  // Empty if key doesn't exist
    }
    
    return values[0]  // Return first element
}
```

**Examples:**

```go
// Query: ?page=1&page=2&page=3
params := c.Request.URL.Query()
params["page"]      // {"1", "2", "3"}
params.Get("page")  // "1" (only first!)

// Query: ?page=5
params := c.Request.URL.Query()
params["page"]      // {"5"}
params.Get("page")  // "5"

// Query: (no page parameter)
params := c.Request.URL.Query()
params["page"]      // nil
params.Get("page")  // "" (empty string)
```

### Complete Example

```go
func (h *Handler) GetProducts(c *gin.Context) {
    // Get query parameters
    queryParams := c.Request.URL.Query()
    
    // Extract page and limit as strings
    pageStr := queryParams.Get("page")
    limitStr := queryParams.Get("limit")
    
    fmt.Println("Page:", pageStr)    // "4"
    fmt.Println("Limit:", limitStr)  // "10"
    
    // But they're strings! We need integers!
}
```

### Converting String to Int

**Using strconv.ParseInt:**

```go
import "strconv"

// String to int64
page, err := strconv.ParseInt(pageStr, 10, 64)
//                            ↑      ↑   ↑
//                         string  base bits

// If successful:
// page = 4 (int64)
// err = nil

// If error (invalid string):
// page = 0
// err = <error message>
```

**Parameters explained:**

```go
strconv.ParseInt(string, base, bitSize)

string:  "123" → the string to convert
base:    10 → decimal (base 10)
         2 → binary, 8 → octal, 16 → hex
bitSize: 32 → int32, 64 → int64
```

**Complete conversion:**

```go
// Convert page
page, err := strconv.ParseInt(pageStr, 10, 64)
if err != nil {
    page = 1  // Default to page 1
}

// Convert limit
limit, err := strconv.ParseInt(limitStr, 10, 64)
if err != nil {
    limit = 10  // Default to 10 items
}
```

**Why errors happen:**

```go
// Valid conversions:
strconv.ParseInt("123", 10, 64)   // ✓ 123
strconv.ParseInt("0", 10, 64)     // ✓ 0
strconv.ParseInt("-456", 10, 64)  // ✓ -456

// Invalid conversions:
strconv.ParseInt("abc", 10, 64)   // ❌ Error!
strconv.ParseInt("", 10, 64)      // ❌ Error!
strconv.ParseInt("12.5", 10, 64)  // ❌ Error! (float)
```

---

## Calculating Offset

### What is Offset?

**Offset = Number of records to skip**

**Example:**

```
Database has 100 products:
[1, 2, 3, 4, ..., 98, 99, 100]

Page 1 (limit 10): Skip 0, take 10
→ Products: 1-10

Page 2 (limit 10): Skip 10, take 10
→ Products: 11-20

Page 3 (limit 10): Skip 20, take 10
→ Products: 21-30
```

### Offset Formula

**Formula:**
```
offset = (page - 1) × limit
```

**Examples:**

**Page 1, Limit 10:**
```
offset = (1 - 1) × 10 = 0
Skip 0 records, start from 1
Products: 1-10
```

**Page 2, Limit 10:**
```
offset = (2 - 1) × 10 = 10
Skip 10 records, start from 11
Products: 11-20
```

**Page 3, Limit 20:**
```
offset = (3 - 1) × 20 = 40
Skip 40 records, start from 41
Products: 41-60
```

**Page 4, Limit 10:**
```
offset = (4 - 1) × 10 = 30
Skip 30 records, start from 31
Products: 31-40
```

### Why This Works

**Visual explanation:**

```
Page 4, Limit 10:

First 3 pages showed:
Page 1: 10 products
Page 2: 10 products
Page 3: 10 products
Total: 30 products shown

So Page 4 should start AFTER 30 products:
Offset = 30
Start from product 31
```

### Offset in Go

```go
// Given page and limit
page := int64(4)
limit := int64(10)

// Calculate offset
offset := (page - 1) * limit
// offset = (4 - 1) × 10 = 30

// Add 1 to get start position
start := offset + 1
// start = 30 + 1 = 31

// End position
end := page * limit
// end = 4 × 10 = 40

// So Page 4 shows products 31-40
```

---

## Implementing Pagination in Handler

### Step 1: Update Handler to Accept Query Params

**File: `handler/product/handler.go`**

```go
func (h *Handler) GetProducts(c *gin.Context) {
    // 1. Get query parameters
    queryParams := c.Request.URL.Query()
    
    // 2. Extract page and limit as strings
    pageStr := queryParams.Get("page")
    limitStr := queryParams.Get("limit")
    
    // 3. Convert to int64
    page, _ := strconv.ParseInt(pageStr, 10, 64)
    limit, _ := strconv.ParseInt(limitStr, 10, 64)
    
    // 4. Set defaults if invalid
    if page <= 0 {
        page = 1  // Default: page 1
    }
    
    if limit <= 0 {
        limit = 10  // Default: 10 items per page
    }
    
    // 5. Call service with pagination
    products, err := h.productService.List(page, limit)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{
            "error": err.Error(),
        })
        return
    }
    
    // 6. Return products
    c.JSON(http.StatusOK, products)
}
```

**What changed:**

```go
// Before:
products, err := h.productService.List()

// After:
products, err := h.productService.List(page, limit)
//                                     ↑      ↑
//                              Now accepts pagination params!
```

### Step 2: Import strconv

**Add import:**

```go
import (
    "net/http"
    "strconv"  // ← Add this
    
    "github.com/gin-gonic/gin"
    "yourproject/domain"
    productDomain "yourproject/domain/product"
)
```

---

## Updating Service Layer

### Update Service Interface

**File: `domain/product/port.go`**

**Before:**
```go
type Service interface {
    List() ([]*domain.Product, error)
}
```

**After:**
```go
type Service interface {
    List(page, limit int64) ([]*domain.Product, error)
    //      ↑      ↑
    //   Add pagination parameters
}
```

### Update Service Implementation

**File: `domain/product/service.go`**

**Before:**
```go
func (s *service) List() ([]*domain.Product, error) {
    return s.productRepo.List()
}
```

**After:**
```go
func (s *service) List(page, limit int64) ([]*domain.Product, error) {
    // Pass pagination to repository
    return s.productRepo.List(page, limit)
}
```

**What happens:**
```
Handler (page=4, limit=10)
    ↓
Service.List(4, 10)
    ↓
Repository.List(4, 10)
    ↓
Database (OFFSET 30 LIMIT 10)
```

---

## Updating Repository Layer

### Update Repository Interface

**File: `domain/product/port.go`**

**Before:**
```go
type Repository interface {
    List() ([]*domain.Product, error)
}
```

**After:**
```go
type Repository interface {
    List(page, limit int64) ([]*domain.Product, error)
}
```

### Update Repository Implementation

**File: `database/product.go`**

**Before:**
```go
func (r *ProductRepository) List() ([]*domain.Product, error) {
    var products []*domain.Product
    
    query := "SELECT * FROM products ORDER BY id"
    
    err := r.DB.Select(&products, query)
    if err != nil {
        return nil, err
    }
    
    return products, nil
}
```

**After:**
```go
func (r *ProductRepository) List(page, limit int64) ([]*domain.Product, error) {
    var products []*domain.Product
    
    // Calculate offset
    offset := (page - 1) * limit
    
    // Query with LIMIT and OFFSET
    query := `
        SELECT * FROM products 
        ORDER BY id 
        LIMIT $1 OFFSET $2
    `
    
    err := r.DB.Select(&products, query, limit, offset)
    if err != nil {
        return nil, err
    }
    
    return products, nil
}
```

**SQL explanation:**

```sql
-- Page 1, Limit 10:
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 0;
-- Returns products 1-10

-- Page 2, Limit 10:
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 10;
-- Returns products 11-20

-- Page 4, Limit 10:
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 30;
-- Returns products 31-40
```

**Placeholders:**
```go
query := "... LIMIT $1 OFFSET $2"
//                   ↑         ↑
//              First arg  Second arg

r.DB.Select(&products, query, limit, offset)
//                            ↑      ↑
//                          $1=$2
```

---

## Implementing Count Method

### Why We Need Count

**Problem:**

```go
// We return 10 products
// But frontend needs to know:
// - Total products in database
// - Total pages available
// - Current page number
```

**Solution: Count total products**

```sql
SELECT COUNT(*) FROM products;
-- Returns: 300010 (total products)
```

### Add Count to Service Interface

**File: `domain/product/port.go`**

```go
type Service interface {
    List(page, limit int64) ([]*domain.Product, error)
    Count() (int64, error)  // ← Add this
}
```

### Implement Count in Service

**File: `domain/product/service.go`**

**Create new file: `domain/product/count.go`**

```go
package product

func (s *service) Count() (int64, error) {
    // Delegate to repository
    return s.productRepo.Count()
}
```

**Why separate file?**

> **"If your service.go gets bigger than 100-200 lines, split it into separate files: create.go, list.go, count.go, etc. Each file handles ONE responsibility."**

### Add Count to Repository Interface

**File: `domain/product/port.go`**

```go
type Repository interface {
    List(page, limit int64) ([]*domain.Product, error)
    Count() (int64, error)  // ← Add this
}
```

### Implement Count in Repository

**File: `database/product.go`**

```go
func (r *ProductRepository) Count() (int64, error) {
    var count int64
    
    query := "SELECT COUNT(*) FROM products"
    
    // QueryRow for single value
    err := r.DB.QueryRow(query).Scan(&count)
    if err != nil {
        return 0, err
    }
    
    return count, nil
}
```

**Why QueryRow?**

```go
// For multiple rows: Select()
err := r.DB.Select(&products, query)

// For single value: QueryRow().Scan()
err := r.DB.QueryRow(query).Scan(&count)
```

**What happens:**

```sql
SELECT COUNT(*) FROM products;
-- Returns single row: 300010

Scan(&count) → count = 300010
```

---

## Creating Pagination Response

### Pagination Metadata Structure

**What frontend needs:**

```json
{
  "data": [...],
  "page": 4,
  "limit": 10,
  "total_items": 300010,
  "total_pages": 30001
}
```

### Create Pagination Struct

**File: `handler/product/handler.go`**

```go
type PaginatedResponse struct {
    Data       []*domain.Product `json:"data"`
    Page       int64             `json:"page"`
    Limit      int64             `json:"limit"`
    TotalItems int64             `json:"total_items"`
    TotalPages int64             `json:"total_pages"`
}
```

### Update Handler to Return Pagination

**File: `handler/product/handler.go`**

```go
func (h *Handler) GetProducts(c *gin.Context) {
    // ... (extract page and limit) ...
    
    // Get products
    products, err := h.productService.List(page, limit)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{
            "error": err.Error(),
        })
        return
    }
    
    // Get total count
    totalItems, err := h.productService.Count()
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{
            "error": err.Error(),
        })
        return
    }
    
    // Calculate total pages
    totalPages := totalItems / limit
    if totalItems%limit != 0 {
        totalPages++  // Add 1 if there's a remainder
    }
    
    // Create paginated response
    response := PaginatedResponse{
        Data:       products,
        Page:       page,
        Limit:      limit,
        TotalItems: totalItems,
        TotalPages: totalPages,
    }
    
    c.JSON(http.StatusOK, response)
}
```

### Total Pages Calculation

**Formula:**
```go
totalPages := totalItems / limit

// Integer division examples:
300010 / 10 = 30001 ✓ (exact division)
107 / 10 = 10 ❌ (missing last page!)

// Solution: Check remainder
if totalItems % limit != 0 {
    totalPages++  // 10 + 1 = 11 ✓
}
```

**Examples:**

```go
// Case 1: Exact division
totalItems = 100
limit = 10
totalPages = 100 / 10 = 10
remainder = 100 % 10 = 0
Result: 10 pages ✓

// Case 2: With remainder
totalItems = 107
limit = 10
totalPages = 107 / 10 = 10
remainder = 107 % 10 = 7 (7 items left!)
totalPages++ → 11 pages ✓

// Case 3: Less than limit
totalItems = 5
limit = 10
totalPages = 5 / 10 = 0
remainder = 5 % 10 = 5
totalPages++ → 1 page ✓
```

---

## Testing Pagination

### Start the Server

```bash
go run main.go
```

**Output:**
```
Successfully connected to PostgreSQL!
Successfully migrated database!
Server running on port 4000
```

### Test Page 1

**Request:**
```bash
GET http://localhost:4000/api/products?page=1&limit=10
Authorization: Bearer <your_jwt_token>
```

**Response:**
```json
{
  "data": [
    {"id": 2, "title": "Product 2", "price": 99.99, ...},
    {"id": 3, "title": "Product 3", "price": 99.99, ...},
    ...
    {"id": 11, "title": "Product 11", "price": 99.99, ...}
  ],
  "page": 1,
  "limit": 10,
  "total_items": 300010,
  "total_pages": 30001
}
```

**Time:** 13 milliseconds ⚡
**Size:** A few kilobytes 📊

**Compare to before:**
```
Before pagination:
Time: 705 ms (55× slower!)
Size: 20 MB (thousands× larger!)
```

### Test Page 2

**Request:**
```bash
GET http://localhost:4000/api/products?page=2&limit=10
```

**Response:**
```json
{
  "data": [
    {"id": 12, "title": "Product 12", ...},
    {"id": 13, "title": "Product 13", ...},
    ...
    {"id": 21, "title": "Product 21", ...}
  ],
  "page": 2,
  "limit": 10,
  "total_items": 300010,
  "total_pages": 30001
}
```

**Products 12-21 (correct!)** ✓

### Test Page 4

**Request:**
```bash
GET http://localhost:4000/api/products?page=4&limit=10
```

**Response starts from ID 32:**
```json
{
  "data": [
    {"id": 32, "title": "Product 32", ...},
    {"id": 33, "title": "Product 33", ...},
    ...
  ],
  "page": 4,
  ...
}
```

**Calculation:**
```
Page 4, Limit 10:
offset = (4 - 1) × 10 = 30
Skip first 30 products
Start from 31st product

But our data shows ID 32 starting...
(Probably ID 1, 30, 31 deleted in earlier tests)

The pagination LOGIC is correct! ✓
```

### Test Default Values

**Request (no parameters):**
```bash
GET http://localhost:4000/api/products
```

**Result:**
```json
{
  "data": [...],  // 10 products
  "page": 1,      // Default
  "limit": 10,    // Default
  ...
}
```

**Defaults work!** ✓

### Test Invalid Values

**Request:**
```bash
GET http://localhost:4000/api/products?page=abc&limit=-5
```

**Result:**
```json
{
  "data": [...],
  "page": 1,      // Invalid → default to 1
  "limit": 10,    // Invalid → default to 10
  ...
}
```

**Error handling works!** ✓

---

## Memory Usage Comparison

### Check Memory Usage

**Open Activity Monitor (Mac) or Task Manager (Windows):**

```
Process: main (your Go app)
```

### Before Pagination (from Chapter 62)

```
Loading ALL 300,000 products:

Memory usage:
- Start: 94 MB
- After request: 175 MB
- After 5 requests: 280 MB+
- Keeps growing with each request!

Response:
- Time: 705 ms
- Size: 20 MB
```

### After Pagination

```
Loading 10 products per page:

Memory usage:
- Start: 4.8 MB
- After request: 4.9 MB
- After 100 requests: 5.0 MB
- Stays stable! 📊

Response:
- Time: 13 ms (54× faster!)
- Size: A few KB (thousands× smaller!)
```

### Memory Increase Analysis

**Send multiple requests:**

```
Request 1: 4.8 MB
Request 2: 4.8 MB
Request 3: 4.9 MB
Request 4: 4.9 MB
Request 5: 5.0 MB
Request 6: 5.0 MB

Increase: Only 0.2 MB for 6 requests!
```

**Why so small?**

```
Each request:
- Loads 10 products only
- ~1 KB per product
- Total: ~10 KB per request
- Go's GC cleans up after response
- Memory stays low!
```

### Scaling Calculation

**Without pagination:**
```
1 user = 20 MB
10 users = 200 MB
100 users = 2 GB
1000 users = 20 GB
Result: Server crashes! 💥
```

**With pagination:**
```
1 user = 10 KB
10 users = 100 KB
100 users = 1 MB
1000 users = 10 MB
10,000 users = 100 MB
Result: Server happy! 😊
```

**Difference: 200× more efficient!**

---

## Summary

### What We Accomplished

**1. Implemented complete pagination:**
```go
✓ Query parameters extraction
✓ Default values (page=1, limit=10)
✓ Offset calculation
✓ SQL LIMIT and OFFSET
✓ Count total items
✓ Calculate total pages
✓ Pagination response structure
```

**2. Updated all layers:**
```
Handler:
✓ Extract query params
✓ Convert to int64
✓ Set defaults
✓ Build paginated response

Service:
✓ Pass pagination to repository
✓ Implement Count method

Repository:
✓ SQL with LIMIT and OFFSET
✓ COUNT(*) query
```

**3. Domain-Driven Design maintained:**
```
✓ Parent defines interface (port.go)
✓ Child implements (service.go, repository)
✓ Changes flow top-down
✓ Compile errors guide us
✓ One change = predictable impact
```

### Performance Improvements

**Before pagination:**
```
Request: GET /api/products

Response time: 705 ms
Response size: 20 MB
Memory usage: 175-280 MB
Scalability: Poor (crashes at 100 users)
Cost: $43,616/month
```

**After pagination:**
```
Request: GET /api/products?page=1&limit=10

Response time: 13 ms (54× faster!)
Response size: Few KB (thousands× smaller!)
Memory usage: 4.9 MB (stable!)
Scalability: Excellent (handles 10,000+ users)
Cost: $448/month (97× cheaper!)
```

### Key Concepts Learned

**1. Query Parameters:**
```
URL: ?page=4&limit=10
Access: c.Request.URL.Query()
Extract: queryParams.Get("page")
```

**2. Maps in Go:**
```go
map[KeyType]ValueType
map[string][]string  // Query params
map[string]int       // Simple mapping
```

**3. Offset Calculation:**
```
offset = (page - 1) × limit

Page 4, Limit 10:
offset = (4-1) × 10 = 30
Start from 31st item
```

**4. SQL Pagination:**
```sql
SELECT * FROM products
ORDER BY id
LIMIT 10 OFFSET 30;
```

**5. Total Pages:**
```go
totalPages := totalItems / limit
if totalItems % limit != 0 {
    totalPages++
}
```

---

## What's Next

**In Chapter 64, we'll enhance pagination!**

### Advanced Features

**1. Better response structure:**
```json
{
  "data": [...],
  "pagination": {
    "current_page": 4,
    "per_page": 10,
    "total": 300010,
    "total_pages": 30001,
    "has_next": true,
    "has_previous": true
  }
}
```

**2. Navigation helpers:**
```json
"links": {
  "first": "/api/products?page=1&limit=10",
  "prev": "/api/products?page=3&limit=10",
  "self": "/api/products?page=4&limit=10",
  "next": "/api/products?page=5&limit=10",
  "last": "/api/products?page=30001&limit=10"
}
```

**3. Reusable pagination utility:**
```go
// Can be used for ANY resource
pagination := utils.Paginate(totalItems, page, limit)
```

**4. Filtering + Pagination:**
```
GET /api/products?page=1&limit=10&category=electronics&min_price=100
```

**5. Sorting + Pagination:**
```
GET /api/products?page=1&limit=10&sort=price&order=desc
```

**6. Search + Pagination:**
```
GET /api/products?page=1&limit=10&search=laptop
```

### The Complete Picture

**After next chapter:**
```
✓ Pagination (done!)
✓ Metadata (next)
✓ Navigation (next)
✓ Filters (next)
✓ Sorting (next)
✓ Search (next)

Result: Production-ready API! 🚀
```

---

**Key Takeaway:**

> **"This was a 30-minute class because I'm showing you REAL implementation, not just theory. You now understand query parameters, maps, offset calculation, and complete pagination. Memory usage dropped from 280 MB to 5 MB. Response time dropped from 705ms to 13ms. This is REAL engineering!"**

**Remember:**
- Always paginate large datasets
- Never load all data at once
- Query parameters for optional filters
- Maps are key-value pairs
- Offset = (page - 1) × limit
- Count total items for metadata
- Memory and performance matter!

**See you in Chapter 64 where we make pagination even better!** 🎉
