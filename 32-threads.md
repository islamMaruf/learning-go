# Chapter 32: Threads — The "Virtual Process"

> **Goal of this chapter:** Understand the **thread**: a lightweight unit of execution *inside* a process. You'll learn why threads exist (a server handling 100 users, a music player doing two jobs), exactly what threads **share** and what they keep **private**, why thread switches are cheaper than process switches, and what can go wrong. This is the direct stepping stone to **goroutines**.

**Difficulty:** 🔴 Intermediate–Advanced (conceptual)  **Estimated time:** 2.5–3 hours  **Prerequisite:** [Chapters 28, 30, 31](28-breaking-the-cpu-and-understanding-the-process.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Recap: what a process is](#2-recap-what-a-process-is)
3. [The problem: one process, one flow of execution](#3-the-problem-one-flow-of-execution)
4. [What is a thread?](#4-what-is-a-thread)
5. [What threads share and what they own](#5-what-threads-share-and-what-they-own)
6. [Example 1: the music player](#6-example-1-the-music-player)
7. [Example 2: a backend server and 100 users](#7-example-2-a-backend-server-and-100-users)
8. [How the CPU runs threads](#8-how-the-cpu-runs-threads)
9. [Thread switch vs. process switch](#9-thread-switch-vs-process-switch)
10. [Many processes, many threads](#10-many-processes-many-threads)
11. [The dangers: shared memory](#11-the-dangers-shared-memory)
12. [The cost of OS threads](#12-the-cost-of-os-threads)
13. [See real threads from Go](#13-see-real-threads-from-go)
14. [Where this leads: goroutines](#14-where-this-leads-goroutines)
15. [Process vs. thread cheat sheet](#15-process-vs-thread-cheat-sheet)
16. [Common misconceptions](#16-common-misconceptions)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. What you will learn

- What a **thread** is and how it differs from a **process**
- What threads **share** (code, data, heap, files) and what each **owns** (stack, registers, PC)
- Why servers and apps use threads
- Why a thread switch is cheaper than a process switch
- The main danger of threads: **shared memory → race conditions**
- Why OS threads are still "expensive", motivating Go's **goroutines**
- How to observe real threads (and thread IDs) from a Go program

---

## 2. Recap: what a process is

From Chapter 28: a **process** is a program in execution: a private memory space (code, data, heap, stack) plus CPU state, and one flow of execution, which is one **PC** stepping through instructions.

```
┌────────────────────────── PROCESS ──────────────────────────┐
│  Code | Data | Heap | Stack        ← memory                 │
│                                                             │
│  ONE execution flow: PC → SP → BP → registers               │
└─────────────────────────────────────────────────────────────┘
```

A classic process does **one thing at a time**, from `main`'s first line to its last.

---

## 3. The problem: one flow of execution

Real programs must do **several things at once**:

- A **music player** must keep playing audio *while* the UI responds to your clicks.
- A **web server** must serve **hundreds of users** at the same time.
- A **browser** downloads a file *while* you scroll a page.

Solution 1: **run multiple processes.** Works, but processes are **isolated** and **heavy**:

| Problem with using several processes | |
|--------------------------------------|--|
| Each has its own private memory, so sharing data (e.g., the current song's state) needs **inter-process communication** (pipes, sockets, shared-memory setup): awkward and slow | |
| Creating a process is **expensive** (new memory space, page tables, bookkeeping) | |
| Switching between processes is **expensive** (Chapter 30: swap address spaces, flush caches/TLB) | |
| Memory is **duplicated** (each process needs its own copy of everything) | |

We want something like several "workers" inside **one** program that can share its memory easily. That's the thread.

---

## 4. What is a thread?

> A **thread** is an independent **flow of execution *within* a process**. A process can contain **one or many** threads; all threads of a process **share its memory**, but each has its **own PC, registers, and stack**.

Nicknames you'll hear:
- "**Lightweight process**": it's cheaper than a process.
- "**Virtual process**" (the phrase this course uses): each thread behaves like its own little process (own instruction stream and stack), but *inside* a shared address space.

```
┌────────────────────────── ONE PROCESS ──────────────────────────┐
│                                                                 │
│   SHARED by all threads                                         │
│   ┌────────────┬────────────┬────────────┐                      │
│   │ Code       │ Data       │ Heap       │  + open files        │
│   └────────────┴────────────┴────────────┘                      │
│                                                                 │
│   PRIVATE to each thread                                        │
│   ┌──────────────────┐ ┌──────────────────┐ ┌────────────────┐  │
│   │ Thread 1         │ │ Thread 2         │ │ Thread 3       │  │
│   │  own STACK       │ │  own STACK       │ │  own STACK     │  │
│   │  own PC, SP, BP  │ │  own PC, SP, BP  │ │  own PC,SP,BP  │  │
│   │  own registers   │ │  own registers   │ │  own registers │  │
│   └──────────────────┘ └──────────────────┘ └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

Every process has at least **one thread**: the *main thread*. (When you ran your first Go program, you already had a process with several threads: Go's runtime starts extra ones for its scheduler and garbage collector.)

### A thread is the unit the OS schedules

Modern operating systems schedule **threads** onto cores (not whole processes). A process with one thread is the classic case; a process with 8 threads can use up to 8 cores at once. The kernel keeps a **Thread Control Block** (TCB; in Linux both processes and threads are `task_struct`s) analogous to the PCB.

---

## 5. What threads share and what they own

This table is the heart of the chapter.

| Resource | Shared by all threads of a process? | Why |
|----------|-------------------------------------|-----|
| **Code segment** | ✅ Shared | All threads run the same program; code is read-only |
| **Data segment** (package-level variables) | ✅ Shared | One copy of globals |
| **Heap** | ✅ Shared | Objects allocated by one thread are visible to all |
| **Open files / sockets** | ✅ Shared | Belong to the process |
| **Process ID (PID)** | ✅ Same | It's *one* process |
| **Stack** | ❌ **Private** | Each thread has its own call chain and local variables |
| **Registers (PC, SP, BP, general)** | ❌ **Private** | Each thread is at a different point of execution |
| **Thread ID (TID)** | ❌ Unique | The OS identifies each thread |
| **Thread-local storage, signal mask** | ❌ Private | Per-thread settings |

**Why must each thread have its own stack?** Because a stack records "who called whom and where to return". If two threads shared one stack, their frames would be interleaved and hopelessly corrupted. Thread 1 might be deep inside `handleLogin()` while Thread 2 is in `playAudio()`; they need different **return addresses, local variables, SP and BP** (Chapter 29). (This is exactly the next chapter's topic: *Separate Stack for Separate Thread*.)

**Why can they share heap and globals?** Because that's the whole point: threads cooperate by reading and writing the same data. It makes communication trivially fast (just memory), and it's also the source of the biggest danger (section 11).

```
Process memory with 3 threads:

  ┌─────────────────────────────┐
  │ Code            (shared)    │
  │ Data / globals  (shared)    │
  │ Heap            (shared)    │
  │ ─────────────────────────── │
  │ Stack of Thread 3  (own)    │  ← each thread's stack is a separate region
  │ Stack of Thread 2  (own)    │
  │ Stack of Thread 1  (own)    │
  └─────────────────────────────┘
```

---

## 6. Example 1: the music player

Imagine a music player app. It must do two jobs:

1. **Job A:** decode the file and send audio to the speakers, continuously.
2. **Job B:** listen for your clicks: pause, next track, volume.

With a **single flow of execution**, these compete:

```
loop:
    play a little audio
    check for a click      ← if the check is slow, the audio stutters
                             if audio is slow, the buttons feel frozen
```

With **two threads**:

```
Process: MusicPlayer
 ├── Thread 1: audio loop      (own stack: decoder state)
 └── Thread 2: UI/event loop   (own stack: handling clicks)

Shared memory: "isPlaying", "currentTrack", "volume"
```

- The audio thread never waits on the UI; the UI never waits on audio decoding.
- When you press *Pause*, the UI thread simply sets `isPlaying = false` in **shared memory**; the audio thread notices on its next loop. No message passing, no copying.

Both threads run "at the same time": on one core by **time-slicing** (Chapter 30), on two cores **truly in parallel** (Chapter 31).

---

## 7. Example 2: a backend server and 100 users

You're building a backend (like the e-commerce API we'll build in Part 8):

```
Client ── HTTP request ──► [ Server process ]
```

A user logs in: the server must (1) read the request, (2) query the database (**wait** ~10 ms), (3) build a response, (4) send it.

### With one flow of execution (sequential)

```
Time ──────────────────────────────────────────────────────────►
User 1: [work][ wait for DB ][work]
User 2:                              [work][ wait for DB ][work]
User 3:                                                          [work][wait][work]
```

If each request takes 200 ms and **100 users** arrive together, the last user waits **100 × 200 ms = 20 seconds**. The CPU is *idle* during every database wait. Terrible.

### With one thread per request

```
Time ──────────────────────────────────────────────────────────►
Thread 1: [work][ wait DB ][work]         ← user 1
Thread 2: [work][ wait DB ][work]         ← user 2
Thread 3: [work][ wait DB ][work]         ← user 3
...       (all overlapping)
```

While Thread 1 waits for the database, the OS runs Thread 2, 3, ... (or they run in parallel on other cores). **All 100 users are served in roughly the time of one**, not 100×.

### Comparison

| | Sequential | Thread per request |
|--|-----------|--------------------|
| 100 users, 200 ms each | ~20 s for the last user | ~200 ms for everyone (if resources allow) |
| CPU during DB waits | Idle | Busy with other requests |
| User experience | 😡 | 😊 |

This "one thread (or goroutine) per request" is exactly how Go's `net/http` server works (using goroutines instead of OS threads).

---

## 8. How the CPU runs threads

The CPU's control unit doesn't know about "threads". It only executes whatever instructions its **PC** points to, using its current registers. So how does one core run several threads?

**The OS does it, with context switching** (Chapter 30), one level down:

```
Core 1 timeline:
 ┌──────────┬──────────┬──────────┬──────────┬──────────┐
 │ Thread 1 │ Thread 2 │ Thread 1 │ Thread 3 │ Thread 2 │ ...
 └──────────┴──────────┴──────────┴──────────┴──────────┘
       ▲          ▲
       │          └── save Thread 1's PC/SP/BP/regs in its TCB;
       │              load Thread 2's saved PC/SP/BP/regs into the CPU
       └── each switch swaps ONLY registers/stack pointer state
```

The control unit is "blind": it keeps fetching from whatever PC currently holds. By **overwriting PC, SP, BP, and the other registers** with a thread's saved values, the OS makes the CPU continue *that thread's* code on *that thread's* stack. With multiple cores, each core independently runs some thread. The **OS scheduler** decides which thread goes on which core and for how long.

---

## 9. Thread switch vs. process switch

Recall from Chapter 30 what a **process** switch involves. A **thread** switch within the **same process** is cheaper:

| | Switch between **processes** | Switch between **threads of the same process** |
|--|------------------------------|-----------------------------------------------|
| Save/restore registers (PC, SP, BP, ...) | ✅ | ✅ |
| Switch memory mapping (address space, page tables) | ✅ | ❌ **Not needed**: same address space |
| Flush TLB (address-translation cache) | ✅ (often) | ❌ (usually not) |
| CPU caches still warm? | Mostly cold | Mostly warm (same code and data) |
| Typical cost | Higher | Lower |

```
Process switch:   [save regs] [swap address space] [flush TLB] [load regs]   ← slower
Thread switch:    [save regs]                                  [load regs]   ← faster
```

Because threads share the address space, the kernel has less to change. That's a big reason threads are called **lightweight**.

---

## 10. Many processes, many threads

A realistic computer, mid-afternoon:

```
Process 1: Browser         Threads: UI, network, renderer, JS engine, 20 more...
Process 2: Music player    Threads: audio, UI
Process 3: Your Go server  Threads: Go scheduler workers, GC helpers, blocking-syscall threads...
Process 4: Editor          Threads: UI, syntax highlighter, file watcher
...
Total: hundreds of processes, thousands of threads
```

The OS scheduler sees **all the threads** and time-slices them over the available cores. (VS Code, for example, is commonly shown by task managers with dozens of threads in a single process.)

Structure:

```
Computer
 └── Process (own memory)
      └── Thread (own stack + registers; shares the process's memory)
           └── (in Go) Goroutines, multiplexed onto threads (Chapter 36)
```

---

## 11. The dangers: shared memory

Sharing memory is both the **superpower** and the **curse** of threads. If two threads read *and write* the same variable at the same time without coordination, results become unpredictable: a **race condition**.

Classic example: two threads each do `counter++` a thousand times. `counter++` is really *three* CPU steps (load, add, store). Interleaved badly, one thread's update overwrites the other's:

```
counter = 5

Thread A: load counter (5)
Thread B: load counter (5)         ← reads the same old value
Thread A: add 1 → 6; store 6
Thread B: add 1 → 6; store 6       ← overwrites: one increment is LOST
counter = 6   (should be 7)
```

Race conditions are:
- **Non-deterministic**: they appear only sometimes, depending on timing.
- **Hard to reproduce and debug.**
- The reason concurrent programming has its reputation.

The tools to prevent them: **mutexes/locks**, **atomic operations**, and **message passing (channels)**. Go gives you all three, and Chapters 67–69 cover them in depth ("Do not communicate by sharing memory; instead, share memory by communicating").

Other thread hazards: **deadlock** (two threads waiting on each other forever), **starvation**, and complexity of reasoning.

---

## 12. The cost of OS threads

OS threads are lighter than processes, but they're **not free**:

| Cost | Typical value |
|------|---------------|
| **Stack size** | Linux default: **8 MB** of *virtual* address space per thread (physical memory is committed as used; often a few tens of KB up to MBs each) |
| **Creation time** | Tens of microseconds |
| **Switch time** | ~1–5 µs, plus cache effects |
| **Kernel resources** | A TCB, kernel stack, scheduler bookkeeping |
| **Practical limit** | Thousands to low tens of thousands per machine before things get painful |

Try imagining **100,000 simultaneous connections** with one OS thread each: 100,000 × 8 MB = **800 GB** of virtual stack space, plus enormous scheduling overhead. This is the famous **C10K problem**: handling ten thousand concurrent clients with thread-per-connection was hard. It pushed the world toward event loops (Node.js), async I/O, and, for Go, **goroutines**.

---

## 13. See real threads from Go

A Go program is already multi-threaded. Linux exposes thread counts in `/proc/self/status`, and `syscall.Gettid()` returns the calling thread's ID (Linux). Let's look:

```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"strings"
	"sync"
	"syscall"
	"time"
)

func threads() string {
	data, _ := os.ReadFile("/proc/self/status")
	for _, l := range strings.Split(string(data), "\n") {
		if strings.HasPrefix(l, "Threads") {
			return strings.Join(strings.Fields(l), " ")
		}
	}
	return "?"
}

func main() {
	fmt.Println("PID:", os.Getpid(), "main TID:", syscall.Gettid(), "|", threads())

	var wg sync.WaitGroup
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			runtime.LockOSThread() // pin this goroutine to its own OS thread
			defer runtime.UnlockOSThread()
			time.Sleep(50 * time.Millisecond)
			fmt.Printf("goroutine %d: PID=%d TID=%d\n", i, os.Getpid(), syscall.Gettid())
		}()
		time.Sleep(5 * time.Millisecond)
	}
	fmt.Println("during:", threads())
	wg.Wait()
}
```

Real output from one run (Linux):

```
PID: 2007151 main TID: 2007151 | Threads: 5
during: Threads: 7
goroutine 0: PID=2007151 TID=2007151
goroutine 1: PID=2007151 TID=2007154
goroutine 2: PID=2007151 TID=2007155
```

Read it carefully:

- **Same PID** (`2007151`) everywhere: all threads belong to **one process**.
- **Different TIDs** (`…151`, `…154`, `…155`): they're **different OS threads**. (On Linux the main thread's TID equals the PID.)
- A "hello world" Go process already had **5 threads** at start-up (the runtime's scheduler and helper threads), and pinning goroutines to threads raised it to 7.

`runtime.LockOSThread()` is rarely needed in ordinary code. We used it only to make goroutine ↔ thread relationships visible.

---

## 14. Where this leads: goroutines

OS threads solved *concurrency inside a process*, but with costs (section 12). Go's designers asked: *what if we made a much lighter "thread" managed by the language runtime instead of the kernel?* That's a **goroutine**:

| | OS thread | Goroutine |
|--|-----------|-----------|
| Managed by | The OS kernel | The **Go runtime** (user space) |
| Initial stack | ~1–8 MB (virtual) | **~2 KB** (grows and shrinks) |
| Switch cost | ~1–5 µs (kernel) | ~100–200 ns (no kernel) |
| Practical count | Thousands | **Hundreds of thousands to millions** |
| Created with | `pthread_create`, etc. | `go f()` |

Go multiplexes many goroutines onto a few OS threads (an **M:N** scheduler). Let's see it: launching **100,000 goroutines**, all parked waiting:

```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"strings"
	"sync"
	"time"
)

func threads() string {
	data, _ := os.ReadFile("/proc/self/status")
	for _, l := range strings.Split(string(data), "\n") {
		if strings.HasPrefix(l, "Threads") {
			return strings.Join(strings.Fields(l), " ")
		}
	}
	return "?"
}

func main() {
	var wg sync.WaitGroup
	var before, after runtime.MemStats
	runtime.ReadMemStats(&before)
	stop := make(chan struct{})

	start := time.Now()
	for i := 0; i < 100000; i++ {
		wg.Add(1)
		go func() { defer wg.Done(); <-stop }() // each goroutine just waits
	}
	time.Sleep(200 * time.Millisecond)
	runtime.ReadMemStats(&after)

	fmt.Printf("100000 goroutines alive: %s, stack memory ≈ %d MiB, launched in ≈ %v\n",
		threads(), (after.StackInuse-before.StackInuse)>>20, time.Since(start).Round(time.Millisecond))
	close(stop)
	wg.Wait()
}
```

Real output:

```
100000 goroutines alive: Threads: 54, stack memory ≈ 195 MiB, launched in ≈ 322ms
```

(The `Threads: 54` was left over from earlier experiments in the same process; on its own this program stays in single or low double digits, since the goroutines are all *parked*, not running.) **100,000 concurrent flows of execution for about 195 MiB of stack (~2 KB each)**. With OS threads at 8 MB virtual each, that would be 800 GB.

Two more real observations from the same experiments:

- 50 goroutines stuck in **blocking system calls** pushed the process to **54 OS threads**: when a goroutine blocks in the kernel, the runtime hands the OS thread to that syscall and starts another thread to keep running other goroutines. (That's the "M" in M:N.)
- Goroutines merely waiting on channels or timers **don't** need a thread each; they're just data structures in the scheduler.

You'll dissect this machinery in Chapters 35, 36, and 39.

---

## 15. Process vs. thread cheat sheet

| | **Process** | **Thread** |
|--|-------------|-----------|
| Definition | Program in execution with its own address space | A flow of execution inside a process |
| Memory | **Private** (isolated) | **Shared** with its process's other threads (except its own stack) |
| Owns | Address space, files, PID | Stack, registers (PC, SP, BP), TID |
| Creation cost | High | Lower |
| Switch cost | High | Lower |
| Communication | IPC (pipes, sockets, shared memory): slower, safer | Just memory: fast, but needs synchronization |
| A crash | Usually only affects that process | Can bring down the **whole process** |
| Protection | Strong (OS-enforced isolation) | None among threads of one process |
| Example | Chrome's separate tab processes | The many threads inside one server |

---

## 16. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "A thread is a small process" | It lives *inside* a process and shares its memory; a process has its own isolated memory |
| "Threads have separate memory" | They share everything except their own stack and registers |
| "Threads always run in parallel" | Only if cores are available; otherwise they're time-sliced (concurrent, not parallel) |
| "Each thread has its own copy of global variables" | Globals are shared, which is why races happen |
| "A goroutine is an OS thread" | It's a much lighter unit multiplexed *onto* OS threads |
| "More threads = faster" | Beyond the core count, CPU-bound work gets no faster and switching overhead grows |
| "Threads are free" | Each costs stack memory, kernel structures, and switch time |
| "If one thread crashes, the others continue" | An unhandled crash (e.g., a segfault, or an unrecovered panic in Go) takes down the whole process |

---

## 17. Exercises

### Exercise 1: Shared or private?
For each, say shared or private among threads of one process: (a) a package-level variable, (b) a local variable in a function, (c) the program counter, (d) a heap-allocated struct, (e) an open file.

<details><summary>Solution</summary>

(a) shared, (b) private (lives on that thread's stack), (c) private, (d) shared, (e) shared.
</details>

### Exercise 2: Why own stacks?
Explain in two sentences why two threads cannot share one stack.

<details><summary>Solution</summary>

A stack records the chain of function calls, local variables, and return addresses for one flow of execution. Two independent flows would push and pop frames in interleaved, conflicting ways, corrupting each other's return addresses and locals. Each thread therefore needs its own stack (and SP/BP).
</details>

### Exercise 3: Time it
A server handles requests that each spend 5 ms computing and 195 ms waiting on a database. How long for 100 requests (a) sequentially, (b) with 100 threads on a 4-core machine (assume waits overlap perfectly)? What limits (b)?

<details><summary>Solution</summary>

(a) 100 × 200 ms = 20 s. (b) The 195 ms waits overlap entirely; the CPU work (100 × 5 ms = 500 ms of CPU) is spread across 4 cores ≈ 125 ms. So roughly **~320 ms** total. What limits it: the CPU portion (cores) and the database's ability to serve concurrent queries.
</details>

### Exercise 4: Race
Trace two threads each running `x = x + 1` on shared `x = 0` with a bad interleaving that ends with `x == 1`.

<details><summary>Solution</summary>

A loads x (0). B loads x (0). A computes 1, stores x = 1. B computes 1, stores x = 1. Final x = 1, not 2 (one update lost).
</details>

### Exercise 5: Real threads
Run the section 13 program. What differs between the goroutines' TIDs and PIDs? Change `LockOSThread` usage (remove it), and see whether you still get different TIDs. Why might they vary?

<details><summary>Solution</summary>

PIDs are identical (one process); TIDs differ when goroutines run on different OS threads. Without `LockOSThread`, the Go scheduler may run several goroutines on the *same* OS thread over time (or move them), so TIDs may repeat or change. The runtime decides which thread runs which goroutine; you shouldn't care.
</details>

### Exercise 6: Memory budgeting
How much virtual stack memory would 50,000 OS threads with 8 MB stacks reserve? And 50,000 goroutines with ~2 KB stacks?

<details><summary>Solution</summary>

50,000 × 8 MB = 400,000 MB ≈ **390 GB** (virtual); 50,000 × 2 KB = 100,000 KB ≈ **98 MB**.
</details>

### Exercise 7 (challenge): Design
Design a chat server's threading model: many clients connected, messages must be delivered to everyone in a room. Which data is shared between threads? What must be protected? (You'll implement this pattern with goroutines and channels later.)

<details><summary>Solution</summary>

One thread/goroutine per connection (reading that client's messages) is natural. Shared data: the list of clients per room, and any message history. That shared structure must be protected (a mutex) or, better, owned by a *single* goroutine that others talk to via channels (Chapter 69), so only one flow ever mutates it.
</details>

---

## 18. Quiz

1. What does a thread own privately? What does it share?
2. Why is a thread switch cheaper than a process switch?
3. What is a race condition?
4. Why does Linux give each OS thread its own stack?
5. What is a rough per-thread stack reservation on Linux, and why is that a problem at scale?
6. How is a goroutine different from an OS thread?

<details><summary>Answers</summary>

1. Owns: stack, registers (PC, SP, BP…), TID. Shares: code, globals, heap, open files with the rest of its process.
2. No address-space switch/TLB flush is needed since threads share memory.
3. A bug where the result depends on the timing of unsynchronized access to shared data.
4. Each thread has its own call chain, locals, and return addresses.
5. ~8 MB virtual; 10,000+ threads consume huge address space and kernel resources (the C10K problem).
6. A goroutine is scheduled by the Go runtime, has a ~2 KB growable stack, and switches in ~100s of ns; OS threads are kernel-scheduled with MB stacks and µs switches.
</details>

---

## 19. Summary

- A **thread** is a flow of execution *inside* a process. Every process has at least one; many have dozens.
- Threads **share** code, globals, heap, and open files; each thread has its **own stack, registers (PC/SP/BP), and TID**.
- Threads make **servers and apps responsive** (overlap waiting, do several jobs) with cheap communication via shared memory.
- The OS **schedules threads** onto cores. A thread switch is **cheaper than a process switch** (no address-space change).
- Shared memory is dangerous: **race conditions**, deadlocks. Locks, atomics, and channels are the tools.
- OS threads still cost **MBs of stack** and µs per switch, which can't scale to 100,000s of concurrent tasks.
- Go answers with **goroutines**: user-space threads with ~2 KB stacks and ~150 ns switches, multiplexed onto a small number of OS threads. That's the road to Chapters 33–36 and Part 14.

### ➡️ What's next?

Part 6 begins: **advanced Go concepts** built on this foundation. [Chapter 33](33-data-types-in-depth.md) starts with how Go stores data of different types in memory ("bogus data types"), and then the `defer` statement, per-thread stacks, and finally goroutines themselves.
