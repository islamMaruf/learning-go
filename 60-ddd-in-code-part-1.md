# Chapter 60: DDD in Code, Part 1 — The User Domain

> **Goal of this chapter:** Apply Chapter 59 to real code. You'll pull the *business rules* of user accounts (email format, password policy, duplicate detection, login with timing protection) out of the HTTP handler and into a **domain package** with an **entity**, a **value object** (`Email`), **domain errors**, **ports** (`Repository`, `PasswordHasher`, `TokenIssuer`) and a **`Service`** that orchestrates the use cases. The outside world (HTTP, PostgreSQL, bcrypt, JWT) becomes thin **adapters** that depend *inward* on the domain. Every HTTP behavior stays **exactly** the same, and you'll gain unit tests that run in milliseconds with no server, no database, and no bcrypt.

**Difficulty:** 🔴 Advanced  **Estimated time:** 8 hours  **Prerequisite:** [Chapter 59](59-domain-driven-design.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The refactoring plan](#2-the-refactoring-plan)
3. [Step 1: the entity and its errors](#3-step-1-the-entity-and-its-errors)
4. [Step 2: a value object, `Email`](#4-step-2-a-value-object-email)
5. [Step 3: the ports](#5-step-3-the-ports)
6. [Step 4: the service](#6-step-4-the-service)
7. [Step 5: adapters (bcrypt, JWT, PostgreSQL, memory)](#7-step-5-adapters)
8. [Step 6: the HTTP handler becomes thin](#8-step-6-the-http-handler-becomes-thin)
9. [Step 7: wiring](#9-step-7-wiring)
10. [Tests: fast, and at the right level](#10-tests-fast-and-at-the-right-level)
11. [Proving nothing changed](#11-proving-nothing-changed)
12. [The golden rules, checked](#12-the-golden-rules-checked)
13. [Change-impact analysis](#13-change-impact-analysis)
14. [Common mistakes](#14-common-mistakes)
15. [Interview questions](#15-interview-questions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- How to **refactor safely**: keep behavior fixed, move code in small steps, keep tests green
- Designing a **domain package** with an entity, value object, errors, ports, and service
- **Make illegal states unrepresentable**: an `Email` that can only exist if valid
- **Domain errors** (`ErrEmailTaken`, `ErrInvalidCredentials`, `*ValidationError`) and how outer layers translate them
- **Consumer-defined interfaces** at two levels: domain ports (`Repository`…) and handler-side `Service`
- Separate **wire** (JSON) and **storage** (row) shapes from the domain entity, and the mapping code that costs
- **Constant-time-ish login** kept in the domain, and tested with a counting fake
- Unit testing use cases with **fakes**: hundreds of assertions in a few milliseconds

---

## 2. The refactoring plan

Today (after Chapter 58) the user feature looks like this:

```
user/                      ← handlers AND rules AND a Store interface, all in one package
├── handler.go             Handler{store, jwtSecret, jwtTTL, logger, dummyHash}
├── register.go            validate email/password + hash + store.Create + JSON
├── login.go               find user + bcrypt + timing defense + JWT + JSON
├── me.go                  read ID from context + store.FindByID + JSON
└── store.go               Store interface
models/user.go             User struct with json + db tags, NormalizeEmail, ErrEmailTaken
```

After this chapter:

```
user/                      ← the DOMAIN: knows nothing about HTTP, SQL, bcrypt or JWT
├── user.go                entity User
├── email.go               value object Email
├── errors.go              ErrNotFound, ErrEmailTaken, ErrInvalidCredentials, ValidationError
├── port.go                ports: Repository, PasswordHasher, TokenIssuer
├── service.go             use cases: Register, Login, Get
├── service_test.go        unit tests with fakes
├── email_test.go
└── storetest/             contract suite for any Repository

rest/userhandler/          ← ADAPTER (HTTP): JSON in/out, status codes; calls a Service
auth/adapters.go           ← ADAPTERS: BcryptHasher (PasswordHasher), TokenIssuer
postgres/user_store.go     ← ADAPTER: implements user.Repository (row struct ↔ entity)
database/user_store.go     ← ADAPTER: in-memory implementation
cmd/wire.go                ← COMPOSITION ROOT
```

The refactoring recipe, which Chapter 61 will repeat for products:

1. Create the domain package with the entity and errors (compiles, unused).
2. Add the value object(s).
3. Define the ports.
4. Write the **service** and its unit tests (green, in isolation).
5. Adapt the outside: repositories, hashers, token issuers.
6. Slim the handler to call the service.
7. Rewire `cmd`, delete the old code, run **all** the old end-to-end tests. They must pass unchanged.

---

## 3. Step 1: the entity and its errors

```go
// file: user/user.go
// Package user is the Identity domain: accounts, registration and login.
// It contains business rules only. It must not import net/http, database/sql, bcrypt or JWT code.
package user

import "time"

// User is a registered account (an entity: identified by ID).
type User struct {
	ID           int
	Email        Email
	PasswordHash string
	CreatedAt    time.Time
}
```

Notice what is *missing* compared with the old `models.User`: **no `json:` tags and no `db:` tags**. Those describe how the user is *represented* on the wire and in the table: outer-layer concerns. The domain's `User` is just what a user *is*. (`PasswordHash` stays here: the domain needs it to log people in, but the JSON adapter will simply never expose it. This replaces the `json:"-"` trick with a stronger guarantee: the response type *doesn't have the field at all*.)

The domain speaks in its **own errors**, so callers depend on the domain's vocabulary rather than on `models` or on `sql`:

```go
// file: user/errors.go
package user

import "errors"

var (
	// ErrNotFound means no user matches the lookup.
	ErrNotFound = errors.New("user not found")
	// ErrEmailTaken means the email address is already registered.
	ErrEmailTaken = errors.New("email already registered")
	// ErrInvalidCredentials means the email/password pair is wrong. It deliberately does not say
	// which of the two was wrong, so attackers can't discover which emails exist.
	ErrInvalidCredentials = errors.New("invalid email or password")
)

// ValidationError means the caller's input broke a business rule. Message is written for end users.
type ValidationError struct{ Message string }

func (e *ValidationError) Error() string { return e.Message }

func invalid(message string) error { return &ValidationError{Message: message} }
```

Two kinds of domain errors, mirroring Chapter 41: **sentinels** for fixed conditions (`errors.Is`), and a **typed error** for conditions that carry data (`errors.As`). An HTTP adapter maps them: `*ValidationError` → 422, `ErrEmailTaken` → 409, `ErrInvalidCredentials` → 401. The domain itself has no idea what a status code is.

---

## 4. Step 2: a value object, `Email`

```go
// file: user/email.go
package user

import (
	"net/mail"
	"strings"
)

// Email is a valid, normalized (trimmed, lower-case) email address.
// It is a value object: immutable, compared by value, and the zero Email is "no email".
// The only way to obtain one is NewEmail, so any Email in the program is known to be valid.
type Email struct{ value string }

// NewEmail validates and normalizes s.
func NewEmail(s string) (Email, error) {
	s = strings.ToLower(strings.TrimSpace(s))
	addr, err := mail.ParseAddress(s)
	if err != nil || addr.Address != s { // reject "Name <a@b.co>" forms: we want a bare address
		return Email{}, invalid("email must be a valid address like name@example.com")
	}
	return Email{value: s}, nil
}

// String returns the address.
func (e Email) String() string { return e.value }

// IsZero reports whether e is the zero value (no address).
func (e Email) IsZero() bool { return e.value == "" }
```

Why bother? Before, the string `"asha@example.com"` was passed through five functions, and *each* had to remember whether it had been checked or lower-cased. Now `Repository.Create(ctx, email Email, …)` **cannot be called with an unvalidated address**: the compiler won't allow it. The Chapter 53 helper `models.NormalizeEmail` disappears because normalization happens once, at construction.

Since the struct has one comparable field, `==` compares emails by value (which is exactly what value-object equality means), so the in-memory repository can use `u.Email == email`.

---

## 5. Step 3: the ports

The domain declares **what it needs from the outside world** as interfaces, next to the code that uses them:

```go
// file: user/port.go
package user

import "context"

// Repository stores users. It is a port: implemented by adapters such as postgres.UserStore.
type Repository interface {
	// Create stores a new user. It returns ErrEmailTaken if the email is already registered.
	Create(ctx context.Context, email Email, passwordHash string) (User, error)
	// FindByEmail and FindByID return ErrNotFound if there is no such user.
	FindByEmail(ctx context.Context, email Email) (User, error)
	FindByID(ctx context.Context, id int) (User, error)
}

// PasswordHasher turns passwords into storable hashes and checks them. (Implemented with bcrypt in package auth.)
type PasswordHasher interface {
	Hash(plain string) (string, error)
	Matches(hash, plain string) bool
}

// TokenIssuer creates the credential a logged-in user presents on later requests. (JWT in package auth.)
type TokenIssuer interface {
	Issue(userID int) (string, error)
}
```

Three ports, and each represents a genuine *decision the business shouldn't care about*: where users are stored, how passwords are hashed, what the login token looks like. Everything else (`fmt`, `errors`, `time`) is plain language machinery. Contrast with the earlier design where `user.Handler` held `jwtSecret []byte` and `jwtTTL`: the domain had to know about token *configuration*.

---

## 6. Step 4: the service

The **service** implements the use cases. It reads like the business description: *"To register: check the email and password rules, hash the password, store the user (which fails if the email is taken)."*

```go
// file: user/service.go
package user

import (
	"context"
	"errors"
	"fmt"
)

// Password policy.
const (
	minPasswordLength = 8
	maxPasswordLength = 72 // bcrypt only uses the first 72 bytes
)

// Service implements the user use cases on top of the ports.
type Service struct {
	repo   Repository
	hasher PasswordHasher
	tokens TokenIssuer

	// dummyHash lets Login spend the same time on unknown emails as on known ones.
	dummyHash string
}

// NewService creates a Service.
func NewService(repo Repository, hasher PasswordHasher, tokens TokenIssuer) (*Service, error) {
	dummy, err := hasher.Hash("dummy-password-for-timing")
	if err != nil {
		return nil, fmt.Errorf("prepare timing-safe login: %w", err)
	}
	return &Service{repo: repo, hasher: hasher, tokens: tokens, dummyHash: dummy}, nil
}

// validatePassword applies the password policy.
func validatePassword(password string) error {
	switch {
	case len(password) < minPasswordLength:
		return invalid("password must be at least 8 characters")
	case len(password) > maxPasswordLength:
		return invalid("password must be at most 72 bytes")
	}
	return nil
}

// Register creates an account. Errors: *ValidationError, ErrEmailTaken, or an unexpected failure.
func (s *Service) Register(ctx context.Context, rawEmail, password string) (User, error) {
	email, err := NewEmail(rawEmail)
	if err != nil {
		return User{}, err
	}
	if err := validatePassword(password); err != nil {
		return User{}, err
	}

	hash, err := s.hasher.Hash(password)
	if err != nil {
		return User{}, fmt.Errorf("hash password: %w", err)
	}

	u, err := s.repo.Create(ctx, email, hash)
	if err != nil {
		if errors.Is(err, ErrEmailTaken) {
			return User{}, err // an expected business outcome: pass it on as is
		}
		return User{}, fmt.Errorf("create user: %w", err)
	}
	return u, nil
}

// Login checks the credentials and returns a token. The only credential failure it ever reports is
// ErrInvalidCredentials, and it does the same expensive work whether or not the email exists.
func (s *Service) Login(ctx context.Context, rawEmail, password string) (string, error) {
	hash := s.dummyHash // compared against when the account doesn't exist, to equalize timing
	var found User
	exists := false

	if email, err := NewEmail(rawEmail); err == nil { // a malformed email simply can't exist
		u, err := s.repo.FindByEmail(ctx, email)
		switch {
		case err == nil:
			found, exists, hash = u, true, u.PasswordHash
		case !errors.Is(err, ErrNotFound):
			return "", fmt.Errorf("find user: %w", err) // a genuine storage failure, not "wrong password"
		}
	}

	passwordOK := s.hasher.Matches(hash, password) // ALWAYS runs
	if !exists || !passwordOK {
		return "", ErrInvalidCredentials
	}

	token, err := s.tokens.Issue(found.ID)
	if err != nil {
		return "", fmt.Errorf("issue token: %w", err)
	}
	return token, nil
}

// Get returns the user with the given ID, or ErrNotFound.
func (s *Service) Get(ctx context.Context, id int) (User, error) {
	u, err := s.repo.FindByID(ctx, id)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			return User{}, err
		}
		return User{}, fmt.Errorf("find user: %w", err)
	}
	return u, nil
}
```

What to notice:

- **The rules are now readable in one place.** The email rule is in `NewEmail`; the password policy in `validatePassword`; registration order in `Register`. No JSON, no `http.Error`.
- **Expected outcomes vs. failures.** `ErrEmailTaken` and `ErrNotFound` are *business outcomes*: returned unwrapped so callers can `errors.Is` them (wrapping with `%w` would work too). Anything else is wrapped with context (`"create user: …"`): an *unexpected failure*, which the HTTP layer will log and turn into a 500.
- **Timing protection is domain logic** now, and it's *testable*: Login always calls `hasher.Matches`, even for a malformed email, even for an unknown one. We'll assert that with a counting fake.
- `Service` holds **ports**, not concrete types: swap bcrypt for argon2 by writing a new `PasswordHasher`; nothing here changes.
- The service has **no logger**. It returns errors; the adapter that *handles* the request decides what to log (one place logs each failure, once).

---

## 7. Step 5: adapters

An **adapter** translates between the domain's ports and a concrete technology. Dependencies point **inward**: each adapter imports `user`, never the other way around.

### Password hashing and tokens (package `auth`)

`auth` already has the bcrypt and JWT functions. Two small types make them satisfy the ports (implicitly: `auth` doesn't import `user`):

```go
// file: auth/adapters.go
package auth

import "time"

// BcryptHasher implements user.PasswordHasher with bcrypt.
type BcryptHasher struct{}

// Hash returns a salted bcrypt hash of plain.
func (BcryptHasher) Hash(plain string) (string, error) { return HashPassword(plain) }

// Matches reports whether plain matches the stored bcrypt hash.
func (BcryptHasher) Matches(hash, plain string) bool { return CheckPassword(hash, plain) }

// TokenIssuer implements user.TokenIssuer with signed JWTs.
type TokenIssuer struct {
	Secret []byte
	TTL    time.Duration
	Now    func() time.Time // optional: tests can control the clock; nil means time.Now
}

// Issue creates a signed token for the user, valid for TTL.
func (t TokenIssuer) Issue(userID int) (string, error) {
	now := time.Now
	if t.Now != nil {
		now = t.Now
	}
	return CreateToken(t.Secret, userID, t.TTL, now())
}
```

The token *configuration* (secret, TTL) now belongs to this adapter, and the domain never sees it.

### PostgreSQL: a row shape of its own

The database row and the domain entity are different things that happen to look alike today. The adapter owns the row shape (with `db` tags) and converts:

```go
// file: postgres/user_store.go
package postgres

import (
	"context"
	"database/sql"
	"ecommerce/user"
	_ "embed" // needed for //go:embed
	"errors"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5/pgconn"
	"github.com/jmoiron/sqlx"
)

//go:embed queries/users_insert.sql
var insertUserSQL string

//go:embed queries/users_by_email.sql
var userByEmailSQL string

//go:embed queries/users_by_id.sql
var userByIDSQL string

// userRow is how a user looks in the users table (the columns the queries return).
type userRow struct {
	ID           int       `db:"id"`
	Email        string    `db:"email"`
	PasswordHash string    `db:"password_hash"`
	CreatedAt    time.Time `db:"created_at"`
}

// toUser converts a row to the domain entity. Stored emails were validated on the way in, so a
// failure here means the table was edited by hand: report it rather than pretend.
func (r userRow) toUser() (user.User, error) {
	email, err := user.NewEmail(r.Email)
	if err != nil {
		return user.User{}, fmt.Errorf("users row %d has an invalid email: %w", r.ID, err)
	}
	return user.User{ID: r.ID, Email: email, PasswordHash: r.PasswordHash, CreatedAt: r.CreatedAt.UTC()}, nil
}

// UserStore keeps users in PostgreSQL. It is safe for concurrent use: *sqlx.DB is a connection pool.
// It implements user.Repository.
type UserStore struct {
	db *sqlx.DB
}

// NewUserStore creates a store on top of an existing connection pool (see infra/db).
func NewUserStore(db *sqlx.DB) *UserStore { return &UserStore{db: db} }

// Create stores a new user, or returns user.ErrEmailTaken if the email is already registered.
func (s *UserStore) Create(ctx context.Context, email user.Email, passwordHash string) (user.User, error) {
	var row userRow
	if err := s.db.GetContext(ctx, &row, insertUserSQL, email.String(), passwordHash); err != nil {
		if isUniqueViolation(err) {
			return user.User{}, user.ErrEmailTaken
		}
		return user.User{}, fmt.Errorf("insert user: %w", err)
	}
	return row.toUser()
}

// FindByEmail looks a user up by email, or returns user.ErrNotFound.
func (s *UserStore) FindByEmail(ctx context.Context, email user.Email) (user.User, error) {
	return s.find(ctx, "find user by email", userByEmailSQL, email.String())
}

// FindByID looks a user up by ID, or returns user.ErrNotFound.
func (s *UserStore) FindByID(ctx context.Context, id int) (user.User, error) {
	return s.find(ctx, "find user by id", userByIDSQL, id)
}

// find runs a single-row user query and translates "no rows" into user.ErrNotFound.
func (s *UserStore) find(ctx context.Context, what, query string, arg any) (user.User, error) {
	var row userRow
	if err := s.db.GetContext(ctx, &row, query, arg); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return user.User{}, user.ErrNotFound
		}
		return user.User{}, fmt.Errorf("%s: %w", what, err)
	}
	return row.toUser()
}

// isUniqueViolation reports whether err is PostgreSQL's unique_violation (SQLSTATE 23505).
func isUniqueViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23505"
}
```

That `userRow` → `user.User` conversion is **the cost of DDD** from Chapter 59 in concrete form: ~10 extra lines per entity. What it buys: the table can gain columns, or the domain entity can gain fields, independently.

### In memory (for tests and demos)

```go
// file: database/user_store.go
package database

import (
	"context"
	"ecommerce/user"
	"sync"
	"time"
)

// UserStore keeps users in memory. It is safe for concurrent use and implements user.Repository.
// Production uses the PostgreSQL store; this one backs fast tests and demos.
type UserStore struct {
	mu     sync.RWMutex
	users  []user.User
	nextID int
}

// NewUserStore creates an empty user store.
func NewUserStore() *UserStore { return &UserStore{nextID: 1} }

// Create stores a new user, or returns user.ErrEmailTaken. The duplicate check and the insert
// share one lock, so two simultaneous registrations of the same email cannot both succeed.
func (s *UserStore) Create(_ context.Context, email user.Email, passwordHash string) (user.User, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	for _, u := range s.users {
		if u.Email == email { // Email is a value object: == compares by value
			return user.User{}, user.ErrEmailTaken
		}
	}
	u := user.User{ID: s.nextID, Email: email, PasswordHash: passwordHash, CreatedAt: time.Now().UTC()}
	s.nextID++
	s.users = append(s.users, u)
	return u, nil
}

// FindByEmail looks a user up by email, or returns user.ErrNotFound.
func (s *UserStore) FindByEmail(_ context.Context, email user.Email) (user.User, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, u := range s.users {
		if u.Email == email {
			return u, nil
		}
	}
	return user.User{}, user.ErrNotFound
}

// FindByID looks a user up by ID, or returns user.ErrNotFound.
func (s *UserStore) FindByID(_ context.Context, id int) (user.User, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, u := range s.users {
		if u.ID == id {
			return u, nil
		}
	}
	return user.User{}, user.ErrNotFound
}
```

### The shared error and model packages shrink

`models.User`, `models.NormalizeEmail` and `models.ErrEmailTaken` moved into the domain. `models` now only holds the product pieces (until Chapter 61):

```go
// file: models/errors.go
package models

import "errors"

// ErrNotFound is returned by product stores when the requested record does not exist.
var ErrNotFound = errors.New("not found")
```

<!-- delete: models/user.go -->

---

## 8. Step 6: the HTTP handler becomes thin

The handler now does only **delivery** work: decode JSON, call the service, translate the result. It defines the (tiny) interface it needs: the *consumer-defined* interface from Chapter 51, now one level up. `*user.Service` satisfies it without knowing it exists.

```go
// file: rest/userhandler/handler.go
// Package userhandler is the HTTP adapter for the user domain: JSON in, JSON out, status codes.
// It contains no business rules; those live in package user.
package userhandler

import (
	"context"
	"ecommerce/auth"
	"ecommerce/user"
	"ecommerce/util"
	"errors"
	"log/slog"
	"net/http"
	"time"
)

// Service is what this handler needs from the user domain. *user.Service satisfies it.
type Service interface {
	Register(ctx context.Context, email, password string) (user.User, error)
	Login(ctx context.Context, email, password string) (token string, err error)
	Get(ctx context.Context, id int) (user.User, error)
}

// Handler serves registration, login, and the current-user endpoint.
type Handler struct {
	svc    Service
	logger *slog.Logger
}

// New creates a Handler.
func New(svc Service, logger *slog.Logger) *Handler { return &Handler{svc: svc, logger: logger} }

// Routes registers this feature's endpoints on mux.
func (h *Handler) Routes(mux *http.ServeMux, authn func(http.Handler) http.Handler) {
	mux.HandleFunc("POST /users", h.Register)
	mux.HandleFunc("POST /login", h.Login)
	mux.Handle("GET /me", authn(http.HandlerFunc(h.Me)))
}

// credentials is the JSON body of registration and login.
type credentials struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

// userResponse is the JSON representation of a user. It has no password-hash field at all,
// so no bug can leak one.
type userResponse struct {
	ID        int       `json:"id"`
	Email     string    `json:"email"`
	CreatedAt time.Time `json:"createdAt"`
}

func toResponse(u user.User) userResponse {
	return userResponse{ID: u.ID, Email: u.Email.String(), CreatedAt: u.CreatedAt}
}

// Register handles POST /users.
func (h *Handler) Register(w http.ResponseWriter, r *http.Request) {
	var req credentials
	if !util.DecodeJSON(w, r, &req) {
		return
	}

	u, err := h.svc.Register(r.Context(), req.Email, req.Password)
	if err != nil {
		h.fail(w, r, err)
		return
	}
	util.SendData(w, http.StatusCreated, toResponse(u))
}

// Login handles POST /login: it returns a signed token for valid credentials.
func (h *Handler) Login(w http.ResponseWriter, r *http.Request) {
	var req credentials
	if !util.DecodeJSON(w, r, &req) {
		return
	}

	token, err := h.svc.Login(r.Context(), req.Email, req.Password)
	if err != nil {
		h.fail(w, r, err)
		return
	}
	util.SendData(w, http.StatusOK, map[string]string{"token": token})
}

// Me handles GET /me. The route is wrapped in the Authenticate middleware (see Routes),
// which is what puts the caller's ID into the request context.
func (h *Handler) Me(w http.ResponseWriter, r *http.Request) {
	id, ok := auth.UserIDFrom(r.Context())
	if !ok { // only possible if the route was registered without authentication: a programming error
		util.SendError(w, http.StatusUnauthorized, "authentication required")
		return
	}

	u, err := h.svc.Get(r.Context(), id)
	if errors.Is(err, user.ErrNotFound) { // a valid token for an account that no longer exists
		util.SendError(w, http.StatusUnauthorized, "account not found")
		return
	}
	if err != nil {
		h.fail(w, r, err)
		return
	}
	util.SendData(w, http.StatusOK, toResponse(u))
}

// fail translates a domain error into an HTTP response. This is the ONLY place that knows how.
func (h *Handler) fail(w http.ResponseWriter, r *http.Request, err error) {
	var invalid *user.ValidationError
	switch {
	case errors.As(err, &invalid):
		util.SendError(w, http.StatusUnprocessableEntity, invalid.Message)
	case errors.Is(err, user.ErrEmailTaken):
		util.SendError(w, http.StatusConflict, err.Error())
	case errors.Is(err, user.ErrInvalidCredentials):
		util.SendError(w, http.StatusUnauthorized, err.Error())
	default:
		util.ServerError(w, r, h.logger, err) // unexpected: log the cause, hide it from the client
	}
}
```

Compare with the old `Register` (about 40 lines mixing validation, hashing, storage): this one is **decode → call → translate**. The mapping from domain error to status code is in one function, `fail`, so adding a new domain error means adding one `case`. (The `err.Error()` messages, "email already registered" and "invalid email or password", are exactly the texts the API returned before.)

`rest/server.go` follows the new handler type:

```go
// file: rest/server.go
package rest

import (
	"ecommerce/config"
	"ecommerce/health"
	"ecommerce/middleware"
	"ecommerce/product"
	"ecommerce/rest/userhandler"
	"log/slog"
	"net/http"
)

// Server assembles the HTTP application from its already-constructed parts.
type Server struct {
	cfg      *config.Config
	logger   *slog.Logger
	products *product.Handler
	users    *userhandler.Handler
	health   *health.Handler
}

// NewServer connects the feature handlers to the shared configuration and logger.
func NewServer(cfg *config.Config, logger *slog.Logger, products *product.Handler, users *userhandler.Handler, health *health.Handler) *Server {
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

The old handler files are deleted; the old `Store` interface (now `user.Repository`) and old handler test go with them:

<!-- delete: user/handler.go -->
<!-- delete: user/register.go -->
<!-- delete: user/login.go -->
<!-- delete: user/me.go -->
<!-- delete: user/store.go -->
<!-- delete: user/handler_test.go -->

---

## 9. Step 7: wiring

Only the composition root knows every concrete type. Building the service can now *fail* (hashing the timing-dummy), so `buildHandler` returns an error:

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
	hasher := auth.BcryptHasher{}
	tokens := auth.TokenIssuer{Secret: []byte(cfg.JWTSecret), TTL: cfg.JWTTTL}
	productStore := postgres.NewProductStore(conn)

	// domain services: business rules, built from ports
	userService, err := user.NewService(userRepo, hasher, tokens)
	if err != nil {
		return nil, fmt.Errorf("build user service: %w", err)
	}

	// delivery adapters
	userHandler := userhandler.New(userService, logger)
	productHandler := product.NewHandler(productStore, logger)
	healthHandler := health.NewHandler(conn, logger)

	return rest.NewServer(cfg, logger, productHandler, userHandler, healthHandler).Handler(), nil
}
```

`Serve` passes the error on:

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

	"github.com/jmoiron/sqlx"
)

const shutdownTimeout = 10 * time.Second

// Serve starts the HTTP server and blocks until it stops.
// It shuts down gracefully when the process receives SIGINT (Ctrl+C) or SIGTERM.
func Serve(cfg *config.Config, logger *slog.Logger, conn *sqlx.DB) error {
	handler, err := buildHandler(cfg, logger, conn)
	if err != nil {
		return err
	}

	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           handler,
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

---

## 10. Tests: fast, and at the right level

The refactoring paid for itself here. Three kinds of tests, each at the lowest level that can check its subject:

| Level | Tests | Needs | Speed |
|-------|-------|-------|-------|
| **Domain** (`user`) | rules and orchestration with fakes | nothing | milliseconds |
| **Adapter** (`userhandler`) | JSON ↔ status-code mapping with a fake `Service` | `httptest` only | milliseconds |
| **Integration** (`postgres`, `rest`) | real repository, whole HTTP stack | PostgreSQL / bcrypt | slower |

### The value object

```go
// file: user/email_test.go
package user

import (
	"errors"
	"testing"
)

func TestNewEmailNormalizesAndValidates(t *testing.T) {
	good := map[string]string{
		"asha@example.com":         "asha@example.com",
		"  Asha@Example.COM  ":     "asha@example.com",
		"first.last+tag@sub.x.org": "first.last+tag@sub.x.org",
		"o'brien@example.com":      "o'brien@example.com",
	}
	for in, want := range good {
		got, err := NewEmail(in)
		if err != nil || got.String() != want {
			t.Errorf("NewEmail(%q) = %q, %v; want %q", in, got, err, want)
		}
	}

	for _, in := range []string{"", "   ", "no-at-sign", "@example.com", "a@", "a b@example.com",
		"Asha <asha@example.com>", "a@b@c.com", "two@example.com, three@example.com"} {
		_, err := NewEmail(in)
		var ve *ValidationError
		if !errors.As(err, &ve) {
			t.Errorf("NewEmail(%q) should return a *ValidationError, got %v", in, err)
		}
	}
}

func TestEmailsCompareByValue(t *testing.T) {
	a, _ := NewEmail("Asha@Example.com")
	b, _ := NewEmail(" asha@example.com ")
	c, _ := NewEmail("ravi@example.com")

	if a != b {
		t.Error("the same address, differently typed, must be equal after normalization")
	}
	if a == c {
		t.Error("different addresses must differ")
	}
	if !(Email{}).IsZero() || a.IsZero() {
		t.Error("only the zero Email is zero")
	}
}
```

### The service, with fakes

The fakes are ordinary structs, written by hand (Chapter 51). The hasher is a **fast fake that counts calls**; no real bcrypt (which is deliberately slow):

```go
// file: user/service_test.go
package user

import (
	"context"
	"errors"
	"strings"
	"testing"
	"time"
)

// ---- fakes -----------------------------------------------------------------

type fakeRepo struct {
	users   []User
	err     error // if set, every call fails with it
	created int
}

func (f *fakeRepo) Create(_ context.Context, email Email, hash string) (User, error) {
	if f.err != nil {
		return User{}, f.err
	}
	for _, u := range f.users {
		if u.Email == email {
			return User{}, ErrEmailTaken
		}
	}
	f.created++
	u := User{ID: len(f.users) + 1, Email: email, PasswordHash: hash, CreatedAt: time.Now()}
	f.users = append(f.users, u)
	return u, nil
}

func (f *fakeRepo) FindByEmail(_ context.Context, email Email) (User, error) {
	if f.err != nil {
		return User{}, f.err
	}
	for _, u := range f.users {
		if u.Email == email {
			return u, nil
		}
	}
	return User{}, ErrNotFound
}

func (f *fakeRepo) FindByID(_ context.Context, id int) (User, error) {
	if f.err != nil {
		return User{}, f.err
	}
	for _, u := range f.users {
		if u.ID == id {
			return u, nil
		}
	}
	return User{}, ErrNotFound
}

// fakeHasher "hashes" by prefixing, and counts how often it was asked to compare.
type fakeHasher struct {
	hashErr error
	matches int
}

func (f *fakeHasher) Hash(plain string) (string, error) {
	if f.hashErr != nil {
		return "", f.hashErr
	}
	return "hashed:" + plain, nil
}

func (f *fakeHasher) Matches(hash, plain string) bool {
	f.matches++
	return hash == "hashed:"+plain
}

type fakeTokens struct{ err error }

func (f fakeTokens) Issue(id int) (string, error) {
	if f.err != nil {
		return "", f.err
	}
	return "token-for-" + string(rune('0'+id)), nil
}

// newTestService builds a Service on fresh fakes and returns the fakes for inspection.
func newTestService(t *testing.T) (*Service, *fakeRepo, *fakeHasher) {
	t.Helper()
	repo, hasher := &fakeRepo{}, &fakeHasher{}
	svc, err := NewService(repo, hasher, fakeTokens{})
	if err != nil {
		t.Fatal(err)
	}
	hasher.matches = 0 // NewService hashes a dummy password; forget that
	return svc, repo, hasher
}

var ctx = context.Background()

// ---- Register --------------------------------------------------------------

func TestRegisterStoresANormalizedUserWithAHashedPassword(t *testing.T) {
	svc, repo, _ := newTestService(t)

	u, err := svc.Register(ctx, "  Asha@Example.COM ", "correct-horse")
	if err != nil {
		t.Fatal(err)
	}
	if u.Email.String() != "asha@example.com" {
		t.Errorf("email = %q", u.Email)
	}
	if u.PasswordHash == "correct-horse" || !strings.HasPrefix(u.PasswordHash, "hashed:") {
		t.Errorf("the stored value must be a hash, got %q", u.PasswordHash)
	}
	if len(repo.users) != 1 {
		t.Errorf("expected 1 stored user, got %d", len(repo.users))
	}
}

func TestRegisterEnforcesTheRules(t *testing.T) {
	tests := []struct {
		name, email, password, wantMessage string
	}{
		{"bad email", "not-an-email", "long-enough-pw", "email must be a valid address"},
		{"empty email", "", "long-enough-pw", "email must be a valid address"},
		{"short password", "a@example.com", "short", "at least 8 characters"},
		{"empty password", "a@example.com", "", "at least 8 characters"},
		{"password too long for bcrypt", "a@example.com", strings.Repeat("x", 73), "at most 72 bytes"},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			svc, repo, _ := newTestService(t)
			_, err := svc.Register(ctx, tc.email, tc.password)

			var ve *ValidationError
			if !errors.As(err, &ve) || !strings.Contains(ve.Message, tc.wantMessage) {
				t.Fatalf("expected a ValidationError containing %q, got %v", tc.wantMessage, err)
			}
			if repo.created != 0 {
				t.Error("an invalid registration must not reach the repository")
			}
		})
	}
}

func TestPasswordLengthBoundaries(t *testing.T) {
	for length, ok := range map[int]bool{7: false, 8: true, 72: true, 73: false} {
		svc, _, _ := newTestService(t)
		_, err := svc.Register(ctx, "a@example.com", strings.Repeat("p", length))
		if (err == nil) != ok {
			t.Errorf("length %d: accepted=%v, want %v (err=%v)", length, err == nil, ok, err)
		}
	}
}

func TestRegisterReportsDuplicates(t *testing.T) {
	svc, _, _ := newTestService(t)
	if _, err := svc.Register(ctx, "asha@example.com", "long-enough-pw"); err != nil {
		t.Fatal(err)
	}
	// same address, different typing: still a duplicate, because emails are normalized
	if _, err := svc.Register(ctx, " ASHA@Example.com", "another-password"); !errors.Is(err, ErrEmailTaken) {
		t.Errorf("expected ErrEmailTaken, got %v", err)
	}
}

func TestRegisterWrapsUnexpectedFailures(t *testing.T) {
	boom := errors.New("connection reset")

	svc, repo, _ := newTestService(t)
	repo.err = boom
	_, err := svc.Register(ctx, "a@example.com", "long-enough-pw")
	if !errors.Is(err, boom) || errors.Is(err, ErrEmailTaken) {
		t.Errorf("a storage failure must be wrapped, not disguised: %v", err)
	}

	svc, _, hasher := newTestService(t)
	hasher.hashErr = boom
	if _, err := svc.Register(ctx, "a@example.com", "long-enough-pw"); !errors.Is(err, boom) {
		t.Errorf("a hashing failure must surface: %v", err)
	}
}

// ---- Login -----------------------------------------------------------------

func registered(t *testing.T) (*Service, *fakeRepo, *fakeHasher) {
	t.Helper()
	svc, repo, hasher := newTestService(t)
	if _, err := svc.Register(ctx, "asha@example.com", "correct-horse"); err != nil {
		t.Fatal(err)
	}
	hasher.matches = 0
	return svc, repo, hasher
}

func TestLoginReturnsATokenForTheRightCredentials(t *testing.T) {
	svc, _, _ := registered(t)

	token, err := svc.Login(ctx, " ASHA@example.com ", "correct-horse")
	if err != nil || token != "token-for-1" {
		t.Errorf("got %q, %v", token, err)
	}
}

func TestLoginFailuresAreIndistinguishable(t *testing.T) {
	svc, _, _ := registered(t)

	for name, tc := range map[string]struct{ email, password string }{
		"wrong password":    {"asha@example.com", "wrong-password"},
		"unknown email":     {"ghost@example.com", "correct-horse"},
		"malformed email":   {"not-an-email", "correct-horse"},
		"empty credentials": {"", ""},
	} {
		_, err := svc.Login(ctx, tc.email, tc.password)
		if err != ErrInvalidCredentials {
			t.Errorf("%s: got %v, want exactly ErrInvalidCredentials", name, err)
		}
	}
}

func TestLoginAlwaysDoesTheExpensiveComparison(t *testing.T) {
	// Timing side channel: if unknown emails skipped the (slow) password check, response times
	// would reveal which emails are registered. The domain must run the comparison every time.
	for name, email := range map[string]string{
		"known email":     "asha@example.com",
		"unknown email":   "ghost@example.com",
		"malformed email": "not-an-email",
	} {
		svc, _, hasher := registered(t)
		svc.Login(ctx, email, "whatever-password")
		if hasher.matches != 1 {
			t.Errorf("%s: the password comparison ran %d times, want exactly 1", name, hasher.matches)
		}
	}
}

func TestLoginDistinguishesBrokenStorageFromBadCredentials(t *testing.T) {
	svc, repo, _ := registered(t)
	boom := errors.New("disk on fire")
	repo.err = boom

	_, err := svc.Login(ctx, "asha@example.com", "correct-horse")
	if !errors.Is(err, boom) || errors.Is(err, ErrInvalidCredentials) {
		t.Errorf("a storage failure must not be reported as bad credentials: %v", err)
	}
}

func TestLoginReportsTokenFailures(t *testing.T) {
	repo, hasher := &fakeRepo{}, &fakeHasher{}
	boom := errors.New("signing key unavailable")
	svc, _ := NewService(repo, hasher, fakeTokens{err: boom})
	svc.Register(ctx, "asha@example.com", "correct-horse")

	if _, err := svc.Login(ctx, "asha@example.com", "correct-horse"); !errors.Is(err, boom) {
		t.Errorf("got %v", err)
	}
}

// ---- Get -------------------------------------------------------------------

func TestGet(t *testing.T) {
	svc, repo, _ := registered(t)

	if u, err := svc.Get(ctx, 1); err != nil || u.Email.String() != "asha@example.com" {
		t.Errorf("got %+v, %v", u, err)
	}
	if _, err := svc.Get(ctx, 99); !errors.Is(err, ErrNotFound) {
		t.Errorf("expected ErrNotFound, got %v", err)
	}

	repo.err = errors.New("boom")
	if _, err := svc.Get(ctx, 1); err == nil || errors.Is(err, ErrNotFound) {
		t.Errorf("a storage failure must not look like ErrNotFound: %v", err)
	}
}

func TestNewServiceFailsIfHashingIsBroken(t *testing.T) {
	if _, err := NewService(&fakeRepo{}, &fakeHasher{hashErr: errors.New("no entropy")}, fakeTokens{}); err == nil {
		t.Error("expected NewService to report a hasher that cannot hash")
	}
}
```

Read `TestLoginAlwaysDoesTheExpensiveComparison` twice: it turns a **security property** ("response time must not reveal which emails exist") into an assertion about *behavior*: the comparison runs exactly once in all three cases. Before the refactoring, that property could only be checked by timing real bcrypt calls through a real HTTP server.

### The HTTP adapter, with a fake service

It only has to check the *translation*. The rules themselves were tested above and need not be repeated:

```go
// file: rest/userhandler/handler_test.go
package userhandler

import (
	"bytes"
	"context"
	"ecommerce/auth"
	"ecommerce/user"
	"encoding/json"
	"errors"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

// fakeService lets each test decide what the domain answers.
type fakeService struct {
	register func(email, password string) (user.User, error)
	login    func(email, password string) (string, error)
	get      func(id int) (user.User, error)
}

func (f fakeService) Register(_ context.Context, e, p string) (user.User, error) {
	return f.register(e, p)
}
func (f fakeService) Login(_ context.Context, e, p string) (string, error) { return f.login(e, p) }
func (f fakeService) Get(_ context.Context, id int) (user.User, error)     { return f.get(id) }

var _ Service = (*user.Service)(nil) // the real service satisfies the handler's interface

func newHandler(svc Service) (*Handler, *bytes.Buffer) {
	var logs bytes.Buffer
	return New(svc, slog.New(slog.NewTextHandler(&logs, nil))), &logs
}

func post(h http.HandlerFunc, body string) *httptest.ResponseRecorder {
	rec := httptest.NewRecorder()
	h(rec, httptest.NewRequest(http.MethodPost, "/", strings.NewReader(body)))
	return rec
}

func mustEmail(s string) user.Email {
	e, err := user.NewEmail(s)
	if err != nil {
		panic(err)
	}
	return e
}

func TestRegisterMapsDomainOutcomesToStatusCodes(t *testing.T) {
	tests := []struct {
		name string
		err  error
		want int
		body string
	}{
		{"validation", &user.ValidationError{Message: "password must be at least 8 characters"}, 422, "password must be at least 8 characters"},
		{"duplicate", user.ErrEmailTaken, 409, "email already registered"},
		{"unexpected", errors.New("connection to db lost: password=hunter2"), 500, "internal server error"},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			h, logs := newHandler(fakeService{register: func(string, string) (user.User, error) { return user.User{}, tc.err }})
			rec := post(h.Register, `{"email":"a@b.co","password":"whatever-it-is"}`)

			if rec.Code != tc.want || !strings.Contains(rec.Body.String(), tc.body) {
				t.Errorf("got %d %s, want %d containing %q", rec.Code, rec.Body.String(), tc.want, tc.body)
			}
			if tc.want == 500 {
				if strings.Contains(rec.Body.String(), "hunter2") {
					t.Error("internal details leaked to the client")
				}
				if !strings.Contains(logs.String(), "connection to db lost") {
					t.Errorf("the cause must be logged: %q", logs.String())
				}
			}
		})
	}
}

func TestRegisterResponseNeverContainsTheHash(t *testing.T) {
	created := time.Date(2026, 9, 26, 6, 0, 0, 0, time.UTC)
	h, _ := newHandler(fakeService{register: func(string, string) (user.User, error) {
		return user.User{ID: 7, Email: mustEmail("asha@example.com"), PasswordHash: "$2a$10$SECRET", CreatedAt: created}, nil
	}})

	rec := post(h.Register, `{"email":"asha@example.com","password":"long-enough-pw"}`)
	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d", rec.Code)
	}
	if strings.Contains(rec.Body.String(), "SECRET") || strings.Contains(strings.ToLower(rec.Body.String()), "hash") {
		t.Errorf("the response leaks the password hash: %s", rec.Body.String())
	}

	var got map[string]any
	json.Unmarshal(rec.Body.Bytes(), &got)
	if got["id"] != float64(7) || got["email"] != "asha@example.com" || got["createdAt"] != "2026-09-26T06:00:00Z" {
		t.Errorf("unexpected JSON: %v", got)
	}
}

func TestBadRequestBodiesNeverReachTheDomain(t *testing.T) {
	called := false
	h, _ := newHandler(fakeService{
		register: func(string, string) (user.User, error) { called = true; return user.User{}, nil },
		login:    func(string, string) (string, error) { called = true; return "", nil },
	})

	for _, body := range []string{``, `{`, `[]`, `{"email":1}`, `{"email":"a@b.co","password":"x","admin":true}`} {
		if rec := post(h.Register, body); rec.Code != http.StatusBadRequest {
			t.Errorf("Register(%q): expected 400, got %d", body, rec.Code)
		}
		if rec := post(h.Login, body); rec.Code != http.StatusBadRequest {
			t.Errorf("Login(%q): expected 400, got %d", body, rec.Code)
		}
	}
	if called {
		t.Error("the domain must not be called for undecodable requests")
	}
}

func TestLogin(t *testing.T) {
	h, _ := newHandler(fakeService{login: func(email, password string) (string, error) {
		if email == "asha@example.com" && password == "correct-horse" {
			return "the-token", nil
		}
		return "", user.ErrInvalidCredentials
	}})

	rec := post(h.Login, `{"email":"asha@example.com","password":"correct-horse"}`)
	if rec.Code != http.StatusOK || !strings.Contains(rec.Body.String(), `"token":"the-token"`) {
		t.Errorf("got %d %s", rec.Code, rec.Body.String())
	}

	rec = post(h.Login, `{"email":"asha@example.com","password":"wrong"}`)
	if rec.Code != http.StatusUnauthorized || !strings.Contains(rec.Body.String(), "invalid email or password") {
		t.Errorf("got %d %s", rec.Code, rec.Body.String())
	}
}

func TestMe(t *testing.T) {
	h, _ := newHandler(fakeService{get: func(id int) (user.User, error) {
		switch id {
		case 1:
			return user.User{ID: 1, Email: mustEmail("asha@example.com")}, nil
		case 2:
			return user.User{}, user.ErrNotFound // token for a deleted account
		}
		return user.User{}, errors.New("boom")
	}})

	call := func(withID int, authenticated bool) *httptest.ResponseRecorder {
		req := httptest.NewRequest(http.MethodGet, "/me", nil)
		if authenticated {
			req = req.WithContext(auth.ContextWithUserID(req.Context(), withID))
		}
		rec := httptest.NewRecorder()
		h.Me(rec, req)
		return rec
	}

	if rec := call(1, true); rec.Code != 200 || !strings.Contains(rec.Body.String(), "asha@example.com") {
		t.Errorf("known user: %d %s", rec.Code, rec.Body.String())
	}
	if rec := call(2, true); rec.Code != 401 || !strings.Contains(rec.Body.String(), "account not found") {
		t.Errorf("deleted account: %d %s", rec.Code, rec.Body.String())
	}
	if rec := call(3, true); rec.Code != 500 {
		t.Errorf("storage failure: expected 500, got %d", rec.Code)
	}
	if rec := call(0, false); rec.Code != 401 {
		t.Errorf("no identity in the context: expected 401, got %d", rec.Code)
	}
}
```

### The repository contract, updated

The contract suite now speaks in `user` types. The case-insensitivity scenario moved into the domain (`TestEmailsCompareByValue`), because a `Repository` can only ever receive normalized `Email`s:

```go
// file: user/storetest/storetest.go
// Package storetest holds a behavior suite that every user.Repository implementation must pass.
package storetest

import (
	"context"
	"ecommerce/user"
	"errors"
	"sync"
	"testing"
	"time"
)

// Factory returns a fresh, empty repository for one test.
type Factory func(t *testing.T) user.Repository

func email(t *testing.T, s string) user.Email {
	t.Helper()
	e, err := user.NewEmail(s)
	if err != nil {
		t.Fatal(err)
	}
	return e
}

// Run executes the contract against repositories made by newRepo.
func Run(t *testing.T, newRepo Factory) {
	ctx := context.Background()

	t.Run("create returns the stored user", func(t *testing.T) {
		s := newRepo(t)
		u, err := s.Create(ctx, email(t, "asha@example.com"), "hash-1")
		if err != nil {
			t.Fatal(err)
		}
		if u.ID < 1 {
			t.Errorf("expected a generated ID, got %d", u.ID)
		}
		if u.Email != email(t, "asha@example.com") {
			t.Errorf("email = %q", u.Email)
		}
		if u.PasswordHash != "hash-1" {
			t.Errorf("hash = %q", u.PasswordHash)
		}
		if age := time.Since(u.CreatedAt); age < -time.Second || age > time.Minute {
			t.Errorf("CreatedAt %v is not about now", u.CreatedAt)
		}
		if u.CreatedAt.Location() != time.UTC {
			t.Errorf("CreatedAt must be in UTC, got %v", u.CreatedAt.Location())
		}
	})

	t.Run("find by email", func(t *testing.T) {
		s := newRepo(t)
		want, _ := s.Create(ctx, email(t, "asha@example.com"), "h")

		got, err := s.FindByEmail(ctx, email(t, "  ASHA@Example.com")) // NewEmail normalizes
		if err != nil {
			t.Fatal(err)
		}
		if got.ID != want.ID || got.PasswordHash != "h" {
			t.Errorf("got %+v, want %+v", got, want)
		}
	})

	t.Run("find by id", func(t *testing.T) {
		s := newRepo(t)
		want, _ := s.Create(ctx, email(t, "asha@example.com"), "h")

		got, err := s.FindByID(ctx, want.ID)
		if err != nil || got.Email != want.Email {
			t.Errorf("got %+v, %v", got, err)
		}
	})

	t.Run("missing users are ErrNotFound", func(t *testing.T) {
		s := newRepo(t)
		if _, err := s.FindByEmail(ctx, email(t, "ghost@example.com")); !errors.Is(err, user.ErrNotFound) {
			t.Errorf("FindByEmail: got %v", err)
		}
		if _, err := s.FindByID(ctx, 12345); !errors.Is(err, user.ErrNotFound) {
			t.Errorf("FindByID: got %v", err)
		}
	})

	t.Run("duplicate emails are ErrEmailTaken", func(t *testing.T) {
		s := newRepo(t)
		if _, err := s.Create(ctx, email(t, "asha@example.com"), "h"); err != nil {
			t.Fatal(err)
		}
		if _, err := s.Create(ctx, email(t, " ASHA@EXAMPLE.COM "), "h2"); !errors.Is(err, user.ErrEmailTaken) {
			t.Errorf("got %v, want ErrEmailTaken", err)
		}
	})

	t.Run("IDs are distinct and increasing", func(t *testing.T) {
		s := newRepo(t)
		a, _ := s.Create(ctx, email(t, "a@example.com"), "h")
		b, _ := s.Create(ctx, email(t, "b@example.com"), "h")
		if a.ID == b.ID || b.ID < a.ID {
			t.Errorf("IDs must be distinct and increasing: %d then %d", a.ID, b.ID)
		}
	})

	t.Run("awkward but valid addresses are stored as plain data", func(t *testing.T) {
		s := newRepo(t)
		awkward := email(t, "o'brien+tag@example.com") // an apostrophe: a classic SQL-injection character
		if _, err := s.Create(ctx, awkward, "h"); err != nil {
			t.Fatal(err)
		}
		got, err := s.FindByEmail(ctx, awkward)
		if err != nil || got.Email != awkward {
			t.Errorf("got %+v, %v", got, err)
		}
		if _, err := s.Create(ctx, email(t, "after@example.com"), "h"); err != nil {
			t.Errorf("the repository must still work afterwards: %v", err)
		}
	})

	t.Run("concurrent duplicate registrations: exactly one wins", func(t *testing.T) {
		s := newRepo(t)

		const attempts = 30
		var wg sync.WaitGroup
		var mu sync.Mutex
		wins := 0

		for i := 0; i < attempts; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()
				_, err := s.Create(ctx, email(t, "race@example.com"), "h")
				switch {
				case err == nil:
					mu.Lock()
					wins++
					mu.Unlock()
				case !errors.Is(err, user.ErrEmailTaken):
					t.Errorf("unexpected error: %v", err)
				}
			}()
		}
		wg.Wait()

		if wins != 1 {
			t.Fatalf("exactly one registration must succeed, got %d", wins)
		}
	})
}
```

Both adapters run it:

```go
// file: database/user_store_test.go
package database

import (
	"ecommerce/user"
	"ecommerce/user/storetest"
	"testing"
)

var _ user.Repository = (*UserStore)(nil)

func TestUserStoreContract(t *testing.T) {
	storetest.Run(t, func(t *testing.T) user.Repository { return NewUserStore() })
}
```

```go
// file: postgres/user_store_test.go
package postgres

import (
	"context"
	"ecommerce/user"
	"ecommerce/user/storetest"
	"errors"
	"testing"
)

var _ user.Repository = (*UserStore)(nil) // the real store satisfies the domain's port

func TestUserStoreContract(t *testing.T) {
	storetest.Run(t, func(t *testing.T) user.Repository { return NewUserStore(newTestDB(t).DB) })
}

func mustEmail(t *testing.T, s string) user.Email {
	t.Helper()
	e, err := user.NewEmail(s)
	if err != nil {
		t.Fatal(err)
	}
	return e
}

func TestPasswordHashIsPersisted(t *testing.T) {
	d := newTestDB(t)
	NewUserStore(d.DB).Create(context.Background(), mustEmail(t, "asha@example.com"), "$2a$10$abcdef")

	var stored string
	if err := d.Get(&stored, "SELECT password_hash FROM users"); err != nil || stored != "$2a$10$abcdef" {
		t.Errorf("stored hash = %q (%v)", stored, err)
	}
}

func TestDatabaseRejectsWhatTheStoreWouldNot(t *testing.T) {
	d := newTestDB(t)

	// bypass the Go code entirely: the constraints still protect the data
	insert := func(email string) error {
		_, err := d.Exec("INSERT INTO users (email, password_hash) VALUES ($1, 'h')", email)
		return err
	}
	if err := insert("dup@example.com"); err != nil {
		t.Fatal(err)
	}
	if err := insert("DUP@example.com"); err == nil {
		t.Error("the unique index on lower(email) must reject a differently-cased duplicate")
	}
	if err := insert(""); err == nil {
		t.Error("the CHECK constraint must reject an empty email")
	}
}

func TestCancelledContextStopsTheQuery(t *testing.T) {
	s := NewUserStore(newTestDB(t).DB)
	ctx, cancel := context.WithCancel(context.Background())
	cancel()

	if _, err := s.FindByID(ctx, 1); err == nil || errors.Is(err, user.ErrNotFound) {
		t.Errorf("a cancelled context must give an error that is not ErrNotFound, got %v", err)
	}
}

func TestCorruptRowsAreReportedNotHidden(t *testing.T) {
	d := newTestDB(t)
	// Someone edited the table by hand and stored something that is not an email address.
	if _, err := d.Exec("INSERT INTO users (email, password_hash) VALUES ('not an email', 'h')"); err != nil {
		t.Fatal(err)
	}
	var id int
	d.Get(&id, "SELECT id FROM users")

	_, err := NewUserStore(d.DB).FindByID(context.Background(), id)
	if err == nil || errors.Is(err, user.ErrNotFound) {
		t.Errorf("a corrupt row must be a loud error, not a silent NotFound: %v", err)
	}
}

func TestDataOutlivesThePool(t *testing.T) {
	// Persistence in miniature: close every connection, open new ones, and the user is still there.
	d := newTestDB(t)
	created, err := NewUserStore(d.DB).Create(context.Background(), mustEmail(t, "durable@example.com"), "h")
	if err != nil {
		t.Fatal(err)
	}
	fresh := d.reconnect(t) // a second, independent pool on the same schema

	d.DB.Close() // the first pool is gone, along with all its connections
	got, err := NewUserStore(fresh).FindByID(context.Background(), created.ID)
	if err != nil || got.Email.String() != "durable@example.com" {
		t.Errorf("after the first pool closed: %+v, %v", got, err)
	}
}
```

### End-to-end tests: unchanged, apart from wiring

The `rest` tests (the whole HTTP stack against in-memory repositories) are the **safety net for the refactoring**: their *assertions* don't change at all. Only the helper that builds the application does:

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
	t        *testing.T
	cfg      *config.Config
	handler  http.Handler
	products *database.ProductStore
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

	// the same wiring as production (cmd/wire.go), with in-memory storage
	userService, err := user.NewService(database.NewUserStore(), auth.BcryptHasher{},
		auth.TokenIssuer{Secret: []byte(cfg.JWTSecret), TTL: cfg.JWTTTL})
	if err != nil {
		t.Fatal(err)
	}

	server := NewServer(cfg, quietLogger,
		product.NewHandler(products, quietLogger),
		userhandler.New(userService, quietLogger),
		health.NewHandler(alwaysUp{}, quietLogger),
	)
	return &testApp{t: t, cfg: cfg, handler: server.Handler(), products: products}
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

Because the domain is now separate, the old `database/stores_test.go` (product tests) still compiles unchanged, and the `models.ErrNotFound` product errors are untouched until Chapter 61.

Run everything:

```bash
go vet ./... && go test -race ./...
TEST_DATABASE_URL='postgres://postgres:devpass@127.0.0.1:15432/ecommerce?sslmode=disable' go test -race -count=1 ./...
```

---

## 11. Proving nothing changed

A refactoring is *behavior-preserving*. Besides the unchanged end-to-end tests, drive the real server (PostgreSQL behind it) with the same requests as in Chapter 53:

```
$ curl -X POST localhost:18080/users -d '{"email":"Asha@Example.com","password":"correct-horse-battery"}'
HTTP/1.1 201 Created
{"id":1,"email":"asha@example.com","createdAt":"2026-09-26T10:29:48.479789Z"}

$ curl -X POST localhost:18080/users -d '{"email":"ASHA@example.com","password":"another-password"}'
HTTP/1.1 409 Conflict
{"error":"email already registered"}

$ curl -X POST localhost:18080/users -d '{"email":"nope","password":"another-password"}'
HTTP/1.1 422 Unprocessable Entity
{"error":"email must be a valid address like name@example.com"}

$ curl -X POST localhost:18080/users -d '{"email":"a@b.co","password":"short"}'
HTTP/1.1 422 Unprocessable Entity
{"error":"password must be at least 8 characters"}

$ curl -X POST localhost:18080/login -d '{"email":"asha@example.com","password":"wrong-password"}'
HTTP/1.1 401 Unauthorized
{"error":"invalid email or password"}

$ curl -X POST localhost:18080/login -d '{"email":"ghost@example.com","password":"wrong-password"}'   # unknown email: same answer
HTTP/1.1 401 Unauthorized
{"error":"invalid email or password"}

$ curl localhost:18080/me -H "Authorization: Bearer $TOKEN"
HTTP/1.1 200 OK
{"id":1,"email":"asha@example.com","createdAt":"2026-09-26T10:29:48.479789Z"}
```

Identical statuses and bodies to the pre-refactoring API (`201` with `id/email/createdAt`, `409 email already registered`, `401 invalid email or password`, `200` on `/me`, and `422` for bad input).

And here is the speed dividend. The *entire domain* suite (15 tests, dozens of assertions) of business rules, security properties and failure modes:

```
$ go test -count=1 -v ./user/ | grep -E "^(--- |ok)"
--- PASS: TestDomainDoesNotImportTechnology (0.00s)
--- PASS: TestNewEmailNormalizesAndValidates (0.00s)
--- PASS: TestEmailsCompareByValue (0.00s)
--- PASS: TestRegisterStoresANormalizedUserWithAHashedPassword (0.00s)
--- PASS: TestRegisterEnforcesTheRules (0.00s)
--- PASS: TestPasswordLengthBoundaries (0.00s)
--- PASS: TestRegisterReportsDuplicates (0.00s)
--- PASS: TestRegisterWrapsUnexpectedFailures (0.00s)
--- PASS: TestLoginReturnsATokenForTheRightCredentials (0.00s)
--- PASS: TestLoginFailuresAreIndistinguishable (0.00s)
--- PASS: TestLoginAlwaysDoesTheExpensiveComparison (0.00s)
--- PASS: TestLoginDistinguishesBrokenStorageFromBadCredentials (0.00s)
--- PASS: TestLoginReportsTokenFailures (0.00s)
--- PASS: TestGet (0.00s)
--- PASS: TestNewServiceFailsIfHashingIsBroken (0.00s)
ok  	ecommerce/user	0.003s
```

(The old way to test "wrong passwords are indistinguishable from unknown emails" involved a running HTTP server, a database, and real bcrypt at about 60-100 ms per hash.)

---

## 12. The golden rules, checked

Chapter 59's rules, and how the code above obeys them:

| Rule | Evidence |
|------|----------|
| The domain imports nothing technical | `user` imports only `context`, `errors`, `fmt`, `net/mail`, `strings`, `time` |
| Outer layers import the domain, never reverse | `postgres`, `database`, `userhandler`, `cmd` import `user`; `user` imports none of them |
| Rules exist once | email → `NewEmail`; password policy → `validatePassword`; login timing → `Service.Login` |
| Only the composition root knows all concrete types | `cmd/wire.go` |
| Access through interfaces | handler → `Service`; service → `Repository`/`PasswordHasher`/`TokenIssuer` |

The first rule can be *enforced by a test*. A compact one for this project:

```go
// file: user/architecture_test.go
package user

import (
	"go/parser"
	"go/token"
	"strings"
	"testing"
)

// The domain must stay free of technology: fail the build if it starts importing any.
func TestDomainDoesNotImportTechnology(t *testing.T) {
	forbidden := []string{"net/http", "database/sql", "encoding/json", "ecommerce/postgres", "ecommerce/database",
		"ecommerce/rest", "ecommerce/auth", "ecommerce/util", "github.com/"}

	fset := token.NewFileSet()
	pkgs, err := parser.ParseDir(fset, ".", nil, parser.ImportsOnly)
	if err != nil {
		t.Fatal(err)
	}
	for _, pkg := range pkgs {
		for name, file := range pkg.Files {
			if strings.HasSuffix(name, "_test.go") {
				continue // tests may use whatever helps
			}
			for _, imp := range file.Imports {
				path := strings.Trim(imp.Path.Value, `"`)
				for _, bad := range forbidden {
					if strings.HasPrefix(path, bad) {
						t.Errorf("%s imports %q: the domain must not depend on outer layers or technology", name, path)
					}
				}
			}
		}
	}
}
```

Now the architecture is *enforced*, not just documented: add `import "net/http"` to `service.go` and CI goes red.

---

## 13. Change-impact analysis

Test the design by asking "what would I have to touch if…?"

| Change | Before (Chapter 58) | Now |
|--------|---------------------|-----|
| Require passwords ≥ 12 characters | `user/register.go` (HTTP code), tests | `user/service.go` (1 constant) + its unit test |
| Add a `phone` field to users | `models`, handlers, store, SQL, tests | `user.User`, `userRow` mapping, SQL, migration (handler if exposed) |
| Replace bcrypt with argon2 | `auth` + handlers that call it | write `Argon2Hasher` in `auth`; change **one line** in `wire.go` |
| Switch JWT for opaque session tokens | handlers, middleware | new `TokenIssuer`, new middleware; the **service is untouched** |
| Add a gRPC API next to REST | duplicate the rules in new handlers | write a gRPC adapter calling the same `user.Service` |
| Add a CLI command "create admin user" | re-implement validation | call `user.Service.Register` |
| Move users to another database | store + SQL | new `Repository` adapter + contract test; domain untouched |

The pattern in the right-hand column is the whole argument for the architecture: *changes stay local*, and the **rules live where the business words are**.

---

## 14. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Refactoring without a safety net | Silent behavior changes | Keep the end-to-end tests green at every step |
| 2 | Domain imports `net/http`/`sql`/JSON tags | Domain can't be reused/tested alone | Ports and adapters; the architecture test |
| 3 | Returning HTTP status codes from the service | Domain knows about delivery | Return domain errors; map in the adapter |
| 4 | Returning `sql.ErrNoRows` from the repository adapter | Handlers depend on the database | Translate to `user.ErrNotFound` |
| 5 | Logging in the service *and* the handler | Duplicate log lines | Return errors; log once where handled |
| 6 | Interface declared in the *implementation* package | Import direction wrong | Declare ports in the domain; adapters implement them implicitly |
| 7 | Anemic `User` + logic scattered in services | Lost encapsulation | Put rules that concern one object on it (`NewEmail`) |
| 8 | Sharing the domain entity as the JSON response | Fields leak (hash!), API coupled to entity | Response DTO (`userResponse`) |
| 9 | Fakes that don't behave like the real thing | Tests pass, production fails | Contract tests run against the fakes' real counterparts |
| 10 | Testing rules only through HTTP | Slow tests, hard to isolate | Unit-test the service; keep a few end-to-end tests |
| 11 | A `Manager`/`Helper` grab-bag package | Vocabulary drift | Name after the business (`Service`, `Register`, `Login`) |
| 12 | Constructing dependencies inside the domain (`bcrypt.Generate…`) | Hidden coupling | Inject ports from `cmd` |
| 13 | Skipping the mapping code "because it's boring" | Domain forced to carry DB/JSON tags | Accept the mapping cost at boundaries |
| 14 | Big-bang rewrite | Weeks of red tests | The 7-step recipe, green throughout |

---

## 15. Interview questions

**Q1. What did you move, and why?**
Business rules and orchestration moved from the HTTP handler into a domain `Service`, so they're reusable, testable without HTTP/DB, and change in one place.

**Q2. What is a port, and who owns it?**
An interface describing what the domain needs (`Repository`, `PasswordHasher`). The domain owns it; adapters implement it.

**Q3. Why does `Email` have an unexported field?**
So the only way to construct one is `NewEmail`, making every `Email` provably valid and normalized ("make illegal states unrepresentable").

**Q4. How do you keep login timing independent of whether the email exists?**
Always run the expensive password comparison, against a dummy hash when the user doesn't exist; test that the comparison happens exactly once in every path.

**Q5. Where are HTTP status codes decided?**
Only in the HTTP adapter's error-translation function; the domain returns typed/sentinel errors.

**Q6. Why a separate `userRow` in the PostgreSQL adapter?**
To keep persistence details (tags, column names, types) out of the domain, at the price of a small mapping function.

**Q7. How do you know a refactoring didn't break behavior?**
Unchanged end-to-end tests plus manual comparison of responses; small steps, tests green throughout.

**Q8. What would you do if two adapters (memory, PostgreSQL) disagreed?**
The contract suite would fail for one; fix the adapter (or refine the contract if the behavior was unspecified).

**Q9. Is this over-engineering for a user service?**
For a demo CRUD, maybe; for a system with security-sensitive rules that several entry points must share and change over time, the separation pays for itself. Scale the structure to the complexity.

---

## 16. Exercises

### Exercise 1: Change the password policy
Require at least one digit. Where do you change code? Write the failing test first.

<details><summary>Solution</summary>

Add a failing case to `TestRegisterEnforcesTheRules` (`"onlyletters"` → message containing "digit"), then extend `validatePassword` in `user/service.go`. No handler, adapter, or SQL change: the payoff of centralizing the rule.
</details>

### Exercise 2: A new use case
Add `ChangePassword(ctx, userID int, oldPassword, newPassword string) error` to the service. Which port method is missing? Which errors does it return?

<details><summary>Solution</summary>

The repository needs `UpdatePasswordHash(ctx, id int, hash string) error` (with a contract-suite scenario and implementations in both adapters). The service: `Get` the user (`ErrNotFound`), check `Matches(old)` (else `ErrInvalidCredentials`), `validatePassword(new)`, hash, update. Add `ChangePassword` to the handler's `Service` interface and a `PUT /me/password` route.
</details>

### Exercise 3: A stricter `Email`
Reject addresses longer than 254 characters and reject domains without a dot (`a@localhost`). Add table-driven tests for `NewEmail`. Does anything outside `email.go` need to change?

<details><summary>Solution</summary>

Add length and `strings.Contains(domain, ".")` checks in `NewEmail` plus cases in `TestNewEmailNormalizesAndValidates`. Nothing else changes: every `Email` in the program passes through this constructor, which is the point of the value object.
</details>

### Exercise 4: Rate limiting as a port
Login must allow at most 5 failed attempts per email per 15 minutes. Design the port (`AttemptLimiter`), where it's called in `Login`, and how you'd unit-test it.

<details><summary>Solution</summary>

```go
type AttemptLimiter interface {
	Allow(key string) bool   // false when blocked
	RecordFailure(key string)
	Reset(key string)
}
```
In `Login`: if `!limiter.Allow(email)` return `ErrTooManyAttempts` (new domain error → 429); call `RecordFailure` on `ErrInvalidCredentials`, `Reset` on success. Test with a fake limiter (and a clock port if the fake needs time). Adapters: in-memory, or Redis for multiple instances.
</details>

### Exercise 5: Prove the adapters are swappable
Write an adapter `user.Repository` backed by a JSON file. Run the contract suite against it. What did the suite catch?

<details><summary>Solution</summary>

Typical catches: forgetting the mutex (the concurrency scenario fails under `-race`), not returning `ErrEmailTaken`, non-UTC times, ID reuse after restart. Passing the contract is the definition of "a valid `Repository`".
</details>

### Exercise 6 (challenge): Fail the build on a bad dependency
Extend the architecture test to check *every* domain package (`user`, and later `product`), and to also forbid the domain packages from importing **each other** (Identity ↔ Catalog independence).

<details><summary>Solution</summary>

Loop over a list of domain directories; for each, parse imports and assert none start with `ecommerce/` other than an explicit allow-list (e.g., none at all). When a genuine dependency is needed (Orders → Catalog), make it explicit in the allow-list *and* one-directional.
</details>

---

## 17. Quiz

1. Where do password rules live now?
2. Who decides the HTTP status for `ErrEmailTaken`?
3. Why does `Service.Login` call `hasher.Matches` even for unknown emails?
4. What's the difference between `ErrInvalidCredentials` and `*ValidationError`?
5. Why don't domain types carry `json`/`db` tags?
6. Which layer imports which?
7. What tests would still catch a regression in `postgres.UserStore`?
8. What's the cost of the row/entity mapping, and the benefit?

<details><summary>Answers</summary>

1. In the domain (`validatePassword` in `user/service.go`).
2. The HTTP adapter's `fail` function (409).
3. To equalize timing so response times don't reveal which emails are registered.
4. `ErrInvalidCredentials` is a fixed sentinel for a wrong email/password pair (401); `*ValidationError` carries a user-facing message about malformed input (422).
5. Those are representation concerns; the domain shouldn't change when the wire format or table layout does.
6. Adapters (`rest/userhandler`, `postgres`, `database`, `auth`, `cmd`) import the domain (`user`); the domain imports none of them.
7. The repository contract suite (run against PostgreSQL), plus the PostgreSQL-specific tests.
8. A few lines of conversion per entity; independence of table shape, API shape, and domain model.
</details>

---

## 18. Summary

- The **user domain** (`package user`) now holds the entity, the `Email` **value object**, **domain errors**, three **ports**, and a **`Service`** implementing `Register`, `Login`, `Get`, with **no dependency on HTTP, SQL, bcrypt, or JWT**.
- **Adapters** point inward: `postgres.UserStore` (with its own `userRow`), the in-memory store, `auth.BcryptHasher`/`TokenIssuer`, and a thin `userhandler` that only decodes, calls, and **translates errors to status codes** in one function.
- The **response type has no hash field**, and the HTTP behavior is byte-for-byte what it was; the old end-to-end tests are the proof.
- **Tests moved to the right level:** domain rules and security properties (timing equalization, indistinguishable failures) as millisecond unit tests with fakes; translation tests with a fake `Service`; a **contract suite** for repositories; a few end-to-end tests. An **architecture test** enforces the dependency rule.
- Cost: more files, mapping code, a `buildHandler` that can fail. Benefit: **changes stay local** (Section 13).

### ➡️ What's next?

[Chapter 61](61-ddd-in-code-part-2.md) repeats the recipe for the **product domain**: entity, validation rules, `Service`, ports, adapters: and finishes the job by deleting the shared `models` package. Then we'll discuss file organization (when to split a file), and how this structure prepares us for serious unit testing.
