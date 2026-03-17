# Day 1: Binary & Logic Gates - Understanding Computer Fundamentals

## 🎯 Learning Objectives

- Binary number system and why computers use it
- Hexadecimal notation and its relationship to binary
- Bit significance (LSB/MSB - Least/Most Significant Bit)
- Endianness (Little-Endian vs Big-Endian byte ordering)
- Integer representation including Two's Complement for negative numbers
- Floating-point representation (IEEE 754 standard)
- Overflow behavior and its implications in games
- Boolean logic gates and their symbols
- Bitwise operations and their practical applications
- How logic gates combine to build computational units

---

## 📚 Binary Number System

### **Why Binary?**

Computers use **binary** (base-2) because electrical circuits have two stable states:
- **0** = Low voltage (0V)
- **1** = High voltage (typically 3.3V or 5V)

Building circuits with more states (like base-10) would be:
- Less reliable (hard to distinguish 10 voltage levels)
- More expensive (complex circuitry)
- Slower (longer to stabilize)

---

### **Decimal vs Binary**

**Decimal (base-10):** Uses digits 0-9
```
Position:  10³  10²  10¹  10⁰
Value:     1000 100  10   1
Number:      4    2    5   7  = 4257

Calculation: 4×1000 + 2×100 + 5×10 + 7×1 = 4257
```

**Binary (base-2):** Uses digits 0-1
```
Position:  2⁷  2⁶  2⁵  2⁴  2³  2²  2¹  2⁰
Value:     128 64  32  16  8   4   2   1
Binary:     1   0   1   0   1   1   0   1  = 173

Calculation: 1×128 + 0×64 + 1×32 + 0×16 + 1×8 + 1×4 + 0×2 + 1×1 = 173
```

---

### **Why Read Binary from Right to Left?**

We read binary (and all positional number systems) from right to left because:

1. **The rightmost bit has the smallest value** (2⁰ = 1)
2. **Each position to the left is worth more** (doubles each time)
3. **This matches how we write numbers** (least significant on right)

```
Binary: 1 0 1 1
        ↑ ↑ ↑ ↑
        │ │ │ └─ 2⁰ = 1  (start here, smallest)
        │ │ └─── 2¹ = 2
        │ └───── 2² = 4
        └─────── 2³ = 8

Value: 8 + 0 + 2 + 1 = 11

When converting, always start from the RIGHT!
```

**Analogy:** Just like in decimal where 1234 has:
- 4 in the "ones" place (rightmost)
- 3 in the "tens" place
- 2 in the "hundreds" place
- 1 in the "thousands" place (leftmost)

---

### **Least Significant Bit (LSB) vs Most Significant Bit (MSB)**

```
Binary number: 1 0 1 1 0 1 0 0
               ↑             ↑
               MSB           LSB

MSB (Most Significant Bit):
- Leftmost bit
- Has the HIGHEST value (2⁷ = 128 in an 8-bit number)
- Changing it causes the LARGEST change to the number
- In signed integers, this is the SIGN BIT (0=positive, 1=negative)

LSB (Least Significant Bit):
- Rightmost bit
- Has the LOWEST value (2⁰ = 1)
- Changing it causes the SMALLEST change to the number
- Determines if number is EVEN (LSB=0) or ODD (LSB=1)
```

**Examples:**
```
Original:  1010 (10 in decimal)
           ↑  ↑
          MSB LSB

Flip MSB:  0010 (2 in decimal)  ← Changed by 8!
Flip LSB:  1011 (11 in decimal) ← Changed by 1

MSB has 8× more impact than LSB!
```

**Game Usage:**
```csharp
// Check if frame number is even (for alternating effects)
if ((frameCount & 1) == 0)  // Check LSB
{
    // Even frame
}

// Pack multiple booleans into one byte
byte flags = 0;
flags |= (1 << 0);  // Set LSB (flag 0)
flags |= (1 << 7);  // Set MSB (flag 7)
// flags = 10000001 in binary
```

---

### **Binary Conversion Examples**

#### **Binary to Decimal:**
```
Binary: 11010110

Step 1: Write powers of 2
Position: 7  6  5  4  3  2  1  0
Power:    2⁷ 2⁶ 2⁵ 2⁴ 2³ 2² 2¹ 2⁰
Value:    128 64 32 16 8  4  2  1
Binary:   1  1  0  1  0  1  1  0

Step 2: Multiply and add
1×128 + 1×64 + 0×32 + 1×16 + 0×8 + 1×4 + 1×2 + 0×1
= 128 + 64 + 16 + 4 + 2
= 214

Answer: 214
```

#### **Decimal to Binary:**
```
Decimal: 87

Step 1: Find largest power of 2 ≤ 87
64 (2⁶) fits, 128 (2⁷) doesn't

Step 2: Subtract and repeat
87 - 64 = 23   (place 1 in position 6)
23 - 16 = 7    (place 1 in position 4)
7 - 4 = 3      (place 1 in position 2)
3 - 2 = 1      (place 1 in position 1)
1 - 1 = 0      (place 1 in position 0)

Position: 6  5  4  3  2  1  0
Binary:   1  0  1  0  1  1  1

Answer: 1010111
```

---

## 📚 Hexadecimal and Data Representation

### **Hexadecimal (base-16)**

**Why use hex?** Binary is tedious to write and read.

```
Binary:      11010110 10111001
Hexadecimal: D6       B9

Much shorter and easier to read!
```

**Hexadecimal digits:** 0-9, A-F
```
Decimal: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
Hex:     0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
Binary:  0000 0001 0010 0011 0100 0101 0110 0111 1000 1001 1010 1011 1100 1101 1110 1111
```

**Key insight:** One hex digit = exactly 4 binary bits (a "nibble")

---

### **How to Read Hexadecimal Numbers**

**Yes, hex is also read right to left!** Just like decimal and binary.

```
Hex number: A F 2 C
            ↑ ↑ ↑ ↑
            │ │ │ └─ 16⁰ = 1     (12 × 1 = 12)
            │ │ └─── 16¹ = 16    (2 × 16 = 32)
            │ └───── 16² = 256   (15 × 256 = 3840)
            └─────── 16³ = 4096  (10 × 4096 = 40960)

Value: 40960 + 3840 + 32 + 12 = 44844

Start from the RIGHT, each position is 16× the previous!
```

**Common hex prefixes:**
```
0xAF2C   (C/C#/C++)
$AF2C    (Assembly)
#AF2C    (Colors in web/graphics)
```

---

### **How Hex Digits Combine**

**CRITICAL: Hex digits are POSITIONAL, not additive!**

```
❌ WRONG: FF = 15 + 15 = 30

✅ CORRECT: FF = (15 × 16¹) + (15 × 16⁰)
                = (15 × 16) + (15 × 1)
                = 240 + 15
                = 255

Think of it like decimal:
99 is NOT 9 + 9 = 18
99 IS (9 × 10) + (9 × 1) = 90 + 9 = 99

Same principle!
```

**More examples:**
```
0x10 = (1 × 16) + (0 × 1) = 16  (not 10!)
0xAB = (10 × 16) + (11 × 1) = 160 + 11 = 171
0x100 = (1 × 256) + (0 × 16) + (0 × 1) = 256
```

---

### **Hex to Binary Conversion (Easy!)**

Each hex digit = exactly 4 bits:

```
Hex: B  7  A  3
     ↓  ↓  ↓  ↓
Bin: 1011 0111 1010 0011

Just memorize the 16 hex digits (0-F) and their 4-bit binary!

Common values:
0 = 0000
1 = 0001
7 = 0111
8 = 1000
F = 1111
A = 1010
C = 1100
```

**Practical example: Color codes**
```
Color: #FF5733
       RR GG BB

Red:   FF = 11111111 = 255
Green: 57 = 01010111 = 87
Blue:  33 = 00110011 = 51

RGB(255, 87, 51) = Orange-red color
```

---

### **Byte, Word, and Data Sizes**

```
1 bit     = 0 or 1
1 nibble  = 4 bits = 1 hex digit
1 byte    = 8 bits = 2 hex digits
1 word    = 16 bits = 2 bytes = 4 hex digits (on 16-bit systems)
1 dword   = 32 bits = 4 bytes = 8 hex digits (double word)
1 qword   = 64 bits = 8 bytes = 16 hex digits (quad word)

Examples:
Byte:   0xFF        (8 bits)
Word:   0xABCD      (16 bits)
Dword:  0x12345678  (32 bits)
Qword:  0x123456789ABCDEF0 (64 bits)
```

**Data types:**
```csharp
byte   b = 0xFF;           // 8 bits  (0 to 255)
short  s = 0x7FFF;         // 16 bits (-32768 to 32767)
int    i = 0x7FFFFFFF;     // 32 bits (-2.1B to 2.1B)
long   l = 0x7FFFFFFFFFFFFFFF; // 64 bits (huge range)
```

---

## 📚 Integer Types and Two's Complement

### **Signed vs Unsigned Integers**

```
Unsigned (only positive):
byte:  0 to 255
ushort: 0 to 65,535
uint:  0 to 4,294,967,295

Signed (positive and negative):
sbyte:  -128 to 127
short:  -32,768 to 32,767
int:    -2,147,483,648 to 2,147,483,647

Notice: Same number of values (256, 65536, 4.3B)
Just shifted to include negatives!
```

**Why the asymmetry?** (-128 to +127, not -127 to +127)
```
We have 256 possible values (8 bits: 2⁸ = 256)

If symmetric:    -127 to +127 = 255 values (missing one!)
Two's complement: -128 to +127 = 256 values (all used!)

The "extra" negative value comes from how Two's Complement works
```

---

### **Two's Complement Explained**

**Why Two's Complement?**

Early computers tried different methods for negative numbers:
1. **Sign-Magnitude:** Use MSB as sign, rest as value
   - Problem: Two zeros (+0 and -0)
   - Problem: Addition doesn't work simply

2. **One's Complement:** Flip all bits for negative
   - Problem: Still two zeros
   - Problem: Addition requires extra step

3. **Two's Complement:** Current standard
   - One zero
   - Addition works perfectly
   - Subtraction = Addition of negative

---

### **How Two's Complement Works**

**The Bit Circle Concept:**

Think of 8-bit integers as a clock:

```
        0 (00000000)
       / \
     1     255 (11111111) = -1
    /         \
  2            254 (-2)
 /              \
127            128 (-128)
(01111111)     (10000000)

Going backwards from 0:
0 - 1 = 255 (in unsigned)
    = -1 (in signed, via Two's Complement)

The MSB being 1 means "negative side of the circle"
```

---

### **Two's Complement: Step-by-Step**

**To negate a number (make positive → negative or vice versa):**

**Method 1: Invert and Add 1**
```
Example: Find -42 in 8-bit

Step 1: Write 42 in binary
42 = 00101010

Step 2: Invert all bits (flip 0↔1)
00101010 → 11010101

Step 3: Add 1
  11010101
+        1
----------
  11010110  = -42 in Two's Complement

Verification: 11010110 = -42
Let's add 42 + (-42):
  00101010  (42)
+ 11010110  (-42)
----------
 100000000  (256, but we only have 8 bits)
Overflow bit discarded → 00000000 = 0 ✓
```

**Method 2: Keep rightmost 1 and everything right of it, flip everything left**
```
Example: -42 again

42 = 00101010
     ───────│  Find rightmost 1
            ↓
     00101010
     ││││││└─  Keep this 1 and the 0 to its right
     flip these ─────┘

     11010110 = -42
```

---

### **Why Does This Work?**

**Mathematical insight:**
```
In binary, to get negative:
-X = (2ⁿ - X)  where n = number of bits

For 8-bit:
-42 = (256 - 42) = 214

214 in binary = 11010110 ✓

This is why Two's Complement works!
When you "invert + 1", you're computing (2ⁿ - X)
```

---

### **Two's Complement Range**

**Why -128 to +127 (not symmetric)?**

```
8 bits = 256 values total

Positive numbers (MSB = 0):
00000000 = 0
00000001 = 1
00000010 = 2
...
01111111 = 127   ← Largest positive (MSB = 0)
Total: 128 values (0 to 127)

Negative numbers (MSB = 1):
10000000 = -128  ← Most negative!
10000001 = -127
10000010 = -126
...
11111110 = -2
11111111 = -1
Total: 128 values (-128 to -1)

Grand total: 128 + 128 = 256 ✓

The "extra" negative (-128) has no positive counterpart!
```

**Special case: -128 (10000000)**
```
If we try to negate -128:
Step 1: Invert bits
10000000 → 01111111

Step 2: Add 1
  01111111
+        1
----------
  10000000  = -128 again!

-128 is its own negation!
This is why abs(-128) causes overflow in 8-bit signed integers.
```

---

### **Overflow: What Happens When Numbers Get Too Big**

**Overflow** occurs when a calculation exceeds the maximum value a type can hold.

```
Example: 8-bit unsigned byte (max = 255)

255 + 1 = ?

Binary:
  11111111  (255)
+        1
----------
 100000000  (256 in 9 bits)

But we only have 8 bits!
Result: 00000000 = 0

255 + 1 = 0 (overflow wraps around!)
```

---

### **Overflow in Signed Integers**

```
Example: 8-bit signed sbyte (max = 127)

127 + 1 = ?

Binary:
  01111111  (127)
+        1
----------
  10000000  (-128)

MSB flipped from 0 to 1 → now "negative"!

127 + 1 = -128 (overflow wraps to most negative!)
```

---

### **Overflow Implications in Games**

**1. Score Wrapping**
```csharp
byte score = 255;
score += 10;
// score = 9 (wrapped around!)

// Fix: Use int or check before adding
int score = 255;
if (score <= int.MaxValue - 10)
    score += 10;
```

**2. Timer Overflow**
```csharp
float gameTime = 0f;

void Update()
{
    gameTime += Time.deltaTime;
}

// After ~5 days at 60 FPS:
// gameTime ≈ 16,777,216 seconds
// float precision breaks down (can't represent small Time.deltaTime)
// Time "stops" advancing!

// Fix: Use double, or reset timer periodically
```

**3. Position Overflow (Famous Bugs)**
```
Civilization 1 (Gandhi Bug):
- Gandhi's aggression starts at 1
- Democracy reduces aggression by 2
- 1 - 2 = -1, but stored as unsigned byte
- -1 wraps to 255 (max aggression!)
- Gandhi becomes nuclear warmonger!

Pac-Man Level 256:
- Level stored in byte (0-255)
- Level 256 overflows to 0
- Game tries to draw 256 fruits
- Half the screen becomes garbage (kill screen)

Fix: Use larger types or clamp values:
aggression = Math.Max(0, aggression - 2);
```

**4. Health Overflow**
```csharp
sbyte health = 100;
int damage = 150;

health -= (sbyte)damage;  // Casting problem!
// health = -50, wraps to 206!
// Player healed by taking damage!

// Fix: Check bounds
health = (sbyte)Math.Max(0, health - damage);
```

**5. Experience/Currency Overflow**
```csharp
int coins = 2000000000;  // Near int.MaxValue (2.1B)
coins += 200000000;
// Overflow! coins = negative value

// Fix: Use long or detect overflow
long coins = 2000000000L;
coins += 200000000;  // No overflow
```

---

### **Detecting Overflow**

```csharp
// C# checked context
checked
{
    byte b = 255;
    b += 1;  // Throws OverflowException at runtime
}

// Unchecked (default, wraps silently)
unchecked
{
    byte b = 255;
    b += 1;  // b = 0, no exception
}

// Manual check before operation
if (a > int.MaxValue - b)
{
    // Would overflow!
    Debug.LogError("Addition would overflow!");
}
else
{
    result = a + b;  // Safe
}
```

---

## 📚 Floating-Point Numbers and IEEE 754

### **What Does IEEE 754 Stand For?**

**IEEE 754:** Institute of Electrical and Electronics Engineers Standard 754

This is the **international standard** for representing decimal numbers in binary, established in 1985.

**Why needed:** Computers can't naturally represent decimals like 3.14159  
**Solution:** Split number into three parts: sign, exponent, mantissa

---

### **IEEE 754 Single Precision (float - 32 bits)**

```
Total: 32 bits = 1 + 8 + 23

┌─┬────────┬───────────────────────┐
│S│Exponent│      Mantissa         │
│ │ (8 bit)│      (23 bit)         │
└─┴────────┴───────────────────────┘
 1    8             23 bits

S = Sign bit (0 = positive, 1 = negative)
Exponent = Power of 2 (biased by 127)
Mantissa = Fractional part (also called "significand")
```

---

### **How IEEE 754 Works (Example)**

**Representing 12.375:**

**Step 1: Convert to binary**
```
Integer part: 12 = 1100
Decimal part: 0.375
  0.375 × 2 = 0.75  → 0
  0.75 × 2 = 1.5    → 1
  0.5 × 2 = 1.0     → 1
  
12.375 = 1100.011 in binary
```

**Step 2: Normalize (scientific notation)**
```
1100.011 = 1.100011 × 2³

Move decimal point until one digit left of point
Count how many places moved = exponent
```

**Step 3: Extract parts**
```
Sign: Positive → 0

Exponent: 3
Add bias (127): 3 + 127 = 130
130 in binary = 10000010

Mantissa: 100011 (drop the leading 1)
Pad with zeros: 10001100000000000000000 (23 bits)

Final 32-bit representation:
0 10000010 10001100000000000000000
│     │              │
Sign  Exp          Mantissa
```

---

### **Special IEEE 754 Values**

```
Zero:
Sign = 0 or 1, Exponent = 0, Mantissa = 0
+0.0: 0 00000000 00000000000000000000000
-0.0: 1 00000000 00000000000000000000000
(Yes, two zeros exist!)

Infinity:
Exponent = all 1s (255), Mantissa = 0
+∞: 0 11111111 00000000000000000000000
-∞: 1 11111111 00000000000000000000000

NaN (Not a Number):
Exponent = all 1s (255), Mantissa ≠ 0
NaN: X 11111111 XXXXXXXXXXXXXXXXXXXXXXX
```

---

### **Floating-Point Precision Issues**

```csharp
// Problem 1: Not all decimals representable exactly
float a = 0.1f;
float b = 0.2f;
float c = a + b;

Debug.Log(c);  // 0.30000001 (not exactly 0.3!)
Debug.Log(c == 0.3f);  // False!

// Fix: Use epsilon comparison
const float epsilon = 0.0001f;
if (Mathf.Abs(c - 0.3f) < epsilon)
{
    Debug.Log("Close enough!");
}
```

**Why?** 0.1 in decimal = infinite repeating binary (like 1/3 in decimal = 0.333...)
```
0.1 decimal = 0.00011001100110011... (binary, repeating)

Float only has 23 bits for mantissa → must round
This introduces tiny errors
```

---

### **Float vs Double Precision**

```
float (32-bit):
- 1 sign bit
- 8 exponent bits
- 23 mantissa bits
- ~7 decimal digits precision
- Range: ±1.5 × 10⁻⁴⁵ to ±3.4 × 10³⁸

double (64-bit):
- 1 sign bit
- 11 exponent bits
- 52 mantissa bits
- ~15-16 decimal digits precision
- Range: ±5.0 × 10⁻³²⁴ to ±1.7 × 10³⁰⁸

Recommendation for games: Use float unless you need precision
(double is 2x memory and slower on GPU)
```

---

### **Game-Specific Float Issues**

```csharp
// Issue 1: Large coordinates lose precision
Vector3 farPosition = new Vector3(1000000f, 0f, 0f);
farPosition.x += 0.01f;  // Might not actually change!

// At 1,000,000, float precision is ~0.0625
// Adding 0.01 is smaller than precision → ignored!

// Fix: Use relative coordinates (origin nearby)

// Issue 2: Timer accumulation
float totalTime = 0f;
void Update()
{
    totalTime += Time.deltaTime;  // Eventually loses precision
}

// After several hours, Time.deltaTime (0.016s) is smaller
// than precision at totalTime → stops incrementing!

// Fix: Use double or reset periodically

// Issue 3: Comparison
if (transform.position.y == 0f)  // Almost never true!
{
    // Floating point math rarely produces exact values
}

// Fix: Use epsilon
if (Mathf.Abs(transform.position.y) < 0.001f)
{
    // Close enough to ground
}
```

---

## 📚 Boolean Logic and Gate Symbols

### **Boolean Logic Gates**

Logic gates are physical electronic circuits that perform Boolean operations.

Each gate:
- Has 1-2 input wires (carrying 0 or 1)
- Has 1 output wire (producing 0 or 1)
- Performs a specific logical operation
- Is built from transistors (switches made of silicon)

---

### **Gate Symbols and What They Convey**

**The symbols are standardized (IEEE/ANSI) and convey:**
1. **Shape:** Identifies the gate type
2. **Inputs:** Lines on the left (usually 2, sometimes more)
3. **Output:** Line on the right (always 1)
4. **Bubble (○):** Means "NOT" or "inverted"

---

### **Basic Gate Symbols**

#### **1. AND Gate**
```
Symbol:
    A ──┐
        ├──[D]── Output
    B ──┘

Shape: D-shaped, flat on input side
Meaning: Output is 1 only if A AND B are both 1

Truth Table:
A │ B │ Out
──┼───┼────
0 │ 0 │ 0
0 │ 1 │ 0
1 │ 0 │ 0
1 │ 1 │ 1   ← Only case where output is 1

Real-world analogy:
Two switches in SERIES (both must be closed for current to flow)
```

#### **2. OR Gate**

```
Symbol:
    A ──┐
        )──── Output
    B ──┘

Shape: Curved on both sides (like )
Meaning: Output is 1 if A OR B (or both) is 1

Truth Table:
A │ B │ Out
──┼───┼────
0 │ 0 │ 0   ← Only case where output is 0
0 │ 1 │ 1
1 │ 0 │ 1
1 │ 1 │ 1

Real-world analogy:
Two switches in PARALLEL (either can complete the circuit)
```

#### **3. NOT Gate (Inverter)**

```
Symbol:
    A ──▷○── Output

Shape: Triangle with bubble at output
Bubble (○): Means "NOT" or "invert"
Meaning: Output is opposite of input

Truth Table:
A │ Out
──┼────
0 │ 1
1 │ 0

Real-world analogy:
A switch that does the opposite (normally-closed relay)
```

#### **4. XOR Gate (Exclusive OR)**

```
Symbol:
    A ──┐
        )=── Output
    B ──┘

Shape: Like OR but with extra curved line on input side
Meaning: Output is 1 if inputs are DIFFERENT

Truth Table:
A │ B │ Out
──┼───┼────
0 │ 0 │ 0
0 │ 1 │ 1   ← Different
1 │ 0 │ 1   ← Different
1 │ 1 │ 0

Real-world analogy:
Light switch at top AND bottom of stairs
(flip either one to toggle the light)
```

---

### **Compound Gates (Made from Basic Gates)**

#### **5. NAND Gate (NOT-AND)**

```
Symbol:
    A ──┐
        ├──[D]○── Output
    B ──┘
          ↑
        Bubble = NOT

Meaning: AND followed by NOT
Output is 0 only if A AND B are both 1

Truth Table:
A │ B │ Out
──┼───┼────
0 │ 0 │ 1
0 │ 1 │ 1
1 │ 0 │ 1
1 │ 1 │ 0   ← Only 0 case

Special property: NAND is "universal"
(can build ANY gate using only NAND gates!)
```

#### **6. NOR Gate (NOT-OR)**

```
Symbol:
    A ──┐
        )○── Output
    B ──┘
         ↑
       Bubble = NOT

Meaning: OR followed by NOT
Output is 1 only if both A AND B are 0

Truth Table:
A │ B │ Out
──┼───┼────
0 │ 0 │ 1   ← Only 1 case
0 │ 1 │ 0
1 │ 0 │ 0
1 │ 1 │ 0

Special property: NOR is also "universal"
(can build ANY gate using only NOR gates!)
```

---

### **What the Symbols Convey Physically**

**Each symbol represents:**

1. **Physical Circuit:** Made of transistors
```
AND gate = 4-6 transistors
NOT gate = 2 transistors
```

2. **Electrical Behavior:**
```
Input signals: Voltage levels (0V or 5V)
Output signal: Resulting voltage based on gate logic
Propagation delay: Time for output to change (nanoseconds)
```

3. **Boolean Function:**
```
AND: Output = A · B (multiplication)
OR:  Output = A + B (addition)
NOT: Output = Ā (bar over A)
XOR: Output = A ⊕ B (exclusive or symbol)
```

4. **Building Block:**
```
Gates combine to form:
- Adders (for arithmetic)
- Multiplexers (for selection)
- Memory cells (for storage)
- ALUs (for computation)
```

---

### **Reading Circuit Diagrams**

```
Example: (A AND B) OR C

    A ──┐
        ├──[AND]──┐
    B ──┘         │
                  )──[OR]── Output
    C ────────────┘

Reading left to right:
1. A and B go into AND gate
2. AND output goes into OR gate
3. C also goes into OR gate
4. Final output = (A AND B) OR C

Truth Table:
A │ B │ C │(A∧B)│ Out
──┼───┼───┼─────┼────
0 │ 0 │ 0 │  0  │ 0
0 │ 0 │ 1 │  0  │ 1
0 │ 1 │ 0 │  0  │ 0
0 │ 1 │ 1 │  0  │ 1
1 │ 0 │ 0 │  0  │ 0
1 │ 0 │ 1 │  0  │ 1
1 │ 1 │ 0 │  1  │ 1   ← AND produced 1
1 │ 1 │ 1 │  1  │ 1
```

---

## 📚 Bitwise Operations in Detail

### **Bitwise vs Logical Operators**

```csharp
// Logical (for booleans, returns bool)
bool a = true;
bool b = false;
bool result = a && b;  // Logical AND: false

// Bitwise (for integers, returns int)
int x = 5;   // 0101
int y = 3;   // 0011
int result = x & y;  // Bitwise AND: 0001 = 1

Key difference:
Logical: Works on true/false
Bitwise: Works on each bit individually
```

---

### **Basic Bitwise Operators**

#### **1. AND (&) - Single Ampersand**

```csharp
int a = 12;  // 1100
int b = 10;  // 1010
int c = a & b;

Bit-by-bit:
  1100
& 1010
------
  1000  = 8

c = 8

Rule: Output bit is 1 only if BOTH input bits are 1
```

**Usage: Bit Masking (Check/Extract specific bits)**
```csharp
// Check if bit 2 is set
int flags = 0b1010;  // Binary literal
bool bit2Set = (flags & 0b0100) != 0;  // Mask with 0100
// 1010 & 0100 = 0000 → false

bool bit3Set = (flags & 0b1000) != 0;
// 1010 & 1000 = 1000 → true

// Extract lower 4 bits (nibble)
int value = 0xA7;  // 10100111
int lowerNibble = value & 0x0F;  // 0F = 00001111
// 10100111 & 00001111 = 00000111 = 7
```

---

#### **2. OR (|) - Single Pipe**

```csharp
int a = 12;  // 1100
int b = 10;  // 1010
int c = a | b;

Bit-by-bit:
  1100
| 1010
------
  1110  = 14

c = 14

Rule: Output bit is 1 if EITHER input bit is 1
```

**Usage: Setting Bits**
```csharp
// Set bit 2
int flags = 0b1000;  // 1000
flags |= 0b0100;     // Set bit 2
// 1000 | 0100 = 1100

// Set multiple bits at once
int permissions = 0;
permissions |= READ_FLAG;   // Set read
permissions |= WRITE_FLAG;  // Set write
// permissions now has both flags set
```

---

#### **3. XOR (^) - Caret**

```csharp
int a = 12;  // 1100
int b = 10;  // 1010
int c = a ^ b;

Bit-by-bit:
  1100
^ 1010
------
  0110  = 6

c = 6

Rule: Output bit is 1 if input bits are DIFFERENT
```

**Usage: Toggle Bits**
```csharp
// Toggle bit 2
int flags = 0b1100;  // 1100
flags ^= 0b0100;     // Toggle bit 2
// 1100 ^ 0100 = 1000 (bit 2 turned OFF)

flags ^= 0b0100;     // Toggle again
// 1000 ^ 0100 = 1100 (bit 2 turned ON)

// Swap without temp variable (XOR trick)
int a = 5, b = 7;
a ^= b;  // a = 5 ^ 7
b ^= a;  // b = 7 ^ (5 ^ 7) = 5
a ^= b;  // a = (5 ^ 7) ^ 5 = 7
// a and b swapped!
```

---

#### **4. NOT (~) - Tilde**

```csharp
int a = 12;  // 00001100 (8-bit for clarity)
int b = ~a;

Bit-by-bit:
  00001100
~
-----------
  11110011  = -13 (in Two's Complement!)

b = -13

Rule: Flip every bit (0→1, 1→0)
```

**Why ~12 = -13?**
```
In Two's Complement:
~X = -(X + 1)

~12 = -(12 + 1) = -13

Verification:
12 in binary:  00001100
Flip all bits: 11110011
This is -13 in Two's Complement!

To see: Apply Two's Complement to get positive
Invert: 00001100
Add 1:  00001101 = 13
So 11110011 = -13 ✓
```

**Usage: Clearing Bits**
```csharp
// Clear bit 2
int flags = 0b1110;
flags &= ~0b0100;  // AND with NOT of bit 2
// 1110 & 1011 = 1010 (bit 2 cleared)

// Why AND with NOT works:
// ~0b0100 = 0b1011 (all bits set EXCEPT bit 2)
// ANDing preserves all bits except where mask is 0
```

---

### **Compound Assignment Operators**

#### **1. AND Assignment (&=)**

```csharp
int flags = 0b1111;
flags &= 0b1100;  // Keep only bits 2 and 3

// Equivalent to:
flags = flags & 0b1100;

// 1111 & 1100 = 1100
// flags = 12

Usage: Clear specific bits
flags &= ~(1 << 2);  // Clear bit 2
```

#### **2. OR Assignment (|=)**

```csharp
int flags = 0b1000;
flags |= 0b0010;  // Set bit 1

// Equivalent to:
flags = flags | 0b0010;

// 1000 | 0010 = 1010
// flags = 10

Usage: Set bits (most common)
flags |= INVINCIBLE_BIT;
```

#### **3. XOR Assignment (^=)**

```csharp
int flags = 0b1010;
flags ^= 0b0110;  // Toggle bits 1 and 2

// Equivalent to:
flags = flags ^ 0b0110;

// 1010 ^ 0110 = 1100
// flags = 12

Usage: Toggle bits
flags ^= FLASH_BIT;  // Toggle flashing on/off
```

---

### **Single & in If Statements**

```csharp
// Double && (Logical AND)
if (health > 0 && isAlive)
{
    // Short-circuit: If health <= 0, doesn't check isAlive
}

// Single & (Bitwise AND, but works on bools too!)
if (health > 0 & isAlive)
{
    // NO short-circuit: Always evaluates BOTH conditions
}

Difference:
&& : Stops at first false (faster, preferred)
&  : Evaluates both sides always (rare use)

When to use single &:
if ((UpdatePlayer() & UpdateEnemy()) == true)
{
    // Both functions must run even if first returns false
}

Usually, use && for if statements!
```

---

### **Bit Shifting**

**Bit shifting moves all bits left or right.**

#### **Left Shift (<<)**

```csharp
int a = 5;    // 0101
int b = a << 2;

Bit-by-bit:
  0101
<< 2 (shift left 2 positions)
------
  10100  = 20

b = 20

What happens:
- Bits move LEFT
- Zeros fill in from right
- Leftmost bits fall off

Mathematical meaning:
x << n  =  x × 2ⁿ

5 << 2 = 5 × 2² = 5 × 4 = 20 ✓
```

**Usage: Fast Multiplication by Powers of 2**
```csharp
// Multiply by 2: x << 1
int doubled = value << 1;  // 2x faster than value * 2

// Multiply by 4: x << 2
int quadrupled = value << 2;

// Multiply by 16: x << 4
int times16 = value << 4;

// Create bit masks
int bit5 = 1 << 5;  // 00100000 (bit 5 set)
int bit0 = 1 << 0;  // 00000001 (bit 0 set)
```

---

#### **Right Shift (>>)**

```csharp
int a = 20;   // 10100
int b = a >> 2;

Bit-by-bit:
  10100
>> 2 (shift right 2 positions)
------
  00101  = 5

b = 5

What happens:
- Bits move RIGHT
- Leftmost bits filled based on sign:
  * Unsigned: Fill with 0
  * Signed positive: Fill with 0
  * Signed negative: Fill with 1 (preserve sign)
- Rightmost bits fall off

Mathematical meaning:
x >> n  =  x ÷ 2ⁿ (integer division)

20 >> 2 = 20 ÷ 4 = 5 ✓
```

**Usage: Fast Division by Powers of 2**
```csharp
// Divide by 2: x >> 1
int halved = value >> 1;

// Divide by 4: x >> 2
int quartered = value >> 2;

// Extract specific bits
int byte2 = (value >> 16) & 0xFF;  // Extract bits 16-23

// Color unpacking
Color Unpack(uint packed)
{
    byte r = (byte)((packed >> 24) & 0xFF);
    byte g = (byte)((packed >> 16) & 0xFF);
    byte b = (byte)((packed >> 8) & 0xFF);
    byte a = (byte)(packed & 0xFF);
    return new Color32(r, g, b, a);
}
```

---

### **Practical Game Examples**

```csharp
// Example 1: Player State Flags (instead of multiple bools)
[Flags]
enum PlayerState
{
    None = 0,           // 00000000
    Grounded = 1 << 0,  // 00000001
    Jumping = 1 << 1,   // 00000010
    Attacking = 1 << 2, // 00000100
    Invincible = 1 << 3,// 00001000
    Poisoned = 1 << 4,  // 00010000
}

PlayerState state = PlayerState.None;

// Set flags
state |= PlayerState.Grounded;
state |= PlayerState.Jumping;
// state = 00000011 (both flags set)

// Check flag
if ((state & PlayerState.Invincible) != 0)
{
    // Player is invincible
}

// Clear flag
state &= ~PlayerState.Jumping;
// state = 00000001 (jumping cleared)

// Toggle flag
state ^= PlayerState.Attacking;

// Check multiple flags
if ((state & (PlayerState.Grounded | PlayerState.Attacking)) ==
    (PlayerState.Grounded | PlayerState.Attacking))
{
    // Player is grounded AND attacking
}
```

```csharp
// Example 2: Pack Color into 32-bit Integer
uint PackColor(Color32 color)
{
    return ((uint)color.r << 24) |
           ((uint)color.g << 16) |
           ((uint)color.b << 8) |
           ((uint)color.a);
}

// 255, 128, 64, 32 packed:
// RRGGBBAA
// 11111111 10000000 01000000 00100000
// = 0xFF804020
```

```csharp
// Example 3: Fast Even/Odd Check
bool IsEven(int n)
{
    return (n & 1) == 0;  // Check LSB
    // Much faster than: n % 2 == 0
}

// Example 4: Power of Two Check
bool IsPowerOfTwo(int n)
{
    return (n > 0) && ((n & (n - 1)) == 0);
    
    // Why this works:
    // Powers of 2 have exactly one bit set:
    // 4 = 0100
    // 4-1 = 3 = 0011
    // 4 & 3 = 0000 ✓
    
    // Non-powers have multiple bits:
    // 6 = 0110
    // 6-1 = 5 = 0101
    // 6 & 5 = 0100 (not zero)
}
```

---

## 📚 Building Computational Units from Gates

### **What is an Adder?**

An **adder** is a digital circuit that performs addition of binary numbers.

**Purpose:** Arithmetic operations are fundamental to computing  
**Built from:** Logic gates (AND, OR, XOR)  
**Types:** Half Adder, Full Adder, Ripple-Carry Adder

---

### **Half Adder: Adding Two Bits**

**The Problem:**
```
Add two 1-bit numbers:
0 + 0 = 00
0 + 1 = 01
1 + 0 = 01
1 + 1 = 10  ← Result is 2 bits! (Sum=0, Carry=1)
```

**The Circuit:**
```
Inputs: A, B
Outputs: Sum, Carry

Logic:
Sum = A XOR B    (1 if different)
Carry = A AND B  (1 if both 1)

Truth Table:
A │ B │ Sum │ Carry
──┼───┼─────┼──────
0 │ 0 │  0  │  0
0 │ 1 │  1  │  0
1 │ 0 │  1  │  0
1 │ 1 │  0  │  1    ← 1+1 = 10 binary

Circuit Diagram:
A ──┬──[XOR]── Sum
    │
B ──┼──[AND]── Carry
    │
```

**Why "Half"?**
Can only add two bits. Doesn't handle carry-in from previous position.

---

### **Full Adder: Adding Three Bits**

**The Problem:**
```
When adding multi-bit numbers, need to include carry from previous position:

Example: Add 11 + 01
Position 0: 1 + 1 = 10 (Sum=0, Carry=1)
Position 1: 1 + 0 + Carry(1) = 10 (Sum=0, Carry=1)

Need to add THREE bits: A + B + Carry-In
```

**The Circuit:**
```
Inputs: A, B, Carry_In
Outputs: Sum, Carry_Out

Built from TWO half adders:

       A ──┐
           ├──[Half Adder 1]── Sum1, Carry1
       B ──┘
       
  Sum1 ────┐
           ├──[Half Adder 2]── Sum (final)
Carry_In ──┘                     Carry2
           
Carry_Out = Carry1 OR Carry2

Logic equations:
Sum = (A XOR B) XOR Carry_In
Carry_Out = (A AND B) OR (Carry_In AND (A XOR B))
```

**Truth Table:**
```
A │ B │C_in│Sum│C_out
──┼───┼────┼───┼─────
0 │ 0 │ 0  │ 0 │  0
0 │ 0 │ 1  │ 1 │  0
0 │ 1 │ 0  │ 1 │  0
0 │ 1 │ 1  │ 0 │  1
1 │ 0 │ 0  │ 1 │  0
1 │ 0 │ 1  │ 0 │  1
1 │ 1 │ 0  │ 0 │  1
1 │ 1 │ 1  │ 1 │  1   ← 1+1+1 = 11 binary
```

---

### **4-Bit Ripple-Carry Adder**

**Chain four full adders to add 4-bit numbers:**

```
    A₃B₃  A₂B₂  A₁B₁  A₀B₀
      ↓↓    ↓↓    ↓↓    ↓↓
     [FA3] [FA2] [FA1] [FA0]
      ↓C    ↓C    ↓C    ↓C
    S₃    S₂    S₁    S₀
    
C = Carry connection between adders
S = Sum output
FA = Full Adder

Example: 1101 + 0011
         (13)  (3)

Position 0:
A₀=1, B₀=1, C_in=0
Sum₀ = 0, C_out = 1

Position 1:
A₁=0, B₁=1, C_in=1
Sum₁ = 0, C_out = 1

Position 2:
A₂=1, B₂=0, C_in=1
Sum₂ = 0, C_out = 1

Position 3:
A₃=1, B₃=0, C_in=1
Sum₃ = 0, C_out = 1

Result: 1 0000 (16 in 5 bits)
        ↑ ↑↑↑↑
      overflow S₃S₂S₁S₀
```

**Properties:**
- **Gate Count:** ~20 gates per full adder × 4 = 80 gates
- **Propagation Delay:** Carry "ripples" through each stage
   - Each gate: ~1 nanosecond
   - 4-bit addition: ~10 nanoseconds total
- **Scaling:** 64-bit adder = 64 full adders in chain

**This is how your CPU adds numbers!**

---

### **From Adders to ALU**

The **Arithmetic Logic Unit (ALU)** combines multiple circuits:

```
        ┌─────────────────────────┐
        │          ALU            │
        │                         │
Inputs: │  A (64 bits)            │
        │  B (64 bits)            │
        │  Operation (4 bits)     │
        │                         │
        │  ┌──────────┐           │
        │  │ Adder    │ (for +/-) │
        │  ├──────────┤           │
        │  │ AND      │ (bitwise) │
        │  ├──────────┤           │
        │  │ OR       │ (bitwise) │
        │  ├──────────┤           │
        │  │ XOR      │ (bitwise) │
        │  ├──────────┤           │
        │  │ Shifter  │ (<<, >>)  │
        │  ├──────────┤           │
        │  │ Compare  │ (>, <, ==)│
        │  └──────────┘           │
        │       ↓                 │
        │  [Multiplexer]          │
        │   Selects result        │
        │   based on operation    │
        │       ↓                 │
Output: │  Result (64 bits)       │
        │  Flags (Z, N, C, V)     │
        └─────────────────────────┘

Operation codes:
0000 = ADD
0001 = SUB
0010 = AND
0011 = OR
0100 = XOR
0101 = SHL (shift left)
0110 = SHR (shift right)
0111 = CMP (compare)
...
```

**ALU Output Flags:**
```
Z (Zero): Result is zero
N (Negative): Result is negative (MSB = 1)
C (Carry): Addition produced carry-out
V (Overflow): Signed overflow occurred
```

---

## 📚 Assembly Instructions and Low-Level Code

### **What is Assembly Language?**

**Assembly:** Human-readable representation of machine code  
**Machine Code:** Binary instructions the CPU executes directly

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

### **Common CPU Instructions**

**Instruction format:** `OPCODE destination, source`

| Instruction | Full Name | Description | Example | Explanation |
|-------------|-----------|-------------|---------|-------------|
| **mov** | Move | Copy data | `mov rax, 42` | rax = 42 |
| **movw** | Move Word | Copy 16-bit data | `movw ax, 100` | ax = 100 (16-bit) |
| **movl** | Move Long | Copy 32-bit data | `movl eax, 1000` | eax = 1000 (32-bit) |
| **movq** | Move Quad | Copy 64-bit data | `movq rax, 10000` | rax = 10000 (64-bit) |
| **add** | Add | Add two values | `add rax, rbx` | rax = rax + rbx |
| **addw** | Add Word | Add 16-bit values | `addw ax, bx` | ax = ax + bx (16-bit) |
| **addl** | Add Long | Add 32-bit values | `addl eax, ebx` | eax = eax + ebx (32-bit) |
| **addq** | Add Quad | Add 64-bit values | `addq rax, rbx` | rax = rax + rbx (64-bit) |
| **sub** | Subtract | Subtract values | `sub rax, rbx` | rax = rax - rbx |
| **imul** | Integer Multiply | Multiply (signed) | `imul eax, ebx` | eax = eax × ebx (32-bit signed) |
| **imulq** | Integer Multiply Quad | Multiply 64-bit (signed) | `imulq rax, rbx` | rax = rax × rbx (64-bit signed) |
| **mul** | Multiply | Multiply (unsigned) | `mul ebx` | eax = eax × ebx (unsigned) |
| **idiv** | Integer Divide | Divide (signed) | `idiv ebx` | eax = eax ÷ ebx (quotient), edx = remainder |
| **div** | Divide | Divide (unsigned) | `div ebx` | eax = eax ÷ ebx (unsigned) |
| **and** | Bitwise AND | AND operation | `and rax, rbx` | rax = rax & rbx |
| **or** | Bitwise OR | OR operation | `or rax, rbx` | rax = rax \| rbx |
| **xor** | Bitwise XOR | XOR operation | `xor rax, rax` | rax = 0 (common idiom) |
| **not** | Bitwise NOT | NOT operation | `not rax` | rax = ~rax |
| **shl** | Shift Left | Shift bits left | `shl rax, 2` | rax = rax << 2 |
| **shr** | Shift Right | Shift bits right | `shr rax, 2` | rax = rax >> 2 |
| **cmp** | Compare | Compare values | `cmp rax, rbx` | Sets flags based on rax - rbx |
| **jmp** | Jump | Unconditional jump | `jmp LABEL` | Go to LABEL |
| **je** | Jump if Equal | Jump if equal | `je LABEL` | Jump if Z flag set |
| **jne** | Jump if Not Equal | Jump if not equal | `jne LABEL` | Jump if Z flag clear |
| **jg** | Jump if Greater | Jump if greater | `jg LABEL` | Jump if rax > rbx (after cmp) |
| **jl** | Jump if Less | Jump if less | `jl LABEL` | Jump if rax < rbx (after cmp) |
| **call** | Call Function | Function call | `call FUNC` | Push return address, jump to FUNC |
| **ret** | Return | Function return | `ret` | Pop return address, jump back |
| **push** | Push to Stack | Save to stack | `push rax` | Stack grows, store rax |
| **pop** | Pop from Stack | Restore from stack | `pop rax` | Retrieve rax, stack shrinks |
| **lea** | Load Effective Address | Calculate address | `lea rax, [rbx+4]` | rax = address of (rbx + 4) |
| **nop** | No Operation | Do nothing | `nop` | Wastes one clock cycle (for alignment) |

---

### **Instruction Suffixes (x86-64)**

The suffix indicates the **operand size:**

| Suffix | Size | Bits | Example Register | Range |
|--------|------|------|------------------|-------|
| **b** | Byte | 8 | al, bl, cl | 0-255 / -128 to 127 |
| **w** | Word | 16 | ax, bx, cx | 0-65535 / -32768 to 32767 |
| **l** | Long | 32 | eax, ebx, ecx | 0-4.3B / -2.1B to 2.1B |
| **q** | Quad | 64 | rax, rbx, rcx | 0-18.4Q / huge range |

**Examples:**
```assembly
movb al, 42     ; Move 8-bit value 42 into al
movw ax, 1000   ; Move 16-bit value 1000 into ax
movl eax, 100000 ; Move 32-bit value 100000 into eax
movq rax, 1000000000 ; Move 64-bit value into rax
```

---

### **Example: C# to Assembly**

```csharp
int a = 10;
int b = 20;
int sum = a + b;
```

**Assembly (x86-64):**
```assembly
; Allocate variables on stack
movl $10, -4(%rbp)     ; int a = 10  (store at [rbp-4])
movl $20, -8(%rbp)     ; int b = 20  (store at [rbp-8])

; Load into registers
movl -4(%rbp), %eax    ; Load a into eax
movl -8(%rbp), %ebx    ; Load b into ebx

; Perform addition
addl %ebx, %eax        ; eax = eax + ebx

; Store result
movl %eax, -12(%rbp)   ; int sum = eax (store at [rbp-12])
```

---

### **SIMD Instructions (Single Instruction, Multiple Data)**

**SIMD:** Single Instruction, Multiple Data

Modern CPUs have special instructions that operate on multiple data points simultaneously.

**Without SIMD (scalar):**
```assembly
; Add 4 floats (4 instructions)
addss xmm0, xmm1    ; xmm0[0] += xmm1[0]
addss xmm0, xmm1    ; xmm0[1] += xmm1[1]
addss xmm0, xmm1    ; xmm0[2] += xmm1[2]
addss xmm0, xmm1    ; xmm0[3] += xmm1[3]

Total: 4 instructions
```

**With SIMD (vectorized):**
```assembly
; Add 4 floats (1 instruction!)
addps xmm0, xmm1    ; xmm0[0-3] += xmm1[0-3] simultaneously

Total: 1 instruction (4× faster!)
```

**SIMD Instruction Sets:**
- **SSE (Streaming SIMD Extensions):** 128-bit registers (4 floats)
- **AVX (Advanced Vector Extensions):** 256-bit registers (8 floats)
- **AVX-512:** 512-bit registers (16 floats)

**Unity Burst Compiler** automatically uses SIMD for compatible code!

```csharp
// C# with Burst
[BurstCompile]
public struct VectorAddJob : IJobParallelFor
{
    public NativeArray<float4> a;
    public NativeArray<float4> b;
    public NativeArray<float4> result;
    
    public void Execute(int index)
    {
        result[index] = a[index] + b[index];  // Burst uses SIMD!
    }
}

// Compiles to single SIMD instruction:
// vaddps ymm0, ymm1, ymm2   (AVX: 8 floats at once!)
```

---

## 🎯 Recap & Practical Exercises

### **Key Concepts Summary**

#### **1. Number Systems**
```
Binary (base-2): Read right to left
MSB (Most Significant Bit): Leftmost, highest value
LSB (Least Significant Bit): Rightmost, determines even/odd

Hexadecimal (base-16): Read right to left
Each hex digit = 4 binary bits
FF = (15×16) + (15×1) = 255, NOT 15+15=30!
```

#### **2. Two's Complement**
```
To negate: Invert bits + add 1
Why asymmetric? -128 to +127 uses all 256 values
Overflow wraps: 127 + 1 = -128
```

#### **3. IEEE 754 Floating-Point**
```
IEEE 754: Institute of Electrical and Electronics Engineers Standard 754
float: 1 sign + 8 exponent + 23 mantissa = 32 bits
Precision issues: 0.1 + 0.2 ≠ 0.3 exactly
Use epsilon for comparisons
```

#### **4. Endianness**
```
Little-Endian: LSB stored at lowest address (x86)
Big-Endian: MSB stored at lowest address (network)

Example: 0x12345678 in memory
Little: 78 56 34 12
Big:    12 34 56 78
```

#### **5. Logic Gates**
```
AND: Output 1 only if both inputs 1
OR: Output 1 if any input 1
NOT: Flip input
XOR: Output 1 if inputs different
NAND/NOR: Universal (can build any gate)
```

#### **6. Bitwise Operations**
```
& (AND): Bit masking, checking bits
| (OR): Setting bits
^ (XOR): Toggling bits
~ (NOT): Flipping all bits, clearing bits (~X = -(X+1))
<< (Left shift): Multiply by 2ⁿ
>> (Right shift): Divide by 2ⁿ

Single & in if: No short-circuit (evaluates both sides)
```

#### **7. Adders**
```
Half Adder: Adds 2 bits (Sum, Carry)
Full Adder: Adds 3 bits (A, B, Carry-In)
Ripple-Carry: Chain of full adders for multi-bit addition
ALU: Combines adders, logic gates, shifters
```

---

### **Practical Exercises**

**Exercise 1: Binary/Hex Conversion**
```
Convert to binary:
a) 173
b) 0xAF
c) 0xFF

Convert to hex:
a) 11010110
b) 255
c) 1000000000 (binary)
```

**Exercise 2: Two's Complement**
```
Find the Two's Complement representation (8-bit):
a) -42
b) -1
c) -128

What decimal values do these represent (8-bit signed)?
a) 11111111
b) 10000000
c) 01111111
```

**Exercise 3: Bitwise Operations**
```csharp
int flags = 0b10110010;

// What is the result of:
a) flags & 0b00001111  (extract lower nibble)
b) flags | 0b01000000  (set bit 6)
c) flags ^ 0b10000000  (toggle bit 7)
d) ~flags              (flip all bits)
e) flags << 2          (shift left 2)
f) flags >> 3          (shift right 3)
```

**Exercise 4: Overflow Detection**
```csharp
// Will these overflow? (8-bit unsigned byte)
a) 200 + 100
b) 150 + 50
c) 255 + 1

// What about signed sbyte (-128 to 127)?
a) 100 + 50
b) -100 - 50
c) 127 + 1
```

**Exercise 5: Float Precision**
```csharp
// Which comparisons are safe?
a) (0.1f + 0.2f) == 0.3f
b) Mathf.Abs((0.1f + 0.2f) - 0.3f) < 0.0001f
c) transform.position.y == 0f
d) Mathf.Approximately(transform.position.y, 0f)
```

---

## 📚 Additional Resources

- [Binary Tutorial](https://www.tutorialspoint.com/computer_logical_organization/number_system_conversion.htm)
- [Two's Complement Explained](https://www.cs.cornell.edu/~tomf/notes/cps104/twoscomp.html)
- [IEEE 754 Visualization](https://float.exposed/)
- [Logic Gates Simulator](https://logic.ly/demo/)
- [Ben Eater - Binary Addition](https://www.youtube.com/watch?v=wvJc9CZcvBc)