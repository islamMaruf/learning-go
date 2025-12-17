# Chapter 15: Anonymous Functions and IIFE

## 📚 Table of Contents
1. [Introduction](#introduction)
2. [What Is an Anonymous Function?](#what-is-an-anonymous-function)
3. [Named vs Anonymous Functions](#named-vs-anonymous-functions)
4. [Why Anonymous Functions Can't Stand Alone in Go](#why-anonymous-functions-cant-stand-alone-in-go)
5. [What Is IIFE?](#what-is-iife)
6. [Understanding "Invoke"](#understanding-invoke)
7. [Understanding "Expression"](#understanding-expression)
8. [Complete IIFE Examples](#complete-iife-examples)
9. [When to Use Anonymous Functions](#when-to-use-anonymous-functions)
10. [Common Mistakes](#common-mistakes)
11. [Practice Exercises](#practice-exercises)
12. [Summary](#summary)
13. [What's Next?](#whats-next)

---

## Introduction

**Today's Topics: Anonymous Functions and IIFE!**

### What We'll Learn

1. ✅ **Anonymous Functions** - Functions without names
2. ✅ **IIFE** - Immediately Invoked Function Expression
3. ✅ **Expression** - What does this term mean?
4. ✅ **Invoke** - The proper term for calling functions
5. ✅ **Interview preparation** - Common questions!

### Quick Preview

```go
// Named function (we know this!)
func add(a, b int) {
    fmt.Println(a + b)
}

// Anonymous function (NEW!)
func(a, b int) {
    fmt.Println(a + b)
}(5, 7)  // ← Immediately invoked!
```

**Notice:** The second function has NO NAME! 👻

---

## What Is an Anonymous Function?

### Simple Definition

> **"Anonymous means UNKNOWN - someone you don't know. An anonymous person is someone whose name you don't know. An anonymous function is a function that HAS NO NAME - you can't identify it, you can't catch it!"**

### The Core Concept

**Anonymous = No Name**

```
Named Function:
    func add(a, b int) { }
         ↑
      Has NAME!

Anonymous Function:
    func(a, b int) { }
         ↑
      NO NAME!
```

---

### Visual Comparison

```go
// NAMED (Standard Function)
func add(a, b int) int {
    return a + b
}

// Can call it by name:
add(5, 7)  ✅


// ANONYMOUS (No Name!)
func(a, b int) int {
    return a + b
}

// How do we call it? 🤔
// No name to reference!
```

---

## Named vs Anonymous Functions

### Named Function Example

```go
package main

import "fmt"

func add(a, b int) {
    fmt.Println(a + b)
}

func main() {
    add(5, 7)  // Output: 12
}
```

**Properties:**
- ✅ Has name: `add`
- ✅ Stored in memory with name
- ✅ Can be called from anywhere
- ✅ Type: Standard/Named function

---

### Anonymous Function (Attempt 1 - ERROR!)

```go
package main

import "fmt"

// ❌ This will ERROR!
func(a, b int) {
    c := a + b
    fmt.Println(c)
}

func main() {
    // How do we call it?
}
```

**Error:** `syntax error: unexpected func`

**Why?**
> **"In Go, you CANNOT just leave an anonymous function lying around like this! JavaScript allows it, but Go doesn't!"**

---

### Anonymous Function (Correct - IIFE!)

```go
package main

import "fmt"

func main() {
    // Define AND invoke immediately!
    func(a, b int) {
        c := a + b
        fmt.Println(c)
    }(5, 7)  // ← Invoke immediately!
}
```

**Output:** `12`

**This works!** ✅

---

## Why Anonymous Functions Can't Stand Alone in Go

### The Memory Problem

**Named function:**
```
Memory:
┌─────────────────────┐
│  Name: "add"        │ ← Computer can find it by name!
│  Code: [function]   │
└─────────────────────┘

Call it:
add(5, 7)  ← Computer searches for "add"
```

---

**Anonymous function:**
```
Memory:
┌─────────────────────┐
│  Name: ???          │ ← NO NAME! How to find?
│  Code: [function]   │
└─────────────────────┘

Call it:
???(5, 7)  ← What name to use? ❌
```

---

### The Analogy

> **"If a function has no name, how can I CATCH it? How can I HOLD it? It's like a ghost - you can't grab something without a name!"**

**Think of it like:**
```
Named Function = Person with ID card
    ↓
Can find them, call them, store contact

Anonymous Function = Mystery person
    ↓
Can't find them later, can't call them back
```

---

### The Solution

**Two ways to use anonymous functions:**

1. **IIFE** - Invoke immediately (use once, right now)
2. **Assign to variable** - Give it a name (we'll learn later!)

**Today we focus on IIFE!** ✨

---

## What Is IIFE?

### Full Name

**IIFE = Immediately Invoked Function Expression**

```
I - Immediately
I - Invoked
F - Function
E - Expression
```

**Pronounced:** "iffy" or "I-F-E"

---

### Breaking It Down

**Immediately:**
- 🕐 Right now
- 🕐 Without delay
- 🕐 As soon as defined

**Invoked:**
- 📞 Called/executed
- 📞 Proper programming term
- 📞 More professional than "call"

**Function:**
- 🔧 The function itself
- 🔧 Anonymous function

**Expression:**
- 📝 A complete statement
- 📝 A line of code
- 📝 Something that evaluates

---

### IIFE Structure

```go
func(parameters) {
    // function body
}(arguments)  ← Invoke immediately!
   ↑
   This invokes the function above
```

**Visual breakdown:**
```
┌──────────────────────────────┐
│ func(a, b int) {             │ ← Function definition
│     fmt.Println(a + b)       │
│ }                            │
└──────────────────────────────┘
  ↓
  (5, 7)  ← Immediate invocation
```

---

### Complete IIFE Example

```go
package main

import "fmt"

func init() {
    fmt.Println("I will be called first")
}

func main() {
    // IIFE - Define and invoke immediately
    func(a, b int) {
        c := a + b
        fmt.Println(c)
    }(5, 7)
}
```

**Output:**
```
I will be called first
12
```

---

### How It Works

**Step-by-step:**

1. **Define the function:**
   ```go
   func(a, b int) {
       c := a + b
       fmt.Println(c)
   }
   ```

2. **Invoke it immediately:**
   ```go
   (5, 7)  ← These are the arguments!
   ```

3. **What happens:**
   - `a` receives `5`
   - `b` receives `7`
   - `c = 5 + 7 = 12`
   - Prints `12`

---

## Understanding "Invoke"

### The Proper Term

> **"We shouldn't say 'call' - we should say INVOKE! That's the proper programming term!"**

### Why "Invoke" Not "Call"?

**Common terms (less formal):**
- ❌ "Call the function"
- ❌ "Execute the function"
- ❌ "Run the function"

**Professional term:**
- ✅ "Invoke the function"

---

### Invoke in Action

```go
func add(a, b int) {
    fmt.Println(a + b)
}

func main() {
    add(2, 4)  // ← We INVOKE the add function here
}
```

> **"We invoke the add function here! Invoke means to call/execute/run the function. But 'invoke' is the correct term!"**

---

### Teaching Note

> **"I say 'call' or 'execute' to make it easier for you to understand. But actually, I should say 'invoke'! Sometimes I say invoke, sometimes I say call - sorry! But remember: INVOKE is the proper term!"**

---

## Understanding "Expression"

### What Is an Expression?

**Expression = A statement/line of code**

### Variable Declaration Expression

```go
a := 10
```

**This entire line is an EXPRESSION!**

> **"In any programming language - C++, Java, Python, whatever - this single line is called an expression!"**

---

### If Expression

```go
if a > 0 {
    fmt.Println("a is greater than zero")
}
```

**Parts:**
- `if a > 0` ← **If Expression**
- `{ ... }` ← **If Block**

**Together:** The whole thing can be called an "If Expression"

---

### Function Expression

```go
func add(a, b int) {
    fmt.Println(a + b)
}
```

**This is a function expression!**
- The function definition = Expression
- The function block = Block
- Together = Function Expression

---

### Function Invocation Expression

```go
add(2, 4)
```

**This is a function invocation expression!**

> **"Invoking a function is an expression!"**

---

### IIFE = Expression

```go
func(a, b int) {
    fmt.Println(a + b)
}(5, 7)
```

**This entire thing is an expression:**
- Function definition + Immediate invocation
- Hence: "Immediately Invoked Function Expression"

---

### Expression Examples Summary

```go
a := 10                           // Variable declaration expression

if a > 0 { }                      // If expression

for i := 0; i < 5; i++ { }        // Loop expression

func add() { }                    // Function expression

add()                             // Function invocation expression

func() { }()                      // IIFE expression
```

**All are expressions!** 📝

---

## Complete IIFE Examples

### Example 1: Basic IIFE

```go
package main

import "fmt"

func init() {
    fmt.Println("I will be called first")
}

func main() {
    func(a, b int) {
        c := a + b
        fmt.Println(c)
    }(5, 7)
}
```

**Output:**
```
I will be called first
12
```

---

### Example 2: Change Arguments

```go
package main

import "fmt"

func main() {
    func(a, b int) {
        fmt.Println(a + b)
    }(4, 7)  // Changed to 4 and 7
}
```

**Output:** `11`

**How it works:**
- `a` receives `4`
- `b` receives `7`
- `4 + 7 = 11`
- Prints `11`

---

### Example 3: IIFE with String

```go
package main

import "fmt"

func main() {
    func(name string, age int) {
        fmt.Printf("Name: %s, Age: %d\n", name, age)
    }("Alice", 25)
}
```

**Output:** `Name: Alice, Age: 25`

---

### Example 4: IIFE with No Parameters

```go
package main

import "fmt"

func main() {
    func() {
        fmt.Println("I'm an IIFE with no parameters!")
    }()  // ← Empty parentheses, but still needed!
}
```

**Output:** `I'm an IIFE with no parameters!`

**Note:** Even with no parameters, you need `()` to invoke!

---

### Example 5: Multiple IIFEs

```go
package main

import "fmt"

func main() {
    func(x int) {
        fmt.Println("First IIFE:", x)
    }(10)
    
    func(x int) {
        fmt.Println("Second IIFE:", x)
    }(20)
    
    func(x, y int) {
        fmt.Println("Third IIFE:", x+y)
    }(5, 15)
}
```

**Output:**
```
First IIFE: 10
Second IIFE: 20
Third IIFE: 20
```

---

### Example 6: IIFE with Return Value

```go
package main

import "fmt"

func main() {
    // IIFE that returns a value
    result := func(a, b int) int {
        return a * b
    }(4, 5)
    
    fmt.Println("Result:", result)
}
```

**Output:** `Result: 20`

**How it works:**
- IIFE calculates `4 * 5 = 20`
- Returns `20`
- Stored in `result`

---

### Example 7: IIFE for Initialization

```go
package main

import "fmt"

func main() {
    // Use IIFE to initialize a complex variable
    config := func() map[string]string {
        m := make(map[string]string)
        m["env"] = "development"
        m["version"] = "1.0.0"
        return m
    }()  // ← Invoke immediately!
    
    fmt.Println("Config:", config)
}
```

**Output:** `Config: map[env:development version:1.0.0]`

---

## When to Use Anonymous Functions

### Use Case 1: One-Time Operations

```go
func main() {
    // Need to do something ONCE, right now
    func() {
        fmt.Println("Setup complete")
        fmt.Println("Ready to start")
    }()
}
```

**Good for:** Quick setup that doesn't need a separate function

---

### Use Case 2: Scope Isolation

```go
func main() {
    x := 10
    
    // Create isolated scope
    func() {
        x := 20  // This x is local to IIFE
        fmt.Println("Inside IIFE:", x)  // 20
    }()
    
    fmt.Println("Outside IIFE:", x)  // 10
}
```

**Output:**
```
Inside IIFE: 20
Outside IIFE: 10
```

---

### Use Case 3: Defer with Parameters

```go
func main() {
    x := 5
    
    defer func(val int) {
        fmt.Println("Deferred value:", val)
    }(x)  // Capture current value of x
    
    x = 10
    fmt.Println("Current x:", x)
}
```

**Output:**
```
Current x: 10
Deferred value: 5
```

---

### Use Case 4: Goroutines (Advanced)

```go
func main() {
    for i := 0; i < 3; i++ {
        go func(n int) {
            fmt.Println("Goroutine:", n)
        }(i)  // Pass i to avoid closure issue
    }
    
    time.Sleep(time.Second)
}
```

**Good for:** Concurrent operations with closures

---

## Common Mistakes

### Mistake 1: Forgetting Invocation Parentheses

❌ **Wrong:**
```go
func main() {
    func(a, b int) {
        fmt.Println(a + b)
    }  // ← Missing () - function not invoked!
}
```

**Error:** Function defined but never executed!

✅ **Correct:**
```go
func main() {
    func(a, b int) {
        fmt.Println(a + b)
    }(5, 7)  // ← Invocation!
}
```

---

### Mistake 2: Trying to Store Without Variable

❌ **Wrong:**
```go
package main

func(a, b int) {  // ← ERROR! Can't stand alone
    fmt.Println(a + b)
}
```

✅ **Correct options:**

**Option 1: IIFE**
```go
func main() {
    func(a, b int) {
        fmt.Println(a + b)
    }(5, 7)
}
```

**Option 2: Assign to variable (later topic)**
```go
var myFunc = func(a, b int) {
    fmt.Println(a + b)
}
```

---

### Mistake 3: Wrong Parameter Count

❌ **Wrong:**
```go
func main() {
    func(a, b int) {
        fmt.Println(a + b)
    }(5)  // ← Missing second argument!
}
```

**Error:** `not enough arguments`

✅ **Correct:**
```go
func main() {
    func(a, b int) {
        fmt.Println(a + b)
    }(5, 7)  // ← Both arguments provided
}
```

---

### Mistake 4: Confusing with Named Functions

❌ **Wrong thinking:**
```
"I can invoke an anonymous function later by name"
```

❌ **No!** Without a name or variable, you CAN'T invoke it later!

✅ **Correct understanding:**

**IIFE = Use immediately, then gone:**
```go
func() {
    fmt.Println("I run once and disappear!")
}()
```

**Named function = Can invoke multiple times:**
```go
func greet() {
    fmt.Println("Hello!")
}

greet()  // Call 1
greet()  // Call 2
greet()  // Call 3
```

---

### Mistake 5: Not Understanding Use Cases

❌ **Bad use:**
```go
func main() {
    // Using IIFE for something you need multiple times
    func() {
        fmt.Println("Hello")
    }()
    
    // Can't call it again! Have to rewrite!
    func() {
        fmt.Println("Hello")  // Duplicate code!
    }()
}
```

✅ **Good use:**
```go
func main() {
    // If you need it multiple times, use named function
    greet := func() {
        fmt.Println("Hello")
    }
    
    greet()  // Call 1
    greet()  // Call 2
}
```

---

## Practice Exercises

### Exercise 1: Basic IIFE

Create an IIFE that:
- Takes two numbers
- Prints their difference

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    func(a, b int) {
        fmt.Println("Difference:", a-b)
    }(10, 3)
}
```

**Output:** `Difference: 7`

</details>

---

### Exercise 2: IIFE with Multiple Operations

Create an IIFE that:
- Takes a name and age
- Prints a greeting
- Prints eligibility to vote (age >= 18)

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    func(name string, age int) {
        fmt.Printf("Hello, %s!\n", name)
        
        if age >= 18 {
            fmt.Println("You can vote!")
        } else {
            fmt.Println("You cannot vote yet.")
        }
    }("Bob", 20)
}
```

**Output:**
```
Hello, Bob!
You can vote!
```

</details>

---

### Exercise 3: IIFE with Return

Create an IIFE that calculates area of rectangle and stores result:

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    area := func(length, width float64) float64 {
        return length * width
    }(10.5, 5.2)
    
    fmt.Println("Area:", area)
}
```

**Output:** `Area: 54.6`

</details>

---

### Exercise 4: Multiple IIFEs

Create three IIFEs in sequence that:
1. Print "Starting..."
2. Calculate and print sum of 10 and 20
3. Print "Done!"

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    func() {
        fmt.Println("Starting...")
    }()
    
    func(a, b int) {
        fmt.Println("Sum:", a+b)
    }(10, 20)
    
    func() {
        fmt.Println("Done!")
    }()
}
```

**Output:**
```
Starting...
Sum: 30
Done!
```

</details>

---

### Exercise 5: IIFE Scope Isolation

Show that IIFE creates its own scope:

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
    x := 100
    
    func(x int) {
        fmt.Println("Inside IIFE:", x)
        x = 999  // Modifies parameter, not outer x
    }(x)
    
    fmt.Println("Outside IIFE:", x)
}
```

**Output:**
```
Inside IIFE: 100
Outside IIFE: 100
```

**Explanation:** IIFE's `x` is a parameter, shadows outer `x`

</details>

---

### Exercise 6: IIFE vs Named Function

Compare performance of IIFE vs named function:

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

// Named function
func namedAdd(a, b int) int {
    return a + b
}

func main() {
    // Named function - can call multiple times
    fmt.Println("Named 1:", namedAdd(5, 3))
    fmt.Println("Named 2:", namedAdd(10, 7))
    
    // IIFE - define and use once
    result := func(a, b int) int {
        return a + b
    }(15, 8)
    fmt.Println("IIFE:", result)
}
```

**Output:**
```
Named 1: 8
Named 2: 17
IIFE: 23
```

**Key difference:** Named can be reused, IIFE is one-time!

</details>

---

## Summary

### Key Takeaways

1. ✅ **Anonymous Function = Function without a name**
2. ✅ **IIFE = Immediately Invoked Function Expression**
3. ✅ **Go doesn't allow standalone anonymous functions**
4. ✅ **Must invoke immediately or assign to variable**
5. ✅ **"Invoke" is the proper term (not "call")**
6. ✅ **Expression = A statement/line of code**
7. ✅ **IIFEs are interview questions!**

---

### Function Types Progress

```
✅ Chapter 13: Standard/Named Functions
✅ Chapter 14: Init Functions
✅ Chapter 15: Anonymous Functions & IIFE ← YOU ARE HERE
⏭️  Next: Function Expressions (assigned to variables)
⏭️  Then: Higher-Order Functions
⏭️  Then: Callbacks
⏭️  And more...
```

---

### Visual Summary

```
NAMED FUNCTION:
    func add(a, b int) { }
         ↑
      Has NAME
         ↓
    Can call by name: add(5, 7)

ANONYMOUS FUNCTION:
    func(a, b int) { }
         ↑
      NO NAME
         ↓
    Must invoke immediately: func(a, b int) { }(5, 7)
                                                   ↑
                                                 IIFE!
```

---

### IIFE Structure

```go
func(parameters) {
    // function body
}(arguments)
  ↑
  Immediate invocation
  That's what makes it "IIFE"!
```

---

### When to Use

| Scenario | Use Named | Use IIFE |
|----------|-----------|----------|
| Need multiple times | ✅ | ❌ |
| One-time operation | ❌ | ✅ |
| Need to store/pass | ✅ | ❌ |
| Scope isolation | ❌ | ✅ |
| Simple and immediate | ❌ | ✅ |

---

### Interview Preparation

**Common questions:**
1. ❓ "What is an anonymous function?"
2. ❓ "What does IIFE stand for?"
3. ❓ "Why can't we leave anonymous functions standalone in Go?"
4. ❓ "What's the difference between 'invoke' and 'call'?"
5. ❓ "What is an expression in programming?"

**You can answer all of these now!** 🎓

---

### The Interview Channel

> **"I have a Discord group with an 'Interview Questions and Answers' section. I will post questions from this topic there with answers and video links. Check that channel before interviews and no one can stop you!"**

**Prepare with:**
- ✅ Questions from each chapter
- ✅ Detailed answers
- ✅ Video references
- ✅ Practice problems

---

## What's Next?

### Chapter 16 Preview: Function Expressions (Variables)

Now that we know IIFE, let's learn to store anonymous functions!

**Topics covered:**
- **Assigning functions to variables** - Give anonymous functions names
- **Function as first-class citizens** - Functions are values
- **Type of function variables** - What type are they?
- **Calling function variables** - How to invoke them
- **Passing functions around** - Functions as data

**The contrast:**
```go
// IIFE (today) - Use once immediately
func(a, b int) int {
    return a + b
}(5, 7)

// Function Expression (next) - Store and reuse
add := func(a, b int) int {
    return a + b
}
add(5, 7)  // Call it
add(10, 3) // Call it again!
```

---

### Why This Progression Matters

```
Chapter 13: Standard Functions (WITH name, stored globally)
Chapter 14: Init Functions (SPECIAL name, auto-invoked)
Chapter 15: Anonymous Functions (NO name, IIFE) ← YOU ARE HERE
Chapter 16: Function Expressions (Anonymous → Variable) ← NEXT!
```

**Each builds on the last!** 🏗️

---

### The Journey Continues

**Understanding levels:**
```
Level 1: Functions have names (Standard)
Level 2: Special function without call (Init)
Level 3: Functions can have no name (Anonymous/IIFE)
Level 4: Functions can be values (Function Expressions) ← Next!
Level 5: Functions can take functions (Higher-Order) ← Later!
```

---

### Final Wisdom

**The Anonymous Concept:**
> **"Anonymous = Unknown person. You don't know their name. If a function has no name, you can't catch it, can't hold it - it's like a ghost!"**

**The Invoke Terminology:**
> **"Say 'invoke' not 'call' - it's the proper programming term! Even though I sometimes say 'call' or 'execute' to help you understand, remember: INVOKE is correct!"**

**The Expression Understanding:**
> **"Expression = A line of code. Variable declaration = expression. If statement = expression. Function definition = expression. Function invocation = expression. IIFE = expression!"**

**The Interview Reality:**
> **"These are interview questions! IIFE, anonymous functions, expressions - they WILL ask you these terms. Know them!"**

**The Practical Use:**
> **"IIFE is useful for one-time operations. Define it, use it immediately, and it's gone. Like a ghost - appears, does its job, disappears!"**

---

### Quick Reference Card

```go
// ANONYMOUS FUNCTION (IIFE)
func(parameters) returnType {
    // body
}(arguments)  ← Must invoke immediately!

// TEMPLATE
func(a, b int) {
    fmt.Println(a + b)
}(5, 7)

// KEY POINTS:
✅ No function name
✅ Defined in main or other function
✅ Must invoke immediately with ()
✅ Takes parameters like normal function
✅ Can return values
✅ Called "IIFE" if invoked immediately
```

---

### Terminology Glossary

| Term | Meaning |
|------|---------|
| **Anonymous** | Without name |
| **IIFE** | Immediately Invoked Function Expression |
| **Invoke** | Call/execute a function (proper term) |
| **Expression** | A statement/line of code |
| **Standard Function** | Function with a name |
| **Named Function** | Same as standard function |

---

**See you in the next class!** 👋

**Allah Hafez!** 

---

*End of Chapter 15 - Anonymous Functions & IIFE Mastered!* 🎯

