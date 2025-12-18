# Chapter 47: Real Project Structure - Configuration Management

## Table of Contents
- [Introduction](#introduction)
- [The Problem with Hardcoded Values](#the-problem-with-hardcoded-values)
- [Creating the Config Package](#creating-the-config-package)
- [Understanding Environment Variables](#understanding-environment-variables)
- [The .env File](#the-env-file)
- [Loading Environment Variables with godotenv](#loading-environment-variables-with-godotenv)
- [Reading Environment Variables](#reading-environment-variables)
- [Type Conversion: String to Int](#type-conversion-string-to-int)
- [Type Conversion: Int to String](#type-conversion-int-to-string)
- [Global Configuration State](#global-configuration-state)
- [Using Configuration in Serve](#using-configuration-in-serve)
- [Complete Implementation](#complete-implementation)
- [Practice Questions](#practice-questions)
- [Summary](#summary)

---

## Introduction

Welcome to **real project structure**! 🏗️

Until now, we've been hardcoding values like port numbers directly in our code. This works for learning, but in **real production applications**, this is a disaster waiting to happen.

**What we'll learn:**
- ✅ Why hardcoding is bad
- ✅ Configuration management with .env files
- ✅ Creating a config package
- ✅ Loading environment variables
- ✅ Type conversions (string ↔ int)
- ✅ Global configuration state

**By the end:** You'll have a production-ready configuration system!

---

## The Problem with Hardcoded Values

**Current code (BAD ❌):**

```go
// cmd/serve.go
func Serve() {
    mux := http.NewServeMux()
    
    // Routes...
    
    // Hardcoded port! ❌
    http.ListenAndServe(":8080", handler)
}
```

**Why is this bad?**

1. **❌ Can't change without editing code**
   - Want to run on port 3000? Edit code and recompile
   - Want to run on port 8080? Edit code and recompile

2. **❌ Different environments need different values**
   - Development: Port 3000
   - Staging: Port 8080
   - Production: Port 80
   - Each needs code changes!

3. **❌ Secrets in code**
   - Database passwords
   - API keys
   - All visible in source code = security risk!

4. **❌ Team conflicts**
   - Developer A likes port 3000
   - Developer B likes port 8080
   - Git conflicts every time!

**Real-world scenario:**

```
You: "Deploy to production!"
DevOps: "What port?"
You: "Uhhh... it's hardcoded to 8080"
DevOps: "We use 80 in production"
You: "Let me change the code and redeploy..."
DevOps: "🤦"
```

**The solution: Configuration files!** ✅

---

## Creating the Config Package

**Step 1: Create config folder**

```
project/
├── main.go
├── cmd/
│   └── serve.go
├── config/              ← NEW FOLDER
│   └── config.go        ← Configuration logic
├── middleware/
└── handlers/
```

**Step 2: Create config struct (config/config.go)**

```go
package config

// Config holds all application configuration
type Config struct {
    Version     string
    ServiceName string
    HTTPPort    int64
}
```

**Why a struct?**

- ✅ **Type safety** - Port must be int64, not string
- ✅ **All config in one place** - Easy to see what's configurable
- ✅ **Easy to pass around** - Pass entire config to functions
- ✅ **Documentation** - Struct fields document what config exists

---

## Understanding Environment Variables

**What are environment variables?**

Every process (running program) is like a **virtual computer**. This virtual computer has its own set of variables called **environment variables**.

**Try it yourself:**

```bash
# In your terminal
env
```

**You'll see:**

```
PWD=/Users/habib/Desktop/go_project
USER=habib
HOME=/Users/habib
PATH=/usr/bin:/bin:/usr/local/bin
...
```

**These are environment variables!**

**In Go, we can read them:**

```go
import "os"

func main() {
    user := os.Getenv("USER")
    fmt.Println(user)  // Prints your username
}
```

**The process environment:**

```
┌─────────────────────────────────┐
│      Your Go Program            │
│      (Process)                  │
│                                 │
│  Environment Variables:         │
│  ├─ USER=habib                  │
│  ├─ PWD=/home/habib/project     │
│  ├─ PORT=3000                   │
│  └─ DB_PASSWORD=secret123       │
│                                 │
└─────────────────────────────────┘
```

---

## The .env File

**Convention:** Store configuration in a `.env` file

**Create .env in project root:**

```env
VERSION=1.0.0
SERVICE_NAME=ecommerce
HTTP_PORT=3000
```

**Naming conventions:**

```env
# ✅ CORRECT (all uppercase, underscores)
VERSION=1.0.0
SERVICE_NAME=ecommerce
HTTP_PORT=3000
DATABASE_URL=postgres://...

# ❌ WRONG
version=1.0.0          # lowercase
ServiceName=ecommerce  # camelCase
http-port=3000         # hyphens
```

**Why this convention?**

- ✅ **Standard across all languages** - Python, Node.js, Ruby all use UPPER_CASE
- ✅ **Easy to distinguish** - Environment variables stand out in code
- ✅ **Shell compatibility** - Works well with Unix shells

**Important: Add to .gitignore!**

```gitignore
# .gitignore
.env
```

**Why?** `.env` contains secrets! Never commit to Git!

---

## Loading Environment Variables with godotenv

**The problem:** `.env` file exists, but Go doesn't automatically load it into the process environment.

**The solution:** Use `godotenv` library

**Step 1: Install godotenv**

```bash
go get github.com/joho/godotenv
```

**What happens:**

```
# Terminal output
go: downloading github.com/joho/godotenv v1.5.1
go: added github.com/joho/godotenv v1.5.1
```

**Your go.mod updates:**

```go
// go.mod
module ecommerce

go 1.22

require github.com/joho/godotenv v1.5.1
```

**Step 2: Import and use**

```go
package config

import (
    "github.com/joho/godotenv"
    "log"
    "os"
)

func LoadConfig() Config {
    // Load .env file into process environment
    err := godotenv.Load()
    if err != nil {
        log.Fatal("Failed to load .env file:", err)
    }
    
    // Now we can read environment variables!
    version := os.Getenv("VERSION")
    serviceName := os.Getenv("SERVICE_NAME")
    httpPort := os.Getenv("HTTP_PORT")
    
    // ... convert and return config
}
```

**What godotenv does:**

```
┌────────────────────┐
│   .env File        │
│                    │
│  VERSION=1.0.0     │
│  SERVICE_NAME=app  │
│  HTTP_PORT=3000    │
└─────────┬──────────┘
          │
          │ godotenv.Load()
          │
          ▼
┌────────────────────┐
│  Process Env       │
│                    │
│  VERSION=1.0.0     │ ← Now available via os.Getenv()
│  SERVICE_NAME=app  │
│  HTTP_PORT=3000    │
└────────────────────┘
```

---

## Reading Environment Variables

**Using os.Getenv():**

```go
package config

import (
    "fmt"
    "log"
    "os"
    "github.com/joho/godotenv"
)

func LoadConfig() Config {
    // Load .env file
    err := godotenv.Load()
    if err != nil {
        log.Fatal("Failed to load .env file:", err)
        os.Exit(1)
    }
    
    // Read environment variables
    version := os.Getenv("VERSION")
    if version == "" {
        fmt.Println("VERSION is required")
        os.Exit(1)
    }
    
    serviceName := os.Getenv("SERVICE_NAME")
    if serviceName == "" {
        fmt.Println("SERVICE_NAME is required")
        os.Exit(1)
    }
    
    httpPort := os.Getenv("HTTP_PORT")
    if httpPort == "" {
        fmt.Println("HTTP_PORT is required")
        os.Exit(1)
    }
    
    // ... convert httpPort to int64
}
```

**Understanding os.Exit():**

```go
os.Exit(0)  // Success
os.Exit(1)  // Error occurred
```

**Exit codes:**

| Code | Meaning |
|------|---------|
| 0 | Success (no error) |
| 1 | General error |
| 2 | Misuse of command |
| ... | ... |
| 125 | Maximum portable exit code |

**When to use os.Exit(1):**

- ✅ Missing required configuration
- ✅ Failed to connect to database
- ✅ Invalid configuration values
- ✅ Application cannot start

**Don't confuse with return:**

```go
// ❌ return only exits the function
if err != nil {
    fmt.Println("Error!")
    return  // Function exits, program continues
}

// ✅ os.Exit() terminates entire program
if err != nil {
    fmt.Println("Error!")
    os.Exit(1)  // Program terminates immediately
}
```

---

## Type Conversion: String to Int

**Problem:** `os.Getenv()` returns string, but we need int64 for port number.

```go
httpPort := os.Getenv("HTTP_PORT")  // "3000" (string)
// But Config.HTTPPort is int64!
```

**Solution: strconv.ParseInt()**

```go
import "strconv"

// Convert string to int64
port, err := strconv.ParseInt(httpPort, 10, 64)
if err != nil {
    fmt.Println("HTTP_PORT must be a number")
    os.Exit(1)
}
```

**Understanding ParseInt parameters:**

```go
strconv.ParseInt(s string, base int, bitSize int)
                 ▲        ▲         ▲
                 │        │         │
                 │        │         └─ Bit size: 8, 16, 32, 64
                 │        └─────────── Number base (2, 8, 10, 16)
                 └──────────────────── String to parse
```

**Parameter 1: s (string to parse)**

```go
"3000"   // Decimal number
"FF"     // Could be hexadecimal
"1010"   // Could be binary
```

**Parameter 2: base (number system)**

```go
base 2   // Binary:      0, 1
base 8   // Octal:       0-7
base 10  // Decimal:     0-9  ← We use this!
base 16  // Hexadecimal: 0-9, A-F
```

**Parameter 3: bitSize (int size)**

```go
bitSize 8   // int8   (-128 to 127)
bitSize 16  // int16  (-32,768 to 32,767)
bitSize 32  // int32  (-2 billion to 2 billion)
bitSize 64  // int64  (-9 quintillion to 9 quintillion) ← We use this!
```

**Complete example:**

```go
// Convert "3000" to int64
httpPort := os.Getenv("HTTP_PORT")  // "3000"
port, err := strconv.ParseInt(httpPort, 10, 64)
if err != nil {
    fmt.Println("HTTP_PORT must be a number")
    os.Exit(1)
}
// port is now int64(3000)
```

**Why check for errors?**

```go
// If HTTP_PORT=habib
port, err := strconv.ParseInt("habib", 10, 64)
// err != nil (can't convert "habib" to number!)
```

---

## Type Conversion: Int to String

**Problem:** We have int64 port, but `http.ListenAndServe()` needs string address like `":3000"`.

**Solution: Type casting or strconv**

### Method 1: Type Casting (WRONG for int64!)

```go
// ❌ This is WRONG for int64!
port := int64(3000)
portStr := string(port)  // This converts to rune/character, not string!
fmt.Println(portStr)     // Prints weird characters!
```

**Why this fails:**

`string()` converts int to **rune** (character), not decimal string:

```go
string(65)   // "A" (ASCII 65)
string(3000) // Some weird character
```

### Method 2: strconv.FormatInt() (CORRECT ✅)

```go
port := int64(3000)
portStr := strconv.FormatInt(port, 10)
fmt.Println(portStr)  // "3000" ✅
```

### Method 3: strconv.Itoa() (for int only)

```go
// ✅ If you have regular int
port := 3000  // int (not int64)
portStr := strconv.Itoa(port)
fmt.Println(portStr)  // "3000"

// ❌ But won't work for int64
port64 := int64(3000)
portStr := strconv.Itoa(port64)  // ERROR: cannot use int64 as int
```

### Method 4: fmt.Sprintf() (easiest!)

```go
// ✅ Works for any type
port := int64(3000)
portStr := fmt.Sprintf("%d", port)
fmt.Println(portStr)  // "3000"
```

**Recommended approach:**

```go
// Build complete address
port := int64(3000)
address := fmt.Sprintf(":%d", port)  // ":3000"
```

---

## Global Configuration State

**Problem:** Config is loaded in `LoadConfig()`, but disappears when function returns (stack frame destroyed).

**Stack frame lifecycle:**

```
┌─────────────────────────┐
│  LoadConfig() called    │
│  ┌───────────────────┐  │
│  │ Stack Frame       │  │
│  │                   │  │
│  │ cnf := Config{    │  │
│  │   Version: "1.0"  │  │
│  │ }                 │  │
│  └───────────────────┘  │
│                         │
│  return cnf             │
└─────────────────────────┘
          │
          ▼
Function ends → Stack frame destroyed → cnf lost! ❌
```

**Solution: Global variable (data segment)**

```go
package config

var configurations Config  // Global variable (lowercase = private)

func LoadConfig() Config {
    // Load .env file...
    // Read variables...
    
    cnf := Config{
        Version:     version,
        ServiceName: serviceName,
        HTTPPort:    port,
    }
    
    // Store in global variable
    configurations = cnf
    
    return cnf
}

func GetConfig() Config {
    return configurations
}
```

**Memory segments:**

```
┌──────────────────────────────────┐
│        Code Segment              │  (Your functions)
├──────────────────────────────────┤
│        Data Segment              │  ← Global variables here
│  configurations Config           │     (Persists throughout program)
├──────────────────────────────────┤
│        Stack                     │  (Function local variables)
│  (Grows/shrinks with calls)      │
├──────────────────────────────────┤
│        Heap                      │  (Dynamic allocations)
└──────────────────────────────────┘
```

**Why global variable works:**

- ✅ **Persists** - Lives entire program lifetime
- ✅ **Accessible** - Can be read from any function
- ✅ **Single source of truth** - One config for entire app

**Why lowercase?**

```go
var configurations Config  // ✅ Lowercase = private to package

// ❌ If uppercase (public):
var Configurations Config  // Anyone can modify!

// Bad code in other package:
config.Configurations.HTTPPort = 9999  // Changed port!
```

**Public getter, private storage:**

```go
// Private storage
var configurations Config  // Can't access from outside

// Public getter
func GetConfig() Config {  // Can access from outside
    return configurations  // Returns copy, can't modify original
}
```

---

## Using Configuration in Serve

**Update cmd/serve.go:**

```go
package cmd

import (
    "fmt"
    "log"
    "net/http"
    "ecommerce/config"
    "ecommerce/middleware"
    "ecommerce/routes"
)

func Serve() {
    // Load configuration
    cnf := config.GetConfig()
    
    // Create middleware manager
    manager := middleware.NewManager()
    manager.Use(middleware.CorsWithPreflight)
    manager.Use(middleware.Logger)
    
    // Create router
    mux := http.NewServeMux()
    
    // Initialize routes
    routes.InitRoutes(mux, manager)
    
    // Apply global middleware
    handler := manager.With(mux)
    
    // Build address from config
    address := fmt.Sprintf(":%d", cnf.HTTPPort)
    
    // Log server start
    log.Printf("Server running on port %d", cnf.HTTPPort)
    
    // Start server
    err := http.ListenAndServe(address, handler)
    if err != nil {
        log.Fatal("Server failed:", err)
        os.Exit(1)
    }
}
```

**Update main.go:**

```go
package main

import (
    "ecommerce/cmd"
    "ecommerce/config"
)

func main() {
    // Load configuration first!
    config.LoadConfig()
    
    // Start server
    cmd.Serve()
}
```

---

## Complete Implementation

### File: .env

```env
VERSION=1.0.0
SERVICE_NAME=ecommerce
HTTP_PORT=3000
```

### File: config/config.go

```go
package config

import (
    "fmt"
    "log"
    "os"
    "strconv"
    "github.com/joho/godotenv"
)

// Config holds application configuration
type Config struct {
    Version     string
    ServiceName string
    HTTPPort    int64
}

// Global configuration storage (private)
var configurations Config

// LoadConfig loads configuration from .env file
func LoadConfig() Config {
    // Load .env file into process environment
    err := godotenv.Load()
    if err != nil {
        log.Fatal("Failed to load .env file:", err)
        os.Exit(1)
    }
    
    // Read VERSION
    version := os.Getenv("VERSION")
    if version == "" {
        fmt.Println("VERSION is required")
        os.Exit(1)
    }
    
    // Read SERVICE_NAME
    serviceName := os.Getenv("SERVICE_NAME")
    if serviceName == "" {
        fmt.Println("SERVICE_NAME is required")
        os.Exit(1)
    }
    
    // Read HTTP_PORT
    httpPort := os.Getenv("HTTP_PORT")
    if httpPort == "" {
        fmt.Println("HTTP_PORT is required")
        os.Exit(1)
    }
    
    // Convert port to int64
    port, err := strconv.ParseInt(httpPort, 10, 64)
    if err != nil {
        fmt.Println("HTTP_PORT must be a number")
        os.Exit(1)
    }
    
    // Create config
    cnf := Config{
        Version:     version,
        ServiceName: serviceName,
        HTTPPort:    port,
    }
    
    // Store globally
    configurations = cnf
    
    // Also return for immediate use
    return cnf
}

// GetConfig returns current configuration
func GetConfig() Config {
    return configurations
}
```

### File: cmd/serve.go

```go
package cmd

import (
    "fmt"
    "log"
    "net/http"
    "os"
    "ecommerce/config"
    "ecommerce/middleware"
    "ecommerce/routes"
)

func Serve() {
    // Get configuration
    cnf := config.GetConfig()
    
    // Log startup info
    log.Printf("Starting %s v%s", cnf.ServiceName, cnf.Version)
    
    // Create middleware manager
    manager := middleware.NewManager()
    manager.Use(middleware.CorsWithPreflight)
    manager.Use(middleware.Logger)
    
    // Create router
    mux := http.NewServeMux()
    
    // Initialize routes
    routes.InitRoutes(mux, manager)
    
    // Apply middleware
    handler := manager.With(mux)
    
    // Build address
    address := fmt.Sprintf(":%d", cnf.HTTPPort)
    
    // Start server
    log.Printf("Server running on port %d", cnf.HTTPPort)
    err := http.ListenAndServe(address, handler)
    if err != nil {
        log.Fatal("Server failed:", err)
        os.Exit(1)
    }
}
```

### File: main.go

```go
package main

import (
    "ecommerce/cmd"
    "ecommerce/config"
)

func main() {
    // Load configuration
    config.LoadConfig()
    
    // Start server
    cmd.Serve()
}
```

---

## Practice Questions

### Question 1: Why Global Variable?

**Question:** Why do we store configuration in a global variable instead of passing it as a parameter to every function?

<details>
<summary>Answer</summary>

**Reasons for global config:**

**1. Configuration is truly global**
   - Every part of application needs config
   - Port, database URL, API keys - used everywhere
   - Passing to every function is tedious

**2. Single source of truth**
   - One place to store config
   - No synchronization issues
   - Everyone reads same values

**3. Convenience**
   ```go
   // ❌ Without global (tedious)
   func Handler(cnf Config) {
       Process(cnf)
   }
   func Process(cnf Config) {
       Save(cnf)
   }
   func Save(cnf Config) {
       // Use cnf.DBUrl
   }
   
   // ✅ With global (clean)
   func Handler() {
       Process()
   }
   func Process() {
       Save()
   }
   func Save() {
       cnf := config.GetConfig()  // Get when needed
   }
   ```

**4. Read-only access**
   ```go
   // Private storage, public getter
   var configurations Config  // Can't modify from outside
   
   func GetConfig() Config {
       return configurations  // Returns copy
   }
   ```

**When NOT to use globals:**

- ❌ Mutable state that changes during execution
- ❌ State specific to requests/operations
- ❌ Testing becomes harder (but config is exception)

**For config specifically: Global is the right choice!**

</details>

---

### Question 2: Type Casting Challenge

**Question:** What's wrong with this code?

```go
port := int64(3000)
address := ":" + string(port)
```

Fix it three different ways.

<details>
<summary>Answer</summary>

**Problem:**

`string(port)` doesn't convert to decimal string! It converts to rune (character).

```go
port := int64(3000)
str := string(port)
fmt.Println(str)  // Weird character, not "3000"!
```

**Why?** `string()` interprets int as ASCII/Unicode code point.

---

**Solution 1: strconv.FormatInt() (for int64)**

```go
port := int64(3000)
portStr := strconv.FormatInt(port, 10)  // "3000"
address := ":" + portStr                 // ":3000"
```

---

**Solution 2: strconv.Itoa() (convert to int first)**

```go
port := int64(3000)
portInt := int(port)                    // Convert int64 → int
portStr := strconv.Itoa(portInt)        // "3000"
address := ":" + portStr                 // ":3000"
```

---

**Solution 3: fmt.Sprintf() (easiest!)**

```go
port := int64(3000)
address := fmt.Sprintf(":%d", port)  // ":3000"
```

**Best practice:** Use `fmt.Sprintf()` for simplicity!

</details>

---

### Question 3: Exit Codes

**Question:** Explain the difference between these:

```go
// A
if err != nil {
    return
}

// B
if err != nil {
    os.Exit(0)
}

// C
if err != nil {
    os.Exit(1)
}
```

<details>
<summary>Answer</summary>

**A: `return`**
```go
if err != nil {
    return  // Exit function only
}
```
- ✅ Exits current function
- ✅ Returns to caller
- ✅ Program continues running
- **Use when:** Recoverable error in function

---

**B: `os.Exit(0)`**
```go
if err != nil {
    os.Exit(0)  // Exit program with "success"
}
```
- ✅ Terminates entire program immediately
- ✅ Exit code 0 = "success"
- ❌ Confusing - error but success code?
- **Use when:** Almost never! Don't signal success on error

---

**C: `os.Exit(1)`**
```go
if err != nil {
    os.Exit(1)  // Exit program with "error"
}
```
- ✅ Terminates entire program immediately
- ✅ Exit code 1 = "error occurred"
- ✅ Shell/scripts can detect failure
- **Use when:** Fatal error, app can't continue

---

**Comparison:**

| Code | Scope | Continues? | Signal |
|------|-------|------------|--------|
| `return` | Function | Yes | None |
| `os.Exit(0)` | Program | No | Success |
| `os.Exit(1)` | Program | No | Error |

**Real example:**

```go
func LoadConfig() {
    err := godotenv.Load()
    if err != nil {
        // Can't start without config
        log.Fatal("No .env file")
        os.Exit(1)  // ✅ Fatal - terminate
    }
}

func ProcessRequest(r *Request) error {
    if r.Body == nil {
        return fmt.Errorf("no body")  // ✅ Return error
        // Don't exit - just one bad request
    }
    return nil
}
```

</details>

---

### Question 4: ParseInt Parameters

**Question:** Explain what each parameter does in:

```go
port, err := strconv.ParseInt("FF", 16, 32)
```

What is the value of `port`? What if we change to base 10?

<details>
<summary>Answer</summary>

**Parameter breakdown:**

```go
strconv.ParseInt("FF", 16, 32)
                 ▲     ▲   ▲
                 │     │   │
                 │     │   └─ bitSize: Result fits in 32 bits
                 │     └───── base: Hexadecimal (base 16)
                 └─────────── s: String to parse
```

---

**With base 16 (hexadecimal):**

```go
port, err := strconv.ParseInt("FF", 16, 32)
// "FF" in hex = 255 in decimal
// port = 255
// err = nil (success)
```

**Why?**
- F in hex = 15
- FF = (15 × 16) + 15 = 255

---

**With base 10 (decimal):**

```go
port, err := strconv.ParseInt("FF", 10, 32)
// port = 0
// err != nil (error!)
```

**Why?** "FF" has letters, not valid in base 10!

---

**Different bases:**

```go
// Binary (base 2)
strconv.ParseInt("1010", 2, 32)  // 10 (decimal)

// Octal (base 8)  
strconv.ParseInt("377", 8, 32)   // 255 (decimal)

// Decimal (base 10)
strconv.ParseInt("255", 10, 32)  // 255 (decimal)

// Hexadecimal (base 16)
strconv.ParseInt("FF", 16, 32)   // 255 (decimal)
```

---

**BitSize parameter:**

```go
// bitSize 32 - result fits in int32
strconv.ParseInt("255", 10, 32)
// Range: -2,147,483,648 to 2,147,483,647

// bitSize 64 - result fits in int64
strconv.ParseInt("255", 10, 64)
// Range: -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807
```

**Always use bitSize 64 for ports** - safe upper limit!

</details>

---

### Question 5: Implementing Database Config

**Challenge:** Add database configuration support.

**Requirements:**
1. Add to .env: `DATABASE_URL=postgres://user:pass@localhost/dbname`
2. Add to Config struct
3. Load and validate
4. Make accessible via GetConfig()

<details>
<summary>Answer</summary>

**Step 1: Update .env**

```env
VERSION=1.0.0
SERVICE_NAME=ecommerce
HTTP_PORT=3000
DATABASE_URL=postgres://user:pass@localhost:5432/ecommerce
```

---

**Step 2: Update Config struct**

```go
package config

type Config struct {
    Version     string
    ServiceName string
    HTTPPort    int64
    DatabaseURL string  // ← New field
}
```

---

**Step 3: Update LoadConfig()**

```go
func LoadConfig() Config {
    // Load .env
    err := godotenv.Load()
    if err != nil {
        log.Fatal("Failed to load .env:", err)
        os.Exit(1)
    }
    
    // Existing code...
    version := os.Getenv("VERSION")
    if version == "" {
        fmt.Println("VERSION is required")
        os.Exit(1)
    }
    
    serviceName := os.Getenv("SERVICE_NAME")
    if serviceName == "" {
        fmt.Println("SERVICE_NAME is required")
        os.Exit(1)
    }
    
    httpPort := os.Getenv("HTTP_PORT")
    if httpPort == "" {
        fmt.Println("HTTP_PORT is required")
        os.Exit(1)
    }
    
    port, err := strconv.ParseInt(httpPort, 10, 64)
    if err != nil {
        fmt.Println("HTTP_PORT must be a number")
        os.Exit(1)
    }
    
    // ← NEW: Load database URL
    databaseURL := os.Getenv("DATABASE_URL")
    if databaseURL == "" {
        fmt.Println("DATABASE_URL is required")
        os.Exit(1)
    }
    
    // Validate postgres URL format
    if !strings.HasPrefix(databaseURL, "postgres://") {
        fmt.Println("DATABASE_URL must start with postgres://")
        os.Exit(1)
    }
    
    // Create config
    cnf := Config{
        Version:     version,
        ServiceName: serviceName,
        HTTPPort:    port,
        DatabaseURL: databaseURL,  // ← Include in config
    }
    
    // Store globally
    configurations = cnf
    
    return cnf
}
```

---

**Step 4: Use in application**

```go
package database

import "ecommerce/config"

func Connect() (*sql.DB, error) {
    cnf := config.GetConfig()
    
    db, err := sql.Open("postgres", cnf.DatabaseURL)
    if err != nil {
        return nil, err
    }
    
    return db, nil
}
```

---

**Bonus: Optional config with defaults**

```go
// Optional config with fallback
timeout := os.Getenv("DB_TIMEOUT")
if timeout == "" {
    timeout = "30"  // Default 30 seconds
}

maxConns := os.Getenv("DB_MAX_CONNECTIONS")
if maxConns == "" {
    maxConns = "10"  // Default 10 connections
}
```

</details>

---

## Summary

**What we learned:**

1. **Configuration Management**
   - ✅ Why hardcoding is bad
   - ✅ Using .env files for configuration
   - ✅ Environment variables in processes
   - ✅ Loading with godotenv library

2. **Package Structure**
   - ✅ Creating config package
   - ✅ Config struct for type safety
   - ✅ Global configuration state
   - ✅ Public getter, private storage

3. **Type Conversions**
   - ✅ String → Int: `strconv.ParseInt()`
   - ✅ Int → String: `fmt.Sprintf()` or `strconv.FormatInt()`
   - ✅ Understanding number bases
   - ✅ Bit sizes for different int types

4. **Error Handling**
   - ✅ Validating required config
   - ✅ Using `os.Exit()` for fatal errors
   - ✅ Exit codes (0 = success, 1 = error)
   - ✅ Graceful failure messages

5. **Production Patterns**
   - ✅ Separating config from code
   - ✅ Never commit .env to Git
   - ✅ Different config per environment
   - ✅ Single source of truth for config

**Project structure now:**

```
ecommerce/
├── .env                  ← Configuration (not in Git!)
├── .gitignore           ← Add .env here
├── main.go              ← Load config, start server
├── go.mod               ← godotenv dependency
├── config/
│   └── config.go        ← Configuration management
├── cmd/
│   └── serve.go         ← Use config for port
├── middleware/
│   ├── logger.go
│   └── corsWithPreflight.go
├── handlers/
│   └── test.go
└── routes/
    └── routes.go
```

**You now have:**
- ✅ Production-ready configuration system
- ✅ Environment-specific settings
- ✅ Type-safe configuration
- ✅ Proper error handling
- ✅ Clean project structure

**Next chapter:** We'll add more configuration, database setup, and advanced project organization!

---

**Pro tip:** Always think about the **worst case scenario** as a programmer. If you plan for the worst, normal cases will be easy! 🎯
