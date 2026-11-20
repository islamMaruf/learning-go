# Chapter 56: CRUD in Go — A PostgreSQL Product Store with `Update` and `Delete`

> **Goal of this chapter:** Finish the move from memory to PostgreSQL. You'll create the `products` table, implement **all five store operations** (`List`, `Get`, `Create`, `Update`, `Delete`) with `sqlx`, add the new `PUT /products/{id}` and `DELETE /products/{id}` endpoints, and prove that the in-memory and PostgreSQL stores behave **identically** with a shared contract test. Along the way: `SelectContext` vs. `GetContext` vs. `ExecContext`, iterating rows safely (`rows.Close`, `rows.Err`), detecting "not found" from `RowsAffected`, the empty-list-is-`[]`-not-`null` trap, `204 No Content`, and `PUT` semantics.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 7 hours  **Prerequisite:** [Chapters 53, 54 and 55](55-sql-crud.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The plan](#2-the-plan)
3. [The `products` table](#3-the-products-table)
4. [The `sqlx` toolbox](#4-the-sqlx-toolbox)
5. [The queries](#5-the-queries)
6. [Extending the `Store` interface](#6-extending-the-store-interface)
7. [The PostgreSQL `ProductStore`](#7-the-postgresql-productstore)
8. [Iterating rows by hand](#8-iterating-rows-by-hand)
9. [The in-memory store keeps up](#9-the-in-memory-store-keeps-up)
10. [Handlers: `PUT` and `DELETE`](#10-handlers-put-and-delete)
11. [Wiring](#11-wiring)
12. [Tests](#12-tests)
13. [Trying it with `curl`](#13-trying-it-with-curl)
14. [Why the repository pattern paid off](#14-why-the-repository-pattern-paid-off)
15. [Common mistakes](#15-common-mistakes)
16. [Interview questions](#16-interview-questions)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. What you will learn

- Applying the Chapter 54 design: the real `products` table, with constraints
- `sqlx` methods: `GetContext`, `SelectContext`, `ExecContext`, `QueryxContext`, and when to use each
- Safe manual row iteration: `defer rows.Close()`, `rows.Err()`
- `UPDATE … RETURNING` and `DELETE` with `RowsAffected` → `models.ErrNotFound`
- `PUT` (replace) vs. `PATCH` (partial) vs. `POST`; `200` vs. `201` vs. `204`
- The **empty result** trap: `[]` vs. `null` in JSON
- A **contract test** for `product.Store`, run on both implementations
- Extending an interface without breaking anything: what the compiler tells you to change

---

## 2. The plan

```
                 HTTP                          product package                     storage
 GET    /products         ─┐                ┌──────────────────────┐
 GET    /products/{id}    ─┤                │  Handler             │        ┌──────────────────────┐
 POST   /products  (auth) ─┼──────────────► │   List Get Create    │ ─────► │ postgres.ProductStore│  ← production
 PUT    /products/{id} 🆕 ─┤                │   Update 🆕 Delete 🆕 │  Store │  (sqlx + SQL files)  │
 DELETE /products/{id} 🆕 ─┘                └──────────────────────┘  iface └──────────────────────┘
                                                                            ┌──────────────────────┐
                                                                            │ database.ProductStore│  ← tests/fakes
                                                                            └──────────────────────┘
```

Steps: table → queries → interface → PostgreSQL store → in-memory store → handlers → wiring → tests. Notice the **order**: when the interface grows two methods, the compiler immediately shows every implementation that must follow (and nothing else).

---

## 3. The `products` table

The design from Chapter 54, saved as the second schema file (Chapter 58 replaces this manual step with migrations):

```sql
-- file: sql/002_create_products.sql
CREATE TABLE products (
    id          BIGINT        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title       TEXT          NOT NULL CHECK (char_length(btrim(title)) BETWEEN 1 AND 100),
    description TEXT          NOT NULL DEFAULT '',
    price       NUMERIC(12,2) NOT NULL CHECK (price > 0),
    image_url   TEXT          NOT NULL DEFAULT '',
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT now()
);
```

And optional demo data (kept out of `sql/` top level so the test helper won't load it as schema):

```sql
-- file: sql/seed/products.sql
INSERT INTO products (title, description, price, image_url) VALUES
  ('Orange', 'Orange is juicy and full of vitamin C.', 100, 'https://example.com/orange.jpg'),
  ('Apple',  'A crunchy apple a day...',                40, 'https://example.com/apple.jpg'),
  ('Banana', 'Great for a quick snack.',                 5, 'https://example.com/banana.jpg');
```

```bash
docker exec -i shop-db psql -U postgres -d ecommerce -v ON_ERROR_STOP=1 < sql/002_create_products.sql
docker exec -i shop-db psql -U postgres -d ecommerce -v ON_ERROR_STOP=1 < sql/seed/products.sql
```

```
CREATE TABLE
INSERT 0 3
```

The `created_at` / `updated_at` columns exist in the database for auditing but are **not** in the Go model: the API doesn't expose them (yet). A table can have columns your struct ignores, as long as your `SELECT` doesn't list them.

---

## 4. The `sqlx` toolbox

`sqlx` extends `database/sql`. The methods you'll use, and the rule for choosing:

| Method | Returns | Use for |
|--------|---------|---------|
| `GetContext(ctx, &dest, query, args…)` | **exactly one row** scanned into `dest` (struct or scalar); `sql.ErrNoRows` if none | `SELECT … WHERE id = $1`, and `INSERT/UPDATE … RETURNING` (they return one row) |
| `SelectContext(ctx, &slice, query, args…)` | **all rows** appended to a slice of structs/scalars | lists |
| `ExecContext(ctx, query, args…)` | a `sql.Result` (`RowsAffected()`, `LastInsertId()` (not supported by PostgreSQL)) | statements with no rows to return: `DELETE`, plain `UPDATE` |
| `QueryxContext(ctx, query, args…)` | a `*sqlx.Rows` cursor you loop over | huge results processed one row at a time |
| `NamedExecContext` / `NamedQueryContext` | like above, with `:name` placeholders bound from a struct | many parameters (Exercise 5) |

Plain `Get`/`Select`/`Exec` (without `Context`) exist too. **Always use the `…Context` versions**: they're what make cancellation and timeouts work (Chapter 52).

Two behaviors worth knowing:

- `GetContext` on a query returning **several** rows uses the *first* and ignores the rest, so add `WHERE` (or `LIMIT 1`) deliberately.
- `SelectContext` scans **into the slice you pass** and *appends*. If nothing matched, a `nil` slice stays `nil`. We'll see why that matters for JSON in §7.

---

## 5. The queries

Column lists are explicit and identical everywhere, so the `Product` struct's `db` tags line up:

```sql
-- file: postgres/queries/products_list.sql
SELECT id, title, description, price, image_url
FROM products
ORDER BY id
```

```sql
-- file: postgres/queries/products_by_id.sql
SELECT id, title, description, price, image_url
FROM products
WHERE id = $1
```

```sql
-- file: postgres/queries/products_insert.sql
INSERT INTO products (title, description, price, image_url)
VALUES ($1, $2, $3, $4)
RETURNING id, title, description, price, image_url
```

```sql
-- file: postgres/queries/products_update.sql
UPDATE products
SET title = $2, description = $3, price = $4, image_url = $5, updated_at = now()
WHERE id = $1
RETURNING id, title, description, price, image_url
```

```sql
-- file: postgres/queries/products_delete.sql
DELETE FROM products WHERE id = $1
```

Notes:

- `ORDER BY id` gives a **deterministic order** (Chapter 55). Without it, the order can change after updates.
- The `UPDATE` sets **every** column: `PUT` means "replace the whole resource". It also refreshes `updated_at` (the database doesn't do that automatically).
- `RETURNING` on `INSERT`/`UPDATE` gives us the stored row (with the price rounded to two decimals by `NUMERIC(12,2)`) without a second query.
- `$1` is the ID in `UPDATE` (it identifies the row), then `$2…$5` are the new values. Keep parameter numbering easy to audit against the Go call.

---

## 6. Extending the `Store` interface

Two new methods, each documenting its error contract:

```go
// file: product/store.go
package product

import (
	"context"
	"ecommerce/models"
)

// Store is what the product feature needs from a storage backend.
// It is defined here, next to its only user, and is as small as it can be.
type Store interface {
	// List returns every product ordered by ID; the slice is empty (never nil) when there are none.
	List(ctx context.Context) ([]models.Product, error)
	// Get returns models.ErrNotFound if there is no product with that ID.
	Get(ctx context.Context, id int) (models.Product, error)
	// Create stores p (ignoring p.ID) and returns it with its assigned ID.
	Create(ctx context.Context, p models.Product) (models.Product, error)
	// Update replaces the product with the given ID. It returns models.ErrNotFound if there is none.
	Update(ctx context.Context, id int, p models.Product) (models.Product, error)
	// Delete removes the product with the given ID. It returns models.ErrNotFound if there is none.
	Delete(ctx context.Context, id int) error
}
```

Build now and the compiler complains, exactly where work is needed:

```
cmd/wire.go: cannot use productStore (variable of type *database.ProductStore) as product.Store value in argument
    to product.NewHandler: *database.ProductStore does not implement product.Store (missing method Delete)
```

That's the interface doing its job: it tells us which implementations are incomplete.

The model gets `db` tags (and the same JSON as before):

```go
// file: models/product.go
package models

// Product is the data model of the shop.
type Product struct {
	ID          int     `json:"id"          db:"id"`
	Title       string  `json:"title"       db:"title"`
	Description string  `json:"description" db:"description"`
	Price       float64 `json:"price"       db:"price"`
	ImageURL    string  `json:"imageUrl"    db:"image_url"`
}
```

`Price float64` scanning from `NUMERIC(12,2)` works (the driver hands `12.50` as text and `database/sql` converts it). As Chapter 54 discussed, that's a teaching compromise; Exercise 6 replaces it with integer cents.

---

## 7. The PostgreSQL `ProductStore`

```go
// file: postgres/product_store.go
package postgres

import (
	"context"
	"database/sql"
	"ecommerce/models"
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

// ProductStore keeps products in PostgreSQL. It is safe for concurrent use: *sqlx.DB is a connection pool.
type ProductStore struct {
	db *sqlx.DB
}

// NewProductStore creates a store on top of an existing connection pool (see infra/db).
func NewProductStore(db *sqlx.DB) *ProductStore { return &ProductStore{db: db} }

// List returns every product ordered by ID. The result is an empty (non-nil) slice when there are none,
// so it encodes as JSON [] rather than null.
func (s *ProductStore) List(ctx context.Context) ([]models.Product, error) {
	products := make([]models.Product, 0)
	if err := s.db.SelectContext(ctx, &products, listProductsSQL); err != nil {
		return nil, fmt.Errorf("list products: %w", err)
	}
	return products, nil
}

// Get returns the product with the given ID, or models.ErrNotFound.
func (s *ProductStore) Get(ctx context.Context, id int) (models.Product, error) {
	return s.one(ctx, "get product", productByIDSQL, id)
}

// Create inserts p (ignoring p.ID) and returns the stored row.
func (s *ProductStore) Create(ctx context.Context, p models.Product) (models.Product, error) {
	return s.one(ctx, "insert product", insertProductSQL, p.Title, p.Description, p.Price, p.ImageURL)
}

// Update replaces the product with the given ID and returns the stored row, or models.ErrNotFound.
func (s *ProductStore) Update(ctx context.Context, id int, p models.Product) (models.Product, error) {
	return s.one(ctx, "update product", updateProductSQL, id, p.Title, p.Description, p.Price, p.ImageURL)
}

// Delete removes the product with the given ID, or returns models.ErrNotFound if there is none.
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
		return models.ErrNotFound
	}
	return nil
}

// one runs a query that returns a single product row and translates "no rows" into models.ErrNotFound.
func (s *ProductStore) one(ctx context.Context, what, query string, args ...any) (models.Product, error) {
	var p models.Product
	if err := s.db.GetContext(ctx, &p, query, args...); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return models.Product{}, models.ErrNotFound
		}
		return models.Product{}, fmt.Errorf("%s: %w", what, err)
	}
	return p, nil
}
```

Design points:

- **One helper (`one`) for four methods**: `Get`, `Create`, `Update` all "run a query that yields one row, translate no-rows". `Create` can never hit `ErrNoRows` (an insert always returns its row), but sharing the helper costs nothing and keeps the error handling in one place.
- **`Update` gets `ErrNotFound` for free**: `UPDATE … WHERE id = $1 RETURNING …` returns *zero rows* when no product has that ID, and `GetContext` turns that into `sql.ErrNoRows`. That's the "RETURNING and check for no rows" technique from Chapter 55.
- **`Delete` has nothing to return**, so it uses `ExecContext` and inspects **`RowsAffected()`**: `0` → nobody had that ID → `ErrNotFound`. (Deleting a missing row isn't a database error; *we* decide it's "not found" for the API.)
- **`make([]models.Product, 0)`** matters. `SelectContext` leaves a nil slice untouched when there are no rows; Go's `encoding/json` marshals a nil slice as **`null`** but an empty non-nil slice as **`[]`**. A JavaScript client doing `products.map(...)` on `null` crashes; on `[]` it works. Contract test scenario "empty list" (below) guards it.
- The wrapped errors (`"list products: %w"`) give logs a *what*, while `errors.Is/As` still work.

---

## 8. Iterating rows by hand

`SelectContext` is perfect for lists that fit in memory. When you need to process rows one at a time (a large export) or do custom work per row, use a cursor. This is where beginners leak connections, so learn the pattern exactly:

```go
func (s *ProductStore) forEach(ctx context.Context, fn func(models.Product) error) error {
	rows, err := s.db.QueryxContext(ctx, listProductsSQL)
	if err != nil {
		return fmt.Errorf("query products: %w", err)
	}
	defer rows.Close() // ← ALWAYS: returns the connection to the pool

	for rows.Next() {
		var p models.Product
		if err := rows.StructScan(&p); err != nil {
			return fmt.Errorf("scan product: %w", err)
		}
		if err := fn(p); err != nil {
			return err
		}
	}
	return rows.Err() // ← ALWAYS check: an error mid-stream ends the loop with Next() == false
}
```

The three rules:

1. **`defer rows.Close()`** right after the error check. While `rows` is open it **holds one pooled connection**. Forget it (or return early from the loop without it) and the connection leaks; leak enough and the whole app hangs waiting for a free connection (Chapter 52).
2. **`rows.Next()` returns `false` for two different reasons**: no more rows, *or an error*. Only `rows.Err()` tells them apart, so check it after the loop.
3. Use `StructScan` (sqlx) instead of `Scan(&a, &b, …)` to avoid column-order mistakes.

`SelectContext` does all of this internally, which is why it's the default choice.

---

## 9. The in-memory store keeps up

The in-memory store implements the two new methods, so tests and demos can still use it:

```go
// file: database/product_store.go
package database

import (
	"context"
	"ecommerce/models"
	"sync"
)

// ProductStore keeps products in memory. It is safe for concurrent use.
// Production uses the PostgreSQL store; this one backs fast tests and demos.
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

// SampleProducts returns the demo data used by tests.
func SampleProducts() []models.Product {
	return []models.Product{
		{ID: 1, Title: "Orange", Description: "Orange is juicy and full of vitamin C.", Price: 100, ImageURL: "https://example.com/orange.jpg"},
		{ID: 2, Title: "Apple", Description: "A crunchy apple a day...", Price: 40, ImageURL: "https://example.com/apple.jpg"},
		{ID: 3, Title: "Banana", Description: "Great for a quick snack.", Price: 5, ImageURL: "https://example.com/banana.jpg"},
	}
}

// List returns a copy of all products, ordered by ID.
func (s *ProductStore) List(_ context.Context) ([]models.Product, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	out := make([]models.Product, len(s.products))
	copy(out, s.products)
	return out, nil
}

// Get returns the product with the given ID, or models.ErrNotFound.
func (s *ProductStore) Get(_ context.Context, id int) (models.Product, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	if i := s.indexOf(id); i >= 0 {
		return s.products[i], nil
	}
	return models.Product{}, models.ErrNotFound
}

// Create assigns the next ID to p, stores it, and returns the stored product.
func (s *ProductStore) Create(_ context.Context, p models.Product) (models.Product, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	p.ID = s.nextID
	s.nextID++
	s.products = append(s.products, p)
	return p, nil
}

// Update replaces the product with the given ID, or returns models.ErrNotFound.
func (s *ProductStore) Update(_ context.Context, id int, p models.Product) (models.Product, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	i := s.indexOf(id)
	if i < 0 {
		return models.Product{}, models.ErrNotFound
	}
	p.ID = id
	s.products[i] = p
	return p, nil
}

// Delete removes the product with the given ID, or returns models.ErrNotFound.
func (s *ProductStore) Delete(_ context.Context, id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	i := s.indexOf(id)
	if i < 0 {
		return models.ErrNotFound
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

Small design notes: `indexOf` documents "caller must hold the lock" (a very common convention for private helpers); `Update` forces `p.ID = id` so a client can't change a product's identity by sending a different ID in the body.

---

## 10. Handlers: `PUT` and `DELETE`

### HTTP semantics

| Method | Meaning | Idempotent? | Success status |
|--------|---------|-------------|----------------|
| `POST /products` | create a new resource; server picks the ID | no (twice = two products) | `201 Created` + `Location` |
| `PUT /products/{id}` | **replace** the whole resource with the body | **yes** (same body twice = same result) | `200 OK` with the new representation |
| `PATCH /products/{id}` | change **some** fields | not necessarily | `200 OK` |
| `DELETE /products/{id}` | remove | **yes** in effect, though the second call reports `404` | `204 No Content` (no body) |

*Idempotent* means repeating the request has the same effect as doing it once, which is why clients and proxies may safely retry `PUT` and `DELETE` after a network hiccup, but not `POST`. We implement `PUT` (full replacement, so all required fields must be sent); `PATCH` is Exercise 4.

### Shared request handling

`Create` and `Update` accept the same body and follow the same rules, so the type and validation move into one file (it was `createRequest` in Chapter 51):

```go
// file: product/request.go
package product

import (
	"ecommerce/models"
	"ecommerce/util"
	"net/http"
	"strconv"
	"strings"
)

// productRequest is what a client may send to create or replace a product.
// It deliberately has no ID: the server assigns it (or takes it from the URL).
type productRequest struct {
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}

func (r productRequest) validate() string {
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

func (r productRequest) toProduct() models.Product {
	return models.Product{
		Title:       strings.TrimSpace(r.Title),
		Description: r.Description,
		Price:       r.Price,
		ImageURL:    r.ImageURL,
	}
}

// decode reads and validates a request body. On failure it has already written the error response.
func decode(w http.ResponseWriter, r *http.Request) (productRequest, bool) {
	var req productRequest
	if !util.DecodeJSON(w, r, &req) {
		return req, false
	}
	if msg := req.validate(); msg != "" {
		util.SendError(w, http.StatusUnprocessableEntity, msg)
		return req, false
	}
	return req, true
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
```

```go
// file: product/create.go
package product

import (
	"ecommerce/util"
	"fmt"
	"net/http"
)

// Create handles POST /products. The route requires authentication (see Routes).
func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
	req, ok := decode(w, r)
	if !ok {
		return
	}

	p, err := h.store.Create(r.Context(), req.toProduct())
	if err != nil {
		util.ServerError(w, r, h.logger, err)
		return
	}

	w.Header().Set("Location", fmt.Sprintf("/products/%d", p.ID))
	util.SendData(w, http.StatusCreated, p)
}
```

```go
// file: product/get.go
package product

import (
	"ecommerce/models"
	"ecommerce/util"
	"errors"
	"net/http"
)

// Get handles GET /products/{id}.
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
	id, ok := pathID(w, r)
	if !ok {
		return
	}

	p, err := h.store.Get(r.Context(), id)
	switch {
	case errors.Is(err, models.ErrNotFound):
		util.SendError(w, http.StatusNotFound, "product not found")
	case err != nil:
		util.ServerError(w, r, h.logger, err) // unexpected: log the cause, hide it from the client
	default:
		util.SendData(w, http.StatusOK, p)
	}
}
```

### The two new handlers

```go
// file: product/update.go
package product

import (
	"ecommerce/models"
	"ecommerce/util"
	"errors"
	"net/http"
)

// Update handles PUT /products/{id}: it replaces the product with the request body.
// The route requires authentication (see Routes).
func (h *Handler) Update(w http.ResponseWriter, r *http.Request) {
	id, ok := pathID(w, r)
	if !ok {
		return
	}
	req, ok := decode(w, r)
	if !ok {
		return
	}

	p, err := h.store.Update(r.Context(), id, req.toProduct())
	switch {
	case errors.Is(err, models.ErrNotFound):
		util.SendError(w, http.StatusNotFound, "product not found")
	case err != nil:
		util.ServerError(w, r, h.logger, err)
	default:
		util.SendData(w, http.StatusOK, p)
	}
}
```

```go
// file: product/delete.go
package product

import (
	"ecommerce/models"
	"ecommerce/util"
	"errors"
	"net/http"
)

// Delete handles DELETE /products/{id}. The route requires authentication (see Routes).
func (h *Handler) Delete(w http.ResponseWriter, r *http.Request) {
	id, ok := pathID(w, r)
	if !ok {
		return
	}

	err := h.store.Delete(r.Context(), id)
	switch {
	case errors.Is(err, models.ErrNotFound):
		util.SendError(w, http.StatusNotFound, "product not found")
	case err != nil:
		util.ServerError(w, r, h.logger, err)
	default:
		w.WriteHeader(http.StatusNoContent) // success, and nothing to say
	}
}
```

Choices worth noticing:

- **Order of checks**: bad ID (`400`) → bad body (`400`/`422`) → not found (`404`). Cheap validation first; touch the database last.
- **`204 No Content`** has **no body**, so we only write the header. Returning `200` with `{}` also works, but `204` is the conventional "done, nothing to return".
- Both write routes need **authentication**; reads stay public.

Register the routes:

```go
// file: product/handler.go
package product

import (
	"log/slog"
	"net/http"
)

// Handler serves the product endpoints. It depends only on the Store interface, never on a concrete database type.
type Handler struct {
	store  Store
	logger *slog.Logger
}

// NewHandler creates a Handler. Accept interfaces, return structs.
func NewHandler(store Store, logger *slog.Logger) *Handler {
	return &Handler{store: store, logger: logger}
}

// Routes registers this feature's endpoints on mux.
// authn is applied to the routes that change data.
func (h *Handler) Routes(mux *http.ServeMux, authn func(http.Handler) http.Handler) {
	mux.HandleFunc("GET /products", h.List)
	mux.HandleFunc("GET /products/{id}", h.Get)
	mux.Handle("POST /products", authn(http.HandlerFunc(h.Create)))
	mux.Handle("PUT /products/{id}", authn(http.HandlerFunc(h.Update)))
	mux.Handle("DELETE /products/{id}", authn(http.HandlerFunc(h.Delete)))
}
```

(CORS was ready for this: our allowed-methods list already included `PUT` and `DELETE` back in Chapter 42.)

---

## 11. Wiring

Both stores now come from PostgreSQL, and the composition root gets simpler; the only file that knew about "which storage" changes by two words:

```go
// file: cmd/wire.go
package cmd

import (
	"ecommerce/config"
	"ecommerce/health"
	"ecommerce/postgres"
	"ecommerce/product"
	"ecommerce/rest"
	"ecommerce/user"
	"log/slog"
	"net/http"

	"github.com/jmoiron/sqlx"
)

// buildHandler is the composition root: it creates every component, hands each one
// exactly the dependencies it needs, and returns the finished HTTP handler.
func buildHandler(cfg *config.Config, logger *slog.Logger, conn *sqlx.DB) http.Handler {
	// storage: chosen here and nowhere else
	productStore := postgres.NewProductStore(conn)
	userStore := postgres.NewUserStore(conn)

	productHandler := product.NewHandler(productStore, logger)
	userHandler := user.NewHandler(userStore, []byte(cfg.JWTSecret), cfg.JWTTTL, logger)
	healthHandler := health.NewHandler(conn, logger)

	return rest.NewServer(cfg, logger, productHandler, userHandler, healthHandler).Handler()
}
```

The in-memory `database` package is no longer imported by production code, only by tests. That's fine and intentional: it is the fast fake.

---

## 12. Tests

### The contract for `product.Store`

The same idea as Chapter 53's user contract: write the expected behavior once, run it against every implementation.

```go
// file: product/storetest/storetest.go
// Package storetest holds a behavior suite that every product.Store implementation must pass.
package storetest

import (
	"context"
	"ecommerce/models"
	"ecommerce/product"
	"errors"
	"sync"
	"testing"
)

// Factory returns a fresh, empty store for one test.
type Factory func(t *testing.T) product.Store

func sample(title string, price float64) models.Product {
	return models.Product{Title: title, Description: "d", Price: price, ImageURL: "https://example.com/x.jpg"}
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
		if _, err := s.Get(ctx, 12345); !errors.Is(err, models.ErrNotFound) {
			t.Errorf("Get: got %v", err)
		}
		if _, err := s.Update(ctx, 12345, sample("X", 1)); !errors.Is(err, models.ErrNotFound) {
			t.Errorf("Update: got %v", err)
		}
		if err := s.Delete(ctx, 12345); !errors.Is(err, models.ErrNotFound) {
			t.Errorf("Delete: got %v", err)
		}
	})

	t.Run("update replaces every field and keeps the ID", func(t *testing.T) {
		s := newStore(t)
		created, _ := s.Create(ctx, sample("Mango", 75.5))
		other, _ := s.Create(ctx, sample("Other", 2))

		replacement := models.Product{Title: "Alphonso", Description: "new", Price: 120, ImageURL: "https://example.com/a.jpg"}
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
		if _, err := s.Get(ctx, a.ID); !errors.Is(err, models.ErrNotFound) {
			t.Errorf("deleted product still found: %v", err)
		}
		if err := s.Delete(ctx, a.ID); !errors.Is(err, models.ErrNotFound) {
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

Run against the in-memory store (fast, no database):

```go
// file: database/product_store_test.go
package database

import (
	"ecommerce/product"
	"ecommerce/product/storetest"
	"testing"
)

var _ product.Store = (*ProductStore)(nil)

func TestProductStoreContract(t *testing.T) {
	storetest.Run(t, func(t *testing.T) product.Store { return NewProductStore() })
}
```

Run against PostgreSQL. The test helper from Chapter 53 now applies **every numbered schema file** to the fresh per-test schema (so it automatically picks up `002_create_products.sql`, and any future migration files):

```go
// file: postgres/testdb_test.go
package postgres

import (
	"context"
	"ecommerce/infra/db"
	"fmt"
	"net/url"
	"os"
	"path/filepath"
	"sync/atomic"
	"testing"
	"time"

	"github.com/jmoiron/sqlx"
)

var schemaCounter atomic.Int64

// testDB is an isolated schema plus the means to open more pools on it.
type testDB struct {
	*sqlx.DB        // a pool on the isolated schema
	dsn      string // reconnect with this to get a second, independent pool
}

func opts(u string) db.Options {
	return db.Options{URL: u, MaxOpenConns: 8, MaxIdleConns: 4, ConnMaxLifetime: time.Minute}
}

// newTestDB returns a pool whose search_path is a brand-new, empty schema with every numbered
// sql/NNN_*.sql file applied in order. The schema is dropped when the test ends.
// The test is skipped without TEST_DATABASE_URL.
func newTestDB(t *testing.T) *testDB {
	t.Helper()
	base := os.Getenv("TEST_DATABASE_URL")
	if base == "" {
		t.Skip("set TEST_DATABASE_URL to run PostgreSQL integration tests")
	}
	ctx := context.Background()

	schema := fmt.Sprintf("t_%d_%d", os.Getpid(), schemaCounter.Add(1))

	admin, err := db.Connect(ctx, opts(base))
	if err != nil {
		t.Fatal(err)
	}
	if _, err := admin.ExecContext(ctx, "CREATE SCHEMA "+schema); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() {
		admin.ExecContext(ctx, "DROP SCHEMA "+schema+" CASCADE")
		admin.Close()
	})

	u, err := url.Parse(base)
	if err != nil {
		t.Fatal(err)
	}
	q := u.Query()
	q.Set("search_path", schema) // every connection in this pool sees only our schema
	u.RawQuery = q.Encode()

	conn, err := db.Connect(ctx, opts(u.String()))
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { conn.Close() })

	files, err := filepath.Glob("../sql/[0-9][0-9][0-9]_*.sql") // Glob returns them sorted by name
	if err != nil || len(files) == 0 {
		t.Fatalf("no schema files found (%v)", err)
	}
	for _, f := range files {
		ddl, err := os.ReadFile(f)
		if err != nil {
			t.Fatal(err)
		}
		if _, err := conn.ExecContext(ctx, string(ddl)); err != nil {
			t.Fatalf("applying %s: %v", f, err)
		}
	}
	return &testDB{DB: conn, dsn: u.String()}
}

// reconnect opens a second, independent pool on the same schema.
func (d *testDB) reconnect(t *testing.T) *sqlx.DB {
	t.Helper()
	conn, err := db.Connect(context.Background(), opts(d.dsn))
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { conn.Close() })
	return conn
}
```

```go
// file: postgres/product_store_test.go
package postgres

import (
	"context"
	"ecommerce/models"
	"ecommerce/product"
	"ecommerce/product/storetest"
	"errors"
	"testing"
)

var _ product.Store = (*ProductStore)(nil)

func TestProductStoreContract(t *testing.T) {
	storetest.Run(t, func(t *testing.T) product.Store { return NewProductStore(newTestDB(t).DB) })
}

func TestPriceIsStoredWithTwoDecimals(t *testing.T) {
	s := NewProductStore(newTestDB(t).DB)
	got, err := s.Create(context.Background(), models.Product{Title: "Rounded", Price: 19.999})
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
	for name, p := range map[string]models.Product{
		"zero price":     {Title: "Free", Price: 0},
		"negative price": {Title: "Neg", Price: -5},
		"blank title":    {Title: "   ", Price: 1},
		"title too long": {Title: string(make([]byte, 101)), Price: 1},
	} {
		if _, err := s.Create(ctx, p); err == nil || errors.Is(err, models.ErrNotFound) {
			t.Errorf("%s: the database should have rejected the row, got %v", name, err)
		}
	}
}

func TestUpdateRefreshesUpdatedAt(t *testing.T) {
	d := newTestDB(t)
	s := NewProductStore(d.DB)
	ctx := context.Background()

	created, _ := s.Create(ctx, models.Product{Title: "Old", Price: 1})
	var before, after struct{ Created, Updated string }
	d.Get(&before, "SELECT created_at::text AS created, updated_at::text AS updated FROM products WHERE id = $1", created.ID)

	if _, err := d.Exec("SELECT pg_sleep(0.05)"); err != nil {
		t.Fatal(err)
	}
	s.Update(ctx, created.ID, models.Product{Title: "New", Price: 2})
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

	if _, err := s.Get(ctx, 1); err == nil || errors.Is(err, models.ErrNotFound) {
		t.Errorf("a cancelled context must give a real error, got %v", err)
	}
	if err := s.Delete(ctx, 1); err == nil || errors.Is(err, models.ErrNotFound) {
		t.Errorf("a cancelled context must give a real error, got %v", err)
	}
}
```

(`d.Get(&before, …)` scans the two text columns into a small anonymous struct: sqlx matches `created` and `updated` to the exported fields `Created`, `Updated` case-insensitively. One more place where struct scanning saves code.)

### Handler tests for the new endpoints

The `failingStore` fake from Chapter 51 must satisfy the bigger interface: methods can be added from any `_test.go` file of the same package, so `handler_test.go` stays untouched:

```go
// file: product/write_test.go
package product

import (
	"context"
	"ecommerce/models"
	"errors"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func (f *failingStore) Update(context.Context, int, models.Product) (models.Product, error) {
	return models.Product{}, f.err
}
func (f *failingStore) Delete(context.Context, int) error { return f.err }

func TestUpdateAndDeleteStoreFailuresAreGeneric500s(t *testing.T) {
	boom := errors.New("connection to database lost: password=hunter2")

	for name, call := range map[string]func(h *Handler, w http.ResponseWriter){
		"update": func(h *Handler, w http.ResponseWriter) {
			r := httptest.NewRequest(http.MethodPut, "/products/1", strings.NewReader(`{"title":"x","price":1}`))
			r.SetPathValue("id", "1")
			h.Update(w, r)
		},
		"delete": func(h *Handler, w http.ResponseWriter) {
			r := httptest.NewRequest(http.MethodDelete, "/products/1", nil)
			r.SetPathValue("id", "1")
			h.Delete(w, r)
		},
	} {
		t.Run(name, func(t *testing.T) {
			logger, logs := newLogger()
			rec := httptest.NewRecorder()
			call(NewHandler(&failingStore{err: boom}, logger), rec)

			if rec.Code != http.StatusInternalServerError || strings.Contains(rec.Body.String(), "hunter2") {
				t.Errorf("got %d %s", rec.Code, rec.Body.String())
			}
			if !strings.Contains(logs.String(), "connection to database lost") {
				t.Errorf("the cause must be logged: %q", logs.String())
			}
		})
	}
}

func TestUpdateAndDeleteMapNotFoundTo404(t *testing.T) {
	logger, logs := newLogger()
	h := NewHandler(&failingStore{err: models.ErrNotFound}, logger)

	rec := httptest.NewRecorder()
	r := httptest.NewRequest(http.MethodPut, "/products/7", strings.NewReader(`{"title":"x","price":1}`))
	r.SetPathValue("id", "7")
	h.Update(rec, r)
	if rec.Code != http.StatusNotFound {
		t.Errorf("update: expected 404, got %d", rec.Code)
	}

	rec = httptest.NewRecorder()
	r = httptest.NewRequest(http.MethodDelete, "/products/7", nil)
	r.SetPathValue("id", "7")
	h.Delete(rec, r)
	if rec.Code != http.StatusNotFound {
		t.Errorf("delete: expected 404, got %d", rec.Code)
	}
	if logs.Len() != 0 {
		t.Errorf("a missing record is normal and must not be logged as an error: %q", logs.String())
	}
}

func TestUpdateValidatesBeforeTouchingTheStore(t *testing.T) {
	logger, _ := newLogger()
	// If validation failed to stop the request, this store would answer 500 instead of 4xx.
	h := NewHandler(&failingStore{err: errors.New("must not be reached")}, logger)

	for name, tc := range map[string]struct {
		id, body string
		want     int
	}{
		"bad id":         {"abc", `{"title":"x","price":1}`, http.StatusBadRequest},
		"zero id":        {"0", `{"title":"x","price":1}`, http.StatusBadRequest},
		"empty body":     {"1", ``, http.StatusBadRequest},
		"unknown field":  {"1", `{"title":"x","price":1,"id":5}`, http.StatusBadRequest},
		"missing title":  {"1", `{"price":1}`, http.StatusUnprocessableEntity},
		"negative price": {"1", `{"title":"x","price":-1}`, http.StatusUnprocessableEntity},
	} {
		t.Run(name, func(t *testing.T) {
			rec := httptest.NewRecorder()
			r := httptest.NewRequest(http.MethodPut, "/products/"+tc.id, strings.NewReader(tc.body))
			r.SetPathValue("id", tc.id)
			h.Update(rec, r)
			if rec.Code != tc.want {
				t.Errorf("expected %d, got %d (%s)", tc.want, rec.Code, rec.Body.String())
			}
		})
	}
}
```

### End-to-end tests through the whole HTTP stack

Using the `rest` test app from Chapter 50 (with the in-memory store behind it): routing, authentication, JSON, status codes, and `204`:

```go
// file: rest/product_write_test.go
package rest

import (
	"encoding/json"
	"net/http"
	"testing"
)

func TestUpdateAndDeleteNeedAToken(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	body := `{"title":"Blood Orange","description":"d","price":120,"imageUrl":"u"}`

	if rec := a.do(http.MethodPut, "/products/1", body, nil); rec.Code != http.StatusUnauthorized {
		t.Errorf("PUT without a token: expected 401, got %d", rec.Code)
	}
	if rec := a.do(http.MethodDelete, "/products/1", "", nil); rec.Code != http.StatusUnauthorized {
		t.Errorf("DELETE without a token: expected 401, got %d", rec.Code)
	}
	if got := a.count(); got != 3 {
		t.Errorf("unauthenticated requests changed the data: %d products", got)
	}
}

func TestPutReplacesAProduct(t *testing.T) {
	t.Parallel()
	a := newApp(t)

	rec := a.do(http.MethodPut, "/products/1",
		`{"title":"Blood Orange","description":"deep red","price":120,"imageUrl":"https://example.com/b.jpg"}`, a.bearer(1))
	if rec.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d: %s", rec.Code, rec.Body.String())
	}

	var got struct {
		ID    int     `json:"id"`
		Title string  `json:"title"`
		Price float64 `json:"price"`
	}
	json.NewDecoder(a.do(http.MethodGet, "/products/1", "", nil).Body).Decode(&got)
	if got.ID != 1 || got.Title != "Blood Orange" || got.Price != 120 {
		t.Errorf("GET after PUT returned %+v", got)
	}
}

func TestPutToAMissingProductIs404(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	rec := a.do(http.MethodPut, "/products/99", `{"title":"x","price":1}`, a.bearer(1))
	if rec.Code != http.StatusNotFound {
		t.Errorf("expected 404, got %d", rec.Code)
	}
	if a.count() != 3 {
		t.Error("PUT must not create products (that is POST's job)")
	}
}

func TestDeleteRemovesAProduct(t *testing.T) {
	t.Parallel()
	a := newApp(t)

	rec := a.do(http.MethodDelete, "/products/2", "", a.bearer(1))
	if rec.Code != http.StatusNoContent || rec.Body.Len() != 0 {
		t.Fatalf("expected 204 with an empty body, got %d %q", rec.Code, rec.Body.String())
	}
	if got := a.do(http.MethodGet, "/products/2", "", nil).Code; got != http.StatusNotFound {
		t.Errorf("deleted product still readable: %d", got)
	}
	if a.count() != 2 {
		t.Errorf("expected 2 products left, got %d", a.count())
	}
	if got := a.do(http.MethodDelete, "/products/2", "", a.bearer(1)).Code; got != http.StatusNotFound {
		t.Errorf("deleting again: expected 404, got %d", got)
	}
}

func TestDeleteRejectsBadIDs(t *testing.T) {
	t.Parallel()
	a := newApp(t)
	for _, id := range []string{"abc", "0", "-3", "1.5"} {
		if got := a.do(http.MethodDelete, "/products/"+id, "", a.bearer(1)).Code; got != http.StatusBadRequest {
			t.Errorf("DELETE /products/%s: expected 400, got %d", id, got)
		}
	}
}
```

Run everything (the last command exercises PostgreSQL too):

```bash
go vet ./... && go test -race ./...
TEST_DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable' go test -race -count=1 ./...
```

---

## 13. Trying it with `curl`

Start the server against the database from Chapter 52 (schema and seed applied as in §3), register a user, and log in to get a token:

```bash
TOKEN=$(curl -s -X POST localhost:18080/login -d '{"email":"asha@example.com","password":"correct-horse-battery"}' | jq -r .token)
```

(`jq -r .token` extracts the field; without `jq`, copy the token by hand.) Then the full CRUD cycle. Output is trimmed to status and body:

```
$ curl -s localhost:18080/products
HTTP/1.1 200 OK
[{"id":1,"title":"Orange","description":"Orange is juicy and full of vitamin C.","price":100,"imageUrl":"https://example.com/orange.jpg"}, … Apple … Banana]

$ curl -X POST localhost:18080/products -H "Authorization: Bearer $TOKEN" -d '{"title":"Mango","description":"Sweet","price":75.5,"imageUrl":"https://example.com/mango.jpg"}'
HTTP/1.1 201 Created
Location: /products/4
{"id":4,"title":"Mango","description":"Sweet","price":75.5,"imageUrl":"https://example.com/mango.jpg"}

$ curl -X PUT localhost:18080/products/4 -H "Authorization: Bearer $TOKEN" -d '{"title":"Alphonso Mango","description":"Sweeter","price":120,"imageUrl":"https://example.com/mango.jpg"}'
HTTP/1.1 200 OK
{"id":4,"title":"Alphonso Mango","description":"Sweeter","price":120,"imageUrl":"https://example.com/mango.jpg"}

$ curl localhost:18080/products/4                         # the change is visible to everyone
HTTP/1.1 200 OK
{"id":4,"title":"Alphonso Mango",...}

$ curl -X PUT localhost:18080/products/99 -H "Authorization: Bearer $TOKEN" -d '{"title":"x","price":1}'
HTTP/1.1 404 Not Found
{"error":"product not found"}

$ curl -X PUT localhost:18080/products/4 -H "Authorization: Bearer $TOKEN" -d '{"title":"x","price":-1}'
HTTP/1.1 422 Unprocessable Entity
{"error":"price must be greater than 0"}

$ curl -X PUT localhost:18080/products/4 -d '{"title":"x","price":1}'          # no token
HTTP/1.1 401 Unauthorized
{"error":"missing or malformed Authorization header (use: Bearer YOUR_TOKEN)"}

$ curl -X DELETE localhost:18080/products/4 -H "Authorization: Bearer $TOKEN"
HTTP/1.1 204 No Content

$ curl -X DELETE localhost:18080/products/4 -H "Authorization: Bearer $TOKEN"   # again
HTTP/1.1 404 Not Found
{"error":"product not found"}
```

Things to notice: the `DELETE` succeeded with **no body**; the second `DELETE` of the same product returns **404** (the resource is gone, as `DELETE` is idempotent in *effect* but honest about it); and after restarting the server the data is **still there** (product 1 was served again), because they live in PostgreSQL. Peek at the table (Mango is gone; `never_updated` is `created_at = updated_at`, true for rows nobody has changed):

```
 id | title  | price  | never_updated
----+--------+--------+---------------
  1 | Orange | 100.00 | t
  2 | Apple  |  40.00 | t
  3 | Banana |   5.00 | t
(3 rows)
```

**A note on testing tools.** `curl` is great for scripts and documentation. For exploring an API by hand, graphical clients (Postman, Insomnia, Bruno, the VS Code "REST Client" extension) let you save requests and manage tokens as variables. The requests are the same: method, URL, `Authorization: Bearer …`, JSON body.

---

## 14. Why the repository pattern paid off

Count what changed between Chapter 55 (in-memory) and now (PostgreSQL) in the parts that **don't** know about storage:

| Component | Changed for PostgreSQL? |
|-----------|------------------------|
| Handlers (`List/Get/Create`) | **no** (only *new* handlers for the new features) |
| Middleware, routing, JSON helpers, auth | no |
| `rest` end-to-end tests | no (they still run on the fast in-memory store) |
| Composition root | 2 constructor calls |
| **New code** | one `postgres` package (~100 lines) + SQL files |

And what you gained: the contract test guarantees that the fast fake and the real database *agree*, so the fast suite is trustworthy; failure paths are testable with a 5-line fake; and if you ever migrate to another database or add a cache layer, the pattern repeats (a `Decorator` implementing `product.Store` that checks Redis first: Chapter 51's decorator pattern).

---

## 15. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Forgetting `defer rows.Close()` in manual iteration | Leaked connections, eventually a frozen app | Always defer it right after the error check |
| 2 | Not checking `rows.Err()` | Silently truncated results | `return rows.Err()` after the loop |
| 3 | Returning a nil slice from `List` | JSON `null` breaks clients | `make([]T, 0)` |
| 4 | Treating `UPDATE` matching 0 rows as success | `PUT` on a missing ID answers `200` | `RETURNING` + `ErrNoRows`, or `RowsAffected()` |
| 5 | Letting the body change the ID on `PUT` | Identity tampering | The ID comes from the URL; the body has none (and unknown fields are rejected) |
| 6 | Read-modify-write in Go for updates | Lost updates under concurrency | One `UPDATE … SET` statement |
| 7 | `PUT` with partial bodies | Missing fields silently blanked | `PUT` = full replacement (validate required fields); use `PATCH` for partial |
| 8 | Returning `200` + body for `DELETE` inconsistently | Confusing clients | `204 No Content` |
| 9 | Not updating `updated_at` | Misleading audit data | Set it in the `UPDATE` (or a trigger) |
| 10 | Duplicating request decoding/validation in each handler | Drift between `POST` and `PUT` rules | Shared `decode` + `validate` |
| 11 | `SELECT *` in queries | Breaks when columns are added | Explicit column lists |
| 12 | `GetContext` for multi-row queries | Silently uses the first row | `SelectContext` |
| 13 | Testing only the happy path against the real DB | Bugs in not-found/constraint paths | The contract suite + constraint tests |
| 14 | Forgetting to add new SQL files to the test helper | Tests run against an old schema | Glob all numbered files |

---

## 16. Interview questions

**Q1. `Get` vs `Select` vs `Exec` in sqlx?**
`Get`: one row into a struct/scalar (`ErrNoRows` if none). `Select`: all rows into a slice. `Exec`: no rows returned, gives `RowsAffected`.

**Q2. Why `defer rows.Close()`, and why `rows.Err()`?**
An open `Rows` holds a pooled connection: closing releases it. `Next()` returns false on both end-of-data and errors; `Err()` distinguishes them.

**Q3. `PUT` vs `PATCH`?**
`PUT` replaces the whole resource (idempotent); `PATCH` applies a partial change.

**Q4. How do you detect "not found" for an `UPDATE`/`DELETE`?**
Check `RowsAffected() == 0` (for `Exec`) or use `RETURNING` and treat `sql.ErrNoRows` as not found.

**Q5. Why return `[]` and not `null` for an empty list?**
`encoding/json` encodes a nil slice as `null`; clients expect an array. Initialize with `make`.

**Q6. What's a contract test, and why use one here?**
One suite run against every implementation of an interface, ensuring the in-memory fake and PostgreSQL store behave the same.

**Q7. Why is `DELETE` idempotent, yet its second call returns `404`?**
Idempotent refers to the server state (resource absent either way), not the response code.

**Q8. Why should the ID for `PUT` come from the URL, not the body?**
So the URL identifies the resource unambiguously and clients can't move or overwrite a different resource via the body.

**Q9. What is `204 No Content`?**
A success status with no response body.

---

## 17. Exercises

### Exercise 1: `Count`
Add `Count(ctx) (int, error)` to the interface and both stores (`SELECT count(*) FROM products`), with a contract scenario. What does the compiler tell you after you change the interface?

<details><summary>Solution</summary>

```sql
-- postgres/queries/products_count.sql
SELECT count(*) FROM products
```
```go
func (s *ProductStore) Count(ctx context.Context) (int, error) {
	var n int
	if err := s.db.GetContext(ctx, &n, countProductsSQL); err != nil {
		return 0, fmt.Errorf("count products: %w", err)
	}
	return n, nil
}
```
The compiler flags every type used as a `product.Store` that lacks `Count` (the in-memory store, `failingStore`), which is your to-do list.
</details>

### Exercise 2: `GET /products?category=`
Add a `category TEXT NOT NULL DEFAULT 'general'` column and filter the list by an optional query parameter. Which SQL technique keeps it a *single* query with an optional filter, and why must you still use placeholders?

<details><summary>Solution</summary>

`WHERE ($1 = '' OR category = $1)` (or build the `WHERE` conditionally in Go but **always** pass values as placeholders). Never paste `r.URL.Query().Get("category")` into the SQL text: that's an injection (Chapter 53).
</details>

### Exercise 3: Prove the leak
Write a test that calls the manual `forEach` from §8 *without* `defer rows.Close()` in a loop with `MaxOpenConns = 2`; show that the third call hangs (use a context with a timeout). Then fix it.

<details><summary>Solution</summary>

Each un-closed `Rows` keeps its connection; after two calls the pool is exhausted, so the third `QueryxContext` blocks until the context deadline and returns `context deadline exceeded`. With `defer rows.Close()` all calls succeed. `db.Stats().InUse` shows the leaked connections.
</details>

### Exercise 4: `PATCH`
Implement `PATCH /products/{id}` that changes only the fields present in the body (use pointer fields such as `*string`, `*float64` in the request struct to tell "absent" from "zero"). How do you write the SQL so absent fields keep their values?

<details><summary>Solution</summary>

```sql
UPDATE products SET
  title       = COALESCE($2, title),
  description = COALESCE($3, description),
  price       = COALESCE($4, price),
  image_url   = COALESCE($5, image_url),
  updated_at  = now()
WHERE id = $1
RETURNING id, title, description, price, image_url
```
Pass `nil` pointers for absent fields (they become SQL `NULL`, so `COALESCE` keeps the old value). Validate the present fields (`price > 0` if set, etc.).
</details>

### Exercise 5: Named parameters
Rewrite `Create` with `NamedQueryContext`/`NamedExecContext` and `:title`-style placeholders bound from a struct. When is this nicer than positional `$n`?

<details><summary>Solution</summary>

```sql
INSERT INTO products (title, description, price, image_url)
VALUES (:title, :description, :price, :image_url) RETURNING id, title, description, price, image_url
```
Bind with `db.NamedQueryContext(ctx, q, p)` (then `rows.Next()`/`StructScan`; the `db` tags name the parameters). It shines with many columns where positional numbering is error-prone, but for `RETURNING` you must loop like §8, which is more code, so it's a trade-off.
</details>

### Exercise 6: Integer cents
Store `price_cents BIGINT` and make the Go model expose `price` as a decimal number in JSON without ever doing float math on money. Where do you convert?

<details><summary>Solution</summary>

Keep `PriceCents int64` in the model; implement `MarshalJSON`/`UnmarshalJSON` (or a request DTO) that convert at the API boundary by **string/decimal parsing** (`"12.5"` → `1250`), never `float64 × 100`. All arithmetic (discounts, totals) stays in integers; rounding rules are applied explicitly, once.
</details>

### Exercise 7 (challenge): Optimistic locking
Two admins load product 1, then both `PUT` changes. The second silently overwrites the first ("lost update"). Add a `version BIGINT NOT NULL DEFAULT 1` column and make `Update` succeed only if the client's version matches, returning `409 Conflict` otherwise.

<details><summary>Solution</summary>

`UPDATE products SET …, version = version + 1 WHERE id = $1 AND version = $6 RETURNING …`. Zero rows can now mean "not found" *or* "stale version": disambiguate with a follow-up `SELECT` (or `EXISTS`) and map to `404` vs. `409` (`models.ErrConflict`). The client sends the version it read (e.g., in an `If-Match` header or body field).
</details>

---

## 18. Quiz

1. Which sqlx method fits `DELETE FROM products WHERE id = $1`?
2. Why does `List` start with `make([]models.Product, 0)`?
3. How does the store know an `UPDATE` matched nothing?
4. What status does a successful `DELETE` return, and does it have a body?
5. Why must you always check `rows.Err()`?
6. Where does the product ID come from in `PUT /products/{id}`?
7. What does the contract test guarantee?
8. What did adding two methods to `product.Store` force us to change, and how did we find out?

<details><summary>Answers</summary>

1. `ExecContext` (then `RowsAffected`).
2. So an empty result marshals as `[]`, not `null`.
3. `RETURNING` yields no row (`sql.ErrNoRows`), or `RowsAffected()` is 0 for `Exec`.
4. `204 No Content`, no body.
5. `Next()` is false on errors too; only `Err()` reveals a mid-stream failure.
6. The URL path (`r.PathValue("id")`), never the body.
7. Both implementations behave identically on the same scenarios.
8. Every implementation and fake (`ProductStore` in both packages, `failingStore`); the compiler listed them.
</details>

---

## 19. Summary

- The `products` table applies Chapter 54's design (`NUMERIC(12,2)`, `CHECK`s, `TIMESTAMPTZ` audit columns); queries live in embedded `.sql` files with explicit column lists.
- `sqlx`: **`GetContext`** (one row, incl. `INSERT/UPDATE … RETURNING`), **`SelectContext`** (many rows), **`ExecContext`** (no rows; `RowsAffected`), **`QueryxContext`** (cursor: `defer rows.Close()` and `rows.Err()`).
- **Not found** is translated in the store: `sql.ErrNoRows` (via `RETURNING`) or `RowsAffected() == 0` → `models.ErrNotFound`, so handlers answer `404` without knowing about SQL.
- Lists must be **non-nil** to encode as `[]`.
- `PUT` replaces (idempotent, `200`), `DELETE` answers `204`, both authenticated; the ID comes from the URL; decoding and validation are shared with `POST`.
- The **interface change** was guided by the compiler; **contract tests** keep the in-memory fake and PostgreSQL honest; **constraint tests** prove the database itself refuses bad rows.
- Products are now durable: the application's data layer is fully on PostgreSQL.

### ➡️ What's next?

[Chapter 57](57-database-configuration.md) is about **database configuration** as you deploy: building connection strings safely, TLS modes per environment, timeouts, environment-variable precedence (why your `.env` sometimes "doesn't work"), and running the app and database together with Docker Compose.
