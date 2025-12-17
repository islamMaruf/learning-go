# Go Programming Tutorial - Chapter 4
## Introduction to Functions

### 📚 Table of Contents
1. [Introduction](#introduction)
2. [What is a Function?](#what-is-a-function)
3. [Why Do We Need Functions?](#why-do-we-need-functions)
4. [Function Syntax](#function-syntax)
5. [How Functions Work in Memory](#how-functions-work-in-memory)
6. [Function Parameters](#function-parameters)
7. [Calling Functions](#calling-functions)
8. [Multiple Function Calls](#multiple-function-calls)
9. [Function Scope](#function-scope)
10. [Practical Examples](#practical-examples)
11. [Common Mistakes](#common-mistakes)
12. [Summary](#summary)

---

## Introduction

Functions are one of the most important concepts in programming. They help you organize code, avoid repetition, and make your programs more maintainable.

**What You'll Learn:**
- What functions are and why they matter
- How to create and call functions
- How functions work in computer memory (RAM)
- Function parameters and arguments
- Function scope and lifetime
- Memory management with functions

---

## What is a Function?

A **function** is a **reusable block of code** that performs a specific task.

**Analogy:** Think of a function like a recipe:
- **Recipe Name:** Function name
- **Ingredients:** Function parameters (inputs)
- **Instructions:** Function body (code)
- **Dish:** Result of the function

### Without Functions (Repetitive Code)

```go
package main

import "fmt"

func main() {
    // Adding numbers - First time
    a := 10
    b := 20
    sum := a + b
    fmt.Println(sum)  // 30
    
    // Adding numbers - Second time (same code repeated!)
    x := 5
    y := 7
    sum2 := x + y
    fmt.Println(sum2)  // 12
    
    // Adding numbers - Third time (repetition again!)
    p := 100
    q := 200
    sum3 := p + q
    fmt.Println(sum3)  // 300
}
```

**Problem:** We're writing the same logic multiple times! 😫

---

### With Functions (Reusable Code)

```go
package main

import "fmt"

func add(number1 int, number2 int) {
    sum := number1 + number2
    fmt.Println(sum)
}

func main() {
    add(10, 20)    // 30
    add(5, 7)      // 12
    add(100, 200)  // 300
}
```

**Solution:** Write the logic once, use it many times! ✅

---

## Why Do We Need Functions?

### 1. Code Reusability

Write once, use multiple times!

```go
add(10, 20)
add(5, 7)
add(100, 200)
// Same function, different values
```

---

### 2. Organization

Break large programs into smaller, manageable pieces.

```go
func login() { /* login logic */ }
func logout() { /* logout logic */ }
func sendEmail() { /* email logic */ }
func calculatePrice() { /* price logic */ }
```

---

### 3. Maintainability

Fix bugs in one place, not everywhere!

```go
// If there's a bug in addition logic, fix it once:
func add(number1 int, number2 int) {
    // Fix the bug here, all calls benefit!
    sum := number1 + number2
    fmt.Println(sum)
}
```

---

### 4. Abstraction

Hide complex details behind simple names.

```go
sendEmail(to, subject, body)  // Simple to use!
// Complex email logic hidden inside the function
```

---

## Function Syntax

### Basic Structure

```go
func functionName(parameter1 type1, parameter2 type2) {
    // Function body
    // Code to execute
}
```

### Breaking Down Each Part

```go
func add(number1 int, number2 int) {
    sum := number1 + number2
    fmt.Println(sum)
}
```

| Part | Explanation | Example |
|------|-------------|---------|
| `func` | Keyword to declare a function | `func` |
| `add` | Function name (you choose this) | `add` |
| `(...)` | Parentheses for parameters | `(number1 int, number2 int)` |
| `number1 int` | First parameter with type | `number1` is `int` |
| `number2 int` | Second parameter with type | `number2` is `int` |
| `{...}` | Curly braces contain the function body | Code inside `{}` |

---

### Why "func" Instead of "function"?

**Remember:** Programmers are cute! 😊 We love short names:
- **function** → **func** (shorter and easier to type)
- **variable** → **var** (same reason)
- **integer** → **int** (you get the idea!)

---

## How Functions Work in Memory

Let's understand how functions work step-by-step with memory (RAM).

### Simple Example Without Functions

```go
package main

import "fmt"

func main() {
    a := 10
    b := 20
    sum := a + b
    fmt.Println(sum)  // 30
}
```

**Memory (RAM) State:**

```
Step 1: a := 10
┌─────┐
│  a  │
│ 10  │
└─────┘

Step 2: b := 20
┌─────┬─────┐
│  a  │  b  │
│ 10  │ 20  │
└─────┴─────┘

Step 3: sum := a + b
┌─────┬─────┬─────┐
│  a  │  b  │ sum │
│ 10  │ 20  │ 30  │
└─────┴─────┴─────┘

Step 4: Print sum
Output: 30
```

---

### Same Example WITH Functions

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
    add(a, b)  // Call the function
}
```

**Memory (RAM) State - Detailed:**

```
Initial State:
┌────────────────────────────────┐
│         RAM                     │
│  (Empty cells)                  │
└────────────────────────────────┘

Step 1: main() starts
a := 10
┌──────────────────┐
│ Main Function    │
├─────┐            │
│  a  │            │
│ 10  │            │
└─────┴────────────┘

Step 2: b := 20
┌──────────────────┐
│ Main Function    │
├─────┬─────┐      │
│  a  │  b  │      │
│ 10  │ 20  │      │
└─────┴─────┴──────┘

Step 3: add(a, b) - Function called!
┌──────────────────┬──────────────────────┐
│ Main Function    │ add() Function       │
├─────┬─────┐      ├────────────┬─────────┤
│  a  │  b  │      │ number1    │ number2 │
│ 10  │ 20  │      │ 10 (from a)│ 20 (b)  │
└─────┴─────┘      └────────────┴─────────┘

Step 4: Inside add() - sum := number1 + number2
┌──────────────────┬──────────────────────────────┐
│ Main Function    │ add() Function               │
├─────┬─────┐      ├────────────┬─────────┬──────┤
│  a  │  b  │      │ number1    │ number2 │ sum  │
│ 10  │ 20  │      │ 10         │ 20      │ 30   │
└─────┴─────┘      └────────────┴─────────┴──────┘

Step 5: Print sum
Output: 30

Step 6: add() finishes - Memory cleaned up!
┌──────────────────┐
│ Main Function    │  ← add() memory is freed!
├─────┬─────┐      │
│  a  │  b  │      │
│ 10  │ 20  │      │
└─────┴─────┴──────┘

Step 7: main() finishes - Program ends
(All memory freed)
```

**Key Point:** When a function finishes executing, its memory is freed! It's like "dying" - when there's no work left, the function disappears from memory.

---

### The Philosophy of Function Memory

> **"In this world, when someone has nothing left to do, they disappear."**

When a function completes its task:
1. It returns control to the caller
2. Its variables are cleaned up
3. Its memory is freed
4. It "dies" - but can be called again later!

This is called **function scope** and **automatic memory management**.

---

## Function Parameters

### What are Parameters?

**Parameters** are the variables that a function accepts as input.

```go
func add(number1 int, number2 int) {
    //      ↑          ↑
    //   parameter1  parameter2
}
```

### Parameter vs Argument

```go
// Parameters: Variables in function definition
func add(number1 int, number2 int) {
    //   ^^^^^^^^      ^^^^^^^^
    //   Parameters (placeholders)
}

func main() {
    // Arguments: Actual values passed when calling
    add(10, 20)
    //  ^^  ^^
    //  Arguments (actual values)
}
```

**Easy to remember:**
- **Parameters** = **Placeholders** (in function definition)
- **Arguments** = **Actual values** (when calling function)

---

### Parameter Types Must Match

```go
func add(number1 int, number2 int) {
    sum := number1 + number2
    fmt.Println(sum)
}

func main() {
    a := 10  // int type
    b := 20  // int type
    
    add(a, b)  // ✅ Both are int - works!
    
    // add("hello", b)  // ❌ ERROR! String is not int
    // add(3.14, b)     // ❌ ERROR! Float is not int
    // add(true, b)     // ❌ ERROR! Bool is not int
}
```

**Rule:** The type of arguments must match the type of parameters!

---

### Multiple Parameters

```go
func greet(name string, age int) {
    fmt.Println("Name:", name)
    fmt.Println("Age:", age)
}

func main() {
    greet("Habib", 25)
}
```

**Output:**
```
Name: Habib
Age: 25
```

---

## Calling Functions

### Basic Function Call

```go
package main

import "fmt"

func add(number1 int, number2 int) {
    sum := number1 + number2
    fmt.Println(sum)
}

func main() {
    add(10, 20)  // Call the function
}
```

**Output:**
```
30
```

---

### Calling with Variables

```go
func main() {
    a := 10
    b := 20
    add(a, b)  // Pass variables
}
```

**What happens:**
1. `a` has value 10
2. `b` has value 20
3. Function receives: `number1 = 10`, `number2 = 20`
4. Function adds them: `10 + 20 = 30`
5. Function prints: `30`

---

### Calling with Literal Values

```go
func main() {
    add(5, 7)      // Direct values
    add(100, 200)  // Different values
}
```

**Output:**
```
12
300
```

---

## Multiple Function Calls

You can call the same function multiple times with different arguments!

### Example: Adding Different Numbers

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
    
    add(a, b)     // First call: 10 + 20 = 30
    add(5, 7)     // Second call: 5 + 7 = 12
    add(100, 200) // Third call: 100 + 200 = 300
}
```

**Output:**
```
30
12
300
```

---

### Memory During Multiple Calls

```
Call 1: add(10, 20)
┌──────────────────┬────────────────────┐
│ Main             │ add() - Call 1     │
├─────┬─────┐      ├─────────┬──────────┤
│ a=10│ b=20│      │number1=10│number2=20│
└─────┴─────┘      │ sum=30   │         │
                   └──────────┴──────────┘
Print: 30
add() finishes → Memory freed ✅

Call 2: add(5, 7)
┌──────────────────┬────────────────────┐
│ Main             │ add() - Call 2     │
├─────┬─────┐      ├─────────┬──────────┤
│ a=10│ b=20│      │number1=5 │number2=7 │
└─────┴─────┘      │ sum=12   │         │
                   └──────────┴──────────┘
Print: 12
add() finishes → Memory freed ✅

Call 3: add(100, 200)
┌──────────────────┬────────────────────────┐
│ Main             │ add() - Call 3         │
├─────┬─────┐      ├──────────┬─────────────┤
│ a=10│ b=20│      │number1=100│number2=200│
└─────┴─────┘      │ sum=300   │           │
                   └───────────┴─────────────┘
Print: 300
add() finishes → Memory freed ✅

main() finishes → All memory freed ✅
```

**Key Points:**
1. Each function call gets its own memory space
2. After each call completes, its memory is freed
3. Variables `a` and `b` in `main()` stay alive until `main()` ends
4. Function can be called as many times as needed!

---

## Function Scope

### What is Scope?

**Scope** determines where a variable can be accessed.

### Local Variables

Variables declared inside a function are **local** to that function.

```go
package main

import "fmt"

func add(number1 int, number2 int) {
    sum := number1 + number2  // 'sum' is LOCAL to add()
    fmt.Println(sum)
}

func main() {
    a := 10
    b := 20
    add(a, b)
    
    // fmt.Println(sum)  // ❌ ERROR! 'sum' doesn't exist here
    // fmt.Println(number1)  // ❌ ERROR! 'number1' doesn't exist here
}
```

**Why?** Variables inside `add()` only exist while `add()` is running!

---

### Variable Lifetime

```go
func add(number1 int, number2 int) {
    sum := number1 + number2
    // sum is BORN here ↑
    
    fmt.Println(sum)
    
    // sum DIES here when function ends ↓
}
```

**Lifetime:**
- **Born:** When variable is declared
- **Lives:** While function is executing
- **Dies:** When function returns

---

### Separate Memory Spaces

```go
package main

import "fmt"

func add(number1 int, number2 int) {
    sum := number1 + number2
    fmt.Println("Inside add, sum =", sum)
}

func main() {
    sum := 999  // Different 'sum' than the one in add()!
    
    add(10, 20)
    
    fmt.Println("Inside main, sum =", sum)  // Still 999!
}
```

**Output:**
```
Inside add, sum = 30
Inside main, sum = 999
```

**Why?** Each function has its own separate memory space!

---

## Practical Examples

### Example 1: Greeting Function

```go
package main

import "fmt"

func greet(name string) {
    fmt.Println("Hello,", name + "!")
}

func main() {
    greet("Habib")
    greet("Ahmed")
    greet("Sarah")
}
```

**Output:**
```
Hello, Habib!
Hello, Ahmed!
Hello, Sarah!
```

---

### Example 2: Multiple Operations

```go
package main

import "fmt"

func add(a int, b int) {
    fmt.Println("Sum:", a + b)
}

func subtract(a int, b int) {
    fmt.Println("Difference:", a - b)
}

func multiply(a int, b int) {
    fmt.Println("Product:", a * b)
}

func main() {
    x := 10
    y := 5
    
    add(x, y)
    subtract(x, y)
    multiply(x, y)
}
```

**Output:**
```
Sum: 15
Difference: 5
Product: 50
```

---

### Example 3: User Info

```go
package main

import "fmt"

func printUserInfo(name string, age int, country string) {
    fmt.Println("Name:", name)
    fmt.Println("Age:", age)
    fmt.Println("Country:", country)
    fmt.Println("---")
}

func main() {
    printUserInfo("Habib", 25, "Bangladesh")
    printUserInfo("Ahmed", 30, "Pakistan")
}
```

**Output:**
```
Name: Habib
Age: 25
Country: Bangladesh
---
Name: Ahmed
Age: 30
Country: Pakistan
---
```

---

### Example 4: Calculation Function

```go
package main

import "fmt"

func calculateArea(length int, width int) {
    area := length * width
    fmt.Println("Area:", area)
}

func main() {
    calculateArea(10, 5)   // Rectangle 1
    calculateArea(20, 15)  // Rectangle 2
    calculateArea(7, 8)    // Rectangle 3
}
```

**Output:**
```
Area: 50
Area: 300
Area: 56
```

---

### Example 5: Temperature Converter

```go
package main

import "fmt"

func celsiusToFahrenheit(celsius float32) {
    fahrenheit := (celsius * 9/5) + 32
    fmt.Printf("%.2f°C = %.2f°F\n", celsius, fahrenheit)
}

func main() {
    celsiusToFahrenheit(0)
    celsiusToFahrenheit(25)
    celsiusToFahrenheit(100)
}
```

**Output:**
```
0.00°C = 32.00°F
25.00°C = 77.00°F
100.00°C = 212.00°F
```

---

## Common Mistakes

### ❌ Mistake 1: Wrong Number of Arguments

```go
func add(number1 int, number2 int) {
    sum := number1 + number2
    fmt.Println(sum)
}

func main() {
    add(10)          // ❌ ERROR! Missing second argument
    add(10, 20, 30)  // ❌ ERROR! Too many arguments
    
    add(10, 20)      // ✅ CORRECT
}
```

**Rule:** Must provide exactly the number of arguments the function expects!

---

### ❌ Mistake 2: Wrong Type of Arguments

```go
func add(number1 int, number2 int) {
    sum := number1 + number2
    fmt.Println(sum)
}

func main() {
    add("10", "20")     // ❌ ERROR! Strings, not ints
    add(10.5, 20.5)     // ❌ ERROR! Floats, not ints
    add(true, false)    // ❌ ERROR! Bools, not ints
    
    add(10, 20)         // ✅ CORRECT
}
```

---

### ❌ Mistake 3: Accessing Local Variables Outside Function

```go
func calculate() {
    result := 100
}

func main() {
    calculate()
    fmt.Println(result)  // ❌ ERROR! 'result' doesn't exist here
}
```

**Solution:** Variables inside a function are local to that function!

---

### ❌ Mistake 4: Forgetting to Call the Function

```go
func greet() {
    fmt.Println("Hello!")
}

func main() {
    greet  // ❌ WRONG! This doesn't call the function
    
    greet()  // ✅ CORRECT! Parentheses () are needed to call
}
```

---

### ❌ Mistake 5: Function Name Typo

```go
func add(a int, b int) {
    fmt.Println(a + b)
}

func main() {
    Add(10, 20)  // ❌ ERROR! Wrong case - 'Add' vs 'add'
    
    add(10, 20)  // ✅ CORRECT
}
```

**Remember:** Go is case-sensitive! `add` ≠ `Add`

---

## Execution Flow Diagram

### Visual Flow of Function Call

```
Program Start
     ↓
┌─────────────────┐
│   main()        │
│   starts        │
└────────┬────────┘
         ↓
    a := 10
    b := 20
         ↓
    add(a, b)  ← Function call
         ↓
┌─────────────────────────┐
│   add() function        │
│   number1 = 10          │
│   number2 = 20          │
│   sum = 30              │
│   Print: 30             │
│   Return to main        │
└────────┬────────────────┘
         ↓
┌─────────────────┐
│   main()        │
│   continues     │
│   (if more code)│
└────────┬────────┘
         ↓
    Program End
```

---

## Summary

### 🎯 Key Takeaways

1. **Functions are Reusable Code Blocks:**
   - Write once, use many times
   - Reduces code repetition
   - Makes code organized and maintainable

2. **Function Syntax:**
   ```go
   func functionName(parameter1 type1, parameter2 type2) {
       // function body
   }
   ```

3. **Calling Functions:**
   ```go
   functionName(argument1, argument2)
   ```

4. **Parameters vs Arguments:**
   - **Parameters:** Variables in function definition (placeholders)
   - **Arguments:** Actual values passed when calling (real values)

5. **Memory Management:**
   - Each function call gets its own memory space
   - Memory is automatically freed when function ends
   - Variables are local to their function

6. **Function Scope:**
   - Variables inside a function only exist while it's running
   - Cannot access function's local variables from outside
   - Each function has separate memory space

7. **Important Rules:**
   - Number of arguments must match number of parameters
   - Types of arguments must match types of parameters
   - Use `()` to call a function
   - Function names are case-sensitive

---

### 📝 Complete Example: All Concepts

```go
package main

import "fmt"

// Function 1: Simple greeting
func greet(name string) {
    fmt.Println("Hello,", name)
}

// Function 2: Addition
func add(a int, b int) {
    sum := a + b
    fmt.Println("Sum:", sum)
}

// Function 3: Multiple parameters
func userInfo(name string, age int, city string) {
    fmt.Println("Name:", name)
    fmt.Println("Age:", age)
    fmt.Println("City:", city)
    fmt.Println("---")
}

func main() {
    // Call greet function
    greet("Habib")
    greet("Ahmed")
    
    // Call add function
    x := 10
    y := 20
    add(x, y)
    add(5, 7)
    
    // Call userInfo function
    userInfo("Habib", 25, "Dhaka")
    userInfo("Sarah", 22, "Chittagong")
}
```

**Output:**
```
Hello, Habib
Hello, Ahmed
Sum: 30
Sum: 12
Name: Habib
Age: 25
City: Dhaka
---
Name: Sarah
Age: 22
City: Chittagong
---
```

---

## Practice Exercises

### Exercise 1: Create a Multiply Function
Write a function that multiplies two numbers and prints the result.

```go
func multiply(a int, b int) {
    // Your code here
}

func main() {
    multiply(5, 6)   // Should print: 30
    multiply(10, 3)  // Should print: 30
}
```

---

### Exercise 2: Full Name Function
Create a function that takes first name and last name, then prints the full name.

```go
func printFullName(firstName string, lastName string) {
    // Your code here
}

func main() {
    printFullName("Habib", "Ahmed")  // Should print: Habib Ahmed
}
```

---

### Exercise 3: Age Calculator
Create a function that takes birth year and calculates age (assume current year is 2025).

```go
func calculateAge(birthYear int) {
    // Your code here
    // Hint: age = 2025 - birthYear
}

func main() {
    calculateAge(2000)  // Should print: Age: 25
    calculateAge(1995)  // Should print: Age: 30
}
```

---

### Exercise 4: Circle Area
Create a function that calculates the area of a circle given the radius.
Formula: area = π × radius²  (use 3.14 for π)

```go
func circleArea(radius float32) {
    // Your code here
}

func main() {
    circleArea(5)   // Should print area
    circleArea(10)  // Should print area
}
```

---

### Exercise 5: Multiple Functions
Create three functions:
- `square(n int)` - prints n²
- `cube(n int)` - prints n³
- `double(n int)` - prints n×2

Call all three with the number 5.

---

## Getting Help

**Don't understand something? That's perfectly normal!**

### Resources:
1. ✅ **Discord** - Ask in the community
2. ✅ **Facebook Group** - Get help from peers
3. ✅ **Comments** - Leave a comment with your question
4. ✅ **Direct Message** - Contact the instructor

**Remember:**
- Functions might seem complex at first - that's okay!
- Practice by writing many small functions
- Visualize how memory works
- The instructor is here to help you understand
- Keep learning, keep practicing! 🚀

---

## What's Next?

In the next chapters, you'll learn:
- **Return Values** - Functions that return data
- **Multiple Return Values** - Go's special feature
- **Variadic Functions** - Functions with variable arguments
- **Anonymous Functions** - Functions without names
- **Recursion** - Functions calling themselves

**For Now:** Master basic functions! Once you understand how they work in memory and how to call them, you're ready for more advanced topics.

Keep coding! 💻✨

---

*Based on Go Programming Tutorial - Chapter 4*  
*Topics: Functions, Parameters, Arguments, Memory Management, Scope*
