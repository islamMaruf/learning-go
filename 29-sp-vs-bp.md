# Chapter 29: SP vs. BP — The Stack Pointer Dance

> **Goal of this chapter:** Watch, register by register, how the CPU builds and tears down **stack frames** when one function calls another. You'll learn what the **Stack Pointer (SP)** and **Base Pointer (BP)** do, what's inside a frame, why the stack grows **downward**, and how to see all of it in real machine code and real addresses.

**Difficulty:** 🔴 Advanced (conceptual)  **Estimated time:** 2–3 hours  **Prerequisite:** [Chapter 28](28-breaking-the-cpu-and-understanding-the-process.md) (registers) and [Chapter 4](04-introduction-to-functions.md)/[18](18-go-internal-memory.md) (stack frames)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [First, a correction about memory addresses](#2-a-correction-about-memory-addresses)
3. [What's inside a stack frame](#3-whats-inside-a-stack-frame)
4. [Meet the program](#4-meet-the-program)
5. [The instructions that build and destroy frames](#5-the-instructions-that-build-and-destroy-frames)
6. [The dance, step by step](#6-the-dance-step-by-step)
7. [Why save the old BP?](#7-why-save-the-old-bp)
8. [Why does the stack grow downward?](#8-why-does-the-stack-grow-downward)
9. [See it in real Go](#9-see-it-in-real-go)
10. [Where Go differs from the classic textbook model](#10-where-go-differs-from-the-textbook-model)
11. [Stack overflow](#11-stack-overflow)
12. [Common misconceptions](#12-common-misconceptions)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- The precise roles of **SP** and **BP**, and why a CPU wants both
- The layout of a **stack frame**: arguments, return address, saved BP, locals
- The exact sequence of **CALL / PUSH / MOV / SUB / ADD / POP / RET** that runs on every function call
- Why the stack grows toward **lower addresses**
- How to prove all of this with a few lines of Go
- What a **stack overflow** is

> 📝 The examples use small made-up addresses (like 1000, 992) so the arithmetic is easy to follow. Real addresses look like `0xc000090e10`, and section 9 shows some.

---

## 2. A correction about memory addresses

Earlier material (including a previous version of this chapter) simplified memory as "each cell is one *word*: 1 byte on an 8-bit machine, 2 bytes on 16-bit, 4 bytes on 32-bit, 8 bytes on 64-bit, so addresses jump by 2, 4, or 8." Some **historical** computers really were built that way (they're called *word-addressable*).

**Every mainstream CPU today (x86, ARM, RISC-V) is byte-addressable**, regardless of its "bitness":

> **Every single byte has its own address, and addresses go up by 1 per byte.**

A 64-bit CPU still moves 8 bytes at a time in a register; but the *address* of the next byte is simply +1. What "64-bit" affects is the **size of a register / address** (8 bytes), not the spacing between addresses.

What *is* true, and probably the origin of the idea, is **alignment**: a 4-byte `int32` is normally placed at an address divisible by 4; an 8-byte `int64` at an address divisible by 8. So *8-byte values* sit 8 addresses apart, as in the real output below.

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	var nums [4]int64
	for i := range nums {
		fmt.Printf("&nums[%d] = %#x\n", i, uintptr(unsafe.Pointer(&nums[i])))
	}
	var small [3]byte
	for i := range small {
		fmt.Printf("&small[%d] = %#x\n", i, uintptr(unsafe.Pointer(&small[i])))
	}
}
```

Real output (addresses vary per run):

```
&nums[0] = 0xc000090e90
&nums[1] = 0xc000090e98     ← +8 : each int64 is 8 bytes
&nums[2] = 0xc000090ea0
&nums[3] = 0xc000090ea8
&small[0] = 0xc000090e65
&small[1] = 0xc000090e66    ← +1 : each byte is 1 byte
&small[2] = 0xc000090e67
```

So: addresses always count **bytes**; how far apart two *values* are depends on the **size of the type**. The whole rest of this chapter uses byte addresses, and we'll use **8-byte slots** for everything (arguments, return address, saved BP) because that's what a 64-bit CPU pushes.

---

## 3. What's inside a stack frame

Recall: each function call gets a **stack frame** (Chapters 4 and 18). Now we can say exactly what's in one. The classic layout (used by C and most textbooks, and a good model for everything) looks like this, with **higher addresses at the top of the page**:

```
   higher addresses
   ┌─────────────────────────┐
   │  arguments (if passed   │   ← pushed by the CALLER
   │  on the stack)          │
   ├─────────────────────────┤
   │  return address         │   ← pushed by the CALL instruction
   ├─────────────────────────┤
   │  saved (old) BP         │   ← pushed by the callee's prologue
   ├─────────────────────────┤ ◄── BP (points here for the whole call)
   │  local variables        │
   │  (and temporaries)      │
   ├─────────────────────────┤ ◄── SP (moves as the function pushes/pops)
   │  (free stack space)     │
   └─────────────────────────┘
   lower addresses
```

| Item | Who creates it | Purpose |
|------|----------------|---------|
| Arguments | Caller | Inputs for the callee (on modern CPUs many go in registers instead, see section 10) |
| **Return address** | The `CALL` instruction | The address of the instruction *after* the call. `RET` pops it into the PC, so execution resumes in the caller |
| **Saved BP** | Callee's first instructions | The **caller's** base pointer, so it can be restored when we return |
| Locals | Callee (by moving SP) | The function's own variables |

### SP vs. BP in one sentence each

- **SP (stack pointer): "where is the top of the stack right now?"** It **moves** every time something is pushed or popped.
- **BP (base pointer / frame pointer): "where does *this function's* frame start?"** It's **fixed for the duration of the call**, so the function can address its variables at *constant offsets* (`BP − 8`, `BP + 16`, ...) even while SP wobbles.

> **Analogy: a moving shelf.** You're organizing a shelf. SP is your finger at the *current top item*, and it moves up and down as you add and remove items. BP is a **sticker on the shelf's base** that never moves; you say "the third item above the sticker" and can always find it.

---

## 4. Meet the program

We'll trace this program:

```go
package main

import "fmt"

func add(a int, b int) int {
	sum := a + b
	return sum
}

func main() {
	x := 10
	y := 12
	result := add(x, y)
	fmt.Println(result)
}
```

Simplifying assumptions for the drawings:

- 64-bit machine: every slot (argument, address, saved BP, local `int`) is **8 bytes**.
- The stack **grows toward lower addresses**.
- We use the **classic model**: arguments are pushed on the stack. (Go really passes them in registers, section 10, but the frame logic is identical.)
- We'll pretend that before `main` starts, `SP = 1000`.

---

## 5. The instructions that build and destroy frames

Four ideas cover everything.

**`PUSH value`**: put `value` on the stack:
```
SP = SP − 8       (stack grows DOWN, so subtract)
[SP] = value      (write into the new top slot)
```

**`POP reg`**: take the top off:
```
reg = [SP]
SP = SP + 8
```

**`CALL target`**: call a function:
```
PUSH (address of the next instruction)   ← the return address
PC = target
```

**`RET`**: return:
```
POP → PC          ← jump back to the saved return address
```

And every function starts and ends with a standard **prologue/epilogue**:

```
PROLOGUE (start of every function)
    PUSH BP          ; save the caller's BP
    MOV  BP, SP      ; BP = SP: this frame's base is here
    SUB  SP, n       ; reserve n bytes for locals (SP moves down)

EPILOGUE (end of every function)
    ADD  SP, n       ; (or MOV SP, BP) release the locals
    POP  BP          ; restore the caller's BP
    RET              ; pop the return address into PC
```

---

## 6. The dance, step by step

Follow `SP` and `BP` through the whole program. Addresses go **down** as the stack grows.

### Step 0: `main`'s frame is set up

`main` starts. Its prologue runs. Say the previous BP (belonging to Go's runtime, which called `main`) was `2000`.

```
PUSH BP        SP = 992,  [992] = 2000  (saved runtime BP)
MOV BP, SP     BP = 992
SUB SP, 24     SP = 968   (room for x, y, result: 3 × 8 bytes)
```

```
 addr
 1000 ┌──────────────────────┐  ← (return address into runtime lives above; not shown)
  992 │ saved BP = 2000      │ ◄── BP = 992   (main's frame base)
  984 │ x                    │
  976 │ y                    │
  968 │ result               │ ◄── SP = 968
      └──────────────────────┘
```

Then `x := 10`, `y := 12` are stored at `BP−8` (984) and `BP−16` (976).

```
  984 │ x = 10               │
  976 │ y = 12               │
```

### Step 1: `main` calls `add(x, y)`: the caller pushes arguments

Classic convention: push the arguments (right-to-left):

```
PUSH y         SP = 960,  [960] = 12
PUSH x         SP = 952,  [952] = 10
CALL add       (CALL pushes the return address)
```

`CALL` pushes the **return address** (say `0x4A00`, the instruction after the call in `main`) and sets **PC = the address of `add`**:

```
                   SP = 944, [944] = 0x4A00
```

```
 addr
  992 │ saved BP = 2000      │ ◄── BP (still main's)
  984 │ x = 10               │
  976 │ y = 12               │
  968 │ result (unset)       │
  960 │ arg: b = 12          │  ← pushed by main
  952 │ arg: a = 10          │
  944 │ return address 0x4A00│ ◄── SP = 944
```

### Step 2: `add`'s prologue: build its frame

```
PUSH BP        SP = 936, [936] = 992    (save MAIN's BP: 992)
MOV BP, SP     BP = 936                 (now BP marks ADD's frame base)
SUB SP, 8      SP = 928                 (room for the local `sum`)
```

```
 addr
  992 │ saved BP = 2000      │  main's frame
  984 │ x = 10               │
  976 │ y = 12               │
  968 │ result               │
  ────────────────────────────
  960 │ b = 12               │
  952 │ a = 10               │   } arguments (belong to add's call)
  944 │ return address 0x4A00│
  936 │ saved BP = 992       │ ◄── BP = 936   (add's frame base)
  928 │ sum                  │ ◄── SP = 928
```

Notice **two BPs are now on the stack**: `2000` (saved by `main`) and `992` (saved by `add`). Each frame remembers its caller's BP, forming a **linked chain of frames** (debuggers and Go's panic tracebacks walk this chain).

### Step 3: `add` runs: BP-relative addressing

Inside `add` the CPU finds everything relative to the **fixed BP = 936**:

| Variable | Address | How it's found |
|----------|---------|----------------|
| `a` | 952 | `BP + 16` |
| `b` | 960 | `BP + 24` |
| `sum` | 928 | `BP − 8` |

`sum := a + b` → load `[BP+16]` (10), load `[BP+24]` (12), ALU adds → 22, store into `[BP−8]`. The result to return is placed in a register (say `AX = 22`).

If `add` pushed and popped temporaries, **SP would move, but BP wouldn't**, and the offsets to `a`, `b`, `sum` would remain valid. That's why BP exists.

### Step 4: `add`'s epilogue: tear the frame down

```
ADD SP, 8      SP = 936     (release the local sum: SP goes back UP)
POP BP         BP = [936] = 992,  SP = 944   (restore MAIN's BP)
RET            PC = [944] = 0x4A00, SP = 952 (jump back into main)
```

After `RET`:

```
 addr
  992 │ saved BP = 2000      │ ◄── BP = 992  (main's again!)
  984 │ x = 10               │
  976 │ y = 12               │
  968 │ result               │
  960 │ b = 12  (stale)      │
  952 │ a = 10  (stale)      │ ◄── SP = 952
  944 │ ...garbage; add's frame is "free" now
```

The frame of `add` is **gone**, not by erasing it, but simply by moving SP back up. The old bytes are still physically there but are considered free and will be overwritten by the next call. (This is why "uninitialized locals" in C show garbage, and why Go zeroes locals for you.)

### Step 5: `main` cleans up the arguments and stores the result

```
ADD SP, 16     SP = 968   (caller removes its two pushed arguments)
MOV [BP−24], AX           result = 22  (the return value came back in AX)
```

`result` now holds `22`. `main` prints it.

### Step 6: `main` returns

`main`'s epilogue: `ADD SP, 24` → `SP = 992`; `POP BP` → `BP = 2000` (the runtime's); `RET` → back to the runtime. The stack is back to how we found it.

### The whole dance in one table

| Moment | SP | BP | Stack depth |
|--------|----|----|-------------|
| Before `main` | 1000 | 2000 | runtime only |
| `main` prologue done | 968 | 992 | main |
| Args pushed + CALL | 944 | 992 | main + args + ret addr |
| `add` prologue done | 928 | 936 | main + add |
| `add` epilogue done, after RET | 952 | 992 | main (+ stale args) |
| Args removed | 968 | 992 | main |
| `main` returns | 1000 | 2000 | runtime only |

**SP swings down and up; BP jumps to a new frame base on each call and is restored on each return.** That's the "dance".

---

## 7. Why save the old BP?

Because **BP is a single register**, and there are many frames. When `add` sets `BP = 936` it overwrites `main`'s BP (992). If `add` didn't save it first, then after `add` returns, `main` would be lost: it couldn't find `x`, `y`, `result`.

```
without saving:           with PUSH BP / POP BP:

main: BP = 992            main: BP = 992
call add: BP = 936        call add: [saved 992]; BP = 936
return:  BP = 936 ✗       return:  BP = 992 ✓ (restored)
main can't find its       main resumes normally
variables!
```

The saved BPs also form a **chain**: from the current BP you can read the previous BP, then *its* previous one, and so on, up to the beginning. That chain is how debuggers show **stack traces** and how Go prints one when a program panics.

---

## 8. Why does the stack grow downward?

Convention, going back to early computer design:

1. **Two growing regions, one free space.** A process needs room for both the **heap** (grows as you allocate) and the **stack**. In the classic layout the heap starts low and grows **up**; the stack starts high and grows **down**. They grow **toward each other** through one big free gap, so neither needs a fixed limit (until memory runs out).

```
 high addresses  ┌────────────────┐
                 │ STACK    ↓     │
                 │                │
                 │  (free space)  │
                 │                │
                 │ HEAP     ↑     │
                 ├────────────────┤
                 │ DATA           │
                 │ CODE           │
 low addresses   └────────────────┘
```

2. **Hardware convention.** x86 and many other CPUs' `PUSH` instructions decrement SP. It stuck.

**Prove it in Go:** the addresses of locals in nested calls decrease as calls get deeper. Real output:

```go
// a() calls b(), which calls c(). Each returns the address of its own local.
// (uses unsafe purely to print addresses; and //go:noinline to keep real calls)
```

```
a: 0xc000090e10      ← outermost call: highest address
b: 0xc000090df8      ← 0x18 (24) bytes lower
c: 0xc000090de0      ← deeper still: lower again
```

Deeper calls → **lower addresses**: the stack grows downward (`a > b > c`). Also this shows each frame here is about 24 bytes, tiny.

---

## 9. See it in real Go

Disassemble the unoptimized `add` (Chapter 28's output) and match it to our prologue/epilogue:

```bash
go build -gcflags='-N -l' -o sp .
go tool objdump -s 'main.add$' sp
```

```
PUSHQ BP            ; save caller's BP                 ← prologue
MOVQ SP, BP         ; BP = SP                          ← prologue
SUBQ $0x8, SP       ; reserve local space              ← prologue
  ... body: ADDQ BX, AX ...                              ← a + b
ADDQ $0x8, SP       ; release locals                   ← epilogue
POPQ BP             ; restore caller's BP              ← epilogue
RET                 ; pop return address into PC       ← epilogue
```

And in `main.main` you'll find the matching call:

```
PUSHQ BP
MOVQ SP, BP
SUBQ $0x1e0, SP           ; main's frame is 0x1e0 = 480 bytes
...
CALL main.add(SB)         ; push return address, PC = add
```

These are exactly the instructions from section 5, generated by Go's compiler. The dance is not a metaphor; it's literally what happens.

---

## 10. Where Go differs from the textbook model

The mental model above is correct in spirit. Go's real implementation has some differences worth knowing:

| Textbook (C, cdecl) | Go (amd64, recent versions) |
|--------------------|------------------------------|
| Arguments pushed on the stack | Up to 9 integer args and 15 float args passed in **registers** (`AX, BX, CX, DI, SI, R8, R9, R10, R11`); the stack is used for overflow and for spilling |
| Return value in one register | **Multiple** return values in registers (Chapter 5's `(int, error)` needs no allocation) |
| Caller cleans up args | Callee's frame includes space for what it needs; the frame size is one `SUBQ` |
| Frames have a fixed size | Frame size is known at compile time (as in C), **but** the whole goroutine stack can be **moved** and grown when it runs out (see below) |
| One big OS stack per thread (typically 8 MB) | Each **goroutine** has its own small stack, starting at about **2–8 KB**, that **grows and shrinks automatically** (Chapters 35–36) |

Because of that last point, Go checks at the start of most functions whether there's enough stack room (you saw `LEAQ ... CMPQ R12, 0x10(R14); JBE` at the top of `main.main` in the disassembly: "compare SP against the stack limit; if too small, jump to `morestack`"). If a goroutine's stack is full, the runtime **allocates a bigger one, copies the frames over, and fixes up pointers**, a trick that's only possible because Go's compiler knows exactly where every pointer in every frame is. This is what makes goroutines so cheap (Chapter 36).

Everything about SP, BP, frames, and return addresses still applies: it's just optimized.

---

## 11. Stack overflow

Every frame uses stack space; the stack is finite. **Unbounded recursion** never returns, so frames pile up until the limit is hit:

```go
package main

func recurse(n int) int {
	return recurse(n+1) + 1 // never stops
}

func main() {
	recurse(0)
}
```

Go's reaction (abbreviated output):

```
runtime: goroutine stack exceeds 1000000000-byte limit
fatal error: stack overflow
goroutine 1 [running]:
main.recurse(...)
	.../main.go:4 +0x...
...additional frames elided...
```

Go allows a goroutine stack to grow up to **1 GB** on 64-bit systems before giving up (C programs typically overflow at ~8 MB and crash with a segmentation fault). Note that this is a **fatal error**, not a `panic` you can `recover()` from.

**Prevention:** every recursive function needs a **base case** that stops it, and deep recursion (e.g., depth in the millions) should be rewritten as a loop.

---

## 12. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "On a 64-bit machine, memory addresses go up by 8" | Addresses count **bytes** (+1 each); *values* of 8-byte types sit 8 apart |
| "SP and BP are the same" | SP moves with every push/pop; BP stays fixed during a call |
| "Returning from a function erases its frame" | The frame is just *released* (SP moves back); old bytes may linger until overwritten |
| "Locals live at absolute addresses" | They're accessed at **offsets from BP** (or SP), so the same function works at any depth |
| "The return address is stored in a variable" | It's on the stack, pushed by `CALL` and consumed by `RET` |
| "Go always saves BP exactly like C" | It does keep frame pointers on amd64, but passes args in registers |
| "Stack overflow can be recovered" | In Go it's a fatal error, not a recoverable panic |
| "The stack grows up because indices grow up" | On x86 it grows **down** |

---

## 13. Exercises

### Exercise 1: Roles
Which register would you look at to find (a) the top of the stack, (b) local variable `sum` at a constant offset, (c) where to resume after `RET`?

<details><summary>Solution</summary>

(a) SP. (b) BP (`BP − 8`). (c) The return address on the stack, which `RET` pops into the PC.
</details>

### Exercise 2: Do the arithmetic
Start with `SP = 1000`. Execute: `PUSH A; PUSH B; PUSH C; POP X; POP Y`. What is SP after each instruction? Where do the values sit?

<details><summary>Solution</summary>

`PUSH A` → SP 992 ([992]=A). `PUSH B` → 984 ([984]=B). `PUSH C` → 976 ([976]=C). `POP X` → X = C, SP 984. `POP Y` → Y = B, SP 992. (Last in, first out.)
</details>

### Exercise 3: Find the variable
In `add`'s frame from section 6 (BP = 936): at what address are `a`, `b`, and `sum`? Give the BP-relative offset for each.

<details><summary>Solution</summary>

`a` at 952 = `BP+16`; `b` at 960 = `BP+24`; `sum` at 928 = `BP−8`. (The return address is at `BP+8` = 944, and the saved BP at `BP+0` = 936.)
</details>

### Exercise 4: Two levels
Suppose `main` calls `f`, and `f` calls `g`. How many saved BPs are on the stack while `g` runs (not counting the runtime's)? What are they?

<details><summary>Solution</summary>

Two, plus the runtime's: `f`'s prologue saved `main`'s BP; `g`'s prologue saved `f`'s BP. (And `main`'s prologue saved the runtime's.) So three saved BPs in total, forming a chain: g → f → main → runtime.
</details>

### Exercise 5: Why not just use SP?
Explain why compilers traditionally used a separate BP rather than addressing locals relative to SP only.

<details><summary>Solution</summary>

SP changes whenever the function pushes or pops (temporaries, arguments for nested calls), so the offset of each local from SP keeps changing, making code generation and debugging harder. BP is fixed for the call, so offsets are constant. (Optimizing compilers can track SP changes and omit BP, called *frame pointer omission*. But frame pointers make stack traces and profiling simple, which is why Go keeps them on amd64.)
</details>

### Exercise 6: Prove the direction
Write a program with three nested functions that each print the address of a local variable (use `%p` on `&x`, with `//go:noinline` so the calls really happen). Confirm that addresses decrease with depth.

<details><summary>Solution</summary>

```go
package main

import "fmt"

//go:noinline
func c() { x := 3; fmt.Printf("c: %p\n", &x) }

//go:noinline
func b() { x := 2; fmt.Printf("b: %p\n", &x); c() }

//go:noinline
func a() { x := 1; fmt.Printf("a: %p\n", &x); b() }

func main() { a() }
```

Note: taking the address and passing it to `fmt.Printf` makes `x` *escape* to the heap in this version (Chapter 18), so the addresses may look unordered. To see the stack itself, use `unsafe.Pointer` and `uintptr` without letting the pointer escape, as in the code behind section 8's real output.
</details>

### Exercise 7 (challenge): Predict the depth
`func f(n int) int { if n == 0 { return 0 }; return f(n-1) + 1 }`. During `f(3)`, at the deepest point, how many `f` frames exist, and how many return addresses are on the stack (counting the one back into `main`)?

<details><summary>Solution</summary>

`f(3)`, `f(2)`, `f(1)`, `f(0)` → **4** frames. Each call pushed a return address: 1 (into `main`) + 3 (each `f` returning into the previous `f`) = **4** return addresses.
</details>

---

## 14. Quiz

1. What does `CALL` push before jumping?
2. What does `RET` do?
3. Which register is fixed during a function's execution, SP or BP?
4. Why does each function save the caller's BP?
5. In which direction does the stack grow on x86-64?
6. What is the difference between a byte address and a word address? Which do modern CPUs use?
7. What happens on infinite recursion in Go?

<details><summary>Answers</summary>

1. The return address (the address of the next instruction).
2. Pops the return address into the PC, resuming the caller.
3. BP.
4. BP is one register; overwriting it would lose the caller's frame base. It's restored on return.
5. Toward lower addresses.
6. Byte-addressable memory gives every byte its own address (+1 per byte), and modern CPUs use it. Word-addressable machines address whole words.
7. The stack grows until the 1 GB limit, then the runtime reports `fatal error: stack overflow` (not recoverable).
</details>

---

## 15. Summary

- **SP** points to the **top of the stack** and moves on every push/pop. **BP** marks the **base of the current frame** and stays fixed, so locals and arguments live at constant offsets from it.
- A frame holds: (arguments), **return address** (`CALL`), **saved caller BP** (prologue), **locals**.
- **Prologue:** `PUSH BP; MOV BP, SP; SUB SP, n`. **Epilogue:** `ADD SP, n; POP BP; RET`. Creating/destroying a frame is just **moving SP**.
- Saved BPs form a **chain of frames**, the basis of stack traces.
- The stack grows **down**, toward the heap growing up.
- Memory is **byte-addressed**; alignment makes 8-byte values sit 8 bytes apart.
- Go uses the same ideas but passes arguments in **registers** and gives each goroutine a small **growable** stack. Infinite recursion ends in a fatal **stack overflow**.
- You can *see* all of this with `go tool objdump` and `unsafe`-printed addresses.

### ➡️ What's next?

We've seen how *one* process runs on the CPU. [Chapter 30](30-context-switching-the-pcb-and-the-magic-of-concurrency.md) answers the next question: how can the OS run *many* processes on limited CPUs, by saving and restoring exactly the registers we met here, in a **context switch** using the **Process Control Block (PCB)**.
