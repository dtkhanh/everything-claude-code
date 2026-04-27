# Week 1 — Day 3–4: Collections Framework & Generics

> 📖 **Estimated reading time:** 45 minutes  
> 🎯 **Focus:** HashMap internals, List/Set/Queue, Generics wildcards, Comparable vs Comparator

---

## 1. Collections — The Big Picture

Before the internals: know which interface to reach for and why.

```
Collection
├── List          → ordered, allows duplicates
│   ├── ArrayList     → resizable array, O(1) get, O(n) insert middle
│   ├── LinkedList    → doubly-linked, O(1) insert/delete head/tail, O(n) get
│   └── CopyOnWriteArrayList → thread-safe, expensive writes, cheap reads
│
├── Set           → no duplicates
│   ├── HashSet       → O(1) ops, no order, backed by HashMap
│   ├── LinkedHashSet → insertion-ordered, backed by LinkedHashMap
│   └── TreeSet       → sorted (natural or Comparator), O(log n), backed by TreeMap
│
└── Queue / Deque
    ├── ArrayDeque    → fastest stack/queue, no null
    ├── PriorityQueue → min-heap, natural ordering or Comparator
    └── LinkedList    → also implements Deque

Map (not Collection)
├── HashMap           → O(1) avg, no order, allows null key/value
├── LinkedHashMap     → insertion-ordered or access-ordered
├── TreeMap           → sorted by key, O(log n)
├── ConcurrentHashMap → thread-safe, no null keys/values
└── EnumMap           → optimized for Enum keys
```

---

## 2. HashMap Internals — The #1 Interview Topic

```java
// WRONG — using default identity hashCode
Map<User, String> map = new HashMap<>();
User u1 = new User(1L, "alice@x.com");
map.put(u1, "Admin");

User u2 = new User(1L, "alice@x.com"); // same logical user, different object
System.out.println(map.get(u2)); // null ❌ — different hashCode → wrong bucket

// CORRECT — override equals + hashCode (see 01-core-java-oop.md)
// map.get(u2) → "Admin" ✅
```

`HashMap` uses `hashCode()` to pick a bucket, then `equals()` to find the key within the bucket. If your key class doesn't override both, map lookups silently return null.

### 🔑 Key Points
- Default capacity: **16**; load factor: **0.75**
- When entries > capacity × load factor → **resize** (double capacity, rehash all)
- Hash is computed: `(key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)`
- Bucket index: `hash & (capacity - 1)` (bitwise AND, faster than modulo)
- Treeify threshold: 8 (linked list → Red-Black Tree); untreeify: 6

### 💻 Code Example — Resize cost
```java
// Pre-size when you know the expected size to avoid rehashing
Map<String, Integer> wordCount = new HashMap<>(expectedSize, 0.75f);

// Or use Guava
Map<String, Integer> map = Maps.newHashMapWithExpectedSize(1000);
```

### ❓ Interview Question
> "Walk me through what happens internally when you call `map.put("key", value)` on a HashMap."

### ✅ Model Answer
1. Compute hash: `hashCode()` of the key, then XOR with upper 16 bits (`h ^ (h >>> 16)`) to spread entropy
2. Find bucket index: `hash & (n-1)` where n = array length
3. If bucket is **empty** → store the entry directly
4. If **collision** exists:
   - Walk the linked list/tree, call `equals()` on each key
   - If match found → **update** the value
   - If no match → **append** to the list/tree
5. After insertion, check if `size > capacity * loadFactor` → if yes, **resize**: create new array (2×), rehash every entry

### 🔍 Deep Dive
**Why XOR with upper 16 bits?**  
When capacity is small (16, 32), `hash & (n-1)` only uses the **low bits** of the hash. XOR spreads influence from high bits into low bits, reducing collisions for keys that differ only in high bits.

**Java 8 treeification:**  
When a single bucket has ≥8 entries, it converts to a Red-Black Tree (O(log n) instead of O(n) for lookup). This protects against denial-of-service attacks where attackers craft keys with the same hash.

### ⚠️ Gotchas
- `HashMap` is **not thread-safe** — concurrent `put` can cause infinite loops during resize (Java 6, fixed in Java 8 but still data races)
- `null` key is always stored in bucket 0
- `Hashtable` is thread-safe but slower (full method synchronization) — use `ConcurrentHashMap` instead

---

## 3. ConcurrentHashMap vs synchronized Collections

### 📌 Concept Summary
`ConcurrentHashMap` uses **segment-level locking** (Java 7) → **CAS + node-level locking** (Java 8+), allowing multiple threads to write concurrently without blocking each other.

### 💻 Code Example
```java
// Thread-safe options (in order of preference):

// 1. ConcurrentHashMap — best for concurrent read+write
ConcurrentHashMap<String, Integer> concurrent = new ConcurrentHashMap<>();
concurrent.put("key", 1);
// putIfAbsent, compute, merge are atomic operations
concurrent.compute("key", (k, v) -> v == null ? 1 : v + 1); // atomic increment

// 2. Collections.synchronizedMap — wraps any map, coarse-grained lock
Map<String, Integer> synced = Collections.synchronizedMap(new HashMap<>());
// Must manually synchronize iteration:
synchronized (synced) {
    for (Map.Entry<String, Integer> entry : synced.entrySet()) { ... }
}

// 3. Immutable after creation (Java 9+) — no locks needed
Map<String, Integer> immutable = Map.of("a", 1, "b", 2);
```

### ❓ Interview Question
> "What's the difference between `ConcurrentHashMap` and `Collections.synchronizedMap`?"

### ✅ Model Answer
`Collections.synchronizedMap` wraps every method with a `synchronized` block on the entire map — only **one thread** can operate at a time. `ConcurrentHashMap` uses a much finer-grained locking strategy (CAS operations + synchronized on individual nodes in Java 8), allowing multiple threads to read and write **different buckets concurrently**. `ConcurrentHashMap` has ~10× better throughput under contention, but doesn't allow null keys/values. Also, compound operations like `if(!map.containsKey(k)) map.put(k, v)` are NOT atomic with `synchronizedMap` — you must use `putIfAbsent` on `ConcurrentHashMap`.

---

## 4. List — ArrayList vs LinkedList

### 💻 Complexity Comparison
| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| `get(i)` | O(1) | O(n) |
| `add(end)` | O(1) amortized | O(1) |
| `add(middle)` | O(n) — shift elements | O(n) — find position |
| `remove(i)` | O(n) — shift elements | O(n) — find position |
| Memory | Array (contiguous) | Node objects (pointer overhead) |

### ❓ Interview Question
> "When would you actually use `LinkedList` over `ArrayList`?"

### ✅ Model Answer
Rarely. `LinkedList` is faster for **frequent insertions/deletions at the head** (O(1) vs O(n) for ArrayList). However, modern CPUs favor **cache-locality** — iterating an `ArrayList` (contiguous memory) is much faster than `LinkedList` (pointer chasing). In practice, `ArrayDeque` is preferred over `LinkedList` for queue/deque use cases. Use `LinkedList` only when you have a specific benchmark proving it's faster for your access pattern.

---

## 5. Comparable vs Comparator

### 📌 Concept Summary
Two ways to define ordering in Java:
- `Comparable<T>`: class defines its **natural ordering** (one ordering)
- `Comparator<T>`: external ordering strategy (multiple orderings)

### 💻 Code Example
```java
// Comparable — natural ordering (embedded in the class)
public class Product implements Comparable<Product> {
    private String name;
    private double price;

    @Override
    public int compareTo(Product other) {
        return Double.compare(this.price, other.price); // ascending by price
    }
}

// Sort uses natural order
List<Product> products = new ArrayList<>(List.of(...));
Collections.sort(products); // uses compareTo

// Comparator — external, multiple orderings
Comparator<Product> byName = Comparator.comparing(Product::getName);
Comparator<Product> byPriceDesc = Comparator.comparing(Product::getPrice).reversed();
Comparator<Product> byNameThenPrice = byName.thenComparing(byPriceDesc);

products.sort(byNameThenPrice);

// Java Stream
products.stream()
    .sorted(Comparator.comparing(Product::getPrice).reversed())
    .limit(10)
    .collect(Collectors.toList());
```

### ❓ Interview Question
> "What does `compareTo` return and what do the values mean?"

### ✅ Model Answer
`compareTo` returns:
- **Negative** → `this` comes before `other` (this < other)
- **Zero** → they're equal
- **Positive** → `this` comes after `other` (this > other)

Critical: never use `this.value - other.value` for integer comparison — it can **overflow**. Always use `Integer.compare(this.value, other.value)` or `Double.compare(...)`.

⚠️ **Consistency with equals:** If `a.compareTo(b) == 0`, ideally `a.equals(b)` should be `true`. `TreeSet` uses `compareTo`, not `equals` — if they're inconsistent, a `TreeSet` behaves differently from a `HashSet`.

---

## 6. Generics — Wildcards & Bounds

### 📌 Concept Summary
Generics provide **compile-time type safety** without runtime overhead (type erasure). Wildcards (`?`) enable flexible APIs.

### 🔑 Key Points
- `<T>` — unbounded type parameter
- `<T extends Comparable<T>>` — bounded (upper bound)
- `<? extends Number>` — wildcard, read-only (PECS: Producer Extends)
- `<? super Integer>` — wildcard, write-only (PECS: Consumer Super)
- Type erasure: at runtime, `List<String>` is just `List`

### 💻 Code Example — PECS Principle
```java
// PECS = Producer Extends, Consumer Super

// Producer (reading FROM the collection) → use extends
public double sumList(List<? extends Number> numbers) {
    return numbers.stream().mapToDouble(Number::doubleValue).sum();
    // ✅ Can accept List<Integer>, List<Double>, List<BigDecimal>
    // ❌ Cannot add to this list (unknown exact type)
}

// Consumer (writing TO the collection) → use super
public void addNumbers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
    list.add(3);
    // ✅ Can add Integers (and subtypes)
    // ✅ Accept List<Integer>, List<Number>, List<Object>
    // ❌ Cannot read as Integer (only Object guaranteed)
}

// In practice — Collections.copy signature
// static <T> void copy(List<? super T> dest, List<? extends T> src)
```

### 💻 Code Example — Bounded type parameters
```java
// T must be Comparable to itself
public <T extends Comparable<T>> T findMax(List<T> list) {
    return list.stream()
        .max(Comparator.naturalOrder())
        .orElseThrow(NoSuchElementException::new);
}

// Multiple bounds — T extends both Comparable and Serializable
public <T extends Comparable<T> & Serializable> void save(T item) { ... }
```

### ❓ Interview Question
> "What is type erasure in Java Generics and what are its limitations?"

### ✅ Model Answer
At compile time, Java replaces all generic type parameters with their upper bounds (`Object` for unbounded). The bytecode has no information about the generic type — it's a compile-time-only feature. Limitations:
1. **Cannot use primitives:** `List<int>` is invalid → use `List<Integer>`
2. **Cannot check generic type at runtime:** `if (list instanceof List<String>)` won't compile
3. **Cannot create generic arrays:** `T[] arr = new T[10]` doesn't work
4. **Unchecked casts:** Downcasting from raw type to generic type can cause `ClassCastException` at unexpected points

### 🔍 Deep Dive
```java
// Type erasure example
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
System.out.println(strings.getClass() == ints.getClass()); // true! Both are ArrayList at runtime

// Heap pollution — the danger of unchecked casts
List rawList = new ArrayList<String>();
List<Integer> intList = rawList; // unchecked warning
intList.add(42);  // works at runtime
String s = ((List<String>) rawList).get(0); // ClassCastException HERE not on add()
```

---

## 7. Collections Utility Methods (Quick Reference)

```java
// Sorting
Collections.sort(list);                              // natural order
Collections.sort(list, Comparator.reverseOrder());   // reverse natural
list.sort(Comparator.comparing(User::getName));       // preferred in Java 8+

// Searching (list must be sorted first)
int idx = Collections.binarySearch(sortedList, target); // O(log n)

// Immutable wrappers
List<String> immutable = Collections.unmodifiableList(list);
// Java 9+: preferred approach
List<String> listOf = List.of("a", "b", "c");     // immutable, no nulls
List<String> copyOf = List.copyOf(existingList);   // immutable defensive copy
Set<String> setOf = Set.of("x", "y");             // immutable, no duplicates
Map<String, Integer> mapOf = Map.of("a", 1, "b", 2);

// Frequency
int count = Collections.frequency(list, target);

// Min/Max
Optional<String> max = list.stream().max(Comparator.naturalOrder());
String min = Collections.min(list);
```

---

## 8. Queue & Deque

### 📌 Concept Summary
`Queue` is FIFO; `Deque` is double-ended (stack + queue). `ArrayDeque` is almost always the right choice.

```java
// Queue operations — two APIs
// throws exception     returns null/false
queue.add(e)            queue.offer(e)      // insert
queue.remove()          queue.poll()        // remove head
queue.element()         queue.peek()        // inspect head

// ArrayDeque as Stack (faster than java.util.Stack)
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);        // addFirst
stack.push(2);
System.out.println(stack.pop());  // 2 — LIFO via removeFirst()

// ArrayDeque as Queue
Deque<Integer> queue2 = new ArrayDeque<>();
queue2.offer(1);      // addLast
queue2.offer(2);
System.out.println(queue2.poll()); // 1 — FIFO via removeFirst()

// PriorityQueue — min-heap
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Top-K pattern (keep K largest elements using min-heap)
PriorityQueue<Integer> topK = new PriorityQueue<>(k);
for (int num : nums) {
    topK.offer(num);
    if (topK.size() > k) topK.poll(); // remove smallest
}
// topK now contains K largest elements
```

### ❓ Interview Question
> "Why is `Stack` considered a legacy class? What should you use instead?"

### ✅ Model Answer
`java.util.Stack` extends `Vector` (synchronized array-based list), which is legacy and was never a good fit for a stack — it exposes list operations like `get(index)` that break stack semantics and its full synchronization is a performance penalty. `ArrayDeque` is the recommended replacement: faster (not synchronized), more memory-efficient, and has explicit `push`/`pop`/`peek` methods that communicate stack intent clearly.

---

*Next: [03-java8-modern-features.md](./03-java8-modern-features.md)*



