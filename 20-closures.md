# Chapter 20: Closures — The Magic of Function Memory

> **Goal of this chapter:** Understand one of the most powerful ideas in programming: a **closure** is a function that *remembers* the variables around it, even after the function that created it has finished. You'll see exactly how that works in memory (escape analysis + the heap), and build counters, accounts, generators, and middleware with it.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 2–3 hours  **Prerequisite:** [Chapters 16–19](16-function-expressions.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The puzzle a closure solves](#2-the-puzzle-a-closure-solves)
3. [What is a closure?](#3-what-is-a-closure)
4. [The simplest closure: a counter](#4-the-simplest-closure-a-counter)
5. [How it works in memory](#5-how-it-works-in-memory)
6. [Independent closures, shared variables](#6-independent-closures-shared-variables)
7. [Escape analysis: how Go keeps the variable alive](#7-escape-analysis-recap)
8. [The garbage collector's role](#8-the-garbage-collectors-role)
9. [Closures capture *variables*, not values](#9-closures-capture-variables-not-values)
10. [The loop-variable trap (and Go 1.22's fix)](#10-the-loop-variable-trap)
11. [Real-world patterns](#11-real-world-patterns)
12. [Closures vs. structs](#12-closures-vs-structs)
13. [Closures and concurrency (a warning)](#13-closures-and-concurrency)
14. [Common mistakes](#14-common-mistakes)
15. [Interview questions](#15-interview-questions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- What a closure is, in one sentence
- How a function can use a variable from a function that has *already returned*
- The role of **escape analysis** and the **heap**
- That closures capture **variables** (by reference), not copies of values
- The famous **loop variable** trap, and how Go 1.22 changed it
- Practical closure patterns: counters, generators, memoization, middleware
- When to prefer a struct

---

## 2. The puzzle a closure solves

Recall the rule from Chapters 4 and 8: **when a function returns, its local variables die**.

Now suppose we want a function that *counts how many times it's been called*. Where would it store the count?

**Attempt 1: a local variable:**

```go
package main

import "fmt"

func counter() int {
	count := 0 // born on every call...
	count++
	return count // ...and dead on every return
}

func main() {
	fmt.Println(counter(), counter(), counter()) // 1 1 1  ✗
}
```

Every call starts from scratch. **Attempt 2: a package variable:**

```go
var count = 0

func counter() int {
	count++
	return count
}
```

That works (`1 2 3`), but now `count` is global: anybody can change it, only one counter can exist, and it isn't safe for concurrent use.

What we want: **private, persistent state attached to a function.** That is exactly what a closure gives us.

---

## 3. What is a closure?

> A **closure** is a function value that **captures variables from its surrounding scope** and keeps them alive, so it can keep using them even after the outer function has returned.

Two ingredients:

1. **A function** (usually an anonymous function, i.e. a function expression).
2. **Variables from the enclosing function** that it *uses* (its "free variables" / "captured variables").

The function "closes over" those variables, hence *closure*.

> **Analogy: a backpack.** When a function is created, it packs a backpack with the outer variables it needs. Wherever the function travels (even far from where it was born), it carries that backpack. Two functions born at different times carry *different* backpacks.

---

## 4. The simplest closure: a counter

```go
package main

import "fmt"

func outer() func() int {
	count := 0 // a local variable of outer

	return func() int { // an anonymous function that USES count
		count++
		return count
	}
}

func main() {
	next := outer() // outer runs and returns; its frame is popped

	fmt.Println(next()) // 1
	fmt.Println(next()) // 2
	fmt.Println(next()) // 3
}
```

Output:

```
1
2
3
```

Look at what happened:

1. `outer()` ran, created `count`, and **returned** an inner function.
2. `outer`'s frame was popped. By all our earlier rules, `count` should be dead.
3. Yet `next()` increments and remembers `count` across calls.

That's the magic. **`count` survived `outer`'s return because the inner function still needs it.**

Every piece of the notation:

```go
func outer() func() int {   // outer returns "a function taking nothing, returning int"
	count := 0                 // captured variable
	return func() int {        // the closure
		count++                // uses captured variable → this makes it a closure
		return count
	}
}
```

If the inner function did **not** reference `count` (or any other outer local), it would be an ordinary function value with nothing captured, and not a closure in practice.

---

## 5. How it works in memory

Program: (same as above, with a `call()` wrapper to make the picture bigger)

```go
package main

import "fmt"

func outer() func() int {
	count := 0
	return func() int {
		count++
		return count
	}
}

func main() {
	inc1 := outer()
	fmt.Println(inc1()) // 1
	fmt.Println(inc1()) // 2
}
```

**Phase 1: compile time**

The code of `outer` and its inner function (`outer.func1`) go to the **code segment**. The compiler also runs **escape analysis** and discovers: *`count` is used by an inner function that is returned, so `count` outlives `outer`'s frame → allocate `count` on the **heap**.* You can see this yourself:

```bash
go build -gcflags='-m' .
# ./main.go:6:2: moved to heap: count
```

**Phase 2: run time**

**Step 1: `main` calls `outer()`**

```
STACK                          HEAP
┌────────────────────┐         ┌───────────────────┐
│ outer              │         │ count = 0         │ ◄─┐  (allocated on the heap)
│   count ───────────┼────────►└───────────────────┘   │
├────────────────────┤                                 │
│ main               │                                 │
└────────────────────┘                                 │
```

**Step 2: `outer` executes `return func() {...}`.** It builds a **closure object**: the function's code reference **plus a pointer to the captured variable** `count`:

```
HEAP
┌──────────────────────────────┐
│ closure                      │
│   code ─► outer.func1        │  (code segment)
│   count ─────────────┐       │
└──────────────────────┼───────┘
                       ▼
                ┌──────────────┐
                │ count = 0    │
                └──────────────┘
```

**Step 3: `outer` returns.** Its **frame is popped**, but `count` and the closure are on the **heap**, so they survive. `main` stores the closure reference in `inc1`:

```
STACK                       HEAP
┌───────────────┐           ┌──────────────────────┐    ┌───────────┐
│ main          │           │ closure              │    │ count = 0 │
│   inc1 ───────┼──────────►│   code, ptr to count ├───►│           │
└───────────────┘           └──────────────────────┘    └───────────┘
```

**Step 4: `inc1()`.** Follow the reference, run the code in a new frame. The code reaches `count` **through the closure's pointer**, increments it (`0 → 1`), and returns `1`. The frame is popped, but the heap variable stays:

```
HEAP:  count = 1
```

**Step 5: `inc1()` again.** Same closure, same `count`: `1 → 2`. It returns `2`.

The only way it can "remember" is that `count` lives on the heap, reachable from the closure.

---

## 6. Independent closures, shared variables

### Each call to `outer` creates a **new** closure with its **own** variable

```go
package main

import "fmt"

func outer() func() int {
	count := 0
	return func() int {
		count++
		return count
	}
}

func main() {
	a := outer()
	b := outer()

	fmt.Println(a()) // 1
	fmt.Println(a()) // 2
	fmt.Println(b()) // 1  ← independent!
	fmt.Println(a()) // 3
	fmt.Println(b()) // 2
}
```

Output:

```
1
2
1
3
2
```

```
HEAP
┌─ closure A ──┐   ┌─────────┐
│ ptr ─────────┼──►│count = 3│
└──────────────┘   └─────────┘
┌─ closure B ──┐   ┌─────────┐
│ ptr ─────────┼──►│count = 2│
└──────────────┘   └─────────┘
```

Each `outer()` call runs `count := 0` afresh, producing a brand-new heap variable.

### Two closures made in the **same** call share the same variable

```go
package main

import "fmt"

func newAccount(initial int) (deposit func(int), balance func() int) {
	bal := initial // ONE variable...

	deposit = func(amount int) { bal += amount } // ...used by this closure
	balance = func() int { return bal }         // ...and by this one

	return deposit, balance
}

func main() {
	deposit, balance := newAccount(100)

	fmt.Println(balance()) // 100
	deposit(50)
	fmt.Println(balance()) // 150
	deposit(-30)
	fmt.Println(balance()) // 120
}
```

Output:

```
100
150
120
```

```
HEAP
┌─ deposit closure ─┐
│ ptr ──────────────┼──┐
└───────────────────┘  │    ┌──────────────┐
                       ├───►│ bal = 120    │  ← shared
┌─ balance closure ─┐  │    └──────────────┘
│ ptr ──────────────┼──┘
└───────────────────┘
```

Notice how nobody outside can touch `bal` directly: it's only reachable through the two functions. That's **encapsulation** (Chapter 19), now with private, persistent state. There's no way to set `bal = -1000` from `main`.

---

## 7. Escape analysis recap

How does Go know to put `count` on the heap? At compile time it asks:

> *Can this variable still be referenced after the function that declared it returns?*

- Used only inside its own function, never leaks → **stack** (fast, auto-freed).
- Captured by a closure that's **returned**, stored in a longer-lived place, or launched as a goroutine → **escapes** → **heap**.

```go
func noEscape() int {
	x := 1
	f := func() { x++ }   // closure never leaves noEscape
	f()
	return x              // x can stay on the stack
}

func escapes() func() {
	x := 1
	return func() { x++ } // closure leaves escapes(): x must live on the heap
}
```

You don't write anything special. The compiler does it automatically. Check with `go build -gcflags=-m`.

---

## 8. The garbage collector's role

The stack is cleaned by popping frames. But **who frees `count` on the heap** once we're done with the closure? The **garbage collector** (Chapter 18).

```
main ends / inc1 goes out of scope
        │
        ▼
closure object unreachable ──► GC marks it garbage
count variable unreachable ──► GC marks it garbage
        │
        ▼
memory reclaimed automatically
```

As long as *any* variable can still reach the closure, the closure and its captured variables stay alive. When nothing can, the GC eventually frees them. You never call `free`.

**Caution, memory:** a long-lived closure keeps everything it captured alive. If it captures a huge slice you no longer need, that memory can't be freed. Capture only what you need.

---

## 9. Closures capture variables, not values

This is the single most important subtlety.

```go
package main

import "fmt"

func main() {
	x := 10

	show := func() { fmt.Println("x is", x) } // captures the VARIABLE x

	show() // x is 10
	x = 99
	show() // x is 99   ← sees the update!
}
```

The closure doesn't take a snapshot of `x`; it shares the same variable `x` as `main`. Changes on either side are visible on the other.

Compare with **passing a parameter**, which *does* copy:

```go
package main

import "fmt"

func main() {
	x := 10

	snapshot := func(v int) func() {
		return func() { fmt.Println("v is", v) } // captures the parameter v (a copy of x)
	}(x) // IIFE: x's current value is copied into v

	x = 99
	snapshot() // v is 10  ← unaffected by later changes to x
}
```

If you *want* a snapshot, pass the value as a parameter (or make a fresh variable: `v := x`).

---

## 10. The loop-variable trap

### The classic bug (Go 1.21 and earlier)

```go
// Go 1.21 and earlier behavior
for i := 0; i < 3; i++ {
	defer func() { fmt.Println(i) }()
}
```

In older Go, there was **one** `i` for the whole loop, shared by all iterations. Every closure captured that same variable. By the time the deferred functions ran, the loop had ended with `i == 3`, so the output was:

```
3
3
3
```

Programmers worked around it with a per-iteration copy (`i := i`) or by passing `i` as an argument.

### Go 1.22 fixed it

Since **Go 1.22** (for modules whose `go.mod` says `go 1.22` or newer), **each iteration gets its own copy of the loop variable**:

```go
package main

import "fmt"

func main() {
	var funcs []func()

	for i := 0; i < 3; i++ {
		funcs = append(funcs, func() { fmt.Println(i) })
	}

	for _, f := range funcs {
		f()
	}
}
```

Output (Go 1.22+):

```
0
1
2
```

Each closure captured its own iteration's `i`. In Go ≤ 1.21 it would print `3 3 3`.

> ✅ This tutorial assumes Go 1.22+. But you'll meet the old idiom in older code and in interviews:
> ```go
> for i := 0; i < 3; i++ {
>     i := i // shadow: a fresh variable per iteration (Chapter 12!)
>     funcs = append(funcs, func() { fmt.Println(i) })
> }
> ```
> It's now redundant, but harmless.

The version in `go.mod` controls the behavior. A file with `go 1.21` in `go.mod` still uses the old semantics. That's how Go stays backward compatible.

---

## 11. Real-world patterns

### 11.1 Generators / sequences

```go
package main

import "fmt"

func fibonacci() func() int {
	a, b := 0, 1
	return func() int {
		result := a
		a, b = b, a+b
		return result
	}
}

func main() {
	next := fibonacci()
	for i := 0; i < 10; i++ {
		fmt.Print(next(), " ")
	}
	fmt.Println() // 0 1 1 2 3 5 8 13 21 34
}
```

### 11.2 Configuration / function factories

```go
package main

import (
	"fmt"
	"strings"
)

func makeLogger(prefix string) func(string) {
	return func(msg string) {
		fmt.Println(strings.ToUpper(prefix) + ": " + msg)
	}
}

func main() {
	info := makeLogger("info")
	warn := makeLogger("warn")
	info("server started") // INFO: server started
	warn("disk 90% full")  // WARN: disk 90% full
}
```

`prefix` is captured once per logger.

### 11.3 Memoization (cache results)

```go
package main

import "fmt"

func memoize(f func(int) int) func(int) int {
	cache := map[int]int{} // private cache, captured
	return func(n int) int {
		if v, ok := cache[n]; ok {
			fmt.Println("  (cache hit for", n, ")")
			return v
		}
		v := f(n)
		cache[n] = v
		return v
	}
}

func main() {
	slowSquare := func(n int) int { return n * n }
	fastSquare := memoize(slowSquare)

	fmt.Println(fastSquare(9))
	fmt.Println(fastSquare(9))
}
```

Output:

```
81
  (cache hit for 9 )
81
```

(Not goroutine-safe as written; see section 13.)

### 11.4 Lazy initialization

```go
package main

import "fmt"

func lazy(compute func() int) func() int {
	var value int
	var done bool
	return func() int {
		if !done {
			value = compute()
			done = true
		}
		return value
	}
}

func main() {
	get := lazy(func() int {
		fmt.Println("computing (expensive)...")
		return 42
	})
	fmt.Println(get()) // computing... then 42
	fmt.Println(get()) // 42 (no recomputation)
}
```

### 11.5 Middleware (preview of Chapters 45–49)

A web "middleware" wraps a handler and adds behavior. It's a closure over `next`:

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
)

func logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("request:", r.Method, r.URL.Path) // runs BEFORE the handler
		next.ServeHTTP(w, r)                          // captured `next`
	})
}

func main() {
	hello := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "hello")
	})

	// exercise it without starting a real server:
	req := httptest.NewRequest("GET", "/hi", nil)
	rec := httptest.NewRecorder()
	logger(hello).ServeHTTP(rec, req)
	fmt.Println("response:", rec.Body.String())
}
```

Output:

```
request: GET /hi
response: hello
```

### 11.6 Deferred cleanup that remembers state

```go
package main

import "fmt"

func process() {
	resource := "database connection"
	defer func() {
		fmt.Println("closing", resource) // closure over `resource`
	}()
	fmt.Println("using", resource)
}

func main() { process() }
```

---

## 12. Closures vs. structs

Anything a closure does with private state, a struct with methods can do too (Chapters 21–22):

```go
package main

import "fmt"

type Counter struct{ count int }

func (c *Counter) Next() int {
	c.count++
	return c.count
}

func main() {
	c := &Counter{}
	fmt.Println(c.Next(), c.Next()) // 1 2
}
```

| | Closure | Struct + methods |
|--|---------|------------------|
| State visibility | Completely hidden | Fields visible (unless lowercase in another package) |
| Multiple operations on same state | Return several closures (clumsy) | Add more methods (natural) |
| Inspecting/serializing state | Hard | Easy |
| Great for | Small, single-purpose behavior (callbacks, middleware, generators) | Objects with several operations and visible structure |

**Rule of thumb:** one small function with hidden state → closure. Several behaviors sharing data → struct.

---

## 13. Closures and concurrency

Closures are the standard way to hand data to goroutines, but a captured variable is **shared**. If several goroutines modify it without protection you get a **data race** (Chapter 67).

```go
// ⚠️ DATA RACE: many goroutines mutate `count` unsafely
count := 0
for i := 0; i < 1000; i++ {
	go func() { count++ }()
}
```

Fix with a mutex (Chapter 68), an atomic counter, or by giving each goroutine its own copy/using channels (Chapter 69). Run `go run -race .` to have Go detect such races for you.

---

## 14. Common mistakes

| # | Mistake | Explanation / fix |
|---|---------|-------------------|
| 1 | Expecting a fresh variable on each **call of the closure** | Captured variables persist across calls of the *same* closure; only a new call to the *outer* function makes new ones |
| 2 | Thinking closures copy values | They share **variables**. Use a parameter for a snapshot |
| 3 | Loop-variable capture in Go ≤ 1.21 | Copy `i := i`; with Go 1.22+ it's automatic |
| 4 | Capturing more than needed (large data) | Keeps it alive; capture only what's needed |
| 5 | Sharing captured variables across goroutines without locks | Data race |
| 6 | Calling the outer function again when you meant to reuse the closure | `outer()()` makes a *new* counter every time: `outer()()` always returns 1 |
| 7 | Thinking the inner function alone (no captured variables) is a "closure with memory" | Without captured variables it's just a function value |
| 8 | Modifying a captured variable after handing the closure out, and being surprised | The closure sees the change |

Mistake #6:

```go
package main

import "fmt"

func outer() func() int {
	count := 0
	return func() int { count++; return count }
}

func main() {
	fmt.Println(outer()(), outer()(), outer()()) // 1 1 1  (three different counters!)
}
```

---

## 15. Interview questions

**Q1. What is a closure?**
A function value together with the variables it captured from its enclosing scope, which remain alive as long as the function does.

**Q2. How does Go keep a captured variable alive after the outer function returns?**
Escape analysis detects that the variable outlives the function and allocates it on the heap; the closure holds a pointer to it. The GC frees it once unreachable.

**Q3. Do closures capture by value or by reference?**
By reference (they share the variable).

**Q4. What does this print in Go 1.21? In Go 1.22?**
```go
for i := 0; i < 3; i++ { defer func() { fmt.Print(i, " ") }() }
```
1.21: `3 3 3`. 1.22: `2 1 0` (defers run in reverse, with per-iteration `i`).

**Q5. Give three real uses of closures.**
Middleware, generators/iterators, memoization, callbacks with context, lazy initialization, function factories.

**Q6. Can two closures share state?**
Yes, if created in the same call of the outer function and both capture the same variable.

**Q7. Does every anonymous function form a closure?**
Only if it references variables from an enclosing function. Otherwise it captures nothing.

---

## 16. Exercises

### Exercise 1: Counter with step
Write `makeCounter(start, step int) func() int` so `c := makeCounter(10, 5)` gives `10, 15, 20` on successive calls.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func makeCounter(start, step int) func() int {
	current := start - step
	return func() int {
		current += step
		return current
	}
}

func main() {
	c := makeCounter(10, 5)
	fmt.Println(c(), c(), c()) // 10 15 20
}
```
</details>

### Exercise 2: Independent counters
Create two counters from `outer()`. Call `a()` twice and `b()` once. What do `a()` and `b()` return next?

<details><summary>Solution</summary>

`a()` → `3`, `b()` → `2`: each counter has its own captured variable.
</details>

### Exercise 3: Accumulator
Write `adder()` returning a function that takes an `int`, adds it to a running total, and returns the total.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func adder() func(int) int {
	sum := 0
	return func(n int) int {
		sum += n
		return sum
	}
}

func main() {
	add := adder()
	fmt.Println(add(1), add(2), add(10)) // 1 3 13
}
```
</details>

### Exercise 4: Bank account
Extend `newAccount` (section 6) with a `withdraw(amount int) error` that refuses to overdraw.

<details><summary>Solution</summary>

```go
package main

import (
	"errors"
	"fmt"
)

func newAccount(initial int) (deposit func(int), withdraw func(int) error, balance func() int) {
	bal := initial
	deposit = func(n int) { bal += n }
	withdraw = func(n int) error {
		if n > bal {
			return errors.New("insufficient funds")
		}
		bal -= n
		return nil
	}
	balance = func() int { return bal }
	return
}

func main() {
	deposit, withdraw, balance := newAccount(100)
	deposit(50)
	fmt.Println(balance())          // 150
	fmt.Println(withdraw(500))      // insufficient funds
	fmt.Println(withdraw(120), balance()) // <nil> 30
}
```
</details>

### Exercise 5: Predict (shared or not?)

```go
package main

import "fmt"

func main() {
	x := 1
	inc := func() { x++ }
	get := func() int { return x }

	inc()
	inc()
	x = 10
	inc()
	fmt.Println(get(), x)
}
```

<details><summary>Solution</summary>

`11 11`. Both closures and `main` share the single variable `x`: 1 → 2 → 3, then set to 10, then 11.
</details>

### Exercise 6: Loop closures
What does this print with Go 1.22+? And how would it look under Go 1.21?

```go
package main

import "fmt"

func main() {
	var fs []func() int
	for i := 1; i <= 3; i++ {
		fs = append(fs, func() int { return i * i })
	}
	for _, f := range fs {
		fmt.Print(f(), " ")
	}
	fmt.Println()
}
```

<details><summary>Solution</summary>

Go 1.22+: `1 4 9`. Go 1.21 (shared `i`, which ends at 4): `16 16 16`.
</details>

### Exercise 7: Once
Write `once(f func()) func()` returning a function that calls `f` only the first time, no matter how often it's called.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func once(f func()) func() {
	called := false
	return func() {
		if !called {
			called = true
			f()
		}
	}
}

func main() {
	setup := once(func() { fmt.Println("initialized") })
	setup()
	setup()
	setup() // prints "initialized" exactly once
}
```
(The standard library's `sync.Once` does this safely for concurrent code.)
</details>

### Exercise 8 (challenge): Rate limiter
Write `limiter(max int) func() bool` that returns `true` for the first `max` calls and `false` afterward.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func limiter(max int) func() bool {
	used := 0
	return func() bool {
		if used >= max {
			return false
		}
		used++
		return true
	}
}

func main() {
	allow := limiter(2)
	fmt.Println(allow(), allow(), allow(), allow()) // true true false false
}
```
</details>

---

## 17. Quiz

1. What two things make up a closure?
2. Where is a captured variable stored if the closure outlives its outer function?
3. What decides that? At which phase?
4. Do closures copy or share captured variables?
5. What did Go 1.22 change about `for` loops?
6. Who frees a closure's captured variables when they're no longer needed?

<details><summary>Answers</summary>

1. A function (code) and the captured variables from its enclosing scope.
2. On the heap.
3. Escape analysis, at compile time.
4. Share (by reference).
5. Each iteration gets its own copy of the loop variable.
6. The garbage collector.
</details>

---

## 18. Summary

- A **closure** is a function that **captures variables** from its surrounding scope and keeps them alive.
- The compiler's **escape analysis** moves captured variables to the **heap**; the closure holds a pointer to them; the **GC** frees them later.
- Each call to the outer function creates a **new, independent** set of captured variables; closures created together **share** them.
- Closures capture **variables (references)**, not values. For a snapshot, pass a parameter or copy.
- **Go 1.22+** gives each loop iteration its own loop variable, so loop closures behave intuitively.
- Patterns: counters, generators, memoization, lazy init, factories, **middleware**.
- Watch out for **data races** when closures run in goroutines.
- Use a **struct with methods** when there are several operations on the same state.

### ➡️ What's next?

Part 4 begins: [Chapter 21](21-structs.md) introduces **structs**: custom types that group related data, and the start of Go's take on "objects".
