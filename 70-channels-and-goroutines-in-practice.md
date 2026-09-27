# Chapter 70: Channels and Goroutines in Practice — A Concurrent Product List (and Course Wrap-Up)

> **Goal of this chapter:** Apply everything from Chapters 64-69 to the real service. The paginated `GET /products` (Chapter 63) runs `Count` and then `List`, one after the other, although neither needs the other's result. You'll run them **concurrently** the safe way: **one result and one error variable per goroutine** (never the shared `err` of Chapter 67), a `WaitGroup` to join, a **cancellable context** so that the failure of one query stops the other, and a `rootCause` rule so our own cancellation never masks the real error. Then you'll test it properly: proof that the queries *overlap* (peak concurrency 2), that the root cause is reported, that a **mutant** using a shared `err` is caught by both the race detector and a failing assertion, that no goroutine leaks, and that the test double itself must be goroutine-safe. Finally an **honest measurement on 500,000 products** (what concurrency bought us, and what it did not), and a wrap-up of the whole course.

**Difficulty:** 🔴 Advanced  **Estimated time:** 8 hours  **Prerequisite:** [Chapters 63 and 65-69](69-channels.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The target](#2-the-target)
3. [Design: what to share, what not to share](#3-design-what-to-share-what-not-to-share)
4. [The implementation](#4-the-implementation)
5. [Walking through it](#5-walking-through-it)
6. [The test double must be goroutine-safe too](#6-the-test-double-must-be-goroutine-safe-too)
7. [Tests](#7-tests)
8. [The mutant: proving the tests can fail](#8-the-mutant-proving-the-tests-can-fail)
9. [What it bought us: measurements](#9-what-it-bought-us-measurements)
10. [Channels or `WaitGroup` here?](#10-channels-or-waitgroup-here)
11. [Common mistakes](#11-common-mistakes)
12. [Interview questions](#12-interview-questions)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)
16. [Course wrap-up](#16-course-wrap-up)

---

## 1. What you will learn

- Choosing **what** to run concurrently (independent I/O) and what to leave alone
- Structuring concurrent code so that **no memory is shared**: per-goroutine result and error variables
- **Cancellation** with `context.WithCancel`, and reporting the **root cause** of a failure
- Testing concurrent code: overlap proofs, error propagation, cancellation, leaks, `-race`
- **Mutation testing** by hand: break the code on purpose and confirm a test fails
- Making **fakes** goroutine-safe
- Reading benchmark results honestly: what concurrency inside a request can and cannot do under load

---

## 2. The target

Today's `product.Service.List` (Chapter 63):

```go
total, err := s.repo.Count(ctx)          // ~20 ms on 500,000 rows
...
items, err = s.repo.List(ctx, limit, offset)   // ~0.4 ms on page 1, ~25 ms on a deep page
```

The two calls are **independent**: `List` doesn't need the total, `Count` doesn't need the rows. Sequentially the request waits `count + list`; concurrently it can wait `max(count, list)`. (Chapter 63 skipped the list query when the page was past the end, an optimization that *depends* on the count, so it can't survive here: we trade it for the overlap, and a page past the end simply returns an empty list from a very cheap query.)

This is precisely the situation of Chapter 64: **two independent waits inside one request**.

---

## 3. Design: what to share, what not to share

Follow the strategy list from Chapter 67, section 13:

| Data | Written by | Strategy |
|------|-----------|----------|
| `total` | the `Count` goroutine only | its **own** variable |
| `items` | the `List` goroutine only | its **own** variable |
| `countErr` | the `Count` goroutine only | its **own** variable |
| `listErr` | the `List` goroutine only | its **own** variable |
| all of the above, read by the caller | after `wg.Wait()` | `Wait` provides the happens-before edge (Chapter 65) |
| "stop the other query" | either goroutine | `cancel()` from `context.WithCancel` (a channel close underneath: broadcast) |

No mutex is needed: **nothing is written by two goroutines**. That is the cheapest and safest synchronization there is.

The failure story needs a decision: if `Count` fails, `List` is pointless (and vice versa), so the first failure **cancels** the other. But the cancelled goroutine will itself return `context canceled`, and we must report the **root cause** (the database timeout) rather than our own consequence. So: skip a `context.Canceled` error that *we* caused, unless the **caller's** context was the one cancelled (then it *is* the cause).

---

## 4. The implementation

```go
// file: product/service.go
package product

import (
	"context"
	"errors"
	"fmt"
	"sync"
)

// Service implements the catalog use cases on top of the Repository port.
type Service struct {
	repo Repository
}

// NewService creates a Service.
func NewService(repo Repository) *Service { return &Service{repo: repo} }

// List returns one page of the catalog together with the total number of products.
//
// The count and the page are independent queries, so they run at the same time. Each goroutine
// writes only its OWN variables (nothing is shared), and wg.Wait() makes those writes visible here.
// If either query fails, the other is cancelled: there is no point finishing half an answer.
func (s *Service) List(parent context.Context, req PageRequest) (Products, error) {
	ctx, cancel := context.WithCancel(parent)
	defer cancel()

	var (
		total    int
		items    []Product
		countErr error // one error variable PER goroutine, never one shared err (Chapter 67, section 9)
		listErr  error
		wg       sync.WaitGroup
	)

	wg.Add(2)
	go func() {
		defer wg.Done()
		total, countErr = s.repo.Count(ctx)
		if countErr != nil {
			cancel() // tell the other query to stop
		}
	}()
	go func() {
		defer wg.Done()
		items, listErr = s.repo.List(ctx, req.Limit(), req.Offset())
		if listErr != nil {
			cancel()
		}
	}()
	wg.Wait()

	if err := rootCause(parent, wrapIf("count products", countErr), wrapIf("list products", listErr)); err != nil {
		return Products{}, err
	}
	if items == nil {
		items = []Product{} // never nil
	}
	return Products{Items: items, Request: req, Total: total}, nil
}

// wrapIf adds context to a non-nil error.
func wrapIf(what string, err error) error {
	if err == nil {
		return nil
	}
	return fmt.Errorf("%s: %w", what, err)
}

// rootCause picks the error to report when goroutines that were cancelled by OUR cancel() also
// returned errors. A "context canceled" that we caused is a consequence, not the cause, so it is
// skipped, unless the caller's own context was cancelled (then it IS the cause).
func rootCause(parent context.Context, errs ...error) error {
	var fallback error
	for _, err := range errs {
		if err == nil {
			continue
		}
		if errors.Is(err, context.Canceled) && parent.Err() == nil {
			fallback = err
			continue
		}
		return err
	}
	return fallback
}

// Get returns the product with the given ID, or ErrNotFound.
func (s *Service) Get(ctx context.Context, id int) (Product, error) {
	p, err := s.repo.Get(ctx, id)
	return p, wrap("get product", err)
}

// Create validates d and stores it as a new product.
func (s *Service) Create(ctx context.Context, d Details) (Product, error) {
	d, err := d.normalized()
	if err != nil {
		return Product{}, err
	}
	p, err := s.repo.Create(ctx, d.toProduct(0))
	return p, wrap("create product", err)
}

// Update validates d and replaces the product with the given ID. It returns ErrNotFound if there is none.
// (Validation happens first: a malformed request is the client's mistake whether or not the ID exists.)
func (s *Service) Update(ctx context.Context, id int, d Details) (Product, error) {
	d, err := d.normalized()
	if err != nil {
		return Product{}, err
	}
	p, err := s.repo.Update(ctx, id, d.toProduct(id))
	return p, wrap("update product", err)
}

// Delete removes the product with the given ID, or returns ErrNotFound.
func (s *Service) Delete(ctx context.Context, id int) error {
	return wrap("delete product", s.repo.Delete(ctx, id))
}

// wrap adds context to unexpected failures but passes the expected "not found" outcome through untouched.
func wrap(what string, err error) error {
	if err == nil || errors.Is(err, ErrNotFound) {
		return err
	}
	return fmt.Errorf("%s: %w", what, err)
}
```

---

## 5. Walking through it

1. **`ctx, cancel := context.WithCancel(parent)`** derives a context we can cancel ourselves; `defer cancel()` releases its resources however `List` exits (calling `cancel` twice is harmless).
2. **`wg.Add(2)` before starting either goroutine**, in the launcher (Chapter 65's first rule).
3. Each goroutine writes **only its own variables** and calls `cancel()` **only on failure**. A successful `Count` must *not* cancel: the list is still running.
4. **`wg.Wait()`** joins both. From here on we are the only goroutine touching these variables, and `Wait` guarantees we see their writes.
5. **`rootCause(parent, …)`**. Cases:
   - both succeeded → `nil`;
   - `Count` failed (database timeout) and cancelled the list, which returned `context canceled` → the count error is returned, the consequence is skipped;
   - the **caller** gave up (`parent.Err() != nil`), so both returned a context error → `Canceled`/`DeadlineExceeded` **is** the cause and is reported (so the HTTP layer can tell a client timeout from a database failure);
   - both failed genuinely → the first non-cancellation error (count before list) wins; the other is dropped (a possible refinement is `errors.Join`; Exercise 3).
6. `items == nil` → `[]Product{}` keeps the "never nil" promise even if an adapter forgot.

Notice what is *absent*: no `sync.Mutex`, no channels written by hand, no shared `err`. The only synchronization primitives are `WaitGroup` (join) and `context` (cancel), and each variable has exactly one writer.

---

## 6. The test double must be goroutine-safe too

Chapter 63's fake repository recorded calls in plain fields (`calls`, `listed`). Once `List` runs `Count` and `List` in two goroutines, **the fake itself is called concurrently**, and the race detector says so immediately:

```
==================
WARNING: DATA RACE
Read at 0x00c0000de620 by goroutine 24:
  ecommerce/product.(*fakeRepo).touch()
      product/service_test.go:20 +0x58
  ecommerce/product.(*fakeRepo).Count()
      product/service_test.go:33 +0x12
  ecommerce/product.(*Service).List.func1()
      product/service.go:38 +0xe6

Previous write at 0x00c0000de620 by goroutine 25:
  ecommerce/product.(*fakeRepo).touch()
      product/service_test.go:20 +0x84
  ecommerce/product.(*fakeRepo).List()
      product/service_test.go:23 +0x1a
  ecommerce/product.(*Service).List.func2()
      product/service.go:45 +0x141
```

The race was in the *test code*, not in the service, but it fails the build all the same. **A fake must obey the same concurrency rules as the real thing.** The fixed fake guards every field with a mutex, and gains knobs for the concurrency tests: per-query **delays** that stop early when the context is cancelled (like a real database driver does), per-query **errors**, a `listIgnoresCtx` switch that simulates a driver that never notices cancellation, and counters for the **peak number of overlapping queries**:

```go
// file: product/service_test.go
package product

import (
	"context"
	"errors"
	"math"
	"strings"
	"sync"
	"testing"
	"time"
)

// fakeRepo is an in-memory Repository that records what the service asked it to store.
// It is used from several goroutines at once (List runs Count and List concurrently),
// so every field is guarded by mu: a test double must obey the same rules as real code.
type fakeRepo struct {
	mu     sync.Mutex
	items  []Product
	nextID int
	err    error    // if set, every call fails with it
	calls  int      // how many times the repository was touched
	listed [][2]int // the (limit, offset) of every List call

	// knobs for the concurrency tests
	countDelay, listDelay time.Duration // how long each query "takes" (it stops early if ctx is cancelled)
	listIgnoresCtx        bool          // a driver that keeps going even after cancellation
	countErr, listErr     error         // errors for one query only
	inFlight, peak        int           // queries running right now / the most that ever ran at once
}

func (f *fakeRepo) touch() error {
	f.mu.Lock()
	defer f.mu.Unlock()
	f.calls++
	return f.err
}

// begin/end track how many queries overlap.
func (f *fakeRepo) begin() {
	f.mu.Lock()
	f.inFlight++
	f.peak = max(f.peak, f.inFlight)
	f.mu.Unlock()
}

func (f *fakeRepo) end() {
	f.mu.Lock()
	f.inFlight--
	f.mu.Unlock()
}

// wait pretends to work for d, but gives up as soon as ctx is cancelled (like a real database driver).
func wait(ctx context.Context, d time.Duration) error {
	if d == 0 {
		return ctx.Err()
	}
	select {
	case <-time.After(d):
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (f *fakeRepo) List(ctx context.Context, limit, offset int) ([]Product, error) {
	f.begin()
	defer f.end()
	if err := f.touch(); err != nil {
		return nil, err
	}
	listCtx := ctx
	if f.listIgnoresCtx {
		listCtx = context.Background() // this "driver" never notices the cancellation
	}
	if err := wait(listCtx, f.listDelay); err != nil {
		return nil, err
	}

	f.mu.Lock()
	defer f.mu.Unlock()
	if f.listErr != nil {
		return nil, f.listErr
	}
	f.listed = append(f.listed, [2]int{limit, offset}) // remember what was asked for
	if offset >= len(f.items) {
		return []Product{}, nil
	}
	return append([]Product(nil), f.items[offset:min(offset+limit, len(f.items))]...), nil
}

func (f *fakeRepo) Count(ctx context.Context) (int, error) {
	f.begin()
	defer f.end()
	if err := f.touch(); err != nil {
		return 0, err
	}
	if err := wait(ctx, f.countDelay); err != nil {
		return 0, err
	}

	f.mu.Lock()
	defer f.mu.Unlock()
	if f.countErr != nil {
		return 0, f.countErr
	}
	return len(f.items), nil
}

func (f *fakeRepo) Get(_ context.Context, id int) (Product, error) {
	if err := f.touch(); err != nil {
		return Product{}, err
	}
	f.mu.Lock()
	defer f.mu.Unlock()
	for _, p := range f.items {
		if p.ID == id {
			return p, nil
		}
	}
	return Product{}, ErrNotFound
}

func (f *fakeRepo) Create(_ context.Context, p Product) (Product, error) {
	if err := f.touch(); err != nil {
		return Product{}, err
	}
	f.mu.Lock()
	defer f.mu.Unlock()
	f.nextID++
	p.ID = f.nextID
	f.items = append(f.items, p)
	return p, nil
}

func (f *fakeRepo) Update(_ context.Context, id int, p Product) (Product, error) {
	if err := f.touch(); err != nil {
		return Product{}, err
	}
	f.mu.Lock()
	defer f.mu.Unlock()
	for i := range f.items {
		if f.items[i].ID == id {
			p.ID = id
			f.items[i] = p
			return p, nil
		}
	}
	return Product{}, ErrNotFound
}

func (f *fakeRepo) Delete(_ context.Context, id int) error {
	if err := f.touch(); err != nil {
		return err
	}
	f.mu.Lock()
	defer f.mu.Unlock()
	for i := range f.items {
		if f.items[i].ID == id {
			f.items = append(f.items[:i], f.items[i+1:]...)
			return nil
		}
	}
	return ErrNotFound
}

var ctx = context.Background()

func newTestService() (*Service, *fakeRepo) {
	repo := &fakeRepo{}
	return NewService(repo), repo
}

func good() Details {
	return Details{Title: "Mango", Description: "Sweet", Price: 75.5, ImageURL: "https://example.com/mango.jpg"}
}

func TestCreateStoresTheCleanedUpProduct(t *testing.T) {
	svc, repo := newTestService()

	d := good()
	d.Title = "   Mango  " // surrounding spaces are noise
	p, err := svc.Create(ctx, d)
	if err != nil {
		t.Fatal(err)
	}
	if p.ID != 1 || p.Title != "Mango" || p.Price != 75.5 {
		t.Errorf("unexpected product %+v", p)
	}
	if len(repo.items) != 1 || repo.items[0].Title != "Mango" {
		t.Errorf("the repository must receive the cleaned-up title, got %+v", repo.items)
	}
}

func TestTheSameRulesApplyToCreateAndUpdate(t *testing.T) {
	tests := []struct {
		name    string
		mutate  func(*Details)
		message string
	}{
		{"empty title", func(d *Details) { d.Title = "" }, "title is required"},
		{"blank title", func(d *Details) { d.Title = "   \t " }, "title is required"},
		{"title too long", func(d *Details) { d.Title = strings.Repeat("x", 101) }, "at most 100 characters"},
		{"zero price", func(d *Details) { d.Price = 0 }, "price must be greater than 0"},
		{"negative price", func(d *Details) { d.Price = -1 }, "price must be greater than 0"},
		{"NaN price", func(d *Details) { d.Price = math.NaN() }, "price must be greater than 0"},
		{"infinite price", func(d *Details) { d.Price = math.Inf(1) }, "price must be greater than 0"},
	}
	for _, tc := range tests {
		for _, op := range []string{"create", "update"} {
			t.Run(tc.name+"/"+op, func(t *testing.T) {
				svc, repo := newTestService()
				repo.Create(ctx, Product{Title: "existing", Price: 1})
				repo.calls = 0

				d := good()
				tc.mutate(&d)
				var err error
				if op == "create" {
					_, err = svc.Create(ctx, d)
				} else {
					_, err = svc.Update(ctx, 1, d)
				}

				var ve *ValidationError
				if !errors.As(err, &ve) || !strings.Contains(ve.Message, tc.message) {
					t.Fatalf("expected a ValidationError containing %q, got %v", tc.message, err)
				}
				if repo.calls != 0 {
					t.Error("an invalid request must never reach the repository")
				}
			})
		}
	}
}

func TestTitleLengthCountsCharactersNotBytes(t *testing.T) {
	svc, _ := newTestService()

	bengali := strings.Repeat("বা", 50) // 100 characters, 300 bytes
	d := good()
	d.Title = bengali
	if _, err := svc.Create(ctx, d); err != nil {
		t.Errorf("100 characters must be accepted even though they take 300 bytes: %v", err)
	}

	d.Title = bengali + "x" // 101 characters
	if _, err := svc.Create(ctx, d); err == nil {
		t.Error("101 characters must be rejected")
	}
}

func TestBoundaries(t *testing.T) {
	svc, _ := newTestService()
	for title, ok := range map[string]bool{
		strings.Repeat("a", 1):   true,
		strings.Repeat("a", 100): true,
		strings.Repeat("a", 101): false,
	} {
		d := good()
		d.Title = title
		if _, err := svc.Create(ctx, d); (err == nil) != ok {
			t.Errorf("title of %d characters: accepted=%v, want %v", len(title), err == nil, ok)
		}
	}
	d := good()
	d.Price = 0.01 // the smallest price the table's NUMERIC(12,2) can hold
	if _, err := svc.Create(ctx, d); err != nil {
		t.Errorf("0.01 must be a valid price: %v", err)
	}
}

func TestUpdateReplacesAndKeepsTheID(t *testing.T) {
	svc, repo := newTestService()
	created, _ := svc.Create(ctx, good())

	p, err := svc.Update(ctx, created.ID, Details{Title: "Alphonso", Price: 120})
	if err != nil {
		t.Fatal(err)
	}
	if p.ID != created.ID || p.Title != "Alphonso" || p.Description != "" {
		t.Errorf("a PUT replaces every field (unspecified ones become empty): %+v", p)
	}
	if len(repo.items) != 1 {
		t.Errorf("update must not create products, got %d", len(repo.items))
	}
}

func TestMissingProductsAreErrNotFound(t *testing.T) {
	svc, _ := newTestService()

	if _, err := svc.Get(ctx, 9); !errors.Is(err, ErrNotFound) {
		t.Errorf("Get: %v", err)
	}
	if _, err := svc.Update(ctx, 9, good()); !errors.Is(err, ErrNotFound) {
		t.Errorf("Update: %v", err)
	}
	if err := svc.Delete(ctx, 9); !errors.Is(err, ErrNotFound) {
		t.Errorf("Delete: %v", err)
	}
}

func TestDeleteRemovesOnlyThatProduct(t *testing.T) {
	svc, _ := newTestService()
	a, _ := svc.Create(ctx, good())
	b, _ := svc.Create(ctx, good())

	if err := svc.Delete(ctx, a.ID); err != nil {
		t.Fatal(err)
	}
	page, _ := svc.List(ctx, FirstPage())
	if list := page.Items; len(list) != 1 || list[0].ID != b.ID {
		t.Errorf("got %+v", page.Items)
	}
}

func TestStorageFailuresAreWrappedNotDisguised(t *testing.T) {
	boom := errors.New("connection reset")
	svc, repo := newTestService()
	repo.err = boom

	calls := map[string]func() error{
		"list":   func() error { _, err := svc.List(ctx, FirstPage()); return err },
		"get":    func() error { _, err := svc.Get(ctx, 1); return err },
		"create": func() error { _, err := svc.Create(ctx, good()); return err },
		"update": func() error { _, err := svc.Update(ctx, 1, good()); return err },
		"delete": func() error { return svc.Delete(ctx, 1) },
	}
	for name, call := range calls {
		err := call()
		if !errors.Is(err, boom) {
			t.Errorf("%s: the cause must stay reachable with errors.Is, got %v", name, err)
		}
		if errors.Is(err, ErrNotFound) {
			t.Errorf("%s: a storage failure must never look like ErrNotFound", name)
		}
	}
}
```

The paging tests adapt to the new behavior (the "skips the query" test becomes "a page past the end is simply empty"):

```go
// file: product/service_list_test.go
package product

import (
	"errors"
	"testing"
)

// seed puts n products in the fake repository.
func seed(repo *fakeRepo, n int) {
	for i := 1; i <= n; i++ {
		repo.Create(ctx, Product{Title: "P", Price: 1})
	}
	repo.calls = 0
	repo.listed = nil
}

func request(t *testing.T, page, limit int) PageRequest {
	t.Helper()
	r, err := NewPageRequest(page, limit)
	if err != nil {
		t.Fatal(err)
	}
	return r
}

func TestListReturnsThePageAndTheTotals(t *testing.T) {
	svc, repo := newTestService()
	seed(repo, 45)

	page, err := svc.List(ctx, request(t, 2, 20))
	if err != nil {
		t.Fatal(err)
	}
	if len(page.Items) != 20 || page.Items[0].ID != 21 || page.Items[19].ID != 40 {
		t.Errorf("page 2 should be products 21-40, got %d items starting at ID %d", len(page.Items), page.Items[0].ID)
	}
	if page.Total != 45 || page.TotalPages() != 3 {
		t.Errorf("total=%d pages=%d, want 45 and 3", page.Total, page.TotalPages())
	}
	repo.mu.Lock()
	defer repo.mu.Unlock()
	if len(repo.listed) != 1 || repo.listed[0] != [2]int{20, 20} {
		t.Errorf("the repository must be asked for (limit 20, offset 20), got %v", repo.listed)
	}
}

func TestTheLastPageMayBeShort(t *testing.T) {
	svc, repo := newTestService()
	seed(repo, 45)

	page, _ := svc.List(ctx, request(t, 3, 20))
	if len(page.Items) != 5 {
		t.Errorf("the last page should hold the remaining 5 products, got %d", len(page.Items))
	}
}

func TestAPagePastTheEndIsSimplyEmpty(t *testing.T) {
	svc, repo := newTestService()
	seed(repo, 45)

	page, err := svc.List(ctx, request(t, 99, 20))
	if err != nil {
		t.Fatal(err)
	}
	if page.Items == nil || len(page.Items) != 0 {
		t.Errorf("want an empty non-nil slice, got %#v", page.Items)
	}
	if page.Total != 45 {
		t.Errorf("the total must still be reported, got %d", page.Total)
	}
}

func TestAnEmptyCatalogIsAValidResult(t *testing.T) {
	svc, _ := newTestService()
	page, err := svc.List(ctx, FirstPage())
	if err != nil || page.Items == nil || len(page.Items) != 0 || page.Total != 0 || page.TotalPages() != 0 {
		t.Errorf("got %+v, %v", page, err)
	}
}

func TestListFailuresAreReported(t *testing.T) {
	boom := errors.New("connection reset")
	svc, repo := newTestService()
	repo.err = boom

	if _, err := svc.List(ctx, FirstPage()); !errors.Is(err, boom) {
		t.Errorf("got %v", err)
	}
}
```

---

## 7. Tests

```go
// file: product/service_concurrent_test.go
package product

import (
	"context"
	"errors"
	"runtime"
	"testing"
	"time"
)

func TestCountAndListRunAtTheSameTime(t *testing.T) {
	svc, repo := newTestService()
	seed(repo, 30)
	repo.countDelay = 150 * time.Millisecond
	repo.listDelay = 150 * time.Millisecond

	start := time.Now()
	page, err := svc.List(ctx, FirstPage())
	elapsed := time.Since(start)
	if err != nil {
		t.Fatal(err)
	}

	if page.Total != 30 || len(page.Items) != 20 {
		t.Errorf("wrong answer: total=%d items=%d", page.Total, len(page.Items))
	}
	// Sequentially: 150ms + 150ms = 300ms. Overlapped: about 150ms.
	if elapsed > 250*time.Millisecond {
		t.Errorf("took %v: the two queries did not overlap", elapsed)
	}
	repo.mu.Lock()
	defer repo.mu.Unlock()
	if repo.peak != 2 {
		t.Errorf("peak concurrent queries = %d, want 2 (proof that they overlapped)", repo.peak)
	}
}

func TestACountFailureIsReportedAndCancelsTheListQuery(t *testing.T) {
	boom := errors.New("count: database timeout")
	svc, repo := newTestService()
	seed(repo, 30)
	repo.countErr = boom
	repo.listDelay = 2 * time.Second // a slow list query that must be abandoned

	start := time.Now()
	_, err := svc.List(ctx, FirstPage())
	elapsed := time.Since(start)

	if !errors.Is(err, boom) {
		t.Fatalf("the ROOT cause must be reported, got %v", err)
	}
	if errors.Is(err, context.Canceled) {
		t.Errorf("our own cancellation of the other query must not mask the real error: %v", err)
	}
	if elapsed > 500*time.Millisecond {
		t.Errorf("took %v: the slow list query was not cancelled", elapsed)
	}
}

func TestAListFailureIsReportedAndCancelsTheCountQuery(t *testing.T) {
	boom := errors.New("list: connection reset")
	svc, repo := newTestService()
	seed(repo, 30)
	repo.listErr = boom
	repo.countDelay = 2 * time.Second

	start := time.Now()
	_, err := svc.List(ctx, FirstPage())

	if !errors.Is(err, boom) || errors.Is(err, context.Canceled) {
		t.Errorf("got %v, want the list error", err)
	}
	if time.Since(start) > 500*time.Millisecond {
		t.Error("the slow count query was not cancelled")
	}
}

func TestNeitherErrorIsEverLost(t *testing.T) {
	// The shared-err bug of Chapter 67 would sometimes turn this into a success.
	svc, repo := newTestService()
	seed(repo, 30)
	repo.countErr = errors.New("count failed")
	repo.listDelay = 5 * time.Millisecond // the list finishes later...
	repo.listIgnoresCtx = true            // ...and never notices the cancellation, so it completes successfully

	for i := 0; i < 200; i++ {
		if _, err := svc.List(ctx, FirstPage()); err == nil {
			t.Fatalf("iteration %d: a failed count was silently lost", i)
		}
	}
}

func TestACancelledCallerContextIsReportedAsCancellation(t *testing.T) {
	svc, repo := newTestService()
	seed(repo, 30)
	repo.countDelay = time.Second
	repo.listDelay = time.Second

	callerCtx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()

	start := time.Now()
	_, err := svc.List(callerCtx, FirstPage())

	if !errors.Is(err, context.DeadlineExceeded) {
		t.Errorf("the caller's own deadline IS the cause and must be reported, got %v", err)
	}
	if time.Since(start) > 500*time.Millisecond {
		t.Error("List must return soon after the caller gives up")
	}
}

func TestNoGoroutinesAreLeaked(t *testing.T) {
	svc, repo := newTestService()
	seed(repo, 30)
	repo.countDelay = time.Millisecond
	repo.listErr = errors.New("boom") // exercise the cancellation path too

	before := runtime.NumGoroutine()
	for i := 0; i < 300; i++ {
		svc.List(ctx, FirstPage())
	}

	// Give finished goroutines a moment to be reaped, then compare.
	deadline := time.Now().Add(2 * time.Second)
	for runtime.NumGoroutine() > before && time.Now().Before(deadline) {
		time.Sleep(10 * time.Millisecond)
	}
	if after := runtime.NumGoroutine(); after > before {
		t.Errorf("goroutines before=%d after=%d: List leaked", before, after)
	}
}

func TestManyConcurrentListCalls(t *testing.T) {
	// Several callers at once, each running two goroutines: the race detector is the judge.
	svc, repo := newTestService()
	seed(repo, 100)

	done := make(chan struct{})
	for i := 0; i < 50; i++ {
		go func() {
			defer func() { done <- struct{}{} }()
			page, err := svc.List(ctx, request(t, i%5+1, 20))
			if err != nil || page.Total != 100 || len(page.Items) != 20 {
				t.Errorf("caller %d: %+v, %v", i, page, err)
			}
		}()
	}
	for i := 0; i < 50; i++ {
		<-done
	}
}
```

What each test proves:

| Test | Property |
|------|----------|
| `TestCountAndListRunAtTheSameTime` | two 150 ms queries finish in ~150 ms, not 300 ms, **and** the fake saw `peak == 2` (a timing bound alone could pass by luck; the overlap counter can't) |
| `TestACountFailureIsReportedAndCancelsTheListQuery` | the **root cause** is reported (not `context canceled`) and the 2-second list query is abandoned within 500 ms |
| `TestAListFailureIsReportedAndCancelsTheCountQuery` | the same, symmetric |
| `TestNeitherErrorIsEverLost` | 200 iterations with a failing count and a list that *completes successfully*: the failure must never turn into a success (the Chapter 67 bug) |
| `TestACancelledCallerContextIsReportedAsCancellation` | if the **caller** gives up, the answer is `DeadlineExceeded`, and `List` returns promptly |
| `TestNoGoroutinesAreLeaked` | 300 calls including the failure/cancellation path: the goroutine count returns to its baseline |
| `TestManyConcurrentListCalls` | 50 callers × 2 goroutines each under `-race` |

Run them the way concurrent tests deserve, repeated and with the race detector:

```bash
go vet ./... && go test -race -count=5 ./product/
```
```
ok  	ecommerce/product	4.928s
```

---

## 8. The mutant: proving the tests can fail

A test suite that has never failed hasn't proved it *can*. **Mutation testing** deliberately breaks the code and checks that some test notices. Replace the two error variables in `List` by **one shared `err`** (the bug from Chapter 67):

```go
var err error                       // ONE shared variable (the bug)
go func() { total, err = s.repo.Count(ctx) ... }()
go func() { items, err = s.repo.List(ctx, ...) ... }()
```

and run the suite:

```
==================
WARNING: DATA RACE
Write at 0x00c0001122e0 by goroutine 10:
  ecommerce/product.(*Service).List.func2()
      product/service.go:44 +0x1c4

Previous write at 0x00c0001122e0 by goroutine 9:
  ecommerce/product.(*Service).List.func1()
      product/service.go:37 +0x124
...
--- FAIL: TestNeitherErrorIsEverLost (0.01s)
    service_concurrent_test.go:88: iteration 0: a failed count was silently lost
    testing.go:1712: race detected during execution of test
FAIL
```

**Two independent detectors fire**: the race detector (two goroutines write the same variable) *and* the behavioral assertion (the failure vanished: the list, which completes successfully, overwrote it with `nil`).

There is a subtlety worth remembering. The *first* version of this test used a fake list query that **honors cancellation**. Against the mutant it *passed*: the cancelled list returned `context canceled` (non-nil), so the error survived by accident, and the race detector saw an ordering through the context's channel close. Only after we added `listIgnoresCtx` (a driver that never notices the cancellation, as some real drivers and network calls don't) did the mutant fail. **A test is only as good as the failure it can provoke**: that's why we tried to break the code.

---

## 9. What it bought us: measurements

The same experiment as Chapters 62-63, with **500,000 products**: 300 requests against the previous version (sequential `Count` then `List`) and this one, first page and a deep page, one client and ten concurrent clients:

```
                                     sequential (Ch 63)          concurrent (this chapter)
GET /products              c=1       mean 24.6 ms  p95 31.3     mean 23.1 ms  p95 28.3     (-6%)
GET /products              c=10      mean 91.5 ms  p95 157.5    mean 91.2 ms  p95 145.8    (same)
GET /products?page=12500   c=1       mean 53.4 ms  p95 72.4     mean 39.0 ms  p95 53.8     (-27%)
GET /products?page=12500   c=10      mean 208 ms   p95 317      mean 182 ms   p95 299      (-12%)
```

An honest reading:

1. **First page, one client: almost nothing (-6%).** The count takes ~20 ms and the list ~0.4 ms; overlapping them can save at most the *shorter* one: `sequential = count + list`, `concurrent = max(count, list)`. When one query dominates, concurrency has nothing to overlap. (Recall Chapter 64: total time is the *maximum*, not the sum, so the win is bounded by the *smaller* task.)
2. **Deep page, one client: −27%.** Now both queries take real time (count ~25 ms, list ~28 ms with the large offset), so overlapping saves a lot. Not −50%, because both queries compete for the same PostgreSQL CPU (the count itself is already parallelized by the database) and the connection pool.
3. **Ten concurrent clients: the gain nearly vanishes (−0% to −12%).** With ten requests in flight the machine's CPUs are already busy serving *other requests*; running two queries inside one request adds no capacity. This is Little's law again (Chapter 64): concurrency **inside** a request reduces *latency* when there is idle capacity, but **cannot raise the throughput of a saturated system**. Throughput was ~108 requests/second either way.

The engineering lesson is the one from Chapter 62, in miniature: **measure before and after, and expect the improvement to depend on the workload**. Concurrency here is worth having (deep pages get 27% faster, and it costs ~25 lines), but it isn't magic. The real fixes for this endpoint's cost remain the ones from Chapter 63: avoid the `count(*)` (estimated counts, cached counts, `hasNext`) and avoid deep offsets (keyset pagination).

Also note what we *didn't* measure but should keep in mind: two goroutines per request means two database connections per request; with a pool of 10 connections, five concurrent requests can already saturate it (Chapter 52). Concurrency inside requests **multiplies pressure on shared resources**.

---

## 10. Channels or `WaitGroup` here?

Chapter 69 taught channels; this chapter used a `WaitGroup` and per-goroutine variables. Why? Compare the two designs for *this* problem:

**With a channel:**

```go
type countResult struct{ total int; err error }
counted := make(chan countResult, 1)   // buffer 1: the sender never blocks, even if we give up
go func() { total, err := s.repo.Count(ctx); counted <- countResult{total, err} }()

items, listErr := s.repo.List(ctx, limit, offset)   // run the second query in THIS goroutine
c := <-counted
```

It works, and it is *nice*: one fewer goroutine (the caller does the second query itself), and the result travels with its error in one value. Note the buffer of 1: without it, if we return early, the goroutine blocks forever on its send (the leak of Chapter 69). Cancellation and root-cause selection still need the same logic as before.

**With `WaitGroup` + own variables** (what we wrote): equally correct, and easier to extend to a third or fourth independent query, since `wg.Add(n)` and one more variable scale trivially.

Neither is "better". Use the one that reads clearly; both avoid shared writes. `errgroup` (Chapter 65) is a third option that packages "run, collect the first error, cancel the rest": we didn't use it in the **domain** package only to keep the domain free of third-party imports (Chapter 61's architecture test), not because it's worse.

---

## 11. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | One shared `err` for both goroutines | Lost failures + a data race | One error variable per goroutine |
| 2 | Cancelling on *success* | The other query aborted for no reason | `cancel()` only on failure |
| 3 | Reporting `context canceled` that we caused | Real failure hidden | `rootCause` skips our own cancellation |
| 4 | Ignoring the caller's cancellation | Wasted work after the client left | Derive from the caller's `ctx`; report its error |
| 5 | Not testing overlap | "Concurrent" code that is secretly sequential | Assert peak concurrency, not just elapsed time |
| 6 | Timing-only overlap assertions | Pass by luck | Add an overlap counter in the fake |
| 7 | A fake that isn't goroutine-safe | Race in the *test* code | Guard fakes with a mutex |
| 8 | A fake that always honors cancellation | Bugs hidden (as the mutant showed) | Also model drivers that don't |
| 9 | Adding concurrency without measuring | Complexity for a 6% gain | Benchmark realistic workloads |
| 10 | Expecting intra-request concurrency to raise throughput | Disappointment under load | It lowers latency only when there is idle capacity |
| 11 | Forgetting the connection-pool cost | Pool exhaustion, waits | Size the pool for `requests × goroutines` |
| 12 | Losing an optimization that depended on ordering (skip list on empty page) | Slightly more work | Accept the trade, or measure it |
| 13 | Never trying to break your own tests | False confidence | Mutation testing by hand |
| 14 | Leaking goroutines on the error path | Memory growth | Leak test with error/cancel scenarios |

---

## 12. Interview questions

**Q1. How would you run two independent database queries in one request?**
Two goroutines, each writing its own result and error variables, joined with a `WaitGroup` (or `errgroup`), with a shared cancellable context; report the root cause.

**Q2. Why not share one `err` variable?**
Both goroutines write it: a data race, and one goroutine's `nil` can erase the other's failure.

**Q3. What does cancelling the context on the first error achieve?**
The other query stops early (if it honors the context), saving resources and latency; the response is failing anyway.

**Q4. How do you avoid masking the real error with `context canceled`?**
Skip cancellation errors that our own `cancel()` caused, unless the caller's context was cancelled.

**Q5. How do you prove two goroutines actually ran concurrently in a test?**
Track peak in-flight operations in the fake (assert 2), plus a generous timing bound.

**Q6. Why must test doubles be goroutine-safe?**
Production code may call them concurrently; unsynchronized fields race and fail `-race` even though the service is correct.

**Q7. Why did concurrency help little under load?**
Under load the CPUs are already busy; overlapping work inside a request adds no capacity (Little's law), it only reduces latency when idle capacity exists.

**Q8. What is mutation testing?**
Deliberately introducing a bug to check that the test suite detects it.

**Q9. What is the maximum speedup from overlapping two tasks?**
`(a + b) / max(a, b)`; at most 2×, reached only when both take equal time.

---

## 13. Exercises

### Exercise 1: A third query
Add `Stats(ctx)` (e.g., min/max price) to the port and run it concurrently as a third goroutine, returning it in `Products`. What changes, and does the overlap test need updating?

<details><summary>Solution</summary>

`wg.Add(3)`, a `stats`/`statsErr` pair written only by the third goroutine, a third `wrapIf` in `rootCause`, an extra field in `Products`, and `Stats` in the port, both adapters, the contract suite, and the fake. The overlap test's expected peak becomes 3.
</details>

### Exercise 2: Channel version
Rewrite `List` with the channel design from section 10 (one goroutine, one buffered channel, run the second query in the calling goroutine). Which tests still pass unchanged? Which one would catch a missing buffer?

<details><summary>Solution</summary>

All behavioral tests pass unchanged because the observable behavior is the same. `TestNoGoroutinesAreLeaked` catches an unbuffered channel: when `List` errors and returns early, the sender goroutine would block forever.
</details>

### Exercise 3: Join both errors
When both queries fail genuinely, we return only the first. Change `rootCause` to return `errors.Join` of all non-cancellation errors, and write the test. Is that better for callers?

<details><summary>Solution</summary>

Collect non-cancellation errors in a slice and return `errors.Join(errs...)`; `errors.Is` still matches each. It gives operators the complete picture in logs at the cost of slightly noisier messages; the HTTP layer maps it to a 500 either way.
</details>

### Exercise 4: Timeout per query
Give the two queries their own timeouts (`context.WithTimeout`, 2 s each) inside the goroutines. What should happen to the *other* query if one times out?

<details><summary>Solution</summary>

A timeout is a failure like any other: cancel the sibling and report the timeout as the root cause (its error is `context.DeadlineExceeded`, which is *not* our own `Canceled`, so `rootCause` returns it correctly). Test with the fake's delays.
</details>

### Exercise 5: Bounded concurrency across requests
Two connections per request can exhaust the pool. Add a semaphore (a buffered channel from Chapter 69) in front of the concurrent path so that at most N requests use the parallel version at once, and the rest fall back to sequential. Measure the effect at `-c 20`.

<details><summary>Solution</summary>

`select { case sem <- struct{}{}: defer func(){ <-sem }(); concurrent path; default: sequential path }`: a non-blocking acquire. Under saturation, requests degrade gracefully to using one connection instead of queueing for two.
</details>

### Exercise 6 (challenge): Prove the claim
Section 9 claims the speedup is bounded by `min(count, list)` in time saved. Write a benchmark with fake queries of durations (a, b) for several pairs, run both versions of `List`, and verify `saved ≈ min(a, b)`.

<details><summary>Solution</summary>

With delays as in `TestCountAndListRunAtTheSameTime`: for (150, 150) saved ≈ 150; for (150, 10) saved ≈ 10; for (200, 0) saved ≈ 0. Keep the sequential implementation as a test helper to compare against.
</details>

---

## 14. Quiz

1. Why can `Count` and `List` run concurrently?
2. Why must each goroutine write its own error variable?
3. What does `wg.Wait()` guarantee for the caller's reads?
4. Why does `rootCause` skip `context.Canceled` sometimes?
5. Why is a timing bound alone a weak overlap proof?
6. What did the mutant reveal about the first version of the test?
7. Why did concurrency help less at `c=10`?
8. What is the upper bound on the time saved by overlapping two queries?

<details><summary>Answers</summary>

1. Neither needs the other's result.
2. A shared variable is a data race and one goroutine can overwrite the other's failure.
3. The goroutines' writes (made before `Done`) are visible after `Wait` returns.
4. When *our* `cancel()` caused it, the cancellation is a consequence, not the root cause; a *caller* cancellation is the cause and is reported.
5. It could pass by luck; the fake's peak-in-flight counter proves overlap.
6. A fake that honors cancellation hid the bug; only a driver that ignores cancellation made the test fail.
7. The CPUs were already saturated by other requests; intra-request concurrency adds no capacity.
8. The duration of the *shorter* query (`min(a, b)`).
</details>

---

## 15. Summary

- `product.Service.List` now runs **`Count` and `List` concurrently**: two goroutines, **each writing only its own variables** (`total`/`countErr`, `items`/`listErr`), joined by a `WaitGroup`, with **no mutex and no shared `err`**.
- A **cancellable context** stops the sibling query on the first failure; **`rootCause`** reports the real error and ignores cancellation *we* caused, while a *caller's* cancellation is reported as such.
- The **test double had to become goroutine-safe** (the race detector flagged it at once); fakes gained delays, per-query errors, a "driver ignores cancellation" switch, and a **peak-concurrency counter**.
- Tests prove **overlap** (peak 2, ~150 ms for two 150 ms queries), **root-cause reporting**, **prompt cancellation**, **no lost errors**, **no goroutine leaks**, and clean `-race` runs.
- A hand-made **mutant** (one shared `err`) was caught by the race detector *and* by a failing assertion; the first draft of the test missed it, which is why we test our tests.
- **Measured on 500,000 products:** first page −6%, deep page −27% at one client, and almost no gain at ten clients: **overlap saves at most the shorter query, and cannot raise throughput of a saturated system**. The real fixes for this endpoint remain avoiding `count(*)` and deep offsets.

---

## 16. Course wrap-up

You started with `fmt.Println("hello")`. Over seventy chapters you built, step by step, and *verified* (every code block compiled, the project tested with the race detector, every output pasted from a real run):

| Part | Chapters | What you learned | What it produced |
|------|----------|------------------|------------------|
| **Go fundamentals** | 1-6 | syntax, variables and data types, decisions, functions and return values | the vocabulary of Go |
| **Functions and scope** | 7-17 | scope, shadowing, packages and modules, `init`, anonymous and higher-order functions | how code is organized and how names resolve |
| **Memory and internals** | 18-20 | code/data segments, stack and heap, the garbage collector, closures | what really happens when a program runs |
| **Structs, methods, data structures** | 21-25 | structs, methods, arrays, pointers, slices | your own types and Go's core collections |
| **Computers and operating systems** | 26-32 | architecture, processes, context switching, concurrency vs. parallelism, threads | why concurrency exists and what it costs |
| **Advanced Go concepts** | 33-36 | data types in depth, `defer`, per-thread stacks, goroutines | Go-specific power tools |
| **Backend fundamentals** | 37-39 | how the web and REST work, the journey of a request, the Go runtime | a mental model of a server |
| **The e-commerce API: HTTP and routing** | 40-46 | `net/http`, JSON, CORS and preflight, refactoring, routing, middleware | a working REST API with a request pipeline |
| **Configuration and authentication** | 47-49 | configuration, timeouts, graceful shutdown, JWT, auth middleware | a secure, configurable service |
| **Architecture and patterns** | 50-51, 59-61 | dependency injection, interfaces, design patterns, Domain-Driven Design | code that is easy to change and to test |
| **Databases** | 52-58 | PostgreSQL, `database/sql`, pools, SQL, data types, CRUD, configuration, migrations | durable, safe storage |
| **Performance** | 62-63 | load testing, percentiles, pagination, the cost of `count(*)` and deep offsets | measured, bounded endpoints |
| **Concurrency** | 64-70 | goroutines, `WaitGroup`, races, mutexes, atomics, channels, cancellation | safe, faster code |

The recurring habits matter more than any single API:

1. **Measure before you optimize**, and after (Chapters 62, 63, 70).
2. **Make illegal states unrepresentable** and put each rule in one place (Chapters 59-61).
3. **Fail fast and loudly** at the boundaries; never leak internals or secrets (Chapters 41, 48, 51, 52, 57).
4. **Test at the right level**, with fakes, contract suites, table tests, and `-race` (throughout).
5. **Don't share memory you don't have to**; when you must, protect it completely (Chapters 64-70).
6. **Prove your tests can fail** (Chapters 53, 68, 70).
7. **Read the error message and the source**: Go's runtime and standard library are readable.

### Where to go next

- **The `context` package in depth**, timeouts and deadlines across service boundaries.
- **Observability**: structured logs (`slog`), metrics (Prometheus), tracing (OpenTelemetry), `pprof` profiling.
- **Deployment**: Docker images (multi-stage builds), Kubernetes basics, CI pipelines, health checks (Chapter 52 started this), graceful rollouts.
- **Security**: authorization models beyond "logged in", refresh tokens, rate limiting, input validation as a discipline, dependency scanning (`govulncheck`).
- **Generics and advanced language features**: constraints, iterators, `iter` package.
- **Other transports**: gRPC and Protocol Buffers, WebSockets, message queues (NATS, Kafka).
- **More databases and patterns**: transactions and isolation levels in depth, caching (Redis), full-text search, outbox pattern.
- **Read good code**: the standard library (`net/http`, `sync`, `bufio`), and well-known open-source Go projects.

### How to use this material as a learning platform

- Read a chapter, **type the code yourself**, and run it. Change one thing and predict the result before running.
- Do the **exercises before opening the solutions**; the quizzes are a good check that the ideas stuck.
- When something surprises you, **write a small experiment** as we did throughout: a 20-line program beats a page of speculation.
- Keep the **project** running: each project chapter builds on the previous one, and its tests are your safety net for experiments.

You now have the tools to build, test, measure, and reason about real Go services. Keep building.
