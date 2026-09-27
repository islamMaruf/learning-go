# Chapter 2: Variables and Data Types

> **Goal of this chapter:** Learn how a program *remembers* information. You will meet variables, Go's basic data types, constants, and the different ways to declare them.

**Difficulty:** 🟢 Beginner  **Estimated time:** 1–1.5 hours  **Prerequisite:** [Chapter 1](01-your-first-go-program.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What is a variable?](#2-what-is-a-variable)
3. [How variables live in memory](#3-how-variables-live-in-memory)
4. [Data types](#4-data-types)
5. [Declaring variables (four ways)](#5-declaring-variables-four-ways)
6. [Zero values](#6-zero-values)
7. [Reassigning and the "loyal variable" rule](#7-reassigning-variables)
8. [Multiple variables at once](#8-multiple-variables-at-once)
9. [Constants](#9-constants)
10. [Type conversion](#10-type-conversion)
11. [Naming rules and conventions](#11-naming-rules-and-conventions)
12. [Unused variables](#12-unused-variables)
13. [Common mistakes](#13-common-mistakes)
14. [Exercises](#14-exercises)
15. [Quiz](#15-quiz)
16. [Summary](#16-summary)

---

## 1. What you will learn

- What a variable is and why programs need them
- What happens in memory when you create one
- The core types: `int`, `float64`, `bool`, `string`
- Four ways to declare a variable, and which one to prefer
- What *zero values* are
- How `const` differs from `var`
- How to convert between types
- How to name things the "Go way"

---

## 2. What is a variable?

Programs work with **data**: a user's name, the price of a product, whether someone is logged in. A **variable** is a **named container** that holds one piece of data so you can use it, change it, and print it later.

```
Real world                     Programming
──────────────────             ──────────────────────
A labeled jar of coffee   →    A variable called `coffee`
The label                 →    The variable's *name*
What's inside the jar     →    The variable's *value*
"Only fits liquids"       →    The variable's *type*
```

In Go, every variable has three things:

| Part | Example | Meaning |
|------|---------|---------|
| **Name** | `age` | How you refer to it |
| **Type** | `int` | What kind of data it can hold |
| **Value** | `25` | What it holds right now |

---

## 3. How variables live in memory

Your computer has **RAM** (Random Access Memory), a huge row of tiny numbered storage slots. When your program runs, variables live in RAM.

```
RAM (simplified)
address:  1000   1008   1016   1024
        ┌──────┬──────┬──────┬──────┐
        │      │      │      │      │   ← empty slots
        └──────┴──────┴──────┴──────┘
```

When you write:

```go
var age int = 10
```

Go does three things:
1. **Reserves** a slot big enough for an `int`
2. **Labels** that slot `age` (you use the name; the computer uses the address)
3. **Stores** `10` in it

```
        ┌──────┬──────┬──────┬──────┐
        │ age  │      │      │      │
        │  10  │      │      │      │
        └──────┴──────┴──────┴──────┘
```

Changing the value **overwrites** the old one:

```go
age = 20   // 10 is gone, slot now holds 20
age = 50   // 20 is gone, slot now holds 50
```

> **Analogy:** A variable is a room that fits one person. Put Alice in. Later, remove Alice and put Bob in. If you look inside, you only find Bob; the room has no memory of Alice.

You'll explore memory much more deeply in [Chapter 18](18-go-internal-memory.md) and [Chapter 24](24-pointers.md).

---

## 4. Data types

A **type** tells Go (1) what kind of data a variable holds and (2) how much memory it needs and what operations make sense (you can add numbers, but "adding" two booleans makes no sense).

```
Basic types
├── Numbers
│   ├── Integers  (whole numbers)   int, int8…int64, uint, uint8…uint64
│   └── Floats    (decimals)        float32, float64
├── Booleans      (true / false)    bool
└── Strings       (text)            string
```

### 4.1 Integers: whole numbers

Examples: `10`, `-5`, `0`, `1005`

| Type | Size | Range |
|------|------|-------|
| `int` | 32 or 64 bit (depends on your CPU; 64 on modern machines) | about ±9.2 × 10¹⁸ on 64-bit |
| `int8` | 8 bit | −128 to 127 |
| `int16` | 16 bit | −32,768 to 32,767 |
| `int32` | 32 bit | about ±2.1 billion |
| `int64` | 64 bit | about ±9.2 × 10¹⁸ |
| `uint` | 32/64 bit | 0 up to twice as large as int's max (**u**nsigned = never negative) |
| `uint8` | 8 bit | 0 to 255 |
| `uint16`, `uint32`, `uint64` | 16/32/64 bit | 0 to 65,535 / ~4.29 billion / ~1.8 × 10¹⁹ |

> ✅ **Beginner rule:** Use plain **`int`** unless you have a specific reason not to. The sized types matter when you care about memory or file/network formats (later chapters).

Also worth knowing: `byte` is another name for `uint8`, and `rune` is another name for `int32` (a single text character, see below).

### 4.2 Floating-point numbers: decimals

Examples: `3.14`, `-0.5`, `99.99`

| Type | Size | Precision |
|------|------|-----------|
| `float32` | 32 bit | about 7 decimal digits |
| `float64` | 64 bit | about 15–16 decimal digits |

> ✅ **Beginner rule:** Use **`float64`**. It's the default type Go picks when you write `x := 3.14`.

> ⚠️ **Floats are approximate.** Computers store decimals in binary, so some numbers can't be represented exactly. If `a := 0.1` and `b := 0.2` are `float64` variables, then `a + b` gives `0.30000000000000004` (and `a+b == 0.3` is `false`). (Literal constants like `0.1 + 0.2` are computed exactly by the compiler, so *that* prints `0.3`.) For money, professionals store **whole cents as integers** (e.g., `1999` for $19.99) or use a decimal library.

### 4.3 Booleans: true or false

Only two possible values: `true` and `false`. Used for yes/no questions (`isLoggedIn`, `hasPermission`). You'll use them heavily in [Chapter 3](03-making-decisions.md).

### 4.4 Strings: text

Text between **double quotes**: `"Hello"`, `"Go is fun"`, even `"100"` (that's text, not a number!).

- Strings are **immutable**: you can't change a character inside one, but you can build a new string.
- `len("hello")` gives `5` (the number of **bytes**).
- Backtick strings (`` `like this` ``) are "raw": backslashes aren't special, and they can span multiple lines.

### 4.5 Bytes and runes (a peek)

Go strings are stored as UTF-8 bytes. A single character can take 1 to 4 bytes.

- A **`byte`** is one byte.
- A **`rune`** is one Unicode character, written with single quotes: `'A'`, `'ব'`, `'😀'`.

```go
package main

import "fmt"

func main() {
	fmt.Println(len("Go"))     // 2
	fmt.Println(len("বাংলা"))   // 15  (5 characters, 3 bytes each)
	var letter rune = 'A'
	fmt.Println(letter)        // 65 (the numeric code of 'A')
	fmt.Println(string(letter)) // A
}
```

Don't worry if that feels advanced; we revisit it when we work with text later.

### 4.6 Type summary

| Category | Type | Example |
|----------|------|---------|
| Integer | `int` | `10`, `-3` |
| Float | `float64` | `3.14`, `99.99` |
| Boolean | `bool` | `true`, `false` |
| String | `string` | `"Hello"` |
| Character | `rune` | `'A'` |

---

## 5. Declaring variables (four ways)

You only *need* one way, but you'll see all four in other people's code.

### Way 1: Full declaration: `var name type = value`

```go
package main

import "fmt"

func main() {
	var x int = 10
	fmt.Println(x) // 10
}
```

| Part | Meaning |
|------|---------|
| `var` | keyword: "declare a variable" |
| `x` | the name you choose |
| `int` | the type |
| `=` | assignment: "store this value" |
| `10` | the value |

### Way 2: Type inference: `var name = value`

Go looks at the value and works out the type for you.

```go
package main

import "fmt"

func main() {
	var a = 10       // int
	var b = 40.34    // float64
	var c = "Hello"  // string
	var d = true     // bool

	fmt.Printf("%v is %T\n", a, a)
	fmt.Printf("%v is %T\n", b, b)
	fmt.Printf("%v is %T\n", c, c)
	fmt.Printf("%v is %T\n", d, d)
}
```

Output:

```
10 is int
40.34 is float64
Hello is string
true is bool
```

> 💡 `%T` prints the type of a value. It's a great tool for learning.

### Way 3: Short declaration: `name := value` ⭐ (most common)

```go
package main

import "fmt"

func main() {
	a := 10
	name := "Asha"
	fmt.Println(a, name)
}
```

The `:=` operator ("walrus", because it looks like eyes and tusks) means **declare and assign** in one step.

Rules:
- ✅ Works **only inside functions**
- ✅ Go infers the type
- ✅ Preferred by most Go programmers for local variables

### Way 4: Declaration without a value: `var name type`

```go
package main

import "fmt"

func main() {
	var count int
	var name string
	var ready bool
	fmt.Println(count, name, ready) // 0  false
}
```

The variable exists and holds its **zero value** (next section). Note that `name` prints as an empty string, so the output looks like `0  false` with two spaces.

### Which one should I use?

| Situation | Best choice |
|-----------|-------------|
| Inside a function, you know the value | `x := 10` |
| Package level (outside any function) | `var x = 10` (`:=` is not allowed there) |
| You want a specific type (e.g., `float64` from an integer literal) | `var x float64 = 10` |
| You want the zero value and set it later | `var x int` |

---

## 6. Zero values

In many languages, an uninitialized variable holds random garbage. **Go never does that.** Every variable starts with a predictable **zero value**:

| Type | Zero value |
|------|-----------|
| `int`, `float64` (all numbers) | `0` |
| `bool` | `false` |
| `string` | `""` (empty string) |
| pointers, slices, maps, functions, interfaces | `nil` (covered in later chapters) |

This makes Go programs safer: there's no "uninitialized memory" surprise.

---

## 7. Reassigning variables

### 7.1 Use `=` to change a value

```go
package main

import "fmt"

func main() {
	a := 10
	fmt.Println("Initially:", a)

	a = 20
	fmt.Println("After a = 20:", a)

	a = 50
	fmt.Println("After a = 50:", a)
}
```

Output:

```
Initially: 10
After a = 20: 20
After a = 50: 50
```

### 7.2 `:=` is only for the *first* time

```go
// INTENTIONAL ERROR: does not compile
package main

func main() {
	a := 10
	a := 20 // ❌ no new variables on left side of :=
}
```

| Symbol | Means | Use when |
|--------|-------|----------|
| `:=` | *declare* a new variable and give it a value | first time |
| `=` | *change* an existing variable | every later time |

### 7.3 The "loyal variable" rule: types never change

Go is **statically typed**: once a variable has a type, it keeps it forever.

```go
// INTENTIONAL ERROR: does not compile
package main

func main() {
	isReady := true
	isReady = "yes" // ❌ cannot use "yes" (string) as bool value
}
```

This feels restrictive at first, but it lets the compiler catch a huge class of bugs before your program runs.

---

## 8. Multiple variables at once

```go
package main

import "fmt"

func main() {
	// Short form: several at once
	name, age, salary := "Habib", 25, 50000.50
	fmt.Println(name, age, salary)

	// Grouped declaration with var ( ... )
	var (
		city    string = "Dhaka"
		year    int    = 2025
		isAlive bool   = true
	)
	fmt.Println(city, year, isAlive)

	// Swapping two values: no temporary variable needed!
	x, y := 1, 2
	x, y = y, x
	fmt.Println(x, y) // 2 1
}
```

Output:

```
Habib 25 50000.5
Dhaka 2025 true
2 1
```

---

## 9. Constants

A **constant** is a named value that **can never change** after it is defined.

```go
package main

import "fmt"

const AppName = "MyShop" // package-level constant

func main() {
	const Pi = 3.14159
	const MaxUsers = 100

	fmt.Println(AppName, Pi, MaxUsers)

	// Pi = 3.14 // ❌ cannot assign to Pi (neither addressable nor a map index expression)
}
```

Output:

```
MyShop 3.14159 100
```

### Why use constants?
- **Safety:** nobody can accidentally change them.
- **Clarity:** `MaxRetries` is more meaningful than the mystery number `3` scattered around ("magic numbers").
- **One place to edit:** change it once, and every use updates.

### Grouped constants and `iota`

`iota` is a counter Go provides for making sequences of related constants (like an "enum"):

```go
package main

import "fmt"

const (
	Sunday = iota // 0
	Monday        // 1
	Tuesday       // 2
)

func main() {
	fmt.Println(Sunday, Monday, Tuesday) // 0 1 2
}
```

### Untyped constants (a Go superpower)
A constant like `const x = 5` has no fixed type until you use it, so it can be used as an `int` *or* a `float64`:

```go
package main

import "fmt"

const x = 5

func main() {
	var i int = x
	var f float64 = x
	fmt.Println(i, f) // 5 5
}
```

Constants must be known **at compile time**, so `const now = time.Now()` is not allowed.

---

## 10. Type conversion

Go **never converts types automatically**. Mixing types is an error:

```go
// INTENTIONAL ERROR: does not compile
package main

import "fmt"

func main() {
	a := 10
	b := 2.5
	fmt.Println(a * b) // ❌ mismatched types int and float64
}
```

You must convert explicitly with `Type(value)`:

```go
package main

import "fmt"

func main() {
	a := 10
	b := 2.5
	fmt.Println(float64(a) * b) // 25

	pi := 3.99
	fmt.Println(int(pi)) // 3  (decimals are cut off, NOT rounded)

	n := 65
	fmt.Println(string(rune(n))) // A
}
```

Converting numbers to and from text needs the `strconv` package:

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	n, err := strconv.Atoi("123") // string → int
	fmt.Println(n+1, err)         // 124 <nil>

	s := strconv.Itoa(456) // int → string
	fmt.Println(s + "!")   // 456!
}
```

> ⚠️ `string(65)` does **not** give `"65"`; it gives the *character* with code 65 (`"A"`). Use `strconv.Itoa` for numbers → text.

---

## 11. Naming rules and conventions

**Rules (enforced by the compiler):**
- Must start with a letter or `_`
- May contain letters, digits, `_`
- Can't be a keyword (`func`, `var`, `if`, `for`, `return`, …)
- Case-sensitive: `age`, `Age`, and `AGE` are three different names

**Conventions (followed by the community):**

| Convention | Example |
|------------|---------|
| Use **camelCase** for multi-word names | `userAge`, `totalPrice` |
| **Never** use `snake_case` | ~~`user_age`~~ |
| Short names in small scopes | `i`, `n`, `err` |
| Descriptive names in larger scopes | `customerEmail` |
| Capital first letter = visible outside the package | `MaxUsers` (exported) vs `maxUsers` (private) |
| Acronyms stay uppercase | `userID`, `HTTPServer` |

---

## 12. Unused variables

Go refuses to compile if you declare a local variable and never use it:

```go
// INTENTIONAL ERROR: does not compile
package main

func main() {
	x := 10 // ❌ declared and not used
}
```

This keeps code clean. If you truly want to ignore a value, assign it to the **blank identifier** `_`:

```go
package main

import "fmt"

func main() {
	_, err := fmt.Println("hi") // ignore the byte count, keep err
	_ = err
}
```

---

## 13. Common mistakes

| # | Mistake | Fix |
|---|---------|-----|
| 1 | `a := 10` then `a := 20` | Use `a = 20` for the second |
| 2 | `x := 10` then `x = "hello"` | Types can't change; use a different variable |
| 3 | `:=` outside a function | Use `var x = 10` at package level |
| 4 | Assigning to a `const` | Use `var` if the value must change |
| 5 | Single quotes for strings | `"text"`, not `'text'` |
| 6 | Unused variable | Use it, delete it, or assign to `_` |
| 7 | `int * float64` | Convert: `float64(a) * b` |
| 8 | Relying on `float` for money | Store integer cents |
| 9 | `x := 5 / 2` expecting `2.5` | Integer division gives `2`; use `5.0 / 2` |

Number 9 deserves a demo, because it surprises everyone once:

```go
package main

import "fmt"

func main() {
	fmt.Println(5 / 2)     // 2    (both are integers → integer result)
	fmt.Println(5.0 / 2)   // 2.5  (one side is a float)
	fmt.Println(5 % 2)     // 1    (remainder)
}
```

---

## 14. Exercises

### Exercise 1: Profile card
Declare `name` (string), `age` (int), `height` (float64, in metres), and `isStudent` (bool) using `:=`. Print them with `Printf`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	name := "Asha"
	age := 22
	height := 1.65
	isStudent := true

	fmt.Printf("Name: %s, Age: %d, Height: %.2fm, Student: %t\n", name, age, height, isStudent)
}
```
Output: `Name: Asha, Age: 22, Height: 1.65m, Student: true`
</details>

### Exercise 2: Swap
Given `a := "left"` and `b := "right"`, swap them and print.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	a, b := "left", "right"
	a, b = b, a
	fmt.Println(a, b) // right left
}
```
</details>

### Exercise 3: Circle area
Using a constant `Pi = 3.14159` and `radius := 5.0`, compute and print the area (π × r²).

<details><summary>Solution</summary>

```go
package main

import "fmt"

const Pi = 3.14159

func main() {
	radius := 5.0
	area := Pi * radius * radius
	fmt.Printf("Area = %.2f\n", area) // Area = 78.54
}
```
</details>

### Exercise 4: Temperature
Convert 36.6 °C to Fahrenheit: `F = C × 9/5 + 32`. Careful with integer division!

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	c := 36.6
	f := c*9/5 + 32
	fmt.Printf("%.1f°C = %.1f°F\n", c, f) // 36.6°C = 97.9°F
}
```
Because `c` is a `float64`, the whole expression is floating-point.
</details>

### Exercise 5: Predict the output (no running!)

```go
package main

import "fmt"

func main() {
	x := 7
	y := 2
	fmt.Println(x / y)
	fmt.Println(x % y)
	fmt.Println(float64(x) / float64(y))
}
```

<details><summary>Solution</summary>

`3`, `1`, `3.5`
</details>

### Exercise 6 (challenge): Fix the errors

```go
// INTENTIONAL ERROR: find every bug
package main

import "fmt"

func main() {
	const limit = 10
	limit = 20
	count := 5
	count := 6
	var name string = 42
	unused := "hi"
	fmt.Println(count, name)
}
```

<details><summary>Solution</summary>

1. `limit = 20`: can't assign to a constant. Remove it or use `var`.
2. `count := 6`: use `count = 6`.
3. `var name string = 42`: `42` isn't a string; use `"42"`.
4. `unused` is never used. Delete it.
</details>

---

## 15. Quiz

1. What are the three parts of a variable?
2. What's the zero value of a `string`? Of a `bool`?
3. When must you use `var` instead of `:=`?
4. Why does `fmt.Println(7 / 2)` print `3`?
5. Can you write `var a int = 3.7`?
6. What's the difference between `const` and `var`?

<details><summary>Answers</summary>

1. Name, type, value.
2. `""` and `false`.
3. At package level (outside functions), or when you want the zero value / a specific type without assigning.
4. Both operands are integers, so Go does integer division and drops the fraction.
5. No. `3.7` isn't a whole number so it can't be an `int` (compile error). Convert explicitly with `int(3.7)` on a *variable*, not a constant.
6. A `const` can never change and must be known at compile time; a `var` can be reassigned.
</details>

---

## 16. Summary

- A **variable** is a named slot in memory holding a value of a fixed **type**.
- Core types: `int`, `float64`, `bool`, `string` (plus `rune`, `byte`, sized ints).
- Declare with `var x int = 1`, `var x = 1`, `x := 1` (⭐ preferred inside functions), or `var x int` (zero value).
- Every variable has a **zero value**; there is no garbage in Go.
- `:=` declares, `=` reassigns. **Types never change.**
- **`const`** values are fixed at compile time; `iota` builds numbered sets of constants.
- Go **never converts types implicitly.** Convert with `T(x)`; use `strconv` for text ↔ number.
- Unused variables and imports are compile errors.

### ➡️ What's next?

[Chapter 3](03-making-decisions.md) teaches your program to **make decisions** with `if`, `else`, and `switch`.
