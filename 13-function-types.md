# Chapter 13: Function Types — Standard (Named) Functions

> **Goal of this chapter:** Get the *map* of all the kinds of functions in Go, then study the most common one in depth: the **standard (named) function**. Knowing the official names for things also makes you sound (and think) like a professional.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 1–1.5 hours  **Prerequisite:** [Chapters 4–6](04-introduction-to-functions.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why terminology matters](#2-why-terminology-matters)
3. [The map: kinds of functions](#3-the-map-kinds-of-functions)
4. [What is a standard (named) function?](#4-what-is-a-standard-named-function)
5. [Anatomy of a standard function](#5-anatomy-of-a-standard-function)
6. [Examples](#6-examples)
7. [Memory behavior of standard functions](#7-memory-behavior)
8. [Functions are values too (a teaser)](#8-functions-are-values-too)
9. [Naming and documenting functions](#9-naming-and-documenting-functions)
10. [Common mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)
12. [Quiz](#12-quiz)
13. [Summary](#13-summary)

---

## 1. What you will learn

- The vocabulary for the different kinds of functions you'll meet in Go
- The full anatomy of a standard function (every part has a name)
- How Go stores named functions in memory and why you can call them from anywhere in the package
- How to name and document functions the Go way

---

## 2. Why terminology matters

Programmers communicate in precise words. Compare:

- ❌ "Um, there's this function thingy that runs by itself and doesn't take anything?"
- ✅ "That's the `init` function."

Knowing the terms helps you:

| Situation | How it helps |
|-----------|--------------|
| **Interviews** | "What's the difference between a closure and an anonymous function?" is a very common question |
| **Reading docs** | Documentation uses these words constantly |
| **Code review** | "This should be a method" or "make this a higher-order function" are precise, quick comments |
| **Searching** | You can't google what you can't name |

---

## 3. The map: kinds of functions

The categories overlap (a function can be *several* of these at once), so treat this as a **vocabulary list** rather than a strict classification.

| # | Kind | One-line description | Chapter |
|---|------|----------------------|---------|
| 1 | **Standard / named function** | Has a name, declared with `func name() {}` | **13** (this one) |
| 2 | **`init` function** | Special: runs automatically before `main` | 14 |
| 3 | **Anonymous function** | A function with no name | 15 |
| 4 | **IIFE** | Anonymous function invoked immediately | 15 |
| 5 | **Function expression** | A function stored in a variable | 16 |
| 6 | **Higher-order function** | Takes functions as arguments and/or returns one | 17 |
| 7 | **First-order function** | Doesn't take/return functions | 17 |
| 8 | **Callback** | A function passed to another function to be called later | 17 |
| 9 | **Closure** | A function that *remembers* variables from where it was created | 20 |
| 10 | **Variadic function** | Accepts any number of trailing arguments | 6 |
| 11 | **Deferred function** | Scheduled to run when the surrounding function returns | 34 |
| 12 | **Receiver function (method)** | A function attached to a type | 22 |
| 13 | **Goroutine function** | A function run concurrently with `go` | 36 |

That's a lot! 😱 Don't worry, we take them one at a time, and each new one builds on the previous.

```
You are here ▼
[1 Standard] → [2 init] → [3-4 Anonymous & IIFE] → [5 Expressions] → [6-8 Higher-order]
        → [9 Closures] → [12 Methods] → [11 defer] → [13 goroutines]
```

---

## 4. What is a standard (named) function?

> A **standard function**, also called a **named function**, is a function **declared with a name** at package level using the `func` keyword.

Everything you've written so far, `add`, `greet`, `printMenu`, `main`, is a standard function.

"Standard" because it's the **default, ordinary** form. "Named" because it **has a name** you use to call it. They're two words for the same thing.

The key property: **its name is declared in package scope**, so once the compiler has read the package, any function in that package can call it, in any order (Chapter 11).

---

## 5. Anatomy of a standard function

```go
//   keyword  name    parameters              results
//     │       │         │                       │
      func    add(number1 int, number2 int)     int {
	sum := number1 + number2   // ← body
	return sum
}
```

| Part | Example | Rules |
|------|---------|-------|
| **Keyword** | `func` | Always required |
| **Name** | `add` | Identifier. Uppercase first letter = exported |
| **Parameter list** | `(number1 int, number2 int)` | Zero or more `name type` pairs. Must have parentheses even if empty |
| **Result list** | `int` or `(int, error)` | Optional. Zero, one, or several types |
| **Body** | `{ ... }` | Braces required; `{` on the same line as `func` |

### Full syntax forms

```go
func name()                              { }   // no in, no out
func name(a int)                         { }   // in only
func name() int                          { return 1 }   // out only
func name(a, b int) int                  { return a + b }
func name(a int) (int, error)            { return a, nil }
func name(a int) (result int, err error) { return }      // named results
func name(prefix string, nums ...int)    { }             // variadic
```

### The "signature"

Everything except the body is the function's **signature**: its name, parameter types, and result types. Two functions have the **same type** if their parameters and results match (names don't matter). This becomes important in Chapters 16–17, where signatures become *types* you can use in variables.

For instance both `add` and `subtract` below have the type `func(int, int) int`:

```go
func add(a, b int) int      { return a + b }
func subtract(x, y int) int { return x - y }
```

---

## 6. Examples

### Example 1: Basic

```go
package main

import "fmt"

func add(number1 int, number2 int) int {
	return number1 + number2
}

func main() {
	fmt.Println(add(10, 20)) // 30
}
```

### Example 2: Several standard functions cooperating

```go
package main

import "fmt"

func square(n int) int { return n * n }

func cube(n int) int { return n * square(n) }

func describe(n int) {
	fmt.Printf("%d: square=%d cube=%d\n", n, square(n), cube(n))
}

func main() {
	for i := 1; i <= 3; i++ {
		describe(i)
	}
}
```

Output:

```
1: square=1 cube=1
2: square=4 cube=8
3: square=9 cube=27
```

### Example 3: Multiple returns

```go
package main

import "fmt"

func minMax(nums ...int) (min, max int) {
	min, max = nums[0], nums[0]
	for _, n := range nums {
		if n < min {
			min = n
		}
		if n > max {
			max = n
		}
	}
	return
}

func main() {
	lo, hi := minMax(4, 9, 1, 7)
	fmt.Println(lo, hi) // 1 9
}
```

### Example 4: No return value

```go
package main

import "fmt"

func printBox(text string) {
	border := "+" + repeat("-", len(text)+2) + "+"
	fmt.Println(border)
	fmt.Println("| " + text + " |")
	fmt.Println(border)
}

func repeat(s string, n int) string {
	out := ""
	for i := 0; i < n; i++ {
		out += s
	}
	return out
}

func main() { printBox("Hello Go") }
```

Output:

```
+----------+
| Hello Go |
+----------+
```

Note `printBox` uses `repeat`, which is defined *after* it. Fine for standard functions.

---

## 7. Memory behavior

Recall the two phases from Chapter 11.

**Phase 1: declaration (compile time).** The compiler reads all `func name(...) { ... }` declarations and stores each function's **machine code** in the program's **code segment** (read-only memory). The **name** becomes an entry in the package scope that points to that code.

```
CODE SEGMENT (read-only)                PACKAGE SCOPE (names)
┌─────────────────────────┐             ┌──────────────────────┐
│ 0x400100: <add's code>  │◄────────────│ add    → 0x400100    │
│ 0x400180: <main's code> │◄────────────│ main   → 0x400180    │
└─────────────────────────┘             └──────────────────────┘
```

**Phase 2: execution.** `main()` runs. When it hits `add(10, 20)`:

1. Look up the name `add` in scope → address `0x400100`.
2. Create a **stack frame** for this call; copy in `10` and `20`.
3. Jump to the code at that address.
4. On `return`, copy the result back, destroy the frame, resume `main`.

```
main's frame:  result := add(10,20)
                    │  (1) find add's address
                    ▼
              add's code (shared, exists once)
                    │  (2) run with its OWN frame: number1=10, number2=20
                    ▼
              return 30  → destroy frame → result = 30
```

**Key insight:** the **code exists once**; each *call* creates a fresh set of variables (frame). That's why the same function can be called again and again, and even *recursively*, without the calls interfering.

---

## 8. Functions are values too

In Go, a function name isn't only something you can call. It's also a **value** you can assign, pass, and return. Peek:

```go
package main

import "fmt"

func add(a, b int) int { return a + b }

func main() {
	f := add // no parentheses: we're copying the function, not calling it
	fmt.Println(f(2, 3)) // 5
	fmt.Printf("%T\n", f) // func(int, int) int
}
```

`add` (no parentheses) is a value of type `func(int, int) int`. This idea, that functions are **first-class values**, powers the next several chapters.

---

## 9. Naming and documenting functions

**Naming**
- Use **camelCase**: `calculateTotal`. Exported: `CalculateTotal`.
- Start with a **verb** for actions: `saveUser`, `parseConfig`, `sendEmail`.
- Booleans read like questions: `isValid`, `hasPermission`, `canRetry`.
- Constructors are conventionally `NewThing`.
- Don't stutter with the package name: `user.Save`, not `user.SaveUser`.
- Keep names short in short scopes (`n`, `i`, `err`) and descriptive in long ones.

**Documenting** (`go doc` reads these): a comment directly above the function, starting with its name.

```go
// CalculateTotal returns the sum of prices, adding tax at the given rate
// (0.15 means 15%). It returns 0 for an empty slice.
func CalculateTotal(prices []float64, taxRate float64) float64 {
	total := 0.0
	for _, p := range prices {
		total += p
	}
	return total * (1 + taxRate)
}
```

---

## 10. Common mistakes

| # | Mistake | Explanation / fix |
|---|---------|-------------------|
| 1 | Treating "function" as one concept | There are many kinds; learn the vocabulary |
| 2 | Using `f` instead of `f()` (or vice versa) | `f` is the function *value*; `f()` *calls* it |
| 3 | Declaring a named function inside another function | `func inner() {}` inside a function is illegal. Use an anonymous function (`inner := func() {}`, Chapters 15–16) |
| 4 | Duplicate function names in a package | Each package-level name must be unique across all files |
| 5 | Vague names (`doIt`, `process`, `handle`) | Name what it does |
| 6 | Very long functions | If you can't describe it in one sentence, split it |
| 7 | Forgetting to document exported functions | Add a comment starting with the name |

Mistake #3 shows up often:

```go
// INTENTIONAL ERROR
package main

func main() {
	func helper() { // ❌ syntax error: unexpected name helper, expected (
	}
}
```

---

## 11. Exercises

### Exercise 1: Identify the parts
For `func area(w, h float64) (float64, error) { ... }` name: keyword, function name, parameters, results, signature.

<details><summary>Solution</summary>

Keyword: `func`. Name: `area`. Parameters: `w, h float64`. Results: `(float64, error)`. Signature: `area(w, h float64) (float64, error)`; its *type* is `func(float64, float64) (float64, error)`.
</details>

### Exercise 2: Same type?
Which of these have the same function type?

```go
func a(x int) int          { return x }
func b(y int) int          { return y * 2 }
func c(x int) string       { return "" }
func d(x, y int) int       { return x }
```

<details><summary>Solution</summary>

`a` and `b` share `func(int) int` (names don't matter). `c` returns a `string`, and `d` takes two parameters, so they're different types.
</details>

### Exercise 3: Write and document
Write an exported, documented function `Clamp(n, lo, hi int) int` that limits `n` to the range `[lo, hi]`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

// Clamp returns n limited to the inclusive range [lo, hi].
func Clamp(n, lo, hi int) int {
	if n < lo {
		return lo
	}
	if n > hi {
		return hi
	}
	return n
}

func main() {
	fmt.Println(Clamp(15, 0, 10), Clamp(-3, 0, 10), Clamp(5, 0, 10)) // 10 0 5
}
```
</details>

### Exercise 4: Memory trace
For `result := cube(3)` (where `cube` calls `square`, from Example 2), list the frames at the deepest point, and the final value.

<details><summary>Solution</summary>

Frames (top → bottom): `square(n=3)`, `cube(n=3)`, `main`. `square` returns 9, `cube` returns `3*9 = 27`. Result: `27`.
</details>

### Exercise 5 (interview): Name the difference
What's the difference between a *standard function* and an *anonymous function*?

<details><summary>Solution</summary>

A standard (named) function is declared at package level with a name (`func add(...)`), callable by that name anywhere in the package. An anonymous function has no name; it's written inline as an expression (`func(...) {...}`), and must be invoked immediately or stored/passed as a value (Chapters 15–16).
</details>

### Exercise 6 (challenge): Function tables
Store `add`, `subtract`, and `multiply` in a `map[string]func(int, int) int` and call them by name.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func add(a, b int) int      { return a + b }
func subtract(a, b int) int { return a - b }
func multiply(a, b int) int { return a * b }

func main() {
	ops := map[string]func(int, int) int{
		"add":      add,
		"subtract": subtract,
		"multiply": multiply,
	}
	for _, name := range []string{"add", "subtract", "multiply"} {
		fmt.Println(name, ops[name](6, 3))
	}
}
```
Output: `add 9`, `subtract 3`, `multiply 18`. (Maps are covered later; the point is that functions are values.)
</details>

---

## 12. Quiz

1. Is "standard function" different from "named function"?
2. Where is a function's machine code stored?
3. Does calling a function twice create two copies of its code?
4. What is a function's *signature*?
5. Can you declare `func helper() {}` inside `main`?

<details><summary>Answers</summary>

1. No, they're the same thing.
2. In the program's code segment (read-only memory).
3. No. The code exists once; each call creates a new stack frame for its variables.
4. Its name, parameter types, and result types (everything except the body).
5. No. Use `helper := func() { ... }`.
</details>

---

## 13. Summary

- Go has many *kinds* of functions (standard, anonymous, closures, methods, …). Learn the names; you'll meet them all.
- A **standard/named function** is declared at package level with `func name(params) results { body }`.
- Its **signature** (parameters + results) determines its **type**.
- The **code** is stored once; **each call** gets its own frame.
- Functions are **first-class values** (`f := add`), the seed of the next chapters.
- Name with verbs, camelCase; document exported functions.

### ➡️ What's next?

[Chapter 14](14-the-init-function.md) covers a very special standard function that you can *never* call yourself: **`init`**.
