# Chapter 49: Authentication Middleware — Verifying Tokens and Protecting Your APIs

> **Goal of this chapter:** Close the authentication loop. Chapter 48 issued tokens; now the server must **verify** them. You'll parse the `Authorization: Bearer <token>` header, **validate the signature** (with a constant-time comparison), **pin the algorithm**, check **expiry**, and package all of it into an **authentication middleware** that protects individual routes and makes the caller's identity available to handlers through `context`. Then you'll attack your own implementation with tampered, forged, and expired tokens to see it hold.

**Difficulty:** 🔴 Advanced  **Estimated time:** 4–5 hours  **Prerequisite:** [Chapters 46, 48](48-authentication-with-jwt.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The authentication flow, end to end](#2-the-authentication-flow)
3. [The `Authorization` header](#3-the-authorization-header)
4. [Verifying a token: the algorithm](#4-verifying-a-token)
5. [Constant-time comparison](#5-constant-time-comparison)
6. [`ParseToken`](#6-parsetoken)
7. [Carrying identity: context helpers in `auth`](#7-carrying-identity-context-helpers)
8. [The authentication middleware](#8-the-authentication-middleware)
9. [Route-level protection (and why not global)](#9-route-level-protection)
10. [A protected endpoint: `GET /me`](#10-a-protected-endpoint)
11. [The complete code](#11-the-complete-code)
12. [Tests: attacking our own verifier](#12-tests)
13. [Trying it out](#13-trying-it-out)
14. [The passport analogy](#14-the-passport-analogy)
15. [Where tokens live in a browser, and other real-world issues](#15-real-world-issues)
16. [Problems we still have](#16-problems-we-still-have)
17. [Common mistakes](#17-common-mistakes)
18. [Exercises](#18-exercises)
19. [Quiz](#19-quiz)
20. [Summary](#20-summary)

---

## 1. What you will learn

- How clients send tokens: the **`Authorization: Bearer`** header
- The exact steps to **verify** a JWT, and why each step matters
- Why signatures must be compared with **`hmac.Equal`** (constant time), never `==`
- Defending against the **`alg: none`** and algorithm-confusion attacks by **pinning** the algorithm
- Writing an **authentication middleware** and attaching the user ID to the request **context**
- Applying middleware to **specific routes** with `mux.Handle`
- How to test a security feature by **attacking it**

---

## 2. The authentication flow

The complete picture (Chapter 48 built the left half; this chapter builds the right):

```
 CLIENT                                                         SERVER
   │  POST /login {email, password}                               │
   │ ────────────────────────────────────────────────────────────►│  check bcrypt hash
   │                                                              │  sign {sub, iat, exp}
   │  200 {"token": "eyJ..."}                                     │
   │ ◄────────────────────────────────────────────────────────────│
   │                                                              │
   │  POST /products                                              │
   │  Authorization: Bearer eyJ...                                │
   │ ────────────────────────────────────────────────────────────►│  ┌─ Authenticate middleware ──┐
   │                                                              │  │ 1. read the header         │
   │                                                              │  │ 2. verify signature        │
   │                                                              │  │ 3. check expiry            │
   │                                                              │  │ 4. put user ID in context  │
   │                                                              │  └───────────┬────────────────┘
   │                                                              │              ▼
   │                                                              │      CreateProduct handler
   │  201 Created                                                 │
   │ ◄────────────────────────────────────────────────────────────│
```

If any check fails, the middleware answers `401` and the handler never runs.

---

## 3. The `Authorization` header

The HTTP standard defines an `Authorization` request header carrying credentials. For token-based auth the format is:

```
Authorization: Bearer <token>
```

- **`Authorization`**: the standard header name.
- **`Bearer`**: the *scheme*, meaning "whoever bears (holds) this token is treated as its subject". It's a **bearer token**: possession is proof. That's why tokens must be protected like passwords (HTTPS always, never in URLs or logs).
- **`<token>`**: our JWT, separated from the scheme by **one space**.

```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." http://localhost:8080/me
```

In Postman/Insomnia/Bruno: *Authorization tab → Bearer Token → paste*; the tool builds the header.

**Parsing it in Go:**

```go
func bearerToken(r *http.Request) (string, bool) {
	header := r.Header.Get("Authorization")           // "Bearer eyJhbGci..."
	scheme, token, found := strings.Cut(header, " ")   // split at the FIRST space
	if !found || !strings.EqualFold(scheme, "Bearer") { // scheme names are case-insensitive
		return "", false
	}
	token = strings.TrimSpace(token)
	if token == "" || strings.Contains(token, " ") {
		return "", false
	}
	return token, true
}
```

`strings.Cut(s, sep)` (Go 1.18+) returns the text before and after the first `sep`, plus whether it was found: a tidy alternative to `strings.Split(header, " ")` plus index checks. It handles the missing-header case for free: `header == ""` → `found == false`.

---

## 4. Verifying a token

Given a token string, the server must decide: *did I issue this, is it intact, and is it still valid?* The algorithm:

```
1. Split on "." → exactly three non-empty parts.       (malformed → reject)
2. Decode the HEADER. Check alg == "HS256".            (pin the algorithm!)
3. Recompute  sig' = HMAC-SHA256(secret, part0 + "." + part1)
4. Decode the presented signature (part2).
5. Compare sig' and the presented signature in CONSTANT TIME.   (mismatch → reject)
6. ONLY NOW decode the PAYLOAD (it's authenticated).
7. Check claims: sub present, exp present and in the future.     (expired → reject)
8. Success: return the claims.
```

Two orderings deserve emphasis:

- **The algorithm check (2) comes before trusting anything**, because the header is attacker-controlled.
- **The payload is decoded only after the signature verifies (6)**. Never act on unauthenticated data.

### What each attack targets

| Attack | What the attacker tries | Which step stops it |
|--------|------------------------|---------------------|
| **Tampering** | Edit the payload (`sub: 1`) and keep the old signature | 3–5: recomputed HMAC differs |
| **Forgery** | Invent a token without the secret | 3–5 |
| **`alg: none`** | Send `{"alg":"none"}` with an empty signature | 2: algorithm is not HS256 |
| **Algorithm confusion** | Claim `RS256` hoping the server uses a public key as an HMAC secret | 2 |
| **Replay of an expired token** | Re-use an old stolen token | 7: `exp` is in the past |
| **Malformed input** | Crash the parser with junk | 1, decode/JSON errors → reject cleanly |

---

## 5. Constant-time comparison

Comparing the expected and presented signatures looks like a job for `==`:

```go
if string(gotSig) == string(wantSig) { ... }    // ❌ DON'T
```

The problem: ordinary comparison **stops at the first differing byte**. An attacker who can measure response times *very* precisely could learn how many leading bytes of their forged signature were correct, then guess the rest byte by byte (a **timing attack**). Fixing it costs nothing:

```go
if hmac.Equal(gotSig, wantSig) { ... }          // ✅ takes the same time whatever the input
```

`hmac.Equal` (like `crypto/subtle.ConstantTimeCompare`) examines every byte regardless of where a mismatch occurs. **Rule: compare secrets, signatures, and password hashes only with constant-time functions.** (`bcrypt.CompareHashAndPassword`, used in Chapter 48, is already constant-time.)

---

## 6. `ParseToken`

We extend `auth/jwt.go` (Chapter 48's `CreateToken` stays exactly as it was). Complete file:

```go
// file: auth/jwt.go
package auth

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"errors"
	"fmt"
	"strconv"
	"strings"
	"time"
)

// Claims are the facts a token asserts about its holder.
type Claims struct {
	Subject   string `json:"sub"` // the user ID
	IssuedAt  int64  `json:"iat"` // Unix seconds
	ExpiresAt int64  `json:"exp"` // Unix seconds
}

// UserID converts the subject claim to the numeric user ID.
func (c Claims) UserID() (int, error) { return strconv.Atoi(c.Subject) }

var (
	// ErrInvalidToken means the token is malformed, forged, or otherwise untrustworthy.
	ErrInvalidToken = errors.New("invalid token")
	// ErrExpiredToken means the token was genuine but is past its expiry time.
	ErrExpiredToken = errors.New("token expired")
)

var (
	enc = base64.RawURLEncoding // Base64URL without padding, as JWT requires

	// The header never changes: HMAC-SHA256, type JWT. Pre-encode it once.
	encodedHeader = enc.EncodeToString([]byte(`{"alg":"HS256","typ":"JWT"}`))
)

// CreateToken issues a signed token for the user, valid for ttl from now.
// (now is a parameter so tests can control time.)
func CreateToken(secret []byte, userID int, ttl time.Duration, now time.Time) (string, error) {
	if len(secret) == 0 {
		return "", errors.New("auth: signing secret must not be empty")
	}

	claims := Claims{
		Subject:   strconv.Itoa(userID),
		IssuedAt:  now.Unix(),
		ExpiresAt: now.Add(ttl).Unix(),
	}
	payload, err := json.Marshal(claims)
	if err != nil {
		return "", err
	}

	signingInput := encodedHeader + "." + enc.EncodeToString(payload)
	signature := sign(secret, signingInput)

	return signingInput + "." + enc.EncodeToString(signature), nil
}

// ParseToken verifies a token and returns its claims.
// Errors wrap ErrInvalidToken (bad token) or ErrExpiredToken (genuine but too old).
func ParseToken(secret []byte, token string, now time.Time) (Claims, error) {
	if len(secret) == 0 {
		return Claims{}, errors.New("auth: verification secret must not be empty")
	}

	// 1. exactly three parts
	parts := strings.Split(token, ".")
	if len(parts) != 3 || parts[0] == "" || parts[1] == "" || parts[2] == "" {
		return Claims{}, fmt.Errorf("%w: expected three non-empty parts", ErrInvalidToken)
	}

	// 2. the header: insist on the one algorithm we use
	headerJSON, err := enc.DecodeString(parts[0])
	if err != nil {
		return Claims{}, fmt.Errorf("%w: header is not valid Base64URL", ErrInvalidToken)
	}
	var header struct {
		Alg string `json:"alg"`
		Typ string `json:"typ"`
	}
	if err := json.Unmarshal(headerJSON, &header); err != nil {
		return Claims{}, fmt.Errorf("%w: header is not valid JSON", ErrInvalidToken)
	}
	if header.Alg != "HS256" { // never trust the token to choose its own algorithm
		return Claims{}, fmt.Errorf("%w: unsupported algorithm %q", ErrInvalidToken, header.Alg)
	}

	// 3-5. recompute the signature and compare in constant time
	presented, err := enc.DecodeString(parts[2])
	if err != nil {
		return Claims{}, fmt.Errorf("%w: signature is not valid Base64URL", ErrInvalidToken)
	}
	expected := sign(secret, parts[0]+"."+parts[1])
	if !hmac.Equal(presented, expected) {
		return Claims{}, fmt.Errorf("%w: signature mismatch", ErrInvalidToken)
	}

	// 6. the token is authentic: now it is safe to read the payload
	payloadJSON, err := enc.DecodeString(parts[1])
	if err != nil {
		return Claims{}, fmt.Errorf("%w: payload is not valid Base64URL", ErrInvalidToken)
	}
	var claims Claims
	if err := json.Unmarshal(payloadJSON, &claims); err != nil {
		return Claims{}, fmt.Errorf("%w: payload is not valid JSON", ErrInvalidToken)
	}

	// 7. claims must make sense
	if claims.Subject == "" {
		return Claims{}, fmt.Errorf("%w: missing subject", ErrInvalidToken)
	}
	if claims.ExpiresAt == 0 {
		return Claims{}, fmt.Errorf("%w: missing expiry", ErrInvalidToken) // tokens must expire
	}
	if now.Unix() >= claims.ExpiresAt {
		return Claims{}, ErrExpiredToken
	}

	return claims, nil
}

// sign computes HMAC-SHA256(secret, input).
func sign(secret []byte, input string) []byte {
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(input))
	return mac.Sum(nil)
}
```

Details worth reading twice:

- **Wrapped sentinel errors.** `fmt.Errorf("%w: ...", ErrInvalidToken)` keeps the *specific reason* for logs while letting callers test `errors.Is(err, auth.ErrInvalidToken)` (Chapter 41's `errors.Is`). The middleware turns any invalid-token error into the same generic `401` for the client, and can log the precise reason.
- **Expiry uses `>=`.** A token expiring at second `T` is invalid *at* `T`.
- We **require** `exp`: a token with no expiry would live forever.
- `now` is a parameter, so tests can travel through time without `time.Sleep`.
- `sub` is validated as *present* here; converting to an integer is `Claims.UserID()`.

---

## 7. Carrying identity: context helpers

After authentication, handlers need to know **who** is calling. As in Chapter 46 (request IDs), the answer is `context.Context`. We put the accessors in the **`auth`** package rather than in the middleware package, so that *handlers* can read the user without importing *middleware*:

```
handlers ──► auth ◄── middleware        (both depend on auth; neither depends on the other)
```

```go
// file: auth/context.go
package auth

import "context"

// contextKey is private, so no other package can create a colliding key.
type contextKey struct{}

// ContextWithUserID returns a copy of ctx carrying the authenticated user's ID.
func ContextWithUserID(ctx context.Context, userID int) context.Context {
	return context.WithValue(ctx, contextKey{}, userID)
}

// UserIDFrom returns the authenticated user's ID, if the request passed authentication.
func UserIDFrom(ctx context.Context) (int, bool) {
	id, ok := ctx.Value(contextKey{}).(int)
	return id, ok
}
```

An empty `struct{}` as the key type is a common idiom: it takes no memory and, being unexported, is unforgeable.

---

## 8. The authentication middleware

```go
// file: middleware/auth.go
package middleware

import (
	"ecommerce/auth"
	"ecommerce/util"
	"errors"
	"net/http"
	"strings"
	"time"
)

// Authenticate returns a middleware that only lets requests with a valid Bearer token through.
// On success the user's ID is available to downstream handlers via auth.UserIDFrom(r.Context()).
func Authenticate(secret []byte) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			token, ok := bearerToken(r)
			if !ok {
				unauthorized(w, "missing or malformed Authorization header (use: Bearer YOUR_TOKEN)")
				return
			}

			claims, err := auth.ParseToken(secret, token, time.Now())
			if err != nil {
				if errors.Is(err, auth.ErrExpiredToken) {
					unauthorized(w, "token expired")
				} else {
					unauthorized(w, "invalid token")
				}
				return
			}

			userID, err := claims.UserID()
			if err != nil {
				unauthorized(w, "invalid token")
				return
			}

			ctx := auth.ContextWithUserID(r.Context(), userID)
			next.ServeHTTP(w, r.WithContext(ctx)) // hand the request, now carrying the identity, on
		})
	}
}

// bearerToken extracts the token from an "Authorization: Bearer <token>" header.
func bearerToken(r *http.Request) (string, bool) {
	scheme, token, found := strings.Cut(r.Header.Get("Authorization"), " ")
	if !found || !strings.EqualFold(scheme, "Bearer") {
		return "", false
	}
	token = strings.TrimSpace(token)
	if token == "" || strings.Contains(token, " ") {
		return "", false
	}
	return token, true
}

func unauthorized(w http.ResponseWriter, message string) {
	w.Header().Set("WWW-Authenticate", "Bearer") // tells clients which scheme to use
	util.SendError(w, http.StatusUnauthorized, message)
}
```

Notes:

- **`401` + `WWW-Authenticate: Bearer`** is the standard pairing: it tells the client *how* to authenticate.
- **`Authenticate(secret)` is a middleware factory** (like `Logger` and `CORS`): the secret is captured once, in a closure.
- The middleware **stops the chain** on failure by returning before `next.ServeHTTP` (Chapter 44).
- **Error messages**: we tell the client "token expired" (useful: it should log in again or refresh) but otherwise say only "invalid token". The precise reason (bad signature? wrong algorithm?) belongs in server logs, not in a response that helps an attacker learn what to adjust.
- On success we pass **`r.WithContext(ctx)`**, not `r` (Chapter 46, mistake #4).

---

## 9. Route-level protection

Which routes need a token?

| Route | Access |
|-------|--------|
| `GET /products`, `GET /products/{id}` | **public**: browse the shop |
| `POST /users`, `POST /login` | **public**: you can't have a token before logging in! |
| `POST /products` | **authenticated** |
| `GET /me` | **authenticated** |

Because only *some* routes are protected, `Authenticate` is **route-level middleware**: we wrap individual handlers with it using `mux.Handle` (not `HandleFunc`), as introduced in Chapter 46.

```go
authn := middleware.Authenticate([]byte(cfg.JWTSecret))

mux.HandleFunc("GET /products", handlers.GetProducts)                             // public
mux.Handle("POST /products", authn(http.HandlerFunc(handlers.CreateProduct)))     // protected
mux.Handle("GET /me", authn(http.HandlerFunc(users.Me)))                          // protected
```

`http.HandlerFunc(handlers.CreateProduct)` converts the function into an `http.Handler` (which `authn` requires), and `authn(...)` wraps it.

### Why not make authentication global?

Suppose we tried `global.Then(mux)` with `Authenticate` in the global stack:

1. **Public routes break**: `/login` would demand a token, but you get a token *from* `/login`. A chicken-and-egg deadlock.
2. **Preflight `OPTIONS` requests break**: browsers send preflights **without** credentials (Chapter 42). Global auth would answer them `401`, and every cross-origin request would be blocked by the browser.

Our global `CORS` already runs *outside* everything else and answers preflights itself, and per-route `Authenticate` runs only *inside* the mux after routing: the ordering problem simply doesn't arise. (If you did put auth in the global stack, it would have to go *inside* `CORS`, and you'd need an allow-list of public paths.)

### The full execution order for `POST /products`

```
RequestID → Logger → CORS → Recover → JSONErrors → mux (routing) → Authenticate → CreateProduct
```

Everything before the mux is global; `Authenticate` is the route-specific layer closest to the handler.

---

## 10. A protected endpoint

To show identity flowing from the middleware into a handler, add `GET /me`: "who am I?" It needs a way to find a user by ID:

```go
// file: database/users.go
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

var (
	userMu     sync.RWMutex
	users      []models.User
	nextUserID = 1
)

func normalizeEmail(email string) string { return strings.ToLower(strings.TrimSpace(email)) }

// ResetUsers empties the user table. (Useful for tests.)
func ResetUsers() {
	userMu.Lock()
	defer userMu.Unlock()
	users = nil
	nextUserID = 1
}

// CreateUser stores a new user and returns it. The check-then-insert happens under one lock,
// so two simultaneous registrations for the same email cannot both succeed.
func CreateUser(email, passwordHash string) (models.User, error) {
	email = normalizeEmail(email)

	userMu.Lock()
	defer userMu.Unlock()

	for _, u := range users {
		if u.Email == email {
			return models.User{}, ErrEmailTaken
		}
	}
	u := models.User{ID: nextUserID, Email: email, PasswordHash: passwordHash, CreatedAt: time.Now().UTC()}
	nextUserID++
	users = append(users, u)
	return u, nil
}

// FindUserByEmail looks a user up by (case-insensitive) email.
func FindUserByEmail(email string) (models.User, bool) {
	email = normalizeEmail(email)

	userMu.RLock()
	defer userMu.RUnlock()
	for _, u := range users {
		if u.Email == email {
			return u, true
		}
	}
	return models.User{}, false
}

// FindUserByID looks a user up by ID.
func FindUserByID(id int) (models.User, bool) {
	userMu.RLock()
	defer userMu.RUnlock()
	for _, u := range users {
		if u.ID == id {
			return u, true
		}
	}
	return models.User{}, false
}
```

(`FindUserByID` is the only addition to Chapter 48's file.) And the handler, added to `handlers/user.go`. Here is the complete updated file:

```go
// file: handlers/user.go
package handlers

import (
	"ecommerce/auth"
	"ecommerce/config"
	"ecommerce/database"
	"ecommerce/util"
	"errors"
	"net/http"
	"net/mail"
	"time"
)

// UserHandler serves registration, login, and the current-user endpoint.
type UserHandler struct {
	jwtSecret []byte
	jwtTTL    time.Duration
}

// NewUserHandler builds a UserHandler from the application configuration.
func NewUserHandler(cfg *config.Config) *UserHandler {
	return &UserHandler{jwtSecret: []byte(cfg.JWTSecret), jwtTTL: cfg.JWTTTL}
}

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

// CreateUser handles POST /users (registration).
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
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

	user, err := database.CreateUser(req.Email, hash)
	if errors.Is(err, database.ErrEmailTaken) {
		util.SendError(w, http.StatusConflict, "email already registered")
		return
	}
	if err != nil {
		util.SendError(w, http.StatusInternalServerError, "could not create the account")
		return
	}

	util.SendData(w, http.StatusCreated, user)
}

// dummyHash lets Login spend the same time on unknown emails as on known ones.
var dummyHash, _ = auth.HashPassword("dummy-password-for-timing")

// Login handles POST /login: verifies credentials and returns a signed token.
func (h *UserHandler) Login(w http.ResponseWriter, r *http.Request) {
	var req credentials
	if !util.DecodeJSON(w, r, &req) {
		return
	}

	user, found := database.FindUserByEmail(req.Email)

	hash := dummyHash
	if found {
		hash = user.PasswordHash
	}
	passwordOK := auth.CheckPassword(hash, req.Password)

	if !found || !passwordOK {
		util.SendError(w, http.StatusUnauthorized, "invalid email or password")
		return
	}

	token, err := auth.CreateToken(h.jwtSecret, user.ID, h.jwtTTL, time.Now())
	if err != nil {
		util.SendError(w, http.StatusInternalServerError, "could not create a token")
		return
	}

	util.SendData(w, http.StatusOK, map[string]string{"token": token})
}

// Me handles GET /me: it returns the authenticated user. The route must be wrapped
// in the Authenticate middleware, which is what puts the user ID into the context.
func (h *UserHandler) Me(w http.ResponseWriter, r *http.Request) {
	id, ok := auth.UserIDFrom(r.Context())
	if !ok { // can only happen if the route was registered without Authenticate: a programming error
		util.SendError(w, http.StatusUnauthorized, "authentication required")
		return
	}

	user, found := database.FindUserByID(id)
	if !found { // valid token for a user that no longer exists
		util.SendError(w, http.StatusUnauthorized, "account not found")
		return
	}
	util.SendData(w, http.StatusOK, user)
}
```

Notice that `Me` never touches a token, a header, or the secret: **by the time a handler runs, authentication is already done**, and the handler just asks the context "who is this?". That's the separation of concerns middleware buys.

---

## 11. The complete code

Files changed this chapter: `auth/jwt.go` (adds `ParseToken`), `auth/context.go` (new), `middleware/auth.go` (new), `database/users.go` (adds `FindUserByID`), `handlers/user.go` (adds `Me`), `rest/router.go`, and the router tests.

```go
// file: rest/router.go
package rest

import (
	"ecommerce/config"
	"ecommerce/handlers"
	"ecommerce/middleware"
	"log/slog"
	"net/http"
)

// NewRouter builds the complete HTTP handler for the app: routes wrapped in the middleware pipeline.
func NewRouter(logger *slog.Logger, cfg *config.Config) http.Handler {
	mux := http.NewServeMux()

	users := handlers.NewUserHandler(cfg)
	authn := middleware.Authenticate([]byte(cfg.JWTSecret)) // route-level middleware

	// Public
	mux.HandleFunc("GET /products", handlers.GetProducts)
	mux.HandleFunc("GET /products/{id}", handlers.GetProduct)
	mux.HandleFunc("POST /users", users.CreateUser)
	mux.HandleFunc("POST /login", users.Login)

	// Requires a valid token
	mux.Handle("POST /products", authn(http.HandlerFunc(handlers.CreateProduct)))
	mux.Handle("GET /me", authn(http.HandlerFunc(users.Me)))

	global := middleware.NewStack(
		middleware.RequestID,
		middleware.Logger(logger),
		middleware.CORS(cfg.AllowedOrigins),
		middleware.Recover(logger),
		middleware.JSONErrors,
	)
	return global.Then(mux)
}
```

Because creating products now needs a token, the Chapter 47 router tests must send one. The test config gets a secret, and a helper mints a valid token. Complete updated file:

```go
// file: rest/router_test.go
package rest

import (
	"ecommerce/auth"
	"ecommerce/config"
	"ecommerce/database"
	"encoding/json"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"sync"
	"testing"
	"time"
)

var testConfig = &config.Config{
	Env:            "test",
	Port:           "0",
	AllowedOrigins: []string{"http://localhost:5173"},
	LogFormat:      "text",
	JWTSecret:      "0123456789abcdef0123456789abcdef0123",
	JWTTTL:         time.Hour,
}

var quietLogger = slog.New(slog.NewTextHandler(io.Discard, nil))

type product struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
}

func do(method, target, body string, headers map[string]string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	if body != "" {
		req.Header.Set("Content-Type", "application/json")
	}
	for k, v := range headers {
		req.Header.Set(k, v)
	}
	rec := httptest.NewRecorder()
	NewRouter(quietLogger, testConfig).ServeHTTP(rec, req)
	return rec
}

// bearer returns headers carrying a valid token for the given user.
func bearer(t testing.TB, userID int) map[string]string {
	t.Helper()
	token, err := auth.CreateToken([]byte(testConfig.JWTSecret), userID, time.Hour, time.Now())
	if err != nil {
		t.Fatal(err)
	}
	return map[string]string{"Authorization": "Bearer " + token}
}

func TestListAndGet(t *testing.T) {
	database.ResetProducts()

	var list []product
	rec := do(http.MethodGet, "/products", "", nil) // public: no token needed
	if rec.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rec.Code)
	}
	json.NewDecoder(rec.Body).Decode(&list)
	if len(list) != 3 {
		t.Fatalf("expected 3 products, got %d", len(list))
	}

	for path, want := range map[string]int{"/products/2": 200, "/products/99": 404, "/products/abc": 400} {
		if got := do(http.MethodGet, path, "", nil).Code; got != want {
			t.Errorf("GET %s: expected %d, got %d", path, want, got)
		}
	}
}

func TestCreateProductRequiresAToken(t *testing.T) {
	database.ResetProducts()
	body := `{"title":"Mango","description":"d","price":200,"imageUrl":"u"}`

	// no header at all
	rec := do(http.MethodPost, "/products", body, nil)
	if rec.Code != http.StatusUnauthorized {
		t.Fatalf("expected 401 without a token, got %d", rec.Code)
	}
	if rec.Header().Get("WWW-Authenticate") != "Bearer" {
		t.Errorf("WWW-Authenticate = %q", rec.Header().Get("WWW-Authenticate"))
	}

	// nothing was created
	var list []product
	json.NewDecoder(do(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	if len(list) != 3 {
		t.Errorf("an unauthenticated request created a product (%d products)", len(list))
	}

	// a garbage token
	rec = do(http.MethodPost, "/products", body, map[string]string{"Authorization": "Bearer not.a.token"})
	if rec.Code != http.StatusUnauthorized {
		t.Errorf("expected 401 for a garbage token, got %d", rec.Code)
	}
}

func TestCreateProductWithAToken(t *testing.T) {
	database.ResetProducts()
	rec := do(http.MethodPost, "/products", `{"title":"Mango","description":"d","price":200,"imageUrl":"u"}`, bearer(t, 1))
	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d (%s)", rec.Code, rec.Body.String())
	}
	if loc := rec.Header().Get("Location"); loc != "/products/4" {
		t.Errorf("Location = %q", loc)
	}
	if rec := do(http.MethodPost, "/products", `{"title":"","price":5}`, bearer(t, 1)); rec.Code != http.StatusUnprocessableEntity {
		t.Errorf("validation must still work behind auth: expected 422, got %d", rec.Code)
	}
}

func TestPipelineErrorsAreJSONWithCORSAndRequestID(t *testing.T) {
	origin := map[string]string{"Origin": "http://localhost:5173"}

	rec := do(http.MethodGet, "/no-such-route", "", origin)
	if rec.Code != 404 || !strings.Contains(rec.Body.String(), `{"error":"route not found"}`) {
		t.Errorf("404: code=%d body=%q", rec.Code, rec.Body.String())
	}
	if rec.Header().Get("Access-Control-Allow-Origin") != "http://localhost:5173" {
		t.Error("404 is missing CORS headers")
	}
	if len(rec.Header().Get("X-Request-ID")) != 16 {
		t.Errorf("missing/odd X-Request-ID: %q", rec.Header().Get("X-Request-ID"))
	}

	rec = do(http.MethodDelete, "/products", "", nil)
	if rec.Code != 405 || rec.Header().Get("Allow") != "GET, HEAD, POST" {
		t.Errorf("405: code=%d allow=%q", rec.Code, rec.Header().Get("Allow"))
	}
}

func TestCORSUsesConfiguredOrigins(t *testing.T) {
	preflight := func(origin string) *httptest.ResponseRecorder {
		return do(http.MethodOptions, "/products", "", map[string]string{
			"Origin":                         origin,
			"Access-Control-Request-Method":  "POST",
			"Access-Control-Request-Headers": "content-type,authorization",
		})
	}

	// The preflight carries NO credentials, yet must succeed for a protected route.
	if rec := preflight("http://localhost:5173"); rec.Code != http.StatusNoContent ||
		rec.Header().Get("Access-Control-Allow-Origin") != "http://localhost:5173" ||
		!strings.Contains(rec.Header().Get("Access-Control-Allow-Headers"), "Authorization") {
		t.Errorf("preflight for a protected route failed: code=%d headers=%v", rec.Code, rec.Header())
	}
	if rec := preflight("https://evil.example"); rec.Header().Get("Access-Control-Allow-Origin") != "" {
		t.Error("unconfigured origin must not be allowed")
	}
}

func TestConcurrentCreates(t *testing.T) {
	database.ResetProducts()
	headers := bearer(t, 1)

	const n = 50
	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			do(http.MethodPost, "/products", `{"title":"Concurrent","price":1}`, headers)
		}()
	}
	wg.Wait()

	var list []product
	json.NewDecoder(do(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	if len(list) != 3+n {
		t.Fatalf("expected %d products, got %d", 3+n, len(list))
	}
}
```

---

## 12. Tests

A security feature deserves **adversarial** tests: try to break it.

### `ParseToken`

```go
// file: auth/parse_test.go
package auth

import (
	"encoding/json"
	"errors"
	"strings"
	"testing"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

var now = time.Date(2026, 9, 26, 12, 0, 0, 0, time.UTC)

func mustToken(t *testing.T, userID int, ttl time.Duration, at time.Time) string {
	t.Helper()
	tok, err := CreateToken(secret, userID, ttl, at) // secret is defined in auth_test.go
	if err != nil {
		t.Fatal(err)
	}
	return tok
}

func TestParseTokenAcceptsAFreshToken(t *testing.T) {
	claims, err := ParseToken(secret, mustToken(t, 42, time.Hour, now), now.Add(time.Minute))
	if err != nil {
		t.Fatal(err)
	}
	id, err := claims.UserID()
	if err != nil || id != 42 {
		t.Errorf("UserID = %d, %v; want 42", id, err)
	}
}

func TestParseTokenExpiry(t *testing.T) {
	tok := mustToken(t, 42, time.Hour, now)

	if _, err := ParseToken(secret, tok, now.Add(59*time.Minute)); err != nil {
		t.Errorf("token should still be valid after 59 minutes: %v", err)
	}
	_, err := ParseToken(secret, tok, now.Add(time.Hour)) // exactly at exp
	if !errors.Is(err, ErrExpiredToken) {
		t.Errorf("token must be expired AT its exp time, got %v", err)
	}
	_, err = ParseToken(secret, tok, now.Add(48*time.Hour))
	if !errors.Is(err, ErrExpiredToken) {
		t.Errorf("expected ErrExpiredToken, got %v", err)
	}
}

func TestParseTokenRejectsTamperedPayload(t *testing.T) {
	tok := mustToken(t, 42, time.Hour, now)
	parts := strings.Split(tok, ".")

	// An attacker changes sub to "1" (an admin, say) and keeps the old signature.
	forged, _ := json.Marshal(Claims{Subject: "1", IssuedAt: now.Unix(), ExpiresAt: now.Add(time.Hour).Unix()})
	tampered := parts[0] + "." + enc.EncodeToString(forged) + "." + parts[2]

	_, err := ParseToken(secret, tampered, now)
	if !errors.Is(err, ErrInvalidToken) {
		t.Fatalf("a tampered token must be rejected, got %v", err)
	}
}

func TestParseTokenRejectsTamperedSignature(t *testing.T) {
	tok := mustToken(t, 42, time.Hour, now)
	flipped := tok[:len(tok)-2] + "AA" // damage the end of the signature
	if flipped == tok {
		flipped = tok[:len(tok)-2] + "BB"
	}
	if _, err := ParseToken(secret, flipped, now); !errors.Is(err, ErrInvalidToken) {
		t.Errorf("expected ErrInvalidToken, got %v", err)
	}
}

func TestParseTokenRejectsTheWrongSecret(t *testing.T) {
	tok := mustToken(t, 42, time.Hour, now)
	if _, err := ParseToken([]byte("some-other-secret-with-32-plus-chars!!"), tok, now); !errors.Is(err, ErrInvalidToken) {
		t.Errorf("expected ErrInvalidToken, got %v", err)
	}
}

func TestParseTokenRejectsAlgNone(t *testing.T) {
	// The classic attack: {"alg":"none"} and an empty signature.
	header := enc.EncodeToString([]byte(`{"alg":"none","typ":"JWT"}`))
	payload := enc.EncodeToString([]byte(`{"sub":"1","iat":1,"exp":9999999999}`))

	for _, tok := range []string{header + "." + payload + ".", header + "." + payload + ".AAAA"} {
		if _, err := ParseToken(secret, tok, now); !errors.Is(err, ErrInvalidToken) {
			t.Errorf("alg=none token %q was not rejected: %v", tok, err)
		}
	}
}

func TestParseTokenRejectsOtherAlgorithms(t *testing.T) {
	header := enc.EncodeToString([]byte(`{"alg":"RS256","typ":"JWT"}`))
	payload := enc.EncodeToString([]byte(`{"sub":"1","iat":1,"exp":9999999999}`))
	// even with a signature that IS a valid HMAC of this header/payload, the algorithm is not ours
	sig := enc.EncodeToString(sign(secret, header+"."+payload))
	if _, err := ParseToken(secret, header+"."+payload+"."+sig, now); !errors.Is(err, ErrInvalidToken) {
		t.Errorf("expected the algorithm to be rejected, got %v", err)
	}
}

func TestParseTokenRejectsMalformedInput(t *testing.T) {
	good := mustToken(t, 42, time.Hour, now)
	parts := strings.Split(good, ".")

	bad := map[string]string{
		"empty":             "",
		"one part":          "abc",
		"two parts":         parts[0] + "." + parts[1],
		"four parts":        good + ".extra",
		"empty signature":   parts[0] + "." + parts[1] + ".",
		"not base64 header": "!!!." + parts[1] + "." + parts[2],
		"not base64 sig":    parts[0] + "." + parts[1] + ".***",
		"padded signature":  good + "=",
		"whitespace":        " " + good,
		"bearer prefix":     "Bearer " + good,
		"header not json":   enc.EncodeToString([]byte("nope")) + "." + parts[1] + "." + parts[2],
		"just dots":         "..",
		"unicode junk":      "héllo.wörld.✓",
	}
	for name, tok := range bad {
		t.Run(name, func(t *testing.T) {
			if _, err := ParseToken(secret, tok, now); !errors.Is(err, ErrInvalidToken) {
				t.Errorf("expected ErrInvalidToken for %q, got %v", tok, err)
			}
		})
	}
}

func TestParseTokenRequiresSubjectAndExpiry(t *testing.T) {
	// mint builds a correctly SIGNED token around an arbitrary payload, so only the claims are wrong.
	mint := func(payload string) string {
		p := enc.EncodeToString([]byte(payload))
		return encodedHeader + "." + p + "." + enc.EncodeToString(sign(secret, encodedHeader+"."+p))
	}
	for name, payload := range map[string]string{
		"no subject": `{"iat":1,"exp":9999999999}`,
		"no expiry":  `{"sub":"1","iat":1}`,
		"not json":   `[1,2,3]`,
	} {
		if _, err := ParseToken(secret, mint(payload), now); !errors.Is(err, ErrInvalidToken) {
			t.Errorf("%s: expected ErrInvalidToken, got %v", name, err)
		}
	}
}

// Interoperability in the other direction: the reference library mints, we verify.
func TestWeAcceptTokensFromTheReferenceLibrary(t *testing.T) {
	lib := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
		"sub": "77",
		"iat": now.Unix(),
		"exp": now.Add(time.Hour).Unix(),
	})
	tok, err := lib.SignedString(secret)
	if err != nil {
		t.Fatal(err)
	}

	claims, err := ParseToken(secret, tok, now.Add(time.Minute))
	if err != nil {
		t.Fatalf("we rejected a valid golang-jwt token: %v", err)
	}
	if id, _ := claims.UserID(); id != 77 {
		t.Errorf("UserID = %d, want 77", id)
	}
}

func TestParseTokenNeedsASecret(t *testing.T) {
	if _, err := ParseToken(nil, "a.b.c", now); err == nil {
		t.Error("expected an error for an empty secret")
	}
}
```

The suite is a catalogue of real attacks: **tampering, forgery, `alg: none`, algorithm confusion, expiry, malformed input, missing claims**, plus **two-way interoperability** with the reference library.

### The middleware

```go
// file: middleware/auth_test.go
package middleware

import (
	"ecommerce/auth"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

var secret = []byte("0123456789abcdef0123456789abcdef0123")

func serve(t *testing.T, header string) (*httptest.ResponseRecorder, bool, int) {
	t.Helper()
	called := false
	var gotID int
	h := Authenticate(secret)(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		called = true
		gotID, _ = auth.UserIDFrom(r.Context())
		w.WriteHeader(http.StatusOK)
	}))

	req := httptest.NewRequest(http.MethodGet, "/protected", nil)
	if header != "" {
		req.Header.Set("Authorization", header)
	}
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, req)
	return rec, called, gotID
}

func TestAuthenticateLetsValidTokensThrough(t *testing.T) {
	tok, _ := auth.CreateToken(secret, 42, time.Hour, time.Now())

	rec, called, id := serve(t, "Bearer "+tok)
	if rec.Code != http.StatusOK || !called {
		t.Fatalf("expected the handler to run, got %d called=%v", rec.Code, called)
	}
	if id != 42 {
		t.Errorf("the handler saw user %d, want 42", id)
	}
}

func TestAuthenticateSchemeIsCaseInsensitive(t *testing.T) {
	tok, _ := auth.CreateToken(secret, 1, time.Hour, time.Now())
	if rec, called, _ := serve(t, "bearer "+tok); rec.Code != http.StatusOK || !called {
		t.Errorf("lowercase scheme rejected: %d", rec.Code)
	}
}

func TestAuthenticateRejectsBadRequests(t *testing.T) {
	valid, _ := auth.CreateToken(secret, 1, time.Hour, time.Now())
	expired, _ := auth.CreateToken(secret, 1, time.Hour, time.Now().Add(-2*time.Hour))
	otherSecret, _ := auth.CreateToken([]byte("a-completely-different-secret-value-32"), 1, time.Hour, time.Now())

	tests := []struct {
		name, header, wantMsg string
	}{
		{"no header", "", "missing or malformed"},
		{"wrong scheme", "Basic dXNlcjpwYXNz", "missing or malformed"},
		{"no token", "Bearer", "missing or malformed"},
		{"empty token", "Bearer  ", "missing or malformed"},
		{"extra parts", "Bearer " + valid + " extra", "missing or malformed"},
		{"garbage token", "Bearer abc.def.ghi", "invalid token"},
		{"wrong secret", "Bearer " + otherSecret, "invalid token"},
		{"expired", "Bearer " + expired, "token expired"},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			rec, called, _ := serve(t, tc.header)
			if rec.Code != http.StatusUnauthorized {
				t.Errorf("expected 401, got %d", rec.Code)
			}
			if called {
				t.Error("the protected handler must not run")
			}
			if rec.Header().Get("WWW-Authenticate") != "Bearer" {
				t.Errorf("WWW-Authenticate = %q", rec.Header().Get("WWW-Authenticate"))
			}
			if !strings.Contains(rec.Body.String(), tc.wantMsg) {
				t.Errorf("body %q should contain %q", rec.Body.String(), tc.wantMsg)
			}
			if strings.Contains(rec.Body.String(), "signature") {
				t.Error("the response must not reveal why verification failed")
			}
		})
	}
}
```

### The endpoints

```go
// file: rest/me_test.go
package rest

import (
	"ecommerce/database"
	"encoding/json"
	"net/http"
	"strings"
	"testing"
)

func TestFullAuthenticationFlow(t *testing.T) {
	database.ResetUsers()
	database.ResetProducts()

	// register → login → use the token
	if rec := doAuth(http.MethodPost, "/users", `{"email":"asha@example.com","password":"correct-horse-battery"}`); rec.Code != http.StatusCreated {
		t.Fatalf("register: %d %s", rec.Code, rec.Body.String())
	}
	rec := doAuth(http.MethodPost, "/login", `{"email":"asha@example.com","password":"correct-horse-battery"}`)
	var resp struct {
		Token string `json:"token"`
	}
	json.NewDecoder(rec.Body).Decode(&resp)
	if resp.Token == "" {
		t.Fatalf("login gave no token: %s", rec.Body.String())
	}
	authz := "Bearer " + resp.Token

	me := func(header string) (int, string) {
		req := newRequest(http.MethodGet, "/me", "", header)
		w := serveWith(authConfig, req)
		return w.Code, w.Body.String()
	}

	if code, body := me(authz); code != http.StatusOK || !strings.Contains(body, `"email":"asha@example.com"`) {
		t.Errorf("GET /me with a valid token: %d %s", code, body)
	}
	if code, _ := me(""); code != http.StatusUnauthorized {
		t.Errorf("GET /me without a token: expected 401, got %d", code)
	}

	// a token issued for a user who doesn't exist (e.g. deleted since) is refused
	database.ResetUsers()
	if code, _ := me(authz); code != http.StatusUnauthorized {
		t.Errorf("GET /me for a vanished user: expected 401, got %d", code)
	}
}
```

(This test uses two small helpers, `newRequest` and `serveWith`, that we add next to the existing helpers in `rest/users_test.go`:)

```go
// file: rest/helpers_test.go
package rest

import (
	"ecommerce/config"
	"net/http"
	"net/http/httptest"
	"strings"
)

func newRequest(method, target, body, authorization string) *http.Request {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	if body != "" {
		req.Header.Set("Content-Type", "application/json")
	}
	if authorization != "" {
		req.Header.Set("Authorization", authorization)
	}
	return req
}

func serveWith(cfg *config.Config, req *http.Request) *httptest.ResponseRecorder {
	rec := httptest.NewRecorder()
	NewRouter(quietLogger, cfg).ServeHTTP(rec, req)
	return rec
}
```

Run everything:

```bash
go vet ./... && go test -race ./...
```

---

## 13. Trying it out

```bash
go run .
```

**1. Protected route without a token:**

```bash
curl -si -X POST localhost:8080/products -H 'Content-Type: application/json' -d '{"title":"Mango","price":200}'
```

```
HTTP/1.1 401 Unauthorized
Www-Authenticate: Bearer
Content-Type: application/json
...
{"error":"missing or malformed Authorization header (use: Bearer YOUR_TOKEN)"}
```

**2. Register and log in, saving the token in a shell variable:**

```bash
curl -s -X POST localhost:8080/users -H 'Content-Type: application/json' \
     -d '{"email":"asha@example.com","password":"correct-horse-battery"}' > /dev/null

TOKEN=$(curl -s -X POST localhost:8080/login -H 'Content-Type: application/json' \
     -d '{"email":"asha@example.com","password":"correct-horse-battery"}' | python3 -c 'import sys,json; print(json.load(sys.stdin)["token"])')
```

**3. Use it:**

```bash
curl -si -X POST localhost:8080/products \
     -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
     -d '{"title":"Mango","description":"King of fruits","price":200,"imageUrl":"https://example.com/mango.jpg"}'
# HTTP/1.1 201 Created  ...

curl -s localhost:8080/me -H "Authorization: Bearer $TOKEN"
# {"id":1,"email":"asha@example.com","createdAt":"..."}
```

**4. Tamper with it.** Change one character in the payload section of the token and retry:

```bash
BAD="${TOKEN%%.*}.eyJzdWIiOiIxIiwiaWF0IjoxNzkwMzgwODAwLCJleHAiOjk5OTk5OTk5OTl9.${TOKEN##*.}"
curl -si localhost:8080/me -H "Authorization: Bearer $BAD" | sed -n '1p;$p'
```

```
HTTP/1.1 401 Unauthorized
{"error":"invalid token"}
```

(The payload now claims a far-future expiry, but the old signature no longer matches, so the token is rejected.)

**5. Browser check.** From a page on `http://localhost:5173`, `fetch("http://localhost:8080/products", {method:"POST", headers:{Authorization:"Bearer ...", "Content-Type":"application/json"}, ...})` triggers a preflight with `Access-Control-Request-Headers: authorization,content-type`. Our CORS middleware (global, outside auth) answers it without a token, then the real request carries the token. Two layers cooperating exactly as designed.

---

## 14. The passport analogy

| Real-world passport | JWT system |
|---------------------|-----------|
| Issued once by a trusted authority (the passport office) | Issued once at **login** by the server |
| Contains your name, photo, number (claims) | Contains **claims**: `sub`, `iat`, `exp` |
| Has an **expiry date** | `exp` claim |
| Hard to forge: holograms, watermarks, official seal | The **HMAC signature** with the server's secret |
| Border officers **inspect** it; they don't phone the passport office each time | Any server instance **verifies the signature** locally; no database lookup |
| You show it at every border | The client sends `Authorization: Bearer <token>` with every request |
| If someone steals it, they can pretend to be you until it expires or is cancelled | A **stolen token works** until it expires: protect tokens, keep them short-lived |
| Only the government can issue valid ones | Only holders of the **secret** can create valid tokens |
| Entering a country ≠ getting into the VIP lounge | **Authentication** (valid passport) ≠ **authorization** (what you may do) |

Can someone fake it? Only by knowing the secret, or by breaking HMAC-SHA256 (infeasible). **Who can validate it?** Any server that holds the same secret. The client can't (it doesn't have the secret), and doesn't need to: it just carries the token.

---

## 15. Real-world issues

Our system is sound for learning and small projects. Production systems face further questions:

**Where does the browser store the token?**

| Storage | Pros | Cons |
|---------|------|------|
| `localStorage` / `sessionStorage` | Easy | Readable by any JavaScript on the page → **XSS** can steal it |
| **`HttpOnly` cookie** | JavaScript can't read it | Sent automatically → need **CSRF** protection (`SameSite` cookies, CSRF tokens) |
| In memory (JS variable) | Not persisted | Lost on refresh; XSS still a risk |

There's no free lunch: the choice is between XSS exposure and CSRF exposure, and both need mitigations (Content-Security-Policy, `SameSite=Lax/Strict`, careful dependencies).

**Logout and revocation.** A JWT is stateless, so the server can't "delete" one. Options: short-lived **access tokens** (5–15 min) + longer-lived **refresh tokens** stored server-side (revocable); a **denylist** of revoked token IDs (`jti`); or a per-user `tokenVersion`/`passwordChangedAt` compared to `iat`.

**Key rotation.** Changing the secret invalidates every token. Real systems support multiple keys (a `kid` header) so they can rotate gradually.

**Asymmetric algorithms (RS256/ES256).** With HS256 every verifier holds the *signing* secret. With public-key signatures, only the issuer holds the private key; other services verify with the public key. Essential when many services (or third parties) must verify tokens.

**Don't roll your own in production.** We built this by hand to *understand* it and cross-checked it against `golang-jwt`. For production code, prefer a maintained library and keep our lessons: pin algorithms, validate `exp`, protect secrets. Same for OAuth2/OIDC: use an identity provider.

**HTTPS everywhere.** Bearer tokens and passwords are readable to anyone who can see the traffic on plain HTTP.

---

## 16. Problems we still have

The auth feature works, but the project has structural weaknesses that will get worse as it grows. Naming them motivates the next chapters:

| Problem | Why it hurts | Fixed in |
|---------|--------------|----------|
| **No real database**: users and products vanish on restart | Data loss; can't scale to several instances | Chapters 52–58 |
| **Handlers reach into `database` package globals** | Can't test handlers with fake storage or swap PostgreSQL in without editing handlers: tight coupling | Chapters 50–51 |
| **Mixed handlers in one package** (`user`, `product` code side by side) | As features multiply, `handlers/` becomes a junk drawer | Chapters 50, 59–61 |
| **Everything depends on `config`/`database` concretions** | Hard to change or mock | Chapter 51 (interfaces) |
| **No roles/authorization**: any logged-in user can create products | Should be staff only | later |
| **No rate limiting on login** | Brute-force attacks | exercises |

---

## 17. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Comparing signatures with `==` | Timing attack | `hmac.Equal` |
| 2 | Trusting the header's `alg` | `alg: none` / confusion attacks | Pin the algorithm |
| 3 | Decoding/using the payload before verifying the signature | Acting on forged data | Verify first |
| 4 | Not checking `exp` | Tokens valid forever | Require and enforce `exp` |
| 5 | Global auth middleware covering `/login` and preflights | Deadlock; CORS breaks | Route-level auth; CORS outermost |
| 6 | Passing `r` instead of `r.WithContext(ctx)` to `next` | Handlers can't see the user | Pass the new request |
| 7 | Detailed verification errors in responses | Helps attackers | Generic "invalid token"; details in logs |
| 8 | `401` without `WWW-Authenticate` | Non-standard | Add `WWW-Authenticate: Bearer` |
| 9 | Returning `403` for "not logged in" | Wrong semantics | `401` = unauthenticated, `403` = forbidden |
| 10 | Putting tokens in URLs (`?token=`) | Ends up in logs/history/referrers | Use the header |
| 11 | Logging the `Authorization` header | Token leakage | Never log it |
| 12 | Trusting a valid token for a deleted user | Ghost accounts act | Look up the user (or use short expiry) |
| 13 | Long-lived tokens with no revocation plan | Stolen tokens live for months | Short access tokens + refresh tokens |
| 14 | Forgetting HTTPS | Credential sniffing | TLS termination in production |

---

## 18. Exercises

### Exercise 1: Read the flow
For `GET /me` with a token whose `exp` was 5 minutes ago, list which middleware run, in order, and the final status and message.

<details><summary>Solution</summary>

`RequestID → Logger → CORS → Recover → JSONErrors → mux → Authenticate`. `Authenticate` extracts the token, `ParseToken` returns `ErrExpiredToken`, and the middleware responds `401 {"error":"token expired"}` with `WWW-Authenticate: Bearer`; the handler never runs. `Logger` records `status=401` at WARN level.
</details>

### Exercise 2: Leeway for clock skew
Servers' clocks differ by a little. Add a 30-second leeway to expiry checks (a token counts as expired only 30 s after `exp`). Where does the change go, and what test do you add?

<details><summary>Solution</summary>

In `ParseToken`: `if now.Unix() >= claims.ExpiresAt+30 { return ErrExpiredToken }` (define `const leewaySeconds = 30`). Update the exact-expiry test to `now.Add(time.Hour + 31*time.Second)`, and add a test that a token 10 seconds past `exp` is still accepted.
</details>

### Exercise 3: Roles and a `RequireRole` middleware
Add a `role` claim (`"customer"`/`"admin"`), and write `RequireRole(role string) Middleware` that returns `403` unless the authenticated user has that role. Which context helper do you extend?

<details><summary>Solution</summary>

Add `Role string \`json:"role"\`` to `Claims`; put it in the context (`auth.ContextWithClaims(ctx, claims)` / `auth.ClaimsFrom(ctx)`); `RequireRole` reads it and answers `403 forbidden` when it doesn't match. Compose as `authn(RequireRole("admin")(handler))`. Because roles can change after a token is issued, re-check the database for sensitive operations.
</details>

### Exercise 4: Refresh tokens (design)
Sketch a refresh-token flow: what does `/login` return, what does `/refresh` accept, what's stored server-side, and how does logout work?

<details><summary>Solution</summary>

`/login` returns a short-lived access JWT (15 min) and a long random refresh token (stored server-side as a hash with the user ID and expiry). `/refresh` accepts the refresh token, checks it exists and is unexpired, **rotates** it (issues a new one, deletes the old), and returns a new access token. Logout deletes the refresh token, so no new access tokens can be minted; existing access tokens die within 15 minutes.
</details>

### Exercise 5: Test the attack yourself
Using `curl`, decode a token's payload, change `exp` to a huge value, re-encode it (`base64url`, no padding), reattach the *old* signature, and send it to `/me`. What response do you get, and which step of verification stopped you?

<details><summary>Solution</summary>

`401 invalid token`. Step 3–5: the recomputed HMAC over `header.newPayload` doesn't equal the old signature (`signature mismatch` in the server-side error). Without the secret you can't produce the right one.
</details>

### Exercise 6: Log the reason, not to the client
Modify `Authenticate` so the precise `ParseToken` error is logged (at DEBUG or WARN) with the request ID, while the client still only sees the generic message. What must be injected?

<details><summary>Solution</summary>

Give the factory a logger: `Authenticate(secret []byte, logger *slog.Logger)`, and in the failure branch: `logger.WarnContext(r.Context(), "authentication failed", "reason", err.Error(), "request_id", RequestIDFrom(r.Context()), "remote", r.RemoteAddr)`. Update `rest/router.go` to pass the logger. Never log the token itself.
</details>

### Exercise 7 (challenge): Benchmark verification
Benchmark `ParseToken` and `bcrypt.CompareHashAndPassword`. Why is token verification so much cheaper, and why does that matter for scaling?

<details><summary>Solution</summary>

`ParseToken` is a couple of Base64 decodes, a JSON parse, and one HMAC: roughly **1–3 microseconds**, whereas bcrypt is **~50 ms**, tens of thousands of times slower. That's the point of tokens: the expensive password check happens once at login; every subsequent request costs microseconds, so the API can handle huge request volumes without a database or session-store lookup on the hot path.
</details>

---

## 19. Quiz

1. What's the format of the `Authorization` header for bearer tokens?
2. Why is `hmac.Equal` used instead of `==`?
3. Why must the verifier check `alg` itself?
4. When may the server trust the payload?
5. What do `401` and `403` mean here?
6. Why is `Authenticate` route-level rather than global?
7. How does a handler learn who the caller is?
8. Can a server revoke a single JWT before it expires? What are the workarounds?

<details><summary>Answers</summary>

1. `Authorization: Bearer <token>`.
2. It compares in constant time, preventing timing attacks that reveal correct bytes.
3. The header is attacker-controlled; trusting it enables `alg: none` and algorithm-confusion attacks.
4. Only after the signature has been verified.
5. `401`: not authenticated (missing/invalid/expired token); `403`: authenticated but not permitted.
6. `/login` and `/users` must be public, and preflights arrive without credentials; only specific routes need protection.
7. From the request context (`auth.UserIDFrom(r.Context())`), populated by the middleware.
8. Not by itself (it's stateless). Workarounds: short lifetimes + refresh tokens, a denylist of token IDs, or a per-user token version.
</details>

---

## 20. Summary

- Clients send **`Authorization: Bearer <token>`**; the server extracts it with `strings.Cut`, then **verifies**: three parts → **header `alg` pinned to HS256** → recompute **HMAC** and compare with **`hmac.Equal`** (constant time) → *then* parse the payload → require `sub` and enforce **`exp`**.
- Errors use **wrapped sentinels** (`ErrInvalidToken`, `ErrExpiredToken`): callers branch with `errors.Is`, logs get detail, clients get generic messages.
- The **`Authenticate` middleware** stops the chain with `401` + `WWW-Authenticate: Bearer` on failure; on success it stores the user ID in the **context** (helpers in the `auth` package so handlers don't import middleware) and passes `r.WithContext(ctx)` down.
- **Route-level** protection with `mux.Handle(pattern, authn(handler))`: public routes (`/login`, `/users`, product listing) and preflights stay open; `POST /products` and `GET /me` require a token.
- We tested by **attacking the verifier**: tampering, forgery, `alg: none`, algorithm confusion, expiry, malformed input, and two-way interoperability with `golang-jwt`.
- Production realities: token storage (XSS vs CSRF), revocation and refresh tokens, key rotation, asymmetric algorithms, HTTPS, and preferring vetted libraries.
- Remaining structural problems (global state, no database, tight coupling) drive the next part of the course.

### ➡️ What's next?

**Part 10** cleans up the architecture. [Chapter 50](50-removing-tight-coupling.md) removes **tight coupling**: organizing code by *feature* (`user`, `product`) and injecting dependencies instead of reaching into globals.
