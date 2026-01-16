# 3.1 Versioning Strategies

## Why Version APIs?

**Problem:**
```
API change breaks existing clients
Mobile app v1.0 stops working
Third-party integrations fail
Angry users
```

**Solution: API Versioning**
- Maintain backward compatibility
- Allow gradual migration
- Support multiple versions simultaneously

---

## Versioning Strategies

### 1. URL Path Versioning ✅ Recommended

```
https://api.example.com/v1/users
https://api.example.com/v2/users
```

**Pros:**
- ✅ Clear and explicit
- ✅ Easy to test (just change URL)
- ✅ Simple routing
- ✅ Works everywhere (browser, curl, Postman)
- ✅ Cacheable

**Cons:**
- ❌ Version in every URL
- ❌ URL duplication

**Implementation (Go):**
```go
r := mux.NewRouter()

// v1 routes
v1 := r.PathPrefix("/api/v1").Subrouter()
v1.HandleFunc("/users", v1.GetUsers)
v1.HandleFunc("/users/{id}", v1.GetUser)

// v2 routes
v2 := r.PathPrefix("/api/v2").Subrouter()
v2.HandleFunc("/users", v2.GetUsers)
v2.HandleFunc("/users/{id}", v2.GetUser)
```

**Real examples:**
- Stripe: `https://api.stripe.com/v1/charges`
- Twitter: `https://api.twitter.com/2/tweets`
- GitHub: (date-based versioning in headers)

---

### 2. Header Versioning

```http
GET /api/users
Accept: application/vnd.myapi.v2+json
```

or

```http
GET /api/users
API-Version: 2
```

**Pros:**
- ✅ Clean URLs
- ✅ RESTful purist approach
- ✅ Same endpoint for all versions

**Cons:**
- ❌ Less discoverable
- ❌ Harder to test in browser
- ❌ Complex routing
- ❌ Caching issues

**Implementation (Go):**
```go
func VersionMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        version := r.Header.Get("API-Version")
        
        if version == "" {
            version = "1" // Default
        }
        
        ctx := context.WithValue(r.Context(), "api_version", version)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func GetUsers(w http.ResponseWriter, r *http.Request) {
    version := r.Context().Value("api_version").(string)
    
    switch version {
    case "1":
        getUsersV1(w, r)
    case "2":
        getUsersV2(w, r)
    default:
        http.Error(w, "Unsupported version", 400)
    }
}
```

---

### 3. Query Parameter

```
GET /api/users?version=2
```

**Pros:**
- ✅ Simple

**Cons:**
- ❌ Caching issues
- ❌ Not RESTful
- ❌ Query params for filtering, not versioning

**Rarely used in production.**

---

### 4. Content Negotiation (Media Type)

```http
GET /api/users
Accept: application/vnd.myapi.v2+json
```

**Pros:**
- ✅ Very RESTful
- ✅ Standards-compliant

**Cons:**
- ❌ Complex
- ❌ Hard to test
- ❌ Overkill for most APIs

---

### 5. Subdomain Versioning

```
https://v1.api.example.com/users
https://v2.api.example.com/users
```

**Pros:**
- ✅ Clear separation
- ✅ Can deploy separately

**Cons:**
- ❌ DNS/SSL complexity
- ❌ Infrastructure overhead

---

## Comparison Table

| Strategy | Discoverability | Simplicity | Caching | RESTful |
|----------|----------------|------------|---------|---------|
| URL Path | ✅ High | ✅ Simple | ✅ Easy | ⚠️ Debatable |
| Header | ❌ Low | ❌ Complex | ⚠️ Tricky | ✅ Yes |
| Query Param | ✅ High | ✅ Simple | ❌ Hard | ❌ No |
| Media Type | ❌ Low | ❌ Complex | ⚠️ Tricky | ✅ Yes |
| Subdomain | ✅ High | ⚠️ Medium | ✅ Easy | ⚠️ Debatable |

---

## Best Practices

### 1. Start with v1

```
❌ /api/users
✅ /api/v1/users
```

Even for first version. Adding versioning later is painful.

---

### 2. Major Versions Only

```
✅ v1, v2, v3
❌ v1.0, v1.1, v1.2
```

Minor changes should be backward compatible.

---

### 3. Default Version

```go
func GetUsers(w http.ResponseWriter, r *http.Request) {
    version := r.URL.Query().Get("version")
    
    if version == "" {
        version = "2" // Latest stable
    }
    
    // Route to version
}
```

Or redirect:
```go
http.Redirect(w, r, "/api/v2/users", http.StatusMovedPermanently)
```

---

### 4. Sunset Policy

Announce deprecation timeline:

```http
GET /api/v1/users

Sunset: Sat, 31 Dec 2025 23:59:59 GMT
Deprecation: true
Link: </api/v2/users>; rel="successor-version"
```

Example timeline:
```
Jan 1:  v2 released
Mar 1:  v1 deprecated (announced)
Sep 1:  v1 sunset warning (6 months)
Jan 1:  v1 removed (1 year total)
```

---

### 5. Document Changes

```markdown
# API Changelog

## v2 (2025-01-15)
### Breaking Changes
- `/users` response format changed
- `created_at` now ISO 8601 format

### New Features
- Added `/users/{id}/posts` endpoint

### Deprecated
- `user_type` field (use `role` instead)

## v1 (2024-06-01)
- Initial release
```

---

## Implementation Example

### Complete Versioned API (Go)

```go
package main

import (
    "encoding/json"
    "net/http"
    
    "github.com/gorilla/mux"
)

// v1 User model
type UserV1 struct {
    ID        int    `json:"id"`
    Name      string `json:"name"`
    Email     string `json:"email"`
    CreatedAt string `json:"created_at"` // String format
}

// v2 User model
type UserV2 struct {
    ID        int    `json:"id"`
    Name      string `json:"name"`
    Email     string `json:"email"`
    Role      string `json:"role"`        // New field
    CreatedAt string `json:"created_at"`  // ISO 8601
}

// v1 handlers
func (h *HandlerV1) GetUsers(w http.ResponseWriter, r *http.Request) {
    users := []UserV1{
        {1, "John", "john@example.com", "2024-01-15"},
    }
    
    json.NewEncoder(w).Encode(users)
}

func (h *HandlerV1) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id := vars["id"]
    
    user := UserV1{1, "John", "john@example.com", "2024-01-15"}
    
    // Add deprecation warning
    w.Header().Set("Deprecation", "true")
    w.Header().Set("Sunset", "Sat, 31 Dec 2025 23:59:59 GMT")
    w.Header().Set("Link", "</api/v2/users/"+id+">; rel=\"successor-version\"")
    
    json.NewEncoder(w).Encode(user)
}

// v2 handlers
func (h *HandlerV2) GetUsers(w http.ResponseWriter, r *http.Request) {
    users := []UserV2{
        {1, "John", "john@example.com", "admin", "2024-01-15T10:00:00Z"},
    }
    
    json.NewEncoder(w).Encode(users)
}

func (h *HandlerV2) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    
    user := UserV2{1, "John", "john@example.com", "admin", "2024-01-15T10:00:00Z"}
    
    json.NewEncoder(w).Encode(user)
}

func main() {
    r := mux.NewRouter()
    
    v1Handler := &HandlerV1{}
    v2Handler := &HandlerV2{}
    
    // v1 routes
    v1 := r.PathPrefix("/api/v1").Subrouter()
    v1.HandleFunc("/users", v1Handler.GetUsers).Methods("GET")
    v1.HandleFunc("/users/{id}", v1Handler.GetUser).Methods("GET")
    
    // v2 routes
    v2 := r.PathPrefix("/api/v2").Subrouter()
    v2.HandleFunc("/users", v2Handler.GetUsers).Methods("GET")
    v2.HandleFunc("/users/{id}", v2Handler.GetUser).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}

type HandlerV1 struct{}
type HandlerV2 struct{}
```

---

## Migration Strategy

### Option 1: Proxy Pattern

```go
// v1 calls v2 internally with transformation
func (h *HandlerV1) GetUsers(w http.ResponseWriter, r *http.Request) {
    // Get data from v2
    usersV2 := getUsersV2Internal()
    
    // Transform to v1 format
    usersV1 := transformV2ToV1(usersV2)
    
    json.NewEncoder(w).Encode(usersV1)
}
```

**Benefit:** Single source of truth, easier to maintain.

---

### Option 2: Separate Handlers

```go
// v1 and v2 completely separate
// Each has own logic
```

**Benefit:** No coupling, can optimize separately.

---

### Option 3: Feature Flags

```go
func GetUsers(w http.ResponseWriter, r *http.Request) {
    version := getVersion(r)
    
    users := fetchUsers()
    
    if version == "v2" {
        // Include new fields
        users = enrichWithRoles(users)
    }
    
    json.NewEncoder(w).Encode(users)
}
```

**Benefit:** Gradual rollout, A/B testing.

---

## Handling Breaking vs Non-Breaking Changes

### Non-Breaking (Don't need new version)

**✅ Safe changes:**
- Add new endpoint
- Add new optional field
- Add new query parameter
- Add new HTTP method to existing endpoint

```go
// v1 stays same, just add feature
type UserV1 struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Phone string `json:"phone,omitempty"` // New optional field - OK!
}
```

---

### Breaking (Need new version)

**❌ Breaking changes:**
- Remove field
- Rename field
- Change field type
- Change response structure
- Remove endpoint
- Change error codes
- Change authentication method

```go
// v1
type UserV1 struct {
    UserType string `json:"user_type"`
}

// v2 - BREAKING CHANGE
type UserV2 struct {
    Role string `json:"role"` // Renamed field
}
```

---

## Real-World Example: Stripe

Stripe uses **date-based versioning** in headers:

```http
POST /v1/charges
Stripe-Version: 2024-11-20
```

**Benefits:**
- Multiple minor versions
- Granular control
- No URL changes

**Changes between versions:**
```
2023-08-16: Introduced Payment Intents
2023-10-16: Removed legacy Tokens API
2024-11-20: Changed error format
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The most common API versioning strategy is __________ versioning.
2. Breaking changes require a new __________ version.
3. When deprecating an API version, you should set the __________ header.
4. The recommended versioning format is __________ not v1.1, v1.2.

### True/False

1. ⬜ You should version your API from v0, not v1
2. ⬜ Adding a new optional field is a breaking change
3. ⬜ Query parameter versioning is the most RESTful approach
4. ⬜ You should support old API versions forever
5. ⬜ Renaming a field is a non-breaking change

### Multiple Choice

1. Which is a breaking change?
   - A) Adding new endpoint
   - B) Adding optional field
   - C) Removing a field
   - D) Adding new query parameter

2. What's the recommended versioning strategy?
   - A) Query parameter
   - B) URL path
   - C) Header
   - D) Subdomain

3. How long should you support deprecated versions?
   - A) Forever
   - B) 1 week
   - C) 6-12 months
   - D) Never

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. URL path (or URL)
2. major
3. Sunset (or Deprecation)
4. v1, v2, v3

**True/False:**
1. ❌ False - Start with v1
2. ❌ False - Optional fields are non-breaking
3. ❌ False - URL path is most common, RESTful is debatable
4. ❌ False - Deprecate and sunset old versions
5. ❌ False - Renaming is breaking (clients expect old name)

**Multiple Choice:**
1. **C** - Removing a field breaks existing clients
2. **B** - URL path versioning is most common and recommended
3. **C** - 6-12 months is typical deprecation period

</details>

</details>