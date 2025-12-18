# Chapter 54: PostgreSQL Data Types - Complete Guide

## Table of Contents
- [Introduction](#introduction)
- [Numeric Data Types](#numeric-data-types)
  - [Auto-Incrementing Types](#auto-incrementing-types)
  - [Integer Types](#integer-types)
  - [Decimal Types](#decimal-types)
- [String Data Types](#string-data-types)
  - [Fixed vs Variable Length](#fixed-vs-variable-length)
  - [Unlimited Text](#unlimited-text)
- [Boolean Data Type](#boolean-data-type)
- [Date and Time Types](#date-and-time-types)
  - [Date Only](#date-only)
  - [Time Only](#time-only)
  - [Timestamp Types](#timestamp-types)
- [Choosing the Right Data Type](#choosing-the-right-data-type)
- [Complete Table Example](#complete-table-example)
- [Storage Size Comparison](#storage-size-comparison)
- [Best Practices](#best-practices)
- [Summary](#summary)
- [Practice Questions](#practice-questions)

---

## Introduction

When creating database tables, every column needs a **data type**. Just like Go has `int`, `string`, `bool`, PostgreSQL has its own data types.

**In this chapter:**
- ✓ Understand all PostgreSQL data types
- ✓ Learn when to use each type
- ✓ Calculate storage sizes
- ✓ Choose optimal types for performance

**Why data types matter:**
```
Wrong type choice:
- Wasted storage space → Higher costs
- Slower queries → Poor performance
- Data overflow → Application crashes
```

Let's explore every data type systematically!

---

## Numeric Data Types

### Auto-Incrementing Types

#### SERIAL (32-bit)

**Definition:**
```sql
id SERIAL PRIMARY KEY
```

**How it works:**
```
User 1 → id = 1
User 2 → id = 2
User 3 → id = 3
User 4 → id = 4
... automatically increments
```

**Storage:** 4 bytes (32 bits)

**Range:** 1 to 2,147,483,647 (2.1 billion)

**Calculation:**
```
2^32 = 4,294,967,296
Divided by 2 (for positive only) = 2,147,483,648
Range: 1 to 2,147,483,647
```

**Real-world capacity:**
```
✓ Enough for 2.1 billion users
✓ More than entire population of many countries
✓ Suitable for most applications
```

**When to use:**
- User IDs
- Product IDs
- Order IDs
- Most auto-increment scenarios

**Example:**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,  -- Auto: 1, 2, 3, 4...
    name VARCHAR(100)
);
```

#### BIGSERIAL (64-bit)

**Definition:**
```sql
id BIGSERIAL PRIMARY KEY
```

**Storage:** 8 bytes (64 bits)

**Range:** 1 to 9,223,372,036,854,775,807

**That's:**
```
9.2 quintillion (18 zeros after 9)
9,223,372,036,854,775,807

Compare to SERIAL:
SERIAL:    2,147,483,647 (10 digits)
BIGSERIAL: 9,223,372,036,854,775,807 (19 digits)
```

**When to use:**
- High-traffic applications (millions of records per day)
- Transaction logs
- Event tracking systems
- When you never want to worry about running out of IDs

**Example:**
```sql
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,  -- Will never run out!
    amount DECIMAL(10,2)
);
```

**Comparison:**

| Type | Storage | Range | Use Case |
|------|---------|-------|----------|
| **SERIAL** | 4 bytes | 2.1 billion | Most applications |
| **BIGSERIAL** | 8 bytes | 9.2 quintillion | High-volume systems |

**Important:** You never manually set SERIAL/BIGSERIAL values—they auto-generate!

---

### Integer Types

#### SMALLINT (16-bit)

**Definition:**
```sql
age SMALLINT NOT NULL
```

**Storage:** 2 bytes (16 bits)

**Range:** -32,768 to 32,767

**When to use:**
- Age (0-120 max)
- Small quantities
- Status codes
- Ratings (1-5 stars)

**Example:**
```sql
CREATE TABLE profiles (
    id SERIAL PRIMARY KEY,
    age SMALLINT NOT NULL,  -- 2 bytes instead of 4
    rating SMALLINT DEFAULT 0  -- 1-5 stars
);
```

#### INTEGER / INT (32-bit)

**Definition:**
```sql
quantity INT NOT NULL
```

**Storage:** 4 bytes (32 bits)

**Range:** -2,147,483,648 to 2,147,483,647

**When to use:**
- Quantities
- Prices in cents
- Counts
- Most numeric values

**Example:**
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    stock_quantity INT NOT NULL,
    price_cents INT NOT NULL  -- Store $99.99 as 9999
);
```

#### BIGINT (64-bit)

**Definition:**
```sql
total_views BIGINT DEFAULT 0
```

**Storage:** 8 bytes (64 bits)

**Range:** -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807

**When to use:**
- Large counters (video views, page visits)
- Financial calculations (large amounts)
- Timestamps in milliseconds
- When INT is not enough

**Example:**
```sql
CREATE TABLE videos (
    id SERIAL PRIMARY KEY,
    view_count BIGINT DEFAULT 0,  -- Can handle billions of views
    like_count BIGINT DEFAULT 0
);
```

**Integer Type Comparison:**

| Type | Storage | Range | Example Use |
|------|---------|-------|-------------|
| **SMALLINT** | 2 bytes | ±32K | Age, ratings |
| **INTEGER** | 4 bytes | ±2.1B | Quantities, prices |
| **BIGINT** | 8 bytes | ±9.2 quintillion | Large counters |

**Memory matters:**
```
1 million users:
- SMALLINT for age: 2 MB
- INT for age:      4 MB
- BIGINT for age:   8 MB

Choose wisely to save costs!
```

---

### Decimal Types

#### REAL (32-bit)

**Definition:**
```sql
temperature REAL
```

**Storage:** 4 bytes (32 bits)

**Precision:** 6 decimal digits

**Example values:**
```
10.32
-99.5678
123456.0
```

**When to use:**
- Temperature readings
- Scientific measurements (where precision doesn't matter much)
- Non-critical calculations

**⚠️ Warning:**
```sql
-- NOT for money!
price REAL  -- ❌ Bad! Can lose precision
```

**Precision loss example:**
```
Stored: 10.32
Might become: 10.319999694824219
```

#### DOUBLE PRECISION (64-bit)

**Definition:**
```sql
account_balance DOUBLE PRECISION NOT NULL
```

**Storage:** 8 bytes (64 bits)

**Precision:** 15 decimal digits

**Example values:**
```
10.32154678
-9999.123456789012
1234567890.12345
```

**When to use:**
- ✓ Money calculations
- ✓ Financial data
- ✓ Precise measurements
- ✓ When precision matters

**Example:**
```sql
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    balance DOUBLE PRECISION NOT NULL,  -- Best for money
    interest_rate DOUBLE PRECISION DEFAULT 0.05
);
```

**Decimal Type Comparison:**

| Type | Storage | Precision | Use Case |
|------|---------|-----------|----------|
| **REAL** | 4 bytes | 6 digits | Temperature, approx values |
| **DOUBLE PRECISION** | 8 bytes | 15 digits | Money, exact calculations |

**Rule of thumb:**
```
For money:     DOUBLE PRECISION ✓
For anything else: Consider REAL (smaller)
```

**Why DOUBLE PRECISION for money:**
```sql
-- User has $100.50
-- Buys item for $25.25
-- Remaining: $75.25

WITH REAL (bad):
100.50 - 25.25 = 75.24999... ❌ (precision loss)

WITH DOUBLE PRECISION (good):
100.50 - 25.25 = 75.25 ✓ (accurate)
```

---

## String Data Types

### Fixed vs Variable Length

#### CHAR(n) - Fixed Length

**Definition:**
```sql
country_code CHAR(2)
```

**How it works:**
```
Declared: CHAR(6)
Store "Hi"
Result: "Hi    " (4 spaces added to fill 6 chars)
Memory: Always 6 bytes allocated
```

**Storage:** Exactly N bytes (always)

**When to use:**
- Country codes (US, BD, IN)
- Fixed-length codes
- Status codes (ACTIVE, CLOSED)

**❌ Problem:**
```sql
CREATE TABLE users (
    name CHAR(100)  -- Wastes space!
);

-- Store "John" (4 chars)
-- But allocates 100 bytes anyway!
-- Wasted: 96 bytes per user
-- 1 million users = 96 MB wasted
```

**I personally never use CHAR(n)** - it wastes memory!

#### VARCHAR(n) - Variable Length

**Definition:**
```sql
first_name VARCHAR(100)
```

**How it works:**
```
Declared: VARCHAR(100)
Store "Hi"
Result: "Hi" (only 2 bytes used)
Memory: 2 bytes + overhead

Store "Alexander"
Result: "Alexander" (9 bytes used)
Memory: 9 bytes + overhead
```

**Storage:** Actual length + 1-4 bytes overhead

**When to use:**
- ✓ Names (first_name, last_name)
- ✓ Email addresses
- ✓ Passwords (hashed)
- ✓ Titles
- ✓ Most string data

**Example:**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL  -- For bcrypt hash
);
```

**Why VARCHAR(255) for email?**
```
RFC 5321 email standard:
- Local part (before @): max 64 chars
- Domain part (after @): max 255 chars
- Total: max 320 chars (but 255 is practical limit)
```

---

### Unlimited Text

#### TEXT - Unlimited Length

**Definition:**
```sql
description TEXT
```

**Storage:** Actual length + overhead (no limit!)

**When to use:**
- Blog post content
- Product descriptions
- Comments
- Bio/About sections
- Any text where length is unknown

**Example:**
```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,  -- Titles are usually short
    content TEXT NOT NULL,        -- Content can be very long
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**String Type Comparison:**

| Type | Storage | Length | Use Case |
|------|---------|--------|----------|
| **CHAR(n)** | Fixed (n bytes) | Exactly n | ❌ Avoid (wastes space) |
| **VARCHAR(n)** | Variable (up to n) | Up to n | ✓ Names, emails, titles |
| **TEXT** | Variable (unlimited) | Unlimited | ✓ Long content |

**Visual example:**

```
CHAR(10):
Store "Hi" → "Hi        " (8 spaces wasted)
Memory: 10 bytes always

VARCHAR(10):
Store "Hi" → "Hi"
Memory: 2 bytes + overhead

TEXT:
Store "Hi" → "Hi"
Store "1000 word essay" → "1000 word essay"
Memory: Only what's needed
```

**When to use which:**

```sql
-- Known, limited length
first_name VARCHAR(100)    -- ✓ Good
email VARCHAR(255)         -- ✓ Good

-- Unknown length
blog_post TEXT             -- ✓ Good
user_bio TEXT              -- ✓ Good
product_description TEXT   -- ✓ Good

-- Fixed codes (rare cases)
country_code CHAR(2)       -- Acceptable for "US", "BD"
```

---

## Boolean Data Type

**Definition:**
```sql
is_shop_owner BOOLEAN DEFAULT FALSE
```

**Storage:** 1 byte

**Values:**
- `TRUE` or `true` or `'t'` or `'true'` or `'yes'` or `'1'`
- `FALSE` or `false` or `'f'` or `'false'` or `'no'` or `'0'`
- `NULL` (if nullable)

**When to use:**
- Flags (is_active, is_verified)
- Status (is_admin, is_premium)
- Permissions (can_edit, can_delete)
- Any yes/no field

**Example:**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    is_shop_owner BOOLEAN DEFAULT FALSE,
    is_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    can_post BOOLEAN DEFAULT TRUE
);
```

**Using DEFAULT:**

```sql
-- User doesn't specify is_shop_owner
INSERT INTO users (email) VALUES ('user@example.com');

-- Automatically becomes:
is_shop_owner = FALSE (default value)
```

**Query examples:**

```sql
-- Find all shop owners
SELECT * FROM users WHERE is_shop_owner = TRUE;

-- Find active users
SELECT * FROM users WHERE is_active = TRUE;

-- Find inactive shop owners
SELECT * FROM users 
WHERE is_shop_owner = TRUE 
AND is_active = FALSE;
```

**Best practices:**

```sql
-- ✓ Good: Positive naming with DEFAULT
is_active BOOLEAN DEFAULT TRUE

-- ❌ Confusing: Negative naming
is_not_active BOOLEAN DEFAULT FALSE  -- Avoid double negatives

-- ✓ Good: Clear naming
is_verified BOOLEAN DEFAULT FALSE
is_premium BOOLEAN DEFAULT FALSE
```

---

## Date and Time Types

### Date Only

#### DATE

**Definition:**
```sql
date_of_birth DATE
```

**Format:** `YYYY-MM-DD`

**Example values:**
```
2025-09-20
1990-05-15
2000-01-01
```

**Storage:** 4 bytes

**Range:** 4713 BC to 5874897 AD (basically unlimited)

**When to use:**
- Date of birth
- Registration date
- Event dates
- Any date without time

**Example:**
```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    date_of_birth DATE,
    hire_date DATE NOT NULL
);

-- Insert
INSERT INTO employees (name, date_of_birth, hire_date)
VALUES ('John Doe', '1990-05-15', '2024-01-01');
```

---

### Time Only

#### TIME

**Definition:**
```sql
office_opens TIME
```

**Format:** `HH:MI:SS` (24-hour format)

**Example values:**
```
09:32:00
14:50:43
23:59:59
```

**Storage:** 8 bytes

**When to use:**
- Business hours (opens/closes)
- Scheduled tasks (daily backup at 02:00:00)
- Time of day (without date)

**Example:**
```sql
CREATE TABLE business_hours (
    id SERIAL PRIMARY KEY,
    day_of_week VARCHAR(10),
    opens_at TIME NOT NULL,
    closes_at TIME NOT NULL
);

-- Insert
INSERT INTO business_hours (day_of_week, opens_at, closes_at)
VALUES ('Monday', '09:00:00', '18:00:00');
```

**⚠️ Note:** TIME has no timezone info!

---

### Timestamp Types

#### TIMESTAMP (without timezone)

**Definition:**
```sql
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
```

**Format:** `YYYY-MM-DD HH:MI:SS`

**Example values:**
```
2025-09-20 14:50:43
2024-01-01 09:30:00
2023-12-31 23:59:59
```

**Storage:** 8 bytes

**Range:** 4713 BC to 294276 AD

**⚠️ No timezone information!**

**When to use:**
- created_at / updated_at fields
- Event timestamps
- Most timestamp needs

**Example:**
```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**CURRENT_TIMESTAMP explained:**

```sql
-- When creating record
INSERT INTO posts (title) VALUES ('My Post');

-- created_at automatically becomes:
created_at = 2025-09-20 14:50:43  (current server time)
```

#### TIMESTAMP WITH TIME ZONE (TIMESTAMPTZ)

**Definition:**
```sql
created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
```

**Format:** `YYYY-MM-DD HH:MI:SS+TZ`

**Example values:**
```
2025-09-20 14:50:43+06  (Bangladesh time, UTC+6)
2025-09-20 08:50:43+00  (UTC)
2025-09-20 03:50:43-05  (US Eastern, UTC-5)
```

**Storage:** 8 bytes

**How it works:**
```
User in Bangladesh (UTC+6) creates post at 2:50 PM
Stored as: 2025-09-20 14:50:43+06

User in London (UTC+0) queries same post
Displays as: 2025-09-20 08:50:43+00

Same moment in time, different timezone display!
```

**When to use:**
- Global applications (users in different timezones)
- When timezone matters
- International businesses

**Example:**
```sql
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    event_time TIMESTAMPTZ NOT NULL,  -- Includes timezone
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Insert
INSERT INTO events (title, event_time)
VALUES ('Conference Call', '2025-09-20 14:00:00+06');
```

#### Shorthand: TIMESTAMPTZ

**Instead of writing:**
```sql
created_at TIMESTAMP WITH TIME ZONE
```

**Write:**
```sql
created_at TIMESTAMPTZ
```

**Both are identical!** I personally always use `TIMESTAMPTZ` (shorter and cleaner).

---

### Date/Time Type Comparison

| Type | Format | Timezone? | Storage | Use Case |
|------|--------|-----------|---------|----------|
| **DATE** | 2025-09-20 | ❌ | 4 bytes | Birth dates, hire dates |
| **TIME** | 14:50:43 | ❌ | 8 bytes | Business hours |
| **TIMESTAMP** | 2025-09-20 14:50:43 | ❌ | 8 bytes | created_at, updated_at |
| **TIMESTAMPTZ** | 2025-09-20 14:50:43+06 | ✓ | 8 bytes | Global apps |

**Which to use?**

```
Local application (one timezone):
✓ TIMESTAMP is fine

Global application (multiple timezones):
✓ TIMESTAMPTZ is better

Just date (no time):
✓ DATE

Just time (no date):
✓ TIME
```

**My recommendation:**
```sql
-- I personally use TIMESTAMPTZ for everything
created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP

-- Simple and handles all cases!
```

---

## Choosing the Right Data Type

### Decision Tree

**For IDs:**
```
How many records?
├─ < 2 billion → SERIAL
└─ Unlimited   → BIGSERIAL
```

**For numbers:**
```
What kind of number?
├─ Whole number (no decimals)
│  ├─ Small (-32K to 32K)      → SMALLINT
│  ├─ Medium (-2B to 2B)       → INTEGER
│  └─ Large (billions+)        → BIGINT
│
└─ Decimal number
   ├─ Low precision needed     → REAL
   └─ High precision (money)   → DOUBLE PRECISION
```

**For text:**
```
How long is the text?
├─ Known max length  → VARCHAR(n)
├─ Unknown length    → TEXT
└─ Fixed length      → CHAR(n) [rarely used]
```

**For dates/times:**
```
What do you need?
├─ Just date               → DATE
├─ Just time               → TIME
├─ Date + time             → TIMESTAMP
└─ Date + time + timezone  → TIMESTAMPTZ
```

### Real-World Examples

**Users table:**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,                    -- Auto-increment ID
    first_name VARCHAR(100) NOT NULL,         -- Name (known max)
    last_name VARCHAR(100) NOT NULL,          -- Name (known max)
    email VARCHAR(255) NOT NULL UNIQUE,       -- Email (standard max)
    password VARCHAR(255) NOT NULL,           -- Hashed password
    age SMALLINT,                             -- Age (0-120)
    account_balance DOUBLE PRECISION DEFAULT 0, -- Money (precision)
    is_verified BOOLEAN DEFAULT FALSE,        -- Flag
    date_of_birth DATE,                       -- Just date
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**Products table:**
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,                    -- Product ID
    name VARCHAR(200) NOT NULL,               -- Product name
    description TEXT,                         -- Long description
    price_cents INT NOT NULL,                 -- Price in cents ($99.99 = 9999)
    stock_quantity INT DEFAULT 0,             -- Stock count
    rating SMALLINT DEFAULT 0,                -- 1-5 stars
    view_count BIGINT DEFAULT 0,              -- Can be millions
    is_active BOOLEAN DEFAULT TRUE,           -- Active/inactive
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**Posts table:**
```sql
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,                 -- Lots of posts
    user_id INT NOT NULL,                     -- Foreign key
    title VARCHAR(200) NOT NULL,              -- Post title
    content TEXT NOT NULL,                    -- Long content
    view_count BIGINT DEFAULT 0,              -- High traffic
    like_count INT DEFAULT 0,                 -- Likes
    is_published BOOLEAN DEFAULT FALSE,       -- Draft/published
    published_at TIMESTAMPTZ,                 -- When published
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

---

## Complete Table Example

Let's create a comprehensive users table with all data types:

```sql
CREATE TABLE users (
    -- Auto-increment ID
    id SERIAL PRIMARY KEY,
    
    -- String fields
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    bio TEXT,  -- Can be long
    
    -- Numeric fields
    age SMALLINT,  -- 0-120
    account_balance DOUBLE PRECISION DEFAULT 0.00,
    total_purchases INT DEFAULT 0,
    total_spent DOUBLE PRECISION DEFAULT 0.00,
    
    -- Boolean fields
    is_shop_owner BOOLEAN DEFAULT FALSE,
    is_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Date/Time fields
    date_of_birth DATE,
    last_login TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**Why these choices?**

| Field | Type | Reason |
|-------|------|--------|
| id | SERIAL | Auto-increment, enough for 2B users |
| first_name | VARCHAR(100) | Names rarely > 100 chars |
| email | VARCHAR(255) | Email standard max length |
| bio | TEXT | Unknown length, can be long |
| age | SMALLINT | Max 120, saves space |
| account_balance | DOUBLE PRECISION | Money needs precision |
| total_purchases | INT | Count won't exceed 2B |
| is_shop_owner | BOOLEAN | Yes/no flag |
| date_of_birth | DATE | No time needed |
| created_at | TIMESTAMPTZ | Full timestamp with timezone |

---

## Storage Size Comparison

### Example: 1 Million Users

**Scenario:** Store age for 1 million users

**Option 1: SMALLINT (2 bytes)**
```
1,000,000 users × 2 bytes = 2 MB
```

**Option 2: INTEGER (4 bytes)**
```
1,000,000 users × 4 bytes = 4 MB
```

**Option 3: BIGINT (8 bytes)**
```
1,000,000 users × 8 bytes = 8 MB
```

**Savings:** Using SMALLINT instead of BIGINT saves 6 MB per million users!

### Real Cost Example

**10 million users, 100 columns, wrong choices:**

```
Using BIGINT everywhere instead of appropriate types:
Extra storage: 60 MB × 10 = 600 MB wasted

Cloud storage costs:
$0.10 per GB per month
600 MB ≈ 0.6 GB
Cost: $0.06/month

Seems small? But:
- 100 million users: $6/month = $72/year wasted
- Plus slower queries (more data to read)
- Plus higher backup costs
- Plus more bandwidth
```

**Choose wisely!**

---

## Best Practices

### 1. Choose Smallest Type That Fits

**❌ Bad:**
```sql
age BIGINT  -- Way too large!
```

**✓ Good:**
```sql
age SMALLINT  -- Perfect for 0-120
```

### 2. Use VARCHAR Over CHAR

**❌ Bad:**
```sql
name CHAR(100)  -- Wastes space
```

**✓ Good:**
```sql
name VARCHAR(100)  -- Variable length
```

### 3. DOUBLE PRECISION for Money

**❌ Bad:**
```sql
price REAL  -- Loses precision!
```

**✓ Good:**
```sql
price DOUBLE PRECISION
-- Or store as cents:
price_cents INT  -- $99.99 = 9999
```

### 4. Use TEXT for Unknown Lengths

**❌ Bad:**
```sql
description VARCHAR(1000)  -- What if longer?
```

**✓ Good:**
```sql
description TEXT  -- Unlimited
```

### 5. Always Use TIMESTAMPTZ

**❌ Okay:**
```sql
created_at TIMESTAMP
```

**✓ Better:**
```sql
created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
```

### 6. NOT NULL for Required Fields

**❌ Bad:**
```sql
email VARCHAR(255)  -- Can be NULL
```

**✓ Good:**
```sql
email VARCHAR(255) NOT NULL  -- Must provide
```

### 7. DEFAULT Values for Flags

**❌ Okay:**
```sql
is_active BOOLEAN
```

**✓ Better:**
```sql
is_active BOOLEAN DEFAULT TRUE
```

### 8. Plural Table Names

**❌ Bad:**
```sql
CREATE TABLE user ...
```

**✓ Good:**
```sql
CREATE TABLE users ...
```

---

## Summary

### Data Type Categories

**1. Auto-Incrementing**
- `SERIAL` (32-bit) - Most IDs
- `BIGSERIAL` (64-bit) - High-volume IDs

**2. Integers**
- `SMALLINT` (16-bit) - Small numbers
- `INTEGER` (32-bit) - Medium numbers
- `BIGINT` (64-bit) - Large numbers

**3. Decimals**
- `REAL` (32-bit) - Low precision
- `DOUBLE PRECISION` (64-bit) - High precision (money)

**4. Strings**
- `CHAR(n)` - Fixed length (avoid)
- `VARCHAR(n)` - Variable length (use this)
- `TEXT` - Unlimited length

**5. Boolean**
- `BOOLEAN` - TRUE/FALSE

**6. Date/Time**
- `DATE` - Just date
- `TIME` - Just time
- `TIMESTAMP` - Date + time
- `TIMESTAMPTZ` - Date + time + timezone (recommended)

### Quick Reference

| Data | Best Type |
|------|-----------|
| ID | SERIAL or BIGSERIAL |
| Age | SMALLINT |
| Money | DOUBLE PRECISION |
| Name | VARCHAR(100) |
| Email | VARCHAR(255) |
| Long text | TEXT |
| Flag | BOOLEAN |
| Date | DATE |
| Timestamp | TIMESTAMPTZ |

### Key Takeaways

1. **Size matters** - Wrong choices cost money
2. **VARCHAR > CHAR** - Variable length is better
3. **TEXT for unknown** - When length is unpredictable
4. **DOUBLE PRECISION for money** - Precision matters
5. **TIMESTAMPTZ always** - Handles all timezone cases
6. **NOT NULL when required** - Enforce data integrity
7. **DEFAULT for flags** - Sensible defaults

**Now you understand every data type!** 🎉

---

## Practice Questions

### Question 1: Optimize This Table

**Question:**
The following table wastes storage space. Identify the problems and rewrite it optimally:

```sql
CREATE TABLE products (
    id BIGINT,
    name CHAR(200),
    description VARCHAR(100),
    price REAL,
    stock BIGINT,
    rating BIGINT,
    is_featured CHAR(10),
    created_at TIMESTAMP
);
```

<details>
<summary>Click to see answer</summary>

**Answer:**

**Problems identified:**

1. **id BIGINT** - Overkill for most products, SERIAL is enough
2. **name CHAR(200)** - Wastes space, use VARCHAR
3. **description VARCHAR(100)** - Too limiting, use TEXT
4. **price REAL** - Loses precision for money, use DOUBLE PRECISION
5. **stock BIGINT** - Overkill, INT is enough
6. **rating BIGINT** - Way too big, SMALLINT (1-5) is enough
7. **is_featured CHAR(10)** - Should be BOOLEAN
8. **created_at TIMESTAMP** - Should use TIMESTAMPTZ

**Optimized version:**

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,                      -- Changed from BIGINT
    name VARCHAR(200) NOT NULL,                 -- Changed from CHAR(200)
    description TEXT,                           -- Changed from VARCHAR(100)
    price DOUBLE PRECISION NOT NULL,            -- Changed from REAL
    stock INT DEFAULT 0,                        -- Changed from BIGINT
    rating SMALLINT CHECK (rating >= 1 AND rating <= 5), -- Changed from BIGINT
    is_featured BOOLEAN DEFAULT FALSE,          -- Changed from CHAR(10)
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP  -- Changed from TIMESTAMP
);
```

**Storage savings per row:**

| Field | Before | After | Saved |
|-------|--------|-------|-------|
| id | 8 bytes | 4 bytes | 4 bytes |
| name | 200 bytes | ~20 bytes avg | ~180 bytes |
| description | ~50 bytes | ~100 bytes | -50 bytes (but more flexible) |
| price | 4 bytes | 8 bytes | -4 bytes (but more accurate) |
| stock | 8 bytes | 4 bytes | 4 bytes |
| rating | 8 bytes | 2 bytes | 6 bytes |
| is_featured | 10 bytes | 1 byte | 9 bytes |
| created_at | 8 bytes | 8 bytes | 0 bytes |

**Total saved per row:** ~149 bytes (considering average name length)

**For 1 million products:**
```
1,000,000 × 149 bytes = 149 MB saved!
```

**Plus:** More accurate prices, more flexible descriptions, proper boolean logic!

</details>

---

### Question 2: Design a Blog Schema

**Question:**
Design a complete database schema for a blog with the following requirements:
- Users can write posts
- Posts can be very long (thousands of words)
- Track view counts (could be millions)
- Track likes (reasonable count)
- Posts have tags (short text, comma-separated)
- Users have profiles with bio (variable length)
- Track all timestamps with timezone support

Use optimal data types for each field.

<details>
<summary>Click to see answer</summary>

**Answer:**

**Complete blog schema:**

```sql
-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,  -- bcrypt hash
    full_name VARCHAR(150),
    bio TEXT,  -- Can be long
    avatar_url VARCHAR(500),
    is_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    date_of_birth DATE,
    last_login TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Posts table
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,  -- Could have millions of posts
    user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(300) NOT NULL,
    slug VARCHAR(350) NOT NULL UNIQUE,  -- URL-friendly title
    content TEXT NOT NULL,  -- Can be thousands of words
    excerpt VARCHAR(500),  -- Short preview
    tags VARCHAR(200),  -- Comma-separated: "golang,database,tutorial"
    
    -- Counters
    view_count BIGINT DEFAULT 0,  -- Could be millions
    like_count INT DEFAULT 0,     -- Reasonable count
    comment_count INT DEFAULT 0,
    
    -- Status
    status VARCHAR(20) DEFAULT 'draft',  -- draft, published, archived
    is_featured BOOLEAN DEFAULT FALSE,
    
    -- SEO
    meta_title VARCHAR(70),
    meta_description VARCHAR(160),
    
    -- Timestamps
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    
    -- Indexes for performance
    CONSTRAINT valid_status CHECK (status IN ('draft', 'published', 'archived'))
);

-- Comments table
CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,  -- Many comments possible
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_id BIGINT REFERENCES comments(id) ON DELETE CASCADE,  -- For nested comments
    content TEXT NOT NULL,
    like_count INT DEFAULT 0,
    is_edited BOOLEAN DEFAULT FALSE,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Likes table (many-to-many)
CREATE TABLE post_likes (
    id BIGSERIAL PRIMARY KEY,
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(post_id, user_id)  -- User can like post only once
);

-- Create indexes for performance
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_published_at ON posts(published_at);
CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
CREATE INDEX idx_post_likes_post_id ON post_likes(post_id);
CREATE INDEX idx_post_likes_user_id ON post_likes(user_id);
```

**Data type explanations:**

**Users table:**
- `id SERIAL` - Max 2B users (enough)
- `username VARCHAR(50)` - Twitter-style short usernames
- `email VARCHAR(255)` - Standard email length
- `bio TEXT` - Can be long paragraph
- `date_of_birth DATE` - No time needed
- `last_login TIMESTAMPTZ` - With timezone

**Posts table:**
- `id BIGSERIAL` - Could have millions of posts
- `title VARCHAR(300)` - Long titles allowed
- `content TEXT` - Thousands of words
- `tags VARCHAR(200)` - Short comma-separated list
- `view_count BIGINT` - Could reach millions
- `like_count INT` - Max 2B (enough for likes)
- `published_at TIMESTAMPTZ` - When published (with timezone)

**Comments table:**
- `id BIGSERIAL` - Potentially millions of comments
- `content TEXT` - Comments can be long
- `parent_id BIGINT` - Self-referencing for nested comments

**Post_likes table:**
- `UNIQUE(post_id, user_id)` - Ensures user likes post only once
- `BIGSERIAL` - Many like records

**Storage estimation (1 million posts):**

```
Posts table:
- Basic fields: ~50 bytes
- title (avg 50 chars): 50 bytes
- content (avg 2000 chars): 2000 bytes
- Total per row: ~2100 bytes
- 1M posts: ~2 GB

Optimized with proper types!
```

</details>

---

### Question 3: Data Type Quiz

**Question:**
For each scenario, choose the optimal data type and explain why:

1. Store YouTube video view count
2. Store user's profile picture file size in bytes
3. Store product rating (1-5 stars with half stars: 1.5, 2.5, etc.)
4. Store IP address (e.g., "192.168.1.1")
5. Store country name
6. Store order amount in dollars and cents
7. Store user's gender
8. Store website session duration in seconds
9. Store medical prescription (very long text)
10. Store social security number (fixed 9 digits)

<details>
<summary>Click to see answer</summary>

**Answer:**

**1. YouTube video view count**
```sql
view_count BIGINT DEFAULT 0
```
**Why:** Popular videos can have billions of views (Gangnam Style: 4.6B views). BIGINT handles up to 9 quintillion.

---

**2. Profile picture file size (bytes)**
```sql
file_size_bytes INT NOT NULL
```
**Why:** Image files typically < 10 MB (10,485,760 bytes). INT handles up to 2.1B bytes (2 GB), which is way more than needed for profile pictures.

---

**3. Product rating (1-5 stars with half stars)**
```sql
rating REAL CHECK (rating >= 0 AND rating <= 5)
-- OR
rating NUMERIC(2,1) CHECK (rating >= 0 AND rating <= 5)
```
**Why:** Need decimal precision (1.5, 2.5, 4.5). REAL works fine for ratings. NUMERIC(2,1) is even better (2 digits total, 1 after decimal: 0.0 to 9.9).

**Example values:** 1.5, 2.5, 3.0, 4.5, 5.0

---

**4. IP address**
```sql
ip_address VARCHAR(45)
-- OR better
ip_address INET
```
**Why:** 
- IPv4: "192.168.1.1" (max 15 chars)
- IPv6: "2001:0db8:85a3:0000:0000:8a2e:0370:7334" (max 39 chars)
- VARCHAR(45) handles both
- **Better:** PostgreSQL has `INET` type specifically for IP addresses!

```sql
-- Using INET (best)
ip_address INET
-- Stores efficiently and validates format
```

---

**5. Country name**
```sql
country VARCHAR(100)
-- OR
country_code CHAR(2)  -- ISO 3166-1 alpha-2
```
**Why:** 
- Full name: "Democratic Republic of the Congo" (31 chars), VARCHAR(100) is safe
- **Better:** Store 2-letter code (BD, US, IN) using CHAR(2) - saves space and standardized

---

**6. Order amount (dollars and cents)**
```sql
-- Option 1: Store as decimal
amount NUMERIC(10,2) NOT NULL
-- Example: 1234.56

-- Option 2: Store as cents (recommended)
amount_cents INT NOT NULL
-- Example: $1234.56 = 123456 cents
```
**Why:** 
- NUMERIC(10,2) = 10 digits total, 2 after decimal (up to $99,999,999.99)
- Storing as cents avoids decimal precision issues
- INT for cents handles up to $21 million (enough for most orders)

**I recommend Option 2** (cents as INT) for simplicity and precision.

---

**7. User's gender**
```sql
-- Option 1: VARCHAR
gender VARCHAR(20)
-- Values: 'male', 'female', 'non-binary', 'prefer_not_to_say'

-- Option 2: ENUM (PostgreSQL)
CREATE TYPE gender_enum AS ENUM ('male', 'female', 'non-binary', 'prefer_not_to_say');
gender gender_enum;

-- Option 3: CHAR code
gender CHAR(1)
-- Values: 'M', 'F', 'N', 'X'
```
**Why:** Modern applications need inclusivity. VARCHAR(20) is flexible for future values. ENUM is type-safe but harder to change.

**Recommended:** VARCHAR(20) for flexibility

---

**8. Session duration (seconds)**
```sql
duration_seconds INT NOT NULL
```
**Why:** 
- Average session: 2-5 minutes (120-300 seconds)
- Long session: 2 hours (7,200 seconds)
- INT handles up to 2.1B seconds (68 years) - way more than needed

If tracking milliseconds:
```sql
duration_milliseconds BIGINT NOT NULL
```

---

**9. Medical prescription (very long text)**
```sql
prescription TEXT NOT NULL
```
**Why:** Medical prescriptions can include detailed instructions, dosages, warnings, contraindications - potentially thousands of words. TEXT has no length limit.

---

**10. Social Security Number (9 digits)**
```sql
-- Option 1: Store as string (recommended)
ssn CHAR(11)  -- Format: 123-45-6789
-- OR
ssn VARCHAR(11)  -- Format: 123-45-6789

-- Option 2: Store without dashes
ssn CHAR(9)  -- Format: 123456789
```
**Why:** 
- SSN is not a number (leading zeros matter: 001-23-4567)
- Should store as string
- CHAR(11) includes dashes for readability
- **Important:** Should be encrypted in real applications!

**Security consideration:**
```sql
-- Store encrypted
ssn_encrypted BYTEA NOT NULL  -- Encrypted binary data
```

**Never store SSN in plain text in production!**

---

**Summary table:**

| Scenario | Data Type | Reasoning |
|----------|-----------|-----------|
| YouTube views | BIGINT | Billions possible |
| File size | INT | MB range, < 2GB |
| Rating | NUMERIC(2,1) | Decimal precision |
| IP address | INET | Built-in type |
| Country | VARCHAR(100) | Variable length names |
| Money | NUMERIC(10,2) | Exact precision |
| Gender | VARCHAR(20) | Inclusive & flexible |
| Duration | INT | Seconds < 2B |
| Prescription | TEXT | Very long text |
| SSN | CHAR(11) | Fixed format string |

</details>

---

**Next Chapter Preview:**

In Chapter 55, we'll cover:
1. **Primary Keys** - Unique identifiers
2. **Foreign Keys** - Table relationships
3. **Constraints** - Data validation
4. **Indexes** - Query performance
5. **Unique constraints** - Duplicate prevention

We've mastered data types—now let's learn how to **relate tables and optimize performance**! 🚀
