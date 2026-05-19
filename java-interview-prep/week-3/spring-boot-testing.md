# Week 3 — Bonus: Spring Boot Testing

> 📖 **Estimated reading time:** 45 minutes  
> 🎯 **Focus:** @WebMvcTest, @DataJpaTest, @SpringBootTest, Testcontainers, Mocking patterns  
> ⚠️ **Why it matters:** Q20 in the master Q&A — "How do you write testable code?" — is almost always asked. AdvaHealth needs engineers who can test healthcare APIs correctly.

---

## 1. Testing Pyramid for Spring Boot

```
        /\
       /E2E\         @SpringBootTest + real DB (Testcontainers)
      /______\       — few, slow, test full flows
     /        \
    /Integration\    @WebMvcTest + @DataJpaTest
   /____________\    — test slice: controller or DB layer
  /              \
 /   Unit Tests   \  Plain JUnit 5 + Mockito
/_________________ \ — many, fast, test isolated logic
```

**Rule:** Write mostly unit tests, some integration tests, few E2E tests.

---

## 2. Unit Testing — JUnit 5 + Mockito

```java
// ✅ Plain unit test — no Spring context, blazing fast
@ExtendWith(MockitoExtension.class)
class PatientServiceTest {

    @Mock
    PatientRepository patientRepository;
    
    @Mock
    EhrIntegrationClient ehrClient;

    @InjectMocks
    PatientService patientService;

    // AAA: Arrange → Act → Assert
    @Test
    void shouldReturnPatient_whenPatientExists() {
        // Arrange
        Patient patient = new Patient(1L, "MRN-00001", "John Doe");
        when(patientRepository.findById(1L)).thenReturn(Optional.of(patient));

        // Act
        Patient result = patientService.getPatient(1L);

        // Assert
        assertThat(result.getName()).isEqualTo("John Doe");
        verify(patientRepository).findById(1L);
    }

    @Test
    void shouldThrow_whenPatientNotFound() {
        // Arrange
        when(patientRepository.findById(99L)).thenReturn(Optional.empty());

        // Act & Assert
        assertThatThrownBy(() -> patientService.getPatient(99L))
            .isInstanceOf(PatientNotFoundException.class)
            .hasMessageContaining("99");
    }

    @Test
    void shouldThrowDuplicateMRN_whenMRNAlreadyExists() {
        // Arrange
        CreatePatientRequest req = new CreatePatientRequest("MRN-00001", "Jane Doe");
        when(patientRepository.existsByMrn("MRN-00001")).thenReturn(true);

        // Act & Assert
        assertThatThrownBy(() -> patientService.createPatient(req))
            .isInstanceOf(DuplicateMRNException.class);
        
        // Verify save was never called
        verifyNoMoreInteractions(patientRepository);
    }
}
```

### Key Mockito Methods
```java
// Stubbing
when(mock.method(arg)).thenReturn(value);
when(mock.method(any())).thenThrow(new RuntimeException());
doNothing().when(mock).voidMethod();
doThrow(ex).when(mock).voidMethod();

// Argument matchers
when(repo.findByStatus(eq(OrderStatus.ACTIVE))).thenReturn(list);
when(repo.findByName(anyString())).thenReturn(list);
when(repo.save(argThat(p -> p.getName().startsWith("J")))).thenReturn(saved);

// Verification
verify(mock).method(arg);                // called exactly once
verify(mock, times(3)).method(any());   // called 3 times
verify(mock, never()).method(any());    // never called
verifyNoMoreInteractions(mock);         // no other calls

// Capture argument
ArgumentCaptor<Patient> captor = ArgumentCaptor.forClass(Patient.class);
verify(repo).save(captor.capture());
assertThat(captor.getValue().getMrn()).isEqualTo("MRN-00001");
```

---

## 3. @WebMvcTest — Controller Layer Tests

```java
// Tests ONLY the controller layer
// Auto-configures: MockMvc, Jackson, Spring Security (partial)
// Does NOT start: full app context, datasource, service beans

@WebMvcTest(PatientController.class)
@Import(SecurityConfig.class)  // import security if your controller requires auth
class PatientControllerTest {

    @Autowired
    MockMvc mockMvc;                     // HTTP simulation

    @Autowired
    ObjectMapper objectMapper;           // JSON serialization

    @MockBean                            // Spring mock — replaces real bean in context
    PatientService patientService;

    @Test
    @WithMockUser(roles = "CLINICIAN")  // simulate authenticated user
    void shouldReturn200_whenPatientFound() throws Exception {
        // Arrange
        PatientDTO dto = new PatientDTO(1L, "MRN-00001", "John Doe");
        when(patientService.getPatient(1L)).thenReturn(dto);

        // Act & Assert
        mockMvc.perform(get("/api/v1/patients/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.mrn").value("MRN-00001"))
            .andExpect(jsonPath("$.name").value("John Doe"));
    }

    @Test
    @WithMockUser
    void shouldReturn404_whenPatientNotFound() throws Exception {
        when(patientService.getPatient(99L))
            .thenThrow(new PatientNotFoundException(99L));

        mockMvc.perform(get("/api/v1/patients/99"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.errorCode").value("PATIENT_NOT_FOUND"));
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldReturn201_whenPatientCreated() throws Exception {
        // Arrange
        CreatePatientRequest req = new CreatePatientRequest("MRN-00001", "Jane Doe",
                                                             LocalDate.of(1985, 3, 15), "jane@example.com");
        PatientDTO created = new PatientDTO(2L, "MRN-00001", "Jane Doe");
        when(patientService.createPatient(any())).thenReturn(created);

        // Act & Assert
        mockMvc.perform(post("/api/v1/patients")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.id").value(2));
    }

    @Test
    @WithMockUser
    void shouldReturn400_whenMRNBlank() throws Exception {
        // Send invalid request — MRN is @NotBlank
        CreatePatientRequest req = new CreatePatientRequest("", "Jane Doe",
                                                             LocalDate.of(1985, 3, 15), "jane@example.com");
        mockMvc.perform(post("/api/v1/patients")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"))
            .andExpect(jsonPath("$.fieldErrors[0].field").value("mrn"));
    }

    @Test
    void shouldReturn401_whenNotAuthenticated() throws Exception {
        mockMvc.perform(get("/api/v1/patients/1"))
            .andExpect(status().isUnauthorized());
    }
}
```

---

## 4. @DataJpaTest — Repository Layer Tests

```java
// Tests ONLY the JPA layer
// Auto-configures: in-memory H2 DB, JPA, Hibernate, Spring Data
// Rolls back each test automatically
// Does NOT start: web layer, services, security

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // use Testcontainers instead
class PatientRepositoryTest {

    @Autowired
    PatientRepository patientRepository;

    @Autowired
    TestEntityManager em;  // helper for test setup without going through the repo

    @Test
    void shouldFindByMrn_whenMrnExists() {
        // Arrange — persist directly via EntityManager
        Patient saved = em.persistAndFlush(
            new Patient("MRN-00001", "John Doe", LocalDate.of(1980, 1, 1))
        );

        // Act
        Optional<Patient> found = patientRepository.findByMrn("MRN-00001");

        // Assert
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John Doe");
    }

    @Test
    void shouldReturnEmpty_whenMrnNotExists() {
        Optional<Patient> result = patientRepository.findByMrn("NONEXISTENT");
        assertThat(result).isEmpty();
    }

    @Test
    void shouldCountActivePatients() {
        em.persistAndFlush(new Patient("MRN-001", "Alice", PatientStatus.ACTIVE));
        em.persistAndFlush(new Patient("MRN-002", "Bob", PatientStatus.ACTIVE));
        em.persistAndFlush(new Patient("MRN-003", "Charlie", PatientStatus.DISCHARGED));

        long count = patientRepository.countByStatus(PatientStatus.ACTIVE);

        assertThat(count).isEqualTo(2);
    }

    @Test
    void shouldDetectN1Problem() {
        // Create 5 patients with orders
        for (int i = 0; i < 5; i++) {
            Patient p = em.persistAndFlush(new Patient("MRN-00" + i, "Patient " + i));
            em.persistAndFlush(new Order(p, OrderStatus.ACTIVE));
        }
        em.clear(); // clear L1 cache to force DB queries

        // Count queries with datasource-proxy or statistics
        Statistics stats = em.getEntityManager().getEntityManagerFactory()
            .unwrap(SessionFactory.class).getStatistics();
        stats.setStatisticsEnabled(true);
        stats.clear();

        List<Order> orders = orderRepository.findAll();
        orders.forEach(o -> o.getPatient().getName()); // triggers lazy load

        assertThat(stats.getQueryExecutionCount())
            .as("Should use JOIN FETCH to avoid N+1")
            .isEqualTo(1); // fail if > 1 — you have N+1!
    }
}
```

---

## 5. @SpringBootTest + Testcontainers — Integration Tests

```java
// Full integration test — real PostgreSQL via Docker
// Spring context fully loaded
// Use sparingly — slow but highest confidence

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class PatientIntegrationTest {

    @Container
    @ServiceConnection  // Spring Boot 3.1+ auto-configures datasource from container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("advahealth_test")
        .withUsername("test")
        .withPassword("test");

    @Autowired
    TestRestTemplate restTemplate;      // Makes real HTTP calls to the app

    @Autowired
    PatientRepository patientRepository;

    @BeforeEach
    void setUp() {
        patientRepository.deleteAll(); // clean state before each test
    }

    @Test
    void shouldCreateAndRetrievePatient() {
        // Create
        CreatePatientRequest req = new CreatePatientRequest(
            "MRN-INT-001", "Integration Test Patient",
            LocalDate.of(1990, 5, 15), "integration@test.com"
        );
        
        ResponseEntity<PatientDTO> createResponse = restTemplate
            .withBasicAuth("admin", "password")
            .postForEntity("/api/v1/patients", req, PatientDTO.class);
        
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        Long id = createResponse.getBody().id();
        
        // Retrieve
        ResponseEntity<PatientDTO> getResponse = restTemplate
            .withBasicAuth("clinician", "password")
            .getForEntity("/api/v1/patients/" + id, PatientDTO.class);
        
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().mrn()).isEqualTo("MRN-INT-001");
    }
}
```

### pom.xml — Testcontainers dependencies
```xml
<!-- Testcontainers BOM -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 6. Testing JWT-Protected Endpoints

```java
@WebMvcTest(PatientController.class)
class SecuredPatientControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean PatientService patientService;
    @MockBean JwtService jwtService;           // mock the JWT validation

    @Test
    void shouldAllow_withValidToken() throws Exception {
        // Arrange — mock JWT validation
        String token = "eyJ...valid.token";
        UserDetails userDetails = User.builder()
            .username("clinician@hospital.com")
            .password("")
            .roles("CLINICIAN")
            .build();
        
        when(jwtService.extractUsername(token)).thenReturn("clinician@hospital.com");
        when(jwtService.isTokenValid(eq(token), any())).thenReturn(true);
        when(userDetailsService.loadUserByUsername("clinician@hospital.com"))
            .thenReturn(userDetails);

        mockMvc.perform(get("/api/v1/patients/1")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }

    @Test
    void shouldBlock_withNoToken() throws Exception {
        mockMvc.perform(get("/api/v1/patients/1"))
            .andExpect(status().isUnauthorized());
    }

    // ✅ Simpler approach for most tests — use @WithMockUser
    @Test
    @WithMockUser(username = "clinician@hospital.com", roles = "CLINICIAN")
    void shouldAllow_withMockUser() throws Exception {
        when(patientService.getPatient(1L))
            .thenReturn(new PatientDTO(1L, "MRN-001", "John"));
        
        mockMvc.perform(get("/api/v1/patients/1"))
            .andExpect(status().isOk());
    }
}
```

---

## 7. Testing @Async and Events

```java
@SpringBootTest
class PatientEventTest {

    @Autowired PatientService patientService;
    @SpyBean ApplicationEventPublisher eventPublisher;  // spy = real bean + verify calls

    @Test
    void shouldPublishPatientCreatedEvent_onCreatePatient() {
        // Arrange
        CreatePatientRequest req = new CreatePatientRequest("MRN-EVT-001", "Event Test");

        // Act
        patientService.createPatient(req);

        // Assert — event was published
        ArgumentCaptor<PatientCreatedEvent> captor =
            ArgumentCaptor.forClass(PatientCreatedEvent.class);
        verify(eventPublisher).publishEvent(captor.capture());
        assertThat(captor.getValue().getMrn()).isEqualTo("MRN-EVT-001");
    }
}
```

---

## 8. Test Coverage — What to Test at Each Layer

| Layer | What to test | What NOT to test |
|-------|-------------|-----------------|
| **Service (Unit)** | Business logic, validation, edge cases, exception throwing | Getters/setters, framework behavior |
| **Controller (@WebMvcTest)** | Request/response mapping, status codes, validation, auth | Service logic (mock it) |
| **Repository (@DataJpaTest)** | Custom queries, derived query correctness, N+1 detection | Standard Spring Data methods |
| **Integration** | Full request-response flow, DB constraints, cross-service interactions | Every edge case (leave for unit) |

---

## Interview Q&A

**Q: "What's the difference between `@Mock` and `@MockBean`?"**

📢 **Headline:** "`@Mock` is a Mockito annotation — creates a mock object, no Spring context. Use in plain unit tests with `@ExtendWith(MockitoExtension.class)`. `@MockBean` is a Spring Boot annotation — creates a mock AND registers it as a Spring bean in the test application context, replacing the real bean. Use in slice tests (`@WebMvcTest`, `@SpringBootTest`) where Spring needs to wire the context."

**Q: "When would you use Testcontainers vs H2?"**

📢 **Headline:** "H2 is fast and simple but dialect differences can hide production bugs (H2 vs PostgreSQL behavior). Testcontainers runs the real PostgreSQL in Docker — tests are slower but faithfully test production behavior including indexes, constraints, JSONB, and full-text search. For healthcare, I prefer Testcontainers on CI and use H2 only for quick local unit tests of query logic."

**Q: "How do you test `@Transactional` behavior?"**

📢 **Headline:** "Use `@DataJpaTest` with Testcontainers. Call the service method, flush and clear the EntityManager, then verify the DB state directly via repository queries. For rollback behavior: set up data, call the service method that should fail, catch the exception, then assert the data is unchanged."

```java
@Test
void shouldRollback_whenPaymentFails() {
    // Arrange
    Order order = em.persistAndFlush(new Order(OrderStatus.PENDING));
    
    // Act — this method throws after creating order line items
    assertThatThrownBy(() -> orderService.processWithPayment(order.getId()))
        .isInstanceOf(PaymentFailedException.class);
    
    em.clear();
    
    // Assert — order should still be PENDING, no items created
    Order reloaded = orderRepository.findById(order.getId()).orElseThrow();
    assertThat(reloaded.getStatus()).isEqualTo(OrderStatus.PENDING);
    assertThat(reloaded.getItems()).isEmpty();
}
```

---

*Next: [10-database-jpa-advanced.md](../week-4/10-database-jpa-advanced.md)*
