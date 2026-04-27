# ☕ Java & Spring Boot — Interview Prep for Middle+ Developers

> **Target audience:** Mid-level Java developer (2–5 years, Spring Boot in production) with gaps in theory & internals.  
> **Goal:** Walk into a technical interview feeling confident within **4 weeks**.  
> **Daily commitment:** 30–45 min/day

---

## 🚦 Start Here — Day 1 Gap Map

Before reading anything else, open [`week-4/12-interview-qna-master.md`](./week-4/12-interview-qna-master.md) and scan **Q1–Q10**.

- Answer each one out loud without looking at the answer.
- Mark the ones you fumbled.
- Those gaps are your Week 1 priority — not the topics you already know.

The Q&A master is not only a final review. It is a diagnostic tool. Use it on Day 1 and again on Day 28.

---

## 📊 Self-Assessment Checklist

Use this at the **end of each week** to confirm you can answer under pressure — not just recognize the right answer when you see it.

### Week 1
- [ ] Explain all 4 OOP principles with a live code example (no notes)
- [ ] Walk through `HashMap` internals: buckets → collision → treeify → resize
- [ ] Write a Stream pipeline — `filter → map → collect` — in under 30 seconds
- [ ] Explain `Optional.orElse` vs `orElseGet` difference and when it matters

### Week 2
- [ ] Explain `volatile` vs `synchronized` vs `AtomicInteger` — which to use when
- [ ] Name all JVM heap regions and what causes `OutOfMemoryError` in each
- [ ] Explain what a `record` is and why you can't use it as a JPA `@Entity`
- [ ] Use `CompletableFuture.thenCompose` correctly (not `thenApply` for async chain)

### Week 3
- [ ] Explain Spring Boot auto-configuration — step by step, no hand-waving
- [ ] Trace a JWT request from `Authorization` header to `SecurityContext`
- [ ] Explain `@Transactional` REQUIRED vs REQUIRES_NEW with a real use case
- [ ] Write a `@Query` with `JOIN FETCH` to eliminate an N+1

### Week 4
- [ ] Explain Hibernate L1 vs L2 cache: scope, invalidation, when each applies
- [ ] Answer "how would you scale this API?" with 5 concrete steps
- [ ] Pass a mock interview: 10 questions, answers under 90 seconds each

---

## 📅 4-Week Study Plan

| Week | Topics | Est. Time |
|------|--------|-----------|
| **Week 1** | Core Java: OOP, Collections, Generics, Java 8 features | 30–45 min/day |
| **Week 2** | Concurrency, JVM Internals, Java 9–21 features | 30–45 min/day |
| **Week 3** | Spring Boot Core, Spring Security, Spring Data JPA | 30–45 min/day |
| **Week 4** | Advanced JPA/DB, System Design, Full Q&A Review | 30–45 min/day |

---

## 📂 Folder Structure

```
java-interview-prep/
├── README.md                        ← You are here
│
├── week-1/
│   ├── 01-core-java-oop.md          ← OOP principles, SOLID, design patterns
│   ├── 02-collections-generics.md   ← Collections framework, Generics, comparators
│   └── 03-java8-modern-features.md  ← Lambda, Stream, Optional, Functional interfaces
│
├── week-2/
│   ├── 04-concurrency-threading.md  ← Threads, ExecutorService, CompletableFuture, locks
│   ├── 05-jvm-internals.md          ← Memory model, GC, ClassLoader, JIT
│   └── 06-java9-21-features.md      ← Records, Sealed classes, Switch expressions, Virtual threads
│
├── week-3/
│   ├── 07-spring-boot-core.md       ← Auto-config, DI, Bean lifecycle, Actuator, AOP
│   ├── 08-spring-security.md        ← JWT, OAuth2, Method security, Filter chain
│   └── 09-spring-data-jpa.md        ← Repositories, JPQL, Entity mapping, Transactions
│
├── week-4/
│   ├── 10-database-jpa-advanced.md  ← Hibernate cache, connection pool, migrations
│   ├── 11-system-design-microservices.md ← REST, microservices, caching, messaging
│   └── 12-interview-qna-master.md   ← 20 Q&A with headline answers + deep dives ⭐
│
└── cheatsheets/
    ├── java-cheatsheet.md           ← Quick reference: syntax, APIs, patterns
    └── spring-boot-cheatsheet.md    ← Annotations, config, common patterns
```

---

## 🎯 How to Use Each Document

Each topic section follows this structure — **code first, explanation after**:

```
Code example (bad/good pattern or concrete output)
→ 1–2 line explanation of why
→ Key Points to memorize
→ Interview Question
→ 📢 Headline Answer (1–2 sentences — what you say first)
→ Full Answer + internals
→ Gotchas (what trips people up)
```

### Study Strategy (per topic):
1. **Read the code example first.** If you understand it already, skip to the Interview Question.
2. **Cover the answer. Try to answer it.** Write it out or say it aloud.
3. **Check the Headline Answer.** Can you say it in under 20 seconds?
4. **Read the Full Answer + Deep Dive** only for topics you fumbled.
5. **Note the Gotchas.** These are the difference between a pass and a fail.

---

## 🏆 Topics Most Frequently Asked (Middle+ Level)

### High Frequency 🔥
- `HashMap` internals (hash collision, load factor, resize)
- `volatile` vs `synchronized` vs `AtomicInteger`
- Spring Bean lifecycle + `@Scope`
- `@Transactional` propagation + rollback rules
- N+1 query problem + solutions
- JWT authentication flow in Spring Security
- `CompletableFuture` chaining
- Java Stream API terminal vs intermediate operations

### Medium Frequency 🌡️
- JVM memory model (heap, stack, metaspace)
- Garbage Collector types + G1GC tuning
- Spring Auto-configuration mechanism
- JPA `@OneToMany` bidirectional mapping pitfalls
- `ConcurrentHashMap` vs `Collections.synchronizedMap`
- Design patterns: Factory, Strategy, Observer, Builder
- SOLID principles with examples

### Always Asked 📝
- "Explain OOP principles with real examples"
- "What's the difference between `==` and `.equals()`?"
- "How does Spring dependency injection work?"
- "What's the difference between `@Component`, `@Service`, `@Repository`?"

---

## 🚀 Key Annotations Quick Reference

| Annotation | Layer | Purpose |
|-----------|-------|---------|
| `@RestController` | Controller | REST API, combines @Controller + @ResponseBody |
| `@Service` | Service | Business logic |
| `@Repository` | Data | Data access + persistence exception translation |
| `@Transactional` | Service | Transaction management |
| `@Entity` | Model | JPA entity |
| `@SpringBootTest` | Test | Full application context |
| `@WebMvcTest` | Test | Controller layer only |
| `@DataJpaTest` | Test | JPA/repository only |

---

*Middle to Senior level. Start with the gap map, not page 1. 🎯*
