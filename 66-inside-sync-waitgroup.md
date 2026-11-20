# Chapter 66: Inside `sync.WaitGroup` — One Atomic Word and a Semaphore

> **Goal of this chapter:** Open the hood. You'll see how `WaitGroup` fits **two counters into one 64-bit word** and manipulates it with atomic instructions instead of locks, how a waiting goroutine is put to sleep on a **runtime semaphore** (and how the runtime finds it again), why the panics from Chapter 65 exist, and what the `noCopy` field is for. To make it concrete you'll read a **working re-implementation** (`minigroup`, about 70 lines, tested under the race detector 20 times in a row) that follows the real design, watch its state bits change step by step, and benchmark it against the real one and against a mutex-protected counter. This is optional knowledge for daily use, but it is the best available introduction to how Go's concurrency primitives are built.

**Difficulty:** 🔴 Advanced  **Estimated time:** 5 hours  **Prerequisite:** [Chapter 65](65-sync-waitgroup.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The structure](#2-the-structure)
3. [Bits: two counters in one word](#3-bits-two-counters-in-one-word)
4. [Atomic operations](#4-atomic-operations)
5. [How `Add` and `Done` work](#5-how-add-and-done-work)
6. [How `Wait` works](#6-how-wait-works)
7. [The runtime semaphore](#7-the-runtime-semaphore)
8. [Building our own: `minigroup`](#8-building-our-own-minigroup)
9. [Watching the state change](#9-watching-the-state-change)
10. [Why the panics exist](#10-why-the-panics-exist)
11. [Why lock-free? A benchmark](#11-why-lock-free-a-benchmark)
12. [Odds and ends: `noCopy`, the race detector, `synctest`](#12-odds-and-ends-nocopy-the-race-detector-synctest)
13. [Questions you may have](#13-questions-you-may-have)
14. [Common mistakes](#14-common-mistakes)
15. [Exercises](#15-exercises)
16. [Quiz](#16-quiz)
17. [Summary](#17-summary)

---

## 1. What you will learn

- The fields of `sync.WaitGroup` and its **16-byte** size
- How two 32-bit quantities share one `atomic.Uint64`, and the shifts and masks that separate them
- **Compare-and-swap (CAS)** loops and why they replace locks here
- The exact protocol of `Add`, `Done`, `Wait`, and the wake-up of sleeping waiters
- What a **semaphore** is and how the runtime parks and readies goroutines
- Which invariant each `WaitGroup` panic protects
- Why a lock-free counter beats a mutex under contention (measured)

> **A note on versions.** The details below are from the standard library of **Go 1.25** (the `sync/waitgroup.go` and `runtime/sema.go` files on your machine: `$(go env GOROOT)/src/sync/waitgroup.go`). Internals change between releases: the *design* has been stable for years, but field layout, helper names, and extra features (like the `synctest` support bit) move. Read your own installed source to confirm.

---

## 2. The structure

```go
type WaitGroup struct {
	noCopy noCopy            // zero-size marker: makes `go vet` flag copies (section 12)

	state atomic.Uint64      // counter AND waiter count, packed into one 64-bit word
	sema  uint32             // a runtime semaphore: where waiting goroutines sleep
}
```

That is the whole type: a marker, one 64-bit atomic word, and a 32-bit semaphore. A real measurement:

```
sizeof(sync.WaitGroup): 16 bytes
```

(8 for `state`, 4 for `sema`, plus padding to keep the struct 8-byte aligned.) A `WaitGroup` is *tiny*: you can afford one per request or per batch.

---

## 3. Bits: two counters in one word

`WaitGroup` must remember two numbers:

1. the **task counter**: how many `Done` calls are still owed (what `Add`/`Done` change), and
2. the **waiter count**: how many goroutines are currently blocked in `Wait`.

Both are stored in the single `state` word:

```
 63                              32 31                               0
 ┌───────────────────────────────────┬───────────────────────────────────┐
 │        task counter (v)           │   (a flag bit) + waiter count (w) │
 │        high 32 bits               │        low 32 bits                │
 └───────────────────────────────────┴───────────────────────────────────┘
```

(In Go 1.25 one bit of the low half is reserved for a *"this group belongs to a `testing/synctest` bubble"* flag, so the waiter count uses the low 31 bits; the code masks it with `0x7fffffff`. Older versions used all 32.)

**Why pack two numbers into one word?** Because a *single* atomic instruction can then change **both consistently**. If they were separate variables, another goroutine could observe the counter after it changed but before the waiter count did: a torn view that would cause missed or premature wake-ups.

### Extracting and updating the halves

```go
counter := int32(state >> 32)      // shift the high half down; int32 so that "negative" is detectable
waiters := uint32(state & 0x7fffffff) // mask off everything but the low bits
```

```go
state.Add(uint64(delta) << 32)     // Add(3): adds 3 in the HIGH half; the low half is untouched
state + 1                          // registering a waiter: adds 1 in the LOW half
```

Try it on paper. State after `Add(3)`: `counter = 3, waiters = 0`:

```
high: 00000000 00000000 00000000 00000011      low: 00000000 00000000 00000000 00000000
```

A goroutine calls `Wait`: `low += 1`:

```
high: ...00000011                              low: ...00000001        counter 3, waiters 1
```

`Done()` is `Add(-1)`, i.e., adding `-1 << 32` as an unsigned number (two's complement subtracts 1 from the high half). Why does `int32(state >> 32)` reveal a negative counter? Because subtracting from a zero high half wraps to `0xFFFFFFFF`, which is `-1` as an `int32`: that's how "negative WaitGroup counter" is detected *after* the (atomic) subtraction has already happened.

---

## 4. Atomic operations

An **atomic** operation completes as one indivisible step: no other goroutine can observe it half-done. Modern CPUs provide them as single instructions (`LOCK XADD`, `LOCK CMPXCHG` on x86; `LDADD`/`CAS` on ARM64), exposed in Go by `sync/atomic`. The two we need:

| Operation | Meaning |
|-----------|---------|
| `x.Add(delta)` | atomically `x += delta`, and return the **new** value |
| `x.CompareAndSwap(old, new)` | atomically: *if* `x == old` then set `x = new` and return `true`; *else* change nothing and return `false` |

`Add` never fails. **CAS is the building block for optimistic updates**: read the value, compute what you want to write, and write it *only if nobody changed it in the meantime*. If someone did, CAS fails and you retry with the fresh value:

```go
for {
	old := x.Load()            // 1. read
	new := f(old)              // 2. compute a new value from what we read
	if x.CompareAndSwap(old, new) {   // 3. write only if x is still `old`
		break                  //    success
	}                          //    somebody else got there first: loop and try again
}
```

No lock is held at any point, so a goroutine that is descheduled in the middle of this loop can't block the others (that is what "**lock-free**" means).

---

## 5. How `Add` and `Done` work

`Done` is literally `Add(-1)` (you saw `sync.(*WaitGroup).Done(...)` → `Add` in the panic trace). `Add(delta)` does this:

```
1. state = wg.state.Add(delta << 32)          ← ONE atomic instruction changes the counter half
   v = counter half (as int32),  w = waiter half

2. if v < 0                          → panic("sync: negative WaitGroup counter")
3. if w != 0 && delta > 0 && v == delta
                                     → panic("...Add called concurrently with Wait")
4. if v > 0 || w == 0                → return         (work outstanding, or nobody is waiting)

   -- here: counter is now 0 AND there are waiters: it is our job to wake them --
5. if wg.state.Load() != state       → panic("...misuse")   (someone changed it: illegal)
6. wg.state.Store(0)                                        (reset both halves)
7. for ; w > 0; w-- { runtime_Semrelease(&wg.sema) }        (wake each waiter once)
```

Follow the two common cases:

- **`Done` with tasks remaining** (`v > 0`): one atomic add and a return. This is the *fast path* and the overwhelmingly common one: a handful of nanoseconds, no locks, no scheduler involvement.
- **The last `Done`** (`v == 0`, `w > 0`): the goroutine that brings the counter to zero **wakes the waiters** by releasing the semaphore once per waiter. Exactly one goroutine reaches step 7, because only one `Add` can produce the state where the counter hits zero.

Step 3 catches an `Add` from zero that races with a `Wait` (Chapter 65, mistake 1's cousin: "calls with a positive delta that occur when the counter is zero must happen before a Wait").

---

## 6. How `Wait` works

```
loop:
  state = wg.state.Load()
  v = counter half, w = waiter half

  if v == 0                          → return            (nothing outstanding: don't block)

  if wg.state.CompareAndSwap(state, state + 1) {        (register myself: waiter count + 1)
        runtime_SemacquireWaitGroup(&wg.sema)           (SLEEP until a Semrelease wakes me)
        if wg.state.Load() != 0      → panic("...reused before previous Wait has returned")
        return
  }
  (CAS failed: the state changed under us: go around the loop and look again)
```

Three observations:

1. **The CAS is the correctness core.** Between reading the state and registering as a waiter, the last `Done` might run and bring the counter to 0. The CAS notices (the state word changed, so `old != current`), fails, and the loop re-reads: now `v == 0`, so `Wait` returns without sleeping. Without CAS, a `Wait` could go to sleep *after* the last `Done` had already issued its wake-ups, and **sleep forever** ("lost wake-up").
2. **`Wait` can be called by several goroutines**: each registers (`w` increases) and each is woken once.
3. After waking, the *state must be zero*. If it isn't, someone called `Add` again before all the waiters of the old batch had returned: the "reused" panic (section 10).

---

## 7. The runtime semaphore

A **semaphore** is a counter with two operations: *acquire* (wait until the counter is positive, then decrement) and *release* (increment, waking a sleeper if any). `WaitGroup.sema` is a plain `uint32` whose *address* identifies a semaphore managed by the Go runtime (`runtime/sema.go`).

**`runtime_SemacquireWaitGroup(&wg.sema)`**, roughly:

1. **Fast path:** try to atomically decrement the semaphore if it's positive (`cansemacquire`). If that works, return without sleeping.
2. **Slow path:** find this address's *bucket* in a global table, create a small record for this goroutine (a **`sudog`**: "goroutine waiting on address X"), add it to the bucket's wait queue, re-check the semaphore once more (to avoid a missed wake-up), and then **park** the goroutine (`gopark`): mark it *waiting*, take it off the run queue and let its thread run something else.

**`runtime_Semrelease(&wg.sema)`**: increment the semaphore; look up the bucket; if a `sudog` is waiting for that address, **dequeue** it and mark its goroutine **runnable** (`goready`): the scheduler will run it soon. The waiter wakes inside `Semacquire`, decrements the semaphore, and returns into `Wait`.

### The global table

Where do waiters wait? Not inside the `WaitGroup` (16 bytes has no room for a queue!). The runtime keeps **one global table of 251 buckets** (`semTabSize = 251`); an address is mapped to a bucket by hashing (`(address >> 3) % 251`). Each bucket holds a balanced tree (a *treap*) of `sudog`s keyed by address, protected by a small lock. So millions of `WaitGroup`s and mutexes can be waited on, and the memory for the queues exists only while goroutines are actually sleeping.

```
   wg1.sema (addr 0xc000012118) ──hash──►  bucket 17 ──► treap: [sudog(G7), sudog(G12)]  (two goroutines waiting on wg1)
   wg2.sema (addr 0xc0000a4230) ──hash──►  bucket 203 ──► treap: [sudog(G3)]
   ...251 buckets, each with its own lock
```

That's the "global map that tracks sleeping goroutines". It's why **sleeping is cheap** (a goroutine blocked in `Wait` uses no CPU, only its ~2 KB stack and one `sudog`), and why a deadlock trace shows the goroutine's state as `[sync.WaitGroup.Wait]`: the runtime remembers *why* each goroutine was parked (its `waitReason`).

The `G` structure, by the way, is the runtime's per-goroutine record (Chapter 64's "G"): its stack bounds, saved registers, status (running / runnable / waiting), the reason it is waiting, and more. `gopark` changes its status to *waiting*; `goready` changes it back to *runnable* and puts it on a run queue.

---

## 8. Building our own: `minigroup`

The best test of understanding is building it. `minigroup` mirrors the design above, replacing the runtime semaphore (which user code can't call) with an **unbuffered channel**: a goroutine blocked receiving from a channel is asleep in exactly the same way (Chapter 69), and sending on the channel wakes one receiver.

```go
// file: minigroup/minigroup.go
// Package minigroup is a teaching re-implementation of sync.WaitGroup.
//
// It follows the same design as the real one: a single 64-bit atomic word holds both the task
// counter (high 32 bits) and the number of goroutines blocked in Wait (low 32 bits), so that
// Add, Done and Wait never need a mutex. The real WaitGroup sleeps waiters on a runtime
// semaphore; here an unbuffered channel plays that part.
package minigroup

import (
	"fmt"
	"sync/atomic"
)

// WaitGroup waits for a collection of tasks to finish. The zero value is ready to use.
type WaitGroup struct {
	state atomic.Uint64                 // high 32 bits: counter, low 32 bits: waiters
	wake  atomic.Pointer[chan struct{}] // created on first use; each waiter receives exactly one value
}

// channel returns the wake-up channel, creating it on first use. Racing creators are harmless:
// the compare-and-swap lets exactly one channel win, and everyone then uses the winner.
func (wg *WaitGroup) channel() chan struct{} {
	if p := wg.wake.Load(); p != nil {
		return *p
	}
	ch := make(chan struct{})
	wg.wake.CompareAndSwap(nil, &ch)
	return *wg.wake.Load()
}

// Add adds delta (which may be negative) to the counter. If the counter reaches zero, every
// goroutine blocked in Wait is released. If the counter goes negative, Add panics.
func (wg *WaitGroup) Add(delta int) {
	s := wg.state.Add(uint64(int64(delta)) << 32) // only the high half changes
	counter := int32(s >> 32)
	waiters := uint32(s)

	if counter < 0 {
		panic("minigroup: negative WaitGroup counter")
	}
	if waiters != 0 && delta > 0 && counter == int32(delta) {
		panic("minigroup: WaitGroup misuse: Add called concurrently with Wait")
	}
	if counter > 0 || waiters == 0 {
		return // still work outstanding, or nobody is waiting
	}

	// The counter just reached zero and there are waiters. Nobody may change the state now
	// (that would be misuse: Add racing with Wait).
	if wg.state.Load() != s {
		panic("minigroup: WaitGroup misuse: Add called concurrently with Wait")
	}
	wg.state.Store(0) // reset: the group can be reused once the waiters are gone
	wake := wg.channel()
	for ; waiters > 0; waiters-- {
		wake <- struct{}{} // release one sleeping waiter
	}
}

// Done decrements the counter by one.
func (wg *WaitGroup) Done() { wg.Add(-1) }

// Wait blocks until the counter is zero.
func (wg *WaitGroup) Wait() {
	for {
		s := wg.state.Load()
		if int32(s>>32) == 0 {
			return // nothing to wait for
		}
		// Register as a waiter (low half + 1). If another goroutine changed the state between
		// our Load and this CAS, try again with the fresh value.
		if wg.state.CompareAndSwap(s, s+1) {
			<-wg.channel() // sleep until Add releases us
			if wg.state.Load() != 0 {
				panic("minigroup: WaitGroup is reused before previous Wait has returned")
			}
			return
		}
	}
}

// String shows the two halves of the state word, for learning and debugging.
func (wg *WaitGroup) String() string {
	s := wg.state.Load()
	return fmt.Sprintf("counter=%d waiters=%d  bits=%032b|%032b", int32(s>>32), uint32(s), uint32(s>>32), uint32(s))
}
```

Compare with sections 5 and 6: `Add` has the same numbered steps (including the two misuse panics), and `Wait` has the same read/CAS/sleep loop. Differences from the real thing: a channel instead of the runtime semaphore (created lazily with a CAS so the zero value works), no race-detector or `synctest` hooks, and it wakes waiters with blocking channel sends.

Tests (run with `-race`, repeated 20 times because concurrency tests can pass by luck):

```go
// file: minigroup/minigroup_test.go
package minigroup

import (
	"strings"
	"sync"
	stdsync "sync"
	"sync/atomic"
	"testing"
	"time"
)

func TestWaitReturnsImmediatelyWhenNothingIsPending(t *testing.T) {
	var wg WaitGroup
	done := make(chan struct{})
	go func() { wg.Wait(); close(done) }()

	select {
	case <-done:
	case <-time.After(time.Second):
		t.Fatal("Wait on a zero counter must not block")
	}
}

func TestWaitBlocksUntilEveryTaskIsDone(t *testing.T) {
	var wg WaitGroup
	var finished atomic.Int32

	const n = 50
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			time.Sleep(time.Duration(i%5) * 5 * time.Millisecond)
			finished.Add(1)
		}()
	}
	wg.Wait()

	if got := finished.Load(); got != n {
		t.Errorf("Wait returned with %d of %d tasks finished", got, n)
	}
}

func TestSeveralGoroutinesMayWaitAtOnce(t *testing.T) {
	var wg WaitGroup
	wg.Add(1)

	var released atomic.Int32
	var waiting stdsync.WaitGroup
	for i := 0; i < 10; i++ {
		waiting.Add(1)
		go func() {
			defer waiting.Done()
			wg.Wait()
			released.Add(1)
		}()
	}

	time.Sleep(50 * time.Millisecond) // let all ten reach Wait and go to sleep
	if released.Load() != 0 {
		t.Fatal("waiters must stay asleep while the counter is positive")
	}
	if !strings.Contains(wg.String(), "waiters=10") {
		t.Errorf("all ten should be registered as waiters: %s", wg.String())
	}

	wg.Done() // one Done releases all ten
	waiting.Wait()
	if released.Load() != 10 {
		t.Errorf("released %d of 10 waiters", released.Load())
	}
}

func TestNegativeCounterPanics(t *testing.T) {
	defer func() {
		r := recover()
		if r == nil || !strings.Contains(r.(string), "negative WaitGroup counter") {
			t.Errorf("expected the negative-counter panic, got %v", r)
		}
	}()
	var wg WaitGroup
	wg.Add(1)
	wg.Done()
	wg.Done()
}

func TestTheGroupCanBeReusedAfterWaitReturns(t *testing.T) {
	var wg WaitGroup
	for round := 0; round < 3; round++ {
		var count atomic.Int32
		for i := 0; i < 20; i++ {
			wg.Add(1)
			go func() { defer wg.Done(); count.Add(1) }()
		}
		wg.Wait()
		if count.Load() != 20 {
			t.Fatalf("round %d: %d of 20", round, count.Load())
		}
	}
}

func TestStateHalvesAreIndependent(t *testing.T) {
	var wg WaitGroup
	wg.Add(3)
	if got := wg.String(); !strings.HasPrefix(got, "counter=3 waiters=0") {
		t.Errorf("got %s", got)
	}
	wg.Add(-1)
	if got := wg.String(); !strings.HasPrefix(got, "counter=2 waiters=0") {
		t.Errorf("got %s", got)
	}
	wg.Add(-2)
	if got := wg.String(); !strings.HasPrefix(got, "counter=0 waiters=0") {
		t.Errorf("got %s", got)
	}
}

// The race detector is the real judge: hammer the group from many goroutines.
func TestNoRacesUnderLoad(t *testing.T) {
	var wg WaitGroup
	var mu sync.Mutex
	sum := 0
	for i := 1; i <= 500; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			mu.Lock()
			sum += i
			mu.Unlock()
		}()
	}
	wg.Wait()
	if sum != 500*501/2 {
		t.Errorf("sum = %d", sum)
	}
}
```

```
$ go test -race -count=20 ./...
ok  	minigroup	1.748s
```

Note `TestSeveralGoroutinesMayWaitAtOnce`: ten goroutines call `Wait`, the test *observes* `waiters=10` in the state word, then a single `Done` releases all ten. That's the low half of the word doing its job.

---

## 9. Watching the state change

A tiny program prints the state word (in decimal and binary) at each step. The waiting goroutine really goes to sleep, so the waiter bit is set:

```go
// file: minigroup/demo/main.go
package main

import (
	"fmt"
	"sync"
	"time"
	"unsafe"

	"minigroup"
)

func main() {
	var real sync.WaitGroup
	var mini minigroup.WaitGroup
	fmt.Println("sizeof(sync.WaitGroup):", unsafe.Sizeof(real), "bytes")
	fmt.Println("sizeof(mini WaitGroup):", unsafe.Sizeof(mini), "bytes")
	fmt.Println()

	fmt.Println("start:              ", mini.String())
	mini.Add(3)
	fmt.Println("Add(3):             ", mini.String())
	mini.Done()
	fmt.Println("Done():             ", mini.String())

	released := make(chan struct{})
	go func() {
		mini.Wait() // registers as a waiter, then sleeps
		close(released)
	}()
	time.Sleep(50 * time.Millisecond)
	fmt.Println("one goroutine Wait()s:", mini.String())

	mini.Done()
	fmt.Println("Done():             ", mini.String())
	mini.Done() // counter reaches 0: the waiter is released
	<-released
	fmt.Println("last Done + wake-up:", mini.String())
}
```

```
sizeof(sync.WaitGroup): 16 bytes
sizeof(mini WaitGroup): 16 bytes

start:               counter=0 waiters=0  bits=00000000000000000000000000000000|00000000000000000000000000000000
Add(3):              counter=3 waiters=0  bits=00000000000000000000000000000011|00000000000000000000000000000000
Done():              counter=2 waiters=0  bits=00000000000000000000000000000010|00000000000000000000000000000000
one goroutine Wait()s: counter=2 waiters=1  bits=00000000000000000000000000000010|00000000000000000000000000000001
Done():              counter=1 waiters=1  bits=00000000000000000000000000000001|00000000000000000000000000000001
last Done + wake-up: counter=0 waiters=0  bits=00000000000000000000000000000000|00000000000000000000000000000000
```

Read the bit columns left half (high, counter) and right half (low, waiters):

- `Add(3)`: the high half becomes `…011` (3). The low half is untouched.
- `Done()`: high half `…010` (2).
- The goroutine calls `Wait`: the **low half becomes `…001`**: one waiter registered (and asleep).
- The last `Done` drops the counter to zero; because `waiters = 1`, that `Done` also **resets the whole word to 0** and releases the waiter.

(The mini version is also 16 bytes: 8 for the state and 8 for the pointer to the channel.)

---

## 10. Why the panics exist

Each panic guards an *invariant* that, if broken, would make the state word meaningless:

| Panic | Invariant | Typical cause |
|-------|-----------|---------------|
| `sync: negative WaitGroup counter` | the counter is a count of *outstanding* tasks: never below 0 | one `Done` too many |
| `sync: WaitGroup misuse: Add called concurrently with Wait` | a positive `Add` from zero must not race with `Wait`; the wake-up logic assumes the waiters it releases belong to the batch that just ended | `Add` inside the goroutine (Chapter 65, mistake 1) |
| `sync: WaitGroup is reused before previous Wait has returned` | after the last `Done`, all waiters of that batch must return before the group starts a new batch (the state is reset while they wake) | `Add` for batch 2 while batch 1's `Wait` is still waking up |

These are **best-effort detections**, not guarantees: races can slip through without triggering them. Treat the panics as a *helpful bonus*, and follow the usage rules so you never rely on them.

---

## 11. Why lock-free? A benchmark

Design alternative: protect a plain integer counter with a `sync.Mutex`. What does the atomic design buy? A benchmark: many goroutines each doing `Add(1)` then `Done()` as fast as they can, with the number of CPUs allowed to run varied (`-cpu 1,4,12`):

```go
// file: minigroup/bench_test.go
package minigroup

import (
	"sync"
	"testing"
)

// Each benchmark: b.N pairs of Add(1)/Done() from many goroutines at once.

func BenchmarkRealWaitGroup(b *testing.B) {
	var wg sync.WaitGroup
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			wg.Add(1)
			wg.Done()
		}
	})
}

func BenchmarkMiniWaitGroup(b *testing.B) {
	var wg WaitGroup
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			wg.Add(1)
			wg.Done()
		}
	})
}

// A counter protected by a mutex, for comparison with the lock-free design.
type mutexCounter struct {
	mu sync.Mutex
	n  int
}

func (c *mutexCounter) Add(d int) {
	c.mu.Lock()
	c.n += d
	c.mu.Unlock()
}

func BenchmarkMutexCounter(b *testing.B) {
	var c mutexCounter
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			c.Add(1)
			c.Add(-1)
		}
	})
}
```

```
BenchmarkRealWaitGroup       	88380294	        14.06 ns/op
BenchmarkRealWaitGroup-4     	17937309	        58.90 ns/op
BenchmarkRealWaitGroup-12    	17886344	        76.18 ns/op
BenchmarkMiniWaitGroup       	75854469	        15.65 ns/op
BenchmarkMiniWaitGroup-4     	22414270	        61.74 ns/op
BenchmarkMiniWaitGroup-12    	17335034	        76.47 ns/op
BenchmarkMutexCounter        	26811451	        45.36 ns/op
BenchmarkMutexCounter-4      	 9262840	       126.2 ns/op
BenchmarkMutexCounter-12     	 4494861	       262.3 ns/op
```

What the numbers say (each `op` is an `Add` plus a `Done`... for the mutex version, two lock/unlock pairs):

1. **Our `minigroup` matches the real `WaitGroup`** within noise (15.7 vs 14.1 ns; 76.5 vs 76.2 ns at 12 CPUs): the design, not clever tuning, is what makes it fast.
2. **On one CPU, atomics are about 3× cheaper** than a mutex (14 vs 45 ns).
3. **Under contention, everything slows down**: more CPUs hammering the *same* memory location means the cache line holding it bounces between cores. The atomic versions go from 14 → 76 ns (5×), the mutex from 45 → 262 ns (5.8×). Sharing a hot variable is expensive **even when lock-free**.
4. At 12 CPUs the mutex is **3.4× slower** than the atomics (262 vs 76 ns), and it has an extra problem the benchmark doesn't show: under contention mutex waiters go to sleep and must be woken, costing far more than the raw numbers suggest.

The practical lesson isn't "always use atomics": it's that *the hot path of a shared counter is the one place where lock-free pays*, and that the best fix for contention is often **not sharing** (per-goroutine counters merged at the end, or the results-in-own-slot pattern from Chapter 65).

---

## 12. Odds and ends: `noCopy`, the race detector, `synctest`

**`noCopy`.** The zero-size field

```go
type noCopy struct{}
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

does nothing at run time. But because it has `Lock`/`Unlock` methods, `go vet`'s **copylocks** check treats any struct containing it as "a lock that must not be copied", producing the `passes lock by value` warning from Chapter 65. It costs zero bytes and turns a runtime deadlock into a compile-time-ish warning.

**Race detector hooks.** The real code contains `if race.Enabled { ... }` blocks that tell the race detector (`go test -race`) about the *happens-before* relationships `WaitGroup` creates: "everything before `Done` happens before the return of `Wait`". That's how the detector knows that reading `results` after `Wait` is safe. The hooks compile away entirely in normal builds.

**`synctest` bubbles.** Go 1.25 added `testing/synctest`, which runs a test in a "bubble" with a **fake clock**, where sleeping and timers advance instantly and deterministically when all goroutines are blocked. `WaitGroup` needs to know whether it belongs to a bubble (so that a bubble can tell that a goroutine blocked in `Wait` is "durably blocked"), hence the flag bit in the state word. You can ignore it until you write tests full of timers.

**Older layouts.** Before Go 1.20, the state was two `uint32`s in a 12-byte array arranged so that the 64-bit half was 8-byte aligned even on 32-bit platforms (atomic 64-bit operations require alignment there). Since `atomic.Uint64` guarantees alignment, that trick is gone. If you read old blog posts about "`state1 [3]uint32`", that's why.

---

## 13. Questions you may have

**Why not just use a `sync.Mutex` and a `sync.Cond`?**
You could (a `Cond` is a wait queue), and simple implementations do. The atomic-word design avoids taking a lock on the hot path (`Add`/`Done` with tasks outstanding), as the benchmark showed.

**What if a waiter is registered but the last `Done` hasn't run its releases yet?**
That's fine: the waiter is already sleeping (or about to sleep) on the semaphore. The semaphore is a *counter*: if the release happens *before* the waiter reaches `Semacquire`, the semaphore is already positive and the acquire's fast path succeeds without sleeping. That is exactly why a *semaphore* (not a bare "signal") is used: it remembers releases that happen early.

**Can `Wait` be called concurrently by many goroutines?**
Yes; each registers in the waiter half and each gets one release.

**Is `WaitGroup` fair? Which waiter wakes first?**
Unspecified: don't depend on it.

**How does the runtime avoid one giant lock for all semaphores?**
By the 251 buckets: unrelated waits usually hit different buckets and locks.

**Does a sleeping goroutine waste CPU?**
No: it's off the run queue; only memory (its stack plus a `sudog`) is used.

**Where can I read the real code?**
`$(go env GOROOT)/src/sync/waitgroup.go` (~250 lines) and `$(go env GOROOT)/src/runtime/sema.go`.

---

## 14. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Assuming internals are stable API | Code breaks on upgrades | Use only the documented methods |
| 2 | Building your own primitive for production | Subtle bugs (lost wake-ups) | Use `sync`; build toys only to learn |
| 3 | Reading counter and waiters separately (in a homemade version) | Torn reads, missed wake-ups | One atomic word / one lock |
| 4 | Store instead of CAS when registering a waiter | Lost update: a wake-up is missed, `Wait` sleeps forever | CAS loop |
| 5 | Using atomics for compound updates across two variables | Inconsistent views | Pack into one word or use a mutex |
| 6 | Believing lock-free means contention-free | Cache-line bouncing still hurts (5× in our benchmark) | Avoid sharing hot variables |
| 7 | Ignoring `go vet` copylocks warnings | Deadlocks from copied WaitGroups | Fix the warning |
| 8 | Reusing a WaitGroup while old waiters wake | Reuse panic | New group per batch |
| 9 | Treating misuse panics as a safety net | Races that don't panic pass silently | Follow the rules; use `-race` |
| 10 | Benchmarking without `-cpu` variations | Misses contention effects | `-cpu 1,4,N` |

---

## 15. Exercises

### Exercise 1: Decode a state word
The state word is `0x0000000500000003`. What are the counter and the waiter count? After one `Done`, what is the word? What if the counter were `1` instead of `5`?

<details><summary>Solution</summary>

High half `0x00000005` → counter 5; low half `0x00000003` → 3 waiters. One `Done` subtracts `1<<32`: `0x0000000400000003` (counter 4). With counter 1, `Done` brings it to 0 while 3 waiters exist → the state is reset to 0 and three releases are issued.
</details>

### Exercise 2: Break the CAS
In `minigroup.Wait`, replace the CAS with a plain `wg.state.Store(s + 1)` (read-then-store). Write a stress test that shows a lost wake-up (a `Wait` that never returns). Why does it happen?

<details><summary>Solution</summary>

Interleaving: the waiter loads `s` (counter 1); the last `Done` runs (counter 0, waiters 0 → returns without waking anyone); the waiter stores `s+1` (counter 1, waiters 1) *overwriting* the finished state, then sleeps: no one will ever call `Done` again. A stress test with many iterations of "one `Done` racing one `Wait`" and a timeout per iteration will hang or time out. CAS prevents it because the store only succeeds if the state is still what was read.
</details>

### Exercise 3: Add a `Go` method
Add `func (wg *WaitGroup) Go(f func())` to `minigroup`. Test that it makes the "Add inside goroutine" bug impossible.

<details><summary>Solution</summary>

```go
func (wg *WaitGroup) Go(f func()) {
	wg.Add(1)
	go func() {
		defer wg.Done()
		f()
	}()
}
```
Test: launch 100 `Go` calls, `Wait`, assert 100 tasks ran. It can't return early because `Add(1)` runs in the caller before the goroutine exists.
</details>

### Exercise 4: Per-goroutine counters
Modify the benchmark so each goroutine has its *own* counter and they're summed at the end. Compare with the shared atomic at 12 CPUs. What does it tell you?

<details><summary>Solution</summary>

Independent counters (padded to separate cache lines) scale nearly linearly: each op stays near the single-CPU cost (~14 ns) because no cache line is shared. The lesson: contention costs come from *sharing*; design to avoid it.
</details>

### Exercise 5: Read the real thing
Open `sync/waitgroup.go` and `runtime/sema.go` on your machine. Find: (a) the line that packs the counter, (b) `semTabSize`, (c) where a waiter is parked. Which parts did `minigroup` leave out?

<details><summary>Solution</summary>

(a) `wg.state.Add(uint64(delta) << 32)` in `Add`; (b) `const semTabSize = 251`; (c) `goparkunlock` inside `semacquire1`. `minigroup` omits the race-detector hooks, `synctest` bubble handling, the runtime semaphore and its tree of `sudog`s (replaced by a channel), and the profiling/trace hooks.
</details>

### Exercise 6 (challenge): A semaphore of your own
Implement a counting semaphore `Sem` with `Acquire()` and `Release()` using only `sync/atomic` and a channel-free wait strategy (for instance `runtime.Gosched()` spinning), then compare with a buffered channel used as a semaphore in a benchmark. Why is spinning a bad idea when waits are long?

<details><summary>Solution</summary>

Spinning burns a CPU for the whole wait and can starve the very goroutine that would release the semaphore (especially with `GOMAXPROCS=1`); the runtime's semaphore *parks* the waiter (zero CPU) and wakes it when needed. The buffered channel version is nearly as fast for short waits and vastly better for long ones.
</details>

---

## 16. Quiz

1. What two numbers does `WaitGroup.state` hold, and in which halves?
2. Why pack them into one word?
3. What does `CompareAndSwap(old, new)` do?
4. What prevents a lost wake-up between `Wait`'s read of the state and its sleep?
5. Where do sleeping goroutines wait, given that a `WaitGroup` is only 16 bytes?
6. What is `noCopy` for?
7. Why does `Done` with tasks remaining cost only a few nanoseconds?
8. Why was the mutex counter slower under contention?

<details><summary>Answers</summary>

1. Task counter in the high 32 bits; waiter count in the low bits.
2. One atomic instruction can then update/observe both consistently.
3. Atomically sets `new` only if the current value equals `old`, returning whether it did.
4. The registration is a CAS; if the last `Done` changed the state, the CAS fails and `Wait` re-reads (and returns); the semaphore also remembers early releases.
5. In the runtime's global semaphore table (251 buckets of `sudog` trees), keyed by the semaphore's address.
6. It makes `go vet` flag copies of a `WaitGroup` (or anything containing one).
7. It is one atomic add and a couple of comparisons: no lock, no scheduler work.
8. Lock acquisition serializes goroutines, and contended waiters must sleep and be woken; atomics only pay for cache-line traffic.
</details>

---

## 17. Summary

- `sync.WaitGroup` is **16 bytes**: a `noCopy` marker, a `state atomic.Uint64` (**task counter** in the high half, **waiter count** in the low bits) and a `sema uint32`.
- `Add`/`Done` is **one atomic add** on the high half; only the *last* `Done` (counter → 0 with waiters) takes the slow path: reset the state and **release the semaphore once per waiter**.
- `Wait` is a **read → CAS-register → sleep** loop: the CAS guarantees a waiter can't register after the wake-ups were issued; the runtime semaphore (which counts early releases) guarantees it can't miss one.
- Sleeping goroutines are recorded as `sudog`s in the runtime's **251-bucket semaphore table**, parked with `gopark` and readied with `goready`: no CPU while asleep, and the reason (`sync.WaitGroup.Wait`) appears in deadlock traces.
- The three **panics** guard the invariants of the state word; they are best-effort diagnostics, not guarantees.
- We built **`minigroup`** with the same protocol, tested it under `-race` ×20, watched its bits change, and benchmarked it against the real one (identical) and a mutex (3.4× slower at 12 CPUs). Contention on a shared word hurts even lock-free code (14 → 76 ns): the best fix is **not sharing**.

### ➡️ What's next?

[Chapter 67](67-race-conditions.md) shows what happens when goroutines **do** share data without protection: **race conditions**. We'll reproduce lost updates with real numbers, use the race detector to catch them, and find the same class of bug in code shaped like our own service.
