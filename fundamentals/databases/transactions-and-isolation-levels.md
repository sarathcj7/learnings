# Transactions & Isolation Levels

## TL;DR

- **ACID**: Atomicity, Consistency, Isolation, Durability
- **Isolation levels** (weak to strong): Read Uncommitted, Read Committed, Repeatable Read, Serializable
- **Anomalies**: Dirty read, non-repeatable read, phantom read
- **MVCC**: Snapshot isolation, multiple versions of data, readers don't block writers
- **Trade-off**: Stronger isolation = slower, higher lock contention

## ACID Guarantees

### Atomicity
All-or-nothing. No partial commits.

```
Transfer $100: Alice → Bob
  BEGIN
    UPDATE alice SET balance -= 100
    UPDATE bob SET balance += 100
  COMMIT

If second UPDATE fails: Both rolled back (no orphaned money)
```

### Consistency
Data obeys constraints before/after.

```
Constraint: alice.balance >= 0

Violating transaction rejected. Data never inconsistent.
```

### Isolation
Concurrent txns don't interfere. Each has consistent view.

```
Txn 1: SELECT balance WHERE id=1  → $100
Txn 2: UPDATE balance -= 50 WHERE id=1  [concurrent]

Txn 1 should see either $100 or $50, not $75 (intermediate)
```

### Durability
Committed data survives crashes.

```
COMMIT at t=0
Crash at t=5
On restart: Data still there (survived crash)
```

---

## Isolation Levels

### Read Uncommitted (Weakest)

Txn can read uncommitted writes from other txns (dirty reads).

```
Txn 1: UPDATE balance = 0 WHERE id=1
Txn 2: SELECT balance WHERE id=1  → 0 (dirty read!)
Txn 1: ROLLBACK

Txn 2 read committed data that rolled back. Dangerous!
```

**Use**: Rarely. Only when consistency doesn't matter (analytics on best-guess data).

---

### Read Committed (Most Common)

Can only read committed writes. No dirty reads.

```
Txn 1: UPDATE balance = 0 WHERE id=1
Txn 2: SELECT balance WHERE id=1  → Blocked (waits for Txn 1)
Txn 1: ROLLBACK
Txn 2: SELECT balance WHERE id=1  → Original value (safe)
```

**Anomaly**: Non-repeatable read

```
Txn 1: SELECT balance WHERE id=1  → 100
Txn 2: UPDATE balance = 50 WHERE id=1, COMMIT
Txn 1: SELECT balance WHERE id=1  → 50 (changed!)
```

**Use**: Default in PostgreSQL, MySQL. Safe for most applications.

---

### Repeatable Read

Same query returns same results within txn. No non-repeatable reads.

```
Txn 1: SELECT balance WHERE id=1  → 100 (snapshot at t=0)
Txn 2: UPDATE balance = 50 WHERE id=1, COMMIT
Txn 1: SELECT balance WHERE id=1  → 100 (still old snapshot, repeatable!)
```

**Anomaly**: Phantom read

```
Txn 1: SELECT COUNT(*) FROM users WHERE age > 30  → 10
Txn 2: INSERT user(age=35), COMMIT
Txn 1: SELECT COUNT(*) FROM users WHERE age > 30  → 11 (phantom!)
```

**Use**: When you need repeatable reads within txn (usually safe enough).

---

### Serializable (Strongest)

Txns execute as if serial (one after another). No anomalies.

```
Txn 1 and Txn 2 execute, result is same as if one ran then the other.

No dirty reads, non-repeatable reads, or phantom reads.
```

**Cost**: Very slow (locks everything, high contention).

**Use**: Financial systems, inventory (need atomicity across reads/writes).

---

## MVCC (Multi-Version Concurrency Control)

Readers don't block writers. Each reader gets snapshot.

```
Database: { id: 1, balance: 100 }

Txn A: SELECT balance → Read version 1 (balance: 100)
Txn B: UPDATE balance = 50 → Create version 2

Txn A: SELECT balance again → Still version 1 (balance: 100, repeatable)
Txn B: COMMIT
Txn A: SELECT balance → Version 2 (balance: 50) after Txn A commits

Result: Readers and writers don't block each other. Concurrency!
```

**Trade-off**: Must maintain multiple versions (storage overhead).

---

## Anomalies & Trade-offs

| Anomaly | Read Committed | Repeatable Read | Serializable |
|---|---|---|---|
| **Dirty reads** | No | No | No |
| **Non-repeatable reads** | Yes | No | No |
| **Phantom reads** | Yes | Yes | No |
| **Concurrency** | High | Medium | Low |

---

## Production Considerations

1. **Choose level based on need**: Read Committed is default, strong enough for most
2. **Watch lock contention**: Serializable transactions can bottleneck
3. **Deadlocks**: Concurrent writes can deadlock (A waits for B, B waits for A). Retry logic needed
4. **Long-running txns**: Snapshot isolation ages, performance degrades
5. **MVCC storage**: Old versions consume space until vacuum/cleanup

---

## References

- "Designing Data-Intensive Applications" — Kleppmann, Chapter 7
- PostgreSQL documentation: "Isolation Levels"

---

## Related Fundamentals

- [Relational Fundamentals](relational-fundamentals.md) – ACID overview
- [Distributed Transactions](../distributed-transactions/) – Isolation across shards
