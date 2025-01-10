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

Again, the solution can be verified by executing the code. As for the reasoning, ask hao...

Guides: [Basic overview](../Guide/LuauTableLengthOverview.md), [In-depth view](../Guide/LuauTableLengthInDepth.md)
