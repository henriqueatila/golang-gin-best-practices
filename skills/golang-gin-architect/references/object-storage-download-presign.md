# Object Storage — Download and Presigned URLs

Streaming downloads and presigned URL patterns. Companion to `object-storage-multipart-minio.md` (multipart upload, MinIO Docker Compose).

---

## Download and Streaming

Stream S3 objects directly to the Gin response writer — avoid loading into memory.

```go
// GetObject streams an S3 object to the provided writer.
func (c *S3Client) GetObject(ctx context.Context, key string, w io.Writer) (string, error) {
    result, err := c.client.GetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(c.bucket),
        Key:    aws.String(key),
    })
    if err != nil {
        return "", fmt.Errorf("s3 get object %q: %w", key, err)
    }
    defer result.Body.Close()

    contentType := aws.ToString(result.ContentType)
    if _, err := io.Copy(w, result.Body); err != nil {
        return "", fmt.Errorf("s3 stream object %q: %w", key, err)
    }
    return contentType, nil
}
```

```go
func (h *FileHandler) Download(c *gin.Context) {
    key := c.Param("key")

    contentType, err := h.svc.Download(c.Request.Context(), key, c.Writer)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "object not found"})
        return
    }

    c.Header("Content-Type", contentType)
    c.Header("Content-Disposition", "attachment; filename="+filepath.Base(key))
}
```

---

## Presigned URLs

Presigned URLs delegate upload/download directly between client and S3 — server issues a time-limited signed URL and never proxies the bytes.

```go
// PresignPutURL generates a signed URL for direct client upload.
// expiry: typically 15 minutes for uploads, 1 hour for downloads.
func (c *S3Client) PresignPutURL(ctx context.Context, key, contentType string, expiry time.Duration) (string, error) {
    presigner := s3.NewPresignClient(c.client)
    req, err := presigner.PresignPutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(c.bucket),
        Key:         aws.String(key),
        ContentType: aws.String(contentType),
    }, s3.WithPresignExpires(expiry))
    if err != nil {
        return "", fmt.Errorf("presign put %q: %w", key, err)
    }
    return req.URL, nil
}

// PresignGetURL generates a signed URL for direct client download.
func (c *S3Client) PresignGetURL(ctx context.Context, key string, expiry time.Duration) (string, error) {
    presigner := s3.NewPresignClient(c.client)
    req, err := presigner.PresignGetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(c.bucket),
        Key:    aws.String(key),
    }, s3.WithPresignExpires(expiry))
    if err != nil {
        return "", fmt.Errorf("presign get %q: %w", key, err)
    }
    return req.URL, nil
}
```

Handler — request presigned upload URL:

```go
func (h *FileHandler) PresignUpload(c *gin.Context) {
    var req struct {
        Filename    string `json:"filename" binding:"required"`
        ContentType string `json:"content_type" binding:"required"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    key := fmt.Sprintf("uploads/%s%s", uuid.New().String(), filepath.Ext(req.Filename))
    url, err := h.store.PresignPutURL(c.Request.Context(), key, req.ContentType, 15*time.Minute)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "could not generate URL"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"upload_url": url, "key": key, "expires_in": 900})
}
```

Client then `PUT`s directly to `upload_url` with file bytes and matching `Content-Type` header.
