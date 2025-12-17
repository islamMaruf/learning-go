# CPU Architecture Tutorial for CSE Students
## Breaking Down the CPU and Processor

### 📚 Table of Contents
1. [Introduction](#introduction)
2. [CPU Architecture Overview](#cpu-architecture-overview)
3. [Processing Unit](#processing-unit)
4. [Register Set](#register-set)
5. [Fetch-Execute Cycle](#fetch-execute-cycle)
6. [Practical Examples](#practical-examples)
7. [Summary](#summary)

---

## Introduction

The **Central Processing Unit (CPU)** is the brain of a computer. It executes instructions, performs calculations, and manages data flow between different components. Understanding CPU architecture is fundamental for any Computer Science student, especially when studying computer organization, assembly language, or low-level programming.

**Key Learning Objectives:**
- Understand the main components of a CPU
- Learn how instructions are fetched and executed
- Understand the role of registers in computation
- Comprehend binary execution at the hardware level

---

## CPU Architecture Overview

The CPU consists of three primary components:

```
┌─────────────────────────────────────┐
│              CPU                     │
│  ┌──────────────────────────────┐  │
│  │    Processing Unit           │  │
│  │  ┌───────────────────────┐   │  │
│  │  │  ALU                  │   │  │
│  │  │  (Arithmetic Logic    │   │  │
│  │  │   Unit)               │   │  │
│  │  └───────────────────────┘   │  │
│  │  ┌───────────────────────┐   │  │
│  │  │  Control Unit (CU)    │   │  │
│  │  └───────────────────────┘   │  │
│  └──────────────────────────────┘  │
│  ┌──────────────────────────────┐  │
│  │    Register Set              │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
```

**Important**: All operations within the CPU happen in **binary format** (0s and 1s).

---

## Processing Unit

The Processing Unit contains two critical sub-components:

### 1. Arithmetic Logic Unit (ALU)

The ALU is responsible for performing all arithmetic and logical operations.

#### Arithmetic Operations:
- **Addition** (`+`): Adds two numbers
- **Subtraction** (`-`): Subtracts one number from another
- **Multiplication** (`*`): Multiplies two numbers
- **Division** (`÷`): Divides one number by another

#### Logical Operations:
- **AND** (`&`): Bitwise AND operation
- **OR** (`|`): Bitwise OR operation
- **NOT** (`!`): Bitwise NOT operation (inversion)

**Example:**
```
Addition in Binary:
  0101  (5 in decimal)
+ 0011  (3 in decimal)
------
  1000  (8 in decimal)

Logical AND:
  1010  (10 in decimal)
& 1100  (12 in decimal)
------
  1000  (8 in decimal)
```

### 2. Control Unit (CU)

The Control Unit is the coordinator of the CPU. It manages the flow of data and instructions.

**Primary Functions:**
1. **Fetch** instructions from memory using the Programme Counter (PC)
2. **Decode** the instruction to understand what operation to perform
3. **Execute** the instruction by directing the appropriate components
4. **Store** the instruction temporarily in the Instruction Register (IR)

**Control Flow:**
```
Programme Counter → Fetch Instruction → Instruction Register
                          ↓
                    Decode Instruction
                          ↓
                    Execute via ALU/Memory
```

**Instruction Register (IR)**: A special register that holds the current instruction being executed.

---

## Register Set

Registers are small, high-speed storage locations within the CPU. They temporarily hold data, addresses, and instructions during processing.

### Special Purpose Registers

These registers have specific dedicated functions:

#### 1. Programme Counter (PC)
- **Purpose**: Points to the address of the next instruction to be executed
- **Functionality**: Automatically increments after each instruction fetch
- **Also known as**: Instruction Pointer (IP)

**Example:**
```
Address    Instruction
0x0000  →  LOAD AL, 5     ← PC points here
0x0001     ADD AL, 3
0x0002     STORE AL, 0x100
```
After fetching the instruction at 0x0000, PC increments to 0x0001.

#### 2. Stack Pointer (SP)
- **Purpose**: Points to the top of the stack in memory
- **Functionality**: Decrements when data is pushed, increments when data is popped
- **Usage**: Function calls, local variables, return addresses

**Stack Example:**
```
High Memory
    ↑
    │   [Empty]
    │   [Value C]  ← SP points here (top of stack)
    │   [Value B]
    │   [Value A]
    ↓
Low Memory
```

#### 3. Base Pointer (BP)
- **Purpose**: Points to the base of the current stack frame
- **Functionality**: Provides a fixed reference point for accessing local variables and parameters
- **Also known as**: Frame Pointer (FP)

### General Purpose Registers (8-bit Computer)

These registers can be used for various purposes by the programmer:

#### 1. Accumulator Register (AL)
- **Size**: 8 bits (can hold values 0-255)
- **Primary Use**: Stores results of arithmetic and logical operations
- **Most frequently used register for calculations**

**Example:**
```
MOV AL, 10    ; Load 10 into AL
ADD AL, 5     ; AL = AL + 5 (AL now contains 15)
```

#### 2. Base Register (BL)
- **Size**: 8 bits
- **Primary Use**: Often used for base addressing or temporary storage
- **Can be used in indexed addressing modes**

#### 3. Counter Register (CL)
- **Size**: 8 bits
- **Primary Use**: Loop counters, shift operations, rotation counts
- **Automatically decremented in loop instructions**

**Example:**
```
MOV CL, 10    ; Set loop counter to 10
LOOP_START:
    ; Loop body
    DEC CL    ; Decrement counter
    JNZ LOOP_START  ; Jump if not zero
```

#### 4. Data Register (DL)
- **Size**: 8 bits
- **Primary Use**: General data storage, I/O operations
- **Often used in data transfer operations**

---

## Fetch-Execute Cycle

The CPU operates in a continuous cycle known as the **Fetch-Execute Cycle** or **Instruction Cycle**.

### Step-by-Step Process:

```
┌─────────────────────────────────────────┐
│  1. FETCH                               │
│     PC → Memory Address                 │
│     Instruction → IR                    │
│     PC = PC + 1                         │
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│  2. DECODE                              │
│     Control Unit analyzes IR            │
│     Determines operation and operands   │
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│  3. EXECUTE                             │
│     ALU performs operation              │
│     Or data is moved between registers  │
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│  4. STORE (if needed)                   │
│     Result written to register/memory   │
└────────────┬────────────────────────────┘
             ↓
           Repeat
```

### Detailed Example:

**Program:**
```assembly
Address    Instruction         Binary Representation
0x0000     MOV AL, 5          10110000 00000101
0x0002     ADD AL, 3          00000100 00000011
0x0004     MOV BL, AL         10001000 11000011
```

**Execution Trace:**

**Cycle 1:**
- **Fetch**: PC = 0x0000, IR = `10110000 00000101` (MOV AL, 5), PC → 0x0002
- **Decode**: Control Unit identifies: "Move immediate value 5 to AL"
- **Execute**: AL ← 5
- **Store**: AL now contains 00000101 (5 in binary)

**Cycle 2:**
- **Fetch**: PC = 0x0002, IR = `00000100 00000011` (ADD AL, 3), PC → 0x0004
- **Decode**: Control Unit identifies: "Add immediate value 3 to AL"
- **Execute**: ALU computes AL + 3 = 5 + 3 = 8
- **Store**: AL now contains 00001000 (8 in binary)

**Cycle 3:**
- **Fetch**: PC = 0x0004, IR = `10001000 11000011` (MOV BL, AL), PC → 0x0006
- **Decode**: Control Unit identifies: "Copy AL to BL"
- **Execute**: BL ← AL
- **Store**: BL now contains 00001000 (8 in binary)

---

## Practical Examples

### Example 1: Simple Addition Program

**High-Level Code (C-like):**
```c
int a = 10;
int b = 20;
int sum = a + b;
```

**Assembly Equivalent:**
```assembly
MOV AL, 10      ; Load 10 into AL
MOV BL, 20      ; Load 20 into BL
ADD AL, BL      ; Add BL to AL, result in AL
MOV [0x100], AL ; Store result in memory location 0x100
```

**Register State:**
```
Initial:  AL = 00000000, BL = 00000000
Step 1:   AL = 00001010 (10), BL = 00000000
Step 2:   AL = 00001010 (10), BL = 00010100 (20)
Step 3:   AL = 00011110 (30), BL = 00010100 (20)
Step 4:   Memory[0x100] = 00011110 (30)
```

### Example 2: Loop Counter

**High-Level Code:**
```c
for (int i = 0; i < 5; i++) {
    // Do something
}
```

**Assembly Equivalent:**
```assembly
MOV CL, 5           ; Initialize counter to 5
LOOP_START:
    ; Loop body instructions here
    DEC CL          ; Decrement counter (CL = CL - 1)
    JNZ LOOP_START  ; Jump if CL ≠ 0
```

### Example 3: Logical Operations

**Task**: Check if a number is even (last bit = 0)

```assembly
MOV AL, 10          ; Load number (00001010)
AND AL, 1           ; AND with 00000001
JZ IS_EVEN          ; Jump if zero (even number)
JMP IS_ODD          ; Otherwise odd

IS_EVEN:
    ; Even number handling
    
IS_ODD:
    ; Odd number handling
```

**Explanation:**
- `10 AND 1` = `00001010 AND 00000001` = `00000000` (Result is 0, so 10 is even)
- `11 AND 1` = `00001011 AND 00000001` = `00000001` (Result is 1, so 11 is odd)

---

## Summary

### Key Takeaways:

1. **CPU Components:**
   - **ALU**: Performs arithmetic (`+`, `-`, `*`, `÷`) and logical operations (`&`, `|`, `!`)
   - **Control Unit**: Orchestrates fetch-decode-execute cycle
   - **Registers**: Fast temporary storage for data and instructions

2. **Special Purpose Registers:**
   - **PC**: Tracks next instruction address
   - **SP**: Manages stack operations
   - **BP**: Provides stack frame reference
   - **IR**: Holds current instruction

3. **General Purpose Registers (8-bit):**
   - **AL**: Accumulator for calculations
   - **BL**: Base register
   - **CL**: Counter register
   - **DL**: Data register

4. **Critical Concepts:**
   - All CPU operations occur in **binary format**
   - Instructions are fetched from memory, decoded, and executed sequentially
   - Registers provide high-speed access compared to main memory
   - The fetch-execute cycle is the fundamental operation of any processor

### Further Study:

- **Assembly Language Programming**: Practice writing low-level code
- **Computer Organization**: Study how different components interact
- **Digital Logic Design**: Understand how ALU operations are implemented using logic gates
- **Modern CPU Architecture**: Learn about pipelining, cache, and parallel processing
- **Instruction Set Architectures**: Compare RISC vs CISC designs

### Practice Questions:

1. What happens to the Programme Counter after each instruction fetch?
2. Why are registers faster than main memory?
3. How would you implement subtraction if your ALU only had addition and NOT operations?
4. Trace the fetch-execute cycle for: `MOV AL, 15` followed by `MUL AL, 2`
5. What's the maximum value an 8-bit register can hold in unsigned representation?

**Answer to Q5**: $2^8 - 1 = 255$ (range: 0 to 255)

---

## Additional Resources

- **Books**: "Computer Organization and Design" by Patterson & Hennessy
- **Practice**: Try x86 assembly programming with tools like NASM or MASM
- **Simulators**: Use CPU simulators to visualize instruction execution
- **Videos**: Look for CPU architecture visualization videos

---

*Tutorial based on Class 34 notes - CPU and Processor Architecture*
