# Chapter 49: Authentication Middleware - Protecting Your APIs

## Table of Contents
- [Introduction](#introduction)
- [Understanding the Authentication Flow](#understanding-the-authentication-flow)
- [Returning JWT on Login](#returning-jwt-on-login)
- [Sending JWT in Requests](#sending-jwt-in-requests)
- [Parsing JWT from Headers](#parsing-jwt-from-headers)
- [Validating JWT Signature](#validating-jwt-signature)
- [Testing JWT Validation](#testing-jwt-validation)
- [Creating Authentication Middleware](#creating-authentication-middleware)
- [Protecting Routes with Middleware](#protecting-routes-with-middleware)
- [The JWT Passport Analogy](#the-jwt-passport-analogy)
- [Current Project Problems](#current-project-problems)
- [Summary](#summary)
- [Practice Questions](#practice-questions)

---

## Introduction

In the last chapter, we learned how to create users and generate JWT tokens. But we didn't actually **use** those tokens to protect our APIs. Currently, anyone can create products without authentication!

Today, we'll implement **authentication middleware** that:
1. Checks if JWT token is present in request headers
2. Validates the JWT signature
3. Blocks unauthorized requests
4. Allows authenticated requests to proceed

This is how real production APIs work - you must prove your identity before performing sensitive operations.

---

## Understanding the Authentication Flow

Let's visualize the complete authentication system:

```
┌─────────────┐                          ┌─────────────┐
│  Frontend   │                          │  Backend    │
└──────┬──────┘                          └──────┬──────┘
       │                                        │
       │  1. POST /api/users/login             │
       │     {email, password}                  │
       ├───────────────────────────────────────►│
       │                                        │
       │                                        │  2. Check DB
       │                                        ├─────────►┌──────┐
       │                                        │          │  DB  │
       │                                        │◄─────────┤      │
       │                                        │          └──────┘
       │                                        │
       │                                        │  3. Generate JWT
       │  4. Return JWT                         │     (if valid)
       │◄───────────────────────────────────────┤
       │                                        │
       │  5. Store JWT in                       │
       │     localStorage/memory                │
       │                                        │
       │  6. POST /api/products                 │
       │     Header: Authorization: Bearer JWT  │
       ├───────────────────────────────────────►│
       │                                        │
       │                                        │  7. Verify JWT
       │                                        │     signature
       │                                        │
       │  8. Return 401 if invalid              │
       │     OR 200 with product if valid       │
       │◄───────────────────────────────────────┤
       │                                        │
```

### The Process

**Step 1-4: Login and receive JWT**
- User logs in with email/password
- Backend validates credentials
- Backend generates JWT token
- Frontend receives and stores JWT

**Step 5-8: Use JWT for protected operations**
- Frontend sends JWT in Authorization header
- Backend validates JWT signature
- If valid: allow operation
- If invalid: return 401 Unauthorized

---

## Returning JWT on Login

Currently, our login handler returns the entire user object. Let's change it to return a JWT token instead.

### Update Login Handler

**File: `handler/login.go`**

```go
package handler

import (
    "encoding/json"
    "net/http"
    "yourproject/config"
    "yourproject/database"
    "yourproject/utils"
)

func Login(w http.ResponseWriter, r *http.Request) {
    var loginReq LoginRequest
    err := json.NewDecoder(r.Body).Decode(&loginReq)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid request data"))
        return
    }
    
    // Find user by credentials
    user := database.Find(loginReq.Email, loginReq.Password)
    
    if user == nil {
        w.WriteHeader(http.StatusBadRequest)
        w.Write([]byte("Invalid credentials"))
        return
    }
    
    // Generate JWT token
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
    
    // Return JWT token
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "access_token": accessToken,
    })
}
```

### Add JWT Secret to Config

**File: `config/config.go`**

```go
type Config struct {
    Version     string
    ServiceName string
    HTTPPort    int64
    JWTSecret   string  // Add this field
}

func LoadConfig() {
    // ... existing code ...
    
    jwtSecret := os.Getenv("JWT_SECRET")
    if jwtSecret == "" {
        fmt.Println("JWT_SECRET is required")
        os.Exit(1)
    }
    
    cfg = &Config{
        Version:     version,
        ServiceName: serviceName,
        HTTPPort:    port,
        JWTSecret:   jwtSecret,
    }
}
```

### Update .env File

**File: `.env`**

```bash
VERSION=1.0.0
SERVICE_NAME=ecommerce-api
HTTP_PORT=4000
JWT_SECRET=my_super_secret_key_12345_very_secure
```

**Important:** Use a strong, random secret in production!

### Test Login

Restart server and test login:

**Request:**
```
POST http://localhost:4000/api/users/login
{
  "email": "habib@gmail.com",
  "password": "12345678"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjEsImZpcnN0X25hbWUiOiJIYWJpYiIsImxhc3RfbmFtZSI6IlJhaG1hbiIsImVtYWlsIjoiaGFiaWJAZ21haWwuY29tIn0.kF7x2B..."
}
```

Perfect! Now the login returns a JWT token.

---

## Sending JWT in Requests

When making requests to protected endpoints, the frontend must send the JWT token in the **Authorization header**.

### Authorization Header Format

```
Authorization: Bearer <token>
```

- **Authorization**: Header name (standard)
- **Bearer**: Token type (standard for JWT)
- **<token>**: The actual JWT string

### Example in Postman

**Headers tab:**
```
Key: Authorization
Value: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Important:** There's a **space** between "Bearer" and the token!

```
Correct:   Bearer eyJhbGci...
Wrong:     BearereyJhbGci...  (no space)
Wrong:     bearer eyJhbGci... (lowercase)
```

### Setting Up in Postman

1. Create environment variable `JWT_TOKEN`
2. After login, save token to variable
3. Use `{{JWT_TOKEN}}` in Authorization header

**Postman environment:**
```
Variable: JWT_TOKEN
Value: (paste token here)
```

**Postman header:**
```
Authorization: Bearer {{JWT_TOKEN}}
```

---

## Parsing JWT from Headers

Now let's implement JWT validation in our product creation handler.

### Step 1: Extract Authorization Header

**File: `handler/create_product.go`**

```go
func CreateProduct(w http.ResponseWriter, r *http.Request) {
    // Step 1: Get Authorization header
    header := r.Header.Get("Authorization")
    
    // Step 2: Check if header exists
    if header == "" {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }
    
    // Step 3: Split "Bearer" and token
    headerParts := strings.Split(header, " ")
    
    // Step 4: Verify format (must be 2 parts)
    if len(headerParts) != 2 {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }
    
    // Step 5: Extract token (second part)
    accessToken := headerParts[1]
    
    // Now we have the token, let's validate it...
}
```

### Understanding the Split

When we split `"Bearer eyJhbGci..."` by space:

```go
header = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

headerParts = strings.Split(header, " ")
// Result: ["Bearer", "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."]

headerParts[0] // "Bearer"
headerParts[1] // "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

We want `headerParts[1]` (the actual token).

---

## Validating JWT Signature

Now comes the critical part: **verifying the JWT signature**.

### JWT Structure Review

A JWT has three parts separated by dots:

```
eyJhbGci...  .  eyJzdWIi...  .  kF7x2B...
   ↑              ↑              ↑
 Header         Payload       Signature
```

### Validation Algorithm

```
1. Split JWT by dots → get header, payload, signature
2. Concatenate header + "." + payload → message
3. Hash message with HMAC-SHA256 + secret → new signature
4. Compare new signature with original signature
5. If match: valid ✓
   If not match: invalid ✗
```

### Implementation

```go
func CreateProduct(w http.ResponseWriter, r *http.Request) {
    // ... (previous code to get accessToken)
    
    // Step 6: Split token into three parts
    tokenParts := strings.Split(accessToken, ".")
    
    if len(tokenParts) != 3 {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }
    
    // Step 7: Extract three parts
    jwtHeader := tokenParts[0]
    jwtPayload := tokenParts[1]
    jwtSignature := tokenParts[2]
    
    // Step 8: Create message (header + "." + payload)
    message := jwtHeader + "." + jwtPayload
    
    // Step 9: Get secret from config
    cfg := config.GetConfig()
    secretKey := []byte(cfg.JWTSecret)
    
    // Step 10: Generate new signature using HMAC-SHA256
    h := hmac.New(sha256.New, secretKey)
    h.Write([]byte(message))
    hash := h.Sum(nil)
    
    // Step 11: Encode hash to Base64
    enc := base64.URLEncoding.WithPadding(base64.NoPadding)
    newSignature := enc.EncodeToString(hash)
    
    // Step 12: Compare signatures
    if newSignature != jwtSignature {
        http.Error(w, "Unauthorized - you hacker!", http.StatusUnauthorized)
        return
    }
    
    // Step 13: Signature valid - proceed with product creation
    // ... (existing product creation code)
}
```

### Required Imports

```go
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
    "strings"
    // ... other imports
)
```

### Signature Validation Flow

```
┌────────────────────────────────────────┐
│ JWT from client:                       │
│ header.payload.signature               │
└──────────┬─────────────────────────────┘
           │
           ▼
    ┌─────────────┐
    │ Split by "."│
    └──────┬──────┘
           │
           ├──────────┬──────────┬
           ▼          ▼          ▼
       header      payload   signature
           │          │          │
           └────┬─────┘          │
                │                │
                ▼                │
         message = header.payload│
                │                │
                ▼                │
    ┌──────────────────────┐    │
    │ HMAC-SHA256          │    │
    │ (message + secret)   │    │
    └──────────┬───────────┘    │
               │                │
               ▼                │
         newSignature           │
               │                │
               └────────┬───────┘
                        │
                        ▼
                ┌───────────────┐
                │ newSignature  │
                │ == signature? │
                └───────┬───────┘
                        │
              ┌─────────┴─────────┐
              │                   │
             Yes                 No
              │                   │
              ▼                   ▼
        ┌──────────┐        ┌──────────┐
        │ Allowed  │        │ 401 Error│
        └──────────┘        └──────────┘
```

---

## Testing JWT Validation

### Test 1: Without Token

**Request:**
```
POST http://localhost:4000/api/products
Headers: (no Authorization header)
Body: {"name": "Test Product"}
```

**Response:**
```
401 Unauthorized
```

✓ Correctly blocked!

### Test 2: With Valid Token

**Request:**
```
POST http://localhost:4000/api/products
Headers: Authorization: Bearer <valid_token>
Body: {"name": "Test Product"}
```

**Response:**
```json
{
  "id": 1,
  "name": "Test Product"
}
```

✓ Successfully created!

### Test 3: Tampered Token

Try changing one character in the payload:

**Original JWT:**
```
eyJhbGci...  .  eyJzdWIiOjJ9  .  kF7x2B...
                     ↑
                  sub: 2
```

**Tampered JWT:**
```
eyJhbGci...  .  eyJzdWIiOjN9  .  kF7x2B...
                     ↑
                  sub: 3 (changed from 2!)
```

**Request:**
```
POST http://localhost:4000/api/products
Headers: Authorization: Bearer <tampered_token>
```

**Response:**
```
401 Unauthorized - you hacker!
```

✓ Attack blocked! The signature validation caught the tampering.

---

## Creating Authentication Middleware

Repeating validation code in every handler is tedious. Let's create **middleware** to handle authentication once.

### Create Authentication Middleware

**File: `middleware/authenticate_jwt.go`**

```go
package middleware

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/base64"
    "net/http"
    "strings"
    "yourproject/config"
)

// AuthenticateJWT validates JWT tokens in Authorization header
func AuthenticateJWT(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Step 1: Get Authorization header
        header := r.Header.Get("Authorization")
        
        if header == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        // Step 2: Split Bearer and token
        headerParts := strings.Split(header, " ")
        
        if len(headerParts) != 2 {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        // Step 3: Extract token
        accessToken := headerParts[1]
        
        // Step 4: Split token into parts
        tokenParts := strings.Split(accessToken, ".")
        
        if len(tokenParts) != 3 {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        // Step 5: Extract header, payload, signature
        jwtHeader := tokenParts[0]
        jwtPayload := tokenParts[1]
        jwtSignature := tokenParts[2]
        
        // Step 6: Create message
        message := jwtHeader + "." + jwtPayload
        
        // Step 7: Get secret from config
        cfg := config.GetConfig()
        secretKey := []byte(cfg.JWTSecret)
        
        // Step 8: Generate new signature
        h := hmac.New(sha256.New, secretKey)
        h.Write([]byte(message))
        hash := h.Sum(nil)
        
        // Step 9: Base64 encode
        enc := base64.URLEncoding.WithPadding(base64.NoPadding)
        newSignature := enc.EncodeToString(hash)
        
        // Step 10: Compare signatures
        if newSignature != jwtSignature {
            http.Error(w, "Unauthorized - you hacker!", http.StatusUnauthorized)
            return
        }
        
        // Step 11: Valid token - call next handler
        next.ServeHTTP(w, r)
    })
}
```

### Clean Up Product Handler

Now remove all JWT validation code from `create_product.go`:

```go
func CreateProduct(w http.ResponseWriter, r *http.Request) {
    // No more JWT validation code here!
    // Middleware handles it before this runs
    
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

Much cleaner! The handler only focuses on creating products.

---

## Protecting Routes with Middleware

Now let's apply authentication middleware to routes that need protection.

### Update Route Registration

**File: `cmd/serve.go`**

```go
func Start() {
    mux := http.NewServeMux()
    
    // Public routes (no authentication needed)
    mux.HandleFunc("GET /api/products", handler.GetProducts)
    mux.HandleFunc("GET /api/products/{id}", handler.GetProduct)
    mux.HandleFunc("POST /api/users", handler.CreateUser)
    mux.HandleFunc("POST /api/users/login", handler.Login)
    
    // Protected routes (authentication required)
    mux.HandleFunc("POST /api/products", handler.CreateProduct)
    mux.HandleFunc("PUT /api/products/{id}", handler.UpdateProduct)
    mux.HandleFunc("DELETE /api/products/{id}", handler.DeleteProduct)
    
    // Apply middleware
    handler := middleware.Logger(mux)
    handler = middleware.CorsWithPreflight(handler)
    handler = middleware.AuthenticateJWT(handler) // JWT middleware
    
    // Start server
    cfg := config.GetConfig()
    addr := fmt.Sprintf(":%d", cfg.HTTPPort)
    fmt.Printf("Server running on port %d\n", cfg.HTTPPort)
    http.ListenAndServe(addr, handler)
}
```

Wait, there's a problem! This applies JWT middleware to **all routes**, including public ones like GET products and login!

### The Middleware Order Problem

Current flow:
```
Request → Logger → CORS → AuthenticateJWT → Route Handler
                              ↑
                              │
                    Blocks login and public routes!
```

We need **selective middleware** - some routes need auth, others don't.

### Solution: Route-Level Middleware

Instead of global middleware, apply authentication per route:

```go
func Start() {
    mux := http.NewServeMux()
    
    // Public routes
    mux.HandleFunc("GET /api/products", handler.GetProducts)
    mux.HandleFunc("GET /api/products/{id}", handler.GetProduct)
    mux.HandleFunc("POST /api/users", handler.CreateUser)
    mux.HandleFunc("POST /api/users/login", handler.Login)
    
    // Protected routes - wrap with middleware
    mux.Handle("POST /api/products",
        middleware.AuthenticateJWT(
            http.HandlerFunc(handler.CreateProduct),
        ),
    )
    
    mux.Handle("PUT /api/products/{id}",
        middleware.AuthenticateJWT(
            http.HandlerFunc(handler.UpdateProduct),
        ),
    )
    
    mux.Handle("DELETE /api/products/{id}",
        middleware.AuthenticateJWT(
            http.HandlerFunc(handler.DeleteProduct),
        ),
    )
    
    // Apply global middleware (logger, CORS)
    handler := middleware.Logger(mux)
    handler = middleware.CorsWithPreflight(handler)
    
    cfg := config.GetConfig()
    addr := fmt.Sprintf(":%d", cfg.HTTPPort)
    fmt.Printf("Server running on port %d\n", cfg.HTTPPort)
    http.ListenAndServe(addr, handler)
}
```

### Execution Flow

**Public route (GET /api/products):**
```
Request → Logger → CORS → GetProducts → Response
```

**Protected route (POST /api/products):**
```
Request → Logger → CORS → AuthenticateJWT → CreateProduct → Response
                              ↑
                              │
                         Validates JWT
```

Perfect! Now authentication only applies to routes that need it.

---

## The JWT Passport Analogy

Think of JWT like a **passport**:

### Your Passport

```
┌───────────────────────────────────┐
│ 🛂 PASSPORT                        │
├───────────────────────────────────┤
│ Name: Habib Rahman                │
│ Country: Bangladesh               │
│ DOB: 1990-01-15                   │
│ Passport No: AB1234567            │
│                                   │
│ ┌─────────────────┐               │
│ │                 │               │
│ │  Your Photo     │               │
│ │                 │               │
│ └─────────────────┘               │
│                                   │
│ ────────────────────────          │
│ Government Signature & Seal       │
│ [Official Stamp]                  │
└───────────────────────────────────┘
```

### JWT Token

```
┌───────────────────────────────────┐
│ JWT TOKEN                         │
├───────────────────────────────────┤
│ Header: {alg: HS256, typ: JWT}    │  ← Public info
│                                   │
│ Payload:                          │  ← Public info
│   Subject: 1                      │
│   FirstName: Habib                │
│   LastName: Rahman                │
│   Email: habib@gmail.com          │
│                                   │
│ ────────────────────────          │
│ Signature: kF7x2B...              │  ← Government seal!
│ [Signed with secret]              │
└───────────────────────────────────┘
```

### The Comparison

| Aspect | Passport | JWT |
|--------|----------|-----|
| **Information** | Name, DOB, country | User ID, email, name |
| **Visibility** | Anyone can read | Anyone can decode Base64 |
| **Trust** | Government signature | Server signature |
| **Validation** | Only government can verify | Only server with secret can verify |
| **Tampering** | Change name → signature invalid | Change payload → signature invalid |

### Real-World Scenario

**At Airport:**
```
You: "Here's my passport"
Officer: *Reads your name* ✓
Officer: *Checks signature* ✓
Officer: "Welcome to the country"
```

**With API:**
```
Client: "Here's my JWT"
Server: *Reads your data* ✓
Server: *Checks signature* ✓
Server: "Request approved"
```

### Can You Fake It?

**Passport:**
- You can change your name in passport
- But signature becomes invalid
- Officer: "This passport is tampered!"

**JWT:**
- You can change user ID in payload
- But signature becomes invalid
- Server: "This token is tampered - you hacker!"

### Who Can Validate?

**Passport:**
- Only the **issuing government** can validate
- They have the secret sealing process

**JWT:**
- Only the **server with the secret key** can validate
- They have the secret signing key

This is why JWT is secure - only the server can create valid tokens!

---

## Current Project Problems

Our authentication works, but we have several issues to fix:

### Problem 1: No Database

```
❌ Using in-memory arrays
✓ Need PostgreSQL database
```

When server restarts, all users disappear. Not acceptable for production!

### Problem 2: Single Routes File

```go
// cmd/routes.go - TOO MANY ROUTES IN ONE FILE!
mux.HandleFunc("POST /api/users", ...)
mux.HandleFunc("POST /api/users/login", ...)
mux.HandleFunc("GET /api/products", ...)
mux.HandleFunc("POST /api/products", ...)
mux.HandleFunc("GET /api/products/{id}", ...)
mux.HandleFunc("PUT /api/products/{id}", ...)
mux.HandleFunc("DELETE /api/products/{id}", ...)
// ... 100 more routes?
// ... 1000 more routes??
```

Real applications have hundreds or thousands of routes. We need to **split route files**.

### Problem 3: Mixed Handlers

```
handler/
├── create_product.go     ← Product feature
├── get_product.go        ← Product feature
├── update_product.go     ← Product feature
├── delete_product.go     ← Product feature
├── create_user.go        ← User feature
├── login.go              ← User feature
└── get_user.go           ← User feature
```

All handlers in one folder - messy! We need **feature-based folders**:

```
handler/
├── product/
│   ├── create.go
│   ├── get.go
│   ├── update.go
│   └── delete.go
└── user/
    ├── create.go
    ├── login.go
    └── get.go
```

### Problem 4: Config Loading on Every Request

Every time we call `config.GetConfig()`, it loads from `.env` file:

```go
func AuthenticateJWT(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        cfg := config.GetConfig()  // ← Loads .env EVERY request!
        // ...
    })
}
```

This is inefficient! We should load config **once** at startup.

### Summary of Problems

1. **Database missing** - Need PostgreSQL
2. **Single routes file** - Need to split routes
3. **Mixed handlers** - Need feature-based organization
4. **Config reloading** - Load once, use many times

We'll fix these in upcoming chapters!

---

## Summary

In this chapter, we implemented complete JWT authentication:

### What We Built

1. **Return JWT on Login**
   - Modified login to return JWT token
   - Added JWT_SECRET to configuration
   - Tested token generation

2. **Send JWT in Requests**
   - Authorization header format: `Bearer <token>`
   - Proper spacing and casing
   - Postman environment setup

3. **Parse and Validate JWT**
   - Extract Authorization header
   - Split Bearer and token
   - Split token into three parts
   - Recreate signature with HMAC-SHA256
   - Compare signatures

4. **Authentication Middleware**
   - Created reusable middleware
   - Applied to protected routes
   - Kept public routes accessible

### Key Concepts

**JWT Structure:**
```
header.payload.signature
```

**Validation Algorithm:**
```
1. Split JWT → header, payload, signature
2. message = header + "." + payload
3. newSig = HMAC-SHA256(message, secret)
4. if newSig == signature: valid ✓
```

**Middleware Pattern:**
```go
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w, r) {
        // Validation logic
        next.ServeHTTP(w, r)  // Call next
    })
}
```

### Security Principles

1. **JWT is like a passport** - public info, government seal
2. **Signature proves authenticity** - only server can create valid tokens
3. **Tampering is detected** - changing payload breaks signature
4. **Secret must stay secret** - never commit to Git!

### Next Steps

In upcoming chapters:
- Chapter 50: Database Integration (PostgreSQL)
- Chapter 51: Project Restructuring (feature-based organization)
- Chapter 52: Advanced JWT (refresh tokens, expiration)
- Chapter 53: Password Hashing (bcrypt)

---

## Practice Questions

### Question 1: Why Two Signatures?

**Question:**
When validating JWT, we create a NEW signature and compare it with the OLD signature. Explain why we need to generate a new signature instead of just decoding and checking the old one.

<details>
<summary>Click to see answer</summary>

**Answer:**

We generate a new signature to **prove the JWT hasn't been tampered with**. Here's why:

### The Problem

JWT payload is Base64-encoded, NOT encrypted:

```go
// Anyone can decode payload
payload := "eyJzdWIiOjEsImVtYWlsIjoiaGFiaWJAZ21haWwuY29tIn0"
decoded := base64Decode(payload)
// Result: {"sub":1,"email":"habib@gmail.com"}
```

An attacker could:
1. Decode the payload
2. Change user ID from 1 to 2
3. Re-encode it to Base64
4. Send modified JWT

### Why New Signature Detection Works

**Original JWT (user ID = 1):**
```
header.payload.signature_A
```

Signature A was created with:
```
HMAC-SHA256(header + "." + payload, secret)
```

**Attacker changes payload (user ID = 2):**
```
header.modified_payload.signature_A
```

But signature_A was created from **original** payload!

**Server validation:**
```go
// Server creates NEW signature from RECEIVED data
newSig = HMAC-SHA256(header + "." + modified_payload, secret)

// Compare
if newSig == signature_A {
    // Valid
} else {
    // Invalid - DETECTED TAMPERING!
}
```

Since `modified_payload` is different, the hash will be **completely different** (avalanche effect). The attacker can't create a valid signature because they don't have the secret key!

### Why Can't Attacker Fix Signature?

To create valid signature:
```
signature = HMAC-SHA256(header.payload, SECRET_KEY)
                                        ↑
                              Only server has this!
```

Without the secret key:
- Attacker can change payload ✓
- Attacker can re-encode Base64 ✓
- Attacker **CANNOT** create valid signature ✗

### Complete Attack Scenario

```
1. Attacker intercepts JWT:
   eyJhbGci...  .  eyJzdWIiOjF9  .  abc123...
   
2. Attacker decodes payload:
   {"sub":1,"email":"user@test.com"}
   
3. Attacker changes to:
   {"sub":2,"email":"admin@test.com"}  ← Now admin!
   
4. Attacker encodes back to Base64:
   eyJzdWIiOjJ9
   
5. Attacker sends modified JWT:
   eyJhbGci...  .  eyJzdWIiOjJ9  .  abc123...
                      ↑               ↑
                   Modified      Old signature
                   
6. Server validates:
   newSig = HMAC(header.eyJzdWIiOjJ9, secret)
   newSig = "xyz789..."
   
   xyz789 != abc123  ← MISMATCH!
   
7. Server responds:
   401 Unauthorized - you hacker!
```

### Why Decoding Old Signature Doesn't Help

Even if we could decode the signature back to the hash:
```
signature → decode → hash value
```

We still need to verify that hash was created from:
- This specific header + payload
- Using the correct secret key

The only way to verify is to **recreate** the hash and compare!

### Summary

- **Generate new signature**: Proves data hasn't changed
- **Compare with old signature**: Detects tampering
- **Secret key protection**: Attacker can't create valid signatures
- **Avalanche effect**: Any change → completely different signature

This is the foundation of JWT security!

</details>

---

### Question 2: Middleware Order

**Question:**
Explain what would happen if we applied middleware in this order: `AuthenticateJWT` → `CORS` → `Logger` → `RouteHandler`. Would login still work? Why or why not?

<details>
<summary>Click to see answer</summary>

**Answer:**

**No, login would NOT work!** Let's trace through what happens:

### The Problem

**Middleware order:**
```
AuthenticateJWT → CORS → Logger → RouteHandler
```

**Login request flow:**
```
1. POST /api/users/login
   Body: {email, password}
   Headers: (no Authorization header)
   
2. Enters AuthenticateJWT middleware
   header := r.Header.Get("Authorization")
   // header = "" (empty!)
   
   if header == "" {
       http.Error(w, "Unauthorized", 401)
       return  // ← STOPS HERE!
   }
   
3. Never reaches Login handler!
```

**Result:** Login fails with 401 Unauthorized

### Why This Happens

The `AuthenticateJWT` middleware **blocks all requests** that don't have a JWT token. But to get a JWT token, you must login first! This creates a **chicken-and-egg problem**:

```
To login → Need JWT
To get JWT → Must login
↓
Infinite loop! ❌
```

### Correct Approach: Selective Middleware

**Solution 1: Route-level middleware**

```go
// Public routes - NO authentication
mux.HandleFunc("POST /api/users/login", handler.Login)
mux.HandleFunc("POST /api/users", handler.CreateUser)

// Protected routes - WITH authentication
mux.Handle("POST /api/products",
    middleware.AuthenticateJWT(
        http.HandlerFunc(handler.CreateProduct),
    ),
)
```

**Solution 2: Smart middleware with bypass**

```go
func AuthenticateJWT(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Bypass authentication for certain paths
        if r.URL.Path == "/api/users/login" ||
           r.URL.Path == "/api/users" {
            next.ServeHTTP(w, r)
            return
        }
        
        // For other paths, validate JWT
        // ... validation code ...
    })
}
```

### Correct Middleware Order

**Global middleware should be non-blocking:**
```
Logger → CORS → (route-specific auth) → RouteHandler
```

- **Logger**: Records all requests (no blocking)
- **CORS**: Adds CORS headers (no blocking)
- **Auth**: Only on protected routes (selective blocking)

### Request Flow Examples

**Public route:**
```
POST /api/users/login
 ↓
Logger (logs request)
 ↓
CORS (adds headers)
 ↓
Login handler (processes login)
 ↓
Response with JWT
```

**Protected route:**
```
POST /api/products
 ↓
Logger (logs request)
 ↓
CORS (adds headers)
 ↓
AuthenticateJWT (checks token)
 ↓
CreateProduct handler
 ↓
Response with product
```

### What If Auth Was First?

```
POST /api/users/login
 ↓
AuthenticateJWT (no token! 401 ✗)
 ↓
STOPS HERE
 ↓
(Never reaches login handler)
```

### Summary

Middleware order matters!
- **Global middleware**: Logger, CORS (non-blocking)
- **Selective middleware**: Auth (only on protected routes)
- **Never block**: Public endpoints like login

Correct pattern:
```go
handler := Logger(mux)
handler = CORS(handler)
// Don't wrap with Auth globally!
```

Use route-level auth:
```go
mux.Handle("protected-route",
    Auth(handler),
)
```

</details>

---

### Question 3: Implementing Role-Based Authorization

**Question:**
Our current authentication only checks IF a user is logged in. Implement a system where only users with `isShopOwner: true` can create products. Other authenticated users should get a 403 Forbidden response.

<details>
<summary>Click to see answer</summary>

**Answer:**

We need to add **authorization** (checking permissions) after **authentication** (verifying identity).

### Step 1: Decode JWT Payload

Currently, we only validate the signature. We also need to **decode the payload** to read user information:

```go
// After validating signature...
payloadBytes, err := base64.URLEncoding.DecodeString(jwtPayload)
if err != nil {
    http.Error(w, "Invalid token", http.StatusUnauthorized)
    return
}

// Parse JSON payload
var payload struct {
    Subject   int    `json:"sub"`
    FirstName string `json:"first_name"`
    LastName  string `json:"last_name"`
    Email     string `json:"email"`
    IsShopOwner bool `json:"is_shop_owner"`
}

err = json.Unmarshal(payloadBytes, &payload)
if err != nil {
    http.Error(w, "Invalid token", http.StatusUnauthorized)
    return
}
```

### Step 2: Store User Info in Request Context

Pass decoded user info to the handler:

```go
import "context"

// Create context with user info
ctx := context.WithValue(r.Context(), "user", payload)
r = r.WithContext(ctx)

// Call next handler with updated request
next.ServeHTTP(w, r)
```

### Step 3: Create Authorization Middleware

```go
// middleware/authorize_shop_owner.go
package middleware

import (
    "net/http"
)

// AuthorizeShopOwner checks if user is a shop owner
func AuthorizeShopOwner(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Get user from context
        user := r.Context().Value("user")
        if user == nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        // Type assert to our user struct
        userPayload, ok := user.(map[string]interface{})
        if !ok {
            http.Error(w, "Internal error", http.StatusInternalServerError)
            return
        }
        
        // Check if shop owner
        isShopOwner, ok := userPayload["is_shop_owner"].(bool)
        if !ok || !isShopOwner {
            http.Error(w, "Forbidden - Shop owners only", http.StatusForbidden)
            return
        }
        
        // User is shop owner - allow
        next.ServeHTTP(w, r)
    })
}
```

### Step 4: Apply Both Middleware

```go
// cmd/serve.go
mux.Handle("POST /api/products",
    middleware.AuthenticateJWT(
        middleware.AuthorizeShopOwner(
            http.HandlerFunc(handler.CreateProduct),
        ),
    ),
)
```

### Complete Flow

```
POST /api/products
 ↓
AuthenticateJWT
 ├─ Validate signature ✓
 ├─ Decode payload
 └─ Add user to context
 ↓
AuthorizeShopOwner
 ├─ Read user from context
 ├─ Check is_shop_owner field
 └─ Allow if true, block if false
 ↓
CreateProduct handler
```

### Testing

**Test 1: Shop owner (should succeed)**
```
Login as shop owner:
{
  "email": "shop@test.com",
  "is_shop_owner": true
}

Use JWT to create product:
POST /api/products
Authorization: Bearer <token>

Response: 201 Created ✓
```

**Test 2: Regular user (should fail)**
```
Login as regular user:
{
  "email": "user@test.com",
  "is_shop_owner": false
}

Try to create product:
POST /api/products
Authorization: Bearer <token>

Response: 403 Forbidden - Shop owners only ✗
```

### Enhanced Version: Multiple Roles

For more complex systems:

```go
type Role string

const (
    RoleUser      Role = "user"
    RoleShopOwner Role = "shop_owner"
    RoleAdmin     Role = "admin"
)

func RequireRole(role Role) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            user := r.Context().Value("user")
            // ... check if user has required role ...
            next.ServeHTTP(w, r)
        })
    }
}

// Usage:
mux.Handle("POST /api/products",
    AuthenticateJWT(
        RequireRole(RoleShopOwner)(
            http.HandlerFunc(handler.CreateProduct),
        ),
    ),
)

mux.Handle("DELETE /api/users/{id}",
    AuthenticateJWT(
        RequireRole(RoleAdmin)(
            http.HandlerFunc(handler.DeleteUser),
        ),
    ),
)
```

### Summary

**Authentication:** Who are you?
- Checks JWT signature
- Validates token

**Authorization:** What can you do?
- Checks user permissions
- Validates roles/privileges

Both are needed for secure APIs!

</details>

---

**Next Chapter Preview:**

In Chapter 50, we'll integrate PostgreSQL database to:
1. Persist users and products
2. Use proper SQL queries
3. Handle database connections
4. Implement migrations

Get ready to make our application production-ready! 🚀
