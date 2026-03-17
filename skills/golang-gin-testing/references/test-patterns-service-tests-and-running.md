# Test Patterns — Service Tests with Mocked Repository and Running Tests

## Service Tests with Mocked Repository

Test business logic without touching the database. The mock implements `domain.UserRepository`.

```go
// internal/service/user_service_test.go
package service_test

// mockUserRepository implements domain.UserRepository for service tests.
type mockUserRepository struct {
    createFn     func(ctx context.Context, user *domain.User) error
    getByEmailFn func(ctx context.Context, email string) (*domain.User, error)
    getByIDFn    func(ctx context.Context, id string) (*domain.User, error)
}

func (m *mockUserRepository) Create(ctx context.Context, user *domain.User) error {
    return m.createFn(ctx, user)
}
func (m *mockUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    return m.getByEmailFn(ctx, email)
}
func (m *mockUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    if m.getByIDFn != nil { return m.getByIDFn(ctx, id) }
    return nil, domain.ErrNotFound
}
func (m *mockUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) { return nil, 0, nil }
func (m *mockUserRepository) Update(ctx context.Context, user *domain.User) error { return nil }
func (m *mockUserRepository) Delete(ctx context.Context, id string) error         { return nil }

func TestUserService_Create_DuplicateEmail(t *testing.T) {
    repo := &mockUserRepository{
        getByEmailFn: func(_ context.Context, email string) (*domain.User, error) {
            return &domain.User{Email: email}, nil // email already taken
        },
        createFn: func(_ context.Context, user *domain.User) error { return nil },
    }

    svc := service.NewUserService(repo, slog.Default())
    _, err := svc.Create(context.Background(), domain.CreateUserRequest{
        Name: "Alice", Email: "alice@example.com", Password: "secret123",
    })

    // Use errors.As to unwrap *AppError and inspect the HTTP status code.
    // errors.As is always safe; errors.Is only works if AppError implements Is().
    var appErr *domain.AppError
    if !errors.As(err, &appErr) || appErr.Code != 409 {
        t.Errorf("expected ErrConflict (409 AppError), got %v", err)
    }
}
```

---

## Running Tests

```bash
# All tests with race detector and coverage
go test -v -race -cover ./...

# Specific package
go test -v -race ./internal/handler/...

# Unit tests only (exclude integration build tag)
go test -v -race -cover -tags='!integration' ./...

# Integration tests only (requires Docker or live DB)
go test -v -race -tags=integration ./internal/repository/...

# Coverage report
go test -race -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

> For test helpers and handler tests: see [test-patterns-helpers-and-handler-tests.md](test-patterns-helpers-and-handler-tests.md).
