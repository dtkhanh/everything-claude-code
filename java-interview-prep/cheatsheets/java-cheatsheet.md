# Java Cheatsheet — Quick Reference

> Use before your interview for rapid review

---

## Collections Quick Reference

```java
// INTERFACES
List       → ArrayList (get O(1)), LinkedList (insert head O(1))
Set        → HashSet (O(1)), LinkedHashSet (ordered), TreeSet (sorted O(log n))
Map        → HashMap (O(1)), LinkedHashMap (ordered), TreeMap (sorted O(log n))
Queue/Deque → ArrayDeque (fast stack/queue), PriorityQueue (min-heap)

// THREAD-SAFE
ConcurrentHashMap    // best concurrent map, no null k/v
CopyOnWriteArrayList // read-heavy, snapshot on write
LinkedBlockingQueue  // producer-consumer, blocks on put/take

// IMMUTABLE (Java 9+)
List.of(), Set.of(), Map.of()         // null-prohibited, fixed size
List.copyOf(), Set.copyOf()           // defensive copy

// COMPLEXITY
ArrayList: get O(1), add O(1) amortized, remove middle O(n)
HashMap: get/put O(1) avg, worst O(log n) (treeified)
TreeMap: get/put O(log n)
PriorityQueue: peek O(1), add/poll O(log n)
```

---

## Streams Quick Reference

```java
// INTERMEDIATE (lazy)
filter(Predicate)          map(Function)             flatMap(Function<T,Stream>)
distinct()                 sorted()                  sorted(Comparator)
peek(Consumer)             limit(long)               skip(long)
takeWhile(Predicate) [9+]  dropWhile(Predicate) [9+]

// TERMINAL (eager, triggers pipeline)
collect(Collector)         forEach(Consumer)
count()                    reduce(identity, BinaryOperator)
findFirst()                findAny()                 // Optional
anyMatch()                 allMatch()                noneMatch() // boolean
min(Comparator)            max(Comparator)            // Optional
toArray()
toList() [16+]             // unmodifiable

// COLLECTORS
Collectors.toList()                    → ArrayList
Collectors.toUnmodifiableList()        → immutable
Collectors.toSet()                     → HashSet
Collectors.toMap(keyFn, valueFn)
Collectors.toMap(keyFn, valueFn, mergeFn)
Collectors.groupingBy(classifier)      → Map<K, List<V>>
Collectors.groupingBy(classifier, downstream)
Collectors.counting()
Collectors.joining(", ", "[", "]")
Collectors.partitioningBy(Predicate)   → Map<Boolean, List<T>>

// NUMERIC
mapToInt/Long/Double().sum()/.average()/.min()/.max()
IntStream.range(0, 10)    // 0..9
IntStream.rangeClosed(1, 10) // 1..10
```

---

## Optional Quick Reference

```java
Optional.of(value)                 // NPE if null
Optional.ofNullable(value)         // safe
Optional.empty()

// Consuming
.isPresent() / .isEmpty()          // check
.get()                             // ❌ avoid — throws if empty
.orElse(default)                   // eager — always evaluates default
.orElseGet(() -> compute())        // lazy — only if empty
.orElseThrow(NotFoundException::new)
.ifPresent(Consumer)
.ifPresentOrElse(Consumer, Runnable)  // [9+]

// Transforming
.map(Function)                     // T → Optional<R>
.flatMap(Function<T, Optional<R>>) // flatten nested Optional
.filter(Predicate)                 // Optional<T> → Optional<T> or empty

// Converting
.stream()                          // Optional<T> → Stream<T> [9+] (0 or 1 element)
```

---

## Concurrency Quick Reference

```java
// ATOMIC OPERATIONS
AtomicInteger.incrementAndGet()
AtomicInteger.compareAndSet(expect, update)   // CAS
AtomicReference.updateAndGet(fn)

// LOCKS
ReentrantLock lock = new ReentrantLock(true); // fair
lock.lock();
try { ... } finally { lock.unlock(); }
lock.tryLock(100, TimeUnit.MILLISECONDS)    // non-blocking

ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();    // multiple readers
rwLock.writeLock().lock();   // exclusive writer

// EXECUTORS
Executors.newFixedThreadPool(n)
Executors.newCachedThreadPool()
Executors.newSingleThreadExecutor()
Executors.newVirtualThreadPerTaskExecutor()  // [21+]
Executors.newScheduledThreadPool(n)

// COMPLETABLE FUTURE
CompletableFuture.supplyAsync(Supplier)            // async supplier
CompletableFuture.runAsync(Runnable)               // async void
.thenApply(Function)        // sync transform
.thenApplyAsync(Function)   // async transform
.thenCompose(Function<T,CF<U>>) // flatMap
.thenCombine(CF, BiFunction)    // combine two CFs
.exceptionally(Function<Throwable, T>) // error fallback
.handle(BiFunction<T, Throwable, R>)   // success + error
CompletableFuture.allOf(cf1, cf2, cf3) // wait all
CompletableFuture.anyOf(cf1, cf2)      // first to complete

// COUNTDOWN AND BARRIERS
CountDownLatch latch = new CountDownLatch(3);
latch.countDown();           // decrement
latch.await();               // wait until 0

CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("Go!"));
barrier.await();             // wait for all parties
```

---

## Java 8–21 Features Summary

```java
// RECORDS [16]
record Point(int x, int y) {}
Point p = new Point(3, 4);
p.x(); p.y();  // accessors
// Auto: equals, hashCode, toString

// SEALED CLASSES [17]
sealed interface Shape permits Circle, Rectangle {}
final class Circle implements Shape { ... }
final class Rectangle implements Shape { ... }

// PATTERN MATCHING [16,21]
if (obj instanceof String s && s.length() > 5) { ... }
switch (obj) {
    case Integer i -> "int: " + i;
    case String s  -> "str: " + s;
    default        -> "other";
}

// VIRTUAL THREADS [21]
Thread.ofVirtual().start(runnable);
Executors.newVirtualThreadPerTaskExecutor();
// spring.threads.virtual.enabled=true

// TEXT BLOCKS [15]
String sql = """
    SELECT * FROM users
    WHERE status = :status
    """;

// SWITCH EXPRESSION [14]
String result = switch (day) {
    case MON, TUE -> "weekday";
    case SAT, SUN -> "weekend";
};
```

---

## Common Exceptions Quick Reference

| Exception | Cause | Prevention |
|-----------|-------|-----------|
| `NullPointerException` | Calling method on null | Optional, null checks, `Objects.requireNonNull` |
| `ClassCastException` | Wrong type cast | `instanceof` check, generics |
| `ArrayIndexOutOfBoundsException` | Invalid array index | Bounds check |
| `ConcurrentModificationException` | Modifying collection during iteration | Use iterator.remove(), CopyOnWriteArrayList |
| `StackOverflowError` | Infinite/too-deep recursion | Add base case, increase stack |
| `OutOfMemoryError: Java heap space` | Too many/large objects | Profile memory, fix leaks |
| `OutOfMemoryError: Metaspace` | Too many classes loaded | Check dynamic class generation |
| `OptimisticLockException` | Concurrent modification detected | Retry logic, or switch to pessimistic |
| `LazyInitializationException` | Hibernate lazy load outside session | `JOIN FETCH`, `@Transactional` on service |

---

## equals/hashCode Contract

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;               // 1. identity shortcut
    if (!(o instanceof MyClass that)) return false; // 2. type check + cast
    return Objects.equals(field1, that.field1) &&   // 3. field comparison
           Objects.equals(field2, that.field2);
}

@Override
public int hashCode() {
    return Objects.hash(field1, field2);      // consistent with equals
}

// Rules:
// equals → same hashCode (MUST)
// same hashCode → equals might be false (OK, hash collision)
// Never use mutable fields in hashCode if object stored in Map/Set
```

---

## String Operations

```java
// Comparison — NEVER use == for String
s1.equals(s2)
s1.equalsIgnoreCase(s2)
s1.compareTo(s2)       // negative/0/positive

// Check
s.isEmpty()            // length == 0
s.isBlank()            // empty or whitespace only [11+]
s.contains("sub")
s.startsWith("pre")
s.endsWith("suf")
s.matches("regex")

// Transform
s.toLowerCase() / toUpperCase()
s.trim()               // removes ASCII whitespace
s.strip()              // removes Unicode whitespace [11+]
s.replace("a", "b")    // all occurrences
s.replaceAll("regex", "replacement")
s.split(",")
s.substring(start, end) // end exclusive

// Build
String.valueOf(obj)
String.format("%s = %d", key, val)
"%s = %d".formatted(key, val)  // instance method [15+]
String.join(", ", "a", "b", "c")

// StringBuilder
var sb = new StringBuilder();
sb.append("hello").append(" world");
sb.insert(5, ",");
sb.delete(5, 6);
sb.reverse();
sb.toString();

// String Pool
"literal"             // goes to pool
new String("literal") // heap, NOT pool
s.intern()            // get pool ref (or add)
```

