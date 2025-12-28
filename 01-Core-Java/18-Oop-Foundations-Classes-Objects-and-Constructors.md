# OOP Foundations — Classes, Objects & Constructors (JVM-Backed)

Most developers learn OOP as a set of rules. This document explains OOP from the JVM and memory perspective, so concepts make sense instead of being memorized.

**Focus:**
- What a class really is
- What an object really is
- What the `new` keyword does
- Why constructors exist

> No inheritance or polymorphism yet — only solid foundations.

---

## 1. What Is a Class? (Beyond "Blueprint")

**Textbook definition:** "A class is a blueprint."

This is incomplete.

### JVM Reality

A class is **runtime metadata** loaded into the JVM.

When a class is loaded, the JVM creates metadata containing:

| Component       | Description                          |
|-----------------|--------------------------------------|
| Class Name      | Fully qualified name of the class    |
| Fields Info     | Field names, types, and modifiers    |
| Methods Info    | Method signatures and access flags   |
| Bytecode        | Compiled instructions for methods    |
| Constant Pool   | Literals and symbolic references     |
| Access Flags    | Visibility and modifiers             |

This metadata lives in:
- **Method Area / Metaspace** (NOT in the heap)

### Example

```java
class User {
    int id;
    void sayHello() {}
}
```

This creates **class metadata**, not objects.

---

## 2. Class Loading Happens Before Object Creation

Before any object exists:

1. `ClassLoader` loads the `.class` file
2. Bytecode is verified
3. Class metadata is stored in Metaspace
4. Static fields are initialized

**Only after this can objects be created.**

This directly connects to:
- ClassLoader system
- Metaspace memory area

---

## 3. What Is an Object? (Real Meaning)

An object is a **block of memory in the heap**.

Each object contains:
- Instance fields
- Object header
- Reference to class metadata

### Memory View

```
Object (Heap)
 ├── Object Header
 │    ├── Mark Word
 │    └── Class Pointer
 ├── Field 1
 ├── Field 2
 └── ...
```

> **Important:** Methods are NOT inside the object. They are stored once in class metadata.

---

## 4. Object Header (Why It Exists)

Every object has an **object header** containing:

| Component           | Purpose                                |
|---------------------|----------------------------------------|
| Mark Word           | GC info, locking state, identity hash  |
| Class Pointer       | Reference to class metadata            |

This is why:
- `hashCode()` works — identity hash stored in mark word
- `synchronized` works on objects — lock state in mark word
- GC can track object state — age and flags in mark word

### Object Header Size

| JVM Mode              | Header Size |
|-----------------------|-------------|
| 32-bit JVM            | 8 bytes     |
| 64-bit JVM            | 12 bytes    |
| 64-bit with CompressedOops | 12 bytes |

---

## 5. What the `new` Keyword Actually Does

```java
User u = new User();
```

### Step-by-Step Execution

| Step | Action                                      |
|------|---------------------------------------------|
| 1    | JVM checks if `User` class is loaded        |
| 2    | Memory is allocated in heap                 |
| 3    | Object header is initialized                |
| 4    | Instance fields set to default values       |
| 5    | Constructor is invoked                      |
| 6    | Reference assigned to variable `u`          |

**Summary:** `new` = allocation + initialization

---

## 6. Default Field Initialization (Why It Happens)

```java
class User {
    int age;
    String name;
    boolean active;
}
```

Before constructor runs:

| Type      | Default Value |
|-----------|---------------|
| `int`     | `0`           |
| `long`    | `0L`          |
| `float`   | `0.0f`        |
| `double`  | `0.0d`        |
| `boolean` | `false`       |
| `char`    | `'\u0000'`    |
| Object    | `null`        |

**Reason:** JVM guarantees objects start in a safe default state. This prevents reading garbage memory.

---

## 7. Why Constructors Exist (Real Purpose)

**Constructors do not create objects.**

The object already exists before constructor runs.

Constructors exist to:
- Move object from **default state → valid state**
- Enforce invariants
- Accept required dependencies

### Example

```java
class User {
    int age;
    
    User(int age) {
        this.age = age;
    }
}
```

---

## 8. Constructor Internals

| Property                | Details                                   |
|-------------------------|-------------------------------------------|
| Internal Name           | `<init>` in bytecode                      |
| Return Type             | None (not even `void`)                    |
| Invocation              | Automatically by JVM during `new`         |
| Explicit Call           | Not possible (except via `this()` or `super()`) |

They are **special initialization methods**, not regular methods.

### Constructor Chaining

```java
class User {
    int id;
    String name;
    
    User() {
        this(0, "Unknown");  // Calls parameterized constructor
    }
    
    User(int id, String name) {
        this.id = id;
        this.name = name;
    }
}
```

> **Note:** `this()` or `super()` must be the first statement in a constructor.

---

## 9. Multiple Constructors (Why Allowed)

Objects may be valid in different initial states.

```java
User()                        // Default user
User(int age)                 // User with age only
User(int age, String name)    // Fully initialized user
```

Each constructor ensures:
- Object ends in a **valid, usable state**

This is called **constructor overloading**.

---

## 10. Where Instance Variables Live

Each object has its **own copy** of instance variables.

```java
User u1 = new User();
User u2 = new User();

u1.age = 10;
u2.age = 20;
```

### Heap Memory Layout

```
Heap:
┌─────────────┐    ┌─────────────┐
│   u1        │    │   u2        │
│  age = 10   │    │  age = 20   │
└─────────────┘    └─────────────┘
```

---

## 11. Why Methods Are Shared

Methods:
- Belong to class metadata
- Loaded once into Metaspace
- Shared by all objects of that class

This makes Java **memory-efficient** — methods aren't duplicated per object.

### Critical For

| Concept                  | Why Methods Sharing Matters           |
|--------------------------|---------------------------------------|
| Polymorphism             | Method lookup via class pointer       |
| Dynamic Method Dispatch  | Runtime method resolution             |
| Overriding               | Child class metadata overrides method |

---

## 12. The `this` Keyword (Deep but Simple)

`this` is a reference to the **current object**.

### Use Cases

**1. Resolve ambiguity:**
```java
class User {
    int age;
    
    User(int age) {
        this.age = age;  // Field vs parameter
    }
}
```

**2. Constructor chaining:**
```java
User() {
    this(0);  // Call another constructor
}
```

**3. Pass current object:**
```java
void register() {
    registry.add(this);
}
```

---

## 13. Stack vs Heap (Applied View)

```java
User u = new User();
```

### Memory Layout

```
Stack Frame                    Heap
┌─────────────────┐           ┌─────────────────────┐
│ u (reference) ──┼──────────►│ User Object         │
│                 │           │  ├── Object Header  │
└─────────────────┘           │  ├── id = 0         │
                              │  └── name = null    │
                              └─────────────────────┘
```

**Key Insight:** Stack stores references, heap stores objects.

---

## 14. Common Beginner Misconceptions (Cleared)

| Misconception                      | Reality                                    |
|------------------------------------|--------------------------------------------|
| "Methods are inside objects"       | Methods live in class metadata (Metaspace) |
| "Constructor creates object"       | JVM creates object, constructor initializes|
| "Class is only a blueprint"        | Class is runtime metadata in Metaspace     |
| "Objects store everything"         | Objects store fields + header only         |

---

## 15. Mental Model to Remember

| Concept     | What It Really Is                    |
|-------------|--------------------------------------|
| Class       | Runtime metadata (description)       |
| Object      | Heap memory instance (real data)     |
| `new`       | Allocate + Initialize                |
| Constructor | Make object valid                    |

If this model is clear, OOP becomes intuitive.

---

## 16. What We Intentionally Skipped

The following topics require clarity first:

- Inheritance
- Polymorphism
- Interfaces
- Abstraction
- Static members

---

## 17. Summary

| Concept           | Key Takeaway                              |
|-------------------|-------------------------------------------|
| Class             | Runtime metadata in Metaspace             |
| Object            | Heap memory block with fields + header    |
| Methods           | Shared across objects, stored once        |
| `new`             | Allocates heap memory and initializes     |
| Constructor       | Validates and sets up object state        |
| `this`            | Reference to current object               |
| Stack vs Heap     | References on stack, objects on heap      |

---

## 18. Interview Questions

### Q1: What is a class at runtime?
**Answer:** At runtime, a class is **metadata stored in the Method Area (Metaspace)**. When a class is loaded, the JVM creates a data structure containing the class name, field information, method bytecode, constant pool, and access flags. This metadata is used by the JVM to create objects and invoke methods. It is NOT a "blueprint" in the abstract sense — it's actual data in memory.

---

### Q2: Where do methods live in memory?
**Answer:** Methods live in **class metadata within Metaspace** (Method Area). They are NOT stored inside objects. All objects of a class share the same method implementations. When a method is invoked, the JVM uses the object's class pointer to locate the method in Metaspace, then executes it with `this` referring to that object.

---

### Q3: What does `new` do internally?
**Answer:** The `new` keyword performs these steps:
1. Checks if the class is loaded; if not, triggers class loading
2. Allocates memory in the heap for the object
3. Initializes the object header (mark word + class pointer)
4. Sets all instance fields to their default values
5. Invokes the constructor (`<init>` method)
6. Returns the reference to the newly created object

---

### Q4: Does constructor create object?
**Answer:** No. The **object is already created before the constructor runs**. The JVM allocates heap memory and initializes default values first. The constructor's job is to move the object from a default state to a valid, usable state by setting field values and enforcing invariants.

---

### Q5: What is object header?
**Answer:** The object header is a fixed-size metadata block at the start of every object containing:
- **Mark Word:** Stores identity hashcode, GC age, lock state, and biased locking information
- **Class Pointer:** Reference to the class metadata in Metaspace

This header enables `hashCode()`, `synchronized` blocks, and garbage collection to work.

---

### Q6: Why does JVM initialize fields to default values?
**Answer:** The JVM initializes fields to default values to **guarantee memory safety**. Without this:
- Primitive fields might contain garbage from previously freed memory
- Object references might point to random memory addresses
- Programs could crash unpredictably or have security vulnerabilities

By zeroing memory, Java ensures predictable, safe initial state before constructor logic runs.

---

### Q7: What is the difference between stack and heap in object creation?
**Answer:** 
- **Stack:** Stores local variables including object references; memory is automatically freed when the method exits
- **Heap:** Stores actual objects; memory is managed by the garbage collector

When you write `User u = new User()`:
- `u` (the reference) is stored on the stack
- The `User` object is stored on the heap
- `u` contains the memory address of the heap object

---

### Q8: Can you call a constructor explicitly like a method?
**Answer:** No. Constructors cannot be called like regular methods. They are special `<init>` methods invoked only by the JVM during object creation via `new`. However, from within a constructor, you can:
- Call another constructor of the same class using `this(...)`
- Call a superclass constructor using `super(...)`

Both must be the first statement in the constructor.