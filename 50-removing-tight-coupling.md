# Chapter 50: Removing Tight Coupling — Feature-Based Structure and Dependency Injection

> **Goal of this chapter:** Restructure the project the way real production Go services are organized. You'll diagnose **tight coupling** (handlers reaching into package-level globals), learn the vocabulary of **coupling and cohesion**, reorganize the code by **feature** (`product/`, `user/`) instead of by technical layer, turn the global in-memory "database" into **injected store objects**, and wire everything together in one place, the **composition root**. The payoff is measurable: every test gets its own isolated world, tests can run in parallel, and swapping the storage (Chapter 52's PostgreSQL) will no longer require editing handlers.

**Difficulty:** 🔴 Advanced  **Estimated time:** 5 hours  **Prerequisite:** [Chapters 22, 43, 49](49-authentication-middleware.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Diagnosing the current problems](#2-diagnosing-the-current-problems)
3. [Coupling and cohesion](#3-coupling-and-cohesion)
4. [Organizing by layer vs. by feature](#4-organizing-by-layer-vs-by-feature)
5. [The target structure](#5-the-target-structure)
6. [Dependency injection, explained](#6-dependency-injection-explained)
7. [Step 1: stores become objects](#7-step-1-stores-become-objects)
8. [Step 2: the `product` feature](#8-step-2-the-product-feature)
9. [Step 3: the `user` feature](#9-step-3-the-user-feature)
10. [Step 4: the `rest` server](#10-step-4-the-rest-server)
11. [Step 5: the composition root](#11-step-5-the-composition-root)
12. [Tests: isolation for free](#12-tests-isolation-for-free)
13. [Before vs. after](#13-before-vs-after)
14. [What is still coupled](#14-what-is-still-coupled)
15. [Common mistakes](#15-common-mistakes)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- What **coupling** and **cohesion** mean, and how to recognize tight coupling in Go code
- Why **package-level globals** make code hard to test and change
- **Feature-based** package organization vs. **layer-based**, and the trade-offs
- **Dependency injection (DI)** with plain Go: constructors and struct fields, no framework
- The **composition root**: the one place where objects are created and connected
- How to give **each test its own isolated instance** of the app

---

## 2. Diagnosing the current problems

Everything works, and 100% of tests pass. Yet if we honestly review the structure, several smells appear. Name them first:

### Problem 1: handlers reach into globals

```go
// handlers/product.go
func GetProducts(w http.ResponseWriter, r *http.Request) {
	util.SendData(w, http.StatusOK, database.ListProducts())   // ← a package-level function backed by package-level variables
}
```

`database.ListProducts()` secretly reads a hidden global slice. The handler's *signature* (`w, r`) says nothing about needing a database. Consequences:

- **Can't substitute the storage.** To test the handler against fake data, or to switch to PostgreSQL, we must edit `handlers`.
- **Tests share state.** Every test mutates the *same* global list; we needed `database.ResetProducts()` before each test, and tests can't safely run in parallel.
- **Hidden dependencies.** You can't tell what a function needs by reading its signature.

### Problem 2: mixed features in one package

`handlers/` holds user code *and* product code together. With ten features it becomes a junk drawer; a change to "orders" touches the same package as "users".

### Problem 3: one giant routes function

`rest/router.go` registers every route of every feature. Adding a feature means editing the central file, and merge conflicts follow.

### Problem 4: package-level side effects

`var dummyHash, _ = auth.HashPassword(...)` runs bcrypt (about 50 ms) **when the package is imported**, even in programs and tests that never log in. Package-level initialization should be cheap and side-effect free.

(What we *avoided*: reading configuration from a global inside middleware. Because Chapter 47 passes a `*Config` explicitly and `Authenticate(secret)` receives its secret as an argument, that class of tight coupling never crept in.)

---

## 3. Coupling and cohesion

Two words from software design worth knowing precisely.

**Coupling** = *how much one module depends on the internals of another.*

| | Tight coupling | Loose coupling |
|--|----------------|----------------|
| Depends on | concrete details, globals, other modules' internals | small, explicit interfaces/parameters |
| Change one part | ripples into many others | stays local |
| Testing | needs the real thing | can substitute fakes |
| Example | handler calls `database.ListProducts()` (global) | handler receives a store it was *given* |

**Cohesion** = *how well the things inside one module belong together.*

| | Low cohesion | High cohesion |
|--|--------------|---------------|
| Package contents | user code + product code + helpers | everything about products, and nothing else |
| Reason to change | many unrelated reasons | one reason (SRP, Chapter 43) |

**The goal: high cohesion, low coupling.** Things that change together live together (cohesion); modules interact through narrow, explicit connections (loose coupling).

> **Analogy: a relationship.** *Tightly coupled*: two people who can't do anything independently: one's mood and schedule dictate the other's, and if one leaves, the other collapses. *Loosely coupled*: two independent people who collaborate through clear agreements; either can change jobs or cities without breaking the other. *Coupling* is how much they depend on each other's internals. *Cohesion* is how focused each person's own life is.

---

## 4. Organizing by layer vs. by feature

**By layer** (what we have): group files by *technical role*.

```
handlers/   user.go  product.go  order.go ...
models/     user.go  product.go  order.go ...
database/   users.go products.go orders.go ...
```

To change *"products"* you edit files in three or four folders.

**By feature** (also called *vertical slices* or *domain-oriented*): group by *what the code is about*.

```
product/    handler.go  list.go  get.go  create.go  (routes + handlers + validation for products)
user/       handler.go  register.go  login.go  me.go
order/      ...
```

To change *"products"* you stay in `product/`. Related code has **high cohesion**; features touch each other only through narrow connections.

| | By layer | By feature |
|--|----------|------------|
| Finding all code for a feature | scattered | one folder |
| Adding a feature | touch many folders | add one folder |
| Merge conflicts | frequent in shared folders | rare |
| Shared code | obvious (`util/`) | must be deliberately placed |
| Risk | folders become junk drawers | duplication across features; tempting to import each other |

Go idiom leans toward **feature/domain packages**, with small shared packages (`auth`, `middleware`, `util`, `config`) for genuinely cross-cutting code. Chapters 59–61 push this further with **domain-driven design**.

---

## 5. The target structure

```
ecommerce/
├── main.go                       load config, logger → cmd.Serve
├── cmd/
│   ├── serve.go                  run the HTTP server (timeouts, graceful shutdown)
│   └── wire.go                   COMPOSITION ROOT: create stores → handlers → server
├── config/                       configuration
├── auth/                         passwords, JWT, identity in context
├── models/                       Product, User (plain data)
├── database/                     ProductStore, UserStore (in-memory; PostgreSQL later)
├── product/                      FEATURE: handler.go, list.go, get.go, create.go
├── user/                         FEATURE: handler.go, register.go, login.go, me.go
├── middleware/                   CORS, logger, recover, request ID, auth, ...
├── rest/
│   └── server.go                 assembles routes of all features + global middleware
└── util/                         JSON helpers
```

(`handlers/` and `rest/router.go` disappear.)

Dependency arrows still point one way:

```
main ──► cmd ──► rest ──► product ──┐
                  │        user ────┼──► database ──► models
                  │                 ├──► auth
                  │                 └──► util
                  └──► middleware ──► auth, util
```

---

## 6. Dependency injection, explained

> **Dependency injection (DI)** means an object **receives** the things it depends on from the outside (usually through its constructor) instead of **creating or locating** them itself.

Compare two ways for a handler to get its storage:

```go
// ❌ Handler finds its dependency itself (a global). Tightly coupled, hidden.
func GetProducts(w http.ResponseWriter, r *http.Request) {
	products := database.ListProducts() // reaches out to global state
	...
}

// ✅ Handler is GIVEN its dependency. Loosely coupled, explicit.
type Handler struct {
	store *database.ProductStore // what I need, declared in the open
}

func NewHandler(store *database.ProductStore) *Handler {
	return &Handler{store: store}
}

func (h *Handler) List(w http.ResponseWriter, r *http.Request) {
	products := h.store.List() // uses the store it was given
	...
}
```

Nothing magical: **a struct field plus a constructor.** The method still has the `(w, r)` signature `net/http` requires, and its needs come from the receiver `h`. (This is the pattern from Chapter 22: methods on a struct that carries state.)

> **Analogy: a power socket.** A lamp doesn't build its own power station (creating dependencies) or wire itself to one specific station (a global). It has a **plug** and receives power from whatever socket you connect it to: mains, a generator, a battery pack. Same lamp, different sources, and in tests you can plug into a safe dummy.

### Benefits

| Benefit | Why |
|---------|-----|
| **Testability** | Give each test its own fresh dependency (or a fake) |
| **Flexibility** | Swap the in-memory store for PostgreSQL without touching handlers |
| **Explicitness** | Constructor signatures document what each component needs |
| **No hidden state** | No global to accidentally share or mutate |
| **Concurrency-friendly** | Independent instances can't interfere (parallel tests) |

### The composition root

If every object is *given* its dependencies, **someone** must create the objects and connect them. That someone is the **composition root**: one place (near `main`) where the whole object graph is assembled:

```
   composition root (cmd/wire.go)
   ┌──────────────────────────────────────────────────────────────┐
   │ productStore := database.NewProductStore(...)                │
   │ userStore    := database.NewUserStore()                      │
   │ productHandler := product.NewHandler(productStore)           │
   │ userHandler    := user.NewHandler(userStore, secret, ttl)    │
   │ server := rest.NewServer(cfg, logger, productHandler, ...)   │
   └──────────────────────────────────────────────────────────────┘
```

Everywhere else, code only *uses* what it was handed and never calls `New...` for its own collaborators. No framework is required: in Go, DI is just **passing arguments**. (Libraries like `google/wire` or `uber-go/fx` automate the wiring in huge codebases, but plain constructors carry you very far.)

**Two rules of thumb:**

1. **Accept what you need, in the constructor.** Only what you need: `user.NewHandler(store, secret, ttl)` takes the *secret and TTL*, not the whole `Config`. Depending on less is looser coupling.
2. **Construct in the composition root only.**

---

## 7. Step 1: stores become objects

The package-level variables (`products`, `nextID`, `mu`) become **fields of a struct**. Each `NewProductStore()` call creates an independent store with its own lock.

```go
// file: database/product_store.go
package database

import (
	"ecommerce/models"
	"sync"
)

// ProductStore keeps products in memory. It is safe for concurrent use.
type ProductStore struct {
	mu       sync.RWMutex
	products []models.Product
	nextID   int
}

// NewProductStore creates a store, optionally pre-loaded with seed products.
// New products get IDs after the highest seeded ID.
func NewProductStore(seed ...models.Product) *ProductStore {
	s := &ProductStore{nextID: 1}
	for _, p := range seed {
		s.products = append(s.products, p)
		if p.ID >= s.nextID {
			s.nextID = p.ID + 1
		}
	}
	return s
}

// SampleProducts returns the demo data used when the app starts.
func SampleProducts() []models.Product {
	return []models.Product{
		{ID: 1, Title: "Orange", Description: "Orange is juicy and full of vitamin C.", Price: 100, ImageURL: "https://example.com/orange.jpg"},
		{ID: 2, Title: "Apple", Description: "A crunchy apple a day...", Price: 40, ImageURL: "https://example.com/apple.jpg"},
		{ID: 3, Title: "Banana", Description: "Great for a quick snack.", Price: 5, ImageURL: "https://example.com/banana.jpg"},
	}
}

// List returns a copy of all products.
func (s *ProductStore) List() []models.Product {
	s.mu.RLock()
	defer s.mu.RUnlock()
	out := make([]models.Product, len(s.products))
	copy(out, s.products)
	return out
}

// Get returns the product with the given ID.
func (s *ProductStore) Get(id int) (models.Product, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, p := range s.products {
		if p.ID == id {
			return p, true
		}
	}
	return models.Product{}, false
}

// Create assigns the next ID to p, stores it, and returns the stored product.
func (s *ProductStore) Create(p models.Product) models.Product {
	s.mu.Lock()
	defer s.mu.Unlock()
	p.ID = s.nextID
	s.nextID++
	s.products = append(s.products, p)
	return p
}
```

```go
// file: database/user_store.go
package database

import (
	"ecommerce/models"
	"errors"
	"strings"
	"sync"
	"time"
)

// ErrEmailTaken is returned when registering an email that already exists.
var ErrEmailTaken = errors.New("email already registered")

// UserStore keeps users in memory. It is safe for concurrent use.
type UserStore struct {
	mu     sync.RWMutex
	users  []models.User
	nextID int
}

// NewUserStore creates an empty user store.
func NewUserStore() *UserStore { return &UserStore{nextID: 1} }

func normalizeEmail(email string) string { return strings.ToLower(strings.TrimSpace(email)) }

// Create stores a new user. The duplicate check and the insert share one lock,
// so two simultaneous registrations of the same email cannot both succeed.
func (s *UserStore) Create(email, passwordHash string) (models.User, error) {
	email = normalizeEmail(email)

	s.mu.Lock()
	defer s.mu.Unlock()

	for _, u := range s.users {
		if u.Email == email {
			return models.User{}, ErrEmailTaken
		}
	}
	u := models.User{ID: s.nextID, Email: email, PasswordHash: passwordHash, CreatedAt: time.Now().UTC()}
	s.nextID++
	s.users = append(s.users, u)
	return u, nil
}

// FindByEmail looks a user up by (case-insensitive) email.
func (s *UserStore) FindByEmail(email string) (models.User, bool) {
	email = normalizeEmail(email)

	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, u := range s.users {
		if u.Email == email {
			return u, true
		}
	}
	return models.User{}, false
}

// FindByID looks a user up by ID.
func (s *UserStore) FindByID(id int) (models.User, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, u := range s.users {
		if u.ID == id {
			return u, true
		}
	}
	return models.User{}, false
}
```

Compare with the old code: the *logic* is identical, but state lives in `s`, not in package variables. `ResetProducts`/`ResetUsers` are gone; a fresh test simply creates a fresh store. There's also no `init()` that seeds data implicitly (Chapter 14's cautions): seeding is now an explicit argument.

Also delete the old files:

<!-- delete: database/products.go -->
<!-- delete: database/users.go -->
<!-- delete: handlers -->
<!-- delete: rest/router.go -->
<!-- delete: rest/router_test.go -->
<!-- delete: rest/users_test.go -->
<!-- delete: rest/me_test.go -->
<!-- delete: rest/helpers_test.go -->

```bash
rm database/products.go database/users.go rest/router.go
rm -r handlers
rm rest/router_test.go rest/users_test.go rest/me_test.go rest/helpers_test.go
```

---

## 8. Step 2: the `product` feature

One folder, everything about products: its handler struct, its routes, its handlers, its request validation.

```go
// file: product/handler.go
package product

import (
	"ecommerce/database"
	"net/http"
)

// Handler serves the product endpoints. Its dependencies are supplied by the constructor.
type Handler struct {
	store *database.ProductStore
}

// NewHandler creates a Handler that reads and writes products in store.
func NewHandler(store *database.ProductStore) *Handler {
	return &Handler{store: store}
}

// Routes registers this feature's endpoints on mux.
// authn is applied to the routes that require a logged-in user.
func (h *Handler) Routes(mux *http.ServeMux, authn func(http.Handler) http.Handler) {
	mux.HandleFunc("GET /products", h.List)
	mux.HandleFunc("GET /products/{id}", h.Get)
	mux.Handle("POST /products", authn(http.HandlerFunc(h.Create)))
}
```

Notice `authn func(http.Handler) http.Handler`: we accept the *shape* of a middleware, not the `middleware.Middleware` type. That way `product` doesn't import `middleware` at all (`middleware.Middleware` is assignable to it). Less coupling.

```go
// file: product/list.go
package product

import (
	"ecommerce/util"
	"net/http"
)

// List handles GET /products.
func (h *Handler) List(w http.ResponseWriter, r *http.Request) {
	util.SendData(w, http.StatusOK, h.store.List())
}
```

```go
// file: product/get.go
package product

import (
	"ecommerce/util"
	"net/http"
	"strconv"
)

// Get handles GET /products/{id}.
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id < 1 {
		util.SendError(w, http.StatusBadRequest, "id must be a positive integer")
		return
	}

	p, found := h.store.Get(id)
	if !found {
		util.SendError(w, http.StatusNotFound, "product not found")
		return
	}
	util.SendData(w, http.StatusOK, p)
}
```

```go
// file: product/create.go
package product

import (
	"ecommerce/models"
	"ecommerce/util"
	"fmt"
	"net/http"
	"strings"
)

// createRequest is what a client may send to create a product.
// It deliberately has no ID: the server assigns it.
type createRequest struct {
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}

func (r createRequest) validate() string {
	switch {
	case strings.TrimSpace(r.Title) == "":
		return "title is required"
	case len(r.Title) > 100:
		return "title must be at most 100 characters"
	case r.Price <= 0:
		return "price must be greater than 0"
	}
	return ""
}

// Create handles POST /products. The route requires authentication (see Routes).
func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
	var req createRequest
	if !util.DecodeJSON(w, r, &req) {
		return
	}
	if msg := req.validate(); msg != "" {
		util.SendError(w, http.StatusUnprocessableEntity, msg)
		return
	}

	p := h.store.Create(models.Product{
		Title:       strings.TrimSpace(req.Title),
		Description: req.Description,
		Price:       req.Price,
		ImageURL:    req.ImageURL,
	})

	w.Header().Set("Location", fmt.Sprintf("/products/%d", p.ID))
	util.SendData(w, http.StatusCreated, p)
}
```

**One file per handler** keeps files short and makes `git blame`/reviews focused: as a feature grows (update, delete, search, reviews) you add files, not lines to a giant one. It's a convention, not a rule; for two-line handlers a single file is fine too.

---

## 9. Step 3: the `user` feature

```go
// file: user/handler.go
package user

import (
	"ecommerce/auth"
	"ecommerce/database"
	"net/http"
	"time"
)

// Handler serves registration, login, and the current-user endpoint.
type Handler struct {
	store     *database.UserStore
	jwtSecret []byte
	jwtTTL    time.Duration

	// dummyHash lets Login spend the same time on unknown emails as on known ones.
	dummyHash string
}

// NewHandler creates a user Handler. It takes only what it needs: the store, the signing secret,
// and the token lifetime, not the whole application configuration.
func NewHandler(store *database.UserStore, jwtSecret []byte, jwtTTL time.Duration) *Handler {
	dummy, _ := auth.HashPassword("dummy-password-for-timing") // computed once per handler, not at import time
	return &Handler{store: store, jwtSecret: jwtSecret, jwtTTL: jwtTTL, dummyHash: dummy}
}

// Routes registers this feature's endpoints on mux.
func (h *Handler) Routes(mux *http.ServeMux, authn func(http.Handler) http.Handler) {
	mux.HandleFunc("POST /users", h.Register)
	mux.HandleFunc("POST /login", h.Login)
	mux.Handle("GET /me", authn(http.HandlerFunc(h.Me)))
}
```

```go
// file: user/register.go
package user

import (
	"ecommerce/auth"
	"ecommerce/database"
	"ecommerce/util"
	"errors"
	"net/http"
	"net/mail"
)

const (
	minPasswordLength = 8
	maxPasswordLength = 72 // bcrypt only uses the first 72 bytes
)

type credentials struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

func (c credentials) validateForRegistration() string {
	addr, err := mail.ParseAddress(c.Email)
	if err != nil || addr.Address != c.Email {
		return "email must be a valid address like name@example.com"
	}
	if len(c.Password) < minPasswordLength {
		return "password must be at least 8 characters"
	}
	if len(c.Password) > maxPasswordLength {
		return "password must be at most 72 bytes"
	}
	return ""
}

// Register handles POST /users.
func (h *Handler) Register(w http.ResponseWriter, r *http.Request) {
	var req credentials
	if !util.DecodeJSON(w, r, &req) {
		return
	}
	if msg := req.validateForRegistration(); msg != "" {
		util.SendError(w, http.StatusUnprocessableEntity, msg)
		return
	}

	hash, err := auth.HashPassword(req.Password)
	if err != nil {
		util.SendError(w, http.StatusInternalServerError, "could not create the account")
		return
	}

	u, err := h.store.Create(req.Email, hash)
	if errors.Is(err, database.ErrEmailTaken) {
		util.SendError(w, http.StatusConflict, "email already registered")
		return
	}
	if err != nil {
		util.SendError(w, http.StatusInternalServerError, "could not create the account")
		return
	}

	util.SendData(w, http.StatusCreated, u) // the password hash is excluded by its json:"-" tag
}
```

```go
// file: user/login.go
package user

import (
	"ecommerce/auth"
	"ecommerce/util"
	"net/http"
	"time"
)

// Login handles POST /login: it verifies credentials and returns a signed token.
func (h *Handler) Login(w http.ResponseWriter, r *http.Request) {
	var req credentials
	if !util.DecodeJSON(w, r, &req) {
		return
	}

	u, found := h.store.FindByEmail(req.Email)

	// Always run a bcrypt comparison, even for unknown emails, so timing doesn't reveal which exist.
	hash := h.dummyHash
	if found {
		hash = u.PasswordHash
	}
	passwordOK := auth.CheckPassword(hash, req.Password)

	if !found || !passwordOK {
		util.SendError(w, http.StatusUnauthorized, "invalid email or password")
		return
	}

	token, err := auth.CreateToken(h.jwtSecret, u.ID, h.jwtTTL, time.Now())
	if err != nil {
		util.SendError(w, http.StatusInternalServerError, "could not create a token")
		return
	}
	util.SendData(w, http.StatusOK, map[string]string{"token": token})
}
```

```go
// file: user/me.go
package user

import (
	"ecommerce/auth"
	"ecommerce/util"
	"net/http"
)

// Me handles GET /me. The route is wrapped in the Authenticate middleware (see Routes),
// which is what puts the caller's ID into the request context.
func (h *Handler) Me(w http.ResponseWriter, r *http.Request) {
	id, ok := auth.UserIDFrom(r.Context())
	if !ok { // only possible if the route was registered without authentication: a programming error
		util.SendError(w, http.StatusUnauthorized, "authentication required")
		return
	}

	u, found := h.store.FindByID(id)
	if !found { // a valid token for an account that no longer exists
		util.SendError(w, http.StatusUnauthorized, "account not found")
		return
	}
	util.SendData(w, http.StatusOK, u)
}
```

What changed relative to Chapter 49: the same logic, but (a) methods on `*user.Handler` using `h.store` instead of `database.FindUserByEmail(...)` globals, (b) `dummyHash` moved from a package variable into the constructor, so importing the package is free.

---

## 10. Step 4: the `rest` server

`rest` no longer knows *which* routes exist: each feature registers its own. The server only **assembles**: create the mux, ask each feature to register its routes, and wrap with the global middleware.

```go
// file: rest/server.go
package rest

import (
	"ecommerce/config"
	"ecommerce/middleware"
	"ecommerce/product"
	"ecommerce/user"
	"log/slog"
	"net/http"
)

// Server assembles the HTTP application from its already-constructed parts.
type Server struct {
	cfg      *config.Config
	logger   *slog.Logger
	products *product.Handler
	users    *user.Handler
}

// NewServer connects the feature handlers to the shared configuration and logger.
func NewServer(cfg *config.Config, logger *slog.Logger, products *product.Handler, users *user.Handler) *Server {
	return &Server{cfg: cfg, logger: logger, products: products, users: users}
}

// Handler returns the complete HTTP handler: every feature's routes wrapped in the global middleware.
func (s *Server) Handler() http.Handler {
	mux := http.NewServeMux()

	authn := middleware.Authenticate([]byte(s.cfg.JWTSecret)) // route-level middleware, given to features
	s.products.Routes(mux, authn)
	s.users.Routes(mux, authn)

	global := middleware.NewStack(
		middleware.RequestID,
		middleware.Logger(s.logger),
		middleware.CORS(s.cfg.AllowedOrigins),
		middleware.Recover(s.logger),
		middleware.JSONErrors,
	)
	return global.Then(mux)
}
```

To add an `order` feature later: create `order/`, build its handler, add one field and one `s.orders.Routes(mux, authn)` line here. Nothing else in the application changes.

---

## 11. Step 5: the composition root

`cmd/wire.go` is the only place that creates stores and handlers and connects them:

```go
// file: cmd/wire.go
package cmd

import (
	"ecommerce/config"
	"ecommerce/database"
	"ecommerce/product"
	"ecommerce/rest"
	"ecommerce/user"
	"log/slog"
	"net/http"
)

// buildHandler is the composition root: it creates every component, hands each one
// exactly the dependencies it needs, and returns the finished HTTP handler.
func buildHandler(cfg *config.Config, logger *slog.Logger) http.Handler {
	// storage
	productStore := database.NewProductStore(database.SampleProducts()...)
	userStore := database.NewUserStore()

	// features, each receiving only what it needs
	productHandler := product.NewHandler(productStore)
	userHandler := user.NewHandler(userStore, []byte(cfg.JWTSecret), cfg.JWTTTL)

	// the HTTP server that assembles them
	return rest.NewServer(cfg, logger, productHandler, userHandler).Handler()
}
```

And `Serve` now calls it (the only change is the `Handler:` line and the import list):

```go
// file: cmd/serve.go
package cmd

import (
	"context"
	"ecommerce/config"
	"errors"
	"fmt"
	"log/slog"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

const shutdownTimeout = 10 * time.Second

// Serve starts the HTTP server and blocks until it stops.
// It shuts down gracefully when the process receives SIGINT (Ctrl+C) or SIGTERM.
func Serve(cfg *config.Config, logger *slog.Logger) error {
	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           buildHandler(cfg, logger),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      30 * time.Second,
		IdleTimeout:       60 * time.Second,
		ErrorLog:          slog.NewLogLogger(logger.Handler(), slog.LevelError),
	}

	ln, err := net.Listen("tcp", srv.Addr)
	if err != nil {
		return fmt.Errorf("cannot listen on %s: %w", srv.Addr, err)
	}

	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	logger.Info("server starting", "addr", srv.Addr, "env", cfg.Env)
	return run(ctx, srv, ln, shutdownTimeout, logger)
}

// run serves on ln until ctx is cancelled, then shuts the server down gracefully.
func run(ctx context.Context, srv *http.Server, ln net.Listener, timeout time.Duration, logger *slog.Logger) error {
	serveErr := make(chan error, 1)
	go func() { serveErr <- srv.Serve(ln) }()

	select {
	case err := <-serveErr:
		if errors.Is(err, http.ErrServerClosed) {
			return nil
		}
		return err

	case <-ctx.Done():
		logger.Info("shutting down", "timeout", timeout)

		shutdownCtx, cancel := context.WithTimeout(context.Background(), timeout)
		defer cancel()

		if err := srv.Shutdown(shutdownCtx); err != nil {
			return fmt.Errorf("graceful shutdown failed: %w", err)
		}
		<-serveErr
		logger.Info("server stopped cleanly")
		return nil
	}
}
```

The whole startup story is now explicit and readable top to bottom: `main` → `config.Load` → `cmd.Serve` → `buildHandler` (**the object graph**) → `http.Server`.

---

## 12. Tests: isolation for free

### An isolated app per test

Because nothing is global, each test can build its **own** stores and server. No `Reset...`, no ordering dependence, and tests can run **in parallel** (`t.Parallel()`), which also makes the race detector far more effective.

```go
// file: rest/app_test.go
package rest

import (
	"ecommerce/auth"
	"ecommerce/config"
	"ecommerce/database"
	"ecommerce/product"
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

// testApp is a complete, private instance of the application.
type testApp struct {
	t        *testing.T
	cfg      *config.Config
	handler  http.Handler
	products *database.ProductStore
	users    *database.UserStore
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
	products := database.NewProductStore(database.SampleProducts()...)
	users := database.NewUserStore()

	server := NewServer(cfg, quietLogger,
		product.NewHandler(products),
		user.NewHandler(users, []byte(cfg.JWTSecret), cfg.JWTTTL),
	)
	return &testApp{t: t, cfg: cfg, handler: server.Handler(), products: products, users: users}
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

func (a *testApp) count() int {
	var list []struct{ ID int }
	json.NewDecoder(a.do(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	return len(list)
}
```

And the tests, which use only that helper (nothing global, nothing to reset):

```go
// file: rest/server_test.go
package rest

import (
	"encoding/json"
	"net/http"
	"strings"
	"sync"
	"testing"
)

func TestPublicProductEndpoints(t *testing.T) {
	t.Parallel()
	a := newApp(t)

	if got := a.count(); got != 3 {
		t.Fatalf("expected 3 products, got %d", got)
	}
	for path, want := range map[string]int{"/products/2": 200, "/products/99": 404, "/products/abc": 400} {
		if got := a.do(http.MethodGet, path, "", nil).Code; got != want {
			t.Errorf("GET %s: expected %d, got %d", path, want, got)
		}
	}
}

func TestCreatingProductsNeedsAToken(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	body := `{"title":"Mango","description":"d","price":200,"imageUrl":"u"}`

	if rec := a.do(http.MethodPost, "/products", body, nil); rec.Code != http.StatusUnauthorized {
		t.Fatalf("expected 401 without a token, got %d", rec.Code)
	}
	if a.count() != 3 {
		t.Fatal("an unauthenticated request created a product")
	}

	rec := a.do(http.MethodPost, "/products", body, a.bearer(1))
	if rec.Code != http.StatusCreated || rec.Header().Get("Location") != "/products/4" {
		t.Fatalf("expected 201 and Location /products/4, got %d %q", rec.Code, rec.Header().Get("Location"))
	}
	if a.count() != 4 {
		t.Errorf("expected 4 products after creating one")
	}
}

func TestEachTestHasItsOwnData(t *testing.T) {
	t.Parallel()
	a, b := newApp(t), newApp(t)

	a.do(http.MethodPost, "/products", `{"title":"Only in A","price":1}`, a.bearer(1))

	if a.count() != 4 || b.count() != 3 {
		t.Errorf("instances share state: a=%d b=%d", a.count(), b.count())
	}
}

func TestRegisterLoginMeFlow(t *testing.T) {
	t.Parallel()
	a := newApp(t)

	rec := a.do(http.MethodPost, "/users", `{"email":"Asha@Example.com","password":"correct-horse-battery"}`, nil)
	if rec.Code != http.StatusCreated {
		t.Fatalf("register: %d %s", rec.Code, rec.Body.String())
	}
	if strings.Contains(rec.Body.String(), "$2") || strings.Contains(strings.ToLower(rec.Body.String()), "password") {
		t.Errorf("registration leaked password material: %s", rec.Body.String())
	}
	if rec := a.do(http.MethodPost, "/users", `{"email":"ASHA@example.com","password":"another-good-password"}`, nil); rec.Code != http.StatusConflict {
		t.Errorf("duplicate email: expected 409, got %d", rec.Code)
	}

	rec = a.do(http.MethodPost, "/login", `{"email":"asha@example.com","password":"correct-horse-battery"}`, nil)
	var resp struct {
		Token string `json:"token"`
	}
	json.NewDecoder(rec.Body).Decode(&resp)
	if rec.Code != http.StatusOK || resp.Token == "" {
		t.Fatalf("login: %d %s", rec.Code, rec.Body.String())
	}

	me := a.do(http.MethodGet, "/me", "", map[string]string{"Authorization": "Bearer " + resp.Token})
	if me.Code != http.StatusOK || !strings.Contains(me.Body.String(), `"email":"asha@example.com"`) {
		t.Errorf("GET /me: %d %s", me.Code, me.Body.String())
	}
	if rec := a.do(http.MethodGet, "/me", "", nil); rec.Code != http.StatusUnauthorized {
		t.Errorf("GET /me without a token: expected 401, got %d", rec.Code)
	}
}

func TestLoginFailuresAreIndistinguishable(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	a.do(http.MethodPost, "/users", `{"email":"asha@example.com","password":"correct-horse-battery"}`, nil)

	wrongPassword := a.do(http.MethodPost, "/login", `{"email":"asha@example.com","password":"wrong-password-here"}`, nil)
	unknownEmail := a.do(http.MethodPost, "/login", `{"email":"ghost@example.com","password":"correct-horse-battery"}`, nil)

	if wrongPassword.Code != http.StatusUnauthorized || unknownEmail.Code != http.StatusUnauthorized {
		t.Fatalf("expected two 401s, got %d and %d", wrongPassword.Code, unknownEmail.Code)
	}
	if wrongPassword.Body.String() != unknownEmail.Body.String() {
		t.Error("the two failures must be indistinguishable to clients")
	}
}

func TestPipelineStillWraps(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	origin := map[string]string{"Origin": "http://localhost:5173"}

	// preflight for a protected route is answered without credentials
	rec := a.do(http.MethodOptions, "/products", "", map[string]string{
		"Origin":                         "http://localhost:5173",
		"Access-Control-Request-Method":  "POST",
		"Access-Control-Request-Headers": "content-type,authorization",
	})
	if rec.Code != http.StatusNoContent || rec.Header().Get("Access-Control-Allow-Origin") != "http://localhost:5173" {
		t.Errorf("preflight: %d %v", rec.Code, rec.Header())
	}

	// mux-generated errors are JSON, carry CORS headers and a request ID
	rec = a.do(http.MethodGet, "/no-such-route", "", origin)
	if rec.Code != 404 || !strings.Contains(rec.Body.String(), `"route not found"`) ||
		rec.Header().Get("Access-Control-Allow-Origin") == "" || len(rec.Header().Get("X-Request-ID")) != 16 {
		t.Errorf("404: %d %q %v", rec.Code, rec.Body.String(), rec.Header())
	}
	if rec := a.do(http.MethodDelete, "/products", "", nil); rec.Code != 405 || rec.Header().Get("Allow") != "GET, HEAD, POST" {
		t.Errorf("405: %d allow=%q", rec.Code, rec.Header().Get("Allow"))
	}
}

func TestConcurrentCreates(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	headers := a.bearer(1)

	const n = 50
	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			a.do(http.MethodPost, "/products", `{"title":"Concurrent","price":1}`, headers)
		}()
	}
	wg.Wait()

	if got := a.count(); got != 3+n {
		t.Fatalf("expected %d products, got %d", 3+n, got)
	}
}
```

### Testing a feature in isolation

DI also means we can test a **handler alone**: no router, no middleware, no JWT: just the handler and a store we control.

```go
// file: product/handler_test.go
package product

import (
	"ecommerce/database"
	"ecommerce/models"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestCreateValidationLeavesTheStoreUntouched(t *testing.T) {
	tests := []struct {
		name string
		body string
		want int
	}{
		{"empty body", ``, http.StatusBadRequest},
		{"truncated json", `{"title": `, http.StatusBadRequest},
		{"wrong type", `{"title":"x","price":"cheap"}`, http.StatusBadRequest},
		{"unknown field", `{"id":9,"title":"x","price":1}`, http.StatusBadRequest},
		{"two objects", `{"title":"x","price":1}{"title":"y","price":2}`, http.StatusBadRequest},
		{"missing title", `{"price":5}`, http.StatusUnprocessableEntity},
		{"blank title", `{"title":"   ","price":5}`, http.StatusUnprocessableEntity},
		{"negative price", `{"title":"x","price":-1}`, http.StatusUnprocessableEntity},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			t.Parallel()
			store := database.NewProductStore() // a private, empty store: the whole "database" for this test
			h := NewHandler(store)

			rec := httptest.NewRecorder()
			h.Create(rec, httptest.NewRequest(http.MethodPost, "/products", strings.NewReader(tc.body)))

			if rec.Code != tc.want {
				t.Errorf("expected %d, got %d (%s)", tc.want, rec.Code, rec.Body.String())
			}
			if n := len(store.List()); n != 0 {
				t.Errorf("an invalid request stored %d products", n)
			}
		})
	}
}

func TestCreateStoresTheProductAndAssignsAnID(t *testing.T) {
	store := database.NewProductStore(models.Product{ID: 41, Title: "Existing", Price: 1})
	h := NewHandler(store)

	rec := httptest.NewRecorder()
	h.Create(rec, httptest.NewRequest(http.MethodPost, "/products",
		strings.NewReader(`{"title":"  New thing  ","description":"d","price":9.5}`)))

	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d (%s)", rec.Code, rec.Body.String())
	}
	if loc := rec.Header().Get("Location"); loc != "/products/42" {
		t.Errorf("IDs continue after the highest seeded ID: Location = %q", loc)
	}
	got, ok := store.Get(42)
	if !ok || got.Title != "New thing" { // the title was trimmed
		t.Errorf("stored product = %+v (found=%v)", got, ok)
	}
}
```

### Testing a store on its own

```go
// file: database/stores_test.go
package database

import (
	"errors"
	"sync"
	"testing"
)

func TestUserStoreRejectsDuplicatesUnderConcurrency(t *testing.T) {
	s := NewUserStore()

	const attempts = 50
	var wg sync.WaitGroup
	var mu sync.Mutex
	successes := 0

	for i := 0; i < attempts; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			// 50 goroutines register "the same" email, with sloppy spacing and capitals
			_, err := s.Create("  ASHA@Example.com ", "hash")
			if err == nil {
				mu.Lock()
				successes++
				mu.Unlock()
			} else if !errors.Is(err, ErrEmailTaken) {
				t.Errorf("unexpected error: %v", err)
			}
		}()
	}
	wg.Wait()

	if successes != 1 {
		t.Fatalf("exactly one registration must win, got %d", successes)
	}
}

func TestProductStoreIDsAreUniqueAndSequential(t *testing.T) {
	s := NewProductStore()
	const n = 100

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			s.Create(SampleProducts()[0])
		}()
	}
	wg.Wait()

	list := s.List()
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

func TestListReturnsACopy(t *testing.T) {
	s := NewProductStore(SampleProducts()...)
	list := s.List()
	list[0].Title = "tampered"

	if got, _ := s.Get(1); got.Title == "tampered" {
		t.Error("callers must not be able to modify the store's internal slice")
	}
}
```

Run everything (it's now faster, thanks to parallel tests):

```bash
go vet ./... && go test -race ./...
```

```
?   	ecommerce	[no test files]
ok  	ecommerce/auth	5.0s
ok  	ecommerce/cmd	1.6s
ok  	ecommerce/config	1.0s
ok  	ecommerce/database	1.0s
ok  	ecommerce/middleware	1.0s
ok  	ecommerce/product	1.0s
ok  	ecommerce/rest	2.5s
...
```

---

## 13. Before vs. after

| | Chapter 49 | Chapter 50 |
|--|-----------|------------|
| Storage | package globals + `Reset...()` helpers | `ProductStore`/`UserStore` objects, one per app/test |
| Handlers | `handlers` package with plain functions and a mixed bag | `product/` and `user/` feature packages, methods on structs holding their dependencies |
| Routes | One central function knowing every route | Each feature has `Routes(mux, authn)`; `rest` just assembles |
| Wiring | Implicit (globals + package init) | Explicit, in `cmd/wire.go` (the composition root) |
| Import-time side effects | `dummyHash` (bcrypt) computed on import | None |
| Tests | Shared state; must reset; can't run in parallel | Private instance per test; `t.Parallel()`; unit tests for a single handler |
| Adding a feature | Edit `handlers/`, `rest/router.go`, `database/` | Add a folder; add ~2 lines to `rest/server.go` and `cmd/wire.go` |
| Replacing storage | Edit every handler | Change what the composition root builds (next chapter: hide it behind an interface) |

The *behavior* is unchanged, and the tests prove it: the same scenarios pass against the new structure.

---

## 14. What is still coupled

Honest self-review again:

1. **Features depend on a concrete type**: `product.Handler` holds a `*database.ProductStore`. To use PostgreSQL we'd still have to change the type in `product`. What the handler *needs* ("something that can list, get, and create products") is smaller than what it *is given* (a specific struct). → **Chapter 51**: define an **interface** and depend on that.
2. **`user`/`product` import `database`**, and `database` imports `models`. The business rules (validation, password policy) live in HTTP handlers, mixed with request/response code. → **Chapters 59–61** (domain-driven design: services and repositories).
3. **Shared `models` package**: every feature imports the same `Product`/`User` structs, which can grow into a dumping ground.
4. **The stores are still in memory**: data disappears on restart. → **Chapters 52–58** (PostgreSQL).

Progress in software design is a *sequence* of small improvements, each removing the biggest current pain.

---

## 15. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Constructing dependencies inside handlers/services (`store := database.NewProductStore()` in `Create`) | Tight coupling; each request gets a fresh store! | Construct once at the composition root; inject |
| 2 | Passing the whole `Config` everywhere | Everything depends on everything | Pass only the needed values (`secret`, `ttl`) |
| 3 | Using a service locator / global registry (`app.Get("db")`) | Hidden dependencies again | Explicit constructor parameters |
| 4 | Feature packages importing each other (`product` ↔ `user`) | Import cycles, tangled features | Depend on narrow interfaces, or lift shared logic to a third package |
| 5 | A single `Handler` struct with 30 dependencies | "God object" | Split by feature; each handler takes only its own dependencies |
| 6 | Doing real work (network, bcrypt, file I/O) in `init()` or package-level `var` | Slow imports, surprising failures | Do it in constructors called from `main`/the composition root |
| 7 | Forgetting the pointer receiver on a struct with a `sync.Mutex` | Copying a mutex (`go vet` warns: "passes lock by value") | Methods on `*Store`; store by pointer |
| 8 | Returning the store's internal slice from `List()` | Callers mutate internals; data races | Return a copy |
| 9 | Over-engineering DI for a 100-line program | More ceremony than value | Introduce DI when tests or growth demand it |
| 10 | Creating the object graph in several places | Inconsistent configurations | One composition root |
| 11 | Moving everything to `internal/` or deep folders "for architecture's sake" | Confusion for little benefit | Add structure when you feel pain |
| 12 | Tests that still depend on package-level state | Flaky parallel tests | Give each test its own instances |

---

## 16. Exercises

### Exercise 1: Classify the coupling
Rank from *tightest* to *loosest*: (a) a function that calls `os.Getenv("DB_URL")` internally, (b) a function receiving a `*sql.DB` parameter, (c) a function receiving a `Store` interface parameter, (d) a function that reads a package-level `var db *sql.DB`.

<details><summary>Solution</summary>

Tightest → loosest: (a) and (d) (hidden global/environment dependencies; (a) is arguably tighter since it also bakes in the environment), then (b) (explicit, but a concrete type), then (c) (explicit *and* abstract).
</details>

### Exercise 2: Add an `order` feature skeleton
Sketch (folders/files/types) what you'd add for `GET /orders` and `POST /orders`, and which *existing* files need a one-line change.

<details><summary>Solution</summary>

New: `models/order.go`, `database/order_store.go` (`OrderStore`), `order/handler.go` (`Handler{store}`, `NewHandler`, `Routes`), `order/list.go`, `order/create.go`. Changes: `rest/server.go` (field + `NewServer` parameter + `s.orders.Routes(mux, authn)`), `cmd/wire.go` (`orderStore := ...; orderHandler := ...`). Everything else is untouched.
</details>

### Exercise 3: Find the hidden dependency
What does this function secretly depend on? How would you make the dependency explicit?

```go
func SendWelcomeEmail(to string) error {
	return smtp.SendMail(os.Getenv("SMTP_HOST"), auth, "noreply@shop.com", []string{to}, msg)
}
```

<details><summary>Solution</summary>

It depends on the environment (`SMTP_HOST`) and the network/SMTP server (plus an implicit `auth` value). Make it explicit with a struct: `type Mailer struct { host string; auth smtp.Auth; from string }`, `NewMailer(host, auth, from)`, and `func (m *Mailer) SendWelcome(to string) error`. Then tests can build a `Mailer` pointing at a fake SMTP server (or, after Chapter 51, depend on an interface).
</details>

### Exercise 4: Parallel proof
Add `t.Parallel()` to a test in the *old* (global) design and run with `-race`. What happens, and why doesn't the new design have the problem?

<details><summary>Solution</summary>

With shared globals plus `ResetProducts()`, parallel tests reset and mutate the same slice, causing flaky assertion failures and (when a lock is missing) `DATA RACE` reports. In the new design each test owns its stores, so there's nothing shared to race on.
</details>

### Exercise 5: Constructor validation
`user.NewHandler` accepts a nil store or an empty secret and fails later at request time. Make it fail fast. What are the trade-offs of returning an error vs panicking?

<details><summary>Solution</summary>

`func NewHandler(store *database.UserStore, secret []byte, ttl time.Duration) (*Handler, error)` returning an error if `store == nil`, `len(secret) == 0`, or `ttl <= 0`; the composition root reports it and exits. A `panic` is acceptable for *programmer errors* (a nil store can never be valid), but returning an error is friendlier when values come from configuration.
</details>

### Exercise 6: Second route, same feature
Add `GET /products/count` returning `{"count": N}`. Which files change? Does the pattern conflict with `GET /products/{id}`?

<details><summary>Solution</summary>

Only `product/handler.go` (register the route) and a new `product/count.go`. No conflict: `GET /products/count` is *more specific* than `GET /products/{id}` (Chapter 44), so `/products/count` reaches the new handler and `/products/7` still matches the wildcard.
</details>

### Exercise 7 (challenge): Swap the storage
Without changing `product/`, how could you make the app use a *different* product store implementation? What prevents you today, and what's the minimal change that would allow it?

<details><summary>Solution</summary>

`product.Handler` requires the concrete `*database.ProductStore`, so any other implementation is a different type and won't compile. The minimal change: define in `product` a small interface (`type Store interface { List() []models.Product; Get(id int) (models.Product, bool); Create(models.Product) models.Product }`) and have `Handler` depend on it; `*database.ProductStore` already satisfies it implicitly. That's Chapter 51.
</details>

---

## 17. Quiz

1. Define tight coupling and low cohesion.
2. Why do package-level variables hurt testing?
3. What is dependency injection, in Go terms?
4. What is a composition root, and where does it live in our project?
5. What's the advantage of organizing by feature?
6. Why does `user.NewHandler` take `jwtSecret` and `jwtTTL` instead of `*config.Config`?
7. Why is computing `dummyHash` in the constructor better than in a package-level `var`?

<details><summary>Answers</summary>

1. Tight coupling: a module depends on another's internals/globals so changes ripple; low cohesion: a module's contents don't belong together (many reasons to change).
2. Tests share and mutate the same state, so they need resets, can't run in parallel, and interfere.
3. Passing an object's dependencies in through its constructor/fields instead of creating or locating them inside.
4. The single place where the object graph is created and connected: `cmd/wire.go` (`buildHandler`).
5. Related code is together (high cohesion); features can be added/changed with little impact on others.
6. To depend on only what it needs (looser coupling, easier testing).
7. Importing the package no longer triggers a costly bcrypt computation; work happens only when a handler is actually built.
</details>

---

## 18. Summary

- **Coupling** (dependence on other modules' internals) should be **low**; **cohesion** (things belong together) should be **high**. Package-level globals and hidden dependencies create tight coupling.
- Organize by **feature** (`product/`, `user/`) so related code lives together; keep small shared packages (`auth`, `middleware`, `util`, `config`) for cross-cutting concerns. Each feature owns its handler struct and its **`Routes`**.
- **Dependency injection = constructors and struct fields**: `NewHandler(store)`. Accept only what you need. No framework required.
- Turn global state into **store objects**; **construct everything once** in the **composition root** (`cmd/wire.go`) and pass it down.
- Payoffs: **isolated, parallel tests**, feature-level unit tests, no import-time side effects, and a structure ready for swapping storage.
- Remaining coupling (handlers depend on a concrete store type; business rules live in handlers) is addressed next with **interfaces** and later with domain-driven design.

### ➡️ What's next?

[Chapter 51](51-interfaces-and-design-patterns.md) introduces **interfaces**, Go's mechanism for abstraction: define what a handler *needs* as a small interface, and any type that provides it (an in-memory store, PostgreSQL, a test fake) can be plugged in.
