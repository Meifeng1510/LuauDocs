###### By me
# Basic guide to Calculating the Length of a Table in Luau 

This guide explains how to calculate the length of a table in Luau, focusing on how the length operator `#` works, the underlying mechanics, and how tables are structured internally.

---

## Introduction to Tables in Luau

In Luau, a table is a versatile data structure that can act as:
1. **An array**: Sequentially indexed with positive integers.
   ```lua
   local array = {1, 2, 3, 4}
   ```
2. **A dictionary** (hash portion): Indexed with for non-sequential indices or non-integer keys.
   ```lua
   local dictionary = {name = "Luau", version = 1.0}
   ```

Tables can also combine these two functionalities.

For example:
```lua
-- Array portion (sequential integers)
local arrayExample = {1, 2, 3, 4} 

-- Hash portion (non-sequential/non-integer keys)
local dictionaryExample = {
    name = "John",
    age = 25,
    [1.5] = "float key",
    ["key"] = "string key"
}

-- Mixed table using both portions
local mixedTable = {
    1, 2, 3,           -- Array portion
    name = "John",     -- Hash portion
    [5] = "five"       -- May belong to either portion
}
```

---

## The Length Operator (#)

The length operator `#` when used on a table returns the **length of the array portion**, assuming no `__len` metamethod is defined.

```lua
local tbl = {1, 2, 3}
print(#tbl) -- Output: 3
```

However, determining this length isn't as straightforward as it might seem.

### Understanding Array Boundaries

The *boundary* of an array is defined as a positive integer index `n` where:
- `array[n]` is non-nil
- `array[n+1] & tbl[n+1]` is nil

However, this doesn't mean the array must be contiguous (have no gaps). Consider:

```lua
local tbl = {1, 2, nil, 4}
print(#tbl) -- Outputs: 4
```

You might expect the length to be 2 since there's a nil at index 3, but it's actually 4. Why? The answer lies in how Luau count, allocates and manages table memory.

---

## How Luau Calculates the Array Length

Luau uses a binary search to efficiently find the boundary of the array. The process works as follows:
1. **Initial Check**: If the value at the largest allocated index (*`allocatedSize`) is non-`nil`, that is the boundary.
2. **Binary Search**: If the value at index *`allocatedSize` is `nil`, Luau searches for the last non-`nil` index by halving the search range iteratively. This reduces the time complexity to `O(log n)`. Keep in mind the source code is in C++, so the index will be offseted by 1 in Luau.

In addition, the length will be cached when it's first calculated.

\*`allocatedSize` will be explained in the next section *How Tables Are Stored Internally*

### Example 1: Contiguous Array
```lua
local tbl = {1, 2, 3}
print(#tbl) -- Output: 3
```

### Example 2: Non-Contiguous Array
```lua
-- Array size here is 8, how we calculate that will be explained in the next section
local tbl = {1, nil, 3, 4, nil, 6, 7, nil}
-- Binary search finds the last non-nil value at index 4.
print(#tbl) -- Output: 4

-- Process:
-- Initial Check:   allocatedSize = 8, and array[8] is nil. Binary search is triggered.

-- First Midpoint:  Check at index mid (⌊size / 2⌋ = ⌊8 / 2⌋ = 4)
--                  array[4] is nil (tbl[5] in Luau). 
--                  Search in the lower half (indices 1 to 3).

-- Second Midpoint: Check index mid (⌊4 / 2⌋ = 2)
--                  array[2] is non-nil (tbl[3] in Luau).
--                  Search in the upper half of the current range (indices 3 to 4).

-- Third Midpoint: Check index mid (2 + ⌊2 / 2⌋ = 3)
--                 array[3] is non-nil (tbl[4] in Luau)
--                 The next search size is now 0, so we stop here.

-- Thus, the length of the array is 4.
```

### Example 3: Non-Contiguous Array
```lua
local tbl = table.create(100)
-- Creates an array with allocated space for 100 elements, all initialized to `nil`.
tbl[100] = true
print(#tbl) -- Output: 100

-- Process:
-- Initial Check:   `allocatedSize` = 100, and array[`allocatedSize` - 1] = array[99] is non-nil.
--                   Return allocatedSize (100) as the boundary.
```

---

### Why Sequential Counting is Inefficient

If you were wondering why Luau don't just check index up until the first nil value, if Luau were to calculate the array length by sequentially counting indices, it would have to:
1. Start counting from index `1`
2. Check the value of each index.
3. Repeat this until the first `nil` value.

This method is computationally expensive, especially for large tables, as it scales linearly with the size of the array (`O(n)`).

---

## How Tables Are Stored Internally

Tables in Luau consist of two parts:
1. **Array Portion**: Stores values indexed by positive integers (if possible).
2. **Hash Portion**: Stores non-integer indices and indices that can't fit into the array.

Each portion has a certain amount of **allocated memory**, which determines how many values can be stored without reallocation. When memory is exhausted, Luau reallocates more memory to accommodate new values.

---

## How Table Memory Works

### Memory Allocation

Tables in Luau start with size 0 and grow as needed. When new elements are added, Luau:
1. Checks if there's enough space
2. Reallocates memory if needed
3. Calculates new sizes for both array and hash portions

The reallocation process is expensive, so Luau doesn't allocate exact sizes. Instead, it balances between:
- Avoiding frequent reallocations by allocating extra spaces for further values.
- Preventing excessive memory waste from over-allocation.

### The Rehash Process

When reallocation occurs, Luau's `rehash` function determines the optimal sizes through several steps:

1. **Counting Phase**
   - Counts elements that could be stored in the array portion. These will be referred to as a `candidate`. These indices are positive integers with non-`nil` values.
   - Groups counts into power-of-2 ranges (1, 2, 3-4, 5-8, 9-16, etc.)

2. **Size Computation**
   - Finds the largest power of 2 (n) where more than half the slots between 1 and n are used. This is known as the optimal array size.

3. **Size Adjustment**
   - Increment the size by 1 repeatedly until `tbl[size + 1]` is `nil`.  
   - This ensures the boundary invariant, where the value at `size + 1` must always be `nil`.  
   - The process accounts for elements that may shift between the hash and array portions, optimizing memory usage and performance.  
   - The adjusted array size will be the optimal size plus twice the additional size from the adjustment. For example, if the optimal size is 8 and the adjustment increases it to 10, the new array size will be 12.  
   - Once the adjustment is complete, repeat the process again without doubling the extra size.

## Calculating Table Length

Let's look at some examples to understand how length is calculated:

```lua
local tbl = {1, 2, nil, 4, nil, 6, nil, 8, 9}
```

In this case:

1. Finding array size
    - More than half the slots up to 8 are used (5 non-nil values), so that is the optimal size
    - Index 9 is non-nil, so the size is adjusted from 8 -> 9
    - Extra is equal to 1, so the new array size is 8 + 1 * 2 = 10 
    - Final size is 10

2. Finding array's length
    - Check index at `allocatedSize`, `tbl[10]` is nil, so binary check will be performed
    - Check at midpoint, `tbl[6]` is non-nil, search upperhalf next
    - Check at midpoint, `tbl[8]` is non-nil, search upperhalf next
    - Check at midpoint, `tbl[9]` is non-nil, next search size is 0 so that's the final result

3. Final result is 9


Another example:
```lua
local tbl = {1, 2, nil, 4, nil, 6}
```

Here:
1. Finding array size
    - More than half the slots up to 4 are used (3 non-nil values)
    - Index 5 is nil, so the size cannot be adjusted further
    - Final size is 4

2. Finding array's length
    - Check index at `allocatedSize`, `tbl[4]` is non-nil, so that is the boundary

3. Final result is 4

## Practice Exercises

Try determining the length of these tables:

```lua
-- Exercise 1
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

-- Exercise 2
local tbl = {
    [0] = "r",
    [1] = "u",
    [2] = "m",
    [3] = "i",
    [5] = "n",
    [8] = "e",
    [10] = "is",
}

-- Exercise 3
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
```

## Additional Info

### Table Size Limits

Note that Luau has a maximum table size limit:

- **`MAXBITS`**: A constant defining the maximum size of the array and hash portions (26 by default).
- The maximum size for array and hash is `1 << MAXBITS` (2^26, approximately 67 million elements).
- Attempting to exceed this limit will result in a "table overflow" error

```lua
table.create(2^26)     -- Works fine
table.create(2^26 + 1) -- Table overflow error
```

### When is rehash triggered

The function `rehash` (used for re-allocation) is triggered when `newkey` function is called and the following criterias are met:

- If the key we are currently inserting doesn't exist (the value at index is void, not nil)
    - This happens when you are inserting a key into the hash portion, and the key doesn't exist yet.
- And either
    - The key being inserted is an integer and is equal to `allocatedSize` + 1
    - or the hash portion is full

---

## Conclusion

Understanding how tables and the `#` operator work in Luau is crucial for writing efficient scripts. By knowing how the array boundary is determined and how memory allocation works, you can optimize your use of tables and avoid unexpected behavior.

For more details, you can explore the [Luau source code](https://github.com/luau-lang/luau/blob/master/VM/src/ltable.cpp) or check the [in-depth guide](LuauTableLengthIndepth.md).
