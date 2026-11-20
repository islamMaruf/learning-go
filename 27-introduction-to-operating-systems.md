# Chapter 27: Introduction to Operating Systems — The Birth of Automation

> **Goal of this chapter:** Understand *why* operating systems exist by living through the problem they solved. You'll travel back to the 1950s, see how early computers were run by hand, why that broke down, and how the **operating system (OS)** was invented to automate it. Then we'll look at what an OS does today and how Go programs talk to it.

**Difficulty:** 🟡 Beginner-friendly  **Estimated time:** 1.5–2 hours  **Prerequisite:** [Chapter 26](26-computer-architecture-and-a-short-history-of-computing.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Time travel to the 1950s](#2-time-travel-to-the-1950s)
3. [How a program ran back then](#3-how-a-program-ran-back-then)
4. [The first programmers](#4-the-first-programmers)
5. [The human operator problem](#5-the-human-operator-problem)
6. [The one-program-at-a-time problem](#6-the-one-program-at-a-time-problem)
7. [Why it couldn't continue](#7-why-it-couldnt-continue)
8. [The birth of the operating system](#8-the-birth-of-the-operating-system)
9. [What is an operating system?](#9-what-is-an-operating-system)
10. [What an OS does](#10-what-an-os-does)
11. [Kernel, user space, and system calls](#11-kernel-user-space-and-system-calls)
12. [Batch → multiprogramming → time-sharing](#12-from-batch-to-time-sharing)
13. [Today's operating systems](#13-todays-operating-systems)
14. [How Go programs meet the OS](#14-how-go-programs-meet-the-os)
15. [Common misconceptions](#15-common-misconceptions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- What computing looked like **before** operating systems
- The concrete problems (wasted time, manual work, one program at a time) that made an OS necessary
- What an OS is, in one sentence, and its **five main jobs**
- The difference between the **kernel**, **user space**, and a **system call**
- How Go programs use the OS behind the scenes

The story matters: once you feel the problem, the solution (and later, processes, threads and goroutines) is obvious rather than arbitrary.

---

## 2. Time travel to the 1950s

Imagine it's about 1950. Computers exist, but they look nothing like yours:

- **ENIAC** (1945): filled a large room, about 30 tons, built with roughly 18,000 vacuum tubes.
- **IBM 701** (1952): IBM's first commercial scientific computer.

Compared to your laptop, these machines had:

| Component | 1950s computer |
|-----------|----------------|
| CPU | ✅ Yes (built from vacuum tubes) |
| Memory | ✅ Yes, but tiny (a few thousand "words", i.e., a few KB) |
| Hard disk | ❌ Not yet available (magnetic tape and drums came slightly later) |
| Keyboard | ❌ No |
| Monitor | ❌ No |
| Input | **Punched cards** (Chapter 26) or paper tape |
| Output | **Punched cards** or a **printer** |

They were also **absurdly expensive** (millions of today's dollars), so every minute of machine time was precious.

---

## 3. How a program ran back then

Suppose you wanted to compute a table of numbers. This is what you did:

```
Step 1: WRITE the program on paper.
Step 2: PUNCH it, one card per line (100 lines = 100 cards).
Step 3: Carry the stack of cards to the computer room and hand it to an OPERATOR.
Step 4: WAIT. (Your job is queued behind others.)
Step 5: The operator loads your cards into the reader.
Step 6: The computer runs your program (and only yours).
Step 7: The operator collects the printout and puts it in your pigeonhole.
Step 8: You read the output. If there was one mistake: go back to Step 1.
```

```
   You             Operator           Computer
    │                 │                   │
    │── card deck ───►│                   │
    │  (hours later)  │── load cards ────►│
    │                 │                   │── run ──┐
    │                 │◄── printout ──────│◄────────┘
    │◄── results ─────│                   │
```

A single typo could cost you a **whole day** (or more) of waiting to find out. Debugging was an act of patience.

---

## 4. The first programmers

Historians note that many of the earliest programmers were **women**. ENIAC's original programming team was six women (Kay McNulty, Betty Jennings, Betty Snyder, Marlyn Wescoff, Fran Bilas, and Ruth Lichterman). ENIAC had no stored programs: they physically **rewired cables and set switches** to "program" it, working from wiring diagrams, with no manual or programming language to guide them.

Grace Hopper (who later created one of the first compilers, and helped bring us COBOL) worked on Harvard's Mark I and UNIVAC. Programming was widely seen as clerical work, but it was actually high-skill logical engineering, and this workforce laid the foundations for the discipline. It's a reminder that programming began as *problem solving*, not as a keyboard hobby.

---

## 5. The human operator problem

The **operator** was a human employee whose job was to keep the machine busy: load card decks, mount tapes, restart after failures, and distribute printouts.

A day in the computer room:

```
9:00  Operator loads Alice's program → computer runs for 5 min
9:05  Computer waits while the operator walks to the shelf for Bob's deck
9:08  Operator loads Bob's deck → run 3 min
9:11  Computer idle: operator changes the paper in the printer
9:20  Alice's program crashed due to a typo → the operator must reload Alice's fixed deck later
...
```

Problems:

| Problem | Effect |
|---------|--------|
| **Humans are slow** (walking, loading, mounting) | The multi-million-dollar CPU sat **idle** most of the time |
| **Humans make mistakes** | Decks dropped, loaded in the wrong order, wrong tapes mounted |
| **Setup time** | Each job needed setup (load the compiler, then the program, then the data), often longer than the run itself |
| **Human availability** | Nights, breaks, shift changes |
| **Scale** | More users = more chaos |

The CPU (millions of operations per second) was being throttled by human speed (one action per few seconds).

---

## 6. The one-program-at-a-time problem

A second, deeper problem: the machine could run **only one program at a time**.

Imagine today's typical use: you have a browser, a music player, a code editor, and a chat app open. Your CPU switches among them so smoothly it *feels* simultaneous.

In the 1950s:

- Program A starts → **runs to completion**. Nobody else can use the machine.
- If Program A is waiting for the printer or the card reader (very slow devices), the **CPU is idle**: it can't help with Program B because B isn't loaded.

```
Time  ──────────────────────────────────────────────────►
CPU:  [ A computes ][.. idle, waiting for card reader ..][ A computes ][.. idle ..]
                     ▲ the CPU is wasted here
```

Slow input/output devices meant that most of the time the expensive CPU did nothing. This is the same "CPU is much faster than I/O" idea from the memory hierarchy (Chapter 26), just far more extreme.

There was also **no protection**: a buggy program could overwrite the memory of another, or hang the machine forever, and there was no supervisor to stop it.

---

## 7. Why it couldn't continue

Put the problems together:

```
   ┌──────────────────────┐   ┌────────────────────────┐   ┌───────────────────────┐
   │ Humans do repetitive │ + │ CPU sits idle while    │ + │ One program at a time,│
   │ manual work (slow,   │   │ waiting for slow I/O   │   │ no protection between │
   │ error-prone)         │   │                        │   │ programs              │
   └──────────────────────┘   └────────────────────────┘   └───────────────────────┘
                                          │
                                          ▼
                       Enormous waste of the world's most expensive machines
```

Everyone agreed something had to change. And the change was this insight:

> **"The work the human operator does is mechanical. A program can do it."**

---

## 8. The birth of the operating system

The solution: write a **special, permanent program that stays in memory and manages the computer**, doing the operator's job and more. That program is the **operating system (OS)**.

### Milestones

| Year | System | What it introduced |
|------|--------|--------------------|
| ~1955–56 | **GM-NAA I/O** (General Motors / North American Aviation, on the IBM 704) | Often called the **first operating system**: automatically ran one job after another (**batch processing**) |
| ~1957 | IBM's **IBSYS**, and others | Job control languages and standard I/O routines |
| early 1960s | **Atlas**, **CTSS** | **Multiprogramming** (several programs in memory at once) and **time-sharing** (many users interactively) |
| 1969–71 | **Unix** (Bell Labs, Ken Thompson & Dennis Ritchie, who later co-created... Ken Thompson co-created Go!) | The design most modern operating systems descend from |
| 1980s | MS-DOS, Mac OS | Personal-computer operating systems |
| 1991 | **Linux** | Open-source Unix-like kernel |

(Fun fact connecting to this course: **Ken Thompson**, a co-creator of Unix, is also one of the three creators of the **Go** language.)

### What the first OSes automated

```
BEFORE (human operator):             AFTER (operating system):
  operator loads card deck             OS reads the next job automatically
  operator starts the run              OS starts it
  operator collects printout           OS spools the output to the printer
  operator loads the next deck         OS immediately begins the next job
```

Human delays vanished. The CPU stayed busy.

---

## 9. What is an operating system?

> An **operating system** is the software that **manages the computer's hardware** (CPU, memory, storage, devices) and **provides services** to the programs that run on it.

Two roles in one:

1. **Resource manager (the manager / referee).** Decides *which program gets the CPU, how much memory each gets, who may use the disk or network*, so programs don't collide.
2. **Abstraction layer (the translator / hotel concierge).** Hides ugly hardware details behind clean interfaces. Your Go program says "open the file `notes.txt`". It doesn't need to know if it's on an SSD, HDD, or network drive, or how its bytes are laid out on the platter.

```
┌───────────────────────────────────────────────┐
│  Applications: your Go program, browser, ...  │
├───────────────────────────────────────────────┤
│  Operating system                             │  ← manager + abstraction
│  (Linux / Windows / macOS)                    │
├───────────────────────────────────────────────┤
│  Hardware: CPU, RAM, disk, network, screen    │
└───────────────────────────────────────────────┘
```

> **Analogy: a busy restaurant's manager.** Customers (programs) come in wanting tables (memory), the chef's time (CPU), and ingredients (files). The manager seats them, keeps order, makes sure nobody takes everyone else's food, and coordinates the kitchen. Customers don't need to know how the kitchen works.

---

## 10. What an OS does

Five core jobs (each of which we'll touch in later chapters):

### 1. Process management

Starts programs, tracks each running program as a **process**, and decides **who runs on the CPU and when**. It switches quickly among processes, creating the illusion of doing many things at once (multitasking). *(Chapters 28–31.)*

### 2. Memory management

Gives each process its **own private memory** space and prevents one process from reading or overwriting another's. It uses **virtual memory**: each process thinks it has the whole address space to itself; the OS maps those virtual addresses onto real RAM (and can swap rarely-used memory to disk). *(This is why the stack/heap/code layout in Chapter 18 is the same for every program.)*

### 3. File system management

Organizes disk storage into files and folders, tracks permissions, and reads/writes data on request. Your program says `os.Open("data.txt")`. The OS handles the rest.

### 4. Device management (drivers)

Talks to keyboards, screens, disks, network cards, and printers through **device drivers**, so programs use a uniform interface.

### 5. Security and protection

Users and permissions ("this file is only readable by its owner"), isolating programs from each other, and stopping misbehaving processes. It's the difference between one crashed program and a crashed computer.

Plus: networking (sockets, TCP/IP), a **shell/GUI** for you to interact with, timers, and more.

### Summary table

| Job | Question the OS answers |
|-----|------------------------|
| Processes | Who runs next? For how long? |
| Memory | Who gets which memory? Can I touch that address? |
| Files | Where's `notes.txt`? May this user read it? |
| Devices | How do I make the screen/disk/network card do X? |
| Security | Is this action allowed? |

---

## 11. Kernel, user space, and system calls

### The kernel

The **kernel** is the **core** of the OS: the always-loaded part with **full control of the hardware**. When people say "the Linux kernel" they mean that core. "Linux" the operating system also includes shells, tools, libraries, and so on around it.

### Two modes of execution

CPUs support (at least) two privilege levels:

| | **Kernel mode** ("ring 0") | **User mode** ("ring 3") |
|--|---------------------------|--------------------------|
| Who runs here | The OS kernel | Your programs (Go binaries, browsers, ...) |
| Power | Can touch any memory and any device | Restricted: can only touch **its own** memory |
| If it crashes | The whole machine may crash | Only that program dies |

This is what makes protection possible: **an ordinary program cannot directly touch hardware or other programs' memory.** The CPU itself enforces it.

### System calls

So how does your program read a file if it can't touch the disk? It **asks the kernel**, using a **system call** ("syscall"): a controlled doorway from user mode into kernel mode.

```
   YOUR PROGRAM (user mode)                 KERNEL (kernel mode)
   ┌─────────────────────┐                  ┌──────────────────────────┐
   │ os.ReadFile("a.txt")│                  │                          │
   │      │              │   system call    │  1. check permissions    │
   │      └──── read() ──┼─────────────────►│  2. find the file on disk│
   │                     │                  │  3. read bytes into RAM  │
   │   ◄── data ─────────┼──────────────────│  4. give them back       │
   └─────────────────────┘                  └──────────────────────────┘
```

Common system calls (Linux names): `open`, `read`, `write`, `close`, `fork`/`clone`, `exec`, `mmap` (memory), `socket`, `exit`. Each switch into the kernel has a cost (Chapter 30 explains context switches), which is one reason Go's runtime does its own lightweight scheduling on top (Chapter 36).

On Linux you can watch a program's system calls with `strace ./yourprogram`.

---

## 12. From batch to time-sharing

The OS story continues in three big steps.

### Batch processing

Jobs are collected into a **batch** (a tray of card decks or a tape). The OS runs them **one after another** automatically. Removes human setup time, but the CPU still idles during I/O, and you still wait a long time for results.

### Multiprogramming

Keep **several programs in memory at once**. When Program A waits for I/O, the OS **switches the CPU to Program B**:

```
Time ───────────────────────────────────────────────────►
CPU: [ A ][ B ][ A ][ C ][ B ][ A ]...    ← the CPU is always busy
A:   run   wait  run       wait  run
B:         run         run
C:                   run
```

CPU utilization shoots up. This creates the need for **memory protection** and for a **scheduler**. Both were new inventions of this stage.

### Time-sharing

Switch between programs **so quickly** (every few milliseconds) that each *interactive* user feels they have the whole machine. This gave us terminals, then modern desktops: you type, and something responds immediately. The mechanism (**context switching**) is the subject of Chapter 30.

```
Evolution:   manual operator  →  batch  →  multiprogramming  →  time-sharing  →  today's multitasking
             (idle CPU)          (no human   (CPU busy during      (interactive,        (many cores, many
                                  delay)      I/O)                  responsive)          processes and threads)
```

---

## 13. Today's operating systems

| Family | Examples | Common use |
|--------|----------|------------|
| **Linux** | Ubuntu, Debian, Fedora, Alpine, Android (Linux kernel) | Servers, cloud, containers, embedded, Android phones |
| **Windows** | Windows 10/11, Windows Server | Desktops, enterprise |
| **macOS / iOS** | macOS, iOS, iPadOS (Unix-based Darwin) | Apple devices |
| **BSD** | FreeBSD, OpenBSD | Servers, networking gear |
| **Real-time OSes** | FreeRTOS, VxWorks | Embedded devices, medical, aerospace |

Most **backend servers, including the Go web applications you'll build in this course, run on Linux**. Go compiles natively for all of the above (recall cross-compilation in Chapter 19).

---

## 14. How Go programs meet the OS

You use the OS every time you touch the outside world. Go's standard library wraps system calls in friendly packages:

| You write | The OS does (roughly) |
|-----------|----------------------|
| `fmt.Println("hi")` | `write` syscall to standard output |
| `os.ReadFile("a.txt")` | `open` + `read` + `close` |
| `os.Args`, `os.Getenv("PORT")` | Info the OS passed to your process at start |
| `net.Listen("tcp", ":8080")` | `socket`, `bind`, `listen` |
| `time.Sleep(time.Second)` | Ask the OS/runtime to wake the goroutine later |
| `go func() {...}()` | Go's runtime schedules a goroutine on OS **threads** (Chapters 32, 36) |

Try this:

```go
package main

import (
	"fmt"
	"os"
	"runtime"
)

func main() {
	fmt.Println("Process ID (assigned by the OS):", os.Getpid())
	fmt.Println("Parent process ID:", os.Getppid())
	fmt.Println("Operating system:", runtime.GOOS)
	fmt.Println("CPU cores the OS exposes:", runtime.NumCPU())

	host, err := os.Hostname()
	if err == nil {
		fmt.Println("Hostname:", host)
	}

	// Ask the OS for a file; it may say no.
	_, err = os.ReadFile("/definitely/not/here.txt")
	fmt.Println("Reading a missing file:", err)
}
```

Sample output (values vary):

```
Process ID (assigned by the OS): 48213
Parent process ID: 48190
Operating system: linux
CPU cores the OS exposes: 12
Hostname: my-laptop
Reading a missing file: open /definitely/not/here.txt: no such file or directory
```

That last line is the **OS speaking**: the kernel's answer ("no such file") relayed through Go's `error` value, exactly the `(result, error)` idea from Chapter 5. Each time you run the program it gets a different **process ID (PID)**: the number the OS uses to track it. Processes are the topic of the next chapter.

---

## 15. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "The OS is just the desktop/GUI" | The GUI is one small part; the kernel (process, memory, file, device management) is the heart |
| "Windows/macOS/Linux are all the same program" | Different kernels and designs, though they offer similar services |
| "Linux is an OS by itself" | Strictly, Linux is a **kernel**; distributions (Ubuntu...) bundle it with other software |
| "My program talks directly to the hardware" | It goes through the OS via system calls |
| "The OS only matters to OS developers" | Every performance, security, and concurrency behavior you'll debug involves it |
| "Multitasking means the computer does many things at the *exact* same moment" | On one core it *switches* very fast; true simultaneity needs multiple cores (Chapter 31) |
| "The first OS was Windows" | Operating systems date from the mid-1950s |

---

## 16. Exercises

### Exercise 1: Explain the problem
In 3–4 sentences, explain to a friend why 1950s computers had idle CPUs.

<details><summary>Solution</summary>

Input and output devices (card readers, printers) and human operators were far slower than the CPU. Because only one program could be loaded at a time, the CPU had to wait, idle, while cards were read, decks were swapped, and results were printed. Since the computers were extremely expensive, this idle time was very costly.
</details>

### Exercise 2: Match the job
Match each task to an OS job (process, memory, file, device, security): (a) deciding which program runs next on the CPU, (b) preventing program A from reading program B's variables, (c) saving `report.txt`, (d) sending data to the printer, (e) checking that a user may delete a file.

<details><summary>Solution</summary>

(a) Process management. (b) Memory management (and protection). (c) File system. (d) Device management (driver). (e) Security/permissions.
</details>

### Exercise 3: Kernel or user?
For each, say whether the code runs in user mode or kernel mode: (a) your `for` loop adding numbers, (b) the code that reads bytes from the SSD, (c) `fmt.Println` formatting text, (d) the scheduler picking the next process.

<details><summary>Solution</summary>

(a) user, (b) kernel (via a driver), (c) user (formatting), then a `write` syscall enters the kernel to output it, (d) kernel.
</details>

### Exercise 4: Timeline
Draw the CPU timeline for two programs A and B that each compute for 2 ms, then wait 6 ms for I/O, then compute 2 ms. Compare (i) running one after the other with no multiprogramming and (ii) multiprogramming.

<details><summary>Solution</summary>

(i) Sequential: A: 2 run + 6 idle + 2 run = 10 ms; then B: 10 ms → 20 ms total, with CPU busy only 8 of 20 ms (40%).
(ii) Multiprogramming: A runs 0–2, then B runs 2–4 while A waits; A's I/O finishes at 8, B's at 10; A finishes 8–10, B finishes 10–12 → ~12 ms total, CPU busy 8 of 12 ms (67%). Same work, in 40% less time.
</details>

### Exercise 5: Observe the OS from Go
Write a program that prints its PID, then sleeps 30 seconds. While it sleeps, find it from another terminal (`ps -p <pid>` on Linux/macOS, Task Manager on Windows).

<details><summary>Solution</summary>

```go
package main

import (
	"fmt"
	"os"
	"time"
)

func main() {
	fmt.Println("My PID is", os.Getpid())
	time.Sleep(30 * time.Second)
}
```
In a second terminal: `ps -p <pid> -o pid,ppid,stat,cmd` shows the OS's process table entry for your program.
</details>

### Exercise 6 (challenge): Two views of one call
Explain in order what happens, from your Go code to the disk and back, when you call `data, err := os.ReadFile("notes.txt")`.

<details><summary>Solution</summary>

1. `os.ReadFile` (user mode) makes system calls (`open`, `read`, `close`).
2. The CPU switches to kernel mode; the kernel checks that the file exists and that your user has permission (security + file system).
3. The file system code finds the file's blocks; the device driver asks the disk to read them.
4. The kernel copies the bytes into your process's memory (RAM), then returns to user mode.
5. `ReadFile` returns `(data, nil)`, or `(nil, err)` if any step failed (e.g., "permission denied").
</details>

---

## 17. Quiz

1. Name two problems of early "operator-driven" computing.
2. What's the one-sentence definition of an OS?
3. What is the kernel?
4. Why can't ordinary programs touch hardware directly? What do they use instead?
5. What is multiprogramming, and what problem does it solve?
6. What OS do most Go web servers run on?

<details><summary>Answers</summary>

1. e.g., idle CPU due to slow humans and devices; one program at a time; manual, error-prone setup.
2. Software that manages hardware resources and provides services to programs.
3. The privileged core of the OS with full hardware control.
4. Protection (user mode is restricted); they use **system calls** to ask the kernel.
5. Keeping several programs in memory and switching the CPU to another when one waits for I/O, which raises CPU utilization.
6. Linux.
</details>

---

## 18. Summary

- Early computers (1940s–50s) ran **one program at a time**, loaded **by human operators** from punched cards, so the **expensive CPU sat idle** and mistakes were common.
- The fix was to automate the operator's job with a resident program: the **operating system** (first examples around 1955–56).
- An OS is a **resource manager** (CPU, memory, files, devices, security) and an **abstraction layer** hiding hardware details.
- Its five jobs: **processes, memory, files, devices, security**.
- The **kernel** runs in **privileged mode**; your programs run in **user mode** and reach the kernel only through **system calls**.
- OSes evolved: **batch → multiprogramming → time-sharing → modern multitasking**. Each step kept the CPU busier and users happier.
- Go programs use the OS through the standard library (`os`, `net`, `time`, ...) and every running program is a **process** with a PID.

### ➡️ What's next?

The OS's central abstraction is the **process**. [Chapter 28](28-breaking-the-cpu-and-understanding-the-process.md) opens up the CPU and explains what a process really is, in memory and on the CPU.
