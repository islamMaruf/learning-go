# Chapter 1: Your First Go Program — Hello, World!

> **Goal of this chapter:** Install Go, write a tiny program, run it, and understand every single word in it. No prior programming experience is assumed.

**Difficulty:** 🟢 Beginner  **Estimated time:** 45–60 minutes

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [What is Go, and why learn it?](#2-what-is-go-and-why-learn-it)
3. [How programming languages work (the 2-minute version)](#3-how-programming-languages-work-the-2-minute-version)
4. [Installing Go](#4-installing-go)
5. [Setting up a project](#5-setting-up-a-project)
6. [Writing your first program](#6-writing-your-first-program)
7. [Anatomy of the program, word by word](#7-anatomy-of-the-program-word-by-word)
8. [Running vs. building](#8-running-vs-building)
9. [Printing more things: `Print`, `Println`, `Printf`](#9-printing-more-things-print-println-printf)
10. [Comments](#10-comments)
11. [Reading compiler errors](#11-reading-compiler-errors)
12. [Common mistakes](#12-common-mistakes)
13. [Exercises (with solutions)](#13-exercises)
14. [Quick quiz](#14-quick-quiz)
15. [Summary and what's next](#15-summary)

---

## 1. What you will learn

By the end of this chapter you will be able to:

- ✅ Explain what Go is and where it is used
- ✅ Install Go and verify the installation
- ✅ Create a Go module (a project) with `go mod init`
- ✅ Write, run, and build a program
- ✅ Explain what `package main`, `import`, and `func main()` mean
- ✅ Read a basic compiler error message and fix it

**Prerequisites:** A computer, a text editor (we recommend [VS Code](https://code.visualstudio.com/)), and curiosity. That's it.

---

## 2. What is Go, and why learn it?

**Go** (sometimes called **Golang**, because the website is golang.org) is a programming language created at **Google** in 2007 by Robert Griesemer, Rob Pike, and Ken Thompson (a co-creator of Unix and the C language). It was released publicly in 2009.

They designed it because they were frustrated: the languages they used at Google were either *fast but painful to write* (C++) or *pleasant but slow* (Python). They wanted a language that is:

| Goal | What it means for you |
|------|----------------------|
| **Simple** | Small language: you can learn almost all of it in a few weeks |
| **Fast** | Compiles to native machine code, so programs run quickly |
| **Great at concurrency** | Doing many things at once is built into the language (Chapters 36, 64–70) |
| **Easy to ship** | Produces a single file you can copy to a server and run |
| **Readable** | Everyone's Go code looks alike, thanks to the built-in formatter |

**Where is Go used?** Docker, Kubernetes, Terraform, Prometheus, and GitHub CLI are all written in Go. Companies like Google, Uber, Twitch, Dropbox, and Cloudflare use it for web servers, cloud tooling, and command-line programs.

---

## 3. How programming languages work (the 2-minute version)

A computer's processor (CPU) only understands **machine code**: long sequences of 0s and 1s. Humans can't comfortably write that, so we write **source code** in a readable language and use a tool to translate it.

```
 You write             Compiler translates             CPU runs
┌───────────┐  go build  ┌──────────────────┐  execute  ┌────────┐
│ main.go   │ ─────────► │ machine code     │ ────────► │ output │
│ (text)    │            │ (executable file)│           │        │
└───────────┘            └──────────────────┘           └────────┘
```

- **Source code** — the text you write (`main.go`).
- **Compiler** — a program that translates source code to machine code. Go's compiler is built into the `go` command.
- **Executable / binary** — the translated program the computer can run directly.

Go is a **compiled** language. This is different from Python or JavaScript, which are usually *interpreted* (translated line by line while running). Compiling has two big benefits: programs run fast, and the compiler catches many mistakes *before* your program ever runs.

---

## 4. Installing Go

### Step 1: Download

Go to **https://go.dev/dl/** and download the installer for your operating system. Choose the latest stable version (this tutorial needs **Go 1.22 or newer**, because Chapter 44 uses features added in 1.22).

- **Windows:** run the `.msi` installer and click through it.
- **macOS:** run the `.pkg` installer, or use Homebrew: `brew install go`
- **Linux:** follow the tarball instructions on the download page, or use your package manager (make sure the version is new enough).

### Step 2: Verify

Open a **terminal** (on Windows: *PowerShell* or *Command Prompt*; on macOS: *Terminal*; on Linux: your terminal app) and run:

```bash
go version
```

You should see something like:

```
go version go1.22.0 linux/amd64
```

The parts mean: `go1.22.0` is the Go version, `linux` is your operating system, `amd64` is your CPU type.

> 💡 **"command not found"?** Close and reopen your terminal. If it still fails, the Go `bin` folder isn't on your `PATH` (the list of folders your shell searches for commands). Re-run the installer or check the official install guide.

### Step 3: Install VS Code + the Go extension (recommended)

1. Install VS Code.
2. Open the *Extensions* panel (`Ctrl+Shift+X`), search for **Go** (published by the Go Team at Google), and install it.
3. When VS Code offers to install "Go tools", accept. This gives you autocompletion, error underlines, and automatic formatting on save.

---

## 5. Setting up a project

### Step 1: Make a folder

```bash
# Linux / macOS
mkdir -p ~/go_projects/hello
cd ~/go_projects/hello
```

```powershell
# Windows PowerShell
mkdir $HOME\go_projects\hello
cd $HOME\go_projects\hello
```

### Step 2: Create a *module*

A Go **module** is a project: a folder of Go code with a name and a list of dependencies. Every real Go project starts with:

```bash
go mod init hello
```

This creates one file, `go.mod`:

```
module hello

go 1.22
```

- `module hello` — the name of your project. (For projects you publish, the name is usually a URL such as `github.com/yourname/hello`.)
- `go 1.22` — the Go language version your code expects.

> 📝 **Note:** You can run a single file with `go run main.go` even without a `go.mod`, and the original version of this tutorial did that. But modules are how every real project works, and you will need them from Chapter 10 onward, so we start the good habit now.

### Step 3: Open the folder in VS Code

```bash
code .
```

(or use **File → Open Folder** and choose `hello`).

---

## 6. Writing your first program

Create a new file named **`main.go`** inside the folder and type this in **by hand** (typing builds muscle memory; copy-pasting doesn't):

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
```

Now run it in the terminal (inside the same folder):

```bash
go run .
```

Output:

```
Hello, World!
```

🎉 **You just ran your first Go program.**

> `go run .` means "compile and run the package in the current folder (`.`)". `go run main.go` also works and does the same thing here.

---

## 7. Anatomy of the program, word by word

Let's take the program apart.

```
package main                       ← 1. which package am I in?

import "fmt"                       ← 2. what tools do I borrow?

func main() {                      ← 3. where does the program start?
	fmt.Println("Hello, World!")   ← 4. what does it do?
}
```

### 7.1 `package main`

Go organizes code into **packages**: named groups of related code. Every Go file must begin by saying which package it belongs to.

`main` is a *special* package name. It tells Go: "this is a program you can run", not a library that other programs import.

> 📌 **Rule:** A runnable program needs `package main` **and** a `func main()`.

### 7.2 `import "fmt"`

`import` lets you use code that someone else already wrote. `fmt` (pronounced "fumpt" or "eff-em-tee", short for **format**) is part of Go's **standard library**, a large set of ready-made packages that ships with Go.

> **Analogy:** Your kitchen (your program) doesn't grow its own flour. You *import* it from the shop (the standard library) and then use it.

### 7.3 `func main()`

```go
func main() {
    ...
}
```

| Piece | Meaning |
|-------|---------|
| `func` | Keyword meaning "I am defining a **function**". A function is a named block of code you can run. |
| `main` | The function's name. When your program starts, Go automatically runs the function named `main`. This is the **entry point**. |
| `()` | Parameter list: things the function receives. `main` receives nothing, so it's empty. |
| `{ }` | Curly braces enclose the function body — the instructions to run. |

> ⚠️ Go is strict about brace placement: the opening `{` **must** be on the same line as `func main()`.

### 7.4 `fmt.Println("Hello, World!")`

| Piece | Meaning |
|-------|---------|
| `fmt` | The package we imported |
| `.` | "look inside": access something that belongs to `fmt` |
| `Println` | A function inside `fmt`. It **prints** its argument, then a new **line** (`ln`) |
| `("Hello, World!")` | The **argument**: the value we hand to the function |
| `"Hello, World!"` | A **string**: text between double quotes |

Names that start with a **capital letter** (like `Println`) are *exported*, meaning they're usable from other packages. That's why `fmt.Println` is capitalized. (More in Chapter 10.)

### 7.5 Whitespace and tabs

Go code is indented with **tabs**, and the tool `gofmt` does this for you. In VS Code, formatting happens on save. You can also run:

```bash
gofmt -w main.go
```

Because everyone uses `gofmt`, all Go code looks the same, which makes reading other people's code easy.

---

## 8. Running vs. building

There are two ways to turn your source code into a running program.

### 8.1 `go run` — compile, run, throw away

```bash
go run .
```

Go compiles your code into a temporary executable in a hidden folder, runs it, and deletes it. Great for quick experiments.

### 8.2 `go build` — make a real executable

```bash
go build
```

This creates a file named after your module (`hello` on Linux/macOS, `hello.exe` on Windows) in the current folder. Run it directly:

```bash
./hello        # Linux / macOS
.\hello.exe    # Windows
```

This file is **standalone**: you can copy it to another computer with the same OS and CPU, and it runs *without Go installed*. That is one of Go's superpowers, and why it's so popular for servers and tools.

### 8.3 Useful commands at a glance

| Command | What it does |
|---------|--------------|
| `go version` | Show the installed Go version |
| `go run .` | Compile and run the current package |
| `go build` | Compile to an executable file |
| `go fmt ./...` | Format all code in the project |
| `go vet ./...` | Look for suspicious code the compiler allows |
| `go mod init name` | Start a new module |
| `go help` | List all commands |

---

## 9. Printing more things: `Print`, `Println`, `Printf`

`fmt` has a family of printing functions. Try this program:

```go
package main

import "fmt"

func main() {
	fmt.Println("Line one")
	fmt.Println("Line two")

	fmt.Print("No newline here. ")
	fmt.Print("See? Same line.\n") // \n means "new line"

	// Println can take several values and puts spaces between them
	fmt.Println("I am", 30, "years old")

	// Printf: print with a *format*. %s = string, %d = integer
	fmt.Printf("My name is %s and I am %d.\n", "Asha", 30)
}
```

Output:

```
Line one
Line two
No newline here. See? Same line.
I am 30 years old
My name is Asha and I am 30.
```

| Function | Behavior |
|----------|----------|
| `Print` | Prints exactly what you give it. No automatic newline. |
| `Println` | Prints values separated by spaces, then adds a newline. |
| `Printf` | Prints a template. Placeholders like `%s` and `%d` are replaced by the values that follow. **You** must add `\n` for a newline. |

**Common format verbs** (you'll use them constantly):

| Verb | Meaning | Example |
|------|---------|---------|
| `%s` | string | `"hi"` |
| `%d` | integer | `42` |
| `%f` | floating-point number | `3.140000` (use `%.2f` for `3.14`) |
| `%t` | boolean | `true` |
| `%v` | *any* value in default format | works for everything |
| `%T` | the **type** of the value | `int`, `string` |
| `%q` | string with quotes | `"hi"` |

### Special characters inside strings

| Sequence | Meaning |
|----------|---------|
| `\n` | New line |
| `\t` | Tab |
| `\"` | A literal double quote |
| `\\` | A literal backslash |

```go
package main

import "fmt"

func main() {
	fmt.Println("She said, \"Go is fun!\"")
	fmt.Println("Name:\tAsha")
	fmt.Println(`Backticks make a "raw" string, so \n stays as \n`)
}
```

Output:

```
She said, "Go is fun!"
Name:	Asha
Backticks make a "raw" string, so \n stays as \n
```

---

## 10. Comments

Comments are notes for humans. The compiler ignores them.

```go
package main

import "fmt"

// This is a single-line comment.

/*
   This is a multi-line comment.
   Useful for longer explanations.
*/

func main() {
	fmt.Println("Comments are ignored") // a comment can also follow code
}
```

> 💡 Good comments explain **why** you did something, not **what** the code does (the code already shows that).

---

## 11. Reading compiler errors

Beginners fear error messages, but Go's are helpful. Let's break something on purpose.

```go
// INTENTIONAL ERROR: this does not compile
package main

import "fmt"

func main() {
	fmt.println("Hello")
}
```

Running it prints:

```
./main.go:6:6: undefined: fmt.println
```

Read it like this:

```
./main.go : 6 : 6 :   undefined: fmt.println
   │        │   │            └── what's wrong
   │        │   └── column (character position) in the line
   │        └── line number
   └── file name
```

The message says `fmt.println` doesn't exist. The fix: capital **P** → `fmt.Println`.

> 🧭 **Debugging habit:** Read the error from the top. Go to the exact file and line it names. Fix the *first* error first; later errors are often just consequences of it.

---

## 12. Common mistakes

| # | Mistake | Example | Fix |
|---|---------|---------|-----|
| 1 | Lowercase function name | `fmt.println("x")` | `fmt.Println("x")` — Go is **case-sensitive** |
| 2 | Single quotes for a string | `fmt.Println('Hi')` | Use double quotes `"Hi"`. (Single quotes are for single characters — Chapter 2.) |
| 3 | Missing quotes | `fmt.Println(Hello)` | `"Hello"` — otherwise Go thinks `Hello` is a variable name |
| 4 | Unused import | `import "fmt"` but never used | Remove it or use it. Go refuses to compile with unused imports. |
| 5 | Wrong package name | `package app` with `func main()` | Runnable programs need `package main` |
| 6 | Brace on its own line | `func main()` ⏎ `{` | Put `{` on the same line |
| 7 | Running in the wrong folder | `go: cannot find main module` | `cd` into the folder that has `go.mod`, or run `go mod init` |
| 8 | File not saved | Old output appears | Save the file (`Ctrl+S`) before running |

Example of mistake #4:

```go
// INTENTIONAL ERROR: unused import
package main

import "fmt"

func main() {
}
```

```
./main.go:3:8: "fmt" imported and not used
```

Go is strict about this on purpose: unused code is usually a bug or clutter.

---

## 13. Exercises

Try each one **before** reading the solution.

### Exercise 1: Introduce yourself
Print your name and your favorite hobby on two separate lines.

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	fmt.Println("My name is Asha")
	fmt.Println("My hobby is painting")
}
```
</details>

### Exercise 2: Use `Printf`
Using **one** `Printf` call, print: `Asha is 30 years old and lives in Dhaka.`

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	fmt.Printf("%s is %d years old and lives in %s.\n", "Asha", 30, "Dhaka")
}
```
</details>

### Exercise 3: Draw a box
Print this exactly (use `Println` several times):

```
+------+
| Go!  |
+------+
```

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func main() {
	fmt.Println("+------+")
	fmt.Println("| Go!  |")
	fmt.Println("+------+")
}
```
</details>

### Exercise 4: Fix the bugs
This program has **three** mistakes. Find and fix them without running it first.

```go
// INTENTIONAL ERROR: find the three bugs
package main

import "fmt"
import "os"

func main()
{
    fmt.println('Hello, Go')
}
```

<details>
<summary>Solution</summary>

1. `"os"` is imported but never used → remove it.
2. `{` must be on the same line as `func main()`.
3. `fmt.println` → `fmt.Println`, and `'Hello, Go'` → `"Hello, Go"` (double quotes).

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, Go")
}
```
</details>

### Exercise 5 (challenge): Build and ship
Use `go build` to create an executable, then run it directly *without* `go run`. Look at the file size (`ls -lh` / `dir`). Why is it a few MB for such a tiny program?

<details>
<summary>Solution</summary>

Go executables include the **Go runtime** (memory management, the scheduler, garbage collector; see Chapter 39), which is why even "Hello, World" is around 1–2 MB. That's the price of needing nothing else installed on the target machine.
</details>

---

## 14. Quick quiz

1. What does `package main` tell Go?
2. Why is `Println` capitalized?
3. What is the difference between `go run` and `go build`?
4. What happens if you import a package but don't use it?
5. What does `\n` do inside a string?

<details>
<summary>Answers</summary>

1. That this file is part of a runnable program (as opposed to a library).
2. Capitalized names are *exported*: usable from other packages.
3. `go run` compiles to a temporary file and runs it. `go build` leaves a permanent executable.
4. The compiler stops with an error (`imported and not used`).
5. It inserts a line break.
</details>

---

## 15. Summary

- **Go** is a fast, simple, compiled language made at Google, great for servers, cloud tools, and concurrent programs.
- A project is a **module**, created with `go mod init <name>`.
- Every runnable program has **`package main`** and **`func main()`**.
- **`import`** brings in packages; **`fmt`** handles printing and formatting.
- **`go run .`** to try it, **`go build`** to produce a standalone executable.
- Go is **case-sensitive**, uses **double quotes** for strings, forbids **unused imports**, and is formatted by **`gofmt`**.
- Error messages tell you `file:line:column: what's wrong`. Read them.

### ➡️ What's next?

In [Chapter 2](02-variables-and-data-types.md) you'll learn how programs **remember information** using variables, and Go's data types: numbers, text, and true/false values.
