---
name: golang-backend-development
description: Complete guide for Go backend development including concurrency patterns, web servers, database integration, microservices, and production deployment
tags: [golang, go, concurrency, web-servers, microservices, backend, goroutines, channels, grpc, rest-api]
tier: tier-1
---

# Go Backend Development

A comprehensive skill for building production-grade backend systems with Go. Master goroutines, channels, web servers, database integration, microservices architecture, and deployment patterns for scalable, concurrent backend applications.

## When to Use This Skill

Use this skill when:

- Building high-performance web servers and REST APIs
- Developing microservices architectures with gRPC or HTTP
- Implementing concurrent processing with goroutines and channels
- Creating real-time systems requiring high throughput
- Building database-backed applications with connection pooling
- Developing cloud-native applications for containerized deployment
- Writing performance-critical backend services
- Building distributed systems with service discovery
- Implementing event-driven architectures
- Creating CLI tools and system utilities with networking capabilities
- Developing WebSocket servers for real-time communication
- Building data processing pipelines with concurrent workers

**Go excels at:**
- Network programming and HTTP services
- Concurrent processing with lightweight goroutines
- System-level programming with garbage collection
- Cross-platform compilation
- Fast compilation times for rapid development
- Built-in testing and benchmarking

## Core Concepts

### 1. Goroutines: Lightweight Concurrency

Goroutines are lightweight threads managed by the Go runtime. They enable concurrent execution with minimal overhead.

**Key Characteristics:**
- Extremely lightweight (start with ~2KB stack)
- Multiplexed onto OS threads by the runtime
- Thousands or millions can run concurrently
- Scheduled cooperatively with integrated scheduler

**Basic Goroutine Pattern:**

```go
func main() {
    // Launch concurrent computation
    go expensiveComputation(x, y, z)
    anotherExpensiveComputation(a, b, c)
}
```

The `go` keyword launches a new goroutine, allowing `expensiveComputation` to run concurrently with `anotherExpensiveComputation`. This is fundamental to Go's concurrency model.

**Common Use Cases:**
- Background processing
- Concurrent API calls
- Parallel data processing
- Real-time event handling
- Connection handling in servers

### 2. Channels: Safe Communication

Channels provide type-safe communication between goroutines, eliminating the need for explicit locks in many scenarios.

**Channel Types:**

```go
// Unbuffered channel - synchronous communication
ch := make(chan int)

// Buffered channel - asynchronous up to buffer size
ch := make(chan int, 100)

// Read-only channel
func receive(ch <-chan int) { /* ... */ }

// Write-only channel
func send(ch chan<- int) { /* ... */ }
```

**Synchronization with Channels:**

```go
func computeAndSend(ch chan int, x, y, z int) {
    ch <- expensiveComputation(x, y, z)
}

func main() {
    ch := make(chan int)
    go computeAndSend(ch, x, y, z)
    v2 := anotherExpensiveComputation(a, b, c)
    v1 := <-ch  // Block until result available
    fmt.Println(v1, v2)
}
```

This pattern ensures both computations complete before proceeding, with the channel providing both communication and synchronization.

**Channel Patterns:**
- Producer-consumer
- Fan-out/fan-in
- Pipeline stages
- Timeouts and cancellation
- Semaphores and rate limiting

### 3. Select Statement: Multiplexing Channels

The `select` statement enables multiplexing multiple channel operations, similar to a switch for channels.

**Timeout Implementation:**

```go
timeout := make(chan bool, 1)
go func() {
    time.Sleep(1 * time.Second)
    timeout <- true
}()

select {
case <-ch:
    // Read from ch succeeded
case <-timeout:
    // Operation timed out
}
```

**Context-Based Cancellation:**

```go
select {
case result := <-resultCh:
    return result
case <-ctx.Done():
    return ctx.Err()
}
```

### 4. Context Package: Request-Scoped Values

The `context.Context` interface manages deadlines, cancellation signals, and request-scoped values across API boundaries.

**Context Interface:**

```go
type Context interface {
    // Done returns a channel closed when work should be canceled
    Done() <-chan struct{}

    // Err returns why context was canceled
    Err() error

    // Deadline returns when work should be canceled
    Deadline() (deadline time.Time, ok bool)

    // Value returns request-scoped value
    Value(key any) any
}
```

**Creating Contexts:**

```go
// Background context - never canceled
ctx := context.Background()

// With cancellation
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// With timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// With deadline
deadline := time.Now().Add(10 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// With values (use sparingly - only for request-scoped data)
ctx = context.WithValue(parentCtx, key, value)
```

**Context Propagation Example:**

```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // Request context is automatically canceled when:
    // 1. Client disconnects
    // 2. Handler returns
    ctx := r.Context()
    
    // Pass context down the call chain
    result, err := fetchData(ctx, userID)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            // Client disconnected, don't log as error
            return
        }
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Request timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(result)
}

func fetchData(ctx context.Context, userID string) (*Data, error) {
    // Check for cancellation before expensive operations
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    // Perform work, passing context to database/HTTP calls
    return queryDatabase(ctx, userID)
}
```

**Best Practices:**
- Always pass context as first parameter: `func DoSomething(ctx context.Context, ...)`
- Call `defer cancel()` immediately after creating cancelable context
- Propagate context through call chain
- Check `ctx.Done()` in long-running operations
- Use context values only for request-scoped data, not optional parameters
- Never store contexts in struct fields (except for bridging APIs)

### 5. WaitGroup: Coordinating Goroutines

`sync.WaitGroup` waits for a collection of goroutines to finish.

**Basic Pattern:**

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        // Do work
    }(i)
}

wg.Wait()  // Block until all goroutines complete
```

**Common Use Cases:**
- Waiting for parallel tasks
- Coordinating worker pools
- Ensuring cleanup completion
- Synchronizing shutdown

### 6. Mutex: Protecting Shared State

When shared state is necessary, use `sync.Mutex` or `sync.RWMutex` for protection.

**Mutex Pattern:**

```go
var (
    service   map[string]net.Addr
    serviceMu sync.Mutex
)

func RegisterService(name string, addr net.Addr) {
    serviceMu.Lock()
    defer serviceMu.Unlock()
    service[name] = addr
}

func LookupService(name string) net.Addr {
    serviceMu.Lock()
    defer serviceMu.Unlock()
    return service[name]
}
```

**RWMutex for Read-Heavy Workloads:**

```go
var (
    cache   map[string]interface{}
    cacheMu sync.RWMutex
)

func Get(key string) interface{} {
    cacheMu.RLock()
    defer cacheMu.RUnlock()
    return cache[key]
}

func Set(key string, value interface{}) {
    cacheMu.Lock()
    defer cacheMu.Unlock()
    cache[key] = value
}
```

### 7. Concurrent Web Server Pattern

Go's standard pattern for handling concurrent connections:

```go
for {
    rw := l.Accept()
    conn := newConn(rw, handler)
    go conn.serve()  // Handle each connection concurrently
}
```

Each accepted connection is handled in its own goroutine, allowing the server to scale to thousands of concurrent connections efficiently.

## Web Server Development

### HTTP Server Basics

**Simple HTTP Server:**

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe("localhost:8080", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello!")
}
```

### Request Handling Patterns

**Handler Functions:**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Read request
    method := r.Method
    path := r.URL.Path
    query := r.URL.Query()

    // Write response
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    fmt.Fprintf(w, `{"message": "success"}`)
}
```

**Handler Structs:**

```go
type APIHandler struct {
    db *sql.DB
    logger *log.Logger
}

func (h *APIHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Access dependencies
    h.logger.Printf("Request: %s %s", r.Method, r.URL.Path)
    // Handle request
}
```

### JSON Handling Best Practices

**Struct Tags and Security:**

```go
type User struct {
    ID        int       `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    Password  string    `json:"-"`                    // Never send password in JSON
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at,omitempty"` // Omit if zero value
    DeletedAt *time.Time `json:"deleted_at,omitempty"` // Omit if nil
}

// Separate request/response types for better security
type CreateUserRequest struct {
    Email    string `json:"email"`
    Name     string `json:"name"`
    Password string `json:"password"`
}

type UserResponse struct {
    ID        int       `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    CreatedAt time.Time `json:"created_at"`
}
```

**Strict JSON Parsing:**

```go
func createUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    
    // Limit request body size (prevent DoS)
    r.Body = http.MaxBytesReader(w, r.Body, 1048576) // 1MB limit
    
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields() // Reject unknown fields
    
    if err := decoder.Decode(&req); err != nil {
        var syntaxError *json.SyntaxError
        var unmarshalTypeError *json.UnmarshalTypeError
        
        switch {
        case errors.As(err, &syntaxError):
            http.Error(w, "Invalid JSON syntax", http.StatusBadRequest)
        case errors.As(err, &unmarshalTypeError):
            http.Error(w, fmt.Sprintf("Invalid type for field %s", unmarshalTypeError.Field), http.StatusBadRequest)
        case strings.HasPrefix(err.Error(), "json: unknown field"):
            http.Error(w, "Unknown field in request", http.StatusBadRequest)
        default:
            http.Error(w, "Invalid JSON", http.StatusBadRequest)
        }
        return
    }
    
    // Validate request
    if req.Email == "" || req.Password == "" {
        http.Error(w, "Missing required fields", http.StatusBadRequest)
        return
    }
    
    // Process request...
}
```

**JSON Response Helper:**

```go
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    
    if err := json.NewEncoder(w).Encode(data); err != nil {
        log.Printf("Error encoding JSON: %v", err)
    }
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

// Usage
func handler(w http.ResponseWriter, r *http.Request) {
    user, err := getUser(id)
    if err != nil {
        respondError(w, http.StatusNotFound, "User not found")
        return
    }
    
    respondJSON(w, http.StatusOK, UserResponse{
        ID:    user.ID,
        Email: user.Email,
        Name:  user.Name,
    })
}
```

### Middleware Pattern

**Logging Middleware:**

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// Usage
http.Handle("/api/", loggingMiddleware(apiHandler))
```

**Authentication Middleware:**

```go
// Use custom type for context keys to avoid collisions between packages
type contextKey string
const userIDKey contextKey = "userID"

func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if !isValidToken(token) {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        
        userID := extractUserID(token)
        ctx := context.WithValue(r.Context(), userIDKey, userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**CORS Middleware:**

```go
// allowedOrigins: use specific origins in production, e.g. []string{"https://myapp.com"}
// Avoid "*" in production - it allows any site to make credentialed requests
func corsMiddleware(allowedOrigins []string) func(http.Handler) http.Handler {
    originSet := make(map[string]bool)
    for _, o := range allowedOrigins {
        originSet[o] = true
    }
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")
            if origin != "" && (len(originSet) == 0 || originSet["*"] || originSet[origin]) {
                w.Header().Set("Access-Control-Allow-Origin", origin)
            }
            w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
            w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
            
            if r.Method == "OPTIONS" {
                w.WriteHeader(http.StatusOK)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// Usage: corsMiddleware([]string{"https://myapp.com"})(handler) or corsMiddleware([]string{"*"})(handler) for dev
```

**Middleware Chaining:**

```go
// Manual chaining (corsMiddleware takes allowed origins, returns middleware)
handler := loggingMiddleware(authMiddleware(corsMiddleware([]string{"https://myapp.com"})(apiHandler)))
http.Handle("/api/", handler)

// Helper function for cleaner chaining
func chain(h http.Handler, middleware ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middleware) - 1; i >= 0; i-- {
        h = middleware[i](h)
    }
    return h
}

// Usage
handler := chain(apiHandler, loggingMiddleware, authMiddleware, corsMiddleware([]string{"https://myapp.com"}))
```

**Popular Router with Middleware (chi):**

```go
import "github.com/go-chi/chi/v5"

func main() {
    r := chi.NewRouter()
    
    // Global middleware
    r.Use(loggingMiddleware)
    r.Use(corsMiddleware([]string{"https://myapp.com"}))
    
    // Public routes
    r.Post("/auth/login", loginHandler)
    
    // Protected routes
    r.Group(func(r chi.Router) {
        r.Use(authMiddleware)
        r.Get("/api/users", listUsers)
        r.Get("/api/users/{id}", getUser)
        r.Post("/api/users", createUser)
    })
    
    http.ListenAndServe(":8080", r)
}
```

### Context in HTTP Handlers

**HTTP Request with Context:**

```go
func handleSearch(w http.ResponseWriter, req *http.Request) {
    ctx, cancel := context.WithTimeout(req.Context(), 5*time.Second)
    defer cancel()

    // Check query parameter
    query := req.FormValue("q")
    if query == "" {
        http.Error(w, "missing query", http.StatusBadRequest)
        return
    }

    // Perform search with context
    results, err := performSearch(ctx, query)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Search timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Render results
    respondJSON(w, http.StatusOK, results)
}
```

**Context-Aware HTTP Request:**

```go
func httpDo(ctx context.Context, req *http.Request,
            f func(*http.Response, error) error) error {
    c := &http.Client{}

    // Run request in goroutine
    ch := make(chan error, 1)
    go func() {
        ch <- f(c.Do(req))
    }()

    // Wait for completion or cancellation
    select {
    case <-ctx.Done():
        <-ch  // Wait for f to return
        return ctx.Err()
    case err := <-ch:
        return err
    }
}
```

### Routing Patterns

**Custom Router:**

```go
type Router struct {
    routes map[string]http.HandlerFunc
}

func (r *Router) Handle(pattern string, handler http.HandlerFunc) {
    r.routes[pattern] = handler
}

func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    if handler, ok := r.routes[req.URL.Path]; ok {
        handler(w, req)
    } else {
        http.NotFound(w, req)
    }
}
```

**RESTful API Structure:**

```go
// GET /api/users
func listUsers(w http.ResponseWriter, r *http.Request) { /* ... */ }

// GET /api/users/:id
func getUser(w http.ResponseWriter, r *http.Request) { /* ... */ }

// POST /api/users
func createUser(w http.ResponseWriter, r *http.Request) { /* ... */ }

// PUT /api/users/:id
func updateUser(w http.ResponseWriter, r *http.Request) { /* ... */ }

// DELETE /api/users/:id
func deleteUser(w http.ResponseWriter, r *http.Request) { /* ... */ }
```

### WebSocket Server Example

**Basic WebSocket Server:**

```go
import (
    "github.com/gorilla/websocket"
    "log"
    "net/http"
)

// allowedOrigins: in production, use your app's origins e.g. []string{"https://myapp.com"}
func newUpgrader(allowedOrigins []string) websocket.Upgrader {
    originSet := make(map[string]bool)
    for _, o := range allowedOrigins {
        originSet[o] = true
    }
    return websocket.Upgrader{
        CheckOrigin: func(r *http.Request) bool {
            origin := r.Header.Get("Origin")
            if origin == "" {
                return true
            }
            return originSet["*"] || originSet[origin]
        },
    }
}

var upgrader = newUpgrader([]string{"*"}) // Replace with specific origins in production

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
}

type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte
}

func (h *Hub) run() {
    for {
        select {
        case client := <-h.register:
            h.clients[client] = true
        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}

func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()
    
    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            break
        }
        c.hub.broadcast <- message
    }
}

func (c *Client) writePump() {
    defer c.conn.Close()
    
    for message := range c.send {
        err := c.conn.WriteMessage(websocket.TextMessage, message)
        if err != nil {
            break
        }
    }
}

func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println(err)
        return
    }
    
    client := &Client{hub: hub, conn: conn, send: make(chan []byte, 256)}
    client.hub.register <- client
    
    go client.writePump()
    go client.readPump()
}
```

## Concurrency Patterns

### 1. Pipeline Pattern

Pipelines process data through multiple stages connected by channels.

**Generator Stage:**

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}
```

**Processing Stage:**

```go
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}
```

**Pipeline Usage:**

```go
func main() {
    // Set up pipeline
    c := gen(2, 3)
    out := sq(c)

    // Consume output
    for n := range out {
        fmt.Println(n)  // 4 then 9
    }
}
```

**Buffered Generator (No Goroutine Needed):**

```go
func gen(nums ...int) <-chan int {
    out := make(chan int, len(nums))
    for _, n := range nums {
        out <- n
    }
    close(out)
    return out
}
```

### 2. Fan-Out/Fan-In Pattern

Distribute work across multiple workers and merge results.

**Fan-Out: Multiple Workers:**

```go
func main() {
    in := gen(2, 3, 4, 5)

    // Fan out: distribute work across two goroutines
    c1 := sq(in)
    c2 := sq(in)

    // Fan in: merge results
    for n := range merge(c1, c2) {
        fmt.Println(n)
    }
}
```

**Merge Function (Fan-In):**

```go
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Start output goroutine for each input channel
    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        wg.Done()
    }

    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    // Close out once all outputs are done
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

### 3. Explicit Cancellation Pattern

**Cancellation with Done Channel:**

```go
func gen(done <-chan struct{}, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-done:
                return
            }
        }
    }()
    return out
}

func sq(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-done:
                return
            }
        }
    }()
    return out
}
```

**Broadcasting Cancellation:**

```go
func main() {
    done := make(chan struct{})
    defer close(done)  // Broadcast to all goroutines

    in := gen(done, 2, 3, 4)
    c1 := sq(done, in)
    c2 := sq(done, in)

    // Process subset of results
    out := merge(done, c1, c2)
    fmt.Println(<-out)

    // done closed on return, canceling all pipeline stages
}
```

**Merge with Cancellation:**

```go
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out <- n:
            case <-done:
                return
            }
        }
    }

    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

### 4. Worker Pool Pattern

**Fixed Number of Workers:**

```go
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // Start handlers
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // Wait to exit
}
```

**Semaphore Pattern:**

```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1        // Acquire
    process(r)
    <-sem           // Release
}

func Serve(queue chan *Request) {
    for req := range queue {
        go handle(req)
    }
}
```

**Limiting Goroutine Creation:**

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1  // Acquire before creating goroutine
        go func() {
            process(req)
            <-sem  // Release
        }()
    }
}
```

### 5. Query Racing Pattern

Query multiple sources and return first result:

```go
func Query(conns []Conn, query string) Result {
    ch := make(chan Result, len(conns)) // Buffered to prevent goroutine leaks
    for _, conn := range conns {
        go func(c Conn) {
            select {
            case ch <- c.DoQuery(query):
            default:
            }
        }(conn)
    }
    return <-ch
}
```

### 6. Parallel Processing Example

**Serial MD5 Calculation:**

```go
import (
    "crypto/md5"
    "os"
    "path/filepath"
)

func MD5All(root string) (map[string][md5.Size]byte, error) {
    m := make(map[string][md5.Size]byte)
    err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        if !info.Mode().IsRegular() {
            return nil
        }
        data, err := os.ReadFile(path)
        if err != nil {
            return err
        }
        m[path] = md5.Sum(data)
        return nil
    })
    return m, err
}
```

**Parallel MD5 with Pipeline:**

```go
import (
    "crypto/md5"
    "errors"
    "os"
    "path/filepath"
)

type result struct {
    path string
    sum  [md5.Size]byte
    err  error
}

func sumFiles(done <-chan struct{}, root string) (<-chan result, <-chan error) {
    c := make(chan result)
    errc := make(chan error, 1)

    go func() {
        defer close(c)
        err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
            if err != nil {
                return err
            }
            if !info.Mode().IsRegular() {
                return nil
            }

            // Start goroutine for each file (capture path to avoid closure bug)
            filePath := path
            go func() {
                data, err := os.ReadFile(filePath)
                select {
                case c <- result{filePath, md5.Sum(data), err}:
                case <-done:
                }
            }()

            // Check for early cancellation
            select {
            case <-done:
                return errors.New("walk canceled")
            default:
                return nil
            }
        })

        select {
        case errc <- err:
        case <-done:
        }
    }()
    return c, errc
}

func MD5All(root string) (map[string][md5.Size]byte, error) {
    done := make(chan struct{})
    defer close(done)

    c, errc := sumFiles(done, root)

    m := make(map[string][md5.Size]byte)
    for r := range c {
        if r.err != nil {
            return nil, r.err
        }
        m[r.path] = r.sum
    }

    if err := <-errc; err != nil {
        return nil, err
    }
    return m, nil
}
```

### 7. Leaky Buffer Pattern

Efficient buffer reuse:

```go
var freeList = make(chan *Buffer, 100)

func server() {
    for {
        b := <-serverChan  // Wait for work
        process(b)

        // Try to reuse buffer
        select {
        case freeList <- b:
            // Buffer on free list
        default:
            // Free list full, GC will reclaim
        }
    }
}
```

## Database Integration

### Connection Management

**Database Connection Pool:**

```go
import "database/sql"

func initDB(dataSourceName string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dataSourceName)
    if err != nil {
        return nil, err
    }

    // Configure connection pool
    db.SetMaxOpenConns(25)                 // Maximum number of open connections
    db.SetMaxIdleConns(5)                  // Maximum number of idle connections
    db.SetConnMaxLifetime(5 * time.Minute) // Maximum lifetime of a connection
    db.SetConnMaxIdleTime(10 * time.Minute) // Maximum idle time

    // Verify connection
    if err := db.Ping(); err != nil {
        return nil, err
    }

    return db, nil
}
```

**Connection with Retry Logic:**

```go
func connectWithRetry(dsn string, maxRetries int) (*sql.DB, error) {
    var db *sql.DB
    var err error
    
    for i := 0; i < maxRetries; i++ {
        db, err = sql.Open("postgres", dsn)
        if err == nil {
            // Test the connection
            ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            err = db.PingContext(ctx)
            cancel()
            
            if err == nil {
                return db, nil
            }
        }
        
        // Exponential backoff
        waitTime := time.Second * time.Duration(math.Pow(2, float64(i)))
        log.Printf("Failed to connect to database (attempt %d/%d): %v. Retrying in %v...", 
            i+1, maxRetries, err, waitTime)
        time.Sleep(waitTime)
    }
    
    return nil, fmt.Errorf("failed to connect after %d retries: %w", maxRetries, err)
}
```

### Query Patterns

**Single Row Query:**

```go
func getUser(ctx context.Context, db *sql.DB, userID int) (*User, error) {
    user := &User{}
    err := db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = $1",
        userID,
    ).Scan(&user.ID, &user.Name, &user.Email)

    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("user not found")
    }
    if err != nil {
        return nil, fmt.Errorf("query user: %w", err)
    }

    return user, nil
}
```

**Multiple Row Query:**

```go
func listUsers(ctx context.Context, db *sql.DB) ([]*User, error) {
    rows, err := db.QueryContext(ctx, "SELECT id, name, email FROM users")
    if err != nil {
        return nil, fmt.Errorf("query users: %w", err)
    }
    defer rows.Close()

    var users []*User
    for rows.Next() {
        user := &User{}
        if err := rows.Scan(&user.ID, &user.Name, &user.Email); err != nil {
            return nil, fmt.Errorf("scan user: %w", err)
        }
        users = append(users, user)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("iterate users: %w", err)
    }

    return users, nil
}
```

**Insert/Update with Context:**

```go
func createUser(ctx context.Context, db *sql.DB, user *User) error {
    query := "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id"
    err := db.QueryRowContext(ctx, query, user.Name, user.Email).Scan(&user.ID)
    if err != nil {
        return fmt.Errorf("create user: %w", err)
    }
    return nil
}
```

### SQL Injection Prevention

**CRITICAL: Always use parameterized queries**

```go
// ❌ DANGEROUS - SQL Injection vulnerability
func getUserByNameUnsafe(db *sql.DB, name string) (*User, error) {
    query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", name)
    // Attacker could pass: "admin' OR '1'='1" to bypass authentication
    return queryUser(db, query)
}

// ✅ SAFE - Use parameterized queries
func getUserByNameSafe(ctx context.Context, db *sql.DB, name string) (*User, error) {
    query := "SELECT id, name, email FROM users WHERE name = $1"
    user := &User{}
    err := db.QueryRowContext(ctx, query, name).Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        return nil, err
    }
    return user, nil
}

// ✅ SAFE - Dynamic IN clause with proper parameters
func getUsersByIDs(ctx context.Context, db *sql.DB, ids []int) ([]*User, error) {
    if len(ids) == 0 {
        return []*User{}, nil
    }
    
    // Build placeholders: $1, $2, $3, etc.
    placeholders := make([]string, len(ids))
    args := make([]interface{}, len(ids))
    for i, id := range ids {
        placeholders[i] = fmt.Sprintf("$%d", i+1)
        args[i] = id
    }
    
    query := fmt.Sprintf("SELECT id, name, email FROM users WHERE id IN (%s)",
        strings.Join(placeholders, ","))
    
    rows, err := db.QueryContext(ctx, query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var users []*User
    for rows.Next() {
        user := &User{}
        if err := rows.Scan(&user.ID, &user.Name, &user.Email); err != nil {
            return nil, err
        }
        users = append(users, user)
    }
    
    return users, rows.Err()
}
```

### Transaction Handling

```go
func transferFunds(ctx context.Context, db *sql.DB, from, to int, amount decimal.Decimal) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback()  // Rollback if not committed

    // Debit from account
    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        amount, from)
    if err != nil {
        return fmt.Errorf("debit account: %w", err)
    }

    // Credit to account
    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, to)
    if err != nil {
        return fmt.Errorf("credit account: %w", err)
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }

    return nil
}
```

**Transaction with Isolation Level:**

```go
func updateWithIsolation(ctx context.Context, db *sql.DB) error {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
        ReadOnly:  false,
    })
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Perform operations...
    
    return tx.Commit()
}
```

### Prepared Statements

```go
func insertUsers(ctx context.Context, db *sql.DB, users []*User) error {
    stmt, err := db.PrepareContext(ctx, "INSERT INTO users (name, email) VALUES ($1, $2)")
    if err != nil {
        return fmt.Errorf("prepare statement: %w", err)
    }
    defer stmt.Close()

    for _, user := range users {
        _, err := stmt.ExecContext(ctx, user.Name, user.Email)
        if err != nil {
            return fmt.Errorf("insert user %s: %w", user.Name, err)
        }
    }

    return nil
}
```

### Batch Inserts (More Efficient)

```go
func batchInsert(ctx context.Context, db *sql.DB, users []*User) error {
    if len(users) == 0 {
        return nil
    }
    
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback()
    
    // Build bulk insert query
    valueStrings := make([]string, 0, len(users))
    valueArgs := make([]interface{}, 0, len(users)*2)
    
    for i, user := range users {
        valueStrings = append(valueStrings, fmt.Sprintf("($%d, $%d)", i*2+1, i*2+2))
        valueArgs = append(valueArgs, user.Name, user.Email)
    }
    
    query := fmt.Sprintf("INSERT INTO users (name, email) VALUES %s",
        strings.Join(valueStrings, ","))
    
    _, err = tx.ExecContext(ctx, query, valueArgs...)
    if err != nil {
        return fmt.Errorf("batch insert: %w", err)
    }
    
    return tx.Commit()
}
```

### Alternative Database Libraries

**sqlx - Extension of database/sql:**

```go
import "github.com/jmoiron/sqlx"

type User struct {
    ID    int    `db:"id"`
    Name  string `db:"name"`
    Email string `db:"email"`
}

func getUser(ctx context.Context, db *sqlx.DB, userID int) (*User, error) {
    user := &User{}
    err := db.GetContext(ctx, user, "SELECT * FROM users WHERE id = $1", userID)
    return user, err
}

func listUsers(ctx context.Context, db *sqlx.DB) ([]*User, error) {
    users := []*User{}
    err := db.SelectContext(ctx, &users, "SELECT * FROM users")
    return users, err
}
```

**pgx - High-performance PostgreSQL driver:**

```go
import (
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

func initPgxPool(ctx context.Context, connString string) (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(connString)
    if err != nil {
        return nil, err
    }
    
    config.MaxConns = 25
    config.MinConns = 5
    
    pool, err := pgxpool.NewWithConfig(ctx, config)
    if err != nil {
        return nil, err
    }
    
    return pool, nil
}

func queryUsers(ctx context.Context, pool *pgxpool.Pool) ([]*User, error) {
    rows, err := pool.Query(ctx, "SELECT id, name, email FROM users")
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    users, err := pgx.CollectRows(rows, pgx.RowToStructByName[User])
    return users, err
}
```

## Error Handling

### Custom Error Types

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// Usage
func validateUser(user *User) error {
    if user.Email == "" {
        return &ValidationError{Field: "email", Message: "required"}
    }
    if !strings.Contains(user.Email, "@") {
        return &ValidationError{Field: "email", Message: "invalid format"}
    }
    return nil
}
```

### Error Wrapping

```go
import (
    "errors"
    "fmt"
)

func processData(data []byte) error {
    err := validateData(data)
    if err != nil {
        return fmt.Errorf("process data: %w", err)
    }
    return nil
}

// Unwrapping and checking
func handleError(err error) {
    // Check for specific error type
    if errors.Is(err, ErrValidation) {
        // Handle validation error
    }
    
    // Extract error type
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        log.Printf("Validation failed for field: %s", validationErr.Field)
    }
}
```

### Sentinel Errors

```go
var (
    ErrNotFound      = errors.New("not found")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrInvalidInput  = errors.New("invalid input")
    ErrAlreadyExists = errors.New("already exists")
)

// Usage
func getUser(id int) (*User, error) {
    user, err := db.QueryUser(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNotFound
        }
        return nil, fmt.Errorf("query user: %w", err)
    }
    return user, nil
}

// Handler
func handleGetUser(w http.ResponseWriter, r *http.Request) {
    user, err := getUser(id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            http.Error(w, "User not found", http.StatusNotFound)
            return
        }
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    respondJSON(w, http.StatusOK, user)
}
```

## Testing

### Unit Tests

```go
func TestGetUser(t *testing.T) {
    db := setupTestDB(t)
    defer db.Close()

    user, err := getUser(context.Background(), db, 1)
    if err != nil {
        t.Fatalf("getUser failed: %v", err)
    }

    if user.Name != "John Doe" {
        t.Errorf("expected name John Doe, got %s", user.Name)
    }
}
```

### Table-Driven Tests

```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        {"valid email", "user@example.com", false},
        {"missing @", "userexample.com", true},
        {"empty string", "", true},
        {"missing domain", "user@", true},
        {"multiple @", "user@@example.com", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := validateEmail(tt.email)
            if (err != nil) != tt.wantErr {
                t.Errorf("validateEmail(%q) error = %v, wantErr %v",
                    tt.email, err, tt.wantErr)
            }
        })
    }
}
```

### Test Helpers

```go
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("Failed to open test database: %v", err)
    }
    
    // Run migrations
    if err := runMigrations(db); err != nil {
        t.Fatalf("Failed to run migrations: %v", err)
    }
    
    return db
}

func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("Unexpected error: %v", err)
    }
}

func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}
```

### Benchmarks

```go
func BenchmarkConcurrentMap(b *testing.B) {
    m := make(map[string]int)
    var mu sync.Mutex

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            mu.Lock()
            m["key"]++
            mu.Unlock()
        }
    })
}

func BenchmarkUserValidation(b *testing.B) {
    user := &User{
        Name:  "John Doe",
        Email: "john@example.com",
    }
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = validateUser(user)
    }
}
```

### HTTP Handler Testing

```go
import (
    "encoding/json"
    "io"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/api/users", nil)
    w := httptest.NewRecorder()

    handler(w, req)

    resp := w.Result()
    if resp.StatusCode != http.StatusOK {
        t.Errorf("expected status 200, got %d", resp.StatusCode)
    }

    body, _ := io.ReadAll(resp.Body)
    var users []User
    if err := json.Unmarshal(body, &users); err != nil {
        t.Errorf("Failed to parse response: %v", err)
    }
    
    if len(users) == 0 {
        t.Error("Expected users in response")
    }
}
```

### Mocking Interfaces

```go
type UserRepository interface {
    GetUser(ctx context.Context, id int) (*User, error)
    CreateUser(ctx context.Context, user *User) error
}

type MockUserRepository struct {
    GetUserFunc    func(ctx context.Context, id int) (*User, error)
    CreateUserFunc func(ctx context.Context, user *User) error
}

func (m *MockUserRepository) GetUser(ctx context.Context, id int) (*User, error) {
    if m.GetUserFunc != nil {
        return m.GetUserFunc(ctx, id)
    }
    return nil, errors.New("not implemented")
}

func (m *MockUserRepository) CreateUser(ctx context.Context, user *User) error {
    if m.CreateUserFunc != nil {
        return m.CreateUserFunc(ctx, user)
    }
    return errors.New("not implemented")
}

// Test with mock
func TestUserService(t *testing.T) {
    mockRepo := &MockUserRepository{
        GetUserFunc: func(ctx context.Context, id int) (*User, error) {
            return &User{ID: id, Name: "Test User"}, nil
        },
    }
    
    service := NewUserService(mockRepo)
    user, err := service.GetUser(context.Background(), 1)
    
    if err != nil {
        t.Fatalf("Unexpected error: %v", err)
    }
    if user.Name != "Test User" {
        t.Errorf("Expected 'Test User', got %s", user.Name)
    }
}
```

## Observability and Monitoring

### Structured Logging

```go
import "log/slog"

func setupLogger() *slog.Logger {
    return slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
}

func handler(w http.ResponseWriter, r *http.Request) {
    logger := slog.With(
        "method", r.Method,
        "path", r.URL.Path,
        "remote", r.RemoteAddr,
        "request_id", generateRequestID(),
    )

    start := time.Now()
    logger.Info("handling request")

    // Process request

    duration := time.Since(start)
    logger.Info("request completed",
        "status", 200,
        "duration_ms", duration.Milliseconds(),
    )
}
```

### Prometheus Metrics

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request latencies in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
    
    activeConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )
)

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Wrap ResponseWriter to capture status code
        ww := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        
        activeConnections.Inc()
        defer activeConnections.Dec()
        
        next.ServeHTTP(ww, r)
        
        duration := time.Since(start).Seconds()
        httpRequestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, 
            strconv.Itoa(ww.statusCode)).Inc()
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

// Expose metrics endpoint
func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.Handle("/api/", metricsMiddleware(apiHandler))
    http.ListenAndServe(":8080", nil)
}
```

### Distributed Tracing

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/trace"
)

func handleRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    tracer := otel.Tracer("api-service")
    
    ctx, span := tracer.Start(ctx, "handleRequest")
    defer span.End()
    
    span.SetAttributes(
        attribute.String("http.method", r.Method),
        attribute.String("http.path", r.URL.Path),
    )
    
    // Call downstream services with context
    result, err := fetchData(ctx, userID)
    if err != nil {
        span.RecordError(err)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    respondJSON(w, http.StatusOK, result)
}

func fetchData(ctx context.Context, userID string) (*Data, error) {
    tracer := otel.Tracer("api-service")
    ctx, span := tracer.Start(ctx, "fetchData")
    defer span.End()
    
    span.SetAttributes(attribute.String("user.id", userID))
    
    // Database or HTTP call with traced context
    return queryDatabase(ctx, userID)
}
```

## Production Patterns

### Graceful Shutdown

```go
func main() {
    srv := &http.Server{
        Addr:    ":8080",
        Handler: router,
        // Security timeouts
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Start server in goroutine
    go func() {
        log.Println("Starting server on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server failed: %s\n", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("Shutting down server...")

    // Graceful shutdown with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    log.Println("Server exited")
}
```

### Configuration Management

```go
import "github.com/caarlos0/env/v10"

type Config struct {
    ServerPort     int           `env:"PORT" envDefault:"8080"`
    DBHost         string        `env:"DB_HOST" envDefault:"localhost"`
    DBPort         int           `env:"DB_PORT" envDefault:"5432"`
    DBUser         string        `env:"DB_USER" envDefault:"postgres"`
    DBPassword     string        `env:"DB_PASSWORD,required"`
    DBName         string        `env:"DB_NAME" envDefault:"myapp"`
    LogLevel       string        `env:"LOG_LEVEL" envDefault:"info"`
    Timeout        time.Duration `env:"TIMEOUT" envDefault:"30s"`
    MaxConnections int           `env:"MAX_CONNECTIONS" envDefault:"25"`
    Environment    string        `env:"ENVIRONMENT" envDefault:"development"`
}

func loadConfig() (*Config, error) {
    cfg := &Config{}
    if err := env.Parse(cfg); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }
    return cfg, nil
}

// Build DSN from config
func (c *Config) DatabaseDSN() string {
    return fmt.Sprintf("host=%s port=%d user=%s password=%s dbname=%s sslmode=disable",
        c.DBHost, c.DBPort, c.DBUser, c.DBPassword, c.DBName)
}
```

### Health Checks

```go
func healthHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
        defer cancel()
        
        health := map[string]string{
            "status": "healthy",
        }
        
        // Check database
        if err := db.PingContext(ctx); err != nil {
            health["status"] = "unhealthy"
            health["database"] = err.Error()
            respondJSON(w, http.StatusServiceUnavailable, health)
            return
        }
        
        // Check other dependencies (Redis, external APIs, etc.)
        
        respondJSON(w, http.StatusOK, health)
    }
}

// Liveness probe - is the app running?
func livenessHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}

// Readiness probe - can the app handle requests?
func readinessHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if err := db.Ping(); err != nil {
            http.Error(w, "Not ready", http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("Ready"))
    }
}
```

### Rate Limiting

```go
import "golang.org/x/time/rate"

func rateLimitMiddleware(limiter *rate.Limiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Per-IP rate limiting
type IPRateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewIPRateLimiter(r rate.Limit, b int) *IPRateLimiter {
    return &IPRateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    b,
    }
}

func (i *IPRateLimiter) GetLimiter(ip string) *rate.Limiter {
    i.mu.Lock()
    defer i.mu.Unlock()
    
    limiter, exists := i.limiters[ip]
    if !exists {
        limiter = rate.NewLimiter(i.rate, i.burst)
        i.limiters[ip] = limiter
    }
    
    return limiter
}

// Usage
limiter := rate.NewLimiter(rate.Limit(10), 20)  // 10 req/sec, burst 20
handler := rateLimitMiddleware(limiter)(apiHandler)
```

### Panic Recovery

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                // Log stack trace
                log.Printf("panic: %v\n%s", err, debug.Stack())
                
                // Send error response
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}

// Safer goroutine execution
func safeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("goroutine panic recovered: %v\n%s", r, debug.Stack())
            }
        }()
        fn()
    }()
}

// Panic recovery that converts to error
func safelyDo(work func() error) (err error) {
    defer func() {
        if r := recover(); r != nil {
            // Log stack trace
            log.Printf("panic recovered: %v\n%s", r, debug.Stack())
            // Convert panic to error
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    return work()
}
```

## Microservices Patterns

### Service Structure with Dependency Injection

```go
// Define interfaces for dependencies
type UserRepository interface {
    GetUser(ctx context.Context, id string) (*User, error)
    CreateUser(ctx context.Context, user *User) error
}

type CacheService interface {
    Get(ctx context.Context, key string) (interface{}, error)
    Set(ctx context.Context, key string, value interface{}, ttl time.Duration) error
}

// Service with injected dependencies
type UserService struct {
    repo   UserRepository
    cache  CacheService
    logger *slog.Logger
}

func NewUserService(repo UserRepository, cache CacheService, logger *slog.Logger) *UserService {
    return &UserService{
        repo:   repo,
        cache:  cache,
        logger: logger,
    }
}

func (s *UserService) GetUser(ctx context.Context, userID string) (*User, error) {
    // Check cache first
    cacheKey := fmt.Sprintf("user:%s", userID)
    if cached, err := s.cache.Get(ctx, cacheKey); err == nil {
        s.logger.Info("cache hit", "user_id", userID)
        return cached.(*User), nil
    }

    // Query database
    user, err := s.repo.GetUser(ctx, userID)
    if err != nil {
        return nil, fmt.Errorf("get user from db: %w", err)
    }

    // Update cache asynchronously
    go func() {
        ctx := context.Background()
        if err := s.cache.Set(ctx, cacheKey, user, 5*time.Minute); err != nil {
            s.logger.Error("failed to cache user", "error", err)
        }
    }()

    return user, nil
}

// Wire everything together
func setupServices(db *sql.DB, redis *redis.Client, logger *slog.Logger) *UserService {
    repo := NewPostgresUserRepository(db)
    cache := NewRedisCacheService(redis)
    return NewUserService(repo, cache, logger)
}
```

### gRPC Service

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

type server struct {
    pb.UnimplementedUserServiceServer
    userService *UserService
}

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := s.userService.GetUser(ctx, req.GetId())
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            return nil, status.Errorf(codes.NotFound, "user not found")
        }
        return nil, status.Errorf(codes.Internal, "internal error")
    }

    return &pb.User{
        Id:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}

func startGRPCServer(userService *UserService) error {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        return err
    }

    grpcServer := grpc.NewServer(
        grpc.UnaryInterceptor(loggingInterceptor),
    )
    
    pb.RegisterUserServiceServer(grpcServer, &server{userService: userService})
    
    return grpcServer.Serve(lis)
}

// gRPC interceptor (middleware)
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("gRPC %s duration=%v error=%v", info.FullMethod, time.Since(start), err)
    return resp, err
}
```

### Service Discovery

```go
type ServiceRegistry struct {
    services map[string][]string
    mu       sync.RWMutex
}

func NewServiceRegistry() *ServiceRegistry {
    return &ServiceRegistry{
        services: make(map[string][]string),
    }
}

func (r *ServiceRegistry) Register(name, addr string) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.services[name] = append(r.services[name], addr)
}

func (r *ServiceRegistry) Deregister(name, addr string) {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    addrs := r.services[name]
    for i, a := range addrs {
        if a == addr {
            r.services[name] = append(addrs[:i], addrs[i+1:]...)
            break
        }
    }
}

func (r *ServiceRegistry) Discover(name string) (string, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    addrs := r.services[name]
    if len(addrs) == 0 {
        return "", fmt.Errorf("service %s not found", name)
    }

    // Simple round-robin (in production, use proper load balancing)
    return addrs[rand.Intn(len(addrs))], nil
}

// Health checking
func (r *ServiceRegistry) StartHealthChecks(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            r.checkHealth()
        }
    }
}

func (r *ServiceRegistry) checkHealth() {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    for name, addrs := range r.services {
        for i, addr := range addrs {
            if !isHealthy(addr) {
                log.Printf("Removing unhealthy service: %s at %s", name, addr)
                r.services[name] = append(addrs[:i], addrs[i+1:]...)
            }
        }
    }
}
```

### Circuit Breaker

```go
type CircuitBreaker struct {
    maxFailures int
    timeout     time.Duration
    failures    int
    lastFailure time.Time
    state       string  // closed, open, half-open
    mu          sync.Mutex
}

func NewCircuitBreaker(maxFailures int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        maxFailures: maxFailures,
        timeout:     timeout,
        state:       "closed",
    }
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()

    if cb.state == "open" {
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = "half-open"
            cb.failures = 0
        } else {
            cb.mu.Unlock()
            return errors.New("circuit breaker open")
        }
    }

    cb.mu.Unlock()

    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.maxFailures {
            cb.state = "open"
            log.Printf("Circuit breaker opened after %d failures", cb.failures)
        }
        return err
    }

    // Success - reset
    if cb.state == "half-open" {
        cb.state = "closed"
        log.Println("Circuit breaker closed")
    }
    cb.failures = 0
    return nil
}

// Usage
func callExternalService(cb *CircuitBreaker) error {
    return cb.Call(func() error {
        resp, err := http.Get("https://api.example.com/data")
        if err != nil {
            return err
        }
        defer resp.Body.Close()
        
        if resp.StatusCode >= 500 {
            return fmt.Errorf("server error: %d", resp.StatusCode)
        }
        
        return nil
    })
}
```

## Deployment and Containerization

### Docker Multi-Stage Build

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main ./cmd/server

# Final stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copy binary from builder
COPY --from=builder /app/main .

# Copy migrations (if needed)
COPY --from=builder /app/migrations ./migrations

EXPOSE 8080

CMD ["./main"]
```

### Embed Static Files

```go
import "embed"

//go:embed static/*
var staticFiles embed.FS

//go:embed migrations/*.sql
var migrationFiles embed.FS

func main() {
    // Serve embedded static files
    http.Handle("/static/", http.FileServer(http.FS(staticFiles)))
    
    // Read embedded migration files
    migration, _ := migrationFiles.ReadFile("migrations/001_init.sql")
}
```

## Project Structure

Reference: [golang-standards/project-layout](https://github.com/golang-standards/project-layout)

```
project/
├── cmd/                     # Main applications for this project
│   └── server/
│       └── main.go          # Application entry point
├── internal/                # Private application and library code
│   ├── api/                 # HTTP handlers
│   │   ├── handler.go
│   │   └── middleware.go
│   ├── service/             # Business logic
│   │   └── user.go
│   ├── repository/          # Data access layer
│   │   └── user.go
│   └── model/               # Data models
│       └── user.go
├── pkg/                     # Public library code (use sparingly)
│   └── utils/
├── api/                     # API definitions (OpenAPI/Swagger, protobuf)
│   └── openapi.yaml
├── migrations/              # Database migrations
│   ├── 001_init.sql
│   └── 002_add_users.sql
├── scripts/                 # Build and install scripts
│   └── build.sh
├── deployments/             # IaaS, PaaS, orchestration configs
│   ├── docker-compose.yml
│   └── kubernetes/
├── test/                    # Integration tests
│   └── integration_test.go
├── docs/                    # Documentation
├── go.mod
├── go.sum
├── Dockerfile
├── Makefile
└── README.md
```

## Best Practices

### 1. Goroutine Management

- Always consider goroutine lifecycle and cleanup
- Use contexts for cancellation propagation
- Avoid goroutine leaks by ensuring all goroutines can exit
- Be cautious with closures in loops - pass values explicitly

**Anti-pattern:**
```go
for _, v := range values {
    go func() {
        fmt.Println(v)  // All goroutines share same v
    }()
}
```

**Correct:**
```go
for _, v := range values {
    go func(val string) {
        fmt.Println(val)  // Each goroutine gets its own copy
    }(v)
}
```

### 2. Channel Best Practices

- Close channels from sender, not receiver
- Use buffered channels to prevent goroutine leaks when appropriate
- Consider using `select` with `default` for non-blocking operations
- Remember: sending on closed channel panics, receiving returns zero value

### 3. Error Handling

- Return errors, don't panic (except for truly exceptional cases)
- Wrap errors with context using `fmt.Errorf("context: %w", err)`
- Use custom error types for programmatic handling
- Log errors with sufficient context
- Don't ignore errors: `_ = someFunc()` should be rare and justified

### 4. Performance

- Use `sync.Pool` for frequently allocated objects
- Profile before optimizing: `go test -bench . -cpuprofile=cpu.prof`
- Consider `sync.Map` for concurrent map access patterns
- Use buffered channels for known capacity
- Avoid unnecessary allocations in hot paths
- Reuse HTTP clients (they maintain connection pools)

### 5. Security

- Validate all inputs
- Use prepared statements for SQL queries (prevents SQL injection)
- Implement rate limiting
- Use HTTPS in production
- Sanitize error messages sent to clients
- Use context timeouts to prevent resource exhaustion
- Implement proper authentication and authorization
- Never log sensitive data (passwords, tokens, PII)

### 6. Testing

- Write table-driven tests
- Use `t.Helper()` for test helper functions
- Mock external dependencies with interfaces
- Use `httptest` for HTTP handler testing
- Write benchmarks for performance-critical code
- Aim for >80% test coverage on business logic
- Use subtests with `t.Run()` for better organization

### 7. Code Organization

- Keep packages focused and cohesive
- Use interfaces to define contracts
- Avoid circular dependencies
- Put interfaces in the package that uses them, not implements them
- Use internal/ directory for non-public code
- Group by feature/domain, not by layer (in most cases)

## Common Pitfalls

### 1. Race Conditions

**Problem:**
```go
var service map[string]net.Addr

func RegisterService(name string, addr net.Addr) {
    service[name] = addr  // RACE CONDITION
}

func LookupService(name string) net.Addr {
    return service[name]  // RACE CONDITION
}
```

**Solution:**
```go
var (
    service   map[string]net.Addr
    serviceMu sync.RWMutex
)

func RegisterService(name string, addr net.Addr) {
    serviceMu.Lock()
    defer serviceMu.Unlock()
    service[name] = addr
}

func LookupService(name string) net.Addr {
    serviceMu.RLock()
    defer serviceMu.RUnlock()
    return service[name]
}
```

**Detection: Run tests with race detector**
```bash
go test -race ./...
```

### 2. Goroutine Leaks

**Problem:**
```go
func process() {
    ch := make(chan int)
    go func() {
        ch <- expensive()  // Blocks forever if no receiver
    }()
    // Returns without reading from ch
}
```

**Solution:**
```go
func process() {
    ch := make(chan int, 1)  // Buffered channel
    go func() {
        ch <- expensive()  // Won't block
    }()
    // Goroutine can complete even if we don't read
}

// Or use context for cancellation
func processBetter(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case ch <- expensive():
        case <-ctx.Done():
            return
        }
    }()
}
```

### 3. Not Closing Channels

Receivers need to know when no more values are coming:

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)  // IMPORTANT: close when done
    }()
    return out
}
```

### 4. Blocking on Unbuffered Channels

```go
// This will deadlock
ch := make(chan int)
ch <- 1  // Blocks forever - no receiver
v := <-ch
```

Use buffered channels or separate goroutines.

### 5. Not Checking Context Cancellation

```go
// Bad: ignores cancellation
func work(ctx context.Context) error {
    for i := 0; i < 1000000; i++ {
        doExpensiveOperation(i)
    }
    return nil
}

// Good: respects cancellation
func work(ctx context.Context) error {
    for i := 0; i < 1000000; i++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            doExpensiveOperation(i)
        }
    }
    return nil
}
```

### 6. Improper Defer Usage

```go
// Bad: defer in loop can cause memory issues
func processFiles(files []string) error {
    for _, file := range files {
        f, err := os.Open(file)
        if err != nil {
            return err
        }
        defer f.Close()  // Won't close until function returns!
        // process file
    }
    return nil
}

// Good: close in loop
func processFiles(files []string) error {
    for _, file := range files {
        if err := processFile(file); err != nil {
            return err
        }
    }
    return nil
}

func processFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close()
    // process file
    return nil
}
```

## Resources and References

### Official Documentation
- Go Documentation: https://go.dev/doc/
- Effective Go: https://go.dev/doc/effective_go
- Go Blog: https://go.dev/blog/
- Go by Example: https://gobyexample.com/
- Go Code Review Comments: https://github.com/golang/go/wiki/CodeReviewComments

### Concurrency Resources
- Go Concurrency Patterns: https://go.dev/blog/pipelines
- Context Package: https://go.dev/blog/context
- Share Memory By Communicating: https://go.dev/blog/codelab-share
- Advanced Go Concurrency: https://go.dev/blog/io2013-talk-concurrency

### Standard Library
- net/http: https://pkg.go.dev/net/http
- database/sql: https://pkg.go.dev/database/sql
- context: https://pkg.go.dev/context
- sync: https://pkg.go.dev/sync
- log/slog: https://pkg.go.dev/log/slog

### Tools
- Race Detector: `go test -race`
- Profiler: `go tool pprof`
- Benchmarking: `go test -bench`
- Static Analysis: `go vet`, `staticcheck`
- Linting: `golangci-lint`

### Popular Libraries
- Chi Router: https://github.com/go-chi/chi
- GORM ORM: https://gorm.io
- sqlx: https://github.com/jmoiron/sqlx
- pgx PostgreSQL: https://github.com/jackc/pgx
- Gorilla WebSocket: https://github.com/gorilla/websocket
- Prometheus Client: https://github.com/prometheus/client_golang
- Viper Config: https://github.com/spf13/viper
- Testify: https://github.com/stretchr/testify

---

**Skill Version**: 2.0.0
**Last Updated**: February 2026
**Skill Category**: Backend Development, Systems Programming, Concurrent Programming
**Prerequisites**: Basic programming knowledge, understanding of HTTP, familiarity with command line
**Recommended Next Skills**: docker-deployment, kubernetes-orchestration, grpc-microservices, postgresql-optimization
