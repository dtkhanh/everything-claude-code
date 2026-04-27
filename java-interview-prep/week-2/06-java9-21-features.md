# Week 2 — Day 5–7: Java 9–21 Modern Features

> 📖 **Estimated reading time:** 35 minutes  
> 🎯 **Focus:** Records, Sealed classes, Pattern matching, Switch expressions, Virtual Threads

---

## 1. Records (Java 16) — Immutable Data Classes

### 📌 Concept Summary
`record` is a special class for **immutable data carriers**. Auto-generates constructor, getters, `equals`, `hashCode`, and `toString`.

```java
// Traditional DTO — 40+ lines of boilerplate
// Record — same semantics, 1 line
public record UserDTO(Long id, String name, String email) {}

UserDTO dto = new UserDTO(1L, "Alice", "alice@example.com");
dto.id();     // getter (no "get" prefix)

// Records can have custom validation
public record UserDTO(Long id, String name, String email) {
    public UserDTO {  // compact constructor
        Objects.requireNonNull(id, "id must not be null");
        name = name.trim();
    }
    public String displayName() { return name + " <" + email + ">"; }
}
```

### ❓ Interview Question
> "When should you use a `record` instead of a regular class?"

### ✅ Model Answer
Use a `record` for **pure data carriers** (DTOs, value objects, API responses) when you want immutability by default and structural equality. Don't use for JPA `@Entity` (requires no-arg constructor and mutable setters), mutable state, or inheritance from another class.

---

## 2. Sealed Classes (Java 17) — Restricted Hierarchy

```java
public sealed interface PaymentResult
    permits SuccessPayment, FailedPayment, PendingPayment {}

public record SuccessPayment(String transactionId, double amount) implements PaymentResult {}
public record FailedPayment(String reason, String errorCode) implements PaymentResult {}
public record PendingPayment(String waitingFor, Instant estimatedTime) implements PaymentResult {}

// Exhaustive switch — no default needed!
String message = switch (result) {
    case SuccessPayment s -> "Paid " + s.amount();
    case FailedPayment f  -> "Failed: " + f.reason();
    case PendingPayment p -> "Waiting for: " + p.waitingFor();
};
```

---

## 3. Pattern Matching (Java 16–21)

```java
// instanceof pattern matching (Java 16)
if (obj instanceof String s && s.length() > 5) {
    System.out.println("Long string: " + s);
}

// Switch pattern matching (Java 21)
static String format(Object obj) {
    return switch (obj) {
        case Integer i -> "int: " + i;
        case String s  -> "String[" + s.length() + "]: " + s;
        case null      -> "null";
        default        -> "Other: " + obj.getClass().getName();
    };
}

// Record deconstruction (Java 21)
record Point(int x, int y) {}
record Circle(Point center, double radius) {}

switch (shape) {
    case Circle(Point(int x, int y), double r) ->
        System.out.println("Circle at (" + x + "," + y + ") r=" + r);
}
```

---

## 4. Virtual Threads (Java 21) — Project Loom

### 📌 Concept Summary
Virtual Threads are JVM-managed lightweight threads. You can create **millions** cheaply. They shine for **I/O-bound** workloads — when a virtual thread blocks on I/O, the carrier OS thread is freed.

```java
// ExecutorService with virtual threads
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofMillis(100)); // parks virtual thread, frees OS thread
            return fetchFromDB();
        });
    }
}

// Spring Boot 3.2+ — enable in application.properties
// spring.threads.virtual.enabled=true
```

### ❓ Interview Question
> "What are Virtual Threads and when would you use them?"

### ✅ Model Answer
Platform threads map 1:1 to OS threads (~1MB each, limited to ~10K). Virtual threads are JVM-managed (~few KB, millions possible). When a virtual thread blocks on I/O, the JVM unmounts it from its carrier OS thread, allowing that thread to run other virtual threads. This means synchronous-style I/O code can scale like async code.

Use for: web servers with many concurrent requests, microservices with lots of DB/HTTP calls.
Don't use for: CPU-bound tasks (still need real OS threads), tasks that hold `synchronized` locks during I/O (pin the carrier thread — use `ReentrantLock` instead).

---

## 5. Other Key Features

### Collections Factories (Java 9)
```java
List<String> list = List.of("a", "b", "c");        // immutable
Set<String> set = Set.of("x", "y", "z");
Map<String,Integer> map = Map.of("one", 1, "two", 2);
List<String> copy = List.copyOf(mutableList);       // defensive copy
```

### Switch Expressions (Java 14)
```java
String dayType = switch (day) {
    case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "WEEKDAY";
    case SATURDAY, SUNDAY -> "WEEKEND";
};
```

### Text Blocks (Java 15)
```java
String sql = """
        SELECT u.id, u.name
        FROM users u
        WHERE u.status = :status
        """;
```

### Stream Enhancements
```java
// Java 16: Stream.toList() — cleaner terminal
List<String> names = users.stream().map(User::getName).toList();

// Java 9: takeWhile
Stream.of(1,2,3,4,5).takeWhile(n -> n < 4).toList(); // [1,2,3]

// Java 9: Stream.iterate with predicate
Stream.iterate(0, n -> n < 10, n -> n + 2).toList(); // [0,2,4,6,8]
```

---

## 6. Java Version Summary

| Version | Key Features |
|---------|-------------|
| **Java 8** | Lambdas, Stream, Optional, java.time, CompletableFuture |
| **Java 11** | String.strip/isBlank/lines, HTTP Client API — **LTS** |
| **Java 17** | Records, Sealed classes, Pattern matching, Text blocks — **LTS** |
| **Java 21** | Virtual threads, Pattern matching switch, Record patterns — **LTS** |

---

*Next: [Week 3 — Spring Boot Core](../week-3/07-spring-boot-core.md)*
