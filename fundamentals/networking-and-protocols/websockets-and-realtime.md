# WebSockets & Real-Time Transport

Persistent bidirectional connections for real-time applications (chat, notifications, collaborative editing).

---

## TL;DR

- **WebSocket**: TCP upgrade, full-duplex communication
- **HTTP polling**: Client polls server repeatedly (wasteful)
- **Long polling**: Server holds request until data available (better)
- **Server-Sent Events (SSE)**: Server pushes to client (one-way)
- **WebSocket ideal for**: Chat, games, collaboration (bidirectional real-time)

---

## Transport Comparison

### HTTP Polling (Worst)

```
Client polls every 1 second:
  GET /messages (no new data)
  wait 1 sec
  GET /messages (no new data)
  wait 1 sec
  ...
  GET /messages (new message!)
  
Latency: Up to 1 second (if message arrives right after poll)
Bandwidth: 1 request/sec = 86,400 requests/day = wasted!
```

---

### Long Polling (Better)

```
Client sends: GET /messages (wait for data)

Server:
  Check: Any new messages?
  No → Hold connection open (wait up to 30 seconds)
  Message arrives → Send immediately
  Client receives and closes connection
  
Result: Message delivered within 30s
Latency: ~5-30ms (much better!)
Bandwidth: Only requests when data available
```

---

### Server-Sent Events (SSE)

```
Client: 
  connection = new EventSource('/stream')
  connection.onmessage = handleMessage

Server:
  Keep connection open
  Push messages whenever they arrive:
    data: {"type": "message", "text": "hello"}
  
Result: Server can push anytime (low latency)
Bandwidth: Efficient (only when data)

Limitation: Server → Client only (no client → server over same connection)
```

---

### WebSocket (Best for Bidirectional)

```
Client initiates:
  GET /chat HTTP/1.1
  Upgrade: websocket
  Connection: Upgrade

Server accepts:
  101 Switching Protocols
  Upgrade: websocket

Result: Persistent TCP connection
Both send messages anytime:
  Client: {"type": "message", "text": "hello"}
  Server: {"type": "notification", "text": "user joined"}
  
Latency: <10ms (bidirectional)
Bandwidth: Efficient (persistent connection)
```

---

## WebSocket Handshake

### Upgrade Process

```
Client sends HTTP request with Upgrade header:
  GET /chat HTTP/1.1
  Host: example.com
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13

Server responds (if accepting):
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

Result: TCP connection upgraded to WebSocket
No more HTTP semantics (no request/response)
Bidirectional frames sent
```

---

## WebSocket Frames

### Frame Structure

```
Each message sent as frame:

[FIN (1 bit)] [RSV (3 bits)] [OPCODE (4 bits)]
[MASK (1 bit)] [PAYLOAD LENGTH (7-63 bits)]
[MASKING KEY (32 bits if MASK=1)]
[PAYLOAD DATA]

Opcodes:
  0x1: Text frame
  0x2: Binary frame
  0x8: Close frame
  0x9: Ping (keep-alive)
  0xA: Pong (response to ping)

Masking: Required for client→server (security)
```

---

## Scaling WebSockets

### Single Server

```
Node.js with Socket.io:
  ~10,000 concurrent connections per server
  Memory: ~50KB per connection = 500MB for 10k
  CPU: ~50-100% when active (high message rate)
```

---

### Multiple Servers (Horizontal Scaling)

```
Problem:
  Client A connected to Server 1
  Client B connected to Server 2
  Client A sends message to B → Server 1 doesn't know about Server 2

Solution 1: Shared Session Store
  Server 1: Receives message from A
  Server 1: Publish to Redis pub/sub channel
  Server 2: Subscribed to channel, receives message
  Server 2: Sends to B via WebSocket
  
Implementation:
  Socket.io Adapter: Redis
  Result: Any server can reach any client
```

---

### Scaling Considerations

```
10k concurrent WebSockets per server
100k users online → Need 10+ servers

Load balancer:
  Sticky sessions (same user → same server)
  OR session affinity via shared state

Shared state options:
  Redis (fast, in-memory)
  Database (durable, slower)
  Kafka (event streaming, more complex)
```

---

## Real-World Example: Chat

### Chat with WebSocket

```
User A connects to Server 1
User B connects to Server 2

User A sends message to User B:
  1. A: WebSocket message to Server 1
  2. Server 1: Publish to Redis: "channel:chat_123"
  3. Server 2: Subscribed, receives message
  4. Server 2: Send to B via WebSocket
  5. B: Receives in real-time (~10ms)

Multiple recipients (group chat):
  Server 1: Publish to "channel:group_456"
  All servers subscribed receive
  All connected users get message
```

---

## Fallbacks & Compatibility

### Browser Support

```
WebSocket: Supported in 99%+ browsers
Fallbacks for older browsers:
  1. Try WebSocket (modern)
  2. Fall back to long polling (older)
  3. Fall back to HTTP polling (ancient)

Libraries (automatic fallback):
  Socket.io
  SockJS
  ws + fallback library
```

---

## Production Patterns

### Keepalive

```
Problem:
  Idle connections can timeout (firewall, proxy)
  Client thinks connected, actually dead

Solution:
  Server sends PING frame every 30 seconds
  Client responds with PONG
  
  If no PONG received: Assume dead, close
  If PONG received: Connection alive, continue
```

---

### Graceful Shutdown

```
Server shutting down:

1. Stop accepting new WebSocket connections
2. Send close frame to all connected clients:
   {close: true, reason: "server maintenance"}
3. Wait 30 seconds for clients to reconnect elsewhere
4. Force close any remaining connections
5. Shutdown complete

Result: Clients reconnect automatically (no message loss)
```

---

## When to Use Each

| Transport | Latency | Bandwidth | Complexity | Use Case |
|---|---|---|---|---|
| **HTTP Polling** | 1s+ | Wasteful | Low | Dashboard (slow updates) |
| **Long Polling** | 100ms | Good | Low | Fallback, simple updates |
| **SSE** | 10-100ms | Good | Low | Notifications, server push |
| **WebSocket** | <10ms | Efficient | Medium | Chat, games, collaboration |

---

## Related Fundamentals

- [HTTP/2 & HTTP/3](http-and-versions.md) – Connection management
- [Message Queues](../messaging-and-streaming/kafka-architecture.md) – Message distribution
- [Reliability](../reliability-and-resiliency/circuit-breakers.md) – Connection resilience

---

**Status**: ✅ Complete. Covers polling, SSE, WebSocket, scaling, patterns.

