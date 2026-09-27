# Chapter 9: Local Scope and Block Scope

> **Goal of this chapter:** Learn that *every pair of curly braces* `{ }` creates its own scope. You'll master block scope in `if`, `for`, `switch`, and bare blocks, and learn how nested scopes are searched.

**Difficulty:** 🟡 Beginner–Intermediate  **Estimated time:** 1.5 hours  **Prerequisite:** [Chapter 8](08-what-is-scope.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The scopes Go has (big picture)](#2-the-scopes-go-has)
3. [What is a block?](#3-what-is-a-block)
4. [Block scope: the core idea](#4-block-scope-the-core-idea)
5. [`if` blocks: a full memory simulation](#5-if-blocks-a-full-memory-simulation)
6. [Where else blocks hide: `for`, `switch`, init statements](#6-where-else-blocks-hide)
7. [Nested blocks and the lookup order](#7-nested-blocks-and-the-lookup-order)
8. [Sibling blocks can't see each other](#8-sibling-blocks-cant-see-each-other)
9. [Declaration order matters inside a block](#9-declaration-order-matters)
10. [Why block scope is good design](#10-why-block-scope-is-good-design)
11. [Common mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)
13. [Quiz](#13-quiz)
14. [Summary](#14-summary)

---

## 1. What you will learn

- What a **block** is
- That blocks nest, and each has its own scope
- How variables declared in `if`, `for`, and `switch` disappear after the block
- How Go searches nested scopes for a name
- How to fix "undefined" errors caused by block scope
- Why keeping variables in the smallest possible scope makes code safer

---

## 2. The scopes Go has

From widest to narrowest:

```
┌─────────────────────────────────────────────────────────┐
│ UNIVERSE   : built-ins (int, string, true, len, nil …)  │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ PACKAGE  : top-level names of the package           │ │
│ │ ┌─────────────────────────────────────────────────┐ │ │
│ │ │ FUNCTION : parameters + top of function body    │ │ │
│ │ │ ┌─────────────────────────────────────────────┐ │ │ │
│ │ │ │ BLOCK : any { ... } inside it (nestable)    │ │ │ │
│ │ │ └─────────────────────────────────────────────┘ │ │ │
│ │ └─────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

(There's also *file scope* for `import` names, meaning an import is visible only in the file that imports it.)

Chapter 8 covered package and function scope. **Today: blocks.** A function body is technically just the outermost block of that function.

---

## 3. What is a block?

> A **block** is a sequence of statements enclosed in curly braces `{ }`.

Blocks appear in many places:

```go
package main

import "fmt"

func main() { // ← function block
	if true { // ← if block
		fmt.Println("in if")
	}

	for i := 0; i < 2; i++ { // ← for block (and the loop header has its own scope)
		fmt.Println("in for", i)
	}

	switch 1 { // ← each case clause is its own block
	case 1:
		fmt.Println("in case")
	}

	{ // ← a bare block: legal, and creates a scope!
		fmt.Println("in a bare block")
	}
}
```

Output:

```
in if
in for 0
in for 1
in case
in a bare block
```

Blocks can sit inside other blocks (**nesting**).

```
func main() {                 ← block A
    if cond {                 ← block B (inside A)
        for … {               ← block C (inside B)
        }
    }
}
```

---

## 4. Block scope: the core idea

> **A variable declared inside a block is visible only from its declaration to the closing brace of that block.**

```go
package main

import "fmt"

func main() {
	fmt.Println("start")

	{
		message := "I live inside the block"
		fmt.Println(message) // ✅ works
	}

	// fmt.Println(message) // ❌ undefined: message
	fmt.Println("end")
}
```

Output:

```
start
I live inside the block
end
```

At the closing `}` the variable goes out of scope.

---

## 5. `if` blocks: a full memory simulation

```go
package main

import "fmt"

var country = "Bangladesh" // package scope

func main() {
	age := 20 // function scope

	if age >= 18 {
		status := "mature" // block scope (inside the if)
		fmt.Println(status, country, age)
	}

	// fmt.Println(status) // ❌ undefined: status
}
```

Let's watch the "memory" (scopes) at each step.

**Step 1: package scope is created**

```
┌─────────────────────────┐
│ PACKAGE                 │
│  country = "Bangladesh" │
│  main    = <code>       │
└─────────────────────────┘
```

**Step 2: `main` starts, so function scope appears and `age := 20` executes**

```
┌─────────────────────────┐
│ PACKAGE                 │
│  country = "Bangladesh" │
└─────────────────────────┘
      ┌──────────────────┐
      │ main             │
      │  age = 20        │
      └──────────────────┘
```

**Step 3: the condition `age >= 18` is evaluated: `true`. Go enters the block.**
It creates a **new, nested scope** for it:

```
┌─────────────────────────┐
│ PACKAGE                 │
│  country = "Bangladesh" │
└─────────────────────────┘
      ┌──────────────────┐
      │ main             │
      │  age = 20        │
      │  ┌─────────────┐ │
      │  │ if block    │ │
      │  │ status =    │ │
      │  │   "mature"  │ │
      │  └─────────────┘ │
      └──────────────────┘
```

**Step 4: the print statement runs.** It looks up three names:
- `status` → found in the *if block* ✅
- `country` → not in the block, not in `main`, found in *package* ✅
- `age` → not in the block, found in `main` ✅

Prints `mature Bangladesh 20`.

**Step 5: the closing `}` is reached, so the block scope is destroyed**

```
      ┌──────────────────┐
      │ main             │
      │  age = 20        │
      └──────────────────┘     ← `status` no longer exists
```

**Step 6: any later use of `status` fails**: `undefined: status`.

---

## 6. Where else blocks hide

### 6.1 `for` loops

The loop variable belongs to the loop, not the surrounding function:

```go
package main

import "fmt"

func main() {
	for i := 0; i < 3; i++ {
		square := i * i // new variable each iteration
		fmt.Println(i, square)
	}
	// fmt.Println(i)      // ❌ undefined: i
	// fmt.Println(square) // ❌ undefined: square
}
```

Output:

```
0 0
1 1
2 4
```

> Each iteration gets a **fresh** `square`. Since Go 1.22, each iteration also gets a fresh `i` (a change that fixes a famous closure bug; see Chapter 20).

### 6.2 `if` with an init statement

```go
package main

import "fmt"

func main() {
	if n := 42; n%2 == 0 {
		fmt.Println(n, "is even") // ✅ n visible in the if AND any else
	} else {
		fmt.Println(n, "is odd")  // ✅ still visible here
	}
	// fmt.Println(n)            // ❌ undefined: n
}
```

The variable declared in the header (`n := 42`) is visible in **all** branches of the `if / else if / else` chain, but not after it.

### 6.3 `switch`

```go
package main

import "fmt"

func main() {
	switch day := 3; day {
	case 3:
		msg := "midweek"
		fmt.Println(day, msg)
	default:
		// msg is NOT visible here: each case clause has its own block
	}
}
```

### 6.4 Function parameters

Parameters live in the function's scope:

```go
func greet(name string) { // name is local to greet
	fmt.Println(name)
}
```

---

## 7. Nested blocks and the lookup order

Inner blocks can see **everything in the blocks that enclose them**. Outer blocks **cannot** see inside inner ones.

```go
package main

import "fmt"

func main() {
	a := 1 // level 1
	if true {
		b := 2 // level 2
		if true {
			c := 3 // level 3
			fmt.Println(a, b, c) // ✅ sees all three
		}
		fmt.Println(a, b) // ✅ sees a, b (c is gone)
		// fmt.Println(c) // ❌ undefined: c
	}
	fmt.Println(a) // ✅ only a
	// fmt.Println(b) // ❌ undefined: b
}
```

Output:

```
1 2 3
1 2
1
```

```
┌── main ────────────────────┐
│ a                          │
│ ┌── if block ────────────┐ │
│ │ b                      │ │
│ │ ┌── inner if block ──┐ │ │
│ │ │ c                  │ │ │
│ │ │ sees: a, b, c      │ │ │
│ │ └────────────────────┘ │ │
│ │ sees: a, b             │ │
│ └────────────────────────┘ │
│ sees: a                    │
└────────────────────────────┘
```

**One-way glass:** you can look *outward* from inside; you can't look *inward* from outside.

### The lookup order (recap)

Go looks for a name starting at the innermost block and moves outward:

```
innermost block → enclosing blocks → function → package → universe → ❌ undefined
```

The first match wins. If two blocks declare the same name, the inner one **shadows** the outer (details in Chapter 12).

---

## 8. Sibling blocks can't see each other

Two blocks at the same level are strangers:

```go
// INTENTIONAL ERROR
package main

import "fmt"

func main() {
	x := 5
	if x > 3 {
		label := "big"
		_ = label
	} else {
		fmt.Println(label) // ❌ undefined: label (belongs to the *other* block)
	}
}
```

If both branches need a value, declare it **before** the `if`, in the shared outer scope:

```go
package main

import "fmt"

func main() {
	x := 5
	var label string // declared in the shared outer scope
	if x > 3 {
		label = "big"
	} else {
		label = "small"
	}
	fmt.Println(label) // big
}
```

---

## 9. Declaration order matters

Inside a function, a variable exists only **after** its declaration line:

```go
// INTENTIONAL ERROR
package main

import "fmt"

func main() {
	fmt.Println(count) // ❌ undefined: count (not declared yet)
	count := 10
	fmt.Println(count)
}
```

(At package level the order doesn't matter; Go resolves all package-level names together. Inside functions it does.)

---

## 10. Why block scope is good design

Keep every variable in the **smallest scope that works**.

| Benefit | Why |
|---------|-----|
| **Fewer bugs** | A variable that can't be reached can't be misused |
| **Easier reading** | When you see `}`, you know the variable is finished with |
| **Name reuse** | `i`, `err`, `n` can be reused freely |
| **Memory friendliness** | Short-lived variables are cheap (they usually live on the stack) |

A classic example is the `if err := ...; err != nil` idiom:

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	if n, err := strconv.Atoi("12x"); err != nil {
		fmt.Println("bad number:", err)
	} else {
		fmt.Println("number is", n)
	}
	// err and n are gone, and can't be mixed up with later code
}
```

Compare with a version that leaves `err` alive for the rest of the function: more chances to accidentally reuse a stale error.

---

## 11. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Using a block variable after the block | `undefined: x` | Declare it in the outer scope |
| 2 | Declaring with `:=` inside the block when you meant to update an outer variable | Outer variable stays unchanged (**shadowing**) | Use `=` |
| 3 | Reading a variable from a sibling block | `undefined` | Move the declaration to a shared parent scope |
| 4 | Using a variable before declaring it | `undefined` | Declare first |
| 5 | Losing track of `}`, and so of where scopes end | odd `undefined` errors | Let `gofmt` indent; use editor bracket matching |

Mistake #2 is the sneakiest, because the code *compiles*:

```go
package main

import "fmt"

func main() {
	result := 0
	if true {
		result := 99 // ❌ NEW variable! meant: result = 99
		_ = result
	}
	fmt.Println(result) // 0, not 99
}
```

Output: `0`. This is exactly why Chapter 12 exists.

---

## 12. Exercises

### Exercise 1: Block scope detective
Which lines compile, and what's printed?

```go
// INTENTIONAL ERROR (L3)
package main

import "fmt"

func main() {
	a := 1
	{
		b := 2
		fmt.Println(a, b) // L1
	}
	fmt.Println(a) // L2
	fmt.Println(b) // L3
}
```

<details><summary>Solution</summary>

L1 ✅ prints `1 2`. L2 ✅ prints `1`. L3 ❌ `undefined: b`, so the program doesn't compile until L3 is removed.
</details>

### Exercise 2: Fix the scope error

```go
// INTENTIONAL ERROR
package main

import "fmt"

func main() {
	score := 75
	if score >= 50 {
		grade := "pass"
	} else {
		grade := "fail"
	}
	fmt.Println(grade)
}
```

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	score := 75
	var grade string
	if score >= 50 {
		grade = "pass"
	} else {
		grade = "fail"
	}
	fmt.Println(grade) // pass
}
```
</details>

### Exercise 3: Nested blocks
What does this print?

```go
package main

import "fmt"

func main() {
	x := 1
	if x == 1 {
		y := x + 1
		if y == 2 {
			z := y + 1
			fmt.Println(x, y, z)
		}
		fmt.Println(x, y)
	}
	fmt.Println(x)
}
```

<details><summary>Solution</summary>

```
1 2 3
1 2
1
```
</details>

### Exercise 4: Loop scope
Why does this fail, and how do you make it print the last value of `i`?

```go
// INTENTIONAL ERROR
package main

import "fmt"

func main() {
	for i := 0; i < 3; i++ {
	}
	fmt.Println(i)
}
```

<details><summary>Solution</summary>

`i` belongs to the `for` statement's scope, so it's `undefined` afterward. Declare it outside:

```go
package main

import "fmt"

func main() {
	var i int
	for i = 0; i < 3; i++ {
	}
	fmt.Println(i) // 3
}
```
</details>

### Exercise 5: The sneaky shadow
Predict the output, then explain.

```go
package main

import "fmt"

func main() {
	total := 10
	for i := 1; i <= 3; i++ {
		total := total + i
		_ = total
	}
	fmt.Println(total)
}
```

<details><summary>Solution</summary>

`10`. Inside the loop, `total := total + i` creates a **new** local `total` each iteration (initialized from the outer one) and the outer `total` never changes. To accumulate, write `total = total + i` (or `total += i`).
</details>

### Exercise 6 (challenge): Package vs. block

```go
package main

import "fmt"

var level = "package"

func show() { fmt.Println(level) }

func main() {
	level := "function"
	if true {
		level := "block"
		fmt.Println(level)
	}
	fmt.Println(level)
	show()
}
```

<details><summary>Solution</summary>

```
block
function
package
```
Three different variables named `level`, each visible at its own depth. `show()` is defined at package level, so it sees the package `level`.
</details>

---

## 13. Quiz

1. What creates a new scope besides a function?
2. Can code *outside* a block see variables declared inside it?
3. Can code *inside* a nested block see the outer block's variables?
4. Where is `n` visible in `if n := 5; n > 3 { } else { }`?
5. Why does declaring a variable in the smallest possible scope help?

<details><summary>Answers</summary>

1. Any `{ }` block: `if`, `for`, `switch` cases, or a bare block.
2. No.
3. Yes, until shadowed by a same-named inner variable.
4. In the condition, the `if` body, and the `else` body (not after).
5. It reduces accidental misuse and makes code easier to reason about.
</details>

---

## 14. Summary

- **Every `{ }` is a block, and every block is a scope.**
- A variable lives from its declaration to the block's closing `}`.
- Inner blocks **see outward**; outer code **can't see inward**. Sibling blocks can't see each other.
- `for`, `if` and `switch` headers introduce variables scoped to the statement.
- Lookup runs inner → outer → package → universe.
- Mixing up `:=` and `=` in an inner block creates an accidental new variable (**shadowing**), covered in Chapter 12.
- Keep variables in the **smallest scope** that works.

### ➡️ What's next?

[Chapter 10](10-package-scope.md) zooms out to the **package**: how to split code across files and folders, create your own packages, and control visibility with capital letters.
