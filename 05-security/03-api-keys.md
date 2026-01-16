# 5.3 API Keys

## What are API Keys?

**API Key** = Secret token that identifies the client application

```
X-API-Key: sk_live_4eC39HqLyjWDarjtT1
```

**Purpose:**
- Identify applications
- Rate limiting per client
- Track usage
- Control access
- Monetization (different tiers)

---

## API Key vs Other Auth

| Feature | API Key | JWT | OAuth |
|---------|---------|-----|-------|
| **Purpose** | Identify app | Authenticate user | Delegated access |
| **Stateful** | Yes | No | Yes |
| **Revocable** | ✅ Yes | ❌ No | ✅ Yes |
| **User context** | ❌ No | ✅ Yes | ✅ Yes |
| **Best for** | Server-to-server | User sessions | Third-party access |

---

## API Key Format

### Common Formats

**1. Prefixed (Stripe-style)**
```
sk_live_4eC39HqLyjWDarjtT1
pk_test_TYooMQauvdEDq54NiTphI7jx

Prefix:
sk_ = Secret key
pk_ = Publishable key
live_ = Production
test_ = Testing
```

**2. UUID**
```
550e8400-e29b-41d4-a716-446655440000
```

**3. Random String**
```
a8f5k2p9m1q7z5x4w6n2b8v5c1d9e7f0a3g6h4j2
```

---

## Generating API Keys

### Secure Generation (Go)

```go
package main

import (
    "crypto/rand"
    "encoding/base64"
    "encoding/hex"
    "fmt"
)

// Generate random API key
func GenerateAPIKey() (string, error) {
    bytes := make([]byte, 32)
    _, err := rand.Read(bytes)
    if err != nil {
        return "", err
    }
    
    return hex.EncodeToString(bytes), nil
}

// Generate with prefix (Stripe-style)
func GeneratePrefixedKey(prefix string) (string, error) {
    bytes := make([]byte, 24)
    _, err := rand.Read(bytes)
    if err != nil {
        return "", err
    }
    
    encoded := base64.RawURLEncoding.EncodeToString(bytes)
    return fmt.Sprintf("%s_%s", prefix, encoded), nil
}

func main() {
    // Simple key
    key, _ := GenerateAPIKey()
    fmt.Println(key)
    // Output: a8f5k2p9m1q7z5x4w6n2b8v5c1d9e7f0a3g6h4j2
    
    // Prefixed key
    secretKey, _ := GeneratePrefixedKey("sk_live")
    fmt.Println(secretKey)
    // Output: sk_live_4eC39HqLyjWDarjtT1
    
    publicKey, _ := GeneratePrefixedKey("pk_live")
    fmt.Println(publicKey)
    // Output: pk_live_TYooMQauvdEDq54NiTphI7jx
}
```

---

## Storing API Keys

### Database Schema

```sql
CREATE TABLE api_keys (
    id SERIAL PRIMARY KEY,
    key_hash VARCHAR(64) NOT NULL UNIQUE,
    key_prefix VARCHAR(20),
    user_id INT NOT NULL,
    name VARCHAR(255),
    scopes TEXT[],
    rate_limit INT DEFAULT 1000,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMP,
    expires_at TIMESTAMP,
    revoked BOOLEAN DEFAULT FALSE,
    
    INDEX idx_key_hash (key_hash),
    INDEX idx_user_id (user_id)
);
```

---

### Hash API Keys (Don't Store Plain Text!)

```go
import (
    "crypto/sha256"
    "encoding/hex"
)

// Hash API key before storing
func HashAPIKey(key string) string {
    hash := sha256.Sum256([]byte(key))
    return hex.EncodeToString(hash[:])
}

// Create API key
func CreateAPIKey(userID int, name string) (string, error) {
    // Generate key
    key, err := GeneratePrefixedKey("sk_live")
    if err != nil {
        return "", err
    }
    
    // Hash for storage
    keyHash := HashAPIKey(key)
    
    // Store only hash
    _, err = db.Exec(`
        INSERT INTO api_keys (key_hash, key_prefix, user_id, name)
        VALUES (?, ?, ?, ?)
    `, keyHash, "sk_live", userID, name)
    
    // Return plain key ONCE (user must save it)
    return key, nil
}
```

**Important:** Return plain key only once at creation!

---

## Validating API Keys

### Middleware

```go
func APIKeyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Get API key from header
        apiKey := r.Header.Get("X-API-Key")
        if apiKey == "" {
            // Try Authorization header
            auth := r.Header.Get("Authorization")
            if strings.HasPrefix(auth, "Bearer ") {
                apiKey = strings.TrimPrefix(auth, "Bearer ")
            }
        }
        
        if apiKey == "" {
            http.Error(w, "Missing API key", 401)
            return
        }
        
        // Validate key
        keyInfo, err := ValidateAPIKey(apiKey)
        if err != nil {
            http.Error(w, "Invalid API key", 401)
            return
        }
        
        // Check if revoked
        if keyInfo.Revoked {
            http.Error(w, "API key has been revoked", 401)
            return
        }
        
        // Check expiry
        if keyInfo.ExpiresAt != nil && time.Now().After(*keyInfo.ExpiresAt) {
            http.Error(w, "API key has expired", 401)
            return
        }
        
        // Update last used
        UpdateLastUsed(keyInfo.ID)
        
        // Add to context
        ctx := context.WithValue(r.Context(), "api_key", keyInfo)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

type APIKeyInfo struct {
    ID         int
    UserID     int
    Name       string
    Scopes     []string
    RateLimit  int
    Revoked    bool
    ExpiresAt  *time.Time
    LastUsedAt *time.Time
}

func ValidateAPIKey(key string) (*APIKeyInfo, error) {
    // Hash the provided key
    keyHash := HashAPIKey(key)
    
    // Look up in database
    var info APIKeyInfo
    err := db.QueryRow(`
        SELECT id, user_id, name, scopes, rate_limit, revoked, expires_at, last_used_at
        FROM api_keys
        WHERE key_hash = ?
    `, keyHash).Scan(
        &info.ID, &info.UserID, &info.Name, &info.Scopes,
        &info.RateLimit, &info.Revoked, &info.ExpiresAt, &info.LastUsedAt,
    )
    
    if err == sql.ErrNoRows {
        return nil, errors.New("invalid API key")
    }
    
    return &info, nil
}

func UpdateLastUsed(keyID int) {
    db.Exec("UPDATE api_keys SET last_used_at = ? WHERE id = ?", time.Now(), keyID)
}
```

---

## API Key Scopes

**Scopes** = Permissions for what the key can do

```go
type APIKeyInfo struct {
    Scopes []string
}

// Check scope
func (k *APIKeyInfo) HasScope(scope string) bool {
    for _, s := range k.Scopes {
        if s == scope {
            return true
        }
    }
    return false
}

// Scope middleware
func RequireScope(scope string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            keyInfo := r.Context().Value("api_key").(*APIKeyInfo)
            
            if !keyInfo.HasScope(scope) {
                http.Error(w, "Insufficient permissions", 403)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// Usage
r.Handle("/api/users", 
    RequireScope("users:read")(
        http.HandlerFunc(GetUsers),
    ),
)
```

**Example scopes:**
```
users:read
users:write
payments:read
payments:write
admin:*
```

---

## Rate Limiting per API Key

```go
func APIKeyRateLimiter(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        keyInfo := r.Context().Value("api_key").(*APIKeyInfo)
        
        // Rate limit key (e.g., using Redis)
        allowed, remaining, resetTime := CheckRateLimit(keyInfo.ID, keyInfo.RateLimit)
        
        // Set headers
        w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%d", keyInfo.RateLimit))
        w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%d", remaining))
        w.Header().Set("X-RateLimit-Reset", fmt.Sprintf("%d", resetTime.Unix()))
        
        if !allowed {
            http.Error(w, "Rate limit exceeded", 429)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}
```

---

## API Key Management Endpoints

### Create API Key

```go
func CreateAPIKeyHandler(w http.ResponseWriter, r *http.Request) {
    userID := getCurrentUserID(r)
    
    var req struct {
        Name      string   `json:"name"`
        Scopes    []string `json:"scopes"`
        ExpiresAt *time.Time `json:"expires_at,omitempty"`
    }
    
    json.NewDecoder(r.Body).Decode(&req)
    
    // Generate key
    key, err := CreateAPIKey(userID, req.Name, req.Scopes, req.ExpiresAt)
    if err != nil {
        http.Error(w, "Failed to create API key", 500)
        return
    }
    
    json.NewEncoder(w).Encode(map[string]string{
        "api_key": key,
        "message": "Save this key - it won't be shown again!",
    })
}
```

---

### List API Keys

```go
func ListAPIKeysHandler(w http.ResponseWriter, r *http.Request) {
    userID := getCurrentUserID(r)
    
    rows, _ := db.Query(`
        SELECT id, key_prefix, name, scopes, created_at, last_used_at, revoked
        FROM api_keys
        WHERE user_id = ?
        ORDER BY created_at DESC
    `, userID)
    
    keys := []map[string]interface{}{}
    for rows.Next() {
        var key map[string]interface{}
        // Scan...
        keys = append(keys, key)
    }
    
    json.NewEncoder(w).Encode(keys)
}
```

**Note:** Never return full key, only prefix!

---

### Revoke API Key

```go
func RevokeAPIKeyHandler(w http.ResponseWriter, r *http.Request) {
    userID := getCurrentUserID(r)
    keyID := mux.Vars(r)["id"]
    
    // Revoke key (only if belongs to user)
    result, err := db.Exec(`
        UPDATE api_keys
        SET revoked = TRUE
        WHERE id = ? AND user_id = ?
    `, keyID, userID)
    
    if err != nil {
        http.Error(w, "Failed to revoke key", 500)
        return
    }
    
    rows, _ := result.RowsAffected()
    if rows == 0 {
        http.Error(w, "API key not found", 404)
        return
    }
    
    w.WriteHeader(204)
}
```

---

## Security Best Practices

### 1. Never Log API Keys

```go
❌ log.Info("Request", "api_key", apiKey)
✅ log.Info("Request", "api_key_prefix", apiKey[:10]+"...")
```

---

### 2. Hash Keys in Database

```go
❌ Store plain: sk_live_4eC39HqLyjWDarjtT1
✅ Store hash: 2c26b46b68ffc68ff99b453c1d30413413422d706483bfa0f98a5e886266e7ae
```

---

### 3. Use HTTPS Only

```
❌ http://api.example.com?api_key=sk_live_...
✅ https://api.example.com
   Authorization: Bearer sk_live_...
```

---

### 4. Rotate Keys Regularly

```go
// Notify users to rotate keys
if time.Since(key.CreatedAt) > 365*24*time.Hour {
    // Send warning email
}
```

---

### 5. Set Expiration

```go
expiresAt := time.Now().Add(90 * 24 * time.Hour) // 90 days
```

---

### 6. Different Keys for Environments

```
sk_test_... → Development
sk_live_... → Production
```

---

### 7. Limit Scopes

```go
// Only grant necessary scopes
scopes := []string{"users:read"} // Not "admin:*"
```

---

## API Key vs Environment Variables

**For deployment secrets (database passwords, etc.):**
```bash
❌ API keys (meant for API clients)
✅ Environment variables
```

**For API clients:**
```bash
✅ API keys
❌ Environment variables
```

---

## Complete Example

```go
package main

import (
    "crypto/rand"
    "crypto/sha256"
    "database/sql"
    "encoding/base64"
    "encoding/hex"
    "encoding/json"
    "net/http"
    "time"
)

type APIKey struct {
    ID         int
    KeyHash    string
    KeyPrefix  string
    UserID     int
    Name       string
    Scopes     []string
    RateLimit  int
    CreatedAt  time.Time
    LastUsedAt *time.Time
    ExpiresAt  *time.Time
    Revoked    bool
}

func GenerateAPIKey(prefix string) (string, error) {
    bytes := make([]byte, 24)
    _, err := rand.Read(bytes)
    if err != nil {
        return "", err
    }
    
    encoded := base64.RawURLEncoding.EncodeToString(bytes)
    return prefix + "_" + encoded, nil
}

func HashAPIKey(key string) string {
    hash := sha256.Sum256([]byte(key))
    return hex.EncodeToString(hash[:])
}

func CreateAPIKey(db *sql.DB, userID int, name string, scopes []string) (string, error) {
    // Generate key
    key, err := GenerateAPIKey("sk_live")
    if err != nil {
        return "", err
    }
    
    // Hash key
    keyHash := HashAPIKey(key)
    
    // Store in database
    _, err = db.Exec(`
        INSERT INTO api_keys (key_hash, key_prefix, user_id, name, scopes, rate_limit)
        VALUES (?, ?, ?, ?, ?, ?)
    `, keyHash, "sk_live", userID, name, scopes, 1000)
    
    return key, err
}

func ValidateAPIKey(db *sql.DB, key string) (*APIKey, error) {
    keyHash := HashAPIKey(key)
    
    var apiKey APIKey
    err := db.QueryRow(`
        SELECT id, user_id, name, scopes, rate_limit, revoked, expires_at
        FROM api_keys
        WHERE key_hash = ?
    `, keyHash).Scan(
        &apiKey.ID, &apiKey.UserID, &apiKey.Name,
        &apiKey.Scopes, &apiKey.RateLimit, &apiKey.Revoked,
        &apiKey.ExpiresAt,
    )
    
    if err == sql.ErrNoRows {
        return nil, errors.New("invalid API key")
    }
    
    if apiKey.Revoked {
        return nil, errors.New("API key revoked")
    }
    
    if apiKey.ExpiresAt != nil && time.Now().After(*apiKey.ExpiresAt) {
        return nil, errors.New("API key expired")
    }
    
    // Update last used
    db.Exec("UPDATE api_keys SET last_used_at = ? WHERE id = ?", time.Now(), apiKey.ID)
    
    return &apiKey, nil
}

func APIKeyMiddleware(db *sql.DB) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            apiKey := r.Header.Get("X-API-Key")
            if apiKey == "" {
                http.Error(w, "Missing API key", 401)
                return
            }
            
            keyInfo, err := ValidateAPIKey(db, apiKey)
            if err != nil {
                http.Error(w, err.Error(), 401)
                return
            }
            
            ctx := context.WithValue(r.Context(), "api_key", keyInfo)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. API keys should be __________ before storing in database.
2. API keys should only be transmitted over __________.
3. The __________ prefix indicates a production secret key in Stripe.
4. API keys are used for __________ identification, not user authentication.

### True/False

1. ⬜ API keys should be stored in plain text
2. ⬜ API keys should be logged for debugging
3. ⬜ API keys can be used for user authentication
4. ⬜ Different keys should be used for test and production
5. ⬜ Full API keys should be shown in the dashboard

### Multiple Choice

1. How should API keys be stored?
   - A) Plain text
   - B) Encrypted
   - C) Hashed
   - D) Base64 encoded

2. Where should API keys be sent?
   - A) URL query parameter
   - B) Request body
   - C) HTTP header
   - D) Cookie

3. What's the purpose of API key scopes?
   - A) Rate limiting
   - B) Permissions
   - C) Expiration
   - D) Identification

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. hashed
2. HTTPS
3. sk_live
4. application (or client)

**True/False:**
1. ❌ False - Must be hashed
2. ❌ False - Never log full keys (only prefix)
3. ❌ False - API keys identify apps, not users
4. ✅ True - Separate test/production keys
5. ❌ False - Show only prefix (sk_live_...)

**Multiple Choice:**
1. **C** - Hashed (like passwords)
2. **C** - HTTP header (X-API-Key or Authorization)
3. **B** - Permissions (what the key can do)

</details>

</details>