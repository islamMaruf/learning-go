# Chapter 65: `sync.WaitGroup` — Waiting for Goroutines the Right Way

> **Goal of this chapter:** Replace the `time.Sleep` guess with **`sync.WaitGroup`**, Go's tool for "wait until all of these goroutines have finished". You'll learn its three methods (`Add`, `Done`, `Wait`) and the newer `Go`, see with **real output** what each classic mistake does (a `Wait` that returns too early, the fatal *deadlock* trace, the *negative counter* panic, the *copied* WaitGroup that `go vet` catches), learn the patterns for collecting **results** and **errors** safely (including `errgroup` with cancellation and concurrency limits), and build and test a small concurrent package with the race detector.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 5 hours  **Prerequisite:** [Chapter 64](64-why-concurrency-matters.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The problem with `time.Sleep`](#2-the-problem-with-timesleep)
3. [WaitGroup: a counter you can wait on](#3-waitgroup-a-counter-you-can-wait-on)
4. [The API](#4-the-api)
5. [Basic usage](#5-basic-usage)
6. [The classic mistakes, with real output](#6-the-classic-mistakes-with-real-output)
7. [Collecting results and errors](#7-collecting-results-and-errors)
8. [`wg.Go` (Go 1.25)](#8-wggo-go-125)
9. [`errgroup`: errors, cancellation, limits](#9-errgroup-errors-cancellation-limits)
10. [A complete example: `checker`](#10-a-complete-example-checker)
11. [Revisiting the load generator](#11-revisiting-the-load-generator)
12. [Testing concurrent code](#12-testing-concurrent-code)
13. [Common mistakes](#13-common-mistakes)
14. [Interview questions](#14-interview-questions)
15. [Exercises](#15-exercises)
16. [Quiz](#16-quiz)
17. [Summary](#17-summary)

---

## 1. What you will learn

- Why sleeping is the wrong way to wait, with numbers
- The **counter model** of `sync.WaitGroup` and its rules
- `Add` before `go`, `defer Done()`, `Wait` in the waiting goroutine
- What a **deadlock** report looks like and how to read it
- Why a `WaitGroup` must **never be copied**, and how `go vet` helps
- Patterns: results by index, results under a mutex, first error, cancel-on-error, bounded concurrency
- `wg.Go` and `errgroup`
- Testing goroutine code: `-race`, ordering, timing, cancellation

---

## 2. The problem with `time.Sleep`

Four tasks take 60, 80, 120 and 140 ms. `main` must wait until they finish. Using a *guessed* sleep:

```go
package main

import (
	"fmt"
	"sort"
	"sync"
	"time"
)

// runTasks starts one goroutine per duration and returns the durations of the tasks that
// finished by the time main stops waiting.
func runTasks(durations []time.Duration, mainWaits time.Duration) []time.Duration {
	var mu sync.Mutex
	var finished []time.Duration

	for _, d := range durations {
		go func() {
			time.Sleep(d) // "work"
			mu.Lock()
			finished = append(finished, d)
			mu.Unlock()
		}()
	}

	time.Sleep(mainWaits) // the guess

	mu.Lock()
	defer mu.Unlock()
	out := append([]time.Duration(nil), finished...)
	sort.Slice(out, func(i, j int) bool { return out[i] < out[j] })
	return out
}

func main() {
	tasks := []time.Duration{60 * time.Millisecond, 80 * time.Millisecond, 120 * time.Millisecond, 140 * time.Millisecond}

	start := time.Now()
	got := runTasks(tasks, 100*time.Millisecond)
	fmt.Printf("guess 100ms: finished %v of %d tasks (%v)  waited %v\n", len(got), len(tasks), got, time.Since(start).Round(10*time.Millisecond))

	start = time.Now()
	got = runTasks(tasks, 1*time.Second)
	fmt.Printf("guess 1s:    finished %v of %d tasks (%v)  waited %v (the slowest task needed only 140ms)\n", len(got), len(tasks), got, time.Since(start).Round(10*time.Millisecond))
}
```

```
guess 100ms: finished 2 of 4 tasks ([60ms 80ms])  waited 100ms
guess 1s:    finished 4 of 4 tasks ([60ms 80ms 120ms 140ms])  waited 1s (the slowest task needed only 140ms)
```

Two failure modes of guessing:

- **Too short:** half the work is silently lost (the results are wrong, and *no error is reported*).
- **Too long:** everything is correct, but we waited **1000 ms for work that needed 140 ms**: 7× slower than necessary.

And the "right" duration doesn't exist: task times vary with load, data size, and network conditions. A sleep that works on your laptop fails in production. We need to wait for **completion**, not for **time**.

---

## 3. WaitGroup: a counter you can wait on

`sync.WaitGroup` is, at heart, **a counter plus the ability to block until the counter reaches zero**.

```
   teacher counting students on a school trip

   before the bus leaves:  "5 students"   →  counter = 5          (Add)
   each student boards:                       counter -1           (Done)
   the driver waits for:   counter == 0      →  bus departs        (Wait)
```

- **`Add(n)`** raises the counter by *n*: "n more tasks to wait for".
- **`Done()`** lowers it by one: "one task finished".
- **`Wait()`** blocks until the counter is zero.

Nothing about *which* goroutines or *what* they do: the WaitGroup only counts. That is why the discipline around `Add` and `Done` matters so much.

---

## 4. The API

| Method | Meaning | Rule |
|--------|---------|------|
| `wg.Add(n)` | counter += n | call **before** starting the goroutine, in the *launching* goroutine |
| `wg.Done()` | counter −= 1 (same as `Add(-1)`) | call **exactly once** per `Add(1)`, ideally with `defer` |
| `wg.Wait()` | block until counter == 0 | call from the goroutine that needs to wait (usually the launcher) |
| `wg.Go(f)` | `Add(1)`, run `f` in a new goroutine, `Done()` when it returns (**Go 1.25+**) | the simplest correct form |

The zero value is ready to use: `var wg sync.WaitGroup`. And the golden rule: **a WaitGroup must not be copied after first use**; always share it by pointer or by capturing it in a closure.

---

## 5. Basic usage

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	tasks := []time.Duration{60 * time.Millisecond, 80 * time.Millisecond, 120 * time.Millisecond, 140 * time.Millisecond}

	var wg sync.WaitGroup // the counter starts at 0
	results := make([]time.Duration, len(tasks))

	start := time.Now()
	for i, d := range tasks {
		wg.Add(1) // BEFORE starting the goroutine: counter is now one higher
		go func() {
			defer wg.Done() // counter goes down by one when this function returns (even on panic)
			time.Sleep(d)
			results[i] = d
		}()
	}
	wg.Wait() // blocks until the counter is back to 0

	fmt.Println("all finished:", results)
	fmt.Println("waited exactly as long as needed:", time.Since(start).Round(10*time.Millisecond))
}
```

```
all finished: [60ms 80ms 120ms 140ms]
waited exactly as long as needed: 140ms
```

All four results, and **140 ms** total (the slowest task), not 1,000 ms and not a lucky guess.

Walk through the counter: it starts at 0; the loop makes it 1, 2, 3, 4; as tasks finish it drops 3, 2, 1, 0; when it reaches 0, `Wait` returns. Memory visibility is also handled: everything a goroutine wrote *before* calling `Done` is guaranteed to be visible to the code after `Wait` returns (Go's memory model says `Done` "happens before" the return of `Wait`). That's why reading `results` after `Wait` is safe without a lock.

Three habits in the code above:

1. **`wg.Add(1)` before `go`**, in the launching goroutine.
2. **`defer wg.Done()` as the first line** of the goroutine, so *every* exit path (return, early return, panic) calls it.
3. Each goroutine writes **only to its own slot** of `results`.

---

## 6. The classic mistakes, with real output

### Mistake 1: `Add` inside the goroutine

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	var mu sync.Mutex
	ran := 0

	for i := 0; i < 5; i++ {
		go func() {
			wg.Add(1) // ❌ too late: the goroutine may not have started when Wait is called
			defer wg.Done()
			mu.Lock()
			ran++
			mu.Unlock()
		}()
	}
	wg.Wait() // the counter may still be 0 here, so Wait returns immediately

	mu.Lock()
	fmt.Println("goroutines that ran before Wait returned:", ran, "of 5")
	mu.Unlock()
}
```

```
goroutines that ran before Wait returned: 0 of 5
```

(We ran it five times; each time, 0 of 5.) `main` reached `wg.Wait()` while the counter was still 0 (the goroutines had been *created* but had not yet *executed* `Add`), so `Wait` returned instantly and `main` moved on. **No error, no warning**, just missing work. `Add` must happen in the *launching* goroutine, before the `go` statement.

### Mistake 2: forgetting `Done`: deadlock

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func() {
			fmt.Println("working on task", i)
			// ❌ forgot wg.Done()
		}()
	}

	wg.Wait() // waits for a counter that never reaches zero
	fmt.Println("never printed")
}
```

```
working on task 2
working on task 0
working on task 1
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
sync.runtime_SemacquireWaitGroup(0xc000012110?, 0x80?)
	/usr/local/go/src/runtime/sema.go:114 +0x2e
sync.(*WaitGroup).Wait(0xc0000120e0)
	/usr/local/go/src/sync/waitgroup.go:206 +0x85
main.main()
	/tmp/pgt/c65/forgotdone/main.go:19 +0x85
exit status 2
```

How to read a deadlock report:

- **`fatal error: all goroutines are asleep - deadlock!`**: the runtime noticed that *every* goroutine is blocked and none can ever wake another. This is a **fatal error**: it cannot be `recover`ed.
- **`goroutine 1 [sync.WaitGroup.Wait]`**: goroutine 1 (`main`) is blocked in `WaitGroup.Wait`.
- The frames below tell you *where*: `main.main()` at `main.go:19`, our `wg.Wait()` line. (The `runtime_SemacquireWaitGroup` frame at the top previews Chapter 66: `Wait` puts the goroutine to sleep on a runtime semaphore.)

Note the print order `2, 0, 1`: goroutines run in unpredictable order (Chapter 64). Also note that the runtime's deadlock detector only fires when *all* goroutines are stuck. In a real server, other goroutines (the HTTP listener, other requests) are alive, so a forgotten `Done` just makes one request **hang forever**, with no error at all. That is far harder to notice, and a good reason to always write `defer wg.Done()`.

### Mistake 3: too many `Done` calls: panic

```go
package main

import "sync"

func main() {
	var wg sync.WaitGroup
	wg.Add(1)
	wg.Done()
	wg.Done() // ❌ one Done too many: the counter would be -1
}
```

```
panic: sync: negative WaitGroup counter

goroutine 1 [running]:
sync.(*WaitGroup).Add(0xc0000120c0, 0xffffffffffffffff)
	/usr/local/go/src/sync/waitgroup.go:118 +0x23a
sync.(*WaitGroup).Done(...)
	/usr/local/go/src/sync/waitgroup.go:156
main.main()
```

The counter can never go below zero: `Done` (which is `Add(-1)`) panics. (Notice `Done` is literally `Add(-1)` in the stack trace.) Typical cause: `Done` called twice for one task, for instance both in a `defer` and at the end of the function.

### Mistake 4: copying the WaitGroup

```go
package main

import (
	"fmt"
	"sync"
)

// worker receives a COPY of the WaitGroup, so its Done() decrements the copy, not the original.
func worker(id int, wg sync.WaitGroup) {
	defer wg.Done()
	fmt.Println("worker", id)
}

func main() {
	var wg sync.WaitGroup
	wg.Add(1)
	go worker(1, wg) // ❌ passes the WaitGroup by value
	wg.Wait()
}
```

Two tools catch this. First, `go vet` (which also runs during `go test`):

```
./main.go:9:24: worker passes lock by value: sync.WaitGroup contains sync.noCopy
./main.go:17:15: call of worker copies lock value: sync.WaitGroup contains sync.noCopy
```

Second, if you ignore vet, running it prints `worker 1` and then the deadlock report from mistake 2: the worker decremented *its own copy*, and the original counter stayed at 1 forever. **Rule: pass `*sync.WaitGroup`** (or, better, keep the WaitGroup local and use closures).

```go
func worker(id int, wg *sync.WaitGroup) { defer wg.Done(); ... }
go worker(1, &wg)
```

### Mistake 5: `Done` not deferred

```go
go func() {
	result, err := fetch()
	if err != nil {
		return          // ❌ forgot Done on this path: Wait hangs forever
	}
	use(result)
	wg.Done()
}()
```

Every early `return` and every panic is a path that skips the trailing `Done`. `defer wg.Done()` at the top makes it impossible to forget.

### Mistake 6: reusing a WaitGroup too early

A `WaitGroup` may be reused for a new batch **only after `Wait` has returned** for the previous one. Calling `Add` with a positive count while a `Wait` from the *previous* batch is still waking up is a race the runtime detects and reports as `panic: sync: WaitGroup is reused before previous Wait has returned`. Simplest defense: create a new `WaitGroup` per batch.

### Summary of the rules

| Rule | Why |
|------|-----|
| `Add` **before** `go`, in the launcher | otherwise `Wait` can return early |
| `defer Done()` as the goroutine's first statement | every exit path counts down |
| **Exactly one** `Done` per `Add(1)` | fewer: hang; more: panic |
| Pass by **pointer** (or capture), never by value | copies have separate counters |
| New batch → new `WaitGroup` (or wait for the old `Wait` to return first) | avoid the reuse panic |

---

## 7. Collecting results and errors

A `WaitGroup` says *when* goroutines are done, not *what they produced*. Three safe ways to bring results back:

**A. Each goroutine writes its own slot** (as above). No synchronization needed beyond `Wait`, results keep input order, and it's the simplest. Requires knowing the count in advance.

```go
results := make([]Result, len(inputs))
for i, in := range inputs {
	wg.Add(1)
	go func() { defer wg.Done(); results[i] = process(in) }()
}
wg.Wait()
```

**B. Append under a mutex** when the number of results isn't known in advance or order doesn't matter (Chapter 68):

```go
var mu sync.Mutex
var out []Result
// in each goroutine:
mu.Lock(); out = append(out, r); mu.Unlock()
```

**C. Send on a channel** (Chapters 69-70):

```go
ch := make(chan Result, len(inputs))   // buffered: senders never block
// goroutines: ch <- r
wg.Wait(); close(ch)
for r := range ch { ... }
```

**Never** let goroutines `append` to a shared slice or write a shared variable *without* synchronization: that is a **data race** (Chapter 67).

**Errors** need a plan too. A goroutine can't `return` an error to its launcher. Options: store errors in a per-index slice (`errs[i] = err`) and inspect after `Wait` (with `errors.Join` to combine them, Chapter 48), send them on a channel, or use `errgroup` (section 9), which does exactly this and adds cancellation.

---

## 8. `wg.Go` (Go 1.25)

Since Go 1.25, `WaitGroup` has a `Go` method that performs the three-step dance in one call. It is the **preferred form** when your `go.mod` allows it (`go 1.25` or later):

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup
	results := make([]int, 5)

	for i := range results {
		wg.Go(func() { // Go 1.25+: Add(1), start the goroutine, and defer Done() in one call
			time.Sleep(50 * time.Millisecond)
			results[i] = i * i
		})
	}
	wg.Wait()

	fmt.Println(results)
}
```

```
[0 1 4 9 16]
```

`wg.Go(f)` makes mistakes 1, 2 (for this goroutine) and 5 impossible: `Add` is done before the goroutine starts, and `Done` is called when `f` returns, however it returns. Older code and older Go versions use the manual form; you should be fluent in both.

---

## 9. `errgroup`: errors, cancellation, limits

`golang.org/x/sync/errgroup` (an official, semi-standard package: `go get golang.org/x/sync`) is a `WaitGroup` for functions that **return errors**, with two extras:

- **`errgroup.WithContext`**: the *first* goroutine to return an error **cancels the shared context**, so siblings can stop early; `Wait` returns that first error.
- **`SetLimit(n)`**: at most *n* goroutines run at once (`g.Go` blocks when the limit is reached): built-in **bounded concurrency**.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"sync/atomic"
	"time"

	"golang.org/x/sync/errgroup"
)

// fetch pretends to call a service. Service 3 fails quickly; the others are slow.
func fetch(ctx context.Context, id int) error {
	if id == 3 {
		time.Sleep(50 * time.Millisecond)
		return errors.New("service 3 is down")
	}
	select {
	case <-time.After(500 * time.Millisecond): // a slow, healthy service
		return nil
	case <-ctx.Done(): // told to stop: another task failed
		return ctx.Err()
	}
}

func main() {
	// 1. First error cancels the rest.
	g, ctx := errgroup.WithContext(context.Background())
	var finishedOK atomic.Int32
	start := time.Now()
	for id := 1; id <= 5; id++ {
		g.Go(func() error {
			err := fetch(ctx, id)
			if err == nil {
				finishedOK.Add(1)
			}
			return err
		})
	}
	err := g.Wait() // waits for ALL goroutines, then returns the FIRST error
	fmt.Printf("1) Wait returned %q after %v; tasks that completed normally: %d\n",
		err, time.Since(start).Round(10*time.Millisecond), finishedOK.Load())

	// 2. A concurrency limit: at most 2 goroutines run at once.
	var running, peak atomic.Int32
	g2 := new(errgroup.Group)
	g2.SetLimit(2)
	start = time.Now()
	for i := 0; i < 6; i++ {
		g2.Go(func() error { // Go blocks when the limit is reached
			n := running.Add(1)
			for {
				old := peak.Load()
				if n <= old || peak.CompareAndSwap(old, n) {
					break
				}
			}
			time.Sleep(100 * time.Millisecond)
			running.Add(-1)
			return nil
		})
	}
	g2.Wait()
	fmt.Printf("2) 6 tasks of 100ms with limit 2: took %v, peak concurrency %d\n",
		time.Since(start).Round(10*time.Millisecond), peak.Load())
}
```

```
1) Wait returned "service 3 is down" after 50ms; tasks that completed normally: 0
2) 6 tasks of 100ms with limit 2: took 300ms, peak concurrency 2
```

Case 1: service 3 failed at 50 ms; the group's context was cancelled; the four healthy 500 ms calls **noticed and stopped** (`ctx.Done()`), so the whole operation ended after **50 ms instead of 500 ms**: none completed normally. Case 2: six 100 ms tasks with a limit of 2 took 300 ms (three waves of two) and the peak concurrency was exactly 2, as asked. (`atomic.Int32` is a race-free counter; Chapter 68.)

Cancellation is **cooperative**: goroutines stop early *only if they check the context* (`select` on `ctx.Done()`, or pass `ctx` to functions that honor it, like `db.QueryContext` and `http.NewRequestWithContext`). A goroutine that ignores its context keeps running until it finishes, and `Wait` will wait for it.

Use plain `WaitGroup` when tasks can't fail or you handle errors yourself; use `errgroup` for "run these, fail fast on the first error, maybe limit concurrency".

---

## 10. A complete example: `checker`

A small package that probes several URLs at the same time, using everything so far:

```go
// file: checker/checker.go
// Package checker probes several URLs at the same time.
package checker

import (
	"context"
	"net/http"
	"sync"
	"time"
)

// Result is what one probe found.
type Result struct {
	URL     string
	Status  int           // 0 if no response arrived
	Elapsed time.Duration // how long this probe took
	Err     error
}

// CheckAll requests every URL concurrently and returns the results IN THE SAME ORDER as urls.
//
// Each goroutine writes only to its own element of the results slice, so no locking is needed.
// wg.Wait() also guarantees that all those writes are visible to the caller when CheckAll returns.
func CheckAll(ctx context.Context, client *http.Client, urls []string) []Result {
	results := make([]Result, len(urls))

	var wg sync.WaitGroup
	for i, url := range urls {
		wg.Add(1) // before the goroutine starts: never inside it
		go func() {
			defer wg.Done() // runs even if the code below panics
			results[i] = check(ctx, client, url)
		}()
	}
	wg.Wait()

	return results
}

func check(ctx context.Context, client *http.Client, url string) Result {
	start := time.Now()
	res := Result{URL: url}

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		res.Err = err
		return res
	}
	resp, err := client.Do(req)
	res.Elapsed = time.Since(start)
	if err != nil {
		res.Err = err
		return res
	}
	resp.Body.Close()
	res.Status = resp.StatusCode
	return res
}
```

Design notes:

- **Results in input order**: the caller sees `results[i]` for `urls[i]` regardless of which probe finished first: much friendlier than completion order.
- A failed probe records its error in *its own* `Result` and does **not** affect the others (a slow or dead server is normal in this domain).
- The `context` flows into `http.NewRequestWithContext`, so cancelling it aborts the in-flight requests.
- **No mutex**: every write targets a distinct element; `Wait` provides the happens-before edge for the caller's reads.

The tests use `httptest` servers with controlled delays, and verify not just *correctness* but the *concurrency properties*:

```go
// file: checker/checker_test.go
package checker

import (
	"context"
	"net/http"
	"net/http/httptest"
	"sync/atomic"
	"testing"
	"time"
)

// slowServer answers with the given status after the given delay.
func slowServer(t *testing.T, delay time.Duration, status int) *httptest.Server {
	t.Helper()
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(delay)
		w.WriteHeader(status)
	}))
	t.Cleanup(srv.Close)
	return srv
}

func TestResultsComeBackInInputOrder(t *testing.T) {
	// The FIRST url is the slowest, so a naive "append as they finish" would put it last.
	slow := slowServer(t, 150*time.Millisecond, http.StatusOK)
	fast := slowServer(t, 10*time.Millisecond, http.StatusNotFound)
	mid := slowServer(t, 80*time.Millisecond, http.StatusAccepted)

	results := CheckAll(context.Background(), http.DefaultClient, []string{slow.URL, fast.URL, mid.URL})

	want := []int{http.StatusOK, http.StatusNotFound, http.StatusAccepted}
	for i, r := range results {
		if r.Status != want[i] || r.Err != nil {
			t.Errorf("result %d: status %d, err %v; want status %d", i, r.Status, r.Err, want[i])
		}
	}
}

func TestProbesRunConcurrently(t *testing.T) {
	var urls []string
	for i := 0; i < 5; i++ {
		urls = append(urls, slowServer(t, 200*time.Millisecond, http.StatusOK).URL)
	}

	start := time.Now()
	CheckAll(context.Background(), http.DefaultClient, urls)
	elapsed := time.Since(start)

	// Sequentially this would take 5 x 200ms = 1s. Concurrently: about 200ms.
	if elapsed > 600*time.Millisecond {
		t.Errorf("5 probes of 200ms took %v: they are not running concurrently", elapsed)
	}
}

func TestAFailingProbeDoesNotAffectTheOthers(t *testing.T) {
	good := slowServer(t, 0, http.StatusOK)
	dead := httptest.NewServer(http.NotFoundHandler())
	deadURL := dead.URL
	dead.Close() // nothing is listening any more

	results := CheckAll(context.Background(), http.DefaultClient, []string{good.URL, deadURL, good.URL})

	if results[0].Status != 200 || results[2].Status != 200 {
		t.Errorf("healthy probes must succeed: %+v", results)
	}
	if results[1].Err == nil || results[1].Status != 0 {
		t.Errorf("the dead server must produce an error and no status: %+v", results[1])
	}
}

func TestCancellationStopsWaiting(t *testing.T) {
	slow := slowServer(t, 500*time.Millisecond, http.StatusOK)
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()

	start := time.Now()
	results := CheckAll(ctx, http.DefaultClient, []string{slow.URL, slow.URL})

	if time.Since(start) > 300*time.Millisecond {
		t.Error("CheckAll must return soon after the context expires")
	}
	for _, r := range results {
		if r.Err == nil {
			t.Errorf("expected a context error, got %+v", r)
		}
	}
}

func TestEveryGoroutineIsAccountedFor(t *testing.T) {
	var hits atomic.Int64
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) { hits.Add(1) }))
	defer srv.Close()

	urls := make([]string, 200)
	for i := range urls {
		urls[i] = srv.URL
	}
	results := CheckAll(context.Background(), http.DefaultClient, urls)

	if hits.Load() != 200 || len(results) != 200 {
		t.Errorf("hits=%d results=%d, want 200 each", hits.Load(), len(results))
	}
	for i, r := range results {
		if r.Status != 200 {
			t.Fatalf("result %d incomplete: %+v (Wait returned before the goroutine finished?)", i, r)
		}
	}
}
```

Run with the race detector:

```bash
go vet ./... && go test -race -count=1 -v ./checker/
```
```
--- PASS: TestResultsComeBackInInputOrder (0.15s)
--- PASS: TestProbesRunConcurrently (0.20s)
--- PASS: TestAFailingProbeDoesNotAffectTheOthers (0.00s)
--- PASS: TestCancellationStopsWaiting (0.55s)
--- PASS: TestEveryGoroutineIsAccountedFor (0.04s)
ok  	checker	1.900s
```

(Notice `TestProbesRunConcurrently` takes **0.20 s** for five 200 ms probes.) Tests worth stealing as patterns: the **order test** deliberately makes the *first* URL the *slowest*; the **concurrency test** asserts a time bound that's only possible if tasks overlap (with generous slack, since timing tests can be flaky on loaded machines); the **completeness test** runs 200 probes and checks every slot: it would catch a `Wait` that returns early.

---

## 11. Revisiting the load generator

Chapter 62's `loadgen` used exactly this recipe. Read its core again with the new vocabulary:

```go
jobs := make(chan int)
var (
	mu  sync.Mutex
	res = result{status: map[int]int{}}
	wg  sync.WaitGroup
)

for w := 0; w < cfg.concurrency; w++ {
	wg.Add(1)                    // one worker more to wait for
	go func() {
		defer wg.Done()          // this worker is finished when the jobs channel is drained
		for i := range jobs {
			...send, measure...
			mu.Lock()            // results are shared: protect them (Chapter 68)
			res.latencies = append(res.latencies, took)
			mu.Unlock()
		}
	}()
}
for i := 1; i <= cfg.requests; i++ { jobs <- i }
close(jobs)                      // "no more jobs": the workers' range loops end (Chapter 69)
wg.Wait()                        // ...and only then do we read the results
```

Here the `WaitGroup` counts **workers** (a fixed pool of `concurrency` goroutines), not tasks: a common variant that bounds concurrency to the pool size while feeding it any number of jobs through a channel. Once you know channels and mutexes (the next chapters), this pattern (*worker pool*) is one you'll use constantly.

---

## 12. Testing concurrent code

Concurrency bugs are **timing-dependent**: a test may pass a thousand times and fail once. Habits that help:

- **Always run with `-race`** (`go test -race ./...`). It instruments memory accesses and reports data races even when the output happens to be correct (Chapter 67).
- **Repeat**: `go test -race -count=100 -run TestX ./pkg` shakes out flakiness.
- **Assert properties, not schedules**: "all results present", "in input order", "peak concurrency ≤ limit", never "goroutine A ran before B".
- **Make timing assertions generous** (2-3× slack) and use *big* delays relative to scheduling noise (100+ ms, not 1 ms).
- **Prefer deterministic synchronization** (channels, `WaitGroup`s, contexts) over `time.Sleep` in tests, for the same reason as in production code. (Go 1.25's `testing/synctest` package can run tests with a *fake clock* so sleeping tests finish instantly and deterministically; worth learning once you write a lot of concurrent code.)
- **Check for leaks**: a goroutine that never exits keeps memory and possibly connections alive. `runtime.NumGoroutine()` before and after a test can reveal it (or use a leak-detection helper such as `go.uber.org/goleak`).

`go vet` (run automatically by `go test`) catches copied WaitGroups and mutexes: keep it in CI.

---

## 13. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | `time.Sleep` to wait for goroutines | Lost work or wasted time | `WaitGroup` |
| 2 | `Add` inside the goroutine | `Wait` may return early | `Add` before `go` (or `wg.Go`) |
| 3 | Forgetting `Done` | Hang / deadlock | `defer wg.Done()` first |
| 4 | Extra `Done` | `panic: negative WaitGroup counter` | One `Done` per `Add(1)` |
| 5 | Passing `WaitGroup` by value | Separate counter; deadlock; vet warning | Pass `*sync.WaitGroup` or capture |
| 6 | Reusing a WaitGroup while an old `Wait` is waking | Reuse panic | New WaitGroup per batch |
| 7 | Writing shared variables from goroutines without sync | Data races | Own slots, mutex, channel (Chapters 67-69) |
| 8 | Ignoring errors in goroutines | Silent failures | `errgroup` or per-index errors |
| 9 | Unbounded goroutines for large inputs | Memory/connection exhaustion | Worker pool or `errgroup.SetLimit` |
| 10 | Goroutines that ignore `ctx` | Cancellation doesn't stop them; `Wait` hangs | `select` on `ctx.Done()` / context-aware calls |
| 11 | Assuming completion order = start order | Results out of order | Index-based results or sort |
| 12 | Calling `Wait` inside the goroutines being waited for | Self-deadlock | Wait from the launcher |
| 13 | Timing-dependent tests with tiny delays | Flaky tests | Large delays, generous bounds, properties not schedules |
| 14 | Not running `-race` | Races reach production | `go test -race` in CI |

---

## 14. Interview questions

**Q1. What does `sync.WaitGroup` do?**
Maintains a counter of outstanding tasks; `Add`/`Done` change it, `Wait` blocks until it reaches zero.

**Q2. Why must `Add` be called before starting the goroutine?**
If `Add` runs inside the goroutine, `Wait` might execute first with a zero counter and return early.

**Q3. Why use `defer wg.Done()`?**
It runs on every exit path, including early returns and panics, preventing hangs.

**Q4. What happens if the counter goes negative?**
`panic: sync: negative WaitGroup counter`.

**Q5. Why must a WaitGroup not be copied?**
The copy has its own counter, so `Done` on the copy never releases `Wait` on the original (deadlock); `go vet` flags it via `noCopy`.

**Q6. How do you get a value out of a goroutine?**
Write to a per-goroutine slot (slice element) read after `Wait`, protect a shared structure with a mutex, or send on a channel.

**Q7. What does `errgroup` add?**
Error propagation (first error), context cancellation on failure, and `SetLimit` for bounded concurrency.

**Q8. How does a deadlock report help?**
It shows which goroutines are blocked and on what (`goroutine 1 [sync.WaitGroup.Wait]`) with file and line numbers.

**Q9. Does `wg.Wait()` guarantee visibility of goroutine writes?**
Yes: `Done` happens-before the return of `Wait`, so writes made before `Done` are visible after `Wait`.

---

## 15. Exercises

### Exercise 1: Fix the bug
What's wrong here, and how would you fix it in two ways?

```go
var wg sync.WaitGroup
for _, url := range urls {
	go func() {
		wg.Add(1)
		defer wg.Done()
		fetch(url)
	}()
}
wg.Wait()
```

<details><summary>Solution</summary>

`Add` runs inside the goroutine, so `Wait` can return before any goroutine has registered. Fix 1: move `wg.Add(1)` before `go func()`. Fix 2 (Go 1.25+): `wg.Go(func() { fetch(url) })`.
</details>

### Exercise 2: Bounded `CheckAll`
Add a `limit int` parameter to `CheckAll` so at most `limit` probes run at once. Do it (a) with `errgroup.SetLimit` and (b) with a buffered channel used as a semaphore. Test that the peak concurrency is ≤ `limit`.

<details><summary>Solution</summary>

(b): `sem := make(chan struct{}, limit)`; in the loop `sem <- struct{}{}` **before** starting the goroutine (this blocks when `limit` are running), and `<-sem` in a `defer` inside the goroutine. Test with a server that counts in-flight requests and records the peak (like `TestConcurrencyIsBoundedByTheWorkerCount` in Chapter 62).
</details>

### Exercise 3: Reproduce each mistake
Write and run programs producing: (a) an early `Wait` return, (b) the deadlock trace, (c) the negative-counter panic, (d) the vet warning. Then explain each output line by line.

<details><summary>Solution</summary>

See section 6. In (b) the important line is `goroutine 1 [sync.WaitGroup.Wait]` plus the `main.main()` frame with your line number; in (c) the panic message names the invariant that was broken (`negative WaitGroup counter`).
</details>

### Exercise 4: First error wins
Rewrite `CheckAll` to return early when any probe returns a non-2xx status, cancelling the others (`errgroup.WithContext`). What must each probe do for cancellation to work?

<details><summary>Solution</summary>

Return an error from the `g.Go` function when the status isn't 2xx; pass the group's `ctx` to `http.NewRequestWithContext`. Without honoring `ctx`, the other probes keep running and `Wait` still waits for them.
</details>

### Exercise 5: Count the goroutines
Print `runtime.NumGoroutine()` before, during and after `CheckAll` on 100 URLs. Is it back to the starting value after `Wait`? (Hint: `http.DefaultClient` keeps idle connection goroutines around for a while.)

<details><summary>Solution</summary>

It returns close to the starting value but may be a few higher: the HTTP transport's connection-reading goroutines stay alive for idle keep-alive connections. Use `client.CloseIdleConnections()` (or a dedicated `http.Transport` with `DisableKeepAlives`) in tests that check leaks.
</details>

### Exercise 6 (challenge): A parallel `Map`
Write `func Map[T, R any](ctx context.Context, in []T, workers int, f func(context.Context, T) (R, error)) ([]R, error)` that applies `f` to every element with at most `workers` goroutines, returns results in input order, and stops on the first error. Test order, the worker limit, error propagation, and cancellation.

<details><summary>Solution</summary>

Use `errgroup.WithContext` + `SetLimit(workers)`; allocate `out := make([]R, len(in))` and have each goroutine write `out[i]`; return `out, g.Wait()`. Tests: order with reversed delays, peak concurrency ≤ workers, an error in element 3 cancels others (they observe `ctx.Err()`), an already-cancelled context does no work.
</details>

---

## 16. Quiz

1. What are the three core `WaitGroup` methods and what does each do to the counter?
2. Why must `Add` come before `go`?
3. What does `fatal error: all goroutines are asleep - deadlock!` tell you?
4. Why can't you pass a `sync.WaitGroup` by value?
5. How can goroutines return values safely?
6. What does `wg.Go(f)` do?
7. What does `errgroup.WithContext` do when one goroutine fails?
8. Why does `Wait` make it safe to read results afterwards?

<details><summary>Answers</summary>

1. `Add(n)` raises it, `Done()` lowers it by one, `Wait()` blocks until it is zero.
2. Otherwise `Wait` might run first, see zero, and return before the goroutine registered.
3. Every goroutine is blocked forever; the trace shows where each is stuck.
4. The copy has a separate counter, so `Done` on it never releases `Wait` on the original.
5. Own slice slots, a mutex-protected structure, or channels.
6. Adds one to the counter, starts `f` in a goroutine, and calls `Done` when `f` returns.
7. Cancels the shared context (so siblings can stop) and `Wait` returns the first error.
8. Go's memory model: `Done` happens-before `Wait` returns, so writes before `Done` are visible.
</details>

---

## 17. Summary

- **Sleeping guesses; `WaitGroup` waits for completion**: with four tasks the guess either lost half the work (100 ms) or wasted 7× the time (1 s); `WaitGroup` took exactly 140 ms.
- **Counter model:** `Add(n)` up, `Done()` down, `Wait()` blocks at zero. Rules: `Add` **before** `go`; `defer Done()` first; one `Done` per `Add`; share by **pointer**; new WaitGroup per batch.
- **Mistakes, observed:** `Add` inside the goroutine → `Wait` returned with 0 of 5 done; missing `Done` → *fatal deadlock* trace; extra `Done` → `panic: negative WaitGroup counter`; copied WaitGroup → vet warning + deadlock.
- **Results and errors:** write to your **own slot**, append under a mutex, or send on a channel; use **`wg.Go`** (Go 1.25+) or **`errgroup`** (first error, cancel-on-failure, `SetLimit`): a failing service ended a 500 ms operation after 50 ms.
- `checker.CheckAll` shows the pattern end to end (ordered results, isolated failures, cancellation), tested for **properties** with `-race`.
- Concurrency bugs are timing-dependent: test properties, use generous bounds, run `-race` and `-count=N`.

### ➡️ What's next?

[Chapter 66](66-inside-sync-waitgroup.md) looks **inside** `WaitGroup`: the 64-bit state word, why `Add` and `Wait` are lock-free, how the runtime **semaphore** puts waiting goroutines to sleep, and what the panics you just triggered are really checking. We'll build a working mini-`WaitGroup` to see the design.
