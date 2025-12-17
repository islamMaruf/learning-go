# Go Programming Tutorial - Chapter 2
## Variables and Data Types

### 📚 Table of Contents
1. [Introduction](#introduction)
2. [What is a Variable?](#what-is-a-variable)
3. [How Variables Work in Memory](#how-variables-work-in-memory)
4. [Data Types in Go](#data-types-in-go)
5. [Declaring Variables](#declaring-variables)
6. [Constants](#constants)
7. [Practical Examples](#practical-examples)
8. [Common Mistakes](#common-mistakes)
9. [Summary](#summary)

---

## Introduction

This chapter is slightly longer than previous ones, but it's fundamental to understanding Go programming. Whether you're a complete beginner or have programming experience, this lesson will solidify your understanding of variables and data types.

**What You'll Learn:**
- What variables are and why we need them
- How computer memory works with variables
- Different data types in Go
- Multiple ways to declare variables
- When and how to use constants

---

## What is a Variable?

### The Container Concept

A **variable** is like a **container** - think of it as:
- A bucket 🪣
- A bowl 🥣
- A cup ☕
- A glass 🥤

Just like these containers hold different things (water, food, coffee), variables hold **data**.

```
Container (Real World)    →    Variable (Programming)
─────────────────────────      ──────────────────────
Cup holds coffee              Variable holds numbers
Bucket holds water            Variable holds text
Bowl holds rice               Variable holds true/false
```

**Key Point:** In programming, especially in Go, everything revolves around **data**. Variables are how we store and manage that data.

---

## How Variables Work in Memory

### Computer Components Overview

```
┌────────────────────────────────┐
│       Computer                  │
│  ┌──────────┐                  │
│  │   CPU    │  (Processor)     │
│  └──────────┘                  │
│  ┌──────────┐                  │
│  │   RAM    │  (Memory)        │
│  └──────────┘                  │
│  ┌──────────┐                  │
│  │ Hard Disk│  (Storage)       │
│  └──────────┘                  │
└────────────────────────────────┘
```

### RAM: Where Variables Live

When you declare a variable, it's stored in **RAM (Random Access Memory)**.

**RAM Structure (Simplified):**

```
RAM Memory
┌─────┬─────┬─────┬─────┬─────┬─────┐
│     │     │     │     │     │     │  ← Empty cells
└─────┴─────┴─────┴─────┴─────┴─────┘
```

Each box is a **memory cell** where data can be stored.

### Variable Declaration in Memory

When you write:
```go
var a int = 10
```

**What happens:**

1. Go finds an empty cell in RAM
2. Labels that cell with the name `a`
3. Stores the value `10` in that cell

```
RAM Memory
┌─────┬─────┬─────┬─────┬─────┬─────┐
│  a  │     │     │     │     │     │
│ 10  │     │     │     │     │     │
└─────┴─────┴─────┴─────┴─────┴─────┘
  ↑
  Variable "a" occupies this cell
```

### Changing Variable Values

```go
var a int = 10   // Cell contains 10
a = 20           // Old value kicked out, now contains 20
a = 50           // 20 is replaced, now contains 50
```

**Memory Changes:**

```
Step 1: var a int = 10
┌─────┐
│  a  │
│ 10  │
└─────┘

Step 2: a = 20
┌─────┐
│  a  │
│ 20  │  ← 10 is replaced
└─────┘

Step 3: a = 50
┌─────┐
│  a  │
│ 50  │  ← 20 is replaced
└─────┘
```

**Analogy:** It's like putting someone in a dark room. First, you put Soleman in. Then you remove Soleman and put Mokles in. When you check the room, you'll only find the last person you put in!

---

## Data Types in Go

### Overview of Data Types

```
Data Types
├── Numeric
│   ├── Integer (whole numbers)
│   └── Float (decimal numbers)
├── Boolean (true/false)
└── String (text)
```

### 1. Numeric Types

#### Integer Numbers

**Definition:** Whole numbers without decimals

**Examples:**
- `10`
- `100`
- `1005`
- `-2`
- `-5`

**Go Type:** `int` (short for "integer")

**Variations:**
```
int     → General integer
int8    → 8-bit integer
int16   → 16-bit integer
int32   → 32-bit integer
int64   → 64-bit integer

uint    → Unsigned integer (positive only)
uint8   → 8-bit unsigned integer
uint16  → 16-bit unsigned integer
uint32  → 32-bit unsigned integer
uint64  → 64-bit unsigned integer
```

**Note:** The numbers (8, 16, 32, 64) relate to how much memory is used. You'll learn more about this when studying operating systems. **For now, just use `int`.**

---

#### Floating-Point Numbers

**Definition:** Numbers with decimal points

**Examples:**
- `10.5`
- `3.14`
- `99.99`

**Go Type:** `float32` or `float64`

**Common Usage:** `float32` (32-bit floating-point number)

---

### 2. Boolean Type

**Definition:** Represents truth values

**Only Two Possible Values:**
- `true`
- `false`

**Go Type:** `bool` (short for "boolean")

**Usage Examples:**
```go
var isLoggedIn bool = true
var hasPermission bool = false
```

---

### 3. String Type

**Definition:** Text data enclosed in double quotes

**Examples:**
- `"Hello, World!"`
- `"My name is Habib"`
- `"100"` (this is a string, not a number!)

**Go Type:** `string`

**Important:** Strings MUST use double quotes (`"`), not single quotes (`'`)

---

### Data Types Summary Table

| Type Category | Go Type | Example Values | Description |
|--------------|---------|----------------|-------------|
| **Integer** | `int` | `10`, `-5`, `1000` | Whole numbers |
| **Float** | `float32` | `10.5`, `3.14`, `99.99` | Decimal numbers |
| **Boolean** | `bool` | `true`, `false` | True or False only |
| **String** | `string` | `"Hello"`, `"Go"` | Text in quotes |

---

## Declaring Variables

Go provides multiple ways to declare variables. You only need to master **one way**, but knowing all helps you understand others' code.

### Method 1: Full Declaration with Type

```go
var x int = 10
```

**Breaking it down:**

| Part | Meaning |
|------|---------|
| `var` | Keyword indicating a variable declaration |
| `x` | Variable name (you choose this) |
| `int` | Data type (integer) |
| `=` | Assignment operator |
| `10` | Value to store |

**Example:**
```go
package main

import "fmt"

func main() {
    var x int = 10
    fmt.Println(x)  // Output: 10
}
```

---

### Method 2: Type Inference

Go can automatically detect the type based on the value!

```go
var a = 10          // Go knows this is int
var b = 10.5        // Go knows this is float64
var c = "Hello"     // Go knows this is string
var d = true        // Go knows this is bool
```

**Example:**
```go
package main

import "fmt"

func main() {
    var a = 10
    var b = 40.34
    var c = "Hello, World!"
    var d = true
    
    fmt.Println(a)  // 10
    fmt.Println(b)  // 40.34
    fmt.Println(c)  // Hello, World!
    fmt.Println(d)  // true
}
```

---

### Method 3: Short Declaration (Most Common)

The **`:=`** operator is shorthand for declaring variables.

```go
a := 10
b := 40.34
c := "Hello"
d := true
```

**Rules:**
- `:=` can **only be used inside functions**
- `:=` automatically infers the type
- **Most Go programmers prefer this method**

**Example:**
```go
package main

import "fmt"

func main() {
    a := 10
    fmt.Println(a)  // Output: 10
}
```

---

### Method 4: Declaration Without Initial Value

```go
var x int        // Declares x as int, default value is 0
var name string  // Declares name as string, default value is ""
var flag bool    // Declares flag as bool, default value is false
```

**Default Values:**

| Type | Default Value |
|------|---------------|
| `int` | `0` |
| `float32` | `0.0` |
| `bool` | `false` |
| `string` | `""` (empty string) |

---

### Variable Declaration Rules

✅ **Can do:**
```go
a := 10        // First time declaration
a = 20         // Reassigning value (no colon!)
a = 50         // Reassigning again
```

❌ **Cannot do:**
```go
a := 10        // First declaration
a := 20        // ERROR! Cannot redeclare
```

**Remember:** Use `:=` **only for the first declaration**. After that, use `=` for reassignment.

---

## Constants

### What is a Constant?

A **constant** is a variable whose value **cannot be changed** after declaration.

**Keyword:** `const`

**Syntax:**
```go
const PI = 3.14159
```

### Constant vs Variable

```go
// Variable - can be changed
var age = 25
age = 26          // ✅ OK
age = 27          // ✅ OK

// Constant - cannot be changed
const PI = 3.14159
PI = 3.14         // ❌ ERROR! Cannot assign to constant
```

### Example

```go
package main

import "fmt"

func main() {
    const PI = 3.14159
    const AppName = "MyApp"
    
    fmt.Println(PI)       // 3.14159
    fmt.Println(AppName)  // MyApp
    
    // PI = 3.14          // This would cause an error
}
```

**Use Cases for Constants:**
- Mathematical constants (`PI`, `E`)
- Configuration values that shouldn't change
- API keys (though use environment variables in production)
- Maximum/minimum limits

**Memory Analogy:** Once you put something in a constant's "room" and lock it, you can never change it!

---

## Practical Examples

### Example 1: Basic Variable Declaration

```go
package main

import "fmt"

func main() {
    // Different ways to declare the same thing
    var x int = 10      // Method 1
    var y = 10          // Method 2
    z := 10             // Method 3 (most common)
    
    fmt.Println(x)      // 10
    fmt.Println(y)      // 10
    fmt.Println(z)      // 10
}
```

---

### Example 2: Working with Different Data Types

```go
package main

import "fmt"

func main() {
    // Integer
    var age int = 25
    
    // Float
    var price float32 = 99.99
    
    // Boolean
    var isStudent bool = true
    
    // String
    var name string = "Habib"
    
    fmt.Println("Age:", age)
    fmt.Println("Price:", price)
    fmt.Println("Is Student:", isStudent)
    fmt.Println("Name:", name)
}
```

**Output:**
```
Age: 25
Price: 99.99
Is Student: true
Name: Habib
```

---

### Example 3: Reassigning Variables

```go
package main

import "fmt"

func main() {
    a := 10
    fmt.Println("Initial value:", a)  // 10
    
    a = 20
    fmt.Println("After first change:", a)  // 20
    
    a = 50
    fmt.Println("After second change:", a)  // 50
}
```

**Output:**
```
Initial value: 10
After first change: 20
After second change: 50
```

**What's happening:**
1. `a` is declared with value 10
2. Value 10 is replaced by 20
3. Value 20 is replaced by 50
4. Final value is 50

---

### Example 4: Type Mismatch Error

```go
package main

import "fmt"

func main() {
    a := true          // a is declared as bool
    fmt.Println(a)     // true
    
    a = false          // ✅ OK - false is also bool
    fmt.Println(a)     // false
    
    // a = "Habib"     // ❌ ERROR! Cannot assign string to bool
}
```

**Important:** Once a variable's type is set, you cannot change its type!

```
Variable "a" is LOYAL!
Once it's boolean, it stays boolean forever! ❤️
```

---

### Example 5: Multiple Variables

```go
package main

import "fmt"

func main() {
    // Declare multiple variables
    var (
        name   string = "Habib"
        age    int    = 25
        salary float32 = 50000.50
    )
    
    fmt.Println("Name:", name)
    fmt.Println("Age:", age)
    fmt.Println("Salary:", salary)
}
```

**Alternative (short form):**
```go
name, age, salary := "Habib", 25, 50000.50
```

---

### Example 6: Understanding Variable Scope

```go
package main

import "fmt"

func main() {
    x := 100
    fmt.Println("Before:", x)    // 100
    
    x = 50
    fmt.Println("After:", x)     // 50
    
    x = 109
    // This changes x, but we don't print it
    
    // Program ends, final value of x is 109 (but not printed)
}
```

**Output:**
```
Before: 100
After: 50
```

**Explanation:** The value 109 is assigned to `x`, but since there's no `fmt.Println()` after it, we never see it!

---

## Common Mistakes

### ❌ Mistake 1: Using := for Reassignment

```go
// ❌ WRONG
a := 10
a := 20  // Error: no new variables on left side of :=

// ✅ CORRECT
a := 10
a = 20   // Use = for reassignment, not :=
```

---

### ❌ Mistake 2: Mixing Data Types

```go
// ❌ WRONG
var x int = 10
x = "Hello"  // Error: cannot use string as int

// ✅ CORRECT
var x int = 10
x = 20       // Both are integers
```

---

### ❌ Mistake 3: Changing Constants

```go
// ❌ WRONG
const PI = 3.14
PI = 3.14159  // Error: cannot assign to constant

// ✅ CORRECT
var pi = 3.14
pi = 3.14159  // OK - it's a variable
```

---

### ❌ Mistake 4: Wrong Quote Type for Strings

```go
// ❌ WRONG
name := 'Habib'  // Single quotes - ERROR

// ✅ CORRECT
name := "Habib"  // Double quotes
```

---

### ❌ Mistake 5: Declaring Without Using

```go
// ❌ WRONG - Go doesn't allow unused variables
package main

func main() {
    x := 10  // Declared but never used - ERROR
}

// ✅ CORRECT
package main

import "fmt"

func main() {
    x := 10
    fmt.Println(x)  // Now it's used
}
```

---

## Visual Summary

### Variable Declaration Methods

```go
// Method 1: Full declaration
var x int = 10

// Method 2: Type inference
var x = 10

// Method 3: Short declaration (most popular)
x := 10

// Method 4: Declaration without value
var x int  // Default value: 0
```

### Data Types Quick Reference

```go
// Integers
var age int = 25
var count int = 100

// Floats
var price float32 = 99.99
var temperature float32 = 36.5

// Booleans
var isActive bool = true
var hasError bool = false

// Strings
var name string = "Habib"
var message string = "Hello, World!"
```

### Memory Model Recap

```
┌──────────────────────────────────────────┐
│            Computer RAM                   │
├──────┬──────┬──────┬──────┬──────┬──────┤
│  a   │  b   │  c   │      │      │      │
│  10  │ 10.5 │"Hi"  │      │      │      │
└──────┴──────┴──────┴──────┴──────┴──────┘
   ↑      ↑      ↑
   int   float  string
```

---

## Summary

### 🎯 Key Takeaways

1. **Variables are Containers:**
   - Store data in memory (RAM)
   - Each variable occupies a memory cell
   - Variable name = cell name
   - Variable value = cell contents

2. **Data Types:**
   - **Numeric:** `int` (whole), `float32` (decimal)
   - **Boolean:** `bool` (true/false)
   - **String:** `string` (text in quotes)

3. **Declaration Methods:**
   - `var x int = 10` - Full declaration
   - `var x = 10` - Type inference
   - `x := 10` - Short declaration (most common)
   - `var x int` - Declaration with default value

4. **Constants:**
   - Use `const` keyword
   - Value cannot be changed
   - Good for fixed values like PI, configuration

5. **Important Rules:**
   - Use `:=` only for **first declaration**
   - Use `=` for **reassignment**
   - Cannot change variable type once set
   - Strings use **double quotes** only

---

### 📝 Complete Example

```go
package main

import "fmt"

func main() {
    // Constants
    const PI = 3.14159
    const AppName = "MyGoApp"
    
    // Variables - Different types
    var age int = 25
    var price float32 = 99.99
    var isStudent bool = true
    var name string = "Habib"
    
    // Short declaration (most common)
    city := "Dhaka"
    year := 2025
    
    // Print everything
    fmt.Println("App Name:", AppName)
    fmt.Println("PI:", PI)
    fmt.Println("Name:", name)
    fmt.Println("Age:", age)
    fmt.Println("City:", city)
    fmt.Println("Price:", price)
    fmt.Println("Is Student:", isStudent)
    fmt.Println("Year:", year)
    
    // Reassigning variables
    age = 26
    city = "Chittagong"
    fmt.Println("Updated Age:", age)
    fmt.Println("Updated City:", city)
}
```

---

## Practice Exercises

### Exercise 1: Declare Variables
Create variables for:
- Your name (string)
- Your age (int)
- Your height in meters (float32)
- Whether you're a student (bool)

Print all of them.

---

### Exercise 2: Variable Reassignment
```go
// Start with these
x := 100
y := 200

// Swap their values
// After swapping: x should be 200, y should be 100
```

**Hint:** You'll need a temporary variable!

---

### Exercise 3: Constants Practice
Create constants for:
- Days in a week (7)
- Hours in a day (24)
- Your birth year

Try to change one of them and observe the error.

---

### Exercise 4: Type Exploration
Experiment with these:
```go
a := 10
b := 10.5
c := "10"

// What are the types of a, b, c?
// Can you add a + b?
// Can you add a + c?
```

---

## Getting Help

**If you encounter problems:**

1. ✅ Read error messages carefully
2. ✅ Check your syntax (quotes, colons, semicolons)
3. ✅ Make sure variables are declared before use
4. ✅ Verify you're using the correct data type

**Resources:**
- 💬 Discord community
- 📘 Facebook group
- 💼 LinkedIn messages
- 🎥 YouTube comments

**Remember:** There's a volunteer team ready to help! If 10,000 people ask for help, 10,000 people will get help. Don't hesitate!

**Pro Tip:** Study with friends! If 5 friends are learning together, at least one will understand and can help explain to others. Learn together! 🤝

---

## What's Next?

In the next chapters, you'll learn:
- **Operators** - Mathematical and logical operations
- **User Input** - Getting data from users
- **Conditionals** - if/else statements
- **Loops** - Repeating code
- **Functions** - Organizing your code

Keep practicing! With just this knowledge of variables and data types, you can already start building simple projects! 🚀

---

*Based on Go Programming Tutorial - Chapter 2*  
*Topics: Variables, Data Types, Memory Management, Constants*
