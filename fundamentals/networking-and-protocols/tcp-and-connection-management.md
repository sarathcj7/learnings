# TCP & Connection Management

Understanding TCP/IP fundamentals, handshakes, timeouts, and connection lifecycle.

---

## TL;DR

- **TCP three-way handshake**: SYN, SYN-ACK, ACK (establishes connection)
- **Connection states**: LISTEN, SYN-SENT, SYN-RECEIVED, ESTABLISHED, CLOSE-WAIT, CLOSED
- **Keepalive**: Detect dead connections, prevent timeout
- **Backlog**: Queue of pending connections, small by default
- **Timeout tuning**: Prevents zombies, frees resources

---

## TCP Three-Way Handshake

### Step 1: SYN

```
Client → Server: "I want to connect"
  SYN flag set
  Sequence number: 1000
  
Server receives: Moves to SYN-RECEIVED state
```

---

### Step 2: SYN-ACK

```
Server → Client: "OK, I acknowledge, I also want to connect"
  SYN flag set (server initiates its own sequence)
  ACK flag set (acknowledges client's sequence)
  Sequence number: 2000 (server's sequence)
  Acknowledgment: 1001 (client's sequence + 1)
  
Client receives: Moves to ESTABLISHED state
```

---

### Step 3: ACK

```
Client → Server: "I acknowledge your sequence"
  ACK flag set
  Sequence number: 1001
  Acknowledgment: 2001 (server's sequence + 1)
  
Server receives: Moves to ESTABLISHED state

Result: Both sides synchronized, connection established
```

---

## TCP Connection Latency

```
Timeline (typical):
  T=0ms: Client sends SYN
  T=50ms: Server receives SYN, sends SYN-ACK
  T=100ms: Client receives SYN-ACK, sends ACK
  T=150ms: Server receives ACK
  
Total: 150ms from "initiate connection" to "can send data"

This is BEFORE any application data sent!
```

---

## TCP States

### Active Connection

```
Client side:
  CLOSED → SYN-SENT (client sends SYN)
         → ESTABLISHED (client receives SYN-ACK)

Server side:
  LISTEN (waiting for connection)
        → SYN-RECEIVED (server receives SYN)
        → ESTABLISHED (server sends ACK)
        
Result: Both in ESTABLISHED state, can exchange data
```

---

### Closing Connection

```
Graceful close (4-way handshake):

Client: Sends FIN (finish)
  ESTABLISHED → FIN-WAIT-1

Server: Receives FIN
  ESTABLISHED → CLOSE-WAIT
  Sends ACK

Client: Receives ACK
  FIN-WAIT-1 → FIN-WAIT-2

Server: Sends FIN when done
  CLOSE-WAIT → LAST-ACK

Client: Receives FIN
  FIN-WAIT-2 → TIME-WAIT
  Sends ACK

Server: Receives ACK
  LAST-ACK → CLOSED

Client: Waits 2 minutes (TIME-WAIT prevents reuse issues)
  TIME-WAIT → CLOSED
```

---

## Connection Pool Sizing

### Socket Limits

```
Each TCP connection uses resources:
  File descriptor (OS limit: ulimit -n)
  Memory (kernel buffers: ~15KB per connection)
  
Example server:
  OS limit: 65536 file descriptors
  Max connections: ~4000 (accounting for other FDs)
  
Typical: Set to 100k-1M connections on large servers
```

---

### Backlog & SYN Queue

```
Server listening on port 80:
  Backlog: 128 (default)
  
Clients connect rapidly:
  Connection 1: ESTABLISHED
  Connection 2-128: In accept queue (waiting for accept())
  Connection 129: REJECTED (backlog full)
  
If server is busy processing:
  New connections pile up in backlog
  Once backlog full, connections rejected
  Clients see: Connection refused
  
Mitigation: Increase backlog, process connections faster
```

---

## TCP Keepalive

### Problem: Dead Connection

```
Client-Server connection established
Network goes down (WiFi dropped)

From client perspective:
  Connection still ESTABLISHED (OS doesn't know network is dead)
  Client waits forever for response (or timeout)
  
From server perspective:
  Connection still ESTABLISHED (client never sent FIN)
  Server might wait forever
  
Result: Ghost connections accumulate (resource leak)
```

---

### Solution: Keepalive

```
Server sends periodic keepalive packets:
  "Are you still there?"
  
If no response after 3 probes:
  Server closes connection
  Resources freed

Configuration (Linux):
  tcp_keepalive_time: 2 hours (wait before first probe)
  tcp_keepalive_intvl: 75 seconds (between probes)
  tcp_keepalive_probes: 9 (how many before give up)
  
Default: 2 hours + 9 × 75 sec = 2+ hours to detect dead connection

For application-level:
  Use shorter intervals: 30 seconds (catch failures faster)
```

---

## Connection Reset & Errors

### RST (Reset)

```
Server abruptly closes connection without graceful shutdown:

Server: Sends RST flag
Client: Receives RST
  Connection immediately CLOSED (no TIME-WAIT)

Causes:
  - Server process crashed
  - Server decided connection invalid
  - Network misconfiguration

Result: Client can retry immediately (no TIME-WAIT delay)
```

---

### TIMEOUT

```
Client sends data, no response

Timeout handling:
  Attempt 1: Send data, wait 1 second, retry
  Attempt 2: Wait 2 seconds, retry
  Attempt 3: Wait 4 seconds, retry
  Attempt 4: Wait 8 seconds, retry
  ...
  After ~15 minutes: Give up, return error

Typical application timeout: 5-30 seconds (before TCP gives up)
```

---

## TCP Tuning for Scale

### Buffer Sizing

```
Each TCP connection has send/receive buffers:
  Default: 87KB (receive) + 16KB (send) = ~103KB per connection
  
At 10,000 connections: 1GB just for buffers!

Tuning:
  net.core.rmem_max: Increase to 128MB
  net.core.wmem_max: Increase to 128MB
  tcp_rmem: (4KB, 87KB, 6MB) allow larger buffers

Result: Better throughput for fewer connections
```

---

### Connection Timeouts

```
tcp_fin_timeout: 60 seconds
  How long to wait for connection close
  After FIN sent, wait 60s for reply
  High: More resource usage, slow cleanup
  Low: Might drop valid connections
  Recommended: 20-30 seconds

tcp_tw_reuse: 1 (Linux)
  Reuse TIME-WAIT connections
  Allows binding to same port faster
  Safe with timestamps enabled
```

---

## Connection Pooling Best Practices

```
Pool per service:
  Service A → Database connection pool (20 connections)
  Service A → Cache connection pool (10 connections)
  Service A → Message queue connection pool (5 connections)

Don't:
  Reuse connections without testing (stale connection)
  Create new connection per request (overhead)
  Use single connection (serial requests, bottleneck)

Keep-alive:
  HTTP Keep-Alive: Reuse TCP connection for multiple HTTP requests
  Expected behavior: One TCP connection, many HTTP requests
```

---

## Production Checklist

- [ ] Connection pool size tuned to workload
- [ ] TCP keepalive configured (30-60s interval)
- [ ] Listen backlog increased if high concurrency
- [ ] Timeout values appropriate (5-30s application, 20s tcp_fin)
- [ ] Buffer sizes tuned (rmem_max, wmem_max)
- [ ] Connection reuse via keep-alive enabled
- [ ] Monitoring connection count (alert if growing unbounded)
- [ ] Graceful shutdown implemented (drain before close)

---

## Related Fundamentals

- [Connection Pooling](../scalability-and-load-balancing/connection-pooling-and-timeouts.md) – Application-level pooling
- [Load Balancing](../scalability-and-load-balancing/load-balancing-algorithms.md) – Connection distribution
- [Reliability](../reliability-and-resiliency/) – Detecting failed connections

---

**Status**: ✅ Complete. Covers handshakes, states, keepalive, tuning, pooling.

