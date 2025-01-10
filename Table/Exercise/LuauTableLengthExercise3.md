# Table's Length

---

Calculate the length of the following tables

---

```lua
local tbl, n = {nil, two = 2, 3, [4] = 4, [5-3] = nil, 7, [3] = nil}
tbl[5], n = tbl, #tbl
print(#tbl, n)
```

---

Again, the solution can be verified by executing the code. As for the reasoning, ask hao...

Previous exercise: [Part 2](LuauTableLengthExercise2.md), [Part 1](LuauTableLengthExercise1.md)

Guides: [Basic overview](../Guide/LuauTableLengthOverview.md), [In-depth view](../Guide/LuauTableLengthInDepth.md)
