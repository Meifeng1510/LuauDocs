###### By me
# In-Depth Guide: Calculating the Length of a Table in Luau

This guide dives deeply into understanding how Luau calculates the length of a table. If you haven't read [Basic Guide to Calculating the Length of a Table in Luau](LuauTableLengthOverview.md), it's recommended to do so first, as it provides the foundational knowledge needed for this guide.

---

## Detailed Explanation of `getn`

This section focuses on `getn`, the function used internally to determine the length of a table in Luau when no `__len` metamethod is set.

### Key Notes:
- The source code is implemented in C++, where array indexing starts at 0.
- For clarity, this section uses 0-based indexing (C++). To translate to Luau's 1-based indexing, add 1 to the indices.

### `getn` Function Source Code (C++ Implementation):

```Cpp
int luaH_getn(Table* t)
{
    int boundary = getaboundary(t);

    if (boundary > 0)
    {
        if (!ttisnil(&t->array[t->sizearray - 1]) && t->node == dummynode)
            return t->sizearray; // fast-path: the end of the array in `t' already refers to a boundary
        if (boundary < t->sizearray && !ttisnil(&t->array[boundary - 1]) && ttisnil(&t->array[boundary]))
            return boundary; // fast-path: boundary already refers to a boundary in `t'

        int foundboundary = updateaboundary(t, boundary);
        if (foundboundary > 0)
            return foundboundary;
    }

    int j = t->sizearray;

    if (j > 0 && ttisnil(&t->array[j - 1]))
    {
        // "branchless" binary search from Array Layouts for Comparison-Based Searching, Paul Khuong, Pat Morin, 2017.
        // note that clang is cmov-shy on cmovs around memory operands, so it will compile this to a branchy loop.
        TValue* base = t->array;
        int rest = j;
        while (int half = rest >> 1)
        {
            base = ttisnil(&base[half]) ? base : base + half;
            rest -= half;
        }
        int boundary = !ttisnil(base) + int(base - t->array);
        maybesetaboundary(t, boundary);
        return boundary;
    }
    else
    {
        // validate boundary invariant
        LUAU_ASSERT(t->node == dummynode || ttisnil(luaH_getnum(t, j + 1)));
        return j;
    }
}
```

### Function Overview

The `getn` function follows a straightforward process:

1. **Retrieve Cached Boundary:**
   - The function first calls `getaboundary`, which retrieves the cached size of the table or the allocated size if it’s the first time `getn` is called on the table.

   Example:
   ```lua
   local tbl = table.create(100, true)
   tbl[100] = nil

   for i = 1, 98 do
      tbl[i] = nil
   end
   print(#tbl) -- 0, computed boundary from scratch with binary search.
   ```

   Since there are no valid boundary cached (the default `sizearray` is also an invalid boundary),
   the boundary will be computed from scratch with binary search.

   ```lua
   local tbl = table.create(100, true)
   tbl[100] = nil

   print(#tbl) -- 99
   for i = 1, 98 do
       tbl[i] = nil
   end
   print(#tbl) -- 99, as long as the cached boundary remains valid, it won't be recomputed.
   ```

   The first `getn` calls compute the boundary using binary search, then cache the result.

   The second `getn` calls, even though we removed most of the value,
   as long as the boundary remains valid, it won't be re-computed and instead return the cache result.

2. **Fast Path Validation:**
   - If `array[sizearray - 1]` is non-`nil` and the hash portion is empty, the allocated size is returned as the boundary (fast path).
   - If the `boundary` is under the allocated size and `array[boundary - 1]` is non-`nil` while `array[boundary]` is `nil`, the `boundary` is valid and returned.

3. **Update Boundary:**
   - If neither fast path applies, the function attempts to update the boundary using the `updateaboundary` function.

4. **Binary Search (Fallback):**
   - If the boundary cannot be updated, it is recomputed from scratch using a binary search to find the last non-`nil` element.

### Updating the Boundary

The `updateaboundary` function recalculates the boundary when it's invalid or needs adjustment:

```Cpp
static int updateaboundary(Table* t, int boundary)
{
    if (boundary < t->sizearray && ttisnil(&t->array[boundary - 1]))
    {
        if (boundary >= 2 && !ttisnil(&t->array[boundary - 2]))
        {
            maybesetaboundary(t, boundary - 1);
            return boundary - 1;
        }
    }
    else if (boundary + 1 < t->sizearray && !ttisnil(&t->array[boundary]) && ttisnil(&t->array[boundary + 1]))
    {
        maybesetaboundary(t, boundary + 1);
        return boundary + 1;
    }

    return 0;
}
```

#### Key Behavior:

- The current and new boundary must lower than `arraysize`
- If `array[boundary - 1]` is `nil`, but `array[boundary - 2]` is non-`nil`:
  - Update the boundary to `boundary - 1`.
- If `array[boundary]` is non-`nil` and `array[boundary + 1]` is `nil`:
  - Update the boundary to `boundary + 1`.

#### Detailed Explanation

1. **Boundary Adjustment When `nil` is Found:**
   - If `array[boundary - 1]` is `nil` but `array[boundary - 2]` is non-`nil`:
     - Update the boundary to `boundary - 1`.
   - Explanation (in Luau):
     - A `boundary` is an integer index such that `t[i]` is non-`nil` and `t[i+1]` is `nil`.
     - If `t[i]` is `nil`, check if `t[i-1]` is non-`nil` to determine if there is a valid boundary directly below.
     - Ensure `i-1` is greater than or equal to 1 to avoid invalid indices.
     - This approach allows efficient boundary updates without recomputing from scratch.

     Example:
     ```lua
     local tbl = table.create(100, true)
     tbl[100] = nil

     print(#tbl) -- 99
     for i = 1, 97 do
       tbl[i] = nil
     end
     print(#tbl) -- 99, boundary is still valid
     tbl[99] = nil -- boundary is now invalid
     print(#tbl) -- 98, there is a valid boundary directly below so updateaboundary from 99 -> 98
     ```
     Instead of recomputing, the function checks if `t[98]` is non-`nil` and determines the new boundary as `98`.

2. **Boundary Adjustment When non-`nil` is Found Above:**
   - If `array[boundary]` is non-`nil` and `array[boundary + 1]` is `nil`:
     - Update the boundary to `boundary + 1`.
   - This occurs when a new index is set directly above the current boundary.
   - Explanation (in Luau):
     - A `boundary` is an integer index such that `t[i]` is non-`nil` and `t[i+1]` is `nil`.
     - If `t[i]` is non-`nil`, check if `t[i+1]` is nil to determine if there is a valid boundary directly above
     - If so, update the boundary to `boundary + 1`

     Example:
     ```lua
     local tbl = table.create(100, true)
     tbl[100] = nil
     tbl[99] = nil

     print(#tbl) -- 98
     for i = 1, 97 do
         tbl[i] = nil
     end
     print(#tbl) -- 98, boundary is still valid
     tbl[99] = true -- boundary is now invalid
     print(#tbl) -- 99, there is a valid boundary directly above so updateaboundary from 98 -> 99
     ```
     Instead of recomputing, the function checks if `t[99]` is non-`nil` and if `t[100]` is `nil`.
     Since it is, the boundary will be moved to `99`.

     ```lua
     local tbl = table.create(100, true)
     tbl[100] = nil
     tbl[99] = nil
     tbl[98] = nil

     print(#tbl) -- 97
     for i = 1, 96 do
         tbl[i] = nil
     end
     print(#tbl) -- 97, boundary is still valid
     tbl[98] = true -- boundary is now invalid
     tbl[99] = true
     print(#tbl) -- 0, boundary directly above is not valid (tbl[99] is non-nil), compute new boundary with from scratch with binary search
     ```
     The function checks if `t[98]` is non-`nil` and if `t[99]` is `nil`.
     Since it isn't, `updateaboundary` fail and boundary will be recomputed from scratch with binary search.

3. **Fallback to Recompute from Scratch:**
   - If the boundary is invalid and cannot be updated, the function recomputes the boundary from scratch. The boundary is then cached for future use.
   - Steps:
     - If `array[allocatedsize - 1]` is non-`nil`, return `allocatedsize` as the boundary.
     - Otherwise, perform a binary search to determine the last non-`nil` value.

---

### Binary Search for the Last Non-`nil` Value

If the boundary cannot be updated directly, a binary search is performed to locate the last non-`nil` value in the array.
This method ensures efficiency in determining the array boundary.

#### Binary Search Logic:

```cpp
TValue* base = t->array;
int rest = j;
while (int half = rest >> 1)
{
    base = ttisnil(&base[half]) ? base : base + half;
    rest -= half;
}
```

- **Steps:**
  1. Start with `rest = array size`.
  2. Halve `rest` each iteration and check the midpoint (`base + half`).
  3. If the value at the midpoint is non-`nil`, adjust `base` by adding `half`.
  4. Repeat until `half == 0`.

This approach reduces the search space logarithmically, resulting in `O(log n)` complexity.

#### Detailed example

How Luau searches for the last non-`nil` value using binary search:

##### Example Workflow

Assume the array size is 8:
- **Initial Setup:**
  - `base = 0`
  - `rest = 8`

1. Compute `half = rest // 2 = 4`.
   - Check `tbl[base + 4]` (equivalent to `tbl[5]` in Luau).
   - If the value is `nil`, `base` remains the same.
   - If the value is non-`nil`, increment `base` by `half`.

2. Reduce `rest` by half.
   - `rest = 4`
   - Compute `half = rest // 2 = 2`.
   - Check `tbl[base + 2]` (equivalent to `tbl[base + 3]` in Luau).
   - Adjust `base` as needed.

3. Repeat until `half == 0`.
   - At each step, the search size is halved, allowing efficient narrowing down of the boundary.

4. Return `boundary + 1` (adjusted for Luau's 1-based indexing).

##### Length operation in Action

**Example 1:**
```lua
local tbl = {1, 2, nil, 4}
```
- Array size: 4 (more on why later).
- Since `array[allocatedSize - 1 (3)]` (equivalent to `tbl[4]` in Luau) is non-`nil`, the boundary is 3.
- Final result: `boundary + 1 = 4`.

**Example 2:**
```lua
local tbl = {1, nil, 3, 4, nil, 6, 7, nil}
```
- Array size: 8 (more on why later).
- Since `array[allocatedSize - 1 (7)]` (equivalent to `tbl[8]` in Luau) is `nil`, binary search begins:
  1. Check `array[4]` (equivalent to `tbl[5]`): `nil`.
  2. Check `array[2]` (equivalent to `tbl[3]`): non-`nil`.
  3. Check `array[3]` (equivalent to `tbl[4]`): non-`nil`.
  4. Boundary found at `3` (0-based indexing).

- Final result: `boundary + 1 = 4`.
- Verify by running:
  ```lua
  print(#tbl) -- Outputs: 4
  ```

## Understanding Allocated Size

Now that you understand how `getn` works, you may wonder: how is the allocated size determined?
While Luau provides a way to allocate the array portion explicitly:

```lua
local tbl = table.create(size)
```

If the table is created directly, such as:

```lua
local tbl = {1, 2, nil, 4}
```

there isn't an easy way to retrieve the internally allocated size. So, how do we know how much memory is allocated for the array portion?

---

### Internally Allocated Size in Luau

Let's first start by analyzing the behaviour of `table.create`, whe allocated size of the array portion can be observed:

```lua
local tbl = table.create(size)

print(#tbl) -- Output: 0 (initialized with nil values)
tbl[size] = true
print(#tbl) -- Output: size (boundary updated with non-nil value)
```

If a table is created directly (e.g., `local tbl = {1, 2, nil, 4}`), there is no direct way to retrieve the allocated size.

---

### How Luau Allocates Table Memory

When inserting values, Luau ensures sufficient space by reallocating the table. This process involves:

1. Counting valid indices in both the array and hash parts.
2. Calculating new sizes for each portion.
3. Resizing the table based on these calculations.

These steps ensure the table size is optimized to balance performance and memory usage, minimizing the frequency of reallocations while avoiding excessive memory waste.

These processes are done in order to compute the optimal size for the table, such that re-allocation would not occur frequently, and not too much memory is wasted.

#### Reallocation Process

In Luau, a table starts with an array size of 0. Every time a value is inserted:

1. Luau checks if there is enough space in the array or hash portion.
2. If there isn't enough space, or
   if memory can be optimized by relocating values from the hash portion to the array portion,
   the table is resized using the `rehash` function.

#### Array Size Calculation

During reallocation, the new array size `n` is determined such that:

- At least half of the slots between `1` and `n` are used.
- `tbl[n+1]` is `nil`.

This ensures efficient memory usage while maintaining performance.

---

### The `rehash` Function

To understand how the optimal size is calculated, let’s examine the `rehash` function, which is responsible for computing the new size of the table. The function dynamically adjusts the array and hash portions based on the distribution of elements in the table.

#### Example Code: Rehash Logic

```cpp
static void rehash(lua_State* L, Table* t, const TValue* ek)
{
    int nums[MAXBITS + 1]; // nums[i] = number of keys between 2^(i-1) and 2^i
    for (int i = 0; i <= MAXBITS; i++)
        nums[i] = 0;                          // reset counts
    int nasize = numusearray(t, nums);        // count keys in array part
    int totaluse = nasize;                    // all those keys are integer keys
    totaluse += numusehash(t, nums, &nasize); // count keys in hash part

    // count extra key
    if (ttisnumber(ek))
        nasize += countint(nvalue(ek), nums);
    totaluse++;

    // compute new size for array part
    int na = computesizes(nums, &nasize);
    int nh = totaluse - na;

    // enforce the boundary invariant; for performance, only do hash lookups if we must
    int nadjusted = adjustasize(t, nasize, ek);

    // count how many extra elements belong to array part instead of hash part
    int aextra = nadjusted - nasize;

    if (aextra != 0)
    {
        // we no longer need to store those extra array elements in hash part
        nh -= aextra;

        // because hash nodes are twice as large as array nodes, the memory we saved for hash parts can be used by array part
        // this follows the general sparse array part optimization where array is allocated when 50% occupation is reached
        nasize = nadjusted + aextra;

        // since the size was changed, it's again important to enforce the boundary invariant at the new size
        nasize = adjustasize(t, nasize, ek);
    }

    // resize the table to new computed sizes
    resize(L, t, nasize, nh);
}
```

This may seem daunting at first, but we will go through it step by step.
We will focus on the array part for now, as that’s the primary goal of this guide.

#### First Part: Counting

```cpp
    int nums[MAXBITS + 1]; // nums[i] = number of keys between 2^(i-1) and 2^i
    for (int i = 0; i <= MAXBITS; i++)
        nums[i] = 0;                          // reset counts
    int nasize = numusearray(t, nums);        // count keys in array part
```

In the first section of the `rehash` function, we count all the candidates for inclusion in the array portion.
A candidate for the array is an index such that it is a positive integer, and the value at that index is non-nil.

##### Initializing the Counting Array

The first step of counting is initializing a C++ array that will store the results. This array is initialized with a size of `MAXBITS + 1`.

- **`MAXBITS`**: A constant defining the maximum size of the array and hash portions (26 by default).
- This sets the maximum size of the table. The maximum array size is `1 << MAXBITS` (2^26, roughly 67 million).

```lua
-- Examples:
local tbl = table.create(2^26) -- Valid
local tbl = table.create(2^26 + 1) -- Table overflow
```

##### Why Initialize with `MAXBITS`?

The counting array is initialized to slice the table into exponentially increasing ranges (a power of 2). The `numusearray` function is responsible for this slicing.

- The array is divided into ranges:
  - `nums[0]` stores the number of candidates in the range `[1, 1]`.
  - `nums[1]` stores the number of candidates in the range `[2, 2]`.
  - `nums[2]` stores the number of candidates in the range `[3, 4]`.
  - `nums[3]` stores the number of candidates in the range `[5, 8]`.
  - `nums[4]` stores the number of candidates in the range `[9, 16]`.
  - ...
  - `nums[n]` stores the number of candidates in the range `[2^(n-1) + 1, 2^n]`.
  - This continues until `n = MAXBITS`.

This process is handled by the `numusearray` function.

```cpp
static int numusearray(const Table* t, int* nums)
{
    int lg;
    int ttlg;     // 2^lg
    int ause = 0; // summation of `nums'
    int i = 1;    // count to traverse all array keys
    for (lg = 0, ttlg = 1; lg <= MAXBITS; lg++, ttlg *= 2)
    {               // for each slice
        int lc = 0; // counter
        int lim = ttlg;
        if (lim > t->sizearray)
        {
            lim = t->sizearray; // adjust upper limit
            if (i > lim)
                break; // no more elements to count
        }
        // count elements in range (2^(lg-1), 2^lg]
        for (; i <= lim; i++)
        {
            if (!ttisnil(&t->array[i - 1]))
                lc++;
        }
        nums[lg] += lc;
        ause += lc;
    }
    return ause;
}
```

Keep in mind, `rehash` is triggered when the `newkey` function is called and certain criteria are met.
The key responsible for triggering `rehash` is also passed to the function and checked for candidacy.

Now that we’ve covered counting, let’s move on to the second part.

#### Second Part: Computing the Optimal Array Sizes

In the second part, we are determining the optimal array sizes using the `computesizes` function.

##### Definition of Optimal Size

The optimal size is defined as the largest number n such that more than half of the slots between 1 and n are used, where n is a power of 2 (i.e., n = 2^x, where x is an integer).

For example:

```lua
local tbl = {1, 2, nil, 4}
```
In this case, the optimal size is **4** because 4 is the largest n where more than half of the slots (3 out of 4) are used.
This explanation should be sufficient for most cases, but if you're interested in understanding how slices are counted, follow along with the detailed explanation below. Otherwise, you can skip to the third part.

##### How `computesizes` Works

```cpp
static int computesizes(int nums[], int* narray)
{
    int i;
    int twotoi; // 2^i
    int a = 0;  // number of elements smaller than 2^i
    int na = 0; // number of elements to go to array part
    int n = 0;  // optimal size for array part
    for (i = 0, twotoi = 1; twotoi / 2 < *narray; i++, twotoi *= 2)
    {
        if (nums[i] > 0)
        {
            a += nums[i];
            if (a > twotoi / 2)
            {               // more than half elements present?
                n = twotoi; // optimal size (till now)
                na = a;     // all elements smaller than n will go to array part
            }
        }
        if (a == *narray)
            break; // all elements already counted
    }
    *narray = n;
    LUAU_ASSERT(*narray / 2 <= na && na <= *narray);
    return na;
}
```

The `computesizes` function iterates through the `nums` array and tracks the cumulative sum. If the sum exceeds 2^i / 2, then more than half of the slots between 1 and 2^i are used, and this value is considered the current most optimal size.

The process continues until the sum equals the number of elements in the array (i.e., until all the numeric indices are accounted for).

#### Third Part: Adjusting the Array Size

After computing the optimal size using the `computesizes` function, the array size is not yet final. It still needs to be adjusted.

##### Adjusting the Array Size with the `adjustasize` Function

The array size is adjusted using the `adjustasize` function: 

```cpp
static int adjustasize(Table* t, int size, const TValue* ek)
{
    bool tbound = t->node != dummynode || size < t->sizearray;
    int ekindex = ek && ttisnumber(ek) ? arrayindex(nvalue(ek)) : -1;
    // move the array size up until the boundary is guaranteed to be inside the array part
    while (size + 1 == ekindex || (tbound && !ttisnil(luaH_getnum(t, size + 1))))
        size++;
    return size;
}
```

What this does is, if the value at index `size + 1` is non-nil, it increases the size by 1.
The process continues until `tbl[size + 1]` is nil.

##### Role of the Trigger Key

Remember the key that triggered the rehash is passed into the function.
Here it is used in the condition `size + 1 == ekindex`, meaning the size is adjusted until either `tbl[size + 1]` is nil
or `size + 1` does not equal the new key being inserted.

In other words, the function behaves as though the new key is already in the table, ensuring that the array size accommodates the new key.

##### Calculating the Adjusted Size

After obtaining the new size from `adjustasize`, we calculate the extra size gained from the adjustment:

```
extra = adjusted size - optimal size
```

If `extra` is 0, then the optimal size is the new array size.
Otherwise, the extra size came from indices which otherwise would go to the hash portion.

##### Memory Implications

Had we not adjusted the array size, the extra indices would have been placed in the hash portion.
Since the hash portion stores both keys and values, it takes up twice as much memory as the array portion.

Thus, the extra size represents the number of values we moved from the hash section to the array section.
And since the hash portion requires twice the memory of the array portion, for each extra value moved,
the array size will expand twice the amount.

##### Calculating the New Array Size

Instead of simply setting the new array size to the optimal size plus the extra size, we adjust the array size as follows:

```
new array size = optimal size + (extra * 2)
```

In the `rehash` portion, this is achieved by adding `adjusted size + extra`, which is equivalent to `optimal size + (extra * 2)`.

##### Enforcing the Boundary Invariant

After adjusting the array size, we need to ensure the value at `tbl[size + 1]` is still nil
(enforcing the boundary invariant that the value at `tbl[arraySize + 1]` must be nil).

To do this, we call `adjustasize` a second time. But this time, we add only the extra size (not the extra multiplied by 2) to ensure that `tbl[size]` is nil.
This works because `adjustasize` adjusts the size until `tbl[size + 1]` is nil.

Now, that will be the final adjustment to the array size, and `resize` will be called in order to re-allocate.

---

### Size of the Hash Portion in Lua Tables

In Luau, the size of the hash portion of a table depends on how the keys are distributed.

#### Key Calculations for Hash Portion
The computation starts with counting keys in the array and hash part and then determining the total usage:
```cpp
int nasize = numusearray(t, nums);        // Count keys in the array part
int totaluse = nasize;                    // Initialize with array keys
totaluse += numusehash(t, nums, &nasize); // Add hash part keys
```

Here:
- `nasize`: Number of integer keys in the array portion.
- `totaluse`: Total number of keys in the table.

Next, the optimal size of the array is computed,
which we will use to subtract `totaluse` for the total amount of key in hash portion
```cpp
// Compute the new size for the array part
int na = computesizes(nums, &nasize); // Optimal size for the array part
int nh = totaluse - na;               // Remaining keys for the hash part
```

#### Adjusting for Rehashed Keys
When rehashing:
```cpp
// Adjust the hash size based on keys moved from hash to array
nh -= aextra; // `aextra` is the number of keys moved to the array
```

The final size of the hash portion is:
```
nh = total keys - keys in the final array portion
```

This ensures that only the keys not fitting in the array part are retained in the hash part.

#### Alignment to Powers of 2
The hash size (`nh`) must be a power of 2. If `nh` is not a power of 2, it is rounded up to the next power:
```cpp
lsize = ceillog2(size);               // Calculate log2 size
if (lsize > MAXBITS)                 // Ensure it doesn't exceed limits
    luaG_runerror(L, "table overflow");
size = twoto(lsize);                  // Convert to 2^lsize
```

For example:
- If `nh = 3`, the hash size becomes `2^(ceil(log2(3))) = 4`.
- This maintains efficient lookup performance.

---

### Example Lua Table Behavior

```lua
local tbl = {1, 2, 3, 4, 5, 6, 7, 8} -- Array size is 8
tbl["a"] = true                      -- Add a hash key
tbl["b"] = true                      -- Add another hash key
tbl["c"] = true                      -- Add a third hash key
print(#tbl) -- Output: 8 (array size is unchanged)

-- Remove most array elements
for i = 1, 7 do
    tbl[i] = nil
end

print(#tbl) -- Output: 8 (rehash criteria not yet met)

-- Add more hash keys
tbl["d"] = true
print(#tbl) -- Output: 8 (still no rehash)

tbl["e"] = true
print(#tbl) -- Output: 0 (rehash triggered; array portion emptied)
```

---

### Explanation of the Example
1. **Initial State:**
   - `tbl` starts with an array size of 8.
   - Three hash keys are added (`"a"`, `"b"`, `"c"`), but the array size remains unchanged as the criteria for rehashing are not met.

2. **Removing Array Keys:**
   - Array elements are removed, but `#tbl` still reports `8` because rehashing hasn't occurred yet.

3. **Adding More Hash Keys:**
   - Adding the fourth and fifth hash keys (`"d"` and `"e"`) triggers a rehash because the hash portion becomes full.
   - The array portion is emptied during rehashing, and `#tbl` now reports `0`.

---

## Understanding Array Size Calculation and Adjustments in Luau

Now that we know how to calculate the array size and understand how `getn` works, let's look at some examples.
We will be working with Luau index.

### Example 1:

```lua
local tbl = {1, 2, nil, 4, nil, nil, nil, 8, 9}
```
#### Process:

##### Calculating Array Size

**Step 1: Counting**

We count the total number of indices:

- `nums[0] = 1` at index 1 (range is [1, 1])
- `nums[1] = 1` at index 2 (range is [2, 2])
- `nums[2] = 1` at index 3 (range is [3, 4])
- `nums[3] = 1` at index 5 (range is [5, 8])
- `nums[4] = 1` at index 9 (range is [9, 16])

Total indices: 5

**Step 2: Computing Optimal Array Sizes**

- At index 1, there are 1 index total, which is above ⌊1/2⌋ (0) so 1 is the current most optimal size
- From index 1 to 2, there are 2 index total, which is above ⌊2/2⌋ (1) so 2 is the current most optimal size
- From index 1 to 4, there are 3 index total, which is above ⌊4/2⌋ (2) so 4 is the current most optimal size
- From index 1 to 8, there are 4 index total, which isn't above ⌊8/2⌋ (4) so 4 is still the most optimal size
- From index 1 to 16, there are 5 index total, which isn't above ⌊16/2⌋ (8) so 4 is still the most optimal size
- Since we counted all the indice, we now have the highest optimal size

Thus, the final optimal size is 4.

**Step 3: Adjusting Array Size**

- The current size is 4.
- Check `tbl[5]`; since it is `nil`, the array size cannot be adjusted further.
- Final array size remains 4.

**Step 4: Final Size is 4**

##### Calculating Array Length

**Step 1:** Check index at `allocatedSize`, which is `tbl[4]`. Since the value is non-nil, it is the boundary.

**Step 2:** The final result for the array length is 4.

---

### Example 2:

```lua
local tbl = {1, 2, nil, 4, nil, 6, nil, 8}
```

#### Process:

##### Calculating Array Size

**Step 1: Counting**

We count the total number of indices:

- `nums[0] = 1` at index 1 (range is [1, 1])
- `nums[1] = 1` at index 2 (range is [2, 2])
- `nums[2] = 1` at index 4 (range is [3, 4])
- `nums[3] = 2` at index 6, 8 (range is [5, 8])

Total indices: 6

**Step 2: Computing Optimal Array Sizes**

- At index 1, there are 1 index total, which is above ⌊1/2⌋ (0) so 1 is the current most optimal size
- From index 1 to 2, there are 2 index total, which is above ⌊2/2⌋ (1) so 2 is the current most optimal size
- From index 1 to 4, there are 3 index total, which is above ⌊4/2⌋ (2) so 4 is the current most optimal size
- From index 1 to 8, there are 5 index total, which is above ⌊8/2⌋ (4) so 8 is the current most optimal size
- Since we counted all the indice, we now have the highest optimal size

Thus, the final optimal size is 4.

**Step 3: Adjusting Array Size**

- The current size is 8.
- Check `tbl[9]`; since it is `nil`, the array size cannot be adjusted further.
- Final array size remains 8.

**Step 4: Final Size is 8**

##### Calculating Array Length

**Step 1:** Check index at `allocatedSize`, which is `tbl[8]`. Since the value is non-nil, it is the boundary.

**Step 2:** The final result for the array length is 8.

---

### Example 3:

Let's change up a bit

```lua
local tbl = {1, 2, nil, 4, nil, 6, nil, 8, nil}
```

Normally, we have to check inserting key by key, but most of the time it can be skipped without affecting final result (like in the 2 examples above), we just skipped it.
For this example, let's assume we are inserting the ninth key (the final key, [9] = nil)

```lua
local tbl = {1, 2, nil, 4, nil, 6, nil, 8, nil}
```

##### Calculating Array Size

**Step 1: Counting**

We count the total number of indices:

- `nums[0] = 1` at index 1 (range is [1, 1])
- `nums[1] = 1` at index 2 (range is [2, 2])
- `nums[2] = 1` at index 4 (range is [3, 4])
- `nums[3] = 2` at index 6, 8 (range is [5, 8])
- `nums[4] = 1` at index 9 (the key we are inserting) (range is [9, 16])

Total indices: 6

**Step 2: Computing Optimal Array Sizes**

- At index 1, there are 1 index total, which is above ⌊1/2⌋ (0) so 1 is the current most optimal size
- From index 1 to 2, there are 2 index total, which is above ⌊2/2⌋ (1) so 2 is the current most optimal size
- From index 1 to 4, there are 3 index total, which is above ⌊4/2⌋ (2) so 4 is the current most optimal size
- From index 1 to 8, there are 5 index total, which is above ⌊8/2⌋ (4) so 8 is the current most optimal size
- From index 1 to 16, there are 6 index total, which isn't above ⌊16/2⌋ (8) so 8 is is still the most optimal size
- Since we counted all the indice, we now have the highest optimal size

Thus, the final optimal size is 8.

**Step 3: Adjusting Array Size**

- The current size is 8.
- Check `tbl[9]`; even though it's `nil`, the index we're inserting is equal to 9 (`size + 1 == ek`), so the array size increases to 9.
- Check `tbl[10]`; since it is `nil`, the array size cannot be adjusted further.
- `extra = 9 - 8 = 1`
- New array size = `8 + (1 * 2) = 10`.
- `adjustasize` is called again to enforce boundary invariant, but since `tbl[11]` is nil, the size will remain 10

**Step 4: Final Size is 10**

##### Calculating Array Length

**Step 1:** Check index at `allocatedSize`, which is `tbl[10]`. Since the value is nil, binary search will be performed.

**Step 2:** Check midpoint `tbl[6]`, since it is non-nil, search upperhalf next.

**Step 3:** Check midpoint `tbl[8]`, since it is non-nil, search upperhalf next.

**Step 4:** Check midpoint `tbl[9]`, since it is non-nil and the next search size is 0, that's the final result.

**Step 5:** The final result for the array length is 9.

### Example 4:

```lua
local tbl = {
    [1] = 1,
    [2] = 2,
    [3] = nil,
    [4] = 4,
    [5] = nil,
    [6] = 6,
    [7] = nil,
    [8] = 8,
    [10] = 10,
    [9] = nil
}
```

For this example, let's assume we are inserting the tenth, and the ninth key

##### Inserting the Tenth Index

###### Calculating Array Size

**Step 1: Counting**

We count the total number of indices:

- `nums[0] = 1` at index 1 (range is [1, 1])
- `nums[1] = 1` at index 2 (range is [2, 2])
- `nums[2] = 1` at index 4 (range is [3, 4])
- `nums[3] = 2` at index 6, 8 (range is [5, 8])
- `nums[4] = 1` at index 10 (the key we are inserting) (range is [9, 16])

Total indices: 6

**Step 2: Computing Optimal Array Sizes**

- At index 1, there are 1 index total, which is above ⌊1/2⌋ (0) so 1 is the current most optimal size
- From index 1 to 2, there are 2 index total, which is above ⌊2/2⌋ (1) so 2 is the current most optimal size
- From index 1 to 4, there are 3 index total, which is above ⌊4/2⌋ (2) so 4 is the current most optimal size
- From index 1 to 8, there are 5 index total, which is above ⌊8/2⌋ (4) so 8 is the current most optimal size
- From index 1 to 16, there are 6 index total, which isn't above ⌊16/2⌋ (8) so 8 is is still the most optimal size
- Since we counted all the indice, we now have the highest optimal size

Thus, the final optimal size is 8.

**Step 3: Adjusting Array Size**

- The current size is 8.
- Check `tbl[9]`; since it is `nil`, the array size cannot be adjusted further.
- Final array size remains 8.

**Step 4: Final Size is 8**

##### Inserting the Ninth Index

###### Calculating Array Size

**Step 1: Counting**

We count the total number of indices:

- `nums[0] = 1` at index 1 (range is [1, 1])
- `nums[1] = 1` at index 2 (range is [2, 2])
- `nums[2] = 1` at index 4 (range is [3, 4])
- `nums[3] = 2` at index 6, 8 (range is [5, 8])
- `nums[4] = 2` at index 9, 10 (9 is the key we are inserting) (range is [9, 16])

Total indices: 7

**Step 2: Computing Optimal Array Sizes**

- At index 1, there are 1 index total, which is above ⌊1/2⌋ (0) so 1 is the current most optimal size
- From index 1 to 2, there are 2 index total, which is above ⌊2/2⌋ (1) so 2 is the current most optimal size
- From index 1 to 4, there are 3 index total, which is above ⌊4/2⌋ (2) so 4 is the current most optimal size
- From index 1 to 8, there are 5 index total, which is above ⌊8/2⌋ (4) so 8 is the current most optimal size
- From index 1 to 16, there are 7 index total, which isn't above ⌊16/2⌋ (8) so 8 is is still the most optimal size
- Since we counted all the indice, we now have the highest optimal size

Thus, the final optimal size is 8.

**Step 3: Adjusting Array Size**

- The current size is 8.
- Check `tbl[9]`; even though it's `nil`, the index we're inserting is equal to 9 (`size + 1 == ek`), so the array size increases to 9.
- Check `tbl[10]`; since the value is non-`nil`, the array size increases to 10.
- Check `tbl[11]`; since it is `nil`, the array size cannot be adjusted further.
- `extra = 10 - 8 = 2`
- New array size = `8 + (2 * 2) = 12`.
- `adjustasize` is called again to enforce boundary invariant, but since `tbl[13]` is nil, the size will remain 12

**Step 4: Final Size is 12**


###### Calculating Array Length

**Step 1:** Check index at `allocatedSize`, which is `tbl[12]`. Since the value is nil, binary search will be performed.

**Step 2:** Check midpoint `tbl[7]`, since it is nil, search lowerhalf next.

**Step 3:** Check midpoint `tbl[4]`, since it is non-nil, search upperhalf next.

**Step 4:** Check midpoint `tbl[5]`, since it is nil and, search lowerhalf next.

**Step 3:** Check midpoint `tbl[5]`, since it is nil, and the next search size is 0, that's the final result.

**Step 5:** The final result for the array length is 4.

## Exercise For Reader

Compute the length of the following table, then verify by running the code.

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

### Table Memory Usage

The `Table` structure in Luau defines the memory layout of a Luau table:

```Cpp
typedef struct Table
{
    CommonHeader;


    uint8_t tmcache;    // 1<<p means tagmethod(p) is not present
    uint8_t readonly;   // sandboxing feature to prohibit writes to table
    uint8_t safeenv;    // environment doesn't share globals with other scripts
    uint8_t lsizenode;  // log2 of size of `node' array
    uint8_t nodemask8; // (1<<lsizenode)-1, truncated to 8 bits

    int sizearray; // size of `array' array
    union
    {
        int lastfree;  // any free position is before this position
        int aboundary; // negated 'boundary' of `array' array; iff aboundary < 0
    };


    struct Table* metatable;
    TValue* array;  // array part
    LuaNode* node;
    GCObject* gclist;
} Table;
```

The `Table` structure itself requires **45 bytes**, with **3 bytes of padding** for alignment.
This results in a total size of **48 bytes** for the structure alone,
excluding the memory required for the `array` and `node`.

There is also an additional **16 bytes** for the `TValue` that is holding the `Table`.

#### Memory Usage Breakdown

1. **Array Part (`array`):**
   - The `array` part's memory depends on the size (`sizearray`) and the type of values stored.
   - Each value in Luau (a `TValue`) occupies **16 bytes**.
   - Total memory required for the `array` part:
      - `sizearray` * 16 bytes
   - Note: If the stored values are strings, tables, or other complex types, their additional memory usage must also be considered.

2. **Hash Part (`node`):**
   - The `node` (hash) part's memory depends on the hash size (`sizenode`) and the types of keys and values stored.
   - Each `LuaNode` contains both a key (a `TKey`) and a value (a `TValue`), with each occupying **16 bytes**.
   - Total memory required for the hash part:
      - `sizenode` \* 16 bytes \* 2
   - Additional memory may be consumed by complex keys or values (e.g., strings, tables).

This result in a final size of:

`TValue` **16 bytes** + `struct` **48 bytes** + `sizearray` \* **16 bytes** + `sizenode` \* **32 bytes** + any additional memory from key/value

= **64 bytes** overhead + `sizearray` \* **16 bytes** + `sizenode` \* **32 bytes** + any additional memory from key/value

---

## **Summary**

um no

## References

[ltable.cpp](https://github.com/luau-lang/luau/blob/master/VM/src/ltable.cpp), [lvm.cpp](https://github.com/luau-lang/luau/blob/master/VM/src/lvmexecute.cpp) - Luau source code
[lobject.h](https://github.com/luau-lang/luau/blob/master/VM/src/lobject.h) - Structure information
