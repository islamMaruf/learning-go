# Chapter 11: Scope Deep Dive — A "Boring" Example That Teaches Everything

> **Goal of this chapter:** Trace a complete program (three functions calling each other) through memory, phase by phase. You'll master the *call stack*, learn why function order doesn't matter, and see where Go sits among programming paradigms.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 1.5–2 hours  **Prerequisite:** [Chapters 8–10](08-what-is-scope.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why study a "boring" example?](#2-why-study-a-boring-example)
3. [The program](#3-the-program)
4. [The two-phase model: compile, then run](#4-the-two-phase-model)
5. [Full memory simulation](#5-full-memory-simulation)
6. [The call stack](#6-the-call-stack)
7. [Lifecycle of a function call](#7-lifecycle-of-a-function-call)
8. [The big reveal: function order doesn't matter](#8-function-order-doesnt-matter)
9. [Seeing the call stack for real](#9-seeing-the-call-stack-for-real)
10. [Programming paradigms and where Go fits](#10-programming-paradigms)
11. [Why functions are everywhere in Go](#11-why-functions-are-everywhere-in-go)
12. [Common mistakes](#12-common-mistakes)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- To trace a program by hand, the single most useful debugging skill
- How functions **call each other** and how the **call stack** grows and shrinks
- Why Go doesn't care in what order you write your functions
- What "programming paradigm" means, and why Go is a practical mix

---

## 2. Why study a "boring" example?

Athletes practice basic drills for years. Musicians play scales. Programmers should **trace simple programs until it's second nature**.

> When you can trace a *boring* program flawlessly, advanced topics (closures, goroutines, pointers) stop being scary, because they all build on the same mental machine.

This chapter is intentionally simple. **Don't skim it.** Grab a pencil and follow along.

---

## 3. The program

```go
package main

import "fmt"

var a = 10
var b = 20

func add(x int, y int) {
	result := x + y
	printNumber(result)
}

func printNumber(n int) {
	fmt.Println("Result:", n)
}

func main() {
	add(a, b)
}
```

**What it does:** `main` calls `add(a, b)`; `add` computes the sum and calls `printNumber`, which prints it.

Expected output:

```
Result: 30
```

Four scopes are involved: **package**, `main`, `add`, `printNumber`.

---

## 4. The two-phase model

Go handles a program in two clear phases.

### Phase A: compile time (the whole package is read first)

The compiler reads **all** the package-level declarations before running anything: variables, functions, types. It checks that every name used is declared somewhere visible, and generates machine code.

### Phase B: run time (execution starts at `main`)

The program starts, package-level variables are initialized (`a = 10`, `b = 20`), then Go calls `main()`.

```
Source code ──► [ compiler: read everything, check names, generate code ] ──► executable
                                                                                  │
                                                            run ──► init package vars ──► main()
```

This explains a lot: for example, why `main` can call `add` even though `add` is defined *earlier or later*: by run time the compiler has already seen every function.

---

## 5. Full memory simulation

We'll show the state of memory after each step. Read them like frames in a film.

### Step 1: package scope created and initialized

```
┌────────────────────────────────┐
│ PACKAGE SCOPE                  │
│  a           = 10              │
│  b           = 20              │
│  add         = <function code> │
│  printNumber = <function code> │
│  main        = <function code> │
└────────────────────────────────┘
```

### Step 2: Go calls `main()`, so main's scope is created

`main` has no local variables of its own here, so its scope is empty.

```
┌────────────────────────────────┐
│ PACKAGE SCOPE  a=10  b=20 …    │
└────────────────────────────────┘
   ┌──────────────────────┐
   │ main                 │
   │  (no locals)         │
   └──────────────────────┘
```

### Step 3: `main` calls `add(a, b)`

Arguments are evaluated first: `a` is not local to `main`, so Go looks outward and finds package `a = 10`; likewise `b = 20`. The **values are copied** into `add`'s new scope:

```
┌────────────────────────────────┐
│ PACKAGE SCOPE  a=10  b=20 …    │
└────────────────────────────────┘
   ┌──────────────────────┐
   │ main                 │
   └──────────────────────┘
   ┌──────────────────────┐
   │ add                  │
   │  x      = 10 (copy)  │
   │  y      = 20 (copy)  │
   │  result = 30         │  ← after executing result := x + y
   └──────────────────────┘
```

### Step 4: `add` calls `printNumber(result)`

`result`'s value 30 is copied into `printNumber`'s parameter `n`:

```
   ┌──────────────────────┐
   │ main                 │
   ├──────────────────────┤
   │ add                  │
   │  x=10 y=20 result=30 │
   ├──────────────────────┤
   │ printNumber          │
   │  n = 30 (copy)       │
   └──────────────────────┘
```

Inside, `fmt.Println("Result:", n)` prints:

```
Result: 30
```

`printNumber` can see `n` (its own) and package names (`a`, `b`, `fmt`), **but not** `x`, `y`, or `result`; those belong to `add`'s scope.

### Step 5: `printNumber` ends, so its scope is destroyed

```
   ┌──────────────────────┐
   │ main                 │
   ├──────────────────────┤
   │ add                  │
   │  x=10 y=20 result=30 │
   └──────────────────────┘
```

### Step 6: `add` ends, so its scope is destroyed

```
   ┌──────────────────────┐
   │ main                 │
   └──────────────────────┘
```

`add` has no more work to do, so its variables `x`, `y`, `result` vanish.

### Step 7: `main` ends, so the program ends

`main` returns; the runtime exits; package variables and everything else are released.

### Timeline in one picture

```
time ─►
main      ████████████████████████████████████
add          ██████████████████████████
printNumber       ███████████
                  (prints "Result: 30")
```

**Last called = first finished.** `printNumber` started last and ended first. This last-in-first-out behavior is exactly what a **stack** is.

---

## 6. The call stack

The **call stack** is the list of *currently active* function calls, with the most recent on top.

```
Moment 1:  ┌────────────┐
           │ main       │
           └────────────┘

Moment 2:  ┌────────────┐
           │ add        │  ← currently running
           ├────────────┤
           │ main       │
           └────────────┘

Moment 3:  ┌────────────┐
           │ printNumber│  ← currently running
           ├────────────┤
           │ add        │
           ├────────────┤
           │ main       │
           └────────────┘

Moment 4:  (printNumber returns)  back to the Moment 2 picture
Moment 5:  (add returns)          back to the Moment 1 picture
Moment 6:  (main returns)         empty → program exits
```

Each box is a **stack frame** (Chapter 4): it holds that call's parameters and local variables. When a function returns, control goes back to the caller **at the line right after the call**. The stack is how the computer remembers where to return to.

> **Analogy: a stack of plates or a browser's "back" history.** New plates go on top; you can only remove the top one.

---

## 7. Lifecycle of a function call

```
        call                     work                    return
  ┌──────────────┐        ┌──────────────┐        ┌──────────────────┐
  │ create frame │  ───►  │ run the body │  ───►  │ destroy frame    │
  │ copy args in │        │ (may call    │        │ hand back result │
  │              │        │  others)     │        │ resume the caller│
  └──────────────┘        └──────────────┘        └──────────────────┘
```

- **Born** when called.
- **Alive** while its body runs (including while it waits for functions *it* called).
- **Dead** when it returns. A new call creates a completely fresh frame; nothing carries over.

> "In this world, when someone has no work left, they disappear." Functions are like that: when the work is done, the frame is gone.

---

## 8. Function order doesn't matter

Here is the **same program** with the functions in a different order:

```go
package main

import "fmt"

var a = 10
var b = 20

func main() {
	add(a, b) // add is defined BELOW: and that's fine
}

func printNumber(n int) {
	fmt.Println("Result:", n)
}

func add(x int, y int) {
	result := x + y
	printNumber(result)
}
```

Output: `Result: 30`. It still works.

**Why?** Because of the two-phase model: the compiler reads *all* package-level declarations first (Phase A). By the time `main` runs (Phase B), it already knows about `add` and `printNumber` regardless of where they were written.

### Placement rules

| Thing | Order matters? |
|-------|---------------|
| Package-level functions | ❌ No, any order |
| Package-level variables | ❌ No (Go resolves dependencies between them automatically) |
| Local variables inside a function | ✅ **Yes**, must be declared before use |
| Imports | Must be at the top of each file, after `package` |

**Style convention:** put `main` at the top or bottom, and keep related functions near each other. Many teams put `main` first as a "table of contents", others put helpers first. Pick one, and be consistent.

> **Package variables dependency demo:** `var x = y + 1; var y = 5` is legal at package level; Go initializes `y` first. Inside a function the same order would be an error.

---

## 9. Seeing the call stack for real

You can ask Go to print the current call stack. This is a great way to *see* the stack we drew:

```go
package main

import (
	"fmt"
	"runtime/debug"
)

func level3() {
	fmt.Println("=== stack at level3 ===")
	fmt.Println(string(debug.Stack()))
}

func level2() { level3() }
func level1() { level2() }

func main() { level1() }
```

The printed trace lists `main.level3`, `main.level2`, `main.level1`, `main.main` from top (most recent) to bottom, exactly like our diagram. When a Go program **panics** (crashes), it prints this same kind of trace, and you'll learn to read it as a map to the bug.

---

## 10. Programming paradigms

A **paradigm** is a *style* of organizing programs. Three you'll hear about:

### 1. Structured / procedural programming

Programs are built from **sequences, choices (`if`/`switch`), loops (`for`), and procedures (functions)**. Data and functions are separate. C and Pascal are classic examples.

```go
func area(w, h float64) float64 { return w * h }
```

### 2. Object-oriented programming (OOP)

Data and behavior are bundled into **objects** (instances of classes); features include inheritance and polymorphism. Java, C#, and C++ are typical.

### 3. Functional programming (FP)

Programs are built by **composing functions**. Functions are *values* (you can store them in variables, pass them to and return them from other functions). Data tends to be immutable, and functions "pure". Haskell and Elixir are FP languages.

### Where does Go fit?

Go is a **pragmatic mix**:

| Paradigm | Go's support |
|----------|--------------|
| Procedural | ✅ Core style: packages, functions, structs |
| Object-oriented | ⚠️ Partial: **structs + methods + interfaces**, but **no classes and no inheritance** (uses *composition* instead) |
| Functional | ⚠️ Partial: functions are **first-class values**, closures work (Chapters 15–20), but no immutability enforcement, no built-in `map`/`filter` (you write loops) |
| Concurrent | ✅ First-class: goroutines and channels |

> Go's designers deliberately kept it small. You'll be productive in a "functions + structs + interfaces" style, and that is what nearly all Go code looks like.

---

## 11. Why functions are everywhere in Go

In real Go projects you'll write:

- **HTTP handlers**: functions called for each web request (Chapter 40+)
- **Middleware**: functions that *wrap* other functions (Chapter 45)
- **Goroutines**: `go someFunction()` launches a function concurrently (Chapter 36)
- **Callbacks / closures**: functions passed around as values (Chapters 15–20)
- **Deferred calls**: `defer cleanup()` runs a function on exit (Chapter 34)

That is why this course spends so many chapters on functions: **functions are Go's fundamental building block**, and their scoping and memory behavior underlies goroutines, closures, and middleware.

---

## 12. Common mistakes

| # | Mistake | Explanation |
|---|---------|-------------|
| 1 | Thinking `add` can read `main`'s locals | It only gets **copies through parameters** |
| 2 | Thinking a function "remembers" variables between calls | Each call is a fresh frame (use closures/structs for memory) |
| 3 | Believing order of functions matters | It doesn't at package level |
| 4 | Believing local declarations can come after their use | Inside functions, declare before use |
| 5 | Confusing "package variable" with "shared across programs" | Package variables live only inside one running program |
| 6 | Not tracing on paper | When puzzled, draw the stack. It always helps. |

---

## 13. Exercises

### Exercise 1: Trace it
Draw the stack at its tallest point and give the output.

```go
package main

import "fmt"

var factor = 3

func triple(n int) int { return n * factor }

func addTriples(a, b int) int {
	return triple(a) + triple(b)
}

func main() {
	fmt.Println(addTriples(1, 2))
}
```

<details><summary>Solution</summary>

Tallest point (inside the first or second `triple` call):
```
┌───────────────┐
│ triple: n=…   │  ← running
├───────────────┤
│ addTriples    │
├───────────────┤
│ main          │
└───────────────┘
```
`triple(1)=3`, `triple(2)=6`. Output: `9`. `triple` reads `factor` from package scope.
</details>

### Exercise 2: Rearrange
Move `main` to the top of your solution above and re-run it. Does it still work? Why?

<details><summary>Solution</summary>

Yes. Go reads all package-level declarations at compile time, so order is irrelevant.
</details>

### Exercise 3: Who can see what?
In the program below, which of the names `p`, `q`, `k`, `g` can `helper` use?

```go
package main

var g = 1

func helper() { /* here */ }

func main() {
	p := 1
	if p == 1 {
		q := 2
		_ = q
	}
	k := 3
	_ = k
	helper()
}
```

<details><summary>Solution</summary>

Only `g` (package scope), plus `helper`, `main`, and built-ins. `p`, `q`, `k` are locals of `main`.
</details>

### Exercise 4: Predict the output

```go
package main

import "fmt"

func c() { fmt.Println("c") }
func b() { fmt.Println("b start"); c(); fmt.Println("b end") }
func a() { fmt.Println("a start"); b(); fmt.Println("a end") }

func main() { a() }
```

<details><summary>Solution</summary>

```
a start
b start
c
b end
a end
```
</details>

### Exercise 5: Stack depth
Write a function chain that recurses to a depth of 5 (`countdown(5)` prints `5 4 3 2 1 liftoff`). How many `countdown` frames exist at the deepest point?

<details><summary>Solution</summary>

```go
package main

import "fmt"

func countdown(n int) {
	if n == 0 {
		fmt.Println("liftoff")
		return
	}
	fmt.Print(n, " ")
	countdown(n - 1)
}

func main() { countdown(5) }
```
Frames at the deepest point: `countdown(5)`, `(4)`, `(3)`, `(2)`, `(1)`, `(0)` = **6** frames on top of `main`. (A function calling itself is *recursion*; each call gets its own frame and its own `n`.)
</details>

### Exercise 6 (challenge): Chain of three
Build `double(n)`, `addOne(n)`, and `process(n)` where `process` returns `addOne(double(n))`. Print `process(5)` and draw the stack when `double` runs.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func double(n int) int { return n * 2 }
func addOne(n int) int { return n + 1 }
func process(n int) int { return addOne(double(n)) }

func main() { fmt.Println(process(5)) } // 11
```
Stack while `double` runs: `double` on top of `process` on top of `main`.
</details>

---

## 14. Quiz

1. What is a stack frame?
2. Why is the call stack called a "stack"?
3. In what order do functions finish relative to when they started?
4. Can `main` call a function defined later in the file?
5. Which paradigms does Go support well?

<details><summary>Answers</summary>

1. The memory block holding one function call's parameters, local variables, and return information.
2. The most recently called function finishes first (last-in, first-out).
3. Reverse order (nested calls finish before their callers).
4. Yes. Package-level functions are visible everywhere in the package.
5. Procedural (strongest), plus partial OOP (structs/methods/interfaces) and functional (first-class functions), plus first-class concurrency.
</details>

---

## 15. Summary

- Go **compiles the whole package first**, then runs from `main`. That's why **function order doesn't matter**.
- Every call creates a **stack frame**; frames pile up as functions call others and vanish as they return: **last in, first out**.
- Callers pass **copies** of values; callees can read package-level names but never a caller's locals.
- Tracing a program by hand, drawing the package scope and one box per call, is the best way to understand or debug it.
- Go is **procedural at heart**, with light OOP (structs, methods, interfaces) and functional features (first-class functions, closures), plus built-in concurrency.

### ➡️ What's next?

[Chapter 12](12-variable-shadowing.md) tackles **variable shadowing**, when an inner variable hides an outer variable with the same name. It's a favorite interview question and a common real-world bug.
