# Chapter 48: Authentication with JWT - Building Secure APIs

## Table of Contents
- [Introduction](#introduction)
- [Understanding the Problem](#understanding-the-problem)
- [Creating the User Model](#creating-the-user-model)
- [Building User Routes](#building-user-routes)
- [Implementing User Creation](#implementing-user-creation)
- [Implementing User Login](#implementing-user-login)
- [Testing User APIs](#testing-user-apis)
- [Understanding Base64 Encoding](#understanding-base64-encoding)
- [What is Base64?](#what-is-base64)
- [Base64 in Go](#base64-in-go)
- [Why Use Base64?](#why-use-base64)
- [Understanding SHA-256 Hashing](#understanding-sha-256-hashing)
- [What is a Hash Algorithm?](#what-is-a-hash-algorithm)
- [Summary](#summary)
- [Practice Questions](#practice-questions)

---

## Introduction

Welcome to Chapter 48! Today's topic is **JWT (JSON Web Token)** authentication. Now, you've probably heard of JWT before - everyone talks about JSON Web Token, authentication tokens, and all that stuff. But here's the thing: most people think they understand JWT, but they really don't.

I know many senior engineers who watch my classes (about 50% of students are senior engineers, 30% are juniors, and 20% are actual beginners - the ones this course is really for). But even if you're a senior engineer, I'm telling you right now: **you probably don't fully understand JWT**. After today's class, you can test yourself and see if you really understand it.

To understand JWT properly, you need to understand several prerequisite concepts. If you don't understand these concepts, you'll be like many senior engineers who just make things work without truly understanding what's happening under the hood.

You can stop this chapter right here if you want to test yourself. Or you can stay and let me show you how deep this rabbit hole goes.

Let me first check if I'm recording... yes, recording is on. Okay, let's start!

---

## Understanding the Problem

### The Current Situation

Let's run our server:

```bash
go run main.go
```

Server starts on port 4000. Now, let's open Postman.

Currently, we're working with products. We can:
- **GET products** - Retrieve products
- **POST products** - Create products  
- **DELETE products** - Delete products
- **UPDATE products** - Update products

Think about this logically: In an e-commerce site like Daraz (a popular e-commerce platform), products exist. That's normal. Products get created, deleted, and updated. That's also normal.

But here's the problem: **Who creates these products?**

### The Real-World Scenario

When you go to Daraz and see a product - let's say "Diamond Chips" for 350 Taka - someone created that product. That's why you can see it. The person who created that product is a **user** of the system.

That user:
1. **Signed up** to the system (as a shop owner or company owner)
2. **Logged in** to their account
3. **Only then** could they create products

The key insight: **You cannot create products without signing up and logging in.**

Currently, what are we doing? Anyone can create products! Look:

```json
POST http://localhost:4000/api/products
{
  "name": "Gold"
}
```

Product created! Just like that. No authentication, no authorization. Anyone can create products. **That's not how it should work.**

I must be a **signed-in user** of the system to create products. However:
- **Anyone can GET products** (products are public)
- **Only authenticated users can CREATE products**

So let's fix this. First, we need to build a user creation API, then we'll add security.

---

## Creating the User Model

### The User Structure

We need a user in our system. Let's create a new file:

**File: `database/user.go`**

```go
package database

// User represents a user in our system
type User struct {
    ID          int    `json:"id"`
    FirstName   string `json:"first_name"`
    LastName    string `json:"last_name"`
    Email       string `json:"email"`
    Password    string `json:"password"`
    IsShopOwner bool   `json:"is_shop_owner"`
}
```

### Field Explanations

- **ID**: Unique identifier for each user
- **FirstName**: User's first name
- **LastName**: User's last name
- **Email**: User's email address (used for login)
- **Password**: User's password (we'll hash this later)
- **IsShopOwner**: Boolean flag - if true, user can create products; if false, they cannot

### The Database Layer

Just like we have `products` storage, we need `users` storage:

```go
// Global users storage (in-memory for now)
var users []User

// Store saves a new user to the database
func (u *User) Store() User {
    // Check if user already exists (ID != 0 means already created)
    if u.ID != 0 {
        return *u
    }
    
    // Generate new ID
    u.ID = len(users) + 1
    
    // Append to users list
    users = append(users, *u)
    
    // Return the created user
    return *u
}
```

### Understanding the ID Check

Why do we check if `u.ID != 0`?

In Go, when you create a new variable:
- **Strings** default to empty string `""`
- **Integers** default to `0`
- **Booleans** default to `false`
- **Pointers** default to `nil`

So if a user's ID is `0`, we know it's a brand new user that hasn't been created yet. If ID is anything other than `0`, the user already exists in our system.

```
Initial state:  u.ID = 0   (new user, not created)
After creation: u.ID = 1   (user created with ID)
```

### The Store Method Logic

```
┌─────────────────────────────────────┐
│   u.Store() called                  │
└────────────┬────────────────────────┘
             │
             ▼
      ┌──────────────┐
      │ Is u.ID != 0?│
      └──────┬───────┘
             │
        ┌────┴─────┐
        │          │
       Yes        No
        │          │
        │          ▼
        │    ┌──────────────────┐
        │    │ Generate new ID  │
        │    │ u.ID = len + 1   │
        │    └────────┬─────────┘
        │             │
        │             ▼
        │    ┌──────────────────┐
        │    │ Append to users  │
        │    └────────┬─────────┘
        │             │
        └─────────────┘
                      │
                      ▼
              ┌───────────────┐
              │  Return user  │
              └───────────────┘
```

### The Find Method

We also need to find users by email and password (for login):

```go
// Find searches for a user by email and password
// Returns pointer to user if found, nil if not found
func Find(email string, password string) *User {
    // Loop through all users
    for _, u := range users {
        // Check if email AND password match
        if u.Email == email && u.Password == password {
            // Found! Return address of user
            return &u
        }
    }
    
    // Not found, return nil
    return nil
}
```

### Why Return a Pointer?

Notice we return `*User` (pointer to User), not `User` directly. Why?

If we returned `User` directly:
- When NOT found, we'd have to return an empty `User{}` struct
- How would the caller know if we found nothing or found an empty user?

By returning `*User`:
- When found, we return `&u` (address of user)
- When NOT found, we return `nil`
- Caller can easily check: `if user == nil` means not found

```go
// With pointer return (*User):
user := Find("test@example.com", "password123")
if user == nil {
    // Not found - clear and unambiguous
    fmt.Println("User not found")
} else {
    // Found - use user
    fmt.Println("Welcome", user.FirstName)
}

// Without pointer return (User):
user := Find("test@example.com", "password123")
// How do we know if this is "not found" or "found an empty user"?
// Ambiguous!
```

---

## Building User Routes

### Adding User Routes

Now let's add routes for users in our [serve.go](cmd/serve.go):

**File: `cmd/serve.go`**

```go
// Existing product routes...
mux.HandleFunc("POST /api/products", handler.CreateProduct)
mux.HandleFunc("GET /api/products", handler.GetProducts)
// ... other product routes

// New user routes
mux.HandleFunc("POST /api/users", handler.CreateUser)
mux.HandleFunc("POST /api/users/login", handler.Login)
```

### Route Explanation

- `POST /api/users` - Create a new user (signup)
- `POST /api/users/login` - Login existing user

Notice:
- Both are **POST** methods (sending data to server)
- `/api/users` is **plural** (RESTful convention - collection endpoint)
- `/api/users/login` is a specific action on the users collection

---

## Implementing User Creation

### The CreateUser Handler

Create a new file for the user handler:

**File: `handler/create_user.go`**

```go
package handler

import (
    "encoding/json"
    "net/http"
    "yourproject/database"
)

// CreateUser handles user creation (signup)
func CreateUser(w http.ResponseWriter, r *http.Request) {
    // Step 1: Decode JSON request body into User struct
    var newUser database.User
    err := json.NewDecoder(r.Body).Decode(&newUser)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid request data"))
        return
    }
    
    // Step 2: Store the user in database
    createdUser := newUser.Store()
    
    // Step 3: Return created user with 201 status
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated) // 201 Created
    json.NewEncoder(w).Encode(createdUser)
}
```

### Understanding HTTP Status Codes

Why `http.StatusCreated` instead of `http.StatusOK`?

```go
// Instead of manual numbers:
w.WriteHeader(201)  // What does 201 mean?

// Use Go constants:
w.WriteHeader(http.StatusCreated)  // Clear meaning!
```

Common HTTP status codes:
- **200** `http.StatusOK` - Request successful
- **201** `http.StatusCreated` - Resource successfully created
- **400** `http.StatusBadRequest` - Invalid request data
- **401** `http.StatusUnauthorized` - Authentication required
- **404** `http.StatusNotFound` - Resource not found
- **500** `http.StatusInternalServerError` - Server error

Using constants makes your code self-documenting:

```go
// Bad - magic numbers
if statusCode == 400 {
    // What does 400 mean again?
}

// Good - descriptive constants
if statusCode == http.StatusBadRequest {
    // Clear! Bad request.
}
```

---

## Implementing User Login

### The Login Request Structure

For login, we don't need all user fields. We only need email and password:

```go
// LoginRequest represents login credentials
type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}
```

### The Login Handler

**File: `handler/login.go`**

```go
package handler

import (
    "encoding/json"
    "net/http"
    "yourproject/database"
)

// LoginRequest contains login credentials
type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

// Login handles user authentication
func Login(w http.ResponseWriter, r *http.Request) {
    // Step 1: Decode login request
    var loginReq LoginRequest
    err := json.NewDecoder(r.Body).Decode(&loginReq)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid request data"))
        return
    }
    
    // Step 2: Find user by email and password
    user := database.Find(loginReq.Email, loginReq.Password)
    
    // Step 3: Check if user was found
    if user == nil {
        // Not found - invalid credentials
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid credentials"))
        return
    }
    
    // Step 4: User found - return user data
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

### Login Flow Diagram

```
┌─────────────────────────────────────┐
│  POST /api/users/login              │
│  Body: {email, password}            │
└────────────┬────────────────────────┘
             │
             ▼
     ┌───────────────┐
     │ Decode JSON   │
     │ to LoginReq   │
     └───────┬───────┘
             │
             ▼
      ┌──────────────┐
      │ Find user by │
      │ credentials  │
      └──────┬───────┘
             │
             ▼
      ┌──────────────┐
      │ user == nil? │
      └──────┬───────┘
             │
        ┌────┴─────┐
        │          │
       Yes        No
        │          │
        │          ▼
        │    ┌──────────────┐
        │    │ Return user  │
        │    │ data (200)   │
        │    └──────────────┘
        │
        ▼
  ┌─────────────────────┐
  │ Return 400 error    │
  │ "Invalid creds"     │
  └─────────────────────┘
```

---

## Testing User APIs

### Testing User Creation

Let's create a new folder in Postman for user requests:

**Postman Request: Create User**

```
Method: POST
URL: http://localhost:4000/api/users
Body: JSON
```

Request body:

```json
{
    "first_name": "Habib",
    "last_name": "Rahman",
    "email": "habib@gmail.com",
    "password": "12345678",
    "is_shop_owner": true
}
```

**Important**: The JSON field names must match exactly:
- `first_name` (not `firstName` or `firstname`)
- `last_name` (not `lastName` or `lastname`)
- And so on...

If you get an error, check:
1. Field names match exactly
2. JSON syntax is correct
3. Server is running

Use Postman's "Beautify" button to format JSON properly.

### First Attempt (Error)

If you send the request and get:

```
400 Bad Request
"Invalid request data"
```

Check your struct tags! The error might be because:
- You have `is_shop_owner` as `bool` in JSON
- But sent `"true"` as a string instead of boolean `true`

Fix the JSON:

```json
{
    "first_name": "Habib",
    "last_name": "Rahman",
    "email": "habib@gmail.com",
    "password": "12345678",
    "is_shop_owner": true
}
```

Also, **restart your server** after code changes:

```bash
# Stop server (Ctrl+C)
# Start again
go run main.go
```

### Successful Creation

Response:

```json
{
    "id": 1,
    "first_name": "Habib",
    "last_name": "Rahman",
    "email": "habib@gmail.com",
    "password": "12345678",
    "is_shop_owner": true
}
```

Status: **201 Created** - "A new resource was successfully created"

Notice the password is visible in the response. **This is not good practice** - we'll fix this later. In production, never return passwords in API responses!

### Testing Login

**Postman Request: Login**

```
Method: POST
URL: http://localhost:4000/api/users/login
Body: JSON
```

Request body:

```json
{
    "email": "habib@gmail.com",
    "password": "12345678"
}
```

### Successful Login

Response:

```json
{
    "id": 1,
    "first_name": "Habib",
    "last_name": "Rahman",
    "email": "habib@gmail.com",
    "password": "12345678",
    "is_shop_owner": true
}
```

Status: **200 OK** - Login successful!

### Failed Login (Wrong Password)

Change password to wrong value:

```json
{
    "email": "habib@gmail.com",
    "password": "wrong"
}
```

Response:

```
400 Bad Request
"Invalid credentials"
```

This means authentication failed - email or password doesn't match.

---

## Understanding Base64 Encoding

Before we dive into JWT, we need to understand **Base64 encoding**. This is foundational knowledge that many engineers skip, which is why they don't truly understand JWT.

### What is Base64?

Let me ask ChatGPT:

> **What is Base64?**
>
> Base64 is a method for **encoding binary data into ASCII string format**.

Let's break this down:

```
Binary Data  ────[Base64 Encoding]───►  ASCII String
(bytes)                                  (text)
```

Base64 converts:
- Binary data (like image files or bytes)
- Into a set of **64 printable ASCII characters**

### The 64 Characters

Base64 uses exactly 64 characters:

```
A-Z  (26 characters)  A B C D E F ... X Y Z
a-z  (26 characters)  a b c d e f ... x y z
0-9  (10 characters)  0 1 2 3 4 5 6 7 8 9
+    (1 character)    +
/    (1 character)    /
────────────────────
Total: 64 characters
```

Let's verify: 26 + 26 + 10 + 2 = **64 characters**

Any binary data gets converted to a combination of these 64 characters.

### How Base64 Works

Base64 represents data as **groups of 6 bits**:

```
Binary Data (8-bit bytes)
    ↓
Converted to 6-bit groups
    ↓
Each 6-bit group = one of 64 characters
```

Why 64? Because 2^6 = 64 (6 bits can represent 64 different values: 0-63)

---

## Base64 in Go

### Setting Up

Let's write some code to understand Base64:

**File: `main.go`** (comment out server code for testing)

```go
package main

import (
    "encoding/base64"
    "fmt"
)

func main() {
    // Original string
    s := "a"
    
    // Print original
    fmt.Println("Original:", s)
}
```

### Step 1: Convert String to Bytes

Base64 works on **binary data** (bytes). First, convert string to bytes:

```go
func main() {
    // Original string
    s := "a"
    
    // Convert to bytes
    bt := []byte(s)
    
    // Print both
    fmt.Println("String:", s)
    fmt.Println("Bytes:", bt)
}
```

Run it:

```bash
go run main.go
```

Output:

```
String: a
Bytes: [97]
```

Why 97? Because **97 is the ASCII code for 'a'**.

### ASCII Code Reference

Let's verify:

```
Character  →  ASCII Code
'a'        →  97
'b'        →  98
'A'        →  65
'B'        →  66
```

Try different characters:

```go
s := "A"  // Output: [65]
s := "B"  // Output: [66]
s := "ab" // Output: [97 98]
```

Every character has an ASCII code. When you convert a string to bytes, you get the ASCII codes.

### Step 2: Encode to Base64

Now let's encode these bytes to Base64:

```go
package main

import (
    "encoding/base64"
    "fmt"
)

func main() {
    // Original string
    s := "a"
    
    // Convert to bytes
    bt := []byte(s)
    
    // Create Base64 encoder
    enc := base64.URLEncoding.WithPadding(base64.NoPadding)
    
    // Encode to Base64
    b64String := enc.EncodeToString(bt)
    
    fmt.Println("Original:", s)
    fmt.Println("Base64:", b64String)
}
```

### Understanding the Encoder

```go
enc := base64.URLEncoding.WithPadding(base64.NoPadding)
```

- `base64.URLEncoding` - A Base64 encoder object
- `.WithPadding()` - Set padding mode
- `base64.NoPadding` - No padding characters

**What is padding?**

Padding adds extra characters (usually `=`) to make the output a certain length:

```
Without padding: "YWJj"
With padding:    "YWJj=="
```

For example, if your data is 2 bytes but Base64 works in 3-byte groups, it pads with `=` signs:

```
Data:    [97, 98]       (2 bytes)
Padded:  [97, 98, 0, 0] (padded to 4)
Base64:  "YWI="         (with padding)
```

We're using `NoPadding` to keep output clean.

### Running Base64 Encoding

```bash
go run main.go
```

Output:

```
Original: a
Base64: YQ
```

So `"a"` becomes `"YQ"` in Base64!

Try different inputs:

```go
s := "ab"
// Output: "YWI"

s := "abc"
// Output: "YWJj"

s := "Hello World"
// Output: "SGVsbG8gV29ybGQ"
```

### Step 3: Decode from Base64

To reverse the process:

```go
func main() {
    // Original string
    s := "Hello World"
    
    // Convert to bytes
    bt := []byte(s)
    
    // Create encoder
    enc := base64.URLEncoding.WithPadding(base64.NoPadding)
    
    // Encode to Base64
    b64String := enc.EncodeToString(bt)
    fmt.Println("Encoded:", b64String)
    
    // Decode back to bytes
    decodedBytes, err := enc.DecodeString(b64String)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    
    // Convert bytes back to string
    decodedString := string(decodedBytes)
    fmt.Println("Decoded:", decodedString)
}
```

Output:

```
Encoded: SGVsbG8gV29ybGQ
Decoded: Hello World
```

### Complete Base64 Flow

```
┌─────────────────┐
│ "Hello World"   │  Original string
└────────┬────────┘
         │
         │ []byte()
         ▼
┌─────────────────┐
│ [72 101 108...] │  Bytes (ASCII codes)
└────────┬────────┘
         │
         │ enc.EncodeToString()
         ▼
┌─────────────────┐
│ "SGVsbG8gV29..." │  Base64 string
└────────┬────────┘
         │
         │ enc.DecodeString()
         ▼
┌─────────────────┐
│ [72 101 108...] │  Bytes again
└────────┬────────┘
         │
         │ string()
         ▼
┌─────────────────┐
│ "Hello World"   │  Original string back!
└─────────────────┘
```

### Online Base64 Tool

You can verify using online tools. Search "online base64 encoder" and try:

Input: `Hello World`
Output: `SGVsbG8gV29ybGQ=` (with padding)

Input: `a`
Output: `YQ==` (with padding)

Our code produces slightly different output because we use `NoPadding`, but the core encoding is the same.

---

## Why Use Base64?

Now you might ask: **Why do we need Base64?**

### Data Transfer Optimization

When you transfer data from one system to another:

```
┌──────────┐                    ┌──────────┐
│ System A │                    │ System B │
└────┬─────┘                    └─────┬────┘
     │                                 │
     │  ──────────network───────────►  │
     │                                 │
```

If you send **raw data** directly:
- Transfer might be slow
- Some characters might cause problems in HTTP
- Size might be larger than needed

If you **Base64 encode** first:
- Faster transfer (in some cases)
- Safe for HTTP transmission (only printable ASCII)
- Standardized format

### When System A Sends to System B

```
System A                          System B
────────                          ────────
1. Raw data                       
2. Base64 encode ─────┐          
3. Send ──────────────┼──►  4. Receive
                      │     5. Base64 decode
                      └───► 6. Use raw data
```

Base64 ensures data can be safely transmitted as text, even if the original data is binary (like images or files).

### Why It's Faster (Sometimes)

This requires deeper understanding of networking:
- How network protocols work
- What kind of data travels easily
- HTTP header and body encoding
- Character encoding issues

We won't dive into networking details now (that would take us off track), but just know: **Base64 is a standard way to encode binary data as text for safe transmission.**

---

## Understanding SHA-256 Hashing

Now let's talk about another crucial concept: **hashing**.

### What is SHA?

SHA stands for:

```
S - Secure
H - Hash
A - Algorithm
```

**Secure Hash Algorithm**

### SHA Versions

- **SHA-1** (first version, now considered weak)
- **SHA-256** (current standard, very secure)
- **SHA-512** (even more secure, larger output)

### What is a Hash Algorithm?

A hash algorithm:
- Takes **input** (any data, any size)
- Produces **output** (fixed size, looks random)
- Is **one-way** (cannot reverse)
- Is **deterministic** (same input = same output always)

```
┌─────────────────┐
│ Input (any size)│
│ "password123"   │
└────────┬────────┘
         │
         │ [SHA-256]
         ▼
┌─────────────────┐
│ Output (fixed)  │
│ "a665a45920...  │  ← Always 64 hex characters
└─────────────────┘
```

### Key Properties

1. **One-way**: Can't reverse hash to get original
   ```
   "password" → hash → "5e884898..." ✓
   "5e884898..." → unhash → ??? ✗ (impossible!)
   ```

2. **Deterministic**: Same input = same output
   ```
   "hello" → "2cf24dba..."  (always)
   "hello" → "2cf24dba..."  (every time)
   ```

3. **Avalanche effect**: Tiny change = completely different hash
   ```
   "hello"  → "2cf24dba5fb0..."
   "Hello"  → "185f8db32271..."  (completely different!)
   ```

4. **Fixed size**: Any input → always same output length
   ```
   "a"      → 64 characters
   "hello"  → 64 characters
   "hello world hello world hello world" → 64 characters
   ```

### SHA-256 Output Size

SHA-256 always produces:
- **256 bits** of output
- Which is **32 bytes** (256 ÷ 8 = 32)
- Or **64 hexadecimal characters** (each byte = 2 hex digits)

```
SHA-256("password")
= 5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8
  └────────────────── 64 hex characters ──────────────────┘
```

---

## Summary

In this chapter, we covered the foundational concepts needed for JWT:

### What We Built

1. **User Model**
   - User struct with ID, name, email, password, shop owner flag
   - In-memory storage (users slice)
   - Store() method to save users
   - Find() method to search by credentials

2. **User APIs**
   - `POST /api/users` - Create new user (signup)
   - `POST /api/users/login` - Authenticate user (login)
   - Proper HTTP status codes (201, 400, 200)

3. **Testing**
   - Created users via Postman
   - Tested login with correct/incorrect credentials

### Concepts We Learned

1. **Base64 Encoding**
   - Converts binary data to 64 printable ASCII characters
   - Used for safe data transmission
   - Reversible (can encode and decode)
   - Not encryption (anyone can decode!)

2. **SHA-256 Hashing**
   - Secure Hash Algorithm
   - One-way function (cannot reverse)
   - Fixed output size (256 bits / 64 hex chars)
   - Used for password storage

### Why This Matters

Before we implement JWT, you **must** understand:
- How Base64 works (JWT uses it internally)
- How hashing works (for password verification)
- Why we need authentication (security!)

Many developers use JWT without understanding these fundamentals. They just copy-paste code and it works, but they don't know why. **Don't be that developer.**

### Next Steps

In the next chapter, we'll:
1. Hash passwords with SHA-256 (secure storage)
2. Understand JWT structure (header, payload, signature)
3. Generate JWT tokens on login
4. Validate JWT tokens on protected routes
5. Implement proper authentication middleware

But first, make sure you understand **everything** in this chapter. Test yourself:

- Can you explain Base64 to a junior developer?
- Can you implement Base64 encoding from scratch in Go?
- Do you understand why hashing is one-way?
- Can you explain the difference between encoding and hashing?

If you can answer these confidently, you're ready to move forward. If not, **re-read this chapter**. Understanding the fundamentals is more important than rushing ahead.

---

## Practice Questions

### Question 1: Why Pointer Return?

**Question:**
In our `Find()` function, we return `*User` (pointer) instead of `User` directly. Explain why this design is better.

<details>
<summary>Click to see answer</summary>

**Answer:**

Returning a pointer (`*User`) allows us to return `nil` when a user is not found, which provides clear semantics:

```go
// With pointer return (*User):
func Find(email, password string) *User {
    for _, u := range users {
        if u.Email == email && u.Password == password {
            return &u  // Found: return address
        }
    }
    return nil  // Not found: return nil
}

// Calling code is clear:
user := Find("test@test.com", "pass")
if user == nil {
    // Not found - unambiguous!
    return errors.New("invalid credentials")
}
// Use user safely
fmt.Println(user.FirstName)
```

If we returned `User` directly (value):

```go
// Without pointer return (User):
func Find(email, password string) User {
    for _, u := range users {
        if u.Email == email && u.Password == password {
            return u  // Found: return user
        }
    }
    return User{}  // Not found: return empty struct
}

// Calling code is ambiguous:
user := Find("test@test.com", "pass")
// How do we know if this is "not found" or "found empty user"?
if user.Email == "" {  // Checking empty field? Unreliable!
    // What if someone registered with empty email somehow?
}
```

**Key advantages of pointer return:**
1. **Clear nil check**: `if user == nil` is unambiguous
2. **No confusion**: Empty struct vs. "not found" is distinct
3. **Memory efficient**: Don't copy entire struct when not needed
4. **Idiomatic Go**: This pattern is standard in Go for "maybe found" scenarios

**Alternative approaches:**
```go
// Option 1: Return user + boolean
func Find(email, password string) (User, bool) {
    // Return (user, true) if found
    // Return (User{}, false) if not found
}

// Option 2: Return user + error
func Find(email, password string) (User, error) {
    // Return (user, nil) if found
    // Return (User{}, errors.New("not found")) if not found
}

// Option 3: Return pointer (what we chose)
func Find(email, password string) *User {
    // Return &user if found
    // Return nil if not found
}
```

For simple "found/not found" cases, returning a pointer is the cleanest solution.

</details>

---

### Question 2: Base64 vs Encryption

**Question:**
Is Base64 encoding the same as encryption? If someone intercepts Base64-encoded data, can they read it? Explain the difference.

<details>
<summary>Click to see answer</summary>

**Answer:**

**NO**, Base64 is **NOT** encryption! This is a critical distinction:

### Base64 Encoding (Reversible, Public)

```
Original:  "password123"
Encoded:   "cGFzc3dvcmQxMjM="
Decoded:   "password123"  ← Anyone can decode!
```

Base64 is just a **data format conversion**:
- Converts binary/text to ASCII characters
- **Anyone** can decode it (no secret key needed)
- Used for **data transmission**, not security
- Completely reversible

```go
// Encoding (anyone can do this)
encoded := base64.StdEncoding.EncodeToString([]byte("secret"))
// encoded = "c2VjcmV0"

// Decoding (anyone can do this too!)
decoded, _ := base64.StdEncoding.DecodeString("c2VjcmV0")
// decoded = "secret"  ← Original revealed!
```

### Encryption (Secure, Requires Key)

```
Original:   "password123"
Encrypted:  "8f7a9d2b1e4c..." ← looks random, needs key
Decrypted:  "password123"  ← Only with correct key!
```

Encryption is **security protection**:
- Requires a **secret key** to decrypt
- Designed to be **hard to break**
- Used for **confidentiality**
- Cannot decrypt without key

```go
// Encryption (requires secret key)
encrypted := encrypt("secret message", secretKey)
// encrypted = "j8d9f7s8d9f7..." ← meaningless without key

// Decryption (requires same secret key)
decrypted := decrypt(encrypted, secretKey)
// decrypted = "secret message"

// Without key, decryption fails:
decrypt(encrypted, wrongKey)  // ✗ garbage or error
```

### Key Differences

| Aspect | Base64 | Encryption |
|--------|---------|-----------|
| **Purpose** | Data formatting | Security |
| **Reversible?** | Yes, easily | Yes, but only with key |
| **Protects data?** | NO | YES |
| **Key required?** | NO | YES |
| **Anyone can decode?** | YES ⚠️ | NO ✓ |
| **Use case** | Data transmission | Data protection |

### Real-World Example

**Scenario 1: Email attachment (Base64)**
```
┌─────────────┐
│ Image file  │
└──────┬──────┘
       │ Base64 encode
       ▼
┌─────────────┐
│"iVBORw0KG..." │ ← Sent via email
└──────┬──────┘
       │ Base64 decode (anyone can do this!)
       ▼
┌─────────────┐
│ Image file  │ ← Original recovered
└─────────────┘
```

Email requires text, so we encode binary images as Base64. **Not secure** - anyone reading the email can decode it!

**Scenario 2: Encrypted password (Encryption)**
```
┌─────────────┐
│ "password"  │
└──────┬──────┘
       │ Encrypt with key
       ▼
┌─────────────┐
│"8f7a9d2b..." │ ← Stored in database
└──────┬──────┘
       │ Decrypt with key (only server has key!)
       ▼
┌─────────────┐
│ "password"  │ ← Only recoverable with key
└─────────────┘
```

Even if attacker steals database, they **cannot** decrypt passwords without the key.

### Important Security Note

**NEVER** use Base64 for security:

```go
// ✗ BAD - This provides NO security!
password := "secret123"
encoded := base64.StdEncoding.EncodeToString([]byte(password))
// Store encoded in database
// ✗ Anyone with database access can decode instantly!

// ✓ GOOD - Use hashing (one-way) for passwords
password := "secret123"
hashed := sha256.Sum256([]byte(password))
// Store hashed in database
// ✓ Cannot reverse hash to get original password
```

### Summary

- **Base64**: Data format (like converting Word doc to PDF)
- **Encryption**: Security lock (like putting doc in safe)
- **Hashing**: One-way transformation (like shredding doc - can't reconstruct)

If you see Base64 and think "this is secure" - **WRONG!** Base64 provides **zero security**. It's just a different way to represent data.

</details>

---

### Question 3: SHA-256 Avalanche Effect

**Question:**
What is the "avalanche effect" in hash functions? Demonstrate with SHA-256 by showing how changing one character in the input drastically changes the hash output. Why is this property important for security?

<details>
<summary>Click to see answer</summary>

**Answer:**

The **avalanche effect** means that a tiny change in input causes a **massive change** in output. Let's demonstrate:

### Demonstration

```go
package main

import (
    "crypto/sha256"
    "fmt"
)

func main() {
    // Test 1: "hello"
    input1 := "hello"
    hash1 := sha256.Sum256([]byte(input1))
    fmt.Printf("Input: %s\n", input1)
    fmt.Printf("Hash:  %x\n\n", hash1)
    
    // Test 2: "Hello" (capital H)
    input2 := "Hello"
    hash2 := sha256.Sum256([]byte(input2))
    fmt.Printf("Input: %s\n", input2)
    fmt.Printf("Hash:  %x\n\n", hash2)
    
    // Test 3: "hello " (space added)
    input3 := "hello "
    hash3 := sha256.Sum256([]byte(input3))
    fmt.Printf("Input: %s\n", input3)
    fmt.Printf("Hash:  %x\n\n", hash3)
}
```

### Output

```
Input: hello
Hash:  2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

Input: Hello
Hash:  185f8db32271fe25f561a6fc938b2e264306ec304eda518007d1764826381969

Input: hello 
Hash:  d98cf3cb47cc6c4c6c4c6c4c6c4c6c4c6c4c6c4c6c4c6c4c6c4c6c4c6c4c6c4
```

**Analysis:**
- Original: `hello`
- Changed: ONE character (`h` → `H`)
- Result: **Completely different hash**

Not just a few characters different - **EVERY CHARACTER** is different!

### Why Avalanche Effect Matters

**1. Prevents Pattern Detection**

Without avalanche effect:
```
"password1" → "abc123..."
"password2" → "abc124..."  ← Similar! (bad)
"password3" → "abc125..."  ← Pattern visible! (very bad)
```

An attacker could see the pattern and guess:
"Oh, these hashes are similar, so passwords must be similar!"

With avalanche effect:
```
"password1" → "5e884898..."
"password2" → "6b3a55e0..."  ← Completely different (good)
"password3" → "8d969eef..."  ← No pattern (very good)
```

No pattern to exploit!

**2. Prevents Rainbow Table Attacks**

Rainbow tables are precomputed hashes:
```
Password    → Hash
"password"  → "5e884898..."
"12345678"  → "8d969eef..."
"qwerty"    → "65e84be3..."
```

If similar passwords had similar hashes, attackers could:
- Find one password's hash
- Search nearby hash values
- Find similar passwords quickly

With avalanche effect, they must:
- Check EVERY possible password
- No shortcuts available

**3. Data Integrity Verification**

Used to detect file tampering:
```
Original file: "Contract: Pay $100"
Hash: "a8f5b2..."

Tampered file: "Contract: Pay $1000"  ← one digit changed
Hash: "3d7e9c..."  ← Completely different!
```

Even changing one bit causes completely different hash, so tampering is immediately detected.

### Mathematical Property

In good hash functions like SHA-256:
- Changing 1 bit of input
- Changes approximately **50%** of output bits
- This is the **avalanche criterion**

```
Input bit change:     1 bit
Output bits changed:  ~128 bits (out of 256)
Percentage:           50%
```

### Real Code Example

```go
package main

import (
    "crypto/sha256"
    "fmt"
)

func hashPassword(password string) string {
    hash := sha256.Sum256([]byte(password))
    return fmt.Sprintf("%x", hash)
}

func main() {
    // Similar passwords
    passwords := []string{
        "MyPassword123",
        "MyPassword124",  // Last digit changed
        "myPassword123",  // First letter lowercase
        "MyPassword123 ", // Space added
    }
    
    fmt.Println("Avalanche Effect Demonstration:")
    fmt.Println("================================\n")
    
    for i, pwd := range passwords {
        hash := hashPassword(pwd)
        fmt.Printf("%d. Password: %s\n", i+1, pwd)
        fmt.Printf("   Hash:     %s\n\n", hash)
    }
    
    // Show that completely different passwords
    // produce no more different hashes than slightly different ones
    fmt.Println("\nConclusion:")
    fmt.Println("Tiny changes produce completely different hashes!")
    fmt.Println("This prevents attackers from guessing patterns.")
}
```

### Why This Defeats Attacks

**Without avalanche effect (bad):**
```
Attacker thinks: "If I find hash 'abc123', 
maybe 'abc124' is similar password!"
```

**With avalanche effect (good):**
```
Attacker thinks: "This hash tells me NOTHING 
about what the password might be like!"
```

### Summary

The avalanche effect ensures:
1. **No patterns** in hash outputs
2. **Cannot guess** similar inputs from similar outputs
3. **Must brute force** every possibility
4. **Detects tampering** even from 1-bit changes

This is why SHA-256 is secure for password hashing and data integrity. Even the smallest change completely scrambles the output!

</details>

---

### Question 4: Implementing GetUser Endpoint

**Question:**
Implement a `GET /api/users/:id` endpoint that retrieves a single user by ID. Include proper error handling for "user not found" cases.

<details>
<summary>Click to see answer</summary>

**Answer:**

Here's a complete implementation:

### Step 1: Add GetByID Method to Database

**File: `database/user.go`**

```go
// GetByID finds a user by their ID
// Returns pointer to user if found, nil if not found
func GetByID(id int) *User {
    for _, u := range users {
        if u.ID == id {
            return &u
        }
    }
    return nil
}
```

### Step 2: Create GetUser Handler

**File: `handler/get_user.go`**

```go
package handler

import (
    "encoding/json"
    "net/http"
    "strconv"
    "yourproject/database"
)

// GetUser retrieves a single user by ID
func GetUser(w http.ResponseWriter, r *http.Request) {
    // Step 1: Extract ID from URL path parameter
    // Path looks like: /api/users/5
    idStr := r.PathValue("id")
    
    // Step 2: Convert string ID to integer
    id, err := strconv.Atoi(idStr)
    if err != nil {
        // Invalid ID format (not a number)
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid user ID"))
        return
    }
    
    // Step 3: Find user in database
    user := database.GetByID(id)
    
    // Step 4: Check if user was found
    if user == nil {
        // User not found
        w.WriteHeader(http.StatusNotFound)
        w.Write([]byte("User not found"))
        return
    }
    
    // Step 5: User found - return user data
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

### Step 3: Add Route

**File: `cmd/serve.go`**

```go
// User routes
mux.HandleFunc("POST /api/users", handler.CreateUser)
mux.HandleFunc("POST /api/users/login", handler.Login)
mux.HandleFunc("GET /api/users/{id}", handler.GetUser)  // New route!
```

### Understanding Path Parameters

The route pattern `GET /api/users/{id}` means:
- `{id}` is a **path parameter**
- Matches any value in that position
- Access with `r.PathValue("id")`

Examples of matching URLs:
```
GET /api/users/1     → id = "1"
GET /api/users/42    → id = "42"
GET /api/users/999   → id = "999"
GET /api/users/abc   → id = "abc" (will fail conversion)
```

### Error Handling

We handle three types of errors:

**1. Invalid ID Format (400 Bad Request)**
```
GET /api/users/abc
→ 400 Bad Request: "Invalid user ID"
```

The ID must be a number. If it's not, `strconv.Atoi()` returns an error.

**2. User Not Found (404 Not Found)**
```
GET /api/users/999
→ 404 Not Found: "User not found"
```

The ID is valid, but no user exists with that ID.

**3. Success (200 OK)**
```
GET /api/users/1
→ 200 OK: {user JSON}
```

User exists, return their data.

### Testing with Postman

**Test 1: Valid user**
```
GET http://localhost:4000/api/users/1
```

Response:
```json
{
    "id": 1,
    "first_name": "Habib",
    "last_name": "Rahman",
    "email": "habib@gmail.com",
    "password": "12345678",
    "is_shop_owner": true
}
```

**Test 2: User not found**
```
GET http://localhost:4000/api/users/999
```

Response:
```
404 Not Found
"User not found"
```

**Test 3: Invalid ID**
```
GET http://localhost:4000/api/users/abc
```

Response:
```
400 Bad Request
"Invalid user ID"
```

### Improved Version with JSON Errors

For better API design, return JSON error messages:

```go
func GetUser(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    
    // Extract and convert ID
    idStr := r.PathValue("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{
            "error": "Invalid user ID",
        })
        return
    }
    
    // Find user
    user := database.GetByID(id)
    if user == nil {
        w.WriteHeader(http.StatusNotFound)
        json.NewEncoder(w).Encode(map[string]string{
            "error": "User not found",
        })
        return
    }
    
    // Success
    json.NewEncoder(w).Encode(user)
}
```

Now errors are also JSON:
```json
{
    "error": "User not found"
}
```

This is better for frontend applications that expect JSON responses.

</details>

---

### Question 5: Security Problem Analysis

**Question:**
Look at our current implementation. We're returning the user's password in login responses and storing passwords as plain text. Explain why this is dangerous and outline the steps needed to fix it (we'll implement the fix in the next chapter).

<details>
<summary>Click to see answer</summary>

**Answer:**

Our current implementation has **CRITICAL SECURITY VULNERABILITIES**:

### Vulnerability 1: Plaintext Password Storage

**Current code:**
```go
type User struct {
    ID          int    `json:"id"`
    FirstName   string `json:"first_name"`
    LastName    string `json:"last_name"`
    Email       string `json:"email"`
    Password    string `json:"password"`  // ← Stored as plain text!
    IsShopOwner bool   `json:"is_shop_owner"`
}

// In database/user.go
users = append(users, User{
    ID:       1,
    Email:    "habib@gmail.com",
    Password: "12345678",  // ← Plain text in memory!
})
```

**Why this is dangerous:**

```
┌─────────────────────────────────────┐
│ Database (or Memory)                │
├─────────────────────────────────────┤
│ ID: 1                               │
│ Email: habib@gmail.com              │
│ Password: "12345678"  ← VISIBLE!    │
└─────────────────────────────────────┘
```

**Attack scenarios:**

1. **Data breach**: If someone hacks your database:
   ```
   Attacker sees: "12345678"
   Attacker knows: User's password!
   Attacker can: Log in as that user!
   ```

2. **Insider threat**: Any developer with database access:
   ```
   Developer runs: SELECT * FROM users;
   Developer sees: All passwords in plain text
   Developer can: Log into any account
   ```

3. **Log files**: Passwords appear in logs:
   ```
   [DEBUG] User login: email=test@test.com password=12345678
   ↑ Password leaked in log file!
   ```

4. **Password reuse**: Users often reuse passwords:
   ```
   User's password: "12345678"
   Same password on:
   - Gmail
   - Facebook
   - Bank account
   → Hacker now has access to ALL accounts!
   ```

### Vulnerability 2: Password in API Response

**Current code:**
```go
// login.go
func Login(w http.ResponseWriter, r *http.Request) {
    user := database.Find(loginReq.Email, loginReq.Password)
    
    // Return entire user including password!
    json.NewEncoder(w).Encode(user)
}
```

**Response:**
```json
{
    "id": 1,
    "first_name": "Habib",
    "last_name": "Rahman",
    "email": "habib@gmail.com",
    "password": "12345678",  ← WHY IS THIS HERE?!
    "is_shop_owner": true
}
```

**Why this is dangerous:**

1. **Network sniffing**: Password travels over network:
   ```
   Client ←─── {"password": "12345678"} ───→ Server
              ↑ Anyone monitoring network sees this!
   ```

2. **Browser console**: Frontend developers can see it:
   ```javascript
   fetch('/api/users/login')
     .then(res => res.json())
     .then(user => {
       console.log(user.password);  // ← Visible in dev tools!
     });
   ```

3. **Client-side code**: Password stored in JavaScript:
   ```javascript
   const user = { 
     id: 1, 
     password: "12345678"  // ← Now in client memory
   };
   localStorage.setItem('user', JSON.stringify(user));
   // ← Password saved in browser storage!
   ```

4. **Logs and monitoring**: API monitoring tools log responses:
   ```
   Response body: {"id": 1, "password": "12345678", ...}
   ↑ Password logged by monitoring service!
   ```

### How Bad Is This?

**Severity: CRITICAL** 🔥🔥🔥

Industry standards (OWASP Top 10):
- Plaintext password storage: **#2 Critical vulnerability**
- Exposing sensitive data: **#3 Critical vulnerability**

**Real-world consequences:**
- Company lawsuits (GDPR violations)
- User account takeovers
- Data breach headlines
- Loss of customer trust
- Potential jail time for negligence

### The Fix (Overview)

We need to implement **password hashing**:

**Step 1: Hash passwords before storage**

```go
// When user signs up:
plainPassword := "12345678"
hashedPassword := hashPassword(plainPassword)
// hashedPassword = "ef92b778bafe771e89245b89ecbc08a44a4e166c06659911881f383d4473e94f"

user := User{
    Email:    "habib@gmail.com",
    Password: hashedPassword,  // ← Store hash, not plain text
}
```

**Step 2: Compare hashes during login**

```go
// When user logs in:
inputPassword := "12345678"  // From login request
hashedInput := hashPassword(inputPassword)

// Compare hashes:
if user.Password == hashedInput {
    // Passwords match!
}
```

**Step 3: Never return password in API**

```go
// Option A: Use json:"-" tag
type User struct {
    Password string `json:"-"`  // Never included in JSON
}

// Option B: Create separate struct for responses
type UserResponse struct {
    ID          int    `json:"id"`
    FirstName   string `json:"first_name"`
    // No password field!
}
```

### Proper Implementation (Next Chapter)

```go
import "crypto/sha256"

// Hash password using SHA-256
func hashPassword(password string) string {
    hash := sha256.Sum256([]byte(password))
    return fmt.Sprintf("%x", hash)
}

// Create user with hashed password
func CreateUser(w http.ResponseWriter, r *http.Request) {
    var newUser User
    json.NewDecoder(r.Body).Decode(&newUser)
    
    // Hash password before storing!
    newUser.Password = hashPassword(newUser.Password)
    
    createdUser := newUser.Store()
    
    // Remove password from response
    createdUser.Password = ""  // Clear it
    json.NewEncoder(w).Encode(createdUser)
}

// Login with hashed password comparison
func Login(w http.ResponseWriter, r *http.Request) {
    var loginReq LoginRequest
    json.NewDecoder(r.Body).Decode(&loginReq)
    
    // Hash the input password
    hashedInput := hashPassword(loginReq.Password)
    
    // Find user with hashed password
    user := database.FindByEmailAndHash(loginReq.Email, hashedInput)
    
    if user == nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid credentials"))
        return
    }
    
    // Remove password from response
    user.Password = ""
    json.NewEncoder(w).Encode(user)
}
```

### Security After Fix

**Database:**
```
ID: 1
Email: habib@gmail.com
Password: "ef92b778bafe771e89245b89ecbc08a44a4e166c06659911881f383d4473e94f"
         ↑ Hash - cannot reverse to get original!
```

**API Response:**
```json
{
    "id": 1,
    "first_name": "Habib",
    "email": "habib@gmail.com"
}
```
No password field at all!

**If database is breached:**
```
Attacker sees: "ef92b778bafe..."
Attacker tries: Cannot reverse hash
Attacker fails: Cannot log in!
```

### Summary

**Current Problems:**
1. ❌ Passwords stored as plain text
2. ❌ Passwords returned in API responses
3. ❌ Anyone with database access sees all passwords
4. ❌ Network monitoring reveals passwords
5. ❌ If one system is breached, all user accounts compromised

**Solutions (Next Chapter):**
1. ✅ Hash passwords with SHA-256
2. ✅ Never return passwords in responses
3. ✅ Use json:"-" tags or separate response structs
4. ✅ Compare password hashes, not plain text
5. ✅ Even if database breached, passwords are safe

**Remember:** Security is not optional. It's not something you add "later." It's foundational. Fix security issues **immediately**, not after shipping to production!

</details>

---

**Next Chapter Preview:**

In Chapter 49, we'll implement:
1. SHA-256 password hashing
2. Secure password comparison
3. JWT token generation
4. JWT token validation
5. Protected routes with authentication middleware

Make sure you understand everything in this chapter before moving forward! 🚀
