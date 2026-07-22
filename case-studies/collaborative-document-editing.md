# Collaborative Document Editing

*Design Google Docs / Figma: real-time collaboration, conflict resolution, offline mode, rich media.*

## Problem Statement

Build a collaborative editor. Multiple users edit same document simultaneously. Changes must be visible in real-time to all collaborators. Works offline, syncs when reconnected.

## Challenges

### Conflict Resolution

```
User A types "Hello"
User B types " World" (concurrently)

If A finishes first: "Hello"
If B finishes first: " World"
Result order matters!

Solution: Operational Transform or CRDTs
  OT: Transform each op based on concurrent ops
  CRDT: Data structure designed for merging (no transform needed)
```

### Offline Support

```
User works offline:
  Local changes stored in browser
  Send changes to server when online
  Server merges with other users' changes
  Send merged version back (reconcile)
```

## Architecture

```
Client (Figma, Google Docs):
  Local CRDT state
  Local edits applied immediately (perceived latency = 0)
  Send ops to server
  Receive remote ops, apply to local state

Server:
  Central authority for op ordering
  Persist all ops (immutable log)
  Broadcast ops to all clients
  Handle conflicts (ops from multiple clients arriving simultaneously)

Persistence:
  Operational log: {op_id, user_id, change, timestamp}
  Snapshots: Every 1000 ops (for fast loading)
```

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **CRDT** | OT | CRDTs are simpler, no transform needed |
| **Immutable ops** | Mutable document | Ops form audit trail, enable undo/replay |
| **Eventual consistency** | Strong consistency | High latency for immediate consistency |

---

**Status**: ✅ Complete. Shows real-time collaboration, CRDTs, conflict resolution.
