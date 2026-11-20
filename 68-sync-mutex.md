# Chapter 68: `sync.Mutex` — Protecting Shared Data

> **Goal of this chapter:** Fix the races of Chapter 67. You'll learn **mutual exclusion** (only one goroutine at a time inside a *critical section*), the `sync.Mutex` API, and the rules that make it correct: keep the check-and-act **inside one critical section**, always `defer Unlock`, never copy a mutex, never lock twice, always lock in the same order. Each failure mode is shown with **real output**: `fatal error: sync: unlock of unlocked mutex`, the self-deadlock, the two-lock deadlock, the lock leaked by a panic. You'll measure what a mutex costs (129 ms vs 28 ms for atomics on a million increments), when `RWMutex` helps (about 1.7× at 12 CPUs, less than you may expect), why blocked goroutines burn **0 CPU**, and, in our own project, prove that removing a single `Lock` makes `go test` **pass** and `go test -race` **fail**.

**Difficulty:** 🔴 Advanced  **Estimated time:** 6 hours  **Prerequisite:** [Chapter 67](67-race-conditions.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The idea: locking](#2-the-idea-locking)
3. [The API](#3-the-api)
4. [Fixing the counter](#4-fixing-the-counter)
5. [Fixing the bank account](#5-fixing-the-bank-account)
6. [What "blocked" really means](#6-what-blocked-really-means)
7. [`defer Unlock`, and what happens without it](#7-defer-unlock-and-what-happens-without-it)
8. [More ways to get it wrong](#8-more-ways-to-get-it-wrong)
9. [`RWMutex`: many readers, one writer](#9-rwmutex-many-readers-one-writer)
10. [Atomics: a lighter tool for one variable](#10-atomics-a-lighter-tool-for-one-variable)
11. [Designing with mutexes](#11-designing-with-mutexes)
12. [Other tools: `Once`, `sync.Map`, channels](#12-other-tools-once-syncmap-channels)
13. [Our project: one missing `Lock`](#13-our-project-one-missing-lock)
14. [Common mistakes](#14-common-mistakes)
15. [Interview questions](#15-interview-questions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- **Mutual exclusion** and **critical sections**
- `Lock`, `Unlock`, `TryLock`, and the rules of `sync.Mutex` (not reentrant, not copyable, unlock what you locked)
- Making **compound operations** (check-then-act) atomic
- Why waiting goroutines cost no CPU
- `defer` for unlocking, and `defer` ordering
- The classic failure modes: **deadlock** (self, and lock-ordering), leaked locks, copies
- `sync.RWMutex`, `sync/atomic`, `sync.Once`, and when each is right
- Guidelines for designing lock-protected types

---

## 2. The idea: locking

The race in Chapter 67 exists because two goroutines were *inside* the "read-modify-write" region at once. The cure is to allow **only one at a time**. The picture is a room with one key:

```
   goroutine A ─┐
   goroutine B ─┼──►  [ lock ]  ──►  critical section  ──►  [ unlock ]
   goroutine C ─┘        ▲           (touches the shared data)
                         │
              one at a time: the others WAIT outside
```

- The region of code that must not be run by two goroutines at once is the **critical section**.
- **`Lock()`** enters it (waiting if someone else is inside); **`Unlock()`** leaves it and lets one waiter in.
- The lock *doesn't know* what it protects. **You** decide which data a mutex guards and you must take it on **every** access to that data (reads included). A single unprotected access reintroduces the race.

The word *mutex* is short for **mut**ual **ex**clusion. Also, unlike a semaphore, a mutex has an **owner concept by convention**: whoever locks it should unlock it.

---

## 3. The API

```go
var mu sync.Mutex     // the zero value is an unlocked mutex: ready to use

mu.Lock()             // blocks until the mutex is acquired
mu.Unlock()           // releases it; panics (fatally) if it isn't locked
ok := mu.TryLock()    // acquires it if it is free right now and returns true; never blocks (Go 1.18+)
```

`TryLock` exists but its documentation warns that correct uses are rare: needing it usually signals a design problem. We won't use it.

The rules (each demonstrated below):

| Rule | Consequence of breaking it |
|------|---------------------------|
| Take the lock for **every** access to the guarded data | data race |
| **Unlock exactly what you locked**, exactly once per `Lock` | fatal error / permanent block |
| Mutexes are **not reentrant**: don't lock a mutex you already hold | self-deadlock |
| Never **copy** a mutex (or a struct containing one) after first use | independent locks; `go vet` warning |
| Lock multiple mutexes in a **consistent global order** | deadlock |
| Keep critical sections **short**; no slow I/O inside | throughput collapse |

---

## 4. Fixing the counter

The Chapter 67 counter, with one added mutex:

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var (
		mu      sync.Mutex // guards counter
		counter int
		wg      sync.WaitGroup
	)

	start := time.Now()
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				mu.Lock()
				counter++ // only one goroutine at a time can be between Lock and Unlock
				mu.Unlock()
			}
		}()
	}
	wg.Wait()

	fmt.Println("expected:", 1000*1000)
	fmt.Println("actual:  ", counter)
	fmt.Println("took:    ", time.Since(start).Round(time.Millisecond))
}
```

```
expected: 1000000
actual:   1000000
took:     129ms
```

Exactly one million, every run, and `-race` reports nothing. The mutex also creates the **happens-before** edges from Chapter 67: everything a goroutine did before `Unlock` is visible to the next goroutine that `Lock`s. The price: 129 ms, where the *wrong* version took 4 ms. Correctness has a cost; the goal is to pay it only where needed.

Convention: **declare the mutex next to the data it guards** and say so in a comment. In a struct, put it above the fields it protects:

```go
type Stats struct {
	mu    sync.Mutex // guards hits and misses
	hits  int
	misses int
}
```

---

## 5. Fixing the bank account

In Chapter 67, five withdrawals of 60 from a balance of 100 all succeeded (final balance −200). The bug was **check-then-act** split across two steps. The fix is to put *both* inside one critical section:

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Account struct {
	mu      sync.Mutex
	balance int
}

// Withdraw makes the check and the act ONE critical section.
func (a *Account) Withdraw(amount int) bool {
	a.mu.Lock()
	defer a.mu.Unlock()

	if a.balance < amount {
		return false
	}
	time.Sleep(time.Millisecond) // slow work inside the lock: bad for throughput, but now harmless for correctness
	a.balance -= amount
	return true
}

func (a *Account) Balance() int {
	a.mu.Lock()
	defer a.mu.Unlock()
	return a.balance
}

func main() {
	acc := &Account{balance: 100}
	var wg sync.WaitGroup
	var succeeded int
	var mu sync.Mutex

	for i := 0; i < 5; i++ {
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

	fmt.Println("withdrawals of 60 that succeeded:", succeeded)
	fmt.Println("final balance:", acc.Balance())
}
```

```
withdrawals of 60 that succeeded: 1
final balance: 40
```

Exactly one withdrawal wins; the other four see a balance of 40 and are refused. Notice three design points:

1. **The lock is inside the type**, and callers can't forget it: `Withdraw` and `Balance` lock internally. Callers see a safe API. (Unexported field `mu`; never expose the mutex to callers.)
2. **`Balance()` locks too.** *Reads* need the lock as well: reading while another goroutine writes is a race.
3. The `time.Sleep` inside the lock is deliberate and deliberately *bad*: while it holds the lock, all other goroutines wait, so throughput collapses to one operation per millisecond. It is here to show that correctness doesn't depend on timing any more, and to remind you: **never hold a lock across slow work** (database calls, HTTP requests, disk I/O). Do the slow part outside; take the lock only around the shared-data access.

---

## 6. What "blocked" really means

What do 1,000 goroutines waiting for a lock cost? Test it: `main` holds the mutex while 1,000 goroutines try to take it; we measure the **CPU time** the whole process consumes (via `getrusage`) during two seconds of waiting:

```go
package main

import (
	"fmt"
	"sync"
	"syscall"
	"time"
)

// cpuTime returns the CPU time this process has used so far (user + system).
func cpuTime() time.Duration {
	var ru syscall.Rusage
	syscall.Getrusage(syscall.RUSAGE_SELF, &ru)
	return time.Duration(ru.Utime.Nano() + ru.Stime.Nano())
}

func main() {
	var mu sync.Mutex
	var wg sync.WaitGroup

	mu.Lock() // main holds the lock...
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() { // ...so all 1000 goroutines block here
			defer wg.Done()
			mu.Lock()
			mu.Unlock()
		}()
	}

	start, cpuBefore := time.Now(), cpuTime()
	time.Sleep(2 * time.Second) // 1000 goroutines wait for 2 seconds
	fmt.Printf("waited %v of wall-clock time; CPU consumed by 1000 blocked goroutines: %v\n",
		time.Since(start).Round(100*time.Millisecond), (cpuTime() - cpuBefore).Round(time.Millisecond))

	mu.Unlock()
	wg.Wait()
}
```

```
waited 2s of wall-clock time; CPU consumed by 1000 blocked goroutines: 0s
```

**Zero CPU.** A goroutine blocked in `Lock` is *parked*: the runtime takes it off the run queue and puts it to sleep on a semaphore (the same mechanism as `WaitGroup.Wait`, Chapter 66); when the mutex is released, one sleeper is made runnable. Waiting costs memory (a stack and a `sudog`), not CPU. (Very briefly, before parking, a goroutine may *spin*, retrying for a few microseconds in case the holder is about to release: a small optimization that pays off when critical sections are short. The mutex also has a *starvation mode*: if a waiter has been blocked longer than 1 ms, the lock is handed directly to it so that no goroutine waits forever.)

This is why mutexes are cheap when *uncontended* (a single atomic instruction, ~15 ns) and merely *serializing* when contended: the harm isn't CPU waste but **lost parallelism**.

---

## 7. `defer Unlock`, and what happens without it

Every code path out of a critical section must unlock: early `return`s, `panic`s, and errors. Writing the `Unlock` at the bottom is fragile, so the idiom is:

```go
mu.Lock()
defer mu.Unlock()   // runs when the function returns, however it returns
```

What breaks without it: a panic inside the critical section, *even one that is recovered somewhere else*, leaks the lock:

```go
package main

import (
	"fmt"
	"sync"
)

var mu sync.Mutex

// withoutDefer forgets to unlock when the code between Lock and Unlock panics.
func withoutDefer() {
	defer func() { recover() }() // the caller survives the panic...
	mu.Lock()
	var m map[string]int
	m["boom"] = 1 // panics: assignment to entry in nil map
	mu.Unlock()   // ...but this line never runs
}

func main() {
	withoutDefer()
	fmt.Println("recovered; now trying to lock again...")
	mu.Lock() // blocks forever: the mutex is still held
	fmt.Println("never printed")
}
```

```
recovered; now trying to lock again...
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.Mutex.Lock]:
internal/sync.runtime_SemacquireMutex(0xc000062028?, 0x30?, 0x27?)
...
```

The program "recovered" from the panic and carried on, but the mutex stayed locked forever. In a web server (where `net/http` and our `Recover` middleware, Chapter 46, keep the server alive after a handler panic) this is a **permanently locked** resource: every later request that needs it hangs. `defer mu.Unlock()` makes this impossible.

### Order of multiple defers

Deferred calls run **last-in, first-out**:

```go
mu.Lock()
defer fmt.Println("3rd deferred call registered FIRST  -> runs LAST")
defer fmt.Println("2nd deferred call")
defer mu.Unlock() // registered last -> runs FIRST
fmt.Println("inside the critical section")
```
```
inside the critical section
2nd deferred call
3rd deferred call registered FIRST  -> runs LAST
TryLock after the function returned: true
```

`Unlock` was registered last, so it ran first (the lock was free before the prints), and `TryLock` afterwards returned `true`. Consequence: if you `defer` other cleanup that is slow, *register `Unlock` after it* so the lock is released first, or restructure into a small helper function so the lock scope is obvious:

```go
func (s *Store) Get(k string) (int, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	v, ok := s.items[k]
	return v, ok
}
```

A `defer` costs a few nanoseconds (much less since Go 1.14). Don't skip it for speed unless a profile *proves* it matters.

---

## 8. More ways to get it wrong

### Unlocking an unlocked mutex

```go
var mu sync.Mutex
mu.Unlock() // never locked
```
```
fatal error: sync: unlock of unlocked mutex

goroutine 1 [running]:
internal/sync.fatal({0x487e99?, 0x47b920?})
```

A **fatal error** (not a recoverable panic): the mutex's state is corrupt and nothing sensible can continue. Typical cause: `Unlock` called twice (once in a `defer`, once manually), or on an error path where `Lock` wasn't reached.

### Locking twice: the self-deadlock

Go's mutexes are **not reentrant**: a goroutine that already holds the lock and calls `Lock` again waits for *itself*:

```go
type Store struct {
	mu    sync.Mutex
	items map[string]int
}

func (s *Store) Get(k string) int {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.items[k]
}

// Total locks, then calls Get, which tries to lock AGAIN.
func (s *Store) Total(keys ...string) int {
	s.mu.Lock()
	defer s.mu.Unlock()
	sum := 0
	for _, k := range keys {
		sum += s.Get(k) // ❌ Go mutexes are not reentrant: this waits for itself
	}
	return sum
}
```
```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.Mutex.Lock]:
...
main.(*Store).Get(...)   main.go:14
main.(*Store).Total(...) main.go:24
```

The trace tells the whole story: `Total` → `Get` → `Lock`, blocked. **Why Go doesn't provide reentrant locks:** they encourage muddled designs where nobody knows what state the data is in when a method is entered. The standard fix is a convention: **exported methods take the lock; unexported helpers assume it is held** (and say so in the name):

```go
func (s *Store) Total(keys ...string) int {
	s.mu.Lock()
	defer s.mu.Unlock()
	sum := 0
	for _, k := range keys {
		sum += s.getLocked(k)
	}
	return sum
}

// getLocked returns items[k]. The caller must hold s.mu.
func (s *Store) getLocked(k string) int { return s.items[k] }
```

(This is the same convention we used in Chapter 56's in-memory store: `indexOf` "the caller must hold the lock".)

### Two locks, opposite order: the classic deadlock

```go
var (
	accountA sync.Mutex
	accountB sync.Mutex
)

go func() { // transfer A -> B: locks A then B
	accountA.Lock()
	time.Sleep(10 * time.Millisecond)
	accountB.Lock()
	...
}()

go func() { // transfer B -> A: locks B then A  (the opposite order!)
	accountB.Lock()
	time.Sleep(10 * time.Millisecond)
	accountA.Lock()
	...
}()
```
```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]: ...
goroutine 6 [sync.Mutex.Lock]:
...
main.main.func1()   main.go:22
goroutine 7 [sync.Mutex.Lock]:
...
main.main.func2()   main.go:34
```

Goroutine 6 holds A and waits for B; goroutine 7 holds B and waits for A. Neither can proceed: a **circular wait**. The four conditions for deadlock (mutual exclusion, hold-and-wait, no preemption, circular wait) are all present; removing *any* one prevents it, and the practical one is **circular wait**:

> **Always acquire multiple locks in the same global order** (e.g., by account ID: always the lower ID first), in *every* code path.

Or, better: **avoid needing two locks**: redesign so that one lock covers the whole operation, or use one goroutine that owns the accounts and receives transfers on a channel (Chapter 69).

### Copying a mutex

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

// Inc has a VALUE receiver: every call works on a COPY of the Counter, including a copy of the mutex.
func (c Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```
```
./main.go:14:9: Inc passes lock by value: x.Counter contains sync.Mutex
n = 0 (the increment was applied to a copy)
```

Two symptoms: `go vet` (run by `go test`) complains, and the program prints `n = 0`: the increment went to a *copy*, and the copied mutex protects nothing shared. **Any type containing a mutex must use pointer receivers and be passed by pointer**, and constructors should return `*T`. (Chapter 22's rule: if any method needs a pointer receiver, use pointer receivers for all.)

---

## 9. `RWMutex`: many readers, one writer

Reads don't conflict with each other; only writes do. `sync.RWMutex` allows **any number of readers *or* one writer**:

```go
var mu sync.RWMutex

mu.RLock();  defer mu.RUnlock()   // shared: many goroutines may hold it at once
mu.Lock();   defer mu.Unlock()    // exclusive: waits for all readers to leave, blocks new ones
```

That's the lock our in-memory stores use (`RLock` in `List`/`Get`/`FindBy…`, `Lock` in `Create`/`Update`/`Delete`). Is it worth the extra complexity? A benchmark: a 1,024-entry map cache hit by many goroutines, once with **1% writes** (read-heavy) and once with **50% writes**, using `-cpu 1,4,12`:

```
BenchmarkReadHeavy_Mutex          	65823157	        17.35 ns/op
BenchmarkReadHeavy_Mutex-4        	24159957	        49.22 ns/op
BenchmarkReadHeavy_Mutex-12       	13608964	        88.86 ns/op
BenchmarkReadHeavy_RWMutex        	66407007	        15.44 ns/op
BenchmarkReadHeavy_RWMutex-4      	37780657	        31.08 ns/op
BenchmarkReadHeavy_RWMutex-12     	25969839	        52.63 ns/op
BenchmarkWriteHeavy_Mutex         	63602419	        18.42 ns/op
BenchmarkWriteHeavy_Mutex-4       	16796211	        71.29 ns/op
BenchmarkWriteHeavy_Mutex-12      	 9574800	       136.4 ns/op
BenchmarkWriteHeavy_RWMutex       	45417210	        26.54 ns/op
BenchmarkWriteHeavy_RWMutex-4     	21203228	        61.29 ns/op
BenchmarkWriteHeavy_RWMutex-12    	19833219	        60.96 ns/op
```

Reading the results honestly:

- **Read-heavy, 12 CPUs:** RWMutex 52.6 ns vs Mutex 88.9 ns, a **1.7× improvement**. Real, but modest.
- **Why not 12×?** Even a *read lock* must update a shared reader counter atomically, and that single memory word is bounced between cores (Chapter 66's cache-line contention). Readers don't block each other, but they still **contend on the counter**. With critical sections this short (one map lookup), that overhead dominates.
- **Write-heavy, single CPU:** RWMutex is *slower* (26.5 vs 18.4 ns): it does more bookkeeping.
- RWMutex pays off when **read critical sections are long** (so that letting readers overlap saves real time) and writes are rare. For tiny sections, plain `Mutex` is simpler and nearly as fast.

Extra `RWMutex` rules: **never upgrade** (holding `RLock` and calling `Lock` deadlocks), **don't take `RLock` recursively** (a waiting writer blocks new readers, so a recursive read lock can deadlock), and never mutate under `RLock`.

**Guidance:** start with `sync.Mutex`. Switch to `RWMutex` when a benchmark on realistic load shows read contention. (Our stores use `RWMutex` because it documents intent, and it costs almost nothing; if you disagree in your own code, use a `Mutex`: simplicity is a virtue.)

---

## 10. Atomics: a lighter tool for one variable

For a *single word* (a counter, a flag), `sync/atomic` performs the read-modify-write as one indivisible CPU instruction: no lock, no waiting:

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	var counter atomic.Int64 // one indivisible read-modify-write instruction per Add
	var wg sync.WaitGroup

	start := time.Now()
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				counter.Add(1)
			}
		}()
	}
	wg.Wait()

	fmt.Println("expected:", 1000*1000)
	fmt.Println("actual:  ", counter.Load())
	fmt.Println("took:    ", time.Since(start).Round(time.Millisecond))
}
```

```
expected: 1000000
actual:   1000000
took:     28ms
```

Correct, and **4.6× faster** than the mutex version (28 ms vs 129 ms). The typed API (`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`, `atomic.Value`) came in Go 1.19: prefer it over the older function style (`atomic.AddInt64(&x, 1)`) because a plain `int64` field can be accessed non-atomically by mistake, while the typed wrapper can't.

Limits of atomics:

| Atomics are good for | Atomics are **not** enough for |
|----------------------|--------------------------------|
| counters, sequence numbers, flags, statistics | **compound invariants** across several variables (`a + b` must equal `c`) |
| publishing an immutable snapshot (`atomic.Pointer[Config]`) | check-then-act on *more than one* word |
| lock-free fast paths (as inside `WaitGroup`) | anything that reads then decides then writes several things |

The bank account needs a mutex (check balance *and* subtract as one unit). A hit counter should be atomic. A rule of thumb: **one variable → atomic; more than one, or a sequence of steps → mutex.**

---

## 11. Designing with mutexes

Guidelines that prevent most lock bugs:

1. **Encapsulate.** Put the mutex and the data it guards in one struct, keep both unexported, and expose methods that lock internally. Callers can't get it wrong.
2. **Document the guard.** `mu sync.Mutex // guards items and count` right above the fields.
3. **Small critical sections.** Copy what you need out of the lock, then do slow work outside:

```go
s.mu.Lock()
item := s.items[id]     // copy out
s.mu.Unlock()
render(item)            // slow work OUTSIDE the lock
```

4. **No I/O, no callbacks, no channel operations while holding a lock.** Calling unknown code under a lock is how deadlocks (and priority inversions) creep in.
5. **Return copies, not references.** If a method returns a slice or map from inside the structure, the caller now holds an *unprotected* alias to guarded data. (Our stores' `List` returns a *copy*: "callers must not be able to modify the store's internal slice", tested in Chapter 56.)
6. **Consistent lock ordering** if more than one lock is ever needed; better, avoid needing two.
7. **Prefer pointer receivers** and `go vet`.
8. **Test with `-race`** and concurrent tests.
9. **Consider not sharing** (Chapter 67, section 13): confinement and channels often produce simpler code than locking.

---

## 12. Other tools: `Once`, `sync.Map`, channels

- **`sync.Once`**: run an initialization exactly once, even if many goroutines race to trigger it (used in Chapter 51's singleton). The completion of `Do(f)` happens-before every `Do` return, so everyone sees the initialized data. Since Go 1.21 there are also `sync.OnceFunc`, `OnceValue`, and `OnceValues`.
- **`sync.Map`**: a concurrent map optimized for two patterns (write-once/read-many keys, and goroutines working on disjoint keys). For ordinary cases a plain `map` + `Mutex`/`RWMutex` is clearer, typed, and often faster.
- **`sync.Cond`**: wait for a condition on shared state; rarely needed (channels usually read better).
- **`sync.Pool`**: reuse temporary objects to reduce garbage-collector pressure (used inside the standard library, e.g. for `fmt` buffers).
- **Channels** (next chapter): hand data to *one owner* instead of sharing it: "don't communicate by sharing memory; share memory by communicating".

---

## 13. Our project: one missing `Lock`

`database.UserStore` (Chapter 60) guards its slice with a `sync.RWMutex`. What would happen without the lock in `Create`? Delete that one `s.mu.Lock()` (and its `defer`) from `Create`, and run the repository contract suite (which includes the "30 goroutines register the same email, exactly one must win" scenario) twice:

```bash
go test -count=1 -run UserStoreContract ./database/            # WITHOUT -race
go test -race -count=1 -run UserStoreContract ./database/      # WITH -race
```

```
ok  	ecommerce/database	0.003s              ← the buggy code PASSES
```
```
==================
WARNING: DATA RACE
Read at 0x00c000130cd8 by goroutine 35:
  ecommerce/database.(*UserStore).Create()
      user_store.go:25 +0x89
  ecommerce/user/storetest.Run.func8.1()
      storetest.go:131 +0x130

Previous write at 0x00c000130cd8 by goroutine 18:
  ecommerce/database.(*UserStore).Create()
      user_store.go:32 +0x567
...
--- FAIL: TestUserStoreContract (0.00s)
    --- FAIL: TestUserStoreContract/concurrent_duplicate_registrations:_exactly_one_wins (0.00s)
        testing.go:1712: race detected during execution of test
FAIL
```

This is the whole story of this part of the course in one experiment:

- **Without `-race`, the broken code passes.** The bug is real, but the specific schedule needed to produce a *wrong result* didn't occur in that run. If we relied on ordinary tests, we'd ship it.
- **With `-race`, it fails deterministically.** The detector saw a read at line 25 (`for _, u := range s.users`) and a write at line 32 (`s.users = append(...)`) from different goroutines, with no lock ordering them: a data race. It doesn't need the wrong *outcome*; the missing synchronization is enough.

Which is why Chapter 67's recommendation is non-negotiable: **`go test -race ./...` in CI, with tests that exercise concurrency.**

---

## 14. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Protecting writes but not reads | Still a data race | Lock (or `RLock`) for every access |
| 2 | Splitting check and act across two critical sections | Overdrafts, double-use | One critical section |
| 3 | `Unlock` without `defer` | Leaked lock on early return/panic | `defer mu.Unlock()` |
| 4 | Locking twice in one call chain | Self-deadlock | `xLocked` helpers; exported methods lock |
| 5 | Copying a struct containing a mutex | Independent lock; vet warning | Pointer receivers; return `*T` |
| 6 | Opposite lock orders | Deadlock | One global order, or one lock |
| 7 | Holding a lock during I/O/network | Throughput collapse, hidden deadlocks | Copy out, unlock, then do the slow work |
| 8 | Returning internal maps/slices | Callers mutate guarded data unlocked | Return copies |
| 9 | `RLock` then `Lock` (upgrade) | Deadlock | Release, then `Lock` and re-check |
| 10 | Assuming `RWMutex` is always faster | Slower for short sections/write-heavy | Benchmark |
| 11 | Using a mutex where an atomic suffices (or vice versa) | Needless cost / broken invariant | One word → atomic; several → mutex |
| 12 | Exporting the mutex (`Mu sync.Mutex`) | Callers lock inconsistently | Unexported, internal locking |
| 13 | Testing without `-race` | Missing lock passes tests | `-race` in CI |
| 14 | Unlocking a mutex you didn't lock | `fatal error: unlock of unlocked mutex` | Pair every `Unlock` with its `Lock` |

---

## 15. Interview questions

**Q1. What is a mutex and what problem does it solve?**
A lock allowing one goroutine at a time into a critical section, preventing data races on shared state.

**Q2. Why must reads be locked too?**
A read concurrent with a write is a data race; it can see torn or stale values.

**Q3. How do you fix a check-then-act race?**
Put the check and the act in the same critical section (or use one atomic operation / database constraint).

**Q4. Why `defer mu.Unlock()`?**
It guarantees release on every exit path, including panics, preventing leaked locks.

**Q5. Are Go mutexes reentrant?**
No: locking a mutex you hold deadlocks; structure code with "locked" helper functions.

**Q6. Describe a deadlock with two mutexes and how to avoid it.**
Goroutine 1 holds A and wants B while goroutine 2 holds B and wants A; always acquire locks in the same global order (or use a single lock).

**Q7. `Mutex` vs `RWMutex`?**
`RWMutex` allows many concurrent readers or one writer; it helps when reads dominate and read sections are long, but has more overhead and pitfalls (no upgrading).

**Q8. When to prefer atomics over a mutex?**
For a single independent variable (counter, flag, pointer swap); mutexes for multi-variable invariants and compound operations.

**Q9. What does a blocked `Lock` cost?**
No CPU: the goroutine is parked; it costs memory and lost parallelism.

---

## 16. Exercises

### Exercise 1: Fix and prove
Take the racy slice-append program from Chapter 67 and fix it three ways: a mutex, an own-slot design, and a channel (preview). Verify each with `-race` and check the count.

<details><summary>Solution</summary>

Mutex: `mu.Lock(); results = append(results, i); mu.Unlock()`. Own-slot: `results := make([]int, 1000)` and `results[i] = i`. Channel: goroutines send on a buffered channel, and the receiver appends in one goroutine. All give 1,000 elements and no race reports.
</details>

### Exercise 2: A safe cache
Write `type Cache struct` with `Get(key) (Product, bool)` and `Set(key, Product)` guarded by an `RWMutex`, plus a test with 100 goroutines doing mixed reads and writes under `-race`. Then remove one lock and confirm `-race` catches it.

<details><summary>Solution</summary>

`Get` uses `RLock`/`defer RUnlock`, `Set` uses `Lock`/`defer Unlock`. The test launches goroutines calling `Set(i%10, …)` and `Get(i%10)` with a `WaitGroup`. Removing `RLock` from `Get` yields a race report between `Get`'s map read and `Set`'s map write.
</details>

### Exercise 3: Order the locks
Write `Transfer(from, to *Account, amount int)` that locks both accounts safely for arbitrary concurrent transfers (including `from == to`). What global order do you use?

<details><summary>Solution</summary>

Give each account an immutable `ID`; lock the account with the smaller ID first, then the larger; if `from == to`, lock once. Test with many goroutines doing random transfers in both directions and verify the total balance is conserved and no deadlock occurs.
</details>

### Exercise 4: Copy trap
Write a struct with a mutex and a method with a value receiver; run `go vet`. Fix it. Then find a case where `vet` *cannot* see the copy (hint: copying through an `interface{}` or a slice of structs).

<details><summary>Solution</summary>

Vet catches direct copies, value receivers, and passing by value, but copies that happen through reflection, `encoding/json`, or generic code may escape it. The safest defense is a design where such structs are only ever handled by pointer (constructors return `*T`).
</details>

### Exercise 5: Measure your own critical section
Benchmark a mutex-protected counter with 0, 100 ns and 1 µs of simulated work inside the lock at 1, 4, and 12 CPUs. How does throughput scale? What does it say about holding locks during work?

<details><summary>Solution</summary>

With work inside the lock, throughput is bounded by `1 / (work + lock overhead)` **regardless of CPU count**: more CPUs don't help because the critical section serializes everything (Amdahl's law). Move the work outside the lock and throughput scales again.
</details>

### Exercise 6 (challenge): A bounded `RWMutex` reader
Explain why the following can deadlock, and fix it:

```go
func (c *Cache) Sum() int {
	c.mu.RLock()
	defer c.mu.RUnlock()
	total := 0
	for k := range c.m {
		total += c.Get(k) // Get also does RLock
	}
	return total
}
```

<details><summary>Solution</summary>

If a writer calls `Lock` between the outer `RLock` and the inner `RLock`, the writer waits for the outer reader, and the *new* reader (inner `Get`) waits behind the pending writer: the goroutine waits on itself through the writer. Fix: a `getLocked` helper that assumes the lock is held and use it inside `Sum`.
</details>

---

## 17. Quiz

1. What is a critical section?
2. Why lock even for reads?
3. What does `defer mu.Unlock()` protect against?
4. What happens if a goroutine locks a mutex it already holds?
5. How do you prevent the two-lock deadlock?
6. When is an atomic better than a mutex?
7. What did removing the lock from `Create` do to `go test` and to `go test -race`?
8. How much CPU do 1,000 goroutines blocked on a mutex use?

<details><summary>Answers</summary>

1. Code that must not be executed by two goroutines at once.
2. A read concurrent with a write is a data race.
3. Leaked locks on early returns and panics.
4. It deadlocks (Go's mutexes are not reentrant).
5. Acquire locks in one global order in every code path, or use a single lock.
6. A single independent variable (counter/flag), no compound invariant.
7. Plain `go test` still passed; `go test -race` failed with a data race report.
8. Essentially none (measured 0 s): they are parked, not spinning.
</details>

---

## 18. Summary

- A **mutex** admits one goroutine at a time into a **critical section**; the guarded data must be protected on **every** access, **reads included**. Lock and data live together in one type, with the locking hidden inside the methods.
- **Fixes, measured:** the counter is exactly 1,000,000 (129 ms, vs 4 ms racy); the bank account allows one $60 withdrawal from $100 (balance 40) because **check and act are one critical section**.
- Waiting goroutines are **parked, not spinning**: 1,000 blocked goroutines used 0 s of CPU; contention costs *parallelism*, not CPU.
- **`defer mu.Unlock()`** always; `defer` runs LIFO. Without it, a recovered panic left the mutex locked forever (deadlock).
- **Failure modes, observed:** `fatal error: sync: unlock of unlocked mutex`; the **self-deadlock** (mutexes aren't reentrant → `xLocked` helpers); the **lock-order deadlock** (one global order); the **copied mutex** (`go vet`, `n = 0`).
- **`RWMutex`** gave only **1.7×** for tiny read-heavy sections at 12 CPUs (readers still contend on the reader count) and was *slower* on one CPU when writes dominate: benchmark before choosing.
- **Atomics** (`atomic.Int64`) did the counter in **28 ms (4.6× faster than the mutex)**, but only work for a single independent word.
- In our project, deleting one `Lock` from `database.UserStore.Create` made plain `go test` **pass** and `go test -race` **fail**: always run `-race` on concurrent tests.

### ➡️ What's next?

[Chapter 69](69-channels.md) introduces **channels**: passing data *between* goroutines instead of sharing it. We'll measure blocking and buffering, see the `all goroutines are asleep` deadlock from the channel side, use `close`, `range` and `select`, and learn to avoid goroutine leaks.
