# Chapter 67: Race Conditions — When Goroutines Share Data

> **Goal of this chapter:** See, with real output, what goes wrong when goroutines read and write the same memory without coordination. A counter that loses more than half its increments, an account that pays out $300 from a $100 balance, a slice that drops elements, a map that crashes the whole program, and an `err` variable that **silently erases a database failure**. You'll learn the precise definition of a **data race**, why `counter++` is three steps and not one, how to use and read the **race detector** (`go test -race`), what Go's **memory model** actually promises (the *happens-before* relation), why "benign" races don't exist, and the strategies for avoiding shared mutable state in the first place. The fixes (mutexes) are the next chapter.

**Difficulty:** 🔴 Advanced  **Estimated time:** 6 hours  **Prerequisite:** [Chapters 64-66](66-inside-sync-waitgroup.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Where we are](#2-where-we-are)
3. [What is a race condition?](#3-what-is-a-race-condition)
4. [The bank account](#4-the-bank-account)
5. [The counter: losing 57% of the work](#5-the-counter-losing-57-of-the-work)
6. [Why: `counter++` is three steps](#6-why-counter-is-three-steps)
7. [Why goroutines can share memory at all](#7-why-goroutines-can-share-memory-at-all)
8. [The race detector](#8-the-race-detector)
9. [Other faces of the same bug](#9-other-faces-of-the-same-bug)
10. [It works on my machine: visibility](#10-it-works-on-my-machine-visibility)
11. [The memory model in one page](#11-the-memory-model-in-one-page)
12. [Why races are dangerous](#12-why-races-are-dangerous)
13. [Strategies for avoiding races](#13-strategies-for-avoiding-races)
14. [Races in our own project](#14-races-in-our-own-project)
15. [Common mistakes](#15-common-mistakes)
16. [Interview questions](#16-interview-questions)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. What you will learn

- The definition of a **data race** and how it differs from a general **race condition**
- Why "`x++`" is not atomic, with a step-by-step interleaving
- **Check-then-act** bugs (the bank account)
- Reading a **race detector** report, and its costs and limits
- Concurrent **map** writes (fatal), **slice** appends (lost data), shared **error variables** (silently lost errors)
- Why code that "works" can still be broken: **visibility** and the compiler
- The **happens-before** relation and the synchronization events Go guarantees
- Design strategies: don't share, confine, copy, own-slot, synchronize

---

## 2. Where we are

Chapter 64 showed that goroutines make waiting overlap; Chapter 65 showed how to *wait* for them. Every example so far avoided one thing: **two goroutines touching the same variable**. Real programs can't always avoid it: a shared counter, a cache, a slice of results, the in-memory stores from Chapters 51-56. This chapter shows what happens when that goes unprotected. It is the most important safety topic in concurrent programming, because these bugs are **silent, intermittent, and data-corrupting**.

---

## 3. What is a race condition?

Two related terms:

> A **data race** occurs when **two or more goroutines access the same memory location concurrently, at least one of the accesses is a write, and there is no synchronization ordering them.**

> A **race condition** is the broader class of bugs where **the outcome depends on the timing** of concurrent operations. (You can have a race condition with perfectly synchronized memory accesses: for example, a "check, then act" sequence that is atomic step by step but not as a whole. And you can, in principle, have a data race that doesn't change the outcome, though Go's memory model gives no guarantee of that.)

Ingredients of a data race, all required:

```
  1. shared memory        (a variable both goroutines can reach)
  2. concurrent access    (no forced order between the accesses)
  3. at least one write   (two reads are always safe)
  4. no synchronization   (no mutex, channel, WaitGroup, atomic between them)
```

Remove any one and the race disappears. Chapters 68-70 remove #4; Section 13 shows how to remove #1 instead.

---

## 4. The bank account

Analogy first. An account holds $100. Five people each try to withdraw $60 at the same moment. The rule is *"you may only withdraw what's in the account"*. The teller's procedure:

```
1. look at the balance            (check)
2. if balance >= amount ...
3.    hand over the money and reduce the balance   (act)
```

Each step is fine alone. But if all five tellers do step 1 before any of them does step 3, **all five see $100, all five approve, and $300 leaves an account holding $100.** In Go:

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Account struct{ balance int }

// Withdraw checks the balance and THEN subtracts: two separate steps.
func (a *Account) Withdraw(amount int) bool {
	if a.balance >= amount { // check
		time.Sleep(time.Millisecond) // (real code: a database call, a log line, any work...)
		a.balance -= amount // act
		return true
	}
	return false
}

func main() {
	acc := &Account{balance: 100}
	var wg sync.WaitGroup
	var mu sync.Mutex
	succeeded := 0

	for i := 0; i < 5; i++ { // five people each try to withdraw 60 from an account holding 100
		wg.Add(1)
		go func() {
			defer wg.Done()
			if acc.Withdraw(60) {
				mu.Lock()
				succeeded++
				mu.Unlock()
			}
		}()
	}
	wg.Wait()

	fmt.Println("withdrawals of 60 that succeeded:", succeeded, "(the account only held 100)")
	fmt.Println("final balance:", acc.balance)
}
```

```
withdrawals of 60 that succeeded: 5 (the account only held 100)
final balance: -200
```

**Five withdrawals succeeded and the balance is −200.** No crash, no error, just money created out of nothing. (The `time.Sleep` widens the window between check and act to make the bug appear every time; without it, it would still exist but appear only occasionally, which is *worse*: it would pass your tests and fail in production under load. Note also that the tiny `succeeded` counter *is* protected by a mutex; the bug isn't about that counter.)

This is a **check-then-act race**: each individual access to `balance` is small, but the *decision* made from the first read is stale by the time the write happens. Fixing it requires making check-and-act **one indivisible unit** (Chapter 68), which is exactly what the `UPDATE ... SET stock = stock - 1 ... CHECK (stock >= 0)` in Chapter 55 achieves inside the database.

---

## 5. The counter: losing 57% of the work

A thousand goroutines each increment a shared counter a thousand times. The answer must be 1,000,000:

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counter := 0
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				counter++ // read, add one, write back: THREE steps, not one
			}
		}()
	}
	wg.Wait()

	fmt.Println("expected:", 1000*1000)
	fmt.Println("actual:  ", counter)
}
```

Four runs of the same program:

```
expected: 1000000
actual:   434973
actual:   387350
actual:   449096
actual:   420188
```

**Fewer than half the increments survived**, and the answer is *different every run*. Note what the program did *not* do: crash, print an error, or hang. It calmly returned a wrong number. `WaitGroup` was used perfectly (all goroutines completed): correct waiting does not fix incorrect sharing.

---

## 6. Why: `counter++` is three steps

To the programmer, `counter++` is one operation. To the CPU it is (at least) three:

```
1. LOAD   counter from memory into a register        r = counter
2. ADD    1 to the register                          r = r + 1
3. STORE  the register back to memory               counter = r
```

Two goroutines, A and B, and `counter = 5`. One unlucky interleaving:

```
time ─►
 A:  LOAD (r=5)                ADD (r=6)   STORE counter=6
 B:            LOAD (r=5)                              ADD (r=6)   STORE counter=6

 counter started at 5, two increments happened, the result is 6, not 7.   ← one update LOST
```

B loaded the value *before* A stored its result, so B's write **overwrites** A's. This is a **lost update**. With a thousand goroutines and a million operations, this happens constantly, which is why more than half the increments vanished.

The instruction sequence isn't an implementation detail you can outsmart: even on one CPU core the OS/runtime can pause a goroutine *between* any two instructions; on multiple cores, two goroutines really do run these steps at the same instant.

---

## 7. Why goroutines can share memory at all

Where does the shared `counter` live, and why can every goroutine reach it? Recall the layout of a process:

```
 ┌─────────────────────────────┐
 │  code (the program)          │
 ├─────────────────────────────┤
 │  globals / static data       │  ← package-level variables: shared by ALL goroutines
 ├─────────────────────────────┤
 │  heap                        │  ← pointers, slices, maps, values that escape (Chapter 18): shared
 ├─────────────────────────────┤
 │  goroutine 1 stack           │  ← each goroutine's own local variables (private... until shared)
 │  goroutine 2 stack           │
 │  goroutine 3 stack           │
 └─────────────────────────────┘
```

All goroutines of a program live in the **same address space**. Local variables are *private*, until you share them, and Go makes sharing very easy:

- a **closure** captures variables **by reference**: the `go func() { counter++ }()` above *shares* `counter` with `main` and with every sibling goroutine (which is why the compiler moves `counter` to the heap: it "escapes"),
- passing a **pointer**, **slice**, **map**, or **channel** to a goroutine shares what it points to,
- **package-level variables** are shared by everyone.

So sharing is the *default* whenever you reference something from two goroutines. The discipline is to notice it.

---

## 8. The race detector

Go ships a **dynamic race detector** based on ThreadSanitizer. Build or test with `-race` and every memory access is instrumented; when two accesses conflict without a happens-before relationship, it prints a report. Running the counter program with `go run -race .`:

```
==================
WARNING: DATA RACE
Read at 0x00c000014118 by goroutine 8:
  main.main.func1()
      /tmp/pgt/c67/counter/main.go:17 +0x99

Previous write at 0x00c000014118 by goroutine 7:
  main.main.func1()
      /tmp/pgt/c67/counter/main.go:17 +0xab

Goroutine 8 (running) created at:
  main.main()
      /tmp/pgt/c67/counter/main.go:14 +0x99

Goroutine 7 (running) created at:
  main.main()
      /tmp/pgt/c67/counter/main.go:14 +0x99
==================
```

How to read a report:

| Part | Meaning |
|------|---------|
| `WARNING: DATA RACE` | a conflicting pair was found |
| `Read at 0x00c000014118 by goroutine 8` | the *current* access: a **read** of address `0x…118`, at `main.go:17` |
| `Previous write at 0x00c000014118 by goroutine 7` | the earlier, **conflicting write** to the **same address** |
| `Goroutine 8 (running) created at:` | where each goroutine was started: usually the fastest way to identify *which* goroutines these are |
| exit status **66** | a program that had races exits with status 66 (`go test` reports `FAIL`) |

Both accesses are at line 17, `counter++`: two goroutines, same line, same address. The fix location is obvious.

### What the race detector is (and isn't)

- **Precise, not speculative:** if it reports a race, it *is* a real race (no false positives in practice).
- **Dynamic:** it detects races that **actually happen in the run**. Code paths your tests never execute, or interleavings that don't occur, go unreported. *Tests + `-race` + realistic concurrency* is the recipe; a passing `-race` run proves the absence of races **only for the executions observed**.
- **Costly:** typically 5-10× slower and 5-10× more memory. (The racy counter took **4 ms** normally; the `-race` build took about **1 s**, part of which was printing reports.) Use it in tests and CI and staging, not usually in production.
- **Needs concurrency in the test:** a race between two goroutines can only be found if the test runs both. This is why the concurrency scenarios in our repository contract suite (Chapters 53 and 56) run 30-40 goroutines.
- **Limited by `GOMAXPROCS` and timing** but *not* by luck in the sense that matters: it observes memory-access ordering, not just wrong results, so it can flag a race in a run where the output happened to be correct.

Make it part of your workflow: `go test -race ./...` in CI, always. Every test run in this course's project since Chapter 50 used `-race`.

---

## 9. Other faces of the same bug

### Concurrent map writes: a fatal error

Go's maps are not safe for concurrent writes. The runtime *detects some* violations and kills the program:

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counts := map[int]int{}
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				counts[j] = i // many goroutines write the same map
			}
		}()
	}
	wg.Wait()
	fmt.Println("done, entries:", len(counts))
}
```

```
fatal error: concurrent map writes

goroutine 14 [running]:
internal/runtime/maps.fatal({0x4bfa6b?, 0x0?})
	/usr/local/go/src/runtime/panic.go:1046 +0x18
main.main.func1()
	/tmp/pgt/c67/mapwrite/main.go:17 +0x6c
```

Note **`fatal error`**, not `panic`: it *cannot be recovered*. The whole process dies (in a web server: every in-flight request). Go chose to crash loudly rather than let a map's internal structure be silently corrupted. The check is best-effort: it doesn't catch every unsafe access (a concurrent read and write, for instance, is a race that may or may not trigger it), so the race detector remains essential. This is *the* classic web-server bug: a global `map` used as a cache, with handlers (which run concurrently) writing to it.

### Appending to a shared slice: lost elements

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var results []int
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			results = append(results, i) // read len/cap/pointer, write a new slice header: not atomic
		}()
	}
	wg.Wait()

	fmt.Println("expected 1000 results, got", len(results))
}
```

```
expected 1000 results, got 991
expected 1000 results, got 959
expected 1000 results, got 964
```

A slice is a three-word header (pointer, length, capacity). `append` reads the header, writes the element, and stores a *new* header: two goroutines can read the same old length and both write to the same slot, so one element is lost (or, when the slice grows, one goroutine's entire growth is discarded). No crash, no error: **missing data**. (In Chapter 65 we avoided this with the own-slot pattern, `results[i] = ...`, where each goroutine writes a *different* element.)

### The shared `err`: a lost failure

This one is shaped exactly like code you may write soon (and like Chapter 70's target: the paginated list running `Count` and `List` at once). Two goroutines, and *one* `err` variable:

```go
package main

import (
	"errors"
	"fmt"
	"sync"
	"time"
)

// countProducts and listProducts stand in for the two repository calls of the paginated list.
func countProducts() (int, error) { return 500, errors.New("count failed: database timeout") }
func listProducts() ([]string, error) {
	time.Sleep(10 * time.Millisecond) // a slightly slower query
	return []string{"a", "b"}, nil
}

func main() {
	var (
		total int
		items []string
		err   error // ❌ ONE variable shared by both goroutines
		wg    sync.WaitGroup
	)

	wg.Add(2)
	go func() {
		defer wg.Done()
		total, err = countProducts() // writes err
	}()
	go func() {
		defer wg.Done()
		items, err = listProducts() // ALSO writes err (with nil): may erase the failure above
	}()
	wg.Wait()

	fmt.Println("total:", total, "items:", items)
	fmt.Println("err:", err)
}
```

```
total: 500 items: [a b]
err: <nil>
total: 500 items: [a b]
err: <nil>
total: 500 items: [a b]
err: <nil>
```

The count **failed**, and the program reports **no error**: `listProducts` finished later and overwrote `err` with `nil`. The caller would happily use `total = 500` (a value returned *alongside* an error, i.e., garbage) and serve wrong pagination metadata. This is the most dangerous kind of race: **a silently swallowed failure**. `go run -race` reports it:

```
WARNING: DATA RACE
Write at 0x00c000124170 by goroutine 8:
  main.main.func2()   main.go:32
Previous write at 0x00c000124170 by goroutine 7:
  main.main.func1()   main.go:28
...
Found 1 data race(s)
exit status 66
```

The fix is *not* a mutex: it is **not sharing**: give each goroutine its own error variable (`countErr`, `listErr`) and combine them after `Wait`. We do exactly that in Chapter 70.

---

## 10. It works on my machine: visibility

A subtler class: a race where the program "works". Here `main` spins until another goroutine sets a flag:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	done := false
	go func() {
		time.Sleep(10 * time.Millisecond)
		done = true // no synchronization with the reader below
	}()

	for !done { // the compiler may read `done` once and never look again
	}
	fmt.Println("finished")
}
```

On our machine (Go 1.25): it printed `finished` every time we ran it. **And it is still broken.** `go run -race` reports:

```
WARNING: DATA RACE
Write at 0x00c00001411f by goroutine 7:
  main.main.func1()  main.go:12
Previous read at 0x00c00001411f by main goroutine:
  main.main()  main.go:15
```

Why is it broken when it works? Because Go's memory model gives **no guarantee** that a write in one goroutine ever becomes *visible* to a read in another without synchronization. The compiler is allowed to assume no other goroutine modifies `done` and **hoist the read out of the loop**, turning it into `for { }` (an infinite loop); modern CPUs also have per-core caches and store buffers that delay visibility. Whether the loop terminates depends on the compiler version, optimization decisions, inlining, the CPU, and the day of the week. A compiler upgrade can turn "works for years" into "hangs in production".

> **There are no benign data races in Go.** "It's just a flag" and "it's only a statistics counter" are the famous last words. Fix every race the detector reports.

---

## 11. The memory model in one page

Go's memory model defines when a read in one goroutine is *guaranteed* to observe a write from another. The core idea is the **happens-before** relation: if event A *happens before* event B, then B is guaranteed to see A's effects. Within one goroutine, statements happen in program order. Between goroutines, **only synchronization creates happens-before edges**:

| Synchronization | Guarantee |
|-----------------|-----------|
| `go f()` | the `go` statement happens before `f` begins |
| **Channel send → receive** | a send happens before the corresponding receive completes |
| **Channel close → receive** | close happens before a receive that returns because it's closed |
| Unbuffered channel **receive → send completes** | the receive happens before the send completes |
| `sync.Mutex`: `Unlock` → next `Lock` | everything before `Unlock` is visible after the next `Lock` |
| `sync.WaitGroup`: `Done` → `Wait` returns | writes before `Done` are visible after `Wait` (Chapter 65) |
| `sync.Once`: `f` returns → any `Do` returns | one-time initialization is visible to everyone |
| `sync/atomic` operations | atomics are sequentially consistent and synchronize |

If two accesses to the same variable (one a write) are **not** ordered by a chain of these edges, they're a data race and **all bets are off**: torn values (for words larger than the machine word, e.g. an interface value or slice header), stale values, or values that never arrive.

This is why everything we did in Chapter 65 was correct: each goroutine wrote its own slot, and `wg.Wait()` supplied the happens-before edge for the reader.

---

## 12. Why races are dangerous

| Property | Consequence |
|----------|-------------|
| **Silent** | no crash, no error message: just wrong data |
| **Intermittent** | depends on timing: passes tests, fails under production load (or vice-versa) |
| **Not reproducible** | rerunning the program changes the interleaving; the bug hides |
| **Heisenbugs** | adding a `Println` or a debugger changes the timing and hides it |
| **Far from the cause** | corrupted data is used minutes later, elsewhere |
| **Security impact** | check-then-act races on permissions, balances, and uniqueness are exploitable ("time-of-check to time-of-use" vulnerabilities: double-spend, coupon reuse) |
| **Undefined semantics** | the language makes no promises for racy programs |

Compare with an ordinary bug: a wrong formula gives the *same* wrong answer every time and can be debugged. A race gives a *different* answer every time. Which is why prevention (design) and detection (`-race`) matter more than debugging.

---

## 13. Strategies for avoiding races

In order of preference:

1. **Don't share.** Give each goroutine its own data: its own slot (`results[i]`), its own error variable, its own copy (pass by value), its own buffer. With no shared memory there is nothing to synchronize. *(Chapter 65's patterns, Chapter 70's fix.)*
2. **Share only immutable data.** Data that is never modified after being published can be read by any number of goroutines freely (build it, then start the goroutines).
3. **Confine to one goroutine.** One goroutine owns the data; others *ask it* to act via channels ("don't communicate by sharing memory; share memory by communicating": Chapters 69-70).
4. **Synchronize access.** A mutex around every access (`sync.Mutex`, `RWMutex`), or atomics for a single word (`sync/atomic`): Chapter 68.
5. **Make the whole operation atomic when it's a compound one.** For check-then-act, the check and the act must be inside *one* critical section (or one database statement).

Then verify with `go test -race` on tests that **actually run the concurrent paths**.

Quick checklist when you write `go`:

- What variables does this closure capture? Are any of them written by anyone else (including the launching goroutine after the `go` statement)?
- Am I passing a pointer, slice, map, or channel? Who else has it?
- Is there a package-level variable involved?
- Is there a compound "read then write" that must be atomic?

---

## 14. Races in our own project

`net/http` runs each request in its own goroutine (Chapter 64), so **every piece of state that outlives a request is shared**. Audit what we built:

| State | Shared? | Protection |
|-------|---------|------------|
| `database.ProductStore` / `UserStore` (in-memory) | yes | `sync.RWMutex` in every method (Chapters 51-56); the contract suite's concurrency scenarios (30-40 goroutines) verify it under `-race` |
| PostgreSQL pool (`*sqlx.DB`) | yes | thread-safe by design (documented) |
| Config, `slog.Logger`, JWT secret | yes | **immutable after start-up** (strategy 2) |
| `user.Service` fields (`dummyHash` etc.) | yes | set once in the constructor, never modified (strategy 2) |
| Request-scoped values (`context`, request body) | no | one goroutine per request (strategy 1/3) |
| Any package-level `var` map/slice/counter | would be | we deliberately have none (Chapter 50 warned against globals) |

Two places where a *future* change could introduce a race: (1) the in-memory stores if someone adds a method and forgets the lock; (2) the concurrent `Count` + `List` we will add in Chapter 70, which is exactly the shared-`err` bug of section 9 waiting to happen. In Chapter 68 we prove the first with an experiment (delete a lock and watch the contract suite fail under `-race`).

---

## 15. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | "It's just a counter/flag, a race doesn't matter" | Lost updates; compiler-dependent hangs | No benign races: synchronize |
| 2 | Global `map` cache written by handlers | `fatal error: concurrent map writes` | Mutex, `sync.Map`, or confinement |
| 3 | Appending to a shared slice from goroutines | Lost elements | Own slots, mutex, or channel |
| 4 | One `err` variable for several goroutines | Errors silently lost | One error per goroutine, or `errgroup` |
| 5 | Check-then-act without atomicity | Overdrafts, double-use, oversell | One critical section / database constraint |
| 6 | Trusting "it passed 100 times" | Race hides until load | `-race` + design review |
| 7 | Running tests without concurrency | Races never triggered | Concurrent test scenarios |
| 8 | Only running `-race` locally | Regressions reach `main` | `-race` in CI |
| 9 | Using `time.Sleep` to "order" goroutines | Timing-dependent, still a race | Synchronization primitives |
| 10 | Sharing loop variables in closures (pre-1.22) | Wrong values, races | Go ≥ 1.22 / copy |
| 11 | Reading shared data after starting goroutines that write it | Race with the launcher | `Wait` first, or synchronize |
| 12 | Assuming reads are always safe | Concurrent read + write is a race | Protect reads too (RWMutex) |
| 13 | Ignoring a `-race` report because the output looks right | Latent corruption | Fix all reports |
| 14 | Copying a struct that contains a mutex or WaitGroup | Independent locks; `go vet` warning | Pass pointers |

---

## 16. Interview questions

**Q1. Define a data race.**
Two goroutines access the same memory location concurrently, at least one access is a write, and no synchronization orders them.

**Q2. Data race vs. race condition?**
A data race is unsynchronized conflicting memory access; a race condition is any timing-dependent incorrect behavior (which can occur even with atomic accesses, e.g. check-then-act).

**Q3. Why is `counter++` not atomic?**
It is load, add, store; another goroutine can interleave between steps so that updates overwrite each other.

**Q4. What does the race detector do, and what can't it do?**
It instruments memory accesses and reports conflicting accesses without happens-before; it only finds races that occur in the executed run, and costs ~5-10× time/memory.

**Q5. What is the happens-before relation?**
The partial order guaranteed by program order and synchronization events (channels, mutexes, `WaitGroup`, `Once`, atomics); only ordered accesses are guaranteed to observe each other's effects.

**Q6. Why can't Go maps be written concurrently?**
Their internal structure (buckets, growth) would be corrupted; the runtime detects some cases and aborts with `fatal error: concurrent map writes`.

**Q7. How do you fix a shared-variable race without locks?**
Stop sharing: per-goroutine variables/slots merged after `Wait`, immutable data, or ownership by one goroutine communicating through channels.

**Q8. Is a "benign" data race ever acceptable in Go?**
No: the memory model gives no guarantees for racy programs; compilers and CPUs may reorder, cache, or tear accesses.

**Q9. How does `wg.Wait()` make reading results safe?**
`Done` happens-before `Wait` returns, so writes made before `Done` are visible afterwards.

---

## 17. Exercises

### Exercise 1: Reproduce and detect
Run the counter and the account programs several times each, then with `-race`. How do results vary? Which line does each report point to?

<details><summary>Solution</summary>

The counter total differs every run (and is far below 1,000,000); the account always overdraws in the sleep version. `-race` points at `counter++` and at `a.balance` read/write lines respectively.
</details>

### Exercise 2: Fix by not sharing
Rewrite the counter so each goroutine counts locally and the results are combined after `Wait`, with no mutex. What's the result and why is it correct?

<details><summary>Solution</summary>

`partial := make([]int, 1000)`; goroutine `i` does `n := 0; for j… { n++ }; partial[i] = n`; after `wg.Wait()` sum `partial`. Always exactly 1,000,000: each variable has one writer, and `Wait` orders the reads.
</details>

### Exercise 3: The shared `err`
Write a version of the `Count`/`List` example with separate `countErr` and `listErr`, combined with `errors.Join`. Confirm that with `-race` there is no report and that the error is never lost.

<details><summary>Solution</summary>

```go
var totalErr, listErr error
go func() { defer wg.Done(); total, totalErr = countProducts() }()
go func() { defer wg.Done(); items, listErr = listProducts() }()
wg.Wait()
err := errors.Join(totalErr, listErr)
```
Each goroutine writes a different variable; `Wait` orders the reads.
</details>

### Exercise 4: Find the races
List every race in this handler (assume many requests run at once):

```go
var cache = map[int]Product{}
var hits int

func Get(w http.ResponseWriter, r *http.Request) {
	id, _ := strconv.Atoi(r.PathValue("id"))
	if p, ok := cache[id]; ok {
		hits++
		json.NewEncoder(w).Encode(p)
		return
	}
	p := load(id)
	cache[id] = p
	json.NewEncoder(w).Encode(p)
}
```

<details><summary>Solution</summary>

`cache` read and write from concurrent handlers (map race, possibly fatal); `hits++` (lost updates); plus the check-then-act between reading `cache[id]` and writing it (two requests may both load the same product; harmless here but wasteful). Fixes: a mutex (or `sync.Map`) for the cache and an `atomic.Int64` for `hits`.
</details>

### Exercise 5: Make the detector work for you
Take any test in the project and run it with `-race -count=50`. Then temporarily remove one `mu.Lock()` from `database.UserStore.Create` and rerun the contract suite. What happens?

<details><summary>Solution</summary>

The concurrent-duplicate scenario (30 goroutines) fails, and `-race` prints a data race pointing at the unprotected slice access. (Chapter 68 shows the exact output.)
</details>

### Exercise 6 (challenge): A racy rate limiter
This token bucket is used from many goroutines; explain every problem and sketch a correct version.

```go
type Limiter struct{ tokens int; last time.Time }
func (l *Limiter) Allow() bool {
	if time.Since(l.last) > time.Second { l.tokens = 10; l.last = time.Now() }
	if l.tokens > 0 { l.tokens--; return true }
	return false
}
```

<details><summary>Solution</summary>

Data races on `tokens` and `last` (unsynchronized reads/writes) and check-then-act atomicity: two goroutines can both see `tokens == 1` and both proceed, or both refill. Fix: a `sync.Mutex` held for the whole body of `Allow` (compound operation), or a channel of tokens filled by a ticker goroutine.
</details>

---

## 18. Quiz

1. What four conditions make a data race?
2. Why did the counter lose increments?
3. What exit status does a program with detected races return?
4. Why can `for !done {}` hang even though another goroutine sets `done`?
5. What happens on concurrent map writes?
6. Which strategy is preferred over locking?
7. Does a correct `WaitGroup` prevent data races on shared variables?
8. Name two sync events that create happens-before edges.

<details><summary>Answers</summary>

1. Shared memory, concurrent access, at least one write, no synchronization.
2. `counter++` is load-add-store; interleaved goroutines overwrite each other's stores.
3. 66 (and `go test` marks the test failed).
4. Without synchronization the compiler/CPU may never re-read `done`.
5. `fatal error: concurrent map writes` (unrecoverable) when detected.
6. Not sharing: own slots/variables, immutable data, confinement.
7. No: `Wait` only orders reads *after* completion; concurrent accesses *during* the run still race.
8. Any two of: channel send/receive, mutex unlock/lock, `Done`/`Wait`, `Once`, atomics, `go` statement.
</details>

---

## 19. Summary

- A **data race** = shared memory + concurrent access + at least one write + no synchronization. A **race condition** = timing-dependent wrongness (check-then-act bugs included).
- Measured damage: a counter that kept **~42%** of a million increments (and a different answer each run), an account that paid **5 × $60 from $100** (balance −200), a slice that lost **up to 4%** of its elements, a map that killed the process with `fatal error: concurrent map writes`, and a shared `err` that **erased a database failure**.
- `counter++` is **three steps** (load, add, store); goroutines interleave between them: **lost updates**.
- The **race detector** (`-race`) reports both accesses with goroutine creation sites; it's precise but dynamic (only finds executed races) and costs 5-10×: run it in CI on concurrent tests.
- "It works" is not "it's correct": without happens-before edges, the compiler and CPU are free to hide writes (**visibility**). **There are no benign races.**
- Happens-before edges come from channels, mutexes, `WaitGroup`, `Once`, atomics and the `go` statement.
- Avoid races by design: **don't share** → immutable → confine → synchronize → make compound operations atomic.
- In our project, shared state is protected by `RWMutex`, immutable configuration, or per-request ownership; Chapter 70's concurrent list must give each goroutine its own error variable.

### ➡️ What's next?

[Chapter 68](68-sync-mutex.md) fixes these bugs with **`sync.Mutex`**: mutual exclusion, `defer Unlock`, `RWMutex`, atomics as the lighter alternative, and a look at what mutexes cost (with benchmarks). We'll also prove the in-memory stores are correct by deleting a lock and watching the tests fail.
