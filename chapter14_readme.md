# Chapter 14: Init Function - The Automatic Initializer

## 📚 Table of Contents
1. [Introduction](#introduction)
2. [What Is the Init Function?](#what-is-the-init-function)
3. [The Truth About Execution Order](#the-truth-about-execution-order)
4. [Init Function Rules](#init-function-rules)
5. [Why Init Cannot Be Called](#why-init-cannot-be-called)
6. [Complete Memory Simulation](#complete-memory-simulation)
7. [Init Function Use Cases](#init-function-use-cases)
8. [Multiple Init Functions](#multiple-init-functions)
9. [The Philosophy of Organization](#the-philosophy-of-organization)
10. [Common Mistakes](#common-mistakes)
11. [Practice Exercises](#practice-exercises)
12. [Summary](#summary)
13. [What's Next?](#whats-next)

---

## Introduction

**Today's Topic: Init Function!**

### The Special Function

> **"This function CANNOT be called by you! You CANNOT call it even if you want to! The computer calls it AUTOMATICALLY!"**

**Let's explore this mysterious function!**

### What Makes Init Special?

```go
func init() {
    // I run automatically!
    // You cannot call me!
    // I have no inputs!
    // I have no outputs!
    // I am SPECIAL! ✨
}
```

---

## What Is the Init Function?

### Simple Definition

> **The `init` function is a SPECIAL function in Go that:**
> - ✅ Executes AUTOMATICALLY before `main()`
> - ✅ Takes NO parameters
> - ✅ Returns NO values
> - ✅ CANNOT be called manually
> - ✅ Used for initialization tasks

### Basic Example

```go
package main

import "fmt"

func init() {
    fmt.Println("I am the FIRST function that executes FIRST")
}

func main() {
    fmt.Println("Hello Init Function")
}
```

**Output:**
```
I am the FIRST function that executes FIRST
Hello Init Function
```

**Notice:** Init prints FIRST, then main prints!

---

## The Truth About Execution Order

### The Confession

> **"Sorry guys, I was teaching you WRONG before! I was lying! The mistake was: I said Global → Main. SORRY! The truth is: Global → Init → Main!"**

### The Progressive Teaching

> **"You were in Class 1 before. Now you're in Class 2, so I'm telling you the truth! When you reach Class 3, I'll tell you even MORE truth! Do you need to know ALL the truth of the world? No! The person who knows LESS is HAPPIER!"**

**Philosophical note:** Sometimes we teach in steps for easier learning! 📚

---

### WRONG Execution Order (Old Teaching)

```
❌ WRONG ORDER:
1. Global Scope
2. main() executes
```

---

### CORRECT Execution Order (New Truth!)

```
✅ CORRECT ORDER:
1. Global Scope (variables, functions declared)
2. init() executes (if defined)
3. main() executes
```

---

### Visual Execution Flow

```
Program Start
     ↓
┌─────────────────────┐
│  GLOBAL SCOPE       │
│  (Declarations)     │
└─────────────────────┘
     ↓
┌─────────────────────┐
│  init()             │  ← Automatic!
│  (Initialization)   │  ← No call needed!
└─────────────────────┘
     ↓
┌─────────────────────┐
│  main()             │  ← Automatic!
│  (Main logic)       │
└─────────────────────┘
     ↓
Program End
```

---

## Init Function Rules

### The 5 Sacred Rules

#### Rule 1: Must Be Named "init"

```go
✅ CORRECT:
func init() {
    // This works!
}

❌ WRONG:
func initialize() {  // Wrong name!
    // This is NOT an init function
}
```

---

#### Rule 2: No Parameters Allowed

```go
✅ CORRECT:
func init() {
    // No parameters
}

❌ WRONG:
func init(x int) {  // ERROR!
    // Init cannot take parameters
}
```

**Why?** Computer calls it automatically - what parameters would it pass?

---

#### Rule 3: No Return Values Allowed

```go
✅ CORRECT:
func init() {
    // No return
}

❌ WRONG:
func init() int {  // ERROR!
    return 42
}
```

**Why?** Computer calls it automatically - who would receive the return?

---

#### Rule 4: Cannot Be Called Manually

```go
func init() {
    fmt.Println("Init running")
}

func main() {
    init()  // ❌ ERROR: undefined: init
}
```

**Error message:** `undefined: init`

**Why?** Init is called by the runtime, not by your code!

---

#### Rule 5: Runs Automatically Once

```go
func init() {
    fmt.Println("I run ONCE automatically")
}

func main() {
    fmt.Println("Main running")
    // No way to run init() again!
}
```

**Output:**
```
I run ONCE automatically
Main running
```

**Init runs exactly ONCE, before main!**

---

## Why Init Cannot Be Called

### The Attempt

```go
package main

import "fmt"

func init() {
    fmt.Println("Init function")
}

func main() {
    init()  // Try to call it
}
```

**Error:**
```
./main.go:9:2: undefined: init
```

---

### The Explanation

**Why undefined error?**

> **"You CANNOT call init! Even if you TRY, the computer will say 'undefined'. The computer calls init automatically. You cannot call it. You cannot give it input. You cannot see its output (return value)."**

**Think of it like:**
```
init() is like your heartbeat
    ↓
You don't control it manually
    ↓
It just happens automatically
    ↓
You can't say "beat now!" to your heart
```

---

## Complete Memory Simulation

### Code to Simulate

```go
package main

import "fmt"

var a = 10  // Global variable

func init() {
    fmt.Println(a)  // Print a
    a = 20          // Change a
}

func main() {
    fmt.Println(a)  // Print a
}
```

**What will the output be?** 🤔

---

### Phase 1: Global Scope Created

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  a = 10                                 │
│  init = [function code]                 │
│  main = [function code]                 │
└─────────────────────────────────────────┘
```

**What happened:**
1. ✅ Computer reads `var a = 10` → Stores in global
2. ✅ Computer reads `func init()` → Stores function
3. ✅ Computer reads `func main()` → Stores function
4. ⏸️ No execution yet - just declarations!

---

### Phase 2: Computer Checks for Init

```
Computer thinks:
"Is there an 'init' function defined?"
    ↓
Searches global scope
    ↓
Found: YES! init exists!
    ↓
"I must call init() FIRST!"
```

**Computer AUTOMATICALLY calls init!**

---

### Phase 3: Init Function Executes

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  a = 10  ← Will be modified!            │
│  init = [function code] ← EXECUTING     │
│  main = [function code]                 │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         INIT SCOPE (Created)            │
├─────────────────────────────────────────┤
│  (executing line by line)               │
└─────────────────────────────────────────┘
```

**Step 1:** `fmt.Println(a)`
- Look for `a` in init scope → Not found
- Look for `a` in global scope → Found! (a = 10)
- **Print: 10** ✅

**Step 2:** `a = 20`
- Look for `a` in init scope → Not found
- Look for `a` in global scope → Found!
- Modify global `a` from 10 to 20
- Global `a` is now 20! ✅

---

### Phase 4: Init Scope Destroyed

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  a = 20  ← MODIFIED by init!            │
│  init = [function code]                 │
│  main = [function code]                 │
└─────────────────────────────────────────┘

[INIT SCOPE DESTROYED! 💀]
```

**Init's job is DONE!**

> **"Init lived and died quickly! Born, worked, gone!"**

---

### Phase 5: Main Function Executes

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  a = 20  ← Modified by init             │
│  main = [function code] ← EXECUTING     │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         MAIN SCOPE (Created)            │
├─────────────────────────────────────────┤
│  (executing line by line)               │
└─────────────────────────────────────────┘
```

**Step:** `fmt.Println(a)`
- Look for `a` in main scope → Not found
- Look for `a` in global scope → Found! (a = 20)
- **Print: 20** ✅

---

### Phase 6: Program Ends

```
┌─────────────────────────────────────────┐
│         MAIN SCOPE                      │
│         (Destroyed!)                    │
└─────────────────────────────────────────┘

[MAIN SCOPE DESTROYED! 💀]
```

> **"When main ends, EVERYTHING ends! Main is the SOUL (আত্মা) of the program. When the soul dies, everything dies!"**

---

### Phase 7: All Memory Cleaned

```
[GLOBAL SCOPE DESTROYED! 💀]
[EVERYTHING DESTROYED! 💀]

Program execution complete.
```

**The world ends when main ends!**

> **"Main is the REAL thing (আসল). When the REAL thing ends, everything ends!"**

---

### Final Output

```
10
20
```

**Explanation:**
1. Init prints 10 (original value)
2. Init changes a to 20
3. Main prints 20 (modified value)

---

## Init Function Use Cases

### Use Case 1: Initialize Global Variables

```go
package main

import "fmt"

var globalConfig map[string]string

func init() {
    // Initialize the map
    globalConfig = make(map[string]string)
    globalConfig["env"] = "development"
    globalConfig["version"] = "1.0.0"
    fmt.Println("Config initialized")
}

func main() {
    fmt.Println("Environment:", globalConfig["env"])
}
```

**Output:**
```
Config initialized
Environment: development
```

---

### Use Case 2: Setup Database Connection

```go
package main

import "fmt"

var dbConnection string

func init() {
    // Simulate database connection setup
    fmt.Println("Connecting to database...")
    dbConnection = "postgresql://localhost:5432/mydb"
    fmt.Println("Database connected!")
}

func main() {
    fmt.Println("Using connection:", dbConnection)
}
```

**Output:**
```
Connecting to database...
Database connected!
Using connection: postgresql://localhost:5432/mydb
```

---

### Use Case 3: Register Drivers/Plugins

```go
package main

import "fmt"

var registeredDrivers []string

func init() {
    // Auto-register drivers
    registeredDrivers = append(registeredDrivers, "mysql")
    registeredDrivers = append(registeredDrivers, "postgres")
    fmt.Println("Drivers registered")
}

func main() {
    fmt.Println("Available drivers:", registeredDrivers)
}
```

**Output:**
```
Drivers registered
Available drivers: [mysql postgres]
```

---

### Use Case 4: Validate Environment

```go
package main

import (
    "fmt"
    "os"
)

func init() {
    // Check critical environment variable
    apiKey := os.Getenv("API_KEY")
    if apiKey == "" {
        fmt.Println("ERROR: API_KEY not set!")
        os.Exit(1)  // Exit before main even runs!
    }
    fmt.Println("Environment validated")
}

func main() {
    fmt.Println("Application starting...")
}
```

**If API_KEY not set:**
```
ERROR: API_KEY not set!
(Program exits before main runs!)
```

**If API_KEY is set:**
```
Environment validated
Application starting...
```

---

## Multiple Init Functions

### The Secret: You Can Have Multiple Init Functions!

**Yes! You can define MULTIPLE init functions in the SAME package!**

```go
package main

import "fmt"

func init() {
    fmt.Println("Init 1")
}

func init() {
    fmt.Println("Init 2")
}

func init() {
    fmt.Println("Init 3")
}

func main() {
    fmt.Println("Main")
}
```

**Output:**
```
Init 1
Init 2
Init 3
Main
```

**They execute in ORDER of appearance!** ✅

---

### Multiple Init Across Multiple Files

**File: config.go**
```go
package main

import "fmt"

func init() {
    fmt.Println("Config init")
}
```

**File: database.go**
```go
package main

import "fmt"

func init() {
    fmt.Println("Database init")
}
```

**File: main.go**
```go
package main

import "fmt"

func init() {
    fmt.Println("Main file init")
}

func main() {
    fmt.Println("Main function")
}
```

**Execution order:**
```
Config init
Database init
Main file init
Main function
```

**Order depends on alphabetical file processing!** (Usually)

---

## The Philosophy of Organization

### The Life Lesson

> **"Engineers must make EVERYTHING beautiful! Every detail must be beautiful!"**

### The Desktop Analogy

**Messy Desktop = Messy Brain:**

```
Messy Desktop:
    File here, file there
    No organization
    Can't find anything
    ↓
Reflects:
    Scattered brain
    Disorganized thoughts
    Poor memory retrieval
```

**Clean Desktop = Clean Brain:**

```
Organized Desktop:
    Files in folders
    Everything labeled
    Easy to find
    ↓
Reflects:
    Organized brain
    Clear thoughts
    Good memory retrieval
```

---

### The Library Analogy

> **"A librarian can find ONE BOOK among 10 LAKH (1 million) books in minutes. Why? ORGANIZATION!"**

**But you with 20 books:**
```
"Where's my wallet?"
"Where did I put that book?"
"Can't find anything!"
```

**Why?** Disorganized brain = Disorganized life!

---

### The Mother's Wisdom

> **"My mother was the HAPPIEST person in the world. Why? She didn't know/care about worldly things. The person who knows LESS is HAPPIER!"**

**But we programmers:**
```
"Why did they look at me that way?"
"Why did they talk like that?"
"Why not say it more softly?"

We notice EVERYTHING!
We care about EVERYTHING!
We have NO PEACE! 😅
```

---

### The Engineering Principle

> **"When you start making everything beautiful, everything AROUND you becomes beautiful:**
> - Beautiful relationships
> - Beautiful family connections
> - Beautiful social media
> - Beautiful career
> - EVERYTHING becomes beautiful!"**

**Start with small things:**
1. ✅ Clean your room daily
2. ✅ Organize your desktop
3. ✅ Write beautiful code
4. ✅ Write beautiful sentences (like this tutorial!)
5. ✅ Everything else follows!

---

### The Code Beauty

**Why spend time on a single line?**

```go
// Ugly
fmt.Println("I am the FIRST function that executes FIRST")

// Could be
fmt.Println("First function")

// But the longer one is MORE BEAUTIFUL!
// It's a PROPER SENTENCE!
// Life should be PROPER!
```

> **"If life is not proper, your mind feels disturbed. Make every corner of life beautiful!"**

---

### The Brain Information Storage

**Fact:** Humans NEVER forget information!

- 👥 You see thousands of people on the street
- 📸 Every face is stored in your brain
- 💾 Every detail remains forever
- 🧠 But RETRIEVAL depends on ORGANIZATION!

**Those who can retrieve information well:**
- ✅ Have organized brains
- ✅ Like a well-organized library
- ✅ Can find anything quickly

**Those who can't:**
- ❌ Have scattered information
- ❌ Like a messy library
- ❌ Can't find anything

---

### The Song Memory

> **"Lalon's song comes to mind: 'The mind doesn't agree for false things. If I die, I'll find the real thing. There's no real thing - running after fake things!'"**

**Meaning:** The brain stores EVERYTHING - even random song lyrics!

---

## Common Mistakes

### Mistake 1: Trying to Call Init

❌ **Wrong:**
```go
func init() {
    fmt.Println("Initializing...")
}

func main() {
    init()  // ERROR: undefined: init
}
```

✅ **Correct:**
```go
func init() {
    fmt.Println("Initializing...")
}

func main() {
    // Init already ran automatically!
    fmt.Println("Main running")
}
```

---

### Mistake 2: Adding Parameters

❌ **Wrong:**
```go
func init(config string) {  // ERROR!
    fmt.Println(config)
}
```

✅ **Correct:**
```go
var config string = "default"

func init() {
    fmt.Println(config)  // Use global variables
}
```

---

### Mistake 3: Returning Values

❌ **Wrong:**
```go
func init() int {  // ERROR!
    return 42
}
```

✅ **Correct:**
```go
var result int

func init() {
    result = 42  // Set global variable
}
```

---

### Mistake 4: Assuming Init Runs After Main

❌ **Wrong thinking:**
```go
func main() {
    fmt.Println("First")
}

func init() {
    fmt.Println("Second")
}

// Output: First, Second ??? NO!
```

✅ **Correct understanding:**
```go
func main() {
    fmt.Println("Second")
}

func init() {
    fmt.Println("First")
}

// Output: First, Second ✅
```

**Init ALWAYS runs BEFORE main!**

---

### Mistake 5: Not Knowing Multiple Inits Are Allowed

❌ **Wrong assumption:**
```
"I can only have ONE init function"
```

✅ **Correct:**
```go
func init() {
    fmt.Println("Init 1")
}

func init() {
    fmt.Println("Init 2")  // This is ALLOWED!
}
```

**Multiple inits are PERFECTLY FINE!**

---

## Practice Exercises

### Exercise 1: Predict the Output

```go
package main

import "fmt"

var x = 5

func init() {
    x = x * 2
    fmt.Println("Init:", x)
}

func main() {
    x = x + 10
    fmt.Println("Main:", x)
}
```

**Question:** What is the output?

<details>
<summary>Answer</summary>

**Output:**
```
Init: 10
Main: 20
```

**Explanation:**
1. Global: x = 5
2. Init: x = 5 * 2 = 10, prints "Init: 10"
3. Main: x = 10 + 10 = 20, prints "Main: 20"

</details>

---

### Exercise 2: Multiple Inits

```go
package main

import "fmt"

var counter = 0

func init() {
    counter++
    fmt.Println("Init A:", counter)
}

func init() {
    counter++
    fmt.Println("Init B:", counter)
}

func main() {
    counter++
    fmt.Println("Main:", counter)
}
```

**Question:** What is the output?

<details>
<summary>Answer</summary>

**Output:**
```
Init A: 1
Init B: 2
Main: 3
```

**Explanation:** Inits run in order, then main!

</details>

---

### Exercise 3: Init Without Main Usage

```go
package main

import "fmt"

var initialized bool

func init() {
    initialized = true
    fmt.Println("System initialized")
}

func main() {
    if initialized {
        fmt.Println("Ready to go!")
    } else {
        fmt.Println("Not initialized")
    }
}
```

**Question:** What is the output?

<details>
<summary>Answer</summary>

**Output:**
```
System initialized
Ready to go!
```

**Explanation:** Init sets initialized = true before main runs!

</details>

---

### Exercise 4: Create Config Initializer

Create a program that:
1. Uses init to set up a configuration map
2. Adds default values in init
3. Main function reads and prints config

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

var config map[string]string

func init() {
    config = make(map[string]string)
    config["app"] = "MyApp"
    config["version"] = "1.0.0"
    config["env"] = "production"
    fmt.Println("Configuration initialized")
}

func main() {
    fmt.Println("App:", config["app"])
    fmt.Println("Version:", config["version"])
    fmt.Println("Environment:", config["env"])
}
```

**Output:**
```
Configuration initialized
App: MyApp
Version: 1.0.0
Environment: production
```

</details>

---

### Exercise 5: Order Challenge

```go
package main

import "fmt"

var a = 1

func init() {
    a = 2
}

func init() {
    fmt.Println(a)
    a = 3
}

func main() {
    fmt.Println(a)
}
```

**Question:** What prints?

<details>
<summary>Answer</summary>

**Output:**
```
2
3
```

**Explanation:**
1. Global: a = 1
2. Init 1: a = 2
3. Init 2: prints 2, then a = 3
4. Main: prints 3

</details>

---

### Exercise 6: Real-World Database Mock

Create a realistic database initialization:

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

var (
    dbConnected bool
    dbName      string
    maxConn     int
)

func init() {
    fmt.Println("=== Database Initialization ===")
    
    // Simulate connection setup
    dbName = "myapp_db"
    maxConn = 10
    
    fmt.Println("Connecting to database:", dbName)
    fmt.Println("Setting max connections:", maxConn)
    
    // Simulate successful connection
    dbConnected = true
    fmt.Println("Database connected successfully!")
    fmt.Println("===============================")
}

func main() {
    if !dbConnected {
        fmt.Println("ERROR: Database not connected!")
        return
    }
    
    fmt.Println("Application started")
    fmt.Println("Using database:", dbName)
    fmt.Println("Max connections:", maxConn)
}
```

**Output:**
```
=== Database Initialization ===
Connecting to database: myapp_db
Setting max connections: 10
Database connected successfully!
===============================
Application started
Using database: myapp_db
Max connections: 10
```

</details>

---

## Summary

### Key Takeaways

1. ✅ **Init function is SPECIAL** - Runs automatically before main
2. ✅ **Execution order: Global → Init → Main**
3. ✅ **Cannot be called manually** - Compiler error if you try
4. ✅ **No parameters, no return values**
5. ✅ **Used for initialization** - Setup before main runs
6. ✅ **Multiple inits allowed** - Run in order of appearance
7. ✅ **Package-level feature** - Each package can have inits

---

### Init Function Rules Summary

| Rule | Description |
|------|-------------|
| **Name** | Must be exactly `init` |
| **Parameters** | None allowed |
| **Return** | None allowed |
| **Calling** | Cannot call manually - automatic only |
| **Execution** | Once before main, automatically |
| **Multiple** | Multiple inits allowed in same package |
| **Purpose** | Initialization and setup |

---

### Visual Summary

```
Program Start
     ↓
┌─────────────────────────┐
│  1. GLOBAL SCOPE        │
│     var a = 10          │
│     func init() {...}   │
│     func main() {...}   │
└─────────────────────────┘
     ↓
┌─────────────────────────┐
│  2. INIT FUNCTION       │  ← Automatic!
│     (Setup/Initialize)  │  ← Before main!
└─────────────────────────┘
     ↓
┌─────────────────────────┐
│  3. MAIN FUNCTION       │  ← Automatic!
│     (Main logic)        │  ← After init!
└─────────────────────────┘
     ↓
Program End
```

---

### The Execution Truth

```
❌ OLD (Wrong): Global → Main
                ↓
✅ NEW (Correct): Global → Init → Main
                ↓
🎓 Truth Level: Class 1 → Class 2 → Class 3...
(More truth revealed as you progress!)
```

---

### When to Use Init

**Good use cases:**
- ✅ Initialize global variables
- ✅ Setup database connections
- ✅ Register drivers/plugins
- ✅ Validate environment
- ✅ Pre-compute values
- ✅ Setup logging

**Bad use cases:**
- ❌ Complex business logic
- ❌ User input processing
- ❌ Long-running operations
- ❌ Anything that should be controlled by user

---

## What's Next?

### Chapter 15 Preview: Anonymous Functions

Now that we know about standard and init functions, let's explore functions WITHOUT names!

**Topics covered:**
- **What are anonymous functions?** - Functions without names
- **Function literals** - Writing functions inline
- **Assigning functions to variables** - Functions as values
- **Passing functions as arguments** - Functions as parameters
- **Returning functions** - Functions returning functions
- **Immediate invocation** - IIFE pattern

**The contrast:**
```go
// Standard (has name)
func add(a, b int) int {
    return a + b
}

// Init (special name)
func init() {
    // Automatic execution
}

// Anonymous (NO name!)
func(a, b int) int {
    return a + b
}  // ← How do we use this? 🤔
```

---

### Why This Progression Matters

```
Chapter 13: Standard Functions (WITH name)
Chapter 14: Init Function (SPECIAL name) ← YOU ARE HERE
Chapter 15: Anonymous Functions (NO name) (Next!)
```

**Building knowledge step by step!** 🏗️

---

### The Journey Continues

**What we've learned so far:**
- 🎯 Functions (Chapter 7-11)
- 🎯 Scope (Chapter 8-11)
- 🎯 Shadowing (Chapter 12)
- 🎯 Standard Functions (Chapter 13)
- 🎯 Init Functions (Chapter 14) ← Current
- 🎯 Anonymous Functions (Chapter 15) ← Next!

**Each chapter builds understanding!** 📚

---

### Final Wisdom

**The Progressive Truth:**
> **"Class 1 → Class 2 → Class 3... Each level, more truth! You don't need ALL truth at once. Learn step by step!"**

**The Automatic Power:**
> **"Init is like your heartbeat - automatic! You don't control it. It just happens. That's the power of init!"**

**The Organization Philosophy:**
> **"Make everything beautiful - your code, your desktop, your life. When you organize small things, big things organize themselves!"**

**The Mother's Lesson:**
> **"Sometimes knowing LESS makes you HAPPIER. But for programmers, knowing MORE makes you BETTER! Choose wisely!"**

**The Main Soul:**
> **"Main is the SOUL (আত্মা) of the program. When main ends, everything ends. Init prepares the world, main lives in it!"**

---

### Quick Reference Card

```go
// INIT FUNCTION TEMPLATE
func init() {
    // Setup code here
    // Runs ONCE, AUTOMATICALLY
    // BEFORE main()
    // No parameters
    // No return values
    // Cannot be called manually
}

// EXECUTION ORDER
1. Global declarations
2. init() ← Automatic!
3. main() ← Automatic!
```

---

### The Memory Pattern

```
┌─────────────────────┐
│  GLOBAL SCOPE       │  ← Created first
│  a = 10             │
└─────────────────────┘
         ↓
┌─────────────────────┐
│  INIT SCOPE         │  ← Created, executed, destroyed
│  (modifies global)  │
└─────────────────────┘
         ↓
┌─────────────────────┐
│  MAIN SCOPE         │  ← Created, executed, destroyed
│  (uses global)      │
└─────────────────────┘
         ↓
    [All destroyed]
```

---

**Allah Hafez (আল্লাহ হাফেজ)!** 

**See you in the next class!** 👋

---

*End of Chapter 14 - Init Function Mastered!* ✨
