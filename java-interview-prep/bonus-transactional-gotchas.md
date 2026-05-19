# @Transactional Đang Lừa Bạn — 5 Cách Annotation Này Không Hoạt Động Như Bạn Tưởng

> **Đọc trong:** 14 phút  
> **Áp dụng cho:** Spring Boot 2.x, 3.x — tất cả phiên bản  
> **Tags:** `spring-boot` `transactions` `jpa` `hibernate` `gotchas`

---

Đây là bug tôi gặp lần đầu ở một e-commerce service xử lý thanh toán:

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        notificationService.sendConfirmationEmail(order); // throws RuntimeException
        // Expected: order bị rollback
        // Actual: order vẫn được lưu vào DB
    }
}
```

Unit test pass. Staging pass. Production: khách hàng bị charge tiền nhưng không nhận được email xác nhận, order vẫn tồn tại trong DB dù exception đã throw.

`@Transactional` có trên method. RuntimeException được throw. Rollback phải xảy ra — đó là cách nó hoạt động.

Nhưng nó không rollback. Và lý do thì không phải Spring bị bug.

---

## Tại sao `@Transactional` phức tạp hơn bạn nghĩ

Spring xử lý `@Transactional` bằng **AOP proxy**. Khi bạn gọi một `@Transactional` method, bạn không gọi trực tiếp object đó — bạn gọi qua một proxy được Spring tạo ra ở runtime:

```
Caller → [Spring Proxy] → Your Service Method
              ↓
    begin transaction
    invoke method
    commit / rollback
```

Cơ chế này hoạt động tốt trong 80% trường hợp. Nhưng 5 edge cases dưới đây sẽ khiến transaction silently không hoạt động — không lỗi compile, không warning runtime, chỉ là data corrupt.

---

## Lỗi 1: Self-invocation — gọi method trong cùng class

Đây là lỗi phổ biến nhất. Và cũng là lỗi khó debug nhất vì code trông hoàn toàn hợp lý:

```java
@Service
public class UserService {

    public void registerUser(UserRequest request) {
        validateRequest(request);
        createUserWithTransaction(request); // ← gọi method trong cùng class
    }

    @Transactional
    public void createUserWithTransaction(UserRequest request) {
        userRepository.save(new User(request));
        auditRepository.save(new AuditLog(request));
        // Exception ở đây sẽ KHÔNG rollback
    }
}
```

Khi `registerUser` gọi `createUserWithTransaction` — nó gọi trực tiếp trên `this`, không qua proxy. Transaction annotation bị **bỏ qua hoàn toàn**. Không có transaction nào được mở.

Cách fix:

```java
// Option 1 — Inject chính service vào mình (Spring cho phép)
@Service
public class UserService {

    @Autowired
    private UserService self; // Spring inject proxy, không phải this

    public void registerUser(UserRequest request) {
        validateRequest(request);
        self.createUserWithTransaction(request); // qua proxy → transaction hoạt động
    }

    @Transactional
    public void createUserWithTransaction(UserRequest request) {
        userRepository.save(new User(request));
        auditRepository.save(new AuditLog(request));
    }
}

// Option 2 — Tách ra service khác (cleaner)
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserCreationService userCreationService;

    public void registerUser(UserRequest request) {
        validateRequest(request);
        userCreationService.createUser(request);
    }
}

@Service
public class UserCreationService {
    @Transactional
    public void createUser(UserRequest request) { ... }
}
```

---

## Lỗi 2: Checked exception không trigger rollback

Quay lại bug thanh toán ở đầu bài. `notificationService.sendConfirmationEmail()` throw một **checked exception** — `MessagingException`. Không phải `RuntimeException`.

```java
@Transactional // mặc định chỉ rollback cho unchecked exceptions
public void placeOrder(OrderRequest request) throws MessagingException {
    Order order = orderRepository.save(new Order(request));
    notificationService.sendConfirmationEmail(order); // throws MessagingException (checked)
    // Transaction COMMIT — không rollback
}
```

Theo mặc định, `@Transactional` chỉ rollback khi gặp `RuntimeException` và `Error`. Checked exception → commit bình thường.

Lý do: Spring theo convention của Java EE — checked exception được coi là "expected business errors", không phải "unexpected failures".

```java
// Fix rõ ràng — khai báo rollbackFor
@Transactional(rollbackFor = Exception.class)
public void placeOrder(OrderRequest request) throws MessagingException {
    Order order = orderRepository.save(new Order(request));
    notificationService.sendConfirmationEmail(order); // throws MessagingException
    // Bây giờ rollback đúng
}

// Hoặc cụ thể hơn
@Transactional(rollbackFor = MessagingException.class)
public void placeOrder(OrderRequest request) throws MessagingException { ... }
```

Ngược lại, nếu bạn muốn RuntimeException **không** rollback:

```java
@Transactional(noRollbackFor = ValidationException.class)
public void processWithValidation(Request request) {
    // ValidationException throw nhưng transaction vẫn commit
    // Dùng khi bạn muốn log lỗi nhưng vẫn save partial data
}
```

---

## Lỗi 3: `@Transactional` trên private method

```java
@Service
public class PaymentService {

    public void processPayment(Payment payment) {
        savePaymentRecord(payment); // gọi private method
    }

    @Transactional // ← annotation này bị IGNORE hoàn toàn
    private void savePaymentRecord(Payment payment) {
        paymentRepository.save(payment);
        ledgerRepository.save(new LedgerEntry(payment));
    }
}
```

Spring proxy chỉ intercept **public methods**. `@Transactional` trên private method không tạo ra transaction. Không có lỗi. Không có cảnh báo. Annotation chỉ nằm đó làm màu.

```java
// Fix — public method
@Transactional
public void savePaymentRecord(Payment payment) { // public
    paymentRepository.save(payment);
    ledgerRepository.save(new LedgerEntry(payment));
}
```

Nếu bạn dùng IntelliJ, nó sẽ warn: *"@Transactional on private method will be ignored"*. Nếu không có IDE warning — chú ý tay.

---

## Lỗi 4: Transaction propagation bẫy N+1

Đây là lỗi không phải về rollback, mà về **quá nhiều transactions**:

```java
@Service
public class ReportService {

    @Transactional(readOnly = true)
    public List<OrderSummary> generateDailyReport() {
        List<Order> orders = orderRepository.findAllToday(); // 500 orders
        return orders.stream()
            .map(this::buildSummary)
            .collect(Collectors.toList());
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW) // ← vấn đề
    private OrderSummary buildSummary(Order order) {
        // Đây sẽ mở transaction mới cho MỖI order
        // → 500 orders = 500 transactions = 500 DB roundtrips
        return new OrderSummary(order, auditRepository.findByOrderId(order.getId()));
    }
}
```

`REQUIRES_NEW` luôn mở transaction mới, suspend transaction hiện tại. Trong loop trên 500 items = 500 transaction open/close cycles. Report chạy 30 giây thay vì 2 giây.

```java
// Fix — giữ một transaction cho toàn bộ report
@Transactional(readOnly = true)
public List<OrderSummary> generateDailyReport() {
    List<Order> orders = orderRepository.findAllToday();

    // Fetch tất cả audit logs trong một query
    List<Long> orderIds = orders.stream().map(Order::getId).collect(Collectors.toList());
    Map<Long, AuditLog> auditMap = auditRepository.findByOrderIdIn(orderIds)
        .stream().collect(Collectors.toMap(AuditLog::getOrderId, Function.identity()));

    return orders.stream()
        .map(order -> new OrderSummary(order, auditMap.get(order.getId())))
        .collect(Collectors.toList());
}
```

Khi nào dùng `REQUIRES_NEW` đúng: khi bạn cần một operation **luôn commit độc lập** bất kể transaction cha có rollback hay không — ví dụ: log lỗi vào DB ngay cả khi business transaction fail.

```java
// Dùng đúng REQUIRES_NEW — audit log luôn save dù order fail
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditLog(AuditEvent event) {
    auditRepository.save(new AuditLog(event)); // always commits
}
```

---

## Lỗi 5: `readOnly = true` — hiểu sai công dụng

```java
// Nhiều developer nghĩ readOnly = true sẽ NGĂN ghi vào DB
@Transactional(readOnly = true)
public User getUserById(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    user.setLastAccessTime(LocalDateTime.now()); // thay đổi entity
    return user; // vẫn trả về, không save
}
```

Trên một số cấu hình Hibernate, `readOnly = true` **vẫn có thể** flush thay đổi vào DB — behavior phụ thuộc vào JPA provider và flush mode.

Nhưng quan trọng hơn: `readOnly = true` không phải security feature. Công dụng thực sự là:

```
1. Hint cho database → có thể dùng read replica
2. Hibernate tắt dirty checking → giảm memory overhead
3. Một số DB driver optimize read-only connections
```

```java
// Đúng: readOnly = true cho các query methods, nhưng không mutate entity
@Transactional(readOnly = true)
public List<OrderDto> getOrderHistory(Long userId) {
    return orderRepository.findByUserId(userId)
        .stream()
        .map(OrderDto::from) // map sang DTO, không mutate entity
        .collect(Collectors.toList());
}

// Nếu cần update lastAccess — dùng transaction ghi
@Transactional
public User getUserAndUpdateAccess(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    user.setLastAccessTime(LocalDateTime.now());
    return user; // Hibernate dirty check sẽ save khi transaction commit
}
```

---

## Quick reference — transaction propagation hay dùng nhất

| Propagation | Behavior | Dùng khi |
|-------------|----------|----------|
| `REQUIRED` (default) | Join existing hoặc tạo mới | 95% cases |
| `REQUIRES_NEW` | Luôn tạo transaction mới, suspend cái cũ | Audit log, retry logic |
| `SUPPORTS` | Join nếu có, không tạo mới nếu không có | Read-only optional transaction |
| `NOT_SUPPORTED` | Suspend transaction hiện tại | Cache hits không cần transaction |
| `NEVER` | Throw exception nếu có transaction đang chạy | Enforce no-transaction zones |
| `MANDATORY` | Join existing, throw nếu không có | Enforce "must be called inside transaction" |

---

## Checklist trước khi ship bất kỳ `@Transactional` method nào

```
□ Method có phải public không?
□ Được gọi từ ngoài class hay self-invocation?
□ Nếu có checked exception — có rollbackFor không?
□ readOnly = true nhưng không mutate entity không?
□ Nếu dùng REQUIRES_NEW — có thực sự cần độc lập không?
□ Transaction scope có quá rộng (bao toàn bộ request) không?
```

---

`@Transactional` là annotation đơn giản nhất để thêm vào, và là một trong những annotation hay bị hiểu sai nhất trong Spring. Không phải vì nó phức tạp — mà vì proxy-based AOP là cơ chế ẩn, behavior mặc định không phải lúc nào cũng là behavior bạn cần, và failures thì im lặng.

Năm lỗi trên đều có cùng một đặc điểm: code compile được, tests thường pass, chỉ production với concurrent load mới lộ ra. Checklist ở trên tốn 2 phút review. Data corruption thì tốn nhiều hơn.

---

## Đọc thêm

- [Spring Docs — Transaction Management](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction) — propagation matrix đầy đủ
- [Vlad Mihalcea — Spring @Transactional gotchas](https://vladmihalcea.com/spring-transactional-annotation/) — deep dive về Hibernate interaction
- [Baeldung — Transaction Propagation and Isolation](https://www.baeldung.com/spring-transactional-propagation-isolation) — ví dụ cho từng propagation type
