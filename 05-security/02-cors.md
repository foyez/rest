# 5.2 CORS (Cross-Origin Resource Sharing)

## What is CORS?

**Same-Origin Policy:** Browser blocks requests to different origins

```
Origin = Protocol + Domain + Port

https://api.example.com:443
  ↓        ↓           ↓
Protocol Domain      Port
```

**Same origin:**
```
https://example.com  →  https://example.com/api  ✅
```

**Different origin (blocked by default):**
```
https://example.com  →  https://api.example.com  ❌ (different subdomain)
https://example.com  →  http://example.com       ❌ (different protocol)
https://example.com  →  https://example.com:8080 ❌ (different port)
```

---

## Why CORS?

**Security:** Prevent malicious websites from accessing your API

**Example attack without CORS:**
```
1. User logs into bank.com
2. User visits evil.com
3. evil.com sends request to bank.com/transfer
4. Browser automatically sends cookies
5. Money transferred!
```

**With CORS:**
```
1. User visits evil.com
2. evil.com sends request to bank.com
3. Browser blocks (different origin)
4. No cookies sent
5. Attack prevented ✅
```

---

## How CORS Works

### Simple Request

```
┌──────────┐                                  ┌─────────┐
│ Browser  │                                  │  Server │
└────┬─────┘                                  └────┬────┘
     │                                             │
     │ GET /api/users                              │
     │ Origin: https://example.com                 │
     │ ──────────────────────────────────────────> │
     │                                             │
     │                                         Check origin
     │                                             │
     │ Access-Control-Allow-Origin: *              │
     │ <────────────────────────────────────────── │
     │                                             │
     Allow request
```

---

### Preflight Request (Complex)

**Triggered when:**
- Method: PUT, DELETE, PATCH
- Custom headers
- Content-Type: application/json

```
┌──────────┐                                  ┌─────────┐
│ Browser  │                                  │  Server │
└────┬─────┘                                  └────┬────┘
     │                                             │
     │ OPTIONS /api/users (preflight)              │
     │ Origin: https://example.com                 │
     │ Access-Control-Request-Method: POST         │
     │ Access-Control-Request-Headers: Content-Type│
     │ ──────────────────────────────────────────> │
     │                                             │
     │                                         Check if allowed
     │                                             │
     │ Access-Control-Allow-Origin: https://ex...  │
     │ Access-Control-Allow-Methods: POST, GET     │
     │ Access-Control-Allow-Headers: Content-Type  │
     │ Access-Control-Max-Age: 86400               │
     │ <────────────────────────────────────────── │
     │                                             │
     │ POST /api/users (actual request)            │
     │ ──────────────────────────────────────────> │
     │                                             │
     │ Response                                    │
     │ <────────────────────────────────────────── │
```

---

## CORS Headers

### Response Headers

**1. Access-Control-Allow-Origin**
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Origin: https://example.com
```

**Wildcard (*) WARNING:**
```
❌ Access-Control-Allow-Origin: *
   Access-Control-Allow-Credentials: true
   
This combination is NOT allowed!
```

---

**2. Access-Control-Allow-Methods**
```
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
```

---

**3. Access-Control-Allow-Headers**
```
Access-Control-Allow-Headers: Content-Type, Authorization, X-Custom-Header
```

---

**4. Access-Control-Allow-Credentials**
```
Access-Control-Allow-Credentials: true
```

Allows cookies to be sent.

---

**5. Access-Control-Max-Age**
```
Access-Control-Max-Age: 86400
```

Cache preflight for 24 hours.

---

**6. Access-Control-Expose-Headers**
```
Access-Control-Expose-Headers: X-Custom-Header, X-Request-ID
```

Allow JavaScript to access these headers.

---

## Implementation (Go)

### Basic CORS Middleware

```go
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Set CORS headers
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        // Handle preflight
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}
```

---

### Secure CORS (Whitelist Origins)

```go
func CORSMiddleware(allowedOrigins []string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")
            
            // Check if origin is allowed
            allowed := false
            for _, allowedOrigin := range allowedOrigins {
                if origin == allowedOrigin {
                    allowed = true
                    break
                }
            }
            
            if allowed {
                w.Header().Set("Access-Control-Allow-Origin", origin)
                w.Header().Set("Access-Control-Allow-Credentials", "true")
                w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
                w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
                w.Header().Set("Access-Control-Max-Age", "86400")
            }
            
            // Handle preflight
            if r.Method == "OPTIONS" {
                w.WriteHeader(http.StatusOK)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// Usage
allowedOrigins := []string{
    "https://example.com",
    "https://www.example.com",
    "http://localhost:3000", // Dev
}

r.Use(CORSMiddleware(allowedOrigins))
```

---

### Production-Ready CORS

```go
package main

import (
    "net/http"
    "strings"
    
    "github.com/gorilla/mux"
)

type CORSConfig struct {
    AllowedOrigins   []string
    AllowedMethods   []string
    AllowedHeaders   []string
    ExposedHeaders   []string
    AllowCredentials bool
    MaxAge           int
}

func NewCORSMiddleware(config CORSConfig) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")
            
            // Check if origin is allowed
            allowed := false
            if contains(config.AllowedOrigins, "*") {
                allowed = true
                w.Header().Set("Access-Control-Allow-Origin", "*")
            } else if contains(config.AllowedOrigins, origin) {
                allowed = true
                w.Header().Set("Access-Control-Allow-Origin", origin)
                w.Header().Set("Vary", "Origin")
            }
            
            if !allowed {
                next.ServeHTTP(w, r)
                return
            }
            
            // Set methods
            if len(config.AllowedMethods) > 0 {
                w.Header().Set("Access-Control-Allow-Methods",
                    strings.Join(config.AllowedMethods, ", "))
            }
            
            // Set headers
            if len(config.AllowedHeaders) > 0 {
                w.Header().Set("Access-Control-Allow-Headers",
                    strings.Join(config.AllowedHeaders, ", "))
            }
            
            // Set exposed headers
            if len(config.ExposedHeaders) > 0 {
                w.Header().Set("Access-Control-Expose-Headers",
                    strings.Join(config.ExposedHeaders, ", "))
            }
            
            // Set credentials
            if config.AllowCredentials {
                w.Header().Set("Access-Control-Allow-Credentials", "true")
            }
            
            // Set max age
            if config.MaxAge > 0 {
                w.Header().Set("Access-Control-Max-Age",
                    fmt.Sprintf("%d", config.MaxAge))
            }
            
            // Handle preflight
            if r.Method == "OPTIONS" {
                w.WriteHeader(http.StatusNoContent)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

func main() {
    r := mux.NewRouter()
    
    // CORS config
    corsConfig := CORSConfig{
        AllowedOrigins: []string{
            "https://example.com",
            "https://www.example.com",
        },
        AllowedMethods: []string{
            "GET", "POST", "PUT", "DELETE", "OPTIONS",
        },
        AllowedHeaders: []string{
            "Content-Type",
            "Authorization",
            "X-Request-ID",
        },
        ExposedHeaders: []string{
            "X-Request-ID",
        },
        AllowCredentials: true,
        MaxAge:           86400, // 24 hours
    }
    
    r.Use(NewCORSMiddleware(corsConfig))
    
    http.ListenAndServe(":8080", r)
}

func contains(slice []string, item string) bool {
    for _, s := range slice {
        if s == item {
            return true
        }
    }
    return false
}
```

---

## Common CORS Scenarios

### Scenario 1: Public API

```go
// Allow all origins
w.Header().Set("Access-Control-Allow-Origin", "*")
```

---

### Scenario 2: Frontend + Backend (Same Company)

```go
// Whitelist your domains
allowedOrigins := []string{
    "https://example.com",
    "https://app.example.com",
}
```

---

### Scenario 3: Development

```go
allowedOrigins := []string{
    "http://localhost:3000",
    "http://localhost:3001",
    "https://example.com", // Production
}
```

---

### Scenario 4: Mobile Apps

Mobile apps don't have origin, so CORS doesn't apply. But you can check User-Agent:

```go
userAgent := r.Header.Get("User-Agent")
if strings.Contains(userAgent, "MyMobileApp") {
    // Allow
}
```

---

## CORS with Credentials

**When sending cookies:**

```javascript
// Frontend
fetch('https://api.example.com/users', {
    credentials: 'include'  // Send cookies
})
```

**Backend must:**
```go
w.Header().Set("Access-Control-Allow-Origin", "https://example.com") // Specific origin
w.Header().Set("Access-Control-Allow-Credentials", "true")
```

**Cannot use wildcard with credentials:**
```go
❌ w.Header().Set("Access-Control-Allow-Origin", "*")
   w.Header().Set("Access-Control-Allow-Credentials", "true")
```

---

## Debugging CORS

### Browser Console Errors

**Error:** "No 'Access-Control-Allow-Origin' header"
```
Solution: Add Access-Control-Allow-Origin header
```

**Error:** "CORS policy: Response to preflight request doesn't pass"
```
Solution: Handle OPTIONS requests properly
```

**Error:** "Credentials flag is 'true', but Access-Control-Allow-Origin is '*'"
```
Solution: Use specific origin, not wildcard
```

---

### cURL Testing

```bash
# Simple request
curl -H "Origin: https://example.com" https://api.example.com/users

# Preflight request
curl -X OPTIONS \
  -H "Origin: https://example.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type" \
  https://api.example.com/users
```

---

## Security Best Practices

### 1. Don't Use Wildcard with Credentials

```go
❌ Access-Control-Allow-Origin: *
   Access-Control-Allow-Credentials: true
```

### 2. Whitelist Specific Origins

```go
✅ allowedOrigins := []string{
    "https://example.com",
    "https://app.example.com",
}
```

### 3. Validate Origin

```go
origin := r.Header.Get("Origin")
if !isAllowed(origin) {
    // Don't set CORS headers
    return
}
```

### 4. Be Careful with Null Origin

```go
if origin == "null" {
    // Local file:// or sandboxed iframe
    // Usually don't allow
}
```

---

## CORS vs CSRF

**CORS** = Prevents cross-origin requests
**CSRF** = Prevents unwanted actions

**Both needed:**
- CORS: Blocks evil.com from reading response
- CSRF tokens: Prevents evil.com from sending requests

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. CORS stands for Cross-Origin __________ Sharing.
2. The __________ header specifies allowed origins.
3. __________ requests are sent before actual requests.
4. Wildcard origin cannot be used with __________.

### True/False

1. ⬜ CORS is a server-side security feature
2. ⬜ Same origin = same protocol + domain + port
3. ⬜ OPTIONS requests are preflight requests
4. ⬜ Wildcard origin works with credentials
5. ⬜ Mobile apps are subject to CORS

### Multiple Choice

1. Which triggers a preflight request?
   - A) GET request
   - B) POST with Content-Type: application/x-www-form-urlencoded
   - C) POST with Content-Type: application/json
   - D) All requests

2. What's the correct CORS header for all origins?
   - A) Access-Control-Allow-Origin: all
   - B) Access-Control-Allow-Origin: *
   - C) Access-Control-Allow-Origin: true
   - D) CORS-Allow-All: true

3. When is preflight request cached?
   - A) Access-Control-Cache
   - B) Access-Control-Max-Age
   - C) Cache-Control
   - D) Never

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. Resource
2. Access-Control-Allow-Origin
3. Preflight (or OPTIONS)
4. credentials

**True/False:**
1. ❌ False - CORS is enforced by browsers (client-side)
2. ✅ True - All three must match
3. ✅ True - OPTIONS is preflight method
4. ❌ False - Wildcard doesn't work with credentials
5. ❌ False - Native mobile apps don't have CORS

**Multiple Choice:**
1. **C** - JSON triggers preflight
2. **B** - Asterisk (*) is wildcard
3. **B** - Access-Control-Max-Age caches preflight

</details>

</details>