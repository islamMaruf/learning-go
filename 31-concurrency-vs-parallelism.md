# Chapter 31: Concurrency vs. Parallelism — The Multi-Core Revolution

> **Goal of this chapter:** Learn the difference between two words that are constantly confused, **concurrency** and **parallelism**, and understand the hardware that makes parallelism possible: **cores**, **logical CPUs (hyper-threading)**, and why the industry went multi-core. You'll measure real speedups in Go and see a case where *concurrency without parallelism* is exactly what you want.

**Difficulty:** 🔴 Intermediate–Advanced (conceptual + measurements)  **Estimated time:** 2.5–3 hours  **Prerequisite:** [Chapter 30](30-context-switching-the-pcb-and-the-magic-of-concurrency.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Quick recap](#2-quick-recap)
3. [The single-CPU era](#3-the-single-cpu-era)
4. [Why the industry went multi-core](#4-why-the-industry-went-multi-core)
5. [Cores and logical CPUs](#5-cores-and-logical-cpus)
6. [See your own CPU](#6-see-your-own-cpu)
7. [Concurrency: dealing with many things](#7-concurrency-dealing-with-many-things)
8. [Parallelism: doing many things](#8-parallelism-doing-many-things)
9. [Comparison table](#9-comparison-table)
10. [Analogies](#10-analogies)
11. [Experiment 1: parallel speedup in Go](#11-experiment-1-parallel-speedup)
12. [Experiment 2: concurrency without parallelism](#12-experiment-2-concurrency-without-parallelism)
13. [Limits of parallelism (Amdahl's law)](#13-limits-of-parallelism)
14. [What to use when: CPU-bound vs. I/O-bound](#14-cpu-bound-vs-io-bound)
15. [Common misconceptions](#15-common-misconceptions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- Why CPUs stopped getting faster in clock speed and started getting **more cores**
- What a **core** is, what a **logical CPU / hardware thread** is, and how to read your own CPU's specs
- The precise definitions of **concurrency** and **parallelism**
- How to decide which one your problem needs
- How to see both in action with real Go code and real timings
- Why "more goroutines" doesn't always mean "faster"

---

## 2. Quick recap

From the last two chapters:

- The OS uses **context switching** to share a CPU among many processes by rapid switching, giving the illusion of simultaneity.
- Each switch saves the running process's registers into its **PCB** and loads another's.
- On **one core**, only **one** instruction stream actually runs at any instant.

That's the world of the 1970s–early 2000s. Then hardware changed.

---

## 3. The single-CPU era

For decades, chips had **one core**. Programs got faster mainly because each new chip generation had a **higher clock speed** (from MHz to GHz). Software authors did nothing: *the same program simply ran faster on next year's computer*. This was sometimes called "the free lunch."

```
1990: ~ 33 MHz      2000: ~ 1 GHz       2004: ~ 3.8 GHz     2024: ~ 3–5 GHz  (barely moved!)
```

Multitasking existed (Chapter 30), but it was always **concurrency by time-slicing on one core**: only one thing ran at a time.

---

## 4. Why the industry went multi-core

Around **2004–2005**, clock speeds stopped climbing. The reason is physics:

- Power (and therefore heat) grows **faster than linearly** with clock frequency.
- Around 4 GHz, chips became too hot to cool economically ("the power wall").
- Transistors kept shrinking (Moore's law: roughly double per ~2 years), so there was plenty of **space**, but no way to make one core much faster.

The industry's answer: **use the extra transistors to build more cores** on one chip. Instead of one very fast worker, put several ordinary workers on the same chip.

```
Before:   [ ONE fast core ]           →  faster clock each year
After:    [Core][Core][Core][Core]    →  more cores each year
```

**Consequence for programmers ("The free lunch is over"):** a single-threaded program doesn't automatically use extra cores. *To go faster on modern hardware, software must be written to do multiple things at once.* This is the environment **Go was designed for** (2007–2009): make concurrent, multi-core programming simple and safe. That's why the second half of this course is about goroutines and channels.

---

## 5. Cores and logical CPUs

### Core

A **core** is a complete, independent processing unit: its own **control unit, ALU, and register set** (PC, IR, SP, BP, general registers; Chapter 28). Each core can run **one instruction stream at a time**.

```
┌───────────────────── One CPU chip (package) ─────────────────────┐
│   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐              │
│   │ Core 1 │   │ Core 2 │   │ Core 3 │   │ Core 4 │   ...        │
│   │ CU ALU │   │ CU ALU │   │ CU ALU │   │ CU ALU │              │
│   │ regs   │   │ regs   │   │ regs   │   │ regs   │              │
│   └────────┘   └────────┘   └────────┘   └────────┘              │
│           shared L3 cache + memory controller                    │
└──────────────────────────────────────────────────────────────────┘
```

4 cores → up to **4 instruction streams truly at the same instant**.

### Logical CPUs (hyper-threading / SMT)

Many chips run **two hardware threads per core** (Intel calls it *Hyper-Threading*; the general name is **SMT**, simultaneous multithreading). To the operating system, one physical core then looks like **two logical CPUs**.

Why? A core often sits partly idle: while one instruction stream waits for data from RAM (a "cache miss"), the core's execution units have nothing to do. SMT gives the core a **second set of registers/state** so it can *interleave two streams* and keep its execution units busier.

```
Physical core (SMT, 2 threads):
   ┌────────────────────────────────────────────┐
   │  Thread A state (own PC, SP, BP, regs)     │ ← logical CPU 1
   │  Thread B state (own PC, SP, BP, regs)     │ ← logical CPU 2
   │  ─── shared: ALUs, caches, execution units ─│
   └────────────────────────────────────────────┘
```

> ⚠️ **Precision note.** A simplified description says each logical CPU has its *own* ALU and control unit. In reality, the two logical CPUs on a core **duplicate the register state** but **share the execution resources** (ALUs, caches). That's why hyper-threading gives roughly **+15–30%** throughput on suitable workloads, *not* 2×. "Logical" CPUs are real hardware threads, not fake, but they aren't full extra cores.

> **Analogy: a bank.** A physical **core** is a teller *window*. **Hyper-threading** is one teller who can keep two customers' paperwork open at once, so that when a customer goes off to fetch a document, the teller continues with the other. One teller, two customers in flight, but not double the speed.

### Terminology cheat sheet

| Term | Meaning |
|------|---------|
| **Socket / package** | One physical chip |
| **Core** (physical core) | An independent processing unit |
| **Logical CPU / hardware thread / vCPU** | What the OS schedules onto; = cores × threads-per-core |
| **Processor** (loosely) | Often means logical CPU in Task Manager/`nproc` |

**Example:** a 10-core chip with hyper-threading on some cores might show **12 logical processors**. (The machine used to test this chapter's code: an Intel Core i7-1255U: 2 performance cores with 2 threads each + 8 efficiency cores with 1 thread each = **10 cores, 12 logical CPUs**.)

---

## 6. See your own CPU

**Linux**

```bash
lscpu | grep -E 'Model name|^CPU\(s\)|Thread|Core\(s\)|Socket'
nproc
```

Real output (the test machine):

```
CPU(s):                12
Model name:            12th Gen Intel(R) Core(TM) i7-1255U
Thread(s) per core:    2
Core(s) per socket:    10
Socket(s):             1
```

**Windows:** Task Manager (`Ctrl+Shift+Esc`) → *Performance* → *CPU*: shows **Cores**, **Logical processors**. The CPU graph can be switched to "Logical processors" to see each one's usage.

**macOS:** Activity Monitor → *Window → CPU History*; or `sysctl hw.physicalcpu hw.logicalcpu`.

**From Go:**

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println("logical CPUs:", runtime.NumCPU())
	fmt.Println("GOMAXPROCS:  ", runtime.GOMAXPROCS(0)) // 0 = just report the current value
}
```

`runtime.NumCPU()` returns **logical CPUs**. `GOMAXPROCS` is the **maximum number of OS threads that can execute Go code simultaneously**. It defaults to the number of logical CPUs. (More in Chapter 36.)

---

## 7. Concurrency: dealing with many things

> **Concurrency** is about **dealing with** many things at once: *structuring* a program so that multiple tasks can be **in progress** during overlapping time periods.

It doesn't require multiple cores. On a single core, the system **interleaves** tasks by switching among them:

```
ONE core, three tasks A, B, C (all "in progress" together):

Time ─────────────────────────────────────────────►
Core 1: [A][B][C][A][B][C][A][B][C]
```

At any *instant* only one task executes, but **all three make progress over time**. It's like a chef alone in a kitchen: while the soup simmers (A is waiting), he chops vegetables (B), then stirs the soup, then checks the oven. He's *concurrently* handling three dishes, though his hands do one thing at a time.

**Concurrency is a property of the program's structure** ("these parts are independent and may interleave"). It's about *design*.

---

## 8. Parallelism: doing many things

> **Parallelism** is about **doing** many things **at the same instant**: literally running multiple computations simultaneously on **multiple cores**.

```
THREE cores, three tasks A, B, C:

Time ─────────────────────────────────────────────►
Core 1: [A][A][A][A][A][A]
Core 2: [B][B][B][B][B][B]
Core 3: [C][C][C][C][C][C]
```

Three chefs in three kitchens, each cooking a dish at the same moment.

**Parallelism is a property of the execution** (the hardware is genuinely doing simultaneous work). It's about *speed*.

### The famous line

> *"Concurrency is about **dealing with** lots of things at once. Parallelism is about **doing** lots of things at once."*
> — Rob Pike (co-creator of Go), from his talk "Concurrency is not parallelism"

Concurrency **enables** parallelism: if you structure a program as independent concurrent pieces, the runtime can run them in parallel when cores are available. But you can have either without the other:

|  | **Not parallel** | **Parallel** |
|--|------------------|--------------|
| **Not concurrent** | A plain sequential program on one core | (SIMD on one task: a rare edge case) |
| **Concurrent** | Tasks interleaved on **one core** | Tasks interleaved **and** running on several cores: what Go programs do on multi-core machines |

---

## 9. Comparison table

| | **Concurrency** | **Parallelism** |
|--|-----------------|-----------------|
| Definition | Managing many tasks that overlap in time | Running many tasks at the same instant |
| Needs multiple cores? | ❌ No | ✅ Yes |
| Mechanism | Interleaving / context switching | Simultaneous execution on separate cores |
| Main goal | **Structure**, responsiveness, handling waiting | **Speed** / throughput for heavy computation |
| Best for | I/O-bound work (network, disk, waiting) | CPU-bound work (number crunching) |
| Is it "at the same time"? | Appears so (interleaved) | Truly |
| Go tools | Goroutines, channels, `select` | Same tools + `GOMAXPROCS` > 1 (automatic) |

---

## 10. Analogies

**Chefs and kitchens 👨‍🍳**

- *Sequential*: one chef, one dish at a time, standing idle while water boils.
- *Concurrent*: one chef, three dishes: starts the soup, and while it simmers prepares the salad, checks the oven, returns to the soup. Everything progresses; one pair of hands.
- *Parallel*: three chefs, each cooking one dish. Dishes finish about three times sooner.
- *Concurrent **and** parallel*: three chefs, each juggling several dishes.

**Boss and workers 👔👷**

A boss (the OS or Go's scheduler) hands out tasks. With **one worker**, tasks are done in turns (concurrency). With **four workers**, four tasks are done at once (parallelism).

**A person on the phone 📞**

Talking to one caller and *waiting* while they look something up, then checking email in the meantime: concurrent. Two people on two phones simultaneously: parallel.

---

## 11. Experiment 1: parallel speedup

Let's *see* parallelism: a **CPU-bound** job (counting primes below six million) split among goroutines, run with different numbers of cores.

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func countPrimes(lo, hi int) int {
	n := 0
	for i := lo; i < hi; i++ {
		if i < 2 {
			continue
		}
		isPrime := true
		for d := 2; d*d <= i; d++ {
			if i%d == 0 {
				isPrime = false
				break
			}
		}
		if isPrime {
			n++
		}
	}
	return n
}

func run(workers, limit int) (int, time.Duration) {
	var wg sync.WaitGroup
	results := make([]int, workers)
	chunk := limit / workers
	start := time.Now()
	for w := 0; w < workers; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			lo, hi := w*chunk, (w+1)*chunk
			if w == workers-1 {
				hi = limit
			}
			results[w] = countPrimes(lo, hi)
		}()
	}
	wg.Wait()
	total := 0
	for _, r := range results {
		total += r
	}
	return total, time.Since(start)
}

func main() {
	const limit = 6_000_000
	for _, procs := range []int{1, 2, 4, 8} {
		runtime.GOMAXPROCS(procs) // allow this many cores
		total, d := run(procs*4, limit)
		fmt.Printf("GOMAXPROCS=%d: primes=%d time=%v\n", procs, total, d.Round(time.Millisecond))
	}
}
```

*(This program uses Go 1.22's per-iteration loop variables, so `w` in the goroutine is safe: see Chapter 20. It also uses `sync.WaitGroup`, formally introduced in Chapter 65.)*

Real output on the 12-logical-CPU test machine:

```
GOMAXPROCS=1: primes=412849 time=2.015s
GOMAXPROCS=2: primes=412849 time=1.06s
GOMAXPROCS=4: primes=412849 time=576ms
GOMAXPROCS=8: primes=412849 time=363ms
```

Same answer every time (`412849` primes), but:

| Cores | Time | Speedup vs. 1 core |
|-------|------|--------------------|
| 1 | 2.015 s | 1.0× |
| 2 | 1.06 s | 1.9× |
| 4 | 0.576 s | 3.5× |
| 8 | 0.363 s | 5.6× |

That's **parallelism**: more cores, less wall-clock time. (Not perfectly linear: 8 cores gave 5.6×, not 8×. Section 13 explains why.)

---

## 12. Experiment 2: concurrency without parallelism

Now the *opposite*: a job that's mostly **waiting**. We start **1,000 goroutines that each wait 100 ms** (imagine 1,000 network requests) but restrict Go to **one core**:

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {
	runtime.GOMAXPROCS(1) // ONE core: no parallelism at all

	var wg sync.WaitGroup
	start := time.Now()
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			time.Sleep(100 * time.Millisecond) // pretend to wait for the network
		}()
	}
	wg.Wait()
	fmt.Println("took", time.Since(start).Round(10*time.Millisecond))
}
```

Real output:

```
took 100ms
```

**1,000 tasks × 100 ms = 100 seconds if done one after another, yet it finished in about 100 ms on one core.** No parallel execution occurred; all 1,000 goroutines were *waiting* at the same time, and waiting costs no CPU. That is **concurrency**: dealing with many things at once by overlapping their waiting.

A web server works this way: 10,000 clients connected, almost all of them idle at any instant, served by a handful of cores.

---

## 13. Limits of parallelism

Why did 8 cores give only 5.6×?

1. **Not everything can be parallelized.** Some steps (splitting the work, adding up the results, waiting for a common resource) run serially. **Amdahl's law:** if a fraction *s* of the work is inherently serial, the maximum speedup with *N* cores is

   ```
   speedup ≤ 1 / ( s + (1 − s)/N )
   ```
   If 10% of the work is serial (`s = 0.1`), even infinite cores can't beat **10×**; with 8 cores: `1 / (0.1 + 0.9/8) ≈ 4.7×`.

2. **Coordination costs.** Goroutines/threads must synchronize, share memory, and wait for one another (locks, channels; Chapters 67–70).
3. **Hardware sharing.** Cores share caches, memory bandwidth, and (with SMT) execution units. The last cores in our test were hyper-threads and efficiency cores, which are slower than the main cores.
4. **Uneven work.** Chunks of primes near the top of the range cost more than those near the bottom, so some workers finish earlier than others and then sit idle.
5. **Overheads.** Starting goroutines, scheduling, and merging results aren't free.

**Practical lesson:** measure; don't assume. And the problem must be *divisible into independent pieces* to benefit.

---

## 14. CPU-bound vs. I/O-bound

| | CPU-bound | I/O-bound |
|--|-----------|-----------|
| The limit is | Speed of computation | Waiting for disk/network/user |
| Examples | Image processing, compression, prime search, ML | Web servers, API calls, database queries, file downloads |
| Helped by | **Parallelism** (more cores) | **Concurrency** (overlapping the waits) |
| More goroutines than cores? | Doesn't help, often slightly hurts | Helps a lot (thousands are fine) |
| Go approach | ~One worker per core, split the data | A goroutine per request/connection |

The e-commerce backend we'll build is **I/O-bound** (waiting on clients and the database), so Go's cheap concurrency shines there. A video encoder is **CPU-bound**, so Go's ease of parallelism shines there.

---

## 15. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "Concurrency and parallelism are the same" | Concurrency = structure/interleaving; parallelism = simultaneous execution |
| "More threads/goroutines = always faster" | Only if there's independent work *and* cores (or waiting to overlap); otherwise you add overhead |
| "A 4-core CPU with hyper-threading is an 8-core CPU" | It has 4 cores / 8 hardware threads that share execution units (≈ +15–30%, not 2×) |
| "Logical CPUs are fake / not real" | They're real hardware threads, but not full duplicate cores |
| "Concurrency requires multiple cores" | It works on a single core via interleaving |
| "Parallel programs are always deterministic" | Concurrent access to shared data creates **race conditions** (Chapter 67) |
| "A single-threaded program uses all cores" | It uses one; you must structure it concurrently to use more |
| "Go automatically parallelizes my code" | It *schedules* goroutines across cores; **you** decide what runs concurrently |

---

## 16. Exercises

### Exercise 1: Classify
Concurrency, parallelism, both, or neither?

(a) One core alternates between a music player and a browser. (b) Four cores each compress a different file. (c) A single thread runs a `for` loop. (d) A web server on 8 cores handles 5,000 connections.

<details><summary>Solution</summary>

(a) Concurrency only. (b) Parallelism (and structured concurrently). (c) Neither. (d) Both: concurrency (5,000 in-flight connections) and parallelism (8 cores executing at once).
</details>

### Exercise 2: Count your CPU
Run `runtime.NumCPU()`. Compare with your OS's reported cores and logical processors. Which does `NumCPU` match?

<details><summary>Solution</summary>

It matches the number of **logical** processors (hardware threads).
</details>

### Exercise 3: Predict the timing
You have 4 cores and 8 tasks that each need exactly 1 s of pure CPU work. Roughly how long does it take (ideally) with (a) 1 worker (b) 4 workers (c) 8 workers?

<details><summary>Solution</summary>

(a) 8 s. (b) 2 s (two rounds of 4). (c) about 2 s: with only 4 cores, eight CPU-bound workers can't go faster than four; the extra are time-sliced on the same cores.
</details>

### Exercise 4: Amdahl
20% of a program is serial. What's the maximum speedup on 4 cores? On 16 cores? On infinite cores?

<details><summary>Solution</summary>

`1/(0.2 + 0.8/4) = 2.5×`; `1/(0.2 + 0.8/16) = 4.0×`; limit `1/0.2 = 5×`.
</details>

### Exercise 5: Run experiment 2 with I/O
Change experiment 2's sleep to a *busy loop* of about 100 ms (CPU work) instead of `time.Sleep`. Predict how long 1000 goroutines take on 1 core, and on all cores.

<details><summary>Solution</summary>

CPU work can't be overlapped on one core: 1000 × 100 ms ≈ **100 s** on one core, and about `100 s / cores` (e.g., ~8–10 s on 12 logical CPUs). This shows that goroutines make *waiting* cheap, but not *computing*.
</details>

### Exercise 6: Design
You must download 200 web pages and then compress them. Which part is I/O-bound, which CPU-bound, and how would you structure the goroutines?

<details><summary>Solution</summary>

Downloading is I/O-bound → launch many goroutines (e.g., dozens or hundreds at once, perhaps limited by a worker pool). Compression is CPU-bound → use about `runtime.NumCPU()` workers. A pipeline (download workers → channel → compress workers) handles both (Chapters 69–70).
</details>

### Exercise 7 (challenge): Measure it yourself
Take experiment 1, add `GOMAXPROCS` values 12 and 24. Does 24 help? Plot or tabulate your results and explain the shape.

<details><summary>Solution</summary>

Expect improvement up to your logical CPU count, then a plateau: extra workers have no extra cores to run on. Speedup is sub-linear because of serial parts, shared caches, uneven chunks, and slower hyper-threads/efficiency cores.
</details>

---

## 17. Quiz

1. Why did CPU makers switch from higher clock speeds to more cores?
2. What is a logical CPU?
3. Can concurrency happen on a single core?
4. Which of concurrency/parallelism speeds up a CPU-bound job? Which helps I/O-bound work?
5. What does `runtime.GOMAXPROCS` control?
6. Who said, "Concurrency is not parallelism"?

<details><summary>Answers</summary>

1. Heat/power limits stopped clock scaling, while shrinking transistors provided space for more cores.
2. A hardware thread the OS can schedule onto; with SMT, one physical core provides two.
3. Yes, via interleaving/context switching.
4. Parallelism speeds CPU-bound work; concurrency (overlapping waits) helps I/O-bound.
5. The maximum number of OS threads that execute Go code simultaneously.
6. Rob Pike, one of Go's designers.
</details>

---

## 18. Summary

- Clock speeds stalled around 2004 (heat/power), so chips gained **more cores**. Software now must be **concurrent** to use them: the world Go was made for.
- A **core** has its own CU, ALU and registers; a **logical CPU** (hyper-thread) is a second register set on a core that *shares* execution units (≈ +15–30%, not 2×).
- **Concurrency** = *dealing with many things at once* (structure, interleaving, works on one core). **Parallelism** = *doing many things at once* (simultaneous execution on multiple cores).
- Parallelism speeds up **CPU-bound** work (measured: 2.0 s → 0.36 s across 1 → 8 cores); concurrency makes **I/O-bound** work cheap (1,000 × 100 ms waits finished in 100 ms on one core).
- Speedups are limited (**Amdahl's law**, coordination costs, shared hardware).
- In Go you *structure* the program with goroutines; the runtime spreads them over cores (`GOMAXPROCS`).

### ➡️ What's next?

We keep saying "OS thread". [Chapter 32](32-threads.md) explains exactly what a **thread** is, how it differs from a process, and why threads are the bridge to goroutines.
