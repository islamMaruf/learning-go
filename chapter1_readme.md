# Go Programming Tutorial - Chapter 1
## Your First Go Program: Hello World

### 📚 Table of Contents
1. [Prerequisites](#prerequisites)
2. [Setting Up Your Project](#setting-up-your-project)
3. [Understanding the Basic Structure](#understanding-the-basic-structure)
4. [Writing Your First Program](#writing-your-first-program)
5. [Running the Program](#running-the-program)
6. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
7. [Summary](#summary)

---

## Prerequisites

Before starting this tutorial, ensure you have the following installed:

✅ **VS Code** - Your code editor  
✅ **Go Programming Language** - Version **1.22 or higher**

### Checking Your Go Installation

Open a terminal and run:

```bash
go version
```

You should see something like:
```
go version go1.22 linux/amd64
```

If you see version 1.22 or higher, you're ready to proceed!

---

## Setting Up Your Project

### Step 1: Create a Project Folder

Create a folder structure for organizing your Go projects:

```
Desktop/
└── go_projects/
    └── first_program/
```

**On Linux/macOS:**
```bash
cd ~/Desktop
mkdir go_projects
cd go_projects
mkdir first_program
```

**On Windows:**
```bash
cd Desktop
mkdir go_projects
cd go_projects
mkdir first_program
```

### Step 2: Open the Project in VS Code

1. Launch **VS Code**
2. Click **File** → **Open Folder**
3. Navigate to `Desktop/go_projects/`
4. Select the `first_program` folder (single click to select, not double-click)
5. Click **Open**

You should now see `first_program` in the VS Code sidebar.

---

## Understanding the Basic Structure

Every Go program has a fundamental structure. Let's understand each component:

### 1. Package Declaration

```go
package main
```

**What it means:**
- **Every Go file MUST start with a package declaration**
- `package main` indicates this is an executable program
- Without `package main`, your program won't run as a standalone application

📌 **Rule:** Every Go file needs a package. For executable programs, use `package main`.

---

### 2. Import Statements

```go
import "fmt"
```

**What it means:**
- `import` brings in external packages/libraries
- `fmt` stands for **"format"** - a built-in Go package
- The `fmt` package contains pre-written functions for input/output operations

**Analogy:** Think of it like buying a dress from a shop. If you want to use `fmt`, you must first import it from Go's standard library.

---

### 3. Main Function

```go
func main() {
    // Your code goes here
}
```

**Breaking it down:**

| Component | Explanation |
|-----------|-------------|
| `func` | Short for "function" (programmers love cute short names!) |
| `main` | The name of the function - this is the entry point |
| `()` | Parentheses (round brackets) - for parameters (empty here) |
| `{}` | Curly braces - contains the code to execute |

📌 **Critical Rule:** A Go file with a `main` function MUST have `package main` at the top.

---

## Writing Your First Program

### Step 1: Create main.go File

In VS Code, create a new file in the `first_program` folder:

**Filename:** `main.go`

### Step 2: Write the Code

Type the following code exactly as shown:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

Let's understand each line:

#### Line-by-Line Explanation:

```go
package main
```
↳ Declares this is the main package (executable program)

```go
import "fmt"
```
↳ Imports the format package for printing capabilities

```go
func main() {
```
↳ Defines the main function (program entry point)

```go
    fmt.Println("Hello, World!")
```
↳ Prints "Hello, World!" to the console

```go
}
```
↳ Closes the main function

---

### Understanding fmt.Println()

```go
fmt.Println("Hello, World!")
```

**Breaking it down:**

- **`fmt`** - The package we imported
- **`.`** - Dot notation to access functions inside the package
- **`Println`** - Print Line function
  - `Print` = Print text
  - `ln` = Line (adds a newline/enter after printing)
- **`"Hello, World!"`** - The string to print (must be in double quotes)

**Example Output:**

```
Hello, World!
[cursor moves to next line automatically]
```

The `ln` in `Println` automatically adds an "Enter" after the text!

---

## Running the Program

### Using VS Code Terminal

#### Step 1: Open Terminal in VS Code

1. Click **Terminal** → **New Terminal** (at the top menu)
2. A terminal panel will appear at the bottom

#### Step 2: Run Your Program

In the terminal, type:

```bash
go run main.go
```

Press **Enter**, and you should see:

```
Hello, World!
```

🎉 **Congratulations!** You've just run your first Go program!

---

### Understanding the Command

```bash
go run main.go
```

| Part | Explanation |
|------|-------------|
| `go` | The Go command-line tool |
| `run` | Command to compile and execute |
| `main.go` | The file to run |

---

## Common Mistakes to Avoid

### ❌ Mistake 1: Wrong Case Sensitivity

```go
// ❌ WRONG
fmt.println("Hello")  // lowercase 'p'

// ✅ CORRECT
fmt.Println("Hello")  // uppercase 'P'
```

**Why:** Go is case-sensitive. `Println` starts with a capital P.

---

### ❌ Mistake 2: Missing or Wrong Quotes

```go
// ❌ WRONG
fmt.Println('Hello, World!')  // Single quotes

// ❌ WRONG
fmt.Println(Hello, World!)    // No quotes

// ✅ CORRECT
fmt.Println("Hello, World!")  // Double quotes
```

**Rule:** Strings must be enclosed in **double quotes** (`"`).

---

### ❌ Mistake 3: Adding Semicolons

```go
// ❌ WRONG (but might work)
fmt.Println("Hello");  // Unnecessary semicolon

// ✅ CORRECT
fmt.Println("Hello")   // No semicolon needed
```

**Note:** Go automatically inserts semicolons. Don't add them manually!

---

### ❌ Mistake 4: Wrong Package Name

```go
// ❌ WRONG
package myprogram  // Wrong package name

import "fmt"

func main() {
    fmt.Println("Hello")
}
```

**Error:** If you have a `main` function, the package must be `package main`.

---

### ❌ Mistake 5: Importing Unused Packages

```go
// ❌ WRONG - will cause compile error
package main

import "fmt"  // Imported but not used below

func main() {
    // Not using fmt here
}
```

**Rule:** Go doesn't allow unused imports. If you import it, use it!

---

### ❌ Mistake 6: Wrong Folder/File Path

Make sure:
- You opened the **correct folder** (`first_program`) in VS Code
- Your file is named **`main.go`** exactly
- You're in the correct directory in the terminal

**Check current directory:**
```bash
pwd  # Linux/macOS
cd   # Windows
```

---

## Program Structure Visualization

```
┌─────────────────────────────────────────┐
│ package main                            │ ← Package Declaration
├─────────────────────────────────────────┤
│ import "fmt"                            │ ← Import Statement
├─────────────────────────────────────────┤
│ func main() {                           │ ← Main Function Start
│     ┌─────────────────────────────────┐ │
│     │ fmt.Println("Hello, World!")    │ │ ← Your Code
│     └─────────────────────────────────┘ │
│ }                                       │ ← Main Function End
└─────────────────────────────────────────┘
```

---

## Summary

### 🎯 Key Concepts Learned

1. **Package Declaration:**
   - Every Go file must start with a package declaration
   - Executable programs use `package main`

2. **Import Statements:**
   - Use `import` to bring in external packages
   - `fmt` package provides formatting and printing functions

3. **Main Function:**
   - `func main()` is the entry point of your program
   - Must be in a file with `package main`

4. **Printing Output:**
   - `fmt.Println()` prints text to console
   - `Println` adds a newline automatically
   - Strings must be in double quotes

5. **Running Go Programs:**
   - Use `go run filename.go` to execute
   - Run from the correct directory

---

### 📝 Complete Program Template

Use this as a template for future programs:

```go
package main

import "fmt"

func main() {
    // Your code here
    fmt.Println("Your message here")
}
```

---

### 🔧 Syntax Rules Checklist

- [ ] File starts with `package main`
- [ ] Imported packages are actually used
- [ ] `Println` has capital P
- [ ] Strings use double quotes (`"`)
- [ ] Curly braces `{}` are properly matched
- [ ] No unnecessary semicolons
- [ ] Function name is `main` (lowercase)

---

## Practice Exercises

### Exercise 1: Modify the Message
Change the program to print your name:
```go
fmt.Println("Hello, [Your Name]!")
```

### Exercise 2: Multiple Lines
Print multiple lines:
```go
fmt.Println("Line 1")
fmt.Println("Line 2")
fmt.Println("Line 3")
```

**Expected Output:**
```
Line 1
Line 2
Line 3
```

### Exercise 3: Print Different Messages
Try printing:
- Your favorite programming language
- Today's date
- A motivational quote

---

## Getting Help

If you encounter any problems:

1. **Check your syntax** - Compare with the examples above
2. **Read error messages** - They often tell you what's wrong
3. **Use ChatGPT** - Ask for help debugging your code
4. **Google the error** - You're likely not the first to encounter it
5. **Ask in communities:**
   - Discord programming servers
   - Facebook programming groups
   - YouTube comments section
   - Reddit r/golang

**Remember:** Everyone makes mistakes! Don't get frustrated. Keep trying, and reach out for help when needed. Even experienced programmers look up syntax and ask for help!

---

## Next Steps

After mastering this basic program, you'll learn:
- Variables and data types
- User input
- Conditional statements (if/else)
- Loops
- Functions

Keep practicing! 🚀

---

*Based on Go Programming Tutorial - Chapter 1*
*Prerequisites: Go 1.22+, VS Code*
