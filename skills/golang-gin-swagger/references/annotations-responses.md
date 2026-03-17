# Swagger Annotations — Response Patterns

All response annotation patterns: single object, array, paginated, no-content, primitives, failures, grouped codes, and response headers.

## Syntax

`@Success <code> {<type>} <model> "<description>"`

## Single Object and Array

```go
// @Success  200  {object}  domain.User  "OK"
// @Success  201  {object}  domain.User  "Created"
// @Success  200  {array}   domain.User  "List of users"
```

## Paginated Response

Define a wrapper struct for pagination:

```go
type PaginatedResponse struct {
    Data       []domain.User `json:"data"`
    Page       int           `json:"page"        example:"1"`
    Limit      int           `json:"limit"       example:"20"`
    TotalCount int           `json:"total_count"  example:"150"`
    TotalPages int           `json:"total_pages"  example:"8"`
}

// @Success  200  {object}  PaginatedResponse  "Paginated list"
```

## No Content and Primitives

```go
// @Success  204  "No content"
// @Success  200  {string}   string  "Plain text response"
// @Success  200  {integer}  int     "Count"
// @Success  200  {boolean}  bool    "Status"
```

## Failure Responses

Document all possible error codes:

```go
// @Failure  400  {object}  domain.ErrorResponse  "Bad request"
// @Failure  401  {object}  domain.ErrorResponse  "Unauthorized"
// @Failure  403  {object}  domain.ErrorResponse  "Forbidden"
// @Failure  404  {object}  domain.ErrorResponse  "Not found"
// @Failure  409  {object}  domain.ErrorResponse  "Conflict"
// @Failure  422  {object}  domain.ErrorResponse  "Validation failed"
// @Failure  429  {object}  domain.ErrorResponse  "Rate limit exceeded"
// @Failure  500  {object}  domain.ErrorResponse  "Internal server error"
```

## Grouped Failure Codes

When multiple codes share the same response type:

```go
// @Failure  400,401,404,500  {object}  domain.ErrorResponse
```

## Response Headers

Document headers returned with responses:

```go
// @Success  200  {object}  domain.User
// @Header   200  {string}   X-Request-ID          "Unique request identifier"
// @Header   200  {integer}  X-RateLimit-Remaining  "Remaining requests"
// @Header   200  {string}   X-RateLimit-Reset      "Reset timestamp"
```
