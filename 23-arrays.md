# Chapter 23: Arrays — The Flower Garland Analogy

> **Goal of this chapter:** Learn to store **many values of the same type** under one name. You'll master fixed-size **arrays**: declaring them, indexing (starting at zero), looping over them, copying them, and how they sit in memory. Arrays are the foundation for **slices** (Chapter 25), the collection you'll use most.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 2 hours  **Prerequisite:** [Chapter 3](03-making-decisions.md) (loops), [Chapter 21](21-structs.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why arrays?](#2-why-arrays)
3. [The flower-garland analogy](#3-the-flower-garland-analogy)
4. [Declaring arrays](#4-declaring-arrays)
5. [Indexing and zero-based counting](#5-indexing-and-zero-based-counting)
6. [Reading and writing elements](#6-reading-and-writing-elements)
7. [Length is part of the type](#7-length-is-part-of-the-type)
8. [Looping over arrays](#8-looping-over-arrays)
9. [Memory model](#9-memory-model)
10. [Arrays are values: copying](#10-arrays-are-values-copying)
11. [Comparing arrays](#11-comparing-arrays)
12. [Multi-dimensional arrays](#12-multi-dimensional-arrays)
13. [Out-of-bounds errors](#13-out-of-bounds-errors)
14. [Global vs. local arrays](#14-global-vs-local-arrays)
15. [Arrays' limitations, and why slices exist](#15-limitations-and-why-slices-exist)
16. [Common mistakes](#16-common-mistakes)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. What you will learn

- What an **array** is and when to use one
- All the ways to declare and initialize arrays
- **Zero-based indexing** and why it exists
- That arrays are **values** (copied on assignment) with a **fixed length** that's part of their **type**
- How arrays look in memory: one contiguous block
- What happens when you index past the end
- Why you'll usually prefer slices, and what arrays are still good for

---

## 2. Why arrays?

Suppose you want to store the marks of 5 students. With what we know:

```go
mark1 := 85
mark2 := 90
mark3 := 72
mark4 := 66
mark5 := 95
```

Five variables. Now imagine 500 students, or asking "what's the average?" Impossible to loop over `mark1 … mark500`.

An **array** stores **many values of the same type** in **one variable**, side by side, with a **number (index)** to pick each one:

```go
marks := [5]int{85, 90, 72, 66, 95}
fmt.Println(marks[2]) // 72
```

You can loop over it, pass it to a function, compute averages, and so on.

---

## 3. The flower-garland analogy

Picture a **garland** (a string of flowers):

```
  🌺 ─ 🌺 ─ 🌺 ─ 🌺 ─ 🌺
  #0    #1    #2    #3    #4
```

- The garland has a **fixed number of flowers** (5); you can't stretch it by adding a sixth without making a new garland.
- All flowers are the **same kind** (all roses): like all array elements share one type.
- Flowers are **in a row**, one after another with no gaps.
- You refer to a flower by its **position**: "the third flower".

An array is exactly that: a fixed-length, same-type, contiguous sequence with numbered positions.

---

## 4. Declaring arrays

The type of an array is written `[N]T`: *N elements, each of type T*.

### 4.1 `var` with a length: elements start at zero values

```go
package main

import "fmt"

func main() {
	var marks [5]int
	fmt.Println(marks) // [0 0 0 0 0]

	var names [3]string
	fmt.Printf("%q\n", names) // ["" "" ""]

	var flags [2]bool
	fmt.Println(flags) // [false false]
}
```

### 4.2 Array literal: give the values

```go
package main

import "fmt"

func main() {
	marks := [5]int{85, 90, 72, 66, 95}
	fmt.Println(marks) // [85 90 72 66 95]
}
```

### 4.3 Fewer values than the length: the rest are zero

```go
scores := [5]int{10, 20} // [10 20 0 0 0]
```

More values than the length is a compile error.

### 4.4 Let the compiler count: `[...]`

```go
days := [...]string{"Mon", "Tue", "Wed"} // the length (3) is inferred
```

The result is still a fixed-size `[3]string`; `...` just saves you counting.

### 4.5 Set specific indexes

```go
package main

import "fmt"

func main() {
	a := [5]int{1: 10, 3: 30} // index: value
	fmt.Println(a)            // [0 10 0 30 0]
}
```

Useful for sparse arrays or for arrays indexed by named constants.

### 4.6 The long and short forms

```go
var a [3]int = [3]int{1, 2, 3} // long
var b = [3]int{1, 2, 3}        // inferred type
c := [3]int{1, 2, 3}           // short (inside functions), most common
```

### Syntax summary

| Form | Example | Notes |
|------|---------|-------|
| Zero-filled | `var a [5]int` | Every element is 0 |
| Literal | `a := [3]int{1, 2, 3}` | Most common |
| Inferred length | `a := [...]int{1, 2, 3}` | Compiler counts |
| Indexed literal | `a := [5]int{1: 9}` | Others zero |
| Array of structs | `[2]User{{"A", 1}, {"B", 2}}` | Element type can be anything |

The **length must be a constant** known at compile time: `[n]int` with a variable `n` is an error.

---

## 5. Indexing and zero-based counting

Elements are numbered starting at **0**, not 1. An array of length `N` has valid indexes `0` to `N-1`.

```
marks := [5]int{85, 90, 72, 66, 95}

index:   0    1    2    3    4
       ┌────┬────┬────┬────┬────┐
       │ 85 │ 90 │ 72 │ 66 │ 95 │
       └────┴────┴────┴────┴────┘
```

### The building-floor analogy 🏢

In many countries, the floor you enter at street level is the **ground floor (0)**. The next one *up* is floor 1. The "first" floor you climb to has *one* staircase behind you. Index = **distance from the start** (how many steps you move), not "which one is it":

- `marks[0]` → the element **0 steps** from the start (the first).
- `marks[3]` → **3 steps** from the start (the fourth).

### Why zero-based?

- The computer finds an element by **address = start + index × element size**. Index 0 means "no offset", the very start of the block.
- It makes many algorithms simpler (`i` goes from `0` to `len-1`).

Almost every mainstream language (C, Java, Python, JavaScript, Go) does this. It takes a day to get used to.

---

## 6. Reading and writing elements

```go
package main

import "fmt"

func main() {
	marks := [5]int{85, 90, 72, 66, 95}

	fmt.Println(marks[0]) // 85  (read the first)
	fmt.Println(marks[4]) // 95  (read the last)

	marks[2] = 80         // write: replace 72 with 80
	marks[3] += 5         // read-modify-write: 66 → 71
	fmt.Println(marks)    // [85 90 80 71 95]

	fmt.Println(len(marks)) // 5
	last := marks[len(marks)-1]
	fmt.Println(last)       // 95: the idiom for "last element"
}
```

Each element is an ordinary variable of the element type; `marks[2]++`, `marks[i] = x`, and `&marks[2]` (its address) all work.

---

## 7. Length is part of the type

`[3]int` and `[5]int` are **different types**.

```go
// INTENTIONAL ERROR
package main

func main() {
	var a [3]int
	var b [5]int
	a = b // ❌ cannot use b (variable of type [5]int) as [3]int value in assignment
	_ = a
}
```

This means a function taking `[3]int` accepts only arrays of exactly 3 ints. (Another reason slices, which have no length in the type, are so much more flexible.)

`len(a)` is a **compile-time constant** for arrays, so you can even use it in constant expressions and other array sizes.

---

## 8. Looping over arrays

### 8.1 Classic `for` with an index

```go
package main

import "fmt"

func main() {
	marks := [5]int{85, 90, 72, 66, 95}

	total := 0
	for i := 0; i < len(marks); i++ {
		total += marks[i]
	}
	fmt.Println("total:", total)                      // 408
	fmt.Println("average:", float64(total)/float64(len(marks))) // 81.6
}
```

### 8.2 `for … range` ✅ (idiomatic)

`range` gives you **index and value** for each element:

```go
package main

import "fmt"

func main() {
	fruits := [3]string{"apple", "banana", "cherry"}

	for i, fruit := range fruits {
		fmt.Println(i, fruit)
	}
}
```

Output:

```
0 apple
1 banana
2 cherry
```

Variations:

```go
for i := range fruits { }         // index only
for _, fruit := range fruits { }  // value only (blank identifier discards the index)
for range fruits { }              // just repeat len(fruits) times
```

> ⚠️ `range` over an array iterates over a **copy** of the array. Modifying `fruits` inside the loop doesn't affect the values you get. (Range over a slice, next chapters, or over a pointer to the array, sees changes.)

---

## 9. Memory model

```go
package main

import "fmt"

func main() {
	marks := [5]int{85, 90, 72, 66, 95}
	fmt.Println(marks[2])
}
```

An array occupies **one contiguous block** of memory, big enough for all elements. On a 64-bit machine an `int` is 8 bytes, so `[5]int` takes 40 bytes:

```
STACK (main's frame)
 marks: address starts at 1000
 ┌────────┬────────┬────────┬────────┬────────┐
 │  85    │  90    │  72    │  66    │  95    │
 └────────┴────────┴────────┴────────┴────────┘
  1000     1008     1016     1024     1032       ← addresses (8 bytes apart)
  [0]      [1]      [2]      [3]      [4]
```

How is `marks[2]` found? **address = start + index × size = 1000 + 2 × 8 = 1016.** One multiplication and one add: it takes the same time for index 2 or index 2,000,000. That's why arrays give **O(1) random access**.

**Why contiguity matters:**
- Fast: CPUs love reading neighboring memory (cache lines).
- Predictable size: `unsafe.Sizeof([5]int{})` is exactly `5 × 8 = 40`.
- The whole array is **one value**: variable `marks` *is* all five numbers, not a pointer to them (a big difference from Java/Python/JS arrays).

**Where does it live?** A local array is in the **stack frame** (or on the heap if it escapes or is huge). A package-level array is in the **data segment**.

---

## 10. Arrays are values: copying

Because the variable *is* the whole array, **assignment copies every element**:

```go
package main

import "fmt"

func main() {
	a := [3]int{1, 2, 3}
	b := a // copies all three elements

	b[0] = 99
	fmt.Println(a) // [1 2 3]  ← unchanged
	fmt.Println(b) // [99 2 3]
}
```

```
a: [ 1 | 2 | 3 ]
b: [99 | 2 | 3 ]       two separate blocks
```

Passing to a function copies, too:

```go
package main

import "fmt"

func zero(arr [3]int) {
	arr[0] = 0 // changes the function's COPY
}

func main() {
	a := [3]int{1, 2, 3}
	zero(a)
	fmt.Println(a) // [1 2 3]
}
```

To modify the caller's array, pass a **pointer** to it: `func zero(arr *[3]int) { arr[0] = 0 }`, called as `zero(&a)`. (Go allows `arr[0]` on a pointer to an array as shorthand.) Or, more commonly, use a slice.

> Copying a huge array (say `[1000000]int` = 8 MB) on every call is expensive. Another reason slices and pointers are preferred.

---

## 11. Comparing arrays

Arrays of comparable elements can be compared with `==` (same length required). They're equal when **all elements are equal**:

```go
package main

import "fmt"

func main() {
	a := [3]int{1, 2, 3}
	b := [3]int{1, 2, 3}
	c := [3]int{1, 2, 4}

	fmt.Println(a == b) // true
	fmt.Println(a == c) // false
}
```

This also makes arrays usable as **map keys**: `map[[2]int]string{{0, 0}: "origin"}`.

---

## 12. Multi-dimensional arrays

An array of arrays: a grid, board, or matrix.

```go
package main

import "fmt"

func main() {
	var grid [3][4]int // 3 rows, 4 columns

	for r := 0; r < 3; r++ {
		for c := 0; c < 4; c++ {
			grid[r][c] = r * c
		}
	}

	for _, row := range grid {
		fmt.Println(row)
	}
}
```

Output:

```
[0 0 0 0]
[0 1 2 3]
[0 2 4 6]
```

```
grid[r][c]:      c=0 c=1 c=2 c=3
           r=0 [  0   0   0   0 ]
           r=1 [  0   1   2   3 ]
           r=2 [  0   2   4   6 ]
```

Literal form:

```go
board := [3][3]string{
	{"X", "O", "X"},
	{" ", "X", " "},
	{"O", " ", "O"},
}
```

In memory it's still one contiguous block (row after row).

---

## 13. Out-of-bounds errors

Valid indexes for `[5]int` are `0..4`.

### 13.1 Constant index: caught by the compiler

```go
// INTENTIONAL ERROR
package main

import "fmt"

func main() {
	marks := [5]int{1, 2, 3, 4, 5}
	fmt.Println(marks[5]) // ❌ invalid argument: index 5 out of bounds [0:5]
}
```

### 13.2 Variable index: caught at run time (a panic)

```go
package main

import "fmt"

func main() {
	marks := [5]int{1, 2, 3, 4, 5}
	i := 5
	defer func() { fmt.Println("recovered:", recover()) }()
	fmt.Println(marks[i]) // runtime panic
}
```

Output:

```
recovered: runtime error: index out of range [5] with length 5
```

Without the `recover`, the program **crashes** with `panic: runtime error: index out of range [5] with length 5` plus a stack trace (Chapter 11). Go always bounds-checks, unlike C, which would silently read garbage memory (a classic security bug).

**Fix:** loop with `i < len(a)`, or use `range`, and validate user-supplied indexes.

---

## 14. Global vs. local arrays

```go
package main

import "fmt"

var primes = [5]int{2, 3, 5, 7, 11} // package level → data segment

func main() {
	local := [3]int{1, 2, 3} // function level → stack frame
	fmt.Println(primes, local)
}
```

| | Package-level array | Local array |
|--|---------------------|-------------|
| Memory | **Data segment** | **Stack frame** (or heap if it escapes) |
| Lifetime | Whole program | Until function returns |
| Visible to | Whole package | That function only |
| Use `:=`? | ❌ Must use `var` | ✅ |

The same scoping rules as for any variable (Chapters 8–12) apply.

---

## 15. Limitations, and why slices exist

Arrays are rigid:

| Limitation | Consequence |
|------------|-------------|
| **Fixed length** | You can't append. Adding an element means making a new array |
| **Length in the type** | `[3]int` ≠ `[4]int`, so you can't write one function for "any-length int array" |
| **Copy on assign/pass** | Large arrays are costly to move around |

So in everyday Go you'll use **slices** (`[]int`), a flexible, growable *view* over an array, with a length that's *not* part of the type. Slices are the subject of **Chapter 25**, and their design makes sense once you know arrays.

**Where arrays are still the right tool:**
- A truly fixed size (a chess board `[8][8]`, RGB `[3]uint8`, an IPv4 address `[4]byte`, a SHA-256 hash `[32]byte`)
- Small values you want copied (value semantics)
- Comparison and use as map keys (`[2]int` coordinates)
- Performance-sensitive code with known sizes

---

## 16. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Off-by-one: using `marks[len(marks)]` | `index out of range` | Last index is `len(marks)-1` |
| 2 | Treating index as starting at 1 | Wrong element / out of range | Count from 0 |
| 3 | Assuming assignment shares data | Modifying one array doesn't change the other | Arrays copy. Use a pointer/slice to share |
| 4 | `var a [n]int` with variable `n` | `invalid array length n` | Use a constant, or use a slice: `make([]int, n)` |
| 5 | Passing an array and expecting the function to change it | Caller's array unchanged | Pass `*[N]T` or use a slice |
| 6 | Mismatched lengths: `[3]int` vs `[5]int` | `cannot use ... as ... value` | They're different types |
| 7 | Too many literal values | `index 3 out of bounds` (for `[3]int{1,2,3,4}`) | Match the length or use `[...]` |
| 8 | Using `range` and modifying the array expecting to see updates | `range` iterates over a copy of arrays | Iterate over `&arr` or a slice |
| 9 | Using `:=` for a package-level array | syntax error | `var a = [3]int{...}` |
| 10 | Comparing arrays of different lengths | compile error | Not allowed |

---

## 17. Exercises

### Exercise 1: Create and print
Create an array of your 5 favorite numbers and print the first, last, and the length.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	nums := [5]int{3, 7, 11, 21, 42}
	fmt.Println(nums[0], nums[len(nums)-1], len(nums)) // 3 42 5
}
```
</details>

### Exercise 2: Sum and average
Compute the sum and average of `[6]float64{2.5, 3.5, 4, 5, 6.5, 8}`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	a := [6]float64{2.5, 3.5, 4, 5, 6.5, 8}
	sum := 0.0
	for _, v := range a {
		sum += v
	}
	fmt.Printf("sum=%.1f avg=%.2f\n", sum, sum/float64(len(a))) // sum=29.5 avg=4.92
}
```
</details>

### Exercise 3: Find the maximum and its index

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	a := [...]int{4, 19, 7, 33, 12}
	maxIdx := 0
	for i := range a {
		if a[i] > a[maxIdx] {
			maxIdx = i
		}
	}
	fmt.Println("max", a[maxIdx], "at index", maxIdx) // max 33 at index 3
}
```
</details>

### Exercise 4: Reverse in place

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	a := [5]int{1, 2, 3, 4, 5}
	for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
		a[i], a[j] = a[j], a[i]
	}
	fmt.Println(a) // [5 4 3 2 1]
}
```
</details>

### Exercise 5: Predict (copy semantics)

```go
package main

import "fmt"

func modify(arr [3]int) [3]int {
	arr[0] = 100
	return arr
}

func main() {
	a := [3]int{1, 2, 3}
	b := modify(a)
	fmt.Println(a, b)
}
```

<details><summary>Solution</summary>

`[1 2 3] [100 2 3]`. `modify` received a copy; we captured the modified copy in `b`.
</details>

### Exercise 6: Tic-tac-toe board
Create a `[3][3]string` initialized with `"."`, place `"X"` at the center, and print the board row by row.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	var board [3][3]string
	for r := range board {
		for c := range board[r] {
			board[r][c] = "."
		}
	}
	board[1][1] = "X"

	for _, row := range board {
		fmt.Println(strings.Join(row[:], " "))
	}
}
```
Output:
```
. . .
. X .
. . .
```
(`row[:]` turns the array into a slice, which `strings.Join` needs. More in Chapter 25.)
</details>

### Exercise 7 (challenge): Frequency of digits
Given a `[10]int` of counters and the string `"1233"`, count how often each digit appears using the digit as an index.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	var counts [10]int
	for _, ch := range "1233421" {
		counts[ch-'0']++ // '3' - '0' == 3: the character code doubles as an index
	}
	fmt.Println(counts) // [0 2 2 2 1 0 0 0 0 0]
}
```
</details>

---

## 18. Quiz

1. What's the index of the first element? Of the last in `[7]int`?
2. Is `[3]int` the same type as `[4]int`?
3. What happens on `b := a` where `a` is an array?
4. Can an array's length be a variable?
5. What happens when you use an out-of-range variable index?
6. Why is random access to an array element O(1)?

<details><summary>Answers</summary>

1. `0`; `6`.
2. No, the length is part of the type.
3. All elements are copied into a new independent array.
4. No, it must be a compile-time constant.
5. A run-time panic (`index out of range`).
6. Address = start + index × element size, a single calculation.
</details>

---

## 19. Summary

- An **array** `[N]T` is a **fixed-length**, **same-type**, **contiguous** sequence of elements, indexed from **0** to `N-1`.
- Declare with `var a [5]int`, `a := [3]int{1,2,3}`, `[...]int{…}`, or `[5]int{1: 9}`. Length must be a **constant**.
- The **length is part of the type**: `[3]int` ≠ `[4]int`.
- Arrays are **values**: assigning or passing **copies** every element. `==` compares element-wise.
- Random access is **O(1)** thanks to contiguous layout; out-of-bounds indexes **panic** at run time (or fail to compile if constant).
- Loop with `for i := range a` / `for i, v := range a`.
- Arrays are rigid, so day-to-day code uses **slices**, built on arrays (Chapter 25), after we understand **pointers** (Chapter 24).

### ➡️ What's next?

Arrays taught us that variables live at addresses, and that passing copies has costs. [Chapter 24](24-pointers.md) introduces **pointers**, the tool for sharing and modifying data in place.
