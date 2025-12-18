# Chapter 55: SQL CRUD Operations - Complete Guide

## Table of Contents
- [Chapter 55: SQL CRUD Operations - Complete Guide](#chapter-55-sql-crud-operations---complete-guide)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [What is CRUD?](#what-is-crud)
    - [CRUD Breakdown](#crud-breakdown)
    - [SQL Command Mapping](#sql-command-mapping)
  - [Setting Up Query Tool](#setting-up-query-tool)
    - [Opening pgAdmin Query Tool](#opening-pgadmin-query-tool)
  - [READ Operations (SELECT)](#read-operations-select)
    - [Select All Columns](#select-all-columns)
    - [Select Specific Columns](#select-specific-columns)
    - [Understanding Rows and Columns](#understanding-rows-and-columns)
  - [CREATE Operations (INSERT)](#create-operations-insert)
    - [Basic Insert Syntax](#basic-insert-syntax)
    - [Insert with Specific Fields](#insert-with-specific-fields)
    - [Common Insert Mistakes](#common-insert-mistakes)
  - [UPDATE Operations](#update-operations)
    - [Update Single Field](#update-single-field)
    - [Update Multiple Fields](#update-multiple-fields)
    - [Update with WHERE Clause](#update-with-where-clause)
    - [Common UPDATE Mistakes](#common-update-mistakes)
  - [DELETE Operations](#delete-operations)
    - [Delete Specific Row](#delete-specific-row)
    - [Delete with Conditions](#delete-with-conditions)
    - [⚠️ DANGER: DELETE Without WHERE](#️-danger-delete-without-where)
  - [Saving SQL Commands](#saving-sql-commands)
    - [Why Save SQL Files?](#why-save-sql-files)
    - [File Organization](#file-organization)
    - [Save Each Operation](#save-each-operation)
  - [Common Mistakes and Fixes](#common-mistakes-and-fixes)
    - [Mistake Summary Table](#mistake-summary-table)
    - [Debugging Tips](#debugging-tips)
  - [Best Practices](#best-practices)
    - [1. Always Use WHERE in UPDATE/DELETE](#1-always-use-where-in-updatedelete)
    - [2. Use Transactions for Safety](#2-use-transactions-for-safety)
    - [3. Soft Delete Instead of Hard Delete](#3-soft-delete-instead-of-hard-delete)
    - [4. Use Prepared Statements (Never String Concatenation)](#4-use-prepared-statements-never-string-concatenation)
    - [5. Select Only Needed Columns](#5-select-only-needed-columns)
    - [6. Use LIMIT for Large Tables](#6-use-limit-for-large-tables)
    - [7. Format SQL for Readability](#7-format-sql-for-readability)
  - [Summary](#summary)
    - [CRUD Operations Quick Reference](#crud-operations-quick-reference)
    - [Key Takeaways](#key-takeaways)
    - [SQL Syntax Cheatsheet](#sql-syntax-cheatsheet)
  - [Practice Questions](#practice-questions)
    - [Question 1: Complete CRUD Implementation](#question-1-complete-crud-implementation)
    - [Question 2: Fix SQL Mistakes](#question-2-fix-sql-mistakes)
    - [Question 3: Complex Query Challenge](#question-3-complex-query-challenge)

---

## Introduction

CRUD is the foundation of all database applications. Every app you use—Facebook, Instagram, YouTube—performs these four operations:

**CRUD = Create, Read, Update, Delete**

```
C - CREATE  → Add new data
R - READ    → Retrieve data
U - UPDATE  → Modify data
D - DELETE  → Remove data
```

**In this chapter:**
- ✓ Master SQL syntax for all CRUD operations
- ✓ Learn common mistakes and how to avoid them
- ✓ Practice with real examples
- ✓ Save SQL commands for reference

**Real-world examples:**
```
Facebook post:
- CREATE: Write new post
- READ: View posts in feed
- UPDATE: Edit your post
- DELETE: Remove post

User registration:
- CREATE: Sign up new user
- READ: Login (check credentials)
- UPDATE: Change profile info
- DELETE: Delete account
```

Let's master these operations!

---

## What is CRUD?

### CRUD Breakdown

**C - CREATE (INSERT)**
```sql
INSERT INTO users (first_name, last_name, email)
VALUES ('John', 'Doe', 'john@example.com');
```
**Purpose:** Add new records to database

---

**R - READ (SELECT)**
```sql
SELECT * FROM users;
SELECT first_name, email FROM users WHERE id = 1;
```
**Purpose:** Retrieve data from database

---

**U - UPDATE**
```sql
UPDATE users 
SET first_name = 'Jane', last_name = 'Smith'
WHERE id = 1;
```
**Purpose:** Modify existing records

---

**D - DELETE**
```sql
DELETE FROM users WHERE id = 1;
```
**Purpose:** Remove records from database

---

### SQL Command Mapping

| CRUD | SQL Command | HTTP Method | Purpose |
|------|-------------|-------------|---------|
| **CREATE** | INSERT | POST | Add new data |
| **READ** | SELECT | GET | Retrieve data |
| **UPDATE** | UPDATE | PUT/PATCH | Modify data |
| **DELETE** | DELETE | DELETE | Remove data |

**Remember:** Every application uses these four operations!

---

## Setting Up Query Tool

### Opening pgAdmin Query Tool

**Step 1: Navigate to your database**
```
pgAdmin 4
└─ Servers
    └─ PostgreSQL 15
        └─ Databases
            └─ ecommerce
                └─ Right-click → Query Tool
```

**Step 2: Query Tool opens**
```
+--------------------------------------------------+
|  File  Edit  View  Query  Tools  Help           |
+--------------------------------------------------+
|                                                  |
|  [Write SQL queries here]                       |
|                                                  |
|                                                  |
+--------------------------------------------------+
|  [Results appear here]                          |
+--------------------------------------------------+
```

**Step 3: Execute queries**
- Write SQL command
- Click ▶️ Execute/Refresh button (F5)
- View results below

---

## READ Operations (SELECT)

### Select All Columns

**Syntax:**
```sql
SELECT * FROM table_name;
```

**Example:**
```sql
SELECT * FROM users;
```

**What this does:**
- `SELECT *` = Select all columns
- `FROM users` = From the users table
- Returns every row and every column

**Result:**
```
 id | first_name | last_name |        email         | password | is_shop_owner |      created_at       
----+------------+-----------+----------------------+----------+---------------+----------------------
  1 | Habib      | Rahman    | habib@example.com    | pass123  | t             | 2024-01-15 10:30:00
  2 | Khalil     | Ahmed     | khalil@example.com   | pass456  | t             | 2024-01-16 11:20:00
  3 | Rahim      | Khan      | rahim@example.com    | pass789  | f             | 2024-01-17 09:15:00
  4 | Karim      | Ali       | karim@example.com    | pass012  | f             | 2024-01-18 14:45:00
```

**Visual representation:**

```
Table: users
+----+------------+-----------+-------------------+
| id | first_name | last_name | email             |  ← COLUMNS
+----+------------+-----------+-------------------+
|  1 | Habib      | Rahman    | habib@example.com |  ← ROW 1
+----+------------+-----------+-------------------+
|  2 | Khalil     | Ahmed     | khalil@example.com|  ← ROW 2
+----+------------+-----------+-------------------+
|  3 | Rahim      | Khan      | rahim@example.com |  ← ROW 3
+----+------------+-----------+-------------------+
```

---

### Select Specific Columns

**Syntax:**
```sql
SELECT column1, column2 FROM table_name;
```

**Example:**
```sql
SELECT first_name, last_name FROM users;
```

**Result (only 2 columns):**
```
 first_name | last_name 
------------+-----------
 Habib      | Rahman
 Khalil     | Ahmed
 Rahim      | Khan
 Karim      | Ali
```

**Multiple columns:**
```sql
SELECT first_name, last_name, email FROM users;
```

**Result:**
```
 first_name | last_name |        email         
------------+-----------+----------------------
 Habib      | Rahman    | habib@example.com
 Khalil     | Ahmed     | khalil@example.com
 Rahim      | Khan      | rahim@example.com
 Karim      | Ali       | karim@example.com
```

**Why select specific columns?**
```
SELECT * FROM users;
- Returns: id, first_name, last_name, email, password, is_shop_owner, created_at, updated_at
- Size: ~500 bytes per row
- 1 million rows: 500 MB

SELECT first_name, email FROM users;
- Returns: first_name, email only
- Size: ~100 bytes per row
- 1 million rows: 100 MB
- Savings: 400 MB! Faster queries!
```

**Best practice:** Only select columns you need!

---

### Understanding Rows and Columns

**Column** = Vertical data (field/property)
```
first_name  ← This is a COLUMN
    ↓
  Habib
  Khalil
  Rahim
  Karim
```

**Row** = Horizontal data (record/entry)
```
| 1 | Habib | Rahman | habib@example.com | ← This is a ROW
```

**Terminology:**

| Term | Meaning | Example |
|------|---------|---------|
| **Row** | Single record | One user's complete data |
| **Column** | Single field | first_name column |
| **Cell** | Single value | "Habib" |
| **Table** | Collection of rows | All users |

---

## CREATE Operations (INSERT)

### Basic Insert Syntax

**Syntax:**
```sql
INSERT INTO table_name (column1, column2, column3)
VALUES (value1, value2, value3);
```

**Example:**
```sql
INSERT INTO users (first_name, last_name, email, password)
VALUES ('Israfil', 'Islam', 'israfil@gmail.com', '123456');
```

**What this does:**
1. Insert into `users` table
2. Specify columns to fill
3. Provide values for those columns

**Result:**
```
Query returned successfully: 1 row affected, 42 msec.
```

**Verify insertion:**
```sql
SELECT * FROM users;
```

**New row appears:**
```
 id | first_name | last_name |        email          | password | is_shop_owner |      created_at       
----+------------+-----------+-----------------------+----------+---------------+----------------------
  5 | Israfil    | Islam     | israfil@gmail.com     | 123456   | f             | 2024-01-20 15:30:00
```

---

### Insert with Specific Fields

**You don't need to insert all fields!**

**Example (minimal fields):**
```sql
INSERT INTO users (first_name, last_name, email, password)
VALUES ('Israfil', 'Islam', 'israfil@gmail.com', '123456');
```

**What happens to other fields?**
```
id              → Auto-generated (SERIAL)
is_shop_owner   → Gets DEFAULT value (FALSE)
created_at      → Gets DEFAULT value (CURRENT_TIMESTAMP)
updated_at      → Gets DEFAULT value (CURRENT_TIMESTAMP)
```

**If field is NOT NULL and no default:**
```sql
-- This will FAIL
INSERT INTO users (first_name)
VALUES ('John');

-- Error: null value in column "email" violates not-null constraint
```

**You MUST provide:**
- All NOT NULL fields without defaults
- Example: first_name, last_name, email, password

---

### Common Insert Mistakes

**❌ Mistake 1: Using double quotes**
```sql
INSERT INTO users (first_name, last_name, email)
VALUES ("Israfil", "Islam", "israfil@gmail.com");  -- ❌ Wrong!
```

**✓ Fix: Use single quotes**
```sql
INSERT INTO users (first_name, last_name, email)
VALUES ('Israfil', 'Islam', 'israfil@gmail.com');  -- ✓ Correct!
```

**Why?**
- PostgreSQL uses single quotes `'...'` for strings
- Double quotes `"..."` are for identifiers (table/column names)

---

**❌ Mistake 2: Missing comma**
```sql
INSERT INTO users (first_name, last_name, email, password)
VALUES ('Israfil', 'Islam', 'israfil@gmail.com', '123456');  -- ✓ Correct

-- If you add comma after last value:
VALUES ('Israfil', 'Islam', 'israfil@gmail.com', '123456',);  -- ❌ Syntax error
```

**✓ Fix: No comma after last value**

---

**❌ Mistake 3: Column count mismatch**
```sql
-- 4 columns specified
INSERT INTO users (first_name, last_name, email, password)
-- But only 3 values provided
VALUES ('Israfil', 'Islam', 'israfil@gmail.com');  -- ❌ Error!
```

**✓ Fix: Match column count with value count**
```sql
INSERT INTO users (first_name, last_name, email, password)
VALUES ('Israfil', 'Islam', 'israfil@gmail.com', '123456');  -- ✓ 4 and 4
```

---

**❌ Mistake 4: Wrong order**
```sql
-- Columns declared in this order:
INSERT INTO users (first_name, last_name, email)
-- But values in different order:
VALUES ('israfil@gmail.com', 'Israfil', 'Islam');  -- ❌ Wrong order!

-- Result:
-- first_name = 'israfil@gmail.com'  (email in name field!)
-- last_name  = 'Israfil'
-- email      = 'Islam'
```

**✓ Fix: Match order exactly**
```sql
INSERT INTO users (first_name, last_name, email)
VALUES ('Israfil', 'Islam', 'israfil@gmail.com');  -- ✓ Correct order
```

---

## UPDATE Operations

### Update Single Field

**Syntax:**
```sql
UPDATE table_name
SET column_name = new_value
WHERE condition;
```

**Example:**
```sql
UPDATE users
SET last_name = 'Rahman'
WHERE id = 5;
```

**What this does:**
- Update `users` table
- Set `last_name` to 'Rahman'
- Only for row where `id = 5`

**Before:**
```
 id | first_name | last_name |        email          
----+------------+-----------+-----------------------
  5 | Israfil    | Islam     | israfil@gmail.com
```

**After:**
```
 id | first_name | last_name |        email          
----+------------+-----------+-----------------------
  5 | Israfil    | Rahman    | israfil@gmail.com
```

---

### Update Multiple Fields

**Syntax:**
```sql
UPDATE table_name
SET column1 = value1,
    column2 = value2,
    column3 = value3
WHERE condition;
```

**Example:**
```sql
UPDATE users
SET first_name = 'Bondhon',
    last_name = 'Rahman'
WHERE id = 5;
```

**What this does:**
- Update both first_name AND last_name
- Only for user with id = 5

**Before:**
```
 id | first_name | last_name |        email          
----+------------+-----------+-----------------------
  5 | Israfil    | Islam     | israfil@gmail.com
```

**After:**
```
 id | first_name | last_name |        email          
----+------------+-----------+-----------------------
  5 | Bondhon    | Rahman    | israfil@gmail.com
```

**Important:** Separate multiple columns with commas!

---

### Update with WHERE Clause

**⚠️ DANGER: Without WHERE**
```sql
UPDATE users
SET first_name = 'Test';
-- No WHERE clause!
```

**Result:**
```
ALL users get first_name = 'Test'

Before:
 id | first_name | last_name
----+------------+-----------
  1 | Habib      | Rahman
  2 | Khalil     | Ahmed
  3 | Rahim      | Khan

After:
 id | first_name | last_name
----+------------+-----------
  1 | Test       | Rahman    ← Changed!
  2 | Test       | Ahmed     ← Changed!
  3 | Test       | Khan      ← Changed!

DISASTER! All names changed!
```

**✓ Always use WHERE:**
```sql
UPDATE users
SET first_name = 'Test'
WHERE id = 1;  -- Only update id=1
```

**WHERE clause examples:**

```sql
-- Update by ID
UPDATE users SET email = 'new@email.com' WHERE id = 5;

-- Update by email
UPDATE users SET is_verified = TRUE WHERE email = 'user@example.com';

-- Update multiple rows with condition
UPDATE users SET is_active = FALSE WHERE last_login < '2023-01-01';

-- Update with multiple conditions
UPDATE users 
SET is_premium = TRUE 
WHERE is_shop_owner = TRUE AND total_purchases > 100;
```

---

### Common UPDATE Mistakes

**❌ Mistake 1: Forgetting SET keyword**
```sql
UPDATE users
first_name = 'Bondhon'  -- ❌ Missing SET
WHERE id = 5;
```

**✓ Fix:**
```sql
UPDATE users
SET first_name = 'Bondhon'  -- ✓ SET keyword
WHERE id = 5;
```

---

**❌ Mistake 2: Using comma before WHERE**
```sql
UPDATE users
SET first_name = 'Bondhon',
    last_name = 'Rahman',  -- ❌ Extra comma before WHERE
WHERE id = 5;
```

**✓ Fix:**
```sql
UPDATE users
SET first_name = 'Bondhon',
    last_name = 'Rahman'   -- ✓ No comma before WHERE
WHERE id = 5;
```

---

**❌ Mistake 3: Single quotes vs double quotes**
```sql
UPDATE users
SET first_name = "Bondhon"  -- ❌ Double quotes
WHERE id = 5;
```

**✓ Fix:**
```sql
UPDATE users
SET first_name = 'Bondhon'  -- ✓ Single quotes
WHERE id = 5;
```

---

## DELETE Operations

### Delete Specific Row

**Syntax:**
```sql
DELETE FROM table_name
WHERE condition;
```

**Example:**
```sql
DELETE FROM users
WHERE id = 5;
```

**What this does:**
- Delete from `users` table
- Only row where `id = 5`

**Before:**
```
 id | first_name | last_name |        email          
----+------------+-----------+-----------------------
  1 | Habib      | Rahman    | habib@example.com
  2 | Khalil     | Ahmed     | khalil@example.com
  5 | Israfil    | Islam     | israfil@gmail.com
```

**After:**
```
 id | first_name | last_name |        email          
----+------------+-----------+-----------------------
  1 | Habib      | Rahman    | habib@example.com
  2 | Khalil     | Ahmed     | khalil@example.com
(Row with id=5 deleted)
```

**Result message:**
```
Query returned successfully: 1 row deleted, 35 msec.
```

---

### Delete with Conditions

**Delete by email:**
```sql
DELETE FROM users
WHERE email = 'spam@example.com';
```

**Delete multiple rows:**
```sql
-- Delete all inactive users
DELETE FROM users
WHERE is_active = FALSE;

-- Delete old accounts
DELETE FROM users
WHERE created_at < '2020-01-01';

-- Delete unverified users older than 7 days
DELETE FROM users
WHERE is_verified = FALSE 
AND created_at < NOW() - INTERVAL '7 days';
```

---

### ⚠️ DANGER: DELETE Without WHERE

**❌ EXTREMELY DANGEROUS:**
```sql
DELETE FROM users;
-- No WHERE clause!
```

**Result:**
```
ALL USERS DELETED!

Before:
100,000 users in table

After:
0 users in table

COMPLETE DATA LOSS!
```

**How to prevent:**
1. **Always test with SELECT first:**
```sql
-- First, see what will be deleted
SELECT * FROM users WHERE id = 5;

-- If correct, then delete
DELETE FROM users WHERE id = 5;
```

2. **Use transactions:**
```sql
BEGIN;
DELETE FROM users WHERE id = 5;
SELECT * FROM users;  -- Verify
-- If good:
COMMIT;
-- If bad:
ROLLBACK;
```

3. **Soft delete instead:**
```sql
-- Don't actually delete, just mark as deleted
UPDATE users
SET is_deleted = TRUE, deleted_at = NOW()
WHERE id = 5;

-- Query active users:
SELECT * FROM users WHERE is_deleted = FALSE;
```

---

## Saving SQL Commands

### Why Save SQL Files?

**Benefits:**
- ✓ Reference later
- ✓ Version control
- ✓ Share with team
- ✓ Documentation
- ✓ Easy to reuse

### File Organization

**Create SQL folder:**
```bash
mkdir -p infra/db/queries
```

**File structure:**
```
infra/db/queries/
├── 01_create_users_table.sql
├── 02_insert_users.sql
├── 03_select_users.sql
├── 04_update_users.sql
└── 05_delete_users.sql
```

---

### Save Each Operation

**File: `02_insert_users.sql`**
```sql
-- Insert new user into users table
INSERT INTO users (first_name, last_name, email, password)
VALUES ('Israfil', 'Islam', 'israfil@gmail.com', '123456');

-- Insert multiple users at once
INSERT INTO users (first_name, last_name, email, password)
VALUES 
    ('User1', 'Test1', 'user1@test.com', 'pass1'),
    ('User2', 'Test2', 'user2@test.com', 'pass2'),
    ('User3', 'Test3', 'user3@test.com', 'pass3');
```

---

**File: `03_select_users.sql`**
```sql
-- Select all users
SELECT * FROM users;

-- Select specific columns
SELECT first_name, last_name, email FROM users;

-- Select with conditions
SELECT * FROM users WHERE is_shop_owner = TRUE;

-- Select with sorting
SELECT * FROM users ORDER BY created_at DESC;

-- Select with limit
SELECT * FROM users LIMIT 10;
```

---

**File: `04_update_users.sql`**
```sql
-- Update single user
UPDATE users
SET first_name = 'Bondhon',
    last_name = 'Rahman'
WHERE id = 5;

-- Update multiple users
UPDATE users
SET is_verified = TRUE
WHERE email LIKE '%@company.com';

-- Update with timestamp
UPDATE users
SET last_login = NOW()
WHERE id = 1;
```

---

**File: `05_delete_users.sql`**
```sql
-- Delete single user
DELETE FROM users WHERE id = 5;

-- Delete with condition
DELETE FROM users WHERE is_active = FALSE;

-- Soft delete (preferred)
UPDATE users
SET is_deleted = TRUE, deleted_at = NOW()
WHERE id = 5;
```

---

## Common Mistakes and Fixes

### Mistake Summary Table

| Mistake | Wrong | Correct |
|---------|-------|---------|
| **Quotes** | `"value"` | `'value'` |
| **Comma** | `VALUES (a, b,);` | `VALUES (a, b);` |
| **SET keyword** | `UPDATE table column = val` | `UPDATE table SET column = val` |
| **WHERE missing** | `UPDATE table SET x = y` | `UPDATE table SET x = y WHERE id = 1` |
| **Column count** | 4 columns, 3 values | 4 columns, 4 values |
| **Semicolon** | Often forgotten | Always end with `;` |

---

### Debugging Tips

**1. Check syntax carefully**
```sql
-- Common typos:
SELCET * FROM users;  -- ❌ Typo: SELCET
SELECT * FORM users;  -- ❌ Typo: FORM
SELECT * FROM user;   -- ❌ Wrong table name (missing 's')
```

**2. Use pgAdmin error messages**
```
ERROR: syntax error at or near "FORM"
LINE 1: SELECT * FORM users;
                 ^
```
The `^` points to the error!

**3. Test with SELECT first**
```sql
-- Before UPDATE:
SELECT * FROM users WHERE id = 5;  -- See what will change

-- Before DELETE:
SELECT * FROM users WHERE is_active = FALSE;  -- See what will be deleted
```

**4. Count affected rows**
```sql
-- After operation, check:
SELECT COUNT(*) FROM users;

-- Or see affected rows in result:
UPDATE users SET ... ;
-- Result: "1 row affected" (good)
-- Result: "1000 rows affected" (might be bad!)
```

---

## Best Practices

### 1. Always Use WHERE in UPDATE/DELETE

**❌ Dangerous:**
```sql
UPDATE users SET password = 'new_pass';  -- Updates ALL users!
DELETE FROM users;  -- Deletes ALL users!
```

**✓ Safe:**
```sql
UPDATE users SET password = 'new_pass' WHERE id = 1;
DELETE FROM users WHERE id = 1;
```

---

### 2. Use Transactions for Safety

**Wrap changes in transaction:**
```sql
BEGIN;
UPDATE users SET email = 'new@email.com' WHERE id = 1;
-- Check result
SELECT * FROM users WHERE id = 1;
-- If good:
COMMIT;
-- If bad:
ROLLBACK;
```

---

### 3. Soft Delete Instead of Hard Delete

**Hard delete (gone forever):**
```sql
DELETE FROM users WHERE id = 1;  -- Can't undo!
```

**Soft delete (can recover):**
```sql
-- Add columns:
ALTER TABLE users ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- "Delete" user:
UPDATE users
SET is_deleted = TRUE, deleted_at = NOW()
WHERE id = 1;

-- Query active users:
SELECT * FROM users WHERE is_deleted = FALSE;

-- Restore user:
UPDATE users
SET is_deleted = FALSE, deleted_at = NULL
WHERE id = 1;
```

---

### 4. Use Prepared Statements (Never String Concatenation)

**❌ SQL Injection vulnerability:**
```go
// NEVER DO THIS!
email := r.FormValue("email")
query := "SELECT * FROM users WHERE email = '" + email + "'"
db.Query(query)

// If user enters: ' OR '1'='1
// Query becomes: SELECT * FROM users WHERE email = '' OR '1'='1'
// Returns ALL users!
```

**✓ Safe with prepared statements:**
```go
// Always use placeholders!
email := r.FormValue("email")
query := "SELECT * FROM users WHERE email = $1"
db.Query(query, email)  // Safe!
```

---

### 5. Select Only Needed Columns

**❌ Wasteful:**
```sql
SELECT * FROM users;  -- Returns all columns
```

**✓ Efficient:**
```sql
SELECT id, first_name, email FROM users;  -- Only what you need
```

---

### 6. Use LIMIT for Large Tables

**❌ Slow:**
```sql
SELECT * FROM users;  -- Returns 1 million rows!
```

**✓ Fast:**
```sql
SELECT * FROM users LIMIT 100;  -- Returns 100 rows
SELECT * FROM users LIMIT 100 OFFSET 100;  -- Next 100 rows (pagination)
```

---

### 7. Format SQL for Readability

**❌ Hard to read:**
```sql
INSERT INTO users(first_name,last_name,email,password)VALUES('John','Doe','john@example.com','pass123');
```

**✓ Easy to read:**
```sql
INSERT INTO users (
    first_name,
    last_name,
    email,
    password
)
VALUES (
    'John',
    'Doe',
    'john@example.com',
    'pass123'
);
```

---

## Summary

### CRUD Operations Quick Reference

**CREATE (INSERT):**
```sql
INSERT INTO users (first_name, last_name, email)
VALUES ('John', 'Doe', 'john@example.com');
```

**READ (SELECT):**
```sql
-- All data
SELECT * FROM users;

-- Specific columns
SELECT first_name, email FROM users;

-- With condition
SELECT * FROM users WHERE id = 1;
```

**UPDATE:**
```sql
-- Single field
UPDATE users SET first_name = 'Jane' WHERE id = 1;

-- Multiple fields
UPDATE users 
SET first_name = 'Jane', last_name = 'Smith'
WHERE id = 1;
```

**DELETE:**
```sql
-- Hard delete
DELETE FROM users WHERE id = 1;

-- Soft delete (preferred)
UPDATE users SET is_deleted = TRUE WHERE id = 1;
```

---

### Key Takeaways

1. **CRUD = Create, Read, Update, Delete** - Foundation of all apps
2. **Single quotes for strings** - `'value'` not `"value"`
3. **Always use WHERE** - Except when you really mean to affect all rows
4. **Test with SELECT first** - Before UPDATE or DELETE
5. **Use transactions** - For safety
6. **Soft delete preferred** - Can recover data
7. **Prepared statements** - Prevent SQL injection
8. **Save SQL files** - For reference and documentation

---

### SQL Syntax Cheatsheet

```sql
-- CREATE
INSERT INTO table (col1, col2) VALUES (val1, val2);

-- READ
SELECT col1, col2 FROM table WHERE condition;

-- UPDATE
UPDATE table SET col1 = val1, col2 = val2 WHERE condition;

-- DELETE
DELETE FROM table WHERE condition;

-- Common WHERE clauses
WHERE id = 1
WHERE email = 'user@example.com'
WHERE is_active = TRUE
WHERE created_at > '2024-01-01'
WHERE age BETWEEN 18 AND 65
WHERE email LIKE '%@gmail.com'
WHERE id IN (1, 2, 3, 4)
```

**Everyone makes mistakes—even experienced developers!** Use ChatGPT, documentation, and saved SQL files as reference. Practice makes perfect! 🎯

---

## Practice Questions

### Question 1: Complete CRUD Implementation

**Question:**
Write SQL commands to:
1. Create a new table `products` with columns: id (serial primary key), name (varchar 200), description (text), price (numeric 10,2), stock (integer), is_active (boolean default true)
2. Insert 3 products
3. Select all active products
4. Update the price of product with id=2 to $49.99
5. Soft delete product with id=3

<details>
<summary>Click to see answer</summary>

**Answer:**

**Step 1: CREATE TABLE**
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price NUMERIC(10,2) NOT NULL,
    stock INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**Step 2: INSERT 3 products**
```sql
INSERT INTO products (name, description, price, stock)
VALUES 
    ('Laptop', 'High-performance laptop for developers', 999.99, 50),
    ('Mouse', 'Wireless ergonomic mouse', 29.99, 200),
    ('Keyboard', 'Mechanical RGB keyboard', 149.99, 100);
```

**Verify insertion:**
```sql
SELECT * FROM products;
```

**Result:**
```
 id |   name   |            description             |  price  | stock | is_active
----+----------+------------------------------------+---------+-------+-----------
  1 | Laptop   | High-performance laptop...         | 999.99  | 50    | t
  2 | Mouse    | Wireless ergonomic mouse           | 29.99   | 200   | t
  3 | Keyboard | Mechanical RGB keyboard            | 149.99  | 100   | t
```

**Step 3: SELECT all active products**
```sql
SELECT * FROM products WHERE is_active = TRUE;
```

**Or more specifically:**
```sql
SELECT id, name, price, stock 
FROM products 
WHERE is_active = TRUE 
AND is_deleted = FALSE
ORDER BY name;
```

**Step 4: UPDATE price of product id=2**
```sql
UPDATE products
SET price = 49.99,
    updated_at = CURRENT_TIMESTAMP
WHERE id = 2;
```

**Verify update:**
```sql
SELECT id, name, price FROM products WHERE id = 2;
```

**Result:**
```
 id | name  | price 
----+-------+-------
  2 | Mouse | 49.99
```

**Step 5: SOFT DELETE product id=3**
```sql
UPDATE products
SET is_deleted = TRUE,
    deleted_at = CURRENT_TIMESTAMP,
    is_active = FALSE
WHERE id = 3;
```

**Verify soft delete:**
```sql
-- Active products (won't show id=3)
SELECT * FROM products WHERE is_deleted = FALSE;
```

**Result:**
```
 id |   name  |  price  | is_deleted
----+---------+---------+------------
  1 | Laptop  | 999.99  | f
  2 | Mouse   | 49.99   | f
```

**View deleted products:**
```sql
SELECT id, name, deleted_at 
FROM products 
WHERE is_deleted = TRUE;
```

**Result:**
```
 id |   name   |      deleted_at       
----+----------+-----------------------
  3 | Keyboard | 2024-01-20 16:45:30
```

**Complete workflow saved:**

**File: `infra/db/queries/products_crud.sql`**
```sql
-- 1. Create products table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price NUMERIC(10,2) NOT NULL,
    stock INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 2. Insert products
INSERT INTO products (name, description, price, stock)
VALUES 
    ('Laptop', 'High-performance laptop', 999.99, 50),
    ('Mouse', 'Wireless ergonomic mouse', 29.99, 200),
    ('Keyboard', 'Mechanical RGB keyboard', 149.99, 100);

-- 3. Select active products
SELECT * FROM products WHERE is_active = TRUE AND is_deleted = FALSE;

-- 4. Update price
UPDATE products
SET price = 49.99, updated_at = CURRENT_TIMESTAMP
WHERE id = 2;

-- 5. Soft delete
UPDATE products
SET is_deleted = TRUE, deleted_at = CURRENT_TIMESTAMP, is_active = FALSE
WHERE id = 3;

-- Restore deleted product (bonus)
UPDATE products
SET is_deleted = FALSE, deleted_at = NULL, is_active = TRUE
WHERE id = 3;
```

</details>

---

### Question 2: Fix SQL Mistakes

**Question:**
The following SQL commands have errors. Identify and fix each one:

```sql
-- 1.
SELCT * FROM users;

-- 2.
INSERT INTO users (first_name, last_name)
VALUES ("John", "Doe");

-- 3.
UPDATE users
first_name = 'Jane'
WHERE id = 1;

-- 4.
DELETE users WHERE id = 5;

-- 5.
INSERT INTO users (first_name, last_name, email)
VALUES ('John', 'Doe');

-- 6.
UPDATE users
SET first_name = 'Jane',
WHERE id = 1;

-- 7.
SELECT * FORM users WHERE email = 'test@example.com;
```

<details>
<summary>Click to see answer</summary>

**Answer:**

**1. Typo in SELECT**
```sql
-- ❌ Wrong:
SELCT * FROM users;

-- ✓ Correct:
SELECT * FROM users;
```
**Error:** Typo "SELCT" should be "SELECT"

---

**2. Double quotes instead of single quotes**
```sql
-- ❌ Wrong:
INSERT INTO users (first_name, last_name)
VALUES ("John", "Doe");

-- ✓ Correct:
INSERT INTO users (first_name, last_name)
VALUES ('John', 'Doe');
```
**Error:** PostgreSQL requires single quotes `'...'` for string values

---

**3. Missing SET keyword**
```sql
-- ❌ Wrong:
UPDATE users
first_name = 'Jane'
WHERE id = 1;

-- ✓ Correct:
UPDATE users
SET first_name = 'Jane'
WHERE id = 1;
```
**Error:** UPDATE requires SET keyword

---

**4. Missing FROM keyword**
```sql
-- ❌ Wrong:
DELETE users WHERE id = 5;

-- ✓ Correct:
DELETE FROM users WHERE id = 5;
```
**Error:** DELETE requires FROM keyword

---

**5. Column count doesn't match value count**
```sql
-- ❌ Wrong:
INSERT INTO users (first_name, last_name, email)
VALUES ('John', 'Doe');  -- Only 2 values for 3 columns!

-- ✓ Correct:
INSERT INTO users (first_name, last_name, email)
VALUES ('John', 'Doe', 'john@example.com');  -- 3 values for 3 columns
```
**Error:** 3 columns specified but only 2 values provided

---

**6. Comma before WHERE**
```sql
-- ❌ Wrong:
UPDATE users
SET first_name = 'Jane',  -- Extra comma!
WHERE id = 1;

-- ✓ Correct:
UPDATE users
SET first_name = 'Jane'   -- No comma before WHERE
WHERE id = 1;
```
**Error:** No comma should appear before WHERE clause

---

**7. Multiple errors**
```sql
-- ❌ Wrong:
SELECT * FORM users WHERE email = 'test@example.com;

-- ✓ Correct:
SELECT * FROM users WHERE email = 'test@example.com';
```
**Errors:**
- Typo: "FORM" should be "FROM"
- Missing closing quote: `'test@example.com;` should be `'test@example.com';`

</details>

---

### Question 3: Complex Query Challenge

**Question:**
Given a `users` table and `orders` table:

```sql
-- users table
id | first_name | last_name | email | is_active
1  | John       | Doe       | ...   | true
2  | Jane       | Smith     | ...   | true
3  | Bob        | Johnson   | ...   | false

-- orders table
id | user_id | product_name | amount | created_at
1  | 1       | Laptop       | 999.99 | 2024-01-15
2  | 1       | Mouse        | 29.99  | 2024-01-16
3  | 2       | Keyboard     | 149.99 | 2024-01-17
```

Write SQL to:
1. Count total orders per user
2. Calculate total spent per user
3. Find users who spent more than $500
4. Update user status to 'premium' if they spent > $1000

<details>
<summary>Click to see answer</summary>

**Answer:**

**Setup tables:**

```sql
-- Create users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    is_active BOOLEAN DEFAULT TRUE,
    is_premium BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Create orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL REFERENCES users(id),
    product_name VARCHAR(200) NOT NULL,
    amount NUMERIC(10,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO users (first_name, last_name, email)
VALUES 
    ('John', 'Doe', 'john@example.com'),
    ('Jane', 'Smith', 'jane@example.com'),
    ('Bob', 'Johnson', 'bob@example.com');

INSERT INTO orders (user_id, product_name, amount)
VALUES 
    (1, 'Laptop', 999.99),
    (1, 'Mouse', 29.99),
    (2, 'Keyboard', 149.99);
```

---

**1. Count total orders per user**

```sql
SELECT 
    u.id,
    u.first_name,
    u.last_name,
    COUNT(o.id) AS total_orders
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.first_name, u.last_name
ORDER BY total_orders DESC;
```

**Result:**
```
 id | first_name | last_name | total_orders
----+------------+-----------+--------------
  1 | John       | Doe       | 2
  2 | Jane       | Smith     | 1
  3 | Bob        | Johnson   | 0
```

---

**2. Calculate total spent per user**

```sql
SELECT 
    u.id,
    u.first_name,
    u.last_name,
    COALESCE(SUM(o.amount), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.first_name, u.last_name
ORDER BY total_spent DESC;
```

**Result:**
```
 id | first_name | last_name | total_spent
----+------------+-----------+-------------
  1 | John       | Doe       | 1029.98
  2 | Jane       | Smith     | 149.99
  3 | Bob        | Johnson   | 0.00
```

---

**3. Find users who spent more than $500**

```sql
SELECT 
    u.id,
    u.first_name,
    u.last_name,
    u.email,
    SUM(o.amount) AS total_spent
FROM users u
INNER JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.first_name, u.last_name, u.email
HAVING SUM(o.amount) > 500
ORDER BY total_spent DESC;
```

**Result:**
```
 id | first_name | last_name |      email       | total_spent
----+------------+-----------+------------------+-------------
  1 | John       | Doe       | john@example.com | 1029.98
```

**Note:** HAVING filters after GROUP BY, WHERE filters before

---

**4. Update to premium if spent > $1000**

```sql
-- First, find users who qualify
SELECT 
    u.id,
    u.first_name,
    SUM(o.amount) AS total_spent
FROM users u
INNER JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.first_name
HAVING SUM(o.amount) > 1000;
```

**Then update them:**
```sql
UPDATE users
SET is_premium = TRUE
WHERE id IN (
    SELECT u.id
    FROM users u
    INNER JOIN orders o ON u.id = o.user_id
    GROUP BY u.id
    HAVING SUM(o.amount) > 1000
);
```

**Verify update:**
```sql
SELECT id, first_name, last_name, is_premium
FROM users;
```

**Result:**
```
 id | first_name | last_name | is_premium
----+------------+-----------+------------
  1 | John       | Doe       | t          ← Updated!
  2 | Jane       | Smith     | f
  3 | Bob        | Johnson   | f
```

**Alternative: Update with JOIN (PostgreSQL specific)**
```sql
UPDATE users u
SET is_premium = TRUE
FROM (
    SELECT user_id, SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
    HAVING SUM(amount) > 1000
) AS spending
WHERE u.id = spending.user_id;
```

**Bonus: Create a view for user spending**
```sql
CREATE VIEW user_spending AS
SELECT 
    u.id,
    u.first_name,
    u.last_name,
    u.email,
    COUNT(o.id) AS total_orders,
    COALESCE(SUM(o.amount), 0) AS total_spent,
    u.is_premium
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.first_name, u.last_name, u.email, u.is_premium;

-- Now query easily:
SELECT * FROM user_spending WHERE total_spent > 500;
```

</details>

---

**Next Chapter Preview:**

In Chapter 56, we'll cover:
1. **Advanced SELECT queries** - JOIN, GROUP BY, HAVING
2. **Aggregate functions** - COUNT, SUM, AVG, MIN, MAX
3. **Subqueries** - Nested queries
4. **Indexes** - Speed up queries
5. **Query optimization** - Performance tuning

We've mastered basic CRUD—now let's become SQL experts! 🚀
