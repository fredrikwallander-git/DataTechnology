# Day 3: Memory Hierarchy & Performance Profiling (C++)

## 🎯 Learning Objectives

- The complete memory hierarchy from registers to disk storage
- How caches work in detail (cache lines, sets, ways, replacement policies)
- Memory access patterns and their dramatic performance impact
- What makes code cache-friendly vs cache-hostile
- Virtual memory, paging, and the Translation Lookaside Buffer (TLB)
- Memory management strategies (stack, heap, memory pools) in C++
- How to profile C++ game code to find performance bottlenecks
- Data-Oriented Design principles for high-performance C++ games
- Practical optimization techniques with measurable results

---

## 📚 Memory Hierarchy Deep Dive

### **The Memory Pyramid**

```
               Speed ↑              Size ↓              Cost ↑
               
    ┌─────────────────────┐
    │   CPU Registers     │  1 cycle   (~0.3 ns)    32 × 64-bit   
    ├─────────────────────┤
    │     L1 Cache        │  4 cycles  (~1 ns)      32-64 KB      
    ├─────────────────────┤
    │     L2 Cache        │  12 cycles (~3 ns)      256 KB-1 MB    
    ├─────────────────────┤
    │     L3 Cache        │  40 cycles (~10 ns)     8-32 MB        
    ├─────────────────────┤
    │       RAM           │  100 cycles (~30 ns)    8-64 GB        
    ├─────────────────────┤
    │    SSD Storage      │  50,000 cycles (~15 μs) 256 GB-4 TB    
    ├─────────────────────┤
    │   HDD Storage       │  300,000 cycles (~8 ms) 1-10 TB        
    └─────────────────────┘

Key principle: Each level is ~10× slower but ~10× larger
```

### **Speed Comparison in Concrete Terms**

If accessing a register took **1 second**, then:

| Memory Level | Real Time | Equivalent Human Time Scale |
|--------------|-----------|------------------------------|
| **Register** | 0.3 ns | 1 second |
| **L1 Cache** | 1 ns | 3 seconds |
| **L2 Cache** | 3 ns | 10 seconds |
| **L3 Cache** | 10 ns | 33 seconds |
| **RAM** | 30 ns | 1.5 minutes |
| **SSD** | 15 μs | **14 hours** (overnight wait) |
| **HDD** | 8 ms | **9 months** (pregnancy duration) |

**Better understanding:**

- **RAM is like waiting minutes** for something that should be instant
- **SSD is like overnight shipping** - you wait until tomorrow
- **HDD is like international shipping** - you wait for months

**Key insight:** The speed differences are MASSIVE. Going from cache to RAM is like going from instant response to waiting several minutes. Going to disk is like going from instant response to waiting overnight or longer.

---

### **What is "Hot," "Warm," and "Cold" Data?**

This terminology describes **how frequently data is accessed** and **where it should ideally be stored**.

#### **Hot Data**
```
Definition: Data accessed VERY frequently (multiple times per frame)

Examples in games:
- Player position (read/written every frame)
- Camera transform (read every frame for rendering)
- Active enemy AI states (updated every frame)
- Current frame's rendering data

Should be:
- In CPU registers (ideal)
- In L1/L2 cache (realistic)
- NEVER on disk
- Minimize RAM accesses

C++ example:
for (int i = 0; i < 1000000; i++) {
    sum += array[i];  // 'sum' is HOT (used every iteration)
}                      // Compiler keeps it in a register
```

#### **Warm Data**
```
Definition: Data accessed occasionally (few times per frame, or every few frames)

Examples in games:
- Inactive enemy data (updated when visible)
- UI element positions (updated on screen changes)
- Weapon statistics (read when firing/switching)
- Sound effect metadata (accessed when playing sound)

Should be:
- In RAM (acceptable)
- L3 cache (if lucky)
- Pre-loaded from disk before needed

C++ example:
void FireWeapon() {
    // Weapon stats accessed occasionally
    int damage = currentWeapon.damage;  // WARM data
    float spread = currentWeapon.spread;
}
```

#### **Cold Data**
```
Definition: Data accessed rarely (once per level, or less)

Examples in games:
- Level geometry for unloaded levels
- Texture data for distant/unrendered objects
- Save game data (accessed on load/save only)
- Configuration files (read at startup)
- Audio files not currently playing

Should be:
- On disk until needed
- Loaded to RAM only when required
- Cached in RAM if re-use likely

C++ example:
void LoadLevel(const std::string& levelName) {
    // Level data is COLD until player enters
    LevelData data = LoadFromDisk(levelName);  // COLD data
}
```

**Strategy:**
```
GOAL: Keep hot data in cache, warm data in RAM, cold data on disk

Bad: Loading all texture data into RAM at startup
     (cold data taking up RAM space)

Good: Stream textures from disk as needed
      (keep only visible textures in RAM)

Best: Predictive loading of warm data
      (load next level data while playing current level)
```

---

### **Memory Access Patterns**

**Memory access patterns** refer to the ORDER and LOCATION in which your program reads/writes data from memory.

The pattern you use has a **MASSIVE** impact on performance due to how caches work.

#### **Spatial Locality**

**Definition:** If you access memory address X, you will likely access nearby addresses soon.

**Why it matters:** Caches load entire "cache lines" (64 bytes at once). If you access nearby addresses, you've already loaded them!

```cpp
// GOOD spatial locality
int array[1000];
int sum = 0;
for (int i = 0; i < 1000; i++) {
    sum += array[i];  // Accessing addresses in sequence
}

Memory: [array[0]][array[1]][array[2]]...[array[999]]
        └─────────64 bytes──────────┘ Loaded together!
                (16 ints)

Cache loads array[0-15] in one operation
Next 15 accesses hit cache!
```

```cpp
// BAD spatial locality  
struct Node {
    int value;
    Node* next;  // Could point ANYWHERE in memory
};

Node* current = head;
int sum = 0;
while (current != nullptr) {
    sum += current->value;  // Nodes scattered randomly
    current = current->next; // Jump to random location
}

Memory: [Node1 at 0x1000] ... [Node2 at 0x9F3C] ... [Node3 at 0x2A14]
        └────────random jumps─────────┘
        
Each access likely to miss cache (nodes not near each other)
```

**Impact:**
```
Array (good locality):  1 cache miss per 16 ints
Linked list (bad):      1 cache miss per node

Array: 1M ints = ~62,500 cache misses
List:  1M nodes = 1,000,000 cache misses

16× more cache misses = 16× slower!
```

---

#### **Temporal Locality**

**Definition:** If you access data now, you will likely access it again soon.

**Why it matters:** Data you recently accessed stays in cache, making subsequent accesses fast.

```cpp
// GOOD temporal locality
void UpdatePlayer(float deltaTime) {
    // player.position accessed multiple times
    player.position += player.velocity * deltaTime;  // Read + write
    
    if (player.position.y < 0) {  // Read again (still in cache!)
        player.position.y = 0;    // Write again (still in cache!)
    }
    
    renderer.DrawSprite(player.sprite, player.position); // Read again
}

player.position loaded to cache on first access
All subsequent accesses hit cache (fast!)
```

```cpp
// BAD temporal locality
void UpdateAllEnemies() {
    for (int i = 0; i < 10000; i++) {
        UpdateEnemy(enemies[i]);  // Touch each enemy once
    }
    
    // 100 lines of other code...
    
    for (int i = 0; i < 10000; i++) {
        RenderEnemy(enemies[i]);  // Touch each enemy again
    }
}

First loop: Load all enemies (evicts each other from cache)
Second loop: Cache misses again (enemies were evicted)

Better: Update AND render in same loop!
```

**Impact:**
```
Good temporal locality:  Data accessed multiple times while in cache
Bad temporal locality:   Data loaded, evicted, loaded again

Wasted cache loads = wasted time
```

---

### **Why Access Patterns Matter: Real Example**

```cpp
// Example: Processing 1 million 3D vectors

// BAD: Struct of Arrays of Structs (cache-hostile)
struct Vector3 {
    float x, y, z;
};

struct Enemy {
    Vector3 position;
    Vector3 velocity;
    int health;
    float attackRange;
    // ... 20 more fields (200 bytes total)
};

Enemy* enemies = new Enemy[1000000];

// Update positions
for (int i = 0; i < 1000000; i++) {
    enemies[i].position.x += enemies[i].velocity.x * dt;
    enemies[i].position.y += enemies[i].velocity.y * dt;
    enemies[i].position.z += enemies[i].velocity.z * dt;
}

Problem:
- Each Enemy is 200 bytes
- Cache line is 64 bytes
- Loads position + velocity + health + attackRange + garbage
- Only uses position and velocity (24 bytes)
- Wastes 176 bytes per cache line!

Cache efficiency: 24/200 = 12%
```

```cpp
// GOOD: Struct of Arrays (cache-friendly)
struct EnemyData {
    float* posX;  // All X positions together
    float* posY;  // All Y positions together
    float* posZ;  // All Z positions together
    float* velX;  // All X velocities together
    float* velY;  // All Y velocities together
    float* velZ;  // All Z velocities together
    // Other data in separate arrays
};

EnemyData enemies;
enemies.posX = new float[1000000];
enemies.posY = new float[1000000];
// ... allocate others

// Update positions
for (int i = 0; i < 1000000; i++) {
    enemies.posX[i] += enemies.velX[i] * dt;
    enemies.posY[i] += enemies.velY[i] * dt;
    enemies.posZ[i] += enemies.velZ[i] * dt;
}

Better:
- Cache line loads 16 floats at once (64 bytes / 4 bytes)
- ALL 16 are used (no waste!)
- No health, attackRange, or other fields loaded

Cache efficiency: 100%

Performance: 8-10× faster!
```

---

### **🎮 Game Dev Connection**

Let's trace what happens at the hardware level when you write game code:

```cpp
// Example: Simple game loop updating 1000 enemies

// Object-Oriented approach (what you might write first)
class Enemy {
public:
    Vector3 position;
    Vector3 velocity;
    int health;
    float attackRange;
    Texture* sprite;
    AudioClip* soundEffect;
    // ... 10 more fields
    
    void Update(float dt) {
        position += velocity * dt;  // What we actually need
    }
};

std::vector<Enemy> enemies(1000);

void GameLoop() {
    for (auto& enemy : enemies) {
        enemy.Update(deltaTime);
    }
}
```

**What happens in hardware:**

```
Frame 1, Enemy 0:
1. CPU: "Load enemy[0]" (issues load from RAM)
2. Cache: Miss! Fetch from RAM (100 cycles = 30ns)
3. Cache: Load 64-byte cache line starting at enemy[0]
   Contains: position(12B) + velocity(12B) + health(4B) + 
             attackRange(4B) + sprite*(8B) + soundEffect*(8B) + more...
4. CPU: Read position (12B) - Cache HIT
5. CPU: Read velocity (12B) - Cache HIT  
6. CPU: Compute position += velocity * dt
7. CPU: Write new position (12B) - Cache HIT

   Used: 24 bytes (position + velocity)
   Wasted: 40 bytes (health, attackRange, pointers we don't need)

Frame 1, Enemy 1:
1. CPU: "Load enemy[1]"
2. Cache: Miss again! (enemy[1] is 100+ bytes away)
3. Repeat process...

1000 enemies = ~1000 cache misses = ~30,000 ns = 30 microseconds
Just for cache misses alone!
```

**Now with Data-Oriented Design:**

```cpp
// Data-Oriented approach
struct EnemySystem {
    std::vector<Vector3> positions;   // Positions packed together
    std::vector<Vector3> velocities;  // Velocities packed together
    std::vector<int> healths;         // Healths packed together (not needed now)
    // ... other data separated
};

EnemySystem enemies;
enemies.positions.resize(1000);
enemies.velocities.resize(1000);

void GameLoop() {
    for (int i = 0; i < 1000; i++) {
        enemies.positions[i] += enemies.velocities[i] * deltaTime;
    }
}
```

**What happens in hardware:**

```
Frame 1, Enemies 0-4:
1. CPU: "Load positions[0]"
2. Cache: Miss! Fetch from RAM (100 cycles)
3. Cache: Load 64-byte cache line
   Contains: positions[0-4] (5 × 12 bytes = 60 bytes)
4. CPU: Read positions[0] - Cache HIT
5. CPU: Read velocities[0] - Different cache line, likely HIT if recently accessed
6. CPU: Compute and write position[0]

7. CPU: "Load positions[1]" - Cache HIT! (already loaded)
8. Repeat for positions[2-4] - ALL Cache HITS!

Frame 1, Enemies 5-9:
1. CPU: "Load positions[5]"
2. Cache: Miss! Load next cache line (positions[5-9])
3. Next 5 updates are cache hits...

1000 enemies = ~200 cache misses = ~6,000 ns = 6 microseconds

5× fewer cache misses = 5× faster!
```

**Example numbers from profiling:**

```
Object-Oriented layout:
1000 enemies updated: 50 microseconds
10,000 enemies: 500 microseconds (0.5ms)
At 60 FPS budget (16.6ms): Can afford ~30,000 enemies

Data-Oriented layout:
1000 enemies updated: 8 microseconds
10,000 enemies: 80 microseconds (0.08ms)
At 60 FPS budget: Can afford ~200,000 enemies!

6× more enemies in your game!
```

---

## 📚 Cache Architecture in Detail

### **What is a Cache Line?**

A **cache line** (also called **cache block**) is the minimum unit of data transfer between cache and RAM.

**Key properties:**
- **Fixed size:** Typically 64 bytes on modern CPUs
- **Atomic unit:** Cache loads entire line at once, not individual bytes
- **Alignment:** Lines start at addresses divisible by 64

```
Memory addresses aligned to cache lines:

Address 0x0000: [────── Cache Line 0 ──────] (bytes 0-63)
Address 0x0040: [────── Cache Line 1 ──────] (bytes 64-127)
Address 0x0080: [────── Cache Line 2 ──────] (bytes 128-191)
Address 0x00C0: [────── Cache Line 3 ──────] (bytes 192-255)

If you access byte at address 0x0055:
Cache loads entire line 0 (bytes 0-63)
```

**Why cache lines matter:**

```cpp
// Example: Array of integers
int array[16];  // 16 ints × 4 bytes = 64 bytes = 1 cache line!

int sum = array[0];  // CACHE MISS: Loads entire cache line (array[0-15])
sum += array[1];     // CACHE HIT (already loaded)
sum += array[2];     // CACHE HIT
// ... 
sum += array[15];    // CACHE HIT

1 cache miss, 15 cache hits!
```

**Bad alignment:**

```cpp
struct Data {
    char padding[60];
    int value;  // Starts at byte 60
};

Data d;
int x = d.value;  // Loads bytes 60-63

Problem: Cache line is bytes 0-63
         value spans TWO cache lines! (bytes 60-63 and 64-67 if struct is 64 bytes)
         
Could require loading 2 cache lines for 1 int!
```

**Good practice: Align to cache lines**

```cpp
// Ensure structures are multiples of 64 bytes or fit within one line
struct alignas(64) CacheFriendly {
    int values[16];  // Exactly 64 bytes = 1 cache line
};
```

---

### **Cache Organization: Sets, Ways, and Associativity**

Modern caches use **set-associative** organization to reduce cache thrashing.

#### **Direct-Mapped Cache (Simple, but has problems)**

```
Concept: Each memory address maps to exactly ONE cache line

Cache with 8 lines (numbered 0-7):

Memory address → Cache line mapping:
Address & 0x7 (last 3 bits) determines cache line

Address 0x00 → Line 0
Address 0x08 → Line 0 (same line!)
Address 0x10 → Line 0 (same line!)
Address 0x40 → Line 0 (same line!)

Address 0x01 → Line 1
Address 0x09 → Line 1 (same line!)

Problem: Many addresses map to same cache line
```

**Cache Thrashing Example:**

```cpp
int a[1000];  // Assume starts at address 0x1000
int b[1000];  // Assume starts at address 0x1000 + 0x1000 = 0x2000

// If both arrays map to same cache lines:
for (int i = 0; i < 1000; i++) {
    a[i] = b[i];  // Loads b[i] → evicts a[i]
                  // Writes a[i] → evicts b[i]
}

Constant eviction = cache thrashing!
Every access is a miss!
```

---

#### **Set-Associative Cache (Modern CPUs)**

**Concept:** Each memory address can go into N cache lines (N-way associative).

```
4-way set-associative cache (common):

Cache divided into SETS, each set has 4 WAYS (lines)

Example: 32 KB L1 cache, 64-byte lines, 4-way associative
- Total lines: 32768 / 64 = 512 lines
- Number of sets: 512 / 4 = 128 sets
- Each set: 4 ways (4 lines)

Memory address breakdown:
┌──────────┬──────────┬────────────┐
│   Tag    │  Index   │   Offset   │
│ (varies) │ (7 bits) │  (6 bits)  │
└──────────┴──────────┴────────────┘
     ↑          ↑           ↑
     │          │           └─ Which byte in cache line (0-63)
     │          └───────────── Which set (0-127)
     └──────────────────────── Is this the right data?

Address mapping:
1. Index bits determine SET (which of 128 sets)
2. Can go in ANY of the 4 ways in that set
3. Tag bits confirm if data is correct

Much less thrashing!
```

**Example:**

```
Address 0x1000:
- Offset: 0x00 (byte 0)
- Index:  0x40 (set 64)
- Tag:    0x01

Can be stored in set 64, any of 4 ways:
Set 64: [Way 0][Way 1][Way 2][Way 3]
         ↑ Could go here
         ↑ Or here
         ↑ Or here
         ↑ Or here

Much more flexible than direct-mapped!
```

---

#### **What is Associativity?**

**Associativity** is the number of different cache lines a memory address can be stored in.

```
1-way (direct-mapped):
Each address maps to exactly 1 line
└─ Simple, fast, but lots of conflicts

4-way set-associative:
Each address can go in 4 different lines
└─ More flexible, fewer conflicts

8-way set-associative:
Each address can go in 8 different lines
└─ Even more flexible, almost no conflicts

Fully associative:
Address can go in ANY cache line
└─ No conflicts, but slow and complex (used for TLB)
```

**Trade-offs:**

```
Higher associativity:
✓ Fewer conflict misses
✓ Better cache utilization
✗ More complex hardware (must check N tags simultaneously)
✗ Slightly slower lookup

Modern CPUs:
- L1: 8-way associative
- L2: 8-way or 16-way
- L3: 16-way or 20-way
```

---

### **Cache Replacement Policies**

When all ways in a set are full, which line gets evicted?

#### **LRU (Least Recently Used)**

```
Most common policy: Evict the line accessed longest ago

Set 5, 4-way associative:
Way 0: [Data A] Last accessed: 100 cycles ago
Way 1: [Data B] Last accessed: 50 cycles ago
Way 2: [Data C] Last accessed: 200 cycles ago  ← LRU (evict this)
Way 3: [Data D] Last accessed: 10 cycles ago

New data arrives → Evict Way 2 (least recently used)
```

**Implementation:**
```
Each cache line has a timestamp or counter
On access: Update timestamp
On eviction: Find oldest timestamp

Hardware maintains this automatically
```

---

#### **Pseudo-LRU (Practical Implementation)**

```
True LRU requires tracking access order for all ways
Expensive in hardware!

Pseudo-LRU: Approximate LRU with less hardware

Binary tree approach (4-way example):
          Root
         /    \
       Bit0   Bit1
       / \    / \
     W0  W1  W2  W3

Access W0: Set tree bits → next eviction will be W2 or W3
Access W2: Set tree bits → next eviction will be W0 or W1

Not perfect LRU, but close enough and much cheaper!
```

---

### **Why Does CPU Use More Than One Cache?**

Modern CPUs have **3 levels of cache** (L1, L2, L3). Here's why:

#### **Reason 1: Speed vs Size Trade-Off**

```
Faster cache = physically smaller
- Signals travel short distances (speed of light limit!)
- Fewer bits to check
- Simpler design

Larger cache = physically bigger
- Signals travel longer distances (slower)
- More bits to search through
- More complex design

Can't have both in one cache!

Solution: Multiple levels with different trade-offs
```

---

#### **Reason 2: Hit Rate Optimization**

```
L1 (small, 32-64 KB):
- Very fast (4 cycles)
- Limited capacity
- Hit rate: ~90%
- Catches the hottest data

L2 (medium, 256 KB-1 MB):
- Medium speed (12 cycles)
- More capacity
- Hit rate: ~95% of L1 misses
- Catches warm data

L3 (large, 8-32 MB):
- Slower (40 cycles)
- Large capacity
- Hit rate: ~90% of L2 misses
- Catches data that doesn't fit in L1/L2

Combined: ~99.9% hit rate!
```

**Without multiple levels:**

```
Single 8 MB L1 cache:
- Too large → 40+ cycle latency (no better than L3!)
- Wasted speed potential

Single 32 KB cache:
- Too small → constant evictions
- Hit rate: ~60%
- Most accesses go to RAM (100+ cycles)

Multiple levels give best of both worlds!
```

---

#### **Reason 3: Specialization**

```
L1 split into:
- L1 Instruction cache (I-cache): Stores program code
- L1 Data cache (D-cache): Stores program data

Why split?
- CPU fetches instruction and data simultaneously
- No contention between code and data
- Doubles bandwidth

L2 and L3: Unified (code + data together)
- By L2, don't need as much parallelism
- Simpler design
```

---

#### **Reason 4: Multi-Core Sharing**

```
Modern CPU (4 cores):

Each core has private:
- L1 cache (32 KB I + 32 KB D)
- L2 cache (256 KB)

All cores share:
- L3 cache (8 MB)

Why share L3?
- Core 0 computes data
- Core 1 needs same data
- Data already in L3 (no RAM access needed!)
- Enables efficient inter-core communication

Each core: 32+32+256 = 320 KB private, 8 MB shared
```

---

### **Cache Hits and Misses (Comprehensive)**

#### **What is a Cache Hit?**

A **cache hit** occurs when the CPU requests data and finds it in cache.

```
CPU: "I need data at address 0x1234"
     
Check L1 cache:
1. Extract index bits → Determine set
2. Check all ways in that set
3. Compare tags
4. Tag matches! → HIT
5. Return data from cache line

Total time: 4 cycles (1-2 nanoseconds)
```

**What happens:**
```
1. No RAM access needed (huge time saving!)
2. Data copied to CPU register
3. Cache line marked as "recently used" (for LRU)
4. CPU continues immediately
```

**Performance:**
```
L1 hit: 1 ns
L2 hit: 3 ns
L3 hit: 10 ns

All much better than RAM (30+ ns)
```

---

#### **What is a Cache Miss?**

A **cache miss** occurs when CPU requests data NOT in cache.

```
CPU: "I need data at address 0x1234"
     
Check L1 cache:
1. Extract index, check ways
2. No tag match → MISS!

Check L2 cache:
1. Extract index, check ways
2. No tag match → MISS!

Check L3 cache:
1. Extract index, check ways
2. No tag match → MISS!

Access RAM:
1. Send request to memory controller (10 cycles)
2. Wait for RAM (100+ cycles)
3. Load 64-byte cache line from RAM
4. Store in L3, L2, L1 (cache hierarchy)
5. Finally give data to CPU

Total time: ~100-300 cycles (30-100 nanoseconds)
```

**What happens:**
```
1. CPU stalls (waits for data)
2. Pipeline might flush
3. Other instructions might execute (out-of-order execution)
4. When data arrives, cache updated
5. CPU resumes with data

30-100× slower than cache hit!
```

---

#### **Types of Cache Misses**

**1. Compulsory Miss (Cold Miss)**

```
Definition: First access to data that has never been in cache

Example:
int* data = new int[1000];  // Allocate array
int sum = 0;
for (int i = 0; i < 1000; i++) {
    sum += data[i];  // First access to each element
}

First loop iteration:
- data[0-15]: Compulsory miss (load from RAM)
- Next 15 accesses: Hits (same cache line)

Cannot be avoided! Data must enter cache somehow.

Mitigation:
- Prefetching (CPU predicts access, loads early)
- Data layout (pack related data together)
```

---

**2. Capacity Miss**

```
Definition: Cache is full, data was evicted, but needed again

Example:
// L1 cache = 32 KB = 8192 ints
int* huge = new int[100000];  // 400 KB (12× larger than L1)

// First pass
for (int i = 0; i < 100000; i++) {
    huge[i] = i;  // Fills L1, evicts earlier data
}

// Second pass
int sum = 0;
for (int i = 0; i < 100000; i++) {
    sum += huge[i];  // Data no longer in L1!
}

Working set (400 KB) >> Cache size (32 KB)
Data constantly evicted and reloaded

Mitigation:
- Process data in blocks that fit in cache
- Use smaller data types
- Compress data
```

**Example: Blocking for cache**

```cpp
// BAD: Process entire array (capacity misses)
for (int i = 0; i < 100000; i++) {
    ProcessElement(huge[i]);
}

// GOOD: Process in cache-sized blocks
const int BLOCK_SIZE = 8000;  // Fits in L1
for (int start = 0; start < 100000; start += BLOCK_SIZE) {
    int end = std::min(start + BLOCK_SIZE, 100000);
    for (int i = start; i < end; i++) {
        ProcessElement(huge[i]);  // Data stays in cache!
    }
}
```

---

**3. Conflict Miss**

```
Definition: Multiple addresses map to same cache set, evict each other

Example (simplified for illustration):
Assume cache has only 2 sets, each set has 2 ways

int a[1000];  // Addresses 0x1000-0x1FA0
int b[1000];  // Addresses 0x2000-0x2FA0

If a[0] and b[0] map to same set:
  
Access a[0]: Miss, load to Set 0, Way 0
Access b[0]: Miss, load to Set 0, Way 1 (same set!)
Access a[0]: Miss! (Way 0 was evicted for b[0])
Access b[0]: Miss! (Way 1 was evicted for a[0])

"Ping-pong" eviction

Mitigation:
- Higher associativity (more ways per set)
- Padding/alignment to avoid address conflicts
- Rearrange data layout
```

**Real example:**

```cpp
// Conflict misses
struct Data {
    int values[16];  // 64 bytes
};

Data array[1000];

// If sizeof(Data) == 64 and cache line == 64:
// array[0], array[n], array[2n] might all map to same set!
// (if 'n' is related to cache size)

Access pattern:
for (int i = 0; i < 1000; i += 16) {
    ProcessData(array[i]);  // Might all conflict!
}

Fix: Add padding to avoid power-of-2 sizes
struct Data {
    int values[16];
    char padding[64];  // Total 128 bytes (avoid conflicts)
};
```

---

### **Cache Miss Impact on Performance**

```cpp
// Benchmark: Sum 1 million integers

// Version 1: Sequential (good cache behavior)
int sum1 = 0;
for (int i = 0; i < 1000000; i++) {
    sum1 += array[i];
}
Time: 2 ms
Cache misses: ~62,500 (1 per 16 ints)

// Version 2: Random access (terrible cache behavior)
int sum2 = 0;
for (int i = 0; i < 1000000; i++) {
    int index = randomIndices[i];
    sum2 += array[index];
}
Time: 50 ms (25× slower!)
Cache misses: ~1,000,000 (1 per access)

Cache misses are devastating!
```

---

## 📚 Array Traversal Orders

### **Row-Major vs Column-Major**

**What are Row-Major and Column-Major?**

These terms describe **how multi-dimensional arrays are laid out in memory**.

C++ uses **row-major** order (rows stored contiguously).

```cpp
// 2D array: 3 rows × 4 columns
int matrix[3][4] = {
    {1,  2,  3,  4},   // Row 0
    {5,  6,  7,  8},   // Row 1
    {9, 10, 11, 12}    // Row 2
};

Memory layout (row-major):
[1][2][3][4][5][6][7][8][9][10][11][12]
 └─ Row 0 ─┘ └─ Row 1 ─┘ └─ Row 2 ──┘

Sequential in memory by ROW
```

**Column-major** (used in Fortran, MATLAB):
```
Memory layout (column-major):
[1][5][9][2][6][10][3][7][11][4][8][12]
 └Col0─┘ └─Col1┘ └─Col2┘ └─Col3─┘

Sequential in memory by COLUMN
```

---

### **Why Order Matters**

```cpp
// ROW-MAJOR ORDER (C++ style, cache-friendly)
for (int row = 0; row < 1000; row++) {
    for (int col = 0; col < 1000; col++) {
        matrix[row][col] = 0;
    }
}

Memory access pattern:
[0][1][2][3]...[999][1000][1001][1002]...
 └──────Sequential────────┘

Cache line loads 16 ints at once (64 bytes / 4 bytes per int)
15 out of 16 accesses hit cache!

Time: 3 ms
Cache miss rate: 6.25%
```

```cpp
// COLUMN-MAJOR ORDER (cache-hostile in C++)
for (int col = 0; col < 1000; col++) {
    for (int row = 0; row < 1000; row++) {
        matrix[row][col] = 0;
    }
}

Memory access pattern:
[0][1000][2000][3000]...[999000][1][1001][2001]...
 └─────Large jumps────────┘

Each access jumps 1000 ints (4000 bytes)
Every access likely a cache miss!

Time: 250 ms (83× slower!)
Cache miss rate: ~100%
```

**Why such a massive difference?**

```
Row-major access (sequential):
Address: 0x1000, 0x1004, 0x1008, 0x100C...
Cache: Loads 0x1000-0x103F (16 ints)
       Next 15 accesses hit!

Column-major access (strided):
Address: 0x1000, 0x1FA0, 0x3F40, 0x5EE0...
         (jumps of 4000 bytes)
Cache: Loads 0x1000-0x103F
       Next access at 0x1FA0 (different cache line!)
       Every access misses!

Result: 16× more cache misses = 83× slower
```

---


### **Object-Oriented vs Data-Oriented: Cache Impact**

#### **Why is Object-Oriented Cache-Hostile?**

**Object-Oriented layout** scatters data the CPU needs:

```cpp
class Enemy {
public:
    Vector3 position;      // 12 bytes (what we need)
    Vector3 velocity;      // 12 bytes (what we need)
    int health;            // 4 bytes
    float attackRange;     // 4 bytes
    Texture* texture;      // 8 bytes (pointer)
    AudioClip* sound;      // 8 bytes (pointer)
    AIState* aiState;      // 8 bytes (pointer)
    Animation* anim;       // 8 bytes (pointer)
    // ... many more fields
    
    void Update(float dt) {
        position += velocity * dt;  // Only touches 24 bytes!
    }
};

std::vector<Enemy> enemies(1000);

// Update loop
for (auto& enemy : enemies) {
    enemy.Update(dt);
}
```

**Memory layout:**
```
[Enemy 0: pos|vel|hp|range|tex*|snd*|ai*|anim*|...]  (200 bytes)
[Enemy 1: pos|vel|hp|range|tex*|snd*|ai*|anim*|...]  (200 bytes)
[Enemy 2: pos|vel|hp|range|tex*|snd*|ai*|anim*|...]  (200 bytes)
...

When updating positions:
- Load Enemy 0: Cache loads 64 bytes (pos + vel + garbage)
- Load Enemy 1: Different address, different cache line
- Load Enemy 2: Different address, different cache line
...

Each enemy requires new cache line!
Wastes cache space on health, pointers, etc.
```

**Cache efficiency:**
```
Needed data: 24 bytes (position + velocity)
Loaded data: 200 bytes (entire struct)
Efficiency: 24/200 = 12%

88% of cache space wasted!
```

---

#### **Why is Data-Oriented Cache-Friendly?**

**Data-Oriented layout** packs similar data together:

```cpp
struct EnemySystem {
    std::vector<Vector3> positions;   // All positions
    std::vector<Vector3> velocities;  // All velocities
    std::vector<int> healths;         // All healths
    std::vector<float> attackRanges;  // All ranges
    // ... other data separated
};

EnemySystem enemies;
enemies.positions.resize(1000);
enemies.velocities.resize(1000);

// Update loop
for (size_t i = 0; i < 1000; i++) {
    enemies.positions[i] += enemies.velocities[i] * dt;
}
```

**Memory layout:**
```
Positions:  [pos0|pos1|pos2|pos3|pos4|...]  (sequential)
Velocities: [vel0|vel1|vel2|vel3|vel4|...]  (sequential)
Healths:    [hp0 |hp1 |hp2 |hp3 |hp4 |...]  (not accessed)

When updating positions:
Cache line 0: [pos0|pos1|pos2|pos3|pos4]  (5 Vector3s = 60 bytes)
- Update pos0: Cache miss, load line
- Update pos1: Cache hit!
- Update pos2: Cache hit!
- Update pos3: Cache hit!
- Update pos4: Cache hit!

Cache line 1: [pos5|pos6|pos7|pos8|pos9]
- Update pos5: Cache miss, load line
- Updates 6-9: All cache hits!

1000 enemies = ~200 cache misses (instead of 1000)
```

**Cache efficiency:**
```
Needed data: 24 bytes (position + velocity)
Loaded data: 24 bytes (only positions/velocities in cache)
Efficiency: 24/24 = 100%

No wasted cache space!
```

**Performance difference:**
```
Object-Oriented: 45 microseconds (1000 cache misses)
Data-Oriented:   8 microseconds (200 cache misses)

5.6× faster!

For 10,000 enemies:
Object-Oriented: 450 microseconds
Data-Oriented:   80 microseconds

For 100,000 enemies:
Object-Oriented: 4.5 ms
Data-Oriented:   0.8 ms

DOD scales much better!
```

---

### **What is Pointer Chasing?**

**Pointer chasing** occurs when you follow pointers to access data, and each pointer points to a random location in memory.

```cpp
// Classic example: Linked List
struct Node {
    int data;
    Node* next;  // Could point ANYWHERE in memory
};

Node* head = BuildLinkedList(1000);

// Traverse list
Node* current = head;
int sum = 0;
while (current != nullptr) {
    sum += current->data;      // Access current node
    current = current->next;   // Jump to random location!
}
```

**Why is it called "pointer chasing"?**

```
CPU must "chase" pointers to find next data:

Step 1: Load head node from address 0x1000
Step 2: Read next pointer → 0x5A3C
Step 3: Load node from address 0x5A3C (CACHE MISS!)
Step 4: Read next pointer → 0x2F18
Step 5: Load node from address 0x2F18 (CACHE MISS!)
...

Each node in random location = every access likely a miss

Cannot prefetch! CPU doesn't know where next node is
until it reads current node's pointer!
```

---

**Memory layout:**
```
Array (cache-friendly):
[Node0][Node1][Node2][Node3][Node4]...
Address: 0x1000  0x1010  0x1020  0x1030  0x1040

Linked List (cache-hostile):
[Node0 @ 0x1000] → [Node1 @ 0x7F3C] → [Node2 @ 0x2A18]
     ↓                   ↓                   ↓
  next=0x7F3C         next=0x2A18         next=0x9B44

Nodes scattered randomly in memory!
```

**Performance impact:**
```cpp
// Benchmark: Sum 1 million integers

// Array (sequential)
int* array = new int[1000000];
int sum = 0;
for (int i = 0; i < 1000000; i++) {
    sum += array[i];
}
Time: 2 ms
Cache misses: ~62,500

// Linked List (pointer chasing)
Node* head = BuildList(1000000);
int sum = 0;
Node* current = head;
while (current) {
    sum += current->data;
    current = current->next;
}
Time: 100 ms (50× slower!)
Cache misses: ~1,000,000

Every access is a cache miss!
```

---

**Why can't CPU prefetch linked lists?**

```
Array:
- CPU sees: array[0], array[1], array[2]
- Predicts: array[3] will be accessed next
- Prefetches array[3-18] into cache
- When you access array[3], it's already there!

Linked List:
- CPU sees: node @ 0x1000
- Reads next pointer: 0x7F3C
- Cannot predict! Could be anywhere!
- Must wait for load to complete
- Then load next node (another wait)
- No prefetching possible

Prefetching requires predictable patterns
Pointer chasing is inherently unpredictable
```

---

**Solutions:**

```cpp
// Solution 1: Use arrays instead of linked structures
std::vector<int> data;  // Fast!

// Solution 2: If you must use pointers, allocate contiguously
Node* nodes = new Node[1000];
for (int i = 0; i < 999; i++) {
    nodes[i].next = &nodes[i + 1];  // Points to next in array
}
// Still a linked list, but cache-friendly!

// Solution 3: Hybrid approach (array of small chunks)
struct Chunk {
    int data[64];  // 64 ints in cache-friendly array
    Chunk* next;   // Pointer to next chunk
};
// Reduces pointer chasing by 64×
```

---

### **Why Use Padding?**

**Padding** adds extra bytes to structures to improve cache behavior, particularly to avoid **false sharing**.

#### **What is False Sharing?**

```
False sharing occurs when two CPU cores access different variables
that happen to be in the same cache line.

Example:
Core 0 writes to variable A
Core 1 writes to variable B

If A and B are in same cache line:
1. Core 0 writes A → Cache line marked "modified" on Core 0
2. Core 0's cache line invalidated on Core 1
3. Core 1 must reload cache line from Core 0
4. Core 1 writes B → Cache line marked "modified" on Core 1
5. Core 1's cache line invalidated on Core 0
6. Core 0 must reload cache line from Core 1
7. Repeat...

"Ping-pong" invalidation!
Even though A and B are independent!
```

**Concrete example:**

```cpp
// BAD: False sharing
struct Counter {
    int thread0Count;  // Bytes 0-3
    int thread1Count;  // Bytes 4-7 (SAME cache line!)
};

Counter counter;

// Thread 0
void Thread0() {
    for (int i = 0; i < 1000000; i++) {
        counter.thread0Count++;  // Writes to cache line
    }
}

// Thread 1
void Thread1() {
    for (int i = 0; i < 1000000; i++) {
        counter.thread1Count++;  // Writes to SAME cache line!
    }
}

Both threads: 8-12 seconds
Why so slow? Constant cache line invalidation between cores!
```

**With padding:**

```cpp
// GOOD: No false sharing
struct Counter {
    alignas(64) int thread0Count;  // Bytes 0-3, rest padded
    char padding0[60];             // Force to next cache line
    
    alignas(64) int thread1Count;  // Bytes 64-67, different cache line!
    char padding1[60];
};

Counter counter;

// Thread 0 and Thread 1 same as before

Both threads: 0.5 seconds (16-24× faster!)
Why? No cache line sharing = no invalidation!
```

---

**Why increase object size?**

```
Without padding:
- Counter: 8 bytes
- Compact in memory (good for space)
- False sharing kills performance

With padding:
- Counter: 128 bytes (16× larger!)
- Wastes memory
- Perfect performance (each core has own cache lines)

Trade-off:
Use 16× more memory to get 20× better performance

When to use padding:
✓ Multi-threaded data accessed frequently
✓ Performance-critical code
✗ Single-threaded code (no benefit)
✗ Memory-constrained (mobile, embedded)
```

**Rule of thumb:**
```
If multiple threads write to the same struct frequently,
pad to separate cache lines (64 bytes apart)

Memory is cheap, performance is precious!
```

---

## 📚 Virtual Memory & Memory Management

### **Virtual vs Physical Memory**

**Physical Memory:** The actual RAM chips in your computer.

**Virtual Memory:** An abstraction that gives each program the illusion of having its own private, contiguous memory space.

---

#### **What is Virtual Memory?**

```
Without virtual memory:
- Program A runs, uses addresses 0x1000-0x5000
- Program B runs, must avoid A's addresses, uses 0x6000-0xA000
- Programs must know about each other!
- Moving programs in memory breaks them (hard-coded addresses)

With virtual memory:
- Program A thinks it uses 0x0000-0x4000 ("virtual" addresses)
- Program B thinks it uses 0x0000-0x4000 (same virtual addresses!)
- OS maps virtual → physical transparently
- Programs don't know about each other
- Programs can be anywhere in physical RAM
```

**Every address you use in C++ is a virtual address!**

```cpp
int* ptr = new int;  // ptr contains a VIRTUAL address
                     // OS translates to physical address

std::cout << ptr;    // Prints virtual address (e.g., 0x00007FFF)
                     // Actual RAM might be at 0x8A32F000
```

---

#### **Why Use Virtual Memory?**

**1. Protection**
```
Program A cannot access Program B's memory:

Program A tries: ptr = (int*)0x12345;
                 *ptr = 666;

OS: "Virtual address 0x12345 doesn't belong to Program A"
    → Segmentation Fault (crash Program A, protect others)
```

**2. Simplification**
```
Every program starts at address 0x0000:
- No need to relocate code
- Linking is simpler
- Programs don't need to know where they're loaded

Linker can use fixed addresses, OS handles translation
```

**3. Memory Overcommit**
```
Physical RAM: 16 GB

Program A thinks it has: 4 GB
Program B thinks it has: 4 GB
Program C thinks it has: 4 GB
Total: 12 GB (fits in 16 GB)

But programs don't use all memory simultaneously!
OS can allocate more virtual memory than physical RAM

Unused virtual memory doesn't consume physical RAM
```

**4. Paging to Disk**
```
If RAM is full:
- OS moves unused pages to disk (swap space)
- Frees physical RAM for active programs
- If accessed again, load back from disk

Allows running more programs than fit in RAM
(at cost of speed when swapping occurs)
```

---

### **What is a Page Table?**

A **page table** is a data structure that maps virtual addresses to physical addresses.

**How it works:**

```
Virtual Memory divided into PAGES (typically 4 KB each)
Physical Memory divided into FRAMES (also 4 KB)

Page Table: Maps virtual pages → physical frames

Example:
Virtual Page 0 → Physical Frame 5
Virtual Page 1 → Physical Frame 12
Virtual Page 2 → Physical Frame 3
Virtual Page 3 → Physical Frame 18
...
```

**Address Translation:**

```
Virtual Address: 0x1234 (decimal 4660)
┌──────────────────┬──────────────┐
│   Page Number    │    Offset    │
│   (bits 12-31)   │  (bits 0-11) │
└──────────────────┴──────────────┘
        0x1                0x234
     (page 1)            (byte 564)

Step 1: Extract page number: 0x1 (page 1)
Step 2: Look up page table: Page 1 → Frame 12
Step 3: Extract offset: 0x234 (byte 564 within page)
Step 4: Physical address = Frame 12 + offset 0x234
        = 0xC000 + 0x234 = 0xC234

CPU: Access virtual 0x1234
OS:  Translate to physical 0xC234
RAM: Return data from physical 0xC234
```

---

### **What is a TLB?**

**TLB:** Translation Lookaside Buffer

A **TLB** is a small, fast cache that stores recent virtual → physical address translations.

**Why needed:**

```
Without TLB:
Every memory access requires page table lookup:
1. Access virtual address
2. Lookup page table (in RAM!)
3. Get physical address
4. Access physical address (in RAM!)

Result: 2 RAM accesses per memory access!
Doubles memory access time!
```

**With TLB:**

```
TLB caches recent translations:

Access virtual 0x1234:
1. Check TLB: Is translation for page 1 cached?
   → YES! Page 1 → Frame 12 (instant!)
2. Physical address = Frame 12 + offset
3. Access physical address

Result: 1 RAM access (translation was free!)

TLB hit: ~1 cycle (essentially free)
TLB miss: Must walk page table (~100 cycles)
```

---

**TLB characteristics:**

```
Size: 64-512 entries (small!)
Associativity: Fully associative (any entry can store any page)
Speed: 1 cycle lookup
Hit rate: 95-99% (very high!)

Why such a small cache works:
- Programs have locality (access nearby pages repeatedly)
- 512 entries × 4 KB = 2 MB coverage (enough for most hot data)
```

**Example:**

```cpp
int array[1000000];  // 4 MB array
for (int i = 0; i < 1000000; i++) {
    sum += array[i];
}

Array spans ~1000 pages (4 MB / 4 KB)
TLB can only cache ~512 pages

First 512 pages: TLB hits (fast)
Pages 513-1000: TLB misses (slower)
Loop again: Most accesses hit TLB (cached from first loop)

Overall TLB hit rate: ~95%
```

---

### **What is a Page Fault?**

A **page fault** occurs when a program accesses a virtual page that is not currently in physical RAM.

**Types:**

**1. Minor Page Fault (Soft Fault)**
```
Page is in RAM but not mapped in page table

Example:
- Allocated memory but never accessed
- OS allocated virtual page but not physical frame
- First access triggers page fault
- OS: "Ah, I need to actually allocate this"
- OS maps virtual page to physical frame
- Program continues

Cost: ~1000 cycles (slow but not terrible)
```

**2. Major Page Fault (Hard Fault)**
```
Page is not in RAM at all (swapped to disk)

Example:
- Program accessed page earlier
- OS needed RAM for something else
- OS wrote page to disk (swap file)
- Program accesses page again
- OS: "This page is on disk!"
- OS reads page from disk (millions of cycles!)
- OS loads into RAM
- OS updates page table
- Program continues

Cost: ~10,000,000 cycles (devastating!)
```

**What happens:**

```
Step-by-step (major page fault):

1. CPU accesses virtual address 0x5234
2. TLB miss, walk page table
3. Page table: "Page 5 not in RAM (on disk)"
4. CPU triggers page fault exception
5. OS interrupt handler runs:
   a. Find free physical frame (evict if needed)
   b. Read page from disk to RAM (~8 ms!)
   c. Update page table (Page 5 → new frame)
   d. Update TLB
   e. Resume program
6. CPU retries memory access
7. Success! (now in RAM)

Total: ~30,000,000 cycles = ~8 milliseconds

One page fault can cost a whole frame at 60 FPS!
```

---

**Impact on games:**

```cpp
// Terrible: Allocate huge array, access randomly
int* huge = new int[100000000];  // 400 MB

for (int i = 0; i < 1000; i++) {
    int randomIndex = rand() % 100000000;
    huge[randomIndex] = i;  // Random page faults!
}

// If only 50% of pages fit in RAM:
// ~50,000 page faults × 8 ms each = 400 seconds!

// Good: Access sequentially (predictable paging)
for (int i = 0; i < 100000000; i++) {
    huge[i] = i;  // Sequential, OS can prefetch
}

// Most pages already loaded when needed
// Minimal page faults
```

---

### **Memory Management Strategies in C++**

#### **Why Different Strategies Matter**

```
Stack allocation:
✓ FAST: Automatic, just move stack pointer
✓ SAFE: Automatic cleanup
✗ LIMITED: Fixed size, function scope only

Heap allocation:
✓ FLEXIBLE: Any size, any lifetime
✗ SLOW: Must search for free block
✗ FRAGMENTATION: Memory gets "holey"

Memory pool:
✓ FAST: Pre-allocated, no search
✓ NO FRAGMENTATION: Fixed-size blocks
✗ MANUAL: Must manage yourself
✗ WASTED: Unused slots waste space
```

---

#### **Stack Allocation**

```cpp
void GameUpdate() {
    int health = 100;           // Stack allocation
    float position[3];          // Stack allocation
    Enemy enemy;                // Stack allocation (if small)
    
    // At function end, ALL automatically freed
}

How it works:
Stack pointer: 0x7FFF0000 (top of stack)

1. int health:
   Stack pointer -= 4 (allocate 4 bytes)
   Store 100 at 0x7FFFEFFC

2. float position[3]:
   Stack pointer -= 12 (allocate 12 bytes)
   
3. Enemy enemy:
   Stack pointer -= sizeof(Enemy)
   Call Enemy constructor

Function ends:
   Stack pointer = 0x7FFF0000 (restore)
   Everything freed! (instant!)

Cost: 1 cycle (decrement stack pointer)
```

**When to use:**
```
✓ Small objects (<1 KB)
✓ Short lifetime (function scope)
✓ Fixed size known at compile time
```

---

#### **Heap Allocation**

```cpp
void CreateEnemies() {
    Enemy* enemy = new Enemy;           // Heap allocation
    int* largeArray = new int[1000000]; // Heap allocation
    
    // Must manually free!
    delete enemy;
    delete[] largeArray;
}

How it works:
1. new Enemy:
   a. Search free list for suitable block (~50-100 cycles)
   b. Split block if too large
   c. Mark as allocated
   d. Call constructor
   e. Return pointer

2. delete enemy:
   a. Call destructor
   b. Mark block as free
   c. Try to merge with adjacent free blocks (coalescing)

Cost: 50-500 cycles (depends on fragmentation)
```

**When to use:**
```
✓ Large objects (>1 KB)
✓ Long lifetime (beyond function scope)
✓ Dynamic size (unknown at compile time)
✓ Polymorphism (virtual functions require heap usually)
```

**Fragmentation problem:**

```
Heap after some allocations/deallocations:
[Used 1KB][Free 2KB][Used 500B][Free 1KB][Used 3KB]

Request 2.5 KB:
- Free blocks: 2 KB + 1 KB = 3 KB total
- But not contiguous!
- Must allocate new block or fail

This is fragmentation: Wasted free space
```

---

#### **Memory Pools**

```cpp
// Pre-allocate pool of Enemy objects
class EnemyPool {
    static const int POOL_SIZE = 1000;
    Enemy enemies[POOL_SIZE];     // Pre-allocated!
    bool active[POOL_SIZE];       // Which are in use?
    
public:
    EnemyPool() {
        for (int i = 0; i < POOL_SIZE; i++) {
            active[i] = false;
        }
    }
    
    Enemy* Allocate() {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (!active[i]) {
                active[i] = true;
                return &enemies[i];  // Fast! Just return pointer
            }
        }
        return nullptr;  // Pool full
    }
    
    void Free(Enemy* enemy) {
        int index = enemy - enemies;  // Calculate index
        active[index] = false;         // Fast! Just clear flag
    }
};

EnemyPool pool;

// Spawn enemy (fast!)
Enemy* enemy = pool.Allocate();  // ~10 cycles

// Remove enemy (fast!)
pool.Free(enemy);  // ~5 cycles

// Compare to heap:
Enemy* enemy = new Enemy;  // ~100 cycles
delete enemy;              // ~50 cycles

10-20× faster!
```

**When to use:**
```
✓ Many small objects of SAME TYPE
✓ Frequent allocation/deallocation
✓ Known maximum count
✓ Performance critical (bullets, particles, enemies)

Examples in games:
- Bullet pool (max 1000 bullets)
- Particle pool (max 10000 particles)
- Enemy pool (max 500 enemies)
- Audio source pool (max 32 channels)
```

**Comparison Table:**

```
┌──────────────┬──────────┬─────────────┬────────────┬──────────────┐
│   Strategy   │  Speed   │Fragmentation│  Safety    │  Use Case    │
├──────────────┼──────────┼─────────────┼────────────┼──────────────┤
│   Stack      │ 1 cycle  │   None      │ Automatic  │ Small/temp   │
│              │ (fastest)│             │ (RAII)     │              │
├──────────────┼──────────┼─────────────┼────────────┼──────────────┤
│   Heap       │50-500    │   High      │  Manual    │ Large/dynamic│
│   (new)      │ cycles   │(over time)  │ (error-    │ lifetime     │
│              │          │             │  prone)    │              │
├──────────────┼──────────┼─────────────┼────────────┼──────────────┤
│ Memory Pool  │5-20      │   None      │  Manual    │ Many same-   │
│              │ cycles   │  (fixed     │ (careful)  │ sized objects│
│              │          │  blocks)    │            │              │
└──────────────┴──────────┴─────────────┴────────────┴──────────────┘
```

---

# Profiling & Data-Oriented Design

## 📚 Profiling C++ Game Code

### **Why Profile?**

**"Premature optimization is the root of all evil"** - Donald Knuth

**Key principle:** Measure before optimizing!

```
What you THINK is slow: Physics calculations
What is ACTUALLY slow: String concatenation in debug log

Optimize physics: 10 hours work, 5% faster
Optimize logging: 10 minutes work, 300% faster!

ALWAYS profile first!
```

---

### **Profiling Tools for C++**

#### **1. gprof (GNU Profiler)**

**What it does:** Statistical sampling profiler, shows which functions take the most time.

**How to use:**

```bash
# Step 1: Compile with profiling enabled
g++ -pg -O2 game.cpp -o game

# Step 2: Run your program (generates gmon.out)
./game

# Step 3: Analyze results
gprof game gmon.out > analysis.txt
```

**Output example:**
```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 68.42      2.45     2.45  1000000     0.00     0.00  UpdateEnemies()
 18.72      3.12     0.67   500000     0.00     0.00  RenderSprite()
  7.23      3.38     0.26   100000     0.00     0.00  PhysicsStep()
  3.91      3.52     0.14        1   140.00   140.00  LoadLevel()
```

**Interpretation:**
- 68% of time spent in `UpdateEnemies()` ← **Optimize this first!**
- Called 1 million times (hot function)
- Only 0.00245 ms per call (not individually slow, but called often)

---

#### **2. perf (Linux Performance Tools)**

**What it does:** Hardware performance counter profiler, shows cache misses, branch mispredictions, etc.

**How to use:**

```bash
# Record performance data
perf record -g ./game

# View report
perf report

# Specific metrics
perf stat -e cache-misses,cache-references ./game
```

**Output example:**
```
Performance counter stats for './game':

    45,328,912      cache-references                                            
    12,437,821      cache-misses              #   27.44% of all cache refs      
     8,234,567      branch-misses             #    4.32% of all branches        

       2.456789 seconds time elapsed
```

**Interpretation:**
- 27% cache miss rate (high! should be <5%)
- Indicates poor memory access patterns
- Focus on improving data layout

---

#### **3. Valgrind (Cachegrind)**

**What it does:** Simulates CPU cache, shows exact cache misses per line of code.

**How to use:**

```bash
# Run with cachegrind
valgrind --tool=cachegrind ./game

# Analyze with annotate tool
cg_annotate cachegrind.out.<pid>
```

**Output example:**
```
Ir   I1mr  ILmr  Dr    D1mr  DLmr  Dw    D1mw  DLmw  File:Function
--------------------------------------------------------------------------------
287M 1,203    89 145M  8.2M   1.3M 72M   421K   89K  game.cpp:UpdateEnemies

Ir: Instruction reads
Dr: Data reads  
Dw: Data writes
D1mr: L1 data cache read misses (HIGH = problem!)
DLmr: Last-level cache read misses
```

**Interpretation:**
- 8.2M L1 cache misses in one function!
- That's one miss every ~17 data reads (terrible!)
- Clear indication of cache-hostile code

---

#### **4. Tracy Profiler (Modern C++ Game Profiler)**

**What it does:** Real-time frame profiler with visualization, designed for games.

**Installation:**
```bash
# Clone Tracy
git clone https://github.com/wolfpld/tracy.git

# Build profiler UI (Windows/Linux/Mac)
cd tracy/profiler/build/unix
make
```

**Integration:**
```cpp
// Add to your code
#include "tracy/Tracy.hpp"

void UpdateEnemies() {
    ZoneScoped;  // Automatic profiling scope
    
    for (auto& enemy : enemies) {
        enemy.Update(deltaTime);
    }
}

void RenderFrame() {
    ZoneScoped;
    // ... rendering code
}

// In main loop
int main() {
    while (running) {
        FrameMark;  // Mark frame boundary
        
        UpdateGame();
        RenderFrame();
    }
}
```

**What you get:**
- Real-time flame graph showing function call hierarchy
- Frame time breakdown
- Memory allocations tracking
- Compare multiple frames
- Visual timeline

**Advantages:**
- Zero impact when profiling disabled (macros compile to nothing)
- Frame-by-frame analysis
- Easy to see frame spikes
- Cross-platform

---

### **Profiling Raylib Game Example**

```cpp
// game.cpp
#include "raylib.h"
#include <vector>

struct Enemy {
    Vector2 position;
    Vector2 velocity;
    int health;
    float size;
};

std::vector<Enemy> enemies;

void UpdateEnemies(float dt) {
    for (auto& enemy : enemies) {
        enemy.position.x += enemy.velocity.x * dt;
        enemy.position.y += enemy.velocity.y * dt;
        
        if (enemy.position.x < 0 || enemy.position.x > 800) {
            enemy.velocity.x *= -1;
        }
        if (enemy.position.y < 0 || enemy.position.y > 600) {
            enemy.velocity.y *= -1;
        }
    }
}

void RenderEnemies() {
    for (const auto& enemy : enemies) {
        DrawCircleV(enemy.position, enemy.size, RED);
    }
}

int main() {
    InitWindow(800, 600, "Profiling Example");
    SetTargetFPS(60);
    
    // Spawn many enemies
    for (int i = 0; i < 10000; i++) {
        enemies.push_back({
            {(float)GetRandomValue(0, 800), (float)GetRandomValue(0, 600)},
            {(float)GetRandomValue(-100, 100), (float)GetRandomValue(-100, 100)},
            100,
            5.0f
        });
    }
    
    while (!WindowShouldClose()) {
        float dt = GetFrameTime();
        
        UpdateEnemies(dt);
        
        BeginDrawing();
        ClearBackground(RAYWHITE);
        RenderEnemies();
        DrawFPS(10, 10);
        EndDrawing();
    }
    
    CloseWindow();
    return 0;
}
```

**Compile and profile:**

```bash
# Compile with profiling
g++ -pg -O2 game.cpp -o game -lraylib -lm -lpthread -ldl -lrt -lX11

# Run (let it run for 10-20 seconds, then close)
./game

# Analyze
gprof game gmon.out > profile.txt
cat profile.txt
```

**Expected output:**
```
  %   cumulative   self              
 time   seconds   seconds    calls  name    
 42.31      0.89     0.89              UpdateEnemies()
 28.57      1.49     0.60              RenderEnemies()
 15.24      1.81     0.32              DrawCircleV()
  8.10      1.98     0.17              GetFrameTime()
```

**Interpretation:**
- `UpdateEnemies()` takes 42% of CPU time
- Processing 10,000 enemies every frame
- Optimization target identified!

---

### **Finding Cache Misses with perf**

```bash
# Compile optimized build
g++ -O2 game.cpp -o game -lraylib -lm -lpthread -ldl -lrt -lX11

# Profile cache behavior
perf stat -e cache-misses,cache-references,instructions,cycles ./game
```

**Output:**
```
Performance counter stats for './game':

   234,567,890      cache-misses              #   18.5% miss rate
 1,267,890,123      cache-references                                  
 8,456,789,012      instructions              #    2.13  insn per cycle
 3,967,543,210      cycles                                            

      15.234567 seconds time elapsed
```

**Analysis:**
- 18.5% cache miss rate (very high!)
- Should be <5% for good performance
- Indicates object-oriented layout issues
- Refactor to Data-Oriented Design!

---

## 📚 Data-Oriented Design in C++

### **DOD Principles**

**Data-Oriented Design (DOD):** Design code around data layout and access patterns, not object hierarchies.

**Key principles:**

1. **Separate hot and cold data**
2. **Pack data by usage, not by entity**
3. **Prefer arrays over pointers**
4. **Process data in bulk**
5. **Think about cache lines**

---

### **Pattern 1: Struct of Arrays (SoA)**

**Traditional OOP (Array of Structs - AoS):**

```cpp
// BAD: Cache-hostile
struct Enemy {
    Vector3 position;    // Hot (accessed every frame)
    Vector3 velocity;    // Hot (accessed every frame)
    int health;          // Warm (accessed when hit)
    int maxHealth;       // Cold (rarely accessed)
    float attackRange;   // Warm (accessed when attacking)
    float attackCooldown;// Warm
    Texture2D* sprite;   // Cold (pointer to texture data elsewhere)
    // ... 10 more fields (200 bytes total)
};

std::vector<Enemy> enemies(10000);

void UpdateMovement(float dt) {
    for (auto& enemy : enemies) {
        enemy.position.x += enemy.velocity.x * dt;
        enemy.position.y += enemy.velocity.y * dt;
        enemy.position.z += enemy.velocity.z * dt;
    }
}

// Cache efficiency: 12%
```

**DOD (Struct of Arrays - SoA):**

```cpp
// GOOD: Cache-friendly
struct EnemySystem {
    // Hot data together
    std::vector<Vector3> positions;
    std::vector<Vector3> velocities;
    
    // Warm data together
    std::vector<int> healths;
    std::vector<float> attackRanges;
    std::vector<float> attackCooldowns;
    
    // Cold data together
    std::vector<int> maxHealths;
    std::vector<Texture2D*> sprites;
};

EnemySystem enemies;
enemies.positions.resize(10000);
enemies.velocities.resize(10000);

void UpdateMovement(float dt) {
    for (size_t i = 0; i < enemies.positions.size(); i++) {
        enemies.positions[i].x += enemies.velocities[i].x * dt;
        enemies.positions[i].y += enemies.velocities[i].y * dt;
        enemies.positions[i].z += enemies.velocities[i].z * dt;
    }
}

// Cache efficiency: 100%
// Result: 5-10× faster!
```

---

### **Pattern 2: Hot/Cold Splitting**

```cpp
// Hot data (accessed every frame)
struct ParticleHot {
    Vector3 position;
    Vector3 velocity;
    float lifetime;
    bool active;
    // Total: 28 bytes
};

// Cold data (rarely accessed)
struct ParticleCold {
    float initialLifetime;
    Texture2D* texture;
    int textureFrame;
    Color color;
};

struct ParticleSystem {
    std::vector<ParticleHot> hotData;
    std::vector<ParticleCold> coldData;
};

void UpdateParticles(float dt) {
    for (auto& p : particles.hotData) {
        if (p.active) {
            p.position += p.velocity * dt;
            p.lifetime -= dt;
            if (p.lifetime <= 0) p.active = false;
        }
    }
}

// Result: 3-5× faster updates!
```

---

### **Pattern 3: Index-Based References**

```cpp
// GOOD: Index-based references
struct Transform {
    Vector3 position;
    int32_t parentIndex;  // Index into same array (-1 = no parent)
};

std::vector<Transform> transforms(1000);

void UpdateTransform(int index) {
    Transform& t = transforms[index];
    
    if (t.parentIndex >= 0) {
        Transform& parent = transforms[t.parentIndex];
        // Parent likely in nearby cache line!
        t.position += parent.position;
    }
}

// Benefits:
// - Indices smaller than pointers (4 vs 8 bytes)
// - Array can be reallocated without invalidating references
// - Better cache locality
// - Easier to serialize
```

---

### **Pattern 4: Chunked Arrays**

```cpp
const int CHUNK_SIZE = 64;

struct EntityChunk {
    Vector3 positions[CHUNK_SIZE];
    Vector3 velocities[CHUNK_SIZE];
    int healths[CHUNK_SIZE];
    uint64_t activeMask;  // Bitfield: which slots active
    int count;
};

std::vector<EntityChunk*> chunks;

void UpdateAllEntities(float dt) {
    for (EntityChunk* chunk : chunks) {
        for (int i = 0; i < chunk->count; i++) {
            if (chunk->activeMask & (1ULL << i)) {
                chunk->positions[i] += chunk->velocities[i] * dt;
            }
        }
    }
}

// Benefits:
// - Better cache usage than full SoA
// - More flexible than object pool
// - Deletion is O(1)
// - Good balance for most games
```

---

### **Pattern 5: Sparse Sets**

```cpp
struct SparseSet {
    std::vector<int> dense;    // Packed array of active entity IDs
    std::vector<int> sparse;   // Maps entity ID → index in dense
    
    void Add(int entityID) {
        if (entityID >= sparse.size()) {
            sparse.resize(entityID + 1, -1);
        }
        sparse[entityID] = dense.size();
        dense.push_back(entityID);
    }
    
    void Remove(int entityID) {
        int denseIndex = sparse[entityID];
        int lastEntity = dense.back();
        
        dense[denseIndex] = lastEntity;
        sparse[lastEntity] = denseIndex;
        
        dense.pop_back();
        sparse[entityID] = -1;
    }
    
    void ForEach(std::function<void(int)> callback) {
        for (int entityID : dense) {
            callback(entityID);
        }
    }
};

// Benefits:
// - O(1) add, remove, contains
// - Dense array always packed (no gaps, perfect for cache)
// - No branching in hot loop
// - Foundation of modern ECS architectures
```

---

## 📚 Cache-Friendly C++ Patterns

### **Loop Optimization**

#### **1. Loop Fusion (Combine Loops)**

```cpp
// BAD: Multiple passes
void Update() {
    for (int i = 0; i < count; i++) {
        positions[i] += velocities[i] * dt;
    }
    for (int i = 0; i < count; i++) {
        if (positions[i].y < 0) positions[i].y = 0;
    }
    for (int i = 0; i < count; i++) {
        Draw(positions[i]);
    }
}

// GOOD: Single pass
void Update() {
    for (int i = 0; i < count; i++) {
        positions[i] += velocities[i] * dt;
        if (positions[i].y < 0) positions[i].y = 0;
        Draw(positions[i]);
    }
}
// Result: 2-3× faster (better temporal locality)
```

---

#### **2. Loop Blocking (Cache Tiling)**

```cpp
// GOOD: Process in cache-sized blocks
void ProcessLargeData() {
    const int BLOCK_SIZE = 8192;  // Fits in L1 cache
    
    for (int start = 0; start < 1000000; start += BLOCK_SIZE) {
        int end = std::min(start + BLOCK_SIZE, 1000000);
        
        for (int i = start; i < end; i++) {
            largeArray[i] = ExpensiveComputation(largeArray[i]);
        }
    }
}
// Result: Fewer capacity misses, 20-30% faster
```

---

### **Memory Alignment**

```cpp
// GOOD: Aligned to cache line
struct alignas(64) Transform {
    Vector3 position;  // 12 bytes
    Vector3 rotation;  // 12 bytes
    Vector3 scale;     // 12 bytes
    char padding[28];  // Pad to 64 bytes
};

// Each transform exactly one cache line
// Every access loads exactly 1 cache line!
```

---

### **SIMD Optimization**

```cpp
#include <immintrin.h>

void AddArraysSIMD(float* a, float* b, float* result, int count) {
    int i = 0;
    
    // Process 8 floats at a time (AVX)
    for (; i + 7 < count; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);
        __m256 vresult = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(&result[i], vresult);
    }
    
    // Handle remaining elements
    for (; i < count; i++) {
        result[i] = a[i] + b[i];
    }
}
// Result: 8× faster!
```

---

## 📚 Performance Best Practices Summary

### **Memory Access Patterns Cheat Sheet**

```
✅ DO:
- Access arrays sequentially
- Process similar data together (SoA)
- Keep hot data in contiguous arrays
- Use indices instead of pointers
- Pack structures to fit cache lines
- Process data in blocks that fit in cache

❌ DON'T:
- Random access patterns
- Linked lists for performance-critical data
- Mix hot and cold data in same struct
- Use pointers when indices suffice
- Ignore alignment
- Process massive arrays without blocking
```

---

### **Cache Optimization Checklist**

**Before optimizing:**
```
1. Profile first! (gprof, perf, Valgrind)
2. Identify hot functions
3. Measure cache miss rate
4. Benchmark before and after
```

**Layout optimizations:**
```
1. Convert AoS → SoA for hot paths
2. Split hot/cold data
3. Align structs to cache lines (64 bytes)
4. Use memory pools for same-sized objects
5. Replace linked lists with arrays
```

**Access pattern optimizations:**
```
1. Access data sequentially
2. Fuse loops accessing same data
3. Block large data to fit in cache
4. Prefetch data you'll need soon
5. Avoid pointer chasing
```

---

# Practical Exercises

## Exercise 1: Row-Major vs Column-Major (Easy-Medium)

**Goal:** Experience the dramatic impact of traversal order.

### **Task:**

Write a program that processes a 2D matrix in both row-major and column-major order.

```cpp
#include <iostream>
#include <chrono>

const int ROWS = 4000;
const int COLS = 4000;

void processRowMajor(int matrix[ROWS][COLS]) {
    // TODO: Implement row-major traversal
    // Traverse: for each row, then each column
}

void processColumnMajor(int matrix[ROWS][COLS]) {
    // TODO: Implement column-major traversal
    // Traverse: for each column, then each row
}

int main() {
    // Allocate matrix (heap allocation for large size)
    int (*matrix)[COLS] = new int[ROWS][COLS];
    
    // Initialize
    for (int i = 0; i < ROWS; i++) {
        for (int j = 0; j < COLS; j++) {
            matrix[i][j] = i * COLS + j;
        }
    }
    
    // TODO: Time both approaches and compare
    
    delete[] matrix;
    return 0;
}
```

**Requirements:**

1. Implement both traversal functions
2. Each function should set every element to 0
3. Time both functions using `std::chrono`
   1. `auto start = std::chrono::high_resolution_clock::now();`
   2. `auto end = std::chrono::high_resolution_clock::now();`
4. Calculate the speedup factor (slower time / faster time)
5. Explain why one is faster than the other

**Expected Results:**
- Row-major should be 20-100× faster than column-major
- If not, you may have implemented them incorrectly!

**Questions:**
- How many cache misses occur in row-major order?
- How many cache misses occur in column-major order?
- What is the cache efficiency (bytes used / bytes loaded) for each?

---

## Exercise 2: Array of Structs vs Struct of Arrays (Medium)

**Goal:** Convert object-oriented code to data-oriented design.

### **Task:**

You have a particle system using Array of Structs (AoS):

```cpp
#include <iostream>
#include <vector>
#include <chrono>

struct Particle {
    float x, y, z;      // Position (12 bytes) - HOT
    float vx, vy, vz;   // Velocity (12 bytes) - HOT
    float lifetime;     // (4 bytes) - HOT
    float r, g, b, a;   // Color (16 bytes) - COLD (only for rendering)
    int textureID;      // (4 bytes) - COLD
    float size;         // (4 bytes) - COLD
    // Total: 56 bytes per particle
};

void updateParticlesAoS(std::vector<Particle>& particles, float dt) {
    for (auto& p : particles) {
        p.x += p.vx * dt;
        p.y += p.vy * dt;
        p.z += p.vz * dt;
        p.lifetime -= dt;
    }
}

int main() {
    const int COUNT = 100000;
    std::vector<Particle> particles(COUNT);
    
    // Initialize particles
    for (int i = 0; i < COUNT; i++) {
        particles[i] = {
            (float)i, (float)i, (float)i,  // position
            1.0f, 1.0f, 1.0f,              // velocity
            10.0f,                         // lifetime
            1.0f, 0.0f, 0.0f, 1.0f,        // color
            0,                             // textureID
            5.0f                           // size
        };
    }
    
    // Benchmark
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int frame = 0; frame < 1000; frame++) {
        updateParticlesAoS(particles, 0.016f);
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "AoS Time: " << duration.count() << " ms" << std::endl;
    
    return 0;
}
```

**Your Tasks:**

1. **Create a Struct of Arrays (SoA) version:**
   ```cpp
   struct ParticleSystemSoA {
      // TODO: Implement SoA structure
   };
   ```

2. **Implement `updateParticlesSoA()` function**

3. **Benchmark both versions** for 1000 frames

4. **Calculate the speedup**

5. **Answer these questions:**
   - What is the cache efficiency of AoS? (hot bytes / total bytes per particle)
   - What is the cache efficiency of SoA? (should be 100% for hot data)
   - What speedup factor did you achieve?
   - Why is SoA faster even though it processes the same amount of data?

**Expected Results:**
- SoA should be 3-8× faster than AoS
- If not, check your implementation!

---

## Exercise 3: Hot/Cold Data Splitting (Medium)

**Goal:** Separate frequently-accessed data from rarely-accessed data.

### **Task:**

Given an enemy system where position/velocity are accessed every frame but other data is accessed rarely:

```cpp
struct Enemy {
    // Hot data (accessed every frame)
    float x, y, z;
    float vx, vy, vz;
    float health;
    bool active;
    
    // Cold data (accessed occasionally)
    int enemyType;
    float maxHealth;
    float attackRange;
    float attackDamage;
    int textureID;
    char name[32];
    
    // Total: ~80 bytes
};
```

**Your Tasks:**

1. **Split into hot and cold structures:**
   ```cpp
   struct EnemyHot {
       // TODO: Only frequently accessed data
   };
   
   struct EnemyCold {
       // TODO: Only rarely accessed data
   };
   ```

2. **Create systems to manage both:**
   ```cpp
   struct EnemySystem {
       std::vector<EnemyHot> hotData;
       std::vector<EnemyCold> coldData;
       
       void update(float dt) {
           // TODO: Update only hot data
       }
       
       void takeDamage(int index, float damage) {
           // TODO: Access both hot and cold data
       }
   };
   ```

3. **Benchmark the original vs split version** for 100,000 enemies over 1000 frames

4. **Calculate:**
   - Cache efficiency of original (bytes used per update / bytes loaded)
   - Cache efficiency of split version
   - Speedup factor

**Questions:**
- How much smaller is the hot data structure?
- How many hot structures fit in one cache line?
- What is the theoretical speedup based on cache efficiency?
- Does the actual speedup match the theoretical speedup?

**Expected Results:**
- Hot data should be ~20 bytes vs 80 bytes original
- 3-4× more hot structures per cache line
- 2-4× speedup in practice

---

## Exercise 4: Memory Pool Implementation (Medium-Hard)

**Goal:** Implement a fast, cache-friendly memory pool for game objects.

### **Task:**

Implement a memory pool for bullets in a game:

```cpp
#include <iostream>
#include <chrono>
#include <cstring>

struct Bullet {
    float x, y, z;
    float vx, vy, vz;
    float damage;
    bool active;
};

class BulletPool {
private:
    static const int POOL_SIZE = 1000;
    Bullet bullets[POOL_SIZE];
    bool inUse[POOL_SIZE];
    
public:
    BulletPool() {
        // TODO: Initialize pool
    }
    
    Bullet* Allocate() {
        // TODO: Find free slot and return pointer
        // Should be O(1) or close to it!
        return nullptr;
    }
    
    void Free(Bullet* bullet) {
        // TODO: Mark slot as free
        // Should be O(1)!
    }
    
    void UpdateAll(float dt) {
        // TODO: Update only active bullets
        // Should be cache-friendly!
    }
};

// For comparison
class BulletHeap {
public:
    std::vector<Bullet*> bullets;
    
    Bullet* Allocate() {
        Bullet* b = new Bullet();
        bullets.push_back(b);
        return b;
    }
    
    void Free(Bullet* bullet) {
        auto it = std::find(bullets.begin(), bullets.end(), bullet);
        if (it != bullets.end()) {
            bullets.erase(it);
            delete bullet;
        }
    }
    
    void UpdateAll(float dt) {
        for (auto* b : bullets) {
            if (b->active) {
                b->x += b->vx * dt;
                b->y += b->vy * dt;
                b->z += b->vz * dt;
            }
        }
    }
};
```

**Requirements:**

1. **Implement all BulletPool methods**
2. **Optimize UpdateAll()** to skip inactive bullets efficiently
3. **Benchmark both implementations:**
   - Allocate 500 bullets
   - Update for 10,000 frames
   - Free all bullets
   - Measure total time

4. **Compare:**
   - Allocation speed (pool vs heap)
   - Update speed (pool vs heap)
   - Deallocation speed (pool vs heap)

**Bonus Challenges:**
- Implement a free list for O(1) allocation
- Keep active bullets packed together for better cache usage
- Add a "defragment" function that compacts active bullets

**Expected Results:**
- Pool allocation: 5-20× faster than heap
- Pool update: 2-5× faster than heap (better cache locality)
- Pool deallocation: 10-50× faster than heap

---

## Exercise 5: Linked List vs Array Benchmark (Medium-Hard)

**Goal:** Measure the performance impact of pointer chasing.

### **Task:**

Compare linked list vs array for the same operations:

```cpp
#include <iostream>
#include <vector>
#include <chrono>

struct Node {
    int value;
    Node* next;
};

class LinkedList {
private:
    Node* head;
    int count;
    
public:
    LinkedList() : head(nullptr), count(0) {}
    
    void Add(int value) {
        // TODO: Implement
    }
    
    int Sum() {
        // TODO: Traverse and sum all values
        return 0;
    }
    
    ~LinkedList() {
        // TODO: Free all nodes
    }
};

class ArrayList {
private:
    std::vector<int> data;
    
public:
    void Add(int value) {
        data.push_back(value);
    }
    
    int Sum() {
        int sum = 0;
        for (int val : data) {
            sum += val;
        }
        return sum;
    }
};
```

**Your Tasks:**

1. **Implement LinkedList methods**

2. **Benchmark both for different sizes:**
   - 1,000 elements
   - 10,000 elements
   - 100,000 elements
   - 1,000,000 elements

3. **Measure:**
   - Insertion time
   - Sum calculation time (measures traversal)

4. **Calculate cache miss estimates:**
   - Array: How many cache misses? (consider 64-byte cache lines, 4-byte ints)
   - Linked List: How many cache misses? (each node likely in different cache line)

**Questions:**
- At what size does the linked list become significantly slower?
- What is the performance ratio (linked list time / array time)?
- How does this ratio change as the size increases?
- Can you measure this with `perf stat -e cache-misses`?

**Expected Results:**
- Small sizes (1K): Similar performance
- Large sizes (1M): Array 20-100× faster
- Linked list: ~100% cache miss rate
- Array: ~6% cache miss rate (1 miss per 16 ints)

