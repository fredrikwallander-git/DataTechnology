# Day 2: CPU Architecture - How Computers Think

## 🎯 Learning Objectives

- How logic gates combine to form functional units (adders, latches, flip-flops)
- What registers are and how they differ from RAM and cache
- The CPU's internal organization (ALU, registers, control unit, cache)
- The fetch-decode-execute cycle that runs all programs
- Instruction sets and assembly language basics
- The compilation pipeline from C# to native machine code
- CPU performance features (pipelining, multiple cores, out-of-order execution)
- How caching works and why cache misses matter

---

## 📚 From Gates to Circuits

### **Recap:**

- Binary (0 and 1)
- Basic gates (AND, OR, NOT, XOR)
- How transistors implement gates

**Now:** We combine these gates into **functional units** that do real work!

---

### **What Are Adders?**

An **adder** is a digital circuit built from logic gates that performs binary addition.

**Why we need adders:**
- All arithmetic in computers starts with addition
- Subtraction = adding negative numbers (Two's Complement)
- Multiplication = repeated addition
- Division = repeated subtraction

**Without adders, computers couldn't do math!**

---

### **Half Adder - Adding Two Single Bits**

**What it is:**  
A circuit that adds two 1-bit binary numbers and produces a 2-bit result.

**Why "half"?**  
It can only add two bits. It doesn't handle a carry-in from a previous addition, so it's incomplete (hence "half").

**The Problem:**
```
When we add two 1-bit numbers:
0 + 0 = 00 (sum=0, carry=0)
0 + 1 = 01 (sum=1, carry=0)
1 + 0 = 01 (sum=1, carry=0)
1 + 1 = 10 (sum=0, carry=1)  ← Result needs 2 bits!
```

**The Circuit:**
```
Inputs: A, B (two 1-bit numbers)
Outputs: Sum, Carry (the 2-bit result)

Logic:
Sum = A XOR B    (1 if inputs are different)
Carry = A AND B  (1 if both inputs are 1)

Truth Table:
A │ B │ Sum │ Carry │ Decimal
──┼───┼─────┼───────┼────────
0 │ 0 │  0  │   0   │ 0 + 0 = 0
0 │ 1 │  1  │   0   │ 0 + 1 = 1
1 │ 0 │  1  │   0   │ 1 + 0 = 1
1 │ 1 │  0  │   1   │ 1 + 1 = 2 (binary: 10)

Gate Implementation:
A ──┬──[XOR]── Sum
    │
B ──┼──[AND]── Carry
    │
```

**What it does:**  
Takes two bits, adds them, outputs a sum bit and a carry bit. That's it!

**Limitation:**  
Can't chain them together to add multi-bit numbers because it doesn't accept a carry-in from the previous position.

---

### **Full Adder - Adding Three Bits**

**What it is:**  
A circuit that adds three 1-bit numbers: A, B, and a Carry-In from a previous addition.

**Why "full"?**  
It's complete! It can handle carry-in, so we can chain multiple full adders together to add numbers of any size.

**The Problem:**
```
When adding multi-bit numbers, we need to include the carry from the previous column:

Example: 11 + 11 = 110
Position 0: 1 + 1 = 10      (sum=0, carry=1)
Position 1: 1 + 1 + carry(1) = 11 (sum=1, carry=1)
Position 2: 0 + 0 + carry(1) = 1  (sum=1, carry=0)

We need to add THREE bits: A + B + Carry-In
```

**The Circuit:**
```
Inputs: A, B, Carry_In (three 1-bit numbers)
Outputs: Sum, Carry_Out (the result)

Built from TWO half adders + one OR gate:

Step 1: Add A and B using Half Adder 1
       A ──┐
           ├──[Half Adder 1]── Sum1, Carry1
       B ──┘

Step 2: Add Sum1 and Carry_In using Half Adder 2
  Sum1 ────┐
           ├──[Half Adder 2]── Sum (final output)
Carry_In ──┘                     Carry2

Step 3: Combine carries with OR gate
Carry_Out = Carry1 OR Carry2
(If either half adder produced a carry, we have a carry out)

Complete Logic:
Sum = (A XOR B) XOR Carry_In
Carry_Out = (A AND B) OR (Carry_In AND (A XOR B))
```

**What it does:**  
Takes three bits (two data bits plus a carry from previous position), adds them, outputs sum and carry-out.

**Why we need it:**  
This is the building block for adding numbers of ANY size!

---

### **4-Bit Ripple-Carry Adder - Real Addition**

**What it is:**  
Four full adders chained together to add two 4-bit numbers.

**How it works:**
```
Adding two 4-bit numbers: 1101 (13) + 0011 (3)

    A₃B₃  A₂B₂  A₁B₁  A₀B₀  ← Input bits
      ↓↓    ↓↓    ↓↓    ↓↓
     [FA3] [FA2] [FA1] [FA0]  ← Full Adders
      ↑C    ↑C    ↑C    ↑C    ← Carry connections
      │     │     │     └──────── Carry starts at 0
      │     │     │
    C_out  S₃    S₂    S₁    S₀  ← Output bits

Position 0 (rightmost):
  A₀ = 1, B₀ = 1, Carry_In = 0
  Sum₀ = 0, Carry_Out = 1  (1+1=10)

Position 1:
  A₁ = 0, B₁ = 1, Carry_In = 1
  Sum₁ = 0, Carry_Out = 1  (0+1+1=10)

Position 2:
  A₂ = 1, B₂ = 0, Carry_In = 1
  Sum₂ = 0, Carry_Out = 1  (1+0+1=10)

Position 3 (leftmost):
  A₃ = 1, B₃ = 0, Carry_In = 1
  Sum₃ = 0, Carry_Out = 1  (1+0+1=10)

Final Result: 10000 (16 in binary) = 1 0000
              ↑ ↑↑↑↑
          overflow  sum bits
```

**Why "Ripple-Carry"?**  
The carry "ripples" from right to left through each adder like a wave. Position 0 must complete before position 1 can, which must complete before position 2, etc.

**Speed:**
- Each full adder has a small delay (~1 nanosecond)
- 4-bit addition: ~4 nanoseconds total (carry must ripple through all 4)
- 64-bit addition: ~64 nanoseconds

**This is how your CPU adds integers!** Modern CPUs use faster designs (carry-lookahead adders), but the principle is the same.

---

### **Sequential Circuits: Memory from Gates**

So far, all our circuits are **combinational** - output depends only on current inputs.

**Sequential circuits** have **memory** - output depends on current inputs AND previous state.

---

### **What is a Latch?**

A **latch** is the simplest memory circuit - it can store one bit indefinitely.

**Definition:**  
A circuit with two stable states (0 or 1) that "latches" onto a value and holds it until explicitly changed.

**Key characteristic:**  
Uses **feedback** - output loops back to input, creating memory.

---

### **SR Latch - Set-Reset Latch**

**SR Latch:** Set-Reset Latch

**What "SR" stands for:**
- **S** = Set (force output to 1)
- **R** = Reset (force output to 0)

**What it is:**  
The most basic memory circuit, built from two NOR gates with cross-coupled feedback.

**The Circuit:**
```
Set ───┬───[NOR]───┬─── Q (Output)
       │           │
       └───────────┼─── Q' (Inverted output)
                   │
Reset ───────[NOR]─┘

Feedback: Output of each NOR feeds into the other
```

**How it works:**
```
Operation Table:
S │ R │ Q │ Q' │ Meaning
──┼───┼───┼────┼─────────────────
0 │ 0 │ Q │ Q' │ HOLD (memory state)
0 │ 1 │ 0 │ 1  │ RESET (store 0)
1 │ 0 │ 1 │ 0  │ SET (store 1)
1 │ 1 │ ? │ ?  │ INVALID (don't do this!)

Key behavior:
When S=0 and R=0: Output stays whatever it was before
This is MEMORY!
```

**Example sequence:**
```
Time │ S │ R │ Q │ Action
─────┼───┼───┼───┼────────────────
  0  │ 0 │ 0 │ 0 │ (Initial state)
  1  │ 1 │ 0 │ 1 │ SET to 1
  2  │ 0 │ 0 │ 1 │ HOLD (still 1)
  3  │ 0 │ 1 │ 0 │ RESET to 0
  4  │ 0 │ 0 │ 0 │ HOLD (still 0)

Notice: When both inputs are 0, output doesn't change!
The latch "remembers" the last SET or RESET.
```

**What it does:**  
Stores one bit. Set input to 1 to store "1", Reset input to 1 to store "0", both to 0 to keep current value.

**This is the foundation of computer memory!**

---

### **What is a Flip-Flop?**

A **flip-flop** is an improved latch with a **clock** input that controls when data is stored.

**Definition:**  
A synchronous memory element that captures input data on a clock edge (0→1 or 1→0 transition).

**Flip-Flop vs Latch:**
```
Latch:
- Level-sensitive (responds to input while enable is high)
- Can change output multiple times while enabled
- Asynchronous (no clock)

Flip-Flop:
- Edge-sensitive (responds only on clock edge)
- Changes output once per clock pulse
- Synchronous (uses clock)
```

**How many bits does a flip-flop store?**  
**ONE bit** - just like a latch, but with controlled timing.

---

### **D Flip-Flop - Data Flip-Flop**

**D Flip-Flop:** Data Flip-Flop

**What it is:**  
The most common type of flip-flop. It captures the input data (D) and stores it when the clock edge occurs.

**The Circuit:**
```
Data (D) ──────┐
               ├──[Flip-Flop]── Q (Output)
Clock ─────────┘

Only captures D when Clock transitions 0→1 (rising edge)
```

**Operation:**
```
Time │ D │ Clock │ Q │ Action
─────┼───┼───────┼───┼────────────────────
  0  │ 0 │   0   │ X │ (Q holds previous value)
  1  │ 1 │   0   │ X │ (Q still holds previous)
  2  │ 1 │  0→1  │ 1 │ CAPTURE D (clock edge!)
  3  │ 0 │   1   │ 1 │ (D changed, but Q unchanged)
  4  │ 0 │  1→0  │ 1 │ (Only rising edge matters)
  5  │ 0 │  0→1  │ 0 │ CAPTURE D again

Key: Q only changes on rising clock edge (0→1)
```

**Why use a clock?**
- **Synchronization:** All flip-flops in a CPU update at the same time
- **Prevents glitches:** Data only captured at controlled moments
- **Enables pipelining:** Can break complex operations into stages

**What it does:**  
Stores one bit, but only updates on clock edges. This is the building block of registers, cache, and RAM.

**How many make a byte?**  
8 D flip-flops = 1 byte of storage  
64 D flip-flops = 1 register in a 64-bit CPU

---

### **🎮 Game Dev Connection:**

```csharp
// Every variable is stored in flip-flops somewhere:
int playerHealth = 100;  
// 32 D flip-flops in a register or RAM storing: 01100100

// When CPU executes "playerHealth = 100":
// 1. Control unit sends "store" signal
// 2. Value 100 appears on data lines
// 3. Clock edge triggers
// 4. 32 flip-flops capture the bits
// 5. Value stored until next clock edge
```

---

## 📚 Building Memory from Logic

### **From Flip-Flops to Registers**

**Registers are just groups of flip-flops!**

```
1 D Flip-Flop = 1 bit
8 D Flip-Flops = 1 byte = 8-bit register
16 D Flip-Flops = 1 word = 16-bit register
32 D Flip-Flops = 1 dword = 32-bit register
64 D Flip-Flops = 1 qword = 64-bit register (modern CPUs)
```

**Example: 8-bit Register**
```
        D₇  D₆  D₅  D₄  D₃  D₂  D₁  D₀  ← 8 data inputs
         │   │   │   │   │   │   │   │
Clock ──┬┴───┴───┴───┴───┴───┴───┴───┴─┐
        │                              │
       [D] [D] [D] [D] [D] [D] [D] [D] │  ← 8 D Flip-Flops
        │   │   │   │   │   │   │   │  │
        Q₇  Q₆  Q₅  Q₄  Q₃  Q₂  Q₁  Q₀ │  ← 8 output bits
        └──────────────────────────────┘

All 8 bits captured simultaneously on clock edge
This stores one 8-bit number (0-255)
```

---

## 📚 CPU Registers and Cache

### **What Are Registers?**

**Register:** A small amount of ultra-fast storage built directly into the CPU chip.

**What registers are:**
- Groups of flip-flops (typically 64 in modern CPUs)
- Located on the CPU die itself (not separate chips)
- Connected directly to the ALU with very short wires
- The CPU's "working memory" for active calculations

**What registers are NOT:**
- Not RAM (RAM is separate chips outside CPU)
- Not cache (cache is larger, slightly slower)
- Not the stack

---

### **Are Registers Part of the Stack?**

**No! Registers and the stack are completely separate.**

```
┌─────────────────────────────────┐
│           CPU Chip              │
│                                 │
│  ┌──────────────┐               │
│  │  Registers   │ ← 16-32 slots │
│  │  (16 × 64bit)│ ← ~128 bytes  │
│  └──────┬───────┘               │
│         ↓                       │
│  ┌──────────────┐               │
│  │     ALU      │               │
│  └──────────────┘               │
│                                 │
└─────────────────────────────────┘
         ↓ (very fast connection)
┌──────────────────────────────────┐
│         RAM Chip                 │
│                                  │
│  ┌──────────────┐                │
│  │  Stack       │ ← Grows/shrinks│
│  │  (in RAM)    │ ← Megabytes    │
│  └──────────────┘                │
│  ┌──────────────┐                │
│  │  Heap        │                │
│  └──────────────┘                │
│                                  │
└──────────────────────────────────┘

Registers: CPU's scratch paper (fast, tiny)
Stack: Function call data (slower, larger, in RAM)
```

**How they interact:**
```assembly
; Push register onto stack
push rax        ; Stack ← rax (copy from register to RAM)

; Pop stack into register
pop rbx         ; rbx ← Stack (copy from RAM to register)

Registers hold working values
Stack stores values we need to remember but aren't using right now
```

---

### **Are Registers Dedicated to Certain Operations?**

**Mostly no, but with some special cases:**

**General-Purpose Registers (can be used for anything):**
```
x86-64 has 16 general-purpose registers:
rax, rbx, rcx, rdx, rsi, rdi, rbp, rsp, r8-r15

You can use most of them for any calculation:
mov rax, 42      ; rax = 42
add rbx, rax     ; rbx = rbx + rax
imul rcx, rdx    ; rcx = rcx × rdx

Not truly "dedicated" to specific operations
```

**Convention-based usage (not enforced, but traditional):**
```
rax: Often holds return values from functions
rcx: Often used as a counter in loops
rsi: Source index (string/array operations)
rdi: Destination index (string/array operations)
rbp: Base pointer (stack frame)
rsp: Stack pointer (MUST point to top of stack)
```

**Actually dedicated registers:**
```
rsp (Stack Pointer):
  - MUST always point to top of stack
  - CPU uses it for push/pop/call/ret
  - Don't use for general math!

rip (Instruction Pointer):
  - Points to current instruction
  - CPU manages this automatically
  - Can't directly use it for calculations

Flags Register (not a general-purpose register):
  - Stores comparison results (Z, N, C, V flags)
  - Used by conditional jumps
```

**Summary:**  
Most registers are general-purpose. Only rsp and rip are truly dedicated. Others have conventional uses but aren't restricted.

---

### **Register vs RAM vs Cache vs Stack**

```
┌────────────────┬──────────┬──────────┬───────────┬─────────┐
│   Storage      │ Location │  Speed   │   Size    │  Use    │
├────────────────┼──────────┼──────────┼───────────┼─────────┤
│ Registers      │ In CPU   │ 1 cycle  │ 128 bytes │ Working │
│                │          │ (~0.3ns) │ (16×64bit)│ values  │
├────────────────┼──────────┼──────────┼───────────┼─────────┤
│ L1 Cache       │ In CPU   │ 4 cycles │ 32-64 KB  │ Recent  │
│                │          │ (~1ns)   │           │ data    │
├────────────────┼──────────┼──────────┼───────────┼─────────┤
│ L2 Cache       │ In CPU   │ 12 cycles│ 256 KB    │ Cached  │
│                │          │ (~3ns)   │ -1 MB     │ data    │
├────────────────┼──────────┼──────────┼───────────┼─────────┤
│ L3 Cache       │ In CPU   │ 40 cycles│ 8-32 MB   │ Shared  │
│                │(shared)  │ (~10ns)  │           │ cache   │
├────────────────┼──────────┼──────────┼───────────┼─────────┤
│ RAM (Stack/Heap│ Separate │100 cycles│8-64 GB    │ Main    │
│                │ chip     │ (~30ns)  │           │ memory  │
└────────────────┴──────────┴──────────┴───────────┴─────────┘

Registers: Fastest, smallest, in CPU, working values
Cache: Fast, medium, in CPU, copies of frequently-used RAM
Stack: Slow, large, in RAM, function calls and local variables
Heap: Slow, large, in RAM, dynamically allocated memory
```

---

### **What is Cache?**

**Cache:** Small, fast memory inside the CPU that stores copies of frequently-accessed data from RAM.

**Cache is NOT:**
- Not a section of stack (stack is in RAM)
- Not registers (cache is separate from registers)
- Not RAM (cache is on the CPU chip)

**What cache IS:**
- A "photocopy machine" for RAM
- Copies of data that you've recently accessed
- Organized in levels (L1, L2, L3)
- Managed automatically by hardware (you don't control it directly)

---

### **How Cache Works**

```
CPU wants data at address 0x1000:

Step 1: Check L1 cache
  ├─ Found? → Return immediately (1ns)
  └─ Not found? → Check L2 cache

Step 2: Check L2 cache
  ├─ Found? → Copy to L1, return (3ns)
  └─ Not found? → Check L3 cache

Step 3: Check L3 cache
  ├─ Found? → Copy to L2 and L1, return (10ns)
  └─ Not found? → Go to RAM (cache miss!)

Step 4: Access RAM (cache miss)
  └─ Load from RAM (30-100ns)
  └─ Copy to L3, L2, and L1
  └─ Return data

Next access to same address hits L1 → 30x faster!
```

**Cache Lines:**
```
When you access one byte, cache loads a whole "line" (typically 64 bytes)

Access byte at 0x1000:
Cache loads: 0x1000 - 0x103F (64 bytes)

Why? Spatial locality!
If you access 0x1000, you'll likely access 0x1001 next

Example: Looping through an array
int[] array = new int[16];  // 16 ints × 4 bytes = 64 bytes
for (int i = 0; i < 16; i++)
    sum += array[i];

First access: Cache miss (loads all 64 bytes)
Next 15 accesses: Cache hits! (all in same cache line)
```

---

### **What is a Cache Miss?**

**Cache Miss:** When the CPU looks for data in cache and doesn't find it, forcing a slow RAM access.

**Types of Cache Misses:**

**1. Compulsory Miss (Cold Miss):**
```
First access to data - it's never been in cache before

Example:
int x = array[0];  // First access → MISS (loads from RAM)
int y = array[0];  // Second access → HIT (now in cache)

Can't avoid these - data has to enter cache somehow
```

**2. Capacity Miss:**
```
Cache is full, old data was evicted, but you need it again

Example:
// L1 cache = 32 KB
int[] huge = new int[100000];  // 400 KB (way bigger than cache)

for (int i = 0; i < 100000; i++)
    huge[i] = i;  // First pass: fills cache, evicts old data

for (int i = 0; i < 100000; i++)
    sum += huge[i];  // Second pass: MISSES (data was evicted)

Solution: Process data in chunks that fit in cache
```

**3. Conflict Miss:**
```
Two addresses map to same cache location, keep evicting each other

Example (simplified):
Assume array[0] and array[1024] map to same cache slot

int x = array[0];     // MISS (load from RAM)
int y = array[1024];  // MISS (evicts array[0])
int z = array[0];     // MISS (evicts array[1024])

"Ping-pong" effect - repeatedly evicting each other
```

**Performance Impact:**
```
Cache Hit:   1-10ns
Cache Miss:  30-100ns

One cache miss = 10-100× slower!

Game with 90% cache hit rate:
Average access time = 0.9×1ns + 0.1×100ns = 10.9ns

Game with 99% cache hit rate:
Average access time = 0.99×1ns + 0.01×100ns = 2ns

5× speed difference just from better cache usage!
```

---

### **🎮 Game Dev Connection:**

```csharp
// BAD - Cache-hostile (random access)
class Enemy {
    Vector3 position;
    int health;
    // ... lots of other data (100+ bytes)
}

foreach (Enemy e in enemies)  // Jump around memory
    e.position += velocity * dt;

// Each enemy in different RAM location
// Cache loads whole enemy (100 bytes), only uses position (12 bytes)
// Wastes 88 bytes per cache line!

// GOOD - Cache-friendly (sequential access)
struct EnemyData {
    Vector3[] positions;   // All positions together
    int[] healths;         // All healths together
}

for (int i = 0; i < count; i++)
    positions[i] += velocity * dt;

// Positions packed together sequentially
// Cache loads 64 bytes = ~5 positions at once
// 5× fewer cache misses!
```

---

## 📚 The Control Unit and Instruction Set

### **The Control Unit**

**What it is:**  
The "brain" of the CPU that orchestrates execution of instructions.

**What it does:**
1. Fetches instructions from memory
2. Decodes instructions (figures out what to do)
3. Sends control signals to other CPU components
4. Manages timing (when things happen)

**Think of it as a conductor of an orchestra:**  
Doesn't play instruments, but tells everyone else what to do and when.

---

### **Instruction Set Architecture (ISA)**

The "language" the CPU speaks. Common ISAs:
- **x86-64:** Intel/AMD desktop CPUs (what you're probably using)
- **ARM:** Mobile phones, Apple M-series chips, consoles
- **RISC-V:** Open-source, education

---

### **Instruction Format**

Every instruction is binary, divided into fields:

```
Example: ADD R1, R2, R3  (R1 = R2 + R3)

32-bit instruction:
┌────────┬────────┬────────┬────────┬──────────────┐
│ Opcode │  R1    │  R2    │  R3    │   Unused     │
│ 6 bits │ 5 bits │ 5 bits │ 5 bits │   11 bits    │
└────────┴────────┴────────┴────────┴──────────────┘
   What     Dest     Src1     Src2
   to do    ination  Source   Source

Binary: 000000 00001 00010 00011 00000000000
```

---

### **Common Assembly Instructions (Detailed)**

| Instruction | Full Name | What It Does | Example | Explanation |
|-------------|-----------|--------------|---------|-------------|
| **Data Movement** |
| `mov` | Move | Copy data between registers/memory | `mov rax, 42` | rax = 42 |
| `lea` | Load Effective Address | Calculate memory address | `lea rax, [rbx+8]` | rax = address of (rbx + 8), not the value |
| `push` | Push | Save register to stack | `push rax` | Put rax on top of stack, decrement rsp |
| `pop` | Pop | Restore register from stack | `pop rax` | Get top of stack into rax, increment rsp |
| **Arithmetic** |
| `add` | Add | Add two values | `add rax, rbx` | rax = rax + rbx |
| `sub` | Subtract | Subtract values | `sub rax, rbx` | rax = rax - rbx |
| `imul` | Integer Multiply (signed) | Multiply signed integers | `imul rax, rbx` | rax = rax × rbx (signed) |
| `idiv` | Integer Divide (signed) | Divide signed integers | `idiv rbx` | rax = rax ÷ rbx (quotient), rdx = remainder |
| `inc` | Increment | Add 1 | `inc rax` | rax = rax + 1 |
| `dec` | Decrement | Subtract 1 | `dec rax` | rax = rax - 1 |
| `neg` | Negate | Two's complement | `neg rax` | rax = -rax |
| **Bitwise** |
| `and` | Bitwise AND | AND bits together | `and rax, rbx` | rax = rax & rbx |
| `or` | Bitwise OR | OR bits together | `or rax, rbx` | rax = rax \| rbx |
| `xor` | Bitwise XOR | XOR bits together | `xor rax, rax` | rax = 0 (common idiom: register XOR itself = 0) |
| `not` | Bitwise NOT | Flip all bits | `not rax` | rax = ~rax |
| `shl` | Shift Left | Shift bits left (×2) | `shl rax, 2` | rax = rax << 2 (multiply by 4) |
| `shr` | Shift Right | Shift bits right (÷2) | `shr rax, 2` | rax = rax >> 2 (divide by 4) |
| **Comparison & Branching** |
| `cmp` | Compare | Subtract and set flags | `cmp rax, rbx` | Compare rax to rbx, set Z/N/C/V flags |
| `test` | Test | AND and set flags | `test rax, rax` | Check if rax is zero (common idiom) |
| `jmp` | Jump | Unconditional jump | `jmp LABEL` | Go to LABEL (change instruction pointer) |
| `je` | Jump if Equal | Jump if Zero flag set | `je LABEL` | Jump if last comparison was equal |
| `jne` | Jump if Not Equal | Jump if Zero flag clear | `jne LABEL` | Jump if last comparison was not equal |
| `jg` | Jump if Greater | Jump if greater (signed) | `jg LABEL` | Jump if first operand > second (signed) |
| `jl` | Jump if Less | Jump if less (signed) | `jl LABEL` | Jump if first operand < second (signed) |
| `ja` | Jump if Above | Jump if greater (unsigned) | `ja LABEL` | Jump if first operand > second (unsigned) |
| `jb` | Jump if Below | Jump if less (unsigned) | `jb LABEL` | Jump if first operand < second (unsigned) |
| `call` | Call Function | Save return address and jump | `call FUNC` | Push next instruction address, jump to FUNC |
| `ret` | Return | Return from function | `ret` | Pop return address, jump there |
| **Special** |
| `nop` | No Operation | Do nothing for 1 cycle | `nop` | Wastes one cycle (for alignment/timing) |

---

### **Control Flow Instructions Explained**

#### **BEQ - Branch if Equal**

**BEQ:** Branch if Equal (common in MIPS/ARM assembly)

**What it does:**  
Jumps to a label if the previous comparison found two values equal.

```assembly
Example:
    cmp rax, rbx      ; Compare rax and rbx
    beq EQUAL_LABEL   ; If rax == rbx, jump to EQUAL_LABEL
    ; otherwise, continue here
    ...

EQUAL_LABEL:
    ; code executes here if rax == rbx
```

**x86-64 equivalent:** `je` (Jump if Equal)

---

#### **JMP - Jump**

**JMP:** Jump (unconditional)

**What it does:**  
Changes the instruction pointer to a different address. Always jumps, no conditions.

```assembly
Example:
    mov rax, 10
    jmp SKIP_THIS     ; Always jump
    mov rax, 20       ; This line NEVER executes
SKIP_THIS:
    ; Continue here (rax is still 10)
```

**Use case:** Infinite loops, skipping code sections

```assembly
GAME_LOOP:
    call ProcessInput
    call UpdateGame
    call RenderFrame
    jmp GAME_LOOP     ; Loop forever
```

---

### **Conditional Branch Instructions**

| Instruction | Stands For | Condition | Flags Checked |
|-------------|------------|-----------|---------------|
| **je** | Jump if Equal | `a == b` | Z = 1 |
| **jne** | Jump if Not Equal | `a != b` | Z = 0 |
| **jg** | Jump if Greater | `a > b` (signed) | Z=0 AND N=V |
| **jge** | Jump if Greater or Equal | `a >= b` (signed) | N = V |
| **jl** | Jump if Less | `a < b` (signed) | N ≠ V |
| **jle** | Jump if Less or Equal | `a <= b` (signed) | Z=1 OR N≠V |
| **ja** | Jump if Above | `a > b` (unsigned) | C=0 AND Z=0 |
| **jb** | Jump if Below | `a < b` (unsigned) | C = 1 |
| **jz** | Jump if Zero | `result == 0` | Z = 1 |
| **jnz** | Jump if Not Zero | `result != 0` | Z = 0 |
| **jo** | Jump if Overflow | Overflow occurred | V = 1 |

---

### **Example: If-Else in Assembly**

```csharp
// C# code
if (health > 50)
{
    speed = 10;
}
else
{
    speed = 5;
}
```

```assembly
; Assembly (x86-64)
    mov eax, [health]   ; Load health
    cmp eax, 50         ; Compare to 50
    jg GREATER          ; If health > 50, jump
    
    ; Else block
    mov [speed], 5      ; speed = 5
    jmp END             ; Skip the if block
    
GREATER:
    ; If block
    mov [speed], 10     ; speed = 10
    
END:
    ; Continue...
```

---

## 📚 Fetch-Decode-Execute Cycle

### **The CPU's Heartbeat**

Every CPU instruction goes through these steps:

```
┌──────────┐
│  FETCH   │ ← Get next instruction from memory
└─────┬────┘
      ↓
┌──────────┐
│  DECODE  │ ← Figure out what instruction means
└─────┬────┘
      ↓
┌──────────┐
│ EXECUTE  │ ← Perform the operation
└─────┬────┘
      ↓
┌──────────┐
│WRITE BACK│ ← Store result
└─────┬────┘
      ↓
   (Repeat)
```

---

### **Step-by-Step Example:**

Let's trace: `ADD R1, R2, R3` (R1 = R2 + R3)

**Initial State:**
- R2 = 5
- R3 = 3
- Program Counter (PC) = 0x1000
- Instruction stored at address 0x1000

---

#### **Step 1: FETCH**

```
1. Read address from PC register (0x1000)
2. Load instruction from memory at that address
3. Store instruction in Instruction Register (IR)
4. Increment PC to point to next instruction (0x1004)

Before:
PC = 0x1000
IR = (empty)

After:
PC = 0x1004  (ready for next instruction)
IR = 00000010001000110000000000000000  (the ADD instruction)
```

---

#### **Step 2: DECODE**

```
Control Unit examines the bits in IR:

IR = 000000 00001 00010 00011 00000000000
     ↑      ↑     ↑     ↑
     │      │     │     └─ R3 = register 3
     │      │     └─────── R2 = register 2
     │      └───────────── R1 = register 1 (destination)
     └────────────────────  Opcode 000000 = ADD

Decoded information:
- Operation: ADD
- Source 1: R2
- Source 2: R3
- Destination: R1

Control Unit activates:
✓ Read R2 signal
✓ Read R3 signal
✓ ALU operation = ADD
✓ Write R1 signal
```

---

#### **Step 3: EXECUTE**

```
1. Read R2 (value = 5) → Send to ALU input A
2. Read R3 (value = 3) → Send to ALU input B
3. ALU performs operation:
   - Operation selector set to ADD
   - A (5) + B (3) = 8
4. ALU outputs result (8)

ALU internal:
[Adder circuit activates]
5 + 3 = 8
Flags: Z=0 (not zero), N=0 (positive), C=0 (no carry), V=0 (no overflow)
```

---

#### **Step 4: WRITE BACK**

```
1. Take ALU result (8)
2. Write to destination register R1
3. Update flags register

Before:
R1 = (unknown)

After:
R1 = 8  ✓

Instruction complete!
PC now points to next instruction (0x1004)
```

---

### **Timing and Clock Cycles**

**Simple CPU (no pipelining):**
```
Each step takes 1 clock cycle

Clock:  |‾|_|‾|_|‾|_|‾|_|
Step:     F   D   E   W

One instruction = 4 cycles

At 1 GHz clock (1 billion cycles/second):
4 cycles = 4 nanoseconds per instruction
```

**IMPORTANT CLARIFICATION:**

The "4ns per instruction" is theoretical and assumes:
1. ✅ CPU is dedicated to one program
2. ✅ No cache misses
3. ✅ No interrupts
4. ✅ No context switches

**In reality (when running Windows + your game):**

```
Your game instruction might take much longer because:

1. Context Switching:
   - Windows runs hundreds of processes
   - OS switches between them (thousands of times/second)
   - Each switch: Save all registers, load new ones (~1000 cycles)

2. Cache Misses:
   - Your data evicted when other programs run
   - Reload from RAM = 100+ cycles

3. Operating System Interrupts:
   - Keyboard input, network packets, timers
   - Each interrupt: Stop, handle, resume (~10,000 cycles)

4. Shared Resources:
   - Other programs using CPU cores
   - Your program might wait for a free core

Real-world game instruction:
Best case: 4 ns (if everything perfect)
Typical: 10-100 ns (context switches, cache misses)
Worst case: 1,000+ ns (OS interrupt, page fault)

This is why:
- Games don't run at theoretical max speed
- Performance varies even on same hardware
- Background programs affect game FPS
```

**Takeaway:** "4ns per instruction" is ideal; real performance depends on OS, other programs, and memory access patterns.

---

## 📚 Assembly Language Deep Dive

### **What Is Assembly?**

**Assembly:** Human-readable representation of machine code. Each assembly instruction = 1 CPU instruction.

```
C# code:      int sum = a + b;
                ↓ (compiler translates)
Assembly:     addl %eax, %ebx
                ↓ (assembler converts)
Machine code: 10000001 11000011
                ↓ (CPU executes)
Binary operation on registers
```

---

### **Example: Sum of Array**

```assembly
; Sum first 5 elements of array
; Input: R0 = array base address
; Output: R2 = sum

        MOV R0, #0x1000    ; R0 = array address
        MOV R1, #5         ; R1 = counter (5 elements)
        MOV R2, #0         ; R2 = sum (accumulator)

LOOP:
        LDR R3, [R0]       ; R3 = array[i] (load from memory)
        ADD R2, R2, R3     ; sum += array[i]
        ADD R0, R0, #4     ; Move to next element (4 bytes per int)
        SUB R1, R1, #1     ; counter--
        CMP R1, #0         ; Compare counter with 0
        BNE LOOP           ; If counter != 0, jump to LOOP

        ; R2 now contains sum
        HALT
```

**Equivalent C# code:**
```csharp
int[] array = new int[5];
int sum = 0;
for (int i = 0; i < 5; i++)
{
    sum += array[i];
}
```

---

## 📚 From C# to Machine Code

### **The Complete Compilation Pipeline**

```
┌──────────────────────────────────────────────────┐
│  C# Source Code (.cs files)                      │
└───────────────────┬──────────────────────────────┘
                    ↓
          ┌─────────────────────┐
          │  Roslyn Compiler    │
          └─────────┬───────────┘
                    ↓
┌──────────────────────────────────────────────────┐
│  IL (Intermediate Language) / Bytecode           │
│  Platform-independent, stored in .dll/.exe       │
└───────────────────┬──────────────────────────────┘
                    ↓
          ┌─────────────────────┐
          │  JIT Compiler  OR   │
          │  IL2CPP / AOT       │
          └─────────┬───────────┘
                    ↓
┌──────────────────────────────────────────────────┐
│  Native Machine Code                             │
│  x86-64, ARM, etc. - CPU can execute directly    │
└───────────────────┬──────────────────────────────┘
                    ↓
          ┌─────────────────────┐
          │   CPU Executes      │
          └─────────────────────┘
```

---

### **What is Roslyn?**

**Roslyn:** The official C# compiler from Microsoft.

**Full Name:** .NET Compiler Platform ("Roslyn" is the codename, now official name)

**What it does:**
```
Input:  C# source code (.cs files)
Output: IL (Intermediate Language) bytecode (.dll or .exe files)

Example:
MyGame.cs → [Roslyn] → MyGame.dll (contains IL, not machine code!)
```

**Why "Roslyn"?**  
Named after Roslyn, Washington (small town near Microsoft headquarters).

**Key features:**
- Open source (you can see the compiler's source code!)
- Provides APIs for code analysis
- Used by Visual Studio for IntelliSense, refactoring, etc.

---

### **What is IL (Intermediate Language)?**

**IL:** Intermediate Language (also called **MSIL** - Microsoft Intermediate Language, or **CIL** - Common Intermediate Language)

**What it is:**
- A low-level, platform-independent bytecode
- Higher level than assembly, lower level than C#
- Stored in .dll and .exe files
- Not directly executable by CPU (needs JIT or AOT compilation)

**Why use IL instead of going straight to machine code?**

**1. Platform Independence:**
```
C# code → IL → Can run on:
  - Windows (x86-64)
  - Linux (x86-64)
  - Mac (ARM)
  - Android (ARM)
  - iOS (ARM)

Same IL works on all platforms!
```

**2. JIT Optimization:**
```
IL allows runtime optimizations:
- Compile for actual CPU (not guessed CPU)
- Optimize for current memory layout
- Inline functions based on real usage
```

**3. Verification and Security:**
```
IL can be verified for safety before execution:
- Type safety checks
- Memory access validation
- Security policy enforcement
```

**Example IL code:**
```csharp
// C# code
int sum = a + b;

// IL (Intermediate Language)
ldloc.0      // Load local variable 0 (a) onto stack
ldloc.1      // Load local variable 1 (b) onto stack
add          // Add top two stack values
stloc.2      // Store result in local variable 2 (sum)
```

**IL is stack-based:**
```
Unlike assembly (register-based), IL uses a virtual stack:

ldloc.0  →  Stack: [a]
ldloc.1  →  Stack: [a, b]
add      →  Stack: [a+b]
stloc.2  →  Stack: []  (result stored in variable)
```

---

### **What is JIT?**

**JIT:** Just-In-Time Compiler

**What it does:**
- Compiles IL to native machine code **at runtime** (when program runs)
- Compiles functions **the first time they're called**
- Cached for future calls (only compile once per function)

**How JIT works:**
```
1. Program starts
2. Function called for first time
   └─ JIT compiles IL → machine code (takes a few milliseconds)
   └─ Stores compiled code in memory
3. Function called again
   └─ Use cached machine code (instant!)

First call: Slow (compilation overhead)
Subsequent calls: Fast (cached)
```

**Advantages:**
```
✓ Optimize for actual CPU (not guessed)
✓ Only compile code that actually runs
✓ Can use runtime profiling data
```

**Disadvantages:**
```
✗ Slower first call (compilation delay)
✗ Can't optimize as aggressively (time limit)
✗ Larger memory usage (IL + compiled code)
```

**Unity example:**
```
Playing in Unity Editor:
- Uses JIT compilation
- Quick iteration (no build step)
- Slower performance (less optimization)
```

---

### **What is AOT?**

**AOT:** Ahead-Of-Time Compiler

**What it does:**
- Compiles IL to native machine code **before running** (during build)
- All code compiled upfront, no runtime compilation
- Used in IL2CPP (Unity), Xamarin, Native AOT (.NET 7+)

**How AOT works:**
```
1. Build time:
   └─ Compile ALL IL → machine code
   └─ Generates native executable
2. Runtime:
   └─ Execute machine code directly (no compilation!)
```

**Advantages:**
```
✓ Faster startup (no compilation at runtime)
✓ Better performance (more optimization time)
✓ Smaller runtime (no JIT compiler needed)
✓ Required for iOS (no JIT allowed)
```

**Disadvantages:**
```
✗ Slower builds (compile everything upfront)
✗ Larger executable (all code compiled, even unused)
✗ Less runtime optimization (CPU unknown at build time)
```

**Unity IL2CPP example:**
```
Building for iOS/Consoles:
- IL → C++ code → Native machine code
- No JIT allowed on these platforms
- Faster runtime, slower builds
```

---

### **What is IL2CPP?**

**IL2CPP:** Intermediate Language to C++

**What it does:**
```
C# → IL → C++ → Native Machine Code

Instead of:
C# → IL → JIT → Machine Code

Why the extra step?
- Platforms that don't allow JIT (iOS, consoles)
- Better performance (C++ compiler very mature)
- Smaller runtime (no JIT needed)
```

---

### **Why So Many Steps? Why Not C# → Machine Code Directly?**

**Short answer:** Flexibility, portability, and optimization.

**Long answer:**

**1. Platform Independence:**
```
Direct compilation:
C# → Windows x86-64 machine code
Can't run on Mac ARM!

With IL:
C# → IL → [JIT/AOT] → Windows x86-64 machine code
C# → IL → [JIT/AOT] → Mac ARM machine code
C# → IL → [JIT/AOT] → Linux x86-64 machine code

Same C# source, runs everywhere!
```

**2. Code Sharing:**
```
With IL:
Unity game, .NET web app, Xamarin mobile app all use same libraries
System.dll contains IL that works everywhere

Without IL:
Would need separate compiled libraries for each platform
Massive duplication!
```

**3. Optimization Opportunities:**
```
C# → IL:
  Roslyn optimizes high-level code
  (inline small functions, constant folding, dead code removal)

IL → Machine Code:
  JIT/AOT optimizes low-level code
  (register allocation, instruction selection, CPU-specific SIMD)

Two-phase optimization = better results!
```

**4. Runtime Flexibility:**
```
JIT can:
- Inline functions based on real usage patterns
- Optimize for actual CPU features (AVX2, AVX-512)
- Adapt to memory layout

Direct compilation can't do this (CPU unknown at compile time)
```

**5. Security and Verification:**
```
IL can be verified for safety:
- Type safety
- Memory access bounds
- Security permissions

Harder to verify machine code directly
```

**6. Reflection and Dynamic Features:**
```
C# supports:
- Reflection (inspect types at runtime)
- Dynamic code generation
- Generics with runtime instantiation

IL makes these possible; direct machine code makes them very hard
```

**Historical reason:**  
Java pioneered this approach in 1995, proved it works well. .NET adopted it in 2002.

---

### **Unity Compilation Modes Compared:**

```
┌──────────────┬─────────────┬─────────────┬──────────────┐
│     Mode     │ Compilation │ Performance │  Build Time  │
├──────────────┼─────────────┼─────────────┼──────────────┤
│ Editor       │ JIT         │ Slow        │ Instant      │
│ (Play mode)  │             │ (debugging) │ (no build)   │
├──────────────┼─────────────┼─────────────┼──────────────┤
│ Mono Build   │ JIT         │ Medium      │ Fast         │
│              │             │             │              │
├──────────────┼─────────────┼─────────────┼──────────────┤
│ IL2CPP       │ AOT         │ Fast        │ Slow         │
│              │             │             │              │
├──────────────┼─────────────┼─────────────┼──────────────┤
│ Burst        │ AOT + SIMD  │ Very Fast   │ Medium       │
│ (with IL2CPP)│             │ (vectorized)│              │
└──────────────┴─────────────┴─────────────┴──────────────┘

Recommendation:
Development: Editor (JIT) - fast iteration
Shipping: IL2CPP + Burst - best performance
```

---

## 📚 CPU Performance Features

### **How Do More Cores Increase CPU Functionality?**

**Each core is a complete CPU** that can run its own instruction stream independently.

```
Single Core:
┌─────────────┐
│   Core 0    │
│  [ALU] [REG]│  Can run ONE instruction stream
└─────────────┘

Four Cores:
┌─────────────┬─────────────┬─────────────┬─────────────┐
│   Core 0    │   Core 1    │   Core 2    │   Core 3    │
│  [ALU] [REG]│  [ALU] [REG]│  [ALU] [REG]│  [ALU] [REG]│
└─────────────┴─────────────┴─────────────┴─────────────┘
Can run FOUR instruction streams simultaneously!
```

**What more cores enable:**

**1. True Parallelism:**
```
Without multiple cores:
Time:  |─────Game─────|─────Browser─────|─────OS─────|
       └── Time-slice (appears simultaneous, but sequential)

With multiple cores:
Core 0: |───────────Game───────────────────────|
Core 1: |───────────Browser────────────────────|
Core 2: |───────────OS──────────────────────────|
Core 3: |───────────Idle─────────────────────────|
       └── Actually simultaneous!
```

**2. Parallel Processing in Your Game:**
```csharp
// Single core: 16ms
for (int i = 0; i < 10000; i++)
    enemies[i].Update();

// Four cores: 4ms (4× faster!)
Parallel.For(0, 10000, i =>
    enemies[i].Update());

Core 0: enemies[0-2499]
Core 1: enemies[2500-4999]
Core 2: enemies[5000-7499]
Core 3: enemies[7500-9999]
All run simultaneously!
```

**3. Specialized Core Usage:**
```
Modern games often use:
Core 0: Main game logic
Core 1: Physics simulation
Core 2: AI pathfinding
Core 3: Audio processing
Core 4-7: Rendering tasks

Each core does different work simultaneously
```

**Important:** More cores only help if work can be parallelized!
```
Sequential work (can't parallelize):
LoadLevel() → ParseData() → InitializeScene()
  │            │              │
  Must wait    Must wait      Done

10 cores won't help - each step depends on previous!

Parallel work (can parallelize):
UpdateEnemy[0]
UpdateEnemy[1]  ← All independent
UpdateEnemy[2]
UpdateEnemy[3]

4 cores = 4× faster!
```

---

### **Instruction Pipelining**

**The Problem (Without Pipelining):**
```
Each instruction takes 4 cycles:

Instruction 1:  |F|D|E|W|
Instruction 2:            |F|D|E|W|
Instruction 3:                      |F|D|E|W|

Total: 12 cycles for 3 instructions
Throughput: 1 instruction per 4 cycles = 0.25 inst/cycle
```

**The Solution (With Pipelining):**
```
Overlap instruction stages:

Time:         | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
Instruction 1:| F | D | E | W |
Instruction 2:    | F | D | E | W |
Instruction 3:        | F | D | E | W |
Instruction 4:            | F | D | E | W |

Total: 7 cycles for 4 instructions
Throughput: ~1 instruction per cycle!

4× more throughput (after pipeline fills)
```

**How It Works:**

Think of a car assembly line:

```
Without Pipeline (One Car at a Time):
Station 1: [Build frame - 1 hour]
Station 2: [Install engine - 1 hour]
Station 3: [Paint - 1 hour]
Station 4: [Install interior - 1 hour]

One car: 4 hours
Three cars: 12 hours (4 hours each)

With Pipeline:
Hour 1: Car A at Station 1
Hour 2: Car A at Station 2, Car B at Station 1
Hour 3: Car A at Station 3, Car B at Station 2, Car C at Station 1
Hour 4: Car A done, Car B at Station 3, Car C at Station 2, Car D at Station 1

Three cars: 6 hours (not 12!)
Throughput: 1 car per hour (after initial 4-hour delay)
```

**Pipeline Stages in Detail:**

```
Stage 1 - Fetch (F):
  - Read instruction from memory
  - Decode which address to fetch next

Stage 2 - Decode (D):
  - Determine what instruction does
  - Identify which registers needed
  - Generate control signals

Stage 3 - Execute (E):
  - Perform ALU operation
  - Calculate memory addresses
  - Do the actual work

Stage 4 - Write-Back (W):
  - Write result to register
  - Update flags
  - Commit changes
```

**Modern CPUs:**
```
Modern CPUs have 10-20 pipeline stages!

Intel Core i7 (Skylake): 14-19 stages
AMD Ryzen: 15-17 stages

More stages = higher clock speed
But: Harder to keep pipeline full
```

**Pipeline Hazards (Things That Slow It Down):**

**1. Data Hazard:**
```assembly
add rax, rbx   ; Stage 4 (writing to rax)
add rcx, rax   ; Stage 2 (trying to read rax)
               ; Must wait! rax not ready yet

Solution: Forwarding (pass result directly, don't wait for write-back)
```

**2. Control Hazard (Branch):**
```assembly
cmp rax, rbx
je LABEL       ; Don't know which instruction to fetch next!

Pipeline must stall or guess (branch prediction)
```

**3. Structural Hazard:**
```
Two instructions need same resource:
Instruction 1: Reading from memory
Instruction 2: Writing to memory (same memory unit!)

Solution: Separate instruction and data caches
```

---

### **Why Multiple Execution Units?**

**Superscalar Execution:** CPU has multiple ALUs, load/store units, etc., so it can execute multiple instructions per cycle.

```
Single Execution Unit:
┌────────────────┐
│  Fetch/Decode  │
└───────┬────────┘
        ↓
    ┌───────┐
    │  ALU  │  ← Bottleneck! Only 1 instruction/cycle
    └───────┘

Multiple Execution Units (Superscalar):
┌────────────────┐
│  Fetch/Decode  │
│  (can fetch 4  │
│   instructions)│
└───┬───┬───┬────┘
    │   │   │
   ┌┴┐ ┌┴┐ ┌┴┐
   │A│ │A│ │L│  ← 2 ALUs, 1 Load/Store
   │L│ │L│ │o│
   │U│ │U│ │a│
   │1│ │2│ │d│
   └─┘ └─┘ └─┘

Can execute UP TO 4 instructions per cycle!
```

**Why needed:**

**1. Keep All Parts Busy:**
```
Instruction stream:
add rax, rbx   ; Uses ALU
mov rcx, [rdx] ; Uses Load unit
mul rsi, rdi   ; Uses Multiply unit
and r8, r9     ; Uses ALU

Without multiple units: 4 cycles (one at a time)
With multiple units: 1 cycle (all simultaneously!)
```

**2. Different Instructions Need Different Resources:**
```
Resources in modern CPU:
- Integer ALU (add, sub, and, or) × 4
- Floating-Point Unit (float math) × 2
- Load/Store Unit (memory access) × 2
- Branch Unit (jumps) × 1
- Multiply/Divide Unit × 1

Can execute: 2 integer ops + 1 float op + 2 memory ops simultaneously!
```

**3. Instruction-Level Parallelism (ILP):**
```csharp
// These are independent:
int a = x + y;  // ALU 1
int b = z * w;  // Multiply Unit
int c = arr[i]; // Load Unit

All execute in same cycle!

// These are dependent:
int a = x + y;  // ALU 1
int b = a * 2;  // Must wait for 'a'

Can't parallelize - second depends on first
```

**Example: Intel Core i7 (Skylake)**
```
Can execute up to 4 "fused μops" per cycle:
- 4 integer ALUs
- 2 load/store units
- 3 vector (SIMD) units
- 1 branch unit

Theoretical: 4 instructions per cycle
Real-world: 1.5-2 instructions per cycle (due to dependencies)
```

---

### **Out-of-Order Execution**

**What it is:**  
CPU reorders instructions to keep execution units busy, even when there are data dependencies.

**Why reorder?**

**Problem: In-Order Execution Wastes Time**
```assembly
; Program order:
load r1, [mem1]  ; Takes 100 cycles (cache miss!)
add  r2, r1, r3  ; Waits 100 cycles for r1
mul  r4, r5, r6  ; Could run now, but waits for add!

In-order: Total 102 cycles
  load: 100 cycles
  add:  1 cycle (waited for load)
  mul:  1 cycle (waited for add)
```

**Solution: Out-of-Order Execution**
```assembly
; Program order:
load r1, [mem1]  ; Start loading (slow)
add  r2, r1, r3  ; Depends on load (can't start yet)
mul  r4, r5, r6  ; Independent! Can run while loading

CPU reorders to:
load r1, [mem1]  ; Cycle 0-100 (slow memory access)
mul  r4, r5, r6  ; Cycle 0 (runs while load happens!)
add  r2, r1, r3  ; Cycle 100 (runs when load finishes)

Out-of-order: Total 100 cycles (2 cycles saved!)
```

**How It Works:**

**1. Rename Registers (Remove False Dependencies):**
```assembly
; Appears to use same register:
add rax, rbx    ; rax = rax + rbx
mov rax, 10     ; Overwrites rax
mul rcx, rax    ; Uses new rax

False dependency: mul seems to depend on add
Actually: mul only depends on mov!

CPU renames internally:
add r_phys1, r_phys2  ; Physical register 1
mov r_phys3, 10       ; Physical register 3 (different!)
mul r_phys4, r_phys3  ; Depends on mov, not add

Now add and mov can run simultaneously!
```

**2. Reservation Stations (Queue Instructions):**
```
CPU maintains queue of instructions ready to execute:

Reservation Stations:
[add r2, r1, r3] ← Waiting for r1 (not ready)
[mul r4, r5, r6] ← r5 and r6 ready → EXECUTE NOW!
[sub r7, r8, r9] ← r8 and r9 ready → EXECUTE NOW!

Executes mul and sub before add (even though add came first)
```

**3. Reorder Buffer (Preserve Program Order for Results):**
```
Execution order: mul, sub, add
Commit order:    add, mul, sub (original program order)

Results stored in Reorder Buffer until all previous instructions complete
This ensures program appears to execute in order (even though it doesn't!)
```

**Why Reorder?**

**1. Hide Memory Latency:**
```
Memory access = 100 cycles
Do 100 other instructions while waiting!

Without reordering: CPU sits idle for 100 cycles
With reordering: CPU does useful work during wait
```

**2. Keep Execution Units Busy:**
```
Instruction stream:
load (uses Load unit, 100 cycles)
add (uses ALU, needs load result)
mul (uses Multiply unit, independent)

Without reordering:
  Load: 100 cycles, ALU idle, Multiply idle
  Add: 1 cycle, Multiply idle
  Mul: 1 cycle

With reordering:
  Load: 100 cycles
  Mul: 1 cycle (runs during load!)
  Add: 1 cycle
  
Multiply unit utilized instead of sitting idle!
```

**3. Maximize Instruction-Level Parallelism:**
```
In a 10-instruction window, might find:
- 3 independent integer ops
- 2 independent float ops
- 1 memory load

Execute all 6 simultaneously (if execution units available)
```

**Example:**
```assembly
; Original program order:
1. load r1, [addr1]    ; 100 cycles (cache miss)
2. add  r2, r1, 5      ; Depends on 1
3. mul  r3, r4, r5     ; Independent
4. sub  r6, r7, r8     ; Independent
5. load r9, [addr2]    ; Independent
6. and  r10, r2, r3    ; Depends on 2 and 3

CPU execution order:
1. load r1, [addr1]    ; Start (cycle 0)
3. mul  r3, r4, r5     ; Run while loading (cycle 0)
4. sub  r6, r7, r8     ; Run while loading (cycle 1)
5. load r9, [addr2]    ; Run while loading (cycle 2)
2. add  r2, r1, 5      ; Run when load done (cycle 100)
6. and  r10, r2, r3    ; Run when add done (cycle 101)

Original order: 104 cycles
Reordered:      101 cycles

3 cycles saved by keeping CPU busy during memory wait!
```

---

### **What is `__m128`?**

**`__m128`:** A 128-bit SIMD (Single Instruction, Multiple Data) register type

**Full explanation:**

**What it is:**
- A data type representing a 128-bit register
- Can hold 4 floats simultaneously (4 × 32-bit = 128 bits)
- Part of SSE (Streaming SIMD Extensions) instruction set
- Defined in `<xmmintrin.h>` header (Intel intrinsics)

**Why use it:**
```
Normal code (scalar):
float result = a + b;  // One addition

SIMD code:
__m128 a_vec = {a1, a2, a3, a4};
__m128 b_vec = {b1, b2, b3, b4};
__m128 result = _mm_add_ps(a_vec, b_vec);

// result = {a1+b1, a2+b2, a3+b3, a4+b4}
// Four additions in ONE instruction!
```

**Example:**
```cpp
#include <xmmintrin.h>  // SSE intrinsics

// Add 4 floats at once
__m128 a = _mm_set_ps(1.0f, 2.0f, 3.0f, 4.0f);
__m128 b = _mm_set_ps(5.0f, 6.0f, 7.0f, 8.0f);
__m128 result = _mm_add_ps(a, b);
// result = {6.0f, 8.0f, 10.0f, 12.0f}

// Extract individual results
float r0 = _mm_cvtss_f32(result);  // 6.0
```

**Common SIMD types:**
```
__m128:  128-bit (4 floats or 2 doubles)
__m256:  256-bit (8 floats or 4 doubles) [AVX]
__m512:  512-bit (16 floats or 8 doubles) [AVX-512]

Unity Burst uses these automatically for float4, float3, etc.!
```

**Unity equivalent:**
```csharp
// C# with Burst
[BurstCompile]
public void AddVectors(NativeArray<float4> a, NativeArray<float4> b, NativeArray<float4> result)
{
    for (int i = 0; i < a.Length; i++)
    {
        result[i] = a[i] + b[i];  // Burst compiles to __m128 instructions!
    }
}

// Becomes (in machine code):
// movaps xmm0, [a]
// addps xmm0, [b]
// movaps [result], xmm0

4× faster than scalar code!
```

---

# Practical Exercises

## Exercise 1: Binary Adder Simulation (Easy)

**Goal:** Implement the full adder circuits we learned about in software.

### **Background:**

Remember from the lecture:
- **Half Adder:** Adds 2 bits → produces Sum and Carry
- **Full Adder:** Adds 3 bits (A, B, Carry-In) → produces Sum and Carry-Out
- **4-Bit Adder:** Chains 4 full adders together

### **Task:**

Implement these adder circuits in C++:

```cpp
#include <iostream>
#include <bitset>

// Half Adder: Adds two bits
struct HalfAdderResult {
    bool sum;
    bool carry;
};

HalfAdderResult HalfAdder(bool a, bool b) {
    // TODO: Implement
    // Sum = A XOR B
    // Carry = A AND B
    return {false, false};
}

// Full Adder: Adds three bits
struct FullAdderResult {
    bool sum;
    bool carryOut;
};

FullAdderResult FullAdder(bool a, bool b, bool carryIn) {
    // TODO: Implement using two half adders
    // Hint: 
    // 1. First half adder: A + B
    // 2. Second half adder: result + CarryIn
    // 3. CarryOut = Carry1 OR Carry2
    return {false, false};
}

// 4-Bit Ripple-Carry Adder
uint8_t Add4Bit(uint8_t a, uint8_t b) {
    // TODO: Implement using 4 full adders
    // Process bits 0-3, carry ripples through
    // Return result (may overflow into bit 4)
    return 0;
}

// 8-Bit Adder
uint8_t Add8Bit(uint8_t a, uint8_t b, bool& overflow) {
    // TODO: Implement using 8 full adders
    // Set overflow flag if result > 255
    return 0;
}
```

**Requirements:**

1. **Implement all four functions** using only bitwise operations and the adder functions
2. **Test your implementations:**
   ```cpp
   int main() {
       // Test Half Adder
       auto ha = HalfAdder(1, 1);
       std::cout << "1 + 1 = Sum:" << ha.sum << " Carry:" << ha.carry << std::endl;
       // Expected: Sum:0 Carry:1 (binary 10 = 2)
       
       // Test Full Adder
       auto fa = FullAdder(1, 1, 1);
       std::cout << "1 + 1 + 1 = Sum:" << fa.sum << " Carry:" << fa.carryOut << std::endl;
       // Expected: Sum:1 Carry:1 (binary 11 = 3)
       
       // Test 4-Bit Adder
       uint8_t result4 = Add4Bit(7, 5);
       std::cout << "7 + 5 = " << (int)result4 << std::endl;
       // Expected: 12
       
       // Test 8-Bit Adder with overflow
       bool overflow;
       uint8_t result8 = Add8Bit(200, 100, overflow);
       std::cout << "200 + 100 = " << (int)result8 << " Overflow:" << overflow << std::endl;
       // Expected: 44 (300 % 256), Overflow:true
       
       return 0;
   }
   ```

3. **Answer these questions:**
    - How many gate operations does a 4-bit addition require?
    - Why is it called "ripple-carry"? What's the downside?
    - What happens when adding 15 + 1 in 4-bit? (overflow behavior)

**Bonus Challenge:**
- Implement a subtraction function using Two's Complement
- Add overflow detection for signed integers
- Visualize the carry propagation through each bit

---

## Exercise 2: Assembly Instruction Simulator (Easy-Medium)

**Goal:** Simulate basic CPU instructions to understand the fetch-decode-execute cycle.

### **Task:**

Create a simple CPU simulator that executes basic assembly instructions:

```cpp
#include <iostream>
#include <vector>
#include <string>

class SimpleCPU {
private:
    // Registers (8 general-purpose registers)
    int registers[8] = {0};
    
    // Program Counter
    int PC = 0;
    
    // Flags
    bool ZeroFlag = false;
    bool NegativeFlag = false;
    
    // Memory (simple array)
    int memory[256] = {0};
    
public:
    // Instruction format: opcode, operand1, operand2
    enum Opcode {
        MOV,    // MOV R1, 42    -> R1 = 42
        ADD,    // ADD R1, R2    -> R1 = R1 + R2
        SUB,    // SUB R1, R2    -> R1 = R1 - R2
        MUL,    // MUL R1, R2    -> R1 = R1 * R2
        CMP,    // CMP R1, R2    -> Compare R1 and R2, set flags
        JMP,    // JMP 10        -> PC = 10
        JE,     // JE 10         -> If ZeroFlag, PC = 10
        LOAD,   // LOAD R1, [50] -> R1 = memory[50]
        STORE,  // STORE R1, [50]-> memory[50] = R1
        HALT    // HALT          -> Stop execution
    };
    
    struct Instruction {
        Opcode opcode;
        int operand1;
        int operand2;
    };
    
    void Execute(const std::vector<Instruction>& program) {
        PC = 0;
        
        while (PC < program.size()) {
            // FETCH
            Instruction instr = program[PC];
            
            // DECODE & EXECUTE
            switch (instr.opcode) {
                case MOV:
                    // TODO: Implement MOV
                    break;
                case ADD:
                    // TODO: Implement ADD
                    break;
                case SUB:
                    // TODO: Implement SUB
                    break;
                case MUL:
                    // TODO: Implement MUL
                    break;
                case CMP:
                    // TODO: Compare operand1 and operand2, set flags
                    // ZeroFlag = (R[op1] == R[op2])
                    // NegativeFlag = (R[op1] < R[op2])
                    break;
                case JMP:
                    // TODO: Unconditional jump
                    break;
                case JE:
                    // TODO: Jump if ZeroFlag set
                    break;
                case LOAD:
                    // TODO: Load from memory
                    break;
                case STORE:
                    // TODO: Store to memory
                    break;
                case HALT:
                    return;
            }
            
            PC++;  // Move to next instruction
        }
    }
    
    void PrintRegisters() {
        std::cout << "Registers:" << std::endl;
        for (int i = 0; i < 8; i++) {
            std::cout << "  R" << i << " = " << registers[i] << std::endl;
        }
        std::cout << "Flags: Zero=" << ZeroFlag << " Negative=" << NegativeFlag << std::endl;
    }
};
```

**Requirements:**

1. **Implement all instruction types**

2. **Test with this program** (sum of 1 to 10):
   ```cpp
   int main() {
       SimpleCPU cpu;
       
       std::vector<SimpleCPU::Instruction> program = {
           {SimpleCPU::MOV, 0, 0},      // R0 = 0 (sum)
           {SimpleCPU::MOV, 1, 1},      // R1 = 1 (counter)
           {SimpleCPU::MOV, 2, 10},     // R2 = 10 (limit)
           // LOOP:
           {SimpleCPU::ADD, 0, 1},      // R0 = R0 + R1 (sum += counter)
           {SimpleCPU::ADD, 1, 3},      // R1 = R1 + 1 (counter++, assuming R3=1)
           {SimpleCPU::CMP, 1, 2},      // Compare counter to limit
           {SimpleCPU::JE, 8, 0},       // If equal, jump to HALT
           {SimpleCPU::JMP, 3, 0},      // Otherwise, jump to LOOP
           {SimpleCPU::HALT, 0, 0}
       };
       
       // Set R3 = 1 for increment
       // You'll need to add a way to initialize this
       
       cpu.Execute(program);
       cpu.PrintRegisters();
       
       // Expected: R0 = 55 (sum of 1 to 10)
       
       return 0;
   }
   ```

3. **Add cycle counting:**
    - Track how many cycles each instruction takes
    - MOV, ADD, SUB: 1 cycle
    - MUL: 3 cycles
    - LOAD/STORE: 4 cycles (memory access!)
    - JMP/JE: 2 cycles
    - Print total cycles at end

4. **Answer these questions:**
    - How many instructions executed for sum of 1-10?
    - How many total cycles?
    - What percentage of time is spent on memory access?
    - How would pipelining improve this?

**Bonus Challenge:**
- Add more instructions (DIV, AND, OR, XOR)
- Implement a stack with PUSH/POP
- Add function CALL/RET
- Visualize the fetch-decode-execute cycle

---

## Exercise 3: Compilation Pipeline Explorer (Medium)

**Goal:** Understand the C++ → Assembly → Machine Code pipeline.

### **Task:**

Explore what the compiler does to your C++ code at each stage.

**Step 1: Write a simple C++ function**

```cpp
// sum.cpp
int Sum(int a, int b) {
    return a + b;
}

int SumArray(int* array, int count) {
    int sum = 0;
    for (int i = 0; i < count; i++) {
        sum += array[i];
    }
    return sum;
}

int main() {
    int x = Sum(5, 3);
    
    int numbers[] = {1, 2, 3, 4, 5};
    int total = SumArray(numbers, 5);
    
    return 0;
}
```

**Step 2: Generate assembly**

```bash
# Compile to assembly (don't assemble)
g++ -S -O0 sum.cpp -o sum_O0.asm    # No optimization
g++ -S -O2 sum.cpp -o sum_O2.asm    # With optimization
g++ -S -O3 sum.cpp -o sum_O3.asm    # Aggressive optimization

# View the assembly
cat sum_O0.asm
```

**Step 3: Generate machine code (view in hex)**

```bash
# Compile to object file
g++ -c sum.cpp -o sum.o

# View machine code in hex
objdump -d sum.o > sum_disassembly.txt
cat sum_disassembly.txt
```

**Requirements:**

1. **Compare the three optimization levels:**
    - Count the number of instructions in `Sum()` at each level
    - Identify what optimizations were applied
    - Note differences in register usage

2. **Analyze `SumArray()` function:**
    - Find the loop in assembly (look for labels and jumps)
    - Identify the array access pattern
    - Count instructions per loop iteration
    - Did the compiler unroll the loop at -O3?

3. **Examine machine code:**
    - Find the `Sum()` function in the disassembly
    - Note the instruction encodings (hex bytes)
    - Compare instruction sizes (some are 2 bytes, some are 5+)

4. **Answer these questions:**
    - How many assembly instructions does `a + b` become?
    - What registers are used for function parameters?
    - How is the return value handled?
    - What is the size (in bytes) of the `Sum()` function in machine code?
    - At -O3, was `Sum()` inlined into `main()`?

**Bonus Challenges:**
- Use `-fverbose-asm` to get annotated assembly
- Compare with `-march=native` (uses CPU-specific instructions)
- Try compiling with `-masm=intel` for Intel syntax instead of AT&T
- Write a function and predict what assembly it will generate, then verify

**Expected Observations:**
- `-O0`: Many instructions, uses stack heavily
- `-O2`: Fewer instructions, uses registers more
- `-O3`: Might inline `Sum()`, might unroll `SumArray()` loop
- Machine code is hard to read but directly executable by CPU

---

## Exercise 4: Branch Prediction Impact (Medium-Hard)

**Goal:** Measure the performance impact of branch misprediction.

### **Background:**

Modern CPUs predict which way branches (if statements, loops) will go. When wrong, the pipeline must flush (10-20 cycle penalty!).

### **Task:**

Write code to demonstrate branch prediction impact:

```cpp
#include <iostream>
#include <chrono>
#include <algorithm>
#include <random>

const int SIZE = 1000000;

// Version 1: Predictable branches
long long SumIfPositive_Sorted(int* data, int size) {
    long long sum = 0;
    
    for (int i = 0; i < size; i++) {
        if (data[i] >= 0) {  // Predictable after sorting
            sum += data[i];
        }
    }
    
    return sum;
}

// Version 2: Unpredictable branches
long long SumIfPositive_Random(int* data, int size) {
    long long sum = 0;
    
    for (int i = 0; i < size; i++) {
        if (data[i] >= 0) {  // Unpredictable!
            sum += data[i];
        }
    }
    
    return sum;
}

// Version 3: Branchless (no if statement)
long long SumIfPositive_Branchless(int* data, int size) {
    long long sum = 0;
    
    for (int i = 0; i < size; i++) {
        // TODO: Implement without 'if'
        // Hint: Use bitwise operations
        // If data[i] >= 0, add it. Otherwise add 0.
        // Trick: (data[i] >> 31) gives 0 for positive, -1 for negative
        int mask = ~(data[i] >> 31);  // 0xFFFFFFFF if positive, 0 if negative
        sum += data[i] & mask;
    }
    
    return sum;
}
```

**Requirements:**

1. **Implement the branchless version**

2. **Create test data:**
   ```cpp
   int main() {
       // Allocate arrays
       int* randomData = new int[SIZE];
       int* sortedData = new int[SIZE];
       
       // Fill with random positive/negative numbers
       std::random_device rd;
       std::mt19937 gen(rd());
       std::uniform_int_distribution<> dis(-100, 100);
       
       for (int i = 0; i < SIZE; i++) {
           randomData[i] = dis(gen);
           sortedData[i] = randomData[i];
       }
       
       // Sort one array
       std::sort(sortedData, sortedData + SIZE);
       
       // TODO: Benchmark all three versions on both arrays
       
       delete[] randomData;
       delete[] sortedData;
       return 0;
   }
   ```

3. **Benchmark each version:**
    - Run each function 100 times on random data
    - Run each function 100 times on sorted data
    - Measure average time using `std::chrono`
    - Calculate speedup

4. **Measure branch misses (Linux only):**
   ```bash
   # Compile
   g++ -O2 branch_test.cpp -o branch_test
   
   # Measure branch misses
   perf stat -e branches,branch-misses ./branch_test
   ```

5. **Answer these questions:**
    - Which version is fastest on random data?
    - Which version is fastest on sorted data?
    - What is the speedup from sorting (for the branching version)?
    - What is the branch misprediction rate for each?
    - When is branchless code worth it?

**Expected Results:**
```
Random data:
- Branching version: ~50% branch mispredictions, SLOW
- Branchless version: No branches, FAST

Sorted data:
- Branching version: ~0% branch mispredictions, FAST
- Branchless version: No branches, similar speed

Speedup: Sorting can make branching code 2-5× faster!
```

**Bonus Challenges:**
- Try with different data patterns (90% positive, 50% positive, etc.)
- Implement other branchless patterns (max, min, abs)
- Use compiler intrinsics: `__builtin_expect()` for hints
- Compare with SIMD version (process 8 values at once)

---

## Exercise 5: Cache Simulation (Advanced)

**Goal:** Build a simplified cache simulator to understand cache hits and misses.

### **Task:**

Implement a simple direct-mapped cache simulator:

```cpp
#include <iostream>
#include <vector>
#include <cmath>

class CacheSimulator {
private:
    struct CacheLine {
        bool valid;
        uint32_t tag;
        uint8_t data[64];  // 64-byte cache line
    };
    
    int cacheSize;      // Total cache size in bytes
    int lineSize;       // Cache line size (64 bytes)
    int numLines;       // Number of cache lines
    
    std::vector<CacheLine> cache;
    
    // Statistics
    int hits;
    int misses;
    
public:
    CacheSimulator(int cacheSizeKB) 
        : cacheSize(cacheSizeKB * 1024), 
          lineSize(64),
          numLines(cacheSize / lineSize),
          hits(0), 
          misses(0) {
        
        cache.resize(numLines);
        for (auto& line : cache) {
            line.valid = false;
            line.tag = 0;
        }
    }
    
    bool Access(uint32_t address) {
        // Extract offset, index, and tag from address
        // Offset: bits 0-5 (6 bits for 64-byte line)
        // Index: determines which cache line
        // Tag: remaining bits (check if correct data)
        
        uint32_t offset = address & 0x3F;  // Lower 6 bits
        uint32_t index = (address >> 6) & (numLines - 1);
        uint32_t tag = address >> (6 + (int)log2(numLines));
        
        // TODO: Check if hit or miss
        CacheLine& line = cache[index];
        
        if (line.valid && line.tag == tag) {
            // HIT
            hits++;
            return true;
        } else {
            // MISS - load new cache line
            line.valid = true;
            line.tag = tag;
            misses++;
            return false;
        }
    }
    
    void PrintStats() {
        int total = hits + misses;
        float hitRate = (total > 0) ? (100.0f * hits / total) : 0;
        
        std::cout << "Cache Statistics:" << std::endl;
        std::cout << "  Hits: " << hits << std::endl;
        std::cout << "  Misses: " << misses << std::endl;
        std::cout << "  Hit Rate: " << hitRate << "%" << std::endl;
    }
    
    void Reset() {
        hits = 0;
        misses = 0;
        for (auto& line : cache) {
            line.valid = false;
        }
    }
};

// Test different access patterns
void TestSequential(CacheSimulator& cache, int arraySize) {
    std::cout << "\nSequential Access Pattern:" << std::endl;
    cache.Reset();
    
    // Access array sequentially (int = 4 bytes)
    for (int i = 0; i < arraySize; i++) {
        uint32_t address = i * 4;  // Simulate int array
        cache.Access(address);
    }
    
    cache.PrintStats();
}

void TestStrided(CacheSimulator& cache, int arraySize, int stride) {
    std::cout << "\nStrided Access (stride=" << stride << "):" << std::endl;
    cache.Reset();
    
    for (int i = 0; i < arraySize; i += stride) {
        uint32_t address = i * 4;
        cache.Access(address);
    }
    
    cache.PrintStats();
}

void TestRandom(CacheSimulator& cache, int arraySize) {
    std::cout << "\nRandom Access Pattern:" << std::endl;
    cache.Reset();
    
    for (int i = 0; i < 10000; i++) {
        uint32_t address = (rand() % arraySize) * 4;
        cache.Access(address);
    }
    
    cache.PrintStats();
}
```

**Requirements:**

1. **Complete the implementation** (Access method is mostly done, verify it's correct)

2. **Test with different cache sizes:**
   ```cpp
   int main() {
       const int ARRAY_SIZE = 100000;  // 100K ints = 400 KB
       
       // Test with 32 KB cache
       CacheSimulator cache32(32);
       TestSequential(cache32, ARRAY_SIZE);
       TestStrided(cache32, ARRAY_SIZE, 16);
       TestRandom(cache32, ARRAY_SIZE);
       
       // Test with 256 KB cache
       CacheSimulator cache256(256);
       TestSequential(cache256, ARRAY_SIZE);
       TestStrided(cache256, ARRAY_SIZE, 16);
       TestRandom(cache256, ARRAY_SIZE);
       
       return 0;
   }
   ```

3. **Extend the simulator:**
    - Add a write-through policy (mark lines as dirty)
    - Implement LRU replacement for set-associative cache
    - Track the number of evictions
    - Calculate average access time (hit time + miss penalty)

4. **Answer these questions:**
    - What is the hit rate for sequential access? Why?
    - How does stride affect hit rate?
    - What is the difference between 32 KB and 256 KB cache?
    - What access pattern gives worst performance?
    - Calculate: If hit time = 4 cycles, miss time = 100 cycles, what is average?

**Bonus Challenges:**
- Implement 4-way set-associative cache
- Add L2 cache (check L1 first, then L2, then "RAM")
- Simulate the cache behavior of actual array code
- Visualize which cache lines are occupied
- Compare simulation results with real hardware (`perf stat`)

**Expected Observations:**
```
Sequential access: 
- 32 KB cache: ~93% hit rate (1 miss per 16 ints)
- 256 KB cache: ~93% hit rate (same, plenty of space)

Strided access (stride=16):
- 32 KB cache: ~93% hit rate (still sequential in cache lines)
- Stride=1024: much worse (jumping between cache lines)

Random access:
- 32 KB cache: ~40% hit rate (working set too large)
- 256 KB cache: ~90% hit rate (fits in cache)
```

---

## 🎯 Recap & Key Takeaways

1. **Building Blocks:**
   - Half Adder: Adds 2 bits
   - Full Adder: Adds 3 bits (with carry)
   - SR Latch: Set-Reset memory (stores 1 bit)
   - D Flip-Flop: Clock-controlled memory (stores 1 bit)
   - Register: Group of flip-flops (stores multiple bits)

2. **CPU Organization:**
   - Registers: Ultra-fast (1 cycle), tiny (~128 bytes)
   - Cache: Fast (4-40 cycles), small (32KB-32MB), copies of RAM
   - Stack: Slow (100+ cycles), large (megabytes), in RAM
   - Cache ≠ stack, cache ≠ registers (all separate!)

3. **Instructions:**
   - Every operation (mov, add, jmp, etc.) is one CPU instruction
   - BEQ = Branch if Equal
   - JMP = Unconditional jump

4. **Compilation Pipeline:**
   - C# → (Roslyn) → IL → (JIT/AOT) → Machine Code
   - IL: Platform-independent bytecode
   - JIT: Compile at runtime (fast iteration)
   - AOT: Compile before running (better performance)

5. **Performance Features:**
   - Pipelining: Overlap instruction stages (4× throughput)
   - Multiple cores: True parallelism
   - Multiple execution units: Execute multiple instructions/cycle
   - Out-of-order: Reorder to keep CPU busy
   - SIMD (__m128): Process 4 floats simultaneously

---

## 📚 Resources

- [Ben Eater - 8-bit CPU](https://eater.net/8bit) - Build a CPU from scratch
- [CPU Architecture](https://cpulator.01xz.net/) - Interactive ARM simulator
- [Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/) - SIMD reference
- [.NET IL Disassembler](https://sharplab.io/) - See C# → IL → Assembly
