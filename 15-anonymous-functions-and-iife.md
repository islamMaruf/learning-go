# Chapter 15: Anonymous Functions and IIFE

> **Goal of this chapter:** Learn functions that have **no name**, how to run one immediately (an **IIFE**), and why you'd ever want to.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 1.5 hours  **Prerequisite:** [Chapter 13](13-function-types.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What is an anonymous function?](#2-what-is-an-anonymous-function)
3. [Named vs anonymous, side by side](#3-named-vs-anonymous-side-by-side)
4. [Why an anonymous function can't "stand alone"](#4-why-an-anonymous-function-cant-stand-alone)
5. [IIFE: Immediately Invoked Function Expression](#5-iife)
6. ["Invoke" and "expression"](#6-invoke-and-expression)
7. [IIFE examples](#7-iife-examples)
8. [Where anonymous functions shine](#8-where-anonymous-functions-shine)
9. [A memory view](#9-a-memory-view)
10. [Common mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)
12. [Quiz](#12-quiz)
13. [Summary](#13-summary)

---

## 1. What you will learn

- What an anonymous function (function literal) is
- What **IIFE** stands for, and how to write one
- The difference between a *statement* and an *expression*
- Practical uses: one-off work, scope isolation, `defer`, and goroutines
- Why Go refuses a bare anonymous function statement

---

## 2. What is an anonymous function?

> An **anonymous function** is a function **without a name**.

You write `func`, skip the name, and continue with the parameter list and body. In the Go specification it's officially called a **function literal**, just like `42` is an integer literal and `"hi"` is a string literal.

```go
func(a, b int) {          // ← no name between `func` and `(`
	fmt.Println(a + b)
}
```

By itself that snippet isn't a complete statement. Go needs to know **what to do with it**: call it, store it, or pass it. That's what the rest of this chapter (and the next) is about.

---

## 3. Named vs anonymous, side by side

**Named function**: you declare it once, then call it by name:

```go
package main

import "fmt"

func add(a, b int) {
	fmt.Println(a + b)
}

func main() {
	add(5, 7) // call by name → 12
}
```

**Anonymous function**: there's no name to call, so you *invoke it right where it's written* by adding `(arguments)` after the closing brace:

```go
package main

import "fmt"

func main() {
	func(a, b int) {
		fmt.Println(a + b)
	}(5, 7) // ← invoke immediately with a=5, b=7 → 12
}
```

```
func(a, b int) { ... }   (5, 7)
└──── the function ────┘  └─ invoke with these arguments
```

Both print `12`. The difference is *reusability*: the named one can be called any number of times from anywhere in the package; the anonymous one is defined and used **in one place**.

---

## 4. Why an anonymous function can't "stand alone"

This is an error:

```go
// INTENTIONAL ERROR
package main

import "fmt"

func main() {
	func(a, b int) { // ❌ syntax error: function must have a name (at package level)
		fmt.Println(a + b)
	}
}
```

Two different reasons depending on where you write it:

1. **At package level:** every top-level declaration must have a name (`func name...`). There is nothing to refer to an unnamed one.
2. **Inside a function**, without invoking or storing it, the compiler complains `func literal evaluated but not used`. A function value that nobody can reach is useless, so Go rejects it.

> **Analogy:** Imagine hiring an employee who has no name, no desk, and no manager, and whom nobody has been told to talk to. They'd sit there forever without doing anything. To be useful they must be *given a task right now* (**invoke**) or *given a badge to be reached later* (**store in a variable**, Chapter 16).

The two ways to use an anonymous function:

| Way | Chapter |
|-----|---------|
| Invoke it **immediately** → **IIFE** | 15 (here) |
| **Store** it in a variable → **function expression** | 16 |
| **Pass** it as an argument (callback) | 17 |

---

## 5. IIFE

**IIFE** = **I**mmediately **I**nvoked **F**unction **E**xpression (pronounced "iffy").

```
Immediately  →  runs right now, right here
Invoked      →  called
Function     →  a function
Expression   →  something that produces a value (see below)
```

```go
package main

import "fmt"

func main() {
	func() {
		fmt.Println("I ran immediately!")
	}() // ← the () at the end is what makes it an IIFE
}
```

Output: `I ran immediately!`

### Anatomy

```
func  ( params )  { body }  ( arguments )
 │        │          │            │
 │        │          │            └── invocation: run it now
 │        │          └── what it does
 │        └── inputs (can be empty)
 └── keyword (no name follows!)
```

### Forgetting the invocation

```go
// INTENTIONAL ERROR
package main

func main() {
	func() {
		println("hi")
	} // ❌ func literal evaluated but not used
}
```

That trailing `()` is essential.

---

## 6. "Invoke" and "expression"

### Why "invoke" and not "call"?

They mean the same thing. **Call** is casual; **invoke** is the formal term ("to invoke" = to summon). Use whichever, but understand both when reading docs or interviews.

### What is an expression?

An **expression** is code that **evaluates to a value**. A **statement** is code that **does something** (and needn't produce a value).

| Code | Expression? | Value |
|------|-------------|-------|
| `5` | ✅ | `5` |
| `2 + 3` | ✅ | `5` |
| `x > 3` | ✅ | `true` / `false` |
| `add(2, 3)` | ✅ | whatever `add` returns |
| `func(a int) int { return a * 2 }` | ✅ | **a function value** |
| `func(a int) int { return a * 2 }(4)` | ✅ | `8` |
| `if x > 3 { ... }` | ❌ statement | none |
| `for ... { }` | ❌ statement | none |

So a function literal is an **expression** whose value is *a function*. Invoking it makes a bigger expression whose value is *the function's result*. Hence "Immediately Invoked Function **Expression**".

Because it's an expression, you can use it wherever a value is expected:

```go
package main

import "fmt"

func main() {
	x := func() int { return 42 }() // the IIFE's RESULT (42) is stored in x
	fmt.Println(x)                  // 42
}
```

Note: `x` holds `42`, **not** a function, because we invoked it (`()` at the end).

---

## 7. IIFE examples

### Example 1: with parameters

```go
package main

import "fmt"

func main() {
	func(a, b int) {
		c := a + b
		fmt.Println("Sum:", c)
	}(5, 7)
}
```

Output: `Sum: 12`

### Example 2: different arguments each time

```go
package main

import "fmt"

func main() {
	func(a, b int) { fmt.Println(a * b) }(3, 4)
	func(a, b int) { fmt.Println(a * b) }(10, 10)
}
```

Output: `12` then `100`. (If you're writing the same body twice you should really use a named function.)

### Example 3: with a string

```go
package main

import "fmt"

func main() {
	func(name string) {
		fmt.Println("Hello,", name)
	}("Asha")
}
```

### Example 4: returning a value

```go
package main

import "fmt"

func main() {
	area := func(w, h int) int {
		return w * h
	}(4, 5)

	fmt.Println("Area:", area) // 20
}
```

### Example 5: computing an initial value with multiple steps

```go
package main

import "fmt"

func main() {
	discount := func() float64 {
		// several steps to decide a value, without polluting main's scope
		base := 100.0
		rate := 0.15
		return base * rate
	}()

	fmt.Println("Discount:", discount) // 15
}
```

`base` and `rate` exist only inside the IIFE and vanish afterward. `discount` is the only thing that "escapes."

### Example 6: package-level IIFE for initialization

An IIFE is allowed in a package-level variable declaration:

```go
package main

import "fmt"

var config = func() map[string]string {
	m := map[string]string{}
	m["env"] = "dev"
	m["port"] = "8080"
	return m
}()

func main() {
	fmt.Println(config["env"], config["port"]) // dev 8080
}
```

This runs during package initialization (Chapter 14). It's a neat alternative to `init` when you only need to compute one variable.

---

## 8. Where anonymous functions shine

### 8.1 One-time work that needs private variables

```go
package main

import "fmt"

func main() {
	total := 0
	func() {
		tmp := 21 // exists only in here
		total = tmp * 2
	}()
	fmt.Println(total) // 42
	// fmt.Println(tmp) // ❌ undefined
}
```

Notice that the anonymous function can **read and modify `total`**, a variable from the surrounding function. That ability, to reach variables *outside* itself, is the seed of **closures** (Chapter 20).

### 8.2 Scope isolation

Keeps short-lived variables out of the surrounding scope. (You can also do that with a bare `{ }` block, but an IIFE can *return a value* too.)

### 8.3 `defer` with arguments

`defer` schedules a call to run when the surrounding function returns (full treatment in Chapter 34). Deferring an anonymous function is extremely common:

```go
package main

import "fmt"

func main() {
	defer func() {
		fmt.Println("cleanup done")
	}()

	fmt.Println("working...")
}
```

Output:

```
working...
cleanup done
```

`defer` requires a *call expression*, and `func(){...}()` is one. The trailing `()` doesn't run the function now: `defer` records the call and *postpones* it until `main` returns.

### 8.4 Goroutines

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(1)

	go func() { // launch an anonymous function concurrently
		defer wg.Done()
		fmt.Println("hello from a goroutine")
	}()

	wg.Wait()
}
```

Anonymous functions are the standard way to start a goroutine (`go func() { ... }()`) for a small piece of work (Chapters 36 and 64–70).

### 8.5 Callbacks (preview)

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	nums := []int{5, 2, 8, 1}
	sort.Slice(nums, func(i, j int) bool { return nums[i] < nums[j] })
	fmt.Println(nums) // [1 2 5 8]
}
```

`sort.Slice` takes a function that decides ordering. Writing a whole named function for a one-line comparison would be noise. (Chapter 17.)

---

## 9. A memory view

```go
package main

import "fmt"

func main() {
	x := 10
	func(n int) {
		y := n * 2
		fmt.Println(x, y)
	}(5)
}
```

```
Step 1: main starts.  x = 10
┌──────────────┐
│ main: x=10   │
└──────────────┘

Step 2: the function literal is invoked with n=5 → a NEW frame, like any call
┌──────────────────────┐
│ literal: n=5, y=10   │  ← can also see x from main's scope
├──────────────────────┤
│ main: x=10           │
└──────────────────────┘
          prints "10 10"

Step 3: the literal returns → its frame disappears
┌──────────────┐
│ main: x=10   │
└──────────────┘
```

It behaves exactly like a named function, except (a) it has no name in package scope, and (b) it can see the **enclosing function's** local variables (`x`).

---

## 10. Common mistakes

| # | Mistake | What happens | Fix |
|---|---------|--------------|-----|
| 1 | Forgetting the trailing `()` | `func literal evaluated but not used` | Add `(args)` |
| 2 | Wrong number of arguments in the invocation | `not enough arguments in call to func literal` | Match parameters |
| 3 | Writing an anonymous function at package level as a statement | `syntax error: unexpected (, expected name` | Assign it: `var f = func(){...}` |
| 4 | Putting `()` and expecting a function back | You get the **result** (since it ran) | Omit `()` to store the function itself (Chapter 16) |
| 5 | Copy-pasting the same anonymous function many times | Duplication | Use a named function |
| 6 | Very long anonymous functions | Hard to read | Extract into a named function |
| 7 | Expecting the IIFE's variables to be visible afterward | They're local | Return what you need |

---

## 11. Exercises

### Exercise 1: Basic IIFE
Write an IIFE that prints "Welcome to Go!".

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	func() {
		fmt.Println("Welcome to Go!")
	}()
}
```
</details>

### Exercise 2: With parameters
Write an IIFE taking `name string, age int` and printing `Asha is 30 years old`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	func(name string, age int) {
		fmt.Printf("%s is %d years old\n", name, age)
	}("Asha", 30)
}
```
</details>

### Exercise 3: Returning a value
Use an IIFE to compute `10!` (factorial) and store it in `result`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	result := func(n int) int {
		f := 1
		for i := 2; i <= n; i++ {
			f *= i
		}
		return f
	}(10)
	fmt.Println(result) // 3628800
}
```
</details>

### Exercise 4: Predict

```go
package main

import "fmt"

func main() {
	n := 1
	func() {
		n = n + 10
	}()
	func(n int) {
		n = n + 100
	}(n)
	fmt.Println(n)
}
```

<details><summary>Solution</summary>

The first IIFE modifies the **outer** `n` (it has no parameter or local named `n`) → `n = 11`. The second IIFE receives a **copy** of `n` (11) as its own parameter `n` and changes only that copy. Outer `n` stays **11**.
Output: `11`.
</details>

### Exercise 5: Package-level IIFE
Create a package variable `primes` (a `[]int`) filled with the primes below 30 using an IIFE.

<details><summary>Solution</summary>

```go
package main

import "fmt"

var primes = func() []int {
	var out []int
	for n := 2; n < 30; n++ {
		isPrime := true
		for d := 2; d*d <= n; d++ {
			if n%d == 0 {
				isPrime = false
				break
			}
		}
		if isPrime {
			out = append(out, n)
		}
	}
	return out
}()

func main() { fmt.Println(primes) } // [2 3 5 7 11 13 17 19 23 29]
```
</details>

### Exercise 6 (challenge): Defer order
What does this print?

```go
package main

import "fmt"

func main() {
	for i := 1; i <= 3; i++ {
		defer func() { fmt.Println("deferred", i) }()
	}
	fmt.Println("done")
}
```

<details><summary>Solution</summary>

```
done
deferred 3
deferred 2
deferred 1
```
Deferred calls run last-in, first-out. With Go 1.22+ each loop iteration has its own `i`, so each closure sees `3`, `2`, `1`. (In Go ≤ 1.21 all three would have printed `deferred 4`, a famous gotcha explained in Chapter 20.)
</details>

---

## 12. Quiz

1. What is another name for an anonymous function in the Go spec?
2. What does IIFE stand for?
3. What does `x := func() int { return 5 }()` store in `x`?
4. Why does a bare, un-invoked function literal fail to compile?
5. Can an anonymous function read variables of the enclosing function?

<details><summary>Answers</summary>

1. Function literal.
2. Immediately Invoked Function Expression.
3. The integer `5` (the *result* of invoking it).
4. It's a function value that's never used or stored: `func literal evaluated but not used`.
5. Yes. This is what makes closures possible.
</details>

---

## 13. Summary

- An **anonymous function** (function literal) has **no name**: `func(params) { body }`.
- It must be **used** at once: **invoked** (`}(args)`, an IIFE), **stored** (Chapter 16), or **passed** (Chapter 17).
- An **IIFE** runs immediately; its result can initialize a variable, and its local variables stay private.
- It's an **expression**: it evaluates to a function value; invoking it evaluates to the result.
- Everyday uses: `defer func() {...}()`, `go func() {...}()`, callbacks like `sort.Slice`, and one-shot initialization.
- Anonymous functions can **see the enclosing function's variables**: the doorway to closures.

### ➡️ What's next?

In [Chapter 16](16-function-expressions.md) we stop invoking immediately and instead **store** an anonymous function in a variable: a **function expression**.
