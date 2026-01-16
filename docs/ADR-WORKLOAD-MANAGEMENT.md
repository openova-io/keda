# ADR: Workload Management (KEDA + VPA)

**Status:** Accepted
**Date:** 2024-06-01

## Context

Need autoscaling based on both event-driven metrics and resource utilization.

## Decision

Use KEDA for event-driven scaling and VPA for resource right-sizing.

## Rationale

### KEDA vs HPA

| Feature | HPA | KEDA |
|---------|-----|------|
| CPU/Memory scaling | ✅ | ✅ |
| Custom metrics | Limited | ✅ |
| Event-driven | ❌ | ✅ |
| Scale-to-zero | ❌ | ✅ |
| Queue-based | ❌ | ✅ |

### VPA Benefits

- Automatic resource recommendation
- Right-sizing for cost optimization
- Works with KEDA (complementary)

**Key Decision Factors:**
- Scale-to-zero for cost savings
- Queue-based scaling for workers
- Resource optimization via VPA

## KEDA Configuration

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: <tenant>-worker
  namespace: <tenant>-prod
spec:
  scaleTargetRef:
    name: <tenant>-worker
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: redis
      metadata:
        address: dragonfly.databases:6379
        listName: <tenant>-job-queue
        listLength: "5"
```

## VPA Configuration

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: <tenant>-api-vpa
  namespace: <tenant>-prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <tenant>-api
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2
          memory: 4Gi
```

## Scaling Patterns

| Workload | Strategy | Scaler |
|----------|----------|--------|
| API | Request-based | KEDA (Prometheus) |
| Workers | Queue-based | KEDA (Redis/Redpanda) |
| Background | Cron-based | KEDA (Cron) |
| All | Right-sizing | VPA |

## Consequences

**Positive:** Scale-to-zero, event-driven, cost optimization
**Negative:** Additional components, complexity

## Related

- [SPEC-AUTOSCALING-CONFIGURATION](./SPEC-AUTOSCALING-CONFIGURATION.md)
