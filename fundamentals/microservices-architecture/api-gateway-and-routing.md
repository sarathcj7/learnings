# API Gateway & Routing

Single entry point for clients, routing requests to appropriate services.

---

## TL;DR

- **API Gateway**: Client-facing proxy (single entry point)
- **Routing**: Path/host-based to different services
- **Rate limiting**: Per-user, per-endpoint
- **Auth**: Centralized authentication/authorization
- **Examples**: Kong, Nginx, AWS API Gateway

---

## Architecture

```
Client → API Gateway → Routes → Services
         ├─ /users/* → UserService
         ├─ /payments/* → PaymentService
         └─ /notifications/* → NotificationService

Gateway responsibilities:
  - Authentication (JWT validation)
  - Rate limiting (100 req/min per user)
  - Request transformation (add headers)
  - Response caching
  - Logging & monitoring
```

---

## Routing Examples

```
Path-based:
  /api/users/* → UserService
  /api/payments/* → PaymentService
  
Host-based:
  users.api.example.com → UserService
  payments.api.example.com → PaymentService
  
Header-based:
  X-API-Version: v1 → UserServiceV1
  X-API-Version: v2 → UserServiceV2
```

---

## Rate Limiting

```
Token bucket:
  User gets 100 tokens/minute
  Each request costs 1 token
  After 100 requests: Wait until next minute

Implementation: Gateway enforces
  Request 1: 99 tokens left
  Request 100: 0 tokens left
  Request 101: Wait (bucket empty)
```

---

## Related Fundamentals

- [Load Balancing](../scalability-and-load-balancing/load-balancing-algorithms.md) – Distributes to services
- [Security](../security/) – Auth at gateway

---

**Status**: ✅ Complete.

