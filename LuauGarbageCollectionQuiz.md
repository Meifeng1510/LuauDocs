# Luau Garbage Collection Quiz

--- 

1. What is the primary purpose of garbage collection in programming languages?  

    a) To allocate more memory  
    b) To reclaim unused memory  
    c) To optimize CPU usage  
    d) To prevent file corruption  

2. In Luau, what type of garbage collector is employed?  
    a) Generational  
    b) Reference counting  
    c) Mark-and-Sweep  
    d) Mark-and-compact  

3. Which of the following colors represents reachable and fully processed objects in the Mark Phase?  
    a) White  
    b) Gray  
    c) Black  
    d) Green  

4. What is the default color assigned to objects during the Mark Phase in Luau?
    a) White
    b) Gray
    c) Black
    d) Green

5. What does the **atomic phase** ensure?
   a) Allocated memory is optimized  
   b) Marking operations are finalized and consistent  
   c) Weak objects are deferred  
   d) Objects are rearranged in memory, preventing fragmentation  

6. What does `collectgarbage()` do in Roblox?  
    a) Return heap size in bytes  
    b) Return heap size in kilobytes  
    c) Invoke garbage collection
    d) None of the above  

7. What is the purpose of the **paged sweeper** in the Sweep Phase?  
    a) To allocate memory efficiently  
    b) To improve memory locality and reduce metadata overhead  
    c) To mark objects as unreachable  
    d) To ensure the heap grows  

8. Which objects are never marked as black during garbage collection?  
    a) Strings  
    b) Open upvalues  
    c) Global variables  
    d) Active threads  

9. What is the significance of the **gray set** in the incremental marking process?  
    a) It stores objects marked for compaction  
    b) It represents unreachable objects  
    c) It tracks objects currently being processed  
    d) It determines the allocation threshold  

10. What is the data structure of the **gray set**?  
    a) An array  
    b) A linked list  
    c) A set  
    d) Hash map  

11. What does the **Mark Phase** do?  
   a) Reclaims memory of unreachable objects  
   b) Identifies and marks reachable (alive) objects  
   c) Relocates objects in memory  
   d) Handles weak references  

12. Which **phase** processes **weak references**?  
   a) Mark Phase  
   b) Sweep Phase  
   c) Atomic Phase  
   d) Allocation Phase  

13. What is memory fragmentation?  
   a) Allocating memory in large chunks  
   b) Scattering memory in non-contiguous blocks  
   c) Compacting memory blocks  
   d) Allocating memory for unreachable objects  

14. What is the primary purpose of the **Sweep Phase**?  
   a) Reclaim memory occupied by unreachable objects  
   b) Mark all reachable objects  
   c) Optimize CPU performance  
   d) Handle weak references  

15. What is the role of **forward barriers** in garbage collection?  
   a) Reclaim memory from unreachable objects  
   b) Mark newly referenced white objects as gray  
   c) Rescan black objects  
   d) Update references to relocated objects  

    *What is a key advantage of the **paged sweeper**?  
   a) Reduces memory fragmentation  
   b) Allocates objects in linked lists  
   c) Frees memory in 32 KB chunks  
   d) Improves memory locality and reduces overhead  

17. How does **incremental marking** minimize program disruption?  
   a) By relocating memory blocks  
   b) By marking objects in one large step  
   c) By marking objects in smaller steps over time  
   d) By avoiding memory allocation  

18. What is the main drawback of **reference counting**?  
   a) It cannot handle circular references  
   b) It requires object relocation  
   c) It uses excessive memory  
   d) It lacks weak table support  

19. What is the purpose of **escape analysis**?  
   a) To identify unreachable objects  
   b) To allocate objects on the stack instead of the heap  
   c) To improve heap fragmentation  
   d) To mark objects incrementally  

20. What is the primary benefit of Luau's **non-moving** garbage collector?  
    a) Improves weak table handling  
    b) Avoids the need to update references  
    c) Reduces incremental marking time  
    d) Increases allocation rates  

---

[Part 1 Solution](LuauGarbageCollectionQuizPart1Solution.md)
[Part 2](LuauGarbageCollectionQuizPart2.md)
