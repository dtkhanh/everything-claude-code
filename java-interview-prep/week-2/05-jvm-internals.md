# Week 2 — Day 3–4: JVM Internals

> 📖 **Estimated reading time:** 40 minutes  
> 🎯 **Focus:** Memory model, GC algorithms, ClassLoader, JIT compilation

---

## 1. JVM Memory Areas

You call `new User()` a million times in a load test. Eventually:

```
java.lang.OutOfMemoryError: Java heap space
```

That tells you the heap is full — but which part? And what's the fix? The JVM splits memory into distinct regions, each with different behavior and different error signatures:

```
┌─────────────────────────────────────────────────────────────┐
│                         JVM Memory                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   HEAP (shared)                     │   │
│  │  ┌──────────────────┐  ┌────────────────────────┐  │   │
│  │  │  Young Generation │  │    Old Generation      │  │   │
│  │  │  ┌─────┐ ┌────┐  │  │  (Tenured Space)       │  │   │
│  │  │  │Eden │ │ S0 │  │  │  long-lived objects    │  │   │
│  │  │  │     │ ├────┤  │  │                        │  │   │
│  │  │  │     │ │ S1 │  │  │                        │  │   │
│  │  │  └─────┘ └────┘  │  │                        │  │   │
│  │  └──────────────────┘  └────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────┐  ┌────────────┐  ┌───────────────────────┐  │
│  │ Metaspace│  │ Code Cache │  │  Thread Stack (each)  │  │
│  │  (class  │  │ (JIT comp) │  │  - Stack frames       │  │
│  │  metadata│  │            │  │  - Local variables    │  │
│  │  statics)│  │            │  │  - Method calls       │  │
│  └──────────┘  └────────────┘  └───────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 🔑 Key Points
| Area | What's Stored | GC'd? | OOM Type |
|------|--------------|-------|----------|
| **Eden** | New objects | Yes (Minor GC) | java.lang.OutOfMemoryError: Java heap space |
| **Survivor (S0/S1)** | Survived > 1 GC | Yes (Minor GC) | same |
| **Old Gen** | Long-lived objects | Yes (Major GC) | same |
| **Metaspace** | Class metadata, static vars | Yes (rare) | OutOfMemoryError: Metaspace |
| **Stack** | Method frames, local vars | Automatic (frame pop) | StackOverflowError |
| **Code Cache** | JIT-compiled native code | Rarely | OutOfMemoryError: Code Cache |

### ❓ Interview Question
> "What's the difference between heap and stack memory in Java?"

### ✅ Model Answer
- **Stack**: Per-thread, stores method call frames, local variables, and method parameters. Memory is allocated/freed automatically (LIFO) as methods are called/return. Fast but limited size.
- **Heap**: Shared across all threads, stores **all objects** created with `new`. Managed by the GC. Larger but slower access.

```java
void example() {
    int x = 5;                    // x on STACK
    String s = "hello";           // s reference on STACK, "hello" object on HEAP (String Pool)
    User user = new User("Alice"); // user reference on STACK, User object on HEAP
}
// When method returns: x, s, user popped from stack; User object stays on heap until GC
```

### ⚠️ Gotcha: StackOverflowError vs OutOfMemoryError
```java
// StackOverflowError — too deep recursion
void recursive() { recursive(); } // each call adds a frame until stack full

// OutOfMemoryError: Java heap space — too many/large objects
List<byte[]> leak = new ArrayList<>();
while (true) leak.add(new byte[1024 * 1024]); // heap fills up
```

---

## 2. Garbage Collection

### 📌 Concept Summary
The GC automatically frees memory for objects that are no longer reachable. "Reachable" means there's a chain of references from a **GC Root** (static fields, stack variables, JNI references) to the object.

### 🔑 The GC Process (Generational)

**Minor GC (Young Generation):**
1. Eden fills up → trigger Minor GC
2. Mark live objects in Eden + one Survivor
3. Copy live objects to the other Survivor; increment **age counter**
4. Objects with age ≥ tenure threshold (default 15) → promoted to Old Gen
5. Eden + used Survivor cleared → very fast (milliseconds)

**Major GC / Full GC (Old Generation):**
- Old Gen fills → Major GC
- Much slower: entire heap must be scanned
- Causes **stop-the-world** pauses — application pauses completely

### 🔑 GC Algorithms

| Collector | Use Case | Pause | Throughput |
|-----------|----------|-------|------------|
| **Serial GC** | Single-core, small heaps | Long | Good |
| **Parallel GC** | Batch, throughput-focused (default Java 8) | Medium | Excellent |
| **G1GC** | Low-latency server apps (default Java 9+) | Short, predictable | Good |
| **ZGC** | Very low latency (<10ms), large heaps | Ultra-short | Good |
| **Shenandoah** | Similar to ZGC | Ultra-short | Moderate |

### 💻 JVM Flags
```bash
# Heap size
-Xms512m        # initial heap (set same as -Xmx to avoid resizing)
-Xmx2g          # max heap

# GC selection
-XX:+UseG1GC
-XX:+UseZGC

# G1 tuning
-XX:MaxGCPauseMillis=200      # target max pause (G1 tries to meet this)
-XX:G1HeapRegionSize=16m      # region size (1-32MB)
-XX:InitiatingHeapOccupancyPercent=45 # when to start concurrent marking

# GC logging (production must-have)
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=5,filesize=20m

# Print heap details on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/dumps/heap.hprof
```

### ❓ Interview Question
> "What is a memory leak in Java? I thought GC handles that automatically."

### ✅ Model Answer
A memory leak in Java occurs when objects are **still reachable** (GC roots hold references) but are **no longer needed** by the application logic. The GC can't free them because they're technically live. Common causes:

1. **Static collections**: `static Map<K,V> cache` that grows forever
2. **Listeners not removed**: event listeners registered but never deregistered
3. **ThreadLocal not cleared**: ThreadLocal in a thread pool — the thread lives forever, so does the stored value
4. **Inner class references**: non-static inner class holds implicit reference to outer class

```java
// Common memory leak: adding to a static cache without eviction
public class UserCache {
    private static final Map<Long, User> cache = new HashMap<>(); // grows forever ❌
    
    public static void cache(User user) {
        cache.put(user.getId(), user); // never removed
    }
    
    // Fix: use bounded cache
    private static final Map<Long, User> boundedCache = 
        Collections.synchronizedMap(new LinkedHashMap<>(1000, 0.75f, true) {
            protected boolean removeEldestEntry(Map.Entry<Long, User> eldest) {
                return size() > 1000; // evict when > 1000 entries
            }
        });
}
```

---

## 3. ClassLoader

### 📌 Concept Summary
ClassLoaders load `.class` files into the JVM. They form a **hierarchy** (delegation model): child loaders delegate to parent before loading themselves.

```
Bootstrap ClassLoader   (loads JDK core: java.lang, java.util, ...)
    ↑ parent
Extension ClassLoader   (loads JDK extensions)
    ↑ parent
Application ClassLoader (loads your classpath: src/, lib/)
    ↑ parent
Custom ClassLoaders     (Spring, OSGi, hot-reload frameworks)
```

### 💻 Code Example
```java
// Check which ClassLoader loaded a class
System.out.println(String.class.getClassLoader());          // null (Bootstrap, native)
System.out.println(ClassLoader.class.getClassLoader());     // null
System.out.println(MyService.class.getClassLoader());       // AppClassLoader

// Class.forName — dynamic loading
Class<?> cls = Class.forName("com.example.UserService");    // uses current CL
Object instance = cls.getDeclaredConstructor().newInstance();

// Custom ClassLoader (used by frameworks for hot-reload, plugins)
ClassLoader customLoader = new URLClassLoader(
    new URL[] { new URL("file:///plugins/my-plugin.jar") },
    Thread.currentThread().getContextClassLoader() // parent
);
Class<?> pluginClass = customLoader.loadClass("com.plugin.PluginImpl");
```

### ❓ Interview Question
> "Explain the delegation model of ClassLoader. What is `ClassNotFoundException` vs `NoClassDefFoundError`?"

### ✅ Model Answer
When asked to load a class, a ClassLoader first **delegates to its parent**. Only if the parent can't find it does the child try to load it. This prevents applications from overriding core JDK classes.

- `ClassNotFoundException`: thrown when `Class.forName()` or `loadClass()` can't find the class — class literally doesn't exist on the classpath
- `NoClassDefFoundError`: class **was** found at compile time but is **missing at runtime** — often a dependency not in the deployment artifact (missing JAR), or a class failed to initialize (static initializer threw exception)

---

## 4. JIT Compilation

### 📌 Concept Summary
The JVM interprets bytecode initially, then the **JIT compiler** identifies "hot" code (frequently executed) and compiles it to **native machine code** at runtime.

### 🔑 Key Points
- **C1 (Client)**: fast compile, less optimization — used during warmup
- **C2 (Server)**: slow compile, heavy optimization — used for hot methods
- **Tiered compilation** (default): C1 first, then C2 for hot methods
- JIT can perform: **inlining**, dead code elimination, loop unrolling, escape analysis

### 💻 JVM Ergonomics
```bash
# Check JIT compilation (verbose — use for debugging only)
-XX:+PrintCompilation

# Control inlining
-XX:MaxInlineSize=35       # inline methods <= 35 bytecodes
-XX:FreqInlineSize=325     # inline hot methods <= 325 bytecodes

# Escape analysis: if object doesn't escape a method → allocate on STACK (not heap!)
# This means: small short-lived objects may never hit the heap at all
-XX:+DoEscapeAnalysis      # enabled by default
```

### ❓ Interview Question
> "Why does Java application performance often improve after startup?"

### ✅ Model Answer
JVM **warmup**: initially, code runs interpreted (slow). The JIT compiler monitors execution and progressively compiles hot code to native machine code with aggressive optimizations. Additionally:
- Class loading and verification happen lazily
- JIT may deoptimize and reoptimize as runtime profile changes
- Connection pools, caches, and thread pools fully initialize

For latency-sensitive applications: use **Class Data Sharing (CDS)** and **AOT compilation** (GraalVM native-image) to reduce warmup time.

---

## 5. Common JVM Diagnostics

```bash
# Heap analysis
jmap -heap <pid>                        # heap summary
jmap -dump:format=b,file=heap.hprof <pid>  # heap dump

# Thread analysis
jstack <pid>                            # thread dump (detect deadlocks)
jstack <pid> | grep -A 20 "BLOCKED"    # find blocked threads

# GC monitoring
jstat -gcutil <pid> 1000 10             # GC stats every 1s, 10 times

# JVM flags in use
jinfo -flags <pid>

# Living heap histogram
jcmd <pid> GC.heap_info
jcmd <pid> VM.native_memory             # native memory usage

# VisualVM / JConsole — GUI tools for profiling
# IntelliJ Profiler / async-profiler — production profiling
```

---

*Next: [06-java9-21-features.md](./06-java9-21-features.md)*


