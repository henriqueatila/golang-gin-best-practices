# Object Storage — Multipart Upload and MinIO Docker Compose

Large file uploads and local dev setup. Companion to `object-storage-download-presign.md`.

---

## Multipart Upload for Large Files

For files > 100 MB, use S3 multipart upload to avoid timeouts and enable resumable transfers.

```go
const partSize = 10 << 20 // 10 MB per part (minimum 5 MB required by S3)

// PutLargeObject uses the SDK upload manager (handles multipart automatically).
func (c *S3Client) PutLargeObject(ctx context.Context, key, contentType string, body io.Reader) error {
    uploader := manager.NewUploader(c.client, func(u *manager.Uploader) {
        u.PartSize = partSize
        u.Concurrency = 3 // parallel part uploads
    })

    _, err := uploader.Upload(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(c.bucket),
        Key:         aws.String(key),
        Body:        body,
        ContentType: aws.String(contentType),
    })
    if err != nil {
        return fmt.Errorf("s3 multipart upload %q: %w", key, err)
    }
    c.logger.InfoContext(ctx, "large object uploaded", "key", key)
    return nil
}
```

`manager.NewUploader` transparently falls back to single-part upload when body < 5 MB — safe to use for all sizes.

---

## MinIO Docker Compose (Dev)

```yaml
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"   # S3 API — point S3_ENDPOINT=http://localhost:9000
      - "9001:9001"   # Web console
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 5s
      timeout: 5s
      retries: 5

  createbuckets:
    image: minio/mc:latest
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
        mc alias set local http://minio:9000 minioadmin minioadmin;
        mc mb --ignore-existing local/myapp;
        mc anonymous set download local/myapp/public;
        exit 0;
      "

volumes:
  minio_data:
```

MinIO is wire-compatible with AWS S3 SDK v2. Switch to real AWS by removing `S3_ENDPOINT` and setting real credentials — no code changes required.
