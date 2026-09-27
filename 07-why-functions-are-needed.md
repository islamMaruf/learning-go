# Chapter 7: Why Functions Are Needed — A Real-World Refactor

> **Goal of this chapter:** See, with a concrete program, *why* professionals split code into small functions. You'll read user input, watch a messy program grow, and then clean it up using the **Single Responsibility Principle**.

**Difficulty:** 🟢 Beginner  **Estimated time:** 1.5 hours  **Prerequisite:** [Chapter 6](06-more-function-examples.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The story: too many jobs at once](#2-the-story-too-many-jobs-at-once)
3. [Reading user input with `fmt.Scanln`](#3-reading-user-input)
4. [Version 1: everything inside `main`](#4-version-1-everything-inside-main)
5. [What's wrong with Version 1?](#5-whats-wrong-with-version-1)
6. [The Single Responsibility Principle (SRP)](#6-the-single-responsibility-principle)
7. [Version 2: refactoring into functions](#7-version-2-refactoring-into-functions)
8. [Benefits, compared side by side](#8-benefits-of-the-refactor)
9. [Going further: input with spaces and validation](#9-going-further)
10. [How to decide where to split](#10-how-to-decide-where-to-split)
11. [Common mistakes](#11-common-mistakes)
12. [Exercises](#12-exercises)
13. [Quiz](#13-quiz)
14. [Summary](#14-summary)

---

## 1. What you will learn

- How to read text and numbers typed by the user
- Why a giant `main` function becomes a nightmare
- What the **Single Responsibility Principle** means (the "S" in SOLID)
- How to refactor working code into clean, well-named functions **without changing what it does**
- How `main` can become a readable "table of contents" for your program

---

## 2. The story: too many jobs at once

Imagine one person trying to:

- study for exams,
- work a part-time job,
- run a side business,
- organize every family event,
- and learn to code,

all in the same hour, mixing everything up. They'd forget things and do everything badly.

Now imagine a *team* where each person has **one clear job**. Work is faster, mistakes are easier to trace, and people can be replaced or helped without chaos.

**Functions are your team members.** `main` is the manager: it doesn't do every job, it just calls the right specialist at the right time.

---

## 3. Reading user input

To make the example interactive we need input. The simplest tool is `fmt.Scanln`.

```go
package main

import "fmt"

func main() {
	var name string
	fmt.Println("Enter your name:")
	fmt.Scanln(&name)
	fmt.Println("Hello,", name)
}
```

Run it: the program **pauses** and waits until you type something and press **Enter**.

```
Enter your name:
Asha            ← you typed this
Hello, Asha
```

### What does `&name` mean?

`Scanln` needs to **put a value into your variable**. But (remember Chapter 4) functions receive *copies* of arguments, so passing `name` would only let `Scanln` change a copy.

So we give it the variable's **address** (its location in memory) using `&`:

```
memory:   name  (address 0xC000012345)
          ┌────┐
          │ "" │   ← Scanln receives the ADDRESS, walks there,
          └────┘     and writes "Asha" into the real variable.
```

> `&x` means "**the address of** x". Pointers get a full chapter ([Chapter 24](24-pointers.md)). For now just remember: **`Scanln` needs `&`**.

### Reading numbers

```go
package main

import "fmt"

func main() {
	var n int
	fmt.Println("Enter a number:")
	fmt.Scanln(&n)
	fmt.Println("Double is", n*2)
}
```

`Scanln` looks at the *type* of the variable (`int`) and converts the typed text to a number for you. If the user types `abc`, the conversion fails. We'll handle that in section 9.

### Two limits of `Scanln` (good to know)

1. It stops reading a string at the first **space** (typing `Asha Rahman` gives you just `Asha`).
2. It reports failure through a returned `error` (which beginners often ignore).

Section 9 shows a sturdier way. For now, `Scanln` keeps the example simple.

---

## 4. Version 1: everything inside `main`

**The application:**

1. Welcome the user
2. Ask for their name
3. Ask for two numbers
4. Add them
5. Display the results
6. Say goodbye

```go
package main

import "fmt"

func main() {
	// Print welcome message
	fmt.Println("Welcome to the application")

	// Get user name as input
	var name string
	fmt.Println("Enter your name:")
	fmt.Scanln(&name)

	// Get two numbers
	var number1 int
	var number2 int

	fmt.Println("Enter first number:")
	fmt.Scanln(&number1)

	fmt.Println("Enter second number:")
	fmt.Scanln(&number2)

	// Add numbers
	sum := number1 + number2

	// Display results
	fmt.Println("Hello", name)
	fmt.Println("Summation =", sum)

	// Print goodbye message
	fmt.Println("Thank you for using the application")
	fmt.Println("Good Bye")
}
```

Sample run:

```
Welcome to the application
Enter your name:
Habib
Enter first number:
7
Enter second number:
6
Hello Habib
Summation = 13
Thank you for using the application
Good Bye
```

It **works**. So what's the problem?

---

## 5. What's wrong with Version 1?

The program is 30 lines, so it's still readable. But **real** programs grow. Picture this `main` with 500 lines: logging, validation, saving to a database, sending emails, and cleanup.

| Problem | Why it hurts |
|---------|--------------|
| **Hard to read** | You must read everything to understand anything |
| **Hard to change** | Editing the welcome message means hunting through unrelated code |
| **No reuse** | Need the goodbye text in two places? Copy-paste it (and now you have two to maintain) |
| **Hard to test** | You can't test "add" without also running the prompts and prints |
| **Hard to collaborate** | Two developers editing one giant function cause merge conflicts |
| **Bugs hide** | A variable declared at line 10 might be silently changed at line 300 |

---

## 6. The Single Responsibility Principle

**SOLID** is a famous list of five design principles:

| Letter | Principle | Idea |
|--------|-----------|------|
| **S** | Single Responsibility | A unit of code should have **one job / one reason to change** |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtypes must be usable in place of their base types |
| **I** | Interface Segregation | Prefer small, focused interfaces |
| **D** | Dependency Inversion | Depend on abstractions, not concrete details |

We'll meet O, L, I, and D in Chapters 50–51. Today: **S**.

> **Single Responsibility Principle:** *A function should do one thing, and do it well.*

How to check yourself: **describe the function in one sentence without using "and".**

- ✅ "`add` adds two numbers."
- ✅ "`getUserName` asks the user for their name and returns it."
- ❌ "`main` prints a welcome, asks for numbers **and** adds them **and** prints results **and** says goodbye." (too many jobs, so let's split.)

---

## 7. Version 2: refactoring into functions

**Refactoring** means restructuring code *without changing its behavior*. We take each responsibility and give it a function.

### Step 1: Welcome

```go
func printWelcomeMessage() {
	fmt.Println("Welcome to the application")
}
```
*Job: print the welcome message.*

### Step 2: Get the name

```go
func getUserName() string {
	var name string
	fmt.Println("Enter your name:")
	fmt.Scanln(&name)
	return name
}
```
*Job: ask for a name, return it.*

### Step 3: Get two numbers

```go
func getTwoNumbers() (int, int) {
	var number1, number2 int

	fmt.Println("Enter first number:")
	fmt.Scanln(&number1)

	fmt.Println("Enter second number:")
	fmt.Scanln(&number2)

	return number1, number2
}
```
*Job: ask for two numbers, return both. (Multiple returns, Chapter 5.)*

### Step 4: Add

```go
func add(number1 int, number2 int) int {
	return number1 + number2
}
```
*Job: add.* Notice it does **no printing and no input**: a pure function.

### Step 5: Display

```go
func display(name string, sum int) {
	fmt.Println("Hello", name)
	fmt.Println("Summation =", sum)
}
```
*Job: show results.*

### Step 6: Goodbye

```go
func printGoodbyeMessage() {
	fmt.Println("Thank you for using the application")
	fmt.Println("Good Bye")
}
```

### The finished program

```go
package main

import "fmt"

func printWelcomeMessage() {
	fmt.Println("Welcome to the application")
}

func getUserName() string {
	var name string
	fmt.Println("Enter your name:")
	fmt.Scanln(&name)
	return name
}

func getTwoNumbers() (int, int) {
	var number1, number2 int

	fmt.Println("Enter first number:")
	fmt.Scanln(&number1)

	fmt.Println("Enter second number:")
	fmt.Scanln(&number2)

	return number1, number2
}

func add(number1 int, number2 int) int {
	return number1 + number2
}

func display(name string, sum int) {
	fmt.Println("Hello", name)
	fmt.Println("Summation =", sum)
}

func printGoodbyeMessage() {
	fmt.Println("Thank you for using the application")
	fmt.Println("Good Bye")
}

func main() {
	printWelcomeMessage()
	name := getUserName()
	number1, number2 := getTwoNumbers()
	sum := add(number1, number2)
	display(name, sum)
	printGoodbyeMessage()
}
```

Look at `main`. Even a stranger can read it top to bottom and understand the whole program in ten seconds. 🎉

### How data flows

```
printWelcomeMessage()
        │
getUserName() ──────────────► name ─────────────────┐
        │                                            ▼
getTwoNumbers() ─► number1, number2 ─► add() ─► sum ─► display(name, sum)
                                                            │
                                                  printGoodbyeMessage()
```

Each function receives what it needs through **parameters** and passes results on through **return values**. They don't secretly share variables.

---

## 8. Benefits of the refactor

| Benefit | Before (Version 1) | After (Version 2) |
|---------|--------------------|--------------------|
| **Readability** | Read 30 lines of mixed details | `main` reads like a story |
| **Reuse** | Copy-paste the goodbye text | Call `printGoodbyeMessage()` anywhere |
| **Change** | Hunt through code | Edit one function |
| **Testing** | Impossible without typing input | `add(5, 7)` is trivially testable |
| **Teamwork** | One big function = conflicts | Each person owns functions |
| **Bug isolation** | Bug could be anywhere | Wrong sum? Look in `add` |

### Example: testing `add` on its own

```go
package main

import "fmt"

func add(a, b int) int { return a + b }

func main() {
	if add(5, 7) != 12 {
		fmt.Println("BUG: add is broken")
	} else {
		fmt.Println("add works")
	}
}
```

Try writing that check for Version 1: you'd have to run the whole program and type input each time. (Chapter 44+ mention `go test`, the professional way.)

### Example: changing behavior in one place

Suppose the boss wants a friendlier greeting. In Version 2 you edit **one** function:

```go
func printWelcomeMessage() {
	fmt.Println("🎉 Welcome to our amazing app! 🎉")
}
```

Every caller gets the change.

---

## 9. Going further

### 9.1 Reading a full line (names with spaces)

Use `bufio.Reader` and `strings.TrimSpace`:

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"
)

func readLine(prompt string) string {
	fmt.Print(prompt)
	reader := bufio.NewReader(os.Stdin)
	text, _ := reader.ReadString('\n')
	return strings.TrimSpace(text)
}

func main() {
	name := readLine("Full name: ")
	fmt.Println("Hello,", name)
}
```

- `os.Stdin` is "the keyboard input".
- `ReadString('\n')` reads up to and including the Enter key.
- `TrimSpace` strips the trailing newline.

> ℹ️ A new `bufio.Reader` per call can swallow buffered input if you call `readLine` repeatedly on piped input. A better design keeps one reader in a variable and passes it around. We keep it simple here.

### 9.2 Validating numbers

`Scanln` returns `(n int, err error)`. Check the error:

```go
package main

import "fmt"

func getNumber(prompt string) int {
	for {
		var n int
		fmt.Print(prompt)
		_, err := fmt.Scanln(&n)
		if err == nil {
			return n
		}
		fmt.Println("That's not a whole number. Try again.")
		var discard string
		fmt.Scanln(&discard) // clear the bad input
	}
}

func main() {
	a := getNumber("First number: ")
	b := getNumber("Second number: ")
	fmt.Println("Sum:", a+b)
}
```

This is another payoff of functions: the retry logic lives in **one** place, and we call `getNumber` twice.

### 9.3 Return an error instead of looping forever

For code that might be reused (in a web server, say) it's better to *return* errors than to loop on the keyboard. The rule of thumb from Chapter 6 still holds: **compute in functions, do input/output at the edges.**

---

## 10. How to decide where to split

Ask these questions about a block of code:

1. **Can I give it a clear verb-phrase name?** (`validateEmail`, `calculateTax`.) If yes, it's a function candidate.
2. **Does it appear (or might it appear) more than once?**
3. **Does it have a clear input and output?**
4. **Does the function mix *levels* of work?** (High-level "run the checkout" beside low-level "add two numbers".) Split them.
5. **Would I want to test it alone?**

Don't over-split. A one-line function called once, whose body is clearer than its name, is noise. Aim for **clarity**, not maximum function count.

**Rule of thumb for `main`:** it should read like an outline: *what* happens, not *how*.

---

## 11. Common mistakes

| # | Mistake | Fix |
|---|---------|-----|
| 1 | Forgetting `&` in `Scanln(&x)` | `Scanln(x)` won't set `x`. `go vet` warns. Always pass `&x` |
| 2 | Expecting `Scanln` to read `"Asha Rahman"` into one string | It stops at spaces. Use `bufio` (section 9.1) |
| 3 | A function that prompts, calculates, *and* prints | Split into `get…`, calculate, `display…` |
| 4 | Passing lots of data via global variables | Use parameters and return values |
| 5 | Function names that lie (`add` that also prints) | Name what it does, or do only what the name says |
| 6 | Ignoring `Scanln`'s error | Check it (section 9.2) |
| 7 | Over-splitting into one-line functions no one needs | Keep code together when it's clearer |

---

## 12. Exercises

### Exercise 1: Refactor a messy program
Split this into three functions (`readAge`, `isAdult`, `report`) and keep `main` short.

```go
package main

import "fmt"

func main() {
	var age int
	fmt.Println("Enter age:")
	fmt.Scanln(&age)
	adult := age >= 18
	if adult {
		fmt.Println("Adult")
	} else {
		fmt.Println("Minor")
	}
}
```

<details><summary>Solution</summary>

```go
package main

import "fmt"

func readAge() int {
	var age int
	fmt.Println("Enter age:")
	fmt.Scanln(&age)
	return age
}

func isAdult(age int) bool {
	return age >= 18
}

func report(adult bool) {
	if adult {
		fmt.Println("Adult")
	} else {
		fmt.Println("Minor")
	}
}

func main() {
	report(isAdult(readAge()))
}
```
</details>

### Exercise 2: Rectangle program
Ask for width and height, compute area and perimeter with separate functions, print results with a third.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func readFloat(prompt string) float64 {
	var v float64
	fmt.Print(prompt)
	fmt.Scanln(&v)
	return v
}

func area(w, h float64) float64      { return w * h }
func perimeter(w, h float64) float64 { return 2 * (w + h) }

func show(a, p float64) {
	fmt.Printf("Area: %.2f\nPerimeter: %.2f\n", a, p)
}

func main() {
	w := readFloat("Width: ")
	h := readFloat("Height: ")
	show(area(w, h), perimeter(w, h))
}
```
</details>

### Exercise 3: Describe in one sentence
For each function name, write a one-sentence job description *without* "and". Which of these names hints at a violation of SRP?

`validateAndSaveUser`, `calculateTotal`, `printReceipt`, `loadConfigAndConnect`

<details><summary>Solution</summary>

`validateAndSaveUser` and `loadConfigAndConnect` contain "And" in the name, a strong smell. Split them: `validateUser` + `saveUser`, and `loadConfig` + `connect`. The other two are single-purpose.
</details>

### Exercise 4: Menu-driven calculator (challenge)
Ask for two numbers and an operator (`+ - * /`). Use one function to read input, one to calculate (returning `(float64, error)`), and one to print. Handle divide by zero.

<details><summary>Solution</summary>

```go
package main

import (
	"errors"
	"fmt"
)

func readInputs() (float64, float64, string) {
	var a, b float64
	var op string
	fmt.Print("First number: ")
	fmt.Scanln(&a)
	fmt.Print("Operator (+ - * /): ")
	fmt.Scanln(&op)
	fmt.Print("Second number: ")
	fmt.Scanln(&b)
	return a, b, op
}

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
			return 0, errors.New("cannot divide by zero")
		}
		return a / b, nil
	}
	return 0, fmt.Errorf("unknown operator %q", op)
}

func show(result float64, err error) {
	if err != nil {
		fmt.Println("Error:", err)
		return
	}
	fmt.Println("Result:", result)
}

func main() {
	a, b, op := readInputs()
	show(calculate(a, b, op))
}
```

Note `show(calculate(a, b, op))`: Go lets you pass a multi-value call directly as the arguments of another function whose parameters match.
</details>

---

## 13. Quiz

1. What does the Single Responsibility Principle say?
2. Why does `Scanln` need `&`?
3. What is refactoring?
4. Give two benefits of small functions.
5. What should `main` look like in a well-structured program?

<details><summary>Answers</summary>

1. Each function (or unit) should have one job / one reason to change.
2. It must modify *your* variable, so it needs the variable's address, not a copy.
3. Restructuring code without changing what it does.
4. e.g., reuse, readability, testability, easier maintenance.
5. A short outline that calls other functions.
</details>

---

## 14. Summary

- Real programs grow, so **structure matters**. A monolithic `main` becomes unreadable, untestable, and fragile.
- The **Single Responsibility Principle**: one function, one job (describe it without "and").
- **Refactoring** improves structure without changing behavior.
- Functions communicate through **parameters** and **return values**.
- `fmt.Scanln(&x)` reads input; `&` gives it the variable's address (pointers are coming in Chapter 24).
- Keep `main` an outline; push details into well-named functions.

### ➡️ What's next?

You've seen that variables exist "inside" functions. But *where exactly* can each variable be used? [Chapter 8](08-what-is-scope.md) introduces **scope**, the most important concept of this part of the course.
