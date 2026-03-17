# Dockerfile — Image Size Optimization, .dockerignore, and Complete Production Dockerfile

## Image Size Optimization

**`-ldflags="-s -w"`** — strips symbol table (~30%) and DWARF debug info (~10%):
```bash
go build -ldflags="-s -w" ...
```

**`-trimpath`** — removes local filesystem paths from the binary; improves reproducibility:
```bash
go build -trimpath ...
```

**`upx` compression** (optional tradeoff — ~50–70% reduction, +100–300ms startup):
```dockerfile
FROM builder AS compressor
RUN apt-get install -y upx-ucl && upx --best /app/server
```
Use only for extremely size-constrained environments.

**Size comparison for a typical Gin API:**

| Configuration | Binary | Final image (distroless) |
|---------------|--------|--------------------------|
| Default build | ~12 MB | ~14 MB |
| `-ldflags="-s -w"` | ~8 MB | ~10 MB |
| `-ldflags="-s -w"` + UPX | ~3 MB | ~5 MB |

---

## .dockerignore

```dockerignore
.git; .gitignore; .github
.air.toml; air.toml; .golangci.yml; Makefile
.env; .env.local; .env.*
!.env.example
docker-compose*.yml; Dockerfile*
*.md; docs/; plans/
*_test.go; coverage.html; coverage.out; *.test
.vscode/; .idea/; *.swp
```

**Critical:** always exclude `.env` — even if not explicitly `COPY`'d, it exists in the build context and could be accidentally included by a future broad `COPY . .`.

---

## Complete Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
# docker build --build-arg VERSION=$(git describe --tags --always) -t myapp:latest .

# Stage 1: Dependency cache
FROM golang:1.24-bookworm AS deps
WORKDIR /build
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download && go mod verify

# Stage 2: Builder
FROM deps AS builder
ARG VERSION=dev
ARG BUILD_TIME
ARG TARGETARCH
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build \
    -ldflags="-s -w -X main.Version=${VERSION} -X main.BuildTime=${BUILD_TIME}" \
    -trimpath -o /app/server ./cmd/api

# Stage 3: Runtime
FROM gcr.io/distroless/static-debian12:nonroot
LABEL org.opencontainers.image.source="https://github.com/myorg/myapp"
LABEL org.opencontainers.image.description="Gin REST API"
LABEL org.opencontainers.image.licenses="MIT"
COPY --from=builder /app/server /server
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD ["/server", "-health-check"]
ENTRYPOINT ["/server"]
```

```bash
# Build
docker build \
  --build-arg VERSION=$(git describe --tags --always) \
  --build-arg BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t myapp:latest .

# Run
docker run --rm -p 8080:8080 \
  -e DATABASE_URL="postgres://user:pass@host:5432/db?sslmode=require" \
  -e JWT_SECRET="your-secret" -e GIN_MODE=release \
  myapp:latest

# Inspect size
docker images myapp:latest
```

> For multi-stage build and base images: see [dockerfile-multistage-and-base-images.md](dockerfile-multistage-and-base-images.md).
> For build args, secrets, non-root user, and HEALTHCHECK: see [dockerfile-build-args-security-and-healthcheck.md](dockerfile-build-args-security-and-healthcheck.md).
