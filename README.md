# Learning Go: From Your First Program to Concurrent, Database-Backed Web Services

> A complete, beginner-friendly and depth-first Go course in 70 chapters. Every chapter explains *what*, *why*, and *how*, with analogies, annotated code, diagrams, **real program output**, common mistakes, exercises with solutions, a quiz, and a summary. You start with `Hello, World!` and finish with a tested, layered, PostgreSQL-backed REST API with authentication, pagination and safe concurrency.

## 📚 Overview

This course is written for people who want to *understand* Go, not just copy it:

- **Beginner friendly:** no Go knowledge needed; programming basics in any language help. Concepts are introduced with everyday analogies before the code.
- **Useful at every level:** intermediate and advanced developers will find the memory model, the scheduler, race conditions, PostgreSQL internals, DDD, and performance experiments with measured numbers.
- **Verified:** code blocks were compiled and run; the e-commerce project was built stage by stage and tested with `go test -race`; outputs, timings, memory numbers and error messages shown in the text come from real runs (PostgreSQL 16 in Docker, Go 1.25). Where the original material said something inaccurate, the chapters correct it (for example floating-point constants, byte-vs-character lengths, and pagination costs).
- **A learning platform:** each chapter follows the same template, so you always know where to look.

### The chapter template

| Section | Purpose |
|---------|---------|
| Title, goal, difficulty, time, prerequisites | know what you're getting into |
| Table of contents, "What you will learn" | orientation |
| Analogies and diagrams | build intuition first |
| Annotated code with real output | see it work |
| Common mistakes (table) | learn from typical errors, often reproduced with real output |
| Interview questions | check yourself and prepare |
| Exercises (with collapsible solutions) | practise |
| Quiz (with answers) | verify |
| Summary and "What's next?" | consolidate and connect |

Difficulty labels: 🟢 beginner · 🟡 intermediate · 🔴 advanced.

## 🎯 Learning Path

### Part 1: Go Fundamentals (Chapters 1-6)
Syntax, data types, decisions, and your first functions.

- [Chapter 1: Your First Go Program: Hello, World!](01-your-first-go-program.md)
- [Chapter 2: Variables and Data Types](02-variables-and-data-types.md)
- [Chapter 3: Making Decisions: `if`, `else`, `switch` (and a First Look at `for`)](03-making-decisions.md)
- [Chapter 4: Introduction to Functions](04-introduction-to-functions.md)
- [Chapter 5: Functions with Return Values](05-functions-with-return-values.md)
- [Chapter 6: More Function Examples: Practising Every Shape](06-more-function-examples.md)

### Part 2: Functions and Scope (Chapters 7-17)
Why functions exist, scope rules, packages, and every flavor of function.

- [Chapter 7: Why Functions Are Needed: A Real-World Refactor](07-why-functions-are-needed.md)
- [Chapter 8: What Is Scope? The Most Important Concept in This Section](08-what-is-scope.md)
- [Chapter 9: Local Scope and Block Scope](09-local-scope-and-block-scope.md)
- [Chapter 10: Package Scope: Multiple Files, Custom Packages, and Modules](10-package-scope.md)
- [Chapter 11: Scope Deep Dive: A "Boring" Example That Teaches Everything](11-scope-deep-dive.md)
- [Chapter 12: Variable Shadowing: The Shadow Effect](12-variable-shadowing.md)
- [Chapter 13: Function Types: Standard (Named) Functions](13-function-types.md)
- [Chapter 14: The `init` Function: The Automatic Initializer](14-the-init-function.md)
- [Chapter 15: Anonymous Functions and IIFE](15-anonymous-functions-and-iife.md)
- [Chapter 16: Function Expressions: Storing Functions in Variables](16-function-expressions.md)
- [Chapter 17: Parameters vs. Arguments, First-Order vs. Higher-Order Functions](17-parameters-vs-arguments-first-order-vs-higher-order-functions.md)

### Part 3: Memory and Internals (Chapters 18-20)
How Go programs use memory, and what a closure really is.

- [Chapter 18: Go Internal Memory: Code Segment, Data Segment, Stack, Heap, and the Garbage Collector](18-go-internal-memory.md)
- [Chapter 19: End of Internal Memory: Function Expressions in Memory, Compilation vs. Execution](19-end-of-internal-memory.md)
- [Chapter 20: Closures: The Magic of Function Memory](20-closures.md)

### Part 4: Structs, Methods, and Data Structures (Chapters 21-25)
Your own types and Go's core collections.

- [Chapter 21: Structs: Custom Types, Objects, and Properties](21-structs.md)
- [Chapter 22: Receiver Functions (Methods): Adding Behavior to Types](22-receiver-functions-methods.md)
- [Chapter 23: Arrays: The Flower Garland Analogy](23-arrays.md)
- [Chapter 24: Pointers: Understanding Memory Addresses](24-pointers.md)
- [Chapter 25: Slices: The Most Important Interview Topic](25-slices.md)

### Part 5: Computer Architecture and Operating Systems (Chapters 26-32)
The systems knowledge behind concurrency.

- [Chapter 26: Computer Architecture and a Short History of Computing](26-computer-architecture-and-a-short-history-of-computing.md)
- [Chapter 27: Introduction to Operating Systems: The Birth of Automation](27-introduction-to-operating-systems.md)
- [Chapter 28: Breaking the CPU and Understanding the Process](28-breaking-the-cpu-and-understanding-the-process.md)
- [Chapter 29: SP vs. BP: The Stack Pointer Dance](29-sp-vs-bp.md)
- [Chapter 30: Context Switching, the PCB, and the Magic of Concurrency](30-context-switching-the-pcb-and-the-magic-of-concurrency.md)
- [Chapter 31: Concurrency vs. Parallelism: The Multi-Core Revolution](31-concurrency-vs-parallelism.md)
- [Chapter 32: Threads: The "Virtual Process"](32-threads.md)

### Part 6: Advanced Go Concepts (Chapters 33-36)
Data types in depth, `defer`, and goroutines.

- [Chapter 33: Data Types in Depth: Integers, Floats, Booleans, Bytes, Runes, and Strings](33-data-types-in-depth.md)
- [Chapter 34: `defer`: Delaying Work Until a Function Returns](34-defer.md)
- [Chapter 35: A Separate Stack for Every Thread](35-a-separate-stack-for-every-thread.md)
- [Chapter 36: Goroutines: Complex and Beautiful](36-goroutines.md)

### Part 7: Backend Development Fundamentals (Chapters 37-39)
How the web works, what REST means, and what the Go runtime does.

- [Chapter 37: Into Backend Development: How the Web Works, and What REST Really Means](37-into-backend-development.md)
- [Chapter 38: OS or Go Server? The Complete Journey of a Request](38-os-or-go-server-the-complete-journey-of-a-request.md)
- [Chapter 39: The Go Runtime: The Heart of Go](39-the-go-runtime.md)

### Part 8: E-commerce Project: HTTP and Routing (Chapters 40-46)
Build a real REST API with `net/http`, JSON, CORS, routing and a middleware pipeline.

- [Chapter 40: E-commerce Project: Your First Real Endpoint (`GET /products`)](40-e-commerce-project-get-products.md)
- [Chapter 41: E-commerce Project: `POST /products` (Creating Data), JSON Decoding, and CORS](41-e-commerce-project-post-products.md)
- [Chapter 42: The Preflight Request and the `OPTIONS` Method: The Browser's Security Guard](42-the-preflight-request-and-the-options-method.md)
- [Chapter 43: Refactoring the Codebase: Clean Code and the Single Responsibility Principle](43-refactoring-the-codebase.md)
- [Chapter 44: Advanced Routing (Go 1.22+) and the Middleware Idea](44-advanced-routing-go-1-22-and-the-middleware-idea.md)
- [Chapter 45: Building Your First Real Middleware: A Request Logger](45-building-your-first-real-middleware.md)
- [Chapter 46: Advanced Middleware: Understanding the Request Pipeline](46-advanced-middleware.md)

### Part 9: Configuration and Authentication (Chapters 47-49)
Twelve-factor configuration, graceful shutdown, JWT and protected routes.

- [Chapter 47: A Real Project Structure: Configuration Management, Timeouts, and Graceful Shutdown](47-a-real-project-structure.md)
- [Chapter 48: Authentication with JWT: Users, Passwords, and Signed Tokens](48-authentication-with-jwt.md)
- [Chapter 49: Authentication Middleware: Verifying Tokens and Protecting Your APIs](49-authentication-middleware.md)

### Part 10: Clean Architecture and Design Patterns (Chapters 50-51)
Decoupling with dependency injection and interfaces.

- [Chapter 50: Removing Tight Coupling: Feature-Based Structure and Dependency Injection](50-removing-tight-coupling.md)
- [Chapter 51: Interfaces and Design Patterns: The Power of Abstraction](51-interfaces-and-design-patterns.md)

### Part 11: Database Integration (Chapters 52-58)
PostgreSQL from connection pool to migrations.

- [Chapter 52: Connecting to PostgreSQL: Infrastructure, `database/sql`, and Connection Pools](52-connecting-to-postgresql.md)
- [Chapter 53: Users in PostgreSQL: Tables, `INSERT`, `SELECT`, and a Real Store](53-users-in-postgresql.md)
- [Chapter 54: PostgreSQL Data Types: Choosing the Right Type for Every Column](54-postgresql-data-types.md)
- [Chapter 55: SQL CRUD: `SELECT`, `INSERT`, `UPDATE`, `DELETE` in Depth](55-sql-crud.md)
- [Chapter 56: CRUD in Go: A PostgreSQL Product Store with `Update` and `Delete`](56-crud-in-go.md)
- [Chapter 57: Database Configuration: DSNs, TLS, Timeouts, Precedence, and Docker Compose](57-database-configuration.md)
- [Chapter 58: Database Migrations: Versioned, Repeatable, Safe Schema Changes](58-database-migrations.md)

### Part 12: Domain-Driven Design (Chapters 59-61)
Organizing code around the business: entities, value objects, ports and adapters.

- [Chapter 59: Domain-Driven Design: Organizing Code Around the Business](59-domain-driven-design.md)
- [Chapter 60: DDD in Code, Part 1: The User Domain](60-ddd-in-code-part-1.md)
- [Chapter 61: DDD in Code, Part 2: The Product Domain (and Retiring `models`)](61-ddd-in-code-part-2.md)

### Part 13: Performance and Pagination (Chapters 62-63)
Measure first, then fix: load testing and pagination.

- [Chapter 62: Experiments Before Optimizing: Why Returning Everything Breaks Down](62-experiments-before-optimizing.md)
- [Chapter 63: Pagination: Query Parameters, Offsets, Totals, and the Trade-offs](63-pagination.md)

### Part 14: Concurrency Deep Dive (Chapters 64-70)
Goroutines, synchronization, races, locks and channels, applied to the project.

- [Chapter 64: Why Concurrency Matters: Goroutines, Waiting, and Server Capacity](64-why-concurrency-matters.md)
- [Chapter 65: `sync.WaitGroup`: Waiting for Goroutines the Right Way](65-sync-waitgroup.md)
- [Chapter 66: Inside `sync.WaitGroup`: One Atomic Word and a Semaphore](66-inside-sync-waitgroup.md)
- [Chapter 67: Race Conditions: When Goroutines Share Data](67-race-conditions.md)
- [Chapter 68: `sync.Mutex`: Protecting Shared Data](68-sync-mutex.md)
- [Chapter 69: Channels: Passing Data Between Goroutines](69-channels.md)
- [Chapter 70: Channels and Goroutines in Practice: A Concurrent Product List (and Course Wrap-Up)](70-channels-and-goroutines-in-practice.md)

## 🧭 Suggested paths

| If you are… | Do this |
|-------------|---------|
| **New to programming** | Chapters 1-25 in order (type every example), then 26-39 for how computers and servers work, then the project |
| **Coming from another language** | Skim 1-17, read 18-25 carefully (memory, closures, slices, pointers), then 33-36, then the project |
| **Already writing Go** | Jump to 40-58 for the project stack, then 59-61 (DDD) and 62-70 (performance and concurrency); use earlier chapters as reference |
| **Preparing for interviews** | Every chapter's *Interview questions*; especially 18-25, 33-36, 51, 59, 64-70 |
| **Focused on concurrency** | 28-32, 36, 39, then 64-70 |
| **Focused on backend and databases** | 37-63 |

## 🚀 What you will build

An **e-commerce backend API**, evolving chapter by chapter (each stage compiled and tested):

```
ecommerce/
├── main.go, cmd/            start-up: commands (serve, migrate), composition root, graceful shutdown
├── config/                  validated configuration (12-factor), redacted secrets
├── auth/                    bcrypt, hand-built HS256 JWT (cross-checked against a library in tests)
├── user/  product/          DOMAINS: entities, value objects, rules, ports, services (no HTTP/SQL imports)
├── rest/                    HTTP adapters: handlers, JSON, status codes, routing
├── middleware/              request ID, logging, CORS, recover, JSON errors, authentication
├── postgres/  database/     repository adapters: PostgreSQL (sqlx) and in-memory (fast fake)
├── migrations/              versioned SQL (goose), embedded in the binary
├── infra/db, infra/migrate  connection pool, timeouts, migrations runner, test databases
├── health/                  /healthz (liveness) and /readyz (readiness)
└── tools/loadgen            a small load generator with percentiles (Chapter 62)
```

Features: REST endpoints (`GET/POST/PUT/DELETE`), JWT authentication, PostgreSQL with migrations, paginated listing with concurrent count/list queries, structured logging, graceful shutdown, health checks, and a test suite that includes contract tests (both storage implementations behave identically), integration tests (real PostgreSQL), architecture tests (dependency rules), and race-detector runs.

## 📖 Prerequisites and setup

- **Knowledge:** none beyond basic computer use. Programming experience in any language helps but is not required.
- **Go:** version **1.22 or newer** for the language features used (dependencies added along the way may raise the `go` line in `go.mod`, as Chapters 48 and 52 explain); **1.25** for `sync.WaitGroup.Go` in Chapter 65. Install from <https://go.dev/dl/>.
- **Docker** (Chapters 52-63 and 70): the course runs PostgreSQL 16 in a container (`postgres:16-alpine`). Chapter 52 shows the command; use any free local port (the chapters use `15432`) so you don't clash with a database you already have.
- **`curl`** to call the API (Chapters 40+); optionally `jq`, and a graphical client such as Postman, Insomnia or Bruno.
- **An editor** with Go support (VS Code with the Go extension, GoLand, or Neovim with `gopls`).
- Chapters that start the server use a **non-default port** in their examples (`18080`) to avoid conflicts with other software on your machine; change it freely.

Useful commands you will meet throughout:

```bash
go run .                        # run the program in the current directory
go build ./...                  # compile everything
go vet ./...                    # static checks (copied locks, printf mistakes, ...)
gofmt -l .                      # list files that are not formatted
go test ./...                   # run tests
go test -race ./...             # run tests with the data-race detector (always, for concurrent code)
go test -bench . -benchmem      # run benchmarks
```

## 💡 Philosophy

- **Understand, then use.** Every tool is introduced with the problem it solves, often by first showing what goes wrong without it (a race, an injection, a leak, an outage).
- **Measure, don't guess.** Performance and concurrency chapters use experiments and report real numbers, including results that disappoint.
- **One rule, one place.** The project is refactored toward code where each business rule and each technical concern lives in exactly one place.
- **Tests are part of the design.** You'll see how fakes, contract tests, and the race detector turn correctness from hope into evidence, and how to prove a test *can* fail.
- **Be honest about trade-offs.** Layers, patterns, and concurrency all have costs; the chapters say when *not* to use them.

## 🎓 Course completion

After Chapter 70 you will be able to:

- write idiomatic Go, and explain its memory model, closures, slices, pointers and error handling;
- build and test an HTTP API with routing, middleware, authentication and graceful shutdown;
- design a layered, testable codebase (dependency injection, ports and adapters, DDD-lite);
- use PostgreSQL safely from Go (parameterized SQL, pools, transactions basics, migrations, constraints);
- measure and fix performance problems (pagination, load testing, reading query plans);
- write concurrent code correctly: goroutines, `WaitGroup`, mutexes, atomics, channels, contexts, and detect races.

## 🔜 Where to go next

- Observability (structured logs, metrics, tracing, `pprof`)
- Deployment (multi-stage Docker builds, Kubernetes, CI/CD)
- Security beyond login (authorization, rate limiting, `govulncheck`)
- Transactions and isolation levels in depth; caching with Redis
- gRPC and message queues
- Generics and the newest language features
- Reading the standard library and well-known Go projects

## 🛠 Reproducing the verification

The project stages used to verify the chapters can be rebuilt by concatenating each project chapter's `// file: path` code blocks (Chapters 40-63 and 70 each state which files they add or replace, and which they delete). If a chapter's output differs slightly on your machine (timings, memory, timestamps, ports), that is expected; the *shape* of the results is what matters.

## 📝 License

This educational content is provided as-is for learning purposes.

---

**Happy learning! 🎉** *Take your time. Understanding deeply is worth more than finishing quickly.*
