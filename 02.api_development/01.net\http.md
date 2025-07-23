# Introduction to net/http: Go's HTTP Foundation

## Table of Contents
1. [What is net/http?](#what-is-nethttp)
2. [HTTP Fundamentals](#http-fundamentals)
3. [Basic HTTP Server](#basic-http-server)
4. [HTTP Handlers: The Heart of Web Services](#http-handlers-the-heart-of-web-services)
5. [HTTP Client: Making Requests](#http-client-making-requests)
6. [Request and Response Anatomy](#request-and-response-anatomy)
7. [Routing and Multiplexing](#routing-and-multiplexing)
8. [Middleware Patterns](#middleware-patterns)
9. [Real-World Examples](#real-world-examples)

---

## What is net/http?

The `net/http` package is Go's **standard library package for HTTP**. It provides everything you need to:
- Build HTTP servers (web servers, APIs)
- Make HTTP requests (HTTP client)
- Handle HTTP protocols, headers, cookies, and more

### Why net/http is Special

1. **Production Ready**: Used by companies like Google, Docker, Kubernetes
2. **Complete**: Handles HTTP/1.1, HTTP/2, and WebSockets
3. **Concurrent**: Built on Go's goroutines - handles thousands of connections
4. **Simple**: Clean, intuitive API design
5. **Fast**: Excellent performance out of the box

---

## HTTP Fundamentals

Before diving into code, let's understand what HTTP actually is:

### What is HTTP?

**HTTP (HyperText Transfer Protocol)** is a protocol for communication between clients and servers.

```
Client (Browser/App)  ←→  Server (Your Go Program)
        │                       │
        │  1. HTTP Request       │
        │ ──────────────────→    │
        │                       │
        │  2. HTTP Response      │
        │ ←──────────────────    │
```

### HTTP Request Structure

```
GET /api/users HTTP/1.1          ← Request Line (Method, Path, Version)
Host: example.com                ← Headers
Content-Type: application/json   ← Headers
Authorization: Bearer token123   ← Headers
                                 ← Empty line
{"name": "John"}                 ← Body (optional)
```

### HTTP Response Structure

```
HTTP/1.1 200 OK                  ← Status Line (Version, Status Code, Status Text)
Content-Type: application/json   ← Headers
Content-Length: 25              ← Headers
                                ← Empty line
{"id": 1, "name": "John"}       ← Body
```

### Common HTTP Methods

```go
GET    // Retrieve data
POST   // Create new resource
PUT    // Update/replace resource
PATCH  // Partial update
DELETE // Remove resource
HEAD   // Get headers only
OPTIONS // Get allowed methods
```

---

## Basic HTTP Server

### Your First HTTP Server

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    // Start a simple HTTP server
    http.ListenAndServe(":8080", nil)
}
```

**Wait, this doesn't do anything!** That's because we haven't defined any handlers. Let's fix that:

```go
package main

import (
    "fmt"
    "net/http"
)

// Handler function
func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    // Register handler for the "/" path
    http.HandleFunc("/", helloHandler)
    
    // Start server on port 8080
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

### Understanding the Code

1. **`http.HandleFunc("/", helloHandler)`**: Register a handler function for the "/" path
2. **`helloHandler`**: Function that handles HTTP requests
3. **`http.ResponseWriter`**: Interface to write the HTTP response
4. **`*http.Request`**: Struct containing the HTTP request data
5. **`http.ListenAndServe(":8080", nil)`**: Start server on port 8080

### Multiple Routes

```go
package main

import (
    "fmt"
    "net/http"
)

func homeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Welcome to the Home Page!")
}

func aboutHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "This is the About Page!")
}

func contactHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Contact us at: contact@example.com")
}

func main() {
    // Register multiple routes
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/about", aboutHandler)
    http.HandleFunc("/contact", contactHandler)
    
    fmt.Println("Server running on http://localhost:8080")
    fmt.Println("Routes:")
    fmt.Println("  http://localhost:8080/")
    fmt.Println("  http://localhost:8080/about")
    fmt.Println("  http://localhost:8080/contact")
    
    http.ListenAndServe(":8080", nil)
}
```

---

## HTTP Handlers: The Heart of Web Services

### What is a Handler?

A handler is a function that processes HTTP requests and generates responses. In Go, handlers must have this signature:

```go
func handlerName(w http.ResponseWriter, r *http.Request) {
    // Process request and write response
}
```

### Handler Function Deep Dive

```go
func detailedHandler(w http.ResponseWriter, r *http.Request) {
    // 1. Read information from the request
    method := r.Method        // GET, POST, PUT, etc.
    path := r.URL.Path       // /api/users
    query := r.URL.Query()   // ?name=john&age=25
    headers := r.Header      // Request headers
    
    // 2. Log request information
    fmt.Printf("Method: %s\n", method)
    fmt.Printf("Path: %s\n", path)
    fmt.Printf("Query: %v\n", query)
    
    // 3. Set response headers
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-Custom-Header", "Hello from Go!")
    
    // 4. Set status code (optional, defaults to 200)
    w.WriteHeader(http.StatusOK) // 200
    
    // 5. Write response body
    fmt.Fprintf(w, `{"message": "Hello from detailed handler", "path": "%s"}`, path)
}
```

### Different Ways to Create Handlers

#### 1. Function Handlers
```go
func simpleHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Simple function handler")
}

http.HandleFunc("/simple", simpleHandler)
```

#### 2. Handler Interface
```go
type CustomHandler struct {
    message string
}

// Implement the http.Handler interface
func (h CustomHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Custom handler says: %s", h.message)
}

func main() {
    customHandler := CustomHandler{message: "Hello from custom handler!"}
    http.Handle("/custom", customHandler)
    
    http.ListenAndServe(":8080", nil)
}
```

#### 3. Handler with Closure (for sharing data)
```go
func createCounterHandler() http.HandlerFunc {
    count := 0
    return func(w http.ResponseWriter, r *http.Request) {
        count++
        fmt.Fprintf(w, "This handler has been called %d times", count)
    }
}

func main() {
    http.HandleFunc("/counter", createCounterHandler())
    http.ListenAndServe(":8080", nil)
}
```

### HTTP Methods Handling

```go
func methodHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        fmt.Fprintf(w, "This is a GET request")
    case http.MethodPost:
        fmt.Fprintf(w, "This is a POST request")
    case http.MethodPut:
        fmt.Fprintf(w, "This is a PUT request")
    case http.MethodDelete:
        fmt.Fprintf(w, "This is a DELETE request")
    default:
        // Method not allowed
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Method %s not allowed", r.Method)
    }
}

func main() {
    http.HandleFunc("/api", methodHandler)
    http.ListenAndServe(":8080", nil)
}
```

---

## HTTP Client: Making Requests

Go's `net/http` package also provides a powerful HTTP client for making requests to other servers.

### Basic HTTP Client

```go
package main

import (
    "fmt"
    "io"
    "net/http"
)

func main() {
    // Simple GET request
    resp, err := http.Get("https://api.github.com/users/golang")
    if err != nil {
        fmt.Printf("Error making request: %v\n", err)
        return
    }
    defer resp.Body.Close() // Always close the response body!
    
    // Read response body
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        fmt.Printf("Error reading response: %v\n", err)
        return
    }
    
    // Print response information
    fmt.Printf("Status Code: %d\n", resp.StatusCode)
    fmt.Printf("Status: %s\n", resp.Status)
    fmt.Printf("Content-Type: %s\n", resp.Header.Get("Content-Type"))
    fmt.Printf("Body: %s\n", string(body))
}
```

### Different HTTP Methods with Client

```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "net/http"
    "strings"
)

func makeGETRequest() {
    resp, err := http.Get("https://httpbin.org/get")
    if err != nil {
        fmt.Printf("GET Error: %v\n", err)
        return
    }
    defer resp.Body.Close()
    
    fmt.Printf("GET Status: %s\n", resp.Status)
}

func makePOSTRequest() {
    // JSON data
    jsonData := `{"name": "John", "email": "john@example.com"}`
    
    resp, err := http.Post(
        "https://httpbin.org/post",
        "application/json",
        strings.NewReader(jsonData),
    )
    if err != nil {
        fmt.Printf("POST Error: %v\n", err)
        return
    }
    defer resp.Body.Close()
    
    fmt.Printf("POST Status: %s\n", resp.Status)
}

func makeCustomRequest() {
    // Create request
    req, err := http.NewRequest("PUT", "https://httpbin.org/put", bytes.NewReader([]byte("data")))
    if err != nil {
        fmt.Printf("Error creating request: %v\n", err)
        return
    }
    
    // Set headers
    req.Header.Set("Content-Type", "text/plain")
    req.Header.Set("Authorization", "Bearer token123")
    
    // Make request
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        fmt.Printf("PUT Error: %v\n", err)
        return
    }
    defer resp.Body.Close()
    
    fmt.Printf("PUT Status: %s\n", resp.Status)
}

func main() {
    makeGETRequest()
    makePOSTRequest()
    makeCustomRequest()
}
```

---

## Request and Response Anatomy

### Understanding http.Request

```go
func analyzeRequest(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "=== REQUEST ANALYSIS ===\n\n")
    
    // Basic information
    fmt.Fprintf(w, "Method: %s\n", r.Method)
    fmt.Fprintf(w, "URL: %s\n", r.URL.String())
    fmt.Fprintf(w, "Protocol: %s\n", r.Proto)
    fmt.Fprintf(w, "Host: %s\n", r.Host)
    fmt.Fprintf(w, "Remote Address: %s\n", r.RemoteAddr)
    
    // URL components
    fmt.Fprintf(w, "\n=== URL COMPONENTS ===\n")
    fmt.Fprintf(w, "Scheme: %s\n", r.URL.Scheme)
    fmt.Fprintf(w, "Host: %s\n", r.URL.Host)
    fmt.Fprintf(w, "Path: %s\n", r.URL.Path)
    fmt.Fprintf(w, "RawQuery: %s\n", r.URL.RawQuery)
    fmt.Fprintf(w, "Fragment: %s\n", r.URL.Fragment)
    
    // Query parameters
    fmt.Fprintf(w, "\n=== QUERY PARAMETERS ===\n")
    for key, values := range r.URL.Query() {
        for _, value := range values {
            fmt.Fprintf(w, "%s: %s\n", key, value)
        }
    }
    
    // Headers
    fmt.Fprintf(w, "\n=== HEADERS ===\n")
    for key, values := range r.Header {
        for _, value := range values {
            fmt.Fprintf(w, "%s: %s\n", key, value)
        }
    }
    
    // Form data (for POST requests)
    if r.Method == "POST" {
        r.ParseForm()
        fmt.Fprintf(w, "\n=== FORM DATA ===\n")
        for key, values := range r.Form {
            for _, value := range values {
                fmt.Fprintf(w, "%s: %s\n", key, value)
            }
        }
    }
}

func main() {
    http.HandleFunc("/analyze", analyzeRequest)
    
    fmt.Println("Visit http://localhost:8080/analyze?name=john&age=25")
    http.ListenAndServe(":8080", nil)
}
```

### Working with Request Body

```go
func handleJSONRequest(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Only POST method is allowed")
        return
    }
    
    // Read request body
    body, err := io.ReadAll(r.Body)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Error reading request body: %v", err)
        return
    }
    defer r.Body.Close()
    
    // Check content type
    contentType := r.Header.Get("Content-Type")
    if contentType != "application/json" {
        w.WriteHeader(http.StatusUnsupportedMediaType)
        fmt.Fprintf(w, "Expected application/json, got %s", contentType)
        return
    }
    
    // Process JSON (in real app, you'd unmarshal to struct)
    fmt.Printf("Received JSON: %s\n", string(body))
    
    // Send response
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    fmt.Fprintf(w, `{"status": "success", "received": %s}`, string(body))
}
```

### Understanding http.ResponseWriter

```go
func demonstrateResponse(w http.ResponseWriter, r *http.Request) {
    // 1. Set headers BEFORE writing body or status code
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-API-Version", "1.0")
    w.Header().Set("Cache-Control", "no-cache")
    
    // 2. Set status code (optional, defaults to 200)
    w.WriteHeader(http.StatusOK) // 200
    
    // 3. Write response body
    response := `{
        "message": "This is a JSON response",
        "timestamp": "2024-01-01T12:00:00Z",
        "data": {
            "user": "john",
            "role": "admin"
        }
    }`
    
    w.Write([]byte(response))
    
    // Note: Once you write to the body, you can't change headers or status code!
}
```

---

## Routing and Multiplexing

### Default ServeMux

Go's default router (ServeMux) is simple but limited:

```go
func main() {
    // Default ServeMux patterns
    http.HandleFunc("/", homeHandler)              // Matches exactly "/" or longer paths
    http.HandleFunc("/api/", apiHandler)           // Matches "/api/" and subpaths
    http.HandleFunc("/users/profile", profileHandler) // Exact match only
    
    http.ListenAndServe(":8080", nil) // nil uses DefaultServeMux
}
```

### Custom ServeMux

```go
func main() {
    // Create custom ServeMux
    mux := http.NewServeMux()
    
    mux.HandleFunc("/", homeHandler)
    mux.HandleFunc("/api/users", usersHandler)
    mux.HandleFunc("/api/posts", postsHandler)
    
    // Use custom mux
    http.ListenAndServe(":8080", mux)
}
```

### Path Parameters (Manual Parsing)

```go
func userHandler(w http.ResponseWriter, r *http.Request) {
    // Extract user ID from path: /users/123
    path := r.URL.Path
    if strings.HasPrefix(path, "/users/") {
        userID := strings.TrimPrefix(path, "/users/")
        fmt.Fprintf(w, "User ID: %s", userID)
    } else {
        w.WriteHeader(http.StatusNotFound)
        fmt.Fprintf(w, "User not found")
    }
}

func main() {
    http.HandleFunc("/users/", userHandler) // Note the trailing slash
    http.ListenAndServe(":8080", nil)
}
```

---

## Middleware Patterns

Middleware allows you to wrap handlers with additional functionality.

### Basic Middleware

```go
// Middleware function type
type Middleware func(http.HandlerFunc) http.HandlerFunc

// Logging middleware
func loggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        fmt.Printf("[%s] %s %s", 
            time.Now().Format("2006-01-02 15:04:05"),
            r.Method, 
            r.URL.Path)
        
        // Call the next handler
        next(w, r)
        
        duration := time.Since(start)
        fmt.Printf(" - %v\n", duration)
    }
}

// Authentication middleware
func authMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Check for authorization header
        auth := r.Header.Get("Authorization")
        if auth == "" {
            w.WriteHeader(http.StatusUnauthorized)
            fmt.Fprintf(w, "Authorization header required")
            return
        }
        
        // In real app, validate the token
        if auth != "Bearer valid-token" {
            w.WriteHeader(http.StatusUnauthorized)
            fmt.Fprintf(w, "Invalid token")
            return
        }
        
        // Token is valid, continue
        next(w, r)
    }
}

// CORS middleware
func corsMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        // Handle preflight requests
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }
        
        next(w, r)
    }
}

// Chain multiple middlewares
func chainMiddlewares(handler http.HandlerFunc, middlewares ...Middleware) http.HandlerFunc {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

func protectedHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "This is a protected resource!")
}

func publicHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "This is a public resource!")
}

func main() {
    // Public endpoint with logging and CORS
    http.HandleFunc("/public", 
        chainMiddlewares(publicHandler, loggingMiddleware, corsMiddleware))
    
    // Protected endpoint with all middlewares
    http.HandleFunc("/protected", 
        chainMiddlewares(protectedHandler, loggingMiddleware, corsMiddleware, authMiddleware))
    
    fmt.Println("Server running on :8080")
    fmt.Println("Try:")
    fmt.Println("  curl http://localhost:8080/public")
    fmt.Println("  curl http://localhost:8080/protected")
    fmt.Println("  curl -H 'Authorization: Bearer valid-token' http://localhost:8080/protected")
    
    http.ListenAndServe(":8080", nil)
}
```

---

## Real-World Examples

### Example 1: Simple REST API

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "strings"
    "time"
)

// User model
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

// In-memory storage (in real app, use database)
var users = []User{
    {ID: 1, Name: "John Doe", Email: "john@example.com", CreatedAt: time.Now()},
    {ID: 2, Name: "Jane Smith", Email: "jane@example.com", CreatedAt: time.Now()},
}
var nextID = 3

// Get all users
func getUsersHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        w.WriteHeader(http.StatusMethodNotAllowed)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

// Get user by ID
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        w.WriteHeader(http.StatusMethodNotAllowed)
        return
    }
    
    // Extract ID from path: /users/123
    path := strings.TrimPrefix(r.URL.Path, "/users/")
    if path == "" {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "User ID required")
        return
    }
    
    id, err := strconv.Atoi(path)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Invalid user ID")
        return
    }
    
    // Find user
    for _, user := range users {
        if user.ID == id {
            w.Header().Set("Content-Type", "application/json")
            json.NewEncoder(w).Encode(user)
            return
        }
    }
    
    w.WriteHeader(http.StatusNotFound)
    fmt.Fprintf(w, "User not found")
}

// Create new user
func createUserHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        w.WriteHeader(http.StatusMethodNotAllowed)
        return
    }
    
    var newUser User
    err := json.NewDecoder(r.Body).Decode(&newUser)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Invalid JSON")
        return
    }
    
    // Validate required fields
    if newUser.Name == "" || newUser.Email == "" {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Name and email are required")
        return
    }
    
    // Create user
    newUser.ID = nextID
    nextID++
    newUser.CreatedAt = time.Now()
    
    users = append(users, newUser)
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(newUser)
}

// Router function
func usersRouter(w http.ResponseWriter, r *http.Request) {
    path := r.URL.Path
    
    switch {
    case path == "/users" && r.Method == http.MethodGet:
        getUsersHandler(w, r)
    case path == "/users" && r.Method == http.MethodPost:
        createUserHandler(w, r)
    case strings.HasPrefix(path, "/users/") && r.Method == http.MethodGet:
        getUserHandler(w, r)
    default:
        w.WriteHeader(http.StatusNotFound)
        fmt.Fprintf(w, "Endpoint not found")
    }
}

func main() {
    http.HandleFunc("/users", usersRouter)
    http.HandleFunc("/users/", usersRouter)
    
    fmt.Println("REST API Server running on :8080")
    fmt.Println("Endpoints:")
    fmt.Println("  GET    /users      - Get all users")
    fmt.Println("  POST   /users      - Create user")
    fmt.Println("  GET    /users/{id} - Get user by ID")
    fmt.Println()
    fmt.Println("Try these commands:")
    fmt.Println("  curl http://localhost:8080/users")
    fmt.Println("  curl http://localhost:8080/users/1")
    fmt.Println("  curl -X POST -H 'Content-Type: application/json' \\")
    fmt.Println("       -d '{\"name\":\"Bob\",\"email\":\"bob@example.com\"}' \\")
    fmt.Println("       http://localhost:8080/users")
    
    http.ListenAndServe(":8080", nil)
}
```

### Example 2: File Server with Upload

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "os"
    "path/filepath"
)

func uploadHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        w.WriteHeader(http.StatusMethodNotAllowed)
        return
    }
    
    // Parse multipart form (32MB max memory)
    err := r.ParseMultipartForm(32 << 20)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Error parsing form: %v", err)
        return
    }
    
    // Get file from form
    file, header, err := r.FormFile("file")
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Error getting file: %v", err)
        return
    }
    defer file.Close()
    
    // Create uploads directory if it doesn't exist
    os.MkdirAll("uploads", 0755)
    
    // Create destination file
    dst, err := os.Create(filepath.Join("uploads", header.Filename))
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintf(w, "Error creating file: %v", err)
        return
    }
    defer dst.Close()
    
    // Copy uploaded file to destination
    _, err = io.Copy(dst, file)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintf(w, "Error saving file: %v", err)
        return
    }
    
    fmt.Fprintf(w, "File uploaded successfully: %s", header.Filename)
}

func uploadFormHandler(w http.ResponseWriter, r *http.Request) {
    html := `
    <!DOCTYPE html>
    <html>
    <head>
        <title>File Upload</title>
    </head>
    <body>
        <h2>Upload File</h2>
        <form action="/upload" method="post" enctype="multipart/form-data">
            <input type="file" name="file" required>
            <input type="submit" value="Upload">
        </form>
    </body>
    </html>
    `
    w.Header().Set("Content-Type", "text/html")
    fmt.Fprintf(w, html)
}

func main() {
    // File upload endpoints
    http.HandleFunc("/", uploadFormHandler)
    http.HandleFunc("/upload", uploadHandler)
    
    // Serve uploaded files
    http.Handle("/files/", http.StripPrefix("/files/", http.FileServer(http.Dir("uploads"))))
    
    fmt.Println("File server running on :8080")
    fmt.Println("  http://localhost:8080/           - Upload form")
    fmt.Println("  http://localhost:8080/files/     - View uploaded files")
    
    http.ListenAndServe(":8080", nil)
}
```

### Example 3: HTTP Client with Timeout and Retry

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

type APIClient struct {
    baseURL    string
    httpClient *http.Client
}

func NewAPIClient(baseURL string, timeout time.Duration) *APIClient {
    return &APIClient{
        baseURL: baseURL,
        httpClient: &http.Client{
            Timeout: timeout,
        },
    }
}

func (c *APIClient) Get(endpoint string) (*http.Response, error) {
    url := c.baseURL + endpoint
    
    req, err := http.NewRequest("GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    req.Header.Set("User-Agent", "GoClient/1.0")
    req.Header.Set("Accept", "application/json")
    
    return c.doWithRetry(req, 3)
}

func (c *APIClient) Post(endpoint string, data interface{}) (*http.Response, error) {
    url := c.baseURL + endpoint
    
    jsonData, err := json.Marshal(data)
    if err != nil {
        return nil, err
    }
    
    req, err := http.NewRequest("POST", url, bytes.NewBuffer(jsonData))
    if err != nil {
        return nil, err
    }
    
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("User-Agent", "GoClient/1.0")
    
    return c.doWithRetry(req, 3)
}

func (c *APIClient) doWithRetry(req *http.Request, maxRetries int) (*http.Response, error) {
    var resp *http.Response
    var err error
    
    for attempt := 0; attempt <= maxRetries; attempt++ {
        if attempt > 0 {
            fmt.Printf("Retry attempt %d...\n", attempt)
            time.Sleep(time.Duration(attempt) * time.Second)
        }
        
        resp, err = c.httpClient.Do(req)
        if err == nil && resp.StatusCode < 500 {
            return resp, nil
        }
        
        if resp != nil {
            resp.Body.Close()
        }
    }
    
    return nil, fmt.Errorf("request failed after %d retries: %v", maxRetries, err)
}

func main() {
    client := NewAPIClient("https://httpbin.org", 10*time.Second)
    
    // Test GET request
    fmt.Println("Making GET request...")
    resp, err := client.Get("/get")
    if err != nil {
        fmt.Printf("GET Error: %v\n", err)
    } else {
        fmt.Printf("GET Status: %s\n", resp.Status)
        resp.Body.Close()
    }
    
    // Test POST request
    fmt.Println("\nMaking POST request...")
    data := map[string]interface{}{
        "name":  "John",
        "email": "john@example.com",
    }
    
    resp, err = client.Post("/post", data)
    if err != nil {
        fmt.Printf("POST Error: %v\n", err)
    } else {
        fmt.Printf("POST Status: %s\n", resp.Status)
        resp.Body.Close()
    }
}
```

## Key Takeaways

### Server Side
1. **Handler Functions**: `func(w http.ResponseWriter, r *http.Request)`
2. **Set headers before writing body**: `w.Header().Set()` comes first
3. **Status codes**: Use `w.WriteHeader()` to set custom status codes
4. **Always close request bodies**: `defer r.Body.Close()`
5. **Middleware pattern**: Chain functionality around handlers

### Client Side
1. **Always close response bodies**: `defer resp.Body.Close()`
2. **Set timeouts**: Use `http.Client` with custom timeout
3. **Handle errors gracefully**: Network requests can fail
4. **Custom headers**: Use `req.Header.Set()` for API keys, auth, etc.
5. **Context for cancellation**: Use `context.Context` for request cancellation

### Best Practices
1. **Use custom ServeMux**: Don't rely on default global mux in production
2. **Implement middleware**: Logging, authentication, CORS, etc.
3. **Validate input**: Always validate and sanitize user input
4. **Handle errors properly**: Return appropriate HTTP status codes
5. **Use struct tags**: For JSON marshaling/unmarshaling
6. **Implement graceful shutdown**: Handle server termination gracefully

The `net/http` package provides everything you need to build production-ready web services and HTTP clients. While it's simple to get started, it's also powerful enough for complex applications used by major companies worldwide.
