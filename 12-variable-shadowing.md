# Chapter 12: Variable Shadowing — The Shadow Effect

> **Goal of this chapter:** Understand what happens when an inner scope declares a variable with the *same name* as one in an outer scope. Shadowing is a top interview question and one of the sneakiest sources of real-world bugs.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 1.5 hours  **Prerequisite:** [Chapters 8–11](08-what-is-scope.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What is shadowing?](#2-what-is-shadowing)
3. [The classic interview question](#3-the-classic-interview-question)
4. [Memory simulation, phase by phase](#4-memory-simulation)
5. [Lookup priority](#5-lookup-priority)
6. [Real-life analogies](#6-real-life-analogies)
7. [The `:=` rule: when does Go create a *new* variable?](#7-the--rule)
8. [Interview practice set](#8-interview-practice-set)
9. [Common shadowing patterns](#9-common-shadowing-patterns)
10. [When shadowing is fine](#10-when-shadowing-is-fine)
11. [When shadowing is dangerous (real bugs)](#11-when-shadowing-is-dangerous)
12. [How to catch shadowing bugs](#12-how-to-catch-shadowing-bugs)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- What **shadowing** is, and how to spot it
- How Go decides which variable a name refers to
- The exact difference between `x := 5` (new variable) and `x = 5` (assign to existing)
- The classic `err` shadowing bug
- When shadowing is harmless, and when to avoid it
- Tools that flag suspicious shadowing

---

## 2. What is shadowing?

> **Shadowing** happens when a variable declared in an inner scope has the **same name** as a variable in an outer scope. Inside the inner scope, the new variable *hides* the outer one. The outer variable still exists; it's just temporarily invisible by that name.

```
outer scope:      x = 10   ◄── still alive, untouched
                    │
inner scope:      ┌─┴────────────────┐
                  │  x = 99          │  ← this x "casts a shadow" over the outer x
                  │  Println(x) → 99 │
                  └──────────────────┘
after the block:  Println(x) → 10   (the shadow is gone)
```

Important: **two different variables** exist, in two different memory locations. Changing the inner one does not affect the outer one.

---

## 3. The classic interview question

```go
package main

import "fmt"

var a = 10 // package scope

func main() {
	age := 30 // function scope

	if age > 18 {
		var a int = 47 // block scope: SHADOWS the package-level a
		fmt.Println(a) // (1)
	}

	fmt.Println(a) // (2)
}
```

**What does it print?** Think before you read on. ⏸️

<details>
<summary>Answer</summary>

```
47
10
```

1. Inside the `if` block, the search for `a` finds the **block's** `a` first → `47`.
2. After the block ends, that `a` is destroyed. The search finds the **package** `a` → `10`.

Nothing was overwritten: the package-level `a` was never touched.
</details>

---

## 4. Memory simulation

Let's follow the program from Section 3 through memory.

**Phase 1: package scope**

```
┌───────────────────────┐
│ PACKAGE  a = 10       │
└───────────────────────┘
```

**Phase 2: `main` starts, `age := 30`**

```
┌───────────────────────┐
│ PACKAGE  a = 10       │
└───────────────────────┘
   ┌──────────────────┐
   │ main  age = 30   │
   └──────────────────┘
```

**Phase 3: `age > 18` is true, so the block is entered and a scope is created**

**Phase 4: `var a int = 47` executes. A NEW variable is created *inside the block***

```
┌───────────────────────┐
│ PACKAGE  a = 10       │  ← the original: hidden inside the block
└───────────────────────┘
   ┌───────────────────────┐
   │ main  age = 30        │
   │  ┌─────────────────┐  │
   │  │ if block        │  │
   │  │  a = 47   ◄─────┼──┼── NEW variable, different memory
   │  └─────────────────┘  │
   └───────────────────────┘
```

**Phase 5: `fmt.Println(a)`.** Lookup starts in the innermost scope: the block has `a` → **47** (search stops).

**Phase 6: the block ends: its scope, and its `a`, are destroyed**

```
┌───────────────────────┐
│ PACKAGE  a = 10       │  ← visible again!
└───────────────────────┘
   ┌──────────────────┐
   │ main  age = 30   │
   └──────────────────┘
```

**Phase 7: `fmt.Println(a)`.** Search: `main`'s scope has no `a` → package scope has `a` → **10**.

---

## 5. Lookup priority

```
Priority 1 (highest)   innermost block
Priority 2             enclosing block(s)
Priority 3             function scope (parameters + top-level locals)
Priority 4             package scope
Priority 5 (lowest)    universe scope (int, string, true, nil, len …)
```

**The first match wins.** An inner variable *always* takes priority over an outer one with the same name.

The universe scope can be shadowed too (legal but terrible):

```go
package main

import "fmt"

func main() {
	true := false // legal!! shadows the built-in true
	fmt.Println(true) // false
}
```

Never do this. It's only mentioned so nothing surprises you: `len := 3`, `string := "x"`, `error := ...` are all legal, and all confusing.

---

## 6. Real-life analogies

**The birthmark.** A man has a birthmark on his arm. His son is born with the same mark on the same arm. At home, people say "the birthmark" and mean the son's. The dad's is still there, but the *nearest* one gets the name.

**"Ask Dad for money."** Inside the house, "Dad" means *your* dad. In your friend's house, the same word "Dad" means *their* dad. Same name, different people, depending on which house (scope) you're standing in.

**Nicknames in a class.** If two students are named Sam, the teacher says "Sam" and everyone looks at the *nearest* Sam.

---

## 7. The `:=` rule

Whether you create a new variable (shadowing) or modify an existing one comes down to `:=` versus `=`.

| Statement | Meaning |
|-----------|---------|
| `x := 5` | **Declare** a new `x` in the **current scope** (shadowing any outer `x`) |
| `x = 5` | **Assign** to the **nearest existing** `x` (no new variable) |
| `var x = 5` | Same as `:=` (declare new in current scope) |

```go
package main

import "fmt"

var x = 10

func main() {
	if true {
		x = 20 // assignment: modifies the PACKAGE x
	}
	fmt.Println(x) // 20

	if true {
		x := 30 // declaration: new block-local x
		_ = x
	}
	fmt.Println(x) // 20 (unchanged by the block-local one)
}
```

Output:

```
20
20
```

### The multi-variable `:=` subtlety

`:=` with several names on the left requires **at least one new variable** *in the current scope*. Names that already exist **in that same scope** are simply **assigned** (reused), not redeclared:

```go
package main

import "fmt"

func main() {
	a, b := 1, 2
	b, c := 20, 30 // b is reused (assigned), c is new: legal!
	fmt.Println(a, b, c) // 1 20 30
}
```

But if the existing name is in an **outer** scope, `:=` creates a **new** variable for it, and this is where bugs come from (section 11).

---

## 8. Interview practice set

### Q1: Function shadows package
```go
package main

import "fmt"

var x = 5

func main() {
	x := 10
	fmt.Println(x)
}
```
**Answer:** `10`. The local `x` shadows the package `x`.

### Q2: Block shadows function
```go
package main

import "fmt"

func main() {
	x := 10
	if true {
		x := 20
		fmt.Println(x) // A
	}
	fmt.Println(x) // B
}
```
**Answer:** A: `20`, B: `10`.

### Q3: Three levels
```go
package main

import "fmt"

var x = 1

func main() {
	x := 2
	if true {
		x := 3
		if true {
			x := 4
			fmt.Println(x) // A
		}
		fmt.Println(x) // B
	}
	fmt.Println(x) // C
	other()        // D
}

func other() { fmt.Println(x) }
```
**Answer:** A `4`, B `3`, C `2`, D `1` (`other` sits at package level, so it sees the package `x`).

### Q4: No shadowing, just assignment
```go
package main

import "fmt"

var x = 10

func main() {
	if true {
		x = 20
		fmt.Println(x)
	}
	fmt.Println(x)
}
```
**Answer:** `20` then `20`. There's no `:=` or `var`, so the package variable is modified.

### Q5: Tricky mix
```go
package main

import "fmt"

var a = 100

func main() {
	a := 200
	b := 300
	if true {
		a := 400
		fmt.Println(a, b) // A
	}
	fmt.Println(a, b) // B
}
```
**Answer:** A `400 300`, B `200 300`. The package `a = 100` is never used.

### Q6: Shadowed in a loop
```go
package main

import "fmt"

func main() {
	total := 0
	for i := 0; i < 3; i++ {
		total := total + i // ?
		fmt.Println(total)
	}
	fmt.Println(total)
}
```
**Answer:** `0`, `1`, `2` inside the loop (each iteration's new `total` = outer `total (0)` + `i`) then `0` after. The outer `total` never changes.

---

## 9. Common shadowing patterns

### Pattern 1: parameter shadows a package variable ✅ safe

```go
package main

import "fmt"

var name = "Package"

func greet(name string) { // parameter hides the package variable
	fmt.Println("Hello,", name)
}

func main() {
	greet("Alice")     // Hello, Alice
	fmt.Println(name)  // Package
}
```

### Pattern 2: loop variable ✅ safe

```go
package main

import "fmt"

var i = 100

func main() {
	for i := 0; i < 3; i++ {
		fmt.Print(i, " ")
	}
	fmt.Println(i) // 0 1 2 100
}
```

### Pattern 3: `if x := ...; cond` ✅ idiomatic

The whole point is to create a short-lived variable.

### Pattern 4: re-declaring in an inner block ⚠️ confusing

```go
x := 10
if x > 5 {
	x := 20 // reader must notice this is NEW, not an update
	fmt.Println(x)
}
fmt.Println(x) // still 10
```

### Pattern 5: `err` re-declared in a block ❌ dangerous

See the next two sections.

---

## 10. When shadowing is fine

**1. Narrow-scope helper variables**

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	if n, err := strconv.Atoi("42"); err == nil {
		fmt.Println("parsed", n)
	}
	if n, err := strconv.Atoi("x"); err != nil { // reuse of n, err: different scope
		fmt.Println("failed", n, err)
	}
}
```

Each `n`, `err` lives only in its own `if`. No confusion.

**2. Parameter reuse**: `func (s *Server) handle(w http.ResponseWriter, r *http.Request)` type names as parameters often shadow imported package names (`url`, `path`), and that's harmless if you don't need the package in that function.

**3. Deliberate "copy" for goroutines/closures** (pre-Go 1.22 idiom): `i := i` inside a loop body created a per-iteration copy. Since Go 1.22 loop variables are per-iteration already, but you'll see this in older code.

---

## 11. When shadowing is dangerous

### Bug 1: the `err` that never arrived

```go
package main

import (
	"errors"
	"fmt"
)

func fetch() (string, error) { return "", errors.New("network down") }

func load() error {
	var err error
	if true {
		data, err := fetch() // ❌ := creates NEW data AND NEW err in this block
		_ = data
		_ = err
	}
	return err // returns the OUTER err: still nil! The failure was lost.
}

func main() {
	fmt.Println(load()) // <nil>  ← the caller thinks everything is fine
}
```

Output: `<nil>`. The error was swallowed. The fix: declare `data` beforehand and use `=`:

```go
package main

import (
	"errors"
	"fmt"
)

func fetch() (string, error) { return "", errors.New("network down") }

func load() error {
	var err error
	if true {
		var data string
		data, err = fetch() // ✅ assigns to the OUTER err
		_ = data
	}
	return err
}

func main() {
	fmt.Println(load()) // network down
}
```

### Bug 2: accumulating into a shadow

```go
sum := 0
for _, n := range nums {
	sum := sum + n // ❌ new sum every iteration
	_ = sum
}
// sum is still 0
```

Fix: `sum += n`.

### Bug 3: shadowing an imported package

```go
import "strings"

func f() {
	strings := []string{"a", "b"} // shadows the package
	// strings.ToUpper("x")        // ❌ now an error: strings is a slice!
}
```

### Bug 4: named results (Go stops you)

```go
// INTENTIONAL ERROR
package main

import "errors"

func f() (err error) {
	if true {
		err := errors.New("boom")
		return // ❌ compile error: result parameter err not in scope at return
	}
	return
}

func main() { f() }
```

The compiler catches this special case for you (naked `return` with a shadowed named result).

---

## 12. How to catch shadowing bugs

1. **Read `:=` carefully.** Ask: "Do I intend a *new* variable here?"
2. **Prefer `=` when updating**, declaring the variable once above.
3. **Use a linter.** The official (optional) analyzer:
   ```bash
   go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow@latest
   go vet -vettool=$(which shadow) ./...
   ```
   Many editors and `golangci-lint` also enable shadow checks.
4. **Give outer variables descriptive names** so an inner short name is unlikely to collide.
5. **Keep functions short.** Less code between declarations means fewer surprises.
6. **Trace on paper** for anything puzzling (Chapters 8–11 habit).

---

## 13. Exercises

### Exercise 1: Predict

```go
package main

import "fmt"

var v = "package"

func show() { fmt.Println(v) }

func main() {
	v := "function"
	{
		v := "block"
		fmt.Println(v)
	}
	fmt.Println(v)
	show()
}
```

<details><summary>Solution</summary>

```
block
function
package
```
</details>

### Exercise 2: Find and fix the bug
This function is supposed to count even numbers but always returns 0.

```go
package main

import "fmt"

func countEvens(nums []int) int {
	count := 0
	for _, n := range nums {
		if n%2 == 0 {
			count := count + 1
			_ = count
		}
	}
	return count
}

func main() { fmt.Println(countEvens([]int{1, 2, 3, 4})) }
```

<details><summary>Solution</summary>

`count := count + 1` declares a **new** `count` inside the `if`, which shadows the outer one. Change it to `count++` (or `count = count + 1`). Prints `2`.
</details>

### Exercise 3: Which `x`?
Label each `Println` with the value it prints.

```go
package main

import "fmt"

var x = 1

func main() {
	fmt.Println(x) // (a)
	x := 2
	fmt.Println(x) // (b)
	{
		fmt.Println(x) // (c)
		x := 3
		fmt.Println(x) // (d)
	}
	fmt.Println(x) // (e)
}
```

<details><summary>Solution</summary>

(a) `1`: the local `x` isn't declared yet, so the package `x` is found. (b) `2`. (c) `2`: block has no `x` yet, so the enclosing `main`'s. (d) `3`. (e) `2`.

Notice (a) vs (b): the *same name* refers to different variables at different lines of the same function!
</details>

### Exercise 4: The lost error
Rewrite the buggy `load` from section 11 so that the caller **does** receive the error, *without* naming an outer `err` at all. (Hint: return directly.)

<details><summary>Solution</summary>

```go
package main

import (
	"errors"
	"fmt"
)

func fetch() (string, error) { return "", errors.New("network down") }

func load() error {
	if _, err := fetch(); err != nil {
		return err
	}
	return nil
}

func main() { fmt.Println(load()) } // network down
```
Returning as soon as the error occurs removes the need for an outer variable entirely.
</details>

### Exercise 5 (challenge): Explain each output

```go
package main

import "fmt"

var n = 1

func inc() int { n++; return n }

func main() {
	n := 10
	n = inc()
	fmt.Println(n)
	fmt.Println(inc())
}
```

<details><summary>Solution</summary>

`inc` is package-level, so it works on the **package** `n`: 1 → 2, returns 2. In `main`, the local `n` (10) is assigned that 2 → prints `2`. The second `inc()` bumps the package `n` to 3 and returns `3`.
Output: `2` then `3`. The local `n` shadowed the package `n` in `main` only.
</details>

---

## 14. Quiz

1. Does shadowing overwrite the outer variable?
2. What is the difference between `x := 1` and `x = 1` inside a nested block?
3. In a chain of nested blocks each declaring `x`, which one is used?
4. Why is `data, err := f()` inside an `if` block risky when an `err` exists outside?
5. Name a tool that can flag shadowed variables.

<details><summary>Answers</summary>

1. No. It creates a separate variable; the outer one is untouched.
2. `:=` declares a new variable in the current scope; `=` assigns to the nearest existing one.
3. The innermost one.
4. `:=` creates a new `err` scoped to that block, so the outer `err` isn't set.
5. `go vet` with the `shadow` analyzer, or `golangci-lint`.
</details>

---

## 15. Summary

- **Shadowing** = an inner variable with the same name hides an outer one *within its scope only*.
- Two **separate variables** exist; the outer is untouched and reappears after the inner scope ends.
- **`:=` / `var`** declare (possibly shadowing); **`=`** assigns to the nearest existing variable.
- Lookup is **innermost-first**; the first match wins.
- Shadowing is fine for short-lived helpers (`if n, err := …`) and parameters, and dangerous for **accumulators** and **`err`**.
- Use `=` when you mean to update, keep functions short, and run a shadow linter.

### ➡️ What's next?

You've mastered where names live. [Chapter 13](13-function-types.md) looks at the different **kinds of functions** in Go, starting with standard (named) functions and their full anatomy.
