# 1.1 REST API Basics

## What is an API?

An **API (Application Programming Interface)** is a contract that defines how software components communicate. It specifies:
- What operations are available
- What data to send
- What data you'll get back
- How errors are handled

Think of it like a restaurant menu: it tells you what you can order (endpoints), what information you need to provide (parameters), and what you'll receive (response).

APIs can be:

* Web-based
* Local (inside the same machine)
* Based on different protocols and styles

**Examples**

* Payment API (Stripe, PayPal)
* Weather API
* Database API
* OS APIs (Windows API, POSIX)


## What is REST?

**REST (Representational State Transfer)** is an architectural style for designing networked applications, defined by Roy Fielding in 2000.

A **RESTful API** enables communication between a **client** (frontend, mobile app) and a **server** (backend) over HTTP.

---

## REST Principles

### 1. Client-Server Architecture
- **Client** and **server** are separate
- Client handles UI/UX
- Server manages data and business logic
- They communicate via HTTP requests

```
┌─────────────┐                    ┌─────────────┐
│             │   HTTP Request     │             │
│   Client    │ -----------------> │   Server    │
│ (Frontend)  │                    │  (Backend)  │
│             │ <----------------- │             │
│             │   HTTP Response    │             │
└─────────────┘                    └─────────────┘
```

---

### 2. Stateless Communication
- Each request contains **all information** needed to process it
- Server does **not store** client state between requests
- Authentication info (like JWT) is sent with **every request**

**Example:**
```http
GET /api/v1/users/123
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The server doesn't remember you from the previous request. You must send your token every time.

---

### 3. Resource-Based
- Everything is a **resource** (user, post, product)
- Each resource has a **unique URL**
- Use **nouns**, not verbs in URLs

✅ Good:
```
GET /users/123
POST /products
```

❌ Bad:
```
GET /getUser/123
POST /createProduct
```

---

### 4. Uniform Interface
Use standard HTTP methods:

| Method | Purpose | Example |
|--------|---------|---------|
| GET | Retrieve data | Get user details |
| POST | Create new resource | Create a new post |
| PUT | Replace entire resource | Update all user fields |
| PATCH | Update part of resource | Update user email only |
| DELETE | Remove resource | Delete a comment |

---

### 5. Cacheable
- Responses should indicate if they can be cached
- Improves performance and reduces server load

```http
Cache-Control: max-age=3600
```

---

### 6. Layered System
- Client doesn't know if it's connected directly to the end server
- May go through load balancers, proxies, caches

```
Client -> Load Balancer -> API Gateway -> Microservice -> Database
```

---

## Real-World Example: E-commerce API

### Resource Structure

```
/api/v1/products              # All products
/api/v1/products/456          # Specific product
/api/v1/products/456/reviews  # Reviews for product 456
/api/v1/users/123             # User details
/api/v1/users/123/orders      # Orders for user 123
/api/v1/orders/789            # Specific order
```

### Example Flow: Buying a Product

```
1. Client: GET /api/v1/products?category=electronics&limit=20
   Server: Returns list of 20 electronics products

2. Client: GET /api/v1/products/456
   Server: Returns details of product 456

3. Client: GET /api/v1/products/456/reviews
   Server: Returns reviews for product 456

4. Client: POST /api/v1/orders
   Body: { "product_id": 456, "quantity": 2 }
   Server: Creates order, returns 201 Created

5. Client: GET /api/v1/users/123/orders
   Server: Returns all orders for user 123
```

---

## REST vs SOAP

| Feature | REST | SOAP |
|---------|------|------|
| Protocol | HTTP | HTTP, SMTP, TCP |
| Data Format | JSON, XML | Only XML |
| Performance | Faster | Slower |
| Complexity | Simple | Complex |
| Use Case | Web/Mobile APIs | Enterprise systems |

**Real-world usage:**
- **REST**: Twitter API, GitHub API, Stripe API
- **SOAP**: Banking systems, legacy enterprise apps

---

## REST Maturity Model (Richardson)

### Level 0: The Swamp of POX
- Single endpoint, single HTTP method
- XML over HTTP

```
POST /api
{ "action": "getUser", "id": 123 }
```

### Level 1: Resources
- Multiple endpoints, each representing a resource

```
POST /users
GET /users/123
```

### Level 2: HTTP Verbs
- Use proper HTTP methods
- Proper status codes

```
GET /users/123          -> 200 OK
POST /users             -> 201 Created
DELETE /users/123       -> 204 No Content
```

### Level 3: HATEOAS
- Hypermedia as the Engine of Application State
- Response includes links to related resources

```json
{
  "id": 123,
  "name": "John Doe",
  "_links": {
    "self": "/users/123",
    "orders": "/users/123/orders",
    "address": "/users/123/address"
  }
}
```

Most modern APIs operate at **Level 2**. Level 3 (HATEOAS) is rare in practice.

---

## Key Takeaways

1. ✅ REST is an **architectural style**, not a protocol
2. ✅ Use **nouns** for resources, **HTTP methods** for actions
3. ✅ Each request is **stateless** (self-contained)
4. ✅ Responses should include proper **status codes**
5. ✅ Design around **resources**, not operations

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. REST stands for __________ State Transfer.
2. In RESTful APIs, each request must be __________, meaning the server doesn't store client state between requests.
3. URLs should use __________ (nouns/verbs) to represent resources.
4. The HTTP method __________ is used to update part of a resource.

### True/False

1. ⬜ REST APIs can only return JSON data
2. ⬜ In REST, the server stores session information between requests
3. ⬜ A URL like `/getUser/123` follows REST principles
4. ⬜ REST was defined by Roy Fielding in 2000
5. ⬜ Most modern APIs implement HATEOAS (Level 3 maturity)

### Multiple Choice

1. Which is a correct RESTful endpoint?
   - A) `POST /createUser`
   - B) `POST /users`
   - C) `GET /getAllUsers`
   - D) `POST /user/create`

2. What does stateless mean in REST?
   - A) The server has no database
   - B) Each request contains all needed information
   - C) The API doesn't use sessions
   - D) Both B and C

3. Which HTTP method is idempotent?
   - A) POST
   - B) PUT
   - C) PATCH
   - D) All of the above

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. Representational
2. stateless
3. nouns
4. PATCH

**True/False:**
1. ❌ False - REST APIs can return JSON, XML, HTML, or other formats
2. ❌ False - REST is stateless; server doesn't store session info
3. ❌ False - Should be `/users/123` (noun-based, no verb)
4. ✅ True
5. ❌ False - Most APIs are at Level 2; HATEOAS is rarely implemented

**Multiple Choice:**
1. **B** - `POST /users` follows REST conventions (noun-based)
2. **D** - Both statements are correct
3. **B** - PUT is idempotent (same request produces same result)

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: What is REST and what are its main principles?

### Question 2: Explain the difference between PUT and PATCH

### Question 3: What does "stateless" mean in REST? How is authentication handled in stateless APIs?

### Question 4: Why should we use nouns instead of verbs in REST endpoints?

### Question 5: What is the Richardson Maturity Model?

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

### Answer 1: What is REST and what are its main principles?

REST (Representational State Transfer) is an architectural style for designing networked applications. 

**Main principles:**
1. **Client-Server**: Separation of concerns
2. **Stateless**: Each request is self-contained
3. **Cacheable**: Responses can be cached for performance
4. **Uniform Interface**: Standard HTTP methods and resource-based URLs
5. **Layered System**: Client doesn't know intermediate layers
6. **Resource-Based**: Everything is a resource with a unique URI

---

### Answer 2: Explain the difference between PUT and PATCH

**PUT:**
- Replaces the **entire resource**
- Client must send all fields
- If fields are missing, they are removed/reset

**PATCH:**
- Updates **only specified fields**
- Client sends only fields to change
- Other fields remain unchanged

**Example:**

Existing resource:
```json
{
  "id": 1,
  "name": "John",
  "email": "john@example.com",
  "age": 30
}
```

**PUT /users/1:**
```json
{
  "name": "John Doe",
  "email": "john@example.com"
}
```
Result: `age` field is removed (full replacement)

**PATCH /users/1:**
```json
{
  "name": "John Doe"
}
```
Result: Only `name` is updated, `email` and `age` remain unchanged

---

### Answer 3: What does "stateless" mean in REST? How is authentication handled?

**Stateless** means:
- Server doesn't store client state between requests
- Each request must contain all information needed to process it
- No server-side sessions

**Authentication in stateless APIs:**

Use **tokens** (JWT) sent with each request:

```http
GET /api/v1/users/123
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The token contains:
- User ID
- Expiration time
- Permissions

Server validates the token on every request without storing session data.

---

### Answer 4: Why should we use nouns instead of verbs in REST endpoints?

**Reasons:**

1. **HTTP methods already define the action**
   - GET = retrieve
   - POST = create
   - PUT/PATCH = update
   - DELETE = delete

2. **Consistency and readability**
   ```
   ✅ GET /users/123      (clear, consistent)
   ❌ GET /getUser/123    (redundant verb)
   ```

3. **Resource-oriented design**
   - REST is about resources, not operations
   - Same URL, different methods = different operations

4. **Scalability**
   ```
   GET    /products      # List products
   POST   /products      # Create product
   GET    /products/5    # Get product
   PUT    /products/5    # Update product
   DELETE /products/5    # Delete product
   ```
   All operations on the same resource use the same base URL.

---

### Answer 5: What is the Richardson Maturity Model?

A model to assess how "RESTful" an API is, with 4 levels:

**Level 0: The Swamp of POX**
- Single endpoint
- Everything is POST
- XML/JSON over HTTP

**Level 1: Resources**
- Multiple endpoints
- Each resource has its own URL
- Still mostly POST

**Level 2: HTTP Verbs** ⭐ (Most common)
- Proper HTTP methods (GET, POST, PUT, DELETE)
- Proper status codes (200, 201, 404, etc.)

**Level 3: HATEOAS**
- Hypermedia controls
- Response includes links to related resources
- Self-discoverable API

**Most production APIs are at Level 2.**

Example Level 3 response:
```json
{
  "id": 123,
  "name": "Product X",
  "_links": {
    "self": "/products/123",
    "reviews": "/products/123/reviews",
    "related": "/products?category=electronics"
  }
}
```

</details>

</details>