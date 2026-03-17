# Extensions Toolkit — pgcrypto and uuid-ossp

## pgcrypto — Encryption and Hashing

### Setup

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

### gen_random_uuid()

Built-in since PostgreSQL 13 (no pgcrypto needed for PG 13+):

```sql
SELECT gen_random_uuid();  -- 'a3f4c7d2-...'
```

### PGP Symmetric Encryption — Column Encryption

```sql
-- Encrypt on insert
INSERT INTO patients (id, name, ssn_encrypted)
VALUES (
    gen_random_uuid(), 'John Doe',
    pgp_sym_encrypt('123-45-6789', current_setting('app.encryption_key'))
);

-- Decrypt on read
SELECT name, pgp_sym_decrypt(ssn_encrypted, current_setting('app.encryption_key')) AS ssn
FROM patients WHERE id = $1;

-- Set per-connection before queries (never hardcode in SQL)
SET app.encryption_key = 'your-secret-key-from-env';
```

### When to Encrypt in Go vs PostgreSQL

| Factor | Encrypt in Go | Encrypt in PostgreSQL |
|---|---|---|
| Key management | Easier — key never enters DB | Key passed via SQL (risk of query log exposure) |
| Performance | App scales horizontally | DB CPU used |
| Use when | PII, secrets, bulk data | Legacy schemas, multi-app access to same DB |

**Recommendation:** Encrypt sensitive fields in Go before storing. Use `pgp_sym_encrypt` only when multiple services share the same database.

### AES-GCM Encryption in Go

```go
// EncryptField encrypts a plaintext string using AES-GCM. key must be 16, 24, or 32 bytes.
func EncryptField(plaintext string, key []byte) (string, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return "", fmt.Errorf("new cipher: %w", err)
    }
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", fmt.Errorf("new gcm: %w", err)
    }
    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return "", fmt.Errorf("generate nonce: %w", err)
    }
    ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

// DecryptField decrypts a base64-encoded AES-GCM ciphertext.
func DecryptField(encoded string, key []byte) (string, error) {
    data, err := base64.StdEncoding.DecodeString(encoded)
    if err != nil {
        return "", fmt.Errorf("decode base64: %w", err)
    }
    block, err := aes.NewCipher(key)
    if err != nil {
        return "", fmt.Errorf("new cipher: %w", err)
    }
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", fmt.Errorf("new gcm: %w", err)
    }
    nonceSize := gcm.NonceSize()
    if len(data) < nonceSize {
        return "", fmt.Errorf("ciphertext too short")
    }
    plaintext, err := gcm.Open(nil, data[:nonceSize], data[nonceSize:], nil)
    if err != nil {
        return "", fmt.Errorf("decrypt: %w", err)
    }
    return string(plaintext), nil
}
```

---

## uuid-ossp — UUID Generation

### Setup

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### uuid_generate_v4() vs gen_random_uuid()

| Function | Source | Available |
|---|---|---|
| `uuid_generate_v4()` | uuid-ossp extension | All PostgreSQL versions |
| `gen_random_uuid()` | Built-in | PostgreSQL 13+ |

**Recommendation:** Use `gen_random_uuid()` for PostgreSQL 13+. No extension required.

### Other uuid-ossp Functions

```sql
SELECT uuid_generate_v1();           -- UUID v1: time-based (includes MAC address)
SELECT uuid_generate_v3(uuid_ns_dns(), 'example.com');  -- v3: MD5 hash (deterministic)
SELECT uuid_generate_v5(uuid_ns_url(), 'https://example.com/users/42');  -- v5: SHA-1 hash
```

UUID v5 is useful for deterministic IDs from known inputs (idempotency keys, external ID mapping).

### UUID Primary Key Performance

| Key Type | Insert Pattern | Index Fragmentation | Notes |
|---|---|---|---|
| `BIGINT IDENTITY` | Sequential | Minimal | Best write performance |
| `UUIDv4` | Random | High | Poor for sequential inserts |
| `UUIDv7` | Time-ordered random | Minimal | Best of both worlds |

**For high-insert tables, prefer `BIGINT GENERATED ALWAYS AS IDENTITY` or UUIDv7:**

```go
// Use github.com/google/uuid v1.6+ for UUIDv7
import "github.com/google/uuid"

id, err := uuid.NewV7()
// time-ordered, globally unique, stores as standard UUID column
```
