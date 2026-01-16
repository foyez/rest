# 4.3 Error Handling

## Consistent Error Format

**Bad - Inconsistent:**
```json
// Sometimes:
{ "error": "User not found" }

// Other times:
{ "message": "Invalid input", "code": 422 }

// Even worse:
"Error occurred"
```

**Good - Consistent:**
```json
{
  "error": "validation_failed",
  "message": "One or more fields are invalid",
  "details": {
    "email": "Email is required",
    "age": "Age must be positive"
  },
  "request_id": "req-abc123",
  "timestamp": "2025-01-16T10:30:00Z"
}
```

---

## Error Response Structure

### Minimal Structure

```go
type ErrorResponse struct {
    Error   string `json:"error"`
    Message string `json:"message"`
}
```

```json
{
  "error": "not_found",
  "message": "User with ID 123 not found"
}
```

---

### Recommended Structure

```go
type ErrorResponse struct {
    Error     string            `json:"error"`
    Message   string            `json:"message"`
    Details   map[string]string `json:"details,omitempty"`
    RequestID string            `json:"request_id"`
    Timestamp time.Time         `json:"timestamp"`
}
```

```json
{
  "error": "validation_failed",
  "message": "Request validation failed",
  "details": {
    "email": "Invalid email format",
    "password": "Password must be at least 8 characters"
  },
  "request_id": "req-abc123",
  "timestamp": "2025-01-16T10:30:00Z"
}
```

---

### Extended Structure (With Stack Trace in Dev)

```go
type ErrorResponse struct {
    Error      string            `json:"error"`
    Message    string            `json:"message"`
    Details    map[string]string `json:"details,omitempty"`
    RequestID  string            `json:"request_id"`
    Timestamp  time.Time         `json:"timestamp"`
    Path       string            `json:"path,omitempty"`
    StackTrace string            `json:"stack_trace,omitempty"` // Dev only!
}
```

---

## Error Categories

### 1. Validation Errors (422)

```json
{
  "error": "validation_failed",
  "message": "Request validation failed",
  "details": {
    "email": "Email is required",
    "age": "Age must be between 18 and 120",
    "password": "Password must contain uppercase, lowercase, and number"
  }
}
```

**Implementation:**

```go
type ValidationError struct {
    Field   string
    Message string
}

func ValidateUser(user *User) []ValidationError {
    errors := []ValidationError{}
    
    if user.Email == "" {
        errors = append(errors, ValidationError{
            Field:   "email",
            Message: "Email is required",
        })
    }
    
    if user.Age < 18 || user.Age > 120 {
        errors = append(errors, ValidationError{
            Field:   "age",
            Message: "Age must be between 18 and 120",
        })
    }
    
    return errors
}

func CreateUser(w http.ResponseWriter, r *http.Request) {
    var user User
    json.NewDecoder(r.Body).Decode(&user)
    
    if validationErrors := ValidateUser(&user); len(validationErrors) > 0 {
        details := make(map[string]string)
        for _, err := range validationErrors {
            details[err.Field] = err.Message
        }
        
        errorResponse := ErrorResponse{
            Error:     "validation_failed",
            Message:   "Request validation failed",
            Details:   details,
            RequestID: getRequestID(r),
            Timestamp: time.Now(),
        }
        
        w.WriteHeader(422)
        json.NewEncoder(w).Encode(errorResponse)
        return
    }
    
    // Create user...
}
```

---

### 2. Authentication Errors (401)

```json
{
  "error": "unauthorized",
  "message": "Authentication required",
  "details": {
    "reason": "Token has expired"
  }
}
```

---

### 3. Authorization Errors (403)

```json
{
  "error": "forbidden",
  "message": "You don't have permission to access this resource",
  "details": {
    "required_role": "admin",
    "your_role": "user"
  }
}
```

---

### 4. Not Found Errors (404)

```json
{
  "error": "not_found",
  "message": "User with ID 123 not found"
}
```

---

### 5. Conflict Errors (409)

```json
{
  "error": "resource_exists",
  "message": "User with email john@example.com already exists",
  "details": {
    "field": "email",
    "value": "john@example.com"
  }
}
```

---

### 6. Rate Limit Errors (429)

```json
{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please try again later.",
  "details": {
    "limit": 100,
    "remaining": 0,
    "reset_at": "2025-01-16T11:00:00Z"
  }
}
```

---

### 7. Server Errors (500)

```json
{
  "error": "internal_server_error",
  "message": "An unexpected error occurred. Please try again later.",
  "request_id": "req-abc123"
}
```

**Never expose:**
- Stack traces (in production)
- Database errors
- Internal paths
- Sensitive data

---

## Error Codes

### Standardized Error Codes

```go
const (
    ErrValidationFailed    = "validation_failed"
    ErrUnauthorized        = "unauthorized"
    ErrForbidden           = "forbidden"
    ErrNotFound            = "not_found"
    ErrResourceExists      = "resource_exists"
    ErrRateLimitExceeded   = "rate_limit_exceeded"
    ErrInternalServerError = "internal_server_error"
    ErrBadRequest          = "bad_request"
)
```

**Usage:**

```go
return &ErrorResponse{
    Error:   ErrNotFound,
    Message: fmt.Sprintf("User with ID %d not found", userID),
}
```

---

## Error Helper Functions

```go
package errors

import (
    "encoding/json"
    "net/http"
    "time"
)

type ErrorResponse struct {
    Error     string            `json:"error"`
    Message   string            `json:"message"`
    Details   map[string]string `json:"details,omitempty"`
    RequestID string            `json:"request_id"`
    Timestamp time.Time         `json:"timestamp"`
}

func RespondWithError(w http.ResponseWriter, statusCode int, errorCode, message string, details map[string]string, requestID string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    
    json.NewEncoder(w).Encode(ErrorResponse{
        Error:     errorCode,
        Message:   message,
        Details:   details,
        RequestID: requestID,
        Timestamp: time.Now(),
    })
}

func NotFound(w http.ResponseWriter, message string, requestID string) {
    RespondWithError(w, 404, "not_found", message, nil, requestID)
}

func Unauthorized(w http.ResponseWriter, message string, requestID string) {
    RespondWithError(w, 401, "unauthorized", message, nil, requestID)
}

func Forbidden(w http.ResponseWriter, message string, requestID string) {
    RespondWithError(w, 403, "forbidden", message, nil, requestID)
}

func ValidationFailed(w http.ResponseWriter, details map[string]string, requestID string) {
    RespondWithError(w, 422, "validation_failed", "Request validation failed", details, requestID)
}

func InternalServerError(w http.ResponseWriter, requestID string) {
    RespondWithError(w, 500, "internal_server_error", "An unexpected error occurred", nil, requestID)
}

// Usage
func GetUser(w http.ResponseWriter, r *http.Request) {
    requestID := getRequestID(r)
    
    user, err := db.GetUser(userID)
    if err == sql.ErrNoRows {
        errors.NotFound(w, "User not found", requestID)
        return
    }
    
    json.NewEncoder(w).Encode(user)
}
```

---

## Request ID Tracking

**Why:** Correlate errors across logs

```go
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = generateRequestID()
        }
        
        ctx := context.WithValue(r.Context(), "request_id", requestID)
        w.Header().Set("X-Request-ID", requestID)
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func getRequestID(r *http.Request) string {
    if id := r.Context().Value("request_id"); id != nil {
        return id.(string)
    }
    return ""
}
```

**Error with request ID:**
```json
{
  "error": "internal_server_error",
  "message": "Something went wrong",
  "request_id": "req-abc123"
}
```

**User reports:** "I got an error with request ID req-abc123"

**You search logs:**
```
[req-abc123] Database connection failed: timeout after 30s
[req-abc123] Stack trace: ...
```

---

## Logging vs Error Response

**What to log:**
```go
log.Error("Failed to fetch user",
    "request_id", requestID,
    "user_id", userID,
    "error", err.Error(),
    "stack_trace", debug.Stack(),
)
```

**What to return to client:**
```json
{
  "error": "internal_server_error",
  "message": "An unexpected error occurred",
  "request_id": "req-abc123"
}
```

**Never return to client:**
- Stack traces
- Database errors
- Internal paths
- SQL queries
- Configuration details

---

## Error Handling Middleware

```go
type AppError struct {
    Code    int
    Error   string
    Message string
    Details map[string]string
}

type AppHandler func(http.ResponseWriter, *http.Request) *AppError

func (fn AppHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := fn(w, r); err != nil {
        requestID := getRequestID(r)
        
        // Log error
        log.Error("Request failed",
            "request_id", requestID,
            "error", err.Message,
            "code", err.Code,
        )
        
        // Return error to client
        RespondWithError(w, err.Code, err.Error, err.Message, err.Details, requestID)
    }
}

// Usage
func GetUser(w http.ResponseWriter, r *http.Request) *AppError {
    userID, _ := strconv.Atoi(mux.Vars(r)["id"])
    
    user, err := db.GetUser(userID)
    if err == sql.ErrNoRows {
        return &AppError{
            Code:    404,
            Error:   "not_found",
            Message: "User not found",
        }
    }
    if err != nil {
        return &AppError{
            Code:    500,
            Error:   "internal_server_error",
            Message: "Failed to fetch user",
        }
    }
    
    json.NewEncoder(w).Encode(user)
    return nil
}

// Register
r.Handle("/users/{id}", AppHandler(GetUser))
```

---

## Panic Recovery

```go
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                requestID := getRequestID(r)
                
                // Log panic
                log.Error("Panic recovered",
                    "request_id", requestID,
                    "error", err,
                    "stack_trace", debug.Stack(),
                )
                
                // Return 500 to client
                RespondWithError(w, 500, "internal_server_error",
                    "An unexpected error occurred", nil, requestID)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}
```

---

## Best Practices

### 1. Be Consistent

Same error format everywhere.

### 2. Don't Leak Information

```
❌ "Database connection to postgres://prod-db-1.internal:5432 failed"
✅ "An unexpected error occurred"
```

### 3. Generic Messages for Security

```
❌ "User john@example.com doesn't exist"
✅ "Invalid email or password"
```

Prevents email enumeration.

### 4. Include Request ID

Always include request ID for debugging.

### 5. Validation Errors Are 422, Not 400

- 400: Malformed request (invalid JSON)
- 422: Valid format, invalid values

### 6. Log Everything, Return Little

Log details internally, return generic message to client.

---

## Real-World Example: Stripe

```json
{
  "error": {
    "type": "card_error",
    "code": "card_declined",
    "message": "Your card was declined.",
    "param": "card",
    "doc_url": "https://stripe.com/docs/error-codes/card-declined"
  }
}
```

Features:
- Error type classification
- Specific error code
- User-friendly message
- Documentation link
- Field that caused error

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Error responses should include a __________ for debugging.
2. Stack traces should __________ be exposed in production.
3. For validation errors, use status code __________.
4. Generic messages prevent __________ enumeration attacks.

### True/False

1. ⬜ Stack traces should be returned to clients for debugging
2. ⬜ All errors should return status code 500
3. ⬜ Request IDs help correlate errors in logs
4. ⬜ Database error messages should be returned to clients
5. ⬜ Validation errors should use 400 Bad Request

### Multiple Choice

1. What should you return when user is not found?
   - A) 400 Bad Request
   - B) 401 Unauthorized
   - C) 404 Not Found
   - D) 422 Unprocessable Entity

2. What should you include in error responses?
   - A) Stack traces
   - B) Database errors
   - C) Request ID
   - D) SQL queries

3. What's the correct status code for validation errors?
   - A) 400
   - B) 404
   - C) 422
   - D) 500

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. request_id
2. never
3. 422
4. email (or user)

**True/False:**
1. ❌ False - Never expose stack traces in production
2. ❌ False - Use appropriate status codes (4xx for client, 5xx for server)
3. ✅ True - Request IDs correlate errors across logs
4. ❌ False - Never expose internal errors to clients
5. ❌ False - Use 422 for validation, 400 for malformed requests

**Multiple Choice:**
1. **C** - 404 Not Found is correct for missing resources
2. **C** - Only request ID should be included (safe)
3. **C** - 422 Unprocessable Entity for validation errors

</details>

</details>