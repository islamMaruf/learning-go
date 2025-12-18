# Chapter 45: Building Your First Middleware - Logger Implementation

## Table of Contents
- [Introduction](#introduction)
- [What We're Building](#what-were-building)
- [Project Structure](#project-structure)
- [Understanding Middleware Flow](#understanding-middleware-flow)
- [Creating the Middleware Folder](#creating-the-middleware-folder)
- [Building the Logger Middleware](#building-the-logger-middleware)
- [Understanding the Code Structure](#understanding-the-code-structure)
- [How http.Handler Works](#how-httphandler-works)
- [Understanding http.HandlerFunc](#understanding-httphandlerfunc)
- [Implementing the Logger Logic](#implementing-the-logger-logic)
- [Integrating Middleware with Routes](#integrating-middleware-with-routes)
- [Testing the Logger Middleware](#testing-the-logger-middleware)
- [Understanding Execution Order](#understanding-execution-order)
- [Code Cleanup and Organization](#code-cleanup-and-organization)
- [Practice Questions](#practice-questions)
- [Summary](#summary)
- [What's Next?](#whats-next)

---

## Introduction

In the previous chapter, we learned **what middleware is conceptually**. Now it's time to **actually build one**!

In this chapter, we'll create our **first real middleware** - a **Logger Middleware** that tracks:
- ✅ Which HTTP method was used (GET, POST, etc.)
- ✅ Which route was requested
- ✅ How long the request took to complete

This is one of the most common middleware patterns you'll see in production applications. Every major framework has a logger middleware, and now you'll build one from scratch!

**By the end of this chapter, you'll understand:**
- How to structure middleware in your project
- How `http.Handler` and `http.HandlerFunc` work together
- How to wrap handlers with middleware
- How execution flows through the middleware chain
- How to measure request duration
- How to organize clean middleware code

---

## What We're Building

Let's be clear about our goal. Right now, when requests come to our server, we have **no visibility**:

```
Client → Server → Handler → Response
         ❌ No logs
         ❌ No timing
         ❌ No visibility
```

**This is a problem!** How do we know:
- Which endpoints are being hit?
- Which requests are slow?
- What's happening in production?

**After building logger middleware:**

```
Client → Server → Logger Middleware → Handler → Response
                  ✅ Logs method
                  ✅ Logs path
                  ✅ Logs duration
                  ✅ Perfect visibility!
```

**Real-World Value:**
- 🏢 **Companies**: Can monitor API performance
- 🐛 **Debugging**: Can see which endpoints have issues
- 📊 **Analytics**: Can track which endpoints are popular
- ⚡ **Performance**: Can identify slow requests

---

## Project Structure

Before we start coding, let's organize our project properly. We'll have **multiple middleware** eventually, so we need a dedicated folder:

**Current structure:**
```
go_lang_tutorial/
├── main.go
├── cmd/
│   └── serve.go          (routes defined here)
└── handlers/
    └── test.go           (handlers/controllers)
```

**New structure with middleware:**
```
go_lang_tutorial/
├── main.go
├── cmd/
│   └── serve.go
├── handlers/
│   └── test.go
└── middleware/           ← NEW FOLDER
    └── logger.go         ← Logger middleware
```

**Why a separate folder?**
- ✅ **Organization**: All middleware in one place
- ✅ **Scalability**: Easy to add more middleware (auth, cors, etc.)
- ✅ **Maintainability**: Clear separation of concerns
- ✅ **Reusability**: Middleware can be imported anywhere

---

## Understanding Middleware Flow

Before we write code, let's visualize **exactly** what happens:

```
┌─────────────────────────────────────────────────────────┐
│                    REQUEST ARRIVES                      │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Global Router │
                    │  (mux.Handle) │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │    Logger     │ ← We're building this!
                    │  Middleware   │
                    └───────┬───────┘
                            │
                  ┌─────────┴─────────┐
                  │                   │
        Record start time    Print: "I am middleware"
                  │                   │
                  └─────────┬─────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   Handler/    │
                    │  Controller   │
                    └───────┬───────┘
                            │
                   Print: "I am handler"
                            │
                            ▼
                    ┌───────────────┐
                    │ Back to Logger│
                    │  Middleware   │
                    └───────┬───────┘
                            │
            Calculate duration & print logs
                            │
                            ▼
                    ┌───────────────┐
                    │   RESPONSE    │
                    └───────────────┘
```

**Key Points:**
1. **Middleware runs BEFORE handler** (records start time)
2. **Middleware calls next handler** (let it do its work)
3. **Middleware runs AFTER handler** (calculates duration)

This is called the **"Wrapper Pattern"** or **"Chain Pattern"**.

---

## Creating the Middleware Folder

Let's start by creating our folder structure:

**Step 1: Create the middleware folder**
```bash
mkdir middleware
```

**Step 2: Create logger.go file**
```bash
cd middleware
touch logger.go
```

**Step 3: Set up the package**
```go
package middleware

import (
	"log"
	"net/http"
	"time"
)
```

**Why these imports?**
- `log` - For printing logs to console
- `net/http` - For http.Handler and http.HandlerFunc
- `time` - For measuring request duration

---

## Building the Logger Middleware

Now let's build our logger middleware **step by step**.

### Step 1: Understanding the Function Signature

Every middleware has the same basic pattern:

```go
func Logger(next http.Handler) http.Handler {
	// Input: next handler (what runs after this middleware)
	// Output: http.Handler (what the router expects)
}
```

**Why this signature?**

Let's think about how we use middleware:

```go
mux.Handle("GET /products", middleware.Logger(handler))
                           ▲
                           │
                This must return http.Handler!
```

`mux.Handle()` expects:
1. **Pattern**: `"GET /products"`
2. **Handler**: `http.Handler` type

So our middleware **must return `http.Handler`**.

But we also need to know **what comes next** in the chain. That's why we take `next http.Handler` as input!

---

### Step 2: What is http.Handler?

`http.Handler` is an **interface**:

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

**Translation:** Anything with a `ServeHTTP(w, r)` method is a Handler.

---

### Step 3: What is http.HandlerFunc?

`http.HandlerFunc` is a **function type** that implements `http.Handler`:

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

**Translation:** Any function with signature `func(w, r)` can be converted to a Handler.

**This is how we create handlers:**

```go
handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	// Your handler code
})
```

Now `handler` is of type `http.Handler` and can be used in `mux.Handle()`!

---

### Step 4: Building the Logger Function

Here's our complete logger middleware:

```go
package middleware

import (
	"log"
	"net/http"
	"time"
)

func Logger(next http.Handler) http.Handler {
	// Return an http.Handler
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Record start time
		start := time.Now()
		
		// Log before handler
		log.Println("I am middleware - I run FIRST")
		
		// Call the next handler
		next.ServeHTTP(w, r)
		
		// Log after handler (this runs AFTER next handler completes)
		log.Println("I am middleware - I run LAST")
		
		// Calculate duration
		duration := time.Since(start)
		
		// Print detailed logs
		log.Println(r.Method, r.URL.Path, duration)
	})
}
```

---

## Understanding the Code Structure

Let's break down **every piece** of this middleware:

### Part 1: Function Signature

```go
func Logger(next http.Handler) http.Handler {
```

- **Input**: `next http.Handler` - The next thing in the chain
- **Output**: `http.Handler` - What we return to the router
- **Naming**: `next` is conventional (could be `nextHandler`, `handler`, etc.)

---

### Part 2: Returning http.Handler

```go
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	// Our middleware logic
})
```

**Why `http.HandlerFunc`?**

Because `mux.Handle()` expects `http.Handler`, but we're writing a **function**. 

`http.HandlerFunc` converts our function into a Handler type!

**Analogy:**
```
Your function: "I'm a chef" 👨‍🍳
http.HandlerFunc: "Here's your restaurant license" 📜
http.Handler: "Now you're officially recognized!" ✅
```

---

### Part 3: Recording Start Time

```go
start := time.Now()
```

**What is `time.Now()`?**

Returns the **current timestamp**:
```
2024-01-15 14:30:45.123456789 +0000 UTC
```

**Why record it?**

So we can calculate **how long the request took**:
```
End time - Start time = Duration
```

---

### Part 4: Calling the Next Handler

```go
next.ServeHTTP(w, r)
```

**This is THE MOST IMPORTANT LINE!**

This is where we say: "Okay, I'm done with my part. Let the **next handler** do its work."

**What is `next`?**

It's the handler we received as input! Remember:

```go
func Logger(next http.Handler) http.Handler {
                ▲
                │
            This is next!
```

**What is `.ServeHTTP(w, r)`?**

It's the method that **executes the handler**. Every `http.Handler` has this method.

**What are `w` and `r`?**

- `w` - ResponseWriter (for sending response)
- `r` - Request (contains method, URL, headers, body, etc.)

We **pass them forward** to the next handler!

---

### Part 5: Calculating Duration

```go
duration := time.Since(start)
```

**What is `time.Since()`?**

It calculates: **Now - Start Time**

If start was `14:30:45.000` and now is `14:30:45.500`, then:
```go
duration = 500 milliseconds
```

**Why after `next.ServeHTTP()`?**

Because we want to measure **how long the next handler took**!

```
Start: 14:30:45.000
│
├─ next.ServeHTTP() executes
│  (takes 500ms)
│
End: 14:30:45.500

Duration = End - Start = 500ms
```

---

### Part 6: Printing Logs

```go
log.Println(r.Method, r.URL.Path, duration)
```

**What does this print?**

```
GET /products 407.45µs
```

- `r.Method` - HTTP method (GET, POST, PUT, DELETE, etc.)
- `r.URL.Path` - The route path (`/products`, `/users`, etc.)
- `duration` - How long it took

**What is `µs`?**

Microseconds! 
- 1 second = 1,000 milliseconds (ms)
- 1 millisecond = 1,000 microseconds (µs)
- 407µs = 0.407ms = 0.000407 seconds

**Go is FAST!** 🚀

---

## How http.Handler Works

Let's understand the **type system** Go uses for handlers:

### The Handler Interface

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

**What this means:**

Any type that has a `ServeHTTP` method is automatically a `Handler`.

---

### Example 1: Struct Handler

```go
type MyHandler struct {
	Name string
}

func (h *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello from " + h.Name))
}

// Usage
handler := &MyHandler{Name: "Logger"}
mux.Handle("/", handler) // ✅ Works! MyHandler implements http.Handler
```

---

### Example 2: Function Handler (http.HandlerFunc)

```go
func myHandlerFunc(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}

// Convert function to Handler
handler := http.HandlerFunc(myHandlerFunc)
mux.Handle("/", handler) // ✅ Works! HandlerFunc implements http.Handler
```

---

### Example 3: Inline Handler (Most Common)

```go
mux.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}))
```

**This is what we do in our middleware!**

---

## Understanding http.HandlerFunc

`http.HandlerFunc` is a **type conversion function**. Let's see how it works:

### The Definition

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

### What This Does

1. **Defines a function type**: `HandlerFunc` is a type for functions with signature `func(w, r)`

2. **Adds ServeHTTP method**: The type has a method that **calls itself**

3. **Implements Handler interface**: Because it has `ServeHTTP`, it's a `Handler`!

### Visual Example

```go
// Step 1: You write a function
myFunc := func(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}

// Step 2: Convert to HandlerFunc
handlerFunc := http.HandlerFunc(myFunc)

// Step 3: Now it's an http.Handler!
var handler http.Handler = handlerFunc

// Step 4: Use it
mux.Handle("/", handler)
```

**Magic!** 🎩✨

---

## Implementing the Logger Logic

Now let's write the **complete logger middleware** with all the pieces together:

### Complete Code

```go
package middleware

import (
	"log"
	"net/http"
	"time"
)

// Logger middleware logs HTTP requests
func Logger(next http.Handler) http.Handler {
	// Return a handler that wraps the next handler
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// ============================================
		// BEFORE HANDLER EXECUTION
		// ============================================
		
		// Record the start time
		start := time.Now()
		
		// Log that middleware started
		log.Println("I am middleware - I run FIRST")
		
		// ============================================
		// EXECUTE THE NEXT HANDLER
		// ============================================
		
		// Call the next handler in the chain
		// This could be another middleware or the final handler
		next.ServeHTTP(w, r)
		
		// ============================================
		// AFTER HANDLER EXECUTION
		// ============================================
		
		// Log that middleware continues after handler
		log.Println("I am middleware - I run LAST")
		
		// Calculate how long the request took
		duration := time.Since(start)
		
		// Print detailed request information
		log.Println(r.Method, r.URL.Path, duration)
	})
}
```

### Understanding the Flow

```
1. Request arrives → Router matches route
2. Router calls Logger middleware
3. Logger records start time
4. Logger prints "I am middleware - I run FIRST"
5. Logger calls next.ServeHTTP() → Handler executes
6. Handler prints "I am handler - I run in MIDDLE"
7. Handler finishes, returns to Logger
8. Logger prints "I am middleware - I run LAST"
9. Logger calculates duration
10. Logger prints "GET /products 407µs"
11. Response sent to client
```

---

## Integrating Middleware with Routes

Now let's use our middleware in the actual application!

### Step 1: Our Handler (handlers/test.go)

```go
package handlers

import (
	"log"
	"net/http"
)

// Test handler
var Test = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	log.Println("I am handler - I run in MIDDLE")
	
	w.Write([]byte("Test endpoint"))
})
```

### Step 2: Using Middleware in Routes (cmd/serve.go)

**Before (without middleware):**
```go
package cmd

import (
	"net/http"
	"yourproject/handlers"
)

func Serve() {
	mux := http.NewServeMux()
	
	// Direct handler
	mux.Handle("GET /habib", handlers.Test)
	
	http.ListenAndServe(":8080", mux)
}
```

**After (with middleware):**
```go
package cmd

import (
	"net/http"
	"yourproject/handlers"
	"yourproject/middleware"
)

func Serve() {
	mux := http.NewServeMux()
	
	// Wrap handler with middleware
	mux.Handle("GET /habib", middleware.Logger(handlers.Test))
	
	http.ListenAndServe(":8080", mux)
}
```

**What changed?**

Instead of:
```go
mux.Handle("GET /habib", handlers.Test)
```

We now have:
```go
mux.Handle("GET /habib", middleware.Logger(handlers.Test))
                        ▲                   ▲
                        │                   │
                    Middleware         Next Handler
```

### How This Works

```go
middleware.Logger(handlers.Test)
```

1. **Calls** `Logger()` function
2. **Passes** `handlers.Test` as the `next` parameter
3. **Returns** a new `http.Handler` that wraps `handlers.Test`
4. **Result**: When request arrives, logger runs first, then handler

---

## Testing the Logger Middleware

Let's test our middleware and see it in action!

### Step 1: Start the Server

```bash
go run main.go
```

**Output:**
```
Server running on :8080
```

### Step 2: Make a Request

Using Postman, curl, or browser:

```bash
curl http://localhost:8080/habib
```

### Step 3: Check Server Logs

**In your terminal, you'll see:**
```
I am middleware - I run FIRST
I am handler - I run in MIDDLE
I am middleware - I run LAST
GET /habib 407.45µs
```

**Perfect!** This shows:
1. ✅ Middleware runs FIRST
2. ✅ Handler runs in MIDDLE
3. ✅ Middleware runs LAST (after handler)
4. ✅ Duration is logged

### Step 4: Make Multiple Requests

```bash
curl http://localhost:8080/habib
curl http://localhost:8080/habib
curl http://localhost:8080/habib
```

**Server logs:**
```
I am middleware - I run FIRST
I am handler - I run in MIDDLE
I am middleware - I run LAST
GET /habib 407.45µs

I am middleware - I run FIRST
I am handler - I run in MIDDLE
I am middleware - I run LAST
GET /habib 312.89µs

I am middleware - I run FIRST
I am handler - I run in MIDDLE
I am middleware - I run LAST
GET /habib 398.12µs
```

**Notice:** Each request goes through the same flow, and we can see the timing for each!

---

## Understanding Execution Order

Let's trace through **exactly** what happens, step by step:

### The Code

```go
// Router
mux.Handle("GET /habib", middleware.Logger(handlers.Test))
```

### Step-by-Step Execution

**When server starts:**

```
1. middleware.Logger(handlers.Test) is called
2. Returns a new handler that wraps handlers.Test
3. This wrapped handler is registered with the router
```

**When request arrives:**

```
1. Router receives: GET /habib
2. Router finds: The wrapped handler we registered
3. Router calls: wrappedHandler.ServeHTTP(w, r)

Inside wrapped handler:
4. start := time.Now()                          ← Record start
5. log.Println("I am middleware - FIRST")       ← Print
6. next.ServeHTTP(w, r)                         ← Call handlers.Test
   
   Inside handlers.Test:
   7. log.Println("I am handler - MIDDLE")      ← Print
   8. w.Write([]byte("Test endpoint"))          ← Send response
   9. Return to middleware                      ← Handler done

Back in middleware:
10. log.Println("I am middleware - LAST")       ← Print
11. duration := time.Since(start)               ← Calculate
12. log.Println(r.Method, r.URL.Path, duration) ← Print details
13. Return to router                            ← Middleware done

14. Router sends response to client              ← Complete!
```

### Call Stack Visualization

```
┌──────────────────────────────────────┐
│           Router                     │
│  ┌────────────────────────────────┐  │
│  │      Middleware Logger         │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │      Handler Test        │  │  │
│  │  │                          │  │  │
│  │  │  Print: "I am handler"   │  │  │
│  │  │  Send response           │  │  │
│  │  └──────────────────────────┘  │  │
│  │                                 │  │
│  │  Print: "I run LAST"            │  │
│  │  Print: "GET /habib 407µs"      │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Key insight:** Middleware is like a **sandwich**:
- Middleware BEFORE handler (top bread)
- Handler in the middle (filling)
- Middleware AFTER handler (bottom bread)

---

## Code Cleanup and Organization

Let's make our code **production-ready** by cleaning it up!

### Problem: Debug Logs Are Messy

Right now we have debug logs:
```go
log.Println("I am middleware - I run FIRST")
log.Println("I am handler - I run in MIDDLE")
log.Println("I am middleware - I run LAST")
```

These were great for **learning**, but in production we only want the **useful information**.

### Clean Version: Remove Debug Logs

**handlers/test.go:**
```go
package handlers

import (
	"net/http"
)

// Test handler
var Test = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	// Do actual work (no debug logs)
	w.Write([]byte("Test endpoint"))
})
```

**middleware/logger.go:**
```go
package middleware

import (
	"log"
	"net/http"
	"time"
)

// Logger middleware logs HTTP requests
func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Record start time
		start := time.Now()
		
		// Execute next handler
		next.ServeHTTP(w, r)
		
		// Log request details
		duration := time.Since(start)
		log.Printf("%s %s %v", r.Method, r.URL.Path, duration)
	})
}
```

**Now when you make a request:**
```
GET /habib 407.45µs
```

**Clean, professional, and informative!** ✨

---

### Better Logging Format

Let's make logs even more professional:

```go
// Logger middleware logs HTTP requests with detailed information
func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		
		// Execute handler
		next.ServeHTTP(w, r)
		
		// Calculate duration
		duration := time.Since(start)
		
		// Structured log format
		log.Printf(
			"[%s] %s %s %s",
			start.Format("2006-01-02 15:04:05"),
			r.Method,
			r.URL.Path,
			duration,
		)
	})
}
```

**Output:**
```
[2024-01-15 14:30:45] GET /habib 407.45µs
[2024-01-15 14:30:47] POST /products 1.234ms
[2024-01-15 14:30:50] GET /users 892.67µs
```

**Professional!** 🎯

---

### Production-Ready Logger

For production, you might want even more details:

```go
package middleware

import (
	"log"
	"net/http"
	"time"
)

// Logger middleware logs HTTP requests with full details
func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		
		// Log incoming request
		log.Printf("→ %s %s", r.Method, r.URL.Path)
		
		// Execute handler
		next.ServeHTTP(w, r)
		
		// Calculate duration
		duration := time.Since(start)
		
		// Log completed request
		log.Printf("← %s %s [%v]", r.Method, r.URL.Path, duration)
	})
}
```

**Output:**
```
→ GET /habib
← GET /habib [407.45µs]

→ POST /products
← POST /products [1.234ms]
```

**Shows incoming and outgoing - perfect for debugging!** 🔍

---

## Practice Questions

Test your understanding with these questions!

---

### Question 1: Understanding Middleware Signature

**Question:** Why does the Logger middleware have this signature?

```go
func Logger(next http.Handler) http.Handler
```

What would happen if we wrote it like this instead?

```go
func Logger() http.Handler
```

<details>
<summary>Click to see answer</summary>

**Answer:**

The signature `func Logger(next http.Handler) http.Handler` is essential because:

**1. We need to know what comes next:**
```go
func Logger(next http.Handler) http.Handler {
           ▲
           │
    This tells us: "What should I call after I'm done?"
```

Without `next`, the middleware wouldn't know what handler to execute!

**2. Middleware wraps handlers:**

Think of middleware like Russian nesting dolls:
```
Logger(CORS(Auth(Handler)))
```

Each middleware needs to know: "What's inside me?"

**3. If we wrote `func Logger() http.Handler`:**

```go
func Logger() http.Handler {
	return http.HandlerFunc(func(w, r) {
		start := time.Now()
		
		// ❌ What do we call here? We don't have "next"!
		// We're stuck! The chain is broken!
		
		duration := time.Since(start)
		log.Println(duration)
	})
}
```

**Without `next`, we can't:**
- Call the actual handler
- Continue the chain
- Complete the request

**Correct signature allows chaining:**
```go
mux.Handle("GET /", Logger(CORS(Auth(handler))))
                           ▲    ▲    ▲
                           │    │    │
                     Each knows the next!
```

**Analogy:**

Bad (no next):
```
You → [Secret door] → ???
      (Door locked, no key!)
```

Good (with next):
```
You → [Secret door] → Next Room → Final Room
      (Door has key to next room!)
```

</details>

---

### Question 2: Understanding time.Since()

**Question:** In this code:

```go
start := time.Now()
next.ServeHTTP(w, r)
duration := time.Since(start)
```

Why do we use `time.Since(start)` instead of `time.Now() - start`?

Try both approaches and explain the difference.

<details>
<summary>Click to see answer</summary>

**Answer:**

Both approaches work, but `time.Since()` is **better**! Here's why:

**Approach 1: Manual Subtraction (❌ Doesn't work!)**
```go
start := time.Now()
next.ServeHTTP(w, r)
duration := time.Now() - start  // ❌ ERROR!
```

**Error:**
```
invalid operation: time.Now() - start (mismatched types time.Time and time.Time)
```

**Why?** You can't subtract `time.Time` values directly in Go!

---

**Approach 2: Using Sub() (✅ Works)**
```go
start := time.Now()
next.ServeHTTP(w, r)
duration := time.Now().Sub(start)  // ✅ Returns time.Duration
```

**This works!** Returns: `407.45µs`

---

**Approach 3: Using Since() (✅ Best!)**
```go
start := time.Now()
next.ServeHTTP(w, r)
duration := time.Since(start)  // ✅ Cleaner!
```

**Why `time.Since()` is better:**

1. **More readable:**
   ```go
   time.Since(start)        // Clear: "time since start"
   time.Now().Sub(start)    // Confusing: "now subtract start"
   ```

2. **Cleaner code:**
   ```go
   time.Since(start)        // One function call
   time.Now().Sub(start)    // Two function calls
   ```

3. **Same result:**
   Both return `time.Duration` type:
   ```go
   407.45µs
   ```

**What is time.Duration?**

It's a type representing duration as **nanoseconds**:

```go
type Duration int64

const (
	Nanosecond  Duration = 1
	Microsecond          = 1000 * Nanosecond
	Millisecond          = 1000 * Microsecond
	Second               = 1000 * Millisecond
	Minute               = 60 * Second
	Hour                 = 60 * Minute
)
```

**Example conversions:**
```go
duration := 407450 * time.Nanosecond
fmt.Println(duration)                    // 407.45µs
fmt.Println(duration.Microseconds())     // 407
fmt.Println(duration.Milliseconds())     // 0
fmt.Println(duration.Seconds())          // 0.00040745
```

**Summary:**
- ❌ `time.Now() - start` → Doesn't work
- ✅ `time.Now().Sub(start)` → Works
- ✨ `time.Since(start)` → Best! (more readable)

</details>

---

### Question 3: Understanding http.HandlerFunc

**Question:** Explain what this code does and why it works:

```go
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	log.Println("Hello")
})
```

Why do we need `http.HandlerFunc` wrapper? Why can't we just return the function directly?

<details>
<summary>Click to see answer</summary>

**Answer:**

Let's break this down step by step!

---

**What we're trying to do:**

Return an `http.Handler` (interface) from our middleware.

---

**The Problem:**

```go
func Logger(next http.Handler) http.Handler {
	return func(w http.ResponseWriter, r *http.Request) {  // ❌ ERROR!
		log.Println("Hello")
	}
}
```

**Error:**
```
cannot use func literal (type func(http.ResponseWriter, *http.Request)) 
as type http.Handler in return argument:
	func literal does not implement http.Handler (missing ServeHTTP method)
```

**Why the error?**

Because `http.Handler` is an **interface**:

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

Our function doesn't have a `ServeHTTP` method, so it **doesn't implement** `http.Handler`!

---

**The Solution: http.HandlerFunc**

```go
func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {  // ✅
		log.Println("Hello")
	})
}
```

**What does `http.HandlerFunc` do?**

1. **It's a type** (not a function!):
   ```go
   type HandlerFunc func(ResponseWriter, *Request)
   ```

2. **It has a ServeHTTP method:**
   ```go
   func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
   	f(w, r)  // Just calls itself!
   }
   ```

3. **Therefore, it implements http.Handler!**

---

**Step-by-Step Magic:**

```go
// Step 1: You write a function
myFunc := func(w http.ResponseWriter, r *http.Request) {
	log.Println("Hello")
}

// Type: func(http.ResponseWriter, *http.Request)
// Has ServeHTTP? No ❌
// Is http.Handler? No ❌

// Step 2: Convert to HandlerFunc
handler := http.HandlerFunc(myFunc)

// Type: http.HandlerFunc
// Has ServeHTTP? Yes ✅ (added by HandlerFunc type)
// Is http.Handler? Yes ✅

// Step 3: Use it!
mux.Handle("/", handler)  // ✅ Works!
```

---

**Why does this work?**

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)  // Call the function
}
```

**Explanation:**

1. `HandlerFunc` is a **function type**
2. We add a `ServeHTTP` method to this type
3. `ServeHTTP` just calls the function itself
4. Now `HandlerFunc` implements `http.Handler`!

---

**Visual Analogy:**

**Without HandlerFunc:**
```
You: "I'm a chef"
Restaurant: "Sorry, we only hire people with licenses"
You: ❌ Rejected
```

**With HandlerFunc:**
```
You: "I'm a chef"
HandlerFunc: "Here's a license wrapper"
You: "I'm a chef with a license"
Restaurant: "Welcome aboard!"
You: ✅ Hired
```

---

**Real-World Example:**

```go
// ❌ This doesn't work
mux.Handle("/", func(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
})

// ✅ This works!
mux.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello"))
}))
```

---

**Summary:**

| Without HandlerFunc | With HandlerFunc |
|---------------------|------------------|
| Just a function | Function + ServeHTTP method |
| Not an http.Handler | Is an http.Handler ✅ |
| Can't use in mux.Handle() | Can use in mux.Handle() ✅ |
| ❌ Type mismatch error | ✅ Works perfectly |

**The wrapper gives your function the "license" to be a Handler!**

</details>

---

### Question 4: Execution Order Challenge

**Question:** Given this code:

```go
func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("A")
		next.ServeHTTP(w, r)
		log.Println("B")
	})
}

func CORS(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("C")
		next.ServeHTTP(w, r)
		log.Println("D")
	})
}

func Handler(w http.ResponseWriter, r *http.Request) {
	log.Println("E")
}

// Setup
mux.Handle("GET /", Logger(CORS(http.HandlerFunc(Handler))))
```

**What order will A, B, C, D, E print in?**

Draw the execution flow.

<details>
<summary>Click to see answer</summary>

**Answer:**

**Output:**
```
A
C
E
D
B
```

---

**Execution Flow:**

```
Request arrives
│
▼
Logger middleware
├─ Print "A"
├─ Call next.ServeHTTP() ───┐
│                           │
│                           ▼
│                     CORS middleware
│                     ├─ Print "C"
│                     ├─ Call next.ServeHTTP() ───┐
│                     │                           │
│                     │                           ▼
│                     │                     Handler
│                     │                     ├─ Print "E"
│                     │                     └─ Return ──┐
│                     │                                 │
│                     ◄───────────────────────────────┘
│                     ├─ Print "D"
│                     └─ Return ──┐
│                                 │
◄───────────────────────────────┘
├─ Print "B"
└─ Return

Response sent
```

---

**Call Stack Visualization:**

```
┌────────────────────────────────┐
│ Logger                         │
│  Print "A"                     │ ← Step 1
│  ┌──────────────────────────┐  │
│  │ CORS                     │  │
│  │  Print "C"               │  │ ← Step 2
│  │  ┌────────────────────┐  │  │
│  │  │ Handler            │  │  │
│  │  │  Print "E"         │  │  │ ← Step 3
│  │  └────────────────────┘  │  │
│  │  Print "D"               │  │ ← Step 4
│  └──────────────────────────┘  │
│  Print "B"                     │ ← Step 5
└────────────────────────────────┘
```

---

**Why this order?**

**1. Logger runs first (outermost)**
```go
Logger(CORS(Handler))
▲
│
Outermost → Runs first
```

**2. Logger prints "A", then calls next**
```go
log.Println("A")           // ← Print A
next.ServeHTTP(w, r)       // ← Call CORS
```

**3. CORS runs (middle)**
```go
log.Println("C")           // ← Print C
next.ServeHTTP(w, r)       // ← Call Handler
```

**4. Handler runs (innermost)**
```go
log.Println("E")           // ← Print E
// Handler completes, returns to CORS
```

**5. Back to CORS (after handler)**
```go
next.ServeHTTP(w, r)       // ← Handler finished
log.Println("D")           // ← Print D
// CORS completes, returns to Logger
```

**6. Back to Logger (after CORS)**
```go
next.ServeHTTP(w, r)       // ← CORS finished
log.Println("B")           // ← Print B
// Logger completes, returns to router
```

---

**Middleware is like Russian Nesting Dolls:**

```
┌─────────────────────────────┐
│ Logger                      │ ← Outermost
│  ┌───────────────────────┐  │
│  │ CORS                  │  │ ← Middle
│  │  ┌─────────────────┐  │  │
│  │  │ Handler         │  │  │ ← Innermost
│  │  │                 │  │  │
│  │  └─────────────────┘  │  │
│  └───────────────────────┘  │
└─────────────────────────────┘

Flow: Outside → Inside → Outside
      A → C → E → D → B
```

---

**Real-World Example:**

Imagine entering a building:

```
1. Security checkpoint (Logger)
   - "Check your ID" (A)
   - Let you through...
   
2. Reception desk (CORS)
   - "Sign the visitor log" (C)
   - Let you through...
   
3. Office (Handler)
   - "Do your work" (E)
   - Leave...
   
4. Back to Reception
   - "Sign out" (D)
   - Leave...
   
5. Back to Security
   - "Exit logged" (B)
   - Done!
```

---

**Key Insight:**

Middleware wraps **around** handlers like layers of an onion:

```
Enter  → Logger → CORS → Handler → CORS → Logger → Exit
         (A)      (C)     (E)      (D)     (B)
         
         ─────────────→  ←──────────────
         Going in        Coming back
```

**You go IN through all layers, then come BACK OUT through the same layers!**

</details>

---

### Question 5: Build a Timing Middleware

**Challenge:** Create a timing middleware that:

1. Measures how long each request takes
2. Prints a WARNING if request takes > 100ms
3. Prints SUCCESS if request takes ≤ 100ms
4. Includes the route path and duration

**Expected output:**
```
✅ SUCCESS: GET /fast completed in 45.23µs
⚠️ WARNING: POST /slow completed in 234.56ms
```

**Bonus:** Add color to the output!

<details>
<summary>Click to see answer</summary>

**Answer:**

Here's a complete timing middleware with warnings:

```go
package middleware

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

// Timing middleware measures request duration and warns about slow requests
func Timing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Record start time
		start := time.Now()
		
		// Execute next handler
		next.ServeHTTP(w, r)
		
		// Calculate duration
		duration := time.Since(start)
		
		// Define threshold for slow requests
		threshold := 100 * time.Millisecond
		
		// Check if request is slow
		if duration > threshold {
			// Slow request - print WARNING
			log.Printf("⚠️  WARNING: %s %s completed in %v (SLOW!)", 
				r.Method, 
				r.URL.Path, 
				duration,
			)
		} else {
			// Fast request - print SUCCESS
			log.Printf("✅ SUCCESS: %s %s completed in %v", 
				r.Method, 
				r.URL.Path, 
				duration,
			)
		}
	})
}
```

---

**Usage:**

```go
// In cmd/serve.go
mux.Handle("GET /fast", middleware.Timing(handlers.FastHandler))
mux.Handle("POST /slow", middleware.Timing(handlers.SlowHandler))
```

---

**Test Handlers:**

```go
// handlers/test.go

// Fast handler (completes quickly)
var FastHandler = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Fast response"))
})

// Slow handler (takes 200ms)
var SlowHandler = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	time.Sleep(200 * time.Millisecond)  // Simulate slow operation
	w.Write([]byte("Slow response"))
})
```

---

**Output:**

```
✅ SUCCESS: GET /fast completed in 45.23µs
⚠️ WARNING: POST /slow completed in 201.45ms (SLOW!)
✅ SUCCESS: GET /fast completed in 38.67µs
⚠️ WARNING: POST /slow completed in 200.12ms (SLOW!)
```

---

**Bonus: With Colors (using ANSI codes)**

```go
package middleware

import (
	"log"
	"net/http"
	"time"
)

// ANSI color codes
const (
	ColorReset  = "\033[0m"
	ColorGreen  = "\033[32m"
	ColorYellow = "\033[33m"
	ColorRed    = "\033[31m"
)

// Timing middleware with colored output
func Timing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		duration := time.Since(start)
		
		threshold := 100 * time.Millisecond
		
		if duration > threshold {
			// Red warning
			log.Printf("%s⚠️  WARNING: %s %s completed in %v (SLOW!)%s", 
				ColorRed,
				r.Method, 
				r.URL.Path, 
				duration,
				ColorReset,
			)
		} else {
			// Green success
			log.Printf("%s✅ SUCCESS: %s %s completed in %v%s", 
				ColorGreen,
				r.Method, 
				r.URL.Path, 
				duration,
				ColorReset,
			)
		}
	})
}
```

---

**Even Better: Multiple Thresholds**

```go
package middleware

import (
	"log"
	"net/http"
	"time"
)

const (
	ColorReset  = "\033[0m"
	ColorGreen  = "\033[32m"
	ColorYellow = "\033[33m"
	ColorRed    = "\033[31m"
)

// Timing middleware with multiple thresholds
func Timing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		duration := time.Since(start)
		
		switch {
		case duration < 50*time.Millisecond:
			// Fast (< 50ms) - Green
			log.Printf("%s🚀 FAST: %s %s completed in %v%s", 
				ColorGreen, r.Method, r.URL.Path, duration, ColorReset)
			
		case duration < 100*time.Millisecond:
			// Normal (50-100ms) - Yellow
			log.Printf("%s✅ NORMAL: %s %s completed in %v%s", 
				ColorYellow, r.Method, r.URL.Path, duration, ColorReset)
			
		case duration < 500*time.Millisecond:
			// Slow (100-500ms) - Orange/Yellow
			log.Printf("%s⚠️  SLOW: %s %s completed in %v%s", 
				ColorYellow, r.Method, r.URL.Path, duration, ColorReset)
			
		default:
			// Very slow (> 500ms) - Red
			log.Printf("%s🔥 CRITICAL: %s %s completed in %v (VERY SLOW!)%s", 
				ColorRed, r.Method, r.URL.Path, duration, ColorReset)
		}
	})
}
```

**Output:**
```
🚀 FAST: GET /api completed in 12.34µs
✅ NORMAL: GET /products completed in 67.89ms
⚠️  SLOW: POST /upload completed in 234.56ms
🔥 CRITICAL: POST /process completed in 1.234s (VERY SLOW!)
```

---

**Production-Ready Version:**

```go
package middleware

import (
	"log"
	"net/http"
	"time"
)

// TimingConfig holds configuration for timing middleware
type TimingConfig struct {
	FastThreshold     time.Duration
	NormalThreshold   time.Duration
	SlowThreshold     time.Duration
	EnableColors      bool
	EnableEmojis      bool
}

// DefaultTimingConfig returns sensible defaults
func DefaultTimingConfig() TimingConfig {
	return TimingConfig{
		FastThreshold:   50 * time.Millisecond,
		NormalThreshold: 100 * time.Millisecond,
		SlowThreshold:   500 * time.Millisecond,
		EnableColors:    true,
		EnableEmojis:    true,
	}
}

// TimingWithConfig creates timing middleware with custom config
func TimingWithConfig(config TimingConfig) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			next.ServeHTTP(w, r)
			duration := time.Since(start)
			
			var prefix, color string
			
			switch {
			case duration < config.FastThreshold:
				prefix = "FAST"
				color = "\033[32m"  // Green
			case duration < config.NormalThreshold:
				prefix = "NORMAL"
				color = "\033[33m"  // Yellow
			case duration < config.SlowThreshold:
				prefix = "SLOW"
				color = "\033[33m"  // Yellow
			default:
				prefix = "CRITICAL"
				color = "\033[31m"  // Red
			}
			
			if !config.EnableColors {
				color = ""
			}
			
			log.Printf("%s%s: %s %s %v\033[0m", 
				color, prefix, r.Method, r.URL.Path, duration)
		})
	}
}
```

**Usage:**
```go
// Default config
mux.Handle("GET /", middleware.TimingWithConfig(middleware.DefaultTimingConfig())(handler))

// Custom config
customConfig := middleware.TimingConfig{
	FastThreshold:   10 * time.Millisecond,
	NormalThreshold: 50 * time.Millisecond,
	SlowThreshold:   200 * time.Millisecond,
	EnableColors:    true,
}
mux.Handle("GET /", middleware.TimingWithConfig(customConfig)(handler))
```

**This is production-ready!** 🚀

</details>

---

## Summary

Congratulations! You've built your first middleware from scratch! 🎉

**What we learned:**

1. **✅ Middleware Structure**
   - Middleware folder organization
   - Function signature: `func(next http.Handler) http.Handler`
   - Input: next handler to call
   - Output: wrapped handler

2. **✅ http.Handler vs http.HandlerFunc**
   - `http.Handler` is an interface with `ServeHTTP` method
   - `http.HandlerFunc` converts functions to handlers
   - Why we need the conversion

3. **✅ Middleware Execution Flow**
   - Code before `next.ServeHTTP()` runs BEFORE handler
   - Code after `next.ServeHTTP()` runs AFTER handler
   - Middleware wraps around handlers like layers

4. **✅ Timing Measurements**
   - `time.Now()` captures current time
   - `time.Since()` calculates duration
   - Logging request details

5. **✅ Integration Patterns**
   - Wrapping handlers: `middleware.Logger(handler)`
   - Can chain multiple middleware
   - Clean, professional code organization

**Key Takeaways:**

```go
// Middleware Pattern
func Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// BEFORE handler
		doSomethingBefore()
		
		// Call next handler
		next.ServeHTTP(w, r)
		
		// AFTER handler
		doSomethingAfter()
	})
}
```

**You can now:**
- ✅ Build custom middleware
- ✅ Measure request performance
- ✅ Log HTTP requests professionally
- ✅ Organize middleware in projects
- ✅ Understand the wrapper pattern
- ✅ Create production-ready code

---

## What's Next?

In the next chapters, we'll explore:

**Chapter 46: Multiple Middleware Chaining**
- Combining multiple middleware
- Execution order with multiple middleware
- Creating middleware chains
- Helper functions for chaining

**Chapter 47: Auth Middleware**
- Checking authentication
- Validating JWT tokens
- Stopping the chain (not calling next)
- Sending error responses from middleware

**Chapter 48: Request/Response Middleware**
- Modifying requests before handlers
- Modifying responses after handlers
- Adding headers
- Request validation

**Chapter 49: Error Handling Middleware**
- Recovering from panics
- Logging errors
- Sending error responses
- Structured error handling

**Chapter 50: Production Middleware Patterns**
- Rate limiting
- Request ID tracking
- Compression
- Security headers

---

**You're becoming a professional Go developer!** 🚀

Keep building, keep learning, and remember: **Every expert was once a beginner who didn't give up!**

---

**Pro Tip:** In the next chapter, we'll learn how to chain **multiple middleware** together. Get ready to build Logger + CORS + Auth + Rate Limiter all working together! 🔥
