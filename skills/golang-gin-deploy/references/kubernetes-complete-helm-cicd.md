# Kubernetes — Complete Manifests, Helm Chart, and GitHub Actions CI/CD

## Complete Manifests

```bash
kubectl create namespace myapp
kubectl apply -f k8s/ --namespace=myapp
kubectl rollout status deployment/myapp --namespace=myapp
kubectl get pods --namespace=myapp
kubectl logs -l app=myapp --namespace=myapp --tail=100 -f
kubectl describe pod -l app=myapp --namespace=myapp
```

**Recommended `k8s/` directory layout:** `namespace.yaml`, `configmap.yaml`, `secret.yaml` (gitignored), `deployment.yaml`, `service.yaml`, `ingress.yaml`, `hpa.yaml`, `pdb.yaml`, `netpol.yaml`, `migrate-job.yaml`, and `postgres/` subtree (if self-hosting).

**Migration Job** (`k8s/migrate-job.yaml`) — run before deploying a new app version. Uses `migrate/migrate:v4` image, passes `-path=/migrations -database=$(DATABASE_URL) up`, reads `DATABASE_URL` from secret. For Helm, name the Job `db-migrate-{{ .Values.image.tag }}`; for raw kubectl use `generateName: db-migrate-`.

See the **golang-gin-database** skill (`references/migrations.md`) for zero-downtime migration patterns.

---

## Helm Chart Structure

Helm templates manifests with values for environment-specific config without duplicating YAML.

```
charts/myapp/
├── Chart.yaml           # chart metadata (apiVersion: v2, name, description, version, appVersion)
├── values.yaml          # image repo/tag, replicaCount, service, ingress, resources, autoscaling, config
├── values.prod.yaml     # production overrides
├── templates/
│   ├── _helpers.tpl; deployment.yaml; service.yaml; ingress.yaml; configmap.yaml; hpa.yaml; migrate-job.yaml
└── .helmignore
```

Key `values.yaml` fields: `image.repository/tag/pullPolicy`, `replicaCount: 2`, `service.type: ClusterIP`, `ingress.enabled/className/host/tls`, `resources.requests/limits`, `autoscaling.minReplicas/maxReplicas/targetCPUUtilization`, `config.ginMode/port/migrationsPath`.

```bash
helm install myapp ./charts/myapp --namespace=myapp --create-namespace \
  --values=charts/myapp/values.prod.yaml --set image.tag=v1.2.3
helm upgrade myapp ./charts/myapp --namespace=myapp \
  --values=charts/myapp/values.prod.yaml --set image.tag=v1.2.4
helm rollback myapp 1 --namespace=myapp
```

---

## GitHub Actions CI/CD Workflow

Pipeline: `push to main → test → build → migrate → deploy`. File: `.github/workflows/deploy.yml`.

**`test` job:** `actions/setup-go@v5` (go 1.24, cache: true) → `go test -v -race -cover ./...`

**`build` job** (needs: test): Login to `ghcr.io` via `docker/login-action@v3` → extract tags with `docker/metadata-action@v5` (sha-, branch, semver) → build + push with `docker/build-push-action@v6` (GHA cache, `VERSION` and `BUILD_TIME` build-args) → scan with `aquasecurity/trivy-action` (CRITICAL,HIGH, exit-code: 1).

**`migrate` job** (needs: build, environment: production): Install `golang-migrate` CLI → `migrate -path db/migrations -database "$DATABASE_URL" up`.

**`deploy` job** (needs: [build, migrate], environment: production): `azure/setup-kubectl@v4` → decode base64 `KUBECONFIG` secret → `kubectl set image deployment/myapp myapp=<image>:<sha-tag>` → `kubectl rollout status --timeout=5m` → verify pods.

**Required GitHub secrets:** `DATABASE_URL` (PostgreSQL DSN for migrations), `KUBECONFIG` (base64-encoded kubeconfig).

> For Deployment, Service, ConfigMap, and Secret manifests: see [kubernetes-deployment-service-config.md](kubernetes-deployment-service-config.md).
> For Probes, HPA, Ingress, PVC, PDB, and NetworkPolicy: see [kubernetes-probes-hpa-ingress-pvc.md](kubernetes-probes-hpa-ingress-pvc.md).
