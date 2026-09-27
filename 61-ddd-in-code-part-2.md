# Chapter 61: DDD in Code, Part 2 — The Product Domain (and Retiring `models`)

> **Goal of this chapter:** Repeat Chapter 60's recipe for the **catalog**: a `product` domain package with its own entity, validation rules, domain errors, a `Repository` port and a `Service` (`List`, `Get`, `Create`, `Update`, `Delete`), thin HTTP and PostgreSQL adapters, and fast unit tests, and then **delete the shared `models` package** for good. Along the way we fix a subtle bug the old code had (title length counted in **bytes**, not characters), add an **architecture test** that keeps the domains independent of each other, and discuss how to keep files a sensible size, what "maintainable" means, and how this structure prepares us for serious testing.

**Difficulty:** 🔴 Advanced  **Estimated time:** 6 hours  **Prerequisite:** [Chapter 60](60-ddd-in-code-part-1.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The plan, and what is different from users](#2-the-plan-and-what-is-different-from-users)
3. [The product domain](#3-the-product-domain)
4. [The service](#4-the-service)
5. [Adapters](#5-adapters)
6. [The HTTP handler](#6-the-http-handler)
7. [Wiring and retiring `models`](#7-wiring-and-retiring-models)
8. [Tests](#8-tests)
9. [Keeping domains independent](#9-keeping-domains-independent)
10. [Proving nothing changed](#10-proving-nothing-changed)
11. [Change-impact analysis](#11-change-impact-analysis)
12. [How big should a file be?](#12-how-big-should-a-file-be)
13. [What makes this structure maintainable?](#13-what-makes-this-structure-maintainable)
14. [Preparing for real unit testing](#14-preparing-for-real-unit-testing)
15. [Common mistakes](#15-common-mistakes)
16. [Interview questions](#16-interview-questions)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. What you will learn

- Applying a refactoring recipe a second time, faster, and spotting what's *different*
- A **`Details`** input type: "what a client may specify" vs. the `Product` entity
- Fixing **bytes vs. characters** in a business rule (and why one central rule makes the fix trivial)
- Replacing four small handler files with one adapter package and one error-translation function
- **Architecture tests** for both directions: no technology in domains, no domain importing another domain
- Retiring a shared package (`models`) once each domain owns its types
- When and how to **split files**, and what actually makes a codebase maintainable
- Why this design makes **unit tests cheap** (and a concrete before/after comparison)

---

## 2. The plan, and what is different from users

The seven steps from Chapter 60 apply unchanged:

```
1 entity + errors  →  2 value objects  →  3 ports  →  4 service + unit tests
      →  5 adapters (repository)  →  6 thin handler  →  7 wire, delete old code, keep e2e tests green
```

What differs from the user domain:

| Aspect | Users (Chapter 60) | Products (this chapter) |
|--------|-------------------|-------------------------|
| Collaborators | repository **+ hasher + token issuer** | repository only |
| Value objects | `Email` (worth having: used in two ports) | none needed yet: title and price are simple fields with rules |
| Use cases | register, login, get | list, get, create, **update, delete** |
| Input shape | `email, password` | a **`Details`** struct (title, description, price, image) shared by create and update |
| Security-sensitive behavior | timing equalization | none: mostly validation and CRUD |

That last row is honest: the catalog is closer to *plain CRUD* than users are. We still do the full structure because (a) it's the same shape as everything that will grow around it (orders, inventory, pricing rules), and (b) consistency across domains makes the codebase learnable. If the whole app were only this, the "DDD-lite" would arguably be overkill (Chapter 59, section 12).

The current state to be replaced:

```
product/            handler.go create.go get.go list.go update.go delete.go request.go store.go
                    + handler_test.go write_test.go   (HTTP handlers, request type, Store interface)
models/             product.go (json + db tags) · errors.go (ErrNotFound)
```

---

## 3. The product domain

```go
// file: product/product.go
// Package product is the Catalog domain: what we sell, and the rules for describing it.
// It contains business rules only. It must not import net/http, database/sql or any adapter.
package product

import (
	"math"
	"strings"
	"unicode/utf8"
)

// Product is an item in the catalog (an entity: identified by ID).
type Product struct {
	ID          int
	Title       string
	Description string
	Price       float64 // a teaching simplification: real money should be integer minor units or a decimal type
	ImageURL    string
}

// Details is what a client may specify when creating or replacing a product: everything except
// the ID, which the system assigns. Create and Update accept the same Details, so they share one set of rules.
type Details struct {
	Title       string
	Description string
	Price       float64
	ImageURL    string
}

const maxTitleLength = 100 // characters, not bytes

// normalized returns d with its text cleaned up, or a *ValidationError if a rule is broken.
// This is the ONE place the catalog's input rules live.
func (d Details) normalized() (Details, error) {
	d.Title = strings.TrimSpace(d.Title)

	switch {
	case d.Title == "":
		return Details{}, invalid("title is required")
	case utf8.RuneCountInString(d.Title) > maxTitleLength:
		return Details{}, invalid("title must be at most 100 characters")
	case math.IsNaN(d.Price) || math.IsInf(d.Price, 0) || d.Price <= 0:
		return Details{}, invalid("price must be greater than 0")
	}
	return d, nil
}

// toProduct builds the entity for the given ID.
func (d Details) toProduct(id int) Product {
	return Product{ID: id, Title: d.Title, Description: d.Description, Price: d.Price, ImageURL: d.ImageURL}
}
```

Two things changed from the HTTP-layer rule in Chapter 51:

1. **Characters, not bytes.** The old code used `len(r.Title) > 100`, which counts *bytes*. A 60-character title in Bengali or Japanese is 180 bytes and would have been rejected by Go, while the database's `char_length(...) <= 100` (Chapter 54) would have accepted it. Now Go and SQL agree, and it took **one line in one file** to fix, exactly the benefit of central rules.
2. **`NaN` and `±Inf`** are rejected for price. JSON can't contain them, but *other callers* (a CLI, an import job, a future gRPC API) might pass them; `NaN <= 0` is `false`, so without the explicit check `NaN` would slip through the `> 0` rule. The domain protects itself against every caller, not just today's HTTP handler.

The errors mirror the user domain's:

```go
// file: product/errors.go
package product

import "errors"

// ErrNotFound means no product has the requested ID.
var ErrNotFound = errors.New("product not found")

// ValidationError means the caller's input broke a business rule. Message is written for end users.
type ValidationError struct{ Message string }

func (e *ValidationError) Error() string { return e.Message }

func invalid(message string) error { return &ValidationError{Message: message} }
```

And the single port:

```go
// file: product/port.go
package product

import "context"

// Repository stores products. It is a port: implemented by adapters such as postgres.ProductStore.
type Repository interface {
	// List returns every product ordered by ID.
	List(ctx context.Context) ([]Product, error)
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

---

## 4. The service

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

// List returns every product.
func (s *Service) List(ctx context.Context) ([]Product, error) {
	products, err := s.repo.List(ctx)
	if err != nil {
		return nil, fmt.Errorf("list products: %w", err)
	}
	return products, nil
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

The whole service is ~50 lines. Notice `wrap`: one tiny helper replaces the repeated `if errors.Is(err, ErrNotFound) … else wrap` blocks we wrote by hand in the user service (that duplication is a hint that Chapter 60's service could use the same helper: Exercise 1).

---

## 5. Adapters

### PostgreSQL

The queries and `products` table stay exactly as they were. The adapter gets its own **row type** (with `db` tags), so the domain `Product` has no tags:

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

// List returns every product ordered by ID. The result is never nil.
func (s *ProductStore) List(ctx context.Context) ([]product.Product, error) {
	rows := make([]productRow, 0)
	if err := s.db.SelectContext(ctx, &rows, listProductsSQL); err != nil {
		return nil, fmt.Errorf("list products: %w", err)
	}
	products := make([]product.Product, 0, len(rows))
	for _, r := range rows {
		products = append(products, r.toProduct())
	}
	return products, nil
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

### In memory

The same file as in Chapter 56 with the types changed to `product.Product` and `product.ErrNotFound`:

```go
// file: database/product_store.go
package database

import (
	"context"
	"ecommerce/product"
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

// List returns a copy of all products, ordered by ID.
func (s *ProductStore) List(_ context.Context) ([]product.Product, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	out := make([]product.Product, len(s.products))
	copy(out, s.products)
	return out, nil
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

---

## 6. The HTTP handler

One package, one file, the same shape as `userhandler`: decode, call the service, translate errors in **one** function. The JSON shapes are their own types (with the tags), separate from the entity.

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
	List(ctx context.Context) ([]product.Product, error)
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

// List handles GET /products.
func (h *Handler) List(w http.ResponseWriter, r *http.Request) {
	products, err := h.svc.List(r.Context())
	if err != nil {
		h.fail(w, r, err)
		return
	}
	out := make([]productResponse, 0, len(products)) // never nil: an empty catalog must encode as [] not null
	for _, p := range products {
		out = append(out, toResponse(p))
	}
	util.SendData(w, http.StatusOK, out)
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

Compare: before, seven small files each repeated "decode → validate → call store → `switch { ErrNotFound … }`". Now every error path funnels through `fail`, so adding a rule such as "conflict on duplicate title" means one new domain error and one new `case`.

`rest/server.go` follows:

```go
// file: rest/server.go
package rest

import (
	"ecommerce/config"
	"ecommerce/health"
	"ecommerce/middleware"
	"ecommerce/rest/producthandler"
	"ecommerce/rest/userhandler"
	"log/slog"
	"net/http"
)

// Server assembles the HTTP application from its already-constructed parts.
type Server struct {
	cfg      *config.Config
	logger   *slog.Logger
	products *producthandler.Handler
	users    *userhandler.Handler
	health   *health.Handler
}

// NewServer connects the feature handlers to the shared configuration and logger.
func NewServer(cfg *config.Config, logger *slog.Logger, products *producthandler.Handler, users *userhandler.Handler, health *health.Handler) *Server {
	return &Server{cfg: cfg, logger: logger, products: products, users: users, health: health}
}

// Handler returns the complete HTTP handler: every feature's routes wrapped in the global middleware.
func (s *Server) Handler() http.Handler {
	mux := http.NewServeMux()

	authn := middleware.Authenticate([]byte(s.cfg.JWTSecret)) // route-level middleware, given to features
	s.products.Routes(mux, authn)
	s.users.Routes(mux, authn)
	s.health.Routes(mux)

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

---

## 7. Wiring and retiring `models`

```go
// file: cmd/wire.go
package cmd

import (
	"ecommerce/auth"
	"ecommerce/config"
	"ecommerce/health"
	"ecommerce/postgres"
	"ecommerce/product"
	"ecommerce/rest"
	"ecommerce/rest/producthandler"
	"ecommerce/rest/userhandler"
	"ecommerce/user"
	"fmt"
	"log/slog"
	"net/http"

	"github.com/jmoiron/sqlx"
)

// buildHandler is the composition root: it creates every component, hands each one
// exactly the dependencies it needs, and returns the finished HTTP handler.
func buildHandler(cfg *config.Config, logger *slog.Logger, conn *sqlx.DB) (http.Handler, error) {
	// adapters: concrete technology, chosen here and nowhere else
	userRepo := postgres.NewUserStore(conn)
	productRepo := postgres.NewProductStore(conn)
	hasher := auth.BcryptHasher{}
	tokens := auth.TokenIssuer{Secret: []byte(cfg.JWTSecret), TTL: cfg.JWTTTL}

	// domain services: business rules, built from ports
	userService, err := user.NewService(userRepo, hasher, tokens)
	if err != nil {
		return nil, fmt.Errorf("build user service: %w", err)
	}
	productService := product.NewService(productRepo)

	// delivery adapters
	userHandler := userhandler.New(userService, logger)
	productHandler := producthandler.New(productService, logger)
	healthHandler := health.NewHandler(conn, logger)

	return rest.NewServer(cfg, logger, productHandler, userHandler, healthHandler).Handler(), nil
}
```

Every product-owned piece now lives in the product domain, so the shared `models` package has nothing left to hold, and the old product handler files go too:

<!-- delete: models -->
<!-- delete: product/handler.go -->
<!-- delete: product/create.go -->
<!-- delete: product/get.go -->
<!-- delete: product/list.go -->
<!-- delete: product/update.go -->
<!-- delete: product/delete.go -->
<!-- delete: product/request.go -->
<!-- delete: product/store.go -->
<!-- delete: product/handler_test.go -->
<!-- delete: product/write_test.go -->

A good moment to appreciate what a "shared `models` package" costs: while it existed, `user` and `product` both *had* to import it, so it was a hidden coupling point between two supposedly independent features. Deleting it is the cleanest possible proof of independence: `grep -r ecommerce/models .` returns nothing.

---

## 8. Tests

### Domain: rules and use cases with a fake repository

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
	err    error // if set, every call fails with it
	calls  int   // how many times the repository was touched
}

func (f *fakeRepo) touch() error { f.calls++; return f.err }

func (f *fakeRepo) List(context.Context) ([]Product, error) {
	return append([]Product(nil), f.items...), f.touch()
}

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
	list, _ := svc.List(ctx)
	if len(list) != 1 || list[0].ID != b.ID {
		t.Errorf("got %+v", list)
	}
}

func TestStorageFailuresAreWrappedNotDisguised(t *testing.T) {
	boom := errors.New("connection reset")
	svc, repo := newTestService()
	repo.err = boom

	calls := map[string]func() error{
		"list":   func() error { _, err := svc.List(ctx); return err },
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

Study `TestTheSameRulesApplyToCreateAndUpdate`: 14 cases (7 rules × 2 operations) from a **table** and two loops, and it verifies not just the message but that **the repository was never touched**. And `TestTitleLengthCountsCharactersNotBytes` is the regression test for the bug we just fixed.

### HTTP adapter: translation only

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
	list   func() ([]product.Product, error)
	get    func(id int) (product.Product, error)
	create func(d product.Details) (product.Product, error)
	update func(id int, d product.Details) (product.Product, error)
	del    func(id int) error
	called bool
}

func (f *fakeService) List(context.Context) ([]product.Product, error) {
	f.called = true
	return f.list()
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

func TestEmptyCatalogIsAnEmptyJSONArray(t *testing.T) {
	h, _ := newHandler(&fakeService{list: func() ([]product.Product, error) { return nil, nil }}) // even a nil slice
	rec := call(h.List, http.MethodGet, "", "")
	if rec.Code != 200 || strings.TrimSpace(rec.Body.String()) != "[]" {
		t.Errorf("got %d %q, want 200 []", rec.Code, rec.Body.String())
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

### Repository contract and adapters

The contract suite, now speaking `product` types:

```go
// file: product/storetest/storetest.go
// Package storetest holds a behavior suite that every product.Repository implementation must pass.
package storetest

import (
	"context"
	"ecommerce/product"
	"errors"
	"sync"
	"testing"
)

// Factory returns a fresh, empty store for one test.
type Factory func(t *testing.T) product.Repository

func sample(title string, price float64) product.Product {
	return product.Product{Title: title, Description: "d", Price: price, ImageURL: "https://example.com/x.jpg"}
}

// Run executes the contract against stores made by newStore.
// Prices use at most two decimals, because that is what every implementation can represent exactly.
func Run(t *testing.T, newStore Factory) {
	ctx := context.Background()

	t.Run("an empty store lists an empty, non-nil slice", func(t *testing.T) {
		list, err := newStore(t).List(ctx)
		if err != nil {
			t.Fatal(err)
		}
		if list == nil || len(list) != 0 {
			t.Errorf("want an empty non-nil slice (JSON []), got %#v", list)
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
		list, _ := s.List(ctx)
		if len(list) != 3 || list[0].Title != "C" || list[1].Title != "A" || list[2].Title != "B" {
			t.Errorf("expected creation (ID) order, got %+v", list)
		}
		for i := 1; i < len(list); i++ {
			if list[i].ID <= list[i-1].ID {
				t.Errorf("IDs must increase: %+v", list)
			}
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
		if list, _ := s.List(ctx); len(list) != 1 || list[0].ID != b.ID {
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
		if list, _ := s.List(ctx); len(list) != 1 {
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

Run against both adapters:

```go
// file: database/product_store_test.go
package database

import (
	"ecommerce/product"
	"ecommerce/product/storetest"
	"testing"
)

var _ product.Repository = (*ProductStore)(nil)

func TestProductStoreContract(t *testing.T) {
	storetest.Run(t, func(t *testing.T) product.Repository { return NewProductStore() })
}
```

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

	list, _ := s.List(ctx)
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
	list, _ := s.List(ctx)
	list[0].Title = "tampered"

	if got, _ := s.Get(ctx, 1); got.Title == "tampered" {
		t.Error("callers must not be able to modify the store's internal slice")
	}
}
```

```go
// file: postgres/product_store_test.go
package postgres

import (
	"context"
	"ecommerce/product"
	"ecommerce/product/storetest"
	"errors"
	"testing"
)

var _ product.Repository = (*ProductStore)(nil)

func TestProductStoreContract(t *testing.T) {
	storetest.Run(t, func(t *testing.T) product.Repository { return NewProductStore(newTestDB(t).DB) })
}

func TestPriceIsStoredWithTwoDecimals(t *testing.T) {
	s := NewProductStore(newTestDB(t).DB)
	got, err := s.Create(context.Background(), product.Product{Title: "Rounded", Price: 19.999})
	if err != nil {
		t.Fatal(err)
	}
	if got.Price != 20 {
		t.Errorf("NUMERIC(12,2) should round 19.999 to 20.00, got %v", got.Price)
	}
}

func TestDatabaseConstraintsBackUpTheGoValidation(t *testing.T) {
	d := newTestDB(t)
	s := NewProductStore(d.DB)
	ctx := context.Background()

	// Bypass the HTTP layer's validation entirely: the schema itself must refuse bad rows.
	for name, p := range map[string]product.Product{
		"zero price":     {Title: "Free", Price: 0},
		"negative price": {Title: "Neg", Price: -5},
		"blank title":    {Title: "   ", Price: 1},
		"title too long": {Title: string(make([]byte, 101)), Price: 1},
	} {
		if _, err := s.Create(ctx, p); err == nil || errors.Is(err, product.ErrNotFound) {
			t.Errorf("%s: the database should have rejected the row, got %v", name, err)
		}
	}
}

func TestUpdateRefreshesUpdatedAt(t *testing.T) {
	d := newTestDB(t)
	s := NewProductStore(d.DB)
	ctx := context.Background()

	created, _ := s.Create(ctx, product.Product{Title: "Old", Price: 1})
	var before, after struct{ Created, Updated string }
	d.Get(&before, "SELECT created_at::text AS created, updated_at::text AS updated FROM products WHERE id = $1", created.ID)

	if _, err := d.Exec("SELECT pg_sleep(0.05)"); err != nil {
		t.Fatal(err)
	}
	s.Update(ctx, created.ID, product.Product{Title: "New", Price: 2})
	d.Get(&after, "SELECT created_at::text AS created, updated_at::text AS updated FROM products WHERE id = $1", created.ID)

	if after.Created != before.Created {
		t.Error("created_at must never change")
	}
	if after.Updated == before.Updated {
		t.Error("updated_at must move forward on update")
	}
}

func TestCancelledContextIsNotANotFound(t *testing.T) {
	s := NewProductStore(newTestDB(t).DB)
	ctx, cancel := context.WithCancel(context.Background())
	cancel()

	if _, err := s.Get(ctx, 1); err == nil || errors.Is(err, product.ErrNotFound) {
		t.Errorf("a cancelled context must give a real error, got %v", err)
	}
	if err := s.Delete(ctx, 1); err == nil || errors.Is(err, product.ErrNotFound) {
		t.Errorf("a cancelled context must give a real error, got %v", err)
	}
}
```

### End-to-end tests: the safety net

The `rest` tests' assertions are **unchanged**; only the helper that builds the app takes the new wiring:

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

func (a *testApp) count() int {
	var list []struct{ ID int }
	json.NewDecoder(a.do(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	return len(list)
}
```

Run everything:

```bash
go vet ./... && go test -race ./...
TEST_DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable' go test -race -count=1 ./...
```

---

## 9. Keeping domains independent

Chapter 60's architecture test guarded *one* domain against technology. Now there are two domains, and a **second rule** appears: *Identity and Catalog must not depend on each other.* One test at the module root guards both rules for all domains; it replaces the per-domain test from Chapter 60:

```go
// file: architecture_test.go
package main

import (
	"go/parser"
	"go/token"
	"strings"
	"testing"
)

// domains are the business packages. Each must stay free of technology and independent of the others.
var domains = []string{"user", "product"}

// technology no domain may import.
var technology = []string{
	"net/http", "database/sql", "encoding/json",
	"github.com/",
	"ecommerce/postgres", "ecommerce/database", "ecommerce/rest", "ecommerce/auth",
	"ecommerce/util", "ecommerce/middleware", "ecommerce/infra", "ecommerce/cmd", "ecommerce/config",
}

func TestDomainsStayPureAndIndependent(t *testing.T) {
	for _, domain := range domains {
		forbidden := append([]string(nil), technology...)
		for _, other := range domains {
			if other != domain {
				forbidden = append(forbidden, "ecommerce/"+other) // domains never import each other
			}
		}

		pkgs, err := parser.ParseDir(token.NewFileSet(), domain, nil, parser.ImportsOnly)
		if err != nil {
			t.Fatal(err)
		}
		for _, pkg := range pkgs {
			for file, f := range pkg.Files {
				if strings.HasSuffix(file, "_test.go") {
					continue // tests may use whatever helps
				}
				for _, imp := range f.Imports {
					path := strings.Trim(imp.Path.Value, `"`)
					for _, bad := range forbidden {
						if strings.HasPrefix(path, bad) {
							t.Errorf("%s imports %q: domain %q must stay free of technology and of other domains", file, path, domain)
						}
					}
				}
			}
		}
	}
}
```

<!-- delete: user/architecture_test.go -->

Two subtle things: the `_test.go` exemption (a domain's tests may use helpers; only *production* code is constrained), and the **error message names the rule**, so a failing build teaches the reader why. When Orders arrives, the rule becomes explicit and one-directional: add `"orders"` to `domains`, and allow *only* `ecommerce/product` and `ecommerce/user` as dependencies of `orders`.

---

## 10. Proving nothing changed

The API is the same as in Chapter 56. Drive the real server, with PostgreSQL behind it, through the full product lifecycle (`$AUTH` is `Authorization: Bearer <token>` from Chapter 53). The Bengali title is 100 *characters* (300 bytes): the old byte-counting rule would have rejected it, the database accepted it, and now the domain agrees:

```
$ curl localhost:18080/products/2
HTTP/1.1 200 OK
{"id":2,"title":"Apple","description":"A crunchy apple a day...","price":40,"imageUrl":"https://example.com/apple.jpg"}

$ curl -X POST localhost:18080/products -H "$AUTH" -d '{"title":"<100 Bengali characters = 300 bytes>","price":30}'
HTTP/1.1 201 Created
Location: /products/4
{"id":4,"title":"বাবাবাবা…(100 characters)","description":"","price":30,"imageUrl":""}

$ curl -X POST … -d '{"title":"<101 characters>","price":30}'
HTTP/1.1 422 Unprocessable Entity
{"error":"title must be at most 100 characters"}

$ curl -X POST … -d '{"title":"   ","price":30}'
HTTP/1.1 422 Unprocessable Entity
{"error":"title is required"}

$ curl -X POST … -d '{"title":"Free","price":0}'
HTTP/1.1 422 Unprocessable Entity
{"error":"price must be greater than 0"}

$ curl -X PUT localhost:18080/products/1 -H "$AUTH" -d '{"title":"  Blood Orange ","description":"deep red","price":120}'
HTTP/1.1 200 OK
{"id":1,"title":"Blood Orange","description":"deep red","price":120,"imageUrl":""}

$ curl -X PUT localhost:18080/products/99 -H "$AUTH" -d '{"title":"x","price":1}'
HTTP/1.1 404 Not Found
{"error":"product not found"}

$ curl -X DELETE localhost:18080/products/3 -H "$AUTH"
HTTP/1.1 204 No Content

$ curl -X DELETE localhost:18080/products/3 -H "$AUTH"        # again
HTTP/1.1 404 Not Found
{"error":"product not found"}
```

And the domain suite (no database, no HTTP, no bcrypt):

```
$ go test -count=1 -v ./product/ | grep -E "^(--- |ok)"
--- PASS: TestCreateStoresTheCleanedUpProduct (0.00s)
--- PASS: TestTheSameRulesApplyToCreateAndUpdate (0.00s)
--- PASS: TestTitleLengthCountsCharactersNotBytes (0.00s)
--- PASS: TestBoundaries (0.00s)
--- PASS: TestUpdateReplacesAndKeepsTheID (0.00s)
--- PASS: TestMissingProductsAreErrNotFound (0.00s)
--- PASS: TestDeleteRemovesOnlyThatProduct (0.00s)
--- PASS: TestStorageFailuresAreWrappedNotDisguised (0.00s)
ok  	ecommerce/product	0.006s
```

---

## 11. Change-impact analysis

| Change | Where |
|--------|-------|
| Titles may be 200 characters | `maxTitleLength` in `product/product.go` + one test line |
| Prices must have at most 2 decimals | `Details.normalized` (+ test); *nothing else* |
| Add a `category` to products | `product.Product`/`Details`, the SQL + migration, `productRow`, request/response types |
| Send an event on price change | a new port (`Publisher`), called from `Service.Update`; adapters elsewhere |
| Cache `Get` in Redis | a `product.Repository` **decorator** (Chapter 51) wrapping `postgres.ProductStore` in `wire.go`; domain and handler untouched |
| Serve the catalog over gRPC too | a new adapter calling `product.Service` |
| Import products from CSV | a command calling `Service.Create` (the rules come free) |
| Replace PostgreSQL with another database | a new `Repository` + the contract suite; domain untouched |

Compare with the picture in Chapter 59, section 2: one business idea, scattered across five directories. Now each row of this table names **one place** (or a small, obvious set).

---

## 12. How big should a file be?

Beginners ask for a rule ("split at 200 lines"). The real rule: **split by responsibility and by the reader's needs, not by line count.**

Signals that a file wants splitting:

- You scroll to find things, or use search for basic navigation.
- Two parts of the file change for *different reasons* (say, validation rules and query building).
- Unrelated things are tested together, or the test file is 3x larger than the code.
- Merge conflicts keep landing in the same file because two people work on different features there.

Signals to *leave it alone*:

- The file is one cohesive idea, even at 300 lines: a state machine, a parser, a table of related constants.
- Splitting would force you to export things only to share them between the halves.

Go makes splitting cheap because **all files in a directory form one package**: moving code between files changes nothing for callers. Examples from this project, and how we'd split as they grow:

```
product/service.go (≈60 lines today)  →  when it grows past several use cases:
    service.go            constructor, shared helpers
    service_create.go     Create, Update (they share validation)
    service_query.go      List, Get
    service_delete.go     Delete

product/product.go  →  product.go (entity), details.go (input + validation) if rules multiply
```

And when the *package* is getting too big (many unrelated concepts), that's a sign to split the **domain**, not just the files: e.g., if `product` grows pricing rules, discounts, and inventory, those may deserve their own packages (`pricing`, `inventory`) with clear interfaces between them.

A practical guideline: keep **functions** small enough to understand at a glance (roughly a screenful), keep **files** cohesive, keep **packages** focused on one business capability, and let names carry the meaning (`service_query.go` beats `utils2.go`).

---

## 13. What makes this structure maintainable?

Not the number of layers: the properties they give us:

| Property | Where you see it |
|----------|------------------|
| **Locality of change** | the change-impact tables in Chapters 60 and 61 |
| **Discoverability** | business rule → domain package; status code → `fail`; SQL → `postgres/queries` |
| **Testability** | rules tested in isolation, adapters tested for translation, wiring tested end to end |
| **Explicit dependencies** | constructors take what they need; no globals; `wire.go` shows the whole graph |
| **Enforced boundaries** | the architecture test fails the build on a bad import |
| **Consistency** | users and products follow the same recipe, so a new developer learns it once |
| **Honest errors** | expected outcomes vs. unexpected failures are distinguished everywhere |

And what does *not* make code maintainable: clever abstractions nobody needs, interfaces with one implementation and no consumer variety, or patterns applied because a book said so. The recurring test is Chapter 59's: *does this structure make the next change easier, or just longer?* If a layer doesn't earn its keep, remove it.

---

## 14. Preparing for real unit testing

Compare how you would test "a title with 101 characters is rejected":

**Before the refactoring** (rule inside an HTTP handler, storage concrete):

```
1. start a PostgreSQL (or accept an in-memory store the handler is glued to)
2. build the whole server, routes, middleware
3. create a JWT and put it in a header
4. marshal a JSON body, send an HTTP request
5. read the response, parse JSON, check the status and message
6. hope nothing else in that pipeline (CORS, logging, auth) interfered
```

**After:**

```go
_, err := NewService(&fakeRepo{}).Create(ctx, Details{Title: strings.Repeat("x", 101), Price: 1})
// assert err is a *ValidationError mentioning "100 characters"
```

Two lines, microseconds, no I/O: which is why the domain suite has dozens of cases and still runs in milliseconds, and why you can write **table-driven tests for every edge** without a second thought. The test pyramid falls out naturally:

```
        /\         few   end-to-end tests: whole HTTP stack + wiring          (rest/)
       /  \
      /----\       some  adapter tests: JSON <-> status codes, SQL <-> entity (userhandler, producthandler, postgres)
     /------\
    /--------\     many  domain unit tests: rules and use cases with fakes    (user, product)
```

The next chapters (concurrency) will not add tests to the domain, but the discipline carries over: **if something is hard to test, the design is trying to tell you something.**

---

## 15. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Domain reusing `models`/DB structs with tags | Shared package couples every feature | Own entity per domain; row/response types in adapters |
| 2 | Validation in both the handler and the service | Two rules that drift apart | Domain only; the handler just decodes |
| 3 | Counting title length with `len()` | Bytes vs. characters mismatch with the database | `utf8.RuneCountInString` |
| 4 | `price <= 0` alone as the guard | `NaN` slips through | Explicit `IsNaN`/`IsInf` checks |
| 5 | Service returns `sql.ErrNoRows` / HTTP codes | Layers leak | Domain errors; the adapter translates |
| 6 | Update validating *after* the repository call | A malformed request may hit the database | Validate first |
| 7 | Handler returns a `nil` slice for empty lists | JSON `null` | Build a non-nil response slice |
| 8 | Domain importing another domain's internals | Cycles, tangled dependencies | IDs or narrow interfaces; the architecture test |
| 9 | Splitting files by arbitrary line counts | Fragmented, hard to navigate code | Split by responsibility |
| 10 | Splitting the *package* to shorten files | Import cycles, exported-only-for-sharing symbols | Split files; split packages only along real boundaries |
| 11 | Untested fakes | Fakes diverge from real adapters | Contract suites |
| 12 | Wrapping *every* error, including `ErrNotFound` | Handlers must unwrap; log noise | Pass expected outcomes through, wrap only unexpected failures |
| 13 | Forgetting to delete the old code | Two implementations, confusion | Delete markers; `grep` for the old imports |
| 14 | Skipping the end-to-end tests during refactoring | Broken wiring shipped | Keep them green at every step |

---

## 16. Interview questions

**Q1. How did you decide what belongs in the domain vs. the handler?**
Anything a business person could state as a rule (title required, price > 0, ≤ 100 characters) is domain; anything about JSON, status codes, or URLs is the adapter.

**Q2. Why a separate `Details` type?**
It models "what a client may specify" (no ID), so create and update share one set of rules and the entity's ID stays under the system's control.

**Q3. What bug did centralizing the rules fix?**
Title length counted in bytes in the handler vs. characters in the database; one central rule using `utf8.RuneCountInString` fixed it everywhere.

**Q4. Why retire the `models` package?**
A shared bucket of types couples all features. Each domain should own its entities, and adapters own their representations.

**Q5. How do you test that domains don't depend on each other?**
An architecture test that parses each domain package's imports and fails on forbidden ones (technology packages and other domains).

**Q6. Where would caching live?**
As a `Repository` decorator composed in `wire.go`; the domain, handlers, and other adapters are unaffected.

**Q7. When should a file be split?**
When it has multiple responsibilities that change for different reasons, hurts navigation, or causes merge conflicts, not merely at a line count.

**Q8. Why do fakes need contract tests?**
Otherwise a fake can behave differently from the real adapter and give false confidence. Running the same suite against both keeps them honest.

**Q9. What's the trade-off of this architecture for a simple CRUD like products?**
More files and mapping code than a plain handler-plus-SQL; worth it only if rules, teams, or entry points will grow. Use the smallest structure that serves the complexity.

---

## 17. Exercises

### Exercise 1: Share the `wrap` helper
The user service repeats the "pass `ErrNotFound` through, wrap the rest" logic. Extract one helper, and make sure the user tests stay green.

<details><summary>Solution</summary>

In `user/service.go`, add `func wrap(what string, err error, passThrough ...error) error` (or two small helpers) and use it in `Get`, and adapt `Register` (`ErrEmailTaken`) and `Login`. The existing tests (`TestRegisterWrapsUnexpectedFailures`, `TestGet`, …) prove the behavior is unchanged: exactly what they're for.
</details>

### Exercise 2: A price rule
Products with a price above 1,000,000 need manual review and must be rejected by the API with the message "price exceeds the automatic limit". Where do you implement it, and what tests do you add?

<details><summary>Solution</summary>

In `Details.normalized` (a new `case` and constant), plus table entries in `TestTheSameRulesApplyToCreateAndUpdate` and a boundary in `TestBoundaries` (`1_000_000` accepted, `1_000_000.01` rejected). No handler or adapter change; the HTTP tests need nothing because the `ValidationError` already maps to 422.
</details>

### Exercise 3: A decorator repository
Write `LoggingRepository` implementing `product.Repository` that logs operation and duration, wrapping another repository. Insert it in `wire.go`. Which files change?

<details><summary>Solution</summary>

A new file (say `postgres/logging.go`, or a package `decorate`), plus one changed line in `cmd/wire.go`: `productRepo := decorate.Logging(postgres.NewProductStore(conn), logger)`. The domain, handler, and tests are untouched, and you can run the **contract suite** against the decorator wrapping the in-memory store to prove it changes no behavior.
</details>

### Exercise 4: Pagination in the domain
Add `Page(ctx, limit, offset int)` to the repository port and service, with the rule "limit between 1 and 100, default 20". Where does the default live?

<details><summary>Solution</summary>

The **domain** owns the rule and the default (a `PageRequest` value object with `NewPageRequest(limit, offset)` returning a `ValidationError`), because "default page size" is business policy, not an HTTP detail. The handler parses `?limit=&offset=` strings and passes ints; the SQL adapter implements `LIMIT/OFFSET` (Chapters 62–63).
</details>

### Exercise 5: A price value object
Replace `Price float64` by a `Price` value object holding integer cents, with `NewPrice(cents)` rejecting `<= 0`. What has to change in the row type and the JSON types?

<details><summary>Solution</summary>

`Product.Price` becomes `Price`; `productRow` reads `price` as `NUMERIC` → cents (e.g., `SELECT (price*100)::bigint AS price_cents`); the request/response types convert between JSON decimals (parsed from strings/`json.Number`) and cents at the boundary. All the domain's `Price` rules move into `NewPrice` and the `IsNaN` special-casing disappears (integers can't be NaN).
</details>

### Exercise 6 (challenge): A third domain
Add an `orders` domain skeleton (entity `Order` with lines referencing product IDs, `Repository` port, `Service.Place`) that needs product prices. Design the port through which `orders` asks the catalog for a price, and extend the architecture test accordingly.

<details><summary>Solution</summary>

In `orders`, define `type PriceLookup interface { PriceOf(ctx, productID int) (float64, error) }` and take it in `NewService`. An adapter in `cmd`/`adapters` implements it by calling `product.Service.Get`. Update the architecture test: `orders` may import nothing from `ecommerce/…` (it depends on the *interface*, not on `product`), so the independence rule holds without exceptions.
</details>

---

## 18. Quiz

1. Which file now holds the rule "title at most 100 characters"?
2. Why did the old rule disagree with the database for Bengali titles?
3. What does the architecture test forbid?
4. Who maps `product.ErrNotFound` to `404`?
5. Why is `Details` separate from `Product`?
6. What replaced the shared `models` package?
7. How would you add caching without touching the domain?
8. What is the test pyramid here?

<details><summary>Answers</summary>

1. `product/product.go` (`Details.normalized`).
2. It counted bytes (`len`), the database counts characters.
3. Domains importing technology packages or each other.
4. The HTTP adapter's `fail` function.
5. `Details` is what a client may specify (no ID); `Product` is the entity with identity; create and update share the same rules.
6. Nothing: each domain owns its types; adapters own their row/JSON shapes.
7. A `product.Repository` decorator wired in `cmd/wire.go`.
8. Many domain unit tests, some adapter tests, few end-to-end tests.
</details>

---

## 19. Summary

- The **product domain** (`package product`) has its own **entity**, **`Details`** input, **domain errors**, a **`Repository` port**, and a ~50-line **`Service`**: rules exist **once**, and the service returns expected outcomes (`ErrNotFound`, `*ValidationError`) untouched and wraps everything else.
- Centralizing the rules made a real bug a one-line fix (**characters, not bytes**) and closed a `NaN` loophole; regression tests pin both.
- **Adapters** are thin: `producthandler` (decode → call → one `fail` function), `postgres.ProductStore` with a private `productRow`, the in-memory store; all verified by the **repository contract suite**.
- The shared **`models` package is gone**, and an **architecture test** enforces two rules for every domain: no technology imports, no imports of other domains.
- **Files:** split by responsibility (and packages by capability), not by line count; **maintainability** comes from locality of change, explicit dependencies, enforced boundaries, and consistency: not from the number of layers.
- The domain's unit tests need **no server, no database, no bcrypt**: the payoff that makes the next chapters' concurrency work and any future feature far easier to verify.

### ➡️ What's next?

**Part 13: pagination.** [Chapter 62](62-experiments-before-optimizing.md) explores the *problem* of listing large catalogs (why returning everything breaks down, with measurements), and [Chapter 63](63-pagination.md) implements **pagination** properly: query parameters, `LIMIT/OFFSET`, keyset pagination, and the new port in our domain.
