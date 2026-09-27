# Chapter 33: Data Types in Depth — Integers, Floats, Booleans, Bytes, Runes, and Strings

> **Goal of this chapter:** Go beyond `int` and `string`. You'll learn every basic type Go offers, how many **bits** each uses, the **range** of values each can hold, why there are *signed* and *unsigned* integers, how floats really work, what **`byte`** and **`rune`** are (a favorite interview topic), how strings are stored as UTF-8, and how to format output precisely with `fmt.Printf`.

> *(Earlier editions of this course nicknamed this chapter "Bogus Data Types", a joke about keeping it short. These types are anything but bogus: they're the foundation of everything you build.)*

**Difficulty:** 🟡–🟠 Intermediate  **Estimated time:** 2.5–3 hours  **Prerequisite:** [Chapters 2 and 26](02-variables-and-data-types.md) (bits and bytes)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What is a data type?](#2-what-is-a-data-type)
3. [Bits and ranges: the math](#3-bits-and-ranges-the-math)
4. [Signed integers](#4-signed-integers)
5. [Unsigned integers](#5-unsigned-integers)
6. [`int` and `uint`: the platform-sized types](#6-int-and-uint-platform-sized)
7. [Overflow and wraparound](#7-overflow-and-wraparound)
8. [Integer division and remainder](#8-integer-division-and-remainder)
9. [Number literals](#9-number-literals)
10. [Floating-point types](#10-floating-point-types)
11. [Booleans](#11-booleans)
12. [`byte`: an alias for `uint8`](#12-byte)
13. [`rune`: a Unicode character](#13-rune)
14. [Strings: UTF-8 bytes](#14-strings)
15. [Converting between types](#15-converting-between-types)
16. [Formatted printing with `fmt.Printf`](#16-formatted-printing)
17. [Choosing the right type](#17-choosing-the-right-type)
18. [The Go runtime: your program's built-in "mini OS"](#18-the-go-runtime)
19. [Common mistakes](#19-common-mistakes)
20. [Interview questions](#20-interview-questions)
21. [Exercises](#21-exercises)
22. [Quiz](#22-quiz)
23. [Summary](#23-summary)

---

## 1. What you will learn

- The full family of Go's basic types and their sizes and ranges
- Why signed/unsigned and sized integers exist, and when to use each
- What happens on **overflow**
- How floats work and their pitfalls
- The difference between `byte`, `rune`, and `string`, and why `len("世界")` isn't 2
- How to format numbers and text with `Printf` verbs
- A first look at the **Go runtime**

---

## 2. What is a data type?

A **type** tells the compiler (and you) three things about a value:

1. **How much memory** it occupies (e.g., 1 byte, 8 bytes)
2. **How to interpret** those bits (a number? a letter? a truth value?)
3. **What operations** are allowed (`+` on numbers, not on booleans)

```go
var age int8 = 25
//  │   │   │    └── value
//  │   │   └─────── type: an 8-bit signed integer (1 byte of memory)
//  │   └─────────── name
//  └─────────────── keyword
```

Recall from Chapter 26: memory is a long row of **bytes** (8 bits each). A type decides how many bytes to reserve and how to read them. The very same bits `01000001` are `65` as an `int8`, but `'A'` as a `rune` or `byte` printed with `%c`.

Go's basic types:

```
Basic types
├── Boolean:    bool
├── Integers:   int, int8, int16, int32, int64
│               uint, uint8, uint16, uint32, uint64, uintptr
│               (aliases: byte = uint8, rune = int32)
├── Floats:     float32, float64
├── Complex:    complex64, complex128
└── String:     string
```

---

## 3. Bits and ranges: the math

An integer type with **n bits** can represent **2ⁿ** different values.

- **Unsigned** (no negatives): `0` to `2ⁿ − 1`
- **Signed** (positive and negative): `−2ⁿ⁻¹` to `2ⁿ⁻¹ − 1`

Why is the signed range asymmetric (−128 to **127**)? Zero takes one of the non-negative slots, so there's one more negative value than positive.

| Bits | Values (2ⁿ) | Unsigned range | Signed range |
|------|-------------|----------------|--------------|
| 8 | 256 | 0 … 255 | −128 … 127 |
| 16 | 65,536 | 0 … 65,535 | −32,768 … 32,767 |
| 32 | 4,294,967,296 | 0 … 4,294,967,295 | −2,147,483,648 … 2,147,483,647 |
| 64 | ≈ 1.8 × 10¹⁹ | 0 … 18,446,744,073,709,551,615 | −9,223,372,036,854,775,808 … 9,223,372,036,854,775,807 |

You don't need to memorize these. Go's `math` package has constants: `math.MaxInt8`, `math.MinInt64`, `math.MaxUint32`, etc.

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println(math.MinInt8, math.MaxInt8, math.MaxUint8)     // -128 127 255
	fmt.Println(math.MaxInt16, math.MaxUint16)                 // 32767 65535
	fmt.Println(math.MaxInt32, math.MaxUint32)                 // 2147483647 4294967295
	fmt.Println(math.MinInt64, math.MaxInt64, uint64(math.MaxUint64))
	// -9223372036854775808 9223372036854775807 18446744073709551615
}
```

---

## 4. Signed integers

Signed integers can hold negative numbers, zero, and positive numbers.

| Type | Bits | Bytes | Range |
|------|------|-------|-------|
| `int8` | 8 | 1 | −128 … 127 |
| `int16` | 16 | 2 | −32,768 … 32,767 |
| `int32` | 32 | 4 | ≈ ±2.1 billion |
| `int64` | 64 | 8 | ≈ ±9.2 × 10¹⁸ |
| `int` | 32 or 64 (platform) | 4 or 8 | same as int32 or int64 |

Each type occupies a fixed number of **memory cells** (bytes). An `int8` variable takes **1** cell; an `int64` takes **8** consecutive cells:

```
int8  x = 25:    [ 00011001 ]                                   1 byte

int64 y = 25:    [00000000][00000000][00000000][00000000]
                 [00000000][00000000][00000000][00011001]       8 bytes
```

The compiler chooses a type from the literal if you don't say (`x := 25` → `int`), or you can be explicit:

```go
package main

import "fmt"

func main() {
	var a int8 = 100
	var b int16 = 30000
	var c int32 = 2_000_000_000
	var d int64 = 9_000_000_000_000_000_000

	fmt.Println(a, b, c, d)
}
```

Assigning a value outside the range is a compile-time error for constants:

```go
// INTENTIONAL ERROR
package main

func main() {
	var a int8 = 200 // ❌ cannot use 200 (untyped int constant) as int8 value in variable declaration (overflows)
	_ = a
}
```

Mixing different integer types in arithmetic also requires an explicit conversion (Chapter 2): `int8(1) + int16(2)` is an error; write `int16(a) + b`.

---

## 5. Unsigned integers

**Unsigned** integers have **no sign bit**, so they can't be negative, but they can hold values twice as large as the signed positive range.

| Type | Bytes | Range |
|------|-------|-------|
| `uint8` (= `byte`) | 1 | 0 … 255 |
| `uint16` | 2 | 0 … 65,535 |
| `uint32` | 4 | 0 … 4,294,967,295 |
| `uint64` | 8 | 0 … 18,446,744,073,709,551,615 |
| `uint` | 4 or 8 | platform-sized |
| `uintptr` | 4 or 8 | big enough to hold a pointer's bits (low-level use) |

**Advantages**
- Twice the positive range (e.g., a `uint8` reaches 255, not 127).
- The type itself documents "this can never be negative" (ages, counts, sizes, IDs, pixel values, ports).

**When to use them**
- Raw bytes and bit manipulation (`uint8`/`byte`, `uint32`, `uint64` masks)
- Hashes, checksums, network protocols, file formats (fixed-width fields)
- Image data (RGB values 0–255)

**When *not* to**
- General counting or loop indexes. Go uses plain **`int`** for `len()`, indexes, and most APIs. Mixing `uint` and `int` forces conversions and invites subtle bugs (see below).

A classic unsigned trap: **underflow**.

```go
package main

import "fmt"

func main() {
	var stock uint = 0
	stock-- // no negative numbers exist, so it wraps to the maximum value!
	fmt.Println(stock) // 18446744073709551615 (on a 64-bit machine)
}
```

Never write `for i := uint(len(s)) - 1; i >= 0; i--`: `i >= 0` is *always* true for unsigned, giving an endless loop.

---

## 6. `int` and `uint`: platform-sized

`int` and `uint` are **32 bits on 32-bit platforms** and **64 bits on 64-bit platforms**. On virtually every computer you'll use today, that's 64.

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	fmt.Println(unsafe.Sizeof(int(0)), unsafe.Sizeof(uint(0)), unsafe.Sizeof(uintptr(0)))
	// 8 8 8   (on a 64-bit machine)
}
```

`int` is the **default integer type**. Use it unless you have a reason not to. It matches `len()`, slice indexes, and most standard-library signatures. Use sized types (`int32`, `int64`, `uint8`, …) when the *exact width* matters: file formats, network protocols, database columns, memory-critical arrays, or interoperating with other systems.

(`int` and `int64` are **different types** even when both are 64 bits, so converting is required: `int64(x)`.)

---

## 7. Overflow and wraparound

When arithmetic produces a value outside the type's range, Go **wraps around silently** for integers (no panic, no error):

```go
package main

import "fmt"

func main() {
	var a int8 = 127 // the maximum
	a++
	fmt.Println(a) // -128 (wrapped to the minimum)

	var b uint8 = 255
	b++
	fmt.Println(b) // 0

	var c int8 = -128
	fmt.Println(-c) // -128!  (negating the minimum overflows)
}
```

Think of an **odometer** rolling from 999999 back to 000000. This wraparound is sometimes exploited on purpose (hash functions) but is otherwise a real source of bugs and security issues (e.g., computing a buffer size from user input).

**Protect yourself:**
- Choose types large enough (`int64` for money in cents, timestamps).
- Validate input ranges.
- For critical math, check before operating, or use `math/big` for arbitrary-size numbers.

Conversions truncate too:

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	big := int64(math.MaxInt32) + 1 // 2147483648
	fmt.Println(int32(big))          // -2147483648 (truncated to 32 bits → wraps)
}
```

---

## 8. Integer division and remainder

```go
package main

import "fmt"

func main() {
	fmt.Println(7 / 2)   // 3    (integer division truncates toward zero)
	fmt.Println(-7 / 2)  // -3   (not -4!)
	fmt.Println(7 % 3)   // 1
	fmt.Println(-7 % 3)  // -1   (the remainder takes the sign of the dividend)
	fmt.Println(7.0 / 2) // 3.5  (a float operand makes it a float division)
}
```

Dividing an integer by zero panics at run time (`runtime error: integer divide by zero`); dividing a *float* by zero gives `+Inf` (or `NaN` for `0/0`).

---

## 9. Number literals

Go lets you write numbers in several bases, and add underscores for readability:

```go
package main

import "fmt"

func main() {
	fmt.Println(255)         // decimal
	fmt.Println(0b1111_1111) // binary → 255
	fmt.Println(0o377)       // octal  → 255
	fmt.Println(0xFF)        // hex    → 255
	fmt.Println(1_000_000)   // one million (underscores are ignored)
	fmt.Println(1e3, 2.5e-3) // 1000 0.0025 (float literals)
	fmt.Println('a', '\n')   // 97 10 (rune literals are numbers)
}
```

---

## 10. Floating-point types

| Type | Bits | Bytes | Precision | Range (approx.) |
|------|------|-------|-----------|-----------------|
| `float32` | 32 | 4 | ~7 decimal digits | ±3.4 × 10³⁸ |
| `float64` | 64 | 8 | ~15–16 decimal digits | ±1.8 × 10³⁰⁸ |

`float64` is the **default** (`x := 3.14` is a `float64`), and it's what the `math` package uses. Use `float32` only when memory or a specific format demands it (large arrays of numbers, graphics, ML inference).

### How floats are stored (very briefly)

IEEE 754: a **sign bit**, an **exponent**, and a **fraction** (mantissa), like scientific notation in binary:

```
value = (−1)^sign × 1.fraction × 2^exponent
```

Since the fraction has a fixed number of bits, only some numbers are representable **exactly**. `0.5`, `0.25`, `0.125` are exact (powers of two). `0.1` is *not*: in binary it's an endless repeating fraction that must be cut off.

### The consequences

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	var a, b float64 = 0.1, 0.2
	fmt.Println(a + b)      // 0.30000000000000004
	fmt.Println(a+b == 0.3) // false!

	// Compare floats with a tolerance instead of ==
	fmt.Println(math.Abs((a+b)-0.3) < 1e-9) // true

	// float32 has fewer digits:
	var f float64 = 1.0 / 3
	fmt.Println(f, float32(f)) // 0.3333333333333333 0.33333334

	// Large float32 integers lose precision
	var big float32 = 16777216 // 2^24
	fmt.Println(big+1 == big)  // true: 16777217 isn't representable in float32

	// Special values
	fmt.Println(math.Inf(1), -math.Inf(1)) // +Inf -Inf
	fmt.Println(math.NaN() == math.NaN())  // false (NaN never equals anything, even itself)
	fmt.Println(math.MaxFloat64)           // 1.7976931348623157e+308
}
```

**Rules of thumb:**
- Never compare floats with `==` (use a tolerance).
- Never store **money** in floats. Use integer cents (`int64`) or a decimal library.
- Prefer `float64` unless told otherwise.
- `NaN` and `Inf` exist; check with `math.IsNaN`, `math.IsInf`.

**Complex numbers** (`complex64`, `complex128`) exist too, and are used in scientific code (`z := 3 + 4i`). You'll rarely need them.

---

## 11. Booleans

```go
var isReady bool = true
var isDone = false
var zero bool // false
```

- Only two values: `true` and `false`. **Zero value: `false`.**
- Size: 1 byte (the smallest addressable unit).
- Produced by **comparisons** (`x > 3`) and **logical operators** (`&&`, `||`, `!`).
- **Not a number.** Unlike C or Python, you can't use `1` for true or `if x` where `x` is an int. Go requires a real `bool`.

```go
// INTENTIONAL ERROR
package main

func main() {
	n := 1
	if n { // ❌ non-boolean condition in if statement
	}
}
```

Interview favorite: *"What's the size of a `bool`?"* → **1 byte** (`unsafe.Sizeof(true) == 1`), even though one bit would suffice, because memory is byte-addressed.

---

## 12. `byte`

`byte` is simply **another name (alias) for `uint8`**: an unsigned 8-bit integer, 0–255. The two are *identical types*; the name `byte` signals "this is raw data or an ASCII character", `uint8` signals "this is a small number".

```go
package main

import "fmt"

func main() {
	var b byte = 'A' // a byte holding the ASCII code of 'A'
	fmt.Println(b)          // 65
	fmt.Printf("%c\n", b)   // A
	fmt.Printf("%T\n", b)   // uint8   ← Go reports the real name
}
```

You'll meet `[]byte` everywhere: file contents, network packets, JSON, and hashes.

```go
data := []byte("Go") // [71 111]
```

---

## 13. `rune`

`rune` is an **alias for `int32`**: a 32-bit integer holding a **Unicode code point** (one "character" in any language).

Why do we need it? A `byte` (0–255) can only represent 256 characters. The world has more than 150,000 (Latin, Bengali, Chinese, Arabic, emoji...). **Unicode** assigns every character a number (code point); a `rune` is a number big enough to hold any of them.

```go
package main

import "fmt"

func main() {
	var r rune = 'ব'   // Bengali letter BA
	var e rune = '😀'
	fmt.Println(r, e)             // 2476 128512   ← the code points
	fmt.Printf("%c %c\n", r, e)   // ব 😀
	fmt.Printf("%U\n", r)         // U+09AC        ← the standard notation
	fmt.Printf("%T\n", r)         // int32
}
```

### `byte` vs. `rune` vs. `string`: quick rules

| | `byte` | `rune` | `string` |
|--|--------|--------|----------|
| Underlying type | `uint8` | `int32` | immutable sequence of bytes |
| Represents | One byte (raw data / ASCII char) | One Unicode character (code point) | Text |
| Literal | `'A'` (fits in a byte) | `'A'`, `'世'`, `'😀'` | `"Hello"` |
| Quotes | single | single | double (or backticks) |

> Interview alert: `'a'` (single quotes) is a **rune literal** (an integer!), while `"a"` is a **string**. That's why printing `'a'` with `Println` gives `97`, not `a`.

---

## 14. Strings

A Go `string` is an **immutable sequence of bytes**, conventionally holding **UTF-8**-encoded text.

### UTF-8 in one paragraph

UTF-8 encodes each code point in **1 to 4 bytes**:
- ASCII characters (English letters, digits) → **1 byte** (compatible with ASCII)
- Accented Latin, Greek, Cyrillic → **2 bytes**
- Bengali, Devanagari, CJK (e.g., 世) → **3 bytes**
- Emoji (😀) → **4 bytes**

### The big consequence: `len` counts BYTES

```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	s := "Hello, 世界! 😀"

	fmt.Println(len(s))                    // 19   ← bytes
	fmt.Println(utf8.RuneCountInString(s)) // 12   ← characters (runes)
	fmt.Println(len([]rune(s)))            // 12   (converting to []rune also counts characters)
}
```

Hello, (7 bytes incl. comma & space) + 世界 (3+3) + `!` (1) + space (1) + 😀 (4) = 7 + 6 + 1 + 1 + 4 = **19 bytes**, but only **12 characters**.

### Indexing and `range`

```go
package main

import "fmt"

func main() {
	s := "aé世"

	// Indexing gives BYTES
	fmt.Println(s[0], s[1], s[2])       // 97 195 169   (é is two bytes: 195 169)
	fmt.Println([]byte(s))              // [97 195 169 228 184 150]

	// range decodes RUNES (and gives the BYTE index where each starts)
	for i, r := range s {
		fmt.Printf("byte index %d: %c (code point %d)\n", i, r, r)
	}
	fmt.Println([]rune(s)) // [97 233 19990]
}
```

Output of the loop:

```
byte index 0: a (code point 97)
byte index 1: é (code point 233)
byte index 3: 世 (code point 19990)
```

Notice the indexes `0, 1, 3`, not `0, 1, 2`, because `é` took two bytes.

**Rules:**
- Use `for _, r := range s` to iterate over **characters**.
- Use `[]rune(s)` when you need to index/reverse/count characters.
- Use `s[i]` only when you *know* the text is ASCII, or you really want bytes.
- Use `strings` and `unicode/utf8` for real text processing.

### Strings are immutable

```go
// INTENTIONAL ERROR
package main

func main() {
	s := "hello"
	s[0] = 'H' // ❌ cannot assign to s[0] (neither addressable nor a map index expression)
}
```

To "change" a string, build a new one (`"H" + s[1:]`), or convert to `[]rune`/`[]byte`, modify, and convert back.

### How a string looks in memory

A string value is a small **header**: `{pointer to bytes, length}` (16 bytes on 64-bit), like a slice's header (Chapter 25) without capacity. Copying a string copies the header, not the text, so passing strings around is cheap. (Strings' text lives in read-only data or on the heap.)

### Raw strings and escapes

```go
fmt.Println("Tab:\tNewline:\nQuote:\"")   // escapes are interpreted
fmt.Println(`C:\path\no\escapes`)         // backticks: raw string, no escapes, can span lines
fmt.Println("\u00e9 \U0001F600")          // Unicode escapes: é 😀
```

---

## 15. Converting between types

Go has **no implicit conversions**. Write `T(value)`:

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	// number ↔ number
	i := 42
	f := float64(i) // 42
	price := 3.99
	j := int(price) // 3 (fraction truncated; NOT rounded)
	fmt.Println(f, j)
	// Note: int(3.99) with a constant is a compile error ("constant truncated");
	// conversion of a *variable* truncates at run time.

	// integer → string: character vs. text (a classic trap!)
	fmt.Println(string(rune(65))) // "A"   (the CHARACTER with code point 65)
	fmt.Println(strconv.Itoa(65)) // "65"  (the DIGITS)

	// string ↔ []byte / []rune
	b := []byte("Go")  // [71 111]
	r := []rune("Gö")  // [71 246]
	fmt.Println(string(b), string(r))

	// string ↔ number: use strconv (returns an error!)
	n, err := strconv.Atoi("123")
	fmt.Println(n, err) // 123 <nil>
	_, err = strconv.Atoi("12a")
	fmt.Println(err)    // strconv.Atoi: parsing "12a": invalid syntax

	x, _ := strconv.ParseFloat("3.14", 64)
	ok, _ := strconv.ParseBool("true")
	fmt.Println(x, ok) // 3.14 true
	fmt.Println(strconv.FormatInt(255, 2), strconv.FormatFloat(3.14159, 'f', 2, 64)) // 11111111 3.14
}
```

Beware: `string(65)` for an `int` compiles but is flagged by `go vet` because it gives `"A"`, not `"65"`.

---

## 16. Formatted printing

`fmt.Printf(format, args...)` prints according to **verbs** (`%d`, `%s`, ...). `Sprintf` returns the string; `Fprintf` writes to a file or writer.

### Integers

| Verb | Meaning | Example (`42`) |
|------|---------|----------------|
| `%d` | decimal | `42` |
| `%5d` | width 5, right-aligned | `   42` |
| `%-5d` | width 5, left-aligned | `42   ` |
| `%05d` | zero-padded | `00042` |
| `%+d` | always show sign | `+42` |
| `%b` | binary | `101010` |
| `%o` | octal | `52` |
| `%x` / `%X` | hex lower/upper | `2a` / `2A` |
| `%c` | character for a code point | `*` |
| `%U` | Unicode notation | `U+002A` |
| `%q` | quoted character | `'*'` |

### Floats

| Verb | Meaning | Example (`math.Pi`) |
|------|---------|---------------------|
| `%f` | default 6 decimals | `3.141593` |
| `%.2f` | 2 decimals (rounds) | `3.14` |
| `%8.3f` | width 8, 3 decimals | `   3.142` |
| `%e` | scientific | `3.141593e+00` |
| `%g` | compact (shortest good form) | `3.141592653589793` |

### Strings, booleans, and generic

| Verb | Meaning | Example |
|------|---------|---------|
| `%s` | string | `go` |
| `%10s` / `%-10s` | width, right/left aligned | `        go` / `go        ` |
| `%q` | quoted string | `"go"` |
| `%x` | hex of the bytes | `676f` |
| `%t` | boolean | `true` |
| `%v` | default format for any value | (anything) |
| `%+v` | structs with field names | `{A:1}` |
| `%#v` | Go syntax | `"x"`, `main.User{...}` |
| `%T` | the type | `float64` |
| `%p` | pointer address | `0xc000...` |
| `%%` | a literal percent sign | `%` |

Real output of one run mixing many verbs:

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Printf("|%5d|%-5d|%05d|%+d|%x|%X|%o|%b|%c|%q|%U|\n", 42, 42, 42, 42, 255, 255, 8, 5, 'G', 'G', 'G')
	fmt.Printf("|%f|%.2f|%8.3f|%e|%g|\n", math.Pi, math.Pi, math.Pi, 123456.789, 123456.789)
	fmt.Printf("|%s|%10s|%-10s|%q|%v|%x|\n", "go", "go", "go", "go", "go", "go")
	fmt.Printf("%v %+v %#v %T\n", []int{1}, struct{ A int }{1}, "x", 3.5)
	fmt.Printf("%6.2f%%\n", 95.5)
	fmt.Printf("%[2]d %[1]d\n", 1, 2) // argument indexes: reorder
	fmt.Printf("%*d\n", 6, 42)        // width from an argument
}
```

```
|   42|42   |00042|+42|ff|FF|10|101|G|'G'|U+0047|
|3.141593|3.14|   3.142|1.234568e+05|123456.789|
|go|        go|go        |"go"|go|676f|
[1] {A:1} "x" float64
 95.50%
2 1
    42
```

Wrong verb for the type? Go doesn't crash; it prints an error marker (and `go vet` catches it):

```go
fmt.Printf("%d\n", "hello") // %!d(string=hello)
fmt.Printf("%s\n", 42)      // %!s(int=42)
fmt.Printf("%d %d\n", 1)    // 1 %!d(MISSING)
```

> **You don't need to memorize every verb.** Remember the key ones (`%d %s %f %v %+v %T %q %x`), and look up the rest in `go doc fmt`. What matters is *knowing they exist*.

---

## 17. Choosing the right type

| Situation | Use |
|-----------|-----|
| Counting, indexes, lengths, general integers | `int` |
| Large values (timestamps, IDs, cents, file sizes) | `int64` |
| Values known to fit a small range, in big arrays | `int8`/`int16`/`uint8`... |
| Raw bytes, binary data, ASCII | `byte` / `[]byte` |
| Characters (any language) | `rune` / `[]rune` |
| Text | `string` |
| Decimals (science, measurements) | `float64` |
| Money | `int64` cents or a decimal library, **never floats** |
| True/false | `bool` |
| Bit masks, hashes, protocol fields | `uint32` / `uint64` |

> **Don't over-optimize.** Saving one byte with `int8` rarely matters, and mixed types add conversions (and bugs). Reach for `int` and `float64` by default, and use sized types where a *specification* (file format, protocol, DB column) demands them.

---

## 18. The Go runtime

Every Go program you build contains, in addition to your code, the **Go runtime**: a library of support code that is linked into your binary (Chapter 19: that's part of why "Hello World" is ~1.5 MB). Think of it as a **"mini operating system" inside your process**:

```
┌────────────────── Your Go process (as seen by the real OS) ──────────────────┐
│                                                                              │
│  Your code (main, handlers, ...)                                             │
│  ─────────────────────────────────────────────────────────────────────────   │
│  GO RUNTIME ("mini OS")                                                      │
│    • Scheduler          - runs goroutines on OS threads                      │
│    • Memory allocator   - hands out heap memory quickly                      │
│    • Garbage collector  - reclaims unreachable memory (Chapter 18)           │
│    • Stack manager      - grows/shrinks goroutine stacks                     │
│    • Channel & timer & network-poller machinery                              │
│  ─────────────────────────────────────────────────────────────────────────   │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │ system calls
                       ┌────────▼─────────┐
                       │ Operating system │
                       └──────────────────┘
```

The real OS manages *processes* and *threads*. The Go runtime manages *goroutines*, *memory*, and *scheduling* **inside** your one process, more cheaply. When you run a Go binary, the OS starts a process, and the runtime's start-up code sets up the heap, starts the scheduler and GC, runs package initialization and `init` (Chapter 14), and finally calls `main.main`. Chapter 39 is devoted to it.

---

## 19. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Mixing `int` and `int64`/`float64` without conversion | Compile error | Convert explicitly |
| 2 | Assuming `len(s)` is the number of characters | Wrong for non-ASCII | `utf8.RuneCountInString(s)` |
| 3 | Indexing a string with non-ASCII text and treating bytes as characters | Garbled output | Use `[]rune` or `range` |
| 4 | Comparing floats with `==` | Surprising `false` | Compare with a tolerance |
| 5 | Using `float64` for money | Rounding errors | Integer cents / decimal type |
| 6 | Unsigned underflow (`u-- ` from 0, `i >= 0` loops) | Huge number / infinite loop | Use `int` |
| 7 | Silent integer overflow | Wrong results | Bigger type / range checks |
| 8 | `string(65)` when you wanted `"65"` | `"A"` | `strconv.Itoa(65)` |
| 9 | Single vs. double quotes confusion | `'ab'` is invalid; `"a"` isn't a rune | `'a'` rune, `"a"` string |
| 10 | Using `%d` with a float or `%f` with an int | `%!d(float64=...)` | Match verb to type |
| 11 | Expecting `int(3.99)` to round | It truncates → 3 | `math.Round(3.99)` first |
| 12 | Trying to modify a string in place | Compile error | Convert to `[]byte`/`[]rune` |

---

## 20. Interview questions

**Q1. What's the difference between `byte` and `rune`?**
`byte` = `uint8` (one byte, raw data/ASCII). `rune` = `int32` (one Unicode code point). Use `rune` for characters in any language.

**Q2. What does `len("世界")` return?** `6` (each character is 3 bytes in UTF-8). `utf8.RuneCountInString("世界")` returns `2`.

**Q3. What's the difference between `'a'` and `"a"`?** `'a'` is a rune (integer 97); `"a"` is a one-byte string.

**Q4. What is `int`'s size?** 4 bytes on 32-bit platforms, 8 on 64-bit.

**Q5. How many values can an `int8` hold and what are the bounds?** 256 values: −128 to 127.

**Q6. What is the default type of `x := 3.14`? Of `x := 42`? Of `x := 'a'`?** `float64`, `int`, `rune` (`int32`).

**Q7. What happens when an `int8` with 127 is incremented?** Wraps to −128.

**Q8. Why is `0.1 + 0.2 != 0.3` in floating point?** 0.1 and 0.2 can't be represented exactly in binary; the sum is a slightly different double.

**Q9. What's the size of a `bool`? Of a `string` variable?** 1 byte; 16 bytes (pointer + length) for the header on 64-bit systems.

**Q10. Are Go strings mutable?** No, immutable byte sequences.

---

## 21. Exercises

### Exercise 1: Ranges
Without running, give the range of `int16` and `uint16`. Then verify with `math`.

<details><summary>Solution</summary>

`int16`: −32,768…32,767. `uint16`: 0…65,535. `fmt.Println(math.MinInt16, math.MaxInt16, math.MaxUint16)`.
</details>

### Exercise 2: Predict the overflow

```go
var x uint8 = 200
x += 100
var y int8 = 100
y += 100
fmt.Println(x, y)
```

<details><summary>Solution</summary>

`x`: 300 mod 256 = **44**. `y`: 200 wraps: 200 − 256 = **−56**. Output: `44 -56`.
</details>

### Exercise 3: Count characters
Write `countChars(s string) (bytes, runes int)` and test with `"Go"`, `"café"`, and `"日本語"`.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func countChars(s string) (bytes, runes int) {
	return len(s), utf8.RuneCountInString(s)
}

func main() {
	for _, s := range []string{"Go", "café", "日本語"} {
		b, r := countChars(s)
		fmt.Println(s, b, r) // Go 2 2 / café 5 4 / 日本語 9 3
	}
}
```
</details>

### Exercise 4: Reverse a string (Unicode-safe)

<details><summary>Solution</summary>

```go
package main

import "fmt"

func reverse(s string) string {
	r := []rune(s)
	for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r)
}

func main() { fmt.Println(reverse("Hello, 世界")) } // 界世 ,olleH
```
(Reversing bytes instead of runes would corrupt multi-byte characters.)
</details>

### Exercise 5: Caesar cipher
Shift each lowercase ASCII letter in a string by 3 (`a→d`, `z→c`), leaving other characters unchanged.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func caesar(s string, shift int) string {
	out := []rune(s)
	for i, r := range out {
		if r >= 'a' && r <= 'z' {
			out[i] = 'a' + (r-'a'+rune(shift))%26
		}
	}
	return string(out)
}

func main() { fmt.Println(caesar("hello, xyz!", 3)) } // khoor, abc!
```
</details>

### Exercise 6: Money
Add `$19.99` and `$5.01` correctly using integer cents, and print `$25.00`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	a, b := int64(1999), int64(501) // cents
	total := a + b
	fmt.Printf("$%d.%02d\n", total/100, total%100) // $25.00
}
```
</details>

### Exercise 7: Table formatting
Print a table with columns `Item` (left-aligned, width 10), `Qty` (right-aligned, width 5), `Price` (right-aligned, 2 decimals, width 8) for two rows.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	fmt.Printf("%-10s%5s%8s\n", "Item", "Qty", "Price")
	fmt.Printf("%-10s%5d%8.2f\n", "Notebook", 3, 2.5)
	fmt.Printf("%-10s%5d%8.2f\n", "Pen", 10, 0.75)
}
```
Output:
```
Item        Qty   Price
Notebook      3    2.50
Pen          10    0.75
```
</details>

### Exercise 8 (challenge): Bits
Write `countBits(n uint) int` that counts the 1-bits in `n` using `n & 1` and `n >> 1`. Compare with `math/bits.OnesCount`.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"math/bits"
)

func countBits(n uint) int {
	count := 0
	for n > 0 {
		count += int(n & 1)
		n >>= 1
	}
	return count
}

func main() {
	fmt.Println(countBits(255), bits.OnesCount(255)) // 8 8
}
```
</details>

---

## 22. Quiz

1. How many different values fit in 12 bits?
2. What is the range of `uint8`? Of `int8`?
3. What is `byte` an alias for? And `rune`?
4. Why is `len("héllo")` equal to 6?
5. What does `x := 5 / 2` give? And `x := 5 / 2.0`?
6. Which verb prints a value's type?
7. Why should money not be a `float64`?

<details><summary>Answers</summary>

1. 2¹² = 4,096.
2. 0–255; −128–127.
3. `uint8`; `int32`.
4. `é` takes 2 bytes in UTF-8 (5 characters, 6 bytes).
5. `2` (integer division) and `2.5`.
6. `%T`.
7. Binary floats can't represent most decimal fractions exactly, causing rounding errors.
</details>

---

## 23. Summary

- A **type** fixes a value's **size, interpretation, and allowed operations**. `n` bits give 2ⁿ values.
- **Signed** ints (`int8`…`int64`) span −2ⁿ⁻¹…2ⁿ⁻¹−1; **unsigned** (`uint8`…`uint64`) span 0…2ⁿ−1. `int`/`uint` are 32- or 64-bit (platform), and `int` is the default.
- Integer arithmetic **wraps around silently**; choose ranges carefully. Integer division truncates.
- `float64` is the default float; floats are **approximate**. Never compare with `==` or store money in them.
- `bool` is 1 byte, non-numeric. `byte` = `uint8`, `rune` = `int32` (a Unicode code point).
- `string` = immutable **UTF-8 bytes**: `len` counts **bytes**, `range` yields **runes**, and `[]rune(s)` is character-indexable.
- No implicit conversions: `T(x)`, plus `strconv` for text ↔ number.
- `fmt.Printf` verbs (`%d %s %f %v %+v %T %q %x %c`) control formatting.
- The **Go runtime** (scheduler, allocator, GC) is a mini-OS bundled in every binary.

### ➡️ What's next?

[Chapter 34](34-defer.md) covers `defer`, a deceptively simple keyword that most people *don't* fully understand, including how it interacts with the stack and return values.
