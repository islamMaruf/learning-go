# Chapter 40: E-commerce Project - GET Request (Your First Real Project!)

## Table of Contents
- [Chapter 40: E-commerce Project - GET Request (Your First Real Project!)](#chapter-40-e-commerce-project---get-request-your-first-real-project)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Why This Project Matters](#why-this-project-matters)
  - [What is REST API?](#what-is-rest-api)
    - [REST is an Architecture](#rest-is-an-architecture)
    - [Key REST Concepts](#key-rest-concepts)
  - [HTTP Protocol - The Foundation](#http-protocol---the-foundation)
    - [What is a Protocol?](#what-is-a-protocol)
  - [HTTP Methods - The Big 5](#http-methods---the-big-5)
    - [1. GET](#1-get)
    - [2. POST](#2-post)
    - [3. PUT](#3-put)
    - [4. PATCH](#4-patch)
    - [5. DELETE](#5-delete)
  - [HTTP Status Codes - Server Communication](#http-status-codes---server-communication)
    - [Common Status Codes](#common-status-codes)
  - [Building Our First Endpoint](#building-our-first-endpoint)
    - [Step 1: Set Up Router](#step-1-set-up-router)
  - [Understanding Request and Response](#understanding-request-and-response)
    - [The Handler Function](#the-handler-function)
    - [What is `w` (ResponseWriter)?](#what-is-w-responsewriter)
    - [What is `r` (Request)?](#what-is-r-request)
    - [Check Request Method](#check-request-method)
  - [Creating Product Structure](#creating-product-structure)
    - [Define Product Struct](#define-product-struct)
    - [Why Uppercase Properties?](#why-uppercase-properties)
  - [JSON Encoding Magic](#json-encoding-magic)
    - [Create Global Product List](#create-global-product-list)
    - [Encode and Send JSON](#encode-and-send-json)
  - [Testing with Postman](#testing-with-postman)
    - [What is Postman?](#what-is-postman)
    - [Testing Our Endpoint](#testing-our-endpoint)
  - [Public vs Private Properties](#public-vs-private-properties)
    - [The Problem](#the-problem)
    - [The Solution: JSON Tags](#the-solution-json-tags)
    - [Why Uppercase in Go?](#why-uppercase-in-go)
  - [Complete Code Walkthrough](#complete-code-walkthrough)
    - [Full Working Code](#full-working-code)
    - [Execution Flow](#execution-flow)
  - [Practice Questions](#practice-questions)
    - [Question 1: HTTP Methods](#question-1-http-methods)
    - [Question 2: Status Codes](#question-2-status-codes)
    - [Question 3: ResponseWriter and Request](#question-3-responsewriter-and-request)
    - [Question 4: Public vs Private in Go](#question-4-public-vs-private-in-go)
    - [Question 5: init() Function](#question-5-init-function)
  - [Summary](#summary)
  - [What's Next?](#whats-next)

---

## Introduction

**Today's Class Name:** E-commerce Project - GET Request

Hello everyone! Today we're jumping into a **real project** - an **E-commerce Project**!

**Why no classes for so long?** I've been sick regularly and there's been a lot of pressure at the office. But now I've decided - no matter if I'm sick or the office has pressure, I need to finish this course. There's no point staying stuck here.

**What We'll Build:**
- An E-commerce backend project
- REST API endpoints
- Product listing system
- Complete GET request implementation

**The Big Picture:**
```
┌─────────────────┐          GET /products         ┌─────────────────┐
│                 │  ────────────────────────────>  │                 │
│   Frontend      │                                 │   Backend       │
│   (React)       │  <────────────────────────────  │   (Go)          │
│                 │         Product List            │                 │
└─────────────────┘                                 └─────────────────┘
```

---

## Why This Project Matters

**What You'll Learn:**
1. ✅ How to design REST APIs
2. ✅ Understanding HTTP methods deeply
3. ✅ Working with JSON encoding/decoding
4. ✅ Building real-world backend systems
5. ✅ Testing APIs with Postman

**The Frontend (React):**
I found a random project on GitHub for the frontend. We'll use React to show you the feel of how things work. But remember - **you don't need to know React to be a backend developer!**

Later, when you become more advanced, we won't show any frontend. We can build and test backend without frontend. But for now, to give you that **feel**, we're using React.

---

## What is REST API?

**REST = Representational State Transfer**

Let me break this down:

### REST is an Architecture

REST is an **architecture** (not a language, not a framework). When we build an API following this architecture, we call it a **REST API**.

```
REST Architecture Rules
         ↓
REST API Implementation
```

### Key REST Concepts

**1. Resources:**
Everything in REST is a **resource**. For example:
- `/products` - Products resource
- `/users` - Users resource
- `/orders` - Orders resource

**2. Representational State:**
When frontend requests a resource, backend sends it in a **representation** (like JSON or XML format).

```
Frontend Request:  "Give me products resource"
                         ↓
Backend Response: JSON representation of products
                         ↓
        This is "Representational State Transfer"
```

**Visual Example:**
```
┌──────────────┐                    ┌──────────────┐
│  Frontend    │   Request Resource │  Backend     │
│              │ ──────────────────> │              │
│              │                     │  Has:        │
│  Needs:      │                     │  - Products  │
│  - Product   │                     │  - Users     │
│  - User      │                     │  - Orders    │
│  - Order     │                     │              │
│              │ <────────────────── │              │
│              │  JSON/XML Response  │              │
└──────────────┘                    └──────────────┘
```

---

## HTTP Protocol - The Foundation

**HTTP = HyperText Transfer Protocol**

### What is a Protocol?

**Protocol = Rules**

HTTP is a protocol, which means it's a set of **rules** that define how client and server communicate.

```
┌─────────────────────────────────────────────┐
│         HTTP Protocol (Rules)               │
├─────────────────────────────────────────────┤
│  Rule 1: Use specific methods (GET, POST...) │
│  Rule 2: Use status codes (200, 404, 500...)│
│  Rule 3: Follow request/response format     │
│  Rule 4: Use headers properly               │
└─────────────────────────────────────────────┘
```

**Example Communication:**
```
Client: "Hey Server! I want products (using GET method)"
Server: "Here you go! (200 OK status, JSON data)"

Client: "Hey Server! I want user with ID 999"
Server: "Not found! (404 status)"
```

---

## HTTP Methods - The Big 5

HTTP protocol defines specific methods for communication. The **top 5 methods** that power 99% of applications:

### 1. GET
**Purpose:** Read/Retrieve data  
**Example:** Get list of products, Get user details

### 2. POST
**Purpose:** Create new data  
**Example:** Create new user, Add new product

### 3. PUT
**Purpose:** Update entire resource  
**Example:** Update complete user profile

### 4. PATCH
**Purpose:** Update partial resource  
**Example:** Update only user's email

### 5. DELETE
**Purpose:** Delete resource  
**Example:** Delete a product, Remove a user

```
┌────────────────────────────────────────────┐
│        HTTP Methods = CRUD Operations       │
├────────────────────────────────────────────┤
│  GET    →  Read                            │
│  POST   →  Create                          │
│  PUT    →  Update (Full)                   │
│  PATCH  →  Update (Partial)                │
│  DELETE →  Delete                          │
└────────────────────────────────────────────┘
```

**Real-World Analogy:**

Think of a fruit shop:
```
GET    = "Show me all fruits"
POST   = "Add this new fruit to inventory"
PUT    = "Replace entire fruit basket"
PATCH  = "Just change the price of oranges"
DELETE = "Remove mangoes from stock"
```

**Important Truth:**

If you master these 5 methods, you become a **web developer**! I could have taught you this on day one, and you'd memorize it. But then everyone would apply for Go developer jobs and competition would be crazy! 😄

That's why we're learning **deeply** - so you understand the internals, not just the syntax!

---

## HTTP Status Codes - Server Communication

Status codes tell the client what happened with their request.

### Common Status Codes

**200 - OK**
- Meaning: Everything is fine
- Use: Successful GET request

**201 - Created**
- Meaning: Resource successfully created
- Use: Successful POST request

**400 - Bad Request**
- Meaning: Your request is malformed/wrong
- Use: Invalid data sent by client

**404 - Not Found**
- Meaning: Resource doesn't exist
- Use: Requested item not found on server

**500 - Internal Server Error**
- Meaning: Server couldn't handle the request
- Use: Server-side error occurred

```
┌─────────────────────────────────────────────┐
│          Status Code Categories              │
├─────────────────────────────────────────────┤
│  2xx  →  Success (200, 201)                 │
│  4xx  →  Client Error (400, 404)            │
│  5xx  →  Server Error (500)                 │
└─────────────────────────────────────────────┘
```

**You don't need to memorize these!** As we work with them, you'll naturally remember. Just understand the concept for now.

---

## Building Our First Endpoint

Let's build the `/products` GET endpoint!

### Step 1: Set Up Router

```go
package main

import (
    "encoding/json"
    "net/http"
)

func main() {
    mux := http.NewServeMux()
    
    // Route: /products → Handler: getProducts
    mux.HandleFunc("/products", getProducts)
    
    // Start server on port 8080
    http.ListenAndServe(":8080", mux)
}
```

**What's Happening:**
- `mux` = Router (we learned this before!)
- `/products` = Route (resource name)
- `getProducts` = Handler function (business logic)

---

## Understanding Request and Response

### The Handler Function

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // w = Response Writer (to send data back)
    // r = Request (data from client)
}
```

### What is `w` (ResponseWriter)?

```
┌─────────────────────────────────────────┐
│        ResponseWriter (w)                │
├─────────────────────────────────────────┤
│  Job: Send data back to client          │
│  Usage: w.Write(), w.WriteHeader()      │
│  Example: Sending JSON response          │
└─────────────────────────────────────────┘
```

**Think of it like:**
```
Client: "I want fruits!"
        ↓
Backend: (Uses `w` to write response)
        ↓
Client: "Thanks! Here's my fruit list!"
```

### What is `r` (Request)?

```
┌─────────────────────────────────────────┐
│           Request (r)                    │
├─────────────────────────────────────────┤
│  Job: Contains all client information   │
│  Usage: r.Method, r.URL, r.Body         │
│  Example: Check if GET/POST method      │
└─────────────────────────────────────────┘
```

**Real-World Example:**

When you go to a fruit shop:
```
You say: "Give me 10 kg oranges" 
         ↑
    This is REQUEST (r)
    Contains:
    - What: Oranges
    - How much: 10 kg
    - Method: GET (asking for something)

Shopkeeper gives: Oranges in a bag
                 ↑
            This is RESPONSE (w)
            Contains:
            - Product: Oranges
            - Quantity: 10 kg
            - Price: $100
```

### Check Request Method

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // Only allow GET requests
    if r.Method != http.MethodGet {
        http.Error(w, "Please give me GET request", 400)
        return
    }
    
    // If GET, continue below...
}
```

**Why Check Method?**

This endpoint should only respond to GET requests. If someone sends POST, PUT, PATCH, or DELETE, we reject it with 400 (Bad Request).

```
┌────────────────────────────────────────┐
│     Method Validation Flow             │
├────────────────────────────────────────┤
│  GET Request     →  ✅ Process         │
│  POST Request    →  ❌ 400 Error       │
│  PUT Request     →  ❌ 400 Error       │
│  DELETE Request  →  ❌ 400 Error       │
└────────────────────────────────────────┘
```

---

## Creating Product Structure

### Define Product Struct

```go
type Product struct {
    ID          int     `json:"id"`
    Title       string  `json:"title"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    ImageURL    string  `json:"image"`
}
```

**What's in a Product?**
- **ID:** Unique identifier (1, 2, 3...)
- **Title:** Product name ("Orange", "Apple")
- **Description:** Product details
- **Price:** Cost (100.50, 40.00)
- **ImageURL:** Link to product image

### Why Uppercase Properties?

```go
// ✅ PUBLIC (Accessible outside package)
type Product struct {
    ID    int    // Capital I - Public
    Title string // Capital T - Public
}

// ❌ PRIVATE (Only accessible in this package)
type Product struct {
    id    int    // Lowercase i - Private
    title string // Lowercase t - Private
}
```

**Important Rule in Go:**
- **Uppercase** first letter = **Public** (exportable)
- **Lowercase** first letter = **Private** (internal only)

---

## JSON Encoding Magic

### Create Global Product List

```go
var productList []Product

func init() {
    // This runs before main()!
    
    product1 := Product{
        ID:          1,
        Title:       "Orange",
        Description: "Orange is red. I love orange.",
        Price:       100.00,
        ImageURL:    "https://example.com/orange.jpg",
    }
    
    product2 := Product{
        ID:          2,
        Title:       "Apple",
        Description: "Apple is green. I ate apple.",
        Price:       40.00,
        ImageURL:    "https://example.com/apple.jpg",
    }
    
    product3 := Product{
        ID:          3,
        Title:       "Banana",
        Description: "Banana is boring. I feel bored eating.",
        Price:       5.00,
        ImageURL:    "https://example.com/banana.jpg",
    }
    
    product4 := Product{
        ID:          4,
        Title:       "Grapes",
        Description: "Grapes are tasty. I enjoy them.",
        Price:       140.00,
        ImageURL:    "https://example.com/grapes.jpg",
    }
    
    product5 := Product{
        ID:          5,
        Title:       "Mango",
        Description: "Mango is my favorite. I love it very much.",
        Price:       200.00,
        ImageURL:    "https://example.com/mango.jpg",
    }
    
    product6 := Product{
        ID:          6,
        Title:       "Watermelon",
        Description: "Watermelon is refreshing. Perfect for summer.",
        Price:       80.00,
        ImageURL:    "https://example.com/watermelon.jpg",
    }
    
    // Add all products to list
    productList = append(productList, product1)
    productList = append(productList, product2)
    productList = append(productList, product3)
    productList = append(productList, product4)
    productList = append(productList, product5)
    productList = append(productList, product6)
}
```

**What's `init()` Function?**

```
Program Start
     ↓
init() runs first ✅
     ↓
main() runs second
     ↓
Rest of code...
```

So our `productList` is populated **before** the server even starts!

### Encode and Send JSON

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // Check method
    if r.Method != http.MethodGet {
        http.Error(w, "Please give me GET request", 400)
        return
    }
    
    // Create JSON encoder
    encoder := json.NewEncoder(w)
    
    // Encode productList and send to client
    encoder.Encode(productList)
}
```

**How JSON Encoding Works:**

```
Step 1: Create Encoder with ResponseWriter
        json.NewEncoder(w)
        
Step 2: Encoder converts Go struct to JSON
        encoder.Encode(productList)
        
Step 3: JSON automatically sent to client
        (Because encoder was created with `w`)
```

**Visual Flow:**
```
┌─────────────────────┐
│  Go Struct          │
│  productList        │
│  (6 products)       │
└──────────┬──────────┘
           │
           │ json.NewEncoder(w)
           │ encoder.Encode(productList)
           ↓
┌─────────────────────┐
│  JSON Format        │
│  [                  │
│    {id: 1, ...},    │
│    {id: 2, ...},    │
│    ...              │
│  ]                  │
└──────────┬──────────┘
           │
           │ Sent via ResponseWriter (w)
           ↓
┌─────────────────────┐
│  Client Receives    │
│  JSON Response      │
└─────────────────────┘
```

---

## Testing with Postman

### What is Postman?

**Postman** is a tool to test APIs without needing a frontend.

```
┌────────────────────────────────────┐
│         Postman                     │
├────────────────────────────────────┤
│  - Test REST APIs                  │
│  - Send GET, POST, PUT, DELETE     │
│  - See responses                   │
│  - No frontend needed!             │
└────────────────────────────────────┘
```

### Testing Our Endpoint

**1. Start the server:**
```bash
go run main.go
```

Output:
```
Server running on port 8080
```

**2. Open Postman**

**3. Create GET Request:**
```
Method: GET
URL: http://localhost:8080/products
```

**4. Click "Send"**

**5. See Response:**
```json
[
  {
    "id": 1,
    "title": "Orange",
    "description": "Orange is red. I love orange.",
    "price": 100,
    "image": "https://example.com/orange.jpg"
  },
  {
    "id": 2,
    "title": "Apple",
    "description": "Apple is green. I ate apple.",
    "price": 40,
    "image": "https://example.com/apple.jpg"
  },
  {
    "id": 3,
    "title": "Banana",
    "description": "Banana is boring. I feel bored eating.",
    "price": 5,
    "image": "https://example.com/banana.jpg"
  },
  {
    "id": 4,
    "title": "Grapes",
    "description": "Grapes are tasty. I enjoy them.",
    "price": 140,
    "image": "https://example.com/grapes.jpg"
  },
  {
    "id": 5,
    "title": "Mango",
    "description": "Mango is my favorite. I love it very much.",
    "price": 200,
    "image": "https://example.com/mango.jpg"
  },
  {
    "id": 6,
    "title": "Watermelon",
    "description": "Watermelon is refreshing. Perfect for summer.",
    "price": 80,
    "image": "https://example.com/watermelon.jpg"
  }
]
```

🎉 **Success!** You just built your first REST API endpoint!

---

## Public vs Private Properties

### The Problem

What if we want JSON property names to be different from struct property names?

**Example:**
```go
// Go Struct
type Product struct {
    ID int  // Uppercase (required for public)
}

// But we want JSON like:
{
  "id": 1  // Lowercase
}
```

### The Solution: JSON Tags

```go
type Product struct {
    ID          int     `json:"id"`           // JSON will be "id"
    Title       string  `json:"title"`        // JSON will be "title"
    Description string  `json:"description"`  // JSON will be "description"
    Price       float64 `json:"price"`        // JSON will be "price"
    ImageURL    string  `json:"image"`        // JSON will be "image" (not imageURL)
}
```

**How JSON Tags Work:**
```
┌─────────────────────────────────────────────┐
│         Struct Field    →    JSON Field     │
├─────────────────────────────────────────────┤
│  ID (capital)           →    "id" (small)   │
│  Title (capital)        →    "title" (small)│
│  ImageURL (camelCase)   →    "image" (custom)│
└─────────────────────────────────────────────┘
```

### Why Uppercase in Go?

**The Rule:**
- If property starts with **lowercase** → **Private** (only this package can access)
- If property starts with **Uppercase** → **Public** (any package can access)

**Why This Matters for JSON:**

```go
// ❌ WRONG - Won't work!
type Product struct {
    id    int  // Private - json package can't access!
    title string
}

// ✅ CORRECT - Works!
type Product struct {
    ID    int    `json:"id"`    // Public - json package can access
    Title string `json:"title"` // and we control JSON name with tags
}
```

**Visual Explanation:**
```
┌────────────────────────────────────────────┐
│  Package: main                             │
│  ┌──────────────────────────────┐          │
│  │ type Product struct {        │          │
│  │   id int  // Lowercase       │          │
│  │ }                            │          │
│  └──────────────────────────────┘          │
│                                            │
│  ✅ Can access from main package          │
│  ❌ Cannot access from json package       │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│  Package: main                             │
│  ┌──────────────────────────────┐          │
│  │ type Product struct {        │          │
│  │   ID int  // Uppercase       │          │
│  │ }                            │          │
│  └──────────────────────────────┘          │
│                                            │
│  ✅ Can access from main package          │
│  ✅ Can access from json package          │
└────────────────────────────────────────────┘
```

When `json.NewEncoder()` tries to encode your struct, it's running code from the `encoding/json` package (not your main package). So if your fields are lowercase (private), the json package **cannot see them**!

---

## Complete Code Walkthrough

### Full Working Code

```go
package main

import (
    "encoding/json"
    "net/http"
)

// Product struct - our data model
type Product struct {
    ID          int     `json:"id"`
    Title       string  `json:"title"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    ImageURL    string  `json:"image"`
}

// Global product list
var productList []Product

// init runs before main
func init() {
    // Create products
    product1 := Product{
        ID:          1,
        Title:       "Orange",
        Description: "Orange is red. I love orange.",
        Price:       100.00,
        ImageURL:    "https://example.com/orange.jpg",
    }
    
    product2 := Product{
        ID:          2,
        Title:       "Apple",
        Description: "Apple is green. I ate apple.",
        Price:       40.00,
        ImageURL:    "https://example.com/apple.jpg",
    }
    
    product3 := Product{
        ID:          3,
        Title:       "Banana",
        Description: "Banana is boring. I feel bored eating.",
        Price:       5.00,
        ImageURL:    "https://example.com/banana.jpg",
    }
    
    product4 := Product{
        ID:          4,
        Title:       "Grapes",
        Description: "Grapes are tasty. I enjoy them.",
        Price:       140.00,
        ImageURL:    "https://example.com/grapes.jpg",
    }
    
    product5 := Product{
        ID:          5,
        Title:       "Mango",
        Description: "Mango is my favorite. I love it very much.",
        Price:       200.00,
        ImageURL:    "https://example.com/mango.jpg",
    }
    
    product6 := Product{
        ID:          6,
        Title:       "Watermelon",
        Description: "Watermelon is refreshing. Perfect for summer.",
        Price:       80.00,
        ImageURL:    "https://example.com/watermelon.jpg",
    }
    
    // Populate product list
    productList = append(productList, product1)
    productList = append(productList, product2)
    productList = append(productList, product3)
    productList = append(productList, product4)
    productList = append(productList, product5)
    productList = append(productList, product6)
}

// GET /products handler
func getProducts(w http.ResponseWriter, r *http.Request) {
    // Only allow GET requests
    if r.Method != http.MethodGet {
        http.Error(w, "Please give me GET request", 400)
        return
    }
    
    // Create JSON encoder with ResponseWriter
    encoder := json.NewEncoder(w)
    
    // Encode and send product list
    encoder.Encode(productList)
}

func main() {
    // Create router
    mux := http.NewServeMux()
    
    // Register route
    mux.HandleFunc("/products", getProducts)
    
    // Start server
    println("Server running on port 8080")
    http.ListenAndServe(":8080", mux)
}
```

### Execution Flow

```
Program Start
    ↓
init() function runs
    ↓
6 products added to productList
    ↓
main() function runs
    ↓
Router created
    ↓
Route registered: /products → getProducts
    ↓
Server starts on port 8080
    ↓
Waiting for requests...

[Client sends GET /products]
    ↓
getProducts() called
    ↓
Check if method is GET? Yes ✅
    ↓
Create JSON encoder
    ↓
Encode productList to JSON
    ↓
Send JSON to client via ResponseWriter
    ↓
Client receives JSON response! 🎉
```

---

## Practice Questions

### Question 1: HTTP Methods
**Q:** What are the 5 main HTTP methods and what does each one do?

<details>
<summary><b>Answer</b></summary>

The 5 main HTTP methods are:

1. **GET** - Read/Retrieve data
   - Example: `GET /products` - Get list of products
   - Does not modify server data

2. **POST** - Create new data
   - Example: `POST /products` - Create new product
   - Sends data in request body

3. **PUT** - Update entire resource
   - Example: `PUT /products/1` - Replace product 1 completely
   - Replaces entire resource

4. **PATCH** - Update partial resource
   - Example: `PATCH /products/1` - Update only price of product 1
   - Updates specific fields only

5. **DELETE** - Delete resource
   - Example: `DELETE /products/1` - Remove product 1
   - Removes resource from server

**CRUD Mapping:**
```
GET    → READ
POST   → CREATE
PUT    → UPDATE (Full)
PATCH  → UPDATE (Partial)
DELETE → DELETE
```
</details>

---

### Question 2: Status Codes
**Q:** Explain what these HTTP status codes mean: 200, 201, 400, 404, 500

<details>
<summary><b>Answer</b></summary>

**HTTP Status Codes Explained:**

**200 - OK**
- Everything worked successfully
- Used for: Successful GET requests
- Example: "Here's your product list!"

**201 - Created**
- Resource successfully created
- Used for: Successful POST requests
- Example: "New product created successfully!"

**400 - Bad Request**
- Client sent invalid/malformed request
- Used for: Validation errors, wrong data format
- Example: "You sent invalid data - price cannot be negative"

**404 - Not Found**
- Requested resource doesn't exist on server
- Used for: Resource not found
- Example: "Product with ID 999 not found"

**500 - Internal Server Error**
- Server encountered an error
- Used for: Server-side crashes, bugs
- Example: "Database connection failed"

**Categories:**
```
2xx = Success
4xx = Client Error (your fault)
5xx = Server Error (server's fault)
```
</details>

---

### Question 3: ResponseWriter and Request
**Q:** What are `w http.ResponseWriter` and `r *http.Request` in a handler function? How do they work?

<details>
<summary><b>Answer</b></summary>

**Handler Function Signature:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

**w (ResponseWriter):**
- **Purpose:** Send data back to the client
- **Type:** Interface that implements Write methods
- **Usage:** Writing response data, setting headers, status codes

**Common operations:**
```go
// Send JSON
json.NewEncoder(w).Encode(data)

// Send plain text
w.Write([]byte("Hello"))

// Set header
w.Header().Set("Content-Type", "application/json")

// Set status code
w.WriteHeader(http.StatusOK)
```

**r (Request):**
- **Purpose:** Contains all information from client's request
- **Type:** Pointer to Request struct
- **Usage:** Reading request data, method, headers, body

**Common operations:**
```go
// Check method
if r.Method != http.MethodGet { ... }

// Get URL parameters
id := r.URL.Query().Get("id")

// Read body
body, _ := ioutil.ReadAll(r.Body)

// Get headers
authToken := r.Header.Get("Authorization")
```

**Analogy:**
```
Restaurant Order:

r (Request) = Customer's order
  - What they want (method: GET/POST)
  - Special instructions (headers)
  - Details (body data)

w (ResponseWriter) = Waiter delivers food
  - Brings the food (writes response)
  - Tells status (order ready, not available)
  - Provides receipt (response data)
```
</details>

---

### Question 4: Public vs Private in Go
**Q:** Why do we use uppercase first letters in struct fields? What happens if we use lowercase?

<details>
<summary><b>Answer</b></summary>

**Go's Visibility Rules:**

**Uppercase First Letter = Public (Exported)**
```go
type Product struct {
    ID    int    // ✅ Public - accessible from other packages
    Title string // ✅ Public - accessible from other packages
}
```

**Lowercase First Letter = Private (Unexported)**
```go
type Product struct {
    id    int    // ❌ Private - only accessible in same package
    title string // ❌ Private - only accessible in same package
}
```

**Why This Matters for JSON Encoding:**

When we use `json.NewEncoder()`, it's code from the `encoding/json` package (not our package). 

```go
// ❌ WON'T WORK - json package can't see private fields!
type Product struct {
    id int  // Private - json package can't access
}

// ✅ WORKS - json package can see public fields!
type Product struct {
    ID int `json:"id"`  // Public - json package can access
}
```

**The Flow:**
```
Your Code (main package)
    ↓
Calls json.NewEncoder(w)
    ↓
json package code runs
    ↓
Tries to read Product fields
    ↓
If lowercase → Can't see it! ❌
If uppercase → Can see it! ✅
```

**How to Control JSON Names:**

Use JSON tags to control the JSON field names:
```go
type Product struct {
    ID       int    `json:"id"`    // Go: ID, JSON: id
    ImageURL string `json:"image"` // Go: ImageURL, JSON: image
}
```

**Best Practice:**
- Struct fields: Use **Uppercase** (public)
- JSON output: Control with **json tags**
</details>

---

### Question 5: init() Function
**Q:** What is the `init()` function in Go? When does it run and why do we use it?

<details>
<summary><b>Answer</b></summary>

**What is init()?**

`init()` is a special function in Go that:
- Runs **automatically** before `main()`
- Runs **once** per package
- Cannot be called manually
- Used for setup/initialization

**Execution Order:**
```
1. Package imports
2. Variable declarations
3. init() functions
4. main() function
```

**Visual Flow:**
```
Program Start
    ↓
Step 1: Import packages
    ↓
Step 2: Declare global variables
    var productList []Product
    ↓
Step 3: Run init()
    func init() {
        productList = append(...)
    }
    ↓
Step 4: Run main()
    func main() {
        // productList is already populated!
    }
```

**Example in Our Code:**
```go
var productList []Product  // Declared but empty

func init() {
    // Populate with initial data
    product1 := Product{...}
    productList = append(productList, product1)
    // ... add more products
}

func main() {
    // productList already has 6 products!
    // Ready to use immediately
}
```

**Common Use Cases:**

1. **Database setup**
```go
var db *sql.DB

func init() {
    db = connectToDatabase()
}
```

2. **Configuration loading**
```go
var config Config

func init() {
    config = loadConfig("config.json")
}
```

3. **Initial data population**
```go
var users []User

func init() {
    users = []User{
        {ID: 1, Name: "John"},
        {ID: 2, Name: "Jane"},
    }
}
```

4. **Register handlers**
```go
func init() {
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/api", apiHandler)
}
```

**Multiple init() Functions:**

You can have multiple `init()` in same package:
```go
func init() {
    fmt.Println("First init")
}

func init() {
    fmt.Println("Second init")
}

// Output:
// First init
// Second init
```

They run in the order they appear in the file.

**Why Use init() in Our Project?**

We use it to populate `productList` with initial products before the server starts. This way, when first request comes to `/products`, data is ready!

```
Without init():
Client requests → productList empty → No data to send ❌

With init():
init() populates data → Server starts → Client requests → Send data ✅
```
</details>

---

## Summary

**What We Learned Today:**

1. ✅ **REST API Basics**
   - REST is an architecture
   - Resources, representations, state transfer

2. ✅ **HTTP Protocol**
   - Protocol = set of rules
   - Defines communication between client-server

3. ✅ **HTTP Methods**
   - GET (read), POST (create), PUT (update), PATCH (partial), DELETE (remove)
   - These 5 methods power 99% of web apps!

4. ✅ **HTTP Status Codes**
   - 2xx = Success
   - 4xx = Client error
   - 5xx = Server error

5. ✅ **Building GET Endpoint**
   - Create router with `http.NewServeMux()`
   - Register handler with `mux.HandleFunc()`
   - Handler receives `w` and `r` parameters

6. ✅ **Request and Response**
   - `r *http.Request` = Data from client
   - `w http.ResponseWriter` = Send data to client

7. ✅ **JSON Encoding**
   - Use `json.NewEncoder(w)` to create encoder
   - Call `encoder.Encode(data)` to send JSON
   - Automatically converts Go structs to JSON

8. ✅ **Public vs Private**
   - Uppercase = Public (exportable)
   - Lowercase = Private (package-only)
   - Use JSON tags to control output names

9. ✅ **init() Function**
   - Runs before main()
   - Perfect for initialization
   - Populates initial data

**Key Takeaways:**

```
┌──────────────────────────────────────────────┐
│  Building REST API = 3 Steps                 │
├──────────────────────────────────────────────┤
│  1. Define routes (resources)                │
│  2. Create handler functions                 │
│  3. Send JSON responses                      │
└──────────────────────────────────────────────┘
```

**Remember:**
- You're learning **real backend development**!
- Practice this code multiple times
- Don't memorize - understand the flow
- Each request → handler → response

---

## What's Next?

In the upcoming chapters, we'll build more endpoints:

**Chapter 41: POST Request - Create Products**
- Accept data from client
- Validate input
- Add new products to list
- Return 201 Created status

**Chapter 42: PUT Request - Update Products**
- Find product by ID
- Replace entire product
- Handle not found cases

**Chapter 43: DELETE Request - Remove Products**
- Find and delete products
- Return appropriate status codes

**Chapter 44: Database Integration**
- Connect to PostgreSQL
- CRUD operations with real database
- SQL queries in Go

**Chapter 45: Project Structure**
- Organize code into packages
- Handlers, models, services
- Professional project layout

---

**Final Words:**

Today you built your **first real REST API endpoint**! This is huge! 🎉

You now understand:
- How HTTP works
- How requests/responses flow
- How to build REST APIs
- How JSON encoding works

**Keep practicing!** Build this project multiple times. Change the products, add more fields, experiment!

You're now in the **top 5%** of developers who understand HTTP and REST APIs deeply, not just superficially!

**Stay tuned for the next chapter!** 🚀

---

**Remember:**
> "You don't need to memorize HTTP methods or status codes. As you build more projects, they'll become second nature. Focus on understanding the concepts!" 💡

---
