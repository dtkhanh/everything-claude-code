# Week 4 — Day 3–4: System Design & Microservices

> 📖 **Estimated reading time:** 45 minutes  
> 🎯 **Focus:** REST best practices, microservice patterns, caching, messaging

---

## 1. REST API Best Practices

Bad API design surfaces in interviews as "what's wrong with this endpoint?" Here are the rules, with the reasoning:

### 🔑 Resource Design
```
# Nouns, not verbs. Plural for collections.
GET    /api/users              → list users (with pagination)
POST   /api/users              → create user
GET    /api/users/{id}         → get user by id
PUT    /api/users/{id}         → full update user
PATCH  /api/users/{id}         → partial update user
DELETE /api/users/{id}         → delete user

# Sub-resources for relationships
GET    /api/users/{id}/orders  → orders for a user
POST   /api/orders/{id}/cancel → action as sub-resource (not /cancelOrder)

# Filtering, sorting, pagination — always on collection endpoints
GET /api/orders?status=ACTIVE&page=0&size=20&sort=createdAt,desc
```

### 🔑 HTTP Status Codes
| Code | Meaning | Use When |
|------|---------|----------|
| 200 | OK | Successful GET, PUT |
| 201 | Created | Successful POST — include Location header |
| 204 | No Content | Successful DELETE, PUT with no body |
| 400 | Bad Request | Validation failure, malformed request |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Authenticated but no permission |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate creation, optimistic lock |
| 422 | Unprocessable | Valid format but business rule violation |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server error |
| 503 | Service Unavailable | Maintenance, upstream dependency down |

### 💻 Code Example — Standard API Response Envelope
```java
public record ApiResponse<T>(
    boolean success,
    T data,
    String message,
    List<ApiError> errors
) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, data, null, null);
    }
    
    public static ApiResponse<?> error(String message, List<ApiError> errors) {
        return new ApiResponse<>(false, null, message, errors);
    }
}

public record PagedResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean last
) {
    public static <T> PagedResponse<T> from(Page<T> page) {
        return new PagedResponse<>(
            page.getContent(), page.getNumber(), page.getSize(),
            page.getTotalElements(), page.getTotalPages(), page.isLast());
    }
}

// Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<?> handleValidation(MethodArgumentNotValidException ex) {
        List<ApiError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new ApiError(fe.getField(), fe.getDefaultMessage()))
            .toList();
        return ApiResponse.error("Validation failed", errors);
    }
    
    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiResponse<?> handleNotFound(EntityNotFoundException ex) {
        return ApiResponse.error(ex.getMessage(), null);
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiResponse<?> handleGeneral(Exception ex) {
        log.error("Unhandled exception", ex);
        return ApiResponse.error("Internal server error", null); // don't leak details
    }
}
```

---

## 2. Microservices Patterns

### 📌 API Gateway Pattern
```
Client → API Gateway
              ├── /users/**   → User Service
              ├── /orders/**  → Order Service
              ├── /products/** → Product Service
              └── Rate limiting, Auth, Circuit breaking

Spring Cloud Gateway config:
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service      # lb:// = load-balanced via Eureka
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Source-Service, api-gateway
```

### 📌 Circuit Breaker Pattern
```java
// Using Resilience4j (Spring Boot default)
@Service
public class OrderService {
    
    @CircuitBreaker(name = "payment-service",
                    fallbackMethod = "paymentFallback")
    @Retry(name = "payment-service")
    @TimeLimiter(name = "payment-service")
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> paymentClient.charge(request));
    }
    
    public CompletableFuture<PaymentResult> paymentFallback(
            PaymentRequest request, Throwable ex) {
        log.warn("Payment service unavailable, using fallback", ex);
        return CompletableFuture.completedFuture(PaymentResult.pending(request.orderId()));
    }
}

# application.yml
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        sliding-window-size: 10
        failure-rate-threshold: 50      # 50% failures → OPEN
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
  retry:
    instances:
      payment-service:
        max-attempts: 3
        wait-duration: 1s
        exponential-backoff-multiplier: 2
```

### 📌 Saga Pattern — Distributed Transactions
```
Problem: microservices each have their own DB → no distributed ACID transactions

Choreography Saga (event-driven):
OrderService → OrderCreated event
    → InventoryService hears → reserves stock → InventoryReserved
        → PaymentService hears → charges → PaymentProcessed
            → OrderService hears → confirms → OrderConfirmed
    
If PaymentFailed:
    → InventoryService hears → releases stock (compensating transaction)
    → OrderService hears → cancels order

Orchestration Saga (central coordinator):
SagaOrchestrator:
    step1: call InventoryService.reserve()
    step2: call PaymentService.charge()
    step3: call OrderService.confirm()
    on failure at step N: call compensating transactions for steps 1..N-1
```

---

## 3. Caching Strategies

### 🔑 Cache Patterns
```
Cache-Aside (Lazy Load) — Most Common:
    READ: Check cache → if miss → load from DB → store in cache → return
    WRITE: Write to DB → invalidate cache (or update)

Write-Through:
    WRITE: Write to cache → cache writes to DB synchronously
    READ: Always from cache

Write-Behind (Write-Back):
    WRITE: Write to cache → async write to DB
    Risk: data loss if cache crashes before DB write

Read-Through:
    Cache is the only endpoint — cache loads from DB automatically on miss
```

### 💻 Code Example — Redis Cache with Spring
```java
@Configuration
public class RedisConfig {
    
    @Bean
    public RedisCacheConfiguration defaultCacheConfig() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        Map<String, RedisCacheConfiguration> configs = Map.of(
            "products", defaultCacheConfig().entryTtl(Duration.ofHours(1)),
            "users", defaultCacheConfig().entryTtl(Duration.ofMinutes(5)),
            "sessions", defaultCacheConfig().entryTtl(Duration.ofDays(7))
        );
        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultCacheConfig())
            .withInitialCacheConfigurations(configs)
            .build();
    }
}
```

---

## 4. Message Queues — Kafka Patterns

### 💻 Code Example — Spring Kafka
```java
// Producer
@Service
public class OrderEventProducer {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderEvent event = OrderEvent.created(order);
        kafkaTemplate.send("orders", order.getId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish OrderCreated event for order {}", order.getId(), ex);
                    // IMPORTANT: handle failure — dead letter, retry, or alert
                } else {
                    log.info("OrderCreated published: partition={}, offset={}",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}

// Consumer
@Component
public class InventoryConsumer {
    
    @KafkaListener(
        topics = "orders",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderCreated(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            Acknowledgment ack) {
        try {
            inventoryService.reserve(event.getOrderId(), event.getItems());
            ack.acknowledge(); // manual commit after successful processing
        } catch (InsufficientStockException e) {
            // Publish compensating event
            eventProducer.publishStockInsufficient(event.getOrderId());
            ack.acknowledge(); // still ack — don't retry forever
        } catch (Exception e) {
            // Don't ack — will be redelivered
            // After max retries, goes to DLT (Dead Letter Topic)
            log.error("Failed to process order event, will retry", e);
        }
    }
}
```

### ❓ Interview Question
> "What's the difference between a message queue and event streaming?"

### ✅ Model Answer
**Message Queue (RabbitMQ, SQS):**
- Message delivered to **one consumer** (competing consumers pattern)
- Message deleted after consumption
- Good for: task distribution, work queues, RPC
- Pull model typically

**Event Streaming (Kafka):**
- Event delivered to **all consumer groups** (each group gets all events)
- Events **retained** for configurable period (days/weeks) — consumers can replay
- Good for: event sourcing, audit trail, fan-out, stream processing
- Consumers track offsets independently
- Kafka can handle millions of events/second

---

## 5. Key Architecture Questions

### ❓ "How would you scale a high-traffic API?"

**Answer framework:**
1. **Horizontal scaling**: Multiple instances behind a load balancer (stateless design essential)
2. **Caching**: Redis for frequently-read data, CDN for static assets
3. **Database read replicas**: Route read queries to replicas
4. **Async processing**: Move slow operations to message queue (Kafka/SQS)
5. **Rate limiting**: Protect endpoints from abuse (Bucket4j, Redis + Lua script)
6. **Connection pool tuning**: Ensure pool size matches expected concurrency
7. **Database indexing**: EXPLAIN ANALYZE on slow queries
8. **Circuit breakers**: Prevent cascade failures from dependencies

### ❓ "Explain CAP theorem."

**Answer**: In a distributed system, you can only guarantee **2 of 3**:
- **C**onsistency: every read gets the most recent write
- **A**vailability: every request gets a response (not necessarily latest data)
- **P**artition tolerance: system works despite network partitions

**CP systems** (choose consistency over availability): HBase, ZooKeeper — returns error when can't guarantee consistency
**AP systems** (choose availability over consistency): Cassandra, DynamoDB — eventually consistent, returns potentially stale data

Real-world: Partition tolerance is non-negotiable in distributed systems. The real choice is **CP vs AP** — strong consistency vs availability + eventual consistency.

---

*Next: [12-interview-qna-master.md](./12-interview-qna-master.md)*


