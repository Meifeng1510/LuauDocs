# **Luau Garbage Collection Guide: In-depth**

---

## Introduction

Garbage collection is an essential aspect of modern programming languages, providing automatic memory management that simplifies development and helps prevent memory-related issues. At its core, garbage collection is the process by which a program identifies and reclaims memory that is no longer in use, ensuring the system remains efficient and responsive.

In Luau, this process is particularly important because of its use in environments like Roblox, where performance and memory optimization are critical. Memory management is primarily handled by the garbage collector (GC), which is responsible for automatically reclaiming memory that is no longer in use. This in-depth guide dives into the internal workings of Luau's garbage collection system, exploring its architecture, goals, and the challenges it overcomes to ensure efficient memory management.

---

### Overview of Garbage Collection

Garbage collection is the process of identifying and reclaiming memory occupied by objects that are no longer accessible or needed by the program. Without garbage collection, unused objects would accumulate in memory, leading to memory exhaustion and potential application crashes.

A garbage collector works by:

1. Tracking objects allocated in memory.
2. Identifying objects that are no longer reachable.
3. Reclaiming the memory occupied by such objects and making it available for future allocations.

Modern garbage collectors aim to perform these tasks efficiently, ensuring minimal impact on program execution.

---

### Interaction Between the Operating System and Garbage Collector

The operating system (OS) provides memory to the Luau runtime in the form of large chunks, often referred to as memory pages. The Luau runtime then manages this memory internally, subdividing it into smaller [units](LuauGarbageCollectorInDepth.md#luau-heap-structure-and-memory-allocation) required by scripts and their objects. The garbage collector operates within this framework, ensuring that:

- Allocated memory is actively used.
- Unused memory is reclaimed and returned to the pool of available memory.
- Memory fragmentation (where memory is scattered in non-contiguous blocks) is minimized.

The OS and Luau runtime work in tandem to manage memory allocation and deallocation. While the OS handles the overall physical memory of the system, the garbage collector focuses on optimizing memory usage within the Luau environment.

Luau's garbage collector operates in cycles, periodically scanning the memory for unused objects. This cycle-based approach ensures that memory is reclaimed systematically, balancing performance and resource usage.

---

### Goals of Luau's Garbage Collector

The design of Luau's garbage collector is driven by several key goals:

1. **Performance:** Minimize the overhead introduced during garbage collection cycles to avoid noticeable performance dips.
2. **Scalability:** Handle a wide range of workloads, from small scripts to large-scale applications with numerous objects and complex memory requirements.
3. **Memory Efficiency**: Ensure that unused memory is reclaimed promptly to avoid memory leaks and excessive memory consumption.
4. **Low Latency:** Reduce the impact of garbage collection pauses, particularly in real-time scenarios such as gaming.

---

### Challenges in Garbage Collection

While garbage collection simplifies memory management, it also introduces challenges:

1. **Real-Time Constraints:** In scenarios like games, long pauses for garbage collection are unacceptable. The GC must be designed to operate incrementally or concurrently to ensure responsiveness.
2. **Memory Fragmentation:** Objects of varying sizes can lead to fragmented memory, making it difficult to allocate new objects efficiently.
3. **Weak References:** Managing weakly-referenced objects (e.g., in weak tables) adds complexity, as these objects must be collected without disrupting references elsewhere.

---

### What to Expect in This Guide

This guide provides an in-depth exploration of the garbage collection system in Luau, including:

- **Core Concepts and Terminology**: A comprehensive explanation of garbage collection principles, including object reachability, memory reclamation, and tri-color marking schemes.
- **Internal Mechanisms**: Detailed workflows for the marking, atomic, and sweeping phases of Luau's incremental, non-generational, non-moving garbage collector.
- **Memory Optimization Strategies**: Insights into how Luau handles weak references, minimizes memory fragmentation, and addresses memory pressure.
- **Tuning and Configuration**: Practical guidance on using settings like the GC pacer, step size, multiplier, and goal to optimize garbage collection performance.
- **Special Cases and Advanced Topics**: A closer look at how Luau processes weak tables, strings, threads, and upvalues, along with techniques to reduce GC pauses.

By understanding these aspects, you'll gain the knowledge to optimize your applications for performance and memory efficiency within the Luau runtime.

---

## Garbage Collector  

A **garbage collector (GC)** is a system designed to manage memory automatically by reclaiming unused memory and making it available for future use. In programming, when objects are created, they occupy a portion of the system's memory. When these objects are no longer needed, the garbage collector identifies and removes them, preventing memory leaks and optimizing memory usage.

The garbage collector operates on specific strategies to determine which objects are still in use and which can be discarded. The three foundational strategies are **tracing**, **reference counting**, and **escape analysis**, which form the basis for many garbage collection algorithms.

---

### Basic Garbage Collector Strategies  

1. **Tracing Garbage Collection:**  
   This approach determines reachability starting from a set of root objects (e.g., global state, main thread, global table, and registry). If an object can be reached by traversing references from the roots, it is considered "alive." Tracing garbage collectors use algorithms like **mark-and-sweep** or **mark-and-compact** to identify and clean up unreachable objects.  

2. **Reference Counting:**  
   In this strategy, each object keeps a count of the references pointing to it. When the reference count drops to zero (i.e., no references point to the object), the object is immediately deallocated. However, this method struggles with circular references, where objects reference each other but are no longer reachable.  

3. **Escape Analysis:**  
   Escape analysis is an optimization strategy used by modern compilers to reduce the burden on garbage collection. It analyzes the lifetime and scope of objects during compilation and determines whether an object "escapes" the scope in which it was created.  
   - If the compiler determines that an object does not escape its scope, it can allocate the object on the stack instead of the heap.  
   - Stack-allocated objects are automatically cleaned up when the stack frame is destroyed, avoiding the need for garbage collection entirely.  

   Escape analysis can significantly reduce the number of objects that need to be managed by the garbage collector, leading to improved performance and reduced memory overhead.

---

### Common Types of Garbage Collectors  

1. **Mark-and-Sweep:**  
   This method involves two phases:  
   - **Mark Phase:** Starting from the root, the GC traverses all references and marks reachable objects.  
   - **Sweep Phase:** The GC iterates through memory and reclaims all unmarked objects, freeing their memory.  

2. **Reference Counting:**  
   Each object has a reference count, which tracks the number of references pointing to it.  
   - When the count drops to zero, the object is considered garbage and can be deallocated.  
   This method is simple but can have performance overhead due to frequent updates to reference counts.

3. **Mark-and-Compact:**  
   Similar to mark-and-sweep, but after identifying garbage, the GC compacts the memory by moving live objects together. This reduces fragmentation but requires updating references to moved objects.  

4. **Generational Garbage Collection:**  
   Objects are categorized into generations based on their lifespan. Newer objects (short-lived) are collected more frequently, while older objects (long-lived) are collected less often. This is based on the empirical observation that most objects die young, so by focusing more on collecting short-lived objects, efficiency can be improved.  

5. **Copying Garbage Collection:**  
   The heap is divided into two areas, and live objects are copied from one to the other during collection.  
   - The garbage collector compacts memory by moving live objects to a free space, eliminating fragmentation.  
   - Once copying is complete, the old area is cleared and reused.

---

### Mark-and-Sweep Garbage Collector  

The **mark-and-sweep** garbage collector is one of the simplest and most widely used algorithms due to its straightforward implementation and effectiveness. It operates in two distinct phases:  

1. **Mark Phase:**  
   - Starts from root references (global state, main thread, global table, etc.).  
   - Traverses all references and marks objects that are reachable.  

2. **Sweep Phase:**  
   - Iterates through the memory and identifies unmarked objects.  
   - Reclaims memory from unmarked objects by marking their space as free.  

Mark-and-sweep garbage collectors can be further classified based on how they execute their cycles:  

- **Stop-the-World Mark-and-Sweep:** The entire program is paused during the GC cycle.  
- **Incremental Mark-and-Sweep:** The GC works in small steps, interleaving with the program's execution to reduce noticeable pauses.  

---

## The Luau Garbage Collector  

Luau employs an **incremental, non-generational, non-moving mark-and-sweep garbage collector.**  

1. **Incremental:**  
   - The GC performs its tasks in smaller steps rather than halting the entire program. This ensures smoother performance and minimizes interruptions, especially important for real-time systems like games.  

2. **Non-Generational:**  
   - Unlike generational GCs, Luau does not differentiate objects based on their age. All objects are treated equally during garbage collection.  

3. **Non-Moving:**  
   - Objects are not relocated in memory during garbage collection. This avoids the need to update references to objects, simplifying the process and maintaining compatibility with existing code.  

By combining these characteristics, Luau's garbage collector is designed to balance performance and efficiency, ensuring minimal overhead while maintaining smooth execution in real-time environments.

---

### Garbage Collection Workflow Simplified Overview  

The garbage collection process in Luau follows a structured workflow to identify and reclaim memory occupied by unreachable objects. Below is a step-by-step breakdown of its key stages:  

---

#### **1. Mark Phase**  

The **mark phase** is responsible for identifying all reachable (alive) objects in the memory. This phase ensures that objects in use are preserved while leaving unreachable objects unmarked for later cleanup.  

**Steps in the Mark Phase:**  

1. **Start from Roots:**  
   - The GC begins by identifying root references, such as global variables, local variables in stack frames, and function call contexts.  
2. **Traverse and Mark:**  
   - The GC recursively traverses all references starting from the roots, marking each reachable object.  

**Incremental Marking:**  
To minimize the impact on program execution, Luau performs marking incrementally:  

- Instead of marking all objects in one go, the GC distributes the work over multiple steps.  
- Each step processes a small subset of objects, reducing noticeable pauses in execution.  
- Marking is performed at the granularity of individual objects.  

---

#### **2. Atomic Phase**  

The **atomic phase** acts as a checkpoint to ensure the marking process is complete and consistent before moving to the next stage.  

**Key Characteristics of the Atomic Phase:**  

- **Indivisible Execution:** The atomic phase runs entirely as a single operation to prevent inconsistencies in the marking process caused by simultaneous program activity.  
- **Weak Reference Handling:**  
  - Weak references and weak tables (tables where keys or values are weakly referenced) are processed in this phase.  
  - Weakly referenced objects that are no longer reachable are cleared, ensuring that weak tables remain valid.  

This phase ensures that the marking stage is finalized and that all reachable objects are correctly identified.

---

#### **3. Sweep Phase**  

The **sweep phase** is responsible for reclaiming memory occupied by unreachable (unmarked) objects.  

**Steps in the Sweep Phase:**  

1. **Identify Unmarked Objects:**  
   - Objects that remain unmarked after the mark phase are considered garbage.  
2. **Free Memory:**  
   - The GC reclaims the memory occupied by these objects, marking the memory space as free for future use.  

**Paged Sweeping:**  

- Unlike the mark phase, which operates at the granularity of individual objects, sweeping in Luau is performed at the granularity of [memory pages](LuauGarbageCollectorInDepth.md#luau-heap-structure-and-memory-allocation).  
- This **paged sweeper** approach further reduces GC overhead by processing memory in larger chunks, improving efficiency during cleanup.

---

### Stages of Garbage Collection  

| **Stage**     | **Purpose**                                                      | **Granularity**         | **Details**                                                                                      |  
|----------------|------------------------------------------------------------------|-------------------------|--------------------------------------------------------------------------------------------------|  
| **Mark**       | Identify reachable (alive) objects and mark them.               | Object-level            | Incremental marking minimizes program impact; unmarked (white) objects are considered garbage.   |  
| **Atomic**     | Finalize marking and handle weak references.                    | Full execution (atomic) | Clears weakly referenced objects and ensures marking is consistent before sweeping.              |  
| **Sweep**      | Reclaim memory from unmarked (garbage) objects.                 | Page-level              | Frees memory occupied by unreachable objects, updating the heap for future GC cycles.           |  

By following these stages, Luau's garbage collector ensures efficient memory management with minimal interruptions to the program, making it well-suited for real-time and interactive environments.

---

## Detailed Workflow of Garbage Collection

Having explored the general concept of how Luau garbage collection works, let's now take an in-depth look at its implementation process.
This section will provide a comprehensive breakdown of the inner workings and mechanisms behind garbage collection.

---

### Mark Phase

The Mark Phase is a critical part of the garbage collection (GC) process in Luau, responsible for identifying reachable (alive) objects and ensuring that only those objects are retained in memory. During this phase, objects that are no longer reachable (unreferenced) are left unmarked, allowing them to be eventually collected and removed. The marking process is performed incrementally to minimize the impact on the program and avoid long pause times. Notably, during mark GC doesn't free any objects, and so the heap size will constantly grow.

#### Object Coloring Scheme

In order to efficiently track the status of objects during the marking process, a color-based system is used:

- **White**: Objects that have not been marked as reachable (garbage).
- **Gray**: Reachable objects that still have references to other unmarked objects.
- **Black**: Reachable objects that are fully processed (i.e., all references have been marked).

The tri-color invariant is fundamental to maintaining the correctness of the GC process:

- A black object must never reference a white object, ensuring that all live objects are correctly marked before they are processed.

> Technically, the system employs a four-color scheme. The "white" category is split into **white0** and **white1**, which distinguish between objects allocated during the current GC cycle and those from the previous cycle.

#### Incremental Marking

The marking phase progresses incrementally in small steps, minimizing the interruption to the program. This is achieved through the use of a **gray set**, which is a list of objects that are currently in the gray state. Each incremental step processes one gray object, marking it as black and updating its references as gray if they are white (i.e., unmarked). This ensures that the marking process progresses without overwhelming the system.

The gray set itself is maintained using an **intrusive singly linked list** (gclist), which allows efficient traversal and updating of the objects being processed.

> The gray set exists as gclist field in objects such as functions, tables, threads and protos.

#### Barriers to Maintain the Tri-Color Invariant

To preserve the tri-color invariant during the marking process, barriers are used to ensure correct handling of object references when they are mutated.

- **Forward Barriers**: These barriers immediately mark any newly referenced white objects as gray. An example of this is upvalue writes or the `setmetatable` function, which may create new references to previously unmarked objects.
  
- **Backward Barriers**: These barriers mark black objects as gray if their references change, and place them on a separate `grayagain` list, ensuring that the modified object is re-queued for re-marking. A typical example of this is table writes, where the contents of tables are updated and their references may need to be revisited during the marking phase.

#### Special Cases

The following objects have special handling during the mark phase:

- **Strings**: Strings are semantically black. They are marked as gray objects but are never added to the gray set. Once a string is marked as live, it is never processed again.
  
- **Threads/Coroutine**: Active threads are treated as gray during the marking phase, and their references are rescanned during the atomic phase. Inactive threads, however, are marked black once they have been fully processed, limiting the active rescans and reducing the overall workload during marking. API calls that modify a thread's stack ensure that the thread is marked as gray if necessary.

- **Upvalues**: Open upvalues, which are linked to a thread stack, are never marked as black. They are managed through a global list and processed during the atomic phase. If a thread is dead, its upvalues are closed and unlinked.

#### Two-Phase Marking

In Luau's incremental GC system, the Mark Phase is divided into two parts: the **First-Phase Mark** and the **Second-Phase Mark** (GCSpropagateagain).
After the first phase mark stage finishes traversing the `gray` list, we copy `grayagain` list to `gray` once and incrementally mark it again, emptying out the `gray` list the second time.

##### First-Phase Mark

- **Purpose**: The first-phase mark is the initial step in marking live objects. It begins by scanning all root objects (e.g., global variables, active threads) and marking all objects reachable from them as live.
- **Process**: The first-phase mark proceeds incrementally, ensuring that only objects that have not been modified since the last GC cycle are considered. This phase marks all objects reachable from the root set and does so in small chunks to avoid significant pauses in program execution.

##### Second-Phase Mark (GCSpropagateagain)

- **Purpose**: The second-phase mark, triggered by **GCSpropagateagain**, revisits objects that were modified after the first-phase mark was completed.
- **Process**: Objects that have been updated after the first-phase mark are reprocessed to ensure they are correctly recognized as live. This ensures that no live objects are missed due to modifications after the first-phase mark. The second-phase mark also proceeds incrementally, minimizing the impact on the program.

The second-phase mark reduces the need for rescanning modified objects in the atomic phase, thereby making the garbage collection process more efficient.

---

### Atomic Phase

The Atomic Phase is an essential part of the garbage collection (GC) process in Luau, ensuring that all marking operations are finalized and that no interruptions occur during critical operations. This phase guarantees the consistency of the marking process and handles the necessary cleanup of weak references, ensuring that memory is efficiently reclaimed. The atomic phase runs entirely before transitioning into the sweep phase, and its operations are performed without interference from the running program.

> In Roblox, the atomic phase is executed right after the script is done executing in the wait resume state. (Between Pre-Render and Pre-Animation)

#### Atomic Phase Purpose and Overview

- **Indivisible Operations**: The atomic phase performs operations that must occur without interruption, ensuring that the marking process is fully completed and consistent.
- **Finalization of Marking**: Any marking operations that need to be completed atomically, such as the processing of modified coroutine stacks and weak tables, are handled in this phase.
- **Thread Handling**: Active threads, which are gray during the marking phase, are rescanned in the atomic phase, while inactive threads are marked black and finalized.
- **Weak References**: Weak references and weak tables are processed, ensuring that unreachable keys and values are removed. This phase is crucial for maintaining the integrity of weak tables.

#### Key Operations in the Atomic Phase

1. **Completion of Marking and Consistency**
   - The atomic phase ensures that the marking process is complete and accurate. This includes processing any objects that were modified after the marking phase and ensuring that all references are correctly identified as either reachable (black) or unreachable (white).
   - The **tri-color invariant** is strictly maintained, meaning no black object can reference a white object. This ensures consistency throughout the garbage collection process.

2. **Weak References and Weak Tables**
   - Weak tables linked in special lists (`gclist`) are reassessed during the atomic phase.
   - **Unreachable entries** (those with white keys or values) are removed from weak tables, while entries with strong keys or values are retained.
   - **Shrinkable Weak Tables**: Weak tables can be marked as "shrinkable" by including the `s` flag in the `__mode` metafield. This allows weak tables to be resized to optimal capacity during GC, improving performance in cases with large object caches. However, this approach may result in missing keys during table iteration if the table is resized while iterating, so it should be used with care.

3. **Thread Handling**
   - **Active Threads**: Threads that are currently executing are rescanned in the atomic phase. This ensures that any modifications to the thread stack during marking are properly accounted for.
   - **Inactive Threads**: Once a thread becomes inactive, it is marked black. This indicates that the thread has been fully processed and its references are no longer being tracked.
   - **Thread Barriers**: API calls that modify the thread stack use thread barriers to check if the thread is black. If a thread is black, it is marked gray and placed on a list for rescanning during the atomic phase.

4. **Upvalue Handling**
   - **Open Upvalues**: These are upvalues that reference a stack slot in another thread. Open upvalues are never marked black because doing so would violate the GC invariant, as they might point to a stack slot in a dead thread.
   - Open upvalues are managed through a **global list** (`global_State::uvhead`), which is traversed during the atomic phase to ensure they are correctly handled.
   - If an upvalue points to a dead thread or an invalid location, it is closed, and its reference is unlinked.

5. **GC Coloring Scheme**
   - **Differentiating Allocations**: The GC uses a three-bit color encoding (`white0`, `white1`, and black) to track objects and differentiate between those allocated in the current or previous GC cycle, ensuring that the sweep phase only collects objects that are truly unreachable.
   - **White0 and White1**: Objects marked as white are considered unreachable and are flagged for collection. To differentiate between objects allocated during the current GC cycle and those remaining from the previous cycle, the GC uses two states: `white0` and `white1`.
   - **Gray Objects**: Gray objects are those that are still being processed and have all three bits unset. This allows the GC to track the "current" white state by flipping the white bit during the atomic stage, distinguishing between objects that were allocated in the previous GC cycle and those allocated in the current cycle.
   - **Exclusive Bit Encoding**: The three bits used to encode object colors are exclusive, meaning each object can only be one of the following:
     - **white0**: An object allocated in the current GC cycle that is still considered unreachable.
     - **white1**: An object allocated in a previous GC cycle that is still considered unreachable.
     - **black**: An object that is reachable and fully processed.
   - **Gray and Black Objects**: During the atomic phase, all objects that are gray are fully processed, and no gray objects should remain by the end of this phase. All reachable objects should be black, while unreachable objects should remain white.
6. **Reduced GC Pauses and Optimizations**
   - **Optimized Atomic Steps**: To minimize GC pause times, the atomic phase has been optimized to handle only the necessary operations. The **remarking phase** reduces the need for rescanning modified objects, and **coroutine incremental marking** limits active coroutine rescans.
   - **Efficient Table Shrinking**: Shrinkable tables (`"s"` flag in `__mode`) help manage object caches more efficiently, reducing memory overhead and improving performance.

---

### Sweep Phase

The **Sweep Phase** of garbage collection (GC) is responsible for reclaiming memory used by unreachable (dead) objects and preparing the heap for the next collection cycle. It follows the marking process, where objects are identified as either reachable or unreachable. In this phase, unreachable objects are freed, and live objects are updated for subsequent GC cycles. The sweep phase operates incrementally and uses a highly efficient **paged sweeper** for optimal performance.

#### Sweep Phase Purpose and Overview

- **Memory Reclamation**: The primary goal of the sweep phase is to free memory occupied by objects that were marked as unreachable during the marking phase.
- **Efficient Traversal**: Sweeping operates incrementally at the page level, ensuring that memory reclamation is done efficiently without excessive overhead.
- **Paged Sweeper**: Objects are allocated in 16 KB pages, and the sweeping process works at the granularity of these pages. This approach improves memory locality and reduces metadata overhead compared to linked-list-based sweeping methods.

#### Key Operations in the Sweep Phase

1. **Reclaiming Unreachable Objects**
   - Objects that remain unmarked (i.e., white) after the marking phase are considered unreachable and are therefore freed during the sweep phase.
   - The sweep phase does not perform marking operations, so it cannot immediately reclaim objects that become unreachable after sweeping starts. The atomic phase ensures that all marking operations are completed before sweeping begins, ensuring that only truly unreachable objects are collected.

2. **Efficient Memory Traversal with Paged Sweeper**
   - **Paged Sweeper**: The sweep phase uses a paged sweeper, which allocates objects in 16 KB [pages](LuauGarbageCollectorInDepth.md#luau-heap-structure-and-memory-allocation). This method improves memory locality by ensuring that consecutively swept objects are contiguous in memory. It also eliminates the need for sweep-related metadata on individual objects, reducing memory overhead.
   - **Granularity of Sweeping**: Instead of sweeping objects individually, the sweep phase operates at the granularity of entire pages. Each incremental step of sweeping processes one page at a time, freeing unreachable objects within that page and updating live objects for the next GC cycle.
   - **Improved Performance**: Compared to the linked-list-based sweeping used in Lua and LuaJIT, the paged sweeper is 2-3 times faster, making sweeping more efficient, especially for large heaps.

3. **Coloring Scheme During Sweeping**
   - **Four-Color Scheme**: Objects are allocated as white, but their state is tracked using a three-bit color encoding inside the `GCheader::marked`. These bits represent the following:
     - **white0**: An object allocated in the current GC cycle that is still considered unreachable.
     - **white1**: An object allocated in a previous GC cycle that is still considered unreachable.
     - **black**: An object that is reachable and fully processed.
     - **gray**: An object that is in the process of being marked and is yet to be fully scanned.
   - **Exclusive Bit Encoding**: All color bits are exclusive, meaning an object can only be marked as one of these states at any given time. Gray objects have all three bits unset, indicating they are still being processed. The "current" white bit is flipped during the atomic stage to differentiate between objects that were allocated in the previous cycle and those allocated in the current cycle.
   - **Recoloring for Next Cycle**: During the sweep phase, objects that were marked as white in the previous mark phase are freed. Any other objects are recolored with the current white bit, preparing them for the next marking cycle.

4. **Incremental Sweeping**
   - **Incremental Granularity**: Sweeping is performed incrementally at the page level, which means each step of the sweep phase processes one page of objects at a time. This approach allows the garbage collector to gradually reclaim memory without causing significant pauses, especially for large heaps.
   - **Minimizing GC Pause Times**: By working at the page level, the sweep phase minimizes the duration of GC pauses, ensuring that the system remains responsive while performing memory reclamation. This incremental approach helps to balance performance and memory management efficiency.

5. **Impact on Object Lifespan and Allocation**
   - **Live Objects**: During the sweep phase, only live objects—those marked as black—are retained in memory. All unreachable objects (white) are freed.
   - **Allocations During Sweep**: Any new objects allocated during the sweep phase are considered live and are not swept. This ensures that the heap is always in a consistent state when moving to the next GC cycle.

6. **Barriers and Proactive Marking**
   - **Barriers**: Some barriers may trigger during sweeping, especially when reachable objects are still marked as black. These barriers ensure that the GC maintains its invariants and avoids excessive marking overhead.
   - **Proactive Marking**: To avoid unnecessary barriers from triggering, some objects that are reachable but not yet scanned are proactively marked as white during sweeping. This helps to minimize the amount of work required in subsequent steps of the GC process.

---

### End of GC Cycle

Once sweeping is complete, the heap is cleaned, and memory is reclaimed. The GC cycle ends, and the program can continue execution.

---

## **GC Phases Summary:**

| **Phase**                          | **Description**                                                                                      | **Purpose**                                                                                          | **Actions**                                                                                         |
|------------------------------------|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| **Mark Phase (First Phase)**       | Marks all objects reachable from the root set (globals, active threads, etc.).                      | Identify all live objects in the heap.                                                               | Traverses root objects and marks all reachable objects as live.                                         |
| **Mark Phase (Second Phase)** | Revisits and marks objects that were modified after the first phase.                                | Ensure objects modified after the first mark phase are correctly marked as live.                     | Re-traverses modified objects to ensure they are correctly marked as live.                               |
| **Atomic Phase**                   | Completes operations that must happen without interruption, like finalizing marked objects.         | Perform bookkeeping and finalize the marking process without mutator interference.                   | May involve traversing coroutine stacks and other structures that need atomic operations.               |
| **Sweep Phase**                    | Reclaims memory used by dead objects and performs minor fixups for live objects.                    | Collect and clean up memory by removing unreachable objects and fixing live objects.                | Traverses the heap to reclaim memory from unmarked objects and fix up live objects.                      |
| **End of GC Cycle**                | The heap is cleaned, and memory is reclaimed.                                                        | Complete the garbage collection cycle.                                                               | The program can resume, and memory is efficiently managed.                                               |

These steps occur incrementally, with the GC running in small chunks to avoid long pauses.
The GC assists run during the program’s execution to keep things smooth, and the pacing algorithm
ensures that memory management does not interfere with program performance.

---

## Garbage Collection Settings

In Luau, garbage collection (GC) is managed with a set of tunable parameters that allow fine-grained control over the garbage collector's behavior. These settings enable the GC to balance performance and memory management by adapting to the allocation patterns of the application. The key settings revolve around the **GC pacer**, which helps ensure that the GC can keep up with the application's memory allocation while minimizing overhead. The key settings related to GC pacing are the **GC pacer**, **GC goal**, **step size**, and **GC multiplier**. Each of these settings plays a role in how the GC operates in relation to the application’s memory allocation.

---

### GC Pacer

The **GC pacer** algorithm is designed to keep the garbage collector in sync with the application’s memory allocation. Its goal is to ensure that the GC can "catch up" to the application allocating garbage, but without putting too much strain on the system. The pacer works by triggering GC steps when certain thresholds are reached, allowing the garbage collector to run incrementally and avoid long pauses. This is achieved by adjusting the GC settings to control when and how frequently the GC steps occur.

To configure the pacer in Luau, three variables are primarily used:

1. **GC Goal**
   - The **GC goal** determines the target heap size during the atomic phase in relation to the size of live objects. Specifically, it is defined as the worst-case heap size that the garbage collector will allow, based on the size of the live heap.
   - For example, a **200% GC goal** means that the heap’s maximum size during garbage collection may be twice the size of the live objects (i.e., the total memory used by objects that are still reachable).
   - The GC goal helps manage the overall memory footprint of the application during GC cycles. By setting a GC goal, developers can control how much overhead the GC can add to the live heap before it runs again. This setting ensures that the system doesn’t run out of memory while still performing garbage collection.

2. **Step Size**
   - The **step size** defines how much memory (in kilobytes) the application must allocate before a GC step is triggered. This acts as a threshold that dictates when the garbage collector should step in and start cleaning up memory.
   - By adjusting the step size, developers can control the frequency of garbage collection. A smaller step size may trigger more frequent GC cycles, which can help keep the heap size under control but may also introduce more frequent pauses. On the other hand, a larger step size can reduce the frequency of GC cycles but may result in longer pauses when GC does occur.
   - The step size setting allows fine-tuning of GC behavior based on the application's memory allocation patterns.

3. **GC Multiplier**
   - The **GC multiplier** controls how much the garbage collector tries to mark relative to how much the application has allocated. It adjusts the aggressiveness of the GC in relation to the application’s memory allocation rate.
   - A higher GC multiplier means that the GC will try harder to keep up with the application’s allocation rate. This is important for preventing the heap from growing too large, ensuring that the GC can catch up with the allocation without causing excessive pauses.
   - It is critical that the GC multiplier is set significantly above 1. This allows the garbage collector to effectively “pace” itself and perform garbage collection at a rate that matches the application’s allocation, helping to avoid delays and memory buildup.
   - The GC multiplier and GC goal are tightly linked, with their interaction influencing how quickly the garbage collector will run and how much memory it will allow the application to consume before it takes action.

> The exact behavior of these settings is detailed in the `lua.h` (see reference) comments for `LUA_GCSETGOAL`.

---

> Default settings for GC tunables (settable via lua_gc)
>
> - LUAI_GCGOAL = 200    // 200% (allow heap to double compared to live heap size)
> - LUAI_GCSTEPMUL = 200 // GC runs 'twice the speed' of memory allocation
> - LUAI_GCSTEPSIZE = 1  // GC runs every KB of memory allocation

---

### Garbage Collection Operations

Luau provides several operations to control the behavior of the garbage collector. These operations allow the user to stop, restart, or tune the GC, as well as initiate manual garbage collection steps.

- **LUA_GCSETGOAL**: Tunes the GC goal, which is the ratio between the total heap size and the amount of live data. For more details, see the previous section.
  
- **LUA_GCSETSTEPMUL**: Tunes the step multiplier, which determines how much GC work is done relative to the application's allocation rate. For more details, see the previous section.

- **LUA_GCSETSTEPSIZE**: Tunes the step size in kilobytes. This is the amount of memory that needs to be allocated before the GC step is triggered. For more details, see the previous section.

- **LUA_GCSTOP**: Stops the incremental garbage collection process.
- **LUA_GCRESTART**: Restarts the incremental garbage collection process.
- **LUA_GCCOLLECT**: Runs a full garbage collection cycle. This operation is not recommended for latency-sensitive applications as it may cause longer pauses.
- **LUA_GCCOUNT**: Returns the heap size in kilobytes and the remainder in bytes.
- **LUA_GCCOUNTB**: Returns the heap size in bytes.
- **LUA_GCISRUNNING**: Returns 1 if the GC is currently running, though this does not guarantee that the GC is actively collecting.
- **LUA_GCSTEP**: Performs an explicit GC step with the specified step size in kilobytes. This allows the user to manually trigger incremental garbage collection at specific points in the application to help control when GC work is performed. It can also be used to offset the need for automatic GC assists, which are based on the allocation rate.

The function `lua_gc` can be used to trigger these operations, and it is accessible through the `lapi.h` file.  
To allow `collectgarbage` to accept more options, `lua_collectgarbage` in `Repl.cpp` can be modified to handle additional arguments.

The default argument for `collectgarbage` is `"count"`.

In the default Luau implementation, the global `collectgarbage` can be called with the argument `"collect"` to trigger `LUA_GCCOLLECT`, which runs a garbage collection cycle. The argument `"count"` can also be passed to `collectgarbage`, which behaves the same as `gcinfo`, returning the heap size in kilobytes by calling `LUA_GCCOUNT`.

In Roblox, the global `collectgarbage` can only be called with the `"count"` option to get the heap size in kilobytes. The global `gcinfo` function behaves the same as `collectgarbage("count")`.

---

### Recommended Settings

The `LUA_GCSETSTEPMUL` and `LUA_GCSETGOAL` settings are intricately linked. It is recommended to set the step multiplier **S** within the interval `[100 / (G - 100), 100 + 100 / (G - 100))]` with a minimum value of 150%, where **G** is the GC goal (the ratio of heap size to live data size). For example:

- If **G = 200%**, the recommended step multiplier **S** should be in the interval `[150%, 200%]`.
- If **G = 150%**, the recommended **S** should be in the interval `[200%, 300%]`.
- If **G = 125%**, the recommended **S** should be in the interval `[400%, 500%]`.

These recommended settings help ensure that the GC can keep pace with the application's allocation rate and minimize overhead.

---

## Additional Information

### Luau Heap Structure, Memory Allocation, and Fragmentation

Luau employs a size-segregated page structure for its heap management, using both individual pages and system heap allocations to optimize memory usage, performance, and reduce fragmentation. By combining a size-segregated page structure with a progressive size class system, Luau balances performance, memory efficiency, and minimal overhead, ensuring efficient handling of both garbage-collected objects (GCO) and regular allocations.

---

### Heap Structure and Allocation Strategy

#### **System Heap Allocation with `frealloc` Callback**  

The `frealloc` callback serves as the central mechanism for memory allocation, resizing, and deallocation in Luau. Its interface is defined as:  

```c
void* frealloc(void* ud, void* ptr, size_t oldsize, size_t newsize);
```  

**Key Features of `frealloc`:**  

- Allocates a new memory block: `frealloc(ud, NULL, 0, x)` creates a block of size `x`.  
- Frees a memory block: `frealloc(ud, p, x, 0)` frees the block `p` and must return `NULL`.  
- Resizes a memory block: Adjusting the size is guaranteed to succeed if the new size is equal to or smaller than the old size.  
- Returns `NULL` when memory allocation or resizing fails.  

This callback provides a flexible, albeit slower, fallback for managing memory that doesn’t fit neatly into predefined page structures.

---

#### **Types of Allocations**  

1. **Garbage-Collected Objects (GCO):**  
   These objects are managed by the garbage collector and are allocated within memory pages for efficient sweeping and reclamation.  

2. **Regular Allocations:**  
   These include arrays and other non-GC-managed structures. Regular allocations are handled differently, allowing them to be freed or resized in isolation.

---

### **Heap Layout and Allocation Strategy**

#### **Garbage-Collected Objects (GCO)**  

1. **Page-Based Allocation:**  
   - GCOs are allocated in fixed-size pages (~16 KB).  
   - Each page contains blocks of the same size, maximizing space utilization.  
   - Blocks start with a GC header (`GCheader`), storing metadata like object type, mark bits, and GC state.  

2. **Large GCOs:**  
   - Objects exceeding 512 bytes are allocated in dedicated "small pages" containing a single block.  
   - This approach sacrifices some memory efficiency but maintains uniformity in page management.  

3. **Page Lists:**  
   - **Global list (`global_State::allgcopages`):** Links all GCO pages for sweeping.  
   - **Free list (`global_State::freegcopages`):** Tracks pages with at least one free block, ensuring O(1) allocation.  

4. **Incremental Sweeping:**  
   - GCO blocks are swept by processing entire pages, with freed blocks retaining their headers for metadata access during sweeping.  

---

#### **Regular Allocations**  

1. **Metadata and Isolation:**  
   - Regular allocations are prefixed with metadata, including pointers to the page and the next free block within the page.  
   - Unlike GCOs, these allocations don’t begin with a GC header, allowing isolation and direct deallocation.  

2. **Large Allocations:**  
   - Objects exceeding 512 bytes bypass page allocation and are directly allocated using the `frealloc` system heap.  

3. **Page Free List:**  
   - The `global_State::freepages` tracks pages with free blocks, enabling efficient allocation without requiring a global list traversal.  

---

### **Free List Management**
- **Per-Page Free Lists**:
  - Each page maintains a free list (`freeList`) of deallocated blocks. This ensures efficient reuse of freed memory for future allocations of the same size class.
  - Pages also use a bump pointer (`freeNext`) for initial allocations, reducing setup overhead and fragmentation.

- **Global Free Lists**:
  - Free pages for both GCO and non-GCO blocks are tracked using global free lists (`freegcopages` and `freepages`), ensuring O(1) allocation for pages with available space.

---

### **Memory Management Techniques**

1. **Size Classes:**  
   - Block sizes are rounded up into predefined size classes to optimize page usage and reduce fragmentation.  
   - Size classes are configured using the `SizeClassConfig`, ensuring balanced memory allocation.  

2. **Per-Page Allocation:**  
   - **Bump Pointer Allocation:** Allocates blocks sequentially within a page.  
   - **Free List Reuse:** Reuses blocks freed within the page, combining fast allocation with minimal overhead.  

3. **Page Deallocation:**  
   - When all blocks in a page are freed, the page is immediately deallocated using `frealloc`.  
   - While this minimizes memory retention, it may lead to excessive allocation traffic. Future enhancements could include a page cache to mitigate this issue.

---

### **Memory Fragmentation**

Memory fragmentation occurs when free memory is divided into small, non-contiguous blocks, making it difficult to allocate large blocks of memory even though sufficient total free memory exists. Luau's memory management system incorporates various strategies, including a **size class system**, to reduce fragmentation and maintain efficient memory usage.

---

### **Fragmentation Management**

Fragmentation is an inherent challenge in memory allocation. Luau addresses this through several techniques:

1. **Minimizing Internal Fragmentation**:
   - Uniform block sizes within pages ensure minimal unused space within allocated memory blocks.

2. **Reducing External Fragmentation**:
   - The progressive size class system groups similar allocation sizes, reducing the likelihood of non-contiguous free memory.

3. **Large Allocations**:
   - Blocks >512 bytes are isolated in dedicated pages or allocated directly, preventing fragmentation of smaller pages.

4. **Free List Reuse**:
   - Freed blocks are retained in per-page free lists for quick reuse, reducing allocation churn.

---


### **Size Class System**
Luau's size classes are defined using a progressive scheme to balance internal and external fragmentation:

- **Progressive Alignment**:
  - Block sizes are aligned to multiples of 8 bytes, satisfying pointer alignment requirements and improving memory efficiency.
  - The alignment increases progressively for larger sizes:
    - **8-byte increments** for block sizes between 8 and 64 bytes.
    - **16-byte increments** for block sizes between 64 and 256 bytes.
    - **32-byte increments** for block sizes between 256 and 512 bytes.
    - **64-byte increments** for block sizes between 512 and 1024 bytes.
  - For example, a requested block of 70 bytes is rounded up to 80 bytes to fit within the nearest size class.

- **Lookup Optimization**:
  - The `classForSize` array maps block sizes directly to their corresponding size class, ensuring O(1) lookup for allocation.
  - Gaps in the size lookup table are filled to ensure all block sizes map to the nearest larger size class.

This design reduces external fragmentation by ensuring that memory is allocated in predictable, well-distributed size classes, preventing unused gaps from forming between allocations.

---

### Fragmentation Challenges
- **Inter-Page Fragmentation**:
  - Fragmentation between pages can occur when allocation patterns result in uneven usage across pages of different size classes.

- **Large Allocations**:
  - Dedicated pages for large blocks introduce a slight overhead due to the unused space in these pages and the additional page header.
  
- **Allocation Traffic**:
  - Frequent deallocations and reallocations may increase traffic to the system heap. A **page caching system** could mitigate this by retaining recently freed pages.

---

## Summary

This guide provides an in-depth exploration of Luau's garbage collection system, emphasizing its
architecture, goals, and the challenges it overcomes to ensure efficient memory management. By
understanding these aspects, developers can fine-tune their applications for optimal performance and
memory management within the Luau runtime. This guide serves as a comprehensive resource for mastering
Luau's garbage collection, enabling developers to create efficient and responsive applications.  

If you would like to challenge yourself with what you learned in this guide, here is a [quiz](../Quiz/LuauGarbageCollectionQuizPart1.md) you can take.

## Reference  

1. **Garbage Collector Implementation**  
   - [lgc.cpp - GC implementation in Luau](https://github.com/luau-lang/luau/blob/master/VM/src/lgc.cpp)  
   - [lgc.h - Header file for GC functions](https://github.com/luau-lang/luau/blob/9a102e2aff99ecaf2ad1e5ca59fc1c893d5e9b7c/VM/src/lgc.h)  
   - [Performance Guide for Luau](https://luau.org/performance)  

2. **Memory Management**  
   - [lmem.cpp - Memory allocator and management functions](https://github.com/luau-lang/luau/blob/9a102e2aff99ecaf2ad1e5ca59fc1c893d5e9b7c/VM/src/lmem.cpp)  

3. **API References**  
   - [lapi.cpp - Luau API implementation (L1056)](https://github.com/luau-lang/luau/blob/7d4033071abebe09971b410d362c00ffb3084afb/VM/src/lapi.cpp#L1056)  
   - [lua.h - API definitions (L249)](https://github.com/luau-lang/luau/blob/9a102e2aff99ecaf2ad1e5ca59fc1c893d5e9b7c/VM/include/lua.h#L249)  

4. **Task Scheduling and Profiling**  
   - [Roblox MicroProfiler Task Scheduler](https://create.roblox.com/docs/studio/microprofiler/task-scheduler)
