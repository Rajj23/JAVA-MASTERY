# Abstraction, Abstract Classes & Interfaces (JVM + Design View)

>
> This document completes the core OOP foundation by explaining abstraction, abstract classes, and interfaces from both JVM internals and a real-world design perspective.

Abstraction is not about hiding code — it is **about defining contracts and responsibilities**.

---

## 1. Why Abstraction Exists (The Real Problem)

Consider this code:

```java
Animal a = new Dog();
a.sound();
```

From earlier days:
- Compiler checks `Animal`
- JVM executes `Dog`

**Now ask:**
Why should `Animal` be forced to provide an implementation of `sound()`?

**Sometimes:**
- The base class cannot define behavior.
- Only the child knows how to behave.

This is where abstraction is required.

---

## 2. What Abstraction Really Means

Abstraction means:
- Define **WHAT** must be done, not **HOW** it is done.
- It is about contracts, not hiding implementation.

---

## 3. Abstract Methods (JVM Perspective)

```java
abstract class Animal {
    abstract void sound();
}
```

An abstract method:
- Has no body.
- Is a contract.
- Must be implemented by subclasses.

**At JVM level:**
- Class metadata contains an **unimplemented method slot**.
- JVM expects the subclass to provide implementation at runtime.

---

## 4. Why JVM Supports Abstract Methods

Abstract methods exist because:
- JVM supports **dynamic dispatch**.
- Method resolution happens at runtime.
- Class metadata can safely hold abstract entries.

Without JVM support, abstraction would not be possible.

---

## 5. Abstract Classes (Why They Exist)

Abstract classes are used when:
- Some behavior is **common**.
- Some behavior must be **customized**.

**Example:**

```java
abstract class Vehicle {
    void start() {
        System.out.println("Vehicle starting");
    }

    abstract int wheels();
}
```

Here:
- `start()` → shared behavior.
- `wheels()` → subclass responsibility.

---

## 6. Enforcement of Abstract Methods

If a subclass does not implement all abstract methods:

```java
class Car extends Vehicle {
}
```

**Compile-time error.**

**Why?**
The compiler ensures the class is fully valid before object creation.

---

## 7. Why Abstract Classes Cannot Be Instantiated

```java
Vehicle v = new Vehicle();
```

**Reason:**
- Abstract methods have no implementation.
- The object would be incomplete.
- JVM guarantees object validity.

---

## 8. Abstract Class vs Concrete Class (Runtime View)

- **Abstract class** → metadata only.
- **No object creation**.
- **Subclass** completes missing behavior.
- Only fully implemented classes can be instantiated.

---

## 9. Why Interfaces Exist (Separate from Abstract Classes)

Before Java 8, interfaces existed to solve:
- **Multiple inheritance of state** (avoided).
- **Multiple inheritance of contracts** (allowed).

Interfaces avoid:
- The **diamond problem**.
- Ambiguous memory layout.

---

## 10. What Is an Interface (Correct Meaning)

```java
interface Flyable {
    void fly();
}
```

An interface is:
- A **pure contract** with no object state.
- It defines **capabilities**, not identity.

---

## 11. Why Interfaces Cannot Have Instance Fields

Interfaces:
- Do not participate in object memory layout.
- Cannot store state (instance variables).
- Avoid inheritance ambiguity.

**Note:** Variables in interfaces are implicitly `public static final` (constants).

```java
interface MathConstants {
    double PI = 3.14159; // implicitly public static final
}
```

This design keeps interfaces safe for multiple inheritance.

---

## 12. JVM Handling of Interfaces

**Internally:**
- Interfaces have metadata.
- Method signatures are abstract slots.
- Implementing class provides the implementation.
- JVM resolves method calls dynamically (`invokeinterface`).

Interface method calls still use dynamic dispatch.

---

## 13. Multiple Interfaces (Why It Works)

```java
class Bird implements Flyable, Walkable {
}
```

**Safe because:**
- No shared state.
- No constructor conflicts.
- Only method contracts are inherited.

---

## 14. Default Methods (Java 8+) — Why Added

**Problem:**
- Interfaces could not evolve.
- Adding methods broke existing implementations.

**Solution (Default Methods):**

```java
interface logger {
    default void log() {
        System.out.println("Logging");
    }
}
```

**Benefits:**
- **Backward compatibility**: Existing classes don't break.
- **Shared behavior**: Can provide utility methods.
- Still avoids state inheritance (no instance fields allowed).

---

## 15. Modern Interface Features (Missing pieces in traditional view)

### A. Static Methods (Java 8)
Interfaces can have static helper methods, removing the need for separate utility classes (like `Collections` vs `Collection`).

```java
interface Calculator {
    static int add(int a, int b) {
        return a + b;
    }
}
```

### B. Private Methods (Java 9)
Used to share code between default methods without exposing it to the API.

```java
interface Database {
    default void connectRead() { log("Read"); }
    default void connectWrite() { log("Write"); }
    
    // Private helper, not visible to implementation classes
    private void log(String operation) {
        System.out.println("DB Operation: " + operation);
    }
}
```

### C. Functional Interfaces & Lambdas
An interface with **exactly one abstract method** is a Functional Interface. This is the basis for Lambda expressions.

```java
@FunctionalInterface
interface Action {
    void run();
}

// Usage with Lambda
Action a = () -> System.out.println("Running");
```

### D. Marker Interfaces
Interfaces with **no methods**. They act as metadata/tags for the JVM or libraries.
- Examples: `Serializable`, `Cloneable`, `Remote`.
- Usage: `if (obj instanceof Serializable) { ... }`

---

## 16. Sealed Classes & Interfaces (Java 15+ - The Future)

Traditionally, abstraction was "open" (anyone can implement). **Sealed classes** restrict WHO can implement them.

```java
sealed interface Payment permits CreditCard, PayPal {
    void pay();
}
```

This gives more control over the hierarchy and helps design domain models strictly.

---

## 17. Abstract Class vs Interface (Deep Comparison)

| Aspect | Abstract Class | Interface |
| :--- | :--- | :--- |
| **State** | Can have instance variables | No instance variables (constants only) |
| **Constructors** | Yes (called by subclass) | No |
| **Methods** | Abstract + Concrete | Abstract + Default + Static + Private |
| **Multiple Inheritance** | No | Yes |
| **Purpose** | Share code + Contract (IS-A) | Define Capabilities (CAN-DO) |
| **Access Modifiers** | Any (public, protected, etc.) | Implicitly public (mostly) |
| **Memory Layout** | Part of object memory | No part in object memory |

---

## 18. When to Use What (Design Rules)

**Use Abstract Class when:**
- **IS-A relationship** (e.g., Dog IS-A Animal).
- Shared **implementation/state** exists (non-final fields).
- You need to control inheritance (protected members).

**Use Interface when:**
- **Capability-based design** (e.g., Flyable, Serializable).
- **Multiple inheritance** is needed.
- **API design** (decouple definition from implementation).

---

## 19. Why Frameworks Prefer Interfaces

Frameworks like Spring and Hibernate use interfaces because:
- **Loose coupling**: Dependence on interface, not implementation.
- **Easy replacement**: Swapping implementations (e.g., Mock objects for testing).
- **Proxy generation**: Java's Dynamic Proxy mechanism works on interfaces.
- **Dependency Injection**: Easier to inject different beans implementing the same interface.

---

## 20. Common Misconceptions (Cleared)

- **Misconception:** “Abstract class = partial abstraction, Interface = 100% abstraction”
  - **Reality:** With default/private methods, interfaces also have implementation. It is about **State** vs **No State**.
  
- **Misconception:** “Interfaces are slower”
  - **Reality:** JVM heavily optimizes interface calls (`invokeinterface`).

- **Misconception:** “Use interfaces everywhere”
  - **Reality:** Use based on design need. If you need state, use abstract classes.

---

## 21. Mental Model to Remember

- **Abstract Class** → **What you are** (Identity + Partial Behavior)
- **Interface** → **What you can do** (Capabilities / Contract)

This mental model works well in practice.

---

## 22. Summary (Slow Revision)

1. **Abstraction** defines contracts.
2. **Abstract methods** have no body.
3. **Abstract classes** mix behavior + contract + state.
4. **Interfaces** define pure capabilities (no state).
5. **Interfaces** allow multiple inheritance safely.
6. **Modern Interfaces** support static, default, and private methods.
7. **Functional Interfaces** enable Lambda expressions.
8. **JVM** resolves abstract & interface methods dynamically.
9. **Design choice** matters more than syntax.

---

## 23. Interview Questions

### 1. Why can’t abstract classes be instantiated?
**Answer:**
Because they are incomplete. They may contain abstract methods (contracts) with no implementation. Creating an object of an incomplete class would lead to runtime errors if those methods were called. The JVM creates a class metadata entry but prevents instance allocation for abstract classes to ensure type safety and logical correctness.

### 2. Why does Java separate abstract classes and interfaces?
**Answer:**
- **Abstract Classes** assume an "IS-A" relationship and allow shared state (fields), constructors, and partial implementation.
- **Interfaces** define "CAN-DO" capabilities (contracts) without state (no instance fields).
- Separating them avoids the "Diamond Problem" of multiple inheritance with state (which interfaces avoid by having no state) while allowing multiple inheritance of types (interfaces).

### 3. How does the JVM handle interface method calls (`invokeinterface`)?
**Answer:**
Unlike `invokevirtual` (used for class methods where the method index is fixed in the vtable), `invokeinterface` requires a more complex lookup. Since a class can implement multiple interfaces, the method's position in the method table is not constant across different implementing classes. The JVM searches the object's class method table at runtime to find the matching method, often using `itable` (interface table) caches for performance optimization.

### 4. What is a Functional Interface and how does it relate to Lambdas?
**Answer:**
- A **Functional Interface** is an interface with exactly **one abstract method**.
- It is the target type for **Lambda expressions**. Instead of creating an anonymous inner class, a lambda `(args) -> body` can be used to provide the implementation for that single abstract method inline.

### 5. When would you use a sealed interface over a normal interface?
**Answer:**
When you want to **restrict the hierarchy**. A normal interface is "open" for implementation by anyone. A `sealed` interface permits only specific named classes/interfaces to extend/implement it. This allows for better domain modeling (e.g., `Payment` type can only be `Card` or `Cash`, nothing else) and enables exhaustive pattern matching in switch statements (future Java versions).

### 6. Why are instance variables not allowed in interfaces?
**Answer:**
To avoid the **Diamond Problem** and ambiguity in multiple inheritance. If two interfaces had a variable with the same name, and a class implemented both, which variable would it inherit? By allowing only `static final` constants, Java ensures there is no per-instance state overlap or conflict handling needed for interfaces.
