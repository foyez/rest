# 2.1 Authentication vs Authorization

## Core Concepts

### Authentication (AuthN)
**"Who are you?"**

- Verifying **identity**
- Proving you are who you claim to be
- Like showing your ID at airport

### Authorization (AuthZ)
**"What can you do?"**

- Verifying **permissions**
- What resources you can access
- Like your boarding pass showing which gate/seat

---

## Visual Comparison

```
┌──────────────────────────────────────────────────────┐
│                    Authentication                    │
│              "Prove who you are"                     │
│                                                      │
│  Username + Password  ──>  Token (JWT)              │
│       OR                                             │
│  Email + OTP         ──>   Session                  │
│       OR                                             │
│  Fingerprint         ──>   Verified Identity        │
└──────────────────────────────────────────────────────┘

         ↓ After successful authentication ↓

┌──────────────────────────────────────────────────────┐
│                    Authorization                      │
│            "What can you access?"                    │
│                                                      │
│  User Token  ──>  Check Permissions  ──>  Allow/Deny │
│                                                      │
│  Examples:                                           │
│  • Can this user edit this post?                    │
│  • Can this user access admin panel?                │
│  • Can this user delete this comment?               │
└──────────────────────────────────────────────────────┘
```

---

## Real-World Example: Office Building

### Authentication
```
You: "I'm John from accounting"
Guard: "Show me your ID badge"
You: *shows badge*
Guard: "ID verified. You're John. Enter."
```

### Authorization
```
You: *tries to enter server room*
System: "John is verified, but doesn't have server room access"
System: "Access DENIED"

You: *tries to enter accounting floor*
System: "John has accounting floor access"
System: "Access GRANTED"
```

---

## Authentication Methods

### 1. Basic Authentication

```http
GET /api/users
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

**How it works:**
```
1. Client sends: username:password
2. Base64 encoded: "username:password" → "dXNlcm5hbWU6cGFzc3dvcmQ="
3. Server decodes and validates
```

**Pros:**
- Simple
- Built into HTTP

**Cons:**
- ⚠️ **Not secure** (base64 is encoding, not encryption)
- Must use HTTPS
- Credentials sent with every request

**When to use:** Internal tools, local development only

---

### 2. Token-Based (JWT)

```http
GET /api/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Flow:**
```
1. User logs in with credentials
   POST /auth/login
   { "email": "john@example.com", "password": "secret" }

2. Server validates and returns token
   { "token": "eyJhbGc..." }

3. Client stores token (localStorage/memory)

4. Client sends token with each request
   Authorization: Bearer eyJhbGc...

5. Server validates token on each request
```

**Pros:**
- Stateless (no server-side session)
- Scalable
- Works across domains

**Cons:**
- Can't revoke easily
- Token size larger than session ID

---

### 3. Session-Based

```http
GET /api/users
Cookie: session_id=abc123xyz
```

**Flow:**
```
1. User logs in
2. Server creates session, stores in database/Redis
3. Server sends session ID in cookie
4. Browser automatically sends cookie with requests
5. Server looks up session to verify user
```

**Pros:**
- Can revoke easily (delete session)
- Smaller cookies

**Cons:**
- Requires server-side storage
- Doesn't work well across domains
- Not RESTful (stateful)

---

### 4. OAuth 2.0

**Delegation of access**

```
User wants app to access their Google Drive

1. App redirects to Google login
2. User logs in to Google
3. Google asks: "Allow App to access your Drive?"
4. User approves
5. Google gives app an access token
6. App uses token to access Drive on user's behalf
```

**Use cases:**
- "Sign in with Google"
- "Connect to Facebook"
- Third-party app integrations

---

## Authorization Methods

### 1. Role-Based Access Control (RBAC)

Users have **roles**, roles have **permissions**

```
Roles:
  • admin → can do everything
  • editor → can create/edit posts
  • viewer → can only read

User: John
Role: editor
Can: create posts, edit posts
Cannot: delete users, view analytics
```

**Example:**
```go
func DeletePost(userId, postId int) error {
    user := GetUser(userId)
    
    // Check role
    if user.Role != "admin" && user.Role != "editor" {
        return ErrForbidden // 403
    }
    
    // Check ownership (additional check)
    post := GetPost(postId)
    if post.AuthorId != userId && user.Role != "admin" {
        return ErrForbidden // 403
    }
    
    return DeletePostById(postId)
}
```

---

### 2. Attribute-Based Access Control (ABAC)

Permissions based on **attributes** (user, resource, environment)

```
Rule: Allow if (
    user.department == resource.department AND
    user.clearance_level >= resource.required_level AND
    current_time between 9am-5pm
)
```

**Example:**
```
User: John (department=engineering, clearance=3)
Resource: Document (department=engineering, required=2)
Time: 2pm
Result: ALLOWED

User: Jane (department=marketing, clearance=4)
Resource: Same document
Time: 2pm
Result: DENIED (wrong department)
```

---

### 3. Permission-Based

Fine-grained permissions assigned directly

```
User: John
Permissions:
  • posts.create
  • posts.edit
  • posts.delete (own only)
  • comments.moderate
```

**Code check:**
```go
if !user.HasPermission("posts.delete") {
    return ErrForbidden
}
```

---

## HTTP Status Codes

| Scenario | Code | Example |
|----------|------|---------|
| Not authenticated | 401 | No token provided |
| Invalid credentials | 401 | Wrong password |
| Expired token | 401 | Token expired |
| Valid but no permission | 403 | User trying admin endpoint |
| Resource doesn't exist | 404 | User not found |

---

## Authentication Flow Example (JWT)

```
┌─────────┐                  ┌─────────┐
│ Client  │                  │ Server  │
└────┬────┘                  └────┬────┘
     │                            │
     │ 1. POST /auth/login        │
     │    { email, password }     │
     │ ───────────────────────>   │
     │                            │
     │                        2. Validate
     │                        3. Generate JWT
     │                            │
     │ 4. { token: "..." }        │
     │ <───────────────────────   │
     │                            │
     │ 5. Store token             │
     │                            │
     │ 6. GET /api/posts          │
     │    Authorization: Bearer..│
     │ ───────────────────────>   │
     │                            │
     │                        7. Verify token
     │                        8. Extract user
     │                        9. Check permission
     │                            │
     │ 10. { data: [...] }        │
     │ <───────────────────────   │
```

---

## Authorization Flow Example (RBAC)

```
Request: DELETE /posts/123
Token: valid JWT (userId=456, role=editor)

1. Authenticate:
   ✓ Token is valid
   ✓ User is authenticated (userId=456)

2. Authorize:
   a. Check role:
      User role = editor
      Required role for DELETE = editor or admin
      ✓ Role check passed
   
   b. Check ownership:
      Post 123 author = 456
      Current user = 456
      ✓ Ownership check passed

3. Execute:
   Delete post 123
   Return 204 No Content
```

---

## Common Mistakes

### ❌ Confusing 401 and 403

```go
// Wrong
if user == nil {
    return 403 // Should be 401
}
if !user.IsAdmin() {
    return 401 // Should be 403
}

// Correct
if user == nil {
    return 401 // Not authenticated
}
if !user.IsAdmin() {
    return 403 // Not authorized
}
```

---

### ❌ Client-Side Only Authorization

```javascript
// ❌ BAD: Only checking in frontend
if (user.role === 'admin') {
    showDeleteButton()
}
```

**Always validate on server:**
```go
// ✅ GOOD: Server validation
func DeleteUser(requestUser, targetUserId int) error {
    user := GetUser(requestUser)
    if user.Role != "admin" {
        return ErrForbidden // 403
    }
    return Delete(targetUserId)
}
```

---

### ❌ Exposing Too Much Info

```json
❌ {
  "error": "User john@example.com doesn't exist"
}
```

This reveals if email exists in system (security issue).

```json
✅ {
  "error": "Invalid email or password"
}
```

Generic message doesn't leak info.

---

## Best Practices

1. ✅ **Always use HTTPS** for auth endpoints
2. ✅ **Validate on server**, never trust client
3. ✅ **Use 401 for authentication**, 403 for authorization
4. ✅ **Hash passwords** (never store plain text)
5. ✅ **Use rate limiting** on login endpoints
6. ✅ **Implement token expiration**
7. ✅ **Log authentication failures**
8. ✅ **Use refresh tokens** for long-lived access

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. __________ verifies who you are, while __________ verifies what you can do.
2. HTTP status code __________ means not authenticated, while __________ means not authorized.
3. In RBAC, users have __________ which have __________.
4. JWT stands for __________ __________ __________.

### True/False

1. ⬜ Authentication and authorization are the same thing
2. ⬜ 401 means "you don't have permission"
3. ⬜ Basic authentication is secure without HTTPS
4. ⬜ Session-based auth requires server-side storage
5. ⬜ Client-side authorization checks are sufficient

### Multiple Choice

1. User tries to access admin panel with a valid user token. What status code?
   - A) 400
   - B) 401
   - C) 403
   - D) 404

2. Which authentication method is stateless?
   - A) Session-based
   - B) JWT
   - C) Cookies
   - D) Basic Auth

3. In RBAC, what has permissions?
   - A) Users directly
   - B) Roles
   - C) Resources
   - D) Tokens

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. Authentication, Authorization
2. 401, 403
3. roles, permissions
4. JSON Web Token

**True/False:**
1. ❌ False - They are different; authentication = identity, authorization = permissions
2. ❌ False - 401 = not authenticated; 403 = not authorized
3. ❌ False - Basic auth is NOT secure without HTTPS (base64 encoding is not encryption)
4. ✅ True - Sessions must be stored server-side (database/Redis)
5. ❌ False - Must always validate on server; client checks are for UX only

**Multiple Choice:**
1. **C** - 403 Forbidden (authenticated but not authorized)
2. **B** - JWT is stateless (no server-side storage needed)
3. **B** - Roles have permissions; users have roles

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between authentication and authorization with a real-world example

### Question 2: When should you return 401 vs 403? Give examples of each.

### Question 3: Compare JWT vs Session-based authentication. When would you use each?

### Question 4: What is RBAC? Implement a simple permission check.

### Question 5: How would you handle "Sign in with Google"? Explain the OAuth flow.

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

### Answer 1: Authentication vs Authorization

**Authentication = WHO you are**
**Authorization = WHAT you can do**

**Real-world example: Hospital**

**Authentication (Entrance):**
```
You: "I'm Dr. Smith"
Guard: "Show me your employee badge"
You: *scans badge*
System: ✓ Badge verified
         ✓ Dr. Smith confirmed
Guard: "Welcome, Dr. Smith"
```

**Authorization (Access to areas):**
```
Dr. Smith tries to enter:

Operating Room:
  Check: Is Dr. Smith a surgeon?
  Result: Yes ✓
  Access: GRANTED

Pharmacy Storage:
  Check: Is Dr. Smith a pharmacist?
  Result: No ✗
  Access: DENIED

Cafeteria:
  Check: Is Dr. Smith an employee?
  Result: Yes ✓
  Access: GRANTED
```

**API example:**

```
Authentication:
POST /auth/login
{ "email": "john@example.com", "password": "secret" }

Response:
{ "token": "eyJhbGc..." }  ← User is authenticated

Authorization:
GET /admin/users
Authorization: Bearer eyJhbGc...

Server checks:
✓ Token valid (authenticated)
✗ User role = "user" (not "admin")
Response: 403 Forbidden (not authorized)
```

---

### Answer 2: When to use 401 vs 403

**401 Unauthorized = Authentication problem**
- "I don't know who you are"
- "Prove your identity"

**403 Forbidden = Authorization problem**
- "I know who you are"
- "But you're not allowed"

**Examples:**

**401 scenarios:**
```go
// 1. No token provided
GET /api/posts
// No Authorization header
→ 401 Unauthorized

// 2. Invalid token
GET /api/posts
Authorization: Bearer invalid-token
→ 401 Unauthorized

// 3. Expired token
GET /api/posts
Authorization: Bearer expired-token-abc
→ 401 Unauthorized

// 4. Wrong password
POST /auth/login
{ "email": "john@example.com", "password": "wrong" }
→ 401 Unauthorized
```

**403 scenarios:**
```go
// 1. Non-admin accessing admin route
GET /admin/settings
Authorization: Bearer valid-user-token
→ 403 Forbidden

// 2. Editing someone else's resource
PATCH /posts/123  (owned by user 456)
Authorization: Bearer user-789-token
→ 403 Forbidden

// 3. Suspended account
GET /api/data
Authorization: Bearer valid-but-suspended-user
→ 403 Forbidden

// 4. IP not whitelisted
GET /api/admin
Authorization: Bearer valid-admin-token
X-Forwarded-For: 1.2.3.4 (not whitelisted)
→ 403 Forbidden
```

**Implementation:**
```go
func GetUserProfile(authToken string, userId int) error {
    // Authentication
    user, err := ValidateToken(authToken)
    if err != nil {
        return ErrUnauthorized // 401
    }
    
    // Authorization
    if user.ID != userId && !user.IsAdmin() {
        return ErrForbidden // 403
    }
    
    return GetProfile(userId)
}
```

---

### Answer 3: JWT vs Session-based

**JWT (Token-Based)**

```
┌─────────┐                ┌─────────┐
│ Client  │                │ Server  │
└────┬────┘                └────┬────┘
     │                          │
     │ 1. Login                 │
     │ ──────────────>          │
     │                      2. Generate JWT
     │                          │
     │ 3. Token (JWT)           │
     │ <──────────────          │
     │                          │
     │ 4. Store in memory       │
     │    or localStorage       │
     │                          │
     │ 5. Request + Token       │
     │ ──────────────>          │
     │                      6. Validate JWT
     │                      7. Extract user from token
     │                          │
     │ 8. Response              │
     │ <──────────────          │
```

**Pros:**
- ✅ Stateless (no server-side storage)
- ✅ Scalable (works with load balancers)
- ✅ Works across domains (CORS)
- ✅ Good for microservices
- ✅ Mobile-friendly

**Cons:**
- ❌ Can't revoke easily (until expiry)
- ❌ Larger payload (more bandwidth)
- ❌ Token exposed if stolen (XSS risk)

---

**Session-Based**

```
┌─────────┐                ┌─────────┐      ┌──────────┐
│ Client  │                │ Server  │      │  Redis   │
└────┬────┘                └────┬────┘      └────┬─────┘
     │                          │                │
     │ 1. Login                 │                │
     │ ──────────────>          │                │
     │                      2. Create session    │
     │                      3. Store ──────────> │
     │                          │                │
     │ 4. Set-Cookie:           │                │
     │    session_id=abc        │                │
     │ <──────────────          │                │
     │                          │                │
     │ 5. Request               │                │
     │    Cookie: session_id    │                │
     │ ──────────────>          │                │
     │                      6. Lookup session ──>│
     │                          │ <──────────────│
     │                      7. Validate user     │
     │                          │                │
     │ 8. Response              │                │
     │ <──────────────          │                │
```

**Pros:**
- ✅ Easy to revoke (delete session)
- ✅ Smaller cookies (just session ID)
- ✅ More secure (token on server)
- ✅ Better for sensitive data

**Cons:**
- ❌ Requires server-side storage (Redis/DB)
- ❌ Harder to scale (sticky sessions)
- ❌ CORS issues (cookies don't cross domains easily)
- ❌ Not RESTful (stateful)

---

**When to use each:**

| Use Case | Choice | Reason |
|----------|--------|--------|
| Mobile app | JWT | No cookies |
| Microservices | JWT | Stateless scaling |
| Traditional web app | Session | Easy revocation |
| API for third parties | JWT | Cross-domain |
| Banking/financial | Session | More control |
| Real-time revocation needed | Session | Immediate logout |
| Simple REST API | JWT | Stateless |

**Hybrid approach:**
```
1. Short-lived JWT (15 min)
2. Refresh token (stored server-side like session)
3. Best of both worlds
```

---

### Answer 4: RBAC Implementation

**Role-Based Access Control:**

Users → Roles → Permissions

**Example structure:**
```go
type User struct {
    ID    int
    Email string
    Role  string // "admin", "editor", "viewer"
}

type Permission struct {
    Action   string // "create", "read", "update", "delete"
    Resource string // "posts", "users", "comments"
}

var RolePermissions = map[string][]Permission{
    "admin": {
        {"create", "posts"},
        {"read", "posts"},
        {"update", "posts"},
        {"delete", "posts"},
        {"create", "users"},
        {"read", "users"},
        {"update", "users"},
        {"delete", "users"},
    },
    "editor": {
        {"create", "posts"},
        {"read", "posts"},
        {"update", "posts"}, // Own posts only
        {"delete", "posts"}, // Own posts only
    },
    "viewer": {
        {"read", "posts"},
    },
}
```

**Permission check implementation:**
```go
func HasPermission(user *User, action, resource string) bool {
    permissions, ok := RolePermissions[user.Role]
    if !ok {
        return false
    }
    
    for _, perm := range permissions {
        if perm.Action == action && perm.Resource == resource {
            return true
        }
    }
    
    return false
}

// Usage in handlers
func DeletePost(w http.ResponseWriter, r *http.Request) {
    user := GetAuthenticatedUser(r) // From JWT
    if user == nil {
        http.Error(w, "Unauthorized", 401)
        return
    }
    
    // Check permission
    if !HasPermission(user, "delete", "posts") {
        http.Error(w, "Forbidden", 403)
        return
    }
    
    postID := getPostID(r)
    post := GetPost(postID)
    
    // Additional check: own posts only (unless admin)
    if user.Role != "admin" && post.AuthorID != user.ID {
        http.Error(w, "Forbidden", 403)
        return
    }
    
    DeletePostByID(postID)
    w.WriteHeader(204)
}
```

**Middleware for route protection:**
```go
func RequireRole(roles ...string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            user := GetAuthenticatedUser(r)
            if user == nil {
                http.Error(w, "Unauthorized", 401)
                return
            }
            
            allowed := false
            for _, role := range roles {
                if user.Role == role {
                    allowed = true
                    break
                }
            }
            
            if !allowed {
                http.Error(w, "Forbidden", 403)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

// Usage
r.Handle("/admin/users", RequireRole("admin")(adminHandler))
r.Handle("/posts", RequireRole("admin", "editor")(postsHandler))
```

---

### Answer 5: OAuth 2.0 "Sign in with Google" Flow

**High-level flow:**

```
┌─────────┐         ┌──────────┐         ┌─────────┐
│ Your App│         │  Google  │         │  User   │
└────┬────┘         └────┬─────┘         └────┬────┘
     │                   │                    │
     │ 1. Redirect to Google                 │
     │ ────────────────────────────────────> │
     │                   │                    │
     │                   │ 2. Show login page │
     │                   │ <────────────────  │
     │                   │                    │
     │                   │ 3. User logs in    │
     │                   │ <────────────────  │
     │                   │                    │
     │                   │ 4. Ask permission  │
     │                   │ ────────────────>  │
     │                   │                    │
     │                   │ 5. User approves   │
     │                   │ <────────────────  │
     │                   │                    │
     │ 6. Redirect with code                 │
     │ <─────────────────────────────────────│
     │                   │                    │
     │ 7. Exchange code for token            │
     │ ────────────────> │                    │
     │                   │                    │
     │ 8. Access token   │                    │
     │ <──────────────── │                    │
     │                   │                    │
     │ 9. Get user info  │                    │
     │ ────────────────> │                    │
     │                   │                    │
     │ 10. User data     │                    │
     │ <──────────────── │                    │
```

**Step-by-step:**

**1. User clicks "Sign in with Google"**
```
App redirects to:
https://accounts.google.com/o/oauth2/v2/auth?
  client_id=YOUR_CLIENT_ID&
  redirect_uri=https://yourapp.com/auth/callback&
  response_type=code&
  scope=email profile&
  state=random_string_for_security
```

**2. User logs into Google**
- Google shows login page
- User enters credentials

**3. Google asks for permissions**
```
"YourApp wants to:
 - View your email address
 - View your basic profile info
 
 Allow / Deny"
```

**4. User approves**

**5. Google redirects back to your app**
```
https://yourapp.com/auth/callback?
  code=4/0AX4XfWh...&
  state=random_string_for_security
```

**6. Your app exchanges code for access token**
```http
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

code=4/0AX4XfWh...&
client_id=YOUR_CLIENT_ID&
client_secret=YOUR_CLIENT_SECRET&
redirect_uri=https://yourapp.com/auth/callback&
grant_type=authorization_code
```

**7. Google responds with tokens**
```json
{
  "access_token": "ya29.a0AfH6SMBx...",
  "expires_in": 3600,
  "refresh_token": "1//0eL4...",
  "token_type": "Bearer"
}
```

**8. Use access token to get user info**
```http
GET https://www.googleapis.com/oauth2/v1/userinfo
Authorization: Bearer ya29.a0AfH6SMBx...
```

**9. Google returns user data**
```json
{
  "id": "1234567890",
  "email": "user@gmail.com",
  "name": "John Doe",
  "picture": "https://..."
}
```

**10. Your app creates/logs in user**
```go
// Check if user exists
user := GetUserByGoogleID(googleData.ID)

if user == nil {
    // Create new user
    user = CreateUser{
        Email: googleData.Email,
        Name: googleData.Name,
        GoogleID: googleData.ID,
        Avatar: googleData.Picture,
    }
}

// Generate your app's JWT
token := GenerateJWT(user.ID)

// Return to client
return {
    "token": token,
    "user": user
}
```

**Why OAuth instead of username/password?**

1. ✅ More secure (Google handles password)
2. ✅ No password management
3. ✅ User trust (familiar login)
4. ✅ Single sign-on
5. ✅ Google handles 2FA

</details>

</details>