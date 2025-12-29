# Inheritance, Method Overriding and Dynamic Dispatch (JVM View)


Most developers memorize inheritance rules.
This document explains how inheritance actually works at runtime, how the JVM resolves method calls, and why polymorphism is possible only because of JVM internals.

---

## 1. Why Inheritance Exists (Real Reason)

Inheritance is not only about code reuse.

Its real purpose is:

**Substitutability + runtime behavior change**

Meaning:

- A child object can be treated as a parent
- Behavior can still change at runtime

Without JVM support, inheritance would be useless.

---

## 2. Basic Inheritance Example

```java
class Animal {
    void sound() {
        System.out.println("Animal sound");
    }
}

class Dog extends Animal {
    void sound() {
        System.out.println("Dog barks");
    }
}
```

This simple code triggers dynamic method resolution inside the JVM.

---

## 3. Object Memory Layout with Inheritance

When you create:

```java
Dog d = new Dog();
```

Heap memory layout:

```
Dog object
 ├── Object Header
 ├── Animal fields
 └── Dog fields
```

Important points:

- Only one object exists
- Parent part is embedded inside child
- Fields are laid out top-down (parent → child)

---

## 4. Where Methods Actually Live

Methods:

- Do NOT live inside objects
- Are stored in class metadata (Metaspace)

Inheritance does NOT duplicate methods.

Instead:

- Each class has method metadata
- JVM decides which method to execute

---

## 5. Reference Type vs Object Type (Most Important Concept)

```java
Animal a = new Dog();
```

Two different types exist:

| Type | Description |
|------|-------------|
| Reference type | `Animal` (compile-time) |
| Object type | `Dog` (runtime) |

This single distinction explains most OOP confusion.

---

## 6. Compile-Time Binding (Fields and Static Members)

Fields and static members are resolved at compile time.

Example:

```java
class Animal {
    int x = 10;
}

class Dog extends Animal {
    int x = 20;
}

Animal a = new Dog();
System.out.println(a.x);
```

Output:

```
10
```

**Why?**

Fields depend on reference type, not object type.

Same rule applies to:

- Static variables
- Static methods

---

## 7. Method Overriding (Runtime Binding)

Instance methods are resolved at runtime.

Example:

```java
Animal a = new Dog();
a.sound();
```

Output:

```
Dog barks
```

Because JVM uses dynamic dispatch.

---

## 8. What Is Dynamic Dispatch?

Dynamic dispatch means:

**JVM decides which method to call at runtime, based on the actual object type.**

This is the core of polymorphism.

---

## 9. How JVM Implements Dynamic Dispatch

Each class maintains a **virtual method table (vtable)**.

Simplified view:

```
Animal vtable
sound() → Animal.sound

Dog vtable
sound() → Dog.sound
```

When calling:

```java
a.sound();
```

JVM process:

1. Identify object type → `Dog`
2. Look up Dog's vtable
3. Execute `Dog.sound()`

---

## 10. Why Static Methods Are Not Overridden

Static methods:

- Belong to class
- Resolved at compile time
- Do not participate in vtable lookup

So static methods are:

**Hidden, not overridden**

---

## 11. Why final Methods Cannot Be Overridden

If a method is `final`:

- JVM knows implementation will not change
- JIT can inline aggressively
- Performance improves

This directly connects to JIT optimizations.

---

## 12. Why private Methods Are Not Overridden

Private methods:

- Are not visible to subclasses
- Are not part of vtable
- Resolved at compile time

They are **redefined, not overridden**.

---

## 13. Constructors and Inheritance

Constructors are not inherited.

However:

**Parent constructor always executes first**

Example:

```java
Dog() {
    super(); // implicit
}
```

Reason:

Parent portion of object must be initialized before child.

---

## 14. Method Call Resolution (Step-by-Step)

For:

```java
Animal a = new Dog();
a.sound();
```

Execution flow:

1. Compiler checks `Animal` has `sound()` (compile-time check)
2. JVM inspects object type → `Dog`
3. JVM invokes `Dog.sound()`

**Compile-time safety + runtime behavior.**

---

## 15. Polymorphism Is JVM-Driven

Polymorphism works because:

- Object header points to class metadata
- JVM uses that pointer to resolve methods
- Dynamic dispatch is built into JVM

**No JVM support → no polymorphism.**

---

## 16. Common Misconceptions (Cleared)

| Misconception | Reality |
|---------------|---------|
| "Overriding replaces parent method" | Both methods exist, JVM chooses at runtime |
| "Static methods are polymorphic" | They are compile-time bound |
| "Fields participate in polymorphism" | They do not |

---

## 17. Core Mental Model

```
Reference type → what compiler allows
Object type    → what JVM executes
```

Remember this, and OOP becomes intuitive.

---

## 18. What We Did NOT Cover Yet

- Interfaces
- Abstract classes
- Multiple inheritance

These will be covered next.

---

## 19. Summary

| Concept | Key Point |
|---------|-----------|
| Inheritance | Embeds parent inside child |
| Object count | One object, not multiple |
| Method storage | Live in class metadata |
| Fields and static members | Compile-time binding |
| Instance methods | Runtime binding |
| Dynamic dispatch | Uses vtables |
| Behavior determinant | Object type determines behavior |

---

## 20. Interview Questions

### Q1: How does JVM implement polymorphism?

**Answer:** The JVM implements polymorphism through dynamic dispatch using virtual method tables (vtables). Each class has a vtable that maps method signatures to their implementations. When a method is called on an object, the JVM:
1. Reads the object header to determine the actual class type
2. Looks up the vtable for that class
3. Finds the correct method implementation
4. Executes that method

This allows subclass methods to be invoked even when the reference type is a parent class.

---

### Q2: What is the difference between reference type and object type?

**Answer:**
- **Reference type** is the declared type of the variable (known at compile time). It determines which methods and fields the compiler allows you to access.
- **Object type** is the actual type of the object in memory (known at runtime). It determines which method implementation is executed.

Example:
```java
Animal a = new Dog(); // Reference type: Animal, Object type: Dog
```

The compiler uses `Animal` to check validity, but JVM uses `Dog` to execute methods.

---

### Q3: Why are static methods not overridden?

**Answer:** Static methods are not overridden because:
1. They belong to the class, not to object instances
2. They are resolved at compile time based on the reference type
3. They are not included in the vtable (virtual method table)

When a subclass defines a static method with the same signature as the parent, it is called **method hiding**, not overriding. The method called depends on the reference type, not the object type.

---

### Q4: Why don't fields participate in polymorphism?

**Answer:** Fields are resolved at compile time based on the reference type, not the object type. This is because:
1. Field access is determined by the compiler statically
2. Fields do not go through the vtable mechanism
3. Both parent and child fields exist in memory simultaneously (field hiding)

If `Animal` has field `x` and `Dog` also has field `x`, accessing `a.x` where `a` is of type `Animal` will always return `Animal.x`, regardless of whether the object is actually a `Dog`.

---

### Q5: What is dynamic dispatch?

**Answer:** Dynamic dispatch is the mechanism by which the JVM determines at runtime which method implementation to invoke based on the actual object type, not the reference type. It is the foundation of runtime polymorphism in Java.

The process:
1. At compile time, the compiler verifies the method exists in the reference type
2. At runtime, the JVM checks the object's actual class
3. The JVM looks up the vtable for that class
4. The correct method implementation is invoked

---

### Q6: Why does the parent constructor run first?

**Answer:** The parent constructor runs first because:
1. The child object contains the parent portion embedded within it
2. The parent portion must be fully initialized before the child can use inherited fields or methods
3. Java enforces this by implicitly or explicitly calling `super()` as the first statement in any constructor

This ensures a proper initialization order: `Object` → parent class → child class, guaranteeing the object is in a consistent state at each level of the hierarchy.