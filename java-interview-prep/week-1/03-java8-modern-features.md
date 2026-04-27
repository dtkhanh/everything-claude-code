# Week 1 — Day 5–7: Java 8 Modern Features

> 📖 **Estimated reading time:** 40 minutes  
> 🎯 **Focus:** Lambda, Functional Interfaces, Stream API, Optional, Date/Time API

---

## 1. Lambda Expressions & Functional Interfaces

```java
// Before lambdas — anonymous inner class
Comparator<String> old = new Comparator<String>() {
    public int compare(String a, String b) { return a.length() - b.length(); }
};

// After — lambda
Comparator<String> byLength = (a, b) -> Integer.compare(a.length(), b.length());

// Method reference — shorthand when lambda just delegates
Consumer<String> print = System.out::println;
Function<String, Integer> length = String::length;
```

A lambda is an instance of a **functional interface** — any interface with exactly one abstract method. The JVM implements them via `invokedynamic` which is more memory-efficient than anonymous inner classes.

### 🔑 Key Built-in Functional Interfaces
```java
Function<T, R>     // T → R   (transform)
Consumer<T>        // T → void (side effect)
Supplier<T>        // () → T  (factory/lazy)
Predicate<T>       // T → boolean (test/filter)
UnaryOperator<T>   // T → T   (same type transform)
BinaryOperator<T>  // (T, T) → T
```

### 💻 Code Example — Composing Functions
```java
Function<String, String> trim = String::trim;
Function<String, String> toLowerCase = String::toLowerCase;
Function<String, Integer> length = String::length;

// andThen: first trim, then toLowerCase
Function<String, String> clean = trim.andThen(toLowerCase);
System.out.println(clean.apply("  HELLO  ")); // "hello"

// compose: apply toLowerCase first, then trim (reverse of andThen)
Function<String, String> composed = trim.compose(toLowerCase);

// Predicate composition
Predicate<String> isLong = s -> s.length() > 5;
Predicate<String> startsWithA = s -> s.startsWith("A");

Predicate<String> longAndStartsWithA = isLong.and(startsWithA);
Predicate<String> longOrStartsWithA = isLong.or(startsWithA);
Predicate<String> notLong = isLong.negate();
```

### ❓ Interview Question
> "What is a functional interface? Can you create a custom one?"

### ✅ Model Answer
A functional interface is an interface with **exactly one abstract method**. The `@FunctionalInterface` annotation is optional but recommended — it causes a compile error if you accidentally add a second abstract method. Any lambda expression is an instance of a functional interface. Default and static methods don't count toward the single-abstract-method rule.

```java
@FunctionalInterface
interface Validator<T> {
    boolean validate(T value);
    
    default Validator<T> and(Validator<T> other) {
        return value -> this.validate(value) && other.validate(value);
    }
}

Validator<String> notBlank = s -> s != null && !s.isBlank();
Validator<String> maxLength = s -> s.length() <= 100;
Validator<String> combined = notBlank.and(maxLength);
```

---

## 2. Stream API

```java
// Without Stream — imperative, verbose, easy to get wrong
List<String> result = new ArrayList<>();
for (User user : users) {
    if ("ACTIVE".equals(user.getStatus()) && user.getAge() >= 18) {
        result.add(user.getName());
    }
}
Collections.sort(result);

// With Stream — declarative, composable, lazy
List<String> result = users.stream()
    .filter(u -> "ACTIVE".equals(u.getStatus()))
    .filter(u -> u.getAge() >= 18)
    .map(User::getName)
    .sorted()
    .collect(Collectors.toList());
```

Streams are lazy — intermediate operations don't run until a terminal operation triggers the pipeline. This enables short-circuit evaluation and single-pass execution.

### 🔑 Key Points
- **Intermediate** (lazy, return Stream): `filter`, `map`, `flatMap`, `distinct`, `sorted`, `peek`, `limit`, `skip`
- **Terminal** (eager, triggers pipeline): `collect`, `forEach`, `count`, `reduce`, `findFirst`, `anyMatch`, `min`, `max`
- **Short-circuit**: `findFirst`, `limit`, `anyMatch` stop early
- Streams are consumed once — second terminal call throws `IllegalStateException`

### 💻 Code Example — Common Pipeline Patterns
```java
List<User> users = List.of(
    new User("Alice", 25, "ACTIVE"),
    new User("Bob", 17, "ACTIVE"),
    new User("Charlie", 30, "INACTIVE"),
    new User("Dave", 22, "ACTIVE")
);

// Filter + Map + Collect
List<String> activeAdultNames = users.stream()
    .filter(u -> u.getStatus().equals("ACTIVE"))      // intermediate
    .filter(u -> u.getAge() >= 18)                     // intermediate
    .map(User::getName)                                // intermediate
    .sorted()                                          // intermediate
    .collect(Collectors.toList());                     // terminal

// Count
long activeCount = users.stream()
    .filter(u -> "ACTIVE".equals(u.getStatus()))
    .count();

// GroupingBy — build Map from stream
Map<String, List<User>> byStatus = users.stream()
    .collect(Collectors.groupingBy(User::getStatus));

// GroupingBy + counting
Map<String, Long> countByStatus = users.stream()
    .collect(Collectors.groupingBy(User::getStatus, Collectors.counting()));

// Joining
String names = users.stream()
    .map(User::getName)
    .collect(Collectors.joining(", ", "[", "]")); // "[Alice, Bob, Charlie, Dave]"

// FlatMap — flatten nested collections
List<List<String>> nestedSkills = List.of(
    List.of("Java", "Spring"),
    List.of("SQL", "Redis"),
    List.of("Java", "Docker")
);
List<String> allSkills = nestedSkills.stream()
    .flatMap(Collection::stream)
    .distinct()
    .sorted()
    .collect(Collectors.toList());

// Reduce
int totalAge = users.stream()
    .mapToInt(User::getAge)
    .sum(); // specialized, avoids boxing

OptionalDouble avgAge = users.stream()
    .mapToInt(User::getAge)
    .average();

// Finding and Matching
Optional<User> firstActive = users.stream()
    .filter(u -> "ACTIVE".equals(u.getStatus()))
    .findFirst();

boolean anyMinor = users.stream().anyMatch(u -> u.getAge() < 18);
boolean allAdults = users.stream().allMatch(u -> u.getAge() >= 18);
```

### 💻 Code Example — Collectors.toMap
```java
// Map<name, user>
Map<String, User> userByName = users.stream()
    .collect(Collectors.toMap(
        User::getName,                        // key extractor
        Function.identity(),                  // value extractor (the user itself)
        (existing, replacement) -> existing   // merge function (for duplicate keys)
    ));

// Immutable map (Java 10+)
Map<String, User> immutableMap = users.stream()
    .collect(Collectors.toUnmodifiableMap(User::getName, Function.identity()));
```

### ❓ Interview Question
> "What's the difference between `map` and `flatMap` in streams?"

### ✅ Model Answer
`map` applies a 1-to-1 transformation — each element becomes exactly one new element. `flatMap` applies a 1-to-many transformation and **flattens the result** — each element produces a stream of elements, and flatMap merges all those streams into a single stream.

```java
// map: List<String> → Stream<Stream<Character>> (wrong for this task)
// flatMap: List<String> → Stream<Character> (correct flattening)
List<String> words = List.of("Hello", "World");

// map — gives Stream<String[]>
words.stream().map(w -> w.split("")).collect(Collectors.toList()); // [[H,e,l,l,o],[W,o,r,l,d]]

// flatMap — gives Stream<String>
words.stream().flatMap(w -> Arrays.stream(w.split("")))
    .distinct()
    .collect(Collectors.toList()); // [H,e,l,o,W,r,d]
```

### ❓ Interview Question
> "Are streams lazy? What does that mean and why does it matter?"

### ✅ Model Answer
Yes. Intermediate operations don't execute until a terminal operation is called. This enables two optimizations:
1. **Short-circuit evaluation**: `stream.filter(p).findFirst()` stops as soon as the first match is found — it doesn't filter the entire collection
2. **Fusion**: the JVM can execute multiple operations per element in a single pass instead of creating intermediate collections

```java
// Without short-circuit: filter processes ALL 1 million elements
List<Integer> result = millionItems.stream()
    .filter(expensiveCheck)
    .collect(Collectors.toList());

// With short-circuit: stops at first match
Optional<Integer> first = millionItems.stream()
    .filter(expensiveCheck)
    .findFirst(); // might stop after 1 element
```

### ⚠️ Gotchas
- Streams are **consumed once** — calling a terminal operation twice throws `IllegalStateException`
- `parallel()` is not always faster — overhead of splitting/merging can be worse for small collections
- `forEach` order is not guaranteed on parallel streams — use `forEachOrdered` if order matters
- Modifying the source collection during stream processing is undefined behavior

---

## 3. Optional

```java
// BAD — returns null, callers must remember to check
public User findUser(Long id) {
    return users.get(id); // null if missing — caller might forget to check
}

// GOOD — Optional makes absence explicit
public Optional<User> findUser(Long id) {
    return Optional.ofNullable(users.get(id));
}

// WRONG usage — defeats the purpose
Optional<User> opt = findUser(id);
if (opt.isPresent()) { User u = opt.get(); } // just a verbose null check ❌

// CORRECT — functional style
String city = findUser(id)
    .flatMap(User::getAddress)        // Optional<Address>
    .map(Address::getCity)            // Optional<String>
    .orElse("Unknown");               // String
```

`Optional` is for **return types** where absence is a normal expected outcome — not for every nullable. Never use it as a method parameter or entity field.

### 🔑 Key Points
- `orElse(default)` — always evaluates `default` (even if present)
- `orElseGet(() -> compute())` — lazy: only called if empty
- `orElseThrow(NotFoundException::new)` — throw if empty
- `flatMap` — when the mapper also returns Optional (avoids `Optional<Optional<T>>`)
- Never use as field type in JPA `@Entity`

### ❓ Interview Question
> "When should you NOT use Optional?"

### ✅ Model Answer
- As a **method parameter**: instead use overloaded methods or @Nullable annotation + null check
- As a **field** in an entity: JPA can't serialize Optional, and it adds boxing overhead — use nullable field instead
- As a **Collection element**: Optional in streams or collections adds confusion — filter nulls directly
- For **performance-critical** code: Optional wraps every result in an object — use null returns in hot paths

---

## 4. Date/Time API (java.time)

### 📌 Concept Summary
**java.time** (Java 8) replaced the broken `java.util.Date` / `Calendar` API. It's immutable, thread-safe, and based on ISO-8601.

### 🔑 Key Points
- `LocalDate` → date only, no time, no timezone
- `LocalTime` → time only
- `LocalDateTime` → date + time, no timezone
- `ZonedDateTime` → date + time + timezone (use for user-facing dates)
- `Instant` → machine timestamp (nanoseconds since epoch) — use for DB storage
- `Duration` → amount of time in seconds/nanoseconds
- `Period` → amount of time in years/months/days

### 💻 Code Example
```java
// Creating
LocalDate today = LocalDate.now();
LocalDate birthday = LocalDate.of(1990, Month.MARCH, 15);
LocalDateTime now = LocalDateTime.now();
Instant timestamp = Instant.now(); // for DB/API, serialize as epoch millis

// Manipulation (immutable — returns new instance)
LocalDate nextWeek = today.plusWeeks(1);
LocalDate lastYear = today.minusYears(1);
LocalDate firstOfMonth = today.withDayOfMonth(1);

// Comparing
boolean isBefore = birthday.isBefore(today);
long daysBetween = ChronoUnit.DAYS.between(birthday, today);

// Formatting
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
String formatted = today.format(formatter);
LocalDate parsed = LocalDate.parse("15/03/1990", formatter);

// Timezone conversion
ZonedDateTime utcNow = ZonedDateTime.now(ZoneOffset.UTC);
ZonedDateTime vnNow = utcNow.withZoneSameInstant(ZoneId.of("Asia/Ho_Chi_Minh"));

// Convert to/from legacy Date (for APIs that still use Date)
Date legacyDate = Date.from(Instant.now());
Instant fromLegacy = legacyDate.toInstant();
```

### ❓ Interview Question
> "What were the problems with `java.util.Date` and `Calendar`?"

### ✅ Model Answer
1. **Mutable** — `Date` and `Calendar` are mutable, causing bugs when shared across threads
2. **Confusing API** — months in `Calendar` are 0-indexed (January = 0), year in `Date` is offset from 1900
3. **No timezone awareness** — `Date` is actually a UTC instant but displays in local timezone
4. **Not thread-safe** — `SimpleDateFormat` is famously not thread-safe
5. **Poor design** — `Date` represents a point in time (not a date), making the name misleading

`java.time` solves all these: immutable, clear semantics, type-safe (LocalDate clearly represents a date, not a timestamp), and excellent formatting API.

---

## 5. Method References — Quick Guide

```java
// 4 types of method references:

// 1. Static method
Function<String, Integer> parse = Integer::parseInt;

// 2. Instance method of particular object
String prefix = "Hello, ";
Function<String, String> greet = prefix::concat;

// 3. Instance method of arbitrary object of a type
Function<String, String> toUpper = String::toUpperCase;
// equivalent to: s -> s.toUpperCase()

// 4. Constructor
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<String, User> userFactory = User::new;
```

---

## 6. var — Local Variable Type Inference (Java 10)

```java
// var infers type from right side
var list = new ArrayList<String>();  // inferred as ArrayList<String>
var map = new HashMap<String, List<Integer>>(); // saves typing
var users = userRepository.findAll(); // inferred from return type

// Use in for-each
for (var user : users) {
    System.out.println(user.getName());
}

// Use in try-with-resources
try (var conn = dataSource.getConnection()) { ... }

// DON'T use when type is not obvious from context
var x = doSomething(); // ❌ What type is x? Unreadable
var result = "hello";  // ❌ Prefer String result = "hello"
```

---

*Next: [Week 2 — Day 1: Concurrency & Threading](../week-2/04-concurrency-threading.md)*




