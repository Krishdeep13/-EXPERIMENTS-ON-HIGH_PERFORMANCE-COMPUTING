# Experiment 01: Creating and Joining Threads with pthreads

## Objective

The goal of this experiment is to understand how threads are created, executed, and synchronized using the POSIX Threads (pthreads) library in C.

---

## What is a Thread?

A thread is the smallest unit of execution managed by the operating system scheduler.

A process can contain multiple threads. All threads inside the same process share:

* The same address space
* Global variables
* Heap memory
* Open files

However, each thread has its own:

* Program counter
* Registers
* Stack

Threads allow multiple tasks to execute concurrently within a single process.

In this experiment, four threads execute the same function (`routine`) simultaneously.

---

## What Does `pthread_create()` Do?

The function:

```c
pthread_create(&thread_id, NULL, routine, NULL);
```

creates a new thread and asks the operating system scheduler to execute the function `routine()`.

Parameters:

1. `&thread_id`

   * Stores the identifier of the newly created thread.

2. `NULL`

   * Uses default thread attributes.

3. `routine`

   * The function that the thread will execute.

4. `NULL`

   * Argument passed to the thread function.

After creation, the thread becomes eligible for scheduling and may begin execution immediately.

---

## What Does `pthread_join()` Do?

The function:

```c
pthread_join(thread_id, NULL);
```

causes the calling thread (usually the main thread) to wait until the specified thread finishes execution.

Without joining, the main thread may continue execution and terminate the process before worker threads complete their work.

`pthread_join()` is therefore a synchronization mechanism.

It guarantees:

* The target thread has completed.
* Its resources are released properly.
* The main thread waits for the result.

---

## Why Is the Output Order Not Deterministic?

The operating system scheduler decides when each thread runs.

After creating four threads:

```c
pthread_create(...)
pthread_create(...)
pthread_create(...)
pthread_create(...)
```

the scheduler may execute them in any order.

Possible output:

```text
YES PRINTING
YES PRINTING
YES PRINTING
YES PRINTING
DONE
DONE
DONE
DONE
```

or

```text
YES PRINTING
YES PRINTING
DONE
YES PRINTING
DONE
YES PRINTING
DONE
DONE
```

or many other combinations.

The exact ordering depends on:

* Scheduler decisions
* CPU availability
* System load
* Timing

Because these factors vary between runs, thread execution order is generally unpredictable.

---

## What Happens If We Remove `pthread_join()`?

Suppose we remove:

```c
pthread_join(kd1, NULL);
pthread_join(kd2, NULL);
pthread_join(kd3, NULL);
pthread_join(kd4, NULL);
```

Then the main thread may reach:

```c
return 0;
```

before the worker threads finish.

When the main thread exits, the entire process terminates.

Possible consequences:

* Some threads never finish.
* Some output may never appear.
* Results become inconsistent.

The program may print:

```text
YES PRINTING
YES PRINTING
```

and immediately terminate before any thread reaches:

```text
DONE
```

Therefore, `pthread_join()` is necessary when the main thread must wait for worker threads to complete.

---

## Key Learning

1. Threads are independent execution paths within the same process.
2. `pthread_create()` creates a new thread and schedules it for execution.
3. `pthread_join()` synchronizes execution by waiting for a thread to finish.
4. Thread execution order is controlled by the operating system scheduler and is not deterministic.
5. Removing joins can cause the process to terminate before worker threads complete their work.
6. Thread creation introduces concurrency, while thread joining introduces synchronization.
# Experiment 02: Understanding Thread Execution, Communication, and Determinism

## Question 1: What Changes When a Program Goes from One Thread to Multiple Threads?

### Single-Threaded Execution

A normal C program starts with a single thread called the **main thread**.

Execution follows a single path:

```text
main()
  |
  +--> Step 1
  |
  +--> Step 2
  |
  +--> Step 3
  |
  +--> return
```

Only one instruction stream exists.

The CPU executes one operation after another in a predictable sequence.

---

### Multi-Threaded Execution

When:

```c
pthread_create(...)
```

is called, the operating system creates another execution context.

The process now contains multiple independent instruction streams:

```text
Process
|
+-- Main Thread
|
+-- Thread 1
|
+-- Thread 2
|
+-- Thread 3
|
+-- Thread 4
```

All threads belong to the same process and share the same memory space.

The major difference is that multiple instruction streams are now active simultaneously.

---

## Question 2: How Does This Transformation Actually Happen?

When:

```c
pthread_create(...)
```

is called, several things happen internally.

### Step 1

The operating system allocates a new stack for the thread.

Every thread requires its own:

* Stack
* Registers
* Program Counter

because each thread may be executing different instructions.

---

### Step 2

The operating system creates a Thread Control Block (TCB).

The TCB stores:

* Thread ID
* Register state
* Stack pointer
* Scheduling information

This allows the scheduler to manage the thread.

---

### Step 3

The new thread is added to the scheduler's run queue.

The scheduler now sees:

```text
Ready Threads

Main
Thread 1
Thread 2
Thread 3
Thread 4
```

and decides which one runs next.

---

### Step 4

The scheduler periodically switches between threads.

This operation is called:

**Context Switching**

The CPU saves:

```text
Registers
Program Counter
Stack Pointer
```

for one thread and restores them for another.

This creates the illusion that all threads are running simultaneously.

---

## Question 3: How Do Worker Threads Communicate Results Back to the Main Thread?

Since all threads belong to the same process:

```text
Main Thread
      |
Shared Memory
      |
Worker Threads
```

they can all access:

* Global variables
* Heap memory
* Shared data structures

Example:

```c
int result = 0;
```

Thread:

```c
result = 42;
```

Main Thread:

```c
printf("%d", result);
```

Both access the same memory location.

---

### Communication Through Return Values

A thread may also return a value:

```c
return pointer;
```

and the main thread receives it through:

```c
pthread_join(thread, &result);
```

This is similar to receiving a function's return value.

---

## Question 4: Why Is Execution Order Non-Deterministic?

The scheduler decides which thread runs.

Many factors affect this decision.

### CPU Availability

Suppose:

```text
4 Threads
2 CPU Cores
```

Only two threads can execute simultaneously.

The scheduler constantly decides who waits.

---

### Operating System Activity

The operating system may interrupt execution to:

* Handle I/O
* Schedule other programs
* Handle interrupts

This changes timing.

---

### Cache Effects

A thread may receive a cache hit.

Another may receive a cache miss.

One suddenly runs much faster.

The scheduler's future decisions change.

---

### Timing Differences

Even a difference of a few nanoseconds may completely alter the execution order.

This is known as:

**Race of Execution**

Not necessarily a race condition, but a race in timing.

---

## Question 5: How Can We Force Deterministic Behavior?

We introduce synchronization.

Examples:

### pthread_join()

Forces:

```text
Main waits
Thread finishes
Main continues
```

---

### Mutexes

Force exclusive access:

```text
Thread A enters

Thread B waits

Thread A exits

Thread B enters
```

---

### Barriers

Force all threads to stop at a checkpoint:

```text
Thread 1 ----\
Thread 2 -----\
Thread 3 ------> Barrier
Thread 4 -----/

All continue together
```

---

### Condition Variables

Allow one thread to signal another:

```text
Worker finished

Signal main thread

Main continues
```

---

## Question 6: Why Does Synchronization Create a Trade-Off?

Nothing comes for free.

Synchronization increases predictability but reduces parallelism.

---

### Without Synchronization

```text
Maximum Freedom

Maximum Parallelism

Unpredictable Order
```

Threads run independently.

Performance is high.

Behavior is difficult to reason about.

---

### With Synchronization

```text
Predictable Order

Correct Execution

Reduced Parallelism
```

Threads spend time waiting.

Waiting means fewer useful calculations.

---

### Example

Imagine:

```text
4 Workers
```

All can work simultaneously.

Total work:

```text
4 units per second
```

Now place a barrier after every step.

```text
Worker 1 waits
Worker 2 waits
Worker 3 waits
Worker 4 waits
```

Fast workers spend time idle.

Overall throughput decreases.

---

## The Fundamental Engineering Trade-Off

Concurrency introduces two competing goals:

### Goal 1

Maximize parallel execution.

Benefits:

* Higher performance
* Better CPU utilization

Cost:

* Non-determinism
* Harder debugging

---

### Goal 2

Maximize determinism and correctness.

Benefits:

* Easier reasoning
* Predictable behavior

Cost:

* Synchronization overhead
* Reduced scalability

Modern HPC, operating systems, databases, and distributed systems all revolve around balancing these two opposing forces.

The central question becomes:

"How little synchronization can I use while still guaranteeing correct behavior?"
# Thread Scheduling and Execution Control

## 1. Why Is There Only One Scheduler?

A scheduler is a component of the operating system kernel responsible for deciding which thread runs on which CPU core.

Without a scheduler, hundreds or thousands of threads could attempt to run simultaneously, causing complete chaos over CPU, memory, and hardware resources.

The scheduler acts as a traffic controller:

* Chooses which thread runs.
* Chooses when it runs.
* Chooses how long it runs.
* Chooses which CPU core executes it.

The scheduler has a global view of the system, making it possible to distribute work efficiently across available CPU cores.

---

## 2. Why Can't All Threads Run At Once?

Imagine:

* 100 threads exist.
* CPU has only 8 cores.

A core can execute only one instruction stream at a time.

Therefore:

100 threads cannot physically execute simultaneously.

The operating system must choose:

* 8 threads execute now.
* 92 threads wait.

This selection process is scheduling.

---

## 3. What If We Tried To Run Every Thread At Once?

Suppose 100 threads somehow shared a single CPU core simultaneously.

The processor would have to:

* Fetch instructions from all 100 threads.
* Maintain 100 instruction pointers.
* Execute instructions for everyone at the same instant.

This is physically impossible because a core contains only one execution pipeline (or a small fixed number in superscalar designs).

Therefore the CPU must choose which thread currently owns the core.

---

## 4. Why Does The Scheduler Switch Between Threads?

Consider:

Thread A:

* Computing data

Thread B:

* Waiting for disk

Thread C:

* Waiting for network

If the CPU stayed on Thread B while it waited for disk data:

The CPU would sit idle.

Instead:

1. Thread B blocks.
2. Scheduler notices it cannot continue.
3. Scheduler switches to Thread A.
4. CPU stays productive.

This is called context switching.

The goal is maximizing CPU utilization.

---

## 5. How Does The Scheduler Decide Who Runs?

Modern schedulers maintain information such as:

* Thread priority
* CPU usage history
* Sleep time
* Waiting time
* Fairness metrics

The scheduler attempts to:

* Keep all cores busy.
* Avoid starvation.
* Respect priorities.
* Balance workload.

Linux uses the Completely Fair Scheduler (CFS).

macOS uses a priority-based scheduler derived from Mach.

---

## 6. Can The Scheduler Give Priority To Particular Threads?

Yes.

Threads can be assigned priorities.

Example:

High Priority:

* Audio processing
* Video rendering
* Real-time control

Low Priority:

* Background logging
* File indexing

A high-priority thread generally receives CPU time before a low-priority thread.

Example:

Priority 90:
Audio playback

Priority 10:
Background update

The scheduler favors the audio thread because delays would be noticeable to the user.

---

## 7. Can Programmers Influence Scheduling?

Yes.

Programmers can:

* Set thread priority.
* Pin threads to specific CPU cores.
* Use real-time scheduling policies.
* Control synchronization points.

Examples:

pthread_setschedparam()

CPU affinity APIs

Real-time scheduling classes

These tools allow the programmer to influence execution order.

---

## 8. Why Don't We Fully Control Thread Execution Ourselves?

Suppose every application managed scheduling independently.

Application A:
"I need all cores."

Application B:
"I need all cores."

Application C:
"I need all cores."

Conflicts would immediately occur.

The operating system scheduler exists to coordinate all programs fairly across the machine.

---

## 9. What Is The Trade-Off Of Enforcing A Fixed Execution Order?

Suppose we force:

Thread 1 -> Thread 2 -> Thread 3 -> Thread 4

Advantages:

* Deterministic behavior.
* Easier debugging.
* Predictable results.

Disadvantages:

* Less parallelism.
* More waiting.
* Lower CPU utilization.
* Reduced performance.

The more constraints we impose, the less freedom the scheduler has to optimize execution.

---

## 10. Why Does HPC Often Care About Scheduling?

In High Performance Computing:

A bad schedule can cause:

* Cache misses.
* NUMA penalties.
* Load imbalance.
* Idle cores.

Researchers therefore:

* Pin threads to cores.
* Control data placement.
* Reduce synchronization.
* Minimize context switches.

The goal is keeping every core busy while minimizing communication overhead.

---

## Key Insight

Threads create potential parallelism.

The scheduler decides how much of that potential can actually be realized on the available hardware.

Good performance comes from helping the scheduler:

* Keep cores busy.
* Avoid unnecessary waiting.
* Minimize communication.
* Maximize useful computation.
# Experiment 02: Understanding Threads, Shared Memory, Execution Order, and Mutexes

## Question 1: How is the execution order determined?

### Observation

When multiple threads are created using `pthread_create()`, they do not execute in the order they were written in the source code.

Example:

```c
pthread_create(&t1,NULL,routine1,NULL);
pthread_create(&t2,NULL,routine2,NULL);
```

This does **not** guarantee:

```text
routine1 executes first
routine2 executes second
```

### Why?

After a thread is created, control is handed to the Operating System scheduler.

The scheduler decides:

* Which thread runs first.
* How long it runs.
* When it is paused.
* When another thread gets CPU time.

The scheduler makes these decisions based on:

* Available CPU cores.
* System load.
* Scheduling policy.
* Thread priorities.

Therefore execution order becomes non-deterministic.

### Example

Possible run 1:

```text
Thread 1 starts
Thread 2 starts
Thread 1 finishes
Thread 2 finishes
```

Possible run 2:

```text
Thread 2 starts
Thread 1 starts
Thread 2 finishes
Thread 1 finishes
```

Both are valid.

---

## Question 2: How is memory shared between threads?

### Observation

Threads inside the same process share the same address space.

Shared resources:

* Global variables
* Heap memory
* Open files
* Program code

Private resources:

* Registers
* Program Counter
* Stack

### Example

```c
int x = 0;
```

Thread A:

```c
x++;
```

Thread B:

```c
printf("%d\n",x);
```

Since both threads access the same memory location, Thread B may observe the modification made by Thread A.

Memory view:

```text
Process Memory

Global Variables
    x = 1

Heap

Code

     |
     |
     +------ Thread 1
     |
     +------ Thread 2
```

All threads access the same global memory.

---

## Question 3: What is a Race Condition?

### Definition

A race condition occurs when multiple threads access the same memory location and at least one thread modifies it.

The final result depends on execution timing.

### Example

Initial value:

```c
x = 0;
```

Thread 1:

```c
x++;
```

Thread 2:

```c
x++;
```

Expected:

```text
x = 2
```

Actual:

```text
x = 1
```

may occur.

### Why?

Increment is not a single operation.

The CPU performs:

```text
Read x
Add 1
Write x
```

Possible execution:

```text
Thread 1 reads x = 0

Thread 2 reads x = 0

Thread 1 writes 1

Thread 2 writes 1
```

One update is lost.

This is called a race condition.

---

## Question 4: How do Mutexes prevent race conditions?

### Definition

Mutex = Mutual Exclusion

A mutex ensures that only one thread can enter a critical section at a time.

### Example

```c
pthread_mutex_lock(&lock);

x++;

pthread_mutex_unlock(&lock);
```

Execution:

```text
Thread 1 acquires lock

Thread 2 waits

Thread 1 updates x

Thread 1 releases lock

Thread 2 acquires lock

Thread 2 updates x
```

Final result:

```text
x = 2
```

Correct and deterministic.

---

## Question 5: What is the trade-off of using a Mutex?

### Advantage

Correct results.

Prevents data corruption.

Ensures synchronization.

### Disadvantage

Reduces parallelism.

Threads spend time waiting.

Example:

Without mutex:

```text
4 threads execute simultaneously
```

With mutex:

```text
Only one thread enters critical section

Others wait
```

Thus:

```text
More correctness
Less parallelism
```

This is one of the most fundamental trade-offs in High Performance Computing.

---

## Question 6: Why are Threads useful?

Threads are useful when tasks share data and need lightweight communication.

Examples:

* Matrix multiplication
* Numerical simulations
* Image processing
* Scientific computing
* HPC kernels

Advantages:

* Shared memory
* Fast communication
* Low creation cost
* Efficient resource utilization

---

## Question 7: Why are Processes useful?

Processes provide isolation.

Each process owns its own memory space.

Advantages:

* Fault isolation
* Better security
* Independent execution
* No accidental memory corruption

Examples:

* Web browsers
* Databases
* Operating system services

If one process crashes, other processes usually continue running.

---

## What is `pthread_mutex_t lock`?

`pthread_mutex_t lock` creates a mutex object used for thread synchronization. It provides mutual exclusion by ensuring that only one thread can access a critical section at a time.

### Internal Information Maintained by a Mutex

* Lock state (Locked / Unlocked)
* Owner thread information
* Waiting thread queue
* Mutex attributes and configuration
* Scheduler and OS synchronization metadata

### Why is the Variable Needed?

The mutex variable represents the synchronization object itself. Functions such as:

```c
pthread_mutex_lock(&lock);
pthread_mutex_unlock(&lock);
```

must operate on a specific mutex instance. Without the variable, the operating system would have no synchronization object to manage.

### Technical Insight

A mutex is a shared synchronization data structure used by the operating system and scheduler to coordinate concurrent access to shared memory.

Unlike ordinary variables, mutexes interact with:

* Thread scheduling
* Ownership tracking
* Waiting queues
* Wake-up mechanisms
* Atomic CPU synchronization instructions

This additional coordination overhead makes mutex operations significantly more expensive than normal memory accesses, but necessary for preventing race conditions and ensuring correctness in multithreaded programs.
## Parallel Execution, Contention, and Data Locality

### Case 1: Shared Counter Without Synchronization

Multiple threads concurrently update the same memory location without coordination.

Characteristics:

* Maximum theoretical parallelism
* Undefined behavior due to race conditions
* Lost updates caused by overlapping read-modify-write sequences
* Non-deterministic results
* Incorrect execution despite high concurrency

This model exposes the correctness problem of shared-memory parallel programming.

---

### Case 2: Shared Counter With Mutex Protection

Threads synchronize access to the shared variable using a mutex.

Characteristics:

* Correct and deterministic execution
* Mutual exclusion guarantees data consistency
* Shared resource becomes a serialized critical section
* Threads execute concurrently but compete for ownership of the lock
* Lock acquisition introduces scheduling and synchronization overhead

This model demonstrates contention, where parallel threads exist but progress is limited by a sequentially accessed shared resource.

#### Technical Insight

Parallelism and scalability are not equivalent.

A program may execute with many threads simultaneously, yet achieve poor speedup if all threads repeatedly synchronize on the same shared resource.

Contention reduces parallel efficiency by converting computational progress into waiting time.

---

### Case 3: Thread-Local Computation with Final Reduction

Each thread maintains its own private data and performs computation independently. Results are combined only after all threads complete execution.

Characteristics:

* No shared-state contention during computation
* No mutex overhead
* No synchronization within the main computational loop
* Maximum utilization of available cores
* Communication deferred to a single reduction phase

This model follows a communication-avoiding design philosophy by replacing frequent synchronization with local computation and minimal final communication.

#### Technical Insight

High-performance computing systems favor data locality and computation independence over shared-state synchronization.

A common optimization strategy is:

* Replicate data when necessary
* Perform computation locally
* Minimize communication frequency
* Aggregate results only when required

This principle appears in thread-level parallelism, distributed-memory systems, communication-avoiding linear algebra, and large-scale scientific computing algorithms.

### Key Takeaway

The primary challenge in parallel computing is not creating threads, but minimizing synchronization and communication between them. Efficient parallel algorithms maximize local work while minimizing contention, data movement, and coordination overhead.
# Experiment 3: Local Computation and Result Aggregation using Threads

## Objective

Demonstrate how multiple threads can independently process different portions of a dataset, produce local results, and return those results to the main thread for final aggregation.

---

## Program Structure

The array:

```c
int primes[10]={2,3,5,7,11,13,17,19,23,29};
```

contains 10 prime numbers.

Two threads are created.

Each thread receives a starting index and computes the sum of 5 consecutive elements.

### Thread 1

Processes:

```text
2 + 3 + 5 + 7 + 11
```

Local Sum:

```text
28
```

### Thread 2

Processes:

```text
13 + 17 + 19 + 23 + 29
```

Local Sum:

```text
101
```

---

## Passing Arguments to Threads

Before creating a thread:

```c
int *a = malloc(sizeof(int));
*a = i * 5;
```

Dynamic memory is allocated to store the starting index.

The pointer is passed to the thread:

```c
pthread_create(&th[i], NULL, routine, a);
```

The thread receives this pointer through:

```c
void* routine(void *arg)
```

and extracts the value:

```c
int index = *(int*)arg;
```

---

## Returning Data from Threads

After computing the local sum:

```c
*(int*)arg = sum;
return arg;
```

The thread reuses the same allocated memory to store the computed result and returns the pointer back to the main thread.

The main thread retrieves the returned pointer using:

```c
pthread_join(th[i], (void**)&r);
```

and accesses the result through:

```c
global += *r;
```

---

## Global Reduction

Each thread computes its result independently.

The main thread performs the final reduction:

```c
global = local_sum_1 + local_sum_2;
```

Result:

```text
28 + 101 = 129
```

---

## Why Does the Output Order Change?

Run 1:

```text
LOCAL SUM: 101
LOCAL SUM: 28
GLOBAL SUM: 129
```

Run 2:

```text
LOCAL SUM: 28
LOCAL SUM: 101
GLOBAL SUM: 129
```

The output order is non-deterministic because thread scheduling is controlled by the operating system scheduler.

After thread creation:

```c
pthread_create(...)
```

both threads become runnable.

The scheduler may:

* Execute Thread 1 first
* Execute Thread 2 first
* Switch between them at any time

Therefore the order of intermediate outputs is not guaranteed.

---

## Deterministic vs Non-Deterministic Behavior

### Deterministic

```text
GLOBAL SUM = 129
```

This remains constant because the main thread waits for both worker threads using:

```c
pthread_join(...)
```

before computing the final result.

### Non-Deterministic

```text
LOCAL SUM: 28
LOCAL SUM: 101
```

or

```text
LOCAL SUM: 101
LOCAL SUM: 28
```

The order varies because execution timing depends on scheduler decisions.

---

## HPC Perspective

This program demonstrates a fundamental parallel computing principle:

1. Partition the data.
2. Compute locally and independently.
3. Minimize communication during computation.
4. Aggregate results through a final reduction step.

This approach scales significantly better than repeatedly synchronizing threads on a shared variable and forms the basis of many communication-avoiding and distributed-memory algorithms.
# The Hidden Cost of Thread Wakeups: Performance, CPU Cycles, and Energy Waste

In high-performance systems engineering, thread wakeups are frequently treated as instantaneous operations. In reality, unnecessary thread wakeups—especially when amplified by architectural flaws like the **Thundering Herd** problem—are silent performance and battery killers. 

When an operating system wakes up a sleeping thread, the CPU must halt application processing and execute a massive amount of low-level administrative overhead.

---

## 1. Where do the CPU Cycles Go? (The Context Switch)     #Context switching when waking up a thread is the process where a CPU stops running a current thread and switches to a previously blocked thread that is now ready to run.Why It HappensThread Blocks: A thread pauses to wait for something (like an I/O operation, a timer, or a lock).Thread Moves: The operating system (OS) moves this thread to a "waiting" or "blocked" queue.Event Occurs: The data arrives, the timer expires, or the lock is released.Thread Wakes Up: The OS moves the thread to the "ready" queue.Scheduler Intervenes: If the waking thread has a higher priority than the currently running thread, the OS scheduler triggers a context switch to run the waking thread immediately.
                                                       │
                                                       ▼
[ Loaded Thread  ] <──(Restore State from TCB)─ [ OS Scheduler ]
Save State: The CPU saves the exact state of the currently running thread. This includes CPU registers, the program counter (where it left off), and stack pointers into its Thread Control Block (TCB).Select Next: The OS scheduler selects the newly awakened thread from the ready queue.Restore State: The CPU loads the saved state of the awakened thread from its own TCB back into the CPU registers.Flush Caches (Sometimes): If the awakened thread belongs to a different process, the CPU must also switch memory spaces. This flushes the Translation Lookaside Buffer (TLB), which slows down performance.Resume Execution: The CPU jumps to the restored program counter address and the awakened thread resumes work.Performance Cost ("Overhead")Context switching is essential for multitasking but comes with a performance cost:Direct Overhead: CPU cycles spent executing the OS scheduler code rather than user applications.Indirect Overhead: The new thread finds the CPU cache "cold." It experiences cache misses because the previous thread's data is still in the L1/L2 caches, forcing the CPU to fetch data from slower main memory (RAM).To help me tailor this explanation, could you tell me if you are looking at this from an operating systems design perspective, or are you trying to optimize code in a specific language like Java, C++, or Go?24 sitesPerformance score: Context switching (General) [How-To][How-To] Performance score: Context switching (General) Practical advice for real-world IIS & ASP.NET challenges. This rule tracks...LeanSentryThreads and Concurrency - red key concepts: threads, concurrent execution, timesharing, context switch, interrupts, preemptionWhat Causes Context Switches? ... strives to maintain high CPU utilization. Hence, in addition to timesharing, context switches oc...University of WaterlooUnderstanding Threads & Multi-threading in Java | by Nisal Pubudu | MediumUsing this method, we can pause the execution of current thread for specified time in milliseconds. So, the current thread goes in...MediumShow allYou said: what is context switch in generealA context switch is the process where a computer's CPU stores the state of a running task so it can be paused, and restores the state of a different task so it can resume

When a thread wakes from a sleep state and is assigned to a physical CPU core, the processor must execute a **Context Switch**. This operation burns thousands of raw clock cycles across three distinct phases:

### Phase A: Trapping into Kernel Space
A thread cannot wake itself up. An event (such as a semaphore post or a hardware interrupt) must signal the operating system kernel. The CPU must halt its current user-space execution, swap privilege modes, and enter kernel space to execute the scheduler logic.

### Phase B: Register Swapping
A CPU core features only one physical set of hardware registers (Program Counter, Stack Pointer, General Purpose Registers) to perform computations. 
1. The kernel must copy the architecture state (registers) of the thread *currently* running on that core out into system RAM.
2. The kernel then copies the previously saved registers of your *woken* thread back into the physical CPU core.



### Phase C: Cache and TLB Destruction (The Stalling Phase)
This phase introduces the highest computational latency. Modern CPUs rely on ultra-fast L1/L2/L3 hardware caches to keep instruction pipelines fed.

* When Thread A is kicked off a core and your newly awoken Thread B takes over, the cache remains saturated with Thread A’s data footprint.
* Thread B experiences a massive wave of **Cache Misses**. The CPU core must stall (sit completely idle) for hundreds of clock cycles while it fetches Thread B’s data from the drastically slower main system RAM. 
* Furthermore, the **TLB (Translation Lookaside Buffer)**, which handles virtual-to-physical memory mapping addresses, must often be partially or fully flushed, adding memory resolution lag.

---

## 2. How it Wastes Energy (The Hardware Perspective)

Processors are engineered to be highly aggressive about saving power when threads go to sleep, and equally aggressive about scaling up power consumption when they wake up.

### Power State Transitions (C-States)
When threads sleep, modern Intel, AMD, and ARM CPUs place their cores into deep sleep modes known as **C-States** (e.g., C6 or C7). In these states, portions of the core are physically power-gated (turned off) to eliminate leakage current. Waking a thread forces the core to transition back to **C0 (Active State)**. The physical act of re-energizing the internal circuitry causes a localized electrical power surge.

### DVFS Over-Reaction Spikes
Modern CPUs utilize **DVFS (Dynamic Voltage and Frequency Scaling)** to alter clock speeds based on perceived load. If multiple threads wake up simultaneously (a Thundering Herd), the OS scheduler detects a massive surge in runnable threads. It instantly instructs the motherboard's VRMs (Voltage Regulator Modules) to spike the core's voltage and frequency to maximum, causing exponential power draw ($Power \propto Voltage^2 \times Frequency$).

---

## 3. Quantifying the Waste: Time & Energy Percentages

The architectural efficiency of your synchronization patterns directly dictates the percentage of time and energy wasted on thread management:

| Metrics & Scenarios | Time Cost | CPU Cycle Cost | Overall Performance/Energy Waste |
| :--- | :--- | :--- | :--- |
| **Single Clean Thread Switch** | 1 to 5 microseconds | ~3,000 to 15,000 cycles | **< 1% to 3%** (Normal, acceptable operating overhead) |
| **Severe Cache Pollution (Large Thread)** | 10 to 30 microseconds | ~30,000 to 90,000 cycles | **5% to 15%** (Sub-optimal system efficiency) |
| **The Thundering Herd / Lock Convoy** | Milliseconds of thrashing | Millions of wasted cycles | **30% to 85%** (The system enters "Thrashing" mode) |

### The Worst-Case Scenario: Thrashing

In a worst-case scenario where 50 threads are awakened via a broadcast to contest a single lock, the system enters a catastrophic failure state known as **Thrashing**. 

Because only one thread can successfully win the lock, the remaining 49 threads wake up, execute the context switch, load their registers, experience cache misses, realize the lock is unavailable, and immediately trigger *another* context switch to return to a sleep state.






--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Technical Technical Note: Microarchitectural Analysis of Vector Saturation and Cache Set Contention in Strided GEMM Kernels



1. Executive Summary
When implementing high-performance Basic Linear Algebra Subprograms (BLAS), specifically General Matrix Multiply (GEMM) micro-kernels on modern superscalar architectures (such as ARM Cortex or Apple M-series), vectorization via Single Instruction, Multiple Data (SIMD) can exhibit severe performance degradation—running slower than equivalent scalar implementations. This document explores the microarchitectural mechanics behind this behavior, detailing how the interaction between high-throughput Fused Multiply-Accumulate (vfmaq) instructions and power-of-two strided memory layouts (lda = 512) creates a catastrophic bottleneck through memory bandwidth starvation, execution pipeline stalls, and L1 cache set contention.



2. Microarchitectural Mechanics of Cache Set Contention
Modern CPU designs implement Set-Associative Caches to balance access latency against silicon cost. An \(N\)-way set-associative cache divides its total capacity into \(S\) sets, where each set contains \(N\) individual cache lines (typically 64 bytes wide).

64-Bit Physical Memory Address:
+-------------------------------------+--------------------------+-----------------------+

|              TAG BITS               |        INDEX BITS        |   BLOCK OFFSET BITS   |
+-------------------------------------+--------------------------+-----------------------+

                |                                  |                         |
  Identifies memory block uniqueness        Selects 1 of S Sets       Identifies byte in line

The Power-of-Two Stride Alignment Trap
When traversing a column in a row-major matrix layout, the memory offset between elements in adjacent rows is defined by the formula:
\(\text{Address}(i)=\text{Base}+i\times \text{lda}\times \text{sizeof(float)}\)
When the Leading Dimension of the Array (\(\text{lda}\)) is a clean power-of-two, such as \(512\) single-precision floats (\(512 \times 4 \text{ bytes} = 2048 \text{ bytes} = 2^{11}\) bytes), a structural hazard emerges in the cache mapping hardware:

Row 0 Address: Base + 0      (Binary: ...0000000000000000) -> Maps to Set K
Row 1 Address: Base + 2048   (Binary: ...0000100000000000) -> Maps to Set K
Row 2 Address: Base + 4096   (Binary: ...0001000000000000) -> Maps to Set K
Row 3 Address: Base + 6144   (Binary: ...0001100000000000) -> Maps to Set K
Because the stride is a multiple of the cache line size and the total number of sets (\(S\)), the lower 11 bits of every generated address are completely identical. Consequently, the hardware cache controller extracts the exact same Index Bits for every single row access.

Thrashing and Associativity Flooding
Instead of distributing data evenly across all available sets in the L1 data cache, every column access is funneled into a single, identical cache set.
1. Capacity Overload: In an 8-way set-associative cache, that specific set can hold exactly 8 cache lines. By the time the loop reaches row index i = 8, the set is entirely saturated.
2. Eviction Cascades: Requesting the 9th row forces the Least Recently Used (LRU) eviction policy to kick out the cache line containing row 0.
3. Thrashing Loop: If the algorithm tries to reuse row 0 in a subsequent loop pass or adjacent instruction, it experiences a high-latency Cache Conflict Miss. The hardware must stall execution to retrieve the line from the L2/L3 cache or main system memory.



3. The Vector Saturation Paradox: Why SIMD Runs Slower Than Scalar
It is a common misconception that changing code from scalar instructions to SIMD instructions automatically increases performance. In a strided, unaligned memory environment, vectorization often acts as a performance penalty.

1. Exponential Expansion of Bandwidth Demand
A scalar floating-point instruction handles 1 element (4 bytes). A NEON vector instruction like vfmaq_n_f32 processes 4 elements simultaneously (16 bytes) per clock cycle.
By increasing execution throughput by \(4\times\), the SIMD engine demands data from the memory subsystem \(4\times\) faster. If the memory pipeline cannot satisfy this demand because it is constantly stalling on conflict misses, the execution units spend the majority of their clock cycles idle.

2. Manual Gather Overhead and Line Splits
Because column elements are non-contiguous in memory, a 128-bit vector register cannot be filled with a single hardware load instruction (ldr qX). Instead, the CPU must perform a manual gather:

c
col_vec = vsetq_lane_f32(A[0 * lda], col_vec, 0); // Memory Request 1 (Set K)
col_vec = vsetq_lane_f32(A[1 * lda], col_vec, 1); // Memory Request 2 (Set K)
col_vec = vsetq_lane_f32(A[2 * lda], col_vec, 2); // Memory Request 3 (Set K)
col_vec = vsetq_lane_f32(A[3 * lda], col_vec, 3); // Memory Request 4 (Set K)
Use code with caution.

This code forces 4 separate scalar memory operations to construct a single vector register. Because every access hits the exact same cache set, a single vector construction can trigger 4 consecutive cache thrashing events, multiplying the eviction penalty.

3. Execution Pipeline Starvation
Modern out-of-order execution pipelines rely on deep instruction windows to hide memory latency. However, because vector instructions complete their mathematical operations rapidly, the pipeline runs out of independent instructions to execute while waiting for the 4 strided memory loads to complete. This causes the reservation stations to fill up, completely freezing the CPU pipeline.



4. Why Unoptimized Scalar (One-by-One) Code Wins
When memory is unaligned and badly structured, a primitive scalar loop often outperforms unoptimized SIMD code due to two architectural features:

SIMD Gathering Loop:
[Load Lane 0] -> Miss/Evict -> [Load Lane 1] -> Miss/Evict -> Pipeline Stalls Cold
                                                             
Scalar Sequential Loop:
[Load Float] -> Prefetcher triggers -> [Compute] -> Prefetcher fetches next row smoothly

1. Hardware Stream Prefetcher Compatibility
Modern CPUs feature sophisticated, aggressive Hardware Prefetchers that observe memory access patterns.
* Scalar Behavior: A scalar loop reads one element, performs a calculation, and requests the next element after a regular delay. The prefetcher detects this steady, predictable stride and begins speculatively fetching subsequent rows into the L1/L2 cache before the execution unit asks for them.
* SIMD Disruption: The manual gather pattern inside a SIMD loop fires multiple high-speed, conflicting memory requests simultaneously. This burst of conflicting cache allocations overwhelms and confuses the prefetcher tracking logic, causing it to drop the stream or mispredict, resulting in unmitigated memory latency.

2. Elimination of Register Packing Overhead
Scalar code loads a floating-point value directly from memory into a 32-bit scalar register (s0) and immediately feeds it to the execution unit. There are no cycles wasted executing vector assembly operations (ins / vsetq) to shift, pack, and align scattered pieces of data inside a 128-bit structure.



5. Algorithmic Remediation Strategies
To unleash the full performance of a SIMD engine, the underlying software must be restructured to guarantee high spatial and temporal data locality.

Strategy A: Matrix Dimension Padding
To break the cache set alignment trap, the leading dimension of the matrix must be altered so it is not a power-of-two. By adding a single dummy column of padding elements (\(\text{lda} = 513\)):
\(\text{Stride}=513\times 4\text{\ bytes}=2052\text{\ bytes}=\texttt{0b100000000100}\)
The lower bits of addresses for adjacent rows change dynamically, ensuring memory accesses are distributed evenly across all cache sets instead of funneling into one.

Strategy B: Block Packing (The Micro-Kernel Standard)
High-performance matrix multiplication frameworks (such as OpenBLAS or Eigen) resolve this issue by executing a Data Packing Phase before the main calculation loop.

Strided Layout (Bad for SIMD):       Contiguous Packed Block (Optimal for SIMD):
[A00] -- 512 elements -- [A10]       [A00][A10][A20][A30][A01][A11]...
By gathering the scattered column elements once and storing them sequentially in a continuous array, the memory access stride drops to exactly 1. Subsequent vector processing operations can utilize high-speed contiguous loads (vld1q_f32), keeping the CPU's memory pipelines and execution units fully saturated.
