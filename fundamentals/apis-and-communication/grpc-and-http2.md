# gRPC & HTTP/2

High-performance RPC framework using HTTP/2 multiplexing and protocol buffers. Where REST becomes too slow.

---

## TL;DR

- **gRPC**: Binary protocol, multiplexing, bidirectional streaming, 10x faster than REST
- **HTTP/2**: Multiplexing (multiple requests per connection), server push, compression
- **Protocol Buffers**: Compact binary format, schema-driven, smaller than JSON
- **Use for**: Microservices, real-time, high throughput
- **Don't use for**: Public APIs (REST better), browser clients (limited support)

---

## Why gRPC Exists

### REST Limitations

```
Scenario: Fetch user with 100 posts and 1000 comments

REST approach 1 (nested response):
  GET /users/123?include=posts,comments
  Response: { user: { id, name, posts: [...], comments: [...] } }
  Size: 500KB (large payload, slow to parse)
  
REST approach 2 (separate requests):
  GET /users/123
  GET /users/123/posts
  GET /users/123/comments
  Total: 3 HTTP requests × latency = 150ms (3 × 50ms round-trip)

gRPC approach:
  Call GetUserWithPostsAndComments(123)
  Single request, multiplexed on same connection
  Binary protocol: 50KB (compressed)
  Total: 50ms (one roundtrip, tiny payload)
  
Improvement: 3-10x faster!
```

---

## HTTP/2 Multiplexing

### HTTP/1.1 (Old)

```
Connection 1:
  Request 1: GET /users/1 → wait 50ms for response
  Request 2: GET /users/2 → wait 50ms (blocked until Request 1 complete)
  Request 3: GET /users/3 → wait 50ms
  Total: 150ms
  
Solution: Open 6 connections in parallel
  Each request goes to different connection
  Result: ~50ms total (but wasteful, 6 connections)
```

---

### HTTP/2 (Multiplexing)

```
Single connection with streams:
  Stream 1: GET /users/1 → wait 50ms
  Stream 2: GET /users/2 (sent before Stream 1 completes)
  Stream 3: GET /users/3 (sent immediately)
  
Result: All 3 in flight simultaneously
  One connection, interleaved frames
  Total: 50ms (not 150ms!)
  
Benefit: Lower latency, fewer connections, better resource usage
```

---

## Protocol Buffers

### Compact Binary Format

```
REST (JSON):
  {
    "id": 123,
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30
  }
  Size: 75 bytes
  
gRPC (Protocol Buffer):
  message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
  }
  
  Encoded: \x08\x7b\x12\x05Alice\x1a\x12alice@example.com\x20\x1e
  Size: 38 bytes (50% smaller!)
  
At 1M requests: 75MB vs 38MB = 37MB bandwidth savings
```

---

### Schema Evolution

```
Version 1:
  message User {
    int32 id = 1;
    string name = 2;
  }

Version 2 (backward compatible):
  message User {
    int32 id = 1;
    string name = 2;
    string email = 3;  // New field
  }
  
Old clients (Version 1) can:
  - Parse responses from new server (ignore unknown field 3)
  - Send requests to new server (just don't include email)
  
Result: No breaking changes, smooth migration
```

---

## gRPC Capabilities

### Unary (Request-Response)

```
Client sends one request, server sends one response:

Client:
  CreateUser(User{name: "Alice"})
  
Server:
  → User{id: 123, name: "Alice", created: now()}
  
Equivalent to: POST /users in REST
```

---

### Server Streaming

```
Server sends multiple messages for one request:

Client:
  GetTopPosts(limit: 1000)
  
Server responds in stream:
  → Post{id: 1, title: "..."}
  → Post{id: 2, title: "..."}
  → Post{id: 3, title: "..."}
  ... (1000 posts)
  
Result: Client receives all posts as they're sent
Not: Buffered in JSON array (would need large buffer)
```

---

### Client Streaming

```
Client sends multiple messages, server responds once:

Client:
  → Log{timestamp: ..., level: "INFO"}
  → Log{timestamp: ..., level: "WARN"}
  → Log{timestamp: ..., level: "ERROR"}
  ... (1000 logs)
  
Server:
  (receives all logs)
  → UploadResult{processed: 1000, errors: 0}
  
Use case: Batch uploads, log streaming
```

---

### Bidirectional Streaming

```
Client and server send messages to each other:

Chat application:
  Client → Server: "Hello"
  Server → Client: "User 2 just said 'Hi'"
  Client → Server: "How are you?"
  Server → Client: "User 2 just said 'Good!'"
  ...
  
Result: Real-time bidirectional communication
Similar to: WebSockets, but with gRPC semantics
```

---

## Performance Comparison

```
REST (JSON over HTTP/1.1):
  Payload: 100KB
  Overhead: Headers, text parsing
  Time to first byte: 50ms
  Total request: 150ms

REST (JSON over HTTP/2):
  Payload: 100KB
  Overhead: Headers, text parsing (but multiplexed)
  Time to first byte: 50ms (shared connection)
  Total request: 80ms (2x improvement)

gRPC (protobuf over HTTP/2):
  Payload: 30KB (3x smaller)
  Overhead: Binary, efficient parsing
  Time to first byte: 20ms (faster parsing)
  Total request: 40ms (4x improvement over HTTP/1.1)
```

---

## gRPC vs REST Decision

| Factor | gRPC | REST |
|---|---|---|
| **Performance** | 10x faster | Baseline |
| **Caching** | Hard (binary) | Built-in HTTP caching |
| **Browser support** | Needs gRPC-web | Native |
| **Learning curve** | Medium (protobuf) | Simple |
| **Debugging** | Harder (binary format) | Easy (curl, Postman) |
| **Public API** | No | Yes |
| **Internal services** | Yes | Also yes, but slower |
| **Real-time** | Excellent (streaming) | Requires WebSocket |
| **Mobile** | Good (bandwidth savings) | OK |

---

## When to Use gRPC

```
Use gRPC when:
  ✓ Microservice-to-microservice (internal)
  ✓ High throughput needed (1000+ RPS)
  ✓ Low latency critical (real-time systems)
  ✓ Bidirectional streaming needed
  ✓ Bandwidth savings important (mobile, IoT)
  
Don't use gRPC when:
  ✗ Public API (use REST)
  ✗ Browser client (needs gRPC-web, complex)
  ✗ Need HTTP caching (REST better)
  ✗ Simple CRUD (REST simpler)
  ✗ Debugging is priority (REST easier)
```

---

## gRPC Drawbacks

### Debugging Complexity

```
REST:
  curl http://api.example.com/users/123
  { "id": 123, "name": "Alice" }
  Easy to debug!
  
gRPC:
  grpcurl -d '{"id": 123}' localhost:50051 example.User/GetUser
  Binary response, needs grpcurl tool
  Harder to debug!
```

---

### Caching

```
REST caching:
  GET /users/123 HTTP/1.1
  → Cache for 1 hour
  → Proxy/CDN can cache
  
gRPC caching:
  POST /grpc
  (binary payload in body, can't use HTTP cache headers)
  
Solution: Application-level caching only
```

---

## HTTP/2 in Practice

### HPACK Compression

```
REST request 1:
  GET /users/123 HTTP/1.1
  Host: api.example.com
  User-Agent: MyApp
  Accept: application/json
  Size: 100 bytes
  
REST request 2:
  GET /posts/456 HTTP/1.1
  Host: api.example.com
  User-Agent: MyApp
  Accept: application/json
  Size: 100 bytes
  
With HTTP/2 HPACK (header compression):
  Request 1: 100 bytes (new headers)
  Request 2: 10 bytes (only path differs, headers indexed)
  
Savings: 90 bytes per request, 10x improvement!
```

---

## gRPC Best Practices

### Use Streaming for Large Responses

```
❌ Bad (all in one response):
  GetAllUsers() → Stream of 1M users as array
  Server buffers all → OOM potential
  
✅ Good (server streaming):
  GetAllUsers() → Server streams one user at a time
  Client processes as it receives
  Constant memory usage
```

---

### Keep Payloads Small

```
❌ Large proto:
  message User {
    int32 id;
    string name;
    string email;
    string address;
    string phone;
    string bio;  // Megabyte-sized field
    repeated Post posts;  // 1000 posts
  }
  
✅ Separate calls:
  GetUser() → Basic fields
  GetUserPosts() → Separate request for posts
  GetUserBio() → Separate, load on demand
  
Result: Fast common path, slow optional paths
```

---

## Related Fundamentals

- [REST vs GraphQL](rest-vs-graphql.md) – API design comparison
- [Microservices](../microservices-architecture/) – gRPC primary use case
- [Reliability](../reliability-and-resiliency/) – Streaming failure handling

---

**Status**: ✅ Complete. Covers HTTP/2, protobuf, streaming, performance, trade-offs.

