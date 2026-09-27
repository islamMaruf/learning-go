# Chapter 63: Pagination — Query Parameters, Offsets, Totals, and the Trade-offs

> **Goal of this chapter:** Fix the problem measured in Chapter 62. You'll implement `GET /products?page=2&limit=20` through every layer: read **query parameters** (and finally understand Go **maps**, since `url.Values` is one), model paging as a **domain value object** (`PageRequest`) with a default and a hard maximum, add `LIMIT/OFFSET` and a **`Count`** to the repository, and return a response **envelope** with pagination metadata. Then measure again: memory and payload become constant. Finally we look honestly at what pagination *doesn't* solve (the cost of `count(*)` and of **deep offsets**) and at the alternatives: estimated counts, "has next" flags, and **keyset (cursor) pagination**, with real query plans.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 8 hours  **Prerequisite:** [Chapters 61 and 62](62-experiments-before-optimizing.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The idea: pages](#2-the-idea-pages)
3. [Query parameters](#3-query-parameters)
4. [Maps in Go, through `url.Values`](#4-maps-in-go-through-urlvalues)
5. [Design decisions](#5-design-decisions)
6. [The domain: `PageRequest`](#6-the-domain-pagerequest)
7. [The port and the service](#7-the-port-and-the-service)
8. [The adapters: SQL](#8-the-adapters-sql)
9. [The HTTP handler and the response envelope](#9-the-http-handler-and-the-response-envelope)
10. [Tests](#10-tests)
11. [Measuring again](#11-measuring-again)
12. [What pagination does not fix](#12-what-pagination-does-not-fix)
13. [Alternatives: estimates, "has next", keyset pagination](#13-alternatives-estimates-has-next-keyset-pagination)
14. [Common mistakes](#14-common-mistakes)
15. [Interview questions](#15-interview-questions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- Page arithmetic: **offset = (page − 1) × limit**, total pages by **ceiling division**
- **Query parameters** vs. path parameters, and `r.URL.Query()`
- Go **maps** in depth: `map[K]V`, lookups, the comma-ok idiom, `map[string][]string` (`url.Values`)
- Parsing numbers safely with `strconv.Atoi`, and what to do with bad input (400 vs. 422)
- Modelling **defaults and limits as domain rules**
- `LIMIT` / `OFFSET` SQL and a `Count` query; adding methods to a port and to *both* adapters
- Response **envelopes** and pagination metadata (and the API-compatibility cost of adding one)
- Measuring the effect, and finding the *next* bottleneck
- **Keyset pagination**: when and why it beats offsets

---

## 2. The idea: pages

A **page** is a fixed-size window into an ordered list. With `limit` = the page size:

```
 rows (ordered by id):  1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25
 limit = 10
                        └── page 1 ──┘└─── page 2 ───┘└── page 3 (short) ─┘
                          offset 0      offset 10        offset 20
```

Two formulas do all the work:

```
offset     = (page − 1) × limit        rows to SKIP before the page starts
totalPages = ceil(total / limit)       (total + limit − 1) / limit  with integer division
```

Worked examples (`limit = 10`):

| page | offset | rows returned |
|-----:|-------:|---------------|
| 1 | 0 | 1-10 |
| 2 | 10 | 11-20 |
| 3 | 20 | 21-25 (a short last page) |
| 4 | 30 | (none: past the end) |

Same data with `limit = 4`: page 2 is offset 4 → rows 5-8; and `totalPages = ceil(25/4) = 7`.

**Ordering is essential.** "Page 2" only means something if the rows have a *stable order*. A query without `ORDER BY` may return rows in any order, and different orders for consecutive requests, so pages could repeat or skip rows. We always order by `id` (unique, so ties are impossible).

---

## 3. Query parameters

A URL can carry **query parameters** after a `?`, as `key=value` pairs joined by `&`:

```
http://localhost:8080/products?page=2&limit=20
                              └──────query string───┘
```

| | **Path parameter** | **Query parameter** |
|---|--------------------|---------------------|
| Example | `/products/42` | `/products?page=2&limit=20` |
| Identifies | *which* resource | *how to view/filter* a collection |
| Required? | yes (part of the route) | usually optional, with defaults |
| Go | `r.PathValue("id")` | `r.URL.Query().Get("page")` |
| Good for | IDs | paging, sorting, filtering, search |

Rule of thumb: **path = identity, query = options**. `/products/42` is *the* product 42; `/products?category=fruit&page=2` is *a view* of the collection.

In Postman or `curl`, just put the query string in the URL (quote it in the shell: `&` means "run in the background" otherwise):

```bash
curl 'localhost:18080/products?page=2&limit=20'
```

---

## 4. Maps in Go, through `url.Values`

`r.URL.Query()` returns a **`url.Values`**, which is a Go **map**. Time to learn maps properly.

A **map** stores *key → value* associations with fast lookup (average O(1)). Syntax:

```go
ages := map[string]int{"asha": 30, "ravi": 25} // literal
ages["meera"] = 41                             // insert / update
delete(ages, "ravi")                           // remove
fmt.Println(len(ages))                         // 2

empty := make(map[string]int)                  // create an empty map
var nilMap map[string]int                      // a nil map: reads work, WRITES PANIC
```

Reading a missing key returns the **zero value**, which makes "absent" and "zero" indistinguishable, so use the **comma-ok idiom**:

```go
age := ages["ghost"]         // 0: was that "zero years old" or "not present"?
age, ok := ages["ghost"]     // ok == false: definitely not present
```

Other facts: keys must be **comparable** (strings, numbers, bools, arrays, structs of those; *not* slices, maps, or functions); **iteration order is random** (deliberately: never depend on it; sort the keys if you need order); maps are **reference types** (assigning a map shares it); and they are **not safe for concurrent writes** (Chapters 67-68).

`url.Values` is `map[string][]string`: each key maps to a **slice**, because a query may repeat a key (`?tag=fruit&tag=citrus`). A real run:

```go
package main

import (
	"fmt"
	"net/url"
	"strconv"
)

func main() {
	u, _ := url.Parse("http://localhost:8080/products?page=2&limit=20&tag=fruit&tag=citrus&empty=")
	q := u.Query() // url.Values is a map[string][]string

	fmt.Printf("type: %T\n", q)
	fmt.Println("whole map:      ", q)
	fmt.Println("Get(\"page\"):     ", strconv.Quote(q.Get("page")))
	fmt.Println("Get(\"tag\"):      ", strconv.Quote(q.Get("tag")), "(only the first value)")
	fmt.Println("q[\"tag\"]:        ", q["tag"], "(all values)")
	fmt.Println("Get(\"missing\"):  ", strconv.Quote(q.Get("missing")))
	fmt.Println("Has(\"empty\"):    ", q.Has("empty"), "| Get(\"empty\") =", strconv.Quote(q.Get("empty")))
	fmt.Println("Has(\"missing\"):  ", q.Has("missing"))

	for _, raw := range []string{"2", "", "abc", "1.5", "-3", "99999999999999999999"} {
		n, err := strconv.Atoi(raw)
		fmt.Printf("Atoi(%q) = %d, err = %v\n", raw, n, err)
	}
}
```

```
type: url.Values
whole map:       map[empty:[] limit:[20] page:[2] tag:[fruit citrus]]
Get("page"):      "2"
Get("tag"):       "fruit" (only the first value)
q["tag"]:         [fruit citrus] (all values)
Get("missing"):   ""
Has("empty"):     true | Get("empty") =  ""
Has("missing"):   false
Atoi("2") = 2, err = <nil>
Atoi("") = 0, err = strconv.Atoi: parsing "": invalid syntax
Atoi("abc") = 0, err = strconv.Atoi: parsing "abc": invalid syntax
Atoi("1.5") = 0, err = strconv.Atoi: parsing "1.5": invalid syntax
Atoi("-3") = -3, err = <nil>
Atoi("99999999999999999999") = 9223372036854775807, err = strconv.Atoi: parsing "99999999999999999999": value out of range
```

What to take from it:

- **`Get(key)`** returns the *first* value, or `""` if absent; it can't tell "absent" from "present but empty" (`Has` can).
- **Everything in a URL is text.** Convert with `strconv.Atoi`, which returns an **error** for `""`, `"abc"`, `"1.5"`, and out-of-range numbers (and, notably, returns `MaxInt` along with the error, so *always check the error first*).
- `Atoi("-3")` succeeds: **a syntactically valid number can still break a business rule.** Two different kinds of failure, which lead to two different status codes (next section).

---

## 5. Design decisions

Pagination is a *contract* with clients. Decide deliberately:

| Question | Our choice | Alternatives / why |
|----------|-----------|--------------------|
| Default page size | **20** | small enough to be fast, large enough to be useful |
| Maximum page size | **100**, enforced by the server | **must exist**, or the endpoint is unbounded again (Chapter 62). Some APIs *clamp* an oversized limit silently; we **reject** it with an explicit error: predictable, and clients learn the rule |
| Page numbers start at | **1** (human-friendly) | 0-based is common in code, confusing in URLs |
| Absent parameter | use the default | |
| Malformed (`page=abc`) | **400 Bad Request**: we can't even parse it | |
| Well-formed but invalid (`page=0`, `limit=1000`) | **422 Unprocessable Entity**: a business rule failed | the same split we use for request bodies (Chapter 41) |
| Page past the end | **200 with an empty list** | not an error: the client asked a valid question; the answer is "nothing here" |
| Where do the rules live | the **domain** (`PageRequest`) | "how much data can one request pull?" is business policy, not HTTP detail (Chapter 59) |
| Response shape | an **envelope**: `{"items": [...], "pagination": {...}}` | a bare array can't carry metadata. (Alternative: keep the array and put totals in headers such as `X-Total-Count` and `Link`.) |
| Total count | include `total` and `totalPages` | costs an extra query (see section 12) |

**A word on compatibility.** Until now `GET /products` returned a bare JSON array. Wrapping it in an object is a **breaking change** for existing clients. In a real system you would version the API (`/v2/products`) or negotiate the shape; in this course's project we have no external clients, so we change it and update our tests.

---

## 6. The domain: `PageRequest`

The paging rules become a **value object**, in the same style as `Email` (Chapter 60): the only way to get a `PageRequest` is through `NewPageRequest`, so every `PageRequest` anywhere is valid, and any query built from it is *bounded by construction*.

```go
// file: product/page.go
package product

import (
	"fmt"
	"math"
)

// Paging rules. They are business policy (how much data one request may pull), so they live in the domain.
const (
	DefaultPage  = 1
	DefaultLimit = 20
	MaxLimit     = 100
)

// PageRequest asks for one page of a collection: page numbers start at 1, and Limit is the page size.
// It is a value object: the only way to obtain one is NewPageRequest, so every PageRequest is valid
// and every query built from it is bounded.
type PageRequest struct {
	page  int
	limit int
}

// NewPageRequest validates a page number and page size.
func NewPageRequest(page, limit int) (PageRequest, error) {
	switch {
	case page < 1:
		return PageRequest{}, invalid("page must be at least 1")
	case limit < 1 || limit > MaxLimit:
		return PageRequest{}, invalid(fmt.Sprintf("limit must be between 1 and %d", MaxLimit))
	case page-1 > math.MaxInt/limit: // (page-1)*limit would overflow
		return PageRequest{}, invalid("page is too large")
	}
	return PageRequest{page: page, limit: limit}, nil
}

// FirstPage is the default request: page 1 with the default size.
func FirstPage() PageRequest { return PageRequest{page: DefaultPage, limit: DefaultLimit} }

// Page returns the 1-based page number.
func (r PageRequest) Page() int { return r.page }

// Limit returns the page size: the most rows one page may hold.
func (r PageRequest) Limit() int { return r.limit }

// Offset returns how many rows come before this page.
func (r PageRequest) Offset() int { return (r.page - 1) * r.limit }

// Products is one page of the catalog plus what a client needs to navigate: the request and the total row count.
type Products struct {
	Items   []Product
	Request PageRequest
	Total   int // rows in the whole catalog, not just this page
}

// TotalPages returns how many pages the whole catalog has at this page size (0 for an empty catalog).
func (p Products) TotalPages() int {
	if p.Total == 0 {
		return 0
	}
	return (p.Total + p.Request.limit - 1) / p.Request.limit // integer ceiling division
}
```

Notes:

- **`page-1 > math.MaxInt/limit`**: without this check, `?page=9223372036854775807&limit=100` would make `(page-1)*limit` **overflow** and wrap around to a *negative* offset (which PostgreSQL rejects with an error: a 500 for what is really a client mistake). We test the guard.
- Fields are **unexported**, exposed through methods, so nobody can build an invalid `PageRequest{page: -1}` outside the package.
- `Products` bundles the page's items with the request and the total, and derives `TotalPages()` by **integer ceiling division**: `(total + limit − 1) / limit`. An empty catalog has 0 pages (not 1).
- `DefaultPage`, `DefaultLimit`, `MaxLimit` are exported constants so the HTTP adapter can apply defaults for absent parameters *without* duplicating the numbers.

The tests pin every rule, the formula, and the overflow guard:

```go
// file: product/page_test.go
package product

import (
	"errors"
	"math"
	"strings"
	"testing"
)

func TestNewPageRequestValidatesPageAndLimit(t *testing.T) {
	tests := []struct {
		name        string
		page, limit int
		wantOffset  int
		wantErrPart string // "" means valid
	}{
		{"first page", 1, 20, 0, ""},
		{"third page of ten", 3, 10, 20, ""},
		{"largest allowed limit", 1, MaxLimit, 0, ""},
		{"smallest limit", 5, 1, 4, ""},
		{"page zero", 0, 20, 0, "page must be at least 1"},
		{"negative page", -3, 20, 0, "page must be at least 1"},
		{"limit zero", 1, 0, 0, "limit must be between 1 and 100"},
		{"negative limit", 1, -5, 0, "limit must be between 1 and 100"},
		{"limit over the maximum", 1, MaxLimit + 1, 0, "limit must be between 1 and 100"},
		{"absurd limit", 1, math.MaxInt, 0, "limit must be between 1 and 100"},
		{"page so large the offset overflows", math.MaxInt, MaxLimit, 0, "page is too large"},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			req, err := NewPageRequest(tc.page, tc.limit)
			if tc.wantErrPart != "" {
				var ve *ValidationError
				if !errors.As(err, &ve) || !strings.Contains(ve.Message, tc.wantErrPart) {
					t.Fatalf("expected a ValidationError containing %q, got %v", tc.wantErrPart, err)
				}
				return
			}
			if err != nil {
				t.Fatal(err)
			}
			if req.Page() != tc.page || req.Limit() != tc.limit || req.Offset() != tc.wantOffset {
				t.Errorf("got page=%d limit=%d offset=%d", req.Page(), req.Limit(), req.Offset())
			}
		})
	}
}

func TestFirstPageUsesTheDefaults(t *testing.T) {
	r := FirstPage()
	if r.Page() != 1 || r.Limit() != DefaultLimit || r.Offset() != 0 {
		t.Errorf("FirstPage() = page %d limit %d offset %d", r.Page(), r.Limit(), r.Offset())
	}
}

func TestOffsetFormula(t *testing.T) {
	// offset = (page - 1) * limit: the number of rows that come BEFORE the page
	for _, tc := range []struct{ page, limit, want int }{
		{1, 10, 0}, {2, 10, 10}, {3, 10, 20}, {2, 25, 25}, {100, 20, 1980},
	} {
		req, _ := NewPageRequest(tc.page, tc.limit)
		if req.Offset() != tc.want {
			t.Errorf("page %d, limit %d: offset %d, want %d", tc.page, tc.limit, req.Offset(), tc.want)
		}
	}
}

func TestTotalPagesRoundsUp(t *testing.T) {
	for _, tc := range []struct{ total, limit, want int }{
		{0, 20, 0},   // an empty catalog has no pages
		{1, 20, 1},   // one product still needs a page
		{20, 20, 1},  // exactly full
		{21, 20, 2},  // one over: a second page
		{100, 20, 5}, // exact multiple: no extra page
		{101, 20, 6},
		{5, 1, 5},
	} {
		req, _ := NewPageRequest(1, tc.limit)
		if got := (Products{Request: req, Total: tc.total}).TotalPages(); got != tc.want {
			t.Errorf("total %d, limit %d: %d pages, want %d", tc.total, tc.limit, got, tc.want)
		}
	}
}
```

---

## 7. The port and the service

The repository port loses its unbounded `List` and gains bounded `List` + `Count`. **There is deliberately no "list everything" method**: with the unsafe operation removed from the port, no caller can reintroduce the Chapter 62 problem by accident.

```go
// file: product/port.go
package product

import "context"

// Repository stores products. It is a port: implemented by adapters such as postgres.ProductStore.
type Repository interface {
	// List returns at most limit products ordered by ID, skipping the first offset. The slice is never nil.
	// There is deliberately no "list everything" method: every read is bounded.
	List(ctx context.Context, limit, offset int) ([]Product, error)
	// Count returns how many products exist in total.
	Count(ctx context.Context) (int, error)
	// Get returns ErrNotFound if there is no product with that ID.
	Get(ctx context.Context, id int) (Product, error)
	// Create stores p (ignoring p.ID) and returns it with its assigned ID.
	Create(ctx context.Context, p Product) (Product, error)
	// Update replaces the product with the given ID. It returns ErrNotFound if there is none.
	Update(ctx context.Context, id int, p Product) (Product, error)
	// Delete removes the product with the given ID. It returns ErrNotFound if there is none.
	Delete(ctx context.Context, id int) error
}
```

```go
// file: product/service.go
package product

import (
	"context"
	"errors"
	"fmt"
)

// Service implements the catalog use cases on top of the Repository port.
type Service struct {
	repo Repository
}

// NewService creates a Service.
func NewService(repo Repository) *Service { return &Service{repo: repo} }

// List returns one page of the catalog together with the total number of products.
func (s *Service) List(ctx context.Context, req PageRequest) (Products, error) {
	total, err := s.repo.Count(ctx)
	if err != nil {
		return Products{}, fmt.Errorf("count products: %w", err)
	}

	items := []Product{}      // never nil
	if req.Offset() < total { // a page past the end is simply empty: no need to ask the database
		items, err = s.repo.List(ctx, req.Limit(), req.Offset())
		if err != nil {
			return Products{}, fmt.Errorf("list products: %w", err)
		}
	}
	return Products{Items: items, Request: req, Total: total}, nil
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

`List` asks for the total first, then, **only if the page can contain rows** (`offset < total`), for the page itself. A request for page 99,999 costs one cheap `count` and no list query. `items := []Product{}` guarantees a non-nil result whatever the adapter returns.

Service tests, using the fake repository (which now records every `(limit, offset)` it is asked for, so tests can verify *what the service asked the database*):

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

func TestAPagePastTheEndIsEmptyAndSkipsTheQuery(t *testing.T) {
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
	if len(repo.listed) != 0 {
		t.Error("no rows can exist past the end, so the list query should not run")
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

The existing service test file changes only in its fake repository (paged `List`, new `Count`, the `listed` recorder) and in the two tests that called `List`:

```go
// file: product/service_test.go
package product

import (
	"context"
	"errors"
	"math"
	"strings"
	"testing"
)

// fakeRepo is an in-memory Repository that records what the service asked it to store.
type fakeRepo struct {
	items  []Product
	nextID int
	err    error    // if set, every call fails with it
	calls  int      // how many times the repository was touched
	listed [][2]int // the (limit, offset) of every List call
}

func (f *fakeRepo) touch() error { f.calls++; return f.err }

func (f *fakeRepo) List(_ context.Context, limit, offset int) ([]Product, error) {
	if err := f.touch(); err != nil {
		return nil, err
	}
	f.listed = append(f.listed, [2]int{limit, offset}) // remember what was asked for
	if offset >= len(f.items) {
		return []Product{}, nil
	}
	return append([]Product(nil), f.items[offset:min(offset+limit, len(f.items))]...), nil
}

func (f *fakeRepo) Count(context.Context) (int, error) { return len(f.items), f.touch() }

func (f *fakeRepo) Get(_ context.Context, id int) (Product, error) {
	if err := f.touch(); err != nil {
		return Product{}, err
	}
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
	f.nextID++
	p.ID = f.nextID
	f.items = append(f.items, p)
	return p, nil
}

func (f *fakeRepo) Update(_ context.Context, id int, p Product) (Product, error) {
	if err := f.touch(); err != nil {
		return Product{}, err
	}
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

---

## 8. The adapters: SQL

Two small queries. `LIMIT` caps the rows returned; `OFFSET` skips rows first. Values are, as always, **placeholders**:

```sql
-- file: postgres/queries/products_list.sql
SELECT id, title, description, price, image_url
FROM products
ORDER BY id
LIMIT $1 OFFSET $2
```

```sql
-- file: postgres/queries/products_count.sql
SELECT count(*) FROM products
```

```go
// file: postgres/product_store.go
package postgres

import (
	"context"
	"database/sql"
	"ecommerce/product"
	_ "embed" // needed for //go:embed
	"errors"
	"fmt"

	"github.com/jmoiron/sqlx"
)

//go:embed queries/products_list.sql
var listProductsSQL string

//go:embed queries/products_count.sql
var countProductsSQL string

//go:embed queries/products_by_id.sql
var productByIDSQL string

//go:embed queries/products_insert.sql
var insertProductSQL string

//go:embed queries/products_update.sql
var updateProductSQL string

//go:embed queries/products_delete.sql
var deleteProductSQL string

// productRow is how a product looks in the products table (the columns the queries return).
type productRow struct {
	ID          int     `db:"id"`
	Title       string  `db:"title"`
	Description string  `db:"description"`
	Price       float64 `db:"price"`
	ImageURL    string  `db:"image_url"`
}

func (r productRow) toProduct() product.Product {
	return product.Product{ID: r.ID, Title: r.Title, Description: r.Description, Price: r.Price, ImageURL: r.ImageURL}
}

// ProductStore keeps products in PostgreSQL. It is safe for concurrent use: *sqlx.DB is a connection pool.
// It implements product.Repository.
type ProductStore struct {
	db *sqlx.DB
}

// NewProductStore creates a store on top of an existing connection pool (see infra/db).
func NewProductStore(db *sqlx.DB) *ProductStore { return &ProductStore{db: db} }

// List returns at most limit products ordered by ID, skipping the first offset. The result is never nil.
func (s *ProductStore) List(ctx context.Context, limit, offset int) ([]product.Product, error) {
	rows := make([]productRow, 0, limit)
	if err := s.db.SelectContext(ctx, &rows, listProductsSQL, limit, offset); err != nil {
		return nil, fmt.Errorf("list products: %w", err)
	}
	products := make([]product.Product, 0, len(rows))
	for _, r := range rows {
		products = append(products, r.toProduct())
	}
	return products, nil
}

// Count returns how many products exist.
func (s *ProductStore) Count(ctx context.Context) (int, error) {
	var n int
	if err := s.db.GetContext(ctx, &n, countProductsSQL); err != nil {
		return 0, fmt.Errorf("count products: %w", err)
	}
	return n, nil
}

// Get returns the product with the given ID, or product.ErrNotFound.
func (s *ProductStore) Get(ctx context.Context, id int) (product.Product, error) {
	return s.one(ctx, "get product", productByIDSQL, id)
}

// Create inserts p (ignoring p.ID) and returns the stored row.
func (s *ProductStore) Create(ctx context.Context, p product.Product) (product.Product, error) {
	return s.one(ctx, "insert product", insertProductSQL, p.Title, p.Description, p.Price, p.ImageURL)
}

// Update replaces the product with the given ID and returns the stored row, or product.ErrNotFound.
func (s *ProductStore) Update(ctx context.Context, id int, p product.Product) (product.Product, error) {
	return s.one(ctx, "update product", updateProductSQL, id, p.Title, p.Description, p.Price, p.ImageURL)
}

// Delete removes the product with the given ID, or returns product.ErrNotFound if there is none.
func (s *ProductStore) Delete(ctx context.Context, id int) error {
	res, err := s.db.ExecContext(ctx, deleteProductSQL, id)
	if err != nil {
		return fmt.Errorf("delete product: %w", err)
	}
	n, err := res.RowsAffected()
	if err != nil {
		return fmt.Errorf("delete product: %w", err)
	}
	if n == 0 {
		return product.ErrNotFound
	}
	return nil
}

// one runs a query that returns a single product row and translates "no rows" into product.ErrNotFound.
func (s *ProductStore) one(ctx context.Context, what, query string, args ...any) (product.Product, error) {
	var row productRow
	if err := s.db.GetContext(ctx, &row, query, args...); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return product.Product{}, product.ErrNotFound
		}
		return product.Product{}, fmt.Errorf("%s: %w", what, err)
	}
	return row.toProduct(), nil
}
```

`make([]productRow, 0, limit)` pre-sizes the slice to the page size, so the allocation is bounded by the limit (which the domain guarantees is ≤ 100).

The in-memory adapter implements the same contract. It sorts its seed on construction, since `List` promises ID order:

```go
// file: database/product_store.go
package database

import (
	"context"
	"ecommerce/product"
	"sort"
	"sync"
)

// ProductStore keeps products in memory. It is safe for concurrent use.
// Production uses the PostgreSQL store; this one backs fast tests and demos.
type ProductStore struct {
	mu       sync.RWMutex
	products []product.Product
	nextID   int
}

// NewProductStore creates a store, optionally pre-loaded with seed products.
// New products get IDs after the highest seeded ID.
func NewProductStore(seed ...product.Product) *ProductStore {
	s := &ProductStore{nextID: 1}
	for _, p := range seed {
		s.products = append(s.products, p)
		if p.ID >= s.nextID {
			s.nextID = p.ID + 1
		}
	}
	sort.Slice(s.products, func(i, j int) bool { return s.products[i].ID < s.products[j].ID }) // List promises ID order
	return s
}

// SampleProducts returns the demo data used by tests.
func SampleProducts() []product.Product {
	return []product.Product{
		{ID: 1, Title: "Orange", Description: "Orange is juicy and full of vitamin C.", Price: 100, ImageURL: "https://example.com/orange.jpg"},
		{ID: 2, Title: "Apple", Description: "A crunchy apple a day...", Price: 40, ImageURL: "https://example.com/apple.jpg"},
		{ID: 3, Title: "Banana", Description: "Great for a quick snack.", Price: 5, ImageURL: "https://example.com/banana.jpg"},
	}
}

// List returns a copy of at most limit products ordered by ID, skipping the first offset. Never nil.
// (Products are kept in ID order: seeds are sorted on construction and new IDs only grow.)
func (s *ProductStore) List(_ context.Context, limit, offset int) ([]product.Product, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	if offset >= len(s.products) {
		return []product.Product{}, nil
	}
	end := min(offset+limit, len(s.products))
	out := make([]product.Product, end-offset)
	copy(out, s.products[offset:end])
	return out, nil
}

// Count returns how many products exist.
func (s *ProductStore) Count(_ context.Context) (int, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return len(s.products), nil
}

// Get returns the product with the given ID, or product.ErrNotFound.
func (s *ProductStore) Get(_ context.Context, id int) (product.Product, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	if i := s.indexOf(id); i >= 0 {
		return s.products[i], nil
	}
	return product.Product{}, product.ErrNotFound
}

// Create assigns the next ID to p, stores it, and returns the stored product.
func (s *ProductStore) Create(_ context.Context, p product.Product) (product.Product, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	p.ID = s.nextID
	s.nextID++
	s.products = append(s.products, p)
	return p, nil
}

// Update replaces the product with the given ID, or returns product.ErrNotFound.
func (s *ProductStore) Update(_ context.Context, id int, p product.Product) (product.Product, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	i := s.indexOf(id)
	if i < 0 {
		return product.Product{}, product.ErrNotFound
	}
	p.ID = id
	s.products[i] = p
	return p, nil
}

// Delete removes the product with the given ID, or returns product.ErrNotFound.
func (s *ProductStore) Delete(_ context.Context, id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	i := s.indexOf(id)
	if i < 0 {
		return product.ErrNotFound
	}
	s.products = append(s.products[:i], s.products[i+1:]...)
	return nil
}

// indexOf returns the position of the product with the given ID, or -1. The caller must hold the lock.
func (s *ProductStore) indexOf(id int) int {
	for i, p := range s.products {
		if p.ID == id {
			return i
		}
	}
	return -1
}
```

The **contract suite** grows two scenarios that both adapters must pass: "pages are consecutive slices of the ID-ordered list" (including short last pages, huge limits, and offsets past the end) and "count follows creates and deletes":

```go
// file: product/storetest/storetest.go
// Package storetest holds a behavior suite that every product.Repository implementation must pass.
package storetest

import (
	"context"
	"ecommerce/product"
	"errors"
	"strings"
	"sync"
	"testing"
)

// Factory returns a fresh, empty store for one test.
type Factory func(t *testing.T) product.Repository

func sample(title string, price float64) product.Product {
	return product.Product{Title: title, Description: "d", Price: price, ImageURL: "https://example.com/x.jpg"}
}

// all lists everything (the tests never create more than a few dozen products).
func all(t *testing.T, s product.Repository) []product.Product {
	t.Helper()
	list, err := s.List(context.Background(), 1000, 0)
	if err != nil {
		t.Fatal(err)
	}
	return list
}

// Run executes the contract against stores made by newStore.
// Prices use at most two decimals, because that is what every implementation can represent exactly.
func Run(t *testing.T, newStore Factory) {
	ctx := context.Background()

	t.Run("an empty store lists an empty, non-nil slice", func(t *testing.T) {
		s := newStore(t)
		list, err := s.List(ctx, 20, 0)
		if err != nil {
			t.Fatal(err)
		}
		if list == nil || len(list) != 0 {
			t.Errorf("want an empty non-nil slice (JSON []), got %#v", list)
		}
		if n, err := s.Count(ctx); err != nil || n != 0 {
			t.Errorf("Count = %d, %v; want 0", n, err)
		}
	})

	t.Run("create assigns an ID and returns what was stored", func(t *testing.T) {
		s := newStore(t)
		got, err := s.Create(ctx, sample("Mango", 75.5))
		if err != nil {
			t.Fatal(err)
		}
		if got.ID < 1 {
			t.Errorf("expected a generated ID, got %d", got.ID)
		}
		want := sample("Mango", 75.5)
		want.ID = got.ID
		if got != want {
			t.Errorf("got %+v, want %+v", got, want)
		}
	})

	t.Run("create ignores a client-supplied ID", func(t *testing.T) {
		s := newStore(t)
		p := sample("Mango", 1)
		p.ID = 9999
		got, _ := s.Create(ctx, p)
		if got.ID == 9999 {
			t.Error("the store must assign IDs itself")
		}
	})

	t.Run("get finds what create stored", func(t *testing.T) {
		s := newStore(t)
		created, _ := s.Create(ctx, sample("Mango", 75.5))
		got, err := s.Get(ctx, created.ID)
		if err != nil || got != created {
			t.Errorf("got %+v, %v; want %+v", got, err, created)
		}
	})

	t.Run("list is ordered by ID", func(t *testing.T) {
		s := newStore(t)
		for _, title := range []string{"C", "A", "B"} {
			s.Create(ctx, sample(title, 1))
		}
		list := all(t, s)
		if len(list) != 3 || list[0].Title != "C" || list[1].Title != "A" || list[2].Title != "B" {
			t.Errorf("expected creation (ID) order, got %+v", list)
		}
		for i := 1; i < len(list); i++ {
			if list[i].ID <= list[i-1].ID {
				t.Errorf("IDs must increase: %+v", list)
			}
		}
	})

	t.Run("pages are consecutive slices of the ID-ordered list", func(t *testing.T) {
		s := newStore(t)
		for _, title := range []string{"A", "B", "C", "D", "E"} {
			s.Create(ctx, sample(title, 1))
		}

		titles := func(limit, offset int) string {
			list, err := s.List(ctx, limit, offset)
			if err != nil {
				t.Fatal(err)
			}
			var out []string
			for _, p := range list {
				out = append(out, p.Title)
			}
			return strings.Join(out, "")
		}
		for _, tc := range []struct {
			limit, offset int
			want          string
		}{
			{2, 0, "AB"}, {2, 2, "CD"}, {2, 4, "E"}, // the last page may be short
			{5, 0, "ABCDE"}, {100, 0, "ABCDE"}, // a limit larger than the data
			{2, 5, ""}, {2, 500, ""}, // past the end: empty, not an error
			{1, 3, "D"},
		} {
			if got := titles(tc.limit, tc.offset); got != tc.want {
				t.Errorf("List(limit=%d, offset=%d) = %q, want %q", tc.limit, tc.offset, got, tc.want)
			}
		}

		if past, _ := s.List(ctx, 2, 500); past == nil {
			t.Error("a page past the end must be an empty non-nil slice")
		}
	})

	t.Run("count follows creates and deletes", func(t *testing.T) {
		s := newStore(t)
		a, _ := s.Create(ctx, sample("A", 1))
		s.Create(ctx, sample("B", 1))
		if n, _ := s.Count(ctx); n != 2 {
			t.Errorf("Count = %d, want 2", n)
		}
		s.Delete(ctx, a.ID)
		if n, _ := s.Count(ctx); n != 1 {
			t.Errorf("Count = %d after a delete, want 1", n)
		}
	})

	t.Run("missing products are ErrNotFound", func(t *testing.T) {
		s := newStore(t)
		if _, err := s.Get(ctx, 12345); !errors.Is(err, product.ErrNotFound) {
			t.Errorf("Get: got %v", err)
		}
		if _, err := s.Update(ctx, 12345, sample("X", 1)); !errors.Is(err, product.ErrNotFound) {
			t.Errorf("Update: got %v", err)
		}
		if err := s.Delete(ctx, 12345); !errors.Is(err, product.ErrNotFound) {
			t.Errorf("Delete: got %v", err)
		}
	})

	t.Run("update replaces every field and keeps the ID", func(t *testing.T) {
		s := newStore(t)
		created, _ := s.Create(ctx, sample("Mango", 75.5))
		other, _ := s.Create(ctx, sample("Other", 2))

		replacement := product.Product{Title: "Alphonso", Description: "new", Price: 120, ImageURL: "https://example.com/a.jpg"}
		got, err := s.Update(ctx, created.ID, replacement)
		if err != nil {
			t.Fatal(err)
		}
		replacement.ID = created.ID
		if got != replacement {
			t.Errorf("Update returned %+v, want %+v", got, replacement)
		}

		if again, _ := s.Get(ctx, created.ID); again != replacement {
			t.Errorf("stored %+v, want %+v", again, replacement)
		}
		if untouched, _ := s.Get(ctx, other.ID); untouched != other {
			t.Errorf("another product changed: %+v", untouched)
		}
	})

	t.Run("delete removes only that product", func(t *testing.T) {
		s := newStore(t)
		a, _ := s.Create(ctx, sample("A", 1))
		b, _ := s.Create(ctx, sample("B", 2))

		if err := s.Delete(ctx, a.ID); err != nil {
			t.Fatal(err)
		}
		if _, err := s.Get(ctx, a.ID); !errors.Is(err, product.ErrNotFound) {
			t.Errorf("deleted product still found: %v", err)
		}
		if err := s.Delete(ctx, a.ID); !errors.Is(err, product.ErrNotFound) {
			t.Errorf("second delete should be ErrNotFound, got %v", err)
		}
		if list := all(t, s); len(list) != 1 || list[0].ID != b.ID {
			t.Errorf("expected only B to remain, got %+v", list)
		}
	})

	t.Run("hostile text is stored as plain data", func(t *testing.T) {
		s := newStore(t)
		nasty := "Robert'); DROP TABLE products;--"
		created, err := s.Create(ctx, sample(nasty, 1))
		if err != nil {
			t.Fatal(err)
		}
		if got, _ := s.Get(ctx, created.ID); got.Title != nasty {
			t.Errorf("title = %q", got.Title)
		}
		if list := all(t, s); len(list) != 1 {
			t.Errorf("the table must still be intact, got %d rows", len(list))
		}
	})

	t.Run("concurrent creates get unique IDs", func(t *testing.T) {
		s := newStore(t)
		const n = 40

		ids := make(chan int, n)
		var wg sync.WaitGroup
		for i := 0; i < n; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()
				p, err := s.Create(ctx, sample("Concurrent", 1))
				if err != nil {
					t.Error(err)
					return
				}
				ids <- p.ID
			}()
		}
		wg.Wait()
		close(ids)

		seen := map[int]bool{}
		for id := range ids {
			if seen[id] {
				t.Fatalf("duplicate ID %d", id)
			}
			seen[id] = true
		}
		if len(seen) != n {
			t.Errorf("expected %d products, got %d", n, len(seen))
		}
	})
}
```

The in-memory store's own test file only needed its two `List` calls updated:

```go
// file: database/stores_test.go
package database

import (
	"context"
	"ecommerce/product"
	"errors"
	"sync"
	"testing"
)

var ctx = context.Background()

func TestProductStoreIDsAreUniqueAndSequential(t *testing.T) {
	s := NewProductStore()
	const n = 100

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			s.Create(ctx, SampleProducts()[0])
		}()
	}
	wg.Wait()

	list, _ := s.List(ctx, 1000, 0)
	if len(list) != n {
		t.Fatalf("expected %d products, got %d", n, len(list))
	}
	seen := map[int]bool{}
	for _, p := range list {
		if p.ID < 1 || p.ID > n || seen[p.ID] {
			t.Fatalf("bad or duplicate ID %d", p.ID)
		}
		seen[p.ID] = true
	}
}

func TestProductGetReportsNotFound(t *testing.T) {
	s := NewProductStore(SampleProducts()...)
	if _, err := s.Get(ctx, 2); err != nil {
		t.Errorf("product 2 exists: %v", err)
	}
	if _, err := s.Get(ctx, 99); !errors.Is(err, product.ErrNotFound) {
		t.Errorf("expected ErrNotFound, got %v", err)
	}
}

func TestListReturnsACopy(t *testing.T) {
	s := NewProductStore(SampleProducts()...)
	list, _ := s.List(ctx, 10, 0)
	list[0].Title = "tampered"

	if got, _ := s.Get(ctx, 1); got.Title == "tampered" {
		t.Error("callers must not be able to modify the store's internal slice")
	}
}
```

---

## 9. The HTTP handler and the response envelope

```go
// file: rest/producthandler/handler.go
// Package producthandler is the HTTP adapter for the product domain: JSON in, JSON out, status codes.
// It contains no business rules; those live in package product.
package producthandler

import (
	"context"
	"ecommerce/product"
	"ecommerce/util"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"strconv"
)

// Service is what this handler needs from the catalog domain. *product.Service satisfies it.
type Service interface {
	List(ctx context.Context, req product.PageRequest) (product.Products, error)
	Get(ctx context.Context, id int) (product.Product, error)
	Create(ctx context.Context, d product.Details) (product.Product, error)
	Update(ctx context.Context, id int, d product.Details) (product.Product, error)
	Delete(ctx context.Context, id int) error
}

// Handler serves the product endpoints.
type Handler struct {
	svc    Service
	logger *slog.Logger
}

// New creates a Handler.
func New(svc Service, logger *slog.Logger) *Handler { return &Handler{svc: svc, logger: logger} }

// Routes registers this feature's endpoints on mux.
// authn is applied to the routes that change data.
func (h *Handler) Routes(mux *http.ServeMux, authn func(http.Handler) http.Handler) {
	mux.HandleFunc("GET /products", h.List)
	mux.HandleFunc("GET /products/{id}", h.Get)
	mux.Handle("POST /products", authn(http.HandlerFunc(h.Create)))
	mux.Handle("PUT /products/{id}", authn(http.HandlerFunc(h.Update)))
	mux.Handle("DELETE /products/{id}", authn(http.HandlerFunc(h.Delete)))
}

// productRequest is the JSON body of create and replace. It deliberately has no ID:
// the server assigns it (or takes it from the URL).
type productRequest struct {
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}

func (r productRequest) details() product.Details {
	return product.Details{Title: r.Title, Description: r.Description, Price: r.Price, ImageURL: r.ImageURL}
}

// productResponse is the JSON representation of a product.
type productResponse struct {
	ID          int     `json:"id"`
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}

func toResponse(p product.Product) productResponse {
	return productResponse{ID: p.ID, Title: p.Title, Description: p.Description, Price: p.Price, ImageURL: p.ImageURL}
}

// listResponse is the JSON body of GET /products: one page of items plus what a client needs to navigate.
type listResponse struct {
	Items      []productResponse  `json:"items"`
	Pagination paginationResponse `json:"pagination"`
}

type paginationResponse struct {
	Page       int `json:"page"`
	Limit      int `json:"limit"`
	Total      int `json:"total"`
	TotalPages int `json:"totalPages"`
}

// List handles GET /products?page=1&limit=20.
func (h *Handler) List(w http.ResponseWriter, r *http.Request) {
	page, ok := queryInt(w, r, "page", product.DefaultPage)
	if !ok {
		return
	}
	limit, ok := queryInt(w, r, "limit", product.DefaultLimit)
	if !ok {
		return
	}
	req, err := product.NewPageRequest(page, limit) // the domain decides what is an acceptable page
	if err != nil {
		h.fail(w, r, err)
		return
	}

	result, err := h.svc.List(r.Context(), req)
	if err != nil {
		h.fail(w, r, err)
		return
	}

	items := make([]productResponse, 0, len(result.Items)) // never nil: an empty page must encode as [] not null
	for _, p := range result.Items {
		items = append(items, toResponse(p))
	}
	util.SendData(w, http.StatusOK, listResponse{
		Items: items,
		Pagination: paginationResponse{
			Page: req.Page(), Limit: req.Limit(), Total: result.Total, TotalPages: result.TotalPages(),
		},
	})
}

// queryInt reads an optional integer query parameter, using def when it is absent.
// On a malformed value it has already written the error response.
func queryInt(w http.ResponseWriter, r *http.Request, name string, def int) (int, bool) {
	raw := r.URL.Query().Get(name)
	if raw == "" {
		return def, true
	}
	n, err := strconv.Atoi(raw)
	if err != nil {
		util.SendError(w, http.StatusBadRequest, name+" must be a whole number")
		return 0, false
	}
	return n, true
}

// Get handles GET /products/{id}.
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
	id, ok := pathID(w, r)
	if !ok {
		return
	}
	p, err := h.svc.Get(r.Context(), id)
	if err != nil {
		h.fail(w, r, err)
		return
	}
	util.SendData(w, http.StatusOK, toResponse(p))
}

// Create handles POST /products. The route requires authentication (see Routes).
func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
	var req productRequest
	if !util.DecodeJSON(w, r, &req) {
		return
	}
	p, err := h.svc.Create(r.Context(), req.details())
	if err != nil {
		h.fail(w, r, err)
		return
	}
	w.Header().Set("Location", fmt.Sprintf("/products/%d", p.ID))
	util.SendData(w, http.StatusCreated, toResponse(p))
}

// Update handles PUT /products/{id}: it replaces the product with the request body.
func (h *Handler) Update(w http.ResponseWriter, r *http.Request) {
	id, ok := pathID(w, r)
	if !ok {
		return
	}
	var req productRequest
	if !util.DecodeJSON(w, r, &req) {
		return
	}
	p, err := h.svc.Update(r.Context(), id, req.details())
	if err != nil {
		h.fail(w, r, err)
		return
	}
	util.SendData(w, http.StatusOK, toResponse(p))
}

// Delete handles DELETE /products/{id}.
func (h *Handler) Delete(w http.ResponseWriter, r *http.Request) {
	id, ok := pathID(w, r)
	if !ok {
		return
	}
	if err := h.svc.Delete(r.Context(), id); err != nil {
		h.fail(w, r, err)
		return
	}
	w.WriteHeader(http.StatusNoContent) // success, and nothing to say
}

// pathID reads the {id} path parameter. On failure it has already written the error response.
func pathID(w http.ResponseWriter, r *http.Request) (int, bool) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id < 1 {
		util.SendError(w, http.StatusBadRequest, "id must be a positive integer")
		return 0, false
	}
	return id, true
}

// fail translates a domain error into an HTTP response. This is the ONLY place that knows how.
func (h *Handler) fail(w http.ResponseWriter, r *http.Request, err error) {
	var invalid *product.ValidationError
	switch {
	case errors.As(err, &invalid):
		util.SendError(w, http.StatusUnprocessableEntity, invalid.Message)
	case errors.Is(err, product.ErrNotFound):
		util.SendError(w, http.StatusNotFound, "product not found")
	default:
		util.ServerError(w, r, h.logger, err) // unexpected: log the cause, hide it from the client
	}
}
```

The flow of `List`: **parse** (400 on non-numbers) → **ask the domain to validate** (`NewPageRequest`, 422 on rule violations) → **call the service** → **map** to the envelope. Notice that the HTTP layer contains *no paging rules*: not the default, not the maximum, not the offset formula. Change `MaxLimit` in one place and the handler, tests, and error messages all follow (the message text even comes from the constant).

Wiring (`cmd/wire.go`) is **unchanged**: the composition root only builds objects, and no constructor changed.

---

## 10. Tests

The handler tests use a fake `Service` and check the translation: defaults, passing the requested page to the domain, the envelope's shape (including `"items": []` for an empty page), and that bad parameters never reach the domain:

```go
// file: rest/producthandler/handler_test.go
package producthandler

import (
	"bytes"
	"context"
	"ecommerce/product"
	"encoding/json"
	"errors"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

// fakeService lets each test decide what the domain answers, and records what it was asked.
type fakeService struct {
	list   func(req product.PageRequest) (product.Products, error)
	get    func(id int) (product.Product, error)
	create func(d product.Details) (product.Product, error)
	update func(id int, d product.Details) (product.Product, error)
	del    func(id int) error
	called bool
}

func (f *fakeService) List(_ context.Context, req product.PageRequest) (product.Products, error) {
	f.called = true
	return f.list(req)
}
func (f *fakeService) Get(_ context.Context, id int) (product.Product, error) {
	f.called = true
	return f.get(id)
}
func (f *fakeService) Create(_ context.Context, d product.Details) (product.Product, error) {
	f.called = true
	return f.create(d)
}
func (f *fakeService) Update(_ context.Context, id int, d product.Details) (product.Product, error) {
	f.called = true
	return f.update(id, d)
}
func (f *fakeService) Delete(_ context.Context, id int) error {
	f.called = true
	return f.del(id)
}

var _ Service = (*product.Service)(nil) // the real service satisfies the handler's interface

func newHandler(svc Service) (*Handler, *bytes.Buffer) {
	var logs bytes.Buffer
	return New(svc, slog.New(slog.NewTextHandler(&logs, nil))), &logs
}

// call runs a handler method as the router would, with the {id} path value set.
func call(h http.HandlerFunc, method, id, body string) *httptest.ResponseRecorder {
	r := httptest.NewRequest(method, "/products/"+id, strings.NewReader(body))
	r.SetPathValue("id", id)
	rec := httptest.NewRecorder()
	h(rec, r)
	return rec
}

// get runs GET /products with the given raw query string.
func get(h *Handler, query string) *httptest.ResponseRecorder {
	rec := httptest.NewRecorder()
	h.List(rec, httptest.NewRequest(http.MethodGet, "/products?"+query, nil))
	return rec
}

func TestEmptyCatalogIsAnEmptyJSONArray(t *testing.T) {
	h, _ := newHandler(&fakeService{list: func(req product.PageRequest) (product.Products, error) {
		return product.Products{Items: nil, Request: req, Total: 0}, nil // even a nil slice
	}})
	rec := get(h, "")

	var got struct {
		Items      json.RawMessage `json:"items"`
		Pagination map[string]int  `json:"pagination"`
	}
	json.Unmarshal(rec.Body.Bytes(), &got)
	if rec.Code != 200 || string(got.Items) != "[]" {
		t.Errorf("got %d %s, want 200 with \"items\": []", rec.Code, rec.Body.String())
	}
	if got.Pagination["total"] != 0 || got.Pagination["totalPages"] != 0 {
		t.Errorf("pagination = %v", got.Pagination)
	}
}

func TestListPassesTheRequestedPageToTheDomain(t *testing.T) {
	var asked product.PageRequest
	h, _ := newHandler(&fakeService{list: func(req product.PageRequest) (product.Products, error) {
		asked = req
		return product.Products{Items: []product.Product{{ID: 21, Title: "P21", Price: 1}}, Request: req, Total: 45}, nil
	}})

	rec := get(h, "page=2&limit=20")
	if asked.Page() != 2 || asked.Limit() != 20 {
		t.Errorf("the domain was asked for page %d limit %d", asked.Page(), asked.Limit())
	}

	var got struct {
		Items      []map[string]any `json:"items"`
		Pagination paginationResponse
	}
	json.Unmarshal(rec.Body.Bytes(), &got)
	if len(got.Items) != 1 || got.Items[0]["title"] != "P21" {
		t.Errorf("items = %v", got.Items)
	}
	if want := (paginationResponse{Page: 2, Limit: 20, Total: 45, TotalPages: 3}); got.Pagination != want {
		t.Errorf("pagination = %+v, want %+v", got.Pagination, want)
	}
}

func TestListUsesDefaultsWhenTheQueryIsAbsent(t *testing.T) {
	var asked product.PageRequest
	h, _ := newHandler(&fakeService{list: func(req product.PageRequest) (product.Products, error) {
		asked = req
		return product.Products{Request: req}, nil
	}})

	get(h, "")
	if asked.Page() != product.DefaultPage || asked.Limit() != product.DefaultLimit {
		t.Errorf("defaults not applied: page %d limit %d", asked.Page(), asked.Limit())
	}
	get(h, "limit=5") // one parameter given, the other defaulted
	if asked.Page() != 1 || asked.Limit() != 5 {
		t.Errorf("page %d limit %d", asked.Page(), asked.Limit())
	}
}

func TestBadPagingParametersAreRejectedBeforeTheDomainRuns(t *testing.T) {
	svc := &fakeService{}
	h, _ := newHandler(svc)

	tests := []struct {
		query string
		want  int
		body  string
	}{
		{"page=abc", 400, "page must be a whole number"},
		{"limit=ten", 400, "limit must be a whole number"},
		{"page=1.5", 400, "page must be a whole number"},
		{"page=99999999999999999999", 400, "page must be a whole number"}, // overflows int
		{"page=0", 422, "page must be at least 1"},
		{"page=-1", 422, "page must be at least 1"},
		{"limit=0", 422, "limit must be between 1 and 100"},
		{"limit=101", 422, "limit must be between 1 and 100"},
		{"limit=1000000", 422, "limit must be between 1 and 100"},
	}
	for _, tc := range tests {
		rec := get(h, tc.query)
		if rec.Code != tc.want || !strings.Contains(rec.Body.String(), tc.body) {
			t.Errorf("?%s: got %d %s, want %d containing %q", tc.query, rec.Code, rec.Body.String(), tc.want, tc.body)
		}
	}
	if svc.called {
		t.Error("the service must not be called with a rejected page request")
	}
}

func TestResponsesUseTheDocumentedJSONShape(t *testing.T) {
	h, _ := newHandler(&fakeService{get: func(id int) (product.Product, error) {
		return product.Product{ID: id, Title: "Mango", Description: "Sweet", Price: 75.5, ImageURL: "https://example.com/m.jpg"}, nil
	}})
	rec := call(h.Get, http.MethodGet, "3", "")

	var got map[string]any
	json.Unmarshal(rec.Body.Bytes(), &got)
	want := map[string]any{"id": 3.0, "title": "Mango", "description": "Sweet", "price": 75.5, "imageUrl": "https://example.com/m.jpg"}
	for k, v := range want {
		if got[k] != v {
			t.Errorf("%s = %v, want %v", k, got[k], v)
		}
	}
	if len(got) != len(want) {
		t.Errorf("unexpected extra fields: %v", got)
	}
}

func TestDomainOutcomesMapToStatusCodes(t *testing.T) {
	boom := errors.New("connection to database lost: password=hunter2")
	validation := &product.ValidationError{Message: "price must be greater than 0"}

	tests := []struct {
		name   string
		err    error
		status int
		body   string
	}{
		{"validation", validation, 422, "price must be greater than 0"},
		{"not found", product.ErrNotFound, 404, "product not found"},
		{"wrapped not found", errors.Join(errors.New("query"), product.ErrNotFound), 404, "product not found"},
		{"unexpected", boom, 500, "internal server error"},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			h, logs := newHandler(&fakeService{
				update: func(int, product.Details) (product.Product, error) { return product.Product{}, tc.err },
			})
			rec := call(h.Update, http.MethodPut, "1", `{"title":"x","price":1}`)

			if rec.Code != tc.status || !strings.Contains(rec.Body.String(), tc.body) {
				t.Errorf("got %d %s, want %d containing %q", rec.Code, rec.Body.String(), tc.status, tc.body)
			}
			if tc.status == 500 {
				if strings.Contains(rec.Body.String(), "hunter2") {
					t.Error("internal details leaked to the client")
				}
				if !strings.Contains(logs.String(), "connection to database lost") {
					t.Errorf("the cause must be logged: %q", logs.String())
				}
			} else if logs.Len() != 0 {
				t.Errorf("expected outcomes are not errors and must not be logged: %q", logs.String())
			}
		})
	}
}

func TestCreateAndDeleteStatuses(t *testing.T) {
	svc := &fakeService{
		create: func(d product.Details) (product.Product, error) {
			return product.Product{ID: 42, Title: d.Title, Price: d.Price}, nil
		},
		del: func(int) error { return nil },
	}
	h, _ := newHandler(svc)

	rec := call(h.Create, http.MethodPost, "", `{"title":"Mango","price":5}`)
	if rec.Code != http.StatusCreated || rec.Header().Get("Location") != "/products/42" {
		t.Errorf("create: got %d, Location %q", rec.Code, rec.Header().Get("Location"))
	}

	rec = call(h.Delete, http.MethodDelete, "42", "")
	if rec.Code != http.StatusNoContent || rec.Body.Len() != 0 {
		t.Errorf("delete: got %d %q, want 204 with no body", rec.Code, rec.Body.String())
	}
}

func TestBadRequestsNeverReachTheDomain(t *testing.T) {
	svc := &fakeService{}
	h, _ := newHandler(svc)

	for name, rec := range map[string]*httptest.ResponseRecorder{
		"get non-numeric id":      call(h.Get, http.MethodGet, "abc", ""),
		"get zero id":             call(h.Get, http.MethodGet, "0", ""),
		"delete negative id":      call(h.Delete, http.MethodDelete, "-3", ""),
		"update bad id":           call(h.Update, http.MethodPut, "1.5", `{"title":"x","price":1}`),
		"create empty body":       call(h.Create, http.MethodPost, "", ``),
		"create truncated json":   call(h.Create, http.MethodPost, "", `{"title": `),
		"create wrong type":       call(h.Create, http.MethodPost, "", `{"title":"x","price":"cheap"}`),
		"create unknown field":    call(h.Create, http.MethodPost, "", `{"title":"x","price":1,"id":9}`),
		"update trailing garbage": call(h.Update, http.MethodPut, "1", `{"title":"x","price":1} {}`),
	} {
		if rec.Code != http.StatusBadRequest {
			t.Errorf("%s: expected 400, got %d (%s)", name, rec.Code, rec.Body.String())
		}
	}
	if svc.called {
		t.Error("the domain must not be called for malformed requests")
	}
}
```

End to end, through the whole HTTP stack, with the app's helper now reading the total from the metadata (it can no longer count the items in the body: the first page holds at most 20):

```go
// file: rest/app_test.go
package rest

import (
	"context"
	"ecommerce/auth"
	"ecommerce/config"
	"ecommerce/database"
	"ecommerce/health"
	"ecommerce/product"
	"ecommerce/rest/producthandler"
	"ecommerce/rest/userhandler"
	"ecommerce/user"
	"encoding/json"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

var quietLogger = slog.New(slog.NewTextHandler(io.Discard, nil))

// alwaysUp is a health.Pinger for a database that always answers.
type alwaysUp struct{}

func (alwaysUp) PingContext(context.Context) error { return nil }

// testApp is a complete, private instance of the application.
type testApp struct {
	t       *testing.T
	cfg     *config.Config
	handler http.Handler
}

func newApp(t *testing.T) *testApp {
	t.Helper()
	cfg := &config.Config{
		Env:            "test",
		Port:           "0",
		AllowedOrigins: []string{"http://localhost:5173"},
		LogFormat:      "text",
		JWTSecret:      "0123456789abcdef0123456789abcdef0123",
		JWTTTL:         time.Hour,
	}

	// the same wiring as production (cmd/wire.go), with in-memory storage
	userService, err := user.NewService(database.NewUserStore(), auth.BcryptHasher{},
		auth.TokenIssuer{Secret: []byte(cfg.JWTSecret), TTL: cfg.JWTTTL})
	if err != nil {
		t.Fatal(err)
	}
	productService := product.NewService(database.NewProductStore(database.SampleProducts()...))

	server := NewServer(cfg, quietLogger,
		producthandler.New(productService, quietLogger),
		userhandler.New(userService, quietLogger),
		health.NewHandler(alwaysUp{}, quietLogger),
	)
	return &testApp{t: t, cfg: cfg, handler: server.Handler()}
}

func (a *testApp) do(method, target, body string, headers map[string]string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	if body != "" {
		req.Header.Set("Content-Type", "application/json")
	}
	for k, v := range headers {
		req.Header.Set(k, v)
	}
	rec := httptest.NewRecorder()
	a.handler.ServeHTTP(rec, req)
	return rec
}

// bearer returns headers carrying a valid token for the given user ID.
func (a *testApp) bearer(userID int) map[string]string {
	a.t.Helper()
	token, err := auth.CreateToken([]byte(a.cfg.JWTSecret), userID, time.Hour, time.Now())
	if err != nil {
		a.t.Fatal(err)
	}
	return map[string]string{"Authorization": "Bearer " + token}
}

// count returns the total number of products, as reported by the list endpoint's pagination metadata.
func (a *testApp) count() int {
	var body struct {
		Pagination struct{ Total int } `json:"pagination"`
	}
	json.NewDecoder(a.do(http.MethodGet, "/products", "", nil).Body).Decode(&body)
	return body.Pagination.Total
}
```

```go
// file: rest/pagination_test.go
package rest

import (
	"encoding/json"
	"fmt"
	"net/http"
	"testing"
)

// page is the decoded body of GET /products.
type page struct {
	Items []struct {
		ID    int    `json:"id"`
		Title string `json:"title"`
	} `json:"items"`
	Pagination struct {
		Page       int `json:"page"`
		Limit      int `json:"limit"`
		Total      int `json:"total"`
		TotalPages int `json:"totalPages"`
	} `json:"pagination"`
}

func getPage(t *testing.T, a *testApp, query string) (page, int) {
	t.Helper()
	rec := a.do(http.MethodGet, "/products"+query, "", nil)
	var p page
	json.NewDecoder(rec.Body).Decode(&p)
	return p, rec.Code
}

// addProducts creates n more products through the API (the app starts with 3).
func addProducts(t *testing.T, a *testApp, n int) {
	t.Helper()
	for i := 0; i < n; i++ {
		body := fmt.Sprintf(`{"title":"Extra %d","price":1}`, i)
		if rec := a.do(http.MethodPost, "/products", body, a.bearer(1)); rec.Code != http.StatusCreated {
			t.Fatalf("creating product %d: %d", i, rec.Code)
		}
	}
}

func TestListDefaultsToTheFirstPageOfTwenty(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	addProducts(t, a, 47) // 50 in total

	p, code := getPage(t, a, "")
	if code != http.StatusOK || len(p.Items) != 20 {
		t.Fatalf("got %d with %d items, want 200 with 20", code, len(p.Items))
	}
	if p.Pagination.Page != 1 || p.Pagination.Limit != 20 || p.Pagination.Total != 50 || p.Pagination.TotalPages != 3 {
		t.Errorf("pagination = %+v", p.Pagination)
	}
}

func TestWalkingEveryPageVisitsEveryProductExactlyOnce(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	addProducts(t, a, 47) // 50 in total

	seen := map[int]bool{}
	last := 0
	for pageNo := 1; ; pageNo++ {
		p, code := getPage(t, a, fmt.Sprintf("?page=%d&limit=7", pageNo))
		if code != http.StatusOK {
			t.Fatalf("page %d: status %d", pageNo, code)
		}
		if len(p.Items) == 0 {
			break
		}
		for _, item := range p.Items {
			if seen[item.ID] {
				t.Fatalf("product %d appeared twice", item.ID)
			}
			if item.ID <= last {
				t.Fatalf("items must be ordered by ID: %d after %d", item.ID, last)
			}
			seen[item.ID], last = true, item.ID
		}
		if pageNo > 100 {
			t.Fatal("pagination never ended")
		}
	}
	if len(seen) != 50 {
		t.Errorf("visited %d products, want 50", len(seen))
	}
}

func TestTheLastPageIsShortAndPastTheEndIsEmpty(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	addProducts(t, a, 47)

	if p, _ := getPage(t, a, "?page=3"); len(p.Items) != 10 || p.Pagination.TotalPages != 3 {
		t.Errorf("page 3 of 50 at 20 per page should hold 10 items, got %d (%+v)", len(p.Items), p.Pagination)
	}

	p, code := getPage(t, a, "?page=4")
	if code != http.StatusOK || p.Items == nil || len(p.Items) != 0 || p.Pagination.Total != 50 {
		t.Errorf("a page past the end is a normal, empty result: got %d %+v", code, p)
	}
}

func TestBadPagingParametersAreClientErrors(t *testing.T) {
	t.Parallel()
	a := newApp(t)

	for query, want := range map[string]int{
		"?page=0": 422, "?page=-2": 422, "?limit=0": 422, "?limit=101": 422, "?limit=5000": 422,
		"?page=one": 400, "?limit=": 200 /* empty = default */, "?limit=100": 200, "?page=1&limit=1": 200,
	} {
		if _, code := getPage(t, a, query); code != want {
			t.Errorf("GET /products%s: got %d, want %d", query, code, want)
		}
	}
}

func TestNoRequestCanAskForEverything(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	addProducts(t, a, 197) // 200 in total, twice the maximum page size

	p, _ := getPage(t, a, "?limit=100")
	if len(p.Items) != 100 {
		t.Errorf("the largest page holds %d items, want exactly the maximum (100)", len(p.Items))
	}
	if _, code := getPage(t, a, "?limit=200"); code != http.StatusUnprocessableEntity {
		t.Errorf("asking for all 200 in one go must be refused, got %d", code)
	}
}
```

`TestWalkingEveryPageVisitsEveryProductExactlyOnce` is the *property* pagination must satisfy: walk all pages with an awkward page size (7 doesn't divide 50) and verify no product is skipped, none is repeated, and IDs increase. `TestNoRequestCanAskForEverything` is the regression test for Chapter 62's problem.

Run everything (the last command exercises PostgreSQL):

```bash
go vet ./... && go test -race ./...
TEST_DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable' go test -race -count=1 ./...
```

---

## 11. Measuring again

Repeat Chapter 62's worst case: **500,000 products**. The endpoint now answers:

```bash
curl 'localhost:18080/products?limit=2'
```
```
{"items":[{"id":1,"title":"Product 1","description":"A product created by the bulk seed script, number 1","price":2,"imageUrl":"https://example.com/img/1.jpg"},{"id":2,"title":"Product 2","description":"A product created by the bulk seed script, number 2","price":3,"imageUrl":"https://example.com/img/2.jpg"}],"pagination":{"page":1,"limit":2,"total":500000,"totalPages":250000}}
```

```bash
curl 'localhost:18080/products?page=99999&limit=100'      # far past the end
curl 'localhost:18080/products?page=0'
curl 'localhost:18080/products?limit=101'
curl 'localhost:18080/products?page=abc'
```
```
{"items":[],"pagination":{"page":99999,"limit":100,"total":500000,"totalPages":5000}}
{"error":"page must be at least 1"}
{"error":"limit must be between 1 and 100"}
{"error":"page must be a whole number"}
```

Now the same load test as before, 200 requests with 10 concurrent workers:

```bash
./loadgen -n 200 -c 10 localhost:18080/products
```
```
requests:     200 in 1.36s (147.1 req/s)
latency:      mean 66.83ms | p50 62.523ms | p95 104.795ms | p99 133.63ms | max 151.144ms
status 200:   200
downloaded:   0.63 MB total, 3.1 KB per successful request
```

| Measurement (500,000 products) | Before (Chapter 62) | After |
|--------------------------------|---------------------|-------|
| Response size | 85 MB | **3.1 KB** |
| Server memory, one request | 454 MB peak | **~14 MB** (the idle baseline) |
| Server memory, 10 concurrent | ~4 GB (extrapolated from 5 → 2 GB) | **21 MB** peak after 200 requests |
| Bytes downloaded per user action | 85 MB | 3.1 KB (27,000× less) |

**The memory problem is gone**, and it no longer depends on the table size or on concurrency (the ceiling is now `users × page size`, both bounded). Also measured:

```
?limit=100  (200 requests, c=10):  147 → 163 req/s, 15.5 KB per response
?page=12500 (200 requests, c=10):   70.6 req/s, mean 137 ms, p95 237 ms
```

But look at the latency numbers: the *first page* takes **~22 ms** and the endpoint manages only ~150 requests per second. That is 25× slower than fetching a single product by ID (Chapter 62: 2.6 ms) even though we now return only 20 rows. Something else is expensive. We just moved the bottleneck.

---

## 12. What pagination does not fix

Ask PostgreSQL where the time goes (`EXPLAIN (ANALYZE)` runs the query and reports actual work):

```sql
SELECT count(*) FROM products;
```
```
Time: 24.090 ms    (repeat runs: 16-18 ms)

 Finalize Aggregate
   ->  Gather (workers: 2)
         ->  Partial Aggregate
               ->  Parallel Seq Scan on products (actual rows=166667 loops=3)
 Execution Time: 16.394 ms
```

### Cost 1: `count(*)` reads the whole table

PostgreSQL's storage (MVCC: each transaction may see a different set of rows) means it **cannot** answer `count(*)` from a stored counter; it must scan the table (or an index) and count the rows visible to *this* transaction. On 500,000 rows that's ~16-24 ms, even with two parallel workers, **on every request**, and it grows linearly with the table. It is now the dominant cost of our endpoint.

### Cost 2: deep `OFFSET` reads and discards rows

```sql
SELECT id, title FROM products ORDER BY id LIMIT 20 OFFSET 250000;
```
```
 Limit (actual rows=20 loops=1)
   ->  Index Scan using products_pkey on products (actual rows=250020 loops=1)
 Execution Time: 21.528 ms
```

Look at `actual rows=250020`: to return 20 rows the database read **250,020** and threw away 250,000. Page 12,500 costs 260× more than page 1. Our load test agrees: deep pages ran at **70 req/s vs. 163 req/s** for shallow ones. `OFFSET` pagination gets slower the further you go, and crawlers and "jump to the last page" buttons hit exactly the slow end.

### Cost 3: the two queries are not one snapshot

`Count` and `List` are separate statements. Rows can be inserted or deleted in between, so `total` can disagree slightly with the pages you walk. And even within one series of requests: if a row is **inserted** at the front while a user moves from page 1 to page 2, everything shifts down by one, and the last row of page 1 **reappears** at the top of page 2. (Deleting causes the opposite: a row is *skipped*.) Offsets identify a *position*, and positions move.

None of this makes offset pagination *wrong*. For admin screens and moderate tables it is simple, familiar, and lets users jump to "page 7". But you should know where it stops working.

---

## 13. Alternatives: estimates, "has next", keyset pagination

### Cheaper totals

| Technique | How | Trade-off |
|-----------|-----|-----------|
| **Estimated count** | `SELECT reltuples::bigint FROM pg_class WHERE relname = 'products'` (maintained by `ANALYZE`/autovacuum) | instant (0.85 ms) but approximate; after `ANALYZE` it returned exactly `500000`; on a table nobody analyzed it returned `-1` |
| **Cache the count** | compute every N seconds/minutes or on write, store in memory/Redis | slightly stale |
| **Omit the total** | return `hasNext` instead: query `limit + 1` rows; if you got `limit + 1`, there's a next page (return only `limit`) | no "page 3 of 5000", but exactly what infinite scroll needs; no count query at all |
| **Cap the count** | `SELECT count(*) FROM (SELECT 1 FROM products LIMIT 10001) t`, display "10,000+" | bounded cost |

Big sites rarely show exact totals for large collections, and now you know why.

### Keyset (cursor) pagination

Instead of "skip N rows", say **"give me the rows after the last one I saw"**. The client sends the last `id` it received (a *cursor*), and the query uses the index to jump straight there:

```sql
-- page after id 250000
SELECT id, title FROM products WHERE id > 250000 ORDER BY id LIMIT 20;
```
```
 Limit (actual rows=20 loops=1)
   ->  Index Scan using products_pkey on products (actual rows=20 loops=1)
         Index Cond: (id > 250000)
 Execution Time: 0.081 ms
```

**0.081 ms and 20 rows read**, versus 21.5 ms and 250,020 rows read for the equivalent `OFFSET`: **~265× faster**, and *constant* whether you're at page 2 or page 200,000, because the database seeks to `id > 250000` through the index. It is also **stable under inserts and deletes**: "after id X" doesn't shift when rows appear elsewhere.

The costs of keyset pagination:

- **No jumping** to an arbitrary page number, only *next* (and *previous*, with a mirrored query).
- Sorting by a non-unique column (say `price`) requires a **tiebreaker**: `WHERE (price, id) > ($1, $2) ORDER BY price, id` (row-value comparison, which PostgreSQL can serve from a composite index).
- The cursor should be **opaque** to clients (e.g., base64 of the last key), so you can change its meaning later.

A typical cursor API looks like:

```json
{ "items": [ … 20 products … ],
  "nextCursor": "eyJpZCI6MjUwMDIwfQ" }        // request the next page with ?after=eyJpZCI6MjUwMDIwfQ
```

**Which to choose?** Use **offset** pagination when users need page numbers and tables are modest (admin panels, small catalogs); use **keyset** for large or fast-changing data and infinite scrolling (feeds, logs, big catalogs, APIs consumed by machines). The design of this chapter (a `PageRequest` value object hiding the mechanism, a bounded repository port) makes that switch a *local* change: Exercise 5 implements it.

---

## 14. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | No maximum page size | `?limit=1000000` recreates Chapter 62 | Server-enforced cap (domain rule) |
| 2 | `LIMIT/OFFSET` without `ORDER BY` | Pages overlap or skip rows | Deterministic order, with a unique tiebreaker |
| 3 | Trusting `strconv.Atoi`'s value when it returned an error | Uses `MaxInt` or `0` by accident | Check `err` first |
| 4 | Offset overflow (`page × limit`) | Negative offsets, database errors | Guard against overflow in the domain |
| 5 | Building `LIMIT`/`OFFSET` by string concatenation | SQL injection | Placeholders `$1, $2` |
| 6 | 0-based vs 1-based confusion | Off-by-one: page 1 skips the first page | Document; test `offset(1) == 0` |
| 7 | Treating a page past the end as an error | Clients need special handling | `200` with an empty list |
| 8 | Returning `null` for an empty page | JS clients crash on `items.map` | Always a non-nil slice |
| 9 | `count(*)` on every request of a huge table | The new bottleneck | Estimate, cache, or `hasNext` |
| 10 | Deep `OFFSET` for crawlers/infinite scroll | Slower with every page | Keyset pagination |
| 11 | Page rules duplicated in handler and service | Drift | One `PageRequest` in the domain |
| 12 | Forgetting the response shape change is a breaking change | Existing clients fail | Version the API |
| 13 | Testing with 50 rows | Never sees the performance cliff | Test with production-scale data (Chapter 62) |
| 14 | Non-deterministic ordering in tests (map iteration) | Flaky tests | Sort keys / explicit `ORDER BY` |

---

## 15. Interview questions

**Q1. Derive the offset for page `p` with size `n`.**
`(p − 1) × n` (1-based pages): the number of rows before the page.

**Q2. Path parameter or query parameter for `page`?**
Query: it's an optional view option on a collection, not the identity of a resource.

**Q3. Why validate the page size on the server if the client already limits it?**
Clients can't be trusted (bugs, scripts, attackers); an unbounded page recreates the memory/DoS problem.

**Q4. 400 or 422 for `?limit=abc` and `?limit=0`?**
`abc` can't be parsed → 400. `0` parses but violates a rule → 422 (some APIs use 400 for both: consistency matters more than the exact choice).

**Q5. Why is `OFFSET 250000` slow, and what's the alternative?**
The database reads and discards 250,000 rows. Keyset pagination (`WHERE id > last ORDER BY id LIMIT n`) seeks via the index and reads only the page.

**Q6. Why is `count(*)` slow in PostgreSQL?**
MVCC: row visibility is per-transaction, so it must scan to count visible rows; there's no exact stored counter.

**Q7. How can you provide "has next page" without counting?**
Fetch `limit + 1` rows; if `limit + 1` came back, there is another page (return only `limit`).

**Q8. What can go wrong when rows change during paging with offsets?**
Insertions shift rows down (a row appears on two pages); deletions shift them up (a row is skipped). Keyset cursors avoid this.

**Q9. Why does the domain, not the handler, own the maximum page size?**
It's business policy about resource use, shared by every entry point (HTTP, gRPC, CLI), and testable without HTTP.

---

## 16. Exercises

### Exercise 1: Sorting
Add `?sort=price` and `?sort=-price` (descending) to the list endpoint. Why must you **not** put the parameter into the SQL string directly? How do you keep the order deterministic?

<details><summary>Solution</summary>

Identifiers can't be placeholders, so map the parameter through an **allow-list** in the domain (`SortByID`, `SortByPrice`, `SortByPriceDesc`) and let the adapter pick a *fixed* SQL string per option. Never interpolate user input. For determinism add the unique tiebreaker: `ORDER BY price, id`. Validate unknown values with a 422.
</details>

### Exercise 2: Filtering with paging
Add `?minPrice=` and `?maxPrice=`. Which query must change besides `List`? What happens if you forget?

<details><summary>Solution</summary>

`Count` must apply the same filter, or `total`/`totalPages` describe the whole table while the items are filtered: the client would paginate through empty pages. Best: pass one `Filter` value to both methods (a single source of truth).
</details>

### Exercise 3: Has-next instead of totals
Implement a `?total=false` option (or make it the default) that skips `Count` and returns `"hasNext": true/false` using the `limit + 1` trick. What changes in the port?

<details><summary>Solution</summary>

Keep the port's `List(limit, offset)`, and let the *service* ask for `limit+1`, trim, and set `HasNext`. `Products` gains `HasNext bool` and an optional `Total *int` (`nil` = not computed). Tests: exactly `limit` rows → `hasNext=false`; `limit+1` rows → `true` and only `limit` items returned.
</details>

### Exercise 4: Links
Add RFC 8288 `Link` headers (`rel="next"`, `"prev"`, `"first"`, `"last"`). Where does building them belong?

<details><summary>Solution</summary>

In the **HTTP adapter** (URLs are a delivery concern); the domain only supplies page numbers and totals. Build them with `net/url` (copy the incoming query, replace `page`), and omit `prev` on page 1 and `next` on the last page.
</details>

### Exercise 5 (challenge): Keyset pagination
Add `ListAfter(ctx, afterID, limit)` to the repository, a `Cursor` value object (opaque base64 of the last ID, validated), and `GET /products?after=<cursor>&limit=20` returning `nextCursor`. Extend the contract suite. Measure page 12,500 vs. page 1 with the load generator.

<details><summary>Solution</summary>

SQL: `SELECT … FROM products WHERE id > $1 ORDER BY id LIMIT $2`. `Cursor`: `NewCursor(afterID)`, `ParseCursor(string)` (base64-decode, parse, reject negatives with a `ValidationError`), `String()`. Service: fetch `limit+1`, compute `nextCursor` from the last returned item's ID if more remain. Contract scenario: walking cursors to the end visits every product once *even when products are inserted mid-walk*. Expect deep and shallow pages to have the same latency (~0.1 ms in the database, ≈ 1-3 ms total).
</details>

### Exercise 6: Measure the estimate
Return `total` from `pg_class.reltuples` when the table has more than 100,000 rows and from `count(*)` otherwise. What do you do when `reltuples` is `-1`?

<details><summary>Solution</summary>

`-1` means "never analyzed": fall back to `count(*)` (or run `ANALYZE`). Document that large totals are approximate (`"totalIsEstimate": true`), and test both branches with a fake repository.
</details>

---

## 17. Quiz

1. What's the offset of page 4 with 25 rows per page?
2. How many pages hold 101 items at 20 per page?
3. What is the type of `r.URL.Query()`?
4. Why should `Atoi`'s error be checked before using its value?
5. Why is a page past the end a `200`?
6. What made the first page still take 22 ms after pagination?
7. Why is keyset pagination faster for deep pages?
8. Why is the maximum page size a domain rule?

<details><summary>Answers</summary>

1. `(4 − 1) × 25 = 75`.
2. `ceil(101/20) = 6`.
3. `url.Values`, i.e., `map[string][]string`.
4. On error it returns 0 (or `MaxInt` for out-of-range), which looks like a real value.
5. The request was valid; the honest answer is "no items"; clients need no special handling.
6. `count(*)` scanned all 500,000 rows on every request.
7. It seeks via the index to `id > last` and reads only the page; offsets read and discard every skipped row.
8. It's business policy about resource use, shared by every caller and testable without HTTP.
</details>

---

## 18. Summary

- **Paging formulas:** `offset = (page − 1) × limit`; `totalPages = ceil(total / limit)`. Always `ORDER BY` a unique key.
- **Query parameters** carry options (`?page=2&limit=20`); Go exposes them as `url.Values`, a **`map[string][]string`**; you learned maps: literals, comma-ok, zero values, comparable keys, random iteration order, nil-map writes panic.
- The **domain owns the rules**: `PageRequest` (a validated value object with defaults `1`/`20`, maximum `100`, overflow guard); the port has only **bounded** reads (`List(limit, offset)` and `Count`); the service skips the list query for pages past the end.
- **HTTP**: malformed input → `400`, rule violations → `422`, a page past the end → `200` with `items: []`; the response is an **envelope** with `pagination` metadata (a breaking change worth versioning in real APIs).
- **Measured on 500,000 products:** response 85 MB → 3.1 KB; server memory 454 MB → ~14 MB; memory no longer grows with table size or concurrency.
- **The next bottlenecks:** `count(*)` (~16-24 ms) and deep `OFFSET` (reads 250,020 rows to return 20). Remedies: estimated or cached counts, `hasNext`, and **keyset pagination** (0.081 ms, constant, stable under writes).

### ➡️ What's next?

**Part 14: concurrency.** With the data layer solid, we turn to Go's signature strength. [Chapter 64](64-why-concurrency-matters.md) explains **why goroutines matter**: doing many things at once, and how much faster a program can be when independent work overlaps. Later chapters cover `WaitGroup`, races, mutexes, and channels, the tools behind the load generator you built in Chapter 62.
