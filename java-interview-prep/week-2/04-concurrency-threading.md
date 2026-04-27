# Week 2 — Day 1–2: Concurrency & Threading

> 📖 **Estimated reading time:** 50 minutes  
> 🎯 **Focus:** Thread, ExecutorService, locks, volatile, atomic, CompletableFuture

---

## 1. Thread Fundamentals

### 📌 Concept Summary
A **thread** is a lightweight unit of execution. Java threads map 1-to-1 to OS threads (before Virtual Threads in Java 21). The JVM starts with a main thread; you create more for parallelism.

### 🔑 Key Points
- Thread states: NEW → RUNNABLE → BLOCKED/WAITING/TIMED_WAITING → TERMINATED
- `start()` → creates new thread + calls `run()` in that thread
- `run()` directly → executes in the CURRENT thread (not a new one)
- `join()` → current thread waits for the called thread to finish

### 💻 Code Example
```java
// Way 1: Extend Thread (almost never do this)
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

// Way 2: Implement Runnable (better — separates task from execution)
Runnable task = () -> System.out.println("Task running");
Thread t = new Thread(task);
t.start();

// Way 3: ExecutorService (correct for production code)
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> System.out.println("Submitted task"));
executor.shutdown(); // don't forget!
```

---

## 2. The Memory Visibility Problem — volatile

```java
// WITHOUT volatile: worker thread caches 'running' in a CPU register
// stop() sets it to false on main thread — worker never sees the change → infinite loop

class Worker implements Runnable {
    private volatile boolean running = true; // volatile fixes visibility

    @Override
    public void run() {
        while (running) { doWork(); }
        System.out.println("Worker stopped");
    }

    public void stop() { running = false; }
}
```

`volatile` forces every write to flush to main memory immediately. Every read fetches from main memory, bypassing the CPU cache. The guarantee is **visibility only** — compound operations like `count++` (read-modify-write) are still race conditions.

### 🔑 Key Points
- `volatile` field: write immediately visible to all threads
- Does NOT make compound operations atomic: `count++` is still a race
- Use for: flags, double-checked locking reference
- Does NOT replace `synchronized` for multi-step operations

### 💻 Code Example — Double-checked locking (volatile + synchronized)
```java
class Singleton {
    private static volatile Singleton instance; // volatile required!

    public static Singleton getInstance() {
        if (instance == null) {              // first check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {      // second check (with lock)
                    instance = new Singleton(); // volatile ensures full visibility
                }
            }
        }
        return instance;
    }
}
// Without volatile: partially constructed object could be visible to other threads
```

### ❓ Interview Question
> "What's the difference between `volatile`, `synchronized`, and `AtomicInteger`?"

### ✅ Model Answer
| | volatile | synchronized | AtomicInteger |
|--|----------|--------------|---------------|
| **Visibility** | ✅ | ✅ | ✅ |
| **Atomicity** | ❌ (single write/read only) | ✅ (block of code) | ✅ (single operations) |
| **Mutual exclusion** | ❌ | ✅ | ❌ (lock-free) |
| **Performance** | Fastest | Slowest (blocking) | Fast (CAS) |
| **Use for** | Flags, DCL | Critical sections, complex state | Counters, statistics |

`AtomicInteger` uses CAS (Compare-And-Swap) — a hardware instruction that atomically checks if the current value matches expected, and if so, updates it. No thread blocking, but can spin under high contention.

---

## 3. synchronized — Mutual Exclusion

### 💻 Code Example
```java
class Counter {
    private int count = 0;
    private final Object lock = new Object(); // explicit lock object

    // Synchronized method — lock on 'this'
    public synchronized void increment() {
        count++; // read-modify-write is now atomic
    }

    // Synchronized block — more granular, better performance
    public void incrementWithBlock() {
        // do non-critical work here
        synchronized (lock) { // lock on specific object, not 'this'
            count++;
        }
        // do more non-critical work
    }

    // Synchronized static method — lock on Class object
    public static synchronized void classLevel() { ... }
}
```

### ❓ Interview Question
> "What is a deadlock? How do you prevent it?"

### ✅ Model Answer
A deadlock occurs when two or more threads are each waiting for a lock held by the other, forming a circular dependency. Prevention strategies:
1. **Lock ordering**: Always acquire locks in the same global order (if T1 needs lock A then B, T2 must also acquire A before B)
2. **Lock timeout**: Use `tryLock(timeout)` from `ReentrantLock` — give up if can't acquire in time
3. **Avoid nested locks**: Don't hold one lock while acquiring another
4. **Use higher-level abstractions**: `java.util.concurrent` classes handle locking internally

```java
// Deadlock scenario
// Thread 1: lock(A) then lock(B)
// Thread 2: lock(B) then lock(A)  → deadlock!

// Prevention: always lock in order A → B
private static final Object LOCK_A = new Object();
private static final Object LOCK_B = new Object();

void safeMethod() {
    synchronized (LOCK_A) {         // always A first
        synchronized (LOCK_B) {     // then B
            // critical section
        }
    }
}
```

---

## 4. ReentrantLock — Advanced Locking

### 💻 Code Example
```java
class BankAccount {
    private double balance;
    private final ReentrantLock lock = new ReentrantLock(true); // fair=true → FIFO

    public void transfer(BankAccount other, double amount) {
        lock.lock();
        try {
            if (balance < amount) throw new InsufficientFundsException();
            balance -= amount;
            other.deposit(amount);
        } finally {
            lock.unlock(); // ALWAYS in finally!
        }
    }

    // tryLock — non-blocking attempt
    public boolean tryTransfer(BankAccount other, double amount) {
        try {
            if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
                try {
                    balance -= amount;
                    return true;
                } finally {
                    lock.unlock();
                }
            }
            return false; // couldn't acquire lock in time
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
}
```

### 🔑 ReentrantLock vs synchronized
| | synchronized | ReentrantLock |
|--|-------------|---------------|
| **Timeout** | ❌ | ✅ tryLock(timeout) |
| **Interruptible** | ❌ | ✅ lockInterruptibly() |
| **Fairness** | ❌ | ✅ new ReentrantLock(true) |
| **Condition variables** | wait/notify (clunky) | Condition.await/signal |
| **Syntax** | Automatic release | Manual unlock (finally) |
| **Performance** | Similar in modern JVM | Slightly more overhead |

---

## 5. ExecutorService & Thread Pools

### 📌 Concept Summary
Creating threads manually is expensive. Thread pools reuse threads. `ExecutorService` is the right abstraction for executing tasks asynchronously.

### 💻 Code Example
```java
// Fixed thread pool — N threads, queue unbounded tasks
ExecutorService fixedPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() // CPU-bound: use # of CPUs
);

// Cached thread pool — creates new threads as needed, reuses idle ones
// Good for short-lived async tasks, BAD for long-running (can create too many threads)
ExecutorService cachedPool = Executors.newCachedThreadPool();

// Single thread executor — sequential execution guarantee
ExecutorService singleThread = Executors.newSingleThreadExecutor();

// Scheduled executor
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
scheduler.scheduleAtFixedRate(task, 0, 30, TimeUnit.SECONDS);

// PRODUCTION: use ThreadPoolExecutor directly for full control
ExecutorService customPool = new ThreadPoolExecutor(
    4,                               // corePoolSize
    8,                               // maximumPoolSize
    60L, TimeUnit.SECONDS,           // keepAliveTime for extra threads
    new LinkedBlockingQueue<>(1000), // bounded task queue
    new ThreadFactory() {
        private int count = 0;
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("worker-" + count++);
            t.setDaemon(false);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // when queue full: run in caller thread
);

// Submitting tasks
Future<String> future = fixedPool.submit(() -> {
    return "result";
});
String result = future.get(5, TimeUnit.SECONDS); // blocking with timeout

// Shutdown properly
fixedPool.shutdown(); // wait for tasks to finish
if (!fixedPool.awaitTermination(30, TimeUnit.SECONDS)) {
    fixedPool.shutdownNow(); // force shutdown
}
```

### ❓ Interview Question
> "What happens if the task queue of a ThreadPoolExecutor is full?"

### ✅ Model Answer
The `RejectedExecutionHandler` is invoked. Built-in policies:
- **AbortPolicy** (default): throw `RejectedExecutionException`
- **CallerRunsPolicy**: run the task in the calling thread (natural backpressure)
- **DiscardPolicy**: silently drop the task
- **DiscardOldestPolicy**: drop the oldest queued task and retry

For production services, **CallerRunsPolicy** is usually the safest — it slows down the caller, providing backpressure instead of losing tasks.

---

## 6. CompletableFuture — Async Pipelines

### 📌 Concept Summary
`CompletableFuture` is Java's promise/async mechanism. It lets you chain async operations without blocking threads.

### 💻 Code Example — Chaining
```java
// Basic async
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    return fetchUserFromDB(userId); // runs in ForkJoinPool.commonPool()
});

// Chain transformations
CompletableFuture<UserDTO> pipeline = CompletableFuture
    .supplyAsync(() -> userRepository.findById(userId))  // DB call
    .thenApply(user -> enrichWithProfile(user))           // transform (sync)
    .thenCompose(user -> fetchRecentOrders(userId)        // transform (returns CF)
        .thenApply(orders -> buildDTO(user, orders)))
    .exceptionally(ex -> {
        log.error("Failed to build user DTO", ex);
        return UserDTO.empty();
    });

// Run with custom executor (don't use common pool for blocking I/O!)
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(
    () -> blockingDatabaseCall(),
    Executors.newFixedThreadPool(10) // I/O pool, not common pool
);

// Wait for multiple
CompletableFuture<User> userFuture = fetchUser(userId);
CompletableFuture<List<Order>> ordersFuture = fetchOrders(userId);

CompletableFuture<UserProfile> combined = userFuture.thenCombine(
    ordersFuture,
    (user, orders) -> new UserProfile(user, orders)
);

// Wait for ALL
CompletableFuture<Void> allDone = CompletableFuture.allOf(cf1, cf2, cf3);
allDone.thenRun(() -> System.out.println("All done!"));

// Wait for ANY (first one to complete)
CompletableFuture<String> fastest = CompletableFuture.anyOf(mirror1, mirror2, mirror3)
    .thenApply(result -> (String) result);
```

### 💻 Code Example — thenApply vs thenCompose
```java
// thenApply: Function<T, U> — maps T to U (like Stream.map)
CompletableFuture<String> name = cf.thenApply(user -> user.getName());

// thenCompose: Function<T, CompletableFuture<U>> — flatMap (avoids CF<CF<T>>)
CompletableFuture<List<Order>> orders = cf.thenCompose(user -> fetchOrders(user.getId()));
// NOT: cf.thenApply(user -> fetchOrders(user.getId())) → CF<CF<List<Order>>> ❌
```

### ❓ Interview Question
> "What's the difference between `thenApply`, `thenApplyAsync`, and `thenCompose`?"

### ✅ Model Answer
- `thenApply(fn)` — runs `fn` in the **same thread** that completed the previous stage (no new thread)
- `thenApplyAsync(fn)` — runs `fn` in the **ForkJoinPool** (or provided executor) — use for CPU work
- `thenApplyAsync(fn, executor)` — runs `fn` in specified executor — use for I/O work
- `thenCompose(fn)` — like `thenApply` but fn returns a `CompletableFuture<U>`, flattening it (avoids `CF<CF<U>>`)

⚠️ **Gotcha:** Never do blocking I/O inside `thenApply` without `Async` suffix — you'll block the thread that completed the stage, potentially the ForkJoinPool's common thread, starving other completable futures.

---

## 7. ReadWriteLock — Concurrent Reads, Exclusive Writes

```java
class CachedData {
    private volatile Map<String, String> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    public String get(String key) {
        readLock.lock();        // multiple threads can read concurrently
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public void put(String key, String value) {
        writeLock.lock();       // exclusive — no reads or other writes
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

---

## 8. Thread-Safe Collections Cheat Sheet

```java
// For general concurrent use
ConcurrentHashMap<K, V>     // best concurrent map
CopyOnWriteArrayList<E>     // good for read-heavy, rare writes (snapshots each write)
CopyOnWriteArraySet<E>      // same
ConcurrentLinkedQueue<E>    // non-blocking concurrent queue
ConcurrentSkipListMap<K,V>  // sorted concurrent map

// Blocking queues (producer-consumer pattern)
LinkedBlockingQueue<E>      // bounded or unbounded FIFO
ArrayBlockingQueue<E>       // bounded FIFO, fair option
PriorityBlockingQueue<E>    // unbounded priority queue
SynchronousQueue<E>         // zero capacity — direct handoff

// Producer-consumer pattern
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);

// Producer
queue.put(task);         // blocks if full

// Consumer
Task task = queue.take(); // blocks if empty
```

---

*Next: [05-jvm-internals.md](./05-jvm-internals.md)*


