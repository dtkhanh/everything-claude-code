# Week 4 — Secure Coding for Healthcare Backend

> 📖 **Estimated reading time:** 50 minutes  
> 🎯 **Focus:** OWASP Top 10, Spring Security hardening, PHI/HIPAA basics, input validation  
> ⚠️ **Why this matters:** AdvaHealth is a healthcare SaaS — security is non-negotiable and will be tested

---

## 1. OWASP Top 10 (2021) — Java Spring Boot Lens

---

### A01: Broken Access Control

```java
// ❌ BAD — anyone can access any patient record
@GetMapping("/patients/{id}")
public Patient getPatient(@PathVariable Long id) {
    return patientRepository.findById(id).orElseThrow();
}

// ✅ GOOD — verify caller owns the resource
@GetMapping("/patients/{id}")
@PreAuthorize("@patientAuthService.canAccess(authentication, #id)")
public Patient getPatient(@PathVariable Long id) {
    return patientRepository.findById(id).orElseThrow();
}
```

**Key Points:**
- Always check authorization server-side — never trust client claims
- Principle of least privilege: give minimum permissions needed
- IDOR (Insecure Direct Object Reference) — the #1 healthcare API bug

**Interview Q:** *How do you prevent a user from accessing another user's patient records?*  
**📢 Headline:** "I use Spring Security's method-level security with `@PreAuthorize` to verify the authenticated principal owns the requested resource, preventing IDOR vulnerabilities."

---

### A03: Injection (SQL, LDAP, NoSQL)

```java
// ❌ BAD — SQL injection via string concatenation
String query = "SELECT * FROM patients WHERE name = '" + name + "'";
em.createNativeQuery(query).getResultList();

// ✅ GOOD — parameterized JPA query
@Query("SELECT p FROM Patient p WHERE p.name = :name")
List<Patient> findByName(@Param("name") String name);

// ✅ GOOD — Spring Data method name
List<Patient> findByNameContainingIgnoreCase(String name);
```

**Gotchas:**
- `@Query` with `nativeQuery = true` is still safe — parameters are bound, not concatenated
- `entityManager.createQuery(jpql)` — JPQL is also injection-safe when using `setParameter`
- String interpolation in JPQL: `"WHERE p.name = '" + name + "'"` — **still vulnerable even in JPQL**

---

### A07: Authentication Failures

```java
// ❌ BAD — JWT with no expiry check, no algorithm validation
Claims claims = Jwts.parser()
    .setSigningKey(secret)
    .parseClaimsJws(token)
    .getBody(); // doesn't check expiry, accepts any alg

// ✅ GOOD — explicit algorithm, expiry validation
Claims claims = Jwts.parserBuilder()
    .setSigningKey(Keys.hmacShaKeyFor(secret.getBytes()))
    .requireExpiration()
    .build()
    .parseClaimsJws(token)
    .getBody();
```

**Healthcare-specific:** Never store JWT tokens in localStorage for medical apps — use HttpOnly cookies.

---

## 2. Input Validation — Spring Boot Best Practices

---

### Bean Validation with `@Valid`

```java
// DTO with constraints
public record CreatePatientRequest(
    @NotBlank(message = "Patient name is required")
    @Size(max = 100, message = "Name must not exceed 100 characters")
    String name,

    @NotNull(message = "Date of birth is required")
    @Past(message = "Date of birth must be in the past")
    LocalDate dateOfBirth,

    @Email(message = "Invalid email format")
    @NotBlank
    String email,

    @Pattern(regexp = "^\\+?[1-9]\\d{1,14}$", message = "Invalid phone number")
    String phoneNumber
) {}

// Controller — trigger validation with @Valid
@PostMapping("/patients")
@ResponseStatus(HttpStatus.CREATED)
public PatientResponse createPatient(@Valid @RequestBody CreatePatientRequest request) {
    return patientService.create(request);
}
```

### Custom Validator Example

```java
@Documented
@Constraint(validatedBy = MRNValidator.class)
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidMRN {
    String message() default "Invalid Medical Record Number format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class MRNValidator implements ConstraintValidator<ValidMRN, String> {
    private static final Pattern MRN_PATTERN = Pattern.compile("^MRN-\\d{8}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true; // use @NotNull separately
        return MRN_PATTERN.matcher(value).matches();
    }
}
```

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationErrors(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());

        return new ErrorResponse("VALIDATION_FAILED", errors);
    }

    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex) {
        // ✅ Don't leak internals — generic message only
        return new ErrorResponse("ACCESS_DENIED", "You do not have permission to perform this action");
    }
}
```

**Key Point:** Never return stack traces in production responses — scrub sensitive internals.

---

## 3. Spring Security Hardening Checklist

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            // ✅ Disable CSRF for stateless JWT APIs
            .csrf(csrf -> csrf.disable())

            // ✅ Stateless session — no server-side session
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // ✅ HSTS — enforce HTTPS
            .headers(headers -> headers
                .httpStrictTransportSecurity(hsts ->
                    hsts.includeSubDomains(true).maxAgeInSeconds(31536000))
                // ✅ Prevent clickjacking
                .frameOptions(frame -> frame.deny()))

            // ✅ CORS — explicit whitelist, not "*"
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // ✅ Authorization rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN") // ⚠️ protect actuator!
                .anyRequest().authenticated())

            // ✅ JWT filter before UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)

            .build();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.advahealth.com")); // ✅ explicit, not "*"
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

**Common Actuator Mistake:**  
```yaml
# ❌ BAD — exposes /actuator/env, /actuator/heapdump publicly
management:
  endpoints:
    web:
      exposure:
        include: "*"

# ✅ GOOD — only expose health and info
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: when-authorized
```

---

## 4. Rate Limiting with Bucket4j

```java
// Maven dependency
// io.github.bucket4j:bucket4j-core:8.x
// io.github.bucket4j:bucket4j-redis:8.x (for distributed rate limiting)

@Component
public class RateLimitingFilter extends OncePerRequestFilter {

    private final Cache<String, Bucket> buckets = Caffeine.newBuilder()
        .expireAfterWrite(1, TimeUnit.HOURS)
        .maximumSize(10_000)
        .build();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String clientId = extractClientId(request); // IP or API key
        Bucket bucket = buckets.get(clientId, this::createBucket);

        if (bucket.tryConsume(1)) {
            chain.doFilter(request, response);
        } else {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setHeader("X-Rate-Limit-Retry-After-Seconds",
                String.valueOf(bucket.getAvailableTokens()));
            response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
        }
    }

    private Bucket createBucket(String key) {
        // 100 requests per minute
        return Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build();
    }
}
```

---

## 5. PHI/HIPAA Logging — Healthcare-Specific

```java
// ❌ BAD — logs PHI (Protected Health Information)
log.info("Processing patient: name={}, ssn={}, diagnosis={}", 
    patient.getName(), patient.getSsn(), patient.getDiagnosis());

// ✅ GOOD — log only non-PHI identifiers
log.info("Processing patient request: patientId={}, requestId={}", 
    patient.getId(), requestId);

// ✅ GOOD — audit log for access (separate from application log)
auditLogger.logAccess(AuditEvent.builder()
    .userId(currentUserId)
    .action("VIEW_PATIENT_RECORD")
    .resourceId(patientId)
    .timestamp(Instant.now())
    .ipAddress(request.getRemoteAddr())
    .build());
```

**HIPAA Audit Requirements:**
- Log WHO accessed WHAT patient data WHEN from WHERE
- Audit logs must be tamper-evident and retained for 6 years
- Separate audit logs from application logs

---

## 6. Password Storage

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // ✅ BCrypt — adaptive hashing, built-in salt
    return new BCryptPasswordEncoder(12); // strength 12 = ~250ms per hash
    // Higher strength = more secure but slower — 10-12 is standard
}

// ❌ Never do these:
// MD5 hashing: MessageDigest.getInstance("MD5")       — broken
// SHA-1: MessageDigest.getInstance("SHA-1")           — weak
// Plain text: user.setPassword(password)              — criminal
// Custom salt without BCrypt/Argon2                   — risky
```

---

## 7. Secrets Management

```yaml
# ❌ BAD — hardcoded in application.properties
spring.datasource.password=SuperSecret123
jwt.secret=my-secret-key

# ✅ GOOD — environment variables
spring.datasource.password=${DB_PASSWORD}
jwt.secret=${JWT_SECRET}
```

```java
// ✅ Validate required secrets at startup — fail fast
@Component
public class SecretsValidator implements ApplicationListener<ApplicationReadyEvent> {
    
    @Value("${jwt.secret:}")
    private String jwtSecret;
    
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        if (jwtSecret == null || jwtSecret.isBlank()) {
            throw new IllegalStateException("JWT_SECRET environment variable is not set. Application cannot start.");
        }
        if (jwtSecret.length() < 32) {
            throw new IllegalStateException("JWT_SECRET must be at least 32 characters long.");
        }
    }
}
```

---

## Interview Q&A — Secure Coding

**Q: "How do you secure a healthcare REST API?"**  
**📢 Headline:** "I apply defense in depth: JWT authentication, method-level authorization, input validation, parameterized queries, rate limiting, and HIPAA-compliant logging that never captures PHI."

**Full Answer:**
1. **Auth layer** — JWT with short expiry (15 min) + refresh token rotation
2. **Authorization** — `@PreAuthorize` checks on every endpoint, resource ownership validation
3. **Input validation** — `@Valid` + custom validators at controller layer before any business logic
4. **Database layer** — JPA parameterized queries, never string concatenation
5. **Transport** — HTTPS only, HSTS enabled, certificate pinning for mobile clients
6. **Rate limiting** — Bucket4j, per-user and per-IP limits
7. **Logging** — structured audit logs, no PHI in application logs
8. **Secrets** — env vars only, validated at startup, rotated regularly

**Q: "What is IDOR and how do you prevent it?"**  
**📢 Headline:** "IDOR is Insecure Direct Object Reference — when API endpoint accepts an ID and doesn't verify the caller owns that resource. I prevent it with server-side ownership checks using Spring Security method expressions."

**Q: "How do you handle sensitive data in logs?"**  
**📢 Headline:** "In healthcare, PHI must never appear in logs. I use structured logging with whitelisted fields, mask or exclude sensitive fields, and route access logs to a separate tamper-evident audit trail."

---

## Gotchas (What trips developers up)

- ❌ Protecting actuator endpoints — `/actuator/heapdump` exposes full heap, never expose publicly
- ❌ CORS `allowedOrigins("*")` + `allowCredentials(true)` — Spring will throw error, this is intentional
- ❌ Logging request bodies indiscriminately — may contain PHI
- ❌ `@Transactional` methods with authorization checks — check auth BEFORE entering transaction
- ❌ JWT `alg: none` attack — always specify exact algorithm, never accept `none`
- ❌ Returning 403 vs 404 for unauthorized resources — prefer 404 to not leak resource existence
