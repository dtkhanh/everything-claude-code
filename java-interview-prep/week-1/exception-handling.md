# Week 1 — Bonus: Exception Handling Patterns

> 📖 **Estimated reading time:** 30 minutes  
> 🎯 **Focus:** Checked vs Unchecked, custom exceptions, global handler, production patterns  
> ⚠️ **Why it matters:** AdvaHealth JD explicitly requires "backend system design and debugging" — exception design is always tested

---

## 1. Checked vs Unchecked Exceptions

```java
// Checked — extends Exception (not RuntimeException)
// Compiler FORCES you to handle or declare
public class InsufficientFundsException extends Exception {
    public InsufficientFundsException(double amount) {
        super("Insufficient funds: required " + amount);
    }
}

// Unchecked — extends RuntimeException
// No compile-time enforcement — for programming errors and unexpected states
public class PatientNotFoundException extends RuntimeException {
    private final Long patientId;

    public PatientNotFoundException(Long patientId) {
        super("Patient not found: " + patientId);
        this.patientId = patientId;
    }

    public Long getPatientId() { return patientId; }
}
```

| | Checked | Unchecked |
|--|---------|-----------|
| **Inherits from** | `Exception` | `RuntimeException` |
| **Compiler requires** | yes | no |
| **Use for** | Expected failures (file not found, IO error) | Programming bugs, invalid state |
| **In Spring Boot** | Almost never — use unchecked | Always — Spring only rolls back unchecked by default |

### ❓ Interview Question
> "When would you use a checked vs unchecked exception?"

### ✅ Model Answer
In modern Spring Boot: **almost always unchecked**. Checked exceptions force callers to handle every possible failure path, creating verbose try/catch boilerplate that often just wraps and rethrows. Spring's `@Transactional` only rolls back for `RuntimeException` by default, so checked exceptions can silently commit a broken transaction.

Use checked exceptions only when:
- The caller **realistically can recover** from the exception (e.g., user inputs an invalid file → ask them to re-upload)
- You're implementing a library where callers must handle specific failures

### ⚠️ Gotchas
- `@Transactional` does NOT rollback on checked exceptions unless you add `rollbackFor = CheckedException.class`
- Wrapping checked in unchecked without preserving the cause: `throw new RuntimeException(e)` ✅ (preserves stack), `throw new RuntimeException("failed")` ❌ (loses original cause)

---

## 2. Custom Exception Hierarchy — Healthcare Domain

```java
// Base domain exception
public abstract class AdvaHealthException extends RuntimeException {
    private final String errorCode;
    private final HttpStatus httpStatus;

    protected AdvaHealthException(String message, String errorCode, HttpStatus httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    protected AdvaHealthException(String message, String errorCode,
                                   HttpStatus httpStatus, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    public String getErrorCode() { return errorCode; }
    public HttpStatus getHttpStatus() { return httpStatus; }
}

// Domain-specific exceptions
public class PatientNotFoundException extends AdvaHealthException {
    public PatientNotFoundException(Long id) {
        super("Patient not found: " + id, "PATIENT_NOT_FOUND", HttpStatus.NOT_FOUND);
    }
}

public class DuplicateMRNException extends AdvaHealthException {
    public DuplicateMRNException(String mrn) {
        super("MRN already exists: " + mrn, "DUPLICATE_MRN", HttpStatus.CONFLICT);
    }
}

public class DicomProcessingException extends AdvaHealthException {
    public DicomProcessingException(String filename, Throwable cause) {
        super("Failed to process DICOM file: " + filename, "DICOM_PROCESSING_FAILED",
              HttpStatus.UNPROCESSABLE_ENTITY, cause);
    }
}

public class ExternalEhrException extends AdvaHealthException {
    public ExternalEhrException(String ehrSystem, int statusCode) {
        super("EHR integration error [" + ehrSystem + "] status=" + statusCode,
              "EHR_INTEGRATION_ERROR", HttpStatus.SERVICE_UNAVAILABLE);
    }
}
```

---

## 3. Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // ✅ Domain exceptions — predictable, structured response
    @ExceptionHandler(AdvaHealthException.class)
    public ResponseEntity<ErrorResponse> handleDomainException(
            AdvaHealthException ex, HttpServletRequest request) {
        log.warn("Domain exception [{}]: {} — path={}", 
            ex.getErrorCode(), ex.getMessage(), request.getRequestURI());
        
        return ResponseEntity
            .status(ex.getHttpStatus())
            .body(ErrorResponse.of(ex.getErrorCode(), ex.getMessage(), request));
    }

    // ✅ Validation errors — 400
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex,
                                           HttpServletRequest request) {
        List<FieldError> fieldErrors = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
            .toList();
        
        return ErrorResponse.ofValidation("VALIDATION_FAILED", fieldErrors, request);
    }

    // ✅ JSON parse error — 400
    @ExceptionHandler(HttpMessageNotReadableException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleMalformedJson(HttpMessageNotReadableException ex,
                                              HttpServletRequest request) {
        log.debug("Malformed request body: {}", ex.getMessage());
        return ErrorResponse.of("MALFORMED_REQUEST", "Request body is malformed or missing", request);
    }

    // ✅ Access denied — 403
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(HttpServletRequest request) {
        return ErrorResponse.of("FORBIDDEN", "Access denied", request);
    }

    // ✅ Catch-all — 500 — NEVER leak internal details
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex, HttpServletRequest request) {
        // Log with full stack trace server-side
        log.error("Unhandled exception on path={}", request.getRequestURI(), ex);
        
        // Return generic message — scrub internals from response
        return ErrorResponse.of("INTERNAL_ERROR",
            "An unexpected error occurred. Please try again later.", request);
    }
}

// Error response DTO
public record ErrorResponse(
    String errorCode,
    String message,
    String path,
    Instant timestamp,
    List<FieldError> fieldErrors
) {
    public static ErrorResponse of(String code, String message, HttpServletRequest req) {
        return new ErrorResponse(code, message, req.getRequestURI(), Instant.now(), null);
    }
    
    public static ErrorResponse ofValidation(String code, List<FieldError> errors,
                                              HttpServletRequest req) {
        return new ErrorResponse(code, "Validation failed", req.getRequestURI(), 
                                  Instant.now(), errors);
    }

    public record FieldError(String field, String message) {}
}
```

---

## 4. Exception Handling in Service Layer

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class PatientService {

    private final PatientRepository patientRepository;
    private final EhrIntegrationClient ehrClient;

    public Patient getPatient(Long id) {
        return patientRepository.findById(id)
            .orElseThrow(() -> new PatientNotFoundException(id));
    }

    // ✅ Translate low-level exceptions to domain exceptions
    @Transactional
    public Patient createPatient(CreatePatientRequest request) {
        // Fail fast on business rule violation
        if (patientRepository.existsByMrn(request.mrn())) {
            throw new DuplicateMRNException(request.mrn());
        }

        Patient patient = new Patient(request);
        
        try {
            patientRepository.save(patient);
        } catch (DataIntegrityViolationException ex) {
            // Race condition: another request slipped through the check
            log.warn("Data integrity violation creating patient mrn={}", request.mrn());
            throw new DuplicateMRNException(request.mrn());
        }
        
        return patient;
    }

    // ✅ Wrap external calls — don't let external failures propagate raw
    public Optional<EhrRecord> fetchEhrRecord(String patientRef) {
        try {
            return Optional.ofNullable(ehrClient.lookupPatient(patientRef));
        } catch (FeignException.NotFound e) {
            return Optional.empty(); // 404 from EHR → not found is a valid state
        } catch (FeignException e) {
            // Non-404 EHR error → escalate as domain exception
            log.error("EHR lookup failed for ref={}: status={}", 
                patientRef, e.status(), e);
            throw new ExternalEhrException("EPIC", e.status());
        }
    }
}
```

---

## 5. try-with-resources & AutoCloseable

```java
// Any resource that implements AutoCloseable is automatically closed
// even if an exception is thrown inside the block

// ✅ Correct — both stream and connection auto-closed
public List<String> readDicomSeriesIds(Path dicomFile) throws DicomProcessingException {
    try (var reader = new DicomInputStream(dicomFile.toFile());
         var stream = reader.readDataset()) {
        return stream.getStrings(Tag.SeriesInstanceUID);
    } catch (IOException e) {
        throw new DicomProcessingException(dicomFile.getFileName().toString(), e);
    }
}

// ❌ Classic resource leak — stream never closed if exception in processing
public List<String> badExample(Path file) throws IOException {
    BufferedReader reader = Files.newBufferedReader(file);
    String line;
    List<String> lines = new ArrayList<>();
    while ((line = reader.readLine()) != null) { // exception here → reader never closed
        lines.add(processLine(line));
    }
    reader.close(); // never reached if exception above
    return lines;
}
```

---

## 6. Exception Handling Anti-Patterns

```java
// ❌ Anti-pattern 1: Swallowing exceptions silently
try {
    patientRepository.save(patient);
} catch (Exception e) {
    // NEVER do this — you have no idea anything went wrong
}

// ❌ Anti-pattern 2: Catching Exception to log then re-throw — double logging
try {
    processPayment();
} catch (Exception e) {
    log.error("Error processing payment", e); // logs here
    throw new RuntimeException("Payment failed", e); // then global handler logs again
}
// Fix: either handle OR throw, not both. Let global handler log it.

// ❌ Anti-pattern 3: Using exceptions for flow control
// This is O(n) exception creation for a normal "not found" case
public boolean userExists(String email) {
    try {
        userService.findByEmail(email);
        return true;
    } catch (UserNotFoundException e) {
        return false; // using exception as conditional — BAD
    }
}
// Fix: use Optional or a dedicated exists() method
public boolean userExists(String email) {
    return userRepository.existsByEmail(email); // one query, no exception
}

// ❌ Anti-pattern 4: Losing the original cause
try {
    connection.execute(query);
} catch (SQLException e) {
    throw new RuntimeException("DB query failed"); // original stack trace LOST!
}
// Fix: always pass the cause
throw new RuntimeException("DB query failed", e); // ✅

// ❌ Anti-pattern 5: Catching Throwable / Error
try {
    processRequest();
} catch (Throwable t) { // catches OutOfMemoryError, StackOverflowError too!
    // JVM might be in an unrecoverable state — you cannot safely handle Error
}
// Fix: only catch Exception (or RuntimeException for unchecked)
```

---

## 7. Retry Pattern — Transient Failures

```java
// Spring Retry — for transient external service failures
@Configuration
@EnableRetry
public class RetryConfig {}

@Service
public class EhrSyncService {

    @Retryable(
        retryFor = {ExternalEhrException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000)
    )
    public void syncPatientRecord(String patientId) {
        ehrClient.sync(patientId);
    }

    // Called after all retries exhausted
    @Recover
    public void syncFailed(ExternalEhrException ex, String patientId) {
        log.error("EHR sync failed after retries for patientId={}", patientId, ex);
        alertService.sendAlert("EHR_SYNC_FAILURE", patientId);
        // Put in dead-letter queue for manual review
        deadLetterRepository.save(new FailedSync(patientId, ex.getMessage()));
    }
}
```

---

## Interview Q&A

**Q: "How do you design exception handling for a production REST API?"**

📢 **Headline:** "I use three layers: domain-specific unchecked exceptions that carry HTTP status codes, a `@RestControllerAdvice` global handler that maps them to consistent JSON error responses, and service-level exception translation that wraps third-party failures into domain exceptions. The rule: never let raw `SQLException`, `FeignException`, or `IOException` reach the caller — always translate to a domain exception with meaningful context."

**Q: "What's the difference between `throw` and `throws`?"**

📢 **Headline:** "`throw` is the statement that raises an exception at runtime: `throw new PatientNotFoundException(id)`. `throws` is a **method signature declaration** that says 'this method may throw this checked exception': `public void process() throws IOException`. For unchecked exceptions, `throws` is optional (but serves as documentation)."

**Q: "When does `@Transactional` NOT rollback?"**

📢 **Headline:** "Three cases: (1) Checked exceptions — by default only `RuntimeException` triggers rollback. (2) Self-invocation — `this.method()` bypasses the AOP proxy. (3) Exception caught inside the `@Transactional` method without rethrowing — Spring never sees the exception."

---

*Next: [02-collections-generics.md](./02-collections-generics.md)*
