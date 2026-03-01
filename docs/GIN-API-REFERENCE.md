# GIN-API-REFERENCE.md

> **Verified Gin API surface** — source of truth for all skills in this repository.
>
> Last verified: Gin v1.10+ from github.com/gin-gonic/gin

## Table of Contents

1. [Engine (gin.Engine)](#engine-ginengine)
2. [RouterGroup (gin.RouterGroup)](#routergroup-ginroutergroup)
3. [Context (gin.Context)](#context-gincontext)
4. [Request Binding](#request-binding)
5. [Request Info Accessors](#request-info-accessors)
6. [Response Rendering](#response-rendering)
7. [Flow Control](#flow-control)
8. [Metadata Storage](#metadata-storage)
9. [Binding Tags & Validators](#binding-tags--validators)
10. [Middleware](#middleware)
11. [gin.H](#ginh)
12. [Common Patterns](#common-patterns)

---

## Engine (gin.Engine)

The Engine is Gin's main router and HTTP server handler.

### Constructor
- `gin.New() *Engine` — Creates an empty engine with no middleware
- `gin.Default() *Engine` — Creates engine with Logger + Recovery middleware (recommended for most use cases)

### Running the Server
- `engine.Run(addr ...string) error` — Starts HTTP server on addr (default: "0.0.0.0:8080")
- `engine.RunTLS(addr, certFile, keyFile string) error` — Starts HTTPS server
- `engine.RunUnix(file string) error` — Starts Unix socket server
- `engine.RunFd(fd int) error` — Starts server on file descriptor
- `engine.RunQUIC(addr, certFile, keyFile string) error` — Starts QUIC server
- `engine.RunListener(listener net.Listener) error` — Starts server with custom listener

### Routing
- `engine.Use(middleware ...HandlerFunc) IRoutes` — Adds global middleware
- `engine.Handle(httpMethod, relativePath string, handlers ...HandlerFunc) IRoutes` — Registers handler for HTTP method + path
- `engine.GET(relativePath string, handlers ...HandlerFunc) IRoutes` — Shortcut for GET
- `engine.POST(relativePath string, handlers ...HandlerFunc) IRoutes` — Shortcut for POST
- `engine.PUT(relativePath string, handlers ...HandlerFunc) IRoutes` — Shortcut for PUT
- `engine.DELETE(relativePath string, handlers ...HandlerFunc) IRoutes` — Shortcut for DELETE
- `engine.PATCH(relativePath string, handlers ...HandlerFunc) IRoutes` — Shortcut for PATCH
- `engine.HEAD(relativePath string, handlers ...HandlerFunc) IRoutes` — Shortcut for HEAD
- `engine.OPTIONS(relativePath string, handlers ...HandlerFunc) IRoutes` — Shortcut for OPTIONS
- `engine.Any(relativePath string, handlers ...HandlerFunc) IRoutes` — Registers handler for ALL HTTP methods
- `engine.Match(methods []string, relativePath string, handlers ...HandlerFunc) IRoutes` — Registers handler for specific methods
- `engine.Group(relativePath string, handlers ...HandlerFunc) *RouterGroup` — Creates a route group with optional middleware

### Static Files
- `engine.Static(relativePath, root string) IRoutes` — Serves files from directory
- `engine.StaticFS(relativePath string, fs http.FileSystem) IRoutes` — Serves files from custom FileSystem
- `engine.StaticFile(relativePath, filepath string) IRoutes` — Serves single file
- `engine.StaticFileFS(relativePath, filepath string, fs http.FileSystem) IRoutes` — Serves single file from custom FileSystem

### Error Handlers
- `engine.NoRoute(handlers ...HandlerFunc)` — Handler for 404 (no route found)
- `engine.NoMethod(handlers ...HandlerFunc)` — Handler for 405 (method not allowed)

### HTML Templates
- `engine.LoadHTMLGlob(pattern string)` — Load HTML templates matching glob pattern
- `engine.LoadHTMLFiles(files ...string)` — Load specific HTML template files
- `engine.LoadHTMLFS(fs http.FileSystem, patterns ...string)` — Load templates from custom FileSystem
- `engine.SetHTMLTemplate(templ *template.Template)` — Set custom template object
- `engine.SetFuncMap(funcMap template.FuncMap)` — Set custom template functions

### Configuration
- `engine.Delims(left, right string) *Engine` — Set template delimiters (e.g., "{{", "}}")
- `engine.SecureJsonPrefix(prefix string) *Engine` — Set prefix for JSON responses (security)
- `engine.SetTrustedProxies(trustedProxies []string) error` — Configure trusted proxy IPs for ClientIP()
- `engine.With(opts ...OptionFunc) *Engine` — Create child engine with options

### Introspection
- `engine.Routes() []RouteInfo` — Returns registered routes
- `engine.BasePath() string` — Returns base path
- `engine.Handler() http.Handler` — Returns the engine as http.Handler (for use with http.ListenAndServe)
- `engine.ServeHTTP(w http.ResponseWriter, req *http.Request)` — Standard http.Handler interface
- `engine.HandleContext(c *Context)` — Handle context (internal use)

---

## RouterGroup (gin.RouterGroup)

Groups routes with common path prefix and/or middleware.

### Group Management
- `group.Group(relativePath string, handlers ...HandlerFunc) *RouterGroup` — Create nested group
- `group.BasePath() string` — Get group base path

### Middleware
- `group.Use(middleware ...HandlerFunc) IRoutes` — Add middleware to group

### HTTP Methods (same as Engine)
- `group.GET(relativePath string, handlers ...HandlerFunc) IRoutes`
- `group.POST(relativePath string, handlers ...HandlerFunc) IRoutes`
- `group.PUT(relativePath string, handlers ...HandlerFunc) IRoutes`
- `group.DELETE(relativePath string, handlers ...HandlerFunc) IRoutes`
- `group.PATCH(relativePath string, handlers ...HandlerFunc) IRoutes`
- `group.HEAD(relativePath string, handlers ...HandlerFunc) IRoutes`
- `group.OPTIONS(relativePath string, handlers ...HandlerFunc) IRoutes`
- `group.Any(relativePath string, handlers ...HandlerFunc) IRoutes`
- `group.Match(methods []string, relativePath string, handlers ...HandlerFunc) IRoutes`
- `group.Handle(httpMethod, relativePath string, handlers ...HandlerFunc) IRoutes`

### Static Files
- `group.Static(relativePath, root string) IRoutes`
- `group.StaticFS(relativePath string, fs http.FileSystem) IRoutes`
- `group.StaticFile(relativePath, filepath string) IRoutes`
- `group.StaticFileFS(relativePath, filepath string, fs http.FileSystem) IRoutes`

---

## Context (gin.Context)

Passed to every handler function. Represents the request/response cycle.

### Context Management
- `c.Copy() *Context` — **REQUIRED before launching goroutines** to avoid race conditions
- `c.HandlerName() string` — Returns name of current handler function
- `c.HandlerNames() []string` — Returns names of all handlers in chain
- `c.Handler() HandlerFunc` — Returns current handler function
- `c.FullPath() string` — Returns matched route pattern with path parameters (e.g., "/user/:id")

---

## Request Binding

Bind request data to struct fields. **Use ShouldBind* to avoid auto-abort on error.**

### JSON Binding
- `c.BindJSON(obj any) error` — Bind JSON body (**auto-aborts with 400 on error — avoid in production**)
- `c.ShouldBindJSON(obj any) error` — Bind JSON body, returns error (recommended)
- `c.ShouldBindBodyWithJSON(obj any) error` — Re-read body for binding (for multiple reads)

### Form/Query Binding
- `c.Bind(obj any) error` — Auto-detect format (**auto-aborts with 400 on error**)
- `c.ShouldBind(obj any) error` — Auto-detect format, returns error (recommended)
- `c.ShouldBindQuery(obj any) error` — Bind query string only
- `c.ShouldBindForm(obj any) error` — Bind form data only
- `c.ShouldBindBodyWithForm(obj any) error` — Re-read body for form binding

### URI Binding
- `c.BindURI(obj any) error` — Bind URL path parameters (**auto-aborts with 400 on error**)
- `c.ShouldBindURI(obj any) error` — Bind URL path parameters, returns error (recommended)

### Header Binding
- `c.BindHeader(obj any) error` — Bind HTTP headers (**auto-aborts with 400 on error**)
- `c.ShouldBindHeader(obj any) error` — Bind HTTP headers, returns error (recommended)

### Other Formats
- `c.ShouldBindXML(obj any) error` — Bind XML body
- `c.ShouldBindYAML(obj any) error` — Bind YAML body
- `c.ShouldBindTOML(obj any) error` — Bind TOML body
- `c.ShouldBindPlain(obj any) error` — Bind plain text body
- `c.ShouldBindBodyWithXML(obj any) error` — Re-read XML body
- `c.ShouldBindBodyWithYAML(obj any) error` — Re-read YAML body
- `c.ShouldBindBodyWithTOML(obj any) error` — Re-read TOML body
- `c.ShouldBindBodyWithPlain(obj any) error` — Re-read plain text body

---

## Request Info Accessors

### URL Parameters & Query
- `c.Param(key string) string` — Get URL path parameter (`:id` in `/user/:id`)
- `c.Query(key string) string` — Get query parameter (returns "" if not found)
- `c.DefaultQuery(key, defaultValue string) string` — Get query parameter with fallback
- `c.GetQuery(key string) (string, bool)` — Get query parameter, returns (value, exists)
- `c.QueryArray(key string) []string` — Get multiple values for same query key
- `c.GetQueryArray(key string) ([]string, bool)` — Get query array, returns (values, exists)
- `c.QueryMap(key string) map[string]string` — Get query parameters as map
- `c.GetQueryMap(key string) (map[string]string, bool)` — Get query map, returns (map, exists)

### Form Data (POST/PUT)
- `c.PostForm(key string) string` — Get form field
- `c.DefaultPostForm(key, defaultValue string) string` — Get form field with fallback
- `c.GetPostForm(key string) (string, bool)` — Get form field, returns (value, exists)
- `c.PostFormArray(key string) []string` — Get multiple values for same form key
- `c.GetPostFormArray(key string) ([]string, bool)` — Get form array, returns (values, exists)
- `c.PostFormMap(key string) map[string]string` — Get form parameters as map
- `c.GetPostFormMap(key string) (map[string]string, bool)` — Get form map, returns (map, exists)

### Headers & Request Info
- `c.GetHeader(key string) string` — Get HTTP header value
- `c.ContentType() string` — Get Content-Type header
- `c.ClientIP() string` — Get client IP (respects X-Forwarded-For if trusted)
- `c.RemoteIP() string` — Get remote IP
- `c.IsWebsocket() bool` — Check if WebSocket connection
- `c.GetRawData() ([]byte, error)` — Read request body as bytes
- `c.Request *http.Request` — Access underlying http.Request

### File Upload
- `c.FormFile(name string) (*multipart.FileHeader, error)` — Get single uploaded file
- `c.MultipartForm() (*multipart.Form, error)` — Get all form data including files
- `c.SaveUploadedFile(file *multipart.FileHeader, dst string, perm ...fs.FileMode) error` — Save uploaded file to disk

---

## Response Rendering

Send data back to client. Only first call takes effect (subsequent calls are no-ops).

### JSON/Data
- `c.JSON(code int, obj any)` — Send JSON response
- `c.IndentedJSON(code int, obj any)` — Send pretty-printed JSON
- `c.PureJSON(code int, obj any)` — Send JSON without escaping HTML
- `c.AsciiJSON(code int, obj any)` — Send JSON with ASCII-only characters
- `c.SecureJSON(code int, obj any)` — Send JSON with prefix (security)
- `c.JSONP(code int, obj any)` — Send JSONP response
- `c.Data(code int, contentType string, data []byte)` — Send raw bytes with Content-Type
- `c.DataFromReader(code int, contentLength int64, contentType string, reader io.Reader, extraHeaders map[string]string)` — Stream data from reader

### Markup
- `c.HTML(code int, name string, obj any)` — Render HTML template
- `c.XML(code int, obj any)` — Send XML response
- `c.YAML(code int, obj any)` — Send YAML response
- `c.TOML(code int, obj any)` — Send TOML response
- `c.ProtoBuf(code int, obj any)` — Send Protocol Buffer response
- `c.BSON(code int, obj any)` — Send BSON response

### Text
- `c.String(code int, format string, values ...any)` — Send formatted text (like fmt.Sprintf)

### Files
- `c.File(filepath string)` — Send file as response
- `c.FileFromFS(filepath string, fs http.FileSystem)` — Send file from custom FileSystem
- `c.FileAttachment(filepath, filename string)` — Send file as attachment (triggers download)

### Redirect
- `c.Redirect(code int, location string)` — Send HTTP redirect (typically 301, 302, 307)

### Streaming & Events
- `c.Stream(func(w io.Writer) bool)` — Stream chunks of data
- `c.SSEvent(name, message string)` — Send Server-Sent Event
- `c.Render(code int, r render.Render)` — Render with custom renderer

### Content Negotiation
- `c.Negotiate(code int, config Negotiate)` — Select response format based on Accept header
- `c.NegotiateFormat(offered ...string) string` — Determine best format from offered list
- `c.SetAccepted(formats ...string)` — Set accepted formats

### Response Headers & Status
- `c.Status(code int)` — Set HTTP status code
- `c.Header(key, value string)` — Set HTTP response header
- `c.SetCookie(name, value string, maxAge int, path, domain string, secure, httpOnly bool)` — Set HTTP cookie
- `c.SetCookieData(data setCookieData)` — Set cookie with advanced options
- `c.Cookie(name string) (string, error)` — Get cookie value

---

## Flow Control

Control handler execution and request flow.

- `c.Next()` — Call next middleware/handler in chain (allows middleware to wrap handlers)
- `c.IsAborted() bool` — Check if current request was aborted
- `c.Abort()` — Stop processing, don't call remaining handlers
- `c.AbortWithStatus(code int)` — Abort with HTTP status code
- `c.AbortWithStatusJSON(code int, jsonObj any)` — Abort with status and JSON body
- `c.AbortWithStatusPureJSON(code int, jsonObj any)` — Abort with status and JSON body (no HTML escaping)
- `c.AbortWithError(code int, err error) *Error` — Abort with status and error

---

## Metadata Storage

Per-request key/value store for passing data between handlers.

- `c.Set(key any, value any)` — Store value with key
- `c.Get(key any) (value any, exists bool)` — Retrieve value and check existence
- `c.MustGet(key any) any` — Retrieve value (panics if not found)
- `c.Delete(key any)` — Delete stored value

### Type-Safe Getters (for keys stored as string)
- `c.GetString(key string) string` — Get string value
- `c.GetInt(key string) int` — Get int value
- `c.GetBool(key string) bool` — Get bool value
- `c.GetTime(key string) time.Time` — Get time.Time value
- `c.GetDuration(key string) time.Duration` — Get duration value
- `c.GetError(key string) error` — Get error value
- `c.GetIntSlice(key string) []int` — Get []int value
- `c.GetStringSlice(key string) []string` — Get []string value
- `c.GetStringMap(key string) map[string]any` — Get map[string]any value
- `c.GetStringMapString(key string) map[string]string` — Get map[string]string value
- `c.GetStringMapStringSlice(key string) map[string][]string` — Get map[string][]string value

---

## Binding Tags & Validators

Use struct tags to control binding and validation.

### Binding Tags
- `json:"field_name"` — Maps JSON field name
- `form:"field_name"` — Maps form/query field name
- `uri:"field_name"` — Maps URI parameter name
- `header:"field_name"` — Maps HTTP header name
- `binding:"required"` — Field required
- `binding:"required,email"` — Multiple validators (comma-separated)

### Common Validators (github.com/go-playground/validator/v10)
- `required` — Field must not be empty
- `email` — Valid email format
- `url` — Valid URL
- `uri` — Valid URI
- `min=N` — Minimum length/value
- `max=N` — Maximum length/value
- `len=N` — Exact length
- `gt=N`, `gte=N`, `lt=N`, `lte=N` — Greater/less than
- `oneof=val1 val2` — One of allowed values
- `alphanum` — Alphanumeric only
- `numeric` — Numeric only
- `alpha` — Letters only

### Example
```go
type User struct {
    Name  string `json:"name" binding:"required,min=2"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"required,min=0,max=120"`
}
```

---

## Middleware

### Signature
```go
type HandlerFunc = func(*gin.Context)
```

All handlers and middleware have the same signature.

### Built-in Middleware
- `gin.Logger()` — Log HTTP requests/responses
- `gin.Recovery()` — Recover from panics, return 500

### Middleware Chain Execution
```go
// Middleware can run BEFORE (c.Next) and AFTER handlers
func MyMiddleware(c *gin.Context) {
    // Run before handler
    c.Next()
    // Run after handler
}

// Execution order: Middleware1-before → Middleware2-before → Handler → Middleware2-after → Middleware1-after
```

### Adding Middleware
- Global: `engine.Use(m1, m2)`
- Group: `group.Use(m1, m2)`
- Per-route: `router.GET("/path", m1, m2, handler)`

---

## gin.H

Shorthand for `map[string]any`. Used for JSON response bodies.

```go
c.JSON(200, gin.H{
    "message": "ok",
    "data": someData,
    "count": 42,
})
```

---

## Common Patterns

### Graceful Shutdown (Go 1.8+)
```go
package main

import (
    "context"
    "net/http"
    "time"
    "github.com/gin-gonic/gin"
)

func main() {
    router := gin.Default()

    router.GET("/", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "ok"})
    })

    server := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }

    // Start server in goroutine
    go func() {
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            panic(err)
        }
    }()

    // Wait for interrupt signal, then shutdown gracefully
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        panic(err)
    }
}
```

### Request with Goroutine (MUST use c.Copy())
```go
router.GET("/task", func(c *gin.Context) {
    // WRONG: go func() { c.JSON(...) }() — race condition!

    // CORRECT:
    cCopy := c.Copy()
    go func() {
        cCopy.JSON(200, gin.H{"status": "processing"})
    }()

    c.JSON(200, gin.H{"message": "task queued"})
})
```

### Binding & Validation
```go
type CreateUserReq struct {
    Name  string `json:"name" binding:"required,min=2"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"required,min=18,max=120"`
}

router.POST("/user", func(c *gin.Context) {
    var req CreateUserReq
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // Process req...
    c.JSON(201, req)
})
```

### Middleware with Error Handling
```go
func AuthMiddleware(c *gin.Context) {
    token := c.GetHeader("Authorization")
    if token == "" {
        c.AbortWithStatusJSON(401, gin.H{"error": "missing token"})
        return
    }
    if !validateToken(token) {
        c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
        return
    }
    // Token valid, continue
    c.Next()
}

router.Use(AuthMiddleware)
```

### Route Groups
```go
api := router.Group("/api")
{
    v1 := api.Group("/v1")
    {
        v1.GET("/users", getUsers)
        v1.POST("/users", createUser)
    }
}
// Routes: /api/v1/users (GET/POST)
```

---

**End of GIN-API-REFERENCE.md**
