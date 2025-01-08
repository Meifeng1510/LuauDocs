# Luau Garbage Collector

---

## Introduction

In any programming language, memory management plays a vital role in ensuring efficient and stable application performance.
Every time you create a new value, such as a number, string, or table, memory is allocated to store that data. For more
complex structures like tables, memory allocation becomes even more critical, as these structures can grow dynamically
and hold references to other values.

However, as data becomes outdated or unused, the memory associated with it needs to be cleaned up; otherwise, it will remain allocated,
leading to increased memory usage and, eventually, system slowdowns or crashes. This is where garbage collection comes into play.

A garbage collector is a system that automatically identifies and reclaims memory that is no longer in use,
freeing it for reuse by the program. By doing so, it eliminates the need for developers to manually manage memory
and reduces the risk of memory leaks.

In Luau, the garbage collector is designed with specific goals in mind:

- **Efficiency**: Minimize the impact on application performance during memory management.
- **Incrementality**: Allow for partial collection to reduce pauses, improving responsiveness,
                      especially in real-time applications like games.
- **Scalability**: Handle a wide range of workloads, from small scripts to large and complex systems.

By understanding how the Luau garbage collector works will help developers
write efficient code, avoid memory pitfalls, and optimize their applications.

---

## Basic Overview on How Luau Garbage Collection Works

Luau uses an automatic garbage collection system to manage memory for you. This means that when you create new objects or values,
memory is allocated automatically. As these objects become unused, the garbage collector reclaims their memory without requiring manual cleanup.

### Key Concepts: References and Garbage

#### References

When you declare a variable in Luau, you create a reference to an object in memory. For example:

```lua
local num = 123
```

In this case, a new object is created to hold the value `123`, and the variable `num` acts as a reference to that object.
Whenever you use `num`, you are using it as a reference to the object holding the value `123`.

> Internally, the value `123` is stored as `TValue`, a collectible object.

#### Garbage

If you remove the reference or change the object it is referencing, the original value may become unreachable:

```lua
num = nil
```

Here, the value `123` is no longer referenced by any variable. This makes it a "garbage" value—a value that is no longer reachable
because it lacks any *strong references. Without a reference, the memory occupied by this value is considered unused.

>In Luau, there are 2 types of references:
>
>1. **Strong References**: References that will prevent garbage collector from marking it as unreachable e.g., variable, table.
>2. **Weak References**: References from weak tables, which do not prevent the object from being garbage collected (table with `__mode` metamethod).

### How the Garbage Collector Deals with Garbage

When an object is marked as garbage, it means it is no longer being used by any part of the script.
If the memory it occupies is not reclaimed, it will accumulate over time, leading to *memory leaks*.
To address this issue, Luau uses automatic memory management via its garbage collector.

The garbage collector keeps track of objects and their references. When an object is dereferenced
and there are no longer any strong references to it, it will be marked as garbage:

```lua
local var = 123
-- later
var = nil -- The value `123` no longer have any strong references to it.
```

During the next garbage collection cycle, the garbage collector identifies that the value `123` is no longer referenced.
It then cleans up the memory occupied by the object, marking it as unused and freeing it for other uses.

### Garbage Collection Cycles

The garbage collector operates in cycles, periodically checking for objects that are no longer reachable.
Once identified, these objects are collected, and their memory is freed. This ensures that the program
does not suffer from memory leaks and maintains efficient memory usage.

If you would like more details, check out the [In-depth guide on how Luau garbage collector work](LuauGarbageCollectorIndepth)

---

## Memory leaks

### Understanding Memory Leaks

Now you might be wondering, what is a memory leak?

Memory leaks can be understood through an analogy: Imagine a pool representing the total memory available on a system. Taking
water out of the pool and filling a bucket is like allocating memory for an object. When you no longer need the bucket, you should
pour the water back into the pool to make the memory available again. However, if you forget to return the water to the pool, it's
like the pool is slowly losing water, as if it has leaks. Over time, the pool loses more and more water (memory), and eventually,
there won’t be enough left to fill new buckets (allocate memory). This is a memory leak—unused memory that isn’t returned to the pool,
causing the system to run out of available resources.

Even though Luau have a garbage collector which automatically handle cleaning up memory for you,
it is not fool proof. It can only reduces so much of the risk of memory leaks.

### Detecting and Handling Memory Leaks

Consider the following example:

```lua
local objectList = {}

local function createObject(value)
    local newObject = {}
    newObject.Value = value

    table.insert(objectList, newObject) -- to keep track of objects
    return newObject
end

local function getValue(obj)
    if table.find(objectList, obj) then
        return obj.Value
    end
    error("Invalid object")
end

local function addObject(objA, objB)
    local sum = getValue(objA) + getValue(objB)
    return createObject(sum)
end
```

```lua
local function addNumber(numA, numB)
    local objA = createObject(123)
    local objB = createObject(456)

    local sum = getValue(addObject(objA, objB))
    return sum
end

print(addNumber(123, 456)) -- 579
```

Every time the function `addNumber` is called, we create new objects, add them together, then return the result.
Even though we no longer need `objA` and `objB`, there are still references to the objects in `objectList`, causing a memory leak.

### Solving Memory Leaks

To solve this problem, we need to remove the objects from `objectList` when they are no longer used:

```lua
local function removeObject(obj)
    local index = table.find(objectList, obj)
    if index then table.remove(objectList, index) end
end
```

Here’s the updated code:

```lua
local function addNumber(numA, numB)
    local objA = createObject(numA)
    local objB = createObject(numB)

    local sumObj = addObject(objA, objB)
    local sum = getValue(sumObj)

    removeObject(objA)
    removeObject(objB)
    removeObject(sumObj)

    return sum
end

print(addNumber(123, 456)) -- 579
```

Now, there are no memory leaks caused by calling `addNumber`.

### Practical Examples

With this understanding, let’s explore common scenarios causing memory leaks in Roblox and learn how to apply these concepts effectively.

#### Events

Events are one of the most commonly used concepts in Roblox, but they are also a frequent cause of memory leaks.
Beginners often do not fully comprehend how events work and may misuse them, leading to unintended behavior and memory retention.

Consider the following example:

```lua
-- Memory leak example with events
local UserInputService = game:GetService("UserInputService")
local Tool = Backpack.Tool

Tool.Equipped:Connect(function()
    UserInputService.InputBegan:Connect(function(inputObject, gameProcessingEvent)
        if inputObject.KeyCode == Enum.KeyCode.E then
            print("Pressed E")
        end
    end)
end)
```

In this example, every time the tool is equipped, a new connection to the `InputBegan` event is created.
These connections persist in memory even if the tool is unequipped or destroyed, leading to a memory leak.
Over time, this can result in numerous redundant connections, causing performance issues and unexpected behavior.

**Solution**: Manage event connections carefully by disconnecting them when they are no longer needed.
              One approach is to keep track of the connections and clean them up explicitly:

```lua
-- Properly managing event connections
local UserInputService = game:GetService("UserInputService")
local Tool = Backpack.Tool

local inputConnection

Tool.Equipped:Connect(function()
    inputConnection = UserInputService.InputBegan:Connect(function(inputObject, gameProcessingEvent)
        if inputObject.KeyCode == Enum.KeyCode.E then
            print("Pressed E")
        end
    end)
end)

Tool.Unequipped:Connect(function()
    if inputConnection then
        inputConnection:Disconnect()
        inputConnection = nil
    end
end)
```

Alternatively,

```lua
local UserInputService = game:GetService("UserInputService")
local Tool = Backpack.Tool

local isEquipped = false

Tool.Equipped:Connect(function()
    isEquipped = true
end)

Tool.Unequipped:Connect(function()
    isEquipped = false
end)

UserInputService.InputBegan:Connect(function(inputObject, gameProcessingEvent)
    if isEquipped and inputObject.KeyCode == Enum.KeyCode.E then
        print("Pressed E")
    end
end)
```

#### Tables

Tables are another common source of memory leaks in Roblox.
A typical issue arises when developers create persistent table
retaining references to unused data.

Consider this example:

```lua
-- Memory leak example with persistent Tables
local Players = game:GetService("Players")
local PlayersData = {}

Players.PlayerAdded:Connect(function(player)
    -- After player leave the game, even though player
    -- object is Destroyed, the PlayersData still hold
    -- reference to it which prevents garbage collection.
    PlayersData[player] = {...}
end)
```

In this case, the `PlayersData` table creates a memory leak because it retains references to `player` objects
even after they leave the game. When a player leaves, the `player` object is destroyed, but the reference
in `PlayersData` remains, preventing garbage collection.

This can become problematic if a large number of players join and leave the game over time,
as the `PlayersData` table will continue to grow, holding onto unused data.

**Solution**: Use the `PlayerRemoving` event to clean up references to players when they leave the game:

```lua
-- Prevent memory leaks with tables
local Players = game:GetService("Players")
local PlayersData = {}

Players.PlayerAdded:Connect(function(player)
    PlayersData[player] = {...} -- Store data for the player
end)

Players.PlayerRemoving:Connect(function(player)
    PlayersData[player] = nil -- Remove the player's data when they leave
end)
```

By removing the player's data when they leave, you ensure that the table does not
retain unused references, allowing the garbage collector to clean up properly.

---

## Best Practices

### Avoiding Excessive Object Creation

Creating too many objects in Luau can lead to frequent garbage collection cycles, which may degrade performance. To minimize object creation:

- **Reuse objects where possible:** Instead of creating new tables or instances, consider clearing and reusing existing ones.
- **Batch operations:** Group operations that require temporary objects to reduce the number of objects created.
- **Avoid unnecessary allocations:** For example, use string concatenation sparingly and prefer table-based string building with `table.concat`.

### Managing References Effectively

Improper reference management can prevent objects from being garbage collected, leading to memory leaks. To manage references effectively:

- **Remove unused references:** Explicitly set table's value to `nil` and clear the table when they are no longer needed.
- **Monitor global variables:** Ensure that temporary objects do not inadvertently remain accessible through global scope.

### Reduce GC Load

Reducing the load on the garbage collector can improve application performance:

- **Avoid one-use objects:** Instead of creating new objects (e.g., table or Instances), consider clearing and reusing existing ones i.e., object pooling.
- **Use lightweight alternatives:** Replace Instances with tables/userdata or more lightweight data structures when appropriate.
- **Preallocate memory:** For predictable workloads, preallocate tables or arrays to minimize dynamic resizing.

---

## Troubleshooting

### Identifying Garbage Collection Issues

Garbage collection issues often manifest as high memory usage or performance drops. To identify these problems:

- **Monitor memory usage:** Check [Developer Console](https://create.roblox.com/docs/studio/developer-console) to track memory consumption over time.
- **Look for patterns:** Identify spikes in memory usage corresponding to application activity.

### Detecting Memory Leak

Roblox provides [several tools](https://create.roblox.com/docs/studio/optimization/memory-usage) and [analytics](https://devforum.roblox.com/t/analytics-server-memory-usage-by-server-age/3300529) to help identify memory leaks:

- **Server memory usage:** Use [analytics](https://devforum.roblox.com/t/analytics-server-memory-usage-by-server-age/3300529) to monitor trends in server memory usage. An increase over time could indicate a potential memory leak.
- **Developer Console:** Review the memory graph to observe memory usage trends. Create a snapshot using [Luau Heap](https://create.roblox.com/docs/studio/optimization/memory-usage#luau-heap) to compare memory changes and pinpoint the source of the increase. Trace object references and identify why certain objects are not being collected.

---

## Summary

Understanding the garbage collector in Luau is essential for efficient memory management and optimal application performance. By leveraging its automatic memory management, developers can focus on creating robust and scalable applications without worrying about manual memory allocation or deallocation. However, relying solely on the garbage collector is not foolproof—poor coding practices, such as unintentional reference retention, can still lead to memory leaks and performance issues.

To ensure efficient use of memory and avoid common pitfalls:

- **Understand references:** Distinguish between strong and weak references and their impact on garbage collection.
- **Manage objects proactively:** Remove unused references, disconnect events, and clean up tables when they are no longer needed.
- **Adopt best practices:** Minimize excessive object creation, reuse resources when possible, and monitor memory usage over time.

By following these principles and implementing the strategies outlined in this guide, you can write cleaner, more efficient code and maintain the performance and stability of your Luau-based applications.
