# Relational Fundamentals

## TL;DR

- **Relational model**: Data organized in tables (entities), with relationships (foreign keys)
- **ACID**: Atomicity, Consistency, Isolation, Durability — guarantees for data integrity
- **Normalization**: Reduce redundancy via 1NF, 2NF, 3NF. Higher = less redundancy but more joins
- **Denormalization**: Add redundancy to reduce joins and speed up reads. Trade consistency for speed
- **SQL**: Standard language for queries. Master SELECT, JOIN, WHERE, GROUP BY, INDEX

## Relational Model Basics

### Entities & Relationships

```
Users table:
  userId (PK) | name | email
  1          | Alice | alice@example.com
  2          | Bob   | bob@example.com

Posts table:
  postId (PK) | userId (FK) | title | content | createdAt
  100        | 1          | "Hello" | "..." | 2026-07-23
  101        | 1          | "World" | "..." | 2026-07-23
  102        | 2          | "Hi"    | "..." | 2026-07-24

Relationship: One user → Many posts (1:N)
Foreign key: Posts.userId references Users.userId
```

### Primary Key (PK)
Uniquely identifies a row. Examples: userId, postId, compositeKey (userId + timestamp).

**Properties**:
- Unique (no duplicates)
- Not null
- Immutable (or rarely changes)

### Foreign Key (FK)
Links to another table's primary key. Enforces referential integrity.

```
DELETE user 1? 
  If ON DELETE CASCADE: All posts by user 1 deleted too
  If ON DELETE RESTRICT: Error, can't delete (has posts)
  If ON DELETE SET NULL: User deleted, posts.userId set to NULL
```

## ACID Guarantees

### Atomicity
Transaction is "all or nothing". Either entire transaction succeeds, or all rolls back.

```
Transfer $100 from Alice to Bob:
  BEGIN TRANSACTION
    UPDATE accounts SET balance = balance - 100 WHERE userId = 1  // Alice
    UPDATE accounts SET balance = balance + 100 WHERE userId = 2  // Bob
  COMMIT

If either UPDATE fails, both roll back. No orphaned money.
```

### Consistency
Data obeys all constraints before and after transaction.

```
Constraint: user.email is UNIQUE

Transaction tries to insert alice@example.com twice → Violates constraint → Rejected
Database state: Consistent (constraint still holds)
```

### Isolation
Concurrent transactions don't interfere. Each sees a consistent view.

```
T1: SELECT balance WHERE userId=1  → $1000
T2: UPDATE balance = balance - 500 WHERE userId=1  // Concurrent

T1 should see either $1000 (T2 hasn't committed) or $500 (T2 committed), not intermediate state
```

### Durability
Once committed, data survives crashes/power loss.

```
Transaction committed at t=0
Data written to disk (not just memory)
Server crashes at t=1
On restart: Data is still there (recovered from disk)
```

## Normalization (1NF → 3NF)

Normalization reduces redundancy. Higher normal forms = fewer anomalies.

### 1NF (First Normal Form)
No repeating groups. All atomic values.

```
❌ NOT 1NF:
  User: { userId, name, phoneNumbers: [123-456, 789-012] }

✅ 1NF:
  User: { userId, name }
  Phone: { phoneId, userId, phoneNumber }
```

### 2NF (Second Normal Form)
1NF + No partial dependencies. Non-key columns depend on entire primary key.

```
❌ NOT 2NF (if PK = (courseId, studentId)):
  Enrollment: { courseId, studentId, courseName }
  (courseName depends only on courseId, not entire key)

✅ 2NF:
  Enrollment: { courseId, studentId }
  Course: { courseId, courseName }
```

### 3NF (Third Normal Form)
2NF + No transitive dependencies. Non-key columns depend only on primary key.

```
❌ NOT 3NF:
  Student: { studentId, name, cityId, cityName }
  (cityName depends on cityId, not directly on studentId — transitive dependency)

✅ 3NF:
  Student: { studentId, name, cityId }
  City: { cityId, cityName }
```

### Normalization Trade-off

**Benefits of high normalization**:
- Reduce redundancy (save storage)
- Prevent update anomalies (change courseNameonce, not in 1000 enrollments)
- Data integrity (ACID constraints easier to maintain)

**Costs**:
- More joins required (join courseId → Course → courseName takes 2 tables, slower)
- Complex queries

**When to denormalize**: Read-heavy systems where join cost > redundancy cost.

## Denormalization

Add redundancy strategically to speed up common queries.

```
❌ Normalized (requires JOIN):
  SELECT e.courseId, e.studentId, c.courseName
  FROM Enrollment e
  JOIN Course c ON e.courseId = c.courseId
  WHERE e.studentId = 123

✅ Denormalized (no JOIN needed):
  Enrollment: { enrollmentId, studentId, courseId, courseName }
  
  Tradeoff: courseName is redundant (stored in Course too)
  Benefit: Query is faster (no JOIN)
  Cost: Update is harder (change courseName in Course AND all Enrollments)
```

**When to denormalize**:
- Read-heavy (99% reads, 1% writes): Redundancy cost acceptable
- Aggregate columns (SUM, COUNT): Store pre-computed totals
- Derived data: Store computed values (e.g., denormalize user.followerCount instead of joining Followers table)

## Indexes

Structure to speed up queries. Trade storage for speed.

### B-tree Index (Most Common)

```
Index on User.email:

        "bob@..."
       /          \
  "alice@..."   "charlie@..."
  
Query: SELECT * WHERE email = "alice@example.com"
  Instead of scanning all 1B users (1B comparisons)
  Use index: Navigate tree (log N ≈ 30 comparisons for 1B rows)
  1000x faster
```

### Types

| Index | Use | Cost |
|---|---|---|
| **Single-column** | WHERE name='Alice' | Small |
| **Composite** | WHERE name='Alice' AND age=30 | Medium |
| **Unique** | Email must be unique | Auto-enforced |
| **Covering** | SELECT name, age WHERE age > 30 (both columns in index) | Medium, but query never touches table |

### Index Trade-offs

**Pros**:
- Reads 10–1000x faster

**Cons**:
- Writes slower (must update index too)
- Storage overhead (index copy of data)
- Too many indexes = slower (DB must pick which to use)

### When to Index

- Columns in WHERE clauses (filtration)
- Foreign keys (joins)
- Columns in ORDER BY
- NOT on low-cardinality columns (only 2 values: male/female)

## Query Optimization

### Execution Plan

```
EXPLAIN SELECT * FROM posts WHERE userId = 1 AND createdAt > '2026-07-01' ORDER BY createdAt DESC

Result:
  Seq Scan on posts → (scans all 1B rows, slow)
  
WITH INDEX on (userId, createdAt):
  Index Scan on posts_userId_createdAt → (navigates index, fast)
```

**Better queries**:
```
❌ Slow: SELECT * FROM posts WHERE YEAR(createdAt) = 2026
  (YEAR() prevents index use, must scan all rows and compute)

✅ Fast: SELECT * FROM posts WHERE createdAt >= '2026-01-01' AND createdAt < '2027-01-01'
  (Can use index on createdAt directly)
```

## Production Considerations

1. **Connection pooling**: Reuse connections, don't open new one per query
2. **Prepared statements**: Prevent SQL injection, cache query plans
3. **Read replicas**: Scale reads without burdening primary
4. **Sharding**: Scale writes (partition data)
5. **Monitoring**: Track slow queries (> 100ms), lock contention, disk I/O

---

## References

- "Designing Data-Intensive Applications" — Martin Kleppmann, Chapter 2
- PostgreSQL documentation: "Schema and Data Types"
- MySQL documentation: "Optimization and Indexes"

---

## Related Fundamentals

- [Indexing](indexing.md) – Deep dive on index structures
- [Replication](replication.md) – Scaling reads
- [Sharding & Partitioning](sharding-and-partitioning.md) – Scaling writes
- [Transactions & Isolation Levels](transactions-and-isolation-levels.md) – ACID in detail
