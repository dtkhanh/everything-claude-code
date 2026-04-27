# Spring Boot Cheatsheet — Quick Reference

> Annotations, config properties, and patterns for rapid review

---

## Core Annotations

### Stereotype Annotations
```java
@Component         // generic Spring bean
@Service           // business logic (semantic only)
@Repository        // data access + exception translation
@Controller        // MVC (returns views)
@RestController    // REST API (@Controller + @ResponseBody)
@Configuration     // Java config, contains @Bean methods
@Aspect            // AOP aspect
```

### Bean & DI
```java
@Bean              // method in @Configuration → registers return as bean
@Primary           // prefer this bean when multiple match
@Qualifier("name") // specify exact bean by name
@Lazy              // don't create until first requested
@Scope("prototype") // new instance per injection (default: singleton)
@Conditional*      // conditional bean creation
@Profile("dev")    // only active in specified profile
```

### Injection
```java
@Autowired         // inject by type (skip for constructor injection in single-constructor)
@Value("${prop}")  // inject property value
@Value("#{bean.method()}") // SpEL expression
@ConfigurationProperties(prefix = "app") // map properties to class
```

### Web
```java
@RequestMapping("/api")  // base path on class
@GetMapping("/users")    // GET
@PostMapping("/users")   // POST
@PutMapping("/users/{id}") // PUT
@DeleteMapping("/users/{id}") // DELETE
@PatchMapping("/users/{id}")  // PATCH

@PathVariable("id") Long id   // from URL
@RequestParam("page") int page // from query string
@RequestBody UserRequest req   // from request body (JSON)
@RequestHeader("Authorization") String auth // from header
@ResponseStatus(HttpStatus.CREATED) // set response status code
@Valid             // trigger Bean Validation on @RequestBody
```

### Transaction
```java
@Transactional                                    // default REQUIRED, read-write
@Transactional(readOnly = true)                   // read-only optimization
@Transactional(propagation = Propagation.REQUIRES_NEW)
@Transactional(rollbackFor = Exception.class)     // rollback for checked too
@Transactional(timeout = 30)                      // seconds
```

### JPA / Hibernate
```java
@Entity            // JPA entity
@Table(name = "t") // custom table name
@Id                // primary key
@GeneratedValue(strategy = GenerationType.IDENTITY) // auto-increment
@Column(name = "col", nullable = false, unique = true)
@Enumerated(EnumType.STRING)  // store enum as string
@Lob               // large object (CLOB/BLOB)
@Transient         // not persisted
@Version           // optimistic locking counter

// Relationships
@OneToMany(mappedBy="field", cascade=CascadeType.ALL, fetch=FetchType.LAZY)
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id")
@ManyToMany
@JoinTable(name="user_roles", joinColumns=..., inverseJoinColumns=...)

// Auditing
@EntityListeners(AuditingEntityListener.class)
@CreatedDate @LastModifiedDate
@CreatedBy @LastModifiedBy
// Enable: @EnableJpaAuditing on @Configuration
```

### Security
```java
@EnableWebSecurity       // on @Configuration
@EnableMethodSecurity    // enables @PreAuthorize etc.
@PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
@PostAuthorize("returnObject.owner == authentication.name")
@Secured("ROLE_ADMIN")   // simpler role check
```

### Testing
```java
@SpringBootTest          // full application context
@WebMvcTest(Controller.class)    // MVC layer only
@DataJpaTest             // JPA/DB only (H2 in-memory)
@MockBean                // replace bean with Mockito mock
@SpyBean                 // wrap bean with spy
@AutoConfigureMockMvc    // with @SpringBootTest
@Testcontainers          // Docker containers in tests
@Container               // declare container field
```

### Cache
```java
@EnableCaching           // on @Configuration
@Cacheable(value="users", key="#id")     // cache return value
@CachePut(value="users", key="#id")      // always update cache
@CacheEvict(value="users", key="#id")    // remove from cache
@CacheEvict(allEntries = true)           // clear entire cache
```

---

## application.yml — Key Properties

```yaml
server:
  port: 8080
  servlet:
    context-path: /api/v1

spring:
  application:
    name: my-service
  
  # DataSource
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASS}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 3000
      max-lifetime: 1800000
  
  # JPA
  jpa:
    hibernate:
      ddl-auto: validate          # prod: validate. dev: update or create-drop
    show-sql: false               # dev only
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 25
        generate_statistics: false  # enable for N+1 debugging
  
  # Flyway
  flyway:
    enabled: true
    locations: classpath:db/migration

  # Security
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com

  # Profiles
  profiles:
    active: dev

  # Cache (Redis)
  cache:
    type: redis
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms

  # Kafka
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: my-service
      auto-offset-reset: earliest

# Actuator
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized

# Logging
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.hibernate.SQL: DEBUG      # show SQL queries
    org.hibernate.orm.jdbc.bind: TRACE  # show parameters

# Custom app properties
app:
  jwt:
    secret: ${JWT_SECRET}
    expiration-ms: 86400000
```

---

## Repository Naming Conventions

```java
// Spring Data JPA auto-implements from method name

findBy{Field}                          // List<T> or Optional<T>
findBy{Field}And{Field2}
findBy{Field}Or{Field2}
findBy{Field}OrderBy{Field2}Desc
findBy{Field}In(Collection<T>)         // WHERE field IN (...)
findBy{Field}NotIn(Collection<T>)
findBy{Field}IsNull / IsNotNull
findBy{Field}Like("%val%")
findBy{Field}Containing("val")         // shorter: LIKE '%val%'
findBy{Field}StartingWith("prefix")
findBy{Field}EndingWith("suffix")
findBy{Field}GreaterThan(value)
findBy{Field}Between(from, to)
findBy{Field}True / False              // boolean field

countBy{Field}
existsBy{Field}
deleteBy{Field}

findTop3By{Field}OrderBy{Field2}Desc   // LIMIT 3
findFirst10ByStatus(status, Pageable)  // with pagination
```

---

## Common Spring Boot Patterns

### Global Exception Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(EntityNotFoundException ex) {
        return ErrorResponse.of(ex.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage()).toList();
        return ErrorResponse.of("Validation failed", errors);
    }
}
```

### Pagination Response
```java
// Standard paginated endpoint
@GetMapping
public ResponseEntity<PagedResponse<UserDTO>> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy) {
    
    Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy).descending());
    Page<User> userPage = userService.findAll(pageable);
    return ResponseEntity.ok(PagedResponse.from(userPage.map(UserDTO::from)));
}
```

### Service Pattern
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)         // default read-only for all methods
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final ApplicationEventPublisher eventPublisher;
    
    public UserDTO getUser(Long id) {   // inherits readOnly = true
        return userRepository.findById(id)
            .map(UserDTO::from)
            .orElseThrow(() -> new EntityNotFoundException("User not found: " + id));
    }
    
    @Transactional                      // override: need write transaction
    public UserDTO createUser(CreateUserRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateEmailException(request.email());
        }
        User user = new User(request.email(), passwordEncoder.encode(request.password()));
        user = userRepository.save(user);
        eventPublisher.publishEvent(new UserCreatedEvent(user));
        return UserDTO.from(user);
    }
}
```

---

## Bean Propagation Quick Reference

| Propagation | Existing TX | No TX | Use Case |
|-------------|------------|-------|----------|
| `REQUIRED` (default) | JOIN | CREATE | Standard operations |
| `REQUIRES_NEW` | SUSPEND, CREATE NEW | CREATE | Audit log, must commit independently |
| `NESTED` | SAVEPOINT | CREATE | Partial rollback within outer TX |
| `SUPPORTS` | JOIN | Run without TX | Read-only that can be called anywhere |
| `NOT_SUPPORTED` | SUSPEND | Run without TX | Lock-free quick check |
| `MANDATORY` | JOIN | **EXCEPTION** | Must be called within TX |
| `NEVER` | **EXCEPTION** | Run without TX | Ensure not in TX |

---

## Spring Security Common Configs

```java
// Permit public + require auth for rest
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
    .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated()
)

// Get current user anywhere
SecurityContextHolder.getContext().getAuthentication().getName()

// Get custom UserDetails
User user = (User) SecurityContextHolder.getContext()
    .getAuthentication().getPrincipal();

// In controller
@GetMapping("/me")
public UserDTO me(@AuthenticationPrincipal User user) { ... }
```

---

## Testing Quick Reference

```java
// MockMvc
mockMvc.perform(post("/api/users")
    .contentType(MediaType.APPLICATION_JSON)
    .content(objectMapper.writeValueAsString(request))
    .header("Authorization", "Bearer " + token))
    .andExpect(status().isCreated())
    .andExpect(jsonPath("$.data.email").value("alice@example.com"));

// With asserted exception
assertThatThrownBy(() -> userService.createUser(duplicateRequest))
    .isInstanceOf(DuplicateEmailException.class)
    .hasMessageContaining("alice@example.com");

// Argument capture
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
verify(userRepository).save(captor.capture());
assertThat(captor.getValue().getEmail()).isEqualTo("alice@example.com");

// Testcontainers + Flyway
@SpringBootTest
@Testcontainers
class IntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb");
    
    @DynamicPropertySource
    static void datasourceProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

