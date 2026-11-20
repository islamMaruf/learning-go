# Chapter 48: Authentication with JWT — Users, Passwords, and Signed Tokens

> **Goal of this chapter:** Give the API an identity system. Right now anyone can create products. You'll add **user registration** and **login**, store passwords the only acceptable way (**bcrypt** hashes), and issue **JSON Web Tokens (JWTs)** that prove who a caller is. Along the way you'll learn the building blocks *from the ground up*: **Base64/Base64URL** encoding, **SHA-256** hashing, **HMAC** signatures, and the exact anatomy of a JWT. We'll build the token creation by hand (about 30 lines), then prove it correct by cross-checking against the widely used `golang-jwt` library. (Chapter 49 verifies tokens and protects routes.)

**Difficulty:** 🔴 Advanced  **Estimated time:** 5–6 hours  **Prerequisite:** [Chapters 22, 24, 47](47-a-real-project-structure.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The problem: everyone is anonymous](#2-the-problem)
3. [Authentication vs. authorization](#3-authentication-vs-authorization)
4. [How token-based authentication works](#4-how-token-based-authentication-works)
5. [Building block 1: Base64 and Base64URL](#5-building-block-1-base64)
6. [Building block 2: hashing and SHA-256](#6-building-block-2-hashing-and-sha-256)
7. [Storing passwords: why not SHA-256?](#7-storing-passwords)
8. [Building block 3: HMAC](#8-building-block-3-hmac)
9. [Anatomy of a JWT](#9-anatomy-of-a-jwt)
10. [The user model and store](#10-the-user-model-and-store)
11. [The `auth` package: passwords and tokens](#11-the-auth-package)
12. [Config: the JWT secret](#12-config-the-jwt-secret)
13. [Handlers: register and login](#13-handlers-register-and-login)
14. [Wiring the routes](#14-wiring-the-routes)
15. [Tests](#15-tests)
16. [Trying it out](#16-trying-it-out)
17. [Security checklist](#17-security-checklist)
18. [Common mistakes](#18-common-mistakes)
19. [Exercises](#19-exercises)
20. [Quiz](#20-quiz)
21. [Summary](#21-summary)

---

## 1. What you will learn

- The difference between **authentication** and **authorization**
- Why HTTP APIs use **tokens** instead of remembering logins in server memory (statelessness, Chapter 37)
- **Base64 / Base64URL**: turning bytes into safe text, and why JWTs use the URL variant
- **Hashing**: what SHA-256 is, its key properties, and what it is *not* good for
- **Password storage** with **bcrypt**: salts, cost, and constant-time comparison
- **HMAC**: proving a message wasn't changed by anyone without the secret key
- The three parts of a **JWT** (header, payload, signature) and how to build one
- How to build registration and login endpoints that don't leak information

---

## 2. The problem

Our API has an open door:

```bash
curl -X POST http://localhost:8080/products -d '{"title":"Free money","price":1}'
# 201 Created   ← anybody, anywhere, can add products
```

In a real shop only **staff** should create products, **customers** should manage only *their own* orders, and the server must know *who is asking* on every request. That requires:

1. **Accounts**: a way to register users and store credentials safely.
2. **Authentication**: a way to prove *"I am user 42"* (login).
3. **Carrying that proof** on every later request, without asking for the password again each time.
4. **Authorization**: deciding what the proven identity is *allowed* to do (Chapter 49 and beyond).

---

## 3. Authentication vs. authorization

Two words that sound alike and are constantly confused:

| | **Authentication (AuthN)** | **Authorization (AuthZ)** |
|--|----------------------------|---------------------------|
| Question | **Who are you?** | **What may you do?** |
| Happens | First | After authentication |
| Example | Log in with email + password | Only admins may delete products |
| Failure status | `401 Unauthorized` (really "unauthenticated") | `403 Forbidden` |
| Mechanism (this course) | Passwords + JWT | Roles/permissions (later) |

> **Analogy: an airport.** Showing your passport at the gate (*authentication*) proves who you are. Your boarding pass decides which plane you may board and whether you may enter the lounge (*authorization*).

---

## 4. How token-based authentication works

A REST API is **stateless** (Chapter 37): the server doesn't keep "user 5 is logged in" in memory between requests. Instead, on login it hands the client a **token**: a small piece of signed data saying "this is user 42". The client presents the token on every request.

```
1. LOGIN
   client ── POST /login {email, password} ─────────────────────────────►  server
                                                     verify password (bcrypt)
                                                     create a signed token: "user 42, expires in 24h"
   client ◄───────────────── 200 {"token": "eyJhbGci..."} ───────────────  server

2. LATER REQUESTS
   client ── POST /products  ──  Authorization: Bearer eyJhbGci... ───────►  server
                                                     check the signature and expiry
                                                     → "this really is user 42"
   client ◄────────────────────────── 201 Created ──────────────────────  server
```

**Why this scales:** any server instance can verify the token with the shared secret: no session table, no sticky routing. **The catch:** a token can't be "un-issued" easily; it's valid until it expires, so keep lifetimes reasonably short (Chapter 49 discusses trade-offs).

**The core idea, a passport analogy:** a passport is issued once by a trusted authority (login), contains your identity, is hard to forge (signature), and can be shown at any border (any request). Border officers don't phone the passport office each time; they check the passport's security features. A JWT is a digital passport.

---

## 5. Building block 1: Base64

### The problem Base64 solves

A JWT contains **JSON** (text) and a **signature** (raw binary bytes). It must travel inside an HTTP header, which must be plain, safe ASCII text: no control characters, no newlines. How do we embed arbitrary bytes safely in text? **Base64**.

> **Base64 is not encryption.** It's a *reversible re-encoding* of bytes into a text alphabet. Anyone can decode it. It provides **zero secrecy**.

### How it works

Base64 uses **64 characters**: `A–Z`, `a–z`, `0–9`, and two symbols (`+` and `/` in the standard alphabet). It reads the input **3 bytes (24 bits) at a time** and splits them into **four 6-bit groups** (2⁶ = 64), mapping each group to one character:

```
"Man"  =  M        a        n
ASCII  =  01001101 01100001 01101110      ← 3 bytes = 24 bits
regroup:  010011 010110 000101 101110     ← four 6-bit groups
values:      19     22      5     46
chars:        T      W      F      u      → "TWFu"
```

Real output from Go:

| Input | Standard Base64 |
|-------|-----------------|
| `M` | `TQ==` |
| `Ma` | `TWE=` |
| `Man` | `TWFu` |
| `Go is fun!` | `R28gaXMgZnVuIQ==` |

Output is about **33% larger** than the input (3 bytes → 4 characters). When the input isn't a multiple of 3 bytes, standard Base64 pads with `=` signs.

### Base64URL: the variant JWTs use

The standard alphabet contains `+` and `/` (and `=` padding), which have special meanings in **URLs and query strings**. **Base64URL** swaps them for URL-safe characters and JWTs additionally **drop the padding**:

| Variant | Alphabet's last two | Padding | Go encoder |
|---------|--------------------|---------|-----------|
| Standard | `+` `/` | `=` | `base64.StdEncoding` |
| URL-safe | `-` `_` | `=` | `base64.URLEncoding` |
| **Raw URL-safe (used by JWT)** | `-` `_` | none | **`base64.RawURLEncoding`** |

Real output for the bytes `0xFB 0xFF 0xFE`:

```
std:     +//+
url:     -__-
rawurl:  -__-
```

### Base64 in Go

```go
package main

import (
	"encoding/base64"
	"fmt"
)

func main() {
	text := `{"alg":"HS256","typ":"JWT"}`

	encoded := base64.RawURLEncoding.EncodeToString([]byte(text)) // bytes → text
	fmt.Println(encoded) // eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9

	decoded, err := base64.RawURLEncoding.DecodeString(encoded)   // text → bytes
	fmt.Println(string(decoded), err) // {"alg":"HS256","typ":"JWT"} <nil>
}
```

You can decode any Base64 by hand in a browser console (`atob(...)`) or with online tools, which is exactly why **nothing secret may go into a JWT payload**: anyone holding a token can read it. What the signature protects is *integrity* (no tampering), not *confidentiality*.

---

## 6. Building block 2: hashing and SHA-256

A **hash function** takes input of any size and produces a **fixed-size fingerprint** (the *digest*). **SHA-256** ("Secure Hash Algorithm, 256-bit") always produces **32 bytes** (64 hex characters).

Real output from Go:

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

func main() {
	for _, s := range []string{"hello", "hello!", "Hello", ""} {
		h := sha256.Sum256([]byte(s))
		fmt.Printf("sha256(%q) = %s\n", s, hex.EncodeToString(h[:]))
	}
}
```

```
sha256("hello")  = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
sha256("hello!") = ce06092fb948d9ffac7d1a376e404b26b7575bcc11ee05a4615fef4fec3a308b
sha256("Hello")  = 185f8db32271fe25f561a6fc938b2e264306ec304eda518007d1764826381969
sha256("")       = e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

### The properties that make a hash useful

| Property | Meaning | Visible above |
|----------|---------|---------------|
| **Deterministic** | Same input → always the same digest | `"hello"` always gives `2cf24d…` |
| **Fixed size** | Any input → 32 bytes | even the empty string yields 64 hex chars |
| **One-way** (preimage resistant) | Given a digest, you can't feasibly recover the input | no way back from `2cf24d…` |
| **Avalanche effect** | A tiny input change scrambles the whole output | `"hello"` vs `"hello!"` share nothing |
| **Collision resistant** | Practically impossible to find two inputs with the same digest | |

Uses: file integrity checks, content addressing (Git), deduplication, and as an ingredient in **HMAC** (next). The SHA-2 family also includes SHA-224/384/512; the older **MD5 and SHA-1 are broken** for security use.

> **A hash is not encryption.** Encryption is reversible with a key; a hash has no key and no way back.

---

## 7. Storing passwords

**Rule 1:** never store passwords in plain text. When (not if) a database leaks, plain text exposes every account, *and* every other site where people reused that password.

**Rule 2:** store a **hash** instead. At login, hash what the user types and compare with the stored hash.

**Why plain SHA-256 is still the wrong tool for passwords:**

| Problem | Explanation |
|---------|-------------|
| **Too fast** | SHA-256 is designed to be quick: a GPU computes *billions* of hashes per second. An attacker with a leaked table can try all common passwords almost instantly (dictionary/brute-force attack) |
| **No salt** | Identical passwords produce identical hashes, so cracking one cracks all users with that password, and precomputed "rainbow tables" work |

A proper **password hashing function** fixes both:

- It includes a random **salt** (unique per password, stored alongside the hash) so identical passwords hash differently and precomputed tables are useless.
- It is deliberately **slow and tunable** (a **cost** factor) so each guess costs the attacker real time, and you can raise the cost as hardware improves.

Go's standard-adjacent choice: **`golang.org/x/crypto/bcrypt`** (alternatives: `argon2id`, `scrypt`).

```bash
go get golang.org/x/crypto/bcrypt
```

> 📝 **A real-world gotcha: Go versions and dependencies.** Modules declare the minimum Go version they need. Recent releases of `golang.org/x/crypto` require a very new Go, so `go get` (or `go mod tidy`) may **raise the `go` line in your `go.mod`** (for example to `go 1.26.0`), and, with the default `GOTOOLCHAIN=auto`, Go will quietly **download and use the newer toolchain** for this module. That's normal and safe (you'll see `go: downloading go1.26.0`). If you'd rather stay on the Go you installed, pin an older release that still supports it, e.g. `go get golang.org/x/crypto@v0.31.0`. Check what happened with `go version` and `cat go.mod`. You will meet this again whenever you add dependencies.

```go
hash, err := bcrypt.GenerateFromPassword([]byte("s3cret-pass"), bcrypt.DefaultCost) // cost 10 → 2^10 rounds
err = bcrypt.CompareHashAndPassword(hash, []byte("s3cret-pass"))                     // nil means "matches"
```

A bcrypt hash is a single self-describing string; algorithm, cost, salt, and hash are all inside:

```
$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
 │  │  └──── 22-char salt + 31-char hash ────┘
 │  └ cost (2^10 iterations)
 └ algorithm version
```

Facts to remember:

- Hashing the *same* password twice gives **different** strings (different random salts), yet `Compare` still works (it reads the salt from the stored hash).
- `CompareHashAndPassword` is **constant-time** with respect to the hash comparison.
- bcrypt only looks at the first **72 bytes** of the password (newer versions return an error for longer input): enforce a maximum length.
- `DefaultCost` (10) takes tens of milliseconds; raise it (12+) as hardware improves. That deliberate slowness is the protection.

---

## 8. Building block 3: HMAC

A plain hash proves *nothing* about who made it: anyone can compute `sha256(message)`. We need a fingerprint that **only someone with a secret key** can produce.

**HMAC** (Hash-based Message Authentication Code) mixes a **secret key** into the hash:

```
HMAC-SHA256(key, message)  →  32 bytes
```

Properties:

- Same key + same message → same MAC (deterministic).
- Without the key you **cannot forge** a valid MAC for a new message, and you can't tell what the key is from a MAC.
- Change one bit of the message → a completely different MAC.

Go:

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

func main() {
	mac := hmac.New(sha256.New, []byte("secret")) // key = "secret"
	mac.Write([]byte("message"))
	fmt.Println(hex.EncodeToString(mac.Sum(nil)))
	// 8b5f48702995c1598c573db1e21866a9b825d4a794d169d7060a03605796360b
}
```

**The trust model.** The server holds the secret. It signs tokens it issues. Later it recomputes the HMAC of what a client presents; if it matches, the token can only have been produced by someone who knows the secret (the server), and it hasn't been altered since. That's how a stateless server recognizes its own tokens.

---

## 9. Anatomy of a JWT

A **JSON Web Token** (RFC 7519, pronounced "jot") is three Base64URL strings joined by dots:

```
HEADER . PAYLOAD . SIGNATURE
```

A **real** token produced by the code in this chapter (secret `demo-secret-do-not-use-in-production-1234`, user 42, issued at 2026-09-26T00:00:00Z, valid for one hour):

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsImlhdCI6MTc5MDM4MDgwMCwiZXhwIjoxNzkwMzg0NDAwfQ.IDGmpL8VvXHpyw5f4HlGBMiX6-0LO7ROOr8N1duFBOU
└────────────── header ─────────────┘ └────────────────────── payload ───────────────────────────┘ └────────── signature ──────────┘
```

Decode each part:

**1. Header**: metadata about the token:

```json
{"alg":"HS256","typ":"JWT"}
```
`alg` = the signing algorithm (HMAC-SHA256); `typ` = token type.

**2. Payload**: the **claims**, i.e., the facts the token asserts:

```json
{"sub":"42","iat":1790380800,"exp":1790384400}
```

| Claim | Name | Meaning |
|-------|------|---------|
| `sub` | Subject | Who the token is about (our user ID) |
| `iat` | Issued At | When it was issued (Unix seconds) |
| `exp` | Expiration | After this moment the token is invalid |
| (others) | `iss` issuer, `aud` audience, `nbf` not-before, `jti` token ID, plus your own (`role`, ...) |

**3. Signature**: proves nobody changed the header or payload:

```
signature = HMAC-SHA256( secret, base64url(header) + "." + base64url(payload) )
```

### What the signature protects, and what it doesn't

| ✅ Protects | ❌ Doesn't protect |
|------------|--------------------|
| **Integrity**: changing any character of header/payload breaks the signature | **Confidentiality**: the payload is only Base64-encoded; anyone can read it |
| **Authenticity**: only holders of the secret can create valid tokens | Revocation: a valid token stays valid until it expires |

**Tampering demo.** An attacker changes `"sub":"42"` to `"sub":"1"`, hoping to become user 1 (payload becomes `eyJzdWIiOiIxIiwiaWF0IjoxNzkwMzgwODAwLCJleHAiOjE3OTAzODQ0MDB9`). They can't recompute the signature without the secret, so when the server recomputes `HMAC(secret, header.newPayload)` it gets a different value than the attacker's old signature: **rejected**. (Chapter 49 implements the check.)

**Never put secrets in the payload** (passwords, card numbers). Treat it as a postcard: readable by anyone who carries it.

### The `alg` header trap (know this for interviews)

The header is attacker-controlled. Two historical attacks: `alg: "none"` (an unsigned token that naive libraries accepted) and *algorithm confusion* (RS256 public key used as an HS256 secret). **The verifier must not trust the token's `alg`; it must insist on the algorithm it expects.** Our verifier (Chapter 49) checks the header equals exactly HS256.

---

## 10. The user model and store

```go
// file: models/user.go
package models

import "time"

// User is a registered account.
type User struct {
	ID           int       `json:"id"`
	Email        string    `json:"email"`
	PasswordHash string    `json:"-"` // never serialized: `json:"-"` excludes it from JSON output
	CreatedAt    time.Time `json:"createdAt"`
}
```

The `json:"-"` tag (Chapter 40) is a safety net: even if a handler accidentally returns a `User`, the hash can't leak.

The store, in memory for now (Chapters 52–56 replace it with PostgreSQL):

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
```

Design notes: emails are **normalized** (lowercase, trimmed) so `Asha@Example.com` and `asha@example.com` are the same account; the duplicate check and insert share **one lock** (avoiding a check-then-act race, Chapter 41 Exercise 3).

---

## 11. The `auth` package

Everything about *credentials* lives in one small package, with no HTTP in it.

### Passwords

```go
// file: auth/password.go
package auth

import "golang.org/x/crypto/bcrypt"

// HashPassword returns a salted bcrypt hash of the password.
func HashPassword(plain string) (string, error) {
	hash, err := bcrypt.GenerateFromPassword([]byte(plain), bcrypt.DefaultCost)
	if err != nil {
		return "", err
	}
	return string(hash), nil
}

// CheckPassword reports whether plain matches the stored bcrypt hash.
func CheckPassword(hash, plain string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(plain)) == nil
}
```

### Tokens: creation

```go
// file: auth/jwt.go
package auth

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"errors"
	"strconv"
	"time"
)

// Claims are the facts a token asserts about its holder.
type Claims struct {
	Subject   string `json:"sub"` // the user ID
	IssuedAt  int64  `json:"iat"` // Unix seconds
	ExpiresAt int64  `json:"exp"` // Unix seconds
}

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

// sign computes HMAC-SHA256(secret, input).
func sign(secret []byte, input string) []byte {
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(input))
	return mac.Sum(nil)
}
```

That's the entire token-*creation* logic: **marshal claims → Base64URL → sign `header.payload` → append the Base64URL signature.** Thirty lines, no magic.

---

## 12. Config: the JWT secret

The signing secret is the crown jewel: anyone who has it can mint tokens for any user. It must come from configuration (Chapter 47), be **required** (no default), and be long and random.

Generate one:

```bash
head -c 48 /dev/urandom | base64      # Linux/macOS
```

We extend `Config` with `JWTSecret` and `JWTTTL` and validate them. Full updated file:

```go
// file: config/config.go
package config

import (
	"errors"
	"fmt"
	"io/fs"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/joho/godotenv"
)

// Config holds every setting the application needs. It is loaded once at start-up and then read-only.
type Config struct {
	Env            string        // "development", "test" or "production"
	Port           string        // TCP port to listen on, e.g. "8080"
	AllowedOrigins []string      // browser origins allowed by CORS
	LogFormat      string        // "text" or "json"
	JWTSecret      string        // key used to sign tokens (keep secret!)
	JWTTTL         time.Duration // how long an issued token stays valid
}

// String redacts the secret so a Config can be logged safely.
func (c *Config) String() string {
	return fmt.Sprintf("Config{Env:%s Port:%s Origins:%v LogFormat:%s JWTSecret:*** JWTTTL:%s}",
		c.Env, c.Port, c.AllowedOrigins, c.LogFormat, c.JWTTTL)
}

const minSecretLength = 32

// Load reads configuration from the environment (and an optional .env file).
// Real environment variables take precedence over values in .env.
func Load() (*Config, error) {
	if err := godotenv.Load(); err != nil && !errors.Is(err, fs.ErrNotExist) {
		return nil, fmt.Errorf("reading .env: %w", err)
	}
	return fromLookup(os.LookupEnv)
}

// fromLookup builds a Config using lookup to read variables. Tests can pass a fake lookup.
func fromLookup(lookup func(string) (string, bool)) (*Config, error) {
	get := func(key, def string) string {
		if v, ok := lookup(key); ok && strings.TrimSpace(v) != "" {
			return strings.TrimSpace(v)
		}
		return def
	}

	cfg := &Config{
		Env:            get("APP_ENV", "development"),
		Port:           get("HTTP_PORT", "8080"),
		AllowedOrigins: splitCSV(get("ALLOWED_ORIGINS", "http://localhost:5173")),
		LogFormat:      get("LOG_FORMAT", "text"),
		JWTSecret:      get("JWT_SECRET", ""), // required: no default
	}

	var errs []error

	if n, err := strconv.Atoi(cfg.Port); err != nil || n < 1 || n > 65535 {
		errs = append(errs, fmt.Errorf("HTTP_PORT must be a number between 1 and 65535, got %q", cfg.Port))
	}

	switch cfg.Env {
	case "development", "test", "production":
	default:
		errs = append(errs, fmt.Errorf("APP_ENV must be development, test or production, got %q", cfg.Env))
	}

	switch cfg.LogFormat {
	case "text", "json":
	default:
		errs = append(errs, fmt.Errorf("LOG_FORMAT must be text or json, got %q", cfg.LogFormat))
	}

	for _, origin := range cfg.AllowedOrigins {
		if err := validateOrigin(origin); err != nil {
			errs = append(errs, err)
		}
	}

	// The token lifetime: a duration like "15m" or "24h".
	ttlText := get("JWT_TTL", "24h")
	ttl, err := time.ParseDuration(ttlText)
	if err != nil || ttl <= 0 {
		errs = append(errs, fmt.Errorf("JWT_TTL must be a positive duration like 15m or 24h, got %q", ttlText))
	}
	cfg.JWTTTL = ttl

	// The secret: required, long enough, and (in production) not the example value.
	switch {
	case cfg.JWTSecret == "":
		errs = append(errs, errors.New("JWT_SECRET is required (generate one with: head -c 48 /dev/urandom | base64)"))
	case len(cfg.JWTSecret) < minSecretLength:
		errs = append(errs, fmt.Errorf("JWT_SECRET must be at least %d characters", minSecretLength))
	case cfg.Env == "production" && strings.HasPrefix(cfg.JWTSecret, "change-me"):
		errs = append(errs, errors.New("JWT_SECRET is still the example value; set a real secret in production"))
	}

	if err := errors.Join(errs...); err != nil {
		return nil, err
	}
	return cfg, nil
}

// splitCSV splits "a, b,c" into ["a" "b" "c"], dropping empty items.
func splitCSV(s string) []string {
	var out []string
	for _, part := range strings.Split(s, ",") {
		if part = strings.TrimSpace(part); part != "" {
			out = append(out, part)
		}
	}
	return out
}

// validateOrigin checks that an origin looks like scheme://host[:port] with no path.
func validateOrigin(origin string) error {
	u, err := url.Parse(origin)
	if err != nil || (u.Scheme != "http" && u.Scheme != "https") || u.Host == "" || (u.Path != "" && u.Path != "/") {
		return fmt.Errorf("ALLOWED_ORIGINS entry %q must look like https://host[:port] (no path)", origin)
	}
	return nil
}
```

Note the `String()` method (Chapter 22): if anyone logs the config, the secret prints as `***`.

The example environment file:

```env
# file: .env.example
# Copy this file to ".env" and adjust. Real .env files are NEVER committed to Git.
APP_ENV=development
HTTP_PORT=8080
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:3000
LOG_FORMAT=text

# REQUIRED. At least 32 characters. Generate one with:  head -c 48 /dev/urandom | base64
JWT_SECRET=change-me-please-use-a-long-random-string-at-least-32-chars
# How long a login token stays valid (e.g. 15m, 24h).
JWT_TTL=24h
```

The config tests need updating: every valid environment now needs a `JWT_SECRET`, and there are new cases:

```go
// file: config/config_test.go
package config

import (
	"reflect"
	"strings"
	"testing"
	"time"
)

const goodSecret = "0123456789abcdef0123456789abcdef0123" // 36 characters

func lookupFrom(env map[string]string) func(string) (string, bool) {
	return func(key string) (string, bool) {
		v, ok := env[key]
		return v, ok
	}
}

// withSecret returns env plus a valid JWT_SECRET.
func withSecret(env map[string]string) map[string]string {
	out := map[string]string{"JWT_SECRET": goodSecret}
	for k, v := range env {
		out[k] = v
	}
	return out
}

func TestDefaults(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(withSecret(nil)))
	if err != nil {
		t.Fatal(err)
	}
	want := &Config{
		Env:            "development",
		Port:           "8080",
		AllowedOrigins: []string{"http://localhost:5173"},
		LogFormat:      "text",
		JWTSecret:      goodSecret,
		JWTTTL:         24 * time.Hour,
	}
	if !reflect.DeepEqual(cfg, want) {
		t.Errorf("got %+v, want %+v", cfg, want)
	}
}

func TestOverrides(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(withSecret(map[string]string{
		"APP_ENV":         "production",
		"HTTP_PORT":       "9090",
		"ALLOWED_ORIGINS": "https://shop.example.com, https://admin.example.com/",
		"LOG_FORMAT":      "json",
		"JWT_TTL":         "15m",
	})))
	if err != nil {
		t.Fatal(err)
	}
	if cfg.Env != "production" || cfg.Port != "9090" || cfg.LogFormat != "json" || cfg.JWTTTL != 15*time.Minute {
		t.Errorf("unexpected config %+v", cfg)
	}
	if len(cfg.AllowedOrigins) != 2 {
		t.Errorf("expected 2 origins, got %v", cfg.AllowedOrigins)
	}
}

func TestEmptyValuesFallBackToDefaults(t *testing.T) {
	cfg, err := fromLookup(lookupFrom(withSecret(map[string]string{"HTTP_PORT": "   ", "LOG_FORMAT": ""})))
	if err != nil {
		t.Fatal(err)
	}
	if cfg.Port != "8080" || cfg.LogFormat != "text" {
		t.Errorf("blank values must use the defaults, got %+v", cfg)
	}
}

func TestInvalidValues(t *testing.T) {
	tests := []struct {
		name string
		env  map[string]string
		want string
	}{
		{"port is not a number", map[string]string{"HTTP_PORT": "eighty"}, "HTTP_PORT"},
		{"port too large", map[string]string{"HTTP_PORT": "70000"}, "HTTP_PORT"},
		{"unknown environment", map[string]string{"APP_ENV": "staging"}, "APP_ENV"},
		{"unknown log format", map[string]string{"LOG_FORMAT": "xml"}, "LOG_FORMAT"},
		{"origin without scheme", map[string]string{"ALLOWED_ORIGINS": "localhost:5173"}, "ALLOWED_ORIGINS"},
		{"origin with a path", map[string]string{"ALLOWED_ORIGINS": "https://shop.example.com/app"}, "ALLOWED_ORIGINS"},
		{"ttl not a duration", map[string]string{"JWT_TTL": "tomorrow"}, "JWT_TTL"},
		{"ttl zero", map[string]string{"JWT_TTL": "0s"}, "JWT_TTL"},
		{"ttl negative", map[string]string{"JWT_TTL": "-5m"}, "JWT_TTL"},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			cfg, err := fromLookup(lookupFrom(withSecret(tc.env)))
			if err == nil {
				t.Fatalf("expected an error, got config %+v", cfg)
			}
			if !strings.Contains(err.Error(), tc.want) {
				t.Errorf("error %q should mention %q", err, tc.want)
			}
		})
	}
}

func TestJWTSecretRules(t *testing.T) {
	tests := []struct {
		name string
		env  map[string]string
	}{
		{"missing", map[string]string{}},
		{"empty", map[string]string{"JWT_SECRET": ""}},
		{"too short", map[string]string{"JWT_SECRET": "short"}},
		{"example value in production", map[string]string{
			"APP_ENV":    "production",
			"JWT_SECRET": "change-me-please-use-a-long-random-string-at-least-32-chars",
		}},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			_, err := fromLookup(lookupFrom(tc.env))
			if err == nil || !strings.Contains(err.Error(), "JWT_SECRET") {
				t.Errorf("expected a JWT_SECRET error, got %v", err)
			}
		})
	}
}

func TestExampleSecretIsAcceptedInDevelopment(t *testing.T) {
	_, err := fromLookup(lookupFrom(map[string]string{
		"JWT_SECRET": "change-me-please-use-a-long-random-string-at-least-32-chars",
	}))
	if err != nil {
		t.Errorf("the example .env must work out of the box in development: %v", err)
	}
}

func TestReportsAllProblemsAtOnce(t *testing.T) {
	_, err := fromLookup(lookupFrom(map[string]string{
		"HTTP_PORT":  "abc",
		"APP_ENV":    "moon",
		"LOG_FORMAT": "yaml",
	}))
	if err == nil {
		t.Fatal("expected errors")
	}
	for _, key := range []string{"HTTP_PORT", "APP_ENV", "LOG_FORMAT", "JWT_SECRET"} {
		if !strings.Contains(err.Error(), key) {
			t.Errorf("combined error should mention %s: %q", key, err)
		}
	}
}

func TestStringRedactsTheSecret(t *testing.T) {
	cfg := &Config{JWTSecret: goodSecret, Port: "8080"}
	if strings.Contains(cfg.String(), goodSecret) {
		t.Error("String() leaked the JWT secret")
	}
}
```

---

## 13. Handlers: register and login

The user handlers need **configuration** (the secret and lifetime), so unlike our earlier plain functions they become **methods on a struct** that holds their dependencies, our first step toward dependency injection (Chapter 50).

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

// UserHandler serves registration and login.
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
	if err != nil || addr.Address != c.Email { // reject "Name <a@b.c>" style input
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

	util.SendData(w, http.StatusCreated, user) // PasswordHash is excluded by its json:"-" tag
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

	// Always run a bcrypt comparison, even for unknown emails, so response time doesn't reveal
	// which emails are registered.
	hash := dummyHash
	if found {
		hash = user.PasswordHash
	}
	passwordOK := auth.CheckPassword(hash, req.Password)

	if !found || !passwordOK {
		// One message for both cases: don't reveal whether the email exists.
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
```

Security details worth understanding:

- **Same error for "no such email" and "wrong password"** (`401 invalid email or password`). If they differed, attackers could discover which emails have accounts (**user enumeration**).
- **Timing equalization.** bcrypt takes ~50 ms; *skipping* it for unknown emails would make those responses much faster, leaking the same information through timing. So we compare against `dummyHash` when the user isn't found.
- **`credentials` is reused** for both register and login (same two fields), but only registration enforces password rules: for *login* we must accept whatever was set at signup.
- **Email validation** via `net/mail`, requiring `addr.Address == c.Email` so display-name forms are rejected.
- **`maxPasswordLength`** protects against bcrypt's 72-byte limit, and against giant inputs making hashing expensive.
- Registration *does* reveal that an email exists (`409`); that's a common, accepted trade-off for usability. Sites with stricter privacy needs respond identically and confirm by email.

---

## 14. Wiring the routes

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

	// Products (authentication is added in Chapter 49)
	mux.HandleFunc("GET /products", handlers.GetProducts)
	mux.HandleFunc("POST /products", handlers.CreateProduct)
	mux.HandleFunc("GET /products/{id}", handlers.GetProduct)

	// Accounts
	mux.HandleFunc("POST /users", users.CreateUser)
	mux.HandleFunc("POST /login", users.Login)

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

`users.CreateUser` is a **method value** (Chapter 22): it carries its receiver, so it fits `HandleFunc`'s handler-function signature.

---

## 15. Tests

### The `auth` package: cross-checking against a real library

Writing crypto-adjacent code by hand is risky, so we **prove** ours correct by verifying our tokens with the industry-standard library `github.com/golang-jwt/jwt/v5` (used only in tests):

```go
// file: auth/auth_test.go
package auth

import (
	"strings"
	"testing"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

var (
	secret = []byte("demo-secret-do-not-use-in-production-1234")
	epoch  = time.Date(2026, 9, 26, 0, 0, 0, 0, time.UTC)
)

func TestPasswordHashing(t *testing.T) {
	hash, err := HashPassword("correct horse battery staple")
	if err != nil {
		t.Fatal(err)
	}
	if hash == "correct horse battery staple" || !strings.HasPrefix(hash, "$2") {
		t.Errorf("expected a bcrypt hash, got %q", hash)
	}
	if !CheckPassword(hash, "correct horse battery staple") {
		t.Error("the right password must verify")
	}
	if CheckPassword(hash, "wrong password") {
		t.Error("a wrong password must not verify")
	}

	// same password, different salts → different hashes, both valid
	hash2, _ := HashPassword("correct horse battery staple")
	if hash == hash2 {
		t.Error("two hashes of one password should differ (random salt)")
	}
	if !CheckPassword(hash2, "correct horse battery staple") {
		t.Error("the second hash must verify too")
	}
}

func TestCreateTokenMatchesKnownValue(t *testing.T) {
	// This exact token appears in the chapter text; if the algorithm changes, this fails.
	want := "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9." +
		"eyJzdWIiOiI0MiIsImlhdCI6MTc5MDM4MDgwMCwiZXhwIjoxNzkwMzg0NDAwfQ." +
		"IDGmpL8VvXHpyw5f4HlGBMiX6-0LO7ROOr8N1duFBOU"

	got, err := CreateToken(secret, 42, time.Hour, epoch)
	if err != nil {
		t.Fatal(err)
	}
	if got != want {
		t.Errorf("token mismatch:\n got  %s\n want %s", got, want)
	}
}

func TestCreateTokenHasThreeParts(t *testing.T) {
	tok, _ := CreateToken(secret, 7, time.Hour, epoch)
	parts := strings.Split(tok, ".")
	if len(parts) != 3 {
		t.Fatalf("expected 3 dot-separated parts, got %d", len(parts))
	}
	if strings.ContainsAny(tok, "=+/") {
		t.Errorf("token must use unpadded Base64URL only, got %q", tok)
	}
}

// The important test: a battle-tested library must accept tokens we mint.
func TestTokensAreValidForTheReferenceLibrary(t *testing.T) {
	now := time.Now()
	tok, err := CreateToken(secret, 42, time.Hour, now)
	if err != nil {
		t.Fatal(err)
	}

	claims := jwt.MapClaims{}
	parsed, err := jwt.ParseWithClaims(tok, claims, func(token *jwt.Token) (any, error) {
		return secret, nil
	}, jwt.WithValidMethods([]string{"HS256"}))
	if err != nil {
		t.Fatalf("golang-jwt rejected our token: %v", err)
	}
	if !parsed.Valid {
		t.Fatal("token not valid")
	}
	if claims["sub"] != "42" {
		t.Errorf("sub = %v, want 42", claims["sub"])
	}
	if int64(claims["exp"].(float64)) != now.Add(time.Hour).Unix() {
		t.Errorf("exp = %v", claims["exp"])
	}
}

func TestReferenceLibraryRejectsWrongSecret(t *testing.T) {
	tok, _ := CreateToken(secret, 42, time.Hour, time.Now())
	_, err := jwt.Parse(tok, func(token *jwt.Token) (any, error) { return []byte("another-secret-entirely-32-chars!!"), nil })
	if err == nil {
		t.Error("a token signed with our secret must not verify under a different one")
	}
}

func TestCreateTokenRejectsEmptySecret(t *testing.T) {
	if _, err := CreateToken(nil, 1, time.Hour, epoch); err == nil {
		t.Error("expected an error for an empty secret")
	}
}
```

The first test pins **the exact token printed in this chapter**, so the text and code can never drift apart. The interoperability test is the one that matters: `golang-jwt` accepting our token means our header, payload encoding, and signature are all correct.

### The endpoints

These tests exercise registration and login end to end through the real router, including the "cannot enumerate accounts" property and a check that our tokens verify with the reference library.

```go
// file: rest/users_test.go
package rest

import (
	"ecommerce/config"
	"ecommerce/database"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

var authConfig = &config.Config{
	Env:            "test",
	Port:           "0",
	AllowedOrigins: []string{"http://localhost:5173"},
	LogFormat:      "text",
	JWTSecret:      "0123456789abcdef0123456789abcdef0123",
	JWTTTL:         time.Hour,
}

func doAuth(method, target, body string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	if body != "" {
		req.Header.Set("Content-Type", "application/json")
	}
	rec := httptest.NewRecorder()
	NewRouter(quietLogger, authConfig).ServeHTTP(rec, req)
	return rec
}

func register(email, password string) *httptest.ResponseRecorder {
	return doAuth(http.MethodPost, "/users", `{"email":"`+email+`","password":"`+password+`"}`)
}

func login(email, password string) *httptest.ResponseRecorder {
	return doAuth(http.MethodPost, "/login", `{"email":"`+email+`","password":"`+password+`"}`)
}

func TestRegisterCreatesAnAccountWithoutLeakingTheHash(t *testing.T) {
	database.ResetUsers()

	rec := register("Asha@Example.com", "correct-horse-battery")
	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d (%s)", rec.Code, rec.Body.String())
	}
	body := rec.Body.String()
	if strings.Contains(strings.ToLower(body), "password") || strings.Contains(body, "$2") {
		t.Errorf("response must not contain password material: %s", body)
	}

	var user struct {
		ID    int    `json:"id"`
		Email string `json:"email"`
	}
	json.NewDecoder(rec.Body).Decode(&user)
	if user.ID != 1 || user.Email != "asha@example.com" {
		t.Errorf("unexpected user %+v (email should be normalized to lowercase)", user)
	}
}

func TestRegisterValidation(t *testing.T) {
	tests := []struct {
		name, body string
		want       int
	}{
		{"bad email", `{"email":"not-an-email","password":"long-enough-password"}`, http.StatusUnprocessableEntity},
		{"display-name email", `{"email":"Asha <a@b.co>","password":"long-enough-password"}`, http.StatusUnprocessableEntity},
		{"short password", `{"email":"a@b.co","password":"short"}`, http.StatusUnprocessableEntity},
		{"huge password", `{"email":"a@b.co","password":"` + strings.Repeat("x", 100) + `"}`, http.StatusUnprocessableEntity},
		{"unknown field", `{"email":"a@b.co","password":"long-enough-password","role":"admin"}`, http.StatusBadRequest},
		{"empty body", ``, http.StatusBadRequest},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			database.ResetUsers()
			if rec := doAuth(http.MethodPost, "/users", tc.body); rec.Code != tc.want {
				t.Errorf("expected %d, got %d (%s)", tc.want, rec.Code, rec.Body.String())
			}
		})
	}
}

func TestDuplicateEmailIsRejectedCaseInsensitively(t *testing.T) {
	database.ResetUsers()
	register("asha@example.com", "correct-horse-battery")
	if rec := register("ASHA@EXAMPLE.COM", "another-good-password"); rec.Code != http.StatusConflict {
		t.Errorf("expected 409, got %d", rec.Code)
	}
}

func TestLoginReturnsAValidToken(t *testing.T) {
	database.ResetUsers()
	register("asha@example.com", "correct-horse-battery")

	rec := login("asha@example.com", "correct-horse-battery")
	if rec.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d (%s)", rec.Code, rec.Body.String())
	}
	var resp struct {
		Token string `json:"token"`
	}
	json.NewDecoder(rec.Body).Decode(&resp)
	if strings.Count(resp.Token, ".") != 2 {
		t.Fatalf("expected a JWT, got %q", resp.Token)
	}

	claims := jwt.MapClaims{}
	if _, err := jwt.ParseWithClaims(resp.Token, claims, func(*jwt.Token) (any, error) {
		return []byte(authConfig.JWTSecret), nil
	}, jwt.WithValidMethods([]string{"HS256"})); err != nil {
		t.Fatalf("token does not verify: %v", err)
	}
	if claims["sub"] != "1" {
		t.Errorf("sub = %v, want 1", claims["sub"])
	}
}

func TestLoginFailuresAreIndistinguishable(t *testing.T) {
	database.ResetUsers()
	register("asha@example.com", "correct-horse-battery")

	wrongPassword := login("asha@example.com", "wrong-password-here")
	unknownEmail := login("nobody@example.com", "correct-horse-battery")

	for name, rec := range map[string]*httptest.ResponseRecorder{"wrong password": wrongPassword, "unknown email": unknownEmail} {
		if rec.Code != http.StatusUnauthorized {
			t.Errorf("%s: expected 401, got %d", name, rec.Code)
		}
	}
	if wrongPassword.Body.String() != unknownEmail.Body.String() {
		t.Errorf("responses differ, which would let attackers enumerate accounts:\n%q\n%q",
			wrongPassword.Body.String(), unknownEmail.Body.String())
	}
}
```


```bash
go test -race ./...
```

```
?   	ecommerce	[no test files]
ok  	ecommerce/auth	1.2s
ok  	ecommerce/cmd	1.5s
ok  	ecommerce/config	1.0s
...
ok  	ecommerce/rest	2.4s
```

(bcrypt is deliberately slow, so these tests take a little longer than before.)

---

## 16. Trying it out

Add a JWT secret to your `.env` (`cp .env.example .env` does it), then:

```bash
go run .
```

**Register** (real output shape):

```bash
curl -si -X POST localhost:8080/users -H 'Content-Type: application/json' \
     -d '{"email":"asha@example.com","password":"correct-horse-battery"}'
```

```
HTTP/1.1 201 Created
Content-Type: application/json
...
{"id":1,"email":"asha@example.com","createdAt":"2026-09-26T06:40:12.123456Z"}
```

Note: no password, no hash.

**Register again with the same email:**

```
HTTP/1.1 409 Conflict
{"error":"email already registered"}
```

**Login:**

```bash
curl -s -X POST localhost:8080/login -H 'Content-Type: application/json' \
     -d '{"email":"asha@example.com","password":"correct-horse-battery"}'
```

```
{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxIiwiaWF0IjoxNzkw...<signature>"}
```

**Wrong password or unknown email:**

```
HTTP/1.1 401 Unauthorized
{"error":"invalid email or password"}
```

**Inspect the token.** Paste it into [jwt.io](https://jwt.io) (never paste production tokens into third-party sites), or decode the payload yourself:

```bash
echo 'eyJzdWIiOiIxIiwiaWF0IjoxNzkwMzgwODAwLCJleHAiOjE3OTAzODQ0MDB9' | base64 -d 2>/dev/null; echo
# {"sub":"1","iat":1790380800,"exp":1790384400}
```

You just read the "secret" claims with a one-line command, a reminder that the payload is **public**.

**Missing secret** (fail-fast configuration from Chapter 47):

```
configuration error:
JWT_SECRET is required (generate one with: head -c 48 /dev/urandom | base64)
```

---

## 17. Security checklist

| ✅ Do | ❌ Don't |
|------|---------|
| Hash passwords with **bcrypt/argon2/scrypt** | Store plain text, or plain SHA-256/MD5 |
| Keep the JWT secret in **config**, ≥ 32 random chars | Hard-code it or commit it |
| Use **HTTPS** in production (tokens and passwords cross the network!) | Send credentials over plain HTTP |
| Give tokens a **short expiry** | Issue never-expiring tokens |
| Return the **same error** for unknown email and bad password | Say "no such user" |
| **Rate-limit** login attempts (brute-force protection) | Allow unlimited guesses |
| Limit password **length**; encourage long passphrases | Enforce silly composition rules that produce weak passwords |
| Put only **non-sensitive identifiers** in the payload | Put passwords/PII in a JWT |
| **Pin the algorithm** on verification | Trust the token's `alg` |
| Add `json:"-"` to sensitive fields | Rely on remembering not to return them |
| Log **events** (login failed for email hash/ID, from IP) | Log passwords or full tokens |

**Also:** plan for **logout and revocation** (short-lived access tokens + refresh tokens, or a server-side denylist), **email verification**, **password reset**, and **multi-factor authentication**. Real products use an identity provider or well-tested library for much of this.

---

## 18. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Thinking Base64 is encryption | Secrets exposed in payloads | It's encoding only; don't put secrets in JWTs |
| 2 | Storing SHA-256 of passwords | Fast to brute-force | bcrypt/argon2 with salt and cost |
| 3 | Different errors for "user unknown"/"bad password" | User enumeration | One generic message |
| 4 | Skipping the hash comparison for unknown users | Timing side channel | Compare against a dummy hash |
| 5 | Comparing signatures with `==` | Timing attacks | `hmac.Equal` (Chapter 49) |
| 6 | Trusting the `alg` header | `alg: none`/confusion attacks | Verify the algorithm you expect |
| 7 | Tokens that never expire | Stolen token works forever | `exp` claim and short TTL |
| 8 | Hard-coded or short JWT secret | Tokens can be forged | ≥ 32 random chars from config |
| 9 | Returning the `User` struct with the hash | Hash leaks | `json:"-"`, response DTOs |
| 10 | Logging request bodies or tokens | Secrets in logs | Never log credentials |
| 11 | Padded/standard Base64 in JWTs | Interop failures | `base64.RawURLEncoding` |
| 12 | Not lower-casing emails | Duplicate accounts | Normalize on write and lookup |
| 13 | Password length unlimited | DoS via huge bcrypt inputs; 72-byte truncation surprises | Enforce a maximum |
| 14 | Check-then-insert for duplicates without a lock/constraint | Race → duplicates | One lock, or a DB unique index |

---

## 19. Exercises

### Exercise 1: Decode by hand
Decode the payload `eyJzdWIiOiI0MiIsImlhdCI6MTc5MDM4MDgwMCwiZXhwIjoxNzkwMzg0NDAwfQ` with `base64.RawURLEncoding`. What's the token's lifetime in minutes?

<details><summary>Solution</summary>

`{"sub":"42","iat":1790380800,"exp":1790384400}`; `exp − iat = 3600` seconds = **60 minutes**.
</details>

### Exercise 2: Prove tampering fails
Write a test that signs a token, changes the payload's `sub` to another user, keeps the old signature, and confirms `golang-jwt` rejects it.

<details><summary>Solution</summary>

```go
func TestTamperedPayloadIsRejected(t *testing.T) {
	tok, _ := CreateToken(secret, 42, time.Hour, time.Now())
	parts := strings.Split(tok, ".")
	forged, _ := json.Marshal(Claims{Subject: "1", IssuedAt: time.Now().Unix(), ExpiresAt: time.Now().Add(time.Hour).Unix()})
	tampered := parts[0] + "." + enc.EncodeToString(forged) + "." + parts[2]

	_, err := jwt.Parse(tampered, func(*jwt.Token) (any, error) { return secret, nil })
	if err == nil {
		t.Fatal("tampered token was accepted")
	}
}
```
</details>

### Exercise 3: Add a role claim
Add a `Role string \`json:"role"\`` claim (`"customer"` by default) to tokens and a `role` field to `User`. What must change, and why is it risky to trust the role in the token *if* it can change?

<details><summary>Solution</summary>

Add the field to `models.User`, pass it to `CreateToken` (a `Claims` field), and store it at registration (server-assigned: never accepted from the client). The risk: the token keeps the old role until it expires, so a demoted admin retains privileges until then. Mitigate with short TTLs, re-checking sensitive permissions against the database, or refresh tokens.
</details>

### Exercise 4: Cost tuning
Time `HashPassword` at bcrypt costs 8, 10, 12, 14 on your machine. Which would you choose for a login endpoint, and why?

<details><summary>Solution</summary>

Each +1 doubles the time (roughly 10 ms at cost 8, 40 ms at 10, 150 ms at 12, 600 ms at 14, depending on hardware). Aim for roughly 100–300 ms per hash on production hardware: slow enough to hurt attackers, fast enough for users and to avoid a CPU-exhaustion DoS. Reassess yearly.
</details>

### Exercise 5: Rate limit logins (design)
Sketch a per-IP+email limiter that allows 5 failed logins per 15 minutes. Where does the state live, and what changes with multiple server instances?

<details><summary>Solution</summary>

A map keyed by `ip|email` storing a counter and window start, guarded by a mutex, with periodic cleanup; return `429` with `Retry-After`. With multiple instances the counters must be shared (Redis, database) or you accept per-instance limits. A middleware around `POST /login` (route-level) is the natural place.
</details>

### Exercise 6: Change password
Design `PUT /users/me/password` (needs authentication from Chapter 49): what fields does it accept, what must it verify, and what should happen to existing tokens?

<details><summary>Solution</summary>

Accept `currentPassword` and `newPassword`; verify the current password (bcrypt), validate the new one, store a fresh hash. Existing tokens remain valid until expiry unless you track a per-user "password changed at" timestamp (or token version) and reject tokens with an earlier `iat`: a simple revocation mechanism.
</details>

### Exercise 7 (challenge): Compare hashing speeds
Benchmark `sha256.Sum256` vs `bcrypt.GenerateFromPassword` for a short password. By roughly how many orders of magnitude do they differ, and what does that mean for an attacker with a leaked table?

<details><summary>Solution</summary>

SHA-256 takes about 100–300 *nanoseconds* per hash on one core; bcrypt at cost 10 takes ~50 *milliseconds*, about **five to six orders of magnitude** slower. An attacker who could test 10⁹ SHA-256 guesses per second per GPU manages perhaps only 10³ bcrypt guesses per second: the difference between cracking a leaked table in minutes versus centuries.
</details>

---

## 20. Quiz

1. What's the difference between authentication and authorization? Which HTTP status goes with each failure?
2. Is Base64 encryption? Why does a JWT use Base64URL without padding?
3. Name three properties of a cryptographic hash.
4. Why is plain SHA-256 unsuitable for passwords?
5. What does HMAC add over a plain hash?
6. What are the three parts of a JWT?
7. Why must login return the same error for unknown emails and wrong passwords?
8. Who can read a JWT payload?

<details><summary>Answers</summary>

1. AuthN = who you are (`401`); AuthZ = what you may do (`403`).
2. No: it's reversible encoding. JWTs travel in URLs/headers, so they use the URL-safe alphabet (`-`, `_`) with no `=` padding.
3. Deterministic, fixed-size output, one-way, avalanche effect, collision resistant.
4. It's extremely fast (billions of guesses per second) and unsalted; password hashes must be slow and salted.
5. A secret key: only holders of the key can produce (or verify) a valid MAC.
6. Header, payload (claims), signature, Base64URL-joined with dots.
7. To prevent user enumeration (learning which emails have accounts).
8. Anyone who has the token: it's only encoded, not encrypted.
</details>

---

## 21. Summary

- **Authentication** proves identity (`401`); **authorization** decides permissions (`403`). Stateless APIs prove identity with a signed **token** on each request.
- **Base64URL** (`base64.RawURLEncoding`) turns bytes into URL-safe text: encoding, *not* encryption. **SHA-256** gives a fixed 32-byte deterministic one-way fingerprint. **HMAC-SHA256** adds a secret key so only key-holders can make or verify the fingerprint.
- **Passwords** are stored as **bcrypt** hashes (salted, slow, tunable cost, ≤ 72 bytes), never plain text or fast hashes. `json:"-"` keeps the hash out of responses.
- A **JWT** = `base64url(header) . base64url(payload) . base64url(HMAC(secret, header.payload))`. Claims: `sub`, `iat`, `exp`. The signature guarantees **integrity and authenticity, not secrecy**: keep secrets out of the payload.
- Our `auth` package creates tokens in ~30 lines and is **cross-checked against `golang-jwt`**; `CreateToken` takes `now` as a parameter for testability.
- Registration validates input and normalizes emails; **login** uses one generic error and a dummy-hash comparison to avoid user enumeration and timing leaks.
- The **JWT secret is required configuration** (≥ 32 chars, never in Git, redacted in `Config.String()`).

### ➡️ What's next?

[Chapter 49](49-authentication-middleware.md) closes the loop: **verifying tokens** (signature with constant-time comparison, algorithm pinning, expiry), extracting the user from the `Authorization: Bearer` header in an **authentication middleware**, and protecting `POST /products` with route-level middleware.
