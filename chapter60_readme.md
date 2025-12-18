# Chapter 60: DDD Into Code (Part 1) - User Domain

## Table of Contents
- [Introduction](#introduction)
- [Creating Domain Structure](#creating-domain-structure)
- [Understanding Entities vs Domain Logic](#understanding-entities-vs-domain-logic)
- [The Port and Service Pattern](#the-port-and-service-pattern)
- [DDD Golden Rules](#ddd-golden-rules)
- [Refactoring User Domain](#refactoring-user-domain)
- [Implementing Port Interfaces](#implementing-port-interfaces)
- [Implementing Domain Service](#implementing-domain-service)
- [Updating Repository](#updating-repository)
- [Connecting Everything](#connecting-everything)
- [Testing the Implementation](#testing-the-implementation)
- [Why This Matters](#why-this-matters)
- [Summary](#summary)
- [What's Next](#whats-next)

---

## Introduction

In Chapter 59, we learned **what** Domain-Driven Design is. Now let's see **how** to implement it in code!

**Current situation:**
```
Handlers directly access repositories
Handler → Repository → Database

Problems:
- Tight coupling
- Business logic scattered
- Hard to change
- One change affects multiple files
```

**After DDD:**
```
Clean separation with domains
Handler → Domain Service → Repository → Database

Benefits:
- Loose coupling
- Business logic in one place
- Easy to change
- One change stays in one file!
```

**In this chapter, we'll refactor the User domain!**

---

## Creating Domain Structure

### Step 1: Create Domain Folder

**Project root:**
```bash
mkdir domain
```

**Why a `domain` folder?**

```
domain/ = Central place for all domain entities
- Contains entity definitions (User, Product, Order)
- Shared across all domain packages
- No cyclic dependency issues
```

### Step 2: Move Entities to Domain

**IMPORTANT: Go vs Node.js difference!**

```
Node.js:
✓ Can create folders: domain/user/, domain/product/
✓ Allows cyclic dependencies
✓ Each can import the other

Go:
❌ Cannot create folders inside domain/ for entities
❌ Does NOT allow cyclic dependencies
✓ Must use single files: domain/user.go, domain/product.go

Why? If user/ and product/ are separate packages:
- user imports product
- product imports user
- CYCLIC DEPENDENCY ERROR!

Solution: Keep entities as single files in domain/
```

**Create entity files:**

**File: `domain/user.go`**

```go
package domain

import "time"

// User entity represents a user in the system
type User struct {
    ID        int       `json:"id" db:"id"`
    FirstName string    `json:"first_name" db:"first_name"`
    LastName  string    `json:"last_name" db:"last_name"`
    Email     string    `json:"email" db:"email"`
    Password  string    `json:"password" db:"password"`
    IsBanned  bool      `json:"is_banned" db:"is_banned"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

**File: `domain/product.go`**

```go
package domain

import "time"

// Product entity represents a product in the system
type Product struct {
    ID          int64     `json:"id" db:"id"`
    Title       string    `json:"title" db:"title"`
    Description string    `json:"description" db:"description"`
    Price       float64   `json:"price" db:"price"`
    ImageURL    string    `json:"image_url" db:"image_url"`
    CreatedAt   time.Time `json:"created_at" db:"created_at"`
    UpdatedAt   time.Time `json:"updated_at" db:"updated_at"`
}
```

**Why separate from business logic?**

```
Entity = Data structure (what it IS)
Domain Service = Business logic (what it DOES)

User entity = ID, Name, Email (structure)
User service = CreateUser, Login, UpdateProfile (behavior)
```

### Step 3: Create Domain Folders

**Now create domain-specific folders for business logic:**

```bash
mkdir domain/user
mkdir domain/product
```

**Each domain folder structure:**

```
domain/
├── user.go              # Entity (shared)
├── product.go           # Entity (shared)
├── user/
│   ├── port.go         # Interfaces (dependencies)
│   └── service.go      # Business logic
└── product/
    ├── port.go
    └── service.go
```

**What goes where:**

```
domain/*.go       → Entities (models)
domain/*/port.go  → Interface definitions (dependencies)
domain/*/service.go → Business logic implementation
```

---

## Understanding Entities vs Domain Logic

### Entities (Models)

**What they are:**
- Data structures
- Properties/attributes
- What exists (physically or logically)

**Examples:**

```go
// Entity = Something that EXISTS
type User struct {
    ID    int
    Name  string
    Email string
}

type Product struct {
    ID    int64
    Title string
    Price float64
}

type Chair struct {
    ID       int
    Material string
    Color    string
}

// Anything you can NAME is an entity!
// User, Product, Chair, Table, God, Person, Switch...
```

**Key characteristic:**
> **"An entity is anything that has an existence—physical or logical—that you can name."**

### Domain Logic (Services)

**What they are:**
- Business rules
- Operations
- What happens (behavior)

**Examples:**

```go
// Domain Logic = What you DO with entities
func CreateUser(user User) (*User, error) {
    // Validate email
    // Hash password
    // Save to database
    // Send welcome email
}

func Login(email, password string) (*User, error) {
    // Find user
    // Verify password
    // Generate JWT token
    // Log login attempt
}
```

**Separation of concerns:**

```
❌ BAD: Mix entity and logic
type User struct {
    ID    int
    Name  string
    Email string
}

func (u *User) Save() error {
    // Entity knows about database? NO!
}

✓ GOOD: Separate entity and logic
type User struct {
    ID    int
    Name  string
    Email string
}

type UserService struct {
    repo Repository
}

func (s *UserService) Save(user *User) error {
    // Service handles business logic
}
```

---

## The Port and Service Pattern

### What is Port?

**Port.go defines interfaces for dependencies.**

**Visual explanation:**

```
User Domain (Island)
     ↓
Needs to communicate with:
- Database (Repository)
- Other Domains (Post, Comment)
- Infrastructure (Redis, Email)

Port = Gateway/Interface to outside world
```

**Why "Port"?**

```
Think of a seaport:
- Ships (dependencies) come and go
- Port defines what ships can dock
- Domain doesn't care about ship details
- Just knows the interface (port)

Software Port:
- Dependencies come and go
- Port defines what dependencies can connect
- Domain doesn't care about implementation
- Just knows the interface (port)
```

### Port.go Structure

**File: `domain/user/port.go`**

```go
package user

import "yourproject/domain"

// Repository defines database operations
type Repository interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email string) (*domain.User, error)
}

// Service defines business operations
type Service interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email, password string) (*domain.User, error)
}
```

**Key points:**

```go
import "yourproject/domain"

// Use domain.User, not local User
// Because User entity lives in domain/ package
type Repository interface {
    Create(user *domain.User) (*domain.User, error)
    //           ↑
    //           From domain package
}
```

### Service.go Structure

**File: `domain/user/service.go`**

```go
package user

import "yourproject/domain"

// service implements the Service interface
type service struct {
    userRepo Repository  // Dependency on Repository interface
}

// NewService creates a new user service
func NewService(repo Repository) Service {
    return &service{
        userRepo: repo,
    }
}

// Create implements Service.Create
func (s *service) Create(user *domain.User) (*domain.User, error) {
    return s.userRepo.Create(user)
}

// FindByEmail implements Service.FindByEmail
func (s *service) FindByEmail(email, password string) (*domain.User, error) {
    return s.userRepo.FindByEmail(email)
}
```

**Notice:**
- `service` (lowercase) = struct (implementation)
- `Service` (uppercase) = interface (contract)
- Service depends on `Repository` interface, not concrete implementation!

---

## DDD Golden Rules

### Rule 1: Parent Never Accesses Child Directly

**The family analogy:**

```
Parent (Dad/Mom)
  ↓
Always GIVES to children
- Love
- Care
- Money
- Support

Never TAKES from children
```

**In code:**

```
❌ Parent accessing Child directly:

Handler (Parent)
    ↓
Directly accesses Repository (Child)

type Handler struct {
    UserRepo *database.UserRepository  // Direct access!
}

This is WRONG in DDD!
```

**Why wrong?**

```
Handler = Parent (higher level)
Repository = Child (lower level)

Parent shouldn't know child's details
Parent should only know the interface
```

### Rule 2: Child Can Access Parent

**The family analogy:**

```
Child
  ↓
Can ask parent for things
- Can ask for money
- Can ask for help
- Can access parent's resources
```

**In code:**

```
✓ Child accessing Parent:

Repository (Child)
    ↓
Uses domain.User (Parent entity)

type UserRepository struct {
    DB *sqlx.DB
}

func (r *UserRepository) Create(user *domain.User) error {
    // Repository uses domain entity
    // This is FINE!
}
```

### Rule 3: Access Through Interfaces

**The correct way:**

```
Handler (Parent)
    ↓
Accesses Service interface
    ↓
Service implements interface
    ↓
Service uses Repository interface
    ↓
Repository implements interface
    ↓
Repository accesses Database

Each layer only knows the layer directly below through interfaces!
```

**Benefits:**

```
✓ Handler doesn't know Repository exists
✓ Service doesn't know Database exists
✓ Change Repository → Handler unaffected
✓ Change Database → Service unaffected

One change = One file affected!
```

---

## Refactoring User Domain

### Step 1: Create Port Interface

**File: `domain/user/port.go`**

```go
package user

import "yourproject/domain"

// Repository defines the data layer interface
type Repository interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email string) (*domain.User, error)
}

// Service defines the business logic interface
type Service interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email, password string) (*domain.User, error)
}
```

**Why two interfaces?**

```
Repository interface:
- What domain needs from data layer
- Domain defines, repository implements
- Domain doesn't know HOW it's implemented

Service interface:
- What handlers need from domain
- Domain defines, handlers use
- Handlers don't know HOW it's implemented
```

### Step 2: Implement Service

**File: `domain/user/service.go`**

```go
package user

import "yourproject/domain"

type service struct {
    userRepo Repository
}

func NewService(repo Repository) Service {
    return &service{
        userRepo: repo,
    }
}

func (s *service) Create(user *domain.User) (*domain.User, error) {
    // Business logic here
    createdUser, err := s.userRepo.Create(user)
    if err != nil {
        return nil, err
    }
    
    if createdUser == nil {
        return nil, nil
    }
    
    return createdUser, nil
}

func (s *service) FindByEmail(email, password string) (*domain.User, error) {
    // Business logic here
    user, err := s.userRepo.FindByEmail(email)
    if err != nil {
        return nil, err
    }
    
    if user == nil {
        return nil, nil
    }
    
    // TODO: Verify password
    // This is where business logic goes!
    
    return user, nil
}
```

**Key points:**

```go
// 1. Private struct
type service struct {  // lowercase = private
    userRepo Repository
}

// 2. Public constructor returning interface
func NewService(repo Repository) Service {  // Returns interface!
    return &service{
        userRepo: repo,
    }
}

// 3. Methods implement Service interface
func (s *service) Create(user *domain.User) (*domain.User, error) {
    // Must match Service interface signature
}
```

**Why this pattern?**

```
service (struct) implements Service (interface)
↓
Can be passed where Service interface is expected
↓
NewService returns Service interface
↓
Caller doesn't know about service struct
↓
Only knows Service interface methods
```

---

## Implementing Port Interfaces

### Update Handler to Use Service

**Before (Bad):**

**File: `handler/user/handler.go`**

```go
package user

import "yourproject/database"

type Handler struct {
    UserRepo *database.UserRepository  // ❌ Direct access to child!
}
```

**After (Good):**

**File: `handler/user/handler.go`**

```go
package user

import (
    userDomain "yourproject/domain/user"
)

type Handler struct {
    userService userDomain.Service  // ✓ Access through interface!
}

func NewHandler(service userDomain.Service) *Handler {
    return &Handler{
        userService: service,
    }
}
```

**Why rename import?**

```go
import (
    userDomain "yourproject/domain/user"
)

// Problem: package name is "user"
// Handler is also in "user" package
// Conflict!

// Solution: Rename import
userDomain.Service  // Clear it's from domain/user
```

### Embed Service Interface in Port

**File: `domain/user/port.go`**

```go
package user

import "yourproject/domain"

// Service defines what handlers need
type Service interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email, password string) (*domain.User, error)
}

// Repository defines what service needs
type Repository interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email string) (*domain.User, error)
}
```

**In handler port:**

**File: `handler/user/port.go`** (New file!)

```go
package user

import (
    "yourproject/domain"
    userDomain "yourproject/domain/user"
)

// Service defines what handler needs from user domain
type Service interface {
    userDomain.Service  // Embed domain's Service interface
}
```

**What is embedding?**

```go
type Service interface {
    userDomain.Service
}

// This is like copying methods:
type Service interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email, password string) (*domain.User, error)
}

// Benefits:
// 1. Single source of truth (domain defines interface)
// 2. Handler automatically gets updates
// 3. No duplication
```

---

## Implementing Domain Service

### Service Implementation

**File: `domain/user/service.go`** (Complete)

```go
package user

import "yourproject/domain"

// service implements Service interface
type service struct {
    userRepo Repository
}

// NewService creates new user service
func NewService(repo Repository) Service {
    return &service{
        userRepo: repo,
    }
}

// Create implements Service.Create
func (s *service) Create(user *domain.User) (*domain.User, error) {
    // Call repository
    createdUser, err := s.userRepo.Create(user)
    if err != nil {
        return nil, err
    }
    
    // Check if user is nil
    if createdUser == nil {
        return nil, nil
    }
    
    return createdUser, nil
}

// FindByEmail implements Service.FindByEmail
func (s *service) FindByEmail(email, password string) (*domain.User, error) {
    // Find user by email
    user, err := s.userRepo.FindByEmail(email)
    if err != nil {
        return nil, err
    }
    
    // Check if user exists
    if user == nil {
        return nil, nil
    }
    
    // TODO: Add password verification here
    // if !verifyPassword(user.Password, password) {
    //     return nil, errors.New("invalid password")
    // }
    
    return user, nil
}
```

**Why this works:**

```go
service implements Service interface because:
1. Has Create method matching signature
2. Has FindByEmail method matching signature

Therefore:
service struct can be used where Service interface is expected!

This is polymorphism in Go!
```

---

## Updating Repository

### Update Repository to Implement Interface

**File: `database/user.go`**

```go
package database

import (
    "github.com/jmoiron/sqlx"
    "yourproject/domain"
    userDomain "yourproject/domain/user"
)

// UserRepository implements userDomain.Repository
type UserRepository struct {
    DB *sqlx.DB
}

func NewUserRepository(db *sqlx.DB) userDomain.Repository {
    return &UserRepository{DB: db}
}

// Create implements userDomain.Repository.Create
func (r *UserRepository) Create(user *domain.User) (*domain.User, error) {
    query := `
        INSERT INTO users (first_name, last_name, email, password)
        VALUES (:first_name, :last_name, :email, :password)
        RETURNING id
    `
    
    row, err := r.DB.NamedQuery(query, user)
    if err != nil {
        return nil, err
    }
    defer row.Close()
    
    if row.Next() {
        err = row.Scan(&user.ID)
        if err != nil {
            return nil, err
        }
    }
    
    return user, nil
}

// FindByEmail implements userDomain.Repository.FindByEmail
func (r *UserRepository) FindByEmail(email string) (*domain.User, error) {
    var user domain.User
    
    query := "SELECT * FROM users WHERE email = $1"
    err := r.DB.Get(&user, query, email)
    
    if err == sql.ErrNoRows {
        return nil, nil
    }
    
    if err != nil {
        return nil, err
    }
    
    return &user, nil
}
```

**Key changes:**

```go
// 1. Return interface, not struct
func NewUserRepository(db *sqlx.DB) userDomain.Repository {
    //                                  ↑
    //                                  Interface type!
    return &UserRepository{DB: db}
}

// 2. Use domain.User, not local User
func (r *UserRepository) Create(user *domain.User) (*domain.User, error) {
    //                                ↑
    //                                From domain package
}

// 3. Implements userDomain.Repository interface
// Because has Create and FindByEmail methods matching signatures
```

### Embed Repository Interface in Port

**File: `domain/user/port.go`**

```go
package user

import "yourproject/domain"

// Repository is what service needs
type Repository interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email string) (*domain.User, error)
}
```

**In database layer:**

**File: `database/user.go`**

```go
package database

import (
    "yourproject/domain"
    userDomain "yourproject/domain/user"
)

// UserRepository implements userDomain.Repository
type UserRepository struct {
    DB *sqlx.DB
}

// NewUserRepository returns interface, not struct!
func NewUserRepository(db *sqlx.DB) userDomain.Repository {
    return &UserRepository{DB: db}
}
```

**Why return interface?**

```go
// ✓ GOOD: Returns interface
func NewUserRepository(db *sqlx.DB) userDomain.Repository {
    return &UserRepository{DB: db}
}

// Caller gets interface:
repo := database.NewUserRepository(db)
// Type: userDomain.Repository (interface)
// Actual: *database.UserRepository (struct)

// ❌ BAD: Returns struct
func NewUserRepository(db *sqlx.DB) *UserRepository {
    return &UserRepository{DB: db}
}

// Caller gets struct:
repo := database.NewUserRepository(db)
// Type: *database.UserRepository (concrete)
// Tight coupling!
```

---

## Connecting Everything

### Wire Up in cmd/serve.go

**File: `cmd/serve.go`**

```go
package cmd

import (
    "fmt"
    "os"
    
    "yourproject/cmd/server"
    "yourproject/config"
    "yourproject/database"
    userDomain "yourproject/domain/user"
    "yourproject/handler/user"
    "yourproject/infra/db"
)

func Serve() {
    // 1. Load config
    config.LoadConfig()
    cfg := config.GetConfig()
    
    // 2. Connect to database
    dbConnection, err := db.NewConnection(cfg.DB)
    if err != nil {
        fmt.Println("Failed to connect to database:", err)
        os.Exit(1)
    }
    defer dbConnection.Close()
    
    // 3. Run migrations
    err = db.MigrateDB(dbConnection.DB, "./migrations")
    if err != nil {
        fmt.Println("Failed to run migrations:", err)
        os.Exit(1)
    }
    
    // 4. Create repositories
    userRepo := database.NewUserRepository(dbConnection)
    
    // 5. Create domain services
    userService := userDomain.NewService(userRepo)
    
    // 6. Create handlers
    userHandler := user.NewHandler(userService)
    
    // 7. Start server
    srv := server.NewServer(cfg, userHandler)
    srv.Start()
}
```

**Dependency flow:**

```
1. Database Connection
   ↓
2. Repository (implements domain.Repository interface)
   ↓
3. Domain Service (implements domain.Service interface)
   ↓
4. Handler (uses domain.Service interface)
   ↓
5. Router
   ↓
6. Server
```

**Type casting magic:**

```go
// Step 4: Create repository
userRepo := database.NewUserRepository(dbConnection)
// Type: userDomain.Repository (interface)
// Actual: *database.UserRepository (struct)

// Step 5: Create service
userService := userDomain.NewService(userRepo)
//                                   ↑
//                           Passes Repository interface
// Type: userDomain.Service (interface)
// Actual: *user.service (struct)

// Step 6: Create handler
userHandler := user.NewHandler(userService)
//                              ↑
//                      Passes Service interface
// Type: *user.Handler (struct)
// Contains: userDomain.Service (interface)
```

**Why this works:**

```
UserRepository implements Repository interface
→ Can be passed where Repository is expected

service implements Service interface
→ Can be passed where Service is expected

This is interface-based programming!
```

---

## Testing the Implementation

### Run the Application

```bash
go run main.go
```

**Expected output:**

```
Successfully connected to PostgreSQL!
Successfully migrated 0 migrations!
Server running on port 4000
```

### Test User Creation

**Request:**

```bash
POST http://localhost:4000/api/users/signup
Content-Type: application/json

{
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com",
  "password": "12345678"
}
```

**Response:**

```json
{
  "id": 1,
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com",
  "created_at": "2024-01-20T10:00:00Z"
}
```

### Test User Login

**Request:**

```bash
POST http://localhost:4000/api/users/login
Content-Type: application/json

{
  "email": "john@example.com",
  "password": "12345678"
}
```

**Response:**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@example.com"
  }
}
```

✓ **Everything works!**

---

## Why This Matters

### The 10,000 Files Problem

**Instructor's rant (and he's RIGHT!):**

> **"In Bangladesh's software industry, 98% is filled with 'Bokkol' and 'Mukkul' (incompetent developers) who write code where changing one file breaks 10,000 other files. This is unacceptable!"**

**Bad code example:**

```go
// One change here...
type User struct {
    ID int
    Name string
}

// ...breaks these 10,000 places:
handler1.go:  user.Name
handler2.go:  user.Name
service1.go:  user.Name
service2.go:  user.Name
utils1.go:    user.Name
utils2.go:    user.Name
// ... 9,994 more files! 😱
```

**With DDD:**

```go
// Change in domain/user.go
type User struct {
    ID int
    FullName string  // Changed from Name
}

// Breaks only:
domain/user/service.go  // ✓ ONE file!

// Everything else uses interface:
handler.go:  userService.Create(user)  // ✓ Still works!
```

### Change Impact Analysis

**Without DDD:**

```
Change repository implementation:
↓
Breaks handler (direct dependency)
↓
Breaks router (handler changed)
↓
Breaks server (router changed)
↓
Breaks tests (everything changed)

Result: 100+ files to modify! 😱
```

**With DDD:**

```
Change repository implementation:
↓
Domain service still works (interface unchanged)
↓
Handler still works (interface unchanged)
↓
Router still works (handler unchanged)
↓
Server still works (router unchanged)
↓
Tests still work (interfaces unchanged)

Result: 1 file modified! 🎉
```

### Extensibility

**Adding new feature:**

```
Without DDD:
1. Modify handler
2. Modify repository
3. Modify utilities
4. Modify middleware
5. Pray nothing breaks

With DDD:
1. Add method to Service interface
2. Implement in service
3. Done!
```

---

## Summary

### What We Accomplished

**1. Created domain structure:**
```
domain/
├── user.go           # Entity
├── product.go        # Entity
├── user/
│   ├── port.go      # Interfaces
│   └── service.go   # Business logic
└── product/
    ├── port.go
    └── service.go
```

**2. Defined interfaces in port.go:**
```go
type Service interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email, password string) (*domain.User, error)
}

type Repository interface {
    Create(user *domain.User) (*domain.User, error)
    FindByEmail(email string) (*domain.User, error)
}
```

**3. Implemented service:**
```go
type service struct {
    userRepo Repository
}

func (s *service) Create(user *domain.User) (*domain.User, error) {
    return s.userRepo.Create(user)
}
```

**4. Updated repository:**
```go
func NewUserRepository(db *sqlx.DB) userDomain.Repository {
    return &UserRepository{DB: db}
}
```

**5. Wired everything together:**
```go
userRepo := database.NewUserRepository(dbConnection)
userService := userDomain.NewService(userRepo)
userHandler := user.NewHandler(userService)
```

### DDD Golden Rules Recap

**Rule 1: Parent never accesses child directly**
```
✓ Handler → Service interface (not repository)
✓ Service → Repository interface (not database)
```

**Rule 2: Child can access parent**
```
✓ Repository → domain.User entity
✓ Service → domain.User entity
```

**Rule 3: Dependencies through interfaces**
```
✓ All dependencies defined in port.go
✓ Concrete implementations injected via constructor
```

### Benefits Achieved

**1. Loose coupling:**
```
Handler doesn't know Repository exists
Service doesn't know Database exists
```

**2. Single responsibility:**
```
Handler: HTTP logic
Service: Business logic
Repository: Data logic
```

**3. Easy testing:**
```
Mock Repository → Test Service
Mock Service → Test Handler
```

**4. Easy maintenance:**
```
Change repository → Service unaffected
Change service → Handler unaffected
One change = One file!
```

**5. Clear architecture:**
```
Everyone knows where code lives
No more hunting across 10,000 files!
```

---

## What's Next

**In Chapter 61 (Part 2), we'll:**

1. **Refactor Product domain**
   - Same pattern as User domain
   - Create product/port.go
   - Create product/service.go
   - Update product repository

2. **Add cross-domain dependencies**
   - Order domain needs User AND Product
   - How domains communicate
   - Avoiding cyclic dependencies

3. **Advanced patterns**
   - Domain events
   - Aggregate roots
   - Value objects

4. **Complete refactoring**
   - All domains following DDD
   - Clean architecture achieved
   - Production-ready structure

**The transformation:**

```
Before:
Messy spaghetti code
One change = 10,000 files broken
Impossible to maintain

After:
Clean domain-driven design
One change = 1 file modified
Easy to extend and maintain
```

**Get ready to complete the DDD transformation!** 🚀

---

**Key Insight:**

> **"If you write code where changing one file affects 10,000 other files, you're not a software engineer—you're creating technical debt that will bankrupt the company. Use DDD to ensure one change stays in one place!"**

**Remember:**
- Entities in `domain/` (shared)
- Business logic in `domain/*/service.go`
- Interfaces in `domain/*/port.go`
- Parent → Child through interfaces only
- Child → Parent is allowed

**See you in Chapter 61 where we complete the DDD refactoring!** 💪
