# Chapter 69: Channels — Passing Data Between Goroutines

> **Goal of this chapter:** Learn Go's other answer to shared data: **don't share it, hand it over**. You'll create channels (`make(chan T)`), send and receive, and *measure* what "blocking" means (a sender waited exactly 300 ms for a late receiver); compare **unbuffered** (rendezvous) with **buffered** channels; provoke and read the three deadlock reports (`chan send`, `chan receive`, `chan send (nil chan)`); learn the closing rules (`close`, `range`, the comma-ok form, and the two panics); use `select` for **timeouts**, **non-blocking** operations and **first-of-many**; build a **worker pool** (9 jobs in 150 ms on 3 workers), a cancellable **pipeline**, and a **semaphore**; find and fix a **goroutine leak** (201 stuck goroutines); and get honest numbers on what channels cost compared with mutexes and atomics.

**Difficulty:** 🔴 Advanced  **Estimated time:** 8 hours  **Prerequisite:** [Chapters 65-68](68-sync-mutex.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The idea: a typed pipe](#2-the-idea-a-typed-pipe)
3. [Creating, sending, receiving](#3-creating-sending-receiving)
4. [Blocking, measured](#4-blocking-measured)
5. [Unbuffered vs. buffered](#5-unbuffered-vs-buffered)
6. [Deadlock, from the channel side](#6-deadlock-from-the-channel-side)
7. [Closing channels](#7-closing-channels)
8. [Channel directions](#8-channel-directions)
9. [`select`](#9-select)
10. [Patterns: worker pool, pipeline, semaphore](#10-patterns-worker-pool-pipeline-semaphore)
11. [Goroutine leaks](#11-goroutine-leaks)
12. [Channels or mutexes?](#12-channels-or-mutexes)
13. [The rules on one page](#13-the-rules-on-one-page)
14. [Common mistakes](#14-common-mistakes)
15. [Interview questions](#15-interview-questions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- What a **channel** is and how to create, send, receive, and close one
- **Blocking** semantics, measured
- **Unbuffered** vs. **buffered** channels; `len` and `cap`
- The three **deadlock** reports and what triggers each
- `close`, `range`, comma-ok receive, and the panics on misuse
- **Directional** channel types (`chan<-`, `<-chan`)
- `select`: timeouts, `default`, and waiting on several channels
- Patterns: **worker pool**, **pipeline** with cancellation, **semaphore**
- **Goroutine leaks**: detection and prevention
- When to choose channels and when a mutex is simpler

---

## 2. The idea: a typed pipe

Chapter 68 protected shared memory with locks. Go's philosophy offers an alternative:

> **Do not communicate by sharing memory; share memory by communicating.**

Instead of many goroutines touching one variable, **one goroutine owns the data** and others *send it values*; or a value is **handed over** from one goroutine to another and only the receiver touches it afterwards. A **channel** is the conduit: a typed, synchronized queue that goroutines send to and receive from.

```
   producer goroutine ──► [ chan int ] ──► consumer goroutine
        (sends)          (a typed pipe)        (receives)
```

A channel gives you **two things at once**: transfer of the *value* and **synchronization** between the sender and receiver. In memory-model terms (Chapter 67): a send **happens before** the corresponding receive completes, so anything the sender wrote before sending is visible to the receiver. No mutex needed: the hand-over *is* the synchronization.

---

## 3. Creating, sending, receiving

```go
package main

import "fmt"

func main() {
	ch := make(chan string) // an unbuffered channel of strings

	go func() {
		ch <- "hello from the goroutine" // SEND: blocks until someone receives
	}()

	msg := <-ch // RECEIVE: blocks until someone sends
	fmt.Println(msg)
}
```

```
hello from the goroutine
```

| Syntax | Meaning |
|--------|---------|
| `ch := make(chan T)` | create an **unbuffered** channel carrying values of type `T` |
| `ch := make(chan T, n)` | create a **buffered** channel with room for `n` values |
| `ch <- v` | **send** `v` (blocks until it can be delivered/buffered) |
| `v := <-ch` | **receive** (blocks until a value is available) |
| `v, ok := <-ch` | receive; `ok` is `false` if the channel is closed and empty |
| `close(ch)` | announce "no more values will be sent" |
| `len(ch)`, `cap(ch)` | values currently buffered; buffer capacity |
| `for v := range ch` | receive until the channel is closed and drained |

A channel's **zero value is `nil`**; a nil channel blocks forever on both send and receive (section 6). Always create channels with `make`. Channels are **reference types**: passing one to a function shares the *same* channel. The element type can be anything, including structs, slices, other channels, and `struct{}` (a zero-size signal: "something happened", no data).

---

## 4. Blocking, measured

An unbuffered channel is a **rendezvous**: a send completes only when a receiver takes the value. Make the receiver late and time the sender:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int) // unbuffered

	go func() {
		time.Sleep(300 * time.Millisecond) // the receiver arrives late
		fmt.Println("  receiver: ready to receive")
		<-ch
	}()

	start := time.Now()
	fmt.Println("sender: sending...")
	ch <- 42 // blocks until the receiver is ready: a rendezvous
	fmt.Printf("sender: send completed after %v\n", time.Since(start).Round(50*time.Millisecond))
}
```

```
sender: sending...
  receiver: ready to receive
sender: send completed after 300ms
```

The sender stood still for 300 ms until the receiver arrived. **The blocked goroutine costs no CPU** (it's parked, like a goroutine waiting on a mutex: Chapter 68). The symmetric case also holds: a receiver blocks until a sender arrives. That coupling is the point: unbuffered channels *synchronize* the two goroutines at the hand-over.

---

## 5. Unbuffered vs. buffered

A **buffered** channel has a queue of fixed capacity. A send blocks only when the buffer is **full**; a receive blocks only when it is **empty**:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int, 3) // buffered: room for 3 values

	for i := 1; i <= 3; i++ {
		ch <- i // does not block: there is room in the buffer
		fmt.Printf("sent %d   len=%d cap=%d\n", i, len(ch), cap(ch))
	}

	go func() {
		time.Sleep(200 * time.Millisecond)
		fmt.Println("  receiver: took", <-ch)
	}()

	start := time.Now()
	fmt.Println("sending a 4th value: the buffer is full, so this blocks...")
	ch <- 4
	fmt.Printf("...unblocked after %v (when the receiver made room)\n", time.Since(start).Round(50*time.Millisecond))
	fmt.Printf("len=%d cap=%d\n", len(ch), cap(ch))
}
```

```
sent 1   len=1 cap=3
sent 2   len=2 cap=3
sent 3   len=3 cap=3
sending a 4th value: the buffer is full, so this blocks...
  receiver: took 1
...unblocked after 200ms (when the receiver made room)
len=3 cap=3
```

Three sends returned instantly (buffered), the fourth waited 200 ms until the receiver took a value and freed a slot. Values come out in **FIFO** order (`took 1`).

| | Unbuffered `make(chan T)` | Buffered `make(chan T, n)` |
|---|---------------------------|----------------------------|
| Send blocks when | no receiver is waiting | buffer is full |
| Receive blocks when | no sender is waiting | buffer is empty |
| Synchronization | sender and receiver **meet** (strong) | sender may run ahead by up to `n` values (weaker) |
| Good for | hand-offs, signals, strict ordering | smoothing bursts, decoupling speeds, bounding work, "fire and forget" up to n |
| Danger | deadlock if nobody's on the other side | hides that a consumer is too slow until the buffer fills; a buffer isn't "free" (memory ∝ capacity) |

**Choosing a capacity:** start with **unbuffered** (or 1). Add capacity for a *reason you can state* ("N workers each send one result", "a burst of at most K events"). A large arbitrary buffer (`make(chan T, 10000)`) usually **masks a design problem** and delays the moment the system tells you it is overloaded.

---

## 6. Deadlock, from the channel side

If every goroutine is blocked forever, the runtime aborts. Three flavors, each visible in the trace's goroutine state:

**Send with nobody receiving:**

```go
ch := make(chan int) // unbuffered
ch <- 1              // nobody will ever receive: main blocks forever
```
```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/tmp/pgt/c69/deadlock_send/main.go:7 +0x28
```

**Receive with nobody sending:**

```go
ch := make(chan int)
fmt.Println(<-ch) // nobody will ever send
```
```
goroutine 1 [chan receive]:
main.main()
```

**The nil channel:**

```go
var ch chan int // nil: never initialized with make
ch <- 1
```
```
goroutine 1 [chan send (nil chan)]:
main.main()
```

The bracketed state (`chan send`, `chan receive`, `chan send (nil chan)`) says exactly what the goroutine is waiting for, and the frame beneath gives the line. Remember from Chapter 65: **the runtime only detects a deadlock when *all* goroutines are blocked.** In a server, other goroutines keep running, so a stuck handler just hangs silently. That's one reason to bound every wait with a `select` and a timeout or context (section 9).

Conditions that cause channel deadlocks: sending with no receiver (or after the receiver exited early); receiving with no sender (or after the sender forgot to `close`, when using `range`); forgetting to `make` the channel; a cycle where A waits on B while B waits on A.

---

## 7. Closing channels

`close(ch)` tells receivers **"no more values are coming"**. Rules and behaviors:

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 3)
	ch <- 10
	ch <- 20
	close(ch) // "no more values will be sent"

	for v := range ch { // receives until the channel is closed AND drained
		fmt.Println("range got", v)
	}

	v, ok := <-ch // receiving from a closed, empty channel never blocks
	fmt.Printf("after close: v=%d ok=%v (zero value, ok=false)\n", v, ok)

	defer func() { fmt.Println("recovered:", recover()) }()
	ch <- 30 // sending on a closed channel panics
}
```

```
range got 10
range got 20
after close: v=0 ok=false (zero value, ok=false)
recovered: send on closed channel
```

- **Receivers keep receiving buffered values after `close`**, then get the zero value with `ok == false`.
- **`for v := range ch`** loops until the channel is closed *and* drained: the idiomatic consumer loop.
- **Sending on a closed channel panics** (`send on closed channel`).
- **Closing a closed channel panics** (`close of closed channel`); so does closing a nil channel.
- Closing is **optional**: only needed when receivers must learn that the stream is *finished* (for `range`, or a `done` broadcast). A channel that is simply garbage-collected needs no `close`.

**The golden rule: only the sender closes, and only one goroutine closes.** A receiver must never close a channel it doesn't own (the sender might still be sending: panic). With several senders, arrange for **one** coordinator to close after all senders are done, as the worker pool below does with a `WaitGroup`.

**Closing as broadcast:** `close(done)` wakes **every** goroutine waiting on `<-done`: a one-to-many "stop!" signal. This is the mechanism under `context.Context`'s `Done()` channel.

---

## 8. Channel directions

A channel type can be restricted to one direction, documenting intent and letting the compiler catch mistakes:

```go
func producer(out chan<- int) { out <- 1 }   // send-only:    can send, cannot receive or close... (it CAN close)
func consumer(in <-chan int)  { v := <-in }  // receive-only: can receive, cannot send or close
```

- `chan<- T`: send-only. `<-chan T`: receive-only. A bidirectional `chan T` converts to either implicitly; the reverse isn't allowed.
- The **receive-only** type can't be `close`d (a compile error): consistent with "the sender closes".
- Functions should take the **narrowest** direction they need, and **return** receive-only channels (`func generate() <-chan int`) so callers can't send into or close your channel.

---

## 9. `select`

`select` waits on **several** channel operations and runs the one that becomes ready first: a `switch` for channels.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	slow := make(chan string)
	go func() {
		time.Sleep(500 * time.Millisecond)
		slow <- "the slow answer"
	}()

	// 1. select with a timeout
	select {
	case msg := <-slow:
		fmt.Println("got:", msg)
	case <-time.After(100 * time.Millisecond):
		fmt.Println("timed out after 100ms")
	}

	// 2. select with default: a non-blocking receive
	select {
	case msg := <-slow:
		fmt.Println("got:", msg)
	default:
		fmt.Println("nothing ready right now (default ran immediately)")
	}

	// 3. wait for whichever comes first
	fast := make(chan string)
	go func() { time.Sleep(50 * time.Millisecond); fast <- "the fast answer" }()
	select {
	case msg := <-slow:
		fmt.Println("got:", msg)
	case msg := <-fast:
		fmt.Println("got:", msg)
	}
}
```

```
timed out after 100ms
nothing ready right now (default ran immediately)
got: the fast answer
```

Semantics:

- `select` **blocks** until one case can proceed; if several can, it picks one **at random** (no starvation, no reliance on order).
- A `default` case makes it **non-blocking**: it runs immediately if nothing is ready.
- `time.After(d)` returns a channel that receives after `d`: the standard **timeout**.
- Send cases work too: `case out <- v:`.
- A `nil` channel in a `select` is never ready: a trick to *disable* a case dynamically.
- Loop + `select` is the shape of most long-running goroutines: `for { select { case <-ctx.Done(): return; case job := <-jobs: handle(job) } }`.

Note in the third block that `slow` was *still* sending after 500 ms in the background, but `fast` won; that leftover sender is stuck until someone receives, a small **leak** if it ever happens for real (section 11).

---

## 10. Patterns: worker pool, pipeline, semaphore

### Worker pool

A fixed number of workers pull jobs from a channel: parallelism **bounded** by design (compare Chapter 62's load generator):

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type job struct{ id, n int }
type result struct{ id, square int }

func worker(id int, jobs <-chan job, results chan<- result, wg *sync.WaitGroup) {
	defer wg.Done()
	for j := range jobs { // ends when jobs is closed and drained
		time.Sleep(50 * time.Millisecond) // pretend work
		results <- result{j.id, j.n * j.n}
	}
}

func main() {
	jobs := make(chan job)
	results := make(chan result)

	var wg sync.WaitGroup
	for w := 1; w <= 3; w++ { // three workers
		wg.Add(1)
		go worker(w, jobs, results, &wg)
	}

	go func() { // producer: feed the jobs, then say "no more"
		for i := 1; i <= 9; i++ {
			jobs <- job{id: i, n: i}
		}
		close(jobs)
	}()

	go func() { // closer: when every worker is done, close results so the range below ends
		wg.Wait()
		close(results)
	}()

	start := time.Now()
	sum := 0
	for r := range results {
		sum += r.square
	}
	fmt.Printf("sum of squares 1..9 = %d, computed by 3 workers in %v (9 jobs x 50ms sequentially would be 450ms)\n",
		sum, time.Since(start).Round(50*time.Millisecond))
}
```

```
sum of squares 1..9 = 285, computed by 3 workers in 150ms (9 jobs x 50ms sequentially would be 450ms)
```

Three workers finished nine 50 ms jobs in **150 ms** (three waves). Notice the closing choreography, which is *the* thing to get right:

1. The **producer** closes `jobs` when it has sent everything → the workers' `range jobs` loops end.
2. A **closer goroutine** waits for all workers (`wg.Wait()`) and then closes `results` → the consumer's `range results` ends.
3. Nobody closes a channel they only *receive* from, and `results` is closed exactly once.

(In the results loop, no ordering is guaranteed: results arrive as workers finish.)

### Pipeline with cancellation

Stages connected by channels, each running in its own goroutine. The important part is that stages must **stop** when the consumer loses interest, or they leak:

```go
package main

import (
	"context"
	"fmt"
	"runtime"
	"time"
)

// generate emits 1, 2, 3, ... until ctx is cancelled.
func generate(ctx context.Context) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out) // the goroutine that SENDS is the one that closes
		for n := 1; ; n++ {
			select {
			case out <- n:
			case <-ctx.Done(): // the consumer has gone away: stop instead of blocking forever
				return
			}
		}
	}()
	return out
}

// square reads numbers and emits their squares.
func square(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			select {
			case out <- n * n:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	for v := range square(ctx, generate(ctx)) {
		fmt.Print(v, " ")
		if v >= 100 {
			break // we have seen enough
		}
	}
	fmt.Println()

	cancel() // tell the pipeline to shut down
	time.Sleep(50 * time.Millisecond)
	fmt.Println("goroutines left after cancel:", runtime.NumGoroutine())
}
```

```
1 4 9 16 25 36 49 64 81 100 
goroutines left after cancel: 1
```

The consumer `break`s after 100; `cancel()` closes `ctx.Done()`; **both stage goroutines notice in their `select` and exit**: only `main` remains (1 goroutine). Without the `ctx.Done()` cases, the generator would block forever on `out <- n` with nobody receiving: a leak. **Every send that might never be received needs a `select` with an escape hatch.**

### Semaphore

A buffered channel makes a counting semaphore: bounded concurrency in five lines (the manual version of `errgroup.SetLimit` from Chapter 65):

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	const limit = 3
	sem := make(chan struct{}, limit) // a buffered channel used as a counting semaphore

	var running, peak atomic.Int32
	var wg sync.WaitGroup

	start := time.Now()
	for i := 0; i < 9; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()

			sem <- struct{}{}        // acquire a slot (blocks when `limit` are already running)
			defer func() { <-sem }() // release it

			n := running.Add(1)
			for {
				old := peak.Load()
				if n <= old || peak.CompareAndSwap(old, n) {
					break
				}
			}
			time.Sleep(100 * time.Millisecond) // the work
			running.Add(-1)
		}()
	}
	wg.Wait()

	fmt.Printf("9 tasks of 100ms, at most %d at once: took %v, peak concurrency %d\n",
		limit, time.Since(start).Round(50*time.Millisecond), peak.Load())
}
```

```
9 tasks of 100ms, at most 3 at once: took 300ms, peak concurrency 3
```

Nine tasks, three at a time: 300 ms and a peak of exactly 3. `struct{}` is used because a semaphore carries **no data**, only "a slot is taken".

---

## 11. Goroutine leaks

A **leak** is a goroutine that can never finish, usually blocked forever on a channel. It holds its stack (and everything it references) until the program exits. Consider "ask three servers, use the fastest answer":

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

// firstResult asks three "servers" and returns the fastest answer.
// BUG: the unbuffered channel means the two slower goroutines block forever on their send.
func firstResultLeaky() string {
	ch := make(chan string) // unbuffered
	for i := 1; i <= 3; i++ {
		go func() {
			time.Sleep(time.Duration(i) * 10 * time.Millisecond)
			ch <- fmt.Sprintf("answer from server %d", i)
		}()
	}
	return <-ch // takes ONE value and returns; the other two senders are stuck forever
}

// FIX: a buffered channel with room for every sender lets them all finish.
func firstResultFixed() string {
	ch := make(chan string, 3)
	for i := 1; i <= 3; i++ {
		go func() {
			time.Sleep(time.Duration(i) * 10 * time.Millisecond)
			ch <- fmt.Sprintf("answer from server %d", i)
		}()
	}
	return <-ch
}

func main() {
	fmt.Println("goroutines at start:      ", runtime.NumGoroutine())

	for i := 0; i < 100; i++ {
		firstResultLeaky()
	}
	time.Sleep(200 * time.Millisecond)
	fmt.Println("after 100 leaky calls:    ", runtime.NumGoroutine(), "(each call left 2 goroutines stuck)")

	for i := 0; i < 100; i++ {
		firstResultFixed()
	}
	time.Sleep(200 * time.Millisecond)
	fmt.Println("after 100 fixed calls:    ", runtime.NumGoroutine(), "(no new leaks)")
}
```

```
goroutines at start:       1
after 100 leaky calls:     201 (each call left 2 goroutines stuck)
after 100 fixed calls:     201 (no new leaks)
```

The leaky version leaves **two blocked goroutines per call**; after 100 calls, 200 of them are stuck (201 with `main`). The fixed version (capacity = number of senders) leaves none: the 100 fixed calls added zero. In a server handling a million requests, the leaky version accumulates two million goroutines and their memory (~2 KB stack each plus whatever they reference), until the process dies.

How leaks happen, and cures:

| Cause | Cure |
|-------|------|
| Send with no receiver left (as above) | buffer sized for all senders; or `select` with `ctx.Done()` |
| Receive with no sender left, never closed | the sender closes; consumer uses `range`; or `select` with `ctx.Done()` |
| Waiting on a `WaitGroup` that never finishes | balance `Add`/`Done` |
| Forgotten background goroutines (tickers, workers) | tie their lifetime to a `context`; stop them on shutdown |

Detect leaks with `runtime.NumGoroutine()` in tests (before/after), the goroutine profile (`pprof`), or a helper like `go.uber.org/goleak`. **Every goroutine you start needs a known way to end.**

---

## 12. Channels or mutexes?

"Share memory by communicating" is good advice, not a law. Numbers first: three ways to count with many goroutines (a counter owned by one goroutine receiving increments over a buffered channel; a mutex; an atomic), plus the raw cost of a hand-off:

```
BenchmarkCounterMutex         	94256265	        12.71 ns/op
BenchmarkCounterMutex-4       	35560305	        34.64 ns/op
BenchmarkCounterMutex-12      	11765680	       113.4 ns/op
BenchmarkCounterAtomic        	125956706	         9.912 ns/op
BenchmarkCounterAtomic-4      	41570515	        25.58 ns/op
BenchmarkCounterAtomic-12     	45309105	        28.99 ns/op
BenchmarkCounterChannel       	29429562	        40.43 ns/op
BenchmarkCounterChannel-4     	24543244	        49.67 ns/op
BenchmarkCounterChannel-12    	20088201	        67.40 ns/op
BenchmarkPingPong             	 3242238	       351.7 ns/op
BenchmarkPingPong-12          	 3658204	       319.5 ns/op
```

- On **one CPU** the channel counter is 3× *slower* than the mutex (40 vs 12.7 ns): a channel operation locks internally and may involve the scheduler.
- At **12 CPUs**, the mutex collapses under contention (113 ns) while the channel-owner design *scales better* (67 ns, since the buffer batches the hand-offs); atomics still win (29 ns).
- A synchronous **hand-off** (unbuffered ping-pong, a round trip involving two goroutine switches) costs about **350 ns**: cheap in absolute terms, but ~25× a mutex acquire. Channels are for **coordination and ownership transfer**, not for protecting a hot counter.

Guidance:

| Use a **channel** when | Use a **mutex/atomic** when |
|------------------------|-----------------------------|
| transferring **ownership** of data (work items, results) | protecting a **cache/map/counter** many goroutines read and update |
| **coordinating** goroutines: signals, cancellation, pipelines, worker pools | the state is a small internal detail of one type |
| a goroutine **owns** a resource and serves requests | the critical section is tiny and the code is clearer with a lock |
| composing timeouts and multiple events with `select` | performance of a hot path matters and a benchmark says so |

The Go wiki's advice is exactly this: use whichever is **most expressive and simplest**. Our in-memory stores use a mutex (a small guarded slice); the concurrent list in the next chapter uses a `WaitGroup` and separate result variables (own-slot style), with a channel where it clarifies cancellation.

---

## 13. The rules on one page

| Situation | Behavior |
|-----------|----------|
| Send on unbuffered, no receiver | blocks |
| Send on buffered, buffer full | blocks |
| Receive, nothing available | blocks |
| Send/receive on **nil** channel | blocks forever |
| Receive from **closed** channel | never blocks: buffered values first, then zero value with `ok == false` |
| Send on **closed** channel | **panic** |
| `close` of closed or nil channel | **panic** |
| `range ch` | loops until closed and drained (never ends if nobody closes) |
| `select` with several ready cases | picks one at random |
| `select` with `default` | never blocks |
| Who closes | the **sender**, exactly once |

---

## 14. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Forgetting `make` (nil channel) | Blocks forever (`chan send (nil chan)`) | `make(chan T)` |
| 2 | Sending in `main` with no receiver | `all goroutines are asleep - deadlock!` | Receive in another goroutine, or buffer |
| 3 | Receiver closes the channel | `send on closed channel` panic | Only the sender closes |
| 4 | Closing twice | `close of closed channel` panic | One owner closes (or `sync.Once`) |
| 5 | `range` over a channel nobody closes | Blocks forever | Close it when done sending |
| 6 | Unbuffered channel, fewer receivers than senders | Leaked goroutines | Buffer for senders / `select` with `ctx.Done()` |
| 7 | Huge arbitrary buffer | Hides slow consumers; memory | Small, justified capacity |
| 8 | Sending while holding a mutex | Deadlock/serialization | Never block on channels under a lock |
| 9 | `time.After` in a hot loop | A new timer per iteration (garbage until it fires) | `time.NewTimer` + `Reset`, or a context deadline |
| 10 | Ignoring `ctx.Done()` in long-running goroutines | Leaks on cancellation | `select` on `ctx.Done()` |
| 11 | Assuming a `select` order | Random when several are ready | Design for any order |
| 12 | Using channels to protect a hot counter | 3-25× slower than atomics/mutex | Match the tool to the job |
| 13 | Copying values that contain a lock through a channel | Broken locks | Send pointers |
| 14 | Forgetting a WaitGroup for the closer goroutine | Results channel closed early or never | `wg.Wait()` then `close` |

---

## 15. Interview questions

**Q1. Buffered vs. unbuffered?**
Unbuffered: send and receive synchronize (rendezvous). Buffered: sends block only when full and receives only when empty.

**Q2. What happens when you receive from a closed channel?**
Buffered values are delivered first; afterwards it returns the zero value immediately, with `ok == false`.

**Q3. Who should close a channel?**
The sender (one designated goroutine), never a receiver; closing twice or sending after close panics.

**Q4. How do you implement a timeout for a channel operation?**
`select` with a `time.After(d)` case (or a context's `Done()` channel).

**Q5. What is a goroutine leak and how do you prevent it?**
A goroutine blocked forever (typically on a channel). Give every goroutine a way to exit: close signals, `ctx.Done()` in `select`, right-sized buffers.

**Q6. What does `select` do when several cases are ready?**
Chooses one pseudo-randomly.

**Q7. Explain "share memory by communicating".**
Instead of guarding shared data with locks, pass ownership of data through channels so only one goroutine touches it at a time.

**Q8. How do you limit concurrency with channels?**
A buffered channel as a semaphore (acquire by sending, release by receiving), or a fixed worker pool reading from a jobs channel.

**Q9. Are channels always better than mutexes?**
No: they cost more per operation and are best for ownership transfer and coordination; mutexes/atomics suit small shared state.

---

## 16. Exercises

### Exercise 1: Trace it
What does this print, and does it deadlock?

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
close(ch)
for v := range ch { fmt.Println(v) }
fmt.Println(<-ch)
```

<details><summary>Solution</summary>

`1`, `2`, then `0` (a receive from a closed, drained channel returns the zero value immediately). No deadlock: the buffer holds both values and `close` lets `range` finish.
</details>

### Exercise 2: Fix the deadlock
Make this work without changing `main`'s structure much:

```go
func main() {
	ch := make(chan int)
	ch <- 42
	fmt.Println(<-ch)
}
```

<details><summary>Solution</summary>

Either buffer it (`make(chan int, 1)`) so the send doesn't need a receiver, or send from a goroutine: `go func() { ch <- 42 }()`.
</details>

### Exercise 3: Timeouts
Write `func fetchWithTimeout(d time.Duration, work func() string) (string, error)` returning an error if `work` takes longer than `d`. Make sure the goroutine running `work` doesn't leak when the timeout wins.

<details><summary>Solution</summary>

Run `work` in a goroutine sending into a channel with **capacity 1**, then `select` on that channel and `time.After(d)`. The buffer of 1 lets the abandoned goroutine complete its send and exit even after the timeout won.
</details>

### Exercise 4: Fan-in
Write `merge(cs ...<-chan int) <-chan int` that forwards values from several channels into one, closing the output after all inputs are closed.

<details><summary>Solution</summary>

One goroutine per input copies values to `out`; a `WaitGroup` counts them; a final goroutine does `wg.Wait(); close(out)`. Order between inputs is not defined.
</details>

### Exercise 5: A rate limiter with a ticker
Use `time.NewTicker` to allow at most 5 operations per second across many goroutines (each waits for a tick before proceeding). What must you remember to do with the ticker?

<details><summary>Solution</summary>

`limiter := time.NewTicker(200 * time.Millisecond)`; each operation does `<-limiter.C` first. Call `limiter.Stop()` (usually deferred) when done: an unstopped ticker leaks a timer.
</details>

### Exercise 6 (challenge): Prove the leak, then detect it in a test
Write a test using `runtime.NumGoroutine()` that fails for `firstResultLeaky` and passes for `firstResultFixed`. Why do you need a short sleep or polling loop in the test?

<details><summary>Solution</summary>

Record the count, run the function many times, then *poll* until the count returns to the baseline (or a deadline passes); assert the difference is 0. The extra goroutines take a moment to finish or be reaped, so an immediate comparison is flaky. (`goleak.VerifyNone(t)` does this properly.)
</details>

---

## 17. Quiz

1. What does an unbuffered send wait for?
2. What does a receive from a closed channel return?
3. Which goroutine should close a channel?
4. What does `select` do with a `default` case?
5. What does the trace state `chan send (nil chan)` mean?
6. Why can `for v := range ch` block forever?
7. How do you turn a buffered channel into a semaphore?
8. Why did the leaky `firstResult` leave 2 goroutines per call?

<details><summary>Answers</summary>

1. A receiver to take the value.
2. Buffered values first, then the zero value with `ok == false`, without blocking.
3. The sender (a single designated goroutine).
4. Runs `default` immediately when no other case is ready (non-blocking).
5. The goroutine is blocked sending on a nil (never-`make`d) channel.
6. If no one ever closes the channel and no more values arrive.
7. Capacity = the limit; send to acquire, receive to release.
8. Two of the three senders blocked forever on an unbuffered channel after the first answer was received.
</details>

---

## 18. Summary

- A **channel** is a typed conduit that both **transfers a value** and **synchronizes** sender and receiver (send happens-before receive): "share memory by communicating".
- **Unbuffered** channels rendezvous (a sender waited **300 ms** for a late receiver, using no CPU); **buffered** channels block only when full/empty (the 4th send into a size-3 buffer waited 200 ms for a receiver); start unbuffered and justify any capacity.
- **Deadlock reports** name the state: `chan send`, `chan receive`, `chan send (nil chan)`; the runtime only detects them when *all* goroutines are stuck, so real servers need `select` + timeouts/contexts.
- **Closing:** only the sender closes, once; receivers drain buffered values then see zero/`ok=false`; `range` ends at close; sending on or re-closing a closed channel panics; closing broadcasts to all receivers.
- **`select`** gives timeouts (`time.After`), non-blocking operations (`default`), and first-of-many; ready cases are chosen randomly.
- **Patterns:** worker pool (9 jobs, 3 workers → **150 ms**; close `jobs` after producing, close `results` after `wg.Wait()`), cancellable **pipeline** (both stages exited on `cancel()`: 1 goroutine left), **semaphore** (peak concurrency exactly 3).
- **Leaks** are goroutines that can never finish: the leaky helper left **200** stuck goroutines after 100 calls; the right-sized buffer fixed it. Every goroutine needs a known exit.
- **Cost:** a channel hand-off is ~350 ns vs. ~13-45 ns for a mutex; use channels for **ownership and coordination**, mutexes/atomics for small shared state, whichever is simplest.

### ➡️ What's next?

[Chapter 70](70-channels-and-goroutines-in-practice.md) puts it all to work in our service: run `Count` and `List` **concurrently** in the paginated product list, with correct error handling (one error variable per goroutine), **cancellation** when either fails, tests under `-race` (including a proof that the overlap really happens), an honest measurement on 500,000 products, and then a wrap-up of the whole course.
