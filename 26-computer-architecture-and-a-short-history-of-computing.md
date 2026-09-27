# Chapter 26: Computer Architecture and a Short History of Computing

> **Goal of this chapter:** Take a break from Go syntax to understand the machine your code runs on. You'll learn the three components every programmer should know (**CPU, RAM, disk**), how a computer stores everything as **binary**, what **bits and bytes** are, and how we got here, from the abacus to punch cards. This foundation makes the next chapters (operating systems, processes, threads, and finally goroutines) click.

**Difficulty:** 🟡 Beginner-friendly (no coding background needed)  **Estimated time:** 2 hours  **Prerequisite:** None beyond curiosity (but you'll recognize things from Chapters 2 and 18)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Why learn computer architecture?](#2-why-learn-computer-architecture)
3. [What is computer architecture?](#3-what-is-computer-architecture)
4. [The three components that matter](#4-the-three-components-that-matter)
5. [RAM: primary memory](#5-ram-primary-memory)
6. [Hard disk / SSD: secondary storage](#6-hard-disk--ssd-secondary-storage)
7. [RAM vs. disk](#7-ram-vs-disk)
8. [CPU: the brain](#8-cpu-the-brain)
9. [Binary: the language of computers](#9-binary-the-language-of-computers)
10. [Bits, bytes, and sizes](#10-bits-bytes-and-sizes)
11. [How data is represented](#11-how-data-is-represented)
12. [The memory hierarchy](#12-the-memory-hierarchy)
13. [A short history of computing](#13-a-short-history-of-computing)
14. [Look at your own machine](#14-look-at-your-own-machine)
15. [Common misconceptions](#15-common-misconceptions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- The roles of the **CPU**, **RAM**, and **storage**
- Why RAM is fast but forgetful, and disks are slower but permanent
- How a CPU executes instructions (the **fetch-decode-execute** cycle)
- What **binary**, **bits**, and **bytes** are, and how numbers, text, and negative numbers become 0s and 1s
- The **memory hierarchy** (registers → cache → RAM → disk) and why it matters for performance
- Key people and inventions in computing history (abacus, Leibniz, Babbage, Lovelace, punch cards)

---

## 2. Why learn computer architecture?

You can write Go without knowing how a CPU works, but:

| Reason | Example |
|--------|---------|
| **Interviews** | "Why is stack allocation faster than heap?" "What's a context switch?" |
| **Performance** | Knowing cache and memory behavior explains why one loop is 10× faster than another |
| **Debugging** | Understanding memory layout explains crashes and panics |
| **Concurrency** | Goroutines, threads, and race conditions only make sense once you know how CPUs and memory really work |
| **Better design** | Senior engineers choose data structures based on hardware reality |

Everything we learned about the stack, heap, and code segment in Chapter 18 lives in the **RAM** you're about to meet properly.

---

## 3. What is computer architecture?

**Computer architecture** is the design of a computer's components and how they work together. You don't need to know how to build a computer, only what the parts are and what each is for.

> **Analogy: the human body.** You don't need to be a doctor to know that the brain thinks, the heart pumps blood, and the stomach digests food. Likewise, for a programmer: the **CPU** thinks, **RAM** is short-term memory, and the **disk** is long-term memory.

Open a desktop computer and you'd find a motherboard, fans, a power supply, wires, and more. For a software engineer, three things matter most.

---

## 4. The three components that matter

```
                 ┌─────────────────────────────────┐
                 │            COMPUTER             │
                 │                                 │
                 │   ┌───────┐                     │
                 │   │  CPU  │  ← thinks / computes│
                 │   └───┬───┘                     │
                 │       │  (very fast connection) │
                 │   ┌───▼───┐                     │
                 │   │  RAM  │  ← short-term memory│
                 │   └───┬───┘                     │
                 │       │                         │
                 │   ┌───▼───┐                     │
                 │   │ Disk  │  ← long-term storage│
                 │   └───────┘                     │
                 └─────────────────────────────────┘
```

| Component | Role | Human analogy |
|-----------|------|---------------|
| **CPU** (Central Processing Unit) | Executes instructions, does calculations and decisions | Your **brain** |
| **RAM** (Random Access Memory) | Holds the programs and data being used **right now** | Your **short-term (working) memory** |
| **Disk** (HDD / SSD) | Stores files and programs **permanently** | Your **long-term memory** / a library |

(Also present: input/output devices such as keyboard, screen, network card; but the big three are the core of programming.)

---

## 5. RAM: primary memory

**RAM** = **R**andom **A**ccess **M**emory. It is where **running programs and their data live**. Every variable you've created in this course lives in RAM.

### Structure: a long row of numbered cells

```
address:   0     1     2     3     4     5     6     7   ...
         ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
         │  ?  │  ?  │  ?  │  ?  │  ?  │  ?  │  ?  │  ?  │ ...
         └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                 each cell = 1 byte (8 bits)
```

- Each cell holds **one byte**.
- Each cell has a unique **address** (a number). This is what a **pointer** stores (Chapter 24).
- "**Random access**" means you can jump to *any* address directly and get the same fast speed, unlike a tape, which must be wound to the right place.

### Characteristics

| Property | RAM |
|----------|-----|
| **Speed** | Very fast (tens of nanoseconds to reach) |
| **Volatile?** | **Yes**: contents are **lost when power goes off** |
| **Size** | Typically 8–64 GB in personal computers |
| **Cost per GB** | More expensive than disk |
| **Used for** | Running programs, their variables (stack, heap, data), open documents |

> **Analogy: your desk.** The desk is small and quickly reachable: you keep what you're working on right now there. When you leave (power off), the desk is cleared.

---

## 6. Hard disk / SSD: secondary storage

**Disk storage** keeps your files, photos, and installed programs **permanently**.

| Type | How it works | Speed |
|------|--------------|-------|
| **HDD** (hard disk drive) | Spinning magnetic platters with a moving read/write head | Slow (milliseconds) |
| **SSD** (solid state drive) | Flash memory chips, no moving parts | Much faster than HDD, still much slower than RAM |

### Characteristics

| Property | Disk |
|----------|------|
| **Speed** | Slow compared to RAM (microseconds for SSD to milliseconds for HDD) |
| **Volatile?** | **No**: data persists without power |
| **Size** | 256 GB to many TB |
| **Cost per GB** | Cheap |
| **Used for** | Files, databases, installed programs, your compiled Go binary |

> **Analogy: a library or filing cabinet.** Enormous and permanent, but fetching a book takes a while. You bring the books you need to your desk (RAM) to work.

**How a program runs:** the program (a file, e.g., your Go binary) lives on the **disk**. When you run it, the operating system **copies it into RAM** and the CPU executes it from there. That's why "loading" takes a moment.

---

## 7. RAM vs. disk

| | RAM | Disk (SSD/HDD) |
|--|-----|----------------|
| Purpose | Working memory | Permanent storage |
| Speed | 🚀 Very fast | 🐢 Slower (SSD ~100× slower than RAM, HDD ~100,000×) |
| Persistence | ❌ Lost at power-off | ✅ Kept |
| Capacity | Smaller | Larger |
| Price / GB | Higher | Lower |

### The love-letter analogy 💌

Imagine you're writing a love letter on a whiteboard (RAM). If the power goes out, or someone erases it, your words are gone. If you want to keep it, you must copy it onto paper and put it in a drawer (disk). That's **"Save"**: copying from RAM to disk. An unsaved document is lost when your computer crashes because it only existed in RAM.

---

## 8. CPU: the brain

**CPU** = **C**entral **P**rocessing **U**nit. A small chip (a few centimeters across) containing billions of microscopic switches (**transistors**).

### What does a CPU do?

At its core a CPU does very simple things, very fast:

| Category | Examples of operations |
|----------|------------------------|
| **Arithmetic** | add, subtract, multiply, divide |
| **Logic** | AND, OR, NOT, XOR, shifts |
| **Data movement** | copy a value between RAM and the CPU's tiny internal storage (**registers**) |
| **Comparison and control flow** | compare two values, then *jump* to a different instruction (this is how `if` and `for` work) |

> **Simplification alert.** A popular teaching line says "a CPU can only do 7 operations: + − × ÷ AND OR NOT". That captures the *spirit* (a small set of basic operations) but the real count is larger: modern CPUs implement hundreds of **instructions** (the "instruction set"), grouped in the four categories above. The powerful insight stands: **everything (games, video calls, AI) is built by combining simple operations, billions of times per second.**

### The fetch-decode-execute cycle

Every CPU repeats this loop, billions of times per second (a 3 GHz CPU does about 3 billion cycles per second):

```
   ┌───────────────────────────────────────────────┐
   │                                               │
   ▼                                               │
FETCH    → get the next instruction from RAM       │
DECODE   → figure out what it means                │
EXECUTE  → do it (add, compare, jump, copy...)     │
STORE    → write back the result                   │
   │                                               │
   └───────────────────────────────────────────────┘
```

A **program** (your compiled Go code) is just a long list of such instructions stored in RAM (the **code segment**, Chapter 18). The CPU keeps a special register called the **program counter** that always points to the next instruction to fetch.

Also, **`a := b + c` in Go** becomes roughly: *load `b` into a register; load `c` into another; ADD them; store the result at `a`'s address.*

### Cores and clock speed

- **Clock speed** (GHz): how many cycles per second.
- **Core**: an independent execution unit. A modern CPU has several (4, 8, 16 ...). **Each core can run one instruction stream at a time**: this is the foundation of *parallelism* (Chapter 31).

---

## 9. Binary: the language of computers

### What is binary?

**Binary** is a number system with only two digits: **0** and **1**. Everything in a computer (numbers, text, photos, music, your Go program) is stored as sequences of 0s and 1s.

### Why only 0 and 1?

Computers are built from **transistors**, tiny electronic switches with two reliable states:

```
Switch OFF  → no current → 0
Switch ON   → current    → 1
```

Two states are simple, cheap, and *reliable* (easy to tell apart even with electrical noise). Building hardware with ten distinguishable voltage levels would be far more error-prone.

### Counting in binary

Decimal (base 10) uses digit positions worth 1, 10, 100, 1000... Binary (base 2) uses positions worth **1, 2, 4, 8, 16, 32...**

```
Binary   1  0  1  1  0
Weight  16  8  4  2  1
         │  │  │  │  └─ 0 × 1  = 0
         │  │  │  └──── 1 × 2  = 2
         │  │  └─────── 1 × 4  = 4
         │  └────────── 0 × 8  = 0
         └───────────── 1 × 16 = 16
                              ─────
                               22
```

So `10110` in binary is `22` in decimal.

| Decimal | Binary |
|---------|--------|
| 0 | 0 |
| 1 | 1 |
| 2 | 10 |
| 3 | 11 |
| 4 | 100 |
| 5 | 101 |
| 8 | 1000 |
| 10 | 1010 |
| 12 | 1100 |
| 255 | 11111111 |

### How the CPU does `10 + 12`

```
      1010   (10)
    + 1100   (12)
    ──────
     10110   (22)
```

Binary addition works like decimal addition ("carry the 1"), and a handful of logic gates can do it. The CPU converts *nothing*: it works in binary all along; *we* convert for reading.

### Try it in Go

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	fmt.Printf("%b %b %b\n", 10, 12, 10+12) // 1010 1100 10110

	fmt.Println(strconv.FormatInt(22, 2)) // 10110  (decimal → binary text)

	n, _ := strconv.ParseInt("10110", 2, 64)
	fmt.Println(n) // 22 (binary text → decimal)

	// Bitwise operators work directly on the bits
	fmt.Printf("%08b\n", 0b1100&0b1010) // AND: 00001000
	fmt.Printf("%08b\n", 0b1100|0b1010) // OR:  00001110
	fmt.Printf("%08b\n", 0b1100^0b1010) // XOR: 00000110
	fmt.Println(5<<2)                   // shift left 2 = ×4 → 20
}
```

Every message you've ever sent, every photo, every song, is just a very long string of these 0s and 1s.

---

## 10. Bits, bytes, and sizes

| Unit | Meaning |
|------|---------|
| **Bit** | One binary digit (0 or 1): the smallest unit of information |
| **Byte** | **8 bits**: the basic unit of memory addressing. One RAM cell = 1 byte |
| **Kilobyte, Megabyte, ...** | Multiples of bytes |

### What can a byte hold?

8 bits → 2⁸ = **256** different patterns:

```
00000000 = 0
00000001 = 1
00000010 = 2
...
11111111 = 255
```

So an unsigned byte (`uint8` / `byte` in Go) holds **0 to 255**. That's where Go's integer ranges (Chapter 2) come from: `int8` = 8 bits (−128 to 127), `int16` = 16 bits, `int32`, `int64`.

More bits → more values: **n bits → 2ⁿ values.**

### Units of size

There are two conventions, and it's helpful to know both:

| Name | Symbol | Decimal (SI, disk vendors) | Binary (IEC, RAM/OS) |
|------|--------|----------------------------|----------------------|
| kilo / kibi | KB / KiB | 1,000 bytes | 1,024 bytes |
| mega / mebi | MB / MiB | 1,000,000 | 1,048,576 |
| giga / gibi | GB / GiB | 1,000,000,000 | 1,073,741,824 |
| tera / tebi | TB / TiB | 10¹² | 2⁴⁰ |

Why 1,024? Because computers count in powers of 2 (2¹⁰ = 1024). That's why a "500 GB" drive shows up as about 465 GiB in some operating systems, and why the tutorial line "8 GB of RAM ≈ 8.59 billion cells" is really 8 GiB = 8 × 1,073,741,824 = **8,589,934,592 bytes**.

Rough sizes to build intuition:

| Thing | Approximate size |
|-------|------------------|
| A text character (English) | 1 byte |
| This chapter as a text file | ~50 KB |
| A photo | 2–5 MB |
| A song (MP3) | 4–8 MB |
| An HD movie | 2–5 GB |
| A Go "Hello World" binary | ~1.5 MB |

### Where you meet this in Go

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	var a int8
	var b int64
	var c float64
	var d string
	fmt.Println(unsafe.Sizeof(a), unsafe.Sizeof(b), unsafe.Sizeof(c), unsafe.Sizeof(d))
	// 1 8 8 16
}
```

---

## 11. How data is represented

Everything is bits. The question is just **which encoding** gives meaning to them.

### Unsigned integers
Plain binary: `00001010` = 10.

### Negative integers: two's complement
Signed integers reserve the leftmost bit to help represent negatives. In 8 bits, `-5` is stored as `11111011` (the bit pattern of 256 − 5). This makes addition work the same for positive and negative numbers.

```go
package main

import "fmt"

func main() {
	var x int8 = -5
	fmt.Printf("%08b\n", uint8(x)) // 11111011
}
```

### Overflow
A fixed number of bits can only hold so many values. Go **wraps around** silently for integers:

```go
package main

import "fmt"

func main() {
	var small int8 = 127 // the max for int8
	small++
	fmt.Println(small) // -128 (wrapped!)

	var u uint8 = 255
	u++
	fmt.Println(u) // 0
}
```

This is a source of real-world bugs. Choose types big enough for your data.

### Text
Characters are stored as **numbers**, via an encoding:

- **ASCII** (1960s): 128 characters (English letters, digits, punctuation), one byte each. `'A'` = 65.
- **Unicode / UTF-8**: covers every language and emoji. UTF-8 uses **1 to 4 bytes per character** (ASCII characters still take 1 byte). Go source files and strings are UTF-8.

```go
package main

import "fmt"

func main() {
	for _, r := range "Hi!" {
		fmt.Printf("%c = %d = %08b\n", r, r, r)
	}
}
```

Output:

```
H = 72 = 01001000
i = 105 = 01101001
! = 33 = 00100001
```

### Decimals (floating point)
Floats store a sign, an exponent, and a fraction in binary (the IEEE 754 standard). Many decimal fractions (like 0.1) have no exact binary representation, hence:

```go
package main

import "fmt"

func main() {
	var a, b float64 = 0.1, 0.2
	fmt.Println(a + b)        // 0.30000000000000004
	fmt.Println(a+b == 0.3)   // false
}
```

### Pictures, sound, video, programs
All the same trick: pixel colors are numbers (e.g., 3 bytes for red/green/blue), sound samples are numbers, and your Go **program** is a sequence of numbers that the CPU decodes as instructions.

> **The big idea:** the hardware doesn't know what the bits *mean*. **Types** and **encodings** (which the compiler and programs track) give the bits meaning. The same 8 bits `01000001` can be the number 65, the letter 'A', or part of a pixel.

---

## 12. The memory hierarchy

CPUs are *much* faster than RAM, and RAM is much faster than disk. To hide the gap, computers use layers. Smaller layers are faster and closer to the CPU:

```
            faster, smaller, costlier
                     ▲
   ┌─────────────────────────────┐
   │  CPU registers   (bytes)    │  ~ 0.3 ns    (immediately usable)
   ├─────────────────────────────┤
   │  L1 cache        (~32 KB)   │  ~ 1 ns
   ├─────────────────────────────┤
   │  L2 cache        (~256 KB–1 MB) │ ~ 4 ns
   ├─────────────────────────────┤
   │  L3 cache        (MBs, shared)  │ ~ 10–40 ns
   ├─────────────────────────────┤
   │  RAM             (GBs)      │  ~ 100 ns
   ├─────────────────────────────┤
   │  SSD             (100s GB)  │  ~ 100,000 ns  (100 µs)
   ├─────────────────────────────┤
   │  HDD             (TBs)      │  ~ 10,000,000 ns (10 ms)
   └─────────────────────────────┘
                     ▼
            slower, larger, cheaper
```

(Numbers are typical orders of magnitude, not exact.)

To feel the scale, **if a register access took 1 second**, an L1 hit would take ~3 seconds, RAM ~5 minutes, an SSD read ~3 days, and an HDD seek ~1 year.

**Consequences for programmers:**

1. **Locality matters.** When the CPU fetches one byte from RAM it actually loads a whole **cache line** (typically 64 bytes) into cache. So reading an array **in order** is fast (neighbors are already loaded), while jumping around memory is slow. (Arrays and slices, which are contiguous, are usually faster than linked lists for this reason.)
2. **Stack access is fast** partly because the top of the stack is "hot" in cache (Chapter 18).
3. **Avoid unnecessary disk and network I/O**; they're orders of magnitude slower than memory. (This is why Go's concurrency model is so valuable while waiting for I/O; Chapters 31+.)

---

## 13. A short history of computing

Why learn history? Because the design choices we live with (binary, programs stored in memory, punch-card-width limits like 80 columns) all have origin stories.

### The abacus (~2700 BC onward)

A frame of beads used in Sumer, China, Rome and elsewhere to do arithmetic. Each bead position represents a value; moving beads performs additions and subtractions. It is arguably the **first calculating tool**: something *external* that holds numbers and helps process them.

> **What is a computer, at heart?** A machine that takes **input**, follows **instructions**, holds **state (memory)**, and produces **output**. By that definition even an abacus (with a human as the "CPU") is a primitive computing device.

### Gottfried Wilhelm Leibniz (1646–1716)

The German mathematician **described the modern binary number system** (in 1679, published 1703): every number can be written with just 0 and 1, and arithmetic works on it. He also built an early mechanical calculator (the *Stepped Reckoner*). Nearly 300 years later, this was exactly what electronic switches (on/off) needed.

### Jacquard's loom (1804)

Joseph Marie Jacquard used **punched cards** to control the pattern woven by a loom: holes = "pass the thread", no hole = "don't". The pattern could be changed by swapping cards. *The first widely used programmable machine.*

### Charles Babbage (1791–1871): "father of the computer"

An English mathematician who designed:
- the **Difference Engine** (1820s): a mechanical calculator for tables of numbers, and
- the **Analytical Engine** (1837): a general-purpose, *programmable* mechanical computer, with a "**store**" (memory), a "**mill**" (processor), input via punched cards, and printed output.

It had the same architecture as modern computers, but **was never completed** in his lifetime (the required precision machining and funding were beyond the era). A working Difference Engine No. 2 was built from his plans in 1991 and it works.

### Ada Lovelace (1815–1852): the first programmer

Working with Babbage, **Ada Lovelace** wrote (1843) what's considered the **first published algorithm intended for a machine** (computing Bernoulli numbers) and foresaw that such machines could manipulate symbols and even music, not just numbers. The programming language *Ada* is named after her.

### Herman Hollerith and punched cards (1890)

Hollerith's punched-card tabulating machine processed the 1890 US census in a fraction of the time. His company later became **IBM**. Punch cards became the standard way to program and feed data to computers well into the 1970s.

### Programming with punch cards

Before keyboards and screens:

```
 ┌───────────────────────────────┐
 │ ○ ● ○ ○ ● ○ ○ ○ ● ● ○ ○ ...   │  ← each column: holes encode one character
 │ ○ ○ ● ○ ○ ○ ● ○ ○ ○ ○ ○ ...   │
 │ ...                           │  (typically 80 columns × 12 rows)
 └───────────────────────────────┘
```

- One card = one line (~80 characters). A program was **a stack of cards**.
- You punched them on a machine, handed the stack to an operator, and **waited hours or days** for the printout.
- A single mistake meant **re-punching the card and reinserting it in exactly the right position**, and dropping a stack of cards and scrambling their order was a legendary disaster (people drew diagonal stripes on the edge of the stack to restore order).

Respect to the early programmers 🙏: when your compiler complains in one second, remember they waited a day.

### The road since

| Era | Milestone |
|-----|-----------|
| 1936–37 | Alan Turing's theoretical "universal machine": the concept of a general-purpose computer |
| 1940s | First electronic computers (Colossus, ENIAC), using vacuum tubes; John von Neumann describes the **stored-program** architecture: programs live in memory alongside data (the model still used today) |
| 1947 | The transistor is invented, replacing vacuum tubes |
| 1950s–60s | Assembly and high-level languages (FORTRAN, COBOL, C's ancestors); operating systems appear (Chapter 27) |
| 1971 | First microprocessor (Intel 4004): a whole CPU on one chip |
| 1970s–80s | Personal computers (Apple II, IBM PC), C, Unix |
| 1990s | The web, Linux, multi-tasking OSes everywhere |
| 2000s | **Multi-core CPUs** arrive, because clock speeds stopped rising. Concurrency becomes essential |
| 2007–2009 | Go is designed at Google **for exactly this multicore, networked world** |

Go's designers wanted a language where programs could **use many cores easily**, which is why Part 14 of this course is all about concurrency.

---

## 14. Look at your own machine

Try these to see the concepts on your own computer.

**Linux**

```bash
lscpu          # CPU model, cores, cache sizes
free -h        # RAM total/used
df -h          # disk space
```

**macOS**

```bash
sysctl -n machdep.cpu.brand_string
sysctl hw.memsize
df -h
```

**Windows (PowerShell)**

```powershell
Get-CimInstance Win32_Processor | Select Name, NumberOfCores
Get-CimInstance Win32_ComputerSystem | Select TotalPhysicalMemory
```

**From Go**

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println("CPU cores available:", runtime.NumCPU())
	fmt.Println("OS/Arch:", runtime.GOOS, runtime.GOARCH)
}
```

On the machine used to verify this chapter that printed `CPU cores available: 12` and `OS/Arch: linux amd64`. You'll use `runtime.NumCPU()` again in Chapters 31 and 36.

---

## 15. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "RAM stores my files" | Files live on disk; RAM holds what's *currently in use* |
| "More RAM makes the CPU faster" | It prevents slowdowns from swapping to disk; it doesn't raise CPU speed |
| "GHz is the only performance measure" | Cores, cache, memory speed, and architecture matter, too |
| "The CPU does only 7 things" | It has hundreds of instructions, but all are simple building blocks |
| "Computers understand text/numbers" | They only manipulate bits; software gives bits meaning |
| "1 KB = 1000 bytes always" | Depends on convention (KB vs KiB) |
| "Deleting a file erases it immediately" | Usually just marks space reusable |
| "A program runs directly from the disk" | It's loaded into RAM first |

---

## 16. Exercises

### Exercise 1: Convert to binary
Convert 19, 64, and 100 to binary by hand. Then check with Go's `%b`.

<details><summary>Solution</summary>

19 = 16 + 2 + 1 → `10011`. 64 = 2⁶ → `1000000`. 100 = 64 + 32 + 4 → `1100100`.

```go
package main

import "fmt"

func main() { fmt.Printf("%b %b %b\n", 19, 64, 100) } // 10011 1000000 1100100
```
</details>

### Exercise 2: Convert from binary
What are `1101`, `10000`, and `11111111` in decimal?

<details><summary>Solution</summary>

`1101` = 8+4+1 = **13**. `10000` = **16**. `11111111` = 128+64+32+16+8+4+2+1 = **255**.
</details>

### Exercise 3: How many values?
How many different values fit in 10 bits? In 3 bytes?

<details><summary>Solution</summary>

10 bits → 2¹⁰ = **1,024**. 3 bytes = 24 bits → 2²⁴ = **16,777,216** (which is why "24-bit color" has about 16.7 million colors).
</details>

### Exercise 4: RAM or disk?
Where is each item stored *while a program is running*: (a) the compiled binary file on your laptop before you run it, (b) the variable `count` inside the running program, (c) the source file `main.go`, (d) the current position of a game character.

<details><summary>Solution</summary>

(a) Disk. (b) RAM. (c) Disk (until you open it in an editor, which loads a copy into RAM). (d) RAM.
</details>

### Exercise 5: Overflow prediction
What does this print?

```go
var x uint8 = 250
for i := 0; i < 10; i++ {
	x++
}
fmt.Println(x)
```

<details><summary>Solution</summary>

`4`. From 250 the values go 251, 252, 253, 254, 255, then wrap to 0, 1, 2, 3, 4 (10 increments in total).
</details>

### Exercise 6: Time scales
If a CPU register access takes 1 second in "human time", how long is a RAM access at ~100× that? Why does this matter for loops over large slices?

<details><summary>Solution</summary>

About 100 seconds (real ratio ≈ 100–300). This is why cache-friendly (sequential) access, like walking a slice in order, is much faster than random access: neighbors are pulled into cache together.
</details>

### Exercise 7 (challenge): Write your own converter
Write `toBinary(n int) string` that converts a non-negative integer to a binary string *without* `strconv` or `%b` (repeated division by 2).

<details><summary>Solution</summary>

```go
package main

import "fmt"

func toBinary(n int) string {
	if n == 0 {
		return "0"
	}
	bits := ""
	for n > 0 {
		bits = fmt.Sprint(n%2) + bits // prepend the remainder
		n /= 2
	}
	return bits
}

func main() {
	fmt.Println(toBinary(22), toBinary(255), toBinary(0)) // 10110 11111111 0
}
```
</details>

---

## 17. Quiz

1. Which component executes instructions?
2. Which loses its contents when power is off: RAM or disk?
3. How many bits in a byte? How many values can a byte hold?
4. Why do computers use binary?
5. What is the fetch-decode-execute cycle?
6. Who is often called the "first programmer"? Who designed the Analytical Engine?
7. Why is reading a slice sequentially faster than random access?

<details><summary>Answers</summary>

1. The CPU.
2. RAM.
3. 8 bits; 256 values.
4. Transistors reliably have two states (on/off), making binary simple and dependable.
5. The repeated loop where the CPU fetches the next instruction from memory, decodes it, and executes it.
6. Ada Lovelace; Charles Babbage.
7. Sequential access uses cache lines, since neighboring data is loaded together, and avoids slow trips to RAM.
</details>

---

## 18. Summary

- A programmer's three key components: **CPU** (executes), **RAM** (fast, volatile working memory), **disk** (slower, permanent storage). Programs are copied from disk into RAM to run.
- RAM is a long row of **byte-sized cells with addresses**: the addresses that pointers hold.
- The CPU runs the **fetch-decode-execute** cycle billions of times a second, combining simple **arithmetic, logic, data-movement and jump** instructions into everything a computer does.
- Everything is **binary**: bits (0/1), bytes (8 bits = 256 values). Types and encodings (two's complement, ASCII/UTF-8, IEEE 754) give bits meaning; fixed sizes cause **overflow**.
- The **memory hierarchy** (registers → cache → RAM → disk) trades size for speed; **locality** makes contiguous data (arrays, slices) fast.
- History: abacus → Leibniz's binary → Jacquard/Babbage/Lovelace → Hollerith's cards → transistors → microprocessors → **multi-core**, which is the world Go was designed for.

### ➡️ What's next?

The hardware alone isn't enough: something must share the CPU and RAM among many programs. [Chapter 27](27-introduction-to-operating-systems.md) introduces **operating systems** and the birth of automation.
