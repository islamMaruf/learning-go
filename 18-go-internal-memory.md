# Chapter 18: Go Internal Memory — Code Segment, Data Segment, Stack, Heap, and the Garbage Collector

> **Goal of this chapter:** Look under the hood. You'll learn how a running Go program organizes RAM into **four regions**, what lives in each, what a **stack frame** really is, why some variables live on the **heap**, and what the **garbage collector** does. We'll also verify claims on your own machine with real commands.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 2–2.5 hours  **Prerequisite:** [Chapter 17](17-parameters-vs-arguments-first-order-vs-higher-order-functions.md) (and the "stack frame" idea from Chapters 4 and 11)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The simplified model you've used so far](#2-the-simplified-model-so-far)
3. [The fuller picture: four regions of memory](#3-the-four-regions)
4. [Stack frames in detail](#4-stack-frames-in-detail)
5. [Full simulation: compilation → `init` → `main`](#5-full-simulation)
6. [The heap](#6-the-heap)
7. [Escape analysis: who decides stack vs. heap?](#7-escape-analysis)
8. [The garbage collector](#8-the-garbage-collector)
9. [Prove it on your machine](#9-prove-it-on-your-machine)
10. [Is local really faster than global?](#10-is-local-really-faster-than-global)
11. [Common misconceptions](#11-common-misconceptions)
12. [Interview questions](#12-interview-questions)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- The four memory regions of a running program: **code**, **data**, **stack**, **heap**
- Which of your variables and functions live in which region
- How **stack frames** are pushed and popped
- What **escape analysis** is, and how to see the compiler's decisions
- What the **garbage collector (GC)** does and why Go has one
- Which popular beliefs about memory are simplifications (or wrong)

---

## 2. The simplified model so far

Until now we drew memory like this:

```
┌────────────────┐
│ Global scope   │  ← package variables + functions
├────────────────┤
│ Function scopes│  ← locals, one box per call
└────────────────┘
```

That's a **great mental model for scope** (which is what we needed). But real memory has more structure. Consider this chapter "Class 2" of the same subject: the picture gets more detailed, but nothing you learned is thrown away.

> 📝 **Simplification vs. lie.** Teachers simplify to help you learn. The simplified diagram wasn't wrong; it was *coarser*. Now we zoom in.

---

## 3. The four regions

When you run a program, the operating system gives it a private slice of memory (a *virtual address space*). Conceptually, the Go program divides it into **four regions**:

```
┌──────────────────────────────────────────────┐
│  Your program's memory (RAM)                 │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ 1. CODE SEGMENT   (a.k.a. "text")      │  │  machine code of ALL functions
│  ├────────────────────────────────────────┤  │  (read-only)
│  │ 2. DATA SEGMENT   (data + bss)         │  │  package-level variables, constants
│  ├────────────────────────────────────────┤  │
│  │ 3. STACK          ↓ frames             │  │  function calls: params + locals
│  ├────────────────────────────────────────┤  │
│  │ 4. HEAP           👹 GC                │  │  dynamically allocated data
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

### 3.1 Code segment (a.k.a. text segment)

- Holds the **machine code** of every function: `main`, `add`, standard-library functions, all of it.
- **Read-only**: the program can't accidentally overwrite its own instructions (a security and stability feature).
- Written once, when the program is loaded. Each function's code exists **once**, however many times it's called.
- The "function name" in package scope is essentially a label for an address in this segment.

### 3.2 Data segment (a.k.a. global memory)

- Holds **package-level variables**: `var counter = 100`.
- Constants are mostly baked into the code or read-only data.
- Exists for the **entire life** of the program: allocated at start, freed when the process exits.
- Technically split in two: **data** (variables with a non-zero initial value like `100`) and **bss** (variables that start at zero). You don't need to care about the difference.

### 3.3 Stack

- Holds the **stack frames** of function calls: parameters, local variables, and return information.
- Works in **LIFO** order: last in, first out.
- Very fast: allocating a frame is just moving one pointer (the *stack pointer*, Chapter 29).
- Automatically cleaned: when a function returns, its frame is popped (released).

### 3.4 Heap

- Holds data whose **lifetime is not tied to a single function call**, or whose size isn't known at compile time.
- Allocated at run time; freed by the **garbage collector**, not by function return.
- Slower than the stack (allocation needs bookkeeping; cleanup needs GC work).

### Summary table

| Region | Contains | Lifetime | Managed by | Speed |
|--------|----------|----------|------------|-------|
| **Code** | Machine code of functions | Whole program | Loader (read-only) | n/a |
| **Data** | Package-level variables | Whole program | Loader | Fast |
| **Stack** | Frames: params + locals | One function call | Automatic (push/pop) | Very fast |
| **Heap** | Long-lived / dynamic data | Until unreachable | **Garbage collector** | Slower |

> 🔎 **Real-world caveat:** These are *conceptual* regions and match how C programs are laid out. In Go, the runtime is more elaborate: each **goroutine** has its **own small, growable stack** (Chapters 35–36), and the heap is organized into size classes. The four-region model still explains everything you'll observe day to day.

---

## 4. Stack frames in detail

A **stack frame** (also called an activation record) is the chunk of stack memory reserved for **one call** of one function. It contains:

```
┌──────────────────────────────┐
│ parameters (copied arguments)│
│ local variables              │
│ return address               │  ← where to resume in the caller
│ (saved registers, etc.)      │
└──────────────────────────────┘
```

Rules:
1. Calling a function **pushes** a new frame on top.
2. The **top** frame is the one currently executing.
3. Returning **pops** the top frame; its memory is instantly reusable.
4. A frame only exists while its function is running (or waiting for a function it called).

```
main() calls add(5, 4):

before                after call            after return
┌─────────┐          ┌───────────┐          ┌─────────┐
│  main   │          │ add       │ ◄─ top   │  main   │ ◄─ top
└─────────┘          ├───────────┤          └─────────┘
                     │  main     │
                     └───────────┘
```

Because frames are strictly nested and sequential, the stack needs **no search and no bookkeeping**: allocation is "move the top pointer". That is the source of its speed.

---

## 5. Full simulation

```go
package main

import "fmt"

const a = 10 // constant
var p = 100  // package variable → data segment

func init() {
	fmt.Println("Hello")
}

func call() {
	add := func(x, y int) { // function expression
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

### Phase 1: Compile time / program load

The compiler translates every function to machine code; the loader places it in the **code segment**. Package variables are placed in the **data segment**:

```
CODE SEGMENT (read-only)          DATA SEGMENT
┌───────────────────────┐         ┌───────────────┐
│ init      → machine   │         │ p = 100       │
│ call      → machine   │         └───────────────┘
│ call.func1 (the       │           (const a = 10 is folded into code)
│   function expression)│
│ main      → machine   │
└───────────────────────┘
        STACK: (empty)           HEAP: (empty)
```

Notice that even the **anonymous function** `func(x, y int) {...}` inside `call` is compiled at this stage (the compiler names it something like `main.call.func1`). *Defining* a function expression costs no run-time work; the code already exists.

### Phase 2: run time

**Step 1: `init` runs (Chapter 14).** Frame pushed, prints `Hello`, popped.

**Step 2: `main` starts.**

```
STACK
┌─────────────┐
│ main        │
└─────────────┘
```

**Step 3: `main` calls `call()`.**

```
STACK
┌─────────────────────────────────┐
│ call                            │
│   add = ► (address of func1)    │  ← after the `add := func...` line executes,
├─────────────────────────────────┤     `add` holds a REFERENCE into the code segment
│ main                            │
└─────────────────────────────────┘
```

**Step 4: `add(5, 6)`: a new frame.**

```
STACK
┌─────────────────────────────────┐
│ func1   x=5  y=6  z=11          │  prints 11
├─────────────────────────────────┤
│ call    add=►func1              │
├─────────────────────────────────┤
│ main                            │
└─────────────────────────────────┘
```

**Step 5: return → `func1`'s frame popped.**

**Step 6: `add(p, a)`:** `p` isn't in `call`'s frame, so Go finds it in the **data segment** (100); `a` is the constant 10. New frame: `x=100, y=10, z=110` → prints `110` → popped.

**Step 7: `call` returns → its frame (and the `add` variable) popped.**

**Step 8: `main` prints `10`, returns → program ends → all memory returned to the OS.**

Output:

```
Hello
11
110
10
```

**Insights:**
- Function **code** lives in the code segment; **variables** that refer to functions live in frames.
- A frame exists only while its function is active.
- The data segment persists for the whole run.

---

## 6. The heap

Stack frames are perfect when data dies with the function. But sometimes data must **outlive** the function that created it:

```go
package main

import "fmt"

type point struct{ x, y int }

func newPoint() *point {
	p := point{1, 2}
	return &p // return the ADDRESS of p
}

func main() {
	pt := newPoint()
	fmt.Println(pt.x, pt.y) // 1 2
}
```

If `p` lived in `newPoint`'s frame, then after `newPoint` returned, the frame would be popped and `pt` would point to **dead memory**, a "dangling pointer" (the source of countless C bugs).

Go solves this automatically: the compiler notices `p`'s address escapes the function, so it allocates `p` on the **heap** instead. The heap outlives any single call.

```
newPoint returned:

STACK                    HEAP
┌───────────┐            ┌──────────────┐
│ main      │            │ point{1, 2}  │ ◄─┐
│  pt ──────┼────────────┼──────────────┘   │
└───────────┘            └──────────────────┘
(newPoint's frame is gone; the point lives on)
```

Things that typically end up on the heap:
- Values whose address is returned or stored somewhere that outlives the call
- Values captured by **closures** that outlive their function (Chapter 20)
- Values sent through channels/interfaces in some cases
- Very large values, or data of unknown size at compile time (e.g., a slice grown with `append`, contents of a `map`)

**You don't choose**; the compiler does (next section).

---

## 7. Escape analysis

> **Escape analysis** is a compiler step that decides, for each variable, whether it can safely live on the stack or "escapes" to the heap.

The rule of thumb: *if anything could still refer to the variable after the function returns, it escapes.*

You can ask the compiler what it decided:

```bash
go build -gcflags='-m' .
```

For the program above, the compiler prints (real output from Go 1.25):

```
./main.go:15:2: moved to heap: p
```

`moved to heap: p` is Go telling you `p` escapes.

### Example: stays on the stack

```go
package main

import "fmt"

func square(n int) int {
	result := n * n // no one outside can reach this → stack
	return result   // returns a COPY of the value
}

func main() { fmt.Println(square(6)) }
```

`result`'s **value** is copied to the caller. Nothing refers to the variable itself afterward. It stays on the stack (or even a CPU register).

### Why does Go hide this from you?

In C you must write `malloc`/`free` yourself. In Go there's no such choice: `&x` is always safe. The compiler picks the right place, and the GC cleans the heap. You get safety **and** most of the speed.

### Should you optimize for it?

Not at first. Write clear code, then **measure** (benchmarks, profiling). Heap allocation is a cost, but a small one; premature "avoid the heap" tricks make code worse. Just be aware of it, and know how to look (`-gcflags=-m`).

---

## 8. The garbage collector

**GC = Garbage Collector**: a part of the Go runtime that automatically frees heap memory that is no longer used.

### What is "garbage"?

Heap data that **no variable can reach any more**:

```go
package main

import "fmt"

type big struct{ data [1000]int }

func main() {
	x := &big{} // allocated on the heap (pointer kept in x)
	fmt.Println(len(x.data))

	x = nil // nothing points to the big struct now → it is garbage
	// The GC will reclaim it at some later point. You never call "free".
}
```

### How the GC finds garbage (simplified)

Go uses a **concurrent, tri-color mark-and-sweep** collector:

1. **Mark**: starting from *roots* (package variables, every goroutine's stack), follow every pointer and mark what's *reachable*.
2. **Sweep**: everything unmarked is unreachable → free it for reuse.

```
Roots ──► A ──► B          C  ◄── D (D itself unreachable)
          │               (C unreachable → garbage)
          └──► E
Marked (live): A, B, E    Swept (freed): C, D
```

"Concurrent" means most GC work runs **alongside** your program on other threads, keeping pauses short (typically well under a millisecond).

### Why have a GC?

| Without GC (C/C++) | With GC (Go) |
|--------------------|--------------|
| You must `free` every allocation | Automatic |
| Forget → **memory leak** | Leaks mostly impossible (logical ones remain) |
| Free too early → **use-after-free** crash | Impossible: memory lives while reachable |
| Free twice → **double free** corruption | Impossible |
| Maximum performance control | Small CPU cost + some latency |

### Can I leak memory in Go?

Yes, *logically*: keep references you no longer need (a global slice that only grows, an unstopped goroutine, a map that's never pruned). The GC only frees what's *unreachable*, not what you *forgot about*.

### The stack needs no GC

Frames are popped automatically; only the heap needs collecting. That's another reason the compiler prefers the stack.

### Observing the GC

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	var m runtime.MemStats

	runtime.ReadMemStats(&m)
	before := m.HeapAlloc

	data := make([][]byte, 0)
	for i := 0; i < 100; i++ {
		data = append(data, make([]byte, 1<<20)) // 100 × 1 MiB
	}
	runtime.ReadMemStats(&m)
	fmt.Println("after allocating:", (m.HeapAlloc-before)/(1<<20), "MiB more heap")

	data = nil    // drop all references
	runtime.GC()  // force a collection (for demonstration only)
	runtime.ReadMemStats(&m)
	fmt.Println("after GC:        ", (int64(m.HeapAlloc)-int64(before))/(1<<20), "MiB more heap")
	fmt.Println("GC cycles so far:", m.NumGC)
}
```

Typical output: roughly `100 MiB` more heap before, and about `0 MiB` after (numbers vary slightly per run). Don't call `runtime.GC()` in real programs; the runtime knows when to run.

---

## 9. Prove it on your machine

Don't just trust diagrams. Build the demo below and inspect the compiled program.

```go
package main

import "fmt"

var counter = 100 // package-level variable

type point struct{ x, y int }

func onStack() int {
	n := 42
	return n
}

func newPoint() *point {
	p := point{1, 2}
	return &p // escapes
}

func main() {
	a := onStack()
	p := newPoint()
	fmt.Println(a, p.x, counter)
}
```

**1. Escape analysis**

```bash
go build -gcflags='-m' .
```

Output includes `moved to heap: p`: the variable in `newPoint` escaped.

**2. Segment sizes of the binary**

```bash
go build -o memdemo .
size memdemo          # Linux; shows text / data / bss
```

Real output from one build:

```
   text    data     bss     dec     hex filename
1445143   42348  217064 1704555  1a026b memdemo
```

- **text** (~1.4 MB): the code segment: your functions *plus the Go runtime and `fmt`*.
- **data** and **bss**: package-level variables of your program and of the standard library.

**3. Where symbols live**

```bash
go tool nm -size memdemo | grep -E ' main\.(counter|main|onStack|newPoint)$'
```

Real output:

```
  498220        183 T main.main
  563368          8 D main.counter
```

- `T` = **text** (code segment): `main.main` is a function.
- `D` = **data**: `main.counter` is a package variable.

(`onStack` and `newPoint` don't appear; the compiler *inlined* them into `main`, which the `-m` output also mentions: `can inline onStack`.) 

You just *saw* the code and data segments of a real program.

---

## 10. Is local really faster than global?

You will often hear: *"Local variables are faster than global ones because global ones require a search in another region."*

The truth is more nuanced, and understanding it will make you a better engineer (and a sharper interview candidate):

| Claim | Reality |
|-------|---------|
| Scope *lookup* is a run-time search from frame to global | **No.** Name lookup happens at **compile time** (Chapter 11). At run time the machine code contains direct addresses or offsets. |
| Globals live in a different region | ✅ True: data segment vs. stack |
| Locals are "a bit faster" | Often ✅, but for other reasons: locals frequently live in **CPU registers**, are **cache-friendly** (the top of the stack is hot in cache), and the compiler can optimize them aggressively because no one else can touch them |
| Globals are much slower | ❌ Usually a negligible difference for a single access; a global that's accessed repeatedly is cached just the same |
| **Bigger** reason to avoid globals | **Design**, not speed: hidden dependencies, hard testing, data races between goroutines (Chapter 67) |

So the model "check the frame first, then go to the data segment" is a good picture of **scoping rules**, but *not* of run-time speed. Use locals because they make code clearer and safer; performance is a bonus.

---

## 11. Common misconceptions

| Misconception | Truth |
|---------------|-------|
| "Global memory is a separate box outside the program's memory" | It's the **data segment**, part of the same process |
| "Functions execute *inside* the code segment" | Code is *stored* there (read-only); **execution state** (variables) lives on the **stack** |
| "The stack is permanent" | Frames come and go with calls |
| "All segments have a fixed size" | Stack and heap **grow** as needed (Go stacks start tiny, ~2–8 KB, and grow) |
| "`new`/`&` means heap; plain `x := ...` means stack" | The compiler decides using **escape analysis**, not syntax |
| "Go has no memory leaks" | Reachable-but-unneeded memory is still a leak |
| "The GC makes Go slow" | GC has a cost, but Go's is concurrent and tuned for low latency; most programs never notice |
| "Every variable takes exactly the same space" | Types have sizes: `int8` = 1 byte, `int64` = 8, `string` = 16 (pointer + length), … |

---

## 12. Interview questions

**Q1. What are the memory segments of a Go program?**
Code (text), data (globals), stack, and heap.

**Q2. What's in the code segment? Can it be modified?**
The compiled machine code of all functions; read-only.

**Q3. Where do package-level variables live?**
In the data segment, for the program's lifetime.

**Q4. What is a stack frame?**
The memory for one function call: parameters, locals, return address; pushed on call and popped on return.

**Q5. What is escape analysis?**
A compile-time analysis deciding whether a variable can live on the stack or must move to the heap because it outlives its function.

**Q6. What does the GC do?**
Finds unreachable heap objects (mark and sweep) and frees them, concurrently with the program.

**Q7. Why is stack allocation cheaper than heap allocation?**
Push/pop of a single pointer, no search, no GC involvement, cache-friendly.

**Q8. Does `p := &x` always allocate on the heap?**
No; only if `p` (or the pointer) escapes.

---

## 13. Exercises

### Exercise 1: Which region?
For each item, name the region: (a) the machine code of `add`, (b) `var total = 0` at package level, (c) parameter `n` of a running function, (d) a `point` whose address is returned from a function.

<details><summary>Solution</summary>

(a) Code segment. (b) Data segment. (c) Stack (the function's frame). (d) Heap (it escapes).
</details>

### Exercise 2: Draw the stack
Draw the stack at the deepest moment and give the output:

```go
package main

import "fmt"

func c(n int) { fmt.Println("c", n) }
func b(n int) { c(n + 1) }
func a(n int) { b(n + 1) }

func main() { a(1) }
```

<details><summary>Solution</summary>

Deepest: `c(n=3)` on `b(n=2)` on `a(n=1)` on `main`. Output: `c 3`. Frames pop in reverse order.
</details>

### Exercise 3: Stack or heap?
Which variable escapes?

```go
func f() int {
	x := 5
	return x
}

func g() *int {
	y := 5
	return &y
}
```

<details><summary>Solution</summary>

`y` in `g` escapes (its address is returned). `x` in `f` doesn't (a *copy* of its value is returned). Verify with `go build -gcflags=-m`: `moved to heap: y`.
</details>

### Exercise 4: Frame lifetime
What's wrong with this claim: "After `add` returns, its variable `sum` still exists in memory so I can use it in `main`."

<details><summary>Solution</summary>

`sum` was in `add`'s frame, which is popped on return. The memory may be reused. Also, by scope rules `main` can't name `sum`. To use the value, **return it** (a copy is passed to the caller).
</details>

### Exercise 5: Prove it yourself
Build any small program and run `size`, `go tool nm`, and `go build -gcflags=-m`. Find one function marked `T`, one package variable marked `D` or `B`, and one "moved to heap" message (write code that causes one).

<details><summary>Hint</summary>

Return the address of a local (`return &x`), or store a local's address in a package-level variable, and rebuild with `-gcflags=-m`.
</details>

### Exercise 6 (challenge): Leak or not?
The map below grows forever in a long-running server. Is this a memory leak even with a GC? How would you fix it?

```go
var cache = map[string][]byte{}

func handle(id string, payload []byte) {
	cache[id] = payload // never removed
}
```

<details><summary>Solution</summary>

Yes, a *logical* leak. The map (a package variable, so a GC root) keeps every payload reachable forever, so the GC can't free them. Fix: bound the cache (evict entries by size/age, e.g., an LRU or TTL), or delete entries when no longer needed (`delete(cache, id)`).
</details>

---

## 14. Quiz

1. Name the four memory regions.
2. Which region is read-only?
3. What happens to a function's frame when it returns?
4. Who frees heap memory in Go?
5. What does `moved to heap: x` mean?
6. Is name lookup for variables a run-time search through scopes?

<details><summary>Answers</summary>

1. Code, data, stack, heap.
2. The code segment.
3. It's popped; the memory is reusable.
4. The garbage collector.
5. The compiler determined `x` outlives its function, so it's allocated on the heap.
6. No. Names are resolved at compile time; scope is a *compile-time* concept.
</details>

---

## 15. Summary

- A running program's memory has four conceptual regions: **code** (machine code, read-only), **data** (package variables), **stack** (call frames), **heap** (long-lived data).
- Each function call pushes a **stack frame** (params, locals, return address) and pops it on return: fast and automatic.
- Data that must outlive its function goes on the **heap**; the compiler chooses via **escape analysis** (`go build -gcflags=-m` shows decisions).
- The **garbage collector** frees unreachable heap memory concurrently; you never call `free`, but you can still leak by holding references.
- Function *code* lives in the code segment; a **variable holding a function** lives in a frame and *refers* to that code (next chapter).
- "Locals are faster than globals" is at best a half-truth; the real reasons to prefer locals are **clarity and safety**.

### ➡️ What's next?

[Chapter 19](19-end-of-internal-memory.md) finishes the memory story: **compilation vs. execution**, what a binary file really is, and exactly **where a function expression lives**, which sets up closures.
