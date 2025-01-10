# Table's Length

---

Calculate the length of the following tables

---

```lua
local tbl = {1, 2, nil, 4}
print(#tbl)
```

```lua
local tbl = {1, nil, nil, 4, 5, nil, nil, 8}
print(#tbl)
```

---

```lua
local tbl = {[1] = nil, [2] = nil, [3] = 3}
print(#tbl)
```

```lua
local tbl = {[1] = 1, [2] = nil, [3] = 3}
print(#tbl)
```

---

```lua
local tbl = {[4] = 4, [3] = nil, [2] = 2, [1] = 1}
print(#tbl)
```

```lua
local tbl = {[8] = 8, [7] = nil, [6] = nil, [5] = 5, [4] = 4, [3] = nil, [2] = nil, [1] = 1}
print(#tbl)
```

---


```lua
local tbl = {1, 2, nil, 4, nil, nil, nil}
print(#tbl)
```

```lua
local tbl = {1, nil, 3, nil, 5, nil, 7, nil}
print(#tbl)
```

---

```lua
local tbl = {[5] = 5, [4] = nil, [3] = 3, [2] = 2, [1] = 1, [6] = 6, [7] = nil, [8] = 8, [9] = nil, [10] = 10}
print(#tbl)
```

```lua
local tbl = {[1] = 1, [10] = 10, [2] = 2, [3] = 3, [4] = nil, [5] = 5, [6] = 6, [7] = nil, [8] = 8, [9] = nil}
print(#tbl)
```

```lua
local tbl = {[1] = 1, [10] = 10, [13] = 13, [2] = 2, [3] = nil, [4] = 4, [5] = nil, [6] = 6, [7] = 7, [8] = nil, [9] = nil}
print(#tbl)
```

---

The solution can be verified by executing the code. As for the reasoning, ask hao...

More exercise: [Part 2](LuauTableLengthExercise2.md)

Guides: [Basic overview](../Guide/LuauTableLengthOverview.md), [In-depth view](../Guide/LuauTableLengthInDepth.md)
