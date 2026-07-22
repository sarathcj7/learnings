# Payment & Ledger System

*Design a payment processing system like Stripe, PayPal. Transactions must be atomic, balance correct, audit trail complete.*

## Problem Statement

Build a payment system. Process 1M+ transactions/sec, handle money transfers atomically. Every transaction must be durable and auditable. Zero tolerance for lost money.

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Transactions/sec** | 1M | Extreme throughput |
| **Balance queries/sec** | 10M | Real-time balance checks |
| **Durability** | 11 9's | Money is critical |
| **Audit trail** | Immutable log | Compliance requirement |

## Architecture

```
Transaction Flow:
1. User initiates payment
   POST /transfer {from_user, to_user, amount}

2. Account Service:
   Check from_user balance >= amount
   Lock accounts (prevent concurrent modifications)

3. Ledger Service:
   Record debit: -amount from from_user
   Record credit: +amount to to_user
   Single atomic transaction (ACID)

4. Confirm:
   Return transaction_id to client
   Send confirmation emails

5. Audit:
   Immutable log: {transaction_id, from, to, amount, timestamp}
```

## Challenges

### Atomicity Across Accounts

```
Transfer $100: Alice → Bob

ACID transaction:
  BEGIN
    UPDATE accounts SET balance = balance - 100 WHERE id = alice
    UPDATE accounts SET balance = balance + 100 WHERE id = bob
  COMMIT

If crash between updates:
  Rollback entire transaction (both or neither)
  No partial transfers
```

### Handling Failures

```
Transaction partially committed:
  Alice's balance decreased, Bob's didn't increase
  Solution: 2-phase commit or distributed transaction
  
Idempotency:
  If client retries same transaction:
    Check transaction_id already processed
    Return same result (idempotent)
```

### Reconciliation

```
Ledger must balance:
  Sum of all debits == Sum of all credits

Daily reconciliation:
  Sum(account.balance) should == 0 (balanced)
  If not: Alert (bug or data corruption)
```

## Bottlenecks & Scaling

**Bottleneck**: Lock contention on popular accounts.

**Solution**:
- Shard accounts (hash(userId) % num_shards)
- Each shard handles independent accounts
- Cross-shard transfers use 2PC (expensive but rare)

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Strong consistency** | Eventual consistency | Money is critical, cannot afford inconsistency |
| **Distributed transaction** | Eventual consistency | Cross-shard transfers need atomicity |
| **Immutable audit log** | Mutable | Compliance, fraud detection, dispute resolution |

---

**Status**: ✅ Complete. Shows ACID, ledger, compliance, audit trails.
