# Kafka Consumer Của Bạn Đang Xử Lý Message Hai Lần — Và Bạn Không Biết

> **Đọc trong:** 16 phút  
> **Áp dụng cho:** Spring Boot + Spring Kafka, Apache Kafka  
> **Tags:** `kafka` `spring-boot` `message-queue` `concurrency` `idempotency`

---

Một payment service nhận message từ Kafka để xử lý giao dịch. Mọi thứ hoạt động tốt trong 3 tháng.

Rồi một buổi chiều, database team báo: một số giao dịch bị duplicate — cùng một transaction ID xuất hiện hai lần trong bảng `payments`. Không phải nhiều, khoảng 0.3% tổng số giao dịch. Nhưng đủ để gây ra vấn đề reconciliation nghiêm trọng.

Không có exception nào trong log. Consumer health check xanh. Kafka lag monitor bình thường.

Sau 2 ngày debug, nguyên nhân: **consumer rebalance xảy ra đúng lúc message đang được xử lý, offset chưa commit, message được consume lại từ đầu**.

Đây là loại bug không lộ trong test, không lộ trong staging với tải thấp, và chỉ xuất hiện đủ thường trong production khi scale lên. Bài này là 4 patterns tôi đã thấy phá production Kafka consumers — kèm cách fix cụ thể.

---

## Nền tảng cần biết trước

Kafka không push message về consumer. Consumer **poll** từ broker, xử lý, rồi **commit offset** để báo "tôi đã xử lý đến đây".

```
Broker: [msg-1][msg-2][msg-3][msg-4][msg-5]
                              ↑
                         committed offset = 3
                         → restart: consumer sẽ đọc lại từ msg-4
```

Nếu consumer chết trước khi commit, message được xử lý lại. Đây là **at-least-once delivery** — Kafka đảm bảo message sẽ đến, nhưng không đảm bảo đến đúng một lần.

Spring Kafka mặc định commit offset **sau khi listener method return**. Tưởng an toàn — nhưng có 4 cái bẫy.

---

## Bug 1: Poison message gây infinite retry loop

```java
@KafkaListener(topics = "order-events")
public void handleOrderEvent(OrderEvent event) {
    Order order = orderRepository.findById(event.getOrderId())
        .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));
    // OrderNotFoundException → Spring Kafka retry → cùng message → exception lại
    // Loop mãi mãi
}
```

Một message "độc" — malformed JSON, missing foreign key, corrupted data — khiến consumer throw exception liên tục. Spring Kafka mặc định **không có retry limit**. Consumer bị stuck tại message đó, Kafka lag tăng dần, service sau đó queue up.

Không có alert nào fire vì consumer vẫn "healthy". Lag alert thì cần phải được config trước.

```java
// application.yml — cấu hình retry có limit và dead letter topic
spring:
  kafka:
    listener:
      # Số lần retry trước khi gửi sang dead letter topic
      # Default: vô hạn (BIG problem)
      ack-mode: record

@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    // Retry tối đa 3 lần với backoff
    FixedBackOff backOff = new FixedBackOff(1000L, 3L); // 1s delay, 3 retries

    // Sau 3 lần fail → gửi sang dead-letter topic tự động
    DeadLetterPublishingRecoverer recoverer =
        new DeadLetterPublishingRecoverer(template,
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);

    // Một số exception không nên retry (data errors, not transient)
    handler.addNotRetryableExceptions(
        JsonParseException.class,
        ValidationException.class
    );

    return handler;
}

@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
        ConsumerFactory<String, Object> consumerFactory,
        DefaultErrorHandler errorHandler) {
    ConcurrentKafkaListenerContainerFactory<String, Object> factory =
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    factory.setCommonErrorHandler(errorHandler);
    return factory;
}
```

Consumer cho dead-letter topic — để triage và reprocess thủ công:

```java
@KafkaListener(topics = "order-events.DLT")
public void handleDeadLetter(
        @Payload OrderEvent event,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
        @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage) {

    log.error("Dead letter received | topic={} | reason={} | event={}",
        topic, errorMessage, event);

    // Alert on-call, save to DB for manual reprocessing
    deadLetterRepository.save(new DeadLetterEvent(event, errorMessage));
    alertService.notifyOnCall("Poison message in " + topic);
}
```

---

## Bug 2: Duplicate processing khi rebalance — bug của tôi ở đầu bài

Consumer group rebalance xảy ra khi: consumer mới join, consumer bị coi là "dead" (heartbeat timeout), partition reassignment.

```java
@KafkaListener(topics = "payment-events", concurrency = "3")
public void processPayment(PaymentEvent event) {
    // Bước 1: xử lý payment — tốn ~500ms
    Payment payment = paymentProcessor.charge(event); // external API call

    // Bước 2: save vào DB
    paymentRepository.save(payment);

    // Offset tự động commit sau khi method return
    // Nhưng nếu rebalance xảy ra trong 500ms đó...
}
```

Nếu rebalance xảy ra sau bước 1 nhưng trước khi offset commit — partition được reassign cho consumer khác, consumer khác đọc lại cùng message và charge lại.

Có hai hướng fix:

**Hướng 1 — Idempotency key (recommended)**

```java
@KafkaListener(topics = "payment-events")
public void processPayment(PaymentEvent event) {
    // Check trước, process sau
    if (paymentRepository.existsByIdempotencyKey(event.getIdempotencyKey())) {
        log.info("Duplicate message skipped | key={}", event.getIdempotencyKey());
        return; // offset sẽ vẫn được commit — message bị skip an toàn
    }

    Payment payment = paymentProcessor.charge(event);
    paymentRepository.save(payment); // idempotencyKey là unique constraint trong DB
}
```

```sql
-- Schema cần có
ALTER TABLE payments ADD COLUMN idempotency_key VARCHAR(255) UNIQUE;
CREATE UNIQUE INDEX idx_payments_idempotency ON payments(idempotency_key);
```

**Hướng 2 — Manual offset commit với acknowledgment**

```java
@KafkaListener(topics = "payment-events")
public void processPayment(PaymentEvent event, Acknowledgment ack) {
    try {
        Payment payment = paymentProcessor.charge(event);
        paymentRepository.save(payment);
        ack.acknowledge(); // commit offset chỉ sau khi cả 2 bước thành công
    } catch (Exception e) {
        // Không acknowledge → message sẽ được xử lý lại sau rebalance
        log.error("Payment processing failed, message will be retried", e);
    }
}
```

```yaml
spring:
  kafka:
    listener:
      ack-mode: manual_immediate # bắt buộc khi dùng Acknowledgment
```

Idempotency key là giải pháp bền vững hơn vì nó xử lý được cả duplicate từ producer side, không chỉ rebalance.

---

## Bug 3: Offset commit quá sớm — mất message khi consumer crash

Ngược lại với bug 2: commit offset trước khi xử lý xong.

```java
// application.yml — cấu hình mặc định của nhiều tutorial online
spring:
  kafka:
    consumer:
      enable-auto-commit: true      # ← vấn đề
      auto-commit-interval-ms: 5000 # commit mỗi 5 giây
```

Với `enable-auto-commit: true`, Kafka client commit offset định kỳ **bất kể message đã được xử lý xong chưa**. Consumer poll message lúc T=0, auto-commit lúc T=5s, consumer crash lúc T=3s khi đang xử lý:

```
T=0:  Poll 100 messages
T=1:  Xử lý xong 40 messages
T=3:  Consumer crash (server restart, OOM, etc.)
T=5:  Auto-commit KHÔNG xảy ra vì consumer đã dead

→ Restart: consume lại từ message 41 ✓ (behavior đúng trong trường hợp này)
```

Thực ra `enable-auto-commit` không tệ khi consumer crash trước commit interval. Nhưng nếu commit xảy ra trước crash:

```
T=0:  Poll 100 messages
T=5:  Auto-commit — báo "đã xử lý 100 messages"
T=6:  Consumer crash (vẫn đang xử lý message 50-100)
T=7:  Restart: consume từ message 101
→ Messages 50-100: MẤT
```

```yaml
# Fix — tắt auto-commit, để Spring Kafka quản lý
spring:
  kafka:
    consumer:
      enable-auto-commit: false
    listener:
      ack-mode: batch # commit sau mỗi batch được xử lý hoàn toàn
```

```java
// Với batch ack mode, Spring tự commit sau khi batch listener return
@KafkaListener(topics = "events")
public void processBatch(List<ConsumerRecord<String, Event>> records) {
    for (ConsumerRecord<String, Event> record : records) {
        processRecord(record.value());
    }
    // Spring commit offset cho toàn bộ batch sau khi method return
    // Nếu exception → không commit → batch được retry
}
```

---

## Bug 4: Consumer chậm gây rebalance cascade

```java
@KafkaListener(topics = "data-sync-events", concurrency = "5")
public void syncData(DataSyncEvent event) {
    // Heavy processing — gọi external API, xử lý file lớn
    externalApiClient.syncRecord(event.getData()); // tốn 30 giây
}
```

Kafka coordinator coi consumer là "dead" nếu không nhận được heartbeat trong `session.timeout.ms` (default: 45 giây). Nhưng heartbeat và poll là **cùng một thread** trong Kafka client.

Khi listener đang xử lý 30 giây, poll thread bị block, heartbeat không được gửi. Coordinator kick consumer khỏi group, trigger rebalance, các consumer khác nhận partition. Consumer cũ xử lý xong, cố commit offset — bị từ chối vì đã bị kick.

```java
// Không phải về config Kafka, mà về thiết kế consumer

// Option 1 — Tăng timeout và max poll interval
// application.yml
spring:
  kafka:
    consumer:
      max-poll-records: 10           # giảm số records mỗi poll
      properties:
        max.poll.interval.ms: 300000 # 5 phút — cho heavy processing
        session.timeout.ms: 60000    # 1 phút heartbeat timeout

// Option 2 — Tách processing ra async pool, trả về nhanh
@KafkaListener(topics = "data-sync-events")
public void syncData(DataSyncEvent event) {
    // Không xử lý ở đây — đẩy vào queue nội bộ
    processingQueue.offer(event);
    // Method return ngay → poll thread không bị block → heartbeat normal
}

@Scheduled(fixedDelay = 100)
public void processQueue() {
    DataSyncEvent event = processingQueue.poll();
    if (event != null) {
        externalApiClient.syncRecord(event.getData());
    }
}

// Option 3 — Pause/resume consumer khi đang xử lý heavy task
@KafkaListener(topics = "data-sync-events")
public void syncData(
        ConsumerRecord<String, DataSyncEvent> record,
        @Header(KafkaHeaders.CONSUMER) Consumer<?, ?> consumer) {

    TopicPartition partition = new TopicPartition(record.topic(), record.partition());
    consumer.pause(Collections.singleton(partition)); // tạm dừng nhận message

    try {
        externalApiClient.syncRecord(record.value().getData());
    } finally {
        consumer.resume(Collections.singleton(partition)); // tiếp tục
    }
}
```

Option 3 elegant nhất nhưng cần cẩn thận: pause/resume phải được gọi **từ consumer thread**, không phải từ thread khác.

---

## Checklist trước khi ship Kafka consumer lên production

```
□ Dead letter topic đã được config với retry limit?
□ Idempotency key có trong message contract không?
□ enable-auto-commit = false?
□ ack-mode phù hợp với processing pattern (record/batch/manual)?
□ max.poll.interval.ms lớn hơn max processing time không?
□ Dead letter consumer có alert on-call không?
□ Consumer group ID có unique per environment không? (staging vs prod)
□ Đã test behavior khi consumer restart giữa chừng chưa?
```

---

## Test behavior dưới failure conditions

Cái này thường bị skip nhất. Cách tôi test locally:

```java
@Test
void shouldNotDuplicatePaymentOnConsumerRestart() {
    // Arrange
    String idempotencyKey = UUID.randomUUID().toString();
    PaymentEvent event = new PaymentEvent(idempotencyKey, 100_000L);

    // Act — simulate message được consume 2 lần
    paymentConsumer.processPayment(event);
    paymentConsumer.processPayment(event); // duplicate

    // Assert — chỉ 1 payment trong DB
    long count = paymentRepository.countByIdempotencyKey(idempotencyKey);
    assertThat(count).isEqualTo(1);
}

@Test
void shouldSendPoisonMessageToDLT() {
    // Arrange — message sẽ fail validation
    OrderEvent poisonEvent = new OrderEvent(null, -1L); // invalid data

    // Act — consumer xử lý, fail 3 lần
    for (int i = 0; i <= 3; i++) {
        try { orderConsumer.handleOrderEvent(poisonEvent); }
        catch (Exception ignored) {}
    }

    // Assert — message trong dead letter topic
    verify(deadLetterRepository, times(1)).save(any(DeadLetterEvent.class));
    verify(alertService, times(1)).notifyOnCall(anyString());
}
```

---

Kafka là một trong những infrastructure component dễ dùng sai nhất — bởi vì sai theo kiểu im lặng. Không có exception vào lúc bạn config sai. Không có lỗi compile. Chỉ là 0.3% transactions bị duplicate sau 3 tháng.

Bốn patterns trên đều có thể được bắt bằng test và config đúng trước khi ship. Dead letter topic + idempotency key là hai thứ không thể thiếu cho bất kỳ consumer nào xử lý data quan trọng. Phần còn lại phụ thuộc vào SLA của bạn về message loss vs duplicate.

---

## Đọc thêm

- [Spring Kafka Docs — Error Handling](https://docs.spring.io/spring-kafka/docs/current/reference/html/#annotation-error-handling) — DefaultErrorHandler và DeadLetterPublishingRecoverer
- [Kafka Docs — Consumer Configuration](https://kafka.apache.org/documentation/#consumerconfigs) — max.poll.interval.ms, session.timeout.ms
- [Confluent — Idempotent Kafka Consumers](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/) — exactly-once vs at-least-once
- [Vlad Mihalcea — Idempotent Consumer Pattern](https://vladmihalcea.com/idempotent-consumer/) — database-level idempotency implementation
