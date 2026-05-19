# Week 4 — PostgreSQL Query Optimization Deep Dive

> 📖 **Estimated reading time:** 60 minutes  
> 🎯 **Focus:** Index strategy, EXPLAIN ANALYZE parsing, query tuning, N+1 detection at SQL level, window functions, CTEs, JSONB

---

## 1. Index Types & When to Use Each

### 🎯 B-Tree Index (Default)
Best for: **equality and range queries** — most common.

```sql
-- Simple index on one column
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Composite index on (user_id, status)
-- Serves: WHERE user_id = ? AND status = ?
--     AND: WHERE user_id = ?  (leading column)
-- Does NOT serve: WHERE status = ?  (non-leading column alone)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Descending index for ORDER BY DESC
CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);

-- Partial index — only index active orders (smaller, faster)
-- SELECT * FROM orders WHERE user_id = 123 AND status = 'ACTIVE'
CREATE INDEX idx_orders_active ON orders(user_id) WHERE status = 'ACTIVE';
```

**Interview Q:** "When would you use a partial index?"  
**A:** When 90% of your queries filter on a specific condition (e.g., `status = 'ACTIVE'`). Store only 10% of rows in the index → smaller size, faster lookups, faster inserts (fewer index updates).

---

### 🎯 GIN (Generalized Inverted Index)
Best for: **full-text search, JSONB, array columns**.

```sql
-- Full-text search index
CREATE INDEX idx_articles_fts ON articles USING GIN(
    to_tsvector('english', title || ' ' || body)
);

-- Query
SELECT * FROM articles 
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'database & optimization');

-- JSONB index
CREATE INDEX idx_user_metadata_tags ON users USING GIN(metadata -> 'tags');

-- Array column index
CREATE TABLE user_roles (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    roles TEXT[] NOT NULL  -- ARRAY type
);
CREATE INDEX idx_user_roles ON user_roles USING GIN(roles);

-- Query
SELECT * FROM user_roles WHERE roles @> ARRAY['admin'];  -- contains array
```

**Interview Q:** "Your app searches patient records by diagnosis keywords. What index?"  
**A:** `GIN(to_tsvector('english', diagnosis_text))`. FTS indexes are optimal for text search; B-tree would require prefix matching or be inefficient.

---

### 🎯 GiST (Generalized Search Tree)
Best for: **geometric data, range queries, proximity search**.

```sql
-- Spatial index (PostGIS extension)
CREATE EXTENSION postgis;
CREATE INDEX idx_hospitals_location ON hospitals USING GIST(location);

-- Nearby hospitals query
SELECT * FROM hospitals 
WHERE ST_DWithin(location, ST_GeomFromText('POINT(-74 40)'), 5000);  -- within 5km

-- Range-based index
CREATE INDEX idx_event_dates ON events USING GIST(event_daterange);
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    event_daterange DATERANGE NOT NULL
);

-- Query overlapping date ranges
SELECT * FROM events 
WHERE event_daterange && DATERANGE('2024-01-01', '2024-12-31');  -- overlaps
```

---

### 🎯 BRIN (Block Range Index)
Best for: **very large tables with sorted/sequential data** (time-series, append-only logs).

```sql
-- Instead of B-tree (which stores one entry per row):
-- BRIN stores metadata per "block" (e.g., min/max values in 128MB range)
-- Much smaller than B-tree, perfect for append-only logs

CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    action VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- B-tree index: 500MB on 100M rows
CREATE INDEX idx_logs_created_btree ON audit_logs(created_at);

-- BRIN index: 5MB on 100M rows (100x smaller!)
-- Good if you query by time range (e.g., "last 7 days")
-- Slightly slower than B-tree but acceptable for append-only data
CREATE INDEX idx_logs_created_brin ON audit_logs USING BRIN(created_at);

-- Query still works
SELECT * FROM audit_logs WHERE created_at > NOW() - INTERVAL '7 days';
```

**Use case:** Healthcare audit logs, time-series events, append-only transaction logs.

---

## 2. EXPLAIN ANALYZE — Reading Execution Plans

### 📊 Simple Query

```sql
EXPLAIN ANALYZE
SELECT o.*, u.email 
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'SHIPPED'
ORDER BY o.created_at DESC
LIMIT 100;
```

**Output (Postgres 14+):**

```
Limit  (cost=0.57..2.50 rows=100 width=256) (actual time=0.234..2.103 rows=100 loops=1)
  ->  Sort  (cost=0.57..5.41 rows=500 width=256) (actual time=0.231..2.089 rows=100 loops=1)
        Sort Key: o.created_at DESC
        Sort Space Used: 256 kB
        ->  Nested Loop  (cost=0.29..0.50 rows=500 width=256) (actual time=0.089..1.834 rows=500 loops=1)
              ->  Index Scan using idx_orders_status on orders o  (cost=0.29..0.35 rows=500 width=128)
                    Index Cond: status = 'SHIPPED'
                    Actual Rows: 500
              ->  Index Scan using idx_users_id on users u  (cost=0.00..0.00 rows=1 width=128)
                    Index Cond: id = o.user_id
                    Actual Rows: 1
Planning Time: 0.123 ms
Execution Time: 2.456 ms
```

### 🔍 How to Read It

| Metric | Meaning |
|--------|---------|
| `cost=0.57..2.50` | Planner's **estimated** startup..total cost (relative units, not milliseconds) |
| `actual time=0.234..2.103` | **Actual** clock time in milliseconds |
| `rows=100` | Planner estimated 100 rows; also actual returned rows |
| `loops=1` | How many times this node executed (>1 = problem) |
| `Index Scan using idx_orders_status` | ✅ Using index (good) |
| `Seq Scan on orders` | ⚠️ Sequential scan, not using index (bad for large tables) |

**Key red flags:**
- `Seq Scan` on millions of rows → missing index
- `Nested Loop` with `loops=1000` → N+1 equivalent in SQL
- `actual rows >> cost estimate rows` → statistics are stale, run `ANALYZE table_name`

---

### 💻 Example — Spotting N+1 in SQL

```java
// Java code (Spring Data JPA)
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    System.out.println(order.getUser().getName());  // Lazy load per order
}
```

**Generated SQL (without JOIN FETCH):**
```sql
SELECT o.* FROM orders o;  -- 1 query, gets 1000 rows

-- Then 1000 MORE queries triggered by lazy loading:
SELECT u.* FROM users u WHERE id = ?;  -- repeated 1000 times
SELECT u.* FROM users u WHERE id = ?;
...
```

**EXPLAIN ANALYZE would show:**
```
Nested Loop over 1000 iterations (in the application!)
  -> Seq Scan on orders (1000 rows)
  -> Index Scan on users for each iteration (1000 separate executions if we could see them)
```

**Fix — with JOIN FETCH:**
```jpql
SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = 'SHIPPED'
```

**Generates:**
```sql
SELECT o.*, u.* FROM orders o 
JOIN users u ON o.user_id = u.id 
WHERE o.status = 'SHIPPED';
-- 1 query, retrieves both at once
```

---

## 3. Query Tuning Techniques

### 🎯 Technique 1: Filter Early (Reduce Rows)

```sql
-- SLOW: Join first, filter after
SELECT o.id, u.email 
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at > '2024-01-01'  -- filter after join
  AND u.deleted_at IS NULL;

-- FAST: Filter before join (if possible)
SELECT o.id, u.email 
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at > '2024-01-01'  -- PostgreSQL optimizer may reorder, but be explicit
  AND u.deleted = FALSE;            -- index on users(deleted) helps
```

**Indexes needed:**
```sql
CREATE INDEX idx_orders_created_at ON orders(created_at);
CREATE INDEX idx_users_deleted ON users(deleted_at) WHERE deleted_at IS NOT NULL;
```

---

### 🎯 Technique 2: Window Functions (Avoid Subqueries)

```sql
-- SLOW: Subquery + correlated aggregate
SELECT o.id, o.status,
       (SELECT COUNT(*) FROM orders o2 WHERE o2.user_id = o.user_id) AS total_orders
FROM orders o
WHERE o.status = 'SHIPPED';
-- ^ Subquery runs for EVERY row (expensive)

-- FAST: Window function
SELECT o.id, o.status,
       COUNT(*) OVER (PARTITION BY o.user_id) AS total_orders
FROM orders o
WHERE o.status = 'SHIPPED';
-- ^ Single pass, window aggregate
```

**Other useful window functions:**
```sql
SELECT 
    o.id,
    o.amount,
    ROW_NUMBER() OVER (ORDER BY o.created_at) AS seq,
    RANK() OVER (PARTITION BY o.user_id ORDER BY o.amount DESC) AS amount_rank,
    LAG(o.created_at) OVER (ORDER BY o.created_at) AS prev_order_date,
    LEAD(o.amount) OVER (ORDER BY o.created_at) AS next_order_amount,
    SUM(o.amount) OVER (ORDER BY o.created_at ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7day_sum
FROM orders o;
```

---

### 🎯 Technique 3: Common Table Expressions (CTEs)

```sql
-- CTE 1: Active users
WITH active_users AS (
    SELECT id, email 
    FROM users 
    WHERE deleted_at IS NULL AND last_login > NOW() - INTERVAL '30 days'
),
-- CTE 2: Recent orders
recent_orders AS (
    SELECT o.id, o.user_id, o.amount, o.created_at
    FROM orders o
    WHERE o.created_at > NOW() - INTERVAL '7 days'
)
SELECT au.email, COUNT(ro.id) AS order_count, SUM(ro.amount) AS total_spent
FROM active_users au
LEFT JOIN recent_orders ro ON au.id = ro.user_id
GROUP BY au.id, au.email
ORDER BY total_spent DESC;
```

**Advantage:** Readable, reusable sub-expressions. PostgreSQL optimizes CTEs inline (MATERIALIZED only if needed).

---

## 4. JSONB Optimization

### 💻 Schema Design

```sql
-- Flexible patient metadata stored as JSONB
CREATE TABLE patients (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}'::JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert
INSERT INTO patients (name, metadata) VALUES (
    'John Doe',
    '{"age": 45, "conditions": ["diabetes", "hypertension"], "insurance": {"provider": "Aetna", "premium": 500}}'::JSONB
);

-- Queries
-- 1. Extract top-level value
SELECT metadata->>'age' AS age FROM patients WHERE name = 'John Doe';

-- 2. Check array contains value
SELECT * FROM patients WHERE metadata->'conditions' @> '"diabetes"'::JSONB;

-- 3. Extract nested value
SELECT metadata->'insurance'->>'provider' FROM patients;

-- 4. Search all fields (slow without index!)
SELECT * FROM patients WHERE metadata::text LIKE '%diabetes%';
```

### 📊 Index JSONB for Performance

```sql
-- GIN index on specific path
CREATE INDEX idx_patients_conditions ON patients USING GIN(metadata->'conditions');

-- GIN index on all JSONB (larger but searches any field)
CREATE INDEX idx_patients_metadata ON patients USING GIN(metadata);

-- Generated column + index (best for repeated queries)
ALTER TABLE patients ADD COLUMN first_condition TEXT GENERATED ALWAYS AS (
    (metadata->'conditions'->0)::TEXT
) STORED;
CREATE INDEX idx_patients_first_condition ON patients(first_condition);

-- Now this query is instant
SELECT * FROM patients WHERE first_condition = 'diabetes';
```

---

## 5. Common Gotchas & Anti-Patterns

### ❌ Gotcha 1: SELECT * in Joins

```sql
-- BAD: Retrieves all columns from both tables
SELECT * FROM orders o JOIN users u ON o.user_id = u.id;

-- GOOD: Select only needed columns (reduces network/memory)
SELECT o.id, o.status, u.email FROM orders o JOIN users u ON o.user_id = u.id;
```

---

### ❌ Gotcha 2: NOT IN with NULL

```sql
-- NULL handling in NOT IN
SELECT * FROM orders WHERE user_id NOT IN (SELECT id FROM users WHERE banned = TRUE);
-- If users table has ANY NULL id value in the subquery result, entire NOT IN returns no rows!

-- FIX: Use NOT EXISTS instead
SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id AND u.banned = TRUE);
```

---

### ❌ Gotcha 3: Implicit Type Coercion

```sql
-- Column is VARCHAR, but you pass NUMBER
SELECT * FROM orders WHERE order_number = 123;  -- Postgres converts 123 -> '123' (slow)

-- Can't use index! Plan shows Seq Scan
-- GOOD: Match types
SELECT * FROM orders WHERE order_number = '123'::VARCHAR;
```

---

### ❌ Gotcha 4: LIKE Without Index

```sql
-- No index on text column
CREATE INDEX idx_user_name ON users(name);
SELECT * FROM users WHERE name LIKE 'John%';  -- Can use index (prefix match)
SELECT * FROM users WHERE name LIKE '%John%'; -- ⚠️ Can't use index (substring match)

-- Solution: Full-text search index
CREATE INDEX idx_users_fts ON users USING GIN(to_tsvector('english', name));
SELECT * FROM users WHERE to_tsvector('english', name) @@ to_tsquery('english', 'John');
```

---

## 6. Interview Questions & Answers

### Q1: "You notice query is slow. Walk me through diagnosis steps."

**A:**
1. Run `EXPLAIN ANALYZE SELECT ...` to see actual execution plan
2. Look for `Seq Scan` on large tables — missing index?
3. Check `actual rows vs estimated rows` — statistics stale? Run `ANALYZE table_name`
4. Look for `Nested Loop` with high `loops` count — N+1 equivalent?
5. Check indexes exist: `SELECT * FROM pg_stat_user_indexes WHERE relname = 'table_name'`
6. Test: add index, re-EXPLAIN, compare times

---

### Q2: "You have 10M patient records. Query on status field in JSONB metadata is slow. What do you do?"

**A:**
1. First, make sure there's a GIN index: `CREATE INDEX idx_patients_metadata ON patients USING GIN(metadata)`
2. If query still slow despite index, consider extracting status to a **generated column**:
   ```sql
   ALTER TABLE patients ADD COLUMN status TEXT GENERATED ALWAYS AS 
       ((metadata->>'status')::TEXT) STORED;
   CREATE INDEX idx_patients_status ON patients(status);
   ```
3. Rewrite query: `WHERE patients.status = 'ACTIVE'` instead of `WHERE metadata->>'status' = 'ACTIVE'`
4. Verify with `EXPLAIN ANALYZE`

---

### Q3: "Design indexes for a query: SELECT * FROM orders WHERE user_id = ? AND status = ? ORDER BY created_at DESC LIMIT 100"

**A:**
1. **Composite index** on (user_id, status, created_at):
   ```sql
   CREATE INDEX idx_orders_user_status_created ON orders(user_id, status DESC, created_at DESC);
   ```
   This is a **covering index** — all columns needed for the query are in the index. No need to touch the table.

2. **Verify:**
   ```sql
   EXPLAIN ANALYZE SELECT * FROM orders u.status WHERE user_id = 1 AND status = 'SHIPPED' ORDER BY created_at DESC LIMIT 100;
   ```
   Should see `Index Only Scan` (not even touching the table's main storage).

---

### Q4: "What's the difference between a correlated subquery and a JOIN?"

**A:**
```sql
-- CORRELATED SUBQUERY (runs for each outer row)
SELECT o.id, u.email,
       (SELECT COUNT(*) FROM orders o2 WHERE o2.user_id = o.user_id) AS total_orders
FROM orders o
WHERE o.status = 'SHIPPED';

-- JOIN with GROUP BY (single pass)
SELECT o.id, u.email, COUNT(*) AS total_orders
FROM orders o
JOIN (SELECT user_id, COUNT(*) FROM orders GROUP BY user_id) agg USING(user_id)
WHERE o.status = 'SHIPPED';
```

Correlated subquery **slow for large tables** (runs subquery for each row). Use window functions or JOIN instead.

---

## 7. Production Monitoring Queries

### Monitor Index Usage

```sql
-- Which indexes are NOT being used? (candidates for removal)
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Which indexes are bloated?
SELECT current_database(), schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_blk_read + idx_blk_hit > 1000
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Slow Query Log

```yaml
# postgresql.conf
log_min_duration_statement = 1000  // log queries > 1 second
log_statement = 'ddl'              // log DDL statements
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Then query logs
SELECT * FROM pg_stat_statements 
ORDER BY total_time DESC 
LIMIT 20;
```

---

## 8. Checklist for Interview

- [ ] Can explain B-tree, GIN, BRIN indexes and when to use each
- [ ] Can read EXPLAIN ANALYZE output: cost, rows, index scan vs seq scan
- [ ] Detect N+1 in SQL (`Nested Loop` with high loops count)
- [ ] Write indexes for complex queries (composite, partial, covering)
- [ ] Know window functions (ROW_NUMBER, RANK, LAG/LEAD, SUM() OVER)
- [ ] CTEs vs subqueries — know trade-offs
- [ ] JSONB indexing: GIN, generated columns, query optimization
- [ ] Common gotchas: NOT IN with NULL, type coercion, LIKE without index
- [ ] Production monitoring: index bloat, slow query logs

---

*Next: [12-interview-qna-master.md](./12-interview-qna-master.md)*

