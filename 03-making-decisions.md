# Chapter 3: Making Decisions — `if`, `else`, `switch` (and a First Look at `for`)

> **Goal of this chapter:** Teach your programs to *choose* what to do, and to *repeat* work. Without these, a program can only run the same straight line every time.

**Difficulty:** 🟢 Beginner  **Estimated time:** 1.5–2 hours  **Prerequisite:** [Chapter 2](02-variables-and-data-types.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why programs need decisions](#2-why-programs-need-decisions)
3. [`if` / `else if` / `else`](#3-if--else-if--else)
4. [Comparison operators](#4-comparison-operators)
5. [Logical operators: `&&`, `||`, `!`](#5-logical-operators)
6. [`if` with an init statement](#6-if-with-an-init-statement)
7. [`switch`](#7-switch)
8. [Repeating work with `for`](#8-repeating-work-with-for)
9. [Worked examples](#9-worked-examples)
10. [Common mistakes](#10-common-mistakes)
11. [Exercises](#11-exercises)
12. [Quiz](#12-quiz)
13. [Summary](#13-summary)

---

## 1. What you will learn

- Use `if`, `else if`, and `else` to run code conditionally
- Compare values with `==`, `!=`, `<`, `>`, `<=`, `>=`
- Combine conditions with `&&` (and), `||` (or), `!` (not)
- Choose between many options cleanly with `switch`
- Repeat code using Go's only loop keyword, `for`

---

## 2. Why programs need decisions

Think about logging into a website:

```
Did the user type the correct password?
   ├── YES → show the dashboard
   └── NO  → show "Wrong password"
```

The same program, given different input, does different things. That is what **conditional statements** provide. Every real application (login, shopping cart, games) is full of them.

The key ingredient is the **boolean expression**: anything that evaluates to `true` or `false` (from [Chapter 2](02-variables-and-data-types.md)). An `if` looks at a boolean and decides.

---

## 3. `if` / `else if` / `else`

### 3.1 Basic `if`

```go
package main

import "fmt"

func main() {
	age := 20

	if age >= 18 {
		fmt.Println("You are an adult")
	}

	fmt.Println("Program finished")
}
```

Output:

```
You are an adult
Program finished
```

**Syntax:**

```
if  condition  {
    // runs only when condition is true
}
```

Important Go rules:
- The condition **must be a boolean** (`true`/`false`). Unlike C or JavaScript, `if 1 {}` or `if name {}` is a compile error.
- **No parentheses** are needed around the condition (`if (age >= 18)` works but is un-Go-like; `gofmt` will remove them).
- **Curly braces are mandatory**, even for one line, and `{` must be on the same line as `if`.

### 3.2 `if` … `else`

```go
package main

import "fmt"

func main() {
	age := 15

	if age >= 18 {
		fmt.Println("Adult")
	} else {
		fmt.Println("Minor")
	}
}
```

Output: `Minor`

`else` runs when the condition is false. Exactly one of the two branches runs.

> ⚠️ `else` must be on the same line as the closing `}` of the `if`: `} else {`.

### 3.3 `else if`: more than two paths

```go
package main

import "fmt"

func main() {
	marks := 85

	if marks >= 90 {
		fmt.Println("Grade A+")
	} else if marks >= 80 {
		fmt.Println("Grade A")
	} else if marks >= 70 {
		fmt.Println("Grade B")
	} else {
		fmt.Println("Keep practising!")
	}
}
```

Output: `Grade A`

**How Go evaluates this:**

```
marks = 85
  ├─ marks >= 90?  false → skip
  ├─ marks >= 80?  TRUE  → run "Grade A", then STOP checking
  ├─ (marks >= 70 is never even tested)
  └─ (else is skipped)
```

Go checks conditions **from top to bottom** and runs only the **first** one that is true. That's why the *order* matters: if you wrote `marks >= 70` first, an 85 would be labeled "Grade B".

```
 start
   │
   ▼
 cond1? ──yes──► block 1 ──┐
   │no                     │
   ▼                       │
 cond2? ──yes──► block 2 ──┤
   │no                     │
   ▼                       │
 else block ───────────────┤
                           ▼
                     continue below
```

---

## 4. Comparison operators

Comparison operators take two values and produce a **bool**.

| Operator | Meaning | Example (`a=10`, `b=20`) | Result |
|----------|---------|--------------------------|--------|
| `==` | equal to | `a == b` | `false` |
| `!=` | not equal to | `a != b` | `true` |
| `<` | less than | `a < b` | `true` |
| `>` | greater than | `a > b` | `false` |
| `<=` | less than or equal | `a <= 10` | `true` |
| `>=` | greater than or equal | `a >= 11` | `false` |

```go
package main

import "fmt"

func main() {
	a, b := 10, 20
	fmt.Println(a == b) // false
	fmt.Println(a != b) // true
	fmt.Println(a < b)  // true
	fmt.Println(a >= 10) // true

	// Strings can be compared too (alphabetical / byte order)
	fmt.Println("apple" == "apple") // true
	fmt.Println("apple" < "banana") // true
}
```

> 🔥 **`=` vs `==`**
> - `=` **assigns**: "put this value in the variable."
> - `==` **compares**: "are these equal?"
>
> Writing `if x = 5 {` is a compile error in Go, which protects you from a famous bug in C.

Both sides must be the **same type**. `10 == 10.5` on constants works, but `intVar == floatVar` doesn't; convert first (Chapter 2).

---

## 5. Logical operators

Logical operators combine booleans.

| Operator | Name | True when… |
|----------|------|-----------|
| `&&` | AND | **both** sides are true |
| `\|\|` | OR | **at least one** side is true |
| `!` | NOT | the value is **false** (it flips it) |

### Truth tables

```
  AND (&&)                    OR (||)                   NOT (!)
 A      B     A && B         A      B     A || B         A     !A
 true   true  true           true   true  true           true  false
 true   false false          true   false true           false true
 false  true  false          false  true  true
 false  false false          false  false false
```

**Mnemonic:** AND is *strict* (everyone must agree). OR is *relaxed* (anyone will do).

### Examples

```go
package main

import "fmt"

func main() {
	age := 25
	hasTicket := true
	isVIP := false

	// AND: both must hold
	if age >= 18 && hasTicket {
		fmt.Println("You may enter")
	}

	// OR: either is enough
	if isVIP || age >= 60 {
		fmt.Println("Fast-track lane")
	} else {
		fmt.Println("Regular lane")
	}

	// NOT: flips a bool
	if !isVIP {
		fmt.Println("Not a VIP")
	}
}
```

Output:

```
You may enter
Regular lane
Not a VIP
```

### Short-circuit evaluation

Go evaluates left to right and **stops as soon as the answer is known**:

- `false && anything` → `false` (right side is never evaluated)
- `true || anything` → `true` (right side is never evaluated)

That's useful for safe checks, e.g. `if len(name) > 0 && name[0] == 'A'`. The second part would crash on an empty string, but it's never reached when the first part is false.

### Combining and grouping

`&&` binds tighter than `||` (like × before +). Use parentheses to make your intent obvious:

```go
package main

import "fmt"

func main() {
	age := 70
	isMember := false

	// (age >= 65) || (isMember) then AND with age >= 18
	if (age >= 65 || isMember) && age >= 18 {
		fmt.Println("Discount applies")
	}
}
```

---

## 6. `if` with an init statement

Go lets you run a short statement *before* the condition, separated by `;`. The variable it creates exists **only inside the `if`/`else` chain**:

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	if n, err := strconv.Atoi("42"); err != nil {
		fmt.Println("not a number:", err)
	} else {
		fmt.Println("double is", n*2)
	}
	// n and err do NOT exist here
}
```

Output: `double is 84`

You'll see this pattern *everywhere* in Go, especially for error handling (`if err != nil`), which is covered in later chapters. It keeps variables in the smallest possible **scope** (Chapters 8–9).

---

## 7. `switch`

When you compare one value against many possibilities, a long `else if` chain gets noisy. `switch` is the tidy alternative.

### 7.1 Basic switch

```go
package main

import "fmt"

func main() {
	day := 3

	switch day {
	case 1:
		fmt.Println("Monday")
	case 2:
		fmt.Println("Tuesday")
	case 3:
		fmt.Println("Wednesday")
	default:
		fmt.Println("Some other day")
	}
}
```

Output: `Wednesday`

- Go compares `day` to each `case` from top to bottom.
- The first match runs, and then **the switch ends automatically**. (In C/Java you need `break`; Go does it for you.)
- `default` runs if nothing matches. It's optional, and can appear anywhere, but convention puts it last.

### 7.2 Multiple values per case

```go
package main

import "fmt"

func main() {
	day := "Saturday"

	switch day {
	case "Saturday", "Sunday":
		fmt.Println("Weekend! 🎉")
	case "Monday", "Tuesday", "Wednesday", "Thursday", "Friday":
		fmt.Println("Weekday")
	default:
		fmt.Println("Unknown day")
	}
}
```

### 7.3 Switch with no value ("tagless")

A `switch` with no expression works like a tidy `if / else if` chain. Each `case` is a full boolean condition:

```go
package main

import "fmt"

func main() {
	marks := 76

	switch {
	case marks >= 90:
		fmt.Println("A+")
	case marks >= 80:
		fmt.Println("A")
	case marks >= 70:
		fmt.Println("B")
	default:
		fmt.Println("Below B")
	}
}
```

Output: `B`

### 7.4 `fallthrough` (rare)

By default there's no fall-through. If you *do* want to continue into the next case's body, write `fallthrough`. It's rarely needed:

```go
package main

import "fmt"

func main() {
	switch 2 {
	case 2:
		fmt.Println("two")
		fallthrough
	case 3:
		fmt.Println("three (reached via fallthrough)")
	case 4:
		fmt.Println("four (not reached)")
	}
}
```

Output:

```
two
three (reached via fallthrough)
```

### 7.5 `switch` vs `if / else if`

| Use `switch` when… | Use `if` when… |
|--------------------|----------------|
| Comparing **one value** against several constants | Conditions are unrelated or complex |
| There are 3+ branches | There are 1–2 branches |
| You want a clean, scannable layout | You need `&&`/`||` heavy logic (although tagless `switch` can do this too) |

---

## 8. Repeating work with `for`

> 📝 **Added note:** Later chapters use loops before formally introducing them, so here is a compact primer. You will meet `for` again with slices in Chapter 25.

Go has **only one** loop keyword: `for`. It comes in a few shapes.

### 8.1 Classic three-part loop

```go
package main

import "fmt"

func main() {
	for i := 0; i < 5; i++ {
		fmt.Println("i is", i)
	}
}
```

Output:

```
i is 0
i is 1
i is 2
i is 3
i is 4
```

The three parts, separated by `;`:

| Part | Code | When it runs |
|------|------|--------------|
| **Init** | `i := 0` | once, before the loop starts |
| **Condition** | `i < 5` | before every iteration. If false, the loop ends |
| **Post** | `i++` | after every iteration (`i++` means `i = i + 1`) |

**Trace:**

```
i=0 → 0<5 ✔ run body → i++ → i=1
i=1 → 1<5 ✔ run body → i++ → i=2
...
i=4 → 4<5 ✔ run body → i++ → i=5
i=5 → 5<5 ✘ stop
```

### 8.2 "While" style: condition only

```go
package main

import "fmt"

func main() {
	n := 1
	for n < 100 {
		n *= 2
	}
	fmt.Println(n) // 128
}
```

### 8.3 Infinite loop with `break` and `continue`

```go
package main

import "fmt"

func main() {
	count := 0
	for {
		count++
		if count == 2 {
			continue // skip the rest of this iteration
		}
		if count > 4 {
			break // leave the loop entirely
		}
		fmt.Println("count:", count)
	}
}
```

Output:

```
count: 1
count: 3
count: 4
```

### 8.4 Preview: `range`

`for … range` walks over collections (you'll use it with arrays, slices, and maps in later chapters):

```go
package main

import "fmt"

func main() {
	for i, ch := range "Go!" {
		fmt.Println(i, string(ch))
	}
}
```

Output:

```
0 G
1 o
2 !
```

Go 1.22+ also lets you range over an integer: `for i := range 3 { ... }` prints 0, 1, 2.

---

## 9. Worked examples

### Example 1: Login check

```go
package main

import "fmt"

func main() {
	username := "admin"
	password := "secret123"

	if username == "admin" && password == "secret123" {
		fmt.Println("Login successful")
	} else if username != "admin" {
		fmt.Println("Unknown user")
	} else {
		fmt.Println("Wrong password")
	}
}
```

### Example 2: Leap year

A year is a leap year if divisible by 4, **except** centuries, which must be divisible by 400.

```go
package main

import "fmt"

func main() {
	year := 2100

	if (year%4 == 0 && year%100 != 0) || year%400 == 0 {
		fmt.Println(year, "is a leap year")
	} else {
		fmt.Println(year, "is not a leap year")
	}
}
```

Output: `2100 is not a leap year`

(`%` is the **remainder** operator: `10 % 3` is `1`.)

### Example 3: FizzBuzz (the classic interview warm-up)

Print 1–15. For multiples of 3 print `Fizz`, of 5 print `Buzz`, of both print `FizzBuzz`.

```go
package main

import "fmt"

func main() {
	for i := 1; i <= 15; i++ {
		switch {
		case i%15 == 0:
			fmt.Println("FizzBuzz")
		case i%3 == 0:
			fmt.Println("Fizz")
		case i%5 == 0:
			fmt.Println("Buzz")
		default:
			fmt.Println(i)
		}
	}
}
```

> Why check `i%15` first? Because 15 is divisible by 3 **and** 5, and the first matching case wins.

### Example 4: Nested `if`

```go
package main

import "fmt"

func main() {
	age := 30
	hasLicense := true

	if age >= 18 {
		if hasLicense {
			fmt.Println("You can drive")
		} else {
			fmt.Println("Get a license first")
		}
	} else {
		fmt.Println("Too young")
	}
}
```

Deep nesting gets hard to read. A cleaner style (called *early return* or *guard clauses*) appears once you know functions.

---

## 10. Common mistakes

### ❌ 1. `=` instead of `==`

```go
// INTENTIONAL ERROR
package main

func main() {
	x := 5
	if x = 10 { // ❌ compile error: cannot use x = 10 as value
	}
}
```

### ❌ 2. Using `&&` when you mean `||`

"Is the day Saturday **or** Sunday?"

```go
// ❌ Logic bug: a day can't be both, so this is never true
if day == "Saturday" && day == "Sunday" { }

// ✅
if day == "Saturday" || day == "Sunday" { }
```

### ❌ 3. Non-boolean condition

```go
// INTENTIONAL ERROR
package main

func main() {
	count := 1
	if count { // ❌ non-boolean condition in if statement
	}
}
```

Write `if count != 0`.

### ❌ 4. `else` on its own line

```go
// INTENTIONAL ERROR
package main

func main() {
	x := 1
	if x > 0 {
	}
	else { // ❌ syntax error: unexpected else
	}
}
```

### ❌ 5. Wrong order in an `else if` chain

```go
marks := 95
if marks >= 50 {        // 95 matches here first!
	fmt.Println("Pass")
} else if marks >= 90 { // never reached
	fmt.Println("Excellent")
}
```

Put the **most specific / highest** conditions first.

### ❌ 6. Forgetting `default`

Not an error, but if no case matches, *nothing* happens. Decide whether that's OK. A `default` that reports "unknown value" often helps debugging.

### ❌ 7. Infinite loop by accident

```go
for i := 0; i < 5; { // forgot i++ → runs forever
}
```

---

## 11. Exercises

### Exercise 1: Even or odd
Given `n := 7`, print whether it's even or odd. (Hint: `n % 2`.)

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	n := 7
	if n%2 == 0 {
		fmt.Println(n, "is even")
	} else {
		fmt.Println(n, "is odd")
	}
}
```
</details>

### Exercise 2: Largest of three
Given `a, b, c := 12, 45, 30`, print the largest.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	a, b, c := 12, 45, 30
	largest := a
	if b > largest {
		largest = b
	}
	if c > largest {
		largest = c
	}
	fmt.Println("Largest:", largest)
}
```
</details>

### Exercise 3: Days in a month (switch)
Given `month := 2` (non-leap year), print how many days it has.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	month := 2
	switch month {
	case 1, 3, 5, 7, 8, 10, 12:
		fmt.Println(31)
	case 4, 6, 9, 11:
		fmt.Println(30)
	case 2:
		fmt.Println(28)
	default:
		fmt.Println("invalid month")
	}
}
```
</details>

### Exercise 4: Sum 1 to 100
Use a `for` loop to compute 1 + 2 + … + 100.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	sum := 0
	for i := 1; i <= 100; i++ {
		sum += i
	}
	fmt.Println(sum) // 5050
}
```
</details>

### Exercise 5: Multiplication table
Print the 7 times table (`7 x 1 = 7` … `7 x 10 = 70`).

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	for i := 1; i <= 10; i++ {
		fmt.Printf("7 x %d = %d\n", i, 7*i)
	}
}
```
</details>

### Exercise 6 (challenge): Ticket pricing
Price rules: under 5 → free; 5–17 → 200; 18–59 → 500; 60+ → 300. On weekends add 100 (except free). Given `age := 30` and `weekend := true`, print the price.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	age := 30
	weekend := true

	price := 0
	switch {
	case age < 5:
		price = 0
	case age < 18:
		price = 200
	case age < 60:
		price = 500
	default:
		price = 300
	}

	if weekend && price > 0 {
		price += 100
	}
	fmt.Println("Price:", price) // 600
}
```
</details>

---

## 12. Quiz

1. What's the difference between `=` and `==`?
2. In `if a && b`, when is `b` *not* evaluated?
3. Does Go's `switch` need `break`?
4. What type must an `if` condition have?
5. What does `for {}` do?
6. In an `else if` chain, how many blocks run at most?

<details><summary>Answers</summary>

1. `=` assigns, `==` compares.
2. When `a` is false (short-circuit).
3. No. Cases don't fall through unless you write `fallthrough`.
4. `bool`.
5. Loops forever (until `break`, `return`, or the program exits).
6. One: the first whose condition is true (or the `else`).
</details>

---

## 13. Summary

- **`if / else if / else`** choose one path; conditions must be **booleans**, braces are **mandatory**, no parentheses needed.
- **Comparison:** `== != < > <= >=`. **Logic:** `&&` `||` `!` (with **short-circuiting**).
- **`if init; cond {}`** scopes a variable to the `if` chain, a very common Go idiom.
- **`switch`** is cleaner for one-value-many-options; it needs no `break`, allows multiple values per case, and can be tagless.
- **`for`** is Go's only loop: three-part, condition-only, infinite, or `range`.
- Order matters: the first true condition wins.

### ➡️ What's next?

[Chapter 4](04-introduction-to-functions.md) introduces **functions**: how to package code into reusable, named building blocks.
