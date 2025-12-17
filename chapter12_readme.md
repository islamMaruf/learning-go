# Chapter 12: Variable Shadowing - The Shadow Effect

## 📚 Table of Contents
1. [Introduction](#introduction)
2. [What Is Variable Shadowing?](#what-is-variable-shadowing)
3. [The Classic Shadowing Example](#the-classic-shadowing-example)
4. [Memory Simulation](#memory-simulation)
5. [Scope Lookup Priority](#scope-lookup-priority)
6. [Real-Life Analogies](#real-life-analogies)
7. [Interview Questions](#interview-questions)
8. [Common Shadowing Patterns](#common-shadowing-patterns)
9. [When Shadowing Is Good](#when-shadowing-is-good)
10. [When Shadowing Is Dangerous](#when-shadowing-is-dangerous)
11. [Practice Exercises](#practice-exercises)
12. [Summary](#summary)
13. [What's Next?](#whats-next)

---

## Introduction

**Today's topic: Variable Shadowing!**

Some people call it **"shadowing"**, some call it **"variable shadowing"** - it doesn't matter. What matters is understanding this concept!

### Why This Matters

> **"This is a COMMON INTERVIEW QUESTION! If you make a mistake here, the interviewer will think you don't know anything. They'll think you're not good enough. Your job interview could fail on this one question!"**

### What You'll Learn

1. ✅ What is variable shadowing
2. ✅ How shadowing works in memory
3. ✅ Why the same variable name behaves differently
4. ✅ Scope lookup priority
5. ✅ How to avoid confusion
6. ✅ How to ace interview questions

---

## What Is Variable Shadowing?

### Simple Definition

> **Variable shadowing occurs when a variable declared in an inner scope has the SAME NAME as a variable in an outer scope. The inner variable "shadows" (hides) the outer one within that scope.**

### Visual Concept

```
        Outer Scope
            ↓
    var x = 10  ← Original variable
            ↓
        Inner Scope
            ↓
    var x = 20  ← Shadow variable
            ↓
    (x = 10 is hidden here!)
```

**Like a shadow from the sun:**
- ☀️ **Sun (Parent)** = Outer variable
- 🚶 **Person (Child)** = Inner variable
- 👤 **Shadow on Ground** = The "shadowing" effect

---

## The Classic Shadowing Example

### The Interview Question Code

```go
package main

import "fmt"

var a = 10  // Global scope

func main() {
    age := 30  // Main scope
    
    if age > 18 {
        var a int = 47  // Block scope - SHADOWS global 'a'
        fmt.Println(a)   // What prints here?
    }
    
    fmt.Println(a)  // What prints here?
}
```

### The Interview Question

**Q:** "What is the output of this program?"

**Think before you answer!** ⏸️

<details>
<summary>Click for Answer</summary>

**Output:**
```
47
10
```

**Explanation:**
1. First `Println(a)` is inside the if block → prints **47** (block scope variable)
2. Second `Println(a)` is in main scope → prints **10** (global scope variable)
3. The block's `a` shadows the global `a` only within the block!

</details>

---

## Memory Simulation

Let's trace this code step-by-step in memory!

### Code to Simulate

```go
package main

import "fmt"

var a = 10

func main() {
    age := 30
    
    if age > 18 {
        var a int = 47
        fmt.Println(a)   // Line 12
    }
    
    fmt.Println(a)  // Line 15
}
```

---

### Phase 1: Global Scope Created

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  a = 10                                 │
│  main = [function code]                 │
└─────────────────────────────────────────┘
```

**What happened:**
- ✅ Read `var a = 10` → Store in global scope
- ✅ Read `func main()` → Store function definition

---

### Phase 2: main() Executes

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  a = 10                                 │
│  main = [function code]                 │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         MAIN SCOPE (Created)            │
├─────────────────────────────────────────┤
│  age = 30                               │
└─────────────────────────────────────────┘
```

**What happened:**
- ✅ Created scope for main()
- ✅ Declared `age = 30`

---

### Phase 3: Check if Condition

```go
if age > 18  // Check: Is 30 > 18? YES!
```

**Computer checks:**
1. ✅ Find `age` in main scope → Found! (age = 30)
2. ✅ Check: 30 > 18? → TRUE!
3. ✅ Enter if block

---

### Phase 4: If Block Creates Scope

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  a = 10  ← Original 'a'                 │
│  main = [function code]                 │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         MAIN SCOPE                      │
├─────────────────────────────────────────┤
│  age = 30                               │
│                                         │
│  ┌───────────────────────────────────┐ │
│  │   IF BLOCK SCOPE                  │ │
│  ├───────────────────────────────────┤ │
│  │  a = 47  ← Shadow 'a' !!! 👻     │ │
│  └───────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

**What happened:**
- ✅ Created new scope for if block
- ✅ Declared NEW variable `a = 47` in block scope
- ⚠️ **SHADOWING OCCURS!** Block's `a` hides global `a`

**Key Point:** We now have **TWO variables named `a`**:
1. Global `a = 10`
2. Block `a = 47`

---

### Phase 5: Print Inside Block

```go
fmt.Println(a)  // Line 12
```

**Computer's search for `a`:**

```
Step 1: Check current block (IF BLOCK SCOPE)
        ↓
    Found 'a' = 47 ✅
        ↓
    Use this value!
        ↓
    Print: 47
```

**Why not print 10?**
> "I'm in the block! I check my own block FIRST before checking my parent (main) or grandparent (global)!"

**Output so far:**
```
47
```

---

### Phase 6: If Block Ends - Shadow Disappears

```go
}  // ← If block ends
```

**Memory after block ends:**

```
┌─────────────────────────────────────────┐
│         GLOBAL SCOPE                    │
├─────────────────────────────────────────┤
│  a = 10  ← Still here!                  │
│  main = [function code]                 │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         MAIN SCOPE                      │
├─────────────────────────────────────────┤
│  age = 30                               │
│  (if block destroyed!)                  │
└─────────────────────────────────────────┘

[IF BLOCK SCOPE DESTROYED! 💀]
[Block's 'a = 47' is GONE! 💀]
```

**Key Point:** The shadow variable `a = 47` is destroyed!

---

### Phase 7: Print Outside Block

```go
fmt.Println(a)  // Line 15
```

**Computer's search for `a`:**

```
Step 1: Check main scope
        ↓
    'a' not found ❌
        ↓
Step 2: Check global scope
        ↓
    Found 'a' = 10 ✅
        ↓
    Use this value!
        ↓
    Print: 10
```

**Output so far:**
```
47
10
```

**Final output:**
```
47
10
```

---

## Scope Lookup Priority

### The Search Order

When you use a variable, Go searches in this order:

```
1. CURRENT BLOCK SCOPE (Most Inner)
   ↓ Not found?
2. PARENT BLOCK SCOPE
   ↓ Not found?
3. FUNCTION SCOPE
   ↓ Not found?
4. GLOBAL SCOPE
   ↓ Not found?
5. ERROR: undefined ❌
```

### Priority Visualization

```
Priority:  1st → 2nd → 3rd → 4th
            ↓     ↓     ↓     ↓
         Block  Main  Outer Global
```

**Think of it like asking for money:**

```
Need Money?
    ↓
1. Ask Parents (Closest) 👨👩
    ↓ Don't have?
2. Ask Uncle (Relatives) 👨‍👦
    ↓ Don't have?
3. Ask Friends 👥
    ↓ Don't have?
4. Beg on Street 🙏
```

**Same with variables:**
- ✅ Check closest scope first
- ✅ Move outward if not found
- ✅ Use first match found
- ❌ Error if nowhere found

---

## Real-Life Analogies

### Analogy 1: Father's Birthmark

> "If your father has a birthmark on his forehead, and you also have a birthmark on your forehead, people say: 'Oh! He got his father's shadow!' or 'He inherited his father's mark!'"

**In Code:**
```go
var x = 10  // Father's variable

func main() {
    var x = 20  // Child's variable (shadow)
    // Child's x "inherits" the NAME, but has different value!
}
```

**The "shadow" is the inheritance of the NAME, not the value!**

---

### Analogy 2: Asking for Money

> "You need money. You ask your parents first. If they don't have it, you ask your uncles. If they don't have it, you ask friends. Same with variable lookup!"

**In Code:**
```go
var money = 100  // Global (Friends)

func family() {
    var money = 500  // Function scope (Uncles)
    
    if true {
        var money = 1000  // Block scope (Parents) ← Check here FIRST!
        fmt.Println(money)  // Uses 1000
    }
}
```

---

## Interview Questions

### Question 1: Simple Shadowing

```go
package main

import "fmt"

var x = 5

func main() {
    x := 10
    fmt.Println(x)
}
```

**Q:** What is the output?

<details>
<summary>Answer</summary>

**Output:** `10`

**Explanation:** The local `x` in main() shadows the global `x`.
</details>

---

### Question 2: Block Shadowing

```go
package main

import "fmt"

func main() {
    x := 10
    
    if true {
        x := 20
        fmt.Println(x)  // Line A
    }
    
    fmt.Println(x)  // Line B
}
```

**Q:** What prints at Line A and Line B?

<details>
<summary>Answer</summary>

**Output:**
```
20
10
```

**Explanation:**
- Line A: Block's `x = 20` (shadow)
- Line B: Function's `x = 10` (original)
</details>

---

### Question 3: Multiple Levels

```go
package main

import "fmt"

var x = 1

func main() {
    x := 2
    
    if true {
        x := 3
        
        if true {
            x := 4
            fmt.Println(x)  // Line A
        }
        
        fmt.Println(x)  // Line B
    }
    
    fmt.Println(x)  // Line C
}

func other() {
    fmt.Println(x)  // Line D
}
```

**Q:** What prints at each line?

<details>
<summary>Answer</summary>

**Output:**
```
4
3
2
```

**Explanation:**
- Line A: Innermost block's x = 4
- Line B: Middle block's x = 3
- Line C: Function's x = 2
- Line D: (Not called in main, but would print 1 - global x)
</details>

---

### Question 4: No Shadowing

```go
package main

import "fmt"

var x = 10

func main() {
    if true {
        x = 20  // ← NOTE: No 'var' or ':='
        fmt.Println(x)
    }
    
    fmt.Println(x)
}
```

**Q:** What is the output?

<details>
<summary>Answer</summary>

**Output:**
```
20
20
```

**Explanation:** 
- No new variable declared! Just modifying global `x`.
- `x = 20` changes the global variable.
- No shadowing occurred.
</details>

---

### Question 5: Tricky Mix

```go
package main

import "fmt"

var a = 100

func main() {
    a := 200
    b := 300
    
    if true {
        a := 400
        fmt.Println(a, b)  // Line A
    }
    
    fmt.Println(a, b)  // Line B
}
```

**Q:** What prints at each line?

<details>
<summary>Answer</summary>

**Output:**
```
400 300
200 300
```

**Explanation:**
- Line A: Block's `a = 400` (shadow), main's `b = 300`
- Line B: Main's `a = 200`, main's `b = 300`
- Global `a = 100` is never used (shadowed by main's a)
</details>

---

## Common Shadowing Patterns

### Pattern 1: Function Parameter Shadowing Global

```go
var name = "Global"

func greet(name string) {  // Parameter shadows global
    fmt.Println("Hello,", name)
}

func main() {
    greet("Alice")  // Prints: Hello, Alice
    fmt.Println(name)  // Prints: Global
}
```

**Safe and common pattern!** ✅

---

### Pattern 2: Loop Variable Shadowing

```go
var i = 100

func main() {
    for i := 0; i < 3; i++ {  // Loop 'i' shadows global 'i'
        fmt.Println(i)  // Prints: 0, 1, 2
    }
    
    fmt.Println(i)  // Prints: 100
}
```

**Safe pattern!** ✅

---

### Pattern 3: If Block Shadowing

```go
func main() {
    x := 10
    
    if x > 5 {
        x := 20  // Shadow
        fmt.Println(x)  // 20
    }
    
    fmt.Println(x)  // 10
}
```

**Can be confusing!** ⚠️

---

### Pattern 4: Error Handling Shadowing

```go
func main() {
    var err error
    
    if true {
        data, err := fetchData()  // ← ':=' creates NEW err!
        // ...
    }
    
    // Original err is still nil!
}
```

**Dangerous pattern!** ⚠️ Common bug source!

---

## When Shadowing Is Good

### Good Use Case 1: Limited Scope

```go
func processData() {
    data := fetchFromAPI()  // Might fail
    
    if data == nil {
        data := getDefaultData()  // Different 'data' for error case
        useData(data)
        return
    }
    
    useData(data)  // Original 'data'
}
```

**Benefit:** Clear separation of concerns.

---

### Good Use Case 2: Type Conversion

```go
func process() {
    value := "123"  // string
    
    if needInt {
        value, _ := strconv.Atoi(value)  // int (shadows string)
        calculateWithInt(value)
    } else {
        processString(value)  // Original string
    }
}
```

**Benefit:** Same logical name, different types in different contexts.

---

## When Shadowing Is Dangerous

### Danger 1: Accidental Shadowing

```go
var globalConfig = loadConfig()

func updateConfig() {
    if needsUpdate {
        config := getNewConfig()  // ← Oops! Created local, not updating global
        // Think we updated globalConfig, but didn't!
    }
}
```

**Problem:** Meant to update global, created shadow instead!

**Fix:**
```go
func updateConfig() {
    if needsUpdate {
        globalConfig = getNewConfig()  // ✅ Assignment, not declaration
    }
}
```

---

### Danger 2: Error Variable Shadowing

```go
func readFile() error {
    var err error
    
    file, err := os.Open("file.txt")
    defer file.Close()
    
    if needsValidation {
        data, err := readData(file)  // ← ':=' creates NEW err!
        // Error here doesn't affect outer err
    }
    
    return err  // Returns outer err (might be nil!)
}
```

**Problem:** Inner `err` shadows outer, losing error information!

**Fix:**
```go
func readFile() error {
    var err error
    
    file, err := os.Open("file.txt")
    defer file.Close()
    
    if needsValidation {
        var data []byte
        data, err = readData(file)  // ✅ Assignment to outer err
        // ...
    }
    
    return err  // Returns correct err
}
```

---

## Practice Exercises

### Exercise 1: Predict the Output

```go
package main

import "fmt"

var a = 1
var b = 2

func main() {
    a := 10
    
    if a > 5 {
        b := 20
        a := 30
        fmt.Println(a, b)  // ?
    }
    
    fmt.Println(a, b)  // ?
}
```

---

### Exercise 2: Fix the Bug

```go
package main

import "fmt"

var counter = 0

func increment() {
    if counter < 10 {
        counter := counter + 1  // Bug here!
        fmt.Println("Incremented to:", counter)
    }
}

func main() {
    for i := 0; i < 5; i++ {
        increment()
    }
    fmt.Println("Final counter:", counter)  // Always 0!
}
```

**Task:** Fix the shadowing bug.

---

### Exercise 3: Trace the Shadows

Draw a memory diagram showing all scopes and which `x` is accessible where:

```go
var x = 1

func outer() {
    x := 2
    
    func inner() {
        x := 3
        fmt.Println(x)
    }
    
    inner()
    fmt.Println(x)
}

func main() {
    outer()
    fmt.Println(x)
}
```

---

### Exercise 4: Error Handling

Fix this buggy error handling code:

```go
func processFiles() error {
    var err error
    
    for _, filename := range files {
        data, err := readFile(filename)  // Bug!
        if err != nil {
            continue
        }
        process(data)
    }
    
    return err  // Always returns last error or nil
}
```

---

### Exercise 5: Scope Levels

Identify all shadow variables:

```go
var global = 0

func level1() {
    global := 1
    
    if true {
        global := 2
        
        if true {
            global := 3
            fmt.Println(global)
        }
        
        fmt.Println(global)
    }
    
    fmt.Println(global)
}
```

How many different `global` variables exist?

---

### Exercise 6: Real-World Scenario

Review this code and identify potential shadowing issues:

```go
type Config struct {
    timeout int
}

var config = Config{timeout: 30}

func updateTimeout(newTimeout int) {
    if newTimeout > 0 {
        config := Config{timeout: newTimeout}
        // ... use config ...
    }
    // Is global config updated?
}

func getTimeout() int {
    return config.timeout
}
```

---

## Summary

### Key Takeaways

1. ✅ **Shadowing = Same name, different scopes**
2. ✅ **Inner shadows outer**
3. ✅ **Shadow disappears when scope ends**
4. ✅ **Search order: Inner → Outer → Global**
5. ✅ **Common interview question**
6. ⚠️ **Can cause bugs if not careful**
7. ⚠️ **Especially dangerous with error variables**

### Shadowing Rules

| Rule | Description |
|------|-------------|
| **Same Name Required** | Both variables must have identical names |
| **Different Scopes** | Must be in nested scopes |
| **Inner Wins** | Inner scope variable is used |
| **Temporary** | Shadow only exists within inner scope |
| **No Effect on Outer** | Changing shadow doesn't affect outer variable |

### Visual Summary

```
┌────────────────────────────────────────┐
│  GLOBAL SCOPE                          │
│  var x = 10  ← Original                │
│                                        │
│  ┌──────────────────────────────────┐ │
│  │  FUNCTION SCOPE                  │ │
│  │  var x = 20  ← Shadow (hides 10) │ │
│  │                                  │ │
│  │  ┌────────────────────────────┐ │ │
│  │  │  BLOCK SCOPE               │ │ │
│  │  │  var x = 30  ← Shadow      │ │ │
│  │  │              (hides 20)    │ │ │
│  │  └────────────────────────────┘ │ │
│  │                                  │ │
│  │  (x = 20 here)                   │ │
│  └──────────────────────────────────┘ │
│                                        │
│  (x = 10 here)                         │
└────────────────────────────────────────┘
```

### The Shadow Analogy

```
☀️ Sun (Global variable)
    ↓ casts light on
🚶 Person (Inner scope)
    ↓ creates
👤 Shadow (Same name variable)
    ↓ temporarily hides
☀️ Sun's light (Outer variable)
```

**When person moves away (scope ends), sun shines again!**

---

## What's Next?

Now that you understand shadowing, you're ready for advanced variable concepts!

### Chapter 13 Preview: Pointers
- **What are pointers?** - Variables that store addresses
- **The `&` operator** - Getting the address
- **The `*` operator** - Getting the value
- **Pointer vs Value** - When to use which
- **Nil pointers** - The zero value
- **Pointer receivers** - Methods on pointers

### Why Shadowing Knowledge Was Important

Without understanding shadowing:
- ❌ Can't debug mysterious bugs
- ❌ Fail interview questions
- ❌ Write buggy error handling
- ❌ Confused by scope rules

**With shadowing knowledge:**
- ✅ Write cleaner code
- ✅ Avoid common bugs
- ✅ Ace interviews
- ✅ Understand scope deeply

---

### Final Wisdom

**The Interview Reality:**
> "This is a COMMON interview question! They'll give you code with shadowing and ask: 'What's the output?' If you get it wrong, they'll think you don't know the basics!"

**The Name Inheritance:**
> "Just like inheriting your father's birthmark, the shadow variable 'inherits' the NAME from the outer variable. Same name, different values!"

**The Priority Rule:**
> "Like asking for money: Parents first (closest scope), then uncles (parent scope), then friends (global scope). Always check closest first!"

**Shadowing Is Fun:**
> "Shadowing is interesting! Once you understand it, you'll see it everywhere in code!"

---

**الله حافظ (Allah Hafez)**

---

### Quick Reference Card

```go
// SHADOWING
var x = 10        // Global

func main() {
    var x = 20    // Shadows global (main scope)
    fmt.Println(x)  // 20
    
    if true {
        var x = 30  // Shadows main's x (block scope)
        fmt.Println(x)  // 30
    }
    
    fmt.Println(x)  // 20 (main's x)
}

// NOT SHADOWING (Assignment)
var x = 10

func main() {
    x = 20  // ← Changes global x, no new variable!
    fmt.Println(x)  // 20
}
```

---

*End of Chapter 12 - Variable Shadowing Mastered!* 👻
