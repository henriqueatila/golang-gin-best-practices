# Swagger Annotations — CRUD Handlers and Auth Endpoints

Complete annotations for CRUD operations (Update, Patch, Delete) and auth endpoints (Register, Login, Refresh).

## CRUD Handler Annotations

### Update (PUT)

```go
// Update godoc
//
// @Summary      Update user
// @Description  Replace all user fields
// @Tags         users
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Param        id     path  string                   true  "User ID" format(uuid)
// @Param        input  body  domain.UpdateUserRequest  true  "Fields to update"
// @Success      200    {object}  domain.UserResponse
// @Failure      400    {object}  domain.ErrorResponse
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      404    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users/{id} [put]
func (h *UserHandler) Update(c *gin.Context) {}
```
### Patch (PATCH)

```go
// Patch godoc
//
// @Summary      Partially update user
// @Description  Update only the provided fields; omitted fields are unchanged
// @Tags         users
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Param        id     path  string                  true  "User ID" format(uuid)
// @Param        input  body  domain.PatchUserRequest  true  "Fields to patch"
// @Success      200    {object}  domain.UserResponse
// @Failure      400    {object}  domain.ErrorResponse
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      404    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users/{id} [patch]
func (h *UserHandler) Patch(c *gin.Context) {}
```
### Delete

```go
// Delete godoc
//
// @Summary      Delete user
// @Description  Permanently remove a user account
// @Tags         users
// @Security     BearerAuth
// @Param        id  path  string  true  "User ID" format(uuid)
// @Success      204
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      403    {object}  domain.ErrorResponse
// @Failure      404    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users/{id} [delete]
func (h *UserHandler) Delete(c *gin.Context) {}
```

## Auth Endpoint Annotations

These do not require `@Security` — they produce tokens, not consume them.

### Register

```go
// Register godoc
//
// @Summary      Register a new account
// @Description  Create a user account and return access + refresh tokens
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      domain.RegisterRequest  true  "Registration payload"
// @Success      201      {object}  domain.TokenResponse
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      409      {object}  domain.ErrorResponse   "Email already registered"
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /auth/register [post]
func (h *AuthHandler) Register(c *gin.Context) {}
```

### Login

```go
// Login godoc
//
// @Summary      Log in
// @Description  Authenticate with email and password; returns access + refresh tokens
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      domain.LoginRequest  true  "Login credentials"
// @Success      200      {object}  domain.TokenResponse
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      401      {object}  domain.ErrorResponse  "Invalid credentials"
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /auth/login [post]
func (h *AuthHandler) Login(c *gin.Context) {}
```

### Refresh Token

```go
// RefreshToken godoc
//
// @Summary      Refresh access token
// @Description  Exchange a valid refresh token for a new access token
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      domain.RefreshRequest  true  "Refresh token payload"
// @Success      200      {object}  domain.TokenResponse
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      401      {object}  domain.ErrorResponse  "Refresh token expired or invalid"
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /auth/refresh [post]
func (h *AuthHandler) RefreshToken(c *gin.Context) {}
```

### Supporting Auth DTOs

```go
// domain/auth.go
type RegisterRequest struct {
    Name     string `json:"name"     example:"Jane Doe"        binding:"required,min=2,max=100"`
    Email    string `json:"email"    example:"jane@example.com" binding:"required,email"`
    Password string `json:"password" example:"s3cur3P@ss!"      binding:"required,min=8"`
}
type LoginRequest struct {
    Email    string `json:"email"    example:"jane@example.com" binding:"required,email"`
    Password string `json:"password" example:"s3cur3P@ss!"      binding:"required"`
}
type RefreshRequest struct {
    RefreshToken string `json:"refresh_token" example:"eyJhbGci..." binding:"required"`
}
type TokenResponse struct {
    AccessToken  string `json:"access_token"  example:"eyJhbGci..."`
    RefreshToken string `json:"refresh_token" example:"eyJhbGci..."`
    ExpiresIn    int    `json:"expires_in"    example:"3600"`
}
```
