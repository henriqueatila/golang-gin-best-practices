# Swagger Annotations — File Uploads, Model Renaming, Deprecation, and Extensions

File upload annotations, model renaming, deprecating endpoints, tag metadata, and custom extensions.

## File Uploads

### Single File

```go
// UploadAvatar godoc
//
// @Summary      Upload user avatar
// @Tags         users
// @Accept       multipart/form-data
// @Produce      json
// @Security     BearerAuth
// @Param        id    path      string  true  "User ID"
// @Param        file  formData  file    true  "Avatar image"
// @Success      200   {object}  domain.UploadResponse
// @Failure      400   {object}  domain.ErrorResponse
// @Router       /users/{id}/avatar [post]
func (h *UserHandler) UploadAvatar(c *gin.Context) { ... }
```

### Multiple Files and Mixed Form

```go
// Multiple files
// @Param  files  formData  file  true  "Upload files"  collection(multi)

// File + form fields
// @Accept  multipart/form-data
// @Param   name   formData  string  true  "Document name"
// @Param   file   formData  file    true  "Document file"
```

## Model Renaming

Override the model name in Swagger output (useful for disambiguating types with the same name across packages):

```go
// UserResponse is the public representation of a user.
//
// @name UserResponse
type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}
```

In generated docs, this appears as `UserResponse` instead of `domain.User`.

## Deprecating Endpoints

```go
// GetUserV1 godoc
//
// @Summary      Get user (deprecated)
// @Description  Use GET /api/v2/users/{id} instead
// @Deprecated
// @Tags         users
// @Router       /users/{id} [get]
func (h *UserHandler) GetUserV1(c *gin.Context) { ... }
```

The endpoint appears with a strikethrough in Swagger UI.

## Tag Metadata

Add descriptions to tags in the Swagger UI sidebar (place in the general API info block):

```go
// @tag.name         users
// @tag.description  Operations on user accounts

// @tag.name         auth
// @tag.description  Authentication and token management

// @tag.name         admin
// @tag.description  Administrative operations (requires admin role)
```

## Custom Extensions

Use `@x-` prefix for vendor-specific or custom metadata. Values are JSON:

```go
// General API extensions
// @x-logo  {"url": "https://example.com/logo.png", "altText": "API Logo"}

// Operation-level extensions
// ListUsers godoc
//
// @Summary        List users
// @x-codeSamples  [{"lang": "curl", "source": "curl -H 'Authorization: Bearer {token}' https://api.example.com/users"}]
// @Router         /users [get]

// Field-level extensions
type User struct {
    Role string `json:"role" x-order:"1"`
}
```

Custom extensions appear in the raw `swagger.json` output and are consumed by tools that understand them (e.g., Redoc uses `x-logo`, `x-codeSamples`).
