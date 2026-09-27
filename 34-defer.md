# Chapter 34: `defer` — Delaying Work Until a Function Returns

> **Goal of this chapter:** Master one of Go's most distinctive keywords. `defer` schedules a function call to run **just before the surrounding function returns**, no matter how it exits. You'll learn the LIFO execution order, exactly *when* deferred arguments are evaluated, how `defer` interacts with **named return values** (the topic most engineers get wrong), how it works with `panic`/`recover`, how it's stored in memory, and the idioms and traps you'll meet in real code.

**Difficulty:** 🟠–🔴 Intermediate–Advanced  **Estimated time:** 2.5–3 hours  **Prerequisite:** [Chapters 5, 15–20](05-functions-with-return-values.md) (return values, closures)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why `defer` exists](#2-why-defer-exists)
3. [The basics](#3-the-basics)
4. [Multiple defers: last in, first out](#4-multiple-defers-lifo)
5. [Step-by-step visualization](#5-step-by-step-visualization)
6. [Where does Go store deferred calls?](#6-where-does-go-store-deferred-calls)
7. [Rule 1: arguments are evaluated immediately](#7-rule-1-arguments-are-evaluated-immediately)
8. [Rule 2: deferred closures see later changes](#8-rule-2-deferred-closures-see-later-changes)
9. [Named return values](#9-named-return-values)
10. [Rule 3: `defer` can modify named results](#10-rule-3-defer-can-modify-named-results)
11. [Exactly what happens at `return`](#11-exactly-what-happens-at-return)
12. [`defer` with `panic` and `recover`](#12-defer-with-panic-and-recover)
13. [Everyday idioms](#13-everyday-idioms)
14. [Traps and gotchas](#14-traps-and-gotchas)
15. [Performance](#15-performance)
16. [Challenge questions](#16-challenge-questions)
17. [Common mistakes](#17-common-mistakes)
18. [Exercises](#18-exercises)
19. [Quiz](#19-quiz)
20. [Summary](#20-summary)

---

## 1. What you will learn

- What `defer` does and why every real Go program uses it
- The **LIFO** (stack) order of multiple defers
- The three rules that explain *every* `defer` behavior
- How `defer` and **named results** interact (a famous interview topic)
- How `defer` guarantees cleanup even during a **panic**
- How to use `defer` for closing files, unlocking mutexes, timing, and recovering
- Common traps: defer in loops, ignoring errors from deferred calls, nil functions

---

## 2. Why `defer` exists

Many operations come in pairs: **acquire → release**.

| Acquire | Release |
|---------|---------|
| Open a file | Close it |
| Lock a mutex | Unlock it |
| Open a DB connection | Close it |
| Start a timer | Stop it/report it |
| Begin a transaction | Commit or roll back |

Without `defer`, every exit path must remember the release:

```go
func process(name string) error {
	f, err := os.Open(name)
	if err != nil {
		return err
	}

	data, err := io.ReadAll(f)
	if err != nil {
		f.Close() // ← must not forget!
		return err
	}
	if len(data) == 0 {
		f.Close() // ← again!
		return errors.New("empty file")
	}

	f.Close() // ← and at the end
	return nil
}
```

Three `Close()` calls, and adding another `return` in the middle would easily forget one, leaking the file. With `defer`:

```go
func process(name string) error {
	f, err := os.Open(name)
	if err != nil {
		return err
	}
	defer f.Close() // ← written ONCE, right after acquiring; runs on EVERY exit path

	data, err := io.ReadAll(f)
	if err != nil {
		return err
	}
	if len(data) == 0 {
		return errors.New("empty file")
	}
	return nil
}
```

The release is written **next to** the acquire (easy to review), and it runs on **every** exit: normal returns, early returns, and even panics.

---

## 3. The basics

> **`defer f(args)`** schedules the call `f(args)` to run **when the surrounding function returns**, just before control goes back to its caller.

```go
package main

import "fmt"

func main() {
	fmt.Println("start")
	defer fmt.Println("deferred!") // scheduled; NOT run yet
	fmt.Println("end")
}
```

Output:

```
start
end
deferred!
```

`"deferred!"` prints **last**, even though its line is in the middle. Execution:

1. Print `start`.
2. Reach `defer`: Go *records* the call `fmt.Println("deferred!")` and continues.
3. Print `end`.
4. `main` is about to return → run the recorded calls → print `deferred!`.

```
main starts
  │
  ├─ print "start"
  ├─ defer fmt.Println("deferred!")   ← recorded in a list
  ├─ print "end"
  ▼
main ready to return ──► run deferred list ──► actually return
```

> **Analogy: leaving a note to yourself.** "Before I leave the house, turn off the lights." You write it when you *arrive* (at the top of the function), but the action happens at the *end*, whichever door you leave by.

---

## 4. Multiple defers: LIFO

If you `defer` several calls, they run in **reverse order**: **Last In, First Out**, like a stack of plates.

```go
package main

import "fmt"

func main() {
	fmt.Println("start")
	defer fmt.Println("deferred 1")
	defer fmt.Println("deferred 2")
	defer fmt.Println("deferred 3")
	fmt.Println("end")
}
```

Real output:

```
start
end
deferred 3
deferred 2
deferred 1
```

```
defers recorded:   [1] then [2] then [3]

stack:   ┌───────────┐
         │ deferred 3│ ← top: runs FIRST
         ├───────────┤
         │ deferred 2│
         ├───────────┤
         │ deferred 1│ ← runs LAST
         └───────────┘
```

**Why LIFO?** It mirrors *nesting*. If you open a file, then lock a mutex, you should unlock the mutex **before** closing the file (release in the reverse order of acquisition):

```go
f, _ := os.Open("data")
defer f.Close()     // acquired first  → released last
mu.Lock()
defer mu.Unlock()   // acquired second → released first
```

This is like putting on socks, then shoes: you take them off in the reverse order.

---

## 5. Step-by-step visualization

```go
package main

import "fmt"

func main() {
	fmt.Println("A")
	defer fmt.Println("D1")
	fmt.Println("B")
	defer fmt.Println("D2")
	fmt.Println("C")
}
```

| Step | Executed | Deferred stack (top on the right) | Output so far |
|------|----------|-----------------------------------|---------------|
| 1 | `Println("A")` | `[]` | `A` |
| 2 | `defer D1` | `[D1]` | `A` |
| 3 | `Println("B")` | `[D1]` | `A B` |
| 4 | `defer D2` | `[D1, D2]` | `A B` |
| 5 | `Println("C")` | `[D1, D2]` | `A B C` |
| 6 | function returns → pop `D2` | `[D1]` | `A B C D2` |
| 7 | pop `D1` | `[]` | `A B C D2 D1` |

Final output: `A B C D2 D1` (each on its own line).

---

## 6. Where does Go store deferred calls?

Each **goroutine** keeps a list of its pending deferred calls, conceptually a **stack** (LIFO) attached to the function call that deferred them. Recording a `defer` means pushing an entry (the function to call and its already-evaluated arguments); returning pops and runs the entries belonging to that function.

```
main's stack frame                 Deferred-call list (per function call)
┌──────────────────┐              ┌──────────────────────────────┐
│ local variables  │              │ top → Println("D2")          │
│ ...              │              │       Println("D1")          │
│ pending defers ──┼─────────────►└──────────────────────────────┘
└──────────────────┘
```

Implementation notes (you don't need these to *use* `defer`):
- Historically each `defer` allocated a record on the heap; later versions (Go 1.13) put it on the stack; and since **Go 1.14** most defers are **"open-coded"**: the compiler inlines the deferred call at each function exit and keeps a tiny bitmask on the stack recording which defers were registered. This makes `defer` nearly as cheap as a direct call.
- Defers inside **loops** can't be open-coded (unbounded count), so they use the slower path.
- Each function call has its **own** defer list, tied to *that call's* frame. A deferred call runs when *its own* function returns, not when a caller does.

---

## 7. Rule 1: arguments are evaluated immediately

> **The *arguments* of a deferred call are evaluated at the moment of the `defer` statement, not when the call finally runs.**

```go
package main

import "fmt"

func main() {
	x := 1
	defer fmt.Println("deferred x =", x) // x is evaluated NOW: 1
	x = 100
	fmt.Println("current x =", x)
}
```

Output:

```
current x = 100
deferred x = 1
```

Even though `x` is `100` when the deferred call runs, it prints `1`: the argument was **copied** at the `defer` line. The same applies to the **receiver** of a deferred method call:

```go
package main

import "fmt"

type T struct{ name string }

func (t T) Hello() { fmt.Println("hello", t.name) }

func main() {
	t := T{"a"}
	defer t.Hello() // receiver t is copied NOW (value receiver)
	t.name = "b"
}
```

Output: `hello a`.

---

## 8. Rule 2: deferred closures see later changes

But if you defer an **anonymous function that *uses* a variable** (a closure, Chapter 20), the closure reads the **variable itself** when it runs, so it sees the *latest* value:

```go
package main

import "fmt"

func main() {
	x := 1
	defer fmt.Println("defer with argument sees x =", x) // argument: evaluated now
	defer func() { fmt.Println("closure sees x =", x) }() // captures the variable x
	x = 100
}
```

Real output:

```
closure sees x = 100
defer with argument sees x = 1
```

(Closure runs first: LIFO.) This contrast, **arguments evaluated now** vs. **closure variables read later**, explains most `defer` puzzles.

| Form | `x` seen when the deferred code runs |
|------|-------------------------------------|
| `defer fmt.Println(x)` | value of `x` **at the defer statement** |
| `defer func() { fmt.Println(x) }()` | value of `x` **at the moment it runs** (latest) |
| `defer func(v int) { fmt.Println(v) }(x)` | value **at the defer statement** (passed as an argument) |

---

## 9. Named return values

To understand Rule 3 we need **named return values** (briefly covered in Chapter 5).

### Unnamed result

```go
func normal() int {
	x := 5
	return x
}
```

The result has no name. `return x` computes a value and hands it to the caller.

### Named result

```go
func named() (result int) { // `result` is a variable, declared in the signature
	result = 5
	return // "naked return": returns whatever `result` currently holds
}
```

A **named result** is an ordinary local variable, declared automatically and initialized to its zero value. `return` (with or without a value) *assigns to it* and then returns it.

```
func named() (result int)      ← `result` lives in named()'s stack frame from the start
```

Why this matters: because `result` is a **variable in the frame**, a deferred closure can read and **modify** it after the `return` statement executes.

---

## 10. Rule 3: `defer` can modify named results

The Go specification says:

> *If the deferred function is a function literal and the surrounding function has named result parameters that are in scope within the literal, the deferred function may access and modify the result parameters before they are returned.*

Compare two functions that look almost identical:

```go
package main

import "fmt"

func normal() int {
	x := 5
	defer func() { x = 100 }() // modifies a local variable, but...
	return x                   // ...the return value (5) was already computed
}

func named() (x int) {
	x = 5
	defer func() { x = 100 }() // modifies the RESULT variable itself
	return x
}

func main() {
	fmt.Println(normal()) // 5
	fmt.Println(named())  // 100
}
```

Real output:

```
5
100
```

**Same code shape, different answers.** Why?

**`normal()`**: `return x` *copies* `x`'s current value (5) into a hidden return slot. The deferred closure then changes the local `x` to 100, but the return slot already holds 5. The caller gets `5`.

**`named()`**: the result **is** the variable `x`. `return x` assigns 5 to `x` (a no-op here), then runs deferred functions, and the closure sets `x = 100`. The value returned to the caller is whatever `x` holds *after* the defers finish: `100`.

### A useful application: modify the result on the way out

```go
package main

import "fmt"

func compute() (result int) {
	defer func() { result *= 2 }() // double whatever is returned
	return 21
}

func main() { fmt.Println(compute()) } // 42
```

`return 21` assigns `result = 21`; then the deferred function doubles it → `42`.

---

## 11. Exactly what happens at `return`

Use this as the definitive model. For `return expr`:

```
1. Evaluate `expr`.
2. Assign the value(s) to the result variable(s)   ← named results: the actual variables;
                                                     unnamed: hidden slots
3. Run the deferred functions (LIFO). They may modify NAMED result variables.
4. Return to the caller with the result variables' current values.
```

Let's trace with prints so the order is unmistakable:

```go
package main

import "fmt"

func f() (n int) {
	defer func() {
		fmt.Println("2) deferred runs; n =", n)
		n += 10
	}()
	fmt.Println("1) body runs")
	return func() int {
		fmt.Println("   return expression evaluated")
		return 5
	}()
}

func main() {
	fmt.Println("3) caller receives:", f())
}
```

Output:

```
1) body runs
   return expression evaluated
2) deferred runs; n = 5
3) caller receives: 15
```

Timeline:

```
body runs ─► evaluate `return` expression (5) ─► n = 5 ─► deferred: n += 10 ─► n = 15 ─► return 15
```

And this is the same principle behind the very common **error-wrapping** pattern:

```go
func doWork() (err error) {
	defer func() {
		if err != nil {
			err = fmt.Errorf("doWork failed: %w", err) // decorate every error on the way out
		}
	}()
	// ... many places that `return someErr`
	return errors.New("disk full")
}
```

---

## 12. `defer` with `panic` and `recover`

When a function **panics** (a run-time error such as an out-of-range index, nil pointer, or an explicit `panic(...)` call), Go **unwinds the stack**, running each function's **deferred calls** on the way. This is why `defer` is the right place for cleanup: **it runs even when the code crashes**.

```go
package main

import "fmt"

func main() {
	defer fmt.Println("cleanup runs even on panic")
	defer func() {
		r := recover() // stop the panic and get its value
		fmt.Println("recovered:", r)
	}()

	panic("boom")
}
```

Output:

```
recovered: boom
cleanup runs even on panic
```

- `panic("boom")` starts unwinding. Deferred calls run in LIFO order: first the closure (which **recovers**), then the `Println` cleanup.
- **`recover()`** stops the panic and returns the panic value, **but only when called *directly* inside a deferred function**. Elsewhere it returns `nil`.
- After recovery, `main` returns normally.

### Turning a panic into an error (named results again)

```go
package main

import "fmt"

func safeDivide(a, b int) (result int, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("recovered: %v", r) // set the NAMED result
		}
	}()
	return a / b, nil // b == 0 → runtime panic: integer divide by zero
}

func main() {
	fmt.Println(safeDivide(10, 2)) // 5 <nil>
	fmt.Println(safeDivide(1, 0))  // 0 recovered: runtime error: integer divide by zero
}
```

This is exactly how web servers survive a buggy handler without dying (the standard library's `net/http` does this per request; you'll write a "recovery middleware" in Chapter 46). Recover sparingly: prefer returning `error`s for expected failures, and use `panic` for programmer errors and truly unrecoverable states.

### Real recovery from an out-of-range index

```go
func retNamed() (n int, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("recovered: %v", r)
		}
	}()
	var arr []int
	return arr[3], nil // panics
}
// prints: 0 recovered: runtime error: index out of range [3] with length 0
```

---

## 13. Everyday idioms

### Close resources

```go
f, err := os.Open("data.txt")
if err != nil {
	return err
}
defer f.Close() // immediately after checking the error
```

> ⚠️ Place the `defer` **after** the error check. If `Open` failed, `f` is `nil` and deferring `f.Close()` would be wrong.

### Unlock mutexes (Chapter 68)

```go
mu.Lock()
defer mu.Unlock() // released on every path, including panics
// ... critical section
```

### Wait-group bookkeeping (Chapter 65)

```go
go func() {
	defer wg.Done() // always signal completion, even on early return
	// ... work
}()
```

### Time a function

```go
package main

import (
	"fmt"
	"time"
)

func track(name string) func() {
	start := time.Now()
	return func() { fmt.Printf("%s took %v\n", name, time.Since(start).Round(time.Millisecond)) }
}

func slowWork() {
	defer track("slowWork")() // NOTE the trailing () : track(...) runs NOW, its result runs LATER
	time.Sleep(120 * time.Millisecond)
}

func main() { slowWork() } // slowWork took 120ms
```

The double call is a classic: `track("slowWork")` is evaluated immediately (Rule 1, recording the start time), and the function it returns is what's deferred.

### Transactions: rollback unless committed

```go
tx, err := db.Begin()
if err != nil {
	return err
}
defer tx.Rollback() // no-op if already committed; undoes work if we return early

// ... do queries, return on error ...

return tx.Commit()
```

### Logging enter/exit

```go
func handle() {
	log.Println("enter handle")
	defer log.Println("exit handle")
	// ...
}
```

### Cleaning up temporary files

```go
f, _ := os.CreateTemp("", "scratch")
defer os.Remove(f.Name()) // runs last (LIFO)...
defer f.Close()           // ...after this one
```

---

## 14. Traps and gotchas

### Trap 1: `defer` in a loop

Deferred calls run when the **function** returns, not at the end of each **loop iteration**:

```go
package main

import "fmt"

func main() {
	for i := 0; i < 3; i++ {
		defer fmt.Println("loop defer", i)
	}
	fmt.Println("loop finished")
}
```

Output:

```
loop finished
loop defer 2
loop defer 1
loop defer 0
```

That's harmless here, but with resources it's a **leak**:

```go
// ❌ BAD: none of the files close until processAll returns: you may run out of file handles
func processAll(paths []string) error {
	for _, p := range paths {
		f, err := os.Open(p)
		if err != nil {
			return err
		}
		defer f.Close()
		// ... use f ...
	}
	return nil
}

// ✅ GOOD: put the loop body in its own function so each defer runs per iteration
func processAll(paths []string) error {
	for _, p := range paths {
		if err := processOne(p); err != nil {
			return err
		}
	}
	return nil
}

func processOne(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return err
	}
	defer f.Close()
	// ... use f ...
	return nil
}
```

(Or wrap the body in an immediately invoked function literal: `func() { defer f.Close(); ... }()`.)

### Trap 2: ignoring errors from deferred calls

`defer f.Close()` discards `Close`'s error. That's fine for read-only files, but for **writes**, `Close` may report a failed flush. Capture it via a named result:

```go
func writeFile(name string, data []byte) (err error) {
	f, err := os.Create(name)
	if err != nil {
		return err
	}
	defer func() {
		if cerr := f.Close(); cerr != nil && err == nil {
			err = cerr // keep the FIRST error, but don't lose a close failure
		}
	}()
	_, err = f.Write(data)
	return err
}
```

### Trap 3: `defer` with a `nil` function value

```go
var f func()
defer f() // the defer statement is fine, but when the function returns → panic (nil function call)
```

The panic happens when the deferred call *executes*, not at the `defer` line.

### Trap 4: `os.Exit` skips defers

`os.Exit(1)` terminates the process **immediately**: **no deferred functions run**. Same for `log.Fatal` (which calls `os.Exit`). Structure `main` so real work happens in a function that *returns*, and call `os.Exit` only at the very end:

```go
func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1) // safe: run() already finished and ran ITS defers
	}
}

func run() error {
	defer cleanup() // runs when run() returns
	// ...
	return nil
}
```

### Trap 5: `recover` only works directly inside the deferred function

```go
defer func() {
	helper() // if helper() calls recover(), it returns nil: too deep
}()
```

Call `recover()` in the deferred function's body itself.

### Trap 6: `defer` doesn't change *which* value a value-receiver captured

See Rule 1: `defer t.Hello()` copies `t` immediately. Use a closure or a pointer receiver if you want the latest state.

---

## 15. Performance

Since Go 1.14, `defer` is very cheap (open-coded defers cost around a nanosecond or two in the common case). **Use it freely** for correctness; don't avoid it "for speed" except in extremely hot loops that you've *measured*. The main real cost is `defer` **inside loops** (heap-allocated records, plus delayed execution), which is a correctness problem before it's a performance one.

---

## 16. Challenge questions

*Try to answer before reading the solutions. These are typical of senior-level interviews.*

### Challenge 1

```go
func f() (result int) {
	defer func() { result++ }()
	return 0
}
```
What does `f()` return?

<details><summary>Answer</summary>

`1`. `return 0` sets `result = 0`; the deferred function increments it to 1; the caller gets 1.
</details>

### Challenge 2

```go
func f() int {
	result := 0
	defer func() { result++ }()
	return result
}
```

<details><summary>Answer</summary>

`0`. The result isn't named, so `return result` copied `0` into a hidden slot; incrementing the local afterward doesn't affect it.
</details>

### Challenge 3

```go
func f() (r int) {
	defer func(v int) { r = v * 2 }(r)
	r = 3
	return r
}
```

<details><summary>Answer</summary>

`0`. The argument `r` is evaluated at the defer statement, when `r` is still `0`; the deferred function sets `r = 0 * 2 = 0`, overwriting the `3`. (Rule 1 + Rule 3.)
</details>

### Challenge 4

```go
func main() {
	for i := 0; i < 3; i++ {
		defer func() { fmt.Print(i, " ") }()
	}
}
```
With Go 1.22+? With Go 1.21?

<details><summary>Answer</summary>

Go 1.22+: `2 1 0` (each iteration has its own `i`; LIFO order). Go 1.21: `3 3 3` (one shared `i`, which ended at 3).
</details>

### Challenge 5

```go
func f() {
	defer fmt.Println(1)
	defer func() {
		defer fmt.Println(2)
		fmt.Println(3)
	}()
	fmt.Println(4)
}
```

<details><summary>Answer</summary>

`4`, `3`, `2`, `1`. Body prints 4. Return → run the top defer (the closure): it prints 3, and *its own* deferred `Println(2)` runs when the closure returns → 2. Then the first defer → 1.
</details>

---

## 17. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Expecting defers to run in written order | They run **LIFO** | Reverse your mental order |
| 2 | Using a variable in `defer fmt.Println(x)` and expecting its later value | Arguments are evaluated immediately | Use a closure |
| 3 | `defer` in a loop that opens resources | Resources held until function end | Extract into a function |
| 4 | `defer f.Close()` before checking `err` | `f` may be `nil` | Check the error first |
| 5 | Forgetting `()`: `defer track("x")` vs. `defer track("x")()` | The returned func never runs | Add the trailing call |
| 6 | Expecting `defer` to run after `os.Exit`/`log.Fatal` | It won't | Return from `run()` and exit in `main` |
| 7 | Modifying an unnamed result in a defer | No effect | Use named results |
| 8 | Ignoring `Close()` errors on written files | Silent data loss | Capture via named result |
| 9 | Calling `recover()` in a helper or outside a defer | Returns `nil` | Call it directly in the deferred function |
| 10 | Using `defer` for critical ordering in very hot paths without measuring | Usually unnecessary worry | Measure first |

---

## 18. Exercises

### Exercise 1: Predict the order

```go
func main() {
	defer fmt.Println("a")
	fmt.Println("b")
	defer fmt.Println("c")
	fmt.Println("d")
}
```

<details><summary>Solution</summary>

`b`, `d`, `c`, `a`.
</details>

### Exercise 2: Argument timing
What does this print?

```go
func main() {
	msg := "hello"
	defer fmt.Println(msg)
	msg = "goodbye"
	defer func() { fmt.Println(msg) }()
	msg = "see you"
}
```

<details><summary>Solution</summary>

`see you` (the closure runs first and sees the latest value), then `hello` (the argument was evaluated at the defer line).
</details>

### Exercise 3: Named or not?
Which function returns 10 and which returns 5?

```go
func a() (n int) { n = 5; defer func() { n = 10 }(); return n }
func b() int      { n := 5; defer func() { n = 10 }(); return n }
```

<details><summary>Solution</summary>

`a()` returns **10**; `b()` returns **5**.
</details>

### Exercise 4: Safe file reader
Write `readFirstLine(path string) (line string, err error)` that opens a file, defers `Close`, and returns its first line. Make sure the function closes the file on every path.

<details><summary>Solution</summary>

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func readFirstLine(path string) (line string, err error) {
	f, err := os.Open(path)
	if err != nil {
		return "", err
	}
	defer f.Close()

	sc := bufio.NewScanner(f)
	if sc.Scan() {
		return sc.Text(), nil
	}
	return "", sc.Err()
}

func main() {
	line, err := readFirstLine("/etc/hostname")
	fmt.Println(line, err)
}
```
</details>

### Exercise 5: Recover into an error
Write `mustPositive(n int) (err error)` that calls `panic` if `n <= 0`, but recovers and returns it as an `error` instead.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func mustPositive(n int) (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("invalid input: %v", r)
		}
	}()
	if n <= 0 {
		panic(fmt.Sprintf("%d is not positive", n))
	}
	return nil
}

func main() {
	fmt.Println(mustPositive(3))  // <nil>
	fmt.Println(mustPositive(-1)) // invalid input: -1 is not positive
}
```
</details>

### Exercise 6: Timing decorator
Use the `track` pattern from section 13 to time two different functions, printing their names and durations.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"time"
)

func track(name string) func() {
	start := time.Now()
	return func() { fmt.Printf("%s took %v\n", name, time.Since(start).Round(time.Millisecond)) }
}

func a() { defer track("a")(); time.Sleep(30 * time.Millisecond) }
func b() { defer track("b")(); time.Sleep(60 * time.Millisecond) }

func main() { a(); b() }
```
</details>

### Exercise 7 (challenge): Fix the leak
This function opens N files but only closes them at the end. Fix it without changing its behavior.

```go
func sizes(paths []string) ([]int64, error) {
	var out []int64
	for _, p := range paths {
		f, err := os.Open(p)
		if err != nil {
			return nil, err
		}
		defer f.Close()
		info, err := f.Stat()
		if err != nil {
			return nil, err
		}
		out = append(out, info.Size())
	}
	return out, nil
}
```

<details><summary>Solution</summary>

```go
func fileSize(path string) (int64, error) {
	f, err := os.Open(path)
	if err != nil {
		return 0, err
	}
	defer f.Close() // now runs at the end of EACH fileSize call
	info, err := f.Stat()
	if err != nil {
		return 0, err
	}
	return info.Size(), nil
}

func sizes(paths []string) ([]int64, error) {
	var out []int64
	for _, p := range paths {
		n, err := fileSize(p)
		if err != nil {
			return nil, err
		}
		out = append(out, n)
	}
	return out, nil
}
```
</details>

---

## 19. Quiz

1. When does a deferred call run?
2. In what order do multiple deferred calls run?
3. When are a deferred call's arguments evaluated?
4. Why can a deferred closure change a named result but not an unnamed one?
5. Does `defer` run if the function panics? If it calls `os.Exit`?
6. Where must `recover()` be called for it to work?

<details><summary>Answers</summary>

1. Just before the surrounding function returns (on every exit path).
2. Last in, first out.
3. When the `defer` statement executes.
4. A named result is a variable that still exists (and is what's returned) after defers run; an unnamed result is copied to a hidden slot before defers run.
5. Yes for panics; no for `os.Exit`.
6. Directly inside a deferred function.
</details>

---

## 20. Summary

- **`defer f(args)`** schedules `f(args)` to run when the **surrounding function returns**, on **every** exit path, including panics.
- Multiple defers run **LIFO**, matching the reverse order of acquisition.
- **Rule 1:** a deferred call's **arguments (and method receiver) are evaluated at the `defer` line.**
- **Rule 2:** a deferred **closure** reads captured variables **when it runs** (latest values).
- **Rule 3:** a deferred function literal can **read and modify named result variables** after `return` has assigned them; the caller gets the final values.
- At `return`: evaluate expression → assign to result variables → run defers → return.
- `defer` + `recover()` turns panics into errors; `recover` must be called directly in the deferred function.
- Idioms: `Close`, `Unlock`, `wg.Done`, `tx.Rollback`, timing, logging. Traps: defers in loops, ignored close errors, `os.Exit` skipping defers, nil function values.
- `defer` is cheap (open-coded since Go 1.14): use it for correctness.

### ➡️ What's next?

[Chapter 35](35-a-separate-stack-for-every-thread.md) returns to the thread story: **why each thread needs its own stack**, and how stacks are laid out in memory. This is the last piece before we finally meet **goroutines**.
