# Week 2 — Day 5–7: Java 9–21 Modern Features

>  **Estimated reading time:** 35 minutes  
>  **Focus:** Records, Sealed classes, Pattern matching, Switch expressions, Virtual Threads

---

## 1. Records (Java 16) — Immutable Data Classes

###  Concept Summary
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

###  Concept Summary
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

## 6. Java 11 — String Enhancements & HTTP Client

### String API (Java 11) — Frequently Asked
```java
String s = "  Hello World  ";

// strip() vs trim() — strip() is Unicode-aware
s.strip();           // "Hello World"    ← use this, not trim()
s.stripLeading();    // "Hello World  "
s.stripTrailing();   // "  Hello World"
s.trim();            // "Hello World"    ← only removes ASCII whitespace

// isBlank() — true if empty or contains only whitespace
"".isBlank();         // true
"  ".isBlank();       // true
" a ".isBlank();      // false
// vs isEmpty(): "  ".isEmpty() → false (has chars), "  ".isBlank() → true

// lines() — split by line terminators, returns Stream<String>
"line1\nline2\nline3".lines().toList(); // ["line1", "line2", "line3"]
// Better than split("\\n") for multi-platform line endings (\r\n vs \n)

// repeat() — string multiplication
"-".repeat(20); // "--------------------"
"ab".repeat(3); // "ababab"
```

### HTTP Client (Java 11) — Replaces HttpURLConnection
```java
// Modern, supports HTTP/2, async, non-blocking
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(10))
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build();

// Synchronous request
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.advahealth.com/patients/123"))
    .header("Authorization", "Bearer " + token)
    .header("Content-Type", "application/json")
    .timeout(Duration.ofSeconds(30))
    .GET()
    .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
int statusCode = response.statusCode();  // 200
String body = response.body();           // JSON string

// Async request — non-blocking
CompletableFuture<HttpResponse<String>> future = client.sendAsync(
    request,
    HttpResponse.BodyHandlers.ofString()
);
future.thenApply(HttpResponse::body)
      .thenApply(body -> objectMapper.readValue(body, PatientDTO.class))
      .thenAccept(dto -> processPatient(dto));

// POST with body
HttpRequest postRequest = HttpRequest.newBuilder()
    .uri(URI.create("https://ehr-api.com/sync"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("{\"patientId\": \"123\"}"))
    .build();
```

### ❓ Interview Question
> "Why was `strip()` added in Java 11 when `trim()` already existed?"

### ✅ Model Answer
`trim()` removes characters with ASCII value ≤ 32 (space, tab, etc.) — it's not Unicode-aware. `strip()` uses `Character.isWhitespace()` which handles Unicode whitespace characters like non-breaking space (U+00A0) and other Unicode whitespace that `trim()` leaves untouched. Always prefer `strip()` for internationalized applications.

---

## 7. Java 14–15 — Helpful NullPointerExceptions & Text Blocks

### Helpful NPE Messages (Java 14)
```java
// Before Java 14:
// NullPointerException (no useful info)

// Java 14+:
// "Cannot invoke "String.length()" because "order.patient.name" is null"
// The JVM tells you EXACTLY which reference was null

Order order = getOrder();
int len = order.getPatient().getName().length(); // If patient is null:
// Java 14+: "Cannot invoke 'Patient.getName()' because the return value of
//            'Order.getPatient()' is null"
// Java 13-: "NullPointerException" — no context at all
```

### Text Blocks (Java 15, preview in 13–14)
```java
// JSON payload without escaping
String json = """
        {
          "patientId": "%s",
          "studyType": "CT_SCAN",
          "urgent": true
        }
        """.formatted(patientId);

// SQL without concatenation
String sql = """
        SELECT p.id, p.mrn, p.name, s.study_date
        FROM patients p
        JOIN dicom_studies s ON s.patient_id = p.id
        WHERE s.study_type = :type
          AND s.status = 'PENDING'
        ORDER BY s.study_date DESC
        LIMIT :limit
        """;

// Leading whitespace stripped based on closing """ indentation
// Content indented consistently — no leftover spaces in the result
```

---

## 8. Java 21 — SequencedCollections

```java
// Java 21 introduced SequencedCollection — finally a unified API for ordered access
// Fixes the inconsistency: ArrayList.get(0) vs LinkedList.getFirst() vs Deque.peekFirst()

// All List, Deque, LinkedHashSet, LinkedHashMap now have:
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// First/Last element — consistent across all sequenced types
list.getFirst();  // "a"        (was: list.get(0))
list.getLast();   // "c"        (was: list.get(list.size() - 1))
list.addFirst("z"); // ["z", "a", "b", "c"]
list.addLast("x");  // ["z", "a", "b", "c", "x"]
list.removeFirst(); // "z", list is now ["a", "b", "c", "x"]
list.removeLast();  // "x", list is now ["a", "b", "c"]

// Reversed view — zero-copy
List<String> reversed = list.reversed(); // ["c", "b", "a"] — live view
reversed.forEach(System.out::println);
```

---

## 9. Java Version Summary

| Version | Key Features | LTS? |
|---------|-------------|------|
| **Java 8** | Lambdas, Stream, Optional, java.time, CompletableFuture | — |
| **Java 11** | `String.strip/isBlank/lines`, HTTP Client API, `var` in lambdas | ✅ LTS |
| **Java 14** | Helpful NPE, Records (preview), Pattern matching (preview) | — |
| **Java 15** | Text Blocks GA | — |
| **Java 16** | Records GA, `instanceof` pattern matching GA | — |
| **Java 17** | Sealed classes GA, Pattern matching for switch (preview) | ✅ LTS |
| **Java 21** | Virtual Threads GA, Pattern matching switch GA, SequencedCollections, Record patterns | ✅ LTS |

### JD Note for AdvaHealth
The JD says "Java 17+, preferably Java 21/25". Know these in order of importance:
1. **Virtual Threads (Java 21)** — most likely to be asked, big paradigm shift
2. **Records (Java 16)** — daily use in DTOs/responses
3. **Sealed classes (Java 17)** — shows modern Java knowledge
4. **Pattern matching switch (Java 21)** — elegant code, likely asked
5. **Text Blocks (Java 15)** — SQL, JSON in code — practical

---

*Next: [Week 3 — Spring Boot Core](../week-3/07-spring-boot-core.md)*