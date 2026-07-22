# NoSQL & NewSQL Landscape

## TL;DR

- **NoSQL**: Key-Value, Document, Wide-Column, Graph. Traded ACID for scale/flexibility.
- **NewSQL**: SQL interface + horizontal scale + ACID (Spanner, CockroachDB). Modern sweet spot.
- **Choice matrix**: Structured + ACID = SQL; Flexible + Scale = NoSQL; Both = NewSQL (expensive).
- **BASE vs ACID**: Basically Available, Soft-state, Eventual consistency vs full ACID.

## NoSQL Types

### Key-Value Stores
Simple: key → value. Examples: Redis, Memcached, DynamoDB, Cassandra.

```
SET user:1 "Alice"
GET user:1 → "Alice"
```

**Pros**: Fast (no joins), scales, simple
**Cons**: No queries, no ACID, no aggregations

---

### Document Stores
JSON/BSON documents. Examples: MongoDB, CouchDB, Firestore.

```
Collection: users
  { _id: 1, name: "Alice", address: { city: "NYC" } }
  { _id: 2, name: "Bob", address: { city: "LA" } }

Query: db.users.find({ "address.city": "NYC" })
```

**Pros**: Flexible schema, nested data, reasonably queryable
**Cons**: Joins are harder, eventual consistency (usually), no complex transactions

---

### Wide-Column Stores
Distributed, columnar. Examples: HBase, Cassandra (hybrid KV+column).

```
Table: Users
  Row Key: user_id
  Column Families: profile, activity
    profile: { name, email, address }
    activity: { lastLogin, followerCount, posts }

Optimized for: Retrieve all activity for a user
Not optimized for: Find all users in NYC (full table scan)
```

**Pros**: Horizontal scale, fast range queries by row key
**Cons**: Eventual consistency, complex API

---

### Graph Databases
Relationships as first-class citizens. Examples: Neo4j, Amazon Neptune.

```
Nodes: Users, Posts, Comments
Edges: user -FOLLOWS-> user, user -WROTE-> post, user -COMMENTED-> comment

Query: Find all posts liked by friends of Alice
  → Traverse: Alice -FRIEND-> Friend -LIKES-> Post
  → Much faster than JOIN in SQL
```

**Pros**: Relationships are fast, traversals natural
**Cons**: Not great for aggregations, niche use cases

---

## SQL vs NoSQL vs NewSQL

| Aspect | SQL | NoSQL | NewSQL |
|---|---|---|---|
| **Schema** | Strict | Flexible | Strict |
| **Joins** | Yes (slow at scale) | No | Yes (optimized) |
| **ACID** | Yes | Usually no | Yes |
| **Horizontal scale** | Hard | Easy | Easy |
| **Query language** | SQL | Custom (get/put) | SQL |
| **Cost** | Moderate | Low to high | High |
| **Examples** | PostgreSQL, MySQL | MongoDB, DynamoDB | Spanner, CockroachDB |

---

## When to Choose

### SQL (PostgreSQL, MySQL)

**Use when**:
- Data is structured (schema defined)
- Transactions matter (money, inventory)
- Complex queries (joins, aggregations)
- Scale < 10-100 GB

**Avoid when**:
- Unstructured data (JSON, documents)
- Extreme write scale (1M writes/sec)
- Distributed requirement (multi-region)

---

### NoSQL (MongoDB, DynamoDB, Cassandra)

**Use when**:
- Data is semi-structured (flexible schema)
- Massive scale (1B+ rows, 1M+ QPS)
- High availability (survive region loss)
- Eventual consistency OK

**Avoid when**:
- Complex queries needed
- Transactions across documents
- Strong consistency required

---

### NewSQL (Spanner, CockroachDB)

**Use when**:
- Need both: ACID + horizontal scale
- Multi-region + strong consistency
- Complex queries + scale

**Avoid when**:
- Cost sensitive (NewSQL is expensive)
- Simple key-value access (use NoSQL, faster)

---

## BASE vs ACID

### ACID (Relational)
- **Atomic**: All or nothing
- **Consistent**: Constraints obeyed
- **Isolated**: No interference
- **Durable**: Survives crashes

Example: Bank transfer (must be ACID).

### BASE (NoSQL)
- **Basically Available**: System always responds (might be stale)
- **Soft state**: Data changes over time (eventual consistency)
- **Eventual consistency**: Eventually all nodes agree

Example: Social media like count (BASE is fine, eventual OK).

---

## Polyglot Persistence

Use multiple databases for different needs.

```
Application:
  User profiles → PostgreSQL (relational, transactions)
  Session cache → Redis (fast KV)
  Analytics events → DynamoDB (write scale)
  Timeline → Elasticsearch (search/indexing)
  Social graph → Neo4j (relationships)
  
Tradeoff: Operational complexity vs optimal for each use case.
```

---

## Production Considerations

1. **Schema evolution**: Even NoSQL needs schema management (document versions)
2. **Backup strategy**: Different per DB type (SQL: snapshots, NoSQL: replication)
3. **Monitoring**: Each DB has unique metrics (replication lag, write amplification, GC pauses)
4. **Testing**: NoSQL eventual consistency harder to test
5. **Migration**: Switching DBs mid-scale is painful, choose wisely upfront

---

## References

- "Designing Data-Intensive Applications" — Kleppmann
- "NoSQL Distilled" — Fowler & Sadalage

---

## Related Fundamentals

- [Relational Fundamentals](relational-fundamentals.md) – SQL deep dive
- [Replication](replication.md) – How NoSQL/NewSQL replicate at scale
- [Distributed Transactions](../distributed-transactions/) – Cross-shard ACID (NewSQL specialty)
