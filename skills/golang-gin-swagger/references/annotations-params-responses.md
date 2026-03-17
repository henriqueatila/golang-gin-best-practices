# Swagger Annotations — Params and Responses

Annotation order, general API info, all @Param types, and response patterns.

## Annotation Order

Conventional order (enforced by `swag fmt`):

```go
// FunctionName godoc        ← standard Go doc comment (REQUIRED)
//
// @Summary      Short title
// @Description  Longer description
// @ID           unique-operation-id
// @Tags         tag1,tag2
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Param        ...
// @Success      ...
// @Failure      ...
// @Header       ...
// @Router       /path [method]
```

Always start with a Go doc comment (`// FunctionName godoc`) followed by an empty `//` line before `@Summary`. Without the doc comment, `swag fmt` rejects the annotations.

## General API Info

Place once in `cmd/api/main.go`, directly before `main()`:

```go
// @title           My API
// @version         1.0
// @description     Production REST API

// @contact.name    API Support
// @contact.email   support@example.com
// @contact.url     https://example.com/support

// @license.name    MIT
// @license.url     https://opensource.org/licenses/MIT

// @host            localhost:8080
// @BasePath        /api/v1
// @schemes         http https

// @externalDocs.description  Full API documentation
// @externalDocs.url          https://docs.example.com

// @tag.name         users
// @tag.description  Operations on user accounts

// @x-custom-key     {"env": "production"}

// @securityDefinitions.apikey  BearerAuth
// @in                          header
// @name                        Authorization
// @description                 Enter: Bearer {token}
func main() { ... }
```

Only the first annotation block is used. Do not scatter general annotations across files.

## Param Patterns

Syntax: `@Param <name> <in> <type> <required> "<description>" [attributes]`

### Path Parameters

```go
// @Param  id    path  string  true  "User ID (UUID)"
// @Param  slug  path  string  true  "URL-friendly slug"
```

**Critical:** Use `{id}` in `@Router` (OpenAPI style), not `:id` (Gin style).

### Query Parameters

```go
// @Param  page    query  int     false  "Page number"     default(1) minimum(1)
// @Param  limit   query  int     false  "Items per page"  default(20) minimum(1) maximum(100)
// @Param  role    query  string  false  "Filter by role"  Enums(admin, user, guest)
// @Param  q       query  string  false  "Search query"    minLength(1) maxLength(200)
// @Param  sort    query  string  false  "Sort field"      default(created_at)
// @Param  order   query  string  false  "Sort order"      Enums(asc, desc) default(desc)
// @Param  active  query  bool    false  "Active only"     default(true)
```

### Header Parameters

```go
// @Param  X-Request-ID     header  string  false  "Request tracing ID"
// @Param  Accept-Language  header  string  false  "Preferred language"  default(en)
```

### Body Parameters

```go
// JSON body — reference a named struct
// @Param  request  body  domain.CreateUserRequest  true  "Request body"
// @Param  data     body  string                    true  "Raw text payload"
```

Only one `body` param per endpoint. Use `--parseInternal` for `internal/` packages.

### FormData Parameters

```go
// @Param  name   formData  string  true   "User name"
// @Param  email  formData  string  true   "Email address"
// @Param  age    formData  int     false  "Age"
```

For response patterns (object, array, paginated, primitives, failures, headers): see [annotations-responses.md](annotations-responses.md).
