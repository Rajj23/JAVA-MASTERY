# Generics, Type Erasure & Wildcards (Compiler + JVM View)


This document explains Java Generics from the perspective of compiler behavior, type erasure, and JVM limitations. Understanding this removes confusion around wildcards, raw types, and runtime behavior.

## 1. Why Generics Were Introduced

Before Java 5, collections were not type-safe:

```java
List list = new ArrayList();
list.add("hello");
list.add(10);

String s = (String) list.get(1); // ClassCastException
```

**Problems:**
- runtime errors
- unsafe casts
- no compile-time checking

**Java introduced generics to:**
- Provide compile-time type safety without changing JVM

## 2. What Generics Really Are

**Key truth:**
> Generics are a compile-time feature.

They exist to:
- enforce type safety
- remove explicit casts
- catch bugs early

The JVM has no knowledge of generic types (mostly).

## 3. Generic Syntax (Compile-Time Only)

```java
List<String> names = new ArrayList<>();
```

**Compiler guarantees:**
- only `String` can be added
- no cast needed on retrieval

After compilation, this information disappears.

## 4. Type Erasure (Core Concept)

During compilation:
`List<String>`

becomes:
`List`

This process is called **type erasure**.

**The compiler:**
1. removes generic type information
2. inserts casts
3. ensures type safety before runtime

## 5. Why Java Uses Type Erasure

**Reasons:**
- Backward compatibility with pre-Java 5 code
- No JVM changes required
- Simpler runtime model

Type erasure is a design tradeoff, not a flaw.

## 6. What Compiler Actually Generates

**Source code:**
```java
List<String> list = new ArrayList<>();
String s = list.get(0);
```

**After compilation (Bytecode equivalent):**
```java
List list = new ArrayList();
String s = (String) list.get(0);
```

Casts are inserted automatically.

## 7. Consequences of Type Erasure

Because of erasure:

❌ **Cannot use:**
- `new T()` (Cannot instantiate type parameters)
- `T.class` (No class literal for type parameter)
- `instanceof List<String>` (Runtime doesn't know `<String>`)
    - *Use `instanceof List<?>` instead.*

❌ **Cannot overload methods by generic type alone:**
```java
void m(List<String> l) {}
void m(List<Integer> l) {} // compile error: both erase to m(List)
```

At runtime, both are just `List`.

## 8. Raw Types (Danger Zone)

```java
List list = new ArrayList();
```

**Raw types:**
- disable generics
- remove type safety
- exist only for backward compatibility

**Rule:**
> Never use raw types in new code.

## 9. Generic Classes

```java
class Box<T> {
    private T value;
    T get() { return value; }
}
```

- `T` is a placeholder
- replaced at compile time (usually by `Object` if unbounded)
- erased at runtime

All instantiations (`Box<String>`, `Box<Integer>`) share the same bytecode class `Box`.

## 10. Generic Methods

```java
static <T> T identity(T value) {
    return value;
}
```

- `<T>` belongs to the method
- independent of class generics
- compiler infers type

### Type Inference (Java 7+ and 10+)
- **Diamond Operator (`<>`)**: `List<String> l = new ArrayList<>();` (Infers from left side)
- **Local Variable (`var`)**: `var list = new ArrayList<String>();` (Infers from right side)

## 11. Bounded Type Parameters

`<T extends Number>`

**Meaning:**
- `T` must be `Number` or subclass
- allows safe method calls to `Number` methods (e.g., `doubleValue()`)
- erases to `Number` instead of `Object`

### Example:
```java
<T extends Number> double sum(T a, T b) {
    return a.doubleValue() + b.doubleValue();
}
```

### Multiple Bounds
```java
<T extends Number & Comparable<T>>
```
- `T` must extend `Number` **AND** implement `Comparable`.
- First bound (`Number`) is used for erasure.

## 12. Why Wildcards Exist

**Problem:**
```java
List<Dog> dogs = new ArrayList<>();
List<Animal> animals = dogs; // ❌ Compile Error
```

Generics are **invariant**. A `List<Dog>` is NOT a `List<Animal>`.

Wildcards (`?`) solve variance issues.

## 13. `? extends T` (Producer) - Upper Bounded Wildcard

```java
List<? extends Animal> animals;
```

- safe to **read** as `Animal`
- **unsafe to add** elements (Compiler doesn't know if it's a list of `Dog`, `Cat`, etc.)
- Accepts `List<Animal>`, `List<Dog>`, `List<Cat>`

**Rule:**
> Use `extends` when you only **read** (Producer).

## 14. `? super T` (Consumer) - Lower Bounded Wildcard

```java
List<? super Dog> dogs;
```

- safe to **add** `Dog` (or `Puppy`)
- reading gives `Object` (don't know specific supertype)
- Accepts `List<Dog>`, `List<Animal>`, `List<Object>`

**Rule:**
> Use `super` when you only **write** (Consumer).

## 15. PECS Rule (Critical Rule)

**PECS = Producer Extends, Consumer Super**

- **Producer** (Source of data) → `? extends T`
- **Consumer** (Target for data) → `? super T`

This rule solves most wildcard confusion.

## 16. Advanced: Recursive Generics

Sometimes a type depends on itself.

```java
public class MyClass<T extends Comparable<T>> { ... }
```
Required for `Collections.sort()` etc. It ensures that `T` can be compared to other instances of `T`.

## 17. Advanced: Bridge Methods

When compiling a class that extends a generic class, the compiler may generate synthetic **bridge methods** to preserve polymorphism.

```java
class Node<T> { public void setData(T data) { ... } }
class MyNode extends Node<Integer> { public void setData(Integer data) { ... } }
```
The compiler generates a bridge method in `MyNode`:
```java
public void setData(Object data) { // Synthetic bridge
    setData((Integer) data);
}
```
This ensures that `Node.setData(Object)` correctly calls `MyNode.setData(Integer)` at runtime.

## 18. Advanced: Heap Pollution

Happens when a variable of a parameterized type refers to an object that is not of that type.
This often occurs with **varargs** and generics.

```java
// Warning: Possible heap pollution from parameterized vararg type
static void dangerous(List<String>... stringLists) {
    Object[] objects = stringLists;
    objects[0] = Arrays.asList(42); // Heap pollution
    String s = stringLists[0].get(0); // ClassCastException at runtime
}
```
Use **`@SafeVarargs`** to suppress warnings if you are sure the method is safe (i.e., doesn't store the array or let it escape).

## 19. Arrays vs Generics (Important Difference)

**Arrays:**
- **Covariant** (`Dog[]` is an `Animal[]`)
- **Reified** (Check types at runtime)

```java
Animal[] a = new Dog[10];
a[0] = new Cat(); // ArrayStoreException at runtime
```

**Generics:**
- **Invariant** (`List<Dog>` is NOT `List<Animal>`)
- **Erased** (Compile-time check only)

Java favors type safety over runtime checks.

## 20. Performance of Generics

Generics:
- have **no runtime cost** (no extra memory or instructions)
- are erased during compilation
- generate same bytecode as raw types

**Performance is unchanged.**

## 21. Common Generics Mistakes

- ❌ Using raw types
- ❌ Expecting runtime generic info (`T.class` or `instanceof T`)
- ❌ Misusing wildcards (Adding to `? extends`)
- ❌ Forgetting PECS rule
- ❌ creating arrays of generic types (`new List<String>[10]`) -> Use `ArrayList` instead.

## 22. Mental Model

- Generics = compile-time safety
- Type erasure = runtime simplicity
- `extends` = read
- `super` = write

## 23. Summary

- Generics add compile-time safety
- JVM erases generic info (Type Erasure)
- Compiler inserts casts automatically
- Raw types break safety
- Wildcards handle variance (Use PECS)
- Bridge methods preserve polymorphism
- Generics have **no runtime cost**

## 24. Interview Questions & Answers

### 1. What is type erasure?
**Answer:** Type erasure is the process by which the Java compiler removes all generic type information from the code after checking for type safety. Generic types (like `List<String>`) are replaced by their bounds (like `List` or `List<Object>`), and type casts are inserted where needed. This ensures backward compatibility with older Java versions that didn't support generics.

### 2. Why does the JVM not support generics at runtime?
**Answer:** To maintain **backward compatibility**. If the JVM were modified to support "reified" generics (generics with runtime type info), older Java applications and libraries might break or require recompilation. Using erasure allowed Java 5+ code to interoperate seamlessly with legacy code.

### 3. Why `List<Dog>` is not `List<Animal>`?
**Answer:** Because generics are **invariant**. If `List<Dog>` were a subtype of `List<Animal>`, you could assign `dogs` to `animals` and then add a `Cat` to it (`animals.add(new Cat())`). This would violate the type safety of the original `List<Dog>`, crashing only when you tried to retrieve the cat as a dog later. Invariance prevents this safety hole.

### 4. Explain the PECS rule.
**Answer:** PECS stands for **Producer Extends, Consumer Super**.
- If you need to **read** items from a generic collection (it produces data), use `? extends T`.
- If you need to **write** items into a generic collection (it consumes data), use `? super T`.

### 5. Why are raw types dangerous?
**Answer:** Raw types (e.g., `List` instead of `List<String>`) opt out of generic type checking. The compiler cannot catch type errors, so you can accidentally insert the wrong type (e.g., adding an `Integer` to a list meant for `Strings`). This results in a `ClassCastException` at runtime, which is harder to debug than a compile-time error.