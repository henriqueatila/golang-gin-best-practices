# gRPC Interoperability — Server, Client, and Multiplexer

Companion to `grpc-interop-gateway-docker.md` (gRPC-Gateway REST proxy, Docker Compose).

---

## gRPC Server Setup

```go
func startGRPC(handler *grpchandler.UserGRPCHandler) {
    log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Error("failed to listen", "error", err)
        os.Exit(1)
    }

    grpcServer := grpc.NewServer(
        grpc.ChainUnaryInterceptor(loggingInterceptor(log)),
    )
    pb.RegisterUserServiceServer(grpcServer, handler)

    log.Info("gRPC server starting", "addr", ":50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Error("gRPC server failed", "error", err)
    }
}

func loggingInterceptor(log *slog.Logger) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        resp, err := handler(ctx, req)
        log.Info("grpc call", "method", info.FullMethod, "error", err)
        return resp, err
    }
}
```

---

## HTTP + gRPC Multiplexer (cmux)

Run HTTP and gRPC on the same port. Use when firewall rules restrict port exposure.

```go
import (
    "github.com/soheilhy/cmux"
    "google.golang.org/grpc"
)

func runCombinedServer(ginEngine *gin.Engine, grpcServer *grpc.Server) error {
    lis, err := net.Listen("tcp", ":8080")
    if err != nil {
        return err
    }

    m := cmux.New(lis)

    grpcL := m.MatchWithWriters(
        cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc"),
    )
    httpL := m.Match(cmux.Any())

    go grpcServer.Serve(grpcL)
    go http.Serve(httpL, ginEngine) //nolint:gosec

    return m.Serve()
}
```

Use cmux when single port is required (cloud firewall, load balancer). Use separate ports when you control infrastructure (simpler, preferred).

`go.mod` dependency: `github.com/soheilhy/cmux`

---

## gRPC Client (Service-to-Service)

```go
type UserClient struct {
    client pb.UserServiceClient
    log    *slog.Logger
}

func NewUserClient(addr string, log *slog.Logger) (*UserClient, error) {
    conn, err := grpc.NewClient(addr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        return nil, err
    }
    return &UserClient{client: pb.NewUserServiceClient(conn), log: log}, nil
}

func (c *UserClient) GetUser(ctx context.Context, id string) (*pb.GetUserResponse, error) {
    return c.client.GetUser(ctx, &pb.GetUserRequest{Id: id})
}
```

Notes:
- `insecure.NewCredentials()` dev only — use TLS in production
- Use `grpc.NewClient` (not deprecated `grpc.Dial`)
- Create client once, inject via DI
- `context.Context` propagates deadlines/cancellations natively into gRPC
