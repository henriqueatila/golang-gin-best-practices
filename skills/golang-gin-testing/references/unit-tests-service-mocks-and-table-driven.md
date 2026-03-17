# Unit Tests — Service Testing with Mocks and Table-Driven Tests

## Service Testing with Mocks

Test business logic (password hashing, conflict detection, error wrapping) independent of HTTP or DB.

```go
// internal/service/user_service_test.go
package service_test

type mockUserRepository struct {
    createFn     func(ctx context.Context, user *domain.User) error
    getByEmailFn func(ctx context.Context, email string) (*domain.User, error)
    getByIDFn    func(ctx context.Context, id string) (*domain.User, error)
    listFn       func(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error)
    updateFn     func(ctx context.Context, user *domain.User) error
    deleteFn     func(ctx context.Context, id string) error
}
func (m *mockUserRepository) Create(ctx context.Context, u *domain.User) error {
    if m.createFn != nil { return m.createFn(ctx, u) }; return nil }
func (m *mockUserRepository) GetByEmail(ctx context.Context, e string) (*domain.User, error) {
    if m.getByEmailFn != nil { return m.getByEmailFn(ctx, e) }; return nil, domain.ErrNotFound }
func (m *mockUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    if m.getByIDFn != nil { return m.getByIDFn(ctx, id) }; return nil, domain.ErrNotFound }
func (m *mockUserRepository) List(ctx context.Context, o domain.ListOptions) ([]domain.User, int64, error) {
    if m.listFn != nil { return m.listFn(ctx, o) }; return nil, 0, nil }
func (m *mockUserRepository) Update(ctx context.Context, u *domain.User) error {
    if m.updateFn != nil { return m.updateFn(ctx, u) }; return nil }
func (m *mockUserRepository) Delete(ctx context.Context, id string) error {
    if m.deleteFn != nil { return m.deleteFn(ctx, id) }; return nil }

func TestUserService_Create_HashesPassword(t *testing.T) {
    var savedUser *domain.User
    repo := &mockUserRepository{
        getByEmailFn: func(_ context.Context, _ string) (*domain.User, error) { return nil, domain.ErrNotFound },
        createFn:     func(_ context.Context, u *domain.User) error { savedUser = u; return nil },
    }
    svc := service.NewUserService(repo, slog.Default())
    if _, err := svc.Create(context.Background(), domain.CreateUserRequest{Name: "Alice", Email: "alice@example.com", Password: "plaintext"}); err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if savedUser.PasswordHash == "plaintext" { t.Error("password must not be stored as plaintext") }
    if savedUser.PasswordHash == "" { t.Error("expected PasswordHash to be set") }
}

func TestUserService_Create_ConflictOnDuplicateEmail(t *testing.T) {
    repo := &mockUserRepository{getByEmailFn: func(_ context.Context, email string) (*domain.User, error) {
        return &domain.User{Email: email}, nil // email already taken
    }}
    _, err := service.NewUserService(repo, slog.Default()).Create(context.Background(),
        domain.CreateUserRequest{Name: "Alice", Email: "taken@example.com", Password: "secret123"})
    var appErr *domain.AppError
    if !errors.As(err, &appErr) || appErr.Code != 409 { t.Errorf("expected ErrConflict (409), got %v", err) }
}
```

---

## Mock Generation Patterns

**Manual mocks** (recommended for small interfaces) — struct with function fields (shown above). Zero deps, easy to debug.

**gomock** (recommended for large interfaces) — catches unexpected calls automatically:

```bash
go install go.uber.org/mock/mockgen@latest
mockgen -source=internal/domain/user.go -destination=internal/testutil/mocks/mock_user_repository.go -package=mocks
```

```go
func TestWithGoMock(t *testing.T) {
    ctrl := gomock.NewController(t)
    repo := mocks.NewMockUserRepository(ctrl)
    repo.EXPECT().GetByEmail(gomock.Any(), "alice@example.com").Return(nil, domain.ErrNotFound)
    repo.EXPECT().Create(gomock.Any(), gomock.Any()).Return(nil)
    _, err := service.NewUserService(repo, slog.Default()).Create(context.Background(),
        domain.CreateUserRequest{Name: "Alice", Email: "alice@example.com", Password: "secret123"})
    if err != nil { t.Fatalf("unexpected error: %v", err) }
}
```

---

## Table-Driven Tests with Subtests

```go
func TestUserService_GetByID(t *testing.T) {
    existing := &domain.User{ID: "u1", Name: "Alice", Email: "alice@example.com", Role: "user"}
    tests := []struct{ name string; id string; repoUser *domain.User; repoErr, wantErr error }{
        {"found",            "u1",      existing, nil,                  nil},
        {"not found",        "missing", nil,      domain.ErrNotFound,   domain.ErrNotFound},
        {"repository error", "any",     nil,      errors.New("db lost"), domain.ErrInternal},
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            repo := &mockUserRepository{getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
                return tc.repoUser, tc.repoErr
            }}
            got, err := service.NewUserService(repo, slog.Default()).GetByID(context.Background(), tc.id)
            if tc.wantErr != nil {
                var appErr *domain.AppError
                if !errors.As(err, &appErr) { t.Errorf("want AppError wrapping %v, got %T: %v", tc.wantErr, err, err) }
                return
            }
            if err != nil { t.Fatalf("unexpected error: %v", err) }
            if got.ID != tc.repoUser.ID { t.Errorf("want ID %q, got %q", tc.repoUser.ID, got.ID) }
        })
    }
}
```

> For handler testing: see [unit-tests-handler-and-json.md](unit-tests-handler-and-json.md).
> For fixtures, t.Helper/Cleanup/Parallel, and middleware testing: see [unit-tests-fixtures-helpers-and-middleware.md](unit-tests-fixtures-helpers-and-middleware.md).
