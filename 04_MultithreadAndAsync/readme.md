# Day 4: Multithreading and Parallelism

## Understanding Parallelism

### Why Can't All Code Be Parallelized?

**Amdahl's Law Explained:**

Amdahl's Law states that speedup is limited by the sequential portion of your code:

```
Speedup = 1 / (S + P/N)

Where:
S = Sequential portion (cannot be parallelized)
P = Parallel portion (can be parallelized)
N = Number of processors
```

**Why This Happens - The Core Problem:**

```cpp
// Example: Processing 1000 enemies
void UpdateGame() {
    // SEQUENTIAL part (5% of time) - Cannot parallelize!
    CollectInputs();           // Must read input BEFORE processing
    UpdatePhysicsWorld();      // Must happen in specific order
    
    // PARALLEL part (90% of time) - Can parallelize!
    for (Enemy& enemy : enemies) {
        enemy.Update();        // Each enemy independent
    }
    
    // SEQUENTIAL part (5% of time) - Cannot parallelize!
    ResolveCollisions();       // Must happen AFTER all updates
    Render();                  // Must draw AFTER everything updated
}
```

**Why Sequential Parts Exist:**

1. **Data Dependencies:**
```cpp
// CAN'T parallelize - each step depends on previous
int x = GetInput();        // Must happen first
int y = x * 2;            // Needs x
int z = y + 10;           // Needs y
return z * 3;             // Needs z
```

2. **Order Requirements:**
```cpp
// CAN'T parallelize - order matters!
player.position += velocity;  // Must update position first
CheckCollisions(player);      // Then check collisions
ApplyDamage(player);         // Then apply damage
```

3. **Shared State Modifications:**
```cpp
// CAN'T parallelize safely - race condition!
global_score += enemy.points;  // Multiple threads writing same variable
```

**Real Amdahl's Law Impact:**

```
Example: Game with 95% parallelizable code

With 1 core:  100 ms total
With 2 cores: 50 ms (parallel) + 5 ms (sequential) = 55 ms  → 1.82× speedup
With 4 cores: 25 ms (parallel) + 5 ms (sequential) = 30 ms  → 3.33× speedup
With 8 cores: 12.5 ms (parallel) + 5 ms (sequential) = 17.5 ms → 5.71× speedup
With ∞ cores: 0 ms (parallel) + 5 ms (sequential) = 5 ms → 20× speedup (MAX!)

The 5% sequential part limits maximum speedup to 20×, 
no matter how many cores you add!
```

---

### Why Do Loops Have Parallelism Opportunities?

**Loop Iterations Can Be Independent:**

```cpp
// GOOD for parallelism - each iteration independent
for (int i = 0; i < 1000; i++) {
    enemies[i].Update(deltaTime);  // No dependency between iterations
}

// Each iteration can run on different core:
Thread 0: enemies[0-249].Update()
Thread 1: enemies[250-499].Update()
Thread 2: enemies[500-749].Update()
Thread 3: enemies[750-999].Update()
```

**Why Loops Are Parallelizable:**

1. **No Cross-Iteration Dependencies:**
```cpp
// GOOD - iteration N doesn't need result from iteration N-1
for (Particle& p : particles) {
    p.position += p.velocity * dt;  // Each particle independent
}
```

2. **Read-Only Shared Data:**
```cpp
// GOOD - all threads can read dt simultaneously
for (Enemy& e : enemies) {
    e.Update(dt);  // dt is read-only, shared safely
}
```

3. **No Write Conflicts:**
```cpp
// GOOD - each thread writes to different memory
for (int i = 0; i < count; i++) {
    results[i] = ExpensiveCalculation(inputs[i]);
}
```

**Bad Loop Patterns - Why They Can't Parallelize:**

* Loop carried dependency - for ex. sum addition is carried through the loop
* Writing to shared variables - race conditions can occur
* Order dependent operations - a list might need ordered items 1..X
* Pointer chasing - next data must be loaded before continuing

---

## Processes vs Threads

### Why Can't Processes Share Data?

**Process Memory Isolation:**

```
Process A Memory Space:        Process B Memory Space:
┌────────────────────┐        ┌────────────────────┐
│ Code Segment       │        │ Code Segment       │
│ 0x400000-0x500000  │        │ 0x400000-0x500000  │ ← Same addresses!
├────────────────────┤        ├────────────────────┤
│ Data Segment       │        │ Data Segment       │
│ int x = 42;        │        │ int x = 99;        │ ← Different data!
├────────────────────┤        ├────────────────────┤
│ Heap               │        │ Heap               │
│ malloc'd memory    │        │ malloc'd memory    │
├────────────────────┤        ├────────────────────┤
│ Stack              │        │ Stack              │
└────────────────────┘        └────────────────────┘

CPU Memory Management Unit (MMU) translates:
Process A: Virtual 0x400000 → Physical 0x1000000
Process B: Virtual 0x400000 → Physical 0x2000000
                              ↑ Different physical RAM!
```

**Why This Isolation Exists:**

1. **Security**: Process A can't read Process B's password
2. **Stability**: Process A crash doesn't affect Process B
3. **Simplicity**: Each process thinks it owns all memory

---

### Thread Properties - Why They're Fast

**Thread Memory Model:**

```
Single Process with 3 Threads:
┌──────────────────────────────────────┐
│ Shared Memory (All Threads):         │
│ ┌────────────────┐                   │
│ │ Code Segment   │ ← All threads     │
│ │                │   share code      │
│ ├────────────────┤                   │
│ │ Global Data    │ ← All threads     │
│ │ int score;     │   share globals   │
│ ├────────────────┤                   │
│ │ Heap           │ ← All threads     │
│ │ malloc'd data  │   share heap      │
│ └────────────────┘                   │
├──────────────────────────────────────┤
│ Thread-Local (Per Thread):           │
│ Thread 0:    Thread 1:    Thread 2:  │
│ ┌────────┐  ┌────────┐  ┌────────┐   │
│ │ Stack  │  │ Stack  │  │ Stack  │   │
│ │ Local  │  │ Local  │  │ Local  │   │
│ │ vars   │  │ vars   │  │ vars   │   │
│ │ CPU    │  │ CPU    │  │ CPU    │   │
│ │ regs   │  │ regs   │  │ regs   │   │
│ └────────┘  └────────┘  └────────┘   │
└──────────────────────────────────────┘
```

**Why Thread Creation Is Fast:**

Unlike a process which needs to create an entirely new memory space, load the program and copy all information, a thread only needs to allocate a new stack, set the stack pointer and instruction pointer and a queue to schedule work.

---

**Thread Execution Properties:**

```cpp
// Shared State Example:
int globalScore = 0;  // Shared by all threads

void Thread1() {
    int local = 10;   // Thread-local (on Thread1's stack)
    globalScore += local;  // Can access shared data instantly
}

void Thread2() {
    int local = 20;   // Different local variable (Thread2's stack)
    globalScore += local;  // Same globalScore, no copying needed!
}

// Memory access time:
// - Local variables: ~1 nanosecond (L1 cache)
// - Shared variables: ~1-10 nanoseconds (L2/L3 cache)
// - Process shared memory: ~100 nanoseconds (system call)
```

---

## Creating Threads in C++

### Basic Thread Creation:

```cpp
#include <thread>
#include <iostream>

// Function to run in thread
void WorkerFunction(int id) {
    std::cout << "Thread " << id << " running\n";
}

int main() {
    // Create threads
    std::thread t1(WorkerFunction, 1);
    std::thread t2(WorkerFunction, 2);
    
    // Wait for completion
    t1.join();  // Block until t1 finishes
    t2.join();  // Block until t2 finishes
    
    return 0;
}
```

### Thread with Lambda:

```cpp
std::vector<Enemy> enemies(1000);

// Create thread with lambda
std::thread updateThread([&]() {
    for (Enemy& e : enemies) {
        e.Update(0.016f);
    }
});

updateThread.join();
```

### Passing Data to Threads:

```cpp
// By value (safe - copy made)
std::thread t1(Process, 42);

// By reference (DANGEROUS - ensure lifetime!)
int data = 42;
std::thread t2(Process, std::ref(data));  // Explicitly pass reference
t2.join();  // Must join before 'data' goes out of scope!

// Pointer (DANGEROUS - ensure validity!)
int* ptr = new int(42);
std::thread t3([ptr]() {
    Process(*ptr);
    delete ptr;  // Clean up in thread
});
t3.detach();  // Runs independently
```

---

## Thread Problems and Solutions

### Problem 1: Race Conditions

**What Is a Race Condition:**

```cpp
int globalCounter = 0;

void IncrementCounter() {
    for (int i = 0; i < 1000000; i++) {
        globalCounter++;  // NOT ATOMIC!
    }
}

int main() {
    std::thread t1(IncrementCounter);
    std::thread t2(IncrementCounter);
    t1.join();
    t2.join();
    
    std::cout << globalCounter << std::endl;
    // Expected: 2,000,000
    // Actual:   ~1,234,567
}
```

**Why It Fails - Assembly Level:**

```assembly
; globalCounter++ compiles to 3 instructions:
mov eax, [globalCounter]   ; 1. Read value
add eax, 1                 ; 2. Increment
mov [globalCounter], eax   ; 3. Write back

; Race condition timeline:
Time | Thread 1                    | Thread 2
-----|-----------------------------|--------------------------
  1  | Read globalCounter (0)      |
  2  | Add 1 (result: 1)           | Read globalCounter (0)
  3  |                             | Add 1 (result: 1)
  4  | Write 1 to globalCounter    |
  5  |                             | Write 1 to globalCounter
     |                             |
Result: globalCounter = 1 (should be 2!)
```

**Solution: Mutex (Mutual Exclusion):**

```cpp
#include <mutex>

int globalCounter = 0;
std::mutex counterMutex;

void IncrementCounter() {
    for (int i = 0; i < 1000000; i++) {
        counterMutex.lock();
        globalCounter++;
        counterMutex.unlock();
    }
}
// Now value is correct, but it's slower because of locking
```

---

### Problem 2: Deadlock

**What is a Deadlock?**

```cpp
std::mutex mutex1, mutex2;

// Thread 1
void Function1() {
    mutex1.lock();
    // ... some work ...
    mutex2.lock();  // Waiting for Thread 2!
    // ... critical section ...
    mutex2.unlock();
    mutex1.unlock();
}

// Thread 2
void Function2() {
    mutex2.lock();
    // ... some work ...
    mutex1.lock();  // Waiting for Thread 1!
    // ... critical section ...
    mutex1.unlock();
    mutex2.unlock();
}

// DEADLOCK:
// Thread 1 holds mutex1, waits for mutex2
// Thread 2 holds mutex2, waits for mutex1
// Both wait forever!
```

**Solution: Lock Ordering:**

```cpp
// Always acquire locks in same order
void Function1() {
    mutex1.lock();  // Always lock mutex1 first
    mutex2.lock();  // Then mutex2
    // ... work ...
    mutex2.unlock();
    mutex1.unlock();
}

void Function2() {
    mutex1.lock();  // Same order!
    mutex2.lock();
    // ... work ...
    mutex2.unlock();
    mutex1.unlock();
}
```

---

### Problem 3: Thread Blocking and Waiting

**What Is Thread Blocking:**

```cpp
std::mutex dataMutex;
std::vector<int> sharedData;

void ProcessData() {
    dataMutex.lock();  // If another thread holds lock...
    // Thread BLOCKS here - not running, not consuming CPU
    // OS removes thread from scheduler
    // Thread enters WAITING state
    
    sharedData.push_back(42);
    
    dataMutex.unlock();  // Thread UNBLOCKS, returns to RUNNING
}
```

**Thread States:**

```
NEW → READY → RUNNING → BLOCKED → READY → RUNNING → TERMINATED
       ↑         ↓         ↑
       └─────────┴─────────┘
       (Scheduler cycles)

RUNNING: Actively executing on CPU core
BLOCKED: Waiting for lock/I/O, not consuming CPU
READY:   Waiting for CPU time slice
```

**Cost of Blocking:**

```cpp
// Context switch cost: ~1-10 microseconds
void BadCode() {
    for (int i = 0; i < 1000000; i++) {
        mutex.lock();    // Context switch (~5μs)
        DoTinyWork();    // Actual work (~0.01μs)
        mutex.unlock();  // Context switch (~5μs)
        // 10μs overhead for 0.01μs work = 1000× slower!
    }
}

// Better: Reduce locking
void GoodCode() {
    std::vector<int> localResults;
    for (int i = 0; i < 1000000; i++) {
        localResults.push_back(DoTinyWork());  // No locks!
    }
    
    mutex.lock();    // Lock once
    globalResults.insert(globalResults.end(), 
                        localResults.begin(), 
                        localResults.end());
    mutex.unlock();  // Much faster!
}
```

---

### Why Thread Count ≈ CPU Cores?

**Hardware Reality:**

```
CPU with 8 cores can execute 8 threads simultaneously
More threads = time-slicing = context switches = overhead

Example with 8 cores:
8 threads:   Each gets 100% of one core       = 100% efficiency
16 threads:  Each gets 50% of one core        = ~90% efficiency (context switch overhead)
32 threads:  Each gets 25% of one core        = ~70% efficiency
64 threads:  Each gets 12.5% of one core      = ~50% efficiency
1000 threads: Constant context switching       = ~10% efficiency

Context switches aren't free!
```

**Optimal Thread Count:**

```cpp
// For CPU-bound work:
int optimalThreads = std::thread::hardware_concurrency();
// Returns number of CPU cores (e.g., 8)

// For I/O-bound work (waiting on disk/network):
int optimalThreads = std::thread::hardware_concurrency() * 2;
// Can use more threads since they're often blocked

// For game engine:
int renderThread = 1;     // One render thread
int physicsThreads = 2;   // Small physics team
int audioThread = 1;      // One audio thread
int aiThreads = cores - 4; // Rest for AI/gameplay
```

**Hyperthreading Note:**

```
CPU reports: 16 logical cores
Actual hardware: 8 physical cores with 2-way hyperthreading

Each physical core can run 2 threads simultaneously
BUT: They share execution units
Performance: ~1.3× speedup (not 2×)

Best practice: Treat as physical core count for CPU-bound work
```

---

## Thread Pools and Task Systems

#### Thread Pools Explained:

**Why Thread Pools:**

```cpp
// BAD: Creating threads per task
for (int i = 0; i < 1000; i++) {
    std::thread t([i]() {
        ProcessTask(i);  // ~0.1ms of work
    });
    t.detach();
}
// Creates 1000 threads!
// Thread creation: 1000 × 10μs = 10ms
// Actual work: 1000 × 0.1ms = 100ms
// Overhead: 10% wasted on thread creation!

// GOOD: Thread pool reuses threads
ThreadPool pool(8);  // Create 8 worker threads once
for (int i = 0; i < 1000; i++) {
    pool.EnqueueTask([i]() {
        ProcessTask(i);
    });
}
// Thread creation: 8 × 10μs = 80μs (once)
// Actual work: 100ms
// Overhead: 0.08% - negligible!
```

**Thread Pool Implementation:**

```cpp
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>

class ThreadPool {
private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queueMutex;
    std::condition_variable condition;
    bool stop = false;
    
public:
    ThreadPool(size_t numThreads) {
        for (size_t i = 0; i < numThreads; i++) {
            workers.emplace_back([this]() {
                while (true) {
                    std::function<void()> task;
                    
                    {
                        std::unique_lock<std::mutex> lock(queueMutex);
                        
                        // Wait for task or stop signal
                        condition.wait(lock, [this]() {
                            return stop || !tasks.empty();
                        });
                        
                        if (stop && tasks.empty()) return;
                        
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    
                    task();  // Execute task
                }
            });
        }
    }
    
    void EnqueueTask(std::function<void()> task) {
        {
            std::unique_lock<std::mutex> lock(queueMutex);
            tasks.push(std::move(task));
        }
        condition.notify_one();  // Wake up one worker
    }
    
    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queueMutex);
            stop = true;
        }
        condition.notify_all();
        for (std::thread& worker : workers) {
            worker.join();
        }
    }
};
```

### Thread Pools vs Tasks - Benefits and Drawbacks:

**Thread Pool Approach:**

```cpp
ThreadPool pool(8);

// Enqueue work
for (Enemy& e : enemies) {
    pool.EnqueueTask([&e]() {
        e.Update(0.016f);
    });
}

// Wait for completion (need to add this functionality)
pool.WaitForCompletion();
```

**Benefits:**
- ✓ Reuses threads (no creation overhead)
- ✓ Controls concurrency (won't create 10,000 threads)
- ✓ Simple to use
- ✓ Good for long-running tasks

**Drawbacks:**
- ✗ Manual work distribution
- ✗ No automatic load balancing
- ✗ Need to implement wait/synchronization
- ✗ No task dependencies

**Task-Based Approach:**

```cpp
std::vector<std::future<void>> futures;

for (Enemy& e : enemies) {
    futures.push_back(std::async(std::launch::async, [&e]() {
        e.Update(0.016f);
    }));
}

// Wait for all
for (auto& f : futures) {
    f.wait();
}
```

**Benefits:**
- ✓ Built-in synchronization (futures)
- ✓ Return values from tasks
- ✓ Exception propagation
- ✓ Standard library (no custom code)

**Drawbacks:**
- ✗ May create many threads (implementation-dependent)
- ✗ Less control over scheduling
- ✗ Overhead for small tasks
- ✗ No work stealing

---

## Thread Safety in C++

### Thread Safety Fundamentals:

**What Is Thread-Safe Code:**

```cpp
// NOT thread-safe
class Counter {
    int value = 0;
public:
    void Increment() {
        value++;  // Can cause race conditions!
    }
    int Get() { return value; }
};

// Thread-safe with mutex
class SafeCounter {
    int value = 0;
    mutable std::mutex mutex;  // mutable allows locking
public:
    void Increment() {
        std::lock_guard<std::mutex> lock(mutex);
        value++;
    }
    int Get() const {
        std::lock_guard<std::mutex> lock(mutex);
        return value;
    }
};
```

### Locks and Mutexes in C++:

**Basic Mutex:**

```cpp
#include <mutex>

std::mutex dataMutex;
std::vector<int> sharedData;

void AddData(int value) {
    dataMutex.lock();
    sharedData.push_back(value);
    dataMutex.unlock();
}
```

**RAII Lock Guard (Better):**

```cpp
void AddData(int value) {
    std::lock_guard<std::mutex> lock(dataMutex);
    sharedData.push_back(value);
    // Automatically unlocks when 'lock' goes out of scope
    // Exception-safe!
}
```

**Unique Lock (Most Flexible):**

```cpp
void ComplexOperation() {
    std::unique_lock<std::mutex> lock(dataMutex);
    
    // Can unlock early
    DoWorkWithLock();
    lock.unlock();
    
    DoWorkWithoutLock();  // Don't hold lock unnecessarily
    
    // Can re-lock
    lock.lock();
    FinishWorkWithLock();
    
    // Auto-unlocks at end of scope
}
```

**Shared Mutex (Reader-Writer Lock):**

```cpp
#include <shared_mutex>

std::shared_mutex dataMutex;
std::map<std::string, int> gameData;

// Many readers can access simultaneously
int ReadData(const std::string& key) {
    std::shared_lock<std::shared_mutex> lock(dataMutex);  // Shared lock
    return gameData[key];
    // Multiple threads can hold shared locks simultaneously
}

// Only one writer at a time
void WriteData(const std::string& key, int value) {
    std::unique_lock<std::shared_mutex> lock(dataMutex);  // Exclusive lock
    gameData[key] = value;
    // Exclusive lock blocks all readers and writers
}
```

---

### Atomic Operations in C++:

**What Are Atomics:**

Atomic operations are indivisible - they complete entirely or not at all, with no intermediate visible state.
They were implemented as a means to negate the overhead of mutexes enabling safe data sharing.

#### Why Atomics Are Necessary
Standard operations like counter++ are not atomic; they involve three steps: 
>Read → Increment → Write
 
In a multi-threaded program, another thread can "interleave" between these steps, causing a data race where updates are lost.

```cpp
#include <atomic>

// NOT atomic - can be interrupted mid-operation
int normalCounter = 0;
normalCounter++;  // 3 assembly instructions: load, add, store

// ATOMIC - guaranteed to complete as single operation
std::atomic<int> atomicCounter(0);
atomicCounter++;  // Single atomic instruction: lock inc [mem]
// Or: atomicCounter.fetch_add(1);
```

---

**Common Atomic Operations:**

```cpp
std::atomic<int> counter(0);

// Atomic increment/decrement
counter++;                    // Atomic increment
counter--;                    // Atomic decrement
counter += 5;                 // Atomic add
counter.fetch_add(5);         // adds a non-atomic value to an atomic object and 
// obtains the previous value of the atomic 

// Atomic compare-and-swap
int expected = 0;
int desired = 1;
if (counter.compare_exchange_strong(expected, desired)) {
    // true if the referenced object was successfully changed. 
} else {
    // false otherwise (counter was not '0')
}

// Atomic load/store
int value = counter.load();   // Atomic read
counter.store(42);            // Atomic write
```

**When to Use Atomics:**

```cpp
// ✓ GOOD: Simple counter
std::atomic<int> frameCount(0);
frameCount++;  // Fast, lock-free

// ✓ GOOD: Flag
std::atomic<bool> shouldStop(false);
if (shouldStop.load()) return;

// ✗ BAD: Complex data structure
std::atomic<std::vector<int>> data;  // WON'T COMPILE!
// Atomics only work with trivially copyable types

// ✗ BAD: Multiple operations
std::atomic<int> balance(100);
if (balance > 50) {  // Not atomic with next line!
    balance -= 50;   // Race condition between check and subtract
}

// ✓ CORRECT: Use compare_exchange
int expected, desired;
do {
    expected = balance.load();
    if (expected < 50) break;
    desired = expected - 50;
} while (!balance.compare_exchange_weak(expected, desired));
```

**Memory Ordering in Atomics:**

```cpp
// Relaxed: No ordering guarantees (fastest)
counter.fetch_add(1, std::memory_order_relaxed);

// Acquire/Release: Synchronization (most common)
ready.store(true, std::memory_order_release);  // Writer
if (ready.load(std::memory_order_acquire)) {    // Reader

// Sequential Consistent: Strongest guarantee (default, slowest)
counter++;  // Implicitly uses memory_order_seq_cst
```

---

## Custom Job System Example in C++

### Simple Job System Implementation:

```cpp
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <atomic>

class JobSystem {
public:
    using Job = std::function<void()>;
    
private:
    std::vector<std::thread> workers;
    std::queue<Job> jobQueue;
    std::mutex queueMutex;
    std::condition_variable condition;
    std::atomic<int> activeJobs{0};
    std::atomic<bool> stop{false};
    
public:
    JobSystem(size_t numThreads = std::thread::hardware_concurrency()) {
        for (size_t i = 0; i < numThreads; i++) {
            workers.emplace_back([this]() {
                WorkerLoop();
            });
        }
    }
    
    ~JobSystem() {
        stop = true;
        condition.notify_all();
        for (auto& worker : workers) {
            worker.join();
        }
    }
    
    void AddJob(Job job) {
        {
            std::lock_guard<std::mutex> lock(queueMutex);
            jobQueue.push(std::move(job));
        }
        activeJobs++;
        condition.notify_one();
    }
    
    void WaitForAll() {
        while (activeJobs > 0) {
            std::this_thread::yield();
        }
    }
    
private:
    void WorkerLoop() {
        while (!stop) {
            Job job;
            
            {
                std::unique_lock<std::mutex> lock(queueMutex);
                condition.wait(lock, [this]() {
                    return stop || !jobQueue.empty();
                });
                
                if (stop && jobQueue.empty()) return;
                
                job = std::move(jobQueue.front());
                jobQueue.pop();
            }
            
            job();
            activeJobs--;
        }
    }
};

// Usage:
JobSystem jobs(8);

// Add jobs
for (int i = 0; i < 1000; i++) {
    jobs.AddJob([i]() {
        ProcessEnemy(i);
    });
}

// Wait for completion
jobs.WaitForAll();
```

### Parallel For Helper:

```cpp
void ParallelFor(int start, int end, std::function<void(int)> func) {
    JobSystem& jobs = GetGlobalJobSystem();
    
    int numThreads = std::thread::hardware_concurrency();
    int range = end - start;
    int jobSize = (range + numThreads - 1) / numThreads;
    
    for (int t = 0; t < numThreads; t++) {
        int jobStart = start + t * jobSize;
        int jobEnd = std::min(jobStart + jobSize, end);
        
        if (jobStart >= end) break;
        
        jobs.AddJob([=]() {
            for (int i = jobStart; i < jobEnd; i++) {
                func(i);
            }
        });
    }
    
    jobs.WaitForAll();
}

// Usage:
ParallelFor(0, enemies.size(), [&](int i) {
    enemies[i].Update(deltaTime);
});
```

---

## Async/Await Examples in C++ (Futures and Promises)

### std::async - Simple Async Operations:

```cpp
#include <future>

// Launch async task
std::future<int> result = std::async(std::launch::async, []() {
    // Expensive computation on separate thread
    return ExpensiveCalculation();
});

// Do other work on main thread
DoMainThreadWork();

// Get result (blocks if not ready)
int value = result.get();
```

### Async Texture Loading (Raylib):

```cpp
// Load texture asynchronously
std::future<Image> LoadImageAsync(const char* filename) {
    return std::async(std::launch::async, [filename]() {
        // Load image data on worker thread
        Image img = LoadImage(filename);  // File I/O can be threaded
        return img;
    });
}

// Usage in game loop:
std::future<Image> futureImage = LoadImageAsync("player.png");
bool imageReady = false;
Texture2D texture;

// In update loop:
if (!imageReady && futureImage.wait_for(std::chrono::seconds(0)) == std::future_status::ready) {
    Image img = futureImage.get();
    texture = LoadTextureFromImage(img);  // Must be on main thread!
    UnloadImage(img);
    imageReady = true;
}
```

### std::promise - Manual Control:

```cpp
std::promise<int> promise;
std::future<int> future = promise.get_future();

std::thread worker([&promise]() {
    // Do work...
    int result = 42;
    promise.set_value(result);  // Fulfill promise
});

// Main thread waits for result
int value = future.get();  // Blocks until promise fulfilled
worker.join();
```

---

## Assignment

## Parallel Sum

### **Goal**
Implement parallel summation of a large array and compare performance with single-threaded version. Learn about thread-safe accumulation and measure actual speedup.

### **Background**
Summing an array is a classic parallel problem. Each thread can sum a portion of the array independently, then combine results. However, combining results requires careful synchronization to avoid race conditions.

### **The Problem**
Given an array of 10 million random integers (1-10), compute the sum using multiple threads and compare performance with a single-threaded version.

### **Starter Code**

```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>
#include <chrono>
#include <numeric>

// ============================================
// TODO: Implement these three versions
// ============================================

// Version 1: Single-threaded baseline
long long SequentialSum(const std::vector<int>& data) {
    // TODO: Implement simple sequential sum
    // Hint: Use a for loop or std::accumulate
    return 0;
}

// Version 2: Parallel with mutex (simple but slow)
long long ParallelSum_Mutex(const std::vector<int>& data, int numThreads) {
    long long globalSum = 0;
    std::mutex sumMutex;
    
    // TODO: Create threads
    // Each thread should:
    // 1. Calculate which portion of array to process
    // 2. Sum its portion locally
    // 3. Lock mutex and add to globalSum
    // 4. Unlock mutex
    
    std::vector<std::thread> threads;
    
    // TODO: Divide work among threads
    // Hint: divide data size by your threads
    // int chunkSize = ...
    
    for (int t = 0; t < numThreads; t++) {
        // Calculate start and end indices for this thread
        // int start = t * chunkSize;
        // int end = (t == numThreads - 1) ? data.size() : start + chunkSize;
        
        threads.emplace_back([&, start, end]() {
            // TODO: Sum local portion
            long long localSum = 0;
            // for (int i = start; i < end; i++) { ... }
            
            // TODO: Add to global sum with mutex
        });
    }
    
    // TODO: Join all threads
    // for each thread: t.join();
    
    return globalSum;
}

// Version 3: Parallel with thread-local reduction (fastest)
long long ParallelSum_ThreadLocal(const std::vector<int>& data, int numThreads) {
    // Create vector to store each thread's sum
    std::vector<long long> threadSums(numThreads, 0);
    
    std::vector<std::thread> threads;
    
    for (int t = 0; t < numThreads; t++) {
        // TODO: Similar to Version 2, but store result in threadSums[t]
        // No mutex needed - each thread writes to different array element
        
        threads.emplace_back([&, t]() {
            // TODO: Calculate start and end
            // TODO: Sum local portion
            // TODO: Store in threadSums[t] (no lock needed!)
        });
    }
    
    // TODO: Join threads
    
    // TODO: Final reduction - sum all threadSums
    long long globalSum = 0;
    // for each sum in thread sums..
    
    return globalSum;
}

// ============================================
// Benchmark function (provided)
// ============================================

void Benchmark() {
    const int SIZE = 10000000;  // 10 million elements
    int numThreads = std::thread::hardware_concurrency();
    
    std::cout << "Array size: " << SIZE << " elements\n";
    std::cout << "Hardware threads: " << numThreads << "\n\n";
    
    // Initialize data
    std::vector<int> data(SIZE);
    for (int i = 0; i < SIZE; i++) {
        data[i] = i % 100;
    }
    
    // Test sequential
    auto start = std::chrono::high_resolution_clock::now();
    long long result1 = SequentialSum(data);
    auto end = std::chrono::high_resolution_clock::now();
    auto timeSeq = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    
    std::cout << "Sequential:    " << timeSeq << " ms (sum: " << result1 << ")\n";
    
    // Test parallel with mutex
    start = std::chrono::high_resolution_clock::now();
    long long result2 = ParallelSum_Mutex(data, numThreads);
    end = std::chrono::high_resolution_clock::now();
    auto timeMutex = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    
    std::cout << "Mutex:         " << timeMutex << " ms (sum: " << result2 << ")\n";
    
    // Test parallel with thread-local
    start = std::chrono::high_resolution_clock::now();
    long long result3 = ParallelSum_ThreadLocal(data, numThreads);
    end = std::chrono::high_resolution_clock::now();
    auto timeLocal = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    
    std::cout << "Thread-Local:  " << timeLocal << " ms (sum: " << result3 << ")\n\n";
    
    // Calculate speedups
    std::cout << "Speedups:\n";
    std::cout << "  Mutex vs Sequential:       " << (double)timeSeq / timeMutex << "×\n";
    std::cout << "  Thread-Local vs Sequential: " << (double)timeSeq / timeLocal << "×\n";
}

int main() {
    Benchmark();
    return 0;
}
```

### **Estimated Output**

```
Array size: 10000000 elements
Hardware threads: 16

Sequential:    ~10 ms (sum: 495000000)
Mutex:         ~8 ms (sum: 495000000)
Thread-Local:  ~7 ms (sum: 495000000)

Speedups:
  Mutex vs Sequential:       ~1.25×
  Thread-Local vs Sequential: ~1.43×
```

### **Questions to Answer**

1. Why is the mutex version only slightly faster than sequential?
2. Why is thread-local much faster than mutex?
3. Why don't you get 8× speedup with 8 threads?
4. What would happen if you locked the mutex inside the loop for each addition?

### **Hints**

- **Dividing work:** `int chunkSize = (data.size() + numThreads - 1) / numThreads;`
- **Thread lambda:** Capture `&` for data, but capture `start, end` by value
- **std::lock_guard:** RAII wrapper that automatically unlocks
- **False sharing:** Pad threadSums to avoid cache line ping-pong (bonus)