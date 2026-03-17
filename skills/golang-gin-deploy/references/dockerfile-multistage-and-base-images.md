# Dockerfile — Multi-Stage Build and Base Image Comparison

> Docker patterns are not part of the Gin framework — mainstream Go community patterns.

## Multi-Stage Build Explained

Multiple `FROM` instructions in one Dockerfile. Each stage is independent — only explicitly copied artifacts carry forward. The Go toolchain (~800 MB) stays in the builder stage; only the compiled binary (~8 MB) crosses to the runtime image (~10 MB total).

```
Stage 1 (builder)          Stage 2 (runtime)
─────────────────          ─────────────────
golang:1.24 (~800 MB)      distroless (~2 MB)
+ source code              + /server binary (~8 MB)
+ go modules               ──────────────────────
+ compiled binary          Final image: ~10 MB
```

Key directive: `COPY --from=builder /app/server /server` — source code, module cache, and Go toolchain are discarded.

---

## Base Image Comparison

| Image | Size | Shell | Use case |
|-------|------|-------|----------|
| `gcr.io/distroless/static-debian12` | ~2 MB | No | Production (recommended) |
| `gcr.io/distroless/base-debian12` | ~20 MB | No | When CGO is required |
| `alpine:3.21` | ~7 MB | sh | When shell access needed |
| `scratch` | 0 MB | No | Smallest; no TLS certs or timezone data |

**Distroless (recommended):** No shell = no shell injection attacks. Includes CA certificates and timezone data. `nonroot` tag runs as UID 65532 — no `USER` directive needed. Requires `CGO_ENABLED=0` in builder.

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
```

**Alpine** — use when you need shell for debugging or `apk` to install runtime deps:

```dockerfile
FROM alpine:3.21
RUN apk add --no-cache ca-certificates tzdata && \
    addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=builder /app/server /app/server
ENTRYPOINT ["/app/server"]
```

**Scratch** — absolute minimum size; requires manually copying TLS certs (`/etc/ssl/certs/ca-certificates.crt`); no timezone data (`time.LoadLocation` fails without explicit tz copy).

---

## Layer Caching Optimization

Docker caches each layer. A cache miss invalidates all subsequent layers.

```dockerfile
# BAD — module download re-runs on every source change
COPY . .
RUN go mod download

# GOOD — modules cached until go.mod/go.sum change
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build ...
```

**BuildKit cache mount** — keeps module and build caches across builds on the same host (~5–10x faster rebuilds):

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o /app/server ./cmd/api
```

> For build args, secrets, non-root user, and HEALTHCHECK: see [dockerfile-build-args-security-and-healthcheck.md](dockerfile-build-args-security-and-healthcheck.md).
> For size optimization, .dockerignore, and complete Dockerfile: see [dockerfile-size-optimization-and-complete-example.md](dockerfile-size-optimization-and-complete-example.md).
