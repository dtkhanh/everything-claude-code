# Week 4 — Day 1–2: Advanced Database & JPA

> 📖 **Estimated reading time:** 40 minutes  
> 🎯 **Focus:** Hibernate cache, connection pool, query optimization, migrations

---

## 1. Hibernate Caching

Two requests load `findById(1)`. Without caching: 2 DB queries. With L1: 1 query per session, served from session cache on the second call. With L2: 1 query across all sessions for the rest of the TTL.

```
Request 1: findById(1) → L1 miss → DB query → stored in L1 + L2
Request 1: findById(1) again → L1 HIT (same session) → no DB query

Request 2 (new session): findById(1) → L1 miss → L2 HIT → no DB query

L1 (Session Cache):   per EntityManager / per request — always ON, automatic
L2 (Second-Level):    shared across sessions — OFF by default, configure explicitly
Query Cache:          caches query results — must be enabled per query
```

### 💻 Code Example — L2 Cache Setup
```java
// 1. Add dependency: spring-boot-starter-cache + ehcache/hazelcast/redis

// 2. application.yml
// spring:
//   jpa:
//     properties:
//       hibernate:
//         cache:
//           use_second_level_cache: true
//           region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
//           use_query_cache: true
//   cache:
//     jcache:
//       config: classpath:ehcache.xml

// 3. Mark entity as cacheable
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // or NONSTRICT_READ_WRITE for eventual consistency
public class Category {
    @Id private Long id;
    private String name;
}

// 4. Cache collection
@OneToMany(mappedBy = "category")
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
private List<Product> products;

// 5. Cache query
@Query("SELECT c FROM Category c WHERE c.active = true")
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Category> findActiveCategories();
```

### 💻 Code Example — Spring @Cacheable (application-level)
```java
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id")
    public ProductDTO getProduct(Long id) {
        return productRepository.findById(id)
            .map(ProductDTO::from)
            .orElseThrow(NotFoundException::new);
    }
    
    @CachePut(value = "products", key = "#result.id")
    public ProductDTO updateProduct(Long id, UpdateProductRequest req) {
        // update and return — cache updated with new value
    }
    
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
    
    @CacheEvict(value = "products", allEntries = true)
    public void clearProductCache() { }
    
    // @Caching for multiple cache operations
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#id"),
        @CacheEvict(value = "product-lists", allEntries = true)
    })
    public void expireProduct(Long id) { ... }
}
```

### ❓ Interview Question
> "What's the difference between L1 and L2 cache in Hibernate?"

### ✅ Model Answer
**L1 (Session/Persistence Context)**: Scoped to a single `EntityManager`/`Session`. Always active, no configuration needed. Ensures identity: calling `em.find(User.class, 1L)` twice **in the same session** returns the same Java object. Cleared at end of transaction (or unit of work).

**L2 (Second-Level)**: Shared across all sessions in the same application. Stores serialized entity state, not Java objects. Must be explicitly enabled and configured (Ehcache, Hazelcast, Redis). Read-write strategy ensures consistency. Effective for **reference data** that rarely changes (countries, categories, config).

---

## 2. Connection Pooling — HikariCP

### 📌 Concept Summary
Opening a DB connection is expensive (~50ms). **Connection pools** pre-create connections and reuse them. HikariCP is Spring Boot's default.

### 💻 Configuration
```yaml
# application.yml — production-ready HikariCP config
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10        # don't over-provision! DB has connection limits
      minimum-idle: 5              # keep 5 idle connections ready
      connection-timeout: 3000     # fail fast if no connection available (3s)
      idle-timeout: 600000         # remove idle connections after 10min
      max-lifetime: 1800000        # replace connections after 30min (prevents stale)
      keepalive-time: 30000        # heartbeat query every 30s (prevents firewall drops)
      pool-name: AppHikariPool
      # Connection validation
      connection-test-query: SELECT 1  # for older JDBC drivers that don't support isValid()
```

### ❓ Interview Question
> "How do you determine the right connection pool size?"

### ✅ Model Answer
The famous formula: **`pool_size = (core_count * 2) + effective_spindle_count`** (Hikari docs). For a typical web app with mostly I/O:
- Start with 10 connections
- Monitor pool metrics: `hikaricp.connections.pending` (waiting for connection) — if this is non-zero under normal load, increase pool size
- **Don't over-provision**: PostgreSQL creates a process per connection (~5MB RAM). 1000 connections = 5GB just for connections
- For high-concurrency microservices: use PgBouncer (connection pooler in front of Postgres) to handle thousands of app connections with tens of DB connections

---

## 3. Database Migrations — Flyway

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true  # verify checksums match on startup
```

```
src/main/resources/db/migration/
├── V1__create_users_table.sql
├── V2__create_orders_table.sql
├── V3__add_user_email_index.sql
└── V4__add_order_status_column.sql
```

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    password    VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- V4__add_order_status_column.sql — safe migration pattern (backward compatible)
-- Step 1: Add nullable column (old app still works)
ALTER TABLE orders ADD COLUMN new_status VARCHAR(50);

-- Step 2: Backfill
UPDATE orders SET new_status = status::text WHERE new_status IS NULL;

-- Step 3: Add constraint (in V5 after app deploy)
ALTER TABLE orders ALTER COLUMN new_status SET NOT NULL;
ALTER TABLE orders DROP COLUMN status;
```

---

## 4. PostgreSQL Query Optimization

### 💻 Code Example — Indexing Strategies
```sql
-- B-tree indexes (default) — for equality and range queries
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Composite index — order matters! (user_id, status) serves queries on both
-- user_id alone and (user_id, status) together. NOT on status alone.
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Partial index — only index active orders (smaller, faster)
CREATE INDEX idx_orders_active ON orders(user_id) WHERE status = 'ACTIVE';

-- GIN index for full-text or JSONB
CREATE INDEX idx_products_search ON products USING GIN(to_tsvector('english', name || ' ' || description));

-- EXPLAIN ANALYZE — check if index is used
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123 AND status = 'ACTIVE';
-- Look for: "Index Scan" vs "Seq Scan" — Seq Scan on large table = missing index
```

---

*Next: [11-system-design-microservices.md](./11-system-design-microservices.md)*


