# Java ClassLoader System — Deep JVM Internals + Practical Usage


The ClassLoader subsystem is responsible for loading Java classes into memory.
Understanding it is crucial for debugging classpath issues, working with frameworks like Spring Boot, building modular systems, and mastering JVM internals.

This document covers class loaders in depth: hierarchy, parent delegation, custom loaders, and Spring Boot behavior.

## 1. What Is a ClassLoader?

A ClassLoader is responsible for converting a `.class` file into a `Class<?>` object stored in the JVM Method Area.

It loads classes on demand, not during JVM startup.

**Why lazy loading?**
- faster startup
- load only needed classes
- dynamic class loading
- plugin systems
- security

Every Java program relies on ClassLoaders, even if developers never see them directly.

## 2. ClassLoader Hierarchy (Top → Bottom)

```
Bootstrap ClassLoader
        ↓
Platform (Extension) ClassLoader
        ↓
Application (System) ClassLoader
```

This is the backbone of Java’s loading mechanism.

## 3. Bootstrap ClassLoader (Root Loader)

- Written in native C
- Part of the JVM
- Loads core Java classes:
  - `java.lang.*`
  - `java.util.*`
  - `java.io.*`
  - `java.net.*`
- Loads from `$JAVA_HOME/lib`

This loader cannot be accessed from Java code.

Example:
```java
System.out.println(String.class.getClassLoader());
// null → Bootstrap loaded it
```

## 4. Platform (Extension) ClassLoader

Responsible for loading:
- JDK extension libraries
- Security modules
- Crypto libraries
- Some parts of the Java platform API

Example:
```java
System.out.println(javax.crypto.Cipher.class.getClassLoader());
// jdk.internal.loader.ClassLoaders$PlatformClassLoader
```

## 5. Application (System) ClassLoader

This class loader loads:
- Your compiled classes (`/target/classes`)
- External libraries (JAR files)
- Resources on your classpath

This is the default class loader for user code.

Example:
```java
System.out.println(MyClass.class.getClassLoader());
// AppClassLoader
```

## 6. Parent Delegation Model (Very Important)

Before loading a class, a class loader first delegates the request to its parent.

Flow:
```
Application
   ↓
Platform
   ↓
Bootstrap
```

### Why does parent delegation exist?

1. **Security**
   - Prevents overriding core Java classes.
   - Example: User cannot define a fake `java.lang.String`.

2. **Consistency**
   - Ensures only one definition of core classes exists in JVM.

3. **Type Safety**
   - A class loaded by two different loaders is treated as two different types.

## 7. How Class Loading Works Internally (3 Phases)

When JVM loads a class, it follows this sequence:

### Phase 1: Loading
- ClassLoader reads `.class` file bytes
- Creates a `Class<?>` object in Method Area
- Chooses parent loader

### Phase 2: Linking
Linking has three steps:

**a) Verification**
- Ensures:
  - correct bytecode format
  - no illegal instructions
  - type safety
- Prevents JVM crashes or security issues.

**b) Preparation**
- Allocates memory for static variables
- Sets default values
  - Example: `static int x; // default 0`

**c) Resolution**
- Converts symbolic references to actual memory references.

### Phase 3: Initialization
- Executes:
  - static initializers
  - static variable assignments

Example:
```java
static {
    System.out.println("Class Initialized");
}
```

## 8. Custom ClassLoaders (Powerful Feature)

Java allows you to create your own class loaders for custom behavior:

```java
class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) {
        // read bytes → defineClass()
        return defineClass(name, byteArray, 0, byteArray.length);
    }
}
```

**Why use Custom ClassLoaders?**
1. **Plugin systems**: Load external code dynamically (example: IDE plugins).
2. **Hot reloading engines**: Spring Boot devtools uses custom loaders for fast refresh.
3. **Encrypted class loading**: Decode encrypted `.class` files.
4. **Isolated module loading**: Different modules use different versions of same class.

## 9. ClassLoaders in Spring Boot (Very Practical)

Spring Boot uses a `LaunchedURLClassLoader`, not the regular `AppClassLoader`.

**Why?**
Because Spring Boot’s fat JAR structure is not a standard classpath layout:
```
app.jar
  ├── BOOT-INF/classes
  └── BOOT-INF/lib/*.jar
```

Spring Boot needs:
- nested JAR loading
- isolated classpaths
- custom resource resolution
- reloading engine for DevTools

### DevTools Hot Reload
DevTools separates:
- application classes → in a child class loader
- third-party libraries → in parent loader

This allows:
- instant restart
- no reload of big libraries (Spring, Hibernate, Jackson)

## 10. ClassLoader Problems & Errors

### 1) ClassNotFoundException
- Class not present in classpath.
- Triggered by: `Class.forName("abc.Test");`

### 2) NoClassDefFoundError
- Class was found during compile, but not available at runtime.
- Caused by:
  - missing JAR
  - wrong classpath
  - class loader isolation

### 3) ClassCastException due to ClassLoader

Example:
```java
A obj = (A) loader2.loadClass("A").newInstance();
```

Even if both are `A`, JVM sees them as two different classes:
`A (loaded by Loader1) != A (loaded by Loader2)`

Common in:
- containers (Tomcat WAR loading)
- plugin frameworks
- microservices using isolation

## 11. Real-World Use Cases of ClassLoaders

1. **Spring Boot Auto Configuration**
   - ClassLoader is used to discover `META-INF/spring.factories`, configuration classes, and bean definitions.

2. **Application Servers (Tomcat, WebLogic)**
   - Each WAR file gets its own child class loader → isolation.

3. **Big Data Systems**
   - Dynamic loading of jobs & filters.

4. **IDEs**
   - Eclipse & IntelliJ load plugins with custom class loaders.

5. **Monitoring Agents**
   - Java agents use custom class loaders to instrument bytecode.

## 12. Summary (1-Minute Revision)

- ClassLoaders load `.class` files into JVM memory
- Bootstrap → Platform → Application hierarchy
- Parent delegation prevents malicious overriding
- Class loading phases: Loading, Linking, Initialization
- Custom ClassLoaders enable dynamic behavior
- Spring Boot uses a special loader for nested JARs
- ClassNotFoundException = not in classpath
- NoClassDefFoundError = found at compile time, missing at runtime
- Two classes loaded by different class loaders are NOT equal

## 13. Interview Questions

1. **Explain the ClassLoader hierarchy in Java.**
   - **Bootstrap** (Core JDK) -> **Platform** (Extensions) -> **Application** (Classpath). Each delegates to its parent before loading.
2. **Why is the parent delegation model important?**
   - For security (prevents core class overriding) and consistency (ensures unique class definitions).
3. **What is the difference between ClassNotFoundException and NoClassDefFoundError?**
   - **ClassNotFoundException**: Checked exception when explicit loading fails (e.g., `Class.forName()`).
   - **NoClassDefFoundError**: Runtime error when a class available at compile time is missing at runtime (e.g., missing dependency JAR).
4. **How does Spring Boot load nested JARs?**
   - It uses a custom `LaunchedURLClassLoader` that can read `.jar` files inside the main `app.jar` (BOOT-INF/lib), which standard loaders cannot do.
5. **Can you override java.lang.String using a custom ClassLoader? Why not?**
   - No. Due to parent delegation, the request goes to Bootstrap Loader first, which finds the real `String`. And `java.*` packages are protected by the JVM (Prohibited package name).
6. **What are the three phases of class loading?**
   - **Loading** (Bytecode to Class object), **Linking** (Verification, Preparation, Resolution), and **Initialization** (Static blocks).
7. **How do custom ClassLoaders work?**
   - By extending `ClassLoader` and overriding `findClass()`. They can bypass parent delegation or load from non-standard sources (network, encrypted files).
8. **Why can two identical classes loaded by different loaders not be cast to each other?**
   - In JVM, class identity is defined by `ClassName + ClassLoader`. If loaders differ, they are distinct types, causing `ClassCastException`.