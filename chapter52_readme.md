# Chapter 52: Database Connection Setup - PostgreSQL Integration

## Table of Contents
- [Introduction](#introduction)
- [Understanding Infrastructure](#understanding-infrastructure)
- [Creating Infrastructure Folder Structure](#creating-infrastructure-folder-structure)
- [Database Libraries Overview](#database-libraries-overview)
- [Installing Required Libraries](#installing-required-libraries)
- [Connection String Format](#connection-string-format)
- [Implementing Database Connection](#implementing-database-connection)
- [Testing the Connection](#testing-the-connection)
- [Understanding the Connection Flow](#understanding-the-connection-flow)
- [Summary](#summary)
- [Practice Questions](#practice-questions)

---

## Introduction

We've been using **in-memory arrays** to store users and products. This is fine for learning, but **production applications need real databases**!

Today, we'll integrate **PostgreSQL** into our project. This is a short, focused class—**no time to waste**, let's get straight to coding!

**What We'll Do:**
1. Create infrastructure folder structure
2. Install database libraries
3. Write connection code
4. Connect to PostgreSQL

**Prerequisites:**
- PostgreSQL installed (covered in previous chapter)
- pgAdmin 4 set up
- Database created (we'll use `ecommerce`)

---

## Understanding Infrastructure

### What Is Infrastructure?

**Infrastructure** = External services your application depends on

Think of it as **utilities** for your app:

```
Your Application
      ↓
Needs infrastructure:
├─ Database (PostgreSQL, MySQL)
├─ Cache (Redis)
├─ Message Queue (RabbitMQ, Kafka)
├─ File Storage (S3, MinIO)
└─ Search Engine (Elasticsearch)
```

**Infrastructure components:**

```
1. Database
   └─ Store data permanently
   └─ Examples: PostgreSQL, MySQL, MongoDB

2. Redis
   └─ Cache data for fast access
   └─ Store temporary data

3. RabbitMQ
   └─ Message broker
   └─ Queue tasks for async processing

4. Kafka
   └─ Event streaming
   └─ Real-time data pipelines

5. File Storage
   └─ Store images, videos, documents
   └─ S3, MinIO, local storage
```

**Why separate infrastructure folder?**
- **Organization** - All external services in one place
- **Isolation** - Easy to swap implementations
- **Clarity** - Know what external dependencies exist

---

## Creating Infrastructure Folder Structure

### Folder Structure

**Create this structure:**

```
project/
├─ cmd/
├─ config/
├─ handler/
├─ middleware/
├─ infra/              ← NEW
│  ├─ db/              ← Database
│  ├─ redis/           ← Redis (if needed)
│  ├─ mq/              ← Message queue (if needed)
│  └─ kafka/           ← Kafka (if needed)
├─ database/
└─ main.go
```

**Create the folders:**

```bash
mkdir -p infra/db
```

**Why this structure?**

```
infra/
├─ db/           ← All database code
├─ redis/        ← All Redis code
├─ mq/           ← All message queue code
└─ kafka/        ← All Kafka code

Each infrastructure component isolated!
```

**Real-world example:**

```
Facebook architecture:
├─ Database (user data, posts)
├─ Redis (session cache)
├─ Message Queue (notifications)
├─ File Storage (photos, videos)
└─ Search (Elasticsearch for posts)

Each has its own folder/service!
```

---

## Database Libraries Overview

### Available Go Database Libraries

**Three main options for PostgreSQL:**

```
1. database/sql + pq
   └─ Standard library approach
   └─ Most control, most code
   └─ Speed: ⭐⭐⭐⭐⭐

2. sqlx
   └─ Extension of database/sql
   └─ More features, easier to use
   └─ Speed: ⭐⭐⭐⭐
   └─ Developer-friendly: ⭐⭐⭐⭐

3. GORM
   └─ Full ORM (Object-Relational Mapping)
   └─ Very easy, lots of features
   └─ Speed: ⭐⭐⭐
   └─ Developer-friendly: ⭐⭐⭐⭐⭐
```

### Comparison Table

| Library | Speed | Developer Friendly | Power | Complexity |
|---------|-------|-------------------|-------|------------|
| **database/sql** | Fastest | ⭐⭐ | Full control | High |
| **sqlx** | Fast | ⭐⭐⭐⭐ | Good balance | Medium |
| **GORM** | Slower | ⭐⭐⭐⭐⭐ | Easy, heavy | Low |

### Why sqlx?

**We're using sqlx because:**

```
✓ Fast (almost as fast as raw database/sql)
✓ Developer-friendly (easier than raw SQL)
✓ Powerful (can write any query)
✓ Flexible (can drop to raw SQL when needed)
✓ Popular (widely used in production)
```

**database/sql:**
```go
// More code, more manual work
rows, err := db.Query("SELECT id, name, age FROM users")
for rows.Next() {
    var id int
    var name string
    var age int
    err := rows.Scan(&id, &name, &age)  // Manual scanning
    // ...
}
```

**sqlx:**
```go
// Less code, automatic scanning
var users []User
err := db.Select(&users, "SELECT * FROM users")  // Auto-scan into struct!
```

**GORM:**
```go
// Even less code, but slower
var users []User
db.Find(&users)  // Very easy, but less control
```

**Our choice: sqlx** - Best balance of speed and ease of use!

---

## Installing Required Libraries

### Install sqlx

**sqlx** = Extension of database/sql with extra features

```bash
go get github.com/jmoiron/sqlx
```

**What sqlx provides:**
- Automatic struct scanning
- Named parameters
- Better error messages
- Convenience methods

### Install PostgreSQL Driver

**pq** = PostgreSQL driver for Go

```bash
go get github.com/lib/pq
```

**What pq provides:**
- PostgreSQL protocol implementation
- Connection to PostgreSQL server
- Execute queries against PostgreSQL

**Check go.mod:**

After installation, your `go.mod` should include:

```go
module yourproject

go 1.21

require (
    github.com/jmoiron/sqlx v1.3.5
    github.com/lib/pq v1.10.9
    // ... other dependencies
)
```

**Why two libraries?**

```
sqlx
  └─ High-level API (easy to use)
      ↓
database/sql
  └─ Standard interface
      ↓
pq driver
  └─ PostgreSQL-specific implementation
```

Think of it like:
- **sqlx** = Steering wheel (what you use)
- **database/sql** = Car mechanism (standard interface)
- **pq** = Engine (PostgreSQL-specific)

---

## Connection String Format

### PostgreSQL Connection String

**Format:**
```
user=<username> password=<password> host=<host> port=<port> dbname=<database>
```

**Our connection string:**
```
user=postgres password=123456789 host=localhost port=5432 dbname=ecommerce
```

**Components explained:**

| Component | Value | Description |
|-----------|-------|-------------|
| **user** | postgres | Default PostgreSQL user |
| **password** | 123456789 | Your password (set during install) |
| **host** | localhost | Your machine (127.0.0.1) |
| **port** | 5432 | Default PostgreSQL port |
| **dbname** | ecommerce | Database name to connect to |

**Important notes:**

```
✓ Spaces between components required
✓ No quotes around values
✓ Order doesn't matter
✓ All components required
```

**Common mistakes:**

```go
// ❌ Missing spaces
"user=postgrespassword=123host=localhost"

// ❌ Wrong separator
"user=postgres,password=123,host=localhost"

// ❌ Quotes around values
"user='postgres' password='123' host='localhost'"

// ✓ Correct format
"user=postgres password=123 host=localhost port=5432 dbname=ecommerce"
```

### Creating Database

**Before connecting, create database in pgAdmin:**

```sql
CREATE DATABASE ecommerce;
```

**Or via terminal:**

```bash
psql -U postgres
CREATE DATABASE ecommerce;
\q
```

**Verify database exists:**

```
pgAdmin 4
  └─ Servers
      └─ PostgreSQL 15
          └─ Databases
              └─ ecommerce ← Should see this
                  └─ Schemas
                      └─ public
                          └─ Tables (empty for now)
```

---

## Implementing Database Connection

### File Structure

**File: `infra/db/connection.go`**

```go
package db

import (
    "fmt"
    "os"
    
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"  // PostgreSQL driver
)
```

**Why `_ "github.com/lib/pq"`?**

The underscore means:
- Import for side effects only
- Don't use it directly in code
- Driver registers itself automatically

### GetConnectionString Function

**Purpose:** Build connection string from credentials

```go
package db

import (
    "fmt"
    
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

// GetConnectionString returns PostgreSQL connection string
func GetConnectionString() string {
    user := "postgres"
    password := "123456789"
    host := "localhost"
    port := "5432"
    dbname := "ecommerce"
    
    return fmt.Sprintf(
        "user=%s password=%s host=%s port=%s dbname=%s",
        user, password, host, port, dbname,
    )
}
```

**Better approach with environment variables:**

```go
func GetConnectionString() string {
    user := getEnv("DB_USER", "postgres")
    password := getEnv("DB_PASSWORD", "123456789")
    host := getEnv("DB_HOST", "localhost")
    port := getEnv("DB_PORT", "5432")
    dbname := getEnv("DB_NAME", "ecommerce")
    
    return fmt.Sprintf(
        "user=%s password=%s host=%s port=%s dbname=%s",
        user, password, host, port, dbname,
    )
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

### NewConnection Function

**Purpose:** Connect to PostgreSQL and return database client

```go
// NewConnection creates a new database connection
func NewConnection() (*sqlx.DB, error) {
    // Get connection string
    dbSource := GetConnectionString()
    
    // Connect to PostgreSQL
    dbConnection, err := sqlx.Connect("postgres", dbSource)
    if err != nil {
        fmt.Println("Error connecting to database:", err)
        return nil, err
    }
    
    // Success
    return dbConnection, nil
}
```

**Understanding the code:**

```go
sqlx.Connect("postgres", dbSource)
     ↑              ↑
     |              |
   Driver      Connection string
```

**Parameters:**
1. `"postgres"` - Driver name (tells sqlx to use pq driver)
2. `dbSource` - Connection string with credentials

**Returns:**
- `*sqlx.DB` - Database client (use this to execute queries)
- `error` - Error if connection fails

### Complete connection.go File

**File: `infra/db/connection.go`**

```go
package db

import (
    "fmt"
    "os"
    
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"  // PostgreSQL driver
)

// GetConnectionString returns PostgreSQL connection string
func GetConnectionString() string {
    user := getEnv("DB_USER", "postgres")
    password := getEnv("DB_PASSWORD", "123456789")
    host := getEnv("DB_HOST", "localhost")
    port := getEnv("DB_PORT", "5432")
    dbname := getEnv("DB_NAME", "ecommerce")
    
    return fmt.Sprintf(
        "user=%s password=%s host=%s port=%s dbname=%s",
        user, password, host, port, dbname,
    )
}

// NewConnection creates a new database connection
func NewConnection() (*sqlx.DB, error) {
    // Get connection string
    dbSource := GetConnectionString()
    
    // Connect to PostgreSQL
    dbConnection, err := sqlx.Connect("postgres", dbSource)
    if err != nil {
        fmt.Println("Error connecting to database:", err)
        return nil, err
    }
    
    fmt.Println("Successfully connected to PostgreSQL!")
    return dbConnection, nil
}

// Helper function to get environment variable with default
func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

---

## Testing the Connection

### Update cmd/serve.go

**File: `cmd/serve.go`**

```go
package cmd

import (
    "fmt"
    "os"
    
    "yourproject/cmd/server"
    "yourproject/config"
    "yourproject/handler/product"
    "yourproject/handler/user"
    "yourproject/infra/db"  // ← Import database package
)

func Serve() {
    // Load config
    config.LoadConfig()
    cfg := config.GetConfig()
    
    // Connect to database
    dbConnection, err := db.NewConnection()
    if err != nil {
        fmt.Println("Failed to connect to database:", err)
        os.Exit(1)  // Exit if database connection fails
    }
    defer dbConnection.Close()  // Close connection when done
    
    // Create handlers
    userHandler := user.NewHandler()
    productHandler := product.NewHandler()
    
    // Create server
    srv := server.NewServer(
        cfg,
        userHandler,
        productHandler,
    )
    
    // Start server
    srv.Start()
}
```

**Key additions:**

```go
// Connect to database
dbConnection, err := db.NewConnection()
if err != nil {
    fmt.Println("Failed to connect to database:", err)
    os.Exit(1)  // Exit application if no database
}
defer dbConnection.Close()  // Always close connection
```

### Run the Application

```bash
go run main.go
```

**Expected output:**

```
Successfully connected to PostgreSQL!
Server running on port 4000
```

**If connection fails:**

```
Error connecting to database: dial tcp [::1]:5432: connect: connection refused
Failed to connect to database: <error details>
```

**Common errors and fixes:**

| Error | Cause | Fix |
|-------|-------|-----|
| `connection refused` | PostgreSQL not running | Start PostgreSQL service |
| `password authentication failed` | Wrong password | Check password in connection string |
| `database "ecommerce" does not exist` | Database not created | Create database in pgAdmin |
| `role "postgres" does not exist` | Wrong username | Check username in connection string |

---

## Understanding the Connection Flow

### Visual Flow

```
Application
    ↓
1. GetConnectionString()
    └─ Returns: "user=postgres password=123..."
    ↓
2. sqlx.Connect("postgres", connectionString)
    ↓
    ├─ Uses pq driver (because "postgres")
    ├─ Connects to localhost:5432
    ├─ Authenticates with user/password
    └─ Selects database "ecommerce"
    ↓
3. Returns *sqlx.DB (database client)
    ↓
4. Use client to execute queries
```

### Connection Object Lifecycle

```go
// 1. Create connection
db, err := db.NewConnection()

// 2. Use connection (entire app lifetime)
//    - Execute queries
//    - Insert data
//    - Update data
//    - Delete data

// 3. Close connection (when app shuts down)
defer db.Close()
```

**Important:** 
- Create connection **once** at startup
- **Reuse** same connection throughout app
- **Close** when application exits

### Connection Pooling

**sqlx automatically manages connection pool:**

```
Database
    ↑
Connection Pool
├─ Connection 1 (idle)
├─ Connection 2 (in use)
├─ Connection 3 (in use)
├─ Connection 4 (idle)
└─ Connection 5 (in use)
    ↑
Multiple Goroutines
```

**Benefits:**
- **Reuse connections** - Don't create new connection for each query
- **Limit connections** - Don't overwhelm database
- **Automatic** - sqlx handles it for you

**Configure pool (optional):**

```go
db, err := db.NewConnection()
if err != nil {
    return nil, err
}

// Configure connection pool
db.SetMaxOpenConns(25)      // Max 25 connections
db.SetMaxIdleConns(5)       // Keep 5 idle connections
db.SetConnMaxLifetime(5 * time.Minute)  // Recycle after 5 min
```

---

## Summary

### What We Accomplished

**1. Infrastructure Organization**
```
infra/
└─ db/
   └─ connection.go  ← Database connection code
```

**2. Installed Libraries**
```
✓ sqlx - Database operations
✓ pq - PostgreSQL driver
```

**3. Connection Implementation**
```go
GetConnectionString() → "user=postgres password=..."
NewConnection() → *sqlx.DB (database client)
```

**4. Integration with App**
```go
cmd/serve.go
├─ Connect to database on startup
├─ Exit if connection fails
└─ Close connection on shutdown
```

### Key Concepts

**Connection String:**
```
Format: user=X password=Y host=Z port=P dbname=D
Example: user=postgres password=123 host=localhost port=5432 dbname=ecommerce
```

**Database Client:**
```
*sqlx.DB = Object used to execute queries
Created by: sqlx.Connect()
Lifetime: Entire application
```

**Error Handling:**
```
Connection fails → Print error → Exit application
Can't run without database!
```

### Connection Flow Summary

```
1. Application starts
   ↓
2. Load config
   ↓
3. Connect to database
   ├─ Build connection string
   ├─ Call sqlx.Connect()
   ├─ Get database client
   └─ Store for later use
   ↓
4. Start HTTP server
   ↓
5. Use database client in handlers
   ↓
6. Application shuts down → Close connection
```

### Next Steps

In the next chapter, we'll:
1. **Create tables** - Users, Products tables
2. **Write queries** - INSERT, SELECT, UPDATE, DELETE
3. **Replace in-memory storage** - Use real database
4. **Implement Repository Pattern** - Clean data access

We now have database connection ready—next we'll actually **use it**! 🚀

---

## Practice Questions

### Question 1: Environment Variables for Connection

**Question:**
Modify the connection code to read credentials from environment variables. Create a `.env` file with database credentials and load them in `GetConnectionString()`.

<details>
<summary>Click to see answer</summary>

**Answer:**

**Step 1: Update .env file**

**File: `.env`**
```env
HTTP_PORT=4000
JWT_SECRET=your-secret-key

# Database credentials
DB_USER=postgres
DB_PASSWORD=123456789
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
```

**Step 2: Update config.go**

**File: `config/config.go`**
```go
package config

import (
    "os"
    "strconv"
    
    "github.com/joho/godotenv"
)

type Config struct {
    HTTPPort  int
    JWTSecret string
    
    // Database config
    DBUser     string
    DBPassword string
    DBHost     string
    DBPort     string
    DBName     string
}

var cfg *Config

func LoadConfig() {
    godotenv.Load()
    
    cfg = &Config{
        HTTPPort:   getEnvInt("HTTP_PORT", 4000),
        JWTSecret:  getEnv("JWT_SECRET", "secret"),
        
        // Load database config
        DBUser:     getEnv("DB_USER", "postgres"),
        DBPassword: getEnv("DB_PASSWORD", ""),
        DBHost:     getEnv("DB_HOST", "localhost"),
        DBPort:     getEnv("DB_PORT", "5432"),
        DBName:     getEnv("DB_NAME", "ecommerce"),
    }
}

func GetConfig() *Config {
    if cfg == nil {
        LoadConfig()
    }
    return cfg
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intValue, err := strconv.Atoi(value); err == nil {
            return intValue
        }
    }
    return defaultValue
}
```

**Step 3: Update connection.go to use config**

**File: `infra/db/connection.go`**
```go
package db

import (
    "fmt"
    
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
    
    "yourproject/config"
)

// GetConnectionString builds connection string from config
func GetConnectionString(cfg *config.Config) string {
    return fmt.Sprintf(
        "user=%s password=%s host=%s port=%s dbname=%s sslmode=disable",
        cfg.DBUser,
        cfg.DBPassword,
        cfg.DBHost,
        cfg.DBPort,
        cfg.DBName,
    )
}

// NewConnection creates database connection using config
func NewConnection(cfg *config.Config) (*sqlx.DB, error) {
    // Build connection string from config
    dbSource := GetConnectionString(cfg)
    
    // Connect
    dbConnection, err := sqlx.Connect("postgres", dbSource)
    if err != nil {
        fmt.Println("Error connecting to database:", err)
        return nil, err
    }
    
    fmt.Println("Successfully connected to PostgreSQL!")
    return dbConnection, nil
}
```

**Step 4: Update serve.go**

**File: `cmd/serve.go`**
```go
func Serve() {
    // Load config
    config.LoadConfig()
    cfg := config.GetConfig()
    
    // Connect to database using config
    dbConnection, err := db.NewConnection(cfg)  // ← Pass config
    if err != nil {
        fmt.Println("Failed to connect to database:", err)
        os.Exit(1)
    }
    defer dbConnection.Close()
    
    // ... rest of code
}
```

**Benefits:**
- ✓ No hardcoded credentials
- ✓ Easy to change per environment (dev, staging, prod)
- ✓ Secure (don't commit .env to git)
- ✓ Follows 12-factor app principles

**Security tip:**

**File: `.gitignore`**
```
.env
*.env
```

Never commit `.env` file to version control!

</details>

---

### Question 2: Connection Health Check

**Question:**
Implement a health check function that verifies the database connection is alive. The function should ping the database and return an error if connection is dead.

<details>
<summary>Click to see answer</summary>

**Answer:**

**File: `infra/db/connection.go`**

```go
package db

import (
    "context"
    "fmt"
    "time"
    
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

// NewConnection creates and verifies database connection
func NewConnection() (*sqlx.DB, error) {
    dbSource := GetConnectionString()
    
    // Connect
    dbConnection, err := sqlx.Connect("postgres", dbSource)
    if err != nil {
        fmt.Println("Error connecting to database:", err)
        return nil, err
    }
    
    // Verify connection is working
    err = HealthCheck(dbConnection)
    if err != nil {
        fmt.Println("Database health check failed:", err)
        dbConnection.Close()
        return nil, err
    }
    
    fmt.Println("Successfully connected to PostgreSQL!")
    return dbConnection, nil
}

// HealthCheck verifies database connection is alive
func HealthCheck(db *sqlx.DB) error {
    // Create context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    // Ping database
    err := db.PingContext(ctx)
    if err != nil {
        return fmt.Errorf("database ping failed: %w", err)
    }
    
    return nil
}

// IsConnectionAlive checks if connection is still alive
func IsConnectionAlive(db *sqlx.DB) bool {
    err := HealthCheck(db)
    return err == nil
}
```

**Usage in application:**

```go
// In a background goroutine
func MonitorDatabaseConnection(db *sqlx.DB) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        if !db.IsConnectionAlive(db) {
            fmt.Println("⚠️  Database connection lost! Attempting reconnect...")
            // Implement reconnection logic here
        } else {
            fmt.Println("✓ Database connection healthy")
        }
    }
}
```

**Create health check endpoint:**

**File: `handler/health/handler.go`**
```go
package health

import (
    "encoding/json"
    "net/http"
    
    "github.com/jmoiron/sqlx"
    "yourproject/infra/db"
)

type Handler struct {
    DB *sqlx.DB
}

func NewHandler(database *sqlx.DB) *Handler {
    return &Handler{DB: database}
}

type HealthResponse struct {
    Status   string `json:"status"`
    Database string `json:"database"`
}

func (h *Handler) HealthCheck(w http.ResponseWriter, r *http.Request) {
    response := HealthResponse{
        Status: "healthy",
    }
    
    // Check database
    err := db.HealthCheck(h.DB)
    if err != nil {
        response.Status = "unhealthy"
        response.Database = "disconnected"
        w.WriteHeader(http.StatusServiceUnavailable)
    } else {
        response.Database = "connected"
        w.WriteHeader(http.StatusOK)
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

**Register route:**

```go
// In routes.go
mux.HandleFunc("GET /health", healthHandler.HealthCheck)
```

**Test:**
```
GET http://localhost:4000/health

Response:
{
  "status": "healthy",
  "database": "connected"
}
```

**Benefits:**
- ✓ Monitor database connectivity
- ✓ Kubernetes/Docker health checks
- ✓ Load balancer integration
- ✓ Early detection of connection issues

</details>

---

### Question 3: Connection Pool Configuration

**Question:**
Configure the connection pool with optimal settings for a production application. Explain each setting and why it matters.

<details>
<summary>Click to see answer</summary>

**Answer:**

**File: `infra/db/connection.go`**

```go
package db

import (
    "fmt"
    "time"
    
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

// ConnectionConfig holds connection pool settings
type ConnectionConfig struct {
    MaxOpenConns    int           // Maximum open connections
    MaxIdleConns    int           // Maximum idle connections
    ConnMaxLifetime time.Duration // Maximum connection lifetime
    ConnMaxIdleTime time.Duration // Maximum idle time
}

// DefaultConnectionConfig returns recommended production settings
func DefaultConnectionConfig() *ConnectionConfig {
    return &ConnectionConfig{
        MaxOpenConns:    25,              // Max 25 concurrent connections
        MaxIdleConns:    5,               // Keep 5 connections idle
        ConnMaxLifetime: 5 * time.Minute, // Recycle after 5 minutes
        ConnMaxIdleTime: 30 * time.Second, // Close if idle for 30 seconds
    }
}

// NewConnection creates database connection with pool configuration
func NewConnection(poolConfig *ConnectionConfig) (*sqlx.DB, error) {
    dbSource := GetConnectionString()
    
    // Connect
    dbConnection, err := sqlx.Connect("postgres", dbSource)
    if err != nil {
        fmt.Println("Error connecting to database:", err)
        return nil, err
    }
    
    // Configure connection pool
    if poolConfig == nil {
        poolConfig = DefaultConnectionConfig()
    }
    
    dbConnection.SetMaxOpenConns(poolConfig.MaxOpenConns)
    dbConnection.SetMaxIdleConns(poolConfig.MaxIdleConns)
    dbConnection.SetConnMaxLifetime(poolConfig.ConnMaxLifetime)
    dbConnection.SetConnMaxIdleTime(poolConfig.ConnMaxIdleTime)
    
    // Verify connection
    err = HealthCheck(dbConnection)
    if err != nil {
        fmt.Println("Database health check failed:", err)
        dbConnection.Close()
        return nil, err
    }
    
    fmt.Printf("Database connection pool configured:\n")
    fmt.Printf("  Max Open Connections: %d\n", poolConfig.MaxOpenConns)
    fmt.Printf("  Max Idle Connections: %d\n", poolConfig.MaxIdleConns)
    fmt.Printf("  Connection Lifetime: %v\n", poolConfig.ConnMaxLifetime)
    fmt.Printf("  Idle Timeout: %v\n", poolConfig.ConnMaxIdleTime)
    
    return dbConnection, nil
}
```

**Understanding each setting:**

**1. MaxOpenConns (default: 25)**
```
What: Maximum number of open connections to database
Why: Prevent overwhelming database server
Example:
  - Too high (1000): Database can't handle it
  - Too low (2): Requests wait in queue
  - Optimal (25-100): Based on server capacity

Formula:
MaxOpenConns = (Number of CPUs × 2) + Number of Disks
For 4 CPU, 1 disk: (4 × 2) + 1 = 9 connections

Production: Start with 25, monitor and adjust
```

**2. MaxIdleConns (default: 5)**
```
What: Maximum idle connections kept open
Why: Balance between reuse and resource consumption
Example:
  - Too high (25): Waste database resources
  - Too low (0): Create new connection each time (slow)
  - Optimal (5-10): Ready for bursts, minimal waste

Rule of thumb: MaxIdleConns = MaxOpenConns / 5
```

**3. ConnMaxLifetime (default: 5 minutes)**
```
What: Maximum time a connection can be reused
Why: Prevent stale connections, handle database restarts
Example:
  - Too high (24 hours): Stale connections accumulate
  - Too low (10 seconds): Constant reconnection overhead
  - Optimal (5 minutes): Fresh connections, minimal overhead

Production: 5-15 minutes depending on database
```

**4. ConnMaxIdleTime (default: 30 seconds)**
```
What: Maximum time connection can sit idle
Why: Free resources during low traffic
Example:
  - Too high (10 minutes): Waste during quiet periods
  - Too low (5 seconds): Constant open/close
  - Optimal (30-60 seconds): Adapt to traffic patterns

Production: 30 seconds - 2 minutes
```

**Usage in application:**

```go
// cmd/serve.go
func Serve() {
    config.LoadConfig()
    cfg := config.GetConfig()
    
    // Create custom pool config
    poolConfig := &db.ConnectionConfig{
        MaxOpenConns:    50,                 // High traffic app
        MaxIdleConns:    10,                 // Keep 10 ready
        ConnMaxLifetime: 10 * time.Minute,   // Longer lifetime
        ConnMaxIdleTime: 1 * time.Minute,    // 1 min idle timeout
    }
    
    // Or use defaults
    // poolConfig := db.DefaultConnectionConfig()
    
    dbConnection, err := db.NewConnection(poolConfig)
    if err != nil {
        fmt.Println("Failed to connect to database:", err)
        os.Exit(1)
    }
    defer dbConnection.Close()
    
    // ... rest of code
}
```

**Monitoring pool stats:**

```go
// Get pool statistics
func GetPoolStats(db *sqlx.DB) {
    stats := db.Stats()
    
    fmt.Printf("Database Connection Pool Stats:\n")
    fmt.Printf("  Open Connections: %d\n", stats.OpenConnections)
    fmt.Printf("  In Use: %d\n", stats.InUse)
    fmt.Printf("  Idle: %d\n", stats.Idle)
    fmt.Printf("  Wait Count: %d\n", stats.WaitCount)
    fmt.Printf("  Wait Duration: %v\n", stats.WaitDuration)
    fmt.Printf("  Max Idle Closed: %d\n", stats.MaxIdleClosed)
    fmt.Printf("  Max Lifetime Closed: %d\n", stats.MaxLifetimeClosed)
}
```

**Production recommendations:**

| Scenario | MaxOpen | MaxIdle | Lifetime | IdleTime |
|----------|---------|---------|----------|----------|
| **Low traffic** | 10 | 2 | 5 min | 1 min |
| **Medium traffic** | 25 | 5 | 5 min | 30 sec |
| **High traffic** | 50-100 | 10-20 | 10 min | 1 min |
| **Microservice** | 10-20 | 3-5 | 5 min | 30 sec |

**Key takeaways:**
- ✓ Start with conservative settings
- ✓ Monitor actual usage
- ✓ Adjust based on metrics
- ✓ Consider database server limits
- ✓ Test under load

</details>

---

**Next Chapter Preview:**

In Chapter 53, we'll:
1. **Create database tables** - Users and Products schemas
2. **Write SQL queries** - INSERT, SELECT, UPDATE, DELETE
3. **Implement Repository Pattern** - Clean data layer
4. **Replace in-memory storage** - Use PostgreSQL for real data

Our database connection is ready—now we'll actually store data! 🚀
