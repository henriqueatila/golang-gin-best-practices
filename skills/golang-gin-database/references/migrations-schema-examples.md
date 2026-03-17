# Migrations — Schema Examples: Roles, Junction Table, and Trigger

Example migrations 000002–000004: roles table, user-roles junction, and updated_at trigger.

> **Architectural recommendation:** golang-migrate is not part of the Gin framework. These are mainstream Go community patterns.

## 000002: Create Roles Table

```sql
-- db/migrations/000002_create_roles_table.up.sql
CREATE TABLE roles (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(50) NOT NULL UNIQUE,
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO roles (name, description) VALUES
    ('admin',  'Full system access'),
    ('user',   'Standard user access'),
    ('viewer', 'Read-only access');
```

```sql
-- db/migrations/000002_create_roles_table.down.sql
DROP TABLE IF EXISTS roles;
```

## 000003: User-Roles Junction Table

```sql
-- db/migrations/000003_add_user_roles_junction.up.sql
CREATE TABLE user_roles (
    user_id    UUID        NOT NULL REFERENCES users(id)  ON DELETE CASCADE,
    role_id    UUID        NOT NULL REFERENCES roles(id)  ON DELETE CASCADE,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by UUID        REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_user_roles_role_id ON user_roles(role_id);
```

```sql
-- db/migrations/000003_add_user_roles_junction.down.sql
DROP TABLE IF EXISTS user_roles;
```

## 000004: Add updated_at Trigger

Automatically keeps `updated_at` in sync without requiring application-level code.

```sql
-- db/migrations/000004_add_updated_at_trigger.up.sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

```sql
-- db/migrations/000004_add_updated_at_trigger.down.sql
DROP TRIGGER IF EXISTS users_set_updated_at ON users;
DROP FUNCTION IF EXISTS set_updated_at();
```
