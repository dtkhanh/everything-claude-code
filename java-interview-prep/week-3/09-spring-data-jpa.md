# Week 3 — Day 6–7: Spring Data JPA

> 📖 **Estimated reading time:** 50 minutes  
> 🎯 **Focus:** Entity mapping, repositories, JPQL, @Transactional, N+1 problem

---

## 1. Entity Mapping

The wrong mapping causes silent bugs, LazyInitializationExceptions, and cascade deletes you didn't intend. Here's a production-grade entity covering the patterns you'll be asked about:

### 💻 Code Example — Full Entity
```java
@Entity
@Table(name = "orders",
    indexes = {
        @Index(name = "idx_order_status", columnList = "status"),
        @Index(name = "idx_order_user_id", columnList = "user_id")
    })
@EntityListeners(AuditingEntityListener.class) // auto-fill created/updated fields
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)         // LAZY = load on access
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @OneToMany(mappedBy = "order",             // "mappedBy" = inverse side
               cascade = CascadeType.ALL,      // persist/remove items with order
               orphanRemoval = true,           // delete items removed from list
               fetch = FetchType.LAZY)
    private List<OrderItem> items = new ArrayList<>();

    @Enumerated(EnumType.STRING)               // store as "PENDING" not 0
    @Column(nullable = false)
    private OrderStatus status = OrderStatus.PENDING;

    @Column(precision = 10, scale = 2)
    private BigDecimal totalAmount;

    @CreatedDate
    @Column(updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @Version                                   // optimistic locking
    private Long version;

    // Business methods (rich domain model)
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
        recalculateTotal();
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
        recalculateTotal();
    }
}
```

### ❓ Interview Question
> "What's the difference between `CascadeType.ALL` and `orphanRemoval = true`?"

### ✅ Model Answer
- `CascadeType.ALL`: When you save/delete the `Order`, cascade the operation to all `OrderItem`s — they get saved/deleted too
- `orphanRemoval = true`: When an `OrderItem` is **removed from the `items` collection** (not necessarily deleting the Order), it's automatically deleted from the database

They work together: `CASCADE.PERSIST` saves items with order, `orphanRemoval` deletes items when removed from the collection. Without `orphanRemoval`, you'd need to manually call `orderItemRepository.delete(item)`.

---

## 2. Relationships — Bidirectional Mapping

```java
// The OWNING side has the FK column (@JoinColumn)
// The INVERSE side has mappedBy pointing to the owning field

// WRONG — two unrelated @OneToMany/@ManyToOne
// CORRECT — one bidirectional pair

@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    // Helper methods to keep both sides in sync
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this); // set the owning side
    }
    
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null); // clear the owning side
    }
}

@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id") // FK column — OWNING SIDE
    private Order order;
}
```

### ⚠️ Gotcha — equals/hashCode in @Entity
```java
// DON'T use @Data from Lombok on entities — it generates equals/hashCode using all fields
// including the id which is null before persistence

// DO: override equals/hashCode based only on business key (email, code, etc.)
// OR: use id only but handle null case for new entities
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String email;  // business key
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User user)) return false;
        return Objects.equals(email, user.email);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(email);
    }
}
```

---

## 3. Repository Pattern

```java
// Spring Data JPA auto-implements based on method names
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // Method name query (auto-generated)
    List<Order> findByStatus(OrderStatus status);
    List<Order> findByUserIdAndStatusOrderByCreatedAtDesc(Long userId, OrderStatus status);
    Optional<Order> findTopByUserIdOrderByCreatedAtDesc(Long userId);
    
    // @Query — explicit JPQL (entity-oriented, not SQL)
    @Query("SELECT o FROM Order o WHERE o.user.id = :userId AND o.totalAmount > :minAmount")
    List<Order> findHighValueOrders(@Param("userId") Long userId,
                                   @Param("minAmount") BigDecimal minAmount);
    
    // JOIN FETCH — solve N+1!
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.user.id = :userId")
    List<Order> findWithItemsByUser(@Param("userId") Long userId);
    
    // Pagination
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
    
    // @Modifying + @Transactional for update/delete queries
    @Modifying
    @Transactional
    @Query("UPDATE Order o SET o.status = :status WHERE o.id IN :ids")
    int bulkUpdateStatus(@Param("ids") List<Long> ids, @Param("status") OrderStatus status);
    
    // Native SQL query
    @Query(value = "SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days'",
           nativeQuery = true)
    List<Order> findRecentOrders();
    
    // Projection — only select needed fields
    @Query("SELECT new com.example.dto.OrderSummary(o.id, o.status, o.totalAmount) FROM Order o WHERE o.user.id = :userId")
    List<OrderSummary> findSummariesByUser(@Param("userId") Long userId);
    
    // Exists
    boolean existsByUserIdAndStatus(Long userId, OrderStatus status);
    
    // Count
    long countByStatus(OrderStatus status);
}
```

---

## 4. @Transactional — Deep Dive

### 🔑 Key Attributes
```java
@Transactional(
    propagation = Propagation.REQUIRED,      // default: join existing or create new
    isolation = Isolation.READ_COMMITTED,    // default in most DBs
    readOnly = false,                        // true for read-only service methods
    timeout = 30,                            // seconds until rollback
    rollbackFor = {BusinessException.class}, // rollback for checked exceptions too
    noRollbackFor = {OptimisticLockException.class}
)
public void processOrder(Long orderId) { ... }
```

### 🔑 Propagation Behaviours (Most Asked)
| Propagation | Behaviour |
|-------------|-----------|
| `REQUIRED` | Join existing transaction OR create new (default) |
| `REQUIRES_NEW` | **Always** create new transaction, suspend existing |
| `NESTED` | Create nested transaction within existing (savepoint) |
| `SUPPORTS` | Use existing if present, non-transactional otherwise |
| `NOT_SUPPORTED` | Run non-transactionally, suspend existing |
| `MANDATORY` | Must have existing transaction, else exception |
| `NEVER` | Must NOT have existing transaction, else exception |

### 💻 Code Example — REQUIRES_NEW use case
```java
@Service
public class AuditService {
    
    // REQUIRES_NEW: audit log must be saved even if the main transaction rolls back
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAuditEvent(String event, String userId) {
        auditRepository.save(new AuditLog(event, userId, Instant.now()));
        // goes to DB even if OrderService's Transaction rolls back
    }
}

@Service
public class OrderService {
    private final AuditService auditService;
    
    @Transactional
    public void createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        auditService.logAuditEvent("ORDER_CREATED", request.userId()); // new TX
        
        if (someValidationFails()) {
            throw new BusinessException(); // rolls back THIS transaction
            // but AuditLog is ALREADY committed (in its own REQUIRES_NEW TX)
        }
    }
}
```

### ❓ Interview Question
> "When does `@Transactional` rollback? Does it rollback for checked exceptions?"

### ✅ Model Answer
By default, `@Transactional` rolls back for **unchecked exceptions** (`RuntimeException` and `Error`) only. Checked exceptions do **NOT** trigger rollback by default. To rollback for a checked exception:
```java
@Transactional(rollbackFor = IOException.class)
```

Also: transactions only work if the method is called **through a Spring proxy** (external call). If `ServiceA.methodA()` calls `this.methodB()` which has `@Transactional`, the transaction is NOT applied because the call bypasses the proxy.

---

## 5. The N+1 Query Problem

> You have 100 orders. Loading them fires 1 query. Printing each order's user name fires 100 more. That's N+1.

```java
// You have 100 orders. You want to print each order with its user name.
List<Order> orders = orderRepository.findAll(); // 1 query: SELECT * FROM orders

for (Order order : orders) {
    // Each call to order.getUser() triggers a separate query:
    // SELECT * FROM users WHERE id = ?
    System.out.println(order.getUser().getName()); // 100 more queries!
}
// Total: 1 + 100 = 101 queries → N+1 problem
```

### ✅ Solutions

**Solution 1: JOIN FETCH (JPQL)**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findWithUser(@Param("status") OrderStatus status);
// 1 query with JOIN → no lazy loading needed
```

**Solution 2: @EntityGraph**
```java
@EntityGraph(attributePaths = {"user", "items"})
List<Order> findByStatus(OrderStatus status);
// Spring generates JOIN FETCH automatically based on attribute paths
// Can also name and reuse graphs:
@NamedEntityGraph(name = "Order.withItems",
    attributeNodes = @NamedAttributeNode("items"))
@Entity
public class Order { ... }
```

**Solution 3: Batch size (Hibernate)**
```yaml
# application.yml — load N related entities in batches
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 25
# Instead of N queries, Hibernate uses WHERE id IN (1,2,3,...,25)
# Reduces 100 queries to ~4 batches
```

**Solution 4: DTO projection (best for read-only)**
```java
// Interface projection — Spring generates optimized query
public interface OrderSummary {
    Long getId();
    String getStatus();
    BigDecimal getTotalAmount();
    String getUserEmail(); // navigates through: order.user.email
}

List<OrderSummary> findByStatus(OrderStatus status);
// Spring generates: SELECT o.id, o.status, o.total_amount, u.email
//                   FROM orders o JOIN users u ON o.user_id = u.id
//                   WHERE o.status = ?
// No entity loaded, no N+1 possible
```

### ❓ Interview Question
> "How would you detect and fix the N+1 problem in a Spring Boot application?"

### ✅ Model Answer
**Detect:**
1. Enable SQL logging: `spring.jpa.show-sql=true` + formatted
2. Use Hibernate statistics: `spring.jpa.properties.hibernate.generate_statistics=true`
3. Use P6Spy or datasource-proxy to count queries per request
4. Detect in tests: `@DataJpaTest` + assert query count

**Fix (in order of preference):**
1. `JOIN FETCH` in JPQL or `@EntityGraph` for specific queries
2. DTO projections for read-only endpoints
3. `@BatchSize` for tolerable N/batch-size queries
4. Never use `FetchType.EAGER` on collections — it causes N+1 everywhere the entity is loaded

---

## 6. Optimistic vs Pessimistic Locking

### 💻 Optimistic Locking (@Version)
```java
@Entity
public class Inventory {
    @Id private Long id;
    private int quantity;
    
    @Version    // auto-incremented on update, checked on save
    private Long version;
}

// When two transactions try to update simultaneously:
// TX1: UPDATE inventory SET quantity=9, version=2 WHERE id=1 AND version=1  ← succeeds
// TX2: UPDATE inventory SET quantity=9, version=2 WHERE id=1 AND version=1  ← 0 rows affected
//      Hibernate throws OptimisticLockException → retry or error
```

### 💻 Pessimistic Locking (SELECT FOR UPDATE)
```java
@Repository
public interface InventoryRepository extends JpaRepository<Inventory, Long> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)  // SELECT ... FOR UPDATE
    @Query("SELECT i FROM Inventory i WHERE i.id = :id")
    Optional<Inventory> findByIdForUpdate(@Param("id") Long id);
}

// Use for high-contention scenarios where optimistic retry is too expensive
// e.g., ticket booking, payment processing
@Transactional
public void reserveTicket(Long eventId, int quantity) {
    Inventory inv = inventoryRepository.findByIdForUpdate(eventId)
        .orElseThrow(NotFoundException::new);
    
    if (inv.getQuantity() < quantity) throw new InsufficientInventoryException();
    inv.setQuantity(inv.getQuantity() - quantity);
    inventoryRepository.save(inv);
}
```

---

*Next: [Week 4 — Advanced JPA & Database](../week-4/10-database-jpa-advanced.md)*



