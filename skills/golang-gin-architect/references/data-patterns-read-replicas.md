# Data Patterns — Read Replicas

Go implementation for PostgreSQL read replica pool with lag handling.

Companion to `data-patterns-cqrs.md` (CQRS pattern, materialized views).

**Default stance:** Add indexes and run EXPLAIN ANALYZE before splitting to replicas.

---

## Gate — all must be true

- [ ] Read/write ratio exceeds 5:1 AND you have measured that the primary is the bottleneck
- [ ] Replication lag is acceptable for your read use cases (typically <100ms for sync replica)
- [ ] You have a plan for handling replication lag in code (stale reads, retry on not-found)

**Simpler alternative:** Add indexes. Run `EXPLAIN ANALYZE` on your slowest queries. PostgreSQL with correct indexes routinely handles 10K+ reads/sec on modest hardware. Measure before splitting.

---

## Go Implementation

```go
// internal/repository/db.go
type DBPool struct {
    Primary *sqlx.DB // reads + writes
    Replica *sqlx.DB // reads only (can be nil — falls back to Primary)
}

func (p *DBPool) ReadDB() *sqlx.DB {
    if p.Replica != nil {
        return p.Replica
    }
    return p.Primary
}

func NewDBPool(primaryDSN, replicaDSN string) (*DBPool, error) {
    primary, err := sqlx.Connect("postgres", primaryDSN)
    if err != nil {
        return nil, fmt.Errorf("connect primary: %w", err)
    }
    pool := &DBPool{Primary: primary}
    if replicaDSN != "" {
        replica, err := sqlx.Connect("postgres", replicaDSN)
        if err != nil {
            slog.Warn("replica unavailable, using primary for reads", "err", err)
        } else {
            pool.Replica = replica
        }
    }
    return pool, nil
}
```

---

## Usage in Repository

```go
func (r *UserRepository) FindByID(ctx context.Context, id string) (*User, error) {
    var u User
    err := r.pool.ReadDB().GetContext(ctx, &u, `SELECT * FROM users WHERE id = $1`, id)
    if err != nil {
        return nil, fmt.Errorf("find user: %w", err)
    }
    return &u, nil
}

func (r *UserRepository) Create(ctx context.Context, u *User) error {
    _, err := r.pool.Primary.NamedExecContext(ctx,
        `INSERT INTO users (id, email, name) VALUES (:id, :email, :name)`, u)
    return err
}
```

---

## Handling Replication Lag (Read-Your-Writes)

```go
type contextKey string
const ReadYourWritesKey contextKey = "read_your_writes"

func (r *UserRepository) readDB(ctx context.Context) *sqlx.DB {
    if ryw, _ := ctx.Value(ReadYourWritesKey).(bool); ryw {
        return r.pool.Primary
    }
    return r.pool.ReadDB()
}
```

Set `ctx = context.WithValue(ctx, ReadYourWritesKey, true)` immediately after a write when the caller needs to read back their own write within the same request.
