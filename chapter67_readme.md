# Chapter 67: Race Condition - The Concurrency Problem

## Table of Contents
1. [Introduction: Apology and Context](#introduction-apology-and-context)
2. [Recap: Where We Are](#recap-where-we-are)
3. [What is a Race Condition?](#what-is-a-race-condition)
4. [The Bank Account Analogy](#the-bank-account-analogy)
5. [Process Memory Layout](#process-memory-layout)
6. [Shared Memory and Goroutines](#shared-memory-and-goroutines)
7. [The Problem in GetProducts](#the-problem-in-getproducts)
8. [Simple Demonstration: Counter Example](#simple-demonstration-counter-example)
9. [Running the Race Condition Test](#running-the-race-condition-test)
10. [Detecting Race Conditions with -race Flag](#detecting-race-conditions-with-race-flag)
11. [Why Race Conditions Are Dangerous](#why-race-conditions-are-dangerous)
12. [Rules of Race Conditions](#rules-of-race-conditions)
13. [Summary](#summary)
14. [What's Next](#whats-next)

---

## Introduction: Apology and Context

**Hello guys, I'm extremely sorry!** 😔

I know I've been very late with these Go classes. The last class was about **WaitGroup internals**, and that was quite a while ago! 

Before that, we covered:
- **Chapter 64**: Why Goroutines Matter (why WaitGroup and channels are important)
- **Chapter 65**: WaitGroup basics
- **Chapter 66**: WaitGroup internals

And now, finally, **Chapter 67** is here!

I know this delay isn't ideal, but I promise the wait will be worth it. Today we're covering something **CRITICAL** for concurrent programming: **Race Conditions**.

---

### Quick Note: Audio Language Settings

Some of you mentioned that the video audio is switching to English automatically. If you want Bengali audio:

1. Go to **Settings** (⚙️ icon)
2. Click **Audio Track**
3. Select **Bengali**

This is YouTube's automatic feature - I don't control it. But you can switch between languages as needed!

Now, let's dive into **Race Conditions**! 🚀

---

## Recap: Where We Are

Let's quickly remember what we built in previous chapters.

### Our GetProducts Handler

We have an API endpoint that returns a paginated list of products:

```go
func (h *Handler) GetProducts(c *gin.Context) {
    // Extract page and limit from query parameters
    pageStr := c.Request.URL.Query().Get("page")
    limitStr := c.Request.URL.Query().Get("limit")
    
    // Convert to integers
    page, _ := strconv.ParseInt(pageStr, 10, 64)
    limit, _ := strconv.ParseInt(limitStr, 10, 64)
    
    // Validation
    if page <= 0 {
        page = 1
    }
    if limit <= 0 {
        limit = 10
    }
    
    // Get products list
    products, _ := h.productService.List(page, limit)
    
    // Get total count using goroutine
    var wg sync.WaitGroup
    var count int64
    
    wg.Add(1)
    go func() {
        defer wg.Done()
        count1, _ := h.productService.Count()
        count = count1  // ⚠️ PROBLEM HERE!
    }()
    
    wg.Wait()
    
    // Build response
    response := PaginatedResponse{
        Data:       products,
        Page:       page,
        Limit:      limit,
        TotalItems: count,
        TotalPages: (count + limit - 1) / limit,
    }
    
    c.JSON(http.StatusOK, response)
}
```

**The Problem:**
- We're using goroutines
- We're accessing shared variable `count`
- **This creates a RACE CONDITION!** ⚠️

But what IS a race condition? Let's understand it properly.

---

## What is a Race Condition?

A **race condition** occurs when:
1. **Multiple goroutines** (or threads) access the **same shared data**
2. **At least one** of them is **writing** (modifying) that data
3. The outcome depends on the **timing** of execution (unpredictable!)

### Simple Definition

> **Race Condition**: When multiple goroutines "race" to access and modify shared data, and the final result depends on which one finishes last.

### Visual Example

Imagine you have a variable:

```
count = 5
```

Now, 3 goroutines try to increment it at the same time:

```
Goroutine 1:         Goroutine 2:         Goroutine 3:
Read: 5              Read: 5              Read: 5
Add 1: 6             Add 1: 6             Add 1: 6
Write: 6             Write: 6             Write: 6
```

**What happened?**
- Expected: 5 + 1 + 1 + 1 = **8**
- Actual: **6** (last writer wins!)

**This is a race condition!** The final value depends on who writes last, not on the actual logic.

---

## The Bank Account Analogy

Let me explain with a **real-world example** that makes this crystal clear.

### Scenario: Your Business Account

Imagine you're a businessman. Your Bkash account has **5,000 taka**.

Your business is doing well! Products are selling everywhere. Multiple customers are paying you **at the same time**.

### Three Customers Pay Simultaneously

**Customer 1:**
1. Checks balance: 5,000 taka ✓
2. Pays 1,000 taka
3. Updates balance: 5,000 + 1,000 = 6,000

**Customer 2 (at the exact same moment):**
1. Checks balance: 5,000 taka ✓ (reads old value!)
2. Pays 1,000 taka
3. Updates balance: 5,000 + 1,000 = 6,000

**Customer 3 (also at the exact same moment):**
1. Checks balance: 5,000 taka ✓ (reads old value!)
2. Pays 1,000 taka
3. Updates balance: 5,000 + 1,000 = 6,000

### The Problem

**All three read the same initial value: 5,000**

Then all three write their result:
- Customer 1 writes: 6,000
- Customer 2 writes: 6,000 (overwrites Customer 1!)
- Customer 3 writes: 6,000 (overwrites Customer 2!)

**Final balance: 6,000 taka**
**Expected balance: 8,000 taka** (5,000 + 1,000 + 1,000 + 1,000)

**You just lost 2,000 taka!** 💸

### Would You Accept This?

**NO WAY!** 

If you're building a financial application, this is a **DISASTER**. You MUST handle concurrency properly.

This is why understanding race conditions is **CRITICAL** for any production application.

---

## Process Memory Layout

To understand race conditions deeply, we need to understand how memory works in a process.

### Four Memory Segments

Every process has **four main memory segments**:

```
┌─────────────────────────────────────┐
│                                     │
│     CODE SEGMENT                    │  ← Your program code
│     (Instructions)                  │
│                                     │
├─────────────────────────────────────┤
│                                     │
│     DATA SEGMENT                    │  ← Global variables (SHARED!)
│     (Global Data)                   │
│                                     │
├─────────────────────────────────────┤
│                                     │
│     STACK SEGMENT                   │  ← Local variables (per goroutine)
│     (Function Calls)                │
│                                     │
├─────────────────────────────────────┤
│                                     │
│     HEAP SEGMENT                    │  ← Dynamically allocated memory
│     (Dynamic Memory)                │
│                                     │
└─────────────────────────────────────┘
```

### What Goes Where?

**1. Code Segment:**
- Your compiled program instructions
- Read-only
- Shared by all goroutines

**2. Data Segment:** ⚠️ **IMPORTANT!**
- **Global variables**
- **Static variables**
- **Shared by ALL goroutines**
- This is where race conditions happen!

**3. Stack Segment:**
- Local variables
- Function parameters
- Return addresses
- Each goroutine has its **own stack** (NOT shared!)

**4. Heap Segment:**
- Dynamically allocated memory (`new`, `make`)
- Can be shared if pointers are passed around

### The Key Point

**Data Segment variables are SHARED!**

When you declare:

```go
var count int64  // Global variable
```

This `count` lives in the **Data Segment** and is **accessible by ALL goroutines**.

---

## Shared Memory and Goroutines

### Threads/Goroutines and Shared Data

Let's visualize a process with multiple goroutines:

```
Process: Your Go Application
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  Code Segment:    [Your Program Instructions]                │
│                                                               │
│  Data Segment:    [count = 5]  ← SHARED BY ALL!             │
│                   [Global vars]                              │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Goroutine 1 │  │ Goroutine 2 │  │ Goroutine 3 │         │
│  │             │  │             │  │             │         │
│  │ Stack: ...  │  │ Stack: ...  │  │ Stack: ...  │         │
│  │ (private)   │  │ (private)   │  │ (private)   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### The Problem Scenario

When multiple goroutines access shared data:

```
Data Segment: count = 5

Goroutine 1 →  Reads count (5)
Goroutine 2 →  Reads count (5)  ← Same time!
Goroutine 3 →  Reads count (5)  ← Same time!

Goroutine 1 →  Adds 1 → 6
Goroutine 2 →  Adds 1 → 6
Goroutine 3 →  Adds 1 → 6

Goroutine 1 →  Writes 6 to count
Goroutine 2 →  Writes 6 to count  ← Overwrites!
Goroutine 3 →  Writes 6 to count  ← Overwrites!

Final: count = 6 (Expected: 8)
```

**This is the race condition!**

---

## The Problem in GetProducts

Now let's look at our actual code problem.

### The Problematic Code

```go
var count int64  // ← Global variable in Data Segment (SHARED!)

func (h *Handler) GetProducts(c *gin.Context) {
    // ... extract page, limit ...
    
    products, _ := h.productService.List(page, limit)
    
    var wg sync.WaitGroup
    
    // Launch goroutine to get count
    wg.Add(1)
    go func() {
        defer wg.Done()
        count1, _ := h.productService.Count()
        count = count1  // ⚠️ RACE CONDITION!
    }()
    
    wg.Wait()
    
    // Use count in response...
}
```

### Why Is This a Problem?

**When multiple requests hit this API:**

```
Request 1 hits API → Creates GetProducts goroutine (G1)
Request 2 hits API → Creates GetProducts goroutine (G2)
Request 3 hits API → Creates GetProducts goroutine (G3)
```

**Each GetProducts goroutine launches ANOTHER goroutine:**

```
G1 launches → Count goroutine (C1)
G2 launches → Count goroutine (C2)
G3 launches → Count goroutine (C3)
```

**All three Count goroutines access the SAME global `count` variable:**

```
C1: count1 = 5      → writes count = 5
C2: count1 = 7      → writes count = 7  (overwrites!)
C3: count1 = 9      → writes count = 9  (overwrites!)
```

**Timing matters!**

```
Nanosecond 4.0: C1 writes count = 5
Nanosecond 5.0: C2 writes count = 7  ← Overwrites!
Nanosecond 5.5: C3 writes count = 9  ← Overwrites!
```

**Now all three requests see `count = 9`:**

```
Request 1: Expected 5,  Got 9  ❌
Request 2: Expected 7,  Got 9  ❌
Request 3: Expected 9,  Got 9  ✓ (lucky!)
```

**This is unpredictable! This is a race condition!**

### The Solution (Preview)

We'll solve this with **Mutex** in the next chapter. For now, let's demonstrate the problem clearly.

---

## Simple Demonstration: Counter Example

Let's create a simple program that clearly shows the race condition.

### The Code

Create a new file `main.go` for testing:

```go
package main

import (
    "fmt"
    "sync"
)

var count int64  // Global variable (shared!)

func main() {
    var wg sync.WaitGroup
    
    // Launch 1,000 goroutines
    for i := 1; i <= 1000; i++ {
        wg.Add(1)
        
        go func() {
            defer wg.Done()
            
            // Each goroutine increments count
            temp := count   // Read
            temp = temp + 1 // Increment
            count = temp    // Write
        }()
    }
    
    // Wait for all goroutines to finish
    wg.Wait()
    
    // Print final count
    fmt.Println("Final count:", count)
}
```

### What Should Happen?

**Expected:**
- 1,000 goroutines
- Each increments count by 1
- Starting from 0
- **Final count should be: 1,000**

### What Actually Happens?

**Let's trace through:**

```
Initial: count = 0

Goroutine 1:  Read 0 → Add 1 → Write 1
Goroutine 2:  Read 0 → Add 1 → Write 1  ← Read old value!
Goroutine 3:  Read 0 → Add 1 → Write 1  ← Read old value!
...
```

**Many goroutines read the same old value!**

**Result: count < 1,000** (some increments are lost!)

---

## Running the Race Condition Test

Let's run our test program and see the race condition in action.

### Run 1: 1,000 Goroutines

```bash
go run main.go
```

**Output:**
```
Final count: 964
```

**Expected: 1,000**
**Actual: 964**
**Lost: 36 increments!** 😱

### Run It Again

```bash
go run main.go
```

**Output:**
```
Final count: 974
```

**Different result!** This is **unpredictable**!

### Run 2: 100,000 Goroutines

Let's make it worse:

```go
for i := 1; i <= 100000; i++ {  // 100,000 instead of 1,000
    // ...
}
```

**Run:**
```bash
go run main.go
```

**Output:**
```
Final count: 91968
```

**Expected: 100,000**
**Actual: 91,968**
**Lost: 8,032 increments!** 💥

**More goroutines = More race conditions = More lost updates!**

### Why This Happens

Let's trace what's happening in nanoseconds:

```
Time   | Goroutine | Action              | count value
-------|-----------|---------------------|-------------
0 ns   | -         | Initial             | 0
1 ns   | G1        | Read: 0             | 0
1 ns   | G2        | Read: 0             | 0  ← Same!
1 ns   | G3        | Read: 0             | 0  ← Same!
2 ns   | G1        | Compute: 0+1=1      | 0
2 ns   | G2        | Compute: 0+1=1      | 0
2 ns   | G3        | Compute: 0+1=1      | 0
3 ns   | G1        | Write: 1            | 1
4 ns   | G2        | Write: 1            | 1  ← No change!
5 ns   | G3        | Write: 1            | 1  ← No change!
```

**Three increments, but count only went from 0 → 1!**

**Two increments were lost!** This happens thousands of times with 100,000 goroutines.

---

## Detecting Race Conditions with -race Flag

Go has a **built-in race detector**! 🔍

### The -race Flag

```bash
go run -race main.go
```

This enables Go's **race detector** which monitors your program and reports race conditions!

### What It Does

The race detector:
1. Instruments your code
2. Tracks all memory accesses
3. Detects when multiple goroutines access the same memory
4. Reports when at least one is writing

### Example Output

```bash
go run -race main.go
```

**Output:**
```
==================
WARNING: DATA RACE
Write at 0x00c000014088 by goroutine 7:
  main.main.func1()
      /path/to/main.go:18 +0x50

Previous write at 0x00c000014088 by goroutine 6:
  main.main.func1()
      /path/to/main.go:18 +0x50

Goroutine 7 (running) created at:
  main.main()
      /path/to/main.go:13 +0x98

Goroutine 6 (finished) created at:
  main.main()
      /path/to/main.go:13 +0x98
==================
Final count: 964
Found 1 data race(s)
```

**The race detector found the problem!** ✓

### Understanding the Output

```
WARNING: DATA RACE
```
→ Go detected a race condition!

```
Write at 0x00c000014088 by goroutine 7:
  main.main.func1()
      /path/to/main.go:18 +0x50
```
→ Goroutine 7 wrote to memory address `0x00c000014088` at line 18

```
Previous write at 0x00c000014088 by goroutine 6:
```
→ Goroutine 6 ALSO wrote to the SAME address!

**Two goroutines writing to the same memory = DATA RACE!** ⚠️

---

## Why Race Conditions Are Dangerous

Race conditions are **EXTREMELY DANGEROUS** in production systems!

### 1. Unpredictable Behavior

```go
// Sometimes works correctly
count: 1000

// Sometimes doesn't
count: 964

// No way to predict!
```

**You can't rely on the results!**

### 2. Silent Data Corruption

```go
// Money transfers
Expected: 8,000 taka
Actual:   6,000 taka
Lost:     2,000 taka  ← Where did it go?!
```

**Data gets corrupted without any error!**

### 3. Hard to Debug

```go
// Works fine on your laptop
count: 1000  ✓

// Fails in production with high traffic
count: 987  ❌

// Can't reproduce locally!
```

**Only appears under high concurrency!**

### 4. Security Vulnerabilities

```go
// Race condition in authentication
isAuthenticated = true  ← Overwritten by another request!
isAuthenticated = false ← Now user is logged out unexpectedly!
```

**Can lead to security breaches!**

### 5. Financial Losses

```go
// E-commerce inventory
Available: 10 items
Orders:    15 placed  ← Oversold!
Result:    Angry customers, financial loss
```

**Real money at stake!**

---

## Rules of Race Conditions

A race condition occurs when **BOTH** of these are true:

### Rule 1: Multiple Concurrent Access

```
Multiple goroutines/threads access the SAME shared data
```

**Example:**
```go
var count int64  // Shared!

go func() { count++ }()  // Goroutine 1 accesses count
go func() { count++ }()  // Goroutine 2 accesses count
go func() { count++ }()  // Goroutine 3 accesses count
```

### Rule 2: At Least One Write

```
At least ONE of them performs a WRITE operation
```

**Examples:**

**✓ Race condition (write present):**
```go
// All reading and writing
go func() { count++ }()  // Read + Write
go func() { count++ }()  // Read + Write
```

**✓ Race condition (one write):**
```go
// One writing, others reading
go func() { fmt.Println(count) }()  // Read only
go func() { fmt.Println(count) }()  // Read only
go func() { count = 10 }()          // WRITE ← Race!
```

**✗ No race condition (all reading):**
```go
// All just reading
go func() { fmt.Println(count) }()  // Read only
go func() { fmt.Println(count) }()  // Read only
go func() { fmt.Println(count) }()  // Read only
// No writes = No race condition ✓
```

### Summary of Rules

```
Race Condition = (Multiple Access) AND (At Least One Write)

If only reading → Safe ✓
If one writer → DANGER ⚠️
```

---

## Summary

Let's recap what we learned about race conditions:

### What Is a Race Condition?

**Race Condition**: Multiple goroutines accessing shared data, with at least one writing, causing unpredictable results.

### How It Happens

1. **Shared data** exists (global variable)
2. **Multiple goroutines** access it simultaneously
3. **Read-Modify-Write** pattern:
   ```
   Read → Modify → Write
   ```
4. **Timing matters** - whoever writes last "wins"

### The Bank Account Example

```
Initial: 5,000 taka
Three customers pay 1,000 each
Expected: 8,000 taka
Actual:   6,000 taka (race condition!)
Lost:     2,000 taka
```

### Memory Layout

```
Process Memory:
┌─────────────────┐
│ Code Segment    │ ← Shared, read-only
├─────────────────┤
│ Data Segment    │ ← Shared, WRITABLE (danger zone!)
├─────────────────┤
│ Stack Segment   │ ← Private per goroutine
├─────────────────┤
│ Heap Segment    │ ← Can be shared
└─────────────────┘
```

### The Counter Test

```go
// 1,000 goroutines incrementing count
Expected: 1,000
Actual:   964  (36 lost!)

// 100,000 goroutines
Expected: 100,000
Actual:   91,968  (8,032 lost!)
```

### Detection

```bash
# Use -race flag to detect race conditions
go run -race main.go

# Output: WARNING: DATA RACE
```

### Two Rules

Race condition requires **BOTH**:
1. Multiple concurrent access to shared data
2. At least one write operation

---

## What's Next

Now that you understand the **problem**, let's learn the **solution**!

### Chapter 68: Mutex - The Solution to Race Conditions

In the next chapter, we'll learn:

**Mutex (Mutual Exclusion)**:
- What is a mutex?
- How mutex prevents race conditions
- `sync.Mutex` in Go
- `Lock()` and `Unlock()` methods
- Critical sections
- Protecting shared data

**Example:**
```go
var count int64
var mu sync.Mutex  // ← The solution!

go func() {
    mu.Lock()      // ← Only one goroutine at a time!
    count++
    mu.Unlock()
}()
```

**Result:**
```
With race condition:  964 / 1,000  ❌
With mutex:           1,000 / 1,000  ✓
```

### After Mutex

**Chapter 69: RWMutex** - Optimized for read-heavy workloads
**Chapter 70: Channels** - The Go way of handling concurrency
**Chapter 71: Select Statement** - Multiplexing channels

---

## Practice Exercise

Try this yourself!

### Exercise 1: Reproduce the Race

Create this program:

```go
package main

import (
    "fmt"
    "sync"
)

var balance int64 = 5000  // Starting balance

func main() {
    var wg sync.WaitGroup
    
    // 3 customers each deposit 1,000
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go func(customerID int) {
            defer wg.Done()
            
            // Read balance
            temp := balance
            
            // Add deposit
            temp = temp + 1000
            
            // Write back
            balance = temp
            
            fmt.Printf("Customer %d deposited 1,000\n", customerID)
        }(i)
    }
    
    wg.Wait()
    
    fmt.Printf("\nExpected balance: 8,000\n")
    fmt.Printf("Actual balance:   %d\n", balance)
}
```

**Run it multiple times:**
```bash
go run main.go
go run main.go
go run main.go
```

**Do you get 8,000 every time?** Probably not! That's the race condition!

### Exercise 2: Detect with -race

```bash
go run -race main.go
```

**Can you see the WARNING: DATA RACE message?**

---

## Final Thoughts

Race conditions are:
- **Common** - Easy to introduce accidentally
- **Dangerous** - Cause data corruption
- **Subtle** - Hard to detect and debug
- **Critical** - Must be handled in production

**Good news:** Go provides excellent tools to:
1. **Detect** race conditions (`-race` flag)
2. **Prevent** race conditions (Mutex, Channels)
3. **Reason about** concurrent code

**In the next chapter**, we'll fix this problem using **Mutex**!

Until then, remember:
- Shared data + Multiple goroutines + Writing = **DANGER!** ⚠️
- Always think about concurrency when using global variables
- Use `-race` flag during development and testing

**Stay safe, and see you in the next chapter!** 🚀

---

**Next Chapter:** [Chapter 68: Mutex - Protecting Shared Data](chapter68_readme.md)

---

*Remember: The race is not always to the swift... but to those who use proper synchronization!* 😄
