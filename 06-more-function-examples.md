# Chapter 6: More Function Examples — Practising Every Shape

> **Goal of this chapter:** Build fluency. You will practise the four "shapes" of functions, work with string parameters, learn every way to combine text and values, and meet variadic functions.

**Difficulty:** 🟢 Beginner  **Estimated time:** 1.5 hours  **Prerequisite:** [Chapter 5](05-functions-with-return-values.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The four function shapes](#2-the-four-function-shapes)
3. [Shape 1: no input, no output](#3-shape-1-no-input-no-output)
4. [Shape 2: input, no output](#4-shape-2-input-no-output)
5. [Shape 3: no input, output](#5-shape-3-no-input-output)
6. [Shape 4: input and output](#6-shape-4-input-and-output)
7. [Working with string parameters](#7-working-with-string-parameters)
8. [Combining text and values: comma, `+`, `Printf`, `Sprintf`](#8-combining-text-and-values)
9. [Variadic functions: any number of arguments](#9-variadic-functions)
10. [Choosing the right shape](#10-choosing-the-right-shape)
11. [Practical mini-programs](#11-practical-mini-programs)
12. [Tips and best practices](#12-tips-and-best-practices)
13. [Common mistakes](#13-common-mistakes)
14. [Exercises](#14-exercises)
15. [Quiz](#15-quiz)
16. [Summary](#16-summary)

---

## 1. What you will learn

- Recognize and write all four combinations of *parameters* × *return value*
- Pass and manipulate strings in functions
- Understand the difference between `Println("a", b)` and `Println("a" + b)`
- Build formatted strings with `fmt.Sprintf`
- Write functions that accept **any number** of arguments (`...`)
- Pick good function names and sizes

---

## 2. The four function shapes

Every function either **takes input** or doesn't, and either **gives output** or doesn't. That makes four shapes:

|  | **No return value** | **Returns a value** |
|--|---------------------|---------------------|
| **No parameters** | `func hello()` | `func getYear() int` |
| **Has parameters** | `func greet(name string)` | `func add(a, b int) int` |

```
        ┌──────────┐
input ─►│ function │─► output
        └──────────┘
   (optional)     (optional)
```

---

## 3. Shape 1: no input, no output

Does a fixed job, the same way each time. Good for banners, menus, and separators.

```go
package main

import "fmt"

func printMenu() {
	fmt.Println("1. Start")
	fmt.Println("2. Settings")
	fmt.Println("3. Quit")
}

func main() {
	printMenu()
}
```

---

## 4. Shape 2: input, no output

Takes data and does something (prints, saves, sends) but doesn't hand a result back. These are often called **procedures** or functions with **side effects**.

```go
package main

import "fmt"

func greet(name string) {
	fmt.Println("Welcome to the Go course,", name)
}

func main() {
	greet("Asha")
	greet("Rahim")
}
```

Output:

```
Welcome to the Go course, Asha
Welcome to the Go course, Rahim
```

---

## 5. Shape 3: no input, output

Produces a value from nothing you pass in: constants, the current time, configuration.

```go
package main

import "fmt"

func appName() string {
	return "MyShop"
}

func maxRetries() int {
	return 3
}

func main() {
	fmt.Println(appName(), maxRetries())
}
```

---

## 6. Shape 4: input and output

The most useful: a pure calculation. Data in → answer out. The caller decides what to do with the answer.

```go
package main

import "fmt"

func fullName(first, last string) string {
	return first + " " + last
}

func main() {
	name := fullName("Asha", "Rahman")
	fmt.Println(name) // Asha Rahman
}
```

> 💡 Functions that only depend on their inputs and have no side effects are called **pure functions**. They're the easiest kind to test and reason about. Prefer them where you can.

---

## 7. Working with string parameters

Strings behave like any other parameter. Here are a few things you can do with them:

```go
package main

import (
	"fmt"
	"strings"
)

func shout(text string) string {
	return strings.ToUpper(text) + "!"
}

func describe(text string) {
	fmt.Println("Text:", text)
	fmt.Println("Length (bytes):", len(text))
	fmt.Println("First byte:", text[0], "=", string(text[0]))
	fmt.Println("Contains 'go':", strings.Contains(text, "go"))
}

func main() {
	fmt.Println(shout("hello"))
	describe("golang")
}
```

Output:

```
HELLO!
Text: golang
Length (bytes): 6
First byte: 103 = g
Contains 'go': true
```

Handy `strings` functions:

| Function | Example | Result |
|----------|---------|--------|
| `strings.ToUpper(s)` | `ToUpper("go")` | `"GO"` |
| `strings.ToLower(s)` | `ToLower("GO")` | `"go"` |
| `strings.Contains(s, sub)` | `Contains("golang","lang")` | `true` |
| `strings.HasPrefix(s, p)` | `HasPrefix("golang","go")` | `true` |
| `strings.Split(s, sep)` | `Split("a,b,c", ",")` | `["a" "b" "c"]` |
| `strings.TrimSpace(s)` | `TrimSpace("  hi ")` | `"hi"` |
| `strings.Repeat(s, n)` | `Repeat("ab", 3)` | `"ababab"` |
| `strings.Replace(s, old, new, n)` | `Replace("aaa","a","b",2)` | `"bba"` |

---

## 8. Combining text and values

There are four common ways to put text and values together. Know all four:

### 8.1 Commas in `Println`: automatic spaces

```go
fmt.Println("Name:", name, "Age:", age)
```
- Works with **any types** mixed.
- Adds a **space** between each value and a newline at the end.

### 8.2 The `+` operator: concatenation (strings only)

```go
fmt.Println("Hello, " + name)
```
- **Both sides must be strings.** `"Age: " + 25` is a compile error.
- **No automatic space.** You add spaces yourself.

```go
package main

import "fmt"

func main() {
	name := "Asha"
	fmt.Println("Welcome" + name)   // WelcomeAsha   (no space!)
	fmt.Println("Welcome", name)    // Welcome Asha  (comma adds a space)
	fmt.Println("Welcome " + name)  // Welcome Asha  (you added the space)
}
```

### 8.3 `Printf`: templates

```go
fmt.Printf("%s is %d years old\n", name, age)
```
Best when you want control: decimal places (`%.2f`), padding (`%5d`), alignment (`%-10s`).

### 8.4 `Sprintf`: build a string without printing

`Sprintf` works like `Printf` but **returns** the text instead of printing it, which is ideal for functions:

```go
package main

import "fmt"

func label(name string, age int) string {
	return fmt.Sprintf("%s (%d)", name, age)
}

func main() {
	fmt.Println(label("Asha", 30))  // Asha (30)
	fmt.Println(label("Rahim", 25)) // Rahim (25)
}
```

> The `S` stands for **S**tring. Family: `Print`/`Println`/`Printf` write to the screen; `Sprint`/`Sprintln`/`Sprintf` return a string.

### 8.5 Formatting cheat sheet

```go
package main

import "fmt"

func main() {
	fmt.Printf("%d\n", 42)        // 42
	fmt.Printf("%5d|\n", 42)      //    42|   (width 5, right-aligned)
	fmt.Printf("%-5d|\n", 42)     // 42   |   (left-aligned)
	fmt.Printf("%05d\n", 42)      // 00042    (zero padded)
	fmt.Printf("%.2f\n", 3.14159) // 3.14
	fmt.Printf("%8.3f|\n", 3.14159) //    3.142|
	fmt.Printf("%s\n", "go")      // go
	fmt.Printf("%q\n", "go")      // "go"
	fmt.Printf("%v %T\n", 7, 7)   // 7 int
	fmt.Printf("%t\n", true)      // true
	fmt.Printf("%x\n", 255)       // ff
	fmt.Printf("%b\n", 5)         // 101
	fmt.Printf("%c\n", 'A')       // A
	fmt.Printf("100%%\n")         // 100%   (write %% for a literal percent)
}
```

---

## 9. Variadic functions

What if you don't know how many arguments you'll get? Put `...` before the type of the **last** parameter:

```go
package main

import "fmt"

func sum(numbers ...int) int {
	total := 0
	for _, n := range numbers {
		total += n
	}
	return total
}

func main() {
	fmt.Println(sum())            // 0
	fmt.Println(sum(5))           // 5
	fmt.Println(sum(1, 2, 3, 4))  // 10
}
```

Inside the function, `numbers` behaves like a **slice** of ints (Chapter 25). `for _, n := range numbers` visits each one (`_` discards the index).

To pass an existing slice, add `...` after it:

```go
package main

import "fmt"

func sum(numbers ...int) int {
	total := 0
	for _, n := range numbers {
		total += n
	}
	return total
}

func main() {
	scores := []int{10, 20, 30}
	fmt.Println(sum(scores...)) // 60
}
```

You've been calling a variadic function all along: `fmt.Println(a ...any)` accepts anything, any number of times.

Rules: only the **last** parameter can be variadic, and there can be only one.

```go
func report(title string, values ...float64) { /* OK */ }
```

---

## 10. Choosing the right shape

| Question | Suggests |
|----------|----------|
| Does it need outside data? | Add parameters |
| Will the caller need the answer for something else? | Return a value |
| Is it purely about display or storing? | No return |
| Can it fail? | Return an `error` too |
| Does the number of inputs vary? | Variadic |

A common design: **calculate in one function, print in another.**

```go
package main

import "fmt"

// Pure: easy to test, reusable
func average(a, b, c float64) float64 {
	return (a + b + c) / 3
}

// Presentation only
func printReport(name string, avg float64) {
	fmt.Printf("%-8s average: %6.2f\n", name, avg)
}

func main() {
	printReport("Asha", average(80, 90, 100))
	printReport("Rahim", average(55, 60, 70))
}
```

Output:

```
Asha     average:  90.00
Rahim    average:  61.67
```

---

## 11. Practical mini-programs

### 11.1 Simple calculator

```go
package main

import (
	"errors"
	"fmt"
)

func calculate(a, b float64, op string) (float64, error) {
	switch op {
	case "+":
		return a + b, nil
	case "-":
		return a - b, nil
	case "*":
		return a * b, nil
	case "/":
		if b == 0 {
			return 0, errors.New("division by zero")
		}
		return a / b, nil
	default:
		return 0, fmt.Errorf("unknown operator %q", op)
	}
}

func main() {
	tests := []struct {
		a, b float64
		op   string
	}{
		{10, 5, "+"}, {10, 5, "/"}, {10, 0, "/"}, {2, 3, "^"},
	}
	for _, t := range tests {
		res, err := calculate(t.a, t.b, t.op)
		if err != nil {
			fmt.Println("error:", err)
			continue
		}
		fmt.Printf("%.0f %s %.0f = %.2f\n", t.a, t.op, t.b, res)
	}
}
```

Output:

```
10 + 5 = 15.00
10 / 5 = 2.00
error: division by zero
error: unknown operator "^"
```

### 11.2 Palindrome checker

```go
package main

import (
	"fmt"
	"strings"
)

func isPalindrome(s string) bool {
	s = strings.ToLower(s)
	runes := []rune(s)
	for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
		if runes[i] != runes[j] {
			return false
		}
	}
	return true
}

func main() {
	fmt.Println(isPalindrome("Level")) // true
	fmt.Println(isPalindrome("Go"))    // false
}
```

### 11.3 Receipt printer (all shapes together)

```go
package main

import "fmt"

func line() { fmt.Println("--------------------------") }

func item(name string, qty int, price float64) float64 {
	total := float64(qty) * price
	fmt.Printf("%-12s %2d x %6.2f = %7.2f\n", name, qty, price, total)
	return total
}

func main() {
	line()
	sum := item("Notebook", 3, 2.50)
	sum += item("Pen", 10, 0.75)
	line()
	fmt.Printf("%-25s %.2f\n", "TOTAL", sum)
}
```

Output:

```
--------------------------
Notebook      3 x   2.50 =    7.50
Pen          10 x   0.75 =    7.50
--------------------------
TOTAL                     15.00
```

---

## 12. Tips and best practices

1. **One function, one job.** If you need "and" to describe it ("validates *and* saves *and* emails"), split it.
2. **Name by intent.** Use verbs for actions (`printReport`, `sendEmail`), and nouns or `is`/`has` for values and booleans (`fullName`, `isPrime`).
3. **Keep them short.** If a function doesn't fit on one screen, it probably does too much.
4. **Prefer returning over printing** so other code can reuse the result.
5. **Fewer parameters is better.** More than 3–4 is a hint to group them into a struct (Chapter 21).
6. **Keep `main` small.** It should read like a table of contents.
7. **Comment exported functions** starting with the function's name:

```go
// Average returns the mean of a, b, and c.
func Average(a, b, c float64) float64 { return (a + b + c) / 3 }
```

---

## 13. Common mistakes

| # | Mistake | Why | Fix |
|---|---------|-----|-----|
| 1 | `"Age: " + 25` | `+` needs two strings | `"Age: " + strconv.Itoa(25)` or use commas / `Sprintf` |
| 2 | `fmt.Println("Hi" + name)` prints `Hiname`-style with no space | `+` doesn't add spaces | `"Hi " + name` or `Println("Hi", name)` |
| 3 | Calling a value-returning function and ignoring the value | e.g. `strings.ToUpper(s)` alone does nothing | `s = strings.ToUpper(s)` (strings are immutable) |
| 4 | `Printf` without `\n` | Output runs together | Add `\n` |
| 5 | Wrong verb: `%d` with a string | prints `%!d(string=hi)` | Use `%s` |
| 6 | Variadic param not last | compile error | Move it to the end |
| 7 | Passing a slice to `...T` without `...` | type mismatch | `f(slice...)` |
| 8 | `Printf("%s")` with too few args | `%!s(MISSING)` | Match verbs to args (`go vet` catches it) |

Mistake #3 is a classic:

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	s := "hello"
	strings.ToUpper(s)      // ❌ result thrown away: s is unchanged
	fmt.Println(s)          // hello
	s = strings.ToUpper(s)  // ✅ capture the result
	fmt.Println(s)          // HELLO
}
```

---

## 14. Exercises

### Exercise 1: Four shapes
Write one function of each shape: `printLine()`, `welcome(name string)`, `pi() float64`, `triple(n int) int`. Call all from `main`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func printLine()           { fmt.Println("==========") }
func welcome(name string)  { fmt.Println("Welcome,", name) }
func pi() float64          { return 3.14159 }
func triple(n int) int     { return n * 3 }

func main() {
	printLine()
	welcome("Asha")
	fmt.Println(pi(), triple(7))
	printLine()
}
```
</details>

### Exercise 2: Initials
Write `initials(first, last string) string` returning e.g. `"A.R."` for `("Asha","Rahman")`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func initials(first, last string) string {
	return string(first[0]) + "." + string(last[0]) + "."
}

func main() { fmt.Println(initials("Asha", "Rahman")) } // A.R.
```
(This works for ASCII names. Names with multi-byte characters need `[]rune`.)
</details>

### Exercise 3: Maximum of many
Write `max(nums ...int) int`. Return 0 for no arguments.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func maxOf(nums ...int) int {
	if len(nums) == 0 {
		return 0
	}
	best := nums[0]
	for _, n := range nums[1:] {
		if n > best {
			best = n
		}
	}
	return best
}

func main() {
	fmt.Println(maxOf(3, 9, 2), maxOf(), maxOf(-5, -1)) // 9 0 -1
}
```
</details>

### Exercise 4: Formatted price
Write `formatPrice(amount float64) string` returning `"$12.50"` for `12.5`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func formatPrice(amount float64) string {
	return fmt.Sprintf("$%.2f", amount)
}

func main() { fmt.Println(formatPrice(12.5)) } // $12.50
```
</details>

### Exercise 5: Word counter
Write `countWords(text string) int` using `strings.Fields`.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"strings"
)

func countWords(text string) int {
	return len(strings.Fields(text))
}

func main() { fmt.Println(countWords("Go is   really fun")) } // 4
```
`strings.Fields` splits on any whitespace and ignores extra spaces.
</details>

### Exercise 6 (challenge): Temperature table
Print a table of Celsius 0, 10, …, 100 with Fahrenheit, using a helper `toF(c float64) float64` and `Printf` alignment.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func toF(c float64) float64 { return c*9/5 + 32 }

func main() {
	fmt.Printf("%6s %8s\n", "C", "F")
	for c := 0.0; c <= 100; c += 10 {
		fmt.Printf("%6.0f %8.1f\n", c, toF(c))
	}
}
```
</details>

---

## 15. Quiz

1. Name the four function shapes.
2. Why does `fmt.Println("Hi" + name)` produce no space?
3. What's the difference between `Printf` and `Sprintf`?
4. Which parameter may be variadic?
5. What does `f(nums...)` do?
6. Why doesn't `strings.ToUpper(s)` change `s`?

<details><summary>Answers</summary>

1. No-in/no-out, in/no-out, no-in/out, in/out.
2. `+` joins strings exactly as they are; only commas in `Println` insert spaces.
3. `Printf` prints; `Sprintf` returns the formatted string.
4. Only the last one.
5. Expands a slice into individual arguments for a variadic parameter.
6. Strings are immutable. The function returns a *new* string you must capture.
</details>

---

## 16. Summary

- Functions come in **four shapes** (input? output?). Choose based on need.
- Prefer **pure** functions that compute and return; keep printing at the edges.
- Combine text with **commas** (spaces added), **`+`** (strings only), **`Printf`** (formatted), or **`Sprintf`** (formatted → string).
- **Variadic** (`...T`) accepts any number of arguments as a slice.
- Strings are **immutable**: capture the result of string functions.
- Keep functions **short, single-purpose, well named**.

### ➡️ What's next?

Part 2 begins. [Chapter 7](07-why-functions-are-needed.md) explains, with a real-world story, *why* we carve code into functions, and sets up the big idea of **scope**.
