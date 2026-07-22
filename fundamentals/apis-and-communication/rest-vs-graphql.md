# REST vs GraphQL

Understanding API design philosophies, their trade-offs, and when to use each.

---

## TL;DR

- **REST**: Simple, cacheable, stateless, but requires multiple requests (over-fetching/under-fetching)
- **GraphQL**: Flexible queries, single request, but complex to implement and cache
- **REST for**: Public APIs, caching importance, simple CRUD operations
- **GraphQL for**: Complex, interconnected data, mobile clients (reduce requests), internal APIs
- **Hybrid**: Many companies use both (REST for public, GraphQL internally)

---

## REST Principles

### Core Concepts

```
Resource-oriented:
  Users → /users
  Specific user → /users/:id
  User posts → /users/:id/posts
  
HTTP methods:
  GET /users/:id → Fetch user
  POST /users → Create user
  PUT /users/:id → Update user
  DELETE /users/:id → Delete user

Stateless:
  Server doesn't store client context
  Each request contains all info needed
  
Cacheable:
  HTTP caching headers (Cache-Control, ETag, Last-Modified)
  Proxies and CDNs can cache responses
```

---

### Example: REST for Blog

```
GET /posts/:id
Response:
  {
    id: 1,
    title: "REST vs GraphQL",
    body: "...",
    authorId: 5,
    commentIds: [10, 11, 12, 13, ...]
  }

Limitations:
  - Over-fetching: Got title/body but only needed title
  - Under-fetching: Got authorId but need author name
    Must do: GET /users/5 (second request)
    Then: GET /comments/10, /11, /12, /13 (four more requests)
    Total: 6 API calls for one blog post!
```

---

## GraphQL Basics

### Query Model

```
Client sends query:
  query {
    post(id: 1) {
      title
      author {
        name
        email
      }
      comments {
        text
        author { name }
      }
    }
  }

Server returns exactly what was asked:
  {
    post: {
      title: "REST vs GraphQL",
      author: { name: "Alice", email: "alice@..." },
      comments: [
        { text: "Great post!", author: { name: "Bob" } },
        { text: "Disagree", author: { name: "Carol" } },
      ]
    }
  }

Result: 1 request, exactly the data needed!
```

---

### Advantages

```
Single request:
  No over-fetching (get only needed fields)
  No under-fetching (nest related data in one query)
  
Strongly typed:
  Schema defines all possible queries/mutations
  Client can validate query before sending
  
Introspection:
  Client can discover API capabilities
  Self-documenting (GraphQL playground shows schema)
```

---

## REST vs GraphQL: Trade-offs

| Aspect | REST | GraphQL |
|---|---|---|
| **Simplicity** | Simple (HTTP methods) | Complex (query language) |
| **Requests** | Many (N+1 problem) | One (resolved) |
| **Caching** | Built-in (HTTP caching) | Hard (query-based, not URL-based) |
| **Bandwidth** | Over-fetch, larger payloads | Exact fields, smaller payloads |
| **Server load** | Spread across multiple endpoints | Single endpoint, complex query evaluation |
| **Debugging** | Easy (URLs, HTTP methods) | Hard (complex query structures) |
| **Versioning** | Easy (v1, v2 in URL) | Hard (schema evolution) |
| **Monitoring** | Simple (per-endpoint metrics) | Complex (per-field metrics) |

---

## REST Caching Strategy

```
HTTP caching headers:

GET /users/:id
Response:
  Cache-Control: public, max-age=3600
  ETag: "abc123"
  Last-Modified: 2026-01-01T00:00:00Z
  
Client behavior:
  First request: Fetch and cache for 1 hour
  Subsequent requests: Serve from cache (if not expired)
  After 1 hour: Validate with server (If-None-Match header)
  
Result: Zero requests if cached, one request if stale
```

---

## GraphQL Caching Challenges

```
Problem: Query as cache key
  Same data, different queries = different cache entries
  
Query 1:
  query { post(id: 1) { title author { name } } }
  Cache key: ?
  
Query 2:
  query { post(id: 1) { title author { name email } } }
  Different query = different cache key
  But overlaps significantly with Query 1
  
Solution attempts:
  1. HTTP caching (query in POST body, can't cache)
  2. Manual field-level caching (complex)
  3. Apollo client caching (application-level, not standard)
  4. Accept non-cached (let application handle)
```

---

## GraphQL Query Complexity & Denial of Service

```
Naive GraphQL query:
  query {
    users {
      posts {
        comments {
          author {
            posts {
              comments { ... }
            }
          }
        }
      }
    }
  }
  
This can:
  - Spawn exponential queries (fetch 1M users, each has 100 posts, etc.)
  - Crash server with runaway query execution
  - Become DDoS vector

Mitigation:
  1. Query depth limits (max nesting 3 levels)
  2. Query complexity scoring (complex fields cost more)
  3. Rate limiting per user
  4. Query timeouts
  5. Analyze query before execution
```

---

## REST Caveats: Over-Fetching

```
Mobile app needs: User name, avatar URL only

REST endpoint: GET /users/:id
Response: { id, name, email, phone, address, company, preferences, ... }
  Size: 5KB (when only needed 200 bytes)
  
Bandwidth waste: 5KB vs 200B = 25x overhead
On 1M requests: 5GB vs 200MB = 4.8GB wasted!

Solutions:
  1. GraphQL (exact fields)
  2. Sparse fieldsets (/users/:id?fields=name,avatar_url)
  3. Custom mobile endpoints (separate REST endpoints for mobile)
```

---

## REST Best Practices

### Hypermedia (HATEOAS)

```
Response includes links to related resources:

GET /posts/1
Response: {
  id: 1,
  title: "...",
  links: {
    self: "/posts/1",
    author: "/users/5",
    comments: "/posts/1/comments",
    next: "/posts/2"
  }
}

Benefit: Client discovers API dynamically
Drawback: Verbose, less common in practice
```

---

### Pagination

```
GET /users?page=1&limit=50
Response: {
  data: [...50 users...],
  pagination: {
    total: 1000,
    page: 1,
    limit: 50,
    hasNext: true,
    nextCursor: "abc123"
  }
}

Cursor-based (better for large datasets):
  GET /users?cursor=abc123&limit=50
  More stable with insertions/deletions
```

---

## GraphQL Best Practices

### Field-Level Security

```
type User {
  id: ID!
  name: String!
  email: String! @auth(requires: OWNER)
  admin: Boolean! @auth(requires: ADMIN)
}

Query:
  query { users { id name email } }
  
Security check per field:
  id, name: Public
  email: Only if user == owner
  admin: Only if requester is admin
  
Result: Return partial data based on permissions
```

---

### Mutations for State Changes

```
REST:
  POST /transfer
  { from: 100, to: 200, amount: 50 }
  
GraphQL:
  mutation {
    transferMoney(from: 100, to: 200, amount: 50) {
      success
      transaction { id amount timestamp }
      error
    }
  }

GraphQL advantage:
  - Clear what changed (mutation semantics)
  - Response includes result of mutation
  - Nested data in single roundtrip
```

---

## Decision Matrix

| Scenario | Recommendation | Rationale |
|---|---|---|
| **Public API** | REST | Easy to cache, well-understood, tooling |
| **Mobile app** | GraphQL | Reduce requests, exact fields, bandwidth savings |
| **Simple CRUD** | REST | Simpler to implement and operate |
| **Complex interconnected data** | GraphQL | Single request for related data |
| **Internal microservices** | gRPC or REST | Performance > flexibility |
| **Mixed usage** | Both | REST public, GraphQL for web/mobile |

---

## Hybrid Approach

```
Platform:
  Public API: REST (caching, simplicity)
  Web app: GraphQL (complex queries, web performance)
  Mobile app: Custom GraphQL (exact fields needed)
  Backend-to-backend: gRPC (performance)
  
Benefit: Choose best tool for each use case
Cost: Multiple APIs to maintain
```

---

## Related Fundamentals

- [Caching](../caching/) – REST caching vs GraphQL challenges
- [Reliability](../reliability-and-resiliency/) – Query complexity DoS prevention
- [Load Balancing](../scalability-and-load-balancing/) – Query distribution

---

**Status**: ✅ Complete. Covers REST/GraphQL trade-offs, caching, complexity, best practices.

