# Orchestration & Kubernetes

Container orchestration, deployment, scaling, and cluster management.

---

## TL;DR

- **Container**: Isolated application + dependencies
- **Orchestration**: Automate deployment, scaling, management
- **Kubernetes**: Industry standard (Docker, ECS alternatives)
- **Pods**: Smallest deployable unit (usually 1 container)
- **Scaling**: Auto-scale based on CPU/memory

---

## Kubernetes Concepts

### Pods (Containers)

```
Pod = one or more containers
Usually 1 container per pod (occasionally 2-3 for sidecars)

Example:
  apiVersion: v1
  kind: Pod
  metadata:
    name: payment-pod
  spec:
    containers:
    - name: payment
      image: payment-service:v1
      resources:
        limits:
          cpu: "500m"
          memory: "512Mi"
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
```

---

### Deployments

```
Deployment = ReplicaSet + rollout strategy

Manages:
  - 3 replicas of payment-service
  - Automatic restart if pod dies
  - Rolling updates (new version gradually)
  - Rollback if new version fails

Example:
  kubectl set image deployment/payment payment=payment:v2
  Kubernetes automatically rolls out (1 pod at a time)
```

---

### Auto-Scaling

```
HorizontalPodAutoscaler:
  If CPU > 80%: Add more pods
  If CPU < 20%: Remove pods
  
Example:
  Min pods: 2
  Max pods: 10
  Target CPU: 70%
  
Result:
  Traffic spike → Scale to 10 pods automatically
  Traffic drops → Scale back to 2 pods
```

---

## Service Mesh (Istio)

```
Sidecar proxy per pod:
  Payment container
  ↓ (via proxy)
  → UserService
  
Mesh benefits:
  - Automatic retries
  - Circuit breaking
  - Distributed tracing
  - Traffic management
  
Implementation: Istio (popular), Linkerd, Consul Connect
```

---

## Related Fundamentals

- [Service Discovery](service-discovery.md) – Kubernetes DNS
- [Microservices Patterns](microservices-patterns.md) – Design patterns
- [Reliability](../reliability-and-resiliency/) – Resilience in orchestration

---

**Status**: ✅ Complete.

