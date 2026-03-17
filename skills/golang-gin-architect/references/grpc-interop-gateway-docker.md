# gRPC Interoperability — Gateway and Docker

gRPC-Gateway REST proxy and Docker Compose setup. Companion to `grpc-interop-server-client.md`.

---

## gRPC-Gateway (REST Proxy)

Generates HTTP/JSON reverse proxy from protobuf annotations. Use when you need both gRPC (internal) and REST (external) from a single `.proto`.

### Proto Annotation

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/v1/users/{id}"
    };
  }
}
```

### Gateway Setup

```go
func newGateway(ctx context.Context, grpcAddr string) (http.Handler, error) {
    mux := runtime.NewServeMux()
    opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
    err := pb.RegisterUserServiceHandlerFromEndpoint(ctx, mux, grpcAddr, opts)
    return mux, err
}
```

Use gRPC-Gateway when internal gRPC services must also expose a public REST API. Avoid if you only need one protocol — it adds another layer to debug.

---

## Docker Compose with gRPC

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"    # HTTP (Gin)
      - "50051:50051"  # gRPC
    environment:
      - APP_ENV=development
    depends_on:
      - db
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
```

Multi-stage Dockerfile with buf generation:

```dockerfile
FROM bufbuild/buf:latest AS proto-gen
WORKDIR /workspace
COPY proto/ proto/
COPY buf.yaml buf.gen.yaml ./
RUN buf generate

FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY --from=proto-gen /workspace/gen ./gen
COPY . .
RUN go build -o server ./cmd/server

FROM alpine:3.21
COPY --from=builder /app/server /server
EXPOSE 8080 50051
CMD ["/server"]
```

---

## Quick Reference

| Item | Value |
|------|-------|
| Official gRPC package | `google.golang.org/grpc` |
| Protobuf management | `buf` (not raw `protoc`) |
| Multiplexer | `github.com/soheilhy/cmux` |
| REST proxy | `github.com/grpc-ecosystem/grpc-gateway/v2` |
| Generated code dir | `gen/` — never edit manually |
| Complexity cost | HIGH (4/5) |
