# Chapter 54: PostgreSQL Data Types — Choosing the Right Type for Every Column

> **Goal of this chapter:** Learn the data types you will use every day in PostgreSQL (integers, exact and floating-point numbers, text, booleans and `NULL`, dates and times, UUIDs, JSON, arrays, enums), **what each one costs and where each one bites**, and how each maps to Go. Every claim is demonstrated with a real query and its real output. By the end you'll design the `products` table for Chapter 56 and know exactly *why* every column has the type it has (including why a price must **never** be a `float`).

**Difficulty:** 🟡 Intermediate  **Estimated time:** 5 hours  **Prerequisite:** [Chapter 53](53-users-in-postgresql.md)

---

## 📚 Table of Contents

1. [Why types matter](#1-why-types-matter)
2. [How to follow along](#2-how-to-follow-along)
3. [Integers](#3-integers)
4. [Exact numbers: `NUMERIC`](#4-exact-numbers-numeric)
5. [Floating point: `REAL` / `DOUBLE PRECISION`](#5-floating-point-real--double-precision)
6. [Money: the classic trap](#6-money-the-classic-trap)
7. [Text: `TEXT`, `VARCHAR`, `CHAR`](#7-text-text-varchar-char)
8. [Boolean and `NULL`](#8-boolean-and-null)
9. [Dates and times](#9-dates-and-times)
10. [UUID, JSONB, arrays, enums, bytea](#10-uuid-jsonb-arrays-enums-bytea)
11. [How much space do types take?](#11-how-much-space-do-types-take)
12. [PostgreSQL types ↔ Go types](#12-postgresql-types--go-types)
13. [Choosing: a decision guide](#13-choosing-a-decision-guide)
14. [Designing the `products` table](#14-designing-the-products-table)
15. [Common mistakes](#15-common-mistakes)
16. [Interview questions](#16-interview-questions)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. Why types matter

A column's type is a **promise the database enforces**:

| Benefit | Example |
|---------|---------|
| **Validation** | `'free'` can't be stored in a numeric column; `2026-02-30` can't be a date |
| **Correctness** | exact decimal arithmetic vs. binary floating-point rounding |
| **Storage** | a `smallint` takes 2 bytes, a `bigint` 8; across 100 million rows that's real money |
| **Speed** | comparing integers is faster than comparing strings; the right type enables the right index |
| **Expressiveness** | date arithmetic (`+ INTERVAL '1 month'`), JSON operators, array search |

Go is statically typed for the same reasons: the more the *compiler and database* know, the fewer bugs reach production.

---

## 2. How to follow along

Every example below runs in `psql` against the container from Chapter 52:

```bash
docker exec -it shop-db psql -U postgres -d ecommerce
```

(Or pipe a file with `docker exec -i shop-db psql -U postgres -d ecommerce < file.sql`.) Statements end with `;`. `SELECT expr;` with no table simply evaluates an expression: a perfect calculator for experiments. `::type` (or `CAST(x AS type)`) converts a value to a type.

---

## 3. Integers

| Type | Alias | Size | Range | Go type |
|------|-------|------|-------|---------|
| `SMALLINT` | `INT2` | 2 bytes | −32,768 … 32,767 | `int16` |
| `INTEGER` | `INT`, `INT4` | 4 bytes | −2,147,483,648 … 2,147,483,647 | `int32` |
| `BIGINT` | `INT8` | 8 bytes | ±9.22 × 10¹⁸ | `int64` (`int` on 64-bit) |

Real check of the limits, and what happens when you cross them:

```sql
SELECT 32767::smallint AS smallint_max, 2147483647::int AS int_max, 9223372036854775807::bigint AS bigint_max;
SELECT 32768::smallint;
SELECT 2147483647::int + 1;
```

```
 smallint_max |  int_max   |     bigint_max
--------------+------------+---------------------
        32767 | 2147483647 | 9223372036854775807

ERROR:  smallint out of range
ERROR:  integer out of range
```

PostgreSQL **raises an error** on overflow. It never silently wraps like Go's integer arithmetic (Chapter 33) does; the database is stricter than your code.

**Which one?**

- Default to `INTEGER` for ordinary counts (quantity in stock, ratings).
- Use **`BIGINT` for primary keys** of tables that may grow. Running out of a 2.1-billion `INTEGER` id is a famous, painful outage (a table inserting 1,000 rows/second exhausts it in ~25 days). Our `users.id` is already `BIGINT`.
- `SMALLINT` only when the domain is tiny and fixed (a star rating 1–5) and you have millions of rows.

### Integer division

```sql
SELECT 7 / 2 AS int_div, 7 / 2.0 AS numeric_div, 7 % 2 AS remainder;
```

```
 int_div |    numeric_div     | remainder
---------+--------------------+-----------
       3 | 3.5000000000000000 |         1
```

Just like Go: integer ÷ integer = integer (truncated). Make one side non-integer for a fractional result. Dividing by zero is an error (`division by zero`), not `Inf`.

### Auto-increment IDs

```sql
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY     -- modern, standard (use this)
id BIGSERIAL PRIMARY KEY                                -- older PostgreSQL-specific spelling
```

Both use a **sequence** behind the scenes. Gaps are normal (a rolled-back insert still consumed a number); never assume IDs are contiguous.

---

## 4. Exact numbers: `NUMERIC`

`NUMERIC(precision, scale)` (alias `DECIMAL`) stores **exact decimal** numbers:

- **precision** = total number of significant digits;
- **scale** = digits after the decimal point.

`NUMERIC(10,2)` holds up to 8 digits before the point and 2 after: `99999999.99`.

```sql
SELECT 19.995::numeric(10,2) AS rounded, 123.456::numeric(10,1) AS one_decimal;
SELECT 12345678.9::numeric(5,2);
```

```
 rounded | one_decimal
---------+-------------
   20.00 |       123.5

ERROR:  numeric field overflow
DETAIL:  A field with precision 5, scale 2 must round to an absolute value less than 10^3.
```

Two behaviors to remember: too many decimals are **rounded** (half away from zero), but too many *integer digits* are an **error**.

Exactness in action:

```sql
SELECT 0.1::numeric + 0.2::numeric AS numeric_sum;   -- 0.3
SELECT sum(0.1::numeric) AS numeric_total FROM generate_series(1,10);
```

```
 numeric_sum
-------------
         0.3

 numeric_total
---------------
           1.0
```

`NUMERIC` is slower than integers/floats (arbitrary-precision arithmetic in software) and takes more space, but for money and anything legally or financially significant, **correctness beats speed**.

Unconstrained `NUMERIC` (no precision/scale) accepts any size, useful for scientific data, but **constrain money** so bad data is rejected.

---

## 5. Floating point: `REAL` / `DOUBLE PRECISION`

| Type | Alias | Size | Precision | Go type |
|------|-------|------|-----------|---------|
| `REAL` | `FLOAT4` | 4 bytes | ~6 decimal digits | `float32` |
| `DOUBLE PRECISION` | `FLOAT8` | 8 bytes | ~15 decimal digits | `float64` |

These are IEEE-754 binary floating-point, exactly like Go's `float32`/`float64` (Chapter 33). They are **fast** and have a huge range, but they **cannot represent most decimal fractions exactly**:

```sql
SELECT 0.1::float8 + 0.2::float8 AS float_sum, 0.1::numeric + 0.2::numeric AS numeric_sum;
SELECT (0.1::float8 + 0.2::float8) = 0.3::float8 AS float_equal,
       (0.1::numeric + 0.2::numeric) = 0.3::numeric AS numeric_equal;
SELECT sum(0.1::float8) AS float_total FROM generate_series(1,10);
```

```
      float_sum      | numeric_sum
---------------------+-------------
 0.30000000000000004 |         0.3

 float_equal | numeric_equal
-------------+---------------
 f           | t

    float_total
--------------------
 0.9999999999999999
```

Ten additions of `0.1` do **not** give `1`. Never compare floats with `=` and never use them where exact decimals matter. Floats are right for **measurements** (temperature, sensor readings, latitude/longitude, machine-learning scores) where tiny errors are inherent anyway.

Special values exist: `'NaN'`, `'Infinity'`. Note also that PostgreSQL rounds *floats* half-to-even but *numerics* half-up:

```sql
SELECT round(2.5::numeric) AS round_25, round(2.5::float8) AS round_float_25, round(3.5::float8) AS round_float_35;
```

```
 round_25 | round_float_25 | round_float_35
----------+----------------+----------------
        3 |              2 |              4
```

Same function name, different results by type. That kind of surprise is why financial code sticks to one exact type.

---

## 6. Money: the classic trap

**Never store money in `float`/`double precision`/`REAL`.** The errors above compound across additions, discounts, and tax calculations, and eventually an invoice is off by a cent, which is enough to fail an audit.

Two correct designs:

| Design | Column | Example: ৳12.50 / $12.50 | Pros | Cons |
|--------|--------|--------------------------|------|------|
| **Exact decimal** | `NUMERIC(12,2)` | `12.50` | Human-readable in SQL; database does the math | Go has no built-in decimal type (use a string or a library like `shopspring/decimal`) |
| **Integer minor units** | `BIGINT` (cents) | `1250` | Simple, fast, exact in Go *and* SQL; what Stripe's API uses | Must convert for display; currency needs its own column |

(PostgreSQL also has a `MONEY` type. Avoid it: its behavior depends on a server locale setting, and it can't hold different currencies sensibly.)

**For this course:** the API already exposes `price` as a JSON number, and Go's `float64` is the model field. We store it in **`NUMERIC(12,2)`** so the *database* is exact, and we treat the Go float as a transport format for two-decimal values. It's a pragmatic teaching compromise; **in a real payment system, use integer cents (or a decimal library) in Go too**. Exercise 5 has you do exactly that.

---

## 7. Text: `TEXT`, `VARCHAR`, `CHAR`

| Type | Meaning | Go type |
|------|---------|---------|
| `TEXT` | variable length, no limit (up to ~1 GB) | `string` |
| `VARCHAR(n)` | variable length, **at most n characters** | `string` |
| `CHAR(n)` | **fixed** length, padded with spaces | `string` |

In PostgreSQL, `TEXT` and `VARCHAR` have **identical performance and storage**; `VARCHAR(n)` only adds a length check. Unlike some other databases, there is no speed benefit to picking a "smaller" limit. So:

- **Default to `TEXT`**, adding a `CHECK` constraint if you need a limit (it can be changed later without rewriting the table, unlike shrinking a `VARCHAR`).
- `VARCHAR(n)` is fine when the limit is a true business rule (a 2-letter country code: `CHAR(2)` or `VARCHAR(2)`).
- **Avoid `CHAR(n)`.** Its blank-padding causes surprises:

```sql
SELECT 'ab'::char(5) AS char_padded, length('ab'::char(5)) AS char_len, octet_length('ab'::char(5)) AS char_bytes,
       'ab'::varchar(5) AS varchar_val, length('ab'::varchar(5)) AS varchar_len;
```

```
 char_padded | char_len | char_bytes | varchar_val | varchar_len
-------------+----------+------------+-------------+-------------
 ab          |        2 |          5 | ab          |           2
```

The `CHAR(5)` value physically occupies 5 bytes (padded), but PostgreSQL pretends the padding isn't there when comparing, which makes behavior hard to reason about.

### A limit violation is an error… on insert

```sql
CREATE TEMP TABLE t (code varchar(5), fixed char(3), price numeric(10,2), qty smallint);
INSERT INTO t VALUES ('abcdef', 'x', 1, 1);
```

```
ERROR:  value too long for type character varying(5)
```

(An *explicit cast*, `'toolongvalue'::varchar(5)`, on the other hand **silently truncates** to `toolo`. Casting is "I know what I'm doing"; inserting is validated. Prefer explicit validation in your Go code *and* a constraint.)

More failed inserts from the same table:

```sql
INSERT INTO t VALUES ('abc', 'x', 1.005, 40000);   -- qty doesn't fit a smallint
INSERT INTO t VALUES ('abc', 'x', 'free', 4);      -- not a number
INSERT INTO t VALUES ('abc', 'x', 1.005, 4);       -- fine: 1.005 is rounded to 1.01
```

```
ERROR:  smallint out of range
ERROR:  invalid input syntax for type numeric: "free"
INSERT 0 1
```

### Characters vs. bytes (UTF-8)

PostgreSQL text is (normally) UTF-8, so one *character* can be several *bytes*, exactly as with Go's `len(string)` vs. `utf8.RuneCountInString` (Chapter 33):

```sql
SELECT length('বাংলা') AS chars, octet_length('বাংলা') AS bytes,
       length('é') AS e_chars, octet_length('é') AS e_bytes;
```

```
 chars | bytes | e_chars | e_bytes
-------+-------+---------+---------
     5 |    15 |       1 |       2
```

SQL's `length`/`char_length` count **characters**; `octet_length` counts **bytes**. Go's `len(s)` matches `octet_length`, not `length`, so validating a title's length in Go with `len()` and in SQL with `char_length()` can disagree for non-ASCII text. Use `utf8.RuneCountInString` in Go when the rule is "characters".

### Everyday text operations

```sql
SELECT upper('go'), 'Go' = 'go' AS equal_ci, lower('Go') = lower('GO') AS equal_lower,
       'Go' || ' ' || 'lang' AS concat, substring('postgres' from 1 for 4) AS sub,
       'abc' LIKE 'a%' AS starts_a, 'ABC' ILIKE 'a%' AS ilike_a;
```

```
 upper | equal_ci | equal_lower | concat  | sub  | starts_a | ilike_a
-------+----------+-------------+---------+------+----------+---------
 GO    | f        | t           | Go lang | post | t        | t
```

- `=` on text is **case-sensitive** by default; compare `lower(a) = lower(b)` (as our users table does) or use the `citext` extension.
- `||` concatenates. `LIKE` matches patterns (`%` = any run of characters, `_` = exactly one); `ILIKE` ignores case.
- ⚠️ `x LIKE '%foo%'` cannot use an ordinary B-tree index; for serious text search look at `pg_trgm` or full-text search.

---

## 8. Boolean and `NULL`

`BOOLEAN` stores `true`, `false`, or **`NULL`** (unknown). Go type: `bool`.

```sql
SELECT 'yes'::boolean, 't'::boolean, '0'::boolean, 'off'::boolean;
```

```
 bool | bool | bool | bool
------+------+------+------
 t    | t    | f    | f
```

Accepted spellings: `true/false`, `t/f`, `yes/no`, `y/n`, `on/off`, `1/0`.

### `NULL` means "unknown", not "zero" or "empty"

`NULL` is the most misunderstood concept for newcomers. It represents the *absence of a value*, and it makes SQL use **three-valued logic** (true / false / unknown):

```sql
SELECT true AND NULL AS and_null, false AND NULL AS false_and_null, true OR NULL AS or_null,
       NULL = NULL AS null_eq_null, NULL IS NULL AS is_null, NOT NULL::boolean AS not_null;
SELECT COALESCE(NULL, NULL, 'fallback') AS coalesced, NULLIF('', '') AS nullif_empty, 5 + NULL AS five_plus_null;
```

```
 and_null | false_and_null | or_null | null_eq_null | is_null | not_null
----------+----------------+---------+--------------+---------+----------
          | f              | t       |              | t       |

 coalesced | nullif_empty | five_plus_null
-----------+--------------+----------------
 fallback  |              |
```

(Empty cells are `NULL`.) The rules:

- **Any arithmetic or comparison with `NULL` yields `NULL`**: `5 + NULL` is `NULL`; `NULL = NULL` is *not true*: it's `NULL`.
- Test with **`IS NULL` / `IS NOT NULL`**, never `= NULL`. A `WHERE price = NULL` matches **nothing**, ever.
- `false AND NULL` is `false` (the answer is false whatever the unknown is); `true AND NULL` is `NULL`.
- **`COALESCE(a, b, c)`** returns the first non-NULL; **`NULLIF(a, b)`** returns `NULL` if `a = b` (handy for turning `''` into `NULL`).
- `WHERE` keeps only rows where the condition is **true**; rows where it's `NULL` are dropped. That's a classic source of "missing rows" bugs.

### `NULL` in Go

Go has no `NULL`; a `string` can't be "absent". Scanning a `NULL` into a plain `string` or `int` **fails**:

```go
var s string
err := db.QueryRowContext(ctx, `SELECT NULL::text`).Scan(&s)
```

```
sql: Scan error on column index 0, name "text": converting NULL to string is unsupported
```

Three ways to handle nullable columns:

| Approach | Example | Notes |
|----------|---------|-------|
| **Avoid NULL** (preferred): `NOT NULL DEFAULT ''` / `DEFAULT 0` | `description TEXT NOT NULL DEFAULT ''` | Simplest Go code; use when "empty" and "unknown" mean the same |
| **Pointer** | `var note *string` | `nil` means NULL; JSON marshals it as `null` |
| **`sql.NullString`, `sql.NullInt64`, …** | `var n sql.NullString; n.Valid` | Explicit `Valid` flag; awkward in JSON |

Rule of thumb: **make columns `NOT NULL` unless "unknown/not applicable" is a genuinely different state from any real value.** (A `deleted_at TIMESTAMPTZ` that is `NULL` for live rows is the classic legitimate use.)

---

## 9. Dates and times

| Type | Stores | Example | Go type |
|------|--------|---------|---------|
| `DATE` | a calendar date | `2026-09-26` | `time.Time` (midnight) |
| `TIME` | a time of day | `15:45:10` | `string`/custom |
| `TIMESTAMP` | date + time, **no time zone** | `2026-09-26 12:00:00` | `time.Time` |
| **`TIMESTAMPTZ`** | an **absolute instant** | `2026-09-26 06:00:00+00` | `time.Time` |
| `INTERVAL` | a duration | `3 days 06:00:00` | `time.Duration` (convert) / string |

### `TIMESTAMPTZ` vs. `TIMESTAMP`: the rule

> **Always use `TIMESTAMPTZ` for moments in time.**

Despite its name, `TIMESTAMPTZ` does **not** store a zone. It stores an absolute instant (internally UTC) and *displays* it in the session's time zone. Plain `TIMESTAMP` stores just a wall-clock reading with **no idea** which zone it meant, so the same value means different instants for different readers. Compare the same two literals under three session time zones:

```sql
SET TIME ZONE 'UTC';
SELECT '2026-09-26 12:00:00+06'::timestamptz AS tstz, '2026-09-26 12:00:00'::timestamp AS ts_naive;
SET TIME ZONE 'Asia/Dhaka';
SELECT '2026-09-26 12:00:00+06'::timestamptz AS tstz, '2026-09-26 12:00:00'::timestamp AS ts_naive;
SET TIME ZONE 'America/New_York';
SELECT '2026-09-26 12:00:00+06'::timestamptz AS tstz, '2026-09-26 12:00:00'::timestamp AS ts_naive;
```

```
          tstz          |       ts_naive
------------------------+---------------------
 2026-09-26 06:00:00+00 | 2026-09-26 12:00:00      ← session zone: UTC

          tstz          |       ts_naive
------------------------+---------------------
 2026-09-26 12:00:00+06 | 2026-09-26 12:00:00      ← session zone: Asia/Dhaka

          tstz          |       ts_naive
------------------------+---------------------
 2026-09-26 02:00:00-04 | 2026-09-26 12:00:00      ← session zone: America/New_York
```

The `timestamptz` is *the same instant* shown three ways (06:00 UTC = 12:00 in Dhaka = 02:00 in New York). The naive `timestamp` shows `12:00:00` everywhere: **is that 12:00 in Dhaka or in New York?** Nobody knows. Use `TIMESTAMP` only for deliberately zone-less values ("the store opens at 09:00 local time").

### Working with Go

The driver returns a `time.Time` for `TIMESTAMPTZ`. Its `Location` may be your machine's local zone, so in our stores we call **`.UTC()`** (Chapter 53) to make every environment behave identically. JSON output (`time.Time` → RFC 3339, e.g. `2026-09-26T06:37:24.922112Z`) is unambiguous.

### Date arithmetic

```sql
SET TIME ZONE 'UTC';
SELECT DATE '2026-09-26' + 30 AS plus_30_days,
       TIMESTAMPTZ '2026-01-31 00:00:00+00' + INTERVAL '1 month' AS jan31_plus_month,
       INTERVAL '1 day 2 hours' * 3 AS interval_mult;
SELECT age(DATE '2026-09-26', DATE '2000-02-29') AS age_years,
       DATE '2026-09-26' - DATE '2026-01-01' AS days_between,
       EXTRACT(dow FROM DATE '2026-09-26') AS dow,
       date_trunc('month', TIMESTAMPTZ '2026-09-26 15:45:10+00') AS month_start,
       to_char(TIMESTAMPTZ '2026-09-26 15:45:10+00', 'DD Mon YYYY HH24:MI') AS formatted;
```

```
 plus_30_days |    jan31_plus_month    |  interval_mult
--------------+------------------------+-----------------
 2026-10-26   | 2026-02-28 00:00:00+00 | 3 days 06:00:00

        age_years        | days_between | dow |      month_start       |     formatted
-------------------------+--------------+-----+------------------------+-------------------
 26 years 6 mons 26 days |          268 |   6 | 2026-09-01 00:00:00+00 | 26 Sep 2026 15:45
```

- `date + integer` adds **days**; `date - date` gives a day count; `+ INTERVAL '1 month'` is calendar-aware (Jan 31 + 1 month = **Feb 28**, clamped).
- `EXTRACT(dow …)` gives the day of week (0 = Sunday, so `6` = Saturday); `date_trunc('month', …)` snaps to the start of the month, ideal for "sales per month" reports; `to_char` formats.
- Invalid dates are rejected:

```sql
SELECT '2026-02-30'::date;
```
```
ERROR:  date/time field value out of range: "2026-02-30"
```

### `now()` and defaults

`now()` returns the **start time of the current transaction** (the same value for every use within a transaction), which is what you want for `created_at DEFAULT now()`. `clock_timestamp()` is the actual wall clock, changing during a statement.

---

## 10. UUID, JSONB, arrays, enums, bytea

### `UUID`

A 128-bit identifier such as `3b241101-e2bb-4255-8caf-4136c566a962`. Stored in 16 bytes, generated by `gen_random_uuid()` (built in since PostgreSQL 13):

```sql
SELECT length(gen_random_uuid()::text) AS uuid_text_len, pg_column_size(gen_random_uuid()) AS uuid_bytes;
```
```
 uuid_text_len | uuid_bytes
---------------+------------
            36 |         16
```

Use UUIDs when IDs are created **outside** the database (clients, several services, offline apps) or must be **unguessable** (`/orders/1042` invites enumeration; a random UUID doesn't). Costs: twice the size of a `BIGINT`, poorer index locality (random inserts). Go: the driver returns them as `string` (or use `github.com/google/uuid`).

### `JSONB`

Stores JSON documents in a **binary, indexable** format. Use `JSONB`, not `JSON` (which stores the raw text, keeps duplicate keys and whitespace, and can't be indexed well).

```sql
SELECT '{"name":"Orange","tags":["fruit","citrus"],"price":100}'::jsonb AS doc;
SELECT '{"name":"Orange","tags":["fruit","citrus"]}'::jsonb -> 'name'  AS json_val,
       '{"name":"Orange","tags":["fruit","citrus"]}'::jsonb ->> 'name' AS text_val,
       '{"a":1,"a":2}'::json  AS json_keeps,
       '{"a":1,"a":2}'::jsonb AS jsonb_dedups,
       '{"tags":["x","y"]}'::jsonb @> '{"tags":["x"]}' AS contains;
```

```
                              doc
---------------------------------------------------------------
 {"name": "Orange", "tags": ["fruit", "citrus"], "price": 100}

 json_val | text_val |  json_keeps   | jsonb_dedups | contains
----------+----------+---------------+--------------+----------
 "Orange" | Orange   | {"a":1,"a":2} | {"a": 2}     | t
```

- `->` returns JSON (note the quotes around `"Orange"`), `->>` returns **text**.
- `@>` means "contains"; with a GIN index it makes "find products tagged citrus" fast.
- `JSONB` is great for **genuinely flexible** data (per-product attributes that differ by category, webhook payloads). **It is not a substitute for columns**: you lose type checks, constraints, and foreign keys. If every row has a `price`, make it a `price` column.
- Go: scan into `[]byte`/`json.RawMessage`, then `json.Unmarshal`.

### Arrays

Any type can be an array: `TEXT[]`, `INTEGER[]`. **Indexing starts at 1**, unlike Go:

```sql
SELECT ARRAY['fruit','citrus'] AS tags, (ARRAY['a','b','c'])[1] AS first_is_one,
       'citrus' = ANY(ARRAY['fruit','citrus']) AS has_citrus, array_length(ARRAY[1,2,3], 1) AS len;
```
```
      tags      | first_is_one | has_citrus | len
----------------+--------------+------------+-----
 {fruit,citrus} | a            | t          |   3
```

Convenient for small, simple lists that are always read and written as a whole. If you need to query or link the items independently (tags with their own pages, counts), use a **separate table** instead (Chapter 59+). In Go, use the driver's array helpers (for `pgx`, `pgtype.FlatArray[string]`) rather than parsing `{a,b}` yourself.

### Enums

```sql
CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');
SELECT 'happy'::mood, 'sad'::mood < 'happy'::mood AS ordered;
SELECT 'meh'::mood;
```
```
 mood  | ordered
-------+---------
 happy | t

ERROR:  invalid input value for enum mood: "meh"
```

Enums give you a closed set of allowed values, stored compactly, ordered by declaration. The catch: **removing or reordering values is hard**. A common alternative is `TEXT` with a `CHECK (status IN ('pending','paid','shipped'))` constraint, easier to evolve with a migration. Either way, mirror the set as **constants** in Go.

### `BYTEA`

Raw bytes (`\xDEADBEEF`), Go `[]byte`. Fine for small binary values (hashes, tokens); for large files (images), store them in object storage (S3) and keep only the URL in the database, as our `image_url` column does.

---

## 11. How much space do types take?

`pg_column_size(x)` reports bytes of a value:

```sql
SELECT pg_column_size(1::smallint) AS smallint_b, pg_column_size(1::int) AS int_b,
       pg_column_size(1::bigint) AS bigint_b, pg_column_size(1::float8) AS float8_b,
       pg_column_size(123456.78::numeric) AS numeric_b, pg_column_size(true) AS bool_b,
       pg_column_size(now()) AS timestamptz_b, pg_column_size(current_date) AS date_b,
       pg_column_size('hello'::text) AS text5_b, pg_column_size(repeat('x', 100)) AS text100_b,
       pg_column_size(gen_random_uuid()) AS uuid_b;
```

```
 smallint_b | int_b | bigint_b | float8_b | numeric_b | bool_b | timestamptz_b | date_b | text5_b | text100_b | uuid_b
------------+-------+----------+----------+-----------+--------+---------------+--------+---------+-----------+--------
          2 |     4 |        8 |        8 |        12 |      1 |             8 |      4 |       9 |       104 |     16
```

Notes:

- Variable-length values (`text`, `numeric`) carry a small **header** (4 bytes here: `'hello'` = 5 + 4 = 9).
- Very large values are compressed and moved out of line automatically (**TOAST**), so a 1 MB text column doesn't bloat scans of other columns.
- On disk, rows are also *aligned*: mixing 2-byte and 8-byte columns can waste padding. Ordering columns from widest to narrowest saves space at very large scale; measure before caring.
- **Bytes multiply**: 10 million rows × 8 bytes for an unneeded `BIGINT` instead of `SMALLINT` is 60 MB; the index copies it again.

---

## 12. PostgreSQL types ↔ Go types

What the `pgx` driver hands to Go (a real run, scanning each column into `any`):

```
smallint     -> int64
int          -> int64
bigint       -> int64
float8       -> float64
numeric      -> string
text         -> string
boolean      -> bool
timestamptz  -> time.Time
date         -> time.Time
interval     -> string
uuid         -> string
jsonb        -> []uint8
bytea        -> []uint8
NULL         -> <nil>
```

And what you should **scan into** in your structs:

| PostgreSQL | Recommended Go field | Notes |
|------------|---------------------|-------|
| `SMALLINT` / `INTEGER` / `BIGINT` | `int16` / `int32` / `int64` (or `int`) | overflow when scanning is an error, not a wrap |
| `NUMERIC(p,s)` | `string`, `float64`, or a decimal type | scanning into `float64` works (reads `12.50` → `12.5`) but reintroduces float rounding; use a decimal library or integer cents for real money |
| `REAL` / `DOUBLE PRECISION` | `float32` / `float64` | |
| `TEXT` / `VARCHAR` | `string` | `NULL` needs `*string` / `sql.NullString` |
| `BOOLEAN` | `bool` | |
| `TIMESTAMPTZ` / `DATE` | `time.Time` | call `.UTC()`; zero `time.Time` is year 1, not NULL |
| `INTERVAL` | `string` or a custom type | rarely needed in application code |
| `UUID` | `string` / `uuid.UUID` | |
| `JSONB` | `json.RawMessage` / `[]byte` | then `json.Unmarshal` |
| `BYTEA` | `[]byte` | |
| `TEXT[]` etc. | `pgtype.FlatArray[string]` / `[]string` via helper | driver-specific |
| any nullable column | pointer or `sql.Null*` | or make it `NOT NULL` |

Scanning errors are explicit; the driver won't guess:

```
text into int: sql: Scan error on column index 0, name "text": converting driver.Value type string ("abc") to a int: invalid syntax
```

And a successful real scan of the "types you'd really use", showing the timezone behavior:

```go
var price float64                // numeric(10,2) 12.50
var created time.Time            // timestamptz '2026-09-26 12:00:00+06'
...
fmt.Println(price)               // 12.5
fmt.Println(created.UTC())       // 2026-09-26 06:00:00 +0000 UTC
```

---

## 13. Choosing: a decision guide

| The column holds… | Use | Avoid |
|-------------------|-----|-------|
| A primary key | `BIGINT GENERATED ALWAYS AS IDENTITY` (or `UUID`) | `INTEGER` for growing tables |
| A count, quantity, small whole number | `INTEGER` | `NUMERIC` for whole numbers |
| Money | `NUMERIC(12,2)` or `BIGINT` cents | `REAL`, `DOUBLE PRECISION`, `MONEY` |
| A measurement (weight, temperature, score) | `DOUBLE PRECISION` | `NUMERIC` if speed matters and exactness doesn't |
| A name, title, description, URL | `TEXT` (+ `CHECK` for limits) | `CHAR(n)` |
| Yes/no flag | `BOOLEAN NOT NULL DEFAULT false` | `INTEGER` 0/1, `'Y'/'N'` |
| A moment in time | `TIMESTAMPTZ` | `TIMESTAMP`, or strings/integers |
| A calendar date (birthday) | `DATE` | `TIMESTAMPTZ` |
| A fixed set of statuses | `TEXT` + `CHECK`, or `ENUM` | magic numbers |
| Flexible, schema-less attributes | `JSONB` | `JSON`, or JSON for data that should be columns |
| Small list read/written as a whole | array | arrays for data you query relationally |
| Externally generated/unguessable ID | `UUID` | sequential IDs in public URLs |
| Files / images | object storage + URL `TEXT` | `BYTEA` for big blobs |

General principles:

1. **Make columns `NOT NULL`** unless "unknown" is meaningful.
2. **Use the most specific type** that matches the meaning (dates as dates, not strings).
3. **Put rules in the schema** (`CHECK`, `UNIQUE`, `NOT NULL`, foreign keys): it's the last line of defense.
4. **Prefer exactness** (`NUMERIC`, integers) over speed until profiling says otherwise.
5. **Default to bigger** for IDs and **smaller** for high-volume lookup values.

---

## 14. Designing the `products` table

Putting the rules together for the shop's products (Chapter 56 turns this into a real migrated table and store):

```sql
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

Every choice, explained:

| Column | Type | Why |
|--------|------|-----|
| `id` | `BIGINT … IDENTITY` | never runs out; database-assigned; matches Go's `int` on 64-bit |
| `title` | `TEXT` + `CHECK` on **characters** (`char_length`, trimmed) | same 1–100 rule as the Go validation (Chapter 51), enforced even if Go is bypassed; counts characters, not bytes |
| `description` | `TEXT NOT NULL DEFAULT ''` | "no description" is just empty; avoids `NULL` handling in Go |
| `price` | `NUMERIC(12,2)` + `CHECK (price > 0)` | exact to the cent, up to 10 billion; the rule "price must be > 0" is also in Go |
| `image_url` | `TEXT NOT NULL DEFAULT ''` | a URL; the image itself lives in object storage |
| `created_at` / `updated_at` | `TIMESTAMPTZ NOT NULL DEFAULT now()` | audit trail with an unambiguous instant (`updated_at` is set by our update query) |

Test the constraints yourself (in a scratch `TEMP TABLE`, or after Chapter 56's real migration):

```sql
INSERT INTO products (title, price) VALUES ('Orange', 100.00);      -- ok
INSERT INTO products (title, price) VALUES ('  ', 5);               -- blank title
INSERT INTO products (title, price) VALUES ('Free lunch', 0);       -- price must be > 0
INSERT INTO products (title, price) VALUES ('Tiny', 0.005);         -- rounds to 0.01
INSERT INTO products (title, price) VALUES ('Tinier', 0.004);       -- rounds to 0.00
SELECT id, title, price FROM products;
```

Real output:

```
INSERT 0 1
ERROR:  new row for relation "products" violates check constraint "products_title_check"
ERROR:  new row for relation "products" violates check constraint "products_price_check"
INSERT 0 1
ERROR:  new row for relation "products" violates check constraint "products_price_check"

 id | title  | price
----+--------+--------
  1 | Orange | 100.00
  4 | Tiny   |   0.01
```

Two lessons hide in that output:

1. **`0.005` was accepted, `0.004` rejected.** The value is rounded to the column's scale *first*, then the `CHECK` runs on the rounded number (`0.01 > 0` passes, `0.00 > 0` fails). If a half-cent price is a bug for you, validate in Go *before* the database sees it.
2. **The IDs jump from 1 to 4.** The three failed inserts each *consumed* an identity number even though no row was stored. Sequences never roll back. That's why IDs are never gap-free (§3).

---

## 15. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Money in `FLOAT`/`REAL` | Rounding drift; failed audits | `NUMERIC(p,2)` or integer cents |
| 2 | `INTEGER` primary key on a busy table | ID exhaustion at 2.1 billion | `BIGINT` |
| 3 | `TIMESTAMP` for real moments | Ambiguous times across zones/servers | `TIMESTAMPTZ` |
| 4 | `WHERE col = NULL` | Matches nothing | `IS NULL` |
| 5 | Scanning a nullable column into `string`/`int` | Runtime `converting NULL … unsupported` | `NOT NULL` column, pointer, or `sql.Null*` |
| 6 | `CHAR(n)` | Padding surprises | `TEXT` / `VARCHAR(n)` |
| 7 | `len(s)` in Go vs `char_length` in SQL | Different limits for non-ASCII text | `utf8.RuneCountInString` for character limits |
| 8 | `JSON` instead of `JSONB`; JSON where columns belong | No indexing / no constraints | `JSONB` for truly flexible data only |
| 9 | Storing files in `BYTEA` | Huge tables, slow backups | Object storage + URL |
| 10 | Assuming sequential, gap-free IDs | Broken logic | Treat IDs as opaque |
| 11 | Case-sensitive comparisons by accident | "Asha" ≠ "asha" | `lower()` on both sides, or `citext` |
| 12 | Comparing floats with `=` | Random `false` | Compare with a tolerance, or use `NUMERIC` |
| 13 | Relying on Go to enforce all rules | Bad data via other tools or bugs | Constraints in the schema too |
| 14 | Storing dates/numbers as strings | No validation, wrong sort order (`'10' < '9'`) | Real types |

---

## 16. Interview questions

**Q1. Why not use `FLOAT` for prices?**
Binary floating-point can't represent most decimal fractions exactly (`0.1 + 0.2 ≠ 0.3`), and errors accumulate. Use `NUMERIC` or integer minor units.

**Q2. `TIMESTAMP` vs `TIMESTAMPTZ`?**
`TIMESTAMPTZ` is an absolute instant (stored UTC, displayed in the session zone); `TIMESTAMP` is a zone-less wall-clock value. Use `TIMESTAMPTZ` for events.

**Q3. `TEXT` vs `VARCHAR(n)`?**
Same performance and storage in PostgreSQL; `VARCHAR(n)` adds a length check. Prefer `TEXT` (+ `CHECK` if needed).

**Q4. What does `NULL = NULL` return?**
`NULL` (unknown), not `true`. Use `IS NULL`. Comparisons and arithmetic with `NULL` yield `NULL`.

**Q5. When would you choose `UUID` over `BIGINT` for a key?**
When IDs are generated outside the database, must be unguessable, or must be unique across systems; accept the size/index-locality cost.

**Q6. `JSON` vs `JSONB`?**
`JSONB` is parsed into a binary form: indexable, deduplicated keys, faster operators; `JSON` keeps the text verbatim. Prefer `JSONB`.

**Q7. How do you handle a nullable column in Go?**
Prefer `NOT NULL` with a default; otherwise scan into a pointer or `sql.NullXxx`.

**Q8. What happens on integer overflow in PostgreSQL?**
An error (`integer out of range`), not a silent wrap.

**Q9. How is `NUMERIC(5,2)` different from `NUMERIC(5,0)`?**
Precision 5 = five significant digits total; scale 2 vs. 0 = digits after the point. `NUMERIC(5,2)` maxes at `999.99`; `NUMERIC(5,0)` at `99999`.

---

## 17. Exercises

### Exercise 1: Predict, then run
Predict each result, then check in `psql`:
`SELECT 10 / 4, 10 / 4.0, 10 % 4, 2.675::numeric(5,2), NULL IS NULL, NULL = 0;`

<details><summary>Solution</summary>

`2`, `2.5000000000000000`, `2`, `2.68` (numeric rounds half up), `true`, `NULL`.
</details>

### Exercise 2: Fix the schema
What's wrong here, and how would you fix each column?

```sql
CREATE TABLE orders (
  id INTEGER PRIMARY KEY,
  total FLOAT,
  placed TIMESTAMP,
  status VARCHAR(255),
  paid CHAR(1)
);
```

<details><summary>Solution</summary>

`id` → `BIGINT GENERATED ALWAYS AS IDENTITY`; `total` → `NUMERIC(12,2) NOT NULL CHECK (total >= 0)` (money in float!); `placed` → `TIMESTAMPTZ NOT NULL DEFAULT now()`; `status` → `TEXT NOT NULL CHECK (status IN ('pending','paid','shipped','cancelled'))` (or an enum); `paid` → `BOOLEAN NOT NULL DEFAULT false` (no `'Y'/'N'` chars). Also `NOT NULL` where missing.
</details>

### Exercise 3: Time zones
Insert `'2026-12-31 23:30:00-05'` into a `timestamptz` column. What does a client in `Asia/Dhaka` see? What if the column were `timestamp`?

<details><summary>Solution</summary>

As `timestamptz` it's the instant 2027-01-01 04:30 UTC, shown to a Dhaka client as `2027-01-01 10:30:00+06` (New Year already arrived!). As plain `timestamp` PostgreSQL silently **drops** the `-05` offset and stores `2026-12-31 23:30:00`: information lost.
</details>

### Exercise 4: NULL logic
`products` has 10 rows, 3 with `discount IS NULL`, 4 with `discount = 0`, 3 with `discount > 0`. How many rows do `WHERE discount = 0`, `WHERE discount <> 0`, and `WHERE NOT (discount = 0)` return? How would you get all rows that have *no discount*?

<details><summary>Solution</summary>

`= 0` → 4; `<> 0` → 3; `NOT (discount = 0)` → 3 (the NULL rows are neither equal nor unequal, so they vanish from both). "No discount" = `WHERE discount IS NULL OR discount = 0`, or `WHERE COALESCE(discount, 0) = 0`.
</details>

### Exercise 5: Cents in Go
Change the design so prices are stored as `price_cents BIGINT`. Sketch the Go model, the conversion to/from the JSON `price` (a decimal number like `12.5`), and explain where rounding must happen.

<details><summary>Solution</summary>

```go
type Product struct {
	ID         int   `json:"-"          db:"id"`
	PriceCents int64 `json:"-"          db:"price_cents"`
}

func (p Product) MarshalJSON() ([]byte, error) { /* emit price as float64(p.PriceCents)/100 */ }
```
On input, parse the JSON price into a string or `json.Number` and convert to cents with **integer/decimal parsing** (not `float64*100`, which yields `1249.9999…`); reject more than two decimals. Rounding happens **once, at the boundary**, and all internal math is integer. Alternatively use `shopspring/decimal`.
</details>

### Exercise 6: Size it
A table has 50 million rows, each with an `INTEGER` id, a `BIGINT` account id, a `SMALLINT` status, and a `BOOLEAN`. Estimate the payload bytes per row and total, then repeat if `status` were `BIGINT`.

<details><summary>Solution</summary>

4 + 8 + 2 + 1 = 15 bytes payload (real rows add a ~24-byte header and alignment padding, so measure with `pg_column_size`/`pg_relation_size`). × 50M ≈ 750 MB payload. With `BIGINT` status: 4 + 8 + 8 + 1 = 21 bytes → ~1.05 GB; +300 MB, and the same extra in every index containing that column.
</details>

---

## 18. Quiz

1. Which type for money, and why not `float`?
2. What's the difference between `length()` and `octet_length()`?
3. Why is `WHERE x = NULL` a bug?
4. What does `TIMESTAMPTZ` actually store?
5. What does PostgreSQL do when an `INTEGER` overflows?
6. Which JSON type should you use and why?
7. What's wrong with `CHAR(n)`?
8. Why prefer `BIGINT` for primary keys?

<details><summary>Answers</summary>

1. `NUMERIC(p,2)` or integer cents; floats are inexact for decimal fractions.
2. Characters vs. bytes (they differ for non-ASCII text).
3. Comparison with `NULL` is never true; use `IS NULL`.
4. An absolute instant (UTC internally); the zone is applied only for display.
5. Raises `integer out of range`.
6. `JSONB`: binary, indexable, deduplicated keys, more operators.
7. Blank padding makes lengths/comparisons confusing, with no performance benefit.
8. So the table can never exhaust its ID space.
</details>

---

## 19. Summary

- **Types are enforced promises**: they validate, save space, and decide which operations and indexes work.
- **Integers**: `INTEGER` by default, **`BIGINT` for keys**; overflow is an *error*.
- **Money = `NUMERIC(p,2)` or integer cents**, never floats (`0.1 + 0.2 = 0.30000000000000004`; ten `0.1`s sum to `0.9999999999999999`).
- **Text**: prefer `TEXT` (+ `CHECK`); `VARCHAR(n)` costs the same; avoid `CHAR(n)`; remember characters vs. bytes and case-sensitive comparisons.
- **`NULL` = unknown**: three-valued logic, `IS NULL`, `COALESCE`; in Go use `NOT NULL` columns, pointers, or `sql.Null*`.
- **Time**: `TIMESTAMPTZ` for instants (a zone-less `TIMESTAMP` is ambiguous); use `INTERVAL` and `date_trunc` for calendar math; `.UTC()` in Go.
- **`UUID`, `JSONB`, arrays, enums, `BYTEA`** each have a place, but columns beat JSON when the shape is known.
- The `products` table: `BIGINT` identity, checked `TEXT` title, `NUMERIC(12,2)` price, `TIMESTAMPTZ` audit columns: every constraint mirrors a Go validation rule.

### ➡️ What's next?

[Chapter 55](55-sql-crud.md) is **SQL CRUD**: `SELECT`, `INSERT`, `UPDATE`, `DELETE` in depth (filtering, sorting, limiting, aggregates, `RETURNING`, transactions preview), so that Chapter 56 can turn them into Go.
