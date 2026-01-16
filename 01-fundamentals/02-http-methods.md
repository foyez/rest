# 1.2 HTTP Methods

## Overview

HTTP methods define the **action** to perform on a resource.

| Method | Purpose | Safe? | Idempotent? | Has Body? |
|--------|---------|-------|-------------|-----------|
| GET | Retrieve data | ✅ | ✅ | No |
| POST | Create resource | ❌ | ❌ | Yes |
| PUT | Replace resource | ❌ | ✅ | Yes |
| PATCH | Update partial | ❌ | ❌* | Yes |
| DELETE | Remove resource | ❌ | ✅ | No |
| HEAD | Get headers only | ✅ | ✅ | No |
| OPTIONS | Get allowed methods | ✅ | ✅ | No |

\* PATCH can be designed to be idempotent

---

## Key Concepts

### Safe Methods
- **Don't modify** server state
- Read-only operations
- Examples: GET, HEAD, OPTIONS

### Idempotent Methods
- Making the **same request multiple times** produces the **same result**
- Examples: GET, PUT, DELETE

**Example:**
```
DELETE /users/123  -> 204 No Content (user deleted)
DELETE /users/123  -> 404 Not Found (already deleted)
```
Same result: user 123 doesn't exist

---

## GET - Retrieve Data

**Purpose:** Fetch data without modifying it

**Characteristics:**
- ✅ Safe (no side effects)
- ✅ Idempotent
- ✅ Can be cached
- ❌ Should not have request body

### Examples

#### Get Single Resource
```http
GET /api/v1/users/123
```

**Response:**
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com"
}
```

#### Get Collection
```http
GET /api/v1/products?category=electronics&limit=20
```

**Response:**
```json
{
  "data": [
    { "id": 1, "name": "Laptop", "price": 999 },
    { "id": 2, "name": "Mouse", "price": 25 }
  ],
  "total": 150,
  "page": 1,
  "limit": 20
}
```

### Common Use Cases
- View user profile
- List products
- Search results
- Dashboard data

---

## POST - Create Resource

**Purpose:** Create a new resource

**Characteristics:**
- ❌ Not safe (modifies state)
- ❌ Not idempotent (creates new resource each time)
- ✅ Has request body

### Examples

#### Create User
```http
POST /api/v1/users
Content-Type: application/json

{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "password": "securepass123"
}
```

**Response:**
```http
201 Created
Location: /api/v1/users/124

{
  "id": 124,
  "name": "Jane Doe",
  "email": "jane@example.com",
  "created_at": "2025-01-16T10:30:00Z"
}
```

#### Add Comment to Post
```http
POST /api/v1/posts/456/comments

{
  "text": "Great article!",
  "author_id": 123
}
```

**Response:**
```http
201 Created

{
  "id": 789,
  "post_id": 456,
  "text": "Great article!",
  "author_id": 123,
  "created_at": "2025-01-16T10:35:00Z"
}
```

### Why POST is Not Idempotent

```
POST /api/v1/orders { "product_id": 1, "qty": 2 }
-> Creates order #101

POST /api/v1/orders { "product_id": 1, "qty": 2 }
-> Creates order #102 (different result!)
```

---

## PUT - Replace Resource

**Purpose:** Replace the entire resource

**Characteristics:**
- ❌ Not safe
- ✅ Idempotent
- ✅ Has request body
- ⚠️ Client must send **all fields**

### Example

**Current resource:**
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30,
  "city": "New York"
}
```

**PUT Request:**
```http
PUT /api/v1/users/123

{
  "name": "John Smith",
  "email": "john.smith@example.com"
}
```

**Result:**
```json
{
  "id": 123,
  "name": "John Smith",
  "email": "john.smith@example.com"
  // age and city are REMOVED
}
```

### Why PUT is Idempotent

```
PUT /api/v1/users/123 { "name": "John", "email": "john@ex.com" }
-> User 123 updated

PUT /api/v1/users/123 { "name": "John", "email": "john@ex.com" }
-> User 123 has same data (same result)
```

---

## PATCH - Partial Update

**Purpose:** Update only specific fields

**Characteristics:**
- ❌ Not safe
- ⚠️ Can be idempotent (depends on implementation)
- ✅ Has request body
- ✅ Client sends only fields to change

### Example

**Current resource:**
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30,
  "city": "New York"
}
```

**PATCH Request:**
```http
PATCH /api/v1/users/123

{
  "email": "newemail@example.com"
}
```

**Result:**
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "newemail@example.com",  // Updated
  "age": 30,                         // Unchanged
  "city": "New York"                 // Unchanged
}
```

### PUT vs PATCH Comparison

| Scenario | PUT | PATCH |
|----------|-----|-------|
| Update email only | Must send all fields | Send only email |
| Missing fields | Removed/reset | Unchanged |
| Idempotent | ✅ Always | ⚠️ Depends |
| Common in REST | Less common | ✅ Preferred |

**Real-world preference:** Most APIs use **PATCH** for updates because it's safer.

---

## DELETE - Remove Resource

**Purpose:** Delete a resource

**Characteristics:**
- ❌ Not safe
- ✅ Idempotent
- ❌ Usually no request body
- ✅ Response often 204 No Content

### Examples

#### Delete User
```http
DELETE /api/v1/users/123
```

**First request:**
```http
204 No Content
```

**Second request (same user):**
```http
404 Not Found

{
  "error": "User not found"
}
```

#### Soft Delete Pattern

Instead of permanently deleting:

```http
PATCH /api/v1/users/123

{
  "deleted_at": "2025-01-16T10:40:00Z",
  "is_active": false
}
```

Benefits:
- Data recovery possible
- Audit trail preserved
- Maintains referential integrity

---

## HEAD - Get Metadata

**Purpose:** Get response headers without body

**Use cases:**
- Check if resource exists
- Get file size before downloading
- Check last modified time

### Example

```http
HEAD /api/v1/users/123
```

**Response:**
```http
200 OK
Content-Type: application/json
Content-Length: 256
Last-Modified: Wed, 15 Jan 2025 09:00:00 GMT
```

No body returned, but you know:
- Resource exists (200)
- Size is 256 bytes
- Last updated time

---

## OPTIONS - Allowed Methods

**Purpose:** Check what HTTP methods are allowed

**Use cases:**
- CORS preflight requests
- API discovery

### Example

```http
OPTIONS /api/v1/users/123
```

**Response:**
```http
200 OK
Allow: GET, PUT, PATCH, DELETE
Access-Control-Allow-Methods: GET, PUT, PATCH, DELETE
Access-Control-Allow-Origin: https://example.com
```

---

## Real-World Example: Blog API

```
┌─────────────────────────────────────────────────────┐
│                    Blog API                         │
├─────────────────────────────────────────────────────┤
│ GET    /posts              List all posts           │
│ POST   /posts              Create new post          │
│ GET    /posts/123          Get specific post        │
│ PUT    /posts/123          Replace entire post      │
│ PATCH  /posts/123          Update post fields       │
│ DELETE /posts/123          Delete post              │
│                                                      │
│ GET    /posts/123/comments List comments            │
│ POST   /posts/123/comments Add comment              │
│ DELETE /comments/456       Delete specific comment  │
└─────────────────────────────────────────────────────┘
```

### Workflow Example

```
1. User creates post:
   POST /posts { "title": "My Post", "content": "..." }
   -> 201 Created, id: 123

2. User views their post:
   GET /posts/123
   -> 200 OK

3. User fixes typo:
   PATCH /posts/123 { "title": "My Corrected Post" }
   -> 200 OK

4. Someone adds comment:
   POST /posts/123/comments { "text": "Nice!" }
   -> 201 Created

5. User deletes post:
   DELETE /posts/123
   -> 204 No Content
```

---

## Method Selection Guide

| Scenario | Use This Method |
|----------|----------------|
| View product catalog | GET /products |
| Search users | GET /users?q=john |
| Register new user | POST /users |
| Place order | POST /orders |
| Update email only | PATCH /users/123 |
| Update entire profile | PUT /users/123 |
| Remove item from cart | DELETE /cart/items/5 |
| Check if file exists | HEAD /files/doc.pdf |
| CORS preflight | OPTIONS /api/endpoint |

---

## Common Mistakes

### ❌ Using GET with body
```http
GET /users
{
  "filter": "active"
}
```
✅ Use query parameters instead:
```http
GET /users?status=active
```

### ❌ Using POST for updates
```http
POST /users/123/update
```
✅ Use PATCH or PUT:
```http
PATCH /users/123
```

### ❌ Returning different status codes for same operation
```http
DELETE /users/123 -> 200 OK
DELETE /users/124 -> 204 No Content
```
✅ Be consistent:
```http
DELETE /users/* -> Always 204 No Content
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. __________ methods don't modify server state and are read-only.
2. An __________ method produces the same result when called multiple times.
3. The HTTP method __________ is used to create a new resource.
4. __________ updates only specific fields of a resource, while __________ replaces the entire resource.

### True/False

1. ⬜ GET requests can have a request body
2. ⬜ POST is idempotent
3. ⬜ DELETE is safe but not idempotent
4. ⬜ PUT is idempotent
5. ⬜ PATCH should be used instead of PUT for partial updates

### Multiple Choice

1. Which method should you use to create a new blog post?
   - A) GET
   - B) POST
   - C) PUT
   - D) PATCH

2. What happens if you send a PUT request with missing fields?
   - A) Error is returned
   - B) Missing fields are filled with defaults
   - C) Missing fields are removed from the resource
   - D) Nothing happens

3. Which methods are idempotent?
   - A) GET, POST, DELETE
   - B) GET, PUT, DELETE
   - C) POST, PUT, PATCH
   - D) Only GET

### Code Challenge

Given this user resource:
```json
{
  "id": 100,
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin",
  "age": 28
}
```

What will be the result after this request?
```http
PATCH /users/100
{
  "name": "Alice Smith"
}
```

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. Safe
2. idempotent
3. POST
4. PATCH, PUT

**True/False:**
1. ❌ False - GET requests should not have a body (not standard)
2. ❌ False - POST creates new resources each time, not idempotent
3. ❌ False - DELETE is idempotent but NOT safe (it modifies state)
4. ✅ True - Calling PUT with same data produces same result
5. ✅ True - PATCH is preferred for partial updates

**Multiple Choice:**
1. **B** - POST is used to create new resources
2. **C** - PUT replaces the entire resource, missing fields are removed
3. **B** - GET, PUT, DELETE are idempotent; POST and PATCH typically aren't

**Code Challenge:**
```json
{
  "id": 100,
  "name": "Alice Smith",        // Updated
  "email": "alice@example.com",  // Unchanged
  "role": "admin",               // Unchanged
  "age": 28                      // Unchanged
}
```
Only the `name` field is updated; other fields remain the same.

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between safe and idempotent methods. Give examples.

### Question 2: Why is POST not idempotent? How would you make a POST operation idempotent?

### Question 3: When would you use PUT vs PATCH? Give a real-world example.

### Question 4: A client keeps calling DELETE on the same resource. Is this a problem? Why or why not?

### Question 5: Should GET requests have a request body? Why or why not?

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

### Answer 1: Safe vs Idempotent

**Safe Methods:**
- Don't modify server state
- Read-only operations
- Can be called repeatedly without side effects
- Examples: GET, HEAD, OPTIONS

**Idempotent Methods:**
- Calling the same request multiple times produces the same final state
- May modify state, but repeated calls don't change it further
- Examples: GET, PUT, DELETE, HEAD, OPTIONS

**Key Difference:**
```
Safe      -> Doesn't modify state at all
Idempotent -> Can modify state, but repeated calls have same effect
```

**Example:**
```
GET /users/123          -> Safe & Idempotent (read only)
PUT /users/123 {...}    -> Idempotent but NOT safe (modifies state)
DELETE /users/123       -> Idempotent but NOT safe (modifies state)
POST /orders {...}      -> NOT safe, NOT idempotent (creates each time)
```

---

### Answer 2: Why POST is not idempotent & how to make it idempotent

**Why POST is not idempotent:**

Each POST creates a new resource:
```
POST /orders { "product": "Laptop", "qty": 1 }
-> Order #101 created

POST /orders { "product": "Laptop", "qty": 1 }
-> Order #102 created (different result!)
```

**How to make POST idempotent:**

**Method 1: Idempotency Key**
```http
POST /orders
Idempotency-Key: unique-client-generated-id-123

{
  "product": "Laptop",
  "qty": 1
}
```

Server logic:
```go
func CreateOrder(idempotencyKey string, order Order) {
    // Check if this key was already processed
    if existingOrder := db.GetByIdempotencyKey(idempotencyKey); existingOrder != nil {
        return existingOrder // Return existing, don't create new
    }
    
    // Create new order
    newOrder := db.Create(order)
    db.SaveIdempotencyKey(idempotencyKey, newOrder.ID)
    return newOrder
}
```

**Method 2: Client-Generated ID**
```http
PUT /orders/client-generated-uuid-123

{
  "product": "Laptop",
  "qty": 1
}
```

Using PUT with client-provided ID makes it idempotent.

**Real-world example:** Stripe uses idempotency keys for payment API.

---

### Answer 3: PUT vs PATCH - When to use each

**Use PUT when:**
- You want to **replace the entire resource**
- Client has complete resource data
- All fields must be specified

**Use PATCH when:**
- You want to **update specific fields only**
- Client has partial data
- Other fields should remain unchanged

**Real-world example: User Profile**

**Scenario 1: User updates entire profile (PUT)**
```http
PUT /users/123

{
  "name": "John Smith",
  "email": "john@example.com",
  "phone": "+1234567890",
  "address": "123 Main St",
  "bio": "Software engineer"
}
```
All fields are specified. Anything missing would be removed.

**Scenario 2: User updates only email (PATCH) ✅ Preferred**
```http
PATCH /users/123

{
  "email": "newemail@example.com"
}
```
Only email changes. Name, phone, address, bio remain unchanged.

**Most modern APIs prefer PATCH** because:
- Safer (doesn't accidentally remove fields)
- More bandwidth-efficient
- Matches real user behavior (partial updates)

**Real examples:**
- GitHub API: Uses PATCH for updates
- Stripe API: Uses POST for updates (legacy reasons)
- Twitter API: Uses PATCH for profile updates

---

### Answer 4: Calling DELETE multiple times

**Not a problem - DELETE is idempotent.**

```
DELETE /users/123 (1st call)
-> 204 No Content (user deleted)

DELETE /users/123 (2nd call)
-> 404 Not Found (already deleted)
```

**Final state is the same:** User 123 doesn't exist.

**Why this matters:**

1. **Network retries are safe**
   - If client doesn't receive response (timeout)
   - Safe to retry DELETE

2. **Distributed systems**
   - Multiple processes might try to delete same resource
   - No side effects from duplicate deletes

**Implementation example (Go):**
```go
func DeleteUser(id int) error {
    result := db.Delete(&User{}, id)
    
    if result.RowsAffected == 0 {
        return ErrNotFound // 404
    }
    
    return nil // 204
}
```

Both calls succeed (200/404), and final state is identical.

---

### Answer 5: Should GET have a request body?

**No, GET requests should NOT have a body.**

**Reasons:**

1. **HTTP specification**
   - GET is defined as safe and cacheable
   - Body is not part of cache key
   - Proxies/caches may ignore body

2. **Semantic meaning**
   - GET = "retrieve data"
   - Body implies "sending data to modify"
   - Contradicts REST principles

3. **Practical issues**
   ```
   Many HTTP clients don't support GET with body:
   - Browser fetch() may strip body
   - Some proxies drop body
   - Load balancers may reject it
   ```

4. **Alternatives exist**
   - Query parameters: `GET /users?status=active&role=admin`
   - POST for complex searches: `POST /users/search`

**Example - Wrong approach:**
```http
❌ GET /users
{
  "filters": {
    "status": "active",
    "age": { "min": 18, "max": 65 }
  }
}
```

**Correct approaches:**

**Option 1: Query parameters (simple filters)**
```http
✅ GET /users?status=active&age_min=18&age_max=65
```

**Option 2: POST for complex search (if query params become unwieldy)**
```http
✅ POST /users/search
{
  "filters": {
    "status": "active",
    "age": { "min": 18, "max": 65 },
    "tags": ["developer", "senior"]
  }
}
```

**Real-world example:**
- Elasticsearch uses `POST /_search` for complex queries
- GitHub API uses query parameters for simple searches

</details>

</details>