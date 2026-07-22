# Notification System

*Design a notification service like Firebase Cloud Messaging, Twilio. Send billions of push notifications, emails, SMS reliably.*

## Problem Statement

Send notifications to 500M users. 1M notifications/sec (user actions trigger them). Deliver 99.9% of notifications within 60 seconds. Route via push/email/SMS based on user preference.

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Notifications/sec** | 1M | 86B/day |
| **Delivery channels** | 3 (push, email, SMS) | Parallel sends |
| **Retry attempts** | 3× per notification | Reliability |

## Architecture

```
Event Source:
  User action (comment, like, follow) → Event published to Kafka
  
Notification Service:
  Consume event
  Build notification payload
  Determine channels (user preferences)
  Send to channel-specific workers
  
Channel Workers:
  Push: Firebase, APNs, GCM
  Email: SendGrid, mailgun
  SMS: Twilio, Nexmo
  
Persistence:
  Track delivery status
  Retry failed sends
```

## Delivery Guarantees

**Goal**: At-least-once delivery (user gets notification, may be duplicate)

```
Workflow:
1. Event → Kafka (persisted)
2. Read from Kafka
3. Send notification
4. Mark as sent (in database)

On failure:
5. Retry (exponential backoff)
6. After 3 retries, mark as failed

Dead letter queue:
  Persistent failures → Human review
```

## Challenges

### Rate Limiting

Send too many → User unsubscribes.

```
Per-user limits:
  Max 10 notifications/hour from same app
  Max 50/day total
  
Backoff:
  User mutes app → No notifications 24h
```

### Delivery Latency

**Challenge**: Email is slow (5-60 min), push is fast (< 1 sec).

**Solution**: 
- Push immediately (best effort, < 1 sec)
- Email batched (send every 5 min, lower cost)

### Failure Modes

Push service down → Queue and retry.

```
Firebase down:
  Enqueue to Redis (durable)
  Retry with exponential backoff
  Eventually succeeds when Firebase back up
```

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Multiple channels** | Single channel | Redundancy + user preference |
| **At-least-once** | Exactly-once | Simpler, duplicates are OK for notifications |
| **Exponential backoff** | Fixed retry | Avoid thundering herd on channel recovery |

---

**Status**: ✅ Complete. Shows event-driven, multi-channel, reliable delivery.
