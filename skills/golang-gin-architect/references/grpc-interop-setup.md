# gRPC Interoperability — When to Use and Project Setup

**Default stance:** Do not add gRPC unless you have a real need. REST with Gin is sufficient for most projects. gRPC adds protobuf compilation, code generation, and operational complexity.

**Cost: HIGH (4/5).** Only justified when REST genuinely cannot meet requirements.

---

## When to Use gRPC

```
START: Do you need gRPC?
  ├── All clients are web/mobile? → No. REST with Gin. Stop.
  └── Service-to-service communication?
      ├── < 5 services, simple payloads? → REST is fine. Stop.
      └── High throughput, strict contracts, streaming needed?
          ├── Streaming (bidirectional)? → gRPC. Worth the complexity.
          └── Just strict contracts + perf? → gRPC, but consider
                gRPC-Gateway for external REST consumers.
```

**You DO need gRPC when:** bidirectional or server-side streaming; tight latency budgets (< 10ms p99); strict schema contracts enforced at build time; 10+ internal services.

**You do NOT need gRPC when:** all consumers are browsers or mobile apps; fewer than 5 services; team unfamiliar with protobuf toolchain; no existing gRPC infrastructure.

---

## Project Structure

```
myapp/
├── cmd/
│   ├── api/main.go          # HTTP entry point (Gin)
│   └── server/main.go       # Combined HTTP + gRPC entry point
├── internal/user/
│   ├── domain/user.go       # Domain types (shared)
│   ├── usecase/             # Business logic (shared by both handlers)
│   ├── handler/             # Gin HTTP handlers
│   ├── grpchandler/         # gRPC handlers (calls same usecase)
│   └── repository/
├── proto/
│   ├── buf.yaml
│   ├── buf.gen.yaml
│   └── user/v1/user.proto
└── gen/                     # Generated protobuf Go code (do not edit)
    └── user/v1/
```

Key principle: both `handler/` (HTTP) and `grpchandler/` (gRPC) import from `usecase/`. Business logic lives once.

### buf.yaml

```yaml
version: v2
modules:
  - path: proto
deps:
  - buf.build/googleapis/googleapis
```

### buf.gen.yaml

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt:
      - paths=source_relative
```

Generate with: `buf generate`

---

## Shared Service Layer

Both handlers call the same `UserService`. Business logic is never duplicated.

```go
// internal/user/usecase/user_service.go
type UserService struct {
    repo UserRepository
    log  *slog.Logger
}

func (s *UserService) GetByID(ctx context.Context, id string) (*domain.User, error) {
    return s.repo.FindByID(ctx, id)
}
```

```go
// internal/user/handler/user_handler.go — HTTP (Gin)
func (h *UserHandler) GetByID(c *gin.Context) {
    id := c.Param("id")
    user, err := h.svc.GetByID(c.Request.Context(), id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"id": user.ID, "name": user.Name})
}
```

```go
// internal/user/grpchandler/user_handler.go — gRPC
func (h *UserGRPCHandler) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    user, err := h.svc.GetByID(ctx, req.GetId())
    if err != nil {
        return nil, err // wrap with grpc/status codes in production
    }
    return &pb.GetUserResponse{Id: user.ID, Name: user.Name}, nil
}
```

---

## See Also

- `grpc-interop-server-client.md` — gRPC server setup, cmux multiplexer, client, gateway, Docker
