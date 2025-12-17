# Go Programming Tutorial - Chapter 6
## More Function Examples

### 📚 Table of Contents
1. [Introduction](#introduction)
2. [Functions Without Parameters or Returns](#functions-without-parameters-or-returns)
3. [Functions with String Parameters](#functions-with-string-parameters)
4. [Printing Multiple Values](#printing-multiple-values)
5. [Function Variations Summary](#function-variations-summary)
6. [Practical Examples](#practical-examples)
7. [Common Mistakes](#common-mistakes)
8. [Summary](#summary)

---

## Introduction

In previous chapters, we learned about functions with parameters and return values, mostly working with numbers. In this chapter, we'll explore more function variations and work with different data types, especially **strings**!

**What You'll Learn:**
- Functions with no input and no output
- Functions that work with strings
- How to print multiple values at once
- Different function patterns and when to use them
- More practical examples

---

## Functions Without Parameters or Returns

### The Simplest Function

Just like `main()`, you can create functions that don't take any input and don't return anything!

```go
package main

import "fmt"

func printSomething() {
    fmt.Println("Education must be free")
}

func main() {
    printSomething()
}
```

**Output:**
```
Education must be free
```

---

### Understanding the Structure

```go
func printSomething() {
//                  ↑↑
//                  └─ Empty parentheses = No parameters
//   No return type = Returns nothing

    fmt.Println("Education must be free")
}
```

**Comparison with main():**

```go
// Your custom function
func printSomething() {
    fmt.Println("Education must be free")
}

// The main function (also no params, no return!)
func main() {
    printSomething()
}
```

**Both are similar!**
- Both use `func` keyword
- Both have empty `()` (no parameters)
- Both have no return type
- Both just execute code

---

### When to Use This Pattern

Use functions without parameters/returns when you want to:
- Execute a fixed task
- Display static messages
- Perform initialization
- Run setup code

**Examples:**

```go
func displayWelcomeMessage() {
    fmt.Println("Welcome to our application!")
    fmt.Println("Please login to continue")
}

func showMenu() {
    fmt.Println("1. Add Item")
    fmt.Println("2. Remove Item")
    fmt.Println("3. Exit")
}

func printSeparator() {
    fmt.Println("====================")
}
```

---

## Functions with String Parameters

So far, we've mostly worked with numbers (`int`, `float32`). Let's work with **strings**!

### Basic String Function

```go
package main

import "fmt"

func sayHello(name string) {
    fmt.Println("Welcome to the Golang course,", name)
}

func main() {
    sayHello("Habib")
}
```

**Output:**
```
Welcome to the Golang course, Habib
```

---

### Breaking Down the String Function

```go
func sayHello(name string) {
//            ↑    ↑
//            │    └─ Type: string (text data)
//            └─ Parameter name: name

    fmt.Println("Welcome to the Golang course,", name)
    //          ↑                                ↑
    //          String literal                   Variable
}
```

**Key Points:**
- `name` is a parameter of type `string`
- Must pass a string when calling
- `"Habib"` is a string literal (text in quotes)

---

### Why It Works

```go
func sayHello(name string) {
    fmt.Println("Welcome to the Golang course,", name)
}

func main() {
    sayHello("Habib")
    //       ↑
    //       This is a string!
}
```

**Flow:**
1. Call `sayHello("Habib")`
2. `name` parameter receives `"Habib"`
3. Inside function: `name = "Habib"`
4. Print: `"Welcome to the Golang course," + name`
5. Result: `Welcome to the Golang course, Habib`

---

### What Happens If You Don't Pass a Value?

```go
func sayHello(name string) {
    fmt.Println("Welcome to the Golang course,", name)
}

func main() {
    sayHello()  // ❌ ERROR! Missing argument
}
```

**Error:**
```
not enough arguments in call to sayHello
    have ()
    want (string)
```

**Why?** The function **requires** a string parameter. You must provide it!

---

### Must Match the Type

```go
func sayHello(name string) {
    fmt.Println("Welcome to the Golang course,", name)
}

func main() {
    sayHello("Habib")   // ✅ String - OK!
    sayHello(123)       // ❌ ERROR! Number, not string
    sayHello(true)      // ❌ ERROR! Boolean, not string
    sayHello(10.5)      // ❌ ERROR! Float, not string
}
```

**Rule:** The argument type must match the parameter type!

---

## Printing Multiple Values

### Using Commas in fmt.Println()

You can print multiple values by separating them with commas!

```go
package main

import "fmt"

func main() {
    name := "Habib"
    age := 25
    
    fmt.Println("Name:", name, "Age:", age)
}
```

**Output:**
```
Name: Habib Age: 25
```

**Note:** `fmt.Println()` automatically adds spaces between values!

---

### Multiple Values in Functions

```go
package main

import "fmt"

func sayHello(name string) {
    fmt.Println("Welcome to the Golang course,", name)
    //          ↑                                ↑
    //          String literal                   Variable
    //          Both separated by comma
}

func main() {
    sayHello("Habib")
}
```

**Output:**
```
Welcome to the Golang course, Habib
```

---

### How Comma Separation Works

```go
fmt.Println("Hello", "World", "!")
//          ↑      ↑       ↑
//          value1, value2, value3
```

**Output:**
```
Hello World !
```

**What happens:**
- Each value is converted to text
- Values are separated by spaces
- All printed on one line
- Newline added at the end

---

### Mixing Types in Println

You can mix different types!

```go
package main

import "fmt"

func main() {
    name := "Habib"
    age := 25
    isStudent := true
    gpa := 3.8
    
    fmt.Println("Name:", name, "Age:", age, "Student:", isStudent, "GPA:", gpa)
}
```

**Output:**
```
Name: Habib Age: 25 Student: true GPA: 3.8
```

**Magic!** `fmt.Println()` handles all types automatically!

---

### Without Commas vs With Commas

#### Without Space (String Concatenation)

```go
fmt.Println("Welcome" + name)
// No space between "Welcome" and name
```

**Output:**
```
WelcomeHabib
```

---

#### With Automatic Spacing (Comma Separation)

```go
fmt.Println("Welcome", name)
// Automatic space added!
```

**Output:**
```
Welcome Habib
```

---

#### Adding Manual Spaces

```go
fmt.Println("Welcome to the course,", " ", name)
//                                    ↑
//                            Extra space string
```

**Output:**
```
Welcome to the course,   Habib
```

---

### Printing Multiple Variables

```go
package main

import "fmt"

func greet(firstName string, lastName string, age int) {
    fmt.Println("Hello,", firstName, lastName)
    fmt.Println("You are", age, "years old")
}

func main() {
    greet("Habib", "Ahmed", 25)
}
```

**Output:**
```
Hello, Habib Ahmed
You are 25 years old
```

---

## Function Variations Summary

### All Four Patterns

```go
package main

import "fmt"

// Pattern 1: No input, no output
func pattern1() {
    fmt.Println("I take nothing, return nothing")
}

// Pattern 2: Input, no output
func pattern2(name string) {
    fmt.Println("Hello,", name)
}

// Pattern 3: No input, output
func pattern3() string {
    return "I return a value!"
}

// Pattern 4: Input and output
func pattern4(a int, b int) int {
    return a + b
}

func main() {
    pattern1()
    
    pattern2("Habib")
    
    message := pattern3()
    fmt.Println(message)
    
    result := pattern4(10, 20)
    fmt.Println(result)
}
```

**Output:**
```
I take nothing, return nothing
Hello, Habib
I return a value!
30
```

---

### Pattern Comparison Table

| Pattern | Input | Output | Use Case | Example |
|---------|-------|--------|----------|---------|
| **1** | ❌ | ❌ | Display messages, setup | `showMenu()` |
| **2** | ✅ | ❌ | Process and display | `greet(name)` |
| **3** | ❌ | ✅ | Generate values | `getDate()` |
| **4** | ✅ | ✅ | Transform data | `add(a, b)` |

---

### When to Use Each Pattern

#### Pattern 1: No Input, No Output
```go
func displayBanner() {
    fmt.Println("======================")
    fmt.Println("  Welcome to Go Lang  ")
    fmt.Println("======================")
}
```

**Use when:** Fixed output, no customization needed

---

#### Pattern 2: Input, No Output
```go
func greetUser(name string) {
    fmt.Println("Hello,", name, "!")
}
```

**Use when:** Displaying customized messages

---

#### Pattern 3: No Input, Output
```go
func getCurrentYear() int {
    return 2025
}
```

**Use when:** Generating or calculating values without external input

---

#### Pattern 4: Input and Output
```go
func calculateTax(amount float32) float32 {
    return amount * 0.15
}
```

**Use when:** Transforming input into output (most common!)

---

## Practical Examples

### Example 1: Greeting Functions

```go
package main

import "fmt"

// No parameters
func sayHi() {
    fmt.Println("Hi there!")
}

// One parameter
func greet(name string) {
    fmt.Println("Hello,", name, "!")
}

// Two parameters
func greetFull(firstName string, lastName string) {
    fmt.Println("Welcome,", firstName, lastName)
}

// Multiple types
func introduce(name string, age int, city string) {
    fmt.Println("My name is", name)
    fmt.Println("I am", age, "years old")
    fmt.Println("I live in", city)
}

func main() {
    sayHi()
    greet("Habib")
    greetFull("Habib", "Ahmed")
    introduce("Habib", 25, "Dhaka")
}
```

**Output:**
```
Hi there!
Hello, Habib !
Welcome, Habib Ahmed
My name is Habib
I am 25 years old
I live in Dhaka
```

---

### Example 2: Display Functions

```go
package main

import "fmt"

func showWelcome() {
    fmt.Println("╔════════════════════════╗")
    fmt.Println("║  Welcome to Go Course  ║")
    fmt.Println("╚════════════════════════╝")
}

func showMenu() {
    fmt.Println("Main Menu:")
    fmt.Println("1. Start Learning")
    fmt.Println("2. Practice")
    fmt.Println("3. Exit")
}

func showFooter() {
    fmt.Println("---")
    fmt.Println("© 2025 Go Tutorial")
}

func main() {
    showWelcome()
    showMenu()
    showFooter()
}
```

**Output:**
```
╔════════════════════════╗
║  Welcome to Go Course  ║
╚════════════════════════╝
Main Menu:
1. Start Learning
2. Practice
3. Exit
---
© 2025 Go Tutorial
```

---

### Example 3: User Info Display

```go
package main

import "fmt"

func displayUserInfo(name string, age int, email string, isActive bool) {
    fmt.Println("===== User Information =====")
    fmt.Println("Name:", name)
    fmt.Println("Age:", age, "years")
    fmt.Println("Email:", email)
    fmt.Println("Active:", isActive)
    fmt.Println("===========================")
}

func main() {
    displayUserInfo("Habib", 25, "habib@example.com", true)
    displayUserInfo("Sarah", 22, "sarah@example.com", false)
}
```

**Output:**
```
===== User Information =====
Name: Habib
Age: 25 years
Email: habib@example.com
Active: true
===========================
===== User Information =====
Name: Sarah
Age: 22 years
Email: sarah@example.com
Active: false
===========================
```

---

### Example 4: Message Functions

```go
package main

import "fmt"

func successMessage(action string) {
    fmt.Println("✓ Success:", action, "completed successfully!")
}

func errorMessage(action string) {
    fmt.Println("✗ Error:", action, "failed!")
}

func warningMessage(message string) {
    fmt.Println("⚠ Warning:", message)
}

func main() {
    successMessage("Login")
    errorMessage("File upload")
    warningMessage("Low disk space")
}
```

**Output:**
```
✓ Success: Login completed successfully!
✗ Error: File upload failed!
⚠ Warning: Low disk space
```

---

### Example 5: Personalized Greetings

```go
package main

import "fmt"

func morningGreet(name string) {
    fmt.Println("Good morning,", name, "! Have a great day!")
}

func eveningGreet(name string) {
    fmt.Println("Good evening,", name, "! Hope you had a good day!")
}

func nightGreet(name string) {
    fmt.Println("Good night,", name, "! Sleep well!")
}

func main() {
    name := "Habib"
    
    morningGreet(name)
    eveningGreet(name)
    nightGreet(name)
}
```

**Output:**
```
Good morning, Habib ! Have a great day!
Good evening, Habib ! Hope you had a good day!
Good night, Habib ! Sleep well!
```

---

### Example 6: Course Information

```go
package main

import "fmt"

func showCourseInfo(courseName string, instructor string, duration int) {
    fmt.Println("Course Name:", courseName)
    fmt.Println("Instructor:", instructor)
    fmt.Println("Duration:", duration, "weeks")
    fmt.Println("")
}

func main() {
    showCourseInfo("Go Programming", "Habib", 8)
    showCourseInfo("Python Basics", "Ahmed", 6)
    showCourseInfo("Web Development", "Sarah", 12)
}
```

**Output:**
```
Course Name: Go Programming
Instructor: Habib
Duration: 8 weeks

Course Name: Python Basics
Instructor: Ahmed
Duration: 6 weeks

Course Name: Web Development
Instructor: Sarah
Duration: 12 weeks

```

---

## Common Mistakes

### ❌ Mistake 1: Forgetting to Pass Required Parameter

```go
func greet(name string) {
    fmt.Println("Hello,", name)
}

func main() {
    greet()  // ❌ ERROR! Missing required argument
}
```

**Error:** `not enough arguments`

**Fix:**
```go
greet("Habib")  // ✅ CORRECT
```

---

### ❌ Mistake 2: Wrong Type of Argument

```go
func greet(name string) {
    fmt.Println("Hello,", name)
}

func main() {
    greet(123)  // ❌ ERROR! Number, not string
}
```

**Error:** `cannot use 123 (type int) as type string`

**Fix:**
```go
greet("Habib")  // ✅ CORRECT - Pass string
```

---

### ❌ Mistake 3: Using + Instead of Comma

```go
func greet(name string, age int) {
    fmt.Println("Name:" + name + "Age:" + age)  // ❌ ERROR!
    // Can't concatenate int with string using +
}
```

**Fix:**
```go
func greet(name string, age int) {
    fmt.Println("Name:", name, "Age:", age)  // ✅ CORRECT - Use commas
}
```

---

### ❌ Mistake 4: Wrong Order of Arguments

```go
func introduce(name string, age int) {
    fmt.Println(name, "is", age, "years old")
}

func main() {
    introduce(25, "Habib")  // ❌ ERROR! Wrong order
}
```

**Error:** `cannot use 25 (type int) as type string`

**Fix:**
```go
introduce("Habib", 25)  // ✅ CORRECT - Right order
```

---

### ❌ Mistake 5: Forgetting Quotes for Strings

```go
func greet(name string) {
    fmt.Println("Hello,", name)
}

func main() {
    greet(Habib)  // ❌ ERROR! Habib is not defined
}
```

**Fix:**
```go
greet("Habib")  // ✅ CORRECT - String needs quotes
```

---

### ❌ Mistake 6: Too Many Arguments

```go
func greet(name string) {
    fmt.Println("Hello,", name)
}

func main() {
    greet("Habib", "Ahmed")  // ❌ ERROR! Too many arguments
}
```

**Error:** `too many arguments in call to greet`

**Fix:**
```go
// Option 1: Call with one argument
greet("Habib")

// Option 2: Modify function to accept two
func greet(firstName string, lastName string) {
    fmt.Println("Hello,", firstName, lastName)
}
```

---

## Tips and Best Practices

### 1. Use Descriptive Function Names

```go
// ❌ Not clear
func f1() {
    fmt.Println("Welcome")
}

// ✅ Clear and descriptive
func displayWelcomeMessage() {
    fmt.Println("Welcome")
}
```

---

### 2. Keep Functions Focused

```go
// ❌ Does too many things
func doEverything() {
    fmt.Println("Welcome")
    fmt.Println("Menu")
    fmt.Println("Footer")
}

// ✅ Separate responsibilities
func showWelcome() {
    fmt.Println("Welcome")
}

func showMenu() {
    fmt.Println("Menu")
}

func showFooter() {
    fmt.Println("Footer")
}
```

---

### 3. Use Consistent Naming

```go
// ✅ GOOD - Consistent verb + noun pattern
func displayMenu()
func displayWelcome()
func displayFooter()

func printUserInfo()
func printCourseInfo()
func printResult()
```

---

### 4. Group Related Functions

```go
// User-related functions
func getUserName() string { }
func getUserAge() int { }
func displayUser() { }

// Menu-related functions
func showMainMenu() { }
func showSettingsMenu() { }
func showHelpMenu() { }
```

---

## Visual Summary

### Function Patterns

```
Pattern 1: No Input, No Output
┌──────────────────┐
│ func display() { │
│   fmt.Println()  │
│ }                │
└──────────────────┘

Pattern 2: Input, No Output
┌─────────────────────────┐
│ func greet(name string) {│
│   fmt.Println(name)      │
│ }                        │
└─────────────────────────┘

Pattern 3: No Input, Output
┌─────────────────────────┐
│ func getName() string {  │
│   return "Habib"         │
│ }                        │
└─────────────────────────┘

Pattern 4: Input and Output
┌──────────────────────────────┐
│ func add(a int, b int) int { │
│   return a + b               │
│ }                            │
└──────────────────────────────┘
```

---

## Summary

### 🎯 Key Takeaways

1. **Functions Without Parameters or Returns:**
   - Use `func name() { }` syntax
   - No input needed, no output given
   - Similar to `main()` function
   - Good for fixed, repetitive tasks

2. **String Parameters:**
   - Use `name string` to accept text input
   - Must pass string when calling
   - String literals use double quotes: `"text"`
   - Type must match when calling

3. **Multiple Values in Println:**
   - Separate values with commas
   - Automatic spacing between values
   - Can mix different types
   - All printed on one line

4. **Four Function Patterns:**
   - No input, no output: Display functions
   - Input, no output: Customized displays
   - No input, output: Value generators
   - Input and output: Data transformers

5. **Best Practices:**
   - Use descriptive names
   - Keep functions focused
   - Match parameter types
   - Provide required arguments

---

### 📝 Complete Example: All Concepts

```go
package main

import "fmt"

// Pattern 1: No input, no output
func printSeparator() {
    fmt.Println("====================")
}

// Pattern 2: String input, no output
func greetUser(name string) {
    fmt.Println("Welcome,", name, "!")
}

// Pattern 2: Multiple inputs, no output
func displayInfo(name string, age int, city string) {
    fmt.Println("Name:", name)
    fmt.Println("Age:", age)
    fmt.Println("City:", city)
}

// Pattern 3: No input, returns output
func getGreeting() string {
    return "Hello from Go!"
}

// Pattern 4: Input and output
func fullName(first string, last string) string {
    return first + " " + last
}

func main() {
    // Using Pattern 1
    printSeparator()
    
    // Using Pattern 2
    greetUser("Habib")
    displayInfo("Habib", 25, "Dhaka")
    
    // Using Pattern 3
    message := getGreeting()
    fmt.Println(message)
    
    // Using Pattern 4
    name := fullName("Habib", "Ahmed")
    fmt.Println("Full name:", name)
    
    printSeparator()
}
```

**Output:**
```
====================
Welcome, Habib !
Name: Habib
Age: 25
City: Dhaka
Hello from Go!
Full name: Habib Ahmed
====================
```

---

## Practice Exercises

### Exercise 1: Create Display Functions
Create three functions that display different messages:
- `showHeader()` - Displays a header
- `showFooter()` - Displays a footer  
- `showContent()` - Displays some content

---

### Exercise 2: Personalized Greeting
Create a function `personalGreet(name string, time string)` that prints:
```
Good [time], [name]!
```

---

### Exercise 3: Student Info
Create a function that displays student information:
```go
func displayStudent(name string, id int, major string, gpa float32)
```

---

### Exercise 4: Multiple Calls
Create a menu display function and call it 3 times to show different menus.

---

### Exercise 5: Mixed Types
Create a function that takes 5 different types of parameters and displays them all.

---

## What's Next?

In the next chapters, you'll learn:
- **Named Return Values** - Go's convenient syntax
- **Variadic Functions** - Variable number of parameters
- **Anonymous Functions** - Functions without names
- **Function as Values** - Passing functions around
- **Closures** - Functions inside functions

**Great job!** You now understand different function patterns and can work with various data types!

Keep practicing! 🚀✨

---

*Based on Go Programming Tutorial - Chapter 6*  
*Topics: Function Variations, String Parameters, Multiple Values, Display Functions*
