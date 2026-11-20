# Chapter 39: The Go Runtime — The Heart of Go

> **Goal of this chapter:** Open the box that ships inside every Go program. You'll learn the split between **kernel space** and **user space**, what the **Go runtime** is (a "mini operating system" inside your process), exactly what happens between "the OS starts your binary" and "`main.main` runs", how the **`epoll`** event mechanism lets the runtime wait for thousands of I/O events at once, and how the **garbage collector** works, all verified with real traces from your own machine.

**Difficulty:** 🔴 Advanced  **Estimated time:** 3–4 hours  **Prerequisite:** [Chapters 18, 27, 36, 38](38-os-or-go-server-the-complete-journey-of-a-request.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Kernel space vs. user space](#2-kernel-space-vs-user-space)
3. [System calls: the bridge](#3-system-calls-the-bridge)
4. [What is the Go runtime?](#4-what-is-the-go-runtime)
5. [A "mini OS" inside your process](#5-a-mini-os-inside-your-process)
6. [Where the runtime lives](#6-where-the-runtime-lives)
7. [How a Go program starts](#7-how-a-go-program-starts)
8. [`epoll`: waiting for many things at once](#8-epoll-waiting-for-many-things-at-once)
9. [The scheduler (recap)](#9-the-scheduler-recap)
10. [Memory management: the allocator](#10-memory-management-the-allocator)
11. [The garbage collector in depth](#11-the-garbage-collector-in-depth)
12. [Other runtime services](#12-other-runtime-services)
13. [Observing the runtime](#13-observing-the-runtime)
14. [Putting it together: one HTTP request](#14-putting-it-together-one-http-request)
15. [Common misconceptions](#15-common-misconceptions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- What **kernel space** and **user space** are, and why the split exists
- What a **system call** costs and why the runtime tries to avoid them
- The eight-ish jobs of the **Go runtime**
- The start-up sequence from OS process creation to `main.main`
- How `epoll` works and why it's efficient
- How Go allocates memory and how its **concurrent garbage collector** finds and frees garbage, with real `gctrace` output
- Tools to look inside a running program

---

## 2. Kernel space vs. user space

Chapter 27 introduced two CPU modes. Now let's see how they map onto **memory**.

Every process sees a large **virtual address space**. The operating system divides it into two regions:

```
 High addresses
 ┌──────────────────────────────────────────┐
 │              KERNEL SPACE                │  ← the OS kernel's code and data
 │  • kernel code, drivers                  │    (mapped in every process, but
 │  • page tables, process table (PCBs)     │     NOT accessible from user mode)
 │  • network stack, file-system caches     │
 │  • hardware access                       │
 ├──────────────────────────────────────────┤  ← boundary: touching kernel memory
 │              USER SPACE                  │     from user mode = fault
 │  ┌────────────────────────────────────┐  │
 │  │ Stack                              │  │
 │  │ Heap                               │  │  ← YOUR process:
 │  │ Data segment                       │  │     your Go code + the Go runtime
 │  │ Code segment                       │  │
 │  └────────────────────────────────────┘  │
 └──────────────────────────────────────────┘
 Low addresses
```

| | **Kernel space** | **User space** |
|--|------------------|----------------|
| Who runs there | The OS kernel | Your programs (browser, editor, Go binary) |
| CPU mode | Privileged (ring 0) | Restricted (ring 3) |
| Can access hardware? | ✅ Yes | ❌ No |
| Can touch other processes' memory? | ✅ Yes | ❌ No |
| If it crashes | The whole machine (kernel panic / blue screen) | Only that process |
| Contains | Drivers, scheduler, TCP/IP, file systems | Applications and libraries |

**Why the split?** **Protection.** A bug in your program (a bad pointer, an infinite loop) must not be able to corrupt the OS, read another user's data, or crash the machine. The CPU hardware enforces the boundary.

> **Analogy: a bank vault.** Customers (user programs) may stand in the lobby (user space) and fill in forms. Only bank staff (the kernel) may enter the vault (kernel space) where cash and records live. If you want cash, you don't walk into the vault: you hand a form to a teller. That form is a **system call**.

**Important:** your Go program, *including the entire Go runtime*, is **user-space code**. The Go runtime is **not** part of the operating system.

---

## 3. System calls: the bridge

A **system call** ("syscall") is how user-space code asks the kernel to do something privileged:

```
User space                                   Kernel space
───────────                                  ────────────
fmt.Println("hi")
   └─► os.Stdout.Write
         └─► syscall write(1, "hi\n", 3) ──► switch to kernel mode
                                             validate arguments
                                             copy bytes to the terminal's buffer
                                             switch back to user mode ◄──────────
         ◄── returns 3
```

Common syscalls: `read`, `write`, `open`, `close`, `socket`, `bind`, `listen`, `accept4`, `mmap` (get memory from the OS), `clone` (create a thread), `futex` (thread sleep/wake), `epoll_wait`, `exit`.

**They aren't free.** Each syscall means: a mode switch, saving registers, validating pointers, running kernel code, and switching back: hundreds of nanoseconds to microseconds, plus cache disturbance. So a well-designed runtime **minimizes syscalls**: it asks the OS for large chunks of memory (not one `malloc` per object), keeps its own thread pool, schedules goroutines without kernel help, and uses `epoll` to wait on *many* sockets with *one* call.

You saw real syscalls in Chapter 38's `strace` output. You can watch any program's syscalls with `strace -c ./prog` (a summary by count).

---

## 4. What is the Go runtime?

> The **Go runtime** is a body of Go (and a little assembly) code, **linked into every Go binary**, that provides the services your program needs at run time: scheduling goroutines, allocating and freeing memory, waiting for I/O, handling panics, and more.

A C program's `main` is called almost directly by the OS. A Go program's `main.main` is called *by the Go runtime*, after the runtime has set up an environment for it. Your program's true first instruction belongs to the runtime.

```
┌────────────────────── your Go binary (one file) ──────────────────────┐
│                                                                       │
│   YOUR CODE          STANDARD LIBRARY           GO RUNTIME            │
│   package main       fmt, net/http, os ...      scheduler, GC, memory │
│   (main.main ...)    (mostly Go code that       allocator, netpoller, │
│                       calls the runtime & OS)   stack manager, ...    │
└────────────────────────────────────────────────────────────────────────┘
                                   │  system calls
                                   ▼
                         Operating system kernel
```

This is one reason "Hello, World" is ~1.5 MB: the runtime (plus `fmt`) travels with it (Chapter 19). A binary we inspected contained **1,714 symbols** starting with `runtime.`, even though the program didn't mention the runtime once.

---

## 5. A "mini OS" inside your process

The runtime does for goroutines and memory what the OS does for processes and RAM, only in user space and tuned for Go:

| Service | Operating system does it for... | Go runtime does it for... |
|---------|---------------------------------|---------------------------|
| **Scheduling** | Processes / threads on CPU cores | **Goroutines** on OS threads |
| **Memory management** | Physical RAM ↔ virtual memory for processes | The **heap** inside your process: allocation and garbage collection |
| **Stacks** | Fixed thread stacks | **Growable goroutine stacks** |
| **I/O waiting** | Blocking syscalls, interrupts | **Netpoller** on top of `epoll` |
| **Timers** | Kernel timers | `time.Sleep`, `time.After`, tickers (a heap of timers) |
| **Synchronization** | Semaphores, futexes | Channels, mutexes, `WaitGroup`s |
| **Fault handling** | Signals, segfaults | `panic`, `recover`, nil-pointer and bounds-check faults |
| **Type information** | (n/a) | Type descriptors for interfaces, `reflect`, maps, `fmt %v` |

The eight main jobs:

1. **Scheduler**: create and run goroutines; the GMP model (Chapter 36).
2. **Memory allocator**: fast heap allocation with per-P caches.
3. **Garbage collector**: reclaim unreachable memory concurrently.
4. **Stack management**: goroutine stack growth and shrinking.
5. **Network poller**: `epoll`/`kqueue`/IOCP integration.
6. **Channels, `select`, timers**: implemented in the runtime.
7. **Panic/recover, defer**, runtime errors (Chapter 34).
8. **Type system support**: interface dispatch, reflection, maps, slices growth, string concatenation helpers.

---

## 6. Where the runtime lives

The runtime is ordinary Go source in the standard library. Look at yours:

```bash
ls $(go env GOROOT)/src/runtime | head
```

On the test machine (Go 1.25) the directory has **781 files**. A few you'll recognize by name:

| File | Content | Size |
|------|---------|------|
| `proc.go` | **The scheduler** (`schedinit`, `newproc`, `schedule`, `findRunnable`, ...) | ~7,700 lines |
| `malloc.go` | **The memory allocator** | ~2,100 lines |
| `mgc.go` (+ `mgc*.go`) | **The garbage collector** | ~2,000 lines (+ many files) |
| `stack.go` | Goroutine stack allocation and growth | |
| `chan.go`, `select.go` | Channels and `select` (Chapter 69) | |
| `panic.go` | `panic`, `recover`, `defer` machinery | |
| `netpoll_epoll.go` | The Linux **epoll** integration | |
| `time.go` | Timers and `Sleep` | |
| `map*.go`, `slice.go`, `string.go` | Built-in data structures | |
| `asm_amd64.s` | Assembly entry points for x86-64 | |

Reading `proc.go` is one of the best ways to level up as a Go engineer: it's well commented. The entry points in this version: `runtime·rt0_go` at `asm_amd64.s:159` and `schedinit` at `proc.go:832` (line numbers change between releases).

---

## 7. How a Go program starts

You run `./myserver`. Here is the sequence, from the shell to `main.main`:

```
1. SHELL asks the OS to execute the file                        [kernel: execve]
2. KERNEL   creates a PROCESS: new address space, loads the binary's
            code/data segments, sets up the main thread's stack   (Chapter 28)
3. KERNEL   sets the PC to the binary's entry point: _rt0_amd64_linux
            (the GO RUNTIME's code, not your main!)               [user mode begins]
4. RUNTIME  rt0_go (assembly): set up the first goroutine's stack, detect CPU features
5. RUNTIME  schedinit(): initialize the memory allocator, the scheduler (create the Ps,
            GOMAXPROCS), the garbage collector, read environment settings (GODEBUG, GOGC...)
6. RUNTIME  newproc(runtime.main): create the FIRST GOROUTINE, whose entry function is
            runtime.main (still not yours!)
7. RUNTIME  mstart(): start scheduling. The main thread (M0) attaches a P and runs goroutines
8. runtime.main (in the first goroutine):
       • start the sysmon background monitor thread (preemption, netpoll checks)
       • run runtime's own init tasks
       • run the init tasks of every imported package, in dependency order,
         then main.init variable initialization and init() functions   (Chapter 14)
       • enable the GC
       • call main.main   ◄── YOUR CODE FINALLY RUNS
9. When main.main returns → runtime calls exit(0) → OS tears down the process
```

So **your `main` is just a goroutine**, the first one the runtime creates. That's why calling `os.Exit` or returning from `main` ends everything.

You can *see* package initialization order (step 8) with the runtime's own tracing:

```bash
GODEBUG=inittrace=1 ./myserver 2>&1 | head
```

Real output:

```
init internal/bytealg @0 ms, 0 ms clock, 0 bytes, 0 allocs
init runtime @0.016 ms, 0.039 ms clock, 0 bytes, 0 allocs
init crypto/internal/fips140deps/cpu @0.28 ms, 0.002 ms clock, 0 bytes, 0 allocs
init math @0.29 ms, 0.002 ms clock, 0 bytes, 0 allocs
init errors @0.30 ms, 0 ms clock, 0 bytes, 0 allocs
init iter @0.31 ms, 0.003 ms clock, 16 bytes, 1 allocs
...
```

The whole runtime initialization takes a fraction of a millisecond: process start-up is fast.

---

## 8. `epoll`: waiting for many things at once

### The problem

A server has thousands of open connections. Which ones have data to read *right now*? Three approaches:

**Approach 1: one blocked thread per connection.**
Each thread calls `read` and sleeps until data arrives. Simple, but 10,000 connections = 10,000 threads (stacks, scheduling, context switches; Chapter 32). ❌ Doesn't scale.

**Approach 2: `select` / `poll`** (older).
One thread hands the kernel the **full list** of sockets and asks "which are ready?". Each call the kernel scans **all N** and the program scans the result. Cost per call: **O(N)**. With 10,000 sockets, 10,000 checks on *every* loop, even when only 3 are ready. ❌ Slow at scale.

**Approach 3: `epoll` (Linux)** (also `kqueue` on BSD/macOS, IOCP on Windows).
The kernel **remembers** which sockets you care about, and gives you *only the ones that became ready*. Cost per wait: **O(ready)**, not O(total).

### The three `epoll` calls

```
epoll_create1()          create an epoll instance (a kernel object; returns an fd)
epoll_ctl(ADD/DEL, fd)   register (or remove) a socket you want to watch
epoll_wait()             sleep until ≥1 registered socket is ready; return the ready list
```

```
   Runtime (user space)                       Kernel
   ────────────────────                       ──────
   epoll_create1()                       ───►  creates the epoll set
   for each new connection:
     epoll_ctl(ADD, connfd)              ───►  "watch connfd for readability"
   ...
   epoll_wait()  (idle: thread sleeps)   ───►  (blocks; no CPU used)
                                               ... data arrives on connfd 7 ...
                 ◄────────────────────────    returns [connfd 7]
   wake the goroutine parked on fd 7
```

In Chapter 38's real `strace` you saw the runtime doing exactly this:

```
epoll_ctl(5, EPOLL_CTL_ADD, 4, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, ...}) = 0
```

- `5` is the epoll instance; `4` is the client connection.
- `EPOLLIN` = notify me when it's readable; `EPOLLOUT` = writable; `EPOLLRDHUP` = peer closed.
- `EPOLLET` = **edge-triggered**: tell me when the socket *becomes* ready (a change), not repeatedly while it stays ready. Efficient, but the runtime must read until `EAGAIN`.

### How Go uses it: the netpoller

```
goroutine G1:  conn.Read(buf)
   │  kernel says EAGAIN (no data yet)
   ▼
G1 is PARKED; socket registered with epoll     ← G1 costs no thread, no CPU
   ...
scheduler (or the sysmon thread) calls epoll_wait, gets "fd 4 readable"
   ▼
G1 made runnable again, put on a run queue
   ▼
G1 continues: Read returns the bytes
```

To *your* code, `conn.Read` looks like a simple **blocking** call. Underneath, it's event-driven. This is the trick that lets Go programmers write straightforward sequential code and still get event-loop scalability: **no callbacks, no `async/await`.**

---

## 9. The scheduler (recap)

Chapter 36 covered this in detail; here is the runtime-level view:

- **G**oroutines, **M**achine threads, **P**rocessors (`GOMAXPROCS` of them); each P has a local run queue; idle Ps steal work.
- Blocking on channels/mutexes/timers/network I/O → **park** the goroutine; blocking syscalls → **hand off the P**.
- **`sysmon`**: a special thread that runs *without a P*, waking every 20 µs–10 ms to: preempt goroutines running > 10 ms, retake Ps from threads stuck in syscalls, and poll the network if nobody else has.

The scheduler's entry point in the source is `schedule()` → `findRunnable()` in `proc.go`.

---

## 10. Memory management: the allocator

When you write `p := &User{...}` or `make([]int, n)` and the value escapes to the **heap** (Chapter 18), the runtime's **allocator** (`mallocgc`) provides the memory.

The design (simplified from TCMalloc):

```
OS (mmap large regions: "arenas")
   └── Heap: divided into PAGES (8 KiB)
         └── SPANS: runs of pages dedicated to ONE object size class
               e.g., a span of 8 KiB holds 1,024 objects of 8 bytes
                     a span holds 512 objects of 16 bytes ... up to ~32 KiB objects
                     bigger objects get their own span
```

- **Size classes:** small allocations are rounded up to one of ~68 sizes (8, 16, 24, 32, 48, 64, ...). That's why capacities of slices like `append` growth look odd (Chapter 25: `848`, `71`): the runtime rounds to size-class boundaries.
- **Per-P cache (`mcache`):** each P has a private cache of free objects per size class → allocation is **lock-free** in the common case.
- If a size class runs dry, the P refills from the shared **`mcentral`**, which gets spans from the **`mheap`**, which gets memory from the **OS** via `mmap`.
- **Tiny allocator:** very small pointer-free objects (< 16 B) are packed together.
- Freed memory returns to the runtime's free lists; a background **scavenger** returns unused pages to the OS.

Allocation therefore takes about **tens of nanoseconds** for small objects, and is one reason Go code is fast even though it allocates a lot.

---

## 11. The garbage collector in depth

Chapter 18 introduced the GC at a high level. Now the mechanism.

### The goals

- **Low latency:** keep "stop-the-world" pauses tiny (well under a millisecond).
- **Concurrent:** do most work **while your program runs**.
- **Simple to tune:** essentially one knob (`GOGC`).

### Algorithm: concurrent, tri-color, mark-and-sweep

All heap objects are conceptually colored:

| Color | Meaning |
|-------|---------|
| **White** | Not yet seen: candidate garbage |
| **Grey** | Seen (reachable), but its outgoing pointers haven't been scanned yet |
| **Black** | Seen, and all its pointers scanned |

**One GC cycle:**

```
1. SWEEP TERMINATION  (short stop-the-world) — finish the previous cycle; enable the write barrier
2. MARK  (concurrent) — start from the ROOTS: package variables + every goroutine's stack
        (registers included); grey everything they point to; then repeatedly:
        take a grey object, scan its pointers (grey the white objects it points to),
        turn it black. Runs on dedicated GC worker goroutines (~25% of Ps)
        plus "mutator assists" (if your code allocates too fast, it helps out).
3. MARK TERMINATION   (short stop-the-world) — verify that no grey objects remain; disable barrier
4. SWEEP  (concurrent, lazy) — everything still WHITE is unreachable → freed/reused as needed
```

**The write barrier** is the trick that lets marking run *while your program mutates pointers*: when your code writes a pointer into the heap during the mark phase, a small piece of compiler-inserted code records it so the collector can't miss a newly reachable object.

```
   roots ──► [A grey] ──► [B white] ──► ...        (marking)
   roots ──► [A black]──► [B grey]  ──► [C grey]
   ...finally: all reachable = black; the rest = white → garbage
```

### When does a GC cycle run? `GOGC`

By default a cycle starts when the heap has grown to **2×** the live heap left after the previous cycle (`GOGC=100`: allow 100% growth). Set `GOGC=200` to collect less often (more memory, less CPU), `GOGC=50` for more often. Go 1.19 added a **soft memory limit** (`GOMEMLIMIT=2GiB` or `debug.SetMemoryLimit`), which makes the GC work harder as you near the limit. Useful for containers.

### Watching it: `GODEBUG=gctrace=1`

Real output from a program that allocates and drops 1 MiB slices:

```
gc 1 @0.004s 4%: 0.069+0.76+0.092 ms clock, 0.83+0.63/0.68/0.065+1.1 ms cpu, 3->4->1 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 12 P
gc 2 @0.005s 5%: 0.065+1.8+0.027 ms clock, 0.79+0.22/1.0/0+0.32 ms cpu, 3->4->1 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 12 P
gc 3 @0.008s 5%: 0.042+0.70+0.013 ms clock, ...
```

Decoding the first line:

| Piece | Meaning |
|-------|---------|
| `gc 1` | GC cycle number 1 |
| `@0.004s` | Time since the program started |
| `4%` | Fraction of CPU time spent in GC so far |
| `0.069+0.76+0.092 ms clock` | **Wall-clock time of the three phases**: stop-the-world sweep termination **0.069 ms**, concurrent mark **0.76 ms**, stop-the-world mark termination **0.092 ms** |
| `0.83+0.63/0.68/0.065+1.1 ms cpu` | CPU time by phase (assist / background / idle) |
| `3->4->1 MB` | Heap size at GC start → at GC end → **live** heap after marking |
| `4 MB goal` | The heap size that triggers the next cycle |
| `12 P` | Number of processors used |

The important numbers: the **stop-the-world pauses were 0.069 ms and 0.092 ms**: **microseconds**, that's the "low-latency" claim in action. Most GC work (0.76 ms) happened concurrently while the program kept running.

### Making the GC's job easier (practical tips)

- **Allocate less**: reuse buffers (`sync.Pool`), pre-size slices (`make([]T, 0, n)`), avoid unnecessary pointers and boxing.
- **Fewer pointers = less to scan**: a `[]int` with 1 million elements is nearly free to scan; a `[]*Item` needs 1 million pointer checks.
- **Don't fight escape analysis** blindly; measure with `-gcflags=-m` and profiles.
- Watch for **goroutine leaks** and **unbounded caches**, which keep memory reachable.

---

## 12. Other runtime services

- **Channels and `select`** (`chan.go`, `select.go`): a channel is a struct with a buffer, a lock, and queues of waiting senders/receivers (parked goroutines). Sending to a channel with a waiting receiver hands the value over directly and readies that goroutine. (Chapter 69.)
- **Timers**: `time.Sleep`, `time.After`, `time.NewTicker` are entries in a per-P min-heap. The scheduler checks it when looking for work, so millions of timers don't need millions of threads.
- **Maps** (`map.go`): hash tables implemented in the runtime; Go 1.24+ uses a Swiss-table-based design.
- **Panics and defers** (`panic.go`): the defer stack and unwinding logic (Chapter 34); nil-pointer dereferences are hardware faults turned into panics via signal handlers.
- **Signals**: the runtime installs handlers (for preemption, crashes, `SIGINT` → `os/signal`).
- **`reflect` and interfaces**: every value stored in an interface carries a pointer to a **type descriptor** built by the compiler; the runtime uses them for method dispatch, `==`, `fmt`, and GC pointer maps.

---

## 13. Observing the runtime

You don't have to guess. Go ships excellent tools:

| Tool | Use | How |
|------|-----|-----|
| `runtime.NumGoroutine()`, `NumCPU()`, `GOMAXPROCS(0)` | Quick counts | In code |
| `runtime.ReadMemStats(&m)` | Heap, stacks, GC counters | In code |
| `runtime/metrics` | Detailed, stable runtime metrics | In code |
| `GODEBUG=gctrace=1` | One line per GC cycle | Env var |
| `GODEBUG=schedtrace=1000` | Scheduler state every second | Env var |
| `GODEBUG=inittrace=1` | Package init timings | Env var |
| `go build -gcflags=-m` | Escape-analysis decisions | Build flag |
| `go run -race` | Data-race detector | Flag |
| **`pprof`** | CPU, heap, goroutine, block profiles | `import _ "net/http/pprof"` then `go tool pprof http://localhost:6060/debug/pprof/heap` |
| **`go tool trace`** | Timeline of goroutines, GC, syscalls | `runtime/trace` |
| `strace -f -c` | System-call counts | Linux |

A tiny example combining a few:

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println(runtime.Version(), runtime.GOOS, runtime.GOARCH)

	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Println("goroutines:", runtime.NumGoroutine())
	fmt.Println("heap in use (KiB):", m.HeapAlloc/1024)
	fmt.Println("stacks in use (KiB):", m.StackInuse/1024)
	fmt.Println("GC cycles so far:", m.NumGC)
	fmt.Println("total GC pause (ns):", m.PauseTotalNs)
}
```

---

## 14. Putting it together: one HTTP request

Tie Chapters 36–39 into a single story. A client calls `http.Get("https://example.com")` in your Go program (a *client*, to vary the picture):

```
1. main.main runs as a goroutine (the first one the runtime made)
2. http.Get → net.Dial: the runtime creates a non-blocking socket (syscall socket),
   starts connecting (syscall connect → EINPROGRESS)
3. The goroutine PARKS. The socket is registered with epoll (epoll_ctl ADD)
4. The scheduler runs other goroutines (or the M sleeps in epoll_wait; no CPU)
5. Kernel: the TCP handshake completes → socket becomes writable → epoll reports it
6. Runtime: the goroutine is made runnable; it writes the request (syscall write)
7. Goroutine parks again waiting for the response (registered with epoll)
8. Response bytes arrive (NIC → kernel → socket buffer → epoll event)
9. Runtime wakes the goroutine; it read()s the bytes; net/http parses the response
10. Buffers were allocated via the runtime allocator; when unused, the GC frees them
11. main.main returns → runtime exits the process
```

Every step involves the runtime *quietly* coordinating between your sequential-looking code and the kernel's event machinery.

---

## 15. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "Go runs on a virtual machine like Java" | Go compiles to **native machine code**; the runtime is a *library* linked into the binary, not a VM/interpreter |
| "The Go runtime is part of the OS" | It's **user-space code inside your process** |
| "My `main` is the first thing that runs" | The runtime's entry point (`rt0_go`) runs first; `main.main` runs in the **first goroutine** the runtime creates |
| "The GC stops the world for long periods" | Stop-the-world phases are typically **tens of microseconds**; the mark phase is concurrent |
| "`epoll` is a Go feature" | It's a **Linux kernel** feature the runtime uses (`kqueue`/IOCP elsewhere) |
| "The runtime adds a big slowdown" | It's compiled Go; scheduling, allocation, and GC are highly tuned, and much of the "overhead" replaces work you'd do by hand |
| "I must free memory in Go" | The GC does it; you manage *reachability* |
| "More `GOGC` is always better" | It trades memory for CPU; the right value depends on your workload and limits |
| "Goroutines are scheduled by the OS" | By the Go runtime; the OS only schedules the threads underneath |

---

## 16. Exercises

### Exercise 1: Two spaces
Classify each as kernel-space or user-space: (a) the TCP/IP implementation, (b) `http.ListenAndServe`, (c) the Go garbage collector, (d) a device driver, (e) the Go scheduler.

<details><summary>Solution</summary>

(a) kernel, (b) user (Go code that makes syscalls), (c) user (part of the runtime), (d) kernel, (e) user (part of the runtime).
</details>

### Exercise 2: Count syscalls
On Linux, run `strace -c ./yourprogram` on a small program that prints a line and exits. Which syscalls dominate? Is `write` there?

<details><summary>Solution</summary>

You'll see a table with `mmap`, `rt_sigaction`, `futex`, `clone`/`clone3`, `read`, and `write` (one call for `fmt.Println`), etc. The runtime's start-up does a fair amount of syscalls (memory arenas, signals, threads) before `main`.
</details>

### Exercise 3: Startup order
Put in order: (a) `main.main`, (b) OS creates the process, (c) `schedinit`, (d) package `init` functions, (e) `runtime.main` goroutine starts, (f) `rt0_go`.

<details><summary>Solution</summary>

(b) → (f) → (c) → (e) → (d) → (a).
</details>

### Exercise 4: Why epoll?
A server with 50,000 mostly-idle connections needs to find the 5 that have data. Compare the work per loop for `select`/`poll` versus `epoll`.

<details><summary>Solution</summary>

`select`/`poll` pass and scan all 50,000 descriptors each call (O(N)); `epoll` returns only the ready 5 (O(ready)) because the kernel already maintains the interest list and a ready list.
</details>

### Exercise 5: Read a `gctrace` line
Given `gc 12 @2.100s 3%: 0.020+4.5+0.030 ms clock, ... 40->42->20 MB, 40 MB goal, ... 8 P`, answer: what were the stop-the-world pauses? How large was the live heap after marking? What triggers the next GC?

<details><summary>Solution</summary>

STW pauses: 0.020 ms and 0.030 ms (the first and third numbers); the concurrent mark took 4.5 ms. Live heap after marking: 20 MB. The heap goal of 40 MB (2× live, with `GOGC=100`) determines when the next cycle starts.
</details>

### Exercise 6: Watch the GC yourself
Write a program that allocates a lot of short-lived garbage in a loop, run it with `GODEBUG=gctrace=1`, then set `GOGC=400` and run again. What changes in the number of `gc` lines and in peak memory?

<details><summary>Solution</summary>

With `GOGC=400` the heap may grow to ~5× the live heap before a collection, so there are far fewer `gc N` lines but a larger peak heap (`->` numbers are bigger): a memory-for-CPU trade.
</details>

### Exercise 7 (challenge): Explore the source
Open `$(go env GOROOT)/src/runtime/proc.go` and find (1) the `schedinit` function, (2) where `newproc` is defined, (3) a comment that explains the G, M, P model. Write one sentence about what you learned.

<details><summary>Hint</summary>

Search for `// Goroutine scheduler` near the top of `proc.go`; it contains a long comment explaining the scheduler's design (Gs, Ms, Ps, spinning threads, and so on). `grep -n "func schedinit\|func newproc" proc.go` finds the functions.
</details>

---

## 17. Quiz

1. Why do operating systems separate kernel space from user space?
2. Is the Go runtime part of the kernel?
3. What does the OS run first when you start a Go binary: `main.main` or the runtime's entry point?
4. What advantage does `epoll` have over `select`?
5. What are the three phases of a GC cycle, and which involve stopping the world?
6. What does `GOGC=100` mean?
7. Why is `main.main` "just a goroutine"?

<details><summary>Answers</summary>

1. Protection: user programs can't touch hardware or other programs' memory; crashes stay contained.
2. No: it's user-space code linked into your program.
3. The runtime's entry point (`rt0_go`); the runtime later calls `main.main`.
4. The kernel tracks interest and returns only ready sockets: O(ready) instead of O(all).
5. Sweep termination (STW), concurrent mark, mark termination (STW); sweeping is concurrent, too.
6. Start the next cycle when the heap has grown 100% beyond the live heap of the previous cycle (2×).
7. The runtime creates the first goroutine to run `runtime.main`, which initializes packages and then calls `main.main`.
</details>

---

## 18. Summary

- **Kernel space** (privileged: drivers, TCP/IP, scheduler) and **user space** (your process) are separated for protection; **system calls** cross the boundary and cost real time.
- The **Go runtime** is user-space code linked into every binary: a **"mini OS"** providing the **goroutine scheduler, memory allocator, garbage collector, stack management, netpoller, channels/timers, panic handling, and type support**.
- **Start-up:** OS creates the process → runtime `rt0_go` → `schedinit` → creates the first goroutine (`runtime.main`) → package inits → **`main.main`**.
- **`epoll`** (`kqueue`/IOCP elsewhere) lets one thread wait for thousands of sockets in O(ready) time; the runtime parks and wakes goroutines around it, giving blocking-style code with event-loop scalability.
- The **allocator** (size classes, spans, per-P caches) makes heap allocation fast; the **GC** is **concurrent, tri-color mark-and-sweep** with write barriers and pauses measured in **microseconds** (verified with `gctrace`).
- Tune with `GOGC`/`GOMEMLIMIT`, observe with `pprof`, `trace`, `gctrace`, `schedtrace`, and `-race`.

### ➡️ What's next?

The theory is done. **Part 8 begins the e-commerce project**: in [Chapter 40](40-e-commerce-project-get-products.md) you'll build your first real API endpoint: a `GET /products` handler that returns JSON.
