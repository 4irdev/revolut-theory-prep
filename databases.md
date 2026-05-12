# Databases

## ACID

Every reliable database transaction must satisfy four properties:

- **Atomicity** — the transaction is all-or-nothing. If any step fails, the whole transaction is rolled back as if it never happened.
- **Consistency** — a transaction can only bring the database from one valid state to another. It can never leave data in a broken or half-updated state.
- **Isolation** — concurrent transactions don't interfere with each other. Each transaction sees the database as if it were running alone.
- **Durability** — once a transaction is committed, it stays committed. Even a crash or power loss won't undo it.

---

## Read Phenomena

### Dirty Read

Transaction T2 reads data modified by T1 that hasn't been committed yet. If T1 rolls back, T2 has relied on data that never persisted.

> T1 updates balance to 1200 → T2 reads 1200 → T1 rolls back → T2 has invalid data

### Non-Repeatable Read

A transaction reads the same row twice and gets different results because another transaction modified and committed that row in between.

> T1 reads price = 500 → T2 updates price to 600 and commits → T-1 reads price = 600

### Phantom Read

A transaction runs the same query twice and gets different row sets because another transaction inserted or deleted matching rows in between.

> T1 queries orders > 1000, gets 5 rows → T2 inserts a qualifying order → T1 re-queries, gets 6 rows

[Article Source](https://dev.to/abhivyaktii/understanding-read-phenomena-in-databases-50mk)

---

## Isolation Levels

| Level            | Dirty Read | Non-Repeatable Read | Phantom Read |
| ---------------- | ---------- | ------------------- | ------------ |
| Read Uncommitted | Possible   | Possible            | Possible     |
| Read Committed   | Prevented  | Possible            | Possible     |
| Repeatable Read  | Prevented  | Prevented           | Possible     |
| Serializable     | Prevented  | Prevented           | Prevented    |

### When to use each level

- **Read Uncommitted** — avoid unless consistency doesn't matter at all. Only reasonable for rough analytics or reporting where speed is the only priority.
- **Read Committed** — the default for most applications. Good balance of safety and performance; works for general business logic where occasional stale reads are acceptable.
- **Repeatable Read** — use when you notice data inconsistencies within a transaction, or when data stability across multiple reads inside one transaction is required.
- **Serializable** — reserve for critical operations like financial transactions. Fully safe but slow; can cause deadlocks and forces transaction retries under contention.

---

## Indexes

An **index** is a separate sorted data structure that lets the database find rows without scanning the whole table — like the alphabetical index at the end of a book. Default implementation in most relational DBs is a **B+ tree**: a balanced tree where all data lives in the leaves and leaves are linked together for fast range scans.

Key concepts:

- **Clustered index** — defines the physical order of rows on disk. Only **one per table**. In MySQL/InnoDB the primary key is always the clustered index.
- **Non-clustered index** — separate structure pointing to row locations. Many allowed.
- **Composite index** `(a, b, c)` — useful for queries filtering by `a`, `a+b`, or `a+b+c`. **Order matters**: a query filtering only by `b` won't use it.
- **Covering index** — contains all columns needed by the query, so DB doesn't need to read the row at all.
- **When indexes are not used**: function on the column (`WHERE LOWER(email) = ...`), leading wildcard (`LIKE '%foo'`), implicit type cast, very small tables.
- **Cost**: every `INSERT`/`UPDATE`/`DELETE` must update all indexes on the table → over-indexing kills write performance.

Index types by underlying data structure:

- **B-Tree / B+ Tree** — balanced tree, sorted. Default for almost everything. Best for: equality (`=`, `IN`), ranges (`<`, `>`, `BETWEEN`), prefix `LIKE 'foo%'`, `ORDER BY`. The safe default — pick this unless you have a reason not to.
- **Hash** — hash table, O(1) lookup. Best for: pure equality on high-cardinality columns (sessions, cache keys). No range queries, no sorting. In Postgres rarely beats B-tree in practice.
- **GIN (Generalized Inverted Index)** — inverted index: maps each element → list of rows containing it. Best for: full-text search (`tsvector`), `jsonb` containment (`@>`), arrays, trigram `LIKE '%foo%'`. Slow writes, fast reads.
- **GiST (Generalized Search Tree)** — extensible tree for "nearest" / overlap queries. Best for: geometry (PostGIS), ranges, k-NN ("find 10 closest points"), exclusion constraints (no overlapping bookings).
- **SP-GiST** — space-partitioned GiST (quad-trees, radix trees). Best for: non-balanced data like IP addresses, phone numbers, hierarchies where keys cluster unevenly.
- **BRIN (Block Range Index)** — stores min/max per block range, tiny on disk. Best for: huge append-only tables where data is naturally ordered (timestamps in logs, event tables). Rough but cheap; bad if data is shuffled.
- **Bitmap** — bit per row per value (Oracle/DWH; Postgres builds them on the fly). Best for: low-cardinality columns (gender, status) and combining many filters with AND/OR in analytics. Bad for OLTP — row-level locks become table-level.
- **R-Tree** — bounding-box tree for spatial data. Best for: 2D/3D geometry, "find all shapes intersecting this rectangle". In Postgres exposed via GiST.
- **Full-text index** — tokenized + normalized text (stemming, stop-words). Best for: search engines over articles/comments. In Postgres = GIN over `tsvector`; MySQL has native `FULLTEXT`.
- **Partial index** — B-tree built only over rows matching a `WHERE`. Best for: skewed data ("only active users", "only unprocessed rows"). Smaller and faster than a full index.
- **Expression / Functional index** — index on `f(column)` instead of the column. Best for: case-insensitive search (`LOWER(email)`), `date_trunc('day', created_at)`, computed values.
- **Unique index** — B-tree that rejects duplicates. Best for: enforcing uniqueness (emails, slugs); also speeds up lookups by that key.

Read `EXPLAIN` / `EXPLAIN ANALYZE` to verify the index is actually used.

**Articles:**

- [How B-Tree Indexes Work in SQL Databases: An Intuitive Guide — dev.to](https://dev.to/ashraf_shaik_ee7ab42b0eb3/how-b-tree-indexes-work-in-sql-databases-an-intuitive-guide-3e71)
- [Database Indexes Explained: B-Trees, Composite Indexes and When They Hurt Performance — dev.to](https://dev.to/young_gao/database-indexes-explained-b-trees-composite-indexes-and-when-they-hurt-performance-3gac)
- [Database Indexing Strategies: B-Tree, Hash, and Specialized Indexes — Medium](https://medium.com/@artemkhrenov/database-indexing-strategies-b-tree-hash-and-specialized-indexes-explained-95a3e5e3b632)

---

## N+1 Query Problem

A classic backend anti-pattern: you run **1** query to fetch a list of N parents, then **N** more queries (one per parent) to fetch related children — total **N+1** roundtrips.

```python
# BAD — 1 + N queries
users = session.query(User).all()
for user in users:
    print(user.orders)   # lazy load → triggers a query each time

# GOOD — 1 query with JOIN
users = session.query(User).options(joinedload(User.orders)).all()

# GOOD — 2 queries with IN
users = session.query(User).options(selectinload(User.orders)).all()
```

Why it slips through:

- Doesn't show in unit tests (small data → fast).
- ORMs hide it behind a clean Python API.
- Surfaces only in production under real volume.

How to detect: enable SQL logging in dev, count queries per request, use APM tools (Sentry, Datadog) — they flag N+1 patterns explicitly.

**Articles:**

- [The N+1 Query Problem Explained: SQLAlchemy Performance Pitfalls and Fixes — Substack](https://abhishekrath.substack.com/p/the-n1-query-problem)
- [The N+1 Database Query Problem: A Simple Explanation — Medium](https://medium.com/databases-in-simple-words/the-n-1-database-query-problem-a-simple-explanation-and-solutions-ef11751aef8a)
- [What is the N+1 Query Problem and How to Solve It — PlanetScale](https://planetscale.com/blog/what-is-n-1-query-problem-and-how-to-solve-it)

---

## JOINs

| Type                | Returns                                                       |
| ------------------- | ------------------------------------------------------------- |
| `INNER JOIN`      | Only rows matching in both tables                             |
| `LEFT JOIN`       | All rows from left + matched from right (NULLs for unmatched) |
| `RIGHT JOIN`      | All rows from right + matched from left                       |
| `FULL OUTER JOIN` | All rows from both, NULLs where no match                      |
| `CROSS JOIN`      | Cartesian product (every left row × every right row)         |
| `SELF JOIN`       | Table joined with itself (e.g. employee → manager)           |

**The most common production bug**: `LEFT JOIN` becomes effectively `INNER JOIN` when you put a condition on the right table in `WHERE`:

```sql
-- BAD — drops users who have no orders (NULL in orders.status fails the WHERE)
SELECT u.*, o.status
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.status = 'paid';

-- GOOD — condition belongs in ON
SELECT u.*, o.status
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.status = 'paid';
```

Rule: filters on the **left** table go in `WHERE`; filters on the **right** table of an outer join go in `ON`.

**Articles:**

- [SQL Joins Explained: INNER, LEFT, RIGHT, FULL, and More — dev.to](https://dev.to/_d7eb1c1703182e3ce1782/sql-joins-explained-inner-left-right-full-and-more-5a9e)
- [SQL Joins Demystified: LEFT, RIGHT, INNER, FULL Explained with Tables — dev.to](https://dev.to/crit3cal/sql-joins-demystified-left-right-inner-full-explained-with-tables-e9m)
- [SQL Join types explained visually — Atlassian](https://www.atlassian.com/data/sql/sql-join-types-explained-visually)

---

## Sharding

**Horizontal partitioning** of data across multiple database servers. Each server (shard) holds a subset of rows. Used when a single machine can't handle data volume or write throughput.

Sharding strategies:

- **Range-based** — `user_id 1–1M → shard 1`, `1M–2M → shard 2`. Simple, but risk of **hot shards** (skewed distribution).
- **Hash-based** — `hash(key) % N`. Even distribution, but resharding is painful (every key moves).
- **Consistent hashing** — only ~`1/N` of keys move when adding a node. Used by Cassandra, DynamoDB.
- **Directory-based** — lookup table maps key → shard. Most flexible, but lookup table is a bottleneck.

The hardest decisions:

- **Choosing shard key** — must match access patterns. If most queries don't include the shard key, you'll do **scatter-gather** (query all shards, merge results) — kills the benefit of sharding.
- **Cross-shard transactions** — typically not supported. Either avoid them or use sagas/2PC.
- **Rebalancing** — adding/removing shards.

**Articles:**

- [Database Sharding for System Design Interview — dev.to](https://dev.to/somadevtoo/database-sharding-for-system-design-interview-1k6b)
- [How Sharding Works — Medium (Jeeyoung Kim)](https://medium.com/@jeeyoungk/how-sharding-works-b4dec46b3f6)
- [Database Sharding Explained: How to Split Data to Scale Faster — Medium](https://datascienceafrica.medium.com/database-sharding-explained-how-to-split-data-to-scale-faster-4d455af4b347)

---

## Locking & Deadlocks

Locks coordinate concurrent transactions. 

Granularity: **row-level** (most common, cheapest concurrency) → **page** → **table**.

Lock modes (simplified):

- **Shared (S)** — for reads. Multiple readers OK, blocks writers.
- **Exclusive (X)** — for writes. Blocks everything.
- **Update (U)** - for updates to avoid deadlocks, convert to X when ready to write

**Deadlock** = two transactions each hold a lock the other needs. Coffman conditions (all four required):

1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait

```text
T1: lock(A) → wants lock(B)
T2: lock(B) → wants lock(A)   ← deadlock
```

DB engines detect deadlocks and **kill one transaction** as victim — your app must retry.

Most common senior-level mistakes:

- **Different lock order in different services** — Service X touches `accounts` then `transfers`, Service Y does the reverse. Solution: enforce a single canonical order.
- **Long transactions doing external IO** — opening a transaction, updating a row, then calling an HTTP API or sending an email before `COMMIT`. Holds locks for seconds, blocks everyone.

### Optimistic vs Pessimistic vs Externalized

**Pessimistic locking** — assume conflicts are likely; lock the resource up front, do the work, release. In SQL: `SELECT ... FOR UPDATE`. Safe but blocks other writers; risk of deadlocks; long transactions hurt throughput.

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 42 FOR UPDATE;  -- row locked
UPDATE accounts SET balance = balance - 100 WHERE id = 42;
COMMIT;
```

**Optimistic locking** — assume conflicts are rare; don't lock. Read the row with a version, write with `WHERE version = ?`; if no rows updated, someone else won — retry the whole operation. No deadlocks, high concurrency, but you must handle retries.

```sql
-- read
SELECT balance, version FROM accounts WHERE id = 42;
-- Returns: id=42, stock=5, version=3
-- write
UPDATE accounts SET balance = ?, version = version + 1
WHERE id = 42 AND version = 3;  -- 0 rows affected → retry
```

Use **pessimistic** when conflicts are frequent and retry is expensive (e.g. inventory decrement under heavy contention). Use **optimistic** for everything else — most CRUD apps.

**Externalized (distributed) locking** — when the resource isn't a single DB row: e.g. "only one worker should send this email", "only one job should run cron task X". Lock lives outside the DB, in Redis / Zookeeper / etcd.

- **Redis SETNX with TTL** — simple but unsafe under network partitions or GC pauses (Martin Kleppmann's classic critique).
- **Redlock** — Redis algorithm using N independent masters; client takes lock on majority. Improved safety, still controversial.
- **Zookeeper / etcd** — strongly-consistent coordination services; safer but heavier infra.

Rule: **prefer DB-level locking** when the data lives in a DB. Reach for distributed locks only when truly cross-resource.

**Articles:**

- [Locks and Deadlocks in SQL Server: Understand, Detect, and Prevent — Medium](https://rafaelrampineli.medium.com/locks-and-deadlocks-in-sql-server-understand-detect-and-prevent-0b74fbb09bd2)
- [A Beginner&#39;s Guide to Database Deadlock — Vlad Mihalcea](https://vladmihalcea.com/database-deadlock/)
- [Optimistic vs Pessimistic Locking: What Nobody Tells You Until You&#39;ve Burnt in Production — Medium](https://medium.com/@liberatoreanita/optimistic-vs-pessimistic-locking-what-nobody-tells-you-until-youve-burnt-in-production-c12f972ec90d)
- [How to do distributed locking — Martin Kleppmann](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)

---

## Replication

Copying data across multiple nodes for **fault tolerance**, **read scaling**, and **geo-distribution**.

Topologies:

- **Single-leader** (master-slave) — all writes go to leader, replicated to followers. Reads can fan out. Most common (Postgres, MySQL).
- **Multi-leader** — multiple nodes accept writes, replicate to each other. Higher availability but conflict resolution is hard.
- **Leaderless** — any node accepts writes. Quorum-based reads/writes (Cassandra, DynamoDB). Tunable consistency.

Sync vs async:

- **Synchronous** — leader waits for follower(s) to confirm before returning success. Strong durability, higher latency. A slow follower stalls all writes.
- **Asynchronous** — leader returns immediately, replicates in the background. Fast, but a leader failure can lose unreplicated writes.
- Most production systems use **semi-sync**: leader waits for at least one follower.

**Replication lag** is the gap between leader and follower state. Causes "read your writes" anomalies — user updates profile, immediately reads from a stale replica, sees old data. Mitigations: read-your-writes routing, read from leader for fresh data.

Replication ≠ sharding — they are orthogonal and often combined (each shard is replicated for fault tolerance).

**Articles:**

- [Master-Slave Architecture (Leader-Based Replication) — Medium](https://codexbook.medium.com/master-slave-architecture-leader-based-replication-79b7095443ec)
- [Database Replication &amp; Sharding Explained — Substack](https://hayksimonyan.substack.com/p/database-replication-and-sharding)
- [Designing Data-Intensive Applications, Ch. 5 (notes)](https://notes.shichao.io/dda/ch5/)

---

## Query Execution & Optimization

SQL is **declarative** — you write what you want, the planner decides how. Logical execution order (different from written order):

```text
FROM → ON → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

Why it matters: column aliases defined in `SELECT` aren't visible to `WHERE`/`GROUP BY` (they're processed earlier). Window functions run after `WHERE`/`GROUP BY` but before `ORDER BY`/`LIMIT`.

`EXPLAIN` / `EXPLAIN ANALYZE` shows the actual plan. Operators to recognize:

- **Seq Scan** — full table scan. Fine for small tables, bad for large ones with selective predicates.
- **Index Scan** — uses an index but reads the table for non-index columns.
- **Index Only Scan** — covering index, no table read needed.
- **Nested Loop** — for each row in outer, scan inner. Fast for small inputs.
- **Hash Join** — builds hash on one side, probes with the other. Fast for medium/large.
- **Merge Join** — both sides sorted, merge. Good when inputs are already ordered.

`HAVING` vs `WHERE`: `WHERE` filters rows before grouping (cheap). `HAVING` filters groups after (expensive). Use `WHERE` when possible.

**Articles:**

- [SQL Query Execution Order — Medium (Ajay Vishwakarma)](https://medium.com/@ajay.v0512/sql-query-execution-order-comprehensive-notes-b7457f62a3a2)
- [Understanding SQL Query Execution Plans — Medium](https://divyanshbhatia.medium.com/understanding-sql-query-execution-plans-unveiling-the-path-to-database-performance-ad9aee04b9f7)

---

## Schema Design & Normalization

Normal forms — incrementally remove redundancy:

- **1NF** — atomic values (no lists in a single cell), each row unique.
- **2NF** — 1NF + no partial dependency on a composite primary key (every non-key column depends on the **whole** key).
- **3NF** — 2NF + no transitive dependency (non-key columns depend **only** on the key, not on other non-key columns).

Mnemonic: *"depends on the key, the whole key, and nothing but the key"*.

**Denormalization** — deliberately introducing redundancy for read performance (e.g. precomputed counts, embedded copies of related data). Trade-off: faster reads, more storage, write amplification, risk of inconsistency.

Practical rule:

- Start at 3NF.
- Denormalize only when you've measured a real read bottleneck.
- Document why each denormalized field exists and how it stays in sync.

Senior-level mistake: jumping into table definitions before clarifying access patterns. Always: identify queries first → design schema to serve them.

**Articles:**

- [SQL Normalization vs Denormalization Explained with 1NF, 2NF, and 3NF — dev.to](https://dev.to/davinceleecode/sql-normalization-vs-denormalization-explained-with-1nf-2nf-and-3nf-2gpa)
- [Database Normalization (1NF, 2NF, 3NF) Explained with Simple Examples — Medium](https://medium.com/illumination/database-normalization-1nf-2nf-3nf-explained-with-simple-examples-6df4fd8aaa68)

---

## CAP & PACELC

**CAP**: in a distributed system, during a network **P**artition you can guarantee at most **two of**: **C**onsistency, **A**vailability, **P**artition tolerance. Since partitions happen, the real choice is **CP** vs **AP**.

CAP only describes behavior **during a partition**. **PACELC** extends it to normal operation:

> If **P**artition → trade off **A**vailability vs **C**onsistency. **E**lse → trade off **L**atency vs **C**onsistency.

This matters more in practice — most of the time there's no partition, and you're constantly trading latency for consistency (sync vs async replication, quorum size, etc.).

Real systems:

| System                 | CAP                   | PACELC |
| ---------------------- | --------------------- | ------ |
| Postgres (single node) | CA-ish (no partition) | —     |
| MongoDB                | CP                    | PC/EC  |
| Cassandra              | AP                    | PA/EL  |
| DynamoDB               | AP (tunable)          | PA/EL  |
| Spanner                | CP                    | PC/EC  |

Consistency models (weakest → strongest):

- **Eventual** — replicas converge eventually. Reads may be stale.
- **Read-your-writes** — you always see your own writes (others may not yet).
- **Causal** — operations causally related are seen in order; concurrent ops may differ.
- **Strong / Linearizable** — every read returns the latest committed write. Most expensive.

**Articles:**

- [Explaining the PACELC theorem to new hires — dev.to](https://dev.to/fahimulhaq/explaining-the-pacelc-theorem-to-new-hires-2de2)
- [PACELC Theorem: Extending the CAP Theorem — Medium](https://medium.com/@arieltk/pacelc-theorem-extending-the-cap-theorem-to-capture-latency-consistency-trade-offs-in-distributed-0a18e51ba1e3)
- [CAP and PACELC Theorems in Plain English — luminousmen](https://luminousmen.com/post/cap-and-pacelc-theorems-in-plain-english/)

---

## SQL vs NoSQL

SQL — relational, schema-on-write, ACID, rich querying via JOINs. Use when relationships and consistency dominate.

NoSQL — umbrella for non-relational stores. Four main types:

| Type          | Examples                   | Best for                                                   |
| ------------- | -------------------------- | ---------------------------------------------------------- |
| Document      | MongoDB, Couchbase         | Flexible schemas, JSON-shaped data, content/catalog        |
| Key-Value     | Redis, DynamoDB            | Caches, sessions, leaderboards, fast lookups               |
| Column-family | Cassandra, ScyllaDB, HBase | Write-heavy, time-series, wide rows                        |
| Graph         | Neo4j, Neptune             | Relationship-heavy queries (social, fraud, recommendation) |

When to pick NoSQL:

- Schema must evolve quickly without migrations.
- Write throughput exceeds what a single SQL node can handle.
- Data doesn't map to rows + joins (graphs, deeply nested documents).

When to stay with SQL:

- You need transactions across multiple entities.
- Ad-hoc analytical queries are common.
- The team is small — operational complexity of NoSQL is real.

Modern Postgres covers a lot of NoSQL use cases (JSONB, vectors via pgvector, full-text search, partitioning) — often the safest default.

**Articles:**

- [Intro to 4 Types of NoSQL Databases — dev.to](https://dev.to/aws-builders/intro-to-4-types-of-nosql-databases-45nh)
- [NoSQL Isn&#39;t One-Size-Fits-All: When to Use Document, Key-Value, Column or Graph](https://exabyting.com/blog/nosql-isnt-one-size-fits-all-when-to-use-document-key-value-column-or-graph/)

---

## Connection Pooling & Caching

### Connection pooling

Opening a TCP + TLS + auth connection to a DB costs tens of ms. A **connection pool** keeps a set of pre-opened connections that requests borrow and return.

- App-level pool — built into ORMs (SQLAlchemy `pool_size`, `max_overflow`).
- External pooler — **PgBouncer** for Postgres. Two important modes:
  - **Session pooling** — connection assigned per client session. Safe but limits multiplexing.
  - **Transaction pooling** — connection released after each transaction. Much higher concurrency, but breaks features that rely on session state (`SET`, prepared statements, cursors).

Typical mistake: setting pool size huge "just in case". Postgres handles ~100–300 active connections well; more usually hurts due to contention. Better: small pool + queue waiters.

### Caching strategies

| Strategy                            | Read flow                                           | Write flow                                  | Notes                               |
| ----------------------------------- | --------------------------------------------------- | ------------------------------------------- | ----------------------------------- |
| **Cache-aside** (lazy)        | App: check cache → miss → load DB → put in cache | App writes DB, invalidates cache            | Most common; cache only what's read |
| **Read-through**              | App asks cache; cache loads from DB on miss         | —                                          | Cache layer handles loading         |
| **Write-through**             | Same as read-through                                | App writes cache → cache writes DB         | Strong consistency, slower writes   |
| **Write-behind** (write-back) | —                                                  | App writes cache; cache async flushes to DB | Fast writes, risk of data loss      |

**Cache stampede / thundering herd**: a popular key expires; thousands of concurrent requests miss simultaneously and all hit the DB. Mitigations: lock on miss (single-flight), probabilistic early refresh, jittered TTLs.

**Articles:**

- [Caching Strategies Explained: Write-Through, Write-Back, and Cache-Aside — Medium](https://medium.com/@stoic.engineer/caching-strategies-explained-write-through-write-back-and-cache-aside-8cd1c0365384)
- [Caching Strategies Across Application Layers — dev.to](https://dev.to/budiwidhiyanto/caching-strategies-across-application-layers-building-faster-more-scalable-products-h08)
- [Caching Strategies (cache-aside, read-through, write-around, write-through) — Medium](https://fitsoftwareengineer.medium.com/caching-strategies-753683c25123)

---

## NULL & Three-Valued Logic

SQL has three truth values: **TRUE**, **FALSE**, **UNKNOWN**. Any comparison with `NULL` returns `UNKNOWN`. Only rows where `WHERE` evaluates to `TRUE` are returned.

Common gotchas:

```sql
-- BAD — never matches anything (NULL = NULL is UNKNOWN, not TRUE)
SELECT * FROM users WHERE deleted_at = NULL;

-- GOOD
SELECT * FROM users WHERE deleted_at IS NULL;
```

```sql
-- NOT IN with NULL silently returns nothing
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM orders);
-- if any orders.user_id is NULL → entire result set is empty

-- Safer
SELECT * FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

Aggregates:

- `COUNT(*)` counts all rows. `COUNT(col)` skips NULLs.
- `SUM`, `AVG`, `MIN`, `MAX` ignore NULLs.
- Be aware: `AVG(col)` with all NULLs returns NULL, not 0.

`IS [NOT] DISTINCT FROM` treats NULLs as equal — useful for "diff" comparisons.

**Articles:**

- [Understanding NULL and Three-valued Logic in SQL — Medium](https://medium.com/@seop7566/understanding-null-and-three-valued-logic-in-sql-8aa9dd574fbc)
- [Understanding SQL&#39;s Three-Valued Logic (3VL) — Medium](https://medium.com/@pramodteepireddy/understanding-sqls-three-valued-logic-3vl-true-false-and-null-75b6dc5945aa)
- [Modern SQL: Three-Valued Logic — modern-sql.com](https://modern-sql.com/concept/three-valued-logic)

---

## Partitioning vs Sharding

Both split data into smaller chunks, but at different scope:

- **Partitioning** — splits a table into smaller pieces **inside one database/server**. The DB engine handles routing automatically.
- **Sharding** — splits data **across multiple databases/servers**. The application (or a router) decides which shard to hit.

Sharding is essentially partitioning across machines for scale-out. Single-node partitioning is much simpler — no cross-node transactions, no shard key headaches.

Partitioning types:

- **Range** — by value range (`date >= '2024-01-01' AND date < '2024-02-01'`). Common for time-series.
- **List** — by explicit list of values (e.g. `country IN ('FR', 'DE', 'IT')`).
- **Hash** — by `hash(column) % N`. Even distribution, but bad for range scans.
- **Composite** — combine two of the above (e.g. range by month, sub-partition by hash of `user_id`).

Why partition at all (even on a single node):

- **Faster queries** — DB scans only matching partitions (partition pruning).
- **Cheap archiving / drop** — `DROP PARTITION` is instant; `DELETE FROM huge_table WHERE …` is slow.
- **Index size stays manageable** — each partition has its own indexes.

Rule of thumb: start with partitioning when a single table grows past tens of millions of rows; reach for sharding only when one machine can no longer hold the data or absorb the writes.

**Articles:**

- [Sharding vs. partitioning: What&#39;s the difference? — PlanetScale](https://planetscale.com/blog/sharding-vs-partitioning-whats-the-difference)
- [Partitioning &amp; Sharding — choosing the right scaling method — Medium](https://medium.com/@_amanarora/partitioning-sharding-choosing-the-right-scaling-method-dbc6b2bec1d5)
- [Sharding vs Partitioning — DataCamp](https://www.datacamp.com/blog/sharding-vs-partitioning)

---

## Database Migrations

A **schema migration** changes the DB schema (or seed data) in a controlled, repeatable way — typically a versioned sequence of files (Alembic, Flyway, Liquibase, Django migrations). Two principles separate amateur from senior-level migrations: **zero downtime** and **safe rollback**.

### Expand & Contract (parallel change)

The standard pattern for breaking schema changes without downtime. Three phases, each independently deployable:

1. **Expand** — add new structure (column, table, index) alongside the old. Old code keeps working.
2. **Migrate / dual-write** — application writes to both old and new shapes; backfill historical data. Old code still works because its columns still exist.
3. **Contract** — once all clients are migrated and verified, drop the old structure.

```text
v1: app reads/writes old_col
v2: ADD new_col (nullable), app dual-writes  ← deployable
v3: backfill new_col from old_col            ← background job
v4: app reads new_col, still writes both     ← deployable
v5: app reads/writes only new_col            ← deployable
v6: DROP old_col                             ← deployable
```

The point: every step is reversible without restoring backups. A "big bang" rename (`ALTER TABLE … RENAME COLUMN …`) is a deploy-time bomb.

### Other migration techniques

- **Dual write** — write to both old and new schema during a transition window (the heart of expand/contract).
- **Shadow reads & writes** — also read from both, compare results, log discrepancies. Used when migrating to a new DB or denormalized read model.
- **Blue-green schema** — run two schemas in parallel (often via separate DBs or schemas); cut over once the new one is verified.
- **Online schema change tools** — for MySQL: **gh-ost** (GitHub), **pt-online-schema-change** (Percona). For Postgres: **pg-osc**, **pgroll** (Xata). They build a shadow table, replay writes, swap atomically — avoiding the long table locks of vanilla `ALTER TABLE`.

Senior-level rule: **migrations are deployed in two PRs, not one** — backward-compatible change first, then cleanup after the application is fully rolled out.

**Articles:**

- [The Expand and Contract Pattern for Zero-Downtime Migrations — dev.to](https://dev.to/jp_fontenele4321/the-expand-and-contract-pattern-for-zero-downtime-migrations-445m)
- [Online DDL and Zero Downtime Schema Migrations in Distributed Databases — dev.to](https://dev.to/software_mvp-factory/online-ddl-and-zero-downtime-schema-migrations-in-distributed-databases-36fp)
- [Expand/Contract: making a breaking change without a big bang — Pete Hodgson](https://blog.thepete.net/blog/2023/12/05/expand/contract-making-a-breaking-change-without-a-big-bang/)
- [Using the expand and contract pattern — Prisma Data Guide](https://www.prisma.io/dataguide/types/relational/expand-and-contract-pattern)

---

## Database Rollbacks

Recovering from bad data or bad deploys, in increasing order of cost:

- **Transaction rollback** (`ROLLBACK`) — within a single open transaction. Cheap, instantaneous.
- **Savepoints** (`SAVEPOINT name; ROLLBACK TO name`) — partial rollback inside a long transaction.
- **Application-level compensating actions** — reverse the effect with another committed transaction (e.g. refund). The only option after `COMMIT`.
- **Backups + Point-in-Time Recovery (PITR)** — restore the database (or a copy) to a specific moment.

### PITR — Point-in-Time Recovery

A full backup + a continuous stream of transaction logs (Postgres WAL, MySQL binlog) lets you replay state up to *any* second between the backup and now. Typical scenario: an `UPDATE` without `WHERE` ran at 14:32 — restore a copy of the DB to 14:31:59, extract the rows, patch production.

PITR is critical, but slow (minutes to hours depending on size + lag). Don't rely on it as a primary safety net.

### Rollback after a deploy — the real strategy

Rolling back the **code** is easy. Rolling back a **schema migration** is hard once new data has been written. The expand/contract pattern is the actual rollback strategy: every intermediate state is backward-compatible, so you just redeploy the previous binary — no DB rollback needed.

**Senior takeaways**:

- Test backups by actually restoring them periodically. An untested backup is hope, not a backup.
- Document RPO (Recovery Point Objective — how much data loss is tolerable) and RTO (Recovery Time Objective — how fast must you be back up). They drive backup frequency and replication topology.
- Avoid destructive migrations that can't be undone via the application layer.

**Articles:**

- [Database Backups 101: What is Point in Time Recovery? — Severalnines](https://severalnines.com/blog/database-backups-101-what-is-point-in-time-recovery/)
- [Point-In-Time Recovery (PITR) in PostgreSQL — pgEdge](https://www.pgedge.com/blog/point-in-time-recovery-pitr-in-postgresql)
