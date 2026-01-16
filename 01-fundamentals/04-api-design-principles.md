# 1.4 API Design Principles

## Core Principles

### 1. Resource-Based URLs

Use **nouns**, not **verbs**. HTTP methods provide the action.

**✅ Good:**
```
GET    /products
POST   /products
GET    /products/123
PUT    /products/123
DELETE /products/123
```

**❌ Bad:**
```
GET    /getAllProducts
POST   /createProduct
GET    /getProductById/123
POST   /updateProduct/123
POST   /deleteProduct/123
```

---

### 2. Use Proper HTTP Methods

| Operation | Method | Endpoint |
|-----------|--------|----------|
| List all | GET | /products |
| Get one | GET | /products/123 |
| Create | POST | /products |
| Replace | PUT | /products/123 |
| Update | PATCH | /products/123 |
| Delete | DELETE | /products/123 |

---

### 3. Nested Resources (Relationships)

Show ownership/relationship through nesting.

**✅ Good nesting (2 levels max):**
```
GET /users/123/posts              # User's posts
GET /users/123/posts/456          # Specific post
GET /users/123/posts/456/comments # Comments on post
```

**❌ Too deep:**
```
GET /users/123/posts/456/comments/789/likes/012
```

**Alternative for deep nesting:**
```
GET /comments/789/likes
GET /likes?comment_id=789
```

---

### 4. Plural Nouns for Collections

**✅ Consistent:**
```
/users
/products
/orders
```

**❌ Inconsistent:**
```
/user       # Singular
/products   # Plural
/order      # Singular
```

---

### 5. Use Query Parameters for Filtering

**Never in URL path:**
```
❌ /products/electronics
❌ /products/priceBelow/100
```

**✅ Use query parameters:**
```
GET /products?category=electronics
GET /products?price_max=100
GET /products?category=electronics&price_max=100&sort=price
```

---

## URL Naming Conventions

### Use Kebab-Case

**✅ Recommended:**
```
/product-categories
/user-preferences
/order-history
```

**❌ Avoid:**
```
/productCategories  # camelCase
/Product_Categories # PascalCase
/product_categories # snake_case (common in databases, not URLs)
```

### Lowercase Only

```
✅ /products
❌ /Products
❌ /PRODUCTS
```

---

## Filtering, Sorting & Pagination

### Filtering

```http
GET /products?category=electronics&brand=sony&in_stock=true
```

**Multiple values:**
```http
GET /products?tags=laptop,gaming,portable
GET /products?tags[]=laptop&tags[]=gaming
```

---

### Sorting

**Single field:**
```http
GET /products?sort=price           # Ascending
GET /products?sort=-price          # Descending (minus prefix)
```

**Multiple fields:**
```http
GET /products?sort=category,-price,name
# Sort by category asc, then price desc, then name asc
```

---

### Pagination

**Offset-based (page number):**
```http
GET /products?page=2&limit=20
```

Response:
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

**Cursor-based (for large datasets):**
```http
GET /products?limit=20&cursor=eyJpZCI6MTIzfQ
```

Response:
```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTQzfQ",
    "has_more": true
  }
}
```

---

## API Versioning

### Why Version APIs?

- Avoid breaking existing clients
- Allow gradual migration
- Support multiple versions simultaneously

### Versioning Strategies

#### 1. URL Path (Most Common) ✅

```http
GET /api/v1/products
GET /api/v2/products
```

**Pros:**
- Clear and explicit
- Easy to route
- Easy to test

**Cons:**
- Version in every URL

---

#### 2. Header Versioning

```http
GET /api/products
Accept: application/vnd.api+json; version=1
```

**Pros:**
- Cleaner URLs
- RESTful purist approach

**Cons:**
- Less visible
- Harder to test in browser

---

#### 3. Query Parameter

```http
GET /api/products?version=1
```

**Pros:**
- Simple

**Cons:**
- Caching issues
- Not RESTful

---

### Best Practice: URL Versioning

```
https://api.example.com/v1/products
https://api.example.com/v2/products
```

**Real examples:**
- Stripe: `https://api.stripe.com/v1/charges`
- GitHub: `https://api.github.com/repos/{owner}/{repo}`
- Twitter: `https://api.twitter.com/2/tweets`

---

## Response Structure

### Consistent Format

**Single resource:**
```json
{
  "id": 123,
  "name": "Product Name",
  "price": 99.99,
  "created_at": "2025-01-16T10:00:00Z"
}
```

**Collection:**
```json
{
  "data": [
    { "id": 1, "name": "Product 1" },
    { "id": 2, "name": "Product 2" }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150
  }
}
```

---

### Error Format

```json
{
  "error": "Validation failed",
  "message": "Email is required",
  "fields": {
    "email": "Email is required",
    "age": "Age must be positive"
  },
  "request_id": "abc-123",
  "timestamp": "2025-01-16T10:30:00Z"
}
```

---

## Date & Time Format

**Use ISO 8601 format:**
```json
{
  "created_at": "2025-01-16T10:30:00Z",
  "updated_at": "2025-01-16T12:45:30.123Z"
}
```

**Never:**
```json
❌ "created_at": "01/16/2025"          # Ambiguous
❌ "created_at": "1737025800"          # Unix timestamp
❌ "created_at": "2025-01-16 10:30:00" # Missing timezone
```

---

## Field Naming

### Use snake_case in JSON (Most Common)

```json
{
  "user_id": 123,
  "created_at": "2025-01-16T10:00:00Z",
  "is_active": true
}
```

### Or camelCase (JavaScript preference)

```json
{
  "userId": 123,
  "createdAt": "2025-01-16T10:00:00Z",
  "isActive": true
}
```

**Be consistent!** Don't mix styles.

---

## HATEOAS (Optional)

**Hypermedia as the Engine of Application State**

Include links to related resources:

```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "_links": {
    "self": "/users/123",
    "posts": "/users/123/posts",
    "followers": "/users/123/followers",
    "following": "/users/123/following"
  }
}
```

**Rarely implemented in practice.** Most APIs stop at Level 2 of Richardson Maturity Model.

---

## Security Best Practices

### 1. Always Use HTTPS

```
✅ https://api.example.com
❌ http://api.example.com
```

### 2. Authentication in Headers

```http
✅ Authorization: Bearer token123

❌ GET /users?token=token123
```

Never send tokens in:
- URL parameters
- URL path
- Cookies (unless using session auth)

### 3. Rate Limiting

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 75
X-RateLimit-Reset: 1737027600
```

### 4. Input Validation

Validate all inputs server-side:
- Email format
- Data types
- Length limits
- Allowed values

---

## Idempotency

**Idempotent methods:** Same request → Same result

| Method | Idempotent? |
|--------|-------------|
| GET | ✅ |
| PUT | ✅ |
| DELETE | ✅ |
| POST | ❌ |
| PATCH | ⚠️ Depends |

### Idempotency Keys for POST

```http
POST /orders
Idempotency-Key: unique-client-key-123

{
  "product_id": 456,
  "quantity": 2
}
```

Server tracks key to prevent duplicate orders.

---

## Design Checklist

- [ ] Resource-based URLs (nouns, not verbs)
- [ ] Proper HTTP methods
- [ ] Proper status codes
- [ ] API versioning (/v1/, /v2/)
- [ ] Pagination for collections
- [ ] Filtering via query parameters
- [ ] Consistent error format
- [ ] ISO 8601 dates
- [ ] snake_case or camelCase (consistent)
- [ ] HTTPS only
- [ ] Rate limiting
- [ ] Input validation
- [ ] Documentation

---

## Real-World Example: Blog API

### Endpoints

```
# Posts
GET    /api/v1/posts                    List posts
POST   /api/v1/posts                    Create post
GET    /api/v1/posts/123                Get post
PATCH  /api/v1/posts/123                Update post
DELETE /api/v1/posts/123                Delete post

# Comments
GET    /api/v1/posts/123/comments       List comments for post
POST   /api/v1/posts/123/comments       Add comment
DELETE /api/v1/comments/456             Delete comment

# Users
GET    /api/v1/users/789                Get user profile
GET    /api/v1/users/789/posts          Get user's posts

# Filtering & Pagination
GET    /api/v1/posts?author=john&limit=20&page=2
GET    /api/v1/posts?tags=tech,golang&sort=-created_at
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. REST APIs should use __________ for resources, not verbs.
2. The recommended API versioning strategy is __________ versioning.
3. Dates should be formatted using the __________ standard.
4. Query parameters should be used for __________ and __________.

### True/False

1. ⬜ URLs should use camelCase for consistency
2. ⬜ POST requests are idempotent
3. ⬜ Nesting resources more than 2-3 levels deep is recommended
4. ⬜ Authentication tokens should be sent in URL parameters
5. ⬜ HATEOAS is widely implemented in production APIs

### Multiple Choice

1. Which URL follows REST best practices?
   - A) `/getAllUsers`
   - B) `/users`
   - C) `/user`
   - D) `/getUsers`

2. How should you filter products by category?
   - A) `/products/electronics`
   - B) `/products?category=electronics`
   - C) `/products/filter/electronics`
   - D) `/getProductsByCategory/electronics`

3. Which date format should you use?
   - A) `01/16/2025`
   - B) `2025-01-16T10:30:00Z`
   - C) `1737025800`
   - D) `January 16, 2025`

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. nouns
2. URL path (or URL)
3. ISO 8601
4. filtering, sorting, pagination (any two)

**True/False:**
1. ❌ False - Use kebab-case for URLs
2. ❌ False - POST is NOT idempotent
3. ❌ False - Avoid deep nesting, keep it 2 levels max
4. ❌ False - Tokens should be in Authorization header
5. ❌ False - HATEOAS is rarely used in production

**Multiple Choice:**
1. **B** - `/users` uses plural nouns
2. **B** - Query parameters for filtering
3. **B** - ISO 8601 format with timezone

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: What are the main principles of REST API design?

### Question 2: How would you design endpoints for a blog with posts and comments?

### Question 3: Explain different API versioning strategies and which you prefer

### Question 4: How should pagination be implemented for large datasets?

### Question 5: What's wrong with this API design? `/getUsers?token=abc123`

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

### Answer 1: Main principles of REST API design

1. **Resource-based URLs**
   - Use nouns: `/users`, `/products`
   - Not verbs: `/getUsers`, `/createProduct`

2. **HTTP methods for actions**
   - GET for retrieval
   - POST for creation
   - PUT/PATCH for updates
   - DELETE for deletion

3. **Proper status codes**
   - 200 for success
   - 201 for created
   - 400 for client errors
   - 500 for server errors

4. **Stateless communication**
   - Each request is independent
   - No server-side session storage
   - Auth token in every request

5. **Consistent structure**
   - Standard error format
   - Consistent naming (snake_case or camelCase)
   - ISO 8601 dates

6. **Versioning**
   - Support multiple API versions
   - Usually `/v1/`, `/v2/`

7. **Use query parameters**
   - Filtering: `?category=electronics`
   - Sorting: `?sort=-price`
   - Pagination: `?page=2&limit=20`

---

### Answer 2: Blog API design

```
# Posts
GET    /api/v1/posts
POST   /api/v1/posts
GET    /api/v1/posts/{postId}
PATCH  /api/v1/posts/{postId}
DELETE /api/v1/posts/{postId}

# Comments (nested under posts)
GET    /api/v1/posts/{postId}/comments
POST   /api/v1/posts/{postId}/comments

# Comments (direct access for actions)
GET    /api/v1/comments/{commentId}
PATCH  /api/v1/comments/{commentId}
DELETE /api/v1/comments/{commentId}

# Authors
GET    /api/v1/users/{userId}
GET    /api/v1/users/{userId}/posts

# Filtering & Search
GET    /api/v1/posts?author={userId}
GET    /api/v1/posts?tags=tech,golang
GET    /api/v1/posts?published=true&sort=-created_at
```

**Key decisions:**

1. **Nested for creation:**
   ```
   POST /api/v1/posts/123/comments
   ```
   Shows ownership relationship

2. **Direct for updates/delete:**
   ```
   DELETE /api/v1/comments/456
   ```
   Simpler, shorter URL

3. **Filtering via query params:**
   ```
   GET /api/v1/posts?author=john&tags=tech
   ```
   Not `/posts/author/john/tags/tech`

---

### Answer 3: API Versioning strategies

**1. URL Path Versioning** ✅ Recommended

```
GET /api/v1/products
GET /api/v2/products
```

**Pros:**
- Explicit and clear
- Easy to test (just change URL)
- Simple routing
- Works in browser

**Cons:**
- Version in every URL
- Duplication

**Real examples:** Stripe, GitHub, Twitter

---

**2. Header Versioning**

```http
GET /api/products
Accept: application/vnd.api+json; version=2
```

**Pros:**
- Cleaner URLs
- RESTful purist approach
- Same URL for all versions

**Cons:**
- Less discoverable
- Can't test in browser easily
- More complex routing

---

**3. Query Parameter**

```
GET /api/products?version=2
```

**Pros:**
- Simple to implement

**Cons:**
- Caching issues (different versions cached together)
- Pollutes query string
- Not RESTful

---

**4. Media Type Versioning**

```http
GET /api/products
Accept: application/vnd.api.v2+json
```

**Pros:**
- Very RESTful
- Content negotiation

**Cons:**
- Complex
- Harder to test

---

**My preference: URL Path Versioning**

**Why:**
1. Clear and explicit
2. Easy to test and debug
3. Simple routing logic
4. Industry standard
5. Works everywhere (browsers, curl, Postman)

**Example implementation:**
```go
r.HandleFunc("/api/v1/products", v1.GetProducts)
r.HandleFunc("/api/v2/products", v2.GetProducts)
```

**Version migration strategy:**
```
1. Release v2
2. Deprecate v1 (announce sunset)
3. Support both v1 and v2 (6-12 months)
4. Remove v1
```

---

### Answer 4: Pagination strategies

**1. Offset-based (Page number)** - Simple, most common

```http
GET /products?page=2&limit=20
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 150,
    "total_pages": 8,
    "has_next": true,
    "has_prev": true
  }
}
```

**Pros:**
- Simple to implement
- Easy to jump to specific page
- Total count available

**Cons:**
- Performance issues with large offsets
- Inconsistent results if data changes (new items added)

**SQL:**
```sql
SELECT * FROM products
LIMIT 20 OFFSET 40  -- Page 3
```

---

**2. Cursor-based** - Better for large datasets

```http
GET /products?limit=20&cursor=eyJpZCI6MTIzfQ
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTQzfQ",
    "prev_cursor": "eyJpZCI6MTAzfQ",
    "has_more": true
  }
}
```

**Cursor is base64-encoded:**
```
eyJpZCI6MTIzfQ == {"id":123}
```

**Pros:**
- Consistent results (handles inserts/deletes)
- Better performance
- Works with infinite scroll

**Cons:**
- Can't jump to specific page
- No total count
- More complex implementation

**SQL:**
```sql
SELECT * FROM products
WHERE id > 123  -- Cursor
ORDER BY id
LIMIT 20
```

---

**3. Keyset Pagination** - Hybrid approach

```http
GET /products?after_id=123&limit=20
```

**SQL:**
```sql
SELECT * FROM products
WHERE id > 123
ORDER BY id
LIMIT 20
```

---

**When to use each:**

| Use Case | Strategy |
|----------|----------|
| Admin dashboards | Offset-based |
| Social media feeds | Cursor-based |
| Search results | Offset-based |
| Infinite scroll | Cursor-based |
| Small datasets (<10k) | Offset-based |
| Large datasets (>100k) | Cursor-based |

---

### Answer 5: What's wrong with `/getUsers?token=abc123`

**Multiple problems:**

**1. Verb in URL**
```
❌ /getUsers
✅ /users (GET method implies retrieval)
```

**2. Token in URL**
```
❌ ?token=abc123
✅ Authorization: Bearer abc123 (header)
```

**Why tokens in URL are bad:**

a) **Security risk:**
   - URLs are logged (server logs, proxy logs, browser history)
   - Visible in browser address bar
   - Can be shared accidentally

b) **Caching issues:**
   - CDNs cache by URL
   - Token changes = different URL
   - Can't cache effectively

c) **Not RESTful:**
   - Authentication is not a resource parameter
   - Should be separate concern (header)

**Correct design:**

```http
GET /api/v1/users
Authorization: Bearer abc123
```

**Additional issues not mentioned:**

3. **Singular vs Plural:**
   - Should be `/users` (plural) for consistency

4. **No versioning:**
   - Should be `/api/v1/users`

**Complete before/after:**

**❌ Before:**
```http
GET /getUsers?token=abc123
```

**✅ After:**
```http
GET /api/v1/users
Authorization: Bearer abc123
```

</details>

</details>