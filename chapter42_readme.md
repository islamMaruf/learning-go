# Chapter 42: Preflight Request With OPTIONS Method (The Security Guard!)

## Table of Contents
- [Introduction](#introduction)
- [What is OPTIONS Method?](#what-is-options-method)
- [What is Preflight Request?](#what-is-preflight-request)
- [Simple vs Complex Requests](#simple-vs-complex-requests)
- [Why Preflight Exists](#why-preflight-exists)
- [How Preflight Works](#how-preflight-works)
- [The Browser Detective](#the-browser-detective)
- [Step by Step Example](#step-by-step-example)
- [Adding Custom Headers](#adding-custom-headers)
- [Fixing CORS for Custom Headers](#fixing-cors-for-custom-headers)
- [Complete Flow Visualization](#complete-flow-visualization)
- [Security Aspect](#security-aspect)
- [Practice Questions](#practice-questions)
- [Summary](#summary)
- [What's Next?](#whats-next)

---

## Introduction

**Today's Class:** Preflight Request With OPTIONS Method

Hello everyone! Last class we learned about POST requests and how to create products. Today we're going to learn about something **very important** that you probably never heard about before: **Preflight Requests**!

**What Happened Last Class:**

We had this code in our `createProduct` handler:
```go
if r.Method == http.MethodOptions {
    w.WriteHeader(http.StatusOK)
    return
}
```

But in our `getProducts` handler, we **didn't** have this OPTIONS handling. Why?

**Today's Mystery:**
- What is OPTIONS method?
- What is a preflight request?
- Why does browser send OPTIONS before POST/GET?
- How to handle it properly?

Let's solve this mystery! 🕵️

---

## What is OPTIONS Method?

Remember the **5 main HTTP methods**? Actually, there's a **6th one** that works behind the scenes!

### The HTTP Methods Family

```
┌────────────────────────────────────────┐
│         HTTP Methods                   │
├────────────────────────────────────────┤
│  GET     →  Read data                  │
│  POST    →  Create data                │
│  PUT     →  Update (full)              │
│  PATCH   →  Update (partial)           │
│  DELETE  →  Delete data                │
│  OPTIONS →  Check permissions! ✅      │
└────────────────────────────────────────┘
```

### What Does OPTIONS Do?

**OPTIONS** is like a **security guard** that checks "Can I go through?" before you actually enter!

```
┌────────────────────────────────────────────┐
│         OPTIONS = Permission Checker       │
├────────────────────────────────────────────┤
│                                            │
│  Browser: "Can I send POST here?"          │
│           "Can I use custom headers?"      │
│           "What methods are allowed?"      │
│                                            │
│  Server:  "Yes, you can POST"              │
│           "These headers are allowed"      │
│           "Go ahead!" (200 OK)             │
│                                            │
└────────────────────────────────────────────┘
```

**Real-World Analogy:**

Think of a nightclub:
```
OPTIONS = Security guard checking your ID before you enter
GET/POST = Actually entering the club

Without OPTIONS check:
  Anyone could enter → Dangerous! ❌

With OPTIONS check:
  Security verifies first → Safe! ✅
```

---

## What is Preflight Request?

**Preflight Request** = A request sent **before** the actual request to check permissions.

### Definition

```
Preflight Request:
  - Uses OPTIONS method
  - Sent automatically by browser
  - Checks if actual request is allowed
  - Must return 200 OK to proceed
```

### When Does It Happen?

**NOT all requests trigger preflight!**

```
┌────────────────────────────────────────────┐
│       Simple Request (No Preflight)        │
├────────────────────────────────────────────┤
│  - GET with standard headers               │
│  - POST with simple content-type           │
│  - No custom headers                       │
│                                            │
│  Browser: "This looks safe, send directly" │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│      Complex Request (Preflight!)          │
├────────────────────────────────────────────┤
│  - Custom headers added                    │
│  - Non-standard content-type               │
│  - Methods like PUT, DELETE, PATCH         │
│                                            │
│  Browser: "Wait! Check permissions first!" │
└────────────────────────────────────────────┘
```

---

## Simple vs Complex Requests

### Simple Request Criteria

A request is **simple** if it meets ALL these conditions:

**1. Method must be:**
- GET
- POST
- HEAD

**2. Headers only contain:**
- Accept
- Accept-Language
- Content-Language
- Content-Type (only if: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`)

**3. No custom headers**

### Complex Request Triggers

A request becomes **complex** (triggers preflight) if:

**1. Uses these methods:**
- PUT
- DELETE
- PATCH

**2. Has custom headers:**
- Authorization
- X-Custom-Header
- Any header not in the simple list

**3. Has these content-types:**
- `application/json` ✅ (Most common!)
- `application/xml`
- Any non-simple content-type

### Visual Comparison

```
┌──────────────────────────────────────────────────────┐
│              Simple Request Flow                     │
├──────────────────────────────────────────────────────┤
│                                                      │
│  1. Browser → Server (GET /products)                 │
│                                                      │
│  2. Server → Browser (Response)                      │
│                                                      │
│  Done! No preflight needed.                          │
│                                                      │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│             Complex Request Flow                     │
├──────────────────────────────────────────────────────┤
│                                                      │
│  1. Browser → Server (OPTIONS /products) ✅ Preflight│
│                                                      │
│  2. Server → Browser (200 OK + permissions)          │
│                                                      │
│  3. Browser → Server (GET /products) ✅ Actual request│
│                                                      │
│  4. Server → Browser (Response)                      │
│                                                      │
│  Done! Preflight checked permissions first.          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## Why Preflight Exists

### The Security Problem

**Without Preflight:**
```
Evil Website:
  → Sends DELETE request to yourbank.com
  → Uses your cookies (you're logged in!)
  → Deletes your account! 😱

Result: Disaster!
```

**With Preflight:**
```
Evil Website:
  → Browser sends OPTIONS first
  → Bank server: "This origin not allowed!" ❌
  → Browser blocks the DELETE
  → Your account is safe! ✅
```

### Browser's Protection

The browser acts as a **protective shield**:

```
┌────────────────────────────────────────────┐
│         Browser's Security Check           │
├────────────────────────────────────────────┤
│                                            │
│  1. User visits evil.com                   │
│  2. evil.com tries to call yourbank.com    │
│  3. Browser: "Wait! Different origin!"     │
│  4. Browser sends OPTIONS first            │
│  5. Bank: "evil.com not allowed!"          │
│  6. Browser blocks the request             │
│  7. User protected! 🛡️                    │
│                                            │
└────────────────────────────────────────────┘
```

---

## How Preflight Works

### The Complete Process

```
Step 1: You Make Request
  ↓
Step 2: Browser Checks Request Type
  ↓
Is it Complex? (Custom headers, JSON, etc.)
  ↓
YES → Send OPTIONS First (Preflight)
  ↓
Step 3: Server Responds to OPTIONS
  ↓
200 OK + Allowed Headers/Methods?
  ↓
YES → Send Actual Request
NO  → Block & Show CORS Error
  ↓
Step 4: Server Processes Actual Request
  ↓
Step 5: Browser Receives Final Response
```

### Example Scenario

**You want to GET products with a custom header:**

```javascript
fetch('http://localhost:8080/products', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json',
    'Habib': 'Hello'  // Custom header!
  }
});
```

**What happens:**

```
Phase 1: Preflight (Automatic)
────────────────────────────────
Browser → Server:
  OPTIONS /products
  Headers:
    - Origin: http://localhost:3000
    - Access-Control-Request-Headers: Habib
    - Access-Control-Request-Method: GET

Server → Browser:
  200 OK
  Headers:
    - Access-Control-Allow-Origin: *
    - Access-Control-Allow-Headers: Habib
    - Access-Control-Allow-Methods: GET

Phase 2: Actual Request
────────────────────────────────
Browser → Server:
  GET /products
  Headers:
    - Habib: Hello

Server → Browser:
  200 OK
  Body: [products...]
```

---

## The Browser Detective

Think of the browser as a **detective** investigating every request:

### Investigation Process

```
┌────────────────────────────────────────────────────┐
│         Browser Detective at Work 🕵️              │
├────────────────────────────────────────────────────┤
│                                                    │
│  Detective: "Hmm, let me check this request..."    │
│                                                    │
│  🔍 Checking method... GET ✅                      │
│  🔍 Checking headers... Wait! "Habib: Hello"?      │
│  🔍 That's not standard! ⚠️                        │
│                                                    │
│  Detective: "This is suspicious! Complex request!" │
│                                                    │
│  🚨 Action: Send OPTIONS to check permissions      │
│                                                    │
│  OPTIONS Response:                                 │
│    - Allowed Headers: Habib ✅                     │
│    - Allowed Origin: * ✅                          │
│                                                    │
│  Detective: "OK, it's allowed. Proceed!"           │
│                                                    │
└────────────────────────────────────────────────────┘
```

### What Detective Looks For

**Red Flags (Trigger Preflight):**
```
❌ Custom headers
❌ Content-Type: application/json
❌ Methods: PUT, DELETE, PATCH
❌ Authorization tokens
❌ X-Requested-With headers
```

**Green Flags (No Preflight):**
```
✅ Standard GET/POST
✅ Only Accept, Accept-Language headers
✅ Content-Type: text/plain or form-urlencoded
✅ No custom stuff
```

---

## Step by Step Example

Let's walk through a **real example** with custom headers!

### Initial Setup

**Frontend Code (React):**
```javascript
// Fetching products with custom header
fetch('http://localhost:8080/products', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json',
    'Habib': 'Hello'  // Our custom header
  }
})
.then(response => response.json())
.then(data => console.log(data));
```

**Backend Code (Go) - Before Fix:**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // CORS headers
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    // Only checking GET method
    if r.Method != http.MethodGet {
        http.Error(w, "Please give me GET request", 400)
        return
    }
    
    // Return products
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}
```

### What Goes Wrong

**Browser's Actions:**

1. **Sees custom header "Habib"** → "This is complex!"
2. **Sends OPTIONS first** (automatic preflight)
3. **Server doesn't handle OPTIONS** → Returns 400 error
4. **Browser blocks GET request** → CORS error!

**Console Error:**
```
CORS policy: Response to preflight request doesn't pass access 
control check: It does not have HTTP ok status.
```

**Visual Problem:**
```
Browser                          Server
   │                               │
   │  OPTIONS /products            │
   ├──────────────────────────────>│
   │                               │
   │                    ❌ 400 Bad Request
   │<──────────────────────────────┤
   │                               │
   │  ❌ CORS Error!               │
   │  (Blocks GET request)         │
   │                               │
```

---

## Adding Custom Headers

### Demonstration: Adding "Habib" Header

**Step 1: Add Custom Header in Frontend**

```javascript
// In your fetch call
headers: {
  'Content-Type': 'application/json',
  'Habib': 'Hello'  // Custom header added
}
```

**Step 2: Browser Detects Complex Request**

```
Browser's Internal Logic:
┌────────────────────────────────────┐
│  Checking request...               │
│  - Method: GET ✅                  │
│  - Header: Content-Type ⚠️         │
│  - Header: Habib ⚠️ (Custom!)      │
│                                    │
│  Conclusion: COMPLEX REQUEST       │
│  Action: SEND PREFLIGHT            │
└────────────────────────────────────┘
```

**Step 3: Browser Sends Preflight**

```
OPTIONS /products HTTP/1.1
Host: localhost:8080
Origin: http://localhost:3000
Access-Control-Request-Method: GET
Access-Control-Request-Headers: Habib
```

**Step 4: Server Must Respond Properly**

If server doesn't handle OPTIONS → **CORS Error!**

---

## Fixing CORS for Custom Headers

### Problem: OPTIONS Not Handled

**Current Code (getProducts):**
```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    // This rejects OPTIONS! ❌
    if r.Method != http.MethodGet {
        http.Error(w, "Please give me GET request", 400)
        return
    }
    
    // ... rest of code
}
```

**What happens:**
- Browser sends OPTIONS
- Server sees it's not GET
- Returns 400 error
- Browser blocks actual GET request

### Solution 1: Handle OPTIONS First

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // CORS headers
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    // ✅ Handle OPTIONS preflight
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // Now check for GET
    if r.Method != http.MethodGet {
        http.Error(w, "Please give me GET request", 400)
        return
    }
    
    // Return products
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}
```

**But this still doesn't work!** Why? 🤔

### Solution 2: Allow Custom Headers

Browser checks if our custom header "Habib" is allowed!

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // CORS headers
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    // ✅ Allow custom headers
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Habib")
    
    // Handle OPTIONS
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // Check GET
    if r.Method != http.MethodGet {
        http.Error(w, "Please give me GET request", 400)
        return
    }
    
    // Return products
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}
```

**Now it works!** ✅

### Complete Fixed Code

```go
func getProducts(w http.ResponseWriter, r *http.Request) {
    // Step 1: Set CORS headers
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Content-Type", "application/json")
    
    // Step 2: Allow custom headers (IMPORTANT!)
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Habib")
    
    // Step 3: Handle OPTIONS preflight
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return // Stop here for OPTIONS
    }
    
    // Step 4: Validate GET method
    if r.Method != http.MethodGet {
        http.Error(w, "Please give me GET request", 400)
        return
    }
    
    // Step 5: Return product list
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}
```

---

## Complete Flow Visualization

### Before Fix (Error)

```
┌──────────────────────────────────────────────────────────┐
│                  Failed Request Flow                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Frontend (port 3000)                                    │
│    ↓                                                     │
│  Browser detects custom header "Habib"                   │
│    ↓                                                     │
│  Browser: "Complex request! Send OPTIONS first"          │
│    ↓                                                     │
│  OPTIONS /products                                       │
│    ├────────────────────────────> Backend (port 8080)   │
│    │                                                     │
│    │  getProducts() receives OPTIONS                    │
│    │  Checks: r.Method != GET                           │
│    │  Returns: 400 Bad Request ❌                        │
│    │                                                     │
│    │<──────────────────────────────                     │
│  Browser receives: 400 error                             │
│    ↓                                                     │
│  Browser: "Preflight failed! Block request!"             │
│    ↓                                                     │
│  ❌ CORS ERROR                                           │
│  ❌ Actual GET never sent                                │
│  ❌ No products displayed                                │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### After Fix (Success)

```
┌──────────────────────────────────────────────────────────┐
│                 Successful Request Flow                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Frontend (port 3000)                                    │
│    ↓                                                     │
│  Browser detects custom header "Habib"                   │
│    ↓                                                     │
│  Browser: "Complex request! Send OPTIONS first"          │
│    ↓                                                     │
│  ═══════════ PREFLIGHT REQUEST ═══════════              │
│    ↓                                                     │
│  OPTIONS /products                                       │
│  Headers:                                                │
│    - Access-Control-Request-Headers: Habib              │
│    ├────────────────────────────> Backend (port 8080)   │
│    │                                                     │
│    │  getProducts() receives OPTIONS                    │
│    │  Checks: r.Method == OPTIONS ✅                     │
│    │  Returns: 200 OK                                   │
│    │  Headers:                                          │
│    │    - Access-Control-Allow-Headers: Habib ✅        │
│    │                                                     │
│    │<──────────────────────────────                     │
│  Browser receives: 200 OK                                │
│    ↓                                                     │
│  Browser: "Preflight passed! Send actual request"        │
│    ↓                                                     │
│  ═══════════ ACTUAL REQUEST ═══════════                 │
│    ↓                                                     │
│  GET /products                                           │
│  Headers:                                                │
│    - Habib: Hello                                        │
│    ├────────────────────────────> Backend (port 8080)   │
│    │                                                     │
│    │  getProducts() receives GET                        │
│    │  Checks: r.Method == GET ✅                         │
│    │  Returns: Product list JSON                        │
│    │                                                     │
│    │<──────────────────────────────                     │
│  Browser receives: Product data                          │
│    ↓                                                     │
│  ✅ SUCCESS                                              │
│  ✅ Products displayed                                   │
│  ✅ Custom header worked                                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Key Differences

**Failed Flow:**
```
1. OPTIONS → 400 error
2. Browser blocks GET
3. CORS error shown
```

**Successful Flow:**
```
1. OPTIONS → 200 OK (with allowed headers)
2. Browser allows GET
3. GET → Products returned
4. Success!
```

---

## Security Aspect

### Why This Is Important

**Preflight is a SECURITY FEATURE** that protects users!

### Attack Scenario Without Preflight

```
┌────────────────────────────────────────────┐
│         Without Preflight Protection       │
├────────────────────────────────────────────┤
│                                            │
│  1. You visit evil-site.com                │
│                                            │
│  2. evil-site.com has JavaScript:          │
│     fetch('https://yourbank.com/transfer', {
│       method: 'DELETE',                    │
│       headers: {                           │
│         'Authorization': 'your-token'      │
│       }                                    │
│     })                                     │
│                                            │
│  3. Browser sends DELETE directly ❌       │
│                                            │
│  4. Bank deletes your account! 😱          │
│                                            │
└────────────────────────────────────────────┘
```

### Defense With Preflight

```
┌────────────────────────────────────────────┐
│          With Preflight Protection         │
├────────────────────────────────────────────┤
│                                            │
│  1. You visit evil-site.com                │
│                                            │
│  2. evil-site.com tries same attack        │
│                                            │
│  3. Browser detects:                       │
│     - Different origin                     │
│     - Custom Authorization header          │
│     - DELETE method                        │
│                                            │
│  4. Browser sends OPTIONS first ✅         │
│                                            │
│  5. Bank server responds:                  │
│     "evil-site.com NOT ALLOWED" ❌         │
│                                            │
│  6. Browser BLOCKS the DELETE ✅           │
│                                            │
│  7. Your account is SAFE! 🛡️              │
│                                            │
└────────────────────────────────────────────┘
```

### Browser as Bodyguard

```
Think of browser as your personal bodyguard:

Without Bodyguard:
  Anyone → Bank Account ❌
  Dangerous!

With Bodyguard (Preflight):
  Anyone → Bodyguard checks ID → Bank Account ✅
  Safe!
```

---

## Practice Questions

### Question 1: What is Preflight Request?
**Q:** Explain what a preflight request is, when it happens, and why it exists.

<details>
<summary><b>Answer</b></summary>

**What is Preflight Request?**

A **preflight request** is an automatic HTTP request sent by the browser using the OPTIONS method **before** sending the actual request. It's a security check to verify if the actual request is allowed.

**When Does It Happen?**

Preflight happens when the request is **"complex"**, which means:

**1. Non-simple HTTP methods:**
- PUT
- DELETE
- PATCH

**2. Custom headers:**
```javascript
headers: {
  'Authorization': 'Bearer token',
  'X-Custom-Header': 'value',
  'Habib': 'Hello'  // Any custom header
}
```

**3. Non-simple Content-Type:**
```javascript
'Content-Type': 'application/json'  // Triggers preflight
'Content-Type': 'application/xml'   // Triggers preflight
```

**When It DOESN'T Happen (Simple Requests):**
```javascript
// Simple GET - no preflight
fetch('http://api.example.com/data', {
  method: 'GET',
  headers: {
    'Accept': 'text/html'  // Standard header
  }
});

// Simple POST - no preflight
fetch('http://api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: 'name=John&age=30'
});
```

**Why Does It Exist?**

**Security Protection:**

1. **Prevents unauthorized cross-origin requests**
   - Evil sites can't silently delete your data
   - Protects your cookies and authentication

2. **Gives servers control**
   - Server decides which origins are allowed
   - Server decides which headers are allowed
   - Server decides which methods are allowed

3. **Protects users**
   - Browser checks permissions first
   - Blocks dangerous requests before they happen
   - You don't even know the attack was attempted!

**Real Attack Scenario:**

```
WITHOUT PREFLIGHT:
  evil.com → YourBank.com (DELETE /account)
  → Account deleted! 😱

WITH PREFLIGHT:
  evil.com → OPTIONS check → Bank says "NO!" 
  → DELETE blocked! → Account safe! ✅
```

**Summary:**
- **What:** OPTIONS request sent before actual request
- **When:** Complex requests (custom headers, JSON, PUT/DELETE/PATCH)
- **Why:** Security - verifies permissions before actual request
- **Who:** Browser sends it automatically (you don't control it)
</details>

---

### Question 2: Simple vs Complex Requests
**Q:** What makes a request "simple" vs "complex"? Give examples of each.

<details>
<summary><b>Answer</b></summary>

**Simple Request Criteria:**

A request is **simple** (no preflight) if ALL these conditions are met:

**1. Method is one of:**
- `GET`
- `POST`
- `HEAD`

**2. Only these headers:**
- `Accept`
- `Accept-Language`
- `Content-Language`
- `Content-Type` (only if one of):
  - `application/x-www-form-urlencoded`
  - `multipart/form-data`
  - `text/plain`

**3. No custom headers**

**Simple Request Examples:**

```javascript
// Example 1: Simple GET
fetch('http://api.example.com/products', {
  method: 'GET',
  headers: {
    'Accept': 'text/html',
    'Accept-Language': 'en-US'
  }
});
// No preflight ✅

// Example 2: Simple POST with form data
fetch('http://api.example.com/submit', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: 'name=John&email=john@example.com'
});
// No preflight ✅

// Example 3: Simple POST with plain text
fetch('http://api.example.com/log', {
  method: 'POST',
  headers: {
    'Content-Type': 'text/plain'
  },
  body: 'Log message here'
});
// No preflight ✅
```

**Complex Request Triggers:**

A request becomes **complex** (triggers preflight) if ANY of these:

**1. Non-simple methods:**
- `PUT`
- `DELETE`
- `PATCH`
- `CONNECT`
- `OPTIONS`
- `TRACE`

**2. Custom headers:**
- `Authorization`
- `X-Requested-With`
- `X-Custom-Header`
- Any header not in simple list

**3. Non-simple Content-Type:**
- `application/json` ← Most common!
- `application/xml`
- Any other

**Complex Request Examples:**

```javascript
// Example 1: JSON content (COMPLEX!)
fetch('http://api.example.com/products', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'  // Not simple!
  },
  body: JSON.stringify({ name: 'Product' })
});
// Preflight sent! ⚠️

// Example 2: Authorization header (COMPLEX!)
fetch('http://api.example.com/products', {
  method: 'GET',
  headers: {
    'Authorization': 'Bearer token123'  // Custom header!
  }
});
// Preflight sent! ⚠️

// Example 3: DELETE method (COMPLEX!)
fetch('http://api.example.com/products/1', {
  method: 'DELETE'  // Not simple method!
});
// Preflight sent! ⚠️

// Example 4: Custom header (COMPLEX!)
fetch('http://api.example.com/products', {
  method: 'GET',
  headers: {
    'X-User-ID': '12345',  // Custom!
    'Habib': 'Hello'       // Custom!
  }
});
// Preflight sent! ⚠️
```

**Side-by-Side Comparison:**

```
SIMPLE REQUEST              COMPLEX REQUEST
──────────────              ───────────────
GET                         PUT / DELETE / PATCH
Standard headers            Custom headers
text/plain                  application/json
No preflight                Preflight required
Direct to server            OPTIONS first, then actual
```

**Why The Distinction?**

**Simple requests:**
- Been around since web's beginning
- Can't do much harm
- Browser trusts them

**Complex requests:**
- Modern features (JSON, REST APIs)
- Could be dangerous (DELETE, custom auth)
- Browser checks first with OPTIONS

**Common Gotcha:**

```javascript
// You might think this is simple...
fetch('http://api.example.com/data', {
  method: 'POST',
  body: JSON.stringify({ name: 'John' })
});

// But forgot to set Content-Type!
// Browser still sees JSON and triggers preflight!
```

**Pro Tip:**

In modern web development, **most requests are complex** because we use:
- JSON (`application/json`)
- Authorization tokens
- REST methods (PUT, DELETE)

So always prepare your server to handle OPTIONS!
</details>

---

### Question 3: Handling OPTIONS in Go
**Q:** Write a complete handler function that properly handles OPTIONS preflight for a GET endpoint with custom headers.

<details>
<summary><b>Answer</b></summary>

**Complete Handler with OPTIONS Support:**

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
    ImageURL    string  `json:"image"`
}

// Global product list
var productList []Product

// Complete getProducts with OPTIONS handling
func getProducts(w http.ResponseWriter, r *http.Request) {
    // ========================================
    // STEP 1: Set CORS Headers
    // ========================================
    // Allow any origin (for development)
    w.Header().Set("Access-Control-Allow-Origin", "*")
    
    // Set response content type
    w.Header().Set("Content-Type", "application/json")
    
    // ========================================
    // STEP 2: Allow Custom Headers
    // ========================================
    // CRITICAL: Tell browser which custom headers are allowed
    // Without this, preflight will fail!
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, X-User-ID, Habib")
    
    // ========================================
    // STEP 3: Allow GET Method
    // ========================================
    // Tell browser which methods are allowed
    w.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
    
    // ========================================
    // STEP 4: Handle OPTIONS Preflight
    // ========================================
    // Browser sends OPTIONS before actual GET
    // We must respond with 200 OK
    if r.Method == http.MethodOptions {
        // Return 200 OK for preflight
        w.WriteHeader(http.StatusOK)
        return // STOP HERE - don't process further
    }
    
    // ========================================
    // STEP 5: Validate GET Method
    // ========================================
    // Only allow GET for actual request
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET method is allowed", http.StatusMethodNotAllowed)
        return
    }
    
    // ========================================
    // STEP 6: Return Product List
    // ========================================
    // Encode and send products as JSON
    encoder := json.NewEncoder(w)
    err := encoder.Encode(productList)
    
    if err != nil {
        http.Error(w, "Failed to encode products", http.StatusInternalServerError)
    }
}

// Main function
func main() {
    // Initialize with sample products
    productList = []Product{
        {ID: 1, Title: "Orange", Description: "Fresh orange", Price: 100, ImageURL: "url1"},
        {ID: 2, Title: "Apple", Description: "Red apple", Price: 40, ImageURL: "url2"},
        {ID: 3, Title: "Banana", Description: "Yellow banana", Price: 5, ImageURL: "url3"},
    }
    
    // Create router
    mux := http.NewServeMux()
    
    // Register handler
    mux.HandleFunc("/products", getProducts)
    
    // Start server
    http.ListenAndServe(":8080", mux)
}
```

**Alternative: Middleware Approach (Better!)**

For multiple endpoints, create middleware:

```go
// CORS middleware
func corsMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Set CORS headers for all requests
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, X-User-ID")
        w.Header().Set("Content-Type", "application/json")
        
        // Handle preflight
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusOK)
            return
        }
        
        // Call next handler
        next(w, r)
    }
}

// Simplified handler (CORS handled by middleware)
func getProducts(w http.ResponseWriter, r *http.Request) {
    // Just validate method
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET allowed", http.StatusMethodNotAllowed)
        return
    }
    
    // Return products
    encoder := json.NewEncoder(w)
    encoder.Encode(productList)
}

func main() {
    mux := http.NewServeMux()
    
    // Wrap handler with middleware
    mux.HandleFunc("/products", corsMiddleware(getProducts))
    
    http.ListenAndServe(":8080", mux)
}
```

**Testing the Handler:**

```javascript
// Frontend test with custom header
fetch('http://localhost:8080/products', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json',
    'X-User-ID': '12345'  // Custom header
  }
})
.then(response => response.json())
.then(data => console.log(data));

// What happens:
// 1. Browser sends OPTIONS (preflight)
//    → Server returns 200 OK with allowed headers
// 2. Browser sends GET (actual request)
//    → Server returns product list
// 3. Success! ✅
```

**Common Mistakes to Avoid:**

```go
// ❌ WRONG: Not handling OPTIONS
if r.Method != http.MethodGet {
    http.Error(w, "Only GET", 400)
    return
}
// OPTIONS gets 400 error → CORS fails!

// ❌ WRONG: Not allowing custom headers
w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
// Frontend sends "Authorization" → CORS fails!

// ❌ WRONG: Checking OPTIONS after GET check
if r.Method != http.MethodGet {
    return // OPTIONS blocked here!
}
if r.Method == http.MethodOptions {
    // Never reaches here!
}

// ✅ CORRECT: Check OPTIONS FIRST
if r.Method == http.MethodOptions {
    w.WriteHeader(http.StatusOK)
    return
}
if r.Method != http.MethodGet {
    http.Error(w, "Only GET", 400)
    return
}
```

**Key Takeaways:**

1. **Always handle OPTIONS first**
2. **Set Access-Control-Allow-Headers** for custom headers
3. **Return 200 OK** for OPTIONS
4. **Use middleware** for multiple endpoints
5. **Test with real browser** (Postman doesn't send preflight!)
</details>

---

### Question 4: Debugging CORS Errors
**Q:** You're getting a CORS error. Walk through the debugging steps to identify and fix the issue.

<details>
<summary><b>Answer</b></summary>

**Debugging CORS Errors - Complete Guide:**

### Step 1: Check Browser Console

**Common Error Messages:**

```
Error 1:
"Access to fetch at 'http://localhost:8080/products' from origin 
'http://localhost:3000' has been blocked by CORS policy: 
Response to preflight request doesn't pass access control check: 
No 'Access-Control-Allow-Origin' header is present."

Problem: Missing Allow-Origin header
```

```
Error 2:
"Access to fetch at 'http://localhost:8080/products' from origin 
'http://localhost:3000' has been blocked by CORS policy: 
Request header field authorization is not allowed by 
Access-Control-Allow-Headers in preflight response."

Problem: Custom header not allowed
```

```
Error 3:
"Access to fetch at 'http://localhost:8080/products' from origin 
'http://localhost:3000' has been blocked by CORS policy: 
Method DELETE is not allowed by Access-Control-Allow-Methods."

Problem: Method not allowed
```

### Step 2: Open Network Tab

**Check the requests:**

```
1. Look for OPTIONS request (preflight)
2. Check its status code:
   - 200 OK → Good, check response headers
   - 400/404/500 → Server not handling OPTIONS!
3. Check response headers:
   - Access-Control-Allow-Origin: * (or your origin)
   - Access-Control-Allow-Headers: (your custom headers)
   - Access-Control-Allow-Methods: (your method)
```

**Visual Network Tab Check:**

```
Network Tab:
┌──────────────────────────────────────────┐
│ Request  | Method  | Status | Type       │
├──────────────────────────────────────────┤
│ products | OPTIONS | 400    | preflight  │ ← Problem!
│ products | GET     | (blocked)           │ ← Never sent
└──────────────────────────────────────────┘

Should be:
┌──────────────────────────────────────────┐
│ Request  | Method  | Status | Type       │
├──────────────────────────────────────────┤
│ products | OPTIONS | 200    | preflight  │ ← Good!
│ products | GET     | 200    | xhr        │ ← Success!
└──────────────────────────────────────────┘
```

### Step 3: Check Request Headers

**Click on OPTIONS request:**

```
Request Headers:
  Access-Control-Request-Method: GET
  Access-Control-Request-Headers: authorization, x-user-id
  Origin: http://localhost:3000
```

**These tell you what frontend is asking for!**

### Step 4: Check Response Headers

**Click on OPTIONS response:**

```
Response Headers (What you have):
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET
  
Response Headers (What you need):
  Access-Control-Allow-Origin: *
  Access-Control-Allow-Methods: GET
  Access-Control-Allow-Headers: authorization, x-user-id ← MISSING!
```

### Step 5: Common Fixes

**Fix 1: Server Not Handling OPTIONS**

```go
// ❌ Before (broken)
func handler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    // ...
}

// ✅ After (fixed)
func handler(w http.ResponseWriter, r *http.Request) {
    // Handle OPTIONS first!
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    if r.Method != http.MethodGet {
        http.Error(w, "Only GET", 400)
        return
    }
    // ...
}
```

**Fix 2: Missing Allow-Headers**

```go
// ❌ Before (broken)
w.Header().Set("Access-Control-Allow-Origin", "*")

// ✅ After (fixed)
w.Header().Set("Access-Control-Allow-Origin", "*")
w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
```

**Fix 3: Missing Allow-Methods**

```go
// ❌ Before (broken)
w.Header().Set("Access-Control-Allow-Origin", "*")

// ✅ After (fixed)
w.Header().Set("Access-Control-Allow-Origin", "*")
w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
```

**Fix 4: Typo in Header Name**

```go
// ❌ Wrong (typo)
w.Header().Set("Access-Control-Allow-Headers", "Content-Type Habib")  // Missing comma!

// ✅ Correct
w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Habib")  // With comma
```

### Step 6: Debugging Checklist

```
✅ Check 1: Is server handling OPTIONS?
   if r.Method == http.MethodOptions {
       w.WriteHeader(http.StatusOK)
       return
   }

✅ Check 2: Is Allow-Origin set?
   w.Header().Set("Access-Control-Allow-Origin", "*")

✅ Check 3: Are custom headers allowed?
   w.Header().Set("Access-Control-Allow-Headers", "your-custom-headers")

✅ Check 4: Is method allowed?
   w.Header().Set("Access-Control-Allow-Methods", "GET, POST, ...")

✅ Check 5: Is OPTIONS check BEFORE method check?
   // OPTIONS first, then GET/POST check

✅ Check 6: Correct syntax (commas, spelling)?
   "Content-Type, Authorization" not "Content-Type Authorization"
```

### Step 7: Testing Process

```bash
# 1. Start backend
go run main.go

# 2. Start frontend
npm start

# 3. Open browser to http://localhost:3000

# 4. Open DevTools (F12)

# 5. Go to Network tab

# 6. Trigger request

# 7. Check:
#    - Is OPTIONS present?
#    - Is OPTIONS returning 200?
#    - Are headers correct?
#    - Is actual request sent after OPTIONS?
```

### Step 8: Quick Fix Template

```go
func yourHandler(w http.ResponseWriter, r *http.Request) {
    // 1. Set all CORS headers FIRST
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
    
    // 2. Handle OPTIONS IMMEDIATELY
    if r.Method == http.MethodOptions {
        w.WriteHeader(http.StatusOK)
        return
    }
    
    // 3. Now handle your actual logic
    // ...
}
```

**Pro Tips:**

1. **Test with actual browser**, not Postman (Postman skips preflight!)
2. **Check Network tab** - it shows everything
3. **Read error messages carefully** - they tell you what's missing
4. **Add all headers at top** of handler
5. **Handle OPTIONS first** before any other checks
</details>

---

### Question 5: Real-World Scenario
**Q:** Explain a complete real-world scenario where preflight protects a user from a malicious website.

<details>
<summary><b>Answer</b></summary>

**Complete Attack Scenario with Preflight Protection:**

### The Setup

**Players:**
1. **You** - Regular user browsing the web
2. **YourBank.com** - Your online bank
3. **Evil-Site.com** - Malicious website
4. **Browser** - Your protector (Chrome/Firefox/Safari)

### Scenario: Account Deletion Attack

**Phase 1: You're Logged into Your Bank**

```
1. You visit YourBank.com
2. You log in successfully
3. Browser stores authentication:
   - Cookie: session_token=abc123xyz
   - Stays logged in for 30 minutes
```

### Phase 2: You Visit Evil Site (Unknowingly)

```
4. You click a link to Evil-Site.com
   (Maybe from email, social media, or ad)
5. Evil-Site.com loads
6. You see: "Congratulations! You won $1000!"
```

### Phase 3: Evil Site's Hidden Attack

**Evil Site's Malicious JavaScript:**

```javascript
// Hidden in Evil-Site.com's code
// User can't see this!

// Attempt to delete user's bank account
fetch('https://yourbank.com/api/account/delete', {
  method: 'DELETE',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': 'stolen-token'
  },
  credentials: 'include'  // Send cookies!
});
```

**What Evil Site Hopes:**
```
Send DELETE request → Bank's cookies auto-attached 
→ Bank thinks it's real user → Account deleted! 😱
```

### Phase 4: Browser's Preflight Protection (Hero!)

**Step 1: Browser Detects Attack**

```
Browser's Internal Logic:
┌───────────────────────────────────────┐
│ 🤔 Analyzing request...                │
│                                       │
│ Origin: evil-site.com                 │
│ Target: yourbank.com                  │
│ Method: DELETE                        │
│ Custom header: X-CSRF-Token           │
│                                       │
│ 🚨 ALERT: Cross-origin + Complex!    │
│ 🛡️ MUST check with preflight!        │
└───────────────────────────────────────┘
```

**Step 2: Browser Sends Preflight (OPTIONS)**

```
Browser → YourBank.com:

OPTIONS /api/account/delete HTTP/1.1
Host: yourbank.com
Origin: https://evil-site.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: X-CSRF-Token
```

**Step 3: Bank Server Responds**

```
YourBank.com → Browser:

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://yourbank.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Content-Type
```

**Key Point: Bank does NOT allow:**
- Origin: evil-site.com ❌
- Method: DELETE ❌
- Header: X-CSRF-Token ❌

**Step 4: Browser Blocks Attack**

```
Browser's Decision:
┌───────────────────────────────────────┐
│ Preflight Response Analysis:          │
│                                       │
│ ❌ evil-site.com NOT in allowed origins│
│ ❌ DELETE NOT in allowed methods      │
│ ❌ X-CSRF-Token NOT in allowed headers│
│                                       │
│ 🛑 DECISION: BLOCK REQUEST!           │
│ 🛡️ USER PROTECTED!                    │
└───────────────────────────────────────┘
```

**Console shows:**
```
CORS Error: Request blocked by browser
Origin 'evil-site.com' not allowed
```

**Result:** DELETE request **never sent**! Account safe! ✅

### Complete Flow Visualization

```
┌──────────────────────────────────────────────────────────┐
│              Attack Prevention Flow                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. You visit Evil-Site.com                              │
│     (You think it's harmless)                            │
│                                                          │
│  2. Evil JavaScript tries attack:                        │
│     fetch('yourbank.com/delete')                         │
│     with DELETE method + cookies                         │
│                                                          │
│  3. Browser's Security System Activated! 🚨              │
│     ┌────────────────────────────────┐                  │
│     │ Different Origin Detected!     │                  │
│     │ Complex Request Detected!      │                  │
│     │ → PREFLIGHT REQUIRED           │                  │
│     └────────────────────────────────┘                  │
│                                                          │
│  4. Browser Sends OPTIONS (Preflight):                   │
│     "Can evil-site.com DELETE on yourbank.com?"          │
│                                                          │
│  5. Bank Server Responds:                                │
│     "NO! Only yourbank.com allowed"                      │
│     "DELETE not allowed from other sites"                │
│                                                          │
│  6. Browser's Decision: 🛑 BLOCK!                        │
│     - DELETE never sent                                  │
│     - Cookies never attached                             │
│     - Attack prevented!                                  │
│                                                          │
│  7. You're Safe! 🎉                                      │
│     - Account still exists                               │
│     - Money still there                                  │
│     - You never knew about the attack!                   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Without Preflight (Nightmare Scenario)

```
┌──────────────────────────────────────────────────────────┐
│         What Would Happen WITHOUT Preflight             │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Evil site sends DELETE directly ❌                   │
│     (No preflight check)                                 │
│                                                          │
│  2. Browser attaches your bank cookies ❌                │
│     (You're logged in, cookies sent automatically)       │
│                                                          │
│  3. Bank receives DELETE request ❌                      │
│     (With valid session cookie!)                         │
│                                                          │
│  4. Bank thinks: "This is the real user" ❌              │
│     (Cookie is valid)                                    │
│                                                          │
│  5. Bank deletes account! 😱                             │
│                                                          │
│  6. Your money is gone! 😱                               │
│                                                          │
│  7. You had no idea it happened! 😱                      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Why Preflight Works

**Key Protection Mechanisms:**

1. **Browser checks BEFORE sending sensitive data**
```
Check first → Safe
Send first → Dangerous
```

2. **Server explicitly allows origins**
```
Not whitelisted = Blocked
```

3. **Complex requests require permission**
```
Simple request = Direct
Complex request = Preflight first
```

4. **User protected automatically**
```
No user action required
Browser does everything
```

### Other Attack Scenarios Prevented

**Scenario 2: Data Theft**
```
Evil site tries: GET /api/user/credit-cards
→ Preflight checks
→ Origin not allowed
→ Blocked! ✅
```

**Scenario 3: Money Transfer**
```
Evil site tries: POST /api/transfer
Body: { to: "attacker", amount: 10000 }
→ Preflight checks
→ Origin not allowed
→ Blocked! ✅
```

**Scenario 4: Password Change**
```
Evil site tries: PUT /api/password
Body: { new_password: "hacked123" }
→ Preflight checks
→ Method not allowed from this origin
→ Blocked! ✅
```

### The Security Layering

```
┌────────────────────────────────────────┐
│     Multiple Security Layers           │
├────────────────────────────────────────┤
│                                        │
│  Layer 1: CORS / Preflight             │
│  Layer 2: CSRF Tokens                  │
│  Layer 3: Authentication               │
│  Layer 4: Authorization                │
│  Layer 5: Rate Limiting                │
│                                        │
│  All work together for security! 🛡️   │
│                                        │
└────────────────────────────────────────┘
```

**Remember:**
- Preflight is **automatic** (browser does it)
- You're protected **without knowing**
- It's **not perfect** (needs other security too)
- But it's **essential first line of defense**!

This is why every backend developer **must** understand and properly implement preflight handling!
</details>

---

## Summary

**What We Learned Today:**

1. ✅ **OPTIONS Method**
   - 6th HTTP method (often forgotten!)
   - Used for preflight requests
   - Checks permissions before actual request

2. ✅ **Preflight Request**
   - Automatic browser security check
   - Sent before complex requests
   - Uses OPTIONS method

3. ✅ **Simple vs Complex Requests**
   - Simple: Standard headers, GET/POST
   - Complex: Custom headers, JSON, PUT/DELETE
   - Only complex triggers preflight

4. ✅ **Why Preflight Exists**
   - Security protection
   - Prevents cross-origin attacks
   - Gives servers control

5. ✅ **Handling OPTIONS in Go**
   - Check OPTIONS first
   - Return 200 OK
   - Set Allow-Headers
   - Set Allow-Methods

6. ✅ **Common CORS Errors**
   - Missing OPTIONS handling
   - Missing Allow-Headers
   - Wrong header syntax

**Key Takeaways:**

```
┌──────────────────────────────────────────────┐
│         Preflight Checklist                  │
├──────────────────────────────────────────────┤
│  1. Handle OPTIONS before other methods      │
│  2. Return 200 OK for OPTIONS                │
│  3. Set Access-Control-Allow-Headers         │
│  4. Include all custom headers in list       │
│  5. Use commas to separate headers           │
│  6. Test with real browser (not Postman!)    │
└──────────────────────────────────────────────┘
```

**Remember:**
- Preflight = Security guard checking ID
- Browser sends it automatically
- You must handle it in backend
- Without it, users are vulnerable

---

## What's Next?

**Chapter 43: Code Refactoring - Clean Architecture**

Next class we'll learn about:
- Why our current code is messy
- Software engineering principles
- Clean code organization
- Package structure
- Middleware patterns
- Handling large projects like Daraz/Amazon

**Why This Matters:**

Right now all our code is in one file - this is **terrible** for real projects!

```
Current (Bad):
main.go (everything in one file) 😱

Next (Good):
├── handlers/
│   ├── products.go
│   └── users.go
├── middleware/
│   └── cors.go
├── models/
│   └── product.go
└── main.go
```

Professional software engineering is about:
- **Maintainability** - Easy to change
- **Scalability** - Can grow to millions of lines
- **Team Work** - Multiple developers can work together
- **Clean Code** - Easy to understand

---

**Final Words:**

Today you learned about a **hidden security feature** that protects you every day without you knowing!

**Preflight requests** are like invisible bodyguards:
- You don't see them
- They work automatically
- They keep you safe

Now you understand:
- Why OPTIONS exists
- When preflight happens
- How to handle it properly
- Why it's critical for security

**You're becoming a REAL backend developer!** 🎉

Most developers never understand preflight - they just copy-paste CORS code. But you now understand the **WHY** behind it!

**Next time you see a CORS error, you'll know:**
- It's actually browser protecting you
- What headers to check
- How to fix it properly

**Keep learning! You're in the top 5%!** 🚀

---

**Remember:**
> "CORS errors aren't bugs - they're security features! Understanding preflight makes you a better developer and keeps users safe!" 🛡️

---
