# Week 1 — Day 1–2: Core Java & OOP Principles

> 📖 **Estimated reading time:** 40 minutes  
> 🎯 **Focus:** OOP principles, SOLID, String/Object internals, equals/hashCode contract

---

## 1. OOP — The Four Pillars

---

## 1.1 Encapsulation

```java
// BAD — exposed state, anyone can set balance to -9999
public class BankAccount {
    public double balance; // ❌
}

// GOOD — encapsulated
public class BankAccount {
    private double balance;

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        this.balance += amount;
    }

    public double getBalance() { return balance; }
}
```

Encapsulation hides internal state behind a controlled API. Consumers depend on your contract, not your implementation — you can rename `balance` to `currentBalance` internally without breaking any caller.

### 🔑 Key Points
- `private` fields, public methods only
- No setter = immutable field — the object enforces its own invariants
- Returning `new ArrayList<>(this.items)` not `this.items` — don't leak mutable internal state

### ❓ Interview Question
> "What is encapsulation and why is it important?"

### ✅ Model Answer
Encapsulation bundles data and the methods that operate on it into a single unit (class), while restricting direct access to internal state. It's important because it enforces **invariants** (e.g., balance can't go negative), reduces coupling between classes, and allows the internal implementation to change without affecting callers.

### ⚠️ Gotchas
- Providing a setter for every field negates encapsulation
- Returning mutable internal objects breaks encapsulation: `return new ArrayList<>(this.items)` not `return this.items`

---

## 1.2 Inheritance

```java
// Inheritance — IS-A relationship
public class Animal {
    protected String name;
    public void eat() { System.out.println(name + " eats"); }
}

public class Dog extends Animal {
    @Override
    public void eat() {
        super.eat();
        System.out.println("...dog food");
    }
    public void bark() { System.out.println("Woof!"); }
}

// Composition — HAS-A (preferred when "IS-A" is not a natural fit)
public class Dog {
    private final EatingBehavior eatingBehavior; // composition
    public void eat() { eatingBehavior.eat(this.name); }
}
```

Prefer composition when you want to reuse behavior without locking into a class hierarchy, or when you need to change behavior at runtime.

### 🔑 Key Points
- `extends` for classes (single), `implements` for interfaces (multiple)
- `super` calls parent constructor/methods
- Prefer **composition over inheritance** (GoF, Effective Java Item 18)

### ❓ Interview Question
> "When would you use composition over inheritance?"

### ✅ Model Answer
Composition is preferred when the relationship is "HAS-A" rather than "IS-A", when you want to reuse behavior without imposing a class hierarchy, or when you need to change behavior at runtime. Inheritance creates tight coupling — a change in the parent can silently break all children. The classic example: a `Stack` shouldn't extend `Vector` (Java's mistake) because a Stack IS-NOT a Vector in behavior — it should HAVE a data structure internally.

### 🔍 Deep Dive
The **Liskov Substitution Principle** says a subclass must be substitutable for its parent without breaking correctness. `Stack extends Vector` violates this because Vector exposes `add(int index, E element)` which breaks Stack semantics.

### ⚠️ Gotchas
- Calling an overridable method in a constructor can lead to bugs — the child class isn't fully initialized yet

---

## 1.3 Polymorphism

```java
// Runtime polymorphism
interface Shape {
    double area();
}

class Circle implements Shape {
    private double radius;
    public Circle(double radius) { this.radius = radius; }
    
    @Override
    public double area() { return Math.PI * radius * radius; }
}

class Rectangle implements Shape {
    private double w, h;
    public Rectangle(double w, double h) { this.w = w; this.h = h; }
    
    @Override
    public double area() { return w * h; }
}

// Polymorphic usage — caller doesn't know/care about concrete type
List<Shape> shapes = List.of(new Circle(5), new Rectangle(3, 4));
shapes.forEach(s -> System.out.println(s.area())); // dynamic dispatch at runtime
```

Polymorphism lets calling code work with a `Shape` reference regardless of whether it's a `Circle` or `Rectangle` — the JVM resolves the right `area()` at runtime via the vtable.

### 🔑 Key Points
- **Compile-time:** overloading — same name, different parameters
- **Runtime:** overriding — JVM dispatches to correct subclass via vtable
- `static`/`private` methods cannot be overridden

### ❓ Interview Question
> "What's the difference between method overloading and method overriding?"

### ✅ Model Answer
**Overloading** is resolved at **compile time** — same method name with different parameter types/count. **Overriding** is resolved at **runtime** via dynamic dispatch — a subclass provides a different implementation of a parent method. Overloading is not polymorphism in the true OOP sense; overriding is.

### ⚠️ Gotchas
- `static` methods cannot be overridden (they're hidden, not polymorphic)
- `private` methods cannot be overridden

---

## 1.4 Abstraction

```java
// Abstract class — partial implementation
abstract class PaymentProcessor {
    // Template method pattern
    public final void processPayment(double amount) {
        validate(amount);        // implemented here
        charge(amount);          // delegated to subclass
        sendReceipt(amount);     // implemented here
    }

    private void validate(double amount) {
        if (amount <= 0) throw new IllegalArgumentException();
    }
    
    protected abstract void charge(double amount); // subclass must implement
    
    private void sendReceipt(double amount) {
        System.out.println("Receipt sent for: " + amount);
    }
}

class CreditCardProcessor extends PaymentProcessor {
    @Override
    protected void charge(double amount) {
        System.out.println("Charging credit card: " + amount);
    }
}
```

Abstract class = partial implementation + template for subclasses. Interface = pure contract any class can satisfy. The distinction matters in interview: a class can implement many interfaces but only extend one abstract class.

### 🔑 Key Points
| | Abstract Class | Interface |
|--|----------------|-----------|
| **Inheritance** | Single | Multiple |
| **State** | Instance fields ✅ | No instance fields |
| **Constructor** | Has constructor | No constructor |
| **Methods** | abstract + concrete | abstract, default, static |
| **Use when** | Shared base implementation | Defining a capability |

### ❓ Interview Question
> "What's the difference between an abstract class and an interface? When would you use each?"

### ✅ Model Answer
| | Abstract Class | Interface |
|--|----------------|-----------|
| **Inheritance** | Single | Multiple |
| **State** | Can have instance fields | No instance fields (constants only) |
| **Constructor** | Has constructor | No constructor |
| **Methods** | Mix of abstract + concrete | abstract, default, static |
| **Use when** | Shared base implementation | Defining a contract/capability |

Use an **abstract class** when there's common behavior to share (Template Method pattern). Use an **interface** when you want to define a capability that unrelated classes can implement (e.g., `Comparable`, `Serializable`).

### ⚠️ Gotchas
- If an interface gets too large, it violates the Interface Segregation Principle (SOLID)
- `default` methods in interfaces can cause the "diamond problem" — resolved by compiler error requiring override

---

## 2. SOLID Principles

### 📌 Concept Summary
SOLID is 5 design principles for maintainable, extensible OOP code. Memorize the acronym AND one real example each.

---

### S — Single Responsibility Principle
> A class should have **one reason to change**.

```java
// BAD — does too much
class UserService {
    public void saveUser(User u) { /* DB logic */ }
    public void sendWelcomeEmail(User u) { /* email logic */ }
    public String generateReport(User u) { /* report logic */ }
}

// GOOD — each class has one job
class UserRepository { public void save(User u) { /* DB only */ } }
class EmailService { public void sendWelcome(User u) { /* email only */ } }
class UserReportGenerator { public String generate(User u) { /* report only */ } }
```

---

### O — Open/Closed Principle
> Open for extension, **closed for modification**.

```java
// BAD — adding a new discount type requires modifying this class
class DiscountCalculator {
    public double calculate(String type, double price) {
        if (type.equals("STUDENT")) return price * 0.8;
        if (type.equals("SENIOR")) return price * 0.7;
        // every new type = modify this class ❌
    }
}

// GOOD — extend without modifying
interface DiscountPolicy {
    double apply(double price);
}
class StudentDiscount implements DiscountPolicy {
    public double apply(double price) { return price * 0.8; }
}
class SeniorDiscount implements DiscountPolicy {
    public double apply(double price) { return price * 0.7; }
}
// New discount type? Just add a new class. Don't touch existing code. ✅
```

---

### L — Liskov Substitution Principle
> Subtypes must be substitutable for their base types **without breaking behavior**.

```java
// VIOLATION — Square extends Rectangle breaks LSP
class Rectangle {
    protected int w, h;
    public void setWidth(int w) { this.w = w; }
    public void setHeight(int h) { this.h = h; }
    public int area() { return w * h; }
}
class Square extends Rectangle {
    @Override
    public void setWidth(int side) { this.w = this.h = side; } // ❌ changes both
}
// Code using Rectangle breaks when passed a Square:
// rect.setWidth(5); rect.setHeight(4); assert rect.area() == 20 → FAILS for Square
```

---

### I — Interface Segregation Principle
> Clients should not be forced to depend on interfaces they don't use.

```java
// BAD — fat interface
interface Worker {
    void work();
    void eat();   // robots don't eat
    void sleep(); // robots don't sleep
}

// GOOD — segregated
interface Workable { void work(); }
interface Eatable { void eat(); }
class HumanWorker implements Workable, Eatable { ... }
class RobotWorker implements Workable { ... } // only what it needs
```

---

### D — Dependency Inversion Principle
> High-level modules should not depend on low-level modules. Both should depend on **abstractions**.

```java
// BAD — high-level Service directly depends on low-level MySQLDatabase
class OrderService {
    private MySQLDatabase db = new MySQLDatabase(); // ❌ tightly coupled
}

// GOOD — depend on abstraction (Spring @Autowired + interface)
interface OrderRepository { Order findById(Long id); }
class OrderService {
    private final OrderRepository orderRepository; // depends on abstraction ✅
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
// Spring injects the concrete impl at runtime
```

### ❓ Interview Question
> "Explain SOLID with one example each. Which one do you think is most important?"

### ✅ Model Answer
*(Walk through the 5 principles briefly — see above)*. Most important for day-to-day: **SRP** and **DIP**. SRP keeps classes focused and testable. DIP is the foundation of Spring's entire dependency injection model — it's why `@Autowired` works with any implementation, making testing with mocks trivial.

---

## 3. String Internals & equals/hashCode Contract

### 📌 Concept Summary
`String` is the most commonly discussed class in Java interviews. Understanding the **String Pool** and the **equals/hashCode contract** is essential.

### 🔑 Key Points — String
- `String` is **immutable** — every modification creates a new object
- String Pool (interned strings): literals are stored in a pool in the heap (Metaspace before Java 8, heap after)
- `==` compares references; `.equals()` compares content
- `StringBuilder` is mutable, not thread-safe; `StringBuffer` is thread-safe but slower

### 💻 Code Example — String Pool
```java
String s1 = "hello";           // goes to String Pool
String s2 = "hello";           // same reference from Pool
String s3 = new String("hello"); // new object on heap, NOT in pool
String s4 = s3.intern();       // explicitly puts in pool / returns pool ref

System.out.println(s1 == s2);  // true  (same pool reference)
System.out.println(s1 == s3);  // false (heap vs pool)
System.out.println(s1 == s4);  // true  (intern() returns pool reference)
System.out.println(s1.equals(s3)); // true (content comparison)
```

### 🔑 Key Points — equals/hashCode Contract
The contract (in `java.lang.Object` Javadoc):
1. If `a.equals(b)` → `a.hashCode() == b.hashCode()` (MUST)
2. If `a.hashCode() == b.hashCode()` → `a.equals(b)` might be false (hash collision is OK)
3. `equals` must be reflexive, symmetric, transitive, consistent, and `null`-safe

### 💻 Code Example — Correct Override
```java
public class User {
    private final Long id;
    private final String email;

    // Constructor, getters...

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;               // identity shortcut
        if (!(o instanceof User user)) return false; // type check + cast (Java 16+)
        return Objects.equals(id, user.id) &&
               Objects.equals(email, user.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, email);           // consistent with equals
    }
}
```

### ❓ Interview Question
> "What happens if you override `equals` but not `hashCode`?"

### ✅ Model Answer
The **contract is broken**. If two `User` objects are "equal" by email but return different hash codes, they'll be stored in different buckets in a `HashMap`/`HashSet`. So `set.add(user1)` and `set.add(user2)` would both succeed even if `user1.equals(user2)` is true — the Set would contain "duplicates". This is a silent, hard-to-debug bug.

### 🔍 Deep Dive
In `HashMap`, the lookup process:
1. Compute `hashCode()` → determine bucket
2. Walk the bucket's linked list/tree, call `equals()` on each entry
3. If `hashCode()` is wrong → wrong bucket → `equals()` never called → `get()` returns null

### ⚠️ Gotchas
- Never include **mutable fields** in `equals`/`hashCode` if the object will be stored in a `HashSet`/`HashMap`. Mutating the key after insertion = object becomes unfindable.
- `instanceof` check must use the exact type unless you intentionally want subclass equality (rare)

---

## 4. Design Patterns (Most Asked)

### 📌 Concept Summary
You don't need to know all 23 GoF patterns. Focus on the ones used in Spring Boot daily.

### Singleton
```java
// Thread-safe Singleton with double-checked locking
public class DatabaseConfig {
    private static volatile DatabaseConfig instance; // volatile = memory visibility

    private DatabaseConfig() {}

    public static DatabaseConfig getInstance() {
        if (instance == null) {
            synchronized (DatabaseConfig.class) {
                if (instance == null) {  // second check inside synchronized
                    instance = new DatabaseConfig();
                }
            }
        }
        return instance;
    }
}
// In Spring: every @Bean is a Singleton by default — Spring manages the instance
```

### Builder
```java
// Modern: use Lombok @Builder or Java record
public class CreateUserRequest {
    private final String name;
    private final String email;
    private final UserRole role;

    private CreateUserRequest(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.role = builder.role;
    }

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String name;
        private String email;
        private UserRole role = UserRole.USER; // default

        public Builder name(String name) { this.name = name; return this; }
        public Builder email(String email) { this.email = email; return this; }
        public Builder role(UserRole role) { this.role = role; return this; }
        public CreateUserRequest build() { return new CreateUserRequest(this); }
    }
}

// Usage
CreateUserRequest req = CreateUserRequest.builder()
    .name("Alice")
    .email("alice@example.com")
    .build();
```

### Strategy
```java
// Used everywhere in Spring (HandlerMethodArgumentResolver, HttpMessageConverter, etc.)
@FunctionalInterface
interface SortStrategy<T> {
    List<T> sort(List<T> items);
}

// Different strategies
SortStrategy<User> byName = items -> items.stream()
    .sorted(Comparator.comparing(User::getName))
    .collect(Collectors.toList());

SortStrategy<User> byJoinDate = items -> items.stream()
    .sorted(Comparator.comparing(User::getJoinDate).reversed())
    .collect(Collectors.toList());

// Swap strategy at runtime — no if/else
class UserService {
    private SortStrategy<User> sortStrategy = byName; // default
    public void setSortStrategy(SortStrategy<User> strategy) {
        this.sortStrategy = strategy;
    }
}
```

### ❓ Interview Question
> "Where do you see design patterns used in Spring Boot?"

### ✅ Model Answer
- **Singleton:** Every `@Bean` is a singleton by default
- **Factory:** `BeanFactory`, `ApplicationContext` — Spring container creates beans
- **Proxy:** `@Transactional`, `@Cacheable`, `@Async` — Spring creates AOP proxies around your methods
- **Template Method:** `JdbcTemplate`, `RestTemplate` — defines skeleton of algorithm, delegates steps to subclasses
- **Observer:** `ApplicationEvent` + `@EventListener` — publish/subscribe in Spring
- **Strategy:** `HttpMessageConverter` list — Spring tries each converter to serialize/deserialize

---

*Next: [02-collections-generics.md](./02-collections-generics.md)*








