# Pagination & Filtering

How to efficiently return large datasets without overwhelming the server or client. Critical for APIs serving millions of records.

---

## TL;DR

- **Offset-based pagination**: Simple but slow for large offsets (requires scanning)
- **Cursor-based pagination**: Fast, handles insertions well, standard for APIs
- **Keyset pagination**: Most efficient, works with indexes
- **Filtering**: Delegate to database (indexes), not application
- **Sorting**: On indexed columns only, avoid O(N log N) in application
- **Result size**: Cap at reasonable max (1000 items), prevent DoS

---

## Pagination Strategies

### Offset-Based (Limit-Offset)

```
GET /users?offset=1000&limit=50

Database query:
  SELECT * FROM users ORDER BY id LIMIT 50 OFFSET 1000
  
Problem:
  Offset 1000 means: Skip first 1000 rows, fetch next 50
  Database scans 1050 rows (O(N) overhead)
  
At scale:
  Offset 100,000: Scan 100,050 rows (slow!)
  Offset 1,000,000: Scan 1,000,050 rows (very slow!)

Memory:
  Server must hold 1050 rows in memory, sort/filter
  
API response includes:
  {
    data: [...50 users...],
    pagination: {
      offset: 1000,
      limit: 50,
      total: 5000000,
      hasNext: true
    }
  }
```

---

### Cursor-Based (Recommended)

```
First request:
  GET /users?limit=50
  
Response:
  {
    data: [...50 users...],
    pagination: {
      nextCursor: "user_999",  // ID of last user
      prevCursor: "user_950",  // ID of first user
      hasNext: true,
      limit: 50
    }
  }

Second request (next page):
  GET /users?cursor=user_999&limit=50
  
Database query:
  SELECT * FROM users WHERE id > 999 ORDER BY id LIMIT 50
  
Efficiency:
  No offset overhead (just "find rows after 999")
  Single index lookup + sequential scan (O(N) where N=50)
  Much faster than offset!
```

---

### Keyset Pagination (Most Efficient)

```
Requires: Clustered index or unique key

Query:
  SELECT * FROM users WHERE (user_id, created_at) > (999, '2026-01-01')
  ORDER BY user_id, created_at
  LIMIT 50
  
Efficiency:
  B-tree index on (user_id, created_at)
  Single index lookup (O(log N))
  No scanning required
  
Fastest approach:
  O(log N) to find starting point
  O(50) to fetch 50 rows
  Total: O(log N + 50)
```

---

## Cursor Implementation

### Encoding Cursor

```
Option 1: Base64 encode the ID
  last_id = 999
  cursor = base64("999") = "OTk5"
  
Option 2: Base64 encode JSON
  cursor = base64(json.dumps({"id": 999}))
  More flexible if cursor needs multiple fields

Decode on next request:
  cursor = "OTk5"
  last_id = base64_decode(cursor) = 999
  Query from there
```

---

### Backward Pagination

```
Response includes: prevCursor (previous page)
  GET /users?cursor=user_500&limit=50
  (fetches 50 before user_500)
  
Implementation:
  Query: WHERE id < 500 ORDER BY id DESC LIMIT 50
  Reverse results for display
  
Challenge:
  Expensive if going far back
  Solution: Don't support going back > 10 pages
```

---

## Filtering

### Naive Approach (Bad)

```
GET /users/all
Response: { data: [all 5M users] }

Client filters:
  Filter by age > 30 (2M users remain)
  Filter by country = "US" (500k users remain)
  
Problem:
  Fetched 5M users (huge payload)
  Filtering in client (expensive, slow, unreliable)
  
Response size: 500GB (if 100B per user)
Bandwidth: Unusable
```

---

### Proper Approach (Good)

```
GET /users?age_gt=30&country=US&limit=50

Server builds query:
  SELECT * FROM users 
  WHERE age > 30 AND country = 'US'
  LIMIT 50
  
Database optimization:
  Index on (age, country)
  B-tree lookup finds matching rows directly
  
Response: Only 50 results (1-5KB)
Bandwidth: Efficient
```

---

## Filtering Best Practices

### 1. Index Filtered Columns

```
Query:
  GET /posts?author_id=123&created_after=2026-01-01

Index needed:
  CREATE INDEX idx_posts_author_created 
  ON posts(author_id, created_at)
  
Without index:
  Full table scan of 100M posts (slow!)
  
With index:
  B-tree lookup: O(log N)
  Fetch: O(K) where K = matching posts
  Total: O(log N + K) (fast!)
```

---

### 2. Cap Filter Results

```
Don't allow:
  GET /posts?created_after=2000-01-01
  (Returns 26M posts, crashes server)

Cap maximum result set:
  If matches > 100,000: Return error
  Message: "Query too broad, add more filters"
  
Encourage pagination:
  "Use pagination to fetch in batches of 50"
```

---

### 3. Validate Filters

```
❌ Bad (SQL injection):
  GET /users?name={name}
  Query: SELECT * FROM users WHERE name = '{name}'
  
✓ Good (parameterized):
  GET /users?name={name}
  Query: SELECT * FROM users WHERE name = ?
  Bind: [name]
```

---

## Sorting

### Sorting Efficiently

```
Query:
  GET /users?sort_by=created_at&order=desc

Efficient:
  CREATE INDEX idx_users_created 
  ON users(created_at DESC)
  
  Database uses index directly
  No O(N log N) sorting in memory
  
Inefficient:
  No index on created_at
  Full table scan + in-memory sort
  At 100M users: Very slow!
```

---

### Multi-Field Sort

```
Query:
  GET /users?sort_by=age,name&order=asc,asc
  
Index needed:
  CREATE INDEX idx_users_age_name 
  ON users(age ASC, name ASC)
  
Implementation:
  Index ordered by (age, name)
  Scan index in order: Get all users sorted by age, then name
  Efficient!
```

---

## Result Size Limiting

### Cap Results

```
Default: GET /users → 50 results
Maximum: GET /users?limit=1000

Server logic:
  if limit > 1000:
    limit = 1000 (enforce max)
    
Don't allow: Uncapped queries
  GET /users → Returns 1M users (DoS!)
```

---

### Total Count

```
API response:
  {
    data: [...50 users...],
    pagination: {
      nextCursor: "...",
      total: 5000000  // Total matching
    }
  }

Cost of total count:
  SELECT COUNT(*) FROM users WHERE age > 30
  Requires scanning entire table (slow!)
  
Alternative: Don't include total
  {
    data: [...],
    hasMore: true,  // Just boolean, fast to compute
    nextCursor: "..."
  }
```

---

## API Design Examples

### REST with Cursor

```
GET /posts?cursor=post_1000&limit=50&sort_by=created_at&author_id=user_123

Response:
  {
    data: [
      { id: 1001, title: "...", author_id: 123 },
      ...
    ],
    pagination: {
      nextCursor: "post_1050",
      prevCursor: "post_999",
      hasNext: true,
      limit: 50
    }
  }
```

---

### GraphQL

```
query {
  posts(
    first: 50
    after: "post_1000"
    filter: { authorId: 123, createdAfter: "2026-01-01" }
    orderBy: CREATED_AT
  ) {
    edges {
      node { id title author { name } }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

---

## Trade-offs Summary

| Approach | Speed | Simplicity | Handles Insertions |
|---|---|---|---|
| **Offset** | Slow (O(N)) | Simple | Yes |
| **Cursor** | Fast (O(1)) | Medium | Yes (stable) |
| **Keyset** | Fastest (O(log N)) | Complex | Yes (most stable) |

---

## Pagination Checklist

- [ ] Use cursor-based pagination (not offset)
- [ ] Support filtering with indexed columns
- [ ] Enforce result size caps (max 1000 items)
- [ ] Index sort columns
- [ ] Support sorting by multiple fields
- [ ] Handle backward pagination carefully
- [ ] Document filtering syntax
- [ ] Validate input (prevent SQL injection)
- [ ] Return nextCursor for easy continuation
- [ ] Don't include total count (expensive to compute)

---

## Related Fundamentals

- [REST vs GraphQL](rest-vs-graphql.md) – Filtering in different API styles
- [Databases/Indexing](../databases/indexing.md) – Index design for filtering
- [Scalability](../scalability-and-load-balancing/) – Large dataset handling

---

**Status**: ✅ Complete. Covers offset/cursor/keyset, filtering, sorting, design patterns.

