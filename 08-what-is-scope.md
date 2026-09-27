# Chapter 8: What Is Scope? — The Most Important Concept in This Section

> **Goal of this chapter:** Understand *where* in your program a name (variable, function, constant) can be used. Scope explains countless "undefined" errors, and it underpins closures, goroutines, and clean design later on.

**Difficulty:** 🟡 Beginner–Intermediate  **Estimated time:** 1.5–2 hours  **Prerequisite:** [Chapter 7](07-why-functions-are-needed.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The question scope answers](#2-the-question-scope-answers)
3. [A definition](#3-a-definition)
4. [Two big kinds: package (global) scope and local scope](#4-package-scope-and-local-scope)
5. [The lookup rule](#5-the-lookup-rule)
6. [Memory simulation, step by step](#6-memory-simulation)
7. [Local variables are private to their function](#7-local-variables-are-private)
8. [Lifetime: birth and death of variables](#8-lifetime-birth-and-death)
9. [Should you use global variables?](#9-should-you-use-global-variables)
10. [Common scope errors](#10-common-scope-errors)
11. [Interview questions](#11-interview-questions)
12. [Exercises](#12-exercises)
13. [Quiz](#13-quiz)
14. [Summary](#14-summary)

---

## 1. What you will learn

- What "scope" means and why it matters
- The difference between **package-level ("global")** and **local** scope
- How Go finds the variable you mean (the lookup rule)
- How the computer lays out memory for each function call
- How to read and fix the classic error `undefined: x`

> 💬 **Don't panic if it takes a few reads.** Scope is a *mental model*. Once it clicks, a huge amount of Go (and every other language) makes sense. Experienced developers still sketch scopes on paper when debugging.

---

## 2. The question scope answers

You declared a variable in one function. Why can't you use it in another?

```go
// INTENTIONAL ERROR
package main

import "fmt"

func setup() {
	message := "hello"
}

func main() {
	setup()
	fmt.Println(message) // ❌ undefined: message
}
```

The answer is **scope**.

---

## 3. A definition

> **Scope** is the region of a program in which a name is *visible*, that is, in which you can refer to it.

For any line of code you can ask: **"Can I use the name `x` here?"**

- ✅ Yes → `x` is *in scope* at that line.
- ❌ No → compile error: `undefined: x`.

Consider this program and try to answer the questions before reading on:

```go
package main

import "fmt"

var a = 20 // (1)
var b = 30 // (2)

func main() {
	var p = 30 // (3)
	var q = 40 // (4)
	add(p, q)
}

func add(x int, y int) {
	z := x + y // (5)
	fmt.Println(z)
}
```

1. Can `main` use `a` and `b`?
2. Can `add` use `a` and `b`?
3. Can `add` use `p` and `q`?
4. Can `main` use `z`?

**Answers:** Yes. Yes. **No.** **No.** By the end of the chapter you'll be able to explain *why*.

---

## 4. Package scope and local scope

### 4.1 Package scope (often called "global")

Anything declared **outside all functions** belongs to the **package**. Every function in the package (in *every file* of that package) can see it.

```go
package main

import "fmt"

var appName = "MyApp" // package scope
const version = "1.0" // package scope

func printInfo() {
	fmt.Println(appName, version) // ✅ visible
}

func main() {
	fmt.Println(appName, version) // ✅ visible
	printInfo()
}
```

Output:

```
MyApp 1.0
MyApp 1.0
```

| Property | Package scope |
|----------|---------------|
| Where declared | Outside all functions |
| Who can use it | Every function in the package |
| When created | When the program starts |
| When destroyed | When the program ends |

> 📝 **Terminology note.** Many tutorials say "global scope". Strictly, Go has no true universe-wide globals: a package-level name is visible in its own package, and visible to *other* packages only if it starts with a capital letter (Chapter 10). We'll use "package scope" and "global" interchangeably for now.

Also, at package level you **must** use `var`/`const`/`func`. The short `:=` form is not allowed there.

### 4.2 Local scope (function scope)

Anything declared **inside a function** is *local* to that function.

```go
package main

import "fmt"

func functionA() {
	x := 10 // local to functionA
	fmt.Println("A:", x)
}

func functionB() {
	x := 20 // a completely different x, local to functionB
	fmt.Println("B:", x)
}

func main() {
	functionA()
	functionB()
}
```

Output:

```
A: 10
B: 20
```

Each function has its **own** `x`. They don't collide.

| Property | Local scope |
|----------|-------------|
| Where declared | Inside a function |
| Who can use it | Only that function (and only *after* the declaration line) |
| When created | Each time the function is called |
| When destroyed | When the function returns |

**Parameters** (`x`, `y` in `add(x, y int)`) are also local to their function.

---

## 5. The lookup rule

When you write a name, Go searches for it **from the inside out**, stopping at the first match:

```
1. Current (innermost) scope, i.e. the function you're in
      ↓ not found
2. Enclosing scopes (blocks around you)
      ↓ not found
3. Package scope
      ↓ not found
4. Universe scope (built-ins like int, true, len, nil)
      ↓ not found
5. ❌ compile error: undefined: name
```

### Case A: local wins over package

```go
package main

import "fmt"

var x = 100 // package scope

func test() {
	x := 50 // local: this "shadows" the package-level x
	fmt.Println(x)
}

func main() {
	test()
	fmt.Println(x)
}
```

Output:

```
50
100
```

Inside `test`, the **local** `x` is found first. The package-level `x` is untouched, and `main` still sees `100`. (This is called **shadowing**; Chapter 12 is devoted to it.)

### Case B: fall back to package scope

```go
package main

import "fmt"

var x = 100

func test() {
	fmt.Println(x) // no local x → found in package scope
}

func main() { test() } // prints 100
```

### Case C: not found

```go
// INTENTIONAL ERROR
package main

import "fmt"

func test() {
	x := 50
	_ = x
}

func main() {
	fmt.Println(x) // ❌ undefined: x  (x lives only inside test)
}
```

---

## 6. Memory simulation

Now let's watch a computer execute this program. This is **exactly** how to answer scope questions in an interview.

```go
package main

import "fmt"

var a = 20
var b = 30

func add(x int, y int) {
	z := x + y
	fmt.Println(z)
}

func main() {
	p := 30
	q := 40
	add(p, q)
	add(a, b)
	add(a, p)
}
```

### Phase 1: the package scope is created before anything runs

```
┌──────────────────────────────┐
│ PACKAGE SCOPE                │
│  a    = 20                   │
│  b    = 30                   │
│  add  = <code>               │
│  main = <code>               │
└──────────────────────────────┘
```

Go reads the file, registers package-level variables and functions, then **starts running `main`**.

### Phase 2: `main` starts, so a local scope is created for it

```
┌──────────────────────────────┐
│ PACKAGE SCOPE  a=20  b=30 …  │
└──────────────────────────────┘
        ┌──────────────────────┐
        │ main's SCOPE         │
        │  p = 30              │
        │  q = 40              │
        └──────────────────────┘
```

### Phase 3: `add(p, q)`, so a **new** scope is created for this call

```
┌──────────────────────────────┐
│ PACKAGE SCOPE  a=20  b=30 …  │
└──────────────────────────────┘
        ┌──────────────────────┐
        │ main's SCOPE         │
        │  p = 30   q = 40     │
        └──────────────────────┘
        ┌──────────────────────┐
        │ add's SCOPE (call 1) │
        │  x = 30   (copy of p)│
        │  y = 40   (copy of q)│
        │  z = 70              │
        └──────────────────────┘
```

Prints `70`. Notice:
- `add` **cannot** see `p` and `q` (they belong to `main`'s scope). It sees only its own copies `x` and `y`.
- `add` **can** see `a` and `b` if it wanted to (package scope).

### Phase 4: `add` returns, so its scope is destroyed

```
┌──────────────────────────────┐
│ PACKAGE SCOPE  a=20  b=30 …  │
└──────────────────────────────┘
        ┌──────────────────────┐
        │ main's SCOPE         │
        │  p = 30   q = 40     │
        └──────────────────────┘
                                   ← x, y, z are gone
```

### Phase 5: `add(a, b)`: a brand-new scope again

```
        ┌──────────────────────┐
        │ add's SCOPE (call 2) │
        │  x = 20   y = 30     │
        │  z = 50              │
        └──────────────────────┘
```

Prints `50`. This is a **new** `x`, `y`, `z`; nothing from call 1 survived.

### Phase 6: `add(a, p)`

`x = 20`, `y = 30` (p is 30), `z = 50`. Prints `50`.

### Phase 7: `main` ends, so the program ends and everything is released

**Full output:**

```
70
50
50
```

### Answers to the opening questions

1. `main` can use `a` and `b`: package scope. ✅
2. `add` can use `a` and `b`: package scope. ✅
3. `add` cannot use `p` and `q`: they're local to `main`. ❌ (It gets *copies* through its parameters.)
4. `main` cannot use `z`: local to `add`, and it died when `add` returned. ❌

---

## 7. Local variables are private

Because each call gets its own scope, functions are **isolated**, which is a feature. It means:

- You can reuse short names (`i`, `n`, `err`) everywhere without collisions.
- A bug in one function can't secretly corrupt another function's variables.
- You can understand a function by reading *only that function*.

To share data between functions you make it explicit:

```go
package main

import "fmt"

func double(n int) int { // data IN via parameter, data OUT via return
	return n * 2
}

func main() {
	x := 21
	x = double(x)
	fmt.Println(x) // 42
}
```

---

## 8. Lifetime: birth and death

**Scope** (where a name is visible) is related to, but not the same as, **lifetime** (how long the value exists in memory).

| Kind | Born | Dies |
|------|------|------|
| Package variable | Program start | Program end |
| Local variable | Function call (declaration executes) | Function returns* |
| Parameter | Function call | Function returns* |

\* Mostly. In Chapter 20 (**closures**) you'll see that a local variable can outlive its function if something still refers to it. Go's compiler then quietly moves the variable off the stack (Chapter 18). Scope is a *compile-time* idea; lifetime is a *runtime* idea.

---

## 9. Should you use global variables?

You *can*, but you usually *shouldn't*.

```go
package main

import "fmt"

var counter = 0 // package-level state

func increment() { counter++ }

func main() {
	increment()
	increment()
	fmt.Println(counter) // 2
}
```

| ✅ Acceptable | ❌ Avoid |
|--------------|---------|
| Constants (`const Pi = 3.14`) | Mutable state that many functions change |
| Read-only configuration set once at start-up | Hidden dependencies ("what changed `counter`?") |
| Error values (`var ErrNotFound = errors.New(...)`) | Anything shared between goroutines without protection (Chapter 67: race conditions) |

Why avoid mutable globals?
1. **Hidden coupling:** any function can change them, so bugs are hard to trace.
2. **Hard to test:** tests leak state into one another.
3. **Unsafe with concurrency:** two goroutines writing at once corrupt data (Chapters 67–68).

**Prefer parameters and return values.** Data flow should be visible in function signatures.

---

## 10. Common scope errors

| # | Error / symptom | Cause | Fix |
|---|-----------------|-------|-----|
| 1 | `undefined: x` | Used a local variable outside its function | Return it, pass it, or declare it where both need it |
| 2 | Unexpected old value | A local variable *shadowed* the package one (`x := ...`) | Use `=` to assign the package-level variable |
| 3 | `x declared and not used` | Local declared, never read | Use it or remove it |
| 4 | `syntax error: non-declaration statement outside function body` | `x := 5` at package level | Use `var x = 5` |
| 5 | Function-local state "resets" every call | Locals are recreated on each call | Return it, store it in a struct, or use a package var or closure deliberately |

Example of #5:

```go
package main

import "fmt"

func next() int {
	count := 0 // brand-new variable every call!
	count++
	return count
}

func main() {
	fmt.Println(next(), next(), next()) // 1 1 1  (not 1 2 3)
}
```

---

## 11. Interview questions

**Q1. What is scope?**
The region of code where a name is accessible.

**Q2. What's the difference between package-level and local variables?**
Package-level variables are declared outside functions, visible package-wide, and live for the whole program. Locals are declared inside a function, visible only there, and normally die when the function returns.

**Q3. What does this print?**

```go
var x = 10

func f() { x := 20; fmt.Println(x) }

func main() { f(); fmt.Println(x) }
```

`20` then `10`. The `:=` in `f` creates a new local `x`.

**Q4. Can two functions each have a variable called `x`?**
Yes, they're independent.

**Q5. Why is `:=` not allowed at package level?**
Package-level statements must be declarations starting with a keyword (`var`, `const`, `func`, `type`), which lets the compiler process them in any order.

**Q6. Where is a function's local memory stored?**
On the stack in that call's stack frame (Chapters 4 and 18).

---

## 12. Exercises

### Exercise 1: Will it compile?
For each snippet, say whether it compiles, and what it prints if so.

**(a)**
```go
package main

import "fmt"

var msg = "hi"

func main() { fmt.Println(msg) }
```

**(b)**
```go
// INTENTIONAL ERROR
package main

import "fmt"

func one() { n := 1; fmt.Println(n) }

func main() { one(); fmt.Println(n) }
```

**(c)**
```go
package main

import "fmt"

var n = 5

func main() {
	n := 10
	fmt.Println(n)
}
```

<details><summary>Solution</summary>

(a) Compiles → `hi`.
(b) Does **not** compile: `undefined: n` in `main`.
(c) Compiles → `10` (a local `n` shadows the package `n`).
</details>

### Exercise 2: Trace the scopes
Draw the package scope and each call's scope, and give the output.

```go
package main

import "fmt"

var base = 100

func addBase(n int) int {
	total := n + base
	return total
}

func main() {
	a := addBase(5)
	b := addBase(10)
	fmt.Println(a, b)
}
```

<details><summary>Solution</summary>

Package scope: `base=100`, functions. `main` scope: `a`, `b`. Call 1 scope: `n=5, total=105` → returns 105, then it's destroyed. Call 2 scope: `n=10, total=110` → destroyed.
Output: `105 110`.
</details>

### Exercise 3: Fix the counter
`next()` above always returns 1. Give **two** fixes. Which one do you prefer?

<details><summary>Solution</summary>

**Fix 1: package variable:**
```go
package main

import "fmt"

var count = 0

func next() int {
	count++
	return count
}

func main() { fmt.Println(next(), next(), next()) } // 1 2 3
```

**Fix 2: pass and return state explicitly (preferred):**
```go
package main

import "fmt"

func next(count int) int { return count + 1 }

func main() {
	c := 0
	c = next(c)
	c = next(c)
	c = next(c)
	fmt.Println(c) // 3
}
```
Fix 2 keeps the data flow visible. (Chapter 20 shows a third way: closures.)
</details>

### Exercise 3 (challenge): Predict

```go
package main

import "fmt"

var x = 1

func a() int { x = x + 10; return x }
func b() int { x := 5; x = x + 10; return x }

func main() {
	fmt.Println(a(), b(), x)
}
```

<details><summary>Solution</summary>

`a()` modifies the package `x`: 1 → 11, returns 11. `b()` declares a local `x` (5 → 15), returns 15, and the package `x` is unaffected. Then `x` is 11.
Output: `11 15 11`. (Arguments are evaluated left to right.)
</details>

---

## 13. Quiz

1. Where can a package-level variable be used?
2. What happens to a function's local variables when it returns (normally)?
3. In what order does Go search for a name?
4. Why can't `main` use a variable declared inside `add`?
5. Can `:=` be used at package level?

<details><summary>Answers</summary>

1. In every function of the package.
2. They're destroyed with the function's stack frame.
3. Innermost scope → enclosing scopes → package → universe → error.
4. It's local to `add`, and doesn't even exist once `add` returns.
5. No; use `var`.
</details>

---

## 14. Summary

- **Scope** = where a name is visible. Outside its scope, the name is `undefined`.
- **Package scope:** declared outside functions; visible to the whole package.
- **Local scope:** declared inside a function; visible only there.
- **Lookup rule:** innermost → outward → package → universe → error. Inner names **shadow** outer ones.
- Each function call gets its **own** scope, created on call and destroyed on return.
- Prefer **parameters and return values** over mutable globals.

### ➡️ What's next?

Functions aren't the only things that create scopes. Every `{ ... }` **block** does. [Chapter 9](09-local-scope-and-block-scope.md) explores block scope inside `if`, `for`, and bare braces.
