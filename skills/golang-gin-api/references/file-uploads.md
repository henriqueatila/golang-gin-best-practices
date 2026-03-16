# File Uploads

File upload patterns for Go Gin APIs — single file, multiple files, struct binding, S3/cloud storage, MIME validation, and security.

## Router Configuration

Always set `MaxMultipartMemory` before registering upload routes:

```go
// cmd/api/main.go
r := gin.New()
r.MaxMultipartMemory = 8 << 20 // 8 MiB in memory; remainder spills to temp files
```

## Single File Upload

```go
// internal/handler/upload_handler.go
func (h *UploadHandler) UploadFile(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "file is required"})
        return
    }

    if err := validateFile(file); err != nil {
        c.JSON(http.StatusUnprocessableEntity, gin.H{"error": err.Error()})
        return
    }

    safeName := uuid.NewString() + "_" + filepath.Base(file.Filename)
    dst := filepath.Join(h.uploadDir, safeName)

    if err := c.SaveUploadedFile(file, dst); err != nil {
        h.logger.Error("save upload failed", "err", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "upload failed"})
        return
    }

    c.JSON(http.StatusCreated, gin.H{"filename": safeName})
}

const maxFileSize = 10 << 20 // 10 MiB

var allowedMIME = map[string]bool{
    "image/jpeg": true,
    "image/png":  true,
    "application/pdf": true,
}

func validateFile(file *multipart.FileHeader) error {
    if file.Size > maxFileSize {
        return fmt.Errorf("file exceeds 10 MiB limit")
    }

    src, err := file.Open()
    if err != nil {
        return fmt.Errorf("cannot open file")
    }
    defer src.Close()

    // Read first 512 bytes for MIME sniffing — do NOT trust file.Header["Content-Type"]
    buf := make([]byte, 512)
    if _, err := src.Read(buf); err != nil {
        return fmt.Errorf("cannot read file")
    }

    mimeType := http.DetectContentType(buf)
    if !allowedMIME[mimeType] {
        return fmt.Errorf("file type %q not allowed", mimeType)
    }

    return nil
}
```

## Multiple File Upload

```go
// internal/handler/upload_handler.go
func (h *UploadHandler) UploadFiles(c *gin.Context) {
    form, err := c.MultipartForm()
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid multipart form"})
        return
    }

    files := form.File["files[]"]
    if len(files) == 0 {
        c.JSON(http.StatusBadRequest, gin.H{"error": "no files provided"})
        return
    }

    var saved []string
    for _, file := range files {
        if err := validateFile(file); err != nil {
            c.JSON(http.StatusUnprocessableEntity, gin.H{"error": err.Error()})
            return
        }

        safeName := uuid.NewString() + "_" + filepath.Base(file.Filename)
        if err := c.SaveUploadedFile(file, filepath.Join(h.uploadDir, safeName)); err != nil {
            h.logger.Error("save upload failed", "file", file.Filename, "err", err)
            c.JSON(http.StatusInternalServerError, gin.H{"error": "upload failed"})
            return
        }
        saved = append(saved, safeName)
    }

    c.JSON(http.StatusCreated, gin.H{"filenames": saved})
}
```

## Struct Binding with File Field

`ShouldBind` resolves `*multipart.FileHeader` from `form` tags:

```go
// internal/handler/upload_handler.go
type UploadRequest struct {
    Name        string                `form:"name"        binding:"required,max=100"`
    Description string                `form:"description" binding:"omitempty,max=500"`
    File        *multipart.FileHeader `form:"file"        binding:"required"`
}

func (h *UploadHandler) UploadWithMeta(c *gin.Context) {
    var req UploadRequest
    if err := c.ShouldBind(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    if err := validateFile(req.File); err != nil {
        c.JSON(http.StatusUnprocessableEntity, gin.H{"error": err.Error()})
        return
    }

    url, err := h.storage.Upload(c.Request.Context(), req.File)
    if err != nil {
        h.logger.Error("storage upload failed", "err", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "upload failed"})
        return
    }

    c.JSON(http.StatusCreated, gin.H{"url": url, "name": req.Name})
}
```

## S3 / Cloud Storage

Define a storage interface so handlers stay decoupled from the provider:

```go
// internal/storage/storage.go
type FileStorage interface {
    Upload(ctx context.Context, file *multipart.FileHeader) (url string, err error)
    PresignedURL(ctx context.Context, key string, ttl time.Duration) (string, error)
}
```

```go
// internal/storage/s3_storage.go
type S3Storage struct {
    client *s3.Client
    bucket string
    logger *slog.Logger
}

func (s *S3Storage) Upload(ctx context.Context, file *multipart.FileHeader) (string, error) {
    src, err := file.Open()
    if err != nil {
        return "", fmt.Errorf("open file: %w", err)
    }
    defer src.Close()

    key := uuid.NewString() + "_" + filepath.Base(file.Filename)

    _, err = s.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket:      aws.String(s.bucket),
        Key:         aws.String(key),
        Body:        src,
        ContentType: aws.String(file.Header.Get("Content-Type")),
    })
    if err != nil {
        return "", fmt.Errorf("s3 put object: %w", err)
    }

    return fmt.Sprintf("https://%s.s3.amazonaws.com/%s", s.bucket, key), nil
}

func (s *S3Storage) PresignedURL(ctx context.Context, key string, ttl time.Duration) (string, error) {
    presignClient := s3.NewPresignClient(s.client)
    req, err := presignClient.PresignGetObject(ctx, &s3.GetObjectInput{
        Bucket: aws.String(s.bucket),
        Key:    aws.String(key),
    }, s3.WithPresignExpires(ttl))
    if err != nil {
        return "", fmt.Errorf("presign: %w", err)
    }
    return req.URL, nil
}
```

Local filesystem fallback for development — implement `FileStorage` writing to disk.

## Security Checklist

| Risk | Mitigation |
|------|-----------|
| Directory traversal | `filepath.Base(file.Filename)` strips all path components |
| MIME spoofing | Detect MIME from first 512 bytes with `http.DetectContentType`; never trust `Content-Type` header |
| Oversized files | `router.MaxMultipartMemory` + explicit `file.Size` check in handler |
| Filename collision | Prefix with `uuid.NewString()` |
| Serving uploaded files | Store outside webroot; serve through signed URLs or a dedicated file handler |
| Malware | Integrate ClamAV or cloud AV scan before persisting in production |
| Path exposure | Never return filesystem paths; return opaque keys or signed URLs |
