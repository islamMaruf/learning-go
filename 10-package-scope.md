# Chapter 10: Package Scope — Multiple Files, Custom Packages, and Modules

> **Goal of this chapter:** Learn how real Go projects are organized. You'll split code across files, create your own packages, understand Go **modules** (`go.mod`), and use the capital-letter rule that controls what other packages can see.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 2 hours  **Prerequisite:** [Chapter 9](09-local-scope-and-block-scope.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why split code up?](#2-why-split-code-up)
3. [One package, many files](#3-one-package-many-files)
4. [What is a module? (`go.mod`)](#4-what-is-a-module)
5. [Creating your own package](#5-creating-your-own-package)
6. [Importing your package](#6-importing-your-package)
7. [Exported vs. unexported names](#7-exported-vs-unexported-names)
8. [Package naming and layout conventions](#8-package-naming-and-layout-conventions)
9. [Import details: groups, aliases, `_`, cycles](#9-import-details)
10. [Third-party packages (a preview)](#10-third-party-packages)
11. [A complete example project](#11-a-complete-example-project)
12. [Common errors](#12-common-errors)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- How several `.go` files in one folder form **one package**
- What a **module** is and what `go mod init` does
- How to create a **sub-package** in its own folder and import it
- The **capital-letter rule** for *exported* (public) vs *unexported* (private) names
- How to run and build multi-file projects
- The standard ways to lay out a Go project

---

## 2. Why split code up?

A 5,000-line `main.go` is as unmanageable as a 5,000-line function. Splitting gives you:

| Benefit | Meaning |
|---------|---------|
| **Organization** | Related code lives together (`auth`, `database`, `handlers`) |
| **Reusability** | Another project can import your package |
| **Encapsulation** | A package hides its internals and shows only a clean public surface |
| **Teamwork** | People work in different files/packages without conflict |
| **Faster builds** | Go caches compiled packages |

Two levels of splitting exist:
1. **Files**: cosmetic and organizational, within one package.
2. **Packages**: real boundaries with visibility rules.

---

## 3. One package, many files

**Rule:** all `.go` files in the **same folder** belong to the **same package** and must declare the same `package` name.

```
myproject/
├── go.mod
├── main.go
└── helpers.go
```

**main.go**

```go
// SKIP-CHECK: multi-file example
package main

import "fmt"

var appName = "Demo" // package scope

func main() {
	fmt.Println("Starting", appName)
	greet("Asha") // defined in helpers.go
}
```

**helpers.go**

```go
// SKIP-CHECK: multi-file example
package main

import "fmt"

func greet(name string) {
	fmt.Println("Hello,", name, "from", appName) // can use appName from main.go!
}
```

Because both files are in package `main`, they **share package scope**: `helpers.go` can call things defined in `main.go` and vice versa, with no import needed. Imports (`"fmt"`) are **per-file**, though: each file must import what it uses.

### Running a multi-file program

```bash
go run main.go        # ❌ undefined: greet   (Go only compiled the one file you named)
go run main.go helpers.go   # ✅ works, but tedious
go run .              # ✅ compiles the whole package in this folder
```

**Use `go run .` and `go build`.** Both compile *every* `.go` file in the package. (Files ending in `_test.go` are excluded from normal builds.)

### Order of files doesn't matter

Go processes all files in a package together. Unlike a script, it doesn't matter which file comes first, or where in a file a package-level function sits.

---

## 4. What is a module?

A **module** is a collection of packages that are versioned and released together: usually one repository. It is described by a file named **`go.mod`** at the project root.

### Creating a module

```bash
mkdir myproject && cd myproject
go mod init example.com/myproject
```

This creates `go.mod`:

```
module example.com/myproject

go 1.22
```

| Line | Meaning |
|------|---------|
| `module example.com/myproject` | The module's **path**: the prefix of every import path inside it |
| `go 1.22` | The Go language version the module targets |

### Choosing a module path

- For code you'll publish on GitHub: `github.com/yourname/myproject`.
- For private/learning projects: anything unique, such as `myproject` or `example.com/myproject`.

The path doesn't need to actually exist online unless others will `go get` it.

### What `go.mod` does for you

- Tells Go **where your packages live** so `import "example.com/myproject/mathlib"` resolves to the `mathlib/` folder.
- Records **dependencies** (third-party modules and versions), see section 10.
- Marks the **project root**: commands like `go run .` and `go build ./...` find it by walking up to the nearest `go.mod`.

---

## 5. Creating your own package

Add a sub-folder. **The folder holds one package**, and (by convention) the package name equals the folder name.

```
myproject/
├── go.mod                 module example.com/myproject
├── main.go                package main
└── mathlib/
    └── mathlib.go         package mathlib
```

**mathlib/mathlib.go**

```go
// SKIP-CHECK: library package (no main)
package mathlib

import "fmt"

// Money is exported (capital M), so other packages can use it.
var Money = 100

// debt is unexported (lowercase d), so only this package can use it.
var debt = 50

// Add is exported.
func Add(x, y int) int {
	return x + y
}

// sum is unexported: an internal helper.
func sum(x, y int) int {
	return x + y
}

// Describe is exported and uses the private pieces internally.
func Describe() string {
	return fmt.Sprintf("money=%d debt=%d total=%d", Money, debt, sum(Money, debt))
}
```

Note that `package mathlib` (not `main`). A package without `main` is a **library**: it can be imported but not run on its own.

---

## 6. Importing your package

An import path is: **module path + folder path**.

**main.go**

```go
// SKIP-CHECK: needs the mathlib package
package main

import (
	"fmt"

	"example.com/myproject/mathlib"
)

func main() {
	fmt.Println(mathlib.Add(4, 7)) // 11
	fmt.Println(mathlib.Money)     // 100
	fmt.Println(mathlib.Describe())// money=100 debt=50 total=150

	// fmt.Println(mathlib.debt)   // ❌ undefined: mathlib.debt (unexported)
	// mathlib.sum(1, 2)           // ❌ undefined: mathlib.sum (unexported)
}
```

Run from the project root:

```bash
go run .
```

Output:

```
11
100
money=100 debt=50 total=150
```

How the import path maps to files:

```
import "example.com/myproject/mathlib"
        └───── module path ────┘└─ folder ─┘
                 (from go.mod)

→ /home/you/myproject/mathlib/*.go
```

The name you use in code (`mathlib.Add`) is the **package name declared in the files** (`package mathlib`), which by convention matches the last path element.

---

## 7. Exported vs. unexported names

> **Names beginning with an uppercase letter are exported (visible to other packages). Names beginning with a lowercase letter are unexported (private to their package).**

There is no `public`/`private` keyword in Go; the first letter *is* the keyword.

| Declaration | First letter | Visible outside its package? |
|-------------|-------------|------------------------------|
| `func Add()` | `A` | ✅ Yes |
| `func add()` | `a` | ❌ No |
| `var Money` | `M` | ✅ Yes |
| `var money` | `m` | ❌ No |
| `type Person struct` | `P` | ✅ Yes |
| `type person struct` | `p` | ❌ No |
| Struct field `Name` | `N` | ✅ Yes |
| Struct field `name` | `n` | ❌ No |
| Method `Save()` / `save()` | `S` / `s` | ✅ / ❌ |

It applies to **everything**: functions, variables, constants, types, struct fields, methods.

### Why hide things?

**Encapsulation.** The exported names are your package's **public contract**. Everything else is internal and free to change without breaking anyone.

> **Analogy:** A car's dashboard (public API): steering wheel, pedals. The engine internals (unexported) are hidden. Drivers can't break the engine by accident, and mechanics can rebuild it without changing how you drive.

### Documenting exported names

Start the comment with the name:

```go
// Add returns the sum of x and y.
func Add(x, y int) int { return x + y }
```

`go doc` and editors show these comments to users.

### What if the other package needs to modify something private?

Provide exported functions ("getters/setters") that enforce the rules:

```go
// SKIP-CHECK: library package (no main)
package bank

var balance int // private

func Deposit(n int) {
	if n > 0 { // rule: only positive deposits
		balance += n
	}
}

func Balance() int { return balance }
```

Outside code can't set `balance = -1000`: it must go through `Deposit`.

---

## 8. Package naming and layout conventions

### Package names
- **Short, lowercase, single word:** `mathlib`, `auth`, `store`. Not `math_lib`, `MathLib`, `utils_and_helpers`.
- Avoid meaningless names like `util` or `common` if you can find something specific.
- The name is part of every call (`auth.Login`), so don't repeat it: `auth.Login` not `auth.AuthLogin`.

### Typical layouts

**Small program**

```
hello/
├── go.mod
└── main.go
```

**Medium program**

```
myapp/
├── go.mod
├── main.go
├── config/     config.go
├── handlers/   user.go  product.go
└── storage/    store.go
```

**Larger / conventional layout**

```
myapp/
├── go.mod
├── cmd/
│   └── myapp/
│       └── main.go     ← entry point(s)
├── internal/           ← private packages (see below)
│   ├── auth/
│   └── store/
└── pkg/                ← (optional) packages meant to be imported by others
```

We'll use this style in the e-commerce project (Chapters 40+).

### The special `internal/` folder

A package under a directory named `internal` can be imported **only by code inside the parent of `internal`**. The compiler enforces this, which makes it a great way to say "this is private to my module".

---

## 9. Import details

### Grouping

Use one `import ( ... )` block. By convention: standard library first, blank line, then everything else. `gofmt`/`goimports` do this for you.

```go
import (
	"fmt"
	"os"

	"example.com/myproject/mathlib"
)
```

### Aliases

Rename an import if names collide or are awkward:

```go
import (
	m "example.com/myproject/mathlib"
)
// m.Add(1, 2)
```

### Blank import: `_`

Import a package **only for its side effects** (its `init` function runs, see Chapter 14). Common with database drivers:

```go
import _ "github.com/lib/pq"
```

### Dot import (avoid)

`import . "fmt"` lets you write `Println` unqualified. It hurts readability, so avoid it.

### Import cycles are forbidden

If `a` imports `b`, then `b` cannot import `a` (directly or indirectly):

```
a ──► b ──► a    ❌ "import cycle not allowed"
```

Fix by moving shared code to a third package `c` that both import, or by rethinking who depends on whom (interfaces, Chapter 51, are a great tool for this).

### Unused imports are errors

Same as Chapter 1: import something and not use it, and the compile fails.

---

## 10. Third-party packages

Anyone's module can be added as a dependency:

```bash
go get github.com/google/uuid
```

Go edits `go.mod` (adds a `require` line) and creates/updates **`go.sum`** (checksums that guarantee you get the exact same code every time). Then:

```go
import "github.com/google/uuid"

id := uuid.New()
```

Useful commands:

| Command | Purpose |
|---------|---------|
| `go get pkg@version` | Add or upgrade a dependency |
| `go mod tidy` | Add missing / remove unused dependencies (run it often) |
| `go list -m all` | List all dependencies |
| `go build ./...` | Compile every package in the module |
| `go vet ./...` | Static checks on every package |

We'll use real third-party modules (a JWT library, a PostgreSQL driver) in Part 9 and Part 11.

---

## 11. A complete example project

```
shapes/
├── go.mod                  module example.com/shapes
├── main.go
└── geometry/
    ├── circle.go
    └── rectangle.go
```

**geometry/circle.go**

```go
// SKIP-CHECK: library package (no main)
package geometry

import "math"

// CircleArea returns the area of a circle with the given radius.
func CircleArea(radius float64) float64 {
	return math.Pi * square(radius)
}
```

**geometry/rectangle.go** (same package, a different file, so it shares `square`)

```go
// SKIP-CHECK: library package (no main)
package geometry

// RectangleArea returns width × height.
func RectangleArea(width, height float64) float64 {
	return width * height
}

// square is private, shared by both files of the package.
func square(x float64) float64 { return x * x }
```

**main.go**

```go
// SKIP-CHECK: needs the geometry package
package main

import (
	"fmt"

	"example.com/shapes/geometry"
)

func main() {
	fmt.Printf("Circle:    %.2f\n", geometry.CircleArea(3))
	fmt.Printf("Rectangle: %.2f\n", geometry.RectangleArea(4, 5))
}
```

```bash
go mod init example.com/shapes
go run .
```

Output:

```
Circle:    28.27
Rectangle: 20.00
```

Notice how `circle.go` uses `square`, which is defined in `rectangle.go`. That's **package scope across files**, and yet `main` can't touch `square` because it's unexported.

---

## 12. Common errors

| # | Message | Cause | Fix |
|---|---------|-------|-----|
| 1 | `undefined: add` when running `go run main.go` | Only one file compiled | `go run .` |
| 2 | `found packages main (main.go) and helper (add.go) in …` | Files in one folder disagree on `package` | Use the same package name |
| 3 | `go: cannot find main module` | No `go.mod` in this or a parent folder | `go mod init <path>` |
| 4 | `package example.com/x/mathlib is not in std` / `cannot find package` | Wrong import path (typo, or not module path + folder) | Match `go.mod` module path + folder |
| 5 | `undefined: mathlib.add` | Used an unexported (lowercase) name | Rename it with a capital letter (if it should be public) |
| 6 | `import cycle not allowed` | Two packages import each other | Extract shared code; reverse a dependency |
| 7 | `imported and not used` | Import you never referenced | Remove it |
| 8 | `function main is undeclared in the main package` | `package main` without `func main()` | Add `main`, or rename the package if it's a library |
| 9 | Package name ≠ folder name confusion | Legal but confusing | Keep them identical |
| 10 | Old code still runs | File not saved | Save before running |

---

## 13. Exercises

### Exercise 1: Calculator package
Create a module `example.com/calc` with a package `calculator` exporting `Add`, `Subtract`, `Multiply`, `Divide` (returning `(float64, error)`). Use it from `main`.

<details><summary>Solution</summary>

```
calc/
├── go.mod          (module example.com/calc)
├── main.go
└── calculator/calculator.go
```

**calculator/calculator.go**
```go
// SKIP-CHECK: library package (no main)
package calculator

import "errors"

func Add(a, b float64) float64      { return a + b }
func Subtract(a, b float64) float64 { return a - b }
func Multiply(a, b float64) float64 { return a * b }

func Divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("division by zero")
	}
	return a / b, nil
}
```

**main.go**
```go
// SKIP-CHECK: needs the calculator package
package main

import (
	"fmt"

	"example.com/calc/calculator"
)

func main() {
	fmt.Println(calculator.Add(2, 3))
	if v, err := calculator.Divide(9, 3); err == nil {
		fmt.Println(v)
	}
}
```
</details>

### Exercise 2: Public vs private
Add an exported `Power(base, exp int) int` to a package that internally uses an unexported `multiply(a, b int) int`. Try to call `multiply` from `main` and read the error.

<details><summary>Solution</summary>

```go
// SKIP-CHECK: library package (no main)
package mathx

func multiply(a, b int) int { return a * b }

func Power(base, exp int) int {
	result := 1
	for i := 0; i < exp; i++ {
		result = multiply(result, base)
	}
	return result
}
```
From `main`: `mathx.multiply(2, 3)` → `undefined: mathx.multiply` (unexported). `mathx.Power(2, 10)` → `1024`.
</details>

### Exercise 3: Multi-file
Put `main()` in `main.go` and `printBanner()` in `banner.go`, both in package `main`. Run with `go run .`.

<details><summary>Solution</summary>

**banner.go**
```go
// SKIP-CHECK: multi-file example
package main

import "fmt"

func printBanner() { fmt.Println("=== My App ===") }
```
**main.go**
```go
// SKIP-CHECK: multi-file example
package main

func main() { printBanner() }
```
</details>

### Exercise 4: Encapsulation
Create a package `counter` where the count is private and only `Increment()` and `Value()` are exported. Show that `main` cannot set it directly.

<details><summary>Solution</summary>

```go
// SKIP-CHECK: library package (no main)
package counter

var count int

func Increment()  { count++ }
func Value() int  { return count }
```
`counter.count = 5` from `main` is a compile error: `undefined: counter.count`.
</details>

### Exercise 5 (challenge): Break an import cycle
Package `user` imports `order` (to list a user's orders) and `order` imports `user` (to show the owner's name). Redesign so there is no cycle.

<details><summary>Solution</summary>

Extract the shared bit. For instance `order` only needs an owner's *ID and name*, so pass those as plain values or define a tiny type/interface **in `order`**:

```go
// SKIP-CHECK: sketch
package order

type Owner interface{ DisplayName() string } // order defines what it needs

type Order struct {
	ID    int
	Owner Owner
}
```

`user.User` implements `DisplayName()` without `order` importing `user`. Now only `user → order` remains. (This "consumer defines the interface" idea is the core of Chapter 51.)
</details>

---

## 14. Quiz

1. What must all files in one folder have in common?
2. What does `go mod init` create, and why?
3. How do you make a function visible to other packages?
4. What is an import path made of?
5. Why does `go run main.go` fail when helper files exist?
6. Can package `a` import `b` while `b` imports `a`?

<details><summary>Answers</summary>

1. The same `package` name.
2. `go.mod`, describing the module (path, Go version, dependencies) and enabling imports of your own packages.
3. Start its name with an uppercase letter.
4. Module path (from `go.mod`) + relative folder path.
5. It only compiles the files you list. Use `go run .`.
6. No. Import cycles are compile errors.
</details>

---

## 15. Summary

- A **package** = all `.go` files in one folder, sharing package scope. Use `go run .` / `go build`.
- A **module** = a tree of packages with a `go.mod` (module path + Go version + dependencies).
- Import your own packages with **module path + folder**.
- **Capital first letter = exported; lowercase = private.** This is Go's whole visibility system.
- Keep package names short and lowercase; use `internal/` to hide packages from outsiders.
- **No import cycles.**
- Add third-party code with `go get`; tidy up with `go mod tidy`.

### ➡️ What's next?

[Chapter 11](11-scope-deep-dive.md) puts all the scope rules (global, local, block, package) together in one big worked example, so you can trace any program with confidence.
