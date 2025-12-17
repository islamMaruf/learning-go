# Go Programming Tutorial - Chapter 5
## Functions with Return Values and Types

### 📚 Table of Contents
1. [Introduction](#introduction)
2. [Review: Functions Without Return](#review-functions-without-return)
3. [Functions with Return Values](#functions-with-return-values)
4. [Return Type Declaration](#return-type-declaration)
5. [Single Return Values](#single-return-values)
6. [Multiple Return Values](#multiple-return-values)
7. [How Return Works in Memory](#how-return-works-in-memory)
8. [Capturing Return Values](#capturing-return-values)
9. [Practical Examples](#practical-examples)
10. [Common Mistakes](#common-mistakes)
11. [Summary](#summary)

---

## Introduction

In the previous chapter, we learned about basic functions that perform actions but don't give us back any data. In this chapter, we'll learn about **functions that return values** - one of Go's most powerful features!

**What You'll Learn:**
- How to make functions return values
- Single return values
- Multiple return values (Go's special feature!)
- How return values work in memory
- Best practices for return values

---

## Review: Functions Without Return

### Previous Example (No Return)

```go
package main

import "fmt"

func add(number1 int, number2 int) {
    sum := number1 + number2
    fmt.Println(sum)  // Just prints, doesn't return
}

func main() {
    add(10, 20)  // Prints: 30
}
```

**Problem:** The function prints the result, but we can't use that value for anything else!

```go
func main() {
    add(10, 20)  // Prints 30
    
    // But how do we use this result?
    // result := add(10, 20)  // ❌ Can't capture the value!
}
```

---

## Functions with Return Values

### The Solution: Return Statement

Instead of just printing, we can **return** the value so it can be used elsewhere!

```go
package main

import "fmt"

func add(number1 int, number2 int) int {  // ← Returns int
    sum := number1 + number2
    return sum  // ← Returns the value
}

func main() {
    result := add(10, 20)  // ✅ Captures the returned value
    fmt.Println(result)    // Prints: 30
}
```

**Benefits:**
- ✅ Can capture and reuse the result
- ✅ Can use in calculations
- ✅ Can pass to other functions
- ✅ More flexible!

---

## Return Type Declaration

### Function Anatomy with Return Type

```go
func functionName(param1 type1, param2 type2) returnType {
    // Function body
    return value
}
```

### Breaking It Down

```go
func add(number1 int, number2 int) int {
//  ↑   ↑                          ↑
//  │   │                          └─ Return type
//  │   └─ Input section (parameters)
//  └─ Function name
    
    sum := number1 + number2
    return sum  // Must return an int
}
```

**Sections of a Function:**

| Section | Location | Purpose | Example |
|---------|----------|---------|---------|
| **Keyword** | Start | Declares function | `func` |
| **Name** | After `func` | Function identifier | `add` |
| **Input** | `(...)` | Parameters | `(number1 int, number2 int)` |
| **Output** | After `()` | Return type | `int` |
| **Body** | `{...}` | Code to execute | `return sum` |

---

## Single Return Values

### Example: Addition Function

```go
package main

import "fmt"

func add(number1 int, number2 int) int {
    sum := number1 + number2
    return sum
}

func main() {
    a := 10
    b := 20
    result := add(a, b)
    fmt.Println(result)  // 30
}
```

**Flow:**
1. Call `add(10, 20)`
2. Inside function: `sum = 10 + 20 = 30`
3. `return sum` sends 30 back
4. `result` receives 30
5. Print: 30

---

### How Return Replaces Function Call

When a function returns, the computer **mentally replaces** the function call with the returned value.

```go
result := add(10, 20)
```

**What computer does:**

```
Step 1: Call add(10, 20)
        ↓
Step 2: Execute function
        sum = 10 + 20 = 30
        ↓
Step 3: return 30
        ↓
Step 4: Replace function call with 30
        result := 30
        ↓
Step 5: result now equals 30
```

**Visual:**

```go
// Before execution
result := add(10, 20)

// Computer mentally transforms to
result := 30  // (after function executes and returns)
```

---

### Different Return Types

#### Returning Integer

```go
func getAge() int {
    return 25
}

func main() {
    age := getAge()
    fmt.Println(age)  // 25
}
```

---

#### Returning String

```go
func getName() string {
    return "Habib"
}

func main() {
    name := getName()
    fmt.Println(name)  // Habib
}
```

---

#### Returning Boolean

```go
func isAdult(age int) bool {
    if age >= 18 {
        return true
    }
    return false
}

func main() {
    result := isAdult(20)
    fmt.Println(result)  // true
}
```

---

#### Returning Float

```go
func calculatePrice(basePrice float32, tax float32) float32 {
    total := basePrice + tax
    return total
}

func main() {
    price := calculatePrice(100.0, 15.0)
    fmt.Println(price)  // 115.0
}
```

---

## Multiple Return Values

One of Go's **special features** is the ability to return **multiple values** from a single function!

### Syntax for Multiple Returns

```go
func functionName(params) (returnType1, returnType2) {
    // Function body
    return value1, value2
}
```

**Important:** When returning multiple values, use **parentheses** `()` around the return types!

---

### Example: Sum and Product

```go
package main

import "fmt"

func getNumbers(number1 int, number2 int) (int, int) {
    sum := number1 + number2
    product := number1 * number2
    return sum, product
}

func main() {
    a := 10
    b := 20
    
    p, q := getNumbers(a, b)
    
    fmt.Println("Sum:", p)       // Sum: 30
    fmt.Println("Product:", q)   // Product: 200
}
```

**Breakdown:**

```go
func getNumbers(number1 int, number2 int) (int, int) {
//                                        ↑        ↑
//                                    First   Second
//                                    return  return
//                                    (int)   (int)

    sum := number1 + number2        // 10 + 20 = 30
    product := number1 * number2    // 10 * 20 = 200
    
    return sum, product  // Returns TWO values
}
```

---

### Why Multiple Returns Are Useful

#### Before (Without Multiple Returns):

```go
func calculateSumOnly(a int, b int) int {
    return a + b
}

func calculateProductOnly(a int, b int) int {
    return a * b
}

func main() {
    sum := calculateSumOnly(10, 20)
    product := calculateProductOnly(10, 20)
    // Need two function calls!
}
```

---

#### After (With Multiple Returns):

```go
func calculate(a int, b int) (int, int) {
    return a + b, a * b
}

func main() {
    sum, product := calculate(10, 20)
    // One function call gets both!
}
```

**Benefits:**
- ✅ More efficient (one call instead of two)
- ✅ Related values grouped together
- ✅ Cleaner code

---

### Multiple Returns with Different Types

You can return different types together!

```go
package main

import "fmt"

func getUserInfo(id int) (string, int, bool) {
    name := "Habib"
    age := 25
    isActive := true
    
    return name, age, isActive
}

func main() {
    name, age, active := getUserInfo(1)
    
    fmt.Println("Name:", name)       // Name: Habib
    fmt.Println("Age:", age)         // Age: 25
    fmt.Println("Active:", active)   // Active: true
}
```

**Return types:**
- `string` for name
- `int` for age
- `bool` for active status

---

## How Return Works in Memory

### Single Return Value

```go
package main

import "fmt"

func add(number1 int, number2 int) int {
    sum := number1 + number2
    return sum
}

func main() {
    a := 10
    b := 20
    result := add(a, b)
    fmt.Println(result)
}
```

**Memory Visualization:**

```
Step 1: main() starts
┌─────────────────┐
│ Main Function   │
├──────┬──────┐   │
│ a=10 │ b=20 │   │
└──────┴──────┴───┘

Step 2: add(10, 20) called
┌─────────────────┬────────────────────┐
│ Main Function   │ add() Function     │
├──────┬──────┐   ├─────────┬──────────┤
│ a=10 │ b=20 │   │number1=10│number2=20│
└──────┴──────┘   │ sum=30   │         │
                  └──────────┴──────────┘

Step 3: return sum (returns 30)
        Computer thinks: add(10, 20) → 30
        
Step 4: result := 30
┌─────────────────────┐
│ Main Function       │
├──────┬──────┬───────┤
│ a=10 │ b=20 │result │
│      │      │  =30  │
└──────┴──────┴───────┘

Step 5: add() memory freed
        (Function completed, no longer needed)
```

---

### Multiple Return Values

```go
package main

import "fmt"

func getNumbers(number1 int, number2 int) (int, int) {
    sum := number1 + number2
    product := number1 * number2
    return sum, product
}

func main() {
    a := 10
    b := 20
    p, q := getNumbers(a, b)
    fmt.Println(p, q)
}
```

**Memory Visualization:**

```
Step 1: main() starts
┌─────────────────┐
│ Main Function   │
├──────┬──────┐   │
│ a=10 │ b=20 │   │
└──────┴──────┴───┘

Step 2: getNumbers(10, 20) called
┌─────────────────┬────────────────────────────┐
│ Main Function   │ getNumbers() Function      │
├──────┬──────┐   ├─────────┬──────────┬───────┤
│ a=10 │ b=20 │   │number1=10│number2=20│       │
└──────┴──────┘   │ sum=30   │product=200│      │
                  └──────────┴───────────┴──────┘

Step 3: return sum, product (returns 30, 200)
        Computer thinks: getNumbers(10, 20) → (30, 200)
        
Step 4: p, q := 30, 200
┌──────────────────────────┐
│ Main Function            │
├──────┬──────┬─────┬──────┤
│ a=10 │ b=20 │ p=30│q=200 │
└──────┴──────┴─────┴──────┘

Step 5: getNumbers() memory freed
```

**Key Point:** The function returns TWO values at once, which are captured by TWO variables (`p` and `q`)!

---

## Capturing Return Values

### Capturing Single Return

```go
// Method 1: Declare and assign
result := add(10, 20)

// Method 2: Declare first, then assign
var result int
result = add(10, 20)
```

---

### Capturing Multiple Returns

```go
// Method 1: Declare and assign (short form)
sum, product := getNumbers(10, 20)

// Method 2: Declare first
var sum, product int
sum, product = getNumbers(10, 20)
```

---

### Ignoring Return Values

#### Ignoring with Blank Identifier `_`

If you don't need all returned values, use `_` (underscore) to ignore:

```go
func calculate(a int, b int) (int, int) {
    return a + b, a * b
}

func main() {
    // Only want sum, ignore product
    sum, _ := calculate(10, 20)
    fmt.Println(sum)  // 30
    
    // Only want product, ignore sum
    _, product := calculate(10, 20)
    fmt.Println(product)  // 200
}
```

**Use cases:**
- When you only need some of the return values
- When a function returns error status you don't care about (not recommended!)

---

## Practical Examples

### Example 1: Calculate Rectangle Properties

```go
package main

import "fmt"

func rectangleCalc(length int, width int) (int, int) {
    area := length * width
    perimeter := 2 * (length + width)
    return area, perimeter
}

func main() {
    area, perimeter := rectangleCalc(10, 5)
    
    fmt.Println("Area:", area)           // Area: 50
    fmt.Println("Perimeter:", perimeter) // Perimeter: 30
}
```

---

### Example 2: Division with Remainder

```go
package main

import "fmt"

func divideWithRemainder(dividend int, divisor int) (int, int) {
    quotient := dividend / divisor
    remainder := dividend % divisor
    return quotient, remainder
}

func main() {
    q, r := divideWithRemainder(17, 5)
    
    fmt.Println("Quotient:", q)   // Quotient: 3
    fmt.Println("Remainder:", r)  // Remainder: 2
}
```

---

### Example 3: Temperature Conversion

```go
package main

import "fmt"

func convertTemperature(celsius float32) (float32, float32) {
    fahrenheit := (celsius * 9/5) + 32
    kelvin := celsius + 273.15
    return fahrenheit, kelvin
}

func main() {
    f, k := convertTemperature(25)
    
    fmt.Printf("25°C = %.2f°F = %.2fK\n", f, k)
    // Output: 25°C = 77.00°F = 298.15K
}
```

---

### Example 4: Min and Max

```go
package main

import "fmt"

func minMax(a int, b int) (int, int) {
    if a < b {
        return a, b  // a is min, b is max
    }
    return b, a  // b is min, a is max
}

func main() {
    min, max := minMax(5, 15)
    
    fmt.Println("Min:", min)  // Min: 5
    fmt.Println("Max:", max)  // Max: 15
}
```

---

### Example 5: String Operations

```go
package main

import "fmt"

func stringInfo(text string) (int, string) {
    length := len(text)
    uppercase := "CONVERTED: " + text
    return length, uppercase
}

func main() {
    length, upper := stringInfo("hello")
    
    fmt.Println("Length:", length)  // Length: 5
    fmt.Println(upper)              // CONVERTED: hello
}
```

---

### Example 6: Circle Calculations

```go
package main

import "fmt"

func circleProperties(radius float32) (float32, float32) {
    const pi = 3.14159
    area := pi * radius * radius
    circumference := 2 * pi * radius
    return area, circumference
}

func main() {
    area, circ := circleProperties(5)
    
    fmt.Printf("Area: %.2f\n", area)           // Area: 78.54
    fmt.Printf("Circumference: %.2f\n", circ)  // Circumference: 31.42
}
```

---

## Common Mistakes

### ❌ Mistake 1: Forgetting Return Type

```go
// ❌ WRONG - No return type specified
func add(a int, b int) {
    return a + b  // Error! No return type declared
}

// ✅ CORRECT
func add(a int, b int) int {
    return a + b
}
```

---

### ❌ Mistake 2: Wrong Return Type

```go
// ❌ WRONG - Declares int but returns string
func getName() int {
    return "Habib"  // Error! String, not int
}

// ✅ CORRECT
func getName() string {
    return "Habib"
}
```

---

### ❌ Mistake 3: Not Returning a Value

```go
// ❌ WRONG - Says it returns int but doesn't return
func getNumber() int {
    x := 10
    // Forgot to return!
}

// ✅ CORRECT
func getNumber() int {
    x := 10
    return x
}
```

---

### ❌ Mistake 4: Wrong Number of Return Values

```go
func calculate(a int, b int) (int, int) {
    sum := a + b
    return sum  // ❌ WRONG! Should return 2 values, only returning 1
}

// ✅ CORRECT
func calculate(a int, b int) (int, int) {
    sum := a + b
    product := a * b
    return sum, product  // Returns 2 values
}
```

---

### ❌ Mistake 5: Wrong Number of Variables to Capture

```go
func calculate(a int, b int) (int, int) {
    return a + b, a * b
}

func main() {
    // ❌ WRONG - Function returns 2 values, only capturing 1
    result := calculate(10, 20)
    
    // ✅ CORRECT
    sum, product := calculate(10, 20)
}
```

---

### ❌ Mistake 6: Missing Parentheses for Multiple Returns

```go
// ❌ WRONG - Missing parentheses
func calculate(a int, b int) int, int {
    return a + b, a * b
}

// ✅ CORRECT - Parentheses required for multiple returns
func calculate(a int, b int) (int, int) {
    return a + b, a * b
}
```

---

### ❌ Mistake 7: Not Using Returned Value

```go
func add(a int, b int) int {
    return a + b
}

func main() {
    add(10, 20)  // ⚠️ WARNING - Result not used!
    
    // ✅ BETTER - Capture and use it
    result := add(10, 20)
    fmt.Println(result)
}
```

---

## Return Value Best Practices

### 1. Return Early for Error Conditions

```go
func divide(a int, b int) int {
    if b == 0 {
        fmt.Println("Error: Division by zero")
        return 0  // Return early
    }
    return a / b
}
```

---

### 2. Be Consistent with Return Order

```go
// ✅ GOOD - Consistent order (min first, max second)
func minMax(a int, b int) (int, int) {
    if a < b {
        return a, b  // min, max
    }
    return b, a  // min, max
}
```

---

### 3. Use Descriptive Variable Names

```go
// ❌ OK but not clear
func calc(a int, b int) (int, int) {
    return a + b, a * b
}

// ✅ BETTER
func calculate(a int, b int) (int, int) {
    sum := a + b
    product := a * b
    return sum, product
}
```

---

### 4. Consider Named Return Values

```go
// Advanced: Named return values
func calculate(a int, b int) (sum int, product int) {
    sum = a + b
    product = a * b
    return  // No need to specify what to return
}
```

---

## Comparison: With vs Without Return

### Without Return (Limited)

```go
func add(a int, b int) {
    sum := a + b
    fmt.Println(sum)
}

func main() {
    add(10, 20)  // Just prints, can't reuse
    
    // Can't do this:
    // result := add(10, 20)  // ❌ Error
    // total := result + 100   // ❌ Can't calculate with it
}
```

---

### With Return (Flexible)

```go
func add(a int, b int) int {
    return a + b
}

func main() {
    result := add(10, 20)  // ✅ Capture result
    
    // Can reuse it:
    total := result + 100  // ✅ Use in calculation
    fmt.Println(total)     // ✅ Print when needed
    
    // Can pass to other functions:
    fmt.Println(add(5, 7)) // ✅ Direct use
}
```

---

## Visual Summary

### Function Signature Structure

```go
func name(input params) output types {
//   ↑    ↑              ↑
//   │    │              └─ What function returns
//   │    └─ What function receives
//   └─ Function identifier

    // Function body
    return value(s)
}
```

---

### Single vs Multiple Returns

```go
// Single Return
func add(a int, b int) int {
    return a + b
}
// Returns: one int

// Multiple Returns
func calculate(a int, b int) (int, int) {
    return a + b, a * b
}
// Returns: two ints
```

---

## Summary

### 🎯 Key Takeaways

1. **Return Values Make Functions Flexible:**
   - Can capture and reuse results
   - Can use in calculations
   - Can pass to other functions

2. **Return Type Declaration:**
   ```go
   func name(params) returnType { }     // Single return
   func name(params) (type1, type2) { } // Multiple returns
   ```

3. **Single Return:**
   - Specify one return type
   - Return one value
   - Capture in one variable

4. **Multiple Returns (Go's Special Feature!):**
   - Use parentheses for return types
   - Return multiple values separated by commas
   - Capture in multiple variables

5. **How Return Works:**
   - Function executes
   - Computes result(s)
   - Returns value(s) to caller
   - Computer "replaces" function call with returned value(s)
   - Function memory is freed

6. **The main() Function:**
   - `main()` is also a function!
   - Takes no parameters: `()`
   - Returns nothing: (no return type)
   - Entry point of program

7. **Best Practices:**
   - Always match return type with actual return
   - Return correct number of values
   - Use `_` to ignore unwanted returns
   - Capture return values when needed
   - Use descriptive names

---

### 📝 Complete Example: All Concepts

```go
package main

import "fmt"

// Single return value
func add(a int, b int) int {
    return a + b
}

// Multiple return values
func calculate(a int, b int) (int, int, int) {
    sum := a + b
    diff := a - b
    product := a * b
    return sum, diff, product
}

// Different return types
func getUserData() (string, int, bool) {
    name := "Habib"
    age := 25
    isActive := true
    return name, age, isActive
}

func main() {
    // Using single return
    result := add(10, 20)
    fmt.Println("Sum:", result)  // Sum: 30
    
    // Using multiple returns
    s, d, p := calculate(10, 5)
    fmt.Println("Sum:", s)       // Sum: 15
    fmt.Println("Diff:", d)      // Diff: 5
    fmt.Println("Product:", p)   // Product: 50
    
    // Using returns with different types
    name, age, active := getUserData()
    fmt.Println("Name:", name)       // Name: Habib
    fmt.Println("Age:", age)         // Age: 25
    fmt.Println("Active:", active)   // Active: true
    
    // Ignoring some returns
    sum, _, _ := calculate(20, 10)
    fmt.Println("Only sum:", sum)  // Only sum: 30
}
```

**Output:**
```
Sum: 30
Sum: 15
Diff: 5
Product: 50
Name: Habib
Age: 25
Active: true
Only sum: 30
```

---

## Practice Exercises

### Exercise 1: Square and Cube
Create a function that returns both the square and cube of a number.

```go
func squareAndCube(n int) (int, int) {
    // Your code here
}

func main() {
    sq, cu := squareAndCube(5)
    // Expected: sq = 25, cu = 125
}
```

---

### Exercise 2: String Length and First Char
Create a function that returns the length of a string and its first character.

```go
func stringInfo(text string) (int, string) {
    // Your code here
    // Hint: Use len(text) and text[0:1]
}
```

---

### Exercise 3: Temperature Both Ways
Create a function that converts Celsius to both Fahrenheit and Kelvin.

```go
func convertCelsius(celsius float32) (float32, float32) {
    // Fahrenheit = (C * 9/5) + 32
    // Kelvin = C + 273.15
}
```

---

### Exercise 4: Even or Odd Check
Create a function that checks if a number is even and returns the number and a boolean.

```go
func checkEven(n int) (int, bool) {
    // Return the number and true if even, false if odd
}
```

---

### Exercise 5: Calculate All
Create a function that returns sum, difference, product, and quotient of two numbers.

```go
func calculateAll(a int, b int) (int, int, int, int) {
    // Return sum, diff, product, quotient
}
```

---

## What's Next?

In the next chapters, you'll learn:
- **Error Handling** - Functions returning error values
- **Named Return Values** - Go's convenient feature
- **Variadic Functions** - Functions with variable parameters
- **Defer, Panic, Recover** - Advanced error handling
- **Closures** - Functions inside functions

**You've learned a crucial skill!** Return values make functions incredibly powerful and flexible. Practice writing functions with returns - you'll use this constantly!

Keep coding! 🚀✨

---

*Based on Go Programming Tutorial - Chapter 5*  
*Topics: Return Values, Single Returns, Multiple Returns, Function Signatures*
