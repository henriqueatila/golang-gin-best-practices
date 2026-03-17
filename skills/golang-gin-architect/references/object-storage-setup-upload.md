# Object Storage — S3 Client Setup and Upload Patterns

Practical guide to integrating S3-compatible object storage in Go Gin APIs.

---

## When to Use Object Storage

```
START: Are you storing files > 1MB or binary blobs?
  ├── No  → PostgreSQL BYTEA or local filesystem. Done.
  └── Yes → Do you need CDN delivery?
      ├── Yes → S3 + CloudFront / CDN. Done.
      └── No  → Is it user-uploaded content?
          ├── Yes → S3 with presigned PUT URLs (client uploads directly)
          └── No  → S3 with server-side upload (internal pipelines)
```

**Cost gate:** S3 charges per request + per GB + egress. For internal files rarely downloaded, PostgreSQL BYTEA (< 5 MB) or a mounted volume is simpler. Choose object storage when you expect > 100 MB total data or need CDN/direct-client access.

---

## S3 Client Setup

Dependencies:

```
go get github.com/aws/aws-sdk-go-v2
go get github.com/aws/aws-sdk-go-v2/config
go get github.com/aws/aws-sdk-go-v2/credentials
go get github.com/aws/aws-sdk-go-v2/service/s3
```

```go
// pkg/storage/s3.go
type S3Client struct {
    client *s3.Client
    bucket string
    logger *slog.Logger
}

// Compatible with AWS S3, MinIO, and Cloudflare R2.
// Pass empty endpoint for real AWS.
func NewS3Client(endpoint, region, accessKey, secretKey, bucket string) (*S3Client, error) {
    optFns := []func(*config.LoadOptions) error{
        config.WithRegion(region),
        config.WithCredentialsProvider(
            credentials.NewStaticCredentialsProvider(accessKey, secretKey, ""),
        ),
    }

    cfg, err := config.LoadDefaultConfig(context.Background(), optFns...)
    if err != nil {
        return nil, fmt.Errorf("storage: load config: %w", err)
    }

    clientOpts := []func(*s3.Options){}
    if endpoint != "" {
        clientOpts = append(clientOpts, func(o *s3.Options) {
            o.BaseEndpoint = aws.String(endpoint)
            o.UsePathStyle = true // required for MinIO
        })
    }

    return &S3Client{
        client: s3.NewFromConfig(cfg, clientOpts...),
        bucket: bucket,
        logger: slog.Default().With("component", "s3"),
    }, nil
}
```

Environment variables:

```
S3_ENDPOINT=http://localhost:9000   # empty for real AWS
S3_REGION=us-east-1
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=myapp
```

---

## Upload: Thin Handler

```go
const maxUploadSize = 100 << 20 // 100 MB

func (h *FileHandler) Upload(c *gin.Context) {
    c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxUploadSize)

    file, header, err := c.Request.FormFile("file")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid file: " + err.Error()})
        return
    }
    defer file.Close()

    contentType := header.Header.Get("Content-Type")
    if contentType == "" {
        contentType = "application/octet-stream"
    }

    url, err := h.svc.Upload(c.Request.Context(), file, header.Filename, contentType)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "upload failed"})
        return
    }
    c.JSON(http.StatusCreated, gin.H{"url": url})
}
```

## Upload: Service and Storage Method

```go
func (s *fileService) Upload(ctx context.Context, r io.Reader, filename, contentType string) (string, error) {
    key := fmt.Sprintf("uploads/%s%s", uuid.New().String(), filepath.Ext(filename))
    url, err := s.store.PutObject(ctx, key, contentType, r)
    if err != nil {
        return "", fmt.Errorf("file service upload: %w", err)
    }
    return url, nil
}

func (c *S3Client) PutObject(ctx context.Context, key, contentType string, body io.Reader) (string, error) {
    _, err := c.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(c.bucket),
        Key:         aws.String(key),
        Body:        body,
        ContentType: aws.String(contentType),
    })
    if err != nil {
        return "", fmt.Errorf("s3 put object %q: %w", key, err)
    }
    c.logger.InfoContext(ctx, "object uploaded", "key", key)
    return fmt.Sprintf("s3://%s/%s", c.bucket, key), nil
}
```

---

## See Also

- `object-storage-download-presign.md` — download/streaming, presigned URLs, multipart upload, MinIO Docker Compose
