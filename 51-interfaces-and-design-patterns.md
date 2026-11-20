# Chapter 51: Interfaces and Design Patterns — The Power of Abstraction

> **Goal of this chapter:** Learn Go's most important abstraction tool. An **interface** describes *what something can do* without saying *what it is*; any type with the right methods satisfies it automatically. You'll learn how interfaces work (implicit satisfaction, method sets, pointer vs. value receivers, the dreaded **nil-interface gotcha**, type assertions, and what an interface looks like in memory), the classic **design patterns** that build on them (Strategy, Decorator, Factory, Adapter, Singleton and why Go prefers dependency injection), and then apply it to the project: `product.Store` and `user.Store` interfaces, real error handling, and a **fake store** that lets us finally test the failure paths.

**Difficulty:** 🔴 Advanced  **Estimated time:** 6 hours  **Prerequisite:** [Chapters 22, 50](50-removing-tight-coupling.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Abstraction: the idea](#2-abstraction-the-idea)
3. [What is an interface?](#3-what-is-an-interface)
4. [Implicit satisfaction](#4-implicit-satisfaction)
5. [Interface values and dynamic dispatch](#5-interface-values-and-dynamic-dispatch)
6. [Rules: complete methods, exact signatures](#6-rules-complete-methods-exact-signatures)
7. [Pointer vs. value receivers and method sets](#7-pointer-vs-value-receivers-and-method-sets)
8. [The nil-interface gotcha](#8-the-nil-interface-gotcha)
9. [The empty interface, type assertions, and type switches](#9-the-empty-interface-type-assertions-and-type-switches)
10. [Small interfaces and the standard library](#10-small-interfaces-and-the-standard-library)
11. [Composing interfaces](#11-composing-interfaces)
12. [Where should interfaces live?](#12-where-should-interfaces-live)
13. [Why interfaces? Five reasons](#13-why-interfaces-five-reasons)
14. [Design patterns](#14-design-patterns)
15. [Applying it: `product.Store` and `user.Store`](#15-applying-it-to-the-project)
16. [The complete code](#16-the-complete-code)
17. [Tests: fakes and failure paths](#17-tests-fakes-and-failure-paths)
18. [Common mistakes](#18-common-mistakes)
19. [Interview questions](#19-interview-questions)
20. [Exercises](#20-exercises)
21. [Quiz](#21-quiz)
22. [Summary](#22-summary)

---

## 1. What you will learn

- What **abstraction** is and why it reduces coupling
- Go **interfaces**: declaration, **implicit** satisfaction, and the `-er` naming convention
- **Method sets**: why a value type may *not* satisfy an interface that its pointer does
- The **typed-nil** trap: when `err != nil` is true for a "nil" error
- **Type assertions** and **type switches**; when (not) to use `any`
- The design maxim: **accept interfaces, return structs**, and define interfaces **where they are used**
- **Design patterns** in Go: Strategy, Decorator, Factory, Adapter, Singleton (and its modern alternative)
- How to use interfaces to make the project **storage-independent** and **testable**

---

## 2. Abstraction: the idea

> **Abstraction** means *hiding details and exposing only what matters*.

Driving a car, you use a **steering wheel, pedals, and gear stick**. You don't need to know if the engine is petrol, diesel, or electric, or how fuel injection works. The controls are an **abstraction** of "make the car go". Different cars (implementations) offer the *same* controls (interface), so you can drive any of them. Learn one car, and you can drive them all.

```
Concrete details (hidden):        Abstraction (what you use):
 petrol engine ─┐
 diesel engine ─┼──►   accelerate()  brake()  steer()
 electric motor ┘
```

In software the payoff is **loose coupling** (Chapter 50): code that depends on an abstraction (`"something that can store products"`) doesn't care *which* concrete thing (in-memory map, PostgreSQL, a test fake) sits behind it. Change the implementation, and the caller is untouched.

Levels of abstraction, from concrete to abstract:

| Level | Example | Detail |
|-------|---------|--------|
| Concrete type | `*database.ProductStore` | "a specific in-memory slice guarded by a mutex" |
| Interface | `Store` | "anything with `List`, `Get`, `Create`" |

An interface is a **contract**: *"if you have these methods, I can work with you."* It contains **no code**, only method signatures.

---

## 3. What is an interface?

An **interface type** is a set of method signatures.

```go
type Shape interface {
	Area() float64
	Perimeter() float64
}
```

Anything that has **both** methods, with exactly those signatures, *is* a `Shape`. You can then write functions that accept a `Shape` and work with every shape past, present, and future:

```go
package main

import (
	"fmt"
	"math"
)

type Shape interface {
	Area() float64
	Perimeter() float64
}

type Rectangle struct{ W, H float64 }
type Circle struct{ R float64 }

func (r Rectangle) Area() float64      { return r.W * r.H }
func (r Rectangle) Perimeter() float64 { return 2 * (r.W + r.H) }

func (c Circle) Area() float64      { return math.Pi * c.R * c.R }
func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.R }

// describe knows nothing about rectangles or circles.
func describe(s Shape) {
	fmt.Printf("%T: area=%.2f perimeter=%.2f\n", s, s.Area(), s.Perimeter())
}

func main() {
	shapes := []Shape{Rectangle{3, 4}, Circle{1}, Rectangle{2, 2}}
	for _, s := range shapes {
		describe(s)
	}
}
```

Output:

```
main.Rectangle: area=12.00 perimeter=14.00
main.Circle: area=3.14 perimeter=6.28
main.Rectangle: area=4.00 perimeter=8.00
```

One function, one slice type, many different concrete types: **polymorphism** ("many forms").

### Naming convention

Single-method interfaces are conventionally named with an **`-er` suffix** from the method: `Reader` (`Read`), `Writer` (`Write`), `Stringer` (`String`), `Closer` (`Close`). Multi-method interfaces get descriptive nouns (`Store`, `Shape`).

---

## 4. Implicit satisfaction

In Java or C# a class must declare `implements Shape`. **Go has no `implements`.** A type satisfies an interface **automatically, simply by having the methods**. This is sometimes called *structural typing* or "duck typing that the compiler checks": *if it walks like a duck and quacks like a duck, it's a duck.*

```go
type Sized interface{ Size() int }

// Somewhere else entirely, written without knowing Sized exists:
type File struct{ bytes int }
func (f File) Size() int { return f.bytes }

var s Sized = File{100} // ✅ File satisfies Sized just by having Size() int
```

Why this matters:

1. **Decoupling in time and space.** The type author doesn't need to import the interface's package or even know it exists. *You* (the consumer) can define an interface for the behavior you need, and existing types (even from the standard library or third parties) already satisfy it.
2. **Small, focused interfaces** become natural, since defining one costs nothing.
3. **Compile-time safety:** if a type *doesn't* satisfy the interface, the compiler tells you where you use it.

### Compile-time assertions

To *document and enforce* that a type implements an interface (especially in a library, where nothing else would check), use a blank-variable assertion:

```go
var _ Shape = (*Circle)(nil)   // compile error here if *Circle doesn't implement Shape
var _ Shape = Rectangle{}
```

`_` discards the value; the assignment is checked at compile time and costs nothing at run time.

---

## 5. Interface values and dynamic dispatch

An interface **value** (a variable of interface type) is a small **pair**:

```
interface value = ( type , value )
                    │       │
                    │       └── a copy of the concrete value (or a pointer to it)
                    └── which concrete type is stored (plus its method table)
```

Concretely, an interface value is **two words (16 bytes on 64-bit)**: a pointer to a type descriptor + method table (the "itab") and a pointer/word holding the data.

```go
var s Shape                // (nil, nil): the interface itself is nil
s = Circle{1}              // (Circle, Circle{R:1})
s = Rectangle{3, 4}        // (Rectangle, Rectangle{3,4})
```

Calling `s.Area()` is **dynamic dispatch**: at run time Go looks in the itab for the concrete type's `Area` method and calls it. Cost: an indirect call, typically a few nanoseconds; too small to worry about, but it prevents inlining. Storing a non-pointer value in an interface usually **allocates** (a copy is placed on the heap: escape analysis, Chapter 18), another reason not to sprinkle `any` in hot paths.

Verify the size on your machine:

```go
package main

import (
	"fmt"
	"unsafe"
)

type Shape interface{ Area() float64 }

func main() {
	var s Shape
	var e any
	fmt.Println(unsafe.Sizeof(s), unsafe.Sizeof(e)) // 16 16
}
```

---

## 6. Rules: complete methods, exact signatures

**Rule 1: implement every method.** Missing one means the type doesn't satisfy the interface:

```go
// INTENTIONAL ERROR
package main

type Shape interface {
	Area() float64
	Perimeter() float64
}

type Square struct{ S float64 }

func (s Square) Area() float64 { return s.S * s.S }

func main() {
	var sh Shape = Square{2}
	// ❌ cannot use Square{2} (value of type Square) as Shape value in variable declaration:
	//    Square does not implement Shape (missing method Perimeter)
	_ = sh
}
```

**Rule 2: signatures must match *exactly*** (parameter types, result types, order, count). Names of parameters don't matter, but types do:

```go
// INTENTIONAL ERROR
package main

type Shape interface{ Area() float64 }

type Square struct{ S float64 }

func (s Square) Area() int { return int(s.S * s.S) } // returns int, interface wants float64

func main() {
	var sh Shape = Square{2}
	// ❌ Square does not implement Shape (wrong type for method Area)
	//    have Area() int
	//    want Area() float64
	_ = sh
}
```

The compiler's message even prints `have` vs `want`. Read it carefully.

**Rule 3: a type may have more methods than the interface requires.** Extra methods are fine; the interface just doesn't expose them (you'd need a type assertion to reach them).

---

## 7. Pointer vs. value receivers and method sets

Chapter 22 introduced value receivers `(t T)` and pointer receivers `(t *T)`. They matter for interfaces:

> The **method set** of type `T` contains only methods with **value receivers**.
> The method set of type `*T` contains methods with **both** value and pointer receivers.

A type satisfies an interface only if its **method set** includes all the interface's methods. So:

```go
// INTENTIONAL ERROR
package main

import "fmt"

type Speaker interface{ Speak() string }

type Dog struct{ Name string }

func (d *Dog) Speak() string { return d.Name + " says woof" } // POINTER receiver

func main() {
	var s Speaker = Dog{"Rex"} // ❌ Dog does not implement Speaker (method Speak has pointer receiver)
	fmt.Println(s.Speak())
}
```

**Fix:** use a pointer, because `*Dog` has `Speak` in its method set:

```go
var s Speaker = &Dog{"Rex"} // ✅
```

| Method declared with | `T` satisfies? | `*T` satisfies? |
|----------------------|----------------|-----------------|
| Value receiver `(t T)` | ✅ | ✅ |
| Pointer receiver `(t *T)` | ❌ | ✅ |

**Why the asymmetry?** A pointer method may *modify* the value, but a `T` stored in an interface is a **copy**: calling a pointer method on it would modify only the copy, silently discarding the change. Go refuses to compile that rather than let you be surprised.

Practical rules:

- If a type has *any* pointer-receiver methods (it mutates state, holds a mutex, is large), give **all** its methods pointer receivers and always use `*T` (Chapter 22's consistency rule).
- Constructors for such types return `*T`: `NewProductStore()` returns `*ProductStore`, and that's the type that satisfies our `Store` interface.

---

## 8. The nil-interface gotcha

An interface value is `nil` only when **both** its type and value are nil. Storing a **nil pointer** inside an interface makes a **non-nil** interface:

```go
package main

import "fmt"

type MyError struct{}

func (e *MyError) Error() string { return "my error" }

func mayFail(fail bool) error {
	var err *MyError // a nil *MyError
	if fail {
		err = &MyError{}
	}
	return err // ⚠️ converts the *MyError (possibly nil) into an error interface
}

func main() {
	err := mayFail(false)
	fmt.Println(err == nil) // false  ← !!!
	fmt.Printf("%T %v\n", err, err == nil) // *main.MyError false
}
```

Real output:

```
false
*main.MyError false
```

`err` holds the pair `(type=*MyError, value=nil)`, which is **not** the nil interface `(nil, nil)`. So `if err != nil` is **true**, and the caller thinks an error occurred, then crashes calling methods on the nil pointer.

**The rule:** *never return a concrete pointer type from a function whose result type is an interface, unless you return the literal `nil` for the "no value" case.* Fix:

```go
func mayFail(fail bool) error {
	if fail {
		return &MyError{}
	}
	return nil // the literal nil interface
}
```

This bites in real code with custom error types. Prefer returning `error` directly and constructing the pointer only when needed.

---

## 9. The empty interface, type assertions, and type switches

### `any`: the interface with no methods

```go
type any = interface{}   // (since Go 1.18)
```

Every type has at least zero methods, so **every value satisfies `any`**. It's how `fmt.Println(a ...any)`, `json.Marshal(v any)`, and `[]any{1, "two", 3.0}` work.

```go
var x any = 42
x = "now a string"
x = []int{1, 2, 3}
```

The cost: **you lose static type information.** To use the value you must get the concrete type back.

### Type assertion

```go
var x any = "hello"

s := x.(string)          // asserts x holds a string; PANICS if it doesn't
s, ok := x.(string)      // ✅ safe form: ok is false instead of panicking
n, ok := x.(int)         // ok == false, n == 0
```

```go
if s, ok := x.(string); ok {
	fmt.Println("length:", len(s))
}
```

You can also assert to another **interface**, asking "does this value also have that behavior?":

```go
if st, ok := x.(fmt.Stringer); ok {
	fmt.Println(st.String())
}
```

### Type switch

```go
func describe(v any) string {
	switch t := v.(type) {
	case nil:
		return "nothing"
	case int, int64:
		return fmt.Sprintf("integer %v", t)
	case string:
		return "string of length " + strconv.Itoa(len(t))
	case []int:
		return fmt.Sprintf("slice of %d ints", len(t))
	case error:
		return "error: " + t.Error()
	case fmt.Stringer:
		return "stringer: " + t.String()
	default:
		return fmt.Sprintf("something else (%T)", t)
	}
}
```

In each case `t` has that case's type (`string` in the `string` case).

**When to use `any`?** Rarely in application code. It's appropriate at genuine boundaries (JSON, logging, reflection). For "a function that works on several types", prefer an **interface with the needed methods** or **generics** (Chapter 17's `Map`/`Filter`), which keep type safety. A signature like `func Process(data any) any` is usually a sign of missing design.

---

## 10. Small interfaces and the standard library

Go's most successful interfaces are tiny. Learn these; you'll meet them daily:

| Interface | Definition | Implemented by |
|-----------|------------|----------------|
| `error` | `Error() string` | every error value |
| `fmt.Stringer` | `String() string` | anything printable with `%v`/`%s` |
| `io.Reader` | `Read(p []byte) (n int, err error)` | files, network connections, `strings.Reader`, HTTP request bodies |
| `io.Writer` | `Write(p []byte) (n int, err error)` | files, `os.Stdout`, `http.ResponseWriter`, `bytes.Buffer` |
| `io.Closer` | `Close() error` | files, connections, response bodies |
| `sort.Interface` | `Len()`, `Less(i,j)`, `Swap(i,j)` | anything sortable |
| `http.Handler` | `ServeHTTP(w ResponseWriter, r *Request)` | your handlers, muxes, middleware results |
| `context.Context` | `Deadline`, `Done`, `Err`, `Value` | request contexts |

The power of small interfaces is **composition**: `io.Copy(dst Writer, src Reader)` copies between *any* pair; a file to the network, a string to a hash, an HTTP body to a file: all with one function.

```go
package main

import (
	"crypto/sha256"
	"fmt"
	"io"
	"strings"
)

func main() {
	src := strings.NewReader("hello") // an io.Reader
	dst := sha256.New()               // an io.Writer (a hash!)

	io.Copy(dst, src)                 // works because of the two tiny interfaces
	fmt.Printf("%x\n", dst.Sum(nil))
	// 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
}
```

You've been using interfaces all along: `http.ResponseWriter` (Chapter 45's `statusRecorder` wrapper worked *because* it satisfied that interface), `http.Handler` (all our middleware), `error`.

> **Go proverb:** *"The bigger the interface, the weaker the abstraction."* Prefer 1–3 methods.

---

## 11. Composing interfaces

Interfaces can **embed** other interfaces to build bigger ones:

```go
type Reader interface{ Read(p []byte) (int, error) }
type Writer interface{ Write(p []byte) (int, error) }

type ReadWriter interface { // must satisfy BOTH
	Reader
	Writer
}
```

The standard library does exactly this (`io.ReadWriter`, `io.ReadCloser`, `io.ReadWriteCloser`). Composition lets you ask for exactly the capabilities you need: a function that only reads should take `io.Reader`, not `*os.File`; then it works with files, network connections, strings, and test buffers alike.

---

## 12. Where should interfaces live?

Two guiding ideas, both counter to what Java/C# habits suggest:

### 1. Accept interfaces, return structs

```go
func NewHandler(store Store) *Handler      // ✅ takes the abstraction, returns the concrete type
func NewProductStore() *ProductStore       // ✅ returns the concrete type
```

Returning a concrete type gives callers everything it offers; accepting an interface gives *you* flexibility about what you're given.

### 2. Define the interface in the package that USES it (the consumer)

The `product` package needs "something that lists, gets, and creates products". So `product` declares that (small) interface, describing *only what product needs*. The `database` package doesn't import it and doesn't even know it exists; `*database.ProductStore` satisfies it implicitly.

```
        product package                      database package
  ┌──────────────────────────────┐      ┌─────────────────────────────┐
  │ type Store interface {...}   │      │ type ProductStore struct{}  │
  │ type Handler struct {        │      │ func (s *ProductStore) List │
  │     store Store   ◄──────────┼──────┼─ satisfies it implicitly    │
  │ }                            │      │ (no import of product!)     │
  └──────────────────────────────┘      └─────────────────────────────┘
        depends on an abstraction            knows nothing about product
```

This **inverts the dependency** (the "D" in SOLID): high-level policy (`product`) no longer depends on low-level details (`database`); both depend on the abstraction, which the high-level side owns.

The opposite habit (declare a big interface next to the implementation, "just in case") produces wide interfaces that force fakes to implement dozens of unneeded methods. **Keep interfaces small and near their users.**

---

## 13. Why interfaces? Five reasons

| # | Reason | Meaning here |
|---|--------|--------------|
| 1 | **Flexibility** | Swap the in-memory store for PostgreSQL by changing one line in the composition root |
| 2 | **Testability** | Hand a handler a *fake* store that returns whatever the test needs, including failures |
| 3 | **Decoupling** | `product` doesn't import `database`; the direction of dependencies flips toward stable abstractions |
| 4 | **Polymorphism** | One function, many concrete types (`io.Copy`, `sort.Sort`, `http.Handle`) |
| 5 | **Organization** | Interfaces document *what each part needs from its collaborators* |

---

## 14. Design patterns

> A **design pattern** is a *named, reusable solution* to a common design problem. Not code you copy, but a shape you recognize. They give teams a shared vocabulary ("let's use a strategy here").

Patterns come in families: **creational** (making objects), **structural** (arranging objects), **behavioral** (how objects interact). In Go many classic patterns shrink to a few lines thanks to interfaces, first-class functions, and closures.

### Strategy: swap the algorithm

*Problem:* a behavior varies (payment method, discount rule, sorting order). *Solution:* put each variant behind one interface; the caller picks one.

```go
package main

import "fmt"

type PaymentMethod interface {
	Pay(amount float64) string
}

type Card struct{ Last4 string }
type Wallet struct{ Balance float64 }

func (c Card) Pay(a float64) string { return fmt.Sprintf("charged %.2f to card ending %s", a, c.Last4) }
func (w Wallet) Pay(a float64) string {
	return fmt.Sprintf("paid %.2f from wallet (balance was %.2f)", a, w.Balance)
}

func checkout(total float64, method PaymentMethod) {
	fmt.Println(method.Pay(total)) // checkout never changes when we add a new payment method
}

func main() {
	checkout(99.5, Card{"4242"})
	checkout(20, Wallet{Balance: 100})
}
```

Adding "Bank transfer" means a new type; `checkout` stays untouched: the **Open/Closed principle** ("O" in SOLID). Since functions are values, a strategy can also just be a `func` type (Chapter 17's `Discount func(float64) float64`).

### Decorator: wrap to add behavior

*Problem:* add behavior (logging, caching, metrics, retries) without changing the original. *Solution:* a wrapper that implements the **same interface** and delegates. You have built these: **middleware** *is* the decorator pattern for `http.Handler`.

```go
package main

import (
	"fmt"
	"time"
)

type Fetcher interface{ Fetch(id int) string }

type Slow struct{}

func (Slow) Fetch(id int) string {
	time.Sleep(50 * time.Millisecond)
	return fmt.Sprintf("item-%d", id)
}

// Cached is a Fetcher that wraps another Fetcher.
type Cached struct {
	next  Fetcher
	cache map[int]string
}

func NewCached(next Fetcher) *Cached { return &Cached{next: next, cache: map[int]string{}} }

func (c *Cached) Fetch(id int) string {
	if v, ok := c.cache[id]; ok {
		return v
	}
	v := c.next.Fetch(id)
	c.cache[id] = v
	return v
}

func main() {
	var f Fetcher = NewCached(Slow{}) // callers see just a Fetcher
	start := time.Now()
	f.Fetch(1)
	first := time.Since(start)
	start = time.Now()
	f.Fetch(1)
	fmt.Println(first > 40*time.Millisecond, time.Since(start) < time.Millisecond) // true true
}
```

(A production cache also needs a mutex and eviction; the point here is the shape.)

### Factory: construct without exposing details

*Problem:* creating an object is non-trivial or the concrete type should be hidden. *Go's answer:* a `NewXxx` function; or a function returning different implementations behind an interface:

```go
func NewStore(kind string) (Store, error) {
	switch kind {
	case "memory":
		return database.NewProductStore(), nil
	case "postgres":
		return postgres.NewProductStore(dsn), nil
	}
	return nil, fmt.Errorf("unknown store kind %q", kind)
}
```

(Our `New...` constructors are the simple form of this pattern.)

### Adapter: make one interface look like another

*Problem:* a type has the right *behavior* but the wrong *shape*. *Solution:* wrap it. You've seen it: `http.HandlerFunc` **adapts a plain function to the `http.Handler` interface**:

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) } // a func with a method!
```

This is why `http.HandlerFunc(myFunc)` works: a *function type* can have methods (Chapter 22: methods on any named type), so a function can satisfy an interface. Elegant and tiny.

### Functional options: configurable constructors

*Problem:* constructors with many optional settings. *Go idiom:*

```go
type Server struct{ port int; timeout time.Duration }
type Option func(*Server)

func WithPort(p int) Option              { return func(s *Server) { s.port = p } }
func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }

func NewServer(opts ...Option) *Server {
	s := &Server{port: 8080, timeout: 10 * time.Second} // defaults
	for _, opt := range opts {
		opt(s)
	}
	return s
}

srv := NewServer(WithPort(9090)) // clear, extensible, backwards compatible
```

### Repository (preview of Chapters 59–61)

*Problem:* business logic shouldn't know how data is stored. *Solution:* an interface that speaks in domain terms (`Save`, `FindByID`), with implementations for each storage technology. **Our `Store` interfaces are exactly this pattern.**

### Singleton: one instance for the whole program

*Problem:* exactly one instance should exist (a config, a connection pool, a logger). Classic solution: a global accessor that creates the instance lazily and returns the same one forever. In Go, the safe, idiomatic way is **`sync.Once`**:

```go
package main

import (
	"fmt"
	"sync"
)

type Config struct{ Port string }

var (
	instance *Config
	once     sync.Once
)

// GetConfig returns the single shared Config, creating it on first use.
// once.Do runs its function exactly once, even if many goroutines call GetConfig simultaneously.
func GetConfig() *Config {
	once.Do(func() {
		fmt.Println("loading config (only printed once)")
		instance = &Config{Port: "8080"}
	})
	return instance
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() { defer wg.Done(); _ = GetConfig() }()
	}
	wg.Wait()
	fmt.Println(GetConfig() == GetConfig()) // true: the same pointer
}
```

Output: `loading config (only printed once)` **once**, then `true`.

**Without `sync.Once`**, a naive `if instance == nil { instance = &Config{} }` is a **data race** (Chapter 67): two goroutines can both see `nil` and both create an instance.

**Should you use it?** A singleton is a **global with a nice name**, and inherits every downside from Chapter 50: hidden dependencies, untestable code, shared mutable state. In Go, prefer **one instance created in `main` and passed down (dependency injection)**, which gives "one instance" without the global. Reserve `sync.Once` for lazy initialization of genuinely process-wide, read-only resources (a compiled regex, a lazily opened resource). That's why this course's `Config` is loaded once in `main` and *passed* rather than exposed through `config.GetConfig()`.

### Choosing patterns

Don't go pattern-hunting. Write the simple code; when you feel a specific pain (a growing `switch`, duplicated wrapping logic, hard-to-test code), the matching pattern is usually obvious. The Go community's bias: **simple, explicit, small**.

---

## 15. Applying it to the project

Currently (Chapter 50) `product.Handler` holds a `*database.ProductStore`. We'll change three things:

### 1. Define what the feature needs, in the feature

```go
// file: product/store.go
package product

import (
	"context"
	"ecommerce/models"
)

// Store is what the product feature needs from a storage backend.
// It is defined here, next to its only user, and is as small as it can be.
type Store interface {
	List(ctx context.Context) ([]models.Product, error)
	// Get returns models.ErrNotFound if there is no product with that ID.
	Get(ctx context.Context, id int) (models.Product, error)
	Create(ctx context.Context, p models.Product) (models.Product, error)
}
```

Notice two design decisions that the in-memory store doesn't *need* but real storage does:

- **`context.Context` as the first parameter**: a database call can be slow; the context lets it stop when the client disconnects or a deadline passes.
- **Every method returns an `error`**: network and disk operations *fail*. An interface designed around an in-memory slice ("`List() []Product`", no error) would have to change the moment PostgreSQL appears, and every caller with it. Design the abstraction for the *real* dependency, not the convenient one.
- Missing data is reported with a **sentinel error** (`models.ErrNotFound`), so callers can tell "no such product" (→ `404`) from "the database exploded" (→ `500`) using `errors.Is`.

```go
// file: user/store.go
package user

import (
	"context"
	"ecommerce/models"
)

// Store is what the user feature needs from a storage backend.
type Store interface {
	// Create returns models.ErrEmailTaken if the email is already registered.
	Create(ctx context.Context, email, passwordHash string) (models.User, error)
	// FindByEmail and FindByID return models.ErrNotFound if there is no such user.
	FindByEmail(ctx context.Context, email string) (models.User, error)
	FindByID(ctx context.Context, id int) (models.User, error)
}
```

### 2. Shared sentinel errors

The sentinel errors need a home that both `database` (which returns them) and the features (which test for them) can import without depending on each other: the tiny, dependency-free `models` package.

```go
// file: models/errors.go
package models

import "errors"

var (
	// ErrNotFound is returned by stores when the requested record does not exist.
	ErrNotFound = errors.New("not found")

	// ErrEmailTaken is returned when registering an email that is already in use.
	ErrEmailTaken = errors.New("email already registered")
)
```

(That replaces `database.ErrEmailTaken`.)

### 3. A shared request-ID helper, and a place to log server errors

When a store fails, the handler must (a) tell the client only `internal server error`, and (b) **log the real cause** with the request ID for correlation. Handlers need `RequestIDFrom`, which currently lives in `middleware`, but `middleware` imports `util`, so `util` can't import `middleware` (a cycle). The fix is the standard one: **move the shared piece down** into a tiny leaf package that both can import.

```go
// file: reqid/reqid.go
package reqid

import "context"

// key is private, so no other package can create a colliding context key.
type key struct{}

// NewContext returns a copy of ctx that carries the request ID.
func NewContext(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, key{}, id)
}

// FromContext returns the request ID carried by ctx, or "" if there is none.
func FromContext(ctx context.Context) string {
	id, _ := ctx.Value(key{}).(string)
	return id
}
```

```go
// file: middleware/requestid.go
package middleware

import (
	"context"
	"crypto/rand"
	"ecommerce/reqid"
	"encoding/hex"
	"net/http"
)

// RequestIDHeader is the header used to accept and return the request ID.
const RequestIDHeader = "X-Request-ID"

// RequestID gives every request a unique ID, stored in the request context and echoed in the response header.
// If the client (or an upstream proxy) sent a well-formed X-Request-ID, it is reused so IDs can be traced across services.
func RequestID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get(RequestIDHeader)
		if !validRequestID(id) {
			id = newRequestID()
		}

		w.Header().Set(RequestIDHeader, id)
		next.ServeHTTP(w, r.WithContext(reqid.NewContext(r.Context(), id)))
	})
}

// RequestIDFrom returns the request ID stored in ctx, or "" if there is none.
// (Kept for callers that already use it; it simply delegates to package reqid.)
func RequestIDFrom(ctx context.Context) string { return reqid.FromContext(ctx) }

func newRequestID() string {
	var b [8]byte
	if _, err := rand.Read(b[:]); err != nil {
		return "unknown"
	}
	return hex.EncodeToString(b[:]) // 16 hex characters
}

// validRequestID accepts short IDs made of safe characters, so a client can't inject
// newlines or huge strings into our logs (log injection).
func validRequestID(id string) bool {
	if id == "" || len(id) > 64 {
		return false
	}
	for _, c := range id {
		switch {
		case c >= 'a' && c <= 'z', c >= 'A' && c <= 'Z', c >= '0' && c <= '9', c == '-', c == '_', c == '.':
		default:
			return false
		}
	}
	return true
}
```

The behavior and exported names are unchanged, so every existing middleware test still passes. Now a helper for the "log it, hide it" pattern, placed in `util` beside the other response helpers:

```go
// file: util/servererror.go
package util

import (
	"ecommerce/reqid"
	"log/slog"
	"net/http"
)

// ServerError logs the real cause of an unexpected failure (with the request ID) and
// answers the client with a generic 500, never leaking internal details.
func ServerError(w http.ResponseWriter, r *http.Request, logger *slog.Logger, err error) {
	logger.ErrorContext(r.Context(), "request failed",
		"err", err,
		"method", r.Method,
		"path", r.URL.Path,
		"request_id", reqid.FromContext(r.Context()),
	)
	SendError(w, http.StatusInternalServerError, "internal server error")
}
```

---

## 16. The complete code

### The concrete stores now satisfy the interfaces

The in-memory stores gain the `context` parameter and `error` results. They ignore the context (nothing slow happens); a PostgreSQL store will honor it.

```go
// file: database/product_store.go
package database

import (
	"context"
	"ecommerce/models"
	"sync"
)

// ProductStore keeps products in memory. It is safe for concurrent use.
type ProductStore struct {
	mu       sync.RWMutex
	products []models.Product
	nextID   int
}

// NewProductStore creates a store, optionally pre-loaded with seed products.
// New products get IDs after the highest seeded ID.
func NewProductStore(seed ...models.Product) *ProductStore {
	s := &ProductStore{nextID: 1}
	for _, p := range seed {
		s.products = append(s.products, p)
		if p.ID >= s.nextID {
			s.nextID = p.ID + 1
		}
	}
	return s
}

// SampleProducts returns the demo data used when the app starts.
func SampleProducts() []models.Product {
	return []models.Product{
		{ID: 1, Title: "Orange", Description: "Orange is juicy and full of vitamin C.", Price: 100, ImageURL: "https://example.com/orange.jpg"},
		{ID: 2, Title: "Apple", Description: "A crunchy apple a day...", Price: 40, ImageURL: "https://example.com/apple.jpg"},
		{ID: 3, Title: "Banana", Description: "Great for a quick snack.", Price: 5, ImageURL: "https://example.com/banana.jpg"},
	}
}

// List returns a copy of all products.
func (s *ProductStore) List(_ context.Context) ([]models.Product, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	out := make([]models.Product, len(s.products))
	copy(out, s.products)
	return out, nil
}

// Get returns the product with the given ID, or models.ErrNotFound.
func (s *ProductStore) Get(_ context.Context, id int) (models.Product, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, p := range s.products {
		if p.ID == id {
			return p, nil
		}
	}
	return models.Product{}, models.ErrNotFound
}

// Create assigns the next ID to p, stores it, and returns the stored product.
func (s *ProductStore) Create(_ context.Context, p models.Product) (models.Product, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	p.ID = s.nextID
	s.nextID++
	s.products = append(s.products, p)
	return p, nil
}
```

```go
// file: database/user_store.go
package database

import (
	"context"
	"ecommerce/models"
	"strings"
	"sync"
	"time"
)

// UserStore keeps users in memory. It is safe for concurrent use.
type UserStore struct {
	mu     sync.RWMutex
	users  []models.User
	nextID int
}

// NewUserStore creates an empty user store.
func NewUserStore() *UserStore { return &UserStore{nextID: 1} }

func normalizeEmail(email string) string { return strings.ToLower(strings.TrimSpace(email)) }

// Create stores a new user, or returns models.ErrEmailTaken. The duplicate check and the insert
// share one lock, so two simultaneous registrations of the same email cannot both succeed.
func (s *UserStore) Create(_ context.Context, email, passwordHash string) (models.User, error) {
	email = normalizeEmail(email)

	s.mu.Lock()
	defer s.mu.Unlock()

	for _, u := range s.users {
		if u.Email == email {
			return models.User{}, models.ErrEmailTaken
		}
	}
	u := models.User{ID: s.nextID, Email: email, PasswordHash: passwordHash, CreatedAt: time.Now().UTC()}
	s.nextID++
	s.users = append(s.users, u)
	return u, nil
}

// FindByEmail looks a user up by (case-insensitive) email, or returns models.ErrNotFound.
func (s *UserStore) FindByEmail(_ context.Context, email string) (models.User, error) {
	email = normalizeEmail(email)

	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, u := range s.users {
		if u.Email == email {
			return u, nil
		}
	}
	return models.User{}, models.ErrNotFound
}

// FindByID looks a user up by ID, or returns models.ErrNotFound.
func (s *UserStore) FindByID(_ context.Context, id int) (models.User, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	for _, u := range s.users {
		if u.ID == id {
			return u, nil
		}
	}
	return models.User{}, models.ErrNotFound
}
```

### The product feature depends on the interface

```go
// file: product/handler.go
package product

import (
	"log/slog"
	"net/http"
)

// Handler serves the product endpoints. It depends only on the Store interface, never on a concrete database type.
type Handler struct {
	store  Store
	logger *slog.Logger
}

// NewHandler creates a Handler. Accept interfaces, return structs.
func NewHandler(store Store, logger *slog.Logger) *Handler {
	return &Handler{store: store, logger: logger}
}

// Routes registers this feature's endpoints on mux.
// authn is applied to the routes that require a logged-in user.
func (h *Handler) Routes(mux *http.ServeMux, authn func(http.Handler) http.Handler) {
	mux.HandleFunc("GET /products", h.List)
	mux.HandleFunc("GET /products/{id}", h.Get)
	mux.Handle("POST /products", authn(http.HandlerFunc(h.Create)))
}
```

```go
// file: product/list.go
package product

import (
	"ecommerce/util"
	"net/http"
)

// List handles GET /products.
func (h *Handler) List(w http.ResponseWriter, r *http.Request) {
	products, err := h.store.List(r.Context())
	if err != nil {
		util.ServerError(w, r, h.logger, err)
		return
	}
	util.SendData(w, http.StatusOK, products)
}
```

```go
// file: product/get.go
package product

import (
	"ecommerce/models"
	"ecommerce/util"
	"errors"
	"net/http"
	"strconv"
)

// Get handles GET /products/{id}.
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id < 1 {
		util.SendError(w, http.StatusBadRequest, "id must be a positive integer")
		return
	}

	p, err := h.store.Get(r.Context(), id)
	switch {
	case errors.Is(err, models.ErrNotFound):
		util.SendError(w, http.StatusNotFound, "product not found")
	case err != nil:
		util.ServerError(w, r, h.logger, err) // unexpected: log the cause, hide it from the client
	default:
		util.SendData(w, http.StatusOK, p)
	}
}
```

```go
// file: product/create.go
package product

import (
	"ecommerce/models"
	"ecommerce/util"
	"fmt"
	"net/http"
	"strings"
)

// createRequest is what a client may send to create a product.
// It deliberately has no ID: the server assigns it.
type createRequest struct {
	Title       string  `json:"title"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
	ImageURL    string  `json:"imageUrl"`
}

func (r createRequest) validate() string {
	switch {
	case strings.TrimSpace(r.Title) == "":
		return "title is required"
	case len(r.Title) > 100:
		return "title must be at most 100 characters"
	case r.Price <= 0:
		return "price must be greater than 0"
	}
	return ""
}

// Create handles POST /products. The route requires authentication (see Routes).
func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
	var req createRequest
	if !util.DecodeJSON(w, r, &req) {
		return
	}
	if msg := req.validate(); msg != "" {
		util.SendError(w, http.StatusUnprocessableEntity, msg)
		return
	}

	p, err := h.store.Create(r.Context(), models.Product{
		Title:       strings.TrimSpace(req.Title),
		Description: req.Description,
		Price:       req.Price,
		ImageURL:    req.ImageURL,
	})
	if err != nil {
		util.ServerError(w, r, h.logger, err)
		return
	}

	w.Header().Set("Location", fmt.Sprintf("/products/%d", p.ID))
	util.SendData(w, http.StatusCreated, p)
}
```

### The user feature, likewise

```go
// file: user/handler.go
package user

import (
	"ecommerce/auth"
	"log/slog"
	"net/http"
	"time"
)

// Handler serves registration, login, and the current-user endpoint.
// It depends on the Store interface, never on a concrete database type.
type Handler struct {
	store     Store
	jwtSecret []byte
	jwtTTL    time.Duration
	logger    *slog.Logger

	// dummyHash lets Login spend the same time on unknown emails as on known ones.
	dummyHash string
}

// NewHandler creates a user Handler. It takes only what it needs.
func NewHandler(store Store, jwtSecret []byte, jwtTTL time.Duration, logger *slog.Logger) *Handler {
	dummy, _ := auth.HashPassword("dummy-password-for-timing")
	return &Handler{store: store, jwtSecret: jwtSecret, jwtTTL: jwtTTL, logger: logger, dummyHash: dummy}
}

// Routes registers this feature's endpoints on mux.
func (h *Handler) Routes(mux *http.ServeMux, authn func(http.Handler) http.Handler) {
	mux.HandleFunc("POST /users", h.Register)
	mux.HandleFunc("POST /login", h.Login)
	mux.Handle("GET /me", authn(http.HandlerFunc(h.Me)))
}
```

```go
// file: user/register.go
package user

import (
	"ecommerce/auth"
	"ecommerce/models"
	"ecommerce/util"
	"errors"
	"net/http"
	"net/mail"
)

const (
	minPasswordLength = 8
	maxPasswordLength = 72 // bcrypt only uses the first 72 bytes
)

type credentials struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

func (c credentials) validateForRegistration() string {
	addr, err := mail.ParseAddress(c.Email)
	if err != nil || addr.Address != c.Email {
		return "email must be a valid address like name@example.com"
	}
	if len(c.Password) < minPasswordLength {
		return "password must be at least 8 characters"
	}
	if len(c.Password) > maxPasswordLength {
		return "password must be at most 72 bytes"
	}
	return ""
}

// Register handles POST /users.
func (h *Handler) Register(w http.ResponseWriter, r *http.Request) {
	var req credentials
	if !util.DecodeJSON(w, r, &req) {
		return
	}
	if msg := req.validateForRegistration(); msg != "" {
		util.SendError(w, http.StatusUnprocessableEntity, msg)
		return
	}

	hash, err := auth.HashPassword(req.Password)
	if err != nil {
		util.ServerError(w, r, h.logger, err)
		return
	}

	u, err := h.store.Create(r.Context(), req.Email, hash)
	switch {
	case errors.Is(err, models.ErrEmailTaken):
		util.SendError(w, http.StatusConflict, "email already registered")
	case err != nil:
		util.ServerError(w, r, h.logger, err)
	default:
		util.SendData(w, http.StatusCreated, u) // the password hash is excluded by its json:"-" tag
	}
}
```

```go
// file: user/login.go
package user

import (
	"ecommerce/auth"
	"ecommerce/models"
	"ecommerce/util"
	"errors"
	"net/http"
	"time"
)

// Login handles POST /login: it verifies credentials and returns a signed token.
func (h *Handler) Login(w http.ResponseWriter, r *http.Request) {
	var req credentials
	if !util.DecodeJSON(w, r, &req) {
		return
	}

	u, err := h.store.FindByEmail(r.Context(), req.Email)
	found := err == nil
	if err != nil && !errors.Is(err, models.ErrNotFound) {
		util.ServerError(w, r, h.logger, err) // a genuine storage failure, not "no such user"
		return
	}

	// Always run a bcrypt comparison, even for unknown emails, so timing doesn't reveal which exist.
	hash := h.dummyHash
	if found {
		hash = u.PasswordHash
	}
	passwordOK := auth.CheckPassword(hash, req.Password)

	if !found || !passwordOK {
		util.SendError(w, http.StatusUnauthorized, "invalid email or password")
		return
	}

	token, err := auth.CreateToken(h.jwtSecret, u.ID, h.jwtTTL, time.Now())
	if err != nil {
		util.ServerError(w, r, h.logger, err)
		return
	}
	util.SendData(w, http.StatusOK, map[string]string{"token": token})
}
```

```go
// file: user/me.go
package user

import (
	"ecommerce/auth"
	"ecommerce/models"
	"ecommerce/util"
	"errors"
	"net/http"
)

// Me handles GET /me. The route is wrapped in the Authenticate middleware (see Routes),
// which is what puts the caller's ID into the request context.
func (h *Handler) Me(w http.ResponseWriter, r *http.Request) {
	id, ok := auth.UserIDFrom(r.Context())
	if !ok { // only possible if the route was registered without authentication: a programming error
		util.SendError(w, http.StatusUnauthorized, "authentication required")
		return
	}

	u, err := h.store.FindByID(r.Context(), id)
	switch {
	case errors.Is(err, models.ErrNotFound): // a valid token for an account that no longer exists
		util.SendError(w, http.StatusUnauthorized, "account not found")
	case err != nil:
		util.ServerError(w, r, h.logger, err)
	default:
		util.SendData(w, http.StatusOK, u)
	}
}
```

### The composition root

Only the two constructor calls change, and **that's the whole point**: everything else already talked to interfaces.

```go
// file: cmd/wire.go
package cmd

import (
	"ecommerce/config"
	"ecommerce/database"
	"ecommerce/product"
	"ecommerce/rest"
	"ecommerce/user"
	"log/slog"
	"net/http"
)

// buildHandler is the composition root: it creates every component, hands each one
// exactly the dependencies it needs, and returns the finished HTTP handler.
func buildHandler(cfg *config.Config, logger *slog.Logger) http.Handler {
	// storage: concrete types, chosen HERE and nowhere else
	productStore := database.NewProductStore(database.SampleProducts()...)
	userStore := database.NewUserStore()

	// features receive them as interfaces; if *ProductStore didn't satisfy product.Store,
	// the compiler would reject the next line, which doubles as a compile-time assertion
	productHandler := product.NewHandler(productStore, logger)
	userHandler := user.NewHandler(userStore, []byte(cfg.JWTSecret), cfg.JWTTTL, logger)

	return rest.NewServer(cfg, logger, productHandler, userHandler).Handler()
}
```

When Chapter 52 adds PostgreSQL, the change is confined to these two storage lines (`postgres.NewProductStore(db)` instead of `database.NewProductStore(...)`): `product`, `user`, `rest`, and `middleware` remain untouched.

---

## 17. Tests: fakes and failure paths

### The updated app helper

The `rest` tests only need their constructor calls updated (everything else, `server_test.go` included, is unchanged):

```go
// file: rest/app_test.go
package rest

import (
	"ecommerce/auth"
	"ecommerce/config"
	"ecommerce/database"
	"ecommerce/product"
	"ecommerce/user"
	"encoding/json"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

var quietLogger = slog.New(slog.NewTextHandler(io.Discard, nil))

// testApp is a complete, private instance of the application.
type testApp struct {
	t        *testing.T
	cfg      *config.Config
	handler  http.Handler
	products *database.ProductStore
	users    *database.UserStore
}

func newApp(t *testing.T) *testApp {
	t.Helper()
	cfg := &config.Config{
		Env:            "test",
		Port:           "0",
		AllowedOrigins: []string{"http://localhost:5173"},
		LogFormat:      "text",
		JWTSecret:      "0123456789abcdef0123456789abcdef0123",
		JWTTTL:         time.Hour,
	}
	products := database.NewProductStore(database.SampleProducts()...)
	users := database.NewUserStore()

	server := NewServer(cfg, quietLogger,
		product.NewHandler(products, quietLogger),
		user.NewHandler(users, []byte(cfg.JWTSecret), cfg.JWTTTL, quietLogger),
	)
	return &testApp{t: t, cfg: cfg, handler: server.Handler(), products: products, users: users}
}

func (a *testApp) do(method, target, body string, headers map[string]string) *httptest.ResponseRecorder {
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	if body != "" {
		req.Header.Set("Content-Type", "application/json")
	}
	for k, v := range headers {
		req.Header.Set(k, v)
	}
	rec := httptest.NewRecorder()
	a.handler.ServeHTTP(rec, req)
	return rec
}

// bearer returns headers carrying a valid token for the given user ID.
func (a *testApp) bearer(userID int) map[string]string {
	a.t.Helper()
	token, err := auth.CreateToken([]byte(a.cfg.JWTSecret), userID, time.Hour, time.Now())
	if err != nil {
		a.t.Fatal(err)
	}
	return map[string]string{"Authorization": "Bearer " + token}
}

func (a *testApp) count() int {
	var list []struct{ ID int }
	json.NewDecoder(a.do(http.MethodGet, "/products", "", nil).Body).Decode(&list)
	return len(list)
}
```

### A fake store: testing what couldn't be tested before

Until now we could only test the *happy* storage paths, because the in-memory store never fails. With an interface we can supply a **fake** that fails on demand, and verify the handler's *error handling*:

```go
// file: product/handler_test.go
package product

import (
	"bytes"
	"context"
	"ecommerce/database"
	"ecommerce/models"
	"errors"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

var _ Store = (*database.ProductStore)(nil) // the real store satisfies the interface (compile-time check)
var _ Store = (*failingStore)(nil)          // ...and so does our fake

// failingStore is a Store whose every operation fails with err.
type failingStore struct{ err error }

func (f *failingStore) List(context.Context) ([]models.Product, error) { return nil, f.err }
func (f *failingStore) Get(context.Context, int) (models.Product, error) {
	return models.Product{}, f.err
}
func (f *failingStore) Create(context.Context, models.Product) (models.Product, error) {
	return models.Product{}, f.err
}

func newLogger() (*slog.Logger, *bytes.Buffer) {
	var buf bytes.Buffer
	return slog.New(slog.NewTextHandler(&buf, nil)), &buf
}

func TestStoreFailuresBecomeGeneric500sAndAreLogged(t *testing.T) {
	boom := errors.New("connection to database lost: password=hunter2")

	tests := []struct {
		name string
		call func(h *Handler, w http.ResponseWriter)
	}{
		{"list", func(h *Handler, w http.ResponseWriter) {
			h.List(w, httptest.NewRequest(http.MethodGet, "/products", nil))
		}},
		{"get", func(h *Handler, w http.ResponseWriter) {
			r := httptest.NewRequest(http.MethodGet, "/products/1", nil)
			r.SetPathValue("id", "1")
			h.Get(w, r)
		}},
		{"create", func(h *Handler, w http.ResponseWriter) {
			h.Create(w, httptest.NewRequest(http.MethodPost, "/products", strings.NewReader(`{"title":"x","price":1}`)))
		}},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			logger, logs := newLogger()
			h := NewHandler(&failingStore{err: boom}, logger)

			rec := httptest.NewRecorder()
			tc.call(h, rec)

			if rec.Code != http.StatusInternalServerError {
				t.Fatalf("expected 500, got %d", rec.Code)
			}
			if strings.Contains(rec.Body.String(), "hunter2") || strings.Contains(rec.Body.String(), "database") {
				t.Errorf("internal details leaked to the client: %s", rec.Body.String())
			}
			if !strings.Contains(rec.Body.String(), "internal server error") {
				t.Errorf("expected a generic message, got %s", rec.Body.String())
			}
			if !strings.Contains(logs.String(), "connection to database lost") {
				t.Errorf("the real cause must be logged, got %q", logs.String())
			}
		})
	}
}

func TestNotFoundIsA404NotA500(t *testing.T) {
	logger, logs := newLogger()
	h := NewHandler(&failingStore{err: models.ErrNotFound}, logger)

	rec := httptest.NewRecorder()
	r := httptest.NewRequest(http.MethodGet, "/products/7", nil)
	r.SetPathValue("id", "7")
	h.Get(rec, r)

	if rec.Code != http.StatusNotFound {
		t.Fatalf("expected 404 for ErrNotFound, got %d", rec.Code)
	}
	if logs.Len() != 0 {
		t.Errorf("a missing record is normal and must not be logged as an error: %q", logs.String())
	}
}

func TestWrappedNotFoundStillCounts(t *testing.T) {
	// errors.Is sees through wrapping, so a store may add context to the sentinel.
	wrapped := errors.Join(errors.New("query products"), models.ErrNotFound)
	h := NewHandler(&failingStore{err: wrapped}, slog.New(slog.NewTextHandler(&bytes.Buffer{}, nil)))

	rec := httptest.NewRecorder()
	r := httptest.NewRequest(http.MethodGet, "/products/7", nil)
	r.SetPathValue("id", "7")
	h.Get(rec, r)

	if rec.Code != http.StatusNotFound {
		t.Errorf("expected 404, got %d", rec.Code)
	}
}

func TestCreateValidationLeavesTheStoreUntouched(t *testing.T) {
	tests := []struct {
		name string
		body string
		want int
	}{
		{"empty body", ``, http.StatusBadRequest},
		{"truncated json", `{"title": `, http.StatusBadRequest},
		{"wrong type", `{"title":"x","price":"cheap"}`, http.StatusBadRequest},
		{"unknown field", `{"id":9,"title":"x","price":1}`, http.StatusBadRequest},
		{"missing title", `{"price":5}`, http.StatusUnprocessableEntity},
		{"blank title", `{"title":"   ","price":5}`, http.StatusUnprocessableEntity},
		{"negative price", `{"title":"x","price":-1}`, http.StatusUnprocessableEntity},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			t.Parallel()
			store := database.NewProductStore()
			logger, _ := newLogger()
			h := NewHandler(store, logger)

			rec := httptest.NewRecorder()
			h.Create(rec, httptest.NewRequest(http.MethodPost, "/products", strings.NewReader(tc.body)))

			if rec.Code != tc.want {
				t.Errorf("expected %d, got %d (%s)", tc.want, rec.Code, rec.Body.String())
			}
			if products, _ := store.List(context.Background()); len(products) != 0 {
				t.Errorf("an invalid request stored %d products", len(products))
			}
		})
	}
}

func TestCreateStoresTheProductAndAssignsAnID(t *testing.T) {
	store := database.NewProductStore(models.Product{ID: 41, Title: "Existing", Price: 1})
	logger, _ := newLogger()
	h := NewHandler(store, logger)

	rec := httptest.NewRecorder()
	h.Create(rec, httptest.NewRequest(http.MethodPost, "/products",
		strings.NewReader(`{"title":"  New thing  ","description":"d","price":9.5}`)))

	if rec.Code != http.StatusCreated {
		t.Fatalf("expected 201, got %d (%s)", rec.Code, rec.Body.String())
	}
	if loc := rec.Header().Get("Location"); loc != "/products/42" {
		t.Errorf("IDs continue after the highest seeded ID: Location = %q", loc)
	}
	got, err := store.Get(context.Background(), 42)
	if err != nil || got.Title != "New thing" { // the title was trimmed
		t.Errorf("stored product = %+v (err=%v)", got, err)
	}
}
```

Points to notice:

- **Compile-time assertions** (`var _ Store = ...`) prove both the real store and the fake satisfy the interface. If someone later changes the interface, these lines fail to compile immediately.
- `r.SetPathValue("id", "1")` lets a test call a handler *directly*, setting the path parameter the mux would normally provide.
- The failure test asserts **two things at once**: the client sees only a generic message (no `hunter2`), *and* the real cause is in the logs. That's a security property verified by a test, possible only because a fake could inject the failure.
- `errors.Join(...)`/wrapping still matches `errors.Is(err, models.ErrNotFound)`: sentinel checks see through wrapping (Chapter 41).

### A user-side fake

```go
// file: user/handler_test.go
package user

import (
	"bytes"
	"context"
	"ecommerce/models"
	"errors"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

// brokenStore fails every call, except that it can be told to report a missing user.
type brokenStore struct{ err error }

func (b *brokenStore) Create(context.Context, string, string) (models.User, error) {
	return models.User{}, b.err
}
func (b *brokenStore) FindByEmail(context.Context, string) (models.User, error) {
	return models.User{}, b.err
}
func (b *brokenStore) FindByID(context.Context, int) (models.User, error) {
	return models.User{}, b.err
}

func newHandler(store Store) (*Handler, *bytes.Buffer) {
	var logs bytes.Buffer
	logger := slog.New(slog.NewTextHandler(&logs, nil))
	return NewHandler(store, []byte("0123456789abcdef0123456789abcdef0123"), time.Hour, logger), &logs
}

func post(h http.HandlerFunc, body string) *httptest.ResponseRecorder {
	rec := httptest.NewRecorder()
	h(rec, httptest.NewRequest(http.MethodPost, "/", strings.NewReader(body)))
	return rec
}

func TestLoginDistinguishesMissingUsersFromBrokenStorage(t *testing.T) {
	const body = `{"email":"a@b.co","password":"whatever-it-is"}`

	// no such user → a normal 401, indistinguishable from a wrong password
	h, logs := newHandler(&brokenStore{err: models.ErrNotFound})
	if rec := post(h.Login, body); rec.Code != http.StatusUnauthorized || logs.Len() != 0 {
		t.Errorf("unknown user: code=%d logs=%q", rec.Code, logs.String())
	}

	// broken storage → 500 (not a misleading 401), and the cause is logged
	h, logs = newHandler(&brokenStore{err: errors.New("disk on fire")})
	rec := post(h.Login, body)
	if rec.Code != http.StatusInternalServerError {
		t.Fatalf("expected 500 when the store fails, got %d", rec.Code)
	}
	if !strings.Contains(logs.String(), "disk on fire") || strings.Contains(rec.Body.String(), "fire") {
		t.Errorf("cause must be logged and not returned: body=%q logs=%q", rec.Body.String(), logs.String())
	}
}

func TestRegisterMapsEmailTakenTo409(t *testing.T) {
	h, _ := newHandler(&brokenStore{err: models.ErrEmailTaken})
	rec := post(h.Register, `{"email":"a@b.co","password":"long-enough-password"}`)
	if rec.Code != http.StatusConflict {
		t.Errorf("expected 409, got %d", rec.Code)
	}
}
```

### The store tests, updated for the new signatures

```go
// file: database/stores_test.go
package database

import (
	"context"
	"ecommerce/models"
	"errors"
	"sync"
	"testing"
)

var ctx = context.Background()

func TestUserStoreRejectsDuplicatesUnderConcurrency(t *testing.T) {
	s := NewUserStore()

	const attempts = 50
	var wg sync.WaitGroup
	var mu sync.Mutex
	successes := 0

	for i := 0; i < attempts; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			_, err := s.Create(ctx, "  ASHA@Example.com ", "hash") // sloppy spacing and capitals
			if err == nil {
				mu.Lock()
				successes++
				mu.Unlock()
			} else if !errors.Is(err, models.ErrEmailTaken) {
				t.Errorf("unexpected error: %v", err)
			}
		}()
	}
	wg.Wait()

	if successes != 1 {
		t.Fatalf("exactly one registration must win, got %d", successes)
	}
}

func TestUserStoreLookups(t *testing.T) {
	s := NewUserStore()
	u, _ := s.Create(ctx, "asha@example.com", "hash")

	if got, err := s.FindByEmail(ctx, " ASHA@example.com"); err != nil || got.ID != u.ID {
		t.Errorf("FindByEmail: %+v %v", got, err)
	}
	if got, err := s.FindByID(ctx, u.ID); err != nil || got.Email != "asha@example.com" {
		t.Errorf("FindByID: %+v %v", got, err)
	}
	if _, err := s.FindByEmail(ctx, "ghost@example.com"); !errors.Is(err, models.ErrNotFound) {
		t.Errorf("expected ErrNotFound, got %v", err)
	}
	if _, err := s.FindByID(ctx, 999); !errors.Is(err, models.ErrNotFound) {
		t.Errorf("expected ErrNotFound, got %v", err)
	}
}

func TestProductStoreIDsAreUniqueAndSequential(t *testing.T) {
	s := NewProductStore()
	const n = 100

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			s.Create(ctx, SampleProducts()[0])
		}()
	}
	wg.Wait()

	list, _ := s.List(ctx)
	if len(list) != n {
		t.Fatalf("expected %d products, got %d", n, len(list))
	}
	seen := map[int]bool{}
	for _, p := range list {
		if p.ID < 1 || p.ID > n || seen[p.ID] {
			t.Fatalf("bad or duplicate ID %d", p.ID)
		}
		seen[p.ID] = true
	}
}

func TestProductGetReportsNotFound(t *testing.T) {
	s := NewProductStore(SampleProducts()...)
	if _, err := s.Get(ctx, 2); err != nil {
		t.Errorf("product 2 exists: %v", err)
	}
	if _, err := s.Get(ctx, 99); !errors.Is(err, models.ErrNotFound) {
		t.Errorf("expected ErrNotFound, got %v", err)
	}
}

func TestListReturnsACopy(t *testing.T) {
	s := NewProductStore(SampleProducts()...)
	list, _ := s.List(ctx)
	list[0].Title = "tampered"

	if got, _ := s.Get(ctx, 1); got.Title == "tampered" {
		t.Error("callers must not be able to modify the store's internal slice")
	}
}
```

Run the whole suite:

```bash
go vet ./... && go test -race ./...
```

---

## 18. Common mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Returning a nil *concrete pointer* as an interface (e.g., `var e *MyErr; return e`) | `err != nil` is true (typed-nil trap) | Return the literal `nil` |
| 2 | A value type where a pointer receiver is required | `does not implement (method has pointer receiver)` | Use `&T{}` / `*T` |
| 3 | Huge interfaces (10+ methods) | Hard to implement and to fake | Split into small, focused interfaces |
| 4 | Declaring interfaces next to the implementation "just in case" | Consumers depend on more than they need | Define interfaces where they're used |
| 5 | Returning interfaces from constructors (`func New() Store`) | Callers lose access to the concrete type's extras; harder to evolve | Accept interfaces, **return structs** |
| 6 | An interface with a single implementation and no consumer variety | Needless indirection | Add the interface when a second implementation or a test fake appears |
| 7 | `any` everywhere | Loss of type safety, panicking assertions | Interfaces with methods, or generics |
| 8 | Unchecked type assertion `x.(T)` | Panic | `v, ok := x.(T)` |
| 9 | Designing the interface around the *convenient* implementation (no errors, no context) | Every signature changes when a real DB arrives | Model what can fail and be slow |
| 10 | Treating "not found" as a 500 (or all errors as 404) | Wrong status codes; noisy logs | Sentinel errors + `errors.Is` |
| 11 | Leaking store error text to clients | Information disclosure | Log the cause; return a generic message |
| 12 | Comparing interface values holding uncomparable types (`==` on slices inside) | Runtime panic | Compare fields or use `reflect.DeepEqual`/`slices.Equal` |
| 13 | Singleton globals for things that should be injected | Tight coupling, untestable | Create once in `main`, pass down |
| 14 | Import cycles when moving code around | Compile error | Move the shared piece to a leaf package (like `reqid`) |

---

## 19. Interview questions

**Q1. What is an interface in Go?**
A type defined by a set of method signatures. Any type with those methods satisfies it implicitly, without declaring so.

**Q2. How does Go's interface satisfaction differ from Java's?**
It is implicit (structural): no `implements`. The type author needn't know about the interface; the consumer defines the interface it needs.

**Q3. What is the nil-interface gotcha?**
An interface holding a nil *pointer* is not itself nil (its type part is non-nil), so `iface != nil` is true. Return a literal `nil` instead of a nil concrete pointer.

**Q4. Why can `T` fail to satisfy an interface that `*T` satisfies?**
Method sets: `T` includes only value-receiver methods; `*T` includes both. Pointer-receiver methods can't be called on a copy stored in an interface.

**Q5. What is `any`, and when do you use type assertions?**
`any` = `interface{}`, satisfied by all types. Use assertions/type switches to recover the concrete type at genuine dynamic boundaries; prefer typed interfaces/generics elsewhere.

**Q6. "Accept interfaces, return structs" why?**
Parameters as interfaces maximize what callers can pass (flexibility, testability); returning concrete types gives callers full functionality and lets you add methods without breaking anyone.

**Q7. Where should an interface be declared?**
In the package that consumes it, small and specific, so producers don't import it and fakes stay tiny.

**Q8. How is an interface value represented?**
Two words: a pointer to the type/method table (itab) and a pointer to (or copy of) the data.

**Q9. Singleton in Go?**
`sync.Once` for thread-safe lazy initialization; but prefer dependency injection to avoid hidden global state.

**Q10. What do interfaces buy you in testing?**
The ability to substitute fakes/stubs/mocks, including ones that simulate failure, so handlers/services are tested in isolation, deterministically, and fast.

---

## 20. Exercises

### Exercise 1: Make it satisfy
Which of these compile? Fix the ones that don't.

```go
type Stringer interface{ String() string }

type A struct{}
func (a A) String() string { return "A" }

type B struct{}
func (b *B) String() string { return "B" }

var s1 Stringer = A{}
var s2 Stringer = B{}
var s3 Stringer = &B{}
var s4 Stringer = &A{}
```

<details><summary>Solution</summary>

`s1` ✅; `s2` ❌ (`String` has a pointer receiver, so `B` doesn't satisfy it): use `&B{}`; `s3` ✅; `s4` ✅ (`*A` includes `A`'s value-receiver methods).
</details>

### Exercise 3: The typed-nil trap
Predict the output, then fix it:

```go
type NotFound struct{ ID int }
func (e *NotFound) Error() string { return fmt.Sprintf("id %d not found", e.ID) }

func find(id int) error {
	var err *NotFound
	if id > 100 {
		err = &NotFound{id}
	}
	return err
}

func main() { fmt.Println(find(1) == nil) }
```

<details><summary>Solution</summary>

Prints `false`. `find` returns a non-nil interface holding a nil `*NotFound`. Fix: `if id > 100 { return &NotFound{id} }; return nil`.
</details>

### Exercise 3: Write a Strategy
Implement `Discount` strategies (`NoDiscount`, `PercentOff(10)`, `FlatOff(5)`) behind an interface and apply them at checkout. Add a new one *without touching* `checkout`.

<details><summary>Solution</summary>

```go
type Discount interface{ Apply(price float64) float64 }

type NoDiscount struct{}
type PercentOff struct{ Percent float64 }
type FlatOff struct{ Amount float64 }

func (NoDiscount) Apply(p float64) float64  { return p }
func (d PercentOff) Apply(p float64) float64 { return p * (1 - d.Percent/100) }
func (d FlatOff) Apply(p float64) float64 {
	if p < d.Amount {
		return 0
	}
	return p - d.Amount
}

func checkout(price float64, d Discount) float64 { return d.Apply(price) }
```
A new `BuyOneGetOne` type only needs an `Apply` method; `checkout` never changes.
</details>

### Exercise 4: Decorate the store
Write `LoggingStore` that wraps a `product.Store` and logs each call's name and duration, then delegates. Where do you create it, and what does that show about interfaces?

<details><summary>Solution</summary>

```go
type LoggingStore struct {
	next   product.Store
	logger *slog.Logger
}

func (s LoggingStore) List(ctx context.Context) ([]models.Product, error) {
	start := time.Now()
	res, err := s.next.List(ctx)
	s.logger.Info("store.List", "duration", time.Since(start), "err", err)
	return res, err
}
// Get and Create likewise
```
In `cmd/wire.go`: `product.NewHandler(LoggingStore{next: productStore, logger: logger}, logger)`. The handler is unchanged and unaware: the decorator satisfies the same interface (the pattern behind middleware).
</details>

### Exercise 5: A spy
Write a `spyStore` recording how many times `List` was called, and a test asserting that `GET /products` calls it exactly once.

<details><summary>Solution</summary>

```go
type spyStore struct {
	Store
	listCalls int
}

func (s *spyStore) List(ctx context.Context) ([]models.Product, error) {
	s.listCalls++
	return s.Store.List(ctx)
}
```
Embedding the interface (`Store`) gives the spy all other methods for free, and it overrides just `List`: a neat Go trick for partial fakes.
</details>

### Exercise 6: Pointer receivers and mutexes
`go vet` warns "passes lock by value" for `func (s ProductStore) List()`. Why is a value receiver wrong for a struct with a `sync.RWMutex`?

<details><summary>Solution</summary>

A value receiver copies the struct on each call, including the mutex, so each call locks its *own copy* and provides no mutual exclusion (and copying a locked mutex is a bug). Use pointer receivers so all callers share the one mutex.
</details>

### Exercise 7 (challenge): Interface segregation
`user.Store` has three methods, but `Me` only needs `FindByID`. Sketch how you'd split it so each handler depends only on what it uses, and say whether it's worth it here.

<details><summary>Solution</summary>

Define `type byIDFinder interface{ FindByID(ctx, id) (User, error) }`, `type registrar interface{ Create(...) }`, `type emailFinder interface{ FindByEmail(...) }`, and give each handler method (or a smaller handler type) only its interface, or embed them into `Store` for the composition root. It reduces what a fake must implement (a `Me` test needs only `FindByID`), at the cost of more small types. For three methods, one interface is fine; segregate when fakes become annoying or interfaces grow beyond ~5 methods.
</details>

---

## 21. Quiz

1. What makes a Go type satisfy an interface?
2. Who should define an interface: the producer or the consumer?
3. Why doesn't `T` satisfy an interface whose method has a pointer receiver?
4. What are the two words in an interface value?
5. What does `x.(T)` do if `x` doesn't hold a `T`? How do you avoid that?
6. Why did the `Store` methods gain a `context.Context` and an `error`?
7. What's the advantage of a fake store in tests?
8. Why is `sync.Once` needed for a lazy singleton?

<details><summary>Answers</summary>

1. Having all the interface's methods with exactly matching signatures (in its method set).
2. The consumer, in the package that uses it.
3. A copy stored in the interface can't be modified by a pointer method; `T`'s method set only has value-receiver methods.
4. A type/method-table pointer and a data pointer.
5. It panics; use `v, ok := x.(T)`.
6. Real storage is slow and can fail; the abstraction should model that so swapping in PostgreSQL doesn't change every caller.
7. It can simulate failures and edge cases deterministically, testing paths the real store can't easily produce.
8. Concurrent first calls could each create an instance (a data race); `Once` guarantees a single initialization.
</details>

---

## 22. Summary

- **Abstraction** hides details behind a contract. A Go **interface** is a set of method signatures; types satisfy it **implicitly**, so producers and consumers stay decoupled.
- **Interface values** are `(type, value)` pairs (16 bytes) with **dynamic dispatch**. Beware the **typed-nil** trap and **method sets** (`T` has only value-receiver methods; `*T` has all).
- Prefer **small interfaces** (`io.Reader`, `error`, `http.Handler`), **compose** them by embedding, use **`any`** sparingly, and recover concrete types with **`v, ok := x.(T)`** or a **type switch**.
- **Accept interfaces, return structs**, and **define interfaces where they're used**: this inverts dependencies (the "D" in SOLID).
- **Patterns**: Strategy (swap algorithms), Decorator (wrap to add behavior; middleware), Factory (`New...`), Adapter (`http.HandlerFunc`), Functional options, Repository (our `Store`), Singleton (`sync.Once`, but prefer DI).
- In the project: `product.Store` / `user.Store` interfaces with **`context` and `error`**, sentinel errors (`models.ErrNotFound`, `ErrEmailTaken`) checked with `errors.Is`, a small `reqid` leaf package to avoid an import cycle, and **fake stores** that let us test the failure paths (generic `500`s, causes logged, never leaked).
- Only the **composition root** knows the concrete storage: Chapter 52 swaps in PostgreSQL by changing two lines there.

### ➡️ What's next?

**Part 11: databases.** [Chapter 52](52-connecting-to-postgresql.md) connects to **PostgreSQL** with `database/sql`, sets up a connection pool, and prepares to implement our `Store` interfaces for real.
