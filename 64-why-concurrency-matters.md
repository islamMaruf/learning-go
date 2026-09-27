# Chapter 64: Why Concurrency Matters — Goroutines, Waiting, and Server Capacity

> **Goal of this chapter:** Understand *why* Go programs use goroutines by **measuring** what they buy. You'll see that a server's capacity follows a simple law (**Little's law**) that you can check against the numbers from Chapter 62, that three independent 200 ms waits cost **600 ms sequentially but 200 ms concurrently**, that CPU-bound work speeds up only with more cores while waiting-bound work speeds up on *one*, that a goroutine costs about **2 KB** (100,000 of them fit in 200 MB and ran on 13 OS threads), and that Go 1.22 quietly fixed the classic **loop-variable capture** bug. Every number below comes from a real run.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 5 hours  **Prerequisite:** [Chapters 31-36 (concurrency, threads, goroutines) and Chapter 62](62-experiments-before-optimizing.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Thinking about capacity](#2-thinking-about-capacity)
3. [The problem: waiting in a line](#3-the-problem-waiting-in-a-line)
4. [The same problem with a real database](#4-the-same-problem-with-a-real-database)
5. [Concurrency is not parallelism](#5-concurrency-is-not-parallelism)
6. [What a goroutine costs](#6-what-a-goroutine-costs)
7. [How the scheduler works (a sketch)](#7-how-the-scheduler-works-a-sketch)
8. [`main` does not wait](#8-main-does-not-wait)
9. [The loop-variable trap](#9-the-loop-variable-trap)
10. [Concurrency you already use](#10-concurrency-you-already-use)
11. [When concurrency does not help](#11-when-concurrency-does-not-help)
12. [The roadmap for the next chapters](#12-the-roadmap-for-the-next-chapters)
13. [Common mistakes](#13-common-mistakes)
14. [Interview questions](#14-interview-questions)
15. [Exercises](#15-exercises)
16. [Quiz](#16-quiz)
17. [Summary](#17-summary)

---

## 1. What you will learn

- **Little's law** (`concurrency = throughput × latency`) and how to use it for server sizing
- Why waiting (I/O) is the natural target of concurrency, and how much it saves
- The difference between **concurrency** (structuring independent tasks) and **parallelism** (running simultaneously), measured
- The real cost of a goroutine, compared to an operating-system thread
- A sketch of Go's scheduler (**G, M, P**) and what `GOMAXPROCS` controls
- Why `main` returning kills every goroutine
- The **loop-variable capture** bug, and what changed in Go 1.22
- How to recognize when concurrency will *not* help

---

## 2. Thinking about capacity

A server is a machine that turns *requests into responses*. Three quantities describe it:

| Symbol | Name | Unit |
|--------|------|------|
| **λ** (lambda) | throughput | requests/second |
| **W** | latency (time in the system per request) | seconds |
| **L** | concurrency (requests being handled *right now*) | count |

They are tied together by **Little's law**, one of the most useful equations in systems work:

```
L = λ × W          (average number in the system = arrival rate × average time in the system)
```

It holds for any stable system, with no assumptions about distributions. Check it against the measurement from Chapter 63, where the paginated list endpoint handled 200 requests using 10 concurrent workers:

```
mean latency W = 66.83 ms = 0.06683 s        concurrency L = 10
predicted throughput λ = L / W = 10 / 0.06683 = 149.6 requests/s
measured throughput                              147.1 requests/s        ← within 2%
```

The law becomes a **planning tool**:

- *"Each request takes 50 ms and we want 2,000 requests/second"* → we need **L = 2,000 × 0.05 = 100** requests in flight at once. A server that can handle only one at a time (`L = 1`) tops out at **20 requests/second**, no matter how fast the machine.
- *"Each in-flight request needs 20 MB of memory and we have 8 GB"* → at most **400** concurrent requests; more will run out of memory (Chapter 62).
- *Reduce W* (make requests faster) and the same concurrency yields more throughput; *increase L* (handle more at once) and the same W yields more throughput too.

That is what this part of the course is about: **increasing L safely**. A server that handles requests one after another has `L = 1`. Go's `net/http` already runs *each request in its own goroutine* (section 10), so L is large by default, but inside a single request, you can gain further by overlapping independent work, as the next sections show.

---

## 3. The problem: waiting in a line

Most of the time a web request is not *computing*; it is **waiting**: for a database, another service, a disk. Suppose handling a request needs data from three independent services, each taking 200 ms:

```go
package main

import (
	"fmt"
	"time"
)

// slowCall pretends to be a network call or a slow query: it mostly WAITS.
func slowCall(name string, d time.Duration) string {
	time.Sleep(d)
	return name + " done"
}

func main() {
	start := time.Now()

	fmt.Println(slowCall("users service", 200*time.Millisecond))
	fmt.Println(slowCall("orders service", 200*time.Millisecond))
	fmt.Println(slowCall("stock service", 200*time.Millisecond))

	fmt.Println("sequential total:", time.Since(start).Round(10*time.Millisecond))
}
```

```
users service done
orders service done
stock service done
sequential total: 600ms
```

600 ms: each call starts only after the previous one finished, even though they don't depend on each other. It's like waiting in three separate queues one after another when you could join all three at once.

The concurrent version starts each call in its own **goroutine** (a function running independently, started with the `go` keyword) and waits for all of them:

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func slowCall(name string, d time.Duration) string {
	time.Sleep(d)
	return name + " done"
}

func main() {
	start := time.Now()

	var wg sync.WaitGroup
	results := make([]string, 3) // each goroutine writes its OWN slot: no sharing, no race
	calls := []string{"users service", "orders service", "stock service"}

	for i, name := range calls {
		wg.Add(1)
		go func() {
			defer wg.Done()
			results[i] = slowCall(name, 200*time.Millisecond)
		}()
	}
	wg.Wait() // wait for all three

	for _, r := range results {
		fmt.Println(r)
	}
	fmt.Println("concurrent total:", time.Since(start).Round(10*time.Millisecond))
}
```

```
users service done
orders service done
stock service done
concurrent total: 200ms
```

**200 ms instead of 600 ms**: a 3× improvement, for three added lines. The total is the *longest* call, not the *sum*: `max(200, 200, 200)` versus `200 + 200 + 200`. (`sync.WaitGroup` is how `main` waits for the goroutines; Chapter 65 is entirely about it. For now, read `wg.Add(1)` as "one more task to wait for", `wg.Done()` as "this task finished", and `wg.Wait()` as "block until every task has finished".)

Notice the design detail: each goroutine writes to `results[i]`, **its own element**. They share the slice but never touch the same element, so there's no race. Chapter 67 shows what happens when goroutines write the *same* variable.

---

## 4. The same problem with a real database

`time.Sleep` is a simulation; here is the same experiment with real queries. PostgreSQL's `pg_sleep(0.2)` makes a query take 200 ms while the Go program simply waits for the answer (a perfect stand-in for a slow query):

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"sync"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

// slowQuery asks PostgreSQL to "work" for 200 ms: the Go program just waits for the answer.
func slowQuery(ctx context.Context, db *sql.DB, name string) error {
	_, err := db.ExecContext(ctx, "SELECT pg_sleep(0.2)")
	fmt.Println(name, "finished")
	return err
}

func main() {
	db, err := sql.Open("pgx", "postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable")
	if err != nil {
		panic(err)
	}
	defer db.Close()
	db.SetMaxOpenConns(10) // the pool must allow several queries at once
	ctx := context.Background()
	db.PingContext(ctx)

	names := []string{"query A", "query B", "query C"}

	start := time.Now()
	for _, n := range names {
		slowQuery(ctx, db, n)
	}
	fmt.Println("one after another:", time.Since(start).Round(10*time.Millisecond))

	start = time.Now()
	var wg sync.WaitGroup
	for _, n := range names {
		wg.Add(1)
		go func() {
			defer wg.Done()
			slowQuery(ctx, db, n)
		}()
	}
	wg.Wait()
	fmt.Println("all at once:      ", time.Since(start).Round(10*time.Millisecond))
}
```

```
query A finished
query B finished
query C finished
one after another: 610ms
query C finished
query A finished
query B finished
all at once:       210ms
```

The same 3×. Two details deserve attention:

- **The order changed** in the concurrent run (`C, A, B`): concurrent tasks finish in whatever order the timing dictates. *Never assume an order* between goroutines unless you synchronize.
- **`SetMaxOpenConns(10)`** matters. Each concurrent query needs **its own connection** from the pool (Chapter 52). With `MaxOpenConns(1)`, the three queries would queue for the single connection and take 600 ms again, even though we used goroutines. Concurrency is limited by the *narrowest shared resource*.

---

## 5. Concurrency is not parallelism

These words are often confused:

- **Concurrency** is about *structure*: dealing with many things at once (independent tasks that can make progress independently).
- **Parallelism** is about *execution*: doing many things *at the same instant* on several CPU cores.

A concurrent program *may* run in parallel, if the hardware allows. The experiment above (waiting) didn't need parallelism at all: while one goroutine waits, others run, or simply wait too. Now a **CPU-bound** task, which *computes* rather than waits, and how it responds to the number of cores Go may use (`runtime.GOMAXPROCS`):

```go
package main

import (
	"crypto/sha256"
	"fmt"
	"runtime"
	"sync"
	"time"
)

// work is CPU-bound: it never waits, it computes.
func work() {
	sum := sha256.Sum256([]byte("seed"))
	for i := 0; i < 300_000; i++ {
		sum = sha256.Sum256(sum[:])
	}
	_ = sum
}

func run(tasks int) time.Duration {
	start := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < tasks; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			work()
		}()
	}
	wg.Wait()
	return time.Since(start).Round(time.Millisecond)
}

func main() {
	fmt.Println("CPUs:", runtime.NumCPU())
	const tasks = 12

	fmt.Printf("%d tasks, sequentially (one at a time):   ", tasks)
	start := time.Now()
	for i := 0; i < tasks; i++ {
		work()
	}
	seq := time.Since(start).Round(time.Millisecond)
	fmt.Println(seq)

	for _, procs := range []int{1, 2, 4, runtime.NumCPU()} {
		runtime.GOMAXPROCS(procs)
		d := run(tasks)
		fmt.Printf("%d tasks as goroutines, GOMAXPROCS=%-2d:    %v  (%.1fx faster than sequential)\n", tasks, procs, d, float64(seq)/float64(d))
	}
}
```

```
CPUs: 12
12 tasks, sequentially (one at a time):   261ms
12 tasks as goroutines, GOMAXPROCS=1 :    254ms  (1.0x faster than sequential)
12 tasks as goroutines, GOMAXPROCS=2 :    151ms  (1.7x faster than sequential)
12 tasks as goroutines, GOMAXPROCS=4 :    94ms  (2.8x faster than sequential)
12 tasks as goroutines, GOMAXPROCS=12:    59ms  (4.4x faster than sequential)
```

Compare the two experiments:

| Kind of work | 1 CPU core | Many cores |
|--------------|-----------|------------|
| **Waiting** (I/O: sleep, database, network) | already **3× faster** with goroutines: waiting needs no CPU | no further gain |
| **Computing** (CPU-bound) | **no gain** (1.0×): one core can only compute one thing at a time | scales with cores (1.7×, 2.8×, 4.4×) |

So: **goroutines let waiting overlap; cores let computing overlap.** Most server code is waiting-heavy, so goroutines pay off enormously even on small machines, and CPU-heavy code (hashing, image processing, compression) needs *cores*.

Why only 4.4× with 12 "CPUs"? Reasons include: `NumCPU` counts *hardware threads* (two hyper-threads share one core's execution units, so they aren't worth two full cores); the program is short (261 ms, so start-up and scheduling costs matter); other programs share the machine; and CPUs slow down when many cores are active (thermal and power limits). Real-world speedups are rarely linear (**Amdahl's law**: the serial portion limits the total gain).

---

## 6. What a goroutine costs

Why can Go programs create so many goroutines, when creating that many *threads* is impractical? Measure it: start **100,000** goroutines that all stay alive, and ask the runtime what that costs:

```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"strings"
	"sync"
)

func mb(b uint64) float64 { return float64(b) / 1e6 }

// osThreads reads how many operating-system threads this process has (Linux).
func osThreads() string {
	data, _ := os.ReadFile("/proc/self/status")
	for _, line := range strings.Split(string(data), "\n") {
		if strings.HasPrefix(line, "Threads:") {
			return strings.TrimSpace(strings.TrimPrefix(line, "Threads:"))
		}
	}
	return "?"
}

func main() {
	var before, during runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&before)

	const n = 100_000
	var started, release, done sync.WaitGroup
	started.Add(n)
	release.Add(1)
	for i := 0; i < n; i++ {
		done.Add(1)
		go func() {
			defer done.Done()
			started.Done()
			release.Wait() // stay alive until released
		}()
	}
	started.Wait()

	runtime.ReadMemStats(&during)
	fmt.Println("goroutines alive:      ", runtime.NumGoroutine())
	fmt.Println("OS threads used:       ", osThreads())
	fmt.Printf("stack memory in use:    %.1f MB  (%.0f bytes per goroutine)\n",
		mb(during.StackInuse-before.StackInuse), float64(during.StackInuse-before.StackInuse)/n)
	fmt.Printf("total memory from OS:   %.1f MB more than before\n", mb(during.Sys-before.Sys))

	release.Done()
	done.Wait()
}
```

```
goroutines alive:       100001
OS threads used:        13
stack memory in use:    205.2 MB  (2052 bytes per goroutine)
total memory from OS:   272.6 MB more than before
```

- **~2 KB per goroutine.** A goroutine starts with a tiny stack (2 KB) that **grows and shrinks on demand** (the runtime copies it to a bigger block if needed).
- **13 OS threads** served all 100,000 goroutines. The runtime multiplexes many goroutines onto a few threads (section 7).
- 100,000 goroutines took about 270 MB in total (each also has bookkeeping structures, and these goroutines share the closure state), a size you can afford.

An **operating-system thread**, by contrast, typically reserves a stack of **1-8 MB** (the Linux default is 8 MB of *virtual* address space, committed lazily), needs the kernel to create and to switch between (a context switch costs microseconds; a goroutine switch costs tens of nanoseconds), and the OS imposes limits on how many can exist. A design of "one thread per connection" hits a wall at thousands; "one goroutine per connection" scales to hundreds of thousands. That is why Go is popular for network servers.

| | OS thread | Goroutine |
|---|-----------|-----------|
| Initial stack | 1-8 MB (reserved) | ~2 KB (growable) |
| Created by | kernel (system call) | the Go runtime (a function call) |
| Switching | kernel context switch | runtime scheduler (user space) |
| Practical count | thousands | hundreds of thousands to millions |
| Scheduled by | the operating system | the Go runtime |

Goroutines are **cheap, not free**: 100,000 of them are fine; 100 million would not be. And **an unbounded number of goroutines** (one per incoming item, with no limit) is a memory bug waiting to happen; Chapter 69 covers bounding them.

---

## 7. How the scheduler works (a sketch)

The Go runtime has its own scheduler, built on three concepts (the "GMP model"):

```
  G = Goroutine      the unit of work: a function + its small stack
  M = Machine        an OS thread that actually executes code
  P = Processor      a scheduling context holding a queue of runnable Gs;
                     an M needs a P to run Go code.  Count of Ps = GOMAXPROCS
                     (defaults to the number of CPUs)

        P0 [G G G G]        P1 [G G G]        P2 [G G]        ← local run queues of runnable goroutines
         │                    │                 │
         M0 (thread)          M1 (thread)       M2 (thread)   ← each M runs one G at a time
         │                    │                 │
        CPU core             CPU core          CPU core

   global run queue [G G G ...]        idle Ps steal work from busy Ps' queues
```

What happens when you write `go f()`: the runtime creates a G and puts it on the current P's queue. An M running that P picks Gs one by one. When a goroutine **blocks** (waiting for a channel, a mutex, a `time.Sleep`, network I/O), the scheduler parks it and runs another G on the same thread: **the thread never idles while there is runnable work**. When a goroutine makes a *blocking system call*, the runtime hands its P to another thread so that other goroutines keep running. Idle Ps **steal** work from busy ones, balancing the load.

`GOMAXPROCS` therefore limits **how many goroutines run Go code simultaneously**, not how many exist: that is why the CPU experiment above scaled with it and the waiting experiment did not care. (Since Go 1.14 the scheduler is also *preemptive*: a goroutine running a long loop can be interrupted so others get their turn.)

You rarely interact with the scheduler directly. What matters for correctness: the scheduler is **non-deterministic**: the order in which goroutines run varies between runs, so correct programs never depend on it.

---

## 8. `main` does not wait

A goroutine runs independently of the function that started it. A program ends **when `main` returns**, and all other goroutines are terminated abruptly:

```go
package main

import "fmt"

func main() {
	go fmt.Println("hello from a goroutine")
	fmt.Println("main is done")
	// main returns here: the program ends, whether or not other goroutines finished.
}
```

```
main is done
```

(Run it several times: on our machine the goroutine's line *never* appeared, because `main` finished before the new goroutine was even scheduled. On other runs or machines it might appear, which is exactly the problem: **the outcome depends on timing.**)

The tempting hack is `time.Sleep(time.Second)` at the end of `main` "to give goroutines time". It's wrong for a reason you can state precisely: **sleeping guesses at a duration**. Too short and you lose work; too long and you waste time (and it stays wrong the day the machine is slower). We need a way to wait for *completion*, not for *time to pass*. That is `sync.WaitGroup`, the subject of the next two chapters.

---

## 9. The loop-variable trap

A classic Go bug: goroutines started in a loop all seeing the *same* loop variable. Here is a small program and its output under two language versions (the `go` line in `go.mod`):

```go
package main

import (
	"fmt"
	"sort"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	var mu sync.Mutex
	var seen []int

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			mu.Lock()
			seen = append(seen, i) // which i does each goroutine see?
			mu.Unlock()
		}()
	}
	wg.Wait()

	sort.Ints(seen)
	fmt.Println(seen)
}
```

```
go.mod says "go 1.21":   [5 5 5 5 5]
go.mod says "go 1.22":   [0 1 2 3 4]
```

Before Go 1.22 there was **one** variable `i` for the whole loop. The goroutines ran *after* the loop had finished (when `i` was 5), so all five saw `5`. (The mutex there prevents a data race on `seen`, but not this logic bug: correct synchronization does not fix a wrong idea.) Since **Go 1.22**, each iteration gets **its own copy** of `i`, so the code does what it looks like it does. Two consequences:

1. Code in this course targets Go ≥ 1.22 (our `go.mod` says so), so closures in loops are safe.
2. You will still see the old workaround in older code, and it is *still correct*:

```go
for i := 0; i < 5; i++ {
	i := i // a new variable per iteration, shadowing the loop variable
	go func() { use(i) }()
}
// or pass it explicitly:
go func(n int) { use(n) }(i)
```

**Check your `go.mod`'s `go` line** when copying examples from the internet: the semantics depend on it, not just on the installed toolchain.

---

## 10. Concurrency you already use

Look back at the project. **`net/http` starts a new goroutine for every incoming request.** Your handlers have been running concurrently since Chapter 40, which is why:

- the in-memory stores need a `sync.RWMutex` (Chapters 51-56): two requests can touch the slice at the same moment,
- the request-scoped `context` exists (Chapter 46): to cancel work when a client disconnects,
- the database pool has to be sized (Chapter 52): concurrent requests need concurrent connections,
- `go test -race` matters even for "single-threaded-looking" code.

In the terms of section 2, **L is already large** because of these per-request goroutines. What this part of the course adds is concurrency *inside* a request (overlapping independent waits), and the tools to keep it correct: `WaitGroup` (Chapter 65), an understanding of races (Chapter 67), mutexes (Chapter 68), and channels (Chapters 69-70).

A candidate in our own code, for Chapter 70: the paginated list runs `Count` and then `List` one after the other (Chapter 63). Neither needs the other's result, so they could overlap.

---

## 11. When concurrency does not help

Concurrency has costs (complexity, bugs, scheduling overhead), so use it where it pays:

| Situation | Concurrency helps? |
|-----------|--------------------|
| Several **independent waits** (I/O, RPCs, queries) | ✅ total time drops from the sum to the max |
| **CPU-bound** work with **multiple cores** | ✅ up to the core count |
| **Dependent** steps (B needs A's result) | ❌ inherently sequential |
| A **single shared resource** (one DB connection, a lock everyone holds) | ❌ callers queue up anyway |
| Tiny tasks (a few nanoseconds) | ❌ goroutine start/sync costs more than the work |
| Something already fast enough | ❌ don't add complexity you can't justify: measure first (Chapter 62) |

**Amdahl's law** makes the last bullet quantitative: if 90% of a task can be parallelized, the maximum speedup is 10× *no matter how many cores you add*; the serial 10% dominates.

Rule of thumb: **make it correct, make it measurable, then make it concurrent where the measurements say waiting dominates.**

---

## 12. The roadmap for the next chapters

```
Ch 64  why concurrency matters       (this chapter: measuring the payoff)
Ch 65  sync.WaitGroup                wait for goroutines properly (instead of time.Sleep)
Ch 66  WaitGroup internals            how it works underneath (counters, semaphores)
Ch 67  race conditions                what goes wrong when goroutines share data (and how to detect it: -race)
Ch 68  sync.Mutex                     protecting shared data; RWMutex; atomics
Ch 69  channels                       communicating between goroutines; blocking, buffering, deadlock, select
Ch 70  channels in practice           running Count + List concurrently in our service, with cancellation
```

Each chapter follows the same pattern: a small program that **fails or misbehaves**, a measurement or tool that shows *why*, and the fix.

---

## 13. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | `time.Sleep` to wait for goroutines | Lost work or wasted time; flaky | `sync.WaitGroup` (Chapter 65) |
| 2 | Assuming goroutines run in a particular order | Non-deterministic bugs | Synchronize explicitly |
| 3 | Concurrency limited by a 1-connection pool / global lock | No speedup | Size shared resources; remove needless locks |
| 4 | Starting goroutines for tiny tasks | Overhead exceeds benefit | Batch the work |
| 5 | Unbounded goroutines (one per item, no cap) | Memory exhaustion | Worker pools, semaphores (Chapter 69) |
| 6 | Loop-variable capture in old Go versions | All goroutines use the last value | Go ≥ 1.22, or `i := i` |
| 7 | Expecting `main` to wait | Goroutines killed mid-work | Wait explicitly |
| 8 | Sharing variables between goroutines without synchronization | Data races | Chapters 67-69 |
| 9 | Using goroutines to speed up CPU-bound code on one core | No gain | More cores, or better algorithms |
| 10 | Adding concurrency before measuring | Complexity without benefit | Measure, then decide |
| 11 | Ignoring `GOMAXPROCS` in containers | Runtime may see the host's CPUs, not the container's limit | Recent Go versions read cgroup limits; check yours, or set it explicitly |
| 12 | Forgetting that HTTP handlers already run concurrently | Unprotected shared state | Treat all shared state as concurrent |

---

## 14. Interview questions

**Q1. State Little's law and use it.**
`L = λW`: average concurrency equals throughput times latency. To serve 2,000 req/s at 50 ms each you need ~100 requests in flight.

**Q2. Concurrency vs. parallelism?**
Concurrency is composing independent tasks so they *can* make progress independently; parallelism is executing them simultaneously on multiple cores. Concurrency can exist on one core.

**Q3. Why are goroutines cheaper than threads?**
Small growable stacks (~2 KB vs. MBs), creation and switching in user space by the Go scheduler, and multiplexing many goroutines onto few OS threads.

**Q4. What do G, M and P stand for?**
Goroutine, Machine (OS thread), Processor (scheduling context with a run queue; count = `GOMAXPROCS`).

**Q5. What happens to running goroutines when `main` returns?**
The program exits and they're terminated immediately.

**Q6. What changed for loop variables in Go 1.22?**
Each iteration now has its own copy of the loop variable, fixing the classic "all goroutines see the last value" bug.

**Q7. When does adding goroutines not improve performance?**
Dependent steps, CPU-bound code on one core, a saturated shared resource, or tasks smaller than the concurrency overhead.

**Q8. Why did three 200 ms waits take 200 ms, not 600?**
They were independent and *waiting*, so they overlapped; total time is the maximum, not the sum.

**Q9. What limits how many goroutines run Go code at once?**
`GOMAXPROCS` (the number of Ps), by default the number of CPUs.

---

## 15. Exercises

### Exercise 1: Little's law
A service handles 800 requests/second with a mean latency of 120 ms. (a) How many requests are in flight on average? (b) If each holds 5 MB, how much memory? (c) If a code change cuts latency to 60 ms at the same throughput, what changes?

<details><summary>Solution</summary>

(a) `L = 800 × 0.120 = 96`. (b) `96 × 5 MB = 480 MB`. (c) `L = 800 × 0.060 = 48`: half the concurrency, so half the memory, or the same memory could serve twice the throughput.
</details>

### Exercise 2: Measure your machine
Run the CPU experiment on your machine. What is `runtime.NumCPU()`? At which `GOMAXPROCS` does the speedup stop improving? Try 24 tasks instead of 12.

<details><summary>Solution</summary>

Results vary; expect speedup to rise until it reaches roughly the number of *physical* cores (a bit beyond with hyper-threads), then flatten. More tasks than cores just queue behind each other: the total work is the same, so time barely changes.
</details>

### Exercise 3: Break the speedup
In the `pg_sleep` program set `db.SetMaxOpenConns(1)`. What is the "all at once" time now, and why?

<details><summary>Solution</summary>

About 600 ms: the three goroutines all ask the pool for a connection, but only one exists, so the queries run one after another. Concurrency is capped by the narrowest shared resource.
</details>

### Exercise 4: The cost of a goroutine, cheaper
Change the 100,000-goroutine program to 1,000,000. What happens to memory and to the number of OS threads? At what point would you stop?

<details><summary>Solution</summary>

Memory scales roughly linearly (~2-2.7 KB each → ~2-3 GB), while OS threads stay in the low tens. You'd stop long before that in a real service: bound concurrency to what the *work* justifies (workers ≈ available CPU/connection capacity), not to what goroutines *permit*.
</details>

### Exercise 5: Loop-variable archaeology
Find a place in older Go code you know (or write one) that uses `i := i` or passes the loop variable as an argument. Change the module's `go` line to 1.22 and remove the workaround. Does the behavior stay identical? Then change `go` back to 1.21: what breaks?

<details><summary>Solution</summary>

With `go 1.22+` the shadowing line is redundant but harmless; with `go 1.21` and the workaround removed, closures capture the shared variable and produce the wrong values, exactly as in section 9.
</details>

### Exercise 6 (challenge): Predict the limit
You run three queries, 200 ms each, but the pool allows 2 connections. Predict the total time for the concurrent version, then verify.

<details><summary>Solution</summary>

Two queries run together (200 ms), the third waits for a free connection and then runs (another 200 ms): about **400 ms**. Generally: `ceil(tasks / connections) × taskTime`.
</details>

---

## 16. Quiz

1. What does Little's law say?
2. Why did the waiting experiment speed up on one core but the CPU experiment did not?
3. About how big is a new goroutine's stack?
4. What does `GOMAXPROCS` control?
5. What happens if `main` returns while goroutines are running?
6. What did Go 1.22 change about loop variables?
7. Why is `time.Sleep` the wrong way to wait for goroutines?
8. Name two reasons a concurrent version might not be faster.

<details><summary>Answers</summary>

1. `L = λ × W`: concurrency = throughput × latency.
2. Waiting doesn't need a CPU, so many waits can overlap on one core; computing needs a core for the whole duration.
3. About 2 KB (growable).
4. The number of goroutines that may run Go code simultaneously (the number of Ps).
5. The program exits; the goroutines are killed.
6. Each loop iteration now has its own copy of the loop variable.
7. It guesses a duration instead of waiting for completion: too short loses work, too long wastes time.
8. Dependent steps, a saturated shared resource (e.g., one connection), CPU-bound work with one core, tasks too small to justify the overhead.
</details>

---

## 17. Summary

- **Little's law** (`L = λW`) ties concurrency, throughput, and latency together; it predicted our measured throughput within 2%, and it turns capacity planning into arithmetic.
- **Waiting overlaps**: three independent 200 ms tasks took **600 ms in a line and 200 ms concurrently**, with sleeps *and* with real database queries (610 ms vs 210 ms). Goroutines gave the gain even on one core; **CPU-bound** work only speeds up with **more cores** (1.0× → 4.4× on our 12 hardware threads).
- A **goroutine costs ~2 KB** and 100,000 of them ran on **13 OS threads**; the scheduler (G, M, P) multiplexes them, parks blocked goroutines, and steals work between processors. `GOMAXPROCS` bounds *parallelism*, not the number of goroutines.
- **`main` does not wait**; `time.Sleep` is a guess. We need synchronization (next chapter).
- **Go 1.22** fixed the loop-variable capture bug (`[5 5 5 5 5]` → `[0 1 2 3 4]`), governed by the `go` line in `go.mod`.
- `net/http` already gives every request its own goroutine, so shared state in handlers must be protected. Concurrency is not always faster: dependent steps, one shared resource, or tiny tasks gain nothing; **measure first**.

### ➡️ What's next?

[Chapter 65](65-sync-waitgroup.md) replaces the `time.Sleep` guess with **`sync.WaitGroup`**: how to wait for a group of goroutines properly, the rules for `Add`, `Done`, and `Wait`, the mistakes that cause panics and hangs, and how to test it.
