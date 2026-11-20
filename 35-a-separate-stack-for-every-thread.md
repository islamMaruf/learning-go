# Chapter 35: A Separate Stack for Every Thread

> **Goal of this chapter:** Answer four questions most engineers can't: *Where does each thread's stack live? How many stacks exist in one process? Why can't threads share a stack? How does the OS allocate them?* Then see how Go changes the answer with **small, growable goroutine stacks**, which sets up Chapter 36.

**Difficulty:** 🔴 Advanced (conceptual)  **Estimated time:** 2–2.5 hours  **Prerequisite:** [Chapters 18, 29, 32](32-threads.md)

> ⚠️ **Heads-up:** from this chapter on, the material is *advanced*. It combines operating-system knowledge with Go internals. If something feels heavy, re-read Chapters 28–32, then return. Understanding this chapter is what separates "I use goroutines" from "I understand goroutines".

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Quick recap: processes and threads](#2-quick-recap)
3. [Process anatomy, revisited](#3-process-anatomy-revisited)
4. [The music player, again](#4-the-music-player-again)
5. [The stack problem](#5-the-stack-problem)
6. [Why each thread needs its own stack](#6-why-each-thread-needs-its-own-stack)
7. [Where the stacks live in memory](#7-where-the-stacks-live-in-memory)
8. [A real-world case: many threads in one program](#8-a-real-world-case)
9. [How big is a thread stack?](#9-how-big-is-a-thread-stack)
10. [Who manages threads and their stacks?](#10-who-manages-threads-and-their-stacks)
11. [Guard pages: what happens at the end of a stack](#11-guard-pages)
12. [Go's twist: small, growable goroutine stacks](#12-gos-twist)
13. [Try it: a stack that grows to 256 MiB](#13-try-it)
14. [Complete visualization](#14-complete-visualization)
15. [Common misconceptions](#15-common-misconceptions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- Exactly what is **shared** and what is **private** in a multi-threaded process (memory-map view)
- Why threads **must** have separate stacks: return addresses, locals, SP and BP
- How a stack is **allocated** (a region of virtual memory with a guard page), and typical sizes
- Who creates and manages threads: the **OS kernel** vs. a **language runtime**
- How Go avoids the cost of big thread stacks: **goroutines** with small stacks that **grow on demand**

---

## 2. Quick recap

| Concept | Definition |
|---------|------------|
| **Process** | A program in execution with its own private memory: code, data, heap, stack |
| **Thread** | An independent flow of execution *inside* a process. Owns a stack + registers; shares code, data, heap |
| **Stack** | Where function calls live: frames with arguments, return addresses, saved BP, and locals (Chapters 18 and 29) |

When a process starts it has **one** thread (the *main thread*) and **one** stack. Every extra thread you create adds **one more stack**. That is the fact this chapter unpacks.

---

## 3. Process anatomy, revisited

You double-click a music player. What happens?

```
You launch the app
       ↓
OS loads the binary from disk
       ↓
OS creates a PROCESS and reserves virtual memory for it
       ↓
┌────────────────────────────────────┐
│ Process memory                     │
│  ┌──────────────────────────────┐  │
│  │ Code segment   (instructions)│  │
│  ├──────────────────────────────┤  │
│  │ Data segment   (globals)     │  │
│  ├──────────────────────────────┤  │
│  │ Heap           (dynamic data)│  │
│  ├──────────────────────────────┤  │
│  │ Stack          (main thread) │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
       ↓
The OS also creates ONE thread (the main thread),
starts it at the program's entry point, and its
PC / SP / BP point into the code and this stack.
```

Initial state: **1 process, 1 thread, 1 stack** (about 8 MiB reserved on Linux, see section 9), with code, data, and heap shared (trivially so, since there's only one thread yet).

---

## 4. The music player, again

Remember the music player from Chapter 32. At least three things happen "simultaneously":

```
1. Draw the screen (UI)      ← redraw the song list, buttons, progress bar
2. Play audio                ← decode MP3 → send samples to the sound card, endlessly
3. Track the time            ← update 0:37 → 0:38 every second
```

A single thread doing all three, one after the other, would stutter. So the program creates **more threads**:

```
Process: MusicPlayer
 ├── Thread 1: UI        (draws, handles clicks)
 ├── Thread 2: Audio     (decode + playback loop)
 └── Thread 3: Timer     (updates the clock)
```

Shared: the song list, `isPlaying`, `currentPosition`. But each thread is **doing something different at this instant**, so each is at a *different place in the code* and in the middle of *different function calls*. And that brings us to the problem.

---

## 5. The stack problem

The stack stores **the state of a chain of function calls**. Suppose these are the call chains *right now*:

```
UI thread:      main → runUI → handleClick → showMenu → drawButton
Audio thread:   audioLoop → decodeFrame → applyVolume
Timer thread:   timerLoop → formatTime
```

Each chain has its own:
- **Return addresses**: where to resume after each function ends,
- **Local variables**: `frameBuffer` in `decodeFrame`, `label` in `drawButton`, ...,
- **Saved BP chain and SP position** (Chapter 29).

Now imagine (hypothetically) **all three threads shared ONE stack**. What would happen?

---

## 6. Why each thread needs its own stack

Let's simulate the disaster.

**One shared stack**:

```
 higher addresses
 ┌────────────────────────┐
 │ UI:    main frame      │
 │ UI:    runUI frame     │
 ├────────────────────────┤
 │ Audio: audioLoop frame │  ← Audio thread pushes its frame ON TOP of UI's
 │ Audio: decodeFrame     │
 ├────────────────────────┤ ◄── SP
 │ ...
```

Now the UI thread's `handleClick` returns. A return **pops the top frame**, but the top frame belongs to *Audio*! Then:

1. UI pops Audio's frame (destroying data Audio still needs) and jumps to *Audio's* return address.
2. UI is now running in the wrong code with the wrong locals.
3. Audio later returns and pops the *UI's* frame...

Result: **total corruption**. The stack works only because calls and returns are **strictly nested (LIFO) within one flow of execution**. Two independent flows interleave in an order **nobody controls** (the OS preempts at arbitrary moments), so a shared stack cannot stay a valid stack.

```
Timeline (SHARED stack, broken):
  UI    calls f      → push f
  Audio calls g      → push g          (on top of f)
  UI    returns from f → POPS g!!  ✗   (wrong frame, wrong return address)
```

**Conclusion**, stated as a rule:

> **Each thread must have its own stack.** A stack encodes *one* sequential flow's call chain. Independent flows need independent stacks, each with its own **SP** and **BP**.

What can be shared safely? Things that aren't tied to a *single* call chain: **code** (read-only), **globals**, **heap** objects. Those stay shared. That's exactly the table from Chapter 32.

---

## 7. Where the stacks live in memory

Inside the process's **virtual address space**, the OS reserves a **separate region for each thread's stack**. With three threads:

```
Process virtual memory (high → low addresses):

 ┌──────────────────────────────────────┐
 │  Main thread's stack   (grows ↓)     │  ← "[stack]" in /proc/<pid>/maps
 ├──────────────────────────────────────┤
 │        ... unmapped gap ...          │
 ├──────────────────────────────────────┤
 │ ▓ guard page                         │
 │  Thread 2's stack     (grows ↓)      │  ← a separate mmap'd region
 ├──────────────────────────────────────┤
 │ ▓ guard page                         │
 │  Thread 3's stack     (grows ↓)      │  ← another separate region
 ├──────────────────────────────────────┤
 │        ... heap, libraries ...       │
 ├──────────────────────────────────────┤
 │  Data segment (globals)  — shared    │
 │  Code segment            — shared    │
 └──────────────────────────────────────┘
```

Each thread has:
- its **own stack region** in the shared address space,
- its **own SP** register pointing somewhere inside *its* region,
- its **own BP**, PC, and general registers, all saved in that thread's control block when it's switched out (Chapters 30 and 32).

When the OS switches from Thread 2 to Thread 3, it swaps **SP and BP** (and the other registers). That single change moves the CPU's "current stack" from one region to another; every push/pop/call/return now operates on Thread 3's stack.

Everything else (code, globals, heap) is **the same memory** for all of them. A pointer to a heap object created by Thread 2 is valid in Thread 3.

> ⚠️ *Because all stacks live in one shared address space, a bug in one thread that writes through a bad pointer **can** corrupt another thread's stack.* The separation is by convention and by the OS's bookkeeping, not by hardware protection (unlike between processes). That's another difference between threads and processes.

Also: **a pointer into another thread's stack is legal but dangerous** (the stack frame may vanish). In Go, escape analysis (Chapter 18) moves such variables to the heap for you.

---

## 8. A real-world case

Task managers show thread counts per process. A typical editor like **VS Code** (an Electron app) can show dozens of threads in a single process (numbers like 47 are common):

```
Process: code (main)
  Threads: 47
   ├── UI thread
   ├── Compositor / GPU threads
   ├── Network thread
   ├── File-watcher threads
   ├── JavaScript engine helper threads
   ├── Timer threads
   └── ... 47 threads → 47 stacks
```

That means **47 separate stack regions** in that process, one per thread, each with its own call chain, SP and BP. The operating system's scheduler treats each as an independent flow, running them on the available cores (Chapter 31).

You can look at your own machine:

```bash
# Linux: threads of a process
ls /proc/<pid>/task | wc -l
# or
ps -o nlwp,cmd -p <pid>        # nlwp = number of lightweight processes (threads)

# macOS
ps -M <pid>

# Windows: Task Manager → Details → add the "Threads" column
```

And for a Go program:

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	entries, _ := os.ReadDir("/proc/self/task") // Linux: one directory per thread
	fmt.Println("OS threads in this tiny Go program:", len(entries))
}
```

Even this trivial program prints about `4`–`5` (the Go runtime's own helper threads: scheduler workers, a background monitor, GC workers).

---

## 9. How big is a thread stack?

A thread's stack is a fixed-size region **reserved** when the thread is created. It's *virtual* memory: physical RAM is committed **page by page, only as the stack is actually used**. Typical defaults:

| OS / runtime | Default stack size per thread |
|--------------|-------------------------------|
| **Linux** (main thread; `ulimit -s`) | **8 MiB** (adjustable) |
| **Linux** (`pthread_create` default) | **8 MiB** (from `ulimit -s`) |
| **macOS** (secondary threads) | 512 KiB (main thread 8 MiB) |
| **Windows** | 1 MiB by default |
| **Go goroutine** | **starts at ~2 KiB**, grows on demand, up to 1 GiB by default |

Check your Linux limit:

```bash
ulimit -s      # 8192  (KiB) → 8 MiB
```

Or from Go:

```go
package main

import (
	"fmt"
	"syscall"
)

func main() {
	var rl syscall.Rlimit
	syscall.Getrlimit(syscall.RLIMIT_STACK, &rl) // Linux/macOS
	fmt.Printf("stack limit: %d KiB\n", rl.Cur/1024)
}
```

Real output on the test machine: `stack limit: 8192 KiB`.

### Why not just make stacks tiny, or huge?

- **Too small:** deep recursion or big local arrays overflow quickly (crash).
- **Too big:** with thousands of threads you exhaust *address space* and kernel resources. 10,000 threads × 8 MiB = **80 GiB of virtual space**. Physical memory use stays low if stacks are barely touched, but the reservations, page tables, and guard pages still cost; and 100,000+ threads is out of reach.

The fixed size is a **compromise**. Nobody knows at thread-creation time how much stack the thread will need.

---

## 10. Who manages threads and their stacks?

There are two possible managers, and this distinction is the key to understanding Go.

### A. The operating system kernel (real / native threads)

When a program asks for a thread (e.g., `pthread_create` on Linux, `CreateThread` on Windows), the **kernel**:
1. Allocates the thread's **stack** (a virtual memory region + guard page) and its **kernel-side structures** (a thread control block, a small kernel stack),
2. Adds the thread to the **scheduler's** run queues,
3. **Preempts and switches** it with timer interrupts (Chapter 30),
4. Frees everything when the thread exits.

This is powerful and safe but comparatively heavy (µs to create, MBs of stack reserved, µs to switch).

### B. The language runtime (user-space / "green" threads)

A runtime can implement its **own** lightweight threads on top of a few OS threads: **it** allocates their stacks (from the heap), **it** decides when to switch, **it** stores their saved registers. The kernel doesn't even know they exist.

**Go does this.** A goroutine is a runtime-managed thread with its own small stack, scheduled by the **Go scheduler** (Chapter 36 and 39). The kernel only sees the handful of OS threads that the runtime uses as "workers".

```
        ┌───────────────────── Go process ────────────────────────┐
        │  goroutines: G1  G2  G3  G4  G5  G6 ... G100000        │  ← managed by the Go runtime
        │              │   │   │                                  │     (each has a tiny growable stack)
        │           ┌──▼───▼───▼──┐                               │
        │           │ Go scheduler│                               │
        │           └──┬───────┬──┘                               │
        │           OS thread  OS thread    ...(a few)            │  ← managed by the kernel
        └──────────────┼───────┼──────────────────────────────────┘
                     kernel schedules these onto CPU cores
```

| | OS thread | Goroutine |
|--|-----------|-----------|
| Stack allocated by | Kernel / libc | Go runtime (from the heap) |
| Initial stack size | 1–8 MiB (virtual) | ~2 KiB |
| Can the stack grow? | Fixed size at creation | Yes: automatically |
| Scheduled by | Kernel (preemptive, timer) | Go scheduler (cooperative + preemptive) |
| Switch cost | µs | ~100–200 ns |

---

## 11. Guard pages

What happens if a thread uses more stack than it was given? Below each stack region the OS places a **guard page**: a page marked *no-access*. Overrunning the stack touches the guard page, the CPU raises a **fault**, and the OS terminates the process with a **segmentation fault** ("stack overflow"). A guard page turns silent memory corruption (writing into the neighbor's memory) into a loud, immediate crash.

```
 ┌──────────────────────┐ ◄── stack base (high address); SP starts here
 │   stack (used ↓)     │
 │        ...           │
 ├──────────────────────┤ ◄── stack limit
 │ ▓▓▓ GUARD PAGE ▓▓▓   │  ← no read/write allowed: touching it = fault
 ├──────────────────────┤
 │ (another mapping)    │
```

Go's runtime does something different and friendlier for goroutine stacks: see the next section.

---

## 12. Go's twist

Go can't accept fixed 8 MiB stacks: a server with 100,000 connections would reserve 800 GiB. So it does two clever things:

### 1. Start tiny

Each goroutine begins with a **small stack** (around 2 KiB; the exact figure can change between versions and adaptive sizing may pick larger starts).

### 2. Grow when needed ("segmented → contiguous, copying stacks")

Every function prologue in Go contains a **stack check**: *"Is there enough room between SP and the stack's limit for this function's frame?"* (You saw it in Chapter 29's disassembly: `CMPQ SP-ish, 0x10(R14); JBE morestack`.) If not, the runtime:

1. **Allocates a new stack twice as big**,
2. **Copies** all the frames from the old stack to the new one,
3. **Adjusts every pointer** that pointed into the old stack (possible because the compiler emits exact pointer maps for each frame),
4. Frees the old stack and continues.

```
Before:  [ 2 KiB stack: f g h ]  → h calls i, no room!
Step 1:  allocate [ 4 KiB stack ]
Step 2:  copy frames f g h into it; fix pointers
Step 3:  continue calling i in the new stack
```

The garbage collector can also **shrink** stacks that are mostly empty.

Because of this:
- **Recursion depth is limited only by memory** (default max 1 GiB per goroutine), not by an 8 MiB thread limit.
- **Goroutine creation is cheap** (allocate 2 KiB, not 8 MiB).
- **Pointers into the stack** must be tracked, which is why Go's compiler is careful about which variables live on the stack (Chapter 18: escape analysis).

---

## 13. Try it

Deep recursion that would crash a C program on its 8 MiB main-thread stack works fine in a goroutine, because its stack grows:

```go
package main

import (
	"fmt"
	"runtime"
)

//go:noinline
func stackInuse() uint64 {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return m.StackInuse
}

var peak = make(chan uint64)

func deep(n int) int {
	if n == 0 {
		peak <- stackInuse() // measure at the very bottom of the recursion
		return 0
	}
	var pad [16]int // make each frame ~200 bytes
	pad[n%16] = n
	return deep(n-1) + pad[n%16] - n + 1
}

func main() {
	fmt.Printf("stack in use before: %d KiB\n", stackInuse()/1024)
	go deep(1_000_000) // a million nested calls
	fmt.Printf("stack in use at the bottom of a 1,000,000-deep recursion: %d MiB\n", <-peak>>20)
}
```

Real output:

```
stack in use before: 320 KiB
stack in use at the bottom of a 1,000,000-deep recursion: 256 MiB
```

A million frames of ~200 bytes ≈ 200 MB. The goroutine's stack **grew from 2 KiB to 256 MiB** (stacks double, so 256 MiB is the next power of two above ~200 MB) and the program worked. In C on an 8 MiB stack that would be a crash after ~40,000 calls.

And the limit? Try 50 million levels instead: the runtime stops you at its **1 GB** cap with a message like this (real output from an over-deep run):

```
runtime: goroutine stack exceeds 1000000000-byte limit
fatal error: stack overflow
```

(That's a **fatal error**, not a `panic` you can recover from. See Chapter 29.)

---

## 14. Complete visualization

A process with a Go runtime, three OS threads, and many goroutines:

```
┌──────────────────────────────── ONE PROCESS ─────────────────────────────────┐
│                                                                              │
│  SHARED (all threads and goroutines)                                         │
│  ┌──────────────┬──────────────────┬───────────────────────────────────┐     │
│  │ Code segment │ Data segment     │ Heap (goroutine stacks, objects…) │     │
│  └──────────────┴──────────────────┴───────────────────────────────────┘     │
│                                                                              │
│  OS-THREAD stacks (few, each large and fixed)                                │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                                │
│  │ Thread 1   │ │ Thread 2   │ │ Thread 3   │   each: own SP, BP, PC, regs   │
│  │ stack 8MiB │ │ small sys  │ │ small sys  │   (Go's worker threads mostly  │
│  │ (main)     │ │ stack      │ │ stack      │    run goroutines on THEIR     │
│  └────────────┘ └────────────┘ └────────────┘    stacks, switching SP)       │
│                                                                              │
│  GOROUTINE stacks (many, small, growable, live in the heap)                  │
│  ┌───┐ ┌───┐ ┌─────┐ ┌───┐ ┌───┐   ┌────────────┐ ...                        │
│  │G1 │ │G2 │ │ G3  │ │G4 │ │G5 │   │ G6 (grew   │  each 2KiB → up to 1GiB    │
│  │2K │ │2K │ │ 8K  │ │2K │ │2K │   │  to 64K)   │                            │
│  └───┘ └───┘ └─────┘ └───┘ └───┘   └────────────┘                            │
│   ▲ the scheduler swaps a worker thread's SP between these stacks            │
│     (that's a goroutine switch: save/restore PC, SP, BP: ~150 ns)            │
└──────────────────────────────────────────────────────────────────────────────┘
```

**The big picture**:
- **Processes** are isolated (separate memory).
- **Threads** share memory but need **separate stacks** (own SP/BP/registers).
- **Goroutines** are *"threads" made by the Go runtime*: separate small stacks stored in the heap, multiplexed onto a few OS threads by swapping SP/PC. So, again: **one stack per flow of execution**. Only who manages it, and how big it starts, differs.

---

## 15. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "All threads share one stack" | Each thread has its **own** stack |
| "Threads have separate heaps" | The heap is **shared** by all threads (and goroutines) |
| "A thread stack uses 8 MiB of RAM" | 8 MiB of *virtual* address space is reserved; only touched pages use physical RAM |
| "Stack size is unlimited" | Fixed for OS threads (typically 1–8 MiB); Go goroutines can grow up to 1 GiB by default |
| "Goroutines are just OS threads" | They're runtime-managed with tiny growable stacks, multiplexed onto OS threads |
| "Thread stacks are isolated from each other" | They share an address space; a stray pointer can corrupt another thread's stack |
| "The OS manages goroutine stacks" | The **Go runtime** does |
| "Stack overflow can always be caught with recover" | In Go it's a fatal error; in C it's a segfault |
| "More threads → more stack memory only when running" | Each thread reserves its stack for its whole life |

---

## 16. Exercises

### Exercise 1: Count the stacks
A process starts with its main thread, creates 5 more threads, and then 2 of them exit. How many thread stacks exist?

<details><summary>Solution</summary>

Started with 1 + 5 created = 6 stacks; two exit and their stacks are freed → **4** stacks.
</details>

### Exercise 2: Shared or per-thread?
For each: (a) the address of a global variable, (b) a local `int` inside a function, (c) an object allocated with `&T{}` that escapes, (d) the SP register, (e) the machine code of `main`.

<details><summary>Solution</summary>

(a) shared, (b) per-thread (in that thread's stack), (c) shared (heap), (d) per-thread (each thread has its own saved SP), (e) shared.
</details>

### Exercise 3: Budget
A server creates one OS thread per connection with 8 MiB stacks. How much virtual address space do 20,000 connections reserve? How much for goroutines at 2 KiB?

<details><summary>Solution</summary>

20,000 × 8 MiB = 160,000 MiB ≈ **156 GiB** (virtual). 20,000 × 2 KiB = 40,000 KiB ≈ **39 MiB**.
</details>

### Exercise 4: Explain the corruption
Two threads share one stack. Thread A calls `f`, thread B calls `g`, then A returns from `f`. What goes wrong?

<details><summary>Solution</summary>

`f`'s frame is not on top (g's is). A's return pops `g`'s frame and jumps to *g's* return address, corrupting B's call chain and running the wrong code with the wrong locals. Stacks only work for strictly nested calls of *one* flow.
</details>

### Exercise 5: Measure
Run the section 13 program. Change the recursion depth to 100,000 and to 10,000,000. Compare the printed stack sizes. What is the pattern?

<details><summary>Solution</summary>

Stack size roughly doubles as needed: ~32 MiB at 100,000 depth (≈ 20 MB needed), ~2 GiB *would* be needed at 10,000,000, exceeding the 1 GiB cap, so the program dies with `fatal error: stack overflow`. Sizes are powers of two (2 KiB × 2ⁿ).
</details>

### Exercise 6: Threads in your process
Write a Go program that prints the number of OS threads (`/proc/self/task` on Linux). Then start 100 goroutines that each call a blocking system call (e.g., `syscall.Nanosleep`) and print the count again. Explain the change.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"os"
	"sync"
	"syscall"
	"time"
)

func count() int {
	e, _ := os.ReadDir("/proc/self/task")
	return len(e)
}

func main() {
	fmt.Println("before:", count())
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			syscall.Nanosleep(&syscall.Timespec{Nsec: 300_000_000}, nil) // blocks its OS thread
		}()
	}
	time.Sleep(100 * time.Millisecond)
	fmt.Println("during:", count())
	wg.Wait()
}
```
Each goroutine blocked in a *raw* system call holds an OS thread, so the runtime starts extra threads to keep running other goroutines: the count jumps toward 100+. Goroutines blocked on channels/timers/network I/O don't need threads.
</details>

### Exercise 7 (challenge): Design a stack-growing check
Explain in your own words how a function prologue can decide whether to call `morestack`, and why the copy needs exact knowledge of where the pointers in each frame are.

<details><summary>Solution</summary>

The prologue compares SP (minus this function's frame size) with the stack's *limit* (`stackguard`) stored in the goroutine's `g` struct; if below it, jump to `morestack`. Copying a stack changes every address inside it, so any pointer that points *into* the stack must be rewritten to point into the new copy. That's only safe if the runtime knows exactly which words in each frame are pointers (the compiler emits "stack maps"), which is why Go can move stacks but C cannot.
</details>

---

## 17. Quiz

1. How many stacks does a process with 8 threads have?
2. Why can't two threads share a stack?
3. What's typically the default stack size for a Linux thread?
4. What is a guard page for?
5. Who allocates a goroutine's stack?
6. How does a goroutine's stack handle deep recursion?

<details><summary>Answers</summary>

1. Eight (one per thread).
2. Independent flows push and pop frames in an uncontrolled interleaving, corrupting return addresses and locals.
3. 8 MiB (virtual).
4. To turn a stack overrun into an immediate fault rather than silent corruption of neighboring memory.
5. The Go runtime (from the heap).
6. It grows: the runtime allocates a bigger stack, copies the frames, and fixes pointers, up to a 1 GiB limit.
</details>

---

## 18. Summary

- **One stack per thread.** A stack encodes one flow's nested call chain (return addresses, locals, SP, BP), so independent flows can't share one.
- All stacks live in the same **address space**; **code, globals, heap** are shared, while each thread has its own **SP, BP, PC, registers, and stack region**. A thread switch swaps those registers.
- OS thread stacks are **fixed-size** (Linux: 8 MiB virtual), with a **guard page** below. Physical memory is committed on use.
- Fixed stacks limit how many threads you can have, motivating **runtime-managed threads**.
- **Go goroutines** have **~2 KiB growable stacks** allocated by the runtime; on overflow the runtime **copies to a bigger stack**, so recursion is limited only by memory (1 GiB cap) and goroutines are cheap.
- The scheduler multiplexes many goroutine stacks onto a few OS threads, "one stack per flow" still holds.

### ➡️ What's next?

[Chapter 36](36-goroutines.md) finally brings it all together: **goroutines**, how to start them, how the Go scheduler runs them, and why the design is so powerful and beautiful.
