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
2. **A dictionary** (hash portion): Indexed with non-sequential indices or non-integer keys.
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

You might expect the length to be 2 since there's a nil at index 3, but it's actually 4. Why? The answer lies in how Luau counts, allocates, and manages table memory.

---

## How Luau Calculates the Array Length

Luau uses a binary search to efficiently find the boundary of the array. The process works as follows:
1. **Initial Check**: If the value at the largest allocated index (*`allocatedSize`) is non-`nil`, that is the boundary.
2. **Binary Search**: If the value at index *`allocatedSize` is `nil`, Luau searches for the last non-`nil` index by halving the search range iteratively. This reduces the time complexity to `O(log n)`. Keep in mind the source code is in C++, so the index will be offset by 1 in Luau.

Note that, the boundary will be cached when it's calculated. If the table is modified after the initial length calculation,
there may be more logic involved in how the table boundary is determined, which will be covered in the [in-depth guide](LuauTableLengthInDepth.md).

> `allocatedSize` will be explained in the next section *How Tables Are Stored Internally*

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
-- Initial Check:   allocatedSize = 8, and array[7] (tbl[8] in Luau) is nil. A binary search is triggered.

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

### Example 3: Pre-Allocated Non-Contiguous Array
```lua
local tbl = table.create(100)
-- Creates an array with allocated space for 100 elements, all initialized to `nil`.
tbl[100] = true
print(#tbl) -- Output: 100

-- Process:
-- Initial Check:   `allocatedSize` = 100, and array[`allocatedSize` - 1] = array[99] is non-nil.
--                   Return `allocatedSize` (100) as the boundary.
```

---

### Why Sequential Counting is Inefficient

If you were wondering why Luau doesn't just check the index up until the first nil value, if Luau were to calculate the array length by sequentially counting indices, it would have to:
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

Depending on how the table is declared, the memory allocation process may differ:

#### Tables Declared as a List
When a table is declared as a list, such as:

```lua
local tbl = {1, 2, 3, 4}
```

- The **`setlist`** process determines the table's size.
- The array size is allocated to exactly match the number of elements in the list, which in this case is 4.
- To maintain the boundary invariant, the allocated size increases by 1 until the last value is `nil`.

> Note that, if the optimization level is above 0, all indices wrapped in brackets are positive integers ordered in incremental order, and no values are declared as a list,
> the table size will be exactly equal to the number of values, similar to what happens if we declare the table as a list.
> 
> More details can be found in the [in-depth guide](LuauTableLengthInDepth.md#internally-allocated-size-in-luau)

```lua
local tbl = {[5] = 5, [6] = 6, 1, 2, 3, 4}
```

In this case, due to boundary invariance, the allocated size will be 6.

#### Tables Declared with Indices
For tables declared using explicit indices, such as:

```lua
local tbl = {[4] = 4, [3] = 4, [3] = 2, [1] = 1}
```

- The **`rehash`** process is used to calculate the size of the table.
- Initially, a table starts with size 0 and grows dynamically as new elements are added.

When new elements are added, Luau:
1. Check if there's enough space
2. If not, calculate new sizes for both array and hash portions
3. Reallocates memory if needed

The reallocation process is expensive, so Luau doesn't allocate exact sizes. Instead, it balances between:
- Avoid frequent reallocations by allocating extra spaces for further values.
- Preventing excessive memory waste from over-allocation.

### The Rehash Process

When reallocation occurs, Luau's `rehash` function determines the optimal sizes through several steps:

1. **Counting Phase**
   - Counts elements that could be stored in the array portion. These will be referred to as a `candidate`. These indices are positive integers with non-`nil` values.

2. **Size Computation**
   - Finds the largest power of 2 index (n) where more than half the slots between 1 and n are used. This is known as the optimal array size.

3. **Size Adjustment**
   - Increment the size by 1 repeatedly until `tbl[size + 1]` is `nil`.  
   - This ensures the boundary invariant, where the value at `size + 1` must always be `nil`.  
   - The process accounts for elements that may shift between the hash and array portions, optimizing memory usage and performance.  
   - The adjusted array size will be the optimal size plus twice the additional size from the adjustment. For example, if the optimal size is 8 and the adjustment increases it to 10, the new array size will be 12.  
   - Once the adjustment is complete, repeat the process again without doubling the extra size.
  
Note that, the key which triggers `rehash` is treated as if it's already in the table with a non-`nil` value.

## Calculating Table Length

Let's look at some examples to understand how length is calculated:

```lua
local tbl = {[9] = 9, [8] = 8, [7] = nil, [6] = 6, [5] = nil, [4] = 4, [3] = nil, [2] = 2, [1] = 1}
```

In this case:

1. Finding array size
    - More than half the slots up to 8 are used (5 non-nil values), so that is the optimal size
    - Index 9 is non-nil, so the size is adjusted from 8 -> 9
    - Extra is equal to 1, so the new array size is 8 + 1 * 2 = 10 
    The final size is 10

2. Finding the array's length
    - Check index at `allocatedSize`, `tbl[10]` is nil, so binary check will be performed
    - Check at the midpoint, `tbl[6]` is non-nil, search upper half next
    - Check at the midpoint, `tbl[8]` is non-nil, search upper half next
    - Check at the  midpoint, `tbl[9]` is non-nil, and the next search size is 0 so that's the final result

3. The final result is 9


Another example:
```lua
local tbl = {1, 2, nil, 4, nil, 6, nil}
```

Here:
1. Finding array size
    - Since the table is declared as a list, the `allocatedSize` will be exactly equal to the number of elements, which is 7.

2. Finding the array's length
    - Check index at `allocatedSize`, `tbl[7]` is non-nil, so binary check will be performed
    - Check at the midpoint, `tbl[4]` is non-nil, search upper half next
    - Check at the midpoint, `tbl[6]` is non-nil, search upper half next
    - Check at the midpoint, `tbl[7]` is nil, and the next search size is 0 so 6 is the final result

3. The final result is 6

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

The function `rehash` (used for re-allocation) is triggered when `newkey` function is called and the following criteria are met:

- If the key we are currently inserting doesn't exist (the value at the index is void, not nil)
    - This happens when you are inserting a key into the hash portion, and the key doesn't exist yet.
- And either
    - The key being inserted is an integer and is equal to `allocatedSize` + 1
    - or the hash portion is full

---

## Conclusion

By knowing how the array boundary is determined and how memory allocation works, you can optimize your use of tables and avoid unexpected behavior.

For more details, you can explore the [Luau source code](https://github.com/luau-lang/luau/blob/master/VM/src/ltable.cpp) or check the [in-depth guide](LuauTableLengthInDepth.md).

[Exercises you can take](../Exercise/LuauTableLengthExercise.md)

## References

[ltable.cpp](https://github.com/luau-lang/luau/blob/master/VM/src/ltable.cpp)
[lvm.cpp](https://github.com/luau-lang/luau/blob/master/VM/src/lvmexecute.cpp)
