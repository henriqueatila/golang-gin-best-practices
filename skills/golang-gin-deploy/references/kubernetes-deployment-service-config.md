# Kubernetes — Deployment, Service, ConfigMap, and Secret

> Kubernetes patterns for Go Gin APIs. Not part of the Gin framework — mainstream cloud-native Go community patterns.

## Deployment Manifest

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0   # never reduce below desired replicas during update
      maxSurge: 1         # allow one extra pod during rollout
  template:
    metadata:
      labels:
        app: myapp
    spec:
      terminationGracePeriodSeconds: 30
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: myapp
                topologyKey: kubernetes.io/hostname
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:latest
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: myapp-config
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: jwt-secret
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
          # probes: see kubernetes-probes-hpa-ingress-pvc.md
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 65532
            capabilities:
              drop: ["ALL"]
      securityContext:
        seccompProfile:
          type: RuntimeDefault
```

---

## Service Manifest

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - name: http
      port: 80
      targetPort: http
```

| Type | Use case |
|------|----------|
| `ClusterIP` | Internal only — use with Ingress |
| `NodePort` | Dev/testing only |
| `LoadBalancer` | Cloud provider external LB |

---

## ConfigMap and Secret

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  PORT: "8080"
  GIN_MODE: "release"
  READ_TIMEOUT: "10s"
  WRITE_TIMEOUT: "10s"
  SHUTDOWN_TIMEOUT: "30s"
```

```yaml
# k8s/secret.yaml — values must be base64-encoded
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
  namespace: myapp
type: Opaque
data:
  database-url: <base64>
  jwt-secret: <base64>
```

**Critical:** Never commit real Secret values. Use Vault, AWS Secrets Manager, or External Secrets Operator.

```bash
# Preferred: create secret from CLI (avoids base64 in files)
kubectl create secret generic myapp-secret \
  --from-literal=database-url="postgres://..." \
  --from-literal=jwt-secret="..." \
  --namespace=myapp
```
> For probes, HPA, Ingress, PVC, PDB, NetworkPolicy: see [kubernetes-probes-hpa-ingress-pvc.md](kubernetes-probes-hpa-ingress-pvc.md). For complete manifests, Helm, and CI/CD: see [kubernetes-complete-helm-cicd.md](kubernetes-complete-helm-cicd.md).
