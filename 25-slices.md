# Chapter 25: Slices — The Most Important Interview Topic

> **Goal of this chapter:** Master Go's workhorse collection. A **slice** is a flexible, growable *window* onto an underlying array. You'll learn its three-part internals (pointer, length, capacity), five ways to create slices, how `append` grows them, how slices *share* memory (the source of the famous gotchas), and the everyday operations: copy, delete, insert, filter, and 2D slices.

**Difficulty:** 🔴 Intermediate–Advanced  **Estimated time:** 3–4 hours  **Prerequisite:** [Chapter 23 (arrays)](23-arrays.md) and [Chapter 24 (pointers)](24-pointers.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why slices matter](#2-why-slices-matter)
3. [What is a slice? The pizza analogy](#3-what-is-a-slice)
4. [Slice internals: pointer, length, capacity](#4-slice-internals)
5. [Creating slices: 5 ways](#5-creating-slices)
6. [Memory model: slicing an array](#6-memory-model-slicing-an-array)
7. [Length vs. capacity](#7-length-vs-capacity)
8. [Slicing a slice](#8-slicing-a-slice)
9. [`append` and growth](#9-append-and-growth)
10. [Shared memory: the aliasing gotchas](#10-shared-memory-the-aliasing-gotchas)
11. [Slices and functions](#11-slices-and-functions)
12. [`copy`](#12-copy)
13. [Everyday recipes](#13-everyday-recipes)
14. [`nil` slice vs. empty slice](#14-nil-slice-vs-empty-slice)
15. [Iterating with `range`](#15-iterating-with-range)
16. [Two-dimensional slices](#16-two-dimensional-slices)
17. [Memory leaks and the three-index slice](#17-memory-leaks-and-the-three-index-slice)
18. [The `slices` package](#18-the-slices-package)
19. [Interview questions](#19-interview-questions)
20. [Common mistakes](#20-common-mistakes)
21. [Exercises](#21-exercises)
22. [Quiz](#22-quiz)
23. [Summary](#23-summary)

---

## 1. What you will learn

- What a slice really is (a small **header** describing part of an array)
- The difference between **length** and **capacity**
- Every way to create a slice: literal, `make`, from an array, from another slice, `append`
- How `append` works, when it allocates a new array, and why that matters
- Why two slices can **share** the same data (and how that bites)
- How to copy, delete, insert, filter, and reverse
- Common interview traps, with worked answers

---

## 2. Why slices matter

In real Go code you will use **slices constantly** and arrays rarely:

- `[]string` for lists of names, arguments (`os.Args`), lines of a file
- `[]byte` for raw data (files, network)
- `[]User` for database results and JSON arrays
- Every variadic function receives a slice (Chapter 6)

They are also **the** favorite topic of Go interviewers, because correct answers require understanding memory, pointers, and copying, all the things Chapters 18–24 built up.

---

## 3. What is a slice?

An array (Chapter 23) has a **fixed** size that's part of its type. A **slice** has no fixed size in its type (`[]int`), and can grow.

> A **slice** doesn't own data. It's a **view** (window) onto a section of an underlying **array**.

### The pizza analogy 🍕

Imagine a **whole pizza** cut into 8 slices sitting in the box: that's the **array**, a fixed piece of memory.

- A **slice** of the pizza is *not a new pizza*. It points to some pieces *of the same pizza*.
- Take slices 2–4: you've got a "window" over the box's contents.
- If someone eats piece 3 from the box, your "window" no longer has it: **you were looking at the same pizza**.
- You can also say "and I'm allowed to reach further along the box" (that's **capacity**).

So two slices covering overlapping pieces **share** those pieces. Change a piece through one, and the other sees the change.

```
array (the box):   [ 10 | 20 | 30 | 40 | 50 ]
                          └────────┘
slice s = arr[1:4]:      [ 20   30   40 ]   ← a window onto the same memory
```

---

## 4. Slice internals

A slice is a tiny struct of **three fields**, called the **slice header**:

```go
// conceptually (from the Go runtime):
type slice struct {
	ptr *T   // pointer to the FIRST element the slice can see (inside the array)
	len int  // number of elements the slice currently contains
	cap int  // number of elements from ptr to the END of the underlying array
}
```

On a 64-bit machine that's `8 + 8 + 8 = 24 bytes`, regardless of how much data it refers to. You can check:

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	var s []int
	fmt.Println(unsafe.Sizeof(s)) // 24
}
```

### The three components

| Field | Meaning | Get it with |
|-------|---------|-------------|
| **Pointer** | Address of the slice's first element in the underlying array | (hidden) |
| **Length** | How many elements you can **read/write by index** now | `len(s)` |
| **Capacity** | How many elements exist from the first one to the end of the underlying array, i.e., how far you can grow **without** reallocating | `cap(s)` |

Visual:

```
underlying array:    [ 10 ][ 20 ][ 30 ][ 40 ][ 50 ]
                             ▲
slice header s:  ptr ────────┘
                 len = 3   (sees 20, 30, 40)
                 cap = 4   (from 20 to the end: 20,30,40,50)
```

Everything else in this chapter follows from this picture. **When you pass or assign a slice, you copy the 24-byte header, not the data.**

---

## 5. Creating slices

### Method 1: from an array

```go
package main

import "fmt"

func main() {
	arr := [5]int{10, 20, 30, 40, 50}

	s := arr[1:4] // elements at indexes 1, 2, 3 (4 is excluded!)
	fmt.Println(s, len(s), cap(s)) // [20 30 40] 3 4
}
```

**Slice expression `a[low:high]`:** includes `low`, **excludes** `high`. Length is `high - low`. Capacity is `cap(a) - low`.

Shortcuts:

```go
arr[:3]   // from the start: [10 20 30]
arr[2:]   // to the end:     [30 40 50]
arr[:]    // everything:     [10 20 30 40 50]
```

### Method 2: from another slice

```go
s2 := s[1:3] // slicing a slice: same underlying array again
```

(See section 8.)

### Method 3: slice literal ⭐ (most common)

```go
nums := []int{1, 2, 3, 4, 5}       // note: [] with NO length
names := []string{"Asha", "Rahim"}
```

Behind the scenes Go allocates an array of 5 ints and makes a slice pointing to all of it (`len = cap = 5`). If you're used to array literals, the only difference is `[]` instead of `[5]`.

### Method 4: `make([]T, length)`

```go
s := make([]int, 3) // [0 0 0]: length 3, capacity 3, zero-filled
```

Use when you know how many elements you need and will assign by index.

### Method 5: `make([]T, length, capacity)`

```go
s := make([]int, 0, 10) // [] : length 0, capacity 10: room reserved
```

Use when you'll `append` up to about `capacity` items. It avoids repeated re-allocation.

```go
package main

import "fmt"

func main() {
	a := make([]int, 3)
	b := make([]int, 0, 10)
	fmt.Println(a, len(a), cap(a)) // [0 0 0] 3 3
	fmt.Println(b, len(b), cap(b)) // [] 0 10
}
```

⚠️ **`make([]int, 0, 10)` vs `make([]int, 10)`** is a super common confusion:

```go
b := make([]int, 0, 10)
// b[0] = 5   // ❌ PANIC: index out of range [0] with length 0  (length is 0!)
b = append(b, 5) // ✅ correct way to add
```

Capacity is *reserved space*, not usable elements. Only elements within **length** are accessible by index.

### Summary

| Method | Example | len | cap |
|--------|---------|-----|-----|
| Literal | `[]int{1,2,3}` | 3 | 3 |
| `make(T, n)` | `make([]int, 3)` | 3 | 3 |
| `make(T, n, c)` | `make([]int, 0, 10)` | 0 | 10 |
| From array `a[l:h]` | `arr[1:4]` | h−l | cap(a)−l |
| From slice `s[l:h]` | `s[1:3]` | h−l | cap(s)−l |
| `var s []int` | nil slice | 0 | 0 |

---

## 6. Memory model: slicing an array

```go
package main

import "fmt"

func main() {
	arr := [5]int{10, 20, 30, 40, 50}
	s := arr[1:4]
	fmt.Println(s, len(s), cap(s))
}
```

**Step 1: the array.** `arr` is a real array: five ints in one block on `main`'s stack frame (Chapter 23):

```
STACK (main's frame)
 arr:  ┌────┬────┬────┬────┬────┐
       │ 10 │ 20 │ 30 │ 40 │ 50 │
       └────┴────┴────┴────┴────┘
 addr:  1000 1008 1016 1024 1032
```

**Step 2: the slice.** `s := arr[1:4]` creates **only a 24-byte header**. It copies no data:

```
 arr:  ┌────┬────┬────┬────┬────┐
       │ 10 │ 20 │ 30 │ 40 │ 50 │
       └────┴────┴────┴────┴────┘
 addr:  1000 1008 1016 1024 1032
                ▲
 s:  ┌──────────┴──────────┐
     │ ptr = 1008          │
     │ len = 3   (4 − 1)   │
     │ cap = 4   (5 − 1)   │
     └─────────────────────┘
```

- `ptr` points at `arr[1]` (address 1008).
- `len = 3`: `s[0], s[1], s[2]` → `20, 30, 40`.
- `cap = 4`: from `arr[1]` to the end of `arr` there are four elements. `s` could grow to include `50` without reallocating.

`fmt.Println(s, len(s), cap(s))` prints `[20 30 40] 3 4` (real output).

**Step 3: writes go through to the array.**

```go
s[0] = 999
fmt.Println(arr) // [10 999 30 40 50]
```

Same memory: `s[0]` **is** `arr[1]`.

---

## 7. Length vs. capacity

|  | Length | Capacity |
|--|--------|----------|
| Meaning | Elements in the slice **now** | Elements available **from ptr to the end of the array** |
| Access | Index only `0 … len-1` | Reachable by reslicing (`s[:cap(s)]`) or `append` |
| Changes when | you `append` or reslice | the slice moves start / a new array is allocated |

```go
package main

import "fmt"

func main() {
	s := make([]int, 2, 5) // len 2, cap 5
	fmt.Println(s, len(s), cap(s)) // [0 0] 2 5

	// s[2] = 1          // ❌ panic: index out of range [2] with length 2
	t := s[:4]           // ✅ extend the VIEW up to capacity
	fmt.Println(t, len(t), cap(t)) // [0 0 0 0] 4 5
}
```

**Interview trap:** *"What is the length and capacity of `s := make([]int, 3, 10)`?"* → 3 and 10. *"Can you do `s[5] = 1`?"* → **No** (panic): only indexes below the length are valid.

---

## 8. Slicing a slice

You can slice a slice again. The new slice **shares the same underlying array**:

```go
package main

import "fmt"

func main() {
	arr := [5]int{10, 20, 30, 40, 50}
	s := arr[1:4] // [20 30 40], len 3, cap 4

	t := s[1:3] // relative to s: s[1], s[2] → [30 40]
	fmt.Println(t, len(t), cap(t)) // [30 40] 2 3

	t[0] = 0
	fmt.Println(s)   // [20 0 40]
	fmt.Println(arr) // [10 20 0 40 50]
}
```

Indexes in `s[1:3]` are **relative to `s`**, not to the original array. New `len = 3 − 1 = 2`; new `cap = cap(s) − 1 = 3`.

```
arr:  [10][20][30][40][50]
s  :      [20  30  40]        len 3 cap 4
t  :           [30  40]       len 2 cap 3   ← starts at s[1]
```

**Slicing can only go forward from the start:** there is no way to slice *before* `ptr` (the elements to the left are unreachable from `s`).

---

## 9. `append` and growth

`append` adds elements to the end of a slice and **returns the (possibly new) slice**. You must use the result:

```go
s = append(s, 4)          // add one
s = append(s, 5, 6, 7)    // add several
s = append(s, other...)   // add all elements of another slice
```

### Case A: room in the capacity → no new array

```go
package main

import "fmt"

func main() {
	s := make([]int, 2, 5) // len 2, cap 5
	s = append(s, 7)
	fmt.Println(s, len(s), cap(s)) // [0 0 7] 3 5
}
```

`append` just writes into the next free spot of the **same array** and returns a header with `len+1`. The same array, same `cap`.

```
before: array [0][0][ ][ ][ ]   header len 2 cap 5
after : array [0][0][7][ ][ ]   header len 3 cap 5   (same array!)
```

### Case B: no room → allocate a bigger array and copy

```go
package main

import "fmt"

func main() {
	s := []int{1, 2, 3} // len 3, cap 3 (full)
	fmt.Println(len(s), cap(s)) // 3 3

	s = append(s, 4) // no room!
	fmt.Println(len(s), cap(s)) // 4 6
}
```

When `len == cap`, `append`:

1. Allocates a **new, larger** array,
2. **Copies** all elements into it,
3. Adds the new element,
4. Returns a slice whose pointer points at the **new** array.

```
before: old array [1][2][3]            s ─► old
after : new array [1][2][3][4][ ][ ]   s ─► new  (cap 6)
        old array [1][2][3]  (still there, if anything else refers to it; else garbage)
```

### How much bigger?

For small slices the capacity roughly **doubles**; for larger slices (past ~256 elements) it grows by a smaller factor (about 1.25× plus a constant) to avoid wasting memory. Typical growth of an `[]int` built by repeated `append` on a modern Go release (real output):

```
1 2 4 8 16 32 64 128 256 512 848 1280 1792 2560
```

> ⚠️ **Never depend on exact numbers.** The growth formula changed in Go 1.18, memory allocators round sizes up to allocator "size classes" (for a `[]string` the sequence reaches `2 4 8 16 32 71`, not `64`), and newer Go versions may start a small slice with room for several elements. The **guarantee** is only: growth is *amortized* efficient, and `cap >= len` always.

**Why doubling?** Copying on every append would be O(n²) overall. Growing geometrically means each element is copied only a few times on average, so **appending n items costs O(n) total** ("amortized O(1) per append").

### Pre-allocating when you know the size

```go
result := make([]int, 0, len(input)) // reserve space once
for _, v := range input {
	result = append(result, v*2)
}
```

This avoids repeated re-allocation and copying.

### `append` to a nil slice works

```go
var s []int // nil
s = append(s, 1, 2, 3)
```

No `make` needed; that's why `var results []string` followed by appends is idiomatic Go.

---

## 10. Shared memory: the aliasing gotchas

Because slices share arrays, surprising things can happen. Learn these patterns; interviewers love them.

### Gotcha 1: writing through one slice changes the other

```go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4, 5}
	b := a[1:3] // [2 3]

	b[0] = 99
	fmt.Println(a) // [1 99 3 4 5]
	fmt.Println(b) // [99 3]
}
```

### Gotcha 2: `append` may or may not affect the original

```go
package main

import "fmt"

func main() {
	arr := [5]int{10, 20, 30, 40, 50}
	s := arr[1:4] // [20 30 40], len 3, cap 4

	s = append(s, 99) // cap available → writes INTO arr[4]
	fmt.Println(arr, s) // [10 20 30 40 99] [20 30 40 99]

	s = append(s, 100) // now full (len 4 = cap 4) → NEW array allocated
	s[0] = -1
	fmt.Println(arr, s) // [10 20 30 40 99] [-1 30 40 99 100]  ← arr untouched now
}
```

Real output:

```
[10 20 30 40 99] [20 30 40 99]
[10 20 30 40 99] [-1 30 40 99 100]
```

Read it carefully: the **first** `append` (within capacity) *overwrote `arr[4]`* (50 → 99). The **second** exceeded capacity, so the slice moved to a brand-new array and further writes don't touch `arr`. This "sometimes shared, sometimes not" behavior is why you should treat the slice you appended to as **the only valid one**: always write `s = append(s, ...)`.

### Gotcha 3: appending to a sub-slice overwrites its neighbor

```go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4, 5}
	b := a[:2] // [1 2] with cap 5!

	b = append(b, 100) // fits in cap → overwrites a[2]
	fmt.Println(a, b)  // [1 2 100 4 5] [1 2 100]
}
```

The sub-slice's spare capacity is the *parent's* elements. Fix with a **full slice expression** that caps the capacity (section 17): `b := a[:2:2]`.

---

## 11. Slices and functions

Passing a slice copies the **header** (24 bytes), so the function gets its **own pointer/len/cap**, but pointing to the **same underlying array**.

### Modifying elements: visible to the caller ✅

```go
package main

import "fmt"

func modify(s []int) {
	s[0] = 100 // writes to the shared array
}

func main() {
	a := []int{1, 2, 3}
	modify(a)
	fmt.Println(a) // [100 2 3]
}
```

### Appending inside a function: caller's length does NOT change ❌

```go
package main

import "fmt"

func appendIt(s []int) {
	s = append(s, 4) // modifies the function's OWN header copy
	s[0] = 999
}

func main() {
	a := []int{1, 2, 3} // len 3, cap 3 (full)
	appendIt(a)
	fmt.Println(a, len(a), cap(a)) // [1 2 3] 3 3
}
```

Here `append` hit capacity, so it made a **new** array; `s[0] = 999` changed the new array; `a` still points to the old array. So `a` prints `[1 2 3]`.

But watch what happens when there *is* spare capacity:

```go
package main

import "fmt"

func appendIt(s []int) {
	s = append(s, 4)
	s[0] = 999
}

func main() {
	b := make([]int, 3, 10) // len 3, cap 10
	appendIt(b)
	fmt.Println(b)     // [999 0 0]  (b's length is still 3; but element 0 CHANGED)
	fmt.Println(b[:4]) // [999 0 0 4]  (the 4 was written into the shared array's spare slot!)
}
```

Real output: `[999 0 0]` and `[999 0 0 4]`. The function wrote into the shared array (visible via `b[:4]`), yet `b`'s **length** never changed.

### The rule

> A function can change a slice's **elements** for the caller, but it can **not** change the caller's **length** (or capacity or pointer), because those live in the header, and the header is copied.

To let a function grow the caller's slice you either **return** the new slice (idiomatic):

```go
func addItem(s []int, v int) []int { return append(s, v) }

items = addItem(items, 42)
```

or pass a **pointer to the slice** (`*[]int`), which is rarer.

(This is exactly the same "arguments are copies" story from Chapter 4. The *thing being copied* is just a small header that happens to point to shared data.)

---

## 12. `copy`

`copy(dst, src)` copies elements and returns how many were copied: `min(len(dst), len(src))`. Use it to make an **independent** duplicate:

```go
package main

import "fmt"

func main() {
	src := []int{1, 2, 3}

	dst := make([]int, len(src)) // must have length! copy doesn't grow dst
	n := copy(dst, src)
	fmt.Println(dst, n) // [1 2 3] 3

	dst[0] = 99
	fmt.Println(src, dst) // [1 2 3] [99 2 3]: independent!

	short := make([]int, 2)
	copy(short, src)
	fmt.Println(short) // [1 2]: only 2 fit
}
```

Common idioms for cloning:

```go
clone := append([]int(nil), src...) // classic
clone := slices.Clone(src)          // Go 1.21+, clearest
```

Contrast with assignment: `b := a` copies only the header, so both **share** data.

---

## 13. Everyday recipes

### Remove the element at index `i` (keeping order)

```go
s = append(s[:i], s[i+1:]...)
// or, Go 1.21+:
s = slices.Delete(s, i, i+1)
```

```go
package main

import "fmt"

func main() {
	s := []int{10, 20, 30, 40, 50}
	i := 2
	s = append(s[:i], s[i+1:]...)
	fmt.Println(s) // [10 20 40 50]
}
```

(This *shifts* the tail left in the same array, which changes what other slices sharing the array see.)

### Remove without keeping order (faster, O(1))

```go
s[i] = s[len(s)-1]
s = s[:len(s)-1]
```

### Insert `v` at index `i`

```go
s = append(s[:i], append([]int{v}, s[i:]...)...)
// or, Go 1.21+:
s = slices.Insert(s, i, v)
```

### Filter in place

```go
package main

import "fmt"

func main() {
	nums := []int{1, 2, 3, 4, 5, 6}
	evens := nums[:0] // zero-length slice sharing nums' array
	for _, n := range nums {
		if n%2 == 0 {
			evens = append(evens, n)
		}
	}
	fmt.Println(evens) // [2 4 6]
}
```

(`nums` itself is scrambled afterward; use a fresh `make` if you still need it.)

### Reverse

```go
for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
	s[i], s[j] = s[j], s[i]
}
```

### Stack and queue

```go
stack = append(stack, x)             // push
top := stack[len(stack)-1]           // peek
stack = stack[:len(stack)-1]         // pop

queue = append(queue, x)             // enqueue
front := queue[0]; queue = queue[1:] // dequeue (see leak note in section 17)
```

### Clear all elements

```go
clear(s) // Go 1.21+: sets every element to its zero value (keeps length)
s = s[:0] // or truncate to length 0 (keeps capacity)
```

---

## 14. `nil` slice vs. empty slice

```go
package main

import "fmt"

func main() {
	var n []int  // nil slice
	e := []int{} // empty, non-nil slice

	fmt.Println(n == nil, e == nil) // true false
	fmt.Println(len(n), len(e))     // 0 0
	fmt.Println(n, e)               // [] []

	n = append(n, 1) // ✅ appending to nil works
	fmt.Println(n)   // [1]
}
```

- A **nil** slice has `ptr == nil, len == 0, cap == 0`. It's the **zero value**.
- An **empty** slice (`[]int{}` or `make([]int, 0)`) has a non-nil pointer to a zero-size array.
- **They behave identically** with `len`, `cap`, `range`, `append`. You almost never need to distinguish them.
- One place they differ: **JSON**: a nil slice encodes as `null`, an empty slice as `[]`. That matters in APIs (Chapters 40+).
- Idiom: to test "is it empty?", use `len(s) == 0`, **not** `s == nil`.

---

## 15. Iterating with `range`

```go
package main

import "fmt"

func main() {
	fruits := []string{"apple", "banana", "cherry"}

	for i, f := range fruits {
		fmt.Println(i, f)
	}
	for _, f := range fruits { _ = f } // value only
	for i := range fruits { _ = i }    // index only
}
```

**Two important facts:**

1. The loop variable `f` is a **copy** of each element. Modifying `f` doesn't change the slice:

```go
nums := []int{1, 2, 3}
for _, n := range nums {
	n *= 2 // changes the copy only
}
fmt.Println(nums) // [1 2 3]

for i := range nums {
	nums[i] *= 2 // ✅ modify through the index
}
fmt.Println(nums) // [2 4 6]
```

2. `range` evaluates the slice **once**, before the loop. Appending to the slice inside the loop doesn't extend the iteration (it will iterate over the original length).

For slices of large structs, prefer `for i := range items { item := &items[i] ... }` to avoid copying each struct.

---

## 16. Two-dimensional slices

A "matrix" is a slice of slices. **Each row is its own slice**, and rows can have different lengths:

```go
package main

import "fmt"

func main() {
	rows, cols := 3, 4

	grid := make([][]int, rows) // 3 nil rows
	for r := range grid {
		grid[r] = make([]int, cols) // allocate each row
	}

	grid[1][2] = 7
	for _, row := range grid {
		fmt.Println(row)
	}
}
```

Output:

```
[0 0 0 0]
[0 0 7 0]
[0 0 0 0]
```

Literal form (a "jagged" triangle):

```go
tri := [][]int{
	{1},
	{1, 1},
	{1, 2, 1},
}
```

⚠️ `make([][]int, 3)` gives 3 **nil** rows: you must `make` each row before indexing (`grid[0][0] = 1` panics otherwise).

---

## 17. Memory leaks and the three-index slice

### Keeping a huge array alive by holding a tiny slice

```go
func firstLine(data []byte) []byte {
	i := bytes.IndexByte(data, '\n')
	return data[:i] // tiny slice, but it POINTS INTO the whole `data` array!
}
```

If `data` is a 100 MB file and you keep the returned 20-byte slice, **the whole 100 MB stays alive** (the GC can't free part of an array). Fix: **copy** what you need:

```go
return append([]byte(nil), data[:i]...) // independent 20-byte copy
```

Similarly, a queue implemented as `queue = queue[1:]` never releases the front elements' memory until the array is reallocated. Also, if elements are pointers, set the dequeued slot to `nil`/zero first (`queue[0] = nil`) so the GC can reclaim them.

### The three-index (full) slice expression: `a[low:high:max]`

It limits the **capacity** to `max - low`, so `append` can't overwrite the parent's elements:

```go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4, 5}

	b := a[:2:2] // len 2, cap 2 (capped)
	b = append(b, 100) // must reallocate: cannot touch a[2]

	fmt.Println(a) // [1 2 3 4 5]  ✓ untouched
	fmt.Println(b) // [1 2 100]
}
```

Use it whenever you hand out a sub-slice and want to prevent it from clobbering the rest.

---

## 18. The `slices` package

Since Go 1.21 the standard library has generic helpers (`import "slices"`):

```go
package main

import (
	"fmt"
	"slices"
)

func main() {
	s := []int{5, 2, 8, 1}

	slices.Sort(s)
	fmt.Println(s) // [1 2 5 8]

	fmt.Println(slices.Contains(s, 5))     // true
	fmt.Println(slices.Index(s, 8))        // 3
	fmt.Println(slices.Max(s), slices.Min(s)) // 8 1

	i, found := slices.BinarySearch(s, 5) // s must be sorted
	fmt.Println(i, found)                 // 2 true

	c := slices.Clone(s)
	slices.Reverse(c)
	fmt.Println(c, slices.Equal(s, c)) // [8 5 2 1] false

	s = slices.Insert(s, 1, 100)
	fmt.Println(s) // [1 100 2 5 8]
	s = slices.Delete(s, 1, 2)
	fmt.Println(s) // [1 2 5 8]

	s = slices.Compact([]int{1, 1, 2, 2, 2, 3}) // remove consecutive duplicates
	fmt.Println(s) // [1 2 3]
}
```

Use these instead of hand-rolled loops for clarity. Understanding the raw slice mechanics (this chapter) tells you *what they do and cost*.

---

## 19. Interview questions

**Q1. What is a slice, internally?**
A header of three fields: pointer to an element in an underlying array, length, and capacity (24 bytes on 64-bit).

**Q2. Slice vs. array?**
Arrays have fixed length as part of their type and are values (copied on assignment). Slices are dynamic views over arrays; copying a slice copies only the header.

**Q3. What does this print?**

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]
b = append(b, 100)
fmt.Println(a, b)
```
`b = a[1:3]` is `[2 3]` with len 2 and cap 4, so there is room. The append writes at position 2 of `b`, which is `a[3]`. Result: `a = [1 2 3 100 5]`, `b = [2 3 100]`.

**Q4. What does this print?**

```go
s := make([]int, 3, 10)
s = append(s, 1)
fmt.Println(len(s), cap(s))
```
`4 10`.

**Q5. What does this print?**

```go
s := []int{1, 2, 3}
t := s
t[0] = 99
t = append(t, 4)
t[1] = 88
fmt.Println(s, t)
```
`t` shares `s`'s array, so `t[0] = 99` changes both → `s = [99 2 3]`. `append(t, 4)`: cap is 3 (full) → new array → `t[1] = 88` only affects the new array. Result: `s = [99 2 3]`, `t = [99 88 3 4]`.

**Q6. Is `var s []int` equal to `[]int{}`?**
Both have len 0 and behave the same in most operations; the first is `nil`, the second isn't. They differ for `== nil` and JSON encoding.

**Q7. How does `append` decide to reallocate?**
If `len + newElements > cap`, it allocates a larger array, copies the elements, and returns a slice pointing to the new array.

**Q8. Why must you write `s = append(s, x)`?**
`append` returns a new header (new length, maybe new pointer); the original variable's header isn't modified.

**Q9. How do you copy a slice so changes don't affect the original?**
`copy` into a `make`d slice, `slices.Clone`, or `append([]T(nil), s...)`.

**Q10. What's `s[low:high:max]`?**
The full slice expression: `len = high−low`, `cap = max−low`, preventing appends from overwriting the parent array's later elements.

---

## 20. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | `append(s, x)` without using the result | Nothing changes | `s = append(s, x)` |
| 2 | `make([]int, 10)` then `append` expecting length 1 | Slice has 10 zeros then your item | `make([]int, 0, 10)` |
| 3 | Indexing beyond `len` (but within `cap`) | `index out of range` panic | Append, or reslice `s[:n]` |
| 4 | Assuming `b := a` copies data | Modifying `b` changes `a` | `copy` / `slices.Clone` |
| 5 | Appending inside a function and expecting the caller's length to change | Caller unchanged | Return the slice |
| 6 | Modifying the range value (`for _, v := range s { v++ }`) | Slice unchanged | Use `s[i]++` |
| 7 | Sub-slice `append` clobbering the parent | Parent elements change | Use `a[:n:n]` |
| 8 | Holding a tiny sub-slice of a giant array | Memory not freed | Copy what you need |
| 9 | Deleting inside a `range` loop by index | Skipped elements | Loop backwards, or filter into a new slice |
| 10 | Assuming a specific `cap` after `append` | Tests break on other Go versions | Never rely on exact capacity |
| 11 | Using `==` to compare slices | `invalid operation: slice can only be compared to nil` | `slices.Equal` |
| 12 | Forgetting to `make` each row of a 2D slice | `index out of range` | Allocate rows in a loop |

---

## 21. Exercises

### Exercise 1: Create and inspect
Create `s := []int{10, 20, 30, 40, 50}` and print `s[1:3]`, `s[:2]`, `s[3:]`, along with the length and capacity of `s[1:3]`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	s := []int{10, 20, 30, 40, 50}
	fmt.Println(s[1:3], s[:2], s[3:]) // [20 30] [10 20] [40 50]
	t := s[1:3]
	fmt.Println(len(t), cap(t)) // 2 4
}
```
</details>

### Exercise 2: Predict len and cap

```go
a := make([]int, 2, 6)
b := a[1:4]
c := b[2:]
fmt.Println(len(a), cap(a), len(b), cap(b), len(c), cap(c))
```

<details><summary>Solution</summary>

`a`: len 2, cap 6. `b = a[1:4]`: len 3, cap 5. `c = b[2:]`: len 1, cap 3.
Output: `2 6 3 5 1 3`. (Slicing up to `high=4` is allowed even though `len(a)=2` because `4 <= cap(a)`.)
</details>

### Exercise 3: Shared or not?

```go
x := []int{1, 2, 3}
y := x
y[0] = 100
z := append(x, 4)
z[1] = 200
fmt.Println(x, y, z)
```

<details><summary>Solution</summary>

`y` shares `x`'s array: `x = y = [100 2 3]`. `append(x, 4)` exceeds cap 3 → `z` is a new array `[100 2 3 4]`; `z[1] = 200` changes only `z`.
Output: `[100 2 3] [100 2 3] [100 200 3 4]`.
</details>

### Exercise 4: Remove and insert
From `[]string{"a","b","c","d"}`, remove `"b"` and insert `"X"` at index 1. Do it by hand and with the `slices` package.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"slices"
)

func main() {
	s := []string{"a", "b", "c", "d"}
	s = append(s[:1], s[2:]...)         // remove "b" → [a c d]
	s = append(s[:1], append([]string{"X"}, s[1:]...)...) // insert at 1 → [a X c d]
	fmt.Println(s)

	t := []string{"a", "b", "c", "d"}
	t = slices.Delete(t, 1, 2)
	t = slices.Insert(t, 1, "X")
	fmt.Println(t) // [a X c d]
}
```
</details>

### Exercise 5: Write `unique`
Write `unique(nums []int) []int` returning the elements in first-seen order without duplicates. Don't modify the input.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func unique(nums []int) []int {
	seen := make(map[int]bool)
	result := make([]int, 0, len(nums))
	for _, n := range nums {
		if !seen[n] {
			seen[n] = true
			result = append(result, n)
		}
	}
	return result
}

func main() {
	fmt.Println(unique([]int{3, 1, 3, 2, 1, 4})) // [3 1 2 4]
}
```
</details>

### Exercise 6: Grow through a function
Write `push(s []int, v int)` that adds `v` to the caller's slice. Two versions: one that doesn't work, and one that does.

<details><summary>Solution</summary>

```go
package main

import "fmt"

// ❌ doesn't work: the caller's header never changes
func pushBroken(s []int, v int) { s = append(s, v) }

// ✅ return the new slice
func push(s []int, v int) []int { return append(s, v) }

// ✅ or take a pointer to the slice
func pushPtr(s *[]int, v int) { *s = append(*s, v) }

func main() {
	a := []int{1, 2}
	pushBroken(a, 3)
	fmt.Println(a) // [1 2]

	a = push(a, 3)
	fmt.Println(a) // [1 2 3]

	pushPtr(&a, 4)
	fmt.Println(a) // [1 2 3 4]
}
```
</details>

### Exercise 7: Chunk
Write `chunk(nums []int, size int) [][]int` splitting a slice into pieces of at most `size`. Make sure the chunks can't clobber each other via `append`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func chunk(nums []int, size int) [][]int {
	var out [][]int
	for size < len(nums) {
		// three-index slice caps capacity so appends can't overwrite neighbours
		nums, out = nums[size:], append(out, nums[:size:size])
	}
	return append(out, nums)
}

func main() {
	fmt.Println(chunk([]int{1, 2, 3, 4, 5, 6, 7}, 3)) // [[1 2 3] [4 5 6] [7]]
}
```
</details>

### Exercise 8 (challenge): Explain this output

```go
package main

import "fmt"

func main() {
	s := make([]int, 0, 3)
	a := append(s, 1)
	b := append(s, 2)
	fmt.Println(a, b)
}
```

<details><summary>Solution</summary>

Output: `[2] [2]`. Both `append` calls have room (cap 3) and both start from the *same* empty `s`, so they write into the **same** array's first slot: `a` sees `1`, then `b`'s append overwrites it with `2`. Both slices point at the same array with len 1. It's the classic proof that you must not use the *old* slice after appending, or derive several slices from one shared `s`.
</details>

---

## 22. Quiz

1. What three things does a slice header hold?
2. What's the capacity of `arr[1:4]` if `arr` is `[5]int`?
3. What does `append` do when `len == cap`?
4. Does passing a slice to a function copy its elements?
5. What's the difference between `make([]int, 5)` and `make([]int, 0, 5)`?
6. Why is `s = append(s, x)` required?
7. What does `a[low:high:max]` control?

<details><summary>Answers</summary>

1. Pointer, length, capacity.
2. `4` (from index 1 to the end of the 5-element array).
3. Allocates a bigger array, copies elements, appends, returns a slice pointing at the new array.
4. No, only the 24-byte header is copied; the elements are shared.
5. The first has length 5 (five zeros); the second has length 0 and reserves room for 5.
6. `append` returns a new header (new length / maybe new pointer); it can't modify your variable.
7. The capacity of the resulting slice (`max - low`), limiting how far `append` can extend in place.
</details>

---

## 23. Summary

- A **slice** = **header** (`ptr`, `len`, `cap`) + a shared **underlying array**. It's a *view*, not a container.
- Create with a **literal** `[]T{...}` (most common), **`make`**, by **slicing** an array/slice, or by **appending** to `nil`.
- **len** = elements usable now; **cap** = room to grow before reallocation. Only `0 … len-1` can be indexed.
- **`append`** writes into spare capacity (same array) or, when full, **allocates a bigger array and copies**. Always use `s = append(s, …)`; never rely on exact capacities.
- Slices **share memory**: writes through one show up in others, and appends within capacity can overwrite neighbors. Use **`copy`**/`slices.Clone` for independence and **`a[l:h:m]`** to cap capacity.
- Passing a slice copies the **header**: callee can change **elements**, not the caller's **length**. **Return** the new slice.
- `nil` and empty slices behave alike (`len 0`, `append` works). Prefer `len(s) == 0`.
- Watch for **memory retention** by tiny sub-slices of huge arrays.
- Use the `slices` package for sort/search/insert/delete/clone.

### ➡️ What's next?

**Part 5** switches gears from language syntax to **how computers work**, knowledge that makes Go's concurrency model make sense. [Chapter 26](26-computer-architecture-and-a-short-history-of-computing.md) starts with computer architecture and history.
