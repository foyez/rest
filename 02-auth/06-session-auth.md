# 2.6 Session-Based Authentication

## What is Session-Based Auth?

**Concept:** Server stores user state

```
1. User logs in
2. Server creates session
3. Server stores session in database/memory
4. Server sends session ID in cookie
5. Client sends cookie with each request
6. Server looks up session
```

---

## How It Works

```
┌─────────┐                          ┌─────────┐                    ┌──────────┐
│ Client  │                          │ Server  │                    │ Database │
└────┬────┘                          └────┬────┘                    └────┬─────┘
     │                                    │                              │
     │ 1. POST /login                     │                              │
     │    {email, password}               │                              │
     │ ────────────────────────────────>  │                              │
     │                                    │                              │
     │                                2. Validate credentials            │
     │                                3. Generate session ID             │
     │                                4. Store session ──────────────>   │
     │                                    │   {session_id, user_id,      │
     │                                    │    expires_at, ...}          │
     │                                    │                              │
     │ 5. Set-Cookie: session_id=abc123   │                              │
     │ <──────────────────────────────────│                              │
     │                                    │                              │
     │ 6. Store cookie                    │                              │
     │                                    │                              │
     │ 7. GET /api/profile                │                              │
     │    Cookie: session_id=abc123       │                              │
     │ ────────────────────────────────>  │                              │
     │                                    │                              │
     │                                8. Lookup session ────────────>    │
     │                                    │ <──────────────────────────  │
     │                                    │   {user_id: 123, ...}        │
     │                                9. Get user data ──────────────>   │
     │                                    │ <──────────────────────────  │
     │                                    │   {name: "John", ...}        │
     │                                    │                              │
     │ 10. {user: {...}}                  │                              │
     │ <──────────────────────────────────│                              │
```

---

## Session Storage

### Option 1: In-Memory (Development Only)

```go
var sessions = make(map[string]Session)

type Session struct {
    UserID    int
    CreatedAt time.Time
    ExpiresAt time.Time
}
```

**Pros:** Fast
**Cons:** 
- Lost on restart
- Doesn't scale (single server only)

---

### Option 2: Database

```sql
CREATE TABLE sessions (
    id VARCHAR(255) PRIMARY KEY,
    user_id INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);
```

**Pros:** Persistent, can query
**Cons:** Slower than memory/Redis

---

### Option 3: Redis ✅ Recommended

```redis
SET session:abc123 '{"user_id":123,"created_at":"2025-01-16T10:00:00Z"}'
EXPIRE session:abc123 3600
```

**Pros:** 
- Fast (in-memory)
- Persistent
- Built-in expiration
- Scales horizontally

**Cons:** External dependency

---

## Implementation (Go)

### Session Manager

```go
package main

import (
    "crypto/rand"
    "encoding/base64"
    "encoding/json"
    "time"
    
    "github.com/go-redis/redis/v8"
)

type SessionManager struct {
    redis  *redis.Client
    expiry time.Duration
}

type Session struct {
    UserID    int       `json:"user_id"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

func NewSessionManager(redisClient *redis.Client, expiry time.Duration) *SessionManager {
    return &SessionManager{
        redis:  redisClient,
        expiry: expiry,
    }
}

// Generate session ID
func generateSessionID() (string, error) {
    bytes := make([]byte, 32)
    _, err := rand.Read(bytes)
    if err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(bytes), nil
}

// Create session
func (sm *SessionManager) Create(userID int, email string) (string, error) {
    sessionID, err := generateSessionID()
    if err != nil {
        return "", err
    }
    
    session := Session{
        UserID:    userID,
        Email:     email,
        CreatedAt: time.Now(),
    }
    
    data, _ := json.Marshal(session)
    
    err = sm.redis.Set(ctx, "session:"+sessionID, data, sm.expiry).Err()
    if err != nil {
        return "", err
    }
    
    return sessionID, nil
}

// Get session
func (sm *SessionManager) Get(sessionID string) (*Session, error) {
    data, err := sm.redis.Get(ctx, "session:"+sessionID).Result()
    if err == redis.Nil {
        return nil, errors.New("session not found")
    }
    if err != nil {
        return nil, err
    }
    
    var session Session
    json.Unmarshal([]byte(data), &session)
    
    return &session, nil
}

// Delete session (logout)
func (sm *SessionManager) Delete(sessionID string) error {
    return sm.redis.Del(ctx, "session:"+sessionID).Err()
}

// Refresh session expiry
func (sm *SessionManager) Refresh(sessionID string) error {
    return sm.redis.Expire(ctx, "session:"+sessionID, sm.expiry).Err()
}
```

---

### Login Handler

```go
func LoginHandler(w http.ResponseWriter, r *http.Request, sm *SessionManager) {
    var req struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    
    json.NewDecoder(r.Body).Decode(&req)
    
    // Validate credentials
    user, err := ValidateCredentials(req.Email, req.Password)
    if err != nil {
        http.Error(w, "Invalid email or password", 401)
        return
    }
    
    // Create session
    sessionID, err := sm.Create(user.ID, user.Email)
    if err != nil {
        http.Error(w, "Failed to create session", 500)
        return
    }
    
    // Set cookie
    http.SetCookie(w, &http.Cookie{
        Name:     "session_id",
        Value:    sessionID,
        Path:     "/",
        MaxAge:   3600,           // 1 hour
        HttpOnly: true,           // Prevent JavaScript access
        Secure:   true,           // HTTPS only
        SameSite: http.SameSiteStrictMode,
    })
    
    json.NewEncoder(w).Encode(map[string]interface{}{
        "message": "Login successful",
        "user":    user,
    })
}
```

---

### Auth Middleware

```go
func SessionAuthMiddleware(sm *SessionManager) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Get session cookie
            cookie, err := r.Cookie("session_id")
            if err != nil {
                http.Error(w, "Unauthorized", 401)
                return
            }
            
            // Get session
            session, err := sm.Get(cookie.Value)
            if err != nil {
                http.Error(w, "Invalid session", 401)
                return
            }
            
            // Refresh session expiry
            sm.Refresh(cookie.Value)
            
            // Add session to context
            ctx := context.WithValue(r.Context(), "session", session)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// Protected route
func ProfileHandler(w http.ResponseWriter, r *http.Request) {
    session := r.Context().Value("session").(*Session)
    
    user, _ := GetUserByID(session.UserID)
    
    json.NewEncoder(w).Encode(user)
}
```

---

### Logout Handler

```go
func LogoutHandler(w http.ResponseWriter, r *http.Request, sm *SessionManager) {
    cookie, err := r.Cookie("session_id")
    if err != nil {
        http.Error(w, "No session found", 400)
        return
    }
    
    // Delete session
    sm.Delete(cookie.Value)
    
    // Clear cookie
    http.SetCookie(w, &http.Cookie{
        Name:     "session_id",
        Value:    "",
        Path:     "/",
        MaxAge:   -1,  // Delete cookie
        HttpOnly: true,
        Secure:   true,
    })
    
    w.WriteHeader(204)
}
```

---

## Session vs JWT

| Feature | Session | JWT |
|---------|---------|-----|
| **Storage** | Server-side | Client-side |
| **Stateless** | ❌ No | ✅ Yes |
| **Revocation** | ✅ Easy | ❌ Hard |
| **Scalability** | ⚠️ Needs sticky sessions | ✅ Easy |
| **Size** | Small (just ID) | Large (entire payload) |
| **Security** | ✅ Can't tamper | ⚠️ Can decode |
| **Best for** | Web apps | APIs, microservices |

---

## Session Security

### 1. Secure Cookie Flags

```go
http.SetCookie(w, &http.Cookie{
    Name:     "session_id",
    Value:    sessionID,
    HttpOnly: true,  // ✅ Prevent XSS
    Secure:   true,  // ✅ HTTPS only
    SameSite: http.SameSiteStrictMode,  // ✅ CSRF protection
    Path:     "/",
    MaxAge:   3600,
})
```

---

### 2. Session Fixation Prevention

**Attack:** Attacker sets victim's session ID

**Prevention:** Regenerate session ID on login

```go
func RegenerateSession(oldSessionID string, sm *SessionManager) (string, error) {
    // Get old session
    session, err := sm.Get(oldSessionID)
    if err != nil {
        return "", err
    }
    
    // Delete old session
    sm.Delete(oldSessionID)
    
    // Create new session with same data
    newSessionID, err := sm.Create(session.UserID, session.Email)
    if err != nil {
        return "", err
    }
    
    return newSessionID, nil
}
```

---

### 3. Session Timeout

```go
const SessionTimeout = 30 * time.Minute

// Auto-logout after 30 minutes
func (sm *SessionManager) Create(userID int) (string, error) {
    sessionID, _ := generateSessionID()
    
    // Set expiry
    sm.redis.Set(ctx, "session:"+sessionID, data, SessionTimeout)
    
    return sessionID, nil
}
```

---

### 4. Absolute Timeout

```go
type Session struct {
    UserID      int
    CreatedAt   time.Time
    LastAccess  time.Time
    MaxLifetime time.Duration
}

func (sm *SessionManager) Get(sessionID string) (*Session, error) {
    session, _ := sm.getFromRedis(sessionID)
    
    // Check absolute timeout (e.g., 24 hours max)
    if time.Since(session.CreatedAt) > session.MaxLifetime {
        sm.Delete(sessionID)
        return nil, errors.New("session expired")
    }
    
    // Update last access
    session.LastAccess = time.Now()
    sm.save(sessionID, session)
    
    return session, nil
}
```

---

### 5. IP Address Binding

```go
type Session struct {
    UserID    int
    IPAddress string
}

func (sm *SessionManager) Create(userID int, ip string) (string, error) {
    session := Session{
        UserID:    userID,
        IPAddress: ip,
    }
    // Store...
}

func (sm *SessionManager) Validate(sessionID, currentIP string) error {
    session, _ := sm.Get(sessionID)
    
    if session.IPAddress != currentIP {
        return errors.New("IP address mismatch")
    }
    
    return nil
}
```

---

### 6. User Agent Binding

```go
type Session struct {
    UserID    int
    UserAgent string
}

func (sm *SessionManager) Validate(sessionID, currentUA string) error {
    session, _ := sm.Get(sessionID)
    
    if session.UserAgent != currentUA {
        return errors.New("user agent mismatch")
    }
    
    return nil
}
```

---

## Multiple Sessions Per User

```go
// Allow multiple devices
func (sm *SessionManager) Create(userID int) (string, error) {
    sessionID, _ := generateSessionID()
    
    // Store with user prefix
    sm.redis.Set(ctx, "session:"+sessionID, data, expiry)
    
    // Track user's sessions
    sm.redis.SAdd(ctx, fmt.Sprintf("user:%d:sessions", userID), sessionID)
    
    return sessionID, nil
}

// Logout from all devices
func (sm *SessionManager) LogoutAllDevices(userID int) error {
    // Get all sessions for user
    sessionIDs, _ := sm.redis.SMembers(ctx, fmt.Sprintf("user:%d:sessions", userID)).Result()
    
    // Delete all sessions
    for _, sessionID := range sessionIDs {
        sm.Delete(sessionID)
    }
    
    // Clear tracking set
    sm.redis.Del(ctx, fmt.Sprintf("user:%d:sessions", userID))
    
    return nil
}

// Get active sessions
func (sm *SessionManager) GetActiveSessions(userID int) ([]Session, error) {
    sessionIDs, _ := sm.redis.SMembers(ctx, fmt.Sprintf("user:%d:sessions", userID)).Result()
    
    sessions := []Session{}
    for _, sessionID := range sessionIDs {
        session, err := sm.Get(sessionID)
        if err == nil {
            sessions = append(sessions, *session)
        }
    }
    
    return sessions, nil
}
```

---

## Session Storage with Additional Data

```go
type Session struct {
    UserID     int                    `json:"user_id"`
    Email      string                 `json:"email"`
    Role       string                 `json:"role"`
    Metadata   map[string]interface{} `json:"metadata"`
    CreatedAt  time.Time              `json:"created_at"`
    LastAccess time.Time              `json:"last_access"`
    IPAddress  string                 `json:"ip_address"`
    UserAgent  string                 `json:"user_agent"`
}

// Store shopping cart in session
session.Metadata["cart"] = []int{1, 2, 3}

// Store preferences
session.Metadata["theme"] = "dark"
session.Metadata["language"] = "en"
```

---

## Remember Me Feature

```go
func LoginHandler(w http.ResponseWriter, r *http.Request, sm *SessionManager) {
    var req struct {
        Email      string `json:"email"`
        Password   string `json:"password"`
        RememberMe bool   `json:"remember_me"`
    }
    
    json.NewDecoder(r.Body).Decode(&req)
    
    user, _ := ValidateCredentials(req.Email, req.Password)
    
    // Different expiry based on remember me
    var expiry time.Duration
    if req.RememberMe {
        expiry = 30 * 24 * time.Hour  // 30 days
    } else {
        expiry = 1 * time.Hour  // 1 hour
    }
    
    sessionID, _ := sm.CreateWithExpiry(user.ID, expiry)
    
    http.SetCookie(w, &http.Cookie{
        Name:     "session_id",
        Value:    sessionID,
        MaxAge:   int(expiry.Seconds()),
        HttpOnly: true,
        Secure:   true,
    })
}
```

---

## Complete Example

```go
package main

import (
    "context"
    "crypto/rand"
    "encoding/base64"
    "encoding/json"
    "net/http"
    "time"
    
    "github.com/go-redis/redis/v8"
    "github.com/gorilla/mux"
)

var ctx = context.Background()

type SessionManager struct {
    redis *redis.Client
}

type Session struct {
    UserID    int       `json:"user_id"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

func main() {
    // Redis client
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    
    sm := &SessionManager{redis: rdb}
    
    r := mux.NewRouter()
    
    // Public routes
    r.HandleFunc("/login", func(w http.ResponseWriter, r *http.Request) {
        LoginHandler(w, r, sm)
    }).Methods("POST")
    
    r.HandleFunc("/logout", func(w http.ResponseWriter, r *http.Request) {
        LogoutHandler(w, r, sm)
    }).Methods("POST")
    
    // Protected routes
    api := r.PathPrefix("/api").Subrouter()
    api.Use(SessionAuthMiddleware(sm))
    api.HandleFunc("/profile", ProfileHandler).Methods("GET")
    
    http.ListenAndServe(":8080", r)
}

func (sm *SessionManager) Create(userID int, email string) (string, error) {
    sessionID := generateSessionID()
    
    session := Session{
        UserID:    userID,
        Email:     email,
        CreatedAt: time.Now(),
    }
    
    data, _ := json.Marshal(session)
    sm.redis.Set(ctx, "session:"+sessionID, data, 1*time.Hour)
    
    return sessionID, nil
}

func (sm *SessionManager) Get(sessionID string) (*Session, error) {
    data, err := sm.redis.Get(ctx, "session:"+sessionID).Result()
    if err != nil {
        return nil, err
    }
    
    var session Session
    json.Unmarshal([]byte(data), &session)
    
    return &session, nil
}

func (sm *SessionManager) Delete(sessionID string) error {
    return sm.redis.Del(ctx, "session:"+sessionID).Err()
}

func generateSessionID() string {
    bytes := make([]byte, 32)
    rand.Read(bytes)
    return base64.URLEncoding.EncodeToString(bytes)
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Sessions are stored __________ (client-side/server-side).
2. Session cookies should have the __________ flag to prevent XSS.
3. To prevent session fixation, __________ the session ID on login.
4. Redis is preferred for session storage because it has built-in __________.

### True/False

1. ⬜ Sessions are stateless
2. ⬜ Session IDs should be stored in localStorage
3. ⬜ HttpOnly cookies prevent JavaScript access
4. ⬜ Sessions can be easily revoked
5. ⬜ Session-based auth scales better than JWT

### Multiple Choice

1. Where should sessions be stored in production?
   - A) In-memory map
   - B) Redis
   - C) localStorage
   - D) URL parameters

2. Which cookie flag prevents CSRF?
   - A) HttpOnly
   - B) Secure
   - C) SameSite
   - D) Path

3. What's the main advantage of session-based auth?
   - A) Stateless
   - B) Easy revocation
   - C) No server storage
   - D) Works without cookies

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. server-side
2. HttpOnly
3. regenerate
4. expiration

**True/False:**
1. ❌ False - Sessions are stateful (stored on server)
2. ❌ False - Should be in HttpOnly cookies
3. ✅ True - HttpOnly prevents JavaScript from accessing cookies
4. ✅ True - Just delete from server storage
5. ❌ False - JWT scales better (stateless)

**Multiple Choice:**
1. **B** - Redis (fast, persistent, built-in expiration)
2. **C** - SameSite prevents CSRF attacks
3. **B** - Easy revocation (delete from server)

</details>

</details>