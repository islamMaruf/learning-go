# Chapter 59: Domain-Driven Design — Organizing Code Around the Business

> **Goal of this chapter:** Understand *why* codebases turn into tangled messes as they grow, and learn the ideas of **Domain-Driven Design (DDD)** that prevent it: domains and **bounded contexts**, a **ubiquitous language**, **entities**, **value objects**, **aggregates**, **repositories**, and **services**, plus the layering rules (dependencies point *inward*) that keep business rules independent of HTTP and SQL. This chapter is conceptual (no code changes); Chapters 60 and 61 apply everything to the user and product features. We also stay honest about when full DDD is **too much**.

**Difficulty:** 🔴 Advanced  **Estimated time:** 4 hours  **Prerequisite:** [Chapters 51 and 58](58-database-migrations.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The problem: code organized around technology](#2-the-problem-code-organized-around-technology)
3. [What is Domain-Driven Design?](#3-what-is-domain-driven-design)
4. [Finding the domains: a worked example](#4-finding-the-domains-a-worked-example)
5. [Domain independence](#5-domain-independence)
6. [Strategic DDD: bounded contexts and ubiquitous language](#6-strategic-ddd-bounded-contexts-and-ubiquitous-language)
7. [Tactical DDD: the building blocks](#7-tactical-ddd-the-building-blocks)
8. [Layers and the dependency rule](#8-layers-and-the-dependency-rule)
9. [Where our project stands today](#9-where-our-project-stands-today)
10. [The target structure](#10-the-target-structure)
11. [Benefits, and the costs nobody mentions](#11-benefits-and-the-costs-nobody-mentions)
12. [When NOT to use DDD](#12-when-not-to-use-ddd)
13. [DDD in Go: idioms](#13-ddd-in-go-idioms)
14. [Common mistakes](#14-common-mistakes)
15. [Interview questions](#15-interview-questions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- Why "layer by technology" folders (`controllers/`, `models/`, `repositories/`) hurt as systems grow
- What a **domain** is, and how to discover domains from a business description
- **Bounded contexts** and why the word "customer" can legitimately mean different things
- **Ubiquitous language**: using the business's own words in code
- **Entities, value objects, aggregates, repositories, domain services**, with Go examples
- **The dependency rule:** business logic depends on nothing; HTTP and SQL depend on it
- How to judge whether DDD fits your project (often only *some* of it does)

---

## 2. The problem: code organized around technology

Most beginner projects (ours included, so far) grow like this:

```
handlers/   ← every HTTP handler
models/     ← every struct
database/   ← every SQL query
utils/      ← everything else
```

This is **layering by technology**. It feels tidy at 10 files. At 500 files:

- To change *how discounts work*, you edit `handlers/order.go`, `models/order.go`, `database/order.go`, `utils/pricing.go`, and three more. **One business idea is scattered across the codebase.**
- Any package can call any other. A handler runs SQL directly; a "util" imports the HTTP layer. Nobody knows what depends on what, so nobody dares delete anything.
- Business rules ("a price must be positive", "a user cannot review their own product") hide inside HTTP handlers, so they can't be tested without an HTTP server and can't be reused by a batch job or CLI.
- The vocabulary of the code (`Row`, `Payload`, `Manager`, `Helper`) has nothing to do with the vocabulary of the business people who ask for changes.

A concrete symptom you have probably felt already in this course: `user.Handler` (Chapter 51) validates passwords, hashes them, checks for duplicates, and writes JSON, all in one type. Try to answer "what are the rules for registering a user?" and you must read HTTP code.

```
"Spaghetti": everything depends on everything

   handlers ◄────► models ◄────► database
      ▲    ╲        ▲   ╲         ▲
      │     ╲       │    ╲        │
      ▼      ▼      ▼     ▼       ▼
     utils ◄──────────────────► config
```

We want structure where a change to *one business capability* touches *one place*, and where the direction of every dependency is deliberate.

---

## 3. What is Domain-Driven Design?

**Domain-Driven Design** (Eric Evans, 2003) is an approach to building software whose central idea is:

> **The structure and language of the code should mirror the business domain, not the technology.**

The **domain** is the subject area your software is about: for us, *selling products online*. Within it are **subdomains** (users and accounts, product catalog, orders, payments, shipping). DDD asks you to:

1. **Talk to domain experts** and learn their vocabulary.
2. **Model** the domain's concepts and rules explicitly in code.
3. **Isolate** that model from technical concerns (databases, web frameworks, message queues).
4. **Draw boundaries** between parts of the domain so each can evolve independently.

DDD has two halves:

| Half | Concerns | Examples |
|------|----------|----------|
| **Strategic DDD** | the big picture: how to split a large system | subdomains, bounded contexts, ubiquitous language, context maps |
| **Tactical DDD** | the code inside one context | entities, value objects, aggregates, repositories, services, domain events |

Many teams only need a *light* version of the tactical half plus the dependency rule. We'll use exactly that, and say so when a concept is optional.

---

## 4. Finding the domains: a worked example

Take a simple sentence about a social network:

> *"A user writes a post. Other users can like and comment on it. Followers see the post in their feed, and the author gets a notification."*

Underline the **nouns** (candidate concepts) and the **verbs** (candidate behaviors):

```
user, post, like, comment, follower, feed, author, notification
writes, likes, comments, follows, sees, notifies
```

Group concepts that change *for the same reasons* and are talked about *by the same people*:

```
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐  ┌────────────────┐
│  Identity  │  │   Posts    │  │ Engagement │  │     Feed     │  │ Notifications  │
│ user       │  │ post       │  │ like       │  │ timeline     │  │ notification   │
│ follower   │  │ author     │  │ comment    │  │ ranking      │  │ delivery prefs │
└────────────┘  └────────────┘  └────────────┘  └──────────────┘  └────────────────┘
```

Now apply the same exercise to **our shop**:

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Identity   │  │   Catalog    │  │    Orders    │  │   Payments   │  │  Shipping    │
│ user         │  │ product      │  │ cart, order  │  │ payment      │  │ shipment     │
│ credentials  │  │ price, image │  │ line item    │  │ refund       │  │ address      │
│ token        │  │ category     │  │ total        │  │ provider     │  │ carrier      │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
    (built)           (built)           (future)          (future)          (future)
```

Two of these exist today (Identity = our `user`, Catalog = our `product`). The others will arrive. **If each domain has its own home and clear borders, adding "Orders" doesn't require touching "Catalog"'s internals**, only asking it a question through a defined door ("what is the price of product 7?").

### Domain relationships

Domains *do* need each other, and the relationships have direction:

```
Orders ───needs price of──► Catalog          (Orders depends on Catalog)
Orders ───needs owner of──► Identity         (Orders depends on Identity)
Payments ─needs total of──► Orders
Catalog  ─knows nothing of─ Orders           (a product doesn't care whether it's ever ordered)
```

Rule of thumb: **the more fundamental domain must not depend on the more specific one.** Catalog and Identity are foundations; Orders builds on them. If `Catalog` imported `Orders`, you could never change how orders work without risking the catalog.

---

## 5. Domain independence

The core principle: **a domain should be able to change (or be replaced) without breaking the others.** The human body is a good analogy:

- The **heart** pumps blood; it doesn't know how the **lungs** work. It only relies on the *interface* "oxygenated blood arrives".
- You can have a heart transplant (swap the implementation) because the connections (vessels) are standardized.
- If the heart's internals changed, the lungs need no change, as long as the interface is honored.

In code, "the interface" is a Go interface, a function signature, or a narrow API:

```go
// Orders needs a price; it asks Catalog through a small door it defines itself.
type PriceLookup interface {
	PriceOf(ctx context.Context, productID int) (float64, error)
}
```

`orders` knows nothing about catalog tables, HTTP handlers, or caching. Tomorrow `PriceLookup` might be a local function call; next year a call to a separate catalog service. Orders' code doesn't change.

### Business-logic isolation: the payoff

Consider the rule *"a product's price must be greater than zero"*. In the code we have now, it appears in the HTTP request type. That means:

- A CSV import job or an admin CLI that creates products must **re-implement** the rule, or forget it.
- Testing the rule requires constructing an HTTP request.
- The rule and the JSON field names are mixed together.

With DDD the rule lives in the **domain** (`product.Product.Validate`). HTTP, CLI, and import jobs all call the domain, and the rule exists exactly **once**.

---

## 6. Strategic DDD: bounded contexts and ubiquitous language

### Ubiquitous language

Developers and domain experts must use **the same words**, in conversation *and in code*. If the business says "a customer *places* an order" and "an order is *fulfilled*", then the code should have `order.Place()` and `order.Fulfill()`, not `orderManager.processData(row)`.

```go
// Business-speak in code
func (o *Order) Place() error
func (o *Order) Cancel(reason string) error

// Technology-speak in code (avoid)
func (m *OrderManager) InsertAndUpdateStatusFlag(o *OrderRow, s int) error
```

When a term is ambiguous, *settle it once* and write it down (a glossary in the README is enough). Half of all miscommunication bugs are two people using one word for two things.

### Bounded context

The same word can mean different things in different parts of the business. That's **not a bug to fix**; it's a boundary to respect.

| Word | In **Catalog** it means… | In **Shipping** it means… | In **Billing** it means… |
|------|------------------------|---------------------------|--------------------------|
| **Product** | a sellable item with title, description, images, price | a physical parcel with weight and dimensions | a line on an invoice with tax class |
| **Customer** | (not relevant) | a delivery address and phone | a legal entity with a tax ID |

A **bounded context** is the explicit boundary inside which a term has *one* meaning and one model. Trying to build a single giant `Product` struct serving all three contexts produces a 60-field monster where every team steps on the others. Instead, each context has its own small `Product` model, and they exchange only the data they need (usually by ID).

Practical form in Go: **one package (or a small tree) per bounded context**, with its own types, and communication through narrow interfaces or IDs. For a modular monolith like ours, contexts are packages; for microservices, they're services.

---

## 7. Tactical DDD: the building blocks

### Entity: identity matters

An **entity** is defined by its **identity** and lifecycle, not by its attribute values. Two users with the same name are still different users; a user who changes their email is still the same user.

```go
type User struct {
	ID           int       // identity
	Email        string
	PasswordHash string
	CreatedAt    time.Time
}
```

Entities carry the **behavior** that belongs to them:

```go
func (u User) CanLogInWith(check func(hash, plain string) bool, plain string) bool { … }
```

### Value object: value matters

A **value object** has no identity: it's defined entirely by its values and is **immutable**. Two `Money{Amount: 500, Currency: "USD"}` values are interchangeable. Value objects are how you stop passing raw `string`s and `float64`s around ("primitive obsession") and enforce rules **once**, at construction:

```go
package user

import (
	"errors"
	"net/mail"
	"strings"
)

// Email is a valid, normalized email address. The zero value is not valid.
type Email struct{ value string }

var ErrInvalidEmail = errors.New("invalid email address")

// NewEmail validates and normalizes s. Any Email in the program is therefore known to be valid.
func NewEmail(s string) (Email, error) {
	s = strings.ToLower(strings.TrimSpace(s))
	addr, err := mail.ParseAddress(s)
	if err != nil || addr.Address != s {
		return Email{}, ErrInvalidEmail
	}
	return Email{value: s}, nil
}

func (e Email) String() string { return e.value }
```

Because the field is unexported, **the only way to obtain an `Email` is through `NewEmail`**. A function that accepts `Email` doesn't have to re-validate. The type system carries the guarantee ("make illegal states unrepresentable").

Similarly `Money` should hold integer cents plus a currency and offer `Add` that refuses mixed currencies, instead of a naked `float64` price (recall Chapter 54's warning).

| | Entity | Value object |
|---|--------|--------------|
| Defined by | identity (ID) | its attributes |
| Mutable? | yes, over its lifecycle | no: replace, don't modify |
| Equality | same ID | all fields equal (Go `==` works on comparable structs) |
| Examples | `User`, `Order`, `Product` | `Email`, `Money`, `Address`, `DateRange` |

### Aggregate: a consistency boundary

An **aggregate** is a cluster of objects treated as **one unit** for changes, with a single entry point called the **aggregate root**. Outsiders may hold a reference to the root only; they change inner parts *through* the root, which enforces the rules ("invariants") for the whole cluster.

```
Order (aggregate root)
 ├── OrderLine (product ID, quantity, unit price)
 ├── OrderLine
 └── ShippingAddress (value object)

Rule (invariant): an order's total = sum of its lines, and it must have at least one line.
```

```go
func (o *Order) AddLine(productID, qty int, unit Money) error {
	if o.status != Draft {
		return ErrOrderLocked
	}
	if qty < 1 {
		return ErrInvalidQuantity
	}
	o.lines = append(o.lines, OrderLine{productID, qty, unit})
	return nil
}
```

Nobody reaches into `order.lines` directly; the root guards them. Practical consequences: **load and save the whole aggregate together** (one repository per aggregate), keep aggregates *small*, and refer to other aggregates **by ID**, not by embedding them.

### Repository: a collection-like door to storage

A **repository** presents an aggregate store as if it were an in-memory collection: `Save`, `FindByID`, `Delete`. The domain defines the **interface**; the infrastructure (PostgreSQL) provides the implementation. You have already built this pattern in Chapters 51-56 (`Store`), we are just going to name it properly and move it to the right place.

### Domain service: rules that don't fit one entity

Some business operations don't belong to a single entity: "register a user" involves checking uniqueness (repository), hashing a password (a technical service), and creating an entity. A **service** orchestrates that. Keep the services **thin** (coordination) and put rules that are about *one* object *on that object*. If every rule ends up in services and entities are just bags of fields, you get an **anemic domain model**, one of the most common DDD failures (and, fair warning, a very common state of Go code that says it does DDD).

### Domain events (optional, advanced)

Facts that happened ("OrderPlaced", "UserRegistered") published so other parts react (send an email, reduce stock) without the origin knowing them. Mentioned here so you recognize the term; we don't need it yet.

---

## 8. Layers and the dependency rule

Strategic and tactical ideas come together in one rule:

> **Source-code dependencies point inward, toward the domain. The domain depends on nothing outside itself.**

```
        ┌────────────────────────────────────────────────────────┐
        │  Outer: frameworks, I/O, delivery                       │
        │   HTTP handlers · CLI · PostgreSQL · JWT · bcrypt       │
        │                                                        │
        │      ┌──────────────────────────────────────┐          │
        │      │  Application / service layer          │          │
        │      │  use cases: Register, Login, Place... │          │
        │      │                                      │          │
        │      │     ┌────────────────────────────┐   │          │
        │      │     │  DOMAIN                     │   │          │
        │      │     │  entities, value objects,   │   │          │
        │      │     │  rules, ports (interfaces)  │   │          │
        │      │     └────────────────────────────┘   │          │
        │      └──────────────────────────────────────┘          │
        └────────────────────────────────────────────────────────┘

   imports flow  ─────────►  inward only   (HTTP → service → domain,  PostgreSQL → domain)
```

How does the domain use a database without depending on it? **Ports and adapters** (also called *hexagonal architecture*): the inner layer declares a **port** (a Go interface describing what it needs) and the outer layer supplies an **adapter** implementing it.

```
   HTTP handler ──calls──►  user.Service  ──uses port──►  user.Repository (interface)
   (adapter, outer)         (inner)                          ▲
                                                             │ implements
                                                   postgres.UserStore (adapter, outer)
```

The compile-time dependency arrow for the database points *inward* (`postgres` imports `user`), even though at run time data flows outward. This is Chapter 51's **dependency inversion** applied at architecture scale.

### The golden rules

1. **The domain never imports outer layers**: no `net/http`, no `database/sql`, no JSON tags mattering to the rules.
2. **Outer layers may import inner ones**, never the reverse.
3. **Domains reach each other only through small interfaces or IDs**, never through each other's internals.
4. **Wire everything together in one place** (the composition root, `cmd/`), which is the only code allowed to know all the concrete types.

A cheap guardrail: a test that fails the build if the domain imports a forbidden package (see Exercise 6).

---

## 9. Where our project stands today

An honest audit of the current code:

| Concern | Today | DDD verdict |
|---------|-------|-------------|
| Storage behind interfaces | ✅ `Store` interfaces, two implementations, contract tests | Good start (these are our repository ports) |
| Business rules location | ❌ password/email/price/title rules live in **HTTP handlers/request types** | Rules should be in the domain |
| Orchestration ("register = validate + hash + save") | ❌ inside `user.Handler.Register` | Belongs to a service |
| Entities | ⚠️ `models.User`/`models.Product` are shared, JSON-and-DB-tagged structs in a common `models` package | A shared "models" bucket couples every feature; each domain should own its entities |
| Package = feature | ✅ `product`, `user` are already per-feature | Good |
| Wiring in one place | ✅ `cmd/wire.go` | Good |
| Handlers testable without a DB | ✅ via fake stores | Good, but rules still need HTTP to be tested |

So the plan is a **targeted refactor**, not a rewrite: move the rules and orchestration inward, give each domain its own entities, and keep handlers thin. We keep every behavior and every HTTP test.

---

## 10. The target structure

```
ecommerce/
├── main.go, cmd/            ← composition root (wires concrete types together)
│
├── user/                    ← DOMAIN "Identity"  (Chapter 60)
│   ├── user.go                entity, value rules, NormalizeEmail
│   ├── errors.go              domain errors: ErrNotFound, ErrEmailTaken, ValidationError...
│   ├── port.go                ports: Repository, PasswordHasher, TokenIssuer
│   ├── service.go             use cases: Register, Login, Get
│   └── service_test.go        fast unit tests with fakes (no HTTP, no DB)
│
├── product/                 ← DOMAIN "Catalog"  (Chapter 61)
│   ├── product.go, errors.go, port.go, service.go, service_test.go
│
├── rest/                    ← ADAPTER: HTTP
│   ├── userhandler/           JSON <-> user.Service
│   ├── producthandler/        JSON <-> product.Service
│   └── server.go              routes + middleware
│
├── postgres/                ← ADAPTER: PostgreSQL implements user.Repository, product.Repository
├── database/                ← ADAPTER: in-memory implementations (tests, demos)
├── auth/                    ← ADAPTER: bcrypt hasher, JWT issuer
├── infra/, middleware/, util/, config/ …
```

The handler packages and the storage packages **import** the domain packages; the domain packages import **nothing** of ours except each other's narrow types (and even that is avoided where possible).

---

## 11. Benefits, and the costs nobody mentions

### Benefits

| Benefit | How |
|---------|-----|
| **One home per business rule** | change "password rules" in one file; HTTP, CLI, tests all follow |
| **Testability** | domain logic is plain Go with fakes: milliseconds, no server, no database |
| **Replaceable technology** | swap PostgreSQL for another store, REST for gRPC, bcrypt for argon2, changing only adapters |
| **Parallel teams** | the Catalog team and the Identity team rarely conflict; interfaces are the contract |
| **Onboarding** | new developers find the business rules where the business names them |
| **Smaller blast radius** | a change to one domain can't accidentally break another's internals |

### Costs (be honest)

| Cost | Detail |
|------|--------|
| **More files and indirection** | a simple "create product" now crosses handler → service → repository; a beginner may ask "why so many layers?" |
| **Mapping code** | HTTP DTO ↔ domain entity ↔ database row: three shapes of "product", with conversion between them |
| **Up-front design time** | naming things properly, drawing boundaries |
| **Risk of ceremony** | interfaces with a single implementation, services that only forward calls |
| **Wrong boundaries are expensive** | if you draw contexts badly, changes cut *across* them, which is worse than no boundaries |

These costs are worth paying when the system is **large, long-lived, and changing**, and wasteful when it's a small CRUD app. That leads to the next section.

---

## 12. When NOT to use DDD

- **Simple CRUD**: if the "domain logic" is "save what the user sent", a plain handler + repository is fine. (Ironically, our product API is close to this: the DDD refactor is partly to *teach* the structure.)
- **Prototypes and throwaway tools**: you'll learn the domain by building; structure it later.
- **Tiny teams on tiny systems**: the coordination benefits don't exist yet.
- **Data-pipeline / reporting code**: the "domain" is transformations, not behaviors.

A useful test: *"Are there rules that experts argue about and that change over time?"* (pricing, eligibility, refunds, permissions) → invest in a rich domain model. *"Is it forms and tables?"* → keep it simple. Mature codebases mix both: rich domain core, plain CRUD at the edges.

The pragmatic middle path we'll follow (sometimes called **"DDD-lite"**): per-domain packages, business rules in the domain, ports for outside things, thin adapters, one composition root, and **no** aggregates/events/CQRS until a real need appears.

---

## 13. DDD in Go: idioms

Go doesn't have classes or inheritance, but it fits the ideas well:

| DDD idea | Go idiom |
|----------|----------|
| Bounded context | a package (or package tree), sometimes a module/service |
| Port | a small **interface**, declared in the *consuming* (inner) package |
| Adapter | a struct in an outer package satisfying the interface implicitly |
| Value object | an unexported-field struct with a constructor (`NewEmail`), or a named type with methods |
| Entity | a struct with an ID; behavior as methods; pointer receivers when it mutates |
| Aggregate root | exported type whose invariants are guarded by unexported fields and methods |
| Domain error | sentinel errors (`var ErrNotFound = errors.New(…)`) and custom error types checked with `errors.Is/As` |
| Service | a struct holding its ports, with methods for each use case |
| Composition root | `main`/`cmd` constructs everything (no DI framework needed) |
| Encapsulation | **unexported identifiers**: the package boundary *is* the encapsulation boundary; use it |
| `internal/` | a directory whose packages only the parent module can import: keeps domains from being imported by outsiders |

Go-specific cautions:

- Avoid **package-level global state** in domains (Chapter 50).
- Avoid **cyclic imports** by moving the shared type down into the domain that owns it, or by passing IDs.
- Don't recreate Java: no `AbstractFactoryProvider`. Small interfaces and plain functions suffice.
- Prefer **composition** (a `Service` struct with fields for its ports) over inheritance-like embedding tricks.

---

## 14. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Anemic domain: entities are bags of fields, all logic in services | DDD's ceremony without its benefits | Put rules on the entity/value object they concern |
| 2 | The domain imports `net/http`, `database/sql`, or JSON tags drive its shape | Domain can't be reused or tested alone | Ports and adapters; DTOs at the edge |
| 3 | One giant shared `models` package | Every feature couples to every other | Each domain owns its entities |
| 4 | An interface for everything, each with one implementation | Ceremony, indirection | Interfaces only at real boundaries (storage, hashing, clock, network) |
| 5 | Domains importing each other's internals | Cyclic, tangled dependencies | Interfaces or IDs; more fundamental domains never import specific ones |
| 6 | Huge aggregates (load 50k rows to change one field) | Slow, contended | Small aggregates; reference others by ID |
| 7 | Exposing DB row structs as API responses | Schema changes break clients; leaks columns | Separate DTOs for the wire |
| 8 | Business words missing from code names (`Manager`, `Helper`, `Data`) | Code and business diverge | Ubiquitous language |
| 9 | Applying DDD to a trivial CRUD app | Slow development, no benefit | Match structure to complexity |
| 10 | Big-bang rewrite "to DDD" | Months without shipping | Refactor incrementally, one domain at a time, tests green throughout |
| 11 | Drawing boundaries by *technical* layers again | Same spaghetti in new folders | Draw by *business capability* |
| 12 | Skipping the composition root (domains constructing their own dependencies) | Hidden coupling, untestable | Construct in `cmd`, pass in |

---

## 15. Interview questions

**Q1. What is Domain-Driven Design in one sentence?**
Designing software around a model of the business domain, using its language, with clear boundaries between parts and business rules isolated from technology.

**Q2. Entity vs. value object?**
An entity has identity and a lifecycle (two users with equal fields are still different); a value object is immutable and defined solely by its values (`Money`, `Email`).

**Q3. What is an aggregate?**
A cluster of objects that must stay consistent together, accessed only through its root, which enforces the invariants; it's the unit of loading, saving, and transactions.

**Q4. What is a bounded context?**
An explicit boundary within which a term/model has a single, consistent meaning ("Product" in Catalog vs. Shipping).

**Q5. What is the dependency rule?**
Source-code dependencies point toward the domain; the domain depends on nothing outside itself. Outer layers implement ports the inner layer declares.

**Q6. Ports and adapters?**
The core declares interfaces (ports) for what it needs; infrastructure supplies implementations (adapters): databases, HTTP, message queues.

**Q7. What's an anemic domain model?**
Entities with data but no behavior, and all rules in service classes: an anti-pattern that loses encapsulation.

**Q8. When would you not use DDD?**
Simple CRUD, prototypes, tiny systems; the overhead outweighs the benefit.

**Q9. How does Go's package system help?**
Unexported identifiers give real encapsulation at the package boundary; `internal/` restricts importers; implicit interfaces let outer packages satisfy domain ports without importing them.

---

## 16. Exercises

### Exercise 1: Find the domains
Describe a **library system** (books, members, loans, fines, reservations). List nouns and verbs, group them into contexts, and draw the dependency arrows. Which context is the most "fundamental"?

<details><summary>Solution</summary>

Contexts: **Catalog** (book, author, copy, ISBN), **Membership** (member, card, status), **Lending** (loan, due date, renewal), **Fines** (fine, payment), **Reservations**. Catalog and Membership are foundations; Lending depends on both; Fines depends on Lending; Reservations depends on Catalog, Membership and Lending. Arrows point from specific to fundamental, never back.
</details>

### Exercise 2: Spot the bounded contexts
The word "account" appears in banking (a balance), identity (a login), and accounting (a ledger category). Sketch one small model for each. Would a single `Account` struct serve all three?

<details><summary>Solution</summary>

Banking: `Account{ID, Balance Money, Overdraft}`; Identity: `Account{ID, Email, PasswordHash, Status}`; Accounting: `Account{Code, Name, Type}`. One struct would need all fields and satisfy nobody. Keep three types in three packages, related by IDs.
</details>

### Exercise 3: Entity or value object?
Classify: `Email`, `User`, `Money`, `Order`, `DateRange`, `Address` (as a delivery destination on an order), `Product`.

<details><summary>Solution</summary>

Entities: `User`, `Order`, `Product` (identity matters). Value objects: `Email`, `Money`, `DateRange`, and `Address` on an order (two identical addresses are interchangeable). (If addresses are managed and edited independently as an address book, they may become entities.)
</details>

### Exercise 4: Write a value object
Implement `Money` with integer `Cents` and `Currency`, a constructor rejecting negative amounts and empty currency, and `Add` that returns an error for mixed currencies. Why can `Money` values be compared with `==`?

<details><summary>Solution</summary>

```go
type Money struct {
	cents    int64
	currency string
}

func NewMoney(cents int64, currency string) (Money, error) {
	if cents < 0 || currency == "" {
		return Money{}, errors.New("invalid money")
	}
	return Money{cents, currency}, nil
}

func (m Money) Add(o Money) (Money, error) {
	if m.currency != o.currency {
		return Money{}, errors.New("currency mismatch")
	}
	return Money{m.cents + o.cents, m.currency}, nil
}
```
Struct types whose fields are all comparable support `==`, matching value-object equality (all fields equal).
</details>

### Exercise 5: Audit a handler
Here is a handler; list every business rule hidden in it and say which layer each belongs to:

```go
func (h *Handler) Checkout(w http.ResponseWriter, r *http.Request) {
	var req struct{ ProductID, Qty int }
	json.NewDecoder(r.Body).Decode(&req)
	if req.Qty < 1 || req.Qty > 10 { http.Error(w, "bad qty", 400); return }
	p, _ := h.db.QueryProduct(req.ProductID)
	total := float64(req.Qty) * p.Price
	if total > 5000 { total *= 0.95 }
	h.db.InsertOrder(userID, req.ProductID, total)
}
```

<details><summary>Solution</summary>

Rules: quantity limits (1-10), pricing (qty × price), a bulk discount over 5000 (business policy!), persistence. JSON decoding and status codes are *delivery* concerns; the quantity limit, total calculation, and discount belong in the **domain** (`Order`/pricing). Persistence is an adapter behind a repository. Errors are also ignored, a separate bug.
</details>

### Exercise 6 (challenge): Enforce the dependency rule with a test
Write a Go test that fails if any file in package `user` imports `net/http` or `database/sql`.

<details><summary>Solution</summary>

Use `go/parser` with `parser.ImportsOnly` on the package directory and check every import path against a forbidden list (`net/http`, `database/sql`, `ecommerce/postgres`, `ecommerce/rest`…). Or run `go list -deps` from the test. Either way, the architecture rule becomes a failing test instead of a wiki page.
</details>

---

## 17. Quiz

1. What is the difference between an entity and a value object?
2. What does an aggregate root protect?
3. Which way do imports point in a DDD layout?
4. What is a port? What is an adapter?
5. Why is "the same word means different things" not necessarily a bug?
6. What's an anemic domain model?
7. Name two situations where DDD is overkill.
8. Where should the rule "price must be positive" live, and why?

<details><summary>Answers</summary>

1. Identity and lifecycle vs. immutability and equality by value.
2. The consistency (invariants) of the whole cluster, since all changes go through it.
3. Toward the domain: outer layers import inner ones, never the reverse.
4. A port is an interface the domain declares; an adapter is an outer-layer implementation of it.
5. Different bounded contexts legitimately model the same word differently; forcing one model couples them.
6. Entities without behavior, with all logic in services.
7. Simple CRUD apps, prototypes, tiny systems.
8. In the domain (on the product), so every entry point (HTTP, CLI, import) shares it and it's testable without HTTP.
</details>

---

## 18. Summary

- **Layering by technology** scatters each business idea over many folders and lets everything depend on everything. **DDD organizes code around business capabilities** and their language.
- **Strategic DDD:** split the system into **bounded contexts** (domains) with one consistent model each; use a **ubiquitous language**; let dependencies point from specific domains to fundamental ones.
- **Tactical DDD:** **entities** (identity), **value objects** (immutable values that enforce rules at construction), **aggregates** (consistency boundaries with a root), **repositories** (collection-like ports to storage), **services** (thin orchestration); avoid the **anemic model**.
- **The dependency rule:** dependencies point inward; the domain imports nothing technical. **Ports and adapters** let PostgreSQL, HTTP, bcrypt, and JWT plug in from outside; the **composition root** wires them.
- Our project already has repository-style interfaces and per-feature packages; the missing piece is moving **rules and orchestration** out of HTTP handlers into domain services, and giving each domain its **own entities**.
- DDD has real costs; use the **"DDD-lite"** subset (domain packages, services, ports, thin adapters) and skip aggregates/events until needed.

### ➡️ What's next?

[Chapter 60](60-ddd-in-code-part-1.md) refactors the **user domain**: entity, errors, ports, and a `Register/Login/Get` service with fast unit tests, while the HTTP behavior stays exactly the same. [Chapter 61](61-ddd-in-code-part-2.md) repeats the recipe for **products**.
