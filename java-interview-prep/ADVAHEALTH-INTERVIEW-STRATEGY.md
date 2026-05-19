# 🎯 AdvaHealth Solutions — Backend Developer Interview Strategy

> **Vị trí:** Backend Developer (Java Spring Boot)  
> **Mục tiêu:** Pass technical interview trong **6–8 tuần**  
> **Xuất phát điểm:** 2–3 năm Java & Spring Boot  
> **Commitment:** 45–60 min/day  

---

## CAPABILITY

Bạn đang chuẩn bị ứng tuyển vị trí **Backend Developer tại AdvaHealth Solutions** — một SaaS healthcare platform. Với 2–3 năm kinh nghiệm, bạn đã đáp ứng yêu cầu tối thiểu; nhiệm vụ bây giờ là **lấp gap, làm sâu internals, và thể hiện được tư duy senior** trong buổi phỏng vấn.

---

## CONSTRAINTS (Ràng buộc cần giữ vững)

- ✅ Không học lan man — mỗi tuần có 1 focus theme rõ ràng  
- ✅ Luôn học theo format: **Concept → Code → Interview Q&A → Gotchas**  
- ✅ 80% thời gian cho **Required Skills**, 20% cho **Nice to Have**  
- ✅ Mỗi cuối tuần: mock answer 5 câu hỏi không nhìn tài liệu  
- ⚠️ Healthcare/DICOM chỉ học nhẹ — đủ để *không bị bất ngờ*, không cần chuyên sâu  

---

## GAP ANALYSIS — Điểm xuất phát vs. Yêu cầu JD

| Chủ đề | Yêu cầu JD | Mức độ bạn cần nâng |
|--------|------------|----------------------|
| Java 17+ / Java 21 features | ⭐⭐⭐ Bắt buộc | 🔴 **HIGH** — Virtual threads, Records, Sealed |
| Spring Boot internals | ⭐⭐⭐ Bắt buộc | 🟡 **MEDIUM** — Deepen auto-config, AOP, Actuator |
| PostgreSQL & query optimization | ⭐⭐⭐ Bắt buộc | 🟡 **MEDIUM** — Index, EXPLAIN, N+1 |
| RESTful API design | ⭐⭐⭐ Bắt buộc | 🟢 **LOW** — Consolidate best practices |
| Secure coding / Spring Security | ⭐⭐⭐ Bắt buộc | 🔴 **HIGH** — JWT, OAuth2, OWASP Top 10 |
| Performance optimization | ⭐⭐⭐ Bắt buộc | 🟡 **MEDIUM** — Caching, connection pool, profiling |
| System Design | ⭐⭐ Quan trọng | 🔴 **HIGH** — Microservices, scaling patterns |
| AWS basics | ⭐ Nice to have | 🟡 **MEDIUM** — EC2, RDS, S3, ECS basics |
| React / Frontend | ⭐ Nice to have | 🟢 **LOW** — Biết đủ để nói chuyện được |
| DICOM / Healthcare domain | ⭐ Nice to have | 🟢 **LOW** — Chỉ cần hiểu khái niệm |

---

## IMPLEMENTATION CONTRACT — Kế hoạch 6 tuần chi tiết

### 📅 WEEK 1 — Java 17–21 Deep Dive + OOP Solidify
**Theme:** *"Interviewer sẽ hỏi bạn về Java mới nhất — đừng bị bắt bẽ"*

| Ngày | Chủ đề | Tài liệu |
|------|--------|----------|
| Mon | Records, Sealed Classes, Pattern Matching | `week-2/06-java9-21-features.md` |
| Tue | Virtual Threads (Java 21) — tại sao thay đổi mọi thứ | `bonus-virtual-threads-java21.md` |
| Wed | Switch Expressions, Text Blocks, instanceof | `week-2/06-java9-21-features.md` |
| Thu | Stream API nâng cao: Collectors, flatMap, groupingBy | `week-1/03-java8-modern-features.md` |
| Fri | CompletableFuture chaining + error handling | `week-2/04-concurrency-threading.md` |
| Sat | Review + Mock: 5 câu Java internals không nhìn notes | Self-test |
| Sun | Gap fill — ôn lại chỗ yếu nhất trong tuần | — |

**Checklist cuối tuần:**
- [ ] Giải thích Virtual Threads vs Platform Threads — khi nào dùng?
- [ ] Viết được `record` và giải thích tại sao không dùng với `@Entity`
- [ ] Dùng `Stream.collect(Collectors.groupingBy(...))` không cần Google

---

### 📅 WEEK 2 — Spring Boot Internals + Spring Security (JWT focus)
**Theme:** *"Đây là core của JD — phải thành thạo, không được mơ hồ"*

| Ngày | Chủ đề | Tài liệu |
|------|--------|----------|
| Mon | Auto-configuration deep dive: `@ConditionalOnClass`, `spring.factories` | `week-3/07-spring-boot-core.md` |
| Tue | Bean lifecycle: `@PostConstruct`, `@PreDestroy`, scope, proxy | `week-3/07-spring-boot-core.md` |
| Wed | AOP: `@Aspect`, `@Around`, cross-cutting concerns | `week-3/07-spring-boot-core.md` |
| Thu | Spring Security filter chain + JWT implementation | `week-3/08-spring-security.md` |
| Fri | OAuth2 Resource Server, Method Security (`@PreAuthorize`) | `week-3/08-spring-security.md` |
| Sat | Mock: Spring Security Q&A — trace JWT từ header đến SecurityContext | Self-test |
| Sun | Viết mini JWT auth từ đầu (không copy) và giải thích từng bước | Coding drill |

**Checklist cuối tuần:**
- [ ] Trace một HTTP request qua toàn bộ Spring Security filter chain
- [ ] Giải thích `@Transactional(propagation = REQUIRES_NEW)` với ví dụ thực tế
- [ ] Nói được sự khác nhau giữa `@Component`, `@Service`, `@Repository`, `@Controller`

---

### 📅 WEEK 3 — PostgreSQL, JPA/Hibernate, Performance
**Theme:** *"Healthcare data = critical data — chậm 100ms = patient record bị miss. Tối ưu hóa phải sâu sắc"*

| Ngày | Chủ đề | Tài liệu |
|------|--------|----------|
| Mon | JPA entity mapping: `@OneToMany`, `@ManyToMany`, bidirectional pitfalls | `week-3/09-spring-data-jpa.md` |
| Tue | N+1 problem: detect → fix với `JOIN FETCH`, `@EntityGraph`, `@BatchSize` | `week-3/09-spring-data-jpa.md` |
| Wed | PostgreSQL: Index types (B-tree, GIN, GiST, BRIN), EXPLAIN ANALYZE parsing | `week-4/13-postgresql-query-optimization.md` |
| Thu | Hibernate L1/L2 cache — scope, invalidation, Ehcache/Redis | `week-4/10-database-jpa-advanced.md` |
| Fri | Window functions, CTEs, JSONB optimization, query tuning | `week-4/13-postgresql-query-optimization.md` |
| Sat | `@Transactional` gotchas: self-invocation, rollback rules, nested | `bonus-transactional-gotchas.md` |
| Sun | Mock: viết query phức tạp + giải thích execution plan | Coding drill |

**Checklist cuối tuần:**
- [ ] Phát hiện và fix N+1 trong code review (không cần hint)
- [ ] Đọc được EXPLAIN ANALYZE output — nhìn ra `Seq Scan` vs `Index Scan`
- [ ] Viết composite index cho query phức tạp: `CREATE INDEX idx_orders_user_status_created ...`
- [ ] Biết khi nào dùng window functions vs correlated subqueries
- [ ] JSONB indexing: GIN index, generated column optimization

---

### 📅 WEEK 4 — Secure Coding + OWASP + API Design
**Theme:** *"Healthcare = regulated industry — security là non-negotiable"*

| Ngày | Chủ đề | File cần tạo / Nội dung học |
|------|--------|-----------------------------|
| Mon | OWASP Top 10 (2021) — từng lỗi với ví dụ Java Spring Boot | Xem phần bên dưới |
| Tue | SQL Injection prevention: Parameterized queries, JPA named params | — |
| Wed | Input validation: `@Valid`, `@Validated`, Custom Validators, Bean Validation | — |
| Thu | HTTPS, HSTS, CORS config trong Spring Boot cho healthcare API | — |
| Fri | Rate limiting: Bucket4j + Spring Boot, API abuse prevention | — |
| Sat | Security code review: nhìn code, tìm lỗi, đề xuất fix | Self-test |
| Sun | Mock: "Làm sao bạn đảm bảo API này secure?" — answer dưới 2 phút | Spoken drill |

**OWASP Top 10 Quick Map cho Java Spring Boot:**

| OWASP | Tên lỗi | Cách phòng trong Spring |
|-------|---------|------------------------|
| A01 | Broken Access Control | `@PreAuthorize`, Spring Security, principle of least privilege |
| A02 | Cryptographic Failures | BCrypt passwords, TLS, không log sensitive data |
| A03 | Injection | JPA queries, `@Param`, không dùng string concatenation |
| A04 | Insecure Design | Threat modeling, fail-fast validation |
| A05 | Security Misconfiguration | Actuator endpoints protected, no default creds |
| A06 | Vulnerable Components | Dependency scanning (OWASP Dependency-Check) |
| A07 | Auth Failures | JWT expiry, refresh token rotation, account lockout |
| A08 | Integrity Failures | Signed artifacts, verify checksums |
| A09 | Logging Failures | Structured logging, không log PII/PHI |
| A10 | SSRF | Whitelist external URLs, validate redirects |

**Checklist cuối tuần:**
- [ ] Tìm được ít nhất 3 lỗi security trong một đoạn code cho trước
- [ ] Giải thích cách implement rate limiting không làm chậm legitimate users
- [ ] Nói được HIPAA compliance liên quan đến logging như thế nào (PHI = Protected Health Information)

---

### 📅 WEEK 5 — System Design + Microservices + AWS Basics
**Theme:** *"Họ build SaaS healthcare platform — sẽ hỏi scale, reliability, observability"*

| Ngày | Chủ đề | Tài liệu |
|------|--------|----------|
| Mon | Microservices patterns: API Gateway, Service Discovery, Circuit Breaker | `week-4/11-system-design-microservices.md` |
| Tue | Caching strategies: Redis, Cache-Aside, Write-Through, TTL | `week-4/11-system-design-microservices.md` |
| Wed | Message queues: Kafka basics cho healthcare event streaming | `bonus-kafka-consumer-bugs.md` |
| Thu | AWS core: EC2, RDS (PostgreSQL), S3, ECS, CloudWatch | Tự học + bên dưới |
| Fri | HealthCheck API, Actuator, Distributed tracing concepts | — |
| Sat | System Design drill: "Design a patient data ingestion API that handles 10k req/min" | Self-test |
| Sun | Mock system design interview: 30 phút, whiteboard bằng text/diagram | — |

**AWS Minimum Viable Knowledge cho phỏng vấn này:**

```
EC2  → Virtual servers (biết instance types, Auto Scaling Groups)
RDS  → Managed PostgreSQL (Multi-AZ, Read Replicas, backup)
S3   → Object storage (DICOM images, documents, presigned URLs)
ECS  → Container orchestration (deploy Spring Boot Docker images)
CloudWatch → Logs, Metrics, Alarms
VPC  → Network isolation, security groups, subnets
IAM  → Roles, policies, principle of least privilege
```

**Checklist cuối tuần:**
- [ ] Vẽ được architecture diagram cho một healthcare API service trên AWS
- [ ] Giải thích Circuit Breaker pattern với Resilience4j
- [ ] Trả lời "How would you scale this API?" với 5 concrete steps

---

### 📅 WEEK 6 — Domain Knowledge + Mock Interviews + Gap Fill
**Theme:** *"Chuẩn bị sẵn sàng — không còn gap nào được phép tồn tại"*

| Ngày | Chủ đề |
|------|--------|
| Mon | DICOM & Healthcare domain basics (xem bên dưới) |
| Tue | Full mock interview: 10 câu technical, tự record và review |
| Wed | Behavioral questions: STAR format cho healthcare/product context |
| Thu | Code review session: tự review 1 Spring Boot project cũ của bạn |
| Fri | Gap fill — dựa trên điểm yếu phát hiện sau mock |
| Sat | Company research: AdvaHealth, EHR integrations, HL7/FHIR basics |
| Sun | Final review + mindset prep |

**DICOM / Healthcare Quick Reference:**
```
DICOM = Digital Imaging and Communications in Medicine
       → Standard cho lưu trữ và truyền ảnh y tế (X-ray, MRI, CT)
       → File format: .dcm
       → dcm4che = Java toolkit để đọc/xử lý DICOM files

EHR   = Electronic Health Record (Epic, Cerner, etc.)
FHIR  = Fast Healthcare Interoperability Resources
       → REST-based standard để exchange healthcare data
       → Endpoints: GET /Patient/{id}, GET /Observation

PHI   = Protected Health Information (HIPAA regulation)
HL7   = Health Level 7 — older messaging standard
```

---

## BEHAVIORAL QUESTIONS — STAR Format Prep

AdvaHealth là startup nimble/fast-moving. Họ sẽ hỏi:

| Câu hỏi | STAR angle nên dùng |
|---------|---------------------|
| "Tell me about a time you solved a production incident" | Tốc độ phản ứng, root cause analysis |
| "How do you handle technical debt?" | Trade-off giữa speed và quality |
| "Describe a time you disagreed with a technical decision" | Constructive, data-driven, team-oriented |
| "How do you ensure code quality in a fast-moving team?" | Code review, testing, automation |
| "Tell me about a complex backend system you designed" | Scale, trade-offs, lessons learned |

**STAR Format:**
- **S**ituation — Context ngắn gọn (1–2 câu)
- **T**ask — Bạn phải làm gì cụ thể
- **A**ction — Bạn làm gì, tại sao chọn cách đó
- **R**esult — Outcome đo được (số liệu tốt nhất)

---

## TOP 20 KHẢ NĂNG CAO SẼ BỊ HỎI

### Java & Spring Core
1. Virtual Threads Java 21 — what, why, when to use
2. `@Transactional` propagation levels + self-invocation trap
3. Spring Bean lifecycle — khác biệt `@Singleton` vs `@Prototype`
4. Auto-configuration mechanism — `@ConditionalOnClass` trace
5. JWT security flow — từ header đến SecurityContext

### Database & Performance
6. N+1 problem — detect và fix trong JPA (JOIN FETCH, @EntityGraph)
7. `EXPLAIN ANALYZE` output — đọc được execution plan, identify `Seq Scan` vs `Index Scan`
8. Index types — B-tree, GIN, BRIN, partial indexes — khi nào dùng cái nào
9. Window functions — `ROW_NUMBER()`, `RANK()`, `LAG/LEAD`, vs subqueries
10. HikariCP tuning + L1/L2 Hibernate cache invalidation

### System Design
11. "Design patient data ingestion API chịu 10k req/min" — sharding, rate limiting, resilience
12. Circuit Breaker pattern — tại sao cần trong healthcare (reliability critical)
13. Cache invalidation strategies — TTL, cache-aside, eager invalidation
14. Idempotency trong REST API — tại sao critical với payment/order (healthcare claims)
15. Database migration strategy (Flyway) — zero-downtime, backward compatible schema changes

### Security
16. JWT vs Session — trade-offs trong stateless service (healthcare context)
17. OWASP A03 (Injection) — parameterized queries, JPA protection
18. PHI (Protected Health Information) — HIPAA compliance, encryption, secure logging
19. API rate limiting — Bucket4j, per-user vs per-IP, DDoS protection
20. CORS misconfiguration — healthcare data sensitivity, origins whitelist

---

## DAILY ROUTINE (45–60 min/day)

```
[10 min]  Flashcard review — 5 câu từ week-4/12-interview-qna-master.md
[30 min]  Deep study topic của ngày (xem schedule trên)
[10 min]  Viết tay 1 snippet code không nhìn tài liệu
[5 min]   Note 1 gotcha mới học được hôm nay
```

**Cuối tuần thêm:**
```
[20 min]  Mock interview: 3–5 câu nói to, tính giờ
[15 min]  Review gap từ tuần vừa rồi
[10 min]  Preview topic tuần tới
```

---

## OPEN QUESTIONS (cần confirm trước phỏng vấn)

- [ ] Phỏng vấn có coding test live (LeetCode style) không, hay chỉ design + Q&A?
- [ ] Stack hiện tại của AdvaHealth: Spring Boot version? PostgreSQL version?
- [ ] Họ deploy trên AWS ECS hay EC2 hay Kubernetes?
- [ ] Team size, sprint cadence, on-call rotation như thế nào?
- [ ] Phần React/frontend — họ kỳ vọng contribution bao nhiêu % thời gian?

---

## HANDOFF — File liên quan cần đọc (theo thứ tự ưu tiên)

```
PRIORITY 1 (Must read — week 1-3):
├── week-1/01-core-java-oop.md
├── week-1/03-java8-modern-features.md
├── week-2/04-concurrency-threading.md
├── week-2/05-jvm-internals.md
├── week-2/06-java9-21-features.md          ← Virtual threads, Records
├── week-3/07-spring-boot-core.md
├── week-3/08-spring-security.md
└── week-3/09-spring-data-jpa.md

PRIORITY 2 (Deep dive — week 3-4):
├── week-4/10-database-jpa-advanced.md
├── week-4/11-system-design-microservices.md
├── week-4/12-interview-qna-master.md       ← Run as diagnostic ASAP
├── week-4/13-postgresql-query-optimization.md ← INDEX, EXPLAIN, N+1 deep dive
├── bonus-transactional-gotchas.md
├── bonus-hikaricp-connection-pool.md
└── bonus-kafka-consumer-bugs.md

PRIORITY 3 (Nice to have — week 5-6):
├── bonus-virtual-threads-java21.md
└── cheatsheets/ (Quick reference during review)
```

---

## SUCCESS METRICS

| Milestone | Target | Khi nào |
|-----------|--------|---------|
| Gap Map diagnostic | Identify top 5 weaknesses | Day 1 |
| Java 21 features fluent | Answer without hesitation | End Week 1 |
| Spring Security trace | Full JWT flow from memory | End Week 2 |
| Database optimization | Fix N+1 in code review | End Week 3 |
| Mock interview pass | 80%+ answers under 90 sec | End Week 5 |
| Interview ready | Confident, no big gaps | End Week 6 |

---

*Bắt đầu ngay hôm nay: mở `week-4/12-interview-qna-master.md`, scan Q1–Q10, note chỗ nào bạn mơ hồ. Đó là Week 1 priority của bạn. 🎯*
