# String Internals — String Pool, Immutability, Memory Layout, Performance

## Day 9 – Core Java Mastery

Strings are one of the most optimized, most used, and most special data types in Java.
Understanding how Strings work internally unlocks deeper knowledge of memory management, compiler behavior, class loaders, and performance tuning.

This document explains Strings from the JVM’s internal perspective.

## 1. Why Strings Are Special in Java

Strings are used everywhere:
- class names
- method names
- package names
- file paths
- network protocols
- database queries
- keys in HashMap
- JSON / XML data

Because of this, Java treats Strings differently from all other objects:
- They are immutable
- They use a String Pool
- They get special JIT optimizations
- They are stored in a compact form since Java 9

## 2. Why Are Strings Immutable?

Most explanations mention security or thread-safety, but the real reasons are deeper.

### Reason 1: Required for the String Pool

If Strings were mutable, this would happen:
- pool has "Aspen"
- someone modifies → becomes "Ayushi"

This would break:
- class loading
- constant resolution
- HashMap keys
- JVM metadata

Immutability makes pooling possible and safe.

### Reason 2: ClassLoader and Reflection Depend on Strings

Class names, method names, field names are stored as Strings.
They must never change.

### Reason 3: Cached HashCode

String’s hashCode() is computed once and cached:

```java
private int hash; // cached value
```

If Strings were mutable, cached hashcode would become invalid → HashMap breaks.

## 3. Internal Memory Layout of String

**Before Java 9**
```java
char[] value;
```

**After Java 9 (Compact Strings)**
```java
byte[] value;
byte coder; // LATIN-1 or UTF-16
```

**Benefits of Compact Strings:**
- 50% less memory for Latin characters
- Faster operations
- Lower GC pressure

This is one of the biggest optimizations in Java 9+.

## 4. String Pool (Intern Pool)

The String Pool is a special memory region inside the heap that stores unique string literals.

Example:
```java
String s1 = "Hello";
String s2 = "Hello";
```

1. **Step 1**: When s1 is created, JVM looks in the Pool. It sees it is empty, creates "Hello" there, and makes s1 point to it.
2. **Step 2**: When s2 is created, JVM looks in the Pool. It sees "Hello" already exists.
3. **Step 3**: Instead of creating a new object, it simply makes s2 point to the same "Hello" in the pool.

**Result**: s1 and s2 point to the exact same memory address. `s1 == s2` is true.

Both point to the same pooled object.

**Why a String Pool?**
- avoid duplicate objects
- reduce heap usage
- speed up equality checks (==)
- improve class loading performance

## 5. Where Is the String Pool Stored?

**Java 6 and earlier:**
- Stored in PermGen.

**Java 7+:**
- Moved to Heap (safer & more scalable).

This lets GC manage Strings efficiently.

## 6. How Strings Enter the Pool

### 1. Automatic — String Literals
```java
String s = "hello";
```
Compiler adds "hello" to the pool at class load time.

### 2. Explicit — Using intern()
```java
String s1 = new String("hello");
String s2 = s1.intern();
```
s2 now refers to the pooled instance.

## 7. Why new String("hello") Creates Two Objects

```java
String s = new String("hello");
```

- "hello" in pool (literal)
- a new "hello" object on heap

Therefore:
```java
"hello" == new String("hello") // false
"hello" == new String("hello").intern() // true
```

## 8. String Concatenation Internals

### Compile-Time Concatenation
```java
String s = "a" + "b";
```
Compiler converts to:
```java
String s = "ab";
```
No runtime cost.

### Runtime Concatenation
```java
String s = x + y;
```
Compiler translates to:
```java
StringBuilder sb = new StringBuilder();
sb.append(x);
sb.append(y);
s = sb.toString();
```

**Reason:**
Strings are immutable → concatenation requires new Strings.

## 9. StringBuilder vs StringBuffer

### StringBuilder
- not synchronized
- faster
- used in most cases

### StringBuffer
- synchronized
- thread-safe
- slower
- legacy usage

## 10. String Equality: == vs .equals()

- `==` → reference comparison
- `.equals()` → value comparison

Examples:
```java
"Aspen" == "Aspen"                     // true
new String("Aspen") == "Aspen"         // false
new String("Aspen").intern() == "Aspen" // true
```

## 11. String HashCode Caching

String stores its computed hash:
```java
private int hash; // initially 0
```

Once computed:
- cached
- reused

This significantly speeds up:
- HashMap
- HashSet
- LinkedHashMap

## 12. Performance Considerations

1. **Avoid unnecessary new String()**
   - Wastes memory, skips pool.
2. **Use StringBuilder for loops**
   - Better performance than +.
3. **Avoid many intermediate Strings**
   - Prefer: `StringBuilder`, `StringJoiner`, `String.format`.
4. **Use intern() carefully**
   - Can reduce memory OR increase GC pressure depending on usage.

## 13. Summary (1-Minute Revision)

- Strings are immutable → required for pooling, security, hash code caching
- String Pool stores unique literal strings
- Java 9 uses Compact Strings (byte[])
- intern() returns pooled instance
- "abc" and new String("abc") are different
- Concatenation becomes StringBuilder at runtime
- String hashcode is cached
- Strings are highly optimized by JIT

## 14. Interview Questions

1. **Why is String immutable?**
   - For Security (class names, paths), Thread-safety (shared without sync), String Pool support (caching), and Performance (hashCode caching).
2. **What is the String Pool and why is it used?**
   - A special memory region in the Heap storing unique String literals to save memory and improve performance by reusing common strings.
3. **Difference between "abc" and new String("abc")?**
   - `"abc"` uses the String Pool (reused if exists). `new String("abc")` forces creation of a new object on the Heap, even if "abc" is in the pool.
4. **How does intern() work?**
   - It checks the String Pool. If the string exists, it returns the pooled reference. If not, it adds the string to the pool and returns the new reference.
5. **Why does Java use byte[] for Strings now?**
   - (Compact Strings, Java 9+) Because most strings contain only Latin-1 characters (1 byte). Using `char[]` (2 bytes) wasted 50% memory. `byte[]` with a `coder` flag saves memory.
6. **How does String concatenation work internally?**
   - Compile-time constants (`"a" + "b"`) are joined by the compiler. Runtime concatenation (`s1 + s2`) uses `StringBuilder` (or `makeConcatWithConstants` in newer Java) to avoid creating temporary objects for every `+`.
7. **Why is StringBuilder preferred over + in loops?**
   - Using `+` in a loop creates a new `StringBuilder` and new `String` object *in every iteration*, causing massive memory waste (O(n²)). Explicit `StringBuilder` reuses the buffer (O(n)).
8. **Why is StringBuffer slower?**
   - Because its methods are `synchronized` for thread safety, which adds overhead. `StringBuilder` is non-synchronized and faster for single-threaded use.