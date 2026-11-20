# Chapter 22: Receiver Functions (Methods) — Adding Behavior to Types

> **Goal of this chapter:** Attach functions to your types. A **method** (Go calls it a *function with a receiver*) lets you write `rect.Area()` instead of `area(rect)`. You'll learn the syntax, what really happens in memory, value vs. pointer receivers, and how methods make types printable and ready for interfaces.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 2–2.5 hours  **Prerequisite:** [Chapter 21](21-structs.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why methods?](#2-why-methods)
3. [Syntax of a receiver function](#3-syntax-of-a-receiver-function)
4. [How the call works](#4-how-the-call-works)
5. [Memory simulation](#5-memory-simulation)
6. [A method is a function in disguise](#6-a-method-is-a-function-in-disguise)
7. [Value receivers vs. pointer receivers](#7-value-receivers-vs-pointer-receivers)
8. [Choosing between them](#8-choosing-between-them)
9. [Methods on any local type](#9-methods-on-any-local-type)
10. [Multiple methods and organization](#10-multiple-methods-and-organization)
11. [The `String()` method and `fmt`](#11-the-string-method-and-fmt)
12. [Method values and method expressions](#12-method-values-and-method-expressions)
13. [Methods and embedding](#13-methods-and-embedding)
14. [Methods are a stepping stone to interfaces](#14-methods-are-a-stepping-stone-to-interfaces)
15. [Common mistakes](#15-common-mistakes)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- What a **receiver function** (method) is and how to write one
- How Go binds behavior to a type (and what that means in memory)
- The crucial difference between **value** and **pointer** receivers
- How to make your types print nicely with `String()`
- Why methods matter: they're how types satisfy **interfaces** (Chapter 51)

---

## 2. Why methods?

Suppose we have a `Rectangle`:

```go
type Rectangle struct {
	Width, Height float64
}
```

With ordinary functions:

```go
func area(r Rectangle) float64 { return r.Width * r.Height }

// usage
fmt.Println(area(rect))
```

Works, but:
- The behavior lives *apart* from the data it belongs to. In a big program, where are all the functions that work on rectangles?
- Names collide: `area` for rectangles, `area` for circles, `area` for triangles → `rectangleArea`, `circleArea`, ...

With **methods**:

```go
func (r Rectangle) Area() float64 { return r.Width * r.Height }

// usage
fmt.Println(rect.Area())
```

Benefits:

| Benefit | Explanation |
|---------|-------------|
| **Cohesion** | Data and its behavior are declared together, findable together |
| **Readability** | `rect.Area()` reads like English: "the rectangle's area" |
| **Namespacing** | `Circle.Area` and `Rectangle.Area` coexist without prefixes |
| **Discoverability** | Type `rect.` in your editor and see everything you can do |
| **Interfaces** | Methods are what let different types be used interchangeably (Chapter 51) |

> Go is not class-based. There's no `class` block that holds methods; methods are declared **separately** from the type, anywhere in the same package. But they *feel* like object-oriented methods when called.

---

## 3. Syntax of a receiver function

```go
func (receiverName ReceiverType) MethodName(parameters) returnType {
	// body: can use receiverName like a parameter
}
```

Compare to an ordinary function: the only new part is the extra parenthesized group **before the function name**: the **receiver**.

```go
func (u User) PrintDetails() {
	fmt.Println("Name:", u.Name)
	fmt.Println("Age:", u.Age)
}
```

| Part | Example | Meaning |
|------|---------|---------|
| `func` | `func` | Keyword |
| **Receiver** | `(u User)` | The value the method is called *on*. `u` is its name inside the method; `User` is its type |
| Method name | `PrintDetails` | Called as `value.PrintDetails()` |
| Parameters | `()` | Ordinary parameters (the receiver is *extra*) |
| Body | `{ ... }` | Uses `u.Name`, etc. |

**Receiver naming convention:** one or two letters, the initial(s) of the type (`u` for `User`, `r` for `Rectangle`). Not `this` or `self`.

Complete example:

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func (u User) PrintDetails() {
	fmt.Println("Name:", u.Name)
	fmt.Println("Age:", u.Age)
}

func main() {
	user1 := User{Name: "Habib", Age: 30}
	user1.PrintDetails() // call the method ON user1
}
```

Output:

```
Name: Habib
Age: 30
```

---

## 4. How the call works

`user1.PrintDetails()`:

1. Go looks at the **type** of `user1` (`User`).
2. It finds the method `PrintDetails` **bound** to `User`.
3. It calls it, passing **`user1` as the receiver `u`**.

The dot before the name is not "field access"; it's "call the method belonging to this value's type."

```
user1  .  PrintDetails()
  │            │
  │            └── method of type User
  └── becomes the receiver `u` inside the method
```

Methods with parameters and results work just like normal:

```go
package main

import "fmt"

type Rectangle struct{ Width, Height float64 }

func (r Rectangle) Area() float64      { return r.Width * r.Height }
func (r Rectangle) Perimeter() float64 { return 2 * (r.Width + r.Height) }
func (r Rectangle) Scale(f float64) Rectangle {
	return Rectangle{r.Width * f, r.Height * f}
}

func main() {
	rect := Rectangle{Width: 4, Height: 3}
	fmt.Println(rect.Area())      // 12
	fmt.Println(rect.Perimeter()) // 14
	fmt.Println(rect.Scale(2))    // {8 6}
}
```

---

## 5. Memory simulation

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func (u User) PrintDetails() {
	fmt.Println(u.Name, u.Age)
}

func main() {
	user1 := User{"Habib", 30}
	user1.PrintDetails()
}
```

**Phase 1: compile time**

- The compiler records the **type** `User` and its two fields.
- It compiles `PrintDetails` to machine code in the **code segment** and *binds* it to `User`: internally it's named something like `main.User.PrintDetails`.
- It compiles `main`.

```
CODE SEGMENT
┌────────────────────────────────┐
│ main.User.PrintDetails  ◄─ bound to type User
│ main.main                      │
└────────────────────────────────┘
```

**Phase 2: run time**

**Step 1: `main` starts; `user1 := User{"Habib", 30}`**

```
STACK
┌─────────────────────────────┐
│ main                        │
│  user1 [ "Habib" | 30 ]     │
└─────────────────────────────┘
```

**Step 2: `user1.PrintDetails()`.** Go copies `user1` into the method's receiver parameter `u`, creating a new frame:

```
┌─────────────────────────────┐
│ PrintDetails                │
│  u [ "Habib" | 30 ]  (COPY) │  ← receiver: a copy of user1
├─────────────────────────────┤
│ main                        │
│  user1 [ "Habib" | 30 ]     │
└─────────────────────────────┘
```

It prints `Habib 30`.

**Step 3: return → the frame (and the copy `u`) is destroyed.** `user1` is untouched.

**Insight:** with a *value receiver*, the method operates on a **copy**. That's exactly like passing a struct to a normal function (Chapter 21, section 10).

---

## 6. A method is a function in disguise

This is one of the most clarifying facts about Go methods:

```go
func (u User) PrintDetails() { ... }
```

is **the same as** a normal function that takes the receiver as its first parameter:

```go
func User_PrintDetails(u User) { ... }
```

and `user1.PrintDetails()` is just nice syntax for `User_PrintDetails(user1)`. Go even lets you write it that way explicitly, using a **method expression**:

```go
package main

import "fmt"

type User struct{ Name string }

func (u User) Hello() { fmt.Println("Hi,", u.Name) }

func main() {
	u := User{"Asha"}
	u.Hello()      // normal method call
	User.Hello(u)  // method expression: the function form; receiver is the first argument
}
```

Both lines print `Hi, Asha`. Nothing mysterious: methods aren't a new kind of magic, just a convenient way of calling a function where the receiver is the first argument.

---

## 7. Value receivers vs. pointer receivers

There are two receiver kinds:

```go
func (u User)  Method1() {} // VALUE receiver:   u is a COPY of the value
func (u *User) Method2() {} // POINTER receiver: u points to the ORIGINAL
```

### The difference in one program

```go
package main

import "fmt"

type Counter struct{ N int }

func (c Counter) IncrementValue() { // value receiver: works on a copy
	c.N++
}

func (c *Counter) IncrementPointer() { // pointer receiver: works on the original
	c.N++
}

func main() {
	c := Counter{}

	c.IncrementValue()
	fmt.Println(c.N) // 0  (only a copy changed)

	c.IncrementPointer()
	fmt.Println(c.N) // 1  (original changed)
}
```

Output:

```
0
1
```

Memory picture for `c.IncrementPointer()`:

```
STACK
┌───────────────────────────┐
│ IncrementPointer          │
│  c ───────────────┐       │   c holds the ADDRESS of main's Counter
├───────────────────┼───────┤
│ main              ▼       │
│  c [ N = 1 ]  ◄── modified in place
└───────────────────────────┘
```

(Pointers get their full treatment in [Chapter 24](24-pointers.md). For now: `*T` means "pointer to a `T`", i.e., "the address of the original".)

### Convenient auto-conversion

You wrote `c.IncrementPointer()`, not `(&c).IncrementPointer()`. Go automatically takes the address when `c` is addressable (a variable). Likewise, calling a value-receiver method through a pointer auto-dereferences:

```go
p := &Counter{}
p.IncrementPointer() // fine
p.IncrementValue()   // also fine: Go copies *p
```

### Summary

| | Value receiver `(t T)` | Pointer receiver `(t *T)` |
|--|------------------------|---------------------------|
| Receives | A **copy** | The **address** of the original |
| Can modify the original? | ❌ No | ✅ Yes |
| Cost per call | Copies the whole value | Copies one pointer (8 bytes) |
| Works when called on a `nil` pointer? | n/a (would panic when dereferenced) | ✅ Legal; you can handle `nil` yourself |
| Called on non-addressable values (e.g., `T{}.M()`, map elements)? | ✅ | ❌ Not allowed |

---

## 8. Choosing between them

Practical rules (from the Go team's own guidance):

**Use a pointer receiver if:**
1. The method must **modify** the receiver.
2. The struct is **large** (avoid copying).
3. The struct contains something that **must not be copied**, such as a `sync.Mutex` (Chapter 68).
4. **Consistency**: if *any* method needs a pointer receiver, make *all* methods on that type pointer receivers.

**Use a value receiver if:**
1. The type is a small, immutable-ish value (e.g., `Point`, `time.Time`, a small config).
2. It's a basic type or a map/slice/func alias that you don't reassign.
3. You want *value semantics* (callers can't be surprised by mutation).

**When in doubt: pointer receiver**, and keep them consistent per type.

> ⚠️ **Don't mix** value and pointer receivers on the same type without a reason. It confuses readers and can affect which interfaces the type satisfies (Chapter 51).

---

## 9. Methods on any local type

Methods aren't only for structs. You can define methods on **any type you declare in your package**:

```go
package main

import "fmt"

type Celsius float64

func (c Celsius) ToFahrenheit() float64 { return float64(c)*9/5 + 32 }

type Weekday int

func (d Weekday) IsWeekend() bool { return d == 0 || d == 6 }

type StringList []string

func (l StringList) Len() int { return len(l) }

func main() {
	fmt.Println(Celsius(100).ToFahrenheit()) // 212
	fmt.Println(Weekday(6).IsWeekend())      // true
	fmt.Println(StringList{"a", "b"}.Len())  // 2
}
```

But you **cannot** define methods on types from *other* packages (`int`, `string`, `time.Time`). Create a new named type based on them, as `Celsius` does with `float64`:

```go
// INTENTIONAL ERROR
package main

func (i int) Double() int { return i * 2 } // ❌ cannot define new methods on non-local type int

func main() {}
```

**Why "receiver functions only work with custom types":** a method needs a type *you* declared, to bind to. That's a rule of the language.

---

## 10. Multiple methods and organization

A type can have as many methods as you like, and they can be spread across files of the same package:

```go
package main

import (
	"fmt"
	"strings"
)

type Account struct {
	Owner   string
	Balance float64
}

func (a *Account) Deposit(amount float64) {
	a.Balance += amount
}

func (a *Account) Withdraw(amount float64) error {
	if amount > a.Balance {
		return fmt.Errorf("insufficient funds: have %.2f, need %.2f", a.Balance, amount)
	}
	a.Balance -= amount
	return nil
}

func (a Account) Summary() string {
	return fmt.Sprintf("%s: $%.2f", strings.ToUpper(a.Owner), a.Balance)
}

func main() {
	acc := Account{Owner: "asha", Balance: 100}
	acc.Deposit(50)
	if err := acc.Withdraw(500); err != nil {
		fmt.Println("error:", err)
	}
	_ = acc.Withdraw(30)
	fmt.Println(acc.Summary())
}
```

Output:

```
error: insufficient funds: have 150.00, need 500.00
ASHA: $120.00
```

> (Here `Summary` uses a value receiver while the others use pointers, which is the "mixed" style advised against above. It's shown to prove that it *works*, but for a real type you'd make all three pointer receivers.)

### Binding: which methods belong to which type?

Methods are looked up **by the type of the value**:

```
type User    ─── PrintDetails, Birthday, Email...
type Product ─── PrintDetails, ApplyDiscount...   (a different PrintDetails!)
```

Two types can have methods with the same name without conflict. The receiver type disambiguates. That's how `Circle.Area()` and `Rectangle.Area()` coexist.

---

## 11. The `String()` method and `fmt`

If your type has a method **`String() string`**, `fmt` uses it automatically when printing (this is the `fmt.Stringer` interface):

```go
package main

import "fmt"

type Point struct{ X, Y int }

func (p Point) String() string {
	return fmt.Sprintf("(%d, %d)", p.X, p.Y)
}

func main() {
	p := Point{3, 4}
	fmt.Println(p)          // (3, 4)
	fmt.Printf("%v %s\n", p, p) // (3, 4) (3, 4)
}
```

Without `String()`, `Println(p)` would show `{3 4}`. This is a lovely early taste of **interfaces**: `fmt.Println` doesn't know about `Point`; it only knows "if something has a `String() string` method, call it."

Similar built-in conventions: `Error() string` makes a type an `error`.

---

## 12. Method values and method expressions

A method can be treated as a value (functions are first-class, Chapter 17):

```go
package main

import "fmt"

type Greeter struct{ Greeting string }

func (g Greeter) Greet(name string) string { return g.Greeting + ", " + name }

func main() {
	g := Greeter{"Hello"}

	// Method VALUE: bound to g. Receiver is captured (copied) at this moment.
	greet := g.Greet
	fmt.Println(greet("Asha")) // Hello, Asha

	// Method EXPRESSION: unbound; receiver becomes the first parameter.
	f := Greeter.Greet
	fmt.Println(f(Greeter{"Namaste"}, "Rahim")) // Namaste, Rahim
}
```

Method values are handy as callbacks: `http.HandleFunc("/", server.handleHome)` where `handleHome` is a method whose receiver carries the server's state (Chapter 43+).

---

## 13. Methods and embedding

When a struct **embeds** another, the embedded type's methods are **promoted**:

```go
package main

import "fmt"

type Animal struct{ Name string }

func (a Animal) Speak() string { return a.Name + " makes a sound" }

type Dog struct {
	Animal // embedded
	Breed  string
}

func main() {
	d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Labrador"}
	fmt.Println(d.Speak()) // Rex makes a sound (promoted from Animal)
}
```

`Dog` can also define its *own* `Speak`, which then **overrides** (shadows) the promoted one (Chapter 12's shadowing idea again):

```go
func (d Dog) Speak() string { return d.Name + " barks" }
```

This is Go's alternative to inheritance: **composition**. You'll use it in the clean-architecture chapters.

---

## 14. Methods are a stepping stone to interfaces

An **interface** is a set of method signatures. **Any type with those methods satisfies the interface automatically**, with no `implements` keyword. Preview:

```go
package main

import (
	"fmt"
	"math"
)

type Shape interface {
	Area() float64
}

type Rectangle struct{ W, H float64 }
type Circle struct{ R float64 }

func (r Rectangle) Area() float64 { return r.W * r.H }
func (c Circle) Area() float64    { return math.Pi * c.R * c.R }

func printArea(s Shape) { fmt.Printf("%.2f\n", s.Area()) }

func main() {
	printArea(Rectangle{3, 4}) // 12.00
	printArea(Circle{1})       // 3.14
}
```

`printArea` works with **any** shape. That's polymorphism, powered by methods. Full coverage in [Chapter 51](51-interfaces-and-design-patterns.md).

---

## 15. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Expecting a value-receiver method to modify the receiver | Change disappears | Use a pointer receiver `(u *User)` |
| 2 | Mixing value and pointer receivers on one type | Confusing behavior, interface surprises | Pick one style per type |
| 3 | Defining a method on a non-local type (`int`, `time.Time`) | `cannot define new methods on non-local type` | Define your own named type |
| 4 | Naming the receiver `this`/`self` | Un-idiomatic | Use a short name like `u`, `r` |
| 5 | Calling a method on the type name: `User.PrintDetails()` | Missing receiver argument | Call on an instance: `user1.PrintDetails()` |
| 6 | Method name equals a field name | `field and method with the same name Name` | Rename one (e.g., field `Name`, method `FullName`) |
| 7 | Calling a pointer-receiver method on a non-addressable value: `User{}.SetName("x")` | `cannot call pointer method` | Assign to a variable first |
| 8 | Nil pointer receiver dereferenced | runtime panic | Check `if u == nil` where it's legitimate |
| 9 | Copying a struct with a mutex (value receiver) | `go vet`: copies lock value | Use pointer receivers |
| 10 | Lowercase method name and wondering why other packages can't call it | Unexported | Capitalize to export |

Mistake #6 in code:

```go
// INTENTIONAL ERROR
package main

type User struct{ Name string }

func (u User) Name() string { return "x" } // ❌ field and method with the same name Name

func main() {}
```

---

## 16. Exercises

### Exercise 1: First method
Add `IsAdult() bool` to `Person{Name string; Age int}`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type Person struct {
	Name string
	Age  int
}

func (p Person) IsAdult() bool { return p.Age >= 18 }

func main() {
	fmt.Println(Person{"Asha", 30}.IsAdult(), Person{"Tia", 12}.IsAdult()) // true false
}
```
</details>

### Exercise 2: Method with parameters
Add `Scale(factor float64)` to `Rectangle` that changes it in place, and `Area()`. Print the area before and after `Scale(2)`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type Rectangle struct{ W, H float64 }

func (r Rectangle) Area() float64 { return r.W * r.H }

func (r *Rectangle) Scale(f float64) {
	r.W *= f
	r.H *= f
}

func main() {
	r := Rectangle{3, 4}
	fmt.Println(r.Area()) // 12
	r.Scale(2)
	fmt.Println(r.Area()) // 48
}
```
</details>

### Exercise 3: Predict

```go
package main

import "fmt"

type Box struct{ Items int }

func (b Box) AddV()  { b.Items++ }
func (b *Box) AddP() { b.Items++ }

func main() {
	b := Box{}
	b.AddV()
	b.AddV()
	b.AddP()
	b.AddP()
	b.AddP()
	fmt.Println(b.Items)
}
```

<details><summary>Solution</summary>

`3`. The two `AddV` calls change only copies; the three `AddP` calls change the original.
</details>

### Exercise 4: `String()`
Give `Temperature float64` a `String()` method printing like `36.6°C`. Print it with `Println`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type Temperature float64

func (t Temperature) String() string { return fmt.Sprintf("%.1f°C", float64(t)) }

func main() {
	fmt.Println(Temperature(36.6)) // 36.6°C
}
```
</details>

### Exercise 5: Bank account
Build `Account` with `Deposit`, `Withdraw` (returns error if insufficient), and `Balance() float64`. Use pointer receivers throughout.

<details><summary>Solution</summary>

```go
package main

import (
	"errors"
	"fmt"
)

type Account struct{ balance float64 }

func (a *Account) Deposit(n float64) { a.balance += n }

func (a *Account) Withdraw(n float64) error {
	if n > a.balance {
		return errors.New("insufficient funds")
	}
	a.balance -= n
	return nil
}

func (a *Account) Balance() float64 { return a.balance }

func main() {
	var acc Account
	acc.Deposit(100)
	fmt.Println(acc.Withdraw(150)) // insufficient funds
	fmt.Println(acc.Withdraw(40))  // <nil>
	fmt.Println(acc.Balance())     // 60
}
```
</details>

### Exercise 6: Method on a slice type
Define `type IntSlice []int` with `Sum() int` and `Max() int`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type IntSlice []int

func (s IntSlice) Sum() int {
	total := 0
	for _, n := range s {
		total += n
	}
	return total
}

func (s IntSlice) Max() int {
	m := s[0]
	for _, n := range s[1:] {
		if n > m {
			m = n
		}
	}
	return m
}

func main() {
	nums := IntSlice{3, 9, 4}
	fmt.Println(nums.Sum(), nums.Max()) // 16 9
}
```
</details>

### Exercise 7 (challenge): Method chaining
Make `Builder` support `b.Add("a").Add("b").String()` by having `Add` return the pointer receiver.

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"strings"
)

type Builder struct{ parts []string }

func (b *Builder) Add(s string) *Builder {
	b.parts = append(b.parts, s)
	return b // return the same pointer so calls can chain
}

func (b *Builder) String() string { return strings.Join(b.parts, "-") }

func main() {
	b := &Builder{}
	fmt.Println(b.Add("a").Add("b").Add("c")) // a-b-c
}
```
</details>

---

## 17. Quiz

1. What is the "receiver" in a method declaration?
2. Is `u.Method()` different from `Method(u)`? What's the relationship?
3. When does a method change the original struct?
4. Which types can you attach methods to?
5. What does a `String() string` method do for `fmt.Println`?
6. Why prefer consistent receiver kinds per type?

<details><summary>Answers</summary>

1. The extra parameter before the method name; the value the method is called on.
2. Same thing: a method is a function whose first parameter is the receiver.
3. When it has a pointer receiver (`*T`).
4. Any named type declared in the same package (structs, `type X int`, slices, maps, funcs).
5. `Println` calls it to get the text representation.
6. Mixed styles confuse readers and can change which interfaces the type satisfies.
</details>

---

## 18. Summary

- A **method** is a function with a **receiver**: `func (r T) Name(params) results`. Call it as `value.Name(args)`.
- It's **syntactic sugar**: `v.M(x)` ≡ `T.M(v, x)`. The receiver is the first argument.
- **Value receiver** `(t T)`: works on a **copy**. **Pointer receiver** `(t *T)`: works on the **original**, and can modify it.
- Use pointer receivers for mutation, large structs, or uncopyable fields, and keep one style per type.
- You can attach methods to any **local named type**, not just structs; not to `int`/`string` directly.
- `String()` customizes printing; **embedding** promotes methods; **interfaces** (Chapter 51) are satisfied by having the right methods.

### ➡️ What's next?

We've used slices and pointers a few times without explaining them. Time to fix that: [Chapter 23](23-arrays.md) covers **arrays**, the fixed-size building block beneath slices.
