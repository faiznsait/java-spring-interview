# Topic 16: Database, SQL, and JPA

## Q109. Indexing Types & Non-Indexed Query Behavior

### Concept
An index is a **data structure** (typically B-Tree) that speeds up data retrieval by maintaining a sorted reference to table rows, similar to a book's index.

### Simple Explanation
**Without index:** Find "John" in a 1-million-row table → read all 1 million rows (full table scan).  
**With index on name:** Index is sorted → binary search → find "John" in ~20 lookups (log₂ 1,000,000 ≈ 20).

### Index Types

#### 1. B-Tree Index (Default, Most Common)
```sql
-- Best for: equality, range queries, sorting
CREATE INDEX idx_users_email ON users(email);

-- Efficient queries:
SELECT * FROM users WHERE email = 'john@example.com';  -- O(log n)
SELECT * FROM users WHERE age BETWEEN 25 AND 35;      -- O(log n)
SELECT * FROM users ORDER BY email;                   -- Uses index, no sort
```

**Structure:**
```
        [M]
       /   \
    [A-F]  [N-Z]
    /  \    /  \
  [A] [D] [N] [S]
```

#### 2. Hash Index
```sql
-- Best for: exact equality only (no range, no sorting)
CREATE INDEX idx_users_id USING HASH ON users(id);

-- Fast:
SELECT * FROM users WHERE id = 123;  -- O(1)

-- NOT efficient (can't use hash index):
SELECT * FROM users WHERE id > 100;
SELECT * FROM users ORDER BY id;
```

#### 3. Composite Index (Multi-Column)
```sql
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- Uses index efficiently (left-to-right):
✅ WHERE user_id = 5
✅ WHERE user_id = 5 AND created_at > '2026-01-01'
✅ WHERE user_id = 5 ORDER BY created_at

-- Does NOT use index:
❌ WHERE created_at > '2026-01-01'  (skips user_id — left side)
```

**Rule:** Composite index works **left-to-right**. Must use leftmost column in WHERE clause.

#### 4. Unique Index
```sql
-- Enforces uniqueness + fast lookup
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Prevents duplicate emails at DB level
-- Faster than non-unique index (stops search at first match)
```

#### 5. Full-Text Index
```sql
-- Best for: text search in large text fields
CREATE FULLTEXT INDEX idx_posts_content ON posts(content);

SELECT * FROM posts WHERE MATCH(content) AGAINST('microservices spring boot');
```

#### 6. Partial/Filtered Index
```sql
-- Index only specific rows (PostgreSQL, SQL Server)
CREATE INDEX idx_active_users ON users(email) WHERE status = 'ACTIVE';

-- Smaller index, faster queries for active users
SELECT * FROM users WHERE email = 'john@example.com' AND status = 'ACTIVE';
```

### What Happens on Non-Indexed Column?

```sql
-- Table: orders (10 million rows), no index on user_id
SELECT * FROM orders WHERE user_id = 12345;

Execution:
1. Full Table Scan — reads ALL 10 million rows
2. Checks user_id = 12345 for each row
3. Returns matching rows
4. Time: 5-10 seconds (millions of disk reads)

-- After adding index:
CREATE INDEX idx_orders_user_id ON orders(user_id);

Execution:
1. B-Tree lookup in index → finds rows with user_id = 12345 in ~20 operations
2. Reads only those specific rows (e.g., 50 rows)
3. Time: 10-20ms (thousands of times faster)
```

### JPA Index Annotation
```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_user_id", columnList = "user_id"),
    @Index(name = "idx_status", columnList = "status"),
    @Index(name = "idx_user_status_created", columnList = "user_id, status, created_at")
})
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "user_id", nullable = false)
    private Long userId;
    
    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Column(name = "created_at", nullable = false)
    private Instant createdAt;
}
```

### Interview Answer
> "An index is a sorted data structure (B-Tree) that speeds up lookups from O(n) to O(log n). Without an index, the database does a full table scan — reads every row. With an index, it uses binary search to jump directly to matching rows.
>
> Common types: B-Tree (default, for equality/range/sorting), Hash (exact match only, faster but no range), Composite (multi-column, left-to-right matching), Unique (enforces uniqueness + fast lookup). In my project, I index all foreign keys (user_id, product_id) and common WHERE/ORDER BY columns.
>
> For a query on 10 million rows without an index: 5-10 seconds. With index: 10-20ms. I use EXPLAIN PLAN to verify index usage before deploying queries."

---

## Q110. Impact of Frequent Deletes on Indexed Tables

### The Problem
```sql
-- Over time with frequent deletes:
INSERT 1 million rows → Index builds
DELETE 500,000 rows  → Rows gone BUT index still references them
INSERT 200,000 rows  → Index grows
DELETE 100,000 rows  → More dead references

Result:
  - Index bloat — index physical size >> actual data size
  - Index fragmentation — logical order ≠ physical order
  - Slower queries — index traversal hits dead entries
  - Wasted disk space
```

### How Indexes Handle Deletes

```
B-Tree Index Before Delete:
  [A, B, C, D, E, F, G, H]

DELETE WHERE name = 'D';

B-Tree After Delete (Logical):
  [A, B, C, _, E, F, G, H]
  
  - Entry marked as "deleted" but not physically removed
  - Space not immediately reclaimed
  - Index still grows in size
```

### Performance Impact

| Metric | Before Bloat | After 50% Deletes (Bloated) |
|---|---|---|
| Index size | 500 MB | 800 MB |
| Actual data size | 500 MB | 250 MB |
| Index scan time | 100ms | 180ms |
| Wasted space | 0 MB | 550 MB |

### Solutions

#### 1. REINDEX (PostgreSQL)
```sql
-- Rebuilds index from scratch, removes dead entries
REINDEX INDEX idx_orders_user_id;
REINDEX TABLE orders;  -- Reindex all indexes on table

-- Can be slow on large tables (blocks writes during rebuild)
```

#### 2. REBUILD Index (Oracle, SQL Server)
```sql
-- SQL Server
ALTER INDEX idx_orders_user_id ON orders REBUILD;

-- Oracle
ALTER INDEX idx_orders_user_id REBUILD;
```

#### 3. VACUUM (PostgreSQL Auto-Maintenance)
```sql
-- Reclaims space from deleted rows and index entries
VACUUM ANALYZE orders;

-- Auto-vacuum enabled by default in PostgreSQL
-- Runs automatically when dead tuples exceed threshold
```

#### 4. Soft Delete (Application-Level)
```java
// Instead of DELETE, set a flag
@Entity
public class Order {
    private boolean deleted = false;
    
    @PreRemove
    void markDeleted() {
        this.deleted = true;
    }
}

// Filter deleted records at query time
@Query("SELECT o FROM Order o WHERE o.deleted = false")
List<Order> findAllActive();

// Periodically purge old deleted records (e.g., after 90 days)
@Scheduled(cron = "0 0 2 * * *")  // 2 AM daily
void purgeDeletedOrders() {
    Instant cutoff = Instant.now().minus(90, ChronoUnit.DAYS);
    orderRepo.deleteByDeletedTrueAndDeletedAtBefore(cutoff);
}
```

#### 5. Partitioning (For Time-Series Data)
```sql
-- Partition by month — drop entire partitions instead of DELETE
CREATE TABLE orders (
    id BIGINT,
    user_id BIGINT,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Drop old partition instantly (no DELETE, no index bloat)
DROP TABLE orders_2025_01;  -- Drops partition + indexes instantly
```

### Monitoring Index Health

```sql
-- PostgreSQL: Check index bloat
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    100 * (pg_relation_size(indexrelid)::float / pg_relation_size(tablename::regclass)) AS index_ratio
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- If index_ratio > 50% → index is bloated, consider REINDEX
```

### Interview Answer
> "Frequent deletes cause index bloat — deleted entries stay in the index but marked invalid, wasting space and slowing scans. A 1GB table with 50% deletes may have a 2GB index. Solution: REINDEX/REBUILD periodically to remove dead entries and defragment.
>
> In my project, for audit logs I use soft deletes with a `deleted` flag — avoids DELETE entirely. Periodically purge records older than 90 days. For time-series data (orders, logs), I use partitioning — drop entire partitions monthly instead of millions of DELETEs, no index bloat.
>
> PostgreSQL auto-vacuum helps but isn't instant. For high-churn tables, schedule weekly REINDEX during low-traffic windows. Monitor with `pg_stat_user_indexes` — if index size >> data size, it's bloated."

---

## Q111. Indexing on All Columns — Trade-offs

### Can You Index All Columns?
Technically yes. Practically **no** — massive downsides outweigh benefits.

### Trade-offs

| | With Indexes | Without Indexes |
|---|---|---|
| **SELECT speed** | Fast (O(log n)) | Slow (O(n) full scan) |
| **INSERT speed** | Slow — must update all indexes | Fast — just insert row |
| **UPDATE speed** | Slow — must update indexes on changed columns | Fast — just update row |
| **DELETE speed** | Slow — must update all indexes | Fast — just delete row |
| **Disk space** | High — each index consumes storage | Low — only data stored |
| **Memory usage** | High — indexes loaded in RAM | Low — only recently used data |

### Real Impact Example

```sql
-- Table: users (10 columns, 1 million rows)

Scenario 1: No indexes (except PK)
  INSERT 1000 rows: 200ms
  Disk space: 500 MB

Scenario 2: Index on all 10 columns
  INSERT 1000 rows: 2000ms (10× slower)
  Disk space: 2.5 GB (5× larger)
  
Each INSERT/UPDATE must:
  1. Write to table
  2. Update 10 indexes (10 B-Tree updates)
  3. Write to transaction log
```

### When to Index

```
✅ INDEX:
- Primary Key (always indexed automatically)
- Foreign Keys (user_id, product_id, order_id)
- Columns in WHERE frequently (status, created_at)
- Columns in ORDER BY (created_at, name)
- Columns in JOIN conditions
- Unique constraints (email)

❌ DON'T INDEX:
- Small tables (< 1000 rows) — full scan is fast enough
- Columns with low cardinality (gender: M/F — only 2 values)
- Columns rarely queried
- Columns frequently updated (high write cost)
- Columns with large text (better: full-text index)
- Boolean columns (unless query is highly selective)
```

### Cardinality Rule
```sql
-- Low cardinality — index not helpful
SELECT * FROM users WHERE gender = 'M';
-- Returns 50% of rows — index scan + 500,000 lookups slower than full scan

-- High cardinality — index very helpful
SELECT * FROM users WHERE email = 'john@example.com';
-- Returns 1 row — index lookup finds it in ~20 operations
```

**Rule:** Only index columns with high cardinality (many distinct values).

### Covering Index (Optimization)
```sql
-- Query needs: user_id, created_at, status
SELECT user_id, created_at, status FROM orders WHERE user_id = 123;

-- Instead of separate indexes, create covering index with all needed columns
CREATE INDEX idx_orders_covering ON orders(user_id, created_at, status);

-- Now query is "index-only scan" — no need to access table at all
-- Faster than index + table lookup
```

### Interview Answer
> "You can index all columns but shouldn't — trade SELECT speed for INSERT/UPDATE/DELETE speed and disk space. Each index must be updated on every write operation. A table with 10 indexes takes 10× longer to insert than one with no indexes.
>
> Index selectively: primary keys, foreign keys, WHERE/ORDER BY columns, and high-cardinality columns (email, username). Don't index low-cardinality columns (gender, boolean flags) — a full table scan is often faster. Don't index frequently updated columns unless absolutely necessary.
>
> I profile with EXPLAIN to identify missing indexes (table scans), but I'm conservative — only add indexes that measurably improve query performance. Too many indexes slow down writes and ORM bulk inserts dramatically."

---

## Q112. EXPLAIN PLAN & Query Optimization

### What EXPLAIN Shows
EXPLAIN reveals the **query execution plan** — how the database will execute your query: which indexes, join algorithms, estimated cost, and row counts.

### PostgreSQL EXPLAIN
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- Output:
Seq Scan on orders  (cost=0.00..18334.00 rows=1000 width=120)
  Filter: (user_id = 123)

-- Translation:
-- Sequential Scan (full table scan) — BAD
-- Estimated cost: 18334
-- Estimated rows: 1000
```

After adding index:
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);

EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- Output:
Index Scan using idx_orders_user_id on orders  (cost=0.29..8.31 rows=1000 width=120)
  Index Cond: (user_id = 123)

-- Translation:
-- Index Scan — GOOD
-- Cost reduced from 18334 to 8.31 (2200× cheaper)
```

### EXPLAIN ANALYZE (Shows Actual Execution)
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

-- Output:
Index Scan using idx_orders_user_id on orders  
  (cost=0.29..8.31 rows=1000 width=120) 
  (actual time=0.015..1.234 rows=1050 loops=1)
Planning Time: 0.123 ms
Execution Time: 1.456 ms

-- Shows:
-- Estimated rows: 1000
-- Actual rows: 1050 (close — statistics are accurate)
-- Actual time: 1.456ms
```

### Reading EXPLAIN Output

#### Key Fields

| Field | Meaning |
|---|---|
| **Seq Scan** | Full table scan — no index used (slow for large tables) |
| **Index Scan** | Using index to find rows (good) |
| **Index Only Scan** | Query satisfied entirely from index (best) |
| **Bitmap Heap Scan** | Index + heap access, efficient for multi-condition queries |
| **Nested Loop** | Join algorithm — good for small result sets |
| **Hash Join** | Join algorithm — good for large result sets |
| **Merge Join** | Join algorithm — both sides sorted |
| **cost=0.00..18334** | Estimated cost (relative units, not milliseconds) |
| **rows=1000** | Estimated row count |
| **width=120** | Average row size in bytes |

### Optimization Examples

#### Example 1: Missing Index
```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'PENDING' ORDER BY created_at DESC LIMIT 20;

-- Before:
Seq Scan on orders  (cost=0.00..245678 rows=50000)
Sort  (cost=56789..57890 rows=50000)

-- Problem: Full scan + expensive sort

-- Fix: Add composite index
CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);

-- After:
Index Scan using idx_orders_status_created on orders  (cost=0.29..12.45 rows=20)

-- Result: 20,000× faster
```

#### Example 2: Unused Index
```sql
-- Query:
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';

-- Index exists on email but not used:
EXPLAIN → Seq Scan (Function call prevents index use)

-- Fix: Functional index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- After:
Index Scan using idx_users_email_lower
```

#### Example 3: Join Optimization
```sql
EXPLAIN ANALYZE
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'PENDING';

-- Before (no indexes):
Hash Join  (cost=5678..123456 rows=50000)
  -> Seq Scan on orders  (cost=0..67890 rows=50000)
  -> Hash  (cost=4567..4567 rows=100000)
       -> Seq Scan on users  (cost=0..4567 rows=100000)

-- Fix: Indexes on join and WHERE columns
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_users_id ON users(id);  -- PK, usually exists

-- After:
Nested Loop  (cost=0.29..789 rows=50)
  -> Index Scan using idx_orders_status on orders  (cost=0.29..345 rows=50)
  -> Index Scan using idx_users_pkey on users  (cost=0.29..8.31 rows=1)
```

### Optimization Workflow

```
1. Run EXPLAIN ANALYZE on slow query
2. Look for:
   - Seq Scan on large tables → add index
   - High estimated cost → optimize query or schema
   - Large difference between estimated/actual rows → update statistics
3. Add/modify indexes
4. EXPLAIN ANALYZE again → verify improvement
5. Monitor in production — explain doesn't account for load
```

### Update Statistics
```sql
-- If estimated rows ≠ actual rows (off by > 10×), update stats
ANALYZE orders;  -- PostgreSQL

UPDATE STATISTICS orders;  -- SQL Server
```

### Interview Answer
> "EXPLAIN shows the query execution plan — which indexes are used, join algorithms, and estimated cost. EXPLAIN ANALYZE adds actual execution time and row counts. I use it to identify missing indexes (Seq Scan on large tables), verify index usage after adding indexes, and diagnose slow queries.
>
> Common issues: Seq Scan instead of Index Scan (missing index), function calls preventing index use (LOWER(email) — need functional index), inefficient join algorithms (Hash Join on small result sets — wrong statistics).
>
> Workflow: EXPLAIN slow query → identify Seq Scans → add index → EXPLAIN again → verify Index Scan. Always run ANALYZE table after bulk inserts to update statistics — inaccurate statistics cause bad query plans. In production, I log queries > 1s and analyze them with EXPLAIN in staging."

---

## Q113. Primary Key vs Unique Key

### Side-by-Side Comparison

| | Primary Key | Unique Key |
|---|---|---|
| **Purpose** | Uniquely identify each row | Enforce uniqueness on column |
| **NULL allowed?** | ❌ No (NOT NULL enforced) | ✅ Yes (depends on DB, usually one NULL) |
| **Count per table** | 1 only | Multiple allowed |
| **Index** | Automatically creates clustered index | Automatically creates unique index |
| **Foreign Key reference** | ✅ Can be referenced | ❌ Usually can't be referenced |

### Code Examples

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,              -- Primary Key: auto-increment, unique, not null
    email VARCHAR(255) UNIQUE NOT NULL,    -- Unique Key: unique, explicitly not null
    phone VARCHAR(20) UNIQUE,              -- Unique Key: unique, NULL allowed
    username VARCHAR(50) UNIQUE NOT NULL
);

-- Foreign key references Primary Key
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)  -- References PK of users
);
```

### JPA Annotations
```java
@Entity
@Table(name = "users", 
    uniqueConstraints = @UniqueConstraint(columnNames = {"email"}))
public class User {
    
    @Id  // Primary Key
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)  // Unique + NOT NULL
    private String email;
    
    @Column(unique = true, nullable = false)
    private String username;
    
    @Column(unique = true)  // Unique, allows NULL
    private String phone;
}
```

### Composite Primary Key
```java
@Embeddable
@Data
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long productId;
}

@Entity
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;
    
    private int quantity;
    private BigDecimal price;
}

// Usage
OrderItemId id = new OrderItemId(1L, 100L);
OrderItem item = new OrderItem();
item.setId(id);
orderItemRepo.save(item);
```

### Unique Constraint on Multiple Columns
```java
@Entity
@Table(uniqueConstraints = {
    @UniqueConstraint(columnNames = {"user_id", "product_id"})
})
public class Wishlist {
    @Id
    @GeneratedValue
    private Long id;
    
    @Column(name = "user_id")
    private Long userId;
    
    @Column(name = "product_id")
    private Long productId;
    
    // Ensures: One user can add same product to wishlist only once
}
```

### How Indexes Relate

```sql
-- Primary Key → Clustered Index (default in most DBs)
CREATE TABLE orders (
    id BIGINT PRIMARY KEY  -- Automatically creates clustered index on id
);

-- Physical rows stored sorted by id on disk
-- Fast retrieval by id: O(log n)
-- Range queries fast: SELECT * WHERE id BETWEEN 100 AND 200

-- Unique Key → Non-Clustered Unique Index
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR UNIQUE  -- Automatically creates unique non-clustered index
);

-- Separate index structure, points to PK
-- Fast lookup: SELECT * WHERE email = 'john@example.com'
-- Enforces uniqueness at DB level
```

### NULL Behavior Differences

```sql
-- Unique Key allows NULLs (most DBs allow multiple NULLs)
INSERT INTO users (id, phone) VALUES (1, NULL);  -- ✅ OK
INSERT INTO users (id, phone) VALUES (2, NULL);  -- ✅ OK (multiple NULLs)
INSERT INTO users (id, phone) VALUES (3, '555-1234');  -- ✅ OK
INSERT INTO users (id, phone) VALUES (4, '555-1234');  -- ❌ Unique constraint violation

-- Primary Key never allows NULL
INSERT INTO users (id) VALUES (NULL);  -- ❌ NOT NULL constraint violation
```

### Interview Answer
> "Primary Key uniquely identifies each row, enforced as NOT NULL and unique, one per table. Automatically creates a clustered index — rows physically stored sorted by PK. Foreign keys reference the PK.
>
> Unique Key enforces uniqueness but allows NULL (usually multiple NULLs). Can have multiple unique keys per table. Automatically creates a non-clustered unique index. Used for business uniqueness: email, username, phone.
>
> Both create indexes automatically — PK creates clustered index (data storage order), unique key creates non-clustered index (separate structure). In JPA, @Id for PK, @Column(unique=true) for unique keys. For composite primary keys, use @EmbeddedId or @IdClass."

---

## Q114. SQL Interview Questions

### 1. Second Highest Salary

```sql
-- Approach 1: LIMIT OFFSET (MySQL, PostgreSQL)
SELECT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;

-- Approach 2: Subquery (All databases)
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Approach 3: Dense Rank (handles ties)
WITH RankedSalaries AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rank
    FROM employees
)
SELECT salary FROM RankedSalaries WHERE rank = 2;

-- To get Nth highest:
SELECT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET (N-1);
```

### 2. Customers with Zero Orders

```sql
-- Approach 1: LEFT JOIN + NULL check
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;

-- Approach 2: NOT EXISTS (more efficient for large datasets)
SELECT id, name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- Approach 3: NOT IN (avoid if orders.customer_id can be NULL)
SELECT id, name
FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL);
```

### 3. Duplicate Emails

```sql
SELECT email, COUNT(*) as count
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- To get all duplicate records (not just counts):
SELECT u.*
FROM users u
WHERE email IN (
    SELECT email
    FROM users
    GROUP BY email
    HAVING COUNT(*) > 1
);
```

### 4. Department with Highest Average Salary

```sql
SELECT d.name, AVG(e.salary) AS avg_salary
FROM departments d
JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name
ORDER BY avg_salary DESC
LIMIT 1;
```

### 5. Employees Earning More Than Manager

```sql
SELECT e.name AS employee_name, e.salary AS employee_salary,
       m.name AS manager_name, m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

### 6. Running Total (Cumulative Sum)

```sql
-- PostgreSQL, MySQL 8+, SQL Server (Window Function)
SELECT 
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;

-- Output:
-- order_date  | amount | running_total
-- 2026-01-01  | 100    | 100
-- 2026-01-02  | 150    | 250
-- 2026-01-03  | 75     | 325
```

### 7. Find Gaps in Sequence

```sql
-- Find missing IDs in orders table
SELECT id + 1 AS missing_start
FROM orders o1
WHERE NOT EXISTS (
    SELECT 1 FROM orders o2 WHERE o2.id = o1.id + 1
)
AND id < (SELECT MAX(id) FROM orders);
```

### 8. Top N per Group

```sql
-- Top 3 highest-paid employees per department
WITH RankedEmployees AS (
    SELECT 
        e.*,
        d.name AS dept_name,
        ROW_NUMBER() OVER (PARTITION BY e.department_id ORDER BY e.salary DESC) AS rank
    FROM employees e
    JOIN departments d ON e.department_id = d.id
)
SELECT * FROM RankedEmployees WHERE rank <= 3;
```

### 9. Join Optimization — Reduce Cartesian Product

```sql
-- ❌ BAD: Implicit join without ON → cartesian product
SELECT o.id, u.name, p.name
FROM orders o, users u, products p
WHERE o.user_id = u.id AND o.product_id = p.id;

-- ✅ GOOD: Explicit JOINs
SELECT o.id, u.name, p.name
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id;

-- Indexes needed:
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_product_id ON orders(product_id);
```

### 10. Self-Join — Find Pairs

```sql
-- Find pairs of employees in same department
SELECT e1.name AS employee1, e2.name AS employee2, e1.department_id
FROM employees e1
JOIN employees e2 ON e1.department_id = e2.department_id
WHERE e1.id < e2.id;  -- Avoid duplicates (A-B and B-A)
```

### Interview Answer Template
> "For second highest salary, I use `SELECT MAX(salary) WHERE salary < (SELECT MAX(salary))` — simple and works on all databases. For Nth highest, window functions with DENSE_RANK.
>
> Customers with zero orders: LEFT JOIN + WHERE IS NULL, or NOT EXISTS for better performance on large tables. NOT IN is risky if foreign key can be NULL.
>
> For join optimization, always use explicit JOIN syntax, ensure indexes exist on join columns, and use EXPLAIN to verify join algorithm (Nested Loop for small sets, Hash Join for large)."

---

## Q115. JPA Sorting & Pagination

### Pagination with Pageable

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Spring Data JPA automatically implements pagination
    Page<Order> findByUserId(Long userId, Pageable pageable);
    
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
}

@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepo;
    
    public Page<OrderDto> getOrders(Long userId, int page, int size, String sortBy, String sortDir) {
        // Create Sort object
        Sort sort = Sort.by(Sort.Direction.fromString(sortDir), sortBy);
        
        // Create Pageable
        Pageable pageable = PageRequest.of(page, size, sort);
        
        // Query with pagination
        Page<Order> orderPage = orderRepo.findByUserId(userId, pageable);
        
        // Map to DTO
        return orderPage.map(this::toDto);
    }
}

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    @Autowired
    private OrderService orderService;
    
    @GetMapping
    public ResponseEntity<Page<OrderDto>> getOrders(
            @RequestParam Long userId,
            @RequestParam(defaultValue = "0") int page,        // Zero-based
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDir) {
        
        Page<OrderDto> orders = orderService.getOrders(userId, page, size, sortBy, sortDir);
        return ResponseEntity.ok(orders);
    }
}
```

### Response Structure
```json
{
  "content": [
    { "id": 1, "total": 99.99, "status": "COMPLETED" },
    { "id": 2, "total": 149.99, "status": "PENDING" }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "sort": { "sorted": true, "unsorted": false },
    "offset": 0
  },
  "totalElements": 150,
  "totalPages": 8,
  "last": false,
  "first": true,
  "numberOfElements": 20,
  "size": 20,
  "number": 0
}
```

### Multi-Column Sorting
```java
// Sort by status ASC, then createdAt DESC
Sort sort = Sort.by(
    Sort.Order.asc("status"),
    Sort.Order.desc("createdAt")
);

Pageable pageable = PageRequest.of(page, size, sort);
Page<Order> orders = orderRepo.findAll(pageable);
```

### Dynamic Sorting from Request
```java
@GetMapping
public Page<OrderDto> getOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(required = false) String[] sort) {  // e.g., sort=createdAt,desc&sort=status,asc
    
    List<Sort.Order> orders = new ArrayList<>();
    
    if (sort != null) {
        for (String sortParam : sort) {
            String[] parts = sortParam.split(",");
            String field = parts[0];
            Sort.Direction direction = parts.length > 1 && parts[1].equalsIgnoreCase("asc")
                    ? Sort.Direction.ASC
                    : Sort.Direction.DESC;
            orders.add(new Sort.Order(direction, field));
        }
    }
    
    Pageable pageable = orders.isEmpty()
            ? PageRequest.of(page, size)
            : PageRequest.of(page, size, Sort.by(orders));
    
    return orderService.getOrders(pageable);
}
```

### Custom Query with Pagination
```java
@Query("SELECT o FROM Order o WHERE o.userId = :userId AND o.status = :status")
Page<Order> findByUserIdAndStatus(@Param("userId") Long userId,
                                   @Param("status") OrderStatus status,
                                   Pageable pageable);

// Supports sorting automatically
// Usage: findByUserIdAndStatus(123L, PENDING, PageRequest.of(0, 20, Sort.by("createdAt").descending()))
```

### Slice vs Page

```java
// Page: Includes total count (triggers COUNT query)
Page<Order> page = orderRepo.findAll(pageable);
page.getTotalElements();  // 1500
page.getTotalPages();     // 75 (if size=20)

// Slice: No total count (faster, no COUNT query)
Slice<Order> slice = orderRepo.findAll(pageable);
slice.hasNext();     // true/false
// slice.getTotalElements();  ❌ Not available

// Use Slice for infinite scroll (mobile apps) — don't need total count
```

### Native Query Pagination (PostgreSQL)
```sql
-- Manual pagination in native SQL
SELECT * FROM orders
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;  -- Page 0

-- Page 1 (skip first 20)
LIMIT 20 OFFSET 20;  -- Page 1

-- Page N
LIMIT 20 OFFSET (N * 20);
```

### Interview Answer
> "JPA pagination uses `Pageable` — passed to repository methods. Create with `PageRequest.of(page, size, Sort.by(...))`. Repository returns `Page<T>` with content, total elements, total pages, and metadata.
>
> For REST APIs, accept page/size/sort query params, create Pageable, pass to repository. Response includes pagination metadata for frontend to render page numbers. Use `Sort.by()` for single-field sorting, or `Sort.Order` list for multi-field.
>
> `Page` includes total count (triggers COUNT query) — use for paginated tables with page numbers. `Slice` omits count (faster) — use for infinite scroll where you only need hasNext(). Always default page size to 20-50, enforce max size (e.g., 100) to prevent abuse."

---

## Q116. RDBMS vs NoSQL

### Side-by-Side Comparison

| Feature | RDBMS (PostgreSQL, MySQL) | NoSQL (MongoDB, Cassandra, DynamoDB) |
|---|---|---|
| **Data Model** | Tables with fixed schema | Flexible: documents, key-value, wide-column |
| **Schema** | Strict, predefined | Schema-less or flexible |
| **Relationships** | Foreign keys, JOINs | Denormalization, embedded documents |
| **Transactions** | ACID (Atomicity, Consistency, Isolation, Durability) | Eventually consistent (some offer ACID) |
| **Scaling** | Vertical (bigger server) | Horizontal (more servers) |
| **Query Language** | SQL (standard) | Custom (e.g., MongoDB query language) |
| **Use Cases** | Financial, e-commerce, ERP | Real-time analytics, IoT, content management |
| **Consistency** | Strong consistency | Eventual consistency (tunable in some) |

### When to Use RDBMS

```
✅ Use RDBMS when:
  - ACID transactions required (banking, payments, inventory)
  - Complex queries with JOINs (reporting, analytics)
  - Data has clear relationships (users → orders → products)
  - Schema is stable and well-defined
  - Strong consistency is critical (read-after-write)

Examples:
  - E-commerce (orders, payments, inventory)
  - Banking (accounts, transactions, transfers)
  - ERP systems (structured business data)
  - User management (authentication, profiles)
```

### When to Use NoSQL

```
✅ Use NoSQL when:
  - Massive scale (billions of rows, petabytes)
  - Schema evolves frequently (product catalogs with varying attributes)
  - High write throughput (logs, IoT sensor data, clickstream)
  - Horizontal scaling required (distribute across data centers)
  - Eventual consistency acceptable (social media feeds, product reviews)
  - Denormalized data (avoid JOINs)

Examples:
  - Product Catalog (flexible attributes per product type)
  - Session Store (Redis — key-value, millisecond access)
  - Logs & Metrics (Elasticsearch — full-text search, time-series)
  - Social Feeds (Cassandra — write-heavy, distributed)
  - Real-time Analytics (DynamoDB — low-latency key-value)
```

### Hybrid Approach (Polyglot Persistence)

```java
// My E-Commerce Architecture

@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepo;  // PostgreSQL — ACID for orders/payments
    
    @Autowired
    private RedisTemplate<String, Object> redis;  // Redis — session + cache
    
    @Autowired
    private MongoTemplate mongo;  // MongoDB — product catalog (flexible schema)
    
    @Autowired
    private ElasticsearchTemplate elastic;  // Elasticsearch — product search
    
    public Order createOrder(CreateOrderRequest req) {
        // 1. RDBMS: Order & payment — ACID transaction
        Order order = orderRepo.save(new Order(req));
        
        // 2. NoSQL: Update product catalog in MongoDB
        mongo.updateFirst(
            Query.query(Criteria.where("_id").is(req.getProductId())),
            Update.update("stock", -req.getQuantity()),
            Product.class
        );
        
        // 3. Cache: Invalidate user's cart in Redis
        redis.delete("cart:" + req.getUserId());
        
        // 4. Search: Update Elasticsearch for real-time inventory
        elastic.update(...);
        
        return order;
    }
}
```

### PostgreSQL vs MongoDB Example

#### RDBMS (PostgreSQL)
```sql
-- Normalized schema
CREATE TABLE users (id BIGINT PRIMARY KEY, name VARCHAR, email VARCHAR);
CREATE TABLE orders (id BIGINT PRIMARY KEY, user_id BIGINT, total DECIMAL);
CREATE TABLE order_items (order_id BIGINT, product_id BIGINT, quantity INT);

-- Query with JOINs
SELECT u.name, o.id, o.total, oi.quantity, p.name AS product_name
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE u.id = 123;
```

#### NoSQL (MongoDB)
```javascript
// Denormalized document
{
  "_id": "order_12345",
  "user": {
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com"
  },
  "items": [
    { "productId": 1, "productName": "Laptop", "quantity": 1, "price": 999.99 },
    { "productId": 2, "productName": "Mouse", "quantity": 2, "price": 29.99 }
  ],
  "total": 1059.97,
  "status": "COMPLETED",
  "createdAt": ISODate("2026-03-13T10:00:00Z")
}

// Query (no JOINs, single document lookup)
db.orders.find({ "user.id": 123 });
```

### CAP Theorem Trade-offs

```
CAP Theorem: Pick 2 of 3
  - Consistency: All nodes see same data
  - Availability: System responds to all requests
  - Partition Tolerance: System works despite network partition

RDBMS: CP (Consistency + Partition Tolerance)
  → Sacrifices Availability — may refuse requests during partition

NoSQL (e.g., Cassandra): AP (Availability + Partition Tolerance)
  → Sacrifices Consistency — eventual consistency, stale reads possible

NoSQL (e.g., MongoDB with strong consistency): CP
  → Configurable
```

### Interview Answer
> "RDBMS for ACID transactions, complex JOINs, structured data with clear relationships — orders, payments, user management. Strong consistency, vertical scaling.
>
> NoSQL for massive scale, flexible schema, high write throughput, horizontal scaling — product catalogs (varying attributes), logs, IoT data, session stores. Eventual consistency acceptable.
>
> In my project, polyglot persistence: PostgreSQL for orders/payments (ACID critical), MongoDB for product catalog (flexible schema — electronics vs clothing have different attributes), Redis for sessions and caching (sub-millisecond access), Elasticsearch for product search (full-text, faceted filtering). Choose the right tool per use case, not one-size-fits-all."

---

## Q117. Stored Procedures

### What They Are
Pre-compiled SQL code stored in the database, executed with a single call. Can accept parameters, contain logic (IF/ELSE, loops), and return results.

### Example (PostgreSQL)
```sql
CREATE OR REPLACE FUNCTION process_order(
    p_user_id BIGINT,
    p_product_id BIGINT,
    p_quantity INT
) RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
    v_order_id BIGINT;
    v_stock INT;
BEGIN
    -- Check stock
    SELECT stock INTO v_stock FROM products WHERE id = p_product_id FOR UPDATE;
    
    IF v_stock < p_quantity THEN
        RAISE EXCEPTION 'Insufficient stock';
    END IF;
    
    -- Deduct stock
    UPDATE products SET stock = stock - p_quantity WHERE id = p_product_id;
    
    -- Create order
    INSERT INTO orders (user_id, product_id, quantity)
    VALUES (p_user_id, p_product_id, p_quantity)
    RETURNING id INTO v_order_id;
    
    RETURN v_order_id;
END;
$$;

-- Call from SQL
SELECT process_order(123, 456, 2);
```

### Calling from JPA
```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Procedure(name = "process_order")
    Long processOrder(@Param("p_user_id") Long userId,
                      @Param("p_product_id") Long productId,
                      @Param("p_quantity") int quantity);
}

// Usage
Long orderId = orderRepo.processOrder(123L, 456L, 2);
```

### When to Use Stored Procedures

```
✅ Use Stored Procedures when:
  - Complex multi-step DB operations (better in DB than app)
  - Batch processing (process millions of rows efficiently)
  - Performance critical (reduce network round-trips)
  - Legacy systems (already have business logic in DB)
  - Tight coupling to specific DB is acceptable

❌ Avoid Stored Procedures when:
  - Logic needs unit testing (hard to test DB code)
  - Need to switch databases (vendor lock-in)
  - Want version control with application code
  - Team lacks DB expertise
  - Microservices architecture (logic should be in services)
```

### Stored Procedure vs Application Logic

| | Stored Procedure | Application Logic |
|---|---|---|
| **Performance** | Faster (reduced network calls, compiled) | Slower (multiple round-trips) |
| **Testability** | Hard (requires DB setup) | Easy (unit tests with mocks) |
| **Version Control** | Separate from app code | Same repo as app |
| **Portability** | DB-specific (PostgreSQL ≠ MySQL syntax) | DB-agnostic (works with any DB) |
| **Debugging** | Difficult (limited tools) | Easy (IDE debugger) |
| **Deployment** | Separate deployment process | Part of app deployment |
| **Scalability** | Vertical (scale DB server) | Horizontal (scale app instances) |

### Practical Example — Batch Processing

```sql
-- Stored procedure: Archive old orders (runs faster in DB)
CREATE OR REPLACE PROCEDURE archive_old_orders()
LANGUAGE plpgsql
AS $$
BEGIN
    -- Move orders older than 1 year to archive table
    INSERT INTO orders_archive
    SELECT * FROM orders WHERE created_at < NOW() - INTERVAL '1 year';
    
    -- Delete from main table
    DELETE FROM orders WHERE created_at < NOW() - INTERVAL '1 year';
    
    -- Update stats
    ANALYZE orders;
END;
$$;

-- Scheduled job calls this monthly
CALL archive_old_orders();
```

```java
// Application approach: Fetch → Process → Save (slower, memory-intensive)
@Scheduled(cron = "0 0 2 1 * *")  // 2 AM on 1st of month
public void archiveOldOrders() {
    Instant cutoff = Instant.now().minus(365, ChronoUnit.DAYS);
    
    // Fetch millions of rows into memory — BAD for large datasets
    List<Order> oldOrders = orderRepo.findByCreatedAtBefore(cutoff);
    
    archiveRepo.saveAll(oldOrders);  // Network overhead
    orderRepo.deleteAll(oldOrders);   // Network overhead
}
// Stored procedure does this 10-100× faster, no network overhead
```

### Interview Answer
> "Stored procedures are pre-compiled SQL in the database. I use them sparingly — only for complex batch operations where doing it in the application would require millions of rows transferred over the network. Example: monthly archival of old orders — stored procedure moves data within the DB in seconds versus minutes via application logic.
>
> I avoid stored procedures for business logic — they're hard to unit test, tie you to a specific DB vendor, and make deployment complex. In microservices, business logic belongs in services, not the DB.
>
> Trade-off: Performance and reduced network calls vs testability and portability. Modern approach: keep logic in application, use stored procedures only for DB-intensive batch operations where network is the bottleneck."

---

## Q118. N+1 Query Problem in JPA

### The Problem
```java
// Fetch all orders
List<Order> orders = orderRepo.findAll();  // 1 query

// Access user for each order (lazy loading)
for (Order order : orders) {
    String userName = order.getUser().getName();  // N queries (one per order)
}

// Total queries: 1 + N
// For 100 orders: 101 queries!
```

### Why It Happens
```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)  // Default: LAZY
    private User user;
}

// Lazy loading: user not fetched until accessed
// Each access triggers a separate query
```

### Solutions

#### 1. JOIN FETCH (JPQL)
```java
// Fetch orders WITH users in ONE query
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUsers();

// Generated SQL:
// SELECT o.*, u.* FROM orders o
// INNER JOIN users u ON o.user_id = u.id

// Total queries: 1 (regardless of result count)
```

#### 2. @EntityGraph
```java
@EntityGraph(attributePaths = {"user", "items"})
List<Order> findAll();

// Or dynamic:
@EntityGraph(attributePaths = {"user"})
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatus(@Param("status") String status);
```

#### 3. DTO Projection
```java
// Fetch only needed fields (no entity, no lazy loading)
@Query("SELECT new com.app.dto.OrderDto(o.id, o.total, u.name) " +
       "FROM Order o JOIN o.user u")
List<OrderDto> findAllOrderDtos();

// No lazy loading issue — plain DTO, no proxies
```

#### 4. Batch Fetching
```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @BatchSize(size = 25)  // Hibernate-specific
    private User user;
}

// Fetches users in batches:
// Query 1: Orders (all)
// Query 2: Users WHERE id IN (1, 2, ..., 25)
// Query 3: Users WHERE id IN (26, 27, ..., 50)
// ...
// Total queries: 1 + CEIL(N / 25)
// For 100 orders: 1 + 4 = 5 queries (much better than 101)
```

#### 5. FetchType.EAGER (Use Sparingly)
```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.EAGER)  // Always fetched
    private User user;
}

// Always fetches user, even if not accessed
// Can cause performance issues if user has lazy collections
// General rule: Default to LAZY, fetch eagerly when needed with JOIN FETCH
```

### Real Example — Before & After

**Before (N+1 Problem):**
```java
@GetMapping("/orders")
public List<OrderDto> getOrders() {
    List<Order> orders = orderRepo.findAll();  // 1 query
    
    return orders.stream()
        .map(o -> new OrderDto(
            o.getId(),
            o.getUser().getName(),     // N queries (lazy load)
            o.getTotal()
        ))
        .collect(Collectors.toList());
}

// 1000 orders = 1001 queries
// Response time: 5 seconds
```

**After (JOIN FETCH):**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUsers();

@GetMapping("/orders")
public List<OrderDto> getOrders() {
    List<Order> orders = orderRepo.findAllWithUsers();  // 1 query
    
    return orders.stream()
        .map(o -> new OrderDto(
            o.getId(),
            o.getUser().getName(),     // No query, already loaded
            o.getTotal()
        ))
        .collect(Collectors.toList());
}

// 1000 orders = 1 query
// Response time: 50ms
```

### Detecting N+1 in Development
```yaml
# application-dev.yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

```
Output (N+1 detected):
Hibernate: select order0_.id, ... from orders order0_
Hibernate: select user0_.id, user0_.name from users user0_ where user0_.id=?
Hibernate: select user0_.id, user0_.name from users user0_ where user0_.id=?
Hibernate: select user0_.id, user0_.name from users user0_ where user0_.id=?
... (repeats N times)
```

### Interview Answer
> "N+1 happens with lazy loading — fetch N entities, each lazy access triggers a query. 100 orders = 101 queries if user is lazy-loaded per order. Solution: JOIN FETCH in JPQL fetches everything in one query, @EntityGraph for declarative fetch, or DTO projection to avoid entities entirely.
>
> I enable `show-sql` in dev to spot N+1 immediately — seeing repeated SELECT for same entity type is the telltale sign. For batch operations, @BatchSize reduces queries from N to CEIL(N/batch_size).
>
> Default to LAZY fetch, use JOIN FETCH explicitly when you know you'll access the relation. EAGER fetch can cause performance issues if the related entity has its own lazy collections — cascading eager loads."

---

## Q119. @EmbeddedId & Relationship Mapping

### Composite Primary Key with @EmbeddedId

```java
// Composite key class
@Embeddable
@Data
@NoArgsConstructor
@AllArgsConstructor
public class OrderItemId implements Serializable {
    private static final long serialVersionUID = 1L;
    
    @Column(name = "order_id")
    private Long orderId;
    
    @Column(name = "product_id")
    private Long productId;
    
    // equals() and hashCode() auto-generated by Lombok
}

// Entity with embedded composite key
@Entity
@Table(name = "order_items")
@Data
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;
    
    private int quantity;
    private BigDecimal price;
    
    @ManyToOne
    @MapsId("orderId")  // Maps orderId in composite key to Order entity
    @JoinColumn(name = "order_id")
    private Order order;
    
    @ManyToOne
    @MapsId("productId")  // Maps productId in composite key to Product entity
    @JoinColumn(name = "product_id")
    private Product product;
}

// Parent entity
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    public void addItem(Product product, int quantity, BigDecimal price) {
        OrderItem item = new OrderItem();
        item.setId(new OrderItemId(this.id, product.getId()));
        item.setOrder(this);
        item.setProduct(product);
        item.setQuantity(quantity);
        item.setPrice(price);
        items.add(item);
    }
}
```

### Usage
```java
// Create order with items
Order order = new Order();
order.setUserId(123L);
order.setTotal(BigDecimal.valueOf(99.99));
orderRepo.save(order);  // Generates order ID

// Add items
Product product1 = productRepo.findById(1L).orElseThrow();
Product product2 = productRepo.findById(2L).orElseThrow();

order.addItem(product1, 2, BigDecimal.valueOf(29.99));
order.addItem(product2, 1, BigDecimal.valueOf(39.99));

orderRepo.save(order);  // Saves order + items in one transaction
```

### Relationship Mapping Annotations

#### @ManyToOne
```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)  // FK column
    private User user;
}

// Many orders belong to one user
// Generated: FK constraint orders.user_id → users.id
```

#### @OneToMany (Bidirectional)
```java
@Entity
public class User {
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}

@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
}

// mappedBy = "user" → Order owns the relationship (has FK)
// User side is read-only
```

#### @OneToOne
```java
@Entity
public class User {
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id
    @GeneratedValue
    private Long id;
    
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    private User user;
}
```

#### @ManyToMany
```java
@Entity
public class Student {
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}

// Generates join table: student_course (student_id, course_id)
```

### Cascade Types
```java
@OneToMany(cascade = CascadeType.ALL)  // All operations cascade
private List<OrderItem> items;

// CascadeType.PERSIST → save parent = save children
// CascadeType.MERGE → merge parent = merge children
// CascadeType.REMOVE → delete parent = delete children
// CascadeType.REFRESH → refresh parent = refresh children
// CascadeType.ALL → all of above

// orphanRemoval = true → remove child from collection = delete from DB
```

### Interview Answer
> "@EmbeddedId for composite primary keys — create an @Embeddable class with key fields, embed it with @EmbeddedId. Use @MapsId to map composite key parts to relationships. Common for join tables like OrderItem with composite key (orderId + productId).
>
> @ManyToOne for FK side (many orders to one user), @OneToMany for the inverse (one user has many orders, `mappedBy` means read-only). @OneToOne for 1:1 relationships (user and profile). @ManyToMany generates join table automatically.
>
> CascadeType.ALL propagates save/delete to children. orphanRemoval = true deletes child when removed from collection. Always use LAZY fetch by default, JOIN FETCH when you need the relation to avoid N+1."

---

## Q120. Diagnosing & Optimizing Slow Date Queries

### Scenario
```sql
-- Slow query: Find orders in last 30 days
SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '30 days';

-- Takes 10 seconds on 10 million row table
```

### Step-by-Step Diagnosis

#### Step 1: EXPLAIN the Query
```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '30 days';

-- Output:
Seq Scan on orders  (cost=0.00..234567 rows=50000 actual time=8923.123)
  Filter: (created_at > (now() - '30 days'::interval))
  Rows Removed by Filter: 9950000

-- Problem: Sequential scan — no index on created_at
```

#### Step 2: Add Index on Date Column
```sql
CREATE INDEX idx_orders_created_at ON orders(created_at);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '30 days';

-- Output:
Index Scan using idx_orders_created_at on orders  (cost=0.29..1234 rows=50000 actual time=12.345)
  Index Cond: (created_at > (now() - '30 days'::interval))

-- Result: 10 seconds → 12ms (833× faster)
```

#### Step 3: Optimize for Common Queries (Composite Index)
```sql
-- If queries often filter by created_at + status
SELECT * FROM orders WHERE created_at > ? AND status = 'PENDING';

-- Composite index
CREATE INDEX idx_orders_created_status ON orders(created_at DESC, status);

-- Index used efficiently:
Index Scan using idx_orders_created_status on orders  (cost=0.29..456)
  Index Cond: ((created_at > ...) AND (status = 'PENDING'))
```

#### Step 4: Partial Index (If Querying Specific Subset)
```sql
-- Only recent orders queried frequently, old ones archived
CREATE INDEX idx_orders_recent ON orders(created_at)
WHERE created_at > NOW() - INTERVAL '90 days';

-- Smaller index, faster queries for recent data
```

#### Step 5: Partitioning (For Large Time-Series Tables)
```sql
-- Partition by month
CREATE TABLE orders (
    id BIGINT,
    user_id BIGINT,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Query automatically scans only relevant partitions
SELECT * FROM orders WHERE created_at > '2026-02-15';
-- Scans only orders_2026_02 and orders_2026_03 — faster
```

### Additional Optimizations

#### Covering Index (Avoid Table Access)
```sql
-- Query needs: id, user_id, total, created_at
SELECT id, user_id, total FROM orders WHERE created_at > ?;

-- Covering index includes all queried columns
CREATE INDEX idx_orders_covering ON orders(created_at, id, user_id, total);

-- Index-only scan — no table access needed
Index Only Scan using idx_orders_covering
```

#### Avoid Function Calls in WHERE (Prevents Index Use)
```sql
-- ❌ BAD: Index not used
SELECT * FROM orders WHERE DATE(created_at) = '2026-03-13';

-- ✅ GOOD: Index used
SELECT * FROM orders 
WHERE created_at >= '2026-03-13' AND created_at < '2026-03-14';
```

#### Update Statistics
```sql
-- If EXPLAIN shows wrong estimates (estimate=1000, actual=500000)
ANALYZE orders;

-- PostgreSQL auto-vacuum updates statistics, but manual for immediate update
```

### JPA Optimization
```java
// Indexed date queries
@Entity
@Table(indexes = @Index(name = "idx_created_at", columnList = "created_at"))
public class Order {
    @Column(nullable = false)
    private Instant createdAt;
}

// Query with date range
@Query("SELECT o FROM Order o WHERE o.createdAt > :startDate")
List<Order> findRecentOrders(@Param("startDate") Instant startDate);

// Usage
Instant thirtyDaysAgo = Instant.now().minus(30, ChronoUnit.DAYS);
List<Order> recentOrders = orderRepo.findRecentOrders(thirtyDaysAgo);
```

### Monitoring & Alerting
```yaml
# Log slow queries (PostgreSQL)
# postgresql.conf
log_min_duration_statement = 1000  # Log queries > 1 second

# application.yml
spring:
  jpa:
    properties:
      hibernate:
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 1000  # Hibernate slow query log
```

### Interview Answer
> "For slow date queries, I start with EXPLAIN ANALYZE — usually reveals missing index or sequential scan. Add B-Tree index on the date column, verify with EXPLAIN. If queries filter by date + another column (like status), use composite index with date first.
>
> For large time-series tables (orders, logs), use partitioning by month or year — DB scans only relevant partitions. Partial indexes work for recent-data-only queries — indexes last 90 days, smaller and faster.
>
> Avoid function calls in WHERE like DATE(created_at) — breaks index. Use range: `>= '2026-03-13' AND < '2026-03-14'`. Update statistics with ANALYZE if estimates are way off. Log queries > 1s in production, analyze with EXPLAIN in staging, add indexes conservatively."

---

## Quick Revision Card — Section 12

| Topic | Key Point |
|---|---|
| **Index Types** | B-Tree (default, range), Hash (equality only), Composite (left-to-right), Unique, Partial |
| **Non-Indexed Query** | Full table scan O(n). With index: O(log n). 10 million rows: 10s → 10ms. |
| **Frequent Deletes** | Index bloat — dead entries remain. Fix: REINDEX, VACUUM, soft delete, partitioning. |
| **Index Trade-offs** | Faster SELECTs, slower INSERTs/UPDATEs. Don't index low-cardinality or all columns. |
| **EXPLAIN** | Shows execution plan. Look for Seq Scan (bad), Index Scan (good). Use EXPLAIN ANALYZE for actual times. |
| **PK vs Unique** | PK: one per table, NOT NULL, clustered index. Unique: multiple, allows NULL, non-clustered. |
| **Second Highest Salary** | `MAX(salary) WHERE salary < (SELECT MAX(salary))` or DENSE_RANK window function. |
| **Zero Orders** | LEFT JOIN + IS NULL or NOT EXISTS. NOT EXISTS faster for large tables. |
| **JPA Pagination** | PageRequest.of(page, size, Sort.by(...)). Returns Page<T> with metadata. Slice for infinite scroll. |
| **RDBMS vs NoSQL** | RDBMS: ACID, JOINs, structured. NoSQL: scale, flexible schema, eventual consistency. Polyglot persistence. |
| **Stored Procedures** | Use for batch operations. Avoid for business logic (hard to test, vendor lock-in). |
| **N+1 Problem** | Lazy load triggers N queries. Fix: JOIN FETCH, @EntityGraph, DTO projection, @BatchSize. |
| **@EmbeddedId** | Composite key class @Embeddable. Use @MapsId to link to relations. |
| **Slow Date Query** | Add index on date column. Composite index if + status. Partitioning for time-series. Avoid DATE() in WHERE. |

---

**End of Section 12**

---

## Additional Deep-Dive (Q109-Q120)

### Query Performance Triage Order

1. Validate query plan (`EXPLAIN ANALYZE`) before touching indexes.
2. Remove ORM-generated N+1 patterns using fetch strategy and projections.
3. Add or refine indexes only after confirming predicate/selectivity pattern.
4. Re-measure using p95/p99 query latency and connection pool pressure.

### Real Project Usage

- Mature teams maintain a "top 10 slow queries" dashboard and review it in every performance sprint.
- Index changes are rolled out with migration scripts and rollback plans, not ad-hoc DB console edits.

---

## JPA Lifecycle and Fetching Deep-Dive

### Entity Lifecycle States

| State | Meaning |
|---|---|
| `transient` | New object, not managed by persistence context |
| `managed` (persistent) | Attached to current persistence context; changes auto-tracked |
| `detached` | Previously managed, now outside persistence context |
| `removed` | Marked for deletion; SQL delete at flush/commit |

```java
@Transactional
public void lifecycleDemo(EntityManager em) {
  UserEntity user = new UserEntity();        // transient
  user.setName("Alice");

  em.persist(user);                          // managed
  user.setName("Alice-updated");            // dirty checking -> update at flush

  em.detach(user);                           // detached
  user.setName("Detached-change");          // not persisted automatically

  UserEntity merged = em.merge(user);        // back to managed copy
  em.remove(merged);                         // removed
}
```

### Lazy vs Eager Loading (Practical Rules)

- Default strategy recommendation: prefer `LAZY` for associations, fetch explicitly per use-case.
- `EAGER` often causes over-fetching and hidden query chains.

```java
@Entity
public class OrderEntity {
  @Id
  private Long id;

  @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
  private List<OrderItemEntity> items;
}
```

### Solving N+1 with `@EntityGraph`

```java
public interface OrderRepository extends JpaRepository<OrderEntity, Long> {

  @EntityGraph(attributePaths = {"items"})
  Optional<OrderEntity> findWithItemsById(Long id);
}
```

### Session Boundary + `@Transactional`

Interview rule: lazy proxies must be accessed inside transaction/session boundary, or map to DTO inside transactional service.

### Interview Drill

Question: Why is EAGER not a default recommendation?

Strong answer:
- It couples entity retrieval to maximum graph loading.
- It hides expensive joins and increases payload/memory.
- LAZY + explicit fetch plans (`JOIN FETCH`, `EntityGraph`, projection) is more controllable.

---

## Q121. SQL Isolation Levels — Choosing Between Consistency and Performance

### Concept
SQL isolation levels control how strictly the database locks data and handles concurrent access. Higher isolation prevents data anomalies (dirty reads, lost updates, phantom reads) but reduces concurrency and increases lock contention. Lower isolation is faster but riskier. Production systems must balance performance, data correctness, and business requirements.

### Simple Explanation
Imagine a concert ticket counter:
- **Read Uncommitted** (chaos): Seller counts remaining tickets while someone at another register is still adding more. You might see an incomplete count.
- **Read Committed** (safe snapshot): You only see ticket counts after someone has finished their transaction. No chaos, but counts might differ second-by-second.
- **Repeatable Read** (frozen snapshot): You see one consistent snapshot of tickets for your entire purchase. Another buyer can't undercut your price mid-transaction.
- **Serializable** (one at a time): Only one buyer in the shop at a time. Slowest but completely safe.

### How It Works — The Four Anomalies Isolation Prevents

| Anomaly | Definition | Example | Risk Level |
|---------|-----------|---------|------------|
| **Dirty Read** | Read uncommitted changes from other txns | Read balance before transfer commit completes; shows false amount | HIGH — financial corruption |
| **Non-Repeatable Read** | Same SELECT returns different rows mid-txn | Query customer address twice in same txn; address changed between queries | MEDIUM — incomplete updates |
| **Phantom Read** | Rows inserted/deleted by concurrent txn between SELECTs | COUNT(*) orders = 5, then 6, without seeing the insert | MEDIUM — report inconsistency |
| **Lost Update** | Two txns overwrite each other's changes | Both txns read balance=100, increment it, write back 101 (not 102) | HIGH — data loss |

### Isolation Level Standards (SQL/PostgreSQL/MySQL)

```
┌───────────────────┬──────────┬──────────┬──────────┬────────────┐
│ Isolation Level   │ Dirty    │ Non-Rep  │ Phantom  │ Lock       │
│                   │ Read     │ Read     │ Read     │ Severity   │
├───────────────────┼──────────┼──────────┼──────────┼────────────┤
│ READ UNCOMMITTED  │   ✓      │   ✓      │   ✓      │ None       │
│ READ COMMITTED    │   ✗      │   ✓      │   ✓      │ Row locks  │
│ REPEATABLE READ   │   ✗      │   ✗      │   ✓      │ Range lock │
│ SERIALIZABLE      │   ✗      │   ✗      │   ✗      │ Full txn   │
└───────────────────┴──────────┴──────────┴──────────┴────────────┘
```

### Code — Production Scenarios

#### ❌ Problem 1: Lost Update with READ COMMITTED (Default MySQL/PostgreSQL)

```java
// Scenario: Two requests increment account balance concurrently
@Transactional(isolation = Isolation.READ_COMMITTED)  // Default
public void incrementBalance(Long accountId, BigDecimal amount) {
    AccountEntity account = repo.findById(accountId).get();  // Read: balance = 100
    Thread.sleep(1000);  // Simulate slow processing
    account.setBalance(account.getBalance().add(amount));     // +50 = 150
    repo.save(account);  // Write: 150
    // If another txn reads->modifies->writes same account in parallel,
    // second write overwrites first. Result: 100 + 50 OR 100 + 50 (not 100 + 50 + 50)
}

// Both threads see balance=100, both compute +50, both write 150. Lost the second increment!
```

**Why it happened:** READ COMMITTED doesn't hold read locks. After first txn reads, another txn can read the same row before first txn commits.

#### ✅ Solution 1: Use REPEATABLE_READ or Locking

```java
// Option A: REPEATABLE_READ (PostgreSQL recommendation)
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void incrementBalance(Long accountId, BigDecimal amount) {
    AccountEntity account = repo.findById(accountId).get();
    account.setBalance(account.getBalance().add(amount));
    repo.save(account);
    // Holds read lock; concurrent txn waits until this commits/rolls back.
    // Result: Serialized access. Both increments land: 100 -> 150 -> 200.
}

// Option B: READ_COMMITTED + Explicit Lock (MySQL)
@Transactional(isolation = Isolation.READ_COMMITTED)
public void incrementBalance(Long accountId, BigDecimal amount) {
    // SELECT ... FOR UPDATE (pessimistic lock)
    AccountEntity account = repo.findByIdWithLock(accountId);  // Acquires write lock
    account.setBalance(account.getBalance().add(amount));
    repo.save(account);
    // Concurrent txn waits for lock release.
}

public interface AccountRepository extends JpaRepository<AccountEntity, Long> {
    @Query("SELECT a FROM AccountEntity a WHERE a.id = :id")
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<AccountEntity> findByIdWithLock(@Param("id") Long id);
}

// Option C: Optimistic Lock (version-based, for low contention)
@Entity
public class AccountEntity {
    @Id
    private Long id;
    private BigDecimal balance;
    
    @Version  // Auto-incremented on each update
    private Long version;
}

@Transactional(isolation = Isolation.READ_COMMITTED)
public void incrementBalance(Long accountId, BigDecimal amount) {
    AccountEntity account = repo.findById(accountId).get();
    account.setBalance(account.getBalance().add(amount));
    try {
        repo.save(account);
    } catch (OptimisticLockingFailureException e) {
        // Version mismatch — retry or fail gracefully
    }
}
```

#### ❌ Problem 2: Phantom Read with REPEATABLE_READ

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void reportActiveOrders() throws InterruptedException {
    List<OrderEntity> orders1 = repo.findByStatus("ACTIVE");  // SELECT COUNT: 5
    System.out.println("Active orders: " + orders1.size());     // Prints: 5
    
    Thread.sleep(2000);  // Concurrent INSERT happens here by another txn
    
    List<OrderEntity> orders2 = repo.findByStatus("ACTIVE");  // SELECT COUNT: 6  ← PHANTOM!
    System.out.println("Active orders now: " + orders2.size()); // Prints: 6
    // REPEATABLE_READ prevents changes to existing rows but not new inserts.
}
```

#### ✅ Solution 2: Use SERIALIZABLE or Range Locks

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void reportActiveOrders() throws InterruptedException {
    List<OrderEntity> orders1 = repo.findByStatus("ACTIVE");  // SELECT: 5
    Thread.sleep(2000);
    List<OrderEntity> orders2 = repo.findByStatus("ACTIVE");  // SELECT: 5 (locked range, no insert allowed)
    // Concurrent INSERT waits for this txn to commit before executing.
}

// PostgreSQL example: Named range locks
// SELECT * FROM orders WHERE status='ACTIVE' FOR UPDATE;
// This prevents phantom reads by locking the range matching the predicate.
```

#### ❌ Problem 3: Dirty Read with READ_UNCOMMITTED

```java
@Transactional(isolation = Isolation.READ_UNCOMMITTED)  // Very risky
public BigDecimal checkBalance(Long accountId) {
    AccountEntity account = repo.findById(accountId).get();
    return account.getBalance();  // May return uncommitted balance from concurrent txn
}

// Concurrent txn: BEGIN; UPDATE account SET balance = 9999; -- ROLLBACK;
// checkBalance() sees balance=9999 temporarily, then txn rolls back.
// balance is invalid. Financial disaster.
```

#### ✅ Solution 3: Use READ_COMMITTED Minimum

```java
@Transactional(isolation = Isolation.READ_COMMITTED)  // Safe default
public BigDecimal checkBalance(Long accountId) {
    AccountEntity account = repo.findById(accountId).get();
    return account.getBalance();  // Only sees committed values
}
```

### Spring Boot Configuration

```yaml
# application.yml — PostgreSQL default REPEATABLE_READ
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true
```

```java
// Per-method isolation override
@Transactional(isolation = Isolation.REPEATABLE_READ)  // PostgreSQL-aligned
public void criticalFinancialUpdate(Order order) { ... }

@Transactional(isolation = Isolation.READ_COMMITTED)   // Fast reads, eventual consistency OK
public Order getOrderForDisplay(Long id) { ... }
```

### Real Project Usage

**Scenario: E-commerce order processing.**
- **Order checkout** → SERIALIZABLE: Prevent overselling. Inventory can't drop below 0.
- **Order status API** → READ_COMMITTED: Users see consistent snapshots, but eventual consistency is acceptable. Fast reads.
- **Inventory report** → REPEATABLE_READ: Consistent report per txn. Prevents mid-report inventory changes confusing auditors.
- **Promotional price updates** → REPEATABLE_READ + version check: Prevent discounting errors when two admins edit simultaneously.

**Contention diagnosis:**
```sql
-- PostgreSQL: Check for lock waits
SELECT pid, query, wait_event_type, wait_event FROM pg_stat_activity WHERE wait_event_type IS NOT NULL;

-- MySQL: Check locks
SHOW PROCESSLIST; -- Look for 'Waiting for table metadata lock'
```

### Interview Answer

> "Isolation levels are a sliding scale between consistency and concurrency. READ UNCOMMITTED is dangerous — it allows dirty reads, financial corruption. Almost never used in production.
>
> READ COMMITTED is the safe default for most systems. Transactions see only committed data. Prevents dirty reads. It allows non-repeatable reads and phantoms, but for most read-heavy systems that's acceptable.
>
> REPEATABLE READ, PostgreSQL's default, is MVCC snapshot-based: each transaction reads from a consistent snapshot, which prevents dirty and non-repeatable reads without locking every read path. For strict cross-transaction serial behavior, use SERIALIZABLE.
>
> SERIALIZABLE is slowest. It locks everything — existing rows, gaps, entire ranges. True serialization. Use only for critical operations like payment processing or inventory decrement where even phantom anomalies could cause money loss.
>
> Concretely: Financial operations → SERIALIZABLE. Order status reads → READ_COMMITTED. Periodic reports → REPEATABLE_READ. The key is matching your isolation choice to the risk, not defaulting to SERIALIZABLE and paying the performance penalty everywhere.
>
> I've seen teams use REPEATABLE_READ + pessimistic locks for high-contention scenarios. That gives you the best-of-both: consistency benefits + explicit control over lock points."

**Follow-up likely:** "How would you detect a lost update in production?" → Query logs for concurrent updates to same row, check app metrics for failed optimistic locks, or simulate with load test.

---

## Quick Revision Card — Q121 Isolation Levels

| Level | Dirty Read | Non-Rep | Phantom | Lock Strategy | When to Use |
|-------|-----------|---------|---------|---------------|-----------|
| READ UNCOMMITTED | ✓ | ✓ | ✓ | None | Only external reporting DBs (unsafe) |
| READ COMMITTED | ✗ | ✓ | ✓ | Row | Safe default for most apps |
| REPEATABLE READ | ✗ | ✗ | DB-specific | Snapshot/MVCC | PostgreSQL default; strong consistency reads |
| SERIALIZABLE | ✗ | ✗ | ✗ | Full Txn | Financial, payment, inventory |
| **Pessimistic Lock** | ✗ | ✗ | ✗ (range) | Explicit FOR UPDATE | High contention, critical sections |
| **Optimistic Lock** | ✗ | ✗ | ✗ (app-level) | Version field | Low contention, distributed |

---
