# Chapter 55: SQL CRUD — `SELECT`, `INSERT`, `UPDATE`, `DELETE` in Depth

> **Goal of this chapter:** Master the four operations behind almost every application: **C**reate (`INSERT`), **R**ead (`SELECT`), **U**pdate (`UPDATE`), **D**elete (`DELETE`). You'll filter, sort, page, and summarize data; use `RETURNING` and **upserts** (`ON CONFLICT`); learn the two most dangerous mistakes in SQL (a missing `WHERE`, and forgetting about `NULL`) and the safety net that protects you (**transactions**); and get a first look at **indexes** and `EXPLAIN`. Every query here was run against a real PostgreSQL 16 and the output is real. Chapter 56 turns these queries into Go.

**Difficulty:** 🟡 Intermediate  **Estimated time:** 6 hours  **Prerequisite:** [Chapters 53 and 54](54-postgresql-data-types.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What is CRUD?](#2-what-is-crud)
3. [A practice playground](#3-a-practice-playground)
4. [READ: `SELECT`](#4-read-select)
5. [Sorting and paging](#5-sorting-and-paging)
6. [Expressions, aliases, `DISTINCT`](#6-expressions-aliases-distinct)
7. [Aggregates and `GROUP BY`](#7-aggregates-and-group-by)
8. [CREATE: `INSERT`](#8-create-insert)
9. [UPDATE](#9-update)
10. [DELETE](#10-delete)
11. [The safety net: transactions](#11-the-safety-net-transactions)
12. [Upserts: `ON CONFLICT`](#12-upserts-on-conflict)
13. [Indexes and `EXPLAIN`](#13-indexes-and-explain)
14. [Saving and running SQL files](#14-saving-and-running-sql-files)
15. [Common mistakes](#15-common-mistakes)
16. [Interview questions](#16-interview-questions)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. What you will learn

- The CRUD ↔ SQL ↔ HTTP mapping
- `SELECT` with `WHERE`, `AND`/`OR`/`NOT`, `IN`, `BETWEEN`, `LIKE`, `IS NULL`
- `ORDER BY`, `LIMIT`, `OFFSET`, `DISTINCT`, expressions and aliases
- Aggregates (`count`, `sum`, `avg`, `min`, `max`), `GROUP BY`, `HAVING`, `FILTER`
- `INSERT` (single, multi-row, `RETURNING`), `UPDATE`, `DELETE` and how to know what they changed
- **Upserts** with `ON CONFLICT DO UPDATE / DO NOTHING`
- **Transactions** as an undo button (`BEGIN` / `ROLLBACK` / `COMMIT`)
- How **indexes** change a query plan (`EXPLAIN`)
- Evaluating **SQL logic order**, the source of many "why doesn't this work?" moments

---

## 2. What is CRUD?

Almost every data-driven application is four verbs applied to nouns:

| Verb | SQL | HTTP (our REST API) | Example |
|------|-----|---------------------|---------|
| **C**reate | `INSERT` | `POST /products` | add a product |
| **R**ead | `SELECT` | `GET /products`, `GET /products/{id}` | list or fetch |
| **U**pdate | `UPDATE` | `PUT`/`PATCH /products/{id}` | change price |
| **D**elete | `DELETE` | `DELETE /products/{id}` | remove |

Learn these four well and you can build most business software. (SQL also has DDL statements: `CREATE TABLE`, `ALTER TABLE`; those define *structure*. CRUD statements, called DML, work with the *data*.)

---

## 3. A practice playground

We'll experiment in a **scratch schema** so nothing touches your real tables, with the products design from Chapter 54 plus two extra columns (`category`, `stock`) to make grouping interesting. Save this as `practice.sql` and run it (it drops and recreates the playground every time, so it is safe to re-run):

```sql
DROP SCHEMA IF EXISTS practice CASCADE;
CREATE SCHEMA practice;
SET search_path TO practice;      -- from now on, unqualified names mean practice.<name>

CREATE TABLE products (
    id          BIGINT        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title       TEXT          NOT NULL CHECK (char_length(btrim(title)) BETWEEN 1 AND 100),
    description TEXT          NOT NULL DEFAULT '',
    price       NUMERIC(12,2) NOT NULL CHECK (price > 0),
    image_url   TEXT          NOT NULL DEFAULT '',
    category    TEXT          NOT NULL DEFAULT 'general',
    stock       INTEGER       NOT NULL DEFAULT 0 CHECK (stock >= 0),
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT now()
);

INSERT INTO products (title, description, price, category, stock) VALUES
  ('Orange',   'Juicy and full of vitamin C', 100.00, 'fruit',     50),
  ('Apple',    'A crunchy apple a day',        40.00, 'fruit',     120),
  ('Banana',   'Great for a quick snack',       5.00, 'fruit',     0),
  ('Carrot',   'Crunchy orange root',          25.50, 'vegetable', 80),
  ('Broccoli', 'Tiny trees',                   60.00, 'vegetable', 15),
  ('Milk',     'One litre, full cream',        90.00, 'dairy',     30),
  ('Cheese',   'Aged cheddar',                350.00, 'dairy',     8),
  ('Bread',    'Fresh sourdough',              70.00, 'bakery',    22);
```

```bash
docker exec -i shop-db psql -U postgres -d ecommerce < practice.sql
docker exec -it shop-db psql -U postgres -d ecommerce     # then type: SET search_path TO practice;
```

`psql` tips: `\dt` lists tables, `\d products` describes one, `\x` toggles vertical display for wide rows, `\q` quits. **`search_path` is per session**, so set it again in each new `psql` (or prefix names: `practice.products`).

---

## 4. READ: `SELECT`

The general shape (the parts in brackets are optional):

```sql
SELECT   columns
FROM     table
[WHERE   condition]
[GROUP BY ...] [HAVING ...]
[ORDER BY ...]
[LIMIT n] [OFFSET m];
```

```sql
SELECT id, title, price FROM products;
```

```
 id |  title   | price
----+----------+--------
  1 | Orange   | 100.00
  2 | Apple    |  40.00
  3 | Banana   |   5.00
  4 | Carrot   |  25.50
  5 | Broccoli |  60.00
  6 | Milk     |  90.00
  7 | Cheese   | 350.00
  8 | Bread    |  70.00
(8 rows)
```

`SELECT *` means "all columns": fine for exploring in `psql`, **not** for application code (Chapter 53: name your columns).

### `WHERE`: choosing rows

```sql
SELECT title, price FROM products WHERE price > 50 AND category <> 'dairy';
```
```
  title   | price
----------+--------
 Orange   | 100.00
 Broccoli |  60.00
 Bread    |  70.00
```

| Operator | Meaning |
|----------|---------|
| `=`, `<>` (or `!=`), `<`, `<=`, `>`, `>=` | comparisons (note: **single** `=` for equality) |
| `AND`, `OR`, `NOT` | logic (`AND` binds tighter than `OR`: use parentheses!) |
| `IN ('a','b')` | membership |
| `BETWEEN a AND b` | inclusive range |
| `LIKE 'a%'`, `ILIKE` | pattern (`%` any run, `_` one char); `ILIKE` ignores case |
| `IS NULL`, `IS NOT NULL` | NULL tests (never `= NULL`; Chapter 54) |

```sql
SELECT title FROM products WHERE category IN ('fruit', 'bakery') OR price BETWEEN 60 AND 100;
SELECT title FROM products WHERE title LIKE '%an%';
```
```
  title
----------
 Orange
 Apple
 Banana
 Broccoli
 Milk
 Bread
(6 rows)

 title
--------
 Orange
 Banana
(2 rows)
```

The first query returned the union of two conditions; the second shows `LIKE '%an%'` matching "Or**an**ge" and "B**an**ana". `BETWEEN 60 AND 100` includes both ends: Broccoli (60) and Orange (100).

**Watch operator precedence.** `a OR b AND c` means `a OR (b AND c)`. When mixing them, always add parentheses so the reader (and the database) can't disagree:

```sql
WHERE category = 'fruit' AND (price < 10 OR stock = 0)
```

### `NULL` bites in `WHERE`

A row is returned only if the condition is **true**; rows where it is `NULL` (unknown) are dropped silently. Our `stock` column is `NOT NULL`, but on a nullable column `WHERE discount <> 0` would silently skip every row whose discount is `NULL`. Handle it explicitly: `WHERE discount IS DISTINCT FROM 0` (treats NULL as a normal, different value) or `COALESCE(discount, 0) <> 0`.

---

## 5. Sorting and paging

**Rows have no inherent order.** Without `ORDER BY`, PostgreSQL may return them in any order, and that order can change after an update or a vacuum. If order matters, say so.

```sql
SELECT title, price FROM products ORDER BY price DESC LIMIT 3;
```
```
 title  | price
--------+--------
 Cheese | 350.00
 Orange | 100.00
 Milk   |  90.00
```

Sort by several columns; later ones break ties:

```sql
SELECT title, category, price FROM products ORDER BY category ASC, price DESC;
```
```
  title   | category  | price
----------+-----------+--------
 Bread    | bakery    |  70.00
 Cheese   | dairy     | 350.00
 Milk     | dairy     |  90.00
 Orange   | fruit     | 100.00
 Apple    | fruit     |  40.00
 Banana   | fruit     |   5.00
 Broccoli | vegetable |  60.00
 Carrot   | vegetable |  25.50
```

`ASC` (default) ascending, `DESC` descending; text sorts by the database's collation (usually language-aware, e.g. case handled sensibly); `NULL`s sort **last** ascending (`NULLS FIRST/LAST` overrides).

### `LIMIT` and `OFFSET`: pagination

```sql
SELECT id, title FROM products ORDER BY id LIMIT 3 OFFSET 3;   -- "page 2" with 3 per page
```
```
 id |  title
----+----------
  4 | Carrot
  5 | Broccoli
  6 | Milk
```

`OFFSET (page-1) × page_size`. Two rules: **always pair `LIMIT` with a deterministic `ORDER BY`** (ideally including a unique column such as `id` as the final tiebreaker), or pages can overlap or skip rows. And `OFFSET` gets slow on huge tables (the database still walks the skipped rows); Chapters 62–63 explore this and the alternative (**keyset pagination**).

---

## 6. Expressions, aliases, `DISTINCT`

`SELECT` can compute values, and `AS` names them:

```sql
SELECT title, price, price * 0.9 AS discounted, round(price * 1.15, 1) AS with_tax,
       upper(left(title, 3)) AS code
FROM products WHERE id <= 3;
```
```
 title  | price  | discounted | with_tax | code
--------+--------+------------+----------+------
 Orange | 100.00 |     90.000 |    115.0 | ORA
 Apple  |  40.00 |     36.000 |     46.0 | APP
 Banana |   5.00 |      4.500 |      5.8 | BAN
```

(`5.00 × 1.15 = 5.75`, which `round(…, 1)` turns into `5.8` (half up for `NUMERIC`). Note the extra decimal digits in `discounted`: multiplying `NUMERIC(12,2)` by `0.9` produces scale 3. Exactness has a shape.)

**Aliases** (`AS name`) can be used in `ORDER BY` but **not in `WHERE`**, because of the logical order in which SQL evaluates a query:

```
1. FROM       (which table)
2. WHERE      (filter rows)
3. GROUP BY   (make groups)
4. HAVING     (filter groups)
5. SELECT     (compute output columns and aliases)
6. DISTINCT
7. ORDER BY   (can see aliases)
8. LIMIT/OFFSET
```

`WHERE` runs *before* `SELECT`, so the alias `discounted` doesn't exist yet. Repeat the expression, or wrap the query in a subquery/CTE.

`DISTINCT` removes duplicate result rows:

```sql
SELECT DISTINCT category FROM products ORDER BY category;
```
```
 category
-----------
 bakery
 dairy
 fruit
 vegetable
```

---

## 7. Aggregates and `GROUP BY`

**Aggregate functions** collapse many rows into one value: `count`, `sum`, `avg`, `min`, `max`.

```sql
SELECT count(*) AS n, sum(stock) AS units, round(avg(price), 2) AS avg_price,
       min(price) AS cheapest, max(price) AS priciest
FROM products;
```
```
 n | units | avg_price | cheapest | priciest
---+-------+-----------+----------+----------
 8 |   325 |     92.56 |     5.00 |   350.00
```

`count(*)` counts rows; `count(col)` counts **non-NULL** values of `col`; `sum`/`avg` ignore NULLs (and return NULL for zero rows, so wrap with `COALESCE(sum(x), 0)`).

**`GROUP BY`** computes aggregates *per group*:

```sql
SELECT category, count(*) AS n, sum(price * stock) AS stock_value
FROM products GROUP BY category ORDER BY stock_value DESC;
```
```
 category  | n | stock_value
-----------+---+-------------
 fruit     | 3 |     9800.00
 dairy     | 2 |     5500.00
 vegetable | 2 |     2940.00
 bakery    | 1 |     1540.00
```

Rule: every column in `SELECT` must be either **in `GROUP BY`** or **inside an aggregate**. (`SELECT title, count(*) … GROUP BY category` is an error: which title of the group should it show?)

**`HAVING`** filters *groups* (after aggregation), where `WHERE` filters *rows* (before):

```sql
SELECT category, count(*) AS n FROM products
GROUP BY category HAVING count(*) >= 2 ORDER BY n DESC, category;
```
```
 category  | n
-----------+---
 fruit     | 3
 dairy     | 2
 vegetable | 2
```

`FILTER` computes several conditional counts in one pass:

```sql
SELECT count(*) FILTER (WHERE stock = 0) AS sold_out, count(*) FILTER (WHERE price > 50) AS premium
FROM products;
```
```
 sold_out | premium
----------+---------
        1 |       5
```

---

## 8. CREATE: `INSERT`

```sql
INSERT INTO products (title, price) VALUES ('Mango', 75.00);
```

Rules:

- **List the columns** you're providing. Omitted columns get their `DEFAULT` (or `NULL`, and a `NOT NULL` column without a default is an error).
- Values must match the column order and types you listed.
- `GENERATED ALWAYS` identity columns must be **omitted**.
- Multiple rows in one statement (one round trip, one atomic operation):

```sql
INSERT INTO products (title, price, stock) VALUES ('Grapes', 30, 10), ('Lemon', 8, 200);
```

- **`RETURNING`** hands back what was stored (generated ID, defaults, rounded values), the idiom Go uses (Chapter 53):

```sql
INSERT INTO products (title, price) VALUES ('Kiwi', 12.5) RETURNING id, title, price, created_at;
```

- `INSERT … SELECT` copies rows from a query: `INSERT INTO archive (title, price) SELECT title, price FROM products WHERE stock = 0;`

Constraints protect you when a value is invalid: the *whole statement* fails and **nothing** is inserted:

```
ERROR:  new row for relation "products" violates check constraint "products_stock_check"
```

---

## 9. UPDATE

```sql
UPDATE table SET column = expression [, column = expression ...] WHERE condition [RETURNING ...];
```

```sql
UPDATE products SET price = price * 1.10, updated_at = now()
WHERE category = 'dairy'
RETURNING id, title, price;
```
```
 id | title  | price
----+--------+--------
  6 | Milk   |  99.00
  7 | Cheese | 385.00
(2 rows)

UPDATE 2
```

- The right-hand side can use the **current** values (`price * 1.10`): the database evaluates it atomically for each row, which is exactly what makes "decrement stock" safe under concurrency: `SET stock = stock - 1` is one atomic step, whereas *read the stock into Go, subtract, write it back* is a race (Chapter 67).
- `UPDATE 2` = **the command tag reports rows affected**. Go exposes this as `result.RowsAffected()`; it is how you detect "no such product".
- **An update that violates a constraint fails entirely:**

```sql
UPDATE products SET stock = stock - 5 WHERE id = 1 RETURNING id, title, stock;   -- ok: 45 left
UPDATE products SET stock = stock - 500 WHERE id = 2;
```
```
 id | title  | stock
----+--------+-------
  1 | Orange |    45

ERROR:  new row for relation "products" violates check constraint "products_stock_check"
DETAIL:  Failing row contains (2, Apple, A crunchy apple a day, 40.00, , fruit, -380, …).
```

  You can't oversell: the `CHECK (stock >= 0)` rejected `-380`, even though no Go code checked anything. (This is the "last line of defense" idea from Chapter 53.)

- **Updating a row that doesn't exist is not an error**:

```sql
UPDATE products SET title = 'Green Apple' WHERE id = 999 RETURNING id;
```
```
 id
----
(0 rows)

UPDATE 0
```

  Zero rows matched. If "not found" should be an error in your API (404), *you* must check `RowsAffected() == 0` (or use `RETURNING` and `sql.ErrNoRows`).

- ⚠️ **Maintain `updated_at` yourself** (or with a trigger); PostgreSQL won't do it for you.

---

## 10. DELETE

```sql
DELETE FROM table WHERE condition [RETURNING ...];
```

```sql
DELETE FROM products WHERE stock = 0 RETURNING id, title;
```
```
 id | title
----+--------
  3 | Banana
(1 row)

DELETE 1
```

Like `UPDATE`, `DELETE` reports the rows affected and can `RETURNING` the removed rows (great for audit logs or "undo" features).

| Statement | What it does | Speed | Can be rolled back? | Fires per-row triggers |
|-----------|--------------|-------|---------------------|------------------------|
| `DELETE FROM t` | removes rows one by one | slow on big tables | yes | yes |
| `TRUNCATE t` | empties the whole table instantly | fast | yes (in PostgreSQL, inside a transaction) | no |
| `DROP TABLE t` | removes the table itself | fast | yes (transactional DDL) | no |

Design note: many systems avoid physically deleting business data. A **soft delete** adds a `deleted_at TIMESTAMPTZ` column (`NULL` = live) and every query filters `WHERE deleted_at IS NULL`. It preserves history and enables undo, at the cost of complexity (unique constraints, forgotten filters, privacy laws that *require* real deletion). Choose deliberately.

---

## 11. The safety net: transactions

The most dangerous statement in SQL is the one you forgot the `WHERE` on:

```sql
UPDATE products SET price = 1;          -- every product now costs 1. There is no "are you sure?"
```

A **transaction** groups statements into an all-or-nothing unit. Until you `COMMIT`, nothing is permanent, and `ROLLBACK` undoes everything:

```sql
BEGIN;
UPDATE products SET price = 1;
SELECT count(*) AS rows_priced_1 FROM products WHERE price = 1;
ROLLBACK;
SELECT count(*) AS rows_priced_1_after_rollback FROM products WHERE price = 1;
```
```
BEGIN
UPDATE 7
 rows_priced_1
---------------
             7

ROLLBACK
 rows_priced_1_after_rollback
------------------------------
                            0
```

Inside the transaction you saw the damage (7 rows); after `ROLLBACK` it is as if it never happened. **Habit for manual production work:** `BEGIN;` → run the change → check `SELECT` and the row count → `COMMIT;` only if it's what you expected.

Outside an explicit `BEGIN`, PostgreSQL wraps **each statement in its own tiny transaction** (autocommit), so a single `UPDATE`/`INSERT` is already atomic: all rows or none.

Transactions are also how applications keep multi-step changes consistent ("charge the card **and** create the order **and** reduce stock": all or nothing). Chapter 60 covers them in Go, with isolation levels and deadlocks.

---

## 12. Upserts: `ON CONFLICT`

"Insert this row, or if it already exists, update it" is common enough to have its own syntax. It needs a unique index to define "already exists":

```sql
CREATE UNIQUE INDEX products_title_key ON products (lower(title));

INSERT INTO products (title, price) VALUES ('Apple', 45)
ON CONFLICT (lower(title)) DO UPDATE SET price = EXCLUDED.price, updated_at = now()
RETURNING id, title, price;

INSERT INTO products (title, price) VALUES ('apple', 1)
ON CONFLICT (lower(title)) DO NOTHING;

INSERT INTO products (title, price) VALUES ('Apple', 1);       -- no ON CONFLICT clause
```
```
 id | title | price
----+-------+-------
  2 | Apple | 45.00
(1 row)

INSERT 0 1
INSERT 0 0
ERROR:  duplicate key value violates unique constraint "products_title_key"
DETAIL:  Key (lower(title))=(apple) already exists.
```

- `EXCLUDED` is the row you *tried* to insert; `DO UPDATE SET price = EXCLUDED.price` overwrites the existing row's price with it. The existing row kept its `id` (2).
- `DO NOTHING` silently skips the conflicting row (`INSERT 0 0`: zero rows inserted).
- Without `ON CONFLICT`, the same insert fails with the unique violation (SQLSTATE `23505`) that our Go store translates into `ErrEmailTaken` (Chapter 53).
- An upsert is **atomic**: no window between "check" and "write" for another session to sneak in, unlike `SELECT` followed by `INSERT`/`UPDATE`.

---

## 13. Indexes and `EXPLAIN`

An **index** is a separate, sorted lookup structure (usually a B-tree) that lets the database find rows without reading the whole table, like a book's index versus reading every page. **`EXPLAIN`** shows the plan the database chose. On a tiny table everything is a full scan (`Seq Scan`), which is *correct*: reading 8 rows is cheaper than consulting an index:

```
EXPLAIN SELECT * FROM products WHERE id = 3;

 Seq Scan on products  (cost=0.00..1.09 rows=1 width=172)
   Filter: (id = 3)
```

Let's make the table realistic: 20,000 more rows, and refresh the statistics (`ANALYZE`) the planner uses:

```sql
INSERT INTO products (title, price)
SELECT 'Bulk ' || g, (g % 100) + 1 FROM generate_series(1, 20000) g;
ANALYZE products;

EXPLAIN SELECT * FROM products WHERE id = 5000;
EXPLAIN SELECT * FROM products WHERE price = 42;
```
```
 Index Scan using products_pkey on products  (cost=0.29..8.30 rows=1 width=52)
   Index Cond: (id = 5000)

 Seq Scan on products  (cost=0.00..457.09 rows=200 width=52)
   Filter: (price = '42'::numeric)
```

- `id = 5000` uses the **primary key's index** (`Index Scan`, estimated cost 8.3).
- `price = 42` has no index, so it must read **every row** (`Seq Scan`, cost 457.09), 55× more work. Add one:

```sql
CREATE INDEX products_price_idx ON products (price);
ANALYZE products;
EXPLAIN SELECT * FROM products WHERE price = 42;
```
```
 Bitmap Heap Scan on products  (cost=5.84..221.27 rows=200 width=52)
   Recheck Cond: (price = '42'::numeric)
   ->  Bitmap Index Scan on products_price_idx  (cost=0.00..5.79 rows=200 width=0)
         Index Cond: (price = '42'::numeric)
```

The estimated cost falls from 457 to 221 (the query matches 200 rows, 1% of the table, so the planner picks a "bitmap" scan that visits only the relevant pages).

Guidelines:

- **Costs are estimates in arbitrary units**, not milliseconds. Use `EXPLAIN ANALYZE` to *run* the query and see real times.
- Index columns you **filter, join, or sort by frequently** on big tables. Primary keys and `UNIQUE` constraints already create one.
- Indexes are not free: every `INSERT/UPDATE/DELETE` must maintain them, and they take disk space. Don't index everything.
- An index helps only queries using the **same expression** (`lower(title)`, not `title`; `x LIKE 'abc%'` but not `'%abc'`).
- Always **measure** before and after; guessing is how you get useless indexes.

Chapter 58 covers creating indexes properly through migrations.

---

## 14. Saving and running SQL files

Keep useful SQL in files under version control instead of retyping (or trusting your shell history):

```bash
docker exec -i shop-db psql -U postgres -d ecommerce -v ON_ERROR_STOP=1 -f - < practice.sql   # a whole file
docker exec -i shop-db psql -U postgres -d ecommerce -c "SELECT count(*) FROM practice.products"   # one statement
```

- `-v ON_ERROR_STOP=1` makes `psql` stop at the first error (default: it plows on, hiding failures in long scripts).
- Inside `psql`, `\i file.sql` runs a file that is on the *psql* client's machine (with Docker that means inside the container, so piping via stdin, as above, is easier).
- Clean up the playground: `DROP SCHEMA practice CASCADE;`.

Project convention from here on: **schema changes** live in versioned files (Chapter 58 migrations); **queries** the application runs live in `.sql` files embedded in the Go binary (Chapter 53); **ad-hoc exploration** stays in scratch files outside the repo.

---

## 15. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | `UPDATE`/`DELETE` **without `WHERE`** | Every row changed or destroyed | `BEGIN`, check the affected count, then `COMMIT`; write the `WHERE` first |
| 2 | `WHERE col = NULL` | Matches nothing | `IS NULL` |
| 3 | Mixing `AND`/`OR` without parentheses | Wrong rows | Parenthesize |
| 4 | `LIMIT` without `ORDER BY` | Arbitrary, unstable pages | Deterministic `ORDER BY` (with `id` tiebreaker) |
| 5 | Using an alias in `WHERE` | `column "x" does not exist` | Repeat the expression or use a subquery |
| 6 | Selecting a non-grouped column with `GROUP BY` | Error | Add it to `GROUP BY` or aggregate it |
| 7 | `WHERE` vs `HAVING` confusion | Wrong results / errors | `WHERE` filters rows, `HAVING` filters groups |
| 8 | Read-modify-write in application code (`SELECT stock`, subtract, `UPDATE`) | Lost updates under concurrency | Atomic `SET stock = stock - 1` (+ `CHECK`) |
| 9 | Assuming `UPDATE` errors when nothing matched | Silent no-op | Check rows affected / `RETURNING` |
| 10 | `SELECT *` in application code | Breaks/leaks on schema change | List columns |
| 11 | Deep `OFFSET` on huge tables | Slow pages | Keyset pagination (Chapter 63) |
| 12 | Building SQL by string concatenation | Injection | Placeholders `$1…` (Chapter 53) |
| 13 | Forgetting `updated_at` | Misleading audit data | Set it in the `UPDATE` or use a trigger |
| 14 | Indexing blindly / never checking `EXPLAIN` | Slow writes, no read benefit | Measure with `EXPLAIN (ANALYZE)` |
| 15 | Running ad-hoc changes on production without a transaction | No undo | Always `BEGIN` first |

---

## 16. Interview questions

**Q1. `WHERE` vs `HAVING`?**
`WHERE` filters individual rows before grouping; `HAVING` filters groups after aggregation (so it can use aggregates).

**Q2. Logical order of a `SELECT`?**
`FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`. That's why aliases aren't visible in `WHERE`.

**Q3. What does `RETURNING` do?**
Returns columns of the rows an `INSERT/UPDATE/DELETE` affected, in the same statement (no second query).

**Q4. What is an upsert, and why prefer it to check-then-write?**
`INSERT … ON CONFLICT DO UPDATE/NOTHING`: atomic "insert or update", free of the race between a check and a write.

**Q5. `DELETE` vs `TRUNCATE`?**
`DELETE` removes chosen rows one by one (with `WHERE`, triggers); `TRUNCATE` empties the table at once, much faster, no `WHERE`.

**Q6. How do you make a manual destructive change safe?**
Run it inside `BEGIN`, verify the row count and results, then `COMMIT`, or `ROLLBACK` if wrong.

**Q7. Why is `SET stock = stock - 1` safer than reading and writing from Go?**
It's a single atomic operation evaluated by the database under a row lock; the read-modify-write approach has a race window where two requests read the same value.

**Q8. When does an index *not* help?**
Tiny tables, low-selectivity columns (most rows match), queries that don't use the indexed expression (`LIKE '%x'`, `lower(col)` without an expression index), or when write cost outweighs read benefit.

**Q9. Difference between `count(*)` and `count(col)`?**
`count(*)` counts rows; `count(col)` counts rows where `col` is not NULL.

---

## 17. Exercises

Use the playground from §3 (re-run `practice.sql` to reset it).

### Exercise 1: Read
Write queries for: (a) titles and prices of fruit under 50, cheapest first; (b) the 2 most expensive non-dairy products; (c) the average price per category, rounded to 2 decimals, highest first.

<details><summary>Solution</summary>

```sql
SELECT title, price FROM products WHERE category = 'fruit' AND price < 50 ORDER BY price;
SELECT title, price FROM products WHERE category <> 'dairy' ORDER BY price DESC LIMIT 2;
SELECT category, round(avg(price), 2) AS avg_price FROM products GROUP BY category ORDER BY avg_price DESC;
```
</details>

### Exercise 2: Create
Insert three products in one statement, returning their IDs. Then try inserting one with `price = 0`. What happens to the *other* rows in a failed multi-row insert?

<details><summary>Solution</summary>

`INSERT INTO products (title, price) VALUES ('A',1),('B',2),('C',3) RETURNING id;` The `price = 0` insert fails on `products_price_check`. In a multi-row `INSERT` **one bad row fails the entire statement**; nothing is inserted.
</details>

### Exercise 3: Update safely
Give every product in the `fruit` category a 5% discount, but only if `stock > 0`, and show the new prices. Do it inside a transaction and roll it back.

<details><summary>Solution</summary>

```sql
BEGIN;
UPDATE products SET price = round(price * 0.95, 2), updated_at = now()
WHERE category = 'fruit' AND stock > 0 RETURNING id, title, price;
ROLLBACK;
```
`round(..., 2)` keeps two decimals (otherwise the column would round anyway, but explicit is clearer).
</details>

### Exercise 4: The missing `WHERE`
Deliberately run `DELETE FROM products;` inside `BEGIN … ROLLBACK`. How many rows does it report? Verify they're back afterwards.

<details><summary>Solution</summary>

`DELETE 8` (or however many there are); after `ROLLBACK`, `SELECT count(*)` returns the original count.
</details>

### Exercise 5: Find the bug

```sql
SELECT category, title, avg(price) FROM products GROUP BY category;
SELECT title, price * 2 AS double FROM products WHERE double > 100;
SELECT * FROM products WHERE stock = NULL;
```

<details><summary>Solution</summary>

1. `title` is neither grouped nor aggregated → error; drop it or `GROUP BY category, title`.
2. The alias `double` isn't visible in `WHERE` → use `WHERE price * 2 > 100` (the alias also collides with the type name `double`; pick a clearer alias).
3. `stock = NULL` is never true → `WHERE stock IS NULL` (and `stock` is `NOT NULL` here, so it would return nothing anyway).
</details>

### Exercise 6: Upsert a stock count
Write one statement that sets the stock of `'Mango'` to 25, creating the product (price 75) if it doesn't exist. Which index do you need?

<details><summary>Solution</summary>

```sql
CREATE UNIQUE INDEX IF NOT EXISTS products_title_key ON products (lower(title));
INSERT INTO products (title, price, stock) VALUES ('Mango', 75, 25)
ON CONFLICT (lower(title)) DO UPDATE SET stock = EXCLUDED.stock, updated_at = now();
```
A unique index (or constraint) on the conflict target.
</details>

### Exercise 7 (challenge): Read the plan
Add 20,000 rows as in §13. Compare `EXPLAIN ANALYZE` for `WHERE price = 42` before and after `CREATE INDEX`. Then try `WHERE price > 1` (matches almost everything). Does the planner still use the index? Why?

<details><summary>Solution</summary>

Before: `Seq Scan`. After: `Bitmap Index Scan`, faster. For `price > 1`, nearly all rows match, so reading the index *and* the table is worse than one sequential pass: the planner correctly returns to `Seq Scan`. An index pays off when the query is **selective**.
</details>

---

## 18. Quiz

1. Which clause filters groups?
2. What does `UPDATE … WHERE id = 999` report when no row has that ID?
3. Why must `LIMIT` come with `ORDER BY`?
4. What does `EXCLUDED` refer to in `ON CONFLICT`?
5. How do you undo a mistaken `UPDATE` you haven't committed?
6. What is the difference between `count(*)` and `count(price)`?
7. Why can't you use a `SELECT` alias in `WHERE`?
8. Why is `SET stock = stock - 1` better than read-then-write?

<details><summary>Answers</summary>

1. `HAVING`.
2. `UPDATE 0`, with no error.
3. Without an order the rows come back in arbitrary order, so pages can repeat or skip rows.
4. The row that was proposed for insertion.
5. `ROLLBACK`.
6. `count(*)` counts all rows; `count(price)` skips rows where `price` is NULL.
7. `WHERE` is evaluated before `SELECT` computes the aliases.
8. It is atomic in the database; read-then-write races between requests.
</details>

---

## 19. Summary

- **CRUD** = `INSERT` / `SELECT` / `UPDATE` / `DELETE` ↔ `POST` / `GET` / `PUT`/`PATCH` / `DELETE`.
- `SELECT`: `WHERE` filters rows, `GROUP BY` groups, `HAVING` filters groups, `ORDER BY` sorts (always specify it), `LIMIT/OFFSET` pages. Remember the logical evaluation order and three-valued `NULL` logic.
- `INSERT … RETURNING` returns what was stored; multi-row inserts are all-or-nothing; constraints reject bad data no matter where it comes from.
- `UPDATE`/`DELETE` report **rows affected**; an update matching zero rows is *not* an error; keep changes **atomic** in SQL (`stock = stock - 1`).
- **A missing `WHERE` is the classic disaster**; wrap manual changes in `BEGIN … ROLLBACK/COMMIT`.
- **Upserts** (`ON CONFLICT`) are atomic and need a unique index.
- **Indexes** speed selective lookups (`EXPLAIN` shows the plan) but cost writes and space; measure, don't guess.

### ➡️ What's next?

[Chapter 56](56-crud-in-go.md) puts all of this into Go: a PostgreSQL-backed **product store** with `List`, `Get`, `Create`, `Update`, and `Delete`, plus new `PUT`/`DELETE` endpoints, tests, and the migration of products from memory to the database.
