# Chapter 16: Function Expressions — Storing Functions in Variables

> **Goal of this chapter:** Learn that a function can be **stored in a variable** (a *function expression*), called through that variable, and passed around like any other value. You'll also learn the important **ordering rules** that differ between package scope and local scope.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 1.5–2 hours  **Prerequisite:** [Chapter 15](15-anonymous-functions-and-iife.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [From IIFE to function expression](#2-from-iife-to-function-expression)
3. [Assigning a function to a variable](#3-assigning-a-function-to-a-variable)
4. [The type of a function variable](#4-the-type-of-a-function-variable)
5. [The order rule: local scope](#5-the-order-rule-local-scope)
6. [The order rule: package scope](#6-the-order-rule-package-scope)
7. [Memory simulation](#7-memory-simulation)
8. [Shadowing a named function](#8-shadowing-a-named-function)
9. [Recursion with function expressions](#9-recursion-with-function-expressions)
10. [Reassigning, `nil` functions, and function tables](#10-reassigning-nil-functions-and-function-tables)
11. [Named function vs function expression: which to use?](#11-named-function-vs-function-expression)
12. [Common mistakes](#12-common-mistakes)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- What a **function expression** is
- How to store a function in a variable and call it through that variable
- Why order matters for a local function expression but not for a package-level named function
- How a function expression can *shadow* a named function
- How to write a **recursive** function expression
- How function values enable tables of behavior (dispatch maps)

---

## 2. From IIFE to function expression

In Chapter 15 we invoked an anonymous function immediately:

```go
func(a, b int) {
	fmt.Println(a + b)
}(5, 7) // ← invoked right away
```

Now we **don't invoke it**. We store it in a variable instead, to use later:

```go
add := func(a, b int) {
	fmt.Println(a + b)
}
// ← no () at the end: we are storing the function, not running it
```

> **Function expression** = an anonymous function **assigned to a variable** (or otherwise used as a value).

| | IIFE | Function expression |
|--|------|---------------------|
| Runs when? | Right where it's written | Whenever you call the variable |
| Trailing `()` after `}` | ✅ yes | ❌ no |
| Variable holds | The function's **result** | The **function itself** |
| Reusable? | No | Yes: call it many times |

---

## 3. Assigning a function to a variable

```go
package main

import "fmt"

func main() {
	add := func(a, b int) {
		c := a + b
		fmt.Println(c)
	}

	add(2, 3)  // 5
	add(10, 20) // 30
	add(100, 1) // 101
}
```

Output:

```
5
30
101
```

**Step by step:**

1. `func(a, b int) {...}` is evaluated to a **function value**. Nothing runs yet.
2. `add :=` stores that value in the variable `add`.
3. `add(2, 3)` reads the variable, finds the function, and calls it with `a=2, b=3`.

To Go, `add` is now *a variable whose value happens to be callable*. The call syntax `add(2, 3)` is identical to calling a named function.

### It can return values too

```go
package main

import "fmt"

func main() {
	multiply := func(a, b int) int {
		return a * b
	}

	fmt.Println(multiply(6, 7)) // 42
}
```

---

## 4. The type of a function variable

Every function value has a **type** determined by its signature (Chapter 13):

```go
package main

import "fmt"

func main() {
	add := func(a, b int) int { return a + b }
	greet := func(name string) { fmt.Println("Hi", name) }

	fmt.Printf("%T\n", add)   // func(int, int) int
	fmt.Printf("%T\n", greet) // func(string)
}
```

You can spell the type out when declaring the variable:

```go
var op func(int, int) int // a variable that can hold any func(int,int) int
op = func(a, b int) int { return a - b }
```

And you can give the type a **name** to make signatures readable:

```go
package main

import "fmt"

type BinaryOp func(int, int) int // "a BinaryOp is any func(int, int) int"

func apply(op BinaryOp, a, b int) int { return op(a, b) }

func main() {
	fmt.Println(apply(func(a, b int) int { return a + b }, 3, 4)) // 7
	fmt.Println(apply(func(a, b int) int { return a * b }, 3, 4)) // 12
}
```

Since the type is part of the variable, you **can't** assign a mismatching function:

```go
// INTENTIONAL ERROR
package main

func main() {
	f := func(a int) int { return a }
	f = func(a, b int) int { return a + b } // ❌ cannot use ... as func(a int) int value in assignment
	_ = f
}
```

---

## 5. The order rule: local scope

Inside a function, code runs **top to bottom**, one statement at a time. A local variable exists only **after** its declaration line runs (Chapter 9). A function expression stored in a local variable is no exception.

### ❌ Call before define

```go
// INTENTIONAL ERROR
package main

import "fmt"

func main() {
	add(2, 3) // ❌ undefined: add   (the variable doesn't exist yet)

	add := func(a, b int) {
		fmt.Println(a + b)
	}
}
```

### ✅ Define, then call

```go
package main

import "fmt"

func main() {
	add := func(a, b int) {
		fmt.Println(a + b)
	}

	add(2, 3) // 5
}
```

**Rule:** *Local function expression → define before you call.*

---

## 6. The order rule: package scope

At **package level** you can also write a function expression, using `var`:

```go
package main

import "fmt"

var sum = func(a, b int) {
	fmt.Println(a + b)
}

func main() {
	sum(5, 7) // 12
}
```

And here's the twist: it works **regardless of the position** of the declaration in the file, even if `sum` is declared *below* `main`:

```go
package main

import "fmt"

func main() {
	sum(5, 7) // ✅ still works
}

var sum = func(a, b int) {
	fmt.Println(a + b)
}
```

Output: `12`

**Why?** Recall the two-phase model (Chapter 11 and 14):

1. Package-level declarations are all collected at compile time.
2. **Package-level variables are initialized (including evaluating `func(...) {...}` and storing it in `sum`) before `main` starts.**

So by the time `main` runs, `sum` already holds its function.

```
        Program start
             │
   ┌─────────▼──────────┐
   │ initialize package │  ← `sum` is assigned its function here
   │ variables          │
   └─────────┬──────────┘
             │
        run init()s
             │
        run main()   ← `sum` is ready
```

### Summary of ordering

| Kind of function | Where | Order matters? |
|------------------|-------|----------------|
| Named function `func f() {}` | package level | ❌ No |
| Function expression `var f = func() {}` | package level | ❌ No (initialized before `main`) |
| Function expression `f := func() {}` | inside a function | ✅ **Yes**: define before use |

### A caveat: initialization order between package variables

If one package-level function expression *uses another package-level variable* Go handles the dependency automatically. But if two package-level variables refer to *each other* in their initializers, you get:

```go
// INTENTIONAL ERROR
package main

var a = func() int { return b() }
var b = func() int { return a() } // ❌ initialization cycle: a refers to b refers to a

func main() {}
```

Named functions don't have this problem (`func a() int { return b() }` and `func b() int { return a() }` are fine, since they're declarations rather than initialized variables).

---

## 7. Memory simulation

```go
package main

import "fmt"

var greet = func() { fmt.Println("hello") } // package level

func main() {
	x := 10
	double := func(n int) int { return n * 2 } // local
	fmt.Println(double(x))
	greet()
}
```

**Step 1: package scope: variables initialized, before `main`**

```
┌────────────────────────────────────┐
│ PACKAGE                            │
│  greet = ► <code: prints hello>    │  ← variable holding a function
│  main  = ► <code>                  │
└────────────────────────────────────┘
```

**Step 2: `main` starts; `x := 10`**

```
   ┌─────────────────────────────┐
   │ main   x = 10               │
   └─────────────────────────────┘
```

**Step 3: `double := func...` executes. The function value is created and stored in local `double`**

```
   ┌─────────────────────────────────────────┐
   │ main   x = 10                           │
   │        double = ► <code: n*2>           │
   └─────────────────────────────────────────┘
```

Notice `double` is a *variable that points at code*. Writing `func...` did not run anything.

**Step 4: `double(x)`.** Look up `double` in `main`'s scope → get the code → run with a new frame `n = 10` → returns `20`. Frame destroyed.

**Step 5: `greet()`.** Look up `greet`: not local → package scope → found → run.

**Step 6: `main` ends.** `main`'s locals (`x`, `double`) vanish; the package `greet` lives until the program ends.

Output:

```
20
hello
```

---

## 8. Shadowing a named function

Because a function expression is *just a variable*, the shadowing rules from Chapter 12 apply.

```go
package main

import "fmt"

func add(a, b int) { // named, package scope
	fmt.Println("package add:", a+b)
}

func main() {
	add(1, 2) // no local `add` exists yet → uses the PACKAGE function

	add := func(a, b int) { // new local variable: shadows the package function
		fmt.Println("local add:", a+b)
	}

	add(3, 4) // now the LOCAL one
}
```

Output:

```
package add: 3
local add: 7
```

This is the same story as `x := ...` in Chapter 12: **same name, different variable, decided by which scope the lookup finds first.**

It also fixes a subtle misunderstanding: in this program, calling `add` *before* the local definition doesn't produce an error, because a *package-level* `add` exists. Errors only happen when nothing with that name is visible yet (section 5).

> ⚠️ **Advice:** avoid shadowing named functions with local variables. It's legal but confusing. Give the local a different name (`localAdd`).

---

## 9. Recursion with function expressions

A **recursive** function calls itself. With a *named* function that's trivial:

```go
func factorial(n int) int {
	if n <= 1 {
		return 1
	}
	return n * factorial(n-1)
}
```

With a function expression there's a catch. Inside the literal, the variable you're assigning to **isn't in scope yet** (the declaration takes effect *after* the whole statement):

```go
// INTENTIONAL ERROR
package main

func main() {
	fact := func(n int) int {
		if n <= 1 {
			return 1
		}
		return n * fact(n-1) // ❌ undefined: fact
	}
	_ = fact
}
```

**Fix:** declare the variable first, then assign:

```go
package main

import "fmt"

func main() {
	var fact func(int) int // 1) declare the variable (value: nil)

	fact = func(n int) int { // 2) assign the function; `fact` is now in scope
		if n <= 1 {
			return 1
		}
		return n * fact(n-1)
	}

	fmt.Println(fact(5)) // 120
}
```

The literal refers to the variable `fact` (not to itself directly), so it picks up whatever `fact` holds when called: the function itself. (This "captures a variable" behavior is a closure, Chapter 20.)

---

## 10. Reassigning, `nil` functions, and function tables

### 10.1 Reassigning a function variable

Because it's a variable, you can point it at different functions, as long as the signature matches:

```go
package main

import "fmt"

func main() {
	greet := func() { fmt.Println("Hello") }
	greet() // Hello

	greet = func() { fmt.Println("Namaste") }
	greet() // Namaste
}
```

### 10.2 The zero value is `nil`; calling it panics

```go
package main

import "fmt"

func main() {
	var f func(int) int
	fmt.Println(f == nil) // true

	defer func() { fmt.Println("recovered:", recover()) }()
	f(3) // ❌ panic: runtime error: invalid memory address or nil pointer dereference
}
```

Always ensure a function variable is set before calling, or check `if f != nil`.

### 10.3 Function tables (dispatch maps)

Because functions are values, you can store them in maps and slices:

```go
package main

import "fmt"

func main() {
	calc := map[string]func(float64, float64) float64{
		"+": func(a, b float64) float64 { return a + b },
		"-": func(a, b float64) float64 { return a - b },
		"*": func(a, b float64) float64 { return a * b },
		"/": func(a, b float64) float64 { return a / b },
	}

	for _, op := range []string{"+", "-", "*", "/"} {
		fmt.Printf("10 %s 4 = %.2f\n", op, calc[op](10, 4))
	}
}
```

Output:

```
10 + 4 = 14.00
10 - 4 = 6.00
10 * 4 = 40.00
10 / 4 = 2.50
```

This is a cleaner alternative to a long `switch`, and a stepping stone to **routing tables** in web servers (Chapter 40+), where a URL is mapped to a handler function.

---

## 11. Named function vs function expression

| | Named function | Function expression |
|--|----------------|---------------------|
| Syntax | `func add(a, b int) int {…}` | `add := func(a, b int) int {…}` |
| Where allowed | Package level only | Anywhere (inside functions too) |
| Order sensitivity | None | Local: define before use |
| Recursion | Trivial | Needs `var f func...` first |
| Can capture surrounding locals | ❌ (no surrounding function) | ✅ (closures) |
| Can be reassigned | ❌ | ✅ (variable) |
| Shows up in stack traces as | `main.add` | `main.main.func1` |
| Typical use | Reusable logic | Short-lived helpers, callbacks, closures |

**Rule of thumb:** use **named functions** by default. Reach for function expressions when you need to capture local variables, pass a function as a value, or keep a tiny helper next to where it's used.

---

## 12. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Calling a local function expression before defining it | `undefined: add` | Define first |
| 2 | Adding `()` when storing (`add := func(){...}()`) | `add` gets the *result*, not the function (compile error if no result) | Omit the trailing `()` |
| 3 | Forgetting `()` when calling (`add` vs `add(2,3)`) | You reference the function without running it; often "value not used" | Add arguments in parentheses |
| 4 | Recursion without predeclaring | `undefined: fact` | `var fact func(int) int` first |
| 5 | Mismatched signatures on reassignment | `cannot use ... as func(...) value` | Match parameter and result types |
| 6 | Calling a `nil` function variable | runtime panic | Initialize, or check for `nil` |
| 7 | Reusing a named function's name locally | Silent shadowing | Choose distinct names |
| 8 | Mutual references between package-level function variables | `initialization cycle` | Use named functions |

---

## 13. Exercises

### Exercise 1: Basic function expression
Store a function in `square` that returns the square of an `int`. Print `square(9)`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	square := func(n int) int { return n * n }
	fmt.Println(square(9)) // 81
}
```
</details>

### Exercise 2: Order bug
Fix the error:

```go
// INTENTIONAL ERROR
package main

import "fmt"

func main() {
	greet("Asha")
	greet := func(name string) {
		fmt.Println("Hello,", name)
	}
}
```

<details><summary>Solution</summary>

Define before calling:

```go
package main

import "fmt"

func main() {
	greet := func(name string) {
		fmt.Println("Hello,", name)
	}
	greet("Asha")
}
```
</details>

### Exercise 3: Package vs local
Which of these compile? What do they print?

```go
// A
package main

import "fmt"

func main() { hi() }

var hi = func() { fmt.Println("hi") }
```

```go
// B (INTENTIONAL ERROR)
package main

import "fmt"

func main() {
	hi()
	hi := func() { fmt.Println("hi") }
	_ = hi
}
```

<details><summary>Solution</summary>

A compiles and prints `hi`: package-level variables are initialized before `main`. B doesn't compile: `undefined: hi` (there's no package-level `hi`, and the local one doesn't exist yet).
</details>

### Exercise 4: Function table
Build a `map[string]func(string) string` with `"upper"`, `"lower"`, and `"reverse"`, then apply each to `"Go Language"`.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	transforms := map[string]func(string) string{
		"upper": strings.ToUpper,
		"lower": strings.ToLower,
		"reverse": func(s string) string {
			r := []rune(s)
			for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
				r[i], r[j] = r[j], r[i]
			}
			return string(r)
		},
	}

	for _, name := range []string{"upper", "lower", "reverse"} {
		fmt.Println(name, "→", transforms[name]("Go Language"))
	}
}
```

Note that `strings.ToUpper` (a *named* function from a package) fits right in: named functions are values too.
</details>

### Exercise 5: Recursive Fibonacci
Write a recursive function expression `fib` and print the first 10 Fibonacci numbers.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	var fib func(int) int
	fib = func(n int) int {
		if n < 2 {
			return n
		}
		return fib(n-1) + fib(n-2)
	}

	for i := 0; i < 10; i++ {
		fmt.Print(fib(i), " ")
	}
	fmt.Println() // 0 1 1 2 3 5 8 13 21 34
}
```
</details>

### Exercise 6: A counter (closure preview)
Create `counter := 0` and a function expression `increment` that adds 1 to it and prints it. Call `increment` three times.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	counter := 0
	increment := func() {
		counter++
		fmt.Println("Counter:", counter)
	}
	increment() // 1
	increment() // 2
	increment() // 3
}
```
The function *captures* `counter` from its surrounding scope, which is a closure (Chapter 20).
</details>

### Exercise 7 (challenge): Predict

```go
package main

import "fmt"

var f = func() string { return "package" }

func main() {
	fmt.Println(f())
	f := func() string { return "local" }
	fmt.Println(f())
	{
		f := func() string { return "block" }
		fmt.Println(f())
	}
	fmt.Println(f())
}
```

<details><summary>Solution</summary>

```
package
local
block
local
```
</details>

---

## 14. Quiz

1. What's the difference between `f := func(){...}` and `f := func(){...}()`?
2. Why can `main` call a package-level `var f = func(){...}` declared below it?
3. Why must a local function expression be defined before it is called?
4. How do you write a recursive function expression?
5. What happens if you call a `nil` function variable?

<details><summary>Answers</summary>

1. The first stores the function; the second invokes it immediately and stores its *result*.
2. Package-level variables are initialized before `main` runs.
3. Locals come into existence when their declaration executes, top to bottom.
4. Declare `var f func(...) ...` first, then assign `f = func(...) { ... f(...) ... }`.
5. A runtime panic (nil pointer dereference).
</details>

---

## 15. Summary

- A **function expression** stores an anonymous function in a **variable**: `add := func(a, b int) int {...}`. No trailing `()`.
- Functions are **first-class values** with a **type** (`func(int, int) int`), which you can name with `type`.
- **Local** function expressions must be defined **before** use; **package-level** ones (and named functions) work in any order because they're set up before `main`.
- A local function variable can **shadow** a named function. Avoid that.
- For **recursion**, predeclare the variable: `var f func(int) int`.
- An unset function variable is `nil`; calling it panics.
- Maps/slices of functions give you clean **dispatch tables**.

### ➡️ What's next?

[Chapter 17](17-parameters-vs-arguments-first-order-vs-higher-order-functions.md) clears up **parameters vs. arguments**, and introduces **first-order** and **higher-order functions**, functions that take or return other functions.
