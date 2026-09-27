# Chapter 36: Goroutines — Complex and Beautiful

> **Goal of this chapter:** Answer the most-asked Go interview question, **"What is a goroutine?"**, *properly*: not "a lightweight thread" (which is the answer everyone gives) but with real understanding of *why* it's lightweight, how the **Go scheduler (G, M, P)** runs millions of them on a handful of OS threads, and how to use them safely. You'll launch goroutines, watch the scheduler's behavior, and measure the costs.

**Difficulty:** 🔴 Advanced  **Estimated time:** 3–4 hours  **Prerequisite:** [Chapters 28–35](28-breaking-the-cpu-and-understanding-the-process.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The wrong answers (and the right one)](#2-the-answers)
3. [The idea of "virtual"](#3-the-idea-of-virtual)
4. [The hierarchy](#4-the-hierarchy-from-computer-to-goroutine)
5. [Your first goroutine](#5-your-first-goroutine)
6. [When `main` ends, everything ends](#6-when-main-ends-everything-ends)
7. [Goroutines run in no guaranteed order](#7-no-guaranteed-order)
8. [Goroutine vs. OS thread: the numbers](#8-goroutine-vs-os-thread)
9. [How the scheduler works: G, M, P](#9-how-the-scheduler-works)
10. [What happens when a goroutine blocks](#10-what-happens-when-a-goroutine-blocks)
11. [Preemption](#11-preemption)
12. [Stack growth (recap)](#12-stack-growth)
13. [Watching the scheduler](#13-watching-the-scheduler)
14. [Why goroutines are beautiful](#14-why-goroutines-are-beautiful)
15. [Limits and costs](#15-limits-and-costs)
16. [Pitfalls](#16-pitfalls)
17. [Interview answer script](#17-interview-answer-script)
18. [Common misconceptions](#18-common-misconceptions)
19. [Exercises](#19-exercises)
20. [Quiz](#20-quiz)
21. [Summary](#21-summary)

---

## 1. What you will learn

- A precise, layered definition of a goroutine
- How to start goroutines with `go`, and the rules of their lifetime
- Why their order is **non-deterministic**
- The **GMP scheduler**: **G**oroutines, **M**achine threads, **P**rocessors
- What happens when a goroutine blocks on I/O, a channel, or a system call
- **Measured** numbers: memory per goroutine, creation cost, how many you can run
- Pitfalls (leaks, unbounded spawning, panics) and how to answer the interview question

---

## 2. The answers

**Interviewer:** *"What is a goroutine?"*

| Typical answer | Problem |
|----------------|---------|
| "A lightweight thread." | True but shallow: *lightweight compared to what, and why?* |
| "A function that runs concurrently." | Describes usage, not what it *is* |
| "Go's concurrency primitive." | A label, not an explanation |

**A good answer (we'll build it piece by piece):**

> A goroutine is a **lightweight, user-space thread of execution, created and scheduled by the Go runtime rather than the operating system**. It has its own small (~2 KiB), growable stack; the runtime **multiplexes** many goroutines onto a small number of OS threads (the **M:N** model), switching between them in user space for ~100 ns instead of asking the kernel. That's why a Go program can run hundreds of thousands of goroutines.

Each part of that answer maps to a section below.

---

## 3. The idea of "virtual"

The word **virtual** in computing means: *something that behaves like the real thing but is created and managed by software on top of something else.*

| Example | Real thing | "Virtual" version |
|---------|-----------|-------------------|
| Virtual machine | A physical computer | Software pretending to be a computer, running on a real one |
| Virtual memory | Physical RAM | Each process *believes* it owns a big private address space; the OS maps it |
| Virtual account | Actual cash | Your bank balance shows a number; the bank doesn't keep your money in a drawer |
| **Goroutine** | An OS thread | A thread-like flow of execution that the Go runtime creates and manages |

### The bank analogy 💰

Your bank app says you have ₹/$10,000. The bank doesn't keep exactly that in a box labeled with your name: it lent most of it out. But *logically*, it's yours, and you can spend it. It's a **virtual** balance, backed by a smaller amount of real cash and a set of rules.

Goroutines are similar: you can have **100,000 "threads"** (virtual), but only ~10 real OS threads exist underneath. Logically each goroutine has its own flow of execution; physically they share the few real threads.

---

## 4. The hierarchy: from computer to goroutine

Each level below *virtualizes* the one above it: gives the illusion of many smaller "machines" inside one bigger one.

```
Physical computer
  └── Virtual machines / containers        (hypervisor / container runtime)
        └── Operating system
              └── Processes                (each thinks it owns the machine's memory)
                    └── Threads            (each thinks it owns a CPU)
                          └── Goroutines   (each thinks it owns a thread)
```

Compare pairs:

| Pair | The "virtual" one | What it really shares |
|------|-------------------|-----------------------|
| Computer → **process** | A process behaves like a private mini-computer | The real CPU and RAM (OS divides them) |
| Process → **thread** | A thread behaves like a private mini-process | The process's memory (code, data, heap) |
| Thread → **goroutine** | A goroutine behaves like a private mini-thread | The OS threads underneath it |

### The Facebook analogy 📱

You have **one** Facebook account. In it you have conversations with hundreds of people. Each conversation *looks* like a separate thing: its own messages and its own state. But behind the scenes it's all inside one account, one app, one login.

- The **account** ≈ a process (or an OS thread).
- Each **conversation** ≈ a goroutine: an independent thread of activity that is *virtual* (it exists in software inside the account).

Nobody creates a new Facebook *account* for each conversation. That would be as wasteful as creating an OS thread for each task.

---

## 5. Your first goroutine

Put the keyword **`go`** in front of a function call, and that call runs **concurrently** in a new goroutine.

```go
package main

import (
	"fmt"
	"time"
)

func sayHello() {
	fmt.Println("Hello from a goroutine!")
}

func main() {
	go sayHello() // start it and DON'T wait: main continues immediately

	fmt.Println("Hello from main")
	time.Sleep(100 * time.Millisecond) // crude wait so we can see the goroutine's output
}
```

Possible output:

```
Hello from main
Hello from a goroutine!
```

**What `go f(x)` does:**

1. Evaluates `f` and its **arguments** *now* (in the calling goroutine).
2. Creates a new goroutine (a `g` struct with a ~2 KiB stack) and puts it on a run queue.
3. **Returns immediately**: the caller does not wait.
4. `f` runs, eventually, whenever the scheduler picks it.

Anonymous functions are the usual way to start goroutines (Chapter 15):

```go
go func() {
	fmt.Println("anonymous goroutine")
}()

go func(msg string) {
	fmt.Println(msg)
}("passed as an argument")
```

**Return values are discarded**: `go f()` can't hand results back (use channels, Chapter 69, or shared variables protected by a mutex, Chapter 68).

---

## 6. When `main` ends, everything ends

The program exits when **`main` returns**, *even if other goroutines are still running*. They're killed on the spot.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	go func() {
		time.Sleep(50 * time.Millisecond)
		fmt.Println("this line never prints")
	}()

	fmt.Println("main is done")
	// main returns → process exits → the goroutine dies mid-sleep
}
```

Output: only `main is done`.

So you need a way to **wait** for goroutines. `time.Sleep` (as in our first example) is a hack: you'd have to guess how long the goroutine takes. The right tools:

| Tool | Chapter |
|------|---------|
| `sync.WaitGroup`: wait for N goroutines to finish | 65–66 |
| Channels: receive a "done" signal or a result | 69–70 |
| `context.Context`: cancellation and deadlines | later/advanced |

A quick preview of the correct way:

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println("worker", i)
		}()
	}

	wg.Wait() // block until all three have called Done
	fmt.Println("all done")
}
```

---

## 7. No guaranteed order

Goroutines run **concurrently**, and the scheduler decides when. There is **no guaranteed order**.

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Print("g", i, " ")
		}()
	}
	wg.Wait()
	fmt.Println()
}
```

A real run printed:

```
g5 g1 g2 g3 g4
```

Notice **`g5` came first**, even though it was started last. That's a real scheduler behavior: when a goroutine creates another, the new one is placed in the current processor's special **`runnext`** slot, so the *most recently created* goroutine tends to run first, followed by the others in queue order. Run it several times, or on another machine: you'll see different orders. **Never write code that depends on goroutine order.** If order matters, synchronize explicitly.

(This program uses Go 1.22 loop-variable semantics, so each goroutine sees its own `i`; see Chapter 20.)

---

## 8. Goroutine vs. OS thread

| | **OS thread** | **Goroutine** |
|--|---------------|---------------|
| Created & managed by | OS kernel | Go runtime (user space) |
| Initial stack | 1–8 MiB reserved (virtual) | **2 KiB** (grows/shrinks) |
| Stack growth | Fixed size | Automatic (copying) |
| Creation cost | ~10–50 µs (syscall) | **~1 µs or less** |
| Context-switch cost | ~1–5 µs (kernel, cache effects) | **~100–200 ns** (user space) |
| Practical limit | thousands–tens of thousands | **hundreds of thousands to millions** |
| Identified by | TID (kernel) | Internal goroutine ID (not exposed on purpose) |
| Scheduling | Preemptive, by kernel | Cooperative + async preemption, by Go scheduler |

### Measured on the test machine

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {
	var m0, m1 runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&m0)

	var wg sync.WaitGroup
	stop := make(chan struct{})
	const N = 1_000_000

	start := time.Now()
	for i := 0; i < N; i++ {
		wg.Add(1)
		go func() { defer wg.Done(); <-stop }() // park forever (until we close stop)
	}
	created := time.Since(start)
	time.Sleep(300 * time.Millisecond)
	runtime.ReadMemStats(&m1)

	fmt.Println("goroutines alive:", runtime.NumGoroutine())
	fmt.Printf("created %d in %v (%v each)\n", N, created.Round(time.Millisecond), created/N)
	fmt.Printf("stack memory: %d MiB (%d bytes each)\n", (m1.StackInuse-m0.StackInuse)>>20, (m1.StackInuse-m0.StackInuse)/N)

	close(stop)
	wg.Wait()
}
```

Real output:

```
goroutines alive: 1000001
created 1000000 in 1.363s (1.363µs each)
stack memory: 1953 MiB (2048 bytes each)
```

**One million concurrent goroutines**, each with a **2,048-byte** stack, created at **about 1.4 µs each**, for about **2 GiB** of memory in total. With OS threads at 8 MiB virtual each, this would be **8 TiB** of address space and would be refused by the kernel long before. (For the same reason, real programs rarely need millions; the demonstration shows the *design headroom*.)

---

## 9. How the scheduler works

The Go runtime includes its own scheduler. It uses three kinds of objects, the **GMP model**:

| Letter | Name | What it is |
|--------|------|------------|
| **G** | **Goroutine** | The unit of work: a stack + saved registers (PC, SP, BP) + status |
| **M** | **Machine** | An **OS thread** (managed by the kernel) that executes Go code |
| **P** | **Processor** | A *scheduling context*: a **run queue** of goroutines and a memory-allocator cache. An M needs a P to run Go code |

The number of **P**s is `GOMAXPROCS` (default = number of logical CPUs, 12 on the test machine). So at most `GOMAXPROCS` goroutines run **in parallel**. Exactly matching the number of cores.

```
                        ┌──────────── Go runtime ─────────────┐
   Global run queue:    │  [G9][G10][G11] ...                 │
                        │                                     │
   Local run queues     │   P0: [G1][G2][G3]     P1: [G4][G5] │   ← each P has its own queue
                        │    │                    │           │
                        │    M0 (thread)          M1 (thread) │   ← each running P is attached to an M
                        └────┼────────────────────┼───────────┘
                             ▼                    ▼
                          CPU core 0           CPU core 1        ← kernel schedules the Ms
```

### The loop each M runs

```
loop:
    pick a goroutine G:
        1. check this P's local queue  (fast, no lock)
        2. check the global queue      (occasionally)
        3. check the network poller    (goroutines whose I/O is ready)
        4. STEAL half the goroutines from another P's queue   ← work stealing
    run G (switch: load G's SP/PC/regs)   ← ~100–200 ns
    when G blocks, finishes, or is preempted → back to loop
```

### Work stealing

If a P runs out of work while others have long queues, it **steals** half of another P's queue. This automatically balances load across cores without any programmer effort.

### Why it's fast

- **Local run queues** need no locks in the common case.
- Switching goroutines means saving/restoring **only a few registers** in user space; **no kernel entry, no address-space change**, and the caches stay warm.
- New goroutines are made from the same thread that requested them, often running on the same core soon after (good cache locality, hence that `runnext` behavior we saw).

---

## 10. What happens when a goroutine blocks

Blocking is where the scheduler earns its keep. It handles each kind differently.

### 1. Blocking on a channel, mutex, timer, or `WaitGroup`

The goroutine is **parked**: marked *waiting* and removed from the run queue. **The OS thread does not block**; it immediately picks another goroutine. When the awaited event happens (a send arrives, the lock is released), the runtime makes the goroutine **runnable** again.

```
G1: <-ch   (nothing there yet)   → G1 parked, M continues with G2
G3: ch <- 42                     → G1 made runnable, put on a run queue
```

Cost: a few hundred nanoseconds; no kernel involvement.

### 2. Network I/O

Go uses the OS's efficient event notification (`epoll` on Linux, `kqueue` on macOS, IOCP on Windows) via the **netpoller**. A goroutine reading from a socket that has no data is **parked**, and the netpoller wakes it when data arrives. Thousands of connections cost thousands of parked goroutines, **not** thousands of blocked threads. This is the secret behind Go servers.

### 3. Blocking system calls (file I/O, some syscalls, cgo)

These *do* block an OS thread in the kernel. The runtime handles it: when a goroutine enters a blocking syscall, its **P is detached from that M** and handed to another M (starting a new thread if necessary) so the other goroutines keep running. When the syscall returns, the goroutine tries to reacquire a P.

This is why (as we measured in Chapter 32), 50 goroutines stuck in blocking syscalls raised the process from 5 to 54 OS threads:

```
Before syscall:   M0 ─ P0 ─ [G1 running] [G2 G3 queued]
G1 enters a blocking syscall:
                  M0 (blocked with G1 in the kernel)
                  P0 handed off to M1 ─ [G2 running] [G3 queued]   ← other goroutines continue
```

Summary of the three cases:

| Blocks on | Thread blocked? | Mechanism |
|-----------|-----------------|-----------|
| channel / mutex / timer / WaitGroup | ❌ No | goroutine parked; thread runs others |
| network I/O | ❌ No | netpoller (`epoll`) + parking |
| blocking syscall / cgo | ✅ Yes, but its P is handed off | extra OS threads created as needed |

---

## 11. Preemption

What if a goroutine runs a long loop and never blocks?

```go
go func() {
	for { } // never yields
}()
```

Before Go 1.14, such a goroutine could **starve** others on its P (the scheduler switched only at function calls and blocking points, i.e., **cooperatively**). Since **Go 1.14**, the runtime also uses **asynchronous preemption**: a background monitor (`sysmon`) notices a goroutine running for more than ~10 ms and sends its thread a signal, which interrupts it and lets the scheduler run something else.

So Go schedulers are effectively **cooperative at safe points plus signal-based preemption**: fair to well-behaved code, and robust against runaway loops.

You can also yield voluntarily: `runtime.Gosched()`.

---

## 12. Stack growth

Recall Chapter 35: each goroutine's stack **starts at ~2 KiB and grows** by copying to a bigger stack when a function prologue detects insufficient space (and may shrink during GC). That's why 1,000,000 goroutines fit in ~2 GiB and why recursion inside a goroutine isn't limited by an 8 MiB thread stack.

---

## 13. Watching the scheduler

The runtime can trace its own scheduler. Set `GODEBUG=schedtrace=200` (print every 200 ms) when running any program:

```bash
GODEBUG=schedtrace=200 go run .
```

A line looks like this (real output, from the 1,000,000-goroutine program's start-up):

```
SCHED 0ms: gomaxprocs=12 idleprocs=9 threads=5 spinningthreads=1 needspinning=0 idlethreads=0 runqueue=0 [ 1 0 0 0 0 0 0 0 0 0 0 0 ] schedticks=[ ... ]
```

| Field | Meaning |
|-------|---------|
| `gomaxprocs=12` | Number of **P**s |
| `idleprocs=9` | Ps with nothing to do |
| `threads=5` | Number of **M**s (OS threads) created so far |
| `spinningthreads` | Threads actively looking for work (spinning briefly before sleeping) |
| `runqueue=0` | Length of the **global** run queue |
| `[ 1 0 0 ... ]` | Length of each **P's local** run queue |

Add `scheddetail=1` for per-G/M/P details. And:

```go
runtime.NumGoroutine()  // how many goroutines exist right now
runtime.NumCPU()        // logical CPUs
runtime.GOMAXPROCS(0)   // current number of Ps (0 = query only)
```

---

## 14. Why goroutines are beautiful

**1. Simplicity.** One keyword:

```go
go handleRequest(conn)
```

No thread pools to size, no callbacks to chain, no `async`/`await` "coloring" of functions. You write ordinary sequential, blocking code inside each goroutine, and the runtime handles the rest.

**2. Efficiency.** 2 KiB stacks, 100 ns switches, no kernel involvement for most scheduling.

**3. Scalability.** Servers can hold hundreds of thousands of concurrent connections, one goroutine per connection, in a straightforward style. (Go's `net/http` server starts a goroutine per request.)

**4. Developer experience.** Concurrency becomes *structure* (which goroutines exist and how they communicate) rather than *plumbing* (threads, locks, event loops). Go's motto: **"Don't communicate by sharing memory; share memory by communicating."** Channels (Chapter 69) implement it.

---

## 15. Limits and costs

Goroutines are *cheap*, not *free*:

| Resource | Cost / limit |
|----------|--------------|
| **Memory** | ≥ 2 KiB each (+ whatever the goroutine allocates or its stack grows to). One million ≈ 2 GiB minimum |
| **Creation** | ~1 µs: fine, but not for tiny work in the hottest loops |
| **Scheduling overhead** | Thousands of runnable goroutines contending is fine; but **more runnable goroutines than cores doesn't speed up CPU-bound work** |
| **Coordination** | Locks and channels cost time; shared data needs protection |
| **Garbage collector** | Each goroutine's stack is a GC root scanned every cycle; a million goroutines lengthens GC |
| **OS limits** | Each *blocking syscall* holds an OS thread; the process is capped by `ulimit -u` and memory |

Rules of thumb:
- **I/O-bound work:** goroutines per connection/request are perfect.
- **CPU-bound work:** about **one worker per core** (`runtime.NumCPU()`); more just adds overhead.
- **Bound your concurrency** when the number of tasks is unbounded (e.g., limit to 100 workers with a semaphore or worker pool), or you risk exhausting memory or downstream services.

---

## 16. Pitfalls

### Pitfall 1: forgetting to wait

```go
go doWork()
// main returns immediately → doWork is killed
```

### Pitfall 2: goroutine leaks

A goroutine blocked forever on a channel nobody will use never exits, holding its memory and everything it references:

```go
func leak() {
	ch := make(chan int)
	go func() { <-ch }() // waits forever: nobody sends
}
```

Detect with `runtime.NumGoroutine()` growing over time, or `pprof`'s goroutine profile. Prevent with contexts/cancellation and by always making sure every goroutine has a way to finish.

### Pitfall 3: a panic in any goroutine crashes the whole program

```go
go func() { panic("boom") }() // unrecovered → the entire process exits
```

A `recover()` in `main` does **not** catch panics from other goroutines. Each goroutine you start needs its own `defer recover` if it must be resilient (Chapter 34).

### Pitfall 4: data races

Two goroutines touching the same variable, with at least one writing, without synchronization → undefined behavior. Detect with:

```bash
go run -race .
go test -race ./...
```

(Chapters 67–68 explain and fix them.)

### Pitfall 5: loop variable capture (Go ≤ 1.21)

```go
for i := 0; i < 3; i++ {
	go func() { fmt.Println(i) }() // Go ≤ 1.21: usually prints 3 3 3
}
```

Go 1.22+ gives each iteration its own `i`; older versions need `go func(i int) {...}(i)`. (Chapter 20.)

### Pitfall 6: unbounded goroutine creation

```go
for _, item := range millionItems {
	go process(item) // a million goroutines + a million open files/connections? 
}
```

Use a worker pool (Chapter 70).

### Pitfall 7: assuming goroutines start immediately or in order

They start when scheduled (section 7).

---

## 17. Interview answer script

**Q: "What is a goroutine?"**

> "A goroutine is a lightweight thread of execution that's created and managed by the **Go runtime**, not the operating system. You start one with the `go` keyword. It starts with a tiny stack of about **2 KB** that **grows and shrinks** as needed, unlike an OS thread's fixed 1–8 MB stack.
>
> The runtime's scheduler multiplexes many goroutines onto a small number of OS threads: the **M:N model**, using three concepts: **G** (goroutine), **M** (OS thread) and **P** (processor: a logical CPU with a run queue). There are `GOMAXPROCS` Ps, so that many goroutines can run truly in parallel.
>
> Switching between goroutines happens in **user space**, without a system call: saving and restoring just the PC, SP and a few registers: roughly **100–200 nanoseconds** versus microseconds for a thread switch. When a goroutine blocks on a channel, a mutex, a timer, or network I/O, it's just **parked** and the thread runs another goroutine. For blocking system calls, the runtime hands the P to another thread. Idle Ps **steal work** from busy ones.
>
> Because of this, a Go program can comfortably run hundreds of thousands of goroutines, which is why one-goroutine-per-connection servers are natural in Go."

**Follow-ups to be ready for:**

| Question | Answer |
|----------|--------|
| Goroutine vs thread? | The table in section 8 |
| What's GOMAXPROCS? | Number of Ps: max goroutines executing Go code in parallel; default = logical CPUs |
| What happens when `main` exits? | The program ends; other goroutines are killed |
| How do goroutines communicate? | Channels (preferred), or shared memory with mutexes/atomics |
| What is work stealing? | An idle P takes half of another P's local run queue |
| How do you avoid leaks? | Every goroutine needs an exit path: channel close, `context` cancellation, timeouts |
| What's a data race and how do you find it? | Unsynchronized concurrent access with a write; `-race` detector |

**Avoid saying:** "goroutines are threads" (they aren't), "goroutines are free", or "Go has no threads" (it does; it just manages them for you).

---

## 18. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "A goroutine is an OS thread" | It's multiplexed onto OS threads by the Go scheduler |
| "More goroutines is always faster" | Not for CPU-bound work beyond core count |
| "Goroutines run in the order started" | No guarantee (and often the *last* one starts first) |
| "`go f()` waits for `f`" | It returns immediately |
| "Goroutines are killed when their function's caller returns" | They live until their function returns; only `main` returning ends everything |
| "A blocked goroutine blocks an OS thread" | Usually not (channels, network); only blocking syscalls do |
| "Goroutines share nothing" | They share the whole heap and package variables, hence data races |
| "`recover` in main protects all goroutines" | Panics are per-goroutine; unrecovered ones crash the program |
| "Go has one thread per goroutine" | Typically a handful of threads for thousands of goroutines |
| "Goroutines can't be leaked" | They can, and leaks are a common production bug |

---

## 19. Exercises

### Exercise 1: Hello, goroutines
Start three goroutines that each print their number, and use a `sync.WaitGroup` to wait for them.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println("goroutine", i)
		}()
	}
	wg.Wait()
}
```
</details>

### Exercise 2: Prove they're concurrent
Start two goroutines that each sleep 200 ms, then print. Measure total time with `time.Since`. Is it ~200 ms or ~400 ms? Why?

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	start := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < 2; i++ {
		wg.Add(1)
		go func() { defer wg.Done(); time.Sleep(200 * time.Millisecond) }()
	}
	wg.Wait()
	fmt.Println(time.Since(start).Round(50 * time.Millisecond)) // ~200ms
}
```
About 200 ms: the sleeps overlap, since sleeping goroutines are parked and cost no thread.
</details>

### Exercise 3: Count them
Print `runtime.NumGoroutine()` at start, after launching 100 goroutines that block on a channel, and after closing the channel and waiting.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	fmt.Println("start:", runtime.NumGoroutine()) // 1
	stop := make(chan struct{})
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() { defer wg.Done(); <-stop }()
	}
	fmt.Println("running:", runtime.NumGoroutine()) // 101
	close(stop)
	wg.Wait()
	fmt.Println("after:", runtime.NumGoroutine()) // 1 (the goroutines may take an instant to fully exit)
}
```
</details>

### Exercise 4: Order experiment
Run the section 7 program 10 times. Do you always see the same order? Try `GOMAXPROCS=1 go run .`. What changes?

<details><summary>Solution</summary>

The order varies between runs with several Ps. With `GOMAXPROCS=1` the order is far more repeatable (typically the last created first, then the rest in order), since a single P's run queue is deterministic when no stealing occurs. It's still *not guaranteed* by the language.
</details>

### Exercise 5: Find the leak
What's wrong with this function, and how would you fix it?

```go
func first(urls []string) string {
	ch := make(chan string)
	for _, u := range urls {
		go func() { ch <- fetch(u) }()
	}
	return <-ch // take the first result
}
```

<details><summary>Solution</summary>

Only one goroutine's send is received; the others block forever on `ch <- ...` (an unbuffered channel with no receiver) → leaked goroutines. Fixes: buffer the channel (`make(chan string, len(urls))`) so all sends complete, or use `context` cancellation so slow fetchers stop early.
</details>

### Exercise 6: CPU-bound worker sizing
You have to hash 10,000 files on an 8-core machine. How many goroutines would you run for the hashing, and how would you limit them?

<details><summary>Solution</summary>

Hashing is CPU-bound (plus some I/O), so about `runtime.NumCPU()` (8) workers reading paths from a channel (a worker pool), not 10,000 goroutines: extra ones wouldn't speed up compute and would cost memory and open file handles. (Chapter 70.)
</details>

### Exercise 7 (challenge): Explain `runnext`
In section 7's run, why might the last-created goroutine run first? Predict what you'd see with `GOMAXPROCS=1`.

<details><summary>Solution</summary>

Newly created goroutines are placed in the creating P's `runnext` slot (to preserve locality); each new one *kicks the previous `runnext` goroutine into the regular queue*. After the loop, `main` blocks in `wg.Wait()`, so the P runs `runnext` first (the last goroutine created: `g5`), then the queue in FIFO order (`g1 g2 g3 g4`). With one P you'll see `g5 g1 g2 g3 g4` almost every time; with multiple Ps, idle Ps steal goroutines and the order varies.
</details>

---

## 20. Quiz

1. What does the `go` keyword do?
2. What are G, M, and P?
3. How big is a new goroutine's stack?
4. What happens to running goroutines when `main` returns?
5. What does the scheduler do when a goroutine blocks on a channel?
6. What does `GOMAXPROCS` control?
7. Name two ways goroutines can leak or crash a program.

<details><summary>Answers</summary>

1. Starts the function call in a new goroutine and returns immediately.
2. G = goroutine, M = OS thread, P = logical processor with a run queue (needed to run Go code).
3. About 2 KiB, growable.
4. They're killed; the process exits.
5. Parks it and runs another goroutine on the same thread.
6. The number of Ps: how many goroutines can execute in parallel.
7. Blocking forever on a channel (leak); an unrecovered panic in any goroutine (crash); also unbounded creation exhausting memory.
</details>

---

## 21. Summary

- A **goroutine** is a **runtime-managed, user-space thread** of execution: **~2 KiB growable stack**, **~1 µs** to create, **~150 ns** to switch, and you can run **hundreds of thousands** (we measured 1,000,000 alive at 2,048 bytes each).
- "Virtual thread": like a process to a computer, a thread to a process, a goroutine is a lighter, software-managed "thread" *inside* real OS threads.
- Start one with **`go f(x)`**: it returns immediately; **when `main` returns, all goroutines die**; **order is not guaranteed**.
- The **GMP scheduler**: **G**oroutines run on **M**achines (OS threads) that hold a **P**rocessor (run queue, count = `GOMAXPROCS`); idle Ps **steal work**.
- Blocking on **channels/mutexes/timers/network** just **parks** the goroutine; **blocking syscalls** hand the P to another thread; **long loops** are **preempted** (Go 1.14+).
- Pitfalls: not waiting, **leaks**, **panics** crash everything, **data races**, unbounded spawning.
- Everything else in the course's second half (WaitGroups, mutexes, channels) builds on these.

### ➡️ What's next?

**Part 7 shifts to backend development.** [Chapter 37](37-into-backend-development.md) starts from zero: what a backend is, how the web works (HTTP, ports, servers), and why Go, with its goroutines, is such a good fit.
