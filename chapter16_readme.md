# Chapter 16: Function Expressions - A Noob Example

## 📚 Table of Contents
1. [Introduction](#introduction)
2. [What Is a Function Expression?](#what-is-a-function-expression)
3. [Assigning Functions to Variables](#assigning-functions-to-variables)
4. [The Critical Discovery](#the-critical-discovery)
5. [Global Scope vs Local Scope Behavior](#global-scope-vs-local-scope-behavior)
6. [Complete Memory Simulation](#complete-memory-simulation)
7. [The Order Problem Explained](#the-order-problem-explained)
8. [Shadowing with Function Expressions](#shadowing-with-function-expressions)
9. [Why This Happens](#why-this-happens)
10. [Common Mistakes](#common-mistakes)
11. [Practice Exercises](#practice-exercises)
12. [Summary](#summary)
13. [What's Next?](#whats-next)

---

## Introduction

**Today's Class: The Most Fun (and Chaotic!) Class!**

> **"This class will be very fun! We'll learn function expressions and how to assign functions to variables. We'll do simulation and it will be fun. You'll understand MANY things!"**

### What We'll Learn

1. ✅ **Function Expression** - What is it?
2. ✅ **Assign Function in Variable** - How to store functions
3. ✅ **Simulation** - Memory behavior
4. ✅ **The Order Problem** - A critical discovery!
5. ✅ **Global vs Local** - Different behaviors

### The Honest Teaching Style

> **"I don't prepare before class! Whatever comes to my mind, I start teaching. So that's why this happened... Sorry! But actually it turned out good - you learned even more!"**

**This class had mistakes, confusion, and discoveries - just like real programming!** 🎢

---

## What Is a Function Expression?

### Simple Definition

> **"Function Expression means: Assigning an anonymous function to a variable!"**

### The Concept

**Remember IIFE from last class?**
```go
func(a, b int) {
    fmt.Println(a + b)
}(5, 7)  // ← Immediately invoked
```

**Now: Don't invoke immediately, STORE IT!**
```go
add := func(a, b int) {
    fmt.Println(a + b)
}
// ← No () at the end!
// Stored in variable 'add'
```

---

### From IIFE to Function Expression

```go
// LAST CLASS (IIFE):
func(a, b int) {
    c := a + b
    fmt.Println(c)
}(5, 7)  // ← Invoke immediately

// THIS CLASS (Function Expression):
add := func(a, b int) {
    c := a + b
    fmt.Println(c)
}
// ← NOT invoked yet, just stored!
```

---

### Why It's Called "Expression"

**Remember variable declaration?**
```go
a := 10  // ← Variable declaration expression
```

**This entire line is an EXPRESSION!**

**Now with function:**
```go
add := func(a, b int) {
    fmt.Println(a + b)
}
// ← Function expression!
```

> **"Before I was assigning a VALUE (10). Now I'm assigning an entire FUNCTION! That's why it's called Function Expression!"**

---

## Assigning Functions to Variables

### Basic Example

```go
package main

import "fmt"

func init() {
    fmt.Println("I will be called first")
}

func main() {
    // Assign anonymous function to variable
    add := func(a, b int) {
        c := a + b
        fmt.Println(c)
    }
    
    // Now call it using the variable name
    add(2, 3)
}
```

**Output:**
```
I will be called first
5
```

---

### How It Works

**Step 1: Assign function to variable**
```go
add := func(a, b int) {
    c := a + b
    fmt.Println(c)
}
```

**What happened:**
- ✅ Anonymous function created
- ✅ Stored in variable `add`
- ✅ `add` now "IS" a function!

**Step 2: Call the function**
```go
add(2, 3)
```

**What happened:**
- ✅ `2` goes to `a`
- ✅ `3` goes to `b`
- ✅ `c = 2 + 3 = 5`
- ✅ Prints `5`

---

### Multiple Calls

```go
func main() {
    add := func(a, b int) {
        c := a + b
        fmt.Println(c)
    }
    
    add(2, 3)   // Output: 5
    add(4, 5)   // Output: 9
    add(10, 7)  // Output: 17
}
```

**Beautiful!** We can reuse the function!

---

## The Critical Discovery

### The Problem That Happened

> **"I made a mistake in this class... I used the same name 'add' in multiple places. It caused SO MUCH confusion! But actually, it turned out to be a good lesson!"**

### The Order Problem

**This is THE MOST IMPORTANT DISCOVERY of this class!** 🚨

---

### Experiment 1: Call Before Define (Local Scope)

```go
package main

import "fmt"

func main() {
    add(2, 3)  // ← Call BEFORE defining
    
    add := func(a, b int) {
        c := a + b
        fmt.Println(c)
    }
}
```

**Result:** ❌ **ERROR!**
```
undefined: add
```

---

### Experiment 2: Call After Define (Local Scope)

```go
package main

import "fmt"

func main() {
    add := func(a, b int) {
        c := a + b
        fmt.Println(c)
    }
    
    add(2, 3)  // ← Call AFTER defining
}
```

**Result:** ✅ **WORKS!**
```
5
```

---

### The Discovery

> **"Function expressions in LOCAL SCOPE: You MUST define BEFORE you call! You cannot call before defining!"**

**BUT...**

> **"In GLOBAL SCOPE, it works differently! Let me show you!"**

---

## Global Scope vs Local Scope Behavior

### Global Scope Function Expression

```go
package main

import "fmt"

// Can call this from main BEFORE seeing the definition!
var sum = func(a, b int) {
    fmt.Println(a + b)
}

func main() {
    sum(5, 7)  // This works!
}
```

**Output:** `12`

**This WORKS even though `sum` is defined above main!** ✅

---

### Why Global Works Differently

**Global scope allocation:**
```
Program starts
    ↓
Computer reads ENTIRE global scope first
    ↓
Allocates ALL global variables/functions to memory
    ↓
THEN starts executing main
    ↓
So main can "see" all global things!
```

---

### Local Scope - Sequential Execution

**Local scope execution:**
```
Enter function
    ↓
Execute line by line, TOP to BOTTOM
    ↓
Line 1 executed → then Line 2 → then Line 3...
    ↓
If you try to use something not yet defined → ERROR!
```

---

### Visual Comparison

```go
// GLOBAL SCOPE - Order doesn't matter
var greet = func() {
    fmt.Println("Hello")
}

func main() {
    greet()  // ✅ Works! Global allocated already
}

// ------------------------------------------

// LOCAL SCOPE - Order MATTERS!
func main() {
    greet()  // ❌ ERROR! Not defined yet!
    
    greet := func() {
        fmt.Println("Hello")
    }
}

// ------------------------------------------

// LOCAL SCOPE - Correct order
func main() {
    greet := func() {
        fmt.Println("Hello")
    }
    
    greet()  // ✅ Works! Defined before use
}
```

---

## Complete Memory Simulation

### Code to Simulate

```go
package main

import "fmt"

func sum(a, b int) {
    add(2, 4)  // Calls add (global)
}

func add(a, b int) {
    fmt.Println(a + b)
}

func init() {
    fmt.Println("I will be called first")
}

func main() {
    sum(0, 0)  // Calls sum
    
    add := func(a, b int) {  // Local function expression
        c := a + b
        fmt.Println(c)
    }
    
    add(4, 5)  // Calls local add
}
```

---

### Phase 1: Global Scope Allocation

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  sum = [function code]                  │
│  add = [function code]                  │
│  init = [function code]                 │
│  main = [function code]                 │
└─────────────────────────────────────────┘
```

**What happened:**
- ✅ Computer reads ALL global declarations
- ✅ Allocates memory for each function
- ✅ ALL stored with their names
- ⏸️ No execution yet!

---

### Phase 2: Init Executes

```
┌─────────────────────────────────────────┐
│         INIT SCOPE (Created)            │
├─────────────────────────────────────────┤
│  Executing: Println("I will...")        │
└─────────────────────────────────────────┘
```

**Output:** `I will be called first`

**Then:** Init scope destroyed! 💀

---

### Phase 3: Main Starts

```
┌─────────────────────────────────────────┐
│         MAIN SCOPE (Created)            │
├─────────────────────────────────────────┤
│  (empty - no variables yet)             │
└─────────────────────────────────────────┘
```

---

### Phase 4: sum() Called

**Line:** `sum(0, 0)`

**Computer checks:**
1. Is `sum` in main's local scope? ❌ No
2. Is `sum` in global scope? ✅ YES!
3. Call it!

```
┌─────────────────────────────────────────┐
│         SUM SCOPE (Created)             │
├─────────────────────────────────────────┤
│  a = 0                                  │
│  b = 0                                  │
│  (executing add(2, 4))                  │
└─────────────────────────────────────────┘
```

---

### Phase 5: add() Called from sum

**Line inside sum:** `add(2, 4)`

**Computer checks:**
1. Is `add` in sum's local scope? ❌ No
2. Is `add` in global scope? ✅ YES! (Global function)
3. Call it!

```
┌─────────────────────────────────────────┐
│         ADD SCOPE (Created)             │
├─────────────────────────────────────────┤
│  a = 2                                  │
│  b = 4                                  │
│  (computing 2 + 4 = 6)                  │
└─────────────────────────────────────────┘
```

**Output:** `6`

**Then:** Add scope destroyed! 💀

**Then:** Sum scope destroyed! 💀

---

### Phase 6: Function Expression Defined

**Line:** `add := func(a, b int) { ... }`

```
┌─────────────────────────────────────────┐
│         MAIN SCOPE                      │
├─────────────────────────────────────────┤
│  add = [function code] ← NEW!           │
└─────────────────────────────────────────┘
```

**Important:** Local `add` created!

---

### Phase 7: Local add() Called

**Line:** `add(4, 5)`

**Computer checks:**
1. Is `add` in main's local scope? ✅ YES! (Just defined)
2. Use local one!

```
┌─────────────────────────────────────────┐
│         MAIN SCOPE                      │
├─────────────────────────────────────────┤
│  add = [function code]                  │
└─────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│         ADD SCOPE (Local function)      │
├─────────────────────────────────────────┤
│  a = 4                                  │
│  b = 5                                  │
│  c = 9                                  │
│  (printing 9)                           │
└─────────────────────────────────────────┘
```

**Output:** `9`

**Final Output:**
```
I will be called first
6
9
```

---

## The Order Problem Explained

### The Child Analogy

> **"Imagine: Your child will come 20 years later. You're 100% sure your child will come after 20 years. But can you see your child NOW? NO! You have to wait 20 years!"**

**Same with local function expressions:**

```go
func main() {
    add(5, 7)  // ← Trying to see child NOW
    
    // 20 years pass (lines of code)
    
    add := func(a, b int) {  // ← Child is born HERE
        fmt.Println(a + b)
    }
}
```

**Error:** `undefined: add`

> **"Your child hasn't been born yet! You can't call them before they exist!"**

---

### Why Global Works

> **"Global scope is different! Computer reads ALL global things FIRST, allocates memory, THEN executes. So it's like all your global children are already born before main starts!"**

```go
// Global function expression
var add = func(a, b int) {
    fmt.Println(a + b)
}

func main() {
    add(5, 7)  // ✅ Works! Global 'add' already exists
}
```

**Global = Already born when main starts!** 👶

---

### The Rule

```
┌─────────────────────────────────────────┐
│  FUNCTION EXPRESSION ORDER RULE         │
├─────────────────────────────────────────┤
│                                         │
│  GLOBAL SCOPE:                          │
│    Order doesn't matter ✅              │
│    All allocated before execution       │
│                                         │
│  LOCAL SCOPE:                           │
│    Order MATTERS! 🚨                    │
│    Must define BEFORE calling           │
│    Sequential line-by-line execution    │
│                                         │
└─────────────────────────────────────────┘
```

---

## Shadowing with Function Expressions

### The Accidental Discovery

> **"I made a mistake! I used the name 'add' for both global and local functions! This caused confusion but taught us about shadowing!"**

### Shadowing Example

```go
package main

import "fmt"

// Global function
func add(a, b int) {
    fmt.Println("Global add:", a+b)
}

func main() {
    add(1, 2)  // ← Uses global add
    
    // Local function expression
    add := func(a, b int) {
        fmt.Println("Local add:", a+b)
    }
    
    add(3, 4)  // ← Uses local add (shadows global!)
}
```

**Output:**
```
Global add: 3
Local add: 7
```

---

### Memory with Shadowing

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  add = [global function] ← Original     │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         MAIN SCOPE                      │
├─────────────────────────────────────────┤
│  add = [local function] ← SHADOWS!      │
└─────────────────────────────────────────┘
```

**Inside main:** Local `add` shadows global `add`!

**Remember Chapter 12?** Variable shadowing works the same way!

---

## Why This Happens

### Global Scope Allocation

**What computer does:**

```
Step 1: Read entire file
Step 2: Find all global declarations
Step 3: Allocate memory for ALL of them
Step 4: NOW start executing init()
Step 5: Then execute main()
```

**Result:** All globals "exist" before any code runs!

---

### Local Scope Execution

**What computer does:**

```
Step 1: Enter function
Step 2: Execute line 1
Step 3: Execute line 2
Step 4: Execute line 3
...and so on
```

**Result:** Things only exist AFTER their line executes!

---

### The Difference Visualized

```go
// GLOBAL - All allocated BEFORE execution
var a = 10
var b = 20
var fn = func() { }

func main() {
    // Can use a, b, fn - already exist! ✅
}

// -----------------------------------------

// LOCAL - Sequential execution
func main() {
    // a, b, fn don't exist yet!
    
    a := 10          // ← Now a exists
    b := 20          // ← Now b exists
    fn := func() { } // ← Now fn exists
}
```

---

## Common Mistakes

### Mistake 1: Calling Before Defining (Local)

❌ **Wrong:**
```go
func main() {
    greet()  // ERROR: undefined
    
    greet := func() {
        fmt.Println("Hello")
    }
}
```

✅ **Correct:**
```go
func main() {
    greet := func() {
        fmt.Println("Hello")
    }
    
    greet()  // Works!
}
```

---

### Mistake 2: Forgetting () When Calling

❌ **Wrong:**
```go
func main() {
    greet := func() {
        fmt.Println("Hello")
    }
    
    greet  // ← Forgot (), doesn't call function!
}
```

✅ **Correct:**
```go
func main() {
    greet := func() {
        fmt.Println("Hello")
    }
    
    greet()  // ← () calls the function!
}
```

---

### Mistake 3: Confusing with IIFE

❌ **Wrong (tried to make IIFE but stored it):**
```go
func main() {
    add := func(a, b int) {
        fmt.Println(a + b)
    }(5, 7)  // ← This executes immediately!
    
    // add is now nil or has wrong type!
}
```

✅ **Correct (Function Expression):**
```go
func main() {
    add := func(a, b int) {
        fmt.Println(a + b)
    }  // ← No () here!
    
    add(5, 7)  // Call later
}
```

✅ **Correct (IIFE - different purpose):**
```go
func main() {
    func(a, b int) {
        fmt.Println(a + b)
    }(5, 7)  // ← Execute immediately, don't store
}
```

---

### Mistake 4: Thinking Global and Local Are Same

❌ **Wrong thinking:**
```
"Order doesn't matter anywhere"
```

✅ **Correct understanding:**
```
Global scope: Order doesn't matter
Local scope: Order MATTERS!
```

---

### Mistake 5: Not Understanding Shadowing

❌ **Wrong expectation:**
```go
func add(a, b int) {
    fmt.Println("Global")
}

func main() {
    add := func(a, b int) {
        fmt.Println("Local")
    }
    
    add(1, 2)  // "Will this call global or local?"
}
```

✅ **Correct understanding:**
```
Local shadows global!
Output: "Local"
```

---

## Practice Exercises

### Exercise 1: Basic Function Expression

Create a function expression that squares a number:

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    square := func(n int) int {
        return n * n
    }
    
    fmt.Println(square(5))   // 25
    fmt.Println(square(10))  // 100
}
```

</details>

---

### Exercise 2: Order Bug

Fix this code:

```go
package main

import "fmt"

func main() {
    greet("Alice")
    
    greet := func(name string) {
        fmt.Println("Hello,", name)
    }
}
```

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    greet := func(name string) {
        fmt.Println("Hello,", name)
    }
    
    greet("Alice")  // Move call AFTER definition
}
```

**Key:** Define before calling in local scope!

</details>

---

### Exercise 3: Global vs Local

Predict the output:

```go
package main

import "fmt"

var multiply = func(a, b int) int {
    return a * b
}

func main() {
    fmt.Println(multiply(3, 4))  // ?
    
    multiply := func(a, b int) int {
        return a * b * 2
    }
    
    fmt.Println(multiply(3, 4))  // ?
}
```

<details>
<summary>Answer</summary>

**Output:**
```
12
24
```

**Explanation:**
1. First call: Uses global `multiply` (3 * 4 = 12)
2. Second call: Uses local `multiply` which shadows global (3 * 4 * 2 = 24)

</details>

---

### Exercise 4: Multiple Function Expressions

Create three function expressions for add, subtract, multiply:

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    add := func(a, b int) int {
        return a + b
    }
    
    subtract := func(a, b int) int {
        return a - b
    }
    
    multiply := func(a, b int) int {
        return a * b
    }
    
    fmt.Println("Add:", add(10, 5))           // 15
    fmt.Println("Subtract:", subtract(10, 5)) // 5
    fmt.Println("Multiply:", multiply(10, 5)) // 50
}
```

</details>

---

### Exercise 5: Function Expression with Closure

Create a counter using function expression:

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    counter := 0
    
    increment := func() {
        counter++
        fmt.Println("Counter:", counter)
    }
    
    increment()  // Counter: 1
    increment()  // Counter: 2
    increment()  // Counter: 3
}
```

**Note:** The function "captures" the counter variable! (Closures - later topic)

</details>

---

### Exercise 6: Real-World Calculator

Build a simple calculator using function expressions:

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    calculator := func(operation string, a, b float64) float64 {
        switch operation {
        case "+":
            return a + b
        case "-":
            return a - b
        case "*":
            return a * b
        case "/":
            if b != 0 {
                return a / b
            }
            return 0
        default:
            return 0
        }
    }
    
    fmt.Println("10 + 5 =", calculator("+", 10, 5))
    fmt.Println("10 - 5 =", calculator("-", 10, 5))
    fmt.Println("10 * 5 =", calculator("*", 10, 5))
    fmt.Println("10 / 5 =", calculator("/", 10, 5))
}
```

**Output:**
```
10 + 5 = 15
10 - 5 = 5
10 * 5 = 50
10 / 5 = 2
```

</details>

---

## Summary

### Key Takeaways

1. ✅ **Function Expression = Anonymous function assigned to variable**
2. ✅ **Can be reused unlike IIFE**
3. ✅ **Order matters in LOCAL scope** - Define before calling!
4. ✅ **Order doesn't matter in GLOBAL scope**
5. ✅ **Can shadow global functions**
6. ✅ **Called using variable name: `functionName(args)`**
7. ✅ **Important interview concept!**

---

### Function Types Progress

```
✅ Chapter 13: Standard/Named Functions
✅ Chapter 14: Init Functions
✅ Chapter 15: Anonymous Functions & IIFE
✅ Chapter 16: Function Expressions ← YOU ARE HERE
⏭️  Next: Higher-Order Functions
⏭️  Then: Callbacks
⏭️  And more...
```

---

### Visual Summary

```
FUNCTION EXPRESSION:
    functionName := func(parameters) {
        // body
    }
    
    functionName(arguments)  ← Call using variable name

KEY RULE:
    Global Scope: Order doesn't matter
    Local Scope: Define BEFORE calling!
```

---

### The Order Rule Table

| Scope Type | Order Matters? | Why? |
|------------|----------------|------|
| **Global** | ❌ No | All allocated before execution |
| **Local** | ✅ YES! | Sequential line-by-line execution |

---

### Comparison Table

| Feature | Standard Function | Function Expression |
|---------|-------------------|---------------------|
| **Has name?** | ✅ Yes (function name) | ✅ Yes (variable name) |
| **Global order?** | Doesn't matter | Doesn't matter |
| **Local order?** | Doesn't matter | MATTERS! |
| **Can reassign?** | ❌ No | ✅ Yes (it's a variable) |
| **Syntax** | `func name() {}` | `name := func() {}` |

---

### The Honest Teaching Moment

> **"I don't know how this class went... It was very strange! I made mistakes! I got confused! But actually, that's how real learning happens!"**

**The teacher's honesty:**
- 🎭 "My energy got drained"
- 🎭 "I got frustrated with myself"
- 🎭 "I used the same name twice by mistake"
- 🎭 "But you learned MORE because of the mistakes!"

**Real learning includes:**
- ✅ Mistakes
- ✅ Confusion
- ✅ Discoveries
- ✅ "Aha!" moments

---

## What's Next?

### Chapter 17 Preview: Higher-Order Functions

Now that we can store functions in variables, we can do AMAZING things!

**Topics covered:**
- **What are higher-order functions?** - Functions that take/return functions
- **Passing functions as arguments** - Functions as parameters
- **Returning functions** - Functions returning functions
- **Real-world use cases** - Why this is powerful
- **Callback pattern** - Foundation for callbacks

**The power:**
```go
// Function expression (today)
add := func(a, b int) int {
    return a + b
}

// Higher-order function (next!)
func calculate(operation func(int, int) int, a, b int) int {
    return operation(a, b)
}

result := calculate(add, 5, 7)  // Pass function as argument!
```

---

### Why This Progression Matters

```
Chapter 13: Functions have names (Standard)
Chapter 14: Special automatic function (Init)
Chapter 15: Functions can have no name (Anonymous/IIFE)
Chapter 16: Functions can be stored (Function Expressions) ← YOU ARE HERE
Chapter 17: Functions can take/return functions (Higher-Order) ← NEXT!
```

**Each builds on the previous!** 🏗️

---

### The Learning Journey

**Where we've been:**
```
Functions are blocks of code
    ↓
Functions can have names
    ↓
Functions can have NO names
    ↓
Functions can be stored in variables
    ↓
Now: Functions can be PASSED AROUND!
```

---

### Final Wisdom

**The Child Analogy:**
> **"Your child will be born 20 years later. Can you see them now? NO! Same with local function expressions - they're not born until that line executes!"**

**The Global vs Local:**
> **"Global scope: Computer reads ALL first, allocates ALL, THEN executes. Local scope: Line by line, one at a time. That's why order matters locally!"**

**The Shadowing Reminder:**
> **"Local function expression can shadow global function. Just like variable shadowing we learned in Chapter 12!"**

**The Honest Teaching:**
> **"I made mistakes in this class. Used wrong names. Got confused. Wasted time. But that's REAL teaching! Real programming! We learn from mistakes!"**

**The Order Discovery:**
> **"This class's MAIN purpose: In local scope, function expressions must be defined BEFORE calling. This one rule took the whole class to teach!"**

---

### Quick Reference Card

```go
// FUNCTION EXPRESSION TEMPLATE

// Define first
functionName := func(parameters) returnType {
    // body
}

// Call after
functionName(arguments)

// REMEMBER:
✅ Global scope: Order doesn't matter
✅ Local scope: Define BEFORE calling!
✅ Call using variable name
✅ Can be reassigned (it's a variable)
✅ Can shadow global functions
```

---

### The Memory Pattern

```
GLOBAL SCOPE:
┌──────────────────┐
│ All allocated    │ ← Before execution
│ at once         │
└──────────────────┘

LOCAL SCOPE:
Line 1 executed → Thing 1 exists
Line 2 executed → Thing 2 exists  
Line 3 executed → Thing 3 exists
(Sequential!)
```

---

**See you in the next class!** 👋

---

**Teacher's final note:**
> **"I ran this test at the end just to waste time, since the class was already so long! But we got `9` as output - beautiful! I don't know how this class went - it was very bizarre! Yes!"**

😄

---

*End of Chapter 16 - Function Expressions Mastered (Through Chaos!)* 🎢
