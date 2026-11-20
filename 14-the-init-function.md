# Chapter 14: The `init` Function — The Automatic Initializer

> **Goal of this chapter:** Meet the one function you can **never call yourself**. You'll learn exactly what order Go runs things in when a program starts, and when `init` is (and isn't) a good idea.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 1–1.5 hours  **Prerequisite:** [Chapter 13](13-function-types.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What is `init`?](#2-what-is-init)
3. [The real startup order](#3-the-real-startup-order)
4. [The rules of `init`](#4-the-rules-of-init)
5. [Why can't you call `init`?](#5-why-cant-you-call-init)
6. [Memory simulation](#6-memory-simulation)
7. [Multiple `init` functions](#7-multiple-init-functions)
8. [Package initialization order in detail](#8-package-initialization-order)
9. [Use cases](#9-use-cases)
10. [Why to be careful with `init`](#10-why-to-be-careful-with-init)
11. [Alternatives to `init`](#11-alternatives-to-init)
12. [Common mistakes](#12-common-mistakes)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- What `init` is and how it differs from every other function
- The **exact** order Go executes package-level variables, `init`, and `main`
- That a package (and even a single file) can have **several** `init` functions
- How imported packages' `init` functions fit in
- When to use `init`, and when a plain function is better

> 📝 **A note on "teaching in layers":** Earlier chapters said a program runs "package scope, then `main`". That was a **simplification** to keep you focused. The full truth is: package scope → **`init`** → `main`. There is even more detail in section 8. Learning in layers is normal. You never need all the detail at once.

---

## 2. What is `init`?

`init` is a **special function that Go calls automatically**, once, **before** `main` starts.

```go
package main

import "fmt"

func init() {
	fmt.Println("init runs first")
}

func main() {
	fmt.Println("main runs second")
}
```

Output:

```
init runs first
main runs second
```

Notice that `main` never mentions `init`. We didn't call it. Go did.

Its purpose is **initialization**: preparing things (variables, connections, registrations) so that `main` can start with everything ready.

> **Analogy:** Before a restaurant opens (`main`), staff switch on the ovens, wipe the tables, and unlock the door (`init`). Customers never ask for this; it just happens first.

---

## 3. The real startup order

When you run a Go program, this is the sequence:

```
Program starts
      │
      ▼
┌──────────────────────────────────────────┐
│ 1. Imported packages initialize first    │  (their variables, then their init())
│    (dependencies before dependents)      │
└──────────────────────────────────────────┘
      │
      ▼
┌──────────────────────────────────────────┐
│ 2. Package-level variables of THIS       │  var a = compute()
│    package are initialized               │
└──────────────────────────────────────────┘
      │
      ▼
┌──────────────────────────────────────────┐
│ 3. init() functions of THIS package run  │  automatic, in order
└──────────────────────────────────────────┘
      │
      ▼
┌──────────────────────────────────────────┐
│ 4. main() runs                           │
└──────────────────────────────────────────┘
      │
      ▼
Program ends when main returns
```

Let me prove it. This program has a package variable initialized by a function, an `init`, and `main`:

```go
package main

import "fmt"

var greeting = makeGreeting() // (2) package-level variable initializer

func makeGreeting() string {
	fmt.Println("1) package variable is being initialized")
	return "hello"
}

func init() {
	fmt.Println("2) init runs; greeting =", greeting)
}

func main() {
	fmt.Println("3) main runs")
}
```

Output:

```
1) package variable is being initialized
2) init runs; greeting = hello
3) main runs
```

Variables come first, so `init` can already **use** them.

---

## 4. The rules of `init`

### Rule 1: The name must be exactly `init`

### Rule 2: No parameters

```go
// INTENTIONAL ERROR
package main

func init(x int) {} // ❌ func init must have no arguments and no return values

func main() {}
```

### Rule 3: No return values

`func init() int { ... }` is also a compile error.

### Rule 4: You cannot call or reference it

```go
// INTENTIONAL ERROR
package main

func init() {}

func main() {
	init() // ❌ undefined: init
}
```

The identifier `init` is *not even declared* in the normal way, so you can't refer to it. Not to call it, not to store it in a variable.

### Rule 5: It runs exactly once

Per package, per program run, no matter how many times the package is imported by other packages.

### Rule 6: You can have many

Even several in the same file (next section).

### Summary table

| Property | `init` | Ordinary function |
|----------|--------|-------------------|
| Called by | Go runtime, automatically | You |
| Parameters | none | any |
| Results | none | any |
| How many per package | many allowed | names must be unique |
| Can you call it? | ❌ never | ✅ |
| Can you take it as a value (`f := init`)? | ❌ | ✅ |

---

## 5. Why can't you call `init`?

Think about what it would mean. `init` is designed to run **exactly once**, at startup, to put things into a known good state. If anyone could call it again mid-program, it might:

- reconnect to a database that's already connected,
- reset variables while other code is using them,
- register the same plugin twice.

By making it **uncallable**, Go guarantees the promise: *"When `main` begins, initialization is complete, and it happened exactly once."*

This is a guarantee, so you can rely on it without extra checks.

---

## 6. Memory simulation

```go
package main

import "fmt"

var a = 10

func init() {
	b := 20
	fmt.Println("init:", a+b)
}

func main() {
	fmt.Println("main:", a)
}
```

**Phase 1: package scope created and variables initialized**

```
┌────────────────────────────┐
│ PACKAGE  a = 10            │
│          init = <code>     │
│          main = <code>     │
└────────────────────────────┘
```

**Phase 2: Go finds `init` and runs it, with its own scope like any function**

```
┌────────────────────────────┐
│ PACKAGE  a = 10            │
└────────────────────────────┘
   ┌───────────────────┐
   │ init   b = 20     │      prints "init: 30"
   └───────────────────┘
```

**Phase 3: `init` ends, so its scope (and `b`) are destroyed**

```
┌────────────────────────────┐
│ PACKAGE  a = 10            │
└────────────────────────────┘
```

**Phase 4: `main` runs**

```
┌────────────────────────────┐
│ PACKAGE  a = 10            │
└────────────────────────────┘
   ┌───────────────────┐
   │ main              │      prints "main: 10"
   └───────────────────┘
```

**Phase 5: `main` ends, and the program ends.**

Output:

```
init: 30
main: 10
```

`init` behaves like any function *inside*: it has local variables, a stack frame, etc. Only *how it's called* is special.

---

## 7. Multiple `init` functions

You may define more than one `init`, even in one file. They run **in the order they appear**:

```go
package main

import "fmt"

func init() { fmt.Println("Init 1") }
func init() { fmt.Println("Init 2") }
func init() { fmt.Println("Init 3") }

func main() { fmt.Println("Main") }
```

Output:

```
Init 1
Init 2
Init 3
Main
```

### Across multiple files

If a package has several files, each with an `init`, the Go toolchain presents the files to the compiler **sorted by filename**, so `init`s run in **filename order**, and in source order within each file. The language spec does *not* promise this, though: it says files should be presented "in a manner" the build system defines. In practice, `go build` sorts them.

```
config.go    → func init() { "Config init" }
database.go  → func init() { "Database init" }
main.go      → func init() { "Main file init" }; func main() {...}
```

Typical output:

```
Config init
Database init
Main file init
Main function
```

> ⚠️ **Never write code that depends on the order of `init` functions across files.** It's fragile: renaming a file could change behavior. If ordering matters, use explicit function calls from `main` (section 11).

---

## 8. Package initialization order in detail

Here's a real demonstration with everything at once. **Project:**

```
initdemo/
├── go.mod           module example.com/initdemo
├── main.go
├── zeta.go
└── db/
    └── db.go
```

`db/db.go`:

```go
// SKIP-CHECK: multi-file project
package db

import "fmt"

func init() { fmt.Println("db: init") }

func Connect() { fmt.Println("db: connect") }
```

`main.go`:

```go
// SKIP-CHECK: multi-file project
package main

import (
	"fmt"

	"example.com/initdemo/db"
)

var a = initA()

func initA() int {
	fmt.Println("var a initialized")
	return 1
}

func init() { fmt.Println("main.go: init #1") }
func init() { fmt.Println("main.go: init #2") }

func main() {
	fmt.Println("main runs")
	db.Connect()
	fmt.Println(a, b)
}
```

`zeta.go`:

```go
// SKIP-CHECK: multi-file project
package main

import "fmt"

var b = func() int {
	fmt.Println("var b initialized (zeta.go)")
	return a + 1
}()

func init() { fmt.Println("zeta.go: init") }
```

Real output of `go run .`:

```
db: init
var a initialized
var b initialized (zeta.go)
main.go: init #1
main.go: init #2
zeta.go: init
main runs
db: connect
1 2
```

**Reading the output:**

1. **`db: init` first.** Imported packages are fully initialized before the importing package starts.
2. **Then package-level variables of `main`**: `a`, then `b` (b depends on `a`, so Go guarantees `a` first. The compiler sorts variable initialization by *dependency*, not by textual order).
3. **Then all `init` functions** in file order (`main.go` before `zeta.go`), and in source order within a file.
4. **Then `main`.**

**The full rule:** *(imported packages, recursively) → package variables (dependency order) → `init` functions → `main`.*

---

## 9. Use cases

### 9.1 Initialize state that can't be a simple expression

```go
package main

import "fmt"

var squares [5]int

func init() {
	for i := range squares {
		squares[i] = i * i
	}
}

func main() { fmt.Println(squares) } // [0 1 4 9 16]
```

Filling a table needs a loop, and loops can't sit in a variable declaration. `init` is a natural fit.

### 9.2 Validate the environment

```go
package main

import (
	"fmt"
	"os"
)

var port string

func init() {
	port = os.Getenv("PORT")
	if port == "" {
		port = "8080" // sensible default
	}
}

func main() { fmt.Println("listening on", port) }
```

### 9.3 Register with a registry (the classic use)

Standard-library and third-party packages often use `init` to **register themselves**:

```go
import (
	"database/sql"
	_ "github.com/lib/pq" // blank import: only its init() runs
)
```

The `pq` package's `init` calls `sql.Register("postgres", ...)`, so `sql.Open("postgres", ...)` afterward knows the driver. That's why we import it with `_` (Chapter 52 uses exactly this). Image formats (`image/png`) and `net/http/pprof` work the same way.

### 9.4 Compile regular expressions / build lookup maps once

```go
package main

import (
	"fmt"
	"regexp"
)

var emailRE *regexp.Regexp

func init() {
	emailRE = regexp.MustCompile(`^[^@\s]+@[^@\s]+\.[a-z]+$`)
}

func main() { fmt.Println(emailRE.MatchString("a@b.com")) } // true
```

(Simpler still: `var emailRE = regexp.MustCompile(...)` needs no `init` at all. See section 11.)

---

## 10. Why to be careful with `init`

`init` is convenient but **implicit**: code runs that nobody visibly called.

| Downside | Explanation |
|----------|-------------|
| **Hidden behavior** | Readers can't see the call in `main`; side effects "just happen" |
| **Hard to test** | Tests can't skip or repeat `init` |
| **No error return** | It can't return an error; failure means `panic` or `os.Exit` |
| **Ordering surprises** | Across files/packages, ordering is subtle |
| **Import side effects** | Merely *importing* a package can run code (connect to databases, start goroutines) |

**Guideline:** use `init` only for *simple, deterministic, side-effect-light* setup such as registration and table-building. For anything that can fail or needs configuration (connecting to a database, reading config files), use an explicit function called from `main`.

---

## 11. Alternatives to `init`

**1. Initialize at declaration** (best for simple cases):

```go
var emailRE = regexp.MustCompile(`...`)
var startTime = time.Now()
```

**2. An explicit setup function** called from `main`, so failures are handled and order is visible:

```go
package main

import (
	"fmt"
	"os"
)

type Config struct{ Port string }

func loadConfig() (Config, error) {
	port := os.Getenv("PORT")
	if port == "" {
		return Config{}, fmt.Errorf("PORT is required")
	}
	return Config{Port: port}, nil
}

func main() {
	cfg, err := loadConfig()
	if err != nil {
		fmt.Println("startup error:", err)
		os.Exit(1)
	}
	fmt.Println("running on", cfg.Port)
}
```

This is the style we'll use in the e-commerce project (Chapter 47).

**3. `sync.Once`** for lazy, once-only initialization (Chapter 65+ territory).

---

## 12. Common mistakes

| # | Mistake | Consequence / fix |
|---|---------|-------------------|
| 1 | Calling `init()` | `undefined: init`. You can't |
| 2 | Giving `init` parameters or results | Compile error |
| 3 | Assuming `init` runs before *imported* packages | It runs **after** its imports' initialization |
| 4 | Relying on cross-file `init` order | Fragile; make dependencies explicit |
| 5 | Doing heavy or failing work (network, DB) in `init` | Hard to handle errors/test. Use explicit setup |
| 6 | Expecting `init` to run again per goroutine/call | It runs once |
| 7 | Naming another function `init2` and expecting automatic behavior | Only the exact name `init` is special |
| 8 | Declaring a *variable* called `init` | Illegal: `cannot declare init - must be func` |

---

## 13. Exercises

### Exercise 1: Predict the output

```go
package main

import "fmt"

var x = f("x")

func f(s string) string { fmt.Println("initializing", s); return s }

func init() { fmt.Println("init") }

func main() { fmt.Println("main", x) }
```

<details><summary>Solution</summary>

```
initializing x
init
main x
```
</details>

### Exercise 2: Several inits
How many lines does this print, and in what order?

```go
package main

import "fmt"

func init() { fmt.Println("A") }
func main() { fmt.Println("B") }
func init() { fmt.Println("C") }
```

<details><summary>Solution</summary>

Three lines: `A`, `C`, `B`. Both `init`s run (in source order) before `main`, no matter where `main` sits.
</details>

### Exercise 3: Lookup table
Use `init` to fill a `map[string]int` named `dayNumber` with day names (`"Mon": 1` … `"Sun": 7`). Print `dayNumber["Wed"]` from `main`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

var dayNumber = map[string]int{}

func init() {
	days := []string{"Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"}
	for i, d := range days {
		dayNumber[d] = i + 1
	}
}

func main() { fmt.Println(dayNumber["Wed"]) } // 3
```
(A map literal would also do this without `init`. The exercise is about the mechanism.)
</details>

### Exercise 4: Find the errors

```go
// INTENTIONAL ERROR
package main

import "fmt"

func init() int {
	return 1
}

func main() {
	init()
	fmt.Println("done")
}
```

<details><summary>Solution</summary>

Two errors: `init` must not have a result (`func init must have no arguments and no return values`), and `init()` can't be called (`undefined: init`). Remove both.
</details>

### Exercise 5: Replace `init`
Rewrite Exercise 3 without `init` and without a map literal, by using a function `buildDays() map[string]int` assigned at declaration.

<details><summary>Solution</summary>

```go
package main

import "fmt"

var dayNumber = buildDays()

func buildDays() map[string]int {
	m := map[string]int{}
	for i, d := range []string{"Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"} {
		m[d] = i + 1
	}
	return m
}

func main() { fmt.Println(dayNumber["Wed"]) }
```
</details>

### Exercise 6 (challenge): Trace the whole startup

```go
package main

import "fmt"

var a = trace("a", b+1)
var b = trace("b", 1)

func trace(name string, v int) int {
	fmt.Println("var", name, "=", v)
	return v
}

func init() { fmt.Println("init 1") }
func init() { fmt.Println("init 2") }

func main() { fmt.Println("main", a, b) }
```

<details><summary>Solution</summary>

`a` depends on `b`, so Go initializes `b` first regardless of textual order:

```
var b = 1
var a = 2
init 1
init 2
main 2 1
```
</details>

---

## 14. Quiz

1. Who calls `init`?
2. What runs first: package-level variable initialization or `init`?
3. Can a package have multiple `init` functions?
4. Why can't `init` return an error?
5. What does `import _ "pkg"` do?

<details><summary>Answers</summary>

1. The Go runtime, automatically.
2. Variables first, then `init`.
3. Yes, and they run in the order presented (source order within a file).
4. It has no results by definition. Failure must `panic`/exit. That's a reason to prefer explicit setup for fallible work.
5. Imports the package only for its side effects (its `init`), without referencing it by name.
</details>

---

## 15. Summary

- **`init`** runs automatically, once, **before `main`**. It takes nothing, returns nothing, and can't be called.
- **Order:** imported packages → package variables (dependency order) → `init` functions (file/source order) → `main`.
- A package may have **many** `init` functions; don't depend on cross-file order.
- Great for **registration**, **lookup tables**, and light setup; poor for fallible or heavy work.
- Prefer **initialization at declaration** or an **explicit `setup()` from `main`** when possible.

### ➡️ What's next?

[Chapter 15](15-anonymous-functions-and-iife.md) introduces functions **without names**: anonymous functions and **IIFEs**.
