# Table's Length

---

Calculate the length of the following tables (Assume optimization level 1)

---

```lua
local tbl, n = {nil, two = 2, 3, [4] = 4, [5-3] = nil, 7, [3] = nil}
n, tbl[5] = #tbl, 5
print(#tbl, n)
```

```lua
local tbl, n = {[1] = nil, [2] = 2, [3] = nil, [4] = 4, five = 5}
n, tbl[4] = #tbl, nil
tbl[3], tbl[6] = 3, 6
print(#tbl, n)
```

```lua
local tbl = {[1] = 1, [2] = 2, [3] = nil, [4] = nil, [5] = 5, a = 6, b = 7, c = 8}
local sum = #tbl
tbl[7] = 7
sum = sum + #tbl
tbl[6] = 6
sum = sum + #tbl
tbl[5] = nil
sum = sum + #tbl
print(n)
```

---

Again, the solution can be verified by executing the code. As for the reasoning, ask hao...

Previous exercise: [Part 2](LuauTableLengthExercise2.md), [Part 1](LuauTableLengthExercise1.md)

Guides: [Basic overview](../Guide/LuauTableLengthOverview.md), [In-depth view](../Guide/LuauTableLengthInDepth.md)
