# Chapter 19: End of Internal Memory — Function Expressions in Memory, Compilation vs. Execution

> **Goal of this chapter:** Finish the memory story. You'll learn the two phases of every Go program (**compile** and **execute**), what a **binary** really is, and answer the question *"where does a function expression live?"* This sets the stage for **closures** in Chapter 20.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 2 hours  **Prerequisite:** [Chapter 18](18-go-internal-memory.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The open question](#2-the-open-question)
3. [Two phases: compilation and execution](#3-two-phases-compilation-and-execution)
4. [What is a binary file?](#4-what-is-a-binary-file)
5. [Compilation commands, and what they produce](#5-compilation-commands)
6. [Where does a function expression live?](#6-where-does-a-function-expression-live)
7. [Full simulation](#7-full-simulation)
8. [References in stack frames](#8-references-in-stack-frames)
9. [Scope and encapsulation, at the memory level](#9-scope-and-encapsulation-at-the-memory-level)
10. [A limitation: what if the inner function needs the outer function's variables?](#10-a-limitation-and-a-teaser)
11. [Prove it yourself](#11-prove-it-yourself)
12. [Common mistakes](#12-common-mistakes)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- The two phases of a program's life: **compile time** and **run time**
- What an executable/binary file is, and why you can ship one without Go installed
- How to build for *another* operating system with one command
- Exactly where a function expression's **code** and **variable** live
- What a **reference** is, and what "a variable holding a function" contains
- How memory explains **encapsulation**

---

## 2. The open question

At the end of Chapter 18 we had this:

```go
func call() {
	add := func(x, y int) { // function expression inside call()
		z := x + y
		fmt.Println(z)
	}

	add(5, 6)
	add(10, 3)
}
```

**Where is `add` stored?**

- In the code segment?
- In `call`'s stack frame?
- Somewhere else?

The answer is "**both, in different senses**": the *code* is in the code segment; the *variable* is in the stack frame; and the variable holds a *reference* to the code. To see why, we first need the two-phase model.

---

## 3. Two phases: compilation and execution

```
     ┌────────────────────┐           ┌──────────────────────┐
     │  PHASE 1: COMPILE  │           │  PHASE 2: EXECUTE    │
     │  (on the developer │  binary   │  (on the user's      │
     │   machine, once)   │  ───────► │   machine, every run)│
     └────────────────────┘           └──────────────────────┘
       source → machine code             load → init → main
```

### Phase 1: compilation ("build")

Done by `go build` (or implicitly by `go run`). The compiler and linker:

1. **Parse** your source, checking syntax.
2. **Type-check**: every variable has a valid type, every call has the right arguments, no unused imports/variables.
3. **Resolve names** (scope rules from Chapters 8–12) and report `undefined: x`.
4. **Generate machine code** for every function (including anonymous ones).
5. **Link** everything together (your code + the packages you import + the Go runtime) into one file.

Phase 1 produces the **binary**. *No program logic has run yet.* If compilation fails, you never reach phase 2. Errors such as "declared and not used" and "cannot use string as int" are **compile-time** errors.

### Phase 2: execution ("run")

The operating system **loads** the binary into memory and starts it:

1. Code is placed in the **code segment**; package variables in the **data segment**.
2. The Go **runtime** starts (memory allocator, garbage collector, scheduler).
3. Package variables are initialized, `init()` functions run, then `main()` runs (Chapter 14).
4. Frames are pushed and popped on the **stack** as functions are called.

Errors that occur here are **run-time** errors: division by zero, nil pointer dereference, index out of range.

### What happens in each phase

| | Compile time | Run time |
|--|--------------|----------|
| Who | Compiler + linker | The CPU running your binary |
| When | Once per build | Every execution |
| Sees | Source code | Machine code + real data |
| Detects | Syntax, type, scope, unused-variable errors | Panics, logic errors, I/O failures |
| Produces | The binary | Your program's actual behavior |
| Scope resolution | ✅ Done here | ❌ Already baked into machine code |

> **Analogy: a recipe book vs. cooking.** Compiling is a translator converting the recipe into a language the kitchen robot understands (once). Running is the robot cooking the meal (each time someone orders).

---

## 4. What is a binary file?

A **binary** (executable) is a file containing machine code and data that the operating system can run directly. It's called *binary* because its contents are raw bytes (0s and 1s), not text you can read.

```
main.go  (text)   ──go build──►   myprogram  (binary)
```

Try opening a binary in a text editor and you'll see gibberish. Instead, inspect with tools:

```bash
file myprogram
# myprogram: ELF 64-bit LSB executable, x86-64, statically linked, ...
```

Formats depend on the OS: **ELF** (Linux), **Mach-O** (macOS), **PE** (Windows `.exe`).

### Go's superpower: static linking

A Go binary is normally **statically linked**: it contains everything it needs, including the Go runtime. You can copy it to another machine with the same OS and CPU and run it **without installing Go** (or any libraries). This is why Go is beloved for tools and servers: deployment is "copy one file".

### Cross-compilation

Because the compiler itself produces the machine code, you can build for **other** systems by setting two environment variables:

```bash
GOOS=windows GOARCH=amd64 go build -o myprogram.exe .
GOOS=linux   GOARCH=arm64 go build -o myprogram_arm .
GOOS=darwin  GOARCH=arm64 go build -o myprogram_mac .
```

Real output of `file` for the results of the first two (built on Linux):

```
myprogram.exe: PE32+ executable (console) x86-64, for MS Windows
myprogram_arm: ELF 64-bit LSB executable, ARM aarch64, statically linked
```

List every supported target with `go tool dist list`.

---

## 5. Compilation commands

| Command | What it does | Produces a file? |
|---------|--------------|------------------|
| `go build` | Compiles the package in the current directory. If it's `package main`, writes an executable | ✅ executable (named after the folder/module) |
| `go build -o name` | Same, choosing the output file name | ✅ |
| `go run .` | Compiles to a **temporary** binary, runs it, deletes it | ❌ (temp only) |
| `go install` | Builds and puts the binary into `$GOPATH/bin` (`~/go/bin`) | ✅ (in bin dir) |
| `go vet ./...` | Static checks without producing a binary | ❌ |
| `go build ./...` | Compiles every package (checks it all builds) | ❌ for libraries |

### Libraries don't produce binaries

A package that isn't `package main` has no entry point, so there's nothing to run. `go build` on it just checks that it compiles:

```
$ go build ./lib      # succeeds silently, no file created
```

And a `package main` **without** a `func main()` fails:

```
function main is undeclared in the main package
```

So: **executable = `package main` + `func main()`**.

---

## 6. Where does a function expression live?

Given:

```go
add := func(x, y int) {
	z := x + y
	fmt.Println(z)
}
```

There are really **two separate things**:

| Thing | What it is | Where | When created |
|-------|-----------|-------|--------------|
| **The function's code** | The machine instructions for `x + y` and the print | **Code segment** | Compile time (before the program runs) |
| **The variable `add`** | A local variable whose value is a **reference** to that code | **Stack frame** of the enclosing function | When execution reaches `add := ...` |

```
STACK                                  CODE SEGMENT (read-only)
┌───────────────────────────┐          ┌───────────────────────────────┐
│ call's frame              │          │  #1 init                      │
│   add = ─ ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─►│  #2 call                      │
│         (reference)       │          │  #3 call.func1 ◄── the code   │
└───────────────────────────┘          │  #4 main                      │
                                       └───────────────────────────────┘
```

The variable `add` holds something like *"the code lives at address #3"*. When we write `add(5, 6)`, the runtime reads that reference, jumps to that code, and runs it in a **new frame**.

### Why is this a good design?

1. **Memory efficient:** the code exists **once**, no matter how many variables refer to it or how many times it's called.
2. **Safe:** the code segment is read-only, so nobody can modify it at run time.
3. **Fast to "create":** evaluating `func(...) {...}` doesn't compile or copy anything; it just produces a reference.
4. **Reusable:** every call creates a fresh frame, but the code stays put.

This is exactly the same as for named functions (Chapter 13): a named function's name is a package-scope entry pointing into the code segment. The difference with a function expression is *where the variable holding the reference lives*: in a local frame instead of package scope.

---

## 7. Full simulation

```go
package main

import "fmt"

const a = 10
var p = 100

func init() {
	fmt.Println("Hello")
}

func call() {
	add := func(x, y int) {
		z := x + y
		fmt.Println(z)
	}
	add(5, 6)
	add(p, a)
}

func main() {
	call()
	fmt.Println(a)
}
```

Real output:

```
Hello
11
110
10
```

### Phase 1: compile time

```
CODE SEGMENT                      DATA SEGMENT
┌────────────────────────┐        ┌──────────────┐
│ main.init.0            │        │ p = 100      │
│ main.call              │        └──────────────┘
│ main.call.func1        │  ← anonymous function: compiled NOW
│ main.main              │
└────────────────────────┘
```

The compiler gave the anonymous function a generated name like `main.call.func1` ("the first function literal inside `call`"). It's compiled with everything else, before anything runs. (In real optimized builds the compiler may even *inline* `func1` into `call`, so it may not show up as a separate symbol, but the *model* stays the same.)

### Phase 2: run time

**1. Start-up.** The runtime initializes package variables (`p = 100`), then runs `init`:

```
init frame → prints "Hello" → popped
```

**2. `main` starts.** Stack: `[main]`.

**3. `main` calls `call()`.** Stack: `[call, main]`. `call`'s frame is created with room for `add`.

**4. `add := func...` executes.** The variable `add` in `call`'s frame gets the reference to `call.func1`:

```
┌─────────────────────────────┐
│ call   add = ► func1        │
├─────────────────────────────┤
│ main                        │
└─────────────────────────────┘
```

**5. `add(5, 6)`.** Read `add`, follow the reference, run `func1` in a new frame:

```
┌─────────────────────────────┐
│ func1  x=5  y=6  z=11       │  → prints 11
├─────────────────────────────┤
│ call   add = ► func1        │
├─────────────────────────────┤
│ main                        │
└─────────────────────────────┘
```

Then the `func1` frame is popped.

**6. `add(p, a)`.** Evaluate arguments: `p` → not in `call`'s frame → found in data segment (100); `a` = 10. New frame `x=100, y=10, z=110` → prints `110` → popped.

**7. `call` returns.** Its frame, with `add`, is popped. The **code** of `func1` is untouched in the code segment.

**8. `main` prints `10`, returns.** Program ends.

---

## 8. References in stack frames

A **reference** is just a **memory address**. Think of every location in memory as having a house number:

```
Code segment "street":
 #1     #2     #3          #4
 init   call   call.func1  main
```

Variables can hold *values* (`z = 11`) or *references* (`add = #3`). When `add` is called Go:

1. Reads the variable → gets `#3`.
2. Jumps to the instructions at `#3`.
3. Creates a new frame for that call.

Two variables can hold the **same reference**, and neither copies the code:

```go
package main

import "fmt"

func main() {
	f := func() { fmt.Println("hi") }
	g := f // g gets the SAME reference; the code isn't duplicated
	f()
	g()
}
```

```
main frame:  f = ► #3
             g = ► #3      both point at one copy of the code
```

Two different function *literals*, even with identical text, get **different** code addresses. (That's also why you can't compare functions with `==`: only against `nil`.)

### The `add(p, a)` lookup, in memory terms

```
add(p, a)
     │  └─ constant 10 (baked into the code)
     └─ p: is it in call's frame? NO → is it in the data segment? YES → 100
```

At **run time** there's no real "search": the compiler already decided (by scope rules) that `p` means the data-segment variable and emitted a direct address. The picture is a good *model of the rules*; the hardware just uses the resolved address.

---

## 9. Scope and encapsulation, at the memory level

**Who can call the inner function?**

```go
package main

import "fmt"

func call() {
	add := func(x, y int) { fmt.Println(x + y) }
	add(5, 6) // ✅ visible: add is a local of call
}

func main() {
	call()
	// add(1, 2) // ❌ undefined: add: the VARIABLE `add` isn't in main's scope
}
```

Even though the **code** of `func1` sits in the shared code segment, the only *handle* to it is the **variable `add`**, which exists in `call`'s frame and is visible only inside `call`. No handle in scope ⇒ no way to invoke it.

```
CODE SEGMENT:  func1 (exists, but unnamed at package level: nobody outside can name it)
call's frame:  add = ► func1    ← the only handle; local to call
main's frame:  (no handle)
```

That's **encapsulation**: hiding a helper so only the enclosing function can use it. Chapter 10 did it at package level with lowercase names; this does it at function level with local function expressions. You can even **hand out** the handle deliberately (return the function) to allow controlled access, which is exactly how closures create private state (Chapter 20).

---

## 10. A limitation, and a teaser

So far, our function expressions only used their **own** parameters and locals (`x`, `y`, `z`) and package variables (`p`). What if a function expression uses a **local variable of its enclosing function**?

```go
package main

import "fmt"

func call() {
	bonus := 100 // local to call

	add := func(x, y int) {
		fmt.Println(x + y + bonus) // uses call's local variable!
	}

	add(5, 6)
}

func main() { call() } // 111
```

This works, but it raises a deep question: `bonus` lives in `call`'s frame. What if `call` **returns `add`** and `call`'s frame is popped? Would `add` still find `bonus`?

```go
func makeAdder() func(int, int) {
	bonus := 100
	return func(x, y int) { fmt.Println(x + y + bonus) }
}
```

In C-like languages this would be a disaster (dangling reference). In Go it **just works**: the compiler notices `bonus` outlives `makeAdder`'s frame (escape analysis, Chapter 18) and moves it to the **heap**. The function value carries a reference to `bonus`, a **closure**. That's Chapter 20.

---

## 11. Prove it yourself

Save the Section 7 program in a folder with `go mod init demo`, then:

```bash
go build -o demo .
./demo                         # Hello / 11 / 110 / 10
go tool nm demo | grep -E ' main\.'
```

Real output (yours will differ in addresses):

```
  498280 T main.call
  498220 T main.init.0
  498340 T main.main
  563368 D main.p
```

- `T` = code segment: the functions `call`, `init` (auto-named `init.0`), `main`.
- `D` = data segment: `main.p`.
- `call.func1` isn't listed because the compiler inlined the tiny anonymous function into `call`. To stop inlining and see it, build with `go build -gcflags=-l` and run `nm` again; you should see `main.call.func1` marked `T`.

Also try:

```bash
file demo                          # ELF executable, statically linked
GOOS=windows go build -o demo.exe . && file demo.exe
```

---

## 12. Common mistakes

| # | Mistake | Truth |
|---|---------|-------|
| 1 | "The function is created when execution reaches `func(...)`" | The **code** is created at compile time; only the **reference** is created at run time |
| 2 | "Each call copies the function's code" | Code exists once; each call makes a **frame** for variables |
| 3 | "Scope errors happen at run time" | `undefined: x` is a **compile-time** error |
| 4 | "`go run` doesn't compile" | It compiles to a temp binary, then runs it |
| 5 | "Libraries produce executables" | Only `package main` with `func main` does |
| 6 | "A binary needs Go installed to run" | Go binaries are self-contained |
| 7 | "Compile errors and run-time errors are the same" | Compile errors stop the build; run-time errors (panics) happen while running |
| 8 | "Two function values with identical code share an address" | Each literal is compiled separately (different addresses) |
| 9 | Comparing functions with `==` | Not allowed (only `== nil`) |

---

## 13. Exercises

### Exercise 1: Classify the error
Is each error found at compile time or run time?

(a) `x := 5` never used. (b) `10 / zero` where `zero := 0` is a variable. (c) `var s string = 42`. (d) Indexing `nums[10]` on a 3-element slice using a variable index. (e) Calling `foo()` when no `foo` exists.

<details><summary>Solution</summary>

(a) compile. (b) **run time** (`integer divide by zero` panic; dividing by the *constant* `0` would be caught at compile time). (c) compile (type mismatch). (d) run time (index out of range). (e) compile (`undefined: foo`).
</details>

### Exercise 2: Where does it live?
For the program below, say where each of these lives (code, data, stack, heap): (1) the machine code for `greet`, (2) the variable `f`, (3) the variable `count`.

```go
var count = 0

func main() {
	f := func() { count++ }
	f()
}
```

<details><summary>Solution</summary>

(1) `greet` doesn't exist in this program; the anonymous function's code is in the **code segment**. (2) `f` is a local variable → **stack frame of `main`**, holding a reference to the code. (3) `count` is a package variable → **data segment**.
</details>

### Exercise 3: Draw it
Draw the stack (with references) at the moment `add(5, 6)` is running for the Section 7 program.

<details><summary>Solution</summary>

Top to bottom: `func1 (x=5, y=6, z=11)`, `call (add = ► func1)`, `main`. Code segment contains `func1`.
</details>

### Exercise 4: Cross-compile
Build the same program for Windows and for Linux ARM64. What tool confirms the file type?

<details><summary>Solution</summary>

`GOOS=windows GOARCH=amd64 go build -o demo.exe .` and `GOOS=linux GOARCH=arm64 go build -o demo_arm .`; then `file demo.exe demo_arm`.
</details>

### Exercise 5: Same code, one copy?
After `f := func(){...}; g := f`, how many copies of the code exist? How many references?

<details><summary>Solution</summary>

One copy of the code in the code segment; two variables (`f`, `g`) holding references to it.
</details>

### Exercise 6 (challenge): Predict the failure
Which phase reports each problem, and what is the message?

```go
// INTENTIONAL ERROR: unused variable
package main

import "fmt"

func main() {
	nums := []int{1, 2, 3}
	i := 5
	fmt.Println(nums[i])
	var unused int
}
```

<details><summary>Solution</summary>

`declared and not used: unused` is reported at **compile time**, so the program never runs. If you remove `unused`, it compiles, and then at **run time** it panics with `index out of range [5] with length 3`.
</details>

---

## 14. Quiz

1. What are the two phases of a Go program's life?
2. Where is the machine code of an anonymous function stored?
3. What does the variable `add` contain after `add := func(){...}`?
4. Does `go run` produce a permanent file?
5. Why can a Go binary run on a machine without Go installed?
6. Can code outside `call` invoke `call`'s local function expression?

<details><summary>Answers</summary>

1. Compilation and execution.
2. In the code segment, generated at compile time.
3. A reference (address) to the function's code.
4. No, a temporary binary that's deleted afterward.
5. It's statically linked and includes the Go runtime.
6. No: it has no handle, since the variable is local to `call`.
</details>

---

## 15. Summary

- Every program has **two phases**: **compile time** (source → binary; syntax, types, scope resolved) and **run time** (binary loaded → `init` → `main`).
- A **binary** is a self-contained executable; Go can **cross-compile** for other OSes/CPUs with `GOOS`/`GOARCH`.
- A function expression's **code** is generated at **compile time** in the **code segment**; the **variable** holding it lives in a **stack frame** and stores a **reference**.
- Calling through the variable jumps to the code and creates a **new frame** each time.
- The variable is the only handle, so **scope = access control**: an unreachable variable means an uncallable function (**encapsulation**).
- If an inner function uses its enclosing function's variables and outlives it, those variables move to the heap: a **closure**.

### ➡️ What's next?

[Chapter 20](20-closures.md) is where it all comes together: **closures**, functions that remember the variables around them.
