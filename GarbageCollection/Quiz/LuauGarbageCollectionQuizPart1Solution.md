# **Answer Key and Explanations**

---

### 1. What is the primary purpose of garbage collection in programming languages?

**Answer:** b) To reclaim unused memory

**Explanation:** Garbage collection ensures that memory no longer used by the program is reclaimed and made available for future allocations, preventing memory leaks.

**Reference:** [Overview of Garbage Collection](../Guide/LuauGarbageCollectorInDepth.md#overview-of-garbage-collection)

---

### 2. In Luau, what type of garbage collector is employed?

**Answer:** c) Mark-and-Sweep

**Explanation:** Luau uses a mark-and-sweep garbage collector that is incremental, non-generational, and non-moving, ensuring efficient memory management without relocating objects.

**Reference:** [The Luau Garbage Collector](../Guide/LuauGarbageCollectorInDepth.md#the-luau-garbage-collector)

---

### 3. Which of the following colors represents reachable and fully processed objects in the Mark Phase?

**Answer:** c) Black

**Explanation:** The color black is used to represent objects that are alive and fully processed during the Mark Phase in a tri-color marking system.

**Reference:** [Object Coloring Scheme](../Guide/LuauGarbageCollectorInDepth.md#object-coloring-scheme)

---

### 4. What is the default color assigned to objects during the Mark Phase in Luau?

**Answer:** a) White

**Explanation:** Objects are initially marked as white, indicating they are unmarked or potentially unreachable.

**Reference:** [Object Coloring Scheme](../Guide/LuauGarbageCollectorInDepth.md#object-coloring-scheme)

---

### 5. What does the **atomic phase** ensure?

**Answer:** b) Marking operations are finalized and consistent

**Explanation:** The atomic phase ensures that marking is complete and that weak references and related objects are handled before the Sweep Phase begins.

**Reference:** [Atomic Phase](../Guide/LuauGarbageCollectorInDepth.md#2-atomic-phase)

---

### 6. What does `collectgarbage()` do in Roblox?

**Answer:** b) Return heap size in kilobytes

**Explanation:** In Roblox, `collectgarbage` option defaults to `"count"`. `collectgarbage("count")` is used to return the current heap size in kilobytes.

**Reference:** [Garbage Collection Operations](../Guide/LuauGarbageCollectorInDepth.md#garbage-collection-operations)

---

### 7. What is the purpose of the **paged sweeper** in the Sweep Phase?

**Answer:** b) To improve memory locality and reduce metadata overhead

**Explanation:** The paged sweeper processes memory in larger chunks (pages), improving efficiency and reducing metadata requirements compared to individual object processing.

**Reference:** [Sweep Phase](../Guide/LuauGarbageCollectorInDepth.md#sweep-phase)

---

### 8. Which objects are never marked as black during garbage collection?

**Answer:** b) Open upvalues

**Explanation:** Open upvalues (connected to a thread's stack) are not marked as black to prevent violations of the tri-color invariant.

**Reference:** [Special Cases](../Guide/LuauGarbageCollectorInDepth.md#special-cases)

---

### 9. What is the significance of the **gray set** in the incremental marking process?

**Answer:** c) It tracks objects currently being processed

**Explanation:** The gray set is a list of objects still being processed during incremental marking, ensuring all references are traversed.

**Reference:** [Incremental Marking](../Guide/LuauGarbageCollectorInDepth.md#incremental-marking)

---

### 10. What is the data structure of the **gray set**?

**Answer:** b) A linked list

**Explanation:** In Luau, the gray set is implemented as an intrusive singly linked list (`gclist`) for efficient traversal and management.

**Reference:** [Incremental Marking](../Guide/LuauGarbageCollectorInDepth.md#incremental-marking)

---

### 11. What does the **Mark Phase** do?

**Answer:** b) Identifies and marks reachable (alive) objects

**Explanation:** The Mark Phase ensures that all objects still in use are identified and preserved, leaving unreachable objects for later cleanup.

**Reference:** [Mark Phase](../Guide/LuauGarbageCollectorInDepth.md#1-mark-phase)

---

### 12. Which **phase** processes **weak references**?

**Answer:** c) Atomic Phase

**Explanation:** Weak references and weak tables are finalized during the atomic phase to ensure consistency.

**Reference:** [Atomic Phase](../Guide/LuauGarbageCollectorInDepth.md#2-atomic-phase)

---

### 13. What is memory fragmentation?

**Answer:** b) Scattering memory in non-contiguous blocks

**Explanation:** Memory fragmentation occurs when free memory is scattered across non-contiguous regions, making allocation inefficient.

**Reference:** [Memory Fragmentation](../Guide/LuauGarbageCollectorInDepth.md#memory-fragmentation)

---

### 14. What is the primary purpose of the **Sweep Phase**?

**Answer:** a) Reclaim memory occupied by unreachable objects

**Explanation:** The Sweep Phase reclaims memory from objects marked as unreachable during the Mark Phase.

**Reference:** [Sweep Phase](../Guide/LuauGarbageCollectorInDepth.md#3-sweep-phase)

---

### 15. What is the role of **forward barriers** in garbage collection?

**Answer:** b) Mark newly referenced white objects as gray

**Explanation:** Forward barriers ensure that newly referenced white objects are re-marked to prevent collection during the current cycle.

**Reference:** [Barriers to Maintain the Tri-Color Invariant](../Guide/LuauGarbageCollectorInDepth.md#barriers-to-maintain-the-tri-color-invariant)

---

### 16. What is a key advantage of the **paged sweeper**?

**Answer:** d) Improves memory locality and reduces overhead

**Explanation:** The paged sweeper processes memory at the page level, enhancing efficiency and reducing the overhead of individual object tracking.

**Reference:** [Sweep Phase Purpose and Overview](../Guide/LuauGarbageCollectorInDepth.md#sweep-phase-purpose-and-overview)

---

### 17. **How does incremental marking minimize program disruption?**

**Answer:** c) By marking objects in smaller steps over time

**Explanation:** Incremental marking breaks the marking process into smaller steps to reduce noticeable pauses in program execution.

**Reference:** [The Luau Garbage Collector](../Guide/LuauGarbageCollectorInDepth.md#the-luau-garbage-collector)

---

### 18. What is the main drawback of **reference counting**?

**Answer:** a) It cannot handle circular references

**Explanation:** Reference counting fails when two objects reference each other but are no longer reachable, leading to memory leaks.

**Reference:** [Basic Garbage Collector Strategies](../Guide/LuauGarbageCollectorInDepth.md#basic-garbage-collector-strategies)

---

### 19. What is the purpose of **escape analysis**?

**Answer:** b) To allocate objects on the stack instead of the heap

**Explanation:** Escape analysis identifies objects that do not leave their scope, enabling stack allocation and reducing garbage collection workload.

**Reference:** [Basic Garbage Collector Strategies](../Guide/LuauGarbageCollectorInDepth.md#basic-garbage-collector-strategies)

---

### 20. What is the primary benefit of Luau's **non-moving** garbage collector?

**Answer:** b) Avoids the need to update references

**Explanation:** By not relocating objects during collection, the non-moving GC simplifies implementation and maintains reference integrity.

**Reference:** [The Luau Garbage Collector](../Guide/LuauGarbageCollectorInDepth.md#the-luau-garbage-collector)

---

[Part 1](LuauGarbageCollectionQuizPart1.md)

[Part 2](LuauGarbageCollectionQuizPart2.md)
