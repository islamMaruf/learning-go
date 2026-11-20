# Chapter 37: Into Backend Development — How the Web Works, and What REST Really Means

> **Goal of this chapter:** Start Part 7 by understanding *what a backend is* and *why it looks the way it does*. You'll follow the history of web development (static pages → server-rendered pages → AJAX → REST APIs), learn what **REST** actually stands for (**Re**presentational **S**tate **T**ransfer), see real **HTTP** requests and responses from a working Go server, and understand the contract between **frontend** and **backend** developers. No code beyond a tiny server is needed to follow along.

**Difficulty:** 🟡 Beginner–Intermediate (concepts)  **Estimated time:** 3 hours  **Prerequisite:** [Chapter 36](36-goroutines.md) (helpful, not strictly required)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why backend now?](#2-why-backend-now)
3. [The web in one page: clients, servers, requests](#3-the-web-in-one-page)
4. [A real HTTP conversation](#4-a-real-http-conversation)
5. [A short history of web development](#5-a-short-history-of-web-development)
6. [Web 1.0: the static era](#6-web-10-the-static-era)
7. [Web 2.0: server-side rendering](#7-web-20-server-side-rendering)
8. [The AJAX revolution](#8-the-ajax-revolution)
9. [REST APIs: where the modern backend begins](#9-rest-apis)
10. [Decoding REST: Resource, State, Representation, Transfer](#10-decoding-rest)
11. [The REST rules of the game](#11-the-rest-constraints)
12. [HTTP methods, status codes, headers, JSON](#12-http-methods-status-codes-headers-json)
13. [Designing REST endpoints](#13-designing-rest-endpoints)
14. [Frontend vs. backend: the contract](#14-frontend-vs-backend-the-contract)
15. [Why Go for backends](#15-why-go-for-backends)
16. [What we'll build](#16-what-well-build)
17. [Common misconceptions](#17-common-misconceptions)
18. [Exercises](#18-exercises)
19. [Quiz](#19-quiz)
20. [Summary](#20-summary)

---

## 1. What you will learn

- What a **backend** is and what it does
- How a browser talks to a server: **URL → DNS → TCP → HTTP request → HTTP response**
- The evolution: **static pages → server-side rendered pages → AJAX → REST/JSON APIs**
- What each letter of **REST** means, in plain words (a classic interview topic)
- **HTTP methods** (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`), **status codes**, **headers**, and **JSON**
- How to name and design API **endpoints**
- The division of work between frontend and backend teams
- Why Go's goroutines make it a great backend language

---

## 2. Why backend now?

You've learned the *language* (syntax, functions, structs, slices, pointers) and the *machine* (memory, processes, threads, goroutines). Now we apply everything to what most Go developers actually build: **backend services**: programs that run on servers and respond to requests over a network.

Why teach the history and theory *before* the code? Because the **shape** of a modern backend (JSON over HTTP, stateless requests, resources and endpoints) is the result of a specific evolution. If you understand the *problems* each stage solved, the design choices stop being arbitrary rules to memorize. Many developers can copy a working API handler but can't explain what "REST" means; you'll be able to.

---

## 3. The web in one page

### The players

```
   ┌──────────────┐      request         ┌──────────────┐
   │   CLIENT     │ ───────────────────► │   SERVER     │
   │ (browser,    │                      │ (your Go     │
   │  mobile app, │ ◄─────────────────── │  program on  │
   │  another     │      response        │  a machine)  │
   │  program)    │                      └──────┬───────┘
   └──────────────┘                             │
                                          ┌─────▼──────┐
                                          │ Database   │
                                          └────────────┘
```

- **Client**: anything that *asks*: a browser, a phone app, `curl`, another server.
- **Server**: a program that *listens* for requests and *answers*. It runs continuously (unlike a program that runs once and exits).
- **Backend**: the server-side code + the database behind it: **business logic, data storage, security**.
- **Frontend**: what the user sees and touches (HTML/CSS/JavaScript in the browser, or a mobile UI).

### What happens when you open `https://shop.example.com/products`

```
1. You type the URL.
2. DNS lookup:      "shop.example.com" → 203.0.113.7   (the server's IP address)
3. TCP connection:  your machine ⇄ 203.0.113.7 on port 443 (a reliable byte pipe)
4. TLS handshake:   (for https) encrypt the pipe
5. HTTP REQUEST:    "GET /products, please" sent over the pipe
6. The server's program (listening on that port) reads the request,
   does work (maybe queries a database),
7. HTTP RESPONSE:   "200 OK, here's the data" sent back
8. The client displays or uses it.
```

### Key vocabulary

| Term | Meaning |
|------|---------|
| **URL** | `scheme://host:port/path?query`, e.g. `https://shop.example.com:443/products?page=2` |
| **IP address** | Numeric address of a machine (`203.0.113.7`, `127.0.0.1` = "this machine") |
| **DNS** | The phone book mapping names to IP addresses |
| **Port** | A number (0–65535) identifying *which program* on the machine should receive the data. HTTP = 80, HTTPS = 443. Our dev servers will use `8080` or similar |
| **Socket** | The OS's endpoint for network communication (Chapter 27: system calls `socket`, `bind`, `listen`, `accept`) |
| **HTTP** | **H**yper**T**ext **T**ransfer **P**rotocol: the text-based "language" of requests and responses |
| **Localhost** | `localhost` / `127.0.0.1`: your own machine (used during development) |

**A server, at its core**, is a loop: *accept a connection → read the request → run your handler → write the response.* In Go each connection is handled in its own **goroutine** (Chapter 36), which is why Go servers scale so well.

---

## 4. A real HTTP conversation

HTTP is human-readable text. Here's a **real** session with a tiny Go server. The server code (don't worry about understanding it yet; we'll dissect it in Chapter 40):

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

type Product struct {
	ID    int     `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
}

func main() {
	http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello, Web!"))
	})

	http.HandleFunc("/products", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode([]Product{{1, "Notebook", 2.5}, {2, "Pen", 0.75}})
	})

	log.Fatal(http.ListenAndServe(":8080", nil)) // listen on port 8080, forever
}
```

Run it (`go run .`), then in another terminal use `curl` (a command-line HTTP client) with `-v` to see everything:

```bash
curl -v http://localhost:8080/hello
```

**The request the client sent** (lines starting with `>`):

```
> GET /hello HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/8.5.0
> Accept: */*
>
```

**The response the server returned** (lines starting with `<`):

```
< HTTP/1.1 200 OK
< Date: Sat, 26 Sep 2026 05:34:13 GMT
< Content-Length: 11
< Content-Type: text/plain; charset=utf-8
<
Hello, Web!
```

### Anatomy of an HTTP request

```
GET /hello HTTP/1.1              ← request line: METHOD  PATH  VERSION
Host: localhost:8080             ← headers (key: value), metadata about the request
User-Agent: curl/8.5.0
Accept: */*
                                 ← blank line ends the headers
(optional body)                  ← data being sent (for POST/PUT/PATCH)
```

### Anatomy of an HTTP response

```
HTTP/1.1 200 OK                  ← status line: VERSION  STATUS-CODE  REASON
Content-Type: application/json   ← headers
Content-Length: 76
                                 ← blank line
[{"id":1,"name":"Notebook","price":2.5}, ...]     ← body
```

A JSON endpoint (real output):

```bash
curl -i http://localhost:8080/products
```

```
HTTP/1.1 200 OK
Content-Type: application/json
Date: Sat, 26 Sep 2026 05:34:13 GMT
Content-Length: 76

[{"id":1,"name":"Notebook","price":2.5},{"id":2,"name":"Pen","price":0.75}]
```

And what if the path doesn't exist?

```bash
curl -i http://localhost:8080/nothing
```

```
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
Content-Length: 19

404 page not found
```

The **status code** (`200`, `404`) tells the client *what happened* (section 12). Now you've seen a real HTTP exchange: everything else in this course is building richer versions of exactly this.

---

## 5. A short history of web development

Why has web development changed so much? Each stage fixed a limitation of the previous one.

```
 1990s          2000s            2005+            2010s+
┌────────┐   ┌────────────┐   ┌───────────┐   ┌───────────────────┐
│ Web 1.0│──►│ Web 2.0    │──►│  AJAX     │──►│ REST APIs + JSON  │
│ static │   │ dynamic,   │   │ partial   │   │ (frontend/backend │
│ pages  │   │ server-    │   │ page      │   │  fully separated) │
│        │   │ rendered   │   │ updates   │   │                   │
└────────┘   └────────────┘   └───────────┘   └───────────────────┘
```

---

## 6. Web 1.0: the static era

**Era:** early-to-mid 1990s.

**How it worked:** A "website" was a folder of **files** (`.html`, images). A web server just **found the file and sent it**. Every visitor saw exactly the same content. There was no user data, no login, no database.

```
Browser ── GET /about.html ──► Web server
                                  │ (reads the file from disk)
Browser ◄── about.html ──────────┘
```

**Characteristics:**
- ✅ Simple, fast, easy to host and cache
- ❌ **Same page for everyone**: no personalization
- ❌ **Content changes require a human to edit files**
- ❌ No user accounts, no shopping cart, no comments

Great for a company brochure, hopeless for a shop or a social network.

---

## 7. Web 2.0: server-side rendering

**Era:** ~late 1990s–2000s.

**What changed:** The server became a **program**, not just a file finder. It could run code, talk to a **database**, and **generate the HTML on the fly** for each request ("rendering" on the server). Languages like PHP, Java (JSP), ASP, Ruby (Rails), and Python (Django) powered this.

```
Browser ── GET /profile ──►  Server program
                              │ 1. Who is this user? (cookie/session)
                              │ 2. Query the database
                              │ 3. Build an HTML page with THEIR data
Browser ◄── full HTML page ──┘
```

**Example:** Facebook-style profile: two users request `/profile`; the *same code* produces two *different* pages, using their own data.

**Innovations:** user accounts, personalization, e-commerce, forums, blogs, content management systems.

**Limitations:**
- ⚠️ **Every interaction reloads the whole page.** Click "Like" → the server re-renders and resends the *entire* page (header, menu, images...), and the screen flashes.
- ⚠️ **Server does everything** (data + layout), so heavy load and tight coupling of design and logic.
- ⚠️ Hard to build **rich, app-like** experiences, and hard to reuse the backend for a **mobile app**, which needs data, not HTML.

---

## 8. The AJAX revolution

**Era:** ~2005 (the term "AJAX" was coined then; Gmail and Google Maps popularized it).

**AJAX** = **A**synchronous **J**avaScript **A**nd **X**ML. Idea: JavaScript running in the browser can make **background requests** to the server, get *just the data* it needs, and **update part of the page** without reloading everything.

```
Before (Web 2.0):   click "Like" ─► full page request ─► full HTML page ─► whole page reloads
After (AJAX):      click "Like" ─► JS sends small request ─► server replies with tiny data
                                   ─► JS updates ONLY the like counter
```

**What it introduced:** the server exposes **endpoints that return data** (first XML, soon **JSON**) rather than complete pages. The browser's JavaScript becomes an *application* that consumes those endpoints. (Sending JSON instead of XML became the norm; the name stuck even though "X" is rarely XML now.)

```
GET /api/likes/42     →   {"count": 128}
POST /api/likes/42    →   {"count": 129}
```

**The paradigm shift:** the server's job splits into (a) *serving the app's code* and (b) *serving data through an API*. This paved the way for single-page apps (React, Vue, Angular) and for mobile apps that talk to the very same endpoints.

---

## 9. REST APIs

**Era:** ~2000 (defined in Roy Fielding's PhD dissertation), widely adopted from ~2010.

Once servers exposed data endpoints, every team invented its own conventions: `/getUser?id=5`, `/createNewUser`, `/deleteTheUserPlease`... chaos. **REST** (**Re**presentational **S**tate **T**ransfer) is an **architectural style** that gives a consistent, predictable way to design them, built on top of HTTP's existing features.

A REST-style API looks like this:

```
GET    /users          → list users
GET    /users/5        → get user 5
POST   /users          → create a user
PUT    /users/5        → replace user 5
PATCH  /users/5        → update part of user 5
DELETE /users/5        → delete user 5
```

Same URLs, different **HTTP methods** → different actions. **This is what our Go backend will look like**, starting in Chapter 40.

---

## 10. Decoding REST

REST stands for **Re**presentational **S**tate **T**ransfer. Let's decode each word: interviewers love this.

### Resource

> A **resource** is any *thing* your system exposes: something with an identity that can be named, referenced, and acted upon.

In an e-commerce app: a **user**, a **product**, an **order**, a **cart**. Each resource has a URL that identifies it:

```
/users/5          ← the user with ID 5
/products/42      ← product 42
/orders/1001      ← order 1001
/products         ← the *collection* of all products
```

**Nouns, not verbs.** The URL names the *thing*; the HTTP method says what to *do* to it. `GET /products/42` (not `/getProduct?id=42`).

### State

> **State** is the **current data/condition of a resource at a point in time.**

Product 42's state right now: `{name: "Notebook", price: 2.50, stock: 120}`. Tomorrow after a sale: `{..., price: 2.00, stock: 95}`. Same resource, *different state*. A user's state includes their name, email, and whether they're logged in.

### Representation

> A **representation** is a **description of a resource's state in some format**, which is what actually travels between client and server.

The server doesn't send the product *itself* (which lives in a database on disk); it sends a **representation** of its current state: usually **JSON**:

```json
{ "id": 42, "name": "Notebook", "price": 2.5, "stock": 120 }
```

The same state could be represented as XML, HTML, CSV, or PDF. The **resource** (the thing) is separate from its **representation** (a snapshot in a format). The word "re-presentation" means "presenting again": showing the state again in a transportable form. (Analogy: a photograph of you is a *representation* of you, not you.)

The `Content-Type` header says which format: `application/json`.

### Transfer

> **Transfer** means these representations are **moved between client and server** over HTTP.

- `GET`: server → client: "transfer me the current state (representation)".
- `POST`/`PUT`/`PATCH`: client → server: "here's a representation; update or create the resource's state to match."

### Putting it together

> **REST = Representational State Transfer** = *transferring representations of resource state between client and server.*

```
  CLIENT                                            SERVER
    │  GET /products/42                                │
    │ ────────────────────────────────────────────────►│  looks up product 42's STATE in the DB
    │                                                  │  encodes it as a REPRESENTATION (JSON)
    │  200 OK  {"id":42,"name":"Notebook","price":2.5} │
    │ ◄────────────────────────────────────────────────│   ← TRANSFER
    │                                                  │
    │  PATCH /products/42  {"price": 2.0}              │
    │ ────────────────────────────────────────────────►│  updates the resource's STATE
    │  200 OK  {"id":42,"name":"Notebook","price":2.0} │
    │ ◄────────────────────────────────────────────────│
```

---

## 11. The REST constraints

REST is defined by a handful of design rules (Fielding's "constraints"). The ones you'll feel every day:

| Constraint | Meaning | Why it matters |
|------------|---------|----------------|
| **Client–server** | Separate concerns: the client handles UI, the server handles data & logic | Teams and technologies evolve independently |
| **Stateless** | **Each request contains everything the server needs** (who you are, what you want). The server keeps no per-client conversation state *between requests* | Any server instance can handle any request → easy to scale horizontally; simpler recovery |
| **Cacheable** | Responses say whether they may be cached | Speed, lower load |
| **Uniform interface** | Consistent conventions: resources identified by URLs, standard methods, representations (JSON), self-descriptive messages | Predictability; anyone can guess how the API works |
| **Layered system** | Clients don't know if they're talking to the origin server, a proxy, a CDN, or a load balancer | Flexibility, security, scaling |
| **Code on demand** *(optional)* | Server may send executable code (e.g., JavaScript) | Rarely discussed for APIs |

### Stateless, in practice

"Stateless" doesn't mean the *application* has no state (databases store plenty!). It means the **server doesn't remember your previous requests in memory**. So each request carries **credentials** (e.g., a token in the `Authorization` header: exactly what **JWT** authentication does in Chapter 48). A load balancer can then send request 1 to server A and request 2 to server B with no problem.

> **Analogy: a restaurant with no memory.** Every time you order, you say who you are and what you want ("Table 5, the usual, please"). The waiter doesn't remember you from last time; but the *kitchen* (database) has your history. Any waiter can serve you.

### A note about "RESTful"

Real-world APIs are often called "RESTful" when they follow the popular subset: resource-style URLs, HTTP methods, JSON, proper status codes, and statelessness. Purists argue about the rest (e.g., hypermedia links); in day-to-day work, *this* subset is what people mean. That's what we'll build.

---

## 12. HTTP methods, status codes, headers, JSON

### Methods (verbs)

| Method | Meaning | Body? | Safe? | Idempotent? | Typical use |
|--------|---------|-------|-------|-------------|-------------|
| **GET** | Read a resource | No | ✅ | ✅ | List/fetch |
| **POST** | Create a resource / trigger an action | Yes | ❌ | ❌ | Create user, place order |
| **PUT** | Replace a resource entirely | Yes | ❌ | ✅ | Full update |
| **PATCH** | Modify part of a resource | Yes | ❌ | (usually) | Partial update |
| **DELETE** | Remove a resource | Usually no | ❌ | ✅ | Delete |
| **OPTIONS** | Ask what's allowed (used for CORS preflight: Chapter 42) | No | ✅ | ✅ | Browser security check |
| **HEAD** | Like GET but only headers | No | ✅ | ✅ | Check existence/size |

- **Safe** = doesn't change server state. **Idempotent** = doing it *N* times has the same effect as once (`DELETE /users/5` twice: the user is gone either way; `POST /orders` twice creates *two* orders).

### Status codes

The first digit says the class:

| Range | Meaning | Common codes |
|-------|---------|--------------|
| **1xx** | Informational | `101 Switching Protocols` |
| **2xx** | ✅ Success | `200 OK`, `201 Created`, `204 No Content` |
| **3xx** | ↪ Redirection | `301 Moved Permanently`, `302 Found`, `304 Not Modified` |
| **4xx** | ❌ *Client* error (your fault) | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `405 Method Not Allowed`, `409 Conflict`, `422 Unprocessable Entity`, `429 Too Many Requests` |
| **5xx** | 💥 *Server* error (our fault) | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable` |

**401 vs 403:** `401` = "I don't know who you are (not authenticated)". `403` = "I know who you are, but you're not allowed".

### Headers

Key/value metadata on requests and responses:

| Header | Direction | Purpose |
|--------|-----------|---------|
| `Content-Type` | both | Format of the body (`application/json`) |
| `Accept` | request | Formats the client wants back |
| `Authorization` | request | Credentials (e.g., `Bearer <token>`) |
| `Content-Length` | both | Size of the body in bytes |
| `Location` | response | URL of a newly created resource (with `201`) |
| `Cache-Control` | response | Caching rules |
| `Set-Cookie` / `Cookie` | response / request | Session cookies |

### JSON

**JSON** (JavaScript Object Notation) is the standard representation format for APIs: text with objects `{}`, arrays `[]`, strings, numbers, booleans, and `null`:

```json
{
  "id": 42,
  "name": "Notebook",
  "price": 2.5,
  "tags": ["paper", "school"],
  "inStock": true,
  "discount": null
}
```

Go's `encoding/json` converts between JSON and structs using **struct tags** (Chapter 21):

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Product struct {
	ID    int      `json:"id"`
	Name  string   `json:"name"`
	Price float64  `json:"price"`
	Tags  []string `json:"tags"`
}

func main() {
	p := Product{42, "Notebook", 2.5, []string{"paper", "school"}}

	out, _ := json.Marshal(p) // Go value → JSON text
	fmt.Println(string(out))  // {"id":42,"name":"Notebook","price":2.5,"tags":["paper","school"]}

	var q Product
	json.Unmarshal([]byte(`{"id":7,"name":"Pen","price":0.75,"tags":[]}`), &q) // JSON → Go value
	fmt.Printf("%+v\n", q) // {ID:7 Name:Pen Price:0.75 Tags:[]}
}
```

---

## 13. Designing REST endpoints

Conventions that make an API predictable:

**1. Use plural nouns for collections; IDs for members.**

```
GET    /products          list
POST   /products          create
GET    /products/42       read one
PUT    /products/42       replace
PATCH  /products/42       partial update
DELETE /products/42       delete
```

**2. Nest to show relationships (but not too deep).**

```
GET /users/5/orders          orders of user 5
GET /orders/1001/items       items of order 1001
```

**3. Use query parameters for filtering, sorting, and paging** (Chapter 63):

```
GET /products?category=books&sort=price&page=2&limit=20
```

**4. Never put verbs in URLs.** ❌ `/createProduct`, `/deleteUser/5` → ✅ `POST /products`, `DELETE /users/5`.

**5. Return the right status code**:

| Action | Success response |
|--------|------------------|
| `GET` found | `200 OK` + body |
| `POST` created | `201 Created` + body + `Location` header |
| `DELETE` done | `204 No Content` (or `200`) |
| Invalid input | `400 Bad Request` (or `422`) with an error message |
| Not logged in | `401 Unauthorized` |
| Not allowed | `403 Forbidden` |
| Unknown ID | `404 Not Found` |
| Bug on our side | `500 Internal Server Error` |

**6. Return consistent JSON**, including for errors:

```json
{ "error": "product not found", "code": "PRODUCT_NOT_FOUND" }
```

**7. Version your API** so you can evolve it: `/v1/products`.

---

## 14. Frontend vs. backend: the contract

Modern projects split into (at least) two codebases, often built by different people, meeting at the **API**.

```
   FRONTEND (React app, mobile app, ...)            BACKEND (your Go service)
   ┌──────────────────────────────┐                ┌──────────────────────────────┐
   │ UI design, pages, forms      │                │ Business logic, validation   │
   │ user interactions            │   HTTP + JSON  │ Database access              │
   │ calls the API, renders JSON  │ ◄────────────► │ Authentication/authorization │
   │                              │   THE CONTRACT │ Security, performance, rules │
   └──────────────────────────────┘                └──────────────────────────────┘
```

| Frontend developers | Backend developers |
|---------------------|--------------------|
| Build what users see | Build what makes it work |
| Consume the API | Design and provide the API |
| Handle loading/error display | Return correct data and status codes |
| Don't (and shouldn't) touch the DB | Own the database and business rules |

### The API is the contract

Before either side codes, they agree on: *which endpoints exist, what each accepts, and what each returns.* Example agreement:

> `POST /v1/login` accepts `{"email": "...", "password": "..."}` and returns `200` with `{"token": "..."}` or `401` with `{"error": "invalid credentials"}`.

With this contract, the frontend can build against a mock while the backend is being written. The same backend serves a website, a phone app, and a partner's server, unchanged.

> **Never trust the client.** The frontend can validate forms for good UX, but a malicious user can skip your UI and call the API directly with anything. **The backend must validate and authorize every request.**

---

## 15. Why Go for backends

| Go strength | Why it matters for servers |
|-------------|---------------------------|
| **Goroutines** (Chapter 36) | One goroutine per request; 100,000s of concurrent connections in simple sequential-looking code |
| **Standard library** | `net/http`, `encoding/json`, `database/sql`, `crypto/*`, `testing`: a production-grade server without frameworks |
| **Fast compiled binary** | Low latency, low memory, quick startup |
| **Single static binary** | Deploy by copying one file (Chapter 19); tiny Docker images |
| **Simple language** | Easy to read, review, and maintain |
| **Strong typing + tooling** | Catches errors at compile time; `gofmt`, `go vet`, race detector |
| **Ecosystem** | Docker, Kubernetes, and many cloud tools are written in Go |

You'll start with only the standard library (Chapters 40–46), then add a few third-party packages (JWT, PostgreSQL driver) as needed.

---

## 16. What we'll build

The rest of Part 7 → Part 13 builds a real **e-commerce backend API**, step by step:

```
Part 7   (37–39)  the theory + Go runtime
Part 8   (40–46)  GET & POST endpoints, CORS, routing, middleware (logger)
Part 9   (47–49)  configuration, JWT authentication
Part 10  (50–51)  clean architecture, interfaces
Part 11  (52–58)  PostgreSQL, CRUD, migrations
Part 12  (59–61)  Domain-driven design
Part 13  (62–63)  pagination
Part 14  (64–70)  concurrency deep dive
```

By the end you'll have implemented every idea in this chapter: resources (`users`, `products`), JSON representations, HTTP methods, status codes, stateless auth, and a database behind it all.

---

## 17. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "REST is a protocol/standard" | It's an **architectural style**; HTTP is the protocol |
| "REST = JSON" | JSON is the popular representation; REST doesn't mandate it |
| "REST APIs have no state" | The *server doesn't keep client session state between requests*; data is stored in databases |
| "URLs should contain verbs" | Use nouns; the HTTP method is the verb |
| "`POST` is for creating, `GET` can change things too" | `GET` must be safe (no side effects) |
| "`401` and `403` are the same" | 401 = not authenticated; 403 = authenticated but forbidden |
| "Frontend validation is enough" | The backend must always validate |
| "The backend is just a database wrapper" | It enforces business rules, security, consistency, and orchestrates multiple systems |
| "A 200 with `{error: ...}` is fine" | Use the proper 4xx/5xx status codes |
| "The API is only for the website" | The same API can serve web, mobile, and other servers |

---

## 18. Exercises

### Exercise 1: Match the era
Web 1.0, Web 2.0, AJAX, REST/JSON APIs: which one is described by each?
(a) The server generates a personalized full HTML page per request. (b) The server returns the same static file to everyone. (c) JavaScript fetches small pieces of data and updates part of the page. (d) Resources identified by URLs, manipulated via HTTP methods, exchanged as JSON.

<details><summary>Solution</summary>

(a) Web 2.0. (b) Web 1.0. (c) AJAX. (d) REST APIs.
</details>

### Exercise 2: Design the endpoints
Design REST endpoints for a **library**: books, members, and loans. Include listing books, borrowing a book, and returning it.

<details><summary>Solution</summary>

```
GET    /books                 list books (?author=..&available=true)
POST   /books                 add a book
GET    /books/{id}            one book
PUT    /books/{id}            replace a book
DELETE /books/{id}            remove a book
GET    /members/{id}/loans    a member's loans
POST   /loans                 borrow: {"bookId":42,"memberId":7} → 201 Created
PATCH  /loans/{id}            return a book: {"returned": true}  (or DELETE /loans/{id})
```
Note the nouns; "borrow" is *creating a loan* and "return" is *updating* it.
</details>

### Exercise 3: Pick the status code
What status would you return for: (a) creating a product successfully, (b) missing JSON field `name`, (c) calling `/orders/999` that doesn't exist, (d) a request without a token to a protected endpoint, (e) a logged-in regular user calling an admin-only endpoint, (f) a database outage.

<details><summary>Solution</summary>

(a) `201 Created`. (b) `400 Bad Request` (or `422`). (c) `404 Not Found`. (d) `401 Unauthorized`. (e) `403 Forbidden`. (f) `500` (or `503 Service Unavailable`).
</details>

### Exercise 4: Explain REST
In your own words, explain each of the four words in **Re**presentational **S**tate **T**ransfer, using a *user* as your example resource.

<details><summary>Solution</summary>

**Resource:** the user (identified by `/users/5`). **State:** its current data (name, email, role). **Representation:** a JSON snapshot of that state, `{"id":5,"name":"Asha"}`. **Transfer:** sending that representation between client and server via HTTP (`GET` to retrieve, `PUT`/`PATCH` to send a modified one back).
</details>

### Exercise 5: Stateless or not?
A server stores "user 5 is logged in" in its memory after `/login`, and later requests are recognized by the connection. Is this stateless? What problem arises with 3 servers behind a load balancer? What's the RESTful alternative?

<details><summary>Solution</summary>

Not stateless: the server holds per-client session state. With multiple servers, a later request may hit a server that doesn't know the user is logged in. RESTful alternative: each request carries a **token** (e.g., JWT) in the `Authorization` header that any server can verify without shared memory. (Chapter 48.)
</details>

### Exercise 6: Run your own
Run the section 4 server and use `curl -i` to: (1) `GET /products`, (2) `GET /missing`, (3) `POST /hello` (`curl -i -X POST localhost:8080/hello`). What differences do you see in the status lines and headers? (Our handler doesn't restrict methods, so what does that tell you?)

<details><summary>Solution</summary>

(1) `200 OK` with `Content-Type: application/json`. (2) `404 Not Found`. (3) `200 OK` (a `POST` to `/hello` also works): the handler ignores the method. Real APIs check `r.Method` and return `405 Method Not Allowed`; you'll do that in Chapters 40–44.
</details>

### Exercise 7 (challenge): Critique this API
Point out at least five problems and propose fixes:

```
GET  /getAllProducts
GET  /product?id=5
POST /createProduct
POST /deleteProduct?id=5
GET  /updateProductPrice?id=5&price=3
```

<details><summary>Solution</summary>

Verbs in URLs (`get`, `create`, `delete`, `update`), inconsistent singular/plural, `GET` used to *modify* data (unsafe, cacheable by mistake, triggered by crawlers), `POST` used to delete, IDs as query parameters instead of path segments, and no versioning. Fix:
```
GET    /v1/products
GET    /v1/products/5
POST   /v1/products
DELETE /v1/products/5
PATCH  /v1/products/5     {"price": 3}
```
</details>

---

## 19. Quiz

1. What is the job of the backend?
2. What's the difference between a URL's host and its port?
3. What are the three parts of an HTTP response's first line?
4. What problem did AJAX solve compared with Web 2.0?
5. What does REST stand for?
6. What does "stateless" mean for a REST API?
7. What's the difference between `PUT` and `PATCH`?
8. Which status code family indicates the client made a mistake?

<details><summary>Answers</summary>

1. Business logic, data storage, and security behind the scenes: process requests and return responses.
2. The host identifies the machine; the port identifies the program on that machine.
3. HTTP version, status code, reason phrase (e.g., `HTTP/1.1 200 OK`).
4. Full-page reloads; AJAX updates just part of the page by fetching data in the background.
5. Representational State Transfer.
6. The server keeps no per-client session state between requests; each request carries everything needed (e.g., a token).
7. `PUT` replaces the whole resource; `PATCH` changes only some fields.
8. 4xx.
</details>

---

## 20. Summary

- A **backend** is the server-side program (plus database) that holds business logic, data, and security. Clients talk to it over **HTTP**: `request → response`.
- Web evolution: **Web 1.0** (static files) → **Web 2.0** (server-rendered dynamic pages) → **AJAX** (background data requests, partial updates) → **REST APIs** (JSON endpoints serving any client).
- **REST = Representational State Transfer**: clients and servers transfer **representations** (usually JSON) of a **resource's state**, identified by URLs and manipulated with standard HTTP methods.
- REST's key ideas: **client-server, stateless, cacheable, uniform interface, layered**.
- **HTTP essentials:** methods (`GET/POST/PUT/PATCH/DELETE`), **status codes** (2xx ok, 4xx client error, 5xx server error), **headers**, **JSON** bodies.
- Design endpoints with **plural nouns**, correct methods and status codes, consistent JSON, and versioning.
- Frontend and backend meet at the **API contract**; the backend must **never trust the client**.
- Go is a great fit: goroutine-per-request, a rich standard library, fast static binaries.

### ➡️ What's next?

[Chapter 38](38-os-or-go-server-the-complete-journey-of-a-request.md) follows a request **all the way down**: from the network card through the operating system into your Go server, and back: the complete journey "OS or Go server?".
