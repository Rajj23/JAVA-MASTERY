# Collections Framework Internals — ArrayList, LinkedList, HashMap, HashSet, TreeMap

This document explains the internal working of the most important and commonly used Java Collections.

Understanding these internals is essential for:

- Writing efficient Java code
- Avoiding performance bottlenecks
- Answering interview questions
- Understanding framework internals (Spring, Hibernate, etc.)

---

## 1. ArrayList Internals

ArrayList is a dynamic resizable array implementation of List.

### 1.1 Internal Data Structure

ArrayList uses:

```java
transient Object[] elementData;
```

Default capacity: 10 (after first element insertion).

### 1.2 Access & Insert Complexity

| Operation | Time |
|-----------|------|
| `get(index)` | O(1) |
| `set(index)` | O(1) |
| `add` at end | O(1) amortized |
| insert/remove middle | O(n) |

**Reason:** Shifting elements is required for insert/delete in the middle.

### 1.3 Resizing Logic

When array is full:

```java
new_capacity = old_capacity + (old_capacity >> 1)
```

i.e., **1.5× growth**

Examples:

```
10 → 15 → 22 → 33 → 49 → ...
```

Resizing is costly → avoid unnecessary resizing.

### 1.4 Why ArrayList Is Not Thread-Safe?

Because:

- No synchronization
- Concurrent modifications can corrupt array

**Use:**

- `CopyOnWriteArrayList`
- `Collections.synchronizedList(...)`

---

## 2. LinkedList Internals

LinkedList is a doubly linked list.

### 2.1 Node Structure

Each node:

```java
Node {
    E item;
    Node next;
    Node prev;
}
```

### 2.2 Performance Characteristics

| Operation | Time |
|-----------|------|
| `addFirst` / `addLast` | O(1) |
| `removeFirst` / `removeLast` | O(1) |
| `get(index)` | O(n) |
| `remove(index)` | O(n) |

Random access is slow because traversal is required.

### 2.3 When to Use LinkedList?

Use when:

- Frequent insert/remove at beginning or middle
- No need for random access

---

## 3. HashMap Internals (Java 8+)

HashMap is the most important collection in Java. Used heavily in frameworks and system-level code.

Java 8 redesigned HashMap to handle collisions better.

### 3.1 Internal Structure

HashMap uses:

```java
Node<K,V>[] table;
```

Each bucket can contain:

- Linked List
- Red-Black Tree (after treeification)

### 3.2 Hashing Process

**Step 1:** Compute `hashCode()`

```java
int h = key.hashCode();
```

**Step 2:** Spread hash bits

HashMap applies:

```java
h ^ (h >>> 16)
```

This reduces collisions by mixing high bits with low bits.

### 3.3 Bucket Index Calculation

```java
index = hash & (n - 1)
```

Where `n` is table size (always power of 2).

### 3.4 Collision Handling

If two keys map to same bucket:

- Initially stored as LinkedList
- If bucket size > 8 → convert to Red-Black Tree
- If bucket size < 6 → convert back to LinkedList

This improves worst-case lookup from O(n) to O(log n).

### 3.5 equals() Check

Even if hash matches, HashMap checks:

```java
key.equals(existingKey)
```

To avoid overwriting wrong keys.

### 3.6 Load Factor & Resizing

Default load factor = **0.75**

Resize happens when:

```java
size >= capacity * loadFactor
```

Example:

```
capacity = 16
threshold = 12
```

At size 12 → resize to 32.

### 3.7 Resizing Optimization (Java 8)

During resize:

- Nodes are redistributed
- No need to recalculate full hash

Each node either:

- Stays in same bucket
- Moves to bucket `oldIndex + oldCapacity`

This is much faster than rehashing.

### 3.8 Why HashMap Is Not Thread-Safe?

Because:

- Multiple threads modifying same bucket cause corruption
- Infinite loops possible during resize
- No synchronization

**Use:**

- `ConcurrentHashMap`
- `Collections.synchronizedMap`

---

## 4. HashSet Internals

HashSet is built on top of HashMap.

Internally:

```java
HashMap<E, Object> map;
private static final Object PRESENT = new Object();
```

Each element is stored as:

```java
map.put(element, PRESENT);
```

Duplicate elements are prevented because HashMap keys must be unique.

---

## 5. TreeMap Internals

TreeMap is a sorted map based on Red-Black Tree.

### 5.1 Characteristics

- Sorted keys
- O(log n) insert
- O(log n) delete
- O(log n) search
- Maintains key ordering

Red-Black Tree ensures balancing after each modification.

---

## 6. When to Use Which Collection?

### ArrayList

- Best for random access
- Best for append operations
- Not good for middle inserts

### LinkedList

- Best for insert/delete at boundaries
- Not good for random access

### HashMap

- Best for key/value lookups
- Extremely fast average performance
- Does not maintain order

### HashSet

- Ensures uniqueness
- Backed by HashMap

### TreeMap

- Sorted keys
- Use when ordering matters

---

## 7. Summary (1-Minute Revision)

| Collection | Key Points |
|------------|------------|
| **ArrayList** | Dynamic array, fast random access |
| **LinkedList** | Doubly linked list, fast insert/delete |
| **HashMap** | Buckets + tree (Java 8), hashing + equals checks |
| **HashSet** | Uses HashMap internally |
| **TreeMap** | Red-Black Tree, sorted order |

---

## 8. Interview Questions

### Q1. How does HashMap work internally?

HashMap uses an array of buckets (`Node<K,V>[] table`). When a key-value pair is inserted:
1. The `hashCode()` of the key is computed
2. The hash is spread using `h ^ (h >>> 16)` to reduce collisions
3. Bucket index is calculated as `hash & (n - 1)`
4. If collision occurs, entries are stored as a linked list
5. If bucket size exceeds 8, it converts to a Red-Black Tree for O(log n) lookup
6. Both `hashCode()` and `equals()` are used to find/store entries

### Q2. Why does HashMap convert buckets to Red-Black Trees?

To improve worst-case performance. When many keys hash to the same bucket:
- Linked list provides O(n) lookup time
- Red-Black Tree provides O(log n) lookup time

Conversion happens when bucket size > 8, and reverts to linked list when size < 6 (hysteresis prevents frequent conversion).

### Q3. Explain HashMap resizing mechanism.

When `size >= capacity * loadFactor` (default 0.75):
1. A new array with double capacity is created
2. All entries are redistributed to new buckets
3. Java 8 optimization: entries either stay in the same index or move to `oldIndex + oldCapacity`
4. This avoids full hash recalculation for each entry

### Q4. What is load factor? Why is the default 0.75?

Load factor determines when to resize the HashMap.
- **0.75** balances time and space efficiency
- Higher value (e.g., 0.9) → less memory, more collisions, slower lookups
- Lower value (e.g., 0.5) → more memory, fewer collisions, faster lookups

0.75 provides a good trade-off with approximately 0.5 probability of bucket collisions based on statistical analysis.

### Q5. Difference between HashSet and HashMap?

| HashSet | HashMap |
|---------|---------|
| Stores only values | Stores key-value pairs |
| Implements `Set` interface | Implements `Map` interface |
| Uses `add(element)` | Uses `put(key, value)` |
| Internally uses HashMap | Core data structure |
| Element is the key, value is a dummy `PRESENT` object | Keys map to actual values |

### Q6. ArrayList vs LinkedList?

| ArrayList | LinkedList |
|-----------|------------|
| Dynamic array | Doubly linked list |
| O(1) random access | O(n) random access |
| O(n) insert/delete at middle | O(1) insert/delete at boundaries |
| Better cache locality | Poor cache locality |
| Less memory overhead | More memory (prev/next pointers) |

**Use ArrayList** for most cases. Use **LinkedList** only when frequent insertions/deletions at boundaries are required.

### Q7. Why are HashMap keys usually immutable?

If a key's `hashCode()` changes after insertion:
- The object will be in the wrong bucket
- `get()` will fail to find the entry
- Memory leak: old entry cannot be retrieved or removed

Immutable keys (like `String`, `Integer`) ensure consistent hash codes throughout their lifecycle.

### Q8. What happens when two keys have same hash?

This is called a **hash collision**. HashMap handles it by:
1. Storing both entries in the same bucket
2. Using a linked list to chain entries (separate chaining)
3. Converting to Red-Black Tree if chain length > 8
4. Using `equals()` to distinguish between keys with same hash
5. Both `hashCode()` and `equals()` must be properly implemented