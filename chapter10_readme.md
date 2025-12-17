# Chapter 10: Package Scope - Custom Packages and Modules

## 📚 Table of Contents
1. [Introduction](#introduction)
2. [Review: Three Types of Scope](#review-three-types-of-scope)
3. [What Is Package Scope?](#what-is-package-scope)
4. [Same Package, Multiple Files](#same-package-multiple-files)
5. [Creating Custom Packages](#creating-custom-packages)
6. [Go Modules - go mod init](#go-modules---go-mod-init)
7. [Importing Custom Packages](#importing-custom-packages)
8. [Exported vs Unexported Identifiers](#exported-vs-unexported-identifiers)
9. [Running Multiple Files](#running-multiple-files)
10. [Complete Package Example](#complete-package-example)
11. [Common Package Errors](#common-package-errors)
12. [Practice Exercises](#practice-exercises)
13. [Summary](#summary)
14. [What's Next?](#whats-next)

---

## Introduction

**Package scope is a bit complex!** 🤯

So far, you've learned:
- ✅ Global scope (outside all functions)
- ✅ Local scope (inside functions and blocks)

Now it's time to learn the third type: **Package Scope**!

**What You'll Learn:**
1. ✅ How to organize code across multiple files
2. ✅ How to create your own packages
3. ✅ What is `go mod init` and why you need it
4. ✅ How to import custom packages
5. ✅ Capital vs lowercase letter rules
6. ✅ How to run programs with multiple files

---

## Review: Three Types of Scope

```
┌─────────────────────────────────────┐
│   1. GLOBAL SCOPE                   │
│      - Outside all functions        │
│      - Accessible within same file  │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│   2. LOCAL SCOPE                    │
│      - Inside functions/blocks      │
│      - Limited to that block        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│   3. PACKAGE SCOPE ← Today's Topic! │
│      - Across files in same package │
│      - Controls visibility          │
└─────────────────────────────────────┘
```

---

## What Is Package Scope?

### Definition

> **Package scope determines what can be accessed across different files and different packages.**

### Key Concepts

1. **Same Package:** Multiple files can share the same package name
2. **Different Packages:** Packages can import and use each other
3. **Visibility Rules:** Capital letters = visible outside package
4. **Module System:** Go modules organize packages

---

## Same Package, Multiple Files

### The Rule

> **All files in the same folder MUST have the same package name.**

### Example: Two Files, One Package

#### Project Structure
```
myproject/
├── main.go
└── add.go
```

#### File 1: main.go
```go
package main

import "fmt"

var a = 20
var b = 30

func main() {
    fmt.Println("a =", a)
    fmt.Println("b =", b)
    add(4, 7)
}
```

#### File 2: add.go
```go
package main

import "fmt"

func add(n1 int, n2 int) {
    result := n1 + n2
    fmt.Println("Result:", result)
}
```

### Running Multiple Files

**Wrong way (will fail):**
```bash
go run main.go
```

**Error:**
```
undefined: add
```

**Why?** Go only reads `main.go`, doesn't know about `add.go`!

**Correct way:**
```bash
go run main.go add.go
```

**Or run all Go files:**
```bash
go run *.go
```

**Output:**
```
a = 20
b = 30
Result: 11
```

---

## Creating Custom Packages

### Why Create Custom Packages?

1. ✅ **Organization:** Group related functions
2. ✅ **Reusability:** Use the same code in multiple projects
3. ✅ **Maintainability:** Easier to manage large projects
4. ✅ **Separation:** Keep different concerns separate

### Project Structure for Custom Package

```
myproject/
├── main.go
├── mathlib/
│   └── math.go
└── go.mod
```

### Step-by-Step: Creating a Custom Package

#### Step 1: Create Folder Structure

```bash
myproject/
├── main.go
└── mathlib/
```

#### Step 2: Create mathlib/math.go

```go
package mathlib

import "fmt"

// Add two numbers
func Add(x int, y int) {
    z := x + y
    fmt.Println(z)
}

// Money variable
var Money = 100
```

**Important Notes:**
- ✅ Package name matches folder name: `package mathlib`
- ✅ Function `Add` starts with **capital A** (exported)
- ✅ Variable `Money` starts with **capital M** (exported)

---

## Go Modules - go mod init

### What Are Go Modules?

> **Go modules are Go's dependency management system. They help organize packages and manage imports.**

### Creating a Module

**Command:**
```bash
go mod init example.com
```

**What happens:**
1. ✅ Creates a `go.mod` file
2. ✅ Defines the module path
3. ✅ Enables custom package imports

### Example: Initialize Module

```bash
cd myproject
go mod init example.com
```

**Output:**
```
go: creating new go.mod: module example.com
```

**Generated go.mod file:**
```
module example.com

go 1.22
```

### Why Is This Needed?

Without `go mod init`:
- ❌ Can't import custom packages
- ❌ Go doesn't know where to find your packages
- ❌ Import paths won't work

With `go mod init`:
- ✅ Can import custom packages
- ✅ Go knows the module root
- ✅ Import paths work correctly

---

## Importing Custom Packages

### Import Syntax

**Built-in packages:**
```go
import "fmt"
import "math"
```

**Custom packages:**
```go
import "example.com/mathlib"
```

**Pattern:** `module_name/package_name`

### Example: main.go

```go
package main

import (
    "fmt"
    "example.com/mathlib"
)

func main() {
    fmt.Println("Showing custom package:")
    mathlib.Add(4, 7)
    fmt.Println("Money:", mathlib.Money)
}
```

**Key Points:**
1. ✅ Import path: `example.com/mathlib`
2. ✅ Use dot notation: `mathlib.Add()`
3. ✅ Access exported names only

---

## Exported vs Unexported Identifiers

### The Capital Letter Rule

> **In Go, names that start with a capital letter are EXPORTED (visible outside the package). Names that start with lowercase are UNEXPORTED (private to the package).**

### Examples

#### Exported (Public) - Starts with Capital Letter

```go
package mathlib

// ✅ Exported - can be used from other packages
func Add(x, y int) int {
    return x + y
}

// ✅ Exported - can be accessed from other packages
var Money = 100

// ✅ Exported - visible outside
type Person struct {
    Name string
}
```

#### Unexported (Private) - Starts with Lowercase Letter

```go
package mathlib

// ❌ Unexported - only usable within mathlib package
func add(x, y int) int {
    return x + y
}

// ❌ Unexported - only accessible within mathlib
var money = 100

// ❌ Unexported - not visible outside
type person struct {
    name string
}
```

### Visibility Table

| Name | Starts With | Visibility | Can Access From? |
|------|-------------|------------|------------------|
| `Add` | Capital A | ✅ Exported | Other packages |
| `add` | Lowercase a | ❌ Unexported | Same package only |
| `Money` | Capital M | ✅ Exported | Other packages |
| `money` | Lowercase m | ❌ Unexported | Same package only |

---

## Running Multiple Files

### Scenario: Two Files in Same Package

**Project Structure:**
```
myproject/
├── main.go
└── add.go
```

**main.go:**
```go
package main

import "fmt"

func main() {
    fmt.Println("Calling add function")
    add(10, 20)
}
```

**add.go:**
```go
package main

import "fmt"

func add(a, b int) {
    fmt.Println("Sum:", a+b)
}
```

### How to Run

**Option 1: Specify all files**
```bash
go run main.go add.go
```

**Option 2: Use wildcard**
```bash
go run *.go
```

**Option 3: Use go build**
```bash
go build
./myproject
```

**Output:**
```
Calling add function
Sum: 30
```

---

## Complete Package Example

Let's build a complete example with custom package!

### Project Structure

```
myproject/
├── go.mod
├── main.go
└── mathlib/
    └── math.go
```

### Step 1: Initialize Module

```bash
cd myproject
go mod init example.com
```

### Step 2: Create mathlib/math.go

```go
package mathlib

import "fmt"

// Add - Exported function (Capital A)
func Add(x int, y int) {
    z := x + y
    fmt.Println("Addition result:", z)
}

// sum - Unexported function (lowercase s)
func sum(x int, y int) int {
    return x + y
}

// Money - Exported variable (Capital M)
var Money = 100

// debt - Unexported variable (lowercase d)
var debt = 50
```

### Step 3: Create main.go

```go
package main

import (
    "fmt"
    "example.com/mathlib"
)

func main() {
    fmt.Println("=== Custom Package Demo ===")
    
    // ✅ Can access exported function
    mathlib.Add(4, 7)
    
    // ✅ Can access exported variable
    fmt.Println("Money:", mathlib.Money)
    
    // ❌ Cannot access unexported function
    // mathlib.sum(1, 2)  // This would cause error!
    
    // ❌ Cannot access unexported variable
    // fmt.Println(mathlib.debt)  // This would cause error!
}
```

### Step 4: Run the Program

```bash
go run main.go
```

**Output:**
```
=== Custom Package Demo ===
Addition result: 11
Money: 100
```

---

## Common Package Errors

### ❌ Error 1: Wrong Package Name in Same Folder

**Problem:**
```
myproject/
├── main.go (package main)
└── add.go (package helper)  // ❌ Different package!
```

**Error:**
```
found packages main and helper
```

**Fix:**
Both files must have the same package:
```go
// main.go
package main

// add.go
package main  // ✅ Same package name
```

---

### ❌ Error 2: Cannot Import Without go mod

**Problem:**
```go
import "myproject/mathlib"  // ❌ No go.mod file
```

**Error:**
```
could not import myproject/mathlib
```

**Fix:**
```bash
go mod init myproject
```

Then import becomes:
```go
import "myproject/mathlib"  // ✅ Works now!
```

---

### ❌ Error 3: Trying to Access Unexported Name

**mathlib/math.go:**
```go
package mathlib

func add(x, y int) int {  // lowercase 'a' - unexported
    return x + y
}
```

**main.go:**
```go
package main

import "example.com/mathlib"

func main() {
    mathlib.add(1, 2)  // ❌ Error!
}
```

**Error:**
```
undefined: mathlib.add
```

**Why?** `add` starts with lowercase, so it's unexported (private).

**Fix:**
Change function name to start with capital letter:
```go
func Add(x, y int) int {  // ✅ Capital 'A'
    return x + y
}
```

---

### ❌ Error 4: Running Only main.go When add.go Exists

**Problem:**
```bash
go run main.go  # ❌ Doesn't read add.go
```

**Error:**
```
undefined: add
```

**Fix:**
```bash
go run main.go add.go  # ✅ Include all files
# or
go run *.go  # ✅ Run all Go files
```

---

### ❌ Error 5: Wrong Import Path

**go.mod:**
```
module example.com
```

**Wrong import:**
```go
import "mathlib"  // ❌ Wrong!
```

**Correct import:**
```go
import "example.com/mathlib"  // ✅ Correct!
```

**Pattern:** `module_name/package_folder`

---

### ❌ Error 6: Forgetting to Save File

**Symptom:** Code doesn't work, shows old errors

**Check:** White circle on tab means unsaved!

**Fix:**
- Mac: `Cmd + S`
- Windows/Linux: `Ctrl + S`

---

## Practice Exercises

### Exercise 1: Create a Calculator Package

**Task:** Create a `calculator` package with these exported functions:
- `Add(a, b int) int`
- `Subtract(a, b int) int`
- `Multiply(a, b int) int`
- `Divide(a, b float64) float64`

**Structure:**
```
myproject/
├── go.mod
├── main.go
└── calculator/
    └── calc.go
```

**Use it in main.go to perform calculations.**

---

### Exercise 2: Exported vs Unexported

**Task:** In the calculator package, add:
- Exported function: `Power(base, exp int) int`
- Unexported helper: `multiply(a, b int) int`

Try to call both from `main.go` and observe the error for the unexported one.

---

### Exercise 3: Multi-File Same Package

**Task:** Create two files in the `main` package:
- `main.go` - has `main()` function
- `helpers.go` - has helper functions

Make sure both work together when you run `go run *.go`

---

### Exercise 4: String Utilities Package

**Task:** Create a `strutil` package with:
- `Reverse(s string) string` - reverses a string
- `ToUpper(s string) string` - converts to uppercase
- `ToLower(s string) string` - converts to lowercase

**Bonus:** Add an unexported helper function `isVowel(c rune) bool`

---

### Exercise 5: Constants in Package

**Task:** Create a `constants` package with:
```go
package constants

const Pi = 3.14159
const E = 2.71828
var AppName = "MyApp"
var Version = "1.0.0"
```

Import and use these in `main.go`.

---

### Exercise 6: Fix the Errors

**Given this broken code, fix all errors:**

```go
// go.mod is missing - need to create it

// mathlib/math.go
package mathlib

func add(x, y int) {  // Should be Add
    fmt.Println(x + y)
}

var money = 100  // Should be Money

// main.go
package main

import "mathlib"  // Wrong import path

func main() {
    mathlib.add(5, 10)      // Wrong name
    fmt.Println(mathlib.money)  // Wrong name
}
```

---

## Summary

### Key Takeaways

1. ✅ **Three types of scope:** Global, Local, Package
2. ✅ **Same folder = same package name**
3. ✅ **Custom packages = organized code**
4. ✅ **go mod init = enables custom imports**
5. ✅ **Capital letter = Exported (public)**
6. ✅ **Lowercase letter = Unexported (private)**
7. ✅ **Import path = module_name/package_name**
8. ✅ **Run multiple files with: go run *.go**

### Scope Types Summary

| Scope Type | Location | Accessibility |
|------------|----------|---------------|
| **Global** | Outside functions | Same file |
| **Local** | Inside functions/blocks | That block only |
| **Package** | Across files in package | Same package |
| **Exported** | Capital letter names | Other packages |

### Capital Letter Rules

```go
// mathlib package

// ✅ EXPORTED - Visible outside package
func Add(x, y int) int { ... }
var Money = 100
type Person struct { ... }

// ❌ UNEXPORTED - Only visible inside mathlib package
func add(x, y int) int { ... }
var money = 100
type person struct { ... }
```

### Import Patterns

```go
// Built-in packages (no module prefix)
import "fmt"
import "math"

// Custom packages (with module prefix)
import "example.com/mathlib"
import "myproject/calculator"
import "github.com/username/repo/package"
```

### Running Go Programs

| Scenario | Command |
|----------|---------|
| Single file | `go run main.go` |
| Multiple files | `go run main.go other.go` |
| All files | `go run *.go` |
| Build executable | `go build` |

### Project Structure Best Practices

```
myproject/
├── go.mod              # Module definition
├── main.go             # Entry point
├── helpers.go          # Helper functions (same package)
├── package1/           # Custom package 1
│   └── code.go
├── package2/           # Custom package 2
│   └── code.go
└── utils/              # Utilities package
    ├── string.go
    └── math.go
```

---

## What's Next?

Now that you understand packages and scope, you're ready for more advanced topics!

### Chapter 11 Preview: Arrays and Slices
- **Arrays** - Fixed-size collections
- **Slices** - Dynamic arrays (more flexible)
- **Array operations** - Accessing, modifying elements
- **Slice operations** - Appending, slicing, copying
- **Memory management** - How arrays/slices work in RAM
- **Iteration** - Looping through collections

### Why Package Scope Was Important

Without understanding packages:
- ❌ Can't organize large projects
- ❌ Can't reuse code effectively
- ❌ Can't use third-party libraries
- ❌ Can't understand import errors
- ❌ Can't build real-world applications

**With package knowledge:**
- ✅ Organize code professionally
- ✅ Create reusable libraries
- ✅ Use external packages
- ✅ Build scalable applications
- ✅ Work on team projects

---

### Go Module Commands Reference

```bash
# Initialize a new module
go mod init example.com

# Add missing dependencies
go mod tidy

# Download dependencies
go mod download

# Verify dependencies
go mod verify

# Show module information
go mod graph
```

---

### Real-World Package Examples

**Standard Library Packages:**
- `fmt` - Formatted I/O
- `math` - Mathematical functions
- `strings` - String manipulation
- `time` - Time and date
- `os` - Operating system functionality
- `net/http` - HTTP client and server

**Third-Party Packages:**
- `github.com/gin-gonic/gin` - Web framework
- `github.com/gorilla/mux` - HTTP router
- `gorm.io/gorm` - ORM library

---

### Learning Wisdom

**The Repetition Truth:**

> "Even if explaining the same thing repeatedly is annoying, you should still ask! What matters most is LEARNING. You MUST learn!"

**Package Scope Complexity:**

> "Package scope is a bit complex, but once you understand it, everything else becomes easier!"

**The Persistence Principle:**

> "Don't think things will wait for you when you're done. When your work is finished, you're removed from memory. That's life. That's how computers work. We built computers, and they work how we think."

**Engineering Mindset:**

- ❌ Engineers don't memorize every detail
- ✅ Engineers understand concepts
- ✅ Engineers look up syntax as needed
- ✅ Engineers complete tasks
- ✅ Engineers keep learning

**Study Approach:**
1. ✅ Code while learning (don't just watch/read)
2. ✅ Make mistakes and fix them
3. ✅ Create your own packages
4. ✅ Experiment with exports/unexports
5. ✅ Build small projects
6. ✅ Ask questions when stuck

**Remember:**
> "You need to save your file! White circle on tab = unsaved. Save it or it won't work!"

---

**الله حافظ (Allah Hafez)**

---

### Additional Resources

**Folder Naming Tips:**
- Use descriptive names: `mathlib`, `utils`, `helpers`
- Keep lowercase (Go convention)
- No spaces in names
- Match package name to folder name

**Module Naming:**
- Use your domain: `yourdomain.com/project`
- Or use a placeholder: `example.com/project`
- Or GitHub path: `github.com/username/repo`

**Quick Reference:**

```go
// Exported (Public)
func PublicFunction() {}
var PublicVariable = 10
type PublicType struct {}

// Unexported (Private)
func privateFunction() {}
var privateVariable = 10
type privateType struct {}
```

---

*End of Chapter 10 - Package Scope Mastered!* 📦

