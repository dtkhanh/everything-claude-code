# HikariCP Connection Pool Starvation — Khi App Của Bạn Chậm Lại Và Bạn Cứ Nghĩ Tại Database

> **Loại:** Bonus Blog — Spring Boot Production Deep Dive  
> **Đọc trong:** 15 phút  
> **Áp dụng cho:** Spring Boot 2.x / 3.x, HikariCP (default), bất kỳ RDBMS nào  
> **Tags:** `hikaricp` `connection-pool` `spring-boot` `performance` `database`

---

Đây là alert lúc 2 giờ sáng thứ Sáu:

```
WARN  HikariPool-1 - Connection is not available, 
      request timed out after 30000ms
```

Response time tăng từ 80ms lên 4,000ms. Database CPU bình thường. Disk I/O bình thường. Query execution time bình thường. Không có gì sai ở database cả.

Vấn đề nằm ở **app của bạn** — không phải database. Và nó có tên: **connection pool starvation**.

---

## Vấn đề là gì

HikariCP giữ một pool gồm N connections tới database. Mỗi request cần 1 connection, dùng xong trả lại pool. Default của Spring Boot là **10 connections**.

Starvation xảy ra khi tất cả 10 connections đều đang bị giữ — request thứ 11 đứng chờ. Nếu không có connection nào được trả lại trong `connectionTimeout` (mặc định 30 giây) — `SQLTimeoutException`.

Nghe đơn giản. Nhưng lý do pool cạn kiệt thường **không phải do có quá nhiều request**. Phần lớn là do code đang giữ connection **lâu hơn cần thiết**.

---

## Cách detect trước khi nó thành incident

Bật HikariCP metrics vào log — đây là thứ đầu tiên cần làm trước khi optimize bất cứ thứ gì:

```yaml
# application.yml
logging:
  level:
    com.zaxxer.hikari: DEBUG
    com.zaxxer.hikari.HikariConfig: DEBUG

management:
  endpoints:
    web:
      exposure:
        include: health, metrics
  metrics:
    enable:
      hikari: true
```

Sau đó gọi:
```
GET /actuator/metrics/hikaricp.connections.active
GET /actuator/metrics/hikaricp.connections.pending
GET /actuator/metrics/hikaricp.connections.acquire
```

Nếu `connections.pending` > 0 thường xuyên — pool của bạn đang có vấn đề.

Hoặc dùng HikariCP listener để log realtime:

```java
@Configuration
public class HikariMetricsConfig {

    @Bean
    public DataSourcePoolMetadataProvider hikariPoolMetadataProvider() {
        return dataSource -> {
            if (dataSource instanceof HikariDataSource hikari) {
                HikariPoolMXBean pool = hikari.getHikariPoolMXBean();
                // Log mỗi 10 giây
                Executors.newSingleThreadScheduledExecutor()
                    .scheduleAtFixedRate(() ->
                        log.info("HikariCP — active={}, idle={}, pending={}, total={}",
                            pool.getActiveConnections(),
                            pool.getIdleConnections(),
                            pool.getThreadsAwaitingConnection(),
                            pool.getTotalConnections()),
                        0, 10, TimeUnit.SECONDS);
            }
            return null;
        };
    }
}
```

Khi `ThreadsAwaitingConnection` > 0 — starvation đang bắt đầu.

---

## Bốn nguyên nhân phổ biến nhất

### 1. `@Transactional` mở quá sớm, đóng quá muộn

Đây là pattern tôi thấy nhiều nhất trong code review:

```java
// ❌ Transaction mở từ đầu method — giữ connection suốt
@Transactional
public OrderConfirmation processOrder(OrderRequest request) {
    // 1. Validate request — không cần DB
    validateBusinessRules(request);         // 50ms — CPU only

    // 2. Call external payment gateway — không cần DB!
    PaymentResult payment = stripeClient.charge(request.getAmount()); // 800ms — HTTP call

    // 3. Call SMS service — không cần DB!
    smsService.sendConfirmation(request.getPhone()); // 300ms — HTTP call

    // 4. Finally save to DB — cần 50ms thực sự
    Order order = orderRepository.save(buildOrder(request, payment));
    return buildConfirmation(order);
}
```

Connection được checkout từ pool ngay khi method bắt đầu. Nhưng 1,150ms đầu — không có operation nào cần connection. App đang giữ connection trong khi chờ Stripe và SMS service.

Với 10 concurrent requests theo pattern này: 10 connections bị chiếm, 10 × 1,150ms = thảm họa.

```java
// ✅ Transaction chỉ bao quanh phần thực sự cần DB
public OrderConfirmation processOrder(OrderRequest request) {
    // Validate — no DB, no transaction
    validateBusinessRules(request);

    // External calls — no DB, no transaction
    PaymentResult payment = stripeClient.charge(request.getAmount());
    smsService.sendConfirmation(request.getPhone());

    // Chỉ bao transaction ở đây — connection giữ ~50ms thay vì 1,200ms
    return saveOrderInTransaction(request, payment);
}

@Transactional
private OrderConfirmation saveOrderInTransaction(OrderRequest request, PaymentResult payment) {
    Order order = orderRepository.save(buildOrder(request, payment));
    return buildConfirmation(order);
}
```

Connection hold time giảm từ 1,200ms xuống 50ms. Pool cùng 10 connection phục vụ được 24x nhiều concurrent requests hơn.

---

### 2. N+1 query trong transaction — connection giữ lâu vì query chạy nhiều

```java
// ❌ Load 1000 orders → 1001 queries, connection mở suốt
@Transactional(readOnly = true)
public List<OrderSummary> getOrderSummaries() {
    List<Order> orders = orderRepository.findAll(); // 1 query

    return orders.stream()
        .map(order -> {
            // Hibernate lazy load — 1 query per order
            String customerName = order.getCustomer().getName(); // +1000 queries
            return new OrderSummary(order.getId(), customerName);
        })
        .toList();
}
```

1001 queries chạy trong 1 transaction = connection bị giữ trong suốt thời gian đó. Nếu có nhiều request đồng thời cùng gọi endpoint này — pool cạn.

```java
// ✅ JOIN FETCH — 1 query duy nhất, connection giải phóng nhanh
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findAllWithCustomer(@Param("status") OrderStatus status);

// Hoặc dùng DTO projection thẳng — không cần load entity
@Query("SELECT new com.example.OrderSummary(o.id, c.name) " +
       "FROM Order o JOIN o.customer c")
List<OrderSummary> findOrderSummaries();
```

---

### 3. Scheduled job và batch process không limit concurrency

```java
// ❌ 10,000 items → 10,000 concurrent tasks → tất cả tranh pool
@Scheduled(fixedDelay = 60_000)
public void syncInventory() {
    List<Product> products = productRepository.findAll(); // 10,000 products

    products.parallelStream()  // ← đây là vấn đề
        .forEach(product -> {
            // Mỗi task giữ 1 connection
            inventoryService.syncProduct(product);
        });
}
```

`parallelStream()` dùng ForkJoinPool common pool — có thể spawn hàng trăm thread đồng thời, mỗi thread cần 1 DB connection. Pool 10 connection vỡ ngay.

```java
// ✅ Batch xử lý tuần tự hoặc dùng fixed thread pool nhỏ
@Scheduled(fixedDelay = 60_000)
public void syncInventory() {
    List<Product> products = productRepository.findAll();

    // Xử lý tuần tự — predictable connection usage
    products.forEach(product -> inventoryService.syncProduct(product));

    // Hoặc batch với controlled parallelism
    ExecutorService executor = Executors.newFixedThreadPool(3); // tối đa 3 connections
    try {
        List<CompletableFuture<Void>> futures = products.stream()
            .map(p -> CompletableFuture.runAsync(
                () -> inventoryService.syncProduct(p), executor))
            .toList();
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    } finally {
        executor.shutdown();
    }
}
```

---

### 4. Connection bị leak — không được trả về pool

Đây là nguyên nhân nguy hiểm nhất vì nó **tích lũy** — app hoạt động bình thường ban đầu, sau 6 tiếng thì pool cạn và không tự phục hồi nếu không restart.

```java
// ❌ Connection leak — không close trong finally
public void migrateData() {
    Connection conn = dataSource.getConnection(); // checkout từ pool
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery("SELECT * FROM legacy_data");
    // Nếu exception xảy ra trước đây...
    while (rs.next()) {
        processRow(rs);
    }
    conn.close(); // ...dòng này không bao giờ được gọi
}
```

```java
// ✅ try-with-resources — auto-close đảm bảo trả connection về pool
public void migrateData() {
    try (Connection conn = dataSource.getConnection();
         Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery("SELECT * FROM legacy_data")) {
        while (rs.next()) {
            processRow(rs);
        }
    } // conn.close() được gọi tự động, kể cả khi exception
}
```

HikariCP có thể detect leak cho bạn:

```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 2000  # ms — log warning nếu connection giữ > 2 giây
```

Log khi có leak:
```
WARN  HikariPool-1 - Connection leak detection triggered for connection 
      com.zaxxer.hikari.pool.ProxyConnection@..., stack trace follows
java.lang.Exception: Apparent connection leak detected
    at com.example.DataMigrationService.migrateData(DataMigrationService.java:23)
```

Stack trace này chỉ thẳng đến dòng code gây ra leak. Không cần đoán.

---

## Sizing pool đúng — con số 10 không phải lúc nào cũng đúng

Công thức thực tế của HikariCP:

```
pool size = (core_count * 2) + effective_spindle_count
```

Với server 4 CPU cores, SSD (spindle = 1):
```
pool size = (4 * 2) + 1 = 9 ≈ 10
```

Default 10 chuẩn cho **CPU-bound** workload. Nhưng với **I/O-heavy** workload (nhiều external API call trong transaction — như Bẫy 1), bạn cần tính khác:

```
pool size = concurrent_requests × average_connection_hold_time / request_duration
```

Ví dụ: 50 concurrent requests, mỗi request giữ connection 50ms trong tổng 1,000ms:
```
pool size = 50 × 50ms / 1000ms = 2.5 → 3 connections là đủ
```

App của bạn thực sự không cần pool lớn hơn — cần **giảm connection hold time** (Bẫy 1).

Config production-ready:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10          # đừng tăng mù quáng
      minimum-idle: 2                # giảm cost khi traffic thấp
      connection-timeout: 3000       # 3s — fail fast, đừng để user đợi 30s
      idle-timeout: 600000           # 10 phút — close idle connections
      max-lifetime: 1800000          # 30 phút — recycle để tránh stale
      leak-detection-threshold: 2000 # 2s — bật leak detection ở production
      validation-timeout: 1000       # 1s — timeout khi validate connection còn sống
```

`connection-timeout: 3000` thay vì default 30,000 là thay đổi quan trọng nhất: fail fast sau 3 giây thay vì để user đợi 30 giây. Error 503 sau 3 giây tốt hơn nhiều so với loading spinner 30 giây.

---

## Checklist khi thấy pool starvation

```
□ Bật leak-detection-threshold: 2000 — chạy 1 tiếng, check log
□ Kiểm tra @Transactional bao quanh external HTTP calls không
□ Tìm parallelStream() bên trong @Transactional hoặc Scheduled jobs
□ Kiểm tra N+1: SELECT count > (table row count) trong slow query log
□ Thêm /actuator/metrics/hikaricp.connections.pending vào dashboard
□ Set connection-timeout xuống 3000ms — đừng để user đợi 30 giây
□ Đặt maximum-pool-size dựa trên hold time, không phải concurrent users
```

---

Pool starvation trông giống database down. Metrics nói khác: DB perfect, app chết. Và khi biết tìm ở đâu — connection hold time, leak detection, transaction boundary — thường fix được trong một buổi sáng mà không cần thêm hardware, không cần tăng pool size, không cần đổi database.

Cái cần đổi là **biết connection đang nằm ở đâu trong 1,200ms đó**.

---

## Tài liệu tham khảo

- [HikariCP — About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Connection-Pool-Sizing) — công thức sizing chính thức từ tác giả
- [HikariCP — Connection Leak Detection](https://github.com/brettwooldridge/HikariCP/wiki/MBean-(JMX)-Monitoring-and-Management) — JMX monitoring
- [Spring Boot — Data Access](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.sql.datasource.connection-pool) — HikariCP config reference
- [Vlad Mihalcea — The best way to detect HikariCP connection pool issues](https://vladmihalcea.com/hikaricp-connection-pool/) — deep dive từ Hibernate expert
