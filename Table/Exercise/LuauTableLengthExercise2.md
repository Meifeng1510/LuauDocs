# Table's Length

---

Calculate the length of the following tables

---

```lua
local tbl = {[1] = 1, 2, nil, 4}
print(#tbl)
```

```lua
local tbl = {1, [3] = nil, 3, 2}
print(#tbl)
```

---

```lua
local tbl = {[1] = 1, [2] = nil, [3] = 3, [2] = nil}
print(#tbl)
```

```lua
local tbl = {[3] = 1, [1] = 1, [2] = nil, [4] = 4}
print(#tbl)
```

```lua
local tbl = {[3] = 1, [1] = 1, [2] = nil, four = 4}
print(#tbl)
```

---

```lua
local tbl = {[1] = nil, [2] = nil, [3] = 3, [4] = 4, five = 5}
print(#tbl)
```

```lua
local tbl = {[1] = nil, [2] = nil, [3] = nil, [4] = 4, [5] = nil, [6] = nil, ["7"] = 7}
print(#tbl)
```

---

```lua
local tbl = {[1] = nil, [2] = nil, [3] = 3, [4] = 4, five = 5}
print(#tbl)
```

```lua
local tbl = {[1] = nil, [2] = nil, [3] = nil, [4] = 4, [5] = nil, [6] = nil, ["7"] = 7}
print(#tbl)
```

---

```lua
local tbl = {1, [6] = 6, nil, [1] = 1, nil, [5] = 5, 6, [8] = 8}
print(#tbl)
```

---

```lua
local tbl, n = {nil, two = 2, 3, [4] = 4, [5-3] = nil, 7, [3] = nil}
tbl[5], n = tbl, #tbl
print(#tbl, n)
```

---

Again, the solution can be verified by executing the code. As for the reasoning, ask hao...

Guides: [Basic overview](../Guide/LuauTableLengthOverview.md), [In-depth view](../Guide/LuauTableLengthInDepth.md)
