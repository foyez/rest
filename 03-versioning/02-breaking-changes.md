# 3.2 Breaking vs Non-Breaking Changes

## What is a Breaking Change?

**Breaking change** = Change that breaks existing clients

```
Client (v1.0) expects:
{ "user_type": "admin" }

API changed to:
{ "role": "admin" }

Client breaks! ❌
```

---

## Breaking Changes

### 1. Removing a Field

```go
// v1
type User struct {
    ID       int    `json:"id"`
    UserType string `json:"user_type"`
}

// v2 - BREAKING
type User struct {
    ID int `json:"id"`
    // user_type removed
}
```

**Impact:** Clients expecting `user_type` will fail.

---

### 2. Renaming a Field

```go
// v1
type User struct {
    UserType string `json:"user_type"`
}

// v2 - BREAKING
type User struct {
    Role string `json:"role"`  // Renamed
}
```

**Impact:** Clients reading `user_type` get nothing.

---

### 3. Changing Field Type

```go
// v1
type Product struct {
    Price int `json:"price"`  // Integer
}

// v2 - BREAKING
type Product struct {
    Price float64 `json:"price"`  // Float
}
```

**Impact:** Integer-expecting clients may fail parsing.

---

### 4. Changing Response Structure

```go
// v1
{
  "id": 1,
  "name": "John"
}

// v2 - BREAKING
{
  "user": {
    "id": 1,
    "name": "John"
  }
}
```

**Impact:** Clients accessing top-level fields fail.

---

### 5. Removing an Endpoint

```
// v1
GET /api/users/{id}

// v2 - BREAKING
// Endpoint removed
```

**Impact:** Clients get 404.

---

### 6. Changing Required Fields

```go
// v1
POST /users
{
  "name": "John"
}

// v2 - BREAKING
POST /users
{
  "name": "John",
  "email": "required@example.com"  // Now required
}
```

**Impact:** Old clients missing email get 422.

---

### 7. Changing Status Codes

```
// v1
DELETE /users/123
→ 200 OK

// v2 - BREAKING
DELETE /users/123
→ 204 No Content
```

**Impact:** Clients expecting 200 may fail.

---

### 8. Changing Error Format

```json
// v1
{
  "error": "User not found"
}

// v2 - BREAKING
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found"
  }
}
```

**Impact:** Clients parsing `error` string fail.

---

### 9. Changing Authentication

```
// v1
Basic Auth

// v2 - BREAKING
Bearer Token (JWT)
```

**Impact:** Old authentication stops working.

---

### 10. Removing Query Parameter Support

```
// v1
GET /users?status=active

// v2 - BREAKING
// status parameter no longer supported
```

**Impact:** Filtering stops working.

---

## Non-Breaking Changes

### 1. Adding New Endpoint ✅

```
// v1
GET /users

// v1 (updated)
GET /users
POST /users  // New endpoint
```

**Safe:** Existing clients unaffected.

---

### 2. Adding Optional Field ✅

```go
// v1
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

// v1 (updated)
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Phone string `json:"phone,omitempty"`  // Optional
}
```

**Safe:** Old clients ignore new field.

---

### 3. Adding New Query Parameter ✅

```
// v1
GET /users

// v1 (updated)
GET /users?sort=name  // New optional parameter
```

**Safe:** Old clients don't use it.

---

### 4. Adding New HTTP Method ✅

```
// v1
GET /posts/{id}

// v1 (updated)
GET /posts/{id}
PATCH /posts/{id}  // New method
```

**Safe:** Existing GET still works.

---

### 5. Relaxing Validation ✅

```go
// v1
Minimum password length: 8 characters

// v1 (updated)
Minimum password length: 6 characters  // More permissive
```

**Safe:** Old clients already meet requirement.

---

### 6. Adding More Permissive Values ✅

```go
// v1
enum Status {
    "active",
    "inactive"
}

// v1 (updated)
enum Status {
    "active",
    "inactive",
    "pending"  // New value
}
```

**Safe:** Old clients use existing values.

---

### 7. Improving Performance ✅

```
Same API contract
Faster response
```

**Safe:** Clients benefit.

---

### 8. Bug Fixes ✅

```
Fixed incorrect calculation
Fixed typo in error message
```

**Safe:** Correct behavior.

---

## Decision Matrix

| Change | Breaking? | Example |
|--------|-----------|---------|
| Remove field | ✅ Yes | Remove `user_type` |
| Rename field | ✅ Yes | `user_type` → `role` |
| Change type | ✅ Yes | `int` → `float` |
| Make field required | ✅ Yes | `email` now required |
| Change structure | ✅ Yes | Wrap in `user: {}` |
| Remove endpoint | ✅ Yes | DELETE `/old-endpoint` |
| Change status code | ✅ Yes | 200 → 204 |
| Add field (optional) | ❌ No | Add `phone` |
| Add endpoint | ❌ No | POST `/users` |
| Add query param | ❌ No | `?sort=name` |
| Relax validation | ❌ No | Min length 8 → 6 |

---

## Handling Breaking Changes

### Option 1: New Version ✅ Recommended

```
v1: { "user_type": "admin" }
v2: { "role": "admin" }
```

Support both versions simultaneously.

---

### Option 2: Deprecation Period

```
Step 1: Add new field, keep old
{
  "user_type": "admin",  // Deprecated
  "role": "admin"        // New
}

Step 2 (6 months): Mark as deprecated
Deprecation: true
Sunset: 2025-12-31

Step 3 (1 year): Remove old field
{
  "role": "admin"
}
```

---

### Option 3: Expansion Strategy

**Add before removing:**

```
// Phase 1: Support both
{
  "user_type": "admin",
  "role": "admin"
}

// Phase 2: Clients migrate

// Phase 3: Remove old field
{
  "role": "admin"
}
```

---

## Making Safe Changes

### Adding Optional Fields

```go
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`  // omitempty for optional
}
```

**Client behavior:**
```javascript
// Old client
const user = { id: 1, name: "John" }
// Works fine, ignores email

// New client
const user = { id: 1, name: "John", email: "john@example.com" }
// Can use email
```

---

### Versioning Fields (Alternative)

```go
type User struct {
    ID       int    `json:"id"`
    UserType string `json:"user_type,omitempty"`  // Deprecated
    Role     string `json:"role,omitempty"`       // New
}

// Populate both for backward compatibility
user.UserType = "admin"
user.Role = "admin"
```

---

## Communicating Changes

### 1. Changelog

```markdown
# API Changelog

## v2.0.0 (2025-01-15) - BREAKING CHANGES

### Breaking Changes
- **BREAKING:** Removed `user_type` field
  - Use `role` field instead
  - Migration: Replace `user_type` with `role` in your code

- **BREAKING:** Changed `/users` response structure
  - Data now wrapped in `users` array
  - Old: `[{...}]`
  - New: `{ "users": [{...}] }`

### Migration Guide
```javascript
// Before (v1)
const users = await fetch('/api/v1/users').json()
users.forEach(user => console.log(user.user_type))

// After (v2)
const response = await fetch('/api/v2/users').json()
response.users.forEach(user => console.log(user.role))
```

## v1.5.0 (2024-12-01) - Non-breaking

### Added
- New `phone` field (optional)
- New `GET /users/search` endpoint

### Deprecated
- `user_type` field (use `role` instead)
```

---

### 2. Deprecation Headers

```http
GET /api/v1/users

Deprecation: true
Sunset: Sat, 31 Dec 2025 23:59:59 GMT
Link: </api/v2/users>; rel="successor-version"
```

---

### 3. In-Response Warnings

```json
{
  "data": [...],
  "warnings": [
    {
      "code": "DEPRECATED_FIELD",
      "message": "Field 'user_type' is deprecated. Use 'role' instead.",
      "field": "user_type",
      "sunset_date": "2025-12-31"
    }
  ]
}
```

---

## Real-World Example: Stripe

Stripe uses **date-based API versions** to handle breaking changes:

```
API Version: 2024-11-20
```

**Example breaking change:**

**2023-08-16:**
```json
{
  "type": "card",
  "card": {...}
}
```

**2024-11-20:**
```json
{
  "type": "card",
  "payment_method_details": {
    "card": {...}
  }
}
```

Clients specify version:
```http
POST /v1/charges
Stripe-Version: 2023-08-16
```

Old clients continue working with old version.

---

## Testing for Breaking Changes

### Contract Testing

```go
func TestBackwardCompatibility(t *testing.T) {
    // v1 response
    v1Response := `{"id":1,"user_type":"admin"}`
    
    // Parse with v1 struct
    var user UserV1
    json.Unmarshal([]byte(v1Response), &user)
    
    assert.Equal(t, "admin", user.UserType)
}
```

---

### Schema Validation

```yaml
# OpenAPI Schema
User:
  type: object
  required:
    - id
    - name
  properties:
    id:
      type: integer
    name:
      type: string
    phone:
      type: string  # Optional - safe to add
```

Use tools to detect breaking changes:
- `oasdiff` - Compare OpenAPI specs
- `optic` - API contract testing

---

## Migration Checklist

**Before making breaking changes:**

- [ ] Is there a non-breaking alternative?
- [ ] Can we support both old and new?
- [ ] Document the change
- [ ] Create migration guide
- [ ] Set deprecation timeline
- [ ] Add deprecation warnings
- [ ] Notify API consumers
- [ ] Create new API version
- [ ] Support old version for 6-12 months
- [ ] Monitor usage of old version
- [ ] Sunset old version

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Renaming a field from `user_type` to `role` is a __________ change.
2. Adding an optional field is a __________ change.
3. Before removing a deprecated field, you should give a __________ period.
4. Making a field __________ is a breaking change.

### True/False

1. ⬜ Adding a new endpoint is a breaking change
2. ⬜ Changing field type from int to float is non-breaking
3. ⬜ Removing a field requires a new API version
4. ⬜ Adding an optional query parameter is breaking
5. ⬜ Relaxing validation rules is a breaking change

### Multiple Choice

1. Which is a breaking change?
   - A) Adding new optional field
   - B) Adding new endpoint
   - C) Removing a field
   - D) Adding query parameter

2. What should you do before making a breaking change?
   - A) Just deploy it
   - B) Create new version
   - C) Delete old version
   - D) Change existing version

3. How long should you support deprecated APIs?
   - A) 1 week
   - B) 1 month
   - C) 6-12 months
   - D) Forever

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. breaking
2. non-breaking
3. deprecation (or migration)
4. required

**True/False:**
1. ❌ False - Adding endpoints is non-breaking
2. ❌ False - Type changes are breaking
3. ✅ True - Requires new version or deprecation period
4. ❌ False - Optional parameters are non-breaking
5. ❌ False - Relaxing validation is non-breaking

**Multiple Choice:**
1. **C** - Removing fields breaks existing clients
2. **B** - Create new API version
3. **C** - 6-12 months is industry standard

</details>

</details>