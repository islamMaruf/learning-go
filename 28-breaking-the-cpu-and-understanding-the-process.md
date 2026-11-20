# Chapter 28: Breaking the CPU and Understanding the Process

> **Goal of this chapter:** Open the CPU and look at its parts (**control unit**, **ALU**, **registers**), see how your Go code becomes instructions the CPU runs, and finally answer the question *"What is a process?"* You'll see real machine instructions and a real process's memory map from a program running on your own machine.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 2–2.5 hours  **Prerequisite:** [Chapters 18, 26, 27](26-computer-architecture-and-a-short-history-of-computing.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Inside the CPU](#2-inside-the-cpu)
3. [The Control Unit (CU)](#3-the-control-unit-cu)
4. [The Arithmetic Logic Unit (ALU)](#4-the-arithmetic-logic-unit-alu)
5. [Registers](#5-registers)
6. [Special-purpose registers: IR, PC, SP, BP](#6-special-purpose-registers)
7. [General-purpose registers and "64-bit"](#7-general-purpose-registers-and-64-bit)
8. [From your code to a running program](#8-from-your-code-to-a-running-program)
9. [The execution dance, step by step](#9-the-execution-dance)
10. [See real machine code](#10-see-real-machine-code)
11. [What is a process?](#11-what-is-a-process)
12. [See a real process's memory](#12-see-a-real-process)
13. [Many processes at once](#13-many-processes-at-once)
14. [The complete picture](#14-the-complete-picture)
15. [Common misconceptions](#15-common-misconceptions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- The three parts inside a CPU: **Control Unit**, **ALU**, **registers**
- What the **Instruction Register**, **Program Counter**, **Stack Pointer** and **Base Pointer** are, and what each does
- How source code becomes a binary, is loaded into memory, and is executed
- A **process** = a program in execution = *memory + CPU state*
- How to look at real machine instructions and a real process's memory layout

---

## 2. Inside the CPU

In Chapter 26 the CPU was a black box called "the brain". Let's open it. A (simplified) CPU core has three main parts:

```
┌───────────────────────── CPU core ─────────────────────────┐
│                                                            │
│   ┌────────────────────┐        ┌─────────────────────┐    │
│   │  CONTROL UNIT (CU) │        │  ALU                │    │
│   │  "the boss"        │        │  (Arithmetic Logic  │    │
│   │  fetch, decode,    │◄──────►│   Unit) "the worker"│    │
│   │  coordinate        │        │  + − × ÷ AND OR NOT │    │
│   └─────────▲──────────┘        └──────────▲──────────┘    │
│             │                              │               │
│   ┌─────────▼──────────────────────────────▼──────────┐    │
│   │              REGISTERS (tiny, super-fast storage)  │   │
│   │   IR   PC   SP   BP   + general-purpose registers  │   │
│   └────────────────────────────────────────────────────┘   │
│                                                            │
│   (cache memory sits close by: L1, L2)                     │
└────────────────────────────┬───────────────────────────────┘
                             │  bus
                        ┌────▼────┐
                        │   RAM   │
                        └─────────┘
```

> **Analogy: a small office.** The **Control Unit** is the manager who reads the to-do list and delegates. The **ALU** is the calculator-wielding worker who does the actual math. **Registers** are the sticky notes on the manager's desk: tiny, but instantly reachable. **RAM** is the filing room down the hall.

---

## 3. The Control Unit (CU)

The **Control Unit** directs the whole operation. It doesn't compute; it *orchestrates*:

1. **Fetches** the next instruction from RAM.
2. **Decodes** it: "what operation? on which data?"
3. **Signals** the right components (the ALU, memory, registers) to carry it out.
4. Moves on to the next instruction.

This is the **fetch-decode-execute cycle** from Chapter 26. The CU *is* the thing running that cycle.

---

## 4. The Arithmetic Logic Unit (ALU)

The **ALU** does the actual calculating:

| Arithmetic | Logic |
|-----------|-------|
| add, subtract | AND, OR, NOT, XOR |
| multiply, divide | shifts, comparisons (is A < B?) |

It receives two inputs from registers, does one operation, and puts the result back in a register. For `a + b` the CU tells the ALU "add these two registers", and the ALU produces the sum in binary.

Comparisons (`if x > y`) are also ALU work: the ALU subtracts and sets tiny **flag bits** ("result was zero", "result was negative"). The CU then uses those flags to decide whether to **jump** to a different instruction. That's how `if` and `for` are built from hardware.

---

## 5. Registers

**Registers** are storage locations **inside the CPU itself**: the fastest memory in the entire computer (accessible in a fraction of a nanosecond). There are only a few dozen, each holding one machine word (64 bits on a 64-bit CPU), and the CPU does its real work on *register contents*.

Why not just use RAM directly? Because RAM is ~100× slower (Chapter 26's hierarchy). So the CPU **loads values from RAM into registers**, computes, and **stores results back**:

```
      RAM                       CPU
  ┌─────────┐   load     ┌──────────────┐
  │ a = 10  │ ─────────► │ reg1 = 10    │
  │ b = 12  │ ─────────► │ reg2 = 12    │
  │         │            │  ALU: reg1 + reg2 → reg1 (22) │
  │ c = 22  │ ◄───────── │ reg1 = 22    │
  └─────────┘   store    └──────────────┘
```

Registers fall into two groups:
1. **Special-purpose**: have a fixed job (IR, PC, SP, BP, flags).
2. **General-purpose**: hold whatever the current instructions need.

---

## 6. Special-purpose registers

### 6.1 Instruction Register (IR)

Holds the **instruction currently being decoded/executed**, the "clipboard" or the problem on the teacher's board that everyone looks at right now.

```
┌────────────────────────┐
│ Instruction Register   │
│ 4801d8  (ADD ...)      │  ← the bytes of the current instruction
└────────────────────────┘
```

### 6.2 Program Counter (PC), a.k.a. Instruction Pointer

Holds the **address of the next instruction to fetch**. It's a **bookmark** in your program's machine code.

```
Code in RAM:
 addr:  0x100     0x104     0x108     0x10C
       ┌────────┬─────────┬─────────┬─────────┐
       │ inst A │ inst B  │ inst C  │ inst D  │
       └────────┴─────────┴─────────┴─────────┘
                    ▲
        PC = 0x104  (next to fetch: B)
```

Cycle: fetch from where PC points → PC advances to the next instruction → execute. A **jump** (used for `if`, `for`, function calls) simply **overwrites PC** with a new address. Everything about "control flow" is PC manipulation.

> **Function calls and the PC:** when `main` calls `add`, the CPU saves the *return address* (the PC value after the call) on the stack and sets PC to `add`'s first instruction. `RET` pops that address back into PC. That's how execution "returns to the line after the call" (Chapter 4).

### 6.3 Stack Pointer (SP)

Holds the address of the **top of the stack** (Chapter 18). Push a value → SP moves; pop a value → SP moves back. Creating a function's stack frame is (mostly) just **subtracting from SP** to reserve room, and destroying it is **adding back**.

```
Stack (grows toward LOWER addresses on x86-64):

 higher addresses
    ┌──────────────┐
    │ main's frame │
    ├──────────────┤
    │ add's frame  │
    ├──────────────┤ ◄── SP (top of stack)
    │  (free)      │
    └──────────────┘
 lower addresses
```

### 6.4 Base Pointer (BP), a.k.a. Frame Pointer

Marks the **base of the current function's stack frame**, a fixed reference point. While SP can move around as the function pushes and pops things, BP stays put, so the CPU can find local variables at **fixed offsets from BP** (e.g., "local `x` is at BP − 8").

```
    ┌───────────────┐
    │ local var 2   │
    ├───────────────┤ ◄── SP (moves)
    │ local var 1   │
    ├───────────────┤
    │ saved old BP  │
    │ return address│
    ├───────────────┤ ◄── BP (fixed for this frame)
```

We'll dive deeper into SP and BP, with a step-by-step "dance" and real disassembly, in [Chapter 29](29-sp-vs-bp.md).

### Summary

| Register | Full name | Holds | Analogy |
|----------|-----------|-------|---------|
| **IR** | Instruction Register | The instruction being executed now | The problem on the board |
| **PC** | Program Counter / Instruction Pointer | Address of the **next** instruction | A bookmark / your finger under the line |
| **SP** | Stack Pointer | Address of the **top** of the stack | "Top plate is here" sign |
| **BP** | Base Pointer / Frame Pointer | Address of the **base** of the current frame | The bottom edge of the current frame |

---

## 7. General-purpose registers and "64-bit"

**General-purpose registers** (GPRs) hold whatever values the running code needs: numbers being added, addresses being dereferenced, loop counters. On x86-64 there are 16, named `RAX, RBX, RCX, RDX, RSI, RDI, R8 … R15` (plus SP/BP). Go's compiler assigns your variables to them when it can, which is one reason local variables are so fast.

### What does "8-bit / 16-bit / 32-bit / 64-bit" mean?

It's (roughly) the **size of a register**, the natural "chunk" of data the CPU handles in one operation, and often also the size of a memory address.

| Era | Bits per register | Max unsigned value | Max addressable memory (if address = register size) |
|-----|------------------|--------------------|-----------------------------------------------------|
| 8-bit (1970s micros) | 8 | 255 | 256 bytes (they used tricks to reach 64 KB) |
| 16-bit (1980s) | 16 | 65,535 | 64 KB |
| 32-bit (1990s–2000s) | 32 | ~4.29 billion | **4 GiB** |
| 64-bit (today) | 64 | ~1.8 × 10¹⁹ | 16 EiB in theory (real CPUs use 48–57 bits) |

This explains an old puzzle: a 32-bit computer can't use more than 4 GiB of RAM, because a 32-bit address can only name 2³² different bytes. It's also why `int` in Go is 64 bits on your 64-bit machine (Chapter 2), and why a pointer is 8 bytes (Chapter 24).

---

## 8. From your code to a running program

Let's trace the whole journey. Your source file:

```go
package main

import "fmt"

func add(a, b int) int { return a + b }

func main() { fmt.Println(add(10, 12)) }
```

**Stage 1: Source code (text)** on disk: `main.go`. A human-readable text file.

**Stage 2: Compilation** (Chapter 19): `go build` translates it into **machine code**, a long list of binary CPU instructions, packaged with data and runtime into an **executable file** (`myprogram`) on disk.

**Stage 3: Loading:** you type `./myprogram`. The **operating system** (Chapter 27):
1. Creates a new **process**.
2. Reserves a chunk of RAM for it.
3. **Copies** the code and initialized data from the file on disk into that RAM (the **code segment** and **data segment**), and sets up the **stack** and **heap** areas.
4. Sets the CPU's **PC** to the program's entry point.

**Stage 4: Execution:** the CPU runs the fetch-decode-execute cycle from that entry point, over and over, until the program exits.

```
main.go ──compile──► myprogram (binary file on DISK)
                          │  OS loads it
                          ▼
                 ┌──────────────────────────┐
                 │ RAM: process memory      │
                 │  Code | Data | Stack | Heap│
                 └──────────┬───────────────┘
                            │ PC → instructions
                            ▼
                       CPU executes
```

---

## 9. The execution dance

Follow the CPU as it executes `c := a + b` (with `a = 10`, `b = 12`) in pseudo-assembly:

| Step | Instruction | What the CPU does |
|------|-------------|-------------------|
| 1 | `LOAD R1, [a]` | **Fetch** this instruction (PC → IR). CU decodes it, loads value of `a` from RAM into register R1 (R1 = 10). PC advances |
| 2 | `LOAD R2, [b]` | Same: R2 = 12 |
| 3 | `ADD R1, R2` | CU tells the **ALU** to add R1 and R2; result 22 goes into R1 |
| 4 | `STORE [c], R1` | Write R1 (22) to the RAM location of `c` |

```
PC → step 1 → step 2 → step 3 → step 4 → next instruction...
      LOAD     LOAD     ADD      STORE
```

Each of these is a few nanoseconds or less. A modern core executes billions per second. All of Go's higher-level features (loops, functions, closures, goroutines) are compiled down to sequences like this.

---

## 10. See real machine code

Go can show you the real instructions it generated. Write this program:

```go
package main

import "fmt"

//go:noinline
func add(a, b int) int {
	return a + b
}

func main() {
	fmt.Println(add(10, 12))
}
```

Build it, then disassemble `add`:

```bash
go build -o cpu .
go tool objdump -s 'main.add$' cpu
```

Real output on an x86-64 Linux machine (default optimization):

```
TEXT main.add(SB) main.go
  main.go:11    0x4a5de0    4801d8      ADDQ BX, AX
  main.go:11    0x4a5de3    c3          RET
```

Read it:

| Part | Meaning |
|------|---------|
| `0x4a5de0` | The **address** of the instruction: the value the **PC** holds when it's about to run |
| `4801d8` | The instruction's **machine-code bytes** (what lands in the **IR**) |
| `ADDQ BX, AX` | "Add the 64-bit (Quad) register BX to AX." Go passes the arguments `a` and `b` in registers `AX` and `BX`, so this single instruction *is* `a + b`, with the result left in `AX` |
| `RET` | Return to the caller: pop the saved return address into the **PC** |

Two instructions for the whole function. This is the CU, ALU, and registers doing exactly what this chapter described.

Now build **without optimization** to see more of the machinery:

```bash
go build -gcflags='-N -l' -o cpu_noopt .
go tool objdump -s 'main.add$' cpu_noopt
```

```
  0x4a5f00   55                PUSHQ BP          ; save the caller's base pointer
  0x4a5f01   4889e5            MOVQ SP, BP       ; BP = SP: mark this frame's base
  0x4a5f04   4883ec08          SUBQ $0x8, SP     ; SP -= 8: reserve stack space (the frame!)
  0x4a5f08   4889442418        MOVQ AX, 0x18(SP) ; spill argument a to the stack
  0x4a5f0d   48895c2420        MOVQ BX, 0x20(SP) ; spill argument b to the stack
  0x4a5f12   48c7042400000000  MOVQ $0x0, 0(SP)  ; zero the result slot
  0x4a5f1a   4801d8            ADDQ BX, AX       ; a + b   ← the actual work
  0x4a5f1d   48890424          MOVQ AX, 0(SP)    ; store the result
  0x4a5f21   4883c408          ADDQ $0x8, SP     ; SP += 8: pop the frame
  0x4a5f25   5d                POPQ BP           ; restore the caller's BP
  0x4a5f26   c3                RET               ; return (pop return address → PC)
```

(The comments after `;` are mine.) There it is in real code: `SP` and `BP` being adjusted to create and destroy a stack frame; exactly the story of Chapters 4 and 18, now visible in hardware terms. Chapter 29 unpacks this dance.

---

## 11. What is a process?

Now the central definition of this part of the course.

> A **process** is a **program in execution**: a running instance of a program, made of **its own private memory** plus **the CPU state** needed to run it.

A *program* is a passive file on disk (your compiled binary). A *process* is that program **alive**: loaded into RAM, with a CPU executing it. Run the same program twice and you have **two processes** from one program.

### A process = memory + CPU time

```
┌───────────────────────────── PROCESS ─────────────────────────────┐
│                                                                   │
│  MEMORY (its own private address space)                           │
│  ┌──────────────┐                                                 │
│  │ Code segment │  ← machine instructions (read-only)             │
│  ├──────────────┤                                                 │
│  │ Data segment │  ← package-level variables                      │
│  ├──────────────┤                                                 │
│  │ Heap         │  ← dynamically allocated data                   │
│  ├──────────────┤                                                 │
│  │ Stack        │  ← function frames                              │
│  └──────────────┘                                                 │
│                                                                   │
│  CPU STATE (loaded onto the CPU while it's running)               │
│  PC, IR, SP, BP, general-purpose registers, flags                 │
│                                                                   │
│  OS BOOKKEEPING                                                   │
│  PID, owner, open files, priority, state... (the PCB, Chapter 30) │
└───────────────────────────────────────────────────────────────────┘
```

> **Analogy: a wedding.** The client (you) orders a wedding (runs a program). The planner (**OS**) books a **venue** (RAM), assigns **staff** (CPU time), and coordinates the schedule. The *whole event, from the first guest arriving to the last one leaving,* is one **process**. When it ends, the venue is cleaned up (memory freed).

### The life of a process

```
   run ./myprogram
        │
        ▼
    ┌────────┐  OS allocates memory, loads code, sets PC to entry point
    │ CREATED│
    └───┬────┘
        ▼
    ┌────────┐  instructions execute...
    │RUNNING │ ◄──────┐
    └───┬────┘        │ (chapters 30–31: the OS pauses and resumes it many times)
        ▼             │
    ┌────────┐        │
    │WAITING │────────┘  (e.g., waiting for a file or the network)
    └───┬────┘
        ▼
    ┌────────┐  main returns / os.Exit / crash → OS frees everything
    │ EXITED │
    └────────┘
```

Key properties of a process:

| Property | Meaning |
|----------|---------|
| **Isolated** | Its memory is private. Process A can't read process B's variables (the OS + CPU enforce it) |
| **Has a PID** | A unique number assigned by the OS (`os.Getpid()`) |
| **Heavyweight** | Creating one is relatively costly (new memory space, bookkeeping) |
| **Has at least one thread of execution** | (Chapter 32) |

Your Go program is **one** process. When your `main` returns, that process ends.

---

## 12. See a real process

Linux exposes every process's memory layout as a file. Let's ask a Go program to show **its own**:

```go
package main

import (
	"fmt"
	"os"
	"strings"
)

func main() {
	data, _ := os.ReadFile("/proc/self/maps") // Linux only
	for _, line := range strings.Split(string(data), "\n") {
		if strings.Contains(line, "cpu") || strings.Contains(line, "[stack]") {
			fmt.Println(line)
		}
	}
}
```

Real (abbreviated) output from running a compiled binary named `cpu`:

```
00400000-004a7000 r-xp 00000000 ... /path/to/cpu      ← CODE segment  (r-x: read+execute)
004a7000-0057b000 r--p 000a7000 ... /path/to/cpu      ← read-only data (constants, strings)
0057b000-00586000 rw-p 0017b000 ... /path/to/cpu      ← DATA segment  (rw-: read+write)
7fff17e44000-7fff17e66000 rw-p 00000000 ... [stack]   ← STACK
```

Compare with Chapter 18's diagram:

- **`r-xp`** (read, execute, no write) is the **code segment**: read-only, exactly as promised.
- **`rw-p`** (read + write) is the **data segment**.
- **`[stack]`** is the main thread's stack.
- The Go **heap** doesn't appear as `[heap]`, because Go's runtime manages its own memory arenas (allocated with `mmap`) instead of the classic C heap.

The permissions column (`r`, `w`, `x`) is the OS and CPU enforcing protection: try to *write* to the code segment and the hardware raises a fault. This is a real process's memory map, and it matches our diagrams.

You can also inspect any running process with `cat /proc/<pid>/maps` or `ps -o pid,rss,vsz,cmd -p <pid>`.

---

## 13. Many processes at once

Your computer runs hundreds of processes: browser, editor, music, system services, and your Go program.

```
RAM
┌──────────────────────────┐
│ Process 1: Browser       │ ◄── own code/data/stack/heap
├──────────────────────────┤
│ Process 2: VS Code       │
├──────────────────────────┤
│ Process 3: Your Go app   │
├──────────────────────────┤
│ Process 4: Music player  │
├──────────────────────────┤
│ OS kernel                │
└──────────────────────────┘

CPU (a few cores): runs ONE process's instructions per core at any instant
```

Two big questions arise, and the next chapters answer them:

1. **How do they share a CPU with only a few cores?** By rapid switching: the OS runs process 1 for a few milliseconds, saves its registers, loads process 2's registers, and so on. **(Chapter 30: context switching.)**
2. **How does the CPU know where each process's stack and instructions are, when they're switched in and out?** Each process's SP, BP, PC, etc. are saved and restored. **(Chapter 29: SP and BP; Chapter 30: PCB.)**

(Modern OSes also use **virtual memory**, so each process *believes* its addresses start from the same low numbers, while the OS quietly maps them to different physical RAM. It's why two processes can both "have" address `0x400000` without conflict.)

---

## 14. The complete picture

```
     ┌───────────────┐
     │  main.go      │  ← you write
     └───────┬───────┘
             │ go build (compiler + linker)
             ▼
     ┌───────────────┐
     │ executable    │  ← on DISK: machine code + data
     └───────┬───────┘
             │ ./program  (OS loads it: creates a PROCESS)
             ▼
     ┌───────────────────────────────────────┐
     │ RAM: Code | Data | Heap | Stack       │
     └───────┬───────────────────────────────┘
             │ PC points into Code
             ▼
     ┌───────────────────────────────────────┐
     │ CPU                                   │
     │  CU: fetch → decode → execute         │
     │  ALU: does the math                   │
     │  Registers: IR, PC, SP, BP, GPRs      │
     └───────────────────────────────────────┘
```

---

## 15. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "A process is just code running" | It's the running code **plus** its private memory, CPU state, and OS bookkeeping |
| "A program and a process are the same" | A program is a file; a process is a running instance. One program → many processes |
| "The CPU reads variables directly from RAM as it computes" | It loads them into **registers** first |
| "The ALU decides what to do next" | The **Control Unit** directs; the ALU just computes |
| "The Program Counter holds the current instruction" | It holds the address of the **next** one (the current one is in the IR) |
| "`if` and `for` are magic" | They compile to comparisons and **jumps** that modify the PC |
| "All processes share memory" | Each has private virtual memory (unless explicitly shared) |
| "64-bit means twice as fast as 32-bit" | It means wider registers/addresses, not automatic speed |

---

## 16. Exercises

### Exercise 1: Match the register
Match each to its role: **IR, PC, SP, BP**.
(a) points to the next instruction to fetch; (b) points to the top of the stack; (c) holds the instruction currently being decoded; (d) marks the base of the current stack frame.

<details><summary>Solution</summary>

(a) PC. (b) SP. (c) IR. (d) BP.
</details>

### Exercise 2: Who does what?
For `if x > y { ... }`, describe the roles of the ALU, the CU, and the PC.

<details><summary>Solution</summary>

The ALU compares `x` and `y` (subtracting) and sets flag bits. The CU examines the flags and decides whether to take the branch. If the condition is false, it *jumps*, meaning the PC is overwritten with the address of the code after the `if` block; otherwise the PC just advances into the block.
</details>

### Exercise 3: Address space
How many bytes can a 16-bit address name? A 32-bit address? Express the latter in GiB.

<details><summary>Solution</summary>

2¹⁶ = 65,536 bytes (64 KiB). 2³² = 4,294,967,296 bytes = 4 GiB.
</details>

### Exercise 4: Program vs process
You double-click your Go program's binary twice, and now it's running in two windows. How many programs? How many processes? What differs between them?

<details><summary>Solution</summary>

One program (one file), two processes. Each has its own PID, private memory (own stack, heap, and copy of the data segment; the read-only code pages may be physically shared by the OS), and its own CPU state.
</details>

### Exercise 5: Read the disassembly
In the optimized `add` disassembly (`ADDQ BX, AX; RET`), where are the arguments `a` and `b` and where's the result?

<details><summary>Solution</summary>

`a` is in register `AX`, `b` in `BX` (Go's register-based calling convention). `ADDQ BX, AX` computes `AX = AX + BX`, so the result is left in `AX`, which is where the caller expects the return value.
</details>

### Exercise 6: Explore your own process
On Linux, write a Go program that prints its PID and sleeps 60 seconds; in another terminal run `cat /proc/<pid>/maps | head` and `cat /proc/<pid>/status | grep -E 'Name|Pid|VmRSS'`. Identify the code and data mappings.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"os"
	"time"
)

func main() {
	fmt.Println("PID:", os.Getpid())
	time.Sleep(60 * time.Second)
}
```
In the `maps` output, look for the `r-xp` mapping of your binary (code), the following `r--p` (read-only data) and `rw-p` (data), and `[stack]`. `VmRSS` in `status` shows how much physical RAM the process is using.
</details>

### Exercise 7 (challenge): Trace an `if`
Write pseudo-assembly (LOAD, CMP, JUMP-IF-GREATER-OR-EQUAL, ADD, STORE) for `if x < 10 { x = x + 1 }` and describe the PC at each step.

<details><summary>Solution</summary>

```
100: LOAD  R1, [x]        ; PC=100 → R1 = x        (PC → 101)
101: CMP   R1, 10         ; ALU subtracts, sets flags (PC → 102)
102: JGE   105            ; if x >= 10, PC = 105 (skip the body); else PC → 103
103: ADD   R1, 1          ; body: R1 = R1 + 1      (PC → 104)
104: STORE [x], R1        ; write back             (PC → 105)
105: ...                  ; code after the if
```
The `if` is just a compare plus a conditional jump that rewrites the PC.
</details>

---

## 17. Quiz

1. What are the three main parts of a CPU core?
2. Which unit performs addition?
3. What does the Program Counter hold?
4. What's the difference between SP and BP?
5. What does "64-bit" refer to?
6. Define "process" in one sentence.
7. What does the `r-xp` permission on a memory mapping tell you?

<details><summary>Answers</summary>

1. Control Unit, ALU, registers.
2. The ALU.
3. The address of the next instruction to fetch.
4. SP tracks the top of the stack (moves as you push/pop); BP marks the fixed base of the current frame.
5. The size of the CPU's registers (and typically addresses).
6. A program in execution: its private memory plus CPU state.
7. Read + execute, not writable: a code segment.
</details>

---

## 18. Summary

- A CPU core has a **Control Unit** (directs: fetch → decode → execute), an **ALU** (calculates), and **registers** (tiny, fastest storage).
- Key registers: **IR** (current instruction), **PC** (next instruction address), **SP** (top of stack), **BP** (base of current frame), plus **general-purpose** registers (16 on x86-64).
- `if`/`for`/function calls are **jumps** that rewrite the PC; function frames are created and destroyed by adjusting **SP** (and **BP**).
- Code is compiled to a **binary** on disk; the OS **loads** it into RAM, points the PC at its start, and the CPU executes it.
- A **process** is a **program in execution**: private **memory** (code, data, heap, stack) + **CPU state** + OS bookkeeping. One program can have many processes.
- You can *see* all of it: `go tool objdump` for real instructions, `/proc/self/maps` for a real process's memory map.

### ➡️ What's next?

[Chapter 29](29-sp-vs-bp.md) zooms into **SP and BP**, the "stack pointer dance", to show exactly how the CPU builds and tears down stack frames when functions call each other.
