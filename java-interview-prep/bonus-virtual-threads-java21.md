# Virtual Threads Sẽ Không Cứu Bạn Nếu Bạn Dùng Sai

> **Đọc trong:** 12 phút  
> **Áp dụng cho:** Spring Boot 3.2+, Java 21+  
> **Tags:** `virtual-threads` `project-loom` `spring-boot` `performance` `concurrency`

---

Tôi gặp con số này lần đầu trong một PR review nội bộ:

```
Before (platform threads, Tomcat max=200):  1,200 req/s  |  P99: 380ms
After  (virtual threads, Spring Boot 3.2):  9,800 req/s  |  P99:  42ms
```

Cùng một app. Cùng một database. Cùng hardware. 3 dòng config.

Tôi không tin ngay. Tôi chạy lại. Số giống nhau.

Nhưng sau đó tôi thử trên một service khác — CPU-heavy report generation — và không có gì thay đổi cả. Đó là lúc tôi bắt đầu thực sự tìm hiểu cơ chế, thay vì chỉ copy config và hy vọng.

---

## Tại sao server của bạn nghẽn dù CPU còn thừa

Spring Boot dùng **thread-per-request model**. Mỗi HTTP request giữ một platform thread suốt vòng đời của nó.

Platform thread tốn ~1MB RAM và có OS context-switch cost. Tomcat mặc định giới hạn 200 threads. Khi 200 requests đang đợi database trả lời — request thứ 201 xếp hàng, dù CPU của bạn đang ngồi chơi ở 3%.

Nhìn vào một endpoint điển hình:

```java
User user = userRepository.findById(id).orElseThrow();  // blocked ~5ms
Order order = orderRepository.findByUserId(id);          // blocked ~8ms
Payment pay = paymentClient.getLatest(user.getId());     // blocked ~120ms
```

Thread này "đang xử lý" nhưng thực ra đang ngồi chờ network và disk trong 133ms. CPU không được dùng. Thread không làm gì. Nhưng nó vẫn chiếm slot trong pool.

Cộng đồng Java đã giải quyết bằng reactive programming — WebFlux, Project Reactor. Code không block nữa, nhưng stack trace trở thành:

```
at reactor.core.publisher.FluxFlatMap$FlatMapInner.onNext(FluxFlatMap.java:985)
at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:122)
at reactor.core.publisher.FluxConcatArray$ConcatArraySubscriber.onNext(...)
// ... 43 frames nữa để debug một cái NPE
```

Virtual Threads đi theo hướng khác: **giữ nguyên blocking code style, nhưng khi thread block tại I/O, JVM tự unmount khỏi OS thread và dùng OS thread đó cho việc khác**.

---

## Cơ chế hoạt động

```
┌────────────────────────────────────────────────────────┐
│  Virtual Threads (hàng triệu — chỉ tốn vài KB/cái)   │
│  VT-1  VT-2  VT-3  VT-4  VT-5  VT-6  VT-7  ...       │
│    ↓     ↓     ↓     ↓                                 │
│  ┌─────────────────────────────────┐                   │
│  │  Carrier Threads (= CPU cores)  │                   │
│  │  CT-1    CT-2    CT-3    CT-4   │                   │
│  └─────────────────────────────────┘                   │
└────────────────────────────────────────────────────────┘
```

**Virtual Thread** là object trên JVM heap, vài KB. JVM quản lý lifecycle của nó.

**Carrier Thread** là platform thread thực sự — số lượng bằng CPU cores, chạy trên OS.

Khi VT-1 gọi `userRepository.findById(id)` và block trên database I/O:
1. JVM **unmounts** VT-1 khỏi CT-1
2. CT-1 tự do — JVM mount VT-5 vào CT-1 để tiếp tục chạy
3. Khi database trả kết quả, VT-1 được schedule lại vào CT bất kỳ đang rảnh

Kết quả: 4 CPU cores phục vụ 10,000 concurrent requests đang chờ I/O mà không cần thread pool sizing hay callback hell.

---

## Bật lên trong Spring Boot 3.2+

```yaml
# application.yml — chỉ cần thế này
spring:
  threads:
    virtual:
      enabled: true
```

Ba chữ cái. Spring Boot lo phần còn lại: Tomcat executor, `@Async`, `@Scheduled`, `TaskExecutor` — tất cả tự động chuyển sang virtual threads.

Verify đang chạy đúng:

```java
@GetMapping("/thread-info")
public String threadInfo() {
    Thread t = Thread.currentThread();
    return "virtual=" + t.isVirtual(); // → virtual=true
}
```

Nếu muốn control rõ hơn, programmatic:

```java
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerCustomizer() {
    return handler -> handler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}
```

---

## Ba thứ sẽ phá vỡ performance của bạn

### 1. `synchronized` block ghim virtual thread xuống carrier

Tôi đã thấy cái này trong production code của một payment service. Method xử lý charge được đánh `synchronized` "để thread-safe":

```java
// ❌ Cái bẫy phổ biến nhất
public synchronized void processPayment(Order order) {
    paymentGateway.charge(order); // HTTP call tốn 200ms — bên trong synchronized
}
```

Vấn đề: khi VT gọi blocking I/O **bên trong** `synchronized`, JVM không thể unmount. Carrier thread bị **pinned** — blocked hoàn toàn giống platform thread cũ. 500 concurrent payments = 500 carrier threads bị ghim = server chết.

```java
// ✅ ReentrantLock — VT có thể unmount khi đang chờ lock và khi đang I/O
private final ReentrantLock lock = new ReentrantLock();

public void processPayment(Order order) {
    lock.lock();
    try {
        paymentGateway.charge(order);
    } finally {
        lock.unlock();
    }
}
```

Tìm pinning trong codebase:

```bash
# Scan tất cả synchronized methods có potential I/O bên trong
grep -r "synchronized" src/main/java --include="*.java" -l

# Runtime detection — chạy một lần trong dev
java -Djdk.tracePinnedThreads=full -jar app.jar
# Khi có pinning sẽ in ra: Thread[ForkJoinPool-1-worker-1] / java.lang.VirtualThread$PinnedScope
```

---

### 2. CPU-bound tasks — virtual threads không giúp được gì

Đây là lý do service report generation của tôi không cải thiện khi bật virtual threads:

```java
// ❌ Virtual thread executor cho CPU work — không có lợi ích
@Async
public CompletableFuture<byte[]> generateReport(ReportRequest req) {
    // 100% CPU, không có I/O checkpoint để JVM unmount
    return CompletableFuture.completedFuture(
        pdfRenderer.render(req) // tốn 3 giây CPU liên tục
    );
}
```

VT chỉ unmount tại I/O checkpoint. Không có I/O = không có unmount = virtual thread ngồi chặn carrier thread suốt 3 giây.

```java
// ✅ Tách riêng executor cho CPU tasks
@Bean("cpuPool")
public Executor cpuPool() {
    return new ForkJoinPool(Runtime.getRuntime().availableProcessors());
}

@Async("cpuPool")
public CompletableFuture<byte[]> generateReport(ReportRequest req) {
    return CompletableFuture.completedFuture(pdfRenderer.render(req));
}
```

---

### 3. `ThreadLocal` — không nguy hiểm hơn, nhưng đây là lúc để dọn dẹp

Với platform thread pool, `ThreadLocal` không được clear giữa các requests là data leak nghiêm trọng. Với virtual threads — mỗi request có virtual thread riêng, nên leak ít hơn.

Nhưng Java 21 có **Scoped Values** — sạch hơn, scope tự cleanup, không cần nhớ `remove()`:

```java
// Trước — ThreadLocal, dễ quên clear
private static final ThreadLocal<String> TENANT = new ThreadLocal<>();
TENANT.set(tenantId);
// ... sau khi dùng xong phải gọi TENANT.remove() trong finally

// Sau — Scoped Values, tự cleanup khi ra khỏi scope
private static final ScopedValue<String> TENANT = ScopedValue.newInstance();

ScopedValue.where(TENANT, tenantId).run(() -> {
    dataService.fetchForTenant(); // TENANT.get() available trong đây
});
// Tự cleanup
```

---

## Tự đo trên máy bạn

Đừng tin số của tôi. Chạy cái này:

```java
@Test
void virtualVsPlatform() throws Exception {
    int N = 1000;
    CountDownLatch latch = new CountDownLatch(N);

    // Đổi dòng này để so sánh:
    // ExecutorService exec = Executors.newFixedThreadPool(200);   // platform
    ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor(); // virtual

    long start = System.currentTimeMillis();

    for (int i = 0; i < N; i++) {
        exec.submit(() -> {
            try { Thread.sleep(100); }  // simulate DB wait
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            finally { latch.countDown(); }
        });
    }

    latch.await(30, TimeUnit.SECONDS);
    System.out.println("Elapsed: " + (System.currentTimeMillis() - start) + "ms");
    // Virtual:  ~105ms  (1000 tasks chạy song song)
    // Platform: ~510ms  (5 batches × 200 threads × 100ms)
}
```

Con số chênh lệch rõ nhất khi số concurrent tasks lớn hơn nhiều so với số platform threads.

---

## Nên bật hay không

| Workload | Kết quả |
|----------|---------|
| REST API gọi database | ✅ Lợi rõ — I/O-bound |
| REST API gọi external HTTP | ✅ Lợi rõ — I/O-bound |
| File upload / processing | ✅ Lợi — I/O-bound |
| PDF / Excel generation | ❌ Không đổi — CPU-bound |
| ML inference trong JVM | ❌ Không đổi — CPU-bound |
| Có nhiều `synchronized` + I/O chưa refactor | ⚠️ Scan pinning trước |

---

## Checklist migration

Trước khi bật trong production:

```bash
# 1. Tìm synchronized methods có I/O bên trong
grep -rn "synchronized" src/main/java --include="*.java"

# 2. Chạy dev với pinning detection
java -Djdk.tracePinnedThreads=full -jar app.jar

# 3. Load test — so sánh trước và sau
ab -n 10000 -c 500 http://localhost:8080/your-endpoint
```

Nếu `tracePinnedThreads` không in gì — bật production. Nếu in ra pinning — fix `synchronized` → `ReentrantLock` trước.

---

Virtual Threads giải quyết một vấn đề cụ thể: **platform thread bị lãng phí khi chờ I/O**. Không phải CPU bottleneck. Không phải bad query. Không phải connection pool quá nhỏ. Đúng vấn đề đó thôi.

Với I/O-bound microservices điển hình — đây là migration ít rủi ro nhất mà Java 21 đem lại. 3 dòng config, đo trước và sau, quyết định trong 30 phút.

---

## Đọc thêm

- [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444) — spec đầy đủ, đọc phần "Pinning" và "Scheduling"
- [Spring Blog — Embracing Virtual Threads](https://spring.io/blog/2022/10/11/embracing-virtual-threads) — migration guide từ team Spring
- [JEP 446 — Scoped Values](https://openjdk.org/jeps/446) — thay thế ThreadLocal
- [HikariCP connection pool sizing với VT](https://github.com/brettwooldridge/HikariCP/wiki/About-Connection-Pool-Sizing) — đừng để pool size làm bottleneck sau khi bật VT






