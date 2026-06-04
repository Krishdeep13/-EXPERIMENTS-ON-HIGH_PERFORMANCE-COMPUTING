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
