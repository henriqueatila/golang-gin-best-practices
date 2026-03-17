# Swagger Annotations — Advanced Features

Security definitions, model tags, enums from constants, file uploads, model renaming, deprecation, tag metadata, and custom extensions.

## Security Definitions

### API Key (Bearer JWT)

```go
// In main.go:
// @securityDefinitions.apikey  BearerAuth
// @in                          header
// @name                        Authorization
// @description                 Enter: Bearer {token}

// On each protected handler:
// @Security  BearerAuth
```

### API Key (Custom Header)

```go
// @securityDefinitions.apikey  ApiKeyAuth
// @in                          header
// @name                        X-API-Key
// @description                 API key for service access
```

### Basic Auth and OAuth2

```go
// @securityDefinitions.basic  BasicAuth

// @securityDefinitions.oauth2.implicit       OAuth2Implicit
// @authorizationUrl                          https://auth.example.com/authorize
// @scope.read                                Read access
// @scope.write                               Write access

// @securityDefinitions.oauth2.accessCode     OAuth2AccessCode
// @tokenUrl                                  https://auth.example.com/token
// @authorizationUrl                          https://auth.example.com/authorize
```

### Multiple Schemes on One Endpoint

```go
// Both required (AND logic):
// @Security  BearerAuth
// @Security  ApiKeyAuth

// Either accepted (OR logic):
// @Security  BearerAuth || ApiKeyAuth
```

## Model Tags

Complete list of struct tags recognized by swag:

```go
type Example struct {
    // Value constraints
    Name   string  `json:"name"   example:"Jane"  minLength:"2" maxLength:"100"`
    Age    int     `json:"age"    example:"30"     minimum:"0"   maximum:"150"`
    Score  float64 `json:"score"  example:"9.5"    minimum:"0"   maximum:"10"`

    // Format hints
    ID        string    `json:"id"         example:"550e8400-..."  format:"uuid"`
    Email     string    `json:"email"      example:"j@example.com" format:"email"`
    Website   string    `json:"website"    example:"https://..."   format:"uri"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T10:30:00Z" format:"date-time"`
    Birthday  string    `json:"birthday"   example:"1990-05-15"    format:"date"`

    // Enums
    Role   string `json:"role"   example:"admin"  enums:"admin,user,guest"`
    Status string `json:"status" example:"active" enums:"active,inactive,banned"`

    // Defaults
    Page  int `json:"page"  default:"1"`
    Limit int `json:"limit" default:"20"`

    // Type overrides (for types swag can't infer)
    UpdatedAt  CustomTime    `json:"updated_at"  swaggertype:"string"  format:"date-time"`
    ExternalID sql.NullInt64 `json:"external_id" swaggertype:"integer"`
    Tags       []big.Float   `json:"tags"        swaggertype:"array,number"`
    Metadata   interface{}   `json:"metadata"    swaggertype:"object"`

    // Exclusions
    PasswordHash string `json:"-"             swaggerignore:"true"`
    InternalFlag bool   `json:"internal_flag" swaggerignore:"true"`

    // Read-only / Write-only
    ID2 string `json:"id2" readOnly:"true"`   // shown in response only
    Pwd string `json:"pwd" writeOnly:"true"`  // shown in request only

    // Required (swag infers from binding tag)
    Required string `json:"required" binding:"required"`
}
```

## Enums from Constants

Swag auto-detects Go `const` blocks and generates enum values:

```go
type OrderStatus string

const (
    OrderPending    OrderStatus = "pending"
    OrderProcessing OrderStatus = "processing"
    OrderShipped    OrderStatus = "shipped"
    OrderDelivered  OrderStatus = "delivered"
    OrderCancelled  OrderStatus = "cancelled"
)

type Order struct {
    ID     string      `json:"id"`
    Status OrderStatus `json:"status"` // swagger generates enum automatically
}
```

Works when `OrderStatus` is a named type with `const` values of that type in the same package.

For file uploads, model renaming, deprecation, tag metadata, and custom extensions: see [annotations-extras.md](annotations-extras.md).
