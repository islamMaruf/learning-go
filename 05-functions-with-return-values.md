# Chapter 5: Functions with Return Values

> **Goal of this chapter:** Make functions *give something back*. You'll learn single and multiple return values, named returns, the `(value, error)` pattern, and how a return travels through memory.

**Difficulty:** 🟢 Beginner  **Estimated time:** 1.5 hours  **Prerequisite:** [Chapter 4](04-introduction-to-functions.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why return values?](#2-why-return-values)
3. [Return type syntax](#3-return-type-syntax)
4. [Single return value](#4-single-return-value)
5. [How a return works in memory](#5-how-a-return-works-in-memory)
6. [Multiple return values](#6-multiple-return-values)
7. [Ignoring return values with `_`](#7-ignoring-return-values-with-_)
8. [Named return values](#8-named-return-values)
9. [The `(result, error)` pattern](#9-the-result-error-pattern)
10. [Early return](#10-early-return-and-guard-clauses)
11. [Practical examples](#11-practical-examples)
12. [Common mistakes](#12-common-mistakes)
13. [Exercises](#13-exercises)
14. [Quiz](#14-quiz)
15. [Summary](#15-summary)

---

## 1. What you will learn

- Declare a function's **return type** and use the `return` keyword
- Capture returned values into variables
- Return **several** values at once (a Go specialty)
- Use **named** return values (and when *not* to)
- Handle failure with the idiomatic `value, err` pair
- See how the stack handles returns

---

## 2. Why return values?

In Chapter 4, `add` *printed* its answer. That's limiting:

- The caller can't use the result in further calculations.
- The function is tied to `Println`. What if you want to show the result in a web page instead?

A **return value** hands the result back to whoever called the function, so *they* decide what to do with it.

> **Analogy:** You ask a calculator "what's 10 + 20?" You don't want it to shout the answer to the room; you want to *receive* `30` so you can write it down, add more, or send it to a friend.

Rule of thumb: **functions should compute and return; the caller decides whether to print.**

---

## 3. Return type syntax

```go
func name(parameters) returnType {
	// ...
	return value
}
```

```go
func add(number1 int, number2 int) int {
	//                              ^^^ return type goes AFTER the parentheses
	sum := number1 + number2
	return sum
}
```

| Section | Example | Purpose |
|---------|---------|---------|
| Keyword | `func` | begins a function |
| Name | `add` | identifier |
| Inputs | `(number1 int, number2 int)` | parameters |
| **Output** | `int` | the type of value the function gives back |
| Body | `{ ... return sum }` | the work, ending with `return` |

The compiler enforces the promise: a function declared to return `int` **must** return an `int` on **every** path.

---

## 4. Single return value

```go
package main

import "fmt"

func add(number1 int, number2 int) int {
	sum := number1 + number2
	return sum
}

func main() {
	result := add(10, 20)
	fmt.Println(result) // 30

	// A call is an expression: use it anywhere a value fits
	fmt.Println(add(1, 2) * 10)   // 30
	fmt.Println(add(add(1, 2), 3)) // 6
}
```

**Mental model: the call is replaced by the value.**

```go
result := add(10, 20)
//        └── runs, returns 30 ──► becomes:  result := 30
```

### Different return types

```go
package main

import "fmt"

func getAge() int           { return 25 }
func getPrice() float64     { return 99.99 }
func getName() string       { return "Asha" }
func isAdult(age int) bool  { return age >= 18 }

func main() {
	fmt.Println(getAge(), getPrice(), getName(), isAdult(17))
}
```

Output: `25 99.99 Asha false`

Returning a boolean directly (like `isAdult`) is cleaner than `if cond { return true } else { return false }`.

### Functions that return nothing

If there is no return type, there's no value to return (as in Chapter 4). You may write a bare `return` to exit early.

---

## 5. How a return works in memory

```go
package main

import "fmt"

func add(number1 int, number2 int) int {
	sum := number1 + number2
	return sum
}

func main() {
	a := 10
	b := 20
	result := add(a, b)
	fmt.Println(result)
}
```

```
Step 1: main starts.  a=10, b=20
┌──────────────────────┐
│ main   a=10  b=20    │
└──────────────────────┘

Step 2: add(a, b) called → new frame; values copied into parameters
┌──────────────────────┐
│ add  number1=10      │
│      number2=20      │
│      sum=30          │  ← computed
├──────────────────────┤
│ main   a=10  b=20    │
└──────────────────────┘

Step 3: `return sum` → the VALUE 30 is copied to the caller;
        add's frame is destroyed
┌──────────────────────┐
│ main   a=10  b=20    │
│        result=30     │  ← receives the copy
└──────────────────────┘
```

Two things to notice:
1. What's returned is a **copy** of the value, and the local variable `sum` itself is gone.
2. The result lands in the caller's frame (`result`).

---

## 6. Multiple return values

Go functions can return **more than one** value. List the types in parentheses:

```go
package main

import "fmt"

func sumAndProduct(a, b int) (int, int) {
	return a + b, a * b
}

func main() {
	s, p := sumAndProduct(4, 5)
	fmt.Println("sum:", s, "product:", p)
}
```

Output: `sum: 9 product: 20`

**Rules**
- The number and order of returned values must match the declaration.
- The caller must receive the same number of values (`s, p := ...`), or ignore some with `_`.

**In memory:** the function's frame is destroyed after both values are copied back:

```
sumAndProduct(4, 5) returns (9, 20)
                              │   │
                    s := 9 ◄──┘   └──► p := 20
```

### Why is this so nice?

Other languages return one thing, so they either bundle values into an object/array or use "output parameters". Go's approach is direct and readable, and it powers Go's error handling (section 9).

---

## 7. Ignoring return values with `_`

If you don't need a value, discard it with the **blank identifier** `_`. Go would otherwise complain about the unused variable:

```go
package main

import "fmt"

func divmod(a, b int) (int, int) {
	return a / b, a % b
}

func main() {
	q, _ := divmod(17, 5) // only need the quotient
	fmt.Println(q)        // 3

	_, r := divmod(17, 5) // only need the remainder
	fmt.Println(r)        // 2
}
```

---

## 8. Named return values

You can give return values **names** in the signature. They become variables initialized to their zero values, and a bare `return` sends back their current values:

```go
package main

import "fmt"

func rectangle(length, width int) (area int, perimeter int) {
	area = length * width
	perimeter = 2 * (length + width)
	return // "naked return": returns area and perimeter
}

func main() {
	a, p := rectangle(10, 5)
	fmt.Println(a, p) // 50 30
}
```

**Benefits:** the names document what each value means (`(area, perimeter int)` reads better than `(int, int)`).

> ⚠️ **Style advice:** Named returns are great for *documentation*. But naked `return` in long functions makes code harder to follow. Prefer explicit `return area, perimeter` unless the function is very short.

---

## 9. The `(result, error)` pattern

Real programs fail: files go missing, input is invalid, networks drop. Go's convention is: **the last return value is an `error`.** If it's `nil`, everything went well.

```go
package main

import (
	"errors"
	"fmt"
)

func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("cannot divide by zero")
	}
	return a / b, nil
}

func main() {
	result, err := divide(10, 2)
	if err != nil {
		fmt.Println("Error:", err)
		return
	}
	fmt.Println("Result:", result) // 5

	_, err = divide(1, 0)
	if err != nil {
		fmt.Println("Error:", err) // Error: cannot divide by zero
	}
}
```

Output:

```
Result: 5
Error: cannot divide by zero
```

This `if err != nil { ... }` check is the most common pattern in all Go code. You'll see it in every chapter of the project half of this course. `nil` means "no value / nothing" (Chapter 24).

You've already met this pattern in Chapter 3's `strconv.Atoi("42")`, which returns `(int, error)`.

---

## 10. Early return and guard clauses

A function can `return` in the middle. Handling bad cases first and returning early keeps the "happy path" un-nested:

```go
package main

import "fmt"

func gradeFor(marks int) string {
	if marks < 0 || marks > 100 {
		return "invalid"
	}
	if marks >= 80 {
		return "A"
	}
	if marks >= 60 {
		return "B"
	}
	return "C"
}

func main() {
	fmt.Println(gradeFor(85), gradeFor(65), gradeFor(10), gradeFor(120))
}
```

Output: `A B C invalid`

**The compiler insists every path returns.** This fails:

```go
// INTENTIONAL ERROR
package main

func sign(n int) string {
	if n > 0 {
		return "positive"
	} else if n < 0 {
		return "negative"
	}
	// ❌ missing return: what if n == 0?
}

func main() {}
```

Error: `missing return`. Fix by adding `return "zero"` at the end.

---

## 11. Practical examples

### Example 1: Temperature conversion

```go
package main

import "fmt"

func convert(celsius float64) (fahrenheit, kelvin float64) {
	fahrenheit = celsius*9/5 + 32
	kelvin = celsius + 273.15
	return
}

func main() {
	f, k := convert(25)
	fmt.Printf("25°C = %.2f°F = %.2fK\n", f, k) // 25°C = 77.00°F = 298.15K
}
```

### Example 2: Min and max

```go
package main

import "fmt"

func minMax(a, b int) (min, max int) {
	if a < b {
		return a, b
	}
	return b, a
}

func main() {
	lo, hi := minMax(42, 7)
	fmt.Println(lo, hi) // 7 42
}
```

### Example 3: Is it prime?

```go
package main

import "fmt"

func isPrime(n int) bool {
	if n < 2 {
		return false
	}
	for i := 2; i*i <= n; i++ {
		if n%i == 0 {
			return false
		}
	}
	return true
}

func main() {
	for n := 1; n <= 20; n++ {
		if isPrime(n) {
			fmt.Print(n, " ")
		}
	}
	fmt.Println()
}
```

Output: `2 3 5 7 11 13 17 19 `

### Example 4: Composing functions

```go
package main

import "fmt"

func square(n int) int { return n * n }

func sumOfSquares(a, b int) int {
	return square(a) + square(b)
}

func main() {
	fmt.Println(sumOfSquares(3, 4)) // 25  (the Pythagorean triple: 9 + 16)
}
```

### Example 5: Safe parsing with an error

```go
package main

import (
	"fmt"
	"strconv"
)

func parseAge(text string) (int, error) {
	age, err := strconv.Atoi(text)
	if err != nil {
		return 0, fmt.Errorf("invalid age %q: %w", text, err)
	}
	if age < 0 {
		return 0, fmt.Errorf("age cannot be negative: %d", age)
	}
	return age, nil
}

func main() {
	for _, in := range []string{"30", "abc", "-5"} {
		age, err := parseAge(in)
		if err != nil {
			fmt.Println("error:", err)
			continue
		}
		fmt.Println("age:", age)
	}
}
```

Output:

```
age: 30
error: invalid age "abc": strconv.Atoi: parsing "abc": invalid syntax
error: age cannot be negative: -5
```

---

## 12. Common mistakes

| # | Mistake | Compiler message | Fix |
|---|---------|------------------|-----|
| 1 | Forgetting the return type | `too many return values` | Add it: `func f() int` |
| 2 | Forgetting `return` | `missing return` | Add a `return` on every path |
| 3 | Returning the wrong type | `cannot use "x" (string) as int value in return statement` | Match the declared type |
| 4 | Wrong number of results | `assignment mismatch: 1 variable but f returns 2 values` | Receive both, or use `_` |
| 5 | Ignoring an error silently | (no error, but a hidden bug) | Always check `err` |
| 6 | Code after `return` | `unreachable code` (from `go vet`) | Remove or restructure |
| 7 | Using `:=` for a result that already exists | `no new variables on left side` | Use `=` |

Mistake #4 in code:

```go
// INTENTIONAL ERROR
package main

import "fmt"

func two() (int, int) { return 1, 2 }

func main() {
	x := two() // ❌ assignment mismatch: 1 variable but two() returns 2 values
	fmt.Println(x)
}
```

---

## 13. Exercises

### Exercise 1: Square
Write `square(n int) int` and print `square(9)`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func square(n int) int { return n * n }

func main() { fmt.Println(square(9)) } // 81
```
</details>

### Exercise 2: Circle
Write `circle(radius float64) (area, circumference float64)` using `math.Pi`.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"math"
)

func circle(radius float64) (area, circumference float64) {
	area = math.Pi * radius * radius
	circumference = 2 * math.Pi * radius
	return
}

func main() {
	a, c := circle(3)
	fmt.Printf("area=%.2f circumference=%.2f\n", a, c) // area=28.27 circumference=18.85
}
```
</details>

### Exercise 3: Absolute value
Write `abs(n int) int` returning the absolute value (no library).

<details><summary>Solution</summary>

```go
package main

import "fmt"

func abs(n int) int {
	if n < 0 {
		return -n
	}
	return n
}

func main() { fmt.Println(abs(-7), abs(7), abs(0)) } // 7 7 0
```
</details>

### Exercise 4: Safe square root
Write `safeSqrt(x float64) (float64, error)` that returns an error for negatives.

<details><summary>Solution</summary>

```go
package main

import (
	"errors"
	"fmt"
	"math"
)

func safeSqrt(x float64) (float64, error) {
	if x < 0 {
		return 0, errors.New("cannot take sqrt of a negative number")
	}
	return math.Sqrt(x), nil
}

func main() {
	if v, err := safeSqrt(16); err == nil {
		fmt.Println(v) // 4
	}
	if _, err := safeSqrt(-1); err != nil {
		fmt.Println(err)
	}
}
```
</details>

### Exercise 5: Stats
Write `stats(a, b, c int) (sum int, average float64)`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func stats(a, b, c int) (sum int, average float64) {
	sum = a + b + c
	average = float64(sum) / 3
	return
}

func main() {
	s, avg := stats(10, 20, 25)
	fmt.Printf("sum=%d avg=%.2f\n", s, avg) // sum=55 avg=18.33
}
```
</details>

### Exercise 6 (challenge): Fix it

```go
// INTENTIONAL ERROR
package main

import "fmt"

func describe(n int) string {
	if n > 0 {
		return "positive"
	}
	if n < 0 {
		return "negative"
	}
}

func main() {
	x := describe(5), describe(-1)
	fmt.Println(x)
}
```

<details><summary>Solution</summary>

Two bugs: `describe` lacks a final `return "zero"`, and `x := describe(5), describe(-1)` isn't valid. Fix:

```go
package main

import "fmt"

func describe(n int) string {
	if n > 0 {
		return "positive"
	}
	if n < 0 {
		return "negative"
	}
	return "zero"
}

func main() {
	x, y := describe(5), describe(-1)
	fmt.Println(x, y)
}
```
</details>

---

## 14. Quiz

1. Where does the return type go in a function signature?
2. What does the compiler require of every code path in a function with a return type?
3. How do you ignore one of multiple returned values?
4. In `(int, error)`, what does `err == nil` mean?
5. What is a "naked return"?
6. Is the returned value a copy or the original?

<details><summary>Answers</summary>

1. After the parameter list: `func f(a int) int`.
2. It must end in a `return` (or otherwise not fall off the end).
3. Assign it to `_`.
4. The call succeeded.
5. A `return` with no values in a function that has named results.
6. A copy.
</details>

---

## 15. Summary

- `func f(params) T { return value }`: declare the return type after the parameters.
- The compiler makes sure **every path returns**.
- A call is **replaced by its returned value**, which is **copied** to the caller.
- Go can return **multiple values**; capture with `a, b := f()`, discard with `_`.
- **Named results** document meaning; use naked `return` sparingly.
- The idiomatic failure signal is a final **`error`** result: check `if err != nil`.
- Return early on bad input to keep code flat.

### ➡️ What's next?

[Chapter 6](06-more-function-examples.md) practices all the function shapes (with/without parameters and returns), plus variadic functions.
