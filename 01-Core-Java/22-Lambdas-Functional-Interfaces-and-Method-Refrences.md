# Lambdas, Functional Interfaces & Method References (Bytecode + JVM View)


This document explains Java 8+ lambdas not as syntax sugar, but as a runtime feature enabled by the JVM. Understanding this removes confusion around lambdas, functional interfaces, and method references.

---

## 1. Why Java Introduced Lambdas

Before Java 8, passing behavior required verbose anonymous classes:

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
}).start();
```

**Problems:**
*   Too much boilerplate
*   Hides intent (hard to read)
*   Generates many anonymous class files (`Class$1.class`, etc.)

**Java wanted:**
*   **Behavior as data**
*   This is why lambdas were introduced.

---

## 2. What a Lambda Really Is

**A lambda is NOT:**
*   An anonymous class
*   A normal class instance
*   Just a shortcut

**A lambda IS:**
*   An implementation of a functional interface, created at **runtime**.

**Example:**
```java
Runnable r = () -> System.out.println("Hello");
```

**Important:**
*   A lambda always has a **target type**.
*   That target must be a **functional interface**.

---

## 3. Functional Interface (Real Meaning)

A functional interface:
*   Has **exactly ONE** abstract method.

**Example:**
```java
@FunctionalInterface
interface Task {
    void execute();
}
```

**Why only one?**
Because the JVM must know unambiguously which method the lambda is implementing.

---

## 4. Why Lambdas Require Functional Interfaces

**This is illegal:**
```java
interface A {
    void m1();
    void m2();
}
```

**Now:**
```java
A a = () -> { }; // ambiguous
```
The JVM cannot decide which method to bind the lambda body to.

**Rules:**
*   Exactly **one** abstract method.
*   `default` methods don’t count.
*   `static` methods don’t count.
*   Methods from `java.lang.Object` (toString, equals, etc.) don't count.

---

## 5. Lambda vs Anonymous Class (Critical Difference)

| Aspect | Lambda | Anonymous Class |
| :--- | :--- | :--- |
| **Class creation** | No real class file generated | New `.class` file generated |
| **`this` keyword** | Refers to **enclosing object** | Refers to **anonymous class instance** |
| **Memory** | Lightweight | Heavier (new object overhead) |
| **JVM optimization** | Very high (JIT inlining) | Limited |

Lambdas are **not objects** in the traditional sense until runtime.

---

## 6. How JVM Implements Lambdas (VERY IMPORTANT)

Java 8 introduced the **`invokedynamic`** bytecode instruction.

Instead of generating class files at compile time, the JVM:
1.  **Defers** lambda implementation creation until runtime.
2.  **Binds** behavior dynamically.
3.  **Optimizes** aggressively.

---

## 7. Lambda Execution Flow

For:
```java
Runnable r = () -> System.out.println("Hi");
```

**JVM Flow:**
1.  Bytecode contains `invokedynamic`.
2.  JVM calls **`LambdaMetafactory`**.
3.  Lambda implementation is generated **dynamically** in memory.
4.  Implementation is **cached** (if possible) for reuse.
5.  Method executes.

**No permanent `.class` file is created on disk.**

---

## 8. Why Lambdas Are Faster Than Anonymous Classes

Because:
*   **No extra class loading** (avoids disk I/O for class files).
*   **Fewer objects** created (if non-capturing).
*   **Better JIT inlining** (code is easier for the JVM to understand).
*   **Improved escape analysis**.

This is why modern Java prefers lambdas.

---

## 9. Capturing Variables (Effectively Final Rule)

```java
int x = 10;
Runnable r = () -> System.out.println(x);
```

**Why must `x` be effectively final?**

Because:
1.  Local variables live on the **stack**.
2.  The lambda may execute **later** (after the method returns and the stack frame is gone).
3.  The JVM **copies** the value of `x` into the lambda instance.
4.  If `x` could change, the copy inside the lambda would be out of sync, leading to confusion and concurrency issues.

---

## 10. Lambda Memory Model

**Captured variables:**
*   Are **copied** into the lambda instance.
*   Are treated as **final fields**.
*   Are **safely published** to other threads.

This links directly to:
*   **Immutability**
*   **Concurrency safety**

---

## 11. Method References (What They Really Are)

A Method Reference is:
*   A **compact form** of a lambda.
*   Used when a lambda does nothing but call an existing method.

**Examples:**
```java
System.out::println  // equivalent to x -> System.out.println(x)
String::length       // equivalent to s -> s.length()
obj::method          // equivalent to () -> obj.method()
```

They do not add new behavior — they only reuse existing methods.

---

## 12. Types of Method References

1.  **Static method**
    *   `ClassName::staticMethod`
    *   Ex: `Math::max`

2.  **Instance method (specific object)**
    *   `object::method`
    *   Ex: `System.out::println` (calling println on the specific `System.out` object)

3.  **Instance method (arbitrary object)**
    *   `ClassName::method`
    *   Ex: `String::toLowerCase` (calling toLowerCase on the string passed as argument)

4.  **Constructor reference**
    *   `ClassName::new`

---

## 13. Constructor References (JVM Perspective)

```java
Supplier<User> s = User::new;
```

**Means:**
*   A lambda that calls the constructor.
*   Runtime binding via `invokedynamic`.
*   No special syntax at the JVM level; it's treated just like a method call.

---

## 14. Stateless Nature of Lambdas

Lambdas are designed to be:
*   **Stateless**
*   **Immutable**
*   **Short-lived**

**Stateful lambdas:**
*   Cause **concurrency bugs**.
*   Reduce **predictability**.
*   Are **strongly discouraged**.

---

## 15. Common Functional Interfaces in Java

These are found in `java.util.function`.

| Interface | Descriptor | Method Signature | Use Case |
| :--- | :--- | :--- | :--- |
| **`Runnable`** | `() -> void` | `void run()` | Run a task without return value. |
| **`Callable<V>`** | `() -> V` | `V call() throws Exception` | Run a task that returns a result. |
| **`Supplier<T>`** | `() -> T` | `T get()` | Provide/Factory a value. |
| **`Consumer<T>`** | `(T) -> void` | `void accept(T t)` | Process a value (e.g., printing). |
| **`Function<T, R>`** | `(T) -> R` | `R apply(T t)` | Transform T to R. |
| **`Predicate<T>`** | `(T) -> boolean` | `boolean test(T t)` | Test a condition. |
| **`Comparator<T>`** | `(T, T) -> int` | `int compare(T o1, T o2)` | Compare two objects. |

**All follow the "One abstract method" rule.**

---

## 16. Why Streams Depend on Lambdas

Streams require:
*   **Stateless operations**
*   **Deferred execution** (lazy evaluation)
*   **Composable behavior**

Lambdas provide:
*   Clean way to pass behavior.
*   JVM-optimizable execution structure.

---

## 17. Common Misconceptions (Cleared)

*   ❌ **Lambda = syntax sugar**
    *   ✔ **Truth**: Lambdas can change bytecode (invokedynamic) and the execution model.
*   ❌ **Lambda = anonymous class**
    *   ✔ **Truth**: Implemented completely differently (no class file vs. class file).
*   ❌ **Lambdas are always faster**
    *   ✔ **Truth**: Usually faster (startup & linking overhead exists, but runtime performance is high).

---

## 18. Mental Model to Remember

*   **Lambda** = Behavior
*   **Functional Interface** = Contract
*   **invokedynamic** = Runtime Binding

If this is clear, lambdas will never confuse you.

---

## 19. Summary (Slow Revision)

*   Lambdas pass **behavior as data**.
*   Require **functional interfaces**.
*   Use **`invokedynamic`** instruction.
*   Do **not** generate normal class files.
*   Capture variables **safely** (effectively final).
*   Method references are just **shorthand lambdas**.
*   JVM **heavily optimizes** lambdas.

---

## 20. Interview Questions & Answers

**Q1: How are lambdas implemented in the JVM?**
**Answer:** Lambdas are implemented using the `invokedynamic` bytecode instruction. Unlike anonymous classes, they do not generate a separate `.class` file at compile time. Instead, the `LambdaMetafactory` creates the implementation dynamically at runtime.

**Q2: Why must a functional interface have only one abstract method?**
**Answer:** The JVM needs to unambiguously identify which method the lambda expression corresponds to. If there were multiple abstract methods, the compiler/JVM wouldn't know which one the lambda body is implementing.

**Q3: What is the difference between a lambda and an anonymous class?**
**Answer:**
1.  **Identity:** Anonymous classes create a new object instance with its own `this`; lambdas do not (lexical scoping).
2.  **Compilation:** Anonymous classes generate a physical `.class` file; lambdas use `invokedynamic` and `LambdaMetafactory`.
3.  **Performance:** Lambdas are generally more lightweight and memory-efficient.

**Q4: What is `invokedynamic`?**
**Answer:** `invokedynamic` is a bytecode instruction added in Java 7 (used heavily in Java 8+) that allows the JVM to defer the binding of a method call until runtime. This enables dynamic languages to run on the JVM and supports efficient implementation of lambdas.

**Q5: Why must captured variables be effectively final?**
**Answer:** Local variables are stored on the stack, which is cleared when the method returns. However, a lambda instance might exist long after the method finishes. To handle this, Java **copies** the variable's value into the lambda. If the variable were mutable, the lambda's copy and the original variable could get out of sync, leading to unpredictable behavior. Enforcing "effectively final" ensures consistency.