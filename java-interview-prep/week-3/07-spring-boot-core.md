# Week 3 — Day 1–3: Spring Boot Core

> 📖 **Estimated reading time:** 50 minutes  
> 🎯 **Focus:** Auto-configuration, DI/IoC, Bean lifecycle, AOP, Actuator

---

## 1. How Spring Boot Auto-Configuration Works

Add `spring-boot-starter-data-jpa` to `pom.xml`. No XML, no manual bean setup. Spring Boot creates an `EntityManagerFactory`, a `TransactionManager`, and wires `JpaRepository` support automatically. Here's the exact mechanism:

### 🔑 The Mechanism (Step by Step)
1. `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
2. `@EnableAutoConfiguration` triggers `AutoConfigurationImportSelector`
3. It reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 3) — a list of auto-config classes
4. Each auto-config class is annotated with `@Conditional*` — only applies if condition is met
5. Spring instantiates matching auto-config classes → they register beans

### 💻 Code Example — How a custom auto-config looks
```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)              // only if DataSource is on classpath
@ConditionalOnMissingBean(DataSource.class)        // only if user hasn't defined their own
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnProperty(prefix = "spring.datasource", name = "url")
    public DataSource dataSource(DataSourceProperties properties) {
        return DataSourceBuilder.create()
            .url(properties.getUrl())
            .username(properties.getUsername())
            .password(properties.getPassword())
            .build();
    }
}
```

### 💻 Code Example — @Conditional annotations
```java
@ConditionalOnClass(RedisTemplate.class)          // class exists on classpath
@ConditionalOnMissingClass("org.mongodb.Driver")  // class does NOT exist
@ConditionalOnBean(UserRepository.class)           // bean already defined
@ConditionalOnMissingBean                          // no bean of this type yet
@ConditionalOnProperty("feature.payments.enabled=true") // property set
@ConditionalOnWebApplication                       // running as web app
@ConditionalOnExpression("${server.port} > 8000") // SpEL expression
```

### ❓ Interview Question
> "How does Spring Boot auto-configuration work? How can you override it?"

### ✅ Model Answer
Spring Boot reads auto-configuration classes listed in `spring/…AutoConfiguration.imports`. Each class uses `@Conditional*` annotations to decide whether to apply. To override: define your own `@Bean` of the same type — `@ConditionalOnMissingBean` on the auto-config means it won't create its default if you already have one.

You can also:
- Use `spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration`
- Use `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})`
- Set properties to configure auto-configured beans

---

## 2. Dependency Injection & IoC

### 📌 Concept Summary
**Inversion of Control**: the framework creates and manages objects (beans), not your code. **Dependency Injection**: Spring injects required dependencies into your beans automatically.

### 🔑 Key Points
- 3 injection types: constructor (✅), setter (OK), field (❌)
- Constructor injection is mandatory for required deps — enables immutability and testability
- `@Autowired` is implicit for single-constructor beans in Spring 4.3+

### 💻 Code Example — Injection types
```java
// ✅ PREFERRED: Constructor injection
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final EventPublisher eventPublisher;

    // @Autowired optional if single constructor (Spring 4.3+)
    public OrderService(OrderRepository orderRepository,
                        PaymentService paymentService,
                        EventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
        this.eventPublisher = eventPublisher;
    }
}

// ❌ AVOID: Field injection — can't be final, hard to test
@Service
public class BadService {
    @Autowired private OrderRepository repo; // ❌ not final, needs reflection to inject in tests
}

// OK for optional deps: setter injection
@Service
public class NotificationService {
    private EmailClient emailClient;

    @Autowired(required = false)
    public void setEmailClient(EmailClient emailClient) {
        this.emailClient = emailClient;
    }
}
```

### ❓ Interview Question
> "Why is constructor injection preferred over field injection?"

### ✅ Model Answer
1. **Immutability**: fields can be `final` — object is in consistent state after construction
2. **Testability**: instantiate directly in unit tests without reflection (`new OrderService(mockRepo, mockPayment, mockEvent)`) vs `ReflectionTestUtils.setField()`
3. **Fail-fast**: missing required dependency causes `BeanCreationException` at startup, not NPE at runtime
4. **Circular dependency visibility**: Spring throws an error immediately for circular dependencies with constructor injection (with field injection it uses proxies and it's harder to detect)

---

## 3. Bean Lifecycle

### 📌 Concept Summary
Spring beans go through a lifecycle: instantiation → dependency injection → initialization → in-use → destruction.

```
1. BeanDefinition loading (classpath scan / @Configuration)
2. BeanFactory creates instance (via constructor)
3. Dependency injection (properties, constructor, setter)
4. BeanNameAware, BeanFactoryAware callbacks
5. BeanPostProcessor.postProcessBeforeInitialization()
6. @PostConstruct → afterPropertiesSet() (InitializingBean) → init-method
7. BeanPostProcessor.postProcessAfterInitialization()
   → AOP proxies created HERE
8. Bean is ready to use
9. Container shutdown:
   @PreDestroy → destroy() (DisposableBean) → destroy-method
```

### 💻 Code Example
```java
@Component
@Slf4j
public class CacheWarmupService {
    private final CacheManager cacheManager;
    private final ProductRepository productRepository;

    public CacheWarmupService(CacheManager cacheManager,
                              ProductRepository productRepository) {
        this.cacheManager = cacheManager;
        this.productRepository = productRepository;
        log.info("Constructor called — dependencies injected");
    }

    @PostConstruct  // called after dependency injection, before bean is ready
    public void warmupCache() {
        log.info("Warming up product cache...");
        productRepository.findTopSellers(100)
            .forEach(p -> cacheManager.getCache("products").put(p.getId(), p));
    }

    @PreDestroy    // called before bean is removed from context
    public void cleanup() {
        log.info("Clearing caches before shutdown");
        cacheManager.getCacheNames().forEach(name -> cacheManager.getCache(name).clear());
    }
}
```

### 🔑 Bean Scopes
```java
@Component
@Scope("singleton")    // DEFAULT — one instance per container
public class UserService { ... }

@Component
@Scope("prototype")    // new instance every time requested
public class ShoppingCart { ... }

// Web scopes (only in web context)
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext { ... } // new per HTTP request

@Component
@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSession { ... }   // new per HTTP session
```

### ❓ Interview Question
> "What's the difference between `@Component`, `@Service`, and `@Repository`?"

### ✅ Model Answer
All three are specializations of `@Component` — at the component scan level, they're identical (all register a bean). The differences are semantic and functional:
- `@Component`: generic Spring-managed component
- `@Service`: marks business logic layer (no extra functionality, improves readability, tools use it)
- `@Repository`: marks data access layer + **enables exception translation** — Spring wraps persistence exceptions (SQLException, JPA exceptions) into Spring's `DataAccessException` hierarchy

---

## 4. AOP — Aspect-Oriented Programming

### 📌 Concept Summary
AOP separates **cross-cutting concerns** (logging, caching, transactions, security) from business logic. Spring implements AOP with **proxies** — your bean is wrapped in a proxy that intercepts method calls.

### 🔑 Key Terms
- **Aspect**: class containing cross-cutting logic (`@Aspect`)
- **Advice**: the code that runs (`@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing`)
- **Pointcut**: expression defining which methods to intercept
- **JoinPoint**: a specific method execution being intercepted
- **Proxy**: the wrapped bean (JDK proxy for interfaces, CGLIB proxy for classes)

### 💻 Code Example — Logging Aspect
```java
@Aspect
@Component
@Slf4j
public class LoggingAspect {

    // Pointcut: all methods in service package
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    // @Around: most powerful — controls execution
    @Around("serviceLayer()")
    public Object logExecution(ProceedingJoinPoint jp) throws Throwable {
        String methodName = jp.getSignature().toShortString();
        long start = System.currentTimeMillis();
        
        log.info("→ Entering {}", methodName);
        try {
            Object result = jp.proceed(); // call the real method
            long elapsed = System.currentTimeMillis() - start;
            log.info("← Exiting {} ({}ms)", methodName, elapsed);
            return result;
        } catch (Exception ex) {
            log.error("✗ Exception in {}: {}", methodName, ex.getMessage());
            throw ex;
        }
    }
    
    // @Before: runs before method
    @Before("@annotation(com.example.annotation.AuditLog)")
    public void auditLog(JoinPoint jp) {
        log.info("AUDIT: {} called with args: {}", 
            jp.getSignature().getName(), Arrays.toString(jp.getArgs()));
    }
}
```

### ❓ Interview Question
> "How does Spring AOP work internally? What are its limitations?"

### ✅ Model Answer
Spring AOP uses **dynamic proxies**. When a bean has aspects, Spring creates:
- **JDK Dynamic Proxy** if the bean implements an interface (proxy implements same interface)
- **CGLIB Proxy** if no interface (subclass is created at runtime)

When you call a method on a Spring bean, you're actually calling the proxy, which applies advice, then delegates to the real object.

**Limitations:**
1. **Self-invocation doesn't work**: `this.method()` bypasses the proxy — AOP advice won't be applied. Fix: inject `self` reference or use `AopContext.currentProxy()`
2. **Only public methods**: CGLIB proxies can't intercept private/final methods
3. **Spring-managed beans only**: calling `new MyService()` gives you a plain object, not a proxy

```java
// Self-invocation bug
@Service
class OrderService {
    @Transactional
    public void createOrder(Order o) {
        // this.validateOrder(o) — bypasses @Transactional proxy! ❌
        validateOrder(o); // same: calls real method without transaction advice
    }
    
    @Transactional(readOnly = true)
    public void validateOrder(Order o) { ... }
}
```

---

## 5. Configuration & Properties

### 💻 Code Example — @ConfigurationProperties
```java
// application.yml
// app:
//   cache:
//     ttl: 300
//     max-size: 1000
//     hosts:
//       - redis1:6379
//       - redis2:6379

@ConfigurationProperties(prefix = "app.cache")
@Validated  // enables Bean Validation on properties
public record CacheProperties(
    @DurationUnit(ChronoUnit.SECONDS) Duration ttl,  // 300 → PT5M Duration
    @Min(1) @Max(10000) int maxSize,
    List<String> hosts
) {}

@Configuration
@EnableConfigurationProperties(CacheProperties.class)
public class CacheConfig {
    @Bean
    public CacheManager cacheManager(CacheProperties props) {
        // use props.ttl(), props.maxSize(), props.hosts()
    }
}
```

### 💻 Code Example — Profiles
```java
// application.yml
// spring:
//   profiles:
//     active: dev

// application-dev.yml, application-prod.yml

@Component
@Profile("dev")
public class DevDataSeeder implements CommandLineRunner {
    public void run(String... args) {
        // seed test data only in dev
    }
}

@Configuration
@Profile("!test")  // active in all profiles except test
public class ExternalApiConfig {
    @Bean
    public PaymentGateway paymentGateway(@Value("${payment.api-key}") String key) {
        return new StripeGateway(key);
    }
}
```

---

## 6. Spring Boot Actuator

### 📌 Concept Summary
Actuator exposes **production-ready endpoints** for monitoring, health checks, metrics, and app management.

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
  endpoint:
    health:
      show-details: when-authorized  # prod: never expose full details publicly
  metrics:
    export:
      prometheus:
        enabled: true
```

### Key Endpoints
| Endpoint | Purpose |
|----------|---------|
| `/actuator/health` | Health status (UP/DOWN) + component details |
| `/actuator/info` | App metadata (version, git commit) |
| `/actuator/metrics` | Micrometer metrics |
| `/actuator/prometheus` | Prometheus scrape endpoint |
| `/actuator/loggers` | View and change log levels at runtime |
| `/actuator/env` | Show environment properties |
| `/actuator/threaddump` | Current thread state |
| `/actuator/heapdump` | Download heap dump |

```java
// Custom health indicator
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    private final DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            return Health.up()
                .withDetail("database", conn.getMetaData().getDatabaseProductName())
                .build();
        } catch (SQLException ex) {
            return Health.down().withException(ex).build();
        }
    }
}
```

---

*Next: [08-spring-security.md](./08-spring-security.md)*


