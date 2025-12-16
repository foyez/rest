# RESTful API

* A **RESTful API** is a standard way for a **client (frontend)** to communicate with a **server (backend)** over HTTP.
* **REST (Representational State Transfer)** is an architectural style defined by **Roy Fielding (2000)**.
* An API is **RESTful** if it follows REST principles such as:

  * Stateless communication
  * Resource-based URLs
  * Standard HTTP methods
  * Proper HTTP status codes

---

## RESTful API Design

### 1. Resource-Based Endpoints (Simple Endpoints)

Use **nouns**, not verbs.
HTTP methods describe the action.

| Method | Endpoint        | Use case                  |
| ------ | --------------- | ------------------------- |
| GET    | /posts          | Get all posts             |
| POST   | /posts          | Create a new post         |
| GET    | /posts/{postId} | Get a single post         |
| PUT    | /posts/{postId} | Replace a post completely |
| PATCH  | /posts/{postId} | Update part of a post     |
| DELETE | /posts/{postId} | Delete a post             |

---

### 2. Nested Endpoints (Relationships)

Use nesting only to show ownership or relationship.

| Method | Endpoint                             | Use case                |
| ------ | ------------------------------------ | ----------------------- |
| GET    | /posts/{postId}/comments             | Get comments of a post  |
| POST   | /posts/{postId}/comments             | Add a comment to a post |
| GET    | /posts/{postId}/comments/{commentId} | Get a specific comment  |
| GET    | /posts?author=john                   | Get posts by author     |

**Rule:** Avoid nesting deeper than **2–3 levels** to keep URLs readable.

---

### 3. Filtering, Sorting & Pagination

Use **query parameters** for non-resource data.

| Example                     | Meaning                         |
| --------------------------- | ------------------------------- |
| /posts?tags=go,rust         | Filter posts by tags            |
| /posts?limit=20&offset=40   | Offset pagination               |
| /posts?limit=20&after_id=30 | Cursor/seek pagination          |
| /posts?sort=created,-email  | Sort by created asc, email desc |

---

### 4. API Versioning

Version your API to avoid breaking existing clients.

```
GET /api/v1/products
```

✔ Version in URL is the most common and clear approach.

---

## HTTP Status Codes

### Status Code Categories

| Range | Meaning       | Details                                                             |
| ----: | ------------- | ------------------------------------------------------------------- |
|   1xx | Informational | Request received; processing is continuing                          |
|   2xx | Success       | Request was successfully received, understood, and processed        |
|   3xx | Redirection   | Client must take additional action to complete the request          |
|   4xx | Client Error  | Request is invalid or cannot be fulfilled due to client-side issues |
|   5xx | Server Error  | Request was valid, but the server failed to process it              |

---

## Common HTTP Status Codes

### Success (2xx)

| Code               | When to use                                             |
| ------------------ | ------------------------------------------------------- |
| **200 OK**         | Successful request with response body (GET, PUT, PATCH) |
| **201 Created**    | Resource successfully created (POST)                    |
| **202 Accepted**   | Request accepted but processed asynchronously           |
| **204 No Content** | Success with no response body (DELETE, sometimes PUT)   |

### Example Requests & Responses of 2xx


<details>
<summary>View contents</summary>

## 1. Create a Resource

### **Request**

**Endpoint**

```http
POST /posts
```

**Headers**

```http
Authorization: Bearer <token>
Content-Type: application/json
```

**Body**

```json
{
  "title": "REST API Design",
  "content": "Best practices"
}
```

---

### **Response**

**Status**

```http
201 Created
```

**Body**

```json
{
  "id": 123,
  "title": "REST API Design",
  "content": "Best practices",
  "author": "john",
  "tags": ["api", "rest"],
  "created_at": "2025-01-01T10:00:00Z",
  "updated_at": "2025-01-02T12:00:00Z"
}
```

---

## 2. Get Resources

### **Request**

**Endpoint**

```http
GET /posts?limit=10
```

---

### **Response**

**Status**

```http
200 OK
```

**Body**

```json
[
  {
    "id": 123,
    "title": "REST API Design",
    "content": "Best practices",
    "author": "john",
    "tags": ["api", "rest"],
    "created_at": "2025-01-01T10:00:00Z",
    "updated_at": "2025-01-02T12:00:00Z"
  }
]
```

---

## 3. PUT — Full Replacement

### **Request**

**Endpoint**

```http
PUT /posts/123
```

**Body**

```json
{
  "title": "New title",
  "content": "Updated content",
  "tags": []
}
```

---

### **Response**

**Status**

```http
200 OK
```

**Body**

```json
{
  "id": 123,
  "title": "New title",
  "content": "Updated content",
  "author": "john",
  "tags": [],
  "created_at": "2025-01-01T10:00:00Z",
  "updated_at": "2025-01-02T12:00:00Z"
}
```

✔ Missing fields are removed
✔ Use only when client sends full resource

---

## 4. PATCH — Partial Update (Preferred)

### **Request**

**Endpoint**

```http
PATCH /posts/123
```

**Body**

```json
{
  "title": "New title"
}
```

---

### **Response**

**Status**

```http
200 OK
```

**Body**

```json
{
  "id": 123,
  "title": "New title",
  "content": "Best practices",
  "author": "john",
  "tags": ["api", "rest"],
  "created_at": "2025-01-01T10:00:00Z",
  "updated_at": "2025-01-02T12:00:00Z"
}
```

✔ Updates only provided fields
✔ Safer and more common

## 5. DELETE - Delete a post

```http
DELETE /posts/123
```

**Success**

```http
204 No Content
```

**If not found**

```http
404 Not Found
```

</details>

---

### Client Errors (4xx)

#### 400 vs 422 (Validation Errors)

| Code                         | When to use                                        |
| ---------------------------- | -------------------------------------------------- |
| **400 Bad Request**          | Invalid or malformed request                       |
| **422 Unprocessable Entity** | Validation failed (missing fields, invalid values) |

✔ Example:

* Missing required field → **422**
* Invalid JSON → **400**

---

### Authentication & Authorization

#### 401 vs 403 (Very Common Confusion)

| Code                 | Meaning                                     | Real Use Case                       |
| -------------------- | ------------------------------------------- | ----------------------------------- |
| **401 Unauthorized** | Client is **not authenticated**             | Missing or invalid token            |
| **403 Forbidden**    | Client is authenticated but **not allowed** | User logged in but lacks permission |

✔ Examples:

* No JWT token → **401**
* Logged in user trying to access admin-only endpoint → **403**

**Key difference:**

* **401 = Who are you?**
* **403 = You are known, but not allowed**

---

### Other Common 4xx Codes

| Code                           | When to use                                                  |
| ------------------------------ | ------------------------------------------------------------ |
| **404 Not Found**              | Resource does not exist or you don’t want to reveal it       |
| **405 Method Not Allowed**     | HTTP method not supported (must include `Allow` header)      |
| **406 Not Acceptable**         | Client requested unsupported response format                 |
| **409 Conflict**               | Resource conflict (duplicate email, username already exists) |
| **415 Unsupported Media Type** | Unsupported request body format                              |

---

### Example Requests & Responses of 4xx


<details>
<summary>View contents</summary>

#### 1. Missing Token

```http
GET /posts
```

```http
401 Unauthorized
```

**Meaning:** Client is not authenticated

---

#### 2. No Permission

```http
DELETE /posts/123
```

```http
403 Forbidden
```

**Meaning:** Client is authenticated but lacks permission

---

#### 3. Validation Error

```http
POST /posts
```

```json
{
  "title": ""
}
```

```http
422 Unprocessable Entity
```

```json
{
  "error": "Validation failed",
  "fields": {
    "title": "Title is required"
  }
}
```

**Error Response Best Practices:**

- Consistent format
- Clear message
- Field-level errors

---

#### 4. Conflict Error

```http
POST /users
```

```http
409 Conflict
```

**Use case:** Email or username already exists

</details>

### Server Errors (5xx)

| Code | When to use                    |
| ---- | ------------------------------ |
| 500  | Unexpected server error        |
| 501  | Feature not implemented        |
| 502  | Invalid upstream response      |
| 503  | Server temporarily unavailable |
| 504  | Upstream timeout               |

### 500 — Internal Server Error

**Meaning:**
Unexpected server failure.

**Use when:**

* Unhandled exceptions
* Database crashes
* Null pointer / runtime errors
* Any error you didn’t explicitly handle

### 502 — Bad Gateway

**Meaning:**
The server received an invalid response from an upstream service.

**Use when:**

* API gateway → microservice fails
* Reverse proxy (NGINX) can’t reach backend
* Third-party service returns invalid response

### 503 — Service Unavailable

**Meaning:**
Server is temporarily unavailable.

**Use when:**

* Server is overloaded
* Maintenance mode
* Database is down

### 504 — Gateway Timeout

**Meaning:**
Server didn’t get a response in time from an upstream service.

**Use when:**

* Timeout while calling another service
* Long-running queries time out

### 501 — Not Implemented

**Meaning:**
The server does not support the requested functionality.

**Use when:**

* Endpoint exists but feature isn’t implemented yet
* Method is recognized but intentionally unavailable

```http
501 Not Implemented
```

### 505 — HTTP Version Not Supported

**Meaning:**
Server does not support the HTTP protocol version.

**Use when:**

* Client uses unsupported HTTP version

```http
505 HTTP Version Not Supported
```

### Example Requests & Responses of 5xx

<details>
<summary>View contents</summary>

#### 500 — Internal Server Error

```http
500 Internal Server Error
```

```json
{
  "error": "Internal server error",
  "message": "Something went wrong. Please try again later."
}
```

- Do NOT expose stack traces
- Log details internally

---


#### 502 — Bad Gateway

```http
502 Bad Gateway
```

```json
{
  "error": "Bad gateway",
  "message": "Upstream service failed"
}
```

---

#### 503 — Service Unavailable

```http
503 Service Unavailable
Retry-After: 60
```

```json
{
  "error": "Service unavailable",
  "message": "Please retry after some time"
}
```

✔ Use `Retry-After` header if possible

---

#### 504 — Gateway Timeout

```http
504 Gateway Timeout
```

```json
{
  "error": "Gateway timeout",
  "message": "Upstream service took too long to respond"
}
```

---


#### Best Practices for 5xx Responses

- Keep error messages generic
- Do not expose stack traces or internal details
- Log detailed errors internally
- Use correlation/request IDs
- Return consistent error format
- Prefer retries for 503/504
- Add monitoring & alerts

---

#### Recommended Error Response Format

```json
{
  "error": "Service unavailable",
  "message": "Please try again later",
  "request_id": "abc123"
}
```

---

</details>

## REST API Status Code Cheat Sheet

| Scenario           | Code |
| ------------------ | ---- |
| Success with data  | 200  |
| Created resource   | 201  |
| Delete success     | 204  |
| Invalid request    | 400  |
| Validation error   | 422  |
| Not authenticated  | 401  |
| No permission      | 403  |
| Resource not found | 404  |
| Method not allowed | 405  |
| Conflict           | 409  |
| Server error       | 500  |

## REST API Design Checklist

-  Resource-based URLs (`/posts`, `/users`)
- Use HTTP methods correctly
- No verbs in URLs
- Proper status codes
- Pagination for lists
- Version your API
- Consistent error format
- Stateless requests
- Secure with HTTPS

## Best Practices Summary

- Use **nouns** for endpoints
- Use **HTTP methods** correctly
- Use **query params** for filtering/sorting
- Use **401 for unauthenticated**, **403 for unauthorized**
- Use **204 for DELETE**
- Use **422 for validation errors**
- Version your API


[resource - http status codes](https://restfulapi.net/http-status-codes/)
