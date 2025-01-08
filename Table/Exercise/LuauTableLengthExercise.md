# Table's Length

---

Calculate the length of the following tables

```lua
local tbl = {
    [0] = "r",
    [1] = "u",
    [2] = "m",
    [3] = "i",
    [5] = "n",
    [8] = "e",
    [9] = nil,
    [10] = "is",
}

print(#tbl)
```

```lua
local tbl = {
    [0] = "r",
    [1] = "u",
    [2] = "m",
    [3] = "i",
    [5] = "n",
    [8] = "e",
    [9] = nil,
    [10] = "is",
    [11] = nil,
}

print(#tbl)
```

```lua
local tbl = {
    [0] = "r",
    [1] = "u",
    [2] = "m",
    [3] = "i",
    [5] = "n",
    [8] = "e",
    [10] = "is",
}

print(#tbl)
```

```lua
local tbl = {
    [0] = "r",
    [1] = "u",
    [2] = "m",
    [3] = "i",
    [5] = "n",
    [8] = "e",
    [10] = "is",
    [9] = nil,
}

print(#tbl)
```

---

Exercises from the docs

```lua
local tbl = {1, 2, nil, 4, nil, nil, nil, 8, 9}
print(#tbl)
```

```lua
local tbl = {1, 2, nil, 4, nil, 6, nil, 8}
```

```lua
local tbl = {1, 2, nil, 4, nil, 6, nil, 8, nil}
```

```lua
local tbl = {1, 2, nil, 4, nil, 6, nil, 8, [10] = 10, [9] = nil}
print(#tbl)
```

---

Solution can be verified by executing the code
