# Chapter 24: Pointers — Understanding Memory Addresses

> **Goal of this chapter:** Demystify pointers. A **pointer** is a variable that stores another variable's **memory address**. With pointers, functions can modify the caller's data, big values can be shared without copying, and `nil` can mean "nothing here". By the end, `&` and `*` will feel natural.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 2–2.5 hours  **Prerequisite:** [Chapters 18, 21–23](18-go-internal-memory.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why pointers exist](#2-why-pointers-exist)
3. [The home-address analogy](#3-the-home-address-analogy)
4. [Memory addresses in Go](#4-memory-addresses)
5. [The `&` operator: address-of](#5-the--operator-address-of)
6. [The `*` operator: two meanings](#6-the--operator-two-meanings)
7. [Full memory simulation](#7-full-memory-simulation)
8. [Pointers and functions](#8-pointers-and-functions)
9. [Pointers to structs](#9-pointers-to-structs)
10. [`nil` pointers](#10-nil-pointers)
11. [`new` and returning pointers](#11-new-and-returning-pointers)
12. [Pointer to pointer](#12-pointer-to-pointer)
13. [Reading addresses: hexadecimal](#13-reading-addresses-hexadecimal)
14. [What Go does NOT allow](#14-what-go-does-not-allow)
15. [When to use pointers (and when not)](#15-when-to-use-pointers)
16. [Common mistakes](#16-common-mistakes)
17. [Exercises](#17-exercises)
18. [Quiz](#18-quiz)
19. [Summary](#19-summary)

---

## 1. What you will learn

- What a **pointer** is, in plain words
- How to take an address (`&x`) and follow it (`*p`)
- How pointers let functions modify their arguments
- Why `p.Field` works on pointers to structs
- What `nil` pointers are, and how to avoid crashing on them
- What Go deliberately *removes* (no pointer arithmetic)
- When pointers are the right tool

> 😰 **Many people fear pointers.** They come from C, where pointers can crash or corrupt programs. Go's pointers are **safe and simple**: no arithmetic, garbage collected, and the compiler checks types. If you understood variables living at addresses (Chapters 2 and 18), you already understand pointers.

---

## 2. Why pointers exist

Recall Chapter 4: **arguments are copied**.

```go
package main

import "fmt"

func addTen(n int) {
	n = n + 10 // changes the COPY
}

func main() {
	x := 5
	addTen(x)
	fmt.Println(x) // 5 (not 15!)
}
```

What if we *want* `addTen` to change `x`? We must give the function **`x` itself**, not a copy. Since `x` lives at some address in memory, we can pass the **address**, and the function can walk to that address and change what's there.

Pointers solve three big problems:

| Problem | Pointer solution |
|---------|------------------|
| Functions can't modify caller's variables | Pass the address |
| Copying large structs/arrays is slow | Pass 8 bytes (the address) instead |
| Need to represent "no value" | `nil` pointer |

They also underpin: methods that modify receivers (Chapter 22), linked lists/trees, sharing data between goroutines, and nearly every "reference" in Go.

---

## 3. The home-address analogy

> Your **house** contains your family and belongings (the **value**). Your house has an **address**: "12 Green Road, Dhaka" (the **pointer**).
>
> If a friend wants to visit you, you don't copy your whole house for them: you **give them your address**. They go to that address and see (or rearrange!) the real furniture.

| Analogy | Go |
|---------|-----|
| House | A variable in memory (e.g., `x`) |
| Furniture inside | The variable's value (`5`) |
| Street address | The variable's memory address |
| A piece of paper with the address written on it | A **pointer variable** (`p`) |
| Writing down someone's address | `&x` ("address of x") |
| Going to the address and looking inside | `*p` ("value at p") |
| A paper with no address written (blank) | `nil` |

**Key insight:** the paper with the address is *not the house*. It's a separate small thing (a pointer) that *refers* to the house. You can photocopy the paper cheaply; you can't cheaply photocopy the house.

---

## 4. Memory addresses

Every variable lives at an **address**: a number identifying its location in RAM. You can see it:

```go
package main

import "fmt"

func main() {
	x := 42
	fmt.Println(x)  // 42        (the value)
	fmt.Println(&x) // 0xc000012345  (the address; differs each run)
}
```

Typical output:

```
42
0xc00001c0a8
```

Addresses are printed in **hexadecimal** (base 16), which is why they start with `0x` and contain letters. (Section 13 explains.) The exact number changes from run to run, and that's normal.

---

## 5. The `&` operator: address-of

`&x` means "**the address of x**". Its type is `*T` if `x` is of type `T`.

```go
package main

import "fmt"

func main() {
	a := 10
	p := &a // p holds the address of a

	fmt.Println(a)          // 10
	fmt.Println(p)          // 0xc0000... (an address)
	fmt.Printf("%T %T\n", a, p) // int *int
}
```

**Reading the type:** `*int` = "pointer to int". A pointer variable's type includes what it points to.

**Declaring a pointer type:**

```go
var p *int      // p is a pointer to an int; nil (points at nothing) for now
var q *string   // pointer to a string
var r *User     // pointer to a User struct
```

---

## 6. The `*` operator: two meanings

The `*` symbol does **two different jobs** depending on where it appears. This is the most common source of confusion, so separate them clearly:

### Meaning 1: in a **type**: "pointer to"

```go
var p *int        // "p is a pointer to an int"
func f(p *User)   // "p is a pointer to a User"
```

### Meaning 2: in an **expression**: "dereference" ("the value at")

```go
fmt.Println(*p)   // "the int that p points to"
*p = 99           // "store 99 in the int that p points to"
```

```go
package main

import "fmt"

func main() {
	a := 10
	p := &a // & : get address        (p's TYPE is *int)

	fmt.Println(*p) // * : follow the address → 10

	*p = 20         // write THROUGH the pointer
	fmt.Println(a)  // 20 ← a itself changed!

	a = 30
	fmt.Println(*p) // 30 ← p sees the change: same location
}
```

Output:

```
10
20
30
```

**Cheat sheet:**

| Symbol | Where | Reads as | Example |
|--------|-------|----------|---------|
| `&x` | expression | "address of x" | `p := &x` |
| `*T` | type | "pointer to T" | `var p *int` |
| `*p` | expression | "value at p" (dereference) | `y := *p`, `*p = 5` |

`&` and `*` are opposites: `*(&x)` is just `x`.

---

## 7. Full memory simulation

```go
package main

import "fmt"

func main() {
	a := 10
	p := &a
	*p = 20
	fmt.Println(a, *p)
}
```

**Step 1: `a := 10`.** `main`'s frame gets a slot for `a`. Say it's at address `0xA0`:

```
STACK (main's frame)
 address   variable   value
 0xA0      a          10
```

**Step 2: `p := &a`.** A new slot for `p`, at `0xA8`. It stores the *address* `0xA0`:

```
 address   variable   value
 0xA0      a          10
 0xA8      p          0xA0  ──┐
                             │  (points to a)
 0xA0 ◄──────────────────────┘
```

Picture:

```
   p                 a
┌────────┐        ┌──────┐
│ 0xA0   │ ─────► │  10  │
└────────┘        └──────┘
  0xA8              0xA0
```

**Step 3: `*p = 20`.** Follow `p` to address `0xA0`; write `20` there:

```
   p                 a
┌────────┐        ┌──────┐
│ 0xA0   │ ─────► │  20  │  ← changed through the pointer
└────────┘        └──────┘
```

**Step 4: `fmt.Println(a, *p)`.** `a` is `20`; `*p` follows `p` to `0xA0` and reads `20`. Prints `20 20`.

A pointer is just **a variable whose value is an address**. That's all.

Sizes: on a 64-bit machine every pointer is **8 bytes**, no matter what it points to (an `int` or a 1 MB struct).

---

## 8. Pointers and functions

### 8.1 Modifying the caller's variable

```go
package main

import "fmt"

func addTen(n *int) { // takes a POINTER to an int
	*n = *n + 10 // read through it, write through it
}

func main() {
	x := 5
	addTen(&x) // pass the ADDRESS of x
	fmt.Println(x) // 15 ✓
}
```

Memory during the call:

```
┌─────────────────────────────┐
│ addTen                      │
│   n = 0xA0  ────────────┐   │   n is a COPY of the address (still pointing at x)
├─────────────────────────┼───┤
│ main                    ▼   │
│   x = 5 → 15   ◄── modified │
└─────────────────────────────┘
```

Go still passes by value: it copies the *pointer*. But copies of an address still lead to the same house.

### 8.2 The classic swap

```go
package main

import "fmt"

func swap(a, b *int) {
	*a, *b = *b, *a
}

func main() {
	x, y := 1, 2
	swap(&x, &y)
	fmt.Println(x, y) // 2 1
}
```

(In Go you'd usually write `x, y = y, x` directly, but this is the standard pointer demo.)

### 8.3 Scanf revisited

Remember `fmt.Scanln(&name)` (Chapter 7)? That's why it needs `&`: `Scanln` receives the address and writes the typed value **through the pointer** into your variable.

### 8.4 Avoiding big copies

```go
package main

import "fmt"

type BigData struct {
	Values [100000]int // 800 KB!
}

func sumByValue(d BigData) int { // copies 800 KB on every call
	total := 0
	for _, v := range d.Values {
		total += v
	}
	return total
}

func sumByPointer(d *BigData) int { // copies 8 bytes
	total := 0
	for _, v := range d.Values {
		total += v
	}
	return total
}

func main() {
	d := &BigData{}
	fmt.Println(sumByValue(*d), sumByPointer(d))
}
```

---

## 9. Pointers to structs

The most common use of pointers is with structs.

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func birthday(u *User) {
	u.Age++ // shorthand for (*u).Age++
}

func main() {
	user := User{"Asha", 30}
	birthday(&user)
	fmt.Println(user.Age) // 31

	p := &user
	fmt.Println(p.Name)    // Asha: Go auto-dereferences: same as (*p).Name
	p.Name = "Asha Rahman" // modifies user
	fmt.Println(user.Name) // Asha Rahman
}
```

**Auto-dereference:** writing `(*p).Name` is ugly, so Go lets you write `p.Name`. The compiler inserts the `*` for you. No `->` operator like in C.

This is also why **pointer receivers** work (Chapter 22): `func (u *User) Birthday()` gets the address, and `u.Age++` modifies the original.

Creating structs directly as pointers is idiomatic:

```go
u := &User{Name: "Bina", Age: 25} // *User
```

---

## 10. `nil` pointers

A pointer that doesn't point at anything holds the special value **`nil`**. It's the zero value of every pointer type.

```go
package main

import "fmt"

func main() {
	var p *int
	fmt.Println(p)        // <nil>
	fmt.Println(p == nil) // true
}
```

### Dereferencing `nil` crashes

```go
package main

import "fmt"

func main() {
	var p *int
	defer func() { fmt.Println("recovered:", recover()) }()
	fmt.Println(*p) // ❌ panic: runtime error: invalid memory address or nil pointer dereference
}
```

Output:

```
recovered: runtime error: invalid memory address or nil pointer dereference
```

That's Go's most common runtime panic. **Always check** pointers that might be `nil`:

```go
package main

import "fmt"

type User struct{ Name string }

func greet(u *User) {
	if u == nil {
		fmt.Println("no user")
		return
	}
	fmt.Println("Hello,", u.Name)
}

func main() {
	greet(nil)
	greet(&User{"Asha"})
}
```

### `nil` as "no value"

Pointers let you represent *optional* values: `*int` can be "absent" (nil) or a number. E.g., a JSON field that may be missing, or a database column that may be `NULL`. (Chapters 52–56.)

```go
var age *int         // unknown age
years := 30
age = &years         // now known
```

Also: `nil` is the zero value for **slices, maps, channels, functions, interfaces, and pointers**.

---

## 11. `new` and returning pointers

### `new(T)`

Allocates a zeroed `T` and returns a `*T`:

```go
p := new(int)   // *int pointing at a fresh 0
*p = 7
fmt.Println(*p) // 7
```

Equivalent to `var x int; p := &x`. In practice `&T{...}` for structs is far more common than `new`.

### Returning the address of a local is SAFE in Go

```go
package main

import "fmt"

type User struct{ Name string }

func newUser(name string) *User {
	u := User{Name: name}
	return &u // ✅ safe!
}

func main() {
	u := newUser("Asha")
	fmt.Println(u.Name)
}
```

In C this would be a serious bug (returning a pointer to a dead stack frame). In Go, **escape analysis** (Chapter 18) sees the address escapes and puts `u` on the **heap**, so it lives as long as anything points to it. You can confirm: `go build -gcflags=-m` prints `moved to heap: u`.

---

## 12. Pointer to pointer

A pointer is a variable, so it has an address too. You can point at it:

```go
package main

import "fmt"

func main() {
	a := 10
	p := &a  // *int
	pp := &p // **int: pointer to a pointer

	fmt.Println(a, *p, **pp) // 10 10 10

	**pp = 99
	fmt.Println(a) // 99
}
```

```
   pp               p                a
┌────────┐       ┌────────┐       ┌──────┐
│ &p     │ ────► │ &a     │ ────► │  99  │
└────────┘       └────────┘       └──────┘
  **int            *int             int
```

You'll rarely need `**T` in everyday Go; it appears in a few APIs that need to *replace* a pointer (like `json.Unmarshal` into a `*T` field). It's included so `**` doesn't scare you.

---

## 13. Reading addresses: hexadecimal

Addresses print like `0xc00001c0a8`. Why hex?

- Computers use **binary**; humans find long binary unreadable.
- **Hexadecimal (base 16)** uses digits `0–9` and `a–f` (`a=10 … f=15`). One hex digit = exactly 4 bits, so hex is a compact way to write binary.
- `0x` is the prefix that says "this number is hex".

Converting `0x1f` to decimal: `1×16 + 15 = 31`.

```go
package main

import "fmt"

func main() {
	fmt.Printf("%d %x %X %b %o\n", 255, 255, 255, 255, 255)
	// 255 ff FF 11111111 377
}
```

You never need to convert addresses by hand. Treat them as opaque IDs. Use `%p` to print a pointer explicitly:

```go
x := 1
fmt.Printf("%p\n", &x)
```

---

## 14. What Go does NOT allow

Go's pointers are **deliberately restricted**:

| C allows | Go |
|----------|-----|
| **Pointer arithmetic** (`p + 1`, `p++`) | ❌ Not allowed (without the `unsafe` package) |
| Casting any pointer to any other | ❌ Types are checked |
| Uninitialized pointers holding garbage | ❌ Pointers start at `nil` |
| `free`/dangling pointers to dead stack frames | ❌ GC + escape analysis prevent it |
| Pointers to arbitrary memory | ❌ Only to variables/fields/elements |

That's why Go pointers are safe. To move through an array you use **indexes** and **slices**, not pointer math.

(There is a package literally called `unsafe` for the rare cases that need C-like tricks. Avoid it as a beginner.)

**Comparing pointers:** `p == q` is true if they hold the **same address** (point at the same variable), not if the values are equal:

```go
a, b := 5, 5
pa, pb := &a, &b
fmt.Println(pa == pb, *pa == *pb) // false true
```

---

## 15. When to use pointers

**Use a pointer when:**

- ✅ A function/method **must modify** its argument/receiver.
- ✅ The value is **large** and copying is wasteful.
- ✅ You need a **"no value"** (`nil`) state.
- ✅ Values must be **shared** between parts of the program (one instance, many users).
- ✅ The type must not be copied (contains a `sync.Mutex`).
- ✅ You're implementing linked structures (lists, trees, graphs).

**Prefer values when:**

- ✅ The data is small (numbers, small structs like `Point`).
- ✅ You want **immutability-by-copy** so callers can't be surprised by changes.
- ✅ You'd otherwise use a pointer "just in case". Unneeded pointers cost extra heap allocation and GC work, and make code harder to reason about.

**Do not use pointers to:**
- ❌ `*string`/`*int` just to "save memory" for small things. It usually costs more.
- ❌ Slices, maps, or channels for mutation. They already hold references internally (Chapter 25 explains slices).

---

## 16. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Dereferencing `nil` | `nil pointer dereference` panic | Check `p != nil` |
| 2 | Forgetting `&` when a function wants a pointer | `cannot use x (variable of type int) as *int value` | Pass `&x` |
| 3 | Forgetting `*` to read the value | Prints an address instead of the value | `*p` |
| 4 | Confusing `*` in types vs. `*` as dereference | Syntax confusion | Type position = "pointer to"; expression = "value at" |
| 5 | Assuming `p2 := p` copies the *value* | Both point at the same data | `v := *p` copies the value |
| 6 | Comparing pointers when you mean values | Wrong result | Compare `*a == *b` |
| 7 | Expecting pointer arithmetic (`p++`) | Compile error | Use slices/indexes |
| 8 | Pointer to a loop variable in Go ≤ 1.21 | All pointers refer to the same variable | Go 1.22+ fixes it; or copy `v := v` |
| 9 | Modifying through a pointer unexpectedly | "Spooky action at a distance" | Document; prefer values when no mutation is needed |
| 10 | Returning `&local` and worrying | It's safe in Go | Trust escape analysis |

Mistake #5 shows why sharing can surprise you:

```go
package main

import "fmt"

func main() {
	a := 1
	p := &a
	q := p // copies the ADDRESS, not the value

	*q = 100
	fmt.Println(a, *p, *q) // 100 100 100: all three views of one variable
}
```

---

## 17. Exercises

### Exercise 1: Basic
Declare `n := 7`, take a pointer `p` to it, change `n` to 70 through `p`, and print `n`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	n := 7
	p := &n
	*p = 70
	fmt.Println(n) // 70
}
```
</details>

### Exercise 2: Double it
Write `double(n *int)` so `x := 4; double(&x)` makes `x == 8`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

func double(n *int) { *n *= 2 }

func main() {
	x := 4
	double(&x)
	fmt.Println(x) // 8
}
```
</details>

### Exercise 3: Predict

```go
package main

import "fmt"

func main() {
	a, b := 1, 2
	p := &a
	q := &b
	p, q = q, p
	*p = 10
	fmt.Println(a, b, *p, *q)
}
```

<details><summary>Solution</summary>

After `p, q = q, p`, `p` points to `b` and `q` to `a`. `*p = 10` sets `b = 10`. Output: `1 10 10 1`.
</details>

### Exercise 4: Safe user lookup
Write `findUser(users []User, name string) *User` returning a pointer to the matching user or `nil`. In `main`, handle both cases.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func findUser(users []User, name string) *User {
	for i := range users {
		if users[i].Name == name {
			return &users[i] // pointer to the element itself, not a copy
		}
	}
	return nil
}

func main() {
	users := []User{{"Asha", 30}, {"Rahim", 25}}

	if u := findUser(users, "Rahim"); u != nil {
		u.Age++ // modifies the slice's element
	}
	fmt.Println(users) // [{Asha 30} {Rahim 26}]

	fmt.Println(findUser(users, "Zoya") == nil) // true
}
```
Note `for i := range users` and `&users[i]`: using `for _, u := range users { return &u }` would return a pointer to a *copy*.
</details>

### Exercise 5: Value vs pointer receiver
What does this print?

```go
package main

import "fmt"

type Counter struct{ n int }

func (c Counter) IncV()  { c.n++ }
func (c *Counter) IncP() { c.n++ }

func main() {
	c := Counter{}
	c.IncV()
	c.IncP()
	c.IncP()
	p := &c
	p.IncV()
	p.IncP()
	fmt.Println(c.n)
}
```

<details><summary>Solution</summary>

`3`. Only the three `IncP` calls modify the original (`IncV` calls work on copies, even via `p`).
</details>

### Exercise 6: Optional field
Create `type Profile struct { Name string; Age *int }`. Print "age unknown" when `Age` is `nil`, otherwise the age.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type Profile struct {
	Name string
	Age  *int
}

func describe(p Profile) {
	if p.Age == nil {
		fmt.Println(p.Name, "- age unknown")
		return
	}
	fmt.Println(p.Name, "-", *p.Age)
}

func main() {
	years := 30
	describe(Profile{Name: "Asha", Age: &years})
	describe(Profile{Name: "Rahim"})
}
```
</details>

### Exercise 7 (challenge): Linked list
Build a singly linked list with `type Node struct { Value int; Next *Node }`, add three nodes, and print them by walking the list.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type Node struct {
	Value int
	Next  *Node
}

func main() {
	third := &Node{Value: 3}
	second := &Node{Value: 2, Next: third}
	head := &Node{Value: 1, Next: second}

	for n := head; n != nil; n = n.Next {
		fmt.Print(n.Value, " ")
	}
	fmt.Println() // 1 2 3
}
```
The list ends where `Next` is `nil`. This structure is impossible without pointers, since a `Node` can't contain another `Node` by value (infinite size).
</details>

---

## 18. Quiz

1. What does `&x` produce?
2. What does `*p` do in an expression?
3. Why does a function receiving `*int` change the caller's variable?
4. What is the zero value of a pointer? What happens if you dereference it?
5. Does Go support pointer arithmetic?
6. How does `u.Name` work when `u` is a `*User`?
7. Is `return &localVar` safe in Go?

<details><summary>Answers</summary>

1. The memory address of `x`.
2. Dereferences: reads or writes the value the pointer points to.
3. It receives a *copy of the address*, which still points to the original variable.
4. `nil`; dereferencing it causes a run-time panic.
5. No (outside the `unsafe` package).
6. Go auto-dereferences: `u.Name` means `(*u).Name`.
7. Yes. Escape analysis moves the variable to the heap.
</details>

---

## 19. Summary

- A **pointer** is a variable holding a **memory address**. Type `*T` means "pointer to T".
- **`&x`** = address of `x`. **`*p`** = the value `p` points to (dereference). They're inverses.
- Arguments are always **copied**; passing a **pointer** copies the address, so the callee can change the original.
- `p.Field` **auto-dereferences** pointers to structs; pointer receivers rely on this.
- The zero value is **`nil`**. Dereferencing `nil` panics, so **check first**. `nil` is also Go's "no value".
- Go's pointers are **safe**: no arithmetic, GC-managed, and `&local` is fine thanks to **escape analysis**.
- Use pointers to **mutate**, to **avoid big copies**, to express **optional** values, and to **share** data; otherwise prefer values.

### ➡️ What's next?

You now have everything needed for Go's most important collection type. [Chapter 25](25-slices.md): **slices**, "the most important interview topic".
