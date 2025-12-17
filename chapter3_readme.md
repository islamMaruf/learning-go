# Go Programming Tutorial - Chapter 3
## If-Else and Switch Statements

### 📚 Table of Contents
1. [Introduction](#introduction)
2. [Why Do We Need Conditional Statements?](#why-do-we-need-conditional-statements)
3. [If-Else Statements](#if-else-statements)
4. [Comparison Operators](#comparison-operators)
5. [Logical Operators](#logical-operators)
6. [Switch-Case Statements](#switch-case-statements)
7. [Practical Examples](#practical-examples)
8. [Common Mistakes](#common-mistakes)
9. [Summary](#summary)

---

## Introduction

In this chapter, you'll learn how to make **decisions** in your programs. Just like in real life, programs need to make choices based on different conditions.

**What You'll Learn:**
- How to use if-else statements to make decisions
- Comparison operators (>, <, >=, <=, ==)
- Logical operators (&&, ||, !)
- Switch-case statements as an alternative
- Real-world applications of conditional logic

**Note:** This is a longer chapter, but it's fundamental to programming. Take your time!

---

## Why Do We Need Conditional Statements?

### Real-World Example: Facebook Login

When you log into Facebook:
1. You enter your email and password
2. The server receives your data
3. **Decision time:** Does your email and password match the database?
   - If **yes** → You're logged in ✅
   - If **no** → "Invalid credentials" error ❌

This decision-making is done using **conditional statements**!

### Programming Without Conditions vs With Conditions

**Without Conditions:**
```
Start → Action → End
```
The program always does the same thing.

**With Conditions:**
```
Start → Check Condition
         ├─→ If True: Do Action A
         └─→ If False: Do Action B
End
```
The program can take different paths based on conditions!

---

## If-Else Statements

### Basic Syntax

```go
if condition {
    // Code to execute if condition is true
} else {
    // Code to execute if condition is false
}
```

### Simple Example: Age Check

```go
package main

import "fmt"

func main() {
    age := 20
    
    if age > 18 {
        fmt.Println("You are eligible to be married")
    } else {
        fmt.Println("You are not eligible to be married")
    }
}
```

**Output:**
```
You are eligible to be married
```

**How it works:**

1. Variable `age` is set to 20
2. Condition checks: Is 20 > 18?
3. Yes! 20 is greater than 18
4. Execute code inside the `if` block
5. Skip the `else` block

---

### If-Else-If (Multiple Conditions)

```go
if condition1 {
    // Execute if condition1 is true
} else if condition2 {
    // Execute if condition2 is true
} else {
    // Execute if none of the above are true
}
```

### Example: Age Categories

```go
package main

import "fmt"

func main() {
    age := 20
    
    if age > 18 {
        fmt.Println("You are eligible to be married")
    } else if age < 18 {
        fmt.Println("You are not eligible to be married but you can love someone")
    } else {
        fmt.Println("You are just a teenager, not eligible to be married")
    }
}
```

**Flow Chart:**

```
age = 20
    ↓
Is age > 18? → YES → Print "eligible to be married"
    ↓ NO
Is age < 18? → Skip (already found true condition)
    ↓ NO
else → Skip (already found true condition)
```

---

## Comparison Operators

Comparison operators are used to compare values.

### Operator Table

| Operator | Meaning | Example | Result |
|----------|---------|---------|--------|
| `>` | Greater than | `20 > 18` | `true` |
| `<` | Less than | `5 < 18` | `true` |
| `>=` | Greater than or equal | `18 >= 18` | `true` |
| `<=` | Less than or equal | `18 <= 18` | `true` |
| `==` | Equal to | `18 == 18` | `true` |
| `!=` | Not equal to | `18 != 20` | `true` |

### Greater Than (>)

```go
age := 20

if age > 18 {
    fmt.Println("Age is greater than 18")
}
```

**Question:** Is 20 > 18?  
**Answer:** Yes! 20 is greater than 18 → Condition is `true`

---

### Less Than (<)

```go
age := 5

if age < 18 {
    fmt.Println("Age is less than 18")
}
```

**Question:** Is 5 < 18?  
**Answer:** Yes! 5 is less than 18 → Condition is `true`

---

### Greater Than or Equal (>=)

```go
age := 18

if age >= 18 {
    fmt.Println("You are eligible to be married")
}
```

**Breaking it down:**
- Is 18 > 18? No
- **OR** Is 18 == 18? **Yes!**
- At least one condition is true → Execute the code

**Think of >= as:** "Either bigger OR equal, both work!"

---

### Less Than or Equal (<=)

```go
age := 18

if age <= 18 {
    fmt.Println("You are not eligible to be married")
}
```

**Breaking it down:**
- Is 18 < 18? No
- **OR** Is 18 == 18? **Yes!**
- At least one condition is true → Execute the code

---

### Equal To (==)

**Important:** Use `==` (two equals) for comparison, not `=` (one equal)!

```go
age := 18

if age == 18 {
    fmt.Println("You are exactly 18 years old")
}
```

**Common Mistake:**
```go
// ❌ WRONG
if age = 18 {  // This is assignment, not comparison!
}

// ✅ CORRECT
if age == 18 {  // This is comparison
}
```

---

### Not Equal To (!=)

```go
age := 20

if age != 18 {
    fmt.Println("Your age is not 18")
}
```

**Question:** Is 20 != 18?  
**Answer:** Yes! 20 is not equal to 18 → Condition is `true`

---

## Logical Operators

Logical operators combine multiple conditions.

### Operator Table

| Operator | Name | Meaning | Symbol in Go |
|----------|------|---------|--------------|
| AND | Logical AND | Both conditions must be true | `&&` |
| OR | Logical OR | At least one condition must be true | `\|\|` |
| NOT | Logical NOT | Inverts the condition | `!` |

---

### AND Operator (&&)

**Meaning:** **BOTH** conditions must be true.

**Syntax:**
```go
if condition1 && condition2 {
    // Executes only if BOTH are true
}
```

**Example:**
```go
package main

import "fmt"

func main() {
    age := 20
    sex := "male"
    
    if age == 20 && sex == "male" {
        fmt.Println("You are ready to marry")
    }
}
```

**Truth Table:**

| Condition 1 | Condition 2 | Result |
|-------------|-------------|--------|
| `true` | `true` | ✅ `true` |
| `true` | `false` | ❌ `false` |
| `false` | `true` | ❌ `false` |
| `false` | `false` | ❌ `false` |

**Both must be true, otherwise false!**

---

### Example: Age AND Gender Check

```go
package main

import "fmt"

func main() {
    age := 20
    sex := "male"
    
    if age == 20 && sex == "male" {
        fmt.Println("You are ready to marry")
    }
}
```

**Step-by-step evaluation:**
1. Is `age == 20`? Yes, age is 20 → `true`
2. **AND**
3. Is `sex == "male"`? Yes, sex is "male" → `true`
4. `true AND true` = `true`
5. Execute the code block

**Output:**
```
You are ready to marry
```

---

### What if one condition fails?

```go
age := 18  // Changed from 20
sex := "male"

if age == 20 && sex == "male" {
    fmt.Println("You are ready to marry")
}
```

**Step-by-step evaluation:**
1. Is `age == 20`? No, age is 18 → `false`
2. **AND**
3. Is `sex == "male"`? Yes, sex is "male" → `true`
4. `false AND true` = `false`
5. Skip the code block

**Output:** (nothing)

---

### OR Operator (||)

**Meaning:** **AT LEAST ONE** condition must be true.

**Syntax:**
```go
if condition1 || condition2 {
    // Executes if AT LEAST ONE is true
}
```

**Truth Table:**

| Condition 1 | Condition 2 | Result |
|-------------|-------------|--------|
| `true` | `true` | ✅ `true` |
| `true` | `false` | ✅ `true` |
| `false` | `true` | ✅ `true` |
| `false` | `false` | ❌ `false` |

**At least one must be true!**

---

### Example: Senior Citizen OR Male

```go
package main

import "fmt"

func main() {
    age := 20
    sex := "male"
    
    if age > 60 || sex == "male" {
        fmt.Println("You are ready to marry")
    }
}
```

**Step-by-step evaluation:**
1. Is `age > 60`? No, age is 20 → `false`
2. **OR**
3. Is `sex == "male"`? Yes, sex is "male" → `true`
4. `false OR true` = `true` (at least one is true!)
5. Execute the code block

**Output:**
```
You are ready to marry
```

**Key Point:** Even though the first condition failed, the second one succeeded, so the overall result is `true`!

---

### NOT Operator (!)

**Meaning:** **Inverts** the boolean value.

**How it works:**
- `!true` becomes `false`
- `!false` becomes `true`

**Example:**
```go
package main

import "fmt"

func main() {
    isPretty := false
    
    if !isPretty {
        fmt.Println("Print something")
    }
}
```

**Step-by-step evaluation:**
1. `isPretty` is `false`
2. `!isPretty` inverts it to `true`
3. Condition is `true`
4. Execute the code block

**Output:**
```
Print something
```

---

### Understanding NOT with Examples

```go
// Example 1: NOT false = true
isPretty := false
if !isPretty {
    fmt.Println("This will print")  // ✅ Prints
}

// Example 2: NOT true = false
isPretty := true
if !isPretty {
    fmt.Println("This will NOT print")  // ❌ Doesn't print
}

// Example 3: Without NOT
isPretty := false
if isPretty {
    fmt.Println("This will NOT print")  // ❌ Doesn't print
}
```

**Analogy:** The NOT operator is like saying "opposite day" - everything means the opposite!

---

## Switch-Case Statements

Switch statements are an alternative to multiple if-else-if chains. They're cleaner when checking one variable against many values.

### Basic Syntax

```go
switch variable {
case value1:
    // Code if variable == value1
case value2, value3:
    // Code if variable == value2 OR value3
default:
    // Code if no cases match
}
```

---

### Simple Switch Example

```go
package main

import "fmt"

func main() {
    a := 3
    
    switch a {
    case 1:
        fmt.Println("a is 1")
    case 2, 3:
        fmt.Println("a is either 2 or 3")
    default:
        fmt.Println("a is neither 1, 2, nor 3")
    }
}
```

**Output:**
```
a is either 2 or 3
```

**How it works:**

```
Step 1: a = 3
Step 2: Check case 1: Is a == 1? No (3 ≠ 1)
Step 3: Check case 2, 3: Is a == 2 OR a == 3? Yes! (a is 3)
Step 4: Execute case 2, 3 code block
Step 5: Automatically exit switch (no need for break in Go!)
```

---

### Switch vs If-Else: Same Logic

**Using If-Else:**
```go
a := 3

if a == 1 {
    fmt.Println("a is 1")
} else if a == 2 || a == 3 {
    fmt.Println("a is either 2 or 3")
} else {
    fmt.Println("a is neither 1, 2, nor 3")
}
```

**Using Switch:**
```go
a := 3

switch a {
case 1:
    fmt.Println("a is 1")
case 2, 3:
    fmt.Println("a is either 2 or 3")
default:
    fmt.Println("a is neither 1, 2, nor 3")
}
```

**Result:** Both produce the same output! Switch is cleaner for multiple value checks.

---

### When to Use Switch vs If-Else

**Use Switch When:**
- Checking **one variable** against **many possible values**
- Values are **discrete** (specific numbers/strings)
- You want **cleaner, more readable** code

**Use If-Else When:**
- Checking **complex conditions** (ranges, multiple variables)
- Using **logical operators** (&&, ||)
- Conditions are **more dynamic**

---

### Switch with Different Values

```go
package main

import "fmt"

func main() {
    a := 1
    
    switch a {
    case 1:
        fmt.Println("a is 1")
    case 2, 3:
        fmt.Println("a is either 2 or 3")
    default:
        fmt.Println("a is neither 1, 2, nor 3")
    }
}
```

**Output:**
```
a is 1
```

**Explanation:** First case matched, so it executes and exits.

---

### Switch with Default Case

```go
package main

import "fmt"

func main() {
    a := 100
    
    switch a {
    case 1:
        fmt.Println("a is 1")
    case 2, 3:
        fmt.Println("a is either 2 or 3")
    default:
        fmt.Println("a is neither 1, 2, nor 3")
    }
}
```

**Output:**
```
a is neither 1, 2, nor 3
```

**Explanation:**
- Is a == 1? No (100 ≠ 1)
- Is a == 2 or 3? No (100 ≠ 2 and 100 ≠ 3)
- No cases matched → Execute `default` block

**Default = "None of the above"**

---

## Practical Examples

### Example 1: Login System

```go
package main

import "fmt"

func main() {
    username := "habib"
    password := "secret123"
    
    if username == "habib" && password == "secret123" {
        fmt.Println("Login successful!")
    } else {
        fmt.Println("Invalid credentials")
    }
}
```

**Output:**
```
Login successful!
```

---

### Example 2: Grade Calculator

```go
package main

import "fmt"

func main() {
    score := 85
    
    if score >= 90 {
        fmt.Println("Grade: A")
    } else if score >= 80 {
        fmt.Println("Grade: B")
    } else if score >= 70 {
        fmt.Println("Grade: C")
    } else if score >= 60 {
        fmt.Println("Grade: D")
    } else {
        fmt.Println("Grade: F")
    }
}
```

**Output:**
```
Grade: B
```

**Explanation:** Score is 85, which is >= 80 but < 90, so Grade B.

---

### Example 3: Day of Week (Switch)

```go
package main

import "fmt"

func main() {
    day := 3
    
    switch day {
    case 1:
        fmt.Println("Monday")
    case 2:
        fmt.Println("Tuesday")
    case 3:
        fmt.Println("Wednesday")
    case 4:
        fmt.Println("Thursday")
    case 5:
        fmt.Println("Friday")
    case 6, 7:
        fmt.Println("Weekend!")
    default:
        fmt.Println("Invalid day")
    }
}
```

**Output:**
```
Wednesday
```

---

### Example 4: Age and Gender Check

```go
package main

import "fmt"

func main() {
    age := 20
    gender := "male"
    
    if age >= 18 && gender == "male" {
        fmt.Println("Eligible for military service")
    } else if age >= 18 && gender == "female" {
        fmt.Println("Eligible for voting")
    } else {
        fmt.Println("Too young")
    }
}
```

**Output:**
```
Eligible for military service
```

---

### Example 5: Multiple OR Conditions

```go
package main

import "fmt"

func main() {
    country := "Bangladesh"
    
    if country == "Bangladesh" || country == "India" || country == "Pakistan" {
        fmt.Println("You are from South Asia")
    } else {
        fmt.Println("You are from outside South Asia")
    }
}
```

**Output:**
```
You are from South Asia
```

---

### Example 6: Nested If Statements

```go
package main

import "fmt"

func main() {
    age := 25
    hasLicense := true
    
    if age >= 18 {
        if hasLicense {
            fmt.Println("You can drive")
        } else {
            fmt.Println("You need a license")
        }
    } else {
        fmt.Println("You are too young to drive")
    }
}
```

**Output:**
```
You can drive
```

**Explanation:**
1. Check if age >= 18: Yes (25 >= 18)
2. Enter the if block
3. Check if hasLicense: Yes (true)
4. Print "You can drive"

---

## Common Mistakes

### ❌ Mistake 1: Using = Instead of ==

```go
// ❌ WRONG
age := 20
if age = 18 {  // This is assignment, not comparison!
    fmt.Println("Age is 18")
}

// ✅ CORRECT
if age == 18 {  // This is comparison
    fmt.Println("Age is 18")
}
```

**Remember:** `=` is for assignment, `==` is for comparison!

---

### ❌ Mistake 2: Confusing && and ||

```go
age := 15

// ❌ WRONG LOGIC
if age > 18 && age < 60 {  // Means BOTH must be true
    fmt.Println("Working age")
}
// Output: Nothing (15 is not > 18)

// ✅ CORRECT LOGIC
if age > 18 && age < 60 {
    fmt.Println("Working age")
}
// This is actually correct if you want BOTH conditions
```

**Key:**
- `&&` = BOTH must be true
- `||` = At least ONE must be true

---

### ❌ Mistake 3: Missing Curly Braces

```go
// ❌ WRONG (won't compile)
if age > 18
    fmt.Println("Adult")

// ✅ CORRECT
if age > 18 {
    fmt.Println("Adult")
}
```

**Go requires curly braces for if statements!**

---

### ❌ Mistake 4: Forgetting Default in Switch

```go
// Not necessarily wrong, but can be problematic
a := 100

switch a {
case 1:
    fmt.Println("One")
case 2:
    fmt.Println("Two")
// What if a is 100? Nothing happens!
}

// ✅ BETTER - Add default
switch a {
case 1:
    fmt.Println("One")
case 2:
    fmt.Println("Two")
default:
    fmt.Println("Other number")
}
```

---

### ❌ Mistake 5: Wrong String Comparison

```go
name := "Habib"

// ❌ WRONG (case-sensitive)
if name == "habib" {
    fmt.Println("Match")
}
// Output: Nothing (capital H vs lowercase h)

// ✅ CORRECT
if name == "Habib" {
    fmt.Println("Match")
}
// Output: Match
```

**Go is case-sensitive!** "Habib" ≠ "habib"

---

## Visual Summary

### If-Else Flow Chart

```
Start
  ↓
  Is condition true?
  ├─→ YES → Execute if block → End
  └─→ NO → Execute else block → End
```

### If-Else-If Flow Chart

```
Start
  ↓
  Is condition1 true?
  ├─→ YES → Execute block 1 → End
  └─→ NO
      ↓
      Is condition2 true?
      ├─→ YES → Execute block 2 → End
      └─→ NO → Execute else block → End
```

### Switch Flow Chart

```
Start
  ↓
  Check variable value
  ├─→ Matches case 1? → YES → Execute case 1 → End
  ├─→ Matches case 2? → YES → Execute case 2 → End
  ├─→ Matches case 3? → YES → Execute case 3 → End
  └─→ No match → Execute default → End
```

---

### Comparison Operators Quick Reference

```go
a := 10
b := 20

a > b   // false (10 is not greater than 20)
a < b   // true  (10 is less than 20)
a >= b  // false (10 is not >= 20)
a <= b  // true  (10 is <= 20)
a == b  // false (10 is not equal to 20)
a != b  // true  (10 is not equal to 20)
```

---

### Logical Operators Quick Reference

```go
// AND (&&) - Both must be true
true && true   // true
true && false  // false
false && true  // false
false && false // false

// OR (||) - At least one must be true
true || true   // true
true || false  // true
false || true  // true
false || false // false

// NOT (!) - Inverts the value
!true   // false
!false  // true
```

---

## Summary

### 🎯 Key Takeaways

1. **If-Else Statements:**
   - Used for making decisions in code
   - `if` executes when condition is true
   - `else` executes when condition is false
   - Can chain with `else if` for multiple conditions

2. **Comparison Operators:**
   - `>` Greater than
   - `<` Less than
   - `>=` Greater than or equal
   - `<=` Less than or equal
   - `==` Equal to (use two equals!)
   - `!=` Not equal to

3. **Logical Operators:**
   - `&&` AND - Both conditions must be true
   - `||` OR - At least one condition must be true
   - `!` NOT - Inverts the boolean value

4. **Switch Statements:**
   - Cleaner alternative to multiple if-else
   - Check one variable against many values
   - Use `default` for unmatched cases
   - No `break` needed in Go (automatic)

5. **Best Practices:**
   - Use `==` for comparison, not `=`
   - Always use curly braces `{}`
   - Add `default` case in switch statements
   - Keep conditions simple and readable

---

### 📝 Complete Example: All Concepts Together

```go
package main

import "fmt"

func main() {
    // Variables
    age := 25
    gender := "male"
    country := "Bangladesh"
    hasLicense := true
    
    // If-else with logical operators
    if age >= 18 && hasLicense {
        fmt.Println("✅ You can drive")
    } else {
        fmt.Println("❌ Cannot drive")
    }
    
    // Multiple conditions
    if (age >= 18 && gender == "male") || country == "USA" {
        fmt.Println("✅ Complex condition met")
    }
    
    // Switch statement
    dayOfWeek := 3
    switch dayOfWeek {
    case 1:
        fmt.Println("Monday")
    case 2:
        fmt.Println("Tuesday")
    case 3:
        fmt.Println("Wednesday")
    default:
        fmt.Println("Other day")
    }
    
    // NOT operator
    isChild := false
    if !isChild {
        fmt.Println("✅ Not a child")
    }
}
```

**Output:**
```
✅ You can drive
✅ Complex condition met
Wednesday
✅ Not a child
```

---

## Practice Exercises

### Exercise 1: Number Checker
Write a program that checks if a number is positive, negative, or zero.

```go
num := -5
// Your code here
// Expected output: "Negative number"
```

---

### Exercise 2: Even or Odd
Check if a number is even or odd.

**Hint:** Use the modulo operator `%`. If `num % 2 == 0`, it's even.

```go
num := 7
// Your code here
// Expected output: "Odd number"
```

---

### Exercise 3: Grade with Switch
Convert a numeric grade to letter grade using switch:
- 90-100: A
- 80-89: B
- 70-79: C
- 60-69: D
- Below 60: F

---

### Exercise 4: Login System
Create a simple login system that checks username and password.

```go
username := "admin"
password := "12345"
// Check if both match the stored values
// Print "Login successful" or "Invalid credentials"
```

---

### Exercise 5: Leap Year Checker
A year is a leap year if:
- Divisible by 4 AND
- (Not divisible by 100 OR divisible by 400)

```go
year := 2024
// Your code here
// Expected output: "2024 is a leap year"
```

---

## Getting Help

**Don't understand something? That's okay!**

### Resources:
1. ✅ **ChatGPT** - Ask for explanations 1000 times if needed! It won't get tired
2. ✅ **Practice** - Write 10-20 different if-else programs
3. ✅ **Discord Community** - Ask your questions
4. ✅ **Facebook Group** - Get help from peers
5. ✅ **Direct Message** - Contact the instructor

**Remember:**
- Everyone learns at their own pace
- Making mistakes is part of learning
- If you need help 1000 times, you'll get help 1000 times!
- Study with friends - at least one will understand and can explain
- The instructor is patient and will help you understand 🤝

---

## What's Next?

In the next chapters, you'll learn:
- **Loops** - Repeating code (for, while)
- **Arrays and Slices** - Storing multiple values
- **Functions** - Organizing reusable code
- **Structs** - Creating custom data types

**Note:** You can survive with just if-else! Switch is optional but good to know. Choose whichever you prefer! 

Keep practicing! 🚀

---

*Based on Go Programming Tutorial - Chapter 3*  
*Topics: If-Else, Switch-Case, Conditional Logic, Decision Making*
