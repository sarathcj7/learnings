# Service Discovery

Finding services dynamically in microservices architecture.

---

## TL;DR

- **Service registration**: Service announces itself (IP:port)
- **Service discovery**: Client looks up service location
- **Health checking**: Remove dead services automatically
- **Consul/etcd**: Popular service discovery tools
- **Kubernetes DNS**: Built-in service discovery

---

## Problem

```
Microservices:
  PaymentService: 192.168.1.10:8080
  UserService: 192.168.1.11:8080
  
Issue:
  IPs change (deployment, scaling)
  Hard-coded IPs break
  
Solution: Dynamic discovery
```

---

## Consul (Service Registry)

```
Service starts:
  POST /v1/agent/service/register
  {
    "Name": "payment-service",
    "ID": "payment-1",
    "Address": "192.168.1.10",
    "Port": 8080,
    "Check": {"HTTP": "http://...:8080/health", "Interval": "10s"}
  }
  
Client queries:
  GET /v1/catalog/service/payment-service
  Response: [{"Address": "192.168.1.10", "Port": 8080}, ...]
  
Health check:
  If /health returns error: Consul removes service
  Client never reaches dead service
```

---

## Kubernetes DNS

```
Service in Kubernetes:
  apiVersion: v1
  kind: Service
  metadata:
    name: payment-service
  spec:
    selector:
      app: payment
    ports:
    - port: 8080

Access from other pod:
  curl http://payment-service:8080/
  Kubernetes DNS resolves to service IP
  Load balancer distributes to pods
  
Auto-scaling:
  Pods added/removed automatically
  DNS always has latest list
```

---

## Client-Side vs Server-Side

### Client-Side (Consul)

```
Client queries Consul
Consul returns all healthy instances
Client picks one (load balance locally)
Direct call to instance

Pros: Simple, low latency
Cons: Client must handle load balancing
```

---

### Server-Side (Kubernetes)

```
Client calls Service (stable IP)
Kubernetes Service load balances to pods
Client doesn't know actual pod IP

Pros: Transparent, easy
Cons: Extra hop (service proxy)
```

---

## Related Fundamentals

- [API Gateway](api-gateway-and-routing.md) – Routes to services
- [Reliability](../reliability-and-resiliency/) – Health checks

---

**Status**: ✅ Complete.

