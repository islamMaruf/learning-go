# Chapter 21: Structs — Custom Types, Objects, and Properties

> **Goal of this chapter:** Build your own data types. A **struct** groups related values (a user's name, age, and email) into one unit. You'll learn to declare structs, create *instances*, read and change fields, nest structs, compare and copy them, and see how they sit in memory.

**Difficulty:** 🟠 Intermediate  **Estimated time:** 2–2.5 hours  **Prerequisite:** [Chapter 2](02-variables-and-data-types.md), [Chapter 5](05-functions-with-return-values.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The problem structs solve](#2-the-problem-structs-solve)
3. [What is a struct?](#3-what-is-a-struct)
4. [Declaring a struct type](#4-declaring-a-struct-type)
5. [Creating instances](#5-creating-instances)
6. [Accessing and changing fields](#6-accessing-and-changing-fields)
7. [Type vs. instance vs. value: the plate analogy](#7-type-vs-instance-vs-value)
8. [Memory model](#8-memory-model)
9. [Zero values](#9-zero-values)
10. [Copying: structs are values](#10-copying-structs-are-values)
11. [Comparing structs](#11-comparing-structs)
12. [Nested structs and embedding](#12-nested-structs-and-embedding)
13. [Anonymous structs](#13-anonymous-structs)
14. [Constructor functions](#14-constructor-functions)
15. [Printing structs](#15-printing-structs)
16. [Exported fields, tags, and JSON (a peek)](#16-exported-fields-tags-and-json)
17. [Struct size and padding (optional)](#17-struct-size-and-padding)
18. [Common mistakes](#18-common-mistakes)
19. [Exercises](#19-exercises)
20. [Quiz](#20-quiz)
21. [Summary](#21-summary)

---

## 1. What you will learn

- Why programs need **custom types**
- How to declare a `struct` and create values of it
- The precise words **type**, **instance/value**, **field/property**
- That struct assignment **copies** the data (value semantics)
- How to nest structs and compare them
- How structs are laid out in memory
- The `NewX` constructor convention

---

## 2. The problem structs solve

Imagine storing a user. With only the types we know (`string`, `int`, …) you'd write:

```go
name1 := "Asha"
age1 := 30
email1 := "asha@example.com"

name2 := "Rahim"
age2 := 25
email2 := "rahim@example.com"
```

Problems:
- With 1,000 users you'd need 3,000 variables with invented names.
- Passing a user to a function means passing **three** separate arguments (and getting the order right).
- Nothing says these three variables *belong together*.

We want a **bundle**: one variable holding a user's name, age, and email. That is a **struct**.

---

## 3. What is a struct?

> A **struct** (short for *structure*) is a **custom type** that groups a fixed set of named values, called **fields**, into one unit.

Go gives you **built-in types**: `int`, `string`, `bool`, `float64`. A struct lets you make **your own types**: `User`, `Product`, `Point`, `Order`.

```
Built-in type:    int          →  holds ONE number
Custom struct:    User         →  holds ID + Name + Email + Age (all together)
```

> **Analogy: a paper form.** A blank *registration form* has labeled boxes (Name, Age, Email). The blank form is the **type**. A filled-in form for Asha is one **instance**. Filling another form for Rahim gives another instance. The blank form never changes.

This is Go's building block for modeling the real world: users, products, orders, HTTP requests, configuration. Almost every Go program is full of them.

---

## 4. Declaring a struct type

```go
type User struct {
	Name  string
	Age   int
	Email string
}
```

Syntax:

```
type  TypeName  struct {
    FieldName1  Type1
    FieldName2  Type2
    ...
}
```

| Part | Meaning |
|------|---------|
| `type` | Keyword: "I'm defining a new type" |
| `User` | The new type's name (capitalize to export it) |
| `struct { ... }` | The kind of type: a bundle of fields |
| `Name string` | A **field**: name first, then type (like function parameters) |

Notes:
- Declaring a type **does not create any data**. It only describes a shape (the blank form).
- Declare types at **package level** (usually), outside functions.
- Fields of the same type can share a line: `X, Y int`.
- Field order matters for positional literals, and slightly for memory size (section 17).

```go
type Point struct {
	X, Y float64
}
```

---

## 5. Creating instances

An **instance** (or **value**) of a struct type is one filled-in copy. There are several ways to make one.

### 5.1 Struct literal with field names ✅ (recommended)

```go
package main

import "fmt"

type User struct {
	Name  string
	Age   int
	Email string
}

func main() {
	u := User{
		Name:  "Asha",
		Age:   30,
		Email: "asha@example.com",
	}
	fmt.Println(u) // {Asha 30 asha@example.com}
}
```

Advantages: order doesn't matter, it's self-documenting, and adding a field later doesn't break the code. **Note the trailing comma after the last field**: Go requires it when the closing brace is on its own line.

### 5.2 Positional literal (fragile)

```go
u := User{"Rahim", 25, "rahim@example.com"} // must supply ALL fields, in declaration order
```

Works, but breaks silently if you reorder or add fields. Prefer named fields except for tiny types like `Point{3, 4}`.

### 5.3 Partial literal: unspecified fields get zero values

```go
u := User{Name: "Karim"} // Age = 0, Email = ""
```

### 5.4 `var` declaration: all fields zero

```go
var u User // Name "", Age 0, Email ""
```

### 5.5 Pointer to a new struct

```go
p := &User{Name: "Nadia", Age: 28} // p is *User (a pointer); see Chapter 24
```

### 5.6 `new`

```go
p := new(User) // same as &User{}: a *User pointing at a zeroed User
```

Rarely used; `&User{...}` is more common.

---

## 6. Accessing and changing fields

Use **dot notation**: `variable.Field`.

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func main() {
	u := User{Name: "Asha", Age: 30}

	fmt.Println(u.Name) // read → Asha
	fmt.Println(u.Age)  // 30

	u.Age = 31          // write
	u.Name += " Rahman" // fields work like variables of their type
	fmt.Println(u)      // {Asha Rahman 31}
}
```

Terminology (you'll hear all of these; they mean the same thing):

| Word | Used by |
|------|---------|
| **field** | Go's official term |
| **property** | JavaScript/C# developers |
| **member variable** / **attribute** | Java/C++/Python developers |

Fields are used exactly like ordinary variables of their type: `u.Age++`, `if u.Age >= 18`, `len(u.Name)`.

---

## 7. Type vs. instance vs. value

These words cause a lot of confusion, so let's fix them with a restaurant.

> **The plate analogy.** A restaurant has a **plate design**: round, white, 25 cm. That design is the **type**. It's a *description*, and you can't eat a design. When a customer orders, the kitchen makes an **actual plate with food**: an **instance**. Ten customers → ten separate plates. Changing the food on plate 3 doesn't change plate 4. And the *design* didn't change either.

| Restaurant | Go | Example |
|-----------|-----|---------|
| Plate design | **Type** | `type User struct {...}` |
| A plate with food | **Instance / value / object** | `u1 := User{...}` |
| Compartments of the plate | **Fields** | `Name`, `Age` |
| Food in a compartment | **Field value** | `"Asha"`, `30` |

```go
type User struct { Name string; Age int } // TYPE: takes no memory for data
u1 := User{"Asha", 30}                     // INSTANCE #1
u2 := User{"Rahim", 25}                    // INSTANCE #2 (independent of #1)
```

**"Object" in Go:** Go's documentation avoids the word "object" (it isn't a class-based OO language, Chapter 11), but developers commonly say "a `User` object" for an instance. It's fine in conversation. In this course: *type* = the definition; *instance / value* = the data.

Also: **`int` is a type; `5` is a value of that type.** `User` is a type; `User{"Asha", 30}` is a value. Same relationship.

---

## 8. Memory model

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func main() {
	user1 := User{Name: "Asha", Age: 30}
	user2 := User{Name: "Rahim", Age: 25}
	fmt.Println(user1, user2)
}
```

**Phase 1: compile time.** The compiler learns the *shape* of `User` (two fields, their types and offsets). No memory holds data yet: types exist only in the compiler's head. The code of `main` goes in the code segment.

**Phase 2: run time.**

`user1 := User{...}`: `main`'s frame gets a block big enough for one `User` (a string header + an int), filled in:

```
STACK
┌──────────────────────────────────┐
│ main                             │
│  user1 ┌───────────┬────────┐    │
│        │ Name      │ Age    │    │
│        │ "Asha"    │  30    │    │
│        └───────────┴────────┘    │
└──────────────────────────────────┘
```

`user2 := User{...}`: a **second, separate** block:

```
┌──────────────────────────────────────────┐
│ main                                     │
│  user1  [ "Asha"  | 30 ]                 │
│  user2  [ "Rahim" | 25 ]                 │
└──────────────────────────────────────────┘
```

Key points:
- Each instance has **its own** storage for every field. Changing `user1.Age` never touches `user2.Age`.
- The fields sit **side by side** in one contiguous block: a struct is *one value*, not a collection of separate variables.
- Strings are stored as a small header (pointer + length) pointing to the text bytes; that's an implementation detail, but explains why a `string` field takes 16 bytes.
- When `main` ends, its frame (with both instances) is popped.

---

## 9. Zero values

Every field starts at its type's zero value (Chapter 2), so `var u User` is always safe and predictable:

```go
package main

import "fmt"

type User struct {
	ID    int
	Name  string
	Email string
	Age   int
	Admin bool
}

func main() {
	var u User
	fmt.Printf("%+v\n", u)
}
```

Output:

```
{ID:0 Name: Email: Age:0 Admin:false}
```

Designing types so that the **zero value is useful** is a Go proverb ("make the zero value useful").

---

## 10. Copying: structs are values

Assigning a struct, passing it to a function, or returning it **copies all its fields**.

```go
package main

import "fmt"

type User struct {
	Name string
	Age  int
}

func birthday(u User) {
	u.Age++ // changes the COPY
	fmt.Println("inside:", u.Age)
}

func main() {
	a := User{"Asha", 30}
	b := a // copy

	b.Name = "Bina"
	fmt.Println(a.Name, b.Name) // Asha Bina  (independent)

	birthday(a)
	fmt.Println("outside:", a.Age) // 30 (unchanged)
}
```

Output:

```
Asha Bina
inside: 31
outside: 30
```

This is the same pass-by-value behavior as `int` (Chapter 4). To let a function **modify the original**, pass a **pointer** (Chapter 24):

```go
func birthday(u *User) { u.Age++ } // called as birthday(&a)
```

> Copying a large struct can be costly, so big structs are usually passed by pointer, but *correctness first*: use value semantics when you want isolation.

---

## 11. Comparing structs

Two struct values are `==` if **all fields are equal**, provided all fields are *comparable* types (numbers, strings, bools, pointers, arrays of comparable, other comparable structs):

```go
package main

import "fmt"

type Point struct{ X, Y int }

func main() {
	p1 := Point{1, 2}
	p2 := Point{1, 2}
	p3 := Point{2, 1}

	fmt.Println(p1 == p2) // true
	fmt.Println(p1 == p3) // false
}
```

A struct containing a **slice, map, or function** field is *not comparable* with `==` (compile error). Use `reflect.DeepEqual` or compare fields manually.

A comparable struct can also be used as a **map key**: `visited := map[Point]bool{}`.

---

## 12. Nested structs and embedding

### 12.1 Nested (named) field

A field's type can be another struct:

```go
package main

import "fmt"

type Address struct {
	City    string
	Country string
}

type Person struct {
	Name    string
	Address Address // a struct inside a struct
}

func main() {
	p := Person{
		Name: "Asha",
		Address: Address{
			City:    "Dhaka",
			Country: "Bangladesh",
		},
	}

	fmt.Println(p.Address.City) // Dhaka
	p.Address.City = "Chattogram"
	fmt.Println(p)              // {Asha {Chattogram Bangladesh}}
}
```

In memory the inner struct is laid out **inside** the outer one.

### 12.2 Embedding (composition)

If you write just the type name (no field name), it's **embedded**, and its fields and methods are *promoted*:

```go
package main

import "fmt"

type Address struct {
	City string
}

type Employee struct {
	Name string
	Address // embedded: no field name
}

func main() {
	e := Employee{Name: "Asha", Address: Address{City: "Dhaka"}}
	fmt.Println(e.City)         // promoted: shorthand for e.Address.City
	fmt.Println(e.Address.City) // still works
}
```

Go has **no inheritance**; embedding (composition) is how it reuses structure. Chapters 22 and 51 build on this.

---

## 13. Anonymous structs

For one-off shapes, skip the `type`:

```go
package main

import "fmt"

func main() {
	config := struct {
		Host string
		Port int
	}{
		Host: "localhost",
		Port: 8080,
	}
	fmt.Println(config.Host, config.Port)
}
```

Handy in tests (tables of cases) and for quick JSON shapes. If you use the shape in more than one place, name it.

---

## 14. Constructor functions

Go has no constructors, but the convention is a function named **`NewTypeName`** that returns a ready-to-use value (often a pointer). It can validate and set defaults:

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

type User struct {
	Name  string
	Email string
	Active bool
}

func NewUser(name, email string) (*User, error) {
	if strings.TrimSpace(name) == "" {
		return nil, errors.New("name is required")
	}
	if !strings.Contains(email, "@") {
		return nil, errors.New("invalid email")
	}
	return &User{Name: name, Email: email, Active: true}, nil
}

func main() {
	u, err := NewUser("Asha", "asha@example.com")
	if err != nil {
		fmt.Println("error:", err)
		return
	}
	fmt.Printf("%+v\n", *u)

	_, err = NewUser("", "x")
	fmt.Println("error:", err)
}
```

Output:

```
{Name:Asha Email:asha@example.com Active:true}
error: name is required
```

Because `NewUser` returns `*User` (a pointer to a heap-allocated value; escape analysis!), the same instance can be shared. You'll write many `New...` functions in the e-commerce project.

---

## 15. Printing structs

`fmt` has three handy verbs for structs:

```go
package main

import "fmt"

type User struct {
	ID    int
	Name  string
	Admin bool
}

func main() {
	u := User{1, "Asha", true}
	fmt.Printf("%v\n", u)  // {1 Asha true}
	fmt.Printf("%+v\n", u) // {ID:1 Name:Asha Admin:true}
	fmt.Printf("%#v\n", u) // main.User{ID:1, Name:"Asha", Admin:true}
	fmt.Printf("%T\n", u)  // main.User

	p := &u
	fmt.Printf("%+v\n", p) // &{ID:1 Name:Asha Admin:true}
}
```

| Verb | Shows |
|------|-------|
| `%v` | values only |
| `%+v` | field names + values (**best for debugging**) |
| `%#v` | Go-syntax representation (type + names + values) |
| `%T` | the type |

---

## 16. Exported fields, tags, and JSON

### Exported vs. unexported fields

The capital-letter rule (Chapter 10) applies to fields. Other packages can only see fields starting with a capital:

```go
type Account struct {
	Owner   string // exported
	balance int    // unexported: private to this package
}
```

This gives real encapsulation: force outsiders to use your methods.

### Struct tags & JSON (peek)

A **tag** is a string of metadata after a field. The `encoding/json` package uses it:

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Product struct {
	ID    int     `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
	note  string  // unexported: ignored by JSON
}

func main() {
	p := Product{ID: 1, Name: "Notebook", Price: 2.5, note: "hidden"}

	data, _ := json.Marshal(p) // struct → JSON text
	fmt.Println(string(data))  // {"id":1,"name":"Notebook","price":2.5}

	var back Product
	_ = json.Unmarshal([]byte(`{"id":7,"name":"Pen","price":0.75}`), &back)
	fmt.Printf("%+v\n", back) // {ID:7 Name:Pen Price:0.75 note:}
}
```

Fields must be **exported** (capitalized) to be visible to `encoding/json`, and tags control the JSON key names. This is *the* way web APIs in Go read and write JSON, and we'll do exactly that in Chapters 40–41.

---

## 17. Struct size and padding

*(Optional: for the curious.)* CPUs like data aligned to its size, so Go inserts invisible **padding** bytes between fields. **Field order affects total size**:

```go
package main

import (
	"fmt"
	"unsafe"
)

type A struct { // bool, int64, bool
	a bool
	b int64
	c bool
}

type B struct { // int64, bool, bool
	b int64
	a bool
	c bool
}

func main() {
	fmt.Println(unsafe.Sizeof(A{})) // 24
	fmt.Println(unsafe.Sizeof(B{})) // 16
}
```

Real output: `24` and `16`. Same fields, different order, 8 bytes saved per value! `A` pads after `a` (7 bytes) and after `c` (7 bytes); `B` packs the two bools together.

Don't reorder fields for readability's sake in ordinary code; this matters only when you allocate millions of a struct. Related: a `string` field is 16 bytes (pointer + length), an `int` is 8 on 64-bit systems.

---

## 18. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | Using the type where a value is needed: `User.Name = "x"` | `cannot use User (type) as value` / invalid | Create an instance first: `u := User{}` |
| 2 | Thinking two instances share fields | Modifying one changes the other? No | Each instance is independent (unless they hold pointers to the same thing) |
| 3 | Missing trailing comma in a multi-line literal | `syntax error: unexpected newline` | Add the comma after the last field |
| 4 | Positional literal with the wrong number of values | `too few values in struct literal` | Supply all fields or use named fields |
| 5 | Assuming a function modifies your struct | It gets a **copy** | Pass a pointer `*User` |
| 6 | Comparing structs with slice/map fields | `invalid operation: ... cannot be compared` | Compare fields manually / `reflect.DeepEqual` |
| 7 | Lowercase field names then wondering why JSON is empty | Unexported fields are invisible to other packages | Capitalize |
| 8 | Forgetting `&` when a function wants `*User` | `cannot use u (variable of type User) as *User` | Pass `&u` |
| 9 | Accessing fields on a `nil` pointer | runtime panic | Check for `nil` |
| 10 | Naming a field the same as its type in a confusing way | Readability | Choose distinct names |

Mistake #1 in code:

```go
// INTENTIONAL ERROR
package main

type User struct{ Name string }

func main() {
	User.Name = "Asha" // ❌ a type has no data; you need an instance
}
```

---

## 19. Exercises

### Exercise 1: Your first struct
Define `Book` with `Title`, `Author` (strings) and `Pages` (int). Create two books and print them with `%+v`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type Book struct {
	Title  string
	Author string
	Pages  int
}

func main() {
	b1 := Book{Title: "The Go Programming Language", Author: "Donovan & Kernighan", Pages: 380}
	b2 := Book{Title: "Concurrency in Go", Author: "Katherine Cox-Buday", Pages: 238}
	fmt.Printf("%+v\n%+v\n", b1, b2)
}
```
</details>

### Exercise 2: Independence
Create `p1 := Point{1, 2}`, copy it into `p2`, change `p2.X`, and print both. What do you see, and why?

<details><summary>Solution</summary>

```go
package main

import "fmt"

type Point struct{ X, Y int }

func main() {
	p1 := Point{1, 2}
	p2 := p1
	p2.X = 99
	fmt.Println(p1, p2) // {1 2} {99 2}
}
```
Assignment copies the struct; `p1` and `p2` are separate blocks of memory.
</details>

### Exercise 3: Nested struct
Model a `Rectangle` made of two `Point` corners (`TopLeft`, `BottomRight`) and write `area(r Rectangle) int`.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type Point struct{ X, Y int }

type Rectangle struct {
	TopLeft, BottomRight Point
}

func area(r Rectangle) int {
	width := r.BottomRight.X - r.TopLeft.X
	height := r.BottomRight.Y - r.TopLeft.Y
	return width * height
}

func main() {
	r := Rectangle{TopLeft: Point{0, 0}, BottomRight: Point{4, 3}}
	fmt.Println(area(r)) // 12
}
```
</details>

### Exercise 4: Zero values
Predict the output:

```go
package main

import "fmt"

type Config struct {
	Host    string
	Port    int
	Debug   bool
	Retries []int
}

func main() {
	var c Config
	fmt.Printf("%+v\n", c)
	fmt.Println(c.Retries == nil, len(c.Retries))
}
```

<details><summary>Solution</summary>

```
{Host: Port:0 Debug:false Retries:[]}
true 0
```
A nil slice prints as `[]` and has length 0. It's a perfectly usable zero value (you can `append` to it).
</details>

### Exercise 5: Constructor with validation
Write `NewProduct(name string, price float64) (*Product, error)` that rejects empty names and non-positive prices.

<details><summary>Solution</summary>

```go
package main

import (
	"errors"
	"fmt"
)

type Product struct {
	Name  string
	Price float64
}

func NewProduct(name string, price float64) (*Product, error) {
	if name == "" {
		return nil, errors.New("name is required")
	}
	if price <= 0 {
		return nil, errors.New("price must be positive")
	}
	return &Product{Name: name, Price: price}, nil
}

func main() {
	p, err := NewProduct("Pen", 0.75)
	fmt.Println(p, err)
	_, err = NewProduct("Pen", 0)
	fmt.Println(err)
}
```
</details>

### Exercise 6: Comparison
Which of these compile? For each that does, what's printed?

```go
type A struct{ N int }
type B struct{ Tags []string }

a1, a2 := A{1}, A{1}
b1, b2 := B{}, B{}
fmt.Println(a1 == a2)
fmt.Println(b1 == b2)
```

<details><summary>Solution</summary>

`a1 == a2` compiles → `true`. `b1 == b2` does **not** compile: a struct with a slice field isn't comparable.
</details>

### Exercise 7 (challenge): Modify through a function
Write `rename(u User, name string) User` (returns a modified copy) and `renameInPlace(u *User, name string)` (modifies the original). Show the difference.

<details><summary>Solution</summary>

```go
package main

import "fmt"

type User struct{ Name string }

func rename(u User, name string) User {
	u.Name = name
	return u
}

func renameInPlace(u *User, name string) {
	u.Name = name
}

func main() {
	u := User{"Asha"}

	renamed := rename(u, "Bina")
	fmt.Println(u.Name, renamed.Name) // Asha Bina

	renameInPlace(&u, "Chitra")
	fmt.Println(u.Name) // Chitra
}
```
</details>

---

## 20. Quiz

1. What's the difference between a type and an instance?
2. What are the components of a struct called?
3. What does `var u User` give you?
4. If `b := a` (both structs), and you change `b.Name`, does `a.Name` change?
5. Can you compare two structs with `==`? When not?
6. Why are struct fields lowercase-vs-uppercase important for JSON?

<details><summary>Answers</summary>

1. The type is the definition (blueprint); an instance is an actual value with data.
2. Fields (a.k.a. properties/members).
3. A `User` with every field at its zero value.
4. No: assignment copies.
5. Yes if all fields are comparable; not if it contains slices, maps, or functions.
6. Only exported (capitalized) fields are visible to `encoding/json`.
</details>

---

## 21. Summary

- A **struct** is a custom type grouping named **fields**: `type User struct { Name string; Age int }`.
- The **type** is a blueprint; each **instance** (`User{...}`) has its **own** memory for every field.
- Create with **named-field literals** (best), positional literals, `var`, `&T{}`, or a **`NewT` constructor**.
- Access with **dot notation**; unspecified fields get **zero values**.
- Structs are **values**: assignment and function arguments **copy**. Use **pointers** to share/modify (Chapter 24).
- **Nesting** and **embedding** compose structs; Go has no inheritance.
- Structs are `==`-comparable when all fields are; tags + JSON make them the backbone of web APIs.
- Field **order** affects **padding** and size; the capital-letter rule controls **visibility**.

### ➡️ What's next?

Data alone isn't enough. [Chapter 22](22-receiver-functions-methods.md) attaches **behavior** to structs with **receiver functions (methods)**.
