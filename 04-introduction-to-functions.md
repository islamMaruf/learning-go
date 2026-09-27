# Chapter 4: Introduction to Functions

> **Goal of this chapter:** Learn to package code into named, reusable blocks called **functions**, and to understand what happens in memory when a function is called.

**Difficulty:** 🟢 Beginner  **Estimated time:** 1.5 hours  **Prerequisite:** [Chapter 3](03-making-decisions.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What is a function?](#2-what-is-a-function)
3. [Why functions exist](#3-why-functions-exist)
4. [Function syntax](#4-function-syntax)
5. [Calling a function](#5-calling-a-function)
6. [Parameters vs. arguments (a first look)](#6-parameters-and-arguments)
7. [What happens in memory when you call a function](#7-what-happens-in-memory)
8. [Functions calling functions](#8-functions-calling-functions)
9. [Order of definition doesn't matter](#9-order-of-definition-doesnt-matter)
10. [Practical examples](#10-practical-examples)
11. [Common mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)
13. [Quiz](#13-quiz)
14. [Summary](#14-summary)

---

## 1. What you will learn

- What a function is and why every serious program is built from them
- How to *define* a function and how to *call* it
- How to pass information into a function (parameters)
- How Go uses memory when a function runs, and why local variables "disappear"
- You already use functions: `main()` and `fmt.Println()` are both functions!

---

## 2. What is a function?

A **function** is a **named, reusable block of code** that does one job.

> **Analogy: a recipe.**
> - The recipe **name** → the function name (`makeTea`)
> - The **ingredients** you supply → the parameters (`sugar`, `milk`)
> - The **steps** → the function body
> - You can cook the recipe again and again with different ingredients.

You've already used functions without knowing:

```go
fmt.Println("hi") // Println is a function someone else wrote
```

Now you'll write your own.

---

## 3. Why functions exist

### The problem: repeated code

```go
package main

import "fmt"

func main() {
	a, b := 10, 20
	sum := a + b
	fmt.Println(sum)

	x, y := 5, 7
	sum2 := x + y
	fmt.Println(sum2)

	p, q := 100, 200
	sum3 := p + q
	fmt.Println(sum3)
}
```

The same three steps, copied three times. If you want to change how results are printed, you must edit three places (and might forget one).

### The solution: write once, use many times

```go
package main

import "fmt"

func add(number1 int, number2 int) {
	sum := number1 + number2
	fmt.Println(sum)
}

func main() {
	add(10, 20)   // 30
	add(5, 7)     // 12
	add(100, 200) // 300
}
```

### Benefits

| Benefit | Meaning |
|---------|---------|
| **Reusability** | Write once, call anywhere, any number of times |
| **Organization** | Split a big program into small named pieces (`login`, `sendEmail`, …) |
| **Maintainability** | Fix a bug in one place and every caller benefits |
| **Abstraction** | Callers need to know *what* it does, not *how* |
| **Testability** | Small functions are easy to test alone |

---

## 4. Function syntax

```
func  name  (  parameters  )  {
    body
}
```

```go
func add(number1 int, number2 int) {
	sum := number1 + number2
	fmt.Println(sum)
}
```

| Part | Example | Meaning |
|------|---------|---------|
| `func` | `func` | Keyword: "a function definition starts here" |
| **name** | `add` | You choose it. Use `camelCase`. Start with an uppercase letter to make it visible to other packages (Chapter 10). |
| **parameter list** | `(number1 int, number2 int)` | Inputs: each has a **name** then a **type** (name first, then type, the reverse of C/Java) |
| **body** | `{ ... }` | The code that runs when the function is called |

Shorthand: when consecutive parameters share a type, write the type once:

```go
func add(number1, number2 int) { ... }   // same as (number1 int, number2 int)
```

A function with no inputs has empty parentheses: `func sayHello() { ... }`.

---

## 5. Calling a function

**Defining** a function only describes what it does; nothing runs. To run it, **call** it by writing its name with parentheses:

```go
package main

import "fmt"

func sayHello() {
	fmt.Println("Hello from a function!")
}

func main() {
	fmt.Println("Before")
	sayHello() // ← the call
	sayHello() // ← call again!
	fmt.Println("After")
}
```

Output:

```
Before
Hello from a function!
Hello from a function!
After
```

**Who calls `main`?** Go itself. When your program starts, Go's runtime calls `main()`; when `main` finishes, the program ends.

### Execution flow

```
main() starts
  │
  ├─ print "Before"
  ├─ call sayHello() ──► [ run body: print "Hello…" ] ──┐
  │ ◄────────────────────── return to here ─────────────┘
  ├─ call sayHello() ──► [ run body again ] ──► back
  ├─ print "After"
  ▼
main() ends → program ends
```

When a function is called, execution **jumps** into it; when it finishes, execution **returns to the line right after the call**.

---

## 6. Parameters and arguments

Two words that beginners mix up (Chapter 17 explores this in depth):

- **Parameter**: the variable listed in the function *definition* (a placeholder: `number1`).
- **Argument**: the actual value you pass in the *call* (`10`).

```go
func add(number1 int, number2 int) { ... }  // number1, number2 = PARAMETERS
add(10, 20)                                  // 10, 20         = ARGUMENTS
```

Rules:
- You must pass the **right number** of arguments, in the **right order**, of the **right types**.
- Go passes arguments **by value**: the function gets a *copy*. Changing the parameter inside the function does **not** change the caller's variable.

```go
package main

import "fmt"

func tryToChange(n int) {
	n = 999
	fmt.Println("inside:", n)
}

func main() {
	x := 5
	tryToChange(x)
	fmt.Println("outside:", x)
}
```

Output:

```
inside: 999
outside: 5
```

`x` is untouched: the function only changed its own copy. (To modify the original you use *pointers*, Chapter 24.)

---

## 7. What happens in memory

This section explains *why* the copy behavior above exists. It builds the mental model you'll rely on for scope (Ch. 8), closures (Ch. 20), pointers (Ch. 24), and goroutines (Ch. 36).

Every time a function is **called**, the computer reserves a private chunk of memory for it, called a **stack frame**. It holds that call's parameters and local variables. When the function **returns**, the frame is released.

Program:

```go
package main

import "fmt"

func add(number1 int, number2 int) {
	sum := number1 + number2
	fmt.Println(sum)
}

func main() {
	a := 10
	b := 20
	add(a, b)
}
```

Step by step:

```
Step 1: main() starts and creates a := 10, b := 20
┌─────────────────────┐
│ main's frame        │
│  a = 10   b = 20    │
└─────────────────────┘

Step 2: add(a, b) is called → a NEW frame is placed on top,
        and the argument VALUES are COPIED into the parameters
┌─────────────────────┐
│ add's frame         │  ← top of stack (currently running)
│  number1 = 10       │
│  number2 = 20       │
├─────────────────────┤
│ main's frame        │
│  a = 10   b = 20    │
└─────────────────────┘

Step 3: inside add: sum := number1 + number2
┌─────────────────────┐
│ add's frame         │
│  number1 = 10       │
│  number2 = 20       │
│  sum = 30           │
├─────────────────────┤
│ main's frame        │
└─────────────────────┘
        prints 30

Step 4: add returns → its frame is destroyed
┌─────────────────────┐
│ main's frame        │
│  a = 10   b = 20    │
└─────────────────────┘

Step 5: main finishes → program ends → everything is released
```

Key ideas:

1. **Frames stack up** (last in, first out), which is why this region of memory is called **the stack**.
2. **Parameters are copies** (that's why the caller's variables don't change).
3. **Local variables die with the function.** `sum` doesn't exist after `add` returns. "When a function has no more work, its memory disappears."

> Chapter 18 opens this up further (stack vs. heap, and how Go decides where variables live).

---

## 8. Functions calling functions

Functions can call other functions, which can call others. The stack just gets taller:

```go
package main

import "fmt"

func double(n int) {
	fmt.Println("double:", n*2)
}

func doubleBoth(a, b int) {
	double(a)
	double(b)
}

func main() {
	doubleBoth(3, 4)
}
```

Output:

```
double: 6
double: 8
```

```
main → doubleBoth → double(3) ✓ returns
                  → double(4) ✓ returns
     ← returns
```

---

## 9. Order of definition doesn't matter

Unlike variables, function order at package level is flexible. `main` may appear before or after the functions it calls:

```go
package main

import "fmt"

func main() {
	greet("Asha") // works even though greet is defined below
}

func greet(name string) {
	fmt.Println("Hello,", name)
}
```

Output: `Hello, Asha`

---

## 10. Practical examples

### Example 1: Greeting with a parameter

```go
package main

import "fmt"

func greet(name string) {
	fmt.Println("Hello,", name+"!")
}

func main() {
	greet("Asha")
	greet("Rahim")
}
```

### Example 2: Rectangle area

```go
package main

import "fmt"

func printArea(width, height float64) {
	fmt.Printf("Area of %.1f x %.1f = %.2f\n", width, height, width*height)
}

func main() {
	printArea(3, 4.5)
	printArea(10, 2)
}
```

Output:

```
Area of 3.0 x 4.5 = 13.50
Area of 10.0 x 2.0 = 20.00
```

### Example 3: Even/odd reporter

```go
package main

import "fmt"

func checkParity(n int) {
	if n%2 == 0 {
		fmt.Println(n, "is even")
	} else {
		fmt.Println(n, "is odd")
	}
}

func main() {
	for i := 1; i <= 4; i++ {
		checkParity(i)
	}
}
```

### Example 4: A banner

```go
package main

import (
	"fmt"
	"strings"
)

func banner(title string) {
	line := strings.Repeat("=", len(title)+4)
	fmt.Println(line)
	fmt.Println("| " + title + " |")
	fmt.Println(line)
}

func main() {
	banner("Go Rocks")
}
```

Output:

```
============
| Go Rocks |
============
```

---

## 11. Common mistakes

| # | Mistake | Example | Fix |
|---|---------|---------|-----|
| 1 | Forgetting `()` when calling | `sayHello` | `sayHello()` |
| 2 | Wrong number of arguments | `add(1)` → *not enough arguments in call to add* | `add(1, 2)` |
| 3 | Wrong argument type | `greet(42)` → *cannot use 42 as string value* | `greet("42")` |
| 4 | Parameter order confusion | `func f(name string, age int)` called as `f(30, "Asha")` | Match the order |
| 5 | Expecting the function to change the caller's variable | `tryToChange(x)` | Return a value (Ch. 5) or use a pointer (Ch. 24) |
| 6 | Using a function's local variable outside it | `sum` after `add()` | It doesn't exist there (scope, Ch. 8) |
| 7 | `func` with `{` on the next line | | Same line: `func f() {` |
| 8 | Defining a function inside a function with `func name()` | | Not allowed. Use an anonymous function (Ch. 15) |
| 9 | Two functions with the same name in one package | | Names must be unique per package |

Example of #2 and #3 in action:

```go
// INTENTIONAL ERROR
package main

import "fmt"

func greet(name string) {
	fmt.Println("Hello,", name)
}

func main() {
	greet()     // not enough arguments in call to greet
	greet(42)   // cannot use 42 (untyped int constant) as string value
}
```

---

## 12. Exercises

### Exercise 1: Say hello 3 times
Write `sayHello()` that prints "Hello!" and call it three times from `main`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func sayHello() {
	fmt.Println("Hello!")
}

func main() {
	sayHello()
	sayHello()
	sayHello()
}
```
</details>

### Exercise 2: Multiply
Write `multiply(a, b int)` that prints the product. Call it with (3, 4) and (10, 10).

<details><summary>Solution</summary>

```go
package main

import "fmt"

func multiply(a, b int) {
	fmt.Println(a * b)
}

func main() {
	multiply(3, 4)
	multiply(10, 10)
}
```
</details>

### Exercise 3: Draw a line
Write `drawLine(length int)` that prints `length` dashes on one line. (Hint: loop with `fmt.Print("-")`, then `fmt.Println()`.)

<details><summary>Solution</summary>

```go
package main

import "fmt"

func drawLine(length int) {
	for i := 0; i < length; i++ {
		fmt.Print("-")
	}
	fmt.Println()
}

func main() {
	drawLine(10)
	drawLine(3)
}
```
</details>

### Exercise 4: Trace the memory
For this program, draw the stack when `inner` is running. What are the frames, from top to bottom?

```go
package main

import "fmt"

func inner(x int) {
	y := x + 1
	fmt.Println(y)
}

func outer(a int) {
	inner(a * 2)
}

func main() {
	outer(5)
}
```

<details><summary>Solution</summary>

```
┌────────────────────┐
│ inner: x=10, y=11  │  ← top (running)
├────────────────────┤
│ outer: a=5         │
├────────────────────┤
│ main               │
└────────────────────┘
```
Prints `11`. When `inner` returns its frame vanishes, then `outer`'s, then `main`'s.
</details>

### Exercise 5 (challenge): Predict the output

```go
package main

import "fmt"

func bump(n int) {
	n++
	fmt.Println("in bump:", n)
}

func main() {
	n := 1
	bump(n)
	bump(n)
	fmt.Println("in main:", n)
}
```

<details><summary>Solution</summary>

```
in bump: 2
in bump: 2
in main: 1
```
Each call gets a fresh copy of `n` (which is `1`), and `main`'s `n` is never modified.
</details>

---

## 13. Quiz

1. What's the difference between *defining* and *calling* a function?
2. In `func f(a int)`, is `a` a parameter or an argument?
3. What is a stack frame?
4. What happens to a function's local variables when it returns?
5. Can `main` be defined after the functions it calls?

<details><summary>Answers</summary>

1. Defining says what it does; calling actually runs it.
2. A parameter.
3. The private memory block holding one function call's parameters and local variables.
4. They are destroyed (the frame is released).
5. Yes. Order of top-level functions doesn't matter.
</details>

---

## 14. Summary

- A **function** is a named, reusable block: `func name(params) { body }`.
- **Call** it with `name(args)`. Execution jumps in, then returns to the next line.
- **Parameters** are placeholders in the definition; **arguments** are real values in the call.
- Arguments are **copied** into parameters (pass by value).
- Each call gets its own **stack frame**; it's destroyed on return, so **locals vanish**.
- Functions make code reusable, organized, and maintainable.

### ➡️ What's next?

So far our functions only *print*. In [Chapter 5](05-functions-with-return-values.md) they'll **return values** to their callers, unlocking real power.
