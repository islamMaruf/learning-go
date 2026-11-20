# Chapter 38: OS or Go Server? The Complete Journey of a Request

> **Goal of this chapter:** Build your first real Go web server, then follow a single request **all the way down and back up**: from the browser, across the network, through the network card and the operating system's kernel, into your Go program, through the router to your handler, and back out again. You'll finally be able to answer the classic question: *"When a request arrives, who does the work: the OS or the Go server?"* (Answer: both, in layers, and you'll know exactly which layer does what.) We'll verify each step with real `strace` output.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 3 hours  **Prerequisite:** [Chapters 27, 36, 37](37-into-backend-development.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Build your first Go server](#2-build-your-first-go-server)
3. [Run it and test it](#3-run-it-and-test-it)
4. [Routes, routers, and handlers](#4-routes-routers-and-handlers)
5. [The journey: overview](#5-the-journey-overview)
6. [Step 1: the client makes a request](#6-step-1-the-client-makes-a-request)
7. [Step 2: the network card (NIC)](#7-step-2-the-network-card)
8. [Step 3: the kernel and the socket buffers](#8-step-3-the-kernel-and-the-socket)
9. [Step 4: how your Go program starts listening](#9-step-4-how-your-go-program-starts-listening)
10. [Step 5: how Go waits without wasting threads](#10-step-5-how-go-waits)
11. [Step 6: the router and your handler](#11-step-6-the-router-and-your-handler)
12. [Step 7: the response travels back](#12-step-7-the-response-travels-back)
13. [The complete flow in one picture](#13-the-complete-flow)
14. [Who does what?](#14-who-does-what)
15. [Time budget of a request](#15-time-budget-of-a-request)
16. [Troubleshooting: `address already in use`](#16-troubleshooting)
17. [Common misconceptions](#17-common-misconceptions)
18. [Exercises](#18-exercises)
19. [Quiz](#19-quiz)
20. [Summary](#20-summary)

---

## 1. What you will learn

- How to write, run, and test a minimal Go HTTP server (`net/http`)
- What a **route**, a **router (ServeMux)**, and a **handler** are
- What a **network interface card (NIC)** does when a request arrives
- How the **kernel** receives data, buffers it, and hands it to your process
- What system calls a Go server makes: `socket`, `bind`, `listen`, `accept`, `read`, `write`, `epoll`
- Why a Go server can handle thousands of connections with few threads
- How to diagnose "address already in use"

> ⚠️ **Why this chapter matters:** Many developers write handlers for years without knowing how the request *reaches* the handler. Understanding the whole path makes debugging (timeouts, connection limits, latency) far easier, and turns "magic" into engineering.

---

## 2. Build your first Go server

### Step 1: create a module (Chapter 10)

```bash
mkdir ecommerce && cd ecommerce
go mod init ecommerce
```

### Step 2: write `main.go`

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	// Create a router (called a "ServeMux" in Go)
	mux := http.NewServeMux()

	// Register routes: URL path → handler function
	mux.HandleFunc("/hello", helloHandler)
	mux.HandleFunc("/about", aboutHandler)

	fmt.Println("Server running on :8080")

	// Start listening. This call BLOCKS until the server stops or fails.
	if err := http.ListenAndServe(":8080", mux); err != nil {
		fmt.Println("Error starting server:", err)
	}
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello World")
}

func aboutHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "I am a Go developer")
	fmt.Fprintln(w, "I build backends")
}
```

Line by line:

| Code | Meaning |
|------|---------|
| `http.NewServeMux()` | Creates a **router**: a table mapping URL paths to handler functions |
| `mux.HandleFunc("/hello", helloHandler)` | **Registers a route**: "requests for `/hello` go to `helloHandler`" |
| `http.ListenAndServe(":8080", mux)` | Opens port **8080**, and loops forever accepting connections and dispatching each request to `mux` |
| `:8080` | "All network interfaces, port 8080" (`localhost:8080` would restrict to this machine) |
| `w http.ResponseWriter` | The handler's *pen*: what you write into `w` becomes the response body |
| `r *http.Request` | The handler's *input*: method, URL, headers, body of the incoming request |
| `fmt.Fprintln(w, ...)` | Like `Println`, but writes into any `io.Writer`; here, the HTTP response |

---

## 3. Run it and test it

```bash
go run .
```

```
Server running on :8080
```

The terminal now *blocks*: that's expected. The server is running. Open a **second terminal**:

```bash
curl -i http://localhost:8080/hello
```

Real output:

```
HTTP/1.1 200 OK
Date: Sat, 26 Sep 2026 05:36:38 GMT
Content-Length: 12
Content-Type: text/plain; charset=utf-8

Hello World
```

```bash
curl -i http://localhost:8080/about
```

```
HTTP/1.1 200 OK
Content-Length: 37
Content-Type: text/plain; charset=utf-8

I am a Go developer
I build backends
```

```bash
curl -i http://localhost:8080/nope
```

```
HTTP/1.1 404 Not Found
```

You can also open `http://localhost:8080/hello` in a browser. Stop the server with `Ctrl+C`.

Notice what you *didn't* write: no code to parse HTTP text, manage connections, set `Content-Length`, or detect the content type. `net/http` did all of it.

---

## 4. Routes, routers, and handlers

| Term | Meaning | In our code |
|------|---------|-------------|
| **Route** | A mapping from a request pattern (path, and optionally method) to code | `"/hello"` → `helloHandler` |
| **Router** (a.k.a. *multiplexer*, `ServeMux`) | The component that looks at an incoming request and picks the right handler | `mux` |
| **Handler** | The function that produces the response | `helloHandler` |

```
Request: GET /about
          │
          ▼
   ┌──────────────┐    "/hello" → helloHandler
   │   Router     │    "/about" → aboutHandler   ◄── match!
   │   (mux)      │
   └──────┬───────┘
          ▼
     aboutHandler(w, r)  → writes the response
```

### The handler signature

Every handler has the same shape:

```go
func(w http.ResponseWriter, r *http.Request)
```

That's a *function type* (Chapter 16): anything with this signature can be a handler. `mux.HandleFunc("/x", f)` takes such a function. (There's also the `http.Handler` *interface* with a `ServeHTTP` method, which we'll use for middleware in Chapters 45–46.)

### Multiple routes

```go
mux.HandleFunc("/hello", helloHandler)
mux.HandleFunc("/about", aboutHandler)
mux.HandleFunc("/products", productsHandler)
```

Pattern matching rules for the basic mux: `"/about"` matches exactly `/about`; a pattern ending in `/` (e.g. `"/static/"`) matches the whole subtree. Since **Go 1.22**, patterns can also include the **method** and **path parameters**: `"GET /products/{id}"`. That's the topic of Chapter 44.

### Reading the request (a peek)

```go
func echoHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Method:", r.Method)          // GET
	fmt.Fprintln(w, "Path:", r.URL.Path)          // /echo
	fmt.Fprintln(w, "Query:", r.URL.Query())      // map[name:[Asha]]
	fmt.Fprintln(w, "User-Agent:", r.UserAgent()) // curl/8.5.0
}
```

---

## 5. The journey: overview

Now the main event. When you run `curl http://localhost:8080/hello` (or click a link in a browser), what *actually* happens? Here's the whole path; the next sections zoom into each step.

```
 CLIENT SIDE                    NETWORK               SERVER MACHINE
┌────────────┐                                  ┌─────────────────────────────────────┐
│ 1 Browser/ │                                  │  ┌──────────────┐                   │
│   curl     │                                  │  │ 2 NIC        │ hardware          │
│  builds    │ ── packets over WiFi/cable ────► │  │ receive      │                   │
│  request   │    (router, switches, ISP ...)   │  │ buffer       │                   │
└────────────┘                                  │  └──────┬───────┘                   │
                                                │         │ interrupt                 │
                                                │  ┌──────▼───────┐                   │
                                                │  │ 3 KERNEL     │ OS                │
                                                │  │ TCP/IP stack │                   │
                                                │  │ socket buffer│                   │
                                                │  └──────┬───────┘                   │
                                                │         │ "data ready"              │
                                                │  ┌──────▼───────┐                   │
                                                │  │ 4-5 GO       │ your process      │
                                                │  │ runtime:     │                   │
                                                │  │ netpoller →  │                   │
                                                │  │ goroutine    │                   │
                                                │  ├──────────────┤                   │
                                                │  │ 6 net/http   │                   │
                                                │  │ parse → mux  │                   │
                                                │  │ → handler    │                   │
                                                │  └──────┬───────┘                   │
                                                │         │ 7 write response          │
                                                │         ▼ (back down the stack)     │
                                                └─────────────────────────────────────┘
```

---

## 6. Step 1: the client makes a request

1. You type the URL. The browser (or `curl`) parses it: host `localhost`, port `8080`, path `/hello`.
2. It resolves the host to an IP (for `localhost`: `127.0.0.1` or `::1`; for a real site, a **DNS** lookup, Chapter 37).
3. It opens a **TCP connection** to that IP and port: the three-way handshake (SYN → SYN-ACK → ACK), which sets up a reliable, ordered byte stream.
4. It writes the HTTP request text into the connection (real bytes, as seen by the server, from the trace later):

```
GET /hello HTTP/1.1\r\n
Host: localhost:8080\r\n
User-Agent: curl/8.5.0\r\n
Accept: */*\r\n
\r\n
```

That is **83 bytes**. The client's OS chops the stream into **packets**, adds TCP and IP headers (source/destination addresses and ports), and hands them to *its* network card.

**On the way:** for a remote server, packets cross your **WiFi adapter → home router → ISP → many routers → the server's data center → the server's network card**. Each router just forwards packets closer to the destination IP. For `localhost` the packets never leave your machine (the kernel loops them back), but they go through much of the same OS machinery, which is why local testing is realistic.

---

## 7. Step 2: the network card

### What is a NIC?

A **NIC** (**N**etwork **I**nterface **C**ard/Controller) is the **hardware** that connects a computer to a network: an Ethernet port, or a **WiFi adapter** (the WiFi chip in your laptop is a NIC). It converts between **electrical/radio signals** and **digital data**.

```
  Network cable / WiFi radio waves
             │
      ┌──────▼──────┐
      │     NIC     │   • has a unique hardware (MAC) address
      │  (hardware) │   • has its own small memory for incoming/outgoing frames
      └──────┬──────┘   • checks addresses, verifies checksums
             │  DMA (writes directly into RAM, without the CPU copying bytes)
             ▼
        System RAM
```

### What it does when a packet arrives

1. Receives the signal, reassembles a **frame**, checks it's addressed to this machine.
2. Uses **DMA** (Direct Memory Access) to copy the frame straight into a **receive buffer (ring buffer)** in RAM that the kernel has set aside for it, with no CPU involvement in the copy.
3. Raises a **hardware interrupt** to the CPU: *"data arrived!"*

Modern systems batch and poll to avoid one interrupt per packet under heavy load (NAPI), but the idea is the same.

---

## 8. Step 3: the kernel and the socket

The **interrupt** makes the CPU stop what it's doing (context switch, Chapter 30) and run the **kernel's network driver**. Now the **kernel's TCP/IP stack** takes over:

```
NIC receive ring buffer
        │  interrupt → driver
        ▼
  IP layer:   Is this for me? Which protocol? (TCP)
        ▼
  TCP layer:  Which CONNECTION does this belong to?
              (source IP+port, dest IP+port)
              Put bytes in order, acknowledge them, handle retransmission
        ▼
  SOCKET RECEIVE BUFFER  (a per-connection queue in kernel memory)
        │   ◄── "the bytes GET /hello HTTP/1.1... are now waiting here"
        ▼
  Wake up whoever is waiting for data on this socket
```

Key concepts:

- **Socket**: the kernel object representing one end of a network connection. Your program refers to it with a **file descriptor** (a small integer like `3` or `4`). "Everything is a file" in Unix: sockets are read and written like files.
- **Port → process mapping**: the kernel knows *which process* is listening on port `8080` (because your Go program registered with `bind` + `listen`).
- **TCP** guarantees delivery and order; the **kernel** does all this work: retransmissions, windows, congestion control. Your Go code never sees a lost packet.
- The **receive buffer** decouples the network's timing from your program's: data can arrive while your program is busy, and waits in the buffer until it's ready to `read` it.

**This is the OS's job.** Your Go program has not been involved yet.

---

## 9. Step 4: how your Go program starts listening

Before any request can arrive, your server must have told the kernel: *"I'm listening on port 8080."* That's what `http.ListenAndServe(":8080", mux)` does first. Here are the **real system calls** from tracing our server with `strace` (on Linux):

```bash
strace -f -e trace=socket,bind,listen,accept4,read,write,epoll_ctl -o trace.txt ./server
```

Real (abbreviated) output:

```
socket(AF_INET6, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET6, sin6_port=htons(18081), ... "::" ...}, 28) = 0
listen(3, 4096)                                     = 0
```

| System call | What it does |
|-------------|--------------|
| `socket(...)` | Ask the kernel to create a **socket** (TCP, non-blocking). Returns a file descriptor: `3` |
| `bind(3, ..., port 18081)` | **Attach the socket to a port** (8080 in our example; 18081 in the traced run). If another program already holds the port: `bind: address already in use` (section 16) |
| `listen(3, 4096)` | Mark it as a **listening** socket; the kernel will now complete TCP handshakes and queue up to 4096 pending connections |

After that, the process sits in an **accept loop**, conceptually:

```go
// what http.Server.Serve does, simplified
for {
	conn, err := listener.Accept() // wait for the next connection
	if err != nil { ... }
	go handleConnection(conn)      // one goroutine per connection!  ← Chapter 36
}
```

Each accepted connection gets its own **goroutine**, which reads requests from that connection, runs handlers, and writes responses. That's the whole concurrency model of `net/http`.

---

## 10. Step 5: how Go waits

A server spends almost all its time **waiting**: for connections, for requests, for slow clients. If waiting blocked an OS thread each time, 10,000 idle connections would need 10,000 threads (the C10K problem, Chapter 32). Go avoids it with the **netpoller**.

The trace shows it in action. Real output:

```
accept4(3, ..., SOCK_CLOEXEC|SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable)
```

`accept4` was called on a **non-blocking** socket, and there was **no connection yet**, so the kernel returned **`EAGAIN`** ("try again later") *instead of blocking*. Go's runtime then:

1. **Parks the goroutine** that called `Accept` (Chapter 36).
2. Registers the socket with the kernel's event mechanism, **`epoll`** (`kqueue` on macOS, IOCP on Windows):

```
epoll_ctl(5, EPOLL_CTL_ADD, 4, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, ...}) = 0
```

3. Lets the OS thread run **other goroutines**. When the kernel says "socket 3 has a connection / socket 4 has data" (via `epoll_wait`), the runtime **wakes** the corresponding goroutine.

Then, when a client connects and sends its request, the trace shows:

```
accept4(3, {sa_family=AF_INET6, sin6_port=htons(43566), ... "::1" ...}, ...) = 4
read(4, "GET /hello HTTP/1.1\r\nHost: local"..., 4096) = 83
```

- `accept4` returns a **new file descriptor `4`** for this specific client connection (the client's ephemeral port is `43566`).
- `read(4, ...)` copies the **83 bytes** that the kernel had buffered in the **socket receive buffer** into your process's memory, exactly the request text from Step 1.

So the answer to *"how does the Go process know a request has arrived?"*: **it doesn't poll or block a thread.** The kernel notifies the Go runtime through `epoll`; the runtime resumes the goroutine that was waiting on that socket.

---

## 11. Step 6: the router and your handler

Now we're back in Go code. The connection's goroutine (inside `net/http`) does:

1. **Parse** the bytes: `GET /hello HTTP/1.1` + headers → build an `*http.Request` (method, URL, headers, body reader).
2. Create an `http.ResponseWriter`.
3. Call `mux.ServeHTTP(w, r)`: the **router** looks up the path `/hello` in its table and finds `helloHandler`.
4. Call `helloHandler(w, r)`: **your code runs.** It might parse JSON, query a database (which itself parks the goroutine while waiting; Chapter 52+), and write to `w`.

```
bytes ─► parse ─► *http.Request ─► mux.ServeHTTP ─► match "/hello" ─► helloHandler(w, r)
                                                                          │
                                                       fmt.Fprintln(w, "Hello World")
```

Because **each connection has its own goroutine**, one slow handler doesn't block the others.

---

## 12. Step 7: the response travels back

`fmt.Fprintln(w, "Hello World")` doesn't write to the socket immediately; `net/http` buffers it. When the handler returns, `net/http`:

1. Fills in headers (`Date`, `Content-Length`, `Content-Type` sniffed from the body: `text/plain; charset=utf-8`).
2. **Writes** the full response into the socket. The trace shows:

```
write(4, "HTTP/1.1 200 OK\r\nDate: Sat, 26 S"..., 129) = 129
```

129 bytes: status line + headers + `Hello World\n`.

3. The **kernel** copies them into the socket's **send buffer**, and the TCP layer splits them into packets and hands them to the **NIC**, which transmits them.
4. Packets travel back over the network to the client. The client's kernel reassembles them into the client's socket receive buffer; the browser/curl `read`s them, parses the response, and displays "Hello World".

Then, for HTTP/1.1, the connection typically stays open (keep-alive) for the next request, which is why you see `epoll` events on fd `4` again for later requests. When it closes, `epoll_ctl(EPOLL_CTL_DEL, 4)` and `close(4)` follow.

---

## 13. The complete flow

```
1. Browser/curl builds "GET /hello HTTP/1.1..." (83 bytes)
        │  TCP connection (handshake) → packets
        ▼
2. Network (WiFi adapter → router → ... → server's NIC)
        ▼
3. NIC: receives frames → DMA into RAM receive ring → INTERRUPT
        ▼
4. Kernel: driver → IP → TCP (order, ack) → SOCKET RECEIVE BUFFER
        │  epoll marks socket readable
        ▼
5. Go runtime (netpoller): wakes the goroutine parked on that socket
        ▼
6. net/http: read() the 83 bytes → parse → *http.Request
        ▼
7. Router (ServeMux): "/hello" → helloHandler
        ▼
8. YOUR HANDLER: fmt.Fprintln(w, "Hello World")
        ▼
9. net/http: build headers, write() 129 bytes to the socket
        ▼
10. Kernel: SOCKET SEND BUFFER → TCP segments → NIC
        ▼
11. Network back to the client → browser shows "Hello World"
```

---

## 14. Who does what?

**So, "OS or Go server?" Both, at different layers:**

| Layer | Handled by | Responsibilities |
|-------|-----------|------------------|
| Physical signals, frames, checksums | **NIC hardware** | Receive/transmit, DMA into RAM, interrupt |
| Drivers, IP, **TCP** (reliability, ordering, congestion), buffering | **OS kernel** | Sockets, ports → processes, `bind/listen/accept/read/write`, `epoll` |
| Waiting efficiently, waking goroutines | **Go runtime** | Netpoller, scheduler (Chapter 36) |
| **HTTP**: parsing requests, writing responses, keep-alive | **`net/http`** (Go standard library) | Request/response objects, connection goroutines |
| **Routing** | **Router** (`ServeMux`, or a third-party router) | Path/method → handler |
| **Business logic**, JSON, database, auth | **Your code** | Everything the app is *for* |

**The OS does not understand HTTP.** To the kernel, it's just a stream of bytes on TCP port 8080. **Go does not implement TCP**; it asks the kernel via system calls. Each layer does one job and trusts the layers below.

---

## 15. Time budget of a request

Rough orders of magnitude for a request on a **fast local network**:

| Stage | Typical time |
|-------|--------------|
| Client builds the request | microseconds |
| Network transit (same datacenter) | ~0.1–1 ms; across the internet 10–200+ ms |
| NIC → kernel → socket buffer | ~10–50 µs |
| Wake the goroutine, parse HTTP | ~10–50 µs |
| Route to handler | ~1 µs |
| **Your handler** (with a DB query) | often **1–50 ms**: usually the dominant part |
| Write response, kernel → NIC | ~10–50 µs |

Conclusion: the machinery we described is **microseconds**. Latency in real backends is dominated by *your* code (the database, external calls) and by network distance, which is exactly why good concurrency (overlapping waits, Chapter 31) is what makes servers fast, and why unnecessary work inside handlers matters more than micro-optimizing the plumbing.

---

## 16. Troubleshooting

You may hit this the first time you run a server:

```
Error starting server: listen tcp :3000: bind: address already in use
```

(This happened while writing this very chapter: port 3000 on the author's machine was already held by another program.)

**Meaning:** the `bind` system call failed because **another process already owns that port**. A port can be bound by only one listening process at a time.

**Fixes:**
- Stop the other process, e.g., an old copy of your server still running in another terminal.
- Find who holds it:

```bash
ss -ltnp | grep :3000        # Linux
lsof -i :3000                # Linux / macOS
netstat -ano | findstr :3000 # Windows
```

- Or use a different port (`:8081`, `:18080`, ...).

Other common problems:

| Symptom | Cause |
|---------|-------|
| `permission denied` on `bind` | Ports below 1024 need special privileges. Use 1024+ |
| `curl: (7) Failed to connect` | Server isn't running / wrong port |
| Browser works, `curl` from another machine doesn't | Server bound to `localhost` only, or a firewall blocks the port. Use `:8080` (all interfaces) and open the firewall |
| Server exits immediately | You ignored the error from `ListenAndServe`. Always check it |
| Old behavior after code change | You didn't restart the server. Restart it after every change |

---

## 17. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "Go's `net/http` implements TCP" | The **kernel** implements TCP; Go makes system calls |
| "The OS understands HTTP" | The OS only moves bytes; HTTP is parsed by your process |
| "One thread per connection" | Go uses **one goroutine per connection**, parked cheaply via `epoll` |
| "`ListenAndServe` returns when the server is ready" | It **blocks** forever (returns only on error) |
| "The router is a physical device" | Here, "router" is a **software** table mapping URL patterns to handlers. (The WiFi router is separate hardware that forwards packets.) |
| "A request is one packet" | It's a **byte stream**, possibly split across several packets |
| "The handler is called by me" | It's called by `net/http` after routing |
| "`localhost` skips the OS network stack" | It goes through the kernel's TCP/IP stack (loopback), which is why it's a realistic test |

---

## 18. Exercises

### Exercise 1: Extend the server
Add a `/time` route that writes the current time, and a `/greet` route that reads a `?name=` query parameter and replies `Hello, <name>!` (defaulting to `stranger`).

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/time", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, time.Now().Format(time.RFC1123))
	})

	mux.HandleFunc("/greet", func(w http.ResponseWriter, r *http.Request) {
		name := r.URL.Query().Get("name")
		if name == "" {
			name = "stranger"
		}
		fmt.Fprintf(w, "Hello, %s!\n", name)
	})

	fmt.Println("listening on :8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		fmt.Println(err)
	}
}
```
Test: `curl 'localhost:8080/greet?name=Asha'` → `Hello, Asha!`
</details>

### Exercise 2: Order the steps
Put these in order: (a) `net/http` calls your handler, (b) NIC raises an interrupt, (c) kernel places bytes in the socket receive buffer, (d) Go runtime wakes the goroutine, (e) client sends the request, (f) router matches the path.

<details><summary>Solution</summary>

(e) → (b) → (c) → (d) → (f) → (a). (Parsing happens between (d) and (f).)
</details>

### Exercise 3: Which layer?
Who handles: (a) retransmitting a lost packet, (b) turning bytes into an `*http.Request`, (c) deciding which handler matches `/about`, (d) checking a password, (e) putting bytes on the wire?

<details><summary>Solution</summary>

(a) Kernel (TCP). (b) `net/http`. (c) The router (`ServeMux`). (d) Your code. (e) NIC hardware (after the kernel hands frames to the driver).
</details>

### Exercise 4: Trace a real server (Linux)
Run your server under `strace -f -e trace=socket,bind,listen,accept4,read,write` and make one `curl` request. Find (1) the port in `bind`, (2) the request text in `read`, (3) the response in `write`. What are the byte counts?

<details><summary>Solution</summary>

You should see `bind(... sin6_port=htons(8080) ...)`, then `read(N, "GET /hello HTTP/1.1\r\nHost: ...", 4096) = 83` (or similar, depending on the client), and `write(N, "HTTP/1.1 200 OK\r\n...", 129) = 129`. The exact numbers depend on your headers and response.
</details>

### Exercise 5: Two servers, one port
Start your server, then start a second copy in another terminal. What error do you get, and which system call failed? Then find the PID of the first server with `ss -ltnp` or `lsof -i`.

<details><summary>Solution</summary>

`listen tcp :8080: bind: address already in use`: the `bind` system call failed (`EADDRINUSE`). `ss -ltnp | grep 8080` shows the owning process and PID.
</details>

### Exercise 6: Concurrency in action
Add a route `/slow` that sleeps 3 seconds before replying. Open two terminals and run `curl /slow` in both at the same time, and `curl /hello` while they run. Does `/hello` wait? Why?

<details><summary>Solution</summary>

```go
mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
	time.Sleep(3 * time.Second)
	fmt.Fprintln(w, "finally")
})
```
Both `/slow` calls finish after ~3 s (not 6), and `/hello` answers instantly: each connection has its **own goroutine**, and sleeping goroutines are parked (they use no thread). That's the payoff of the design.
</details>

### Exercise 7 (challenge): Explain "non-blocking"
Why does `accept4` return `EAGAIN` instead of waiting, and how does Go still find out when a client connects?

<details><summary>Solution</summary>

Go opens sockets in **non-blocking** mode (`SOCK_NONBLOCK`) so that a call never blocks an OS thread. `EAGAIN` means "nothing yet". The runtime parks the goroutine and registers the socket with `epoll`; when a connection arrives the kernel reports the socket as readable, `epoll_wait` returns, the runtime marks the goroutine runnable, and it calls `accept4` again, which now succeeds.
</details>

---

## 19. Quiz

1. What does `http.ListenAndServe(":8080", mux)` do, and does it return?
2. What's the signature of an HTTP handler?
3. What is a NIC, and what is DMA?
4. Which part understands TCP: the kernel or `net/http`?
5. What system calls set up a listening server?
6. How does Go wait for thousands of idle connections without thousands of threads?
7. What does `bind: address already in use` mean?

<details><summary>Answers</summary>

1. Binds port 8080, accepts connections, and dispatches requests to the router; it blocks and returns only on error.
2. `func(w http.ResponseWriter, r *http.Request)`.
3. The network hardware; DMA lets it write incoming data directly into RAM without the CPU copying it.
4. The kernel.
5. `socket`, `bind`, `listen` (then `accept4` in a loop).
6. Non-blocking sockets + `epoll` netpoller + parked goroutines.
7. Another process already holds that port.
</details>

---

## 20. Summary

- A Go server is a few lines of `net/http`: a **router** (`ServeMux`) maps paths to **handlers** `func(w http.ResponseWriter, r *http.Request)`, and `http.ListenAndServe(addr, mux)` runs the server forever.
- The full journey: **client → network → NIC (DMA + interrupt) → kernel (IP/TCP → socket receive buffer) → Go runtime (epoll netpoller wakes a goroutine) → `net/http` parses → router → your handler → `write` → kernel send buffer → NIC → client**.
- The **OS** does hardware, TCP, sockets, and ports; **Go** does HTTP, goroutine scheduling, routing, and your logic. The kernel doesn't know HTTP; Go doesn't implement TCP.
- System calls (verified with `strace`): `socket → bind → listen → accept4 (EAGAIN → epoll) → read → write`.
- **One goroutine per connection** + a **netpoller** = thousands of concurrent connections on a few threads.
- Real latency is dominated by handler work (databases, external calls) and network distance, not by the plumbing.
- Always check `ListenAndServe`'s error; `address already in use` means the port is taken.

### ➡️ What's next?

[Chapter 39](39-the-go-runtime.md) opens the **Go runtime** itself: the scheduler, memory allocator, garbage collector, and netpoller that we've been leaning on throughout Part 6 and 7.
