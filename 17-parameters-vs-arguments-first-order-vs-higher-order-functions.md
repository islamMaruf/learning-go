# Chapter 17: Parameters vs. Arguments, First-Order vs. Higher-Order Functions

> **Goal of this chapter:** Nail down two everyday-vocabulary pairs (**parameter/argument** and **first-order/higher-order**), learn **callbacks**, and understand what it means that functions are **first-class citizens**. These ideas power middleware, sorting, goroutines, and most of the "professional Go" material later in this course.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 2 hours  **Prerequisite:** [Chapter 16](16-function-expressions.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Parameter vs. argument](#2-parameter-vs-argument)
3. [First-order functions](#3-first-order-functions)
4. [Higher-order functions](#4-higher-order-functions)
5. [Callbacks](#5-callbacks)
6. [First-class functions and first-class citizens](#6-first-class-functions)
7. [Memory simulation of a callback](#7-memory-simulation)
8. [Where the names come from (a little math)](#8-where-the-names-come-from)
9. [Practical patterns](#9-practical-patterns)
10. [A peek at generics: `Map`, `Filter`, `Reduce`](#10-a-peek-at-generics)
11. [Interview questions](#11-interview-questions)
12. [Common mistakes](#12-common-mistakes)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- The precise difference between a **parameter** and an **argument** (a classic interview question)
- What **first-order** and **higher-order** functions are
- What a **callback** is and why they're everywhere
- What **first-class functions** means
- Practical patterns: strategies, decorators, and function factories

---

## 2. Parameter vs. argument

They sound like synonyms; they aren't.

| Term | Where it appears | What it is |
|------|------------------|------------|
| **Parameter** | In the function **definition** | A named placeholder: "this function expects an `int` called `x`" |
| **Argument** | In the function **call** | The actual value you supply: `5` |

```go
func add(x int, y int) int { // x and y are PARAMETERS
	return x + y
}

result := add(5, 3) // 5 and 3 are ARGUMENTS
```

```
 definition:   func add( x int,  y int )
                          ▲        ▲
              at call:  add( 5,     3 )
                        arguments fill the parameters
```

**Memory trick:**
- **P**arameter → **P**laceholder (in the **P**attern/definition).
- **A**rgument → **A**ctual value (in the **A**ction/call).

> **Analogy:** A restaurant menu says "Pizza with *[topping]*". `[topping]` is the parameter. When you order "Pizza with mushrooms", *mushrooms* is the argument.

When you call `add(5, 3)`, the arguments are **copied** into the parameters (`x = 5`, `y = 3`) in a new stack frame (Chapter 4). People also say "formal parameters" (definition) and "actual parameters" (call) for the same distinction, and sometimes loosely say "parameter" for both. In an interview, use the precise terms.

Another example:

```go
package main

import "fmt"

func greet(name string, times int) { // parameters: name, times
	for i := 0; i < times; i++ {
		fmt.Println("Hello,", name)
	}
}

func main() {
	greet("Asha", 2) // arguments: "Asha", 2
}
```

---

## 3. First-order functions

> A **first-order function** works only with ordinary data (numbers, strings, structs, …). It does **not** take functions as parameters and does **not** return functions.

Everything you wrote before Chapter 15 was first-order:

```go
package main

import "fmt"

func add(x, y int) int { return x + y } // takes ints, returns an int

func main() {
	fmt.Println(add(2, 3))
}
```

---

## 4. Higher-order functions

> A **higher-order function (HOF)** does at least one of these:
> 1. **takes a function as a parameter**, or
> 2. **returns a function**, (or both).

It "works on" functions themselves, one level *higher* than data.

### 4.1 Taking a function as a parameter

```go
package main

import "fmt"

// Higher-order: `operation` is a parameter whose TYPE is a function
func processOperation(a, b int, operation func(int, int)) {
	operation(a, b) // call the function we received
}

// First-order
func add(x, y int) {
	fmt.Println(x + y)
}

func main() {
	processOperation(2, 5, add) // pass the function `add` (no parentheses!)
}
```

Output: `7`

Step by step:

1. `processOperation(2, 5, add)`: arguments `2`, `5`, and *the function `add`*.
2. Inside: `a = 2`, `b = 5`, `operation = add`.
3. `operation(a, b)` is really `add(2, 5)`, so it prints `7`.

We can swap behavior without changing `processOperation`:

```go
package main

import "fmt"

func processOperation(a, b int, operation func(int, int)) {
	operation(a, b)
}

func main() {
	processOperation(2, 5, func(x, y int) { fmt.Println("sum:", x+y) })
	processOperation(2, 5, func(x, y int) { fmt.Println("product:", x*y) })
}
```

Output:

```
sum: 7
product: 10
```

### 4.2 Returning a function

```go
package main

import "fmt"

// Higher-order: the RESULT type is a function
func makeAdder() func(int, int) {
	return func(x, y int) {
		fmt.Println(x + y)
	}
}

func main() {
	sum := makeAdder() // sum now holds the returned function
	sum(4, 3)          // 7
}
```

Note the return type `func(int, int)`. We can also call it in one go: `makeAdder()(4, 3)`.

### 4.3 Both

```go
package main

import "fmt"

// takes a function AND returns a function
func twice(f func(int) int) func(int) int {
	return func(n int) int {
		return f(f(n))
	}
}

func main() {
	addThree := func(n int) int { return n + 3 }
	addSix := twice(addThree)
	fmt.Println(addSix(10)) // 16
}
```

### 4.4 A useful factory: returning a customized function

```go
package main

import "fmt"

func multiplier(factor int) func(int) int {
	return func(n int) int {
		return n * factor // uses `factor` from the outer function
	}
}

func main() {
	double := multiplier(2)
	triple := multiplier(3)
	fmt.Println(double(10), triple(10)) // 20 30
}
```

Wait, how does the returned function still know `factor` after `multiplier` has finished? Its stack frame should be gone! That "memory" is a **closure**, the subject of Chapters 19–20. For now, accept that it works.

### First-order vs. higher-order at a glance

| | First-order | Higher-order |
|--|-------------|--------------|
| Parameters | data only | may include functions |
| Results | data only | may be a function |
| Example | `func add(a, b int) int` | `func apply(f func(int) int, n int) int` |
| Example | `func greet(s string)` | `func makeAdder() func(int, int)` |

---

## 5. Callbacks

> A **callback** is a function that you **pass to another function** so that the other function can **"call it back"** later.

In Example 4.1, `add` is the callback, and `processOperation` is the higher-order function that calls it.

```
you ──► processOperation(2, 5, add) ──► ... does its own work ...
                                       └─► calls add(2, 5)   ← "calling you back"
```

**Callback** describes a function's *role* (something passed in to be called later). **Higher-order** describes the *receiving* function. The same function can be a callback in one place and a normal function in another.

### Why "callback"?

> You call a shop and leave your number. "We'll call you back when it arrives." You *provided* the means (your number) for someone else to run *your* piece of the process later.

### Real callbacks you will use

```go
package main

import (
	"fmt"
	"sort"
	"strings"
)

func main() {
	words := []string{"banana", "kiwi", "apple", "fig"}

	// sort.Slice calls our "less" callback repeatedly to order the data
	sort.Slice(words, func(i, j int) bool {
		return len(words[i]) < len(words[j])
	})
	fmt.Println(words) // [fig kiwi apple banana]

	// strings.Map calls our callback on every character
	fmt.Println(strings.Map(func(r rune) rune {
		if r == 'a' {
			return 'A'
		}
		return r
	}, "banana")) // bAnAnA
}
```

Callbacks let library authors write *general* code ("sort anything"), and let *you* plug in the specific part ("...by length").

---

## 6. First-class functions

> A language has **first-class functions** when functions are treated like any other value.

A **first-class citizen** of a language is something that can be:

1. **assigned to a variable**
2. **passed as an argument**
3. **returned from a function**
4. **stored in data structures** (slices, maps, structs)

Integers, strings, and booleans obviously qualify. In Go, **functions qualify too**:

```go
package main

import "fmt"

func shout(s string) string { return s + "!" }

func main() {
	// 1. assigned to a variable
	f := shout

	// 2. passed as an argument
	apply := func(g func(string) string, s string) string { return g(s) }
	fmt.Println(apply(f, "hi"))

	// 3. returned from a function
	pick := func() func(string) string { return shout }
	fmt.Println(pick()("go"))

	// 4. stored in a data structure
	table := []func(string) string{shout, func(s string) string { return "<" + s + ">" }}
	for _, fn := range table {
		fmt.Println(fn("x"))
	}
}
```

Output:

```
hi!
go!
x!
<x>
```

**"First-class functions" vs. "first-class citizens":** the second is the general idea (any value with those four abilities); the first says *functions* are such citizens. Same concept applied to functions; interviewers use the terms interchangeably.

**Higher-order functions are possible *because* functions are first-class.** In a language where functions can't be passed or returned, HOFs can't exist.

---

## 7. Memory simulation

```go
package main

import "fmt"

func processOperation(a, b int, op func(int, int)) {
	op(a, b)
}

func add(x, y int) {
	fmt.Println(x + y)
}

func main() {
	processOperation(2, 5, add)
}
```

**Step 1: package scope: names bound to code**

```
┌─────────────────────────────────────┐
│ PACKAGE                             │
│  processOperation → <code>          │
│  add              → <code>          │
│  main             → <code>          │
└─────────────────────────────────────┘
```

**Step 2: `main` calls `processOperation(2, 5, add)`.** The *function value* `add` is copied into parameter `op`. (What's copied is a small reference to `add`'s code, not the code itself.)

```
┌────────────────────────────────┐
│ processOperation               │
│   a  = 2                       │
│   b  = 5                       │
│   op = ► add's code            │  ← a parameter holding a function
├────────────────────────────────┤
│ main                           │
└────────────────────────────────┘
```

**Step 3: inside, `op(a, b)`.** Look up `op` → it points to `add`'s code → run it in a new frame:

```
┌────────────────────────────────┐
│ add    x = 2, y = 5            │  prints 7
├────────────────────────────────┤
│ processOperation  a, b, op     │
├────────────────────────────────┤
│ main                           │
└────────────────────────────────┘
```

**Step 4: `add` returns → frame destroyed. `processOperation` returns → frame destroyed.** Back in `main`. Done.

---

## 8. Where the names come from

*(Optional, but explains the odd vocabulary.)*

The words **first-order** and **higher-order** come from **mathematical logic**:

- **First-order logic** quantifies over *individual things* ("for every number x, ...").
- **Higher-order logic** can also quantify over *properties and functions themselves* ("for every function f, ...").

Functional programming languages took the idea: a **first-order function** works with plain values; a **higher-order function** works with *functions as values*. A related origin: **lambda calculus** (Alonzo Church, 1930s), the mathematical model behind "anonymous functions" (some languages literally write `lambda`).

Go isn't a purely functional language (Chapter 11), but it borrows this idea because it's enormously practical.

---

## 9. Practical patterns

### 9.1 Strategy: behavior as a parameter

```go
package main

import "fmt"

type Discount func(price float64) float64

func checkout(price float64, discount Discount) float64 {
	return discount(price)
}

func main() {
	none := func(p float64) float64 { return p }
	tenOff := func(p float64) float64 { return p * 0.9 }
	flat50 := func(p float64) float64 { return p - 50 }

	fmt.Println(checkout(200, none))   // 200
	fmt.Println(checkout(200, tenOff)) // 180
	fmt.Println(checkout(200, flat50)) // 150
}
```

Adding a new discount means writing one small function; `checkout` never changes. That's the **Open/Closed principle** (the "O" in SOLID) in action.

### 9.2 Decorator (wrap a function to add behavior)

```go
package main

import (
	"fmt"
	"time"
)

func timed(name string, f func()) func() {
	return func() {
		start := time.Now()
		f()
		fmt.Printf("%s took %v\n", name, time.Since(start) > 0)
	}
}

func main() {
	work := func() { time.Sleep(10 * time.Millisecond) }
	timedWork := timed("work", work)
	timedWork() // work took true
}
```

This "wrap a function in another function" idea is exactly **middleware** in web servers: logging, authentication, and so on (Chapters 45–49).

### 9.3 Function factories

`multiplier(3)` in section 4.4 builds a *customized* function from configuration, a technique used for validators, loggers, and route handlers.

### 9.4 Deferred and concurrent work

`defer f()` and `go f()` take function calls; wrappers built on HOFs (`sync.Once.Do(f)`, `time.AfterFunc(d, f)`) take callbacks. Once you spot the pattern, you'll see it everywhere.

---

## 10. A peek at generics

Since Go 1.18, **generics** let a single higher-order function work on any type. This is a preview: you don't need to master the syntax now.

```go
package main

import "fmt"

// Map applies f to every element.
func Map[T, U any](items []T, f func(T) U) []U {
	out := make([]U, 0, len(items))
	for _, it := range items {
		out = append(out, f(it))
	}
	return out
}

// Filter keeps the elements for which keep returns true.
func Filter[T any](items []T, keep func(T) bool) []T {
	var out []T
	for _, it := range items {
		if keep(it) {
			out = append(out, it)
		}
	}
	return out
}

// Reduce folds the slice into a single value.
func Reduce[T, A any](items []T, initial A, f func(A, T) A) A {
	acc := initial
	for _, it := range items {
		acc = f(acc, it)
	}
	return acc
}

func main() {
	nums := []int{1, 2, 3, 4, 5, 6}

	squares := Map(nums, func(n int) int { return n * n })
	evens := Filter(nums, func(n int) bool { return n%2 == 0 })
	sum := Reduce(nums, 0, func(acc, n int) int { return acc + n })
	labels := Map(nums, func(n int) string { return fmt.Sprintf("#%d", n) })

	fmt.Println(squares) // [1 4 9 16 25 36]
	fmt.Println(evens)   // [2 4 6]
	fmt.Println(sum)     // 21
	fmt.Println(labels)  // [#1 #2 #3 #4 #5 #6]
}
```

Three higher-order functions, each taking a callback: the classic toolkit of functional programming, written in a dozen lines.

---

## 11. Interview questions

**Q1. What's the difference between a parameter and an argument?**
A parameter is the variable named in a function's definition; an argument is the value passed in a call.

**Q2. What is a first-order function?**
One that only takes and returns ordinary values, no functions.

**Q3. What is a higher-order function?**
One that takes a function as a parameter or returns a function (or both).

**Q4. What is a callback?**
A function passed as an argument to be invoked later by the receiving function.

**Q5. What are first-class functions?**
Functions are values: they can be assigned, passed, returned, and stored.

**Q6. Is `add` in `processOperation(2, 5, add)` higher-order?**
No. `add` itself is first-order (it takes ints). It *plays the role of a callback*. `processOperation` is the higher-order function.

**Q7. Give three higher-order functions from Go's standard library.**
`sort.Slice`, `strings.Map`, `http.HandleFunc` (takes a handler function), `time.AfterFunc`, `sync.Once.Do`.

---

## 12. Common mistakes

| # | Mistake | Explanation / fix |
|---|---------|-------------------|
| 1 | Passing `add()` instead of `add` | `add()` *calls* it and passes the **result**. Pass the function itself: `add` |
| 2 | Mismatched callback signature | `func(int, int)` ≠ `func(int, int) int`. The types must match exactly |
| 3 | Mixing up parameter/argument in an interview | Placeholder vs. actual value |
| 4 | Calling a `nil` callback | Check `if cb != nil` when callbacks are optional |
| 5 | Thinking a callback is a special kind of function | It's a *role*: any function can be one |
| 6 | Over-engineering with HOFs when a plain loop is clearer | Use them when they simplify |
| 7 | Long anonymous callbacks | Extract into a named function |

Mistake #1 in action:

```go
// INTENTIONAL ERROR
package main

import "fmt"

func add(x, y int) int { return x + y }

func run(f func(int, int) int) { fmt.Println(f(1, 2)) }

func main() {
	run(add(1, 2)) // ❌ cannot use add(1, 2) (value of type int) as func(int, int) int
}
```

---

## 13. Exercises

### Exercise 1: Identify
In the code below, list the **parameters** and the **arguments**.

```go
func area(width, height float64) float64 { return width * height }

func main() {
	a := area(3, 4)
	b := area(10, 2.5)
}
```

<details><summary>Solution</summary>

Parameters: `width`, `height`. Arguments: `3, 4` (first call) and `10, 2.5` (second call).
</details>

### Exercise 2: Classify
Which of these is higher-order?

```go
func a(n int) int
func b(f func(int) int, n int) int
func c() func() string
func d(names ...string)
```

<details><summary>Solution</summary>

`b` (takes a function) and `c` (returns a function). `a` and `d` are first-order.
</details>

### Exercise 3: Write a HOF
Write `applyToAll(nums []int, f func(int) int) []int` that returns a new slice with `f` applied to each element. Use it to double the numbers `1..5`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func applyToAll(nums []int, f func(int) int) []int {
	out := make([]int, len(nums))
	for i, n := range nums {
		out[i] = f(n)
	}
	return out
}

func main() {
	fmt.Println(applyToAll([]int{1, 2, 3, 4, 5}, func(n int) int { return n * 2 }))
	// [2 4 6 8 10]
}
```
</details>

### Exercise 4: Return a function
Write `makeGreeter(greeting string) func(name string) string` so that `makeGreeter("Hello")("Asha")` returns `"Hello, Asha!"`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func makeGreeter(greeting string) func(string) string {
	return func(name string) string {
		return greeting + ", " + name + "!"
	}
}

func main() {
	hello := makeGreeter("Hello")
	namaste := makeGreeter("Namaste")
	fmt.Println(hello("Asha"))   // Hello, Asha!
	fmt.Println(namaste("Rahim")) // Namaste, Rahim!
}
```
</details>

### Exercise 5: Callbacks
Write `retry(times int, action func() error) error` that calls `action` up to `times` times, stopping at the first success and returning the last error if all attempts fail. Test it with an action that fails twice, then succeeds.

<details><summary>Solution</summary>

```go
package main

import (
	"errors"
	"fmt"
)

func retry(times int, action func() error) error {
	var err error
	for i := 1; i <= times; i++ {
		if err = action(); err == nil {
			fmt.Println("succeeded on attempt", i)
			return nil
		}
		fmt.Println("attempt", i, "failed:", err)
	}
	return err
}

func main() {
	calls := 0
	err := retry(5, func() error {
		calls++
		if calls < 3 {
			return errors.New("temporary failure")
		}
		return nil
	})
	fmt.Println("final error:", err)
}
```
Output:
```
attempt 1 failed: temporary failure
attempt 2 failed: temporary failure
succeeded on attempt 3
final error: <nil>
```
</details>

### Exercise 6: Decorator
Write `logged(f func(int) int) func(int) int` that prints the argument and result of each call to `f`. Apply it to a `square` function.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func logged(f func(int) int) func(int) int {
	return func(n int) int {
		result := f(n)
		fmt.Printf("f(%d) = %d\n", n, result)
		return result
	}
}

func main() {
	square := func(n int) int { return n * n }
	loggedSquare := logged(square)
	loggedSquare(4) // f(4) = 16
	loggedSquare(7) // f(7) = 49
}
```
</details>

### Exercise 7 (challenge): Compose
Write `compose(f, g func(int) int) func(int) int` returning a function that computes `f(g(x))`. Use it to build `addOneThenDouble`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func compose(f, g func(int) int) func(int) int {
	return func(x int) int { return f(g(x)) }
}

func main() {
	double := func(n int) int { return n * 2 }
	addOne := func(n int) int { return n + 1 }

	addOneThenDouble := compose(double, addOne) // double(addOne(x))
	fmt.Println(addOneThenDouble(5))            // 12
}
```
</details>

---

## 14. Quiz

1. In `func f(x int)` called as `f(10)`, which is the parameter and which the argument?
2. Give the two ways a function becomes higher-order.
3. What is a callback?
4. What does "functions are first-class" allow you to do?
5. Why is `run(add(1, 2))` wrong when `run` expects a function?

<details><summary>Answers</summary>

1. `x` is the parameter, `10` the argument.
2. It takes a function as a parameter, and/or returns a function.
3. A function passed to another to be called later.
4. Assign, pass, return, and store functions like any other value.
5. `add(1, 2)` *calls* `add` and produces an `int`; `run` needs the function `add` itself.
</details>

---

## 15. Summary

- **Parameter** = placeholder in the definition. **Argument** = actual value in the call.
- **First-order function**: works with plain data only. **Higher-order function**: takes and/or returns functions.
- A **callback** is a function handed to another function to be called later: a *role*, not a type.
- Go has **first-class functions**: they can be assigned, passed, returned, and stored.
- HOFs enable **strategies, decorators/middleware, factories**, and generic helpers like `Map`/`Filter`/`Reduce`.
- Pass the function itself (`add`), not its result (`add()`).
- A returned function can remember variables from where it was created. That's a **closure**, coming in Chapter 20, after we peek inside memory in Chapters 18–19.

### ➡️ What's next?

Part 3 begins. [Chapter 18](18-go-internal-memory.md) opens the hood: **how Go organizes memory**: code segment, data segment, stack, heap, and the garbage collector.
