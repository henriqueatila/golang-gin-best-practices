# Kubernetes — Probes, HPA, Ingress, PVC, PDB, and NetworkPolicy

## Liveness and Readiness Probes

Both probes hit `/health` from the **golang-gin-api** skill's `HealthHandler`.

```yaml
livenessProbe:
  httpGet: { path: /health, port: http }
  initialDelaySeconds: 15; periodSeconds: 20; timeoutSeconds: 5; failureThreshold: 3

readinessProbe:
  httpGet: { path: /health, port: http }
  initialDelaySeconds: 5; periodSeconds: 10; timeoutSeconds: 3; failureThreshold: 3
```

| Probe | On failure | On success |
|-------|-----------|------------|
| `livenessProbe` | Restart container | Nothing |
| `readinessProbe` | Remove from Service endpoints | Add to endpoints |

**Split endpoints when DB check is slow:**

```go
r.GET("/health/live",  func(c *gin.Context) { c.JSON(http.StatusOK, gin.H{"status": "ok"}) })
r.GET("/health/ready", healthHandler.Check)
```

**Startup probe (slow-starting apps):** `startupProbe.httpGet.path: /health/live`, `failureThreshold: 30`, `periodSeconds: 10` — 5 min max startup. Runs first — prevents liveness from killing a legitimately slow-starting pod.

---

## Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
  namespace: myapp
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: myapp }
  minReplicas: 2; maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
    - type: Resource
      resource: { name: memory, target: { type: Utilization, averageUtilization: 80 } }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies: [{ type: Pods, value: 1, periodSeconds: 60 }]
    scaleUp:
      stabilizationWindowSeconds: 30
      policies: [{ type: Pods, value: 2, periodSeconds: 60 }]
```

`resources.requests` must be set in the Deployment for HPA to calculate utilization.

---

## Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: myapp-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /; pathType: Prefix
            backend: { service: { name: myapp, port: { name: http } } }
```

---

## PVC for PostgreSQL

Use a managed database (RDS, Cloud SQL) in production. For self-hosted, use a StatefulSet with `postgres:17-alpine`, `containerPort: 5432`, secret-backed `POSTGRES_PASSWORD`, `volumeMounts` at `/var/lib/postgresql/data`, and a `volumeClaimTemplate` requesting `10Gi` with `ReadWriteOnce` access mode.

Resources: `requests: { memory: 256Mi, cpu: 100m }`, `limits: { memory: 512Mi, cpu: 500m }`.

---

## PodDisruptionBudget

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: myapp-pdb, namespace: myapp }
spec:
  minAvailable: 1
  selector: { matchLabels: { app: myapp } }
```

PDB only protects against voluntary disruptions (kubectl drain, upgrades) — not crashes.

---

## NetworkPolicy

```yaml
# k8s/netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: myapp-netpol, namespace: myapp }
spec:
  podSelector: { matchLabels: { app: myapp } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { name: ingress-nginx } }
      ports: [{ port: 8080 }]
```

Label the namespace: `kubectl label namespace ingress-nginx name=ingress-nginx`. Requires a CNI plugin that supports NetworkPolicy (Calico, Cilium, Weave).

> For complete manifests, Helm chart, and GitHub Actions CI/CD: see [kubernetes-complete-helm-cicd.md](kubernetes-complete-helm-cicd.md).
