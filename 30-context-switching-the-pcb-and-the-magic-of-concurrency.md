# Chapter 30: Context Switching, the PCB, and the Magic of Concurrency

> **Goal of this chapter:** Answer the question *"How can a computer run dozens of programs when it has only a few CPU cores?"* You'll learn how fast CPUs really are compared to human perception, what a **context switch** is, what the operating system stores in a **Process Control Block (PCB)**, how the switch works step by step, and what it costs. You'll even measure context switches on your own machine.

**Difficulty:** 🔴 Intermediate–Advanced (conceptual)  **Estimated time:** 2.5–3 hours  **Prerequisite:** [Chapters 27–29](27-introduction-to-operating-systems.md)

---

## 📚 Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [The mystery of many programs](#2-the-mystery-of-many-programs)
3. [How fast is a CPU, really?](#3-how-fast-is-a-cpu-really)
4. [How fast is a human?](#4-how-fast-is-a-human)
5. [The sequential problem](#5-the-sequential-problem)
6. [The idea: context switching](#6-the-idea-context-switching)
7. [The Process Control Block (PCB)](#7-the-process-control-block-pcb)
8. [A context switch, step by step](#8-a-context-switch-step-by-step)
9. [Process states and the scheduler](#9-process-states-and-the-scheduler)
10. [When does a switch happen?](#10-when-does-a-switch-happen)
11. [The cost of context switching](#11-the-cost-of-context-switching)
12. [Measure it yourself](#12-measure-it-yourself)
13. [Concurrency vs. parallelism (a first look)](#13-concurrency-vs-parallelism)
14. [Why this matters for Go](#14-why-this-matters-for-go)
15. [Common misconceptions](#15-common-misconceptions)
16. [Exercises](#16-exercises)
17. [Quiz](#17-quiz)
18. [Summary](#18-summary)

---

## 1. What you will learn

- Why a computer with 4 cores can seem to run 400 programs at once
- The scale gap between CPU speed (nanoseconds) and human perception (milliseconds)
- What a **context switch** is, and how it uses the registers from Chapter 28
- What's stored in a **PCB** and why
- The states a process moves through (**ready, running, waiting**)
- The performance cost of switching, with real measurements from Go
- How the *same* idea, one level up, makes **goroutines** possible

---

## 2. The mystery of many programs

Right now, your computer is probably running: a browser with many tabs, an editor, a music player, a chat app, background services, and (soon) your Go programs. That's **hundreds of processes**.

But your CPU has maybe **4, 8, or 12 cores**, and **each core executes only one instruction stream at any instant**. (Chapter 28: one PC, one set of registers per core.)

```
Processes:  Browser  Editor  Music  Chat  Go app  Updater  ...  (hundreds)
                 ╲     │       │     │     ╱
                  ╲    │       │     │    ╱
CPU cores:         [ Core 1 ] [ Core 2 ] [ Core 3 ] [ Core 4 ]
```

How can hundreds of programs make progress on four cores, *while music plays smoothly and you type without lag*? That's the puzzle. The answer combines two facts: **CPUs are absurdly fast**, and **humans are absurdly slow**.

---

## 3. How fast is a CPU, really?

A modern CPU runs at about **3 GHz**: **3 billion cycles per second**. A simple instruction takes roughly one cycle, and modern CPUs finish several instructions per cycle. So a single core executes on the order of **billions of instructions every second**.

| Time | What a 3 GHz core can do |
|------|--------------------------|
| 1 second | ~3,000,000,000 cycles |
| 1 millisecond (1/1000 s) | ~3,000,000 cycles |
| 1 microsecond (1/1,000,000 s) | ~3,000 cycles |
| 1 nanosecond (1/1,000,000,000 s) | ~3 cycles |

In **one millisecond**, a core can run *millions* of instructions.

**Scale it to human time:** if one CPU cycle were **1 second**, then a millisecond of CPU time would be **about 35 days** of that "slowed-down life". Lots can happen.

---

## 4. How fast is a human?

Human perception is slow by comparison:

- The eye and brain fuse images into continuous motion at roughly **24–60 frames per second**, so a change lasting **~16–40 milliseconds** is the smallest we notice as separate.
- A typical **reaction time** to a visual signal is about **200 milliseconds**.
- A very fast typist presses maybe 10 keys per second: **100 ms** between keys.
- A "smooth" user interface aims to respond within **~100 ms**, beyond which we feel a lag.

So a human notices delays of tens of milliseconds, while a CPU can do **millions of instructions** in a single millisecond.

```
Human perception:  |----------------- ~50 ms -----------------|
CPU:               |a|b|c|d|e|f|g|h|i|j|k|l|m|n|o|p|q|r|s|t|... (hundreds of millions of cycles)
```

There's a big gap, and it's exactly the gap the operating system exploits.

---

## 5. The sequential problem

Suppose the CPU ran programs **one after another** ("sequentially"):

```
Time ────────────────────────────────────────────────────────►
CPU:  [ Music player (plays 3 min) ][ Browser ][ Editor ]...
```

- Your song would play *first*. Until it ends, the browser would be frozen, and you couldn't type.
- Worse: even inside the music player, most of the time is spent **waiting**: for the disk to deliver the next chunk of audio, or for the sound card to finish playing what it has. During those waits the CPU would sit **idle**.

(Recall from Chapter 27: this exact waste is what drove the invention of operating systems.)

We need the CPU to be **shared**.

---

## 6. The idea: context switching

> **Context switching:** the OS runs a process for a very short time, then **pauses it**, **saves** its state, **loads** another process's state, and runs *that* one; over and over, in rotation.

Each process gets a small **time slice** (also called a **quantum**), typically a few **milliseconds**. Because the slices are so short, and each process's state is saved and restored perfectly, **every process appears to run continuously**, even on a single core.

```
Time ─────────────────────────────────────────────────────────►
Core 1: [ Music ][ Browser ][ Editor ][ Music ][ Browser ][ Editor ][ Music ]...
          ~ms       ~ms        ~ms       ~ms      ~ms        ~ms
```

### Real-life analogy: chatting with several people 💬

You're texting five friends. You don't finish a conversation with one before starting the next. You write a line to Asha, switch to Rahim, reply to Bina, back to Asha... Each conversation "pauses" while you're elsewhere, but **each person sees an ongoing chat**. To make this work you must **remember where each conversation left off** (the *context*): what was asked, what you were about to say.

**That memory is what the OS saves and restores.**

### What is the "context"?

The **context** of a process is everything the CPU needs to *resume it exactly where it stopped*, which is mostly the **CPU register values** from Chapter 28:

- **PC**: which instruction comes next
- **SP, BP**: where its stack and current frame are
- **General-purpose registers**: half-computed values
- **Flags**: results of the last comparison
- (and pointers to its memory: its page tables)

Save these, and you can pause a process for a millisecond or a minute. Restore them, and it continues as if nothing happened, unaware it was ever paused.

---

## 7. The Process Control Block (PCB)

Where does the OS keep each process's saved state? In a data structure called the **Process Control Block (PCB)**, one per process. (In Linux it's the `task_struct`.) Think of it as the process's **file folder** in the OS's cabinet.

```
┌─────────────────────── PCB for process 4211 ────────────────────────┐
│ PROCESS ID (PID)         : 4211                                     │
│ PARENT PID               : 4190                                     │
│ STATE                    : ready                                    │
│ PRIORITY                 : normal                                   │
│                                                                     │
│ SAVED CPU CONTEXT                                                   │
│    PC  (program counter) : 0x004A5DE3                               │
│    SP  (stack pointer)   : 0xC000090DF8                             │
│    BP  (base pointer)    : 0xC000090E10                             │
│    general registers     : AX=22, BX=12, CX=0, ...                  │
│    flags                 : ...                                      │
│                                                                     │
│ MEMORY INFORMATION       : where its code/data/heap/stack live      │
│                            (page tables)                            │
│ OPEN FILES               : stdin, stdout, stderr, notes.txt, ...    │
│ ACCOUNTING               : CPU time used, memory used               │
│ OWNER (user), PERMISSIONS: ...                                      │
└─────────────────────────────────────────────────────────────────────┘
```

| PCB field | Why the OS needs it |
|-----------|---------------------|
| **PID / parent PID** | Identify the process and its family tree |
| **State** | Ready? Running? Waiting? (section 9) |
| **Saved registers (PC, SP, BP, ...)** | To **resume** exactly where it left off |
| **Memory info** | To give it back its own private address space |
| **Open files / resources** | To keep its file handles and sockets intact |
| **Priority / scheduling info** | To decide who runs next |
| **Accounting** | To measure CPU usage (`top`, Task Manager) |

The PCB is the *bookmark* that makes pausing and resuming possible.

---

## 8. A context switch, step by step

Two processes, **A** (running now) and **B** (ready and waiting), share one core.

```
                 RAM
    ┌───────────────┐   ┌───────────────┐   ┌─────────────────────┐
    │ Process A     │   │ Process B     │   │ OS kernel           │
    │ code/data/    │   │ code/data/    │   │  PCB_A   PCB_B      │
    │ stack/heap    │   │ stack/heap    │   │  scheduler          │
    └───────────────┘   └───────────────┘   └─────────────────────┘
                      CPU: PC/SP/BP/regs → currently A's values
```

**Step 1: A is running.** The CPU registers hold A's values (PC points into A's code, SP into A's stack).

**Step 2: an event forces a switch.** Either A's time slice expires (a **timer interrupt** fires), or A must wait for something (a disk read). Control jumps into the **kernel**.

**Step 3: save A's context.** The kernel copies all CPU registers into **PCB_A**. PCB_A's state becomes *ready* (if it was preempted) or *waiting* (if it asked for I/O).

```
CPU registers ──copy──► PCB_A  (PC=0x4A5DE3, SP=..., BP=..., AX=22 ...)
```

**Step 4: the scheduler picks the next process.** Say B. (Rules in section 9.)

**Step 5: load B's context.** The kernel copies **PCB_B's** saved values *into* the CPU registers, and switches the memory mapping to B's address space.

```
PCB_B ──copy──► CPU registers  (PC=0x7100, SP=..., BP=..., AX=5 ...)
```

**Step 6: resume B.** The kernel returns to user mode. The CPU fetches from B's PC, and B continues as if it had never stopped, with its own stack, its own registers, and its own variables intact.

```
        save A                       load B
CPU ──────────────► PCB_A     PCB_B ──────────────► CPU
   (Step 3)                       (Step 5)           (now running B)
```

Later, another switch saves B's context in **PCB_B** and reloads **PCB_A**, so A resumes at *exactly* the instruction where it was paused. Because the PC, SP, BP, and all registers are restored, A never notices.

**This is why understanding registers (Chapter 28) and SP/BP (Chapter 29) mattered:** they *are* the context.

---

## 9. Process states and the scheduler

A process isn't always eligible to run. It's in one of several **states**:

```
                   admitted
   ┌─────┐  ─────────────────►  ┌───────┐   scheduler picks    ┌─────────┐
   │ NEW │                      │ READY │ ───────────────────► │ RUNNING │
   └─────┘                      └───┬───┘ ◄─────────────────── └────┬────┘
                                    ▲     time slice expired        │
                                    │     (preempted)               │ waits for I/O,
                                    │                               │ a lock, a timer...
                                    │  I/O finished          ┌──────▼─────┐
                                    └─────────────────────── │  WAITING   │
                                                             │ (blocked)  │
                                                             └────────────┘
                                          exit → TERMINATED
```

| State | Meaning |
|-------|---------|
| **New** | Being created |
| **Ready** | Could run, but waiting for a CPU core |
| **Running** | Currently executing on a core |
| **Waiting / Blocked** | Can't proceed until something happens (disk read done, network data arrives, `time.Sleep` ends) |
| **Terminated** | Finished; OS is cleaning up |

The **scheduler** is the kernel component that picks which *ready* process runs next, using a **scheduling policy**. Common ideas:

| Policy | Idea |
|--------|------|
| **Round robin** | Fixed time slices in a circle: fair and simple |
| **Priority** | Important processes (e.g., the audio player) go first |
| **Multilevel feedback** | Interactive processes get quick turns; CPU hogs get lower priority |
| **Completely Fair Scheduler (Linux)** | Tries to give every runnable process a fair share of CPU time |

A waiting process **uses no CPU**. That's a key optimization: while your music player waits for the next audio chunk from disk, the CPU runs something else. This is what recovers the idle time we saw in the 1950s (Chapter 27).

---

## 10. When does a switch happen?

| Trigger | Kind | Example |
|---------|------|---------|
| **Time slice expires** (timer interrupt) | **Involuntary** (preemption) | A process in a long `for` loop is forced to give up the CPU |
| **The process blocks** | **Voluntary** | Waiting for disk/network, `time.Sleep`, waiting on a lock or channel |
| **A higher-priority process becomes ready** | Involuntary | You press a key, and the editor process wakes and preempts a background job |
| **The process exits or yields** | Voluntary | `os.Exit`, `runtime.Gosched()` (at the Go level) |

The key mechanism for preemption is the **hardware timer**: the CPU is set to raise an interrupt every few milliseconds, forcing control into the kernel, no matter what user code is doing. That's how one badly written infinite loop can't freeze the whole machine.

---

## 11. The cost of context switching

Context switching is *not free*, and this matters for performance:

| Cost | Why |
|------|-----|
| **Direct**: saving/loading registers, switching memory maps, running the scheduler | roughly **1–5 microseconds** (varies widely) |
| **Indirect**: **cache pollution.** Process B's data isn't in the CPU caches yet (they're full of A's data), so B starts **slow** ("cold cache") until it reloads its working set from RAM | Often the bigger cost |
| **TLB flush**: address translation caches must be invalidated when the address space changes | Adds more slowdown |
| **Kernel entry/exit**: mode switches user ↔ kernel | Adds up if thousands per second |

If the time slice is **too short**, the CPU spends all its time switching, not working (called **thrashing**). If it's **too long**, the system feels sluggish. Operating systems tune the quantum (a few milliseconds) to balance responsiveness against overhead.

Rough sense of scale: with ~2 µs per switch and 5 ms slices, switching costs around **0.04%** of the time, negligible. But with **100,000 processes/threads** each demanding tiny slices, it would dominate. That is why creating one OS thread per connection stops scaling, and why **Go built its own, much cheaper switching** (section 14).

---

## 12. Measure it yourself

Linux tracks context switches per process. This program measures its own:

```go
package main

import (
	"fmt"
	"os"
	"strings"
	"time"
)

func ctxt() string {
	data, _ := os.ReadFile("/proc/self/status")
	var out []string
	for _, l := range strings.Split(string(data), "\n") {
		if strings.Contains(l, "ctxt_switches") {
			out = append(out, strings.Join(strings.Fields(l), " "))
		}
	}
	return strings.Join(out, " | ")
}

func main() {
	fmt.Println("start:  ", ctxt())

	for i := 0; i < 100; i++ {
		time.Sleep(time.Millisecond) // block: give up the CPU voluntarily
	}
	fmt.Println("sleeps: ", ctxt())

	end := time.Now().Add(500 * time.Millisecond)
	for time.Now().Before(end) { // busy loop: never gives up the CPU
	}
	fmt.Println("busy:   ", ctxt())
}
```

Real output from one run (yours will differ slightly):

```
start:   voluntary_ctxt_switches: 2 | nonvoluntary_ctxt_switches: 0
sleeps:  voluntary_ctxt_switches: 102 | nonvoluntary_ctxt_switches: 0
busy:    voluntary_ctxt_switches: 102 | nonvoluntary_ctxt_switches: 26
```

Read the story in the numbers:

- After **100 `time.Sleep` calls** the *voluntary* count rose by exactly **100**: each sleep blocked, so the process gave up the CPU by choice.
- The **busy loop** never blocked, yet the *involuntary* count rose by **26** in half a second: the OS **forcibly preempted** it (timer interrupt) so others could run, about every 20 ms on this machine (other processes were competing).

You just watched context switching happen. On macOS/Windows use the Activity Monitor/Task Manager or `top`/`vmstat` (Linux: `vmstat 1` shows system-wide `cs` = context switches per second, typically thousands).

---

## 13. Concurrency vs. parallelism

We can now separate two words that are often confused ([Chapter 31](31-concurrency-vs-parallelism.md) treats them fully):

- **Concurrency**: *dealing with* many things at once. Multiple tasks are **in progress** over the same period, possibly by rapidly **switching on one core** (what we just described). It's about *structure*.
- **Parallelism**: *doing* many things at the same instant, on **multiple cores**. It's about *execution*.

```
Concurrency on ONE core (time-slicing):        Parallelism on TWO cores:
Core 1: A B A B A B                            Core 1: A A A A A A
                                               Core 2: B B B B B B
(interleaved: only one runs at any instant)    (truly simultaneous)
```

Context switching gives **concurrency**. Multiple cores add **parallelism**. Real machines use both at once.

---

## 14. Why this matters for Go

Everything in this chapter repeats one level up inside Go:

| Operating system | Go runtime |
|------------------|-----------|
| Runs **processes** / threads on CPU cores | Runs **goroutines** on OS threads |
| **Context switch** saves/restores registers, switches memory maps | Goroutine switch saves/restores just a few registers (PC, SP, BP) in a tiny struct |
| **PCB** stores a process's context | A **`g` struct** stores a goroutine's context |
| **Scheduler** in the kernel | Go's own **scheduler** in user space |
| Costs ~1–5 µs, plus cache effects | Costs **~100–200 ns**, with no kernel entry |
| A process/thread costs MBs of memory | A goroutine starts with a **~2–8 KB** stack |

Let's measure Go's switch cost. This program bounces a value between two goroutines through unbuffered channels with only one OS thread allowed to run Go code, so every hand-off is a goroutine context switch:

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	runtime.GOMAXPROCS(1) // one core for Go code: switches must be goroutine switches
	ping, pong := make(chan int), make(chan int)

	go func() {
		for v := range ping {
			pong <- v
		}
	}()

	const N = 1_000_000
	start := time.Now()
	for i := 0; i < N; i++ {
		ping <- i
		<-pong
	}
	el := time.Since(start)
	fmt.Printf("%d round trips in %v => about %v per switch\n", N, el.Round(time.Millisecond), el/(2*N))
}
```

Real output on one machine:

```
1000000 round trips in 302ms => about 151ns per switch
```

**About 150 nanoseconds per goroutine switch**, roughly **10–30× cheaper** than an OS thread switch, and it needs no trip into the kernel. That is why a Go program can run **hundreds of thousands** of goroutines comfortably. You'll see how in Chapters 32–36 and use them heavily in Chapters 64–70.

---

## 15. Common misconceptions

| Misconception | Reality |
|---------------|---------|
| "A 4-core CPU can only run 4 programs" | It can run thousands of *processes* by rapid switching (only 4 execute at any instant) |
| "Context switching is instantaneous/free" | It has real direct and cache costs |
| "The process decides when to be paused" | Usually the **OS** decides, using timer interrupts (preemption) |
| "A waiting process still uses CPU" | A blocked process uses **no** CPU until its event happens |
| "Smaller time slices are always better" | Too small and switching overhead dominates |
| "The PCB is part of the process's own memory" | It lives in **kernel** memory, so the process can't tamper with it |
| "Concurrency = parallelism" | Concurrency is about structuring; parallelism needs multiple cores (Chapter 31) |
| "The process notices being paused" | It doesn't: its registers are restored exactly |

---

## 16. Exercises

### Exercise 1: Number sense
A CPU runs at 2 GHz. How many cycles in 5 ms? If a context switch costs 4,000 cycles, what fraction of a 5 ms slice is spent switching?

<details><summary>Solution</summary>

2 GHz = 2,000,000,000 cycles/s → in 5 ms: 10,000,000 cycles. 4,000 / 10,000,000 = 0.04%.
</details>

### Exercise 2: What's in the PCB?
List six pieces of information the OS stores per process. Which are needed *specifically* to resume the process exactly where it stopped?

<details><summary>Solution</summary>

PID, state, saved registers (PC, SP, BP, general registers, flags), memory map, open files, priority/scheduling info, accounting, owner. Needed to *resume exactly*: **saved registers** (especially PC, SP, BP) and the **memory mapping**.
</details>

### Exercise 3: Trace a switch
Process A is running, B is ready. A's time slice expires. Write the 5 steps the OS takes to switch to B.

<details><summary>Solution</summary>

1. Timer interrupt → CPU enters the kernel. 2. Save A's registers into PCB_A; mark A *ready*. 3. Scheduler picks B. 4. Load B's registers from PCB_B and switch to B's address space; mark B *running*. 5. Return to user mode; B continues at its saved PC.
</details>

### Exercise 4: Voluntary or involuntary?
Classify: (a) a program calls `time.Sleep(1s)`, (b) a `for {}` loop runs for 200 ms and is stopped by the OS, (c) a web server waits for a request on a socket, (d) a video player is interrupted because the keyboard driver woke a higher-priority task.

<details><summary>Solution</summary>

(a) voluntary, (b) involuntary, (c) voluntary (blocking), (d) involuntary.
</details>

### Exercise 5: Observe it
Write a Go program that runs a busy loop for 2 seconds and prints its `nonvoluntary_ctxt_switches` before and after (Linux). Then run *two copies at once* on a machine with few cores. What do you expect?

<details><summary>Solution</summary>

Use the `ctxt()` helper from section 12. With more busy processes than cores, each is preempted more often, so the involuntary count rises faster (each competes for the CPU). With a free core, the count stays low, since there's no competition and no reason to preempt.
</details>

### Exercise 6: Design question
Why is it bad for the OS to give each process a time slice of 10 microseconds? Why bad for 10 seconds?

<details><summary>Solution</summary>

10 µs: switching costs (µs) would be a large fraction of each slice (thrashing), and caches never warm up, so little useful work. 10 s: other processes would starve, so the system would feel frozen (the human perception threshold is tens of milliseconds).
</details>

### Exercise 7 (challenge): Compare
Run the goroutine ping-pong program on your machine. Then write a version with `GOMAXPROCS(2)` and compare. Which is faster per round trip, and why might the answer be surprising?

<details><summary>Solution</summary>

With more than one P, the two goroutines may run on different OS threads/cores, so each hand-off may need to wake a sleeping thread via the kernel (futex), often making the round trip **slower** than the single-P case, where the hand-off is a plain goroutine switch. Concurrency ≠ speed: parallelism has coordination costs. (Chapter 31 explores this.)
</details>

---

## 17. Quiz

1. What is a context switch?
2. What does the OS save when it pauses a process? Where?
3. Why does a timer interrupt matter?
4. Which process state uses no CPU while waiting for I/O?
5. Name two costs of context switching.
6. What's cheaper: an OS process switch or a goroutine switch, and roughly by how much?

<details><summary>Answers</summary>

1. Saving the running process's CPU state and loading another's so the CPU runs a different process.
2. Its registers/context (PC, SP, BP, general registers, flags...) in its **PCB** in kernel memory.
3. It forces control into the kernel at regular intervals, enabling preemptive multitasking.
4. *Waiting/blocked*.
5. Direct save/restore + scheduler time; cache/TLB pollution; kernel entry.
6. A goroutine switch; roughly 10–30× cheaper (~150 ns vs several µs).
</details>

---

## 18. Summary

- CPUs execute **millions of instructions per millisecond**; humans notice delays of **tens of ms**. That gap lets the OS **share** a core among many processes.
- **Context switching**: run a process for a short **time slice**, **save** its state, **load** another's, repeat. Each process appears to run continuously.
- The saved state (the **context**) is mainly the **registers** (PC, SP, BP, general registers, flags), stored in the process's **PCB** in kernel memory, along with PID, state, memory map, open files, priority, and accounting.
- Processes move between **ready, running, waiting**. The **scheduler** picks who runs next; **waiting processes use no CPU**.
- Switches are **voluntary** (blocking) or **involuntary** (timer-driven preemption). You can count both in `/proc/self/status`.
- Switching has **direct and indirect (cache) costs**: too-short slices thrash, too-long slices feel laggy.
- **Concurrency** (interleaving) is what context switching gives you; **parallelism** (simultaneous) needs multiple cores.
- Go repeats the same idea at a much cheaper scale: **goroutines** switch in ~150 ns instead of microseconds.

### ➡️ What's next?

[Chapter 31](31-concurrency-vs-parallelism.md) makes the concurrency-vs-parallelism distinction precise, and shows how the multi-core revolution changed how software must be written.
