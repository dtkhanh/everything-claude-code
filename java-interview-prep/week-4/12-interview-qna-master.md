# Week 4 — Day 5–7: Full Interview Q&A Master Review

> 📖 **Use this file on Day 1 as a gap map AND on Day 28 as a final drill.**  
> 🎯 **Protocol:** Cover the answer. Say it aloud. Check the Headline Answer first. Read the full answer only if you stumbled.

---

## How to Read Difficulty Labels

- `[Warmup]` — opener question, not a screen. Every interviewer asks it. Don't fumble it.
- `[Core]` — mid-level screen. Expected at any Spring Boot role.
- `[Deep]` — senior signal. Shows you know internals, not just usage.

---

## 🔥 Tier 1 — Almost Always Asked

---

### Q1 `[Warmup]`: "What's the difference between `==` and `.equals()` in Java?"

```java
String a = new String("hello");
String b = new String("hello");
System.out.println(a == b);       // false — different heap objects
System.out.println(a.equals(b)); // true — same content

Integer x = 127;  Integer y = 127;
System.out.println(x == y);      // true  — Integer cache (-128..127) ⚠️ trap

Integer p = 200;  Integer q = 200;
System.out.println(p == q);      // false — above cache range
```

> 📢 **Headline Answer (say this first):**  
> "`==` compares references — whether two variables point to the same object. `.equals()` compares content. For objects, always use `.equals()`. The Integer autobox cache is a classic trap: `Integer x = 127; Integer y = 127; x == y` is `true` because they share a cached instance, but `200 == 200` is `false`."

⚠️ The Integer cache exists for -128 to 127. If you forget this detail, you will fail with a senior interviewer who adds it as a follow-up.

---

### Q2 `[Core]`: "Explain immutability. How do you create an immutable class?"

```java
public final class Money {                              // 1. final — no subclassing
    private final BigDecimal amount;                   // 2. private final fields
    private final List<String> tags;                   // 3. mutable field needs copy

    public Money(BigDecimal amount, List<String> tags) {
        this.amount = Objects.requireNonNull(amount);
        this.tags = List.copyOf(tags);                 // 4. defensive copy IN
    }

    public List<String> getTags() {
        return List.copyOf(tags);                      // 5. defensive copy OUT
    }

    public Money add(Money other) {
        return new Money(amount.add(other.amount), tags); // 6. return new instance
    }
    // No setters
}
```

> 📢 **Headline Answer:**  
> "Immutable means state can't change after construction. Six rules: `final` class, `private final` fields, no setters, initialize in constructor, defensive copy of mutable inputs, return copies from getters. `String`, `BigDecimal`, and Java records are immutable. It's thread-safe by design — no synchronization needed."

🔍 **Follow-up they may ask:** "Are Java records immutable?" — Yes, by default: fields are `private final`, no setters. But if a record field is a `List`, the list content is still mutable unless you defensively copy it.

---

### Q3 `[Core]`: "What is the difference between `HashMap`, `LinkedHashMap`, and `TreeMap`?"

| Feature | HashMap | LinkedHashMap | TreeMap |
|---------|---------|---------------|---------|
| Order | None | Insertion order (or access order) | Sorted by key |
| Null key | ✅ (1 null key) | ✅ | ❌ (NullPointerException) |
| Performance | O(1) avg | O(1) avg | O(log n) |
| Use when | General purpose | LRU cache, preserve order | Sorted iteration |

```java
// LRU Cache — LinkedHashMap with accessOrder=true
new LinkedHashMap<K, V>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > MAX_CAPACITY;
    }
};
```

> 📢 **Headline Answer:**  
> "`HashMap` is your default — fast O(1), no order. `LinkedHashMap` adds insertion or access order (use it for LRU caches). `TreeMap` keeps keys sorted — use when you need `firstKey()`, `headMap()`, or range queries. O(log n) cost."

---

### Q4 `[Core]`: "What is `volatile` and when should you use it?"

```java
// USE CASE 1: stop-flag visible across threads
class Worker implements Runnable {
    private volatile boolean running = true; // without volatile: may loop forever
    public void run() { while (running) { doWork(); } }
    public void stop() { running = false; }
}

// USE CASE 2 — what volatile does NOT fix
volatile int count = 0;
count++; // NOT atomic — still a race condition (read → increment → write)
// Fix: use AtomicInteger.incrementAndGet()
```

> 📢 **Headline Answer:**  
> "`volatile` guarantees visibility — a write is immediately visible to all threads, bypassing CPU caches. It does NOT guarantee atomicity. `count++` is still a race condition even with `volatile`. Use `volatile` for flags and the double-checked locking pattern. Use `AtomicInteger` for counters, `synchronized` for multi-step critical sections."

---

### Q5 `[Warmup]`: "What is the difference between `Runnable` and `Callable`?"

```java
// Runnable — no return, no checked exception
Runnable r = () -> System.out.println("running");
executor.execute(r); // fire and forget

// Callable — returns a value, can declare checked exceptions
Callable<String> c = () -> {
    Thread.sleep(1000); // checked exception allowed
    return "result";
};
Future<String> future = executor.submit(c);
String result = future.get(5, TimeUnit.SECONDS); // blocks with timeout
```

> 📢 **Headline Answer:**  
> "`Runnable` is void — fire and forget. `Callable<T>` returns a value and can throw checked exceptions. `executor.submit(callable)` gives you a `Future<T>` to retrieve the result. For non-blocking result handling, chain `CompletableFuture` instead of calling `future.get()`."

---

### Q6 `[Core]`: "Explain Spring Boot's auto-configuration in your own words."

> 📢 **Headline Answer:**  
> "You add `spring-boot-starter-data-jpa`. Spring sees Hibernate + a datasource on the classpath. It reads a list of auto-configuration candidates from `META-INF/spring/…AutoConfiguration.imports`. Each candidate is annotated with `@Conditional*` — `@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc. Only matching ones activate, and they register beans with sensible defaults. Define your own `@Bean` and `@ConditionalOnMissingBean` prevents the auto-config bean from being created — your definition wins."

**Step by step:**
1. `@SpringBootApplication` activates `@EnableAutoConfiguration`
2. `AutoConfigurationImportSelector` reads the imports file
3. Conditions evaluated — each auto-config class decides if it applies
4. Active ones register beans
5. Your `@Bean` methods override them via `@ConditionalOnMissingBean`

---

### Q7 `[Core]`: "What is `@Transactional` and how does it work?"

> 📢 **Headline Answer:**  
> "`@Transactional` wraps the method in a DB transaction via an AOP proxy. Success → commit. `RuntimeException` or `Error` → rollback. Checked exceptions don't rollback by default — add `rollbackFor = Exception.class` if needed. Three gotchas: doesn't work on private methods, doesn't work on static methods, and `this.method()` self-invocation bypasses the proxy entirely."

**Three gotchas in code:**
```java
@Service
class OrderService {
    @Transactional
    public void createOrder(Order o) {
        this.validate(o); // ❌ bypasses proxy — @Transactional on validate() has no effect
    }

    @Transactional(readOnly = true) // ignored due to self-invocation above
    private void validate(Order o) { ... } // also private — won't work regardless
}
```

**Key propagations to know cold:**
- `REQUIRED` (default): join existing TX or create new
- `REQUIRES_NEW`: always new TX, suspend existing — use for audit logs
- `NESTED`: savepoint within existing TX — partial rollback possible

---

### Q8 `[Core]`: "How do you solve the N+1 query problem?"

```java
// THE PROBLEM
List<Order> orders = orderRepository.findAll(); // 1 query
for (Order o : orders) {
    o.getUser().getName(); // N queries — one per order ❌
}

// SOLUTION 1: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findWithUser(@Param("status") OrderStatus status);

// SOLUTION 2: @EntityGraph
@EntityGraph(attributePaths = {"user", "items"})
List<Order> findByStatus(OrderStatus status);

// SOLUTION 3: Interface projection — no entity loaded, no N+1 possible
interface OrderSummary { Long getId(); String getUserEmail(); }
List<OrderSummary> findByStatus(OrderStatus status);
```

> 📢 **Headline Answer:**  
> "N+1 is when fetching N entities triggers N more queries for lazy associations. Fix in priority order: `JOIN FETCH` in JPQL for specific queries, `@EntityGraph` for declarative loading, DTO/interface projections for read-only endpoints. Detect with `spring.jpa.show-sql=true` or Hibernate statistics — look for repeated `SELECT … WHERE id = ?` in a loop."

---

### Q9 `[Warmup]`: "What's the difference between `@Component`, `@Service`, `@Repository`, `@Controller`?"

> 📢 **Headline Answer:**  
> "All four are `@Component` specializations — they all register a Spring bean. The only functional difference is `@Repository`: it activates persistence exception translation, wrapping `SQLException` and JPA exceptions into Spring's `DataAccessException` hierarchy. `@Service` and `@Component` are semantic only. `@RestController` = `@Controller` + `@ResponseBody`."

---

## 🌡️ Tier 2 — Frequently Asked

---

### Q10 `[Core]`: "What is a memory leak in Java? Give an example."

> 📢 **Headline Answer:**  
> "A memory leak in Java is when objects are still reachable from a GC root but are no longer needed. The GC can't free them. Three classic causes: static collections that grow forever, event listeners that are never removed, and ThreadLocals in thread pools that are never cleared."

```java
// Leak 1: static cache that grows without bound
static final Map<String, Object> CACHE = new HashMap<>();
public void process(String key, Object data) {
    CACHE.put(key, data); // never evicted — LEAK
}
// Fix: bounded cache with eviction (LinkedHashMap or Caffeine)

// Leak 2: ThreadLocal in a thread pool
ThreadLocal<LargeObject> tl = new ThreadLocal<>();
executor.execute(() -> {
    tl.set(new LargeObject());
    doWork();
    tl.remove(); // ← must call this. Pool thread lives forever → so does the value.
});
```

---

### Q11 `[Deep]`: "What is the Java Memory Model and why does it matter for concurrency?"

> 📢 **Headline Answer:**  
> "The JMM defines visibility guarantees between threads. Without synchronization, the compiler, JVM, and CPU can reorder operations for performance — what Thread A writes may never be visible to Thread B. The JMM establishes 'happens-before' relationships: a `synchronized` release happens-before the next acquire, a `volatile` write happens-before any subsequent read. If you don't have a happens-before chain, you have a data race."

The practical consequence: this is why double-checked locking requires `volatile`. Without it, the JVM can assign the reference before finishing construction — Thread B sees a non-null reference to a partially built object.

---

### Q12 `[Deep]`: "What is Spring AOP? How does `@Transactional` work internally?"

> 📢 **Headline Answer:**  
> "Spring AOP wraps beans in proxies — JDK dynamic proxy if the bean implements an interface, CGLIB subclass if not. When you call `@Transactional`, you call the proxy's method, not yours. The proxy's `TransactionInterceptor` starts a transaction, calls your method, then commits or rolls back. This is why self-invocation via `this` breaks it — `this` refers to the real object, not the proxy."

---

### Q13 `[Deep]`: "What is eventual consistency? How do you handle it in microservices?"

> 📢 **Headline Answer:**  
> "Eventual consistency means distributed services converge to the same state, but not immediately. You handle it with: idempotent operations (safe to retry), the Saga pattern for multi-step flows with compensating transactions, and event sourcing where the event log is the source of truth. In the UI: optimistic updates with 'processing' state while the backend catches up."

```java
// Idempotent payment endpoint — safe to retry with same key
@PostMapping("/payments")
public ResponseEntity<PaymentResult> createPayment(
        @RequestHeader("Idempotency-Key") String key,
        @RequestBody PaymentRequest req) {
    return idempotencyService.getOrProcess(key, () -> paymentService.process(req));
}
```

---

### Q14 `[Core]`: "How would you handle authentication in a stateless REST API?"

> 📢 **Headline Answer:**  
> "JWT: client POSTs credentials, server validates and returns a signed JWT. Client sends `Authorization: Bearer <token>` on every request. A `OncePerRequestFilter` validates the token's signature and expiry, then sets the `SecurityContext`. No session stored server-side. Use short-lived access tokens (15–60 min) plus longer-lived refresh tokens stored in DB so you can revoke them on logout."

---

### Q15 `[Core]`: "What design patterns do you use in your Spring Boot services?"

> 📢 **Headline Answer:**  
> "The ones Spring enforces: Repository (JpaRepository), Factory (ApplicationContext + @Configuration @Bean), Proxy (@Transactional, @Cacheable — AOP proxies), Template Method (JdbcTemplate). The ones I choose: Strategy for swappable implementations (PaymentGateway → StripeGateway / PayPalGateway), Builder for complex request objects, Observer for domain events (ApplicationEventPublisher + @EventListener)."

---

## 📝 Tier 3 — Internals & Deep Dives

---

### Q16 `[Deep]`: "Explain how `HashMap` handles hash collisions."

> 📢 **Headline Answer:**  
> "Same bucket, two different keys: Java 7 chains them in a linked list. Java 8+ starts as a linked list but converts to a Red-Black Tree when a bucket exceeds 8 entries — O(n) becomes O(log n). The hash function `h ^ (h >>> 16)` XORs upper bits into lower bits so small arrays don't clump. This treeification also defends against hash-flooding DoS attacks where an attacker crafts keys that all land in the same bucket."

---

### Q17 `[Deep]`: "What is the difference between `Hashtable` and `ConcurrentHashMap`?"

> 📢 **Headline Answer:**  
> "`Hashtable` puts `synchronized(this)` on every method — one lock for the entire map, no parallelism. `ConcurrentHashMap` uses CAS + per-bucket synchronization in Java 8 — multiple threads can write different buckets concurrently. Also: `ConcurrentHashMap.compute()`, `putIfAbsent()`, and `merge()` are atomic. `Hashtable` is deprecated. Use `ConcurrentHashMap` for concurrent access."

| | Hashtable | ConcurrentHashMap |
|--|-----------|-------------------|
| Thread-safe | ✅ coarse (full lock) | ✅ fine-grained (CAS) |
| Null key/value | ❌ | ❌ |
| Throughput | Poor under contention | Excellent |
| Compound ops | Not atomic | `compute`, `putIfAbsent`, `merge` |

---

### Q18 `[Deep]`: "What happens during Spring Boot startup?"

> 📢 **Headline Answer:**  
> "Scan `@Configuration` and `@Component` classes → build `BeanDefinition` metadata → run `BeanFactoryPostProcessor` (resolves `@Value`) → create beans in dependency order → for each: inject → `@PostConstruct` → `BeanPostProcessor.after` where AOP proxies are created → fire `ContextRefreshedEvent` → run `CommandLineRunner`s → health → UP."

Full sequence:
1. Parse `@Configuration`, discover all `@Bean` and `@Component`
2. Build `BeanDefinition` objects (metadata only, no instances)
3. `BeanFactoryPostProcessor` runs — resolves `@Value`, `@PropertySource`
4. Instantiate beans in topological (dependency) order
5. Inject → `BeanPostProcessor.before` → `@PostConstruct` → `BeanPostProcessor.after` ← **proxies created here**
6. `ContextRefreshedEvent` → `@EventListener` methods fire
7. `CommandLineRunner` / `ApplicationRunner` beans run
8. Actuator `/health` → `UP`

---

### Q19 `[Deep]`: "What is optimistic locking and when should you use it vs pessimistic?"

> 📢 **Headline Answer:**  
> "Optimistic: no DB lock. On UPDATE, add `WHERE id=? AND version=?`. If someone else updated first, you get 0 rows → `OptimisticLockException` → retry. Use for low-contention data. Pessimistic: `SELECT FOR UPDATE` — locks the row until you commit. Use for high-contention scenarios like ticket booking or inventory reservation where retry loops are unacceptable."

```java
// Optimistic — @Version does the work
@Entity class Inventory {
    @Version Long version; // auto-checked on every UPDATE
}
// On conflict: Hibernate throws OptimisticLockException → catch and retry

// Pessimistic — SELECT FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT i FROM Inventory i WHERE i.id = :id")
Optional<Inventory> findByIdForUpdate(@Param("id") Long id);
```

---

### Q20 `[Core]`: "How do you write testable Spring Boot code?"

> 📢 **Headline Answer:**  
> "Constructor injection — dependencies become constructor params, so you instantiate with mocks in tests without reflection. Small methods with one job. No `new` inside business logic — use injected factories. Interfaces for external deps so you can swap test doubles. Then `@WebMvcTest` for controllers, `@DataJpaTest` for repositories, `@SpringBootTest` + Testcontainers for integration."

```java
// Controller test — MVC layer only, service is a mock
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean UserService userService;

    @Test
    void shouldReturn404_whenUserNotFound() throws Exception {
        when(userService.findById(99L)).thenReturn(Optional.empty());
        mockMvc.perform(get("/api/users/99"))
            .andExpect(status().isNotFound());
    }
}

// Repository test — real JPA, H2 in-memory
@DataJpaTest
class OrderRepositoryTest {
    @Autowired OrderRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void shouldFindActiveOrders() {
        em.persistAndFlush(new Order(OrderStatus.ACTIVE));
        assertThat(repo.findByStatus(ACTIVE)).hasSize(1);
    }
}
```

---

## 🎯 Mock Interview Checklist

Before the interview — say each answer aloud without notes, under 90 seconds:

- [ ] `==` vs `.equals()` + Integer cache trap
- [ ] 6 rules for immutable class
- [ ] HashMap internals: hash → bucket → linked list → treeify → resize
- [ ] `volatile` vs `synchronized` vs `AtomicInteger` — which to use when
- [ ] Spring auto-configuration: 5 steps
- [ ] `@Transactional`: rollback rules + 3 gotchas (private, static, self-invokes)
- [ ] N+1: detect + 3 solutions in order of preference
- [ ] JWT auth flow: login → token → filter → SecurityContext
- [ ] Optimistic vs pessimistic locking: when to use each
- [ ] How to make a class immutable (say the 6 rules cold)

---

*Review [`cheatsheets/`](../cheatsheets/) the morning of the interview. 🎯*
