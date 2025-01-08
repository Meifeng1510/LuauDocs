## Part 2

1. **Which of the following best describes why Luau uses both forward and backward barriers in its garbage collection system?**  
   - A) Forward barriers are faster, while backward barriers are more thorough.  
   - B) Forward barriers advance GC progress, while backward barriers defer work for efficiency.  
   - C) Forward barriers are used only for weak tables, and backward barriers are used for strong references.  
   - D) Forward barriers are used during the sweeping phase, while backward barriers are used during the marking phase.  

2. **What is the purpose of the `grayagain` list in Luau's GC implementation?**  
   - A) To store objects that need to be recolored from gray to black during the atomic phase.  
   - B) To handle objects that were marked as gray but were modified after the initial marking phase.  
   - C) To keep track of weak table entries that need processing during the atomic phase.  
   - D) To optimize the sweeping process by deferring work to later steps.  

3. **How does the Luau garbage collector handle strings during incremental marking?**  
   - A) Strings are marked as black and added to the gray set.  
   - B) Strings are treated as semantically black but are never marked as part of a gray list.  
   - C) Strings are treated as weak references and processed during the atomic phase.  
   - D) Strings are marked as white and recolored during the sweeping phase.  

4. **Why are threads treated differently from other objects in Luau's GC system?**  
   - A) Threads cannot be marked as black due to stack slot monitoring limitations.  
   - B) Threads are always treated as weak references to avoid breaking the tri-color invariant.  
   - C) Threads are marked as gray during execution and black when inactive to optimize GC performance.  
   - D) Threads require special handling during sweeping to account for active stack slots.

### Text Questions

1. What are the two phases of marking during the Mark Phase in Luau, and how do they differ in purpose and execution?  

2. What is a paged sweeper and how do they operate?
   
3. What is a **shrinkable table**, how does it work with garbage collection in Luau, and what precautions should be taken when using it?
   
4. What is **GC pacer** algorithm and their function?

5. 

---

6. **What is the significance of using a **coloring scheme** in garbage collection? List all the colors used and their function.**  

7. **Explain the tri-color invariant used in Luau's garbage collector and how forward and backward barriers help maintain this invariant. Provide examples of scenarios where each type of barrier would be applied.**

8. **Describe the role of the atomic phase in Luau's garbage collection cycle. Why is this phase necessary, and what tasks are completed during this stage that cannot be done incrementally?**

9. **Luau's garbage collector uses a "gray set" to track incremental marking progress. Explain how this set is managed during the GC cycle and the significance of the second-phase mark (`GCSpropagateagain`) in maintaining efficiency.**

10. **Weak tables require special handling in Luau's garbage collector. Discuss the challenges these tables pose for incremental marking and explain how weak entries are processed during the atomic phase.**

11. **Upvalues in Luau can be open or closed. Detail the unique challenges open upvalues present to the garbage collector and the special handling they require during the atomic phase. How does Luau ensure that these upvalues are correctly marked or closed?**

---

[Part 2 Solution](LuauGarbageCollectionQuizPart2Solution.md)
[Part 1](LuauGarbageCollectionQuizPart1.md)
