# Go Instructions

## General Principles
- **ALWAYS run `go fmt`** before committing code
- **ALWAYS handle errors** explicitly - never ignore them
- **ALWAYS use meaningful names** - favor clarity over brevity
- **ALWAYS prefer composition** over inheritance
- **NEVER use `panic`** for normal error handling
- **NEVER ignore the `error` return value**

---

# Code Style

## Naming Conventions
```go
// Packages: lowercase, single word preferred
package user
package httputil

// Exported (public): PascalCase
type UserService struct {}
func NewUserService() *UserService {}
const MaxRetryCount = 3

// Unexported (private): camelCase
type userRepository struct {}
func validateEmail(email string) error {}
var defaultTimeout = 30 * time.Second

// Interfaces: -er suffix for single method
type Reader interface {
    Read(p []byte) (n int, err error)
}

type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
}

// Acronyms: all caps or all lowercase
var httpClient *http.Client  // Not Http
type JSONParser struct {}    // Not JsonParser
userID := "123"              // Not userId
```

## Code Organization
```go
// File structure within a package
package user

// 1. Imports (grouped and sorted)
import (
    "context"
    "errors"
    "fmt"
    "time"

    "github.com/company/project/internal/database"
    "github.com/company/project/pkg/validation"

    "github.com/lib/pq"
)

// 2. Constants
const (
    maxNameLength = 100
    defaultTimeout = 30 * time.Second
)

// 3. Package-level variables (minimize these)
var (
    ErrNotFound = errors.New("user not found")
    ErrInvalidEmail = errors.New("invalid email")
)

// 4. Types
type User struct {
    ID        string
    Name      string
    Email     string
    CreatedAt time.Time
}

// 5. Constructor functions
func New(name, email string) (*User, error) {
    // ...
}

// 6. Methods
func (u *User) Validate() error {
    // ...
}
```

---

# Error Handling

## Error Patterns
```go
// GOOD: Explicit error handling
func GetUser(ctx context.Context, id string) (*User, error) {
    user, err := repo.FindByID(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNotFound
        }
        return nil, fmt.Errorf("finding user %s: %w", id, err)
    }
    return user, nil
}

// GOOD: Using errors.Is and errors.As
if errors.Is(err, ErrNotFound) {
    // Handle not found
}

var validationErr *ValidationError
if errors.As(err, &validationErr) {
    // Handle validation error
    log.Printf("validation failed: %v", validationErr.Fields)
}

// BAD: Ignoring errors
user, _ := repo.FindByID(ctx, id)  // Never ignore errors

// BAD: Using panic for expected errors
if err != nil {
    panic(err)  // Don't panic for normal errors
}
```

## Custom Errors
```go
// Sentinel errors for comparison
var (
    ErrNotFound      = errors.New("not found")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrInvalidInput  = errors.New("invalid input")
)

// Structured error with context
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed: %s - %s", e.Field, e.Message)
}

// Error wrapping
func (s *UserService) Create(ctx context.Context, req CreateRequest) (*User, error) {
    if err := req.Validate(); err != nil {
        return nil, fmt.Errorf("validating request: %w", err)
    }

    user, err := s.repo.Save(ctx, req.ToUser())
    if err != nil {
        return nil, fmt.Errorf("saving user: %w", err)
    }

    return user, nil
}
```

---

# Project Structure

## Standard Layout
```
project/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── user/
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── user.go
│   ├── order/
│   └── config/
├── pkg/
│   ├── validation/
│   └── httputil/
├── api/
│   └── openapi.yaml
├── migrations/
├── go.mod
├── go.sum
└── README.md
```

## Package Guidelines
```go
// internal/ - Private packages, not importable by other projects
// pkg/ - Public packages, can be imported by other projects
// cmd/ - Entry points (main packages)

// Keep packages focused and cohesive
// Avoid circular dependencies
// Prefer passing dependencies explicitly (no globals)
```

---

# Concurrency

## Goroutine Patterns
```go
// GOOD: Using errgroup for coordinated goroutines
func FetchAll(ctx context.Context, ids []string) ([]*User, error) {
    g, ctx := errgroup.WithContext(ctx)
    users := make([]*User, len(ids))

    for i, id := range ids {
        i, id := i, id  // Capture loop variables
        g.Go(func() error {
            user, err := FetchUser(ctx, id)
            if err != nil {
                return fmt.Errorf("fetching user %s: %w", id, err)
            }
            users[i] = user
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return users, nil
}

// GOOD: Channel with proper cleanup
func ProcessItems(ctx context.Context, items []Item) error {
    results := make(chan Result, len(items))
    errs := make(chan error, 1)

    go func() {
        defer close(results)
        for _, item := range items {
            select {
            case <-ctx.Done():
                return
            case results <- process(item):
            }
        }
    }()

    for result := range results {
        if err := handleResult(result); err != nil {
            return err
        }
    }
    return nil
}
```

## Context Usage
```go
// ALWAYS pass context as first parameter
func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    // Use context for cancellation
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }

    // Pass context to downstream calls
    return s.repo.FindByID(ctx, id)
}

// Set timeouts appropriately
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

---

# Testing

## Test Structure
```go
func TestUserService_Create(t *testing.T) {
    tests := []struct {
        name    string
        req     CreateRequest
        setup   func(*mockRepo)
        want    *User
        wantErr error
    }{
        {
            name: "valid request creates user",
            req: CreateRequest{
                Name:  "John",
                Email: "john@example.com",
            },
            setup: func(m *mockRepo) {
                m.On("Save", mock.Anything, mock.Anything).Return(nil)
            },
            want: &User{Name: "John", Email: "john@example.com"},
        },
        {
            name: "duplicate email returns error",
            req: CreateRequest{
                Name:  "John",
                Email: "existing@example.com",
            },
            setup: func(m *mockRepo) {
                m.On("Save", mock.Anything, mock.Anything).
                    Return(ErrDuplicateEmail)
            },
            wantErr: ErrDuplicateEmail,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := new(mockRepo)
            tt.setup(repo)

            svc := NewUserService(repo)
            got, err := svc.Create(context.Background(), tt.req)

            if tt.wantErr != nil {
                assert.ErrorIs(t, err, tt.wantErr)
                return
            }

            require.NoError(t, err)
            assert.Equal(t, tt.want.Name, got.Name)
            assert.Equal(t, tt.want.Email, got.Email)
        })
    }
}
```

## Benchmarks
```go
func BenchmarkProcessItems(b *testing.B) {
    items := generateTestItems(1000)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        ProcessItems(context.Background(), items)
    }
}

func BenchmarkProcessItems_Parallel(b *testing.B) {
    items := generateTestItems(1000)

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            ProcessItems(context.Background(), items)
        }
    })
}
```

---

# HTTP Handlers

## Handler Pattern
```go
// Handler with dependencies
type UserHandler struct {
    service *UserService
    logger  *slog.Logger
}

func NewUserHandler(service *UserService, logger *slog.Logger) *UserHandler {
    return &UserHandler{
        service: service,
        logger:  logger,
    }
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    id := chi.URLParam(r, "id")

    user, err := h.service.GetByID(ctx, id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            h.respondError(w, http.StatusNotFound, "user not found")
            return
        }
        h.logger.Error("failed to get user", "error", err, "id", id)
        h.respondError(w, http.StatusInternalServerError, "internal error")
        return
    }

    h.respondJSON(w, http.StatusOK, user)
}

func (h *UserHandler) respondJSON(w http.ResponseWriter, status int, data any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func (h *UserHandler) respondError(w http.ResponseWriter, status int, message string) {
    h.respondJSON(w, status, map[string]string{"error": message})
}
```

---

# Security

## Input Validation
```go
import "github.com/go-playground/validator/v10"

type CreateUserRequest struct {
    Name     string `json:"name" validate:"required,min=1,max=100"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}

var validate = validator.New()

func (r *CreateUserRequest) Validate() error {
    return validate.Struct(r)
}
```

## SQL Injection Prevention
```go
// GOOD: Using parameterized queries
func (r *userRepo) FindByEmail(ctx context.Context, email string) (*User, error) {
    query := `SELECT id, name, email FROM users WHERE email = $1`
    row := r.db.QueryRowContext(ctx, query, email)
    // ...
}

// GOOD: Using sqlx named queries
query := `SELECT * FROM users WHERE name = :name AND status = :status`
rows, err := db.NamedQueryContext(ctx, query, map[string]interface{}{
    "name":   name,
    "status": status,
})

// BAD: String concatenation (SQL injection vulnerable)
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email) // NEVER
```

---

# Anti-Patterns to Avoid

```go
// BAD: Ignoring errors
result, _ := doSomething()  // Never ignore errors

// GOOD: Handle errors
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doing something: %w", err)
}

// BAD: Naked return in long functions
func process() (result string, err error) {
    // ... many lines ...
    return  // Confusing - what's being returned?
}

// GOOD: Explicit returns
func process() (string, error) {
    // ... many lines ...
    return result, nil
}

// BAD: init() for complex initialization
func init() {
    db = connectToDatabase()  // Can panic, hard to test
}

// GOOD: Explicit initialization
func main() {
    db, err := connectToDatabase()
    if err != nil {
        log.Fatal(err)
    }
}

// BAD: Package-level mutable state
var cache = make(map[string]string)  // Concurrency issues

// GOOD: Encapsulate state
type Cache struct {
    mu    sync.RWMutex
    items map[string]string
}
```
