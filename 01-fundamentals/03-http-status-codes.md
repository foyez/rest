# 1.3 HTTP Status Codes

## Overview

HTTP status codes indicate the result of a client's request.

```
┌──────────────────────────────────────────────┐
│  Client                        Server        │
│    │                              │          │
│    │  GET /users/123              │          │
│    │ ──────────────────────────>  │          │
│    │                              │          │
│    │  200 OK                      │          │
│    │  { "name": "John" }          │          │
│    │ <──────────────────────────  │          │
└──────────────────────────────────────────────┘
```

---

## Status Code Categories

| Range | Category | Meaning |
|-------|----------|---------|
| **1xx** | Informational | Request received, continuing |
| **2xx** | Success | Request successfully processed |
| **3xx** | Redirection | Further action needed |
| **4xx** | Client Error | Invalid request or client issue |
| **5xx** | Server Error | Server failed to process valid request |

---

## 2xx Success

### 200 OK
**When:** Request successful with response body

**Use for:** GET, PUT, PATCH

```http
GET /users/123
```
```http
200 OK

{
  "id": 123,
  "name": "John Doe"
}
```

---

### 201 Created
**When:** Resource successfully created

**Use for:** POST

**Must include:** `Location` header with new resource URL

```http
POST /users
{
  "name": "Jane Doe",
  "email": "jane@example.com"
}
```
```http
201 Created
Location: /users/124

{
  "id": 124,
  "name": "Jane Doe",
  "email": "jane@example.com",
  "created_at": "2025-01-16T10:00:00Z"
}
```

---

### 202 Accepted
**When:** Request accepted but processing not complete

**Use for:** Async operations, background jobs

```http
POST /reports/generate
{
  "type": "annual",
  "year": 2024
}
```
```http
202 Accepted

{
  "job_id": "abc123",
  "status": "processing",
  "status_url": "/jobs/abc123"
}
```

**Check status later:**
```http
GET /jobs/abc123
```
```http
200 OK

{
  "job_id": "abc123",
  "status": "completed",
  "result_url": "/reports/annual-2024.pdf"
}
```

---

### 204 No Content
**When:** Success but no response body

**Use for:** DELETE, sometimes PUT/PATCH

```http
DELETE /users/123
```
```http
204 No Content
```

**Real-world:** Most RESTful APIs return 204 for DELETE.

---

## 3xx Redirection

### 301 Moved Permanently
**When:** Resource permanently moved to new URL

```http
GET /old-api/users
```
```http
301 Moved Permanently
Location: /api/v2/users
```

**Use case:** API versioning, URL restructuring

---

### 302 Found (Temporary Redirect)
**When:** Resource temporarily at different URL

```http
GET /reports/latest
```
```http
302 Found
Location: /reports/2024-annual
```

---

### 304 Not Modified
**When:** Resource hasn't changed since last request

**Used with:** Caching, `If-Modified-Since` header

```http
GET /products/123
If-Modified-Since: Wed, 15 Jan 2025 10:00:00 GMT
```
```http
304 Not Modified
```

Client uses cached version. Saves bandwidth!

---

## 4xx Client Errors

### 400 Bad Request
**When:** Request is malformed or invalid syntax

**Examples:**
- Invalid JSON
- Missing required header
- Invalid data type

```http
POST /users
{
  "name": "John",
  "email": "not-an-email"  // Invalid format
}
```
```http
400 Bad Request

{
  "error": "Invalid request",
  "message": "Email format is invalid"
}
```

---

### 401 Unauthorized
**When:** Client is **not authenticated**

**Meaning:** "Who are you? I don't know you."

```http
GET /users/123
```
```http
401 Unauthorized
WWW-Authenticate: Bearer

{
  "error": "Authentication required",
  "message": "Please provide a valid token"
}
```

**Common causes:**
- Missing token
- Invalid token
- Expired token

---

### 403 Forbidden
**When:** Client is authenticated but **not authorized**

**Meaning:** "I know who you are, but you're not allowed."

```http
GET /admin/users
Authorization: Bearer valid-user-token
```
```http
403 Forbidden

{
  "error": "Forbidden",
  "message": "You don't have permission to access this resource"
}
```

**Common causes:**
- User lacks required role (not admin)
- Trying to access another user's data
- IP not whitelisted

---

### 401 vs 403: Real Examples

| Scenario | Code | Example |
|----------|------|---------|
| No token provided | 401 | `GET /profile` (no Authorization header) |
| Invalid token | 401 | `Authorization: Bearer invalid-token` |
| Valid token, not your resource | 403 | Regular user trying to delete another user |
| Valid token, insufficient role | 403 | User trying to access `/admin/settings` |

**Memory aid:**
- **401 = Authentication** (identity)
- **403 = Authorization** (permissions)

---

### 404 Not Found
**When:** Resource doesn't exist

```http
GET /users/999999
```
```http
404 Not Found

{
  "error": "Not found",
  "message": "User with ID 999999 not found"
}
```

**Also use for:**
- Hiding resources for security (pretend doesn't exist)
- Invalid endpoints

---

### 405 Method Not Allowed
**When:** HTTP method not supported for resource

**Must include:** `Allow` header

```http
DELETE /products
```
```http
405 Method Not Allowed
Allow: GET, POST

{
  "error": "Method not allowed",
  "message": "DELETE is not supported on /products"
}
```

---

### 409 Conflict
**When:** Request conflicts with current state

**Common uses:**
- Duplicate unique field (email, username)
- Version conflict
- Resource already exists

```http
POST /users
{
  "email": "john@example.com",
  "name": "John"
}
```
```http
409 Conflict

{
  "error": "Conflict",
  "message": "User with email john@example.com already exists"
}
```

**Another example: Optimistic locking**
```http
PUT /posts/123
If-Match: "v1"

{
  "title": "Updated"
}
```
```http
409 Conflict

{
  "error": "Conflict",
  "message": "Post was modified by another user",
  "current_version": "v2"
}
```

---

### 422 Unprocessable Entity
**When:** Request is valid but contains semantic errors

**Use for:** Validation failures

```http
POST /users
{
  "name": "",
  "email": "john@example.com",
  "age": -5
}
```
```http
422 Unprocessable Entity

{
  "error": "Validation failed",
  "fields": {
    "name": "Name is required",
    "age": "Age must be positive"
  }
}
```

---

### 400 vs 422

| Code | Meaning | Example |
|------|---------|---------|
| 400 | Malformed request | Invalid JSON syntax |
| 422 | Valid format, invalid values | JSON is valid but name is empty |

```http
❌ 400: { "name": "John",  <- Missing closing brace

✅ 422: { "name": "", "email": "invalid" }  <- Valid JSON, invalid values
```

---

### 429 Too Many Requests
**When:** Rate limit exceeded

**Must include:** `Retry-After` header

```http
GET /api/v1/search
```
```http
429 Too Many Requests
Retry-After: 60

{
  "error": "Rate limit exceeded",
  "message": "Try again in 60 seconds",
  "limit": 100,
  "remaining": 0,
  "reset_at": "2025-01-16T11:00:00Z"
}
```

---

## 5xx Server Errors

### 500 Internal Server Error
**When:** Unexpected server error

**Use for:**
- Unhandled exceptions
- Database crashes
- Null pointer errors

```http
500 Internal Server Error

{
  "error": "Internal server error",
  "message": "Something went wrong. Please try again later.",
  "request_id": "abc-123"
}
```

**⚠️ Never expose:**
- Stack traces
- Database errors
- Internal paths

**✅ Do log internally:**
```go
log.Error("Database connection failed",
    "request_id", requestID,
    "error", err,
    "query", query)
```

---

### 502 Bad Gateway
**When:** Server received invalid response from upstream

**Common scenarios:**
- API gateway → microservice fails
- Reverse proxy → backend down
- Third-party API returns garbage

```http
502 Bad Gateway

{
  "error": "Bad gateway",
  "message": "Upstream service returned invalid response"
}
```

**Example flow:**
```
Client -> NGINX -> Go API -> Database
                    ↑
                  Returns malformed response
```

---

### 503 Service Unavailable
**When:** Server temporarily can't handle request

**Use for:**
- Maintenance mode
- Server overloaded
- Database down

**Must include:** `Retry-After` header (optional but recommended)

```http
503 Service Unavailable
Retry-After: 120

{
  "error": "Service unavailable",
  "message": "Server is under maintenance. Try again in 2 minutes"
}
```

---

### 504 Gateway Timeout
**When:** Upstream service didn't respond in time

```http
504 Gateway Timeout

{
  "error": "Gateway timeout",
  "message": "Upstream service took too long to respond"
}
```

**Example:**
```
Client -> API Gateway (30s timeout) -> Slow Microservice (takes 45s)
                                       ↑
                                    Times out
```

---

## Status Code Decision Tree

```
Is request successful?
├─ Yes
│  ├─ Created new resource? → 201
│  ├─ No response body? → 204
│  ├─ Async processing? → 202
│  └─ Otherwise → 200
│
└─ No
   ├─ Client's fault?
   │  ├─ Not authenticated? → 401
   │  ├─ Not authorized? → 403
   │  ├─ Resource not found? → 404
   │  ├─ Method not allowed? → 405
   │  ├─ Duplicate/conflict? → 409
   │  ├─ Validation error? → 422
   │  ├─ Rate limited? → 429
   │  ├─ Malformed request? → 400
   │  └─ Other client error → 400
   │
   └─ Server's fault?
      ├─ Upstream failed? → 502
      ├─ Temporarily down? → 503
      ├─ Upstream timeout? → 504
      └─ Unexpected error → 500
```

---

## Real-World Example: E-commerce API

### Scenario 1: Create Order
```http
POST /orders
Authorization: Bearer token123

{
  "product_id": 456,
  "quantity": 2,
  "shipping_address": "123 Main St"
}
```

**Possible responses:**

| Status | Reason |
|--------|--------|
| 201 | Order created successfully |
| 400 | Invalid JSON |
| 401 | Token missing or invalid |
| 403 | Account suspended |
| 404 | Product 456 doesn't exist |
| 409 | Product out of stock |
| 422 | Quantity must be positive |
| 500 | Payment gateway error |

---

### Scenario 2: Get Order Status
```http
GET /orders/789
Authorization: Bearer token123
```

**Possible responses:**

| Status | Reason |
|--------|--------|
| 200 | Order found, return details |
| 401 | Not authenticated |
| 403 | Not your order |
| 404 | Order doesn't exist |
| 500 | Database error |

---

## Error Response Best Practices

### ✅ Good Error Response
```json
{
  "error": "Validation failed",
  "message": "One or more fields are invalid",
  "fields": {
    "email": "Email format is invalid",
    "age": "Age must be between 18 and 120"
  },
  "request_id": "req-abc123",
  "timestamp": "2025-01-16T10:30:00Z"
}
```

**Include:**
- Clear error type
- Human-readable message
- Field-level errors (for validation)
- Request ID (for debugging)
- Timestamp

---

### ❌ Bad Error Responses

**Too vague:**
```json
{
  "error": "Error"
}
```

**Exposes internals:**
```json
{
  "error": "NullPointerException at line 45 in UserService.java",
  "stack_trace": "..."
}
```

**Inconsistent format:**
```json
// Sometimes:
{ "error": "Not found" }

// Other times:
{ "message": "Resource not found", "code": 404 }
```

---

## Status Code Cheat Sheet

| Scenario | Code |
|----------|------|
| Get resource successfully | 200 |
| Create resource | 201 |
| Delete resource | 204 |
| Async operation started | 202 |
| Malformed request | 400 |
| Not authenticated | 401 |
| Not authorized | 403 |
| Resource not found | 404 |
| Method not supported | 405 |
| Duplicate resource | 409 |
| Validation error | 422 |
| Rate limited | 429 |
| Server error | 500 |
| Upstream service failed | 502 |
| Server down | 503 |
| Upstream timeout | 504 |

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Status code __________ indicates a resource was successfully created.
2. __________ means the client is not authenticated, while __________ means they're authenticated but not authorized.
3. Status code 204 means __________ and has no response body.
4. For validation errors, use status code __________.

### True/False

1. ⬜ 400 and 422 can be used interchangeably
2. ⬜ 500 errors are always the client's fault
3. ⬜ DELETE requests should return 200 OK
4. ⬜ 401 means "Who are you?" and 403 means "You're not allowed"
5. ⬜ A 502 error means the server is down

### Multiple Choice

1. User tries to access `/admin` with a valid user token but isn't an admin. What status code?
   - A) 400
   - B) 401
   - C) 403
   - D) 404

2. User sends `{ "email": "" }` to create account. What status code?
   - A) 400
   - B) 422
   - C) 500
   - D) 409

3. Which status code requires a `Location` header?
   - A) 200
   - B) 201
   - C) 204
   - D) 404

### Scenario-Based

**Scenario 1:**
```http
POST /users
{
  "email": "existing@example.com",
  "name": "John"
}
```
Email already exists in database. What status code?

**Scenario 2:**
```http
DELETE /posts/123
```
Post doesn't exist. What status code?

**Scenario 3:**
```http
GET /reports
```
Report generation takes 5 minutes. What status code for initial request?

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. 201
2. 401, 403
3. No Content
4. 422

**True/False:**
1. ❌ False - 400 for malformed requests, 422 for validation errors
2. ❌ False - 5xx errors are server's fault
3. ❌ False - DELETE should return 204 No Content
4. ✅ True
5. ❌ False - 502 means upstream service failed, not the server itself

**Multiple Choice:**
1. **C** - 403 Forbidden (authenticated but not authorized)
2. **B** - 422 Unprocessable Entity (validation failed)
3. **B** - 201 Created requires Location header

**Scenario-Based:**
1. **409 Conflict** - Email already exists (duplicate resource)
2. **404 Not Found** - Resource doesn't exist (or return 204 if treating DELETE as idempotent)
3. **202 Accepted** - Async operation, return job ID

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between 401 and 403 with real examples

### Question 2: When should you use 400 vs 422?

### Question 3: Should DELETE return 404 if resource doesn't exist, or 204? Why?

### Question 4: What's the difference between 502, 503, and 504?

### Question 5: What should you include in error responses? What should you never include?

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

### Answer 1: 401 vs 403

**401 Unauthorized = Authentication problem** ("Who are you?")
**403 Forbidden = Authorization problem** ("You're not allowed")

**Real examples:**

**401 scenarios:**
```
1. Missing token:
   GET /profile
   -> 401 Unauthorized
   
2. Expired token:
   GET /profile
   Authorization: Bearer expired-token
   -> 401 Unauthorized
   
3. Invalid token:
   Authorization: Bearer invalid-token
   -> 401 Unauthorized
```

**403 scenarios:**
```
1. Regular user accessing admin endpoint:
   GET /admin/users
   Authorization: Bearer valid-user-token
   -> 403 Forbidden
   
2. User trying to delete another user's post:
   DELETE /posts/123  (owned by user 456)
   Authorization: Bearer user-789-token
   -> 403 Forbidden
   
3. User from blocked IP:
   GET /api/data
   Authorization: Bearer valid-token
   X-Forwarded-For: 1.2.3.4 (blocked)
   -> 403 Forbidden
```

**Memory trick:**
- 401: Server doesn't know who you are → "Prove your identity"
- 403: Server knows who you are → "You don't have permission"

---

### Answer 2: 400 vs 422

**400 Bad Request:**
- **Syntax/format errors**
- Request can't be parsed
- Malformed data

**422 Unprocessable Entity:**
- **Semantic/validation errors**
- Request is well-formed but contains invalid values
- Business logic validation failed

**Examples:**

**400 scenarios:**
```json
// Invalid JSON syntax
POST /users
{
  "name": "John",
  "email": "john@example.com"  <- Missing closing brace

// Wrong Content-Type
POST /users
Content-Type: text/plain
{ "name": "John" }

// Missing required header
POST /users
(No Content-Type header)
```

**422 scenarios:**
```json
// Valid JSON, invalid email format
POST /users
{
  "name": "John",
  "email": "not-an-email"
}

// Valid format, business rule violated
POST /users
{
  "name": "",
  "age": -5,
  "email": "valid@example.com"
}

// Valid structure, logic error
POST /orders
{
  "product_id": 999999,  // Doesn't exist
  "quantity": 0          // Must be > 0
}
```

**Decision tree:**
```
Can we parse the request?
├─ No → 400 (malformed)
└─ Yes → Valid but logical errors? → 422 (validation)
```

---

### Answer 3: DELETE and 404 vs 204

**Two schools of thought:**

**Option 1: Always return 204 (Idempotent approach) ✅ Recommended**
```
DELETE /users/123  (exists)     -> 204 No Content
DELETE /users/123  (not exists) -> 204 No Content
```

**Reasoning:**
- DELETE is idempotent
- Final state is the same: resource doesn't exist
- Simplifies client logic
- Matches spec: "same outcome regardless of current state"

**Option 2: Return 404 if not exists**
```
DELETE /users/123  (exists)     -> 204 No Content
DELETE /users/123  (not exists) -> 404 Not Found
```

**Reasoning:**
- More informative
- Client knows if resource was actually deleted
- Helps catch bugs (typos in IDs)

**My recommendation: Use 204 always**

**Implementation (Go):**
```go
func DeleteUser(id int) (int, error) {
    result := db.Delete(&User{}, id)
    
    // Always return 204, even if nothing was deleted
    return 204, nil
}
```

**Why 204 is better:**
- Simpler client code (no error handling for 404)
- True idempotency
- Matches HTTP spec interpretation
- Used by major APIs (GitHub, Stripe)

---

### Answer 4: 502 vs 503 vs 504

All are 5xx (server errors), but indicate different problems:

**502 Bad Gateway**
- **Upstream service returned invalid response**
- Gateway/proxy got corrupted data from backend

**503 Service Unavailable**
- **Server is temporarily unable to handle request**
- Overloaded or in maintenance

**504 Gateway Timeout**
- **Upstream service didn't respond in time**
- Timeout waiting for backend

**Visual explanation:**

```
Client -> Proxy -> Backend Service -> Database

502: Backend returns garbage
     Client -> Proxy -> Backend ❌ (invalid response)
     
503: Server is down or overloaded
     Client -> Proxy ❌ (server can't handle it)
     
504: Backend takes too long
     Client -> Proxy -> Backend ... ⏱️ (timeout)
```

**Real examples:**

**502:**
```
NGINX receives corrupted JSON from app server
API Gateway can't parse response from microservice
```

**503:**
```http
503 Service Unavailable
Retry-After: 300

{
  "error": "Service unavailable",
  "message": "Database is under maintenance"
}
```

**504:**
```
Client timeout: 30s
Backend query takes: 60s
-> 504 Gateway Timeout
```

**How to handle (client side):**
- **502**: Retry after delay (might be temporary)
- **503**: Honor `Retry-After` header
- **504**: Retry with exponential backoff

---

### Answer 5: What to include/exclude in error responses

**✅ Always include:**

```json
{
  "error": "Validation failed",
  "message": "Human-readable description",
  "request_id": "abc-123",
  "timestamp": "2025-01-16T10:30:00Z"
}
```

1. **Error type/code**
   - Clear identifier
   - Can be used programmatically

2. **Human-readable message**
   - Helpful for debugging
   - Clear, actionable

3. **Request ID**
   - For support/debugging
   - Correlate with server logs

4. **Timestamp**
   - When error occurred
   - Useful for auditing

**✅ Include for validation errors:**

```json
{
  "error": "Validation failed",
  "message": "One or more fields are invalid",
  "fields": {
    "email": "Email format is invalid",
    "age": "Age must be between 18 and 120",
    "password": "Password must be at least 8 characters"
  },
  "request_id": "abc-123"
}
```

**❌ Never include:**

1. **Stack traces**
   ```json
   ❌ {
     "error": "NullPointerException",
     "stack_trace": "at com.example.UserService.getUser..."
   }
   ```

2. **Internal paths**
   ```json
   ❌ {
     "error": "File not found: /var/www/internal/secrets.txt"
   }
   ```

3. **Database errors**
   ```json
   ❌ {
     "error": "SQL Error: Duplicate entry 'john@example.com' for key 'users.email'"
   }
   ```

4. **Version info**
   ```json
   ❌ {
     "error": "Server error",
     "server": "Apache 2.4.41",
     "php_version": "7.4.3"
   }
   ```

**Why?**
- Security: Don't expose internals
- Attackers can use info to find vulnerabilities

**✅ Instead, log detailed errors internally:**
```go
log.Error("Database error",
    "request_id", requestID,
    "user_id", userID,
    "error", err.Error(),
    "query", query,
    "stack", debug.Stack())

// Return generic error to client
return 500, map[string]string{
    "error": "Internal server error",
    "message": "Something went wrong",
    "request_id": requestID,
}
```

**Best practice structure:**

```go
type ErrorResponse struct {
    Error     string            `json:"error"`
    Message   string            `json:"message"`
    Fields    map[string]string `json:"fields,omitempty"`
    RequestID string            `json:"request_id"`
    Timestamp time.Time         `json:"timestamp"`
}
```

</details>

</details>