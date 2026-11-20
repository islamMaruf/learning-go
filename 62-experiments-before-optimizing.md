# Chapter 62: Experiments Before Optimizing — Why Returning Everything Breaks Down

> **Goal of this chapter:** Discover, with *measurements*, why an endpoint like `GET /products` cannot return "everything" once the data grows. You'll build a small **load generator** (in Go, about 150 lines, with tests), fill the database with 100,000 and then 500,000 products, and watch response size, latency, and server memory grow with the data (and multiply with concurrent users). You'll learn to read **latency percentiles**, how to measure a process's peak memory, and why "it works on my machine with 3 rows" tells you almost nothing. The fix (pagination) comes in Chapter 63; this chapter earns the right to build it.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 5 hours  **Prerequisite:** [Chapters 58 and 61](61-ddd-in-code-part-2.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The scientific method for performance](#2-the-scientific-method-for-performance)
3. [Measuring: latency, throughput, percentiles](#3-measuring-latency-throughput-percentiles)
4. [Building a load generator](#4-building-a-load-generator)
5. [Baseline: a healthy small catalog](#5-baseline-a-healthy-small-catalog)
6. [Experiment 1: 100,000 products](#6-experiment-1-100000-products)
7. [Experiment 2: many users at once](#7-experiment-2-many-users-at-once)
8. [Experiment 3: 500,000 products](#8-experiment-3-500000-products)
9. [Where does the memory go?](#9-where-does-the-memory-go)
10. [What the database says](#10-what-the-database-says)
11. [The real-world impact](#11-the-real-world-impact)
12. [Why every big service paginates](#12-why-every-big-service-paginates)
13. [Cleaning up](#13-cleaning-up)
14. [Common mistakes](#14-common-mistakes)
15. [Interview questions](#15-interview-questions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- The **measure → hypothesize → change → re-measure** loop, and why guessing about performance fails
- **Latency**, **throughput**, **concurrency**, and why **percentiles** (p95, p99) beat averages
- Writing a **load generator** with goroutines, a channel of jobs, a `sync.WaitGroup` and a mutex (a preview of Chapters 64-70)
- Measuring a process's **peak memory** on Linux (`VmHWM`) and a response's **size** and **time** (`curl -w`)
- How response size, server memory, and latency scale with rows, and how concurrency *multiplies* them
- Why one "harmless" endpoint can take a whole server down
- What to expect from the fix (a preview measured in the database)

---

## 2. The scientific method for performance

Performance intuition is unreliable, even for experts. The reliable approach is the same as in science:

```
1. Observe      what's slow / big / expensive?           (measure it)
2. Hypothesize  why? ("returning all rows is the cost")
3. Experiment   change ONE thing, or scale ONE thing
4. Measure      again, the same way
5. Decide       is the change worth it?
```

This chapter is steps 1-4 for a problem we *suspect*: `GET /products` returns the whole table. The current code (Chapter 61) is:

```go
products, err := h.svc.List(r.Context())   // ALL rows, into memory
...
out := make([]productResponse, 0, len(products))   // a second copy
util.SendData(w, http.StatusOK, out)               // JSON-encoded: a third copy
```

With 3 sample products it is perfect. Is it still fine with 100,000? A million? We will find out by **experiment**, not opinion.

> **Rules for honest experiments:** change one variable at a time; repeat measurements (numbers vary); note the environment (all numbers here come from a 12-core Linux laptop with the app and PostgreSQL on the same machine, so *network* time is absent: on a real network everything below gets *worse*); and remember that the **shape** of the results (how numbers grow) matters more than the exact values on your machine.

---

## 3. Measuring: latency, throughput, percentiles

| Term | Meaning | Unit |
|------|---------|------|
| **Latency** | time for *one* request from send to complete response | ms |
| **Throughput** | requests completed per second | req/s |
| **Concurrency** | requests in flight at the same moment | count |
| **Payload size** | bytes downloaded per response | KB/MB |

**Why not just the average latency?** Suppose 99 requests take 10 ms and one takes 5 seconds. The mean is ≈ 60 ms: a number that describes *nobody's* experience. **Percentiles** describe the distribution:

| Percentile | Meaning |
|------------|---------|
| **p50** (median) | half of requests were faster than this |
| **p95** | 95% were faster; 1 in 20 users waited at least this long |
| **p99** | 1 in 100 waited at least this long |
| **max** | the worst case |

Service-level objectives are usually written in percentiles ("p95 under 300 ms"), because the *slow tail* is where users leave and where timeouts fire.

Tools of the trade for load testing exist (`hey`, `wrk`, `ab`, `vegeta`, `k6`); we'll write our own tiny one. That avoids installing anything, and building it teaches worker pools, which we need to understand anyway.

---

## 4. Building a load generator

The design, as a picture:

```
 main goroutine                  worker goroutines (C of them)
 ┌─────────────┐   jobs chan   ┌────────────┐
 │ for i := 1..N│ ───────────► │ worker 1   │──► send request, time it ─┐
 │   jobs <- i  │ ───────────► │ worker 2   │──► ...                    ├─► results (mutex-protected)
 │ close(jobs)  │ ───────────► │ ...        │                           ┘
 └─────────────┘               └────────────┘
         wg.Wait() until every worker has drained the channel
```

Workers pull job numbers from an unbuffered channel until it is closed; a `sync.WaitGroup` waits for them all; a `sync.Mutex` protects the shared result while workers append to it. (You will study each of these pieces properly in Chapters 64-70; here they are simply the right tools.)

```go
// file: tools/loadgen/main.go
// Command loadgen is a tiny HTTP load generator: it sends N requests with C workers and reports
// throughput, latency percentiles, status codes and bytes transferred.
//
//	go run ./tools/loadgen -n 200 -c 20 http://localhost:18080/products
package main

import (
	"context"
	"flag"
	"fmt"
	"io"
	"net/http"
	"os"
	"sort"
	"strconv"
	"strings"
	"sync"
	"time"
)

// config describes one load-test run.
type config struct {
	url         string
	method      string
	body        string   // "{i}" is replaced by the request number, so every request can differ
	headers     []string // "Name: value"
	requests    int
	concurrency int
}

// result is what a run measured.
type result struct {
	elapsed   time.Duration
	latencies []time.Duration // one per completed request (failed ones included)
	status    map[int]int     // HTTP status -> count
	failures  int             // requests that produced no HTTP response at all
	bytes     int64           // response body bytes received
}

// run sends cfg.requests requests using cfg.concurrency workers and measures them.
func run(ctx context.Context, client *http.Client, cfg config) result {
	jobs := make(chan int)
	var (
		mu  sync.Mutex
		res = result{status: map[int]int{}}
		wg  sync.WaitGroup
	)

	start := time.Now()
	for w := 0; w < cfg.concurrency; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for i := range jobs {
				began := time.Now()
				code, n, err := send(ctx, client, cfg, i)
				took := time.Since(began)

				mu.Lock()
				res.latencies = append(res.latencies, took)
				if err != nil {
					res.failures++
				} else {
					res.status[code]++
					res.bytes += n
				}
				mu.Unlock()
			}
		}()
	}
	for i := 1; i <= cfg.requests; i++ {
		jobs <- i
	}
	close(jobs)
	wg.Wait()

	res.elapsed = time.Since(start)
	return res
}

// send performs one request and returns the status code and the number of body bytes read.
func send(ctx context.Context, client *http.Client, cfg config, i int) (int, int64, error) {
	body := strings.ReplaceAll(cfg.body, "{i}", strconv.Itoa(i))
	req, err := http.NewRequestWithContext(ctx, cfg.method, cfg.url, strings.NewReader(body))
	if err != nil {
		return 0, 0, err
	}
	for _, h := range cfg.headers {
		if name, value, ok := strings.Cut(h, ":"); ok {
			req.Header.Set(strings.TrimSpace(name), strings.TrimSpace(value))
		}
	}

	resp, err := client.Do(req)
	if err != nil {
		return 0, 0, err
	}
	defer resp.Body.Close()
	n, err := io.Copy(io.Discard, resp.Body) // read everything: the download is part of the cost
	return resp.StatusCode, n, err
}

// percentile returns the p-th percentile (0-100) of sorted durations using the nearest-rank method.
func percentile(sorted []time.Duration, p float64) time.Duration {
	if len(sorted) == 0 {
		return 0
	}
	rank := int(p/100*float64(len(sorted)) + 0.999999) // ceil
	if rank < 1 {
		rank = 1
	}
	if rank > len(sorted) {
		rank = len(sorted)
	}
	return sorted[rank-1]
}

// report prints a summary of r.
func (r result) report(w io.Writer) {
	sorted := append([]time.Duration(nil), r.latencies...)
	sort.Slice(sorted, func(i, j int) bool { return sorted[i] < sorted[j] })

	var total time.Duration
	for _, d := range sorted {
		total += d
	}
	var mean time.Duration
	if len(sorted) > 0 {
		mean = total / time.Duration(len(sorted))
	}

	fmt.Fprintf(w, "requests:     %d in %s (%.1f req/s)\n", len(sorted), r.elapsed.Round(time.Millisecond), float64(len(sorted))/r.elapsed.Seconds())
	fmt.Fprintf(w, "latency:      mean %s | p50 %s | p95 %s | p99 %s | max %s\n",
		mean.Round(time.Microsecond), percentile(sorted, 50).Round(time.Microsecond), percentile(sorted, 95).Round(time.Microsecond),
		percentile(sorted, 99).Round(time.Microsecond), percentile(sorted, 100).Round(time.Microsecond))

	codes := make([]int, 0, len(r.status))
	for code := range r.status {
		codes = append(codes, code)
	}
	sort.Ints(codes)
	for _, code := range codes {
		fmt.Fprintf(w, "status %d:   %d\n", code, r.status[code])
	}
	if r.failures > 0 {
		fmt.Fprintf(w, "failures:     %d (no response)\n", r.failures)
	}
	fmt.Fprintf(w, "downloaded:   %.2f MB total, %.1f KB per successful request\n",
		float64(r.bytes)/1e6, float64(r.bytes)/1e3/float64(max(1, successes(r.status))))
}

func successes(status map[int]int) int {
	n := 0
	for _, c := range status {
		n += c
	}
	return n
}

// headerFlag lets -H be given several times.
type headerFlag []string

func (h *headerFlag) String() string     { return strings.Join(*h, "; ") }
func (h *headerFlag) Set(v string) error { *h = append(*h, v); return nil }

func main() {
	var cfg config
	var headers headerFlag
	flag.StringVar(&cfg.method, "X", "GET", "HTTP method")
	flag.StringVar(&cfg.body, "d", "", `request body ("{i}" is replaced by the request number)`)
	flag.Var(&headers, "H", `extra header "Name: value" (repeatable)`)
	flag.IntVar(&cfg.requests, "n", 100, "total number of requests")
	flag.IntVar(&cfg.concurrency, "c", 10, "number of concurrent workers")
	timeout := flag.Duration("timeout", 60*time.Second, "per-request timeout")
	flag.Usage = func() {
		fmt.Fprintln(os.Stderr, "usage: loadgen [flags] URL")
		flag.PrintDefaults()
	}
	flag.Parse()

	if flag.NArg() != 1 || cfg.requests < 1 || cfg.concurrency < 1 {
		flag.Usage()
		os.Exit(2)
	}
	cfg.url = flag.Arg(0)
	if !strings.Contains(cfg.url, "://") {
		cfg.url = "http://" + cfg.url // allow "localhost:8080/products"
	}
	cfg.headers = headers
	cfg.concurrency = min(cfg.concurrency, cfg.requests)

	client := &http.Client{Timeout: *timeout}
	run(context.Background(), client, cfg).report(os.Stdout)
}
```

Key details:

- **`{i}` in the body** is replaced by the request number, so each `POST` can create a *different* product.
- **`io.Copy(io.Discard, resp.Body)`** reads the *entire* response and throws it away: the download is part of the cost, and not reading the body would also stop the HTTP client from reusing connections.
- **Latencies are recorded even for failed requests**, and failures (no HTTP response at all) are counted separately from non-2xx statuses.
- **`percentile`** uses the *nearest-rank* method: the smallest value with at least p% of samples at or below it.
- `cfg.concurrency = min(cfg.concurrency, cfg.requests)` uses the `min` builtin (Go 1.21+).

Test it *before* trusting its numbers: a measuring instrument that lies is worse than none. These tests use `httptest` servers whose behavior we control:

```go
// file: tools/loadgen/main_test.go
package main

import (
	"bytes"
	"context"
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

func TestRunSendsEveryRequestAndCountsStatuses(t *testing.T) {
	var hits atomic.Int64
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		n := hits.Add(1)
		if n%4 == 0 {
			http.Error(w, "boom", http.StatusInternalServerError)
			return
		}
		io.WriteString(w, "0123456789") // 10 bytes
	}))
	defer srv.Close()

	res := run(context.Background(), srv.Client(), config{url: srv.URL, method: "GET", requests: 20, concurrency: 4})

	if hits.Load() != 20 || len(res.latencies) != 20 {
		t.Errorf("hits=%d latencies=%d, want 20 each", hits.Load(), len(res.latencies))
	}
	if res.status[200] != 15 || res.status[500] != 5 {
		t.Errorf("status counts = %v, want 15x200 and 5x500", res.status)
	}
	if res.failures != 0 {
		t.Errorf("failures = %d", res.failures)
	}
	if want := int64(15*10 + 5*len("boom\n")); res.bytes != want {
		t.Errorf("bytes = %d, want %d", res.bytes, want)
	}
}

func TestConcurrencyIsBoundedByTheWorkerCount(t *testing.T) {
	var inFlight, peak atomic.Int64
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		now := inFlight.Add(1)
		for {
			old := peak.Load()
			if now <= old || peak.CompareAndSwap(old, now) {
				break
			}
		}
		time.Sleep(20 * time.Millisecond)
		inFlight.Add(-1)
	}))
	defer srv.Close()

	run(context.Background(), srv.Client(), config{url: srv.URL, method: "GET", requests: 40, concurrency: 5})

	if p := peak.Load(); p > 5 || p < 2 {
		t.Errorf("peak concurrency = %d, want between 2 and 5", p)
	}
}

func TestBodyTemplateAndHeaders(t *testing.T) {
	var mu sync.Mutex
	var bodies []string
	var auth string
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		b, _ := io.ReadAll(r.Body)
		mu.Lock()
		bodies = append(bodies, string(b))
		auth = r.Header.Get("Authorization")
		mu.Unlock()
	}))
	defer srv.Close()

	run(context.Background(), srv.Client(), config{
		url: srv.URL, method: "POST", body: `{"n":{i}}`, requests: 3, concurrency: 1,
		headers: []string{"Authorization: Bearer abc", "X-Other:  v "},
	})

	if strings.Join(bodies, ",") != `{"n":1},{"n":2},{"n":3}` {
		t.Errorf("bodies = %v", bodies)
	}
	if auth != "Bearer abc" {
		t.Errorf("Authorization = %q", auth)
	}
}

func TestUnreachableServersAreCountedAsFailures(t *testing.T) {
	srv := httptest.NewServer(http.NotFoundHandler())
	url := srv.URL
	srv.Close() // nothing listens any more

	res := run(context.Background(), http.DefaultClient, config{url: url, method: "GET", requests: 5, concurrency: 2})
	if res.failures != 5 || len(res.status) != 0 {
		t.Errorf("failures=%d status=%v", res.failures, res.status)
	}
}

func TestPercentiles(t *testing.T) {
	var d []time.Duration
	for i := 1; i <= 100; i++ {
		d = append(d, time.Duration(i)*time.Millisecond)
	}
	for p, want := range map[float64]time.Duration{50: 50 * time.Millisecond, 95: 95 * time.Millisecond, 99: 99 * time.Millisecond, 100: 100 * time.Millisecond} {
		if got := percentile(d, p); got != want {
			t.Errorf("p%v = %v, want %v", p, got, want)
		}
	}
	if percentile(nil, 50) != 0 {
		t.Error("an empty sample has no percentiles")
	}
	if got := percentile([]time.Duration{7 * time.Millisecond}, 99); got != 7*time.Millisecond {
		t.Errorf("a single sample is every percentile, got %v", got)
	}
}

func TestReportMentionsTheImportantNumbers(t *testing.T) {
	res := result{
		elapsed:   2 * time.Second,
		latencies: []time.Duration{10 * time.Millisecond, 30 * time.Millisecond},
		status:    map[int]int{200: 2},
		bytes:     2_000_000,
	}
	var out bytes.Buffer
	res.report(&out)
	for _, want := range []string{"requests:     2 in 2s", "p50 10ms", "p99 30ms", "status 200:   2", "2.00 MB", "1000.0 KB per successful request"} {
		if !strings.Contains(out.String(), want) {
			t.Errorf("report should contain %q, got:\n%s", want, out.String())
		}
	}
}
```

`TestConcurrencyIsBoundedByTheWorkerCount` records the **peak number of requests in flight** on the server and asserts it never exceeds the number of workers: a direct check that "-c 5" means five.

---

## 5. Baseline: a healthy small catalog

Start from a fresh database and the server (Chapter 58's `-auto-migrate` builds the schema), register a user, and log in for a token. Then let the load generator create **2,000 products through the real API** (`POST`, authenticated):

```bash
go build -o loadgen ./tools/loadgen
TOKEN=...   # from POST /login
./loadgen -n 2000 -c 20 -X POST -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"Product {i}","description":"A product created by the load generator","price":9.99,"imageUrl":"https://example.com/img/{i}.jpg"}' \
  localhost:18080/products
```
```
requests:     2000 in 255ms (7854.3 req/s)
latency:      mean 2.521ms | p50 2.154ms | p95 4.458ms | p99 11.978ms | max 17.741ms
status 201:   2000
downloaded:   0.30 MB total, 0.1 KB per successful request
```

2,000 authenticated writes with 20 concurrent workers: about **7,900 requests per second**, p99 of 12 ms. Now *reads*, first of a **single product**, then of the **whole list** (2,000 rows):

```bash
./loadgen -n 2000 -c 20 localhost:18080/products/1
./loadgen -n 100  -c 10 localhost:18080/products
```
```
GET /products/1  (2,000 requests)
requests:     2000 in 224ms (8930.7 req/s)
latency:      mean 2.191ms | p50 1.69ms | p95 4.546ms | p99 13.016ms | max 20.768ms
downloaded:   0.29 MB total, 0.1 KB per successful request

GET /products  (100 requests, 2,000 rows each)
requests:     100 in 131ms (763.6 req/s)
latency:      mean 12.707ms | p50 11.99ms | p95 20.301ms | p99 24.653ms | max 26.527ms
downloaded:   29.67 MB total, 296.7 KB per successful request
```

Already visible with only 2,000 rows: the list endpoint is **about 12× slower** (763 vs 8,930 req/s) and moves **2,000× more bytes** per request (297 KB vs 0.15 KB). Everything is still "fast" in absolute terms: nobody would notice at this scale. That is exactly why the problem stays hidden until it hurts.

---

## 6. Experiment 1: 100,000 products

Creating 98,000 more products through HTTP would take a while; the database can do it in one statement with `generate_series` (a set-returning function that produces a sequence of numbers):

```sql
INSERT INTO products (title, description, price, image_url)
SELECT 'Product ' || g,
       'A product created by the bulk seed script, number ' || g,
       (g % 1000) + 1,
       'https://example.com/img/' || g || '.jpg'
FROM generate_series(1, 98000) g;
```
```
INSERT 0 98000
 count  | pg_size_pretty
--------+----------------
 100000 | 19 MB
```

The table is only **19 MB** on disk, a small database by any standard. Now restart the server (so its memory starts clean) and fetch the list once. `curl -w` prints timing and size, and Linux exposes a process's memory in `/proc/<pid>/status`: **`VmRSS`** is the memory currently resident, **`VmHWM`** is the *peak* ("high water mark"):

```bash
grep -E "VmRSS|VmHWM" /proc/$(pgrep -f 'app serve')/status        # before
curl -s -o /dev/null -w "status=%{http_code} time=%{time_total}s size=%{size_download} bytes\n" localhost:18080/products
grep -E "VmRSS|VmHWM" /proc/$(pgrep -f 'app serve')/status        # after
```
```
before: VmHWM: 12632 kB  VmRSS: 12632 kB
status=200 time=0.163643s size=16708879 bytes
after:  VmHWM: 80284 kB  VmRSS: 80284 kB
```

Read the numbers:

| Measurement | 1 product | 100,000 products |
|-------------|-----------|------------------|
| Response size | 166 bytes | **16.7 MB** |
| Time (curl, same machine, no network) | 2.6 ms | **164 ms** |
| Server memory (peak) | 13 MB | **80 MB** (+67 MB for one request) |

For *one user on localhost*, 164 ms feels fine. But look at what a single request costs: a 16.7 MB download and 67 MB of server memory, to serve a page that a human would scroll through for a few dozen rows at most.

---

## 7. Experiment 2: many users at once

A server exists to handle *many* users. Restart it (clean memory) and send **10 simultaneous** list requests:

```bash
./loadgen -n 10 -c 10 localhost:18080/products
```
```
requests:     10 in 580ms (17.2 req/s)
latency:      mean 551.549ms | p50 540.957ms | p95 580.159ms | p99 580.159ms | max 580.159ms
status 200:   10
downloaded:   167.09 MB total, 16708.9 KB per successful request
mem: VmHWM: 624192 kB  VmRSS: 611412 kB
```

Ten users cost:

- latency **3.4× worse** (551 ms mean vs 164 ms for a lone request): they compete for CPU and memory bandwidth,
- **167 MB** transferred in half a second,
- server memory peak **624 MB**, nearly **8× the single-request peak**: ten requests each hold their own full copy of the result.

Memory grows **linearly with concurrency**: `peak ≈ concurrent users × per-request memory`. That's the part that kills servers: memory is finite, and when it runs out the operating system's **OOM killer** terminates the process. Every user then gets an error, not just the heavy ones.

Control experiment, so we know the database size isn't the problem by itself: fetch a *single* product from the same 100,000-row table:

```
status=200 time=0.002634s size=166        (server peak: 13 MB, unchanged)
```

**2.6 ms** on the 100,000-row table, same as when it had 3 rows. Fetching one row by ID is *independent of table size* (thanks to the primary-key index). It's specifically "return everything" that scales with the data.

---

## 8. Experiment 3: 500,000 products

Grow the table 5×, then repeat:

```
 500000 | 94 MB

one request:        status=200 time=0.79s  size=85,132,764 bytes (85 MB)   server peak: 454 MB
5 concurrent:       mean 1.60s, max 1.71s, 425 MB downloaded              server peak: 1,980 MB (≈ 2 GB)
```

Putting the three experiments side by side (one request each):

| Rows | Response | Time | Server peak memory | Ratio: memory / response |
|-----:|---------:|-----:|-------------------:|-------------------------:|
| 3 | ~0.4 KB | ~2 ms | 13 MB | (baseline) |
| 100,000 | 16.7 MB | 0.16 s | 80 MB | 4.8× |
| 500,000 | 85 MB | 0.79 s | 454 MB | 5.3× |

**Everything scales linearly with the number of rows:** 5× the rows gives about 5× the bytes, the time, and the memory. That's what "O(n)" means in practice. And the concurrent cases show the second multiplication:

```
memory needed ≈ rows × bytes-per-row × copies-per-request × concurrent-requests
```

Extrapolating (this is arithmetic, not a measurement): a million products (≈ 170 MB per response, ≈ 900 MB of server memory per request) with 20 users at once wants **~18 GB**: more than many servers own. The endpoint that passed every test at launch becomes an outage the day the catalog grows.

---

## 9. Where does the memory go?

Why does one 85 MB response need 454 MB of memory? Follow the data through the layers (Chapters 60-61):

```
PostgreSQL rows ──► []productRow (500k structs, each string separately allocated)     ← copy 1
                ──► []product.Product  (row.toProduct() for each)                      ← copy 2
                ──► []productResponse  (toResponse() for each)                         ← copy 3
                ──► json.Encoder builds the full JSON text before writing (85 MB)     ← copy 4
```

Each layer *maps* the data into its own shape (the "mapping cost" from Chapter 59, section 11), and for one product that cost is invisible. For 500,000 it means holding four copies simultaneously, and Go's garbage collector cannot free any of them until the request finishes. (Go's GC also lets the heap grow to roughly twice the live data before collecting; that explains part of the gap between "live data" and "peak RSS".)

Two things follow:

1. **You can tune constants**, streaming the JSON instead of buffering it, avoiding copies, but the cost still grows with the number of rows. Optimizing *the constant* buys a factor; you need to change the **shape** of the problem to remove the growth.
2. **The right fix isn't in Go at all**: don't load 500,000 rows in the first place.

---

## 10. What the database says

Ask PostgreSQL directly how much *its* work differs between "everything" and "one page of 20" (`\timing on` in `psql`):

```sql
-- all rows (counting them, so nothing has to be sent to the client)
SELECT count(*) FROM (SELECT id, title, description, price, image_url FROM products ORDER BY id) t;
-- one page
SELECT count(*) FROM (SELECT id, title, description, price, image_url FROM products ORDER BY id LIMIT 20) t;
-- page 12,501 of size 20 (skipping 250,000 rows)
SELECT count(*) FROM (SELECT id, title, description, price, image_url FROM products ORDER BY id LIMIT 20 OFFSET 250000) t;
```
```
all rows:                     Time: 39.853 ms
first page (LIMIT 20):        Time:  0.377 ms
deep page (OFFSET 250000):    Time: 14.779 ms
```

A first page is **~100× cheaper** than the full scan *inside the database alone*, before counting any transfer, mapping, or JSON. (These timings measure the database's work; they don't include shipping 85 MB to the application.) Notice the third line: a **deep page** using `OFFSET` costs 39× more than the first page, because PostgreSQL still has to walk past the 250,000 skipped rows. That's a preview of a subtlety we'll deal with in Chapter 63 (**keyset pagination**).

---

## 11. The real-world impact

Translate the measurements into consequences for a real product:

**Mobile users.** A phone on a typical mobile connection might sustain 10 Mbit/s (1.25 MB/s). The 100,000-row response (16.7 MB) takes **13 seconds** just to download; the 500,000-row one (85 MB) takes **68 seconds**, and might cost a user real money on a capped data plan. Parsing 85 MB of JSON on a low-end phone can exhaust the app's memory: apps get killed by the OS long before the download finishes.

**Cost.** Cloud providers charge for outbound data transfer (order of magnitude: cents per GB; check your provider's price list). If the home page fetches 85 MB and you have 100,000 daily visitors, that's 8.5 **terabytes** per day, all to display twenty products. Servers must also be sized for the *peak memory* above, so you pay for machines that spend most of their capacity holding copies of data nobody looks at.

**Reliability.** Memory scales with concurrent requests, so a modest traffic spike or a single misbehaving client (a script polling the endpoint in a loop, a crawler) can exhaust memory and take the service down for everyone. An endpoint whose cost is *unbounded by design* is a **denial-of-service vulnerability**, even without malicious intent.

**User experience.** Nobody reads 500,000 products. The user wants "the first few, sorted sensibly, and a way to see more". Large sites behave that way: a video site shows a screenful of videos and loads more as you scroll; a social feed loads a handful of posts at a time; a search engine shows ten results per page. They aren't doing it *because it's tidy*, but because returning everything is physically impossible at their scale.

---

## 12. Why every big service paginates

The experiment reveals the design rule:

> **Every endpoint that returns a collection must have a hard upper bound on the size of its response**, chosen by the server, no matter how large the underlying data is.

That's **pagination**: return the data in bounded *pages*, and let the client ask for the next page. What this buys, measured above:

| Property | Without pagination | With pagination (page size 20) |
|----------|-------------------|--------------------------------|
| Response size | grows with the table (16.7 MB → 85 MB → …) | constant (~3.3 KB) |
| Server memory per request | grows with the table | constant |
| Latency | grows with the table | constant (first pages) |
| Database work | full scan | index-assisted, bounded |
| Behavior under load | memory ∝ users × rows | memory ∝ users × page size |
| Client experience | long wait, huge parse | instant first screen |

In [Chapter 63](63-pagination.md) we implement it: query parameters (`?page=2&limit=20`), an offset calculation, a `Count` to report totals, response metadata, and a hard maximum page size the client can't exceed.

---

## 13. Cleaning up

Experiments leave debris. Reset the data (this deletes everything in `products` and restarts the ID counter):

```bash
docker exec shop-db psql -U postgres -d ecommerce -c "TRUNCATE products RESTART IDENTITY"
```

The 500,000 rows also grew the table's files on disk; `TRUNCATE` reclaims that space immediately (a `DELETE` would leave it for `VACUUM`).

---

## 14. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Testing only with a handful of rows | Problems appear in production, at 100× the data | Test with production-scale data |
| 2 | Judging by average latency | Hides the slow tail | Look at p95/p99/max |
| 3 | Measuring once | Noise looks like signal | Repeat; compare medians |
| 4 | Changing several things between measurements | Can't tell what helped | One change at a time |
| 5 | Load-testing your laptop's *localhost* and trusting absolute numbers | No network, no TLS, different hardware | Trust ratios and growth curves; re-test on a realistic setup |
| 6 | Generating the load generator's requests sequentially | Never exposes concurrency effects | Use concurrent workers |
| 7 | Not draining the response body | Inflated speed, broken connection reuse | Read (or discard) the whole body |
| 8 | Optimizing constants (streaming, fewer copies) instead of the algorithm | 2× better, still O(n) | Bound the result size |
| 9 | Unbounded "list all" endpoints | Memory, cost, and DoS risk | Server-enforced maximum page size |
| 10 | Load-testing production without warning | Real outage | Use staging, or a controlled window |
| 11 | Ignoring memory (looking only at time) | OOM kills in production | Track peak memory (`VmHWM`, container limits) |
| 12 | Forgetting to clean up experiment data | Slow dev database, confusing later tests | `TRUNCATE` / drop the scratch schema |

---

## 15. Interview questions

**Q1. Why are percentiles better than averages for latency?**
Averages hide the tail; p95/p99 show what the slowest users experience, which drives timeouts and complaints.

**Q2. What is the difference between throughput and latency?**
Latency is the time for one request; throughput is requests completed per second. Under load, both change: higher concurrency can raise throughput while worsening latency.

**Q3. How does an unbounded list endpoint fail?**
Response size, latency, and memory grow with the data, and memory also grows with concurrent requests, until the process is killed for lack of memory.

**Q4. How would you measure a server's memory use on Linux?**
`/proc/<pid>/status` (`VmRSS` current, `VmHWM` peak), `ps -o rss`, or the container/cgroup metrics; in Go, also `runtime.ReadMemStats` or pprof.

**Q5. Why is fetching one row by primary key fast regardless of table size?**
The primary-key index (a B-tree) finds it in O(log n) steps; the cost doesn't depend on the number of rows in practice.

**Q6. Why is `OFFSET 250000` slower than `LIMIT 20` from the start?**
The database must still read and discard the skipped rows; the cost grows with the offset.

**Q7. What's wrong with optimizing the JSON encoder to fix the 500,000-row problem?**
It reduces constants but not the growth; the endpoint remains O(n) in memory/time/bytes and still fails eventually.

**Q8. How does the load generator make sure concurrency really equals the requested number?**
A fixed number of worker goroutines read jobs from a channel; a test on the server side verifies the peak in-flight requests never exceeds it.

**Q9. Why include failed requests' latency in the results?**
Slow failures (timeouts) are part of user experience; dropping them makes the numbers look better than reality.

---

## 16. Exercises

### Exercise 1: Reproduce the growth curve
Run the experiments with 10,000, 50,000 and 200,000 rows. Plot (or tabulate) rows vs. response size vs. time vs. peak memory. Is each relationship linear?

<details><summary>Solution</summary>

Response size and memory should be very nearly linear in the row count (≈ 167 bytes per row in the JSON, ≈ 700-900 bytes of peak memory per row for one request in our measurements). Time is close to linear but has a fixed floor at small sizes and may bend upward when the garbage collector or memory pressure kicks in at large sizes.
</details>

### Exercise 2: Find your server's breaking point
Use `loadgen` with increasing concurrency (`-c 1, 2, 5, 10, 20, 50`) against `GET /products` with 100,000 rows. Where does p95 latency cross 1 second? Watch memory with `VmHWM`. (Caution: keep concurrency low enough that your machine doesn't swap.)

<details><summary>Solution</summary>

Expect p95 to rise roughly linearly with concurrency once CPU is saturated (a queueing effect: with all cores busy each extra concurrent request adds its full service time to everyone's latency), and memory to rise ≈ 55-60 MB per additional concurrent request. The point is not your exact numbers but that both grow without bound.
</details>

### Exercise 3: Improve the generator
Add a `-duration 10s` mode that sends requests until time is up (instead of a fixed `-n`), and print a **histogram** of latencies in buckets (<1ms, <5ms, <10ms, <50ms, <100ms, <500ms, ≥500ms). Add tests.

<details><summary>Solution</summary>

Use a `context.WithTimeout` for the run: workers loop `for ctx.Err() == nil { send… }` instead of ranging over a job channel. For the histogram, keep a `[]time.Duration` of bucket upper bounds and count with a `sort.Search` per latency. Test with an `httptest` server whose handler sleeps for a chosen duration and assert the bucket counts.
</details>

### Exercise 4: Stream instead of buffer
Change `util.SendData` for lists to stream JSON with `json.NewEncoder(w).Encode` per element (writing `[`, elements separated by `,`, `]`). How much does peak memory drop for the 500,000-row case? Does the problem go away?

<details><summary>Solution</summary>

You remove the buffered 85 MB text (one of the four copies), so peak memory drops by roughly a quarter; the three row/entity/response slices remain unless you also stream from the database cursor (`QueryxContext` + `rows.Next()`, Chapter 56 §8). Even fully streamed, the *response is still 85 MB*, the download is still 68 seconds on a phone, and the client still can't use it: streaming reduces memory but not the fundamental cost.
</details>

### Exercise 5: Look at it with pprof
Add `net/http/pprof` to a *debug-only* listener, fetch the 100,000-row list, and inspect the heap profile (`go tool pprof -sample_index=alloc_space`). Which functions allocate most?

<details><summary>Solution</summary>

Expect the row scanning (`sqlx` / `database/sql` string allocations), the mapping loops (`toProduct`, `toResponse`), and `encoding/json` encoding to dominate. Never expose pprof on the public port: bind it to `127.0.0.1` or behind authentication.
</details>

### Exercise 6 (challenge): A budget
Your service must serve 1,000 concurrent list requests within a 4 GB memory budget. Using the measured ≈ 400 MB per request at 500,000 rows, compute the maximum page size (in rows) that fits, assuming per-row memory stays proportional.

<details><summary>Solution</summary>

4 GB / 1,000 = 4 MB per request. At ≈ 800 bytes of peak memory per row (400 MB / 500,000), that's about **5,000 rows** per request at most, and far lower once you add headroom for the runtime, the database driver's buffers, and other endpoints. In practice you'd pick a page size like 20-100 and cap it at 100-200: three orders of magnitude of safety.
</details>

---

## 17. Quiz

1. Why is p99 latency more informative than the mean?
2. What happens to server memory when concurrency doubles for an unbounded list endpoint?
3. Why is `GET /products/{id}` fast even with 500,000 rows?
4. What did the 500,000-row experiment show about the ratio of response size to peak memory?
5. Why is "optimize the JSON encoder" not a real fix?
6. What does a hard maximum page size protect against?
7. Why should you read the entire response body in a load generator?
8. What's `VmHWM`?

<details><summary>Answers</summary>

1. It shows the slow tail that individual users experience and that triggers timeouts; means average it away.
2. It roughly doubles (each request holds its own full copy).
3. A primary-key index lookup touches only a few pages regardless of table size.
4. Peak memory was ≈ 5× the response size, because the data exists as several copies (rows, entities, response structs, JSON text).
5. It reduces constants but the cost still grows linearly with rows; only bounding the result changes the shape.
6. Unbounded memory/bandwidth per request: accidental or deliberate denial of service.
7. Downloading is part of the cost, and unread bodies prevent connection reuse and understate latency.
8. The peak resident memory ("high water mark") of a Linux process.
</details>

---

## 18. Summary

- Performance claims need **measurements**: change one variable, repeat, compare; trust growth curves more than absolute numbers from a laptop.
- We built and tested a **load generator** (worker goroutines + job channel + `WaitGroup` + mutex) reporting throughput, **p50/p95/p99/max**, status counts, and bytes.
- With `GET /products` returning everything: response size, latency, and server memory all grew **linearly with rows** (16.7 MB / 0.16 s / 80 MB at 100,000; 85 MB / 0.79 s / 454 MB at 500,000), and memory **multiplies with concurrent users** (624 MB for 10 users at 100,000 rows; ~2 GB for 5 users at 500,000).
- The data passes through **four copies** (rows → entities → response structs → JSON text): mapping costs are invisible for one row and dominant for half a million.
- In the database, a first page is **~100× cheaper** than the full scan, and a *deep* `OFFSET` page is much slower than a first page: a preview of the next chapter.
- The rule: **every collection endpoint needs a server-enforced upper bound on response size**.

### ➡️ What's next?

[Chapter 63](63-pagination.md) implements **pagination**: `?page=` and `?limit=` query parameters (with validation and a hard maximum), the offset calculation, a `Count` to report totals, response metadata, and a new domain rule, all through the layers we built in Chapters 60 and 61.
