# Chapter 13: Function Types - Standard/Named Functions

## 📚 Table of Contents
1. [Introduction](#introduction)
2. [Why Function Types Matter](#why-function-types-matter)
3. [Overview of All Function Types](#overview-of-all-function-types)
4. [What Is a Standard Function?](#what-is-a-standard-function)
5. [Standard Function Anatomy](#standard-function-anatomy)
6. [Standard vs Named - Same Thing](#standard-vs-named---same-thing)
7. [Examples of Standard Functions](#examples-of-standard-functions)
8. [Memory Behavior of Standard Functions](#memory-behavior-of-standard-functions)
9. [Why "Standard" Matters](#why-standard-matters)
10. [Common Mistakes](#common-mistakes)
11. [Practice Exercises](#practice-exercises)
12. [Summary](#summary)
13. [What's Next?](#whats-next)

---

## Introduction

**Today's Topic: Function Types!**

### The Big Picture

Remember when we said Go is primarily a **functional paradigm** language? That means:
- ✅ We write LOTS of functions
- ✅ Functions are everywhere in code
- ✅ Functions are the main building blocks

### The Relationship Analogy

> **"I'm a man. I'm going to spend my whole life with a woman. So what do we do? We spend MORE THAN HALF our lives thinking about women! Why? Because you have to think deeply about what you'll work with your entire life!"**

**Same with functions:**
- 🤔 We work with functions constantly
- 🤔 So we must learn function properties deeply
- 🤔 We must understand what types of functions exist
- 🤔 There's no escape from learning functions properly!

### The Career Reality

> **"If you don't think about women, no problem - you just won't have a girlfriend! But if you don't learn functions, you CAN'T work in Go!"**

**You need to think about what you'll work with for life!**

---

## Why Function Types Matter

### The Interview Perspective

**What employers look for:**
1. ✅ **Terminology knowledge** - Can you communicate?
2. ✅ **Foundation strength** - Can we train you further?
3. ✅ **Learning potential** - Will you grow?

### The Communication Advantage

```
You WITHOUT terminology:
❌ "I know this... thing that does... stuff"
❌ Employer: "He doesn't know anything"

You WITH terminology:
✅ "This is a higher-order function using closures"
✅ Employer: "He knows his concepts! Let's hire him!"
```

### The Salary Reality

> **"If a company offers you 30,000 Taka per month and gives you ONE MONTH to learn, they can train you for their needs. But they'll only hire you if you already show potential!"**

**Key insight:**
- 💰 30,000 Taka is nothing to a company
- 💰 They'll invest in you IF you show foundation
- 💰 They'll promote you after a year IF you deliver value
- 💰 But you need to SHOW you know the basics!

### Why Terminology Matters

> **"When you know terminology, you can talk to me, I can talk to you, the company can talk to you, everyone understands each other. Without terminology, even if you CAN do something, nobody understands what you're saying!"**

**You must learn proper terms!** 📚

---

## Overview of All Function Types

Before diving into standard functions, let's see ALL the function types we'll learn:

### Complete Function Types List

| # | Function Type | Description |
|---|---------------|-------------|
| 1 | **Standard/Named Function** | Function with a name (today's topic!) |
| 2 | **Anonymous Function** | Function without a name |
| 3 | **Function Expression** | Function assigned to a variable |
| 4 | **IIFE** | Immediately Invoked Function Expression |
| 5 | **Higher-Order Function** | Function that takes/returns functions |
| 6 | **Callback Function** | Function passed as argument |
| 7 | **Variadic Function** | Function with variable number of args |
| 8 | **Unit Function** | Function with no return value |
| 9 | **Closure** | Function that captures variables |
| 10 | **Defer Function** | Function delayed until function returns |
| 11 | **Receiver Function** | Method attached to a type |

**Whoa! So many types!** 😱

> **"ওরে খাইছে! (Oh no!) So many function types! Yes, MANY! We'll cover each one in separate classes!"**

### Today's Focus

```
┌─────────────────────────────────────┐
│  ALL FUNCTION TYPES                 │
├─────────────────────────────────────┤
│  [1] Standard/Named ← YOU ARE HERE  │
│  [ ] Anonymous                      │
│  [ ] Function Expression            │
│  [ ] IIFE                           │
│  [ ] Higher-Order                   │
│  [ ] Callback                       │
│  [ ] Variadic                       │
│  [ ] Unit                           │
│  [ ] Closure                        │
│  [ ] Defer                          │
│  [ ] Receiver                       │
└─────────────────────────────────────┘
```

**We start with the simplest: Standard Functions!**

---

## What Is a Standard Function?

### Simple Definition

> **"A Standard Function (or Named Function) is a function that HAS A NAME!"**

**That's it!** Everything we've learned so far has been standard functions!

### The Core Idea

```go
func add(a, b int) {  // ← "add" is the NAME
    fmt.Println(a + b)
}

func main() {  // ← "main" is the NAME
    add(4, 7)
}
```

**Both are standard functions because:**
- ✅ `add` has a name
- ✅ `main` has a name
- ✅ They're "standard" in Go

### The Hint in the Name

> **"Since we're saying functions WITH names are 'standard', that means there must be functions WITHOUT names coming later!"**

**Foreshadowing:** Yes! Anonymous functions have NO name!

---

## Standard Function Anatomy

Let's dissect what makes a function "standard":

### Basic Structure

```go
func functionName(parameters) returnType {
    // function body
}
```

### Detailed Breakdown

```go
func add(a, b int) int {
    return a + b
}

┌────┬──────────────────────────────────┐
│ ✅ │ func keyword                     │
│ ✅ │ "add" - The NAME (makes it std)  │
│ ✅ │ Parameters (a, b int)            │
│ ✅ │ Return type (int)                │
│ ✅ │ Body with logic                  │
└────┴──────────────────────────────────┘
```

### What Makes It "Standard"?

**Only ONE requirement:**

```
HAS A NAME = Standard Function ✅
NO NAME = Anonymous Function (later!)
```

**That's the ONLY difference!**

---

## Standard vs Named - Same Thing

### Two Terms, One Concept

```
Standard Function = Named Function
       ↓                  ↓
   Same Thing!      Same Thing!
       ↓                  ↓
   Has a name        Has a name
```

**You can use either term:**
- ✅ "This is a standard function"
- ✅ "This is a named function"
- ✅ Both mean the same thing!

### Why Two Names?

**Historical reasons:**
- 📚 "Standard" = Common/normal way in Go
- 📚 "Named" = Contrasts with "anonymous"

**Use whichever you prefer!**

---

## Examples of Standard Functions

### Example 1: Simple Addition

```go
package main

import "fmt"

func add(a, b int) {
    fmt.Println(a + b)
}

func main() {
    add(4, 7)  // Output: 11
}
```

**Analysis:**
- ✅ `add` is a standard function (has name "add")
- ✅ `main` is a standard function (has name "main")
- ✅ Both are stored in global scope

**Run it:**
```bash
$ go run main.go
11
```

> **"11 is a prime number! Lucky number!" 🍀**

---

### Example 2: Multiple Standard Functions

```go
package main

import "fmt"

func greet(name string) {
    fmt.Println("Hello,", name)
}

func add(a, b int) int {
    return a + b
}

func multiply(a, b int) int {
    return a * b
}

func main() {
    greet("Miaki")
    
    sum := add(5, 3)
    fmt.Println("Sum:", sum)
    
    product := multiply(4, 7)
    fmt.Println("Product:", product)
}
```

**Output:**
```
Hello, Miaki
Sum: 8
Product: 28
```

**All are standard functions:**
- ✅ `greet` - has name
- ✅ `add` - has name
- ✅ `multiply` - has name
- ✅ `main` - has name

---

### Example 3: Function with Multiple Returns

```go
package main

import "fmt"

func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)
}
```

**Output:**
```
Result: 5
```

**Still a standard function:**
- ✅ `divide` has a name
- ✅ Has multiple return values
- ✅ Still "standard" because it has a name!

---

### Example 4: No Return Value

```go
package main

import "fmt"

func printSeparator() {
    fmt.Println("===================")
}

func displayInfo(name string, age int) {
    printSeparator()
    fmt.Printf("Name: %s\n", name)
    fmt.Printf("Age: %d\n", age)
    printSeparator()
}

func main() {
    displayInfo("Alice", 25)
}
```

**Output:**
```
===================
Name: Alice
Age: 25
===================
```

**All standard functions:**
- ✅ `printSeparator` - has name, no params, no return
- ✅ `displayInfo` - has name, params, no return
- ✅ `main` - has name

**Having or not having returns doesn't matter - the NAME makes it standard!**

---

## Memory Behavior of Standard Functions

### How Standard Functions Are Stored

**Let's trace memory for this code:**

```go
package main

import "fmt"

func add(a, b int) int {
    return a + b
}

func main() {
    result := add(4, 7)
    fmt.Println(result)
}
```

---

### Phase 1: Declaration Phase (Reading Code)

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  add = [function code]                  │
│  main = [function code]                 │
└─────────────────────────────────────────┘
```

**What happened:**
1. ✅ Computer reads `func add(...)` → Stores function definition
2. ✅ Computer reads `func main(...)` → Stores function definition
3. ✅ Both stored in global scope
4. ✅ No execution yet!

**Key Point:** Standard functions are stored by their NAME in global scope!

---

### Phase 2: Execution Phase (main runs)

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  add = [function code]                  │
│  main = [function code] ← EXECUTING     │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         MAIN SCOPE (Created)            │
├─────────────────────────────────────────┤
│  (waiting for result...)                │
└─────────────────────────────────────────┘
```

**Computer finds `main` by NAME and executes it!**

---

### Phase 3: Function Call (add is called)

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  add = [function code] ← FOUND BY NAME  │
│  main = [function code]                 │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         MAIN SCOPE                      │
├─────────────────────────────────────────┤
│  (waiting for add to return...)         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         ADD SCOPE (Created)             │
├─────────────────────────────────────────┤
│  a = 4                                  │
│  b = 7                                  │
│  (computing a + b = 11)                 │
└─────────────────────────────────────────┘
```

**Computer searches for "add" by NAME:**
1. ✅ Looks in global scope
2. ✅ Finds `add` function
3. ✅ Creates scope for add
4. ✅ Executes add(4, 7)

---

### Phase 4: Return and Store

```
┌─────────────────────────────────────────┐
│         MAIN SCOPE                      │
├─────────────────────────────────────────┤
│  result = 11 ← Returned from add        │
└─────────────────────────────────────────┘

[ADD SCOPE DESTROYED! 💀]
```

**What happened:**
1. ✅ `add` returns 11
2. ✅ `add` scope destroyed
3. ✅ `result = 11` stored in main scope
4. ✅ Print result: 11

---

### The Power of Names

**Why names matter:**

```go
add(4, 7)  // ← Computer searches for "add" by NAME
```

**Search process:**
1. Check local scope for "add"
2. Check parent scopes for "add"
3. Check global scope for "add" ✅ FOUND!
4. Execute function

**Without a name, how would computer find the function?**
> **"That's why anonymous functions work differently! We'll see later!"**

---

## Why "Standard" Matters

### Standard = Common Practice

**In Go, standard functions are:**
- ✅ The most common type
- ✅ The clearest to read
- ✅ The easiest to understand
- ✅ The foundation for other types

### Standard Functions Are Everywhere

**Built-in standard functions:**
```go
fmt.Println()   // ← Standard function (name: Println)
len()           // ← Standard function (name: len)
append()        // ← Standard function (name: append)
make()          // ← Standard function (name: make)
```

**Your standard functions:**
```go
func main()         // ← Standard function
func add(a, b int)  // ← Standard function
func greet(name string)  // ← Standard function
```

**All have names → All are standard!**

---

### The Baseline Understanding

**Everything else builds on this:**

```
Standard Functions (with names)
        ↓
    Foundation
        ↓
Learn other types:
- Anonymous (no name)
- Higher-order (takes functions)
- Closures (captures variables)
- etc.
```

**You can't understand advanced types without understanding standard functions first!**

---

## Common Mistakes

### Mistake 1: Thinking All Functions Are The Same

❌ **Wrong thinking:**
```
"All functions are just functions, right?"
```

✅ **Correct thinking:**
```
"Functions have TYPES! Standard is just ONE type.
There are anonymous, higher-order, closures, etc."
```

---

### Mistake 2: Not Knowing The Term

❌ **In interview:**
```
Interviewer: "Is this a standard function?"
You: "Uh... it's just a function..."
Interviewer: 🚩 "He doesn't know terminology"
```

✅ **Better answer:**
```
Interviewer: "Is this a standard function?"
You: "Yes! It's a standard/named function because
     it has a name. Later we'll use anonymous
     functions that don't have names."
Interviewer: ✅ "Good understanding!"
```

---

### Mistake 3: Confusing "Standard" with "Simple"

❌ **Wrong thinking:**
```
"Standard = simple/basic function"
"Complex functions are not standard"
```

✅ **Correct thinking:**
```
"Standard = has a name (that's the ONLY rule!)
Can be simple or complex
Can have any number of parameters
Can have any return type
The NAME makes it 'standard', not complexity!"
```

**Example of complex standard function:**
```go
func processData(
    input []string,
    filter func(string) bool,
    transform func(string) string,
) ([]string, error) {
    // Complex logic here...
}
```

**Still standard because it has a name: `processData`!**

---

### Mistake 4: Not Using Descriptive Names

❌ **Bad names:**
```go
func f() { }      // What does 'f' do?
func x() { }      // What does 'x' do?
func doIt() { }   // Do what?
```

✅ **Good names:**
```go
func calculateTotal() { }
func validateEmail() { }
func fetchUserData() { }
```

**The name is the INTERFACE to your function!**

---

### Mistake 5: Forgetting Functions Are First-Class

❌ **Limited thinking:**
```
"Functions can only be called with ()"
```

✅ **Advanced thinking:**
```
"In Go, functions are first-class citizens!
- Can be assigned to variables (later!)
- Can be passed as arguments (later!)
- Can be returned from functions (later!)
BUT standard functions are still the foundation!"
```

**Standard functions can do ALL these things!**

---

## Practice Exercises

### Exercise 1: Identify Standard Functions

Which of these are standard functions?

```go
package main

import "fmt"

func greet(name string) {  // A
    fmt.Println("Hello", name)
}

var sayBye = func() {  // B
    fmt.Println("Goodbye")
}

func main() {  // C
    greet("Alice")
    sayBye()
}
```

**Question:** Which are standard functions?

<details>
<summary>Answer</summary>

- **A (`greet`)**: ✅ Standard function (has name)
- **B (`sayBye`)**: ❌ NOT standard (it's a function expression - later topic!)
- **C (`main`)**: ✅ Standard function (has name)

**Standard functions: A and C only!**

</details>

---

### Exercise 2: Create Standard Functions

Create standard functions for these tasks:

1. A function named `square` that takes an int and returns its square
2. A function named `isEven` that takes an int and returns bool
3. A function named `max` that takes two ints and returns the larger one
4. A function named `greetMultiple` that takes a name and count, prints greeting count times

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func square(n int) int {
    return n * n
}

func isEven(n int) bool {
    return n%2 == 0
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func greetMultiple(name string, count int) {
    for i := 0; i < count; i++ {
        fmt.Printf("Hello, %s!\n", name)
    }
}

func main() {
    fmt.Println("Square of 5:", square(5))
    fmt.Println("Is 4 even?", isEven(4))
    fmt.Println("Max of 10 and 20:", max(10, 20))
    greetMultiple("Alice", 3)
}
```

**All are standard functions because they all have names!**

</details>

---

### Exercise 3: Convert Main Logic to Standard Functions

Refactor this messy main function using standard functions:

```go
package main

import "fmt"

func main() {
    // Calculate circle area
    radius := 5.0
    area := 3.14159 * radius * radius
    fmt.Println("Area:", area)
    
    // Calculate rectangle area
    length := 10.0
    width := 5.0
    rectArea := length * width
    fmt.Println("Rectangle Area:", rectArea)
    
    // Calculate triangle area
    base := 8.0
    height := 6.0
    triArea := 0.5 * base * height
    fmt.Println("Triangle Area:", triArea)
}
```

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func circleArea(radius float64) float64 {
    return 3.14159 * radius * radius
}

func rectangleArea(length, width float64) float64 {
    return length * width
}

func triangleArea(base, height float64) float64 {
    return 0.5 * base * height
}

func main() {
    fmt.Println("Circle Area:", circleArea(5.0))
    fmt.Println("Rectangle Area:", rectangleArea(10.0, 5.0))
    fmt.Println("Triangle Area:", triangleArea(8.0, 6.0))
}
```

**Much cleaner! Each calculation is now a standard function!**

</details>

---

### Exercise 4: Memory Trace

Trace the memory for this code:

```go
package main

import "fmt"

func double(n int) int {
    return n * 2
}

func main() {
    x := 5
    y := double(x)
    fmt.Println(y)
}
```

Draw the memory state at each phase:
1. After global scope created
2. When main starts
3. When double is called
4. After double returns

<details>
<summary>Solution</summary>

**Phase 1: Global Scope**
```
┌─────────────────────────────────┐
│  GLOBAL SCOPE                   │
├─────────────────────────────────┤
│  double = [function code]       │
│  main = [function code]         │
└─────────────────────────────────┘
```

**Phase 2: Main Starts**
```
┌─────────────────────────────────┐
│  GLOBAL SCOPE                   │
├─────────────────────────────────┤
│  double = [function code]       │
│  main = [function code]         │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  MAIN SCOPE                     │
├─────────────────────────────────┤
│  x = 5                          │
└─────────────────────────────────┘
```

**Phase 3: Double Called**
```
┌─────────────────────────────────┐
│  MAIN SCOPE                     │
├─────────────────────────────────┤
│  x = 5                          │
│  (waiting for y...)             │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  DOUBLE SCOPE                   │
├─────────────────────────────────┤
│  n = 5                          │
│  (computing 5 * 2 = 10)         │
└─────────────────────────────────┘
```

**Phase 4: After Return**
```
┌─────────────────────────────────┐
│  MAIN SCOPE                     │
├─────────────────────────────────┤
│  x = 5                          │
│  y = 10                         │
└─────────────────────────────────┘

[DOUBLE SCOPE DESTROYED]
```

</details>

---

### Exercise 5: Interview Question

**Question:** "What's the difference between a standard function and an anonymous function?"

Prepare your answer!

<details>
<summary>Sample Answer</summary>

**Good Answer:**

"A standard function (also called a named function) is a function that has a name, like:

```go
func add(a, b int) int {
    return a + b
}
```

The name `add` makes it a standard function. These are stored in global scope and can be called by name from anywhere.

An anonymous function is a function without a name, like:

```go
func(a, b int) int {
    return a + b
}
```

Anonymous functions are typically assigned to variables or used inline. They're useful for short operations or when you need a function just once.

The key difference is: **standard has a NAME, anonymous does NOT.**"

**Interviewer:** ✅ "Excellent explanation!"

</details>

---

### Exercise 6: Best Practices

Review this code and suggest improvements:

```go
package main

import "fmt"

func f(a, b int) int {
    return a + b
}

func g(s string) {
    fmt.Println(s)
}

func main() {
    x := f(5, 3)
    g("Result")
    fmt.Println(x)
}
```

<details>
<summary>Solution</summary>

**Improved version:**

```go
package main

import "fmt"

// add returns the sum of two integers
func add(a, b int) int {
    return a + b
}

// printMessage displays a message to the console
func printMessage(message string) {
    fmt.Println(message)
}

func main() {
    sum := add(5, 3)
    printMessage("Result:")
    fmt.Println(sum)
}
```

**Improvements:**
1. ✅ Descriptive names (`add` instead of `f`)
2. ✅ Descriptive names (`printMessage` instead of `g`)
3. ✅ Added comments for documentation
4. ✅ Descriptive variable names (`sum` instead of `x`)
5. ✅ Still all standard functions!

**Principle:** Standard functions need GOOD NAMES to be useful!

</details>

---

## Summary

### Key Takeaways

1. ✅ **Standard/Named Function = Function with a name**
2. ✅ **Everything we've learned so far is standard functions**
3. ✅ **"Standard" and "Named" mean the same thing**
4. ✅ **Names allow functions to be found and called**
5. ✅ **Most common and fundamental function type in Go**
6. ✅ **Foundation for all other function types**
7. ✅ **Knowing terminology helps in interviews and communication**

---

### The Definition

```
┌───────────────────────────────────────┐
│  STANDARD/NAMED FUNCTION              │
├───────────────────────────────────────┤
│  A function that HAS A NAME           │
│                                       │
│  Syntax:                              │
│  func name(params) returnType {       │
│      // body                          │
│  }                                    │
│                                       │
│  Key: The NAME makes it "standard"    │
└───────────────────────────────────────┘
```

---

### Visual Summary

```
ALL FUNCTIONS IN GO
        │
        ├─── With NAME ──→ Standard/Named Functions ✅
        │                  (Today's topic!)
        │
        └─── Without NAME ──→ Anonymous Functions
                              (Next topics!)
```

---

### The Function Types Journey

```
Chapter 13: Standard/Named ← ✅ COMPLETED
            │
            ├─ Next: Anonymous
            ├─ Then: Function Expression
            ├─ Then: IIFE
            ├─ Then: Higher-Order
            ├─ Then: Callback
            ├─ Then: Variadic
            ├─ Then: Closure
            ├─ Then: Defer
            └─ Finally: Receiver
```

**We're just getting started!** 🚀

---

### The Career Lesson

> **"Companies want to hire people they can TRAIN. But you need to show POTENTIAL! Learn terminology, learn foundations, show you can GROW!"**

**Key insights:**
1. 💰 30,000 Taka/month is nothing to a company
2. 💰 They'll invest in you IF you show foundation
3. 💰 Knowing terminology shows professionalism
4. 💰 Understanding basics shows learning potential
5. 💰 Companies hire for potential, train for specifics!

---

### The Communication Principle

```
WITHOUT Terminology:
You: "It's a... function thing..."
Employer: ❌ "Doesn't know basics"

WITH Terminology:
You: "It's a standard function with multiple returns"
Employer: ✅ "Knows the concepts! Hire him!"
```

**Communication = Career success!** 📞

---

## What's Next?

### Chapter 14 Preview: Anonymous Functions

Now that you know functions WITH names, it's time to learn functions WITHOUT names!

**Topics covered:**
- **What is an anonymous function?** - Functions without names
- **Why use anonymous functions?** - Use cases and benefits
- **Function expressions** - Assigning functions to variables
- **Inline anonymous functions** - Using functions on the spot
- **Anonymous vs Standard** - When to use which
- **Common patterns** - Real-world usage

**The contrast:**
```go
// Standard (named)
func add(a, b int) int {
    return a + b
}

// Anonymous (no name!)
func(a, b int) int {
    return a + b
}
```

**Question:** If it has no name, how do we call it? 🤔

**Find out in Chapter 14!**

---

### Why This Progression Matters

```
Step 1: Learn WITH names (Standard)
         ↓
Step 2: Learn WITHOUT names (Anonymous)
         ↓
Step 3: Compare and contrast
         ↓
Step 4: Learn advanced patterns
         ↓
Step 5: Master functions!
```

**You can't appreciate anonymous without understanding standard first!**

---

### The Philosophy

> **"You spend your life with what you work with. I work with functions. So I must understand functions DEEPLY. What types exist? Why do they exist? When to use which? These are not optional questions - they're SURVIVAL questions for a Go developer!"**

**Remember:**
- 🎯 Go is functional paradigm
- 🎯 Functions are everywhere
- 🎯 Master functions = Master Go
- 🎯 We're building up knowledge step by step

---

### What We've Built So Far

```
Chapter 7: Why Functions Are Needed
Chapter 8: What Is Scope
Chapter 9: Local Scope and Block
Chapter 10: Package Scope
Chapter 11: Scope Deep Dive
Chapter 12: Variable Shadowing
Chapter 13: Function Types - Standard ← YOU ARE HERE
Chapter 14: Anonymous Functions (Next!)
```

**Each chapter builds on the last!** 🏗️

---

### Final Wisdom

**The Relationship Analogy:**
> **"Like spending time understanding the woman you'll marry, spend time understanding functions you'll use every day!"**

**The Career Lesson:**
> **"Companies don't expect you to know EVERYTHING. They expect you to know ENOUGH to be trained further. Show foundation, show potential, show terminology - get hired!"**

**The Learning Path:**
> **"We'll cover each function type one by one. No rush. Each class, one type. By the end, you'll know them all!"**

---

**See you in the next class!** 👋

---

### Quick Reference

**Standard Function Checklist:**

```go
func myFunction(params) returnType {  // ✅ All checked?
    // body
}

✅ Has 'func' keyword?
✅ Has a NAME? (myFunction)
✅ Has parameters? (can be empty)
✅ Has return type? (can be none)
✅ Has body? (with logic)

If it has a NAME ──→ STANDARD FUNCTION!
```

---

**Memory Reminder:**

```
1. Declaration Phase: Functions stored by NAME in global scope
2. Execution Phase: Computer finds functions BY NAME
3. Call Phase: Create scope for function
4. Return Phase: Destroy scope, return to caller

The NAME is how computer finds the function!
```

---

*End of Chapter 13 - Standard Functions Mastered!* 🎯

