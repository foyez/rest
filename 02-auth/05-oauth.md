# 2.5 OAuth 2.0

## What is OAuth 2.0?

**OAuth 2.0** = Authorization framework that allows third-party applications to access user resources **without sharing passwords**.

**Not authentication** - OAuth is for **authorization** ("what can you access"), not authentication ("who are you").

---

## Real-World Example

**Scenario:** App wants to access your Google Drive

```
Without OAuth (❌ Bad):
User gives app their Google password
App logs into Google as the user
App can do ANYTHING

With OAuth (✅ Good):
User authorizes app via Google
Google gives app a limited access token
App can only access what user approved
User can revoke access anytime
```

---

## OAuth Roles

| Role | Description | Example |
|------|-------------|---------|
| **Resource Owner** | User who owns the data | You |
| **Client** | App requesting access | Mobile app |
| **Authorization Server** | Issues tokens | Google OAuth |
| **Resource Server** | Hosts protected resources | Google Drive API |

---

## OAuth Flow (Authorization Code)

```
┌─────────┐          ┌─────────┐          ┌─────────┐          ┌──────────┐
│  User   │          │   App   │          │  Google │          │ Google   │
│         │          │ (Client)│          │  Auth   │          │ Drive    │
└────┬────┘          └────┬────┘          └────┬────┘          └────┬─────┘
     │                    │                    │                    │
     │ 1. Click "Connect  │                    │                    │
     │    Google Drive"   │                    │                    │
     │ ─────────────────> │                    │                    │
     │                    │                    │                    │
     │                    │ 2. Redirect to Google                   │
     │                    │ ────────────────────────────────────>   │
     │                    │                    │                    │
     │ 3. Login to Google                      │                    │
     │ ─────────────────────────────────────>  │                    │
     │                    │                    │                    │
     │ 4. "Allow app to   │                    │                    │
     │    access Drive?"  │                    │                    │
     │ <──────────────────────────────────────│                    │
     │                    │                    │                    │
     │ 5. Approve         │                    │                    │
     │ ────────────────────────────────────>   │                    │
     │                    │                    │                    │
     │ 6. Redirect back   │                    │                    │
     │    with auth code  │                    │                    │
     │ <──────────────────────────────────────│                    │
     │                    │                    │                    │
     │ 7. Pass code to app                     │                    │
     │ ─────────────────> │                    │                    │
     │                    │ 8. Exchange code   │                    │
     │                    │    for token       │                    │
     │                    │ ────────────────>  │                    │
     │                    │                    │                    │
     │                    │ 9. Access token    │                    │
     │                    │ <────────────────  │                    │
     │                    │                    │                    │
     │                    │ 10. Access Drive                        │
     │                    │ ─────────────────────────────────────>  │
     │                    │                    │                    │
     │                    │ 11. Files          │                    │
     │                    │ <──────────────────────────────────────│
```

---

## Authorization Code Flow (Most Common)

**Step-by-step:**

**1. User clicks "Sign in with Google"**

App redirects to:
```
https://accounts.google.com/o/oauth2/v2/auth?
  client_id=YOUR_CLIENT_ID&
  redirect_uri=https://yourapp.com/callback&
  response_type=code&
  scope=https://www.googleapis.com/auth/drive.readonly&
  state=random_string_for_security
```

**2. User logs in and approves**

**3. Google redirects back with authorization code**

```
https://yourapp.com/callback?
  code=4/0AX4XfWh...&
  state=random_string_for_security
```

**4. App exchanges code for token**

```http
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

code=4/0AX4XfWh...&
client_id=YOUR_CLIENT_ID&
client_secret=YOUR_SECRET&
redirect_uri=https://yourapp.com/callback&
grant_type=authorization_code
```

**5. Response with access token**

```json
{
  "access_token": "ya29.a0AfH6SMBx...",
  "expires_in": 3600,
  "refresh_token": "1//0eL4z...",
  "scope": "https://www.googleapis.com/auth/drive.readonly",
  "token_type": "Bearer"
}
```

**6. Use access token to call API**

```http
GET https://www.googleapis.com/drive/v3/files
Authorization: Bearer ya29.a0AfH6SMBx...
```

---

## OAuth Grant Types

### 1. Authorization Code ✅ Most Secure

**Use:** Web apps with backend

**Flow:** User approves → Get code → Exchange for token

**Security:** Client secret never exposed to browser

---

### 2. Implicit (Deprecated)

**Use:** Browser-only apps (legacy)

**Flow:** User approves → Get token directly

**Why deprecated:** Token exposed in URL, less secure

---

### 3. Client Credentials

**Use:** Machine-to-machine (no user)

**Flow:** App authenticates directly

```http
POST /token
grant_type=client_credentials&
client_id=YOUR_ID&
client_secret=YOUR_SECRET
```

---

### 4. Password Grant (Not Recommended)

**Use:** First-party apps only

**Flow:** App collects username/password

**Why avoid:** Defeats OAuth purpose

---

## Scopes

**Scopes** define what access token can do.

**Examples:**

```
Google:
- email
- profile
- https://www.googleapis.com/auth/drive.readonly
- https://www.googleapis.com/auth/drive.file

GitHub:
- repo
- user
- read:org

Facebook:
- public_profile
- email
- user_friends
```

**Request specific scopes:**
```
scope=email profile https://www.googleapis.com/auth/drive.readonly
```

---

## Implementation Example (Go)

```go
package main

import (
    "encoding/json"
    "net/http"
    "net/url"
)

const (
    googleAuthURL  = "https://accounts.google.com/o/oauth2/v2/auth"
    googleTokenURL = "https://oauth2.googleapis.com/token"
    clientID       = "YOUR_CLIENT_ID"
    clientSecret   = "YOUR_CLIENT_SECRET"
    redirectURI    = "http://localhost:8080/callback"
)

// 1. Redirect to Google
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    params := url.Values{}
    params.Add("client_id", clientID)
    params.Add("redirect_uri", redirectURI)
    params.Add("response_type", "code")
    params.Add("scope", "email profile")
    params.Add("state", generateRandomState())
    
    authURL := googleAuthURL + "?" + params.Encode()
    http.Redirect(w, r, authURL, http.StatusTemporaryRedirect)
}

// 2. Handle callback
func CallbackHandler(w http.ResponseWriter, r *http.Request) {
    code := r.URL.Query().Get("code")
    state := r.URL.Query().Get("state")
    
    // Verify state (CSRF protection)
    if !verifyState(state) {
        http.Error(w, "Invalid state", 400)
        return
    }
    
    // Exchange code for token
    token, err := exchangeCodeForToken(code)
    if err != nil {
        http.Error(w, "Failed to get token", 500)
        return
    }
    
    // Get user info
    user, err := getUserInfo(token.AccessToken)
    if err != nil {
        http.Error(w, "Failed to get user", 500)
        return
    }
    
    json.NewEncoder(w).Encode(user)
}

// Exchange authorization code for access token
func exchangeCodeForToken(code string) (*TokenResponse, error) {
    data := url.Values{}
    data.Set("code", code)
    data.Set("client_id", clientID)
    data.Set("client_secret", clientSecret)
    data.Set("redirect_uri", redirectURI)
    data.Set("grant_type", "authorization_code")
    
    resp, err := http.PostForm(googleTokenURL, data)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var token TokenResponse
    json.NewDecoder(resp.Body).Decode(&token)
    
    return &token, nil
}

// Get user info from Google
func getUserInfo(accessToken string) (*UserInfo, error) {
    req, _ := http.NewRequest("GET", "https://www.googleapis.com/oauth2/v2/userinfo", nil)
    req.Header.Set("Authorization", "Bearer "+accessToken)
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var user UserInfo
    json.NewDecoder(resp.Body).Decode(&user)
    
    return &user, nil
}

type TokenResponse struct {
    AccessToken  string `json:"access_token"`
    ExpiresIn    int    `json:"expires_in"`
    RefreshToken string `json:"refresh_token"`
    Scope        string `json:"scope"`
    TokenType    string `json:"token_type"`
}

type UserInfo struct {
    ID      string `json:"id"`
    Email   string `json:"email"`
    Name    string `json:"name"`
    Picture string `json:"picture"`
}
```

---

## OAuth vs OpenID Connect

| Feature | OAuth 2.0 | OpenID Connect |
|---------|-----------|----------------|
| Purpose | Authorization | Authentication |
| Returns | Access token | Access token + ID token |
| Use case | Access APIs | Sign in |
| Identity | ❌ No | ✅ Yes |

**OAuth 2.0:**
```
"Can this app access my Google Drive?"
```

**OpenID Connect (OIDC):**
```
"Sign in with Google"
Returns: Who the user is (email, name)
```

---

## Security Best Practices

### 1. Use State Parameter (CSRF Protection)

```go
// Generate random state
state := generateRandomString(32)
storeState(sessionID, state)

// In callback, verify state
if r.URL.Query().Get("state") != getStoredState(sessionID) {
    return errors.New("CSRF detected")
}
```

### 2. Validate Redirect URI

```go
// Only allow whitelisted redirect URIs
allowedURIs := []string{
    "http://localhost:8080/callback",
    "https://yourapp.com/callback",
}
```

### 3. Store Client Secret Securely

```go
❌ clientSecret := "hardcoded-secret"
✅ clientSecret := os.Getenv("OAUTH_CLIENT_SECRET")
```

### 4. Use HTTPS

```
❌ http://yourapp.com/callback
✅ https://yourapp.com/callback
```

### 5. Request Minimum Scopes

```
❌ scope=email profile drive calendar contacts
✅ scope=email profile
```

---

## Common OAuth Providers

| Provider | Auth URL | Token URL |
|----------|----------|-----------|
| Google | https://accounts.google.com/o/oauth2/v2/auth | https://oauth2.googleapis.com/token |
| GitHub | https://github.com/login/oauth/authorize | https://github.com/login/oauth/access_token |
| Facebook | https://www.facebook.com/v12.0/dialog/oauth | https://graph.facebook.com/v12.0/oauth/access_token |
| Microsoft | https://login.microsoftonline.com/common/oauth2/v2.0/authorize | https://login.microsoftonline.com/common/oauth2/v2.0/token |

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. OAuth 2.0 is for __________, not authentication.
2. The most secure OAuth grant type for web apps is __________.
3. The __________ parameter prevents CSRF attacks in OAuth.
4. OAuth tokens have limited __________ that define what they can access.

### True/False

1. ⬜ OAuth 2.0 is an authentication protocol
2. ⬜ Client secret should be exposed to the browser
3. ⬜ The state parameter is optional but recommended
4. ⬜ Authorization code should be exchanged on the backend
5. ⬜ OAuth tokens grant unlimited access to user data

### Multiple Choice

1. What is the purpose of OAuth 2.0?
   - A) User authentication
   - B) Password storage
   - C) Delegated authorization
   - D) Data encryption

2. Which grant type is most secure for web apps?
   - A) Implicit
   - B) Password
   - C) Authorization Code
   - D) Client Credentials

3. Where should you exchange authorization code for token?
   - A) Frontend/Browser
   - B) Backend server
   - C) Mobile app
   - D) Doesn't matter

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. authorization
2. Authorization Code
3. state
4. scopes

**True/False:**
1. ❌ False - OAuth is for authorization, OIDC is for authentication
2. ❌ False - Client secret must be kept on backend only
3. ❌ False - State parameter is required for security (CSRF protection)
4. ✅ True - Backend exchange prevents secret exposure
5. ❌ False - Tokens are limited by scopes

**Multiple Choice:**
1. **C** - Delegated authorization (let apps access resources)
2. **C** - Authorization Code (uses backend, keeps secret safe)
3. **B** - Backend server (keeps client secret safe)

</details>

</details>