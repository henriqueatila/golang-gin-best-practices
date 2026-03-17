# Dockerfile — Build Arguments, Secrets, Non-Root User, and HEALTHCHECK

## Build Arguments and Secrets

**Build arguments** (non-sensitive, embedded in image — visible in `docker history`):

```dockerfile
ARG VERSION=dev
ARG BUILD_TIME

RUN CGO_ENABLED=0 go build \
    -ldflags="-s -w -X main.Version=${VERSION} -X main.BuildTime=${BUILD_TIME}" \
    -o /app/server ./cmd/api
```

```bash
docker build \
  --build-arg VERSION=$(git describe --tags --always) \
  --build-arg BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t myapp:latest .
```

Expose in the health endpoint:

```go
var Version = "dev"; var BuildTime = "unknown"  // set via -ldflags

func (h *HealthHandler) Check(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"status": "ok", "version": Version, "build_time": BuildTime})
}
```

**Secrets** — use BuildKit secret mount; never `ARG` for sensitive values:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=github_token \
    GITHUB_TOKEN=$(cat /run/secrets/github_token) \
    GONOSUMCHECK=github.com/myorg/* go mod download
```

```bash
docker build --secret id=github_token,src=~/.github_token -t myapp .
```

**Warning:** `ARG` values are visible in `docker history` and image metadata — never use for secrets.

---

## Non-Root User

Running as root in a container risks host root access on container escape.

**Distroless:nonroot** — nothing extra needed; already runs as UID 65532:

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

**Alpine** — create a dedicated user:

```dockerfile
FROM alpine:3.21
RUN apk add --no-cache ca-certificates tzdata && \
    addgroup -S appgroup && adduser -S -G appgroup -u 10001 appuser
COPY --from=builder /app/server /app/server
RUN chown appuser:appgroup /app/server
USER appuser
ENTRYPOINT ["/app/server"]
```

If the app writes files (e.g., temp uploads), ensure target directories are owned by the app user or use a volume with appropriate permissions.

---

## HEALTHCHECK Instruction

`HEALTHCHECK` tells Docker Engine and docker-compose how to test container health. Kubernetes ignores it — K8s uses `livenessProbe`/`readinessProbe` instead.

**Problem:** distroless has no shell and no `curl`/`wget`. Embed the health check in the main binary:

```go
// cmd/api/main.go
func main() {
    healthCheck := flag.Bool("health-check", false, "run health check and exit")
    flag.Parse()
    if *healthCheck {
        url := fmt.Sprintf("http://localhost:%s/health", os.Getenv("PORT"))
        resp, err := http.Get(url)
        if err != nil { os.Exit(1) }
        defer resp.Body.Close()
        if resp.StatusCode != http.StatusOK { os.Exit(1) }
        os.Exit(0)
    }
    // ... normal server startup
}
```

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD ["/server", "-health-check"]
```

> For multi-stage build and base images: see [dockerfile-multistage-and-base-images.md](dockerfile-multistage-and-base-images.md).
> For size optimization, .dockerignore, and complete Dockerfile: see [dockerfile-size-optimization-and-complete-example.md](dockerfile-size-optimization-and-complete-example.md).
