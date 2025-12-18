# Chapter 57: Database Configurations

## Table of Contents
- [Chapter 57: Database Configurations](#chapter-57-database-configurations)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [The Problem: Hardcoded Values](#the-problem-hardcoded-values)
    - [Current Implementation](#current-implementation)
  - [Creating DBConfig Struct](#creating-dbconfig-struct)
    - [Define Database Configuration](#define-database-configuration)
  - [Adding Environment Variables](#adding-environment-variables)
    - [Update .env File](#update-env-file)
  - [Loading Database Config](#loading-database-config)
    - [Validation and Parsing](#validation-and-parsing)
  - [Building Connection String](#building-connection-string)
    - [Update Connection Function](#update-connection-function)
  - [Integrating with Serve Command](#integrating-with-serve-command)
    - [Pass Config to Connection](#pass-config-to-connection)
  - [Troubleshooting: Environment Variable Conflicts](#troubleshooting-environment-variable-conflicts)
    - [Common Issue: System Environment Variables](#common-issue-system-environment-variables)
  - [Testing the Configuration](#testing-the-configuration)
    - [Verify Everything Works](#verify-everything-works)
    - [Test Different Environments](#test-different-environments)
  - [Summary](#summary)
    - [What We Accomplished](#what-we-accomplished)
    - [Benefits Achieved](#benefits-achieved)
    - [Key Lessons](#key-lessons)
  - [Practice Questions](#practice-questions)
    - [Question 1: Add Connection Pooling](#question-1-add-connection-pooling)
    - [Question 2: Multiple Database Support](#question-2-multiple-database-support)
    - [Question 3: Configuration from Multiple Sources](#question-3-configuration-from-multiple-sources)

---

## Introduction

Currently, our database connection has **hardcoded credentials**—a major security risk!

**Current problem:**
```go
// infra/db/connection.go
connStr := "user=postgres password=12345678 host=localhost port=5432 dbname=ecommerce"
```

**Issues:**
- ❌ Passwords exposed in source code
- ❌ Can't change database without recompiling
- ❌ Same config for dev/staging/production
- ❌ Not production-ready

**Solution:** Move all database settings to environment variables!

---

## The Problem: Hardcoded Values

### Current Implementation

**File: `infra/db/connection.go`**

```go
func NewConnection() (*sqlx.DB, error) {
    connStr := "user=postgres password=12345678 host=localhost port=5432 dbname=ecommerce sslmode=disable"
    
    db, err := sqlx.Connect("postgres", connStr)
    // ...
}
```

**Why this is bad:**

```
Scenario 1: Different environments
Development:  localhost:5432
Staging:      staging-db.company.com:5432
Production:   prod-db.company.com:5432

With hardcoded values: Need 3 different builds!
With env variables: Same build, different .env files!

Scenario 2: Password rotation
Old password: 12345678
New password: NewSecure@2024

With hardcoded: Change code, commit, deploy
With env variables: Just update .env file!

Scenario 3: Open source project
Hardcoded: Everyone sees your password!
Env variables: .env in .gitignore, safe!
```

---

## Creating DBConfig Struct

### Define Database Configuration

**File: `config/config.go`**

```go
package config

type DBConfig struct {
    Host           string `mapstructure:"DB_HOST"`
    Port           int    `mapstructure:"DB_PORT"`
    Name           string `mapstructure:"DB_NAME"`
    User           string `mapstructure:"DB_USER"`
    Password       string `mapstructure:"DB_PASSWORD"`
    EnableSSLMode  bool   `mapstructure:"DB_ENABLE_SSL_MODE"`
}

type Config struct {
    Port           int       `mapstructure:"PORT"`
    JWTSecret      string    `mapstructure:"JWT_SECRET"`
    DB             *DBConfig `mapstructure:",squash"`  // Embed DB config
}
```

**Key concepts:**

```go
`mapstructure:"DB_HOST"`
// Tells Viper: "Load from environment variable DB_HOST"

`mapstructure:",squash"`
// Flattens embedded struct fields into parent
```

**Why separate struct?**
- Organize related settings
- Easy to pass to database layer
- Can add methods (e.g., ConnectionString())
- Reusable across different database connections

---

## Adding Environment Variables

### Update .env File

**File: `.env`**

```bash
# Server Configuration
PORT=4000
JWT_SECRET=your-super-secret-key-here

# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USER=postgres
DB_PASSWORD=12345678
DB_ENABLE_SSL_MODE=false
```

**Important notes:**

```bash
# Port must be valid integer
DB_PORT=5432  # ✓ Valid
DB_PORT=abc   # ❌ Will fail to parse

# Boolean values
DB_ENABLE_SSL_MODE=false  # ✓ Lowercase
DB_ENABLE_SSL_MODE=False  # ✓ Also works
DB_ENABLE_SSL_MODE=FALSE  # ✓ Also works
DB_ENABLE_SSL_MODE=0      # ❌ Won't work

# No quotes needed for strings
DB_USER=postgres          # ✓ Correct
DB_USER="postgres"        # ✓ Also works (quotes removed)
```

**Security best practices:**

```bash
# .gitignore
.env          # ← Never commit this!
.env.local
.env.*.local

# .env.example (commit this)
DB_HOST=your_host_here
DB_PORT=5432
DB_NAME=your_database_name
DB_USER=your_username
DB_PASSWORD=your_password_here
```

---

## Loading Database Config

### Validation and Parsing

**File: `config/config.go`**

```go
func LoadConfig() {
    // Bind environment variables
    viper.BindEnv("DB_HOST")
    viper.BindEnv("DB_PORT")
    viper.BindEnv("DB_NAME")
    viper.BindEnv("DB_USER")
    viper.BindEnv("DB_PASSWORD")
    viper.BindEnv("DB_ENABLE_SSL_MODE")
    
    // Validate database config
    dbHost := viper.GetString("DB_HOST")
    if dbHost == "" {
        fmt.Println("DB_HOST is required")
        os.Exit(1)
    }
    
    dbPort := viper.GetInt("DB_PORT")
    if dbPort == 0 {
        fmt.Println("DB_PORT is required and must be a valid number")
        os.Exit(1)
    }
    
    dbName := viper.GetString("DB_NAME")
    if dbName == "" {
        fmt.Println("DB_NAME is required")
        os.Exit(1)
    }
    
    dbUser := viper.GetString("DB_USER")
    if dbUser == "" {
        fmt.Println("DB_USER is required")
        os.Exit(1)
    }
    
    dbPassword := viper.GetString("DB_PASSWORD")
    if dbPassword == "" {
        fmt.Println("DB_PASSWORD is required")
        os.Exit(1)
    }
    
    // SSL mode is optional, defaults to false
    enableSSL := viper.GetBool("DB_ENABLE_SSL_MODE")
    
    // Build DB config
    dbConfig := &DBConfig{
        Host:          dbHost,
        Port:          dbPort,
        Name:          dbName,
        User:          dbUser,
        Password:      dbPassword,
        EnableSSLMode: enableSSL,
    }
    
    // Create main config
    config = &Config{
        Port:      viper.GetInt("PORT"),
        JWTSecret: viper.GetString("JWT_SECRET"),
        DB:        dbConfig,
    }
}
```

**Simplified version using Viper's Unmarshal:**

```go
func LoadConfig() {
    viper.AutomaticEnv()  // Automatically read env vars
    
    config = &Config{}
    err := viper.Unmarshal(config)
    if err != nil {
        fmt.Printf("Unable to decode config: %v\n", err)
        os.Exit(1)
    }
    
    // Validate
    if config.DB.Host == "" {
        fmt.Println("DB_HOST is required")
        os.Exit(1)
    }
    // ... other validations
}
```

---

## Building Connection String

### Update Connection Function

**File: `infra/db/connection.go`**

```go
package db

import (
    "fmt"
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
    "yourproject/config"
)

func NewConnection(cfg *config.DBConfig) (*sqlx.DB, error) {
    // Build connection string from config
    connStr := fmt.Sprintf(
        "user=%s password=%s host=%s port=%d dbname=%s",
        cfg.User,
        cfg.Password,
        cfg.Host,
        cfg.Port,
        cfg.Name,
    )
    
    // Add SSL mode if disabled
    if !cfg.EnableSSLMode {
        connStr += " sslmode=disable"
    }
    
    // Connect to database
    db, err := sqlx.Connect("postgres", connStr)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to database: %w", err)
    }
    
    fmt.Println("Successfully connected to PostgreSQL!")
    return db, nil
}
```

**Connection string format:**

```
PostgreSQL connection string parts:
user=postgres           ← Database user
password=12345678       ← User password
host=localhost          ← Server address
port=5432               ← Server port
dbname=ecommerce        ← Database name
sslmode=disable         ← SSL setting (optional)

Full string:
"user=postgres password=12345678 host=localhost port=5432 dbname=ecommerce sslmode=disable"
```

**SSL Mode options:**

```go
sslmode=disable   // No SSL (development only!)
sslmode=require   // SSL required
sslmode=verify-ca // Verify server certificate
sslmode=verify-full // Full verification (production)
```

---

## Integrating with Serve Command

### Pass Config to Connection

**File: `cmd/serve.go`**

```go
package cmd

import (
    "fmt"
    "os"
    
    "yourproject/cmd/server"
    "yourproject/config"
    "yourproject/database"
    "yourproject/handler/product"
    "yourproject/handler/user"
    "yourproject/infra/db"
)

func Serve() {
    // Load config (reads .env and environment variables)
    config.LoadConfig()
    cfg := config.GetConfig()
    
    // Connect to database with config
    dbConnection, err := db.NewConnection(cfg.DB)  // ← Pass DB config
    if err != nil {
        fmt.Println("Failed to connect to database:", err)
        os.Exit(1)
    }
    defer dbConnection.Close()
    
    // Create repositories
    userRepo := database.NewUserRepository(dbConnection)
    productRepo := database.NewProductRepository(dbConnection)
    
    // Create handlers
    userHandler := user.NewHandler(userRepo)
    productHandler := product.NewHandler(productRepo)
    
    // Start server
    srv := server.NewServer(cfg, userHandler, productHandler)
    srv.Start()
}
```

**Dependency flow:**

```
.env file
    ↓
config.LoadConfig()
    ↓
config.GetConfig() → cfg
    ↓
db.NewConnection(cfg.DB)
    ↓
Database connection
    ↓
Repositories
    ↓
Handlers
    ↓
Server
```

---

## Troubleshooting: Environment Variable Conflicts

### Common Issue: System Environment Variables

**Problem:** Sometimes your system has environment variables with the same names!

**Example scenario:**

```bash
# Your .env file
DB_USER=postgres

# But your system already has:
$ echo $USER
habibur_rahman

# Viper reads system USER variable instead of DB_USER!
```

**Why this happens:**

```go
viper.BindEnv("USER")  // Binds to system USER variable

// When you read:
user := viper.GetString("USER")
// Gets "habibur_rahman" from system, not "postgres" from .env!
```

**Solution: Use unique prefixes**

```bash
# ✓ Good naming (no conflicts)
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=12345678

# ❌ Bad naming (conflicts with system)
HOST=localhost         # System has $HOST
USER=postgres          # System has $USER
PASSWORD=12345678      # System has $PASSWORD
```

**Testing for conflicts:**

```bash
# Check if variable exists in system
echo $USER
echo $HOST
echo $HOME
echo $PATH

# If these show values, DON'T use these names in .env!
```

**Debug configuration:**

```go
func LoadConfig() {
    // ... load config
    
    // Debug: Print loaded config
    fmt.Printf("Loaded DB Config: %+v\n", config.DB)
    
    // This helps identify which values were loaded
}
```

**Output example:**

```
Loaded DB Config: {
    Host:localhost 
    Port:5432 
    Name:ecommerce 
    User:habibur_rahman    ← WRONG! Should be "postgres"
    Password:12345678 
    EnableSSLMode:false
}
```

**Fix by renaming:**

```bash
# Change DB_USER to avoid USER conflict
DB_USER=postgres

# Make sure config reads DB_USER not USER
viper.BindEnv("DB_USER")  # ✓ Correct
viper.BindEnv("USER")     # ❌ Wrong (system conflict)
```

---

## Testing the Configuration

### Verify Everything Works

**Step 1: Check .env file**

```bash
cat .env
```

**Expected output:**
```
PORT=4000
JWT_SECRET=your-secret-key
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USER=postgres
DB_PASSWORD=12345678
DB_ENABLE_SSL_MODE=false
```

**Step 2: Run server**

```bash
go run main.go
```

**Expected output:**
```
Successfully connected to PostgreSQL!
Server running on port 4000
```

**Step 3: Test with Postman**

```
POST http://localhost:4000/api/products
Authorization: Bearer <your_jwt_token>
Content-Type: application/json

{
  "title": "Test Product",
  "price": 99.99
}
```

**Response:**
```json
{
  "id": 1,
  "title": "Test Product",
  "price": 99.99,
  "created_at": "2024-01-20T10:00:00Z"
}
```

**Step 4: Verify in database**

```sql
SELECT * FROM products;
```

✓ Product should appear in database!

### Test Different Environments

**Development (.env)**
```bash
DB_HOST=localhost
DB_NAME=ecommerce_dev
```

**Staging (.env.staging)**
```bash
DB_HOST=staging-db.company.com
DB_NAME=ecommerce_staging
DB_ENABLE_SSL_MODE=true
```

**Production (.env.production)**
```bash
DB_HOST=prod-db.company.com
DB_NAME=ecommerce_prod
DB_ENABLE_SSL_MODE=true
```

**Run with different configs:**

```bash
# Development
go run main.go

# Staging
cp .env.staging .env
go run main.go

# Production
cp .env.production .env
go build -o server
./server
```

---

## Summary

### What We Accomplished

**1. Created DBConfig struct**
```go
type DBConfig struct {
    Host          string
    Port          int
    Name          string
    User          string
    Password      string
    EnableSSLMode bool
}
```

**2. Moved to environment variables**
```bash
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USER=postgres
DB_PASSWORD=12345678
DB_ENABLE_SSL_MODE=false
```

**3. Dynamic connection string**
```go
connStr := fmt.Sprintf(
    "user=%s password=%s host=%s port=%d dbname=%s",
    cfg.User, cfg.Password, cfg.Host, cfg.Port, cfg.Name,
)
```

**4. Environment-specific configs**
- Development: localhost
- Staging: staging-db
- Production: prod-db

### Benefits Achieved

| Before | After |
|--------|-------|
| Hardcoded credentials | Environment variables |
| Same config everywhere | Different per environment |
| Passwords in source code | Passwords in .env (gitignored) |
| Recompile to change DB | Just update .env |
| Not production-ready | Production-ready ✓ |

### Key Lessons

**1. Prefix environment variables**
```bash
DB_USER=postgres    # ✓ No conflicts
USER=postgres       # ❌ System conflict
```

**2. Validate all required configs**
```go
if cfg.DB.Host == "" {
    fmt.Println("DB_HOST is required")
    os.Exit(1)
}
```

**3. Use .env.example for documentation**
```bash
# Commit .env.example (template)
# Never commit .env (actual values)
```

**4. SSL mode for production**
```bash
# Development
DB_ENABLE_SSL_MODE=false

# Production
DB_ENABLE_SSL_MODE=true
```

---

## Practice Questions

### Question 1: Add Connection Pooling

**Question:** Add connection pool configuration to control maximum connections, idle connections, and connection lifetime.

**Requirements:**
1. Add pool settings to DBConfig
2. Configure connection pool limits
3. Add health check endpoint

<details>
<summary>Click to see answer</summary>

**Step 1: Update DBConfig**

```go
type DBConfig struct {
    Host             string `mapstructure:"DB_HOST"`
    Port             int    `mapstructure:"DB_PORT"`
    Name             string `mapstructure:"DB_NAME"`
    User             string `mapstructure:"DB_USER"`
    Password         string `mapstructure:"DB_PASSWORD"`
    EnableSSLMode    bool   `mapstructure:"DB_ENABLE_SSL_MODE"`
    MaxOpenConns     int    `mapstructure:"DB_MAX_OPEN_CONNS"`
    MaxIdleConns     int    `mapstructure:"DB_MAX_IDLE_CONNS"`
    ConnMaxLifetime  int    `mapstructure:"DB_CONN_MAX_LIFETIME"` // minutes
}
```

**Step 2: Add to .env**

```bash
# Connection Pool Settings
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME=15
```

**Step 3: Configure pool**

```go
func NewConnection(cfg *config.DBConfig) (*sqlx.DB, error) {
    connStr := fmt.Sprintf(
        "user=%s password=%s host=%s port=%d dbname=%s",
        cfg.User, cfg.Password, cfg.Host, cfg.Port, cfg.Name,
    )
    
    if !cfg.EnableSSLMode {
        connStr += " sslmode=disable"
    }
    
    db, err := sqlx.Connect("postgres", connStr)
    if err != nil {
        return nil, err
    }
    
    // Configure connection pool
    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    db.SetConnMaxLifetime(time.Duration(cfg.ConnMaxLifetime) * time.Minute)
    
    fmt.Printf("Connection pool configured: Max=%d, Idle=%d, Lifetime=%dm\n",
        cfg.MaxOpenConns, cfg.MaxIdleConns, cfg.ConnMaxLifetime)
    
    return db, nil
}
```

**Step 4: Health check endpoint**

```go
func (h *Handler) HealthCheck(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()
    
    err := h.DB.PingContext(ctx)
    if err != nil {
        http.Error(w, "Database unhealthy", http.StatusServiceUnavailable)
        return
    }
    
    stats := h.DB.Stats()
    
    response := map[string]interface{}{
        "status": "healthy",
        "database": map[string]interface{}{
            "open_connections": stats.OpenConnections,
            "in_use":          stats.InUse,
            "idle":            stats.Idle,
            "max_open":        stats.MaxOpenConnections,
        },
    }
    
    json.NewEncoder(w).Encode(response)
}
```

**Test:**
```bash
GET http://localhost:4000/health

Response:
{
  "status": "healthy",
  "database": {
    "open_connections": 3,
    "in_use": 1,
    "idle": 2,
    "max_open": 25
  }
}
```

</details>

---

### Question 2: Multiple Database Support

**Question:** Support multiple databases (PostgreSQL for app data, MongoDB for logs).

<details>
<summary>Click to see answer</summary>

```go
type Config struct {
    Port           int
    JWTSecret      string
    PostgresDB     *DBConfig
    MongoDB        *MongoConfig
}

type MongoConfig struct {
    URI      string `mapstructure:"MONGO_URI"`
    Database string `mapstructure:"MONGO_DATABASE"`
}
```

**.env:**
```bash
# PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USER=postgres
DB_PASSWORD=12345678

# MongoDB
MONGO_URI=mongodb://localhost:27017
MONGO_DATABASE=ecommerce_logs
```

</details>

---

### Question 3: Configuration from Multiple Sources

**Question:** Load config from .env, environment variables, and command-line flags with proper precedence.

<details>
<summary>Click to see answer</summary>

```go
func LoadConfig() {
    // 1. Set defaults
    viper.SetDefault("DB_PORT", 5432)
    viper.SetDefault("DB_MAX_OPEN_CONNS", 25)
    
    // 2. Read from .env file
    viper.SetConfigFile(".env")
    viper.ReadInConfig()
    
    // 3. Read from environment variables (higher priority)
    viper.AutomaticEnv()
    
    // 4. Read from command-line flags (highest priority)
    pflag.String("db-host", "", "Database host")
    pflag.Int("db-port", 0, "Database port")
    pflag.Parse()
    viper.BindPFlags(pflag.CommandLine)
    
    // Precedence: flags > env vars > .env file > defaults
}
```

**Usage:**
```bash
# Use .env file
go run main.go

# Override with env var
DB_HOST=prod-db.com go run main.go

# Override with flag
go run main.go --db-host=prod-db.com --db-port=5433
```

</details>

---

**Congratulations!** Your database configuration is now production-ready with environment variables! 🎉
